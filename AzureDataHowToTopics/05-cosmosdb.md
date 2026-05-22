# Azure CosmosDB
### 🟡 Intermediate → 🔴 Advanced

> *"In CosmosDB, the wrong partition key is a debt you pay forever.  
>  The right partition key is invisible — because everything just works."*

---

## Q1. How do you choose the right partition key for CosmosDB?

**Scenario:** You're designing a CosmosDB container for a banking app that stores customer transactions. Developers suggest using `customerId` as the partition key. The lead architect pushes back. Who is right and why?

```
Partition key evaluation:

  Option A: customerId (e.g., "cust-001")
  ─────────────────────────────────────────
  Total customers: 10,000
  Transactions per customer: 500 avg, 50,000 max (VIP)
  ✅ Most queries are per-customer → data locality
  ❌ Hot partition: VIP customer with 50K transactions = overloaded partition
  ❌ Max partition size = 20GB → VIP customer with millions of transactions hits limit

  Option B: transactionId (GUID)
  ─────────────────────────────────────────
  ✅ Perfect distribution — no hot partitions
  ❌ All queries scan multiple partitions (fan-out = expensive)
  ❌ Fetching all transactions for a customer = cross-partition query

  Option C: /customerId + /transactionDate (composite/synthetic key)
  ─────────────────────────────────────────────────────────────────
  Partition key: "cust-001_2024-01"  (customerId + year-month)
  ✅ Queries for "all Jan 2024 transactions for customer" = single partition
  ✅ Distributes VIP customers across time partitions
  ✅ No single partition exceeds 20GB
  ✅ Time-based expiry via TTL aligns with partition boundary
  → Best choice for time-series financial data
```

```python
# Create CosmosDB container with composite partition key structure
import azure.cosmos.cosmos_client as cosmos_client

client = cosmos_client.CosmosClient(url=COSMOS_URL, credential=COSMOS_KEY)
db = client.get_database_client("banking")

db.create_container_if_not_exists(
    id="transactions",
    partition_key=PartitionKey(path="/partitionKey"),   # Synthetic key field
    offer_throughput=10000   # RUs — or use autoscale below
)

# Application code: always set partitionKey on document write
def write_transaction(tx: dict):
    tx["partitionKey"] = f"{tx['customerId']}_{tx['transactionDate'][:7]}"
    # e.g., "cust-001_2024-01"
    container.upsert_item(tx)
```

```bash
# Create CosmosDB account with autoscale
az cosmosdb create \
  --name cosmos-prod-cc \
  --resource-group rg-data \
  --locations regionName=canadacentral failoverPriority=0 isZoneRedundant=true \
  --locations regionName=canadaeast failoverPriority=1 isZoneRedundant=false \
  --default-consistency-level Session \
  --enable-automatic-failover true

# Create database + container with autoscale RU (400–40000 RU, scales automatically)
az cosmosdb sql container create \
  --account-name cosmos-prod-cc \
  --resource-group rg-data \
  --database-name banking \
  --name transactions \
  --partition-key-path "/partitionKey" \
  --max-throughput 40000   # Autoscale max — billed for actual usage
```

> ⚠️ **Gotcha:** The partition key cannot be changed after container creation. It is immutable. If you choose the wrong key, you must create a new container, migrate all data with ADF or Spark, and swap the application connection. Plan this upfront — changing it later costs days of migration work.

---

## Q2. How do CosmosDB consistency levels work, and which should you use for a banking app?

**Scenario:** Your payment service writes a transaction to CosmosDB in Canada Central. A compliance reporting service in Canada East reads the same document 100ms later and gets the old value. The auditor is asking why the report doesn't match what the customer sees.

```
Five consistency levels (strongest → weakest):

  Strong         : Read always returns latest committed write
                   Every read goes to primary region only
                   ✅ Banking, financial compliance, inventory
                   ❌ 2× higher write latency, no multi-region reads

  Bounded Staleness: Reads lag by at most K versions or T seconds
                   e.g., max 5s behind primary
                   ✅ Near-real-time analytics where slight lag OK
                   ❌ Still complex — avoid unless you need it

  Session (DEFAULT): Consistent within a session (client connection)
                   Read-your-own-writes guaranteed
                   ✅ Best for user-facing apps (user sees their own actions)
                   ❌ Different clients may see different versions briefly

  Consistent Prefix: Reads never see out-of-order writes
                   (no guarantee of recency)

  Eventual        : No ordering guarantees at all
                   ✅ Lowest latency, highest throughput
                   ❌ Reads may return stale or out-of-order data

  For Canadian banking:
  Customer portal → Session (sees own writes, reasonable latency)
  Fraud detection → Strong (must see latest, no stale data)
  Analytics reports → Bounded Staleness (slight lag acceptable)
  Audit compliance  → Strong (regulatory requirement)
```

```bash
# Set consistency level at account level
az cosmosdb update \
  --name cosmos-prod-cc \
  --resource-group rg-data \
  --default-consistency-level Strong
# Note: account-level setting is the MAXIMUM consistency
# Clients can request WEAKER consistency per request — not stronger

# Override per-request in Python SDK (lower consistency for analytics reads)
from azure.cosmos import ConsistencyLevel

container.read_item(
    item="tx-001",
    partition_key="cust-001_2024-01",
    response_hook=lambda resp, headers: None,
    headers={"x-ms-consistency-level": "Session"}   # Override for this read
)
```

> ⚠️ **Gotcha:** `Strong` consistency only applies within a single write region. If you enable multi-region writes (multi-master), CosmosDB automatically downgrades `Strong` to `BoundedStaleness` — you cannot have `Strong` consistency with multi-region writes. For OSFI compliance in banking, this means you must choose: either `Strong` with a single write region, or multi-master with `BoundedStaleness`.

---

## Q3. How do you use CosmosDB Change Feed to trigger event-driven processing?

**Scenario:** When a transaction is written to CosmosDB, you need to: (1) update a customer summary record, (2) publish an event to Service Bus, (3) check for fraud signals. You don't want the payment service to call 3 downstream services directly.

```
Change Feed — decoupled event-driven architecture:

  Payment API
       │ writes transaction to CosmosDB container
       ▼
  CosmosDB (transactions container)
       │ Change Feed: ordered list of ALL writes (not deletes by default)
       │
       ├─ Azure Functions (Change Feed trigger)
       │    │ triggered per batch of changed items
       │    ├── update customer_summary container
       │    └── publish to Service Bus topic
       │
       └─ Databricks Structured Streaming
            │ reads change feed as a Kafka-like stream
            └── fraud ML model scoring (near-real-time)

  Benefits:
  ✅ Payment API has one responsibility: write the transaction
  ✅ Downstream consumers independently scale and fail
  ✅ Change Feed retains events for the container's retention period
```

```csharp
// Azure Function triggered by CosmosDB Change Feed
[FunctionName("ProcessTransactionChangeFeed")]
public static async Task Run(
    [CosmosDBTrigger(
        databaseName: "banking",
        containerName: "transactions",
        Connection = "CosmosDbConnection",
        LeaseContainerName = "leases",    // Tracks which items were processed
        CreateLeaseContainerIfNotExists = true)]
    IReadOnlyList<TransactionDocument> changes,
    ILogger log)
{
    foreach (var transaction in changes)
    {
        log.LogInformation($"Processing change: {transaction.TransactionId}");

        // 1. Update customer summary
        await UpdateCustomerSummaryAsync(transaction);

        // 2. Publish to Service Bus
        await serviceBusClient.SendMessageAsync(
            new ServiceBusMessage(JsonSerializer.Serialize(transaction))
        );
    }
}
```

```python
# Databricks: read CosmosDB Change Feed as a stream
df_changes = (
    spark.readStream
    .format("cosmos.oltp.changeFeed")
    .option("spark.cosmos.accountEndpoint", COSMOS_ENDPOINT)
    .option("spark.cosmos.accountKey", COSMOS_KEY)
    .option("spark.cosmos.database", "banking")
    .option("spark.cosmos.container", "transactions")
    .option("spark.cosmos.changeFeed.startFrom", "Now")
    .load()
)

# Run fraud scoring on each micro-batch
def score_fraud(batch_df, batch_id):
    scored = fraud_model.transform(batch_df)
    scored.filter("fraud_score > 0.8") \
          .write.format("delta") \
          .mode("append") \
          .save("abfss://enriched@adlsprodcc.dfs.core.windows.net/fraud_alerts/")

df_changes.writeStream \
    .foreachBatch(score_fraud) \
    .option("checkpointLocation", ".../checkpoints/fraud/") \
    .start()
```

> ⚠️ **Gotcha:** CosmosDB Change Feed does NOT include deletes by default. If your app deletes documents and downstream consumers need to react (e.g., remove from a search index), use **soft deletes**: add an `isDeleted: true` field and update the TTL to expire the document after a delay. The Change Feed picks up the update event with `isDeleted: true`.

---

## Q4. How do you set up CosmosDB multi-region writes for active-active HA?

**Scenario:** Your payment service must achieve 99.99% availability with < 50ms write latency globally. A single-region write model means Canadian East users write to Canada Central (adds ~10ms) and any region failure causes write unavailability.

```
Single-region write (current):        Multi-region write (target):
  Canada Central: write region         Canada Central: write ✅
  Canada East: read-only replica       Canada East:    write ✅

  User in Canada East → write          User in Canada East → writes to
  → routed to Canada Central           local region (<5ms)
  → 10ms extra latency
  → Central region down = no writes    → Either region down = still writes

  Conflict resolution needed:
  Last-Write-Wins (LWW): CosmosDB keeps item with highest _ts (timestamp)
  Custom conflict resolution: write stored procedure to merge conflicts
```

```bash
# Enable multi-region writes on existing account
az cosmosdb update \
  --name cosmos-prod-cc \
  --resource-group rg-data \
  --locations regionName=canadacentral failoverPriority=0 isZoneRedundant=true \
  --locations regionName=canadaeast failoverPriority=1 isZoneRedundant=true \
  --enable-multiple-write-locations true \
  --default-consistency-level BoundedStaleness
  # Strong consistency is not compatible with multi-master — must use BoundedStaleness or weaker

# Set bounded staleness parameters (max lag for reads)
az cosmosdb update \
  --name cosmos-prod-cc \
  --resource-group rg-data \
  --max-staleness-prefix 100000 \   # Max 100K operations behind
  --max-interval 5                  # Max 5 seconds behind
```

```python
# SDK: use preferred regions for write locality
from azure.cosmos import CosmosClient, PartitionKey
from azure.cosmos.diagnostics import CosmosDiagnostics

client = CosmosClient(
    url=COSMOS_ENDPOINT,
    credential=COSMOS_KEY,
    preferred_locations=["Canada East", "Canada Central"]
    # SDK writes to nearest region, fails over automatically
)
```

> ⚠️ **Gotcha:** Multi-region writes require your application to handle **write conflicts**. If two clients in different regions update the same document within the replication window, CosmosDB applies the conflict resolution policy (default: Last-Write-Wins based on `_ts`). For financial records where the amount matters, implement a custom conflict resolution stored procedure that rejects or merges conflicting updates.
