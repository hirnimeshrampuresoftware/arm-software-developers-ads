---
# User change
title: "Install Redis on a single GCP Arm based instance"

weight: 5 # 1 is first, 2 is second, etc.

# Do not modify these elements
layout: "learningpathall"
---

## Before you begin

Any computer which has the required tools installed can be used for this section. 

You will need a [Google cloud account](https://console.cloud.google.com/?hl=en-au). Create an account if needed.

Following tools are required on the computer you are using. Follow the links to install the required tools.
* [Google Cloud CLI](https://cloud.google.com/sdk/docs/install-sdk#deb)
* [Ansible](https://www.cyberciti.biz/faq/how-to-install-and-configure-latest-version-of-ansible-on-ubuntu-linux/)
* [Terraform](/install-tools/terraform)
* [Redis CLI](https://redis.io/docs/getting-started/installation/install-redis-on-linux/)

## Deploy GCP Arm based instance via terraform

Before deploying GCP Arm based instance [Login to Google Cloud CLI](/content/learning-paths/server-and-cloud/gcp/terraform.md#acquire-user-credentials) and [Generate key-pair using ssh keygen](/content/learning-paths/server-and-cloud/redis/aws_deployment.md#generate-key-pairpublic-key-private-key-using-ssh-keygen).

## Terraform infrastructure

Add resources required to create a virtual machine in **main.tf** file.
```
provider "google" {
  project = "{project_id}"
  region = "us-central1"
  zone = "us-central1-a"
}

resource "google_compute_project_metadata_item" "ssh-keys" {
  key   = "ssh-keys"
  value = "ubuntu:${file("{public_key_location}")}"
}

resource "google_compute_firewall" "rules" {
  project     = "{project_id}"
  name        = "my-firewall-rule"
  network     = "default"
  description = "Open Redis connection port"
  source_ranges = ["0.0.0.0/0"]

  allow {
    protocol  = "tcp"
    ports     = ["6379"]
  }
}

resource "google_compute_instance" "vm_instance" {
  name         = "vm_name"
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

resource "local_file" "inventory" {
    depends_on=[google_compute_instance.vm_instance]
    filename = "inventory.txt"
    content = <<EOF
[all]
ansible-target1 ansible_connection=ssh ansible_host=${google_compute_instance.vm_instance.network_interface.0.access_config.0.nat_ip} ansible_user=ubuntu
                EOF
}
```
**NOTE:-** Replace **{project_id}** and **{public_key_location}** with respective values.

## Terraform commands

To deploy the instances, we need to initialize Terraform, generate an execution plan and apply the execution plan to our cloud infrastructure. Follow this [documentation](/content/learning-paths/server-and-cloud/redis/aws_deployment.md#terraform-commands) to deploy the **main.tf** file.

## Install Redis using Ansible

To install Redis using Ansible follow this [documentation](/content/learning-paths/server-and-cloud/redis/aws_deployment.md#install-redis-using-ansible).

## Connecting to Redis server from local machine

Follow this [documentation](/content/learning-paths/server-and-cloud/redis/aws_deployment.md#connecting-to-redis-server-from-local-machine) to connect to the remote Redis server from our local machine.
