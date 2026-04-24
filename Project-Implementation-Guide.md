# Implementation Guide — Build the Contoso-Azure Lab From Scratch

> **Who this is for:** Someone who has the `README.md` in this folder and wants a true zero-to-finished walkthrough. If a step feels "too obvious," it's there on purpose — this guide is written so a beginner can follow along without getting stuck.

> **How to use this guide:** Read one phase, do it, verify the checkpoint at the bottom of the phase, then move on. Every command has (a) a plain-English explanation of what it does and (b) what successful output looks like. If something fails, check the **"If it breaks"** block at the end of each phase.

> **Expected total time:** 12–16 hours of hands-on work spread over 1–2 weeks.
> **Expected Azure cost:** See Phase 0. Budget **~$250–$450 USD** if you leave everything running for a month; **under $30** if you destroy everything after each session (recommended for a lab).

---

## Table of Contents

- [Phase 0 — Before You Touch Anything: Prerequisites & Cost Awareness](#phase-0--before-you-touch-anything-prerequisites--cost-awareness)
- [Phase 1 — Install Your Local Tools](#phase-1--install-your-local-tools)
- [Phase 2 — Prepare Your Azure Tenant & Subscription](#phase-2--prepare-your-azure-tenant--subscription)
- [Phase 3 — Set Up the Code Repository](#phase-3--set-up-the-code-repository)
- [Phase 4 — Bootstrap Terraform State Storage](#phase-4--bootstrap-terraform-state-storage)
- [Phase 5 — Deploy the Hub Network (Firewall + Bastion)](#phase-5--deploy-the-hub-network-firewall--bastion)
- [Phase 6 — Deploy the Spoke VNets](#phase-6--deploy-the-spoke-vnets)
- [Phase 7 — Deploy Key Vault with Private Endpoint](#phase-7--deploy-key-vault-with-private-endpoint)
- [Phase 8 — Deploy Storage Accounts](#phase-8--deploy-storage-accounts)
- [Phase 9 — Deploy Windows and Linux VMs](#phase-9--deploy-windows-and-linux-vms)
- [Phase 10 — Deploy the App Service (Customer Portal)](#phase-10--deploy-the-app-service-customer-portal)
- [Phase 11 — Log Analytics + Microsoft Sentinel](#phase-11--log-analytics--microsoft-sentinel)
- [Phase 12 — Onboard Tenable.io](#phase-12--onboard-tenableio)
- [Phase 13 — Deliberately Introduce the Three Lab Vulnerabilities](#phase-13--deliberately-introduce-the-three-lab-vulnerabilities)
- [Phase 14 — Deploy Sentinel Analytics Rules](#phase-14--deploy-sentinel-analytics-rules)
- [Phase 15 — Run the Attack Simulation](#phase-15--run-the-attack-simulation)
- [Phase 16 — Validate Detections & Trigger the SOAR Playbook](#phase-16--validate-detections--trigger-the-soar-playbook)
- [Phase 17 — Document the Incident Like a Consultant](#phase-17--document-the-incident-like-a-consultant)
- [Phase 18 — Tear Everything Down (Stop Billing!)](#phase-18--tear-everything-down-stop-billing)
- [Appendix A — Troubleshooting Cheat Sheet](#appendix-a--troubleshooting-cheat-sheet)
- [Appendix B — Glossary of Jargon You'll See](#appendix-b--glossary-of-jargon-youll-see)

---

## Phase 0 — Before You Touch Anything: Prerequisites & Cost Awareness

### 0.1 What you must have

| Item | Why | How to get it |
|---|---|---|
| A valid credit card | Azure requires one even for the free trial | — |
| An email address not already tied to Azure | You'll create a fresh tenant | Gmail/Outlook works |
| Admin rights on your laptop | To install CLI tools | If corporate-managed, ask IT first |
| ~20 GB of free disk space | Terraform + state files + logs | — |
| Stable internet (50 Mbps+) | Terraform makes lots of API calls | — |
| Tenable.io trial account (optional) | For Phase 12 | https://www.tenable.com/products/tenable-io/evaluate — 30-day free trial |

### 0.2 Costs — read this twice

Azure charges by the hour for most resources. The *biggest* costs in this lab are:

| Resource | Approx cost if left on 24×7 | How to avoid |
|---|---:|---|
| Azure Firewall Premium | ~$1,200/mo | **Stop** the firewall when not actively testing (`az network firewall update --allocation none`) |
| 2× Windows VMs (Standard_D4s_v5) | ~$280/mo each | Deallocate when not using (`az vm deallocate`) |
| AKS cluster (if you do the bonus) | ~$220/mo | Scale node count to 0 when idle |
| Log Analytics ingestion | ~$2.30/GB | Will be small in a lab (few $ per month) |
| Everything else (KV, Storage, NSGs) | Pennies/mo | — |

**Golden rule:** If you aren't actively testing today, run Phase 18 (teardown) at the end of your session. Re-deploying from Terraform takes ~45 minutes, and that's cheaper than a surprise bill.

### 0.3 Set a spending cap — **do this before anything else**

Once you have an Azure account (Phase 2), set a budget alert:

```bash
# Replace <sub-id> with your subscription ID after Phase 2
az consumption budget create \
  --budget-name "lab-safety-cap" \
  --amount 100 \
  --time-grain Monthly \
  --start-date $(date +%Y-%m-01) \
  --end-date $(date -d "+1 year" +%Y-%m-01) \
  --category Cost \
  --notifications-enabled true \
  --contact-emails your-email@example.com \
  --threshold 80 \
  --subscription <sub-id>
```

This emails you when you hit 80% of $100 spend. Azure does **not** auto-stop resources when you hit a budget — it only notifies. You still need to teardown manually.

---

## Phase 1 — Install Your Local Tools

You need six tools on your workstation. We'll install them one at a time and verify each.

### 1.1 Install Azure CLI (`az`)

**What it is:** The command line that talks to Azure. Everything you can click in the Azure portal, `az` can do.

**Windows (PowerShell as Administrator):**
```powershell
winget install -e --id Microsoft.AzureCLI
```

**macOS (Terminal):**
```bash
brew install azure-cli
```

**Linux (Ubuntu/Debian):**
```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

**Verify:**
```bash
az --version
```
You should see output starting with `azure-cli    2.x.x`. If you see "command not found," close and reopen your terminal — the PATH won't refresh until then.

### 1.2 Install Terraform

**What it is:** A tool that reads `.tf` files (which we'll write) and creates/updates/destroys cloud resources accordingly. "Infrastructure as Code."

**Windows:**
```powershell
winget install -e --id Hashicorp.Terraform
```

**macOS:**
```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
```

**Linux:**
```bash
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform
```

**Verify:**
```bash
terraform -version
```
Expected: `Terraform v1.7.x` or newer.

### 1.3 Install Git

**What it is:** Version control. You'll use it to track your changes and, later, push to GitHub for your portfolio.

**Windows:** Install **Git for Windows** from https://git-scm.com/download/win (accept defaults).
**macOS:** `brew install git`
**Linux:** `sudo apt install git`

**Verify:**
```bash
git --version
```

### 1.4 Install Visual Studio Code

**What it is:** The editor you'll use to write Terraform and KQL files. Free.

Download from https://code.visualstudio.com.

Then install two helpful extensions (open VS Code → Extensions sidebar → search):
- **HashiCorp Terraform** — syntax highlighting and validation for `.tf` files
- **Azure Account** — signs into Azure from VS Code

### 1.5 Install jq (JSON processor)

**What it is:** A Unix tool that parses JSON output from `az` commands. Saves you from eyeballing 500-line JSON blobs.

**Windows:** `winget install -e --id stedolan.jq`
**macOS:** `brew install jq`
**Linux:** `sudo apt install jq`

**Verify:** `echo '{"x":1}' | jq .x` should print `1`.

### 1.6 (Optional but recommended) Install PowerShell 7 cross-platform

If you're on macOS/Linux, install PowerShell 7 so you can run the attack-simulation commands that use PowerShell.
```bash
# macOS
brew install powershell/tap/powershell
# Linux (Ubuntu 22.04)
sudo snap install powershell --classic
```

### ✅ Phase 1 Checkpoint

Run all five and confirm no errors:
```bash
az --version && terraform -version && git --version && jq --version && code --version
```

---

## Phase 2 — Prepare Your Azure Tenant & Subscription

### 2.1 Create an Azure account (skip if you already have one)

Go to https://azure.microsoft.com/free → Start free. You'll get **$200 credit for 30 days** plus 12 months of popular services free. Use an email *not* already tied to another Microsoft tenant.

### 2.2 Sign in from the CLI

```bash
az login
```
A browser tab opens. Sign in. Close the tab when it says "You are signed in." Your terminal will now list all subscriptions you can access.

### 2.3 Note down your IDs

Run:
```bash
az account show --query "{subscriptionId:id, tenantId:tenantId, user:user.name}" -o table
```
Copy the `subscriptionId` and `tenantId` values into a text file — you'll paste them into config files many times.

For the rest of this guide I'll use these placeholders:
- `<SUB_ID>` — your subscription ID (looks like `abcd1234-...`)
- `<TENANT_ID>` — your tenant ID

### 2.4 Register required resource providers

**What this does:** Azure APIs are grouped into "providers" (Microsoft.Compute, Microsoft.Network, etc.). On a fresh subscription some providers are not registered. Registering them is free and takes a few minutes.

```bash
for p in Microsoft.Compute Microsoft.Network Microsoft.Storage \
         Microsoft.KeyVault Microsoft.OperationalInsights \
         Microsoft.SecurityInsights Microsoft.Web Microsoft.ContainerService \
         Microsoft.ManagedIdentity Microsoft.Insights Microsoft.Authorization; do
  echo "Registering $p ..."
  az provider register --namespace $p
done
```

Wait 2–3 minutes, then verify:
```bash
az provider list --query "[?registrationState=='Registered'].namespace" -o tsv | sort
```
All 11 names above must appear in the output. If one is missing, wait 60 seconds and rerun the registration command for just that provider.

### 2.5 Set your default subscription and location

```bash
az account set --subscription <SUB_ID>
az configure --defaults location=eastus
```
From now on, commands default to the East US region. If you live closer to another region (e.g. `westeurope`, `southeastasia`), swap it in — just stay consistent or you'll have VNet-peering problems later.

### ✅ Phase 2 Checkpoint

```bash
az account show -o table
az provider show -n Microsoft.Network --query registrationState -o tsv
# Should print: Registered
```

---

## Phase 3 — Set Up the Code Repository

The folder `Cybersecurity Lab\` on your computer already contains the Terraform and KQL files I generated earlier. We'll initialize it as a Git repo so you can track changes and later push to GitHub.

### 3.1 Open a terminal in the project folder

**Windows:** Open File Explorer → navigate to `Documents\Claude\Projects\Cybersecurity Lab` → right-click in empty space → "Open in Terminal" (or Git Bash).

**macOS/Linux:**
```bash
cd ~/path/to/Cybersecurity\ Lab
```

### 3.2 Initialize Git

```bash
git init
git branch -m main
```

### 3.3 Create a `.gitignore` (prevents committing secrets/state)

Create a new file called `.gitignore` in the root of the folder with this content:
```
# Terraform
*.tfstate
*.tfstate.*
.terraform/
.terraform.lock.hcl
crash.log

# Local vars
*.tfvars
!example.tfvars

# Credentials
*.pem
*.pfx
*.key
.env
tenable-*.pem

# OS
.DS_Store
Thumbs.db
```

### 3.4 First commit

```bash
git add .
git commit -m "Initial commit: landing zone scaffolding"
```

### ✅ Phase 3 Checkpoint

`git status` should report *"nothing to commit, working tree clean."*

---

## Phase 4 — Bootstrap Terraform State Storage

Terraform tracks what it has deployed in a file called `terraform.tfstate`. Storing that file locally is risky (lose your laptop, lose your state). We'll store it in Azure Blob Storage with locking.

### 4.1 Understand what we're doing

- Create a **resource group** (a logical container for Azure resources)
- Create a **storage account** (cloud file storage) with security hardening
- Create a **blob container** (a folder inside the storage account)
- Point Terraform at it

### 4.2 Pick a globally unique storage account name

Storage account names must be **globally unique across all of Azure**, 3–24 lowercase letters/numbers. Example pattern: `sttfstate<initials><4-digit-year><2-random-digits>`.

Generate one:
```bash
# macOS/Linux
TFSTATE_SA="sttfstate$(whoami | tr -d '[:punct:]' | cut -c1-4)$(date +%y)$((RANDOM % 90 + 10))"
echo $TFSTATE_SA
```
```powershell
# Windows PowerShell
$TFSTATE_SA = "sttfstate" + ($env:USERNAME -replace '[^a-z0-9]','').ToLower().Substring(0,[Math]::Min(4,$env:USERNAME.Length)) + (Get-Date -Format "yy") + (Get-Random -Minimum 10 -Maximum 99)
$TFSTATE_SA
```
Save this name — you'll reuse it.

### 4.3 Create the backend

**Line-by-line explanation of each command:**

```bash
# Create the resource group. -n = name, -l = location.
az group create -n rg-tfstate-prod -l eastus
```

```bash
# Create the storage account.
#   --sku Standard_GRS = Geo-redundant (data copied to another region, safe)
#   --encryption-services blob = encrypt the blob service
#   --min-tls-version TLS1_2 = reject old TLS (security hardening)
#   --allow-blob-public-access false = no anonymous reads possible
#   --default-action Deny + --bypass AzureServices = default-deny firewall, Azure services allowed
az storage account create \
  -n $TFSTATE_SA \
  -g rg-tfstate-prod \
  -l eastus \
  --sku Standard_GRS \
  --encryption-services blob \
  --min-tls-version TLS1_2 \
  --allow-blob-public-access false \
  --default-action Allow      # temporarily Allow; we'll lock down after first use
```

```bash
# Create a blob container called "tfstate" in that account.
# --auth-mode login uses your Entra ID identity rather than an account key.
az storage container create \
  --name tfstate \
  --account-name $TFSTATE_SA \
  --auth-mode login
```

```bash
# Grant yourself the "Storage Blob Data Contributor" role so Terraform can read/write state.
MY_OBJ_ID=$(az ad signed-in-user show --query id -o tsv)
SA_ID=$(az storage account show -n $TFSTATE_SA -g rg-tfstate-prod --query id -o tsv)
az role assignment create \
  --assignee $MY_OBJ_ID \
  --role "Storage Blob Data Contributor" \
  --scope $SA_ID
```

### 4.4 Tell Terraform to use this backend

Create a new file `infrastructure\backend.tf` (same folder as `main.tf`):

```hcl
terraform {
  required_version = ">= 1.7.0"
  required_providers {
    azurerm = { source = "hashicorp/azurerm", version = "~> 3.110" }
    azuread = { source = "hashicorp/azuread", version = "~> 2.50" }
    random  = { source = "hashicorp/random",  version = "~> 3.6" }
  }
  backend "azurerm" {
    resource_group_name  = "rg-tfstate-prod"
    storage_account_name = "REPLACE_WITH_YOUR_TFSTATE_SA"   # ← paste $TFSTATE_SA here
    container_name       = "tfstate"
    key                  = "landing-zone.prod.tfstate"
    use_azuread_auth     = true
  }
}
provider "azurerm" {
  features {
    key_vault { purge_soft_delete_on_destroy = false }
  }
}
```

**What each line does:**
- `required_version` — fails if you're on an old Terraform.
- `required_providers` — pins provider versions so updates don't break you.
- `backend "azurerm"` — stores state in the storage account you just made.
- `use_azuread_auth = true` — uses your Entra ID identity, not a storage-account key.
- `provider "azurerm"` — configures default behaviors for the Azure provider.

### 4.5 Initialize Terraform

```bash
cd infrastructure
terraform init
```

Expected output ends with:
```
Terraform has been successfully initialized!
```

### ✅ Phase 4 Checkpoint

```bash
az storage blob list --account-name $TFSTATE_SA -c tfstate --auth-mode login -o table
```
The list is empty now — after our first apply, it will contain `landing-zone.prod.tfstate`.

### If it breaks

- **`AuthorizationPermissionMismatch`** — role assignment hasn't propagated yet. Wait 2 minutes, retry.
- **`StorageAccountAlreadyTaken`** — name collision. Regenerate `$TFSTATE_SA` with different digits.
- **`terraform init` asks to "migrate state"** — you had a local state from an earlier attempt. Delete `.terraform/` and `terraform.tfstate*` in the folder, then rerun.

---

## Phase 5 — Deploy the Hub Network (Firewall + Bastion)

This is where it gets exciting — we're creating the first real infrastructure. The hub is the network centerpiece: all traffic to/from the spokes goes through it.

### 5.1 Understand the moving pieces

```
Hub VNet 10.0.0.0/16
├── AzureFirewallSubnet 10.0.1.0/26   ← where Azure Firewall lives
├── AzureBastionSubnet  10.0.2.0/26   ← where Bastion lives
└── DnsResolverSubnet   10.0.3.0/28   ← private DNS resolver
```

- **Azure Firewall Premium** inspects ALL traffic leaving or entering the spokes. Has IDPS (detects attacks).
- **Azure Bastion** gives you secure RDP/SSH to VMs *without* exposing them to the internet. You'll use this to log into VMs.

### 5.2 Open the hub module file

Open `infrastructure\modules\hub-network\main.tf` in VS Code. This file was generated for you but you should skim it.

Key resources it creates (with explanations):

```hcl
resource "azurerm_virtual_network" "hub" { ... }
```
Creates the hub VNet (like a virtual data center).

```hcl
resource "azurerm_subnet" "firewall"  { name = "AzureFirewallSubnet" ... }
resource "azurerm_subnet" "bastion"   { name = "AzureBastionSubnet"  ... }
```
Two subnets with **specific required names** — Azure is picky.

```hcl
resource "azurerm_firewall_policy" "hub" {
  sku = "Premium"
  intrusion_detection { mode = "Deny" }
  threat_intelligence_mode = "Alert"
}
```
The firewall's ruleset. `Deny` on IDPS means "block detected attacks"; `Alert` on threat-intel means "log but don't block known-bad IPs until we're comfortable with false positives."

```hcl
resource "azurerm_firewall" "hub"       { ... }
resource "azurerm_bastion_host" "hub"   { ... }
```
The actual firewall and bastion instances.

### 5.3 Create the module stubs that the root calls

The root `main.tf` references modules like `./modules/spoke-network`, `./modules/keyvault`, etc. If those subfolders don't exist yet, Terraform will error. For this guide we'll scaffold each module as a **minimal working stub** and add real resources as we go — that way you can deploy incrementally.

From inside `infrastructure/`:

```bash
# Create module folders
mkdir -p modules/hub-network modules/spoke-network modules/log-analytics \
         modules/sentinel modules/keyvault modules/storage \
         modules/vm-windows modules/vm-linux modules/app-service modules/aks
```

For each module, create three files: `main.tf`, `variables.tf`, `outputs.tf`. The `main.tf` will hold resources, `variables.tf` declares inputs, `outputs.tf` exposes values to the root.

I'm going to give you the full content of the hub-network module now. The others follow the same pattern and you'll fill them in as the phases progress.

**File: `modules/hub-network/variables.tf`**
```hcl
variable "env"                 { type = string }
variable "location"            { type = string }
variable "resource_group_name" { type = string }
variable "hub_cidr"            { type = string }
variable "law_id"              { type = string }
variable "tags"                { type = map(string) }
```

**File: `modules/hub-network/outputs.tf`**
```hcl
output "vnet_id"              { value = azurerm_virtual_network.hub.id }
output "firewall_private_ip"  { value = azurerm_firewall.hub.ip_configuration[0].private_ip_address }
output "firewall_public_ip"   { value = azurerm_public_ip.afw.ip_address }
output "bastion_fqdn"         { value = azurerm_public_ip.bastion.fqdn }
```

**File: `modules/hub-network/main.tf`** — use the content shown in the README §4.3 and add these two public IPs:
```hcl
resource "azurerm_public_ip" "afw" {
  name                = "pip-afw-hub-${var.env}-01"
  location            = var.location
  resource_group_name = var.resource_group_name
  allocation_method   = "Static"
  sku                 = "Standard"
  zones               = ["1","2","3"]
  tags                = var.tags
}

resource "azurerm_public_ip" "bastion" {
  name                = "pip-bas-hub-${var.env}-01"
  location            = var.location
  resource_group_name = var.resource_group_name
  allocation_method   = "Static"
  sku                 = "Standard"
  domain_name_label   = "bastion-contoso-${var.env}-${substr(md5(var.resource_group_name),0,6)}"
  tags                = var.tags
}
```

### 5.4 Plan the deployment (DRY RUN — no resources created yet)

From `infrastructure/`:
```bash
terraform plan -out=hub.tfplan
```

**What happens:** Terraform reads your code, compares it to what exists in Azure (nothing yet), and shows a plan. You should see something like:
```
Plan: 27 to add, 0 to change, 0 to destroy.
```
Read the list. Every `+` line is a resource being created. If any line has a yellow `~` (update) or red `-` (delete), stop and investigate — for a greenfield deploy it should all be `+`.

### 5.5 Apply (actually create things — costs start now)

```bash
terraform apply hub.tfplan
```
**Wait 15–25 minutes.** Azure Firewall alone takes ~10 minutes. Bastion another ~8.

### 5.6 Verify

```bash
# Should list your new hub VNet and subnets
az network vnet list -g rg-plat-connect-prod-eastus-01 -o table

# Firewall status should be "Succeeded"
az network firewall show -n afw-hub-prod-01 -g rg-plat-connect-prod-eastus-01 \
  --query provisioningState -o tsv
```

### ✅ Phase 5 Checkpoint
Firewall provisioning state = `Succeeded`. Bastion visible in the portal.

### If it breaks

- **`SubnetIsFull`** — rare on fresh deploy; increase subnet prefix.
- **`FirewallPolicyPremiumNotAvailable`** — your region doesn't have Premium; change `sku = "Standard"` and remove the `intrusion_detection` block (you lose IDPS; acceptable for lab).
- **Quota errors (`OperationNotAllowed` mentioning `cores`)** — new subscriptions have low default quotas. Request an increase from the Azure portal → Subscriptions → Usage + quotas → + New Quota Request.

---

## Phase 6 — Deploy the Spoke VNets

Now we create the two spokes and peer them to the hub.

### 6.1 Build `modules/spoke-network/main.tf`

```hcl
variable "env"                 { type = string }
variable "location"            { type = string }
variable "resource_group_name" { type = string }
variable "spoke_cidr"          { type = string }
variable "hub_vnet_id"         { type = string }
variable "firewall_private_ip" { type = string }
variable "law_id"              { type = string }
variable "subnet_definitions"  { type = map(object({ cidr = string, delegated_to = string })) }
variable "tags"                { type = map(string) }

resource "azurerm_virtual_network" "spoke" {
  name                = "vnet-${var.env}-${substr(md5(var.spoke_cidr),0,4)}"
  location            = var.location
  resource_group_name = var.resource_group_name
  address_space       = [var.spoke_cidr]
  tags                = var.tags
}

resource "azurerm_subnet" "this" {
  for_each             = var.subnet_definitions
  name                 = "snet-${each.key}"
  resource_group_name  = var.resource_group_name
  virtual_network_name = azurerm_virtual_network.spoke.name
  address_prefixes     = [each.value.cidr]

  dynamic "delegation" {
    for_each = each.value.delegated_to == null ? [] : [1]
    content {
      name = "delegation"
      service_delegation {
        name = each.value.delegated_to
      }
    }
  }
}

# Route table: default route to Azure Firewall (forced tunneling)
resource "azurerm_route_table" "spoke_rt" {
  name                = "rt-${azurerm_virtual_network.spoke.name}"
  location            = var.location
  resource_group_name = var.resource_group_name
  tags                = var.tags

  route {
    name                   = "default-to-firewall"
    address_prefix         = "0.0.0.0/0"
    next_hop_type          = "VirtualAppliance"
    next_hop_in_ip_address = var.firewall_private_ip
  }
}

resource "azurerm_subnet_route_table_association" "assoc" {
  for_each       = azurerm_subnet.this
  subnet_id      = each.value.id
  route_table_id = azurerm_route_table.spoke_rt.id
}

# Hub <-> Spoke peering (bidirectional)
resource "azurerm_virtual_network_peering" "spoke_to_hub" {
  name                         = "peer-to-hub"
  resource_group_name          = var.resource_group_name
  virtual_network_name         = azurerm_virtual_network.spoke.name
  remote_virtual_network_id    = var.hub_vnet_id
  allow_virtual_network_access = true
  allow_forwarded_traffic      = true
  allow_gateway_transit        = false
  use_remote_gateways          = false
}

output "vnet_id"    { value = azurerm_virtual_network.spoke.id }
output "subnet_ids" { value = { for k, v in azurerm_subnet.this : k => v.id } }
```

### 6.2 Explain key concepts

- **Route table with `0.0.0.0/0 → FirewallPrivateIP`** — forces every outbound packet from VMs through Azure Firewall so it gets inspected.
- **Peering** — like "VNet-to-VNet private network cable." Spoke can reach hub, hub can reach spoke. But two different spokes cannot talk directly — traffic must traverse the hub (via Firewall), which is what we want.

### 6.3 You need a **reverse peering** on the hub

Add this in `modules/hub-network/main.tf` via a `for_each` that takes a list of spoke VNet IDs. The simplest way for the lab is to run a one-liner after each spoke deploys:

```bash
HUB_RG=rg-plat-connect-prod-eastus-01
HUB_VNET=vnet-hub-prod-eastus-01
SPOKE_VNET_ID=$(az network vnet show -n vnet-prod-XXXX -g rg-corp-prod-app-eastus-01 --query id -o tsv)

az network vnet peering create \
  --name peer-hub-to-prod \
  --vnet-name $HUB_VNET \
  --resource-group $HUB_RG \
  --remote-vnet $SPOKE_VNET_ID \
  --allow-vnet-access \
  --allow-forwarded-traffic
```

### 6.4 Plan + apply again

```bash
terraform plan -out=spoke.tfplan
terraform apply spoke.tfplan
```

### ✅ Phase 6 Checkpoint

```bash
az network vnet peering list -g $HUB_RG --vnet-name $HUB_VNET -o table
# "PeeringState" should be "Connected" for each spoke
```

---

## Phase 7 — Deploy Key Vault with Private Endpoint

Key Vault stores secrets, keys, and certificates. We'll make it **internet-unreachable** — only accessible via the VNet through a private endpoint.

### 7.1 Explain what a Private Endpoint is

A Private Endpoint is a network interface with a private IP inside your VNet that represents an Azure PaaS resource. So `kv-contoso-prod.vault.azure.net` resolves — *inside our VNet* — to `10.3.1.4`, not a public IP. Outside the VNet, the resource is unreachable.

For this to work, you also need a **Private DNS Zone** (`privatelink.vaultcore.azure.net`) linked to your VNets so DNS queries return the private IP.

### 7.2 Populate `modules/keyvault/main.tf`

Use the example from README §4.4. Add these variables:

```hcl
variable "workload"             { type = string }
variable "env"                  { type = string }
variable "location"             { type = string }
variable "resource_group_name"  { type = string }
variable "pe_subnet_id"         { type = string }
variable "kv_private_dns_zone_id" { type = string }
variable "law_id"               { type = string }
variable "tags"                 { type = map(string) }

resource "random_string" "suffix" {
  length  = 5
  special = false
  upper   = false
}

data "azurerm_client_config" "current" {}
```

And these outputs:
```hcl
output "id"                    { value = azurerm_key_vault.kv.id }
output "uri"                   { value = azurerm_key_vault.kv.vault_uri }
output "disk_encryption_set_id" { value = azurerm_disk_encryption_set.des.id }
output "cmk_key_id"            { value = azurerm_key_vault_key.cmk.id }
output "cmk_identity_id"       { value = azurerm_user_assigned_identity.cmk.id }
```

Plus the CMK + DES resources:
```hcl
resource "azurerm_key_vault_key" "cmk" {
  name         = "cmk-disk-encryption"
  key_vault_id = azurerm_key_vault.kv.id
  key_type     = "RSA-HSM"
  key_size     = 2048
  key_opts     = ["wrapKey","unwrapKey"]
  depends_on   = [azurerm_role_assignment.self_kv_admin]
}

resource "azurerm_user_assigned_identity" "cmk" {
  name                = "id-cmk-${var.workload}-${var.env}"
  location            = var.location
  resource_group_name = var.resource_group_name
}

resource "azurerm_role_assignment" "cmk_crypto" {
  scope                = azurerm_key_vault.kv.id
  role_definition_name = "Key Vault Crypto User"
  principal_id         = azurerm_user_assigned_identity.cmk.principal_id
}

resource "azurerm_disk_encryption_set" "des" {
  name                = "des-${var.workload}-${var.env}"
  location            = var.location
  resource_group_name = var.resource_group_name
  key_vault_key_id    = azurerm_key_vault_key.cmk.id
  encryption_type     = "EncryptionAtRestWithCustomerKey"

  identity {
    type         = "UserAssigned"
    identity_ids = [azurerm_user_assigned_identity.cmk.id]
  }
}

resource "azurerm_role_assignment" "self_kv_admin" {
  scope                = azurerm_key_vault.kv.id
  role_definition_name = "Key Vault Administrator"
  principal_id         = data.azurerm_client_config.current.object_id
}
```

**What those do:**
- `azurerm_key_vault_key.cmk` — the Customer-Managed Key for disk encryption.
- `azurerm_user_assigned_identity.cmk` — an identity Azure will use to fetch that key.
- `azurerm_disk_encryption_set.des` — a resource that VMs reference to encrypt their disks with your key.
- `self_kv_admin` — gives *you* admin rights on this KV so you can create the key.

### 7.3 Plan + apply

```bash
terraform plan -out=kv.tfplan
terraform apply kv.tfplan
```

### 7.4 Add the Nessus linking key as a secret (you'll need this in Phase 12)

For now, add a placeholder so the VM module can reference it:
```bash
KV_NAME=$(az keyvault list -g rg-corp-prod-data-eastus-01 --query "[0].name" -o tsv)
az keyvault secret set --vault-name $KV_NAME --name nessus-linking-key --value "PLACEHOLDER-REPLACE-IN-PHASE-12"
```

### ✅ Phase 7 Checkpoint
From your laptop (public internet), this should **fail with a 403 or DNS failure**:
```bash
az keyvault secret show --vault-name $KV_NAME --name nessus-linking-key
```
Because you've locked public access. You'll only be able to query it from inside the VNet (e.g., from a VM via Bastion) or by temporarily allowing your IP.

To test it from your laptop during lab work, grant your current public IP:
```bash
MY_IP=$(curl -s https://ifconfig.me)
az keyvault network-rule add --name $KV_NAME --ip-address $MY_IP
# don't forget to remove it later: az keyvault network-rule remove ...
```

---

## Phase 8 — Deploy Storage Accounts

### 8.1 Fill `modules/storage/main.tf`

Core resources:
```hcl
resource "azurerm_storage_account" "this" {
  name                     = var.name
  resource_group_name      = var.resource_group_name
  location                 = var.location
  account_tier             = "Standard"
  account_replication_type = "ZRS"
  account_kind             = "StorageV2"

  min_tls_version                = "TLS1_2"
  enable_https_traffic_only      = true
  allow_nested_items_to_be_public = false
  public_network_access_enabled  = false
  shared_access_key_enabled      = false  # RBAC only

  identity {
    type         = "UserAssigned"
    identity_ids = [var.cmk_identity_id]
  }

  customer_managed_key {
    key_vault_key_id          = var.cmk_key_id
    user_assigned_identity_id = var.cmk_identity_id
  }

  blob_properties {
    versioning_enabled       = true
    change_feed_enabled      = true
    delete_retention_policy { days = 30 }
    container_delete_retention_policy { days = 30 }
  }

  network_rules {
    default_action = "Deny"
    bypass         = ["AzureServices"]
  }

  tags = var.tags
}

# Private endpoints (one for blob, one for file)
resource "azurerm_private_endpoint" "blob" {
  name                = "pe-${var.name}-blob"
  location            = var.location
  resource_group_name = var.resource_group_name
  subnet_id           = var.pe_subnet_id

  private_service_connection {
    name                           = "psc-blob"
    private_connection_resource_id = azurerm_storage_account.this.id
    subresource_names              = ["blob"]
    is_manual_connection           = false
  }
  private_dns_zone_group {
    name                 = "default"
    private_dns_zone_ids = [var.blob_dns_zone_id]
  }
}
```

### 8.2 Explain the security-hardening flags

| Flag | What it does |
|---|---|
| `min_tls_version = "TLS1_2"` | Rejects old TLS versions vulnerable to POODLE, BEAST, etc. |
| `enable_https_traffic_only = true` | Blocks HTTP (cleartext). |
| `public_network_access_enabled = false` | No internet reach. |
| `shared_access_key_enabled = false` | Disables the legacy account key (forces RBAC auth). |
| `customer_managed_key` | Encrypts with your KV-held key, so rotating it rotates data encryption. |
| `network_rules.default_action = "Deny"` | Default-deny firewall. |

### 8.3 Apply

```bash
terraform plan -out=storage.tfplan
terraform apply storage.tfplan
```

---

## Phase 9 — Deploy Windows and Linux VMs

### 9.1 The VM modules (skeleton)

Use the example from README §4.5 for Windows. For Linux:

```hcl
resource "azurerm_linux_virtual_machine" "this" {
  name                  = "vm-${var.role}-${var.env}-${format("%02d", var.index)}"
  resource_group_name   = var.resource_group_name
  location              = var.location
  size                  = "Standard_B2s"   # B-series = burstable, cheap for lab
  admin_username        = var.admin_username
  network_interface_ids = [azurerm_network_interface.nic.id]
  zone                  = var.zone

  disable_password_authentication = true

  admin_ssh_key {
    username   = var.admin_username
    public_key = var.ssh_public_key
  }

  identity { type = "SystemAssigned" }

  os_disk {
    name                   = "osdisk-${var.role}-${var.index}"
    caching                = "ReadWrite"
    storage_account_type   = "Premium_LRS"
    disk_encryption_set_id = var.des_id
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }

  encryption_at_host_enabled = true
}
```

### 9.2 Generate an SSH key (for the Linux VM)

```bash
ssh-keygen -t ed25519 -f ~/.ssh/contoso-lab -C "contoso-lab"
# Press Enter twice (no passphrase for lab convenience)
cat ~/.ssh/contoso-lab.pub
```
Copy the `.pub` content into `terraform.tfvars`:
```hcl
admin_ssh_public_key = "ssh-ed25519 AAAA... contoso-lab"
```

### 9.3 Apply

```bash
terraform plan -out=vm.tfplan
terraform apply vm.tfplan
```
VMs take ~4 minutes each. The Nessus agent extension will attempt to install too but will currently fail because the linking key is a placeholder — that's fine, we'll fix in Phase 12.

### 9.4 Log into a VM through Bastion

1. Open the Azure Portal → go to your VM → click **Connect → Bastion**.
2. Username: `contosoadmin`. For Linux, upload private key `~/.ssh/contoso-lab`. For Windows, use the password from Key Vault:
   ```bash
   az keyvault secret show --vault-name $KV_NAME --name vm-app-prod-01-password --query value -o tsv
   ```

### ✅ Phase 9 Checkpoint
You can RDP to `vm-app-prod-01` through Bastion. You can SSH to `vm-build-prod-01`. Neither VM has a public IP — verify:
```bash
az vm list -d --query "[].{name:name, publicIps:publicIps}" -o table
# publicIps column should all be empty/null
```

---

## Phase 10 — Deploy the App Service (Customer Portal)

Skipping to the essential commands — the `modules/app-service/main.tf` follows the same private-endpoint pattern:

```hcl
resource "azurerm_service_plan" "plan" {
  name                = "plan-${var.name}"
  location            = var.location
  resource_group_name = var.resource_group_name
  os_type             = "Linux"
  sku_name            = var.sku
}

resource "azurerm_linux_web_app" "app" {
  name                = var.name
  location            = var.location
  resource_group_name = var.resource_group_name
  service_plan_id     = azurerm_service_plan.plan.id

  https_only                      = true
  public_network_access_enabled   = false
  virtual_network_subnet_id       = var.vnet_integration_subnet_id

  identity { type = "SystemAssigned" }

  site_config {
    minimum_tls_version = var.tls_min_version
    ftps_state          = "Disabled"
    http2_enabled       = true
    always_on           = true
    ip_restriction_default_action = "Deny"

    application_stack {
      docker_image_name   = "nginxdemos/hello:latest"
      docker_registry_url = "https://index.docker.io"
    }
  }

  logs {
    http_logs { file_system { retention_in_days = 7, retention_in_mb = 35 } }
  }
}
```

```bash
terraform apply
```

---

## Phase 11 — Log Analytics + Microsoft Sentinel

### 11.1 What we're building

A central workspace where logs from every resource land, then Sentinel on top of it for detection and SOAR.

### 11.2 Log Analytics module

```hcl
resource "azurerm_log_analytics_workspace" "law" {
  name                = var.name
  location            = var.location
  resource_group_name = var.resource_group_name
  sku                 = "PerGB2018"
  retention_in_days   = var.retention_days
  daily_quota_gb      = var.daily_quota_gb
  tags                = var.tags
}

output "id"   { value = azurerm_log_analytics_workspace.law.id }
output "name" { value = azurerm_log_analytics_workspace.law.name }
```

### 11.3 Sentinel module

```hcl
resource "azurerm_sentinel_log_analytics_workspace_onboarding" "sentinel" {
  workspace_id = var.workspace_id
}

resource "azurerm_sentinel_data_connector_azure_active_directory" "aad" {
  count        = contains(var.data_connectors, "AzureActiveDirectory") ? 1 : 0
  name         = "aad"
  log_analytics_workspace_id = var.workspace_id
}

resource "azurerm_sentinel_data_connector_azure_security_center" "asc" {
  count        = contains(var.data_connectors, "AzureSecurityCenter") ? 1 : 0
  name         = "asc"
  log_analytics_workspace_id = var.workspace_id
}

# repeat for other connectors you want
```

Apply:
```bash
terraform apply
```

### 11.4 Attach Diagnostic Settings to every resource

For Key Vault, Storage, App Service, VMs, NSGs — every resource emits logs only if a `diagnostic_setting` points them at the LAW. We've already shown this pattern in the KV example. Add equivalents to Storage, App Service, and NSGs.

### 11.5 Install the Azure Monitor Agent on VMs

The `AzureMonitorWindowsAgent` and `AzureMonitorLinuxAgent` extensions are already in the VM modules. You also need a **Data Collection Rule (DCR)** to tell the agent what logs to send:

```hcl
resource "azurerm_monitor_data_collection_rule" "win_security" {
  name                = "dcr-win-security"
  location            = var.location
  resource_group_name = var.resource_group_name

  destinations {
    log_analytics {
      workspace_resource_id = var.law_id
      name                  = "law-dest"
    }
  }

  data_flow {
    streams      = ["Microsoft-SecurityEvent"]
    destinations = ["law-dest"]
  }

  data_sources {
    windows_event_log {
      streams        = ["Microsoft-SecurityEvent"]
      x_path_queries = ["Security!*[System[(band(Keywords,13510798882111488))]]"]
      name           = "sec-events"
    }
  }
}

resource "azurerm_monitor_data_collection_rule_association" "assoc" {
  for_each                = module.vm_app
  name                    = "dcr-assoc-${each.key}"
  target_resource_id      = each.value.id
  data_collection_rule_id = azurerm_monitor_data_collection_rule.win_security.id
}
```

### ✅ Phase 11 Checkpoint
Open the Azure Portal → Log Analytics workspace → Logs → run:
```kusto
Heartbeat | where TimeGenerated > ago(10m) | summarize count() by Computer
```
You should see heartbeats from your VMs. If not, wait 10 more minutes — the first heartbeat takes a while.

---

## Phase 12 — Onboard Tenable.io

### 12.1 Get a Tenable.io trial

Sign up at https://www.tenable.com/products/tenable-io/evaluate → 30-day trial.

### 12.2 Run the connector bootstrap script

The script is already in your repo at `tenable/connector-setup.sh`.

First, download Tenable's public certificate: Tenable.io UI → Settings → My Account → API Keys / Certs → download public cert → save as `tenable-public.pem` in the `tenable/` folder.

Then run:
```bash
cd tenable
chmod +x connector-setup.sh
./connector-setup.sh <TENANT_ID> <SUB_ID>
```

The script will output an App ID. Paste it along with your tenant ID and subscription ID into Tenable.io → Settings → Connectors → New → Microsoft Azure.

### 12.3 Deploy Nessus Agents

Get your linking key from Tenable.io → Sensors → Linked Agents → "Linking Key." Update the Key Vault secret:
```bash
az keyvault secret set --vault-name $KV_NAME \
  --name nessus-linking-key --value "YOUR-REAL-LINKING-KEY"
```

Then re-run `terraform apply` — the VM extension will re-run with the real key and the agent will link back to Tenable.io within ~5 minutes.

### 12.4 Kick off the first scan

In Tenable.io → Scans → New Scan → "Basic Agent Scan" → select your agent group → Save → Launch. Wait ~15 minutes. Findings appear under Vulnerabilities.

### ✅ Phase 12 Checkpoint
Tenable.io shows your VMs as "Online agents" and at least one scan has completed.

---

## Phase 13 — Deliberately Introduce the Three Lab Vulnerabilities

This is what makes it a security lab — we *intentionally* create weaknesses so we can detect and remediate them.

### V-001: Open RDP to the Internet

```bash
az network nsg rule create \
  --resource-group rg-corp-prod-app-eastus-01 \
  --nsg-name nsg-snet-app \
  --name Allow-RDP-From-Anywhere \
  --priority 150 \
  --source-address-prefixes Internet \
  --source-port-ranges '*' \
  --destination-port-ranges 3389 \
  --direction Inbound --access Allow --protocol Tcp
```

**What each flag means:**
- `--priority 150` — lower than the default-deny (4000), so this Allow wins.
- `--source-address-prefixes Internet` — Azure service tag meaning "the whole internet."
- `--destination-port-ranges 3389` — RDP.

This rule will trip `DET-012` (Risky NSG rule) within minutes.

### V-002: Unpatched Windows VM

RDP via Bastion to `vm-app-prod-01`, then PowerShell as Admin:
```powershell
Stop-Service wuauserv
Set-Service wuauserv -StartupType Disabled
# Simulate missed patches by rolling back a KB (for lab effect)
wusa /uninstall /kb:5034441 /quiet /norestart
```

### V-003: Over-privileged Managed Identity

```bash
az identity create -g rg-corp-prod-app-eastus-01 -n id-app-overprivileged
PRINCIPAL_ID=$(az identity show -g rg-corp-prod-app-eastus-01 -n id-app-overprivileged --query principalId -o tsv)
az role assignment create --assignee $PRINCIPAL_ID --role "Contributor" --scope /subscriptions/<SUB_ID>

# Attach this identity to vm-app-prod-01
VM_ID=$(az vm show -g rg-corp-prod-app-eastus-01 -n vm-app-prod-01 --query id -o tsv)
MI_ID=$(az identity show -g rg-corp-prod-app-eastus-01 -n id-app-overprivileged --query id -o tsv)
az vm identity assign --ids $VM_ID --identities $MI_ID
```

### ✅ Phase 13 Checkpoint
All three vulnerabilities are in place. Within ~30 minutes, Tenable and Defender for Cloud should flag them.

---

## Phase 14 — Deploy Sentinel Analytics Rules

### 14.1 Convert KQL files to analytics rules

For each KQL in `sentinel/analytics/`:

1. Azure Portal → Microsoft Sentinel → your workspace → Analytics → Create → Scheduled query rule.
2. **Name:** DET-001 RDP Brute Force Then Success.
3. **Tactics:** CredentialAccess, InitialAccess.
4. **Severity:** High.
5. **Rule query:** paste the KQL from `DET-001-rdp-bruteforce-success.kql`.
6. **Query scheduling:** Run every 10 minutes, Lookup data from the last 1 hour.
7. **Alert threshold:** Greater than 0.
8. **Entity mapping:** Account→TargetAccount, IP→SrcIP, Host→Computer.
9. **Incident settings:** Create incidents. Enable grouping.
10. Save.

Repeat for DET-005, DET-010, DET-012.

### 14.2 Or deploy via CI/CD (preferred for portfolio)

Commit the KQL files to GitHub, then use Microsoft's **Sentinel Repositories** feature: Sentinel → Repositories → Configure → point at your GitHub repo → pull request deployment flow. Your rules now live as code, versioned, reviewable.

### ✅ Phase 14 Checkpoint
Analytics rules list shows 4+ rules with "Enabled" status and last-run timestamps within 10 minutes.

---

## Phase 15 — Run the Attack Simulation

Use `threat-sim/atomic-tests.md` as your runbook. **Only run these from the non-prod spoke against the lab VMs you've designated as the detonation range.**

### 15.1 Simulated brute-force

From your laptop (simulating the attacker), with V-001 exposing RDP:
```bash
# Install hydra
sudo apt install hydra -y    # Linux/WSL
brew install hydra           # macOS

# Get the VM's temporary public IP (you'll need to add one for this test)
# Or, even simpler, brute-force through Bastion native client.
# For lab purposes, generate the equivalent 4625 events by running this ON the VM:
```
On `vm-app-prod-01` (PowerShell as Admin):
```powershell
1..25 | ForEach-Object {
  $user = "contoso\fakeuser"
  $pwd = ConvertTo-SecureString "WrongPass$_" -AsPlainText -Force
  $cred = New-Object System.Management.Automation.PSCredential($user,$pwd)
  # This generates authentic 4625 failure events
  Start-Process -FilePath "whoami" -Credential $cred -ErrorAction SilentlyContinue
}
```
Then do a successful logon (4624) from the same "IP" by signing in normally.

### 15.2 PowerShell encoded command (DET-002)

```powershell
$cmd = "Write-Host 'simulated attack'"
$enc = [Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes($cmd))
powershell.exe -enc $enc
```

### 15.3 Managed-Identity abuse (DET-005)

```powershell
$token = (Invoke-WebRequest -UseBasicParsing -Uri 'http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://management.azure.com/' -Headers @{Metadata="true"}).Content | ConvertFrom-Json
$headers = @{ Authorization = "Bearer $($token.access_token)" }
1..60 | ForEach-Object {
  Invoke-RestMethod "https://management.azure.com/subscriptions/<SUB_ID>/resources?api-version=2021-04-01" -Headers $headers | Out-Null
}
```

### 15.4 Bulk blob download (DET-010)

From the Linux VM:
```bash
az login --identity
for i in {1..600}; do
  az storage blob download --account-name stcontosoprod001 \
    --container-name customer-data \
    --name "blob-$i.bin" \
    --file /tmp/x --auth-mode login 2>/dev/null
done
```

### ✅ Phase 15 Checkpoint
Each attack simulation has run at least once.

---

## Phase 16 — Validate Detections & Trigger the SOAR Playbook

### 16.1 Check incidents were created

Sentinel → Incidents. Within ~15 minutes of each attack you should see:
- New incident: "RDP brute force followed by success" (High)
- New incident: "Managed Identity performing broad ARM enumeration" (High)
- New incident: "Bulk blob download from stcontosoprod001" (High)
- New incident: "Risky NSG rule Allow-RDP-From-Anywhere allows Internet→3389" (High)

### 16.2 Deploy the Logic App playbook

```bash
cd sentinel/playbooks

# Deploy via az CLI using Bicep
az deployment group create \
  -g rg-plat-mgmt-prod-eastus-01 \
  --template-file PB-ISOLATE-VM.bicep \
  --parameters quarantineNsgId=$(az network nsg show -g rg-corp-prod-app-eastus-01 -n nsg-quarantine --query id -o tsv) \
               sentinelWorkspaceId=$(az monitor log-analytics workspace show -g rg-plat-mgmt-prod-eastus-01 -n law-contoso-prod-central-01 --query id -o tsv) \
               serviceNowConnectionId="dummy" \
               teamsConnectionId="dummy"
```

### 16.3 Grant the Logic App permissions

```bash
LA_PRINCIPAL=$(az deployment group show -g rg-plat-mgmt-prod-eastus-01 \
    -n PB-ISOLATE-VM --query properties.outputs.logicAppPrincipalId.value -o tsv)

az role assignment create --assignee $LA_PRINCIPAL \
  --role "Virtual Machine Contributor" \
  --scope /subscriptions/<SUB_ID>/resourceGroups/rg-corp-prod-app-eastus-01

az role assignment create --assignee $LA_PRINCIPAL \
  --role "Microsoft Sentinel Responder" \
  --scope $(az monitor log-analytics workspace show -g rg-plat-mgmt-prod-eastus-01 -n law-contoso-prod-central-01 --query id -o tsv)
```

### 16.4 Link the playbook to an analytics rule

Sentinel → Analytics → open DET-001 → Automated response → Add → select "la-pb-isolate-vm-prod-01" → Save.

### 16.5 Re-run DET-001 to trigger auto-isolation

Repeat the brute-force simulation from 15.1. Within a minute you should see:
- Incident created
- Logic App run succeeded
- `vm-app-prod-01` NIC now references `nsg-quarantine`
- A new snapshot `snap-ir-2026MMDDTHHMM` in `rg-ir-evidence`

### ✅ Phase 16 Checkpoint
Sentinel shows "Playbook run: succeeded" on the incident activity log.

---

## Phase 17 — Document the Incident Like a Consultant

### 17.1 Export incident details

Sentinel → Incidents → pick the DET-001 incident → "..." → Export to CSV.

### 17.2 Build a one-page timeline

Fill in the template from README §8.4. Capture:
- MTTD (mean time to detect) — timestamp difference between first 4625 and incident creation.
- MTTC (mean time to contain) — incident creation to Logic App completion.
- MTTR (mean time to remediate) — ticket creation to final "closed" status.

### 17.3 Produce the vuln report

Use the template in README §10.2. Pull totals from Tenable.io: Dashboards → Vulnerabilities → export CSV → summarize P0/P1/P2/P3/P4 counts.

### ✅ Phase 17 Checkpoint
You now have a real, portfolio-quality artifact: a timeline with real timestamps, a vulnerability summary with real counts, and screenshots of the Sentinel incident and Logic App run.

---

## Phase 18 — Tear Everything Down (Stop Billing!)

When you're done for the session:

### Option A — Destroy only expensive resources (keep structure)

```bash
# Deallocate VMs (stops compute billing; disk still billed at ~$0.05/day)
az vm deallocate --ids $(az vm list --query "[].id" -o tsv)

# Stop Azure Firewall (saves ~$40/day on Premium!)
az network firewall update -n afw-hub-prod-01 \
  -g rg-plat-connect-prod-eastus-01 \
  --set ipConfigurations=[]

# Scale AKS to zero (if deployed)
az aks nodepool update -g rg-corp-prod-app-eastus-01 \
  --cluster-name aks-contoso-prod-01 --name system --node-count 1
```

### Option B — Nuke everything (recommended for long pauses)

```bash
cd infrastructure
terraform destroy
```
Takes ~20 minutes. Type `yes` when prompted.

Then delete the tfstate RG too:
```bash
az group delete -n rg-tfstate-prod -y
```

### Verify zero resources

```bash
az group list -o table
```
Everything except default/managed RGs should be gone.

### ✅ Phase 18 Checkpoint
Your Azure bill page (Cost Management) shows forecast trending to $0.

---

## Appendix A — Troubleshooting Cheat Sheet

| Symptom | Likely Cause | Fix |
|---|---|---|
| `terraform apply` hangs on Firewall | Normal — ~10 min | Wait |
| `429 TooManyRequests` from ARM | You ran apply repeatedly | Wait 5 min, retry |
| VM extension `Failed` state | Nessus linking key placeholder | Update KV secret, `az vm extension set` or rerun apply |
| Bastion shows "connection failed" | Browser blocking pop-ups | Allow pop-ups for portal.azure.com |
| KQL rule "Failed to run" | Table not present yet | Wait 30 min for logs to populate |
| Logic App "403 Forbidden" | Missing RBAC on MI | Re-run the role assignments in 16.3 |
| Can't resolve `*.blob.core.windows.net` | Private DNS zone not linked | Verify `azurerm_private_dns_zone_virtual_network_link` exists |
| `SubscriptionNotRegistered` | Resource provider not registered | Re-run Phase 2.4 |
| `PolicyViolation: "Allowed locations"` | Policy assigned that blocks a region | Edit policy or pick an allowed region |
| Terraform state lock stuck | Previous apply crashed | `terraform force-unlock <LOCK_ID>` |

---

## Appendix B — Glossary of Jargon You'll See

| Term | Plain English |
|---|---|
| **ARM** | Azure Resource Manager — the single API everything in Azure runs through |
| **ASG** | Application Security Group — an NSG tag you can attach to NICs instead of IP ranges |
| **Bastion** | Azure's managed jump-box; RDP/SSH without exposing VMs to the internet |
| **CIDR** | IP range notation (10.0.0.0/16 = 65,536 addresses) |
| **CMK** | Customer-Managed Key — encryption key *you* own in Key Vault |
| **DCR** | Data Collection Rule — tells the Azure Monitor Agent what to collect |
| **DES** | Disk Encryption Set — a wrapper that points VMs to your CMK |
| **IMDS** | Instance Metadata Service (169.254.169.254) — where a VM gets MI tokens |
| **KQL** | Kusto Query Language — SQL-like language for Azure logs |
| **LAW** | Log Analytics Workspace — the database where logs land |
| **MDE** | Microsoft Defender for Endpoint — EDR agent |
| **MI** | Managed Identity — password-less identity attached to an Azure resource |
| **NIC** | Network Interface Card — virtual network adapter on a VM |
| **NSG** | Network Security Group — firewall rules at subnet/NIC level |
| **PE** | Private Endpoint — a private IP for a PaaS service inside your VNet |
| **PIM** | Privileged Identity Management — just-in-time elevation for Entra roles |
| **PaaS** | Platform-as-a-Service — managed services (KV, App Service) vs. IaaS (VMs) |
| **RBAC** | Role-Based Access Control — who can do what to which resource |
| **RG** | Resource Group — folder for Azure resources |
| **SAS** | Shared Access Signature — time-limited URL to a storage resource |
| **Sentinel** | Microsoft's cloud SIEM, built on top of Log Analytics |
| **SKU** | Stock-Keeping Unit — a pricing/capability tier (e.g., `Standard_D4s_v5`) |
| **SOAR** | Security Orchestration, Automation, & Response — auto-respond to alerts |
| **Tf** | Terraform |
| **UDR** | User-Defined Route — custom routing rule in a route table |
| **VMSS** | Virtual Machine Scale Set — auto-scaling group of identical VMs |
| **VNet** | Virtual Network — your private network in Azure |

---

## You're Done 🎉

You now have:
- A working Zero Trust-aligned Azure lab
- Continuous vulnerability management via Tenable.io
- Real detections firing against real attacks
- Automated SOAR containment
- Documented evidence suitable for a portfolio or interview

**Next steps for your career:**
1. Take screenshots of each phase's successful outcome. Drop them in `docs/screenshots/`.
2. Record a 3-minute Loom video walking through Sentinel incidents.
3. Push the whole repo to GitHub (public). Pin it on your profile.
4. In your LinkedIn headline: "Built an Azure Zero Trust lab with Sentinel + Tenable + MITRE ATT&CK validation." Link to the repo.

Good luck.

