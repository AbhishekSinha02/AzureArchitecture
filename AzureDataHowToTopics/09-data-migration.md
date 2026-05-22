# Azure Data Migration
### 🟡 Intermediate → 🔴 Advanced

> *"A migration is not a copy. It's a handoff.  
>  The handoff fails when source and destination disagree about what 'done' means."*

---

## Q1. How do you migrate 10TB of files from on-premises to ADLS Gen2 using AzCopy?

**Scenario:** A file server holds 10TB of historical documents (PDFs, Word, CSV). You have a 1Gbps ExpressRoute connection. AzCopy serial copy would take 22 hours. You have a 4-hour window.

```
Throughput calculation:
  1 Gbps = 125 MB/s theoretical
  Practical: ~80 MB/s over ExpressRoute (TCP overhead, headers)
  10TB ÷ 80 MB/s = 34 hours (serial)

  Parallelism with AzCopy:
  --cap-mbps 800                   (limit to 800 Mbps, leave headroom)
  --parallel-level 64              (64 concurrent file transfers)
  --block-size-mb 128              (large block = fewer round trips for big files)
  Effective: ~600 MB/s = 10TB ÷ 600 MB/s ≈ 4.6 hours ← close to window

  For guaranteed 4hr window:
  Use AzCopy --include-pattern to split by folder
  Run 3 AzCopy instances in parallel (different folders, same ER circuit)
  → ~1.8 GB/s = 10TB ÷ 1.8 GB/s ≈ 1.6 hours ✅
```

```bash
# Step 1 — Authenticate AzCopy with Managed Identity (no key needed)
# On a VM with managed identity in the same VNet as the file server:
azcopy login --identity   # Uses VM's managed identity

# Step 2 — Parallel copy with performance tuning
azcopy copy \
  "\\fileserver\documents\*" \
  "https://adlsprodcc.blob.core.windows.net/raw/documents/" \
  --recursive \
  --parallel-level 64 \        # 64 concurrent transfers
  --block-size-mb 128 \        # 128MB blocks (optimal for large files over ER)
  --cap-mbps 800 \             # Leave 200 Mbps headroom for other traffic
  --log-level INFO \
  --check-md5 FailIfDifferent  # Verify MD5 on destination after copy

# Step 3 — Run validation pass after copy
azcopy copy \
  "\\fileserver\documents\*" \
  "https://adlsprodcc.blob.core.windows.net/raw/documents/" \
  --recursive \
  --check-md5 FailIfDifferent \
  --dry-run   # Shows what would be copied — if output is empty, all files match

# Step 4 — Get job summary and check for failures
azcopy jobs list
azcopy jobs show <job-id>
# Look for: "Final Job Status: Completed" and "Number of File Transfer Failures: 0"
```

**Split across folders for maximum throughput:**
```bash
# Run these three in parallel (separate PowerShell windows or parallel jobs)
Start-Job -ScriptBlock {
    azcopy copy "\\fileserver\documents\2020-2021\" \
    "https://adlsprodcc.blob.core.windows.net/raw/documents/2020-2021/" \
    --recursive --parallel-level 32 --cap-mbps 250
}

Start-Job -ScriptBlock {
    azcopy copy "\\fileserver\documents\2022-2023\" \
    "https://adlsprodcc.blob.core.windows.net/raw/documents/2022-2023/" \
    --recursive --parallel-level 32 --cap-mbps 250
}

Start-Job -ScriptBlock {
    azcopy copy "\\fileserver\documents\2024\" \
    "https://adlsprodcc.blob.core.windows.net/raw/documents/2024/" \
    --recursive --parallel-level 32 --cap-mbps 250
}

Get-Job | Wait-Job
Get-Job | Receive-Job
```

> ⚠️ **Gotcha:** AzCopy uses the system's temp directory for job plan files. On a machine with a small C: drive, copying millions of small files generates GB of job plan data that fills the disk and crashes AzCopy mid-transfer. Set `AZCOPY_JOB_PLAN_LOCATION` and `AZCOPY_LOG_LOCATION` to a drive with sufficient space before starting large migrations.

---

## Q2. How do you use Azure Data Box for offline migration of 500TB?

**Scenario:** You have 500TB of cold archive data. Your internet/ExpressRoute connection is 1Gbps. Online transfer would take: 500TB ÷ 80 MB/s = 72 days. Compliance requires migration to Azure within 30 days.

```
Online vs Offline migration decision:

  500TB over 1Gbps ExpressRoute:
  500TB ÷ 80 MB/s effective = 72 days ← exceeds 30-day window

  Azure Data Box family:
  ┌─────────────────┬──────────┬────────────────────────────────┐
  │ Device          │ Capacity │ Use case                       │
  ├─────────────────┼──────────┼────────────────────────────────┤
  │ Data Box Disk   │ 8TB      │ Small transfers, USB-based     │
  │ Data Box        │ 80TB     │ Single appliance, 1U rack      │
  │ Data Box Heavy  │ 770TB    │ Large dataset, wheeled         │
  └─────────────────┴──────────┴────────────────────────────────┘

  For 500TB:
  Option A: 7× Data Box (80TB each) = 560TB capacity
            Ship in 2 batches = 2–3 weeks total
  Option B: 1× Data Box Heavy (770TB) = all in one shipment ✅

  Timeline:
  Day 1:   Order Data Box Heavy from Azure Portal
  Day 3:   Device delivered
  Day 3–7: Copy data (12 TB/day via 4× 1Gbps NIC)
  Day 8:   Ship back to Azure datacenter
  Day 10:  Azure ingests data to Storage Account
  Day 12:  Verify checksums, delete device data
  Total: 12 days ← well within 30-day window ✅
```

```bash
# Step 1 — Order Data Box from Azure Portal (no CLI — use portal or REST API)
az databox job create \
  --resource-group rg-migration \
  --name dbox-archive-migration \
  --location canadacentral \
  --sku DataBoxHeavy \
  --storage-account-resource-id $ADLS_ID \
  --contact-name "Abhishek Sinha" \
  --phone "416-555-0100" \
  --email-list "mail.abhisheksinha@gmail.com" \
  --street-address1 "123 Migration Ave" \
  --city "Toronto" \
  --state-or-province "ON" \
  --country "CA" \
  --postal-code "M5V1J1"

# Step 2 — Track job status
az databox job show \
  --resource-group rg-migration \
  --name dbox-archive-migration \
  --query "{status:status, deliveryDate:deliveryInfo.scheduledDeliveryTime}"

# Step 3 — After physical delivery: copy data to device
# Connect via SMB share (credentials in portal)
# From Windows: net use \\<device-ip>\<share> /u:Administrator <password>
# Copy using robocopy for reliability
robocopy "D:\archive\" "\\DataBoxIP\archivedata\" \
  /E /Z /MT:32 /R:5 /W:5 /LOG:"migration.log"

# Step 4 — Validate checksums before shipping back
# Data Box web UI: "Prepare to Ship" → generates BOM (Bill of Materials) + checksums

# Step 5 — After Azure ingestion: verify files arrived
az storage blob list \
  --account-name adlsprodcc \
  --container-name raw \
  --prefix "archive/" \
  --num-results 5 \
  --auth-mode login
```

> ⚠️ **Gotcha:** Data Box only supports **Azure Blob** and **Azure Files** as destinations — it does NOT support ADLS Gen2 hierarchical namespace natively. Copy to Blob storage first, then use AzCopy (`azcopy copy --recursive`) to move to ADLS Gen2 with HNS enabled. Plan this second transfer step in your timeline.

---

## Q3. How do you validate data integrity after a large-scale migration?

**Scenario:** ADF just finished migrating 200 million rows from SQL Server to Azure SQL. The migration log says "Completed." The data governance team asks "but is the data correct?" How do you prove it?

```
Three-tier validation approach:

  Tier 1: Row count (fast — 5 minutes)
  Source: SELECT COUNT(*) FROM transactions = 200,000,000
  Target: SELECT COUNT(*) FROM transactions = 200,000,000  ✅ or ❌

  Tier 2: Aggregate checksums (medium — 30 minutes)
  Source: SELECT SUM(Amount), MIN(date), MAX(date), COUNT(DISTINCT CustomerId)
  Target: same query — values must match exactly

  Tier 3: Statistical sampling (thorough — 2 hours)
  Random sample 10,000 rows from source
  Verify each row exists in target with identical values
  Acceptable: 0 mismatches for financial data
```

```sql
-- Tier 1: Row count comparison (run on both source and target)
-- Source (SQL Server):
SELECT 
    'transactions' AS table_name,
    COUNT(*) AS row_count,
    MIN(TransactionDate) AS min_date,
    MAX(TransactionDate) AS max_date
FROM transactions;

-- Target (Azure SQL) — identical query:
SELECT 
    'transactions' AS table_name,
    COUNT(*) AS row_count,
    MIN(TransactionDate) AS min_date,
    MAX(TransactionDate) AS max_date
FROM transactions;

-- Tier 2: Aggregate checksum
-- Source:
SELECT
    SUM(CAST(Amount AS BIGINT)) AS total_amount,
    COUNT(DISTINCT CustomerId) AS unique_customers,
    SUM(CASE WHEN Status = 'COMPLETED' THEN 1 ELSE 0 END) AS completed_count,
    CHECKSUM_AGG(CHECKSUM(TransactionId, Amount, Status)) AS row_checksum
FROM transactions;

-- Target: run identical query — all values must match exactly
```

```python
# Tier 3: Statistical sampling validation (Databricks or Python)
import pandas as pd
import pyodbc
import random

# Connect to source and target
source_conn = pyodbc.connect(SOURCE_CONNECTION_STRING)
target_conn = pyodbc.connect(TARGET_CONNECTION_STRING)

# Sample 10,000 random TransactionIds
sample_ids = pd.read_sql(
    "SELECT TOP 10000 TransactionId FROM transactions ORDER BY NEWID()",
    source_conn
)["TransactionId"].tolist()

# Compare each sampled row
mismatches = []
for tx_id in sample_ids:
    source_row = pd.read_sql(
        f"SELECT * FROM transactions WHERE TransactionId = {tx_id}",
        source_conn
    )
    target_row = pd.read_sql(
        f"SELECT * FROM transactions WHERE TransactionId = {tx_id}",
        target_conn
    )
    if not source_row.equals(target_row):
        mismatches.append(tx_id)

print(f"Checked: {len(sample_ids)} rows")
print(f"Mismatches: {len(mismatches)}")
assert len(mismatches) == 0, f"Data integrity FAILED: {mismatches[:5]}"
```

```bash
# For file migrations: verify via AzCopy with MD5 check
azcopy copy \
  "https://adlsprodcc.blob.core.windows.net/raw/documents/*" \
  "https://adlssecondary.blob.core.windows.net/raw/documents/*" \
  --check-md5 FailIfDifferent \
  --dry-run        # Dry run shows mismatches without copying
# Output: "0 files would be copied" = all files identical ✅
# Output: "23 files would be copied" = 23 files differ/missing ❌
```

> ⚠️ **Gotcha:** SQL `COUNT(*)` matches do not guarantee correct data — they only confirm row count. A migration can produce the correct row count but with truncated values (e.g., string `"truncated"` instead of `"full text 4096 chars"`) due to data type mismatches. Always include Tier 2 aggregate checksums AND Tier 3 sampling for financial data migrations.

---

## Q4. How do you perform a zero-downtime cutover for a critical database migration?

**Scenario:** The migration is complete, CDC sync lag is 0 seconds. It's 2am Sunday — your 2-hour cutover window starts now. 500 app instances need to switch from on-prem SQL Server to Azure SQL. How do you execute this without any requests failing?

```
Zero-downtime cutover sequence:

  T-10min: Pre-warm Azure SQL
           - Run DBCC FREEPROCCACHE on Azure SQL (clear any stale plans)
           - Warm connection pools: send 10 test queries from each region

  T-5min:  Enable maintenance mode on API Gateway (APIM policy)
           - APIM returns 503 "scheduled maintenance" for new requests
           - In-flight requests complete (drain timeout = 5min)

  T-0:     Stop new writes to source
           - Verify DMS lag = 0 (source and target in sync)
           - Wait max 30s for in-flight transactions

  T+1:     Validate Azure SQL is current
           - Run row count + MAX(LastModifiedDate) comparison
           - Automated script: exits 0 = OK, exits 1 = DO NOT CUT OVER

  T+3:     Update connection strings
           - Azure App Configuration / Key Vault: update connection string
           - AKS pods: rolling restart (picks up new Key Vault value)
           - APIM: disable maintenance mode

  T+8:     Smoke test
           - 20 automated test transactions against Azure SQL
           - Monitor: error rate, latency, DMS shows "Migration Complete"

  T+30:    Keep source read-only for 72hr rollback window
           - If issues found: revert Key Vault connection string, restart pods
```

```bash
# T-5min: Enable maintenance mode in APIM (custom policy)
az apim api policy set \
  --service-name apim-prod-cc \
  --resource-group rg-prod \
  --api-id payments-api \
  --policy-format xml \
  --value '<policies><inbound><return-response><status code="503" reason="Scheduled Maintenance"/></return-response></inbound></policies>'

# T-0: Verify CDC lag is 0 before cutover
az dms project task show \
  --service-name dms-prod-cc \
  --resource-group rg-data \
  --project-name proj-payments \
  --task-name task-cdc \
  --query "properties.output[0].latency"
# Expected: "0"

# T+1: Run automated validation
python validate_migration.py \
  --source "Server=sql-prod.corp.internal;Database=payments;..." \
  --target "Server=sql-prod-cc.database.windows.net;Database=payments;..."
# Exit code 0 = OK to cut over

# T+3: Update Key Vault with Azure SQL connection string (apps auto-rotate)
az keyvault secret set \
  --vault-name kv-prod \
  --name "sql-connection-string" \
  --value "Server=sql-prod-cc.database.windows.net;Database=payments;Authentication=Active Directory Managed Identity"

# T+3: Disable maintenance mode
az apim api policy delete \
  --service-name apim-prod-cc \
  --resource-group rg-prod \
  --api-id payments-api

# T+3: Rolling restart AKS pods to pick up new Key Vault secret
kubectl rollout restart deployment/payment-api -n production
kubectl rollout status deployment/payment-api -n production --watch
```

> ⚠️ **Gotcha:** Connection string updates via Azure App Configuration or Key Vault are NOT instant for running pods. AKS pods cache the connection string at startup. A rolling restart is required — plan for 5–10 minutes for all pods to restart and reconnect. During this window, reduce the connection pool size on old pods to avoid overwhelming Azure SQL with both old and new connections simultaneously.
