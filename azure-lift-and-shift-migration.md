# ☁️ Azure Lift & Shift Migration Guide

> A complete step-by-step guide for migrating on-premises workloads to Azure using the Lift & Shift (rehost) strategy.

---

## Table of Contents

1. [Phase 1 — Assessment](#phase-1--assessment)
2. [Phase 2 — Subscription & Networking](#phase-2--subscription--networking)
3. [Phase 3 — Replication Setup](#phase-3--replication-setup)
4. [Phase 4 — Test Migration](#phase-4--test-migration)
5. [Phase 5 — Cutover (Production Migration)](#phase-5--cutover-production-migration)
6. [Phase 6 — DNS & Traffic](#phase-6--dns--traffic)
7. [Phase 7 — Post-Migration](#phase-7--post-migration)

---

## Phase 1 — Assessment

> Before touching Azure, thoroughly assess your on-premises environment. This is the most critical phase.

### Step 1: Install Azure Migrate on your on-prem environment

**Azure Portal Steps:**

1. Go to `portal.azure.com` → Search bar → type **"Azure Migrate"**
2. Click **Azure Migrate** → Click **+ Create project**
3. Fill in the following:

| Field | Value |
|---|---|
| Subscription | Your-Subscription-Name |
| Resource group | `rg-migration-prod` |
| Project name | `migration-project-2024` |
| Geography | Choose your nearest region (e.g. India) |

4. Click **Create**

> **What this does:** Creates a project container to track all your migration assets, assessments, and progress in one place.

---

### Step 2: Deploy Azure Migrate Appliance on-premises

**Azure Portal → Azure Migrate → Servers, databases and web apps:**

1. Under **Discovery and assessment** → Click **Discover**
2. Choose: **Are your machines virtualized?**

| Infrastructure | Choose |
|---|---|
| VMware | Yes, with VMware vSphere Hypervisor |
| Hyper-V | Yes, with Hyper-V |
| Physical | No, or other (AWS, GCP, physical) |

3. Click **Download** → Download the OVA (VMware) or VHD (Hyper-V) file
4. Deploy the downloaded appliance on your on-prem server
5. Open appliance browser UI → Complete setup wizard
6. Copy the **Project Key** from portal → Paste into appliance

> **Appliance requirements:** 8 vCPU, 16 GB RAM, 80 GB disk. Must have internet access and be able to reach all VMs you want to discover.

---

### Step 3: Run Discovery and wait for results

**On the Appliance:**
1. Enter your **vCenter credentials** (read-only account is fine)
2. Click **Start discovery**
3. Wait **15–30 minutes** for discovery to complete

**Back in Azure Portal:**
1. Azure Migrate project → **Overview** → You will see discovered servers count
2. Click on **Discovered servers** to see the full list
3. Review: CPU, Memory, Disk, OS for each VM

---

### Step 4: Create and run Assessment

**Azure Portal → Azure Migrate → Assessments tab → + Create assessment:**

1. Choose assessment type: **Azure VM** (for lift and shift)
2. Fill in assessment settings:

| Field | Recommended Value |
|---|---|
| Target location | Central India / East US (your region) |
| Storage type | Automatic (recommended) |
| Savings options | 1-year Reserved (saves ~40%) |
| VM series | D-series, E-series (general purpose) |
| Comfort factor | 1.3 (adds 30% headroom) |

3. Select machines → Click **Create assessment**
4. Wait 5 mins → Click on assessment → Review:
   - **Azure readiness** (Ready / Ready with conditions / Not ready)
   - **Monthly cost estimate**
   - **Recommended VM size**

> **Key output:** Export the assessment as Excel for stakeholder sign-off.

---

## Phase 2 — Subscription & Networking

> Set up your Azure subscription, landing zone, and networking before migrating any workloads.

### Step 1: Create or confirm Azure Subscription

1. `portal.azure.com` → Search **Subscriptions**
2. Confirm your subscription exists. If not, click **+ Add**
3. Note your **Subscription ID** — you'll need it often
4. Click subscription → **Access control (IAM)** → Assign yourself **Owner** role

---

### Step 2: Create Resource Groups

**Azure Portal → Resource Groups → + Create**

Create separate resource groups for logical grouping:

| Name | Purpose |
|---|---|
| `rg-network-prod` | All networking resources |
| `rg-migration-prod` | Migration project resources |
| `rg-web-servers` | Web tier VMs |
| `rg-db-servers` | Database tier VMs |

> **Naming convention:** `rg-{workload}-{environment}` e.g. `rg-webfarm-prod`, `rg-sql-dev`

---

### Step 3: Create Virtual Network (VNet)

**Azure Portal → Search "Virtual Networks" → + Create**

| Field | Value |
|---|---|
| Name | `vnet-prod-centralindia` |
| Region | Central India |
| Resource group | `rg-network-prod` |
| IPv4 address space | `10.0.0.0/16` |

**Add Subnets:**

| Subnet Name | Address Range |
|---|---|
| web-subnet | `10.0.1.0/24` |
| app-subnet | `10.0.2.0/24` |
| db-subnet | `10.0.3.0/24` |
| GatewaySubnet | `10.0.255.0/27` |

> ⚠️ **Important:** Make sure this VNet address space does NOT overlap with your on-prem network. If on-prem is `192.168.x.x`, using `10.0.0.0/16` is safe.

---

### Step 4: Set up VPN Gateway

**Azure Portal → Search "Virtual Network Gateway" → + Create**

| Field | Value |
|---|---|
| Name | `vpngw-prod` |
| Region | Central India |
| Gateway type | VPN |
| VPN type | Route-based |
| SKU | VpnGw1 (up to 650 Mbps) |
| Virtual network | `vnet-prod-centralindia` |
| Public IP | Create new → `vpngw-prod-pip` |

> **Note:** VPN Gateway takes 30–45 mins to deploy. After creation, note the public IP and configure your on-prem firewall/router to connect to it.

---

### Step 5: Create Network Security Groups (NSGs)

**Azure Portal → Search "Network Security Groups" → + Create**

| Field | Value |
|---|---|
| Name | `nsg-web` |
| Resource group | `rg-network-prod` |

**Inbound rules for web NSG:**

| Port | Protocol | Source | Action | Purpose |
|---|---|---|---|---|
| 443 | TCP | Internet | Allow | HTTPS traffic |
| 80 | TCP | Internet | Allow | HTTP traffic |
| 3389 | TCP | Your IP only | Allow | RDP access |
| 22 | TCP | Your IP only | Allow | SSH access |

**Associate NSG:** Go to Subnet → Associate NSG

---

## Phase 3 — Replication Setup

> Configure Azure Site Recovery (ASR) or Azure Migrate to replicate your on-premises VMs to Azure continuously.

### Step 1: Set up Azure Site Recovery Vault

**Azure Portal → Search "Recovery Services Vaults" → + Create**

| Field | Value |
|---|---|
| Name | `rsv-migration-prod` |
| Subscription | Your subscription |
| Resource group | `rg-migration-prod` |
| Region | Central India (must match target) |

Click **Review + create** → **Create**

Then: Open the vault → Click **Site Recovery** → **Prepare Infrastructure**

---

### Step 2: Configure replication in Azure Migrate

**Azure Portal → Azure Migrate → Migration and modernization → Replicate**

1. Choose source: **Yes, with VMware vSphere**
2. Choose Target Settings:

| Field | Value |
|---|---|
| Target subscription | Your subscription |
| Target resource group | `rg-web-servers` |
| Replication storage account | Create new: `stmigrationcache001` |
| Virtual network | `vnet-prod-centralindia` |
| Subnet | `web-subnet` or `app-subnet` |

**Compute Settings (per VM):**

| Field | Value |
|---|---|
| Target VM size | Use assessment recommendation (e.g. `Standard_D4s_v3`) |
| OS disk type | Premium SSD (for prod) |
| Data disk type | Premium SSD |

---

### Step 3: Start initial replication and monitor

**Azure Portal → Azure Migrate → Replicating machines**

1. After configuring, click **Replicate** to start
2. Initial sync takes **several hours to days** depending on VM size
3. Monitor status per machine:

| Status | Meaning |
|---|---|
| Initializing | First-time sync in progress |
| Replicating | Delta sync ongoing ✅ good state |
| Protected | Ready to test migrate ✅ |

> ⚠️ **Do NOT cut over until ALL machines show "Protected" status.**

---

## Phase 4 — Test Migration

> Always run a test migration first. This creates Azure VMs without affecting your on-prem production environment.

### Step 1: Run Test Migration

**Azure Portal → Azure Migrate → Replicating machines**

1. Select a VM → Click **Test migrate** (three-dot menu or top action)
2. Fill in:

| Field | Value |
|---|---|
| Recovery point | Latest (most recent) |
| Virtual network | Create a test VNet (isolated from production) |

3. Click **Test migration**
4. Wait 5–15 mins → A test VM is created (named like: `vmname-test`)

---

### Step 2: Validate on test VM

Connect via RDP/SSH and verify:

- [ ] OS boots and is accessible via RDP/SSH
- [ ] All Windows/Linux services are running
- [ ] Application can be launched and logged into
- [ ] Database connections are functional
- [ ] Disk drives and data are intact
- [ ] Required ports are open and accessible
- [ ] Performance is acceptable (CPU, memory not maxed)

---

### Step 3: Clean up test migration

**Azure Migrate → Replicating machines:**

1. Select VM → Click **Clean up test migration**
2. Tick: **Testing is complete, delete test VM**
3. Click **Cleanup** — this deletes test resources, replication continues normally

---

## Phase 5 — Cutover (Production Migration)

> This is the actual migration. Plan a maintenance window. Migrate dependencies first.

### Cutover order (always follow this sequence)

```
1. Database servers (SQL, MySQL, etc.)
       ↓
2. Application / backend servers
       ↓
3. Web / frontend servers
       ↓
4. Update DNS
```

> 📅 **Maintenance window:** Schedule 4–8 hours on a weekend night. Inform all stakeholders. Keep on-prem running until Azure is confirmed stable.

---

### Step 1: Stop on-prem applications and do final sync

1. **Stop your on-prem applications** gracefully (stop services, not the VM)
2. Let replication do one final delta sync
3. Wait until Azure Migrate shows **RPO < 5 minutes** for each VM

> Stopping apps before shutdown ensures files are closed, DB transactions committed, and buffers flushed — giving Azure a clean, consistent state.

---

### Step 2: Initiate Migrate (Cutover)

**Azure Portal → Azure Migrate → Replicating machines**

1. Select one or more VMs → Click **Migrate**
2. Fill in:

| Field | Value |
|---|---|
| Shutdown on-prem VM? | Yes (recommended — prevents split-brain) |
| Recovery point | Latest (most recent sync point) |

3. Click **Migrate**
4. Azure creates production VMs — takes 5–15 mins per VM
5. Monitor: Azure Migrate → **Jobs** → View migration job status

---

### Step 3: Configure Azure VM post-cutover

**Azure Portal → Virtual Machines → Your migrated VM**

**Assign Static Private IP:**
> VM → Networking → NIC → IP configuration → Set to **Static**

**Add Data Disks if needed:**
> VM → Disks → + Add data disk → Choose Premium SSD

**Enable Auto-shutdown (non-prod):**
> VM → Operations → Auto-shutdown → Enable → Set time

**Enable Monitoring:**
> VM → Monitoring → Diagnostics settings → Enable all metrics

---

## Phase 6 — DNS & Traffic

> Update DNS records to point to Azure VMs. This makes Azure go live for end users.

### Step 1: Assign Static Public IP (if internet-facing)

**Azure Portal → Virtual Machine → Networking → NIC → IP configurations**

1. Click the IP config → Enable **Public IP** → Create new
2. Settings:

| Field | Value |
|---|---|
| Name | `pip-webserver01` |
| SKU | Standard (required for zone redundancy) |
| Assignment | Static (never changes) |

3. Note the public IP shown after assignment

---

### Step 2: Update DNS records

**Azure Portal → DNS Zones → Your zone (e.g. yourcompany.com) → + Record set**

| Field | Value |
|---|---|
| Name | `www` (or `@` for root domain) |
| Type | A |
| TTL | `300` (5 min — keep low during migration) |
| IP address | Your Azure public IP |

> **TTL strategy:** Lower DNS TTL to `300` seconds **24 hours before** migration. After successful go-live, raise it back to `3600`.

---

### Step 3: Set up Azure Load Balancer (if multiple VMs)

**Azure Portal → Search "Load balancers" → + Create**

| Field | Value |
|---|---|
| Name | `lb-web-prod` |
| SKU | Standard |
| Type | Public (internet-facing) or Internal |
| Tier | Regional |

**After creation, configure:**
- **Backend pools** → Add your migrated VMs
- **Health probes** → HTTP on port 80 or 443
- **Load balancing rules** → Port 443 → Port 443

---

## Phase 7 — Post-Migration

> Secure, monitor, and optimize your Azure VMs. Don't skip this phase.

### Step 1: Enable Azure Backup

**Azure Portal → Virtual Machine → Backup (left menu)**

| Field | Value |
|---|---|
| Recovery Services vault | `rsv-migration-prod` |
| Backup policy | DefaultPolicy (daily, 30-day retention) |

Click **Enable backup** — first backup runs within 24 hours.

---

### Step 2: Enable Microsoft Defender for Cloud

**Azure Portal → Search "Microsoft Defender for Cloud" → Environment settings → Your subscription**

| Plan | Setting |
|---|---|
| Servers | On (Defender for Servers) |
| Storage | On |
| SQL | On (if you have SQL servers) |

Click **Save** → Go to **Recommendations** → Fix all HIGH severity items first.

---

### Step 3: Set up Azure Monitor and Alerts

**Azure Portal → Monitor → Alerts → + Create alert rule**

| Alert | Threshold | Action |
|---|---|---|
| CPU usage | > 85% for 5 mins | Email / SMS |
| Memory usage | > 90% for 5 mins | Email / SMS |
| Disk space | > 85% full | Email immediately |
| VM unavailable | Any | Email immediately |

**Create Action Group first:**
> Monitor → Action Groups → + Create → Add email/SMS/webhook

---

### Step 4: Decommission on-premises VMs

> Do this only after **2–4 weeks of stable Azure operation.**

- [ ] Azure VMs running stable for 2+ weeks
- [ ] All monitoring alerts healthy in Azure
- [ ] Users confirm application is working correctly
- [ ] Azure backups completed successfully at least 3 times
- [ ] Take final snapshot of on-prem VM before deletion
- [ ] **Remove replication** in Azure Migrate (stops billing for cache storage)
- [ ] Power off on-prem VMs (keep for 30 days before deleting)
- [ ] Cancel on-premises licenses that moved to Azure

**Stop replication billing:**
> Azure Migrate → Replicating machines → Select VM → **Stop replication**

---

### Step 5: Right-size and apply cost optimizations

**Azure Portal → Search "Advisor" → Cost tab**

| Optimization | Savings |
|---|---|
| Resize underutilized VMs | 10–30% |
| Buy 1-year Reserved Instances | ~40% |
| Buy 3-year Reserved Instances | ~60% |
| Use Spot Instances for dev/test | up to 90% |
| Auto-shutdown dev VMs at 7pm | Proportional |

> 💡 **Azure Hybrid Benefit:** If you have Windows Server or SQL Server licenses with Software Assurance, enable Azure Hybrid Benefit on each VM — saves up to 40% on compute costs.
> VM → Configuration → Azure Hybrid Benefit → **On**

---

## Quick Reference — Key Azure CLI Commands

```bash
# Login to Azure
az login

# Set subscription
az account set --subscription "Your-Subscription-Name"

# List all VMs
az vm list --output table

# Start a VM
az vm start --resource-group rg-web-servers --name myVM

# Stop a VM (deallocated — no compute charge)
az vm deallocate --resource-group rg-web-servers --name myVM

# Check VM status
az vm get-instance-view --resource-group rg-web-servers --name myVM --query instanceView.statuses

# Resize a VM
az vm resize --resource-group rg-web-servers --name myVM --size Standard_D2s_v3

# List NSG rules
az network nsg rule list --nsg-name nsg-web --resource-group rg-network-prod --output table
```

---

## Golden Rules

1. ✅ **Never skip test migration** — always validate before production cutover
2. ✅ **Lower DNS TTL** to 300 seconds, 24 hours before cutover
3. ✅ **Migrate in order** — DB → App → Web → DNS
4. ✅ **Enable Azure Hybrid Benefit** if you have existing Windows/SQL licenses
5. ✅ **Stop replication** in Azure Migrate after cutover to stop cache storage billing
6. ✅ **Keep on-prem VMs** for 30 days after migration before deleting
7. ✅ **Enable backup** on Day 1 in Azure — don't wait

---

*Guide created for Azure Lift & Shift (Rehost) migration strategy.*  
*Last updated: April 2026*
