## Terraform Files Overview

**provider.tf** instructs terraform to use the azure resource manager plugin for terraform. The Azure variables required are located in **variables.tf** 

```# Configure the Microsoft Azure Provider, replace Service Principal and Subscription with your own
provider "azurerm" {
    subscription_id = "${var.SP["subscription_id"]}"
    client_id       = "${var.SP["client_id"]}"
    client_secret   = "${var.SP["client_secret"]}"
    tenant_id       = "${var.SP["tenant_id"]}"
}
```

**main.tf** contains the code to create the Azure networking components, F5, and Nginx resouces. 

1. Create the Azure resource group all components are assigned:
```
# Create a Resource Group for the new Virtual Machine
resource "azurerm_resource_group" "main" {
  name			= "${var.prefix}_rg"
  location = "${var.location}"
}
```
2. Create two virtual networks to acheive the Hub (DMZ) and Spoke (Application) topology. Additional networks could be created for different Applications.  
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
3. Create subnets for Hub (DMZ) and Spoke (Application) virtual networks
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
