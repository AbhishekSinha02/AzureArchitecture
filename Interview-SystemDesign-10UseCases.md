# Azure System Design — Interview Prep: 10 Use Cases with Deep Dives

> **How to use this:** Read each use case as a conversation, not a script.
> The goal is to build a mental model deep enough that you can answer
> follow-up questions you haven't seen before — because you understand
> *why* each decision was made, not just *what* was decided.
>
> For every use case: start with clarifying questions, state your assumptions,
> draw the architecture, then justify each decision with tradeoffs.
> That's what separates a Principal Architect answer from a mid-level answer.

---

## Table of Contents

| # | Use Case | Core Probe Areas |
|---|----------|-----------------|
| 1 | [Real-Time Fraud Detection — Canadian Bank](#use-case-1-real-time-fraud-detection--canadian-bank) | Streaming, ML scoring, latency, OSFI |
| 2 | [Multi-Tenant SaaS Platform on AKS](#use-case-2-multi-tenant-saas-platform-on-aks) | AKS internals, isolation, Workload Identity, KEDA |
| 3 | [500TB Video Migration — Zero Downtime](#use-case-3-500tb-video-migration--zero-downtime) | Data Box, AzCopy, checksums, CDN warm-up |
| 4 | [RAG Compliance Chatbot for Financial Services](#use-case-4-rag-compliance-chatbot-for-financial-services) | AI Search, chunking, governance, hallucination prevention |
| 5 | [Payment Processing API — 99.99% Uptime](#use-case-5-payment-processing-api--9999-uptime) | Active-active, idempotency, circuit breaker, dead-letter |
| 6 | [IoT Factory Platform — 1 Million Sensors](#use-case-6-iot-factory-platform--1-million-sensors) | IoT Hub vs Event Hubs, partitioning, ADX, edge |
| 7 | [Enterprise Data Lakehouse Migration](#use-case-7-enterprise-data-lakehouse-migration) | CDC, ADF, Delta Lake, Purview, zero-downtime cutover |
| 8 | [CI/CD Platform for 50-Team Engineering Org](#use-case-8-cicd-platform-for-50-team-engineering-org) | GitHub Actions, GitOps, ArgoCD, secrets, environments |
| 9 | [Global E-Commerce — Black Friday 10x Spike](#use-case-9-global-e-commerce--black-friday-10x-spike) | Front Door, KEDA, CosmosDB autoscale, cache-aside |
| 10 | [Hub-Spoke Networking + Zero Trust for Enterprise](#use-case-10-hub-spoke-networking--zero-trust-for-enterprise) | ExpressRoute, Private Endpoints, NSG vs Firewall, DNS |

---

---

## Use Case 1: Real-Time Fraud Detection — Canadian Bank

### The Interview Question
> *"Design a real-time fraud detection system for a Canadian bank that processes 50,000 transactions per second, flags suspicious activity in under 200ms, and meets OSFI B-10 model risk requirements."*

---

### Step 1 — Clarifying Questions (always ask these first)

```
You should ask:
  1. Is this card-present (POS), card-not-present (online), or wire transfers?
     → Different latency budgets and feature sets
  2. Are we building the ML model or integrating an existing one?
     → Scopes the ML infra vs inference-only path
  3. What's the false positive tolerance? 1% vs 0.01% changes the threshold design
  4. Does a flagged transaction block or just alert?
     → Block = synchronous path (<200ms hard). Alert = async, more flexibility
  5. What's the audit retention requirement? (FINTRAC = 7 years)
  6. Active-active or active-passive DR?
```

**Assumptions to state aloud:**
- 50K TPS peak, 10K TPS average
- Card-not-present (online transactions) — async block acceptable
- ML model already trained; we own the inference infra
- Canada Central primary, Canada East DR (active-passive)

---

### Step 2 — Architecture

```
Transaction Event
    │
    ▼
Azure Event Hubs (64 partitions, 10 TUs)
    │
    ├──► Azure Stream Analytics (real-time rules: velocity, geo-anomaly)
    │    → flags rule-based hits in <50ms → CosmosDB (flag store)
    │
    └──► Azure Databricks Structured Streaming
         → feature engineering (rolling 24h spend, merchant category)
         → Azure ML real-time endpoint (XGBoost / LightGBM model)
         → fraud score + explanation (SHAP values)
         → CosmosDB (fraud_cases container)
         → Event Grid (trigger downstream: block card, alert, case mgmt)

Decision API (sync path for card-present):
  APIM → Azure Functions → Redis Cache (check flag) → CosmosDB read
  p99 target: <50ms (Redis hit), <150ms (CosmosDB miss)

Case Management:
  Cosmos → Logic Apps → ServiceNow / internal case tool

Audit Trail:
  All events → Log Analytics (WORM blob) — 7 years, FINTRAC compliant
```

---

### Step 3 — Key Design Decisions (justify each)

**Why Event Hubs, not Service Bus?**
```
Service Bus: guaranteed ordered delivery, competing consumers, max ~10K msg/s
Event Hubs:  partitioned log, 1M+ events/s, multiple consumer groups
             (Stream Analytics AND Databricks can read same stream independently)

At 50K TPS → Event Hubs. If this were <1K TPS with guaranteed order → Service Bus.
```

**Why two processing paths (Stream Analytics + Databricks)?**
```
Stream Analytics:
  - SQL-like rules: "3 transactions in 60s from different countries" → <50ms
  - Cheap, zero infrastructure, perfect for deterministic rules
  - Limitation: no ML model scoring, no complex feature stores

Databricks Structured Streaming:
  - Full Python, ML model scoring, complex feature engineering
  - Micro-batch latency: 100–500ms (acceptable for async flag)
  - Cost: cluster always-on (~$2K/month) — justified at 50K TPS

Rule-based catches 80% of obvious fraud fast + cheap.
ML catches the remaining 20% of subtle patterns.
```

**Why CosmosDB for the flag store?**
```
Requirements: sub-10ms reads, global distribution, schema-flexible (fraud patterns vary)
CosmosDB with session consistency:
  - <10ms p99 for point reads (partition key = card_id)
  - Autoscale RU: 400 baseline → 40,000 peak (Black Friday)
  - Change feed → triggers downstream workflows

Alternative rejected: Azure SQL — row locking under high TPS, schema rigid
```

---

### Step 4 — Deep Dive: Where Interviewers Probe

#### Probe 1: "How do you handle Event Hubs partitioning?"

```
Event Hubs partitions are the unit of parallelism AND ordering.
  - Ordering guaranteed WITHIN a partition, NOT across partitions
  - Partition key should be card_id or account_id
    → All events for same card land on same partition → ordered history
  - 64 partitions × 50K TPS = ~780 events/partition/second (manageable)

Pitfall: If you use random partition key (round-robin), you lose ordering
per card. Your Stream Analytics rule "3 txns in 60s from same card" breaks
because events are spread across partitions.

Stream Analytics reads partition-parallel (1 SU per 1MB/s input).
Size SUs: 50K TPS × avg 1KB payload = 50MB/s → need ~50 SUs minimum.
```

#### Probe 2: "Walk me through the ML model serving decision"

```
Options evaluated:
  a) Azure ML Real-time Endpoint (managed AKS behind the scenes)
     + SLA, auto-scaling, Model Registry integration
     - Cold start ~2s if scaled to 0 (not acceptable for fraud)
     Decision: keep minimum 2 instances always warm

  b) AKS direct (custom FastAPI serving)
     + Full control, no cold start with proper HPA config
     + Integrate SHAP explainability directly in inference code
     - You own the serving infrastructure

  c) Databricks model serving
     + Same cluster as feature engineering (latency win)
     + MLflow integration
     At 50K TPS: Databricks serving is expensive; AKS custom is cheaper at scale

Final choice: AKS custom serving with MLflow model loading.
  - 4 replicas, HPA on CPU > 70%
  - Model loaded into memory at startup (no cold start)
  - SHAP TreeExplainer pre-compiled (fast for tree models)
```

#### Probe 3: "How do you meet OSFI B-10 for this ML model?"

```
OSFI B-10 / E-23 model risk requirements:
  1. Model inventory: register model in Azure ML Model Registry
     → version, training data hash, validation metrics, champion/challenger

  2. Independent validation:
     → Hold-out test set from production distribution (not just dev split)
     → Confusion matrix, AUC, KS statistic, Gini coefficient
     → Stress test: does model degrade on COVID-era data? recession data?

  3. Explainability:
     → SHAP values for every decision stored in CosmosDB
     → Example: {"fraud_score": 0.92, "top_factors": [
         {"feature": "amount_vs_avg_30d", "shap": +0.41},
         {"feature": "new_merchant_category", "shap": +0.28}
       ]}
     → Customer/regulator can ask "why was this flagged?" → point to SHAP

  4. Human oversight:
     → Score > 0.95: auto-block (high confidence)
     → Score 0.70–0.95: human review queue (analyst reviews within 4h)
     → Score < 0.70: allow + monitor (soft flag in CosmosDB)

  5. Ongoing monitoring:
     → PSI (Population Stability Index) weekly: flag if feature distribution drifts
     → Champion/challenger: new model gets 5% traffic for 2 weeks before promotion
     → Rollback: Azure ML deployment swap (under 5 min)
```

#### Probe 4: "What happens if Databricks goes down?"

```
Fault tolerance design:
  Event Hubs retention: 7 days (default 1 day — increase this!)
  → If Databricks is down 4 hours, events are preserved in Event Hubs log
  → On recovery: Databricks resumes from last committed checkpoint
     (checkpointing to ADLS Gen2 every 30 seconds)

Stream Analytics (rule-based path) continues independently.
So rule-based fraud detection keeps working. Only ML scoring pauses.

Monitoring: Azure Monitor alert if Databricks job lag > 10,000 events
  → PagerDuty → on-call engineer within 5 min
```

---

### Numbers to Know

| Metric | Value | Why |
|--------|-------|-----|
| Event Hubs ingestion latency | <100ms end-to-end | From producer to consumer |
| Stream Analytics rule evaluation | <50ms | SQL window functions |
| ML model inference (AKS) | 10–30ms | XGBoost/LightGBM, tree-based fast |
| CosmosDB point read | <10ms p99 | On partition key, session consistency |
| Redis flag check | <2ms | In-memory, same region |
| Total fraud decision latency | <200ms | Rule: 50ms, ML: 150ms, write: 10ms |

---

### Follow-Up Questions an Interviewer Will Ask

```
Q: "Your model has 2% false positive rate. 50K TPS means 1,000 incorrect blocks per second.
    How do you handle customer impact?"
A: Tiered response: score 0.95+ = block, score 0.70-0.95 = step-up auth (SMS OTP),
   score < 0.70 = allow. Block only the highest-confidence cases.
   Dispute workflow: customer calls → analyst reviews SHAP → reverse within 15min SLA.

Q: "How do you retrain the model without taking the system down?"
A: Champion/challenger via Azure ML:
   1. Train new model on last 6 months data (offline, Databricks batch job)
   2. Deploy to 'challenger' endpoint (5% traffic via weighted routing in APIM)
   3. A/B compare: precision, recall, false positive rate for 2 weeks
   4. If challenger wins: promote to champion (traffic 100%)
   5. Keep old model deployed (weight 0%) for 30 days — instant rollback

Q: "How do you handle a sudden new fraud pattern that the model hasn't seen?"
A: Two-layer response:
   1. Immediate: Stream Analytics rule (human writes SQL rule in <30 min)
      e.g., "flag all transactions to this merchant_id in the next 2 hours"
   2. Short-term: re-label recent transactions, fine-tune model, hot-swap within 24h
   3. Long-term: add new feature to feature store, full retrain next sprint
```

---

---

## Use Case 2: Multi-Tenant SaaS Platform on AKS

### The Interview Question
> *"Design a multi-tenant SaaS platform on AKS for 500 enterprise customers. Each customer has different data isolation requirements — some want shared infrastructure, some want dedicated compute. How do you architect this?"*

---

### Step 1 — Clarifying Questions

```
1. What's the isolation requirement? Logical (namespace) vs physical (node pool)?
2. Are any customers regulated (HIPAA, OSFI, SOC2)? → may need dedicated
3. What's the pricing model? Shared = cheap. Dedicated = premium.
4. What's the scale per tenant? 100 users vs 10,000 users per tenant matters
5. What's the noisy-neighbor tolerance?
6. Do tenants need to bring their own Azure subscriptions? (rare, but some enterprise deals)
```

**Assumptions:**
- 3 tiers: Starter (shared), Business (isolated namespace), Enterprise (dedicated node pool)
- All tenants in same AKS cluster (single Azure subscription)
- Largest tenant: 50,000 users; smallest: 100 users

---

### Step 2 — Architecture

```
Tier 1 — Starter (shared):
  Namespace: shared-workloads
  All Starter pods share node pool
  ResourceQuota per namespace: 2 CPU, 4Gi RAM limit
  NetworkPolicy: namespace-level isolation (pods can't talk cross-namespace)

Tier 2 — Business (isolated namespace):
  Namespace: tenant-{id}
  Dedicated namespace, ResourceQuota: 8 CPU, 16Gi RAM
  PodDisruptionBudget: minAvailable=1 for critical deployments
  NetworkPolicy: deny-all ingress/egress by default, allow only APIM

Tier 3 — Enterprise (dedicated node pool):
  NodePool: nodepool-{tenant-id} with taint: dedicated=tenant-123:NoSchedule
  Pod toleration: matches taint → guaranteed no other pods land here
  ResourceQuota: no limit (they own the pool)
  Private ingress: dedicated LoadBalancer IP per enterprise tenant

Data isolation:
  CosmosDB: partition key = tenantId (logical isolation)
  PostgreSQL: separate schema per tenant (or separate DB for enterprise)
  Storage: separate containers in shared storage account (SAS per container)
           OR separate storage account for enterprise tier

Identity:
  Azure AD B2C: multi-tenant identity provider
  Each tenant has its own B2C directory (enterprise) or shared directory (starter)
  APIM: validate JWT, extract tenantId claim → route to correct namespace
```

---

### Step 3 — Key Design Decisions

**Why namespace isolation, not separate clusters?**
```
Separate clusters per tenant:
  + Perfect isolation
  - $3,000+/month per cluster control plane overhead
  - 500 clusters × $3K = $1.5M/month just in control plane costs
  - Unmanageable at scale (500 kubeconfig files, 500 upgrade cycles)

Namespace isolation in single cluster:
  + One control plane to manage
  + ResourceQuota + NetworkPolicy + RBAC gives strong logical isolation
  + KEDA scaling works across all namespaces
  - "Noisy neighbor" risk if ResourceQuota mis-configured
  - Single blast radius for cluster-level failures

Hybrid: 1 shared cluster for Starter/Business + dedicated clusters for
top 5 Enterprise customers who need guaranteed blast radius isolation.
```

**NetworkPolicy — this is where interviewers probe hard:**
```yaml
# Default deny-all for a tenant namespace — MUST have this
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: tenant-abc123
spec:
  podSelector: {}  # applies to ALL pods in namespace
  policyTypes:
    - Ingress
    - Egress

---
# Allow ingress only from APIM internal IP range
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-apim-ingress
  namespace: tenant-abc123
spec:
  podSelector:
    matchLabels:
      app: api-server
  policyTypes:
    - Ingress
  ingress:
    - from:
        - ipBlock:
            cidr: 10.0.2.0/24  # APIM subnet CIDR
      ports:
        - protocol: TCP
          port: 8080
```

**Interviewer gotcha:** "Does Kubernetes enforce NetworkPolicy by itself?"
```
NO. NetworkPolicy is a specification. Enforcement requires a CNI plugin that
supports it: Calico, Cilium, or Azure NPM (Azure's built-in for AKS).
If you use the default Azure CNI without Network Policy enabled → all
NetworkPolicy objects are silently ignored. You must enable it at cluster creation:

  az aks create --network-plugin azure --network-policy azure ...
                                        ^^^^^^^^^^^^^^^^^^^^^^^^^^
                                        This is the flag most people forget
```

---

### Step 4 — Deep Dives

#### Probe 1: "How does Workload Identity work in multi-tenant AKS?"

```
Problem: Tenant A's pod must read from Tenant A's CosmosDB.
         Tenant B's pod must NOT be able to read Tenant A's data.
         You can't store per-tenant secrets in pods (secret sprawl).

Solution: Workload Identity (OIDC token exchange)

How it works:
  1. AKS has an OIDC issuer URL (https://oidc.prod-aks.azure.com/{cluster-id}/)
  2. Each tenant namespace has a Kubernetes ServiceAccount
  3. ServiceAccount is annotated with the Azure Managed Identity client ID
  4. Pod uses that ServiceAccount → AKS injects OIDC token as mounted volume
  5. Pod exchanges OIDC token for Azure AD token → accesses only that tenant's resources

Setup per tenant:
  Azure side:
    → Create User-Assigned Managed Identity: mi-tenant-abc123
    → Federated credential on mi: 
       issuer = AKS OIDC URL
       subject = system:serviceaccount:tenant-abc123:workload-sa
       audience = api://AzureADTokenExchange
    → Assign role: mi-tenant-abc123 → Cosmos Reader on tenant-abc123 database

  Kubernetes side:
    ServiceAccount with annotation:
      azure.workload.identity/client-id: <mi-tenant-abc123-client-id>
    Pod with label:
      azure.workload.identity/use: "true"

Result: Pod gets a token scoped ONLY to tenant-abc123's resources.
        No secrets. Token rotates automatically. Works across pod restarts.

Common mistake: forgetting the pod label azure.workload.identity/use: "true"
→ Workload Identity webhook won't inject the token → 401 errors
```

#### Probe 2: "How does KEDA work for per-tenant autoscaling?"

```
KEDA (Kubernetes Event-Driven Autoscaling) scales pods based on external
metrics — Service Bus queue depth, Event Hubs lag, custom metrics.

Per-tenant scaling:
  Each tenant namespace has its own ScaledObject pointing to that tenant's queue.

apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: tenant-abc123-scaler
  namespace: tenant-abc123
spec:
  scaleTargetRef:
    name: api-server
  minReplicaCount: 1
  maxReplicaCount: 20
  triggers:
    - type: azure-servicebus
      metadata:
        queueName: tenant-abc123-jobs
        namespace: sb-saas-platform
        messageCount: "5"  # scale up when >5 messages per pod
      authenticationRef:
        name: keda-trigger-auth  # uses Workload Identity, no secrets

Result: Tenant with 100 users stays at 1 replica.
        Tenant with spike of 10K concurrent users scales to 20 replicas.
        Both in same cluster, zero interference.

Key interview point: KEDA ScaledObject triggers a Horizontal Pod Autoscaler
(HPA) internally. You should NOT also create an HPA on the same deployment
— they will conflict and fight each other.
```

#### Probe 3: "Tenant complains their pods are slow. How do you debug?"

```
Structured debug approach (show this order to demonstrate seniority):

Step 1: Is it the pod or the node?
  kubectl top pods -n tenant-abc123
  kubectl top nodes
  → If pod CPU/mem is at limit → ResourceQuota too tight
  → If node is at capacity → cluster autoscaler hasn't provisioned yet (check CA logs)

Step 2: Is it the network?
  kubectl exec -it <pod> -n tenant-abc123 -- curl -w "%{time_total}" <service>
  → High latency → check if NetworkPolicy is too restrictive
  → Check if DNS resolution is slow: time nslookup <service>.tenant-abc123.svc.cluster.local
  
Step 3: Is it throttled compute?
  kubectl describe pod <name> -n tenant-abc123 | grep -i throttl
  → If CPU throttling > 20% → increase CPU limit or request
  
Step 4: Is it the external dependency (CosmosDB, Redis)?
  Check App Insights / Azure Monitor for CosmosDB RU consumption
  → If RU exhausted → autoscale threshold too conservative
  
Step 5: Is it a noisy neighbor?
  kubectl describe node <node> | grep -A 20 "Allocated resources"
  → If a Starter-tier tenant is using 90% of shared node CPU
  → Tighten ResourceQuota, or move noisy tenant to Business tier
```

---

### Numbers to Know

| Metric | Value |
|--------|-------|
| AKS control plane cost (Standard) | ~$73/month (with uptime SLA) |
| Node pool (Standard_D4s_v5, 4 CPU, 16GB) | ~$140/month |
| Max pods per node (Azure CNI) | 30 per node (CIDR dependent) |
| Max nodes per cluster | 1,000 |
| KEDA scale-up time | 15–30 seconds (from trigger to pod running) |
| Workload Identity token TTL | 24 hours (auto-rotated by Azure AD) |

---

---

## Use Case 3: 500TB Video Migration — Zero Downtime

### The Interview Question
> *"You need to migrate 500TB of video and audio files from an on-premises NAS to Azure. The content is actively used — users are streaming 24/7. Design the migration with zero downtime and no data loss."*

---

### Step 1 — Clarifying Questions

```
1. What's the current streaming infrastructure? (HTTP server? CDN?)
2. How many new files are written per day? (active vs mostly-read)
3. What's the network capacity? (1Gbps ExpressRoute vs public internet)
4. Is there a maintenance window, even a short one?
5. What's the format? (MP4, MKV, raw camera) — affects transcoding needs
6. What's the SLA during migration? Can we tolerate 1% of requests hitting origin?
```

**Assumptions:**
- 500TB, ~2M files, 200GB added per day, 50K concurrent streams at peak
- 1Gbps ExpressRoute available
- No maintenance window (truly zero downtime)
- Content is served via HTTP; no DRM yet (migrate first, add DRM later)

---

### Step 2 — Architecture

```
PHASE 1: Bulk copy (weeks 1–4)
  On-prem NAS → Azure Data Box ×7 (80TB each = 560TB capacity)
  Parallel: ExpressRoute for new files written during Data Box transit

PHASE 2: Delta sync (weeks 4–6)
  ADF Self-Hosted IR → monitors NAS for new/modified files
  AzCopy sync every 15 min (watermark = last_modified timestamp)
  Goal: keep Azure copy within 15 minutes of source

PHASE 3: Dark launch (week 6)
  Configure Azure CDN pointing to Azure Blob (no users sent here yet)
  Warm CDN cache: pre-fetch top 1,000 most-viewed assets
  Test: internal teams use Azure URL — validate quality, latency

PHASE 4: Traffic shift (week 7)
  DNS-based gradual cutover:
    Day 1: 5% traffic → Azure CDN (via Azure Traffic Manager weight)
    Day 3: 25% traffic → Azure CDN
    Day 5: 50% traffic → Azure CDN
    Day 7: 100% traffic → Azure CDN
  Keep NAS HTTP server live (read-only) until Day 14 → rollback possible

PHASE 5: Final sync + decommission (week 8)
  Set NAS to read-only (prevent new writes)
  Final AzCopy sync — wait for delta = 0
  Switch DNS fully to Azure CDN
  Decommission NAS after 30 days (observe for stragglers)
```

---

### Step 3 — Deep Dives

#### Probe 1: "How do you guarantee no data loss? Walk me through checksum validation."

```
This is a common probe. Interviewers want to see you know the actual mechanics.

Step 1: Generate manifest on source BEFORE transfer
  find /nas/media -type f -exec md5sum {} \; > source_manifest.txt
  # Format: <md5hash>  /nas/media/path/to/file.mp4

  Better: use SHA-256 (MD5 has collision risk, unacceptable for audit)
  find /nas/media -type f -exec sha256sum {} \; > source_manifest.txt

Step 2: AzCopy with built-in hash verification
  azcopy copy "/nas/media/*" "https://stg.blob.core.windows.net/videos?<SAS>" \
    --recursive \
    --check-md5 FailIfDifferent \  # AzCopy verifies MD5 on every file
    --block-size-mb 256 \
    --parallel-level 64

  "FailIfDifferent" = if Azure-computed MD5 ≠ local MD5 → fail the file, log it
  AzCopy stores MD5 in blob Content-MD5 header automatically

Step 3: Post-transfer validation (independent of AzCopy)
  # Python script: compare manifest vs Azure Blob properties
  for blob in container.list_blobs():
      props = blob.get_blob_properties()
      azure_md5 = base64.b64decode(props.content_settings.content_md5).hex()
      expected_md5 = manifest[blob.name]
      if azure_md5 != expected_md5:
          failed_files.append(blob.name)

  Log ALL mismatches. Re-transfer failed files.
  Don't proceed to cutover until: failed_files == []

Step 4: Spot-check sampling (for very large migrations)
  Validate 100% of files for first 10TB batch
  Validate 10% random sample for remaining batches (statistically sufficient)
  Full validation of last delta sync before cutover

Step 5: File count validation
  Source: find /nas/media -type f | wc -l → 2,000,000 files
  Azure: az storage blob list --container-name videos --num-results 999999 | 
         jq length → 2,000,000 files
  Counts must match exactly before cutover.
```

#### Probe 2: "How do you handle files that are being written while you're copying?"

```
This is the zero-downtime challenge. New 200GB/day = ~2,300 files/day being written.

Problem: AzCopy copies a file while it's being written to → partial copy → corrupt

Solutions:

Option A: Write-completion signal (preferred)
  Source system emits event when file finalize/close
  Custom watcher: inotifywait (Linux) or FileSystemWatcher (.NET)
    → on IN_CLOSE_WRITE event → add file to Service Bus queue
    → Azure Function pulls queue → triggers AzCopy for that specific file
  Result: only fully-written files are ever transferred

Option B: Modification time guard
  AzCopy with --include-after flag:
    → only copy files where last_modified > (now - 30min)
    → run every 15 min → 15-min overlap ensures no file missed
    → skip files modified in last 5 min (likely still being written)
  Simpler, but ~5min window of potential partial copies

Option C: Rename atomicity (video workflow pattern)
  Video encoders write to temp file: video_abc.mp4.tmp
  On completion: rename to video_abc.mp4
  AzCopy ignore *.tmp files
  Only fully-written, renamed files are copied

Production recommendation: Option A for accuracy, Option C if encoder supports it.
```

#### Probe 3: "How do you warm the CDN before cutover?"

```
Cold CDN = first user after DNS cutover hits origin → high latency, bad experience.
CDN warm-up is a real technique interviewers expect at senior level.

Strategy:
  1. Identify top-1000 assets by view count (query video analytics DB)
  2. Pre-fetch script: curl each asset through the CDN URL
     → CDN fetches from origin, caches at edge PoPs
  3. Run from VMs in multiple Azure regions to hit multiple CDN PoPs

  # Warm-up script
  while IFS= read -r url; do
    curl -s -o /dev/null -w "%{http_code} %{time_total}s $url\n" \
         "https://cdn.mycompany.com/videos/$url"
  done < top_1000_assets.txt

  4. Verify cache HIT rate: check CDN diagnostic logs
     X-Cache: HIT = cached. MISS = still hitting origin.
     Target: >90% HIT rate on top-1000 before cutover

  5. Long-tail assets: accept cold cache. CDN will cache on first real request.
     For 2M assets, you can't pre-warm everything. Top 1K covers ~80% of traffic (Pareto).

CDN TTL strategy:
  Videos (immutable content): TTL = 7 days (never changes once uploaded)
  Manifests/playlists (HLS .m3u8): TTL = 30 seconds (updated frequently)
  Thumbnails: TTL = 1 hour
```

---

---

## Use Case 4: RAG Compliance Chatbot for Financial Services

### The Interview Question
> *"Build an AI chatbot for compliance analysts at a bank. It should answer questions about internal policies, regulatory documents, and past audit findings. It must not hallucinate — wrong answers in compliance are legally dangerous."*

---

### Step 1 — Clarifying Questions

```
1. What's the document corpus? How many docs, what formats?
2. Is PII present in the documents?
3. Does the answer need citations? (yes, always for compliance)
4. Who uses this — analysts only, or external-facing?
5. What's the tolerance for "I don't know"? (should it always answer or refuse if unsure?)
6. Regulatory requirements for AI in this org? (OSFI B-10 model risk register?)
```

---

### Step 2 — Architecture (Indexing + Query)

```
INDEXING PIPELINE (offline):
  SharePoint / Blob / Email → Document Intelligence (extract text, tables, layout)
      → PII Detection (Azure AI Language) → redact before indexing
      → Chunking (512 tokens, 50-token overlap, sentence-boundary aware)
      → Embedding (text-embedding-3-large) → AI Search index
      → Metadata → CosmosDB (doc title, date, classification, source)

QUERY PIPELINE (real-time):
  User question
      → Azure Content Safety (input screening)
      → Query embedding (same model as indexing)
      → AI Search: hybrid BM25 + vector + semantic reranker
      → Top-5 chunks retrieved
      → Prompt construction with strict grounding instruction
      → GPT-4o (low temperature: 0.1)
      → Confidence assessment
      → Azure Content Safety (output screening)
      → Response + citations (doc title, page, section)
      → Audit log → Log Analytics (immutable)
```

---

### Step 3 — Deep Dives

#### Probe 1: "How do you prevent hallucinations? Give me the technical implementation."

```
This is the most common follow-up. Give all four layers:

LAYER 1 — Retrieval grounding (most important)
  System prompt: "Answer ONLY using the provided context below.
                  If the answer is not explicitly stated in the context,
                  respond with: 'I cannot find this in the available documents.'
                  Do not use prior knowledge. Do not speculate."

  This alone eliminates ~70% of hallucinations. The model is instructed to stay
  in-context. GPT-4o is very good at following this instruction at temperature 0.1.

LAYER 2 — Citation enforcement (structural)
  Use structured output (JSON mode in Azure OpenAI):
  {
    "answer": "string",
    "citations": [
      {"doc_title": "string", "chunk_id": "string", "relevant_quote": "string"}
    ],
    "confidence": "HIGH | MEDIUM | LOW",
    "answer_found_in_context": true | false
  }

  If answer_found_in_context = false → return "not found" regardless of answer field.
  This prevents the model from giving an answer when it should say "I don't know."

LAYER 3 — Confidence gate
  If retrieval score (semantic similarity) < 0.72 → the retrieved docs
  are probably not relevant → add disclaimer: "Low confidence — please verify."
  If retrieval score < 0.60 → refuse to answer entirely.

LAYER 4 — Automated faithfulness evaluation (offline)
  RAGAS faithfulness metric: measures if every sentence in the answer
  is supported by at least one retrieved chunk.
  Run weekly on a golden test set of 200 question-answer pairs.
  Alert if faithfulness drops below 0.85.
```

#### Probe 2: "How do you chunk documents, and why does it matter?"

```
Chunking is one of the biggest levers on RAG quality. Most junior answers
say "split by 512 tokens." Senior answers explain the strategy.

Bad chunking:
  - Split mid-sentence: "The regulation requires all firms to maintain..."
    "...records for a minimum of 7 years."
    These two chunks have no meaning independently.
  - Too small (128 tokens): loses context, retrieval returns fragments
  - Too large (2048 tokens): exceeds context window if you retrieve 5 chunks,
    and dilutes the relevance signal for the embedding

Good chunking strategy:

  1. Sentence-boundary aware:
     Use spaCy or NLTK sentence tokenizer. Never split mid-sentence.
     
  2. Semantic coherence:
     Keep paragraphs together when possible.
     Split at heading boundaries (h2, h3 in PDF/HTML).
     
  3. Token budget: 512 tokens per chunk, 50-token overlap
     Overlap preserves context continuity at chunk boundaries.
     
  4. Metadata preservation:
     Each chunk knows its parent document, section heading, page number.
     This enables citation: "Section 4.2, Page 12 of OSFI B-10 Guideline"
     
  5. Special handling for tables:
     Tables extracted by Document Intelligence as HTML.
     Chunk each row as a separate document with table headers prepended.
     "Table: Capital Requirements | Row: Tier 1 Capital | Minimum: 8%"
     
  6. Hierarchical chunking (advanced):
     Parent chunk: full section (2000 tokens) for context
     Child chunk: single paragraph (512 tokens) for precision retrieval
     Retrieve children, but pass parent to LLM for fuller context.
     → LlamaIndex calls this "Small-to-Big" retrieval
```

#### Probe 3: "How do you handle a question that spans multiple documents?"

```
Example: "Compare the 2022 and 2024 OSFI audit findings on credit risk."
This requires retrieving from TWO separate documents and synthesizing.

Standard RAG retrieves top-5 chunks from one semantic query.
This fails if the answer requires combining information from multiple sources.

Solutions:

Option A: Multi-query retrieval (simple, works 80% of the time)
  Decompose the question into sub-queries:
    Sub-query 1: "OSFI audit findings on credit risk 2022"
    Sub-query 2: "OSFI audit findings on credit risk 2024"
  Run both queries → merge top-3 results from each → pass all 6 chunks to LLM
  LLM prompt: "Synthesize these results into a comparison."

Option B: Agentic multi-step (better for complex questions)
  Agent step 1: Search for 2022 audit document → retrieve findings
  Agent step 2: Search for 2024 audit document → retrieve findings
  Agent step 3: Compare the two sets of findings
  Agent step 4: Return comparison with citations from both

Option C: Pre-built comparison index
  At index time: for known document pairs (annual reports), create
  comparison chunks: "2022 vs 2024 credit risk findings: [diff]"
  Stored as a derived document in the same index.
  Works well for predictable comparison patterns.
```

---

---

## Use Case 5: Payment Processing API — 99.99% Uptime

### The Interview Question
> *"Design a payment processing API for a Canadian bank. SLA is 99.99% uptime. Payments must never be lost or double-processed. Peak load is 10,000 TPS."*

---

### Step 1 — Clarifying Questions

```
1. Is this card payments (Visa/MC rails) or bank transfers (Interac, wire)?
2. Synchronous response to user or async with notification?
3. What's the p99 latency budget? (<500ms? <2s?)
4. Is there an existing payment processor (Stripe, FIS) or is this homegrown?
5. Multi-region active-active or active-passive?
6. What's the idempotency requirement? (Can a payment be retried safely?)
```

**Assumptions:**
- Bank transfers (internal + Interac)
- Async: return payment_id immediately, notify on completion
- Active-active: Canada Central + Canada East
- Idempotency keys required (client sends unique key per payment attempt)

---

### Step 2 — Architecture

```
Client → Azure Front Door (global LB, WAF)
      → APIM (auth, rate limit, idempotency check, audit)
      → Payment API (AKS — stateless microservice)
      → Service Bus Premium (geo-redundant, sessions for ordering)
      → Payment Processor (AKS — idempotent handler)
      → CosmosDB (ACID, multi-master active-active)
      → Event Grid (notification fanout: webhook, email, SMS)
      → Audit Log (Log Analytics WORM — FINTRAC 7 years)

Idempotency layer (APIM policy):
  Client sends header: Idempotency-Key: <uuid>
  APIM checks Redis: key exists? → return cached response (skip processing)
  APIM checks Redis: key absent? → proceed, store response in Redis (TTL 24h)
```

---

### Step 3 — Deep Dives

#### Probe 1: "How do you prevent double payments? Walk me through idempotency."

```
Idempotency means: calling the same operation multiple times has the same effect
as calling it once. This is mandatory for payment systems.

Scenario without idempotency:
  Client sends POST /payments → network timeout after 2s → client retries
  → Two payments processed → customer charged twice

Implementation at each layer:

LAYER 1: APIM (edge idempotency check — fastest path)
  <set-variable name="idempotencyKey" value="@(context.Request.Headers['Idempotency-Key'])" />
  <cache-lookup-value key="@((string)context.Variables['idempotencyKey'])" 
                      variable-name="cachedResponse" />
  <choose>
    <when condition="@(context.Variables.ContainsKey('cachedResponse'))">
      <return-response>  <!-- Return cached response without hitting backend -->
        <set-body>@((string)context.Variables["cachedResponse"])</set-body>
      </return-response>
    </when>
  </choose>

LAYER 2: Payment Service (database idempotency)
  CosmosDB upsert with condition:
    IF NOT EXISTS (WHERE idempotency_key = @key)
    INSERT payment record
    RETURN payment_id
  
  If record already exists → return existing payment_id → no duplicate processing
  This handles the race condition: two requests with same key arrive simultaneously
  → CosmosDB's optimistic concurrency (etag) ensures exactly one insert

LAYER 3: Payment Processor (Service Bus deduplication)
  Service Bus Premium: built-in duplicate detection
    MessageId = idempotency_key
    DuplicateDetectionHistoryTimeWindow = 10 minutes
    → If same MessageId arrives twice → second message silently discarded
  
  This means even if two messages somehow enter the queue → only one processed.

LAYER 4: Downstream system (compensating transaction)
  Even with all above: design downstream handlers to be idempotent
  Use "check-then-act" pattern:
    Check: has payment {id} already been applied to account balance?
    If yes: skip (idempotent)
    If no: apply, mark as applied

Result: A payment can be retried 100 times safely. It will process exactly once.
```

#### Probe 2: "Service Bus goes down. What happens to payments in flight?"

```
Service Bus Premium (required for this use case, not Standard):
  - 99.9% SLA (Standard) vs 99.9% SLA with Geo-Disaster Recovery (Premium)
  - Premium: Geo-DR pairing (Canada Central ↔ Canada East)

Failure scenario: Canada Central Service Bus zone failure

Service Bus Geo-DR behavior:
  1. Primary namespace becomes unavailable
  2. You trigger manual failover (or auto-failover if configured):
     az servicebus georecovery-alias fail-over --alias <alias> --namespace-name <secondary>
  3. Alias (logical endpoint) now points to secondary namespace
  4. Applications reconnect to alias → they don't need to change endpoints
  5. Messages IN the dead primary: those are at risk if not yet committed
  
Message durability guarantee:
  Service Bus Premium uses persistent storage. Messages survive zone failures.
  Region-level failure: messages in transit at moment of failure may be lost.
  Mitigation: client-side retry with idempotency key (retried message = safe)

Dead Letter Queue:
  Messages that fail processing after MaxDeliveryCount (e.g., 5 retries)
  → moved to DLQ automatically
  → Alert: Azure Monitor alert when DLQ count > 0
  → On-call engineer manually reviews and reprocesses or escalates
  
  Critical: DLQ messages represent FAILED payments — SLA for human review < 15 min
```

#### Probe 3: "How do you achieve 99.99% uptime? What's the math?"

```
99.99% = 52 minutes downtime per year = 4.4 minutes per month

Single region 99.9% SLA → active-active multi-region is required.

Availability math (independent failures):
  Single AKS in Canada Central: 99.9%
  Active-active (CC + CE): 1 - (0.001 × 0.001) = 99.9999% theoretical

But theory ≠ practice. Real sources of downtime:
  - Deployment errors (you pushed bad code)
  - Dependency failures (downstream bank API down)
  - Config changes (wrong APIM policy)
  - Certificate expiry

Design for 99.99% in practice:
  ✓ Blue-green deployments: never deploy directly to production
    → Deploy to green, test, shift 5% traffic, monitor, then 100%
    → Rollback = shift traffic back to blue (30 seconds)
  ✓ Circuit breaker (Polly in .NET):
    → If downstream bank API fails 5 times in 30s → open circuit
    → Return synthetic "payment queued" response → process when circuit closes
    → User isn't left waiting; payment isn't lost
  ✓ Retry with exponential backoff (not tight retry loops):
    → Retry 1: 100ms, Retry 2: 200ms, Retry 3: 400ms, Retry 4: 800ms, Retry 5: give up
    → Jitter: add random 0–50ms to avoid thundering herd
  ✓ Health probes:
    → Liveness: is the pod alive? (crashes → restart)
    → Readiness: is the pod ready to serve? (starting up / draining → remove from LB)
    → Startup probe: allow 60s for startup before liveness kicks in
  ✓ Chaos engineering (quarterly):
    → Azure Chaos Studio: inject pod failures, zone outages, latency
    → Verify system recovers within SLA; find weaknesses before customers do
```

---

---

## Use Case 6: IoT Factory Platform — 1 Million Sensors

### The Interview Question
> *"Design an IoT platform for a smart manufacturing company. They have 100 factories, each with 10,000 sensors sending telemetry every second. That's 1 million events per second. Design ingestion, processing, storage, and alerting."*

---

### Step 1 — Clarifying Questions

```
1. What kind of sensors? (temperature, vibration, pressure) — affects payload size
2. What's the alerting latency requirement? "Detect overheating in <1s" vs "<30s"
3. Do we need device management? (firmware updates, command-and-control)
4. Is there edge processing needed? (factory may have poor connectivity)
5. What's the retention period? Compliance? (some factories: 10 year equipment history)
6. What's the analytics use case? Real-time dashboard? Batch ML? Both?
```

**Assumptions:**
- Mix of sensors: temperature, vibration, energy consumption
- Average payload: 200 bytes per event
- Alerting: <5 seconds for critical alerts (machine overheating)
- Edge connectivity: factories have intermittent 4G/LTE
- Retention: 2 years hot, 10 years cold

---

### Step 2 — Architecture

```
EDGE (Factory):
  IoT Edge devices (local compute, 1 per factory floor)
    → Local processing: filter noise, aggregate 10s windows
    → Buffer during connectivity loss (queue locally, flush on reconnect)
    → Azure IoT Edge runtime: containerized modules on factory PC

INGESTION:
  Azure IoT Hub (device management + D2C messaging)
    → 100 factories × 10K sensors = 1M devices
    → IoT Hub: S3 tier (10M messages/day per unit × N units)
    → Partition: 32 partitions, routed by factory_id

  Why IoT Hub (not Event Hubs directly)?
    → IoT Hub = Event Hubs + device management + C2D + device twin
    → Device registry: per-device auth (SAS token or X.509 certificate)
    → Device twin: desired/reported state (firmware version, thresholds)
    → Without IoT Hub: no device identity, no C2D, no twin — just a stream

STREAM PROCESSING:
  Azure Stream Analytics (real-time alerts, <5s latency):
    → Rule: "temperature > 85°C for 10 consecutive seconds"
    → Output: Event Grid → Logic Apps → PagerDuty alert + machine shutdown command

  Azure Databricks Structured Streaming (complex analytics):
    → Anomaly detection ML model scoring
    → Feature computation: rolling 1h mean, standard deviation per sensor
    → Output: ADX (hot analytics), ADLS (cold storage)

STORAGE:
  Azure Data Explorer (ADX):
    → Time-series optimized: columnar storage, fast time-range queries
    → Hot: 90 days online, SSD-backed
    → Retention policy: auto-move to ADLS after 90 days
    → Query: "Show me vibration anomalies for machine M-472 in the last 7 days"
    → p99 latency for this query: <500ms (ADX is purpose-built for this)

  ADLS Gen2 (cold storage):
    → Parquet format, partitioned by factory/date/sensor_type
    → 2-year retention at standard tier, 10-year at archive
    → Batch analytics: Databricks reads for ML model training

DEVICE MANAGEMENT:
  Device Twin (IoT Hub):
    → desired.alert_threshold.temperature: 85
    → desired.firmware_version: 2.4.1
    → reported.firmware_version: 2.3.0  ← device reports actual state
    → Delta = firmware update needed → trigger OTA update job

  Azure Digital Twins (ADT):
    → Digital model of each factory: machines, sensors, relationships
    → "Which sensors are attached to conveyor belt CB-7 in Factory 42?"
    → Enables graph queries: downstream impact analysis of a machine failure
```

---

### Step 3 — Deep Dives

#### Probe 1: "IoT Hub vs Event Hubs — when do you choose each?"

```
This is a common direct question. Know the clear answer.

IoT Hub:
  Built on top of Event Hubs internally, but adds:
  ✓ Device identity & registry (per-device auth with SAS or X.509)
  ✓ Cloud-to-Device (C2D) messaging: send commands to specific sensors
  ✓ Device twin: persistent JSON state per device (desired/reported properties)
  ✓ Direct methods: invoke remote procedure on device (<30s)
  ✓ File upload: devices can upload large files (logs, images) to Blob
  ✓ Device provisioning (DPS): zero-touch enrollment at scale
  ✓ Edge integration: Azure IoT Edge orchestration

Event Hubs:
  Pure event streaming, no device identity layer
  ✓ Kafka-compatible surface (drop-in for Kafka producers)
  ✓ Higher throughput ceiling (Event Hubs Dedicated: 2 GB/s)
  ✓ Cheaper for pure streaming scenarios with no device management

Decision rule:
  "Do you need to know which specific device sent the event, 
   and do you need to send commands back to it?" → IoT Hub
  "Is this just streaming data with a known client SDK?" → Event Hubs

  For factory IoT with firmware updates, threshold configuration,
  machine shutdown commands → IoT Hub. Always.
```

#### Probe 2: "How do you handle 1M devices sending data simultaneously? Scaling IoT Hub."

```
IoT Hub scaling units:
  S1: 400K messages/day, $25/month per unit
  S2: 6M messages/day, $250/month per unit
  S3: 300M messages/day, $2,500/month per unit

1M sensors × 1 msg/second = 86.4B messages/day → need multiple S3 units
  86.4B / 300M per S3 unit = 288 S3 units → ~$720K/month
  
This is where Edge processing justifies itself:
  IoT Edge on factory floor aggregates 10K sensors:
    → Raw: 10,000 messages/second per factory
    → After edge aggregation (10s window): 1,000 aggregated messages/second
    → Reduction: 10×
    → Cloud ingestion: 100K messages/second → 8.64B/day → ~30 S3 units → ~$75K/month
  
  PLUS: edge processing enables local alerting even when connectivity is lost.
  Machine overheating alert fires in <100ms locally, not <5s via cloud.

Protocol choices:
  MQTT: lightweight, binary, <100 byte headers → best for constrained sensors
  AMQP: session-based, better for reliable messaging in connected scenarios
  HTTPS: stateless, works through any proxy → use only for infrequent events
  
  IoT Hub supports all three. Sensors: MQTT. Backend consumers: AMQP.
```

#### Probe 3: "ADX vs InfluxDB vs TimescaleDB — why ADX for time series?"

```
All three are purpose-built time-series stores. Know the differences.

Azure Data Explorer (ADX):
  + Kusto Query Language (KQL) — extremely powerful for time-series analytics
  + Petabyte-scale (built for Azure Monitor, used by Microsoft internally)
  + Streaming ingestion: <5s latency from IoT Hub to query-able
  + Built-in anomaly detection functions: series_decompose_anomalies()
  + Free tier: Dev SKU at $0 for small PoC
  - KQL has a learning curve if team knows only SQL
  - Cost at large scale: Compute + storage (but Data Sharding is automatic)

InfluxDB:
  + SQL-like Flux query language (v2)
  + Great for Grafana integration
  + OSS version is free (but limited to single node)
  + Strong in DevOps/monitoring use cases
  - InfluxDB Cloud: limited to 30-day retention on free tier
  - Clustering in OSS is complex; Enterprise is expensive
  - Not PB-scale without significant effort

TimescaleDB (PostgreSQL extension):
  + Native SQL (team already knows it)
  + ACID transactions (if you need transactional consistency)
  + pgvector extension available (unified store for IoT + AI)
  - Not designed for PB-scale time series
  - Compression less efficient than columnar stores at scale
  - Query performance at 1B+ rows: needs careful partitioning

Decision for factory IoT at PB scale:
  ADX = clear winner. KQL series_decompose_anomalies() alone saves months of 
  custom ML work. Streaming ingestion from IoT Hub is native (zero code).
  
  Use InfluxDB if: team loves Grafana and data is <1TB per year.
  Use TimescaleDB if: data is already in PostgreSQL and scale is modest.
```

---

---

## Use Case 7: Enterprise Data Lakehouse Migration

### The Interview Question
> *"A large enterprise has 200TB of data across Oracle, SQL Server, and 50 on-prem NAS shares. They want to migrate to a modern data lakehouse on Azure with less than 4 hours of downtime for their reporting systems."*

---

### Step 1 — Clarifying Questions

```
1. What's the reporting SLA? "4-hour downtime" — during business hours or maintenance window?
2. Are there real-time feeds into the databases? (CDC opportunity)
3. What's the data freshness requirement post-migration? (near-real-time vs nightly)
4. Are there any large tables? (a 50TB single table changes the migration approach)
5. Oracle version? (affects which CDC tools work)
6. Does the organization have Databricks licenses, or are we starting from scratch?
```

---

### Step 2 — Architecture

```
MIGRATION APPROACH:
  Phase 1: Full load (offline, parallel)
    Oracle + SQL Server → Azure Database Migration Service (DMS)
    NAS shares → AzCopy + Data Box (if >40TB per NAS)
    
  Phase 2: CDC sync (while Phase 1 runs)
    Debezium → Kafka (Event Hubs Kafka surface) → Databricks Delta merge
    Captures all changes while migration is in progress
    
  Phase 3: Cutover (the 4-hour window)
    T-0: Freeze source databases (read-only)
    T+0:30: Final CDC drain (wait for lag = 0)
    T+1:00: Validate row counts + checksums
    T+2:00: Switch reporting tools to Databricks SQL endpoints
    T+4:00: Window complete — sources remain hot (read-only) for 72h rollback

LAKEHOUSE ARCHITECTURE:
  Raw Zone:    ADLS Gen2, original format, immutable, partitioned by source+date
  Curated Zone: Delta Lake, schema enforced, deduped, partitioned by domain
  Enriched Zone: Delta Lake, business logic applied, SCD Type 2 for history
  Serving Zone:  Databricks SQL Warehouse, Delta views, Power BI direct query

GOVERNANCE:
  Microsoft Purview: data catalog, lineage (source → raw → curated → enriched)
  Unity Catalog (Databricks): fine-grained table/column access control
  Azure Policy: deny data exfiltration, enforce encryption, enforce private endpoints
```

---

### Step 3 — Deep Dives

#### Probe 1: "Walk me through CDC. How does it actually work?"

```
CDC = Change Data Capture. Instead of polling for changes,
the database's transaction log is read directly.

How Debezium CDC works (Oracle/SQL Server):
  1. Source DB has a transaction log (Oracle: redo log; SQL Server: transaction log)
     Every INSERT, UPDATE, DELETE is written here BEFORE it commits to tables
  2. Debezium runs as a Kafka Connect connector
  3. Debezium reads the log continuously using log mining API
     Oracle: LogMiner API
     SQL Server: SQL Server CDC feature (must be enabled first)
  4. Each change becomes a Kafka message:
     {
       "op": "u",          // u=update, c=create, d=delete, r=read(snapshot)
       "before": { "id": 1, "balance": 1000 },
       "after":  { "id": 1, "balance": 950 },
       "source": { "table": "accounts", "ts_ms": 1716200000000 }
     }
  5. Databricks reads this Kafka topic (Event Hubs Kafka surface)
     → MERGE INTO target USING source ON id = source.id
       WHEN MATCHED THEN UPDATE
       WHEN NOT MATCHED THEN INSERT
       WHEN NOT MATCHED BY SOURCE THEN DELETE  // for delete events

Key gotcha: Oracle CDC requires LogMiner license (additional Oracle cost).
Alternative for Oracle: AWS DMS (ironically) or Attunity/Qlik (commercial CDC tools).

SQL Server CDC: built-in, free. Enable per table:
  EXEC sys.sp_cdc_enable_table 
    @source_schema = 'dbo',
    @source_name = 'accounts',
    @role_name = NULL  -- or restrict to a specific role

Lag monitoring (critical for migration):
  If Kafka consumer lag > 0, CDC is behind. 
  You cannot cut over until lag = 0 AND has been 0 for 5+ minutes.
  Monitor: kafka-consumer-groups.sh --describe → Lag column
  Set alert: Azure Monitor on Event Hubs offset lag > 1000 messages
```

#### Probe 2: "What is Delta Lake and why does it matter?"

```
Delta Lake is an open-source storage layer on top of Parquet files that adds:

1. ACID transactions:
   Parquet files are immutable. Without Delta, concurrent writes corrupt data.
   Delta uses a transaction log (_delta_log/) — each commit is a JSON entry.
   Two writers → optimistic concurrency → one wins, one retries. No corruption.

2. Schema enforcement:
   Writing wrong schema → fails immediately with clear error.
   Without this → silent schema corruption in raw Parquet (discovered weeks later).

3. Schema evolution:
   ALTER TABLE ADD COLUMN: works without rewriting all files.
   SET spark.databricks.delta.schema.autoMerge.enabled = true
   → CDC schema changes don't break the pipeline.

4. Time travel (critical for migration validation):
   SELECT * FROM accounts VERSION AS OF 5  -- read table at commit #5
   SELECT * FROM accounts TIMESTAMP AS OF '2026-05-19'
   
   Use case: "Our report yesterday was wrong. What did the data look like at 6pm?"
   → No data warehouse snapshot needed. Delta gives you this for free.

5. OPTIMIZE + ZORDER:
   Small files problem: Spark writes hundreds of small Parquet files → slow queries.
   OPTIMIZE accounts ZORDER BY (account_id, date)
   → Compacts small files into larger ones, co-locates related data on disk
   → Query improvement: 10–50× for filtered queries

6. Deletion vectors + row-level deletes:
   GDPR right-to-erasure: DELETE FROM accounts WHERE customer_id = 12345
   → Delta records the deletion without rewriting all files (fast)
   → Regular Parquet: you must rewrite the entire partition (slow)
```

---

---

## Use Case 8: CI/CD Platform for 50-Team Engineering Org

### The Interview Question
> *"You're the platform architect for an org with 50 engineering teams, all building microservices that deploy to AKS. Design the CI/CD platform. Each team should be able to deploy independently but you need governance, security scanning, and cost control."*

---

### Step 1 — Clarifying Questions

```
1. What's the current state? (GitHub? Azure DevOps? GitLab?)
2. How many deployments per day? (50 teams × 5 deploys/day = 250 deploys)
3. Are there compliance requirements? (SOX, PCI — change management approval required)
4. Monorepo or polyrepo?
5. How many environments? (dev, staging, prod? Or per-team dev namespaces?)
6. Container registry: one per team or shared?
```

---

### Step 2 — Architecture

```
CODE → BUILD → TEST → PUBLISH → DEPLOY

GitHub (code hosting, PR workflow, branch protection rules)
    │
    ▼ GitHub Actions (CI)
    ├── Build: docker build → push to Azure Container Registry (ACR)
    ├── Test: unit tests, integration tests, SAST (CodeQL), dependency scan
    ├── DAST: OWASP ZAP (on staging deployment)
    ├── Secret scan: detect committed secrets (GitHub Secret Scanning)
    └── Publish: Helm chart version to ACR OCI registry
    │
    ▼ GitOps (CD) — ArgoCD or Flux v2
    ├── App repo: application code + Helm values per environment
    ├── Config repo: ArgoCD ApplicationSet definitions (managed by platform team)
    │
    ▼ AKS cluster (per environment: dev, staging, prod)
    ├── Namespace per team (team-payments, team-accounts, team-fraud)
    ├── ArgoCD watches config repo → syncs to cluster automatically
    └── Drift detection: ArgoCD alerts if cluster state diverges from Git

GOVERNANCE:
  Branch protection: require PR review, require status checks (all CI jobs must pass)
  Environment protection rules (GitHub): production requires approval from 2 reviewers
  Azure Policy: deny images from registries other than company ACR
  OPA/Gatekeeper: enforce pod security standards, resource limits on all deployments
  Image signing: Azure ACR Content Trust (Notation/Cosign) — sign images in CI
```

---

### Step 3 — Deep Dives

#### Probe 1: "How do GitHub Actions authenticate to Azure without secrets?"

```
This is a key security probe. Answer: OIDC (Workload Identity Federation).

Old way (wrong): Store Azure service principal credentials as GitHub secrets
  → If GitHub is compromised → attacker has your Azure credentials
  → Secrets expire, must be rotated manually (people forget)
  → Can't scope per-repo easily

New way (correct): OIDC federated credentials
  1. Create User-Assigned Managed Identity in Azure: mi-github-actions-cicd
  2. Add federated credential:
     Issuer: https://token.actions.githubusercontent.com
     Subject: repo:MyOrg/MyRepo:environment:production
             (or :ref:refs/heads/main for branch-scoped)
     Audience: api://AzureADTokenExchange
  3. Assign RBAC: mi-github-actions-cicd → AcrPush on ACR, Contributor on RG
  4. In GitHub Actions workflow:

  permissions:
    id-token: write   # REQUIRED — allows Actions to request OIDC token
    contents: read

  jobs:
    deploy:
      steps:
        - uses: azure/login@v2
          with:
            client-id: ${{ secrets.AZURE_CLIENT_ID }}       # Not a secret — just an ID
            tenant-id: ${{ secrets.AZURE_TENANT_ID }}       # Not a secret
            subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }} # Not a secret
            # No client_secret! Token exchange happens via OIDC

  How it works:
    Actions requests OIDC token from GitHub → signed JWT with repo/branch claims
    Azure validates JWT against GitHub's JWKS endpoint
    If issuer + subject match federated credential → issues Azure AD token
    No long-lived secret ever stored anywhere.

  Scope per environment:
    repo:MyOrg/payments:environment:production → can only deploy from production env
    (GitHub environment requires approval → humans must approve prod deploys)
```

#### Probe 2: "GitOps: push-based vs pull-based. What's the difference and why does it matter?"

```
Push-based CI/CD (traditional):
  CI pipeline runs kubectl apply or helm upgrade directly
  Pipeline needs kubectl credentials stored as secrets
  If cluster is unavailable during deploy: deploy fails, pipeline has to retry
  Drift: someone manually changes something in cluster → no one notices

Pull-based GitOps (ArgoCD/Flux):
  Agent (ArgoCD) runs INSIDE the cluster
  Agent polls Git repo every 3 minutes (or webhook-triggered)
  If Git says "3 replicas", agent ensures cluster has 3 replicas
  No external credentials needed (agent has in-cluster RBAC)
  Drift detection: if someone does kubectl edit manually → ArgoCD detects + alerts
                   and can auto-revert (sync policy: automated + self-heal: true)

Why pull-based wins for AKS:
  1. Cluster credentials never leave the cluster (no secret in CI)
  2. Disaster recovery: new cluster, point ArgoCD at same Git repo → state restored
  3. Audit trail: Git history IS your deployment history (who, what, when)
  4. Rollback: git revert → ArgoCD auto-syncs → rollback in <3 minutes

ArgoCD multi-team setup (platform team manages this):
  ApplicationSet: generates one ArgoCD Application per team namespace
  
  apiVersion: argoproj.io/v1alpha1
  kind: ApplicationSet
  metadata:
    name: team-apps
  spec:
    generators:
      - git:
          repoURL: https://github.com/MyOrg/platform-config
          files:
            - path: "teams/*/config.yaml"  # one file per team
    template:
      spec:
        source:
          repoURL: "{{repoURL}}"           # from team's config.yaml
          targetRevision: "{{targetRevision}}"
          path: "{{helmChartPath}}"
        destination:
          server: https://kubernetes.default.svc
          namespace: "{{teamNamespace}}"
        syncPolicy:
          automated:
            prune: true       # remove resources deleted from Git
            selfHeal: true    # revert manual cluster changes
```

#### Probe 3: "How do you handle secrets in a GitOps world? You can't put them in Git."

```
This is the critical gap in GitOps. Git stores everything — but secrets must not be in Git.

Option A: External Secrets Operator (ESO) — recommended
  CRD: ExternalSecret → polls Azure Key Vault → creates/updates Kubernetes Secret

  apiVersion: external-secrets.io/v1beta1
  kind: ExternalSecret
  metadata:
    name: payment-api-secrets
    namespace: team-payments
  spec:
    refreshInterval: 1h
    secretStoreRef:
      name: azure-keyvault-store
      kind: SecretStore
    target:
      name: payment-api-secrets  # creates this K8s Secret
    data:
      - secretKey: db-password         # key in K8s Secret
        remoteRef:
          key: payment-db-password     # key in Key Vault

  Result: Secret lives in Key Vault. ExternalSecret in Git (safe — no secret value).
  K8s Secret auto-synced hourly. Secret rotation in Key Vault → pods get new value.

Option B: Sealed Secrets (Bitnami)
  SealedSecret CRD: encrypts the value with cluster-specific public key
  Encrypted value IS safe to store in Git
  Only the cluster can decrypt (private key in cluster)
  Problem: if cluster is destroyed, private key is lost → must re-seal all secrets

Option C: Azure Key Vault CSI Driver
  Mount secrets directly as files or env vars into pods
  No Kubernetes Secret object created (secret never hits etcd)
  Uses Workload Identity for authentication
  Best for: highly sensitive secrets that should never hit etcd

Recommendation:
  External Secrets Operator for most secrets (database passwords, API keys)
  Key Vault CSI Driver for the most sensitive (encryption keys, signing certs)
  Never Sealed Secrets for new projects (operational complexity on cluster rebuild)
```

---

---

## Use Case 9: Global E-Commerce — Black Friday 10x Spike

### The Interview Question
> *"Design the architecture for a global e-commerce platform that normally handles 100K concurrent users but must handle 1 million concurrent users on Black Friday — a known, predictable event. Any downtime during Black Friday costs $500K/minute."*

---

### Step 1 — Architecture (Key Layers)

```
User → Azure Front Door (global anycast, WAF, DDoS Protection Standard)
     → APIM (product catalog cache, rate limit, bot detection)
     → AKS Pods (KEDA-scaled, stateless — critical)
     → Redis Cache (product catalog, cart, session)
     → CosmosDB (orders, inventory) — autoscale 400 → 40,000 RU
     → Service Bus (async order processing — decouple checkout from fulfillment)
     → Azure Functions (order processor, inventory deduct, notification)

CDN:
  Azure Front Door CDN: product images, CSS, JS (TTL 7 days)
  Dynamic caching: category pages, product pages (TTL 5 min)
  → 90% of traffic served from CDN edge, never hits origin
```

---

### Step 2 — Deep Dives

#### Probe 1: "How does KEDA scale pods for a predicted traffic spike?"

```
KEDA (Kubernetes Event-Driven Autoscaler) triggers on external metrics.
For Black Friday: pre-scale + event-driven reactive scale.

Pre-scale (scheduled):
  KEDA v2.10+ supports CronScaledObject:
  
  apiVersion: keda.sh/v1alpha1
  kind: ScaledObject
  metadata:
    name: product-api-scaler
  spec:
    scaleTargetRef:
      name: product-api
    minReplicaCount: 5       # normal baseline
    maxReplicaCount: 200     # Black Friday ceiling
    triggers:
      - type: cron
        metadata:
          timezone: "America/New_York"
          start: "0 6 * * 5"   # 6am Friday (pre-scale before traffic hits)
          end: "0 2 * * 6"     # 2am Saturday (scale back down)
          desiredReplicas: "50" # pre-warm to 50 before first user arrives

      - type: azure-servicebus   # react to real demand too
        metadata:
          queueName: checkout-jobs
          messageCount: "10"    # 1 more pod per 10 messages in queue
        authenticationRef:
          name: keda-trigger-auth

  Combined: 50 pods at 6am Friday (pre-scaled) + reactive growth to 200 if queue fills

Cluster Autoscaler (node-level, separate from KEDA):
  KEDA scales pods. But pods need nodes.
  Node provisioning takes 3–5 minutes (new VM + AKS join).
  Solution: pre-provision spare node pool in advance.
  
  az aks nodepool scale --cluster-name my-aks \
    --name blackfriday-pool \
    --node-count 50  # run this command 30 min before event
  
  Pods go from pending → running in seconds (node already exists).
  After event: scale back to 5 nodes.
  Cost: 50 nodes × 6 hours × $0.10/hour = $30. Worth it for $500K/min downtime risk.
```

#### Probe 2: "Cache-aside pattern — how does it work and what are the failure modes?"

```
Cache-aside (lazy loading) is the most common caching pattern.

Normal flow:
  1. App requests product P-12345
  2. Check Redis: MISS (not in cache)
  3. Query CosmosDB: get product data
  4. Write to Redis: SET "product:P-12345" <data> EX 300  (5 min TTL)
  5. Return data

Subsequent requests:
  1. App requests product P-12345
  2. Check Redis: HIT → return immediately (<2ms)
  3. Never hits CosmosDB

Failure modes to discuss (this is what separates senior answers):

1. Cache stampede (thundering herd):
   Scenario: TTL expires on P-12345 at exactly 12:01:00.000
             1,000 requests arrive at 12:01:00.001 → all MISS → all hit CosmosDB
   Fix: Probabilistic early expiration (PER):
     if remaining_ttl < random_threshold: refresh early, before TTL expires
   OR: Mutex lock — first miss acquires lock, others wait for lock to release with value

2. Cache penetration:
   Scenario: attacker queries product ID "P-DOESNOTEXIST" millions of times
   → Always misses Redis → always hits CosmosDB → DB overwhelmed
   Fix: Cache NULL results with short TTL (30 seconds)
     SET "product:P-DOESNOTEXIST" "null" EX 30
   OR: Bloom filter — fast probabilistic check: "does this ID exist?"
     If Bloom filter says NO → return 404 immediately, never check DB

3. Cache breakdown:
   Scenario: Redis instance fails → all traffic hits CosmosDB directly
   Fix: Redis clustering (Azure Cache for Redis with zone redundancy)
        Circuit breaker: if Redis is down → route to CosmosDB, log warning,
        but don't fail the request (degrade gracefully)
        CosmosDB autoscale: configured to handle burst if cache disappears

4. Stale data:
   Scenario: Product price updated in CosmosDB. Redis still has old price.
   Fix: Write-through (update cache when DB is updated) — adds complexity
   OR: Short TTL for price-sensitive data (30s instead of 5min)
   OR: Event-driven invalidation: CosmosDB change feed → Function → delete Redis key
```

---

---

## Use Case 10: Hub-Spoke Networking + Zero Trust for Enterprise

### The Interview Question
> *"Design the network architecture for a large enterprise migrating to Azure. They have 3 business units, a shared services team, and a connection back to two on-premises data centers. Walk me through the full networking design."*

---

### Step 1 — Architecture

```
ON-PREMISES DATA CENTERS:
  DC-Toronto ──┐ ExpressRoute Circuit 1 (10 Gbps, primary)
  DC-Montréal ──┘ ExpressRoute Circuit 2 (10 Gbps, redundant)
                     │
                     ▼
┌──────────────────────────────────────────────────────┐
│                  HUB VNET (10.0.0.0/16)              │
│  Managed by: Network/Platform team                    │
│                                                       │
│  ┌──────────────────────────────────────────────┐    │
│  │ Azure Firewall Premium (10.0.1.0/26)          │    │
│  │  - L7 inspection, IDPS, TLS inspection        │    │
│  │  - FQDN-based rules (allow *.microsoft.com)   │    │
│  └──────────────────────────────────────────────┘    │
│  ┌──────────────────────────────────────────────┐    │
│  │ VPN Gateway / ER Gateway (10.0.2.0/27)        │    │
│  └──────────────────────────────────────────────┘    │
│  ┌──────────────────────────────────────────────┐    │
│  │ Azure Bastion (10.0.3.0/27)                   │    │
│  │  - RDP/SSH to any spoke VM, no public IPs     │    │
│  └──────────────────────────────────────────────┘    │
│  ┌──────────────────────────────────────────────┐    │
│  │ Private DNS Zones                             │    │
│  │  - privatelink.blob.core.windows.net          │    │
│  │  - privatelink.database.windows.net           │    │
│  │  - (all Azure service private DNS zones)      │    │
│  └──────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────┘
         │                │                │
    VNet Peering      VNet Peering    VNet Peering
         │                │                │
    ┌────┴───┐       ┌────┴───┐      ┌────┴───┐
    │ BU1    │       │ BU2    │      │Shared  │
    │ Spoke  │       │ Spoke  │      │Services│
    │10.1/16 │       │10.2/16 │      │10.3/16 │
    └────────┘       └────────┘      └────────┘
```

---

### Step 2 — Deep Dives

#### Probe 1: "Private Endpoints — explain exactly what they do and where people go wrong."

```
Without Private Endpoint:
  Your AKS pod calls storage.blob.core.windows.net
  DNS resolves to: 52.167.132.100 (public IP)
  Traffic: AKS pod → Azure backbone → public endpoint → storage account
  Problem: data traverses public IP space (even if Azure-internal)
           NSG can't easily filter by service (only IP)
           Storage account must have public access enabled

With Private Endpoint:
  Create Private Endpoint → Azure injects a NIC into your subnet
  NIC gets a private IP: 10.1.2.5
  Azure creates DNS record in privatelink.blob.core.windows.net:
    stgmyaccount.blob.core.windows.net → CNAME → stgmyaccount.privatelink.blob.core.windows.net
    stgmyaccount.privatelink.blob.core.windows.net → 10.1.2.5
  Traffic: AKS pod → 10.1.2.5 (private IP) → storage (never leaves VNet)
  Storage account: public_network_access = Disabled (safe now)

The #1 mistake everyone makes:
  Private DNS Zone not linked to the VNet.
  
  If you create the Private Endpoint but don't link the Private DNS Zone to your VNet:
    nslookup stgmyaccount.blob.core.windows.net from VM in VNet
    → Returns 52.167.x.x (public IP) ← WRONG
    → Traffic hits public endpoint → if public access is disabled → 403 error
  
  Fix: Link the Private DNS Zone to every VNet that needs to resolve the endpoint
    az network private-dns link vnet create \
      --resource-group rg-dns \
      --zone-name privatelink.blob.core.windows.net \
      --name link-to-bu1-spoke \
      --virtual-network bu1-spoke-vnet \
      --registration-enabled false

  Debugging checklist for "can't reach storage from VM":
    1. nslookup <storage>.blob.core.windows.net → should return 10.x.x.x
    2. If returning public IP → Private DNS Zone not linked to this VNet
    3. If returning 10.x.x.x but still failing → NSG blocking port 443 to PE subnet
    4. If NSG OK → check Storage Firewall allows from this subnet or "trusted services"
```

#### Probe 2: "NSG vs Azure Firewall — when do you use each?"

```
This is extremely commonly asked. Know the crisp answer.

NSG (Network Security Group):
  Layer 3/4 only: IP address + port + protocol
  Applied at: subnet level OR NIC level
  Stateful: allows return traffic automatically
  Cost: FREE (included with VNet)
  Rules: allow/deny by source IP, destination IP, port
  
  Cannot do:
    × FQDN filtering ("allow *.microsoft.com") — only raw IPs
    × URL path filtering ("/api/v1" vs "/api/v2")
    × TLS inspection
    × Threat intelligence feeds
    × Logging to Log Analytics (requires NSG Flow Logs — separate setup)

Azure Firewall:
  Layer 3/4 + Layer 7: FQDN rules, URL filtering, TLS inspection
  Applied at: VNet level (hub VNet, all traffic routed through it)
  Cost: ~$1.25/hour = ~$900/month (Firewall Premium: ~$2.75/hour)
  Rules: FQDN tags ("WindowsUpdate"), URL categories, IDPS signatures
  
  Can do:
    ✓ "Allow pods to reach *.azuredatabricks.net" (FQDN)
    ✓ Inspect TLS traffic (Firewall Premium with TLS inspection cert)
    ✓ Azure Threat Intelligence: block known malicious IPs automatically
    ✓ IDPS (Intrusion Detection and Prevention System)

When to use which:
  NSG: subnet-level east-west control within a spoke ("app tier can't talk to DB tier directly")
  Azure Firewall: north-south traffic control (spoke → internet, on-prem → Azure)
  BOTH together: NSG as first line, Firewall as second line (defense in depth)

UDR (User-Defined Route) — the missing piece:
  Azure Firewall only inspects traffic routed through it.
  You MUST create a UDR on each spoke subnet: 0.0.0.0/0 → Firewall private IP
  Without UDR: traffic bypasses Firewall entirely (common mistake)
  
  az network route-table route create \
    --route-table-name rt-bu1-spoke \
    --name default-to-firewall \
    --address-prefix 0.0.0.0/0 \
    --next-hop-type VirtualAppliance \
    --next-hop-ip-address 10.0.1.4  # Firewall private IP
```

#### Probe 3: "ExpressRoute vs VPN Gateway — explain the real technical differences."

```
VPN Gateway:
  IPsec/IKE encrypted tunnel over public internet
  Bandwidth: Basic 100Mbps → VpnGw5 10Gbps
  Latency: variable (internet routing) — typically +10–30ms vs direct
  SLA: 99.9%
  Cost: ~$130–$1,400/month depending on SKU
  Setup: hours (just needs internet + shared key / certificate)
  Use when: Branch office connectivity, dev/test, backup path for ExpressRoute

ExpressRoute:
  Private dedicated circuit via connectivity provider (Bell, Rogers, Equinix)
  NOT over public internet — goes through provider's MPLS backbone to Microsoft
  Bandwidth: 50Mbps to 100Gbps
  Latency: consistent and low (provider-managed path) — typically <5ms to Azure region
  SLA: 99.95% (or 99.99% with redundant circuits)
  Cost: ~$55/month (50Mbps local) to $7,000+/month (10Gbps premium)
  Setup: weeks to months (provider circuit provisioning)
  Use when: Production workloads, compliance (data must not traverse internet), 
            high bandwidth (bulk data transfer, backup), latency-sensitive (trading)

BGP — the protocol both use (interviewers probe this):
  ExpressRoute uses BGP to exchange routes between on-prem and Azure.
  
  Azure advertises: "I can reach 10.0.0.0/16 (hub VNet) and 10.1.0.0/16 (spoke)"
  On-prem advertises: "I can reach 192.168.0.0/16 (corporate network)"
  
  Common BGP issue: on-prem BGP policy filters Azure routes
    → Azure subnets not reachable from on-prem
    Diagnosis: Get-AzExpressRouteCircuitRouteTable → shows what Azure sees from on-prem
    Fix: on-prem network team adjusts route filters / BGP policy
  
  Redundancy: always provision TWO ExpressRoute circuits (different providers or PoPs)
    → Both in active-active OR active-passive
    → BGP will automatically failover to secondary if primary goes down
    → Without redundancy: single ExpressRoute = single point of failure for all Azure connectivity
```

---

---

## 11. Decision Matrix — One-Page Cheat Sheet

```
STREAMING vs QUEUING:
  Need multiple consumers + event replay → Event Hubs
  Need guaranteed delivery + ordering + DLQ → Service Bus Premium
  Need simple pub/sub HTTP webhooks → Event Grid

COMPUTE:
  Stateless HTTP microservices → AKS
  Event-triggered, short-lived → Azure Functions
  Long-running batch, ML → Databricks / Azure ML
  <5 min workflow, low volume → Logic Apps

DATABASES:
  Key-value, global, variable schema → CosmosDB (partition by access pattern)
  Relational, ACID, < 4TB → Azure SQL
  Time-series, PB scale → Azure Data Explorer (ADX)
  Cache, sub-ms → Redis
  Analytics, complex joins → Synapse / Databricks SQL

NETWORKING (quick ref):
  Isolation within a spoke → NSG
  Traffic filtering hub→spoke → Azure Firewall + UDR
  Private access to PaaS services → Private Endpoint (+ DNS Zone link)
  On-prem < 10 Gbps, best-effort → VPN Gateway
  On-prem production, dedicated → ExpressRoute
  Global HTTP LB + WAF → Azure Front Door
  Multi-region DNS failover → Traffic Manager

AKS (quick ref):
  Pod-to-pod isolation → NetworkPolicy (requires Azure CNI + network policy flag)
  No stored secrets → Workload Identity (OIDC federated)
  Scale on queue depth → KEDA ScaledObject
  Multi-tenant isolation → Namespace + ResourceQuota + taint/toleration
  GitOps → ArgoCD (pull-based, drift detection)
  Secrets from Key Vault → External Secrets Operator OR Key Vault CSI Driver

AI/RAG (quick ref):
  Prevent hallucinations → grounding prompt + citation enforcement + confidence gate
  Best retrieval → hybrid BM25+vector+semantic reranker (not pure vector)
  Regulated AI → OSFI B-10: model card + SHAP explainability + human-in-loop + eval suite
  Cost at scale → Redis semantic cache (saves 30-40% OpenAI spend)
```

---

## Interviewer's Scoring Rubric (What They're Actually Evaluating)

```
JUNIOR (L3–L4) answer:
  "I'd use Azure for this" — names services without justification
  Cannot explain the 'why' behind choices
  Misses failure modes, doesn't mention monitoring/alerting

SENIOR (L5–L6) answer:
  Names service + explains tradeoff + knows the limits/gotchas
  Mentions cost, latency, SLA numbers
  Proactively addresses failure modes and DR
  Draws a reasonable architecture without prompting

PRINCIPAL (L7+) answer:
  Starts with clarifying questions — shapes the problem before answering
  States assumptions explicitly
  Explains what would change the answer (scale up: different choice)
  Addresses org/team constraints, not just technology
  Quantifies scale: "at 50K TPS, here's what breaks first and why"
  Mentions regulatory/compliance implications without being asked
  Knows the one gotcha that trips everyone up (NSG vs Firewall, DNS zone links,
  partition key design, idempotency, CDC lag monitoring)
```

---

*Document version: 1.0 | Created: 2026-05-20 | Author: Abhishek Sinha*  
*Repo: https://github.com/AbhishekSinha02/AzureArchitecture*  
*Related: [UnstructuredData-Migration-RAG-AgenticAI.md](./UnstructuredData-Migration-RAG-AgenticAI.md) · [TechStack-Comparison-DataMigration-RAG-AgenticAI.md](./TechStack-Comparison-DataMigration-RAG-AgenticAI.md)*
