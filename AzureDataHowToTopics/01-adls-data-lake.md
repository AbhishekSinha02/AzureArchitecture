# Azure Data Lake Storage Gen2
### 🟢 Beginner → 🟡 Intermediate

> *"A data lake without zones is a data swamp.  
>  Structure is not a constraint — it's what makes the data usable."*

---

## Q1. How do you structure a data lake in ADLS Gen2 for a financial services workload?

**Scenario:** You're building a data platform for a Canadian bank. Raw transaction data arrives from 5 source systems. Analysts need clean data. ML engineers need enriched features. The data governance team needs audit trails. How do you structure the lake?

```
ADLS Gen2 — Four-Zone Medallion Architecture:

  Container: datalake
  ├── raw/           Zone 1: Immutable landing zone
  │   ├── transactions/2024/01/15/batch_001.json
  │   └── accounts/2024/01/15/extract.csv
  │
  ├── curated/       Zone 2: Validated, deduplicated, schema-enforced
  │   ├── transactions/ (Delta Lake format)
  │   └── accounts/     (Parquet, partitioned by date)
  │
  ├── enriched/      Zone 3: Business logic applied, joined, aggregated
  │   ├── customer_360/
  │   └── risk_scores/
  │
  └── serving/       Zone 4: Query-ready for analytics / ML
      ├── powerbi/   (aggregated for Power BI DirectQuery)
      └── features/  (feature store for ML models)

Access pattern:
  ADF ingestion jobs  → raw/  (write only)
  Databricks ETL      → raw/ (read) → curated/ (write)
  Databricks feature  → curated/ (read) → enriched/ (write)
  Synapse SQL         → serving/ (read)
  Power BI            → serving/ (read)
```

```bash
# Create storage account with hierarchical namespace (required for ADLS Gen2)
az storage account create \
  --name adlsprodcc \
  --resource-group rg-data \
  --location canadacentral \
  --sku Standard_LRS \
  --kind StorageV2 \
  --hierarchical-namespace true \    # This flag enables ADLS Gen2 HNS
  --enable-nfs-v3 false \
  --min-tls-version TLS1_2 \
  --allow-blob-public-access false   # No public access — Private Endpoint only

# Create zone containers
for zone in raw curated enriched serving; do
  az storage fs create \
    --name $zone \
    --account-name adlsprodcc \
    --auth-mode login
done
```

> ⚠️ **Gotcha:** Hierarchical Namespace (HNS) cannot be enabled on an existing storage account — it must be set at creation time. Enabling it requires a full data migration to a new account. Plan this before ingesting any data.

> 💡 **Deep dive hint:** Delta Lake format on ADLS Gen2 — why Parquet + transaction log (the `_delta_log/` directory) gives ACID guarantees on top of object storage, and how `OPTIMIZE` compacts small files that accumulate from streaming writes.

---

## Q2. How do you control access to ADLS Gen2 with RBAC and ACLs?

**Scenario:** Three teams need different access: ADF service writes to `raw/`, Databricks reads `raw/` and writes `curated/`, Analysts read only `serving/`. How do you enforce least-privilege without breaking pipelines?

```
ADLS Gen2 — Two-layer access model:

  Layer 1: Azure RBAC (subscription/resource level)
  ─────────────────────────────────────────────────
  Storage Blob Data Contributor  → read + write all containers
  Storage Blob Data Reader       → read all containers
  Storage Blob Data Owner        → full control + set ACLs

  Layer 2: POSIX ACLs (folder/file level — requires HNS)
  ──────────────────────────────────────────────────────
  /raw/transactions/             rwx for ADF MI
  /curated/transactions/         r-x for Databricks MI (read)
                                 rwx for Databricks ETL MI (write)
  /serving/                      r-x for Analysts group

  Evaluation: RBAC checked first → if denied → check ACL
              If either grants access → allowed
```

```bash
# Grant ADF Managed Identity write access to raw/ container only
ADF_MI=$(az datafactory show \
  --name adf-prod-cc -g rg-data \
  --query identity.principalId -o tsv)

ADLS_ID=$(az storage account show \
  --name adlsprodcc -g rg-data --query id -o tsv)

# RBAC at account level (broad — use ACLs to restrict further)
az role assignment create \
  --role "Storage Blob Data Contributor" \
  --assignee $ADF_MI \
  --scope "$ADLS_ID/blobServices/default/containers/raw"   # Scope to container

# Set POSIX ACL on specific folder (requires Storage Blob Data Owner or Owner)
az storage fs access set \
  --acl "user:$ADF_MI:rwx,default:user:$ADF_MI:rwx" \
  --path "raw/transactions" \
  --file-system raw \
  --account-name adlsprodcc \
  --auth-mode login
# "default:" prefix sets the ACL on all NEW files created in this directory

# Check effective ACLs on a path
az storage fs access show \
  --path "raw/transactions" \
  --file-system raw \
  --account-name adlsprodcc \
  --auth-mode login
```

| Role | Can set ACLs | Scope | Use for |
|------|-------------|-------|---------|
| Storage Blob Data Owner | ✅ | Container | Admin / platform team |
| Storage Blob Data Contributor | ❌ | Container | Service principals (ADF, ADB) |
| Storage Blob Data Reader | ❌ | Container | Analysts, Power BI |
| ACL `rwx` | N/A | Folder/file | Fine-grained per-directory control |

> ⚠️ **Gotcha:** Default ACLs (`default:user:...`) must be set on parent directories at creation time, not retroactively. If you create a directory and then add a default ACL, files created BEFORE the ACL change do not inherit it. You must explicitly set ACLs on all existing child items with `--recursive`.

---

## Q3. How do you configure lifecycle policies to auto-tier cold data to Archive?

**Scenario:** Raw transaction files older than 90 days are never accessed but must be retained for 7 years (FINTRAC). Keeping 7 years of raw data in Hot tier costs $140K/year. Archive tier costs $3K/year.

```
Storage tier cost comparison (Canada Central):
  Hot     : ~$0.023/GB/month  ← active data, ms access
  Cool    : ~$0.010/GB/month  ← 30-day minimum, seconds access
  Cold    : ~$0.004/GB/month  ← 90-day minimum, seconds access
  Archive : ~$0.001/GB/month  ← 180-day minimum, hours to rehydrate

Lifecycle policy flow:
  Day 0   → blob created in Hot tier
  Day 30  → auto-move to Cool (last-modified > 30d)
  Day 90  → auto-move to Cold
  Day 180 → auto-move to Archive
  Day 2555 (7yr) → auto-delete
```

```bash
# Apply lifecycle management policy via JSON
az storage account management-policy create \
  --account-name adlsprodcc \
  --resource-group rg-data \
  --policy '{
    "rules": [
      {
        "name": "raw-tiering",
        "enabled": true,
        "type": "Lifecycle",
        "definition": {
          "filters": {
            "blobTypes": ["blockBlob"],
            "prefixMatch": ["raw/"]
          },
          "actions": {
            "baseBlob": {
              "tierToCool": {"daysAfterModificationGreaterThan": 30},
              "tierToCold": {"daysAfterModificationGreaterThan": 90},
              "tierToArchive": {"daysAfterModificationGreaterThan": 180},
              "delete": {"daysAfterModificationGreaterThan": 2555}
            }
          }
        }
      }
    ]
  }'
# Policies run once per day — not instant

# Check current tier of a blob
az storage blob show \
  --name "raw/transactions/2024/01/15/batch_001.json" \
  --container-name raw \
  --account-name adlsprodcc \
  --query "properties.blobTier" \
  --auth-mode login

# Rehydrate an archived blob (needed before reading)
az storage blob set-tier \
  --name "raw/transactions/2023/01/15/batch_001.json" \
  --container-name raw \
  --account-name adlsprodcc \
  --tier Cool \
  --rehydrate-priority High    # High = 1-hour rehydration (Standard = up to 15hr)
```

> ⚠️ **Gotcha:** Archive tier blobs cannot be read directly — they must be rehydrated to Hot or Cool first, which takes 1–15 hours depending on priority. If your Databricks job reads archived files, it will fail silently with a `BlobArchived` error. Always filter out archived blobs before processing.

---

## Q4. How do you enable and query ADLS Gen2 with a Private Endpoint?

**Scenario:** Security policy requires all ADLS access to go through Private Endpoints — no public internet exposure. ADF running in an Azure VNet must reach ADLS without traversing the internet.

```
Private Endpoint architecture for ADLS Gen2:

  ADF Managed VNet
       │
       │ DNS: adlsprodcc.dfs.core.windows.net
       ▼
  Private DNS Zone: privatelink.dfs.core.windows.net
       │ A record: adlsprodcc → 10.1.5.4 (PE IP)
       ▼
  Private Endpoint (NIC in spoke VNet subnet)
  10.1.5.4
       │
       ▼
  ADLS Gen2: adlsprodcc
  (public access disabled — rejects all requests not via PE)

  Two endpoints needed:
  - .dfs.core.windows.net (for ADLS Gen2 HNS / Data Lake API)
  - .blob.core.windows.net (for Blob API — some ADF connectors use this)
```

```bash
# Create Private Endpoint for ADLS Gen2 DFS endpoint
ADLS_ID=$(az storage account show \
  --name adlsprodcc -g rg-data --query id -o tsv)

az network private-endpoint create \
  --name pe-adls-dfs \
  --resource-group rg-data \
  --vnet-name vnet-spoke-data \
  --subnet snet-privateendpoints \
  --private-connection-resource-id $ADLS_ID \
  --group-id dfs \               # dfs = Data Lake Storage endpoint
  --connection-name conn-adls-dfs

# Also create for blob endpoint (needed by some services)
az network private-endpoint create \
  --name pe-adls-blob \
  --resource-group rg-data \
  --vnet-name vnet-spoke-data \
  --subnet snet-privateendpoints \
  --private-connection-resource-id $ADLS_ID \
  --group-id blob \
  --connection-name conn-adls-blob

# Create Private DNS Zone and link to VNet
az network private-dns zone create \
  --resource-group rg-data \
  --name "privatelink.dfs.core.windows.net"

az network private-dns link vnet create \
  --resource-group rg-data \
  --zone-name "privatelink.dfs.core.windows.net" \
  --name link-spoke-data \
  --virtual-network vnet-spoke-data \
  --registration-enabled false

# Auto-register DNS record from PE
az network private-endpoint dns-zone-group create \
  --resource-group rg-data \
  --endpoint-name pe-adls-dfs \
  --name dzg-adls \
  --private-dns-zone "privatelink.dfs.core.windows.net" \
  --zone-name adls

# Disable public access after PE is confirmed working
az storage account update \
  --name adlsprodcc -g rg-data \
  --public-network-access Disabled
```

> ⚠️ **Gotcha:** ADLS Gen2 needs **two** Private Endpoints: one for `dfs` (Data Lake filesystem API) and one for `blob` (Blob API). **Azure Data Factory** and **Databricks** may use either endpoint depending on the connector. Missing the `blob` endpoint causes intermittent failures that are hard to trace because the DFS endpoint works fine.
