# Azure Networking Troubleshooting Playbook
### 🟡 Intermediate → 🔴 Advanced

> Every network problem in Azure falls into one of four categories:  
> **DNS · Routes · NSG/Firewall · Service Configuration**  
> Diagnose in that order and you'll find the answer faster.

---

## Q1. A VM can't reach another VM in the same VNet. Where do you start?

**Scenario:** VM-A (`10.1.0.5`) tries to SSH to VM-B (`10.1.0.6`) — both in the same subnet. Connection times out.

**Systematic approach:**

```
Step 1: Same subnet or different subnet?
  Same subnet → skip route issues, focus on NSG
  Different subnet → check routes first, then NSG

Step 2: Check NSG on destination VM's subnet and NIC
Step 3: Check NSG on source VM's NIC (outbound rules)
Step 4: Use IP Flow Verify tool
Step 5: Check OS-level firewall on destination (iptables / Windows Firewall)
```

**Step 2 — Check effective NSG rules:**
```bash
az network nic show-effective-nsg \
  --name vm-b-nic \
  --resource-group rg-vms \
  --output table
# Look for: any DENY rule matching source 10.1.0.5 or port 22
```

**Step 4 — IP Flow Verify (quickest first check):**
```bash
az network watcher test-ip-flow \
  --vm vm-b \
  --resource-group rg-vms \
  --direction Inbound \
  --protocol TCP \
  --local 10.1.0.6:22 \
  --remote 10.1.0.5:12345
# Output includes:
#   Access: Allow or Deny
#   RuleName: the specific NSG rule that matched
```

**If IP Flow says Allow but connection still fails:**
- The NSG allows it — look at the **OS-level firewall**
- On Linux: `sudo iptables -L -n | grep 22`
- On Windows: `netsh advfirewall firewall show rule name=all | findstr SSH`

---

## Q2. A pod in AKS can't reach an Azure SQL Private Endpoint. Systematic diagnosis.

**Scenario:** AKS pod connects to `sqlserver.database.windows.net:1433` and gets a connection timeout. SQL has a Private Endpoint.

**Checklist — work from the pod outward:**

```
Layer 1: DNS Resolution
Layer 2: Route to Private Endpoint
Layer 3: NSG on Private Endpoint subnet
Layer 4: Azure Firewall rule (if traffic routes through Firewall)
Layer 5: Private Endpoint connection state
Layer 6: SQL Firewall rules
```

**Layer 1 — DNS:**
```bash
# From inside the AKS pod
kubectl exec -it my-pod -- nslookup sqlserver.database.windows.net
# Expected: returns 10.x.x.x (Private Endpoint IP)
# Wrong:    returns 52.x.x.x (public IP → DNS zone not linked to pod's VNet)
```

**Layer 2 — Route:**
```bash
# Check node's effective routes (pod shares node's NIC routing)
az network nic show-effective-route-table \
  --name aks-node-nic \
  --resource-group MC_rg-spoke-aks_aks-prod_canadacentral \
  --output table
# Expected: 10.x.x.x (PE subnet) via VirtualNetwork or VirtualAppliance
```

**Layer 3 — NSG on PE subnet:**
```bash
az network nsg rule list \
  --nsg-name nsg-private-endpoints \
  --resource-group rg-data \
  --output table
# Look for any DENY rule matching AKS pod IP range on port 1433
```

**Layer 4 — Firewall (if traffic goes through Firewall):**
```kusto
// Log Analytics — did Firewall drop the packet?
AzureDiagnostics
| where Category == "AzureFirewallNetworkRule"
| where msg_s contains "1433"
| where action_s == "Deny"
| project TimeGenerated, msg_s, SourceIP_s, DestinationIP_s
```

**Layer 5 — Private Endpoint connection state:**
```bash
az network private-endpoint show \
  --name pe-sqlserver \
  --resource-group rg-spoke-aks \
  --query "privateLinkServiceConnections[0].privateLinkServiceConnectionState"
# Expected: { "status": "Approved", "description": "" }
# If: "Pending" — someone needs to approve the connection
```

**Layer 6 — SQL Firewall:**
```bash
az sql server show \
  --name sqlserver \
  --resource-group rg-data \
  --query "publicNetworkAccess"
# If "Enabled" with no VNet rules: public access might be overriding PE
# Set to "Disabled" for Private Endpoint-only access
```

---

## Q3. ExpressRoute circuit is provisioned but on-premises servers can't reach Azure VMs. How do you diagnose?

**Scenario:** Circuit shows "Provisioned" in the portal, but `ping` from on-premises to Azure VM fails.

**Diagnosis layers:**
```
Layer 1: Circuit provisioning status
Layer 2: BGP peering state  
Layer 3: Route propagation to Azure VNet
Layer 4: Route propagation to on-premises
Layer 5: NSG allowing on-prem CIDR
Layer 6: Azure VM OS firewall
```

**Layer 1 — Circuit status:**
```bash
az network express-route show \
  --name er-circuit-toronto \
  --resource-group rg-hub-connectivity \
  --query "{ServiceProviderProvisioningState:serviceProviderProvisioningState, CircuitProvisioningState:circuitProvisioningState}"
# Both must be "Provisioned" / "Enabled"
# If ServiceProvider = "NotProvisioned": carrier hasn't done their side yet
```

**Layer 2 — BGP peering state:**
```bash
az network express-route peering show \
  --circuit-name er-circuit-toronto \
  --resource-group rg-hub-connectivity \
  --name AzurePrivatePeering \
  --query "state"
# Expected: "Enabled"

# Check BGP peer status (requires PowerShell)
Get-AzExpressRouteCircuitPeeringConfig `
  -ExpressRouteCircuitName "er-circuit-toronto" `
  -ResourceGroupName "rg-hub-connectivity" `
  -Name "AzurePrivatePeering" | Select-Object -ExpandProperty Stats
```

**Layer 3 — Is Azure advertising routes to on-prem?**
```bash
# List routes Azure is sending to on-prem over the circuit
az network express-route list-route-tables \
  --name er-circuit-toronto \
  --resource-group rg-hub-connectivity \
  --peering-name AzurePrivatePeering \
  --path Primary \
  --output table
# Expected: Azure VNet CIDRs (10.0.0.0/16, 10.1.0.0/16) listed here
# If empty: VNet Gateway not linked to circuit, or VNet not peered to Hub
```

**Layer 4 — Is on-prem advertising routes to Azure?**
```bash
# List routes on-prem is sending to Azure
az network express-route list-route-tables-summary \
  --name er-circuit-toronto \
  --resource-group rg-hub-connectivity \
  --peering-name AzurePrivatePeering \
  --path Primary \
  --output table
# Expected: on-prem CIDR (192.168.0.0/16) listed
# If missing: on-prem BGP not advertising, or BGP session down
```

**Layer 5 — NSG on Azure VM subnet:**
```bash
# Does NSG allow inbound from on-prem range?
az network nsg rule list \
  --nsg-name nsg-vm-subnet \
  --resource-group rg-vms \
  --output table | grep -i "192.168\|onprem\|allow"
```

**Most common root causes (in order of frequency):**
1. 🔴 **VNet not linked to ER Gateway** — gateway exists but no Connection resource linking it to the circuit
2. 🟠 **BGP route not propagated** — `DisableBgpRoutePropagation = true` on subnet route table
3. 🟡 **BGP AS number mismatch** — on-prem router using wrong ASN in peering config
4. 🟡 **BGP auth key mismatch** — MD5 key doesn't match between Azure and on-prem config
5. 🟢 **NSG blocking on-prem CIDR** — missing allow rule for `192.168.0.0/16` inbound

---

## Q4. DNS resolution is working but HTTPS connection still fails. How do you trace it?

**Scenario:** `nslookup` returns the correct private IP for a storage account endpoint. But `curl https://myaccount.blob.core.windows.net` from a VM times out.

**Separation: DNS resolved correctly, so the problem is connectivity, not DNS.**

**Step 1 — Test raw TCP connectivity (bypass HTTPS):**
```bash
# From the VM:
# Test if port 443 is reachable at the private endpoint IP
Test-NetConnection -ComputerName "10.2.0.15" -Port 443   # PowerShell
# or
nc -zv 10.2.0.15 443   # Linux
# If TCP fails: routing or NSG problem
# If TCP succeeds but HTTPS fails: TLS/certificate issue
```

**Step 2 — Check TLS certificate:**
```bash
# Is the storage account using a valid certificate that matches the hostname?
openssl s_client -connect 10.2.0.15:443 -servername myaccount.blob.core.windows.net
# Look for: Certificate chain, CN=myaccount.blob.core.windows.net
# If certificate error: TLS inspection by Firewall is rewriting the cert
```

**Step 3 — Check if Azure Firewall TLS inspection is intercepting:**
```bash
# If the cert CN shows "Azure Firewall" instead of the storage account:
# → Firewall is doing TLS inspection and re-signing with its CA cert
# → The VM doesn't trust the Firewall's CA
# Fix: install Firewall CA cert in VM's trusted root store, or exempt storage from TLS inspection
```

**Step 4 — Private Endpoint subnet NSG (confirm port 443 allowed):**
```bash
az network nsg rule list \
  --nsg-name nsg-private-endpoints \
  --resource-group rg-data \
  --output table
# Must have: Allow inbound TCP 443 from the VM's subnet
```

---

## Q5. How do you diagnose and fix asymmetric routing?

**Scenario:** TCP connections from AKS to an external API randomly reset after a few seconds. You suspect asymmetric routing.

**What asymmetric routing is:**
```
NORMAL (symmetric):
  Request:  AKS → Firewall (10.0.0.4) → API (203.0.113.5)
  Response: API (203.0.113.5) → Firewall (10.0.0.4) → AKS
  ✅ Firewall sees both directions — stateful inspection works

ASYMMETRIC (broken):
  Request:  AKS → Firewall (10.0.0.4) → API (203.0.113.5)
  Response: API (203.0.113.5) → Gateway (not via Firewall) → AKS
  ❌ Firewall never saw the request, drops the "unexpected" response
     TCP connection reset
```

**Diagnose:**
```bash
# 1. Check outbound effective route from AKS node
az network nic show-effective-route-table --name aks-node-nic \
  --resource-group MC_... --output table
# Should show: 0.0.0.0/0 via VirtualAppliance (Firewall)

# 2. Check if a UDR is missing on the return path
# For spoke-to-spoke: check that BOTH spokes have routes via Firewall
# For internet: verify no 0.0.0.0/0 route via Internet exists alongside VirtualAppliance

# 3. Check Firewall logs for denied flows
AzureDiagnostics
| where Category == "AzureFirewallNetworkRule"  
| where action_s == "Deny"
| where msg_s contains "203.0.113.5"   # The external API IP
```

**Common fixes:**
```bash
# Fix 1: Missing return route — add UDR for return traffic path
az network route-table route create \
  --route-table-name rt-spoke-aks \
  --resource-group rg-spoke-aks \
  --name "return-path-external" \
  --address-prefix 203.0.113.0/24 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address $FW_IP

# Fix 2: BGP route overriding UDR (on-prem advertising more specific route)
# Solution: ensure BGP route propagation is disabled on AKS route table
az network route-table update \
  --name rt-spoke-aks \
  --resource-group rg-spoke-aks \
  --disable-bgp-route-propagation true
```

---

## Q6. Quick-Reference Troubleshooting Commands

```bash
# ── DIAGNOSTIC COMMANDS ──────────────────────────────────────────────

# 1. Effective routes on a NIC
az network nic show-effective-route-table --name <NIC> -g <RG> --output table

# 2. Effective NSG rules on a NIC (combines subnet + NIC NSGs)
az network nic show-effective-nsg --name <NIC> -g <RG> --output table

# 3. IP Flow Verify — "would this packet be allowed?"
az network watcher test-ip-flow --vm <VM> -g <RG> \
  --direction Inbound --protocol TCP --local <IP:PORT> --remote <IP:PORT>

# 4. Connection troubleshoot — "can this VM reach this address?"
az network watcher test-connectivity \
  --source-resource <VM-ID> \
  --dest-address <IP-or-FQDN> \
  --dest-port 443 \
  --resource-group rg-networking

# 5. Next hop — "what does Azure think is the next hop for this destination?"
az network watcher show-next-hop \
  --vm <VM> -g <RG> \
  --source-ip 10.1.0.5 \
  --dest-ip 10.2.0.20

# 6. VPN connection diagnose
az network watcher troubleshoot start \
  --resource <VPN-CONNECTION-ID> \
  --resource-type vpnConnection \
  --storage-account <SA-ID> \
  --storage-path "https://sa.blob.core.windows.net/vpndiag"

# 7. Private Endpoint connection state
az network private-endpoint show \
  --name <PE-NAME> -g <RG> \
  --query "privateLinkServiceConnections[0].privateLinkServiceConnectionState"

# 8. DNS resolution from Azure VM
az vm run-command invoke --vm <VM> -g <RG> \
  --command-id RunShellScript \
  --scripts "nslookup sqlserver.database.windows.net"

# 9. NSG flow log status
az network watcher flow-log show \
  --nsg <NSG-ID> \
  --location canadacentral

# 10. ExpressRoute BGP routes
az network express-route list-route-tables \
  --name <CIRCUIT> -g <RG> --peering-name AzurePrivatePeering --path Primary
```

> 💡 **Deep dive hint:** Azure Monitor Network Insights — a unified dashboard showing topology, health, and traffic across all your VNets, Gateways, Firewalls, and connections in one view.
