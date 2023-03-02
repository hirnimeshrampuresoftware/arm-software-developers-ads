---
# User change
title: "Install Redis on a single AWS Arm based instance"

weight: 3 # 1 is first, 2 is second, etc.

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

## Generate Access keys (access key ID and secret access key)

The installation of Terraform on your desktop or laptop needs to communicate with AWS. Thus, Terraform needs to be able to authenticate with AWS. For authentication, we need to generate access keys (access key ID and secret access key). These access keys are used by Terraform for making programmatic calls to AWS via the AWS CLI.
  
Go to **Security Credentials**
   
![image](https://user-images.githubusercontent.com/90673309/218450269-d6efed3a-c395-4e93-8790-7b48d88c89a7.png)

On Your **Security Credentials** page click on **Create access keys** (access key ID and secret access key)
   
![image](https://user-images.githubusercontent.com/87687468/190137925-c725359a-cdab-468f-8195-8cce9c1be0ae.png)
   
Copy the **Access Key ID** and **Secret Access Key**

![image](https://user-images.githubusercontent.com/87687468/190138349-7cc0007c-def1-48b7-ad1e-4ee5b97f4b90.png)

## Generate key-pair(public key, private key) using ssh keygen

Before using Terraform, first generate the key-pair (public key, private key) using ssh-keygen. Then associate both public and private keys with AWS EC2 instances.

Generate the **key-pair** using the following command:

```console
ssh-keygen -t rsa -b 2048
```
       
By default, the above command will generate the public as well as private key at location **$HOME/.ssh**. You can override the end destination with a custom path.

Output when a **key-pair** is generated:

![image](https://user-images.githubusercontent.com/90673309/218444335-b136a7b5-15c3-437e-86ce-a01513d16b03.png)
      
**Note:** Use the public key aws_key.pub inside the Terraform file to provision/start the instance and private key aws_key to connect to the instance.

## Deploy AWS Arm based instance via Terraform

After generating the public and private keys, we need to create an AWS Arm based instance. Then we will push our public key to the **authorized_keys** folder in **~/.ssh**. We will also create a security group that opens inbound ports **22**(ssh) and **6379**(Redis). Below is a Terraform file named **main.tf** which will do this for us.


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
    description      = "Open redis connection port"
    from_port        = 6379
    to_port          = 6379
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
**NOTE:-** Replace **public_key**, **access_key**, **secret_key**, and **key_name** with respective values.

Now, use the below Terraform commands to deploy the **main.tf** file.

### Terraform Commands

#### Initialize Terraform

Run **terraform init** to initialize the Terraform deployment. This command is responsible for downloading all dependencies which are required for the AWS provider.

```console
terraform init
```

![image](https://user-images.githubusercontent.com/90673309/218444507-72649677-3ae7-4e9f-942b-b533409802b9.png)

#### Create a Terraform execution plan

Run **terraform plan** to create an execution plan.

```console
terraform plan
```

**NOTE:** The **terraform plan** command is optional. You can directly run **terraform apply** command. But it is always better to check the resources about to be created.

#### Apply a Terraform execution plan

Run **terraform apply** to apply the execution plan to your cloud infrastructure. The below command creates all required infrastructure.

```console
terraform apply
```      

![image](https://user-images.githubusercontent.com/90673309/218444666-e7e4ef27-2f5f-4d30-bf93-e76ea0e8c123.png)


## Install Redis using Ansible
Ansible is a software tool that provides simple but powerful automation for cross-platform computer support.
Ansible allows you to configure not just one computer, but potentially a whole network of computers at once.
To run Ansible, we have to create a **.yml** file, which is also known as **Ansible-Playbook**. The following playbook contains a collection of tasks which install Redis on a single node.

Here is the complete **deploy_redis.yml** file of Ansible-Playbook
```console
---
- hosts: all
  become: true
  become_user: root
  remote_user: ubuntu

  tasks:
    - name: Update the Machine
      shell: apt update -y
    - name: Download redis gpg key
      shell: curl -fsSL "https://packages.redis.io/gpg" | gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg
      args:
        warn: false
    - name: Add redis gpg key
      shell: echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" |  tee /etc/apt/sources.list.d/redis.list
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
          port 6379
          protected-mode yes
          cluster-enabled no
          daemonize yes
          appendonly no
      become_user: ubuntu
    - name: Stop redis-server
      shell: service redis-server stop
    - name: Start redis server with configuration files
      shell: redis-server redis.conf
      args:
        chdir: "/home/ubuntu/redis"
      become_user: ubuntu
    - name: Set Authentication password
      shell: redis-cli -p 6379 CONFIG SET requirepass "{password}"
      become_user: ubuntu
```
**NOTE:-** Replace **{password}** with respective value.

To run a Playbook, we need to use the **ansible-playbook** command.
```console
ansible-playbook {your_yml_file} -i {your_inventory_file} --key-file {path_to_private_key}
```
**NOTE:-** Replace **{your_yml_file}, {your_inventory_file}** and **{path_to_private_key}** with respective values.

Here is the output after the successful execution of the **ansible-playbook** command.

![ansible](https://user-images.githubusercontent.com/71631645/220877481-46f101da-6f84-494a-b38d-e56a4bec88fc.jpg)

## Connecting to Redis server from local machine

We can connect to remote Redis server from local machine. We need to use redis-tools to interact with redis-server.
```console
apt install redis-tools
```
```console
redis-cli -h {ansible_host} -p {port}
```
**NOTE:-** Get value of **{ansible_host}** from **inventory.txt** file and replace **{port}** with its respective value. 

The **redis-cli** will run in interactive mode. Before running any command, we need to authorize Redis with the **{password}** set by us in **deploy_redis.yml** file.
Otherwise, we will keep getting errors on running any command.

![redis-cli](https://user-images.githubusercontent.com/71631645/220876727-d04eab2c-3829-409a-86b2-5af91ee95661.jpg)
