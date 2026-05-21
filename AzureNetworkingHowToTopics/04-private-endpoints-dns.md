# Private Endpoints & DNS — End-to-End
### 🟢 Beginner → 🟡 Intermediate → 🔴 Advanced

---

## Q1. What is a Private Endpoint and why does it matter for security?

**Scenario:** Your AKS application connects to Azure SQL Database over the internet. Security wants no public traffic to the database. What do you do?

**Before Private Endpoint:**
```
AKS Pod ──► Public Internet ──► Azure SQL (52.184.x.x:1433)
             ↑
        Traffic exits VNet,
        traverses internet,
        hits public endpoint
```

**After Private Endpoint:**
```
AKS Pod ──► Private Endpoint (10.1.0.20:1433) ──► Azure SQL
             ↑
        Traffic stays entirely
        within Azure backbone.
        Azure SQL has NO public IP.
```

**What a Private Endpoint does:**
- 🔑 Injects the Azure service into **your subnet** as a NIC with a **private IP**
- 🔑 The service's public endpoint can be **completely disabled** (`publicNetworkAccess: Disabled`)
- 🔑 Traffic never leaves the Azure backbone — not even between Azure regions
- 🔑 Works over **ExpressRoute/VPN** — on-prem servers can also reach Azure services through the private IP

**Services that support Private Endpoints (common ones):**

| Service | DNS zone |
|---------|---------|
| Azure SQL Database | `privatelink.database.windows.net` |
| Azure Storage (Blob) | `privatelink.blob.core.windows.net` |
| Azure Storage (ADLS) | `privatelink.dfs.core.windows.net` |
| Azure Key Vault | `privatelink.vaultcore.azure.net` |
| CosmosDB | `privatelink.documents.azure.com` |
| Service Bus | `privatelink.servicebus.windows.net` |
| Event Hubs | `privatelink.servicebus.windows.net` |
| Azure Container Registry | `privatelink.azurecr.io` |
| AKS API Server | `privatelink.<region>.azmk8s.io` |

---

## Q2. How do you create a Private Endpoint for Azure SQL step by step?

**Scenario:** You have Azure SQL (`mysqlserver.database.windows.net`) and an AKS spoke VNet. Build the Private Endpoint so pods connect via private IP.

**Step 1 — Create the Private Endpoint:**
```bash
# Get SQL server resource ID
SQL_ID=$(az sql server show \
  --name mysqlserver \
  --resource-group rg-data \
  --query id -o tsv)

# Create private endpoint in the AKS spoke subnet
az network private-endpoint create \
  --name pe-mysqlserver \
  --resource-group rg-spoke-aks \
  --vnet-name vnet-spoke-aks \
  --subnet snet-private-endpoints \           # dedicated PE subnet recommended
  --private-connection-resource-id $SQL_ID \
  --group-ids sqlServer \
  --connection-name conn-mysqlserver \
  --location canadacentral
```

**Step 2 — Create the Private DNS Zone:**
```bash
az network private-dns zone create \
  --resource-group rg-hub-connectivity \      # Put DNS zones in Hub for centralised management
  --name "privatelink.database.windows.net"
```

**Step 3 — Link DNS Zone to Hub VNet:**
```bash
az network private-dns link vnet create \
  --resource-group rg-hub-connectivity \
  --zone-name "privatelink.database.windows.net" \
  --name "dns-link-to-hub" \
  --virtual-network vnet-hub-canadacentral \
  --registration-enabled false
```

**Step 4 — Create DNS Zone Group (links PE to DNS Zone automatically):**
```bash
az network private-endpoint dns-zone-group create \
  --endpoint-name pe-mysqlserver \
  --resource-group rg-spoke-aks \
  --name pe-dns-group \
  --private-dns-zone "privatelink.database.windows.net" \
  --zone-name sql
```

**Step 5 — Disable public access on SQL:**
```bash
az sql server update \
  --name mysqlserver \
  --resource-group rg-data \
  --public-network-access Disabled
```

**Step 6 — Verify DNS resolution from pod:**
```bash
# From inside AKS pod:
nslookup mysqlserver.database.windows.net
# Expected: returns 10.1.0.20 (private IP)
# Wrong:    returns 52.184.x.x (public IP → DNS zone not linked)
```

---

## Q3. What is the most common Private Endpoint failure and how do you fix it?

**Scenario:** You created the Private Endpoint, but from the AKS pod `nslookup mysqlserver.database.windows.net` still returns the public IP. Why?

**Root cause: The Private DNS Zone is NOT linked to the VNet where the pod lives.**

```
Lookup flow when DNS zone IS linked:
Pod → VNet DNS (168.63.129.16) → Azure DNS → 
  → "I have a Private DNS Zone for privatelink.database.windows.net linked to this VNet"
  → Returns: 10.1.0.20  ✅

Lookup flow when DNS zone is NOT linked:
Pod → VNet DNS (168.63.129.16) → Azure DNS →
  → "No Private DNS Zone linked for this VNet"
  → Falls back to public DNS → Returns: 52.184.x.x  ❌
```

**Fix: Link the DNS zone to every VNet that needs to resolve it**

```bash
# Check which VNets have the link
az network private-dns link vnet list \
  --resource-group rg-hub-connectivity \
  --zone-name "privatelink.database.windows.net" \
  --output table

# Add link if missing
az network private-dns link vnet create \
  --resource-group rg-hub-connectivity \
  --zone-name "privatelink.database.windows.net" \
  --name "dns-link-to-spoke-aks" \
  --virtual-network vnet-spoke-aks \
  --registration-enabled false
```

**In Hub-Spoke: centralise all DNS Zones in the Hub VNet**
```
Approach: Create all Private DNS Zones in Hub.
          Link each zone to the Hub VNet ONCE.
          All spoke VNets resolve via Hub (peering carries DNS).
          
Result:   When a new spoke is added, you do NOT need to add
          individual DNS zone links for each zone.
          The peering to Hub is enough.
```

> ⚠️ **This only works if** the spoke's peering has `useRemoteGateways = false` AND the Hub's DNS Resolver is configured. For spokes that don't use the Hub's gateway, DNS still traverses the peering to the Hub's linked zones.

---

## Q4. How do you configure Private Endpoints for on-premises access?

**Scenario:** Your on-premises data centre servers need to access Azure Storage via a Private Endpoint through your ExpressRoute connection. The servers run their own DNS servers (Windows DNS). How do you make it work?

**The challenge:**
```
On-prem server                   Azure
──────────────                   ─────
Corp DNS server                  Private DNS Zone: privatelink.blob.core.windows.net
  (doesn't know about             (linked to Hub VNet only)
   Azure private zones)
```

**Solution: DNS forwarding with Azure DNS Private Resolver**

```
On-prem server
  │ DNS query: myaccount.blob.core.windows.net
  ▼
Corp Windows DNS Server
  │ Conditional Forwarder:
  │ blob.core.windows.net → 10.0.3.4  (DNS Resolver inbound endpoint)
  ▼
Azure DNS Private Resolver (Inbound Endpoint: 10.0.3.4)
  │ Resolves against Private DNS Zone linked to Hub VNet
  ▼
Returns: 10.2.0.15 (Private Endpoint IP)  ✅
  │
  ▼
On-prem server connects to 10.2.0.15 via ExpressRoute
```

**Deploy DNS Private Resolver:**
```bash
# Create inbound endpoint subnet (dedicated, /28 minimum)
az network vnet subnet create \
  --name snet-dns-resolver-inbound \
  --vnet-name vnet-hub-canadacentral \
  --resource-group rg-hub-connectivity \
  --address-prefix 10.0.3.0/28 \
  --delegations Microsoft.Network/dnsResolvers

# Create DNS Private Resolver
az dns-resolver create \
  --name dns-resolver-hub \
  --resource-group rg-hub-connectivity \
  --location canadacentral \
  --id $(az network vnet show \
    --name vnet-hub-canadacentral \
    --resource-group rg-hub-connectivity \
    --query id -o tsv)

# Create inbound endpoint
az dns-resolver inbound-endpoint create \
  --dns-resolver-name dns-resolver-hub \
  --resource-group rg-hub-connectivity \
  --name inbound-endpoint \
  --ip-configurations '[{"privateIpAllocationMethod":"Static","privateIpAddress":"10.0.3.4","subnet":{"id":"<subnet-id>"}}]'
```

**Configure on-premises Windows DNS Conditional Forwarder:**
```
On Windows DNS Server:
  → DNS Manager → Forward Lookup Zones → New Zone → Stub Zone
  → Zone name: blob.core.windows.net
  → Master servers: 10.0.3.4 (Azure DNS Resolver inbound endpoint)
  
Repeat for each service:
  database.windows.net → 10.0.3.4
  vaultcore.azure.net → 10.0.3.4
  servicebus.windows.net → 10.0.3.4
```

---

## Q5. How do you create Private Endpoints for multiple services efficiently in Terraform?

**Scenario:** You have 8 services that all need Private Endpoints. Building each one manually takes too long. How do you automate this?

```hcl
# Terraform: iterate over services with a map
locals {
  private_endpoints = {
    "sql" = {
      resource_id  = azurerm_mssql_server.main.id
      subresource  = "sqlServer"
      dns_zone     = "privatelink.database.windows.net"
      private_ip   = "10.1.0.20"
    }
    "storage-blob" = {
      resource_id  = azurerm_storage_account.main.id
      subresource  = "blob"
      dns_zone     = "privatelink.blob.core.windows.net"
      private_ip   = "10.1.0.21"
    }
    "keyvault" = {
      resource_id  = azurerm_key_vault.main.id
      subresource  = "vault"
      dns_zone     = "privatelink.vaultcore.azure.net"
      private_ip   = "10.1.0.22"
    }
    "cosmos" = {
      resource_id  = azurerm_cosmosdb_account.main.id
      subresource  = "Sql"
      dns_zone     = "privatelink.documents.azure.com"
      private_ip   = "10.1.0.23"
    }
  }
}

resource "azurerm_private_endpoint" "services" {
  for_each            = local.private_endpoints
  name                = "pe-${each.key}"
  resource_group_name = azurerm_resource_group.spoke.name
  location            = var.location
  subnet_id           = azurerm_subnet.private_endpoints.id

  private_service_connection {
    name                           = "conn-${each.key}"
    private_connection_resource_id = each.value.resource_id
    subresource_names              = [each.value.subresource]
    is_manual_connection           = false
  }

  private_dns_zone_group {
    name                 = "dns-group-${each.key}"
    private_dns_zone_ids = [azurerm_private_dns_zone.zones[each.key].id]
  }
}
```

> 💡 **Deep dive hint:** Private Endpoint policies — how to require approval for Private Endpoints connecting to your resources (so another tenant can't silently create a PE into your storage account).
