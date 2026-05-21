# Hub-Spoke Networking — Advanced Patterns
### 🔴 Advanced

---

## Q1. How do you build a multi-region Hub-Spoke topology with failover?

**Scenario:** You need active-active deployments in Canada Central and Canada East. Both regions must be able to access on-prem and reach each other's workloads in case of a regional failure.

**Topology:**
```
On-Premises DC
  │
  ├──ExpressRoute──► Hub-CAN-CENTRAL (10.0.0.0/16)
  │                      │
  │              ┌───────┴────────┐
  │         Spoke-CC-1       Spoke-CC-2
  │         10.1.0.0/16     10.2.0.0/16
  │
  └──ExpressRoute──► Hub-CAN-EAST (10.100.0.0/16)
                         │
                 ┌───────┴────────┐
            Spoke-CE-1       Spoke-CE-2
            10.101.0.0/16   10.102.0.0/16

Hub-CAN-CENTRAL ◄──── Global VNet Peering ────► Hub-CAN-EAST
```

**Key decisions:**
- 🔑 **Global VNet Peering** between the two Hubs — cross-region traffic stays on the Azure backbone (low latency, no internet)
- 🔑 Each Hub has its **own ExpressRoute Gateway** linked to a separate ExpressRoute circuit — if one region fails, on-prem can still reach the other
- 🔑 **Azure Firewall** in each Hub — traffic from Canada East spokes to Canada Central spokes transits through *both* Firewalls (CE Firewall → Hub peering → CC Firewall)
- 🔑 **Traffic Manager or Front Door** at the DNS layer directs users to the healthy region

**UDR consideration for cross-region spoke traffic:**
```
Spoke-CC-1 pod wants to reach Spoke-CE-1 (10.101.0.0/16)

Route Table on Spoke-CC-1 subnet:
  10.101.0.0/16  →  VirtualAppliance (CC Hub Firewall: 10.0.0.4)
  
CC Hub Firewall rule:
  Allow: 10.1.0.0/16 → 10.101.0.0/16 TCP 8080
  Next hop: CE Hub VNet (via Global Peering)

CE Hub Firewall rule:
  Allow: 10.1.0.0/16 → 10.101.0.0/16 TCP 8080
```

> ⚠️ **Gotcha:** Global VNet Peering does not support `use-remote-gateways`. Cross-region gateway transit requires **Azure Virtual WAN** or a dedicated ExpressRoute circuit per region.

---

## Q2. What is Azure Virtual WAN and when should you replace Hub-Spoke with it?

**Scenario:** You now have 40 spokes across 6 regions. Managing hundreds of peering connections and UDRs manually is becoming unsustainable. Your manager asks if there's a managed alternative.

**Hub-Spoke (manual) vs vWAN (managed):**

```
MANUAL HUB-SPOKE                    AZURE VIRTUAL WAN
─────────────────                   ──────────────────
You manage peering                  Microsoft manages routing
You write UDRs                      Automatic route propagation
You configure BGP                   BGP managed by vWAN Hub
You peer hubs manually              Any-to-any hub connectivity built-in
Max: manageable up to ~30 spokes    Scales to hundreds of spokes
Cost: Firewall + Gateway costs      vWAN Hub + Firewall + Gateway costs
Flexibility: very high              Some customisation limits
```

**When vWAN makes sense:**
- ✅ More than 20–30 spokes across multiple regions
- ✅ Branch offices connecting via SD-WAN (vWAN has SD-WAN partner integrations)
- ✅ Any-to-any connectivity between all spokes and branches needed
- ✅ You want Microsoft to manage the routing plane

**When to stay with manual Hub-Spoke:**
- ✅ Fewer than 20 spokes
- ✅ Highly customised routing or policy requirements
- ✅ Cost sensitivity (vWAN Hub adds cost on top of Firewall/Gateway)

---

## Q3. How do you isolate two spokes so they can never communicate, even through the Firewall?

**Scenario:** You have a spoke for PCI-compliant workloads (payments) and a spoke for general workloads. Compliance requires absolute network isolation — the PCI spoke must never receive traffic from general workloads under any configuration.

**Layers of isolation:**

```
General Spoke (10.3.0.0/16)           PCI Spoke (10.4.0.0/16)
──────────────────────────             ─────────────────────────
NSG: deny from 10.4.0.0/16            NSG: deny from 10.3.0.0/16
UDR: no route to 10.4.0.0/16          UDR: no route to 10.3.0.0/16
Firewall rule: deny all               Firewall rule: deny all
                  ↓                                  ↓
          Hub Firewall — no rule allows 10.3 ↔ 10.4
```

**1. NSG — deny at subnet level:**
```bash
# On PCI spoke subnet NSG — block ALL inbound from general spoke
az network nsg rule create \
  --nsg-name nsg-pci-subnet \
  --resource-group rg-spoke-pci \
  --name "deny-from-general-spoke" \
  --priority 100 \
  --direction Inbound \
  --access Deny \
  --source-address-prefixes 10.3.0.0/16 \
  --destination-address-prefixes "*" \
  --destination-port-ranges "*" \
  --protocol "*"
```

**2. No UDR route from General Spoke to PCI CIDR** — traffic has no path at the routing level

**3. No Firewall rule allowing 10.3 → 10.4** — even if a packet somehow reaches the Firewall, there's no rule allowing it

**4. Azure Policy deny VNet peering between these two spokes** — governance layer
```json
{
  "if": {
    "allOf": [
      { "field": "type", "equals": "Microsoft.Network/virtualNetworks/virtualNetworkPeerings" },
      { "field": "Microsoft.Network/virtualNetworks/virtualNetworkPeerings/remoteVirtualNetwork.id",
        "contains": "vnet-spoke-pci" }
    ]
  },
  "then": { "effect": "Deny" }
}
```

> 💡 **Deep dive hint:** PCI-DSS network segmentation requirements in Azure — how to document CDE (cardholder data environment) boundary for QSA auditors using Azure Network Topology diagrams.

---

## Q4. How do you implement a "quarantine" spoke for newly onboarded workloads?

**Scenario:** Your security team requires all new VMs to pass a vulnerability scan before being allowed to communicate with production. How do you build a quarantine zone in Azure networking?

**Flow:**
```
New VM provisioned
     │
     ▼
Quarantine Spoke (10.9.0.0/16)
  - NSG: allow ONLY outbound to scanner IP on 443
  - NSG: deny all other outbound
  - NSG: allow inbound from scanner on 443 only
  - UDR: no route to production spokes
     │
     ▼ (Scanner runs, passes)
     │
Production Spoke (10.1.0.0/16) — VM migrated here via automation
```

**NSG for quarantine subnet:**
```bash
# Allow scanner inbound
az network nsg rule create \
  --nsg-name nsg-quarantine \
  --resource-group rg-quarantine \
  --name "allow-scanner-inbound" \
  --priority 100 \
  --direction Inbound \
  --source-address-prefixes "10.0.5.10"  \  # Scanner IP
  --destination-port-ranges "443" \
  --access Allow

# Deny all other inbound
az network nsg rule create \
  --nsg-name nsg-quarantine \
  --resource-group rg-quarantine \
  --name "deny-all-other-inbound" \
  --priority 4000 \
  --direction Inbound \
  --access Deny \
  --source-address-prefixes "*" \
  --destination-address-prefixes "*" \
  --destination-port-ranges "*"

# Allow VM to talk to scanner only (outbound)
az network nsg rule create \
  --nsg-name nsg-quarantine \
  --resource-group rg-quarantine \
  --name "allow-outbound-to-scanner" \
  --priority 100 \
  --direction Outbound \
  --destination-address-prefixes "10.0.5.10" \
  --destination-port-ranges "443" \
  --access Allow

# Deny all other outbound
az network nsg rule create \
  --nsg-name nsg-quarantine \
  --resource-group rg-quarantine \
  --name "deny-all-outbound" \
  --priority 4000 \
  --direction Outbound \
  --access Deny \
  --source-address-prefixes "*" \
  --destination-address-prefixes "*" \
  --destination-port-ranges "*"
```

---

## Q5. What happens to spoke-to-spoke traffic latency when routed through the Hub Firewall?

**Scenario:** The Data team complains that their API calls to the AKS team are taking 40ms longer after you introduced Hub Firewall transit. Is this expected?

**Latency breakdown:**
```
WITHOUT Firewall transit:
Spoke1 VM → [Peering ~0.5ms] → Spoke2 VM    Total: ~0.5ms

WITH Firewall transit:
Spoke1 VM → [Peering ~0.5ms] → Firewall → [Peering ~0.5ms] → Spoke2 VM
+ Firewall processing: ~2–5ms (stateful inspection, rule evaluation)
Total: ~3–6ms additional latency
```

**Mitigations:**
- ✅ For **latency-sensitive** east-west traffic: use Firewall **Network Rules** (Layer 4) instead of Application Rules (Layer 7) — Layer 4 adds ~2ms, Layer 7 adds ~5–10ms
- ✅ For **very high throughput** internal traffic (>10Gbps): consider **direct peering** between those two specific spokes bypassing the Firewall (accept the security trade-off; compensate with tight NSGs)
- ✅ **Azure Firewall Premium** with IDPS can add 15–30ms — evaluate whether full TLS inspection is needed for internal traffic vs. external-only
- ✅ Use **Azure Firewall Basic** for non-sensitive spoke-to-spoke and save the Premium SKU for internet-facing traffic

> 💡 **Deep dive hint:** Azure Firewall performance benchmarks — throughput limits per SKU (Basic: 250Mbps, Standard: 30Gbps, Premium: 100Gbps) and how to right-size for your east-west traffic volume.
