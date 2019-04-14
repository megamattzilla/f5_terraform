# Terraform Files Overview

## **provider.tf** instructs terraform to use the azure resource manager plugin for terraform. The Azure variables required are located in **variables.tf** 

```# Configure the Microsoft Azure Provider, replace Service Principal and Subscription with your own
provider "azurerm" {
    subscription_id = "${var.SP["subscription_id"]}"
    client_id       = "${var.SP["client_id"]}"
    client_secret   = "${var.SP["client_secret"]}"
    tenant_id       = "${var.SP["tenant_id"]}"
}
```

## **main.tf** contains the code to create the Azure networking components, F5, and Nginx resouces. 

#### 1. Create the Azure resource group where all Azure components will be built:
```
# Create a Resource Group for the new Virtual Machine
resource "azurerm_resource_group" "main" {
  name			= "${var.prefix}_rg"
  location = "${var.location}"
}
```
#### 2. Create two virtual networks to acheive the Hub (DMZ) and Spoke (Application) topology. Additional networks could be created for different Applications.  
```# Create a Virtual Network within the Resource Group
resource "azurerm_virtual_network" "main" {
  name			= "${var.prefix}-hub"
  address_space		= ["${var.cidr}", "${var.sslo-cidr}"]
  resource_group_name	= "${azurerm_resource_group.main.name}"
  location		= "${azurerm_resource_group.main.location}"
}

# Create a Virtual Network within the Resource Group
resource "azurerm_virtual_network" "spoke" {
  name                  = "${var.prefix}-spoke"
  address_space         = ["${var.app-cidr}"]
  resource_group_name   = "${azurerm_resource_group.main.name}"
  location              = "${azurerm_resource_group.main.location}"
}
```
#### 3. Create subnets for Hub (DMZ) and Spoke (Application) virtual networks
```
# Create the Mgmt Subnet within the Hub Virtual Network
resource "azurerm_subnet" "Mgmt" {
  name			= "Mgmt"
  virtual_network_name	= "${azurerm_virtual_network.main.name}"
  resource_group_name	= "${azurerm_resource_group.main.name}"
  address_prefix	= "${var.subnets["subnet1"]}"
}

# Create the External Subnet within the Hub Virtual Network
resource "azurerm_subnet" "External" {
  name			= "External"
  virtual_network_name	= "${azurerm_virtual_network.main.name}"
  resource_group_name	= "${azurerm_resource_group.main.name}"
  address_prefix	= "${var.subnets["subnet2"]}"
}

# Create the Untrust Subnet within the Hub Virtual Network
resource "azurerm_subnet" "Untrust" {
  name                  = "Untrust"
  virtual_network_name  = "${azurerm_virtual_network.main.name}"
  resource_group_name   = "${azurerm_resource_group.main.name}"
  address_prefix        = "${var.sslo-subnets["subnet1"]}"
}

# Create the Trust Subnet within the Hub Virtual Network
resource "azurerm_subnet" "Trust" {
  name                  = "Trust"
  virtual_network_name  = "${azurerm_virtual_network.main.name}"
  resource_group_name   = "${azurerm_resource_group.main.name}"
  address_prefix        = "${var.sslo-subnets["subnet2"]}"
}

# Create the App1 Subnet within the Spoke Virtual Network
resource "azurerm_subnet" "App1" {
  name                  = "App1"
  virtual_network_name  = "${azurerm_virtual_network.spoke.name}"
  resource_group_name   = "${azurerm_resource_group.main.name}"
  address_prefix        = "${var.app-subnets["subnet1"]}"
}
```
#### 4. Create an external Azure Load Balancer with a public frontside IP, create the F5 network interfaces, and add the F5 interfaces as the Azure LB pool
```
# Create Azure LB
resource "azurerm_lb" "lb" {
  name                  = "${var.prefix}lb"
  location              = "${azurerm_resource_group.main.location}"
  resource_group_name	= "${azurerm_resource_group.main.name}"

  frontend_ip_configuration {
    name                = "LoadBalancerFrontEnd"
    public_ip_address_id	= "${azurerm_public_ip.lbpip.id}"
  }
}

resource "azurerm_lb_backend_address_pool" "backend_pool" {
  name                  = "BackendPool1"
  resource_group_name	= "${azurerm_resource_group.main.name}"
  loadbalancer_id       = "${azurerm_lb.lb.id}"
}

#Create F5 vm01 External Interface
resource "azurerm_network_interface" "vm01-ext-nic" {
  name                = "${var.prefix}-vm01-ext-nic"
  location            = "${azurerm_resource_group.main.location}"
  resource_group_name = "${azurerm_resource_group.main.name}"
  network_security_group_id = "${azurerm_network_security_group.main.id}"
  enable_ip_forwarding	    = true
  depends_on          = ["azurerm_lb_backend_address_pool.backend_pool"]

  ip_configuration {
    name                          = "primary"
    subnet_id                     = "${azurerm_subnet.External.id}"
    private_ip_address_allocation = "Static"
    private_ip_address            = "${var.f5vm01ext}"
    primary			  = true
  }

  ip_configuration {
    name                          = "secondary"
    subnet_id                     = "${azurerm_subnet.External.id}"
    private_ip_address_allocation = "Static"
    private_ip_address            = "${var.f5vm01ext_sec}"
  }

  tags {
    Name           = "${var.environment}-vm01-ext-int"
    environment    = "${var.environment}"
    owner          = "${var.owner}"
    group          = "${var.group}"
    costcenter     = "${var.costcenter}"
    application    = "${var.application}"
  }
}

#Create F5 vm02 External Interface
resource "azurerm_network_interface" "vm02-ext-nic" {
  name                = "${var.prefix}-vm02-ext-nic"
  location            = "${azurerm_resource_group.main.location}"
  resource_group_name = "${azurerm_resource_group.main.name}"
  network_security_group_id = "${azurerm_network_security_group.main.id}"
  enable_ip_forwarding      = true
  depends_on          = ["azurerm_lb_backend_address_pool.backend_pool"]

  ip_configuration {
    name                          = "primary"
    subnet_id                     = "${azurerm_subnet.External.id}"
    private_ip_address_allocation = "Static"
    private_ip_address            = "${var.f5vm02ext}"
    primary			  = true
  }

  ip_configuration {
    name                          = "secondary"
    subnet_id                     = "${azurerm_subnet.External.id}"
    private_ip_address_allocation = "Static"
    private_ip_address            = "${var.f5vm02ext_sec}"
  }

  tags {
    Name           = "${var.environment}-vm02-ext-int"
    environment    = "${var.environment}"
    owner          = "${var.owner}"
    group          = "${var.group}"
    costcenter     = "${var.costcenter}"
    application    = "${var.application}"
  }
}

# Associate the F5 External Network Interface to the Azure LB BackendPool
resource "azurerm_network_interface_backend_address_pool_association" "bpool_assc_vm01" {
  depends_on          = ["azurerm_lb_backend_address_pool.backend_pool", "azurerm_network_interface.vm01-ext-nic"]
  network_interface_id    = "${azurerm_network_interface.vm01-ext-nic.id}"
  ip_configuration_name   = "secondary"
  backend_address_pool_id = "${azurerm_lb_backend_address_pool.backend_pool.id}"
}

resource "azurerm_network_interface_backend_address_pool_association" "bpool_assc_vm02" {
  depends_on          = ["azurerm_lb_backend_address_pool.backend_pool", "azurerm_network_interface.vm02-ext-nic"]
  network_interface_id    = "${azurerm_network_interface.vm02-ext-nic.id}"
  ip_configuration_name   = "secondary"
  backend_address_pool_id = "${azurerm_lb_backend_address_pool.backend_pool.id}"
}
```
