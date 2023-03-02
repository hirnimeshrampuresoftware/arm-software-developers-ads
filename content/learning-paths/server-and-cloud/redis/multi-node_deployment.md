---
# User change
title: "Install Redis in a multi-node configuration"

weight: 7 # 1 is first, 2 is second, etc.

# Do not modify these elements
layout: "learningpathall"
---

## Before you begin

Any computer which has the required tools installed can be used for this section. 

You will need [an AWS account](https://portal.aws.amazon.com/billing/signup?nc2=h_ct&src=default&redirect_url=https%3A%2F%2Faws.amazon.com%2Fregistration-confirmation#/start). Create an account if needed.

Following tools are required on the computer you are using. Follow the links to install the required tools.
* [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
* [Ansible](https://www.cyberciti.biz/faq/how-to-install-and-configure-latest-version-of-ansible-on-ubuntu-linux/)
* [Terraform](/install-tools/terraform)
* [Redis CLI](https://redis.io/docs/getting-started/installation/install-redis-on-linux/)

## Deploy AWS Arm based instance via Terraform

Before deploying AWS Arm based instance via Terraform, generate [Access keys](/content/learning-paths/server-and-cloud/redis/aws_deployment.md#generate-access-keys-access-key-id-and-secret-access-key) and [key-pair using ssh keygen](/content/learning-paths/server-and-cloud/redis/aws_deployment.md#generate-key-pairpublic-key-private-key-using-ssh-keygen).

After generating the public and private keys, we will push our public key to the **authorized_keys** folder in **~/.ssh**. We will also create a security group that opens inbound ports **22**(ssh). Also every Redis Cluster node requires two TCP connections open. The normal Redis TCP port used to serve clients, for example **6379**, plus the port obtained by adding 10000 to the data port, so **16379** in the example. Below is a Terraform file named **main.tf** which will do this for us. Here we are creating 6 instances.


```console
provider "aws" {
  region = "us-east-2"
  access_key  = "AXXXXXXXXXXXXXXXXXXX"
  secret_key   = "AXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
}
resource "aws_instance" "redis-deployment" {
  ami = "ami-0bc02c3c09aaee8ea"
  count = "6"
  instance_type = "t4g.small"
  key_name= "aws_key"
  vpc_security_group_ids = [aws_security_group.main.id]
}

resource "aws_security_group" "main" {
  name        = "main"
  description = "Allow TLS inbound traffic"

  ingress {
    description      = "Open redis connection port"
    from_port        = 6379
    to_port          = 6379
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
  }
  ingress {
    description      = "Open port for Cluster bus"
    from_port        = 16379
    to_port          = 16379
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
[redis]

${aws_instance.redis-deployment[0].public_dns}
${aws_instance.redis-deployment[1].public_dns}
${aws_instance.redis-deployment[2].public_dns}
${aws_instance.redis-deployment[3].public_dns}
${aws_instance.redis-deployment[4].public_dns}
${aws_instance.redis-deployment[5].public_dns}

[all:vars]
host_key_checking=false
ansible_connection=ssh
ansible_user=ubuntu
                EOF
}

resource "aws_key_pair" "deployer" {
        key_name   = "aws_key"
        public_key = "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC/GFk2t5I2WOGWIP11kk9+sS2hwb+SuZV8b6KAi8IPR50pDjBXtBBt/8Apl+cyTmUjIlVxnyV6rS4sGVdKLC7SDNU8nl1SfDuh1HJRtlbMu8k+OmA3i9T/rihz2Qs9htkbSkdZ3bADCd5tcregPIht1bdQkjFK5zpbmiNHqIC1KJYIKfiwHMCLt+3ZQWr8iw1G19hHLbfpvDr0H/ewlrpMNG3StJSo6E2Jec6NZ09takFMl0a2r9Cej3bSQz5TuDnxWFDm1xk2svLojROnNeSH2sVx6UoPDpt05eniqgpYdMysYzxeOwS+qMHzR2IV2+0UoDFMxgcSgnhM36qlSk7H ubuntu@ip-172-XX-XX-XX"
}
```
**NOTE:-** Replace **public_key**, **access_key**, **secret_key**, and **key_name** with actual values.


### Terraform Commands

To deploy the instances, we need to initialize Terraform, generate an execution plan and apply the execution plan to our cloud infrastructure. Follow this [documentation](/content/learning-paths/server-and-cloud/redis/aws_deployment.md#terraform-commands) to deploy the **main.tf** file.

## Install Redis in a multi-node configuration using Ansible
To run Ansible, we have to create a **.yml** file, which is also known as **Ansible-Playbook**. The following playbook contains a collection of tasks which install Redis in multi-node configuration (3 primary and 3 replica nodes). In our example, we installing Redis Cluster in the 6 instances created above. 

Here is the complete **deploy_redis.yml** file of Ansible-Playbook
```console
---
- name: Redis Cluster Install
  hosts: redis
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
        path: "/home/ubuntu/redis"
        state: directory
      become_user: ubuntu
    - name: Create configuration files
      copy:
       dest: "/home/ubuntu/redis/redis.conf"
       content: |
         bind 0.0.0.0
         protected-mode no
         port 6379
         cluster-enabled yes
         cluster-config-file nodes.conf
         cluster-node-timeout 5000
         daemonize yes
         appendonly yes
      become_user: ubuntu
    - name: Stop redis-server
      shell: service redis-server stop
    - name: Start redis server with configuration files
      shell: redis-server redis.conf
      args:
        chdir: "/home/ubuntu/redis"
      become_user: ubuntu
```
**NOTE:-** Since the allocation of master and slave nodes is random at the time of cluster creation, it is difficult to know whether the nodes at ports `6001-6006` are master or slave nodes. Hence, for the multi-node configuration, we need to turn off **protected-mode**, which is enabled by default, so that we can connect to master and slave nodes. 

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
**Note:-** Get value of `{ansible_host}` from **inventory.txt** file and replace `{port}` with its respective value. The `redis-cli` will run in interactive mode. We can connect to any port from 6001 to 6006, the command will get redirected to master node. Before running any other command, we need to authorize redis with `{password}` set by us in ansible `.yml` file

![image](https://user-images.githubusercontent.com/90673309/215739986-33c378a5-2a35-474c-a621-c292d1e7b357.png)
