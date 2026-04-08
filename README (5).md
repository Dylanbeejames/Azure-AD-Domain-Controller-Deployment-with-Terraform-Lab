# Infrastructure as Code with Terraform — Azure AD Domain Controller Lab

**Author:** Dylan Bryson  
**Role:** Cloud Security Specialist  
**Video Walkthrough:** [▶ Watch on Loom](https://www.loom.com/share/6cce304ef6934a0a8f58d73d218c311e)

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Technologies Used](#technologies-used)
- [Prerequisites](#prerequisites)
- [Part 1 — Scaffold the Project](#part-1--scaffold-the-project)
- [Part 2 — Terraform Configuration Files](#part-2--terraform-configuration-files)
- [Part 3 — Deploy](#part-3--deploy)
- [Part 4 — Connect via RDP](#part-4--connect-via-rdp)
- [Part 5 — Verify AD DS](#part-5--verify-ad-ds)
- [Troubleshooting](#troubleshooting)
- [Teardown](#teardown)
- [Summary](#summary)
- [Next Steps](#next-steps)

---

## Overview

This lab demonstrates deploying Azure infrastructure using **Terraform** as Infrastructure as Code (IaC) — replacing manual portal clicks with declarative configuration files. Rather than provisioning resources one by one through the Azure Portal, the entire environment is described in code and deployed in a single `terraform apply`.

The lab goes beyond basic networking: it provisions a **Windows Server 2022 VM** and uses a **Terraform Custom Script Extension** to automatically install the **Active Directory Domain Services (AD DS)** role and promote the server to a new forest — all without manual intervention.

By the end, the following are fully operational and reproducible from code:

- Resource Group, VNet, Subnet, NSG, NIC, and Public IP
- Windows Server 2022 Domain Controller
- Active Directory forest with DNS

---

## Architecture

```
Public Internet
      │
      ▼ RDP (port 3389)
Public IP — pip-ad-dylan (Static)
      │
      ▼
NSG — nsg-ad-dylan (Allow RDP inbound)
      │
      ▼
NIC — nic-ad-dylan (Static private IP: 10.0.1.4)
      │
      ▼
Subnet — snet-ad (10.0.1.0/24)
      │
      ▼
VNet — vnet-ad-dylan (10.0.0.0/16)
      │
      ▼
Windows Server 2022 VM — vm-ad-dylan
      │
      ▼ (Custom Script Extension)
Active Directory Domain Controller
Domain: corp.dylan.com | NetBIOS: CORP
```

---

## Technologies Used

| Technology | Purpose |
|------------|---------|
| Terraform (v1.3+) | Infrastructure as Code provisioning |
| Azure CLI | Authentication and subscription context |
| Azure Resource Group | Logical container for all resources |
| Azure Virtual Network & Subnet | Network isolation |
| Azure NSG | Inbound RDP access control |
| Azure Public IP (Standard, Static) | RDP access endpoint |
| Windows Server 2022 Datacenter | Domain Controller OS |
| Custom Script Extension | Automated AD DS installation via PowerShell |
| Active Directory Domain Services | Domain and DNS services |

---

## Prerequisites

Before starting, ensure the following are in place:

- Azure CLI installed and authenticated: `az login`
- Terraform v1.3+ installed
- An active Azure subscription
- A local directory to store Terraform files

---

## Part 1 — Scaffold the Project

Run the following to create the project directory and all four Terraform files in one command:

```bash
mkdir -p ~/repos/az-ad-vm && cd ~/repos/az-ad-vm && touch main.tf variables.tf outputs.tf terraform.tfvars
```

---

## Part 2 — Terraform Configuration Files

### `variables.tf`

Declares all input variables used across the configuration:

```hcl
variable "yourname" {
  description = "Your name — used to make resource names unique."
  type        = string
}

variable "location" {
  description = "Azure region."
  type        = string
  default     = "eastus"
}

variable "admin_password" {
  description = "Local admin password for the VM."
  type        = string
  sensitive   = true
}

variable "dsrm_password" {
  description = "Directory Services Restore Mode password for AD DS."
  type        = string
  sensitive   = true
}

variable "domain_name" {
  description = "Fully qualified domain name (e.g. corp.example.com)."
  type        = string
  default     = "corp.example.com"
}

variable "domain_netbios" {
  description = "NetBIOS name for the domain (max 15 characters)."
  type        = string
  default     = "CORP"
}

variable "tags" {
  description = "Tags to apply to all resources."
  type        = map(string)
  default = {
    project = "ad-lab"
  }
}
```

---

### `terraform.tfvars`

Assigns values to the declared variables. **Update these before running `terraform apply`:**

```hcl
yourname       = "dylan"
location       = "eastus"
admin_password = "YourPassword123!"
dsrm_password  = "YourDSRMPassword123!"
domain_name    = "corp.dylan.com"
domain_netbios = "CORP"
```

> **Security Note:** Never commit `terraform.tfvars` to a public repository — it contains sensitive passwords. Add it to `.gitignore`. The `dsrm_password` (Directory Services Restore Mode) is separate from the admin password and is only required for AD recovery operations. Store it securely — it cannot be retrieved after deployment.

---

### `main.tf`

Defines all Azure resources and the Custom Script Extension that installs AD DS:

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "main" {
  name     = "rg-ad-${var.yourname}"
  location = var.location
  tags     = var.tags
}

resource "azurerm_virtual_network" "main" {
  name                = "vnet-ad-${var.yourname}"
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name
  address_space       = ["10.0.0.0/16"]
  tags                = var.tags
}

resource "azurerm_subnet" "main" {
  name                 = "snet-ad"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.1.0/24"]
}

resource "azurerm_public_ip" "main" {
  name                = "pip-ad-${var.yourname}"
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name
  allocation_method   = "Static"
  sku                 = "Standard"
  tags                = var.tags
}

resource "azurerm_network_security_group" "main" {
  name                = "nsg-ad-${var.yourname}"
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name

  security_rule {
    name                       = "allow-rdp"
    priority                   = 1000
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "3389"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }

  tags = var.tags
}

resource "azurerm_network_interface" "main" {
  name                = "nic-ad-${var.yourname}"
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.main.id
    private_ip_address_allocation = "Static"
    private_ip_address            = "10.0.1.4"
    public_ip_address_id          = azurerm_public_ip.main.id
  }

  tags = var.tags
}

resource "azurerm_network_interface_security_group_association" "main" {
  network_interface_id      = azurerm_network_interface.main.id
  network_security_group_id = azurerm_network_security_group.main.id
}

resource "azurerm_windows_virtual_machine" "main" {
  name                  = "vm-ad-${var.yourname}"
  computer_name         = "ad-${var.yourname}"
  location              = var.location
  resource_group_name   = azurerm_resource_group.main.name
  size                  = "Standard_D2s_v3"
  admin_username        = "adadmin"
  admin_password        = var.admin_password
  network_interface_ids = [azurerm_network_interface.main.id]

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Premium_LRS"
    disk_size_gb         = 127
  }

  source_image_reference {
    publisher = "MicrosoftWindowsServer"
    offer     = "WindowsServer"
    sku       = "2022-Datacenter"
    version   = "latest"
  }

  additional_unattend_content {
    content = "<AutoLogon><Password><Value>${var.admin_password}</Value></Password><Enabled>true</Enabled><LogonCount>1</LogonCount><Username>adadmin</Username></AutoLogon>"
    setting = "AutoLogon"
  }

  tags = var.tags
}

resource "azurerm_virtual_machine_extension" "ad_setup" {
  name                 = "install-ad-ds"
  virtual_machine_id   = azurerm_windows_virtual_machine.main.id
  publisher            = "Microsoft.Compute"
  type                 = "CustomScriptExtension"
  type_handler_version = "1.10"

  settings = jsonencode({
    commandToExecute = "powershell -ExecutionPolicy Unrestricted -Command \"Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools; Import-Module ADDSDeployment; Install-ADDSForest -DomainName '${var.domain_name}' -DomainNetbiosName '${var.domain_netbios}' -ForestMode 'WinThreshold' -DomainMode 'WinThreshold' -InstallDns:$true -SafeModeAdministratorPassword (ConvertTo-SecureString '${var.dsrm_password}' -AsPlainText -Force) -Force:$true\""
  })

  tags = var.tags
}
```

---

### `outputs.tf`

Exposes key values after deployment:

```hcl
output "public_ip" {
  description = "Public IP — use this to RDP into the domain controller"
  value       = azurerm_public_ip.main.ip_address
}

output "domain_name" {
  description = "Active Directory domain name"
  value       = var.domain_name
}

output "admin_username" {
  description = "Local admin username"
  value       = "adadmin"
}
```

---

## Part 3 — Deploy

Run these commands from the `~/repos/az-ad-vm` directory:

```bash
terraform init
terraform plan
terraform apply
```

Type `yes` when prompted to confirm.

> **Timing:** `terraform apply` takes approximately 5–8 minutes for resource provisioning. The Custom Script Extension then runs the AD DS installation, adding another 3–5 minutes and triggering an **automatic VM reboot**. Wait 5–10 minutes after apply completes before attempting to RDP.

---

## Part 4 — Connect via RDP

Retrieve the public IP after deployment:

```bash
terraform output public_ip
```

Open **Remote Desktop Connection** and use one of the following credential formats:

| Method | Username | When to Use |
|--------|----------|-------------|
| Domain prefix | `CORP\adadmin` | Standard — use this first |
| UPN format | `adadmin@corp.dylan.com` | If domain prefix fails |
| Local account | `.\adadmin` | Only if AD promotion failed |

> **Note:** After AD DS installs, the VM authenticates using domain credentials. Local account syntax (`.\adadmin`) will only work if the domain promotion did not complete successfully.

---

## Part 5 — Verify AD DS

Once connected via RDP, open **PowerShell as Administrator** and run the following:

```powershell
# Verify AD DS service is running
Get-Service NTDS | Select-Object Name, Status

# Confirm domain configuration
Get-ADDomain

# List domain controllers
Get-ADDomainController -Filter *

# Test DNS resolution
Resolve-DnsName corp.dylan.com
```

All four commands returning without errors confirms the domain controller and DNS are fully operational. `Get-ADDomain` will display the complete domain configuration including forest and domain functional levels.

---

## Troubleshooting

Check the Custom Script Extension provisioning status from your local machine:

```bash
az vm extension show \
  --resource-group rg-ad-dylan \
  --vm-name vm-ad-dylan \
  --name install-ad-ds \
  --query "provisioningState" \
  --output tsv
```

| Issue | Cause | Resolution |
|-------|-------|------------|
| RDP password not working | VM rebooted after domain promotion — auth now requires domain credentials | Use `CORP\adadmin` or `adadmin@corp.dylan.com` |
| Extension status: Failed | AD DS installation failed mid-run | Check `C:\WindowsAzure\Logs` on the VM. Re-run `terraform apply` to retry the extension |
| RDP black screen | VM still rebooting after AD DS install | Wait 5 minutes and retry |
| `Get-ADDomain` not found | AD DS module not loaded | Run `Import-Module ActiveDirectory` then retry |

---

## Teardown

Destroy all resources when done to avoid ongoing charges:

```bash
terraform destroy
```

Type `yes` to confirm. This removes the resource group and everything inside it — VM, disks, NIC, public IP, NSG, VNet, and subnet.

> ⚠️ This action is permanent. Ensure you no longer need the environment before confirming.

---

## Summary

This lab demonstrates a complete IaC-driven deployment of an Active Directory Domain Controller on Azure:

- ✅ Replaced manual portal provisioning with declarative Terraform configuration
- ✅ Deployed full Azure networking stack (VNet, subnet, NSG, NIC, public IP) as code
- ✅ Provisioned a Windows Server 2022 VM with a static private IP
- ✅ Automated AD DS installation and forest promotion via Custom Script Extension
- ✅ Verified domain controller health and DNS resolution via PowerShell
- ✅ Cleaned up all resources with a single `terraform destroy`

---

## Next Steps

| Enhancement | Description |
|-------------|-------------|
| **NSG Hardening** | Restrict RDP (port 3389) to specific source IPs instead of open access |
| **Azure Bastion** | Replace public RDP exposure with secure browser-based access — no public IP required |
| **Terraform Remote State** | Store `terraform.tfstate` in Azure Blob Storage for team collaboration and state locking |
| **Join a Member VM** | Deploy a second VM and domain-join it to `corp.dylan.com` |
| **Azure AD Connect** | Sync the on-prem AD forest with Azure Active Directory (Entra ID) |
| **Key Vault for Secrets** | Move `admin_password` and `dsrm_password` out of `tfvars` and into Azure Key Vault |

---

*© Dylan Bryson — Cloud Security Specialist*
