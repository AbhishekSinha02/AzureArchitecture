# Azure Synapse Analytics
### 🟡 Intermediate → 🔴 Advanced

> *"Synapse is not a data warehouse. It's a workspace.  
>  The data warehouse is one of several engines running inside it."*

---

## Q1. What is the difference between Dedicated SQL Pool and Serverless SQL Pool, and when do you use each?

**Scenario:** A financial services data team is debating architecture. The analytics team runs ad-hoc queries on 2TB of Parquet files in ADLS. The finance team runs monthly regulatory reports on a fixed schema. Which SQL Pool fits each use case?

```
Two SQL engines in Synapse — different billing models:

  DEDICATED SQL POOL                  SERVERLESS SQL POOL
  ─────────────────────               ──────────────────────────────
  Pre-provisioned compute             On-demand: scale to 0 when idle
  Billed by DWU per hour              Billed by TB of data processed
  ($7.20/hour for DW100c)             ($5/TB scanned)
  Data stored IN the pool             Data stays in ADLS Gen2
  (internal columnar store)           (external tables only)
  Best for: known queries,            Best for: ad-hoc exploration,
  high concurrency, repeatable        data discovery, infrequent queries
  workloads, BI dashboards            on raw lake files

  Finance team → Dedicated Pool       Analytics team → Serverless Pool
  (monthly reports, known schema,     (ad-hoc on Parquet, pay per query,
   high concurrency, fast response)    no data movement needed)
```

```sql
-- Serverless SQL Pool: query Parquet files directly from ADLS
-- No data loading required — SQL runs against the files in-place
SELECT
    YEAR(TransactionDate) AS Year,
    COUNT(*) AS TxCount,
    SUM(Amount) AS TotalAmount
FROM
    OPENROWSET(
        BULK 'https://adlsprodcc.dfs.core.windows.net/curated/transactions/**',
        FORMAT = 'PARQUET'
    ) AS txn
WHERE
    TransactionDate >= '2024-01-01'
GROUP BY
    YEAR(TransactionDate)
ORDER BY
    Year;
-- Cost: depends on Parquet file size scanned. Use column pruning and partition filtering.
```

```sql
-- External table on Serverless Pool (reusable, no query rewrite needed)
CREATE EXTERNAL DATA SOURCE adls_curated
WITH (
    LOCATION = 'https://adlsprodcc.dfs.core.windows.net/curated'
);

CREATE EXTERNAL FILE FORMAT parquet_format
WITH (FORMAT_TYPE = PARQUET);

CREATE EXTERNAL TABLE transactions_ext (
    TransactionId   BIGINT,
    Amount          DECIMAL(18,2),
    TransactionDate DATE,
    CustomerId      INT
)
WITH (
    LOCATION = '/transactions/',
    DATA_SOURCE = adls_curated,
    FILE_FORMAT = parquet_format
);

SELECT TOP 100 * FROM transactions_ext
WHERE TransactionDate >= '2024-01-01';
```

> ⚠️ **Gotcha:** Serverless SQL Pool costs $5/TB of data **scanned** — not data returned. A query that selects 2 columns from a 1TB Parquet table still scans the full file unless the file is column-pruned (Parquet is columnar, so column projection reduces scan). Always use column projection (`SELECT col1, col2`) and partition filters (`WHERE year=2024`) to minimize scan cost.

> 💡 **Deep dive hint:** Dedicated SQL Pool distribution types — `ROUND_ROBIN` (default, fast load) vs `HASH` (fast joins on large tables) vs `REPLICATED` (fast joins on small dimension tables). Choosing the wrong distribution causes data movement operations that 10× query time.

---

## Q2. How do you load data into a Dedicated SQL Pool efficiently?

**Scenario:** You have 100GB of Parquet files in ADLS Gen2 and need to load them into a Dedicated SQL Pool table. The naive approach with `INSERT INTO` takes 3 hours. You need it in 20 minutes.

```
Loading performance: worst → best

  INSERT INTO (row by row)     : hours      ← never do this
  BCP                          : slow        ← legacy, avoid
  ADF Copy Activity            : 30–60 min   ← good for medium loads
  COPY INTO (T-SQL native)     : 10–20 min   ← fastest for bulk
  PolyBase (external table)    : 10–20 min   ← equivalent to COPY INTO

  Key: Dedicated SQL Pool is distributed (60 distributions)
  Maximum throughput when:
  ✅ File count = multiple of 60 (or 60 exactly) → each distribution reads one file
  ✅ File size 100MB–1GB each (not too small, not too large)
  ✅ Load in ROUND_ROBIN distribution (avoids data shuffle)
  ✅ Disable indexes during load, rebuild after
```

```sql
-- COPY INTO — fastest bulk load (T-SQL native, no external table needed)
COPY INTO dbo.transactions
(
    TransactionId   1,
    Amount          2,
    TransactionDate 3,
    CustomerId      4
)
FROM 'https://adlsprodcc.dfs.core.windows.net/curated/transactions/*.parquet'
WITH (
    FILE_TYPE = 'PARQUET',
    CREDENTIAL = (
        IDENTITY = 'Managed Identity'   -- Use Synapse workspace MI (no key needed)
    ),
    COMPRESSION = 'snappy'
);

-- Check load progress
SELECT  r.command,
        s.request_id,
        r.status,
        count(distinct input_name) AS nbr_files,
        sum(s.bytes_processed)/1024/1024 AS mb_processed
FROM    sys.dm_pdw_exec_requests r
JOIN    sys.dm_pdw_dms_external_work s ON r.request_id = s.request_id
WHERE   r.label = 'COPY INTO transactions'
GROUP BY r.command, s.request_id, r.status;
```

```sql
-- Best practice: load to staging table, then INSERT INTO production
-- Staging table: HEAP (no indexes) + ROUND_ROBIN distribution = fastest load
CREATE TABLE dbo.transactions_staging
WITH (
    DISTRIBUTION = ROUND_ROBIN,
    HEAP                                 -- No clustered columnstore = faster inserts
)
AS SELECT * FROM dbo.transactions WHERE 1 = 0;   -- Empty copy for schema

-- Load into staging
COPY INTO dbo.transactions_staging
FROM 'https://adlsprodcc.dfs.core.windows.net/curated/transactions/*.parquet'
WITH (FILE_TYPE = 'PARQUET', CREDENTIAL = (IDENTITY = 'Managed Identity'));

-- Validate row count before swapping
SELECT COUNT(*) FROM dbo.transactions_staging;
-- Expected: matches source count

-- Swap staging to production (rename, not INSERT — instant)
RENAME OBJECT dbo.transactions TO dbo.transactions_old;
RENAME OBJECT dbo.transactions_staging TO dbo.transactions;
DROP TABLE dbo.transactions_old;
```

> ⚠️ **Gotcha:** Dedicated SQL Pool has a concurrency limit — DW100c supports 4 concurrent queries. If 10 users run reports while a bulk load is running, queries queue. Use `resource classes` (`largerc` for loads, `smallrc` for reports) to allocate concurrency slots correctly.

---

## Q3. How do you run Spark notebooks in Synapse Analytics and write back to the lake?

**Scenario:** Your data engineering team uses Spark for complex transformations (joins, deduplication, ML feature engineering) that are too complex for SQL. They need a Spark environment integrated with the data lake, not a separate Databricks workspace.

```
Synapse Spark pool:

  Data Engineer opens Synapse Studio notebook
       │ (browser-based, git-integrated)
       ▼
  Spark pool (e.g., sp-prod-medium: 4 nodes × Standard_DS3_v2)
       │ automatically starts on first cell execution (~2 min cold start)
       │ auto-terminates after 15min idle (cost saving)
       ▼
  ADLS Gen2 (native access via Synapse workspace MSI)
  Read: curated/transactions/
  Write: enriched/customer_360/

  No Databricks license needed — Synapse Spark is built-in
```

```python
# Synapse Spark notebook — reads from lake, transforms, writes back
# No configuration needed — Synapse workspace MSI has access automatically

from pyspark.sql import functions as F
from pyspark.sql.window import Window

# Read Delta table from ADLS (Synapse Spark supports Delta natively)
transactions = spark.read.format("delta").load(
    "abfss://curated@adlsprodcc.dfs.core.windows.net/transactions/"
)

customers = spark.read.format("parquet").load(
    "abfss://curated@adlsprodcc.dfs.core.windows.net/customers/"
)

# Build customer 360 — aggregate + join
customer_360 = (
    transactions
    .groupBy("CustomerId")
    .agg(
        F.count("*").alias("total_transactions"),
        F.sum("Amount").alias("lifetime_value"),
        F.max("TransactionDate").alias("last_transaction_date"),
        F.avg("Amount").alias("avg_transaction_amount")
    )
    .join(customers, "CustomerId", "left")
    .withColumn("risk_tier",
        F.when(F.col("lifetime_value") > 100000, "high")
         .when(F.col("lifetime_value") > 10000, "medium")
         .otherwise("low")
    )
)

# Write to enriched zone as Delta table (partition by risk_tier for query performance)
customer_360.write.format("delta") \
    .mode("overwrite") \
    .partitionBy("risk_tier") \
    .save("abfss://enriched@adlsprodcc.dfs.core.windows.net/customer_360/")
```

```bash
# Create Spark pool in Synapse workspace
az synapse spark pool create \
  --name spProdMedium \
  --workspace-name synapse-prod-cc \
  --resource-group rg-data \
  --node-count 3 \
  --node-size Medium \
  --min-node-count 3 \
  --max-node-count 10 \
  --enable-auto-scale true \
  --delay 15 \                   # Auto-pause after 15 minutes idle
  --enable-auto-pause true \
  --spark-version "3.4"
```

> ⚠️ **Gotcha:** Synapse Spark pool cold start takes 2–5 minutes. ADF pipelines that trigger Synapse notebooks must use a generous activity timeout. If the pipeline timeout is shorter than the pool startup time, the notebook activity fails before the Spark job even starts.

---

## Q4. How do you manage Synapse Dedicated SQL Pool costs — pausing and resuming?

**Scenario:** The Dedicated SQL Pool runs 24/7 costing $5,184/month (DW1000c). It's only queried during business hours (8am–6pm weekdays). You need to cut costs by 65% without affecting the team.

```
Cost profile with scheduling:

  DW1000c: $7.20/hr × 24hr × 30d = $5,184/month

  With auto-pause (weekdays only, business hours):
  Active:  10hr/day × 5 days × 4 weeks = 200 hrs/month
  Paused:  (720 - 200) = 520 hrs/month × $0 = $0
  Monthly: 200hr × $7.20 = $1,440/month  ← 72% savings

  Pause/resume takes ~5 minutes each way
  → Schedule resume at 7:50am, pause at 6:05pm
```

```bash
# Pause Dedicated SQL Pool
az synapse sql pool pause \
  --name sqlpool-prod \
  --workspace-name synapse-prod-cc \
  --resource-group rg-data

# Resume Dedicated SQL Pool
az synapse sql pool resume \
  --name sqlpool-prod \
  --workspace-name synapse-prod-cc \
  --resource-group rg-data

# Automate with Logic App or Azure Automation Runbook (schedule trigger)
# Or use ADF pipeline with Web Activity + schedule trigger:
# POST https://management.azure.com/subscriptions/{sub}/resourceGroups/{rg}/
#      providers/Microsoft.Synapse/workspaces/{ws}/sqlPools/{pool}/pause?api-version=2021-06-01

# Check pool status
az synapse sql pool show \
  --name sqlpool-prod \
  --workspace-name synapse-prod-cc \
  --resource-group rg-data \
  --query "status"
# "Online" or "Paused"

# Scale down before pausing if DWU was temporarily bumped for a load job
az synapse sql pool update \
  --name sqlpool-prod \
  --workspace-name synapse-prod-cc \
  --resource-group rg-data \
  --performance-level DW500c    # Scale from DW1000c to DW500c
```

> ⚠️ **Gotcha:** Pausing a Dedicated SQL Pool does NOT delete data — data is preserved in Azure Managed Disks. However, temporary tables and active transactions are lost. Any ADF pipeline that leaves a staging table unlocked when the pool pauses will need to reload that data on resume.
