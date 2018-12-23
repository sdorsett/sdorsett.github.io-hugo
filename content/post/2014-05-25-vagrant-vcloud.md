+++
title = "Introducing the vagrant-vcloud provider"
description = ""
tags = [
    "vagrant",
    "vagrant-vcloud",
]
date = "2014-05-25"
categories = [
    "vagrant",
    "vagrant-cloud",
]
topics = [
    "using-vagrant-with-vcloud-director",
]
highlight = "true"
+++

While continuing to explore what the vagrant-vsphere provider is capable of I came across the [vagrant-vcloud](https://github.com/frapposelli/vagrant-vcloud) provider, which had recently released a new version. I work for the vCHS operations group, so I figured it would be interesting to compare the feature differences of the vsphere & vcloud providers. 

Over the next few blog posts I intend to cover the following vagrant-vcloud provider related topics:

* <a href="../2014-05-26-vagrant-install">Installing Vagant and the vagrant-vcloud plugin on a CentOS 6.x virtual machine in vCloud Directory or vCHS</a>
* Creating a simple vagrant-vcloud Vagrantfile configuration to deploy a vm 
* Creating a more advanced vagrant-vcloud Vagrantfile configuration
* Creating a CentOS 6.x .box that is customized for Vagrant and vcloud director


These posts will hopefully explain the steps necessary to get vagrant-vcloud installed and working with a vCloud Director or vCHS environment.
