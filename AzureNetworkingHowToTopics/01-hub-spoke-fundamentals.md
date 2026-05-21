# Hub-Spoke Networking — Fundamentals
### 🟢 Beginner → 🟡 Intermediate

---

## Q1. What is Hub-Spoke topology and why does every enterprise use it?

**Scenario:** You're designing the Azure network for a bank with 5 application teams. Each team needs isolation, but all teams share an ExpressRoute to their data centre and the same DNS servers.

**The Shape:**
```
                        ┌─────────────────────────────┐
                        │         HUB VNet             │
                        │  ┌──────────┐ ┌───────────┐ │
                        │  │ Firewall │ │  Bastion  │ │
                        │  └──────────┘ └───────────┘ │
                        │  ┌──────────┐ ┌───────────┐ │
                        │  │ VPN/ER   │ │ DNS Rslvr │ │
                        │  │ Gateway  │ │           │ │
                        │  └──────────┘ └───────────┘ │
                        └─────┬────────────┬───────────┘
                              │ Peering    │ Peering
               ┌──────────────┘            └──────────────┐
               │                                          │
    ┌──────────▼──────────┐              ┌────────────────▼────┐
    │  Spoke 1: AKS Team  │              │ Spoke 2: Data Team  │
    │  10.1.0.0/16        │              │ 10.2.0.0/16         │
    └─────────────────────┘              └─────────────────────┘
               │ Peering                             │ Peering
    ┌──────────▼──────────┐              ┌───────────▼─────────┐
    │  Spoke 3: Finance   │              │ Spoke 4: DevTest     │
    │  10.3.0.0/16        │              │ 10.4.0.0/16         │
    └─────────────────────┘              └─────────────────────┘
```

**Why Hub-Spoke?**
- ✅ **Centralised security** — One **Azure Firewall** in the Hub inspects *all* traffic instead of each spoke managing its own
- ✅ **Centralised connectivity** — One **ExpressRoute/VPN Gateway** serves all spokes. No gateway per spoke = major cost saving
- ✅ **Team isolation** — Spoke 1 (AKS team) cannot reach Spoke 2 (Data team) by default — peering is not transitive
- ✅ **Single DNS** — **Azure DNS Private Resolver** in the Hub resolves all private DNS zones for every spoke
- ✅ **Scalability** — Add a new team = create a new spoke VNet and peer it to the Hub. No redesign required

**Core Rules to Remember:**
- Hub ↔ Spoke peering = bidirectional
- Spoke ↔ Spoke = **NOT connected** (must go through Hub Firewall)
- `AllowForwardedTraffic = true` on both sides of peering for transit to work
- `UseRemoteGateways = true` on spoke side so spokes use the Hub's VPN/ExpressRoute gateway

---

## Q2. How do you create a Hub VNet and the required subnets?

**Scenario:** You're building the Hub VNet for a production landing zone. Which subnets must exist and with what CIDRs?

**Required Subnets in Hub VNet (exact names matter — Azure is strict):**

| Subnet name | Minimum size | Purpose | Notes |
|-------------|-------------|---------|-------|
| `AzureFirewallSubnet` | /26 | Azure Firewall | **Exact name required** |
| `AzureFirewallManagementSubnet` | /26 | Firewall management (forced tunnel) | Required only if forced tunnelling |
| `GatewaySubnet` | /27 | VPN / ExpressRoute Gateway | **Exact name required** |
| `AzureBastionSubnet` | /26 | Azure Bastion | **Exact name required** |
| `snet-dns-resolver` | /28 | DNS Private Resolver inbound endpoint | Custom name OK |
| `snet-management` | /24 | Jump boxes, monitoring agents | Custom name |

**Commands:**
```bash
# Create Hub VNet
az network vnet create \
  --name vnet-hub-canadacentral \
  --resource-group rg-hub-connectivity \
  --address-prefix 10.0.0.0/16 \
  --location canadacentral

# Create required subnets
az network vnet subnet create \
  --name AzureFirewallSubnet \
  --vnet-name vnet-hub-canadacentral \
  --resource-group rg-hub-connectivity \
  --address-prefix 10.0.0.0/26

az network vnet subnet create \
  --name GatewaySubnet \
  --vnet-name vnet-hub-canadacentral \
  --resource-group rg-hub-connectivity \
  --address-prefix 10.0.1.0/27

az network vnet subnet create \
  --name AzureBastionSubnet \
  --vnet-name vnet-hub-canadacentral \
  --resource-group rg-hub-connectivity \
  --address-prefix 10.0.2.0/26
```

> ⚠️ **Gotcha:** If you name the subnet anything other than `AzureFirewallSubnet`, the Firewall deployment will fail with a cryptic error. Azure checks the exact string.

---

## Q3. How do you create VNet Peering between Hub and a Spoke?

**Scenario:** Your AKS team's spoke VNet is ready. You need to connect it to the Hub so it can use the central Firewall and ExpressRoute gateway.

**Two peering links are required — one from each side:**

```
Hub VNet ──[Peering: hub-to-spoke1]──► Spoke1 VNet
Hub VNet ◄──[Peering: spoke1-to-hub]── Spoke1 VNet
```

**Commands:**
```bash
HUB_ID=$(az network vnet show \
  --name vnet-hub-canadacentral \
  --resource-group rg-hub-connectivity \
  --query id -o tsv)

SPOKE_ID=$(az network vnet show \
  --name vnet-spoke-aks \
  --resource-group rg-spoke-aks \
  --query id -o tsv)

# Hub → Spoke (grant gateway transit to spoke)
az network vnet peering create \
  --name hub-to-spoke-aks \
  --vnet-name vnet-hub-canadacentral \
  --resource-group rg-hub-connectivity \
  --remote-vnet $SPOKE_ID \
  --allow-vnet-access true \
  --allow-forwarded-traffic true \
  --allow-gateway-transit true      # Hub shares its gateway with spokes

# Spoke → Hub (use hub's gateway and allow forwarded traffic)
az network vnet peering create \
  --name spoke-aks-to-hub \
  --vnet-name vnet-spoke-aks \
  --resource-group rg-spoke-aks \
  --remote-vnet $HUB_ID \
  --allow-vnet-access true \
  --allow-forwarded-traffic true \
  --use-remote-gateways true        # Use Hub's VPN/ER gateway
```

**Key flags explained:**
- `--allow-gateway-transit` on **Hub side** → tells Hub "share my gateway with peers"
- `--use-remote-gateways` on **Spoke side** → tells Spoke "use Hub's gateway, not your own"
- `--allow-forwarded-traffic` on **both sides** → allows traffic that originated *outside* this VNet to pass through (critical for Firewall transit)

> ⚠️ **Gotcha:** `use-remote-gateways` fails if the Hub VNet has no gateway deployed yet. Deploy the VPN/ER Gateway *before* creating spoke peering links.

---

## Q4. How do you force all spoke traffic through the Hub Firewall?

**Scenario:** Spoke 1 (AKS team) is peered to the Hub, but their pods are going directly to the internet, bypassing the Azure Firewall. You need to inspect all outbound traffic.

**What's needed: User Defined Route (UDR)**

```
Without UDR:                    With UDR:
Spoke Pod ──► Internet          Spoke Pod ──► Firewall ──► Internet
             (bypass)                        (inspected)
```

**Steps:**
1. Get the Azure Firewall's **private IP** (e.g., `10.0.0.4`)
2. Create a Route Table with a default route pointing to the Firewall
3. Associate the Route Table to the spoke's subnets

```bash
# Step 1: Get Firewall private IP
FW_IP=$(az network firewall show \
  --name fw-hub-prod \
  --resource-group rg-hub-connectivity \
  --query "ipConfigurations[0].privateIPAddress" -o tsv)

# Step 2: Create route table
az network route-table create \
  --name rt-spoke-aks \
  --resource-group rg-spoke-aks \
  --location canadacentral \
  --disable-bgp-route-propagation true   # Prevent BGP routes overriding UDR

# Step 3: Add default route → Firewall
az network route-table route create \
  --name route-default-to-firewall \
  --route-table-name rt-spoke-aks \
  --resource-group rg-spoke-aks \
  --address-prefix 0.0.0.0/0 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address $FW_IP

# Step 4: Associate to subnet
az network vnet subnet update \
  --name snet-aks-system \
  --vnet-name vnet-spoke-aks \
  --resource-group rg-spoke-aks \
  --route-table rt-spoke-aks
```

> ⚠️ **Gotcha:** `--disable-bgp-route-propagation true` is critical. Without it, BGP routes from ExpressRoute override your UDR and traffic bypasses the Firewall again.

---

## Q5. How do you enable spoke-to-spoke communication through the Hub Firewall?

**Scenario:** The AKS team (Spoke 1) needs to call the Data API (Spoke 2) on port 8080. By default they're isolated. How do you allow just this?

**Traffic path:**
```
Spoke1 (AKS) ──► Hub Firewall ──► Spoke2 (Data API)
10.1.0.0/24                        10.2.0.5:8080
```

**Three things needed:**

**1. UDR on Spoke 1 for Spoke 2's CIDR → Firewall:**
```bash
az network route-table route create \
  --name route-to-spoke2 \
  --route-table-name rt-spoke-aks \
  --resource-group rg-spoke-aks \
  --address-prefix 10.2.0.0/16 \    # Spoke 2 CIDR
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address $FW_IP
```

**2. UDR on Spoke 2 for return traffic → Firewall:**
```bash
az network route-table route create \
  --name route-return-to-spoke1 \
  --route-table-name rt-spoke-data \
  --resource-group rg-spoke-data \
  --address-prefix 10.1.0.0/16 \    # Spoke 1 CIDR
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address $FW_IP
```

**3. Azure Firewall Network Rule: allow Spoke1 → Spoke2 on port 8080:**
```bash
az network firewall network-rule create \
  --firewall-name fw-hub-prod \
  --resource-group rg-hub-connectivity \
  --collection-name "spoke-to-spoke-rules" \
  --priority 200 \
  --action Allow \
  --name "allow-aks-to-data-api" \
  --protocols TCP \
  --source-addresses 10.1.0.0/24 \
  --destination-addresses 10.2.0.5 \
  --destination-ports 8080
```

> ⚠️ **Gotcha:** Forgetting the return-traffic UDR on Spoke 2 means TCP sessions hang — packets go out from Spoke 1, the server responds, but the response takes a different path back and gets asymmetrically routed.

---

## Q6. How do you verify a Hub-Spoke topology is working correctly?

**Scenario:** You've built the topology. How do you confirm traffic is actually flowing through the Firewall and not bypassing it?

**Verification checklist:**

**1. Check peering state:**
```bash
az network vnet peering show \
  --name hub-to-spoke-aks \
  --vnet-name vnet-hub-canadacentral \
  --resource-group rg-hub-connectivity \
  --query "peeringState"
# Expected: "Connected" — any other value = something is wrong
```

**2. Check effective routes on a spoke VM/NIC:**
```bash
az network nic show-effective-route-table \
  --name nic-jumpbox-spoke1 \
  --resource-group rg-spoke-aks \
  --output table
# Look for: 0.0.0.0/0 via VirtualAppliance (10.0.0.4)
# If: 0.0.0.0/0 via Internet — UDR is not applied or was overridden
```

**3. Trace the actual path with Network Watcher:**
```bash
az network watcher test-ip-flow \
  --vm jumpbox-spoke1 \
  --direction Outbound \
  --protocol TCP \
  --local 10.1.0.5:12345 \
  --remote 8.8.8.8:443 \
  --resource-group rg-spoke-aks
# Expected: "Allow" via the Firewall NIC route
```

**4. Confirm Firewall is logging the traffic:**
```kusto
// Log Analytics KQL
AzureDiagnostics
| where Category == "AzureFirewallNetworkRule"
| where msg_s contains "10.1.0.5"
| project TimeGenerated, msg_s, action_s
| order by TimeGenerated desc
```

---

## Q7. What is the CIDR planning strategy for a Hub-Spoke topology?

**Scenario:** You're planning IP ranges for a new landing zone. How do you avoid overlap and leave room to grow?

**Recommended CIDR plan:**

```
Management Group: Enterprise
│
├── Connectivity Subscription (Hub)
│     Hub VNet: 10.0.0.0/16
│       AzureFirewallSubnet:           10.0.0.0/26
│       GatewaySubnet:                 10.0.1.0/27
│       AzureBastionSubnet:            10.0.2.0/26
│       snet-dns-resolver-inbound:     10.0.3.0/28
│       snet-management:               10.0.4.0/24
│
├── Spoke: Production (each team gets a /16, subnets carved within)
│     AKS Spoke:    10.1.0.0/16   (AKS needs large subnets — Azure CNI = 1 IP/pod)
│     Data Spoke:   10.2.0.0/16
│     Finance:      10.3.0.0/16
│     DevTest:      10.4.0.0/16
│
└── On-premises:   192.168.0.0/16  (must not overlap with any Azure range)
```

**Rules:**
- ✅ No two VNets share the same CIDR — peered VNets with overlapping ranges are **rejected**
- ✅ Azure reserves `.0`, `.1`, `.2`, `.3`, `.255` in every subnet — a /24 gives you 251 usable IPs
- ✅ AKS with Azure CNI = 1 IP per pod × pods per node × nodes — use /20 or larger for AKS subnets
- ✅ Leave gaps in your plan (10.5–10.9 reserved for future spokes)

> 💡 **Deep dive hint:** Azure Virtual Network Manager (AVNM) — Microsoft's new centralised tool to manage peering, routing, and security admin rules across dozens of VNets at scale without touching each one individually.
