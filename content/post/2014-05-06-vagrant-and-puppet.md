+++
title = "Creating a Puppet manifest and integrating it with Vagrant"
description = ""
tags = [
    "vagrant",
    "puppet",
]
date = "2014-05-06"
categories = [
    "vagrant",
    "puppet",
]
highlight = "true"
+++

This post will cover configuring Vagrant to automatically run a Puppet manifest on the vm created by "vagrant up." This capability allows you to test your Puppet manifests, make changes and test again, all quickly and easily. Let's get started:

### 1. Create the Puppet manifest & modules we will be using for our Vagrant tests.

For testing purposes we will be creating a Puppet manifest that ensures NTP is installed and is configured to use the following NTP servers:

<pre>
0.pool.ntp.org
1.pool.ntp.org
2.pool.ntp.org
</pre>

* First we should install tree, since it will help us visualize the directory structure of the puppet manifest directories we will be creating:

{{< highlight bash >}}
[root@vagrant vagrant-vms]# yum install -y tree
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirrors.centarra.com
 * epel: mirror.unl.edu
 * extras: mirrors.finalasp.com
 * updates: mirrors.centarra.com
Setting up Install Process
Resolving Dependencies
--> Running transaction check
---> Package tree.x86_64 0:1.5.3-2.el6 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

====================================================================
 Package      Arch           Version             Repository    Size
====================================================================
Installing:
 tree         x86_64         1.5.3-2.el6         base          36 k

Transaction Summary
====================================================================
Install       1 Package(s)

Total download size: 36 k
Installed size: 65 k
Downloading Packages:
tree-1.5.3-2.el6.x86_64.rpm                    |  36 kB     00:00     
Running rpm_check_debug
Running Transaction Test
Transaction Test Succeeded
Running Transaction
  Installing : tree-1.5.3-2.el6.x86_64                          1/1 
  Verifying  : tree-1.5.3-2.el6.x86_64                          1/1 

Installed:
  tree.x86_64 0:1.5.3-2.el6                                           

Complete!
[root@vagrant vagrant-vms]#
{{< /highlight >}}

* Change to the directory that contains the Vagrantfile & example_box directory we've been working in over the last few blog posts:

{{< highlight bash >}}
[root@vagrant ~]# cd ~/vagrant-vms/
[root@vagrant vagrant-vms]# tree
.
├── example_box
│   └── dummy.box
└── Vagrantfile

1 directory, 2 files
[root@vagrant vagrant-vms]#  
{{< /highlight >}}

* Create the following modules & manifests directory structure for our Puppet manifest files:

{{< highlight bash >}}
[root@vagrant vagrant-vms]# mkdir -p {manifests,modules/ntp/manifests,modules/ntp/templates,modules/common/manifests}
[root@vagrant vagrant-vms]# tree
.
├── example_box
│   └── dummy.box
├── manifests
├── modules
│   ├── common
│   │   └── manifests
│   └── ntp
│       ├── manifests
│       └── templates
└── Vagrantfile

8 directories, 2 files
[root@vagrant vagrant-vms]# 
{{< /highlight >}}
* Create manifests/site.pp with the following contents:

{{< highlight bash >}}
[root@vagrant vagrant-vms]# vi manifests/site.pp
node default {
  include common
    class {'ntp':}
}
[root@vagrant vagrant-vms]# 
{{< /highlight >}}

This is the Puppet manifest that we will configure Vagrent to run automatically. The "node default" section will be applied by any hostname that runs this manifest. The "node default" section calls the common & ntp modules which we will create next.

* Create modules/common/manifests/init.pp with the following contents:

{{< highlight bash >}}
[root@vagrant vagrant-vms]# cat modules/common/manifests/init.pp
class common {
  include common::data
}

class common::data {
  $ntpServerList = [ '0.pool.ntp.org','1.pool.ntp.org','2.pool.ntp.org' ]
}
[root@vagrant vagrant-vms]# 
{{< /highlight >}}

This manifest is what gets run when Puppets runs "include common" in the manifests/site.pp manifest. When this manifest is run it creates an array named "common::data::ntpServerList" containing the ntp servers we're needing to have defined.

* Create modules/ntp/manifests/init.pp with the following contents:

{{< highlight bash >}}
[root@vagrant vagrant-vms]# cat modules/ntp/manifests/init.pp
class ntp( $ntpServerList = $common::data::ntpServerList) {
  package { 'ntp':
  ensure => 'present',
  } #package

  file { '/etc/ntp.conf':
    mode    => "644",
    content => template("ntp/client-ntp.conf.erb"),
    notify  => Service["ntpd"],
    require => Package["ntp"],
  } # file

  service { 'ntpd':
    ensure  => running,
    enable  => true,
    require => Package["ntp"],
  } # service
}
[root@vagrant vagrant-vms]# 
{{< /highlight >}}

This manifest is what gets run when Puppet calls "class {'ntp':}" in the manifests/site.pp manifest. When this manifest is run it will ensure the ntp package is installed, the file "/etc/ntp.conf" is created containing our list of ntp servers and the ntpd service is started.    
* Create modules/ntp/templates/client-ntp.conf.erb with the following contents:

{{< highlight bash >}}
# This file is being maintained by Puppet.
# DO NOT EDIT

# Permit time synchronization with our time source, but do not
# permit the source to query or modify the service on this system.
restrict default kod nomodify notrap nopeer noquery
restrict -6 default kod nomodify notrap nopeer noquery

# Permit all access over the loopback interface.  This could
# be tightened as well, but to do so would effect some of
# the administrative functions.
restrict 127.0.0.1
restrict -6 ::1

<% ntpServerList.each do |ntpServer| -%>
server <%= ntpServer %>
<% end -%>

# Drift file.  Put this in a directory which the daemon can write to.
# No symbolic links allowed, either, since the daemon updates the file
# by creating a temporary in the same directory and then rename()'ing
# it to the file.
driftfile /var/lib/ntp/drift
[root@vagrant vagrant-vms]# 
{{< /highlight >}}

This .erb file is a ruby template describing what the file /etc/ntp.conf should contain. The magic of this is the section:

{{< highlight bash >}}
<% ntpServerList.each do |ntpServer| -%>
server <%= ntpServer %>
<% end -%>
{{< /highlight >}}

This section is actually ruby code that runs a foreach loop on the array "ntpServerList" and adds a line for each ntp server contained in that array. 

* Run the tree command and you should see the following file/directory structure:

{{< highlight bash >}}
[root@vagrant vagrant-vms]# tree
.
├── example_box
│   └── dummy.box
├── manifests
│   └── site.pp
├── modules
│   ├── common
│   │   └── manifests
│   │       └── init.pp
│   └── ntp
│       ├── manifests
│       │   └── init.pp
│       └── templates
│           └── client-ntp.conf.erb
└── Vagrantfile

8 directories, 6 files
[root@vagrant vagrant-vms]# 
{{< /highlight >}}

### 2. Modify the Vagrantfile we created in the last post to include the following:

{{< highlight bash >}}
[root@vagrant vagrant-vms]# cat Vagrantfile
Vagrant.configure("2") do |config|
  config.vm.box = 'dummy'
  config.vm.box_url = './example_box/dummy.box'

  config.vm.provider :vsphere do |vsphere|
    vsphere.host = '192.168.1.195'
    vsphere.name = 'vagrant-test'
    vsphere.clone_from_vm = true
    vsphere.template_name = 'vagrant-centos-6.5'
    vsphere.user = 'root@localos'
    vsphere.password = 'S0meR@nd0mP@ssw0rd'
    vsphere.insecure = true
    vsphere.data_store_name = 'vsanDatastore'
    vsphere.linked_clone = true
    vsphere.customization_spec_name = 'vagrant-centos'
  end
  config.vm.provision "puppet" do |puppet|
    puppet.manifests_path = "manifests"
    puppet.manifest_file = "site.pp"
    puppet.module_path = "modules"
  end
end
[root@vagrant vagrant-vms]# 
{{< /highlight >}}

The following new lines have been added:

* "puppet.manifests_path" specifies the sub-directory, from the directory containing the Vagrantfile, that contains the puppet manifests.
* "puppet.manifest_file" specifies the puppet manifest that will be initially run.
* "puppet.module_path" specifies the sub-directory, from the directory containing the Vagrantfile, that contains the puppet modules.

### 3. Test the configuration changes we made by running a "vagrant up":

{{< highlight bash >}}
[root@vagrant vagrant-vms]# vagrant up --provider=vsphere
Bringing machine 'default' up with 'vsphere' provider...
==> default: Calling vSphere CloneVM with the following settings:
==> default:  -- Source VM: vagrant-centos-6.5
==> default:  -- Name: vagrant-test
==> default: Waiting for SSH to become available...
==> default: New virtual machine successfully cloned and started
==> default: Rsyncing folder: /root/vagrant-vms/ => /vagrant
==> default: Rsyncing folder: /root/vagrant-vms/manifests/ => /tmp/vagrant-puppet-1/manifests
==> default: Rsyncing folder: /root/vagrant-vms/modules/ => /tmp/vagrant-puppet-1/modules-0
==> default: Running provisioner: puppet...
Running Puppet with site.pp...
notice: /File[/etc/ntp.conf]/content: content changed '{md5}d7e1e16f9c0cd6382f6b68b486163db1' to '{md5}f7a83a4ca84e1ba2bba0166c5620e9e7'
notice: /Stage[main]/Ntp/Service[ntpd]: Triggered 'refresh' from 1 events
notice: Finished catalog run in 0.72 seconds
[root@vagrant vagrant-vms]# 
{{< /highlight >}}

You might notice a few new lines since the "vagrant up" we ran in the last post. 

* The manifests & modules folders we specified in Vagrantfile have been copied over using rsync. 
* The Puppet agent is running a "puppet apply" using the site.pp file we specified. 
* /etc/ntp.conf was modified by Puppet
* The NTP service was notified that it needed to refresh it's configuration since /etc/ntp.conf had been modified.

### 4. Connect to the vagrant vm to validate that /etc/ntp.conf has been modified to include the three NTP servers we specified:

{{< highlight bash >}}
[root@vagrant vagrant-vms]# vagrant ssh
[vagrant@vagrant-test ~]$ cat /etc/ntp.conf
# This file is being maintained by Puppet.
# DO NOT EDIT

# Permit time synchronization with our time source, but do not
# permit the source to query or modify the service on this system.
restrict default kod nomodify notrap nopeer noquery
restrict -6 default kod nomodify notrap nopeer noquery

# Permit all access over the loopback interface.  This could
# be tightened as well, but to do so would effect some of
# the administrative functions.
restrict 127.0.0.1
restrict -6 ::1

server 0.pool.ntp.org iburst
server 1.pool.ntp.org iburst
server 2.pool.ntp.org iburst

# Drift file.  Put this in a directory which the daemon can write to.
# No symbolic links allowed, either, since the daemon updates the file
# by creating a temporary in the same directory and then rename()'ing
# it to the file.
driftfile /var/lib/ntp/drift
[vagrant@vagrant-test ~]$ exit
logout
Connection to 192.168.1.123 closed.
[root@vagrant vagrant-vms]# 
{{< /highlight >}}


### 5. Modify the site.pp file to add a node definition for our vagrant vm:

{{< highlight bash >}}
[root@vagrant vagrant-vms]# cat site.pp
node default {
  include common
    class {'ntp':}
}
node 'vagrant-test.mylab.net' {
  include common
    class {'ntp':
      ntpServerList => ['3.pool.ntp.org','4.pool.ntp.org','5.pool.ntp.org']
    }
}
{{< /highlight >}}

The new "node 'vagrant-test.mylab.net'" definition will only be run by hosts with a matching hostname. Any host that doesn't have a matching host definition will continue to run the "node default" section. Node definitions allow you to change variables or even modules that will be run by specific hosts and provide flexibility in the changes made to a given host. Our new node section will configure "vagrant-test.mylab.net" to use a different set of ntp servers than any other host that runs this Puppet manifest  

### 6. Run 'vagrant provision' to apply the updated Puppet manifest to our Vagrant vm:

{{< highlight bash >}}
[root@vagrant vagrant-vms]# vagrant provision
==> default: Rsyncing folder: /root/vagrant-vms/ => /vagrant
==> default: Rsyncing folder: /root/vagrant-vms/manifests/ => /tmp/vagrant-puppet-1/manifests
==> default: Rsyncing folder: /root/vagrant-vms/modules/ => /tmp/vagrant-puppet-1/modules-0
==> default: Running provisioner: puppet...
Running Puppet with site.pp...
notice: /File[/etc/ntp.conf]/content: content changed '{md5}f7a83a4ca84e1ba2bba0166c5620e9e7' to '{md5}414e811fdbfa3f8cfa38e1e4e6d0586f'
notice: /Stage[main]/Ntp/Service[ntpd]: Triggered 'refresh' from 1 events
notice: Finished catalog run in 0.34 seconds
[root@vagrant vagrant-vms]#
{{< /highlight >}}

While we could have destroyed and recreated our Vagrant vm to test the change to the Puppet manifest, 'vagrant provision' quickly resyncs the folders defined in our Vagrantfile and re-runs our Puppet manifest.

### 7. Connect to the vagrant vm to validate that /etc/ntp.conf has been modified to include the three NTP servers we defined fof this specific node:
{{< highlight bash >}}
root@vagrant vagrant-vms]# vagrant ssh
Last login: Tue May  6 15:09:50 2014 from 192.168.1.40
[vagrant@vagrant-test ~]$ cat /etc/ntp.conf
# This file is being maintained by Puppet.
# DO NOT EDIT

# Permit time synchronization with our time source, but do not
# permit the source to query or modify the service on this system.
restrict default kod nomodify notrap nopeer noquery
restrict -6 default kod nomodify notrap nopeer noquery

# Permit all access over the loopback interface.  This could
# be tightened as well, but to do so would effect some of
# the administrative functions.
restrict 127.0.0.1
restrict -6 ::1

server 3.pool.ntp.org iburst
server 4.pool.ntp.org iburst
server 5.pool.ntp.org iburst

# Drift file.  Put this in a directory which the daemon can write to.
# No symbolic links allowed, either, since the daemon updates the file
# by creating a temporary in the same directory and then rename()'ing
# it to the file.
driftfile /var/lib/ntp/drift
[vagrant@vagrant-test ~]$ exit
logout
Connection to 192.168.1.123 closed.
[root@vagrant vagrant-vms]#
{{< /highlight >}}

### 8. Shutdown the vagrant machine:

{{< highlight bash >}}
[root@vagrant vagrant-vms]# vagrant destroy
==> default: Calling vSphere PowerOff
==> default: Calling vShpere Destroy
[root@vagrant vagrant-vms]#
{{< /highlight >}}

#### Hopefully you found this post helpful in demonstrating how to use Puppet with Vagrant, as well as the benefits it provides for testing Puppet manfests and modules

---

#### Please provide any feedback or suggestions to my twitter account located on the about page.


