# Hybrid Connectivity Scenarios
### 🟢 Beginner → 🟡 Intermediate → 🔴 Advanced

---

## Q1. How do you set up a Site-to-Site VPN Gateway — complete walkthrough?

**Scenario:** A medium-sized company has an on-premises office (192.168.0.0/16) and a new Azure VNet (10.0.0.0/16). They need private connectivity for domain controllers and file servers.

**Architecture:**
```
On-Premises (192.168.0.0/16)            Azure (10.0.0.0/16)
──────────────────────────              ──────────────────────
Cisco ASA / FortiGate                   VPN Gateway (VpnGw2)
Public IP: 203.0.113.10                 Public IP: 40.xx.xx.xx
            │                                       │
            └──── IPsec/IKEv2 encrypted tunnel ─────┘
```

**Step 1 — Create the VPN Gateway (takes 25–45 minutes):**
```bash
# Create public IP for gateway
az network public-ip create \
  --name pip-vpn-gateway \
  --resource-group rg-hub-connectivity \
  --sku Standard --allocation-method Static

# Create VPN Gateway (Generation 2 recommended)
az network vnet-gateway create \
  --name vpng-hub \
  --resource-group rg-hub-connectivity \
  --vnet vnet-hub-canadacentral \
  --gateway-type Vpn \
  --vpn-type RouteBased \
  --sku VpnGw2 \
  --generation Generation2 \
  --public-ip-address pip-vpn-gateway \
  --no-wait    # This is slow — run in background
```

**Step 2 — Create Local Network Gateway (describes on-prem side):**
```bash
az network local-gateway create \
  --name lgw-onprem-hq \
  --resource-group rg-hub-connectivity \
  --location canadacentral \
  --gateway-ip-address "203.0.113.10" \        # On-prem router public IP
  --address-prefixes "192.168.0.0/16"           # On-prem subnets to reach
```

**Step 3 — Create the Connection:**
```bash
az network vpn-connection create \
  --name conn-onprem-to-azure \
  --resource-group rg-hub-connectivity \
  --vnet-gateway1 vpng-hub \
  --local-gateway2 lgw-onprem-hq \
  --connection-type IPsec \
  --shared-key "MySuperSecretKey123!"           # Must match on-prem router config
```

**Step 4 — Configure on-prem router (Cisco ASA example config):**
```
crypto ikev2 policy 1
  encryption aes-256
  integrity sha256
  group 14               ! DH group 14 (2048-bit) — matches Azure default
  prf sha256
  lifetime seconds 28800

crypto ipsec ikev2 ipsec-proposal azure-proposal
  protocol esp encryption aes-256
  protocol esp integrity sha-256

crypto map azure-vpn 10 match address azure-traffic-acl
crypto map azure-vpn 10 set peer 40.xx.xx.xx   ! Azure VPN Gateway public IP
crypto map azure-vpn 10 set ikev2 ipsec-proposal azure-proposal
crypto map azure-vpn 10 set pfs group14

tunnel-group 40.xx.xx.xx type ipsec-l2l
tunnel-group 40.xx.xx.xx ipsec-attributes
  ikev2 local-authentication pre-shared-key MySuperSecretKey123!
  ikev2 remote-authentication pre-shared-key MySuperSecretKey123!
```

**Verify connection status:**
```bash
az network vpn-connection show \
  --name conn-onprem-to-azure \
  --resource-group rg-hub-connectivity \
  --query "connectionStatus"
# Expected: "Connected"
# If: "Unknown" — IKE negotiation failed (check key, policy match)
# If: "NotConnected" — Gateway still provisioning or firewall blocking UDP 500/4500
```

> ⚠️ **Top failure causes:**
> - Shared key mismatch (copy-paste error — extra space, capitalisation)
> - IKE/IPsec policy mismatch (Azure uses IKEv2 by default, old routers use IKEv1)
> - On-prem firewall blocking **UDP 500** (IKE) and **UDP 4500** (NAT traversal)

---

## Q2. How do you set up ExpressRoute — what's the workflow from order to traffic flowing?

**Scenario:** A major bank needs a 1Gbps private dedicated circuit from their Toronto data centre to Azure Canada Central. Walk through the end-to-end process.

**Participants:**
```
Bank Data Centre (Toronto)
  │
  ├──Physical cross-connect──► Carrier PoP (e.g., Equinix TR1, Toronto)
  │                                │
  │                        ExpressRoute Partner
  │                      (Bell, Rogers, Equinix)
  │                                │
  │                  Microsoft Enterprise Edge (MSEE) Router
  │                    (in the same PoP facility)
  │                                │
  └──────────────────────────────► Azure VNet (Canada Central)
```

**Step 1 — Order circuit in Azure (get service key):**
```bash
az network express-route create \
  --name er-circuit-toronto \
  --resource-group rg-hub-connectivity \
  --location canadacentral \
  --bandwidth 1000 \                   # 1Gbps
  --peering-location "Toronto" \       # Physical location of PoP
  --provider "Bell Canada" \           # Carrier partner
  --sku-family MeteredData \           # or UnlimitedData
  --sku-tier Standard                  # or Premium (global reach)

# Get the service key — give this to your carrier
az network express-route show \
  --name er-circuit-toronto \
  --resource-group rg-hub-connectivity \
  --query "serviceKey"
# Output: "6f0c2095-abcd-1234-5678-..."
```

**Step 2 — Give service key to carrier**
- Carrier provisions the physical cross-connect
- Carrier configures VLAN and BGP peering on their end
- Timeline: 2–8 weeks depending on carrier

**Step 3 — Configure Azure Private Peering (after carrier confirms provisioning):**
```bash
az network express-route peering create \
  --circuit-name er-circuit-toronto \
  --resource-group rg-hub-connectivity \
  --peering-type AzurePrivatePeering \
  --peer-asn 65001 \                   # Your on-prem BGP AS number
  --primary-peer-address-prefix "172.16.0.0/30" \  # /30 for primary BGP link
  --secondary-peer-address-prefix "172.16.0.4/30" \ # /30 for secondary BGP link
  --vlan-id 100 \                      # VLAN ID — must match carrier
  --shared-key "BGPAuthKey123"         # Optional MD5 auth key for BGP
```

**Step 4 — Create ExpressRoute Gateway and link to circuit:**
```bash
# Create ER Gateway (30–45 min)
az network vnet-gateway create \
  --name ergw-hub \
  --resource-group rg-hub-connectivity \
  --vnet vnet-hub-canadacentral \
  --gateway-type ExpressRoute \
  --sku ErGw2AZ \                      # Zone-redundant SKU for HA
  --public-ip-address pip-ergw

# Link Gateway to Circuit
az network vpn-connection create \
  --name conn-er-circuit \
  --resource-group rg-hub-connectivity \
  --vnet-gateway1 ergw-hub \
  --express-route-circuit2 er-circuit-toronto \
  --routing-weight 0
```

**Step 5 — Verify BGP routes are propagating:**
```bash
# Check routes advertised from on-prem to Azure
az network express-route list-route-tables \
  --name er-circuit-toronto \
  --resource-group rg-hub-connectivity \
  --peering-name AzurePrivatePeering \
  --path Primary

# Check routes Azure is advertising to on-prem
az network express-route list-route-tables-summary \
  --name er-circuit-toronto \
  --resource-group rg-hub-connectivity \
  --peering-name AzurePrivatePeering \
  --path Primary
```

---

## Q3. How does Azure Arc extend Azure management to on-premises servers?

**Scenario:** You have 200 on-premises Windows and Linux servers. You want the same Azure Policy, Defender, and monitoring to apply to them as your Azure VMs.

**How Arc works:**
```
On-Premises Server                    Azure (Canada Central)
──────────────────                    ─────────────────────
Arc Agent (azcmagent)                 Azure Arc Resource Provider
  │ Outbound HTTPS (443)                │
  └──────────────────────────────────► Arc-enabled Server resource
                                        │
                                  Same ARM resource as Azure VM:
                                  ├── Azure Policy applies
                                  ├── Defender for Cloud monitors
                                  ├── Azure Monitor collects logs
                                  ├── Update Manager patches
                                  └── Run Command executes scripts
```

**Install Arc agent (Linux):**
```bash
# Generate onboarding script from Azure portal, or use CLI:
az connectedmachine connect \
  --name "server-onprem-01" \
  --resource-group rg-arc-servers \
  --location canadacentral \
  --correlation-id $(uuidgen)

# OR on the server itself:
curl -L https://aka.ms/azcmagent -o ~/install_linux_azcmagent.sh
bash ~/install_linux_azcmagent.sh

azcmagent connect \
  --service-principal-id "<SP-ID>" \
  --service-principal-secret "<SP-SECRET>" \
  --tenant-id "<TENANT-ID>" \
  --subscription-id "<SUB-ID>" \
  --resource-group rg-arc-servers \
  --location canadacentral
```

**What you gain immediately after Arc enrollment:**
- ✅ Server appears in Azure Portal as a resource
- ✅ Azure Policy assignments at the management group apply to it
- ✅ Defender for Servers covers it (same plan as Azure VMs)
- ✅ Azure Monitor Agent collects Windows Event Logs, syslog, performance counters
- ✅ Azure Update Manager patches it on a schedule
- ✅ Run Command executes scripts without SSH/RDP

> ⚠️ **Gotcha:** Arc agent communicates **outbound only** on port 443. If your on-prem proxy intercepts HTTPS, you need to configure proxy settings in the Arc agent: `azcmagent config set proxy.url http://proxy:8080`

---

## Q4. How do you use Azure File Sync for a hybrid file server migration?

**Scenario:** You have a 10TB Windows file server that employees access daily. You want to move it to Azure Files but keep access seamless for on-prem users while shrinking the local storage footprint.

**Architecture with tiering:**
```
On-Premises File Server (500GB local cache)
  │ Sync Agent
  │ ← tiered files (reparse points, no local data)
  │ ← hot files (fully local)
  ▼
Storage Sync Service
  │
  ▼
Azure Files Share (10TB, cloud storage, all data)
  │
  ▼
Azure VM or AKS (can also mount the share directly — no sync needed)
```

**Setup steps:**
```bash
# Step 1: Create Storage Account + Azure Files share
az storage account create \
  --name stfilesyncprod \
  --resource-group rg-data \
  --sku Standard_LRS \          # or Standard_ZRS for zone-redundancy
  --kind StorageV2 \
  --enable-large-file-share

az storage share create \
  --account-name stfilesyncprod \
  --name fileserver-share \
  --quota 10240                 # 10TB in GB

# Step 2: Create Storage Sync Service
az storagesync create \
  --resource-group rg-data \
  --name syncservice-prod \
  --location canadacentral

# Step 3: Create Sync Group
az storagesync sync-group create \
  --resource-group rg-data \
  --storage-sync-service syncservice-prod \
  --name syncgroup-fileserver

# Step 4: Add cloud endpoint (Azure Files share)
az storagesync sync-group cloud-endpoint create \
  --resource-group rg-data \
  --storage-sync-service syncservice-prod \
  --sync-group-name syncgroup-fileserver \
  --name cloud-endpoint \
  --storage-account-resource-id $(az storage account show \
    --name stfilesyncprod --resource-group rg-data --query id -o tsv) \
  --azure-file-share-name fileserver-share
```

**Step 5 — Install agent on Windows server and register:**
```powershell
# On Windows file server — download agent from portal, then:
# Open Storage Sync Agent registration UI
# Enter: Subscription, Resource Group, Storage Sync Service name
# Authentication: sign in with Azure account that has Contributor role

# Step 6: Add server endpoint (local folder → cloud sync)
az storagesync sync-group server-endpoint create \
  --resource-group rg-data \
  --storage-sync-service syncservice-prod \
  --sync-group-name syncgroup-fileserver \
  --server-id "/subscriptions/.../servers/FILE-SERVER-01" \
  --server-local-path "D:\FileShares\HR" \
  --cloud-tiering enabled \
  --volume-free-space-percent 20    # Keep 20% of D: free — older files get tiered
```

> 💡 **Deep dive hint:** Azure File Sync cloud tiering heatmap — how Azure decides which files to tier based on access frequency, and how to force a file back to local with `Invoke-StorageSyncFileRecall`.

---

## Q5. How do you configure hybrid DNS so on-premises servers resolve Azure Private Endpoint hostnames?

**Scenario:** Your on-premises SQL application (Windows) can reach Azure via ExpressRoute, but `sqlserver.database.windows.net` resolves to the public IP instead of the Private Endpoint IP `10.2.0.20`.

**The problem visualised:**
```
On-Prem Windows App Server
  │ DNS query: sqlserver.database.windows.net
  ▼
Windows DNS Server (192.168.0.10)
  │ No conditional forwarder configured for azure zones
  │ Forwards to ISP DNS
  ▼
Public DNS → returns 52.184.x.x  ← WRONG (public IP, but PE exists)

Desired path:
On-Prem Windows App Server
  │ DNS query: sqlserver.database.windows.net
  ▼
Windows DNS Server → Conditional Forwarder: database.windows.net → 10.0.3.4
  ▼
Azure DNS Private Resolver (inbound endpoint: 10.0.3.4)
  │ Checks Private DNS Zone: privatelink.database.windows.net
  ▼
Returns: 10.2.0.20  ← CORRECT (Private Endpoint IP)
```

**Windows DNS Conditional Forwarder configuration (PowerShell on DNS server):**
```powershell
# Add conditional forwarder for each Azure zone
$zones = @(
  "database.windows.net",
  "blob.core.windows.net",
  "dfs.core.windows.net",
  "vaultcore.azure.net",
  "servicebus.windows.net",
  "documents.azure.com",
  "azurecr.io"
)

$resolverIP = "10.0.3.4"  # Azure DNS Private Resolver inbound endpoint

foreach ($zone in $zones) {
  Add-DnsServerConditionalForwarderZone `
    -Name $zone `
    -MasterServers $resolverIP `
    -PassThru
}

# Verify forwarders
Get-DnsServerZone | Where-Object {$_.ZoneType -eq "Forwarder"} | 
  Select-Object ZoneName, MasterServers
```

**Verify from on-prem server:**
```powershell
# Should return 10.2.0.20 (private IP), not a public IP
Resolve-DnsName sqlserver.database.windows.net

# Full DNS resolution path
nslookup sqlserver.database.windows.net 192.168.0.10
# Server: corp-dns (192.168.0.10)
# Address: 10.2.0.20  ← Success
```

> ⚠️ **Gotcha:** If you forward `database.windows.net` instead of `privatelink.database.windows.net`, it works — because Azure's CNAME chain resolves `sqlserver.database.windows.net` → `sqlserver.privatelink.database.windows.net` → private IP. Both forwarding levels work, but `database.windows.net` is broader and safer to forward.
