# CLAUDE.md — Azure Architecture Master Reference
> **Purpose:** Living reference for Architecture design, Presentations, Troubleshooting, Networking, and High-Volume Data Migration across enterprise use cases. Continuously updated. Interview + project depth reference.
> **How to use:** Paste this file into any Claude conversation. Say "Using CLAUDE.md context, help me with [task]." Claude will generate depth-first, production-grade responses aligned to your architect level.

---

## 1. WHO I AM & WHAT I BUILD

```
Role         : Cloud / AI / Data Architect — Senior/Lead level
Target       : Remote/Hybrid, Canadian financial sector + enterprise
Stack        : Azure-first, AWS-aware, cloud-agnostic reasoning
Core skills  : AKS, Terraform, GitHub Actions, Azure OpenAI, AI Search,
               CosmosDB, Service Bus, Event Hubs, Data Factory, Synapse,
               C# .NET 8, FastAPI, Python, PostgreSQL, Redis
Certifications target: AZ-305, AZ-400, DP-203, AI-102
```

---

## 2. HOW CLAUDE SHOULD RESPOND TO ME

```
- Think from a Principal Architect perspective, not just a developer
- Always include: WHY a decision was made + tradeoffs + alternatives rejected
- Lead with numbers (scale, latency, cost, throughput) before naming tools
- Include small / medium / large enterprise variants for every pattern
- Map Azure services to AWS equivalents where relevant
- Flag Canadian regulatory constraints (OSFI B-10, PIPEDA, FINTRAC) when applicable
- Production-grade code: heavily commented, error-handled, retry logic included
- Break large topics into parts — max 1hr reading per document
- Interview mode: surface real questions an interviewer would ask at this level
```

---

## 3. ENTERPRISE SIZE DEFINITIONS

| Size | Users | Data Volume | Team | Budget Signal |
|------|-------|-------------|------|---------------|
| **Small** | <500 | <1TB | 2–5 devs | Managed services, minimal ops |
| **Medium** | 500–10K | 1–50TB | 5–20 devs | Hybrid managed + custom |
| **Large** | 10K+ | 50TB–PB scale | 20+ devs | Full platform, dedicated SRE |

---

## 4. CORE ARCHITECTURE PATTERNS

### 4.1 Cloud Migration — 6Rs Framework
```
Rehost      → Lift & shift. IaaS. Fastest. No cloud optimization.
Replatform  → Minor changes. App Service instead of IIS VM.
Repurchase  → Move to SaaS. Dynamics 365, Salesforce.
Refactor    → Re-architect. AKS microservices, event-driven.
Retire      → Decommission unused systems.
Retain       → Keep on-prem. Compliance, latency, legacy dependency.

Decision matrix:
- Time pressure + low complexity → Rehost
- Cost optimization target → Replatform
- Competitive differentiation → Refactor
- Regulatory constraint → Retain or Rehost in sovereign region
```

### 4.2 High-Volume Media Migration (Video / Audio / Files)
```
Scale signals:
  Small  : <10TB, <1K concurrent streams → Azure Blob + CDN
  Medium : 10–500TB, <50K streams → Azure Media Services + CDN + HLS
  Large  : 500TB–PB, 100K+ streams → Custom pipeline: Blob → Encoder → CDN → HLS/DASH

Key decisions:
  Ingestion   : AzCopy / Storage Transfer Service / Azure Data Box (offline >40TB)
  Storage     : Hot (active) / Cool (30d+) / Archive (180d+) — auto tiering policy
  Encoding    : Azure Media Services OR FFmpeg on AKS (cost tradeoff)
  Delivery    : Azure CDN (Verizon/Akamai) + HLS adaptive bitrate
  Metadata    : CosmosDB (fast lookup) + Azure AI Video Indexer (tagging)
  Security    : SAS tokens, Private Endpoints, DRM (Widevine/PlayReady)

AWS equivalent mapping:
  Azure Blob          → S3
  Azure CDN           → CloudFront
  Azure Media Svcs    → AWS Elemental MediaConvert
  Azure Data Box      → AWS Snowball

Interview Q: "How do you migrate 500TB of video with zero downtime?"
Answer: AzCopy parallel transfer + dark launch + DNS cutover + CDN warm-up
```

### 4.3 Networking Foundations
```
Hub-Spoke topology (enterprise standard):
  Hub VNet     : Firewall, VPN/ExpressRoute, DNS, Bastion
  Spoke VNets  : App workloads, AKS clusters, Data services
  Peering      : Hub ↔ Spoke (NOT spoke ↔ spoke directly)

Key services:
  Azure Firewall      : Layer 7, FQDN rules, threat intel
  NSG                 : Layer 4, subnet/NIC level
  Private Endpoint    : Service into your VNet (no public IP)
  Service Endpoint    : Route to service over Azure backbone
  VPN Gateway         : Site-to-site, P2S (max 10Gbps)
  ExpressRoute        : Dedicated circuit 1–100Gbps, SLA-backed
  Azure Bastion       : RDP/SSH without public IP
  Azure DNS           : Private zones for internal resolution
  Azure Front Door    : Global HTTP LB + WAF + CDN

Troubleshooting order:
  1. NSG flow logs → 2. Network Watcher → 3. Connection Monitor
  4. Effective routes → 5. IP flow verify → 6. Packet capture

Common pitfalls:
  - Forgot to add Private DNS Zone link to VNet
  - NSG on subnet AND NIC conflicting
  - ExpressRoute BGP route not propagated
  - AKS uses different subnet CIDR than planned (exhaustion)
```

### 4.4 AKS Production Architecture
```
Control plane  : Managed by Azure (free tier vs paid uptime SLA)
Node pools     : System pool (reserved) + User pool(s) (workloads)
Networking     : Azure CNI (routable IPs) vs Kubenet (NAT — avoid for large)
Identity       : Workload Identity (OIDC federated) — NOT service principal secrets
Ingress        : NGINX or Azure Application Gateway Ingress Controller (AGIC)
Storage        : Azure Files (RWX), Azure Disk (RWO), Blob CSI driver
Scaling        : HPA (CPU/mem) + KEDA (event-driven) + Cluster Autoscaler
Security       : Azure Policy + OPA/Gatekeeper, Pod security standards
GitOps         : ArgoCD or Flux v2
Observability  : Prometheus + Grafana OR Azure Monitor + Container Insights

Two managed identities (critical interview point):
  1. Control plane MI  → ARM operations (node provisioning, LB rules)
  2. Kubelet MI        → ACR pull, Key Vault, storage access from nodes

Workload Identity flow:
  Pod SA → OIDC token → Azure AD federated credential → Azure resource
  (No secrets stored anywhere — token exchange only)
```

---

## 5. USE CASE PATTERNS

### 5.1 AI / GenAI Use Case
```
Pattern: RAG (Retrieval Augmented Generation)
Stack  : Azure OpenAI (GPT-4o) + AI Search (hybrid BM25+RRF) + Document Intelligence
         + CosmosDB (session/history) + Service Bus (async jobs)

Pipeline:
  Ingest  → Doc Intelligence → chunk → embed (ada-002) → AI Search index
  Query   → embed query → AI Search (vector + keyword) → GPT-4o → response

Scale tiers:
  Small  : Azure OpenAI Consumption + AI Search Basic (1 replica)
  Medium : PTU (Provisioned Throughput Units) + AI Search Standard S2
  Large  : PTU + AI Search Storage Optimized + Redis cache (semantic caching)

AI Governance (critical for banking interviews):
  - Azure AI Content Safety (input/output filtering)
  - Prompt injection detection
  - PII detection + redaction (Presidio / AI Language)
  - Model cards + eval pipelines (LLM evals: RAGAS, PromptFlow)
  - OSFI B-10: explainability, human oversight, model risk management
  - Audit logging: all LLM calls logged to Log Analytics

LLMOps (production AI — key gap area):
  - Drift detection: monitor output quality over time
  - A/B testing models: traffic split via APIM policies
  - Prompt versioning: treat prompts as code in Git
  - Eval suite: faithfulness, answer relevancy, context precision
  - Rollback: keep previous model deployment active

Interview Qs:
  "How do you prevent hallucinations in a banking chatbot?"
  → Grounding + citation enforcement + confidence threshold + human-in-loop
  "How do you handle PII in document processing?"
  → Detect at ingestion (AI Language), redact before indexing, audit log
```

### 5.2 IoT Use Case
```
Ingestion  : IoT Hub (device management + D2C/C2D messaging)
             OR Event Hubs (pure streaming, no device mgmt)
Stream     : Stream Analytics (SQL-based, <100K events/s)
             OR Azure Databricks Structured Streaming (complex, PB scale)
Storage    : Time-series → InfluxDB / Azure Data Explorer (ADX)
             Cold storage → ADLS Gen2 (Parquet/Delta format)
Alerts     : Azure Monitor Alerts + Logic Apps / Functions
Dashboard  : Power BI Embedded OR Grafana

Scale tiers:
  Small  : IoT Hub Free/S1 + Stream Analytics 1 SU + Blob storage
  Medium : IoT Hub S3 + Databricks + ADX + ADLS
  Large  : Event Hubs (Kafka surface) + Databricks + ADX + Synapse

Edge patterns:
  Azure IoT Edge (containerized workloads on device)
  Azure Sphere (secure MCU-level)
  MQTT broker → IoT Hub (bridge pattern)

Interview Q: "Design IoT for 1M connected factory sensors"
→ Event Hubs partitioned by factory ID, Databricks Delta Lake,
  ADX for hot analytics, ADLS for cold, ADT for digital twin
```

### 5.3 Banking / Financial Services
```
Key constraints:
  OSFI B-10     : AI/ML model risk management, explainability required
  PIPEDA/PCI    : Data residency Canada, encryption in transit + rest
  FINTRAC       : AML transaction monitoring, audit trail 7 years
  SR 11-7       : Model validation (US reference, Canadian banks aware)

Architecture non-negotiables:
  - Private Endpoints for ALL data services (no public access)
  - Customer-managed keys (CMK) via Key Vault
  - Azure Policy: deny non-compliant resources at subscription level
  - Immutable audit logs: Log Analytics + Azure Blob (WORM)
  - Network: ExpressRoute preferred over VPN for core banking
  - Disaster recovery: active-active across paired regions (Canada Central + East)

Payments pattern:
  API Gateway (APIM) → Service Bus (guaranteed delivery)
  → Azure Functions (idempotent processing)
  → CosmosDB (ACID with session consistency)
  → Event Grid (notification fanout)

Fraud detection pattern:
  Event Hubs → Stream Analytics / Databricks Streaming
  → ML model scoring (AML Real-time endpoint)
  → Cosmos (flag store) → Alert pipeline

Interview Q: "How do you ensure 99.99% uptime for a payment API?"
→ Active-active deployment, traffic manager, circuit breaker (Polly),
  idempotency keys, dead letter queue, chaos engineering tests
```

### 5.4 Insurance Use Case
```
Core workloads: Policy management, Claims processing, Actuarial ML

Document processing pattern:
  Claims docs → Document Intelligence (custom model)
  → extract fields → validate → CosmosDB
  → trigger workflow (Logic Apps / Durable Functions)

ML for underwriting:
  Tabular data → Synapse → AML AutoML or custom XGBoost
  → explainability: SHAP values (regulatory requirement)
  → model endpoint → APIM → underwriting API

Data platform:
  Source systems (policy, claims, billing) → ADF ingestion
  → ADLS Gen2 (raw/curated/enriched zones) → Synapse Analytics
  → Power BI (actuarial dashboards)

Scale:
  Small  : App Service + SQL DB + Logic Apps + Document Intelligence
  Medium : AKS + CosmosDB + Synapse + AML
  Large  : Full lakehouse + real-time streaming + multi-region

Compliance: SOC2, OSFI, provincial insurance regulators
```

### 5.5 Retail Use Case
```
Core workloads: Product catalog, Inventory, Orders, Personalization

Personalization (AI):
  Clickstream → Event Hubs → Databricks
  → Azure Personalizer / custom rec model
  → CosmosDB (user profile) → API response

Peak load (Black Friday) pattern:
  Azure Front Door (global LB + WAF)
  → APIM (throttling, caching) → AKS (KEDA autoscale)
  → Redis Cache (product/cart) → CosmosDB (orders)
  → Service Bus (order processing async)

Scale design for 10x traffic spikes:
  - KEDA: scale AKS pods on Service Bus queue depth
  - CosmosDB: autoscale RU (400 → 4000 in seconds)
  - Redis: cache-aside for product catalog (TTL 5min)
  - CDN: static assets + product images cached at edge

Omnichannel:
  POS + Web + Mobile + App → APIM (unified) → microservices
  → Cosmos (single customer view)

Interview Q: "Design for 1M concurrent users on product launch day"
→ Front Door + APIM + AKS KEDA + Redis + Cosmos autoscale
   + CDN for static + async order processing via Service Bus
```

### 5.6 Logistics Use Case
```
Core workloads: Shipment tracking, Route optimization, Invoice processing

Real-time tracking:
  IoT/GPS devices → IoT Hub → Stream Analytics
  → Azure Maps (geospatial) → CosmosDB (position store)
  → SignalR (push to dashboard)

Invoice processing (AI):
  PDF invoices → Document Intelligence (custom model)
  → extract: vendor, amount, line items, dates
  → validation rules → ERP integration (ADF)
  → AI Search (invoice search + audit)

Route optimization:
  Azure Maps Route API + custom ML (OR-Tools)
  → recommendations → driver app via API

Multi-modal shipment visibility:
  Air/Sea/Land status feeds → Event Hubs
  → Databricks (enrichment) → CosmosDB → Power BI

Scale: Medium–Large (global logistics = large by default)
Latency target: <500ms for tracking queries (Redis + Cosmos indexed)
```

---

## 6. HIGH-VOLUME DATA MIGRATION PATTERNS

### 6.1 On-Prem to Azure Data Migration
```
Volume tiers:
  <1TB    : AzCopy / Azure Storage Explorer / ADF copy activity
  1–10TB  : ADF self-hosted IR + parallel copy
  10–40TB : AzCopy with parallel threads + ExpressRoute/VPN
  >40TB   : Azure Data Box (offline shipping) or Data Box Heavy

Steps (production migration):
  1. Assess    : Azure Migrate (discovery + sizing)
  2. Baseline  : capture current state, SLAs, dependencies
  3. Network   : size ExpressRoute or VPN bandwidth
  4. Pilot     : migrate 5–10% data, validate checksums
  5. Delta sync: ADF incremental (watermark or CDC pattern)
  6. Cutover   : freeze source, final sync, DNS/app config switch
  7. Validate  : row counts, checksums, app smoke test
  8. Rollback plan: keep source hot for 72h post-cutover

CDC (Change Data Capture) for zero-downtime DB migration:
  Source DB → Debezium (or ADF CDC) → Event Hubs
  → target DB sync → cutover when lag = 0
```

### 6.2 Data Architecture Patterns (Structured + Unstructured)
```
Lakehouse pattern (modern standard):
  Raw zone      : ADLS Gen2 (JSON, CSV, Parquet — as-is)
  Curated zone  : Delta Lake (schema enforced, deduplicated)
  Enriched zone : Delta Lake (business logic applied)
  Serving zone  : Synapse / Databricks SQL (query layer)

Hot / Warm / Cold tiering:
  Hot   : Redis Cache + CosmosDB (ms latency, <1TB)
  Warm  : Synapse dedicated pool / Databricks (min latency, TB scale)
  Cold  : ADLS Gen2 Archive tier (cents/GB, hours retrieval)

Streaming vs Batch decision:
  Streaming : Event-driven, fraud, IoT, real-time personalization
  Batch     : Reports, ETL, nightly aggregation, model training
  Lambda    : Both paths (complex, justify before choosing)
  Kappa     : Stream-only, reprocess from log (simpler ops)

Unstructured at scale (documents, images, video):
  Ingestion  → Azure Blob / Data Lake
  Processing → Document Intelligence / Video Indexer / Custom Vision
  Metadata   → AI Search (searchable) + CosmosDB (structured lookup)
  Archive    → Blob Archive tier + lifecycle policy

Interview Q: "How do you handle 10M documents per day?"
→ Event-driven: Blob trigger → Service Bus → AKS workers
  → Document Intelligence → AI Search index → CosmosDB metadata
  Scale: KEDA on Service Bus queue depth, parallel AKS pods
```

---

## 7. TROUBLESHOOTING PLAYBOOKS

### 7.1 AKS Troubleshooting
```
Pod not starting:
  kubectl describe pod <name> → check Events section
  Common: ImagePullBackOff (ACR auth), CrashLoopBackOff (app error),
          Pending (insufficient nodes — check cluster autoscaler)

Service not reachable:
  1. kubectl get svc → check ClusterIP/ExternalIP
  2. kubectl get endpoints → check pod selectors match
  3. Check NSG on AKS subnet (port blocked?)
  4. Check Ingress logs: kubectl logs -n ingress-nginx

High memory/CPU:
  kubectl top pods / kubectl top nodes
  Check HPA: kubectl get hpa
  Check resource requests/limits set correctly

Workload Identity not working:
  1. Check pod has serviceAccountName set
  2. Check SA has azure.workload.identity/client-id annotation
  3. Check federated credential in Azure AD matches SA + namespace + cluster OIDC URL
  4. Check pod has label: azure.workload.identity/use: "true"
```

### 7.2 Networking Troubleshooting
```
Can't reach Azure service from VNet:
  1. Is Private Endpoint created? → check portal
  2. Is Private DNS Zone linked to VNet? → most common miss
  3. nslookup <service>.blob.core.windows.net from VM in VNet
     → should return 10.x.x.x (private IP), not 52.x (public)
  4. Check NSG allows port 443 to private endpoint subnet

ExpressRoute issues:
  1. Check BGP peer state: Get-AzExpressRouteCircuitPeeringConfig
  2. Check route table: Get-AzExpressRouteCircuitRouteTable
  3. Verify AS path not filtered by on-prem BGP policy
  4. Check BFD (Bidirectional Forwarding Detection) enabled

VPN Gateway:
  Check shared key mismatch, IKE policy mismatch (Phase 1/2),
  on-prem firewall blocking UDP 500/4500
```

### 7.3 Data Pipeline Troubleshooting (ADF / Databricks)
```
ADF pipeline failed:
  1. Check Activity Run output in Monitor tab
  2. Self-hosted IR connectivity: check firewall allows 443 outbound
  3. Credential: MSI vs Service Principal — check role assignment
  4. Throttling: increase DIU (Data Integration Units)

Databricks cluster not starting:
  1. Check workspace quota (vCPU limits per region)
  2. Check subnet has enough IPs (Databricks needs 2 IPs per node)
  3. Check NSG allows Databricks control plane IPs

Delta Lake issues:
  DELTA_SCHEMA_CHANGE: schema evolution not enabled
  → set spark.databricks.delta.schema.autoMerge.enabled = true
  OPTIMISTIC_TRANSACTION_CONFLICT: concurrent writes
  → use merge pattern, not overwrite
```

---

## 8. PRESENTATION STRUCTURE TEMPLATES

### 8.1 Architecture Proposal (Client-facing)
```
Slide 1: Executive Summary — problem, solution, outcome (3 bullets max)
Slide 2: Current State — pain points, scale, constraints
Slide 3: Proposed Architecture — diagram (hub-spoke, data flow)
Slide 4: Key Design Decisions — 3 decisions with WHY + alternatives rejected
Slide 5: Migration Approach — phases, timeline, risk mitigation
Slide 6: Security & Compliance — Zero Trust, regulatory alignment
Slide 7: Cost Model — small/medium/large cost tiers, FinOps optimizations
Slide 8: Success Metrics — SLAs, KPIs, how we measure done
Slide 9: Risks & Mitigations — top 3 risks with mitigation plan
Slide 10: Next Steps — 3 concrete actions with owners
```

### 8.2 Technical Deep Dive (Engineering team)
```
Section 1: Architecture overview + component responsibilities
Section 2: Data flow (ingestion → processing → serving)
Section 3: Security model (identity, network, data)
Section 4: Scalability design (what breaks first + how we fix it)
Section 5: Observability (metrics, logs, traces, alerts)
Section 6: Deployment (IaC, CI/CD, GitOps)
Section 7: Runbooks (key failure scenarios + recovery steps)
```

### 8.3 Interview Whiteboard Answer Structure
```
Step 1: Clarify requirements (scale, latency, SLA, budget, compliance)
Step 2: State assumptions aloud
Step 3: Start with a simple diagram, then add complexity
Step 4: Lead with tradeoffs ("I chose X over Y because at this scale...")
Step 5: Address failure modes proactively
Step 6: Mention cost and operational complexity
Step 7: Summarize: "In short, this gives us [outcome] at [cost] with [SLA]"
```

---

## 9. INTERVIEW QUESTION BANK

### Architecture Design
```
Q: Design a real-time fraud detection system for a Canadian bank
A: Event Hubs → Stream Analytics (rule-based, <100ms) + Databricks (ML scoring)
   → Redis (fast flag lookup) → CosmosDB (case management)
   Compliance: OSFI B-10, immutable audit log, explainability (SHAP)

Q: How would you migrate a 200TB Oracle database to Azure with <4h downtime?
A: Azure Database Migration Service + CDC (Debezium) for delta
   ExpressRoute for bandwidth, Data Box for bulk if needed
   Phases: assess → pilot → incremental sync → cutover window

Q: Design multi-tenant SaaS on AKS for 500 enterprise customers
A: Namespace isolation per tenant OR separate node pool per tier
   Azure AD B2C for auth, APIM for API management
   CosmosDB with partition key = tenantId

Q: How do you handle a 10x traffic spike?
A: Front Door + APIM (throttle/cache) + AKS KEDA
   + CosmosDB autoscale + Redis + async offload via Service Bus
   Pre-scale known events, chaos test regularly
```

### AI / Data
```
Q: How do you ensure AI model fairness in lending decisions?
A: SHAP for explainability, fairness metrics (demographic parity),
   Azure Responsible AI dashboard, human review threshold,
   OSFI model risk management framework

Q: Design a vector search system for 100M documents
A: Azure AI Search (vector + keyword hybrid), HNSW index,
   chunking strategy (512 tokens overlap 50), ada-002 embeddings,
   Redis semantic cache for repeated queries

Q: How do you prevent prompt injection in a customer chatbot?
A: Input validation + Azure Content Safety + system prompt hardening
   + output parsing (never eval user input) + rate limiting per user
```

### AWS Parity (critical for multi-cloud roles)
```
Azure Blob Storage          ↔ AWS S3
Azure Data Factory          ↔ AWS Glue
Azure Databricks            ↔ AWS EMR / AWS Glue (Spark)
Azure Synapse Analytics     ↔ AWS Redshift
Azure Event Hubs            ↔ AWS Kinesis Data Streams
Azure Service Bus           ↔ AWS SQS + SNS
Azure IoT Hub               ↔ AWS IoT Core
Azure Functions             ↔ AWS Lambda
Azure Container Apps        ↔ AWS App Runner / ECS Fargate
Azure AI Search             ↔ AWS OpenSearch
Azure OpenAI                ↔ AWS Bedrock
Azure Key Vault             ↔ AWS Secrets Manager + KMS
Azure AD / Entra ID         ↔ AWS IAM Identity Center
Azure Monitor               ↔ AWS CloudWatch
Azure Policy                ↔ AWS Config + SCP
```

---

## 10. COST OPTIMIZATION (FINOPS)

```
Compute:
  Reserved Instances  : 1yr = ~40% savings, 3yr = ~60%
  Spot VMs            : 90% savings, use for batch/non-critical
  AKS: right-size nodes, use Spot node pools for non-prod
  Dev/Test subscription pricing for non-prod environments

Storage:
  Lifecycle policies   : auto-move Blob to Cool/Archive
  Compression          : Parquet over CSV (5–10x reduction)
  Delta Lake OPTIMIZE  : compact small files (reduces read cost)
  Delete unused disks  : check for orphaned managed disks

Networking:
  Private Endpoints reduce egress vs public access
  CDN reduces repeated egress cost for static content
  Same-region traffic = free; cross-region = $$

Database:
  CosmosDB: autoscale > manual provisioned for variable load
  Synapse: pause dedicated pools when not in use
  Redis: right-size cache tier, set TTL aggressively

FinOps practices:
  Tag everything: env, team, project, cost-center
  Azure Cost Management + budgets + alerts
  Workload right-sizing (Azure Advisor recommendations)
  Chargeback model for shared services
```

---

## 11. TERRAFORM PATTERNS

```hcl
# Standard module structure
modules/
  networking/        # VNet, subnets, NSG, Private DNS
  aks/               # AKS cluster, node pools, addons
  data/              # Storage, CosmosDB, SQL
  monitoring/        # Log Analytics, App Insights, alerts
  security/          # Key Vault, Managed Identities, policies

# OIDC GitHub Actions pattern (no secrets)
resource "azurerm_federated_identity_credential" "github" {
  name                = "github-actions"
  resource_group_name = azurerm_resource_group.main.name
  parent_id           = azurerm_user_assigned_identity.deployer.id
  audience            = ["api://AzureADTokenExchange"]
  issuer              = "https://token.actions.githubusercontent.com"
  subject             = "repo:ORG/REPO:environment:production"
}

# Key patterns:
# - Remote state: Azure Blob with state locking
# - Workspaces per environment (dev/staging/prod)
# - Variable files per env, secrets from Key Vault data source
# - Modules versioned and pinned
```

---

## 12. MIND MAP EXPANSION TRIGGERS

Use these prompts to expand this file in future Claude sessions:

```
"Using CLAUDE.md context, add a deep-dive section on [topic]"
"Add troubleshooting playbook for [service]"
"Add interview Q&A bank for [use case]"
"Add Terraform module pattern for [resource]"
"Add AWS vs Azure parity for [service category]"
"Add cost optimization patterns for [workload type]"
"Add compliance checklist for [regulation]"
"Add architecture pattern for [industry] at [small/medium/large] scale"
"Generate architecture diagram description for [use case]"
"Generate presentation outline for [audience] on [topic]"
```

---

## 13. QUICK REFERENCE CHEAT SHEET

```
Latency targets (production):
  API response      : <200ms p99
  DB query (OLTP)   : <10ms (Redis), <50ms (Cosmos), <200ms (SQL)
  Streaming ingest  : <1s end-to-end (Event Hubs)
  Batch processing  : hours acceptable (Synapse, Databricks)

SLA targets:
  Banking API       : 99.99% = 52 min/yr downtime
  Internal tools    : 99.9% = 8.7 hr/yr
  Batch pipelines   : 99.5% acceptable

Data retention (Canada):
  Financial records : 7 years (FINTRAC)
  Personal data     : as long as necessary (PIPEDA)
  Audit logs        : min 1 year hot, 7 years cold

Azure regions (Canada):
  Canada Central  : Toronto (primary)
  Canada East     : Quebec City (DR pair)
  Data residency  : keep both in Canada for OSFI compliance

Ports to remember:
  HTTPS    : 443
  SQL      : 1433
  Redis    : 6380 (TLS)
  Kafka    : 9093 (Event Hubs TLS)
  AKS API  : 443
  SSH      : 22 (use Bastion, never open to internet)
```

---

## 14. DOCUMENTATION & .MD FILE CREATION STANDARD

> This pattern is **mandatory** for all .md files created in this project —  
> interview prep, how-to guides, scenario playbooks, architecture docs.  
> Confirmed and validated by user on AzureNetworkingHowToTopics (May 2026).

### 14.1 Per-Question Structure (always in this order)

```
1. Question number     : ## Q1. (never skip numbering)
2. Scenario framing    : "Scenario: [concrete real-world situation]" — never abstract
3. ASCII diagram       : topology / before-after / flow — whenever spatial relationships exist
4. Bullet-point answer : never long prose; each bullet = one complete thought
5. Bold key services   : **Azure Firewall**, **Private Endpoint**, **KEDA** etc.
6. Code block          : runnable CLI / PowerShell / KQL / YAML / HCL — not pseudocode
7. Output annotation   : # Expected: ... and # Wrong: ... comments inside code blocks
8. ⚠️ Gotcha           : the one thing that breaks this in production (one sentence)
9. 💡 Deep dive hint   : one-liner naming the follow-up thread topic
```

### 14.2 ASCII Diagram Rules

```
Use these characters: ─ │ ┌ ┐ └ ┘ ├ ┤ ┬ ┴ ┼ ▼ ► ◄ ←
Always show BEFORE and AFTER when the question involves a change
Label IPs, service names, and CIDRs directly on the diagram — no legend needed
Flow diagrams use arrows to show direction of traffic
```

### 14.3 File & Folder Organisation

```
- README.md in every folder with navigation table: # | File | What's inside
- Files numbered: 01-topic.md, 02-topic.md (not by level — levels live inside each file)
- Level indicator at top of file: 🟢 Beginner → 🟡 Intermediate → 🔴 Advanced
- Section dividers (---) between every question
- Opening italicised quote capturing the topic's essence
```

### 14.4 Tables

```
- Use tables for: service comparisons, tier comparisons, rule lists, decision matrices
- Always include a "Use case" or "When to use" column
- Column headers ≤ 3 words
```

### 14.5 Code Blocks

```
- Language tag always present: bash, powershell, kusto, yaml, hcl, json, python
- Inline comments explain WHY, not WHAT
- Use \ line continuation for long commands
- Show expected output as a comment when the output is diagnostic
```

### 14.6 Tone

```
- Wikipedia-meets-practitioner: factual, confident, no fluff
- Short sentences in bullets — each bullet is one complete thought
- Lead with the insight, not the definition
- Questions are scenario-based and numbered — not abstract definitions
```

### 14.7 What NOT to do

```
- ❌ No long prose paragraphs as the main answer body
- ❌ No question without a number
- ❌ No answer without at least one code block (for technical topics)
- ❌ No diagram-less answer when the topic involves topology or flow
- ❌ No abstract questions ("What is X?") without a scenario framing it
- ❌ No trailing summary after delivering work — user reads the output directly
```

---

*Last updated: May 2026 | Next: Add GenAI agent patterns, ADF CDC deep-dive, AWS EKS vs AKS comparison*
