# Azure Networking — Interview Questions

> Covers: VNet, NSG, Private Endpoints, Hub-Spoke, ExpressRoute, VPN, DNS, Azure Firewall, Front Door.
> Levels: Beginner → Intermediate → Advanced

---

## BEGINNER

**Q1. What is a Virtual Network (VNet) and how is it structured?**

**Answer:**

A VNet is an isolated, logically defined network in Azure — your private network in the cloud. Resources in the same VNet can communicate by default; resources in different VNets cannot (unless peered).

**Structure:**
```
VNet: 10.0.0.0/16 (65,534 usable IPs)
  ├── Subnet: snet-frontend     10.0.1.0/24 (251 usable IPs)
  ├── Subnet: snet-backend      10.0.2.0/24
  ├── Subnet: snet-data         10.0.3.0/24
  ├── Subnet: snet-aks          10.0.4.0/22  (1022 IPs — AKS needs more)
  └── Subnet: AzureFirewallSubnet 10.0.0.0/26  (reserved name, /26 minimum)
```

**Reserved IPs per subnet:** Azure reserves 5 IPs per subnet (.0 network, .1 gateway, .2-.3 Azure services, .255 broadcast). A /24 gives 256 - 5 = 251 usable IPs.

**AKS sizing note:** AKS Azure CNI allocates one IP per pod. For 100 nodes × 30 pods each = 3,000 IPs needed → use /20 or larger for AKS subnet.

---

**Q2. What is an NSG (Network Security Group) and how does it work?**

**Answer:**

An NSG is a virtual firewall at Layer 4 (TCP/UDP port level). It contains inbound and outbound security rules.

**How rules are evaluated:**
- Rules have a **priority** (100–4096) — lower number = higher priority
- Rules are evaluated **in order** until a match is found
- If no rule matches → **default rules** apply (allow same-VNet, allow Azure LB, deny all else)

**Common example:**
```
Inbound rules:
  Priority 100: Allow  TCP port 443  from Internet        → ALLOW HTTPS
  Priority 200: Allow  TCP port 22   from 10.0.0.0/8      → ALLOW SSH from internal only
  Priority 300: Allow  any           from VirtualNetwork   → ALLOW VNet-to-VNet
  Priority 400: Allow  any           from AzureLoadBalancer → ALLOW health probes
  Priority 65500: Deny  any          from any              → DENY ALL (default)
```

**NSG applies to:**
- Subnet: all resources in the subnet inherit the rules
- NIC (network interface): applies to a single VM

**Troubleshooting NSG issues:**
```bash
# Check effective security rules on a VM NIC
az network nic show-effective-nsg \
  --name myVMNic \
  --resource-group rg-prod

# Enable NSG Flow Logs to see actual traffic
az network watcher flow-log create \
  --location canadacentral \
  --resource-group rg-networking \
  --nsg myNSG \
  --storage-account mystorageaccount \
  --workspace myLogAnalyticsWorkspace
```

---

**Q3. What is the difference between VNet Peering and VPN Gateway?**

**Answer:**

| Feature | VNet Peering | VPN Gateway |
|---------|-------------|-------------|
| Traffic path | Azure backbone (no internet) | Encrypted tunnel (IPsec) |
| Latency | Very low (sub-ms) | Higher (encryption overhead) |
| Bandwidth | No limit | Max 10Gbps (VpnGw5) |
| Cross-region | Yes (Global Peering) | Yes |
| On-premises connectivity | No | Yes (Site-to-Site) |
| Cost | Per GB transferred | Per gateway hour + per GB |
| Setup time | Minutes | 20-45 minutes (gateway provisioning) |
| Transitivity | Not transitive | Transitive with BGP |

**VNet Peering is not transitive:**
```
Hub ←──peering──→ Spoke A
Hub ←──peering──→ Spoke B

Spoke A CANNOT reach Spoke B through the Hub by default.
To enable: use Azure Firewall in Hub as transit point (allowForwardedTraffic on peering).
```

**When to use each:**
- VNet Peering: Azure-to-Azure connectivity, same or different subscriptions
- VPN Gateway: on-premises to Azure connectivity, or when encryption across peering is required

---

## INTERMEDIATE

**Q4. Explain the Hub-Spoke topology. Why is it the enterprise standard?**

**Answer:**

**Structure:**

```
Hub VNet (10.0.0.0/16) — Canada Central
  ├── Azure Firewall (10.0.0.4) — centralized traffic inspection
  ├── Azure Bastion — secure RDP/SSH without public IPs
  ├── VPN Gateway — on-premises connectivity
  ├── ExpressRoute Gateway — dedicated circuit connectivity
  ├── Azure DNS Private Resolver — centralized DNS
  └── Management Subnet (jumpbox, Ansible, monitoring)

Spoke 1 VNet (10.1.0.0/16) — AKS Workloads
  └── Peered to Hub (allow forwarded traffic, use remote gateway)

Spoke 2 VNet (10.2.0.0/16) — Data Platform
  └── Peered to Hub

Spoke 3 VNet (10.3.0.0/16) — Dev/Test
  └── Peered to Hub

On-premises (192.168.0.0/16)
  └── Connected via ExpressRoute → Hub Gateway → Hub Firewall → Spokes
```

**Why it's the enterprise standard:**

| Benefit | Explanation |
|---------|-------------|
| Centralized security | Azure Firewall in Hub inspects ALL traffic (east-west + north-south) |
| Network segmentation | Each spoke is isolated — Dev can't reach Prod by default |
| Centralized connectivity | Single ExpressRoute/VPN in Hub serves all spokes |
| Cost efficient | One expensive Firewall/Gateway serves all workloads |
| Governance | NSGs + Azure Policy applied at spoke level, Firewall at Hub |
| Scalability | Add new spokes without redesigning networking |

**Common Hub-Spoke mistake:** Spoke-to-Spoke communication without going through the Hub Firewall. Default: it goes through the Firewall (if configured correctly). Missing `allowForwardedTraffic = true` on peering breaks this.

---

**Q5. How does Private DNS work in Azure and what is the most common misconfiguration?**

**Answer:**

**How it works:**

When you create a Private Endpoint for a PaaS service (e.g., Azure SQL), Azure creates:
1. A private IP (e.g., 10.1.0.5) for the SQL endpoint in your subnet
2. A DNS record in a Private DNS Zone (e.g., `privatelink.database.windows.net`)

Without the Private DNS Zone linked to your VNet:
```
nslookup myserver.database.windows.net  → returns 52.x.x.x (PUBLIC IP)
```

With Private DNS Zone linked to your VNet:
```
nslookup myserver.database.windows.net  → returns 10.1.0.5 (PRIVATE IP) ✓
```

**Most common misconfiguration: Private DNS Zone NOT linked to the VNet**

```bash
# Step 1: Create the Private DNS Zone
az network private-dns zone create \
  --resource-group rg-networking \
  --name "privatelink.database.windows.net"

# Step 2: Create a zone link — THIS IS THE MOST MISSED STEP
az network private-dns link vnet create \
  --resource-group rg-networking \
  --zone-name "privatelink.database.windows.net" \
  --name "link-to-hub" \
  --virtual-network hub-vnet \
  --registration-enabled false

# Step 3: Verify DNS resolution from a VM inside the VNet
az vm run-command invoke \
  --resource-group rg-vms \
  --name test-vm \
  --command-id RunShellScript \
  --scripts "nslookup myserver.database.windows.net"
# Expected: 10.x.x.x private IP
# If returning public IP: DNS Zone not linked
```

**Hub-Spoke with Private DNS:**
- Create Private DNS Zones in the Hub VNet (centralized)
- Link to Hub VNet (registration-enabled: false)
- Spokes use Hub DNS via peering (transitively)
- Azure DNS Private Resolver in Hub forwards on-prem DNS queries → Azure

---

**Q6. What is the difference between Azure VPN Gateway and ExpressRoute? When would you use each?**

**Answer:**

| Feature | VPN Gateway | ExpressRoute |
|---------|------------|--------------|
| **Connection type** | Encrypted IPsec/IKE over internet | Dedicated private circuit via carrier |
| **Bandwidth** | Up to 10 Gbps (VpnGw5) | 50 Mbps to 100 Gbps |
| **Latency** | Variable (internet) | Consistent, low (private circuit) |
| **SLA** | 99.95% (zone-redundant 99.99%) | 99.95% (ExpressRoute circuit) |
| **Setup time** | Hours (gateway deployment) | Weeks to months (carrier provisioning) |
| **Cost** | ~$100–$500/month + bandwidth | $500–$5,000+/month + bandwidth |
| **Encryption** | Yes (IPsec) | No (private circuit — physical security) |
| **Failover** | VPN over internet as backup to ER | Recommended |
| **Use for** | Dev/test, small offices, secondary link | Core banking, large data transfers, <5ms latency |

**ExpressRoute bandwidth tiers:**
- 50 Mbps, 100 Mbps, 200 Mbps, 500 Mbps (metered billing)
- 1 Gbps, 2 Gbps, 5 Gbps, 10 Gbps, 100 Gbps (unlimited billing available)

**ExpressRoute circuit setup:**
```
On-premises router
  ↕ [BGP peering]
Carrier / Partner (Bell Canada, Rogers, etc.)
  ↕ [Physical cross-connect in carrier facility]
Azure ExpressRoute Edge Router (Toronto — Canada Central)
  ↕ [BGP peering: AS 12076]
Azure VNet Gateway (ExpressRoute Gateway)
  ↕ [UDR route propagation]
Azure VNets (Hub-Spoke)
```

**Coexistence: VPN + ExpressRoute**
- ExpressRoute as primary link
- VPN as failover (if ExpressRoute circuit fails)
- Achieved via BGP route preference (ExpressRoute has higher MED/local preference)

---

## ADVANCED

**Q7. You have a Hub-Spoke topology with 15 spokes. An app in Spoke 3 cannot reach a database Private Endpoint in Spoke 7. Walk through the troubleshooting steps.**

**Answer:**

**Network path:**
```
App (Spoke 3) → Hub Firewall → Spoke 7 → Private Endpoint → Azure SQL
```

**Troubleshooting order:**

```
Step 1: Confirm VNet peering configuration
  az network vnet peering list --vnet-name spoke3-vnet -g rg-networking
  Check:
    - peeringState: Connected (not Disconnected or Initiated)
    - allowForwardedTraffic: true (required for Hub Firewall transit)
    - allowGatewayTransit: false on spoke (true on hub side)
    - useRemoteGateways: depends on setup

Step 2: Check effective routes on the app VM/pod's NIC
  az network nic show-effective-route-table --name app-nic -g rg-app
  Expected: route to Spoke 7's CIDR via Hub Firewall (nextHopIPAddress: hub-firewall-IP)
  If missing: UDR on Spoke 3 subnet not configured to route to Hub Firewall

Step 3: Check Azure Firewall rules
  # Does Firewall have a rule allowing Spoke 3 → Spoke 7?
  # Check: Network Rules collection (Layer 4)
  # Check: Application Rules collection (if FQDN-based)
  # DENY hits are logged: query AzureDiagnostics in Log Analytics
  AzureDiagnostics
  | where Category == "AzureFirewallNetworkRule"
  | where msg_s contains "Spoke3-IP"
  | project TimeGenerated, msg_s, action_s
  | order by TimeGenerated desc

Step 4: Check NSG on Private Endpoint subnet in Spoke 7
  # NSG on snet-private-endpoints in Spoke 7 must allow inbound from Hub Firewall IP
  # Private Endpoint NSG: network_policies must be ENABLED on subnet
  az network vnet subnet show \
    --name snet-private-endpoints \
    --vnet-name spoke7-vnet \
    --resource-group rg-spoke7 \
    --query "privateEndpointNetworkPolicies"
  # If "Disabled": NSG rules on PE subnet are not enforced

Step 5: DNS resolution check
  # From app VM in Spoke 3:
  nslookup myserver.database.windows.net
  # Expected: returns Spoke 7 private IP (10.7.x.x)
  # If returns public IP: DNS Zone not linked to Spoke 3's VNet
  # Fix: link Private DNS Zone to Hub VNet (Spoke 3 uses Hub DNS via peering)

Step 6: IP Flow Verify (Network Watcher)
  az network watcher test-ip-flow \
    --vm app-vm \
    --direction Inbound \
    --protocol TCP \
    --local 10.7.0.5:1433 \
    --remote 10.3.0.10:54321
  # Returns: Allow or Deny, and which rule caused it

Step 7: Connection Monitor
  az network watcher connection-monitor create \
    --name app-to-sqlpe \
    --resource-group rg-monitoring \
    --source-resource app-vm-id \
    --dest-address 10.7.0.5 \
    --dest-port 1433
  # Continuous monitoring with latency, packet loss, and hop visualization
```

**Root cause probability ranking:**
1. Firewall rule missing (Spoke3 → Spoke7 not allowed) — 40%
2. UDR missing (Spoke3 not routing via Hub Firewall) — 25%
3. DNS Zone not linked to correct VNet — 20%
4. NSG blocking on PE subnet — 10%
5. VNet peering misconfiguration — 5%

---

**Q8. How does BGP work with ExpressRoute and what are common BGP issues in Azure?**

**Answer:**

**BGP basics in ExpressRoute context:**

```
BGP (Border Gateway Protocol) — path vector routing protocol
Used to exchange routes between:
  - Azure (AS 12076) ↔ ExpressRoute Partner (AS varies) ↔ On-premises (customer AS)

Route advertisement:
  Azure → Customer: Azure advertises its VNet prefixes over BGP
  Customer → Azure: Customer advertises on-prem prefixes over BGP

ExpressRoute BGP sessions:
  Primary path:   On-prem router ↔ MSEE (Microsoft Edge Router) primary
  Secondary path: On-prem router ↔ MSEE secondary (HA failover)
  Both use eBGP with MD5 authentication
```

**Common BGP issues and fixes:**

```
Issue 1: BGP session down
  Symptom: ExpressRoute circuit provisioned but no connectivity
  Check: Get-AzExpressRouteCircuitPeeringConfig -CircuitName myCircuit
  Cause: MD5 key mismatch, wrong ASN, wrong VLAN ID, firewall blocking TCP 179
  Fix: Verify MD5 key matches on both sides, check AS numbers

Issue 2: Routes not propagating to Azure VNets
  Symptom: BGP session up, can see routes in circuit route table, but VMs still can't reach on-prem
  Check: Get-AzExpressRouteCircuitRouteTable -CircuitName myCircuit -PeeringType AzurePrivatePeering
  Cause: 
    a. VNet Gateway route propagation disabled (BGP route propagation = Disabled on subnet)
    b. UDR overriding BGP routes on subnet (static route wins over BGP)
    c. Address space overlap: on-prem prefix already exists in Azure VNet CIDR
  Fix:
    a. az network vnet subnet update --disable-private-endpoint-network-policies false \
         → OR re-enable BGP route propagation on route table

Issue 3: Asymmetric routing
  Symptom: TCP sessions reset, some traffic works, some doesn't
  Cause: Inbound traffic via ExpressRoute, outbound via VPN (or vice versa)
  Fix: Set consistent BGP local preference to prefer ExpressRoute:
    On-prem BGP: set higher local-preference for routes learned from ER
    Azure side: route filter to prefer ER over VPN (AS path prepending on VPN)

Issue 4: Route limit exceeded
  Azure ExpressRoute limit: 4,000 routes per BGP session
  On-prem advertising 5,000 specific routes → session crashes
  Fix: Summarize on-prem routes with supernets (aggregate to /20 or /16 prefixes)
       Avoid advertising full BGP table from on-prem
```

---

**Q9. Design the network architecture for a zero-trust, multi-region application on Azure.**

**Answer:**

**Design goals:**
- No implicit trust based on network location
- All traffic encrypted end-to-end
- Centralized inspection and logging
- Micro-segmentation between services

```
GLOBAL LAYER
  Azure Front Door Premium
    → WAF (OWASP 3.2 + custom rules)
    → Private Link to APIM backends (traffic never on public internet)
    → Health probes: 30s interval, automatic failover

REGION: CANADA CENTRAL
  Hub VNet (10.0.0.0/16):
    Azure Firewall Premium
      → IDPS (Intrusion Detection and Prevention)
      → TLS inspection (decrypt, inspect, re-encrypt)
      → FQDN-based app rules (allow only known destinations)
    Azure Bastion (no public IP VMs anywhere)
    Private DNS Resolver (inbound + outbound endpoints)
    ExpressRoute Gateway

  Spoke 1 — AKS (10.1.0.0/16):
    Private AKS cluster (API server: no public IP)
    Calico network policy: default-deny-all pods
    Workload Identity: per-service MI, least privilege
    Container image: only from ACR (signed, scanned)
    
  Spoke 2 — Data (10.2.0.0/16):
    Azure SQL MI (VNet-native, no public endpoint)
    CosmosDB (Private Endpoint)
    ADLS Gen2 (Private Endpoint)
    All services: CMK encryption (Key Vault)

INTER-SPOKE TRAFFIC:
  AKS pod → CosmosDB:
    Pod Workload Identity → Entra ID token exchange
    Traffic: Spoke 1 → Hub Firewall → Spoke 2 → PE → CosmosDB
    Firewall rule: allow AKS subnet → CosmosDB PE on 443 only
    NSG: CosmosDB PE subnet allows only from Hub Firewall IP

  mTLS between pods:
    Istio service mesh (optional) — automatic mTLS between all pods
    Certificate rotation: automated via cert-manager + Vault

NORTH-SOUTH CONTROL:
  Inbound: Front Door → APIM (private) → AKS Ingress (internal LB)
  Outbound: AKS → Hub Firewall → Azure Firewall rules → internet (deny by default)
    Allowed outbound: specific FQDNs only (package registries, telemetry endpoints)

OBSERVABILITY (Zero Trust requires full visibility):
  All Firewall logs → Log Analytics
  All NSG flow logs → Log Analytics  
  All Private Endpoint access logs → Log Analytics
  Microsoft Sentinel: SIEM with anomaly detection
  Alert: any traffic denied by Firewall from unexpected source
```

**Zero Trust validation checklist:**
```
✅ No VM has a public IP address
✅ No PaaS service has a public endpoint (all via Private Endpoints)
✅ No standing privileged access (PIM JIT only)
✅ All pod-to-pod traffic uses Workload Identity (no secrets)
✅ All traffic logged (NSG flows + Firewall logs + App Insights)
✅ Default network policy: deny all; allowlist specific flows
✅ TLS 1.2+ enforced everywhere (Azure Policy: deny TLS < 1.2)
✅ Customer-managed encryption keys for all data at rest
```
