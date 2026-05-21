# Azure Fundamentals — Beginner Interview Questions

> Focus: Core concepts, service categories, resource model, regions, identity basics.
> Target: Cloud practitioner / associate level. Use as warm-up before deeper topics.

---

## Section 1: Cloud & Azure Basics

**Q1. What is the difference between IaaS, PaaS, and SaaS? Give an Azure example of each.**

**Answer:**

| Model | You manage | Azure manages | Azure Example |
|-------|-----------|---------------|---------------|
| IaaS | OS, runtime, app, data | Hardware, network, hypervisor | Azure Virtual Machines |
| PaaS | App code, data | OS, runtime, patching, scaling | Azure App Service, AKS |
| SaaS | Nothing (configuration only) | Everything | Microsoft 365, Dynamics 365 |

Key point: As you move from IaaS → SaaS, you trade **control** for **operational simplicity**. For enterprises migrating on a deadline, IaaS (rehost) is fastest. For new cloud-native builds, PaaS is preferred.

---

**Q2. What is an Azure Region and an Availability Zone? How are they different?**

**Answer:**

- **Region**: A geographic area containing one or more datacenters (e.g., Canada Central = Toronto). Regions are the primary unit of data residency.
- **Availability Zone (AZ)**: Physically separate datacenters within a region with independent power, cooling, and networking. Azure has 3 AZs per zone-enabled region.

**Why it matters:**
- Single-VM → no SLA
- VM in an Availability Set → 99.95% SLA
- VM across Availability Zones → 99.99% SLA

**Canada specifics:** Canada Central (Toronto) supports AZs. Canada East (Quebec City) is the DR paired region for OSFI compliance.

---

**Q3. What is a Resource Group in Azure and what are the rules around it?**

**Answer:**

A Resource Group is a logical container for Azure resources. Rules:
- Every resource belongs to exactly **one** resource group
- Resources in a group can span multiple regions
- Deleting a resource group deletes all resources inside it
- You apply RBAC, policies, and tags at the RG level (they inherit down)
- Resources can interact across resource groups

**Best practice for enterprise:**
```
rg-networking-prod        # Hub VNet, Firewall, DNS
rg-aks-prod               # AKS cluster, node pools
rg-data-prod              # Storage, CosmosDB, SQL
rg-monitoring-prod        # Log Analytics, App Insights
```
Separate RGs per environment (dev/staging/prod) enables independent lifecycle management.

---

**Q4. What is Azure Resource Manager (ARM) and why does it matter?**

**Answer:**

ARM is the deployment and management layer for Azure. Every Azure operation — portal, CLI, PowerShell, Terraform — goes through ARM.

Key features:
- **Declarative templates** (ARM JSON / Bicep) for infrastructure as code
- **Idempotent deployments** — re-running the same template is safe
- **RBAC enforcement** at every operation
- **Dependency management** — ARM resolves resource creation order
- **Tags and policies** applied consistently

Modern preference: **Bicep** (cleaner syntax, compiles to ARM) or **Terraform** (multi-cloud, widely used in enterprise). ARM JSON is legacy but still valid.

---

**Q5. Explain the Azure subscription hierarchy.**

**Answer:**

```
Management Groups          ← policy and RBAC inheritance (enterprise-wide)
  └── Subscriptions        ← billing boundary, policy scope
        └── Resource Groups ← lifecycle boundary, RBAC scope
              └── Resources  ← actual services
```

- **Management Groups**: Apply governance at scale (e.g., "all production subscriptions must enforce HTTPS")
- **Subscriptions**: Separate billing, apply subscription-level Azure Policy, isolate blast radius
- **Resource Groups**: Logical grouping, shared lifecycle

**Enterprise pattern:** One subscription per environment per workload (e.g., `sub-banking-prod`, `sub-banking-dev`). Management Groups enforce enterprise-wide policies like "deny public blob access."

---

## Section 2: Core Services

**Q6. What is the difference between Azure Blob Storage and Azure Data Lake Storage Gen2?**

**Answer:**

| Feature | Azure Blob Storage | ADLS Gen2 |
|---------|-------------------|-----------|
| Namespace | Flat (container/blob) | Hierarchical (folders) |
| ACLs | Container-level | File and folder level (POSIX-like) |
| Analytics optimized | No | Yes (Hadoop-compatible) |
| Use case | Object storage, backups, static content | Big data, analytics, data lake |
| Protocol | REST / HTTPS | REST + HDFS + ABFS |

ADLS Gen2 is built **on top of** Azure Blob Storage — it adds a hierarchical namespace. You enable it as a feature on a storage account. For any analytics workload (Databricks, Synapse, ADF), always use ADLS Gen2.

---

**Q7. What is Azure Active Directory (Entra ID) and how does it differ from on-premises Active Directory?**

**Answer:**

| Aspect | On-premises AD | Azure Entra ID |
|--------|---------------|----------------|
| Protocol | LDAP, Kerberos, NTLM | OAuth 2.0, OIDC, SAML |
| Structure | Domains, OUs, GPOs | Tenants, groups, conditional access |
| Join type | Domain join | Azure AD Join, Hybrid Join |
| Federation | AD FS | Directly built-in |
| MFA | Requires 3rd party or ADFS | Built-in (free tier included) |

Entra ID is an **identity platform**, not just a directory. It powers:
- SSO across thousands of SaaS apps
- Conditional Access (block logins from risky locations)
- Managed Identities (no password required for Azure service-to-service auth)
- Privileged Identity Management (just-in-time admin access)

---

**Q8. What is a Managed Identity and why is it preferred over service principals with secrets?**

**Answer:**

A Managed Identity is an automatically managed identity in Entra ID assigned to an Azure resource. Azure handles credential rotation — **you never store or manage a password or certificate.**

**Types:**
- **System-assigned**: Tied to one resource, deleted with the resource
- **User-assigned**: Standalone, can be assigned to multiple resources

**Why preferred:**
- No secrets to rotate, store, or accidentally expose
- Eliminates credential-related security incidents
- Works with RBAC on any Azure resource
- Native integration: App Service, Functions, AKS, VMs, ADF all support it

**Example:** An ADF pipeline reads from ADLS Gen2. Instead of storing a storage account key in ADF's linked service, you assign ADF a Managed Identity and grant it `Storage Blob Data Reader` on the storage account.

---

**Q9. What is Azure Key Vault and what does it store?**

**Answer:**

Key Vault is a secure, managed store for:
- **Secrets**: Connection strings, API keys, passwords
- **Keys**: Encryption keys (HSM-backed for premium tier)
- **Certificates**: TLS/SSL certificates with auto-renewal

**Why it matters in production:**
- Centralizes secret management (no secrets in code, config files, or environment variables)
- Full audit log: who accessed what secret, when
- Soft-delete + purge protection prevents accidental deletion
- RBAC: `Key Vault Secrets User` role to read secrets, `Key Vault Administrator` to manage

**Integration pattern:** App Service / AKS pod reads secrets from Key Vault via Managed Identity at startup. No passwords anywhere in the deployment pipeline.

---

**Q10. What is the difference between Azure Monitor, Log Analytics, and Application Insights?**

**Answer:**

```
Azure Monitor          ← umbrella platform (collects everything)
  ├── Log Analytics     ← centralized log store + query engine (KQL)
  ├── Application Insights ← application-level APM (traces, exceptions, custom metrics)
  ├── Metrics           ← time-series numeric data (CPU, requests/sec)
  └── Alerts            ← rules that trigger on metrics or log query results
```

- **Log Analytics Workspace**: Where all logs land. Query with KQL (Kusto Query Language).
- **Application Insights**: Instrumented in your app code. Gives request traces, dependency maps, exception details, and custom events.
- **Azure Monitor Metrics**: Near-real-time numeric metrics. Good for dashboards and autoscale triggers.

**In practice:** Always route all diagnostic logs to a central Log Analytics Workspace. Connect Application Insights to the same workspace (workspace-based mode) for cross-correlation.

---

## Section 3: Quick-Fire Concepts

**Q11. What is the difference between a VNet and a Subnet?**

A **VNet** (Virtual Network) is the isolated network boundary in Azure — like a private network you own. A **Subnet** is a range of IPs within a VNet used to segment resources. NSGs attach to subnets to control traffic. Each resource (VM, AKS node, private endpoint) lives in a subnet.

**Q12. What is Azure CDN and when would you use it?**

Azure CDN caches content at edge nodes globally, serving static assets (images, JS, CSS, videos) from a location close to the user. Use it for: static websites, media delivery, software downloads. Reduces origin server load and improves latency for geographically distributed users.

**Q13. What is the difference between horizontal and vertical scaling?**

- **Vertical scaling** (scale up): Bigger machine — more CPU/RAM on the same VM. Has a ceiling and requires restart.
- **Horizontal scaling** (scale out): More instances of the same size. Preferred for cloud-native workloads. Requires stateless design (session state in Redis, not in-memory).

**Q14. What is a SLA and what does 99.9% mean in practice?**

SLA = Service Level Agreement — the uptime guarantee Azure provides. 99.9% = **8.7 hours of allowed downtime per year**. 99.99% = **52 minutes per year**. Multi-region active-active architectures target 99.99%+. Single-VM with no redundancy has no SLA (0%).

**Q15. What is the difference between Azure Firewall and Network Security Groups (NSG)?**

| Feature | NSG | Azure Firewall |
|---------|-----|----------------|
| Layer | Layer 4 (TCP/UDP ports) | Layer 7 (FQDN, URLs, protocols) |
| Scope | Subnet or NIC | VNet or Hub |
| State | Stateless rules | Stateful |
| Cost | Free | ~$1.25/hr + data processing |
| FQDN rules | No | Yes |
| Threat intelligence | No | Yes |

NSGs are the baseline — apply them everywhere. Azure Firewall is the centralized gateway in Hub-Spoke topology for east-west and north-south traffic inspection.
