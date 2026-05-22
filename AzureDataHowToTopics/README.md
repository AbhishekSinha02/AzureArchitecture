# Azure Data How-To — Scenario Library

> *"Data architecture is not about storing data.  
>  It's about making the right data available to the right system at the right latency — reliably."*

### 🟢 Beginner → 🟡 Intermediate → 🔴 Advanced

---

## Files

| # | File | What's inside |
|---|------|---------------|
| 01 | [adls-data-lake.md](01-adls-data-lake.md) | ADLS Gen2 zones, ACLs, lifecycle policies, hierarchical namespace |
| 02 | [azure-data-factory.md](02-azure-data-factory.md) | Pipelines, SHIR, watermark, CDC, checksum validation |
| 03 | [synapse-analytics.md](03-synapse-analytics.md) | Dedicated vs Serverless SQL Pool, PolyBase, Spark integration |
| 04 | [databricks-delta-lake.md](04-databricks-delta-lake.md) | Delta Lake, streaming, MERGE/upsert, OPTIMIZE, Z-ORDER |
| 05 | [cosmosdb.md](05-cosmosdb.md) | Partition key design, consistency levels, Change Feed, multi-master |
| 06 | [event-hubs-streaming.md](06-event-hubs-streaming.md) | Partitions, Kafka surface, Capture to ADLS, consumer groups |
| 07 | [azure-sql-database.md](07-azure-sql-database.md) | HA tiers, elastic pools, migration, Query Store tuning |
| 08 | [redis-cache.md](08-redis-cache.md) | Cache-aside, tier selection, session state, eviction troubleshooting |
| 09 | [data-migration.md](09-data-migration.md) | AzCopy, Data Box, DMS CDC, zero-downtime cutover, validation |
| 10 | [commands-reference.md](10-commands-reference.md) | Every az / AzCopy / SQL / KQL command with context |

---

## How to Read

Every answer follows the validated pattern:
1. **Scenario** — the real situation you're in
2. **Diagram** — ASCII topology or flow
3. **Answer** — bullet points, **services bolded**
4. **Commands** — runnable, annotated with expected output
5. **⚠️ Gotcha** — the one thing that breaks this in production
6. **💡 Deep dive hint** — follow-up thread topic
