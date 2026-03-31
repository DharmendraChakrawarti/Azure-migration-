# Azure Migration – Complete Learning Guide

-----

## PHASE 1: AZURE FUNDAMENTALS

### 1.1 What is Cloud Computing?

Cloud computing is the delivery of computing services—servers, storage, databases, networking, software, analytics, and intelligence—over the internet (“the cloud”) to offer faster innovation, flexible resources, and economies of scale.

**Key Cloud Models:**

- **IaaS (Infrastructure as a Service):** Rent virtualized hardware (VMs, storage, networking). You manage OS, middleware, apps. Example: Azure Virtual Machines.
- **PaaS (Platform as a Service):** Rent a platform for developing, running, and managing apps without dealing with infrastructure. Example: Azure App Service.
- **SaaS (Software as a Service):** Use software delivered over the internet. Example: Microsoft 365.
- **Serverless:** Run code without managing servers. Example: Azure Functions.

**Deployment Models:**

- **Public Cloud:** Resources owned and operated by a cloud provider (Azure, AWS, GCP).
- **Private Cloud:** Cloud infrastructure operated solely for a single organization.
- **Hybrid Cloud:** Combination of public and private clouds, connected together.

-----

### 1.2 Azure Core Architecture

**Geography → Region → Availability Zone → Datacenter**

- **Geography:** A discrete market (e.g., United States, Europe, Asia Pacific) that preserves data residency and compliance requirements.
- **Region:** A set of datacenters deployed within a latency-defined perimeter (e.g., East US, West Europe). Most services are deployed per-region.
- **Region Pair:** Each Azure region is paired with another region in the same geography for disaster recovery (e.g., East US ↔ West US).
- **Availability Zone:** Physically separate datacenters within a region, each with independent power, cooling, and networking. Provides high availability (99.99% SLA for VMs).
- **Availability Set:** Logical grouping of VMs within a datacenter to protect against hardware failures (fault domains) and planned maintenance (update domains).

**Azure Hierarchy:**

```
Management Group
  └── Subscription
        └── Resource Group
              └── Resource (VM, Storage, etc.)
```

- **Management Group:** Governs multiple subscriptions with policies and access controls.
- **Subscription:** Billing boundary and access control boundary. Each subscription has limits (quotas).
- **Resource Group:** A logical container for Azure resources. Resources in a group share the same lifecycle—deployed, updated, and deleted together.
- **Resource:** Any manageable item in Azure (VM, database, virtual network, etc.).

-----

### 1.3 Azure Interfaces

**Azure Portal (portal.azure.com)**

- Web-based GUI for managing Azure resources.
- Useful for beginners and for quick visual management.
- Supports dashboards, resource groups, cost analysis, and monitoring.

**Azure CLI**

```bash
# Install Azure CLI
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Login
az login

# Create a resource group
az group create --name MyRG --location eastus

# Create a VM
az vm create --resource-group MyRG --name MyVM --image UbuntuLTS --admin-username azureuser --generate-ssh-keys

# List all VMs
az vm list --output table
```

**Azure PowerShell**

```powershell
# Install
Install-Module -Name Az -AllowClobber -Scope CurrentUser

# Login
Connect-AzAccount

# Create resource group
New-AzResourceGroup -Name MyRG -Location EastUS

# Get all VMs
Get-AzVM
```

**Azure Cloud Shell**

- Browser-based shell (Bash or PowerShell) built into the Azure Portal.
- No installation needed. Has persistent storage via Azure File Share.

-----

### 1.4 Azure Pricing & Cost Management

**Pricing Models:**

- **Pay-as-you-go:** Pay only for what you use. No upfront commitment.
- **Reserved Instances:** Commit to 1 or 3 years for up to 72% savings on VMs and databases.
- **Spot Instances (Azure Spot VMs):** Use unused Azure capacity at up to 90% discount. Can be evicted with 30 seconds notice.
- **Azure Hybrid Benefit:** Use existing Windows Server or SQL Server licenses to save on Azure costs.
- **Dev/Test Pricing:** Discounted rates for dev/test workloads under Visual Studio subscriptions.

**Cost Management Tools:**

- **Azure Pricing Calculator:** Estimate costs before deployment.
- **TCO Calculator:** Compare on-premises vs Azure costs.
- **Azure Cost Management + Billing:** Monitor and analyze spending. Set budgets and alerts.
- **Azure Advisor:** Provides cost-saving recommendations.

-----

## PHASE 2: AZURE NETWORKING

### 2.1 Virtual Network (VNet)

A VNet is the fundamental building block for your private network in Azure. It enables Azure resources to securely communicate with each other, the internet, and on-premises networks.

**Core Concepts:**

- **Address Space:** CIDR block assigned to the VNet (e.g., 10.0.0.0/16).
- **Subnet:** Subdivision of a VNet’s address space. Resources are deployed into subnets (e.g., 10.0.1.0/24 for web tier, 10.0.2.0/24 for data tier).
- **VNet Peering:** Connect two VNets privately via the Azure backbone network. Traffic never leaves Azure. Can be in same or different regions (Global VNet Peering).
- **Service Endpoints:** Extend VNet identity to Azure services, allowing you to secure resources to your VNet.
- **Private Endpoints:** Bring Azure services (like Azure Storage, SQL DB) into your VNet with a private IP.

**Creating a VNet (CLI):**

```bash
az network vnet create \
  --name MyVNet \
  --resource-group MyRG \
  --address-prefix 10.0.0.0/16 \
  --subnet-name WebSubnet \
  --subnet-prefix 10.0.1.0/24
```

-----

### 2.2 Network Security Groups (NSG)

NSGs act as a virtual firewall for Azure resources. They contain inbound and outbound security rules.

**Rule Properties:**

- Priority (100–4096, lower = higher priority)
- Source/Destination (IP, CIDR, Service Tag, or ASG)
- Protocol (TCP, UDP, ICMP, Any)
- Port range
- Action (Allow or Deny)

**Default Rules (always present, cannot be deleted):**

- AllowVNetInBound (priority 65000): Allow all inbound within VNet.
- AllowAzureLoadBalancerInBound (priority 65001): Allow load balancer probes.
- DenyAllInBound (priority 65500): Deny all other inbound.

**Example – Allow HTTP inbound:**

```bash
az network nsg rule create \
  --resource-group MyRG \
  --nsg-name MyNSG \
  --name AllowHTTP \
  --protocol Tcp \
  --priority 100 \
  --destination-port-range 80 \
  --access Allow \
  --direction Inbound
```

**Application Security Groups (ASG):** Group VMs logically and define NSG rules based on ASG membership rather than IP addresses. Simplifies management.

-----

### 2.3 Hybrid Connectivity

**VPN Gateway:**

- Encrypted tunnel over public internet between Azure VNet and on-premises network.
- Supports Site-to-Site VPN (S2S), Point-to-Site VPN (P2S), and VNet-to-VNet connections.
- SKUs: Basic, VpnGw1–VpnGw5 (higher = more bandwidth and connections).
- Max bandwidth: ~10 Gbps with VpnGw5.

**ExpressRoute:**

- Private dedicated connection from on-premises to Azure (not over internet).
- Bandwidth: 50 Mbps to 100 Gbps.
- Lower latency, higher reliability, and better security than VPN.
- Requires working with a connectivity provider (AT&T, Equinix, etc.).
- Options: ExpressRoute Direct (dedicated port at Microsoft), ExpressRoute with a provider.

**When to use which:**

- VPN: Lower cost, quick setup, tolerates latency, backup for ExpressRoute.
- ExpressRoute: Mission-critical workloads needing low latency, high bandwidth, private connectivity.

-----

### 2.4 Load Balancing in Azure

|Service            |Layer          |Use Case                                      |
|-------------------|---------------|----------------------------------------------|
|Azure Load Balancer|L4 (TCP/UDP)   |Internal or public traffic distribution to VMs|
|Application Gateway|L7 (HTTP/HTTPS)|Web apps, SSL termination, URL routing        |
|Azure Front Door   |L7 Global      |Global HTTP load balancing, CDN, WAF          |
|Traffic Manager    |DNS-based      |Route traffic to endpoints worldwide          |

**Azure Load Balancer:**

- Distributes inbound traffic among healthy VMs.
- Health probes continuously check backend VM health.
- Public Load Balancer: Balances internet traffic to VMs.
- Internal Load Balancer: Balances traffic within VNet.

**Application Gateway:**

- L7 load balancer with Web Application Firewall (WAF).
- Features: SSL offloading, cookie-based session affinity, URL-based routing, multi-site hosting.
- WAF protects against OWASP Top 10 threats.

-----

### 2.5 Azure DNS

- Host your DNS domain in Azure and resolve DNS queries using Azure infrastructure.
- Supports both public DNS zones (for internet-facing domains) and private DNS zones (for VNet-internal name resolution).
- Supports A, AAAA, CNAME, MX, NS, SOA, SRV, and TXT records.
- Azure DNS uses Anycast networking for high availability and performance.

-----

### 2.6 Azure Firewall

- Managed, cloud-native firewall service with built-in high availability and unrestricted cloud scalability.
- Fully stateful firewall as a service.
- Features: FQDN filtering, network traffic filtering rules, outbound SNAT, inbound DNAT, threat intelligence integration.
- Azure Firewall Premium: Adds TLS inspection, IDPS, URL filtering.

**Azure DDoS Protection:**

- Basic: Automatically enabled for all Azure services.
- Standard: Enhanced mitigation against volumetric, protocol, and resource-layer attacks. Provides attack analytics, metrics, and alerts.

-----

## PHASE 3: COMPUTE

### 3.1 Azure Virtual Machines

VMs are IaaS resources providing full control over the OS and software.

**VM Size Families:**

- **General Purpose (B, D, DC):** Balanced CPU-to-memory. Suitable for testing, dev, small web servers.
- **Compute Optimized (F):** High CPU-to-memory. Suitable for medium traffic web servers, batch processing.
- **Memory Optimized (E, M, G):** High memory-to-CPU. Suitable for relational DBs, in-memory caches.
- **Storage Optimized (L):** High disk throughput and IOPS. Suitable for big data, SQL, NoSQL.
- **GPU (N):** For heavy graphics rendering, video editing, deep learning.
- **High Performance Compute (H):** Fastest CPUs, high-throughput network for HPC workloads.

**Managed Disks (OS & Data Disks):**

- Standard HDD: Low cost, dev/test.
- Standard SSD: Web servers, lightly used apps.
- Premium SSD: Production workloads, I/O intensive apps.
- Ultra Disk: Highest performance, mission-critical apps (up to 160,000 IOPS).

**Availability Options:**

- Availability Zones: Protect against datacenter failures (99.99% SLA).
- Availability Sets: Protect against hardware failures within a datacenter (99.95% SLA).
- VM Scale Sets: Auto-scale VMs based on load (supports up to 1000 VMs per scale set).

**Creating a VM (CLI):**

```bash
az vm create \
  --resource-group MyRG \
  --name MyVM \
  --image Win2019Datacenter \
  --admin-username azureadmin \
  --admin-password 'MySecurePass123!' \
  --size Standard_D2s_v3 \
  --vnet-name MyVNet \
  --subnet WebSubnet
```

-----

### 3.2 Azure App Service

A fully managed PaaS platform for hosting web apps, REST APIs, and mobile backends.

**Supported Languages:** .NET, .NET Core, Node.js, Python, Java, PHP, Ruby.

**App Service Plans (Tiers):**

- Free & Shared: Dev/test. No custom domain SSL. Shared compute.
- Basic: Custom domains/SSL. Dedicated compute. Manual scale (up to 3 instances).
- Standard: Auto-scaling, staging slots, daily backups, custom domains.
- Premium: Faster processors, more RAM, more auto-scale instances, VNet integration.
- Isolated: Dedicated Azure environment (App Service Environment). Highest scale and isolation.

**Key Features:**

- Deployment slots (staging, production) for zero-downtime deployments.
- Built-in CI/CD integration with Azure DevOps, GitHub, Bitbucket.
- Auto-scaling (scale out horizontally).
- Custom domains and SSL certificates.
- VNet Integration for private connectivity.
- Application Insights integration for monitoring.

-----

### 3.3 Azure Kubernetes Service (AKS)

AKS is a managed Kubernetes service for running containerized applications.

**Key Concepts:**

- **Pod:** Smallest deployable unit in Kubernetes. Contains one or more containers.
- **Node:** A VM that runs pods. AKS manages the node VMs.
- **Node Pool:** Group of nodes with the same VM size and configuration. AKS supports multiple node pools.
- **Cluster Autoscaler:** Automatically adds/removes nodes based on pending pods.
- **Horizontal Pod Autoscaler (HPA):** Scales the number of pod replicas based on CPU/memory metrics.

**AKS Features:**

- Managed control plane (Microsoft manages Kubernetes master nodes at no cost).
- Integrated monitoring with Azure Monitor and Container Insights.
- Azure AD integration for Kubernetes RBAC.
- Azure Policy for Kubernetes governance.
- Azure CNI or Kubenet networking.
- Automatic upgrades and security patches.

**Creating an AKS Cluster (CLI):**

```bash
az aks create \
  --resource-group MyRG \
  --name MyAKSCluster \
  --node-count 3 \
  --node-vm-size Standard_DS2_v2 \
  --enable-managed-identity \
  --generate-ssh-keys

# Get credentials
az aks get-credentials --resource-group MyRG --name MyAKSCluster

# Verify
kubectl get nodes
```

-----

### 3.4 Azure Functions (Serverless)

Run small pieces of code (“functions”) without managing infrastructure.

**Triggers (what starts execution):**

- HTTP Request
- Timer (cron schedule)
- Blob Storage (file upload)
- Queue Storage (message added)
- Event Hub / Event Grid / Service Bus
- Cosmos DB (document change)

**Hosting Plans:**

- **Consumption Plan:** Pay per execution. Auto-scale. Cold start possible.
- **Premium Plan:** Pre-warmed instances (no cold start). VNet integration. More powerful SKUs.
- **Dedicated (App Service) Plan:** Run on App Service VMs. Good when VMs are already running.

**Example HTTP-triggered Function (Python):**

```python
import azure.functions as func

def main(req: func.HttpRequest) -> func.HttpResponse:
    name = req.params.get('name') or 'World'
    return func.HttpResponse(f"Hello, {name}!")
```

-----

## PHASE 4: STORAGE & DATABASES

### 4.1 Azure Storage

**Storage Account Types:**

- **Standard General-Purpose v2 (GPv2):** Recommended. Supports all storage services. Redundancy options: LRS, GRS, ZRS, GZRS.
- **Premium Block Blobs:** High-performance for block blob workloads.
- **Premium File Shares:** High-performance for Azure Files workloads.
- **Premium Page Blobs:** For unmanaged VM disks.

**Storage Services:**

- **Blob Storage:** Object storage for unstructured data (images, videos, backups, logs). Three types: Block Blobs (general data), Append Blobs (logging), Page Blobs (random I/O, VHDs).
- **Azure Files:** Fully managed file share accessible via SMB or NFS protocol. Can be mounted on Windows, Linux, macOS. Great for lift-and-shift of file server workloads.
- **Queue Storage:** Message queue for decoupling application components. Max message size: 64 KB. Max queue size: 500 TB.
- **Table Storage:** NoSQL key-attribute store for structured data. Low cost. Good for storing flexible datasets.

**Redundancy Options:**

- **LRS (Locally Redundant Storage):** 3 copies within one datacenter. 11 nines durability. Cheapest.
- **ZRS (Zone-Redundant Storage):** 3 copies across availability zones. Protects against datacenter failure.
- **GRS (Geo-Redundant Storage):** 6 copies: 3 in primary region (LRS) + 3 in secondary region. Protects against regional outage.
- **GZRS (Geo-Zone-Redundant Storage):** ZRS in primary + LRS in secondary. Best protection.
- **RA-GRS / RA-GZRS:** Read access to secondary region for reads (read-access geo-redundant).

**Blob Access Tiers:**

- **Hot:** Frequently accessed data. Higher storage cost, lower access cost.
- **Cool:** Infrequently accessed (at least 30 days). Lower storage cost, higher access cost.
- **Archive:** Rarely accessed (at least 180 days). Lowest storage cost. Retrieval takes hours (rehydration needed).

-----

### 4.2 Azure SQL Database

Fully managed PaaS relational database engine built on SQL Server. Handles most database management functions (upgrades, patching, backups, monitoring).

**Deployment Options:**

- **Single Database:** One isolated database. Good for modern cloud apps.
- **Elastic Pool:** Multiple databases sharing a pool of resources (eDTUs or vCores). Cost-effective for variable workloads.
- **SQL Managed Instance:** Near 100% compatibility with on-premises SQL Server. Supports SQL Agent, cross-database queries, CLR. Good for lift-and-shift.

**Purchasing Models:**

- **DTU (Database Transaction Unit):** Bundled CPU, memory, I/O. Simpler. Basic, Standard, Premium tiers.
- **vCore:** Choose number of vCores and memory separately. Supports Azure Hybrid Benefit. General Purpose, Business Critical, Hyperscale.

**Key Features:**

- Built-in high availability (99.99% SLA).
- Automated backups (point-in-time restore up to 35 days).
- Geo-replication (up to 4 secondary replicas in different regions).
- Auto-failover groups for automatic failover.
- Advanced Threat Protection and Transparent Data Encryption (TDE).
- Elastic scaling without application changes.

-----

### 4.3 Azure Cosmos DB

Globally distributed, multi-model NoSQL database service.

**API Models:**

- **NoSQL (formerly Core/SQL):** JSON documents. SQL-like query language. Recommended for new projects.
- **MongoDB:** Wire protocol compatible with MongoDB. Easy migration from MongoDB.
- **Cassandra:** Compatible with Apache Cassandra CQL.
- **Gremlin:** Graph database API (Apache TinkerPop).
- **Table:** Key-value store (Azure Table Storage compatible).
- **PostgreSQL (Cosmos DB for PostgreSQL):** Distributed PostgreSQL using Citus extension.

**Key Features:**

- **Global Distribution:** Replicate data to any Azure region with one click.
- **Multi-region Writes:** Write to any region simultaneously.
- **5 Consistency Levels:** Strong, Bounded Staleness, Session, Consistent Prefix, Eventual.
- **SLA:** 99.999% availability for multi-region accounts.
- **Automatic Indexing:** All fields indexed automatically.
- **Throughput:** Measured in Request Units (RU/s). Can be provisioned (manual) or autoscale.

-----

### 4.4 Azure Database Migration Service (DMS)

Fully managed service for migrating databases to Azure with minimal downtime.

**Supported Sources → Targets:**

- SQL Server → Azure SQL DB, SQL Managed Instance, SQL Server on Azure VMs
- MySQL → Azure Database for MySQL
- PostgreSQL → Azure Database for PostgreSQL
- MongoDB → Cosmos DB for MongoDB
- Oracle → Azure SQL DB, Azure DB for PostgreSQL

**Migration Modes:**

- **Online (Continuous Sync):** Near-zero downtime. Data synced continuously until cutover.
- **Offline:** Downtime required. Data copied at a point in time.

**Steps for SQL Server Migration:**

1. Assess compatibility using Database Migration Assistant (DMA).
1. Create Azure DMS instance.
1. Create migration project (source: SQL Server, target: Azure SQL).
1. Configure source and target connection strings.
1. Select databases and tables.
1. Run migration and monitor progress.
1. Perform cutover (online mode).

-----

## PHASE 5: IDENTITY & SECURITY

### 5.1 Microsoft Entra ID (Azure Active Directory)

Cloud-based identity and access management service.

**Core Concepts:**

- **Tenant:** A dedicated instance of Azure AD representing your organization.
- **User:** An identity (employee, guest, service account).
- **Group:** Collection of users for bulk permissions assignment.
- **Service Principal:** Identity used by applications, services, or automation tools.
- **Managed Identity:** Automatically managed identity for Azure services. No credentials to manage.

**Authentication Methods:**

- Password + MFA (Multi-Factor Authentication)
- Passwordless (Windows Hello, FIDO2 security keys, Microsoft Authenticator app)
- Certificate-based authentication
- SAML 2.0, OAuth 2.0, OpenID Connect for app integration

**Key Features:**

- Single Sign-On (SSO) to thousands of SaaS apps.
- Conditional Access: Control access based on conditions (location, device compliance, risk level).
- Identity Protection: Detect and respond to identity-based risks.
- Privileged Identity Management (PIM): Just-in-time privileged access. Activate roles on-demand.
- B2B Collaboration: Invite external users.
- B2C: Customer identity solution for consumer apps.

-----

### 5.2 Role-Based Access Control (RBAC)

Controls who has access to Azure resources, what they can do, and what areas they have access to.

**RBAC Components:**

- **Security Principal:** User, group, service principal, or managed identity.
- **Role Definition:** Collection of permissions (e.g., read, write, delete).
- **Scope:** The set of resources the access applies to (Management Group, Subscription, Resource Group, or Resource).
- **Role Assignment:** Attaching a role definition to a security principal at a scope.

**Built-in Roles:**

- **Owner:** Full access including ability to delegate access to others.
- **Contributor:** Full access except cannot grant access to others.
- **Reader:** Read-only access to all resources.
- **User Access Administrator:** Manage user access to Azure resources.

**Custom Roles:** Define your own roles with specific action sets when built-in roles don’t fit.

**Assigning a Role (CLI):**

```bash
az role assignment create \
  --assignee user@example.com \
  --role "Contributor" \
  --scope /subscriptions/{subscriptionId}/resourceGroups/MyRG
```

-----

### 5.3 Azure Key Vault

Centralized service for storing and managing application secrets, keys, and certificates.

**What it stores:**

- **Secrets:** Connection strings, passwords, API keys, tokens.
- **Keys:** Cryptographic keys for encryption/decryption (RSA, EC). HSM-backed options available.
- **Certificates:** TLS/SSL certificates with automatic renewal.

**Access Control:**

- Vault Access Policy (legacy): Assign permissions at the vault level.
- Azure RBAC (recommended): Fine-grained permissions per secret/key/certificate.

**Best Practices:**

- Enable soft-delete and purge protection to prevent accidental deletion.
- Use Managed Identities to access Key Vault (no credentials in code).
- Enable diagnostic logging to monitor access.
- Use Private Endpoints to restrict access to within VNet.

**Accessing Secrets in Code (Python):**

```python
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient

credential = DefaultAzureCredential()
client = SecretClient(vault_url="https://MyVault.vault.azure.net/", credential=credential)
secret = client.get_secret("MySecretName")
print(secret.value)
```

-----

### 5.4 Microsoft Defender for Cloud

Cloud Security Posture Management (CSPM) and Cloud Workload Protection Platform (CWPP).

**Two Main Functions:**

- **Security Posture Assessment:** Continuously assesses your resources and provides a Secure Score. Higher score = better security posture.
- **Threat Protection (Defender Plans):** Detects and responds to threats across VMs, containers, databases, storage, Key Vault, DNS, and more.

**Defender Plans (paid):**

- Defender for Servers: Vulnerability assessment, just-in-time VM access, adaptive application controls.
- Defender for SQL: Detect anomalous activities, SQL injection protection.
- Defender for Storage: Detect unusual data access, malware uploads.
- Defender for Containers: Kubernetes cluster security.
- Defender for Key Vault: Detect unusual access patterns.
- Defender for App Service: Detect attacks targeting apps.

**Security Recommendations:** Defender provides actionable recommendations to improve security posture, such as enabling encryption, restricting ports, enabling MFA.

-----

## PHASE 6: MIGRATION TOOLS & METHODOLOGY

### 6.1 Azure Migrate

The primary hub for migrating to Azure. Provides discovery, assessment, and migration in one place.

**Integrated Tools:**

- **Discovery and Assessment:** Discover VMware VMs, Hyper-V VMs, and physical servers. Assess readiness, sizing, and estimated costs.
- **Server Migration:** Migrate VMs to Azure (agentless for VMware, agent-based for others).
- **Database Assessment (DMA):** Assess SQL Server databases for Azure SQL compatibility.
- **Database Migration (DMS):** Migrate databases to Azure managed databases.
- **Web App Migration:** Assess and migrate IIS web apps to Azure App Service.
- **Data Box Integration:** Migrate large amounts of offline data using Azure Data Box.

**Discovery Methods:**

- **VMware:** Deploy Azure Migrate appliance (OVA) in vCenter. Agentless discovery.
- **Hyper-V:** Deploy Azure Migrate appliance (VHD). Agentless discovery.
- **Physical Servers / AWS / GCP:** Deploy agent-based discovery.

**Assessment Report Includes:**

- Azure readiness (Ready, Ready with conditions, Not ready, Unknown)
- Recommended Azure VM size
- Monthly cost estimate
- Storage assessment
- Dependency analysis (what talks to what)

-----

### 6.2 Azure Site Recovery (ASR)

Disaster recovery and migration solution for VMs and physical servers.

**Use Cases:**

- **Disaster Recovery:** Replicate on-premises VMs to Azure. Failover in case of disaster.
- **Migration (Lift-and-Shift):** Replicate and migrate VMs to Azure with minimal downtime.

**How it Works (VMware to Azure):**

1. Set up a Configuration Server on-premises.
1. Install Mobility Service agent on source VMs.
1. Enable replication for VMs.
1. ASR continuously replicates data to Azure Storage (crash-consistent every 5 min, app-consistent every 1-12 hours).
1. Test failover (non-disruptive validation).
1. Actual failover/migration (cutover).

**Recovery Point Objective (RPO):** As low as 15 seconds for supported workloads.
**Recovery Time Objective (RTO):** Typically within minutes for Azure-to-Azure. Hours for on-premises-to-Azure depending on data size.

-----

### 6.3 The 5 R’s of Migration Strategy

**1. Rehost (Lift-and-Shift)**

- Move applications to Azure without changes.
- Fastest migration approach.
- Use Azure Migrate or Azure Site Recovery.
- Good for: applications where source code isn’t available, tight deadlines.
- Trade-off: Doesn’t leverage cloud-native features.

**2. Refactor (Lift-and-Reshape)**

- Minor code changes to make apps work better in Azure.
- Example: Move from IIS to Azure App Service, change connection strings for Azure SQL.
- Good for: Apps that need modest improvement but don’t need full rearchitecting.

**3. Rearchitect (Re-architect)**

- Significant code changes to leverage cloud-native capabilities.
- Example: Breaking a monolith into microservices on AKS, adopting event-driven architecture.
- Good for: Apps needing scalability, resilience, and cloud-native benefits.
- Highest effort but highest cloud benefit.

**4. Rebuild**

- Discard existing app and build a new cloud-native app from scratch.
- Use PaaS services, serverless, containers.
- Good for: Outdated apps where rebuilding is cheaper than modernizing.

**5. Replace (Retire & Replace)**

- Replace existing app with a SaaS solution.
- Example: Replace custom CRM with Salesforce, replace custom HR system with Workday.
- Good for: Commodity applications where SaaS provides better value.

-----

### 6.4 Migration Phases

**Phase 1 – Assess**

- Inventory all workloads (VMs, databases, apps, data).
- Analyze dependencies between workloads.
- Determine migration strategy per workload (5 R’s).
- Estimate costs using Azure Migrate.
- Identify risks and create mitigation plans.

**Phase 2 – Plan**

- Design Azure landing zone (foundation setup: networking, identity, policies).
- Define target Azure architecture per workload.
- Set up Azure subscriptions, management groups, and resource groups.
- Plan network connectivity (VPN/ExpressRoute, VNet design).
- Define governance, compliance, and security requirements.

**Phase 3 – Migrate**

- Set up migration tooling (Azure Migrate, ASR, DMS).
- Pilot migration with non-critical workloads.
- Run parallel testing (validate migrated workload against source).
- Migrate workloads in waves based on dependency groups.
- Update DNS, firewall rules, and connection strings.

**Phase 4 – Optimize**

- Right-size resources (resize VMs based on actual usage).
- Implement autoscaling.
- Set up cost management and budgets.
- Decommission on-premises resources.
- Apply Reserved Instances for predictable workloads.

**Phase 5 – Manage**

- Set up Azure Monitor, Log Analytics, Application Insights.
- Implement backup and disaster recovery.
- Apply Azure Policy for governance.
- Continuous security assessment with Defender for Cloud.
- Regular cost reviews and optimization.

-----

## PHASE 7: DEVOPS & INFRASTRUCTURE AS CODE

### 7.1 Azure DevOps

A set of development services for teams to plan work, collaborate on code, and build/deploy applications.

**Services:**

- **Azure Boards:** Agile project tracking. Work items, sprints, Kanban boards, backlogs.
- **Azure Repos:** Git-based version control. Unlimited private repos.
- **Azure Pipelines:** CI/CD pipelines. Build, test, and deploy automatically.
- **Azure Test Plans:** Manual and automated testing.
- **Azure Artifacts:** Package management (NuGet, npm, Maven, Python, Universal Packages).

**Pipeline YAML Example (CI/CD for a .NET app):**

```yaml
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

steps:
  - task: DotNetCoreCLI@2
    inputs:
      command: 'build'
      projects: '**/*.csproj'

  - task: DotNetCoreCLI@2
    inputs:
      command: 'test'
      projects: '**/*Tests.csproj'

  - task: AzureWebApp@1
    inputs:
      azureSubscription: 'MyServiceConnection'
      appName: 'MyWebApp'
      package: '$(System.DefaultWorkingDirectory)/**/*.zip'
```

-----

### 7.2 ARM Templates

Azure Resource Manager (ARM) templates are JSON files that define the infrastructure and configuration you want to deploy.

**Template Structure:**

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "vmName": {
      "type": "string",
      "defaultValue": "MyVM"
    }
  },
  "variables": {
    "location": "[resourceGroup().location]"
  },
  "resources": [
    {
      "type": "Microsoft.Compute/virtualMachines",
      "apiVersion": "2021-03-01",
      "name": "[parameters('vmName')]",
      "location": "[variables('location')]",
      "properties": { }
    }
  ],
  "outputs": { }
}
```

**Deploy ARM template:**

```bash
az deployment group create \
  --resource-group MyRG \
  --template-file azuredeploy.json \
  --parameters vmName=MyNewVM
```

-----

### 7.3 Bicep

Bicep is a domain-specific language (DSL) for deploying Azure resources declaratively. It’s a simpler, cleaner alternative to ARM JSON templates. Bicep transpiles to ARM JSON.

**Example – Deploy a Storage Account:**

```bicep
param storageAccountName string = 'mystorage${uniqueString(resourceGroup().id)}'
param location string = resourceGroup().location

resource storageAccount 'Microsoft.Storage/storageAccounts@2021-06-01' = {
  name: storageAccountName
  location: location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
}

output storageAccountId string = storageAccount.id
```

**Deploy Bicep:**

```bash
az deployment group create \
  --resource-group MyRG \
  --template-file main.bicep
```

-----

### 7.4 Terraform (on Azure)

HashiCorp Terraform is an open-source IaC tool that supports multiple cloud providers including Azure.

**Key Concepts:**

- **Provider:** Plugin that interacts with the cloud API (azurerm for Azure).
- **Resource:** Infrastructure component to be created.
- **State:** Terraform tracks deployed resources in a state file. Store remotely in Azure Blob Storage.
- **Plan:** Preview changes before applying.
- **Apply:** Execute the plan.

**Example – Create a Resource Group and VNet:**

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
  backend "azurerm" {
    resource_group_name  = "tfstate-rg"
    storage_account_name = "tfstatestorage"
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"
  }
}

provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "main" {
  name     = "MyRG"
  location = "East US"
}

resource "azurerm_virtual_network" "main" {
  name                = "MyVNet"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  address_space       = ["10.0.0.0/16"]
}
```

**Commands:**

```bash
terraform init      # Initialize providers
terraform plan      # Preview changes
terraform apply     # Apply changes
terraform destroy   # Destroy resources
```

-----

## PHASE 8: MONITORING & GOVERNANCE

### 8.1 Azure Monitor

Comprehensive monitoring solution for collecting, analyzing, and acting on telemetry data.

**Data Types Collected:**

- **Metrics:** Numerical time-series data (CPU %, disk IOPS, requests/sec). Stored for 93 days.
- **Logs:** Structured or unstructured text records stored in Log Analytics workspace.
- **Traces:** Distributed tracing for application performance (via Application Insights).

**Key Components:**

- **Log Analytics Workspace:** Central repository for logs from all sources. Query with KQL (Kusto Query Language).
- **Application Insights:** APM (Application Performance Monitoring) for web apps. Tracks requests, exceptions, dependencies, page views, custom events.
- **Alerts:** Trigger actions (email, SMS, webhook, auto-remediation) based on metric thresholds or log query results.
- **Action Groups:** Define who gets notified and how when an alert fires.
- **Workbooks:** Rich interactive reports combining metrics, logs, and visualizations.
- **Dashboards:** Customizable views of metrics and logs.

**KQL Query Example (find failed requests):**

```kusto
requests
| where success == false
| summarize count() by resultCode
| order by count_ desc
```

-----

### 8.2 Azure Policy

Enforce organizational standards and assess compliance at scale.

**How it Works:**

- Define a **Policy Definition** (what to check and what effect).
- Assign the policy to a **Scope** (management group, subscription, resource group).
- Azure evaluates resources against the policy and shows compliance state.

**Policy Effects:**

- **Deny:** Prevents non-compliant resources from being created/updated.
- **Audit:** Creates a compliance warning but allows the operation.
- **Append:** Adds additional fields to the request (e.g., tags).
- **Modify:** Add, update, or remove tags or properties on resources.
- **DeployIfNotExists:** Deploy a resource if it doesn’t exist (e.g., automatically deploy Defender agent).
- **AuditIfNotExists:** Audit if a related resource doesn’t exist.

**Policy Initiative (Policy Set):** Group multiple policies together into a single assignment. Example: Azure Security Benchmark initiative contains 200+ policies.

**Built-in Policy Examples:**

- Require SQL Server TDE.
- Allowed locations (restrict where resources can be created).
- Allowed VM SKUs.
- Require tags on resources.
- Deny public IP addresses.

-----

### 8.3 Azure Cost Management

**Features:**

- **Cost Analysis:** Visualize and analyze spending by service, resource group, tag, region.
- **Budgets:** Set spending thresholds and receive alerts when approaching or exceeding them.
- **Recommendations:** Azure Advisor integration provides cost optimization recommendations.
- **Cost Allocation:** Distribute shared costs across departments using tags.
- **Exports:** Schedule cost data exports to Azure Storage for custom reporting.

**Cost Optimization Strategies:**

- Right-size VMs based on actual CPU/memory utilization.
- Use Reserved Instances for predictable workloads (1-year or 3-year commitments).
- Use Azure Hybrid Benefit for Windows Server and SQL Server.
- Delete unused resources (orphaned disks, public IPs, unattached NICs).
- Use Spot VMs for interruptible, fault-tolerant workloads.
- Use lifecycle management policies on Blob Storage to move data to cooler tiers.

-----

### 8.4 Azure Advisor

Personalized recommendation engine that analyzes your resource configuration and usage.

**Recommendation Categories:**

- **Cost:** Identify unused resources, recommend Reserved Instances.
- **Security:** Identify security vulnerabilities (integrates with Defender for Cloud).
- **Reliability (formerly High Availability):** Improve continuity (e.g., enable backup, use availability zones).
- **Operational Excellence:** Deploy consistently, use monitoring effectively.
- **Performance:** Improve speed and responsiveness (e.g., upgrade disk SKU, scale up App Service).

-----

## PHASE 9: BACKUP & DISASTER RECOVERY

### 9.1 Azure Backup

Cloud-based backup service for protecting data.

**What can be backed up:**

- Azure VMs (full VM backup with application consistency).
- SQL Server in Azure VMs (transaction log backups every 15 min).
- Azure Files (file shares).
- SAP HANA in Azure VMs.
- On-premises servers and workloads (using Microsoft Azure Backup Server or DPM).
- Blobs (Azure Blob Storage operational backup).

**Recovery Services Vault:**

- Container for backup data. Stores backup copies with geo-redundant replication.
- Create per region. Each vault holds data for resources in that region.

**Retention Policies (GFS - Grandfather-Father-Son):**

- Daily recovery points: kept for 7-30 days.
- Weekly recovery points: kept for 1-52 weeks.
- Monthly recovery points: kept for 1-60 months.
- Yearly recovery points: kept for 1-10 years.

**Azure Backup Center:** Unified portal to monitor and manage all backup instances across vaults, subscriptions, regions.

-----

### 9.2 Disaster Recovery with ASR

**RTO (Recovery Time Objective):** Maximum acceptable time to restore service after a disaster.
**RPO (Recovery Point Objective):** Maximum acceptable data loss measured in time.

**ASR Replication Scenarios:**

- Azure VMs to another Azure region.
- VMware VMs to Azure.
- Hyper-V VMs to Azure.
- Physical servers to Azure.

**Failover Types:**

- **Test Failover:** Non-disruptive validation. Creates failover VM in isolated VNet. Source continues running.
- **Planned Failover:** No data loss. Used for planned maintenance. Stop source VM, failover, start target.
- **Unplanned Failover:** Used in actual disaster. Possible minimal data loss depending on RPO.

**Failback:** After failover to Azure, replicate back to on-premises and fail back when primary site is restored.

-----

## PHASE 10: CERTIFICATIONS ROADMAP

### Recommended Order for Azure Migration:

**Step 1: AZ-900 – Azure Fundamentals**

- Cloud concepts, Azure services, pricing, SLAs.
- No prerequisites. Good for anyone starting out.
- Exam: 40-60 questions. 700/1000 passing score.

**Step 2: AZ-104 – Azure Administrator**

- Manage Azure subscriptions, identity, storage, compute, networking, and monitoring.
- Prerequisite: 6 months practical experience recommended.
- Best for: IT admins and infrastructure engineers.

**Step 3: AZ-305 – Azure Solutions Architect Expert**

- Design solutions (compute, networking, storage, monitoring, identity, security).
- Prerequisite: Must pass AZ-104 (or AZ-303/304).
- Best for: Architects leading migration projects.

**Alternative Path – AZ-700 (Networking) or DP-300 (Database):**

- AZ-700: Deep-dive into Azure networking for network specialists.
- DP-300: Database administration on Azure for DBAs.

**Recommended Study Resources:**

- Microsoft Learn (learn.microsoft.com) – Free, official, hands-on modules.
- Azure Documentation (docs.microsoft.com) – Comprehensive reference.
- John Savill’s Technical Training (YouTube) – Highly recommended free video course.
- A Cloud Guru / Pluralsight – Paid video courses with labs.
- Whizlabs / MeasureUp – Practice exams.
- Azure Free Account – Get $200 credit and 55+ always-free services to practice.

-----

## QUICK REFERENCE: KEY SERVICES

|Category  |Service                            |Use Case                              |
|----------|-----------------------------------|--------------------------------------|
|Compute   |Azure VM                           |Full control IaaS workloads           |
|Compute   |App Service                        |Web apps and APIs (PaaS)              |
|Compute   |AKS                                |Containerized apps (Kubernetes)       |
|Compute   |Azure Functions                    |Serverless event-driven code          |
|Networking|VNet                               |Private network foundation            |
|Networking|VPN Gateway                        |Encrypted on-premises connectivity    |
|Networking|ExpressRoute                       |Dedicated private connectivity        |
|Networking|Application Gateway                |L7 load balancing + WAF               |
|Storage   |Blob Storage                       |Unstructured object storage           |
|Storage   |Azure Files                        |Managed SMB/NFS file shares           |
|Database  |Azure SQL DB                       |Managed relational SQL database       |
|Database  |Cosmos DB                          |Global NoSQL database                 |
|Database  |Azure Database for PostgreSQL/MySQL|Open-source managed databases         |
|Identity  |Entra ID                           |Cloud identity and SSO                |
|Security  |Key Vault                          |Secrets, keys, certificates           |
|Security  |Defender for Cloud                 |Security posture + threat protection  |
|Migration |Azure Migrate                      |VM and server migration hub           |
|Migration |Azure Site Recovery                |DR and lift-and-shift migration       |
|Migration |Database Migration Service         |Database migration                    |
|IaC       |ARM Templates / Bicep              |Native Azure IaC                      |
|IaC       |Terraform                          |Multi-cloud IaC                       |
|DevOps    |Azure DevOps                       |CI/CD pipelines and project management|
|Monitoring|Azure Monitor                      |Metrics, logs, alerts                 |
|Monitoring|Application Insights               |App performance monitoring            |
|Governance|Azure Policy                       |Compliance and standards enforcement  |
|Governance|Cost Management                    |Spending visibility and optimization  |
|Backup    |Azure Backup                       |Backup for VMs, DBs, files            |

-----

*Last Updated: 2024 | Study each phase in order, practice on an Azure Free Account, and aim for AZ-900 → AZ-104 → AZ-305 certification path.*
