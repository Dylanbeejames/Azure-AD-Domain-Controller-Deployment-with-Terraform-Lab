# Azure VM Lab — Terraform & Active Directory Domain Controller

**Author:** Dylan  
**Video Walkthrough:** [▶ Watch on Loom](https://www.loom.com/share/0d61ee22ebba4e7b854ae1cd04fc1187)

---

## Table of Contents

- [Overview](#overview)
- [What Was Deployed](#what-was-deployed)
- [Technologies Used](#technologies-used)
- [Phase 1 — Project Setup](#phase-1--project-setup)
- [Phase 2 — Terraform Installation](#phase-2--terraform-installation)
- [Phase 3 — Configure Terraform Files](#phase-3--configure-terraform-files)
- [Phase 4 — Authenticate to Azure](#phase-4--authenticate-to-azure)
- [Phase 5 — Initialize Terraform](#phase-5--initialize-terraform)
- [Phase 6 — Preview the Plan](#phase-6--preview-the-plan)
- [Phase 7 — Deploy Infrastructure](#phase-7--deploy-infrastructure)
- [Phase 8 — Retrieve Public IP](#phase-8--retrieve-public-ip)
- [Phase 9 — Connect via RDP](#phase-9--connect-via-rdp)
- [Phase 10 — Validate AD DS](#phase-10--validate-ad-ds)
- [Phase 11 — Cleanup](#phase-11--cleanup)
- [Summary](#summary)
- [Next Steps](#next-steps)

---

## Overview

This lab demonstrates how to deploy a **Windows Server 2022 VM** in Azure using **Terraform** and configure it as an **Active Directory Domain Controller (AD DC)**.

It covers a full Infrastructure as Code (IaC) workflow — from environment setup and resource provisioning, all the way through AD DS validation and teardown. All resources are defined in code, making the deployment fully repeatable and version-controlled.

---

## What Was Deployed

| Resource | Purpose |
|----------|---------|
| Resource Group | Container for all lab resources |
| Virtual Network (VNet) & Subnet | Private network for the VM |
| Network Security Group (NSG) | Allowed inbound RDP (port 3389) |
| Public IP Address | External access endpoint |
| Network Interface (NIC) | Attached VM to subnet and public IP |
| Windows Server 2022 VM | Promoted to Active Directory Domain Controller |
| Custom Script Extension | Automated AD DS and DNS installation |
| Terraform Outputs | Exposed public IP, domain name, and admin credentials |

---

## Technologies Used

| Technology | Purpose |
|------------|---------|
| Terraform | Infrastructure as Code provisioning |
| Azure CLI | Authentication and subscription management |
| Azure Virtual Network | Network isolation for the VM |
| Azure NSG | Port-level traffic control |
| Windows Server 2022 | Domain Controller OS |
| Active Directory Domain Services (AD DS) | Domain and DNS services |
| PowerShell | Local setup and AD validation |
| Remote Desktop Protocol (RDP) | VM access post-deployment |

---

## Phase 1 — Project Setup

Create the project directory and Terraform configuration files using PowerShell:

```powershell
mkdir C:\Users\17576\repos\az-ad-vm
cd C:\Users\17576\repos\az-ad-vm

New-Item -Name main.tf       -ItemType File
New-Item -Name variables.tf  -ItemType File
New-Item -Name outputs.tf    -ItemType File
New-Item -Name terraform.tfvars -ItemType File
```

> Each file serves a distinct role in the Terraform configuration — see [Phase 3](#phase-3--configure-terraform-files) for details.

---

## Phase 2 — Terraform Installation

Download Terraform from [terraform.io](https://www.terraform.io/downloads) and add the binary to your system `PATH`.

Verify the installation:

```bash
terraform -v
```

---

## Phase 3 — Configure Terraform Files

| File | Purpose |
|------|---------|
| `main.tf` | Defines all Azure resources and the custom script for AD DS installation |
| `variables.tf` | Declares input variable names and types |
| `terraform.tfvars` | Assigns values to variables (passwords, domain name, etc.) |
| `outputs.tf` | Exposes key values post-deploy (public IP, domain, credentials) |

> **Security Note:** Never commit `terraform.tfvars` to a public repository if it contains passwords or sensitive values. Add it to `.gitignore`.

---

## Phase 4 — Authenticate to Azure

Log in to your Azure account via the CLI:

```bash
az login
```

This opens a browser for authentication and sets the active subscription context for Terraform to use.

---

## Phase 5 — Initialize Terraform

Initialize the working directory and download the required Azure provider plugins:

```bash
terraform init
```

---

## Phase 6 — Preview the Plan

Review the resources Terraform will create before applying:

```bash
terraform plan
```

This is a dry run — no resources are created at this stage.

---

## Phase 7 — Deploy Infrastructure

Apply the configuration to provision all resources:

```bash
terraform apply
```

Type `yes` when prompted to confirm. Terraform will:

1. Create the resource group, VNet, subnet, NSG, public IP, and NIC
2. Deploy the Windows Server 2022 VM
3. Run the Custom Script Extension to install AD DS and DNS
4. Automatically reboot the VM after domain controller promotion

---

## Phase 8 — Retrieve Public IP

Once deployment completes, retrieve the VM's public IP:

```bash
terraform output public_ip
```

---

## Phase 9 — Connect via RDP

Open **Remote Desktop Connection** and connect using the public IP. Use one of the following login formats:

| Method | Username |
|--------|----------|
| Domain prefix | `CORP\adadmin` |
| UPN format | `adadmin@corp.dylan.com` |
| Local fallback | `.\adadmin` |

---

## Phase 10 — Validate AD DS

Once connected via RDP, open **PowerShell** and run the following to confirm the domain controller and DNS are operational:

```powershell
# Verify NTDS service is running
Get-Service NTDS | Select-Object Name, Status

# Confirm domain configuration
Get-ADDomain

# List domain controllers
Get-ADDomainController -Filter *

# Test DNS resolution
Resolve-DnsName corp.dylan.com
```

All commands returning valid output confirms the Active Directory Domain Controller is fully functional.

---

## Phase 11 — Cleanup

Destroy all provisioned resources to avoid ongoing Azure charges:

```bash
terraform destroy
```

Type `yes` to confirm. Terraform will remove every resource defined in the configuration.

> ⚠️ This is irreversible. Ensure you no longer need any resources before confirming.

---

## Summary

This lab demonstrates a complete IaC-driven deployment of an Active Directory environment on Azure:

- ✅ Installed and configured Terraform CLI on Windows
- ✅ Authenticated to Azure via CLI
- ✅ Deployed a VM and full networking stack with Terraform
- ✅ Automated AD DS and DNS installation via Custom Script Extension
- ✅ Validated domain controller services and DNS resolution
- ✅ Cleaned up all resources efficiently with `terraform destroy`

---

## Next Steps

| Enhancement | Description |
|-------------|-------------|
| **Azure Bastion** | Replace public RDP exposure with secure browser-based access |
| **Terraform Remote State** | Store `terraform.tfstate` in Azure Blob Storage for team use |
| **Join a Member VM** | Deploy a second VM and domain-join it to `corp.dylan.com` |
| **Azure AD Connect** | Sync on-prem AD with Azure Active Directory |
| **NSG Hardening** | Restrict RDP source to specific IP ranges instead of open access |

---

*© Dylan — Cloud Security Specialist*
