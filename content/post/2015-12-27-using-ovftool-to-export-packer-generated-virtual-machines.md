+++
title = "Using ovftool to convert Packer generated virtual machines into Vagrant .box files"
description = ""
tags = [
    "vagrant",
    "packer",
    "ovftool",
]
date = "2015-12-27"
categories = [
    "vagrant",
    "packer",
    "ovftool",
]
highlight = "true"
+++

This is the sixth in a series of posts on <a href="../2015-12-22-pipeline-for-creating-packer-box-files">using a Packer pipeline to generate Vagrant .box files</a>.

In the last post we covered <a href="../2015-12-26-copy-our-existing-template-and-add-the-puppet-agent">copying our existing CentOS 6.7 template and adding the Puppet agent</a> in order to generate a new Packer template. In this post we will be covering how to use ovftool to convert Packer generated virtual machines into Vagrant .box files. This post will be going over the manual steps on purpose, since I feel it will make more sense when we start to cover automating the steps that you can already performed by hand.

Let's get started...

### 1. Start off by connecting by SSH to CentOS virtual machine we have been using in previous posts.

{{< highlight bash >}}
sdorsett-mbp:~ sdorsett$ ssh root@192.168.1.52
root@192.168.1.52's password:
Last login: Sat Dec 26 20:52:17 2015 from 192.168.1.163
[root@packer-centos ~]#
{{< /highlight >}}

### 2. Create two new directories for downloading virtual machines, one for .ovf and one for .vmx virtual machine formats.

{{< highlight bash >}}
root@packer-centos ~]# mkdir -p ~/box_files/{ovf,vmx}
[root@packer-centos ~]# tree ~/box_files/
/root/box_files/
├── ovf
└── vmx

2 directories, 0 files
[root@packer-centos ~]#
{{< /highlight >}}

### 3. Now that we have locations for our ovftool exported virtual machine, we need to re-register the Packer created virtual machines. SSH to your ESXi virtual machine and cd to the datastore that you created the Packer virtual machines on.

{{< highlight bash >}}
sdorsett-mbp:Desktop sdorsett$ ssh root@192.168.1.51
Password:
The time and date of this login have been sent to the system logs.

VMware offers supported, powerful system administration tools.  Please
see www.vmware.com/go/sysadmintools for details.

The ESXi Shell can be disabled by an administrative user. See the
vSphere Security documentation for more information.
[root@packer-esxi:~] cd /vmfs/volumes/datastore1/
[root@packer-esxi:/vmfs/volumes/5679efd9-3e1658e4-f977-0050569b912b] ls output-*
output-centos67:
centos67-flat.vmdk  centos67.vmdk       centos67.vmx
centos67.nvram      centos67.vmsd       centos67.vmxf

output-centos67-pe-puppet-382:
centos67-pe-puppet-382-flat.vmdk  centos67-pe-puppet-382.vmsd
centos67-pe-puppet-382.nvram      centos67-pe-puppet-382.vmx
centos67-pe-puppet-382.vmdk       centos67-pe-puppet-382.vmxf
[root@packer-esxi:/vmfs/volumes/5679efd9-3e1658e4-f977-0050569b912b]
{{< /highlight >}}

### 4. We next need to register both of the .vmx files of the Packer virtual machines displayed above on the ESXi virtual machine.

{{< highlight bash >}}
[root@packer-esxi:/vmfs/volumes/] vim-cmd solo/registervm /vmfs
/volumes/datastore1/output-centos67/*.vmx
15
[root@packer-esxi:/vmfs/volumes/] vim-cmd solo/registervm /vmfs
/volumes/datastore1/output-centos67-pe-puppet-382/*.vmx
16
[root@packer-esxi:/vmfs/volumes/]
{{< /highlight >}}

You will notice that each of the vim-cmd commands returned a number. That number is the vmid of the virtual machine that was registered. Also if you check the embedded host client, you will see those virtual machines we just registered with the vim-cmd command listed under "Virtual Machines."

![screenshot](/static/01-registered-virtual-machines.png)  

### 5. Once the Packer virtual machines have been registered, we can use ovftool on the CentOS virtual machine to export them as .ovf files.

After the files are exported, we can then compress the .ovf files into a vmware\_ovf compatible .box file. Using the vmware\_ovf format will provide a generic .box file that can be deployed to vCenter, vCloud Director or vCloud Air Vagrant providers.

{{< highlight bash >}}
[root@packer-centos ~]# ovftool vi://root@192.168.1.51/centos67 /root/box_files/ovf/
Enter login information for source vi://192.168.1.51/
Username: root
Password: ********
Opening VI source: vi://root@192.168.1.51:443/centos67
Opening OVF target: /root/box_files/ovf/
Writing OVF package: /root/box_files/ovf/centos67/centos67.ovf
Transfer Completed
Completed successfully

[root@packer-centos ~]# ovftool vi://root@192.168.1.51/centos67-pe-puppet-382 /root/box_files/ovf/
Enter login information for source vi://192.168.1.51/
Username: root
Password: ********
Opening VI source: vi://root@192.168.1.51:443/centos67-pe-puppet-382
Opening OVF target: /root/box_files/ovf/
Writing OVF package: /root/box_files/ovf/centos67-pe-puppet-382/centos67-pe-puppet-382.ovf
Transfer Completed
Completed successfully

[root@packer-centos ~]#
{{< /highlight >}}

### 6. We can also use ovftool to export the Packer virtual machines in a .vmx format for use with VMware Fusion or Workstation:

{{< highlight bash >}}
[root@packer-centos ~]# ovftool -tt=vmx vi://root@192.168.1.51/centos67 /root/box_files/vmx/Enter login information for source vi://192.168.1.51/
Username: root
Password: ********
Opening VI source: vi://root@192.168.1.51:443/centos67
Opening VMX target: /root/box_files/vmx/
Writing VMX file: /root/box_files/vmx/centos67/centos67.vmx
Transfer Completed
Completed successfully

[root@packer-centos ~]# ovftool -tt=vmx vi://root@192.168.1.51/centos67-pe-puppet-382 /root/box_files/vmx/
Enter login information for source vi://192.168.1.51/
Username: root
Password: ********
Opening VI source: vi://root@192.168.1.51:443/centos67-pe-puppet-382
Opening VMX target: /root/box_files/vmx/
Writing VMX file: /root/box_files/vmx/centos67-pe-puppet-382/centos67-pe-puppet-382.vmx
Transfer Completed
Completed successfully

[root@packer-centos ~]#
{{< /highlight >}}

### 7. We will start with converting the .ovf exported templates.

Each of the exported template directories will need a metadata.js and Vagrantfile created. After creating the metadata.js and Vagrantfile files, we will tar all of the files in each directory into a .box file.

{{< highlight bash >}}
[root@packer-centos ~]# cd /root/box_files/ovf/centos67
[root@packer-centos centos67]# echo '{"provider":"vmware_ovf"}' >> metadata.json
[root@packer-centos centos67]# touch Vagrantfile
[root@packer-centos centos67]# tar cvzf /root/box_files/centos67-vmware_ovf-1.0.box ./*
./centos67-disk1.vmdk
./centos67.mf
./centos67.ovf
./metadata.json
./Vagrantfile

[root@packer-centos centos67]# cd /root/box_files/ovf/centos67-pe-puppet-382
[root@packer-centos centos67-pe-puppet-382]# echo '{"provider":"vmware_ovf"}' >> metadata.json
[root@packer-centos centos67-pe-puppet-382]# touch Vagrantfile
[root@packer-centos centos67-pe-puppet-382]# tar cvzf /root/box_files/centos67-pe-puppet-382-vmware_ovf-1.0.box ./*
./centos67-pe-puppet-382-disk1.vmdk
./centos67-pe-puppet-382.mf
./centos67-pe-puppet-382.ovf
./metadata.json
./Vagrantfile

[root@packer-centos centos67-pe-puppet-382]#
{{< /highlight >}}

### 8. Next we will convert the .vmx exported templates.

Each of the exported template directories will also need a slightly different metadata.js file created, Vagrantfile created and finally tar all of the files in each directory into a .box file.

{{< highlight bash >}}
[root@packer-centos centos67-pe-puppet-382]# cd /root/box_files/vmx/centos67
[root@packer-centos centos67]# echo '{"provider":"vmware_desktop"}' >> metadata.json
[root@packer-centos centos67]# touch Vagrantfile
[root@packer-centos centos67]# tar cvzf /root/box_files/centos67-vmware_desktop-1.0.box ./*
./centos67-disk1.vmdk
./centos67.vmx
./metadata.json
./Vagrantfile

[root@packer-centos centos67]# cd /root/box_files/vmx/centos67-pe-puppet-382/
[root@packer-centos centos67-pe-puppet-382]# echo '{"provider":"vmware_desktop"}' >> metadata.json
[root@packer-centos centos67-pe-puppet-382]# touch Vagrantfile
[root@packer-centos centos67-pe-puppet-382]# tar cvzf /root/box_files/centos67-pe-puppet-382-vmware_desktop-1.0.box ./*
./centos67-pe-puppet-382-disk1.vmdk
./centos67-pe-puppet-382.vmx
./metadata.json
./Vagrantfile

[root@packer-centos centos67-pe-puppet-382]#
{{< /highlight >}}

### 9. Finally we can use the tree command to see the overall directory structure of the .ovf and .vmx templates, as well as the list of the .box files:

{{< highlight bash >}}
[root@packer-centos centos67-pe-puppet-382]# tree /root/box_files/
/root/box_files/
├── centos67-pe-puppet-382-vmware_desktop-1.0.box
├── centos67-pe-puppet-382-vmware_ovf-1.0.box
├── centos67-vmware_desktop-1.0.box
├── centos67-vmware_ovf-1.0.box
├── ovf
│   ├── centos67
│   │   ├── centos67-disk1.vmdk
│   │   ├── centos67.mf
│   │   ├── centos67.ovf
│   │   ├── metadata.json
│   │   └── Vagrantfile
│   └── centos67-pe-puppet-382
│       ├── centos67-pe-puppet-382-disk1.vmdk
│       ├── centos67-pe-puppet-382.mf
│       ├── centos67-pe-puppet-382.ovf
│       ├── metadata.json
│       └── Vagrantfile
└── vmx
    ├── centos67
    │   ├── centos67-disk1.vmdk
    │   ├── centos67.vmx
    │   ├── metadata.json
    │   └── Vagrantfile
    └── centos67-pe-puppet-382
        ├── centos67-pe-puppet-382-disk1.vmdk
        ├── centos67-pe-puppet-382.vmx
        ├── metadata.json
        └── Vagrantfile

6 directories, 22 files
[root@packer-centos centos67-pe-puppet-382]#
{{< /highlight >}}

#### This brings us to the end of this post. Again I'm sorry for this post not covering any automated ways of converting the Packer templates to .box files, but in the next post you'll learn how we can ease this manual pain with some  scripts.

---

#### Please provide any feedback or suggestions to my twitter account located on the about page.
