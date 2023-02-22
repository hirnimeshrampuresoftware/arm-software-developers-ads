---
# User change
title: "Install Redis in a multi-node configuration"

weight: 7 # 1 is first, 2 is second, etc.

# Do not modify these elements
layout: "learningpathall"
---

##  Install Redis in a multi-node configuration

## Prerequisites

* An [AWS account](https://portal.aws.amazon.com/billing/signup?nc2=h_ct&src=default&redirect_url=https%3A%2F%2Faws.amazon.com%2Fregistration-confirmation#/start)
* [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
* [AWS IAM authenticator](https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html)
* [Ansible](https://www.cyberciti.biz/faq/how-to-install-and-configure-latest-version-of-ansible-on-ubuntu-linux/)
* [Terraform](/content/install-tools/terraform.md)
* [Redis CLI](https://redis.io/docs/getting-started/installation/install-redis-on-linux/)


## Deploy AWS Arm based instance via Terraform

Before deploying AWS Arm based instance via Terraform, generate [Access keys](/content/learning-paths/server-and-cloud/redis/aws_deployment.md#generate-access-keys-access-key-id-and-secret-access-key) and [key-pair using ssh keygen](/content/learning-paths/server-and-cloud/redis/aws_deployment.md#generate-key-pairpublic-key-private-key-using-ssh-keygen).

After generating the public and private keys, we will push our public key to the **authorized_keys** folder in `~/.ssh`. We will also create a security group that opens inbound ports `22`(ssh) and `6001-6006` (Redis). Below is a Terraform file named **main.tf** which will do this for us.


```console
provider "aws" {
  region = "us-east-2"
  access_key  = "AXXXXXXXXXXXXXXXXXXX"
  secret_key   = "AAXXXXXXXXXXXXXXXXXXXXXXXXXXX"
}
resource "aws_instance" "redis-deployment" {
  ami = "ami-0bc02c3c09aaee8ea"
  instance_type = "t4g.small"
  key_name= "aws_key"
  vpc_security_group_ids = [aws_security_group.main.id]
}

resource "aws_security_group" "main" {
  name        = "main"
  description = "Allow TLS inbound traffic"

  ingress {
    description      = "Open redis connection ports"
    from_port        = 6001
    to_port          = 6006
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
  }
  ingress {
    description      = "Allow ssh to instance"
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
  }
  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
  }
}

resource "local_file" "inventory" {
    depends_on=[aws_instance.redis-deployment]
    filename = "inventory.txt"
    content = <<EOF
[all]
ansible-target1 ansible_connection=ssh ansible_host=${aws_instance.redis-deployment.public_dns} ansible_user=ubuntu
                EOF
}

resource "aws_key_pair" "deployer" {
        key_name   = "aws_key"
        public_key = "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCUZXm6T6JTQBuxw7aFaH6gmxDnjSOnHbrI59nf+YCHPqIHMlGaxWw0/xlaJiJynjOt67Zjeu1wNPifh2tzdN3UUD7eUFSGcLQaCFBDorDzfZpz4wLDguRuOngnXw+2Z3Iihy2rCH+5CIP2nCBZ+LuZuZ0oUd9rbGy6pb2gLmF89GYzs2RGG+bFaRR/3n3zR5ehgCYzJjFGzI8HrvyBlFFDgLqvI2KwcHwU2iHjjhAt54XzJ1oqevRGBiET/8RVsLNu+6UCHW6HE9r+T5yQZH50nYkSl/QKlxBj0tGHXAahhOBpk0ukwUlfbGcK6SVXmqtZaOuMNlNvssbocdg1KwOH ubuntu@ip-172-31-XXXX-XXXX"
}
```
**NOTE:-** Replace `public_key`, `access_key`, `secret_key`, and `key_name` with actual values.


### Terraform Commands

To deploy the instances, we need to initialize Terraform, generate an execution plan and apply the execution plan to our cloud infrastructure. Follow this [documentation](/content/learning-paths/server-and-cloud/redis/aws_deployment.md#terraform-commands) to deploy the **main.tf** file.

## Install Redis in a multi-node configuration using Ansible
To run Ansible, we have to create a `.yml` file, which is also known as `Ansible-Playbook`. The following playbook contains a collection of tasks which install Redis in multi-node configuration (3 primary and 3 replica nodes). In our example, we have used 6 different ports (`6001-6006`) of the same host. 

Here is the complete **deploy_redis.yml** file of Ansible-Playbook
```console
---
- hosts: all
  become: true
  become_user: root
  remote_user: ubuntu

  tasks:
    - name: Update the Machine
      shell: apt update
    - name: Download redis gpg key
      shell: curl -fsSL https://packages.redis.io/gpg | gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg
      args:
        warn: false
    - name: Add redis gpg key
      shell: echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list
    - name: Update the apt sources
      shell: apt update
    - name: Install redis
      shell: apt install -y redis-tools redis
    - name: Create directories
      file:
        path: "/home/ubuntu/{{item}}"
        state: directory
      with_sequence: start=6001 end=6006
      become_user: ubuntu
    - name: Create configuration files
      copy:
        dest: "/home/ubuntu/{{item}}/redis.conf"
        content: |
          protected-mode no
          port {{item}}
          cluster-enabled yes
          cluster-config-file nodes.conf
          cluster-node-timeout 5000
          daemonize yes
          appendonly yes
      with_sequence: start=6001 end=6006
      become_user: ubuntu
    - name: Start redis server with configuration files
      shell: redis-server redis.conf
      args:
        chdir: "/home/ubuntu/{{item}}"
      with_sequence: start=6001 end=6006
      become_user: ubuntu
    - name: Create Redis cluster with 3 primary and 3 replica nodes
      shell: "(redis-cli --cluster create {{ansible_host}}:6001 {{ansible_host}}:6002 {{ansible_host}}:6003 {{ansible_host}}:6004 {{ansible_host}}:6005 {{ansible_host}}:6006 --cluster-replicas 1 --cluster-yes &>/dev/null &)"
      run_once: true
      become_user: ubuntu
```
**NOTE:-** Since the allocation of primary and replica nodes is random at the time of cluster creation, it is difficult to know whether the nodes at ports `6001-6006` are master or slave nodes. Hence, for the multi-node configuration, we need to turn off **protected-mode**, which is enabled by default, so that we can connect to primary and replica nodes. 

To run a Playbook, we need to use the `ansible-playbook` command.
```console
ansible-playbook {your_yml_file} -i {your_inventory_file} --key-file {path_to_private_key}
```
**NOTE:-** Replace `{your_yml_file}`, `{your_inventory_file}` and `{path_to_private_key}` with orignal values.

![image](https://user-images.githubusercontent.com/90673309/215947178-49a1624f-f7c7-4594-8387-5c899913f611.png)

Here is the output after the successful execution of the `ansible-playbook` command.

![image](https://user-images.githubusercontent.com/90673309/215947195-d4186c77-afcd-4d88-89b9-0265459bcbcf.png)

## Connecting to Redis cluster from local machine

We can connect to remote Redis cluster from local machine using:

```console
redis-cli -c -h {ansible_host} -p {port}
```
**Note:-** Get value of `{ansible_host}` from **inventory.txt** file and replace `{port}` with its respective value. The `redis-cli` will run in interactive mode. We can connect to any port from 6001 to 6006, the command will get redirected to primary node. Before running any other command, we need to authorize redis with `{password}` set by us in ansible `.yml` file

![image](https://user-images.githubusercontent.com/90673309/215739986-33c378a5-2a35-474c-a621-c292d1e7b357.png)
