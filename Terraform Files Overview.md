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

## **main.tf** contains the code to create the Azure networking components, F5, and Nginx resources. 

#### 1. Create the Azure resource group where all Azure components will be built:
```
# Create a Resource Group for the new Virtual Machine
resource "azurerm_resource_group" "main" {
  name			= "${var.prefix}_rg"
  location = "${var.location}"
}
```
#### 2. Create two virtual networks to achieve the Hub (DMZ) and Spoke (Application) topology. Additional spoke networks could be created for additional applications.  
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
#### 5. Create the F5 script file which when executed later will install declarative onboarding module (DO) and application services 3 extension (AS3).
Take the shell script onboard.tpl from this repository and render it as a template file called "vm_onboard" using some variables from variables.tf. 
```# Setup Onboarding scripts
data "template_file" "vm_onboard" {
  template = "${file("${path.module}/onboard.tpl")}"

  vars {
    uname        	  = "${var.uname}"
    upassword        	  = "${var.upassword}"
    DO_onboard_URL        = "${var.DO_onboard_URL}"
    AS3_URL		  = "${var.AS3_URL}"
    sslo_URL		  = "${var.sslo_URL}"
    libs_dir		  = "${var.libs_dir}"
    onboard_log		  = "${var.onboard_log}"
  }
}
```
#### 6. Create the Declarative Onboarding (DO) JSON payload for F5 VM01 and VM02. When DO is executed later it will license the F5's, provision LTM and create the F5 self IPs, hostname, sync failover device group, DNS, routing, NTP, username, and password.  
Take the JSON file cluster.json from this repository and render it as two template files called "vm01_do_json" and "vm02_do_json" using some variables from variables.tf.
```
data "template_file" "vm01_do_json" {
  template = "${file("${path.module}/cluster.json")}"

  vars {
    #Uncomment the following line for BYOL
    regkey	    = "${var.license1}"

    host1	    = "${var.host1_name}"
    host2	    = "${var.host2_name}"
    local_host      = "${var.host1_name}"
    local_selfip1   = "${var.f5vm01ext}"
    local_selfip2   = "${var.f5vm01tosrv}"
    local_selfip3   = "${var.f5vm01frsrv}"
    tosrvfl1	    = "${var.f5vm01tosrvfl}"
    tosrvfl2       = "${var.f5vm02tosrvfl}"
    frsrvfl1	    = "${var.f5vm01frsrvfl}"
    frsrvfl2       = "${var.f5vm02frsrvfl}"
    remote_selfip   = "${var.f5vm01ext}"
    gateway	    = "${local.ext_gw}"
    dns_server	    = "${var.dns_server}"
    ntp_server	    = "${var.ntp_server}"
    timezone	    = "${var.timezone}"
    admin_user      = "${var.uname}"
    admin_password  = "${var.upassword}"
  }
}

data "template_file" "vm02_do_json" {
  template = "${file("${path.module}/cluster.json")}"

  vars {
    #Uncomment the following line for BYOL
    regkey         = "${var.license2}"

    host1           = "${var.host1_name}"
    host2           = "${var.host2_name}"
    local_host      = "${var.host2_name}"
    local_selfip1   = "${var.f5vm02ext}"
    local_selfip2   = "${var.f5vm02tosrv}"
    local_selfip3   = "${var.f5vm02frsrv}"
    tosrvfl1       = "${var.f5vm01tosrvfl}"
    tosrvfl2       = "${var.f5vm02tosrvfl}"
    frsrvfl1       = "${var.f5vm01frsrvfl}"
    frsrvfl2       = "${var.f5vm02frsrvfl}"
    remote_selfip   = "${var.f5vm01ext}"
    gateway         = "${local.ext_gw}"
    dns_server      = "${var.dns_server}"
    ntp_server      = "${var.ntp_server}"
    timezone        = "${var.timezone}"
    admin_user      = "${var.uname}"
    admin_password  = "${var.upassword}"
  }
}
```
#### 7. Create the Application Services 3 (AS3) JSON payload for F5 VM01 and VM02. When AS3 is executed later it will create the F5 virtual server, pool, node, and client SSL profile.  
Take the JSON file as3.json from this repository and render it as a template file called "as3_json" using some variables from variables.tf.
```
data "template_file" "as3_json" {
  template = "${file("${path.module}/as3.json")}"

  vars {
    rg_name	    = "${azurerm_resource_group.main.name}"
    subscription_id = "${var.SP["subscription_id"]}"
    tenant_id	    = "${var.SP["tenant_id"]}"
    client_id	    = "${var.SP["client_id"]}"
    client_secret   = "${var.SP["client_secret"]}"
  }
}
```
#### 8. Create F5 VMs. During creation, copy the "vm_onboard" script created in Step#5. to the F5 filesystem at /var/lib/waagent/CustomData. 
```
resource "azurerm_virtual_machine" "f5vm01" {
  name                         = "${var.prefix}-f5vm01"
  location                     = "${azurerm_resource_group.main.location}"
  resource_group_name          = "${azurerm_resource_group.main.name}"
  primary_network_interface_id = "${azurerm_network_interface.vm01-mgmt-nic.id}"
  network_interface_ids        = ["${azurerm_network_interface.vm01-mgmt-nic.id}", "${azurerm_network_interface.vm01-ext-nic.id}", "${azurerm_network_interface.vm01-tosrv-nic.id}", "${azurerm_network_interface.vm01-frsrv-nic.id}"]
  vm_size                      = "${var.instance_type}"
  availability_set_id          = "${azurerm_availability_set.avset.id}"

  # Uncomment this line to delete the OS disk automatically when deleting the VM
  delete_os_disk_on_termination = true


  # Uncomment this line to delete the data disks automatically when deleting the VM
  delete_data_disks_on_termination = true

  storage_image_reference {
    publisher = "f5-networks"
    offer     = "${var.product}"
    sku       = "${var.image_name}"
    version   = "${var.bigip_version}"
  }

  storage_os_disk {
    name              = "${var.prefix}vm01-osdisk"
    caching           = "ReadWrite"
    create_option     = "FromImage"
    managed_disk_type = "Standard_LRS"
    disk_size_gb      = "80"
  }

  os_profile {
    computer_name  = "${var.prefix}vm01"
    admin_username = "${var.uname}"
    admin_password = "${var.upassword}"
    custom_data    = "${data.template_file.vm_onboard.rendered}"
  }

  os_profile_linux_config {
    disable_password_authentication = false
  }

  plan {
    name          = "${var.image_name}"
    publisher     = "f5-networks"
    product       = "${var.product}"
  }

  tags {
    Name           = "${var.environment}-f5vm01"
    environment    = "${var.environment}"
    owner          = "${var.owner}"
    group          = "${var.group}"
    costcenter     = "${var.costcenter}"
    application    = "${var.application}"
  }
}

resource "azurerm_virtual_machine" "f5vm02" {
  name                         = "${var.prefix}-f5vm02"
  location                     = "${azurerm_resource_group.main.location}"
  resource_group_name          = "${azurerm_resource_group.main.name}"
  primary_network_interface_id = "${azurerm_network_interface.vm02-mgmt-nic.id}"
  network_interface_ids        = ["${azurerm_network_interface.vm02-mgmt-nic.id}", "${azurerm_network_interface.vm02-ext-nic.id}", "${azurerm_network_interface.vm02-tosrv-nic.id}", "${azurerm_network_interface.vm02-frsrv-nic.id}"]
  vm_size                      = "${var.instance_type}"
  availability_set_id          = "${azurerm_availability_set.avset.id}"

  # Uncomment this line to delete the OS disk automatically when deleting the VM
  delete_os_disk_on_termination = true


  # Uncomment this line to delete the data disks automatically when deleting the VM
  delete_data_disks_on_termination = true

  storage_image_reference {
    publisher = "f5-networks"
    offer     = "${var.product}"
    sku       = "${var.image_name}"
    version   = "${var.bigip_version}"
  }

  storage_os_disk {
    name              = "${var.prefix}vm02-osdisk"
    caching           = "ReadWrite"
    create_option     = "FromImage"
    managed_disk_type = "Standard_LRS"
    disk_size_gb      = "80"
  }

  os_profile {
    computer_name  = "${var.prefix}vm02"
    admin_username = "${var.uname}"
    admin_password = "${var.upassword}"
    custom_data    = "${data.template_file.vm_onboard.rendered}"
}

  os_profile_linux_config {
    disable_password_authentication = false
  }

  plan {
    name          = "${var.image_name}"
    publisher     = "f5-networks"
    product       = "${var.product}"
  }

  tags {
    Name           = "${var.environment}-f5vm02"
    environment    = "${var.environment}"
    owner          = "${var.owner}"
    group          = "${var.group}"
    costcenter     = "${var.costcenter}"
    application    = "${var.application}"
  }
}
```
#### 9. Create the Nginx Load Balancer (Ubuntu VM) 
After boot install docker and run the custom docker container megamattzilla/nginx-lb which is pre-loaded with required configuration and hosted on docker hub. 
```
# Nginx Load Balancer VM
resource "azurerm_virtual_machine" "nginxlb01" {
    name                  = "nginxlb01"
    location                     = "${azurerm_resource_group.main.location}"
    resource_group_name          = "${azurerm_resource_group.main.name}"

    network_interface_ids = ["${azurerm_network_interface.nginxlb01-ext-nic.id}"]
    vm_size               = "Standard_DS3_v2"

    storage_os_disk {
        name              = "nginxlb01Disk"
        caching           = "ReadWrite"
        create_option     = "FromImage"
        managed_disk_type = "Premium_LRS"
    }

    storage_image_reference {
        publisher = "Canonical"
        offer     = "UbuntuServer"
        sku       = "16.04.0-LTS"
        version   = "latest"
    }

    os_profile {
        computer_name  = "nginxlb01"
        admin_username = "azureuser"
        admin_password = "${var.upassword}"
        custom_data = <<-EOF
              #!/bin/bash
              apt-get update -y
              apt-get install -y docker.io
              docker run -d -p 80:80 --net=host --restart unless-stopped megamattzilla/nginx-lb
              EOF
    }

    os_profile_linux_config {
        disable_password_authentication = false
    }

  tags {
    Name           = "${var.environment}-nginxlb01"
    environment    = "${var.environment}"
    owner          = "${var.owner}"
    group          = "${var.group}"
    costcenter     = "${var.costcenter}"
    application    = "${var.application}"
  }
}
```
#### 10. Create the Nginx Application Server (Ubuntu VM) 
After boot install docker and run the custom docker containers megamattzilla/nginx-node1 and nginx-node2 which is pre-loaded with required configuration and hosted on docker hub. 
```
# Nginx App Server VM
resource "azurerm_virtual_machine" "nginxapp01" {
    name                  = "nginxapp01"
    location                     = "${azurerm_resource_group.main.location}"
    resource_group_name          = "${azurerm_resource_group.main.name}"

    network_interface_ids = ["${azurerm_network_interface.nginxapp01-ext-nic.id}"]
    vm_size               = "Standard_DS3_v2"

    storage_os_disk {
        name              = "nginxapp01Disk"
        caching           = "ReadWrite"
        create_option     = "FromImage"
        managed_disk_type = "Premium_LRS"
    }

    storage_image_reference {
        publisher = "Canonical"
        offer     = "UbuntuServer"
        sku       = "16.04.0-LTS"
        version   = "latest"
    }

    os_profile {
        computer_name  = "nginxapp01"
        admin_username = "azureuser"
        admin_password = "${var.upassword}"
        custom_data = <<-EOF
              #!/bin/bash
              apt-get update -y
              apt-get install -y docker.io
              docker run -d -p 9001:9001 --name=nginx-node1 --restart unless-stopped megamattzilla/nginx-node1
              docker run -d -p 9002:9002 --name=nginx-node2 --restart unless-stopped megamattzilla/nginx-node2
              EOF
    }

    os_profile_linux_config {
        disable_password_authentication = false
    }

  tags {
    Name           = "${var.environment}-nginxapp01"
    environment    = "${var.environment}"
    owner          = "${var.owner}"
    group          = "${var.group}"
    costcenter     = "${var.costcenter}"
    application    = "${var.application}"
  }
}
```
#### 11. Execute the "vm_onboard" script on each F5 Big-IP after boot. This will install and activate DO and AS3 modules. 
```
# Run Startup Script
resource "azurerm_virtual_machine_extension" "f5vm01-run-startup-cmd" {
  name                 = "${var.environment}-f5vm01-run-startup-cmd"
  depends_on           = ["azurerm_virtual_machine.f5vm01", "azurerm_virtual_machine.nginxlb01"]
  location             = "${var.region}"
  resource_group_name  = "${azurerm_resource_group.main.name}"
  virtual_machine_name = "${azurerm_virtual_machine.f5vm01.name}"
  publisher            = "Microsoft.OSTCExtensions"
  type                 = "CustomScriptForLinux"
  type_handler_version = "1.2"
  # publisher            = "Microsoft.Azure.Extensions"
  # type                 = "CustomScript"
  # type_handler_version = "2.0"

  settings = <<SETTINGS
    {
        "commandToExecute": "bash /var/lib/waagent/CustomData"
    }
  SETTINGS

  tags {
    Name           = "${var.environment}-f5vm01-startup-cmd"
    environment    = "${var.environment}"
    owner          = "${var.owner}"
    group          = "${var.group}"
    costcenter     = "${var.costcenter}"
    application    = "${var.application}"
  }
}

resource "azurerm_virtual_machine_extension" "f5vm02-run-startup-cmd" {
  name                 = "${var.environment}-f5vm02-run-startup-cmd"
  depends_on           = ["azurerm_virtual_machine.f5vm02", "azurerm_virtual_machine.nginxlb01"]
  location             = "${var.region}"
  resource_group_name  = "${azurerm_resource_group.main.name}"
  virtual_machine_name = "${azurerm_virtual_machine.f5vm02.name}"
  publisher            = "Microsoft.OSTCExtensions"
  type                 = "CustomScriptForLinux"
  type_handler_version = "1.2"
  # publisher            = "Microsoft.Azure.Extensions"
  # type                 = "CustomScript"
  # type_handler_version = "2.0"

  settings = <<SETTINGS
    {
        "commandToExecute": "bash /var/lib/waagent/CustomData"
    }
  SETTINGS

  tags {
    Name           = "${var.environment}-f5vm02-startup-cmd"
    environment    = "${var.environment}"
    owner          = "${var.owner}"
    group          = "${var.group}"
    costcenter     = "${var.costcenter}"
    application    = "${var.application}"
  }
}
```
#### 12. Send Declarative Onboarding json payload to F5 VMs API endpoint
```
resource "null_resource" "f5vm01-run-REST" {
  depends_on	= ["azurerm_virtual_machine_extension.f5vm01-run-startup-cmd"]
  # Running DO REST API
  provisioner "local-exec" {
    command = <<-EOF
      #!/bin/bash
      curl -k -X GET https://${data.azurerm_public_ip.vm01mgmtpip.ip_address}${var.rest_do_uri} \
              -H "Content-Type: application/json" \
              -u ${var.uname}:${var.upassword}
      sleep 15
      curl -k -X ${var.rest_do_method} https://${data.azurerm_public_ip.vm01mgmtpip.ip_address}${var.rest_do_uri} \
              -H "Content-Type: application/json" \
	      -u ${var.uname}:${var.upassword} \
	      -d @${var.rest_vm01_do_file} 
    EOF
  }
  
  resource "null_resource" "f5vm02-run-REST" {
  depends_on	= ["azurerm_virtual_machine_extension.f5vm02-run-startup-cmd"]
  # Running DO REST API
  provisioner "local-exec" {
    command = <<-EOF
      #!/bin/bash
      curl -k -X GET https://${data.azurerm_public_ip.vm02mgmtpip.ip_address}${var.rest_do_uri} \
              -H "Content-Type: application/json" \
              -u ${var.uname}:${var.upassword}
      sleep 15
      curl -k -X ${var.rest_do_method} https://${data.azurerm_public_ip.vm02mgmtpip.ip_address}${var.rest_do_uri} \
              -H "Content-Type: application/json" \
	      -u ${var.uname}:${var.upassword} \
	      -d @${var.rest_vm02_do_file}
    EOF
  }
```
#### 13. Send Application Service 3 json payload to F5 VMs API endpoint
```
# Running AS3 REST API
  provisioner "local-exec" {
    command = <<-EOF
      #!/bin/bash
      curl -k -X ${var.rest_as3_method} https://${data.azurerm_public_ip.vm01mgmtpip.ip_address}${var.rest_as3_uri} \
              -H "Content-Type: application/json" \
	      -u ${var.uname}:${var.upassword} \
	      -d @${var.rest_vm_as3_file}
    EOF
  }
}

 # Running AS3 REST API
  provisioner "local-exec" {
    command = <<-EOF
      #!/bin/bash
      curl -k -X ${var.rest_as3_method} https://${data.azurerm_public_ip.vm02mgmtpip.ip_address}${var.rest_as3_uri} \
              -H "Content-Type: application/json" \
	      -u ${var.uname}:${var.upassword} \
	      -d @${var.rest_vm_as3_file}
    EOF
  }
}
```
## The main.tf is now fully executed, and upon successful deployment will build all required resources for a two-tier F5 and Nginx deployment. 




  
