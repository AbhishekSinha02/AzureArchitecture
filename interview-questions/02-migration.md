# Azure Migration — Interview Questions

> Covers: 6Rs framework, Azure Migrate, Data Box, CDC, zero-downtime strategies, database migration.
> Levels: Beginner → Intermediate → Advanced

---

## BEGINNER

**Q1. What are the 6Rs of cloud migration?**

**Answer:**

| Strategy | Description | When to use |
|----------|-------------|-------------|
| **Rehost** | Lift & shift to IaaS (same app, Azure VM) | Time pressure, low complexity, large legacy estate |
| **Replatform** | Minor optimizations (e.g., IIS → App Service) | Quick wins without full refactor |
| **Repurchase** | Move to SaaS (e.g., Salesforce, Dynamics 365) | Commodity workloads, reduce ops burden |
| **Refactor** | Re-architect for cloud-native (AKS, event-driven) | Competitive differentiation, long-term investment |
| **Retire** | Decommission unused systems | Reduce licensing + ops cost |
| **Retain** | Keep on-premises | Compliance, latency, legacy dependency |

**Decision rule of thumb:**
- Time pressure + low complexity → **Rehost**
- Cost optimization target → **Replatform**
- Competitive differentiation → **Refactor**
- Regulatory constraint → **Retain** or **Rehost in sovereign region**

---

**Q2. What is Azure Migrate and what does it do?**

**Answer:**

Azure Migrate is a hub of tools for discovering, assessing, and migrating on-premises workloads to Azure.

Key components:
- **Discovery and Assessment**: Agentless discovery of VMs, servers, databases, and web apps. Generates Azure readiness reports and right-sized VM recommendations.
- **Server Migration**: Replicates on-prem VMs (VMware, Hyper-V, physical) to Azure. Supports test migrations before cutover.
- **Database Assessment**: SQL Server compatibility report for Azure SQL / Managed Instance.
- **Web App Assessment**: ASP.NET compatibility for Azure App Service.

**Typical flow:**
```
Deploy Azure Migrate appliance on-prem
→ Discovery (1–7 days)
→ Assessment report (sizing, cost estimate, readiness)
→ Replication (continuous delta sync)
→ Test migration
→ Cutover
```

---

**Q3. What is Azure Data Box and when would you use it over network transfer?**

**Answer:**

Azure Data Box is a physical device Microsoft ships to you. You copy data to it locally, ship it back, and Microsoft uploads it to Azure Storage.

**When to use Data Box vs network transfer:**

| Method | Use when |
|--------|----------|
| AzCopy / ADF | <10TB, good network bandwidth, time not critical |
| Data Box (80TB) | 40TB–80TB, or network would take >2 weeks |
| Data Box Heavy (1PB) | PB-scale migrations |
| Data Box Disk (8TB SSD) | Smaller volumes, fast local USB copy |

**Rule of thumb:** If copying over the network would take more than 1 week, Data Box is faster and often cheaper.

**Example calculation:**
- 100TB at 1Gbps dedicated link = ~222 hours = 9+ days network only
- Data Box: copy locally (fast), ship (2–3 days), upload at Microsoft's datacenter speed

---

**Q4. What is the difference between online and offline migration?**

**Answer:**

- **Offline migration**: Source system is stopped (or in read-only mode) during migration. Simpler, guarantees consistency. Acceptable when downtime window exists (e.g., maintenance window, batch systems).

- **Online migration (zero-downtime)**: Source system keeps running. Delta changes are continuously synced to the target. Requires a final "cutover" when the lag reaches near zero.

**Tools for online migration:**
- Databases: Azure Database Migration Service (DMS) with continuous sync
- Files: AzCopy with incremental sync
- VMs: Azure Migrate continuous replication (agentless)

Online migration is required when SLA demands <4 hours downtime or the system is business-critical.

---

## INTERMEDIATE

**Q5. Walk me through a production database migration from on-premises SQL Server to Azure SQL Managed Instance with minimal downtime.**

**Answer:**

**Step 1 — Assess**
- Run Data Migration Assistant (DMA) on source SQL Server
- Identify: deprecated features, unsupported syntax, linked servers, CLR objects
- Azure SQL MI supports ~99% of SQL Server features — CLR, SQL Agent, linked servers

**Step 2 — Sizing**
- Map cores + RAM to Business Critical or General Purpose tier
- Business Critical: in-memory OLTP, faster I/O (SSD), readable secondary included
- General Purpose: most workloads, cheaper

**Step 3 — Network**
- SQL MI lives inside your VNet (it's a PaaS resource with private endpoint by default)
- Configure Private Endpoint + Private DNS Zone
- On-prem connectivity: ExpressRoute or site-to-site VPN

**Step 4 — Online migration with DMS**
```
Source: SQL Server (on-prem)
  → DMS reads transaction log continuously (log-shipping-based)
  → Applies to Azure SQL MI (continuous sync)
  → Monitor replication lag in DMS portal
  → When lag < 30 seconds: initiate cutover
  → DMS performs final transaction log restore
  → Update connection strings → done
```

**Step 5 — Validate**
- Row counts match on all tables
- Stored procedures and jobs running correctly
- Application smoke test with production traffic pattern

**Rollback plan:** Keep source SQL Server running for 72 hours post-cutover. Connection string rollback = instant.

---

**Q6. How do you migrate 50TB of files from on-premises NAS to Azure Blob / ADLS Gen2 with data integrity validation?**

**Answer:**

**Tools:** AzCopy (primary), ADF Self-hosted IR (orchestration), Azure Data Box (if bandwidth is constrained)

**Integrity validation approach:**

```
Step 1: Generate MD5 checksums on source
  # Linux
  find /nas/data -type f -exec md5sum {} \; > source_checksums.txt
  # Windows
  Get-ChildItem -Recurse | Get-FileHash -Algorithm MD5 > source_checksums.csv

Step 2: Transfer with AzCopy (preserves metadata + MD5)
  azcopy copy '/nas/data/*' \
    'https://storageaccount.blob.core.windows.net/container?<SAS>' \
    --recursive --check-md5=FailIfDifferent \
    --log-level=ERROR

Step 3: AzCopy generates transfer manifest (JSON log)
  # Review: azcopy jobs show <job-id>
  # Failed files are logged — retry individually

Step 4: Post-transfer validation
  # Download blob list with MD5 hashes
  az storage blob list --container-name <name> \
    --query "[].{name:name, md5:properties.contentMd5}" \
    --output json > target_checksums.json

Step 5: Compare source vs target checksums
  # Python comparison script or ADF Validation activity
  # Alert on any mismatch → re-transfer those files
```

**ADF approach (orchestrated, resumable):**
- Self-hosted IR installed on-prem to access NAS
- Copy Activity: source = file system, sink = ADLS Gen2
- Enable: `enableStaging = false`, `validateDataConsistency = true`
- ADF stores MD5 in blob metadata, compares after copy
- Use Foreach + retry on failed files

---

**Q7. What is Change Data Capture (CDC) and how is it used for zero-downtime database migration?**

**Answer:**

CDC captures row-level changes (INSERT, UPDATE, DELETE) from the source database's transaction log without modifying application code.

**Zero-downtime migration pattern:**

```
Phase 1: Initial bulk load
  Source DB → (full snapshot) → Target DB
  [Both DBs now identical at point T0]

Phase 2: CDC continuous sync
  Source DB transaction log
    → Debezium (open source CDC) OR Azure DMS CDC
    → Event Hubs (buffer + decouple)
    → Stream processor (Functions / Databricks Structured Streaming)
    → Target DB (apply changes in order)
  [Target lags source by seconds/minutes]

Phase 3: Monitor replication lag
  → Watch: lag < 5 seconds consistently

Phase 4: Cutover
  → Freeze application writes to source (maintenance page or read-only mode)
  → Wait for lag = 0
  → Switch connection strings to target
  → Remove maintenance page
  → Total downtime: ~30 seconds to 2 minutes
```

**Supported CDC sources:**
- SQL Server → Debezium SQL Server connector or Azure DMS
- Oracle → GoldenGate or Debezium Oracle connector
- PostgreSQL → Debezium PostgreSQL (logical replication slots)
- MySQL → Debezium MySQL connector

---

**Q8. How do you validate data integrity after a large-scale migration?**

**Answer:**

Three-layer validation approach:

**Layer 1 — Row count and structural checks**
```sql
-- Run on both source and target
SELECT
  TABLE_NAME,
  TABLE_ROWS
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'your_database'
ORDER BY TABLE_NAME;
```
Compare results between source and target. Any mismatch triggers investigation.

**Layer 2 — Checksum / hash validation**
```sql
-- SQL Server: hash-based row validation
SELECT CHECKSUM_AGG(BINARY_CHECKSUM(*)) AS table_checksum
FROM your_table
WHERE updated_at >= @migration_start;

-- PostgreSQL
SELECT md5(string_agg(t::text, ',' ORDER BY id)) AS checksum
FROM your_table t;
```

**Layer 3 — Business-level smoke tests**
- Key reports produce identical numbers (daily totals, account balances)
- Search returns expected results
- Authentication flows work
- Critical stored procedures execute successfully

**Tooling options:**
- **Azure Data Factory Validation Activity**: validates file existence + minimum size
- **Great Expectations** (Python): define data quality rules, run against source and target
- **dbt tests**: schema + referential integrity tests on data warehouse migrations

---

## ADVANCED

**Q9. Design a zero-downtime migration strategy for a 200TB Oracle database to Azure SQL Managed Instance for a Canadian bank. Downtime budget: 4 hours.**

**Answer:**

**Step 1 — Clarify constraints**
- OSFI B-10: model and data risk documentation required
- PIPEDA: data must stay in Canada (Canada Central + Canada East only)
- 4-hour window = maintenance window only; production must keep running until cutover

**Step 2 — Architecture**

```
On-premises                    Azure (Canada Central)
─────────────────              ─────────────────────────
Oracle 19c (source)            Azure SQL MI (Business Critical tier)
  │                              │
  │ [Phase 1] SSMA conversion    │
  │ → schema + stored proc       │
  │   migration to T-SQL         │
  │                              │
  │ [Phase 2] Initial bulk load  │
  │ → Oracle Data Pump export    │
  │ → Azure Database Migration   │
  │   Service (full load)        │
  │                              │
  │ [Phase 3] CDC delta sync     │
  │ → Oracle LogMiner / GoldenGate
  │ → Event Hubs (buffer)        │
  │ → Azure Functions (apply)    │
  ▼                              ▼
  [Phase 4] 4-hour window cutover
```

**Step 3 — SSMA (SQL Server Migration Assistant for Oracle)**
- Converts Oracle schema, procedures, sequences, triggers → T-SQL
- Reports conversion errors — fix manually before data migration
- Test all procedures in dev environment first

**Step 4 — Bandwidth**
- 200TB at ExpressRoute 10Gbps = ~44 hours for initial bulk load
- Options: Data Box Heavy (offline, faster) + then CDC for delta
- OR: Use Azure ExpressRoute + dedicated migration circuit

**Step 5 — Cutover (4-hour window)**
```
T-00:00  Stop all writes to Oracle (maintenance page)
T-00:05  Verify CDC lag = 0 in Azure DMS monitor
T-00:10  Final transaction apply on Azure SQL MI
T-00:20  Run validation: row counts, checksums, key reports
T-01:00  Update all application connection strings (Key Vault update → app restart)
T-01:30  Smoke test all critical application paths
T-02:00  Confirm with business owners
T-04:00  Oracle marked read-only (kept hot for 72hr rollback window)
```

**Step 6 — Rollback plan**
- Keep Oracle hot for 72 hours in read-only mode
- If critical issue found: flip connection strings back (< 5 min)
- CDC can be reversed to sync Azure SQL MI changes back to Oracle if needed (rare)

**Step 7 — OSFI documentation**
- Data flow diagram showing migration path
- Encryption in transit: TLS 1.2 on all migration channels
- Data at rest: SQL MI uses TDE (Transparent Data Encryption) by default
- Audit log of all migration activities stored in Log Analytics

---

**Q10. Your data migration pipeline is running but the target is 15% short on row counts after 3 days. How do you diagnose and fix this?**

**Answer:**

**Immediate triage (first 30 minutes):**

```
1. Identify which tables are short
   SELECT source_count, target_count, source_count - target_count AS delta
   FROM migration_audit_table
   WHERE delta > 0
   ORDER BY delta DESC;

2. Check ADF pipeline run logs
   → Which copy activities failed or partially completed?
   → Check 'rowsCopied' vs 'rowsRead' in activity output JSON

3. Check for soft deletes
   → Source may have logical deletes (is_deleted = 1) excluded from your query
   → Reconcile: count(*) vs count where active = true

4. Check partition handling
   → Large tables often use partition-by-column copy in ADF
   → Verify all partition ranges were covered (no range skipped)
```

**Root causes and fixes:**

| Symptom | Root cause | Fix |
|---------|-----------|-----|
| Specific tables missing rows | ADF partition query missed date range | Recalculate partition bounds, re-run for gap |
| Random ~5% missing | Network timeout, retry exhausted | Enable ADF fault tolerance mode, increase retry count |
| All tables ~15% short | CDC started late, missed inserts during bulk load | Re-run bulk load for affected tables with timestamp filter |
| Some rows duplicated | CDC applied insert + bulk load overlap | De-duplicate with ROW_NUMBER() OVER (PARTITION BY pk ORDER BY updated_at DESC) |

**Systematic fix — gap fill:**
```python
# ADF: use watermark-based re-sync for specific tables
SELECT * FROM source_table
WHERE updated_at BETWEEN @last_good_sync AND @now
  AND id NOT IN (SELECT id FROM target_table WHERE ...)
```

**Prevention for future migrations:**
- Run reconciliation query every 4 hours during migration
- Alert when row count delta > 0.1%
- Use ADF `validateDataConsistency` flag on all Copy activities

---

**Q11. How would you migrate a stateful microservices application running on-premises Kubernetes to AKS with zero downtime?**

**Answer:**

**Challenge:** Stateful services (databases, message queues, file stores) can't simply be re-deployed — state must migrate.

**Strategy: Traffic-based blue-green migration**

```
Phase 1: Parallel infrastructure
  → Provision AKS cluster in Azure
  → Set up GitOps (Flux/ArgoCD) pointing at same repo as on-prem K8s
  → Deploy all stateless services to AKS (shadow environment)
  → Migrate stateful backing stores first (databases → Phase 2)

Phase 2: Data layer migration
  → CosmosDB replaces MongoDB: use MongoDB compatibility API + mongomirror
  → Azure Cache for Redis replaces on-prem Redis: redis-cli MIGRATE or REPLICATION
  → Azure Service Bus replaces RabbitMQ: bridge consumers temporarily

Phase 3: Traffic shifting (Front Door / Traffic Manager)
  Week 1:  5% traffic → AKS  (canary — monitor error rates)
  Week 2:  25% traffic → AKS
  Week 3:  50% traffic → AKS
  Week 4:  100% traffic → AKS

Phase 4: Decommission on-prem
  → Keep on-prem hot for 2 weeks after 100% shift
  → Decommission after no rollback events

Monitoring at each phase:
  → Error rate delta between old and new
  → P99 latency comparison
  → Business-level KPIs (orders processed, transactions/sec)
```

**Rollback at each phase:** Traffic Manager/Front Door weight can be set back to 0% on AKS in seconds — instant rollback at any phase.

---

**Q12. What strategies exist for migrating from AWS to Azure? What are the key service mappings?**

**Answer:**

**Key service mappings:**

| AWS | Azure |
|-----|-------|
| EC2 | Azure Virtual Machines |
| S3 | Azure Blob Storage |
| RDS | Azure SQL / PostgreSQL Flexible Server |
| DynamoDB | CosmosDB (NoSQL API) |
| Kinesis | Event Hubs (Kafka surface) |
| SQS/SNS | Service Bus |
| Lambda | Azure Functions |
| EKS | AKS |
| Glue | Azure Data Factory |
| Redshift | Azure Synapse Analytics |
| CloudFront | Azure CDN / Azure Front Door |
| Route 53 | Azure DNS + Traffic Manager |
| IAM | Entra ID + RBAC |
| Secrets Manager | Azure Key Vault |
| CloudWatch | Azure Monitor + Log Analytics |

**Migration approach:**
1. **Assessment**: Azure Migrate can now discover AWS EC2 instances (agentless)
2. **Stateless compute**: Re-deploy containers to AKS (same images, new config)
3. **Databases**: Use DMS for MySQL/PostgreSQL → Azure equivalents
4. **Object storage**: AWS S3 → Azure Blob via AzCopy `--source-type=AmazonS3`
5. **IaC**: Convert Terraform (mostly provider changes, resource names differ)
6. **Networking**: VPC → VNet, Security Groups → NSGs, Transit Gateway → Azure vWAN

**Hardest parts:** IAM policy translation (AWS IAM → Azure RBAC is not 1:1), VPC endpoint patterns (AWS PrivateLink → Azure Private Endpoints similar but different), and Lambda trigger mappings to Azure Functions bindings.
