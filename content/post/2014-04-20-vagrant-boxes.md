+++
title = "Creating a CentOS 6.x template that is customized for Vagrant"
description = ""
tags = [
    "vagrant",
]
date = "2014-04-20"
categories = [
    "vagrant",
]
topics = [
    "using-vagrant-with-vsphere"
]
highlight = "true"
+++

This post will continue our examination of using the vagrant-vsphere plugin, by customizing a CentOS template for integration with Vagrant. Having a Vagrant customize template will allow us to deploy this template and SSH to the cloned vm using the "vagrant ssh" command, as well automatically run puppet manifests when we deploy a vm using Vagrant. Let's get started:

### 1. Ensure you have a DHCP server on the network you will be connecting the Vagrant deployed templates to. While you can utilized guest customization to assign a static IP addresses to a Vagrant deployed vm, using DHCP will make this process much easier.

### 2. Create a CentOS 6.x minimal virtual machine for configuring as our Vagrant CentOS template. 
Create or clone a fresh CentOS 6.x minimal virtual machine. I already have an existing Centos 6.5 minimal template that I simply cloned for this purpose.

### 3. Power on the new cloned vm and connect using SSH to the DHCP assigned IP address for this vm.

### 4. Ensure you have perl, rsync, ruby & puppet agent installed:

* Import the EPEL 6 key and RPM:

{{< highlight bash >}}
[root@centos-6-5 ~]# rpm --import http://dl.fedoraproject.org/pub/epel/RPM-GPG-KEY-EPEL-6
[root@centos-6-5 ~]# rpm -Kih http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
########################################### [100%]
########################################### [100%]
{{< /highlight >}}

* Use yum to install ruby, wget, puppet, facter & rsync:

{{< highlight bash >}}
[root@centos-6-5 ~]# yum install ruby ruby-libs ruby-shad wget yum-priorities puppet facter rsync -y
{{< /highlight >}}


### 5. Ensure you have vmtools installed on the new CentOS template.

* Start the install process by clicking "VM | Guest | Install/Upgrade VMware Tools" in the vShpere client. When prompted select "Automatice Tools Upgrade" and click "OK"

* Create a directory to mount the virtual CDROM to with the following command:

{{< highlight bash >}}
[root@centos-6-5 ~]# mkdir /mnt/cdrom
{{< /highlight >}}

* Mount the virtual CDROM to /mnt/cdrom with the following command:

{{< highlight bash >}}
[root@centos-6-5 ~]# mount /dev/cdrom /mnt/cdrom
mount: block device /dev/sr0 is write-protected, mounting read-only
{{< /highlight >}}

* Copy the VMware Tools installer over to the /root directory and unzip it:

{{< highlight bash >}}
[root@centos-6-5 ~]# cp /mnt/cdrom/VMwareTools-* ~/
[root@centos-6-5 ~]# tar -zxf ~/VMwareTools*
{{< /highlight >}}

* Change directory into the directory we just untared and run the "vmware-install.pl" to start the installer:

{{< highlight bash >}}
[root@centos-6-5 ~]# cd vmware-tools-distrib/
[root@centos-6-5 vmware-tools-distrib]# ./vmware-install.pl
{{< /highlight >}}

You should be able to accept the defaults for all prompts and complete the VMware tools installer. Once the installer has completed check the vm summary of this template vm in the vSphere client. The VMware Tools should show as "Running" and "Current."

* Unmount the virtual CDROM using the following command:

{{< highlight bash >}}
[root@centos-6-5 ~]# umount /dev/cdrom
{{< /highlight >}}

### 6. We will next follow the steps for [creating a base box](https://docs.vagrantup.com/v2/boxes/base.html) from the Vagrant website. 

One thing to point out is we will be configuring the vagrant template for password less SSH using a vagrant user with the password of "vagrant." The Vagrant website mentions 'by default, Vagrant expects a "vagrant" user to SSH into the machine as. This user should be setup with the insecure keypair that Vagrant uses as a default to attempt to SSH. Also, even though Vagrant uses key-based authentication by default, it is a general convention to set the password for the "vagrant" user to "vagrant."'

This vagrant template should only be used for lab or development work and never used for production or publicly facing virtual machines due to the security risks of having known users, passwords & private SSH keys configured by default. 

* Add a new user named "vagrant" to the template. 

{{< highlight bash >}}
[root@centos-6-5 ~]# useradd vagrant
{{< /highlight >}}

* Change the password of the user "vagrant" to "vagrant"

{{< highlight bash >}}
[root@centos-6-5 ~]# [root@centos-6-5 ~]# passwd vagrant
Changing password for user vagrant.
New password: <type the word "vagrant">
BAD PASSWORD: it is based on a dictionary word
BAD PASSWORD: is too simple
Retype new password: <type the word "vagrant" again>
passwd: all authentication tokens updated successfully.
{{< /highlight >}}

* We will next need to copy the contents of the vagrant.pub file located at the [vagrant github site](https://github.com/mitchellh/vagrant/tree/master/keys) into the /home/vagrant/.ssh/authorized_keys file.

First we will su (switch user) to the vagrant user by running the following command:

{{< highlight bash >}}
[root@centos-6-5 ~]# su vagrant
[vagrant@centos-6-5 root]$ 
{{< /highlight >}}

Next we will create the .ssh directory since is does not exist:

{{< highlight bash >}}
[vagrant@centos-6-5 root]$ mkdir ~/.ssh
[vagrant@centos-6-5 root]$ cd ~/.ssh
[vagrant@centos-6-5 .ssh]$ pwd
/home/vagrant/.ssh
{{< /highlight >}}

We will now create /home/vagrant/.ssh/authorized_keys with the contents of [this file](https://raw.githubusercontent.com/mitchellh/vagrant/master/keys/vagrant.pub) from the vagrant github page:

{{< highlight bash >}}
[vagrant@centos-6-5 .ssh]$ echo 'ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEA6NF8iallvQVp22WDkTkyrtvp9eWW6A8YVr+kz4TjGYe7gHzIw+niNltGEFHzD8+v1I2YJ6oXevct1YeS0o9HZyN1Q9qgCgzUFtdOKLv6IedplqoPkcmF0aYet2PkEDo3MlTBckFXPITAMzF8dJSIFo9D8HfdOV0IAdx4O7PtixWKn5y2hMNG0zQPyUecp4pzC6kivAIhyfHilFR61RGL+GPXQ2MWZWFYbAGjyiYJnAmCP3NOTd0jMZEnDkbUvxhMmBYSdETk1rRgm+R4LOzFUGaHqHDLKLX+FIPKcF96hrucXzcWyLbIbEgE98OHlnVYCzRdK8jlqm8tehUc9c9WhQ== vagrant insecure public key' > ~/.ssh/authorized_keys
[vagrant@centos-6-5 .ssh]$ cat ~/.ssh/authorized_keys
ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEA6NF8iallvQVp22WDkTkyrtvp9eWW6A8YVr+kz4TjGYe7gHzIw+niNltGEFHzD8+v1I2YJ6oXevct1YeS0o9HZyN1Q9qgCgzUFtdOKLv6IedplqoPkcmF0aYet2PkEDo3MlTBckFXPITAMzF8dJSIFo9D8HfdOV0IAdx4O7PtixWKn5y2hMNG0zQPyUecp4pzC6kivAIhyfHilFR61RGL+GPXQ2MWZWFYbAGjyiYJnAmCP3NOTd0jMZEnDkbUvxhMmBYSdETk1rRgm+R4LOzFUGaHqHDLKLX+FIPKcF96hrucXzcWyLbIbEgE98OHlnVYCzRdK8jlqm8tehUc9c9WhQ== vagrant insecure public key
{{< /highlight >}}

The vagrant website also mentions 'that OpenSSH is very picky about file permissions. Therefore, make sure that ~/.ssh has 0700 permissions and the authorized keys file has 0600 permissions.' We will correct those permissions now:

{{< highlight bash >}}
[vagrant@centos-6-5 .ssh]$ cd ~/
[vagrant@centos-6-5 ~]$ chmod 700 ~/.ssh
[vagrant@centos-6-5 ~]$ chmod 600 ~/.ssh/authorized_keys
[vagrant@centos-6-5 ~]$ ls -la
total 24
drwx------. 3 vagrant vagrant 4096 Apr 20 14:22 .
drwxr-xr-x. 3 root    root    4096 Apr 20 14:15 ..
-rw-r--r--. 1 vagrant vagrant   18 Jul 18  2013 .bash_logout
-rw-r--r--. 1 vagrant vagrant  176 Jul 18  2013 .bash_profile
-rw-r--r--. 1 vagrant vagrant  124 Jul 18  2013 .bashrc
drwx------. 2 vagrant vagrant 4096 Apr 20 14:26 .ssh
[vagrant@centos-6-5 ~]$ ls -la .ssh
total 12
drwx------. 2 vagrant vagrant 4096 Apr 20 14:26 .
drwx------. 3 vagrant vagrant 4096 Apr 20 14:22 ..
-rw-------. 1 vagrant vagrant  409 Apr 20 14:26 authorized_keys
[vagrant@centos-6-5 ~]$ exit
exit
[root@centos-6-5 ~]#
{{< /highlight >}}

* We will to edit the sudo file. Type the following command to open the sudo file in vi:

{{< highlight bash >}}
root@centos-6-5 ~]# visudo
{{< /highlight >}}

Locate the line containing:

{{< highlight bash >}}
Defaults    requiretty
{{< /highlight >}}

...and change it to read:

{{< highlight bash >}}
# Defaults    requiretty'
{{< /highlight >}}

The # sign at the beginning of the line will cause this line to be ignored as a comment.
We also need to add the following line at the bottom of the file:

{{< highlight bash >}}
vagrant ALL=(ALL) NOPASSWD: ALL
{{< /highlight >}}

* One more recommendation from the vagrant website is to edit /etc/ssh/sshd_config and change any line matching:

{{< highlight bash >}}
UseDNS yes
{{< /highlight >}}

to 

{{< highlight bash >}}
UseDNS no
{{< /highlight >}}

This avoids a reverse DNS lookup on the connecting SSH client which can take many seconds.

### 7. Lastly this is a template, so we will need to remove /etc/udev/rules.d/70-persistent-net.rules file because it contains the MAC address of the current virtual NIC. It will be recreated on boot up or when we deploy other virtual machines from this template.

### 8. We will now shutdown the template by running halt:

{{< highlight bash >}}
[root@centos-6-5 ~]# halt
Broadcast message from root@centos-6-5
	(/dev/pts/1) at 14:51 ...
The system is going down for halt NOW!
{{< /highlight >}}

### 9. We now need to SSH to the vagrant vm that has the vagrant-vsphere plugin installed. This is the vm we created if you followed the <a href="../2014-04-19-vagrant-install">last blog post</a>.

* Edit the Vagrantfile we created to modify the following line to reflect the name you gave the new vagrant template we cloned or created in the steps above:

{{< highlight bash >}}
vsphere.template_name = 'vagrant-centos-6.5'
{{< /highlight >}}

* We can test this new template by issuing the following command:

{{< highlight bash >}}
[root@vagrant vagrant-vms]# vagrant up --provider=vsphere
Bringing machine 'default' up with 'vsphere' provider...
==> default: Calling vSphere CloneVM with the following settings:
==> default:  -- Source VM: vagrant-centos-6.5
==> default:  -- Name: vagrant-test
==> default: Waiting for SSH to become available...
==> default: New virtual machine successfully cloned and started
==> default: Rsyncing folder: /root/vagrant-vms/ => /vagrant
{{< /highlight >}}

Unlike our 'vagrant up" command in the last post we have established a SSH session with the cloned template using the vagrant user we created and the public key in that users authorized_keys file.

You also might notice that the entire directory we launched the 'vagrant up" command from has been copied over to /vagrant on the cloned vm using rsync. We will go into leveraging this feature with puppet in our next blog post.

* Using SSH to connect to the vagrant vm is as easy as 'vagrant ssh'

{{< highlight bash >}}
[root@vagrant vagrant-vms]# hostname
vagrant.mylab.net
[root@vagrant vagrant-vms]# vagrant ssh
[vagrant@vagrant-centos-6-5 ~]$ hostname
vagrant-centos-6-5
[vagrant@vagrant-centos-6-5 ~]$ exit
logout
Connection to 192.168.1.133 closed.
[root@vagrant vagrant-vms]# 
{{< /highlight >}}

* Finally we will power down and delete the cloned template by running the following command:

{{< highlight bash >}}
[root@vagrant vagrant-vms]# vagrant destroy
==> default: Calling vSphere PowerOff
==> default: Calling vShpere Destroy
{{< /highlight >}}

#### Hopefully you found this post helpful in demonstrating how to create a vagrant specific template to use with the vagrant-vsphere plugin. The next blog post will be covering how to take advantage of customization specifications and advanced Vagrantfile configurations on a vm deployed using vagrant.

---

#### Please provide any feedback or suggestions to my twitter account located on the about page.

