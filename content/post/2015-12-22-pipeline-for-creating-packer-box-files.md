+++
title = "Setting up a pipeline for creating Packer .box files"
description = ""
tags = [
    "packer",
]
date = "2015-12-22"
categories = [
    "packer",
]
highlight = "true"
+++

Recently at work, the vCloud Air Zombie team has been using Packer to generate Vagrant templates for use in development and testing.  I have previously covered <a href="../2015-01-03-using-packer-on-centos">how to use Packer to create create a .box template for use with Vagrant</a>, but I thought it might be useful to others to demonstrate how we are using Packer to create images.

This will be the first of several blog posts in which I intend to cover:

* <a href="../2015-12-23-installing-esxi-virtual-machine-for-packer-depolyment">Installing a ESXi 6.0 virtual machine for use with Packer</a> - this is where Packer will be creating the virtual machine image.
* <a href="../2015-12-24-installing-packer-and-ovftool-on-centos">Setting up Packer, ovftool and Apache web server on a CentOS virtual machine</a> - this will be where we will be editing and running the Packer templates.
* <a href="../2015-12-25-creating-a-packer-template-for-installing-centos-67">Creating our first Packer template for installing CentOS 6.7 with vmtools</a> - templates are the instructions for how a Packer image should be built.
* <a href="../2015-12-26-copy-our-existing-template-and-add-the-puppet-agent">Copying our existing CentOS 6.7 template and adding the Puppet agent</a> - having the puppet agent in an image allows us to use puppet to describe configurations using puppet in either Packer and Vagrant.
* <a href="../2015-12-27-using-ovftool-to-export-packer-generated-virtual-machines">Using ovftool to convert Packer generated virtual machines into Vagrant .box files</a> - ovftool allows you to export the Packer created images in either a Fusion or vSphere compatible format.
* <a href="../2015-12-28-scripted-packer-build-and-export">Scripted Packer build, ovftool export and Vagrant .box file creation</a> - pulling everything we have done with Packer template creation and ovftool export into a single script

I have also created a github repository to contain the Packer configuration files using during this series. You can access the github repository at [https://github.com/sdorsett/packer-templates](https://github.com/sdorsett/packer-templates).

These posts will hopefully be helpful by providing details on the process of using Packer to generate Vagrant .box files.
