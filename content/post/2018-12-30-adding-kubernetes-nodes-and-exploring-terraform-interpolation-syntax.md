+++
title = "Adding kubernetes nodes and exploring terraform interpolation syntax"
description = ""
tags = [
    "terraform",
    "vsphere",
    "kubernetes",
]
date = "2018-12-30"
categories = [
    "terraform",
    "vsphere",
    "kubernetes",
]
topics = [
    "learning-terraform-by-deploying-to-vsphere"
]
highlight = "true"
+++
This is the sixth in a series of posts that will walk you through using terraform to deploy and configure virtual machines on vsphere. In this post I will build on the kubernetes cluster that was built in the previous post by add kubernetes nodes to it. We will also use terraform interpolation to combine variables and resource output.

---

### 1. Clone the `https://github.com/sdorsett/terraform-vsphere-kubernetes` repository and switch to the '2018-12-28-post' branch..

We will start by cloning down the `https://github.com/sdorsett/terraform-vsphere-kubernetes` repository:

{{< highlight bash >}}
[root@terraform ~]# git clone https://github.com/sdorsett/terraform-vsphere-kubernetes.git
Cloning into 'terraform-vsphere-kubernetes'...
remote: Enumerating objects: 13, done.
remote: Counting objects: 100% (13/13), done.
remote: Compressing objects: 100% (13/13), done.
remote: Total 13 (delta 0), reused 13 (delta 0), pack-reused 0
Unpacking objects: 100% (13/13), done.
[root@terraform ~]# cd terraform-vsphere-kubernetes
[root@terraform terraform-vsphere-kubernetes]#
{{< /highlight >}}

Checkout the `'2018-12-28-post'` branch which is where we left off at the end of the last post:

{{< highlight bash >}}
[root@terraform terraform-vsphere-kubernetes]# git checkout origin/2018-12-28-post
Note: checking out 'origin/2018-12-28-post'.

You are in 'detached HEAD' state. You can look around, make experimental
changes and commit them, and you can discard any commits you make in this
state without impacting any branches by performing another checkout.

If you want to create a new branch to retain commits you create, you may
do so (now or later) by using -b with the checkout command again. Example:

  git checkout -b new_branch_name

HEAD is now at 489b7b7... added updated files for 2018-12-28 post
[root@terraform terraform-vsphere-kubernetes]#
{{< /highlight >}}

### 2. Update `variables.tf` to add a new variable block for the variables we will need to descirbe the kubernetes node virtual machines. 

The variables needed for the kubernetes nodes will be similar to what we used for the kubernetes controller, but there will be need to have the following new ones:

* count - this variable will specify the number of kubernetes nodes to deploy
* ip_address_network - this variables will specify the IP address range in CIDR notation that the kubernetes nodes will be deployed to
* starting_hostnum - this variable will specify the host number, within the ip_address_network that the first kubernetes node will use. The following kubernetes nodes will increment up from this host number.

The ip_address and netmask variables are no longer needed since these pieces of information can be derived from the two new variables.

{{< highlight bash >}}
[root@terraform terraform-vsphere-kubernetes]# cat variables.tf

...[the original variables.tf lines]...

variable "virtual_machine_kubernetes_node" {
  type                      = "map"
  description               = "Configuration details for kubernetes_controller virtual machine"

  default = {
    # number of kuvernetes node virtual machines to deploy. defaults to 1
    count                   = 1
    # prefix of the virtual machine to be deployed. defaults to "kubernetes-node"
    name_prefix             = "kubernetes-node-"
    # name of the datastore to deploy kubernetes_node virtual machines to. defaults to "datastore1"
    datastore               = "datastore1"
    # name of network to deploy kubernetes_node virtual machines to. defaults to "VM Network"
    network                 = "VM Network"
    # the ip address network that will be used to determine the ip address assigned to kubernetes_node virtual machines. defaults to "192.168.100.0/24"
    ip_address_network       = "192.168.100.0/24"
    # the start_hostnum should be set to the 4th octet of the first kubernetes_node virtual machine. this will be combined with the count.index to determine the 4th octet of the ip address for each kuberenetes_node virtual machine. defaults to "101"
    starting_hostnum      = "101"
    # dns server assigned to kubernetes_node virtual machines. defaults to "8.8.8.8"
    dns_server              = "8.8.8.8"
    # default gateway to be assigned to kubernetes_node virtual machines. empty by default
    gateway                 = "192.168.100.1"
    # resource pool to deploy kubernetes_node virtual machines to. empty by default
    resource_pool           = ""
    # number of vcpu assigned to kubernetes_node virtual machines. default is 2
    num_cpus = 2
    # amount of memory assigned to kubernetes_node virtual machines. default is 4096 (4GB)
    memory   = 4096
  }
}
[root@terraform terraform-vsphere-kubernetes]#
{{< /highlight >}}

### 3. Update 'terraform.tfvar' and set the values for the virtual_machine_kubernetes_nodes variables

{{< highlight bash >}}
[root@terraform terraform-vsphere-kubernetes]# cat terraform.tfvars
vsphere_connection = {
  vsphere_user = "terraform_user@vsphere.local"
  vsphere_password = "SuperSecretPassword"
  vsphere_server = "192.168.100.50"
}
virtual_machine_template = {
  name = "standorsett-centos-7"
  datacenter = "datacenter"
  connection_password = "vagrant"
}
virtual_machine_kubernetes_controller = {
  name = "kubernetes-controller"
  datastore = "datastore-1"
  network = "kubernetes-portgroup"
  ip_address = "192.168.100.100"
  netmask = "24"
  dns_server = "192.168.100.1"
  gateway = "192.168.100.1"
  resource_pool = "cluster/Resources"
}
virtual_machine_kubernetes_node = {
  count = 1
  prefix = "kubernetes-node"
  datastore = "datastore-1"
  network = "kubernetes-postgroup"
  ip_address_network = "192.168.100.0/24"
  starting_hostnum = "101"
  dns_server = "192.168.100.1"
  gateway = "192.168.100.1"
  resource_pool = "cluster/Resources"
}
[root@terraform terraform-vsphere-kubernetes]#
{{< /highlight >}}

I think it is a good idea to set the count to `'1'` while testing, to minimize the deployment time. Once you are confident one kubernetes node is deploying properly, you can then increase the count to the actual number you are wanting to have deployed. 

### 4. Add 'kubernetes_nodes.tf' that will describe how to kubernetes nodes will be deployed and configured.

Much of the terraform code for deploying the kubernetes nodes can be copied from the kubernetes controller code.

{{< highlight bash >}}
[root@terraform terraform-vsphere-kubernetes]# cat kubernetes_nodes.tf
data "vsphere_datastore" "node_datastore" {
  name          = "${var.virtual_machine_kubernetes_node.["datastore"]}"
  datacenter_id = "${data.vsphere_datacenter.template_datacenter.id}"
}

data "vsphere_resource_pool" "node_resource_pool" {
  name          = "${var.virtual_machine_kubernetes_node.["resource_pool"]}"
  datacenter_id = "${data.vsphere_datacenter.template_datacenter.id}"
}

data "vsphere_network" "node_network" {
  name          = "${var.virtual_machine_kubernetes_node.["network"]}"
  datacenter_id = "${data.vsphere_datacenter.template_datacenter.id}"
}

resource "vsphere_virtual_machine" "kubernetes_nodes" {
  count            = "${var.virtual_machine_kubernetes_node.["count"]}"
  name             = "${format("${var.virtual_machine_kubernetes_node.["prefix"]}-%03d", count.index + 1)}"
  resource_pool_id = "${data.vsphere_resource_pool.vm_resource_pool.id}"
  datastore_id     = "${data.vsphere_datastore.vm_datastore.id}"

  num_cpus = "${var.virtual_machine_kubernetes_node.["num_cpus"]}"
  memory   = "${var.virtual_machine_kubernetes_node.["memory"]}"
  guest_id = "${data.vsphere_virtual_machine.template.guest_id}"

  scsi_type = "${data.vsphere_virtual_machine.template.scsi_type}"

  network_interface {
    network_id   = "${data.vsphere_network.vm_network.id}"
    adapter_type = "${data.vsphere_virtual_machine.template.network_interface_types[0]}"
  }

  disk {
    label = "disk0"
    size = "${data.vsphere_virtual_machine.template.disks.0.size}"
  }

  clone {
    template_uuid = "${data.vsphere_virtual_machine.template.id}"

    customize {
      linux_options {
        host_name = "${format("${var.virtual_machine_kubernetes_node.["prefix"]}-%03d", count.index + 1)}"
        domain    = "kubernetes.local"
      }

      dns_server_list = ["${var.virtual_machine_kubernetes_node.["dns_server"]}"]
      dns_suffix_list = ["kubernetes.local"]

      network_interface {
        ipv4_address = "${cidrhost( var.virtual_machine_kubernetes_node.["ip_address_network"], var.virtual_machine_kubernetes_node.["starting_hostnum"]+count.index )}"
        ipv4_netmask = "${element( split("/", var.virtual_machine_kubernetes_node.["ip_address_network"]), 1)}"
      }

      ipv4_gateway = "${var.virtual_machine_kubernetes_node.["gateway"]}"
    }
  }

  provisioner "file" {
    source      = "${var.virtual_machine_kubernetes_controller.["public_key"]}"
    destination = "/tmp/authorized_keys"

    connection {
      type        = "${var.virtual_machine_template.["connection_type"]}"
      user        = "${var.virtual_machine_template.["connection_user"]}"
      password    = "${var.virtual_machine_template.["connection_password"]}"
    }
  }

  provisioner "remote-exec" {
    inline = [
      "mkdir -p /root/.ssh/",
      "chmod 700 /root/.ssh",
      "mv /tmp/authorized_keys /root/.ssh/authorized_keys",
      "chmod 600 /root/.ssh/authorized_keys",
      "sed -i 's/#PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config",
      "service sshd restart"
    ]
    connection {
      type          = "${var.virtual_machine_template.["connection_type"]}"
      user          = "${var.virtual_machine_template.["connection_user"]}"
      password      = "${var.virtual_machine_template.["connection_password"]}"
    }
  }

  provisioner "file" {
    source      = "./scripts/"
    destination = "/tmp/"

    connection {
      type        = "${var.virtual_machine_template.["connection_type"]}"
      user        = "${var.virtual_machine_template.["connection_user"]}"
      private_key = "${file("${var.virtual_machine_kubernetes_controller.["private_key"]}")}"
    }
  }

  provisioner "remote-exec" {
    inline = [
      "chmod +x /tmp/*sh",
      "sudo /tmp/system_setup.sh",
      "sudo /tmp/install_docker.sh",
      "sudo /tmp/install_kubernetes_packages.sh",
    ]
    connection {
      type          = "${var.virtual_machine_template.["connection_type"]}"
      user          = "${var.virtual_machine_template.["connection_user"]}"
      private_key   = "${file("${var.virtual_machine_kubernetes_controller.["private_key"]}")}"
    }
  }

}

resource "null_resource" "kubeadm_join" {
  count            = "${var.virtual_machine_kubernetes_node.["count"]}"
  provisioner "remote-exec" {
    inline = [
       "kubeadm join --token ${data.external.kubeadm-init-info.result.token} ${vsphere_virtual_machine.kubernetes_controller.0.default_ip_address}:6443 --discovery-token-ca-cert-hash sha256:${data.external.kubeadm-init-info.result.certhash}",
    ]
    connection {
      type          = "${var.virtual_machine_template.["connection_type"]}"
      user          = "${var.virtual_machine_template.["connection_user"]}"
      private_key   = "${file("${var.virtual_machine_kubernetes_controller.["private_key"]}")}"
      host          = "${element(vsphere_virtual_machine.kubernetes_nodes.*.default_ip_address, count.index)}"
    }
  }
}
[root@terraform terraform-vsphere-kubernetes]#
{{< /highlight >}}

There are differences in the `'vsphere_virtual_machine.kubernetes_nodes'` resource that were not in the `'vsphere_virtual_machine.kubernetes_controller'` resource: 

* count = "${var.virtual_machine_kubernetes_node.["count"]}" - the number of kubernetes node virtual machines we want to have deployed. The `'count.index'` starts with 0 and will repeat the number of time specified by `'count.'`
* name = "${format("${var.virtual_machine_kubernetes_node.["prefix"]}-%03d", count.index + 1)}" - the format command sets the name of the virtual machine by starting with the `'prefix'` followed by `'-'` and finally the `'count.index'` specified as a 3 digit integer.
* hostname = "${format("${var.virtual_machine_kubernetes_node.["prefix"]}-%03d", count.index + 1)}" - the same as `'name'`
* ipv4_address = "${cidrhost( var.virtual_machine_kubernetes_node.["ip_address_network"], var.virtual_machine_kubernetes_node.["starting_hostnum"]+count.index )}" - `'cidrhost'` takes an IP address range in CIDR notation and creates an IP address with the given host number.
* ipv4_netmask = "${element( split("/", var.virtual_machine_kubernetes_node.["ip_address_network"]), 1)}" - `'split'` will create  list of strings, splitting `'var.virtual_machine_kubernetes_node.["ip_address_network"]'` on the `'/'` character. `'element'` will take an element from a list, in this case taking the `'1'` element (which is the second since numbering starts with `'0'`.).

The other difference from the `'vsphere_virtual_machine.kubernetes_controller'` resource is the `'null_resource.kubeadm_join'` resource:

{{< highlight bash >}}
resource "null_resource" "kubeadm_join" {
  count            = "${var.virtual_machine_kubernetes_node.["count"]}"
  provisioner "remote-exec" {
    inline = [
       "kubeadm join --token ${data.external.kubeadm-init-info.result.token} ${vsphere_virtual_machine.kubernetes_controller.0.default_ip_address}:6443 --discovery-token-ca-cert-hash sha256:${data.external.kubeadm-init-info.result.certhash}",
    ]
    connection {
      type          = "${var.virtual_machine_template.["connection_type"]}"
      user          = "${var.virtual_machine_template.["connection_user"]}"
      private_key   = "${file("${var.virtual_machine_kubernetes_controller.["private_key"]}")}"
      host          = "${element(vsphere_virtual_machine.kubernetes_nodes.*.default_ip_address, count.index)}"
    }
  }
}
{{< /highlight >}}

In the `'null_resource.kubeadm_join'` the IP address of the current kubernetes_node is determined by using `'element'` to select the `'count.index'` element from the list returned by `'vsphere_virtual_machine.kubernetes_nodes.*.default_ip_address'`. Als the `'remote-exec'` command is using the `'token'` and `'certhash'` properties returned from the external data resource that was created in the previous post.

### 5. Update the `'output.tf'` file to display the IP addresses of the node virtual machines that were deployed.

{{< highlight bash >}}
[root@terraform terraform-vsphere-kubernetes]# cat output.tf
output "controller_ip" {
   value = "${vsphere_virtual_machine.kubernetes_controller.*.default_ip_address}"
}
output "node_ips" {
   value = "${vsphere_virtual_machine.kubernetes_nodes.*.default_ip_address}"
}
output "controller_vm-moref" {
   value = "${vsphere_virtual_machine.kubernetes_controller.moid}"
}
output "node_vm-morefs" {
   value = "${vsphere_virtual_machine.kubernetes_nodes.*.moid}"
}
output "kubeadm-init-info" {
   value = "${data.external.kubeadm-init-info.result}"
}

[root@terraform terraform-vsphere-kubernetes]#
{{< /highlight >}}

### 6. Run `terraform init` to initialize terraform and downloaded the provisioners.

{{< highlight bash >}}
[root@terraform terraform-vsphere-kubernetes]# terraform init

Initializing provider plugins...

The following providers do not have any version constraints in configuration,
so the latest version was installed.

To prevent automatic upgrades to new major versions that may contain breaking
changes, it is recommended to add version = "..." constraints to the
corresponding provider blocks in configuration, with the constraint strings
suggested below.

* provider.external: version = "~> 1.0"
* provider.null: version = "~> 1.0"
* provider.vsphere: version = "~> 1.9"

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
[root@terraform terraform-vsphere-kubernetes]#
{{< /highlight >}}

### 7. Run `terraform plan` to have terraform show what will get created.

{{< highlight bash >}}
[root@terraform terraform-vsphere-kubernetes]# terraform plan
Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.

data.vsphere_datacenter.template_datacenter: Refreshing state...
data.vsphere_datastore.vm_datastore: Refreshing state...
data.vsphere_resource_pool.node_resource_pool: Refreshing state...
data.vsphere_network.vm_network: Refreshing state...
data.vsphere_datastore.node_datastore: Refreshing state...
data.vsphere_virtual_machine.template: Refreshing state...
data.vsphere_network.node_network: Refreshing state...
data.vsphere_resource_pool.vm_resource_pool: Refreshing state...

------------------------------------------------------------------------

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create
 <= read (data resources)

Terraform will perform the following actions:

 <= data.external.kubeadm-init-info
      id:                                                   <computed>
      program.#:                                            "2"
      program.0:                                            "/usr/bin/bash"
      program.1:                                            "/root/terraform-vsphere-kubernetes/scripts/kubeadm_init_info.sh"
      query.%:                                              <computed>
      result.%:                                             <computed>

  + null_resource.generate-sshkey
      id:                                                   <computed>

  + null_resource.kubeadm_join
      id:                                                   <computed>

  + null_resource.ssh-keygen-delete
      id:                                                   <computed>

  + vsphere_virtual_machine.kubernetes_controller
      id:                                                   <computed>
      boot_retry_delay:                                     "10000"
      change_version:                                       <computed>
      clone.#:                                              "1"
      clone.0.customize.#:                                  "1"
      clone.0.customize.0.dns_server_list.#:                "1"
      clone.0.customize.0.dns_server_list.0:                "192.168.100.1"
      clone.0.customize.0.dns_suffix_list.#:                "1"
      clone.0.customize.0.dns_suffix_list.0:                "kubernetes.local"
      clone.0.customize.0.ipv4_gateway:                     "192.168.100.1"
      clone.0.customize.0.linux_options.#:                  "1"
      clone.0.customize.0.linux_options.0.domain:           "kubernetes.local"
      clone.0.customize.0.linux_options.0.host_name:        "kubernetes-controller"
      clone.0.customize.0.linux_options.0.hw_clock_utc:     "true"
      clone.0.customize.0.network_interface.#:              "1"
      clone.0.customize.0.network_interface.0.ipv4_address: "192.168.100.100"
      clone.0.customize.0.network_interface.0.ipv4_netmask: "24"
      clone.0.customize.0.timeout:                          "10"
      clone.0.template_uuid:                                "4221e272-e449-97e0-39f4-ab580bebc473"
      clone.0.timeout:                                      "30"
      cpu_limit:                                            "-1"
      cpu_share_count:                                      <computed>
      cpu_share_level:                                      "normal"
      datastore_id:                                         "datastore-15"
      default_ip_address:                                   <computed>
      disk.#:                                               "1"
      disk.0.attach:                                        "false"
      disk.0.datastore_id:                                  "<computed>"
      disk.0.device_address:                                <computed>
      disk.0.disk_mode:                                     "persistent"
      disk.0.disk_sharing:                                  "sharingNone"
      disk.0.eagerly_scrub:                                 "false"
      disk.0.io_limit:                                      "-1"
      disk.0.io_reservation:                                "0"
      disk.0.io_share_count:                                "0"
      disk.0.io_share_level:                                "normal"
      disk.0.keep_on_remove:                                "false"
      disk.0.key:                                           "0"
      disk.0.label:                                         "disk0"
      disk.0.path:                                          <computed>
      disk.0.size:                                          "8"
      disk.0.thin_provisioned:                              "true"
      disk.0.unit_number:                                   "0"
      disk.0.uuid:                                          <computed>
      disk.0.write_through:                                 "false"
      ept_rvi_mode:                                         "automatic"
      firmware:                                             "bios"
      force_power_off:                                      "true"
      guest_id:                                             "rhel7_64Guest"
      guest_ip_addresses.#:                                 <computed>
      host_system_id:                                       <computed>
      hv_mode:                                              "hvAuto"
      imported:                                             <computed>
      latency_sensitivity:                                  "normal"
      memory:                                               "4096"
      memory_limit:                                         "-1"
      memory_share_count:                                   <computed>
      memory_share_level:                                   "normal"
      migrate_wait_timeout:                                 "30"
      moid:                                                 <computed>
      name:                                                 "kubernetes-controller"
      network_interface.#:                                  "1"
      network_interface.0.adapter_type:                     "vmxnet3"
      network_interface.0.bandwidth_limit:                  "-1"
      network_interface.0.bandwidth_reservation:            "0"
      network_interface.0.bandwidth_share_count:            <computed>
      network_interface.0.bandwidth_share_level:            "normal"
      network_interface.0.device_address:                   <computed>
      network_interface.0.key:                              <computed>
      network_interface.0.mac_address:                      <computed>
      network_interface.0.network_id:                       "network-24"
      num_cores_per_socket:                                 "1"
      num_cpus:                                             "2"
      reboot_required:                                      <computed>
      resource_pool_id:                                     "resgroup-8"
      run_tools_scripts_after_power_on:                     "true"
      run_tools_scripts_after_resume:                       "true"
      run_tools_scripts_before_guest_shutdown:              "true"
      run_tools_scripts_before_guest_standby:               "true"
      scsi_bus_sharing:                                     "noSharing"
      scsi_controller_count:                                "1"
      scsi_type:                                            "lsilogic"
      shutdown_wait_timeout:                                "3"
      swap_placement_policy:                                "inherit"
      uuid:                                                 <computed>
      vmware_tools_status:                                  <computed>
      vmx_path:                                             <computed>
      wait_for_guest_net_routable:                          "true"
      wait_for_guest_net_timeout:                           "5"

  + vsphere_virtual_machine.kubernetes_nodes
      id:                                                   <computed>
      boot_retry_delay:                                     "10000"
      change_version:                                       <computed>
      clone.#:                                              "1"
      clone.0.customize.#:                                  "1"
      clone.0.customize.0.dns_server_list.#:                "1"
      clone.0.customize.0.dns_server_list.0:                "192.168.100.1"
      clone.0.customize.0.dns_suffix_list.#:                "1"
      clone.0.customize.0.dns_suffix_list.0:                "kubernetes.local"
      clone.0.customize.0.ipv4_gateway:                     "192.168.100.1"
      clone.0.customize.0.linux_options.#:                  "1"
      clone.0.customize.0.linux_options.0.domain:           "kubernetes.local"
      clone.0.customize.0.linux_options.0.host_name:        "kubernetes-node-001"
      clone.0.customize.0.linux_options.0.hw_clock_utc:     "true"
      clone.0.customize.0.network_interface.#:              "1"
      clone.0.customize.0.network_interface.0.ipv4_address: "192.168.100.101"
      clone.0.customize.0.network_interface.0.ipv4_netmask: "24"
      clone.0.customize.0.timeout:                          "10"
      clone.0.template_uuid:                                "4221e272-e449-97e0-39f4-ab580bebc473"
      clone.0.timeout:                                      "30"
      cpu_limit:                                            "-1"
      cpu_share_count:                                      <computed>
      cpu_share_level:                                      "normal"
      datastore_id:                                         "datastore-15"
      default_ip_address:                                   <computed>
      disk.#:                                               "1"
      disk.0.attach:                                        "false"
      disk.0.datastore_id:                                  "<computed>"
      disk.0.device_address:                                <computed>
      disk.0.disk_mode:                                     "persistent"
      disk.0.disk_sharing:                                  "sharingNone"
      disk.0.eagerly_scrub:                                 "false"
      disk.0.io_limit:                                      "-1"
      disk.0.io_reservation:                                "0"
      disk.0.io_share_count:                                "0"
      disk.0.io_share_level:                                "normal"
      disk.0.keep_on_remove:                                "false"
      disk.0.key:                                           "0"
      disk.0.label:                                         "disk0"
      disk.0.path:                                          <computed>
      disk.0.size:                                          "8"
      disk.0.thin_provisioned:                              "true"
      disk.0.unit_number:                                   "0"
      disk.0.uuid:                                          <computed>
      disk.0.write_through:                                 "false"
      ept_rvi_mode:                                         "automatic"
      firmware:                                             "bios"
      force_power_off:                                      "true"
      guest_id:                                             "rhel7_64Guest"
      guest_ip_addresses.#:                                 <computed>
      host_system_id:                                       <computed>
      hv_mode:                                              "hvAuto"
      imported:                                             <computed>
      latency_sensitivity:                                  "normal"
      memory:                                               "4096"
      memory_limit:                                         "-1"
      memory_share_count:                                   <computed>
      memory_share_level:                                   "normal"
      migrate_wait_timeout:                                 "30"
      moid:                                                 <computed>
      name:                                                 "kubernetes-node-001"
      network_interface.#:                                  "1"
      network_interface.0.adapter_type:                     "vmxnet3"
      network_interface.0.bandwidth_limit:                  "-1"
      network_interface.0.bandwidth_reservation:            "0"
      network_interface.0.bandwidth_share_count:            <computed>
      network_interface.0.bandwidth_share_level:            "normal"
      network_interface.0.device_address:                   <computed>
      network_interface.0.key:                              <computed>
      network_interface.0.mac_address:                      <computed>
      network_interface.0.network_id:                       "network-24"
      num_cores_per_socket:                                 "1"
      num_cpus:                                             "2"
      reboot_required:                                      <computed>
      resource_pool_id:                                     "resgroup-8"
      run_tools_scripts_after_power_on:                     "true"
      run_tools_scripts_after_resume:                       "true"
      run_tools_scripts_before_guest_shutdown:              "true"
      run_tools_scripts_before_guest_standby:               "true"
      scsi_bus_sharing:                                     "noSharing"
      scsi_controller_count:                                "1"
      scsi_type:                                            "lsilogic"
      shutdown_wait_timeout:                                "3"
      swap_placement_policy:                                "inherit"
      uuid:                                                 <computed>
      vmware_tools_status:                                  <computed>
      vmx_path:                                             <computed>
      wait_for_guest_net_routable:                          "true"
      wait_for_guest_net_timeout:                           "5"


Plan: 5 to add, 0 to change, 0 to destroy.

------------------------------------------------------------------------

Note: You didn't specify an "-out" parameter to save this plan, so Terraform
can't guarantee that exactly these actions will be performed if
"terraform apply" is subsequently run.

[root@terraform terraform-vsphere-kubernetes]#
{{< /highlight >}}

Now is a good time to double check the Interpolated values that were set for name, hostname, ip_address and netmask, to make sure they are what we were expecting: 
{{< highlight bash >}}
...
vsphere_virtual_machine.kubernetes_nodes
...
      clone.0.customize.0.linux_options.0.host_name:        "kubernetes-node-001"
      clone.0.customize.0.network_interface.0.ipv4_address: "192.168.100.101"
      clone.0.customize.0.network_interface.0.ipv4_netmask: "24"
      name:                                                 "kubernetes-node-001"
...
{{< /highlight >}}

### 8. Run `terraform apply` to create the described resources.

Running `terraform apply --auto-approve` will have terraform create the resources described in `ssh-key.tf`,`kubernetes_controller.tf` and `kubernetes_nodes.tf`. In order to spare you a whole lot of logs, we will jump to the end of the output:

{{< highlight bash >}}

t@terraform terraform-vsphere-kubernetes]# terraform apply --auto-approve
data.vsphere_datacenter.template_datacenter: Refreshing state...
data.vsphere_datastore.node_datastore: Refreshing state...
data.vsphere_network.vm_network: Refreshing state...
data.vsphere_resource_pool.vm_resource_pool: Refreshing state...
data.vsphere_datastore.vm_datastore: Refreshing state...
data.vsphere_virtual_machine.template: Refreshing state...
data.vsphere_network.node_network: Refreshing state...
data.vsphere_resource_pool.node_resource_pool: Refreshing state...
null_resource.ssh-keygen-delete: Creating...
null_resource.generate-sshkey: Creating...
null_resource.ssh-keygen-delete: Provisioning with 'local-exec'...
null_resource.generate-sshkey: Provisioning with 'local-exec'...
null_resource.ssh-keygen-delete (local-exec): Executing: ["/bin/sh" "-c" "ssh-keygen -R 192.168.100.100"]
null_resource.generate-sshkey (local-exec): Executing: ["/bin/sh" "-c" "yes y | ssh-keygen -b 4096 -t rsa -C 'terraform-vsphere-kubernetes' -N '' -f /root/.ssh/id_rsa-terraform-vsphere-kubernetes"]
null_resource.ssh-keygen-delete (local-exec): Host 192.168.100.100 not found in /root/.ssh/known_hosts
null_resource.ssh-keygen-delete: Creation complete after 0s (ID: 3375410565424976846)
vsphere_virtual_machine.kubernetes_controller: Creating...
...

...
null_resource.kubeadm_join: Provisioning with 'remote-exec'...
null_resource.kubeadm_join (remote-exec): Connecting to remote host via SSH...
null_resource.kubeadm_join (remote-exec):   Host: 192.168.100.101
null_resource.kubeadm_join (remote-exec):   User: root
null_resource.kubeadm_join (remote-exec):   Password: false
null_resource.kubeadm_join (remote-exec):   Private key: true
null_resource.kubeadm_join (remote-exec):   SSH Agent: false
null_resource.kubeadm_join (remote-exec):   Checking Host Key: false
null_resource.kubeadm_join (remote-exec): Connected!
null_resource.kubeadm_join (remote-exec): [preflight] Running pre-flight checks
null_resource.kubeadm_join (remote-exec): [discovery] Trying to connect to API Server "192.168.100.100:6443"
null_resource.kubeadm_join (remote-exec): [discovery] Created cluster-info discovery client, requesting info from "https://192.168.100.100:6443"
null_resource.kubeadm_join (remote-exec): [discovery] Failed to connect to API Server "192.168.100.100:6443": token id "lgfyu1" is invalid for this cluster or it has expired. Use "kubeadm token create" on the master node to creating a new valid token
null_resource.kubeadm_join (remote-exec): [discovery] Trying to connect to API Server "192.168.100.100:6443"
null_resource.kubeadm_join (remote-exec): [discovery] Created cluster-info discovery client, requesting info from "https://192.168.100.100:6443"
null_resource.kubeadm_join (remote-exec): [discovery] Failed to connect to API Server "192.168.100.100:6443": token id "lgfyu1" is invalid for this cluster or it has expired. Use "kubeadm token create" on the master node to creating a new valid token
null_resource.kubeadm_join: Still creating... (10s elapsed)
null_resource.kubeadm_join (remote-exec): [discovery] Trying to connect to API Server "192.168.100.100:6443"
null_resource.kubeadm_join (remote-exec): [discovery] Created cluster-info discovery client, requesting info from "https://192.168.100.100:6443"
null_resource.kubeadm_join (remote-exec): [discovery] Failed to connect to API Server "192.168.100.100:6443": token id "lgfyu1" is invalid for this cluster or it has expired. Use "kubeadm token create" on the master node to creating a new valid token
null_resource.kubeadm_join (remote-exec): [discovery] Trying to connect to API Server "192.168.100.100:6443"
null_resource.kubeadm_join (remote-exec): [discovery] Created cluster-info discovery client, requesting info from "https://192.168.100.100:6443"
null_resource.kubeadm_join (remote-exec): [discovery] Requesting info from "https://192.168.100.100:6443" again to validate TLS against the pinned public key
null_resource.kubeadm_join (remote-exec): [discovery] Cluster info signature and contents are valid and TLS certificate validates against pinned roots, will use API Server "192.168.100.100:6443"
null_resource.kubeadm_join (remote-exec): [discovery] Successfully established connection with API Server "192.168.100.100:6443"
null_resource.kubeadm_join (remote-exec): [join] Reading configuration from the cluster...
null_resource.kubeadm_join (remote-exec): [join] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
null_resource.kubeadm_join (remote-exec): [kubelet] Downloading configuration for the kubelet from the "kubelet-config-1.13" ConfigMap in the kube-system namespace
null_resource.kubeadm_join (remote-exec): [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
null_resource.kubeadm_join (remote-exec): [kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
null_resource.kubeadm_join (remote-exec): [kubelet-start] Activating the kubelet service
null_resource.kubeadm_join (remote-exec): [tlsbootstrap] Waiting for the kubelet to perform the TLS Bootstrap...
null_resource.kubeadm_join (remote-exec): [patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "kubernetes-node-001" as an annotation

null_resource.kubeadm_join (remote-exec): This node has joined the cluster:
null_resource.kubeadm_join (remote-exec): * Certificate signing request was sent to apiserver and a response was received.
null_resource.kubeadm_join (remote-exec): * The Kubelet was informed of the new secure connection details.

null_resource.kubeadm_join (remote-exec): Run 'kubectl get nodes' on the master to see this node join the cluster.

null_resource.kubeadm_join: Creation complete after 17s (ID: 4377642417259073555)

Apply complete! Resources: 5 added, 0 changed, 0 destroyed.

Outputs:

controller_ip = [
    192.168.100.100
]
controller_vm-moref = vm-1148
kubeadm-init-info = {
  certhash = 32f55cd4dfee35941ba3dd5854e40e91b74cc7c39c46dd683b5f47232dde842d
  token = lgfyu1.6mihn3jrkyxusqyc
}
node_ips = [
    192.168.100.101
]
node_vm-morefs = [
    vm-1149
]
[root@terraform terraform-vsphere-kubernetes]#
{{< /highlight >}}

The bottom of the `terraform appy` now displays the `'node_ips'` and `'node_vm-morefs'`. If you look up in the output to the `null_resource.kubeadm_join (remote-exec):` lines, you will notice that  `'This node has joined the cluster'` 

### 9. Validate that the node has successfully joined to cluster by running `'kubectl get nodes`' on the kubernetes controller virtual machine:

{{< highlight bash >}}
[root@terraform terraform-vsphere-kubernetes]# ssh root@192.168.100.100 -o 'IdentityFile ~/.ssh/id_rsa-terraform-vsphere-kubernetes' -o 'StrictHostKeyChecking no' -o 'UserKnownHostsFile /dev/null' 'kubectl get nodes'
Warning: Permanently added '192.168.100.100' (ECDSA) to the list of known hosts.
NAME                    STATUS   ROLES    AGE   VERSION
kubernetes-controller   Ready    master   12m   v1.13.1
kubernetes-node-001     Ready    <none>   12m   v1.13.1
[root@terraform terraform-vsphere-kubernetes]#
{{< /highlight >}}

### 10. Run `terraform destroy` to delete the resources that were previously created.

Once you are ready to destroy the virtual machine terraform created, you can simply run `terraform destroy --force` to have terraform destroy it:

{{< highlight bash >}}
[root@terraform terraform-vsphere-kubernetes]# terraform destroy --force
null_resource.generate-sshkey: Refreshing state... (ID: 1602706465654826282)
null_resource.ssh-keygen-delete: Refreshing state... (ID: 3375410565424976846)
data.vsphere_datacenter.template_datacenter: Refreshing state...
data.vsphere_virtual_machine.template: Refreshing state...
data.vsphere_network.node_network: Refreshing state...
data.vsphere_resource_pool.vm_resource_pool: Refreshing state...
data.vsphere_datastore.node_datastore: Refreshing state...
data.vsphere_network.vm_network: Refreshing state...
data.vsphere_datastore.vm_datastore: Refreshing state...
data.vsphere_resource_pool.node_resource_pool: Refreshing state...
vsphere_virtual_machine.kubernetes_controller: Refreshing state... (ID: 42212384-7bd3-57b6-0ead-bf99e4a223e0)
vsphere_virtual_machine.kubernetes_nodes: Refreshing state... (ID: 4221d0bf-b9a8-6ddc-56ab-ee1f1916ec06)
data.external.kubeadm-init-info: Refreshing state...
null_resource.kubeadm_join: Refreshing state... (ID: 4377642417259073555)
null_resource.kubeadm_join: Destroying... (ID: 4377642417259073555)
null_resource.generate-sshkey: Destroying... (ID: 1602706465654826282)
null_resource.ssh-keygen-delete: Destroying... (ID: 3375410565424976846)
null_resource.generate-sshkey: Destruction complete after 0s
null_resource.ssh-keygen-delete: Destruction complete after 0s
null_resource.kubeadm_join: Destruction complete after 0s
vsphere_virtual_machine.kubernetes_nodes: Destroying... (ID: 4221d0bf-b9a8-6ddc-56ab-ee1f1916ec06)
vsphere_virtual_machine.kubernetes_controller: Destroying... (ID: 42212384-7bd3-57b6-0ead-bf99e4a223e0)
vsphere_virtual_machine.kubernetes_controller: Still destroying... (ID: 42212384-7bd3-57b6-0ead-bf99e4a223e0, 10s elapsed)
vsphere_virtual_machine.kubernetes_nodes: Still destroying... (ID: 4221d0bf-b9a8-6ddc-56ab-ee1f1916ec06, 10s elapsed)
vsphere_virtual_machine.kubernetes_nodes: Still destroying... (ID: 4221d0bf-b9a8-6ddc-56ab-ee1f1916ec06, 20s elapsed)
vsphere_virtual_machine.kubernetes_controller: Still destroying... (ID: 42212384-7bd3-57b6-0ead-bf99e4a223e0, 20s elapsed)
vsphere_virtual_machine.kubernetes_controller: Destruction complete after 28s
vsphere_virtual_machine.kubernetes_nodes: Still destroying... (ID: 4221d0bf-b9a8-6ddc-56ab-ee1f1916ec06, 30s elapsed)
vsphere_virtual_machine.kubernetes_nodes: Destruction complete after 39s

Destroy complete! Resources: 5 destroyed.
[root@terraform terraform-vsphere-kubernetes]#
{{< /highlight >}}

### 11. Update `'vim terraform.tfvars'` to specify a kubernetes_nodes count of 3 and re-run `'terraform apply`'

Modify `'virtual_machine_kubernetes_node["count"]'` to be "3":

{{< highlight bash >}}
[root@terraform terraform-vsphere-kubernetes]# tail -n 11 terraform.tfvars
virtual_machine_kubernetes_node = {
  count = 3
  prefix = "kubernetes-node"
  datastore = "datastore-1"
  network = "VM Network"
  ip_address_network = "192.168.100.0/24"
  starting_hostnum = "101"
  dns_server = "192.168.100.1"
  gateway = "192.168.100.1"
  resource_pool = "cluster/Resources"
}
[root@terraform terraform-vsphere-kubernetes]#
{{< /highlight >}}

Re-run `'terraform apply --auto-approve'` to deploy the desribed resources:

{{< highlight bash >}}
[root@terraform terraform-vsphere-kubernetes]# terraform apply --auto-approve
data.vsphere_datacenter.template_datacenter: Refreshing state...
data.vsphere_resource_pool.node_resource_pool: Refreshing state...
data.vsphere_resource_pool.vm_resource_pool: Refreshing state...
data.vsphere_virtual_machine.template: Refreshing state...
data.vsphere_datastore.node_datastore: Refreshing state...
data.vsphere_datastore.vm_datastore: Refreshing state...
data.vsphere_network.node_network: Refreshing state...
data.vsphere_network.vm_network: Refreshing state...
null_resource.generate-sshkey: Creating...
null_resource.ssh-keygen-delete: Creating...
null_resource.generate-sshkey: Provisioning with 'local-exec'...
null_resource.ssh-keygen-delete: Provisioning with 'local-exec'...
null_resource.ssh-keygen-delete (local-exec): Executing: ["/bin/sh" "-c" "ssh-keygen -R 192.168.100.100"]
null_resource.generate-sshkey (local-exec): Executing: ["/bin/sh" "-c" "yes y | ssh-keygen -b 4096 -t rsa -C 'terraform-vsphere-kubernetes' -N '' -f /root/.ssh/id_rsa-terraform-vsphere-kubernetes"]
null_resource.ssh-keygen-delete (local-exec): Host 192.168.100.100 not found in /root/.ssh/known_hosts
null_resource.ssh-keygen-delete: Creation complete after 0s (ID: 6566019135999696920)

...

...
Apply complete! Resources: 9 added, 0 changed, 0 destroyed.

Outputs:

controller_ip = [
    192.168.100.100
]
controller_vm-moref = vm-1153
kubeadm-init-info = {
  certhash = f625b490da283a3269b0ac106ba23da3f5cb8f23e7f1c926cdd8e0f159ee467a
  token = 1skot6.v55xizpvmobx0erk
}
node_ips = [
    192.168.100.101,
    192.168.100.102,
    192.168.100.103
]
node_vm-mores = [
    vm-1150,
    vm-1151,
    vm-1152
]
[root@terraform terraform-vsphere-kubernetes]#
{{< /highlight >}}

Finally we should again check `'kubectl get nodes'` to validate the kubernetes nodes successfully joined the kubernetes cluster:

{{< highlight bash >}}
[root@terraform terraform-vsphere-kubernetes]# ssh root@192.168.100.100 -o 'IdentityFile ~/.ssh/id_rsa-terraform-vsphere-kubernetes' -o 'StrictHostKeyChecking no' -o 'UserKnownHostsFile /dev/null' 'kubectl get nodes'
Warning: Permanently added '192.168.100.100' (ECDSA) to the list of known hosts.
NAME                    STATUS   ROLES    AGE     VERSION
kubernetes-controller   Ready    master   2m20s   v1.13.1
kubernetes-node-001     Ready    <none>   119s    v1.13.1
kubernetes-node-002     Ready    <none>   118s    v1.13.1
kubernetes-node-003     Ready    <none>   118s    v1.13.1
[root@terraform terraform-vsphere-kubernetes]#
{{< /highlight >}}

#### I hope this post has been useful in demonstrating how to use `'count'` to create multiple instances of the same resource and interpolation to transform variables and lists. The files that were using in this post can be found in <a href="https://github.com/sdorsett/terraform-vsphere-kubernetes/tree/2018-12-30-post">this github repository</a>.

---

#### Please provide any feedback or suggestions to my twitter account located on the about page.
