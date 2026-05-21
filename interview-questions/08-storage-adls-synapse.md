# Azure Storage, ADLS Gen2 & Synapse Analytics — Interview Questions

> Covers: Blob storage tiers, ADLS Gen2 architecture, Synapse pools, lakehouse pattern, hot/cold/archive tiering.
> Levels: Beginner → Intermediate → Advanced

---

## BEGINNER

**Q1. What are the Azure Blob Storage access tiers and when do you use each?**

**Answer:**

| Tier | Cost | Retrieval | Use case | Min storage duration |
|------|------|-----------|----------|---------------------|
| **Hot** | High storage cost, low access cost | Immediate | Frequently accessed data, active files | None |
| **Cool** | Lower storage cost, moderate access cost | Immediate | Infrequently accessed (>30 days between reads) | 30 days |
| **Cold** | Lower than Cool, higher than Archive | Immediate | Rarely accessed data | 90 days |
| **Archive** | Lowest storage cost, high retrieval cost + delay | 1–15 hours to rehydrate | Long-term retention, compliance, audit logs | 180 days |

**Cost example (approximate, Canada Central):**
```
Hot:     $0.021/GB/month storage, $0.004/10K read operations
Cool:    $0.010/GB/month storage, $0.010/10K read operations
Cold:    $0.005/GB/month storage, $0.018/10K read operations
Archive: $0.001/GB/month storage, $5.50/10K read operations + rehydration fee
```

**Lifecycle management policy (automate tiering):**
```json
{
  "rules": [
    {
      "name": "move-to-cool",
      "type": "Lifecycle",
      "definition": {
        "filters": { "blobTypes": ["blockBlob"] },
        "actions": {
          "baseBlob": {
            "tierToCool": { "daysAfterModificationGreaterThan": 30 },
            "tierToArchive": { "daysAfterModificationGreaterThan": 365 },
            "delete": { "daysAfterModificationGreaterThan": 2555 }
          }
        }
      }
    }
  ]
}
```

---

**Q2. What is the difference between Azure Blob Storage and Azure Data Lake Storage Gen2?**

**Answer:**

ADLS Gen2 is NOT a separate service — it's **Azure Blob Storage with the Hierarchical Namespace (HNS) feature enabled.**

| Feature | Blob Storage | ADLS Gen2 (HNS enabled) |
|---------|-------------|------------------------|
| Namespace | Flat (container/blob) | Hierarchical (true directories) |
| Directory rename/delete | O(n) — must rename every blob | O(1) — metadata operation only |
| ACLs | Container + blob level | File and folder-level POSIX ACLs |
| Protocol | HTTPS REST + SDK | HTTPS REST + ABFS (Azure Blob File System) |
| Analytics optimization | Limited | Full Hadoop/Spark/Databricks compatible |
| Performance | Good | Optimized for parallel analytics |

**ABFS (Azure Blob File System) driver:**
- Databricks, Synapse, and HDInsight all use ABFS to connect to ADLS Gen2
- Connection string: `abfss://container@account.dfs.core.windows.net/path`

**Enable HNS:** Must be set at storage account creation — cannot be enabled later without migration.

---

**Q3. What is Azure Synapse Analytics and how does it differ from Azure Databricks?**

**Answer:**

| Feature | Azure Synapse | Azure Databricks |
|---------|--------------|-----------------|
| Primary use | SQL analytics + data warehouse | Data engineering, ML, streaming |
| SQL engine | Dedicated SQL Pool (T-SQL) + Serverless SQL | Databricks SQL (Spark SQL) |
| ETL | Synapse Pipelines (ADF-based) | Notebooks + workflows |
| Spark | Synapse Spark Pools (managed Spark) | Databricks Runtime (optimized Spark) |
| Integration | Deep: Power BI, Purview, Azure ML | Via connectors |
| Administration | Simpler (unified workspace) | More control over cluster config |
| Cost model | Dedicated: always-on billed by DWU; Serverless: billed per TB scanned | DBU (Databricks Units) per hour |

**When to choose:**
- **Synapse**: Business analysts + SQL-first team, need a managed data warehouse, Power BI direct integration
- **Databricks**: Data engineers + ML team, complex ETL, streaming, need Delta Live Tables, ML workflows
- **Both**: Large enterprises often use both — Databricks for ingestion/transform, Synapse for serving/BI

---

## INTERMEDIATE

**Q4. Explain the Lakehouse architecture pattern implemented on Azure.**

**Answer:**

The Lakehouse combines the flexibility of a data lake (raw data, any format, low cost) with the performance and ACID properties of a data warehouse (structured, governed, query-optimized).

**Zones:**

```
RAW ZONE (Bronze)
  ADLS Gen2 — raw/bronze/
  Format: As-is (JSON, CSV, Parquet, Avro, XML)
  Schema: Schema-on-read (no enforcement)
  Retention: Forever (source of truth)
  Access: Data engineers only
  
  Ingestion: ADF Copy Activity, Event Hubs Capture, IoT Hub routing

CURATED ZONE (Silver)
  ADLS Gen2 — curated/silver/
  Format: Delta Lake (Parquet + transaction log)
  Schema: Enforced (data quality rules applied)
  Operations: Deduplication, standardization, null handling
  ACID: Yes (Delta Lake MERGE, UPDATE, DELETE supported)
  
  Processing: Databricks Structured Streaming or ADF Data Flows

ENRICHED ZONE (Gold)
  ADLS Gen2 — enriched/gold/
  Format: Delta Lake (optimized with ZORDER indexing)
  Schema: Business logic applied (joins, aggregations, derived columns)
  Audience: Business analysts, ML teams, BI tools
  
  Processing: Databricks Notebooks or Synapse Spark Pools

SERVING LAYER
  Synapse Analytics Serverless SQL Pool
  → Directly queries Delta Lake files in ADLS Gen2 via OPENROWSET
  → No data movement, pay per TB scanned
  → Power BI connects via Synapse Serverless
  
  OR: Synapse Dedicated SQL Pool
  → Copy aggregated data into DWU for <1s query performance
  → Used for dashboards with high concurrency
```

**Delta Lake advantages over plain Parquet:**
```
1. ACID transactions:
   MERGE INTO target USING source ON target.id = source.id
   WHEN MATCHED THEN UPDATE SET ...
   WHEN NOT MATCHED THEN INSERT ...
   → No more "overwrite all" — safe upserts

2. Time travel:
   spark.read.format("delta").option("versionAsOf", 5).load(path)
   → Read data as it was at any previous version

3. Schema enforcement + evolution:
   New column added upstream → Delta rejects unless schema merge enabled
   → spark.conf.set("spark.databricks.delta.schema.autoMerge.enabled", "true")

4. Optimized file management:
   OPTIMIZE tableName ZORDER BY (customerId, date)
   → Merges small files, co-locates related data (data skipping)
   VACUUM tableName RETAIN 168 HOURS  -- remove files older than 7 days
```

---

**Q5. How do you implement row-level security on ADLS Gen2 data in Synapse Analytics?**

**Answer:**

ADLS Gen2 supports POSIX-style ACLs at file and folder level:

**Folder-level ACL pattern:**
```bash
# Data engineers: full access to raw zone
az storage fs access set \
  --acl "user:engineer-group-oid:rwx" \
  --path "raw/" \
  --file-system data-lake \
  --account-name adlsprod

# Analysts: read-only on gold zone
az storage fs access set \
  --acl "user:analyst-group-oid:r-x" \
  --path "gold/" \
  --file-system data-lake \
  --account-name adlsprod

# No access to silver (intermediate) zone for most users
az storage fs access set \
  --acl "other::---" \
  --path "silver/" \
  --file-system data-lake \
  --account-name adlsprod
```

**Synapse Serverless SQL row-level filtering:**
```sql
-- View that applies row-level filter based on user context
CREATE VIEW dbo.CustomerOrders AS
SELECT *
FROM OPENROWSET(
    BULK 'https://adlsprod.dfs.core.windows.net/gold/orders/*.parquet',
    FORMAT = 'PARQUET'
) WITH (
    customerId NVARCHAR(50),
    amount DECIMAL(18,2),
    region NVARCHAR(50)
) AS orders
WHERE region = SESSION_CONTEXT(N'user_region');  -- Dynamic filter

-- Application sets context before query
EXEC sp_set_session_context @key = N'user_region', @value = N'Canada';
```

**Azure Purview for data governance:**
- Classify sensitive columns (PII, PCI, financial data)
- Data lineage: track data from source → raw → gold
- Access policy: Purview policies can grant/deny data access based on classification
- Data catalog: business users discover what data exists and its meaning

---

**Q6. How do you optimize Synapse Dedicated SQL Pool performance for a 10TB data warehouse?**

**Answer:**

**Distribution strategy (most important design decision):**

```sql
-- HASH distribution: distribute rows by chosen column across 60 distributions
-- Best for: large fact tables, join performance
CREATE TABLE dbo.FactOrders (
    OrderId BIGINT NOT NULL,
    CustomerId INT NOT NULL,
    ...
)
WITH (
    DISTRIBUTION = HASH(CustomerId),  -- join key = partition key
    CLUSTERED COLUMNSTORE INDEX
);

-- ROUND_ROBIN: even distribution, no hotspots
-- Best for: staging tables, tables without clear join key
CREATE TABLE dbo.StagingOrders
WITH (DISTRIBUTION = ROUND_ROBIN, HEAP);  -- HEAP for fast inserts

-- REPLICATE: copy table to all distributions
-- Best for: small dimension tables (<2GB) joined to fact tables
CREATE TABLE dbo.DimCustomer
WITH (DISTRIBUTION = REPLICATE, CLUSTERED COLUMNSTORE INDEX);
```

**Data movement in queries (minimize this):**
```
BroadcastMove: small table copied to all nodes (good — controlled)
ShuffleMove:   rows reshuffled across nodes to match join key (expensive)
LocalJoin:     no data movement (best — tables distributed on same key)

Design for LocalJoin:
  FactOrders HASH(CustomerId) + DimCustomer REPLICATE
  → All FactOrders rows join against local copy of DimCustomer
  → No ShuffleMove
```

**Columnstore index optimization:**
```sql
-- Rebuild fragmented columnstore (after many small inserts)
ALTER INDEX ALL ON dbo.FactOrders REBUILD;

-- Check segment quality
SELECT object_name(object_id), index_id,
    COUNT(*) as total_rowgroups,
    SUM(CASE WHEN state_desc = 'COMPRESSED' THEN 1 ELSE 0 END) compressed,
    SUM(CASE WHEN state_desc = 'OPEN' THEN 1 ELSE 0 END) open_delta
FROM sys.dm_db_column_store_row_group_physical_stats
GROUP BY object_id, index_id;
-- Aim: most rowgroups COMPRESSED with ~1M rows each (not 10K — too small)
```

**DWU scaling:**
```bash
# Scale up for heavy ETL loads, scale down during off-hours
az synapse sql pool scale \
  --name sqlpool-prod \
  --workspace-name synapse-prod \
  --resource-group rg-data \
  --performance-level DW2000c

# Pause during weekends (save ~65% of cost)
az synapse sql pool pause \
  --name sqlpool-prod \
  --workspace-name synapse-prod \
  --resource-group rg-data
```

---

## ADVANCED

**Q7. Design a lakehouse ingestion pipeline for 10 million documents per day at a financial services firm.**

**Answer:**

**Scale analysis:**
- 10M docs/day = ~116 docs/sec average
- Peak: 3× average = ~350 docs/sec
- Document size: average 500KB → 5TB/day → 150TB/month raw storage

**Architecture:**

```
INGESTION LAYER
  Multiple source types:
    Email attachments → Logic Apps → Blob trigger
    API submissions   → APIM → Azure Functions → Event Grid
    Bulk upload       → SFTP → Azure Blob (via Azure Transfer)
    Scan/OCR feeds    → Document Intelligence webhook → Event Grid

  Landing zone: ADLS Gen2 raw/incoming/ (Hot tier)
  Event trigger: Blob Created → Event Grid → Service Bus (durable queue)

PROCESSING LAYER (KEDA-scaled AKS workers)
  Consumer: AKS pod (50 instances max, KEDA on Service Bus queue)
  
  Per document:
    1. Read from ADLS Gen2 raw/incoming/
    2. Azure Document Intelligence (pre-built or custom model)
       → Extract: document type, date, parties, amounts, key fields
    3. PII detection (Azure AI Language - PII entity recognition)
       → Redact PII from extracted text before indexing
    4. Write to ADLS Gen2 curated/silver/ (Delta Lake)
    5. Publish metadata to AI Search (vector + keyword index)
    6. Write metadata to CosmosDB (partition key: documentType + month)
    7. Acknowledge Service Bus message (remove from queue)

DELTA LAKE SCHEMA (Silver zone):
  document_id        STRING  NOT NULL
  source_system      STRING  NOT NULL
  document_type      STRING  NOT NULL  -- invoice, contract, claim, statement
  ingested_at        TIMESTAMP
  processed_at       TIMESTAMP
  extracted_fields   MAP<STRING, STRING>  -- flexible schema per doc type
  pii_detected       BOOLEAN
  content_hash       STRING  -- SHA-256 for deduplication
  status             STRING  -- pending, processed, failed, duplicate

DEDUPLICATION:
  MERGE INTO silver.documents USING incoming ON
    silver.content_hash = incoming.content_hash
  WHEN NOT MATCHED THEN INSERT *
  WHEN MATCHED THEN UPDATE SET status = 'duplicate'

AI SEARCH INDEX (for document retrieval):
  Fields: document_id, document_type, summary (embedding), 
          key_entities, date, amount, parties
  Hybrid search: BM25 keyword + vector (ada-002 embeddings) with RRF reranking
  Index update: push API from processing worker

STORAGE TIERING:
  raw/incoming/: Hot → lifecycle policy → Cool after 7 days, Archive after 365 days
  curated/silver/: Cool (analytics occasional) 
  enriched/gold/: Hot (active reporting)
  
MONITORING:
  Queue depth alert: Service Bus queue > 1M messages → scale up AKS max replicas
  Processing lag: document_processed_at - document_created_at > 5 min → alert
  Error rate: > 1% DI failures → alert
```

---

**Q8. Explain the difference between Synapse Dedicated SQL Pool, Serverless SQL Pool, and when to use each for a BI platform.**

**Answer:**

**Synapse Serverless SQL Pool:**
```
Cost model: Pay per TB scanned ($5/TB)
Data location: Reads directly from ADLS Gen2 / Delta Lake
Scaling: Automatic, no provisioning
Max query: 30-minute timeout, 10TB per query
Concurrency: 200 concurrent queries
Best for:
  → Ad-hoc exploration of data lake
  → Infrequent reporting on large historical datasets
  → Power BI DirectQuery over Delta Lake (small dashboards)
  → Data cataloging / discovery queries

Example:
  SELECT year, SUM(amount) as total_amount
  FROM OPENROWSET(
      BULK 'https://adls.dfs.core.windows.net/gold/orders/year=2024/**',
      FORMAT = 'PARQUET'
  ) AS orders
  GROUP BY year;
```

**Synapse Dedicated SQL Pool:**
```
Cost model: Always-on, billed by DWU ($1.20/DWU/hour)
Data location: In-pool distributed storage (separate from ADLS)
Scaling: Manual or auto-scale (DW100c to DW30000c)
Max query: No hard limit
Concurrency: 128 concurrent queries (at DW3000c)
Best for:
  → High-concurrency BI dashboards (Power BI Import mode)
  → Sub-second query SLAs on aggregated data
  → Complex SQL with many joins (distribution optimization helps)
  → ETL landing zone for transformed data

Example monthly cost:
  DW500c: $1.20 × 500 × 720hr = $432/month
  Pause weekends: save ~28% → $312/month effective
```

**Decision matrix:**

| Scenario | Use |
|----------|-----|
| 50 analysts exploring raw data lake | Serverless |
| 500 Power BI users hitting same dashboard | Dedicated |
| Monthly regulatory reports on 5 years of data | Serverless |
| Real-time P&L dashboard refreshed every minute | Dedicated |
| Data discovery / sampling before building pipelines | Serverless |
| ETL pipeline loading to warehouse | Dedicated (fast inserts) |

**Hybrid pattern (most common):**
```
ADLS Gen2 Delta Lake
  │
  ├──► Synapse Serverless SQL → Power BI DirectQuery (exploratory, low-concurrency)
  │
  └──► Synapse Pipelines (nightly) → Synapse Dedicated Pool → Power BI Import (high-concurrency dashboards)
```
