# Azure SQL Database
### 🟢 Beginner → 🟡 Intermediate → 🔴 Advanced

> *"Azure SQL is not SQL Server in the cloud.  
>  It's a cloud-native service that happens to speak T-SQL."*

---

## Q1. How do you configure high availability for Azure SQL Database in production?

**Scenario:** You're deploying a payment database for a Canadian financial services app. It requires 99.99% uptime, failover in < 30 seconds, and must survive a full AZ failure in Canada Central.

```
Azure SQL HA tiers:

  General Purpose (default)              Business Critical
  ──────────────────────────             ──────────────────────────
  99.99% SLA                             99.99% SLA
  Remote storage (HADR via Azure)        Local SSD + Always On AG
  Failover: ~25 seconds                  Failover: < 5 seconds
  1 read replica (readable secondary)    3 read replicas (readable)
  $0.28/vCore/hr                         $0.85/vCore/hr

  Hyperscale (for very large DBs > 4TB)
  ──────────────────────────────────────
  Up to 100TB
  99.99% SLA
  Scale reads with up to 4 read replicas
  Snapshot backups (instant, no I/O impact)

  For banking payments:
  → Business Critical (fastest failover + 3 readable replicas for reporting)
```

```bash
# Create Business Critical SQL Database with zone redundancy
az sql server create \
  --name sql-prod-cc \
  --resource-group rg-data \
  --location canadacentral \
  --admin-user sqladmin \
  --admin-password "<STRONG_PASSWORD>"

az sql db create \
  --server sql-prod-cc \
  --resource-group rg-data \
  --name payments-db \
  --edition BusinessCritical \
  --service-objective BC_Gen5_4 \    # 4 vCores Business Critical
  --zone-redundant true \            # Spread across AZs in Canada Central
  --backup-storage-redundancy Geo    # Geo-redundant backups to Canada East

# Enable read scale-out (route read queries to secondary replicas)
az sql db update \
  --server sql-prod-cc \
  --resource-group rg-data \
  --name payments-db \
  --read-scale Enabled
# Connection string: ApplicationIntent=ReadOnly → routes to secondary
# ApplicationIntent=ReadWrite → always routes to primary

# Create failover group to Canada East (auto-failover)
az sql failover-group create \
  --name fg-payments \
  --server sql-prod-cc \
  --resource-group rg-data \
  --partner-server sql-dr-ce \       # Pre-created server in Canada East
  --add-db payments-db \
  --failover-policy Automatic \
  --grace-period 1                   # 1 hour before automatic failover
```

```sql
-- Check replica status and lag
SELECT
    ags.name AS group_name,
    ar.replica_server_name,
    rs.is_local,
    rs.synchronization_state_desc,
    rs.synchronization_health_desc,
    rs.last_redone_time
FROM sys.availability_groups ags
JOIN sys.availability_replicas ar ON ags.group_id = ar.group_id
JOIN sys.dm_hadr_database_replica_states rs ON ar.replica_id = rs.replica_id;
```

> ⚠️ **Gotcha:** `--zone-redundant true` is only available for Business Critical and Premium tiers. General Purpose databases do NOT support zone redundancy — failover targets a different node but within the same AZ. For banking SLAs, zone redundancy requires Business Critical.

---

## Q2. How do you migrate an on-premises SQL Server database to Azure SQL with minimal downtime?

**Scenario:** A 300GB SQL Server 2016 database needs to move to Azure SQL. The business allows a 2-hour maintenance window on Sunday 2am. A full backup/restore takes 4 hours. What's the approach?

```
Online migration with DMS + CDC:

  Phase 1: Full load (Friday 6pm — runs in background, ~4 hrs)
  Source: SQL Server 2016 (on-prem)
         → DMS reads full snapshot
         → Target Azure SQL: receives 300GB snapshot
  No impact on source — still receiving writes

  Phase 2: CDC sync (continuous until Sunday 2am)
  Source transaction log
         → DMS reads INSERT/UPDATE/DELETE events
         → Target Azure SQL: applies changes
  Lag: typically < 2 seconds

  Phase 3: Cutover (Sunday 2am — 2hr window)
  2:00am — maintenance mode ON (app returns 503)
  2:01am — DMS lag reaches 0 (tables in sync)
  2:10am — validate: row counts, spot check key records
  2:20am — redirect app connection strings to Azure SQL
  2:25am — maintenance mode OFF ✅
  4:00am — keep source online read-only for 4hr rollback window
```

```bash
# Pre-migration: assess compatibility
az datamigration assessment \
  --connection-string "Server=sql-prod.corp.internal;Database=payments;User ID=sa;Password=<pw>" \
  --target-platform AzureSqlDatabase \
  --output-folder C:\migration-assessment
# Review: SKU recommendations, compatibility issues, feature gaps

# Create DMS service
az dms create \
  --service-name dms-prod-cc \
  --resource-group rg-data \
  --location canadacentral \
  --sku-name Premium_4vCores \
  --virtual-subnet-id $SUBNET_ID   # Must be in same VNet as source (SHIR subnet)

# Create migration project
az dms project create \
  --service-name dms-prod-cc \
  --resource-group rg-data \
  --name proj-payments-migration \
  --source-platform SQL \
  --target-platform SQLDB \
  --location canadacentral

# Monitor migration (watch lag reach 0 before cutover)
az dms project task show \
  --service-name dms-prod-cc \
  --resource-group rg-data \
  --project-name proj-payments-migration \
  --task-name task-payments-cdc \
  --expand output \
  --query "properties.output[0].{state:migrationState, lag:latency}"
# Wait for: state=READY_TO_COMPLETE, lag=0
```

```sql
-- Post-migration validation queries (run on Azure SQL)
-- 1. Row count comparison
SELECT 'transactions' as table_name, COUNT(*) as row_count FROM transactions
UNION ALL
SELECT 'customers', COUNT(*) FROM customers;
-- Expected: match source row counts exactly

-- 2. Check for constraint violations
EXEC sp_MSforeachtable 'ALTER TABLE ? CHECK CONSTRAINT ALL'

-- 3. Verify indexes are present
SELECT name, type_desc, is_disabled
FROM sys.indexes
WHERE object_id = OBJECT_ID('transactions') AND is_disabled = 1;
-- Disabled indexes: rebuild them post-migration
```

> ⚠️ **Gotcha:** SQL Server features not supported in Azure SQL: `xp_cmdshell`, SQL Agent jobs, linked servers, CLR integration (partially), Service Broker cross-database. Run the assessment tool BEFORE migration — discovering unsupported features after the cutover window causes rollback.

---

## Q3. How do you use Elastic Pools for multi-tenant SaaS cost optimization?

**Scenario:** You have 200 customers, each with their own database (database-per-tenant model). Most databases are idle 80% of the time. Provisioning 200 × 4-vCore databases = $8,960/month. You need to cut costs.

```
Elastic Pool — shared compute, isolated storage:

  WITHOUT Pool:                WITH Pool (200 databases):
  200 DBs × 4 vCores each      Elastic Pool: 16 vCores shared
  = 800 vCores committed        = 200 DBs share 16 vCores
  = $8,960/month                = $1,792/month

  Works because:
  At peak, only 20 customers (10%) are active simultaneously
  16 vCores is more than enough for 20 concurrent workloads

  Per-database limits:
  Min DTUs/vCores = 0 (idle DB uses 0)
  Max DTUs/vCores = 4 (no single DB monopolises the pool)

  ⚠️ Risk: if too many customers suddenly active simultaneously
  → pool CPU contention → queries slow for all
  → Monitor: pool CPU > 80% → add pool vCores
```

```bash
# Create elastic pool
az sql elastic-pool create \
  --server sql-prod-cc \
  --resource-group rg-data \
  --name ep-tenants \
  --edition GeneralPurpose \
  --capacity 16 \           # 16 vCores shared across all DBs in pool
  --db-min-capacity 0 \     # Idle DBs use 0 vCores
  --db-max-capacity 4 \     # Single DB max = 4 vCores
  --zone-redundant false    # GP supports ZR in some regions — check availability

# Move existing tenant database into pool
az sql db update \
  --server sql-prod-cc \
  --resource-group rg-data \
  --name tenant-001-db \
  --elastic-pool ep-tenants

# Create new tenant database directly in pool
az sql db create \
  --server sql-prod-cc \
  --resource-group rg-data \
  --name tenant-201-db \
  --elastic-pool ep-tenants
```

```sql
-- Monitor pool resource usage (run on master database)
SELECT
    elastic_pool_name,
    avg_cpu_percent,
    avg_data_io_percent,
    avg_log_write_percent,
    max_worker_percent,
    avg_storage_percent
FROM sys.elastic_pool_resource_stats
WHERE start_time > DATEADD(HOUR, -1, GETUTCDATE())
ORDER BY start_time DESC;
-- If avg_cpu_percent consistently > 70%: time to add vCores to pool
```

> ⚠️ **Gotcha:** All databases in an elastic pool must be on the same logical server. You cannot span a pool across servers or regions. For multi-region SaaS, create separate pools per region — assign tenants to their nearest region's pool.

---

## Q4. How do you diagnose and tune SQL performance using Query Store?

**Scenario:** After a deployment, the payment dashboard takes 45 seconds to load instead of 2 seconds. The change was a schema migration. You need to find which query regressed and why.

```
Query Store — built-in query performance tracking:

  Azure SQL Database (Query Store enabled by default)
       │ records: query text, execution plan, runtime stats
       │ per interval (default: 60 min aggregation)
       ▼
  Views: sys.query_store_query, sys.query_store_plan, sys.query_store_runtime_stats

  Regression detection:
  Query X: avg duration BEFORE migration = 200ms
  Query X: avg duration AFTER  migration = 45,000ms  ← regression!
  
  Cause: schema migration dropped an index → plan changed from
  "Index Seek" (200ms) to "Table Scan" (45,000ms)

  Fix:
  1. Force old execution plan (instant rollback without code change)
  2. Or: recreate the index
```

```sql
-- Find top 10 queries by total duration (last 24 hours)
SELECT TOP 10
    q.query_id,
    qt.query_sql_text,
    SUM(rs.avg_duration) * SUM(rs.count_executions) / 1000000 AS total_duration_seconds,
    SUM(rs.count_executions) AS execution_count,
    AVG(rs.avg_duration) / 1000 AS avg_duration_ms
FROM sys.query_store_query q
JOIN sys.query_store_query_text qt ON q.query_text_id = qt.query_text_id
JOIN sys.query_store_plan p ON q.query_id = p.query_id
JOIN sys.query_store_runtime_stats rs ON p.plan_id = rs.plan_id
JOIN sys.query_store_runtime_stats_interval i ON rs.runtime_stats_interval_id = i.runtime_stats_interval_id
WHERE i.start_time > DATEADD(HOUR, -24, GETUTCDATE())
GROUP BY q.query_id, qt.query_sql_text
ORDER BY total_duration_seconds DESC;

-- Find queries with plan regressions (two plans, one much slower)
SELECT
    q.query_id,
    qt.query_sql_text,
    p.plan_id,
    AVG(rs.avg_duration) / 1000 AS avg_duration_ms,
    AVG(rs.avg_logical_io_reads) AS avg_logical_reads
FROM sys.query_store_query q
JOIN sys.query_store_query_text qt ON q.query_text_id = qt.query_text_id
JOIN sys.query_store_plan p ON q.query_id = p.query_id
JOIN sys.query_store_runtime_stats rs ON p.plan_id = rs.plan_id
WHERE q.query_id = <query_id_from_above>
GROUP BY q.query_id, qt.query_sql_text, p.plan_id
ORDER BY avg_duration_ms DESC;

-- Force the fast plan (instant fix — no code deploy needed)
EXEC sp_query_store_force_plan
    @query_id = <query_id>,
    @plan_id  = <fast_plan_id>;
-- Forces SQL Server to use this plan regardless of optimizer statistics
```

> ⚠️ **Gotcha:** Forcing a query plan via `sp_query_store_force_plan` is a temporary measure. The forced plan may become suboptimal as data grows. Set a reminder to review forced plans quarterly and recreate missing statistics or indexes to allow the optimizer to choose naturally.
