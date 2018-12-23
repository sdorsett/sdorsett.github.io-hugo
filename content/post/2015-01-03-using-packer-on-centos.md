+++
title = "Using packer on CentOS 6.5 to create an ESXi .box template for vagrant deployment"
description = ""
tags = [
    "packer",
    "vsphere",
]
date = "2015-01-03"
categories = [
    "packer",
    "vsphere",
]
topics = [
    "using-packer-with-esxi"
]
highlight = "true"
+++

In the previous post I demonstrated using packer to create a ESXi .box template on OS X with fusion and the vagrant vmware provider. Both of these pieces of software have a cost associated with their usage, so in this post I will demonstrate how to use CentOS 6.5 and ESXi for the same results.  

In this post we will again talk about two helpful gosddc projects:

* [gosddc/packer-post-processor-vagrant-vmware-ovf](https://github.com/gosddc/packer-post-processor-vagrant-vmware-ovf).
   This repo contains a packer post processor that leverages VMware OVF Tool to create a vmware_ovf Vagrant box that is compatible with vagrant-vcloud, vagrant-vcenter and vagrant-vcloudair vagrant providers. It is only compatible with the packer VMware builder. This project is a post processor that makes the generation of the .box file seemless. Unfortunately there currently seems to be [a bug with packer](https://github.com/mitchellh/packer/issues/1457) exporting the virtual machine artifact from a remote ESXi server. Hopefully this issue will be resolved with an upcoming release of packer which will allow this post processor to be used with remote ESXi.

* [gosddc/packer-templates](https://github.com/gosddc/packer-templates).
   This repo contains Packer templates for boxes available at https://vagrantcloud.com/gosddc, they only work with VMware and require the packer-post-processor-vagrant-vmware-ovf post-processor to work. These templates are a good starting point for generating packer templates on VMware products.

We will be using a virtual machine with a minimal install of CentOS 6.5 to install everything we need to build the packer template.

## 1. Let's get started by installing packer.
* Copy the linux 64bit packer installer URL from the following link:

  [https://www.packer.io/downloads.html](https://www.packer.io/downloads.html)

* Use wget on your CentOS vm to download the URL you copied 

{{< highlight bash >}}
[root@vagrant ~]# wget https://dl.bintray.com/mitchellh/packer/packer_0.7.5_linux_amd64.zip
--2015-01-02 21:17:41-- https://dl.bintray.com/mitchellh/packer/packer_0.7.5_linux_amd64.zip
Resolving dl.bintray.com... 108.168.194.91, 108.168.194.92
Connecting to dl.bintray.com|108.168.194.91|:443... connected.
HTTP request sent, awaiting response... 302 
Resolving d29vzk4ow07wi7.cloudfront.net... 54.230.5.16, 54.230.5.11, 54.230.5.30, ...
Connecting to d29vzk4ow07wi7.cloudfront.net|54.230.5.16|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 87262135 (83M) [application/unknown]
Saving to: "packer_0.7.5_linux_amd64.zip"

100%[==================================================================================================>] 87,262,135 1.92M/s in 50s 

2015-01-02 21:18:42 (1.65 MB/s) - "packer_0.7.5_linux_amd64.zip" saved [87262135/87262135]
{{< /highlight >}}

* Install the unzip package on our CentOS vm since that's the format packer comes compressed in:

{{< highlight bash >}}
[root@vagrant ~]# yum install -y unzip
Loaded plugins: fastestmirror
Setting up Install Process
Loading mirror speeds from cached hostfile
 * base: repos.dfw.quadranet.com
 * extras: mirror.anl.gov
 * updates: mirror.us.oneandone.net
base                                                                                                                 | 3.7 kB     00:00     
extras                                                                                                               | 3.4 kB     00:00     
updates                                                                                                              | 3.4 kB     00:00     
Resolving Dependencies
--> Running transaction check
---> Package unzip.x86_64 0:6.0-1.el6 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

============================================================================================================================================
 Package                         Arch                             Version                              Repository                      Size
============================================================================================================================================
Installing:
 unzip                           x86_64                           6.0-1.el6                            base                           149 k

Transaction Summary
============================================================================================================================================
Install       1 Package(s)

Total download size: 149 k
Installed size: 313 k
Downloading Packages:
unzip-6.0-1.el6.x86_64.rpm                                                                                           | 149 kB     00:00     
Running rpm_check_debug
Running Transaction Test
Transaction Test Succeeded
Running Transaction
  Installing : unzip-6.0-1.el6.x86_64                                                                                                   1/1 
  Verifying  : unzip-6.0-1.el6.x86_64                                                                                                   1/1 

Installed:
  unzip.x86_64 0:6.0-1.el6                                                                                                                  

Complete!
[root@vagrant ~]# 
{{< /highlight >}}

* Create the /usr/local/packer_7.5 directory and unzip the packer download to this location. 

{{< highlight bash >}}
[root@vagrant ~]# mkdir /usr/local/packer_7.5
[root@vagrant ~]# cp packer_0.7.5_linux_amd64.zip /usr/local/packer_7.5/
[root@vagrant ~]# cd /usr/local/packer_7.5/
[root@vagrant packer_7.5]# unzip packer_0.7.5_linux_amd64.zip
Archive: packer_0.7.5_linux_amd64.zip
inflating: packer 
inflating: packer-builder-amazon-chroot 
inflating: packer-builder-amazon-ebs 
inflating: packer-builder-amazon-instance 
inflating: packer-builder-digitalocean 
inflating: packer-builder-docker 
inflating: packer-builder-googlecompute 
inflating: packer-builder-null 
inflating: packer-builder-openstack 
inflating: packer-builder-parallels-iso 
inflating: packer-builder-parallels-pvm 
inflating: packer-builder-qemu 
inflating: packer-builder-virtualbox-iso 
inflating: packer-builder-virtualbox-ovf 
inflating: packer-builder-vmware-iso 
inflating: packer-builder-vmware-vmx 
inflating: packer-post-processor-atlas 
inflating: packer-post-processor-compress 
inflating: packer-post-processor-docker-import 
inflating: packer-post-processor-docker-push 
inflating: packer-post-processor-docker-save 
inflating: packer-post-processor-docker-tag 
inflating: packer-post-processor-vagrant 
inflating: packer-post-processor-vagrant-cloud 
inflating: packer-post-processor-vsphere 
inflating: packer-provisioner-ansible-local 
inflating: packer-provisioner-chef-client 
inflating: packer-provisioner-chef-solo 
inflating: packer-provisioner-file 
inflating: packer-provisioner-puppet-masterless 
inflating: packer-provisioner-puppet-server 
inflating: packer-provisioner-salt-masterless 
inflating: packer-provisioner-shell
{{< /highlight >}}


* Add /usr/local/packer_7.5 to your path by running the following command.

{{< highlight bash >}}
[root@vagrant packer_7.5]# export PATH="/usr/local/packer_7.5:$PATH"
{{< /highlight >}}

* Add the export command we just ran into ~/.bash_profile to ensure this path change persists after reboots.

{{< highlight bash >}}
[root@vagrant packer_7.5]# cat ~/.bash_profile
export PATH="/usr/local/packer_7.5:$PATH"
[root@vagrant packer_7.5]# 
{{< /highlight >}}

* Stop iptables since packer will be using a http server to publish the ESXi kickstart files:

{{< highlight bash >}}
[root@vagrant packer_7.5]# service iptables stop
iptables: Setting chains to policy ACCEPT: filter [ OK ]
iptables: Flushing firewall rules: [ OK ]
iptables: Unloading modules: [ OK ]
[root@vagrant packer_7.5]# 
{{< /highlight >}}


## 2. The next thing we need to do is download and install ovftool:

* Download and install the latest version of the VMware OVF tool. VMware-ovftool-3.5.1-1747221-lin.x86_64.bundle is what I used.

{{< highlight bash >}}
[root@vagrant ~]# chmod +x VMware-ovftool-3.5.1-1747221-lin.x86_64.bundle
[root@vagrant ~]# ./VMware-ovftool-3.5.1-1747221-lin.x86_64.bundle
Extracting VMware Installer...done.
You must accept the VMware OVF Tool component for Linux End User
License Agreement to continue. Press Enter to proceed.
VMWARE END USER LICENSE AGREEMENT

1.4 "Intellectual Property Rights" means all worldwide intellectual
property rights, including without limitation, copyrights, trademarks, service
Do you agree? [yes/no]: yes

The product is ready to be installed. Press Enter to begin
installation or Ctrl-C to cancel.

Installing VMware OVF Tool component for Linux 3.5.1
Configuring...
[######################################################################] 100%
Installation was successful.
{{< /highlight >}}

* Verify ovftool is successfully added to your path by running "ovftool -v". This command should output the version of ovftool we installed.

{{< highlight bash >}}
[root@vagrant ~]# ovftool -v
VMware ovftool 3.5.1 (build-1747221)
{{< /highlight >}}

## 3. Now we can download and install the gosddc packer components we will need.

* Download the most recent linux amd64 version of the compiled packer-processor-vagrant-vmware-ovf binary from the following link:

[https://github.com/gosddc/packer-post-processor-vagrant-vmware-ovf/releases](https://github.com/gosddc/packer-post-processor-vagrant-vmware-ovf/releases)

* Unzip packer-post-processor-vagrant-vmware-ovf and copy it to "/usr/local/packer_7.5". Ensure the permissions of this file match the other files in this directory.

* Create a directory to contain the packer templates:

{{< highlight bash >}}
[root@vagrant ~]# mkdir ~/packer
[root@vagrant ~]# cd ~/packer
{{< /highlight >}}

* Clone the gosddc packer-templates repository:

{{< highlight bash >}}
[root@vagrant packer]# git clone https://github.com/gosddc/packer-templates.git
Initialized empty Git repository in /root/packer/packer-templates/.git/
remote: Counting objects: 195, done.
remote: Total 195 (delta 0), reused 0 (delta 0)
Receiving objects: 100% (195/195), 39.32 KiB, done.
Resolving deltas: 100% (105/105), done.
{{< /highlight >}}

## 4. Next we need to download the ESXi 5.5 .iso and copy it into the proper directory location.

* Create an "iso" directory for storing the ESXi iso files:

{{< highlight bash >}}
[root@vagrant packer]# cd packer-templates/
[root@vagrant packer-templates]# mkdir iso
{{< /highlight >}}

* Download and copy the "VMware-VMvisor-Installer-5.5.0-1331820.x86_64.iso" ESXi 5.5 installer to the iso directory.

{{< highlight bash >}}
[root@vagrant packer-templates]# cp ~/VMware-VMvisor-Installer-5.5.0-1331820.x86_64.iso iso/
{{< /highlight >}}

## 5. Finally we will need to modify, validate and build the packer esxi.json packer template we will be using.

* Modify ~/packer/packer-templates/templates/esxi.json to look like the following:

{{< highlight bash >}}
[root@vagrant packer-templates]# vi templates/esxi.json
[root@vagrant packer-templates]# cat templates/esxi.json
{
  "variables": {
  "version": "1.0"
  },
  "builders": [
    {
      "name": "esxi55",
      "vm_name": "esxi55",
      "vmdk_name": "esxi55-disk0",
      "type": "vmware-iso",
      "headless": true,
      "disk_size": 4096,
      "guest_os_type": "centos-64",
      "iso_url": "./iso/VMware-VMvisor-Installer-5.5.0-1331820.x86_64.iso",
      "iso_checksum": "ef599dc7e647177027684c0eee346ccdbc8704f2",
      "iso_checksum_type": "sha1",
      "remote_host": "192.168.1.201",
      "remote_datastore": "esx01-local-sata",
      "remote_username": "root",
      "remote_password": "mySecretP@ssw0rd",
      "remote_type": "esx5",
      "ssh_username": "root",
      "ssh_password": "vagrant",
      "ssh_wait_timeout": "60m",
      "shutdown_command": "esxcli system maintenanceMode set -e true -t 0 ; esxcli system shutdown poweroff -d 10 -r 'Packer Shutdown' ; esxcli system maintenanceMode set -e false -t 0",
      "tools_upload_flavor": "linux",
      "http_directory": ".",
      "boot_wait": "5s",
      "vmx_data": {
        "memsize": "4096",
        "numvcpus": "2",
        "vhv.enable": "TRUE",
        "ethernet0.virtualDev": "e1000",
        "ethernet0.networkName": "vlan2",
        "ethernet0.present": "TRUE"
      },
      "vmx_data_post": {
        "guestos": "vmkernel5",
        "ide1:0.present": "FALSE"
      },
      "boot_command": [
        "<enter><wait>O<wait> ks=http://{% raw %}{{ .HTTPIP }}{% endraw %}:{% raw %}{{ .HTTPPort }}{% endraw %}/scripts/esxi-5-kickstart.cfg<enter>"
      ]
    }
  ],
  "provisioners": [
    {
      "type": "file",
      "source": "puppet/modules/vagrantbaseconfig/files/vagrant.pub",
      "destination": "/etc/ssh/keys-root/authorized_keys"
    },
    {
      "type": "shell",
      "script": "scripts/esxi-vmware-tools_install.sh"
    },
    {
      "type": "shell",
      "script": "scripts/esxi-cloning_configuration.sh"
    }
  ]
}
{{< /highlight >}}

   There are several things I would like to point out in the esxi.json file we just created.

   1. builder - this section specifies that we will be using the "vmware-iso" builder, with a remote ESXi host, to create our packer template. You will need to modify several part of this to match your ESXi server:
     *  remote_host - this is the IP address of our ESXi server.
     *  remote_datastore - this is the datastore our virtual machine will be built on.
     *  remote_username - the ESXi username used to connect
     *  remote_password - the ESXi password used to connect
     *  remote_type - this specifies the remote server is ESXi 5.x
     *  ethernet0.networkName - this is the portgroup that the packer virtual machine connects to. This should be the same portgroup that the CentOS vm is one, since the ESXi install will need a kickstart file served by a http server included with packer.

      Also in the builder section you might notice we are setting the "guest\_os\_type" to be "centos-64" during install. This is due to packer attempting to mount the vmtools .iso at the end of the install process. This has no impact on the install and we change it back to "vmkernel5" in the "vmx\_data\_post" section, which gets processed post after the virtual machine is powered down after the install process has completed.

   2. provisioners - this section specifies we will be using multiple provisioners to modify our template after it has been created:
     * a file provisioner that will copy the vagrant public ssh key into our ESXi template
     * a shell script to install the vmware tools VIB for nested ESXi
     * a shell script to make necessary MAC address changes in our nested ESXi template.
   3. post-processors - I have removed the vagrant-vmware-ovf post processor due to the bug I mentioned earlier.


* Modify the esxi-cloning_configuration.sh script in the scripts directory to read as follows: 

{{< highlight bash >}}
[root@vagrant packer-templates]# vi scripts/esxi-cloning_configuration.sh
[root@vagrant packer-templates]# cat scripts/esxi-cloning_configuration.sh
# Settings to ensure that ESXi cloning goes smooth, thanks @lamw ! see:
# http://www.virtuallyghetto.com/2013/12/how-to-properly-clone-nested-esxi-vm.html
esxcli system settings advanced set -o /Net/FollowHardwareMac -i 1
sed -i '/\/system\/uuid/d' /etc/vmware/esx.conf
sed -i '/\/net\/vmkernelnic\/child\[0000\]\/mac/d' /etc/vmware/esx.conf

# DHCP doesn't refresh correctly upon boot, this will force a renew, it will
# be executed on every boot, if you don't like this behavior you can remove
# the line during the Vagrant provisioning part.
sed -i '/exit 0/d' /etc/rc.local.d/local.sh 
echo 'esxcli network ip interface set -e false -i vmk0; esxcli network ip interface set -e true -i vmk0' >> /etc/rc.local.d/local.sh 
echo 'exit 0' >> /etc/rc.local.d/local.sh 

# Ensure changes are persistent
/sbin/auto-backup.sh
{{< /highlight >}}

We are adding the following line to delete the line that contains the vmkernel port MAC address in /etc/vmware/esx.conf, in order to force it to be regenerated on each clones template:

{{< highlight bash >}}
sed -i '/\/net\/vmkernelnic\/child\[0000\]\/mac/d' /etc/vmware/esx.conf
{{< /highlight >}}

* Validate the packer esxi.json file is ready for building by running the following command:

{{< highlight bash >}}
[root@vagrant packer-templates]# packer validate templates/esxi.json 
Template validated successfully.
{{< /highlight >}}

* Start the build by running:

{{< highlight bash >}}
[root@vagrant packer-templates]# packer build templates/esxi.json 
esxi55 output will be in this color.

==> esxi55: Downloading or copying ISO
esxi55: Downloading or copying: file:///root/packer/packer-templates/iso/VMware-VMvisor-Installer-5.5.0-1331820.x86_64.iso
==> esxi55: Uploading ISO to remote machine...
==> esxi55: Creating virtual machine disk
==> esxi55: Building and writing VMX file
==> esxi55: Starting HTTP server on port 8828
==> esxi55: Registering remote VM...
==> esxi55: Starting virtual machine...
esxi55: The VM will be run headless, without a GUI. If you want to
esxi55: view the screen of the VM, connect via VNC without a password to
esxi55: 192.168.1.201:5988
==> esxi55: Waiting 5s for boot...
==> esxi55: Connecting to VM via VNC
==> esxi55: Typing the boot command over VNC...
==> esxi55: Waiting for SSH to become available...
==> esxi55: Connected to SSH!
==> esxi55: Uploading puppet/modules/vagrantbaseconfig/files/vagrant.pub => /etc/ssh/keys-root/authorized_keys
==> esxi55: Provisioning with shell script: scripts/esxi-vmware-tools_install.sh
esxi55: Installation Result
esxi55: Message: The update completed successfully, but the system needs to be rebooted for the changes to be effective.
esxi55: Reboot Required: true
esxi55: VIBs Installed: VMware_bootbank_esx-tools-for-esxi_9.7.0-0.0.00000
esxi55: VIBs Removed:
esxi55: VIBs Skipped:
==> esxi55: Provisioning with shell script: scripts/esxi-cloning_configuration.sh
esxi55: diff: can't stat '/tmp/auto-backup.35224//etc/ssh/keys-root/authorized_keys': No such file or directory
esxi55: Saving current state in /bootbank
esxi55: Clock updated.
esxi55: Time: 04:39:41 Date: 01/03/2015 UTC
==> esxi55: Gracefully halting virtual machine...
esxi55: Waiting for VMware to clean up after itself...
==> esxi55: Deleting unnecessary VMware files...
esxi55: Deleting: /vmfs/volumes/esx01-local-sata/output-esxi55/vmware.log
==> esxi55: Cleaning VMX prior to finishing up...
esxi55: Unmounting floppy from VMX...
esxi55: Detaching ISO from CD-ROM device...
==> esxi55: Compacting the disk image
==> esxi55: Unregistering virtual machine...
Build 'esxi55' finished.

==> Builds finished. The artifacts of successful builds are:
--> esxi55: VM files in directory: /vmfs/volumes/esx01-local-sata/output-esxi55
[root@vagrant packer-templates]# 

{{< /highlight >}}

If you want to keep an eye on the build process you can connect a VNC client to the address and port listed during the packer build process. For my build this was what was displayed:

{{< highlight bash >}}
esxi55: The VM will be run headless, without a GUI. If you want to
esxi55: view the screen of the VM, connect via VNC without a password to
esxi55: 192168.1.201:5988
{{< /highlight >}}

The other option to track the progress of the ESXi install is to open the console of the virtual machine in the VI or web client.

* The last line of the packer output shows the datastore location of the virtual machine that was created. We need to SSH to the ESXi host in order to re-register the virtual machine, in order to export it using ovftool:

{{< highlight bash >}}
[root@vagrant ~]# ssh root@192.168.1.201 "vim-cmd solo/registervm /vmfs/volumes/esx01-local-sata/output-esxi55/*.vmx"
Password: 
70
{{< /highlight >}}

Update this command to reflect the IP address and path to the virtual machine that packer created. Also take note of the vmid that the command returns, since we'll need this in a future step.

* Use the following command to export the virtual machine we just registered:

{{< highlight bash >}}
[root@vagrant ~]# ovftool vi://root:[esxi_password]@192.168.1.201/esxi55 ./
Opening VI source: vi://root@192.168.1.201:443/esxi55
Opening OVF target: ./
Writing OVF package: ./esxi55/esxi55.ovf
Transfer Completed 
Completed successfully
{{< /highlight >}}

Replace the IP address, password and virtual machine name if needed.

* Unregister the virtual machine template using the following command with the vmid returned by the registration command:

{{< highlight bash >}}
[root@vagrant ~]# ssh root@192.168.1.201 "vim-cmd vmsvc/unregister 70"
Password:
{{< /highlight >}}


## 6. Since we disable the vagrant-vcenter-ovf post processor, we will need to create the .box file according to the specifications[ listed here](https://github.com/gosddc/packer-post-processor-vagrant-vmware-ovf/wiki/vmware_ovf-Box-Format):

* Create the metadata.json file needed by the vagrant-vmware-ovf .box file:

{{< highlight bash >}}
[root@vagrant esxi55]# echo '{"provider":"vmware_ovf"}' >> metadata.json
[root@vagrant esxi55]# 
{{< /highlight >}}

* Create an empty Vagrantfile to include in the .box file:

{{< highlight bash >}}
[root@vagrant esxi55]# touch Vagrantfile
[root@vagrant esxi55]#
{{< /highlight >}}

* Use tar to compress the files in this directory into a .box file:

{{< highlight bash >}}
[root@vagrant esxi55]# tar cvzf esxi55-vmware_ovf-1.1.box ./*
./esxi55-disk1.vmdk
./esxi55.mf
./esxi55.ovf
./metadata.json
./Vagrantfile
[root@vagrant esxi55]# ls -la
total 968296
drwxr-xr-x. 2 root root      4096 Jan  3 15:36 .
drwxr-xr-x. 9 root root      4096 Jan  3 20:12 ..
-rw-r--r--. 1 root root 496975360 Jan  3 15:32 esxi55-disk1.vmdk
-rw-r--r--. 1 root root       125 Jan  3 15:32 esxi55.mf
-rw-r--r--. 1 root root      7522 Jan  3 15:32 esxi55.ovf
-rw-r--r--. 1 root root 494527542 Jan  3 15:36 esxi55-vmware_ovf-1.1.box
-rw-r--r--. 1 root root        26 Jan  3 15:34 metadata.json
-rw-r--r--. 1 root root         0 Jan  3 15:34 Vagrantfile
[root@vagrant esxi55]# 
{{< /highlight >}}

#### That all for this post covering the basics of getting packer and getting it installed/configured on CentOS. In the next blog post I'm planning  on covering how to install vagrant on CentOS and using it to deploy this .box templates to vCenter.

---

#### Please provide any feedback or suggestions to my twitter account located on the about page.

