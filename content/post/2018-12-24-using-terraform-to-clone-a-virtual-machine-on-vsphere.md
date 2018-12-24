+++
title = "Using terraform to clone a virtual machine on vSphere"
description = ""
tags = [
    "terraform",
    "vsphere",
]
date = "2018-12-24"
categories = [
    "terraform",
    "vsphere",
]
topics = [
    "learning-terraform-by-deploying-to-vsphere"
]
highlight = "true"
+++
This is the second in a series of posts that will walk you through using terraform to deploy and configure virtual machines on vsphere. In this post you will get introduced to using terraform to clone an existing template to a new virtual machine. 

---

### 1. Create a new directory for our terraform code.

Start by creating a new directory for the terraform code and `cd` to it.

{{< highlight bash >}}
[root@terraform ~]# mkdir terraform-vsphere-clone
[root@terraform ~]# cd terraform-vsphere-clone/
[root@terraform terraform-vsphere-clone]#
{{< /highlight >}}

### 2. Create `variables.tf` containing the variables that are required.

Before we start creating the terraform file I want to point out Terraform does not require specific types of information to be stored in files with specific names.Terraform loads all configuration files within a directory and appends them together, so there is no techinical reason to split the code into multiple files. The reason the code is typically broken into seperate files it to designate how to code is being used.

A common pattern with structuring terraform code is to use the same file structure that terraform modules use:

* `main.tf` - contains the terraform code that will create resources.
* `variables.tf` - defines the variables that will be used in main.tf.
* `outputs.tf` - defines what output information terraform will output once `terraform apply` has completed.

Let us start with the variables.tf file:

{{< highlight bash >}}
[root@terraform terraform-vsphere-clone]# vim variables.tf
[root@terraform terraform-vsphere-clone]# cat variables.tf
# vsphere login account. defaults to admin account
variable "vsphere_user" {
  default = "administrator@vsphere.local"
}

# vsphere account password. empty by default.
variable "vsphere_password" {}

# vsphere server, defaults to localhost
variable "vsphere_server" {
  default = "localhost"
}

# vsphere datacenter the virtual machine will be deployed to. empty by default.
variable "vsphere_datacenter" {}

# vsphere resource pool the virtual machine will be deployed to. empty by default.
variable "vsphere_resource_pool" {}

# vsphere datastore the virtual machine will be deployed to. empty by default.
variable "vsphere_datastore" {}

# vsphere network the virtual machine will be connected to. empty by default.
variable "vsphere_network" {}

# vsphere virtual machine template that the virtual machine will be cloned from. empty by default.
variable "vsphere_virtual_machine_template" {}

# the name of the vsphere virtual machine that is created. empty by default.
variable "vsphere_virtual_machine_name" {}
[root@terraform terraform-vsphere-clone]#
{{< /highlight >}}

### 3. Create `terraform.tfvars` and set the variable values.

There is nothing that prevents you from setting the specific values for variables in `variables.tf`, but this increases the chances of those values getting applied in version controt. The preferred method for setting variable values is setting them in `terraform.tfvars` or using environment variables as was covered <a href="../2018-12-23-using-environment-variables-with-terraform">in the previous blog post</a>.

For demonstrations purposes this post will use the `terraform.tfvars` method to set the specific value of the variables defined in `variables.tf`.

The following is an example of what `terraform.tfvars` should look like:

{{< highlight bash >}}
[root@terraform terraform-vsphere-clone]# vim terraform.tfvars
[root@terraform terraform-vsphere-clone]# cat terraform.tfvars
vsphere_user = "terraform_user@vsphere.local"
vsphere_password = "SuperSecretPassword"
vsphere_server = "192.168.100.50"
vsphere_datacenter = "datacenter"
vsphere_datastore = "datastore-1"
vsphere_resource_pool = "cluster/Resources"
vsphere_network = "VM Network"
vsphere_virtual_machine_template = "centos_7_template"
vsphere_virtual_machine_name = "terraform-vsphere-clone-test"
[root@terraform terraform-vsphere-clone]#
{{< /highlight >}}

### 4. Create `main.tf` containing the terraform code.

Next we will create `main.tf` to contain the terraform code that will create the resource (virtual machine) as describe by the code.

`main.tf` will contain the following types of blocks:

* `provider` - a provider block describes a terraform provider that will be used. We will be using the 'vsphere' provider
* `data` - a data block describes a resource that a provider will query and return the results. For instance we will be using vsphere_datacenter, vsphere_datastore, vsphere_resource_pool and vsphere_network data sources and using those value to create a new virtual machine
* `resource` - a resource block describes a resource that terraform will ensure exists as described. The virtual machine we are wanted to have created will be described as a resource.

The following is the `main.tf` that I used:
 
{{< highlight bash >}}
[root@terraform terraform-vsphere-clone]# cat main.tf
provider "vsphere" {
  user           = "${var.vsphere_user}"
  password       = "${var.vsphere_password}"
  vsphere_server = "${var.vsphere_server}"
  allow_unverified_ssl = true
}

data "vsphere_datacenter" "dc" {
  name = "${var.vsphere_datacenter}"
}

data "vsphere_datastore" "datastore" {
  name          = "${var.vsphere_datastore}"
  datacenter_id = "${data.vsphere_datacenter.dc.id}"
}

data "vsphere_resource_pool" "pool" {
  name          = "${var.vsphere_resource_pool}"
  datacenter_id = "${data.vsphere_datacenter.dc.id}"
}

data "vsphere_network" "network" {
  name          = "${var.vsphere_network}"
  datacenter_id = "${data.vsphere_datacenter.dc.id}"
}

data "vsphere_virtual_machine" "template" {
  name          = "${var.vsphere_virtual_machine_template}"
  datacenter_id = "${data.vsphere_datacenter.dc.id}"
}

resource "vsphere_virtual_machine" "cloned_virtual_machine" {
  name             = "${var.vsphere_virtual_machine_name}"
  resource_pool_id = "${data.vsphere_resource_pool.pool.id}"
  datastore_id     = "${data.vsphere_datastore.datastore.id}"

  num_cpus = "${data.vsphere_virtual_machine.template.num_cpus}"
  memory   = "${data.vsphere_virtual_machine.template.memory}"
  guest_id = "${data.vsphere_virtual_machine.template.guest_id}"

  scsi_type = "${data.vsphere_virtual_machine.template.scsi_type}"

  network_interface {
    network_id   = "${data.vsphere_network.network.id}"
    adapter_type = "${data.vsphere_virtual_machine.template.network_interface_types[0]}"
  }

  disk {
    label = "disk0"
    size = "${data.vsphere_virtual_machine.template.disks.0.size}"
  }

  clone {
    template_uuid = "${data.vsphere_virtual_machine.template.id}"
  }
}
[root@terraform terraform-vsphere-clone]#
{{< /highlight >}}

You might notice that the variables we defined are used in the vsphere provider block, to specify how to connect to vsphere, and the data blocks, to specify what vsphere resources should be queried. 

### 5. Create `output.tf` containing the information we want terraform to return after a run completes.

In the output.tf you can specify what information terraform should display after the run completes. 

{{< highlight bash >}}
[root@terraform terraform-vsphere-clone]# vim output.tf
[root@terraform terraform-vsphere-clone]# cat output.tf
output "ip" {
   value = "${vsphere_virtual_machine.cloned_virtual_machine.*.default_ip_address}"
}
output "vm-moref" {
   value = "${vsphere_virtual_machine.cloned_virtual_machine.moid}"
}

[root@terraform terraform-vsphere-clone]#
{{< /highlight >}}

In this example terraform will return the ip address and moref of the virtual machine created.

### 6. Run `terraform init` to initialize terraform and downloaded the provisioners.

{{< highlight bash >}}
[root@terraform terraform-vsphere-clone]# terraform init

Initializing provider plugins...
- Checking for available provider plugins on https://releases.hashicorp.com...
- Downloading plugin for provider "vsphere" (1.9.0)...

The following providers do not have any version constraints in configuration,
so the latest version was installed.

To prevent automatic upgrades to new major versions that may contain breaking
changes, it is recommended to add version = "..." constraints to the
corresponding provider blocks in configuration, with the constraint strings
suggested below.

* provider.vsphere: version = "~> 1.9"

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
[root@terraform terraform-vsphere-clone]#
{{< /highlight >}}

### 7. Run `terraform plan` to have terraform show what will get created.

The next step is to run `terraform plan` to have terraform validate the vsphere credentials and display the changes it would like to make:

{{< highlight bash >}}
[root@terraform terraform-vsphere-clone]# terraform plan
Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.

data.vsphere_datacenter.dc: Refreshing state...
data.vsphere_resource_pool.pool: Refreshing state...
data.vsphere_network.network: Refreshing state...
data.vsphere_virtual_machine.template: Refreshing state...
data.vsphere_datastore.datastore: Refreshing state...

------------------------------------------------------------------------

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  + vsphere_virtual_machine.cloned_virtual_machine
      id:                                        <computed>
      boot_retry_delay:                          "10000"
      change_version:                            <computed>
      clone.#:                                   "1"
      clone.0.template_uuid:                     "4221e272-e449-97e0-39f4-ab580bebc473"
      clone.0.timeout:                           "30"
      cpu_limit:                                 "-1"
      cpu_share_count:                           <computed>
      cpu_share_level:                           "normal"
      datastore_id:                              "datastore-15"
      default_ip_address:                        <computed>
      disk.#:                                    "1"
      disk.0.attach:                             "false"
      disk.0.datastore_id:                       "<computed>"
      disk.0.device_address:                     <computed>
      disk.0.disk_mode:                          "persistent"
      disk.0.disk_sharing:                       "sharingNone"
      disk.0.eagerly_scrub:                      "false"
      disk.0.io_limit:                           "-1"
      disk.0.io_reservation:                     "0"
      disk.0.io_share_count:                     "0"
      disk.0.io_share_level:                     "normal"
      disk.0.keep_on_remove:                     "false"
      disk.0.key:                                "0"
      disk.0.label:                              "disk0"
      disk.0.path:                               <computed>
      disk.0.size:                               "8"
      disk.0.thin_provisioned:                   "true"
      disk.0.unit_number:                        "0"
      disk.0.uuid:                               <computed>
      disk.0.write_through:                      "false"
      ept_rvi_mode:                              "automatic"
      firmware:                                  "bios"
      force_power_off:                           "true"
      guest_id:                                  "rhel7_64Guest"
      guest_ip_addresses.#:                      <computed>
      host_system_id:                            <computed>
      hv_mode:                                   "hvAuto"
      imported:                                  <computed>
      latency_sensitivity:                       "normal"
      memory:                                    "1024"
      memory_limit:                              "-1"
      memory_share_count:                        <computed>
      memory_share_level:                        "normal"
      migrate_wait_timeout:                      "30"
      moid:                                      <computed>
      name:                                      "terraform-vsphere-clone-test"
      network_interface.#:                       "1"
      network_interface.0.adapter_type:          "vmxnet3"
      network_interface.0.bandwidth_limit:       "-1"
      network_interface.0.bandwidth_reservation: "0"
      network_interface.0.bandwidth_share_count: <computed>
      network_interface.0.bandwidth_share_level: "normal"
      network_interface.0.device_address:        <computed>
      network_interface.0.key:                   <computed>
      network_interface.0.mac_address:           <computed>
      network_interface.0.network_id:            "network-25"
      num_cores_per_socket:                      "1"
      num_cpus:                                  "1"
      reboot_required:                           <computed>
      resource_pool_id:                          "resgroup-8"
      run_tools_scripts_after_power_on:          "true"
      run_tools_scripts_after_resume:            "true"
      run_tools_scripts_before_guest_shutdown:   "true"
      run_tools_scripts_before_guest_standby:    "true"
      scsi_bus_sharing:                          "noSharing"
      scsi_controller_count:                     "1"
      scsi_type:                                 "lsilogic"
      shutdown_wait_timeout:                     "3"
      swap_placement_policy:                     "inherit"
      uuid:                                      <computed>
      vmware_tools_status:                       <computed>
      vmx_path:                                  <computed>
      wait_for_guest_net_routable:               "true"
      wait_for_guest_net_timeout:                "5"


Plan: 1 to add, 0 to change, 0 to destroy.

------------------------------------------------------------------------

Note: You didn't specify an "-out" parameter to save this plan, so Terraform
can't guarantee that exactly these actions will be performed if
"terraform apply" is subsequently run.

[root@terraform terraform-vsphere-clone]#
{{< /highlight >}}

### 8. Run `terraform apply` to create the described resources.

Running `terraform apply --auto-approve` will have terraform create the resource described in `main.tf`:

{{< highlight bash >}}
[root@terraform terraform-vsphere-clone]# terraform apply --auto-approve
data.vsphere_datacenter.dc: Refreshing state...
data.vsphere_network.network: Refreshing state...
data.vsphere_resource_pool.pool: Refreshing state...
data.vsphere_virtual_machine.template: Refreshing state...
data.vsphere_datastore.datastore: Refreshing state...
vsphere_virtual_machine.cloned_virtual_machine: Creating...
  boot_retry_delay:                          "" => "10000"
  change_version:                            "" => "<computed>"
  clone.#:                                   "" => "1"
  clone.0.template_uuid:                     "" => "4221e272-e449-97e0-39f4-ab580bebc473"
  clone.0.timeout:                           "" => "30"
  cpu_limit:                                 "" => "-1"
  cpu_share_count:                           "" => "<computed>"
  cpu_share_level:                           "" => "normal"
  datastore_id:                              "" => "datastore-15"
  default_ip_address:                        "" => "<computed>"
  disk.#:                                    "" => "1"
  disk.0.attach:                             "" => "false"
  disk.0.datastore_id:                       "" => "<computed>"
  disk.0.device_address:                     "" => "<computed>"
  disk.0.disk_mode:                          "" => "persistent"
  disk.0.disk_sharing:                       "" => "sharingNone"
  disk.0.eagerly_scrub:                      "" => "false"
  disk.0.io_limit:                           "" => "-1"
  disk.0.io_reservation:                     "" => "0"
  disk.0.io_share_count:                     "" => "0"
  disk.0.io_share_level:                     "" => "normal"
  disk.0.keep_on_remove:                     "" => "false"
  disk.0.key:                                "" => "0"
  disk.0.label:                              "" => "disk0"
  disk.0.path:                               "" => "<computed>"
  disk.0.size:                               "" => "8"
  disk.0.thin_provisioned:                   "" => "true"
  disk.0.unit_number:                        "" => "0"
  disk.0.uuid:                               "" => "<computed>"
  disk.0.write_through:                      "" => "false"
  ept_rvi_mode:                              "" => "automatic"
  firmware:                                  "" => "bios"
  force_power_off:                           "" => "true"
  guest_id:                                  "" => "rhel7_64Guest"
  guest_ip_addresses.#:                      "" => "<computed>"
  host_system_id:                            "" => "<computed>"
  hv_mode:                                   "" => "hvAuto"
  imported:                                  "" => "<computed>"
  latency_sensitivity:                       "" => "normal"
  memory:                                    "" => "1024"
  memory_limit:                              "" => "-1"
  memory_share_count:                        "" => "<computed>"
  memory_share_level:                        "" => "normal"
  migrate_wait_timeout:                      "" => "30"
  moid:                                      "" => "<computed>"
  name:                                      "" => "terraform-vsphere-clone-test"
  network_interface.#:                       "" => "1"
  network_interface.0.adapter_type:          "" => "vmxnet3"
  network_interface.0.bandwidth_limit:       "" => "-1"
  network_interface.0.bandwidth_reservation: "" => "0"
  network_interface.0.bandwidth_share_count: "" => "<computed>"
  network_interface.0.bandwidth_share_level: "" => "normal"
  network_interface.0.device_address:        "" => "<computed>"
  network_interface.0.key:                   "" => "<computed>"
  network_interface.0.mac_address:           "" => "<computed>"
  network_interface.0.network_id:            "" => "network-25"
  num_cores_per_socket:                      "" => "1"
  num_cpus:                                  "" => "1"
  reboot_required:                           "" => "<computed>"
  resource_pool_id:                          "" => "resgroup-8"
  run_tools_scripts_after_power_on:          "" => "true"
  run_tools_scripts_after_resume:            "" => "true"
  run_tools_scripts_before_guest_shutdown:   "" => "true"
  run_tools_scripts_before_guest_standby:    "" => "true"
  scsi_bus_sharing:                          "" => "noSharing"
  scsi_controller_count:                     "" => "1"
  scsi_type:                                 "" => "lsilogic"
  shutdown_wait_timeout:                     "" => "3"
  swap_placement_policy:                     "" => "inherit"
  uuid:                                      "" => "<computed>"
  vmware_tools_status:                       "" => "<computed>"
  vmx_path:                                  "" => "<computed>"
  wait_for_guest_net_routable:               "" => "true"
  wait_for_guest_net_timeout:                "" => "5"
vsphere_virtual_machine.cloned_virtual_machine: Still creating... (10s elapsed)
vsphere_virtual_machine.cloned_virtual_machine: Still creating... (20s elapsed)
vsphere_virtual_machine.cloned_virtual_machine: Still creating... (30s elapsed)
vsphere_virtual_machine.cloned_virtual_machine: Still creating... (40s elapsed)
vsphere_virtual_machine.cloned_virtual_machine: Still creating... (50s elapsed)
vsphere_virtual_machine.cloned_virtual_machine: Still creating... (1m0s elapsed)
vsphere_virtual_machine.cloned_virtual_machine: Still creating... (1m10s elapsed)
vsphere_virtual_machine.cloned_virtual_machine: Still creating... (1m20s elapsed)
vsphere_virtual_machine.cloned_virtual_machine: Still creating... (1m30s elapsed)
vsphere_virtual_machine.cloned_virtual_machine: Still creating... (1m40s elapsed)
vsphere_virtual_machine.cloned_virtual_machine: Still creating... (1m50s elapsed)
vsphere_virtual_machine.cloned_virtual_machine: Creation complete after 1m53s (ID: 42217964-1e3c-29b6-0e39-8253913d4558)

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:

ip = [
    192.168.100.178
]
vm-moref = vm-1115
[root@terraform terraform-vsphere-clone]#
{{< /highlight >}}

The bottom of the `terraform appy` displays the IP address and moref of the vm that was specified in the `output.tf` file. 

### 9. Run `terraform destroy` to delete the resources that were previously created.

Once you are ready to destroy the virtual machine terraform created, you can simply run `terraform destroy --force` to have terraform destroy it:

{{< highlight bash >}}
[root@terraform terraform-vsphere-clone]# terraform destroy --force
data.vsphere_datacenter.dc: Refreshing state...
data.vsphere_datastore.datastore: Refreshing state...
data.vsphere_virtual_machine.template: Refreshing state...
data.vsphere_network.network: Refreshing state...
data.vsphere_resource_pool.pool: Refreshing state...
vsphere_virtual_machine.cloned_virtual_machine: Refreshing state... (ID: 42217964-1e3c-29b6-0e39-8253913d4558)
vsphere_virtual_machine.cloned_virtual_machine: Destroying... (ID: 42217964-1e3c-29b6-0e39-8253913d4558)
vsphere_virtual_machine.cloned_virtual_machine: Still destroying... (ID: 42217964-1e3c-29b6-0e39-8253913d4558, 10s elapsed)
vsphere_virtual_machine.cloned_virtual_machine: Still destroying... (ID: 42217964-1e3c-29b6-0e39-8253913d4558, 20s elapsed)
vsphere_virtual_machine.cloned_virtual_machine: Destruction complete after 23s

Destroy complete! Resources: 1 destroyed.
[root@terraform terraform-vsphere-clone]#
{{< /highlight >}}

#### I hope this post has been useful in demonstrating how to clone a vsphere virtual machine from template using terraform. The .tf files that were using in this post can be found in <a href="https://github.com/sdorsett/terraform-vsphere-clone">this github repository</a>.

---

#### Please provide any feedback or suggestions to my twitter account located on the about page.
