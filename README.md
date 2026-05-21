# Azure Architecture Reference Library

> Principal Architect reference for Cloud, AI, Data Migration, RAG Pipelines,  
> and Agentic AI — with production-grade patterns for the Canadian enterprise context.

**Author:** Abhishek Sinha  
**Stack:** Azure-first · OSS-aware · Cloud-agnostic reasoning  
**Regulatory lens:** OSFI B-10 · PIPEDA · FINTRAC

---

## Documents

### 1. [Unstructured Data Migration + RAG + Agentic AI](./UnstructuredData-Migration-RAG-AgenticAI.md)
End-to-end architecture for migrating PB-scale video and audio from on-premises to Azure,  
then building a production RAG pipeline and Agentic AI system on top of that corpus.

| Section | What's covered |
|---------|---------------|
| Migration phases 1–5 | Assessment → AzCopy/Data Box → validation → delta sync → cutover |
| Azure tech stack | Every service with why-this-not-that rationale |
| RAG definition & pipeline | Indexing phase, query phase, hybrid BM25+RRF, timestamp-aware chunking |
| Video & audio RAG | Video Indexer, AI Speech batch, AI Search schema with semantic config |
| Agentic AI definition | ReAct, multi-agent orchestration, human-in-the-loop |
| Semantic Kernel code | C# agent with tool plugins, Python enrichment pipeline, AKS worker |
| AI Governance | OSFI B-10, PIPEDA, FINTRAC controls, PII redaction, RAGAS eval, runtime safety |
| Security | Zero Trust, Terraform Key Vault + Storage with CMK, Workload Identity |
| Cost model | Small / medium / large tiers + 6 optimization strategies |
| Interview Q&A | 3 deep-dive questions with full architect-level structured answers |

---

### 2. [Tech Stack Comparison: Azure Native vs OSS](./TechStack-Comparison-DataMigration-RAG-AgenticAI.md)
Decision framework and side-by-side comparison of Azure Native and Open Source stacks  
across every layer — for data migration, RAG, and Agentic AI at all enterprise scales.

| Section | What's covered |
|---------|---------------|
| Decision framework | 5-lens model: regulation · team skill · lock-in · cost · time-to-value |
| Data migration stacks | Small/medium/large tiers — Azure Native vs OSS vs Hybrid with cost estimates |
| RAG pipeline comparison | Azure Native (bank) · OSS/LangChain/Qdrant (startup) · Hybrid (retail) |
| RAG frameworks | LangChain vs LlamaIndex vs Haystack vs Semantic Kernel |
| Agentic AI frameworks | Semantic Kernel · AutoGen · LangGraph · CrewAI — code examples for all |
| Vector DB deep dive | AI Search vs Qdrant vs Weaviate vs pgvector vs Chroma vs Pinecone |
| LLM serving | Azure OpenAI vs vLLM vs Ollama vs HuggingFace TGI + AKS YAML |
| Pipeline orchestration | ADF vs Airflow vs Prefect vs dbt — when to use each |
| Cloud strategy postures | Azure-First · Hybrid · Cloud-Agnostic — decision criteria |
| Full blueprints | 3 use-case blueprints: bank · startup · global retailer |
| Decision cheat sheet | One-table quick reference for every architecture decision |
| 50+ external links | Azure docs · OSS docs · OSFI/PIPEDA/FINTRAC · research papers · courses |

---

### 3. [Interview Prep: 10 System Design Use Cases](./Interview-SystemDesign-10UseCases.md)
Mentor + interviewer format covering 10 Azure system design scenarios end-to-end.  
Each use case includes clarifying questions, architecture, key decisions with tradeoffs,  
deep-dive probes with full technical answers, and follow-up questions.

| Use Case | Core Probe Areas |
|----------|-----------------|
| Real-time fraud detection (bank) | Event Hubs partitioning, ML scoring, OSFI B-10, champion/challenger |
| Multi-tenant SaaS on AKS | NetworkPolicy enforcement, Workload Identity, KEDA per-tenant scaling |
| 500TB video migration zero-downtime | Checksum validation (SHA-256), file-in-flight problem, CDN warm-up |
| RAG compliance chatbot | Hallucination prevention (4 layers), chunking strategy, multi-doc queries |
| Payment API 99.99% uptime | Idempotency (3 layers), Service Bus Geo-DR, circuit breaker, chaos engineering |
| IoT factory 1M sensors | IoT Hub vs Event Hubs decision, Edge aggregation, ADX vs InfluxDB vs TimescaleDB |
| Data lakehouse migration | CDC with Debezium (how it actually works), Delta Lake internals, cutover playbook |
| CI/CD for 50-team org | OIDC auth (no secrets), GitOps pull-based vs push-based, secrets without Git |
| Black Friday 10x spike | KEDA pre-scale + cron, cache stampede / penetration / breakdown failure modes |
| Hub-spoke + Zero Trust | Private Endpoint DNS gotcha, NSG vs Firewall decision, ExpressRoute BGP |

---

### 4. [Interview Question Bank](./interview-questions/README.md)
Comprehensive beginner → intermediate → advanced Q&A across all Azure domains.  
Includes how-to guides for Logic Apps, ADF checksum pipelines, and networking deep dives.

| File | Topics |
|------|--------|
| [01-fundamentals.md](./interview-questions/01-fundamentals.md) | Core Azure, IAM, storage, networking basics |
| [02-migration.md](./interview-questions/02-migration.md) | 6Rs, Data Box, CDC, zero-downtime cutover |
| [03-databases.md](./interview-questions/03-databases.md) | SQL, CosmosDB, Redis, partitioning, consistency |
| [04-scalability.md](./interview-questions/04-scalability.md) | AKS KEDA, autoscale, caching, traffic spikes |
| [05-disaster-recovery.md](./interview-questions/05-disaster-recovery.md) | RTO/RPO, active-active, failover, backup |
| [06-security-zero-trust.md](./interview-questions/06-security-zero-trust.md) | Zero Trust, Private Endpoints, OSFI/PIPEDA |
| [07-apim.md](./interview-questions/07-apim.md) | APIM policies, versioning, OAuth, circuit breaker |
| [08-storage-adls-synapse.md](./interview-questions/08-storage-adls-synapse.md) | Blob tiers, ADLS Gen2, Synapse, lakehouse |
| [09-data-factory-checksums.md](./interview-questions/09-data-factory-checksums.md) | ADF pipelines, checksums, incremental load, CDC |
| [10-networking.md](./interview-questions/10-networking.md) | Hub-Spoke, NSG, ExpressRoute, BGP, DNS |
| [11-howto-logic-apps.md](./interview-questions/11-howto-logic-apps.md) | How-to: large files, retries, approvals, scheduling |

---

### 5. [How-To Knowledge Base](./howto/README.md)
Short, practical, hands-on Q&A in an engaging Wikipedia-meets-practitioner style.  
Each answer is 3–6 sentences — exactly what you'd say when an interviewer asks *"How did you actually do that?"*

| File | Topic |
|------|-------|
| [01-governance.md](./howto/01-governance.md) | Management groups, policy, tagging, cost allocation |
| [02-landing-zones.md](./howto/02-landing-zones.md) | CAF landing zones, platform subscriptions, vending |
| [03-networking.md](./howto/03-networking.md) | Hub-Spoke, NSG, Private Endpoints, Front Door |
| [04-authentication-authorization.md](./howto/04-authentication-authorization.md) | Managed Identity, RBAC, service principals |
| [05-pim-conditional-access.md](./howto/05-pim-conditional-access.md) | PIM, Conditional Access, hybrid access |
| [06-security-policy.md](./howto/06-security-policy.md) | Defender, Azure Policy, secrets, encryption |
| [07-scalability.md](./howto/07-scalability.md) | Autoscale, KEDA, cache-aside, CosmosDB RU |
| [08-high-availability.md](./howto/08-high-availability.md) | AZs, Traffic Manager, failover groups, chaos testing |
| [09-migration.md](./howto/09-migration.md) | Azure Migrate, AzCopy, DMS, cutover, validation |
| [10-hybrid-connectivity.md](./howto/10-hybrid-connectivity.md) | VPN, ExpressRoute, Arc, Data Box, hybrid identity |

---

### 6. [CLAUDE.md — Architect Context](./CLAUDE.md)
Living reference file used as context for AI-assisted architecture work.  
Contains core patterns, use-case templates, troubleshooting playbooks, and interview Q&A  
for Cloud, AI, Data, Networking, AKS, and Migration across enterprise scales.

---

## Quick Navigation by Topic

| I want to... | Go to |
|---|---|
| Plan a video/audio migration from on-prem | [Migration phases](./UnstructuredData-Migration-RAG-AgenticAI.md#2-migration-architecture--phase-by-phase) |
| Understand what RAG is and how to build it | [RAG definition](./UnstructuredData-Migration-RAG-AgenticAI.md#4-what-is-rag-definition--architecture) |
| Build a RAG pipeline for video content | [Video RAG pipeline](./UnstructuredData-Migration-RAG-AgenticAI.md#5-rag-pipeline-for-video--audio) |
| Understand what Agentic AI is | [Agentic AI definition](./UnstructuredData-Migration-RAG-AgenticAI.md#6-what-is-agentic-ai-definition--architecture) |
| Choose a vector database | [Vector DB deep dive](./TechStack-Comparison-DataMigration-RAG-AgenticAI.md#6-vector-database-deep-dive) |
| Choose between LangChain / SK / LlamaIndex | [RAG framework comparison](./TechStack-Comparison-DataMigration-RAG-AgenticAI.md#45-rag-framework-comparison-langchain-vs-llamaindex-vs-haystack-vs-semantic-kernel) |
| Choose an agentic AI framework | [Agentic AI comparison](./TechStack-Comparison-DataMigration-RAG-AgenticAI.md#54-agentic-framework-comparison) |
| Design AI governance for a regulated project | [AI Governance framework](./UnstructuredData-Migration-RAG-AgenticAI.md#8-ai-governance-framework) |
| Compare Azure Native vs OSS at my scale | [Decision matrix](./TechStack-Comparison-DataMigration-RAG-AgenticAI.md#11-decision-matrix--one-page-cheat-sheet) |
| Find external docs, papers, and courses | [Further reading](./TechStack-Comparison-DataMigration-RAG-AgenticAI.md#12-further-reading--external-resources) |
| Prepare for an architecture interview | [10 Use Cases deep dive](./Interview-SystemDesign-10UseCases.md) |
| Understand how checksums work in migration | [Checksum validation](./Interview-SystemDesign-10UseCases.md#use-case-3-500tb-video-migration--zero-downtime) |
| Understand idempotency for payments | [Idempotency deep dive](./Interview-SystemDesign-10UseCases.md#use-case-5-payment-processing-api--9999-uptime) |
| Debug AKS networking / Private Endpoint DNS | [Zero Trust networking](./Interview-SystemDesign-10UseCases.md#use-case-10-hub-spoke-networking--zero-trust-for-enterprise) |
| Understand how CDC works | [CDC deep dive](./Interview-SystemDesign-10UseCases.md#use-case-7-enterprise-data-lakehouse-migration) |

---

## Repo Structure

```
AzureArchitecture/
├── README.md                                            ← this file
├── CLAUDE.md                                            ← architect context + patterns
├── UnstructuredData-Migration-RAG-AgenticAI.md          ← migration + RAG + Agentic AI design
├── TechStack-Comparison-DataMigration-RAG-AgenticAI.md  ← Azure Native vs OSS comparison
├── Interview-SystemDesign-10UseCases.md                 ← 10 use cases, deep-dive Q&A
├── howto/                                               ← hands-on how-to knowledge base
│   ├── README.md
│   ├── 01-governance.md ... 10-hybrid-connectivity.md
└── interview-questions/                                 ← interview question bank (B/I/A levels)
    ├── README.md                                        ← navigation index
    ├── 01-fundamentals.md                               ← beginner: core concepts
    ├── 02-migration.md                                  ← migration patterns
    ├── 03-databases.md                                  ← SQL, CosmosDB, Redis
    ├── 04-scalability.md                                ← autoscale, KEDA, caching
    ├── 05-disaster-recovery.md                          ← RTO/RPO, failover
    ├── 06-security-zero-trust.md                        ← Zero Trust, OSFI, PIPEDA
    ├── 07-apim.md                                       ← API Management
    ├── 08-storage-adls-synapse.md                       ← storage, lakehouse, Synapse
    ├── 09-data-factory-checksums.md                     ← ADF, checksums, CDC
    ├── 10-networking.md                                 ← Hub-Spoke, ExpressRoute, DNS
    └── 11-howto-logic-apps.md                           ← Logic Apps how-to guides
```

---

*Last updated: 2026-05-21*
