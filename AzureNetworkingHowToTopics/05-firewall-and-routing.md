# Azure Firewall & Routing — Deep Dive
### 🟡 Intermediate → 🔴 Advanced

---

## Q1. What are the three rule types in Azure Firewall and when do you use each?

**Scenario:** You need to control outbound internet access, allow only specific HTTPS destinations, and block everything else. Which rule type handles what?

**Three rule collection types — evaluated in order:**

```
Traffic arrives at Firewall
         │
         ▼
1. DNAT Rules          ← Inbound NAT (translate public IP → internal VM)
   (if match → forward)
         │
         ▼
2. Network Rules       ← Layer 4 (IP/port/protocol)
   (if match → allow/deny and STOP)
         │ no match
         ▼
3. Application Rules   ← Layer 7 (FQDN, URLs, HTTP/HTTPS categories)
   (if match → allow/deny)
         │ no match
         ▼
4. Default: DENY ALL
```

**When to use each:**

| Rule Type | Use for | Example |
|-----------|---------|---------|
| **DNAT** | Expose internal services on Firewall's public IP | `Firewall:443` → internal web server `10.1.0.5:443` |
| **Network Rules** | IP-to-IP on specific ports (Layer 4) | Allow AKS nodes → CosmosDB private IP on 443 |
| **Application Rules** | FQDN or URL-based (Layer 7, HTTPS inspected) | Allow pods → `*.azurecr.io`, block `*.torproject.org` |

**Application Rule Example (allow AKS required FQDNs):**
```bash
az network firewall application-rule create \
  --firewall-name fw-hub-prod \
  --resource-group rg-hub-connectivity \
  --collection-name "aks-required-fqdns" \
  --priority 200 \
  --action Allow \
  --name "allow-aks-registries" \
  --protocols https=443 \
  --source-addresses "10.1.0.0/16" \           # AKS subnet
  --target-fqdns \
    "mcr.microsoft.com" \
    "*.data.mcr.microsoft.com" \
    "management.azure.com" \
    "login.microsoftonline.com" \
    "packages.microsoft.com" \
    "acs-mirror.azureedge.net"
```

**Network Rule Example (allow private endpoint traffic):**
```bash
az network firewall network-rule create \
  --firewall-name fw-hub-prod \
  --resource-group rg-hub-connectivity \
  --collection-name "private-endpoint-rules" \
  --priority 100 \
  --action Allow \
  --name "allow-aks-to-cosmos-pe" \
  --protocols TCP \
  --source-addresses "10.1.0.0/16" \
  --destination-addresses "10.2.0.23" \       # CosmosDB private endpoint IP
  --destination-ports "443"
```

---

## Q2. How does forced tunnelling work and when do you need it?

**Scenario:** Your compliance team requires ALL traffic from Azure VMs — including Azure management traffic — to go through your on-premises security appliance for inspection before reaching the internet.

**Normal routing (without forced tunnelling):**
```
Azure VM
  ├── Traffic to Azure services → Azure backbone
  └── Internet traffic → Azure Firewall → Internet
```

**Forced tunnelling:**
```
Azure VM
  ├── ALL traffic (including Azure management) ──► ExpressRoute/VPN ──► On-prem firewall
  └── On-prem firewall decides what exits to internet
```

**How to configure:**

```
Prerequisites:
  1. VPN Gateway or ExpressRoute Gateway in Hub
  2. AzureFirewallManagementSubnet created (/26 minimum) with its own public IP
     → This subnet bypasses UDR — Firewall management traffic is NOT tunnelled
     → Without this, forced tunnelling breaks Firewall's own management plane

  3. On-prem BGP advertises 0.0.0.0/0 (default route) to Azure
     → Azure learns this route and sends all traffic on-prem
```

```bash
# Create management subnet for Firewall (required for forced tunnel mode)
az network vnet subnet create \
  --name AzureFirewallManagementSubnet \
  --vnet-name vnet-hub-canadacentral \
  --resource-group rg-hub-connectivity \
  --address-prefix 10.0.0.64/26

# Create public IP for Firewall management
az network public-ip create \
  --name pip-fw-management \
  --resource-group rg-hub-connectivity \
  --sku Standard --allocation-method Static

# Enable forced tunnelling on Firewall
az network firewall create \
  --name fw-hub-prod \
  --resource-group rg-hub-connectivity \
  --vnet-name vnet-hub-canadacentral \
  --public-ip "pip-fw-prod" \
  --management-public-ip "pip-fw-management" \   # Enables forced tunnel mode
  --sku AZFW_VNet \
  --tier Premium
```

> ⚠️ **Gotcha:** In forced tunnel mode, `AzureFirewallManagementSubnet` must have a route to the internet (via its own public IP — Firewall management bypasses the tunnelling). If you accidentally apply a UDR to this subnet, the Firewall goes offline.

---

## Q3. How do you use UDRs to control routing between multiple services in the same VNet?

**Scenario:** In a single spoke VNet, you have a web subnet (10.1.1.0/24), an app subnet (10.1.2.0/24), and a DB subnet (10.1.3.0/24). Traffic between tiers must go through a Network Virtual Appliance (NVA) at `10.1.0.4` for Layer 7 inspection.

**Without UDRs:** Azure routes between subnets in the same VNet directly — NVA is bypassed.

**With UDRs:** Force inter-subnet traffic through NVA.

```
Route table on Web subnet:
  10.1.2.0/24 (app tier) → VirtualAppliance 10.1.0.4
  10.1.3.0/24 (db tier)  → VirtualAppliance 10.1.0.4

Route table on App subnet:
  10.1.1.0/24 (web tier) → VirtualAppliance 10.1.0.4
  10.1.3.0/24 (db tier)  → VirtualAppliance 10.1.0.4

Route table on DB subnet:
  10.1.1.0/24 (web tier) → VirtualAppliance 10.1.0.4
  10.1.2.0/24 (app tier) → VirtualAppliance 10.1.0.4
```

```bash
# Create route table for web subnet
az network route-table create \
  --name rt-web-subnet \
  --resource-group rg-app \
  --disable-bgp-route-propagation true

# Route to app tier via NVA
az network route-table route create \
  --route-table-name rt-web-subnet \
  --resource-group rg-app \
  --name "to-app-tier-via-nva" \
  --address-prefix 10.1.2.0/24 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address 10.1.0.4

# Route to db tier via NVA
az network route-table route create \
  --route-table-name rt-web-subnet \
  --resource-group rg-app \
  --name "to-db-tier-via-nva" \
  --address-prefix 10.1.3.0/24 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address 10.1.0.4

# Apply to web subnet
az network vnet subnet update \
  --name snet-web \
  --vnet-name vnet-app \
  --resource-group rg-app \
  --route-table rt-web-subnet
```

> 🔑 **NVA requirement:** The NVA VM must have **IP Forwarding enabled** on its NIC — otherwise it drops packets not addressed to its own IP.

```bash
az network nic update \
  --name nva-nic \
  --resource-group rg-app \
  --ip-forwarding true
```

---

## Q4. How do you diagnose a routing problem — traffic taking the wrong path?

**Scenario:** An AKS pod is supposed to go through the Firewall to reach the internet, but you suspect it's going directly. How do you prove it?

**Step 1 — Check effective routes on the pod's host node NIC:**
```bash
# Find the node the pod is on
kubectl get pod my-pod -o wide
# Note the NODE name (e.g., aks-nodepool1-12345-vmss000001)

# Get NIC name for that node
az vmss nic list \
  --resource-group MC_rg-spoke-aks_aks-prod_canadacentral \
  --vmss-name aks-nodepool1-12345-vmss \
  --query "[].name" -o tsv

# Show effective routes
az network nic show-effective-route-table \
  --name aks-node-nic-name \
  --resource-group MC_rg-spoke-aks_aks-prod_canadacentral \
  --output table
```

**What to look for in the output:**
```
Source    State   AddressPrefix   NextHopType        NextHopIpAddress
────────  ──────  ──────────────  ─────────────────  ────────────────
User      Active  0.0.0.0/0       VirtualAppliance   10.0.0.4   ← Correct (Firewall)
Default   Active  0.0.0.0/0       Internet                      ← Wrong (bypassing)
```
If you see `NextHopType: Internet` for the default route, the UDR is either not attached to the subnet or was overridden.

**Step 2 — Verify UDR is applied to the subnet:**
```bash
az network vnet subnet show \
  --name snet-aks-nodepool1 \
  --vnet-name vnet-spoke-aks \
  --resource-group rg-spoke-aks \
  --query "routeTable.id"
# Should return a non-null route table ID
```

**Step 3 — Check if BGP route propagation overrode UDR:**
```bash
az network route-table show \
  --name rt-spoke-aks \
  --resource-group rg-spoke-aks \
  --query "disableBgpRoutePropagation"
# Should be: true
# If false: BGP routes from VPN/ER may override the 0.0.0.0/0 UDR
```

---

## Q5. How do you enable Azure Firewall Premium's IDPS (Intrusion Detection & Prevention)?

**Scenario:** Your security team requires active blocking of known attack signatures — SQL injection, CVE exploits, C2 callbacks — on traffic leaving your AKS cluster to the internet.

**IDPS modes:**
- `Off` — No intrusion detection
- `Alert` — Log detected signatures but don't block (good for initial tuning)
- `Alert and Deny` — Log AND block matching traffic (production)

```bash
# Create Firewall Policy with IDPS enabled
az network firewall policy create \
  --name fwpolicy-hub-premium \
  --resource-group rg-hub-connectivity \
  --location canadacentral \
  --sku Premium \
  --idps-mode "Alert"         # Start with Alert, switch to AlertAndDeny after tuning

# Add signature overrides (e.g., allow a specific signature that's a false positive)
az network firewall policy intrusion-detection add \
  --policy-name fwpolicy-hub-premium \
  --resource-group rg-hub-connectivity \
  --mode Off \                                 # Override this specific signature to Off
  --signature-id 2008983                       # EICAR test signature

# Enable TLS Inspection (required for IDPS to inspect HTTPS traffic)
az network firewall policy create \
  --name fwpolicy-hub-premium \
  --resource-group rg-hub-connectivity \
  --sku Premium \
  --certificate-authority-name "fw-ca-cert" \
  --key-vault-secret-id "https://kv-prod.vault.azure.net/secrets/fw-ca-cert"
```

**IDPS alert query (Log Analytics):**
```kusto
AzureDiagnostics
| where Category == "AzureFirewallIDPSSignature"
| project TimeGenerated, msg_s, action_s, SignatureId_d
| order by TimeGenerated desc
```

> 💡 **Deep dive hint:** Azure Firewall Premium TLS inspection — how to deploy a CA certificate to Firewall and distribute the CA to all endpoints so HTTPS inspection works without browser certificate warnings.
