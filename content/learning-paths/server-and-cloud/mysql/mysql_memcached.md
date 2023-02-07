---
# User change
title: "Deploy Memcached as a cache for MySQL"

weight: 5 # 1 is first, 2 is second, etc.

# Do not modify these elements
layout: "learningpathall"
---

##  Deploy Memcached as a cache for MySQL 

## Prerequisites

* [An AWS account](https://portal.aws.amazon.com/billing/signup?nc2=h_ct&src=default&redirect_url=https%3A%2F%2Faws.amazon.com%2Fregistration-confirmation#/start)
* [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
* [Ansible](https://www.cyberciti.biz/faq/how-to-install-and-configure-latest-version-of-ansible-on-ubuntu-linux/)
* [Terraform](https://developer.hashicorp.com/terraform/cli/install/apt)
* [Python](https://beebom.com/how-install-python-ubuntu-linux/)
* [Memcached](https://github.com/memcached/memcached/wiki/Install)
* [Telnet](https://adamtheautomator.com/linux-to-install-telnet/)

## Using MySQL with Memcached
**Memcached** is a simple, highly scalable key-based cache that stores data and objects wherever dedicated or spare RAM is available for quick access by applications without going through layers of parsing or disk I/O.
When using Memcached to cache MySQL data, your application must retrieve data from the database and load the appropriate key-value pairs into the cache. Then, subsequent lookups can be done directly from the cache.

Following is the flowchart showing the general sequence for using Memcached:

![flowchart](https://user-images.githubusercontent.com/71631645/212881442-2afcbb74-150a-4997-bd7f-3bca24c94255.jpg)

## Deploy MySQL instances via Terraform

### Generate Access keys (access key ID and secret access key)
The installation of Terraform on your desktop or laptop needs to communicate with AWS. Thus, Terraform needs to be able to authenticate with AWS. For authentication, generate access keys (access key ID and secret access key). These access keys are used by Terraform for making programmatic calls to AWS via the AWS CLI.
To generate an access key and secret key, follow this [documentation](https://github.com/zachlas/arm-software-developers-ads/blob/main/content/learning-paths/server-and-cloud/mysql/ec2_deployment.md#generate-access-keys-access-key-id-and-secret-access-key).

### Generate key-pair(public key, private key)
Before using Terraform, first generate the key-pair (public key and private key) using ssh-keygen. Then associate both public and private keys with AWS EC2 instances.

Generate the key-pair using the following command:
```console
ssh-keygen -t rsa -b 2048
```
![ssh-keygen](https://user-images.githubusercontent.com/71631645/212669106-803c3474-360c-4275-b05e-e9f0aba01d38.jpg)

**Note:** Use the public key `mysql_h.pub` inside the Terraform file to provision/start the instance and the private key `mysql_h` to connect to the instance.

### Create Terraform file (main.tf)
After generating the keys, we have to create the MySQL instances. Then we will push our public key to the **authorized_keys** folder in `~/.ssh`. We will also create a security group that opens inbound ports `22` (ssh) and `3306` (MySQL). Below is a Terraform file called `main.tf` that will do this for us.
    
```console
provider "aws" {
  region = "us-east-2"
  access_key  = "AXXXXXXXXXXXXXXXXXXX"
  secret_key   = "AXXXXXXXXXXXXXXXXXXXXXXXXXXX"
}
  resource "aws_instance" "MYSQL_TEST1" {
  ami           = "ami-064593a301006939b"
  instance_type = "t4g.small"
  security_groups= [aws_security_group.Terraformsecurity1.name]
  key_name = "mysql_h"
  tags = {
    Name = "MYSQL_TEST1"
  }

}
  resource "aws_instance" "MYSQL_TEST2" {
  ami           = "ami-064593a301006939b"
  instance_type = "t4g.small"
  security_groups= [aws_security_group.Terraformsecurity1.name]
  key_name = "mysql_h"
  tags = {
    Name = "MYSQL_TEST2"
  }

}

resource "aws_default_vpc" "main" {
  tags = {
    Name = "main"
  }
}
resource "aws_security_group" "Terraformsecurity1" {
  name        = "Terraformsecurity1"
  description = "Allow TLS inbound traffic"
  vpc_id      = aws_default_vpc.main.id

  ingress {
    description      = "TLS from VPC"
    from_port        = 3306
    to_port          = 3306
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
}
  ingress {
    description      = "TLS from VPC"
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

  tags = {
    Name = "Terraformsecurity1"
  }

 }
resource "local_file" "inventory" {
    depends_on=[aws_instance.MYSQL_TEST1, aws_instance.MYSQL_TEST2]
    filename = "/home/ubuntu/mysql/inventory.txt"
    content = <<EOF
[mysql1]
${aws_instance.MYSQL_TEST1.public_ip}
[mysql2]
${aws_instance.MYSQL_TEST2.public_ip}
[all:vars]
ansible_connection=ssh
ansible_user=ubuntu
                EOF
}

resource "aws_key_pair" "deployer" {
        key_name   = "mysql_h"
        public_key = "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC/GFk2t5I2WOGWIP11kk9+sS2hwb+SuZV8b6KAi8IPR50pDjBXtBBt/8Apl+cyTmUjIlVxnyV6rS4sGVdKLC7SDNU8nl1SfDuh1HJRtlbMu8k+OmA3i9T/rihz2Qs9htkbSkdZ3bADCd5tcregPIht1bdQkjFK5zpbmiNHqIC1KJYIKfiwHMCLt+3ZQWr8iw1G19hHLbfpvDr0H/ewlrpMNG3StJSo6E2Jec6NZ09takFMl0a2r9Cej3bSQz5TuDnxWFDm1xk2svLojROnNeSH2sVx6UoPDpt05eniqgpYdMysYzxeOwS+qMHzR2IV2+0UoDFMxgcSgnhM36qlSk7H ubuntu@ip-172-31-38-39"
} 
    
```
**NOTE:-** Replace `public_key`, `access_key`, `secret_key`, and `key_name` with your values.

Now, use the below Terraform commands to deploy the `main.tf` file.


## Terraform Commands

### Initialize Terraform

Run `terraform init` to initialise the Terraform deployment. This command is responsible for downloading all dependencies that are required by the AWS provider.

```console
terraform init
```

![tf init](https://user-images.githubusercontent.com/71631645/212823559-fa0a8ae9-e58a-47fd-a6c2-cf269fa89334.jpg)

### Create a Terraform execution plan

Run `terraform plan` to create an execution plan.

```console
terraform plan
```

### Apply a Terraform execution plan

Run `terraform apply` to apply the execution plan to your cloud infrastructure. The command below creates all necessary infrastructure.

```console
terraform apply
```

![tf apply](https://user-images.githubusercontent.com/71631645/216935497-ed893abe-5ad2-456f-8a26-a523a34a983f.jpg)

## Configure MySQL through Ansible
To run Ansible, we have to create two `.yml` files, one for each MySQL instance, which is also known as an Ansible-Playbook. Playbook contains a collection of tasks.     
Here is the complete YML file for Ansible-Playbook for both instances.      
For `MYSQL_TEST1`:

```console
---
- hosts: mysql1
  remote_user: root
  become: true

  tasks:
    - name: Update the Machine
      shell: apt-get update -y
    - name: Installing Mysql-Server
      shell: apt-get -y install mysql-server
    - name: Installing PIP for enabling MySQL Modules
      shell: apt -y install python3-pip
    - name: Installing Mysql dependencies
      shell: pip3 install PyMySQL
    - name: start and enable mysql service
      service:
        name: mysql
        state: started
        enabled: yes
    - name: Change Root Password
      shell: sudo mysql -u root -e "ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '{{Your_mysql_password}}'"
    - name: Create database user with password and all database privileges and 'WITH GRANT OPTION'
      mysql_user:
         login_user: root
         login_password: {{Your_mysql_password}}
         login_host: localhost
         name: Local_user
         host: '%'
         password: {{Give_any_password}}
         priv: '*.*:ALL,GRANT'
         state: present
    - name: Create a new database with name 'arm_test1'
      community.mysql.mysql_db:
        name: arm_test1
        login_user: root
        login_password: {{Your_mysql_password}}
        login_host: localhost
        state: present
        login_unix_socket: /run/mysqld/mysqld.sock
    - name: Copy database dump file
      copy:
       src: /home/ubuntu/mysql/table1.sql
       dest: /tmp
    - name: Create a table with dummy values in database
      community.mysql.mysql_db:
       name: arm_test1
       login_user: root
       login_password: {{Your_mysql_password}}
       login_host: localhost
       state: import
       target: /tmp/table1.sql
    - name: MySQL secure installation
      become: yes
      expect:
        command: mysql_secure_installation
        responses:
           'Enter current password for root': '{{Your_mysql_password}}'
           'Set root password': 'n'
           'Remove anonymous users': 'y'
           'Disallow root login remotely': 'n'
           'Remove test database': 'y'
           'Reload privilege tables now': 'y'
        timeout: 1
      register: secure_mariadb
      failed_when: "'... Failed!' in secure_mariadb.stdout_lines"
    - name: Enable remote login by changing bind-address
      lineinfile:
         path: /etc/mysql/mysql.conf.d/mysqld.cnf
         regexp: '^bind-address'
         line: 'bind-address = 0.0.0.0'
         backup: yes
      notify:
         - Restart mysql
  handlers:
    - name: Restart mysql
      service:
        name: mysql
        state: restarted
```

For `MYSQL_TEST2`:
```console
---
- hosts: mysql2
  remote_user: root
  become: true

  tasks:
    - name: Update the Machine
      shell: apt-get update -y
    - name: Installing Mysql-Server
      shell: apt-get -y install mysql-server
    - name: Installing PIP for enabling MySQL Modules
      shell: apt -y install python3-pip
    - name: Installing Mysql dependencies
      shell: pip3 install PyMySQL
    - name: start and enable mysql service
      service:
        name: mysql
        state: started
        enabled: yes
    - name: Change Root Password
      shell: sudo mysql -u root -e "ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '{{Your_mysql_password}}'"
    - name: Create database user with password and all database privileges and 'WITH GRANT OPTION'
      mysql_user:
         login_user: root
         login_password: {{Your_mysql_password}}
         login_host: localhost
         name: Local_user
         host: '%'
         password: {{Give_any_password}}
         priv: '*.*:ALL,GRANT'
         state: present
    - name: Create a new database with name 'arm_test2'
      community.mysql.mysql_db:
        name: arm_test2
        login_user: root
        login_password: {{Your_mysql_password}}
        login_host: localhost
        state: present
        login_unix_socket: /run/mysqld/mysqld.sock
    - name: Copy database dump file
      copy:
       src: /home/ubuntu/mysql/table2.sql
       dest: /tmp
    - name: Create a table with dummy values in database
      community.mysql.mysql_db:
       name: arm_test1
       login_user: root
       login_password: {{Your_mysql_password}}
       login_host: localhost
       state: import
       target: /tmp/table2.sql
    - name: MySQL secure installation
      become: yes
      expect:
        command: mysql_secure_installation
        responses:
           'Enter current password for root': '{{Your_mysql_password}}'
           'Set root password': 'n'
           'Remove anonymous users': 'y'
           'Disallow root login remotely': 'n'
           'Remove test database': 'y'
           'Reload privilege tables now': 'y'
        timeout: 1
      register: secure_mariadb
      failed_when: "'... Failed!' in secure_mariadb.stdout_lines"
    - name: Enable remote login by changing bind-address
      lineinfile:
         path: /etc/mysql/mysql.conf.d/mysqld.cnf
         regexp: '^bind-address'
         line: 'bind-address = 0.0.0.0'
         backup: yes
      notify:
         - Restart mysql
  handlers:
    - name: Restart mysql
      service:
        name: mysql
        state: restarted

```
**NOTE:-** We are using [table1.sql](https://github.com/hirnimeshrampuresoftware/arm-software-developers-ads/files/10433744/table_dot_sql.txt) and [table2.sql](https://github.com/hirnimeshrampuresoftware/arm-software-developers-ads/files/10661786/table2_dot_sql.txt)
 script file to dump data. Specify the path of the files accordingly. Replace `{{Your_mysql_password}}` and `{{Give_any_password}}` with your own password.

In our case, the inventory file will be generated automatically. This file is formed after the `terraform apply` command. 

### Ansible Commands
To run a Playbook, we need to use the `ansible-playbook` command.
```console
ansible-playbook {your_yml_file} -i {your_inventory_file} --key-file {path_to_private_key}
```
**NOTE:-** Run this comamnd for both the yml files. Replace `{your_yml_file}`, `{your_inventory_file}` and `{path_to_private_key}` with your values.
![ansible-start1](https://user-images.githubusercontent.com/71631645/216967739-f64a3a22-da3d-4112-b2fe-9f143b166f8f.jpg)

![ansible-start2](https://user-images.githubusercontent.com/71631645/216967723-99ef0cfb-4fee-4769-9665-6b134a644fae.jpg)

Here is the output after the successful execution of the `ansible-playbook` commands.

![ansible output1](https://user-images.githubusercontent.com/71631645/216938972-1e9e1e1f-b3cb-4849-a68a-232e4e02230a.jpg)

![ansible output2](https://user-images.githubusercontent.com/71631645/216966712-9c8e1f9f-910f-43f5-a655-2e78a35d69de.jpg)

## Script to deploy Memcached as a cache for MySQL using Python
To deploy Memcached as a cache for MySQL using Python, create the following `mem1.py` and `mem2.py` files on the host machine to store data from each of the MySQL instances.     
`mem1.py` file for MYSQL_TEST1:
```console
import sys
import MySQLdb
import pymemcache
from ast import literal_eval

memc = pymemcache.Client("127.0.0.1:11211");

try:
    conn = MySQLdb.connect (host = "{{public_ip of MYSQL_TEST1}}",
                            user = "{{Your_database_user}}",
                            passwd = "{{Your_database_password}}",
                            db = "arm_test1")
except MySQLdb.Error as e:
     print ("Error %d: %s" % (e.args[0], e.args[1]))
     sys.exit (1)

userdata = memc.get('output')

if not userdata:
    cursor = conn.cursor()
    cursor.execute('select user_id, username from user_details order by user_id desc limit 5')
    rows = cursor.fetchall()
    # store the result in the memcached with an expiry time of 60 seconds
    memc.set('output',rows,60)
    print ("Updated memcached with MySQL data")
    for x in rows:
        print(x)
else:
    print ("Loaded data from memcached")
    data = tuple(literal_eval(userdata.decode("utf-8")))
    for row in data:
        print (f"{row[0]},{row[1]}")
```

`mem2.py` file for MYSQL_TEST2:
```console
import sys
import MySQLdb
import pymemcache
from ast import literal_eval

memc = pymemcache.Client("127.0.0.1:11211");

try:
    conn = MySQLdb.connect (host = "{{public_ip of MYSQL_TEST2}}",
                            user = "{{Your_database_user}}",
                            passwd = "{{Your_database_password}}",
                            db = "arm_test2")
except MySQLdb.Error as e:
     print ("Error %d: %s" % (e.args[0], e.args[1]))
     sys.exit (1)

filmdata = memc.get('output1')

if not filmdata:
    cursor = conn.cursor()
    cursor.execute('select movie_id, title from movie order by movie_id desc limit 5')
    rows = cursor.fetchall()
    memc.set('output1',rows,60)
    print ("Updated memcached with MySQL data")
    for x in rows:
        print(x)
else:
    print ("Loaded data from memcached")
    data = tuple(literal_eval(filmdata.decode("utf-8")))
    for row in data:
        print (f"{row[0]},{row[1]}")
```
We are using the `arm_test1` and `arm_test2` databases created above. Replace `{{Your_database_user}}` and `{{Your_database_password}}` with the database user and password created above, and `{{public_ip of MYSQL_TEST1}}` and `{{public_ip of MYSQL_TEST2}}` with the public IPs generated in the `inventory.txt` file after running the Terraform commands.

When the script is executed for the first time, the data is loaded from the MySQL database and stored on the Memcached server.

![mem1](https://user-images.githubusercontent.com/71631645/217153567-bc8748ae-b963-4e00-ac2f-9c5319e70c2f.jpg)
![mem2](https://user-images.githubusercontent.com/71631645/217151671-b4c46cee-3888-4b51-9cd0-350be25f4f6e.jpg)

When executed after that, it loads the data from Memcached. In the example above, the information stored in Memcached is in the form of rows from a Python DB cursor. When accessing the information (within the 60 second expiry time), the data is loaded from Memcached and dumped:

![mem1load](https://user-images.githubusercontent.com/71631645/217153599-9d963f94-61f3-41a5-87e2-07dd849b7511.jpg)                         
![mem12load](https://user-images.githubusercontent.com/71631645/217153604-301a9201-1bab-4728-a724-0e2558a627bf.jpg)           

### Memcached Telnet Commands
To verify that the MySQL query is getting stored in Memcached, connect to the Memcached server with Telnet and start a session:
```console
telnet localhost 11211
```
To retrieve data from Memcached through Telnet:
```console
get <key>
```
![telnet](https://user-images.githubusercontent.com/71631645/217169140-6693c2c0-4cd7-4cfb-9ae3-ded0d11778f0.jpg)
