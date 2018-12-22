+++
title = "Copying our existing CentOS 6.7 template and adding the Puppet agent"
description = ""
tags = [
    "vagrant",
    "packer",
]
date = "2015-12-26"
categories = [
    "vagrant",
    "packer",
]
highlight = "true"
+++

This is the fifth in a series of posts on <a href="../2015-12-22-pipeline-for-creating-packer-box-files">using a Packer pipeline to generate Vagrant .box files</a>.

In the last post we covered <a href="../2015-12-25-creating-a-packer-template-for-installing-centos-67">creating a Packer template for installing CentOS 6.7 with vmtools</a>. In this post we will be basing a new Packer template on the one we created last post and add installing the Puppet Enterprise 3.8.2 agent.

This is an older version of the Puppet Enterprise agent, but it will let us create a Vagrant .box file that can be used for testing Puppet 3.x code.

Let's get started...

### 1. Start off by connecting by SSH to CentOS virtual machine we have been using in previous posts.

{{< highlight bash >}}
sdorsett-mbp:~ sdorsett$ ssh root@192.168.1.52
root@192.168.1.52's password:
Last login: Fri Dec 25 20:22:44 2015 from 192.168.1.253
[root@packer-centos ~]#
{{< /highlight >}}

### 2. Create a new bash script named centos-install-pe-puppet-382.sh at packer-templates/scripts/ to install the Puppet Enterprise 3.8.2 agent:

{{< highlight bash >}}
[root@packer-centos ~]# cd ~/packer-templates/scripts/
[root@packer-centos scripts]# cat centos-install-pe-puppet-382.sh
#!/bin/bash

VERSION='3.8.2'
PKGNAME="puppet-enterprise-${VERSION}-el-6-x86_64"
TARFILE="${PKGNAME}.tar.gz"
URL="https://pm.puppetlabs.com/puppet-enterprise/${VERSION}/${TARFILE}"

TMPDIR='/tmp'
HOSTNAME=`hostname`
cd $TMPDIR
echo "Fetching ${TARFILE}"
[ -e $TARFILE ] || curl -fLO $URL
echo "Extracting ${TARFILE}"
[ -d $PKGNAME ] || tar -xf $TARFILE
cd $PKGNAME

cat > agent.ans <<EOF
q_all_in_one_install=n
q_database_install=n
q_pe_database=n
q_puppetca_install=n
q_puppetdb_install=n
q_puppetmaster_install=n
q_puppet_enterpriseconsole_install=n
q_run_updtvpkg=n
q_continue_or_reenter_master_hostname=c
q_fail_on_unsuccessful_master_lookup=n
q_puppetagent_certname=$HOSTNAME
q_puppetagent_install=y
q_puppetagent_server=puppet
q_puppet_cloud_install=y
q_puppet_symlinks_install=y
q_vendor_packages_install=y
q_install=y
EOF

./puppet-enterprise-installer -a agent.ans

# Remove certname so the system will use host FQDN
sed -i '/certname =/d' /etc/puppetlabs/puppet/puppet.conf

# Symlink so puppet is in vagrant provisioner PATH
ln -s /opt/puppet/bin/facter /usr/bin/facter
ln -s /opt/puppet/bin/puppet /usr/bin/puppet

[root@packer-deploy scripts]#
{{< /highlight >}}

### 3. Copy the Packer template we created previously, as the starting point for a new CentOS 6.7 template that will have Puppet Enterprise 3.8.2 installed

{{< highlight bash >}}
[root@packer-centos ~]# cd ~/packer-templates/templates/
[root@packer-centos templates]# ls
centos67.json
[root@packer-centos templates]# cp centos67.json centos67-pe-puppet-382.json
[root@packer-centos templates]#
[root@packer-deploy scripts]#
{{< /highlight >}}

### 4. Update the provisioners blocker to include the "scripts/centos-install-pe-puppet-382.sh" bash script.

{{< highlight bash >}}
[root@packer-centos templates]# cat centos67-pe-puppet-382.json
{
  "variables": {
    "version": "1.0"
  },
  "builders": [
    {
      "name": "centos67-pe-puppet-382",
      "vm_name": "centos67-pe-puppet-382",
      "vmdk_name": "centos67-pe-puppet-382",
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

      "remote_host": "{{user `packer_remote_host`}}",
      "remote_datastore": "{{user `packer_remote_datastore`}}",
      "remote_username": "{{user `packer_remote_username`}}",
      "remote_password": "{{user `packer_remote_password`}}",
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
        "ethernet0.networkName": "{{user `packer_remote_network`}}",
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
      "script": "scripts/centos-install-pe-puppet-382.sh"
    },
    {
      "type": "shell",
      "execute_command": "echo 'vagrant' | {{ .Vars }} sudo -E -S sh '{{ .Path }}'",
      "script": "scripts/centos-vmware-cleanup.sh"
    }
  ]
}
{{< /highlight >}}

### 5. The packer-templates directory should now contain the following directories and files:

{{< highlight bash >}}
[root@packer-centos packer-templates]# tree /root/packer-templates/
/root/packer-templates/
├── iso
│   ├── CentOS-6.7-x86_64-minimal.iso
│   ├── linux.iso
│   └── vmware-tools-linux.iso -> ./linux.iso
├── packer_cache
├── packer-remote-info.json
├── README.md
├── scripts
│   ├── centos-6-kickstart.cfg
│   ├── centos-install-pe-puppet-382.sh
│   ├── centos-vagrant-settings.sh
│   ├── centos-vmware-cleanup.sh
│   └── centos-vmware-tools_install.sh
└── templates
    ├── centos67.json
    └── centos67-pe-puppet-382.json

4 directories, 12 files
[root@packer-centos packer-templates]#
{{< /highlight >}}

### 6. Push the changes we have made to a github repository
During this step pushed the two new files to the [adding-second-packer-template branch](https://github.com/sdorsett/packer-templates/tree/adding-second-packer-template) of the github repository I have created for this series of blog posts:

{{< highlight bash >}}
[root@packer-centos packer-templates]# git status
# On branch first-packer-template
# Untracked files:
#   (use "git add <file>..." to include in what will be committed)
#
#	scripts/centos-install-pe-puppet-382.sh
#	templates/centos67-pe-puppet-382.json
nothing added to commit but untracked files present (use "git add" to track)
[root@packer-centos packer-templates]# git add .
[root@packer-centos packer-templates]# git commit -m "adding templates/centos67-pe-puppet-382.json and scripts/centos-install-pe-puppet-382.sh"
[first-packer-template 75bb98e] adding templates/centos67-pe-puppet-382.json and scripts/centos-install-pe-puppet-382.sh
 2 files changed, 125 insertions(+), 0 deletions(-)
 create mode 100644 scripts/centos-install-pe-puppet-382.sh
 create mode 100644 templates/centos67-pe-puppet-382.json
[root@packer-centos packer-templates]# git checkout -b adding-second-packer-template
Switched to a new branch 'adding-second-packer-template'
[root@packer-centos packer-templates]# git push origin adding-second-packer-template
Password:
Counting objects: 9, done.
Delta compression using up to 2 threads.
Compressing objects: 100% (6/6), done.
Writing objects: 100% (6/6), 1.95 KiB, done.
Total 6 (delta 1), reused 0 (delta 0)
To https://sdorsett@github.com/sdorsett/packer-templates.git
 * [new branch]      adding-second-packer-template -> adding-second-packer-template
[root@packer-centos packer-templates]#
{{< /highlight >}}

### 7. Have Packer build the new template we created
Now that we have updated the new template file and created the script file that will install the Puppet Enterprise 3.8.2 agent, we can test our build. In order to use the values in the packer-remote-info.json file, we will use the -var-file parameter to specify this file.

{{< highlight bash >}}
root@packer-centos packer-templates]# packer build -var-file=./packer-remote-info.json templates/centos67-pe-puppet-382.json
centos67-pe-puppet-382 output will be in this color.

==> centos67-pe-puppet-382: Downloading or copying ISO
    centos67-pe-puppet-382: Downloading or copying: file:///root/packer-templates/iso/CentOS-6.7-x86_64-minimal.iso
==> centos67-pe-puppet-382: Uploading ISO to remote machine...
==> centos67-pe-puppet-382: Creating virtual machine disk
==> centos67-pe-puppet-382: Building and writing VMX file
==> centos67-pe-puppet-382: Starting HTTP server on port 8019
==> centos67-pe-puppet-382: Registering remote VM...
==> centos67-pe-puppet-382: Starting virtual machine...
==> centos67-pe-puppet-382: Waiting 7s for boot...
==> centos67-pe-puppet-382: Connecting to VM via VNC
==> centos67-pe-puppet-382: Typing the boot command over VNC...
==> centos67-pe-puppet-382: Waiting for SSH to become available...
{{< /highlight >}}

At this point you should see a centos67 virtual machine in the embedded host client of the ESXi virtual machine.

![screenshot](/static/01-centos-67-pe-puppet-382-virtual-machine.png)  


Once the CentOS 6.7 install completes and the virtual machine reboots, you will see the packer build output continue with the provisioners block of the Packer template.

{{< highlight bash >}}
==> centos67-pe-puppet-382: Connected to SSH!
{{< /highlight >}}

The first provisioner is the file provisioner that will copy iso/vmware-tools-linux.iso to /tmp/vmware-tools-linux.iso within the CentOS 6.7 virtual machine.

{{< highlight bash >}}
==> centos67-pe-puppet-382: Uploading iso/vmware-tools-linux.iso => /tmp/vmware-tools-linux.iso
{{< /highlight >}}

The second provisioner is a shell provisioner that will run the scripts/centos-vagrant-settings.sh bash script. This script will added the necessary changes for this Packer image to be used by Vagrant.

{{< highlight bash >}}

==> centos67-pe-puppet-382: Provisioning with shell script: scripts/centos-vagrant-settings.sh

{{< /highlight >}}

The third provisioner is a shell provisioner that will run the scripts/centos-vmware-tools_install.sh bash script. This script will install all the needed dependencies for vmtools and then install vmtools using an answer file.

{{< highlight bash >}}
==> centos67-pe-puppet-382: Provisioning with shell script: scripts/centos-vmware-tools_install.sh
    centos67-pe-puppet-382: Loaded plugins: fastestmirror
    centos67-pe-puppet-382: Setting up Upgrade Process
    centos67-pe-puppet-382: Determining fastest mirrors
    centos67-pe-puppet-382: * base: centos.unixheads.org
    centos67-pe-puppet-382: * extras: mirror.rackspace.com
    centos67-pe-puppet-382: * updates: centos-mirror.jchost.net
    centos67-pe-puppet-382: No Packages marked for Update
    centos67-pe-puppet-382: Loaded plugins: fastestmirror
    centos67-pe-puppet-382: Setting up Install Process
    centos67-pe-puppet-382: Loading mirror speeds from cached hostfile
    centos67-pe-puppet-382: * base: centos.unixheads.org
    centos67-pe-puppet-382: * extras: mirror.rackspace.com
    centos67-pe-puppet-382: * updates: centos-mirror.jchost.net
    centos67-pe-puppet-382: Package 4:perl-5.10.1-141.el6_7.1.x86_64 already installed and latest version
    centos67-pe-puppet-382: Resolving Dependencies
    centos67-pe-puppet-382: --> Running transaction check
    centos67-pe-puppet-382: ---> Package fuse-libs.x86_64 0:2.8.3-4.el6 will be installed
    centos67-pe-puppet-382: --> Finished Dependency Resolution
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: Dependencies Resolved
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: ========================================
    centos67-pe-puppet-382: Package  Arch   Version     Repository
    centos67-pe-puppet-382: Size
    centos67-pe-puppet-382: ========================================
    centos67-pe-puppet-382: Installing:
    centos67-pe-puppet-382: fuse-libs
    centos67-pe-puppet-382: x86_64 2.8.3-4.el6 base  74 k
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: Transaction Summary
    centos67-pe-puppet-382: ========================================
    centos67-pe-puppet-382: Install       1 Package(s)
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: Total download size: 74 k
    centos67-pe-puppet-382: Installed size: 251 k
    centos67-pe-puppet-382: Downloading Packages:
    centos67-pe-puppet-382: fuse-libs-2.8.3- |  74 kB     00:00
    centos67-pe-puppet-382: Running rpm_check_debug
    centos67-pe-puppet-382: Running Transaction Test
    centos67-pe-puppet-382: Transaction Test Succeeded
    centos67-pe-puppet-382: Running Transaction
    centos67-pe-puppet-382:   Installing : fuse-libs-2.8.3-4.   1/1
    centos67-pe-puppet-382: Verifying  : fuse-libs-2.8.3-4.   1/1
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: Installed:
    centos67-pe-puppet-382: fuse-libs.x86_64 0:2.8.3-4.el6
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: Complete!
    centos67-pe-puppet-382: total 488
    centos67-pe-puppet-382: drwxr-xr-x   7 root root   4096 Oct 30 07:10 .
    centos67-pe-puppet-382: drwxrwxrwt.  4 root root   4096 Dec 26 20:56 ..
    centos67-pe-puppet-382: drwxr-xr-x   2 root root   4096 Oct 30 07:10 bin
    centos67-pe-puppet-382: drwxr-xr-x   2 root root   4096 Oct 30 07:10 doc
    centos67-pe-puppet-382: drwxr-xr-x   5 root root   4096 Oct 30 07:10 etc
    centos67-pe-puppet-382: -rw-r--r--   1 root root 269196 Oct 30 07:10 FILES
    centos67-pe-puppet-382: -rw-r--r--   1 root root   2538 Oct 30 07:10 INSTALL
    centos67-pe-puppet-382: drwxr-xr-x   2 root root   4096 Oct 30 07:10 installer
    centos67-pe-puppet-382: drwxr-xr-x  15 root root   4096 Oct 30 07:10 lib
    centos67-pe-puppet-382: -rwxr-xr-x   1 root root 196237 Oct 30 07:10 vmware-install.pl
    centos67-pe-puppet-382: Creating a new VMware Tools installer database using the tar4 format.
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: Installing VMware Tools.
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: In which directory do you want to install the binary files?
    centos67-pe-puppet-382: [/usr/bin]
    centos67-pe-puppet-382: The path "yes" is a relative path. Please enter an absolute path.
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: In which directory do you want to install the binary files?
    centos67-pe-puppet-382: [/usr/bin]
    centos67-pe-puppet-382: What is the directory that contains the init directories (rc0.d/ to rc6.d/)?
    centos67-pe-puppet-382: [/etc/rc.d]
    centos67-pe-puppet-382: What is the directory that contains the init scripts?
    centos67-pe-puppet-382: [/etc/init.d]
    centos67-pe-puppet-382: In which directory do you want to install the daemon files?
    centos67-pe-puppet-382: [/usr/sbin]
    centos67-pe-puppet-382: In which directory do you want to install the library files?
    centos67-pe-puppet-382: [/usr/lib/vmware-tools]
    centos67-pe-puppet-382: The path "/usr/lib/vmware-tools" does not exist currently. This program is
    centos67-pe-puppet-382: going to create it, including needed parent directories. Is this what you want?
    centos67-pe-puppet-382: [yes]
    centos67-pe-puppet-382: In which directory do you want to install the documentation files?
    centos67-pe-puppet-382: [/usr/share/doc/vmware-tools]
    centos67-pe-puppet-382: The path "/usr/share/doc/vmware-tools" does not exist currently. This program
    centos67-pe-puppet-382: is going to create it, including needed parent directories. Is this what you
    centos67-pe-puppet-382: want? [yes]
    centos67-pe-puppet-382: The installation of VMware Tools 9.9.4 build-3193940 for Linux completed
    centos67-pe-puppet-382: successfully. You can decide to remove this software from your system at any
    centos67-pe-puppet-382: time by invoking the following command: "/usr/bin/vmware-uninstall-tools.pl".
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: Before running VMware Tools for the first time, you need to configure it by
    centos67-pe-puppet-382: invoking the following command: "/usr/bin/vmware-config-tools.pl". Do you want
    centos67-pe-puppet-382: this program to invoke the command for you now? [yes]
    centos67-pe-puppet-382: Initializing...
    centos67-pe-puppet-382:
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: Making sure services for VMware Tools are stopped.
    centos67-pe-puppet-382:
    centos67-pe-puppet-382:
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: Found a compatible pre-built module for vmci.  Installing it...
    centos67-pe-puppet-382:
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: Found a compatible pre-built module for vsock.  Installing it...
    centos67-pe-puppet-382:
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: The module vmxnet3 has already been installed on this system by another
    centos67-pe-puppet-382: installer or package and will not be modified by this installer.
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: The module pvscsi has already been installed on this system by another
    centos67-pe-puppet-382: installer or package and will not be modified by this installer.
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: The module vmmemctl has already been installed on this system by another
    centos67-pe-puppet-382: installer or package and will not be modified by this installer.
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: The VMware Host-Guest Filesystem allows for shared folders between the host OS
    centos67-pe-puppet-382: and the guest OS in a Fusion or Workstation virtual environment.  Do you wish
    centos67-pe-puppet-382: to enable this feature? [no]
    centos67-pe-puppet-382: Found a compatible pre-built module for vmhgfs.  Installing it...
    centos67-pe-puppet-382:
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: Found a compatible pre-built module for vmxnet.  Installing it...
    centos67-pe-puppet-382:
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: The vmblock enables dragging or copying files between host and guest in a
    centos67-pe-puppet-382: Fusion or Workstation virtual environment.  Do you wish to enable this feature?
    centos67-pe-puppet-382: [no]
    centos67-pe-puppet-382: VMware automatic kernel modules enables automatic building and installation of
    centos67-pe-puppet-382: VMware kernel modules at boot that are not already present. This feature can be
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: enabled/disabled by re-running vmware-config-tools.pl.
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: Would you like to enable VMware automatic kernel modules?
    centos67-pe-puppet-382: [no]
    centos67-pe-puppet-382: Do you want to enable Guest Authentication (vgauth)? [yes]
    centos67-pe-puppet-382: No X install found.
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: Creating a new initrd boot image for the kernel.
    centos67-pe-puppet-382: vmware-tools-thinprint start/running
    centos67-pe-puppet-382: vmware-tools start/running
    centos67-pe-puppet-382: The configuration of VMware Tools 9.9.4 build-3193940 for Linux for this
    centos67-pe-puppet-382: running kernel completed successfully.
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: You must restart your X session before any mouse or graphics changes take
    centos67-pe-puppet-382: effect.
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: You can now run VMware Tools by invoking "/usr/bin/vmware-toolbox-cmd" from the
    centos67-pe-puppet-382: command line.
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: To enable advanced X features (e.g., guest resolution fit, drag and drop, and
    centos67-pe-puppet-382: file and text copy/paste), you will need to do one (or more) of the following:
    centos67-pe-puppet-382: 1. Manually start /usr/bin/vmware-user
    centos67-pe-puppet-382: 2. Log out and log back into your desktop session; and,
    centos67-pe-puppet-382: 3. Restart your X session.
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: Enjoy,
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: --the VMware team
    centos67-pe-puppet-382:

{{< /highlight >}}

The fourth provisioner is the shell provisioner that will run the scripts/centos-install-pe-puppet-382.sh bash script. This is the script we added to the Packer template to install the Puppet Enterprise 3.8.2 agent.

{{< highlight bash >}}
==> centos67-pe-puppet-382: Provisioning with shell script: scripts/centos-install-pe-puppet-382.sh
    centos67-pe-puppet-382: Fetching puppet-enterprise-3.8.2-el-6-x86_64.tar.gz
    centos67-pe-puppet-382: % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
    centos67-pe-puppet-382: Dload  Upload   Total   Spent    Left  Speed
    centos67-pe-puppet-382: 100  437M  100  437M    0     0  6688k      0  0:01:06  0:01:06 --:--:-- 6879k
    centos67-pe-puppet-382: Extracting puppet-enterprise-3.8.2-el-6-x86_64.tar.gz
    centos67-pe-puppet-382: ========================================================================
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: Puppet Enterprise v3.8.2 installer
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: Puppet Enterprise documentation can be found at http://docs.puppetlabs.com/pe/3.8/
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: ------------------------------------------------------------------------
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: STEP 1: READ ANSWERS FROM FILE
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: ## Reading answers from file: ./agent.ans
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: ------------------------------------------------------------------------
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: STEP 2: SELECT AND CONFIGURE ROLES
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: This installer lets you select and install the various roles
    centos67-pe-puppet-382: required in a Puppet Enterprise deployment: puppet master,
    centos67-pe-puppet-382: console, database, cloud provisioner, and puppet agent.
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: NOTE: when specifying hostnames during installation, use the fully-qualified domain name (foo.example.com) rather than a shortened name (foo).
    centos67-pe-puppet-382:
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: -> puppet master
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: The puppet master serves configurations to a group of puppet
    centos67-pe-puppet-382: agent nodes. This role also provides MCollective's message queue
    centos67-pe-puppet-382: and client interface. It should be installed on a robust,
    centos67-pe-puppet-382: dedicated server.
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: ?? Install puppet master? [y/N] n
    centos67-pe-puppet-382: ?? Puppet master hostname to connect to? [Default: puppet] puppet
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: -> database support
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: This role provides database support for PuppetDB and PE's
    centos67-pe-puppet-382: console. PuppetDB is a centralized data service that caches data
    centos67-pe-puppet-382: generated by Puppet and provides access to it via a robust API.
    centos67-pe-puppet-382: The console uses data provided by a PostgreSQL server and
    centos67-pe-puppet-382: database both of which will be installed along with PuppetDB on
    centos67-pe-puppet-382: the node you specify.
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: IMPORTANT: If you choose not to install PuppetDB at this time,
    centos67-pe-puppet-382: you will be prompted for the host name of the node you intend to
    centos67-pe-puppet-382: use to provide database services. Note that you must install
    centos67-pe-puppet-382: database support on that node for the console to function. When
    centos67-pe-puppet-382: using a separate node, you should install database support on it
    centos67-pe-puppet-382: BEFORE installing the console role.
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: ?? Install PuppetDB? [y/N] n
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: -> console
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: The console is a web interface where you can view reports,
    centos67-pe-puppet-382: classify nodes, control Puppet runs, and invoke MCollective
    centos67-pe-puppet-382: agents. It can be installed on the puppet master's node, but for
    centos67-pe-puppet-382: performance considerations, especially in larger deployments, it
    centos67-pe-puppet-382: can also be installed on a separate node.
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: ?? Install the console? [y/N] n
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: -> cloud provisioner
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: The cloud provisioner can create and bootstrap new machine
    centos67-pe-puppet-382: instances and add them to your Puppet infrastructure. It should
    centos67-pe-puppet-382: be installed on a trusted node where site administrators have
    centos67-pe-puppet-382: shell access.
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: ?? Install the cloud provisioner? [y/N] y
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: -> puppet agent
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: The puppet agent role is automatically installed with the
    centos67-pe-puppet-382: console, puppet master, puppetdb, and cloud provisioner roles.
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: ?? Puppet agent needs a unique name ("certname") for its
    centos67-pe-puppet-382: certificate; this can be an arbitrary string. Certname for this
    centos67-pe-puppet-382: node? [Default: localhost] localhost.localdomain
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: -> Vendor Packages
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: The installer has detected that Puppet Enterprise requires
    centos67-pe-puppet-382: additional packages from your operating system vendor's
    centos67-pe-puppet-382: repositories, and can automatically install them. If you choose
    centos67-pe-puppet-382: not to install these packages automatically, the installer will
    centos67-pe-puppet-382: exit so you can install them manually.
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: Additional vendor packages required for installation:
    centos67-pe-puppet-382: * dmidecode
    centos67-pe-puppet-382: * libxslt
    centos67-pe-puppet-382: * pciutils
    centos67-pe-puppet-382:
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: ?? Install these packages automatically? [Y/n] y
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: ------------------------------------------------------------------------
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: STEP 3: CONFIRM PLAN
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: You have selected to install the following components (and their dependencies)
    centos67-pe-puppet-382: * Cloud Provisioner
    centos67-pe-puppet-382: * Puppet Agent
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: ?? Perform installation? [Y/n] y
    centos67-pe-puppet-382: ## Answers saved in the following files: /tmp/puppet-enterprise-3.8.2-el-6-x86_64/answers.lastrun.localhost and /etc/puppetlabs/installer/answers.install
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: ========================================================================
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: ------------------------------------------------------------------------
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: STEP 4: INSTALL PACKAGES
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: ## Installing packages from repositories...
    centos67-pe-puppet-382: Loaded plugins: fastestmirror
    centos67-pe-puppet-382: Setting up Install Process
    centos67-pe-puppet-382: Loading mirror speeds from cached hostfile
    centos67-pe-puppet-382: * base: centos.unixheads.org
    centos67-pe-puppet-382: * extras: mirror.rackspace.com
    centos67-pe-puppet-382: * updates: centos-mirror.jchost.net
    centos67-pe-puppet-382: Package zlib-1.2.3-29.el6.x86_64 already installed and latest version
    centos67-pe-puppet-382: Package which-2.19-6.el6.x86_64 already installed and latest version
    centos67-pe-puppet-382: Package net-tools-1.60-110.el6_2.x86_64 already installed and latest version
    centos67-pe-puppet-382: Resolving Dependencies
    centos67-pe-puppet-382: --> Running transaction check
    centos67-pe-puppet-382: ---> Package cronie.x86_64 0:1.4.4-15.el6 will be updated
    centos67-pe-puppet-382: --> Processing Dependency: cronie = 1.4.4-15.el6 for package: cronie-anacron-1.4.4-15.el6.x86_64
    centos67-pe-puppet-382: ---> Package cronie.x86_64 0:1.4.4-15.el6_7.1 will be an update
    centos67-pe-puppet-382: ---> Package dmidecode.x86_64 1:2.12-6.el6 will be installed
    centos67-pe-puppet-382: ---> Package libxml2.x86_64 0:2.7.6-20.el6 will be updated
    centos67-pe-puppet-382: ---> Package libxml2.x86_64 0:2.7.6-20.el6_7.1 will be an update
    centos67-pe-puppet-382: ---> Package libxslt.x86_64 0:1.1.26-2.el6_3.1 will be installed
    centos67-pe-puppet-382: ---> Package pciutils.x86_64 0:3.1.10-4.el6 will be installed
    centos67-pe-puppet-382: ---> Package pe-agent.noarch 0:3.8.2-1.pe.el6 will be installed
    centos67-pe-puppet-382: --> Processing Dependency: pe-virt-what >= 1.13-1.el6 for package: pe-agent-3.8.2-1.pe.el6.noarch
    centos67-pe-puppet-382: --> Processing Dependency: pe-rubygem-deep-merge >= 1.0.0-3.pe.el6 for package: pe-agent-3.8.2-1.pe.el6.noarch
    centos67-pe-puppet-382: --> Processing Dependency: pe-ruby-stomp >= 1.3.3-1.pe.el6 for package: pe-agent-3.8.2-1.pe.el6.noarch
    centos67-pe-puppet-382: --> Processing Dependency: pe-ruby-shadow >= 2.2.0-3.pe.el6 for package: pe-agent-3.8.2-1.pe.el6.noarch
    centos67-pe-puppet-382: --> Processing Dependency: pe-ruby-selinux >= 2.0.94-4.pe.el6 for package: pe-agent-3.8.2-1.pe.el6.noarch
    centos67-pe-puppet-382: --> Processing Dependency: pe-ruby-rgen >= 0.6.5-1.pe.el6 for package: pe-agent-3.8.2-1.pe.el6.noarch
    centos67-pe-puppet-382: --> Processing Dependency: pe-ruby-augeas >= 0.5.0-7.pe.el6 for package: pe-agent-3.8.2-1.pe.el6.noarch
    centos67-pe-puppet-382: --> Processing Dependency: pe-ruby >= 1.9.3.551-2.pe.el6 for package: pe-agent-3.8.2-1.pe.el6.noarch
    centos67-pe-puppet-382: --> Processing Dependency: pe-puppet-enterprise-release >= 3.8.2.0-1.pe.el6 for package: pe-agent-3.8.2-1.pe.el6.noarch
    centos67-pe-puppet-382: --> Processing Dependency: pe-puppet >= 3.8.2.0-1.pe.el6 for package: pe-agent-3.8.2-1.pe.el6.noarch
    centos67-pe-puppet-382: --> Processing Dependency: pe-openssl >= 1.0.0s-1.pe.el6 for package: pe-agent-3.8.2-1.pe.el6.noarch
    centos67-pe-puppet-382: --> Processing Dependency: pe-mcollective-common >= 2.7.0.1-1.pe.el6 for package: pe-agent-3.8.2-1.pe.el6.noarch
    centos67-pe-puppet-382: --> Processing Dependency: pe-mcollective >= 2.7.0.1-1.pe.el6 for package: pe-agent-3.8.2-1.pe.el6.noarch
    centos67-pe-puppet-382: --> Processing Dependency: pe-libyaml >= 0.1.6-5.el6 for package: pe-agent-3.8.2-1.pe.el6.noarch
    centos67-pe-puppet-382: --> Processing Dependency: pe-libldap >= 2.4.39-5.pe.el6 for package: pe-agent-3.8.2-1.pe.el6.noarch
    centos67-pe-puppet-382: --> Processing Dependency: pe-hiera >= 1.3.4.5-1.pe.el6 for package: pe-agent-3.8.2-1.pe.el6.noarch
    centos67-pe-puppet-382: --> Processing Dependency: pe-facter >= 2.4.4.0-1.pe.el6 for package: pe-agent-3.8.2-1.pe.el6.noarch
    centos67-pe-puppet-382: --> Processing Dependency: pe-augeas >= 1.3.0-2.pe.el6 for package: pe-agent-3.8.2-1.pe.el6.noarch
    centos67-pe-puppet-382: ---> Package pe-cloud-provisioner.noarch 0:1.2.0-1.el6 will be installed
    centos67-pe-puppet-382: ---> Package pe-cloud-provisioner-libs.x86_64 0:0.3.2-2.pe.el6 will be installed
    centos67-pe-puppet-382: ---> Package pe-ruby-ldap.x86_64 0:0.9.12-7.pe.el6 will be installed
    centos67-pe-puppet-382: ---> Package pe-rubygem-net-ssh.noarch 0:2.1.4-2.pe.el6 will be installed
    centos67-pe-puppet-382: --> Running transaction check
    centos67-pe-puppet-382: ---> Package cronie-anacron.x86_64 0:1.4.4-15.el6 will be updated
    centos67-pe-puppet-382: ---> Package cronie-anacron.x86_64 0:1.4.4-15.el6_7.1 will be an update
    centos67-pe-puppet-382: ---> Package pe-augeas.x86_64 0:1.3.0-2.pe.el6 will be installed
    centos67-pe-puppet-382: ---> Package pe-facter.x86_64 0:2.4.4.0-1.pe.el6 will be installed
    centos67-pe-puppet-382: ---> Package pe-hiera.noarch 0:1.3.4.5-1.pe.el6 will be installed
    centos67-pe-puppet-382: ---> Package pe-libldap.x86_64 0:2.4.39-5.pe.el6 will be installed
    centos67-pe-puppet-382: ---> Package pe-libyaml.x86_64 0:0.1.6-5.el6 will be installed
    centos67-pe-puppet-382: ---> Package pe-mcollective.noarch 0:2.7.0.1-1.pe.el6 will be installed
    centos67-pe-puppet-382: ---> Package pe-mcollective-common.noarch 0:2.7.0.1-1.pe.el6 will be installed
    centos67-pe-puppet-382: ---> Package pe-openssl.x86_64 0:1.0.0s-1.pe.el6 will be installed
    centos67-pe-puppet-382: ---> Package pe-puppet.noarch 0:3.8.2.0-1.pe.el6 will be installed
    centos67-pe-puppet-382: ---> Package pe-puppet-enterprise-release.noarch 0:3.8.2.0-1.pe.el6 will be installed
    centos67-pe-puppet-382: ---> Package pe-ruby.x86_64 0:1.9.3.551-2.pe.el6 will be installed
    centos67-pe-puppet-382: ---> Package pe-ruby-augeas.x86_64 0:0.5.0-7.pe.el6 will be installed
    centos67-pe-puppet-382: ---> Package pe-ruby-rgen.noarch 0:0.6.5-1.pe.el6 will be installed
    centos67-pe-puppet-382: ---> Package pe-ruby-selinux.x86_64 0:2.0.94-4.pe.el6 will be installed
    centos67-pe-puppet-382: ---> Package pe-ruby-shadow.x86_64 0:2.2.0-3.pe.el6 will be installed
    centos67-pe-puppet-382: ---> Package pe-ruby-stomp.noarch 0:1.3.3-1.pe.el6 will be installed
    centos67-pe-puppet-382: ---> Package pe-rubygem-deep-merge.noarch 0:1.0.0-3.pe.el6 will be installed
    centos67-pe-puppet-382: ---> Package pe-virt-what.x86_64 0:1.13-1.el6 will be installed
    centos67-pe-puppet-382: --> Finished Dependency Resolution
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: Dependencies Resolved
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: ================================================================================
    centos67-pe-puppet-382: Package            Arch   Version            Repository                   Size
    centos67-pe-puppet-382: ================================================================================
    centos67-pe-puppet-382: Installing:
    centos67-pe-puppet-382: dmidecode          x86_64 1:2.12-6.el6       base                         74 k
    centos67-pe-puppet-382: libxslt            x86_64 1.1.26-2.el6_3.1   base                        452 k
    centos67-pe-puppet-382: pciutils           x86_64 3.1.10-4.el6       base                         85 k
    centos67-pe-puppet-382: pe-agent           noarch 3.8.2-1.pe.el6     puppet-enterprise-installer 3.7 k
    centos67-pe-puppet-382: pe-cloud-provisioner
    centos67-pe-puppet-382: noarch 1.2.0-1.el6        puppet-enterprise-installer  51 k
    centos67-pe-puppet-382: pe-cloud-provisioner-libs
    centos67-pe-puppet-382: x86_64 0.3.2-2.pe.el6     puppet-enterprise-installer 4.6 M
    centos67-pe-puppet-382: pe-ruby-ldap       x86_64 0.9.12-7.pe.el6    puppet-enterprise-installer  46 k
    centos67-pe-puppet-382: pe-rubygem-net-ssh noarch 2.1.4-2.pe.el6     puppet-enterprise-installer 227 k
    centos67-pe-puppet-382: Updating:
    centos67-pe-puppet-382: cronie             x86_64 1.4.4-15.el6_7.1   updates                      74 k
    centos67-pe-puppet-382: libxml2            x86_64 2.7.6-20.el6_7.1   updates                     803 k
    centos67-pe-puppet-382: Installing for dependencies:
    centos67-pe-puppet-382: pe-augeas          x86_64 1.3.0-2.pe.el6     puppet-enterprise-installer 523 k
    centos67-pe-puppet-382: pe-facter          x86_64 2.4.4.0-1.pe.el6   puppet-enterprise-installer  99 k
    centos67-pe-puppet-382: pe-hiera           noarch 1.3.4.5-1.pe.el6   puppet-enterprise-installer  18 k
    centos67-pe-puppet-382: pe-libldap         x86_64 2.4.39-5.pe.el6    puppet-enterprise-installer 776 k
    centos67-pe-puppet-382: pe-libyaml         x86_64 0.1.6-5.el6        puppet-enterprise-installer 193 k
    centos67-pe-puppet-382: pe-mcollective     noarch 2.7.0.1-1.pe.el6   puppet-enterprise-installer 8.0 k
    centos67-pe-puppet-382: pe-mcollective-common
    centos67-pe-puppet-382: noarch 2.7.0.1-1.pe.el6   puppet-enterprise-installer 129 k
    centos67-pe-puppet-382: pe-openssl         x86_64 1.0.0s-1.pe.el6    puppet-enterprise-installer 6.7 M
    centos67-pe-puppet-382: pe-puppet          noarch 3.8.2.0-1.pe.el6   puppet-enterprise-installer 1.6 M
    centos67-pe-puppet-382: pe-puppet-enterprise-release
    centos67-pe-puppet-382: noarch 3.8.2.0-1.pe.el6   puppet-enterprise-installer  12 k
    centos67-pe-puppet-382: pe-ruby            x86_64 1.9.3.551-2.pe.el6 puppet-enterprise-installer 8.1 M
    centos67-pe-puppet-382: pe-ruby-augeas     x86_64 0.5.0-7.pe.el6     puppet-enterprise-installer  22 k
    centos67-pe-puppet-382: pe-ruby-rgen       noarch 0.6.5-1.pe.el6     puppet-enterprise-installer 238 k
    centos67-pe-puppet-382: pe-ruby-selinux    x86_64 2.0.94-4.pe.el6    puppet-enterprise-installer  57 k
    centos67-pe-puppet-382: pe-ruby-shadow     x86_64 2.2.0-3.pe.el6     puppet-enterprise-installer  11 k
    centos67-pe-puppet-382: pe-ruby-stomp      noarch 1.3.3-1.pe.el6     puppet-enterprise-installer  54 k
    centos67-pe-puppet-382: pe-rubygem-deep-merge
    centos67-pe-puppet-382: noarch 1.0.0-3.pe.el6     puppet-enterprise-installer  74 k
    centos67-pe-puppet-382: pe-virt-what       x86_64 1.13-1.el6         puppet-enterprise-installer  22 k
    centos67-pe-puppet-382: Updating for dependencies:
    centos67-pe-puppet-382: cronie-anacron     x86_64 1.4.4-15.el6_7.1   updates                      31 k
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: Transaction Summary
    centos67-pe-puppet-382: ================================================================================
    centos67-pe-puppet-382: Install      26 Package(s)
    centos67-pe-puppet-382: Upgrade       3 Package(s)
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: Total download size: 25 M
    centos67-pe-puppet-382: Downloading Packages:
    centos67-pe-puppet-382: --------------------------------------------------------------------------------
    centos67-pe-puppet-382: Total                                            11 MB/s |  25 MB     00:02
    centos67-pe-puppet-382: Running rpm_check_debug
    centos67-pe-puppet-382: Running Transaction Test
    centos67-pe-puppet-382: Transaction Test Succeeded
    centos67-pe-puppet-382: Running Transaction
    centos67-pe-puppet-382: Installing : pe-puppet-enterprise-release-3.8.2.0-1.pe.el6.noarch        1/32
    centos67-pe-puppet-382: Installing : pe-openssl-1.0.0s-1.pe.el6.x86_64                           2/32
    centos67-pe-puppet-382: Installing : pe-libldap-2.4.39-5.pe.el6.x86_64                           3/32
    centos67-pe-puppet-382: Installing : pe-libyaml-0.1.6-5.el6.x86_64                               4/32
    centos67-pe-puppet-382: Installing : pe-ruby-1.9.3.551-2.pe.el6.x86_64                           5/32
    centos67-pe-puppet-382: Installing : pe-rubygem-net-ssh-2.1.4-2.pe.el6.noarch                    6/32
    centos67-pe-puppet-382: Installing : pe-ruby-stomp-1.3.3-1.pe.el6.noarch                         7/32
    centos67-pe-puppet-382: Installing : pe-mcollective-common-2.7.0.1-1.pe.el6.noarch               8/32
    centos67-pe-puppet-382: Installing : pe-ruby-selinux-2.0.94-4.pe.el6.x86_64                      9/32
    centos67-pe-puppet-382: Installing : pe-ruby-rgen-0.6.5-1.pe.el6.noarch                         10/32
    centos67-pe-puppet-382: Installing : pe-rubygem-deep-merge-1.0.0-3.pe.el6.noarch                11/32
    centos67-pe-puppet-382: Installing : pe-hiera-1.3.4.5-1.pe.el6.noarch                           12/32
    centos67-pe-puppet-382: Installing : pe-augeas-1.3.0-2.pe.el6.x86_64                            13/32
    centos67-pe-puppet-382: Installing : pe-ruby-augeas-0.5.0-7.pe.el6.x86_64                       14/32
    centos67-pe-puppet-382: Updating   : libxml2-2.7.6-20.el6_7.1.x86_64                            15/32
    centos67-pe-puppet-382: Installing : 1:dmidecode-2.12-6.el6.x86_64                              16/32
    centos67-pe-puppet-382: Installing : pe-virt-what-1.13-1.el6.x86_64                             17/32
    centos67-pe-puppet-382: Installing : libxslt-1.1.26-2.el6_3.1.x86_64                            18/32
    centos67-pe-puppet-382: Installing : pe-cloud-provisioner-libs-0.3.2-2.pe.el6.x86_64            19/32
    centos67-pe-puppet-382: Installing : pe-mcollective-2.7.0.1-1.pe.el6.noarch                     20/32
    centos67-pe-puppet-382: Installing : pe-ruby-ldap-0.9.12-7.pe.el6.x86_64                        21/32
    centos67-pe-puppet-382: Installing : pe-ruby-shadow-2.2.0-3.pe.el6.x86_64                       22/32
    centos67-pe-puppet-382: Updating   : cronie-anacron-1.4.4-15.el6_7.1.x86_64                     23/32
    centos67-pe-puppet-382: Updating   : cronie-1.4.4-15.el6_7.1.x86_64                             24/32
    centos67-pe-puppet-382: Installing : pciutils-3.1.10-4.el6.x86_64                               25/32
    centos67-pe-puppet-382: Installing : pe-facter-2.4.4.0-1.pe.el6.x86_64                          26/32
    centos67-pe-puppet-382: Installing : pe-puppet-3.8.2.0-1.pe.el6.noarch                          27/32
    centos67-pe-puppet-382: Installing : pe-agent-3.8.2-1.pe.el6.noarch                             28/32
    centos67-pe-puppet-382: Installing : pe-cloud-provisioner-1.2.0-1.el6.noarch                    29/32
    centos67-pe-puppet-382: Cleanup    : cronie-anacron-1.4.4-15.el6.x86_64                         30/32
    centos67-pe-puppet-382: Cleanup    : cronie-1.4.4-15.el6.x86_64                                 31/32
    centos67-pe-puppet-382: Cleanup    : libxml2-2.7.6-20.el6.x86_64                                32/32
    centos67-pe-puppet-382: Verifying  : pe-puppet-enterprise-release-3.8.2.0-1.pe.el6.noarch        1/32
    centos67-pe-puppet-382: Verifying  : pe-hiera-1.3.4.5-1.pe.el6.noarch                            2/32
    centos67-pe-puppet-382: Verifying  : pe-mcollective-common-2.7.0.1-1.pe.el6.noarch               3/32
    centos67-pe-puppet-382: Verifying  : pciutils-3.1.10-4.el6.x86_64                                4/32
    centos67-pe-puppet-382: Verifying  : pe-libldap-2.4.39-5.pe.el6.x86_64                           5/32
    centos67-pe-puppet-382: Verifying  : pe-ruby-augeas-0.5.0-7.pe.el6.x86_64                        6/32
    centos67-pe-puppet-382: Verifying  : pe-agent-3.8.2-1.pe.el6.noarch                              7/32
    centos67-pe-puppet-382: Verifying  : pe-ruby-ldap-0.9.12-7.pe.el6.x86_64                         8/32
    centos67-pe-puppet-382: Verifying  : 1:dmidecode-2.12-6.el6.x86_64                               9/32
    centos67-pe-puppet-382: Verifying  : pe-mcollective-2.7.0.1-1.pe.el6.noarch                     10/32
    centos67-pe-puppet-382: Verifying  : pe-cloud-provisioner-libs-0.3.2-2.pe.el6.x86_64            11/32
    centos67-pe-puppet-382: Verifying  : cronie-1.4.4-15.el6_7.1.x86_64                             12/32
    centos67-pe-puppet-382: Verifying  : pe-rubygem-net-ssh-2.1.4-2.pe.el6.noarch                   13/32
    centos67-pe-puppet-382: Verifying  : pe-openssl-1.0.0s-1.pe.el6.x86_64                          14/32
    centos67-pe-puppet-382: Verifying  : pe-facter-2.4.4.0-1.pe.el6.x86_64                          15/32
    centos67-pe-puppet-382: Verifying  : pe-ruby-selinux-2.0.94-4.pe.el6.x86_64                     16/32
    centos67-pe-puppet-382: Verifying  : pe-ruby-stomp-1.3.3-1.pe.el6.noarch                        17/32
    centos67-pe-puppet-382: Verifying  : pe-ruby-shadow-2.2.0-3.pe.el6.x86_64                       18/32
    centos67-pe-puppet-382: Verifying  : libxslt-1.1.26-2.el6_3.1.x86_64                            19/32
    centos67-pe-puppet-382: Verifying  : pe-ruby-rgen-0.6.5-1.pe.el6.noarch                         20/32
    centos67-pe-puppet-382: Verifying  : pe-rubygem-deep-merge-1.0.0-3.pe.el6.noarch                21/32
    centos67-pe-puppet-382: Verifying  : pe-puppet-3.8.2.0-1.pe.el6.noarch                          22/32
    centos67-pe-puppet-382: Verifying  : cronie-anacron-1.4.4-15.el6_7.1.x86_64                     23/32
    centos67-pe-puppet-382: Verifying  : pe-libyaml-0.1.6-5.el6.x86_64                              24/32
    centos67-pe-puppet-382: Verifying  : pe-virt-what-1.13-1.el6.x86_64                             25/32
    centos67-pe-puppet-382: Verifying  : pe-augeas-1.3.0-2.pe.el6.x86_64                            26/32
    centos67-pe-puppet-382: Verifying  : pe-cloud-provisioner-1.2.0-1.el6.noarch                    27/32
    centos67-pe-puppet-382: Verifying  : pe-ruby-1.9.3.551-2.pe.el6.x86_64                          28/32
    centos67-pe-puppet-382: Verifying  : libxml2-2.7.6-20.el6_7.1.x86_64                            29/32
    centos67-pe-puppet-382: Verifying  : libxml2-2.7.6-20.el6.x86_64                                30/32
    centos67-pe-puppet-382: Verifying  : cronie-1.4.4-15.el6.x86_64                                 31/32
    centos67-pe-puppet-382: Verifying  : cronie-anacron-1.4.4-15.el6.x86_64                         32/32
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: Installed:
    centos67-pe-puppet-382: dmidecode.x86_64 1:2.12-6.el6
    centos67-pe-puppet-382: libxslt.x86_64 0:1.1.26-2.el6_3.1
    centos67-pe-puppet-382: pciutils.x86_64 0:3.1.10-4.el6
    centos67-pe-puppet-382: pe-agent.noarch 0:3.8.2-1.pe.el6
    centos67-pe-puppet-382: pe-cloud-provisioner.noarch 0:1.2.0-1.el6
    centos67-pe-puppet-382: pe-cloud-provisioner-libs.x86_64 0:0.3.2-2.pe.el6
    centos67-pe-puppet-382: pe-ruby-ldap.x86_64 0:0.9.12-7.pe.el6
    centos67-pe-puppet-382: pe-rubygem-net-ssh.noarch 0:2.1.4-2.pe.el6
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: Dependency Installed:
    centos67-pe-puppet-382: pe-augeas.x86_64 0:1.3.0-2.pe.el6
    centos67-pe-puppet-382: pe-facter.x86_64 0:2.4.4.0-1.pe.el6
    centos67-pe-puppet-382: pe-hiera.noarch 0:1.3.4.5-1.pe.el6
    centos67-pe-puppet-382: pe-libldap.x86_64 0:2.4.39-5.pe.el6
    centos67-pe-puppet-382: pe-libyaml.x86_64 0:0.1.6-5.el6
    centos67-pe-puppet-382: pe-mcollective.noarch 0:2.7.0.1-1.pe.el6
    centos67-pe-puppet-382: pe-mcollective-common.noarch 0:2.7.0.1-1.pe.el6
    centos67-pe-puppet-382: pe-openssl.x86_64 0:1.0.0s-1.pe.el6
    centos67-pe-puppet-382: pe-puppet.noarch 0:3.8.2.0-1.pe.el6
    centos67-pe-puppet-382: pe-puppet-enterprise-release.noarch 0:3.8.2.0-1.pe.el6
    centos67-pe-puppet-382: pe-ruby.x86_64 0:1.9.3.551-2.pe.el6
    centos67-pe-puppet-382: pe-ruby-augeas.x86_64 0:0.5.0-7.pe.el6
    centos67-pe-puppet-382: pe-ruby-rgen.noarch 0:0.6.5-1.pe.el6
    centos67-pe-puppet-382: pe-ruby-selinux.x86_64 0:2.0.94-4.pe.el6
    centos67-pe-puppet-382: pe-ruby-shadow.x86_64 0:2.2.0-3.pe.el6
    centos67-pe-puppet-382: pe-ruby-stomp.noarch 0:1.3.3-1.pe.el6
    centos67-pe-puppet-382: pe-rubygem-deep-merge.noarch 0:1.0.0-3.pe.el6
    centos67-pe-puppet-382: pe-virt-what.x86_64 0:1.13-1.el6
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: Updated:
    centos67-pe-puppet-382: cronie.x86_64 0:1.4.4-15.el6_7.1       libxml2.x86_64 0:2.7.6-20.el6_7.1
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: Dependency Updated:
    centos67-pe-puppet-382: cronie-anacron.x86_64 0:1.4.4-15.el6_7.1
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: Complete!
    centos67-pe-puppet-382: Loaded plugins: fastestmirror
    centos67-pe-puppet-382: Cleaning repos: puppet-enterprise-installer
    centos67-pe-puppet-382: Cleaning up Everything
    centos67-pe-puppet-382: Cleaning up list of fastest mirrors
    centos67-pe-puppet-382: ## Checking the agent certificate name detection...
    centos67-pe-puppet-382: ## Setting up puppet agent...
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: ------------------------------------------------------------------------
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: STEP 5: DONE
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: Thanks for installing Puppet Enterprise!
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: To learn more and get started using Puppet Enterprise, refer to
    centos67-pe-puppet-382: the Puppet Enterprise Quick Start Guide
    centos67-pe-puppet-382: (http://docs.puppetlabs.com/pe/latest/quick_start.html) and the
    centos67-pe-puppet-382: Puppet Enterprise Deployment Guide
    centos67-pe-puppet-382: (http://docs.puppetlabs.com/guides/deployment_guide/index.html).
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: ========================================================================
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: ## NOTES
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: Puppet Enterprise has been installed to "/opt/puppet," and its
    centos67-pe-puppet-382: configuration files are located in "/etc/puppetlabs".
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: Answers from this session saved to
    centos67-pe-puppet-382: '/tmp/puppet-enterprise-3.8.2-el-6-x86_64/answers.lastrun.localhost'
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: If you have a firewall running, please ensure outbound
    centos67-pe-puppet-382: connections are allowed to the following TCP ports: 8140, 61613
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: ------------------------------------------------------------------------
    centos67-pe-puppet-382:
{{< /highlight >}}

The final provisioner is a shell provisioner that will run the scripts/centos-vmware-cleanup.sh bash script. This script will clean up the centos67 virtual machine and zero out all unused disk space to reduce the size of the image.

{{< highlight bash >}}    
==> centos67-pe-puppet-382: Provisioning with shell script: scripts/centos-vmware-cleanup.sh
    centos67-pe-puppet-382: ==> Pausing for 0 seconds...
    centos67-pe-puppet-382: ==> erasing unused packages to free up space
    centos67-pe-puppet-382: Loaded plugins: fastestmirror
    centos67-pe-puppet-382: Setting up Remove Process
    centos67-pe-puppet-382: No Match for argument: gtk2
    centos67-pe-puppet-382: Determining fastest mirrors
    centos67-pe-puppet-382: * base: centos.unixheads.org
    centos67-pe-puppet-382: * extras: centos.unixheads.org
    centos67-pe-puppet-382: * updates: centos-mirror.jchost.net
    centos67-pe-puppet-382: Package(s) gtk2 available, but not installed.
    centos67-pe-puppet-382: No Match for argument: libX11
    centos67-pe-puppet-382: Package(s) libX11 available, but not installed.
    centos67-pe-puppet-382: No Match for argument: hicolor-icon-theme
    centos67-pe-puppet-382: Package(s) hicolor-icon-theme available, but not installed.
    centos67-pe-puppet-382: No Match for argument: avahi
    centos67-pe-puppet-382: Package(s) avahi available, but not installed.
    centos67-pe-puppet-382: No Match for argument: freetype
    centos67-pe-puppet-382: Package(s) freetype available, but not installed.
    centos67-pe-puppet-382: No Match for argument: bitstream-vera-fonts
    centos67-pe-puppet-382: No Packages marked for removal
    centos67-pe-puppet-382: ==> Cleaning up yum cache
    centos67-pe-puppet-382: Loaded plugins: fastestmirror
    centos67-pe-puppet-382: Cleaning repos: base extras updates
    centos67-pe-puppet-382: Cleaning up Everything
    centos67-pe-puppet-382: Cleaning up list of fastest mirrors
    centos67-pe-puppet-382: ==> Force logs to rotate
    centos67-pe-puppet-382: ==> Clear audit log and wtmp
    centos67-pe-puppet-382: ==> Cleaning up udev rules
    centos67-pe-puppet-382: ==> Remove the traces of the template MAC address and UUIDs
    centos67-pe-puppet-382: ==> Cleaning up tmp
    centos67-pe-puppet-382: ==> Remove the SSH host keys
    centos67-pe-puppet-382: ==> Remove the root user’s shell history
    centos67-pe-puppet-382: ==> yum -y clean all
    centos67-pe-puppet-382: Loaded plugins: fastestmirror
    centos67-pe-puppet-382: Cleaning repos: base extras updates
    centos67-pe-puppet-382: Cleaning up Everything
    centos67-pe-puppet-382: /
    centos67-pe-puppet-382: 24828344
    centos67-pe-puppet-382: /tmp/script_7943.sh: line 59: bc: command not found
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: dd: invalid number `'
    centos67-pe-puppet-382: /boot
    centos67-pe-puppet-382: 60502
    centos67-pe-puppet-382: /tmp/script_7943.sh: line 59: bc: command not found
    centos67-pe-puppet-382:
    centos67-pe-puppet-382: dd: invalid number `'
    centos67-pe-puppet-382: /tmp/script_7943.sh: line 67: /usr/sbin/vgdisplay: No such file or directory
    centos67-pe-puppet-382: ==> Zero out the free space to save space in the final image
    centos67-pe-puppet-382: dd: writing `/EMPTY': No space left on device
    centos67-pe-puppet-382: 12783+0 records in
    centos67-pe-puppet-382: 12782+0 records out
    centos67-pe-puppet-382: 13403570176 bytes (13 GB) copied, 428.651 s, 31.3 MB/s
{{< /highlight >}}

With the provisioners block having been completed, Packer will now shutdown the centos67-pe-puppet-382 virtual machine and unregister it from the ESXi virtual machine.

{{< highlight bash >}}
==> centos67-pe-puppet-382: Gracefully halting virtual machine...
    centos67-pe-puppet-382: Waiting for VMware to clean up after itself...
==> centos67-pe-puppet-382: Deleting unnecessary VMware files...
    centos67-pe-puppet-382: Deleting: /vmfs/volumes/datastore1/output-centos67-pe-puppet-382/vmware.log
==> centos67-pe-puppet-382: Cleaning VMX prior to finishing up...
    centos67-pe-puppet-382: Unmounting floppy from VMX...
    centos67-pe-puppet-382: Detaching ISO from CD-ROM device...
    centos67-pe-puppet-382: Disabling VNC server...
==> centos67-pe-puppet-382: Compacting the disk image
==> centos67-pe-puppet-382: Unregistering virtual machine...
Build 'centos67-pe-puppet-382' finished.

==> Builds finished. The artifacts of successful builds are:
--> centos67-pe-puppet-382: VM files in directory: /vmfs/volumes/datastore1/output-centos67-pe-puppet-382
[root@packer-centos packer-templates]#    
{{< /highlight >}}

The output of the packer build command shows that the template was successfully created at /vmfs/volumes/datastore1/output-centos67-pe-puppet-382 on the ESXi virtual machine.

#### This brings us to the end of this post. I think it is pretty powerful that we only had to copy/update the template and add another provisioner script, in order to modify the Packer template we created last time to install Puppet Enterprise 3.8.2.

#### If you would like to not have to manually create the two files covered in this post, you can clone down [this github repository](https://github.com/sdorsett/packer-templates/tree/adding-second-packer-template) by running the following command:

{{< highlight bash >}}
git clone -b "adding-second-packer-template" https://github.com/sdorsett/packer-templates.git
{{< /highlight >}}  

If you cloned the packer-templates repo in the last post, you can pull down the updates by running the following command:

{{< highlight bash >}}
git fetch --all
git pull origin "adding-second-packer-template"
{{< /highlight >}}  

#### In the next post we will extend what was covered in this post by installing the Puppet agent in Packer image.

---

#### Please provide any feedback or suggestions to my twitter account located on the about page.
