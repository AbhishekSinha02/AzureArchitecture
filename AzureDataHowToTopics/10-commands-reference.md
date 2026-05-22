# Azure Data Commands Reference
### All az / AzCopy / SQL / KQL / Python commands with context

> *"Know the command. Know why you're running it. Know what the output means."*

---

## ADLS Gen2 / Storage

```bash
# Create ADLS Gen2 storage account
az storage account create \
  --name adlsprodcc \
  --resource-group rg-data \
  --location canadacentral \
  --sku Standard_LRS \
  --kind StorageV2 \
  --hierarchical-namespace true \   # Required for ADLS Gen2
  --min-tls-version TLS1_2 \
  --allow-blob-public-access false

# Create filesystem (container) in ADLS Gen2
az storage fs create \
  --name raw \
  --account-name adlsprodcc \
  --auth-mode login

# Create directory in filesystem
az storage fs directory create \
  --name transactions/2024 \
  --file-system raw \
  --account-name adlsprodcc \
  --auth-mode login

# List files in a path
az storage fs file list \
  --path raw/transactions/2024 \
  --file-system raw \
  --account-name adlsprodcc \
  --auth-mode login \
  --output table

# Check effective ACL on a path
az storage fs access show \
  --path "raw/transactions" \
  --file-system raw \
  --account-name adlsprodcc \
  --auth-mode login

# Set ACL on a directory (recursive)
az storage fs access set \
  --acl "user:<object-id>:rwx,default:user:<object-id>:rwx" \
  --path "raw/transactions" \
  --file-system raw \
  --account-name adlsprodcc \
  --auth-mode login

# Check storage account networking (public access enabled/disabled)
az storage account show \
  --name adlsprodcc \
  --resource-group rg-data \
  --query "{publicAccess:publicNetworkAccess, allowBlobPublic:allowBlobPublicAccess}"

# View lifecycle management policy
az storage account management-policy show \
  --account-name adlsprodcc \
  --resource-group rg-data

# Get blob MD5 checksum
az storage blob show \
  --name "raw/transactions/file.parquet" \
  --container-name raw \
  --account-name adlsprodcc \
  --query "properties.contentMd5" \
  --auth-mode login

# Rehydrate archived blob
az storage blob set-tier \
  --name "raw/archive/old-file.json" \
  --container-name raw \
  --account-name adlsprodcc \
  --tier Cool \
  --rehydrate-priority High   # 1 hour; Standard = up to 15 hours
```

---

## AzCopy

```bash
# Authenticate with managed identity (recommended — no keys)
azcopy login --identity

# Copy folder from on-prem to ADLS (recursive, parallel)
azcopy copy \
  "\\fileserver\documents\" \
  "https://adlsprodcc.blob.core.windows.net/raw/documents/" \
  --recursive \
  --parallel-level 64 \
  --block-size-mb 128 \
  --cap-mbps 800 \
  --check-md5 FailIfDifferent

# Copy between two Azure Storage accounts
azcopy copy \
  "https://source.blob.core.windows.net/container/path?<SAS>" \
  "https://dest.blob.core.windows.net/container/path?<SAS>" \
  --recursive \
  --check-md5 FailIfDifferent

# Sync (only copies new/modified files — idempotent)
azcopy sync \
  "\\fileserver\documents\" \
  "https://adlsprodcc.blob.core.windows.net/raw/documents/" \
  --recursive \
  --delete-destination false   # Don't delete extra files in destination

# List jobs
azcopy jobs list

# Show job details and transfer stats
azcopy jobs show <job-id>
# Look for: "Number of File Transfer Failures: 0"

# Dry-run to check what would be copied
azcopy copy "source" "dest" --recursive --dry-run

# Remove files matching a pattern
azcopy remove \
  "https://adlsprodcc.blob.core.windows.net/raw/temp/" \
  --recursive \
  --include-pattern "*.tmp"
```

---

## Azure Data Factory

```bash
# List pipelines in a factory
az datafactory pipeline list \
  --factory-name adf-prod-cc \
  --resource-group rg-data \
  --output table

# Trigger a pipeline run
az datafactory pipeline create-run \
  --factory-name adf-prod-cc \
  --resource-group rg-data \
  --pipeline-name PL_Incremental_Transactions

# Monitor pipeline runs (last 24 hours)
az datafactory pipeline-run query-by-factory \
  --factory-name adf-prod-cc \
  --resource-group rg-data \
  --last-updated-after "$(date -u -d '24 hours ago' +%Y-%m-%dT%H:%M:%S)" \
  --last-updated-before "$(date -u +%Y-%m-%dT%H:%M:%S)" \
  --output table

# Get activity run details for a pipeline run
az datafactory activity-run query-by-pipeline-run \
  --factory-name adf-prod-cc \
  --resource-group rg-data \
  --run-id <pipeline-run-id> \
  --last-updated-after "2024-01-01T00:00:00" \
  --output table

# Create SHIR integration runtime
az datafactory integration-runtime create \
  --factory-name adf-prod-cc \
  --resource-group rg-data \
  --name shir-onprem \
  --properties '{"type":"SelfHosted"}'

# Get SHIR auth key (for registering the runtime on SHIR machine)
az datafactory integration-runtime list-auth-key \
  --factory-name adf-prod-cc \
  --resource-group rg-data \
  --name shir-onprem

# Get SHIR status and node info
az datafactory integration-runtime get-status \
  --factory-name adf-prod-cc \
  --resource-group rg-data \
  --name shir-onprem \
  --query "properties.nodes[].{name:nodeName, status:status}"
```

---

## Azure Synapse Analytics

```bash
# Create Synapse workspace
az synapse workspace create \
  --name synapse-prod-cc \
  --resource-group rg-data \
  --location canadacentral \
  --storage-account adlsprodcc \
  --file-system synapse \
  --sql-admin-login-user sqladmin \
  --sql-admin-login-password "<PASSWORD>"

# Create Dedicated SQL Pool
az synapse sql pool create \
  --name sqlpool-prod \
  --workspace-name synapse-prod-cc \
  --resource-group rg-data \
  --performance-level DW500c

# Pause / Resume Dedicated SQL Pool
az synapse sql pool pause \
  --name sqlpool-prod \
  --workspace-name synapse-prod-cc \
  --resource-group rg-data

az synapse sql pool resume \
  --name sqlpool-prod \
  --workspace-name synapse-prod-cc \
  --resource-group rg-data

# Scale Dedicated SQL Pool
az synapse sql pool update \
  --name sqlpool-prod \
  --workspace-name synapse-prod-cc \
  --resource-group rg-data \
  --performance-level DW1000c

# Create Spark Pool
az synapse spark pool create \
  --name spProdMedium \
  --workspace-name synapse-prod-cc \
  --resource-group rg-data \
  --node-count 3 \
  --node-size Medium \
  --min-node-count 3 \
  --max-node-count 10 \
  --enable-auto-scale true \
  --delay 15 \
  --enable-auto-pause true \
  --spark-version "3.4"

# Check SQL Pool status
az synapse sql pool show \
  --name sqlpool-prod \
  --workspace-name synapse-prod-cc \
  --resource-group rg-data \
  --query "status"
```

---

## CosmosDB

```bash
# Create CosmosDB account (multi-region, zone redundant)
az cosmosdb create \
  --name cosmos-prod-cc \
  --resource-group rg-data \
  --locations regionName=canadacentral failoverPriority=0 isZoneRedundant=true \
  --locations regionName=canadaeast failoverPriority=1 isZoneRedundant=false \
  --default-consistency-level Session \
  --enable-automatic-failover true

# Create database
az cosmosdb sql database create \
  --account-name cosmos-prod-cc \
  --resource-group rg-data \
  --name banking

# Create container with autoscale
az cosmosdb sql container create \
  --account-name cosmos-prod-cc \
  --resource-group rg-data \
  --database-name banking \
  --name transactions \
  --partition-key-path "/partitionKey" \
  --max-throughput 40000     # Autoscale 4000–40000 RU

# Check RU consumption metrics
az monitor metrics list \
  --resource "/subscriptions/<SUB>/resourceGroups/rg-data/providers/Microsoft.DocumentDB/databaseAccounts/cosmos-prod-cc" \
  --metric "TotalRequestUnits" \
  --interval PT1M \
  --output table

# Manual failover (for DR testing)
az cosmosdb failover-priority-change \
  --name cosmos-prod-cc \
  --resource-group rg-data \
  --failover-policies canadaeast=0 canadacentral=1

# Enable multi-region writes
az cosmosdb update \
  --name cosmos-prod-cc \
  --resource-group rg-data \
  --enable-multiple-write-locations true
```

---

## Event Hubs

```bash
# Create Event Hubs namespace (Premium)
az eventhubs namespace create \
  --name eh-ns-prod \
  --resource-group rg-data \
  --location canadacentral \
  --sku Premium \
  --capacity 4

# Create event hub
az eventhubs eventhub create \
  --name eh-transactions \
  --namespace-name eh-ns-prod \
  --resource-group rg-data \
  --partition-count 32 \
  --retention-time-in-hours 168

# Create consumer group
az eventhubs consumer-group create \
  --name cg-databricks \
  --eventhub-name eh-transactions \
  --namespace-name eh-ns-prod \
  --resource-group rg-data

# List consumer groups
az eventhubs consumer-group list \
  --eventhub-name eh-transactions \
  --namespace-name eh-ns-prod \
  --resource-group rg-data \
  --output table

# Enable Capture
az eventhubs eventhub update \
  --name eh-transactions \
  --namespace-name eh-ns-prod \
  --resource-group rg-data \
  --enable-capture true \
  --capture-interval 300 \
  --capture-size-limit 314572800 \
  --destination-name "EventHubArchive.AzureBlockBlob" \
  --storage-account $STORAGE_ID \
  --blob-container raw \
  --archive-name-format "{Namespace}/{EventHub}/{PartitionId}/{Year}/{Month}/{Day}/{Hour}/{Minute}/{Second}"

# Get connection string for Kafka clients
az eventhubs namespace authorization-rule keys list \
  --namespace-name eh-ns-prod \
  --resource-group rg-data \
  --name RootManageSharedAccessKey \
  --query primaryConnectionString
```

---

## Azure SQL Database

```bash
# Create SQL Server
az sql server create \
  --name sql-prod-cc \
  --resource-group rg-data \
  --location canadacentral \
  --admin-user sqladmin \
  --admin-password "<PASSWORD>"

# Create database (Business Critical, zone redundant)
az sql db create \
  --server sql-prod-cc \
  --resource-group rg-data \
  --name payments-db \
  --edition BusinessCritical \
  --service-objective BC_Gen5_4 \
  --zone-redundant true \
  --backup-storage-redundancy Geo

# Create elastic pool
az sql elastic-pool create \
  --server sql-prod-cc \
  --resource-group rg-data \
  --name ep-tenants \
  --edition GeneralPurpose \
  --capacity 16 \
  --db-min-capacity 0 \
  --db-max-capacity 4

# Move database to elastic pool
az sql db update \
  --server sql-prod-cc \
  --resource-group rg-data \
  --name tenant-001-db \
  --elastic-pool ep-tenants

# Create failover group
az sql failover-group create \
  --name fg-payments \
  --server sql-prod-cc \
  --resource-group rg-data \
  --partner-server sql-dr-ce \
  --add-db payments-db \
  --failover-policy Automatic \
  --grace-period 1

# Manual failover (for DR testing)
az sql failover-group set-primary \
  --name fg-payments \
  --server sql-dr-ce \
  --resource-group rg-data
```

---

## Azure Cache for Redis

```bash
# Create Premium Redis cache
az redis create \
  --name redis-prod-cc \
  --resource-group rg-data \
  --location canadacentral \
  --sku Premium \
  --vm-size P2 \
  --enable-non-ssl-port false \
  --minimum-tls-version 1.2

# Get access keys
az redis list-keys \
  --name redis-prod-cc \
  --resource-group rg-data

# Update Redis configuration (change eviction policy)
az redis update \
  --name redis-prod-cc \
  --resource-group rg-data \
  --redis-configuration '{"maxmemory-policy":"volatile-lru"}'

# Check cache size and eviction metrics
az monitor metrics list \
  --resource "/subscriptions/<SUB>/resourceGroups/rg-data/providers/Microsoft.Cache/Redis/redis-prod-cc" \
  --metric "usedmemory,evictedkeys,cachehits,cachemisses" \
  --interval PT5M \
  --output table

# Enable geo-replication (Premium+)
az redis geo-replication link \
  --name redis-prod-cc \
  --resource-group rg-data \
  --server-to-link redis-dr-ce
```

---

## Key SQL Queries for Data Operations

```sql
-- Synapse: load data with COPY INTO
COPY INTO dbo.transactions
FROM 'https://adlsprodcc.dfs.core.windows.net/curated/transactions/*.parquet'
WITH (FILE_TYPE = 'PARQUET', CREDENTIAL = (IDENTITY = 'Managed Identity'));

-- Synapse Serverless: query Parquet directly from ADLS
SELECT COUNT(*), SUM(Amount)
FROM OPENROWSET(
    BULK 'https://adlsprodcc.dfs.core.windows.net/curated/transactions/**',
    FORMAT = 'PARQUET'
) AS txn
WHERE YEAR(TransactionDate) = 2024;

-- Azure SQL: check DTU/vCore utilisation
SELECT
    end_time,
    avg_cpu_percent,
    avg_data_io_percent,
    avg_log_write_percent,
    avg_memory_usage_percent
FROM sys.dm_db_resource_stats
ORDER BY end_time DESC;

-- Azure SQL: find top resource-consuming queries
SELECT TOP 10
    qs.total_elapsed_time / qs.execution_count AS avg_elapsed_time,
    qs.execution_count,
    SUBSTRING(qt.text, (qs.statement_start_offset/2) + 1,
        ((CASE qs.statement_end_offset WHEN -1 THEN DATALENGTH(qt.text)
          ELSE qs.statement_end_offset END - qs.statement_start_offset)/2) + 1) AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
ORDER BY avg_elapsed_time DESC;

-- CosmosDB: check RU consumption per operation (in results metadata)
-- Python: response = container.read_item(...)
-- print(response.headers["x-ms-request-charge"])  -- RUs consumed

-- Check replication lag for SQL Failover Group
SELECT
    ag_name,
    db_name,
    synchronization_health_desc,
    last_commit_time,
    DATEDIFF(SECOND, last_commit_time, GETUTCDATE()) AS lag_seconds
FROM sys.dm_hadr_database_replica_states
JOIN sys.availability_groups ON group_id = group_id;
```

---

## KQL Queries for Data Monitoring

```kusto
// ADF pipeline failure rate (last 7 days)
AzureDiagnostics
| where ResourceType == "FACTORIES/PIPELINES"
| where status_s in ("Succeeded", "Failed")
| where TimeGenerated > ago(7d)
| summarize
    Total = count(),
    Failed = countif(status_s == "Failed")
  by bin(TimeGenerated, 1h), pipelineName_s
| extend FailureRate = (Failed * 100.0) / Total
| where FailureRate > 0
| order by TimeGenerated desc

// CosmosDB throttled requests (HTTP 429 — RU exhausted)
AzureDiagnostics
| where ResourceType == "DATABASEACCOUNTS"
| where statusCode_s == "429"
| where TimeGenerated > ago(1h)
| summarize ThrottledRequests = count() by bin(TimeGenerated, 5m), collectionName_s
| order by TimeGenerated desc

// Event Hubs incoming messages per hour
AzureMetrics
| where ResourceProvider == "MICROSOFT.EVENTHUB"
| where MetricName == "IncomingMessages"
| where TimeGenerated > ago(24h)
| summarize TotalMessages = sum(Total) by bin(TimeGenerated, 1h)
| render timechart

// SQL Database deadlocks in last 24 hours
AzureDiagnostics
| where ResourceType == "SERVERS/DATABASES"
| where Category == "Deadlocks"
| where TimeGenerated > ago(24h)
| project TimeGenerated, DatabaseName = Resource, deadlock_xml_s
| order by TimeGenerated desc

// ADLS Gen2 failed reads (4xx errors)
StorageBlobLogs
| where TimeGenerated > ago(1h)
| where StatusCode >= 400 and StatusCode < 500
| summarize FailedRequests = count() by bin(TimeGenerated, 5m), Uri, StatusCode
| order by FailedRequests desc
```
