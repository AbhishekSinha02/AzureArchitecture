# Azure Databases — Interview Questions

> Covers: Azure SQL, CosmosDB, PostgreSQL Flexible, Redis, service selection, partitioning, consistency models.
> Levels: Beginner → Intermediate → Advanced

---

## BEGINNER

**Q1. What are the main database options on Azure and when would you choose each?**

**Answer:**

| Service | Type | Best for |
|---------|------|----------|
| Azure SQL Database | Relational (SQL Server) | OLTP, existing SQL Server workloads, structured data |
| Azure SQL Managed Instance | Relational (SQL Server — near-full compat) | Lift-and-shift SQL Server with minimal change |
| Azure Database for PostgreSQL Flexible Server | Relational (open source) | Cloud-native relational, open-source preference |
| Azure Database for MySQL Flexible Server | Relational (open source) | Web apps, WordPress, LAMP stack |
| Azure CosmosDB | Multi-model NoSQL | Global distribution, variable schema, high throughput |
| Azure Cache for Redis | In-memory key-value | Caching, session state, real-time leaderboards |
| Azure Synapse Analytics | Analytical (OLAP) | Data warehouse, large-scale analytics, ad-hoc queries |
| Azure Data Explorer (ADX) | Time-series / log analytics | IoT telemetry, logs, time-series queries |

**Decision framework:**
- Relational + existing SQL Server workloads → **Azure SQL MI**
- New relational + open source → **PostgreSQL Flexible Server**
- Global, variable schema, high throughput → **CosmosDB**
- Sub-millisecond reads, caching → **Redis**
- Analytics + reporting + PB scale → **Synapse**

---

**Q2. What is CosmosDB and what consistency levels does it offer?**

**Answer:**

CosmosDB is Azure's globally distributed, multi-model NoSQL database. It supports:
- **APIs**: NoSQL (native), MongoDB, Cassandra, Gremlin (graph), Table
- **Global distribution**: replicate to any Azure region in minutes
- **Multi-master writes**: write to any region, Azure handles conflict resolution
- **Guaranteed SLAs**: latency, throughput, consistency, availability

**5 Consistency Levels (strong → weak):**

| Level | Guarantee | Use case |
|-------|-----------|----------|
| **Strong** | Always reads latest write | Financial transactions requiring 100% accuracy |
| **Bounded Staleness** | Reads lag by X operations or T time | Near-realtime dashboards with controlled lag |
| **Session** | Read your own writes (within session) | Most web apps — single user consistent view |
| **Consistent Prefix** | No out-of-order reads, may be stale | Social feeds, activity streams |
| **Eventual** | Fastest, cheapest, no ordering guarantee | Click counters, likes, non-critical metrics |

**Default**: Session consistency — covers 90%+ of use cases.

---

**Q3. What is the difference between RU/s (Request Units) and DTUs in Azure databases?**

**Answer:**

**RU/s (CosmosDB):**
- Abstracted unit representing CPU + IOPS + memory cost of an operation
- 1 RU = cost of reading 1KB item by partition key
- You provision or autoscale RU/s — CosmosDB guarantees that throughput
- Autoscale: set max RU/s, CosmosDB scales 10% → 100% based on demand

**DTUs (Azure SQL Database — legacy):**
- Blended unit combining CPU, I/O, memory
- Simple to understand but hard to predict performance
- Replaced by **vCore model** (recommended): explicitly choose CPU cores + memory

**vCore model is preferred because:**
- Predictable performance (you know exactly what hardware you're getting)
- Hybrid Benefit: use existing SQL Server licenses → ~55% savings
- Can configure independently: 4 vCores + 20GB RAM

---

**Q4. What is Azure Redis Cache and what are typical use cases?**

**Answer:**

Azure Cache for Redis is a managed, in-memory data store based on open-source Redis. Sub-millisecond read/write latency.

**Tiers:**
- **Basic**: Single node, dev/test only (no SLA)
- **Standard**: Primary + replica with failover, production SLA
- **Premium**: Persistence, clustering, VNet injection, geo-replication
- **Enterprise / Enterprise Flash**: Active geo-replication, largest memory (multi-TB with Flash)

**Common use cases:**

```
1. Cache-aside (most common)
   App → check Redis → miss → query DB → write to Redis → return
   TTL: 60s–5min depending on data freshness requirement

2. Session store
   Replace sticky sessions — store session tokens in Redis
   Multiple app instances can share session state

3. Rate limiting
   Redis INCR + EXPIRE: count requests per user per window

4. Pub/Sub messaging
   Lightweight fan-out within a region (not durable — use Service Bus for durability)

5. Leaderboard / ranking
   Redis Sorted Sets: O(log N) operations, perfect for real-time scoring

6. Distributed lock
   Redis SETNX (set if not exists) — distributed mutex pattern
```

---

## INTERMEDIATE

**Q5. How do you choose a partition key in CosmosDB and why does it matter?**

**Answer:**

The partition key is the most important design decision in CosmosDB. It determines how data is distributed across physical partitions (each max 50GB, max 10,000 RU/s).

**Rules for a good partition key:**
1. **High cardinality**: Enough distinct values to spread data evenly (e.g., userId, orderId — not status, category)
2. **Even read/write distribution**: Avoid "hot partitions" where one key gets 90% of traffic
3. **Matches your most common query patterns**: Filter by partition key → single-partition query (fast + cheap) vs. cross-partition query (fan-out, expensive)
4. **Immutable**: Never changes after the document is written

**Examples:**

| Use case | Bad key | Good key |
|----------|---------|----------|
| E-commerce orders | `status` (only 5 values) | `customerId` or `orderId` |
| IoT telemetry | `deviceType` (3 values) | `deviceId` |
| User activity | `eventType` (10 types) | `userId` |
| Multi-tenant app | `tenantSize` (3 tiers) | `tenantId` |

**Synthetic partition key pattern** (when no single field is ideal):
```json
// Combine two fields for better distribution
{
  "partitionKey": "user123_2024-01",
  "userId": "user123",
  "month": "2024-01"
}
```

**Hot partition symptoms:** RU throttling (429 errors) on specific partition, while others are idle.

---

**Q6. Explain how to implement multi-region writes in CosmosDB and how conflicts are resolved.**

**Answer:**

**Setup:**
1. Enable multi-region writes (multi-master) in CosmosDB account settings
2. Add write regions (e.g., Canada Central + Canada East + East US)
3. Configure conflict resolution policy

**Conflict types and resolution:**

```
Scenario: User updates document in Canada Central and East US simultaneously

Default: Last Writer Wins (LWW)
  → Azure uses _ts (timestamp) field
  → Higher timestamp wins
  → Simple, works for most cases

Custom: Merge procedure (stored procedure-based)
  → You write a stored procedure defining merge logic
  → Example: for financial ledgers, sum both increments rather than overwrite

Manual: Keep conflicting versions
  → All conflicts land in conflict feed
  → Your application reads conflict feed and resolves manually
  → Use for high-value data where automated merge is risky
```

**Application-level pattern:**
```csharp
// Optimistic concurrency via ETag
var item = await container.ReadItemAsync<MyDoc>(id, partitionKey);
var etag = item.ETag;

// Update attempt — will fail if another writer modified it
var options = new ItemRequestOptions { IfMatchEtag = etag };
await container.ReplaceItemAsync(updatedDoc, id, partitionKey, options);
// If conflict: catch CosmosException with status 412, retry
```

---

**Q7. What is the difference between Azure SQL Database and Azure SQL Managed Instance? When would you use each?**

**Answer:**

| Feature | Azure SQL Database | Azure SQL Managed Instance |
|---------|-------------------|---------------------------|
| Compatibility | ~95% SQL Server | ~99% SQL Server |
| SQL Agent | No (use Elastic Jobs) | Yes |
| Linked Servers | No | Yes |
| CLR | No | Yes |
| Cross-DB queries | No (same server only) | Yes |
| SSRS/SSIS/SSAS | No | Limited (SSIS via Azure-SSIS IR) |
| Network | Public or Private Endpoint | Always inside VNet (subnet required) |
| Deployment | Fast (minutes) | Slower (up to 6 hours for provisioning) |
| Cost | Lower | Higher (~2x for same vCores) |

**Choose Azure SQL Database when:**
- New cloud-native application
- No legacy SQL Server features needed
- Want per-database scaling (serverless or hyperscale)
- Budget is a priority

**Choose SQL Managed Instance when:**
- Migrating existing SQL Server that uses CLR, SQL Agent, linked servers
- Application relies on cross-database transactions
- On-premises tooling (DBA scripts) must work unchanged

---

**Q8. How does read-scaling work in Azure SQL Database and CosmosDB?**

**Answer:**

**Azure SQL Database — Read Scale-Out:**
- **Business Critical tier**: Includes a built-in readable secondary (at no extra cost)
- Configure connection string with `ApplicationIntent=ReadOnly` → queries route to secondary
- Use for: reporting queries, BI tools, read-heavy analytics without impacting primary

- **Hyperscale tier**: Up to 4 named read replicas (separately billable)
- **Geo-replicas**: Active geo-replication (up to 4) for read scaling across regions

**CosmosDB — Read scaling:**
- **Multi-region read**: Add read regions — reads can be served from nearest region
- **Preferred locations** in SDK: set region priority order, SDK auto-routes
- **Consistency** matters: Strong consistency requires reading from primary; Session/Eventual can read from replica
- Adding a read region increases RU capacity linearly (10K RU/s in 3 regions = effectively 30K RU/s read capacity)

**Example CosmosDB SDK region routing:**
```csharp
var options = new CosmosClientOptions
{
    ApplicationPreferredRegions = new List<string>
    {
        "Canada Central",   // try this first
        "Canada East",       // fallback
        "East US"            // last resort
    }
};
```

---

## ADVANCED

**Q9. Design the data layer for a real-time payment processing system handling 50,000 transactions per second with 99.99% availability for a Canadian bank.**

**Answer:**

**Requirements clarification:**
- 50K TPS = ~180M tx/hour = high-throughput OLTP
- 99.99% SLA = 52 min/year downtime — requires multi-region active-active
- OSFI B-10 + PIPEDA: Canada Central + Canada East only, immutable audit logs
- ACID required: financial transactions must be atomic

**Architecture:**

```
Write path (50K TPS):
  API Gateway (APIM)
    → Service Bus (guaranteed delivery, buffer)
    → Azure Functions (idempotent processors, 100 instances via KEDA)
    → CosmosDB NoSQL API
        Partition key: accountId
        Consistency: Strong (financial accuracy)
        Multi-region: Canada Central (primary write) + Canada East (secondary)
        Autoscale: 50K RU/s → 500K RU/s on demand
        
Read path (balance lookups, history):
  → Redis Cache (sub-ms balance lookups, TTL 5s)
  → CosmosDB (miss or stale: read account document)
  → ADLS Gen2 (historical archive >90 days)
```

**Idempotency pattern (critical for payments):**
```csharp
// Every payment has a unique idempotency key
public async Task<PaymentResult> ProcessPayment(PaymentRequest req)
{
    // Check if already processed
    var existing = await cosmos.ReadItemAsync<Payment>(
        req.IdempotencyKey, new PartitionKey(req.AccountId));
    if (existing != null) return existing.Result;  // Return cached result
    
    // Process atomically
    var tx = await cosmos.CreateTransactionalBatch(new PartitionKey(req.AccountId))
        .UpsertItem(debitEntry)
        .UpsertItem(paymentRecord)
        .ExecuteAsync();
}
```

**Scaling math:**
- CosmosDB: 1KB document write = ~10 RU → 50K TPS = 500K RU/s needed
- Autoscale max: 1M RU/s → headroom for 100K TPS burst
- Redis: Azure Cache for Redis Premium P4 → 120GB, 250K ops/sec
- Functions: KEDA scaling on Service Bus queue depth (target 100 messages/instance)

**DR strategy:**
- Canada Central: primary writes + reads
- Canada East: secondary writes (active-active via CosmosDB multi-master) + reads
- RPO: near-zero (CosmosDB replication is synchronous for Bounded Staleness)
- RTO: <1 minute (Traffic Manager health probe → automatic failover)

---

**Q10. Explain the CAP theorem and how it applies to CosmosDB's consistency levels.**

**Answer:**

**CAP Theorem:** A distributed system can only guarantee two of three:
- **C**onsistency: All nodes see the same data at the same time
- **A**vailability: Every request gets a response (not necessarily latest data)
- **P**artition tolerance: System continues if network splits nodes

In practice, distributed systems must tolerate network partitions — so the real trade-off is **C vs A** during a partition.

**CosmosDB's position:**

| Consistency Level | CP or AP? | Trade-off |
|------------------|-----------|-----------|
| **Strong** | CP | Reads may fail or be slow during partition (consistency > availability) |
| **Bounded Staleness** | CP (bounded) | Reads lag by defined window; partition may block |
| **Session** | AP | Your session sees your writes; others may see stale |
| **Consistent Prefix** | AP | No stale reads, but may lag; always available |
| **Eventual** | AP | Maximum availability, minimum consistency |

**Banking context:**
- Account balance reads: **Bounded Staleness** (lag <1s acceptable, need accuracy)
- User profile reads: **Session** (user sees their own updates immediately)
- Analytics/dashboards: **Eventual** (stale by seconds is fine, availability more important)
- Legal/compliance writes: **Strong** (no room for inconsistency — pay the latency cost)

**PACELC extension (more precise):** CosmosDB also optimizes for latency when no partition exists. Strong consistency = higher latency even under normal conditions (requires cross-region acknowledgment in multi-master).

---

**Q11. How do you handle database connection pool exhaustion in a high-concurrency Azure SQL scenario?**

**Answer:**

**Symptoms:** `Timeout expired — The timeout period elapsed before obtaining a connection from the pool`

**Root causes and fixes:**

```
Root cause 1: Pool too small
  Default: max 100 connections per connection string
  Fix: increase max pool size cautiously
  "Server=...;Max Pool Size=200;Connection Timeout=30;"
  WARNING: Each connection = SQL Server worker thread, memory cost

Root cause 2: Connections not returned to pool
  Fix: Always use using() or await using() — ensures Dispose() called
  using (var conn = new SqlConnection(connStr))
  {
      await conn.OpenAsync();
      // ... work ...
  }  // Dispose() returns connection to pool

Root cause 3: Long transactions holding connections
  Fix: Keep transactions as short as possible
  Don't do: Open connection → call external API → commit transaction
  Do: Call external API → Open connection → commit transaction

Root cause 4: Too many application instances × pool size > SQL max connections
  Azure SQL max connections: 30 × vCores (General Purpose)
  5 AKS pods × 200 pool size = 1000 connections > 4 vCore limit (120)
  Fix: Use PgBouncer (PostgreSQL) or SQL Server connection proxy
  OR: Reduce pool size per pod to fit within server max
  OR: Scale up SQL tier (more vCores)

Root cause 5: Thundering herd on startup
  Fix: Randomize startup delays, use circuit breaker (Polly) with exponential backoff
```

**Azure SQL-specific mitigation:**
- **Serverless tier**: auto-pauses when idle, scales vCores on demand — avoids over-provisioning
- **Read replicas**: route read-only queries away from primary (`ApplicationIntent=ReadOnly`)
- **Azure SQL Hyperscale**: separates compute from storage, add read replicas independently
