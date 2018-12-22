+++
title = "Installing Vagrant and the vagrant-vsphere plugin on CentOS 6.x"
description = ""
tags = [
    "vagrant",
    "vagrant-vsphere",
    "vsphere",
]
date = "2014-04-19"
categories = [
    "vagrant",
    "vagrant-vsphere",
    "vsphere",
]
highlight = "true"
+++

Some readers might find this post to be a little "in the weeds" and cover details they are already familiar with. I struggled in my first attempts to get Vagrant working on CentOS, because I couldn't find any good tutorials that covered the entire process. If you get bored with basic network configuration or have another preferred method for installing Ruby, please understand I'm trying to provide as much detail as possible to those without much linux experience. I am also assuming you have a working vCenter, ESXi hosts and dhcp server configured to assign IP addresses for the Vagrant virtual machines we will be deploying.

### 1. Create a CentOS 6.x minimal virtual machine for installing Vagrant. 
Create or clone a fresh CentOS 6.x minimal virtual machine. I already had an existing Centos 6.5 minimal template that I simply cloned for this purpose.

### 2. Power on the new cloned vm and connect to the virtual machine console

### 3. Modify /etc/sysconfig/network, /etc/sysconfig/network-scripts/ifcfg-eth0 & /etc/resolv.conf to reflect the network settings for your vagrant vm:

{{< highlight bash >}}
[root@vagrant ~]# cat /etc/sysconfig/network 
NETWORKING=yes 
HOSTNAME=vagrant.mylab.net 
GATEWAY=192.168.1.1 
[root@vagrant ~]# cat /etc/sysconfig/network-scripts/ifcfg-eth0 
DEVICE=eth0 
TYPE=Ethernet 
ONBOOT=yes 
NM_CONTROLLED=no 
BOOTPROTO=static 
IPADDR=192.168.1.40 
NETMASK=255.255.255.0 
[root@vagrant ~]# cat /etc/resolv.conf
 search mylab.net 
nameserver 192.168.1.1 
nameserver 8.8.8.8 
nameserver 8.8.4.4
{{< /highlight >}}

### 4. Restart the vm for the hostname change to take affect

### 5. SSH to the vagrant vm using the IP address you assigned in /etc/sysconf/network-scripts/ifcfg-eth0

### 6. Install Ruby 1.9.3. There are [several ways](http://www.agilesysadmin.net/acquiring-a-modern-ruby-part-one) to install ruby 1.9.3 on CentOS, but I'll cover using ruby-build to accomplish this:

* Use yum to install the packages we'll need to get ruby, rubygems and vagrant installed:

{{< highlight bash >}}
[root@vagrant ~]# yum install -y gcc-c++ glibc-headers openssl-devel readline libyaml-devel readline-devel zlib zlib-devel iconv-devel libxml2 libxml2-devel libxslt libxslt-devel wget git
{{< /highlight >}}

* Pull down the latest version of ruby-build using git:

{{< highlight bash >}}
[root@vagrant ~]# git clone https://github.com/sstephenson/ruby-build.git
{{< /highlight >}}

* Change to the directory the previous command created:

{{< highlight bash >}}
[root@vagrant ~]# cd ruby-build/
{{< /highlight >}}

* Run the 'install.sh' script to install ruby-build:

{{< highlight bash >}}
[root@vagrant ruby-build]# ./install.sh
{{< /highlight >}}

* I have not been successful in getting vagrant-vsphere to run on any version of ruby newer than 1.9.3-p545. As a result I would recommend installing that version with the following command:

{{< highlight bash >}}
[root@vagrant ruby-build]# ruby-build --definitionsruby-build 1.9.3-p545 /usr/local/
Downloading yaml-0.1.6.tar.gz...
-> http://dqw8nmjcqpjn7.cloudfront.net/5fe00cda18ca5daeb43762b80c38e06e
Installing yaml-0.1.6...
Installed yaml-0.1.6 to /usr/local/

Downloading ruby-1.9.3-p545.tar.gz...
-> http://dqw8nmjcqpjn7.cloudfront.net/8e8f6e4d7d0bb54e0edf8d9c4120f40c
Installing ruby-1.9.3-p545...

Installed ruby-1.9.3-p545 to /usr/local/
{{< /highlight >}}

### 7. We will next need to install rubygems. The process I used was to download the latest version from [rubygems.org](https://rubygems.org/pages/download) and install it:

* Use wget to download the latest version listed on the rubygems download page:

{{< highlight bash >}}
[root@vagrant ruby-build]# cd ~/
[root@vagrant ~]# wget http://production.cf.rubygems.org/rubygems/rubygems-2.2.2.tgz
{{< /highlight >}}

* Use tar to unzip the tar-gzipped file we downloaded:

{{< highlight bash >}}
[root@vagrant ~]# tar xvzf rubygems-2.2.2.tgz
[root@vagrant ~]# cd rubygems-2.2.2
{{< /highlight >}}

* Install rubygems by running the 'setup.rb' script:

{{< highlight bash >}}
[root@vagrant rubygems-2.2.2]# ruby setup.rb
{{< /highlight >}}

### 8. Installing Vagrant

* Use wget to download the latest version listed on the [Vagrant download](https://www.vagrantup.com/downloads.html) page:

{{< highlight bash >}}
[root@vagrant rubygems-2.2.2]# cd ~/
[root@vagrant ~]# wget https://dl.bintray.com/mitchellh/vagrant/vagrant_1.5.2_x86_64.rpm
{{< /highlight >}}

* Use the 'rpm' command to install the RPM package we downloaded in the previous step:

{{< highlight bash >}}
[root@vagrant ~]# rpm -i vagrant_1.5.2_x86_64.rpm
{{< /highlight >}}

* Export the directory that the vagrant executable was install to, in order to make it easier to run:

{{< highlight bash >}}
[root@vagrant ~]# export PATH=$PATH:/opt/vagrant/bin/
{{< /highlight >}}

### 9. Installing the vagrant-vsphere plugin

* The nokogiri gem, which is a ruby HTML, XML & SAM parser, is required by vagrant-vsphere so we will install it with the following commands:

{{< highlight bash >}}
[root@vagrant ~]# gem install nokogiri
Building native extensions.  This could take a while...
Successfully installed nokogiri-1.6.1
unable to convert "\x89" from ASCII-8BIT to UTF-8 for ext/nokogiri/tmp/x86_64-redhat-linux/ports/libxslt/1.1.26/libxslt-1.1.26/tests/xmlspec/logo-REC, skipping
unable to convert "\xF6" from ASCII-8BIT to UTF-8 for ext/nokogiri/tmp/x86_64-redhat-linux/ports/libxslt/1.1.26/libxslt-1.1.26/libxslt/xslt.c, skipping
unable to convert "\xF6" from ASCII-8BIT to UTF-8 for ext/nokogiri/tmp/x86_64-redhat-linux/ports/libxslt/1.1.26/libxslt-1.1.26/ChangeLog, skipping
unable to convert "\xE1" from ASCII-8BIT to UTF-8 for ext/nokogiri/tmp/x86_64-redhat-linux/ports/libxslt/1.1.26/libxslt-1.1.26/NEWS, skipping
unable to convert "\xFD" from ASCII-8BIT to UTF-8 for ext/nokogiri/tmp/x86_64-redhat-linux/ports/libxslt/1.1.26/libxslt-1.1.26/win32/Readme.txt, skipping
unable to convert "\xC0" from ASCII-8BIT to UTF-8 for ext/nokogiri/tmp/x86_64-redhat-linux/ports/libxml2/2.8.0/libxml2-2.8.0/test/XInclude/ents/isolatin.txt, skipping
unable to convert "\xE4" from ASCII-8BIT to UTF-8 for ext/nokogiri/tmp/x86_64-redhat-linux/ports/libxml2/2.8.0/libxml2-2.8.0/doc/examples/testWriter.c, skipping
unable to convert "\xF8" from ASCII-8BIT to UTF-8 for ext/nokogiri/tmp/x86_64-redhat-linux/ports/libxml2/2.8.0/libxml2-2.8.0/testapi.c, skipping
unable to convert "\xE9" from ASCII-8BIT to UTF-8 for ext/nokogiri/tmp/x86_64-redhat-linux/ports/libxml2/2.8.0/libxml2-2.8.0/runtest.c, skipping
unable to convert "\xF8" from ASCII-8BIT to UTF-8 for ext/nokogiri/tmp/x86_64-redhat-linux/ports/libxml2/2.8.0/libxml2-2.8.0/entities.c, skipping
unable to convert "\xC0" from ASCII-8BIT to UTF-8 for ports/x86_64-redhat-linux/libxslt/1.1.26/bin/xsltproc, skipping
unable to convert "\xE4" from ASCII-8BIT to UTF-8 for ports/x86_64-redhat-linux/libxml2/2.8.0/share/doc/libxml2-2.8.0/html/testWriter.c, skipping
unable to convert "\xC0" from ASCII-8BIT to UTF-8 for ports/x86_64-redhat-linux/libxml2/2.8.0/bin/xmllint, skipping
unable to convert "\xC0" from ASCII-8BIT to UTF-8 for ports/x86_64-redhat-linux/libxml2/2.8.0/bin/xmlcatalog, skipping
1 gem installed
{{< /highlight >}}

* Next we will install the rbvmomi gem with the following command. rbvmomi is a ruby interface for using the vSphere API.

{{< highlight bash >}}
[root@vagrant ~]# gem install rbvmomi
Successfully installed rbvmomi-1.8.1
1 gem installed
{{< /highlight >}}

* Now we can install the vagrant-vsphere plugin with the following command:

{{< highlight bash >}}
[root@vagrant ~]# vagrant plugin install vagrant-vsphere
Installing the 'vagrant-vsphere' plugin. This can take a few minutes...
Installed the plugin 'vagrant-vsphere (0.8.1)'!
{{< /highlight >}}

### 8. Configuring vagrant-vsphere 
The vagrant-vsphere [README.md on github](https://github.com/nsidc/vagrant-vsphere) mentions we need to create a vSphere box from the metadata.json located in the example_box directory. 

* We can use the linux the "find" command to determine the path to the example_box directory:

{{< highlight bash >}}
[root@vagrant ~]# find / -name example_box
/root/.vagrant.d/gems/gems/vagrant-vsphere-0.8.1/example_box
{{< /highlight >}}

* Now we can change directory to that directory and create the necessary dummy box file:

{{< highlight bash >}}
[root@vagrant ~]# cd /root/.vagrant.d/gems/gems/vagrant-vsphere-0.8.1/example_box
[root@vagrant example_box]# tar cvzf dummy.box ./metadata.json
./metadata.json
{{< /highlight >}}

* Next we will create a new folder under /root for storing the box file we created and other vagrant configuration files

{{< highlight bash >}}
[root@vagrant example_box]# mkdir -p ~/vagrant-vms/example_box
[root@vagrant example_box]# mv dummy.box ~/vagrant-vms/example_box
{{< /highlight >}}

* Vagrant reads it's configuration from a file named Vagrantfile, located in the directory from which the "vagrant up" command is run in. We need to go back to our "vagrant-vms" folder and create this file:

{{< highlight bash >}}
[root@vagrant example_box]# cd ~/vagrant-vms/
[root@vagrant vagrant-vms]# touch Vagrantfile
{{< /highlight >}}

* Modify the Vagrantfile to reflect your vsphere configuration. Here's an example of my working Vagrantfile:

{{< highlight bash >}}
[root@vagrant vagrant-vms]# cat Vagrantfile

Vagrant.configure("2") do |config|
  config.vm.box = 'dummy'
  config.vm.box_url = './example_box/dummy.box'

  config.vm.provider :vsphere do |vsphere|
    vsphere.host = 'vcenter.mylab.net'
    vsphere.name = 'vagrant-test'
    vsphere.clone_from_vm = true
    vsphere.template_name = 'Centos-6.5'
    vsphere.user = 'root@localos'
    vsphere.password = 'S0meR@nd0mP@ssw0rd'
    vsphere.insecure = true
  end
end
{{< /highlight >}}

In this configuration file:

* "vsphere.host" is your vCenter server FQDN or IP address
* "vsphere.name" will be the name of the virtual machine that gets created
* "vsphere.clone\_from\_vm" specifies we will be cloning a vm rather than template
* "vsphere.template_name" is the name of the vm we will be cloning
* "vsphere.user" is the username we will be using to connect to vCenter
* "vsphere.password" is the password we will be using to connect ot vCenter
* "vsphere.insecure" tells vagrant to not worry about validating the certificate on the vCenter server

### 9. Vagrant up!!!!
If everything has gone correctly up to this point we can issue a "vagrant up --provider=vsphere" command from the directory that contains the Vagrantfile:

{{< highlight bash >}}
[root@vagrant vagrant-vms]# vagrant up --provider=vsphere
Bringing machine 'default' up with 'vsphere' provider...
==> default: Calling vSphere CloneVM with the following settings:
==> default:  -- Source VM: CentOS-6.5
==> default:  -- Name: vagrant-test
==> default: Waiting for SSH to become available...
{{< /highlight >}}

You should see a "Clone virtual machine" task being run on your vCenter server and then completing.

### 10. Canceling the "Vagrant up" command and destroying our cloned vm.
Since we have not properly prepared the vm we are cloning (I will have another blog post covering this topic) Vagrant will never be able to successfully connect to the cloned vm using SSH. As a result we will need to cancel the "vagrant up" command:

* Press [control-c] to attempt to gracefully exit "vagrant-up"

{{< highlight bash >}}
^C==> default: Waiting for cleanup before exiting...
{{< /highlight >}}

I've never seen this task complete so I will press [control-c] again to immediately exit:

{{< highlight bash >}}
^C==> default: Exiting immediately, without cleanup!
{{< /highlight >}}

* Destroy the cloned vm by running "vagrant destroy" command:

{{< highlight bash >}}
[root@vagrant vagrant-vms]# vagrant destroy
==> default: Calling vSphere PowerOff
==> default: Calling vShpere Destroy
{{< /highlight >}}

You should see a "Power Off virtual machine" and "Delet virtual machine" tasks being run and then completing on your vCenter server.

#### Hopefully you found this post helpful in getting vagrant and the vagrant-vsphere plugin installed and configured. The next blog post will be covering how to create a template vm that has been prepared for vagrant to connect seemlessly using SSH.

---

#### Please provide any feedback or suggestions to my twitter account located on the about page.

