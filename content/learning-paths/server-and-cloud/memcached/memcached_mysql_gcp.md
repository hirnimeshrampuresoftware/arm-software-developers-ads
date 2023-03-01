---
# User change
title: "Deploy Memcached as a cache for MySQL on Google Cloud Arm based Instance"

weight: 5 # 1 is first, 2 is second, etc.

# Do not modify these elements
layout: "learningpathall"
---

## Before you begin

Any computer which has the required tools installed can be used for this section. 

You will need a [Google Cloud account](https://console.cloud.google.com/). Create an account if needed.

Following tools are required on the computer you are using. Follow the links to install the required tools.
* [Google Cloud CLI](https://cloud.google.com/sdk/docs/install-sdk#deb)
* [Ansible](https://www.cyberciti.biz/faq/how-to-install-and-configure-latest-version-of-ansible-on-ubuntu-linux/)
* [Terraform](/install-tools/terraform)
* [Python](https://beebom.com/how-install-python-ubuntu-linux/)
* [Memcached](/learning-paths/server-and-cloud/memcached/memcached#install-memcached-from-source-on-arm-servers)
* [Telnet](https://adamtheautomator.com/linux-to-install-telnet/)

## Deploy MySQL instances via Terraform

### Acquire user credentials
To obtain user access credentials, follow this [documentation](/learning-paths/server-and-cloud/gcp/terraform#acquire-user-credentials).

### Generate key-pair (public key, private key)
Before using Terraform, first generate the key-pair (public key and private key) using ssh-keygen. Then associate both public and private keys with Arm VMs. To generate the key-pair, follow this [documentation](/learning-paths/server-and-cloud/gcp/terraform#generate-key-pairpublic-key-private-key-using-ssh-keygen).

### Create Terraform file (main.tf)
After generating the keys, we have to create the MySQL instances. Then we will push our public key to the authorized_keys folder in ~/.ssh. We will also create a security group that opens inbound ports **22** (ssh) and **3306** (MySQL). Below is a Terraform file called main.tf that will do this for us. Here we are creating 2 instances.
    
```console
provider "google" {
  project = "snappy-byway-368307"
  region = "us-central1"
  zone = "us-central1-a"
}

resource "google_compute_project_metadata_item" "ssh-keys" {
  key   = "ssh-keys"
  value = "ubuntu:${file("/path/to/public_key.pub")}"
}

resource "google_compute_instance" "MYSQL_TEST" {
  name         = "mysqltest-${count.index+1}"
  count        = "2"
  machine_type = "t2a-standard-1"

  boot_disk {
    initialize_params {
      image = "ubuntu-os-cloud/ubuntu-2204-lts-arm64"
    }
  }

  network_interface {
    network = "default"
    access_config {
      // Ephemeral public IP
    }
  }
}
resource "google_compute_firewall" "default" {
  name    = "test-firewall"
  network = google_compute_network.default.name

  allow {
    protocol = "icmp"
  }

  allow {
    protocol = "tcp"
    ports    = ["22", "3306"]
  }

  source_tags = ["web"]
}

resource "google_compute_network" "default" {
  name = "test-network1"
}
resource "local_file" "inventory" {
    depends_on=[google_compute_instance.MYSQL_TEST]
    filename = "inventory.txt"
    content = <<EOF
[mysql1]
${google_compute_instance.MYSQL_TEST[0].network_interface.0.access_config.0.nat_ip}
[mysql2]
${google_compute_instance.MYSQL_TEST[1].network_interface.0.access_config.0.nat_ip}
[all:vars]
ansible_connection=ssh
ansible_user=ubuntu
                EOF
}
```
**NOTE**:- Replace the path of **public_key** with its respective value.

### Terraform Commands
To deploy the instances, we need to initialize Terraform, generate an execution plan and apply the execution plan to our cloud infrastructure. Follow this [documentation](/learning-paths/server-and-cloud/gcp/terraform#terraform-commands) to deploy the **main.tf** file.

## Configure MySQL through Ansible
An Ansible Playbook installs & enables MySQL in the instances and creates databases & tables inside them. To configure MySQL through Ansible and run the Playbook, follow this [documentation](/learning-paths/server-and-cloud/memcached/memcached_mysql_aws#configure-mysql-through-ansible).

## Deploy Memcached as a cache for MySQL using Python
To deploy Memcached as a cache for MySQL using Python, follow this [documentation](/learning-paths/server-and-cloud/memcached/memcached_mysql_aws#deploy-memcached-as-a-cache-for-mysql-using-python).
