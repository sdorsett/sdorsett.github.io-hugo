+++
title = "Setting up Packer, ovftool and Apache web server on a CentOS virtual machine"
description = ""
tags = [
    "packer",
    "ovftool",
]
date = "2015-12-24"
categories = [
    "packer",
    "ovftool",
]
highlight = "true"
+++

This is the third in a series of posts on <a href="../2015-12-22-pipeline-for-creating-packer-box-files">using a Packer pipeline to generate Vagrant .box files</a>.

In the <a href="../2015-12-23-installing-esxi-virtual-machine-for-packer-depolyment">last post</a> we setup a ESXi virtual machine that would be the target for creating Packer images. In order to follow along with this post you will need two things:

* A fresh CentOS virtual machine on which we will install Packer - I'm using CentOS 6.6 minimal install named "packer-centos" with 2 vCPU, 4GB of memory and a 100GB virtual hard drive. I also gave this virtual machine the IP address of 192.168.1.52
* [VMware ovftool](https://my.vmware.com/web/vmware/details?productId=491&downloadGroup=OVFTOOL410) - if you followed the previous post and are using ESXi 6.0 you will need a copy of ovftool 4.1.

Let's get started...

### 1. SSH to CentOS virtual machine you created.

{{< highlight bash >}}
sdorsett-mbp:~ sdorsett$ ssh root@192.168.1.52
The authenticity of host '192.168.1.52 (192.168.1.52)' cant be established.
RSA key fingerprint is 58:f5:22:2e:f6:64:04:59:6b:0b:76:2f:33:6f:03:85.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '192.168.1.52' (RSA) to the list of known hosts.
root@192.168.1.52's password:
Last login: Thu Oct 22 17:52:17 2015 from 192.168.1.1
[root@packer-centos ~]#
{{< /highlight >}}

### 2. Install the EPEL repository

There will be several package we need to install that are coming from the CentOS EPEL repository, so we will need to install it.

{{< highlight bash >}}
[root@packer-centos ~]# yum install -y epel-release
Loaded plugins: fastestmirror
Setting up Install Process
Determining fastest mirrors
 * base: yum.tamu.edu
 * extras: centos.firehosted.com
 * updates: mirrors.adams.net
base
base/primary_db
extras
extras/primary_db
updates
updates/primary_db
Resolving Dependencies
--> Running transaction check
---> Package epel-release.noarch 0:6-8 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

=============================================================
 Package          Arch       Version     Repository     Size
=============================================================
Installing:
 epel-release     noarch     6-8         extras         14 k

Transaction Summary
=============================================================
Install       1 Package(s)

Total download size: 14 k
Installed size: 22 k
Downloading Packages:
epel-release-6-8.noarch.rpm     |  14 kB     00:00
Running rpm_check_debug
Running Transaction Test
Transaction Test Succeeded
Running Transaction
  Installing : epel-release-6-8.noarch     1/1
  Verifying  : epel-release-6-8.noarch     1/1

Installed:
  epel-release.noarch 0:6-8

Complete!

[root@packer-centos ~]#
{{< /highlight >}}

### 3. Use yum to make sure the following packages are installed:

* git - for version control and the ability to pull down git repositories
* wget - provides the ability to download packages from URLs
* unzip - needed to unzip Packer, since it comes as a .zip
* sshpass - used for running SSH commands, but it allows us to specify a password to use

{{< highlight bash >}}
[root@packer-centos ~]# yum install -y git wget unzip sshpass
Loaded plugins: fastestmirror
Setting up Install Process
Loading mirror speeds from cached hostfile
 * base: yum.tamu.edu
 * epel: fedora-epel.mirror.lstn.net
 * extras: centos.firehosted.com
 * updates: mirrors.adams.net
Package git-1.7.1-3.el6_4.1.x86_64 already installed and latest version
Package wget-1.12-5.el6_6.1.x86_64 already installed and latest version
Resolving Dependencies
--> Running transaction check
---> Package sshpass.x86_64 0:1.05-1.el6 will be installed
---> Package unzip.x86_64 0:6.0-2.el6_6 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

============================================================
 Package     Arch       Version        Repository     Size
============================================================
Installing:
 sshpass     x86_64     1.05-1.el6     epel           19 k
 unzip       x86_64     6.0-2.el6_6    base           149 k

Transaction Summary
============================================================
Install       2 Package(s)

Total download size: 168 k
Installed size: 346 k
Downloading Packages:
(1/2): sshpass-1.05-1.el6.x86_64.rpm     |  19 kB     00:00
(2/2): unzip-6.0-2.el6_6.x86_64.rpm      | 149 kB     00:00
------------------------------------------------------------
Total                                                                                                                                                                     593 kB/s | 168 kB     00:00
warning: rpmts_HdrFromFdno: Header V3 RSA/SHA256 Signature, key ID 0608b895: NOKEY
Retrieving key from file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-6
Importing GPG key 0x0608B895:
 Userid : EPEL (6) <epel@fedoraproject.org>
 Package: epel-release-6-8.noarch (@extras)
 From   : /etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-6
Running rpm_check_debug
Running Transaction Test
Transaction Test Succeeded
Running Transaction
  Installing : sshpass-1.05-1.el6.x86_64     1/2
  Installing : unzip-6.0-2.el6_6.x86_64      2/2
  Verifying  : unzip-6.0-2.el6_6.x86_64      1/2
  Verifying  : sshpass-1.05-1.el6.x86_64     2/2

Installed:
  sshpass.x86_64 0:1.05-1.el6     unzip.x86_64 0:6.0-2.el6_6

Complete!
[root@packer-centos ~]#
{{< /highlight >}}

### 4. Install Packer

Go to [https://packer.io/downloads.html](https://packer.io/downloads.html) and copy the URL for the Linux 64-bit package. In the SSH session to our CentOS virtual machine, perform the following steps:

Create a new directory to install packer into
{{< highlight bash >}}
[root@packer-centos ~]# mkdir /usr/local/packer_0.8.6
[root@packer-centos ~]#
{{< /highlight >}}

Download Packer using the URL you copied from the packer.io download page:
{{< highlight bash >}}
[root@packer-centos ~]# wget https://releases.hashicorp.com/packer/0.8.6/packer_0.8.6_linux_amd64.zip
--2015-12-23 17:22:03--  https://releases.hashicorp.com/packer/0.8.6/packer_0.8.6_linux_amd64.zip
Resolving releases.hashicorp.com... 23.235.44.69
Connecting to releases.hashicorp.com|23.235.44.69|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 132616691 (126M) [application/zip]
Saving to: “packer_0.8.6_linux_amd64.zip”

100%[====================================>] 132,616,691 6.97M/s   in 18s

2015-12-23 17:22:22 (6.95 MB/s) - “packer_0.8.6_linux_amd64.zip” saved [132616691/132616691]

[root@packer-centos ~]# unzip packer_0.8.6_linux_amd64.zip -d /usr/local/packer_8.6/
Archive:  packer_0.8.6_linux_amd64.zip
  inflating: /usr/local/packer_8.6/packer
  inflating: /usr/local/packer_8.6/packer-builder-amazon-chroot
  inflating: /usr/local/packer_8.6/packer-builder-amazon-ebs
  inflating: /usr/local/packer_8.6/packer-builder-amazon-instance
  inflating: /usr/local/packer_8.6/packer-builder-digitalocean
  inflating: /usr/local/packer_8.6/packer-builder-docker
  inflating: /usr/local/packer_8.6/packer-builder-file
  inflating: /usr/local/packer_8.6/packer-builder-googlecompute
  inflating: /usr/local/packer_8.6/packer-builder-null
  inflating: /usr/local/packer_8.6/packer-builder-openstack
  inflating: /usr/local/packer_8.6/packer-builder-parallels-iso
  inflating: /usr/local/packer_8.6/packer-builder-parallels-pvm
  inflating: /usr/local/packer_8.6/packer-builder-qemu
  inflating: /usr/local/packer_8.6/packer-builder-virtualbox-iso
  inflating: /usr/local/packer_8.6/packer-builder-virtualbox-ovf
  inflating: /usr/local/packer_8.6/packer-builder-vmware-iso
  inflating: /usr/local/packer_8.6/packer-builder-vmware-vmx
  inflating: /usr/local/packer_8.6/packer-post-processor-artifice
  inflating: /usr/local/packer_8.6/packer-post-processor-atlas
  inflating: /usr/local/packer_8.6/packer-post-processor-compress
  inflating: /usr/local/packer_8.6/packer-post-processor-docker-import
  inflating: /usr/local/packer_8.6/packer-post-processor-docker-push
  inflating: /usr/local/packer_8.6/packer-post-processor-docker-save
  inflating: /usr/local/packer_8.6/packer-post-processor-docker-tag
  inflating: /usr/local/packer_8.6/packer-post-processor-vagrant
  inflating: /usr/local/packer_8.6/packer-post-processor-vagrant-cloud
  inflating: /usr/local/packer_8.6/packer-post-processor-vsphere
  inflating: /usr/local/packer_8.6/packer-provisioner-ansible-local
  inflating: /usr/local/packer_8.6/packer-provisioner-chef-client
  inflating: /usr/local/packer_8.6/packer-provisioner-chef-solo
  inflating: /usr/local/packer_8.6/packer-provisioner-file
  inflating: /usr/local/packer_8.6/packer-provisioner-powershell
  inflating: /usr/local/packer_8.6/packer-provisioner-puppet-masterless
  inflating: /usr/local/packer_8.6/packer-provisioner-puppet-server
  inflating: /usr/local/packer_8.6/packer-provisioner-salt-masterless
  inflating: /usr/local/packer_8.6/packer-provisioner-shell
  inflating: /usr/local/packer_8.6/packer-provisioner-shell-local
  inflating: /usr/local/packer_8.6/packer-provisioner-windows-restart
  inflating: /usr/local/packer_8.6/packer-provisioner-windows-shell
[root@packer-centos ~]#
{{< /highlight >}}

Update ~/.bashrc to add "/usr/local/packer_8.6" to our path. Run "source ~/.bashrc" to re-read ~/.bashrc and validate $PATH has been updated.
{{< highlight bash >}}
[root@packer-centos ~]# echo 'export PATH="/usr/local/packer_8.6:$PATH"' >> ~/.bashrc
[root@packer-centos ~]# source ~/.bashrc
[root@packer-centos ~]# echo $PATH
/usr/local/packer_8.6:/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin:/root/bin
[root@packer-centos ~]#
{{< /highlight >}}

Ensure the packer binary can be found and is showing the proper version
{{< highlight bash >}}
[root@packer-centos ~]# which packer
/usr/local/packer_8.6/packer
[root@packer-centos ~]# packer -v
0.8.6
[root@packer-centos ~]#
{{< /highlight >}}

### 5. Stop the iptables firewall and disable it from started on reboot of the virtual machine

Packer will start a web server service for providing kickstart and script files to Packer images during install. This service will use a random port within a range, so we will need to stop iptables in order to allow the Packer images to connect.

{{< highlight bash >}}
[root@packer-centos ~]# service iptables stop
[root@packer-centos ~]# chkconfig iptables off
[root@packer-centos ~]#
{{< /highlight >}}

### 6. Install ovftool on the CentOS virtual machine

Download linux 64bit version of ovftool 4.1 on your local machine

![screenshot](/static/01-download-ovftool-linux-64bit.png)  

SCP the downloaded file to the CentOS virtual machine

{{< highlight bash >}}
sdorsett-mbp:~ sdorsett$ scp ~/Downloads/VMware-ovftool-4.1.0-2459827-lin.x86_64.bundle root@192.168.1.52:/root/
root@192.168.1.52's password:
VMware-ovftool-4.1.0-2459827-lin.x86_64.bundle                    100%   37MB  12.4MB/s   00:03
sdorsett-mbp:~ sdorsett$
{{< /highlight >}}

Install ovftool on the CentOS virtual machine and validate the version

{{< highlight bash >}}
[root@packer-centos ~]# ls
anaconda-ks.cfg  install.log  install.log.syslog  packer_0.8.6_linux_amd64.zip  VMware-ovftool-4.1.0-2459827-lin.x86_64.bundle
[root@packer-centos ~]# ./VMware-ovftool-4.1.0-2459827-lin.x86_64.bundle
Extracting VMware Installer...done.
You must accept the VMware OVF Tool component for Linux End User
License Agreement to continue.  Press Enter to proceed.
VMWARE END USER LICENSE AGREEMENT

PLEASE NOTE THAT THE TERMS OF THIS END USER LICENSE AGREEMENT SHALL GOVERN YOUR
USE OF THE SOFTWARE, REGARDLESS OF ANY TERMS THAT MAY APPEAR DURING THE
INSTALLATION OF THE SOFTWARE.

IMPORTANT-READ CAREFULLY:   BY DOWNLOADING, INSTALLING, OR USING THE SOFTWARE,
YOU (THE INDIVIDUAL OR LEGAL ENTITY) AGREE TO BE BOUND BY THE TERMS OF THIS END
USER LICENSE AGREEMENT ("EULA").  IF YOU DO NOT AGREE TO THE TERMS OF THIS
EULA, YOU MUST NOT DOWNLOAD, INSTALL, OR USE THE SOFTWARE, AND YOU MUST DELETE
OR RETURN THE UNUSED SOFTWARE TO THE VENDOR FROM WHICH YOU ACQUIRED IT WITHIN
THIRTY (30) DAYS AND REQUEST A REFUND OF THE LICENSE FEE, IF ANY, THAT YOU PAID
FOR THE SOFTWARE.
...

Do you agree? [yes/no]: yes

The product is ready to be installed.  Press Enter to begin
installation or Ctrl-C to cancel.

Installing VMware OVF Tool component for Linux 4.1.0
    Configuring...
[######################################################################] 100%
Installation was successful.
[root@packer-centos ~]# which ovftool
/usr/bin/ovftool
[root@packer-centos ~]# ovftool -v
VMware ovftool 4.1.0 (build-2459827)
[root@packer-centos ~]#
{{< /highlight >}}

### 7. Install apache web server on the CentOS virtual machine

{{< highlight bash >}}
[root@packer-centos ~]# yum install -y httpd
Loaded plugins: fastestmirror
Setting up Install Process
Loading mirror speeds from cached hostfile
 * base: yum.tamu.edu
 * epel: fedora-epel.mirror.lstn.net
 * extras: centos.firehosted.com
 * updates: mirrors.adams.net
Resolving Dependencies
--> Running transaction check
---> Package httpd.x86_64 0:2.2.15-47.el6.centos.1 will be installed
--> Processing Dependency: httpd-tools = 2.2.15-47.el6.centos.1 for package: httpd-2.2.15-47.el6.centos.1.x86_64
--> Processing Dependency: apr-util-ldap for package: httpd-2.2.15-47.el6.centos.1.x86_64
--> Processing Dependency: /etc/mime.types for package: httpd-2.2.15-47.el6.centos.1.x86_64
--> Processing Dependency: libaprutil-1.so.0()(64bit) for package: httpd-2.2.15-47.el6.centos.1.x86_64
--> Processing Dependency: libapr-1.so.0()(64bit) for package: httpd-2.2.15-47.el6.centos.1.x86_64
--> Running transaction check
---> Package apr.x86_64 0:1.3.9-5.el6_2 will be installed
---> Package apr-util.x86_64 0:1.3.9-3.el6_0.1 will be installed
---> Package apr-util-ldap.x86_64 0:1.3.9-3.el6_0.1 will be installed
---> Package httpd-tools.x86_64 0:2.2.15-47.el6.centos.1 will be installed
---> Package mailcap.noarch 0:2.1.31-2.el6 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

======================================================================
 Package        Arch       Version                  Repository   Size
======================================================================
Installing:
 httpd          x86_64     2.2.15-47.el6.centos.1   updates      830 k
Installing for dependencies:
 apr            x86_64     1.3.9-5.el6_2            base         123 k
 apr-util       x86_64     1.3.9-3.el6_0.1          base         87 k
 apr-util-ldap  x86_64     1.3.9-3.el6_0.1          base         15 k
 httpd-tools    x86_64     2.2.15-47.el6.centos.1   updates      77 k
 mailcap        noarch     2.1.31-2.el6             base         27 k

Transaction Summary
======================================================================
Install       6 Package(s)

Total download size: 1.1 M
Installed size: 3.6 M
Downloading Packages:
(1/6): apr-1.3.9-5.el6_2.x86_64.rpm                   | 123 kB  00:00
(2/6): apr-util-1.3.9-3.el6_0.1.x86_64.rpm            |  87 kB  00:00
(3/6): apr-util-ldap-1.3.9-3.el6_0.1.x86_64.rpm       |  15 kB  00:00
(4/6): httpd-2.2.15-47.el6.centos.1.x86_64.rpm        | 830 kB  00:00
(5/6): httpd-tools-2.2.15-47.el6.centos.1.x86_64.rpm  |  77 kB  00:00
(6/6): mailcap-2.1.31-2.el6.noarch.rpm                |  27 kB  00:00
----------------------------------------------------------------------
Total                                        711 kB/s | 1.1 MB  00:01
Running rpm_check_debug
Running Transaction Test
Transaction Test Succeeded
Running Transaction
  Installing : apr-1.3.9-5.el6_2.x86_64                         1/6
  Installing : apr-util-1.3.9-3.el6_0.1.x86_64                  2/6
  Installing : httpd-tools-2.2.15-47.el6.centos.1.x86_64        3/6
  Installing : apr-util-ldap-1.3.9-3.el6_0.1.x86_64             4/6
  Installing : mailcap-2.1.31-2.el6.noarch                      5/6
  Installing : httpd-2.2.15-47.el6.centos.1.x86_64              6/6
  Verifying  : httpd-tools-2.2.15-47.el6.centos.1.x86_64        1/6
  Verifying  : httpd-2.2.15-47.el6.centos.1.x86_64              2/6
  Verifying  : apr-util-ldap-1.3.9-3.el6_0.1.x86_64             3/6
  Verifying  : apr-1.3.9-5.el6_2.x86_64                         4/6
  Verifying  : mailcap-2.1.31-2.el6.noarch                      5/6
  Verifying  : apr-util-1.3.9-3.el6_0.1.x86_64                  6/6

Installed:
  httpd.x86_64 0:2.2.15-47.el6.centos.1

Dependency Installed:
  apr.x86_64 0:1.3.9-5.el6_2    apr-util.x86_64 0:1.3.9-3.el6_0.1    
  apr-util-ldap.x86_64 0:1.3.9-3.el6_0.1  httpd-tools.x86_64 0:2.2.15-47.el6.centos.1     
  mailcap.noarch 0:2.1.31-2.el6

Complete!
[root@packer-centos ~]#
{{< /highlight >}}

### 7. Remove the apache welcome.conf, to enable directory browsing, and start the httpd service

{{< highlight bash >}}
[root@packer-centos ~]# rm /etc/httpd/conf.d/welcome.conf
rm: remove regular file '/etc/httpd/conf.d/welcome.conf'? y
[root@packer-centos ~]# service httpd start
Starting httpd:                                            [  OK  ]
[root@packer-centos ~]#
{{< /highlight >}}

### 8. create the directory /var/www/html/box-files and test that the directory is visible in a browser

Create the /var/www/hmlt/box-files/ directory and an empty test-file.txt

{{< highlight bash >}}
[root@packer-centos ~]# mkdir /var/www/html/box-files
[root@packer-centos ~]# touch /var/www/html/box-files/test-file.txt
[root@packer-centos ~]#
{{< /highlight >}}

Open the ip address of the CentOS virtual machine in a browser and verify that the test-file.txt is visible

![screenshot](/static/02-browse-apache-box-file-directory.png)  

Delete /var/www/html/box-files/test-file.txt, since it will not be needed:

{{< highlight bash >}}
[root@packer-centos ~]# rm /var/www/html/box-files/test-file.txt
rm: remove regular empty file '/var/www/html/box-files/test-file.txt'? y
[root@packer-centos ~]#
{{< /highlight >}}

#### That all for this post covering how to install Packer, ovftool and Apache web server on a CentOS virtual machine. In the next post we will create a Packer template that uses what we have setup so far.

---

#### Please provide any feedback or suggestions to my twitter account located on the about page.
