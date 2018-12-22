+++
title = "Using packer on OS X to create an ESXi .box template for vagrant deployment"
description = ""
tags = [
    "packer",
    "vsphere",
]
date = "2014-12-28"
categories = [
    "vagrant",
    "vsphere",
]
highlight = "true"
+++


I've been recently working on using packer to create vagrant .box files rather than manually creating them as I documented in a previous post. For this post I will be using fusion and the vagrant vmware provider, each of which have an associated cost, but I will cover a free alternative using packer and CentOS in a future post.

Several github projects by gosddc have helped me in getting packer up and running on my Macbook:

* [gosddc/packer-post-processor-vagrant-vmware-ovf](https://github.com/gosddc/packer-post-processor-vagrant-vmware-ovf).
   This repo contains a packer post processor that leverages VMware OVF Tool to create a vmware_ovf Vagrant box that is compatible with vagrant-vcloud, vagrant-vcenter and vagrant-vcloudair vagrant providers. It is only compatible with the packer VMware builder.

* [gosddc/packer-templates](https://github.com/gosddc/packer-templates).
   This repo contains Packer templates for boxes available at https://vagrantcloud.com/gosddc, they only work with VMware and require the packer-post-processor-vagrant-vmware-ovf post-processor to work. These templates are a good starting point for generating pakcer templates on VMware products.


### 1. Let's get started by installing packer.
* Download packer from the following link:

   [https://www.packer.io/downloads.html](https://www.packer.io/downloads.html)

* Unzip the downloaded files to /usr/local/packer_7.5:

* Add /usr/local/packer_7.5 to your path by running the following command.

{{< highlight bash >}}
sdorsett-mbp:~ sdorsett$ export PATH="/usr/local/packer_7.5:$PATH"
{{< /highlight >}}

* Add the export command we just ran into ~/.bash_profile to ensure this path change persists after reboots.

{{< highlight bash >}}
sdorsett-mbp:~ sdorsett$ cat ~/.bash_profile
export PATH="/usr/local/packer_7.5:$PATH"
sdorsett-mbp:~ sdorsett$
{{< /highlight >}}

### 2. Next we need to download and install VMware ovftool, since the vagrant-vmware-ovf requires it.

* Download and install the latest version of the VMware OVF tool. VMware-ovftool-3.5.0-1274719-mac.x64.dmg is what I used.

* Verify ovftool is succesfully added to your path by running "ovftool -v". This command should output the version of ovftool we installed.

{{< highlight bash >}}
sdorsett-mbp:~ sdorsett$ ovftool -v
VMware ovftool 3.5.0 (build-1274719)
{{< /highlight >}}

### 3. Now we can download and install the gosddc packer components we will need.

* Download the most recent version of the compiled packer-processor-vagrant-vmware-ovf binary from the following link:

[https://github.com/gosddc/packer-post-processor-vagrant-vmware-ovf/releases](https://github.com/gosddc/packer-post-processor-vagrant-vmware-ovf/releases)

* Unzip packer-post-processor-vagrant-vmware-ovf and copy it to "usr/local/packer_7.5". Ensure the permissions of this file match the other files in this directory.

* Create a directory to contain the packer templates:

{{< highlight bash >}}
sdorsett-mbp:~ sdorsett$ mkdir ~/Documents/packer
sdorsett-mbp:~ sdorsett$ cd ~/Documents/packer/
{{< /highlight >}}

* Clone the gosddc packer-templates repository:

{{< highlight bash >}}
sdorsett-mbp:packer sdorsett$ git clone https://github.com/gosddc/packer-templates.git
sdorsett-mbp:packer sdorsett$ cd packer-templates
{{< /highlight >}}

### 4. Now we need to download the ESXi 5.5 .iso and copy it into the proper directory location.

* Create an "iso" directory for storing the ESXi iso files:

{{< highlight bash >}}
sdorsett-mbp:packer-templates sdorsett$ mkdir ~/Documents/packer/packer-templates/iso
{{< /highlight >}}

* Download and copy the "VMware-VMvisor-Installer-5.5.0-1331820.x86_64.iso" ESXi 5.5 installer to the iso directory.

### 5. Finally we will need to modify, validate and build the packer esxi.json packer template we will be using.

* Modify ~/Documents/packer/packer-templates/templates/esxi.json to look like the following:

{{< highlight bash >}}
sdorsett-mbp:packer-templates sdorsett$ cat templates/esxi.json
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
      "guest_os_type": "vmkernel5",
      "iso_url": "./iso/VMware-VMvisor-Installer-5.5.0-1331820.x86_64.iso",
      "iso_checksum": "ef599dc7e647177027684c0eee346ccdbc8704f2",
      "iso_checksum_type": "sha1",
      "ssh_username": "root",
      "ssh_password": "vagrant",
      "ssh_wait_timeout": "60m",
      "shutdown_command": "esxcli system maintenanceMode set -e true -t 0 ; esxcli system shutdown poweroff -d 10 -r 'Packer Shutdown' ; esxcli system maintenanceMode set -e false -t 0",
      "http_directory": ".",
      "boot_wait": "5s",
      "vmx_data": {
        "memsize": "4096",
        "numvcpus": "2",
        "vhv.enable": "TRUE"
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
  ],
  "post-processors": [
   {
     "type": "vagrant-vmware-ovf",
     "compression_level": 9,
     "output": "{% raw %}{{.BuildName}}{% endraw %}-{% raw %}{{.Provider}}{% endraw %}-{% raw %}{{user `version`}}{% endraw %}.box"

   }
  ]
}
{{< /highlight >}}

   There are several things I would like to point out in the esxi.json file we just created.

   1. builder - this section specifies that we will be using the "vmware-iso" builder, with VMware fusion, to create our packer template. We can modify attributes of our template virtual machine in this section:
     * disk size ( disk_size)
     * memory ( memsize )
     * vCPU count ( numvcpus )
   2. provisioners - this section specifies we will be using multiple provisioners to modify our template after it has been created:
     * a file provisioner that will copy the vagrant public ssh key into our ESXi template
     * a shell script to install the vmware tools VIB for nested ESXi
     * a shell script to make necessary MAC address changes in our nested ESXi template.
   3. post-processors - this final section will convert the virtual machine artifact generated in VMware fusion into a vmware-ovf .box file we can use with vagrant

* Validate the packer esxi.json file is ready for building by running the following command:

{{< highlight bash >}}
sdorsett-mbp:packer-templates sdorsett$ packer validate templates/esxi.json
Template validated successfully.
{{< /highlight >}}

* Start the build by running:
{{< highlight bash >}}
sdorsett-mbp:packer-templates sdorsett$ packer build templates/esxi.json
esxi55 output will be in this color.

==> esxi55: Downloading or copying ISO
    esxi55: Downloading or copying: file:///Users/sdorsett/Documents/packer/packer-templates/iso/VMware-VMvisor-Installer-5.5.0-1331820.x86_64.iso
==> esxi55: Creating virtual machine disk
==> esxi55: Building and writing VMX file
==> esxi55: Starting HTTP server on port 8351
==> esxi55: Starting virtual machine...
    esxi55: The VM will be run headless, without a GUI. If you want to
    esxi55: view the screen of the VM, connect via VNC without a password to
    esxi55: 127.0.0.1:5986
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
    esxi55: diff: can't stat '/tmp/auto-backup.35216//etc/ssh/keys-root/authorized_keys': No such file or directory
    esxi55: Saving current state in /bootbank
    esxi55: Clock updated.
    esxi55: Time: 02:09:44   Date: 12/30/2014   UTC
==> esxi55: Gracefully halting virtual machine...
    esxi55: Waiting for VMware to clean up after itself...
==> esxi55: Deleting unnecessary VMware files...
    esxi55: Deleting: output-esxi55/564d2ab2-395b-a9ba-9c17-2fe36682237c.vmem
    esxi55: Deleting: output-esxi55/esxi55.plist
    esxi55: Deleting: output-esxi55/vmware.log
==> esxi55: Cleaning VMX prior to finishing up...
    esxi55: Unmounting floppy from VMX...
    esxi55: Detaching ISO from CD-ROM device...
==> esxi55: Compacting the disk image
==> esxi55: Running post-processor: vagrant-vmware-ovf
==> esxi55 (vagrant-vmware-ovf): Creating Vagrant box for 'vmware_ovf' provider
    esxi55 (vagrant-vmware-ovf): Deleting key: ide1:0.filename
    esxi55 (vagrant-vmware-ovf): Deleting key: floppy0.present
    esxi55 (vagrant-vmware-ovf): Setting key: floppy0.present = FALSE
    esxi55 (vagrant-vmware-ovf): Setting key: ide1:0.present = FALSE
    esxi55 (vagrant-vmware-ovf): Creating directory: output-esxi55/ovf
    esxi55 (vagrant-vmware-ovf): Starting ovftool
    esxi55 (vagrant-vmware-ovf): Reading files in output-esxi55/ovf
    esxi55 (vagrant-vmware-ovf): Copying: esxi55-disk1.vmdk
    esxi55 (vagrant-vmware-ovf): Copying: esxi55.mf
    esxi55 (vagrant-vmware-ovf): Copying: esxi55.ovf
    esxi55 (vagrant-vmware-ovf): Compressing: Vagrantfile
    esxi55 (vagrant-vmware-ovf): Compressing: esxi55-disk1.vmdk
    esxi55 (vagrant-vmware-ovf): Compressing: esxi55.mf
    esxi55 (vagrant-vmware-ovf): Compressing: esxi55.ovf
    esxi55 (vagrant-vmware-ovf): Compressing: metadata.json
Build 'esxi55' finished.

==> Builds finished. The artifacts of successful builds are:
--> esxi55: 'vmware_ovf' provider box: esxi55-vmware_ovf-1.0.box
sdorsett-mbp:packer-templates sdorsett$
{{< /highlight >}}

If you want to keep an eye on the build process you can connect a VNC client to the address and port listed during the packer build process. For my build this was what was displayed:

{{< highlight bash >}}
    esxi55: The VM will be run headless, without a GUI. If you want to
    esxi55: view the screen of the VM, connect via VNC without a password to
    esxi55: 127.0.0.1:5986
{{< /highlight >}}

When I connected the "Chicken of the VNC" client installed on my macbook to "127.0.0.1:5986" I could see where the build was at during the install process:

![screenshot](/static/01-esxi-vnc-client.png)  

* Once the packer build completes you should end up with a "esxi55-vmware_ovf-1.0.box" file in the same directory you ran the "packer build" command in. This .box file can be used with vagrant and the gosddc vagrant providers to deploy this template to ESXi, vCenter, vCloud Director and vCloud Air.

{{< highlight bash >}}
sdorsett-mbp:packer-templates sdorsett$ ls *.box
esxi55-vmware_ovf-1.0.box
{{< /highlight >}}

#### Hopefully you found this post helpful in getting packer and the packer vagrant-vmware-ovf post processor installed/configured. The next blog post will be covering how to install packer on CentOS and build a packer virtual machine on a remote ESXi host.

---

#### Please provide any feedback or suggestions to my twitter account located on the about page.

