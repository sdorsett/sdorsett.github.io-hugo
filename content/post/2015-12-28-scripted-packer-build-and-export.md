+++
title = "Scripted Packer build, ovftool export and Vagrant .box file creation"
description = ""
tags = [
    "packer",
    "ovftool",
    "vagrant",
]
date = "2015-12-28"
categories = [
    "packer",
    "ovftool",
    "vagrant",
]
highlight = "true"
+++

This is the seventh in a series of posts on <a href="../2015-12-22-pipeline-for-creating-packer-box-files">using a Packer pipeline to generate Vagrant .box files</a>.

In the last post we covered <a href="../2015-12-27-using-ovftool-to-export-packer-generated-virtual-machines">using ovftool to convert Packer generated virtual machines into Vagrant .box files</a>. I promised to show you a better way of exporting and creating the Vagrant .box files, so in this post we will be combining the following items in one script:

* Kicking off the Packer build of a specific template
* Exporting the Packer generated virtual machine
* Creating the necessary metadata.json & Vagrantfile files
* Compressing the files into a TAR file with the .box extension
* Copying the Vagrant .box files to /vat/www/html/box-files/ so they can accessed by HTTP

Let's get started...

### 1. Start off by connecting by SSH to CentOS virtual machine we have been using in previous posts.

{{< highlight bash >}}
sdorsett-mbp:~ sdorsett$ ssh root@192.168.1.52
root@192.168.1.52's password:
Last login: Sat Dec 26 22:04:20 2015 from 192.168.1.163
[root@packer-centos ~]#
{{< /highlight >}}

### 2. Create a new directory, in the packer-templates directory, for storing build scripts.

{{< highlight bash >}}
[root@packer-centos ~]# mkdir ~/packer-templates/build-scripts
[root@packer-centos ~]# cd packer-templates/build-scripts/
[root@packer-centos build-scripts]#
{{< /highlight >}}

### 3. Next we will create a generic script for performing a Packer build of a template.

{{< highlight bash >}}
[root@packer-centos build-scripts]# cat generic-packer-build-script.sh
#!/bin/bash

source /root/.bashrc

echo "starting packer build of $PACKER_VM_NAME"
packer build -var-file=packer-remote-info.json /root/packer-templates/templates/$PACKER_VM_NAME.json

echo "registering ${PACKER_VM_NAME} virtual machine on ${PACKER_REMOTE_HOST}"
/usr/bin/sshpass -p ${PACKER_REMOTE_PASSWORD} ssh root@${PACKER_REMOTE_HOST} "vim-cmd solo/registervm /vmfs/volumes/${PACKER_REMOTE_DATASTORE}/output-${PACKER_VM_NAME}/*.vmx"

mkdir -p /root/box_files/ovf/empty_dir/
mkdir -p /root/box_files/vmx/empty_dir/

rm -rf /root/box_files/ovf/${PACKER_VM_NAME}
rm -rf /root/box_files/vmx/${PACKER_VM_NAME}

echo "output of /vmfs/volumes/${PACKER_REMOTE_DATASTORE}/output-${PACKER_VM_NAME}/*.vmxf:"
/usr/bin/sshpass -p ${PACKER_REMOTE_PASSWORD} ssh root@${PACKER_REMOTE_HOST} "cat /vmfs/volumes/${PACKER_REMOTE_DATASTORE}/output-${PACKER_VM_NAME}/*.vmxf"

ovftool vi://root:${PACKER_REMOTE_PASSWORD}@${PACKER_REMOTE_HOST}/${PACKER_VM_NAME} /root/box_files/ovf/
ovftool -tt=vmx vi://root:${PACKER_REMOTE_PASSWORD}@${PACKER_REMOTE_HOST}/${PACKER_VM_NAME} /root/box_files/vmx/

echo "creating metadata.json and Vagrantfile files in ovf virtual machine directory"
echo '{"provider":"vmware_ovf"}' >> /root/box_files/ovf/${PACKER_VM_NAME}/metadata.json
touch /root/box_files/ovf/${PACKER_VM_NAME}/Vagrantfile
cd /root/box_files/ovf/empty_dir/
cd /root/box_files/ovf/${PACKER_VM_NAME}/

echo "compressing ovf virtual machine files to /var/www/html/box-files/${PACKER_VM_NAME}-vmware_ovf-1.0.box"
tar cvzf /var/www/html/box-files/$PACKER_VM_NAME-vmware_ovf-1.0.box ./*

echo "creating metadata.json and Vagrantfile files in vmx virtual machine directory"
echo '{"provider":"vmware_desktop"}' >> /root/box_files/vmx/${PACKER_VM_NAME}/metadata.json
touch /root/box_files/vmx/${PACKER_VM_NAME}/Vagrantfile
cd /root/box_files/vmx/empty_dir/
cd /root/box_files/vmx/${PACKER_VM_NAME}/

echo "compressing vmx virtual machine files to /var/www/html/box-files/${PACKER_VM_NAME}-vmware_desktop-1.0.box"
tar cvzf /var/www/html/box-files/$PACKER_VM_NAME-vmware_desktop-1.0.box ./*

echo "cleaning up /root/box_files directories"
rm -rf /root/box_files/ovf/$PACKER_VM_NAME
rm -rf /root/box_files/vmx/$PACKER_VM_NAME

echo "deleting $PACKER_VM_NAME from $PACKER_REMOTE_HOST"
/usr/bin/sshpass -p ${PACKER_REMOTE_PASSWORD} ssh root@${PACKER_REMOTE_HOST}  "vim-cmd vmsvc/getallvms | grep ${PACKER_VM_NAME} | cut -d ' ' -f 1 | xargs vim-cmd vmsvc/destroy"

echo "packer build of $PACKER_VM_NAME has been  completed"

[root@packer-centos build-scripts]#
{{< /highlight >}}

If you look over this script you will notice it covers all of the tasks we performed manually in the last two posts. You might all notice the following environmental variables listed:

* PACKER\_REMOTE\_HOST
* PACKER\_REMOTE\_USERNAME
* PACKER\_REMOTE\_PASSWORD
* PACKER\_REMOTE\_DATASTORE
* PACKER\_VM\_NAME

We are again using the idea of "separating data from code" to keep environment specific information out of the scripts. This will help in keeping sensitive information from being store with our scripts in a code repository.

### 4. We now need to added the above listed PACKER\_REMOTE environmental variables into the .bashrc file, so the script will know how to connect to vCenter

Add the following lines to the bottom of /root/.bashrc. Make sure to update any of them that do you match your environment:

{{< highlight bash >}}
PACKER_REMOTE_HOST=192.168.1.51
PACKER_REMOTE_USERNAME=root
PACKER_REMOTE_PASSWORD=password
PACKER_REMOTE_DATASTORE=datastore1
{{< /highlight >}}

After updating the ~/.bashrc file, reread the file by running the following command:

{{< highlight bash >}}
[root@packer-centos build-scripts]# source ~/.bashrc
[root@packer-centos build-scripts]#
{{< /highlight >}}

You can also validate the environmental variables exist by using the echo commmand:

{{< highlight bash >}}
[root@packer-centos build-scripts]# echo $PACKER_REMOTE_HOST
192.168.1.51
[root@packer-centos build-scripts]#
{{< /highlight >}}

### 5. We need to delete the existing Packer generated virtual machines from the ESXi virtual machine embedded host client, since the Packer build will halt if the virtual machine it is trying to build already exists:

![screenshot](/static/01-delete-virtual-machine.png)  

### 6. With the generic Packer build script and environmental variables in place we can test our script out. First off we need to ensure the execution bit is set on our generic Packer build script:

{{< highlight bash >}}
[root@packer-centos build-scripts]# cd ~/packer-templates/
[root@packer-centos packer-templates]# chmod +x build-scripts/generic-packer-build-script.sh
[root@packer-centos packer-templates]# ls -la build-scripts/generic-packer-build-script.sh
-rwxr-xr-x 1 root root 2091 Dec 26 22:27 build-scripts/generic-packer-build-script.sh
[root@packer-centos packer-templates]#
{{< /highlight >}}

Now that we have set the script as executable, we can use it to build to centos67 template by passing in the PACKER_VM_NAME environmental variable like the following:

{{< highlight bash >}}
[root@packer-centos packer-templates]# PACKER_VM_NAME=centos67 build-scripts/generic-packer-build-script.sh
starting packer build of centos67
centos67 output will be in this color.

==> centos67: Downloading or copying ISO
    centos67: Downloading or copying: file:///root/packer-templates/iso/CentOS-6.7-x86_64-minimal.iso
==> centos67: Uploading ISO to remote machine...
==> centos67: Creating virtual machine disk
==> centos67: Building and writing VMX file
==> centos67: Starting HTTP server on port 8001
==> centos67: Registering remote VM...
==> centos67: Starting virtual machine...
==> centos67: Waiting 5s for boot...
==> centos67: Connecting to VM via VNC
==> centos67: Typing the boot command over VNC...
==> centos67: Waiting for SSH to become available...
...
{{< /highlight >}}

Checking the embedded host client we can see that the CentOS 6.7 operating system is being installed.

![screenshot](/static/02-centos67-os-install.png)

Once the Packer build completes you will see the following output:

{{< highlight bash >}}
centos67: ==> Zero out the free space to save space in the final image
centos67: dd: writing `/EMPTY': No space left on device
centos67: 12950+0 records in
centos67: 12949+0 records out
centos67: 13578776576 bytes (14 GB) copied, 462.514 s, 29.4 MB/s
==> centos67: Gracefully halting virtual machine...
centos67: Waiting for VMware to clean up after itself...
==> centos67: Deleting unnecessary VMware files...
centos67: Deleting: /vmfs/volumes/datastore1/output-centos67/vmware.log
==> centos67: Cleaning VMX prior to finishing up...
centos67: Unmounting floppy from VMX...
centos67: Detaching ISO from CD-ROM device...
centos67: Disabling VNC server...
==> centos67: Compacting the disk image
==> centos67: Unregistering virtual machine...
Build 'centos67' finished.

==> Builds finished. The artifacts of successful builds are:
--> centos67: VM files in directory: /vmfs/volumes/datastore1/output-centos67
{{< /highlight >}}

At this point the script will re-register the vm Packer built and output the .vmxf file of the virtual machine. This is use for ensuring vmtools was properly installed, since this file contains the vmtool components and their versions that were successfully installed:

{{< highlight bash >}}
registering centos67 virtual machine on 192.168.1.51
18
output of /vmfs/volumes/datastore1/output-centos67/*.vmxf:
<?xml version="1.0"?>
<Foundry><VM/><tools-install-info><installError>0</installError><updateCounter>1</updateCounter></tools-install-info><tools-manifest><monolithic version="9.9.4"/><svga33 version="10.3.0.0" installed="FALSE"/><svga4 version="10.4.0.0" installed="FALSE"/><vmmouse42 version="1.0.0.0" installed="FALSE"/><svga42 version="10.10.2.0" installed="FALSE"/><vmmouse43 version="12.6.4.0" installed="FALSE"/><svga43 version="10.16.7.0" installed="FALSE"/><vmmouse43_64 version="12.6.4.0" installed="FALSE"/><svga43_64 version="10.16.7.0" installed="FALSE"/><vmmouse67 version="12.6.4.0" installed="FALSE"/><svga67 version="10.16.7.0" installed="FALSE"/><vmmouse67_64 version="12.6.4.0" installed="FALSE"/><svga67_64 version="10.16.7.0" installed="FALSE"/><vmmouse68 version="12.6.4.0" installed="FALSE"/><svga68 version="10.16.7.0" installed="FALSE"/><vmmouse68_64 version="12.6.4.0" installed="FALSE"/><svga68_64 version="10.16.7.0" installed="FALSE"/><vmmouse70 version="12.7.0.0" installed="FALSE"/><svga70 version="11.0.99.4" installed="FALSE"/><vmmouse70_64 version="12.7.0.0" installed="FALSE"/><svga70_64 version="11.0.99.4" installed="FALSE"/><vmmouse71 version="12.7.0.0" installed="FALSE"/><svga71 version="11.0.99.4" installed="FALSE"/><vmmouse71_64 version="12.7.0.0" installed="FALSE"/><svga71_64 version="11.0.99.4" installed="FALSE"/><vmmouse73 version="12.7.0.0" installed="FALSE"/><svga73 version="11.0.99.4" installed="FALSE"/><vmmouse73_64 version="12.7.0.0" installed="FALSE"/><svga73_64 version="11.0.99.4" installed="FALSE"/><vmmouse73_99 version="12.7.0.0" installed="FALSE"/><svga73_99 version="11.0.99.4" installed="FALSE"/><vmmouse73_99_64 version="12.7.0.0" installed="FALSE"/><svga73_99_64 version="11.0.99.4" installed="FALSE"/><vmmouse74 version="12.7.0.0" installed="FALSE"/><svga74 version="11.0.99.4" installed="FALSE"/><vmmouse74_64 version="12.7.0.0" installed="FALSE"/><svga74_64 version="11.0.99.4" installed="FALSE"/><vmmouse75 version="12.7.0.0" installed="FALSE"/><svga75 version="11.0.99.4" installed="FALSE"/><vmmouse75_64 version="12.7.0.0" installed="FALSE"/><svga75_64 version="11.0.99.4" installed="FALSE"/><vmmouse76 version="12.7.0.0" installed="FALSE"/><svga76 version="11.0.99.4" installed="FALSE"/><vmmouse76_64 version="12.7.0.0" installed="FALSE"/><svga76_64 version="11.0.99.4" installed="FALSE"/><checkvm version="9.9.4.51858" installed="TRUE"/><vmtoolsd version="9.9.4.51858" installed="TRUE"/><upgrader version="9.9.4.51858" installed="FALSE"/><hgfsclient version="9.9.4.51858" installed="TRUE"/><hgfsmounter version="9.9.4.51858" installed="TRUE"/><vmguestlib version="9.9.4.51858" installed="TRUE"/><vmguestlibjava version="9.9.4.51858" installed="TRUE"/><toolbox-cmd version="9.9.4.51858" installed="TRUE"/><vmci version="9.6.2.0" installed="TRUE"/><vmhgfs version="1.4.20.1" installed="TRUE"/><vmmemctl version="1.2.1.2" installed="FALSE"/><vmsync version="1.1.0.1" installed="FALSE"/><vmxnet version="2.1.0.0" installed="TRUE"/><vmxnet3 version="1.3.0.0" installed="FALSE"/><vmblock version="1.1.2.0" installed="FALSE"/><vsock version="9.6.1.0" installed="TRUE"/><pvscsi version="1.2.3.0" installed="FALSE"/></tools-manifest></Foundry>
{{< /highlight >}}

The next stage of the build script will run the ovftool commands to export .vmx and .ovf format virtual machines from ESXi.

{{< highlight bash >}}
Opening VI source: vi://root@192.168.1.51:443/centos67
Opening OVF target: /root/box_files/ovf/
Writing OVF package: /root/box_files/ovf/centos67/centos67.ovf
Transfer Completed
Completed successfully
Opening VI source: vi://root@192.168.1.51:443/centos67
Opening VMX target: /root/box_files/vmx/
Writing VMX file: /root/box_files/vmx/centos67/centos67.vmx
Transfer Completed
Completed successfully
{{< /highlight >}}

The next stage of the build script will create the metadata.json and Vagrantfiles needed for the Vagrant .box files. It will then TAR up all the virtual machine files contained in the ovf & vmx folders. This stage will also save the TAR files to /var/www/html/box-files/ so that they will be available over http.

{{< highlight bash >}}
creating metadata.json and Vagrantfile files in ovf virtual machine directory
compressing ovf virtual machine files to /var/www/html/box-files/centos67-vmware_ovf-1.0.box
./centos67-disk1.vmdk
./centos67.mf
./centos67.ovf
./metadata.json
./Vagrantfile
creating metadata.json and Vagrantfile files in vmx virtual machine directory
compressing vmx virtual machine files to /var/www/html/box-files/centos67-vmware_desktop-1.0.box
./centos67-disk1.vmdk
./centos67.vmx
./metadata.json
./Vagrantfile
{{< /highlight >}}

The final section of the build script will remove the directories that ovftool generated in /root/box_files (to keep disk space usage to a minimum) and delete the Packer virtual machine from ESXi.

{{< highlight bash >}}
cleaning up /root/box_files directories
deleting centos67 from 192.168.1.51
packer build of centos67 has been  completed
[root@packer-centos packer-templates]#

{{< /highlight >}}

### 6. We can now list the files in /var/www/html/box-files and see the two .box files generated by the script.

{{< highlight bash >}}

[root@packer-centos packer-templates]# ls -lah /var/www/html/box-files/
total 654M
drwxr-xr-x 2 root root 4.0K Dec 28 15:44 .
drwxr-xr-x 3 root root 4.0K Dec 23 18:09 ..
-rw-r--r-- 1 root root 319M Dec 28 15:46 centos67-vmware_desktop-1.0.box
-rw-r--r-- 1 root root 336M Dec 28 15:44 centos67-vmware_ovf-1.0.box
[root@packer-centos packer-templates]#
{{< /highlight >}}

We can also open a browser and see the .box files the script generated.

![screenshot](/static/01-browse-vagrant-box-files.png)  


###If you would like to not have to manually create the build script covered in this post, you can clone down [this github repository](https://github.com/sdorsett/packer-templates/tree/adding-build-script) by running the following command:

{{< highlight bash >}}
git clone -b "adding-build-script" https://github.com/sdorsett/packer-templates.git
{{< /highlight >}}  

If you cloned the packer-templates repo in the last post, you can pull down the updates by running the following command:

{{< highlight bash >}}
git fetch --all
git pull origin "adding-build-script"
{{< /highlight >}}  

#### This brings us to the end of this post. I hope I made good on my promise of showing an automated ways of converting the Packer templates to .box files. In the next post I think we should cover how to test our Vagrant .box files are functional by deploying them using Vagrant.

---

#### Please provide any feedback or suggestions to my twitter account located on the about page.
