+++
title = "Using local-exec and remote-exec provisioners with terraform"
description = ""
tags = [
    "terraform",
    "vsphere",
    "kubernetes",
]
date = "2018-12-26"
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

This is the fourth in a series of posts that will walk you through using terraform to deploy and configure virtual machines on vsphere. In this post you will get introduced to using local-exec and remote-exe provisioners to make local (on the deloying system) and remote (on the deployed system) changes. If everything goes right we will also have a functional kubernetes controller that we can build on in future posts.

One thing to mention early on in this post is that it is written expecting to deploy from a CentOS 7 template. All the bash scripts we will be using to configure the deployed virtual machine are written to install and configure a CentOS 7 virtual machine, so consider yourself warned. 

---

### 1. Create a new directory for our terraform code.

Start by creating a new directory for the terraform code and `cd` to it.

{{< highlight bash >}}
[root@terraform ~]# mkdir terraform-vsphere-kubernetes
[root@terraform ~]# cd terraform-vsphere-kubernetes/
[root@terraform terraform-vsphere-kubernetes]#
{{< /highlight >}}

### 2. Create `variables.tf` containing the variables that are required.

In the past blog post we used a simple `variables.tf `file consisting of variables with string values. In this post I thought we could use maps to group the variables so that they were better organized. Let us start with a new variables.tf file that contains all our variables and their default values:

{{< highlight bash >}}
[root@terraform terraform-vsphere-kubernetes]# cat variables.tf
variable "vsphere_connection" {
  type                      = "map"
  description               = "Configuration details for connecting to vsphere"

  default = {
    # vsphere login account. defaults to administrator@vsphere.local account
    vsphere_user            = "administrator@vsphere.local"
    # vsphere account password. empty by default
    vsphere_password        = ""
    # vsphere server, defaults to localhost
    vsphere_server          = "localhost"
  }
}

variable "virtual_machine_template" {
  type                      = "map"
  description               = "Configuration details for virtual machine template"

  default = {
    # name of the template to deploy from. empty by default
    name                    = ""
    # default connection_type to SSH
    connection_type         = "ssh"
    # username to connect to deployed virtual machines. defaults to "root"
    connection_user         = "root"
    # default password to initially connect to deployed virtual machines. empty by default
    connection_password     = ""
    # vsphere datacenter that the template is located in. empty by default
    datacenter = ""
  }
}

variable "virtual_machine_kubernetes_controller" {
  type                      = "map"
  description               = "Configuration details for kubernetes_controller virtual machine"

  default = {
    # name of the virtual machine to be deployed. defaults to "kubernetes-controller"
    name                    = "kubernetes-controller"
    # name of the datastore to deploy kubernetes_controller to. defaults to "datastore1"
    datastore               = "datastore1"
    # name of network to deploy kubernetes_controller to. defaults to "VM Network"
    network                 = "VM Network"
    # ip address to be assigned to kubernetes_controller. empty by default
    ip_address              = ""
    # netmask assigned to kubernetes_controller. defaults to "24"
    netmask                 = "24"
    # dns server assigned to kubernetes_controller. defaults to "8.8.8.8"
    dns_server              = "8.8.8.8"
    # default gateway to be assigned to kubernetes_controller. empty by default
    gateway                 = ""
    # resource pool to deploy kubernetes_controller to. empty by default
    resource_pool           = ""
    # private key to be used for SSH connections - this will be generated/overwritten on a terraform apply
    private_key = "/root/.ssh/id_rsa-terraform-vsphere-kubernetes"
    # public key to be copied to virtual machine
    public_key = "/root/.ssh/id_rsa-terraform-vsphere-kubernetes.pub"
    # number of vcpu assigned to kubernetes_controller. default is 2
    num_cpus = 2
    # amount of memory assigned to kubernetes_controller. default is 4096 (4GB)
    memory   = 4096
  }
}

[root@terraform terraform-vsphere-kubernetes]
{{< /highlight >}}

In this `variables.tf` we have 3 base variables that are of the map type:

* vsphere_connection - for vsphere connection related details
* virtual_machine_template - for details about the template we will be cloning
* virtual_machine_kubernetes_controller - for details about the kubernetes_controller virtual machine that will be deployed

Variables that have a type of `map` are hashes that contain keys, and each of those keys has assigned values. This structure lets us keep related pieces of information together and also helps make it more clear as far as what resources the variables are related to.

### 3. Create `terraform.tfvars` and set the variable values.

Just like we did we showed in the previous post, we will use the `terraform.tfvars` file to set the specific values of the variables defined in `variables.tf` for our environment. By keeping the actual value outside of the `variables.tf` we can add `terraform.tfvars` to our `.gitignore` file and keep our remote git repository free from the details of this environment:

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
[root@terraform terraform-vsphere-kubernetes]#
{{< /highlight >}}

### 4. Create `kubernetes_controller.tf` to contain our terraform code for creating the kubernetes_controller virtual machine.

Next we will create `kubernetes_controller.tf` to contain the terraform code that will create the virtual machine resource:
 
{{< highlight bash >}}
provider "vsphere" {
  user           = "${var.vsphere_connection.["vsphere_user"]}"
  password       = "${var.vsphere_connection.["vsphere_password"]}"
  vsphere_server = "${var.vsphere_connection.["vsphere_server"]}"
  allow_unverified_ssl = true
}

data "vsphere_datacenter" "template_datacenter" {
  name = "${var.virtual_machine_template.["datacenter"]}"
}

data "vsphere_datastore" "vm_datastore" {
  name          = "${var.virtual_machine_kubernetes_controller.["datastore"]}"
  datacenter_id = "${data.vsphere_datacenter.template_datacenter.id}"
}

data "vsphere_resource_pool" "vm_resource_pool" {
  name          = "${var.virtual_machine_kubernetes_controller.["resource_pool"]}"
  datacenter_id = "${data.vsphere_datacenter.template_datacenter.id}"
}

data "vsphere_network" "vm_network" {
  name          = "${var.virtual_machine_kubernetes_controller.["network"]}"
  datacenter_id = "${data.vsphere_datacenter.template_datacenter.id}"
}

data "vsphere_virtual_machine" "template" {
  name          = "${var.virtual_machine_template.["name"]}"
  datacenter_id = "${data.vsphere_datacenter.template_datacenter.id}"
}

resource "vsphere_virtual_machine" "kubernetes_controller" {
  name             = "${var.virtual_machine_kubernetes_controller.["name"]}"
  resource_pool_id = "${data.vsphere_resource_pool.vm_resource_pool.id}"
  datastore_id     = "${data.vsphere_datastore.vm_datastore.id}"

  num_cpus = "${var.virtual_machine_kubernetes_controller.["num_cpus"]}"
  memory   = "${var.virtual_machine_kubernetes_controller.["memory"]}"
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
        host_name = "${var.virtual_machine_kubernetes_controller.["name"]}"
        domain    = "kubernetes.local"
      }

      dns_server_list = ["${var.virtual_machine_kubernetes_controller.["dns_server"]}"]
      dns_suffix_list = ["kubernetes.local"]

      network_interface {
        ipv4_address = "${var.virtual_machine_kubernetes_controller.["ip_address"]}"
        ipv4_netmask = "${var.virtual_machine_kubernetes_controller.["netmask"]}"
      }

      ipv4_gateway = "${var.virtual_machine_kubernetes_controller.["gateway"]}"
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
{{< /highlight >}}

There are several things in this `.tf` that are different from the the previous post:

* variables are now using the var.<variable_name>.["<property>"] format. As I mentioned earlier this makes our variables.tf file more logical in how it is organized
* the `vsphere_virtual_machine.kubernetes_controller` resource that is being deployed has a `customize` block that describes guest customization that need to be applied to the deploye virtual machine. This allows us to specify that IP address, netmask, gateway, DNS server and hostname of the deployed virtual machine.
* the `vsphere_virtual_machine.kubernetes_controller` resource has provisioner blocks that allow us to copy files and remotely run commands on the deployed virtual machine.

The provisioner blocks that perform the configuration of the deployed virtual machine are the following:

* the `file` provisioner is used to copy the ssh public key to `/tmp/authorized_keys` on the deployed virtual machine. I want to point out that the connection block will be using the `password` parameter to use the default password setup in the template to connect.

{{< highlight bash >}}
  provisioner "file" {
    source      = "${var.virtual_machine_kubernetes_controller.["public_key"]}"
    destination = "/tmp/authorized_keys"

    connection {
      type        = "${var.virtual_machine_template.["connection_type"]}"
      user        = "${var.virtual_machine_template.["connection_user"]}"
      password    = "${var.virtual_machine_template.["connection_password"]}"
    }
  }
{{< /highlight >}}

* the `remote-exec` provisioner will run the command listed in the inline block on the deployed virtual machine. The commands will move the `/tmp/authorized_keys` file that was copied in the previous step to `/root/.ssh/` and change the permissions. This block will finally disable password ssh authentication in `/etc/ssh/sshd_config` and restart the sshd service so the changes take affect: 

{{< highlight bash >}}
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
{{< /highlight >}}

* The next `file` provisioner block will copy all files located in the local `scripts/` directory to `/tmp/` on the deployed virtual machine. Notice that the connection block is no longer using the `password` option, but is now using `private_key` to connect using the ssh key that will be generated. 

{{< highlight bash >}}
  provisioner "file" {
    source      = "./scripts/"
    destination = "/tmp/"

    connection {
      type        = "${var.virtual_machine_template.["connection_type"]}"
      user        = "${var.virtual_machine_template.["connection_user"]}"
      private_key = "${file("${var.virtual_machine_kubernetes_controller.["private_key"]}")}"
    }
  }
{{< /highlight >}}

* the last `remote-exec` block will make all the `.sh` files in `/tmp/` executable, run the listed `.sh` files and finally output a line from `/tmp/kubeadm_init_output.txt` that is denerated when running the `/tmp/kubeadm_init.sh` script:

{{< highlight bash >}}
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
{{< /highlight >}}


### 5. Create the `scripts directory and the files that will be repotely run to configure the virtual machine as needed

* We now need to create the scripts that the last `remote-exec` provisioner will be running on the deployed virtual machine. First create the scripts directory:

{{< highlight bash >}}
[root@terraform terraform-vsphere-kubernetes]# mkdir scripts
[root@terraform terraform-vsphere-kubernetes]#
{{< /highlight >}}

* Secondly we need to create the `system_setup.sh` that will disable swap, disable the firewall and disable SELINUX 

{{< highlight bash >}}
[root@terraform terraform-vsphere-kubernetes]# cat scripts/system_setup.sh
#! /bin/bash

# disable swap since kubeadm documentation suggests disabling it
swapoff -a
sed -i '/swap/d' /etc/fstab

# disable firewalld and SELinux
systemctl disable firewalld
systemctl stop firewalld
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
setenforce 0
[root@terraform terraform-vsphere-kubernetes]#
{{< /highlight >}}

* The next script that will run is `install_docker.sh` which will install docker and configure the systemd service to start docker.

{{< highlight bash >}}
[root@terraform terraform-vsphere-kubernetes]# cat scripts/install_docker.sh
#! /bin/bash

## Install prerequisites.
yum install yum-utils device-mapper-persistent-data lvm2 -y

## Add docker repository.
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

## Install docker.
yum update -y && yum install docker-ce-18.06.1.ce -y

## Create /etc/docker directory.
mkdir /etc/docker

# Setup daemon.
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF

mkdir -p /etc/systemd/system/docker.service.d

# Restart docker.
systemctl daemon-reload
systemctl restart docker
systemctl enable docker
[root@terraform terraform-vsphere-kubernetes]#
{{< /highlight >}}

* The next script that runs will be the `install_kubernetes_packages.sh` script which will install the kubernetes repo and packages: 

{{< highlight bash >}}
[root@terraform terraform-vsphere-kubernetes]# cat scripts/install_kubernetes_packages.sh
#! /bin/bash

cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kube*
EOF

# Set SELinux in permissive mode (effectively disabling it)
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

yum install -y nfs-utils kubelet kubeadm kubectl openssl --disableexcludes=kubernetes

systemctl enable kubelet && systemctl start kubelet

cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sysctl --system
[root@terraform terraform-vsphere-kubernetes]#
{{< /highlight >}}

* the last script we will run is `kubeadm_init.sh` which will run the kubeadmin command to initialized a kubernetes controller on the deployed virtual machine:

{{< highlight bash >}}
[root@terraform terraform-vsphere-kubernetes]# cat scripts/kubeadm_init.sh
#! /bin/bash
IPADDRESS=$(ip address show dev eth0 | grep 'inet ' | awk '{print $2}' | cut -d"/" -f1)
echo "--> pull kubeadm images <--"
kubeadm config images pull

echo "--> run 'kubeadm init' <--"
kubeadm init --apiserver-advertise-address=$IPADDRESS --pod-network-cidr=10.244.0.0/16 > /tmp/kubeadm_init_output.txt

echo "--> setup $HOME/.kube/config <--"
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

echo "--> install flannel <--"
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

[root@terraform terraform-vsphere-kubernetes]#
{{< /highlight >}}

### 6. Create `output.tf` containing the information we want terraform to return after a run completes.

In the output.tf you can specify what information terraform should display after the run completes. 

{{< highlight bash >}}
[root@terraform terraform-vsphere-kubernetes]# cat output.tf
output "ip" {
   value = "${vsphere_virtual_machine.kubernetes_controller.*.default_ip_address}"
}
output "vm-moref" {
   value = "${vsphere_virtual_machine.kubernetes_controller.moid}"
}

[root@terraform terraform-vsphere-kubernetes]#
{{< /highlight >}}

In this example, just like in the previous post,  terraform will return the ip address and moref of the virtual machine created.

### 7. Create `ssh-key.tf` to contain our terraform code for creating a ssh key for connecting to the deployed virtual machine.

The following is the terraform code that I used to create a ssh key. It is using the `local-exec` provisioner to run command on the machine that is running the terraform code.

{{< highlight bash >}}
[root@terraform terraform-vsphere-kubernetes]# cat ssh-key.tf
resource "null_resource" "generate-sshkey" {
    provisioner "local-exec" {
        command = "yes y | ssh-keygen -b 4096 -t rsa -C 'terraform-vsphere-kubernetes' -N '' -f ${var.virtual_machine_kubernetes_controller.["private_key"]}"
    }
}
[root@terraform terraform-vsphere-kubernetes]#
{{< /highlight >}}

### 8. Run `terraform init` to initialize terraform and downloaded the provisioners.

{{< highlight bash >}}
[root@terraform terraform-vsphere-kubernetes]# terraform init

Initializing provider plugins...
- Checking for available provider plugins on https://releases.hashicorp.com...
- Downloading plugin for provider "vsphere" (1.9.0)...
- Downloading plugin for provider "null" (1.0.0)...

The following providers do not have any version constraints in configuration,
so the latest version was installed.

To prevent automatic upgrades to new major versions that may contain breaking
changes, it is recommended to add version = "..." constraints to the
corresponding provider blocks in configuration, with the constraint strings
suggested below.

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

### 9. Run `terraform plan` to have terraform show what will get created.

The next step is to run `terraform plan` to have terraform validate the vsphere credentials and display the changes it would like to make:

{{< highlight bash >}}
[root@terraform terraform-vsphere-kubernetes]# terraform plan
Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.

data.vsphere_datacenter.template_datacenter: Refreshing state...
data.vsphere_virtual_machine.template: Refreshing state...
data.vsphere_network.vm_network: Refreshing state...
data.vsphere_resource_pool.vm_resource_pool: Refreshing state...
data.vsphere_datastore.vm_datastore: Refreshing state...

------------------------------------------------------------------------

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  + null_resource.generate-sshkey
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


Plan: 2 to add, 0 to change, 0 to destroy.

------------------------------------------------------------------------

Note: You didn't specify an "-out" parameter to save this plan, so Terraform
can't guarantee that exactly these actions will be performed if
"terraform apply" is subsequently run.

[root@terraform terraform-vsphere-kubernetes]#
{{< /highlight >}}

### 10. Run `terraform apply` to create the described resources.

Running `terraform apply --auto-approve` will have terraform create the resource described in `main.tf`:

{{< highlight bash >}}
[root@terraform terraform-vsphere-kubernetes]# terraform apply --auto-approve
data.vsphere_datacenter.template_datacenter: Refreshing state...
data.vsphere_datastore.vm_datastore: Refreshing state...
data.vsphere_resource_pool.vm_resource_pool: Refreshing state...
data.vsphere_network.vm_network: Refreshing state...
data.vsphere_virtual_machine.template: Refreshing state...
null_resource.generate-sshkey: Creating...
null_resource.generate-sshkey: Provisioning with 'local-exec'...
null_resource.generate-sshkey (local-exec): Executing: ["/bin/sh" "-c" "yes y | ssh-keygen -b 4096 -t rsa -C 'terraform-vsphere-kubernetes' -N '' -f /root/.ssh/id_rsa-terraform-vsphere-kubernetes"]
null_resource.generate-sshkey (local-exec): Generating public/private rsa key pair.
null_resource.generate-sshkey (local-exec): /root/.ssh/id_rsa-terraform-vsphere-kubernetes already exists.
null_resource.generate-sshkey (local-exec): Overwrite (y/n)? Your identification has been saved in /root/.ssh/id_rsa-terraform-vsphere-kubernetes.
null_resource.generate-sshkey (local-exec): Your public key has been saved in /root/.ssh/id_rsa-terraform-vsphere-kubernetes.pub.
null_resource.generate-sshkey (local-exec): The key fingerprint is:
null_resource.generate-sshkey (local-exec): SHA256:ybdlc+Ms5FO18zpymVoZTb+ocqLzHOtSxGm6RlEaqMs terraform-vsphere-kubernetes
null_resource.generate-sshkey (local-exec): The key's randomart image is:
null_resource.generate-sshkey (local-exec): +---[RSA 4096]----+
null_resource.generate-sshkey (local-exec): |      .          |
null_resource.generate-sshkey (local-exec): |     . . .       |
null_resource.generate-sshkey (local-exec): |    .   = .     o|
null_resource.generate-sshkey (local-exec): |   .   + *     +o|
null_resource.generate-sshkey (local-exec): |  . .   S . = =oo|
null_resource.generate-sshkey (local-exec): |   E   o o * *.++|
null_resource.generate-sshkey (local-exec): |      . o.. +.=+.|
null_resource.generate-sshkey (local-exec): |       =.oo.o+=. |
null_resource.generate-sshkey (local-exec): |      ..*=+..+.. |
null_resource.generate-sshkey (local-exec): +----[SHA256]-----+
null_resource.generate-sshkey: Creation complete after 0s (ID: 8831700834679792902)
vsphere_virtual_machine.kubernetes_controller: Creating...
  boot_retry_delay:                                     "" => "10000"
  change_version:                                       "" => "<computed>"
  clone.#:                                              "" => "1"
  clone.0.customize.#:                                  "" => "1"
  clone.0.customize.0.dns_server_list.#:                "" => "1"
  clone.0.customize.0.dns_server_list.0:                "" => "192.168.100.1"
  clone.0.customize.0.dns_suffix_list.#:                "" => "1"
  clone.0.customize.0.dns_suffix_list.0:                "" => "kubernetes.local"
  clone.0.customize.0.ipv4_gateway:                     "" => "192.168.100.1"
  clone.0.customize.0.linux_options.#:                  "" => "1"
  clone.0.customize.0.linux_options.0.domain:           "" => "kubernetes.local"
  clone.0.customize.0.linux_options.0.host_name:        "" => "kubernetes-controller"
  clone.0.customize.0.linux_options.0.hw_clock_utc:     "" => "true"
  clone.0.customize.0.network_interface.#:              "" => "1"
  clone.0.customize.0.network_interface.0.ipv4_address: "" => "192.168.100.100"
  clone.0.customize.0.network_interface.0.ipv4_netmask: "" => "24"
  clone.0.customize.0.timeout:                          "" => "10"
  clone.0.template_uuid:                                "" => "4221e272-e449-97e0-39f4-ab580bebc473"
  clone.0.timeout:                                      "" => "30"
  cpu_limit:                                            "" => "-1"
  cpu_share_count:                                      "" => "<computed>"
  cpu_share_level:                                      "" => "normal"
  datastore_id:                                         "" => "datastore-15"
  default_ip_address:                                   "" => "<computed>"
  disk.#:                                               "" => "1"
  disk.0.attach:                                        "" => "false"
  disk.0.datastore_id:                                  "" => "<computed>"
  disk.0.device_address:                                "" => "<computed>"
  disk.0.disk_mode:                                     "" => "persistent"
  disk.0.disk_sharing:                                  "" => "sharingNone"
  disk.0.eagerly_scrub:                                 "" => "false"
  disk.0.io_limit:                                      "" => "-1"
  disk.0.io_reservation:                                "" => "0"
  disk.0.io_share_count:                                "" => "0"
  disk.0.io_share_level:                                "" => "normal"
  disk.0.keep_on_remove:                                "" => "false"
  disk.0.key:                                           "" => "0"
  disk.0.label:                                         "" => "disk0"
  disk.0.path:                                          "" => "<computed>"
  disk.0.size:                                          "" => "8"
  disk.0.thin_provisioned:                              "" => "true"
  disk.0.unit_number:                                   "" => "0"
  disk.0.uuid:                                          "" => "<computed>"
  disk.0.write_through:                                 "" => "false"
  ept_rvi_mode:                                         "" => "automatic"
  firmware:                                             "" => "bios"
  force_power_off:                                      "" => "true"
  guest_id:                                             "" => "rhel7_64Guest"
  guest_ip_addresses.#:                                 "" => "<computed>"
  host_system_id:                                       "" => "<computed>"
  hv_mode:                                              "" => "hvAuto"
  imported:                                             "" => "<computed>"
  latency_sensitivity:                                  "" => "normal"
  memory:                                               "" => "4096"
  memory_limit:                                         "" => "-1"
  memory_share_count:                                   "" => "<computed>"
  memory_share_level:                                   "" => "normal"
  migrate_wait_timeout:                                 "" => "30"
  moid:                                                 "" => "<computed>"
  name:                                                 "" => "kubernetes-controller"
  network_interface.#:                                  "" => "1"
  network_interface.0.adapter_type:                     "" => "vmxnet3"
  network_interface.0.bandwidth_limit:                  "" => "-1"
  network_interface.0.bandwidth_reservation:            "" => "0"
  network_interface.0.bandwidth_share_count:            "" => "<computed>"
  network_interface.0.bandwidth_share_level:            "" => "normal"
  network_interface.0.device_address:                   "" => "<computed>"
  network_interface.0.key:                              "" => "<computed>"
  network_interface.0.mac_address:                      "" => "<computed>"
  network_interface.0.network_id:                       "" => "network-24"
  num_cores_per_socket:                                 "" => "1"
  num_cpus:                                             "" => "2"
  reboot_required:                                      "" => "<computed>"
  resource_pool_id:                                     "" => "resgroup-8"
  run_tools_scripts_after_power_on:                     "" => "true"
  run_tools_scripts_after_resume:                       "" => "true"
  run_tools_scripts_before_guest_shutdown:              "" => "true"
  run_tools_scripts_before_guest_standby:               "" => "true"
  scsi_bus_sharing:                                     "" => "noSharing"
  scsi_controller_count:                                "" => "1"
  scsi_type:                                            "" => "lsilogic"
  shutdown_wait_timeout:                                "" => "3"
  swap_placement_policy:                                "" => "inherit"
  uuid:                                                 "" => "<computed>"
  vmware_tools_status:                                  "" => "<computed>"
  vmx_path:                                             "" => "<computed>"
  wait_for_guest_net_routable:                          "" => "true"
  wait_for_guest_net_timeout:                           "" => "5"
vsphere_virtual_machine.kubernetes_controller: Still creating... (10s elapsed)
vsphere_virtual_machine.kubernetes_controller: Still creating... (20s elapsed)
vsphere_virtual_machine.kubernetes_controller: Still creating... (30s elapsed)
vsphere_virtual_machine.kubernetes_controller: Still creating... (40s elapsed)
vsphere_virtual_machine.kubernetes_controller: Still creating... (50s elapsed)
vsphere_virtual_machine.kubernetes_controller: Still creating... (1m0s elapsed)
vsphere_virtual_machine.kubernetes_controller: Still creating... (1m10s elapsed)
vsphere_virtual_machine.kubernetes_controller: Still creating... (1m20s elapsed)
vsphere_virtual_machine.kubernetes_controller: Still creating... (1m30s elapsed)
vsphere_virtual_machine.kubernetes_controller: Still creating... (1m40s elapsed)
vsphere_virtual_machine.kubernetes_controller: Still creating... (1m50s elapsed)
vsphere_virtual_machine.kubernetes_controller: Still creating... (2m0s elapsed)
vsphere_virtual_machine.kubernetes_controller: Still creating... (2m10s elapsed)
vsphere_virtual_machine.kubernetes_controller: Provisioning with 'file'...
{{< /highlight >}}
At this point in the deployment, the virtual machine IP address is reachable from the deploying machine and the ssh public key file has been copied to the deployed virtual machine
{{< highlight bash >}}
vsphere_virtual_machine.kubernetes_controller: Provisioning with 'remote-exec'...
vsphere_virtual_machine.kubernetes_controller (remote-exec): Connecting to remote host via SSH...
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Host: 192.168.100.100
vsphere_virtual_machine.kubernetes_controller (remote-exec):   User: root
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Password: true
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Private key: false
vsphere_virtual_machine.kubernetes_controller (remote-exec):   SSH Agent: false
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Checking Host Key: false
vsphere_virtual_machine.kubernetes_controller (remote-exec): Connected!
vsphere_virtual_machine.kubernetes_controller (remote-exec): Redirecting to /bin/systemctl restart sshd.service
{{< /highlight >}}
The first `remote-exec` block has just moved the ssh public key to the proper location, disabled ssh password authentioned and the restarted sshd service.
{{< highlight bash >}}
vsphere_virtual_machine.kubernetes_controller: Provisioning with 'file'...
{{< /highlight >}}
The `./scripts/` files have now been copied to the deploy virtual machine. Next the `scripts\install_docker.sh` script will run remotely to install the necessary packages.
{{< highlight bash >}}
vsphere_virtual_machine.kubernetes_controller: Provisioning with 'remote-exec'...
vsphere_virtual_machine.kubernetes_controller (remote-exec): Connecting to remote host via SSH...
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Host: 192.168.100.100
vsphere_virtual_machine.kubernetes_controller (remote-exec):   User: root
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Password: false
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Private key: true
vsphere_virtual_machine.kubernetes_controller (remote-exec):   SSH Agent: false
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Checking Host Key: false
vsphere_virtual_machine.kubernetes_controller: Still creating... (2m20s elapsed)
vsphere_virtual_machine.kubernetes_controller (remote-exec): Connected!
vsphere_virtual_machine.kubernetes_controller (remote-exec): setenforce: SELinux is disabled
vsphere_virtual_machine.kubernetes_controller (remote-exec): Loaded plugins: fastestmirror
vsphere_virtual_machine.kubernetes_controller (remote-exec): Determining fastest mirrors
vsphere_virtual_machine.kubernetes_controller (remote-exec):  * base: centos.mirror.lstn.net
vsphere_virtual_machine.kubernetes_controller (remote-exec):  * extras: centos.mirror.lstn.net
vsphere_virtual_machine.kubernetes_controller (remote-exec):  * updates: centos.mirror.lstn.net
vsphere_virtual_machine.kubernetes_controller (remote-exec): base             | 3.6 kB     00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): extras           | 3.4 kB     00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): updates          | 3.4 kB     00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): (1/4): base/7/x86_ | 166 kB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): (2/4): extras/7/x8 | 156 kB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): (3/4): updates/7/x | 1.3 MB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): (4/4): base/7/ 58% | 4.5 MB   --:-- ETA
vsphere_virtual_machine.kubernetes_controller (remote-exec): (4/4): base/7/x86_ | 6.0 MB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): Package device-mapper-persistent-data-0.7.3-3.el7.x86_64 already installed and latest version
vsphere_virtual_machine.kubernetes_controller (remote-exec): Package 7:lvm2-2.02.180-10.el7_6.2.x86_64 already installed and latest version
vsphere_virtual_machine.kubernetes_controller (remote-exec): Resolving Dependencies
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Running transaction check
vsphere_virtual_machine.kubernetes_controller (remote-exec): ---> Package yum-utils.noarch 0:1.1.31-50.el7 will be installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: python-kitchen for package: yum-utils-1.1.31-50.el7.noarch
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: libxml2-python for package: yum-utils-1.1.31-50.el7.noarch
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Running transaction check
vsphere_virtual_machine.kubernetes_controller (remote-exec): ---> Package libxml2-python.x86_64 0:2.9.1-6.el7_2.3 will be installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): ---> Package python-kitchen.noarch 0:1.1.1-5.el7 will be installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: python-chardet for package: python-kitchen-1.1.1-5.el7.noarch
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Running transaction check
vsphere_virtual_machine.kubernetes_controller (remote-exec): ---> Package python-chardet.noarch 0:2.2.1-1.el7_1 will be installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Finished Dependency Resolution

vsphere_virtual_machine.kubernetes_controller (remote-exec): Dependencies Resolved

vsphere_virtual_machine.kubernetes_controller (remote-exec): ========================================
vsphere_virtual_machine.kubernetes_controller (remote-exec):  Package
vsphere_virtual_machine.kubernetes_controller (remote-exec):       Arch   Version         Repository
vsphere_virtual_machine.kubernetes_controller (remote-exec):                                    Size
vsphere_virtual_machine.kubernetes_controller (remote-exec): ========================================
vsphere_virtual_machine.kubernetes_controller (remote-exec): Installing:
vsphere_virtual_machine.kubernetes_controller (remote-exec):  yum-utils
vsphere_virtual_machine.kubernetes_controller (remote-exec):       noarch 1.1.31-50.el7   base 121 k
vsphere_virtual_machine.kubernetes_controller (remote-exec): Installing for dependencies:
vsphere_virtual_machine.kubernetes_controller (remote-exec):  libxml2-python
vsphere_virtual_machine.kubernetes_controller (remote-exec):       x86_64 2.9.1-6.el7_2.3 base 247 k
vsphere_virtual_machine.kubernetes_controller (remote-exec):  python-chardet
vsphere_virtual_machine.kubernetes_controller (remote-exec):       noarch 2.2.1-1.el7_1   base 227 k
vsphere_virtual_machine.kubernetes_controller (remote-exec):  python-kitchen
vsphere_virtual_machine.kubernetes_controller (remote-exec):       noarch 1.1.1-5.el7     base 267 k

vsphere_virtual_machine.kubernetes_controller (remote-exec): Transaction Summary
vsphere_virtual_machine.kubernetes_controller (remote-exec): ========================================
vsphere_virtual_machine.kubernetes_controller (remote-exec): Install  1 Package (+3 Dependent packages)

vsphere_virtual_machine.kubernetes_controller (remote-exec): Total download size: 861 k
vsphere_virtual_machine.kubernetes_controller (remote-exec): Installed size: 4.3 M
vsphere_virtual_machine.kubernetes_controller (remote-exec): Downloading packages:
vsphere_virtual_machine.kubernetes_controller (remote-exec): (1/4): libxml2-pyt | 247 kB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): (2/4): python-char | 227 kB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): (3/4): python-kitc | 267 kB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): (4/4): yum-utils-1 | 121 kB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): ----------------------------------------
vsphere_virtual_machine.kubernetes_controller (remote-exec): Total      5.5 MB/s | 861 kB  00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): Running transaction check
vsphere_virtual_machine.kubernetes_controller (remote-exec): Running transaction test
vsphere_virtual_machine.kubernetes_controller: Still creating... (2m30s elapsed)
vsphere_virtual_machine.kubernetes_controller (remote-exec): Transaction test succeeded
vsphere_virtual_machine.kubernetes_controller (remote-exec): Running transaction

vsphere_virtual_machine.kubernetes_controller (remote-exec): Installed:
vsphere_virtual_machine.kubernetes_controller (remote-exec):   yum-utils.noarch 0:1.1.31-50.el7

vsphere_virtual_machine.kubernetes_controller (remote-exec): Dependency Installed:
vsphere_virtual_machine.kubernetes_controller (remote-exec):   libxml2-python.x86_64 0:2.9.1-6.el7_2.3
vsphere_virtual_machine.kubernetes_controller (remote-exec):   python-chardet.noarch 0:2.2.1-1.el7_1
vsphere_virtual_machine.kubernetes_controller (remote-exec):   python-kitchen.noarch 0:1.1.1-5.el7

vsphere_virtual_machine.kubernetes_controller (remote-exec): Complete!
vsphere_virtual_machine.kubernetes_controller (remote-exec): Loaded plugins: fastestmirror
vsphere_virtual_machine.kubernetes_controller (remote-exec): adding repo from: https://download.docker.com/linux/centos/docker-ce.repo
vsphere_virtual_machine.kubernetes_controller (remote-exec): grabbing file https://download.docker.com/linux/centos/docker-ce.repo to /etc/yum.repos.d/docker-ce.repo
vsphere_virtual_machine.kubernetes_controller (remote-exec): repo saved to /etc/yum.repos.d/docker-ce.repo
vsphere_virtual_machine.kubernetes_controller (remote-exec): Loaded plugins: fastestmirror
vsphere_virtual_machine.kubernetes_controller (remote-exec): Loading mirror speeds from cached hostfile
vsphere_virtual_machine.kubernetes_controller (remote-exec):  * base: centos.mirror.lstn.net
vsphere_virtual_machine.kubernetes_controller (remote-exec):  * extras: centos.mirror.lstn.net
vsphere_virtual_machine.kubernetes_controller (remote-exec):  * updates: centos.mirror.lstn.net
vsphere_virtual_machine.kubernetes_controller (remote-exec): docker-ce-stable | 3.5 kB     00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): (1/2): docker-ce-s |   55 B   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): (2/2): docker-ce-s |  19 kB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): No packages marked for update
vsphere_virtual_machine.kubernetes_controller (remote-exec): Loaded plugins: fastestmirror
vsphere_virtual_machine.kubernetes_controller (remote-exec): Loading mirror speeds from cached hostfile
vsphere_virtual_machine.kubernetes_controller (remote-exec):  * base: centos.mirror.lstn.net
vsphere_virtual_machine.kubernetes_controller (remote-exec):  * extras: centos.mirror.lstn.net
vsphere_virtual_machine.kubernetes_controller (remote-exec):  * updates: centos.mirror.lstn.net
vsphere_virtual_machine.kubernetes_controller (remote-exec): Resolving Dependencies
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Running transaction check
vsphere_virtual_machine.kubernetes_controller (remote-exec): ---> Package docker-ce.x86_64 0:18.06.1.ce-3.el7 will be installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: container-selinux >= 2.9 for package: docker-ce-18.06.1.ce-3.el7.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: libcgroup for package: docker-ce-18.06.1.ce-3.el7.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: libltdl.so.7()(64bit) for package: docker-ce-18.06.1.ce-3.el7.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Running transaction check
vsphere_virtual_machine.kubernetes_controller (remote-exec): ---> Package container-selinux.noarch 2:2.74-1.el7 will be installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: policycoreutils-python for package: 2:container-selinux-2.74-1.el7.noarch
vsphere_virtual_machine.kubernetes_controller (remote-exec): ---> Package libcgroup.x86_64 0:0.41-20.el7 will be installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): ---> Package libtool-ltdl.x86_64 0:2.4.2-22.el7_3 will be installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Running transaction check
vsphere_virtual_machine.kubernetes_controller (remote-exec): ---> Package policycoreutils-python.x86_64 0:2.5-29.el7 will be installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: setools-libs >= 3.3.8-4 for package: policycoreutils-python-2.5-29.el7.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: libsemanage-python >= 2.5-14 for package: policycoreutils-python-2.5-29.el7.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: audit-libs-python >= 2.1.3-4 for package: policycoreutils-python-2.5-29.el7.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: python-IPy for package: policycoreutils-python-2.5-29.el7.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: libqpol.so.1(VERS_1.4)(64bit) for package: policycoreutils-python-2.5-29.el7.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: libqpol.so.1(VERS_1.2)(64bit) for package: policycoreutils-python-2.5-29.el7.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: libapol.so.4(VERS_4.0)(64bit) for package: policycoreutils-python-2.5-29.el7.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: checkpolicy for package: policycoreutils-python-2.5-29.el7.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: libqpol.so.1()(64bit) for package: policycoreutils-python-2.5-29.el7.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: libapol.so.4()(64bit) for package: policycoreutils-python-2.5-29.el7.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Running transaction check
vsphere_virtual_machine.kubernetes_controller (remote-exec): ---> Package audit-libs-python.x86_64 0:2.8.4-4.el7 will be installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): ---> Package checkpolicy.x86_64 0:2.5-8.el7 will be installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): ---> Package libsemanage-python.x86_64 0:2.5-14.el7 will be installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): ---> Package python-IPy.noarch 0:0.75-6.el7 will be installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): ---> Package setools-libs.x86_64 0:3.3.8-4.el7 will be installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Finished Dependency Resolution

vsphere_virtual_machine.kubernetes_controller (remote-exec): Dependencies Resolved

vsphere_virtual_machine.kubernetes_controller (remote-exec): ========================================
vsphere_virtual_machine.kubernetes_controller (remote-exec):  Package
vsphere_virtual_machine.kubernetes_controller (remote-exec):    Arch   Version          Repository
vsphere_virtual_machine.kubernetes_controller (remote-exec):                                    Size
vsphere_virtual_machine.kubernetes_controller (remote-exec): ========================================
vsphere_virtual_machine.kubernetes_controller (remote-exec): Installing:
vsphere_virtual_machine.kubernetes_controller (remote-exec):  docker-ce
vsphere_virtual_machine.kubernetes_controller (remote-exec):    x86_64 18.06.1.ce-3.el7 docker-ce-stable
vsphere_virtual_machine.kubernetes_controller (remote-exec):                                    41 M
vsphere_virtual_machine.kubernetes_controller (remote-exec): Installing for dependencies:
vsphere_virtual_machine.kubernetes_controller (remote-exec):  audit-libs-python
vsphere_virtual_machine.kubernetes_controller (remote-exec):    x86_64 2.8.4-4.el7      base    76 k
vsphere_virtual_machine.kubernetes_controller (remote-exec):  checkpolicy
vsphere_virtual_machine.kubernetes_controller (remote-exec):    x86_64 2.5-8.el7        base   295 k
vsphere_virtual_machine.kubernetes_controller (remote-exec):  container-selinux
vsphere_virtual_machine.kubernetes_controller (remote-exec):    noarch 2:2.74-1.el7     extras  38 k
vsphere_virtual_machine.kubernetes_controller (remote-exec):  libcgroup
vsphere_virtual_machine.kubernetes_controller (remote-exec):    x86_64 0.41-20.el7      base    66 k
vsphere_virtual_machine.kubernetes_controller (remote-exec):  libsemanage-python
vsphere_virtual_machine.kubernetes_controller (remote-exec):    x86_64 2.5-14.el7       base   113 k
vsphere_virtual_machine.kubernetes_controller (remote-exec):  libtool-ltdl
vsphere_virtual_machine.kubernetes_controller (remote-exec):    x86_64 2.4.2-22.el7_3   base    49 k
vsphere_virtual_machine.kubernetes_controller (remote-exec):  policycoreutils-python
vsphere_virtual_machine.kubernetes_controller (remote-exec):    x86_64 2.5-29.el7       base   456 k
vsphere_virtual_machine.kubernetes_controller (remote-exec):  python-IPy
vsphere_virtual_machine.kubernetes_controller (remote-exec):    noarch 0.75-6.el7       base    32 k
vsphere_virtual_machine.kubernetes_controller (remote-exec):  setools-libs
vsphere_virtual_machine.kubernetes_controller (remote-exec):    x86_64 3.3.8-4.el7      base   620 k

vsphere_virtual_machine.kubernetes_controller (remote-exec): Transaction Summary
vsphere_virtual_machine.kubernetes_controller (remote-exec): ========================================
vsphere_virtual_machine.kubernetes_controller (remote-exec): Install  1 Package (+9 Dependent packages)

vsphere_virtual_machine.kubernetes_controller (remote-exec): Total download size: 42 M
vsphere_virtual_machine.kubernetes_controller (remote-exec): Installed size: 46 M
vsphere_virtual_machine.kubernetes_controller (remote-exec): Downloading packages:
vsphere_virtual_machine.kubernetes_controller (remote-exec): (1/10): audit-libs |  76 kB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): (2/10): libcgroup- |  66 kB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): (3/10): container- |  38 kB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): (4/10): libsemanag | 113 kB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): (5/10): checkpolic | 295 kB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): (6/10): libtool-lt |  49 kB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): (7/10): python-IPy |  32 kB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): (8/10): policycore | 456 kB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): (9/10): setools-li | 620 kB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): (10/10): docker 4% | 1.7 MB   --:-- ETA
vsphere_virtual_machine.kubernetes_controller (remote-exec): (10/10): docke 14% | 6.2 MB   00:06 ETA
vsphere_virtual_machine.kubernetes_controller (remote-exec): (10/10): docke 25% |  11 MB   00:04 ETA
vsphere_virtual_machine.kubernetes_controller (remote-exec): (10/10): docke 36% |  15 MB   00:03 ETA
vsphere_virtual_machine.kubernetes_controller (remote-exec): (10/10): docke 46% |  20 MB   00:03 ETA
vsphere_virtual_machine.kubernetes_controller (remote-exec): (10/10): docke 57% |  24 MB   00:02 ETA
vsphere_virtual_machine.kubernetes_controller (remote-exec): (10/10): docke 68% |  29 MB   00:01 ETA
vsphere_virtual_machine.kubernetes_controller (remote-exec): (10/10): docke 78% |  34 MB   00:01 ETA
vsphere_virtual_machine.kubernetes_controller (remote-exec): (10/10): docke 89% |  38 MB   00:00 ETA
vsphere_virtual_machine.kubernetes_controller (remote-exec): warning: /var/cache/yum/x86_64/7/docker-ce-stable/packages/docker-ce-18.06.1.ce-3.el7.x86_64.rpm: Header V4 RSA/SHA512 Signature, key ID 621e9f35: NOKEY
vsphere_virtual_machine.kubernetes_controller (remote-exec): Public key for docker-ce-18.06.1.ce-3.el7.x86_64.rpm is not installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): (10/10): docker-ce |  41 MB   00:03
vsphere_virtual_machine.kubernetes_controller (remote-exec): ----------------------------------------
vsphere_virtual_machine.kubernetes_controller (remote-exec): Total       13 MB/s |  42 MB  00:03
vsphere_virtual_machine.kubernetes_controller (remote-exec): Retrieving key from https://download.docker.com/linux/centos/gpg
vsphere_virtual_machine.kubernetes_controller (remote-exec): Importing GPG key 0x621E9F35:
vsphere_virtual_machine.kubernetes_controller (remote-exec):  Userid     : "Docker Release (CE rpm) <docker@docker.com>"
vsphere_virtual_machine.kubernetes_controller (remote-exec):  Fingerprint: 060a 61c5 1b55 8a7f 742b 77aa c52f eb6b 621e 9f35
vsphere_virtual_machine.kubernetes_controller (remote-exec):  From       : https://download.docker.com/linux/centos/gpg
vsphere_virtual_machine.kubernetes_controller (remote-exec): Running transaction check
vsphere_virtual_machine.kubernetes_controller (remote-exec): Running transaction test
vsphere_virtual_machine.kubernetes_controller (remote-exec): Transaction test succeeded
vsphere_virtual_machine.kubernetes_controller (remote-exec): Running transaction
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Verifying  : libcgroup-0.41-2    1/10
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Verifying  : checkpolicy-2.5-    2/10
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Verifying  : docker-ce-18.06.    3/10
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Verifying  : libsemanage-pyth    4/10
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Verifying  : policycoreutils-    5/10
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Verifying  : python-IPy-0.75-    6/10
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Verifying  : libtool-ltdl-2.4    7/10
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Verifying  : 2:container-seli    8/10
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Verifying  : audit-libs-pytho    9/10
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Verifying  : setools-libs-3.3   10/10

vsphere_virtual_machine.kubernetes_controller (remote-exec): Installed:
vsphere_virtual_machine.kubernetes_controller (remote-exec):   docker-ce.x86_64 0:18.06.1.ce-3.el7

vsphere_virtual_machine.kubernetes_controller (remote-exec): Dependency Installed:
vsphere_virtual_machine.kubernetes_controller (remote-exec):   audit-libs-python.x86_64 0:2.8.4-4.el7
vsphere_virtual_machine.kubernetes_controller (remote-exec):   checkpolicy.x86_64 0:2.5-8.el7
vsphere_virtual_machine.kubernetes_controller (remote-exec):   container-selinux.noarch 2:2.74-1.el7
vsphere_virtual_machine.kubernetes_controller (remote-exec):   libcgroup.x86_64 0:0.41-20.el7
vsphere_virtual_machine.kubernetes_controller (remote-exec):   libsemanage-python.x86_64 0:2.5-14.el7
vsphere_virtual_machine.kubernetes_controller (remote-exec):   libtool-ltdl.x86_64 0:2.4.2-22.el7_3
vsphere_virtual_machine.kubernetes_controller (remote-exec):   policycoreutils-python.x86_64 0:2.5-29.el7
vsphere_virtual_machine.kubernetes_controller (remote-exec):   python-IPy.noarch 0:0.75-6.el7
vsphere_virtual_machine.kubernetes_controller (remote-exec):   setools-libs.x86_64 0:3.3.8-4.el7

vsphere_virtual_machine.kubernetes_controller (remote-exec): Complete!
vsphere_virtual_machine.kubernetes_controller: Still creating... (2m50s elapsed)
vsphere_virtual_machine.kubernetes_controller (remote-exec): Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to /usr/lib/systemd/system/docker.service.
vsphere_virtual_machine.kubernetes_controller (remote-exec): setenforce: SELinux is disabled
{{< /highlight >}}
Next the `./scripts/install_kubernetes.sh` script will be run to install the kuberenetes packages needed:
{{< highlight bash >}}
vsphere_virtual_machine.kubernetes_controller (remote-exec): Loaded plugins: fastestmirror
vsphere_virtual_machine.kubernetes_controller (remote-exec): Loading mirror speeds from cached hostfile
vsphere_virtual_machine.kubernetes_controller (remote-exec):  * base: centos.mirror.lstn.net
vsphere_virtual_machine.kubernetes_controller (remote-exec):  * extras: centos.mirror.lstn.net
vsphere_virtual_machine.kubernetes_controller (remote-exec):  * updates: centos.mirror.lstn.net
vsphere_virtual_machine.kubernetes_controller (remote-exec): kubernetes/signa |  454 B     00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): Retrieving key from https://packages.cloud.google.com/yum/doc/yum-key.gpg
vsphere_virtual_machine.kubernetes_controller (remote-exec): Importing GPG key 0xA7317B0F:
vsphere_virtual_machine.kubernetes_controller (remote-exec):  Userid     : "Google Cloud Packages Automatic Signing Key <gc-team@google.com>"
vsphere_virtual_machine.kubernetes_controller (remote-exec):  Fingerprint: d0bc 747f d8ca f711 7500 d6fa 3746 c208 a731 7b0f
vsphere_virtual_machine.kubernetes_controller (remote-exec):  From       : https://packages.cloud.google.com/yum/doc/yum-key.gpg
vsphere_virtual_machine.kubernetes_controller (remote-exec): Retrieving key from https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
vsphere_virtual_machine.kubernetes_controller (remote-exec): kubernetes/signa | 1.4 kB     00:00 !!!
vsphere_virtual_machine.kubernetes_controller (remote-exec): kubernetes/primary |  41 kB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): Resolving Dependencies
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Running transaction check
vsphere_virtual_machine.kubernetes_controller (remote-exec): ---> Package kubeadm.x86_64 0:1.13.1-0 will be installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: kubernetes-cni >= 0.6.0 for package: kubeadm-1.13.1-0.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: cri-tools >= 1.11.0 for package: kubeadm-1.13.1-0.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): ---> Package kubectl.x86_64 0:1.13.1-0 will be installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): ---> Package kubelet.x86_64 0:1.13.1-0 will be installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: socat for package: kubelet-1.13.1-0.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): ---> Package nfs-utils.x86_64 1:1.3.0-0.61.el7 will be installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: libtirpc >= 0.2.4-0.7 for package: 1:nfs-utils-1.3.0-0.61.el7.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: gssproxy >= 0.7.0-3 for package: 1:nfs-utils-1.3.0-0.61.el7.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: rpcbind for package: 1:nfs-utils-1.3.0-0.61.el7.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: quota for package: 1:nfs-utils-1.3.0-0.61.el7.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: libnfsidmap for package: 1:nfs-utils-1.3.0-0.61.el7.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: libevent for package: 1:nfs-utils-1.3.0-0.61.el7.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: keyutils for package: 1:nfs-utils-1.3.0-0.61.el7.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: libtirpc.so.1()(64bit) for package: 1:nfs-utils-1.3.0-0.61.el7.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: libnfsidmap.so.0()(64bit) for package: 1:nfs-utils-1.3.0-0.61.el7.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: libevent-2.0.so.5()(64bit) for package: 1:nfs-utils-1.3.0-0.61.el7.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): ---> Package openssl.x86_64 1:1.0.2k-16.el7 will be installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: make for package: 1:openssl-1.0.2k-16.el7.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Running transaction check
vsphere_virtual_machine.kubernetes_controller (remote-exec): ---> Package cri-tools.x86_64 0:1.12.0-0 will be installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): ---> Package gssproxy.x86_64 0:0.7.0-21.el7 will be installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: libini_config >= 1.3.1-31 for package: gssproxy-0.7.0-21.el7.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: libverto-module-base for package: gssproxy-0.7.0-21.el7.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: libref_array.so.1(REF_ARRAY_0.1.1)(64bit) for package: gssproxy-0.7.0-21.el7.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: libini_config.so.3(INI_CONFIG_1.2.0)(64bit) for package: gssproxy-0.7.0-21.el7.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: libini_config.so.3(INI_CONFIG_1.1.0)(64bit) for package: gssproxy-0.7.0-21.el7.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: libref_array.so.1()(64bit) for package: gssproxy-0.7.0-21.el7.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: libini_config.so.3()(64bit) for package: gssproxy-0.7.0-21.el7.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: libcollection.so.2()(64bit) for package: gssproxy-0.7.0-21.el7.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: libbasicobjects.so.0()(64bit) for package: gssproxy-0.7.0-21.el7.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): ---> Package keyutils.x86_64 0:1.5.8-3.el7 will be installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): ---> Package kubernetes-cni.x86_64 0:0.6.0-0 will be installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): ---> Package libevent.x86_64 0:2.0.21-4.el7 will be installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): ---> Package libnfsidmap.x86_64 0:0.25-19.el7 will be installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): ---> Package libtirpc.x86_64 0:0.2.4-0.15.el7 will be installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): ---> Package make.x86_64 1:3.82-23.el7 will be installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): ---> Package quota.x86_64 1:4.01-17.el7 will be installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: quota-nls = 1:4.01-17.el7 for package: 1:quota-4.01-17.el7.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: tcp_wrappers for package: 1:quota-4.01-17.el7.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): ---> Package rpcbind.x86_64 0:0.2.0-47.el7 will be installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): ---> Package socat.x86_64 0:1.7.3.2-2.el7 will be installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Running transaction check
vsphere_virtual_machine.kubernetes_controller (remote-exec): ---> Package libbasicobjects.x86_64 0:0.1.1-32.el7 will be installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): ---> Package libcollection.x86_64 0:0.7.0-32.el7 will be installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): ---> Package libini_config.x86_64 0:1.3.1-32.el7 will be installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: libpath_utils.so.1(PATH_UTILS_0.2.1)(64bit) for package: libini_config-1.3.1-32.el7.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Processing Dependency: libpath_utils.so.1()(64bit) for package: libini_config-1.3.1-32.el7.x86_64
vsphere_virtual_machine.kubernetes_controller (remote-exec): ---> Package libref_array.x86_64 0:0.1.5-32.el7 will be installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): ---> Package libverto-libevent.x86_64 0:0.2.5-4.el7 will be installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): ---> Package quota-nls.noarch 1:4.01-17.el7 will be installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): ---> Package tcp_wrappers.x86_64 0:7.6-77.el7 will be installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Running transaction check
vsphere_virtual_machine.kubernetes_controller (remote-exec): ---> Package libpath_utils.x86_64 0:0.2.1-32.el7 will be installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> Finished Dependency Resolution

vsphere_virtual_machine.kubernetes_controller (remote-exec): Dependencies Resolved

vsphere_virtual_machine.kubernetes_controller (remote-exec): ========================================
vsphere_virtual_machine.kubernetes_controller (remote-exec):  Package
vsphere_virtual_machine.kubernetes_controller (remote-exec):        Arch   Version        Repository
vsphere_virtual_machine.kubernetes_controller (remote-exec):                                    Size
vsphere_virtual_machine.kubernetes_controller (remote-exec): ========================================
vsphere_virtual_machine.kubernetes_controller (remote-exec): Installing:
vsphere_virtual_machine.kubernetes_controller (remote-exec):  kubeadm
vsphere_virtual_machine.kubernetes_controller (remote-exec):        x86_64 1.13.1-0       kubernetes
vsphere_virtual_machine.kubernetes_controller (remote-exec):                                   7.9 M
vsphere_virtual_machine.kubernetes_controller (remote-exec):  kubectl
vsphere_virtual_machine.kubernetes_controller (remote-exec):        x86_64 1.13.1-0       kubernetes
vsphere_virtual_machine.kubernetes_controller (remote-exec):                                   8.5 M
vsphere_virtual_machine.kubernetes_controller (remote-exec):  kubelet
vsphere_virtual_machine.kubernetes_controller (remote-exec):        x86_64 1.13.1-0       kubernetes
vsphere_virtual_machine.kubernetes_controller (remote-exec):                                    21 M
vsphere_virtual_machine.kubernetes_controller (remote-exec):  nfs-utils
vsphere_virtual_machine.kubernetes_controller (remote-exec):        x86_64 1:1.3.0-0.61.el7
vsphere_virtual_machine.kubernetes_controller (remote-exec):                              base 410 k
vsphere_virtual_machine.kubernetes_controller (remote-exec):  openssl
vsphere_virtual_machine.kubernetes_controller (remote-exec):        x86_64 1:1.0.2k-16.el7
vsphere_virtual_machine.kubernetes_controller (remote-exec):                              base 493 k
vsphere_virtual_machine.kubernetes_controller (remote-exec): Installing for dependencies:
vsphere_virtual_machine.kubernetes_controller (remote-exec):  cri-tools
vsphere_virtual_machine.kubernetes_controller (remote-exec):        x86_64 1.12.0-0       kubernetes
vsphere_virtual_machine.kubernetes_controller (remote-exec):                                   4.2 M
vsphere_virtual_machine.kubernetes_controller (remote-exec):  gssproxy
vsphere_virtual_machine.kubernetes_controller (remote-exec):        x86_64 0.7.0-21.el7   base 109 k
vsphere_virtual_machine.kubernetes_controller (remote-exec):  keyutils
vsphere_virtual_machine.kubernetes_controller (remote-exec):        x86_64 1.5.8-3.el7    base  54 k
vsphere_virtual_machine.kubernetes_controller (remote-exec):  kubernetes-cni
vsphere_virtual_machine.kubernetes_controller (remote-exec):        x86_64 0.6.0-0        kubernetes
vsphere_virtual_machine.kubernetes_controller (remote-exec):                                   8.6 M
vsphere_virtual_machine.kubernetes_controller (remote-exec):  libbasicobjects
vsphere_virtual_machine.kubernetes_controller (remote-exec):        x86_64 0.1.1-32.el7   base  26 k
vsphere_virtual_machine.kubernetes_controller (remote-exec):  libcollection
vsphere_virtual_machine.kubernetes_controller (remote-exec):        x86_64 0.7.0-32.el7   base  42 k
vsphere_virtual_machine.kubernetes_controller (remote-exec):  libevent
vsphere_virtual_machine.kubernetes_controller (remote-exec):        x86_64 2.0.21-4.el7   base 214 k
vsphere_virtual_machine.kubernetes_controller (remote-exec):  libini_config
vsphere_virtual_machine.kubernetes_controller (remote-exec):        x86_64 1.3.1-32.el7   base  64 k
vsphere_virtual_machine.kubernetes_controller (remote-exec):  libnfsidmap
vsphere_virtual_machine.kubernetes_controller (remote-exec):        x86_64 0.25-19.el7    base  50 k
vsphere_virtual_machine.kubernetes_controller (remote-exec):  libpath_utils
vsphere_virtual_machine.kubernetes_controller (remote-exec):        x86_64 0.2.1-32.el7   base  28 k
vsphere_virtual_machine.kubernetes_controller (remote-exec):  libref_array
vsphere_virtual_machine.kubernetes_controller (remote-exec):        x86_64 0.1.5-32.el7   base  27 k
vsphere_virtual_machine.kubernetes_controller (remote-exec):  libtirpc
vsphere_virtual_machine.kubernetes_controller (remote-exec):        x86_64 0.2.4-0.15.el7 base  89 k
vsphere_virtual_machine.kubernetes_controller (remote-exec):  libverto-libevent
vsphere_virtual_machine.kubernetes_controller (remote-exec):        x86_64 0.2.5-4.el7    base 8.9 k
vsphere_virtual_machine.kubernetes_controller (remote-exec):  make  x86_64 1:3.82-23.el7  base 420 k
vsphere_virtual_machine.kubernetes_controller (remote-exec):  quota x86_64 1:4.01-17.el7  base 179 k
vsphere_virtual_machine.kubernetes_controller (remote-exec):  quota-nls
vsphere_virtual_machine.kubernetes_controller (remote-exec):        noarch 1:4.01-17.el7  base  90 k
vsphere_virtual_machine.kubernetes_controller (remote-exec):  rpcbind
vsphere_virtual_machine.kubernetes_controller (remote-exec):        x86_64 0.2.0-47.el7   base  60 k
vsphere_virtual_machine.kubernetes_controller (remote-exec):  socat x86_64 1.7.3.2-2.el7  base 290 k
vsphere_virtual_machine.kubernetes_controller (remote-exec):  tcp_wrappers
vsphere_virtual_machine.kubernetes_controller (remote-exec):        x86_64 7.6-77.el7     base  78 k

vsphere_virtual_machine.kubernetes_controller (remote-exec): Transaction Summary
vsphere_virtual_machine.kubernetes_controller (remote-exec): ========================================
vsphere_virtual_machine.kubernetes_controller (remote-exec): Install  5 Packages (+19 Dependent packages)

vsphere_virtual_machine.kubernetes_controller (remote-exec): Total download size: 52 M
vsphere_virtual_machine.kubernetes_controller (remote-exec): Installed size: 237 M
vsphere_virtual_machine.kubernetes_controller (remote-exec): Downloading packages:
vsphere_virtual_machine.kubernetes_controller (remote-exec): (1/24): keyutils-1 |  54 kB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): (2/24): gssproxy-0 | 109 kB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): (4/24): 5af5ecd 7% | 4.1 MB   --:-- ETA
vsphere_virtual_machine.kubernetes_controller (remote-exec): (4/24): 5af5ec 14% | 7.4 MB   00:12 ETA
vsphere_virtual_machine.kubernetes_controller (remote-exec): warning: /var/cache/yum/x86_64/7/kubernetes/packages/5af5ecd0bc46fca6c51cc23280f0c0b1522719c282e23a2b1c39b8e720195763-kubeadm-1.13.1-0.x86_64.rpm: Header V4 RSA/SHA512 Signature, key ID 3e1ba8d5: NOKEY
vsphere_virtual_machine.kubernetes_controller (remote-exec): Public key for 5af5ecd0bc46fca6c51cc23280f0c0b1522719c282e23a2b1c39b8e720195763-kubeadm-1.13.1-0.x86_64.rpm is not installed
vsphere_virtual_machine.kubernetes_controller (remote-exec): (3/24): 5af5ecd0bc | 7.9 MB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): (5/24): 785531 21% |  11 MB   00:10 ETA
vsphere_virtual_machine.kubernetes_controller (remote-exec): (4/24): 53edc739a0 | 4.2 MB   00:01
vsphere_virtual_machine.kubernetes_controller (remote-exec): (5/24): 785531 31% |  17 MB   00:07 ETA
vsphere_virtual_machine.kubernetes_controller (remote-exec): (5/24): 785531 39% |  20 MB   00:06 ETA
vsphere_virtual_machine.kubernetes_controller (remote-exec): (5/24): 785531 46% |  24 MB   00:04 ETA
vsphere_virtual_machine.kubernetes_controller (remote-exec): (5/24): 7855313ff2 | 8.5 MB   00:01
vsphere_virtual_machine.kubernetes_controller (remote-exec): (6/24): libbasicob |  26 kB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): (7/24): libevent-2 | 214 kB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): (8/24): libini_con |  64 kB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): (9/24): libnfsidma |  50 kB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): (10/24): libcollec |  42 kB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): (11/24): libpath_u |  28 kB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): (12/24): libref_ar |  27 kB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): (13/24): libtirpc- |  89 kB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): (14/24): libverto- | 8.9 kB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): (15/24): make-3.82 | 420 kB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): (16/24): 25cd9 58% |  30 MB   00:03 ETA
vsphere_virtual_machine.kubernetes_controller (remote-exec): (16/24): openssl-1 | 493 kB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): (17/24): quota-4.0 | 179 kB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): (18/24): quota-nls |  90 kB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): (19/24): nfs-utils | 410 kB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): (20/24): rpcbind-0 |  60 kB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): (21/24): tcp_wrapp |  78 kB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): (23/24): fe330 67% |  35 MB   00:02 ETA
vsphere_virtual_machine.kubernetes_controller (remote-exec): (22/24): socat-1.7 | 290 kB   00:00
vsphere_virtual_machine.kubernetes_controller (remote-exec): (24/24): fe330 75% |  39 MB   00:01 ETA
vsphere_virtual_machine.kubernetes_controller (remote-exec): (24/24): fe330 83% |  44 MB   00:01 ETA
vsphere_virtual_machine.kubernetes_controller (remote-exec): (24/24): fe330 91% |  48 MB   00:00 ETA
vsphere_virtual_machine.kubernetes_controller (remote-exec): (23/24): 25cd948f6 |  21 MB   00:02
vsphere_virtual_machine.kubernetes_controller (remote-exec): (24/24): fe33057ff | 8.6 MB   00:01
vsphere_virtual_machine.kubernetes_controller (remote-exec): ----------------------------------------
vsphere_virtual_machine.kubernetes_controller (remote-exec): Total       12 MB/s |  52 MB  00:04
vsphere_virtual_machine.kubernetes_controller (remote-exec): Retrieving key from https://packages.cloud.google.com/yum/doc/yum-key.gpg
vsphere_virtual_machine.kubernetes_controller (remote-exec): Importing GPG key 0xA7317B0F:
vsphere_virtual_machine.kubernetes_controller (remote-exec):  Userid     : "Google Cloud Packages Automatic Signing Key <gc-team@google.com>"
vsphere_virtual_machine.kubernetes_controller (remote-exec):  Fingerprint: d0bc 747f d8ca f711 7500 d6fa 3746 c208 a731 7b0f
vsphere_virtual_machine.kubernetes_controller (remote-exec):  From       : https://packages.cloud.google.com/yum/doc/yum-key.gpg
vsphere_virtual_machine.kubernetes_controller (remote-exec): Retrieving key from https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
vsphere_virtual_machine.kubernetes_controller (remote-exec): Importing GPG key 0x3E1BA8D5:
vsphere_virtual_machine.kubernetes_controller (remote-exec):  Userid     : "Google Cloud Packages RPM Signing Key <gc-team@google.com>"
vsphere_virtual_machine.kubernetes_controller (remote-exec):  Fingerprint: 3749 e1ba 95a8 6ce0 5454 6ed2 f09c 394c 3e1b a8d5
vsphere_virtual_machine.kubernetes_controller (remote-exec):  From       : https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
vsphere_virtual_machine.kubernetes_controller (remote-exec): Running transaction check
vsphere_virtual_machine.kubernetes_controller (remote-exec): Running transaction test
vsphere_virtual_machine.kubernetes_controller (remote-exec): Transaction test succeeded
vsphere_virtual_machine.kubernetes_controller (remote-exec): Running transaction
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Verifying  : libbasicobjects-    1/24
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Verifying  : kubelet-1.13.1-0    2/24
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Verifying  : 1:openssl-1.0.2k    3/24
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Verifying  : kubectl-1.13.1-0    4/24
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Verifying  : libini_config-1.    5/24
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Verifying  : kubeadm-1.13.1-0    6/24
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Verifying  : 1:quota-nls-4.01    7/24
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Verifying  : cri-tools-1.12.0    8/24
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Verifying  : libpath_utils-0.    9/24
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Verifying  : libevent-2.0.21-   10/24
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Verifying  : 1:nfs-utils-1.3.   11/24
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Verifying  : gssproxy-0.7.0-2   12/24
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Verifying  : libverto-libeven   13/24
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Verifying  : libcollection-0.   14/24
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Verifying  : 1:make-3.82-23.e   15/24
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Verifying  : libref_array-0.1   16/24
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Verifying  : 1:quota-4.01-17.   17/24
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Verifying  : rpcbind-0.2.0-47   18/24
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Verifying  : libnfsidmap-0.25   19/24
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Verifying  : keyutils-1.5.8-3   20/24
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Verifying  : tcp_wrappers-7.6   21/24
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Verifying  : kubernetes-cni-0   22/24
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Verifying  : socat-1.7.3.2-2.   23/24
vsphere_virtual_machine.kubernetes_controller (remote-exec):   Verifying  : libtirpc-0.2.4-0   24/24

vsphere_virtual_machine.kubernetes_controller (remote-exec): Installed:
vsphere_virtual_machine.kubernetes_controller (remote-exec):   kubeadm.x86_64 0:1.13.1-0
vsphere_virtual_machine.kubernetes_controller (remote-exec):   kubectl.x86_64 0:1.13.1-0
vsphere_virtual_machine.kubernetes_controller (remote-exec):   kubelet.x86_64 0:1.13.1-0
vsphere_virtual_machine.kubernetes_controller (remote-exec):   nfs-utils.x86_64 1:1.3.0-0.61.el7
vsphere_virtual_machine.kubernetes_controller (remote-exec):   openssl.x86_64 1:1.0.2k-16.el7

vsphere_virtual_machine.kubernetes_controller (remote-exec): Dependency Installed:
vsphere_virtual_machine.kubernetes_controller (remote-exec):   cri-tools.x86_64 0:1.12.0-0
vsphere_virtual_machine.kubernetes_controller (remote-exec):   gssproxy.x86_64 0:0.7.0-21.el7
vsphere_virtual_machine.kubernetes_controller (remote-exec):   keyutils.x86_64 0:1.5.8-3.el7
vsphere_virtual_machine.kubernetes_controller (remote-exec):   kubernetes-cni.x86_64 0:0.6.0-0
vsphere_virtual_machine.kubernetes_controller (remote-exec):   libbasicobjects.x86_64 0:0.1.1-32.el7
vsphere_virtual_machine.kubernetes_controller (remote-exec):   libcollection.x86_64 0:0.7.0-32.el7
vsphere_virtual_machine.kubernetes_controller (remote-exec):   libevent.x86_64 0:2.0.21-4.el7
vsphere_virtual_machine.kubernetes_controller (remote-exec):   libini_config.x86_64 0:1.3.1-32.el7
vsphere_virtual_machine.kubernetes_controller (remote-exec):   libnfsidmap.x86_64 0:0.25-19.el7
vsphere_virtual_machine.kubernetes_controller (remote-exec):   libpath_utils.x86_64 0:0.2.1-32.el7
vsphere_virtual_machine.kubernetes_controller (remote-exec):   libref_array.x86_64 0:0.1.5-32.el7
vsphere_virtual_machine.kubernetes_controller (remote-exec):   libtirpc.x86_64 0:0.2.4-0.15.el7
vsphere_virtual_machine.kubernetes_controller (remote-exec):   libverto-libevent.x86_64 0:0.2.5-4.el7
vsphere_virtual_machine.kubernetes_controller (remote-exec):   make.x86_64 1:3.82-23.el7
vsphere_virtual_machine.kubernetes_controller (remote-exec):   quota.x86_64 1:4.01-17.el7
vsphere_virtual_machine.kubernetes_controller (remote-exec):   quota-nls.noarch 1:4.01-17.el7
vsphere_virtual_machine.kubernetes_controller (remote-exec):   rpcbind.x86_64 0:0.2.0-47.el7
vsphere_virtual_machine.kubernetes_controller (remote-exec):   socat.x86_64 0:1.7.3.2-2.el7
vsphere_virtual_machine.kubernetes_controller (remote-exec):   tcp_wrappers.x86_64 0:7.6-77.el7

vsphere_virtual_machine.kubernetes_controller (remote-exec): Complete!
vsphere_virtual_machine.kubernetes_controller (remote-exec): Created symlink from /etc/systemd/system/multi-user.target.wants/kubelet.service to /etc/systemd/system/kubelet.service.
vsphere_virtual_machine.kubernetes_controller (remote-exec): * Applying /usr/lib/sysctl.d/00-system.conf ...
vsphere_virtual_machine.kubernetes_controller (remote-exec): net.bridge.bridge-nf-call-ip6tables = 0
vsphere_virtual_machine.kubernetes_controller (remote-exec): net.bridge.bridge-nf-call-iptables = 0
vsphere_virtual_machine.kubernetes_controller (remote-exec): net.bridge.bridge-nf-call-arptables = 0
vsphere_virtual_machine.kubernetes_controller (remote-exec): * Applying /usr/lib/sysctl.d/10-default-yama-scope.conf ...
vsphere_virtual_machine.kubernetes_controller (remote-exec): kernel.yama.ptrace_scope = 0
vsphere_virtual_machine.kubernetes_controller (remote-exec): * Applying /usr/lib/sysctl.d/50-default.conf ...
vsphere_virtual_machine.kubernetes_controller (remote-exec): kernel.sysrq = 16
vsphere_virtual_machine.kubernetes_controller (remote-exec): kernel.core_uses_pid = 1
vsphere_virtual_machine.kubernetes_controller (remote-exec): net.ipv4.conf.default.rp_filter = 1
vsphere_virtual_machine.kubernetes_controller (remote-exec): net.ipv4.conf.all.rp_filter = 1
vsphere_virtual_machine.kubernetes_controller (remote-exec): net.ipv4.conf.default.accept_source_route = 0
vsphere_virtual_machine.kubernetes_controller (remote-exec): net.ipv4.conf.all.accept_source_route = 0
vsphere_virtual_machine.kubernetes_controller (remote-exec): net.ipv4.conf.default.promote_secondaries = 1
vsphere_virtual_machine.kubernetes_controller (remote-exec): net.ipv4.conf.all.promote_secondaries = 1
vsphere_virtual_machine.kubernetes_controller (remote-exec): fs.protected_hardlinks = 1
vsphere_virtual_machine.kubernetes_controller (remote-exec): fs.protected_symlinks = 1
vsphere_virtual_machine.kubernetes_controller (remote-exec): * Applying /etc/sysctl.d/99-sysctl.conf ...
vsphere_virtual_machine.kubernetes_controller (remote-exec): * Applying /etc/sysctl.d/k8s.conf ...
vsphere_virtual_machine.kubernetes_controller (remote-exec): net.bridge.bridge-nf-call-ip6tables = 1
vsphere_virtual_machine.kubernetes_controller (remote-exec): net.bridge.bridge-nf-call-iptables = 1
vsphere_virtual_machine.kubernetes_controller (remote-exec): * Applying /etc/sysctl.conf ...
{{< /highlight >}}
Finally the `./script/kubeadm_init.sh` script will be remotely run that will pull down the kubernetes docker images, perform a `kubeadm init` in stand up the kubernetes controller and configure the flannel network componants:
{{< highlight bash >}}
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> pull kubeadm images <--
vsphere_virtual_machine.kubernetes_controller (remote-exec): [config/images] Pulled k8s.gcr.io/kube-apiserver:v1.13.1
vsphere_virtual_machine.kubernetes_controller: Still creating... (3m20s elapsed)
vsphere_virtual_machine.kubernetes_controller (remote-exec): [config/images] Pulled k8s.gcr.io/kube-controller-manager:v1.13.1
vsphere_virtual_machine.kubernetes_controller (remote-exec): [config/images] Pulled k8s.gcr.io/kube-scheduler:v1.13.1
vsphere_virtual_machine.kubernetes_controller: Still creating... (3m30s elapsed)
vsphere_virtual_machine.kubernetes_controller (remote-exec): [config/images] Pulled k8s.gcr.io/kube-proxy:v1.13.1
vsphere_virtual_machine.kubernetes_controller (remote-exec): [config/images] Pulled k8s.gcr.io/pause:3.1
vsphere_virtual_machine.kubernetes_controller (remote-exec): [config/images] Pulled k8s.gcr.io/etcd:3.2.24
vsphere_virtual_machine.kubernetes_controller: Still creating... (3m40s elapsed)
vsphere_virtual_machine.kubernetes_controller (remote-exec): [config/images] Pulled k8s.gcr.io/coredns:1.2.6
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> run 'kubeadm init' <--
vsphere_virtual_machine.kubernetes_controller: Still creating... (3m50s elapsed)
vsphere_virtual_machine.kubernetes_controller: Still creating... (4m0s elapsed)
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> setup /root/.kube/config <--
vsphere_virtual_machine.kubernetes_controller (remote-exec): --> install flannel <--
vsphere_virtual_machine.kubernetes_controller (remote-exec): clusterrole.rbac.authorization.k8s.io/flannel created
vsphere_virtual_machine.kubernetes_controller (remote-exec): clusterrolebinding.rbac.authorization.k8s.io/flannel created
vsphere_virtual_machine.kubernetes_controller (remote-exec): serviceaccount/flannel created
vsphere_virtual_machine.kubernetes_controller (remote-exec): configmap/kube-flannel-cfg created
vsphere_virtual_machine.kubernetes_controller (remote-exec): daemonset.extensions/kube-flannel-ds-amd64 created
vsphere_virtual_machine.kubernetes_controller (remote-exec): daemonset.extensions/kube-flannel-ds-arm64 created
vsphere_virtual_machine.kubernetes_controller (remote-exec): daemonset.extensions/kube-flannel-ds-arm created
vsphere_virtual_machine.kubernetes_controller (remote-exec): daemonset.extensions/kube-flannel-ds-ppc64le created
vsphere_virtual_machine.kubernetes_controller (remote-exec): daemonset.extensions/kube-flannel-ds-s390x created
vsphere_virtual_machine.kubernetes_controller (remote-exec):   kubeadm join 192.168.100.100:6443 --token 4xr954.lv1sp4plkapb0eza --discovery-token-ca-cert-hash sha256:0d972b6d7e08ebef346a8cf14ea4bcf88b9341351d7ec4d488defdbb4761e5e8
vsphere_virtual_machine.kubernetes_controller: Creation complete after 4m7s (ID: 42210435-ffe9-b519-6191-10dae7698a78)

Apply complete! Resources: 2 added, 0 changed, 0 destroyed.

Outputs:

ip = [
    192.168.100.100
]
vm-moref = vm-1131
[root@terraform terraform-vsphere-kubernetes]#
{{< /highlight >}}

The bottom of the `terraform appy` displays the IP address and moref of the vm that was specified in the `output.tf` file. I also want to point out the output returned in the last `remote-exec` block:

kubeadm join 192.168.100.100:6443 --token 4xr954.lv1sp4plkapb0eza --discovery-token-ca-cert-hash sha256:0d972b6d7e08ebef346a8cf14ea4bcf88b9341351d7ec4d488defdbb4761e5e8

This is the `kubeadm join` command that can be run on any kubernetes node to join them to the kubernetes controller we initialized.

### 11. Validate `kubectl` is working by connecting to the deployed virtual machine.

You can no longer ssh to the deployed virtual machine using the password from the template, but you can use the ssh key generated to connect without a password like in the following example: 

{{< highlight bash >}}
[root@terraform terraform-vsphere-kubernetes]# ssh root@192.168.100.100 -i /root/.ssh/id_rsa-terraform-vsphere-kubernetes
The authenticity of host '192.168.100.100 (192.168.100.100)' can't be established.
ECDSA key fingerprint is SHA256:dpAptyG7C1yv/tiAT2FU6t/nI9EYJcrD0uNjNr1SldM.
ECDSA key fingerprint is MD5:3e:2e:2f:86:57:34:fc:9b:b6:ff:ff:42:df:8b:33:fe.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '192.168.100.100' (ECDSA) to the list of known hosts.
Last login: Thu Dec 27 22:22:14 2018 from 192.168.1.8
[root@kubernetes-controller ~]# kubectl get nodes
NAME                    STATUS   ROLES    AGE   VERSION
kubernetes-controller   Ready    master   5m    v1.13.1
[root@kubernetes-controller ~]# exit
logout
Connection to 192.168.100.100 closed.
[root@terraform terraform-vsphere-kubernetes]#
{{< /highlight >}}

The output of `kubectl get nodes` shows that the controller node is in a `Ready` state.

### 12. Run `terraform destroy` to delete the resources that were previously created.

Once you are ready to destroy the virtual machine terraform created, you can simply run `terraform destroy --force` to have terraform destroy it:

{{< highlight bash >}}
[root@terraform terraform-vsphere-kubernetes]# terraform destroy --force
null_resource.generate-sshkey: Refreshing state... (ID: 8831700834679792902)
data.vsphere_datacenter.template_datacenter: Refreshing state...
data.vsphere_datastore.vm_datastore: Refreshing state...
data.vsphere_network.vm_network: Refreshing state...
data.vsphere_virtual_machine.template: Refreshing state...
data.vsphere_resource_pool.vm_resource_pool: Refreshing state...
vsphere_virtual_machine.kubernetes_controller: Refreshing state... (ID: 42210435-ffe9-b519-6191-10dae7698a78)
null_resource.generate-sshkey: Destroying... (ID: 8831700834679792902)
null_resource.generate-sshkey: Destruction complete after 0s
vsphere_virtual_machine.kubernetes_controller: Destroying... (ID: 42210435-ffe9-b519-6191-10dae7698a78)
vsphere_virtual_machine.kubernetes_controller: Still destroying... (ID: 42210435-ffe9-b519-6191-10dae7698a78, 10s elapsed)
vsphere_virtual_machine.kubernetes_controller: Still destroying... (ID: 42210435-ffe9-b519-6191-10dae7698a78, 20s elapsed)
vsphere_virtual_machine.kubernetes_controller: Destruction complete after 30s

Destroy complete! Resources: 2 destroyed.
[root@terraform terraform-vsphere-kubernetes]#
{{< /highlight >}}

#### I hope this post has been useful in demonstrating how to use `local-exec` and `remote-exec` provisioners to configure virtual machines deployed by terraform. The files that were using in this post can be found in <a href="https://github.com/sdorsett/terraform-vsphere-kubernetes">this github repository</a>.

---

#### Please provide any feedback or suggestions to my twitter account located on the about page.
