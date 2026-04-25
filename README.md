# Lab 01: Provision · Install AD DS · Promote to Domain Controller
 
![Platform](https://img.shields.io/badge/Platform-Windows%20Server%202025-blue?style=flat-square)
![Cloud](https://img.shields.io/badge/Cloud-Azure%20Free%20Tier-0078D4?style=flat-square&logo=microsoftazure)
![Cost](https://img.shields.io/badge/Estimated%20Cost-%240-brightgreen?style=flat-square)
![Duration](https://img.shields.io/badge/Duration-1--2%20hours-orange?style=flat-square)
 
---

Video Lab Link: https://www.loom.com/share/01d82aced167448990fa0926d917f916

---
 
## Overview
 
These three steps establish the entire foundation of the lab environment. By the end of Step 3, you will have a running Domain Controller that owns the `lab.local` forest, handles all authentication, and serves as the DNS authority for every machine that joins the domain.
 
---
 
## Architecture 
 
```
┌──────────────────────────────────────────────────────────┐
│                 Step 1 — Provision the VM                 │
│                                                          │
│   Option A: Azure                  Option B: VirtualBox  │
│   Standard_B2s · East US           4 GB RAM · 60 GB disk │
│                     │                    │               │
│                     └────────┬───────────┘               │
│                              │                           │
│                  RDP in · enable clipboard               │
└──────────────────────────────┬───────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────┐
│         Step 2 — Install Active Directory Domain Services │
│                                                          │
│   Server Manager UI              PowerShell              │
│   Add Roles and Features         Install-WindowsFeature  │
│   Check AD DS · add tools        AD-Domain-Services       │
│                     │                    │               │
│                     └────────┬───────────┘               │
│                              │                           │
│              Also install GPMC now (required for Step 5) │
└──────────────────────────────┬───────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────┐
│       Step 3 — Promote Server to Domain Controller        │
│                                                          │
│   Add a new forest → Root domain: lab.local              │
│   Set DSRM password → Install → Auto-restart             │
│                                                          │
│              lab.local forest is now live                │
│         Authoritative DNS · Authentication ready         │
└──────────────────────────────────────────────────────────┘
```
 
> **Key concept:** Installing the AD DS role (Step 2) and promoting to Domain Controller (Step 3) are two separate actions. The role install puts the software on the server. Promotion is what actually creates the domain, the forest, and the DNS zone. A server with AD DS installed but not promoted is not a Domain Controller.
 
---
 
## Step 1 — Provision the VM
 
### Option A: Azure (Recommended)
 
Azure removes the local hardware requirement entirely. You connect via RDP from your desktop and the VM runs in Microsoft's datacentre at no cost within the free tier.
 
1. Sign in to [portal.azure.com](https://portal.azure.com)
2. Search **Virtual machines** → **Create**
3. Configure per the table below → **Review + Create → Create**
| Setting | Value | Why |
|---|---|---|
| Region | East US | Lowest cost, best free-tier availability |
| Image | Windows Server 2025 Datacenter — Gen2 | Includes 180-day eval licence |
| Size | Standard_B2s (2 vCPU · 4 GB RAM) | Smallest size AD runs comfortably on |
| Authentication | Password | Used for initial RDP access |
| Public inbound ports | Allow RDP (3389) | Required for remote connection |
| OS disk | Standard SSD | Good performance, included in free tier |
 
> **Cost control:** Stop the VM at the end of every session (do not delete it). A B2s costs ~$0.05/hour running. Stopping it pauses compute billing. Your $200 free credit covers weeks of on-and-off lab work if you stop it each time.
 
#### Fix: Enable Clipboard Between Your Machine and the VM
 
Without this fix, you cannot copy and paste commands into the VM.
 
1. Open **Remote Desktop** on your local machine
2. Enter the VM's public IP
3. Click **Show Options → Local Resources**
4. Ensure **Clipboard** is checked under *Local devices and resources*
5. Click **Connect**
> Download the `.rdp` file from the Azure portal (**Connect → Download RDP File**) and open it with the native Remote Desktop app. The browser console does not support clipboard sharing reliably and is not suitable for lab work.
 
---
 
### Option B: VirtualBox (Local)
 
1. Download [VirtualBox](https://www.virtualbox.org) — free, no account needed
2. Download the [Windows Server 2025 Evaluation ISO](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2025)
3. New VM: **4 GB RAM minimum · 60 GB disk · Windows Server 2019/2022 type**
4. Mount the ISO → boot → follow the installation wizard
5. Select **Windows Server 2025 Datacenter with Desktop Experience**
> **Minimum host hardware:** 8 GB RAM (4 GB for the VM, 4 GB for your OS), 60 GB free disk, quad-core CPU with virtualisation enabled in BIOS. If you have less than 8 GB RAM, use the Azure option.
 
---
 
## Step 2 — Install Active Directory Domain Services
 
RDP into the VM. Server Manager opens automatically on login. All work from here is done inside the VM.
 
> **What is a Domain Controller?**  
> The Domain Controller is the brain of Active Directory. Every login on the domain, every access decision, every policy application — all of it flows through the DC. When you create a user, you create it on the DC. When a machine joins the domain, it trusts the DC to make authentication decisions. There is usually more than one in production for redundancy, but one is sufficient for this lab.
 
### Install via Server Manager
 
1. **Manage → Add Roles and Features**
2. Click **Next** to **Server Roles**
3. Check **Active Directory Domain Services**
4. Click **Add Features** when prompted (includes management tools)
5. Click **Next** through remaining pages → **Install**
6. Wait 2–3 minutes → **Close** (do not restart yet)
### Install via PowerShell
 
```powershell
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
```
 
### Install GPMC Now — Do Not Skip This
 
Group Policy Management Console is a separate feature needed for Step 6. Install it now to avoid hitting a wall later.
 
```powershell
Install-WindowsFeature -Name GPMC
```
 
After installation, close and reopen Server Manager. **Group Policy Management** now appears under the **Tools** menu. It is a completely separate application from Active Directory Users and Computers — GPOs are not managed inside ADUC.
 
---
 
## Step 3 — Promote the Server to a Domain Controller
 
> **What is a Forest and Domain?**  
> A **Forest** is the top-level container for your entire Active Directory structure. Think of it as the organisation itself. A **Domain** is a security and management boundary inside the forest. It has a DNS-style name — ours is `lab.local`. Everything inside a domain is managed together and trusts the same Domain Controller.
 
### Promote via Server Manager
 
1. Click the **yellow flag** (notification icon) in Server Manager
2. **Promote this server to a domain controller**
3. **Add a new forest**
4. Root domain name: `lab.local`
5. Click **Next** → set a **DSRM password** (store it securely — disaster recovery only)
6. Click through DNS Options and NetBIOS pages — accept defaults
7. Click **Install** → server restarts automatically when complete
> After the restart, log in with `LAB\Administrator` and your password. You are now inside the domain.
 
### Promote via PowerShell
 
```powershell
Import-Module ADDSDeployment
 
Install-ADDSForest `
  -DomainName 'lab.local' `
  -DomainNetBiosName 'LAB' `
  -InstallDns:$true `
  -SafeModeAdministratorPassword (ConvertTo-SecureString 'YourDSRMPassword!' -AsPlainText -Force) `
  -Force:$true
```
 
> **What just happened:** You created a new Active Directory forest called `lab.local`. This server is now the root DC — it runs DNS for the domain and is the authoritative source for every identity and authentication decision. Every machine that joins `lab.local` will trust this server implicitly.
 
---
 
## Verification Checklist
 
Run these commands after Step 3 completes to confirm the environment is healthy.
 
```powershell
# Verify the domain is up and this server is a DC
Get-ADDomain
 
# Confirm DNS is resolving the domain
Resolve-DnsName lab.local
 
# Check AD DS and DNS services are running
Get-Service ADWS, DNS, Netlogon, NTDS | Select Name, Status
 
# Confirm the forest
Get-ADForest
```
 
All four services (`ADWS`, `DNS`, `Netlogon`, `NTDS`) should show `Running`. `Get-ADDomain` should return `lab.local` with this server listed as a domain controller.
 
---
 
## Troubleshooting
 
| Issue | Cause | Fix |
|---|---|---|
| Promotion fails at DNS validation | DNS role conflict | Accept the default DNS configuration during promotion — the wizard handles it |
| Cannot RDP after server restart | Post-promotion login format changed | Log in as `LAB\Administrator` instead of just `Administrator` |
| Server Manager does not show yellow flag | AD DS role not fully installed | Close and reopen Server Manager, wait 60 seconds after install completes |
| `Get-ADDomain` returns an error | AD Web Services not started | Run `Start-Service ADWS` then retry |
| Clipboard not working in RDP | Connected via browser console | Download the `.rdp` file and connect via native Remote Desktop app |
 
---
 
## Key Concepts
 
| Term | Definition |
|---|---|
| Domain Controller (DC) | Server that runs Active Directory and handles all domain authentication |
| Forest | Top-level AD container. Represents the entire organisation |
| Domain | Management boundary inside the forest, identified by a DNS name (e.g. `lab.local`) |
| DSRM | Directory Services Restore Mode password — offline recovery credential for the DC |
| AD DS | Active Directory Domain Services — the Windows Server role that enables AD |
| GPMC | Group Policy Management Console — separate tool for creating and linking GPOs |
| NetBIOS name | Legacy short name for the domain (`LAB` in this lab) |
 



