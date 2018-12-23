+++
title = "Creating our first Packer template for installing CentOS 6.7 with vmtools"
description = ""
tags = [
    "vagrant",
    "packer",
]
date = "2015-12-25"
categories = [
    "vagrant",
    "packer",
]
topics = [
    "using-packer-with-esxi"
]
highlight = "true"
+++

This is the fourth in a series of posts on <a href="../2015-12-22-pipeline-for-creating-packer-box-files">using a Packer pipeline to generate Vagrant .box files</a>.

In the last two posts we covered <a href="../2015-12-23-installing-esxi-virtual-machine-for-packer-depolyment">installing a ESXi virtual machine for use with Packer</a> and <a href="../2015-12-24-installing-packer-and-ovftool-on-centos">Setting up Packer, ovftool and Apache web server on a CentOS virtual machine</a>. In this post we will be putting all this prep work to use in order to install a CentOS 6.7 image using Packer.

Before we get started I would like to mentioned that many of the configuration files I am using have been influenced by or directly copied from the following github repositories:  

[https://github.com/frapposelli/packer-templates](https://github.com/frapposelli/packer-templates/)  
[https://github.com/nanliu/packer-templates/](https://github.com/nanliu/packer-templates/)  

These public repositories by Fabio and Nan have been extremely helpful in my own understanding of how Packer works.
 Well...enough talking, let's get started...

### 1. Start off by connecting by SSH to CentOS virtual machine we created in the previous post.

{{< highlight bash >}}
sdorsett-mbp:~ sdorsett$ ssh root@192.168.1.52
root@192.168.1.52's password:
Last login: Wed Dec 23 16:58:41 2015 from 192.168.1.163
[root@packer-centos ~]#
{{< /highlight >}}

### 2. Create a new folder for our packer-templates and run "git init" to start tracking changes.

{{< highlight bash >}}
[root@packer-centos ~]# mkdir packer-templates
[root@packer-centos ~]# cd packer-templates/
[root@packer-centos packer-templates]# git init
Initialized empty Git repository in /root/packer-templates/.git/
[root@packer-centos packer-templates]#
{{< /highlight >}}

### 3. Create directory structure for the .iso files, scripts and templates we will be using with Packer.

{{< highlight bash >}}
[root@packer-centos packer-templates]# mkdir -p ~/packer-templates/{iso,scripts,templates}
[root@packer-centos packer-templates]# ls
iso  scripts  templates
[root@packer-centos packer-templates]#
{{< /highlight >}}

### 4. Download the CentOS 6.7 minimal .iso and place it in the packer-templates/iso/ folder

Open [http://isoredirect.centos.org/centos/6/isos/x86_64/](http://isoredirect.centos.org/centos/6/isos/x86_64/) in a browser and pick a mirror to download the .iso from. Copy the URL to the CentOS-6.7-x86_64-minimal.iso image file and use wget to pull the image down in the CentOS virtual machine

{{< highlight bash >}}
[root@packer-centos packer-templates]# cd ~/packer-templates/iso/
[root@packer-centos iso]# wget http://mirror.cs.pitt.edu/centos/6.7/isos/x86_64/CentOS-6.7-x86_64-minimal.iso
--2015-12-23 20:04:30--  http://mirror.cs.pitt.edu/centos/6.7/isos/x86_64/CentOS-6.7-x86_64-minimal.iso
Resolving mirror.cs.pitt.edu... 136.142.23.206
Connecting to mirror.cs.pitt.edu|136.142.23.206|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 414187520 (395M) [application/octet-stream]
Saving to: “CentOS-6.7-x86_64-minimal.iso”

100%[========================================================>] 414,187,520 7.01M/s   in 58s

2015-12-23 20:05:28 (6.77 MB/s) - “CentOS-6.7-x86_64-minimal.iso” saved [414187520/414187520]

[root@packer-centos iso]#
{{< /highlight >}}

### 5. Compare the sha1sum of the downloaded CentOS-6.7-x86_64-minimal.iso file to what is listed on the CentOS mirror.

Open the sha1sum.txt file from the CentOS mirror you chose in your browser. It should show the following:

{{< highlight bash >}}
7bb8c1c23a4fdef93e6f0a6347d570e5764d0b38  CentOS-6.7-x86_64-bin-DVD1.iso
79f58df5723f723fc62d5e8831ada38072096a46  CentOS-6.7-x86_64-bin-DVD2.iso
2ed5ea551dffc3e4b82847b3cee1f6cd748e8071  CentOS-6.7-x86_64-minimal.iso
c3678c6b72cbf2ed9b4e8a1ddb190fd262db8b7f  CentOS-6.7-x86_64-netinstall.iso
a7c05a7c0a4c6bdef8dd1cf8310613c154c2b9d5  CentOS-6.7-x86_64-LiveCD.iso
aebd4c2e81ace92ebda2bc909aca34a139ff2cea  CentOS-6.7-x86_64-LiveDVD.iso
{{< /highlight >}}

Compare the sha1sum from above to the actual sha1sum of the downloaded .iso

{{< highlight bash >}}
[root@packer-centos iso]# sha1sum CentOS-6.7-x86_64-minimal.iso
2ed5ea551dffc3e4b82847b3cee1f6cd748e8071  CentOS-6.7-x86_64-minimal.iso
[root@packer-centos iso]#
{{< /highlight >}}

The two hashes match so we know the file was downloaded without any corruption.

### 6. Download patched vmtools from [https://github.com/rasa/vmware-tools-patches](https://github.com/rasa/vmware-tools-patches) and place the .iso file in the packer-templates/iso/ folder.

I have had issues with the VMware issued vmtools, specifically around CentOS 7 and trying to get hgfs (host guest file system) functional on Fusion. During that troubleshooting I discovered the patched vmware tools listed above and found installing those, rather than the VMware issue vmtools installer, resolved my issues.

{{< highlight bash >}}
[root@packer-centos ~]# wget https://softwareupdate.vmware.com/cds/vmw-desktop/fusion/7.1.3/3204469/packages/com.vmware.fusion.tools.linux.zip.tar
--2015-12-23 21:20:55--  https://softwareupdate.vmware.com/cds/vmw-desktop/fusion/7.1.3/3204469/packages/com.vmware.fusion.tools.linux.zip.tar
Resolving softwareupdate.vmware.com... 23.64.167.143, 2600:1404:a:589::2ef, 2600:1404:a:59a::2ef
Connecting to softwareupdate.vmware.com|23.64.167.143|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 62074880 (59M) [application/x-tar]
Saving to: “com.vmware.fusion.tools.linux.zip.tar”

100%[========================================================>] 62,074,880  6.95M/s   in 8.6s

2015-12-23 21:21:04 (6.91 MB/s) - “com.vmware.fusion.tools.linux.zip.tar” saved [62074880/62074880]

[root@packer-centos ~]# tar xf com.vmware.fusion.tools.linux.zip.tar
[root@packer-centos ~]# unzip com.vmware.fusion.tools.linux.zip
Archive:  com.vmware.fusion.tools.linux.zip
  inflating: manifest.plist
   creating: payload/
  inflating: payload/linux.iso
 extracting: payload/linux.iso.sig
  inflating: payload/tools-linux.plist
[root@packer-centos ~]# cp payload/linux.iso ~/packer-templates/iso/
[root@packer-centos ~]# cd ~/packer-templates/iso/
[root@packer-centos iso]# ln -s ./linux.iso vmware-tools-linux.iso
[root@packer-centos iso]# ls -la
total 466960
drwxr-xr-x 2 root root      4096 Dec 23 21:22 .
drwxr-xr-x 6 root root      4096 Dec 23 21:22 ..
-rw-r--r-- 1 root root 414187520 Aug  4 21:59 CentOS-6.7-x86_64-minimal.iso
-rw-r--r-- 1 root root  63971328 Dec 23 21:21 linux.iso
lrwxrwxrwx 1 root root        11 Dec 23 21:22 vmware-tools-linux.iso -> ./linux.iso
[root@packer-centos iso]#
{{< /highlight >}}

### 7. Create a packer .json file in the template/ folder for our image

Create the following file in the packer-templates/templates/ directory. The centos67.json file contains the instructions Packer will use to install the CentOS 6.7 image.

{{< highlight bash >}}
[root@packer-centos packer-templates]# cat templates/centos67.json
{
  "variables": {
    "version": "1.0"
  },
  "builders": [
    {
      "name": "centos67",
      "vm_name": "centos67",
      "vmdk_name": "centos67",
      "type": "vmware-iso",
      "communicator": "ssh",
      "ssh_pty": "true",
      "headless": false,
      "disk_size": 16384,
      "guest_os_type": "rhel6-64",
      "iso_url": "./iso/CentOS-6.7-x86_64-minimal.iso",
      "iso_checksum": "2ed5ea551dffc3e4b82847b3cee1f6cd748e8071",
      "iso_checksum_type": "sha1",
      "shutdown_command": "echo 'vagrant' | sudo -S /sbin/shutdown -P now",

      "remote_host": {% raw %}"{{user `packer_remote_host`}}",{% endraw %}
      "remote_datastore": {% raw %}"{{user `packer_remote_datastore`}}",{% endraw %}
      "remote_username": {% raw %}"{{user `packer_remote_username`}}",{% endraw %}
      "remote_password": {% raw %}"{{user `packer_remote_password`}}",{% endraw %}
      "remote_type": "esx5",
      "ssh_username": "root",
      "ssh_password": "vagrant",
      "ssh_wait_timeout": "60m",
      "tools_upload_flavor": "linux",
      "http_directory": ".",
      "boot_wait": "5s",
      "vmx_data": {
        "memsize": "4096",
        "numvcpus": "2",
        "ethernet0.networkName": {% raw %}"{{user `packer_remote_network`}}",{% endraw %}
        "ethernet0.present": "TRUE",
        "ethernet0.startConnected": "TRUE",
        "ethernet0.virtualDev": "e1000",
        "ethernet0.addressType": "generated",
        "ethernet0.generatedAddressOffset": "0",
        "ethernet0.wakeOnPcktRcv": "FALSE"

      },
      "vmx_data_post": {
        "ide1:0.startConnected": "FALSE",
        "ide1:0.clientDevice": "TRUE",
        "ide1:0.fileName": "emptyBackingString",
        "ethernet0.virtualDev": "vmxnet3"
      },
      "boot_command": [
        "<tab> text ks=http://{{ .HTTPIP }}:{{ .HTTPPort }}/scripts/centos-6-kickstart.cfg<enter><wait>"
      ]
    }
  ],
  "provisioners": [
    {
      "type": "file",
      "source": "iso/vmware-tools-linux.iso",
      "destination": "/tmp/vmware-tools-linux.iso"
    },
    {
      "type": "shell",
      "execute_command": "echo 'vagrant' | {{ .Vars }} sudo -E -S sh '{{ .Path }}'",
      "script": "scripts/centos-vagrant-settings.sh"
    },
    {
      "type": "shell",
      "execute_command": "echo 'vagrant' | {{ .Vars }} sudo -E -S sh '{{ .Path }}'",
      "script": "scripts/centos-vmware-tools_install.sh"
    },
    {
      "type": "shell",
      "execute_command": "echo 'vagrant' | {{ .Vars }} sudo -E -S sh '{{ .Path }}'",
      "script": "scripts/centos-vmware-cleanup.sh"
    }
  ]
}
{{< /highlight >}}

Let us go over some of what's in this .json file

* the builders block

This section contains builder specific instruction. We are using the vmware-iso builder so much of this is describing how we want to VMware backed virtual machine configured.

You might notice something a little different about remote\_host, remote\_datastore, remote\_username, remote\_password and ethernet0.networkName parameters. These parameters contain information about the ESXi host we are using to create the Packer image, so rather than hard coding this info into each and every template these values can be pulled from user defined variables.

The benefit of separating "data from code" is two fold. First, this prevents sensitive datacenter specific information from being included in our template, which could end up in github or some other git repository. Secondly, we only need to update one location if the password of our remote_host ever needs to be changed.

* the provisioners block

This section contains instruction on what to do after the OS is successfully installed packer. You can use "file" provisioners to copy files into the Packer built OS, as well as "shell" provisioners to run scripts to perform additional configuration. You will get a closer look at "shell" provisioner scripts shortly

### 8. Create the script/ files that Packer will use to install the CentOS 6.7 image

The following files are referenced in the centos67.json template and need to be created in the packer-templates/scripts/ folder

* scripts/centos-6-kickstart.cfg - this is the CentOS 6 kickstart file that describes how CentOS 6 should be installed

{{< highlight bash >}}
[root@packer-centos packer-templates]# cat scripts/centos-6-kickstart.cfg
firewall --disabled

install
cdrom

lang en_US.UTF-8
keyboard us
timezone  Europe/Rome

network --bootproto=dhcp
rootpw vagrant
authconfig --enableshadow --passalgo=sha512

selinux —-disabled
bootloader --location=mbr
text
skipx

logging --level=info
zerombr

clearpart --all --initlabel
autopart

auth  --useshadow  --enablemd5
firstboot --disabled
reboot

%packages --ignoremissing
@Base
@Core
%end

%post
yum install wget git -y
VAGRANT_USER=vagrant
/usr/sbin/groupadd vagrant
/usr/sbin/useradd vagrant -g vagrant -G wheel
echo "vagrant"|passwd --stdin vagrant
echo "vagrant        ALL=(ALL)       NOPASSWD: ALL" >> /etc/sudoers

sed -i "s/^.*requiretty/#Defaults requiretty/" /etc/sudoers
[root@packer-centos packer-templates]#
{{< /highlight >}}

* scripts/centos-vagrant-settings.sh - this is a bash script that describes how vagrant speciffied settings should be configured

{{< highlight bash >}}
[root@packer-centos packer-templates]# cat scripts/centos-vagrant-settings.sh
#!/bin/bash -eux

# Vagrant specific
date > /etc/vagrant_box_build_time

VAGRANT_USER=vagrant
VAGRANT_HOME=/home/$VAGRANT_USER
VAGRANT_KEY_URL=https://raw.github.com/mitchellh/vagrant/master/keys/vagrant.pub

# Add vagrant user (if it doesn't already exist)
if ! id -u $VAGRANT_USER >/dev/null 2>&1; then
    /usr/sbin/groupadd $VAGRANT_USER
    /usr/sbin/useradd $VAGRANT_USER -g $VAGRANT_USER -G wheel
    echo "${VAGRANT_USER}"|passwd --stdin $VAGRANT_USER
    echo "${VAGRANT_USER}        ALL=(ALL)       NOPASSWD: ALL" >> /etc/sudoers
fi

# Installing vagrant keys
mkdir -pm 700 $VAGRANT_HOME/.ssh
echo "ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEA6NF8iallvQVp22WDkTkyrtvp9eWW6A8YVr+kz4TjGYe7gHzIw+niNltGEFHzD8+v1I2YJ6oXevct1YeS0o9HZyN1Q9qgCgzUFtdOKLv6IedplqoPkcmF0aYet2PkEDo3MlTBckFXPITAMzF8dJSIFo9D8HfdOV0IAdx4O7PtixWKn5y2hMNG0zQPyUecp4pzC6kivAIhyfHilFR61RGL+GPXQ2MWZWFYbAGjyiYJnAmCP3NOTd0jMZEnDkbUvxhMmBYSdETk1rRgm+R4LOzFUGaHqHDLKLX+FIPKcF96hrucXzcWyLbIbEgE98OHlnVYCzRdK8jlqm8tehUc9c9WhQ== vagrant insecure public key" >> $VAGRANT_HOME/.ssh/authorized_keys
chmod 0600 $VAGRANT_HOME/.ssh/authorized_keys
chown -R $VAGRANT_USER:$VAGRANT_USER $VAGRANT_HOME/.ssh
[root@packer-centos packer-templates]#
{{< /highlight >}}

* scripts/centos-vmware-tools_install.sh - this is a bash script that describes how vmtools should be installed

{{< highlight bash >}}
[root@packer-centos packer-templates]# cat scripts/centos-vmware-tools_install.sh
#!/bin/bash
sudo yum upgrade ca-certificates -y
sudo yum install perl fuse-libs -y
sudo mkdir /mnt/cdrom
sudo mount -o loop /tmp/vmware-tools-linux.iso /mnt/cdrom
sudo cp /mnt/cdrom/VMwareTools*.tar.gz /tmp/
cd /tmp
tar xfz VMwareTools*.tar.gz
cd /tmp/vmware-tools-distrib
ls -la

# from http://www.virtuallyghetto.com/2015/06/automating-silent-installation-of-vmware-tools-on-linux-wautomatic-kernel-modules.html
# If you wish to change which Kernel modules get installed
# The last four entries (yes,no,no,no) map to the following:
#   VMware Host-Guest Filesystem
#   vmblock enables dragging or copying files
#   VMware automatic kernel modules
#   Guest Authentication
# and you can also change the other params as well
sudo cat > /tmp/answer << __ANSWER__
yes
/usr/bin
/etc
/etc/init.d
/usr/sbin
/usr/lib/vmware-tools
yes
/usr/share/doc/vmware-tools
yes
yes
yes
no
yes
no

__ANSWER__

sudo /tmp/vmware-tools-distrib/vmware-install.pl < /tmp/answer

sudo umount /mnt/cdrom
cd /tmp
rm -rf vmware-tools-distrib
rm -f VMwareTools*.tar.gz
rm /tmp/vmware-tools-linux.iso
[root@packer-centos packer-templates]#
{{< /highlight >}}

* scripts/centos-vmware-cleanup.sh - a bash script that cleans up the Packer image prior to being exported

{{< highlight bash >}}
[root@packer-centos packer-templates]# cat scripts/centos-vmware-cleanup.sh
#!/bin/bash -eux
# Based off the great "Preparing Linux Template VMs"
# (http://lonesysadmin.net/2013/03/26/preparing-linux-template-vms/) article
# by Bob Plankers, thanks Bob!

CLEANUP_PAUSE=${CLEANUP_PAUSE:-0}
echo "==> Pausing for ${CLEANUP_PAUSE} seconds..."
sleep ${CLEANUP_PAUSE}

echo "==> erasing unused packages to free up space"
/usr/bin/yum -y erase gtk2 libX11 hicolor-icon-theme avahi freetype bitstream-vera-fonts

echo "==> Cleaning up yum cache"
/usr/bin/yum clean all

echo "==> Force logs to rotate"
/usr/sbin/logrotate -f /etc/logrotate.conf
/bin/rm -f /var/log/*-???????? /var/log/*.gz

echo "==> Clear audit log and wtmp"
/bin/cat /dev/null > /var/log/audit/audit.log
/bin/cat /dev/null > /var/log/wtmp

echo "==> Cleaning up udev rules"
/bin/rm -f /etc/udev/rules.d/70*

echo "==> Remove the traces of the template MAC address and UUIDs"
/bin/sed -i '/^\(HWADDR\|UUID\)=/d' /etc/sysconfig/network-scripts/ifcfg-eth0

echo "==> Cleaning up tmp"
/bin/rm -rf /tmp/*
/bin/rm -rf /var/tmp/*

echo "==> Remove the SSH host keys"
/bin/rm -f /etc/ssh/*key*

echo "==> Remove the root user’s shell history"
/bin/rm -f ~root/.bash_history
unset HISTFILE

echo "==> yum -y clean all"
yum -y clean all

# Determine the version of RHEL
COND=`grep -i Taroon /etc/redhat-release`
if [ "$COND" = "" ]; then
        export PREFIX="/usr/sbin"
else
        export PREFIX="/sbin"
fi

FileSystem=`grep ext /etc/mtab| awk -F" " '{ print $2 }'`

for i in $FileSystem
do
        echo $i
        number=`df -B 512 $i | awk -F" " '{print $3}' | grep -v Used`
        echo $number
        percent=$(echo "scale=0; $number * 98 / 100" | bc )
        echo $percent
        dd count=`echo $percent` if=/dev/zero of=`echo $i`/zf
        /bin/sync
        sleep 15
        rm -f $i/zf
done

VolumeGroup=`$PREFIX/vgdisplay | grep Name | awk -F" " '{ print $3 }'`

for j in $VolumeGroup
do
        echo $j
        $PREFIX/lvcreate -l `$PREFIX/vgdisplay $j | grep Free | awk -F" " '{ print $5 }'` -n zero $j
        if [ -a /dev/$j/zero ]; then
                cat /dev/zero > /dev/$j/zero
                /bin/sync
                sleep 15
                $PREFIX/lvremove -f /dev/$j/zero
        fi
done

echo "==> Zero out the free space to save space in the final image"
dd if=/dev/zero of=/EMPTY bs=1M
rm -f /EMPTY

# Make sure we wait until all the data is written to disk, otherwise
# Packer might quit too early before the large files are deleted
sync
[root@packer-centos packer-templates]#
{{< \/highlight >}}

### 9. Create packer-remote-info.json file that contains info on our ESXi virtual machine

Create the following file that will contain the ESXi virtual machine specific information used by Packer. We will also add this file to .gitignore in a future step, to ensure this information does not get stored in a git repository.

{{< highlight bash >}}
[root@packer-centos packer-templates]# cat << EOF > ~/packer-templates/packer-remote-info.json
> {
>   "packer_remote_host": "192.168.1.51",
>   "packer_remote_username": "root",
>   "packer_remote_password": "password",
>   "packer_remote_datastore": "datastore1",
>   "packer_remote_network": "VM Network"
> }
> EOF
[root@packer-centos packer-templates]#
{{< /highlight >}}

### 10. Use yum to install the tree command so we can visualize the packer-templates directory structure

The tree command is helpful by allowing you to display the files and folders from a specific point in the file system. We will install this now to assist in showing the directory structure of the packer-templates directory.

{{< highlight bash >}}
[root@packer-centos packer-templates]# yum install -y tree
Loaded plugins: fastestmirror
Setting up Install Process
Loading mirror speeds from cached hostfile
 * base: yum.tamu.edu
 * epel: mirror.utexas.edu
 * extras: centos.firehosted.com
 * updates: mirrors.adams.net
Resolving Dependencies
--> Running transaction check
---> Package tree.x86_64 0:1.5.3-3.el6 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

==========================================================
 Package      Arch      Version         Repository   Size
==========================================================
Installing:
 tree         x86_64    1.5.3-3.el6     base         36 k

Transaction Summary
==========================================================
Install       1 Package(s)

Total download size: 36 k
Installed size: 65 k
Downloading Packages:
tree-1.5.3-3.el6.x86_64.rpm              |  36 kB    00:00
Running rpm_check_debug
Running Transaction Test
Transaction Test Succeeded
Running Transaction
  Installing : tree-1.5.3-3.el6.x86_64                 1/1
  Verifying  : tree-1.5.3-3.el6.x86_64                 1/1

Installed:
  tree.x86_64 0:1.5.3-3.el6

Complete!
[root@packer-centos packer-templates]#
{{< /highlight >}}

Run the tree command to see the directory structure:

{{< highlight bash >}}
[root@packer-centos packer-templates]# tree
.
├── iso
│   ├── CentOS-6.7-x86_64-minimal.iso
│   ├── linux.iso
│   └── vmware-tools-linux.iso -> ./linux.iso
├── packer-remote-info.json
├── scripts
│   ├── centos-6-kickstart.cfg
│   ├── centos-vagrant-settings.sh
│   ├── centos-vmware-cleanup.sh
│   └── centos-vmware-tools_install.sh
└── templates
    └── centos67.json

3 directories, 9 files
[root@packer-centos packer-templates]#
{{< /highlight >}}

### 11. Create .gitignore file that contains everything we don't have to stored in git
Any files or folders listed in the .gitignore will be ignored by git. We will be adding the packer-remote-info.json file, since it contains username and password information, and the iso/ directory, since we don't want to upload the large .iso files to our git repository.

{{< highlight bash >}}
[root@packer-centos packer-templates]# cat .gitignore
iso/
packer-remote-info.json
[root@packer-centos packer-templates]# git status
# On branch master
#
# Initial commit
#
# Untracked files:
#   (use "git add <file>..." to include in what will be committed)
#
#	.gitignore
#	scripts/
#	templates/
nothing added to commit but untracked files present (use "git add" to track)
[root@packer-centos packer-templates]#
{{< /highlight >}}

### 12. Push the changes we have made to a github repository
I created a public github repository for storing all the files created during this Packer series. During this step I will be pushing the current state of the files to the ["first-packer-template" branch](https://github.com/sdorsett/packer-templates/tree/first-packer-template) of this github repository:

{{< highlight bash >}}
[root@packer-centos packer-templates]# git remote add origin https://sdorsett@github.com/sdorsett/packer-templates.git
[root@packer-centos packer-templates]# git add .
[root@packer-centos packer-templates]# git commit -m "first-packer-template"
[master (root-commit) 3fba017] first-packer-template
 6 files changed, 277 insertions(+), 0 deletions(-)
 create mode 100644 .gitignore
 create mode 100644 scripts/centos-6-kickstart.cfg
 create mode 100644 scripts/centos-vagrant-settings.sh
 create mode 100644 scripts/centos-vmware-cleanup.sh
 create mode 100644 scripts/centos-vmware-tools_install.sh
 create mode 100644 templates/centos67.json
[root@packer-centos packer-templates]# git checkout -b first-packer-template
Switched to a new branch 'first-packer-template'
[root@packer-centos packer-templates]# git push origin first-packer-template
Password:
Counting objects: 10, done.
Delta compression using up to 2 threads.
Compressing objects: 100% (8/8), done.
Writing objects: 100% (10/10), 3.90 KiB, done.
Total 10 (delta 0), reused 0 (delta 0)
To https://sdorsett@github.com/sdorsett/packer-templates.git
 * [new branch]      first-packer-template -> first-packer-template
[root@packer-centos packer-templates]#
{{< /highlight >}}

### 13. packer build the template we created
Now that we have all the necessary .iso files downloaded and template/script files created, we can test our build. In order to use the values in the packer-remote-info.json file, we will use the -var-file parameter to specify this file.

{{< highlight bash >}}
[root@packer-centos packer-templates]# packer build -var-file=./packer-remote-info.json templates/centos67.json
centos67 output will be in this color.

==> centos67: Downloading or copying ISO
    centos67: Downloading or copying: file:///root/packer-templates/iso/CentOS-6.7-x86_64-minimal.iso
==> centos67: Uploading ISO to remote machine...
==> centos67: Creating virtual machine disk
==> centos67: Building and writing VMX file
==> centos67: Starting HTTP server on port 8448
==> centos67: Registering remote VM...
==> centos67: Starting virtual machine...
==> centos67: Waiting 5s for boot...
==> centos67: Connecting to VM via VNC
==> centos67: Typing the boot command over VNC...
==> centos67: Waiting for SSH to become available...
{{< /highlight >}}

At this point you should see a centos67 virtual machine in the embedded host client of the ESXi virtual machine.

![screenshot](/static/01-centos67-virtual-machine.png)  

If you open the console of the centos67 virtual machine, you can watch the automated install as it progresses.

![screenshot](/static/02-centos67-automated-install.png)  

![screenshot](/static/03-centos67-automated-install.png)  

![screenshot](/static/04-centos67-automated-install.png)  

![screenshot](/static/05-centos67-install-completed.png)  

Once the CentOS 6.7 install completes and the virtual machine reboots, you will see the packer build output continue with the provisioners block of the Packer template.

{{< highlight bash >}}
==> centos67: Connected to SSH!
{{< /highlight >}}

The first provisioner is the file provisioner that will copy iso/vmware-tools-linux.iso to /tmp/vmware-tools-linux.iso within the CentOS 6.7 virtual machine.

{{< highlight bash >}}
==> centos67: Uploading iso/vmware-tools-linux.iso => /tmp/vmware-tools-linux.iso
{{< /highlight >}}

The second provisioner is a shell provisioner that will run the scripts/centos-vagrant-settings.sh bash script. This script will added the necessary changes for this Packer image to be used by Vagrant.

{{< highlight bash >}}

==> centos67: Provisioning with shell script: scripts/centos-vagrant-settings.sh

{{< /highlight >}}

The third provisioner is a shell provisioner that will run the scripts/centos-vmware-tools_install.sh bash script. This script will install all the needed dependencies for vmtools and then install vmtools using an answer file.

{{< highlight bash >}}

==> centos67: Provisioning with shell script: scripts/centos-vmware-tools_install.sh
    centos67: Loaded plugins: fastestmirror
    centos67: Setting up Upgrade Process
    centos67: Determining fastest mirrors
    centos67: * base: dallas.tx.mirror.xygenhosting.com
    centos67: * extras: mirror.us.oneandone.net
    centos67: * updates: dallas.tx.mirror.xygenhosting.com
    centos67: No Packages marked for Update
    centos67: Loaded plugins: fastestmirror
    centos67: Setting up Install Process
    centos67: Loading mirror speeds from cached hostfile
    centos67: * base: dallas.tx.mirror.xygenhosting.com
    centos67: * extras: mirror.us.oneandone.net
    centos67: * updates: dallas.tx.mirror.xygenhosting.com
    centos67: Resolving Dependencies
    centos67: --> Running transaction check
    centos67: ---> Package fuse-libs.x86_64 0:2.8.3-4.el6 will be installed
    centos67: ---> Package perl.x86_64 4:5.10.1-141.el6_7.1 will be installed
    centos67: --> Processing Dependency: perl-libs = 4:5.10.1-141.el6_7.1 for package: 4:perl-5.10.1-141.el6_7.1.x86_64
    centos67: --> Processing Dependency: perl-libs for package: 4:perl-5.10.1-141.el6_7.1.x86_64
    centos67: --> Processing Dependency: perl(version) for package: 4:perl-5.10.1-141.el6_7.1.x86_64
    centos67: --> Processing Dependency: perl(Pod::Simple) for package: 4:perl-5.10.1-141.el6_7.1.x86_64
    centos67: --> Processing Dependency: perl(Module::Pluggable) for package: 4:perl-5.10.1-141.el6_7.1.x86_64
    centos67: --> Processing Dependency: libperl.so()(64bit) for package: 4:perl-5.10.1-141.el6_7.1.x86_64
    centos67: --> Running transaction check
    centos67: ---> Package perl-Module-Pluggable.x86_64 1:3.90-141.el6_7.1 will be installed
    centos67: ---> Package perl-Pod-Simple.x86_64 1:3.13-141.el6_7.1 will be installed
    centos67: --> Processing Dependency: perl(Pod::Escapes) >= 1.04 for package: 1:perl-Pod-Simple-3.13-141.el6_7.1.x86_64
    centos67: ---> Package perl-libs.x86_64 4:5.10.1-141.el6_7.1 will be installed
    centos67: ---> Package perl-version.x86_64 3:0.77-141.el6_7.1 will be installed
    centos67: --> Running transaction check
    centos67: ---> Package perl-Pod-Escapes.x86_64 1:1.04-141.el6_7.1 will be installed
    centos67: --> Finished Dependency Resolution
    centos67:
    centos67: Dependencies Resolved
    centos67:
    centos67: ========================================
    centos67: Package         Arch   Version
    centos67: Repository
    centos67: Size
    centos67: ========================================
    centos67: Installing:
    centos67: fuse-libs       x86_64 2.8.3-4.el6
    centos67: base     74 k
    centos67: perl            x86_64 4:5.10.1-141.el6_7.1
    centos67: updates  10 M
    centos67: Installing for dependencies:
    centos67: perl-Module-Pluggable
    centos67: x86_64 1:3.90-141.el6_7.1
    centos67: updates  40 k
    centos67: perl-Pod-Escapes
    centos67: x86_64 1:1.04-141.el6_7.1
    centos67: updates  33 k
    centos67: perl-Pod-Simple x86_64 1:3.13-141.el6_7.1
    centos67: updates 213 k
    centos67: perl-libs       x86_64 4:5.10.1-141.el6_7.1
    centos67: updates 579 k
    centos67: perl-version    x86_64 3:0.77-141.el6_7.1
    centos67: updates  52 k
    centos67:
    centos67: Transaction Summary
    centos67: ========================================
    centos67: Install       7 Package(s)
    centos67:
    centos67: Total size: 11 M
    centos67: Total download size: 74 k
    centos67: Installed size: 36 M
    centos67: Downloading Packages:
    centos67: fuse-libs-2.8.3- |  74 kB     00:00
    centos67: warning: rpmts_HdrFromFdno: Header V3 RSA/SHA1 Signature, key ID c105b9de: NOKEY
    centos67: Retrieving key from file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-6
    centos67: Importing GPG key 0xC105B9DE:
    centos67: Userid : CentOS-6 Key (CentOS 6 Official Signing Key) <centos-6-key@centos.org>
    centos67: Package: centos-release-6-7.el6.centos.12.3.x86_64 (@anaconda-CentOS-201508042137.x86_64/6.7)
    centos67: From   : /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-6
    centos67: Running rpm_check_debug
    centos67: Running Transaction Test
    centos67: Transaction Test Succeeded
    centos67: Running Transaction
    centos67:   Installing : 1:perl-Pod-Escapes   1/7
    centos67:   Installing : 1:perl-Module-Plug   2/7
    centos67:   Installing : 3:perl-version-0.7   3/7
    centos67:   Installing : 4:perl-libs-5.10.1   4/7
    centos67:   Installing : 1:perl-Pod-Simple-   5/7
    centos67:   Installing : 4:perl-5.10.1-141.   6/7
    centos67:   Installing : fuse-libs-2.8.3-4.   7/7
    centos67: Verifying  : 1:perl-Pod-Simple-   1/7
    centos67: Verifying  : 1:perl-Pod-Escapes   2/7
    centos67: Verifying  : fuse-libs-2.8.3-4.   3/7
    centos67: Verifying  : 1:perl-Module-Plug   4/7
    centos67: Verifying  : 3:perl-version-0.7   5/7
    centos67: Verifying  : 4:perl-libs-5.10.1   6/7
    centos67: Verifying  : 4:perl-5.10.1-141.   7/7
    centos67:
    centos67: Installed:
    centos67: fuse-libs.x86_64 0:2.8.3-4.el6
    centos67: perl.x86_64 4:5.10.1-141.el6_7.1
    centos67:
    centos67: Dependency Installed:
    centos67: perl-Module-Pluggable.x86_64 1:3.90-141.el6_7.1
    centos67: perl-Pod-Escapes.x86_64 1:1.04-141.el6_7.1
    centos67: perl-Pod-Simple.x86_64 1:3.13-141.el6_7.1
    centos67: perl-libs.x86_64 4:5.10.1-141.el6_7.1
    centos67: perl-version.x86_64 3:0.77-141.el6_7.1
    centos67:
    centos67: Complete!
    centos67: total 488
    centos67: drwxr-xr-x   7 root root   4096 Oct 30 07:10 .
    centos67: drwxrwxrwt.  4 root root   4096 Dec 24 02:34 ..
    centos67: drwxr-xr-x   2 root root   4096 Oct 30 07:10 bin
    centos67: drwxr-xr-x   2 root root   4096 Oct 30 07:10 doc
    centos67: drwxr-xr-x   5 root root   4096 Oct 30 07:10 etc
    centos67: -rw-r--r--   1 root root 269196 Oct 30 07:10 FILES
    centos67: -rw-r--r--   1 root root   2538 Oct 30 07:10 INSTALL
    centos67: drwxr-xr-x   2 root root   4096 Oct 30 07:10 installer
    centos67: drwxr-xr-x  15 root root   4096 Oct 30 07:10 lib
    centos67: -rwxr-xr-x   1 root root 196237 Oct 30 07:10 vmware-install.pl
    centos67: Creating a new VMware Tools installer database using the tar4 format.
    centos67:
    centos67: Installing VMware Tools.
    centos67:
    centos67: In which directory do you want to install the binary files?
    centos67: [/usr/bin]
    centos67: The path "yes" is a relative path. Please enter an absolute path.
    centos67:
    centos67: In which directory do you want to install the binary files?
    centos67: [/usr/bin]
    centos67: What is the directory that contains the init directories (rc0.d/ to rc6.d/)?
    centos67: [/etc/rc.d]
    centos67: What is the directory that contains the init scripts?
    centos67: [/etc/init.d]
    centos67: In which directory do you want to install the daemon files?
    centos67: [/usr/sbin]
    centos67: In which directory do you want to install the library files?
    centos67: [/usr/lib/vmware-tools]
    centos67: The path "/usr/lib/vmware-tools" does not exist currently. This program is
    centos67: going to create it, including needed parent directories. Is this what you want?
    centos67: [yes]
    centos67: In which directory do you want to install the documentation files?
    centos67: [/usr/share/doc/vmware-tools]
    centos67: The path "/usr/share/doc/vmware-tools" does not exist currently. This program
    centos67: is going to create it, including needed parent directories. Is this what you
    centos67: want? [yes]
    centos67: The installation of VMware Tools 9.9.4 build-3193940 for Linux completed
    centos67: successfully. You can decide to remove this software from your system at any
    centos67: time by invoking the following command: "/usr/bin/vmware-uninstall-tools.pl".
    centos67:
    centos67: Before running VMware Tools for the first time, you need to configure it by
    centos67: invoking the following command: "/usr/bin/vmware-config-tools.pl". Do you want
    centos67: this program to invoke the command for you now? [yes]
    centos67: Initializing...
    centos67:
    centos67:
    centos67: Making sure services for VMware Tools are stopped.
    centos67:
    centos67:
    centos67:
    centos67: Found a compatible pre-built module for vmci.  Installing it...
    centos67:
    centos67:
    centos67: Found a compatible pre-built module for vsock.  Installing it...
    centos67:
    centos67:
    centos67: The module vmxnet3 has already been installed on this system by another
    centos67: installer or package and will not be modified by this installer.
    centos67:
    centos67: The module pvscsi has already been installed on this system by another
    centos67: installer or package and will not be modified by this installer.
    centos67:
    centos67: The module vmmemctl has already been installed on this system by another
    centos67: installer or package and will not be modified by this installer.
    centos67:
    centos67: The VMware Host-Guest Filesystem allows for shared folders between the host OS
    centos67: and the guest OS in a Fusion or Workstation virtual environment.  Do you wish
    centos67: to enable this feature? [no]
    centos67: Found a compatible pre-built module for vmhgfs.  Installing it...
    centos67:
    centos67:
    centos67: Found a compatible pre-built module for vmxnet.  Installing it...
    centos67:
    centos67:
    centos67: The vmblock enables dragging or copying files between host and guest in a
    centos67: Fusion or Workstation virtual environment.  Do you wish to enable this feature?
    centos67: [no]
    centos67: VMware automatic kernel modules enables automatic building and installation of
    centos67: VMware kernel modules at boot that are not already present. This feature can be
    centos67:
    centos67: enabled/disabled by re-running vmware-config-tools.pl.
    centos67:
    centos67: Would you like to enable VMware automatic kernel modules?
    centos67: [no]
    centos67: Do you want to enable Guest Authentication (vgauth)? [yes]
    centos67: No X install found.
    centos67:
    centos67: Creating a new initrd boot image for the kernel.
    centos67: vmware-tools-thinprint start/running
    centos67: vmware-tools start/running
    centos67: The configuration of VMware Tools 9.9.4 build-3193940 for Linux for this
    centos67: running kernel completed successfully.
    centos67:
    centos67: You must restart your X session before any mouse or graphics changes take
    centos67: effect.
    centos67:
    centos67: You can now run VMware Tools by invoking "/usr/bin/vmware-toolbox-cmd" from the
    centos67: command line.
    centos67:
    centos67: To enable advanced X features (e.g., guest resolution fit, drag and drop, and
    centos67: file and text copy/paste), you will need to do one (or more) of the following:
    centos67: 1. Manually start /usr/bin/vmware-user
    centos67: 2. Log out and log back into your desktop session; and,
    centos67: 3. Restart your X session.
    centos67:
    centos67: Enjoy,
    centos67:
    centos67: --the VMware team
    centos67:
{{< /highlight >}}

The final provisioner is a shell provisioner that will run the scripts/centos-vmware-cleanup.sh bash script. This script will clean up the centos67 virtual machine and zero out all unused disk space to reduce the size of the image.

{{< highlight bash >}}
==> centos67: Provisioning with shell script: scripts/centos-vmware-cleanup.sh
    centos67: ==> Pausing for 0 seconds...
    centos67: ==> erasing unused packages to free up space
    centos67: Loaded plugins: fastestmirror
    centos67: Setting up Remove Process
    centos67: No Match for argument: gtk2
    centos67: Loading mirror speeds from cached hostfile
    centos67: * base: dallas.tx.mirror.xygenhosting.com
    centos67: * extras: mirror.us.oneandone.net
    centos67: * updates: dallas.tx.mirror.xygenhosting.com
    centos67: Package(s) gtk2 available, but not installed.
    centos67: No Match for argument: libX11
    centos67: Package(s) libX11 available, but not installed.
    centos67: No Match for argument: hicolor-icon-theme
    centos67: Package(s) hicolor-icon-theme available, but not installed.
    centos67: No Match for argument: avahi
    centos67: Package(s) avahi available, but not installed.
    centos67: No Match for argument: freetype
    centos67: Package(s) freetype available, but not installed.
    centos67: No Match for argument: bitstream-vera-fonts
    centos67: No Packages marked for removal
    centos67: ==> Cleaning up yum cache
    centos67: Loaded plugins: fastestmirror
    centos67: Cleaning repos: base extras updates
    centos67: Cleaning up Everything
    centos67: Cleaning up list of fastest mirrors
    centos67: ==> Force logs to rotate
    centos67: ==> Clear audit log and wtmp
    centos67: ==> Cleaning up udev rules
    centos67: ==> Remove the traces of the template MAC address and UUIDs
    centos67: ==> Cleaning up tmp
    centos67: ==> Remove the SSH host keys
    centos67: ==> Remove the root user’s shell history
    centos67: ==> yum -y clean all
    centos67: Loaded plugins: fastestmirror
    centos67: Cleaning repos: base extras updates
    centos67: Cleaning up Everything
    centos67: /
    centos67: 25211696
    centos67: /tmp/script_7943.sh: line 62: bc: command not found
    centos67:
    centos67: dd: invalid number `'
    centos67: /boot
    centos67: 60474
    centos67: /tmp/script_7943.sh: line 62: bc: command not found
    centos67:
    centos67: dd: invalid number `'
    centos67: /tmp/script_7943.sh: line 70: /usr/sbin/vgdisplay: No such file or directory
    centos67: ==> Zero out the free space to save space in the final image
    centos67: dd: writing `/EMPTY': No space left on device
    centos67: 12970+0 records in
    centos67: 12969+0 records out
    centos67: 13599232000 bytes (14 GB) copied, 413.4 s, 32.9 MB/s
{{< /highlight >}}

With the provisioners block having been completed, Packer will now shutdown the centos67 virtual machine and unregister it from the ESXi virtual machine.

{{< highlight bash >}}
==> centos67: Gracefully halting virtual machine...
    centos67: Waiting for VMware to clean up after itself...
==> centos67: Deleting unnecessary VMware files...
    centos67: Deleting: /vmfs/volumes/datastore1/output-centos67/vmware.log
==> centos67: Cleaning VMX prior to finishing up...
    centos67: Unmounting floppy from VMX...
    centos67: Detaching ISO from CD-ROM device...
    centos67: Disabling VNC server...
==> centos67: Compacting the disk image
==> centos67: Unregistering virtual machine...
Build 'centos67' finished.

==> Builds finished. The artifacts of successful builds are:
--> centos67: VM files in directory: /vmfs/volumes/datastore1/output-centos67
[root@packer-centos packer-templates]#
{{< /highlight >}}

The output of the packer build command shows that the template was successfully created at /vmfs/volumes/datastore1/output-centos67 on the ESXi virtual machine.

#### This seems like a good point to take break. We covered a lot of ground in this post, but as a result of all our work we have a working Packer template to build on. ####

#### If you would like to not have to manually create the files covered in this post, you can clone down [this github repository](https://github.com/sdorsett/packer-templates/tree/first-packer-template) by running the following command: ####

{{< highlight bash >}}
git clone -b "first-packer-template" https://github.com/sdorsett/packer-templates.git
{{< /highlight >}}  

#### You will still need to create the iso/ directory, download the needed .iso files and create the packer-remote-info.json file as covered in this post.  ####

#### In the next post we will extend what was covered in this post by installing the Puppet agent in Packer image. ####

---

#### Please provide any feedback or suggestions to my twitter account located on the about page. ####
