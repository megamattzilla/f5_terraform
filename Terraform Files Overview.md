## Terraform Files Overview

### **provider.tf** instructs terraform to use the azure resource manager plugin for terraform. The Azure variables required are located in **variables.tf** 

```# Configure the Microsoft Azure Provider, replace Service Principal and Subscription with your own
provider "azurerm" {
    subscription_id = "${var.SP["subscription_id"]}"
    client_id       = "${var.SP["client_id"]}"
    client_secret   = "${var.SP["client_secret"]}"
    tenant_id       = "${var.SP["tenant_id"]}"
}
```

### **main.tf** contains the code to create the Azure networking components, F5, and Nginx resouces. 

