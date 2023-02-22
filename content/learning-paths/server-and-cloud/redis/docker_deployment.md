---
# User change
title: "Install Redis on a Docker container"

weight: 6 # 1 is first, 2 is second, etc.

# Do not modify these elements
layout: "learningpathall"
---

##  Install Redis on a Docker container

## Prerequisites

* An [AWS account](https://portal.aws.amazon.com/billing/signup?nc2=h_ct&src=default&redirect_url=https%3A%2F%2Faws.amazon.com%2Fregistration-confirmation#/start)
* [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
* [AWS IAM authenticator](https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html)
* [Ansible](https://www.cyberciti.biz/faq/how-to-install-and-configure-latest-version-of-ansible-on-ubuntu-linux/)
* [Terraform](/content/install-tools/terraform.md)
* [Redis CLI](https://redis.io/docs/getting-started/installation/install-redis-on-linux/)


## Deploy AWS Arm based instance via Terraform

Before deploying AWS Arm based instance via Terraform, generate [Access keys](/content/learning-paths/server-and-cloud/redis/aws_deployment.md#generate-access-keys-access-key-id-and-secret-access-key) and [key-pair using ssh keygen](/content/learning-paths/server-and-cloud/redis/aws_deployment.md#generate-key-pairpublic-key-private-key-using-ssh-keygen).

Follow this [documentation](/content/learning-paths/server-and-cloud/redis/aws_deployment.md#deploy-aws-arm-based-instance-via-terraform) to deploy AWS Arm based instance via Terraform.


## Install Redis on a Docker container using Ansible
To run Ansible, we have to create a `.yml` file, which is also known as `Ansible-Playbook`. The following playbook contains a collection of tasks which install Redis on a Docker container.

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
    - name: Install docker dependencies
      shell: apt install -y ca-certificates curl gnupg lsb-release
    - name: Create directory
      file:
        path: /etc/apt/keyrings
        state: directory
    - name: Download docker gpg key
      shell: curl -fsSL 'https://download.docker.com/linux/ubuntu/gpg' | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
      args:
        warn: false
    - name: Add docker gpg key
      shell: echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu  $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
    - name: Update the apt sources
      shell: apt update
    - name: Install docker
      shell: apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
    - name: Start and enable docker service
      service:
        name: docker
        state: started
        enabled: yes
    - name: Start redis container
      shell: docker run --name redis-container -p 6000:6379 -d redis
    - name: Set Authentication password
      shell: docker exec -it redis-container redis-cli CONFIG SET requirepass "{password}"
```
**NOTE:-** Replace `{password}` with respective value.

To access the Redis server running inside the Docker container on port `6379`, we need to expose it to any available port on the machine using `-p {port_no_of_machine}:6379` argument. We need to replace `{port_no_of_machine}` with its respective value. In our example, we have used port number `6000`.


To run a Playbook, we need to use the `ansible-playbook` command.
```console
ansible-playbook {your_yml_file} -i {your_inventory_file} --key-file {path_to_private_key}
```
**NOTE:-** Replace `{your_yml_file}` and `{path_to_private_key}` with respective values.

![image](https://user-images.githubusercontent.com/90673309/218455868-6ab3f027-d36a-46ea-ad0f-d7c25d7a4652.png)

Here is the output after the successful execution of the `ansible-playbook` command.

![image](https://user-images.githubusercontent.com/90673309/218455991-267b7e51-e43a-4257-8808-a04c21041b41.png)

## Connecting to Redis server from local machine

Follow this [documentation](/content/learning-paths/server-and-cloud/redis/aws_deployment.md#connecting-to-redis-server-from-local-machine) to connect to the remote Redis server from our local machine.
