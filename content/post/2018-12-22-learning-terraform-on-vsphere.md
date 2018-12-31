+++
title = "Learning terraform by deploying to vsphere"
description = ""
tags = [
    "terraform",
    "vsphere",
]
date = "2018-12-22"
categories = [
    "terraform",
    "vsphere",
]
topics = [
    "learning-terraform-by-deploying-to-vsphere"
]
highlight = "true"
+++

This post is the beginning of a series of posts that will walt through how to use terraform to deploy and configure virtual machines on vsphere. During this series of posts I will try to show how to do this deployment in a generic way in order to keep the terraform code as free as possible of the details of the environment being deployed into. 

This series assumes you have the following:

* a vsphere instance that you have permissions to create virtual machines on. Terraform can create and configure resources on ESXi, but this series will require vsphere since we will be cloning virtual machines from a base template.
* terraform is installed on your workstation, laptop or a virtual machine that can access the vsphere instance
* a github account for deploying terraform code to (optional)

The posts in this series of blogs that have been released are:

 - <a href="../2018-12-23-using-environment-variables-with-terraform">Using environment variables with terraform</a>
 - <a href="../2018-12-24-using-terraform-to-clone-a-virtual-machine-on-vsphere">Using terraform to clone a virtual machine on vSphere</a>
 - <a href="../2018-12-24-using-gitignore-to-keep-terraform-secrets-secret">Using .gitignore to keep terraform secrets secret</a>
 - <a href="../2018-12-26-using-local-exec-and-remote-exec-provisioners-with-terraform">Using local-exec and remote-exec provisioners with terraform</a>
 - <a href="../2018-12-28-using-an-external-data-source-with-terraform">Using an exteral data source with terraform</a>
 - <a href="../2018-12-30-adding-kubernetes-nodes-and-exploring-terraform-interpolation-syntax">Adding kubernetes nodes and exploring terraform interpolation syntax</a>

---

#### Please provide any feedback or suggestions to my twitter account located on the about page.
