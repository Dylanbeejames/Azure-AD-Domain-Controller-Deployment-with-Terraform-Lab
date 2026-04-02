Watch Me Build The Lab HERE

https://www.loom.com/share/0d61ee22ebba4e7b854ae1cd04fc1187

Azure VM Lab – Terraform Walkthrough

This lab shows how to deploy a Windows Server 2022 VM in Azure using Terraform and configure it as an Active Directory Domain Controller.

It’s a full workflow from setup to cleanup using Infrastructure as Code.

🏗️ What Was Deployed

Using Terraform, the lab created:

Resource Group – container for all resources
Virtual Network (VNet) & Subnet – private network for the VM
Network Security Group (NSG) – allowed RDP (port 3389)
Public IP Address – to connect to the VM
Network Interface (NIC) – attached VM to subnet and public IP
Windows Server 2022 VM – promoted to Active Directory Domain Controller
Custom Script Extension – automatically installed AD DS and DNS
Terraform Outputs – public IP, domain name, and admin credentials
⚡ Steps Performed
1️⃣ Project Setup

Created project folder and files in PowerShell:

mkdir C:\Users\17576\repos\az-ad-vm
cd C:\Users\17576\repos\az-ad-vm
New-Item -Name main.tf -ItemType File
New-Item -Name variables.tf -ItemType File
New-Item -Name outputs.tf -ItemType File
New-Item -Name terraform.tfvars -ItemType File
2️⃣ Terraform Installation
Downloaded Terraform and added it to PATH.
Verified installation:
terraform -v
3️⃣ Configure Terraform Files
main.tf – defines resources and custom script for AD DS
variables.tf – input variables
terraform.tfvars – assigned values (passwords, domain name)
outputs.tf – outputs like public IP
4️⃣ Authenticate to Azure
az login
5️⃣ Initialize Terraform
terraform init
6️⃣ Preview Plan
terraform plan
Checked what resources Terraform would create.
7️⃣ Deploy Infrastructure
terraform apply
Typed yes to confirm.
VM and all networking resources were created.
VM rebooted automatically after AD DS installation.
8️⃣ Get Public IP
terraform output public_ip
9️⃣ Connect via RDP
Open Remote Desktop and log in using:
Method	Username
Domain prefix	CORP\adadmin
UPN format	adadmin@corp.charles.com

Local account	.\adadmin (if needed)
🔟 Validate AD DS
Get-Service NTDS | Select-Object Name, Status
Get-ADDomain
Get-ADDomainController -Filter *
Resolve-DnsName corp.charles.com
Verified the domain controller and DNS were working.
1️⃣1️⃣ Cleanup
terraform destroy
Typed yes to remove all resources and avoid Azure charges.
✅ Summary

This lab demonstrates how to:

Install and use Terraform CLI on Windows
Authenticate to Azure via CLI
Deploy a VM and networking resources with Terraform
Configure a VM as an Active Directory Domain Controller
Verify AD DS services and domain functionality
Clean up all resources efficiently
