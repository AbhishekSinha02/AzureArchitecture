# NSG — Allow, Deny, and Exception Access
### 🟢 Beginner → 🟡 Intermediate → 🔴 Advanced

---

## Q1. What is an NSG and how does rule evaluation work?

**Scenario:** You're asked to lock down a subnet so only HTTPS traffic is allowed in. Where does the rule live and how does Azure decide what to allow?

**What an NSG is:**
- A **stateful packet filter** (Layer 4 — TCP/UDP/ICMP) attached to a **subnet** or a **NIC**
- "Stateful" means: if you allow an inbound TCP connection, the return traffic is automatically allowed — you don't need an outbound rule for it
- Contains **Inbound rules** and **Outbound rules**
- Rules have a **priority** (100–4096) — **lower number = evaluated first**

**Evaluation flow:**
```
Packet arrives at subnet
         │
         ▼
  Rule Priority 100 → Does it match? YES → ALLOW/DENY and stop
         │ NO
         ▼
  Rule Priority 200 → Does it match? YES → ALLOW/DENY and stop
         │ NO
         ▼
  ... continue through all custom rules ...
         │ NO match
         ▼
  Default Rules (always exist, cannot be deleted):
    65000: Allow VirtualNetwork inbound
    65001: Allow AzureLoadBalancer inbound
    65500: Deny All inbound  ← everything else is denied
```

**Built-in default rules (always present):**
```
INBOUND defaults:
  65000  Allow  VirtualNetwork → VirtualNetwork  (VNet-to-VNet within same VNet)
  65001  Allow  AzureLoadBalancer → Any          (health probes from Azure LB)
  65500  Deny   Any → Any                        (implicit deny-all)

OUTBOUND defaults:
  65000  Allow  VirtualNetwork → VirtualNetwork
  65001  Allow  Any → Internet                   (all outbound to internet allowed by default!)
  65500  Deny   Any → Any
```

> ⚠️ **Key insight:** By default, **outbound internet IS allowed** from VMs. You must explicitly deny it if you want to block it.

---

## Q2. How do you allow only HTTPS inbound to a web application subnet?

**Scenario:** You have a subnet with web servers. You want only HTTPS (443) inbound from the internet, and SSH (22) only from your corporate network (10.0.0.0/8). Everything else blocked.

**NSG Rules needed:**

| Priority | Name | Direction | Action | Source | Destination | Port |
|----------|------|-----------|--------|--------|-------------|------|
| 100 | allow-https-inbound | Inbound | Allow | Internet | * | 443 |
| 200 | allow-ssh-corporate | Inbound | Allow | 10.0.0.0/8 | * | 22 |
| 300 | allow-lb-probes | Inbound | Allow | AzureLoadBalancer | * | * |
| 4000 | deny-all-inbound | Inbound | Deny | * | * | * |

```bash
# Create NSG
az network nsg create \
  --name nsg-web-subnet \
  --resource-group rg-app \
  --location canadacentral

# Allow HTTPS from internet
az network nsg rule create \
  --nsg-name nsg-web-subnet \
  --resource-group rg-app \
  --name "allow-https-inbound" \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes Internet \
  --destination-port-ranges 443

# Allow SSH from corporate only
az network nsg rule create \
  --nsg-name nsg-web-subnet \
  --resource-group rg-app \
  --name "allow-ssh-corporate" \
  --priority 200 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes "10.0.0.0/8" \
  --destination-port-ranges 22

# Allow LB health probes (CRITICAL — removing this breaks LB)
az network nsg rule create \
  --nsg-name nsg-web-subnet \
  --resource-group rg-app \
  --name "allow-lb-probes" \
  --priority 300 \
  --direction Inbound \
  --access Allow \
  --source-address-prefixes AzureLoadBalancer \
  --destination-port-ranges "*"

# Deny everything else
az network nsg rule create \
  --nsg-name nsg-web-subnet \
  --resource-group rg-app \
  --name "deny-all-remaining" \
  --priority 4000 \
  --direction Inbound \
  --access Deny \
  --source-address-prefixes "*" \
  --destination-port-ranges "*"

# Attach NSG to subnet
az network vnet subnet update \
  --name snet-web \
  --vnet-name vnet-app \
  --resource-group rg-app \
  --network-security-group nsg-web-subnet
```

> ⚠️ **Gotcha:** Forgetting the `AzureLoadBalancer` rule after adding a deny-all causes the Load Balancer to mark all backend VMs as unhealthy because health probe packets get dropped.

---

## Q3. How do you deny all outbound internet access for a database subnet?

**Scenario:** Your database subnet has SQL VMs that should only talk to the application tier. They must never reach the internet — not even for Windows Update (use WSUS internally). How do you enforce this?

**NSG outbound rules:**

```
Current (default): All outbound internet is ALLOWED
Goal: Block all outbound internet, allow only to application tier
```

```bash
# Allow outbound to app tier only
az network nsg rule create \
  --nsg-name nsg-db-subnet \
  --resource-group rg-data \
  --name "allow-outbound-to-app-tier" \
  --priority 100 \
  --direction Outbound \
  --access Allow \
  --protocol Tcp \
  --destination-address-prefixes "10.1.0.0/24" \  # App subnet
  --destination-port-ranges "1433"                 # SQL port

# Allow outbound to Azure services (monitoring, Key Vault, etc.)
az network nsg rule create \
  --nsg-name nsg-db-subnet \
  --resource-group rg-data \
  --name "allow-azure-services" \
  --priority 200 \
  --direction Outbound \
  --access Allow \
  --destination-address-prefixes AzureCloud \
  --destination-port-ranges "443"

# BLOCK all outbound internet — overrides default allow-internet rule
az network nsg rule create \
  --nsg-name nsg-db-subnet \
  --resource-group rg-data \
  --name "deny-internet-outbound" \
  --priority 1000 \
  --direction Outbound \
  --access Deny \
  --source-address-prefixes "*" \
  --destination-address-prefixes Internet \
  --destination-port-ranges "*"
```

> 🔑 **Why priority 1000 for the deny-internet rule?** The built-in `Allow Internet Outbound` is at priority 65001. Any custom Deny rule with a lower number (e.g., 1000) takes precedence and overrides it.

---

## Q4. How do you give a single VM exceptional access while keeping everything else locked down?

**Scenario:** You have a locked-down production subnet. A vendor VM needs temporary access to port 9090 on a specific internal server for a 2-day engagement. How do you add this without touching the production NSG permanently?

**Two approaches:**

**Option A: Time-limited NSG rule (policy + automation)**
```bash
# Add the exception rule at a low priority number (runs before the deny-all)
az network nsg rule create \
  --nsg-name nsg-prod-subnet \
  --resource-group rg-prod \
  --name "temp-vendor-exception-20240115" \
  --priority 150 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes "10.5.0.10" \  # Vendor VM IP
  --destination-address-prefixes "10.1.0.25" \  # Target server
  --destination-port-ranges "9090"

# Schedule removal via Azure Automation runbook (run 2 days later)
# Runbook: az network nsg rule delete --nsg-name nsg-prod-subnet --name "temp-vendor-exception-20240115"
```

**Option B: NIC-level NSG (cleaner, doesn't touch subnet NSG)**

```
Subnet NSG (deny-all)
      +
NIC NSG on vendor VM's NIC (allow port 9090 to target)

Azure evaluates BOTH — traffic must pass through both NSGs
But the NIC NSG can have more permissive rules for just this VM
```

```bash
# Create NIC-level NSG for the vendor VM
az network nsg create \
  --name nsg-vendor-vm-nic \
  --resource-group rg-prod

az network nsg rule create \
  --nsg-name nsg-vendor-vm-nic \
  --resource-group rg-prod \
  --name "allow-vendor-outbound" \
  --priority 100 \
  --direction Outbound \
  --access Allow \
  --destination-address-prefixes "10.1.0.25" \
  --destination-port-ranges "9090"

# Attach to vendor VM's NIC
az network nic update \
  --name vendor-vm-nic \
  --resource-group rg-prod \
  --network-security-group nsg-vendor-vm-nic
```

> ⚠️ **Important:** NIC NSG + Subnet NSG = **both** must allow the traffic. The more restrictive one wins. NIC-level rules don't override subnet NSG rules — they stack.

---

## Q5. What are Service Tags and when do you use them instead of IP ranges?

**Scenario:** You want to allow your VMs to reach Azure Monitor and Key Vault, but you don't know which IP ranges those services use and they change periodically. How do you write NSG rules that are future-proof?

**Service Tags** are named groups of IP prefixes managed by Microsoft. You use the name instead of hardcoding IPs.

**Commonly used Service Tags:**

| Tag | What it covers |
|-----|---------------|
| `Internet` | All IPs not in a private range |
| `AzureCloud` | All Azure datacenter IPs (global) |
| `AzureCloud.CanadaCentral` | Azure IPs in Canada Central only |
| `AzureLoadBalancer` | Azure Load Balancer health probe source |
| `VirtualNetwork` | All address ranges within the VNet + peered VNets + on-prem ranges via VPN |
| `Storage` | Azure Storage service IPs |
| `Sql` | Azure SQL service IPs |
| `AzureMonitor` | Azure Monitor, Log Analytics, App Insights |
| `KeyVault` | Azure Key Vault service IPs |
| `AzureActiveDirectory` | Entra ID authentication endpoints |

```bash
# Allow outbound to Azure Monitor (diagnostics, metrics)
az network nsg rule create \
  --nsg-name nsg-app-subnet \
  --resource-group rg-app \
  --name "allow-azure-monitor" \
  --priority 110 \
  --direction Outbound \
  --access Allow \
  --destination-address-prefixes AzureMonitor \
  --destination-port-ranges "443"

# Allow outbound to Key Vault
az network nsg rule create \
  --nsg-name nsg-app-subnet \
  --resource-group rg-app \
  --name "allow-key-vault" \
  --priority 120 \
  --direction Outbound \
  --access Allow \
  --destination-address-prefixes KeyVault \
  --destination-port-ranges "443"
```

> 💡 **When NOT to use service tags:** When you have Private Endpoints — traffic to your Private Endpoint goes to a VNet IP, not the service tag. Use the specific private IP or subnet CIDR in that case.

---

## Q6. What are Application Security Groups (ASGs) and how do they simplify NSG rules?

**Scenario:** You have 20 web server VMs, 10 app server VMs, and 5 database VMs. Writing NSG rules with individual IPs or subnets is brittle — IPs change when VMs are replaced. How do you write rules by role?

**ASGs are logical groups of NICs — like tags for NSG rules:**

```
Without ASG:                        With ASG:
source: 10.1.0.5, 10.1.0.6...      source: asg-web-servers
  → 10.2.0.10, 10.2.0.11...           → asg-app-servers
  → TCP 8080                             → TCP 8080

When a VM is replaced, you just     When a VM is replaced, you just
update 20 NSG rules                 add its NIC to the ASG — no
                                    NSG rule changes needed
```

```bash
# Create ASGs
az network asg create --name asg-web-servers --resource-group rg-app
az network asg create --name asg-app-servers --resource-group rg-app
az network asg create --name asg-db-servers  --resource-group rg-app

# Assign a VM's NIC to an ASG
az network nic update \
  --name web-vm1-nic \
  --resource-group rg-app \
  --application-security-groups asg-web-servers

# Write NSG rule using ASG (not IPs)
az network nsg rule create \
  --nsg-name nsg-app-subnet \
  --resource-group rg-app \
  --name "web-to-app-tier" \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --source-asgs asg-web-servers \
  --destination-asgs asg-app-servers \
  --destination-port-ranges "8080"

az network nsg rule create \
  --nsg-name nsg-app-subnet \
  --resource-group rg-app \
  --name "app-to-db-tier" \
  --priority 110 \
  --direction Inbound \
  --access Allow \
  --source-asgs asg-app-servers \
  --destination-asgs asg-db-servers \
  --destination-port-ranges "1433"
```

> 🔑 **ASG constraint:** All NICs in an ASG must be in the **same VNet**. You can't group NICs across VNets in one ASG.

---

## Q7. How do you audit and review what your NSG is actually allowing?

**Scenario:** After a security audit, you need to prove exactly what traffic is permitted on your production subnet. How do you get a readable view?

**Method 1: Effective Security Rules (best for a single NIC)**
```bash
# Shows the combined rules from both NIC-level and Subnet-level NSGs
az network nic show-effective-nsg \
  --name prod-vm-nic \
  --resource-group rg-prod \
  --output table
```

**Method 2: NSG Flow Logs (actual traffic evidence)**
```bash
# Enable flow logs on NSG
az network watcher flow-log create \
  --location canadacentral \
  --resource-group rg-networking \
  --name flowlog-nsg-prod \
  --nsg nsg-prod-subnet \
  --storage-account st-flowlogs-prod \
  --workspace /subscriptions/.../workspaces/law-prod \
  --enabled true \
  --format JSON \
  --log-version 2

# Query allowed traffic in Log Analytics
AzureNetworkAnalytics_CL
| where SubType_s == "FlowLog"
| where FlowStatus_s == "A"   // A = Allowed, D = Denied
| where SrcIP_s startswith "10.1"
| summarize count() by DestIP_s, DestPort_d, L4Protocol_s
| order by count_ desc
```

**Method 3: IP Flow Verify (test a specific flow)**
```bash
# "Would this specific packet be allowed or denied?"
az network watcher test-ip-flow \
  --vm prod-web-vm \
  --direction Inbound \
  --protocol TCP \
  --local 10.1.0.5:80 \
  --remote 52.1.2.3:54321 \
  --resource-group rg-prod
# Output: Access: Allow/Deny | NSG rule: <rule-name>
```

> 💡 **Deep dive hint:** NSG Flow Log version 2 adds traffic volume (bytes/packets) per flow — essential for detecting data exfiltration patterns where an unusual volume of data leaves through an allowed port.
