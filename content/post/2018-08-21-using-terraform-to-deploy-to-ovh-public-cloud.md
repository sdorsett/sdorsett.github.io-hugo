+++
title = "Using Terraform to deploy an OVH public cloud server."
description = ""
tags = [
    "terraform",
    "openstack",
    "OVH",
]
date = "2018-08-21"
categories = [
    "OVH",
    "openstack",
    "terraform",
]
topics = [
    "using-ovh-openstack-public-cloud"
]
highlight = "true"
+++

This is the fifth in a series of posts that will walk you through using the Openstack-based OVH public cloud. In this post you will get introduced to using Terraform to create Openstack servers using the Openstack API.
This post is assumes you have already signed up for an account with ovhcloud.com, added a payment method, created a cloud project, created an Openstack user, created a ssh keypair in the Horizon UI and downloaded the openrc.sh file for this user. If you have not done these steps you can follow the steps in <a href="../2018-08-08-creating-an-openstack-instance-on-ovh-public-cloud">the first</a> and <a href="../2018-08-10-using-the-openstack-cli-to-create-a-server-on-ovh-public-cloud">second blog post</a> in this series that will walk you through completing those steps.

I will attempt in this post to present the options that are available to OVH public cloud customers along side the choices I made that were specific to my Openstack server.
Let's get started...

### 1. Install Terraform

The first thing you will need to do before you can move forward with this post is install Terraform. Go to https://www.terraform.io/downloads.html, download the binary for your specific operating system, unzip the binary and copy it to a directory that is in your path. Here are the steps that I performed on my Mac laptop:

{{< highlight bash >}}
MacBook-Pro:~ standorsett$ cd ~/Downloads/
MacBook-Pro:Downloads standorsett$ wget https://releases.hashicorp.com/terraform/0.11.8/terraform_0.11.8_darwin_amd64.zip
--2018-08-21 19:12:36--  https://releases.hashicorp.com/terraform/0.11.8/terraform_0.11.8_darwin_amd64.zip
Resolving releases.hashicorp.com (releases.hashicorp.com)... 151.101.125.183
Connecting to releases.hashicorp.com (releases.hashicorp.com)|151.101.125.183|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 19269525 (18M) [application/zip]
Saving to: ‘terraform_0.11.8_darwin_amd64.zip’

terraform_0.11.8_darwin_amd64.zip            100%[=============================================================================================>]  18.38M  1.45MB/s    in 21s

2018-08-21 19:12:57 (900 KB/s) - ‘terraform_0.11.8_darwin_amd64.zip’ saved [19269525/19269525]

MacBook-Pro:Downloads standorsett$ unzip terraform_0.11.8_darwin_amd64.zip
Archive:  terraform_0.11.8_darwin_amd64.zip
  inflating: terraform
MacBook-Pro:Downloads standorsett$ mv terraform /usr/local/bin/
MacBook-Pro:Downloads standorsett$ terraform --version
Terraform v0.11.8

MacBook-Pro:Downloads standorsett$
{{< /highlight >}}

### 2. Create a new directory and add your main.tf file

When using the `terraform` command, it will look for any files ending in .tf in the current directory and create them. As a result you should create a new directory for containing the terraform files we will use for creating the Openstack server.

{{< highlight bash >}}
MacBook-Pro:~ standorsett$ mkdir -p ~/Documents/terraform/ovh_public_cloud_centos_7
MacBook-Pro:~ standorsett$ cd ~/Documents/terraform/ovh_public_cloud_centos_7
MacBook-Pro:ovh_public_cloud_centos_7 standorsett$
{{< /highlight >}}

Next you will need to create a file named main.tf in our new directory that will describe the server you are wanting to have created.

{{< highlight bash >}}
MacBook-Pro:ovh_public_cloud_centos_7 standorsett$ cat main.tf
# Configure the OpenStack Provider
provider "openstack" {
#  user_name is unset so it defaults to OS_USERNAME environment variable
#  tenant_name is unset so it defaults to OS_TENANT_NAME or OS_PROJECT_NAME environment variable
#  password is unset so it defaults to OS_PASSWORD environment variable
#  auth_url is unset so it defaults to OS_AUTH_URL environment variable
#  region is unset so it defaults to OS_REGION_NAME environment variable
}

# get an image named "Centos 7"
data "openstack_images_image_v2" "centos_7" {
  name = "Centos 7"
  most_recent = true
}

# get a flavor named "s1-2"
data "openstack_compute_flavor_v2" "s1-2" {
  name = "s1-2"
}

resource "openstack_compute_instance_v2" "basic" {
  name            = "terraform_centos_7"
  image_id        = "${data.openstack_images_image_v2.centos_7.id}"
  flavor_id       = "${data.openstack_compute_flavor_v2.s1-2.id}"
  key_pair        = "vagrant-keypair"                      # ssh keypair name
  security_groups = ["default"]

  network {
    name = "Ext-Net"
  }
}
MacBook-Pro:ovh_public_cloud_centos_7 standorsett$
{{< /highlight >}}

The `provider.keypair_name` should be set to match the name of the keypair you created in the Openstack Horizon UI in the second blog post. The `provider.image` and `provider.flavor` values listed above are the values I used with my testing.

You also might notice that all of the details in the provider block of main.tf are commented out which means they will be pulled from environment variables set in your openrc.sh file.

### 3. Source your openrc.sh file

Source the openrc.sh you previously downloaded for your Openstack user. Enter the password for your Openstack user when prompted. This step will export the environmental variables that the Vagrantfile needs in order to create the Openstack server.

{{< highlight bash >}}
MacBook-Pro:ovh_public_cloud_centos_7 standorsett$ source ~/Downloads/openrc.sh
Please enter your OpenStack Password:
MacBook-Pro:ovh_public_cloud_centos_7 standorsett$
{{< /highlight >}}

### 4. Download all needed Terraform providers by running `terraform init`

Terraform will automatically download all Terraform providers referenced by any .tf files by running `terraform init`. Here is the output from when I ran this command.

{{< highlight bash >}}
MacBook-Pro:ovh_public_cloud_centos_7 standorsett$
Initializing provider plugins...

The following providers do not have any version constraints in configuration,
so the latest version was installed.

To prevent automatic upgrades to new major versions that may contain breaking
changes, it is recommended to add version = "..." constraints to the
corresponding provider blocks in configuration, with the constraint strings
suggested below.

* provider.openstack: version = "~> 1.8"

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
MacBook-Pro:ovh_public_cloud_centos_7 standorsett$
{{< /highlight >}}

### 5. View all the proposed Terraform changes by running `terraform plan`

By running `terraform plan` Terrraform will output all the details of the resources that will be created when commanded to apply the changes.

{{< highlight bash >}}
MacBook-Pro:ovh_public_cloud_centos_7 standorsett$
Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.

data.openstack_images_image_v2.centos_7: Refreshing state...
data.openstack_compute_flavor_v2.s1-2: Refreshing state...

------------------------------------------------------------------------

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  + openstack_compute_instance_v2.basic
      id:                         <computed>
      access_ip_v4:               <computed>
      access_ip_v6:               <computed>
      all_metadata.%:             <computed>
      availability_zone:          <computed>
      flavor_id:                  "a64381e7-c4e7-4b01-9fbe-da405c544d2e"
      flavor_name:                <computed>
      force_delete:               "false"
      image_id:                   "e0a89ff2-e98f-4a34-afae-68f9f6c9b5ad"
      image_name:                 <computed>
      key_pair:                   "vagrant-keypair"
      name:                       "terraform_centos_7"
      network.#:                  "1"
      network.0.access_network:   "false"
      network.0.fixed_ip_v4:      <computed>
      network.0.fixed_ip_v6:      <computed>
      network.0.floating_ip:      <computed>
      network.0.mac:              <computed>
      network.0.name:             "Ext-Net"
      network.0.port:             <computed>
      network.0.uuid:             <computed>
      power_state:                "active"
      region:                     <computed>
      security_groups.#:          "1"
      security_groups.3814588639: "default"
      stop_before_destroy:        "false"


Plan: 1 to add, 0 to change, 0 to destroy.

------------------------------------------------------------------------

Note: You didn't specify an "-out" parameter to save this plan, so Terraform
can't guarantee that exactly these actions will be performed if
"terraform apply" is subsequently run.
MacBook-Pro:ovh_public_cloud_centos_7 standorsett$
{{< /highlight >}}

### 6. Apply the proposed Terraform changes by running `terraform apply -auto-approve`

After verifying that the proposed change outputed by `terraform plan` look good, apply those changes by running `terraform apply -auto-approve`.

{{< highlight bash >}}
MacBook-Pro:ovh_public_cloud_centos_7 standorsett$ terraform apply -auto-approve
data.openstack_images_image_v2.centos_7: Refreshing state...
data.openstack_compute_flavor_v2.s1-2: Refreshing state...
openstack_compute_instance_v2.basic: Creating...
  access_ip_v4:               "" => "<computed>"
  access_ip_v6:               "" => "<computed>"
  all_metadata.%:             "" => "<computed>"
  availability_zone:          "" => "<computed>"
  flavor_id:                  "" => "a64381e7-c4e7-4b01-9fbe-da405c544d2e"
  flavor_name:                "" => "<computed>"
  force_delete:               "" => "false"
  image_id:                   "" => "e0a89ff2-e98f-4a34-afae-68f9f6c9b5ad"
  image_name:                 "" => "<computed>"
  key_pair:                   "" => "vagrant-keypair"
  name:                       "" => "terraform_centos_7"
  network.#:                  "" => "1"
  network.0.access_network:   "" => "false"
  network.0.fixed_ip_v4:      "" => "<computed>"
  network.0.fixed_ip_v6:      "" => "<computed>"
  network.0.floating_ip:      "" => "<computed>"
  network.0.mac:              "" => "<computed>"
  network.0.name:             "" => "Ext-Net"
  network.0.port:             "" => "<computed>"
  network.0.uuid:             "" => "<computed>"
  power_state:                "" => "active"
  region:                     "" => "<computed>"
  security_groups.#:          "" => "1"
  security_groups.3814588639: "" => "default"
  stop_before_destroy:        "" => "false"
openstack_compute_instance_v2.basic: Still creating... (10s elapsed)
openstack_compute_instance_v2.basic: Creation complete after 16s (ID: 130158c7-6e13-4b79-b729-b4250ac82b04)

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

MacBook-Pro:ovh_public_cloud_centos_7 standorsett$
{{< /highlight >}}

### 7. Use `terraform show` to display the configuration details of the deployed Openstack server

After deploying your server with `terraform apply` you can query the configuration details by running `terraform show`. This command will return the details of the data objects queried as well as the server deployed.

{{< highlight bash >}}
MacBook-Pro:ovh_public_cloud_centos_7 standorsett$ terraform show
data.openstack_compute_flavor_v2.s1-2:
  id = a64381e7-c4e7-4b01-9fbe-da405c544d2e
  disk = 10
  is_public = true
  name = s1-2
  ram = 2000
  rx_tx_factor = 1
  swap = 0
  vcpus = 1
data.openstack_images_image_v2.centos_7:
  id = e0a89ff2-e98f-4a34-afae-68f9f6c9b5ad
  checksum = 8abade82e1295d92c7f360f3be1029d4
  container_format = bare
  disk_format = raw
  file = /v2/images/e0a89ff2-e98f-4a34-afae-68f9f6c9b5ad/file
  metadata.% = 0
  min_disk_gb = 0
  min_ram_mb = 0
  most_recent = true
  name = Centos 7
  owner = f0dffa44438a41bb9790b67b57b72dcf
  protected = false
  schema = /v2/schemas/image
  size_bytes = 5242880000
  sort_direction = asc
  sort_key = name
  visibility = public
openstack_compute_instance_v2.basic:
  id = 130158c7-6e13-4b79-b729-b4250ac82b04
  access_ip_v4 = 147.135.76.68
  access_ip_v6 = [2604:2dc0:101:100::42]
  all_metadata.% = 0
  availability_zone = nova
  flavor_id = a64381e7-c4e7-4b01-9fbe-da405c544d2e
  flavor_name = s1-2
  force_delete = false
  image_id = e0a89ff2-e98f-4a34-afae-68f9f6c9b5ad
  image_name = Centos 7
  key_pair = vagrant-keypair
  name = terraform_centos_7
  network.# = 1
  network.0.access_network = false
  network.0.fixed_ip_v4 = 147.135.76.68
  network.0.fixed_ip_v6 = [2604:2dc0:101:100::42]
  network.0.floating_ip =
  network.0.mac = fa:16:3e:ef:3d:4c
  network.0.name = Ext-Net
  network.0.port =
  network.0.uuid = b347ed75-8603-4ce0-a40c-c6c98a8820fc
  power_state = active
  region = US-EAST-VA-1
  security_groups.# = 1
  security_groups.3814588639 = default
  stop_before_destroy = false

MacBook-Pro:ovh_public_cloud_centos_7 standorsett$
{{< /highlight >}}

If you look back at the output of your command you will see a line displaying the `network.0.fixed_ip_v4` IP address that you can use to connect with ssh.

### 8. Connect to the deployed Openstack server using ssh.

You can now connect to the IP address that is displayed with `terraform show` using the private ssh key you created in a previous blog post.

{{< highlight bash >}}
MacBook-Pro:ovh_public_cloud_centos_7 standorsett$ ssh -i ~/.ssh/id_rsa_vagrant-keypair centos@147.135.76.49
The authenticity of host '147.135.76.49 (147.135.76.49)' can't be established.
ECDSA key fingerprint is SHA256:aVSsv0mHKgIizglcvouJkvtBBOiZfNcQTpIc5oxhNO4.
ECDSA key fingerprint is MD5:91:74:c1:46:20:9c:11:0d:07:1d:f5:33:43:a9:11:74.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '147.135.76.49' (ECDSA) to the list of known hosts.
[centos@terraform-centos-7 ~]$ hostname
terraform-centos-7
[centos@terraform-centos-7 ~]$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether fa:16:3e:04:af:e8 brd ff:ff:ff:ff:ff:ff
    inet 147.135.76.49/32 brd 147.135.76.49 scope global dynamic eth0
       valid_lft 86355sec preferred_lft 86355sec
    inet6 fe80::f816:3eff:fe04:afe8/64 scope link
       valid_lft forever preferred_lft forever
[centos@terraform-centos-7 ~]$ exit
logout
Connection to 147.135.76.49 closed.
MacBook-Pro:ovh_public_cloud_centos_7 standorsett$
{{< /highlight >}}

### 9. Destroy your Terraform Openstack server.
After you are finished testing your Openstack server, you should destroy it in order to prevent being charged for it’s usage. This can be done using the `terraform destroy --force` command.

{{< highlight bash >}}
MacBook-Pro:ovh_public_cloud_centos_7 standorsett$ terraform destroy --force
data.openstack_images_image_v2.centos_7: Refreshing state...
data.openstack_compute_flavor_v2.s1-2: Refreshing state...
openstack_compute_instance_v2.basic: Refreshing state... (ID: 130158c7-6e13-4b79-b729-b4250ac82b04)
openstack_compute_instance_v2.basic: Destroying... (ID: 130158c7-6e13-4b79-b729-b4250ac82b04)
openstack_compute_instance_v2.basic: Still destroying... (ID: 130158c7-6e13-4b79-b729-b4250ac82b04, 10s elapsed)
openstack_compute_instance_v2.basic: Destruction complete after 11s

Destroy complete! Resources: 1 destroyed.
MacBook-Pro:ovh_public_cloud_centos_7 standorsett$
{{< /highlight >}}

#### That all for this post covering how to use Terraform to deploy a OVH public cloud server. In this post you have only scratched the surface of what you can do with Terraform, but hopefully if showed how simple it is to get started.

---

#### Please provide any feedback or suggestions to my twitter account located on the about page.

