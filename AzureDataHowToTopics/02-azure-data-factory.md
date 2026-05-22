# Azure Data Factory
### 🟡 Intermediate → 🔴 Advanced

> *"ADF is not a data transformation engine.  
>  It is an orchestration engine. Confuse the two and your pipelines become unmaintainable."*

---

## Q1. How do you build an incremental copy pipeline using the watermark pattern?

**Scenario:** A SQL Server on-premises has a `transactions` table with 500M rows. A full copy takes 6 hours and runs nightly. You need to switch to incremental — copy only rows added or changed since the last run.

```
Watermark Pattern:

  SQL Server (on-prem)              ADLS Gen2 (Azure)
  ─────────────────────             ────────────────────────
  transactions table                raw/transactions/
  LastModifiedDate column           2024-01-15_batch.parquet
       │
       │  WHERE LastModifiedDate > @last_watermark
       │  AND   LastModifiedDate <= @new_watermark
       ▼
  ADF Copy Activity
       │ copies delta rows only
       ▼
  Update watermark table in Azure SQL
  watermarks | pipeline_name | last_watermark_value
  "transactions_copy" | 2024-01-15 02:00:00

  Next run:
  @last_watermark = 2024-01-15 02:00:00
  @new_watermark  = current UTC time
```

**ADF Pipeline JSON (key activities):**
```json
{
  "activities": [
    {
      "name": "LookupOldWatermark",
      "type": "Lookup",
      "typeProperties": {
        "source": {
          "type": "AzureSqlSource",
          "sqlReaderQuery": "SELECT last_watermark FROM watermarks WHERE pipeline_name = 'transactions_copy'"
        }
      }
    },
    {
      "name": "LookupNewWatermark",
      "type": "Lookup",
      "typeProperties": {
        "source": {
          "type": "SqlServerSource",
          "sqlReaderQuery": "SELECT MAX(LastModifiedDate) AS new_watermark FROM transactions"
        }
      }
    },
    {
      "name": "CopyDelta",
      "type": "Copy",
      "inputs": [{"referenceName": "SqlServerTransactions"}],
      "outputs": [{"referenceName": "AdlsParquet"}],
      "typeProperties": {
        "source": {
          "type": "SqlServerSource",
          "sqlReaderQuery": {
            "value": "SELECT * FROM transactions WHERE LastModifiedDate > '@{activity('LookupOldWatermark').output.firstRow.last_watermark}' AND LastModifiedDate <= '@{activity('LookupNewWatermark').output.firstRow.new_watermark}'",
            "type": "Expression"
          }
        }
      }
    },
    {
      "name": "UpdateWatermark",
      "type": "SqlServerStoredProcedure",
      "typeProperties": {
        "storedProcedureName": "sp_update_watermark",
        "storedProcedureParameters": {
          "pipeline_name": {"value": "transactions_copy"},
          "new_watermark": {"value": "@{activity('LookupNewWatermark').output.firstRow.new_watermark}"}
        }
      }
    }
  ]
}
```

```bash
# Trigger pipeline and monitor run
az datafactory pipeline create-run \
  --factory-name adf-prod-cc \
  --resource-group rg-data \
  --pipeline-name PL_Incremental_Transactions

# Monitor pipeline run status
az datafactory pipeline-run query-by-factory \
  --factory-name adf-prod-cc \
  --resource-group rg-data \
  --last-updated-after "2024-01-15T00:00:00" \
  --last-updated-before "2024-01-16T00:00:00" \
  --output table
```

> ⚠️ **Gotcha:** The watermark column must be indexed on the source table. A WHERE clause on an unindexed `LastModifiedDate` column performs a full table scan on 500M rows — the incremental copy takes longer than the full copy. Confirm the index exists before switching to watermark mode.

> 💡 **Deep dive hint:** ADF Change Data Capture (CDC) using SQL Server Change Tracking — instead of a timestamp watermark, SQL Server CDC tracks every row insert/update/delete at the log level, giving true change capture without relying on application-set timestamps.

---

## Q2. How do you configure a Self-Hosted Integration Runtime for on-premises data?

**Scenario:** Your on-prem SQL Server is behind a corporate firewall with no inbound port access. ADF needs to connect to it. The network team says "no inbound firewall rules."

```
Self-Hosted IR — outbound-only connection:

  Corporate Network (on-prem)
  ┌──────────────────────────────────────────────────────┐
  │  SHIR machine (Windows Server)                        │
  │  ├── Outbound HTTPS 443 to *.servicebus.windows.net  │
  │  ├── Outbound HTTPS 443 to *.frontend.clouddatahub   │
  │  └── SQL Server access (same network, port 1433)     │
  │       ↑ No inbound ports required                    │
  └─────────────────────┬────────────────────────────────┘
                        │ HTTPS 443 outbound
                        ▼
  Azure Service Bus relay (Azure backbone)
                        │
                        ▼
  Azure Data Factory (adf-prod-cc)
  Linked Service → SHIR → SQL Server
```

```bash
# Step 1 — Create SHIR in ADF
az datafactory integration-runtime create \
  --factory-name adf-prod-cc \
  --resource-group rg-data \
  --name shir-onprem-sql \
  --properties '{
    "type": "SelfHosted",
    "description": "On-premises SQL Server SHIR"
  }'

# Step 2 — Get auth key to register the SHIR machine
az datafactory integration-runtime list-auth-key \
  --factory-name adf-prod-cc \
  --resource-group rg-data \
  --name shir-onprem-sql
# Returns: authKey1 and authKey2 — copy one

# Step 3 — On the SHIR Windows machine:
# Download SHIR installer from Microsoft and install
# Open Microsoft Integration Runtime Configuration Manager
# Paste the auth key from Step 2 → "Register"
# Status: Connected ✅

# Step 4 — Create Linked Service pointing to SHIR
az datafactory linked-service create \
  --factory-name adf-prod-cc \
  --resource-group rg-data \
  --name ls-onprem-sql \
  --properties '{
    "type": "SqlServer",
    "typeProperties": {
      "connectionString": "Server=sql-prod.corp.internal;Database=transactions;Integrated Security=False;User ID=svc-adf;",
      "password": {
        "type": "AzureKeyVaultSecret",
        "store": {"referenceName": "ls-keyvault", "type": "LinkedServiceReference"},
        "secretName": "sql-svc-adf-password"
      }
    },
    "connectVia": {
      "referenceName": "shir-onprem-sql",
      "type": "IntegrationRuntimeReference"
    }
  }'
```

**SHIR high availability — two nodes:**
```bash
# Install SHIR on a second machine and register with the same auth key
# ADF automatically load-balances and fails over between nodes
# Check node status:
az datafactory integration-runtime get-status \
  --factory-name adf-prod-cc \
  --resource-group rg-data \
  --name shir-onprem-sql \
  --query "properties.nodes[].{name:nodeName, status:status, version:version}"
```

> ⚠️ **Gotcha:** SHIR machines must allow outbound HTTPS to `*.servicebus.windows.net` on port 443. Corporate proxies that perform TLS inspection break the Service Bus relay connection — the SHIR appears registered but copy activities fail with `Connection forcibly closed`. Bypass SHIR machine traffic from TLS inspection.

> 💡 **Deep dive hint:** SHIR performance tuning — `parallelCopies` and `dataIntegrationUnits` (DIUs) only apply to cloud-to-cloud copies. For SHIR copies, performance is bounded by the SHIR machine's CPU, memory, and network bandwidth. Use `maxConcurrentConnections` and schedule multiple SHIR VMs for parallelism.

---

## Q3. How do you validate data integrity with checksums in ADF copy pipelines?

**Scenario:** ADF copies 50GB files from on-prem blob storage to ADLS Gen2. You need proof that every bit was transferred without corruption — for audit purposes. How do you implement end-to-end checksum validation?

```
Checksum validation flow:

  Source (on-prem / blob)          ADF Copy Activity          Destination (ADLS)
  ──────────────────────           ─────────────────          ──────────────────
  file.csv                    → copy with MD5 enabled →      file.csv
  MD5: abc123...                                              MD5: abc123...
       │                                                           │
       └─────────── compare ──────────────────────────────────────┘
                     ✅ match = data integrity confirmed
                     ❌ mismatch = copy failed silently → alert
```

```json
{
  "name": "CopyWithChecksumValidation",
  "type": "Copy",
  "typeProperties": {
    "source": {
      "type": "BlobSource"
    },
    "sink": {
      "type": "AzureBlobFSSink"
    },
    "enableStaging": false,
    "validateDataConsistency": true,
    "logSettings": {
      "enableCopyActivityLog": true,
      "copyActivityLogSettings": {
        "logLevel": "Warning",
        "enableReliableLogging": true
      },
      "logLocationSettings": {
        "linkedServiceName": {"referenceName": "ls-adls-logs"},
        "path": "audit/copy-logs"
      }
    }
  }
}
```

```bash
# After copy: verify MD5 of source and destination match
# Source file MD5 (Azure Blob):
az storage blob show \
  --name "transactions/2024/01/15.csv" \
  --container-name source \
  --account-name storagesource \
  --query "properties.contentMd5" \
  --auth-mode login

# Destination file MD5 (ADLS Gen2):
az storage blob show \
  --name "raw/transactions/2024/01/15.csv" \
  --container-name raw \
  --account-name adlsprodcc \
  --query "properties.contentMd5" \
  --auth-mode login

# Expected: both values identical
# If source has no MD5 (common for files uploaded without MD5):
# ADF computes MD5 during copy and stores it on the destination blob
```

**ADF checksum for large file batches (Web Activity to trigger validation):**
```python
# Databricks notebook called after ADF copy (via Notebook Activity)
# Validates row counts between source and destination
source_count = spark.read.parquet("wasbs://source/transactions/2024/01/15/").count()
dest_count   = spark.read.parquet("abfss://raw@adlsprodcc.dfs.core.windows.net/transactions/2024/01/15/").count()

assert source_count == dest_count, \
    f"Row count mismatch: source={source_count}, dest={dest_count}"
```

> ⚠️ **Gotcha:** `validateDataConsistency: true` in ADF only works for binary-identical copies (file-to-file). It does not validate row counts or schema for SQL-to-Parquet copies where ADF converts data types. For SQL sources, you need a separate row-count validation activity after the copy.

---

## Q4. How do you implement Change Data Capture (CDC) for a near-zero-downtime database migration?

**Scenario:** You need to migrate a 200GB SQL Server database to Azure SQL with less than 30 minutes downtime. Full backup/restore takes 4 hours. You need the source to keep accepting writes during migration.

```
CDC Migration flow — three phases:

  Phase 1: Initial Load (hours)
  Source DB → DMS full migration → Target DB (Azure SQL)
  Source: still accepting writes
  Target: receives snapshot (point-in-time)

  Phase 2: CDC Sync (continuous, until cutover)
  SQL Server transaction log
       │ DMS reads change events (INSERT/UPDATE/DELETE)
       ▼
  Target Azure SQL: applies changes
  Replication lag: typically <1s

  Phase 3: Cutover (< 30 minutes)
  1. Drain connections to source (app maintenance mode)
  2. Wait for DMS lag = 0 (source and target in sync)
  3. Redirect app connection strings to Azure SQL
  4. Re-enable writes on target
  5. Verify app working ✅
  6. Keep source read-only for 72h rollback window
```

```bash
# Step 1 — Enable Change Tracking on source database (required by DMS)
# Run on source SQL Server:
# ALTER DATABASE [transactions_db] SET CHANGE_TRACKING = ON
# (CHANGE_RETENTION = 3 DAYS, AUTO_CLEANUP = ON)
# ALTER TABLE [transactions] ENABLE CHANGE_TRACKING

# Step 2 — Create DMS migration project
az dms project create \
  --service-name dms-prod-cc \
  --resource-group rg-data \
  --name proj-sql-migration \
  --source-platform SQL \
  --target-platform SQLDB \
  --location canadacentral

# Step 3 — Create online migration task (CDC enabled)
az dms project task create \
  --service-name dms-prod-cc \
  --resource-group rg-data \
  --project-name proj-sql-migration \
  --task-name task-transactions-cdc \
  --task-type MigrateSqlServerSqlDb \
  --source-connection-json @source-connection.json \
  --target-connection-json @target-connection.json \
  --database-options-json @migration-options.json

# Step 4 — Monitor replication lag
az dms project task show \
  --service-name dms-prod-cc \
  --resource-group rg-data \
  --project-name proj-sql-migration \
  --task-name task-transactions-cdc \
  --query "properties.output[0].migrationState"
# Expected: "READY_TO_COMPLETE" means lag = 0

# Step 5 — Initiate cutover
az dms project task cutover \
  --service-name dms-prod-cc \
  --resource-group rg-data \
  --project-name proj-sql-migration \
  --task-name task-transactions-cdc \
  --object-name transactions_db
```

> ⚠️ **Gotcha:** DMS online migration requires the source SQL Server to have the CDC Agent running AND sufficient transaction log disk space (at least 3× the log growth rate × migration duration). If the log fills during migration, DMS aborts and you must restart from scratch. Pre-size the source log disk before starting.
