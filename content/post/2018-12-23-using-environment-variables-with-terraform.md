+++
title = "Using environment variables with terraform"
description = ""
tags = [
    "terraform",
]
date = "2018-12-23"
categories = [
    "terraform",
]
topics = [
    "learning-terraform-by-deploying-to-vsphere"
]
highlight = "true"
+++
This is the first in a series of posts that will walk you through using terraform to deploy and configure virtual machines on vsphere. In this post you will get introduced to using environment variables to keep details obout the vsphere infrastructure out of the terraform code. 

There is nothing vsphere specific in this post, but it is more about showing a pattern for keeping deployment specifics out of terraform code.

---

### 1. Create a new directory for our terraform code.

{{< highlight bash >}}
[root@terraform ~]# mkdir terraform-test
[root@terraform ~]# cd terraform-test/
[root@terraform terraform-test]#
{{< /highlight >}}

### 2. Create a test.tf file containing our test variable and code for diplaying this value.

All terraform code needs to be stored in files that end with .tf. In future posts you will see us keep variables in a seperate file than other terraform code, but today We will create a single .tf file that will contains all the code for this post.  

{{< highlight bash >}}
[root@terraform terraform-test]# vim test.tf
[root@terraform terraform-test]# cat test.tf
variable "vsphere_user" {
  default = "administrator@vsphere.local"
}

resource "null_resource" "test-setting-variables" {
    provisioner "local-exec" {
        command = "echo ${var.vsphere_user}"
    }
}
[root@terraform terraform-test]#
{{< /highlight >}}

The the code we defined the following:

* a vsphere_user variable and set the default value to 'administrator@vsphere.local'.
* a 'null-resource' resource that will use that local-exec provisioner to locally execute the `echo` command and output the value of the vsphere_user variable.

### 3. Run `terraform init` to download the terraform provisioners used in our terraform code.

In order to ensure all terraform provisioners and providers referenced are present, we need to run `terraform init` to prepare terraform to be run. 

{{< highlight bash >}}
[root@terraform terraform-test]# terraform init

Initializing provider plugins...
- Checking for available provider plugins on https://releases.hashicorp.com...
- Downloading plugin for provider "null" (1.0.0)...

The following providers do not have any version constraints in configuration,
so the latest version was installed.

To prevent automatic upgrades to new major versions that may contain breaking
changes, it is recommended to add version = "..." constraints to the
corresponding provider blocks in configuration, with the constraint strings
suggested below.

* provider.null: version = "~> 1.0"

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
[root@terraform terraform-test]#
{{< /highlight >}}

The output shows that the terraform null provider has been successfully downloaded and installed.

### 4. Run `terraform plan` to have terraform show what changes will be made

The output of the previous step showed what provisioners were installed, and also that we now can run 'terraform plan' to see any changes that terraform will make.

{{< highlight bash >}}
[root@terraform terraform-test]# terraform plan
Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.


------------------------------------------------------------------------

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  + null_resource.test-setting-variables
      id: <computed>


Plan: 1 to add, 0 to change, 0 to destroy.

------------------------------------------------------------------------

Note: You didn't specify an "-out" parameter to save this plan, so Terraform
can't guarantee that exactly these actions will be performed if
"terraform apply" is subsequently run.

[root@terraform terraform-test]#
{{< /highlight >}}

### 5. Run `terraform apply` to apply the terraform code. 

{{< highlight bash >}}
[root@terraform terraform-test]# terraform apply

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  + null_resource.test-setting-variables
      id: <computed>


Plan: 1 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

null_resource.test-setting-variables: Creating...
null_resource.test-setting-variables: Provisioning with 'local-exec'...
null_resource.test-setting-variables (local-exec): Executing: ["/bin/sh" "-c" "echo administrator@vsphere.local"]
null_resource.test-setting-variables (local-exec): administrator@vsphere.local
null_resource.test-setting-variables: Creation complete after 0s (ID: 952552758006097355)

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
[root@terraform terraform-test]#
{{< /highlight >}}

The output of `terraform apply` shows the value of the vsphere_user variable :

{{< highlight bash >}}
null_resource.test-setting-variables (local-exec): administrator@vsphere.local
{{< /highlight >}}

This variable is still set to the default value of 'administrator@vsphere.local' since we have not updated it to be anything else.

### 6. Set an environment variable to to override the default value of the vsphere_user variable.

Terraform allows you to override the default value of a defined variable by setting an environment variable of `TF_VAR_[variable_name]`. For our example we can update the value of vsphere_user by creating an environment variable TF_VAR_vsphere_user and seeing it to the different value.  

{{< highlight bash >}}
[root@terraform terraform-test]# export TF_VAR_vsphere_user='terraform_user@vsphere.local'
[root@terraform terraform-test]# echo $TF_VAR_vsphere_user
terraform_user@vsphere.local
[root@terraform terraform-test]#
{{< /highlight >}}

In the abot example we set `TF_VAR_vsphere_user` to use to 'terraform_user@vsphere.local' value.

### 7. Taint the terraform resource.

In order to have terraform update the value that was previously set in the 'test-setting-variables' null-resource, we will need to have terraform taint this resource. Tainting the resource causes terraform to invalidate the state of this resource and will force this resource to be updated on future runs of `terraform apply` 

{{< highlight bash >}}
[root@terraform terraform-test]# terraform taint null_resource.test-setting-variables
The resource null_resource.test-setting-variables in the module root has been marked as tainted!
[root@terraform terraform-test]#
{{< /highlight >}}

### 8. Rerun a `terraform apply` to validate the value of the vsphere_user variable has changed.

This time we run terraform, we can run it with the '--auto-approve' option to have have us type 'yes' to confirm we want the have the apply performed.

{{< highlight bash >}}
[root@terraform terraform-test]# terraform apply --auto-approve
null_resource.test-setting-variables: Refreshing state... (ID: 952552758006097355)
null_resource.test-setting-variables: Destroying... (ID: 952552758006097355)
null_resource.test-setting-variables: Destruction complete after 0s
null_resource.test-setting-variables: Creating...
null_resource.test-setting-variables: Provisioning with 'local-exec'...
null_resource.test-setting-variables (local-exec): Executing: ["/bin/sh" "-c" "echo terraform_user@vsphere.local"]
null_resource.test-setting-variables (local-exec): terraform_user@vsphere.local
null_resource.test-setting-variables: Creation complete after 0s (ID: 4928338939744006620)

Apply complete! Resources: 1 added, 0 changed, 1 destroyed.
[root@terraform terraform-test]#
{{< /highlight >}}

You can see in the output that the value of vsphere_user has been updated to 'terraform_user@vsphere.local':

{{< highlight bash >}}
null_resource.test-setting-variables (local-exec): terraform_user@vsphere.local
{{< /highlight >}}

Overriding terraform variables with environment variables is a good way to: 

* Help prevent details about the infrastructure that terraform will be run against from being 'hard-coded' in the terraform code. 
* Make terraform code more reusable across multiple environments. For instance what the vsphere server should be used for deployment or what DNS servers should be used by virtual machines.. 
* Help prevent secrets (usernames, password, ssh keys) from being defined in the terraform code and ultimately being stored in version control.

#### I hope this post has been useful in helping explain how to use environment variables to override terraform variables. This pattern will be used in future posts to keep the details of how to connect to vsphere from being defined in terraform code.

---

#### Please provide any feedback or suggestions to my twitter account located on the about page.
