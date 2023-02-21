---
# User change
title: "Deploy Memcached as a cache for MySQL"

weight: 5 # 1 is first, 2 is second, etc.

# Do not modify these elements
layout: "learningpathall"
---

##  Deploy Memcached as a cache for MySQL 

## Prerequisites

* An Azure portal account
* [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-linux?pivots=apt)
* [Ansible](https://www.cyberciti.biz/faq/how-to-install-and-configure-latest-version-of-ansible-on-ubuntu-linux/)
* [Terraform](https://developer.hashicorp.com/terraform/cli/install/apt)
* [Python](https://beebom.com/how-install-python-ubuntu-linux/)
* [Memcached](https://github.com/memcached/memcached/wiki/Install)
* [Telnet](https://adamtheautomator.com/linux-to-install-telnet/)

## Deploy MySQL instances via Terraform

### Azure authentication
The installation of Terraform on your Desktop/Laptop needs to communicate with Azure. Thus, Terraform needs to be authenticated.
For authentication, follow this [documentation](/content/learning-paths/server-and-cloud/azure/terraform.md#azure-authentication).

### Generate key-pair (public key, private key)
Before using Terraform, first generate the key-pair (public key and private key) using ssh-keygen. Then associate both public and private keys with AWS EC2 instances. To generate the key-pair, follow this [documentation](/content/learning-paths/server-and-cloud/azure/terraform.md#generate-key-pair-public-key-private-key-using-ssh-keygen).

### Create Terraform files
After generating the keys, we have to create the MySQL instances. We will also create a security group that opens inbound ports `22` (ssh) and `3306` (MySQL). The Terraform configuration is broken into three files: **providers.tf**, **variables.tf** and **main.tf**.

Add the following code in **providers.tf** file to configure Terraform to communicate with Azure:
    
```console
terraform {
  required_version = ">=0.12"

  required_providers {
    azurerm = {
      source= "hashicorp/azurerm"
      version = "~>2.0"
    }
    random = {
      source= "hashicorp/random"
      version = "~>3.0"
    }
    tls = {
    source = "hashicorp/tls"
    version = "~>4.0"
    }
  }
}

provider "azurerm" {
  features {}
}
    
```
Create a **variables.tf** file for describing the variables referenced in the other files with their type and a default value.
```console
variable "resource_group_location" {
  default = "eastus2"
  description = "Location of the resource group."
}

variable "resource_group_name_prefix" {
  default = "rg"
  description = "Prefix of the resource group name that's combined with a random ID so name is unique in your Azure subscription."
}
```
Add the resources required to create a virtual machine in main.tf.
```console


**NOTE:-** Replace `public_key`, `access_key`, `secret_key`, and `key_name` with respective values. The Terraform commands will automatically generate the **inventory.txt** file on the path provided in the filename. Specify the path accordingly.

### Terraform Commands
To deploy the instances, we need to initialize Terraform, generate an execution plan and apply the execution plan to our cloud infrastructure. Follow this [documentation](/content/learning-paths/server-and-cloud/mysql/ec2_deployment.md#terraform-commands) to deploy the **main.tf** file.

## Configure MySQL through Ansible
To run Ansible, we have to create a `.yml` file, which is also known as an `Ansible-Playbook`. The Playbook contains a collection of tasks.            
Here is the complete YML file for Ansible-Playbook for both instances. This Playbook installs & enables MySQL in the instances and creates databases & tables inside them.  

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
      register: secure_mysql
      failed_when: "'... Failed!' in secure_mysql.stdout_lines"
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

**NOTE:-** We are using [table1.sql](https://github.com/hirnimeshrampuresoftware/arm-software-developers-ads/files/10729309/table1.txt) and [table2.sql](https://github.com/hirnimeshrampuresoftware/arm-software-developers-ads/files/10729310/table2.txt)
 script file to dump data in `MYSQL_TEST[0]` AND `MYSQL_TEST[1]` instances respectively. Specify the path of the files accordingly. Replace `{{Your_mysql_password}}` and `{{Give_any_password}}` with your own password.

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
We create two `.py` files on the host machine to deploy Memcached as a MySQL cache using Python: **values.py** and **mem.py**.  

**values.py** to store the IP addresses of the instances and the databases created in them.
```console
MYSQL_TEST=[["{{public_ip of MYSQL_TEST[0]}}", "arm_test1"],
["{{public_ip of MYSQL_TEST[1]}}", "arm_test2"]]
```
We are using the `arm_test1` and `arm_test2` databases created above through Ansible-Playbook. Replace `{{public_ip of MYSQL_TEST[0]}}` & `{{public_ip of MYSQL_TEST[1]}}` with the public IPs generated in the **inventory.txt** file after running the Terraform commands.       

**mem.py** to access data from Memcached and, if not present, store it in the Memcached.       
```console
import sys
import MySQLdb
import pymemcache
from values import *
from ast import literal_eval
import argparse
parser = argparse.ArgumentParser()

parser.add_argument("-db", "--database", help="Database")
parser.add_argument("-k", "--key", help="Key")
parser.add_argument("-q", "--query", help="Query")
args = parser.parse_args()

memc = pymemcache.Client("127.0.0.1:11211");

for i in range(0,2):
    if (MYSQL_TEST[i][1]==args.database):
        try:
            conn = MySQLdb.connect (host = MYSQL_TEST[i][0],
                                    user = "{{Your_database_user}}",
                                    passwd = "{{Your_database_password}}",
                                    db = MYSQL_TEST[i][1])
        except MySQLdb.Error as e:
             print ("Error %d: %s" % (e.args[0], e.args[1]))
             sys.exit (1)

        sqldata = memc.get(args.key)

        if not sqldata:
            cursor = conn.cursor()
            cursor.execute(args.query)
            rows = cursor.fetchall()
            memc.set(args.key,rows,120)
            print ("Updated memcached with MySQL data")
            for x in rows:
                print(x)
        else:
            print ("Loaded data from memcached")
            data = tuple(literal_eval(sqldata.decode("utf-8")))
            for row in data:
                print (f"{row[0]},{row[1]}")
        break
else:
    print("this database doesn't exist")            
```
Replace `{{Your_database_user}}` & `{{Your_database_password}}` with the database user and password created through Ansible-Playbook. Also change the `range` in `for loop` according to the number of instances created.

To execute the script, run the following command:
```console
python3 mem.py -db <database_name> -k <key> -q <query>
```
Replace `<database_name>` with the database you want to access, `<query>` with the query you want to run in the database and `<key>` with a variable to store the result of the query in Memcached.

When the script is executed for the first time, the data is loaded from the MySQL database and stored on the Memcached server.

![memupdate1](https://user-images.githubusercontent.com/71631645/218663086-19fec362-7360-4622-bd1c-cdb2389799dc.jpg)
![memupdate2](https://user-images.githubusercontent.com/71631645/218663093-75c6033a-7feb-4326-ba55-3f5d3df03d82.jpg)

When executed after that, it loads the data from Memcached. In the example above, the information stored in Memcached is in the form of rows from a Python DB cursor. When accessing the information (within the 120 second expiry time), the data is loaded from Memcached and dumped.

![memload1](https://user-images.githubusercontent.com/71631645/218662938-f62ec905-89ea-4c6d-ac90-f505a75eca31.jpg)
![memload2](https://user-images.githubusercontent.com/71631645/218662947-27e52617-eb9e-4cda-a2ad-b94a85709605.jpg)           

### Memcached Telnet Commands
To verify that the MySQL query is getting stored in Memcached, connect to the Memcached server with Telnet and start a session:
```console
telnet localhost 11211
```
To retrieve data from Memcached through Telnet:
```console
get <key>
```
**NOTE:-** Key is the variable in which we store the data. In the above command, we are storing the data from table1 and table2 in `AA` and `BB` respectively.

![telnetfinalfinal](https://user-images.githubusercontent.com/71631645/218663147-8a7e0d6f-39d5-4b2e-9487-501d3ffbf40b.jpg)
