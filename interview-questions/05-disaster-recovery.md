# Azure Disaster Recovery — Interview Questions

> Covers: RTO/RPO, backup strategies, geo-redundancy, active-active vs active-passive, failover testing.
> Levels: Beginner → Intermediate → Advanced

---

## BEGINNER

**Q1. What is RTO and RPO and how do they relate to disaster recovery design?**

**Answer:**

| Term | Full name | Definition | Example |
|------|-----------|------------|---------|
| **RTO** | Recovery Time Objective | Max acceptable time to restore service after failure | 4 hours |
| **RPO** | Recovery Point Objective | Max acceptable data loss measured in time | 15 minutes |

**RTO** asks: "How long can we be down?"
**RPO** asks: "How much data can we afford to lose?"

**Cost-complexity relationship:**
```
Stricter RTO/RPO → More expensive and complex architecture

RPO = 0        → Synchronous replication, active-active ($$$$)
RPO = minutes  → Asynchronous geo-replication ($$)
RPO = hours    → Periodic backup restore ($)

RTO = seconds  → Active-active with auto-failover ($$$$)
RTO = minutes  → Warm standby with automated failover ($$)
RTO = hours    → Cold standby / backup restore ($)
```

**Real-world targets by industry:**
- Canadian banking (core systems): RTO <4hr, RPO <15min
- E-commerce checkout: RTO <30min, RPO <5min
- Internal tools: RTO <8hr, RPO <1hr
- Batch analytics: RTO <24hr, RPO <1hr (last batch)

---

**Q2. What is the difference between Azure Backup, Azure Site Recovery, and geo-redundant storage?**

**Answer:**

| Service | What it does | RTO | RPO | Use case |
|---------|-------------|-----|-----|----------|
| **Azure Backup** | Point-in-time backup of VMs, databases, files | Hours (restore from backup) | Hours (last backup) | Data loss protection, accidental deletion |
| **Azure Site Recovery (ASR)** | Continuous VM/workload replication to secondary region | 15–30 min (failover) | Minutes (RPO of seconds to minutes) | Business continuity for VM workloads |
| **Geo-redundant storage** | Automatic data replication across Azure paired regions | Storage level only | Seconds (async replication) | Storage resilience, not application-level DR |

**Key distinction:**
- **Backup** = protection against data loss (human error, ransomware, corruption)
- **Site Recovery** = protection against outage (region failure, datacenter down)
- **Geo-redundant** = storage-level redundancy, doesn't help if your AKS cluster goes down

---

**Q3. What are the Azure storage redundancy options?**

**Answer:**

| Option | Copies | Where | Survives |
|--------|--------|-------|---------|
| **LRS** (Locally Redundant) | 3 | Same datacenter | Hardware failure |
| **ZRS** (Zone Redundant) | 3 | 3 Availability Zones, same region | Datacenter failure within region |
| **GRS** (Geo-Redundant) | 6 (3+3) | Primary + secondary region (async) | Regional outage (read from secondary if enabled) |
| **GZRS** (Geo-Zone Redundant) | 6 | ZRS in primary + LRS in secondary | AZ failure + regional outage |

**Production recommendation:**
- Minimum: ZRS for production storage (protects against single datacenter failure)
- Critical data: GRS or GZRS
- Banking (Canada): GZRS with RA-GZRS (read-access from secondary) — data stays in Canada

**RA-GRS / RA-GZRS**: Adds read access from secondary region. Without RA, secondary is only accessible after Microsoft declares a regional failover. With RA, you can read from secondary anytime (useful for DR testing and read offloading).

---

## INTERMEDIATE

**Q4. Compare active-active vs active-passive DR architectures. When would you choose each?**

**Answer:**

**Active-Passive (Warm Standby):**
```
Canada Central (Primary)     Canada East (Standby)
─────────────────────        ─────────────────────
AKS cluster (running)   →    AKS cluster (minimal, scaled down)
CosmosDB (write)        →    CosmosDB (read replica)
Redis (primary)              Redis (not provisioned — cold start on failover)
Azure SQL (primary)     →    Azure SQL geo-replica

Traffic Manager: health probe on primary
  → If primary unhealthy: failover to secondary (5–10 min)
  → Manual or automated
```

- **RTO**: 5–30 minutes (warm) to hours (cold)
- **RPO**: Seconds to minutes (depends on replication lag)
- **Cost**: Primary runs full; secondary runs at ~30% capacity
- **Use when**: Budget-constrained, <4hr RTO acceptable, secondary region rarely used

**Active-Active:**
```
Canada Central (Active)      Canada East (Active)
─────────────────────        ─────────────────────
AKS cluster (full)      ←→   AKS cluster (full)
CosmosDB (multi-master) ←→   CosmosDB (multi-master)
Redis (independent)          Redis (independent)
Azure SQL (readable 2nd)  ←→ Azure SQL (geo-replica)

Azure Front Door: routes users to nearest healthy region
  → Automatic failover in seconds (TCP anycast)
  → Can also do weighted traffic split
```

- **RTO**: Seconds (Front Door health probe: 30s detection + re-route)
- **RPO**: Near-zero for CosmosDB (synchronous for Strong/Bounded Staleness)
- **Cost**: 2× infrastructure (both regions running full)
- **Use when**: 99.99% SLA required, banking/payments, zero downtime needed

---

**Q5. How do you implement automated failover for an AKS workload using Azure Traffic Manager and Site Recovery?**

**Answer:**

**Architecture components:**

```
DNS Layer:
  Traffic Manager profile (priority routing):
    Endpoint 1: myapp-canadacentral.azurefd.net (Priority 1)
    Endpoint 2: myapp-canadaeast.azurefd.net (Priority 2)
  
  Health probe: GET /health every 30s, timeout 10s
  Failover: if 3 consecutive failures → switch to priority 2

Application Layer (both regions):
  AKS cluster (identical GitOps config via Flux/ArgoCD)
  NGINX Ingress Controller
  /health endpoint: checks DB connectivity + Redis connectivity

Database Layer:
  CosmosDB: auto-failover enabled (set priority order: Canada Central > Canada East)
  Azure SQL: active geo-replication + auto-failover group

Failover sequence (automated):
  T+00:00  Primary region AKS unhealthy (pod crashes, node pool fails)
  T+00:30  Traffic Manager detects 3 failed health probes
  T+01:00  DNS TTL expires (set TTL=30s for fast failover)
  T+01:30  Traffic Manager updates DNS → secondary region
  T+02:00  CosmosDB auto-failover triggers (primary region write moves to Canada East)
  T+02:00  Azure SQL auto-failover group switches primary
  T+05:00  All traffic now on Canada East
  Total estimated RTO: 5–10 minutes
```

**Site Recovery for VM-based workloads:**
```bash
# ASR replication policy
az site-recovery policy create \
  --resource-group rg-asr \
  --vault-name recovery-vault \
  --name prod-replication-policy \
  --recovery-point-threshold-in-minutes 60 \
  --app-consistent-frequency-in-hours 4 \
  --crash-consistent-frequency-in-minutes 5

# Test failover (non-disruptive)
az site-recovery protected-item failover-commit \
  --resource-group rg-asr \
  --vault-name recovery-vault
```

**DR testing cadence:**
- Quarterly: test failover (non-disruptive) on ASR
- Semi-annually: actual failover with traffic cutover (in maintenance window)
- Annually: full DR exercise with business teams

---

**Q6. How do you design backup and recovery for a multi-tier application with Azure SQL, CosmosDB, and ADLS Gen2?**

**Answer:**

**Azure SQL backup strategy:**
```
Automatic backups (always on):
  Full: weekly
  Differential: every 12–24 hours
  Transaction log: every 5–12 minutes

Retention:
  Default: 7 days (Basic tier) to 35 days (Standard/Premium/Business Critical)
  Long-term retention (LTR): weekly/monthly/yearly backups → Azure Blob
  Banking requirement: 7-year retention → enable LTR policy

Point-in-time restore:
  Can restore to any point within the retention window
  Azure portal or:
    az sql db restore \
      --dest-name restored-db \
      --edition Standard \
      --restore-point-in-time "2024-01-15T10:30:00Z"

Geo-restore: restore from geo-redundant backup to any region
  Used when primary region is entirely unavailable
```

**CosmosDB backup strategy:**
```
Two backup modes:

1. Periodic backup (default):
   Full backup every 1–24 hours (configurable)
   Retention: 2–30 days
   Restore: contact Microsoft Support (not self-service)
   RPO: up to the backup interval

2. Continuous backup (recommended for production):
   Point-in-time restore within retention window (7 or 30 days)
   Self-service restore via portal/CLI
   RPO: minutes (not the full backup interval)
   
   az cosmosdb restore \
     --account-name restored-cosmos \
     --restore-timestamp "2024-01-15T10:30:00Z" \
     --source-database-account-name prod-cosmos
```

**ADLS Gen2 backup strategy:**
```
Option 1: Azure Blob soft delete
  → Enables recovery of deleted files within retention period
  → Enable on storage account: 7–90 day retention
  → No extra cost (storage cost only for retained data)

Option 2: ADF pipeline copy to separate "archive" account
  → Daily/weekly incremental copy of critical data folders
  → Different storage account + different region
  → Simple, auditable, meets FINTRAC 7-year requirement

Option 3: Azure Backup for blob storage
  → Operational backup for ADLS Gen2
  → Supports point-in-time restore for blob containers
  → Retention: 1–360 days
```

---

## ADVANCED

**Q7. Design an active-active, multi-region DR architecture for a Canadian banking payment system with RTO < 1 minute and RPO = 0.**

**Answer:**

**Constraints:**
- RTO < 1 minute: eliminates manual failover → fully automated
- RPO = 0: no data loss → synchronous replication required
- OSFI: Canada Central + Canada East only (data residency)
- 99.99% SLA target

**Architecture:**

```
GLOBAL ENTRY:
  Azure Front Door (Premium)
    WAF Policy: OWASP 3.2 + custom payment rules
    Health probes: /health/live every 10s, 2 failures = unhealthy
    Routing: Priority (Canada Central = primary) + automatic failover
    Session affinity: disabled (stateless backend)

CANADA CENTRAL (Active)           CANADA EAST (Active)
──────────────────────            ─────────────────────
AKS cluster                        AKS cluster
  payment-api (30 pods)              payment-api (15 pods — warm, scales up)
  /health returns DB status          same

APIM (Standard tier)               APIM (Standard tier)
  rate limiting                      rate limiting
  request logging                    request logging

CosmosDB Account (multi-master)
  Write region 1: Canada Central
  Write region 2: Canada East
  Consistency: Strong    ← RPO = 0 (synchronous cross-region ack before return)
  
  ⚠ Strong consistency cost: ~2× latency (must ack from both regions)
  Payment API p99: ~80ms (Strong) vs ~40ms (Session)
  Acceptable for payments.

Azure SQL Managed Instance
  Primary: Canada Central (writes)
  Secondary: Canada East (always-on secondary — readable)
  Auto-failover group: failover within 15 seconds on health failure
  
  ⚠ SQL MI does NOT support synchronous replication → RPO = seconds, not 0
  Design decision: use CosmosDB for transaction ledger (RPO=0), SQL for non-critical data

Redis Cache (Premium with geo-replication):
  Canada Central ↔ Canada East (async replication)
  Session data: TTL 30min, loss of active sessions on failover = acceptable
  (Force re-login, not data loss — RPO=0 applies only to transaction data)

Service Bus (Premium, geo-disaster recovery):
  Primary: Canada Central
  Secondary: Canada East
  Namespace pairing: manual failover in <30s
  In-flight messages: survive failover (geo-redundant Premium tier)

FAILOVER SEQUENCE (automated, < 60 seconds):
  T+00  Canada Central AKS health check fails (3 consecutive)
  T+10  Front Door marks Canada Central unhealthy
  T+10  DNS TTL=30s begins expiring
  T+30  New connections route to Canada East only
  T+40  CosmosDB: write traffic shifts to Canada East (already a write region)
  T+45  APIM: Canada East begins serving 100% traffic
  T+60  All new payment requests processing on Canada East
  
  Canada East scales up: Cluster Autoscaler + KEDA handle load increase
  No manual intervention required.

RPO validation:
  CosmosDB Strong: write acknowledged only after both regions confirm = RPO = 0 ✓
  SQL MI: RPO = seconds (acceptable for non-ledger data) 
  Audit log: both regions write to same immutable Log Analytics workspace
```

**Testing regimen:**
- Monthly: chaos test (kill Canada Central AKS namespace, verify failover < 60s)
- Quarterly: full failover with traffic (in off-hours maintenance window)
- All chaos tests documented for OSFI model risk evidence

---

**Q8. How do you recover from a ransomware attack on your Azure environment? Walk through the runbook.**

**Answer:**

**Immediate response (first 30 minutes):**

```
Step 1: Contain — isolate affected resources
  # Deny all outbound traffic from affected VMs immediately
  az network nsg rule create \
    --nsg-name affected-vm-nsg \
    --name BLOCK-ALL-OUTBOUND \
    --priority 100 \
    --direction Outbound \
    --access Deny \
    --protocol '*'

  # Revoke SAS tokens if storage is compromised
  az storage account revoke-delegation-keys \
    --account-name affected-storage

  # Disable compromised Managed Identity
  az ad sp update --id <compromised-mi-id> --set accountEnabled=false

Step 2: Assess blast radius
  # Check Azure Defender for Cloud alerts
  # Review Azure Activity Log: what was accessed, by whom, when
  # Check Key Vault access logs: any secrets accessed unexpectedly
  # Review storage access logs: any unusual bulk download/delete
```

**Recovery (hours 1–6):**

```
Step 3: Verify backup integrity
  → Confirm backups are from BEFORE the attack vector appeared
  → Check backup job logs: any backups deleted or modified?
  → If backups also encrypted: escalate to Microsoft (geo-redundant copies unaffected)

Step 4: Clean environment restore
  → Do NOT restore into the compromised environment
  → Provision new VNet/subnets (clean network)
  → Restore VMs from pre-attack snapshot via ASR or Azure Backup
  → Restore databases using point-in-time restore

Step 5: Infrastructure rebuild (IaC)
  → Preferred: re-run Terraform pipelines against clean subscription
  → Faster + guaranteed clean: Terraform = known good state
  → Rotate ALL secrets, keys, certificates after restore

Step 6: Validate and harden
  → Scan restored environment with Defender for Cloud
  → Apply Azure Policy: deny storage without private endpoint
  → Enable Immutable Blob Storage (WORM) for backup containers
  → Enable Azure Backup soft-delete with 14-day retention
```

**Prevention (post-incident hardening):**
- Enable Defender for Storage + Defender for SQL (detects anomalous access)
- Immutable backup storage: `az storage container immutability-policy create --period-since-creation-in-days 14`
- Privileged Identity Management (PIM): no standing admin access
- Regular backup restore testing — untested backups are not backups
- WORM compliance: Azure Blob immutability policy for audit/financial data
