# Azure Data Factory — Interview Questions

> Covers: ADF architecture, pipeline design, Self-hosted IR, checksums, data validation, CDC, incremental patterns.
> Levels: Beginner → Intermediate → Advanced

---

## BEGINNER

**Q1. What is Azure Data Factory and what are its core components?**

**Answer:**

Azure Data Factory (ADF) is Azure's cloud ETL/ELT service for data integration at scale.

**Core components:**

| Component | Purpose |
|-----------|---------|
| **Pipelines** | Container for activities; the unit of execution and scheduling |
| **Activities** | Steps in a pipeline (Copy, Data Flow, Execute Pipeline, Web, Wait...) |
| **Datasets** | Named reference to data (file in ADLS, table in SQL) — defines schema + location |
| **Linked Services** | Connection definitions (credentials, connection strings, endpoints) |
| **Integration Runtime (IR)** | Compute that executes activities — three types (see Q2) |
| **Triggers** | Schedule, event (blob created), tumbling window, custom event |
| **Data Flows** | Visual code-free data transformation (Spark under the hood) |

**Typical pipeline structure:**
```
Trigger (schedule: daily 2 AM)
  │
  └─► Pipeline: IngestSalesData
        │
        ├─► Activity: ValidateSourceExists (Get Metadata)
        ├─► Activity: CopyFromSQLtoADLS (Copy Activity)
        ├─► Activity: TransformWithDataFlow (Mapping Data Flow)
        └─► Activity: NotifyOnSuccess (Logic App / Email)
```

---

**Q2. What are the three types of Integration Runtime in ADF?**

**Answer:**

| IR Type | Where it runs | Use case |
|---------|--------------|----------|
| **Azure IR** | Managed by Azure in Azure datacenters | Azure-to-Azure transfers, public endpoint sources/sinks |
| **Self-hosted IR (SHIR)** | Customer-managed VM or server (on-prem or IaaS) | On-premises data sources, private network access |
| **Azure-SSIS IR** | Managed Azure cluster | Running SSIS packages in ADF (lift-and-shift SSIS) |

**Self-hosted IR deep dive (most common exam/interview topic):**

```
On-premises SQL Server → SHIR → ADF pipeline → ADLS Gen2

Requirements:
  - Windows Server 2012+ or Windows 10+
  - .NET Framework 4.7.2+
  - 4+ cores, 8+ GB RAM recommended
  - Outbound HTTPS (port 443) to Azure (no inbound firewall rules needed)
  - SHIR downloads jobs, executes locally, sends results back

High availability:
  - Install SHIR on 2+ machines, register to same logical SHIR
  - ADF automatically load balances across nodes
  - One node down = no downtime (failover to other node)

SHIR auto-update:
  ADF pushes SHIR updates automatically
  Use maintenance windows if controlled updates needed
```

---

**Q3. How does ADF schedule pipelines and what trigger types are available?**

**Answer:**

| Trigger type | When it fires | Use case |
|-------------|--------------|---------|
| **Schedule** | Fixed time/recurrence (cron-like) | Daily ETL, nightly reporting |
| **Tumbling Window** | Non-overlapping time windows, sequential, backfill-capable | Hourly data loads with guaranteed ordering |
| **Event-based (Storage)** | Blob created / deleted in Azure Storage | File arrival triggers processing |
| **Custom Event** | Azure Event Grid custom topic | External system signals ADF |
| **Manual** | API call or portal button | On-demand runs, testing |

**Schedule trigger example:**
```json
{
  "type": "ScheduleTrigger",
  "typeProperties": {
    "recurrence": {
      "frequency": "Day",
      "interval": 1,
      "startTime": "2024-01-01T02:00:00Z",
      "timeZone": "Eastern Standard Time",
      "schedule": {
        "hours": [2],
        "minutes": [0]
      }
    }
  }
}
```

**Tumbling Window advantage over Schedule:**
- Guarantees sequential execution (window N+1 doesn't run until window N completes)
- Built-in backfill: if pipeline was down for 3 days, it will catch up all missed windows
- Window parameters automatically passed to pipeline: `@trigger().outputs.windowStartTime`

---

## INTERMEDIATE

**Q4. How do you implement data validation with checksums in ADF pipelines?**

**Answer:**

**Why checksums in ADF pipelines:**
- Detect file corruption during transfer
- Verify source and target are byte-for-byte identical
- Required for regulated migrations (OSFI, PIPEDA)

**Approach 1: ADF built-in data consistency validation**

```json
// In Copy Activity settings:
{
  "enableStaging": false,
  "validateDataConsistency": true,
  "dataConsistencyVerification": {
    "level": "bestEffort"  // or "fatal" — stop on mismatch
  }
}
```
ADF computes MD5 of source file and verifies against the MD5 returned by the target. Works for Azure Blob → Azure Blob. For file systems, ADF uses file size + last modified time by default.

**Approach 2: Custom checksum pipeline (production-grade)**

```
Pipeline: CopyWithChecksum

Activity 1: GenerateSourceChecksum
  Type: Custom Activity (runs on Azure Batch or via SHIR)
  Script: python generate_checksums.py --path /source/data
  Output: source_checksums.json → written to ADLS staging

Activity 2: CopyFiles
  Type: Copy Activity
  Source: On-prem NAS via SHIR
  Sink: ADLS Gen2
  validateDataConsistency: true

Activity 3: GenerateTargetChecksum
  Type: Custom Activity
  Script: az storage blob list ... | extract MD5 headers
  Output: target_checksums.json

Activity 4: CompareChecksums
  Type: Azure Function (inline comparison logic)
  Input: @activity('GenerateSourceChecksum').output.path,
         @activity('GenerateTargetChecksum').output.path
  Logic: compare source_checksums.json vs target_checksums.json
  Output: mismatch_count, mismatch_files[]

Activity 5: FailIfMismatch
  Type: IfCondition
  Condition: @greater(activity('CompareChecksums').output.mismatch_count, 0)
  If True → Fail activity with custom error message listing mismatched files
  If False → Send success notification
```

**Python checksum generation script (for SHIR Custom Activity):**
```python
import hashlib
import json
import os
import sys

def compute_md5(filepath):
    h = hashlib.md5()
    with open(filepath, 'rb') as f:
        for chunk in iter(lambda: f.read(8192), b''):
            h.update(chunk)
    return h.hexdigest()

def generate_checksums(directory):
    checksums = {}
    for root, dirs, files in os.walk(directory):
        for filename in files:
            filepath = os.path.join(root, filename)
            relative_path = os.path.relpath(filepath, directory)
            checksums[relative_path] = {
                "md5": compute_md5(filepath),
                "size": os.path.getsize(filepath)
            }
    return checksums

result = generate_checksums(sys.argv[1])
with open('checksums.json', 'w') as f:
    json.dump(result, f, indent=2)
print(json.dumps({"status": "success", "file_count": len(result)}))
```

**Approach 3: ADF row-count reconciliation (for database migrations)**
```json
// Get Metadata Activity on source table
// Get row count: SELECT COUNT(*) FROM source_table WHERE load_date = @pipelineRunDate
// Get row count: SELECT COUNT(*) FROM target_table WHERE load_date = @pipelineRunDate
// Compare with Set Variable + IfCondition
// Fail pipeline if counts differ
```

---

**Q5. How do you implement incremental data loading (watermark pattern) in ADF?**

**Answer:**

**Watermark pattern:** Track the "high-water mark" (max timestamp or ID) of the last successful load. Next run only processes records newer than the watermark.

**Setup:**

```sql
-- Control table in Azure SQL to store watermarks
CREATE TABLE dbo.WatermarkTable (
    TableName NVARCHAR(255) PRIMARY KEY,
    WatermarkValue DATETIME2,
    LastSuccessfulRunId NVARCHAR(100),
    LastRunTimestamp DATETIME2
);

INSERT INTO dbo.WatermarkTable VALUES
('FactOrders', '2024-01-01 00:00:00', NULL, NULL);
```

**ADF Pipeline: IncrementalLoad**

```
Activity 1: LookupOldWatermark
  Type: Lookup
  Query: SELECT WatermarkValue FROM dbo.WatermarkTable WHERE TableName = 'FactOrders'
  Output: @activity('LookupOldWatermark').output.firstRow.WatermarkValue

Activity 2: LookupNewWatermark
  Type: Lookup  
  Query: SELECT MAX(UpdatedAt) AS NewWatermark FROM source.FactOrders
  Output: @activity('LookupNewWatermark').output.firstRow.NewWatermark

Activity 3: CopyIncremental
  Type: Copy Activity
  Query: |
    SELECT * FROM source.FactOrders
    WHERE UpdatedAt > '@{activity('LookupOldWatermark').output.firstRow.WatermarkValue}'
      AND UpdatedAt <= '@{activity('LookupNewWatermark').output.firstRow.NewWatermark}'
  Sink: ADLS Gen2 curated/silver/fact_orders/

Activity 4: UpdateWatermark
  Type: Stored Procedure (on control DB)
  Procedure: sp_UpdateWatermark
  Parameters:
    TableName: FactOrders
    NewWatermark: @activity('LookupNewWatermark').output.firstRow.NewWatermark
    RunId: @pipeline().RunId
```

**Stored procedure:**
```sql
CREATE PROCEDURE sp_UpdateWatermark
    @TableName NVARCHAR(255),
    @NewWatermark DATETIME2,
    @RunId NVARCHAR(100)
AS
BEGIN
    UPDATE dbo.WatermarkTable
    SET WatermarkValue = @NewWatermark,
        LastSuccessfulRunId = @RunId,
        LastRunTimestamp = GETUTCDATE()
    WHERE TableName = @TableName;
END
```

**Idempotency note:** The CopyIncremental activity uses a half-open interval (`>` old watermark, `<=` new watermark). This means re-running the same pipeline window produces the same output — safe for retries.

---

**Q6. How do you handle pipeline failures and implement retry logic in ADF?**

**Answer:**

**Activity-level retry:**
```json
{
  "name": "CopyActivity",
  "type": "Copy",
  "policy": {
    "timeout": "02:00:00",      // 2 hour timeout
    "retry": 3,                  // Retry up to 3 times
    "retryIntervalInSeconds": 30, // Wait 30s between retries
    "secureOutput": false,
    "secureInput": false
  }
}
```

**Pipeline-level error handling (IfCondition + Error activity):**
```
CopyActivity
  ├── On Success → TransformActivity → NotifySuccess
  └── On Failure → ErrorHandlerActivity
        ├── LogFailureToSQL (stored proc: log error details)
        ├── SendAlertEmail (via Logic App webhook)
        └── Fail (re-raise failure to propagate to parent pipeline)
```

**Dead letter pattern for unrecoverable records:**
```
CopyActivity (with fault tolerance enabled):
  faultTolerance:
    enableSkipIncompatibleRow: true   // Skip bad rows instead of failing
    redirectIncompatibleRowSettings:
      linkedServiceName: adls-linked-service
      path: raw/dead-letter/@pipeline().RunId/   // Store bad rows for investigation
```

**Alerting via ADF Diagnostic Settings:**
```bash
# Route ADF pipeline failure events to Log Analytics
az monitor diagnostic-settings create \
  --name adf-diag \
  --resource /subscriptions/.../resourceGroups/.../providers/Microsoft.DataFactory/factories/... \
  --workspace /subscriptions/.../resourceGroups/.../providers/Microsoft.OperationalInsights/workspaces/... \
  --logs '[{"category": "PipelineRuns", "enabled": true},
           {"category": "ActivityRuns", "enabled": true},
           {"category": "TriggerRuns", "enabled": true}]'

# KQL Alert query: pipelines failed in last hour
AdfActivityRun
| where Status == "Failed"
| where TimeGenerated > ago(1h)
| summarize FailureCount = count() by PipelineName, Error
| where FailureCount > 0
```

---

## ADVANCED

**Q7. Design an ADF pipeline for a full + incremental data migration with checksum validation from on-premises SQL Server to Azure Synapse, handling 500GB/day.**

**Answer:**

**Architecture overview:**
```
On-premises SQL Server (500GB/day new data)
  │  [Self-hosted IR — SHIR cluster, 2 nodes HA]
  │
  ▼
ADF Pipeline: MasterOrchestrator
  │
  ├─► [Phase 1: Historical Full Load — runs once]
  │     ForEach(large tables): parallel copy, 8 DIU per table
  │     Checksum validation after each table
  │     Watermark initialized on success
  │
  └─► [Phase 2: Incremental Load — daily trigger, 2 AM ET]
        LookupWatermarks → IncrementalCopy → ValidateRowCounts → UpdateWatermarks
        
  ▼
ADLS Gen2 (raw landing zone)
  raw/sqlserver/tablename/year=2024/month=01/day=15/
  
  ▼
ADF Data Flow: SilverTransform (runs after incremental copy)
  → Deduplicate (ROW_NUMBER on primary key, keep latest)
  → Type casting and null handling
  → Write to ADLS Gen2 silver/ (Delta Lake format)
  
  ▼
Synapse Pipeline: SynapseLoad (COPY INTO statement)
  → Load silver Delta Lake → Synapse Dedicated SQL Pool
  → Distribution: HASH(CustomerId) for fact tables, REPLICATE for dims
```

**SHIR configuration for 500GB/day:**
```json
// Self-hosted IR optimization
{
  "concurrentJobsLimit": 32,    // Parallel jobs per node
  "maxParallelCopiesPerJob": 8, // Parallel threads per copy
  "taskQueueTimeoutInMinutes": 120
}

// Copy Activity DIU (Data Integration Units) settings
// Azure IR: 4-256 DIU (default 4)
// 1 DIU = 4GB/min for blob-to-blob
// 8 DIU = target for 500GB/day incremental (fits in 4hr window)
{
  "parallelCopies": 8,
  "dataIntegrationUnits": 8
}
```

**End-to-end checksum validation pipeline:**
```
Per table:
  1. Before copy: SELECT COUNT(*), CHECKSUM_AGG(BINARY_CHECKSUM(*)) FROM source_table
                  WHERE UpdatedAt BETWEEN @old_watermark AND @new_watermark
     → Store: row_count=145230, checksum=3847291847

  2. Copy Activity (SHIR → ADLS Gen2)
     → validateDataConsistency: true (file-level MD5)

  3. After copy: Read Parquet from ADLS Gen2, compute:
     SELECT COUNT(*), SUM(CAST(BINARY_CHECKSUM(*) AS BIGINT)) FROM @sink_parquet

  4. Compare:
     IF source_count != target_count OR source_checksum != target_checksum:
       → Log mismatch details to audit table
       → Re-run copy for this table
       → Alert on-call engineer if 2nd attempt fails

  5. Audit log insert (every run, success or fail):
     INSERT INTO dbo.MigrationAudit (
       RunId, TableName, SourceCount, TargetCount, SourceChecksum, TargetChecksum,
       Status, DurationSeconds, RunTimestamp
     )
```

**Cost optimization for 500GB/day:**
```
SHIR: 2× D8s_v3 VMs (8 vCPU, 32GB) on-prem = ~$0 (existing hardware)
ADF Azure IR: 8 DIU × $0.25/DIU/hr × 2hr/day = $4/day
ADLS Gen2 storage: 500GB × $0.021/GB = $10.50/day
Synapse Dedicated SQL COPY: minimal cost within pool
Total: ~$15/day = ~$450/month
```

---

**Q8. How do you implement CDC (Change Data Capture) in ADF for real-time-like database replication?**

**Answer:**

**ADF native CDC (preview/GA for SQL Server):**

```json
// ADF Copy Activity with CDC mode
{
  "source": {
    "type": "SqlServerSource",
    "sqlReaderQuery": null,
    "isolationLevel": "ReadCommitted"
  },
  "enableChangeDataCapture": true,
  "cdcStartLsn": null,         // null = start from current position
  "cdcStopLsn": null,          // null = read all available changes
  "cdcOption": "allChanges"    // net changes or all changes
}
```

**Debezium-based CDC (more powerful, production-grade):**

```yaml
# Debezium SQL Server connector config
{
  "connector.class": "io.debezium.connector.sqlserver.SqlServerConnector",
  "database.hostname": "sqlserver.corp.internal",
  "database.port": "1433",
  "database.user": "debezium",
  "database.password": "${file:/opt/kafka/secrets:db_password}",
  "database.dbname": "FinanceDB",
  "database.server.name": "finance-sqlserver",
  "table.include.list": "dbo.Accounts,dbo.Transactions",
  "database.history.kafka.bootstrap.servers": "eventhubs.servicebus.windows.net:9093",
  "database.history.kafka.topic": "schema-changes.finance",
  
  # Output format
  "transforms": "unwrap",
  "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
  "transforms.unwrap.add.fields": "op,ts_ms",  // Include operation type + timestamp
  "transforms.unwrap.delete.handling.mode": "rewrite"
}
```

**Event Hubs (Kafka surface) → ADF pipeline:**
```
Event Hubs (CDC events)
  ↓
Databricks Structured Streaming (consumes Kafka topic)
  ↓
MERGE INTO delta.accounts USING incoming ON accounts.id = incoming.id
WHEN MATCHED AND incoming.op = 'd' THEN DELETE
WHEN MATCHED AND incoming.op IN ('u', 'r') THEN UPDATE SET *
WHEN NOT MATCHED AND incoming.op = 'c' THEN INSERT *
  ↓
Delta Lake silver/accounts (always up-to-date replica)
```

**Lag monitoring:**
```python
# Check CDC replication lag
lag_seconds = (now - max(cdc_event_timestamp)).total_seconds()
if lag_seconds > 300:  # 5 minute lag threshold
    send_alert(f"CDC lag: {lag_seconds}s on table Transactions")
```
