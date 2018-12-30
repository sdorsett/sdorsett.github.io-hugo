+++
title = "Using an external data source with terraform"
description = ""
tags = [
    "terraform",
    "vsphere",
    "kubernetes",
]
date = "2018-12-28"
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
This is the fifth in a series of posts that will walk you through using terraform to deploy and configure virtual machines on vsphere. In this post you will get introduced to using an external data source with terraform. 

---

### 1. Clone the `https://github.com/sdorsett/terraform-vsphere-kubernetes` repository and switch to the '2018-12-26-post' branch..

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

Checkout the `'2018-12-26-post'` branch which is where we left off at the end of the last post:

{{< highlight bash >}}
[root@terraform terraform-vsphere-kubernetes]# git checkout origin/2018-12-26-post
Note: checking out 'origin/2018-12-26-post'.

You are in 'detached HEAD' state. You can look around, make experimental
changes and commit them, and you can discard any commits you make in this
state without impacting any branches by performing another checkout.

If you want to create a new branch to retain commits you create, you may
do so (now or later) by using -b with the checkout command again. Example:

  git checkout -b new_branch_name

HEAD is now at d072955... Initial commit
[root@terraform terraform-vsphere-kubernetes]#
{{< /highlight >}}

### 2. Update `ssh-key.tf` to add a `local-exec` block for removing any old host entries from the `known_hosts` file 

In the last post we used ssh to connect to the deployed virtual machine and ran 'kubeadm get nodes' to check the status of the kubernetes cluster. When we did this an entry was created in the 'known_hosts' file with the ssh thumbprint of the deployed server. This ssh thumbprint will be incorrect after we destroyed and re-deployed the virtual machine using terraform.

In order to prevent errors when using ssh to connect after destroying and re-deploying the virtual machine, it would be good to add a second `local-exec` provisioner to 'ssh-key.tf' to clean up any old 'known_hosts' entries using the `'ssh-keygen -R'` command:

{{< highlight bash >}}
[root@terraform terraform-vsphere-kubernetes]# cat ssh-key.tf
resource "null_resource" "generate-sshkey" {
    provisioner "local-exec" {
        command = "yes y | ssh-keygen -b 4096 -t rsa -C 'terraform-vsphere-kubernetes' -N '' -f ${var.virtual_machine_kubernetes_controller.["private_key"]}"
    }
}

resource "null_resource" "ssh-keygen-delete" {
    provisioner "local-exec" {
        command = "ssh-keygen -R ${var.virtual_machine_kubernetes_controller.["ip_address"]}"
    }
}
[root@terraform terraform-vsphere-kubernetes]#
{{< /highlight >}}

### 3. Update 'kubernetes_controller.tf' by adding an external data source at the bottom of the file. 

The terraform documentation describes `data.external` as the following:

'The external data source allows an external program implementing a specific protocol to act as a data source, exposing arbitrary data for use elsewhere in the Terraform configuration.'

At the end of the previous post I mentioned a specific line in the output of the `'kubeadm init'` command that would allow us to join kubernetes nodes to the cluster that was created. In order to know the proper parameters to pass in with the `'kubeadm join'` command, we will use a `data.external` provider to collect the needed information.

The below output shows the `data.external` that I added to the `'kubernetes_controller.tf'` file:

{{< highlight bash >}}
[root@terraform terraform-vsphere-kubernetes]# cat kubernetes_controller.tf
provider "vsphere" {
  user           = "${var.vsphere_connection.["vsphere_user"]}"
  password       = "${var.vsphere_connection.["vsphere_password"]}"
  vsphere_server = "${var.vsphere_connection.["vsphere_server"]}"
  allow_unverified_ssl = true
}

...[Lots more lines of terraform code]...

  provisioner "remote-exec" {
    inline = [
      "chmod +x /tmp/*sh",
      "sudo /tmp/system_setup.sh",
      "sudo /tmp/install_docker.sh",
      "sudo /tmp/install_kubernetes_packages.sh",
      "sudo /tmp/kubeadm_init.sh",
      "tail -n2 /tmp/kubeadm_init_output.txt | head -n 1",
    ]
    connection {
      type          = "${var.virtual_machine_template.["connection_type"]}"
      user          = "${var.virtual_machine_template.["connection_user"]}"
      private_key   = "${file("${var.virtual_machine_kubernetes_controller.["private_key"]}")}"
    }
  }

}

data "external" "kubeadm-init-info" {
  program = ["/usr/bin/bash", "${path.module}/scripts/kubeadm_init_info.sh"]
  query = {
    ip_address  = "${vsphere_virtual_machine.kubernetes_controller.0.default_ip_address}"
    private_key = "${var.virtual_machine_kubernetes_controller.["private_key"]}"
  }
}
{{< /highlight >}}

I would like to point out that the `data.external` block is passing in the ip address from the deployed virtual machine and the ssh private key path into the external data source using the query parameter.
I also want to point out that the `data.external` block is running a script specified in the 'program' parameter. When the `data.external` block is call it will run the program specified on the machine running the terraform code.

### 4. Add a new 'kubeadm_init_info.sh' script to the script directory that the `data.external` block will run.

Next we will create the `'scripts/kubeadm_init_info.sh'` script that contains the logic for getting the kubernetes token and certhash that was generated when running 'kubeadm init.' These two pieces of information will be needed when it comes time to add new kuberenetes nodes.

{{< highlight bash >}}
[root@terraform terraform-vsphere-kubernetes]# cat scripts/kubeadm_init_info.sh
#! /bin/bash

eval "$(jq -r '@sh "IP_ADDRESS=\(.ip_address) PRIVATE_KEY=\(.private_key)"')"

token=$(/usr/bin/ssh root@$IP_ADDRESS -o "IdentityFile $PRIVATE_KEY" -o 'StrictHostKeyChecking no' -o 'UserKnownHostsFile /dev/null' "kubeadm token list | grep -v DESCRIPTION | awk '{print \$1}'")
certhash=$(/usr/bin/ssh root@$IP_ADDRESS -o "IdentityFile $PRIVATE_KEY" -o 'StrictHostKeyChecking no' -o 'UserKnownHostsFile /dev/null' "openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'")
jq -n --arg token "$token" --arg certhash "$certhash" '{"token":$token, "certhash":$certhash}'
[root@terraform terraform-vsphere-kubernetes]#
{{< /highlight >}}

Remember the terraform documentation described that `data.external` woud need to impliment 'a specific protocol to act as a data source.' This statment means that all input parameters will be passed into the data.external program as a json object. All output returned by the data.external program will also be expected to be returned as a valid json object.

In order to make this easier the <a href="https://www.terraform.io/docs/providers/external/data_source.html#processing-json-in-shell-scripts">terraform `data.external` documentation</a> shows how to use `jq` to parse the input parameter and form the output json body. 

Below is a line breakdown of the script that will be used to retrive the kubernetes token and certhash :

* The eval line of the `scripts/kubeadm_init_info.sh` script makes use of `jq` to parse the input variables (ip_address and private_key) and assign them to local shell variables (IP_ADDRESS and PRIVATE_KEY). The terraform documentation mentions that 'jq will ensure that the values are properly quoted and escaped for consumption by the shell.'
* The `token` variable is set to contain the output of a ssh session connecting to the `ip_address` parameter using the `private_key` to parse `'kubeadm token list'` output for the token.
* The `certhash` variable is set to contain the output of ssh session connecting to the `ip_address` parameter using the `private_key` to convert `'/etc/kubernetes/pki/ca.crt'` to the needed format. 
* Finally `jq` is used to format the `token` and `certhash` variables in a json body that is returned by the script.

### 5. Install `jq` on the machine running terraform so that the `scripts/kubeadm_init_info.sh` script can leverage it. 

The `'kubeadm_init_info.sh'` script needs the `jq` program, so we need to make sure it is installed on the machine running the terraform code:

{{< highlight bash >}}
[root@kubernetes-controller ~]# yum install epel-release -y
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirror.dal10.us.leaseweb.net
 * extras: mirror.dal10.us.leaseweb.net
 * updates: mirror.dal10.us.leaseweb.net
Resolving Dependencies
--> Running transaction check
---> Package epel-release.noarch 0:7-11 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

=====================================================================================================================================================================================
 Package                                         Arch                                      Version                                   Repository                                 Size
=====================================================================================================================================================================================
Installing:
 epel-release                                    noarch                                    7-11                                      extras                                     15 k

Transaction Summary
=====================================================================================================================================================================================
Install  1 Package

Total download size: 15 k
Installed size: 24 k
Downloading packages:
epel-release-7-11.noarch.rpm                                                                                                                                  |  15 kB  00:00:00
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : epel-release-7-11.noarch                                                                                                                                          1/1
  Verifying  : epel-release-7-11.noarch                                                                                                                                          1/1

Installed:
  epel-release.noarch 0:7-11

Complete!
[root@kubernetes-controller ~]# yum install jq -y
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
epel/x86_64/metalink                                                                                                                                          |  19 kB  00:00:00
 * base: mirror.dal10.us.leaseweb.net
 * epel: mirror.compevo.com
 * extras: mirror.dal10.us.leaseweb.net
 * updates: mirror.dal10.us.leaseweb.net
epel                                                                                                                                                          | 3.2 kB  00:00:00
(1/3): epel/x86_64/group_gz                                                                                                                                   |  88 kB  00:00:00
(2/3): epel/x86_64/updateinfo                                                                                                                                 | 944 kB  00:00:00
(3/3): epel/x86_64/primary                                                                                                                                    | 3.6 MB  00:00:00
epel                                                                                                                                                                     12776/12776
Resolving Dependencies
--> Running transaction check
---> Package jq.x86_64 0:1.5-1.el7 will be installed
--> Processing Dependency: libonig.so.2()(64bit) for package: jq-1.5-1.el7.x86_64
--> Running transaction check
---> Package oniguruma.x86_64 0:5.9.5-3.el7 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

=====================================================================================================================================================================================
 Package                                      Arch                                      Version                                        Repository                               Size
=====================================================================================================================================================================================
Installing:
 jq                                           x86_64                                    1.5-1.el7                                      epel                                    153 k
Installing for dependencies:
 oniguruma                                    x86_64                                    5.9.5-3.el7                                    epel                                    129 k

Transaction Summary
=====================================================================================================================================================================================
Install  1 Package (+1 Dependent package)

Total download size: 282 k
Installed size: 906 k
Downloading packages:
warning: /var/cache/yum/x86_64/7/epel/packages/jq-1.5-1.el7.x86_64.rpm: Header V3 RSA/SHA256 Signature, key ID 352c64e5: NOKEY
Public key for jq-1.5-1.el7.x86_64.rpm is not installed
(1/2): jq-1.5-1.el7.x86_64.rpm                                                                                                                                | 153 kB  00:00:00
(2/2): oniguruma-5.9.5-3.el7.x86_64.rpm                                                                                                                       | 129 kB  00:00:00
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Total                                                                                                                                                1.3 MB/s | 282 kB  00:00:00
Retrieving key from file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7
Importing GPG key 0x352C64E5:
 Userid     : "Fedora EPEL (7) <epel@fedoraproject.org>"
 Fingerprint: 91e9 7d7c 4a5e 96f1 7f3e 888f 6a2f aea2 352c 64e5
 Package    : epel-release-7-11.noarch (@extras)
 From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : oniguruma-5.9.5-3.el7.x86_64                                                                                                                                      1/2
  Installing : jq-1.5-1.el7.x86_64                                                                                                                                               2/2
  Verifying  : oniguruma-5.9.5-3.el7.x86_64                                                                                                                                      1/2
  Verifying  : jq-1.5-1.el7.x86_64                                                                                                                                               2/2

Installed:
  jq.x86_64 0:1.5-1.el7

Dependency Installed:
  oniguruma.x86_64 0:5.9.5-3.el7

Complete!
[root@kubernetes-controller ~]#
{{< /highlight >}}

### 6. Modify `output.tf` to add the `data.external` results to the output.

Add the `output "kubeadm-init-info"` block to the bottom of the `output.tf` file:

{{< highlight bash >}}
[root@terraform terraform-vsphere-kubernetes]# cat output.tf
output "ip" {
   value = "${vsphere_virtual_machine.kubernetes_controller.*.default_ip_address}"
}
output "vm-moref" {
   value = "${vsphere_virtual_machine.kubernetes_controller.moid}"
}
output "kubeadm-init-info" {
   value = "${data.external.kubeadm-init-info.result}"
}

[root@terraform terraform-vsphere-kubernetes]#
{{< /highlight >}}

### 7. Run `terraform init` to initialize terraform and downloaded the provisioners.

{{< highlight bash >}}
[root@terraform terraform-vsphere-kubernetes]# terraform init

Initializing provider plugins...
- Checking for available provider plugins on https://releases.hashicorp.com...
- Downloading plugin for provider "vsphere" (1.9.0)...
- Downloading plugin for provider "external" (1.0.0)...
- Downloading plugin for provider "null" (1.0.0)...

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

### 8. Run `terraform plan` to have terraform show what will get created.

{{< highlight bash >}}
[root@terraform terraform-vsphere-kubernetes]# terraform plan
Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.

data.vsphere_datacenter.template_datacenter: Refreshing state...
data.vsphere_network.vm_network: Refreshing state...
data.vsphere_datastore.vm_datastore: Refreshing state...
data.vsphere_resource_pool.vm_resource_pool: Refreshing state...
data.vsphere_virtual_machine.template: Refreshing state...

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


Plan: 3 to add, 0 to change, 0 to destroy.

------------------------------------------------------------------------

Note: You didn't specify an "-out" parameter to save this plan, so Terraform
can't guarantee that exactly these actions will be performed if
"terraform apply" is subsequently run.

[root@terraform terraform-vsphere-kubernetes]#
{{< /highlight >}}

Notice at the beginning of the output that the `terraform plan` now includes the `null_resource.ssh-keygen-delete` provisioner and `data.external.kubeadm-init-info` data source that was added in this post.

### 9. Run `terraform apply` to create the described resources.

Running `terraform apply --auto-approve` will have terraform create the resources described in `ssh-key.tf` and `kubernetes_controller.tf`. This time we will jump to the end of the output and spare you most of the output generated

{{< highlight bash >}}
[root@terraform terraform-vsphere-kubernetes]# terraform apply --auto-approve
data.vsphere_datacenter.template_datacenter: Refreshing state...
data.vsphere_datastore.vm_datastore: Refreshing state...
data.vsphere_virtual_machine.template: Refreshing state...
data.vsphere_network.vm_network: Refreshing state...
data.vsphere_resource_pool.vm_resource_pool: Refreshing state...
null_resource.generate-sshkey: Creating...
null_resource.ssh-keygen-delete: Creating...
null_resource.generate-sshkey: Provisioning with 'local-exec'...
null_resource.ssh-keygen-delete: Provisioning with 'local-exec'...
null_resource.generate-sshkey (local-exec): Executing: ["/bin/sh" "-c" "yes y | ssh-keygen -b 4096 -t rsa -C 'terraform-vsphere-kubernetes' -N '' -f /root/.ssh/id_rsa-terraform-vsphere-kubernetes"]
null_resource.ssh-keygen-delete (local-exec): Executing: ["/bin/sh" "-c" "ssh-keygen -R 192.168.100.100"]
null_resource.ssh-keygen-delete (local-exec): Host 192.168.100.100 not found in /root/.ssh/known_hosts
null_resource.ssh-keygen-delete: Creation complete after 0s (ID: 5087424040697272589)
vsphere_virtual_machine.kubernetes_controller: Creating...
  boot_retry_delay:                                     "" => "10000"
  change_version:                                       "" => "<computed>"
  clone.#:                                              "" => "1"
  clone.0.customize.#:                                  "" => "1"
  clone.0.customize.0.dns_server_list.#:                "" => "1"

...[lots more log lines here]...

vsphere_virtual_machine.kubernetes_controller (remote-exec): daemonset.extensions/kube-flannel-ds-amd64 created
vsphere_virtual_machine.kubernetes_controller (remote-exec): daemonset.extensions/kube-flannel-ds-arm64 created
vsphere_virtual_machine.kubernetes_controller (remote-exec): daemonset.extensions/kube-flannel-ds-arm created
vsphere_virtual_machine.kubernetes_controller (remote-exec): daemonset.extensions/kube-flannel-ds-ppc64le created
vsphere_virtual_machine.kubernetes_controller (remote-exec): daemonset.extensions/kube-flannel-ds-s390x created
vsphere_virtual_machine.kubernetes_controller (remote-exec):   kubeadm join 192.168.100.100:6443 --token j8tg5z.gwojyd1xfkyltkqd --discovery-token-ca-cert-hash sha256:c43350a00d11aa3001403cce9dd0be2d7cf0abea9dab32bd84be99fa9d7a55e8
vsphere_virtual_machine.kubernetes_controller: Creation complete after 4m10s (ID: 4221aee3-ddf0-7610-525d-aeaefec89097)
data.external.kubeadm-init-info: Refreshing state...

Apply complete! Resources: 3 added, 0 changed, 0 destroyed.

Outputs:

ip = [
    192.168.100.100
]
kubeadm-init-info = {
  certhash = c43350a00d11aa3001403cce9dd0be2d7cf0abea9dab32bd84be99fa9d7a55e8
  token = j8tg5z.gwojyd1xfkyltkqd
}
vm-moref = vm-1135
[root@terraform terraform-vsphere-kubernetes]#
{{< /highlight >}}

The bottom of the `terraform appy` also now displays the `kubeadm-init-info` certhash and token. If you look up at the `kubeadm join` line, you will notice that both of these values are passed as parameters into the `kubeadm join` command. These two pieces of information, along with the IP address of the kubernetes controller, are all that we need in order to reconstruct the `kubeadm join` command to be run on kubernetes nodes to join the cluster. 

### 10. Run `terraform destroy` to delete the resources that were previously created.

Once you are ready to destroy the virtual machine terraform created, you can simply run `terraform destroy --force` to have terraform destroy it:

{{< highlight bash >}}
[root@terraform terraform-vsphere-kubernetes]# terraform destroy --force
null_resource.ssh-keygen-delete: Refreshing state... (ID: 5087424040697272589)
null_resource.generate-sshkey: Refreshing state... (ID: 6665704479624866294)
data.vsphere_datacenter.template_datacenter: Refreshing state...
data.vsphere_virtual_machine.template: Refreshing state...
data.vsphere_resource_pool.vm_resource_pool: Refreshing state...
data.vsphere_network.vm_network: Refreshing state...
data.vsphere_datastore.vm_datastore: Refreshing state...
vsphere_virtual_machine.kubernetes_controller: Refreshing state... (ID: 4221aee3-ddf0-7610-525d-aeaefec89097)
data.external.kubeadm-init-info: Refreshing state...
null_resource.ssh-keygen-delete: Destroying... (ID: 5087424040697272589)
null_resource.generate-sshkey: Destroying... (ID: 6665704479624866294)
null_resource.generate-sshkey: Destruction complete after 0s
null_resource.ssh-keygen-delete: Destruction complete after 0s
vsphere_virtual_machine.kubernetes_controller: Destroying... (ID: 4221aee3-ddf0-7610-525d-aeaefec89097)
vsphere_virtual_machine.kubernetes_controller: Still destroying... (ID: 4221aee3-ddf0-7610-525d-aeaefec89097, 10s elapsed)
vsphere_virtual_machine.kubernetes_controller: Still destroying... (ID: 4221aee3-ddf0-7610-525d-aeaefec89097, 20s elapsed)
vsphere_virtual_machine.kubernetes_controller: Still destroying... (ID: 4221aee3-ddf0-7610-525d-aeaefec89097, 30s elapsed)
vsphere_virtual_machine.kubernetes_controller: Still destroying... (ID: 4221aee3-ddf0-7610-525d-aeaefec89097, 40s elapsed)
vsphere_virtual_machine.kubernetes_controller: Destruction complete after 43s

Destroy complete! Resources: 3 destroyed.
[root@terraform terraform-vsphere-kubernetes]#
{{< /highlight >}}

#### I hope this post has been useful in demonstrating how to use `data.external` sources to retrieve and re-use information from deployed resources. The files that were using in this post can be found in <a href="https://github.com/sdorsett/terraform-vsphere-kubernetes/tree/2018-12-28-post">this github repository</a>.

---

#### Please provide any feedback or suggestions to my twitter account located on the about page.
