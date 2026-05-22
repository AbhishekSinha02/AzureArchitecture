# Azure Databricks & Delta Lake
### 🟡 Intermediate → 🔴 Advanced

> *"Parquet gives you fast reads. Delta Lake gives you correct reads.  
>  In production, you need both."*

---

## Q1. What is Delta Lake and why use it over plain Parquet?

**Scenario:** Your data lake uses raw Parquet files. A Databricks job failed halfway through writing a partition — now the table has half old data and half new data. Downstream Synapse queries are returning wrong results. How does Delta Lake prevent this?

```
Parquet (no guarantees)          Delta Lake (ACID on object storage)
────────────────────────         ───────────────────────────────────
Write job fails at 50%           Write job fails at 50%
→ partial files exist            → transaction log records failure
→ readers see corrupt state      → readers see last committed version
→ manual cleanup required        → no cleanup needed

Delta Lake = Parquet files + _delta_log/ directory
  _delta_log/
    00000000000000000001.json   ← commit 1: add files A, B, C
    00000000000000000002.json   ← commit 2: add files D, delete B
    00000000000000000003.json   ← commit 3: failed — not committed

  Reader asks: "give me latest version"
  → reads log → version 2 = files A, C, D
  → never sees commit 3 (incomplete)
```

```python
# Create a Delta table (saves as Parquet + _delta_log/)
df = spark.read.parquet("abfss://raw@adlsprodcc.dfs.core.windows.net/transactions/")

df.write.format("delta") \
  .mode("overwrite") \
  .partitionBy("year", "month") \
  .save("abfss://curated@adlsprodcc.dfs.core.windows.net/transactions/")

# Read Delta table
df = spark.read.format("delta").load(
    "abfss://curated@adlsprodcc.dfs.core.windows.net/transactions/"
)

# Time travel — read a previous version (audit / debugging)
df_yesterday = spark.read.format("delta") \
    .option("versionAsOf", 5) \
    .load("abfss://curated@adlsprodcc.dfs.core.windows.net/transactions/")

# Or by timestamp
df_audit = spark.read.format("delta") \
    .option("timestampAsOf", "2024-01-15 09:00:00") \
    .load("abfss://curated@adlsprodcc.dfs.core.windows.net/transactions/")
```

```sql
-- Register as SQL table (Databricks SQL / Synapse can query via SQL)
CREATE TABLE transactions
USING DELTA
LOCATION 'abfss://curated@adlsprodcc.dfs.core.windows.net/transactions/';

-- Show Delta table history (all commits)
DESCRIBE HISTORY transactions;
-- Shows: version, timestamp, operation, operationParameters, numFiles added/removed

-- Check table details
DESCRIBE DETAIL transactions;
-- Shows: format=delta, numFiles, sizeInBytes, partitionColumns
```

> ⚠️ **Gotcha:** Delta table `_delta_log/` and data files must be in the same directory. If you move or copy Delta files without copying the `_delta_log/`, the new location sees only Parquet files with no Delta guarantees. Always copy the entire directory structure including `_delta_log/`.

> 💡 **Deep dive hint:** Delta Lake schema evolution — `mergeSchema` option allows adding new columns without failing the write. `overwriteSchema` replaces the schema entirely. Understanding when each is safe prevents schema drift bugs in long-running pipelines.

---

## Q2. How do you implement MERGE (upsert) in Delta Lake for daily incremental loads?

**Scenario:** A daily ADF pipeline drops a delta file containing new rows and updated rows (no deletes). Your Delta table must reflect the latest state — new rows added, updated rows overwritten. A full overwrite would destroy history.

```
MERGE operation:

  Source (daily delta)           Target (Delta table — full history)
  ─────────────────────          ────────────────────────────────────
  ID=1, Amount=150 (update)      ID=1, Amount=100  ← will be updated
  ID=2, Amount=200 (new)         ID=3, Amount=300  ← unchanged
  (ID=3 not in source)           ID=4, Amount=400  ← unchanged

  MERGE result:
  ID=1, Amount=150  ✅ updated
  ID=2, Amount=200  ✅ inserted
  ID=3, Amount=300  ✅ unchanged
  ID=4, Amount=400  ✅ unchanged
```

```python
from delta.tables import DeltaTable

# Load target (existing Delta table)
target = DeltaTable.forPath(
    spark,
    "abfss://curated@adlsprodcc.dfs.core.windows.net/transactions/"
)

# Load source (today's delta)
source = spark.read.parquet(
    "abfss://raw@adlsprodcc.dfs.core.windows.net/transactions/2024/01/16/"
)

# MERGE: upsert based on TransactionId
(
    target.alias("tgt")
    .merge(
        source.alias("src"),
        "tgt.TransactionId = src.TransactionId"   # Match condition
    )
    .whenMatchedUpdateAll()       # If match: update all columns
    .whenNotMatchedInsertAll()    # If no match: insert new row
    .execute()
)
```

```sql
-- SQL equivalent (Databricks SQL)
MERGE INTO transactions AS tgt
USING daily_delta AS src
ON tgt.TransactionId = src.TransactionId
WHEN MATCHED THEN
  UPDATE SET
    Amount          = src.Amount,
    LastModifiedDate = src.LastModifiedDate,
    Status           = src.Status
WHEN NOT MATCHED THEN
  INSERT (TransactionId, Amount, TransactionDate, CustomerId, Status)
  VALUES (src.TransactionId, src.Amount, src.TransactionDate, src.CustomerId, src.Status);
```

```python
# Handle soft deletes (mark rows as deleted instead of physical delete)
(
    target.alias("tgt")
    .merge(
        source.alias("src"),
        "tgt.TransactionId = src.TransactionId"
    )
    .whenMatchedUpdate(
        condition = "src.IsDeleted = true",
        set = {"IsDeleted": "true", "DeletedDate": "current_timestamp()"}
    )
    .whenMatchedUpdateAll(condition = "src.IsDeleted = false")
    .whenNotMatchedInsertAll()
    .execute()
)
```

> ⚠️ **Gotcha:** MERGE performance degrades when the source and target are in different partitions. If `TransactionId` is the merge key but the table is partitioned by `year/month`, Delta scans ALL partitions to find matching rows. Add `year` and `month` to the merge condition to limit partition scan: `tgt.TransactionId = src.TransactionId AND tgt.year = src.year AND tgt.month = src.month`.

---

## Q3. How do you run structured streaming in Databricks to process Event Hubs data?

**Scenario:** Transactions arrive on Event Hubs at 50,000 events/minute. You need to enrich each event in near-real-time (< 2s latency) with customer data from a lookup table, and write to a Delta table in ADLS.

```
Streaming pipeline:

  Event Hubs (eh-transactions)
       │ 50K events/min, 8 partitions
       │ Kafka-compatible endpoint
       ▼
  Databricks Structured Streaming
       │ readStream: Kafka source → parse JSON → enrich with customer lookup
       │ checkpoint: ADLS Gen2 (tracks offset per partition)
       ▼
  Delta table: enriched/transactions_streaming/
       │ writeStream with trigger.processingTime=30s
       ▼
  Power BI: DirectQuery on Delta table (near-real-time)
```

```python
# Read from Event Hubs (Kafka endpoint)
EVENTHUB_NAMESPACE = "eh-ns-prod.servicebus.windows.net"
EVENTHUB_NAME      = "eh-transactions"
CONNECTION_STRING  = dbutils.secrets.get("kv-scope", "eh-connection-string")

df_stream = (
    spark.readStream
    .format("kafka")
    .option("kafka.bootstrap.servers", f"{EVENTHUB_NAMESPACE}:9093")
    .option("subscribe", EVENTHUB_NAME)
    .option("kafka.security.protocol", "SASL_SSL")
    .option("kafka.sasl.mechanism", "PLAIN")
    .option("kafka.sasl.jaas.config",
        f'org.apache.kafka.common.security.plain.PlainLoginModule required '
        f'username="$ConnectionString" password="{CONNECTION_STRING}";')
    .option("startingOffsets", "latest")
    .load()
)

# Parse JSON payload from Kafka value (bytes → JSON fields)
from pyspark.sql.types import *
from pyspark.sql import functions as F

schema = StructType([
    StructField("TransactionId", LongType()),
    StructField("CustomerId", IntegerType()),
    StructField("Amount", DoubleType()),
    StructField("TransactionDate", TimestampType())
])

df_parsed = (
    df_stream
    .select(F.from_json(F.col("value").cast("string"), schema).alias("data"))
    .select("data.*")
)

# Enrich with customer lookup (static DataFrame — broadcast join)
customers = spark.read.format("delta").load(
    "abfss://curated@adlsprodcc.dfs.core.windows.net/customers/"
).select("CustomerId", "CustomerName", "RiskTier")

df_enriched = df_parsed.join(
    F.broadcast(customers), "CustomerId", "left"
)

# Write to Delta table with checkpoint (ensures exactly-once semantics)
query = (
    df_enriched
    .writeStream
    .format("delta")
    .outputMode("append")
    .option("checkpointLocation",
        "abfss://enriched@adlsprodcc.dfs.core.windows.net/_checkpoints/transactions_streaming/")
    .trigger(processingTime="30 seconds")   # Micro-batch every 30s
    .start("abfss://enriched@adlsprodcc.dfs.core.windows.net/transactions_streaming/")
)

# Monitor stream health
query.status           # {"message": "Processing new data", "isDataAvailable": true}
query.recentProgress   # Batch duration, rows processed, input rate
```

> ⚠️ **Gotcha:** The checkpoint directory must NOT be deleted or moved while the stream is running. Deleting the checkpoint loses offset tracking — on restart, the stream re-reads from the beginning (or `latest`, depending on `startingOffsets`) and you'll get duplicate or missing records. Treat checkpoint directories as stateful — back them up.

---

## Q4. How do you optimize Delta Lake performance with OPTIMIZE, Z-ORDER, and VACUUM?

**Scenario:** Your Delta table was built by a streaming job writing micro-batches every 30 seconds for 6 months. Now it has 180,000 small Parquet files (each ~5MB). Queries that used to take 2 minutes now take 45 minutes.

```
Small file problem:

  30s micro-batch × 8 partitions × 180 days × 86,400s/day ÷ 30 = ~500K file operations
  Result: thousands of 5MB files per partition

  Query: reads 500 files × 5MB = 2.5GB data
         but opens 500 file handles = massive overhead

  After OPTIMIZE:
  500 small files → 1 large 2.5GB Parquet file per partition
  Query: reads 1 file × 2.5GB = same data, 100× fewer I/O operations
```

```python
# OPTIMIZE: compact small files into 1GB target files
spark.sql("""
    OPTIMIZE delta.`abfss://enriched@adlsprodcc.dfs.core.windows.net/transactions_streaming/`
""")

# OPTIMIZE + ZORDER: compact AND sort data by commonly queried columns
# Z-ORDER co-locates rows with the same key into the same files
# Benefit: queries with WHERE CustomerId = X skip 90% of files entirely
spark.sql("""
    OPTIMIZE delta.`abfss://enriched@adlsprodcc.dfs.core.windows.net/transactions_streaming/`
    ZORDER BY (CustomerId, TransactionDate)
""")
# Duration: 20-60 min for a large table. Run weekly or after bulk loads.

# Check how many files exist before and after
spark.sql("""
    DESCRIBE DETAIL delta.`abfss://enriched@adlsprodcc.dfs.core.windows.net/transactions_streaming/`
""").select("numFiles", "sizeInBytes").show()

# VACUUM: delete old file versions that are no longer referenced
# Default retention: 7 days (Delta keeps old files for time travel)
spark.sql("""
    VACUUM delta.`abfss://enriched@adlsprodcc.dfs.core.windows.net/transactions_streaming/`
    RETAIN 168 HOURS   -- 7 days minimum (do NOT go below 7 days)
""")
# Frees disk space — removes files that no current or recent version uses
```

**Schedule OPTIMIZE as a maintenance job:**
```python
# Run this notebook weekly on a cheap single-node cluster
# Add to Databricks Workflow with schedule: every Sunday 2am

tables_to_optimize = [
    "abfss://enriched@adlsprodcc.dfs.core.windows.net/transactions_streaming/",
    "abfss://enriched@adlsprodcc.dfs.core.windows.net/customer_360/",
    "abfss://curated@adlsprodcc.dfs.core.windows.net/transactions/",
]

for table_path in tables_to_optimize:
    print(f"Optimizing: {table_path}")
    spark.sql(f"OPTIMIZE delta.`{table_path}`")
    spark.sql(f"VACUUM delta.`{table_path}` RETAIN 168 HOURS")
    print(f"Done: {table_path}")
```

> ⚠️ **Gotcha:** Never run `VACUUM` with a retention period less than 7 days (168 hours). Doing so can delete files that are still being referenced by active streams using checkpoints (the stream's checkpoint references an older file version). This causes the stream to fail with `FileNotFoundException` on the next batch.
