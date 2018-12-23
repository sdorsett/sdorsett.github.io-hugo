+++
title = "Installing vagrant and the vagrant-vcenter provider on CentOS 6.5"
description = ""
tags = [
    "vagrant",
    "vagrant-vcenter",
]
date = "2015-01-05"
categories = [
    "vagrant",
    "vagrant-vcenter",
]
topics = [
    "using-vagrant-with-vsphere"
]
highlight = "true"
+++

If you followed the steps in one of the two previous posts, you have a ESXi .box template in the vagrant-vmware-ovf format. This format allows for deploying the exact same template to vCenter, vCloud Director or vCloud Air, by simply specifying a different provider in vagrant. This post will cover deploying to vCenter, since that is the most readily available of the three. 


In this post we will again talk about the following helpful gosddc project:

* [gosddc/vagrant-vcenter](https://github.com/gosddc/vagrant-vcenter).
   This repo contains a vagrant plugin (provider) that will deploy a vagrant-vmware-ovf generated .box template to a vcenter instance. This provider allows vagrant to seamlessly deploy to vCenter.

We will need a virtual machine with a minimal install of CentOS 6.5 to install vagrant or you can use the same CentOS virtual machine we used <a href="../2015-01-03-using-packer-on-centos">in the last post</a>. For convenience I will be using the same virtual machine.

### 1. Let's get started by installing the necessary dependancies.
* Run the following command to install the some needed packages:

{{< highlight bash >}}
[root@vagrant ~]# yum install -y gcc-c++ glibc-headers openssl-devel readline libyaml-devel readline-devel zlib zlib-devel iconv-devel libxml2 libxml2-devel libxslt libxslt-devel wget git unzip
{{< /highlight >}}

* Download and install ruby-build:

{{< highlight bash >}}
[root@vagrant ~]# git clone https://github.com/sstephenson/ruby-build.git
Initialized empty Git repository in /root/ruby-build/.git/
remote: Counting objects: 4212, done.
remote: Compressing objects: 100% (16/16), done.
remote: Total 4212 (delta 4), reused 2 (delta 0)
Receiving objects: 100% (4212/4212), 761.14 KiB | 730 KiB/s, done.
Resolving deltas: 100% (2142/2142), done.
[root@vagrant ~]# cd ruby-build/
[root@vagrant ruby-build]# ./install.sh
{{< /highlight >}}

* Running the "ruby-build --definitions" command will provide an output of the versions of Ruby that ruby-buid supports: 

{{< highlight bash >}}
[root@vagrant ruby-build]# ruby-build --definitions
1.8.6-p383
1.8.6-p420
1.8.7-p249
1.8.7-p302
1.8.7-p334
1.8.7-p352
1.8.7-p357
1.8.7-p358
1.8.7-p370
1.8.7-p371
1.8.7-p374
1.8.7-p375
1.9.1-p378
1.9.1-p430
1.9.2-p0
1.9.2-p180
1.9.2-p290
1.9.2-p318
1.9.2-p320
1.9.2-p326
1.9.2-p330
1.9.3-dev
1.9.3-preview1
1.9.3-rc1
1.9.3-p0
1.9.3-p125
1.9.3-p194
1.9.3-p286
1.9.3-p327
1.9.3-p362
1.9.3-p374
1.9.3-p385
1.9.3-p392
1.9.3-p429
1.9.3-p448
1.9.3-p484
1.9.3-p545
1.9.3-p547
1.9.3-p550
1.9.3-p551
2.0.0-dev
2.0.0-preview1
2.0.0-preview2
...
{{< /highlight >}}

* Install ruby 1.9.3 build 551 using the following command:

{{< highlight bash >}}
[root@vagrant ruby-build]# ruby-build 1.9.3-p551 /usr/local/
Downloading yaml-0.1.6.tar.gz...
-> http://dqw8nmjcqpjn7.cloudfront.net/7da6971b4bd08a986dd2a61353bc422362bd0edcc67d7ebaac68c95f74182749
Installing yaml-0.1.6...
Installed yaml-0.1.6 to /usr/local/

Downloading ruby-1.9.3-p551.tar.gz...
-> http://dqw8nmjcqpjn7.cloudfront.net/bb5be55cd1f49c95bb05b6f587701376b53d310eb1bb7c76fbd445a1c75b51e8
Installing ruby-1.9.3-p551...
Installed ruby-1.9.3-p551 to /usr/local/
{{< /highlight >}}

* Download and install the latest version of ruby gems. You can always find the URL to the most recent version at [rubygems.org](https://rubygems.org/pages/download).

{{< highlight bash >}}
[root@vagrant ruby-build]# cd ~/
[root@vagrant ~]# wget http://production.cf.rubygems.org/rubygems/rubygems-2.4.5.tgz
--2015-01-02 21:14:39-- http://production.cf.rubygems.org/rubygems/rubygems-2.4.5.tgz
Resolving production.cf.rubygems.org... 54.230.6.155, 54.230.6.120, 54.230.7.122, ...
Connecting to production.cf.rubygems.org|54.230.6.155|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 446665 (436K) [application/x-tar]
Saving to: "rubygems-2.4.5.tgz"

100%[==================================================================================================>] 446,665 1.78M/s in 0.2s 

2015-01-02 21:14:45 (1.78 MB/s) - "rubygems-2.4.5.tgz" saved [446665/446665]

[root@vagrant ~]# tar -zxf rubygems-2.4.5.tgz
[root@vagrant ~]# cd rubygems-2.4.5
[root@vagrant rubygems-2.4.5]# ruby setup.rb
RubyGems 2.4.5 installed
Installing ri documentation for rubygems-2.4.5
{{< /highlight >}}

### 2. Create a directory to keep our vagrant-vmware-ovf .box template to and copy the template we previously created to it:

{{< highlight bash >}}
[root@vagrant esxi55]# mkdir ~/box-files/
[root@vagrant esxi55]# cp /root/packer/packer-templates/esxi55/esxi55-vmware_ovf-1.1.box ~/box-files/
{{< /highlight >}}

### 3. The next thing we need to do is download and install vagrant and the vagrant-vcenter provider:

* Download and install the latest version of vagrant. You can find the URL to the latest versions of the linux 64bit .rpm at [vagrantup.com](https://www.vagrantup.com/downloads.html).

{{< highlight bash >}}
[root@vagrant rubygems-2.4.5]# cd ~/
[root@vagrant ~]# wget https://dl.bintray.com/mitchellh/vagrant/vagrant_1.7.1_x86_64.rpm
--2015-01-03 16:02:34-- https://dl.bintray.com/mitchellh/vagrant/vagrant_1.7.1_x86_64.rpm
Resolving dl.bintray.com... 108.168.194.92, 108.168.194.91
Connecting to dl.bintray.com|108.168.194.92|:443... connected.
HTTP request sent, awaiting response... 302 
Resolving d29vzk4ow07wi7.cloudfront.net... 54.230.7.101, 54.230.5.129, 54.230.6.222, ...
Connecting to d29vzk4ow07wi7.cloudfront.net|54.230.7.101|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 65197711 (62M) [application/unknown]
Saving to: "vagrant_1.7.1_x86_64.rpm"

100%[==================================================================================================>] 65,197,711 1.92M/s in 33s 

2015-01-03 16:03:18 (1.87 MB/s) - "vagrant_1.7.1_x86_64.rpm" saved [65197711/65197711]

[root@vagrant ~]# rpm -i vagrant_1.7.1_x86_64.rpm
[root@vagrant ~]# vagrant -v
Vagrant 1.7.1

{{< /highlight >}}

* Install the vagrant-vcenter plugin using the following commands: 

{{< highlight bash >}}
[root@vagrant ~]# vagrant plugin list
vagrant-share (1.1.4, system)
[root@vagrant ~]# vagrant plugin install vagrant-vcenter
Installing the 'vagrant-vcenter' plugin. This can take a few minutes...
Installed the plugin 'vagrant-vcenter (0.3.2)'!
[root@vagrant ~]# 
{{< /highlight >}}

* Create directory and Vagrantfile for the ESXi virtual machines we will deploy with vagrant:
 
{{< highlight bash >}}
[root@vagrant ~]# mkdir -p ~/vagrant-vms/esxi-test/
[root@vagrant ~]# cd vagrant-vms/esxi-test/
[root@vagrant esxi-test]# vi Vagrantfile
[root@vagrant esxi-test]# cat Vagrantfile
esxi_box_url = '/root/box-files/esxi55-vmware_ovf-1.1.box'
 
nodes = [
  { :hostname => 'esx-01a', :box => 'esxi55', :box_url => esxi_box_url},
  { :hostname => 'esx-02a', :box => 'esxi55', :box_url => esxi_box_url},
]
 
Vagrant.configure('2') do |config|
 
  config.vm.provider :vcenter do |vcenter|
    vcenter.hostname = '192.168.1.195'
    vcenter.username = 'root'
    vcenter.password = 'mySecretP@ssw0rd'
    vcenter.folder_name = 'Vagrant/Deployed'
    vcenter.datacenter_name = 'datacenter-01'
    vcenter.computer_name = 'cluster-01'
    vcenter.datastore_name = 'vsanDatastore'
    vcenter.template_folder_name = 'Vagrant/Templates'
    vcenter.network_name = 'vlan2'
    vcenter.linked_clones = true
    vcenter.enable_vm_customization = false
  end
 
  # Go through nodes and configure each of them.j
  nodes.each do |node|
    config.vm.define node[:hostname] do |node_config|
 
      if node[:hostname].include? 'esx-'
        node_config.ssh.username = 'root'
        node_config.ssh.shell = 'sh'
        node_config.ssh.insert_key = false
        node_config.vm.synced_folder '.', '/vagrant', disabled: true
      end
 
      node_config.vm.box = node[:box]
      node_config.vm.hostname = node[:hostname]
      node_config.vm.box_url = node[:box_url]
    end
  end
end
{{< /highlight >}}

This Vagrantfile is specifying vagrant to use the vagrant-vcenter profiler to create two virtual machines (esx-01a & esx-02a) from the esx55-vmware_ovf-1.1.box file. You will need update the following properties to reflect your own vCenter configuration:

* vcenter.hostname = the IP address of your vCenter server
* vcenter.username = the username used to connect to your vCenter server
* vcenter.password = the password used to connect to your vCenter server
* vcenter.datacenter_name = the vCenter virtual datacenter to use for virtual machine deployment
* vcenter.computer_name = the vCenter host or cluster to use for virtual machine deployment. I've used the cluster in my example.
* vcenter.datastore_name = the vCenter datastore to use for virtual machine deployment
* vcenter.folder_name = the vm folder that the virtual machines will be deployed to
* vcenter.template\_folder\_name = the vm folder that the ESXi template will be created in
* vcenter.network_name = the vCenter portgroup to connect the ESXi templates to. There should be a DHCP server on this portgroup to provide IP addresses to the deployed ESXi templates.

### 4. "vagrant up"

* Now that we have created a Vagrantfile config file and updated it with our vCenter information we can bring up our virtual machine by running "vagrant up" in the directory with the Vagrantfile.

{{< highlight bash >}}
[root@vagrant esxi-test]# vagrant up
Bringing machine 'esx-01a' up with 'vcenter' provider...
Bringing machine 'esx-02a' up with 'vcenter' provider...
==> esx-01a: Box 'esxi55' could not be found. Attempting to find and install...
 esx-01a: Box Provider: vmware_ovf, vcloud, vcenter
 esx-01a: Box Version: >= 0
==> esx-01a: Adding box 'esxi55' (v0) for provider: vmware_ovf, vcloud, vcenter
 esx-01a: Downloading: file:///root/box-files/esxi55-vmware_ovf-1.1.box
==> esx-01a: Successfully added box 'esxi55' (v0) for 'vmware_ovf'!
==> esx-02a: Box 'esxi55' could not be found. Attempting to find and install...
 esx-02a: Box Provider: vmware_ovf, vcloud, vcenter
 esx-02a: Box Version: >= 0
==> esx-02a: Adding box 'esxi55' (v0) for provider: vmware_ovf, vcloud, vcenter
==> esx-02a: Uploading [esxi55]...
==> esx-02a: Adding [esxi55]
2015-01-03 16:28:34 -0600: networks: vlan2 = vlan2
2015-01-03 16:28:34 -0600: Uploading OVF to esx02.tyrell.corp...
==> esx-01a: Uploading [esxi55]...
==> esx-01a: Adding [esxi55]
2015-01-03 16:28:35 -0600: networks: vlan2 = vlan2
2015-01-03 16:28:35 -0600: Uploading OVF to esx01.tyrell.corp...
DEBUG: Timeout: 300
Iteration 1: Trying to get host's IP address ...
 % Total % Received % Xferd Average Speed Time Time Time Current
 Dload Upload Total Spent Left Speed
 15 473M 15 75.3M 0 0 4554k 0 0:01:46 0:00:16 0:01:30 4552k2015-01-03 16:29:02 -0600: Template already exists, waiting for it to be ready
 20 473M 20 98.8M 0 0 5122k 0 0:01:34 0:00:19 0:01:15 7489k2015-01-03 16:29:05 -0600: Template VM found
100 473M 100 473M 0 0 9245k 0 0:00:52 0:00:52 --:--:-- 9.9M
Iteration 1: Trying to access nfcLease.info.entity ...
HttpNfcLeaseComplete succeeded
esxi_box_url = '/root/box-files/esxi55-vmware_ovf-1.1.box'
==> esx-02a: Creating VM...
2015-01-03 16:29:42 -0600: Template fully prepared and ready to be cloned
==> esx-01a: Creating VM...
==> esx-01a: Powering on VM...
==> esx-02a: Powering on VM...

{{< /highlight >}} 

* Once the "vagrant up" command completes, you can check the status of the vms using "vagrant status":

{{< highlight bash >}}
[root@vagrant esxi-test]# vagrant status
Current machine states:

esx-01a running (vcenter)
esx-02a running (vcenter)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
{{< /highlight >}}

* Looking in vCenter you will see the virtual machines that vagrant deployed:

![screenshot](/static/01-esx-01a.png)

* You can now ssh into either of the ESXi virtual machines using "vagrant ssh [vm_name]":

  To connect to esx-01a you simply run "vagrant ssh esx-01a":

{{< highlight bash >}}
[root@vagrant esxi-test]# vagrant ssh esx-01a
==> esx-01a: External IP for esx-01a: 192.168.1.177
The time and date of this login have been sent to the system logs.

VMware offers supported, powerful system administration tools. Please
see www.vmware.com/go/sysadmintools for details.

The ESXi Shell can be disabled by an administrative user. See the
vSphere Security documentation for more information.

~ # esxcli network ip interface list
vmk0
 Name: vmk0
 MAC Address: 00:50:56:6f:9d:66
 Enabled: true
 Portset: vSwitch0
 Portgroup: Management Network
 Netstack Instance: defaultTcpipStack
 VDS Name: N/A
 VDS UUID: N/A
 VDS Port: N/A
 VDS Connection: -1
 MTU: 1500
 TSO MSS: 65535
 Port ID: 33554437
~ # exit
Connection to 192.168.1.177 closed.
[root@vagrant esxi-test]#
{{< /highlight >}}

  The same can be done for esx-02a:

{{< highlight bash >}}
[root@vagrant esxi-test]# vagrant ssh esx-02a
==> esx-02a: External IP for esx-02a: 192.168.1.178
The time and date of this login have been sent to the system logs.

VMware offers supported, powerful system administration tools. Please
see www.vmware.com/go/sysadmintools for details.

The ESXi Shell can be disabled by an administrative user. See the
vSphere Security documentation for more information.
~ # esxcli network ip interface list
vmk0
 Name: vmk0
 MAC Address: 00:50:56:67:d5:b7
 Enabled: true
 Portset: vSwitch0
 Portgroup: Management Network
 Netstack Instance: defaultTcpipStack
 VDS Name: N/A
 VDS UUID: N/A
 VDS Port: N/A
 VDS Connection: -1
 MTU: 1500
 TSO MSS: 65535
 Port ID: 33554437
~ # exit
Connection to 192.168.1.178 closed.
[root@vagrant esxi-test]#
{{< /highlight >}}

* Once we are done using our virtual machines they can be destroyed with a "vagrant destroy" command:

{{< highlight bash >}}
[root@vagrant esxi-test]# vagrant destroy
    esx-02a: Are you sure you want to destroy the 'esx-02a' VM? [y/N] y
==> esx-02a: Powering off VM...
==> esx-02a: Destroying VM...
    esx-01a: Are you sure you want to destroy the 'esx-01a' VM? [y/N] y
==> esx-01a: Powering off VM...
==> esx-01a: Destroying VM...
[root@vagrant esxi-test]# 
{{< /highlight >}}

* If you are ever unsure of the state of the vagrant virtual machines you can run "vagrant status" to check:

{{< highlight bash >}}
[root@vagrant esxi-test]# vagrant status
Current machine states:

esx-01a                   not created (vcenter)
esx-02a                   not created (vcenter)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
[root@vagrant esxi-test]# 
{{< /highlight >}}

#### That all for this post covering the basics of getting vagrant and the vagrant-vcenter provider installed/configured on CentOS. 

---

#### Please provide any feedback or suggestions to my twitter account located on the about page.

