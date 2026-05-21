# Azure Networking — Commands Reference
### Every command you'll use in an interview or production incident

> Format: **What it does** → **The command** → **What to look for in the output**

---

## Section 1: VNet & Subnet Commands

**1. List all VNets in a subscription with their address spaces:**
```bash
az network vnet list --output table \
  --query "[].{Name:name, RG:resourceGroup, CIDR:addressSpace.addressPrefixes[0]}"
```

**2. Show subnets in a VNet:**
```bash
az network vnet subnet list \
  --vnet-name vnet-hub-canadacentral \
  --resource-group rg-hub-connectivity \
  --output table \
  --query "[].{Name:name, CIDR:addressPrefix, NSG:networkSecurityGroup.id, RT:routeTable.id}"
```

**3. Check if two VNet CIDRs overlap (causes peering failure):**
```bash
# There's no built-in CLI command — use this logic:
# VNet1: 10.1.0.0/16 and VNet2: 10.1.128.0/17 OVERLAP
# VNet1: 10.1.0.0/16 and VNet2: 10.2.0.0/16 NO overlap
# Tool: https://cidr.xyz — visual CIDR calculator
```

**4. Update a subnet to add or change an NSG:**
```bash
az network vnet subnet update \
  --name snet-web \
  --vnet-name vnet-app \
  --resource-group rg-app \
  --network-security-group nsg-web-subnet
```

**5. Remove NSG from a subnet (to test if NSG is the problem):**
```bash
az network vnet subnet update \
  --name snet-web \
  --vnet-name vnet-app \
  --resource-group rg-app \
  --network-security-group ""
```

---

## Section 2: Peering Commands

**6. List peerings for a VNet with connection state:**
```bash
az network vnet peering list \
  --vnet-name vnet-hub-canadacentral \
  --resource-group rg-hub-connectivity \
  --output table \
  --query "[].{Name:name, State:peeringState, RemoteVNet:remoteVirtualNetwork.id, FwdTraffic:allowForwardedTraffic}"
# State should be "Connected" — "Initiated" means other side peering not created yet
```

**7. Sync a peering (after adding new address space to a VNet):**
```bash
# When you add a new CIDR to a peered VNet, the peering gets out of sync
az network vnet peering sync \
  --name hub-to-spoke-aks \
  --vnet-name vnet-hub-canadacentral \
  --resource-group rg-hub-connectivity
```

---

## Section 3: NSG Commands

**8. List all rules in an NSG sorted by priority:**
```bash
az network nsg rule list \
  --nsg-name nsg-prod-subnet \
  --resource-group rg-prod \
  --output table \
  --query "sort_by([].{Priority:priority, Name:name, Dir:direction, Access:access, Src:sourceAddressPrefix, Dst:destinationAddressPrefix, Port:destinationPortRange}, &Priority)"
```

**9. Add a quick DENY rule at top priority:**
```bash
az network nsg rule create \
  --nsg-name nsg-prod-subnet \
  --resource-group rg-prod \
  --name "emergency-block-ip" \
  --priority 100 \
  --direction Inbound \
  --access Deny \
  --source-address-prefixes "203.0.113.5" \   # Attacker IP
  --destination-port-ranges "*"
```

**10. Delete a specific NSG rule:**
```bash
az network nsg rule delete \
  --nsg-name nsg-prod-subnet \
  --resource-group rg-prod \
  --name "temp-vendor-exception-20240115"
```

**11. Show effective NSG rules on a VM's NIC (combined subnet + NIC rules):**
```bash
az network nic show-effective-nsg \
  --name vm-prod-01-nic \
  --resource-group rg-vms \
  --output table
```

---

## Section 4: Route Table & Routing Commands

**12. Show all routes in a route table:**
```bash
az network route-table route list \
  --route-table-name rt-spoke-aks \
  --resource-group rg-spoke-aks \
  --output table
```

**13. Show effective routes on a VM NIC (what Azure actually routes):**
```bash
az network nic show-effective-route-table \
  --name vm-prod-nic \
  --resource-group rg-vms \
  --output table
# KEY COLUMNS: AddressPrefix, NextHopType, NextHopIpAddress, Source (User/BGP/Default)
# Look for: 0.0.0.0/0 should show NextHopType: VirtualAppliance (Firewall)
#           NOT Internet
```

**14. Show next hop for a specific destination:**
```bash
az network watcher show-next-hop \
  --resource-group rg-vms \
  --vm my-vm \
  --source-ip 10.1.0.5 \
  --dest-ip 8.8.8.8
# Returns: nextHopType, nextHopIpAddress
```

**15. Add a specific route to a route table:**
```bash
az network route-table route create \
  --route-table-name rt-spoke-aks \
  --resource-group rg-spoke-aks \
  --name "route-to-spoke2" \
  --address-prefix 10.2.0.0/16 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address 10.0.0.4
```

---

## Section 5: Private Endpoint & DNS Commands

**16. List all Private Endpoints in a subscription:**
```bash
az network private-endpoint list \
  --output table \
  --query "[].{Name:name, RG:resourceGroup, SubResource:privateLinkServiceConnections[0].groupIds[0], State:privateLinkServiceConnections[0].privateLinkServiceConnectionState.status}"
```

**17. Show Private DNS Zones and their VNet links:**
```bash
az network private-dns zone list --output table

az network private-dns link vnet list \
  --resource-group rg-hub-connectivity \
  --zone-name "privatelink.database.windows.net" \
  --output table
# Should list all VNets that have resolution for this zone
```

**18. Show all DNS records in a Private DNS Zone:**
```bash
az network private-dns record-set list \
  --resource-group rg-hub-connectivity \
  --zone-name "privatelink.database.windows.net" \
  --output table
```

**19. Manually test DNS resolution from a VM:**
```bash
az vm run-command invoke \
  --resource-group rg-vms \
  --name my-vm \
  --command-id RunShellScript \
  --scripts "nslookup mysqlserver.database.windows.net 168.63.129.16"
# 168.63.129.16 = Azure's internal DNS resolver
```

---

## Section 6: Network Watcher Diagnostic Commands

**20. IP Flow Verify — test if a packet would be allowed:**
```bash
az network watcher test-ip-flow \
  --resource-group rg-vms \
  --vm my-vm \
  --direction Inbound \
  --protocol TCP \
  --local 10.1.0.5:443 \
  --remote 203.0.113.1:54321
# Returns: Access (Allow/Deny), RuleName (which NSG rule matched)
```

**21. Connection Troubleshoot — end-to-end reachability test:**
```bash
az network watcher test-connectivity \
  --resource-group rg-networking \
  --source-resource /subscriptions/.../vm/my-vm \
  --dest-address "10.2.0.20" \
  --dest-port 1433
# Returns: connectionStatus (Reachable/Unreachable), avgLatencyInMs, hops
```

**22. Enable NSG Flow Logs:**
```bash
az network watcher flow-log create \
  --location canadacentral \
  --resource-group rg-networking \
  --name fl-nsg-prod \
  --nsg /subscriptions/.../nsg-prod-subnet \
  --storage-account /subscriptions/.../stflowlogs \
  --workspace /subscriptions/.../workspaces/law-prod \
  --enabled true \
  --format JSON \
  --log-version 2
```

**23. Packet capture on a VM (traffic capture without deploying anything):**
```bash
az network watcher packet-capture create \
  --resource-group rg-vms \
  --vm my-vm \
  --name capture-session-01 \
  --storage-account stpacketcaptures \
  --time-limit 300 \                     # 5 minutes
  --filters '[{"protocol":"TCP","localIPAddress":"10.1.0.5","localPort":"443"}]'

# Stop capture
az network watcher packet-capture stop \
  --name capture-session-01 \
  --location canadacentral

# Get capture file location
az network watcher packet-capture show \
  --name capture-session-01 \
  --location canadacentral \
  --query "storageLocation.storageUrl"
# Download .cap file and open in Wireshark
```

---

## Section 7: Firewall Commands

**24. Show Azure Firewall rules (network rule collections):**
```bash
az network firewall network-rule collection list \
  --firewall-name fw-hub-prod \
  --resource-group rg-hub-connectivity \
  --output table
```

**25. Show FQDN/application rules:**
```bash
az network firewall application-rule collection list \
  --firewall-name fw-hub-prod \
  --resource-group rg-hub-connectivity \
  --output table
```

**26. Query Firewall logs for denied traffic:**
```kusto
// Run in Log Analytics
AzureDiagnostics
| where Category in ("AzureFirewallNetworkRule", "AzureFirewallApplicationRule")
| where action_s == "Deny"
| where TimeGenerated > ago(1h)
| project TimeGenerated, Category, msg_s, action_s
| order by TimeGenerated desc
| limit 50
```

---

## Section 8: VPN & ExpressRoute Commands

**27. Check VPN connection status:**
```bash
az network vpn-connection show \
  --name conn-onprem-to-azure \
  --resource-group rg-hub-connectivity \
  --query "{Status:connectionStatus, Bandwidth:ingressBytesTransferred, EgressBytes:egressBytesTransferred}"
```

**28. View routes Azure is receiving from ExpressRoute:**
```bash
az network express-route list-route-tables \
  --name er-circuit-toronto \
  --resource-group rg-hub-connectivity \
  --peering-name AzurePrivatePeering \
  --path Primary \
  --output table
```

**29. Check ExpressRoute circuit provisioning state:**
```bash
az network express-route show \
  --name er-circuit-toronto \
  --resource-group rg-hub-connectivity \
  --query "{CircuitState:circuitProvisioningState, ProviderState:serviceProviderProvisioningState, Bandwidth:serviceProviderProperties.bandwidthInMbps}"
```

---

## Section 9: AKS Networking Commands

**30. Check AKS cluster network profile:**
```bash
az aks show \
  --name aks-prod \
  --resource-group rg-spoke-aks \
  --query "networkProfile.{Plugin:networkPlugin, Mode:networkPluginMode, ServiceCIDR:serviceCidr, DNSServiceIP:dnsServiceIp}"
```

**31. Find which node a pod is running on (for NIC-level diagnostics):**
```bash
kubectl get pod my-pod -o wide
# NODE column gives you the node name
```

**32. Test connectivity from inside a pod:**
```bash
kubectl exec -it my-pod -- /bin/sh
# Then inside pod:
nslookup sqlserver.database.windows.net
curl -v https://sqlserver.database.windows.net:1433
nc -zv 10.2.0.20 1433
```

**33. Check AKS outbound type (is it using Firewall or Load Balancer for egress?):**
```bash
az aks show \
  --name aks-prod \
  --resource-group rg-spoke-aks \
  --query "networkProfile.outboundType"
# "loadBalancer" = direct outbound (bypass Firewall)
# "userDefinedRouting" = routes through UDR → Firewall
```

> 💡 **Deep dive hint:** `userDefinedRouting` outbound type for AKS requires the Firewall rules for AKS cluster to be in place BEFORE the cluster is created — otherwise AKS creation fails because the API server can't be reached during bootstrap.
