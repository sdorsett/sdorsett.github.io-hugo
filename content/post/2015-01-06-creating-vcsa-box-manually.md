+++
title = "Creating a vCSA 5.5 .box template on CentOS 6.5 for vagrant deployment"
description = ""
tags = [
    "vagrant",
    "vsphere",
]
date = "2015-01-06"
categories = [
    "vagrant",
    "vsphere",
]
highlight = "true"
+++

In the last few blogs post we have created ESXi .box templates, but in order to create a complete virtual lab using vagrant we will also need a vCenter Server Appliance virtual machine. The vCSA comes as a .ova template, so we will need to convert it to a vagrant-vmware-ovf .box template before we can use it with vagrant.

In this post we will need several packages installed, that we have covered in the last few posts:

* vagrant - I used vagrant 1.7.1 for these steps
* [gosddc/vagrant-vcenter](https://github.com/gosddc/vagrant-vcenter).
   This repo contains a vagrant plugin (provider) that will deploy a vagrant-vmware-ovf generated .box template to a vcenter instance. This provider allows vagrant to seamlessly deploy to vCenter.
* ovftool - I have ovftool 3.5.1 installed on my CentOS vm.

I will be using the same CentOS virtual machine we used [in the last post](https://sdorsett.github.io/2015/01/04/installing-vagrant-on-centos/) since it already has all the packages we will be needing.

### 1. Download the vCenter Server Appliance .iso from the [vmware.com download page](https://my.vmware.com/web/vmware/details?downloadGroup=VC55U2&productId=353). 
I will be using "VMware-vCenter-Server-Appliance-5.5.0.20000-2063318_OVF10.ova", which is the vCSA 5.5 U2 .ova. 

### 2. Use ovftool to upload the vCSA .ova to vCenter:

{{< highlight bash >}}
[root@vagrant ~]# ovftool --disableVerification  --noSSLVerify --datastore=esx01-local-sata --name=vcsa-5.5.0.20000-2063318 --network=vlan2 ~/VMware-vCenter-Server-Appliance-5.5.0.20000-2063318_OVF10.ova vi://root@192.168.1.201
Opening OVA source: VMware-vCenter-Server-Appliance-5.5.0.20000-2063318_OVF10.ova
The manifest validates
Enter login information for target vi://192.168.1.201/
Username: root
Password: **********
Opening VI target: vi://root@192.168.1.201:443/
Deploying to VI: vi://root@192.168.1.201:443/
Transfer Completed
Completed successfully
[root@vagrant ~]#
{{< /highlight >}}

You will need to modify the following parameters in the ovftool command:

* --datastore - this is the datastore that ovftool will deploy the virtual machine to
* --name - this is the name the deployed virtual machine will be given
* --network - this is the portgroup the deployed virtual machine will be connected to
* vi://root://[esxi\_host\_ip\_address] - update this part of the command with the IP address of one of your ESXi servers

### 3. Modify the vCSA virtual machine for vagrant deployment.
 
* Power on the vcsa-5.5.0.20000-2063318 virtual machine in the VI or vSphere web client.

* Once the vCSA virtual machine is up and running, SSH to the IP address that is acquired by DHCP.

* Create the ~/.ssh directory

{{< highlight bash >}}
localhost:~ # mkdir ~/.ssh
{{< /highlight >}}

* Add the vagrant public ssh key to ~/.ssh/authorized_keys:

{{< highlight bash >}}
localhost:~ # cd .ssh
localhost:~/.ssh # echo 'ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEA6NF8iallvQVp22WDkTkyrtvp9eWW6A8YVr+kz4TjGYe7gHzIw+niNltGEFHzD8+v1I2YJ6oXevct1YeS0o9HZyN1Q9qgCgzUFtdOKLv6IedplqoPkcmF0aYet2PkEDo3MlTBckFXPITAMzF8dJSIFo9D8HfdOV0IAdx4O7PtixWKn5y2hMNG0zQPyUecp4pzC6kivAIhyfHilFR61RGL+GPXQ2MWZWFYbAGjyiYJnAmCP3NOTd0jMZEnDkbUvxhMmBYSdETk1rRgm+R4LOzFUGaHqHDLKLX+FIPKcF96hrucXzcWyLbIbEgE98OHlnVYCzRdK8jlqm8tehUc9c9WhQ== vagrant insecure public key' > ~/.ssh/authorized_keys
{{< /highlight >}}

* Modify the permissions on ~/.ssh & ~/.ssh/authorized_keys since openssh is particular about file permissions:

{{< highlight bash >}}
localhost:~/.ssh # chmod 700 ~/.ssh
localhost:~/.ssh # chmod 600 ~/.ssh/authorized_keys
{{< /highlight >}}

* Modify the following lines in /etc/ssh/sshd_config:

Uncomment and update the following lines:
{{< highlight bash >}}
PubkeyAuthentication yes                  # line 47
AuthorizedKeysFile ~/.ssh/authorized_keys # line 48
{{< /highlight >}}
Comment out the following line:
{{< highlight bash >}}
#MaxSessions 1                            # line 44
{{< /highlight >}}

* Power down the vcsa-5.5.0.20000-2063318 virtual machine in the VI or vSphere web client.

### 4. Use ovftool to export the vcsa-5.5.0.20000-2063318 virtual machine.

{{< highlight bash >}}
[root@vagrant ~]# ovftool vi://root@192.168.1.201/vcsa-5.5.0.20000-2063318 ./
Enter login information for source vi://192.168.1.201/
Username: root
Password: **********
Opening VI source: vi://root@192.168.1.201:443/vcsa-5.5.0.20000-2063318
Opening OVF target: ./
Writing OVF package: ./vcsa-5.5.0.20000-2063318/vcsa-5.5.0.20000-2063318.ovf
Transfer Completed
Completed successfully
{{< /highlight >}}

"vi://root@192.168.1.201" will need to be updated to use the IP address of the ESXi host that has your vcsa-5.5.0.20000-2063318 virtual machine.

### 5. add the additional vagrant-vmware-ovf files and tar up the vcsa-5.5.0.20000-2063318 .ovf as a .box file:

{{< highlight bash >}}
[root@vagrant ~]# cd vcsa-5.5.0.20000-2063318/
[root@vagrant vcsa-5.5.0.20000-2063318]# echo '{"provider":"vmware_ovf"}' >> metadata.json
[root@vagrant vcsa-5.5.0.20000-2063318]# touch Vagrantfile
[root@vagrant vcsa-5.5.0.20000-2063318]# tar cvzf vcsa-5.5.0.20000-2063318-vmware_ovf-1.0.box ./*
./metadata.json
./Vagrantfile
./vcsa-5.5.0.20000-2063318-disk1.vmdk
./vcsa-5.5.0.20000-2063318-disk2.vmdk
./vcsa-5.5.0.20000-2063318.mf
./vcsa-5.5.0.20000-2063318.ovf
[root@vagrant vcsa-5.5.0.20000-2063318]#
{{< /highlight >}}


### 6. Create a directory to keep our vagrant-vmware-ovf .box template in and move the vCSA template to it:

{{< highlight bash >}}
[root@vagrant vcsa-5.5.0.20000-2063318]# mkdir ~/box-files/
[root@vagrant vcsa-5.5.0.20000-2063318]# mv vcsa-5.5.0.20000-2063318-vmware_ovf-1.0.box ~/box-files/
{{< /highlight >}}

* Create directory and Vagrantfile for the vCSA virtual machines we will deploy with vagrant:
 
{{< highlight bash >}}
[root@vagrant vcsa-5.5.0.20000-2063318]# mkdir -p ~/vagrant-vms/vcsa-test/
[root@vagrant vcsa-5.5.0.20000-2063318]# cd ~/vagrant-vms/vcsa-test/
[root@vagrant vcsa-test]# vi Vagrantfile
[root@vagrant vcsa-test]# cat Vagrantfile
vcsa_box_url = '/root/box-files/vcsa-5.5.0.20000-2063318-vmware_ovf-1.0.box'

$script = <<SCRIPT
#!/bin/bash
# Commands to configure all the necessary services to start the vCenter Service, thanks @lamw ! see:
# http://www.virtuallyghetto.com/2012/02/automating-vcenter-server-appliance.html
 
echo "Accepting EULA ..."
/usr/sbin/vpxd_servicecfg eula accept
 
echo "Configuring Embedded DB ..."
/usr/sbin/vpxd_servicecfg db write embedded
 
echo "Configuring SSO..."
/usr/sbin/vpxd_servicecfg sso write embedded
 
echo "Starting VCSA ..."
/usr/sbin/vpxd_servicecfg service start

# http://www.virtuallyghetto.com/2014/02/how-to-automate-ntp-configurations-on.html
/usr/sbin/vpxd_servicecfg timesync write ntp 'pool.ntp.org' ''

# Command to set IP address to 192.168.1.81:
# http://www.virtuallyghetto.com/2013/02/automating-vcsa-network-configurations.html
echo "Setting static IP address to 192.168.1.81"
/opt/vmware/share/vami/vami_set_network eth0 STATICV4 192.168.1.81 255.255.255.0 192.168.1.1 >&- 2>&- <&- &
SCRIPT

nodes = [
  { :hostname => 'vcsa-01a', :box => 'vcsa-5.5.0.20000-2063318', :box_url => vcsa_box_url}
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
 
      if node[:hostname].include? 'vcsa-'
        node_config.ssh.username = 'root'
        node_config.ssh.insert_key = false
        node_config.vm.synced_folder '.', '/vagrant', disabled: true
        node_config.vm.provision "shell", inline: $script
      end 
      node_config.vm.box = node[:box]
      node_config.vm.hostname = node[:hostname]
      node_config.vm.box_url = node[:box_url]
    end
  end
end
{{< /highlight >}}

This Vagrantfile is specifying vagrant to use the vagrant-vcenter profiler to create one virtual machine (vcsa-01a) from the vcsa-5.5.0.20000-2063318-vmware_ovf-1.0.box file. You will need update the following properties to reflect your own vCenter configuration:

* vcenter.hostname = the IP address of your vCenter server
* vcenter.username = the username used to connect to your vCenter server
* vcenter.password = the password used to connect to your vCenter server
* vcenter.datacenter_name = the vCenter virtual datacenter to use for virtual machine deployment
* vcenter.computer_name = the vCenter host or cluster to use for virtual machine deployment. I've used the cluster in my example.
* vcenter.datastore_name = the vCenter datastore to use for virtual machine deployment
* vcenter.folder_name = the vm folder that the virtual machines will be deployed to
* vcenter.template\_folder\_name = the vm folder that the vCSA template will be created in
* vcenter.network_name = the vCenter portgroup to connect the vCSA template to. There should be a DHCP server on this portgroup to provide IP addresses to the deployed vCSA template.

There is also a script section at the top of the Vagrantfile I would like to point out. William Lam has posted several posts documenting how to automate vCSA configuration, and several of his suggestions have been included in this section. Namely those are:

* Accepting the EULA
* Configuring the vCSA to use the internal database
* Configuring internal SSO
* Starting the vCenter services
* Setting NTP
* Assigning a static IP address using vami\_set\_network command

You might want to adjust the last part of the script, since it will set vCSA to use 192.168.1.81 as it's static IP address. 

{{< highlight bash >}}
echo "Setting static IP address to 192.168.1.81"
/opt/vmware/share/vami/vami_set_network eth0 STATICV4 192.168.1.81 255.255.255.0 192.168.1.1 >&- 2>&- <&- &
{{< /highlight >}}

I prefer to set a static address for the vCSA, so that any scripts that follow the vCSA deployment will know exactly what IP address to use to connect to it. Also you might be wondering about extra characters following the VAMI command. Since this command is modifying the IP address of the virtual machine, we're need to run this command in the background without expecting a response. Here the breakdown of what is being done:

* \>\&- means close stdout.
* 2\>\&- means close stderr.
* \<\&- means close stdin.
* \& means run in the background

If we didn't pipe the vami\_set\_network command through these commands, vagrant would keep waiting on output in the SSH session that the command was successfully run, but the IP address had been changed so it would never receive the expected response. 

### 4. "vagrant up"

* Now that we have created a Vagrantfile config file and updated it with our vCenter information we can bring up our virtual machine by running "vagrant up" in the directory with the Vagrantfile.

{{< highlight bash >}}
[root@vagrant vcsa-test]# vagrant up
Bringing machine 'vcsa-01a' up with 'vcenter' provider...
==> vcsa-01a: Creating VM...
==> vcsa-01a: Powering on VM...
==> vcsa-01a: Running provisioner: shell...
    vcsa-01a: Running: inline script
==> vcsa-01a: Accepting EULA ...
==> vcsa-01a: VC_CFG_RESULT=0
==> vcsa-01a: Configuring Embedded DB ...
==> vcsa-01a: insserv: Service network is missed in the runlevels 4 to use service postgresql
==> vcsa-01a: insserv: Service syslog is missed in the runlevels 4 to use service postgresql
==> vcsa-01a: VC_DB_SCHEMA_VERSION=VirtualCenter Database 5.5
==> vcsa-01a: VC_DB_SCHEMA_INITIALIZED=1
==> vcsa-01a: VC_CFG_RESULT=0
==> vcsa-01a: Configuring SSO...
==> vcsa-01a: VC_CFG_RESULT=0
==> vcsa-01a: Starting VCSA ...
==> vcsa-01a: VC_CFG_RESULT=0
==> vcsa-01a: insserv: Service network is missed in the runlevels 4 to use service postgresql
==> vcsa-01a: insserv: Service syslog is missed in the runlevels 4 to use service postgresql
==> vcsa-01a: VC_CFG_RESULT=0
==> vcsa-01a: Setting static IP address to 192.168.1.81
[root@vagrant vcsa-test]# 
{{< /highlight >}} 

* Once the "vagrant up" command completes, you can check the status of the vCSA vm using "vagrant status":

{{< highlight bash >}}
[root@vagrant vcsa-test]# vagrant status
Current machine states:

vcsa-01a                  running (vcenter)

The VM is running. To stop this VM, you can run `vagrant halt` to
shut it down forcefully, or you can run `vagrant suspend` to simply
suspend the virtual machine. In either case, to restart it again,
simply run `vagrant up`.
[root@vagrant vcsa-test]#
{{< /highlight >}}

* You can now ssh into the vCSA virtual machine using "vagrant ssh":

{{< highlight bash >}}
[root@vagrant vcsa-test]# vagrant ssh
==> vcsa-01a: External IP for vcsa-01a: 192.168.1.81
Last login: Sat Jan 10 20:08:07 UTC 2015 from 192.168.1.24 on pts/0
Last login: Sat Jan 10 20:09:36 2015 from 192.168.1.24
localhost:~ # service vmware-vpxd status
vmware-vpxd is running
tomcat is running
localhost:~ # exit
logout
Connection to 192.168.1.81 closed.
[root@vagrant vcsa-test]# 
{{< /highlight >}}

I wanted to point out that vagrant automatically detected up the IP address change we made through vmtools, so we didn't have to perform any additional steps for "vagrant ssh" to work.

* Once we are done using our virtual machine, it can be destroyed with a "vagrant destroy" command:

{{< highlight bash >}}
[root@vagrant vcsa-test]# vagrant destroy
    vcsa-01a: Are you sure you want to destroy the 'vcsa-01a' VM? [y/N] y
==> vcsa-01a: Powering off VM...
==> vcsa-01a: Destroying VM...
[root@vagrant vcsa-test]#
{{< /highlight >}}

* If you are ever unsure of the state of the vagrant virtual machine you can run "vagrant status" to check:

{{< highlight bash >}}
[root@vagrant vcsa-test]# vagrant status
Current machine states:

vcsa-01a                  not created (vcenter)

The environment has not yet been created. Run `vagrant up` to
create the environment. If a machine is not created, only the
default provider will be shown. So if a provider is not listed,
then the machine is not created for that environment.
[root@vagrant vcsa-test]# 
{{< /highlight >}}

#### That all for this post covering how to create a vCSA .box template and deploy it using the vagrant-vcenter provider. 

---

#### Please provide any feedback or suggestions to my twitter account located on the about page.

