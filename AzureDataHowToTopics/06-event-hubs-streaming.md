# Azure Event Hubs & Streaming
### 🟡 Intermediate → 🔴 Advanced

> *"Event Hubs is a distributed commit log.  
>  Partitions are not a performance knob — they are the unit of parallelism."*

---

## Q1. How do Event Hubs partitions work and how do you size them correctly?

**Scenario:** You're designing an ingestion pipeline for 100,000 IoT sensor events per second. Your team asks "how many partitions do we need?" The default is 4. Is that enough?

```
Event Hubs partition model:

  Event Hubs namespace
  └── event hub: eh-sensors (N partitions)
       ├── Partition 0: [e1, e2, e3, ...]   ← sequential, ordered within partition
       ├── Partition 1: [e4, e5, e6, ...]
       ├── Partition 2: [e7, e8, e9, ...]
       └── Partition 3: [e10, e11, ...]

  Routing:
  - No partition key → round-robin across partitions
  - With partition key (e.g., sensorId) → consistent hash → same sensor = same partition
    → ordered reads per sensor, parallel across sensors

  Sizing rule:
  - Each partition handles ~1 MB/s ingress, ~2 MB/s egress
  - 100K events/s × 1KB avg = 100 MB/s needed
  - 100 MB/s ÷ 1 MB/s per partition = 100 partitions minimum

  IMPORTANT: partition count is FIXED at creation (Standard tier)
             Premium/Dedicated: can increase partitions (not decrease)
             Plan for peak × 2 headroom
```

```bash
# Create Event Hubs namespace — Premium tier (higher throughput, private endpoints)
az eventhubs namespace create \
  --name eh-ns-prod \
  --resource-group rg-data \
  --location canadacentral \
  --sku Premium \
  --capacity 4 \             # Processing units (1 PU = 1 MB/s ingress per partition set)
  --minimum-tls-version 1.2 \
  --public-network-access Disabled    # Private Endpoint only

# Create event hub with enough partitions
az eventhubs eventhub create \
  --name eh-sensors \
  --namespace-name eh-ns-prod \
  --resource-group rg-data \
  --partition-count 32 \       # Fixed after creation on Standard; increasable on Premium
  --retention-time-in-hours 168   # 7 days retention for replay capability
  # Premium supports up to 168 hours (7 days) retention

# Check throughput limits
az eventhubs namespace show \
  --name eh-ns-prod \
  --resource-group rg-data \
  --query "{sku:sku.name, capacity:sku.capacity, throughputUnits:sku.capacity}"
```

| Tier | Throughput | Partitions | Retention | Private Endpoint |
|------|-----------|-----------|-----------|-----------------|
| Basic | 1 MB/s per TU | 2–32 | 1 day | ❌ |
| Standard | 1 MB/s per TU | 2–32 | 7 days | ❌ |
| Premium | 1 MB/s per PU | 2–100 | 90 days | ✅ |
| Dedicated | Unlimited | 2–1000+ | 90 days | ✅ |

> ⚠️ **Gotcha:** On the Standard tier, partition count is set at creation and **cannot be increased**. If you start with 4 partitions and later need 32, you must create a new event hub and replay all events. For production, start with at least 16–32 partitions even if current load is low.

---

## Q2. How do you use Event Hubs with Kafka protocol from existing Kafka producers?

**Scenario:** You have existing services using the Kafka SDK. You want to migrate ingestion to Event Hubs without rewriting any producer or consumer code.

```
Event Hubs Kafka endpoint — wire-compatible:

  Existing Kafka Producer (no code change)
  bootstrap.servers = eh-ns-prod.servicebus.windows.net:9093
  security.protocol = SASL_SSL
  sasl.mechanism    = PLAIN
  sasl.username     = $ConnectionString
  sasl.password     = <connection-string-with-EntityPath>
       │
       │ (looks exactly like a Kafka broker to the client)
       ▼
  Event Hubs namespace (Premium/Standard — Basic does NOT support Kafka)
       │ Topics = Event Hub names
       │ Consumer Groups = Event Hubs Consumer Groups
       │ Partitions = Event Hubs Partitions
       ▼
  Consumers (Kafka consumer group OR EventProcessorClient)
```

```python
# Kafka producer sending to Event Hubs (zero code change from Kafka)
from confluent_kafka import Producer

BOOTSTRAP_SERVERS = "eh-ns-prod.servicebus.windows.net:9093"
CONNECTION_STRING = "Endpoint=sb://eh-ns-prod.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=<key>;EntityPath=eh-sensors"

producer = Producer({
    "bootstrap.servers": BOOTSTRAP_SERVERS,
    "security.protocol": "SASL_SSL",
    "sasl.mechanism": "PLAIN",
    "sasl.username": "$ConnectionString",   # Literal string "$ConnectionString"
    "sasl.password": CONNECTION_STRING,
    "client.id": "sensor-producer-001"
})

producer.produce(
    topic="eh-sensors",          # Event Hub name
    key="sensor-001",            # Partition key — same sensor → same partition
    value=json.dumps(event_data).encode("utf-8")
)
producer.flush()
```

```python
# Kafka consumer reading from Event Hubs (same — no code change)
from confluent_kafka import Consumer

consumer = Consumer({
    "bootstrap.servers": BOOTSTRAP_SERVERS,
    "security.protocol": "SASL_SSL",
    "sasl.mechanism": "PLAIN",
    "sasl.username": "$ConnectionString",
    "sasl.password": CONNECTION_STRING,
    "group.id": "sensor-processor-group",   # Consumer group
    "auto.offset.reset": "earliest"
})

consumer.subscribe(["eh-sensors"])
while True:
    msg = consumer.poll(timeout=1.0)
    if msg is not None:
        process_event(json.loads(msg.value()))
```

> ⚠️ **Gotcha:** The `sasl.username` must be the literal string `"$ConnectionString"` (not your actual username). The `sasl.password` is the full connection string including `EntityPath`. If you omit `EntityPath` from the connection string, all Kafka topics are accessible — adding it scopes to a single event hub. For least-privilege, always include `EntityPath`.

---

## Q3. How do you enable Event Hubs Capture to automatically archive events to ADLS Gen2?

**Scenario:** Compliance requires all ingested sensor events to be stored for 7 years. Event Hubs default retention is 7 days. You need events automatically persisted to the data lake without writing any consumer code.

```
Event Hubs Capture:

  Event Hubs (eh-sensors)
  Events arrive at 100K/s
       │
       │ Capture: every 5 minutes OR when 300MB accumulated
       │ (whichever comes first)
       ▼
  ADLS Gen2 (adlsprodcc)
  raw/eventhubs-capture/{Namespace}/{EventHub}/{Partition}/{Year}/{Month}/{Day}/{Hour}/{Minute}/{Second}.avro

  File format: Apache Avro (binary, self-describing schema)
  Content: up to 300MB of events per file
  Latency: up to 5 minutes from event arrival to file appearance

  Downstream processing:
  ADF → reads Avro files → converts to Parquet → curated/
  Or Databricks reads Avro directly via spark.read.format("avro")
```

```bash
# Enable Capture on existing Event Hub
az eventhubs eventhub update \
  --name eh-sensors \
  --namespace-name eh-ns-prod \
  --resource-group rg-data \
  --enable-capture true \
  --capture-interval 300 \          # Capture every 300 seconds (5 min)
  --capture-size-limit 314572800 \  # 300MB in bytes
  --destination-name "EventHubArchive.AzureBlockBlob" \
  --storage-account "/subscriptions/<SUB>/resourceGroups/rg-data/providers/Microsoft.Storage/storageAccounts/adlsprodcc" \
  --blob-container raw \
  --archive-name-format "{Namespace}/{EventHub}/{PartitionId}/{Year}/{Month}/{Day}/{Hour}/{Minute}/{Second}"
  # This format creates a predictable path for ADF/Databricks to read

# Verify capture is enabled
az eventhubs eventhub show \
  --name eh-sensors \
  --namespace-name eh-ns-prod \
  --resource-group rg-data \
  --query "{captureEnabled:captureDescription.enabled, intervalSeconds:captureDescription.intervalInSeconds}"
```

```python
# Read captured Avro files in Databricks
df = spark.read.format("avro").load(
    "abfss://raw@adlsprodcc.dfs.core.windows.net/eh-ns-prod/eh-sensors/*/*/*/*/*/*"
)

# Avro schema is embedded in the file — inspect it
print(df.schema)

# Convert to Delta for downstream use
df.write.format("delta") \
    .mode("append") \
    .partitionBy("year", "month", "day") \
    .save("abfss://curated@adlsprodcc.dfs.core.windows.net/sensor-events/")
```

> ⚠️ **Gotcha:** Event Hubs Capture grants the Event Hubs namespace's managed identity `Storage Blob Data Contributor` on the target storage account automatically — but only if you enable Capture through the Azure Portal. When enabling via CLI or ARM template, you must manually assign the role. Missing this causes the Capture to silently fail: events are processed but no Avro files appear.

---

## Q4. How do consumer groups work in Event Hubs and why do you need multiple?

**Scenario:** You have two consumers reading from `eh-transactions`: a Databricks streaming job for fraud detection and an Azure Function for real-time notifications. If they share one consumer group, what happens?

```
Consumer group = independent cursor through the event stream:

  Event Hub (eh-transactions) — 8 partitions
  ├── Consumer Group: $Default
  │   └── Databricks streaming job
  │       reads partition 0–7 at its own pace
  │       checkpoint: offset 145,230
  │
  └── Consumer Group: cg-notifications
      └── Azure Function (Event Processor)
          reads partition 0–7 at ITS own pace
          checkpoint: offset 145,187  ← different position!

  Each consumer group maintains its own offset per partition.
  They read the SAME events but advance independently.

  If they shared $Default:
  Consumer 1 reads offset 100 → advances cursor to 101
  Consumer 2 tries to read → gets offset 101 (skips 100!)
  Result: half the events processed by each consumer
  → fraud detection misses some transactions ❌
  → notifications miss some transactions ❌
```

```bash
# Create separate consumer group for each independent consumer
az eventhubs consumer-group create \
  --name cg-databricks-fraud \
  --eventhub-name eh-transactions \
  --namespace-name eh-ns-prod \
  --resource-group rg-data

az eventhubs consumer-group create \
  --name cg-azure-functions \
  --eventhub-name eh-transactions \
  --namespace-name eh-ns-prod \
  --resource-group rg-data

# Maximum consumer groups per event hub: 20 (Standard), unlimited (Premium/Dedicated)

# List consumer groups
az eventhubs consumer-group list \
  --eventhub-name eh-transactions \
  --namespace-name eh-ns-prod \
  --resource-group rg-data \
  --output table
```

```python
# Databricks: specify consumer group
df_fraud = (
    spark.readStream
    .format("eventhubs")
    .options(**{
        "eventhubs.connectionString": CONNECTION_STRING,
        "eventhubs.consumerGroup": "cg-databricks-fraud",   # Dedicated CG
        "eventhubs.startingPosition": json.dumps({"offset": "-1", "seqNo": -1, "enqueuedTime": None, "isInclusive": True})
    })
    .load()
)
```

> ⚠️ **Gotcha:** A single consumer group should have at most **one active reader per partition**. If two Databricks jobs read the same consumer group and the same partition simultaneously, one will fail with `ReceiverDisconnectedException` — Event Hubs allows only one active receiver per consumer group + partition combination. Use separate consumer groups, not separate readers on the same group.
