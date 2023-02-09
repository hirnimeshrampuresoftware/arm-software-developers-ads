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

### Generate key-pair (public key, private key)
Before using Terraform, first generate the key-pair (public key and private key) using ssh-keygen. Then associate both public and private keys with AWS EC2 instances. To generate the key-pair, follow this [documentation](https://github.com/zachlas/arm-software-developers-ads/blob/main/content/learning-paths/server-and-cloud/mysql/ec2_deployment.md#generate-key-pairpublic-key-private-key-using-ssh-keygen)

### Create Terraform file (main.tf)
After generating the keys, we have to create the MySQL instances. Then we will push our public key to the **authorized_keys** folder in `~/.ssh`. We will also create a security group that opens inbound ports `22` (ssh) and `3306` (MySQL). Below is a Terraform file called **main.tf** that will do this for us.
    
```console
provider "aws" {
  region = "us-east-2"
  access_key  = "AXXXXXXXXXXXXXXXXXXX"
  secret_key   = "AXXXXXXXXXXXXXXXXXXXXXXXXXXX"
}
  resource "aws_instance" "MYSQL_TEST" {
  count         = "2"
  ami           = "ami-064593a301006939b"
  instance_type = "t4g.small"
  security_groups= [aws_security_group.Terraformsecurity1.name]
  key_name = "mysql_h"
  tags = {
    Name = "MYSQL_TEST"
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
    depends_on=[aws_instance.MYSQL_TEST]
    filename = "/path/to/inventory/inventory.txt"
    content = <<EOF
[mysql1]
${aws_instance.MYSQL_TEST[0].public_ip}
[mysql2]
${aws_instance.MYSQL_TEST[1].public_ip}
[all:vars]
ansible_connection=ssh
ansible_user=ubuntu
                EOF
}

resource "aws_key_pair" "deployer" {
        key_name   = "mysql_h"
        public_key = "ssh-rsaxxxxxxxxxxxxxxx"
} 
    
```
**NOTE:-** Replace `public_key`, `access_key`, `secret_key`, and `key_name` with respective values. The Terraform commands will automatically generate the **inventory.txt** file in the path provided in the filename. Specify the path accordingly.

### Terraform Commands
Follow this [documentation](https://github.com/zachlas/arm-software-developers-ads/blob/main/content/learning-paths/server-and-cloud/mysql/ec2_deployment.md#terraform-commands) to deploy the **main.tf** file.

## Configure MySQL through Ansible
To run Ansible, we have to create a `.yml` file, which is also known as `Ansible-Playbook`. Playbook contains a collection of tasks.       
Here is the complete YML file for Ansible-Playbook for both instances.      

```console
---
- hosts: mysql1, mysql2
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
      when: "'mysql1' in group_names"
      community.mysql.mysql_db:
        name: arm_test1
        login_user: root
        login_password: {{Your_mysql_password}}
        login_host: localhost
        state: present
        login_unix_socket: /run/mysqld/mysqld.sock
    - name: Copy database dump file
      when: "'mysql1' in group_names"    
      copy:
       src: path/to/table1.sql
       dest: /tmp
    - name: Create a table with dummy values in database
      when: "'mysql1' in group_names"    
      community.mysql.mysql_db:
       name: arm_test1
       login_user: root
       login_password: {{Your_mysql_password}}
       login_host: localhost
       state: import
       target: /tmp/table1.sql
    - name: Create a new database with name 'arm_test2'
      when: "'mysql2' in group_names"
      community.mysql.mysql_db:
        name: arm_test2
        login_user: root
        login_password: {{Your_mysql_password}}
        login_host: localhost
        state: present
        login_unix_socket: /run/mysqld/mysqld.sock
    - name: Copy database dump file
      when: "'mysql2' in group_names"    
      copy:
       src: path/to/table2.sql
       dest: /tmp
    - name: Create a table with dummy values in database
      when: "'mysql2' in group_names"    
      community.mysql.mysql_db:
       name: arm_test2
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

**NOTE:-** We are using [table1.sql](https://github.com/hirnimeshrampuresoftware/arm-software-developers-ads/files/10681894/table1_dot_sql.txt) and [table2.sql](https://github.com/hirnimeshrampuresoftware/arm-software-developers-ads/files/10681895/table2_dot_sql.txt)
 script file to dump data. Specify the path of the files accordingly. Replace `{{Your_mysql_password}}` and `{{Give_any_password}}` with your own password.

### Ansible Commands
To run a Playbook, we need to use the `ansible-playbook` command.
```console
ansible-playbook {your_yml_file} -i {your_inventory_file} --key-file {path_to_private_key}
```
**NOTE:-** Replace `{your_yml_file}`, `{your_inventory_file}` and `{path_to_private_key}` with your values.

![ansible-start-final](https://user-images.githubusercontent.com/71631645/217766884-ec19676f-11b2-4f20-b06f-dd307b08febc.jpg)

Here is the output after the successful execution of the `ansible-playbook` commands.

![ansible-end-final](https://user-images.githubusercontent.com/71631645/217766981-3e00b3f6-6ba8-47eb-9c8d-fefcd1685e36.jpg)

## Deploy Memcached as a cache for MySQL using Python
To deploy Memcached as a cache for MySQL using Python, create the following **mem1.py** and **mem2.py** files on the host machine to store data from each of the MySQL instances.     
**mem1.py** file for MYSQL_TEST[0]:
```console
import sys
import MySQLdb
import pymemcache
from ast import literal_eval

memc = pymemcache.Client("127.0.0.1:11211");

try:
    conn = MySQLdb.connect (host = "{{public_ip of MYSQL_TEST[0]}}",
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
    # store the result in the Memcached with an expiry time of 120 seconds
    memc.set('output',rows,120)
    print ("Updated Memcached with MySQL data")
    for x in rows:
        print(x)
else:
    print ("Loaded data from Memcached")
    data = tuple(literal_eval(userdata.decode("utf-8")))
    for row in data:
        print (f"{row[0]},{row[1]}")
```

**mem2.py** file for MYSQL_TEST[1]:
```console
import sys
import MySQLdb
import pymemcache
from ast import literal_eval

memc = pymemcache.Client("127.0.0.1:11211");

try:
    conn = MySQLdb.connect (host = "{{public_ip of MYSQL_TEST[1]}}",
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
    memc.set('output1',rows,120)
    print ("Updated Memcached with MySQL data")
    for x in rows:
        print(x)
else:
    print ("Loaded data from Memcached")
    data = tuple(literal_eval(filmdata.decode("utf-8")))
    for row in data:
        print (f"{row[0]},{row[1]}")
```
We are using the `arm_test1` and `arm_test2` databases created above through Ansible-Playbook. Replace `{{Your_database_user}}` and `{{Your_database_password}}` with the database user and password created through Ansible-Playbook, and `{{public_ip of MYSQL_TEST[0]}}` and `{{public_ip of MYSQL_TEST[1]}}` with the public IPs generated in the **inventory.txt** file after running the Terraform commands.    

To execute the script, run the following command:
```console
python3 <filename.py>
```
When the script is executed for the first time, the data is loaded from the MySQL database and stored on the Memcached server.

![mem1](https://user-images.githubusercontent.com/71631645/217153567-bc8748ae-b963-4e00-ac2f-9c5319e70c2f.jpg)
![mem2](https://user-images.githubusercontent.com/71631645/217151671-b4c46cee-3888-4b51-9cd0-350be25f4f6e.jpg)

When executed after that, it loads the data from Memcached. In the example above, the information stored in Memcached is in the form of rows from a Python DB cursor. When accessing the information (within the 120 second expiry time), the data is loaded from Memcached and dumped:

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
**NOTE:-** Key is the variable in which we store the data. In the above python script, we are storing the data from table1 and table2 in `output` and `output1` respectively.

![telnet](https://user-images.githubusercontent.com/71631645/217169140-6693c2c0-4cd7-4cfb-9ae3-ded0d11778f0.jpg)
