---
# User change
title: "Deploy Memcached as a cache for MySQL on Azure Arm based Instance"

weight: 4 # 1 is first, 2 is second, etc.

# Do not modify these elements
layout: "learningpathall"
---

## Before you begin

Any computer which has the required tools installed can be used for this section. 

You will need an [Azure portal account](https://azure.microsoft.com/en-in/get-started/azure-portal). Create an account if needed.

Following tools are required on the computer you are using. Follow the links to install the required tools.
* [Azure CLI](/install-tools/azure-cli)
* [Ansible](https://www.cyberciti.biz/faq/how-to-install-and-configure-latest-version-of-ansible-on-ubuntu-linux/)
* [Terraform](/install-tools/terraform)
* [Python](https://beebom.com/how-install-python-ubuntu-linux/)
* [Memcached](/learning-paths/server-and-cloud/memcached/memcached#install-memcached-from-source-on-arm-servers)
* [Telnet](https://adamtheautomator.com/linux-to-install-telnet/)

## Deploy MySQL instances via Terraform

### Azure authentication
The installation of Terraform on your Desktop/Laptop needs to communicate with Azure. Thus, Terraform needs to be authenticated.
For authentication, follow this [documentation](/learning-paths/server-and-cloud/azure/terraform#azure-authentication).

### Generate key-pair (public key, private key)
Before using Terraform, first generate the key-pair (public key and private key) using ssh-keygen. Then associate both public and private keys with Azure instances. To generate the key-pair, follow this [documentation](/learning-paths/server-and-cloud/azure/terraform#generate-key-pair-public-key-private-key-using-ssh-keygen).

### Create Terraform files
After generating the keys, we have to create the MySQL instances. We will create a security group that opens inbound ports **22** (ssh) and **3306** (MySQL). The Terraform configuration is broken into three files: **providers.tf**, **variables.tf** and **main.tf**. Here we are creating 2 instances.

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
Add the resources required to create a virtual machine in **main.tf**.
```console
resource "random_pet" "rg_name" {
  prefix = var.resource_group_name_prefix
}

resource "azurerm_resource_group" "rg" {
  location = var.resource_group_location
  name = random_pet.rg_name.id
}

# Create virtual network
resource "azurerm_virtual_network" "my_terraform_network" {
  name = "myVnet"
  address_space = ["10.1.0.0/16"]
  location = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
}

# Create subnet
resource "azurerm_subnet" "my_terraform_subnet" {
  name = "mySubnet"
  resource_group_name = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.my_terraform_network.name
  address_prefixes = ["10.1.1.0/24"]
}

# Create Public IPs
resource "azurerm_public_ip" "my_terraform_public_ip" {
  name = "myPublicIP${format("%02d", count.index)}-test"
  count= 2
  location = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  allocation_method = "Dynamic"
}

# Create Network Security Group and rule
resource "azurerm_network_security_group" "my_terraform_nsg" {
  name= "myNetworkSecurityGroup"
  location= azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  security_rule {
    name= "SSH"
    priority= 1001
    direction= "Inbound"
    access = "Allow"
    protocol= "Tcp"
    source_port_range= "*"
    destination_port_range = "22"
    source_address_prefix= "*"
    destination_address_prefix = "*"
  }
  security_rule {
    name= "MYSQL"
    priority= 1002
    direction= "Inbound"
    access = "Allow"
    protocol= "Tcp"
    source_port_range= "*"
    destination_port_range = "3306"
    source_address_prefix= "*"
    destination_address_prefix = "*"
  }
}

# Create network interface
resource "azurerm_network_interface" "my_terraform_nic" {
  count= 2
  name= "NIC-${format("%02d", count.index)}-test"
  location= azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  ip_configuration {
    name= "my_nic_configuration"
    subnet_id = azurerm_subnet.my_terraform_subnet.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id= azurerm_public_ip.my_terraform_public_ip.*.id[count.index]
  }
}

# Connect the security group to the network interface
resource "azurerm_network_interface_security_group_association" "example" {
  count= 2
  network_interface_id= azurerm_network_interface.my_terraform_nic.*.id[count.index]
  network_security_group_id = azurerm_network_security_group.my_terraform_nsg.id
}

# Generate random text for a unique storage account name
resource "random_id" "random_id" {
  keepers = {
    # Generate a new ID only when a new resource group is defined
    resource_group = azurerm_resource_group.rg.name
  }

  byte_length = 8
}

# Create storage account for boot diagnostics
resource "azurerm_storage_account" "my_storage_account" {
  name = "diag${random_id.random_id.hex}"
  location = azurerm_resource_group.rg.location
  resource_group_name= azurerm_resource_group.rg.name
  account_tier = "Standard"
  account_replication_type = "LRS"
}

# Create virtual machine
resource "azurerm_linux_virtual_machine" "MYSQL_TEST" {
  name= "MYSQL_TEST${format("%02d", count.index + 1)}"
  count= 2
  location= azurerm_resource_group.rg.location
  resource_group_name= azurerm_resource_group.rg.name
  network_interface_ids = [azurerm_network_interface.my_terraform_nic.*.id[count.index]]
  size= "Standard_D2ps_v5"

  os_disk {
    name = "myOsDisk${format("%02d", count.index + 1)}"
    caching= "ReadWrite"
    storage_account_type = "Premium_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer = "0001-com-ubuntu-server-focal"
    sku= "20_04-lts-arm64"
    version= "20.04.202209200"
  }

  computer_name= "myvm"
  admin_username= "azureuser"
  disable_password_authentication = true

  admin_ssh_key {
    username= "azureuser"
    public_key = file("/path/to/public_key.pub")
  }

  boot_diagnostics {
    storage_account_uri = azurerm_storage_account.my_storage_account.primary_blob_endpoint
  }
}
resource "local_file" "inventory" {
    depends_on=[azurerm_linux_virtual_machine.MYSQL_TEST]
    filename = "inventory.txt"
    content = <<EOF
[mysql1]
${azurerm_linux_virtual_machine.MYSQL_TEST[0].public_ip_address}
[mysql2]
${azurerm_linux_virtual_machine.MYSQL_TEST[1].public_ip_address}
[all:vars]
ansible_connection=ssh
ansible_user=azureuser
                EOF
}
```
**NOTE**:- Replace the path of **public_key** with its respective value.

### Terraform Commands
To deploy the instances, we need to initialize Terraform, generate an execution plan and apply the execution plan to our cloud infrastructure. Follow this [documentation](/learning-paths/server-and-cloud/azure/terraform#terraform-commands) to deploy the **main.tf** file.

## Configure MySQL through Ansible
An Ansible Playbook installs & enables MySQL in the instances and creates databases & tables inside them. To configure MySQL through Ansible and run the Playbook, follow this [documentation](/learning-paths/server-and-cloud/memcached/memcached_mysql_aws#configure-mysql-through-ansible).  


## Deploy Memcached as a cache for MySQL using Python
To deploy Memcached as a cache for MySQL using Python, follow this [documentation](/learning-paths/server-and-cloud/memcached/memcached_mysql_aws#deploy-memcached-as-a-cache-for-mysql-using-python).
