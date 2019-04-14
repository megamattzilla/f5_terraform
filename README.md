# Fork of Azure_Secure_Enclave_Public_DMZ modified for a two tier F5 and Nginx deployment

* Azure_Secure_Enclave_Public_DMZ - This Terraform template uses the Azurerm provider to build out all the necessary Azure objects and then use F5 Declarative Onboarding and AS3 REST orchestration tool for initial onboarding and configuration 

![alt text](https://github.com/megamattzilla/f5_terraform/blob/master/nginx-f5-deployment.png "F5 and Nginx Deployment")

## To get started, clone this repository to a system that has terraform installed

## Edit variables

variables.tf contains a few variables that need modified for each deployment.

1. Enter your Azure subscription and authentication information:
```
# Azure Environment
variable "SP" {
	type = "map"
	default = {
		subscription_id = "xxxxx" 
		client_id       = "xxxxx"
		client_secret   = "xxxxx"
		tenant_id       = "xxxxx"
	}
}
```
2. Edit the prefix name used for the resource group creation:
```
variable prefix	{ default = "my-deployment-name" }
```
3. Create the username and password for the F5 and Nginx servers:
```
variable uname	{ default = "azureuser" }
variable upassword	{ default = "Default12345" }
```
4. Add the F5 eval licenses:
```
variable license1             { default = "xxxxx" }
variable license2             { default = "xxxxx" }
```
If you would rather use Pay as you go licensing leave these as blank and comment-out the line in main.tf that mentions BYOL

## CD to the directory containing the repository .tf files and run terraform init:
`terraform init`
You should see output similar to the following:
```
Initializing provider plugins...

The following providers do not have any version constraints in configuration,
so the latest version was installed.

To prevent automatic upgrades to new major versions that may contain breaking
changes, it is recommended to add version = "..." constraints to the
corresponding provider blocks in configuration, with the constraint strings
suggested below.

* provider.azurerm: version = "~> 1.24"
* provider.local: version = "~> 1.2"
* provider.null: version = "~> 2.1"
* provider.template: version = "~> 2.1"

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```
If you get an error about a failure to download null provider- try agian in a few minutes.

## Apply the template by running terraform apply
`terraform apply`
Enter `yes` at the prompt. This will take ~15 minutes to build. 

At the end you should see output indicating the build was successfull:
`Apply complete! Resources: 47 added, 0 changed, 0 destroyed.`

Note the output for ALB_app1_pip which will be an IP Address

## In a web browser, navigate to page https://<ALB_app1_pip IP Address> 

