# Tech Stack Comparison: Azure Native vs Open Source
## Data Migration · RAG Pipelines · Agentic AI — Architect's Decision Guide

> **Mentor note:** This document is your decision framework, not a vendor pitch.
> Every technology listed here is production-proven. The question is never "which is better" —
> it's "which fits your constraint: regulatory, team skill, cost, timeline, and lock-in tolerance."
> Read the **Why** columns as carefully as the tool names. That's where architecture thinking lives.

---

## Table of Contents

1. [How to Read This Guide](#1-how-to-read-this-guide)
2. [The Core Decision Framework: Azure Native vs OSS](#2-the-core-decision-framework-azure-native-vs-oss)
3. [Data Migration Tech Stack Comparison](#3-data-migration-tech-stack-comparison)
4. [RAG Pipeline Tech Stack Comparison](#4-rag-pipeline-tech-stack-comparison)
5. [Agentic AI Tech Stack Comparison](#5-agentic-ai-tech-stack-comparison)
6. [Vector Database Deep Dive](#6-vector-database-deep-dive)
7. [LLM Serving: Hosted vs Self-Hosted](#7-llm-serving-hosted-vs-self-hosted)
8. [Data Pipeline Orchestration Comparison](#8-data-pipeline-orchestration-comparison)
9. [Cloud Strategy Patterns](#9-cloud-strategy-patterns)
10. [Full Architecture Blueprints by Use Case](#10-full-architecture-blueprints-by-use-case)
11. [Decision Matrix — One-Page Cheat Sheet](#11-decision-matrix--one-page-cheat-sheet)
12. [Further Reading & External Resources](#12-further-reading--external-resources)

---

## 1. How to Read This Guide

### Mentor Framework: Five Lenses for Every Decision

Before picking a tool, run it through these five questions. If you can't answer them, you're not ready to decide — go back and gather the constraint.

```
┌─────────────────────────────────────────────────────────────────────┐
│             THE FIVE CONSTRAINT LENSES                               │
│                                                                      │
│  1. REGULATION    Is this regulated? (OSFI, PIPEDA, FINTRAC, SOC2)  │
│                   Regulated → Azure Native wins (audit, residency)   │
│                                                                      │
│  2. TEAM SKILL    What does your team already know?                  │
│                   Python devs → prefer LangChain/FastAPI/Airflow     │
│                   .NET shops → Semantic Kernel + Azure PaaS          │
│                                                                      │
│  3. LOCK-IN       How portable does this need to be?                 │
│                   K8s + OSS → portable. Azure PaaS → fast but tied.  │
│                                                                      │
│  4. COST HORIZON  1-year budget vs 5-year TCO                        │
│                   OSS = low upfront, high ops. Azure = pay-as-you-go │
│                                                                      │
│  5. TIME TO VALUE How fast to production?                            │
│                   Azure Native → weeks. OSS → months (if greenfield) │
└─────────────────────────────────────────────────────────────────────┘
```

### Terminology Anchor

| Term | One-line Definition |
|------|---------------------|
| **Azure Native** | Microsoft-managed PaaS/SaaS services (APIM, AI Search, OpenAI, ADF) |
| **OSS** | Open-source software you deploy and operate (Airflow, Qdrant, LangChain, AutoGen) |
| **Hybrid** | Azure infrastructure (AKS, ADLS) running OSS workloads |
| **RAG** | Retrieval-Augmented Generation — ground LLM answers in retrieved documents |
| **Agentic AI** | LLM that plans, calls tools, observes results, and iterates toward a goal |
| **LLMOps** | DevOps practices applied to LLM lifecycle (versioning, eval, drift monitoring) |

---

## 2. The Core Decision Framework: Azure Native vs OSS

### 2.1 Spectrum of Choices

```
FULLY AZURE NATIVE ◄─────────────────────────────────► FULLY OSS ON AZURE
                                                        (or other cloud)

Azure OpenAI          Hybrid Zone           vLLM on AKS + Hugging Face
Azure AI Search       (most enterprises     Qdrant / Weaviate / pgvector
Azure Data Factory    land here)            Apache Airflow + dbt
Azure Video Indexer                         LangChain / LlamaIndex
Semantic Kernel                             AutoGen / LangGraph / CrewAI
Azure AI Agent Svc                          MLflow / Weights & Biases
```

### 2.2 Decision Tree

```
START: New AI/Data Platform Decision
            │
            ▼
  Are you in a regulated industry?
  (Banking, Insurance, Healthcare, Gov)
            │
     Yes ───┼─── No
            │         │
            ▼         ▼
  Is data residency   Is your team
  in Canada required? Python-native?
            │              │
     Yes    │         Yes ─┼─ No
            │              │      │
            ▼              ▼      ▼
  Azure Native        OSS-heavy  Either — 
  (OSFI/PIPEDA        on AKS     evaluate
  compliance built-in)(portable) by cost
            │
            ▼
  Do you need <4 week time-to-value?
            │
     Yes ───┼─── No
            │         │
            ▼         ▼
  Azure Native PaaS  OSS if team
  (fastest path)     has capacity
```

### 2.3 Honest Tradeoffs Table

| Dimension | Azure Native | OSS on Azure (AKS) | Winner at Scale |
|-----------|-------------|-------------------|-----------------|
| **Time to first demo** | Days | Weeks | Azure Native |
| **Cost at low scale** | Pay-per-use (can be high) | Fixed AKS cluster | OSS |
| **Cost at high scale** | Expensive PTU/S3 tiers | Commoditized compute | OSS (often 40–60% cheaper) |
| **Compliance coverage** | Built-in (WORM, CMK, Private EP) | Manual setup required | Azure Native |
| **Portability** | Low (Azure lock-in) | High (runs anywhere K8s runs) | OSS |
| **Operational burden** | Low | High (you own upgrades, HA) | Azure Native |
| **Customization ceiling** | Hit limits faster | Unlimited | OSS |
| **Vendor support SLA** | Yes (Microsoft) | Community only | Azure Native |
| **Latest models/features** | OpenAI roadmap gated | Full OSS model choice | OSS |

---

## 3. Data Migration Tech Stack Comparison

### 3.1 Small Enterprise (<10 TB, <5K assets)

**Profile:** Regional insurance company, 3-person data team, first cloud migration.

#### Azure Native Stack

```
On-Prem NAS
    │
    ▼ AzCopy v10 (free, parallel, resumable)
Azure Blob Storage (Hot)
    │
    ▼ Azure Data Factory (drag-drop pipelines, no code)
Azure Data Lake Storage Gen2
    │
    ▼ Azure Synapse Analytics (serverless SQL — pay per query)
Power BI (reporting)
```

| Tool | Purpose | Monthly Cost |
|------|---------|-------------|
| AzCopy | Transfer | Free |
| Azure Blob (LRS) | Storage | ~$20/TB |
| Azure Data Factory | ETL/ELT | ~$200–500 |
| Synapse Serverless | Query | ~$5/TB scanned |
| Power BI Pro | Reporting | $10/user |

**Why Azure Native here:** 3-person team. No Kubernetes expertise. Sub-10TB fits within managed service limits cheaply. ADF has a visual designer — faster onboarding.

---

#### OSS Alternative Stack (same use case)

```
On-Prem NAS
    │
    ▼ rclone (open source, 70+ cloud backends)
MinIO → Azure Blob Storage
    │
    ▼ Apache Airflow (Docker Compose on a VM — not K8s at this scale)
Delta Lake (on ADLS Gen2, managed by Databricks CE or OSS Spark)
    │
    ▼ dbt Core (free) for transformations
Apache Superset (free BI/dashboarding)
```

**Why OSS here:** If the team is Python/data-engineering native. rclone is more flexible than AzCopy for heterogeneous sources. dbt gives SQL-first transformations with built-in lineage. Cost: ~$300–600/month total (1 VM + storage).

**Mentor verdict for small:** Azure Native unless the team already runs Airflow. OSS complexity is not justified at <10TB.

---

### 3.2 Medium Enterprise (10–500 TB, 50K–500K assets)

**Profile:** National logistics company, 10-person data team, migrating media + structured data.

#### Azure Native Stack

```
On-Prem (NAS + SQL Server + Oracle)
    │
    ├── ADF Self-Hosted Integration Runtime → Azure Blob (binary files)
    ├── Azure Database Migration Service → Azure SQL / Cosmos DB
    └── ExpressRoute 1 Gbps (private circuit)
    │
    ▼
ADLS Gen2 (Raw Zone → Curated Zone → Enriched Zone)
    │
    ▼ Azure Data Factory (orchestration, 200+ connectors)
Azure Databricks (PySpark transformations, Delta Lake)
    │
    ▼
Azure Synapse Analytics (Dedicated Pool — SQL DW)
Azure AI Search (unstructured content search)
    │
    ▼
Power BI Premium + Azure Analysis Services
```

---

#### OSS Stack (same scale, different constraint: portability required)

```
On-Prem
    │
    ├── Apache Kafka (Confluent OSS) → streaming delta sync
    ├── Debezium (CDC connector) → captures DB changes in real time
    └── rclone / custom scripts → file transfer
    │
    ▼
Apache Airflow (on AKS, Helm chart) — orchestration
    │
    ▼
ADLS Gen2 (or S3-compatible MinIO if multi-cloud needed)
    │
    ▼ dbt Core / dbt Cloud
Delta Lake (OSS, managed via Apache Spark on AKS or Databricks)
    │
    ▼
Trino (formerly PrestoSQL) — federated SQL over multiple data sources
    OR
Apache Hive Metastore + Spark SQL
    │
    ▼
Apache Superset / Metabase (OSS dashboarding)
```

#### Side-by-Side Comparison: Medium Scale

| Capability | Azure Native | OSS on AKS |
|------------|-------------|------------|
| CDC (Change Data Capture) | ADF CDC connector | Debezium (Kafka) — more mature |
| Streaming ingestion | Event Hubs (managed Kafka) | Apache Kafka OSS (full control) |
| ETL orchestration | Azure Data Factory | Apache Airflow (richer plugin ecosystem) |
| Transformation | Databricks (Azure-managed) | dbt + Spark OSS (portable) |
| Data catalog | Microsoft Purview | Apache Atlas + Amundsen |
| Data quality | Azure DQ rules in ADF | Great Expectations (OSS) |
| Lineage | Purview (visual) | OpenLineage + Marquez |
| Cost (est.) | $5,000–8,000/month | $3,000–5,000/month (ops overhead +$1K) |

**Mentor verdict for medium:** Hybrid wins. Azure infrastructure (ADLS, AKS, ExpressRoute) + OSS tooling on top (Airflow, Debezium, dbt). Best cost/portability balance.

---

### 3.3 Large Enterprise (500 TB – PB scale)

**Profile:** Canadian Tier-1 bank, 50-person platform team, regulated, petabyte-scale media + structured.

#### Azure Native Stack

```
On-Prem Tape / SAN
    │
    ├── Azure Data Box Heavy (1 PB capacity) × multiple
    └── ExpressRoute 10 Gbps × 2 (redundant circuits)
    │
    ▼
Azure Blob (Landing) → Event Grid → Service Bus
    │
    ▼ AKS Workers (KEDA-scaled)
ADLS Gen2 (Delta Lake format — Raw / Curated / Enriched / Serving)
    │
    ▼
Azure Databricks (Unity Catalog — governance across all data)
    │
    ├── Azure Synapse Analytics (SQL DW for structured analytics)
    ├── Azure AI Search (unstructured/vector search)
    └── Azure Purview (catalog + lineage + classification)
    │
    ▼
Azure Analysis Services + Power BI Premium (reporting)
Azure OpenAI (AI layer on top of curated data)
```

---

#### OSS Stack (large scale, multi-cloud or portability mandate)

```
On-Prem
    │
    ├── Apache Kafka (Confluent Platform — enterprise OSS)
    │   Debezium for CDC on Oracle/SQL Server/PostgreSQL
    └── Data Box / large-scale rsync for binary assets
    │
    ▼
Apache Kafka → Apache Flink (stream processing — more powerful than Spark Streaming)
    │
    ▼
ADLS Gen2 / S3 (cloud-agnostic via Apache Iceberg or Delta Lake)
    │
    ▼
Apache Airflow (KubernetesExecutor on AKS) — batch orchestration
dbt Core — SQL transformations with lineage
Apache Spark (on AKS via Spark Operator) — heavy compute
    │
    ▼
Trino (federated query) + Apache Hive Metastore
    OR
Apache Iceberg + Project Nessie (catalog with Git-like branching)
    │
    ▼
Apache Atlas (data catalog) + OpenLineage (lineage)
Great Expectations (data quality) + Monte Carlo (observability)
    │
    ▼
Superset / Metabase / Grafana (dashboards)
Qdrant / Weaviate (vector search for AI)
```

#### Large-Scale Comparison

| Dimension | Azure Native (Databricks Unity Catalog) | OSS (Iceberg + Nessie + Trino) |
|-----------|----------------------------------------|-------------------------------|
| Data catalog governance | Purview + Unity Catalog (excellent) | Apache Atlas + Amundsen (mature, complex) |
| Multi-cloud table format | Delta Lake (Azure-optimized) | Apache Iceberg (vendor-neutral) |
| Streaming | Event Hubs / Databricks Streaming | Apache Kafka + Flink (more powerful) |
| CDC | ADF CDC (limited source support) | Debezium (40+ connectors) |
| Federated query | Synapse (Polybase) | Trino (best-in-class federated SQL) |
| Cost at PB scale | Very high (Databricks DBUs) | Lower, but high ops cost |
| Compliance (OSFI) | Native (Purview, CMK, Private EP) | Manual — needs custom tooling |
| Time to production | 3–4 months | 6–12 months |

**Mentor verdict for large regulated:** Azure Native for governance and compliance layer (Purview, Databricks Unity Catalog, Key Vault, Private Endpoints). OSS for compute-intensive streaming (Kafka, Flink) where Azure Event Hubs limits are hit.

---

## 4. RAG Pipeline Tech Stack Comparison

### 4.1 What to Compare

A RAG pipeline has six discrete stages. Each stage has Azure Native and OSS options:

```
Stage 1: Document Ingestion
Stage 2: Parsing / Extraction
Stage 3: Chunking
Stage 4: Embedding Generation
Stage 5: Vector Index / Storage
Stage 6: Query & Generation
```

---

### 4.2 Azure Native RAG Stack

**Use case:** Compliance chatbot for a Canadian bank — regulated, audit trail required.

```
┌─────────────────────────────────────────────────────────────────┐
│                  AZURE NATIVE RAG STACK                          │
├──────────────────┬──────────────────────────────────────────────┤
│ Stage            │ Service                                       │
├──────────────────┼──────────────────────────────────────────────┤
│ Ingestion        │ Azure Blob (Event Grid trigger) + Service Bus│
│ Parsing/Extract  │ Azure Document Intelligence (Form Recognizer)│
│                  │ Azure Video Indexer (for video/audio)         │
│                  │ Azure AI Speech (batch audio transcription)   │
│ Chunking         │ Custom Python in AKS (512 tokens, overlap 50)│
│ Embedding        │ Azure OpenAI text-embedding-3-large (3072d)  │
│ Vector Store     │ Azure AI Search (HNSW + semantic ranker)     │
│ Metadata Store   │ CosmosDB (NoSQL — fast asset lookup)         │
│ Query/Retrieval  │ AI Search (hybrid BM25 + vector RRF)         │
│ Generation       │ Azure OpenAI GPT-4o (PTU for production)     │
│ Orchestration    │ Semantic Kernel (C# or Python)               │
│ Caching          │ Azure Cache for Redis (semantic cache)       │
│ Monitoring       │ Azure Monitor + Log Analytics (WORM)         │
│ Content Safety   │ Azure Content Safety API                      │
└──────────────────┴──────────────────────────────────────────────┘
```

**Estimated monthly cost (medium scale, 10M chunks):**
- AI Search S2 (2 replicas): $1,000
- Azure OpenAI GPT-4o (consumption): $5,000–15,000
- Azure OpenAI embeddings: $500
- CosmosDB: $400
- Redis (C3): $700
- AKS (4 nodes): $800
- **Total: ~$8,000–18,000/month**

---

### 4.3 OSS RAG Stack

**Use case:** Internal knowledge base for a 200-person tech startup — cost-sensitive, Python team.

```
┌─────────────────────────────────────────────────────────────────┐
│                     OSS RAG STACK                                │
├──────────────────┬──────────────────────────────────────────────┤
│ Stage            │ Tool                                          │
├──────────────────┼──────────────────────────────────────────────┤
│ Ingestion        │ Apache Kafka (or Celery task queue)           │
│ Parsing/Extract  │ Unstructured.io (OSS PDF/HTML/video parser)  │
│                  │ Whisper (OpenAI OSS audio transcription)      │
│                  │ Docling (IBM OSS — structured doc extraction) │
│ Chunking         │ LangChain RecursiveTextSplitter               │
│                  │ OR LlamaIndex SentenceSplitter                │
│ Embedding        │ sentence-transformers (all-mpnet-base-v2)     │
│                  │ OR nomic-embed-text (768d, runs locally)       │
│                  │ OR Hugging Face Inference API (hosted)         │
│ Vector Store     │ Qdrant (self-hosted on AKS) — best OSS option│
│                  │ OR Weaviate, Chroma, pgvector (PostgreSQL ext)│
│ Metadata Store   │ PostgreSQL (with pgvector for hybrid)         │
│ Query/Retrieval  │ LangChain Retriever / LlamaIndex Query Engine │
│ Generation       │ Ollama (local: Llama 3.3, Mistral, Phi-4)    │
│                  │ OR vLLM (GPU-accelerated inference on AKS)    │
│                  │ OR Hugging Face TGI (Text Generation Inference)│
│ Orchestration    │ LangChain / LlamaIndex                        │
│                  │ OR Haystack (deepset.ai)                       │
│ Caching          │ Redis (OSS) + GPTCache                        │
│ Monitoring       │ MLflow + Langfuse (LLM observability)         │
│ Eval             │ RAGAS (OSS RAG evaluation)                    │
└──────────────────┴──────────────────────────────────────────────┘
```

**Estimated monthly cost (AKS, medium scale):**
- AKS cluster (6 nodes, GPU node for vLLM): $2,000
- Qdrant (self-managed): $0 (just AKS cost)
- PostgreSQL (Azure Flexible Server): $300
- Redis OSS (AKS): $0
- Ollama/vLLM (GPU node x1): $500–800
- Storage (ADLS): $200
- **Total: ~$3,000–4,000/month (60% cheaper, but you own operations)**

---

### 4.4 Hybrid RAG Stack

**Use case:** Global retail company — needs both cost efficiency and enterprise governance, multi-region.

```
┌─────────────────────────────────────────────────────────────────┐
│                    HYBRID RAG STACK                              │
│          Azure Infrastructure + OSS Tooling                      │
├──────────────────┬──────────────────────────────────────────────┤
│ Stage            │ Choice                            │ Why       │
├──────────────────┼───────────────────────────────────┼──────────┤
│ Ingestion        │ Azure Event Grid + Service Bus    │ Managed   │
│ Parsing          │ Unstructured.io on AKS            │ OSS flex  │
│                  │ + Azure Doc Intelligence (PDFs)   │ Best PDF  │
│ Chunking         │ LangChain splitters on AKS        │ Flexible  │
│ Embedding        │ Azure OpenAI ada-002 OR           │           │
│                  │ nomic-embed on AKS (cost choice)  │ Cost gate │
│ Vector Store     │ Qdrant on AKS (NOT Azure Search)  │ Save $$$  │
│ Metadata         │ CosmosDB (Azure managed)          │ SLA       │
│ Retrieval        │ LangChain + Qdrant retriever      │ OSS       │
│ Generation       │ Azure OpenAI GPT-4o               │ Quality   │
│ Orchestration    │ LangChain (Python-native)         │ OSS flex  │
│ Monitoring       │ Langfuse + Azure Monitor          │ Both      │
│ Safety           │ Azure Content Safety API          │ Managed   │
└──────────────────┴───────────────────────────────────┴──────────┘
```

---

### 4.5 RAG Framework Comparison: LangChain vs LlamaIndex vs Haystack vs Semantic Kernel

| Dimension | LangChain | LlamaIndex | Haystack | Semantic Kernel |
|-----------|-----------|------------|---------|-----------------|
| **Language** | Python (+ JS) | Python (+ TS) | Python | Python + C# |
| **Primary focus** | General LLM chains | Document RAG & indexing | Search/RAG pipelines | Microsoft AI agents |
| **Learning curve** | Medium | Low | Low | Medium |
| **Agentic support** | LangGraph (excellent) | Agent workflows | Limited | Excellent (native) |
| **Azure integration** | Good (community plugins) | Good | Fair | Native (built by MS) |
| **Community size** | Largest | Large | Medium | Growing fast |
| **Production maturity** | High | High | High | Medium-High |
| **Best for** | Complex chains, agents | Document Q&A, RAG | Search-first RAG | Azure-native agents |
| **License** | MIT | MIT | Apache 2.0 | MIT |

**When to choose what:**
- **LangChain:** Team is Python-native, needs maximum flexibility and OSS ecosystem depth, building complex multi-step agent workflows
- **LlamaIndex:** Document-heavy RAG (PDFs, Word docs, transcripts), needs advanced indexing strategies out of the box
- **Haystack:** When search is the core product (not just a component), strong pipeline abstraction
- **Semantic Kernel:** .NET team, Azure-native deployment, enterprise Microsoft stack (Teams, M365 integration)

---

## 5. Agentic AI Tech Stack Comparison

### 5.1 What Makes an "Agentic" System

```
NON-AGENTIC (simple RAG):
  Input → Retrieve → Generate → Output   (1 pass, no tools, no memory, no iteration)

AGENTIC:
  Input → PLAN → [Tool → Observe → Re-plan] × N → Synthesize → Output
         (multi-step, uses external tools, maintains state, adapts)

MULTI-AGENT:
  Orchestrator decomposes goal → delegates to specialist agents
  → each agent has its own tool set → results aggregated
```

### 5.2 Azure Native Agentic Stack

**Use case:** Compliance analyst assistant at a Canadian bank.

```
┌─────────────────────────────────────────────────────────────────┐
│           AZURE NATIVE AGENTIC AI STACK                          │
├──────────────────────────────────────────────────────────────────┤
│ Agent Framework   │ Azure AI Agent Service (preview 2025)        │
│                   │ + Semantic Kernel (SK) for plugin/tool mgmt  │
│ LLM Backbone      │ Azure OpenAI GPT-4o (PTU)                    │
│ Tool Execution    │ SK Plugins (custom C#/Python functions)       │
│ Memory (short)    │ CosmosDB (session history, tool call log)     │
│ Memory (long)     │ Azure AI Search (semantic memory store)       │
│ Tool: Search      │ Azure AI Search (knowledge base retrieval)    │
│ Tool: Code Exec   │ Azure Container Instances (sandboxed)         │
│ Tool: Web         │ Bing Search API (grounded web results)        │
│ Human-in-loop     │ Azure Logic Apps (approval workflow)          │
│ Serving           │ AKS + APIM                                    │
│ Observability     │ Azure Monitor + App Insights                  │
│ Safety            │ Azure Content Safety (input + output)         │
│ Audit             │ Log Analytics (WORM, FINTRAC-compliant)       │
└──────────────────────────────────────────────────────────────────┘
```

```python
# Semantic Kernel: Multi-agent pattern
from semantic_kernel.agents import AgentGroupChat, ChatCompletionAgent
from semantic_kernel.agents.strategies import TerminationStrategy

# Specialist agents
search_agent = ChatCompletionAgent(
    name="VideoSearchAgent",
    instructions="Search the video knowledge base. Always return timestamps and citations.",
    kernel=build_kernel_with_search_plugin()
)

compliance_agent = ChatCompletionAgent(
    name="ComplianceAgent",
    instructions="Evaluate findings against OSFI B-10 and PIPEDA. Flag violations. Be specific.",
    kernel=build_kernel_with_compliance_rules()
)

summarizer_agent = ChatCompletionAgent(
    name="SummaryAgent",
    instructions="Synthesize findings into a structured report. Include risk ratings.",
    kernel=build_kernel_with_no_tools()
)

# Orchestrated group chat — agents collaborate in sequence
group_chat = AgentGroupChat(
    agents=[search_agent, compliance_agent, summarizer_agent],
    termination_strategy=TerminationStrategy(
        agents=[summarizer_agent],
        maximum_iterations=10
    )
)
```

---

### 5.3 OSS Agentic Stack

**Use case:** Research assistant for a pharmaceutical company — Python team, no Azure lock-in mandate.

#### Option A: AutoGen (Microsoft OSS, most mature for multi-agent)

```python
# AutoGen: Multi-agent research assistant
import autogen

config_list = [{
    "model": "gpt-4o",
    "api_key": os.environ["OPENAI_API_KEY"]
    # OR use local: "base_url": "http://ollama:11434/v1"
}]

# Define agents
user_proxy = autogen.UserProxyAgent(
    name="User",
    human_input_mode="NEVER",  # fully automated for batch jobs
    max_consecutive_auto_reply=10,
    code_execution_config={"work_dir": "workspace", "use_docker": True}
)

researcher = autogen.AssistantAgent(
    name="ResearchAgent",
    system_message="Search knowledge base and synthesize findings. Cite sources.",
    llm_config={"config_list": config_list}
)

critic = autogen.AssistantAgent(
    name="CriticAgent",
    system_message="Review the researcher's findings for accuracy and completeness. Challenge assumptions.",
    llm_config={"config_list": config_list}
)

# Register tools
@user_proxy.register_for_execution()
@researcher.register_for_llm(description="Search the internal knowledge base")
def search_knowledge_base(query: str, top_k: int = 5) -> str:
    return qdrant_client.search(collection_name="documents", query_text=query, limit=top_k)

# Run
user_proxy.initiate_chat(researcher, message="Analyze Q2 board meeting risk discussions")
```

---

#### Option B: LangGraph (best for complex stateful workflows)

```python
# LangGraph: Stateful agent with explicit workflow graph
from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated
import operator

class AgentState(TypedDict):
    messages: Annotated[list, operator.add]
    retrieved_docs: list
    compliance_flags: list
    final_report: str
    human_review_needed: bool

def retrieve_node(state: AgentState) -> AgentState:
    """Retrieve from Qdrant vector store"""
    query = state["messages"][-1].content
    docs = qdrant_retriever.get_relevant_documents(query)
    return {"retrieved_docs": docs}

def analyze_node(state: AgentState) -> AgentState:
    """LLM analysis with retrieved context"""
    response = llm.invoke(build_prompt(state["messages"], state["retrieved_docs"]))
    return {"messages": [response]}

def compliance_check_node(state: AgentState) -> AgentState:
    """Check for compliance flags"""
    flags = run_compliance_rules(state["messages"][-1].content)
    needs_review = any(f["severity"] == "HIGH" for f in flags)
    return {"compliance_flags": flags, "human_review_needed": needs_review}

def route_after_compliance(state: AgentState) -> str:
    """Conditional routing — human review if high severity"""
    return "human_review" if state["human_review_needed"] else "generate_report"

# Build the graph
graph = StateGraph(AgentState)
graph.add_node("retrieve", retrieve_node)
graph.add_node("analyze", analyze_node)
graph.add_node("compliance_check", compliance_check_node)
graph.add_node("generate_report", generate_report_node)
graph.add_node("human_review", human_review_node)

graph.set_entry_point("retrieve")
graph.add_edge("retrieve", "analyze")
graph.add_edge("analyze", "compliance_check")
graph.add_conditional_edges("compliance_check", route_after_compliance)
graph.add_edge("generate_report", END)
graph.add_edge("human_review", END)

agent = graph.compile()
```

---

#### Option C: CrewAI (easiest to get started, role-based)

```python
# CrewAI: Role-based multi-agent (easiest for teams new to agents)
from crewai import Agent, Task, Crew, Process

# Define specialist agents with roles
video_analyst = Agent(
    role="Video Content Analyst",
    goal="Extract key insights from video transcripts with timestamps",
    backstory="Expert at analyzing meeting recordings and identifying key decisions",
    tools=[video_search_tool, transcript_tool],
    llm=ChatOpenAI(model="gpt-4o")
)

risk_analyst = Agent(
    role="Risk & Compliance Analyst",
    goal="Identify regulatory risks in content (OSFI, PIPEDA)",
    backstory="Senior compliance professional with deep OSFI B-10 knowledge",
    tools=[compliance_rules_tool, regulatory_search_tool],
    llm=ChatOpenAI(model="gpt-4o")
)

report_writer = Agent(
    role="Executive Report Writer",
    goal="Synthesize findings into a clear, actionable executive report",
    backstory="Experienced at communicating complex risk findings to board level",
    tools=[],
    llm=ChatOpenAI(model="gpt-4o")
)

# Define tasks
crew = Crew(
    agents=[video_analyst, risk_analyst, report_writer],
    tasks=[search_task, analysis_task, report_task],
    process=Process.sequential,  # or Process.hierarchical for manager-agent
    verbose=True
)
```

---

### 5.4 Agentic Framework Comparison

| Dimension | Semantic Kernel | AutoGen | LangGraph | CrewAI |
|-----------|----------------|---------|-----------|--------|
| **Paradigm** | Plugin/skill model | Conversation-based multi-agent | Stateful graph workflows | Role-based crew |
| **Language** | Python + **C#** | Python | Python | Python |
| **Azure native** | Deep integration | Good | Good | Fair |
| **Complexity** | Medium | Medium | High | Low |
| **State management** | Manual | Implicit | Explicit graph | Implicit |
| **Human-in-loop** | Built-in | Built-in | Built-in | Limited |
| **Debugging** | App Insights | Custom | LangSmith | crewai logs |
| **Best for** | .NET teams, Azure shops | Research agents, complex orchestration | Complex stateful workflows, production agents | Rapid prototyping, role-based tasks |
| **Production maturity** | High (enterprise) | High | High | Medium |
| **License** | MIT | MIT (CC-BY) | MIT | MIT |

---

## 6. Vector Database Deep Dive

### 6.1 The Candidates

Six options evaluated across the same dimensions:

| Database | Type | Hosting | License |
|----------|------|---------|---------|
| **Azure AI Search** | Hybrid (vector + keyword) | Fully managed | Azure proprietary |
| **Qdrant** | Pure vector | Self-hosted or Qdrant Cloud | Apache 2.0 |
| **Weaviate** | Hybrid | Self-hosted or Weaviate Cloud | BSD 3-Clause |
| **pgvector** | Extension on PostgreSQL | Self-hosted or Azure PostgreSQL | PostgreSQL License |
| **Chroma** | Pure vector (dev-focused) | Self-hosted | Apache 2.0 |
| **Pinecone** | Pure vector | Fully managed (proprietary) | Proprietary |

---

### 6.2 Comparison Matrix

| Dimension | Azure AI Search | Qdrant | Weaviate | pgvector | Chroma | Pinecone |
|-----------|----------------|--------|----------|---------|--------|---------|
| **Hybrid BM25 + vector** | Native (RRF) | Manual BM25 needed | Built-in | Manual | No | No |
| **Semantic re-ranking** | Yes (L2 ranker) | No | No | No | No | No |
| **Filtering** | Rich (OData) | Excellent (payload filters) | GraphQL | SQL WHERE | Limited | Good |
| **Scale (vectors)** | 100M+ | 1B+ | 100M+ | <50M practical | <1M (dev only) | 1B+ |
| **Latency (p99)** | <100ms | <10ms | <50ms | <50ms | <20ms | <50ms |
| **Multi-tenancy** | Index-level | Collection-level | Tenant isolation | Schema-level | No | Namespace |
| **Observability** | Azure Monitor | Prometheus metrics | Prometheus | pg_stat | None | Dashboard |
| **Private network** | Private Endpoint | AKS + NSG | AKS + NSG | Private Endpoint | AKS | VPC (AWS only) |
| **WORM / Compliance** | Yes (Azure) | No native | No native | No native | No | No |
| **Cost (10M vectors)** | ~$1,000/month | ~$200/month | ~$300/month | ~$150/month | Free | ~$700/month |
| **Best for** | Regulated enterprise | High perf, OSS | Multi-modal, OSS | Existing Postgres | Dev/test | Simple cloud-native |

---

### 6.3 Code Comparison: Same Query, Different Databases

**Query: "Find compliance risk discussions about OSFI B-10"**

```python
# ─── Azure AI Search ────────────────────────────────────────────────
from azure.search.documents import SearchClient
from azure.search.documents.models import VectorizedQuery

results = search_client.search(
    search_text="OSFI B-10 compliance risk",  # BM25 keyword search
    vector_queries=[VectorizedQuery(
        vector=get_embedding("OSFI B-10 compliance risk"),
        fields="embedding", k_nearest_neighbors=50
    )],
    query_type="semantic",
    semantic_configuration_name="semantic-config",
    select=["text", "asset_title", "start_ms", "speakers"],
    top=5
)

# ─── Qdrant ─────────────────────────────────────────────────────────
from qdrant_client import QdrantClient
from qdrant_client.models import Filter, FieldCondition, MatchValue

qdrant = QdrantClient(host="qdrant-svc", port=6333)
results = qdrant.search(
    collection_name="video-chunks",
    query_vector=get_embedding("OSFI B-10 compliance risk"),
    query_filter=Filter(
        must=[FieldCondition(key="source_type", match=MatchValue(value="board-meeting"))]
    ),
    limit=5,
    with_payload=True
)

# ─── pgvector (PostgreSQL) ───────────────────────────────────────────
import psycopg2

conn = psycopg2.connect(PG_CONNECTION_STRING)
cur = conn.cursor()
embedding = get_embedding("OSFI B-10 compliance risk")

cur.execute("""
    SELECT text, asset_title, start_ms, speakers,
           1 - (embedding <=> %s::vector) AS similarity
    FROM video_chunks
    WHERE source_type = 'board-meeting'
      AND 1 - (embedding <=> %s::vector) > 0.75
    ORDER BY embedding <=> %s::vector
    LIMIT 5
""", (embedding, embedding, embedding))

results = cur.fetchall()
```

---

### 6.4 When to Choose Which

```
Choose Azure AI Search when:
  ✓ Regulated industry (banking, insurance, government)
  ✓ Need hybrid BM25 + vector in one query (avoids building your own fusion)
  ✓ Need semantic re-ranking (significant quality improvement)
  ✓ Team does not want to operate vector DB infrastructure
  ✓ Azure ecosystem already in use

Choose Qdrant when:
  ✓ High throughput, low latency requirements (>1K QPS)
  ✓ Want full control over infrastructure
  ✓ Multi-cloud or cloud-agnostic requirement
  ✓ Budget-sensitive at scale (10M+ vectors)
  ✓ Need advanced filtering on rich payload metadata

Choose pgvector when:
  ✓ Already on PostgreSQL (reduce operational complexity)
  ✓ Small-to-medium scale (<10M vectors)
  ✓ Need transactional consistency between vector and relational data
  ✓ Simple team — one database to operate

Choose Weaviate when:
  ✓ Multi-modal (text + image + video embeddings in same DB)
  ✓ Need GraphQL-based search interface
  ✓ Want built-in vectorizer modules (no separate embedding step)

Avoid Chroma in production:
  ✗ No persistence guarantees at scale
  ✗ No filtering on indexed fields
  ✗ Use only for prototyping / local development

Avoid Pinecone when:
  ✗ Data residency requirement (limited to AWS regions)
  ✗ Need hybrid BM25 + vector natively
  ✗ Budget-sensitive (expensive at scale)
```

---

## 7. LLM Serving: Hosted vs Self-Hosted

### 7.1 Options Landscape

```
FULLY HOSTED (API)             SELF-HOSTED (on AKS / VMs)
─────────────────              ──────────────────────────
Azure OpenAI (GPT-4o)          Ollama (local, CPU/GPU)
OpenAI API (direct)            vLLM (production GPU serving)
Anthropic Claude API           Hugging Face TGI (Text Generation Inference)
Google Vertex AI               LM Studio (desktop dev)
AWS Bedrock                    llama.cpp (minimal, CPU-only)
Mistral API                    Llamafile (single executable)
Groq (fastest inference)       Triton Inference Server (NVIDIA)
```

### 7.2 Comparison Table

| Dimension | Azure OpenAI | vLLM (AKS) | Ollama (local) | Hugging Face TGI |
|-----------|-------------|-----------|----------------|-----------------|
| **Setup time** | Minutes | Hours | Minutes | Hours |
| **Models available** | GPT-4o, GPT-4o-mini, o1 | Any HuggingFace model | 100+ OSS models | Any HuggingFace model |
| **Throughput (tokens/s)** | Varies (managed) | 2,000–10,000+ | 20–200 (CPU) | 500–2,000 |
| **Latency (TTFT)** | 200–500ms | 50–200ms | 500ms–5s | 100–300ms |
| **Cost (1M tokens in/out)** | ~$5–25 | ~$1–3 (GPU amortized) | ~$0 (own hardware) | ~$0–2 |
| **Data residency** | Canada regions available | Fully controlled | Fully controlled | Controlled |
| **Private network** | Private Endpoint | AKS-internal | Localhost | AKS-internal |
| **OSFI compliant** | Yes (with Private EP) | Yes (self-managed) | Dev only | Yes (self-managed) |
| **Operational burden** | None | High | None (dev) | Medium |
| **GPU required** | No (managed) | Yes (A100/H100) | No (CPU) | Yes (recommended) |

---

### 7.3 vLLM Production Setup on AKS

```yaml
# AKS deployment: vLLM serving Mistral-7B-Instruct
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-serving
  namespace: ai-platform
spec:
  replicas: 2
  selector:
    matchLabels:
      app: vllm
  template:
    spec:
      nodeSelector:
        accelerator: nvidia-a100  # GPU node pool
      containers:
        - name: vllm
          image: vllm/vllm-openai:latest
          command:
            - python
            - -m
            - vllm.entrypoints.openai.api_server
            - --model
            - mistralai/Mistral-7B-Instruct-v0.3
            - --tensor-parallel-size
            - "1"
            - --max-model-len
            - "32768"
            - --served-model-name
            - mistral-7b
          resources:
            requests:
              nvidia.com/gpu: "1"
              memory: "32Gi"
            limits:
              nvidia.com/gpu: "1"
          ports:
            - containerPort: 8000
          env:
            - name: HUGGING_FACE_HUB_TOKEN
              valueFrom:
                secretKeyRef:
                  name: hf-token
                  key: token
```

---

### 7.4 When to Use Self-Hosted LLM

```
Use self-hosted (vLLM/TGI) when:
  ✓ Data cannot leave your network (absolute data residency)
  ✓ At sustained >1M tokens/day: self-hosted GPU is cheaper than API
  ✓ Need fine-tuned model not available via API
  ✓ Latency-critical (sub-50ms TTFT required)
  ✓ Want to test latest OSS models (Llama 3.3, Mistral, Phi-4, Qwen2.5)

Stay on Azure OpenAI when:
  ✓ Regulated industry needing Microsoft compliance attestations
  ✓ Team has no GPU/MLOps expertise
  ✓ Scale is moderate (<1M tokens/day)
  ✓ Need latest GPT-4o/o1 capabilities (still ahead of OSS for complex reasoning)
  ✓ Time-to-value is critical (weeks, not months)
```

---

## 8. Data Pipeline Orchestration Comparison

### 8.1 The Candidates

| Tool | Type | Hosting | Best for |
|------|------|---------|---------|
| **Azure Data Factory** | Cloud-native ELT/ETL | Fully managed | Enterprise, no-code/low-code |
| **Apache Airflow** | Workflow orchestrator | Self-hosted (AKS) | Python-native, complex DAGs |
| **Prefect** | Modern Airflow alternative | Cloud or self-hosted | Python-native, modern UX |
| **dbt** | SQL transformation + lineage | Self-hosted or dbt Cloud | Analytics engineering |
| **Azure Databricks Workflows** | Spark-native orchestration | Managed Databricks | Spark-heavy workloads |
| **Apache Kafka + Flink** | Event streaming + processing | Self-hosted | Real-time streaming pipelines |

---

### 8.2 Full Comparison

| Dimension | Azure Data Factory | Apache Airflow | Prefect | dbt |
|-----------|-------------------|---------------|---------|-----|
| **Primary use** | Data movement + light transform | Complex workflow orchestration | Workflow orchestration | SQL transformations + lineage |
| **Code vs GUI** | GUI-first (+ JSON ARM) | Python code (DAGs) | Python code | SQL + YAML |
| **Connectors** | 200+ native (SAP, Salesforce, Oracle) | 700+ community providers | 100+ collections | DB-only (via adapters) |
| **Trigger types** | Schedule, event, manual | Schedule, sensor, manual | Schedule, event, manual | Schedule, manual |
| **Retry/backfill** | Good | Excellent | Excellent | Good |
| **Data lineage** | Azure Purview integration | Limited native (OpenLineage plugin) | Limited | Built-in (column-level) |
| **Monitoring** | Azure Monitor | Airflow UI + Prometheus | Prefect Cloud UI | dbt Cloud / CLI |
| **Self-hosted cost** | N/A (managed, pay per run) | ~$200/month (AKS + DB) | Free tier / $400+/month | Free (OSS) |
| **Managed cost** | $0.25/1K runs + DIU cost | Astronomer $1,000+/month | Prefect Cloud ~$400/month | dbt Cloud $100+/month |
| **Learning curve** | Low | Medium | Low-Medium | Low |
| **Best for** | Azure shop, data engineers, regulated | Python teams, complex orchestration | Modern Airflow alternative | Analytics engineering, data modeling |

---

### 8.3 Recommendation by Use Case

```
Financial services data platform (regulated):
  → ADF for raw ingestion (200+ connectors, Private Endpoint, Purview lineage)
  → dbt Cloud for transformation layer (SQL, lineage, tests)
  → Databricks Workflows for Spark-heavy ML pipelines

Startup / scale-up (Python-native team):
  → Prefect (simpler than Airflow, hosted option reduces ops)
  → dbt Core (free) for SQL transforms
  → Great Expectations for data quality

Large enterprise (multi-cloud, portability required):
  → Apache Airflow on AKS (KubernetesExecutor — fully portable)
  → dbt Core for transforms
  → Apache Kafka for real-time, Flink for stream processing

IoT / real-time pipeline:
  → Skip batch orchestrators entirely
  → Azure Event Hubs (managed Kafka) + Azure Stream Analytics
  → OR: Apache Kafka + Apache Flink (OSS, more powerful)
```

---

## 9. Cloud Strategy Patterns

### 9.1 Three Postures

#### Posture A: Azure-First (Recommended for regulated Canadian enterprises)

```
Philosophy: Maximize use of Azure managed services.
            Minimize operational overhead. Compliance built-in.

Stack:  Azure OpenAI · AI Search · CosmosDB · AKS · ADLS Gen2
        Azure Databricks · ADF · Key Vault · Purview · Monitor

When to choose:
  ✓ OSFI/PIPEDA/FINTRAC regulated
  ✓ < 20-person platform team
  ✓ Need to move fast (weeks not months)
  ✓ Microsoft Enterprise Agreement already in place
  ✓ Teams/M365 integration needed

Lock-in mitigation (even in Azure-first):
  - Store data in open formats: Delta Lake, Apache Parquet, Iceberg
  - Use Kubernetes (AKS) for compute — workloads are portable
  - Avoid vendor-specific stored procedures in Synapse
  - Use standard SQL where possible
```

#### Posture B: Hybrid (Azure infrastructure + OSS tooling)

```
Philosophy: Use Azure for what it does uniquely well (networking, compliance,
            managed identity, private endpoints). Run OSS on top for flexibility.

Infrastructure:  AKS · ADLS Gen2 · ExpressRoute · Key Vault · Entra ID
Tooling:         Airflow · dbt · Qdrant · LangChain · vLLM · MLflow

When to choose:
  ✓ Large Python/data-engineering team
  ✓ Multi-cloud strategy in 2–3 year roadmap
  ✓ Need latest OSS models and tools (not gated by Azure roadmap)
  ✓ Cost optimization at scale (>$50K/month Azure bill)
  ✓ Open formats required (Iceberg, Delta Lake)

Architecture pattern:
  "Azure is the substrate, OSS is the application layer"
```

#### Posture C: Cloud-Agnostic (Kubernetes-native)

```
Philosophy: Everything runs in containers. Cloud is just commodity infrastructure.
            Full portability between Azure, AWS, GCP, on-prem.

Pattern:   Kubernetes everywhere (AKS / EKS / GKE)
           GitOps: ArgoCD or Flux v2
           Service mesh: Istio or Linkerd
           Storage: MinIO (S3-compatible) or cloud-native blob
           Secrets: Vault (HashiCorp) — not Key Vault
           Observability: Prometheus + Grafana + Jaeger (not Azure Monitor)

When to choose:
  ✓ Multi-cloud mandate (board-level decision)
  ✓ Regulated workloads needing portability (avoid single-cloud risk)
  ✓ Large platform team (50+) with K8s expertise
  ✓ Building a product that deploys to customer environments

Cost reality:
  Higher operational complexity. Requires dedicated platform/SRE team.
  Not recommended for teams < 15 engineers.
```

### 9.2 Cloud Strategy Decision Matrix

| Scenario | Recommended Posture | Why |
|----------|--------------------|----|
| Canadian bank, OSFI regulated | Azure-First | Compliance, data residency, SLA |
| Global SaaS startup | Hybrid (Azure + OSS) | Portability, cost, speed |
| Government (Fed/Provincial) | Azure Government + OSS | Sovereignty, Protected B |
| Insurance, medium enterprise | Azure-First | OSFI/provincial compliance |
| Global logistics (multi-region) | Hybrid | Multi-region, CDN, cost |
| Enterprise tech company | Cloud-Agnostic | Multi-cloud resilience |
| Healthcare (PHI/PHIPA) | Azure-First (Canada) | Data residency, PHIPA compliance |

---

## 10. Full Architecture Blueprints by Use Case

### 10.1 Use Case A: Canadian Bank Compliance Chatbot
**Constraint: Regulated, OSFI B-10, no data outside Canada**
**Stack: Fully Azure Native**

```
Stack choice rationale:
  Vector DB     → Azure AI Search (not Qdrant) — compliance, no infra ops
  LLM           → Azure OpenAI GPT-4o (not vLLM) — Microsoft data handling attestation
  Agent         → Semantic Kernel (not LangGraph) — .NET team, Azure integration
  Orchestration → ADF + Databricks (not Airflow) — Purview lineage required
  Audit         → Log Analytics WORM (7-year FINTRAC requirement)

Full stack:
  Migration:  Data Box + ExpressRoute + ADF Self-Hosted IR
  Storage:    ADLS Gen2 (Delta Lake) + Blob Archive
  Processing: Azure Video Indexer + AI Speech + Document Intelligence
  Indexing:   Azure AI Search S2 (2 replicas, 1 partition)
  Metadata:   CosmosDB (serverless for dev, autoscale for prod)
  Cache:      Azure Cache for Redis (C3 tier)
  Agent:      Semantic Kernel on AKS + Azure AI Agent Service
  LLM:        Azure OpenAI GPT-4o PTU (2 PTUs for stable throughput)
  Safety:     Azure Content Safety (input + output)
  Auth:       Entra ID + Workload Identity (no secrets)
  Network:    Private Endpoints on all services, hub-spoke VNet
  Observability: App Insights + Log Analytics + Azure Monitor dashboards
```

---

### 10.2 Use Case B: Startup Internal Knowledge Base
**Constraint: 10-person team, $5K/month budget, Python-native**
**Stack: OSS on Azure AKS**

```
Stack choice rationale:
  Vector DB     → Qdrant (self-hosted on AKS) — free, fast, feature-rich
  LLM           → Azure OpenAI GPT-4o-mini (cheap) + Ollama for dev
  Agent         → LangGraph (Python-native, flexible stateful agents)
  Orchestration → Prefect (simpler than Airflow, hosted tier available)
  Eval          → RAGAS + Langfuse (both OSS)

Full stack:
  Migration:  rclone + AzCopy (one-time load)
  Storage:    Azure Blob (LRS, Cool tier) — cheap at small scale
  Processing: Whisper OSS (audio) + Unstructured.io (docs)
  Embedding:  text-embedding-3-small via Azure OpenAI (cheapest quality option)
  Indexing:   Qdrant (AKS, single node, 1 replica — it's a startup)
  Metadata:   PostgreSQL (Azure Flexible Server) — one DB for everything
  Cache:      Redis OSS on AKS (no separate managed Redis cost)
  Agent:      LangGraph on AKS (2 replicas)
  LLM:        Azure OpenAI GPT-4o-mini (10× cheaper than GPT-4o)
  Safety:     LlamaGuard (OSS) for content filtering
  Auth:       Entra ID + AKS Workload Identity
  Observability: Prometheus + Grafana + Langfuse
  Total cost: ~$2,500–4,000/month
```

---

### 10.3 Use Case C: Global Retailer — Peak Load RAG + Personalization
**Constraint: 10x traffic spikes, multi-region, cost-optimized at scale**
**Stack: Hybrid (Azure infra + OSS tooling)**

```
Stack choice rationale:
  Vector DB     → Qdrant (self-hosted) — AI Search too expensive at 500M vectors
  LLM           → Azure OpenAI GPT-4o for generation (quality) +
                  text-embedding-3-large for embeddings (accuracy)
  Agent         → LangChain (Python team, large OSS ecosystem)
  Orchestration → Apache Airflow on AKS (complex DAGs for product catalog refresh)
  Streaming     → Azure Event Hubs (managed Kafka) for clickstream
  Personalization → Azure Personalizer OR custom RecSys on Databricks

Full stack:
  Migration:  ADF + ExpressRoute (ongoing sync from on-prem PIM/ERP)
  Storage:    ADLS Gen2 (Delta Lake) + Blob CDN for product assets
  Processing: Databricks (product catalog enrichment, NLP tagging)
  Embedding:  Azure OpenAI text-embedding-3-large
  Indexing:   Qdrant (AKS, 3-node cluster, 500M product vectors)
  Metadata:   CosmosDB (product catalog, user profiles)
  Cache:      Redis (Azure managed C4) — product catalog TTL 5min
  Agent:      LangChain on AKS (KEDA scale on request queue)
  LLM:        Azure OpenAI GPT-4o (PTU for peak)
  Network:    Azure Front Door (global LB) + APIM (throttle + cache)
  Observability: Azure Monitor + Prometheus + Grafana
  Total cost: ~$25,000–40,000/month at scale
```

---

## 11. Decision Matrix — One-Page Cheat Sheet

```
┌──────────────────────┬─────────────────────────────────────────────────┐
│ WHAT YOU NEED        │ RECOMMENDED CHOICE                               │
├──────────────────────┼─────────────────────────────────────────────────┤
│ Vector DB            │                                                  │
│  Regulated/OSFI      │ Azure AI Search (hybrid, semantic ranker)        │
│  High scale OSS      │ Qdrant (best perf/cost, Apache 2.0)             │
│  Already on PG       │ pgvector (simple, transactional consistency)     │
│  Prototyping         │ Chroma (local only, not production)             │
├──────────────────────┼─────────────────────────────────────────────────┤
│ LLM                  │                                                  │
│  Best quality        │ Azure OpenAI GPT-4o                             │
│  Cost at scale       │ GPT-4o-mini OR Mistral-7B via vLLM              │
│  Data stays on-prem  │ vLLM + Llama 3.3 / Phi-4 on AKS GPU            │
│  Dev/local           │ Ollama + Llama 3.2 (runs on MacBook)            │
├──────────────────────┼─────────────────────────────────────────────────┤
│ RAG Framework        │                                                  │
│  .NET/C# team        │ Semantic Kernel                                  │
│  Python, most flex   │ LangChain                                        │
│  Document-heavy RAG  │ LlamaIndex                                       │
│  Search-first        │ Haystack                                         │
├──────────────────────┼─────────────────────────────────────────────────┤
│ Agentic AI           │                                                  │
│  Azure native        │ Semantic Kernel + Azure AI Agent Service         │
│  Complex workflows   │ LangGraph (stateful, explicit graph)             │
│  Multi-agent OSS     │ AutoGen (conversation-based, mature)             │
│  Rapid prototyping   │ CrewAI (role-based, easy to start)              │
├──────────────────────┼─────────────────────────────────────────────────┤
│ Pipeline Orchestr.   │                                                  │
│  Azure shop          │ Azure Data Factory + Databricks Workflows        │
│  Python team         │ Airflow (AKS) or Prefect                         │
│  SQL transforms      │ dbt (always — regardless of other choices)       │
│  Real-time           │ Azure Event Hubs + Stream Analytics / Flink      │
├──────────────────────┼─────────────────────────────────────────────────┤
│ Cloud Strategy       │                                                  │
│  Regulated Canada    │ Azure-First                                      │
│  Python + portable   │ Hybrid (Azure infra + OSS tooling)              │
│  Multi-cloud mandate │ Cloud-Agnostic (K8s + HashiCorp + OSS)          │
└──────────────────────┴─────────────────────────────────────────────────┘
```

---

## 12. Further Reading & External Resources

### Azure Native Documentation

| Topic | Link |
|-------|------|
| Azure AI Search — Vector Search | https://learn.microsoft.com/azure/search/vector-search-overview |
| Azure OpenAI Service | https://learn.microsoft.com/azure/ai-services/openai/overview |
| Azure AI Foundry (Agent Service) | https://learn.microsoft.com/azure/ai-studio/overview |
| Semantic Kernel Documentation | https://learn.microsoft.com/semantic-kernel/overview |
| Azure Data Factory | https://learn.microsoft.com/azure/data-factory/introduction |
| Azure Databricks | https://learn.microsoft.com/azure/databricks/introduction |
| Azure Video Indexer | https://learn.microsoft.com/azure/azure-video-indexer/video-indexer-overview |
| Azure AI Speech — Batch Transcription | https://learn.microsoft.com/azure/ai-services/speech-service/batch-transcription |
| Azure Well-Architected Framework | https://learn.microsoft.com/azure/architecture/framework |
| Azure Architecture Center | https://learn.microsoft.com/azure/architecture |
| CosmosDB — Vector Search | https://learn.microsoft.com/azure/cosmos-db/nosql/vector-search |
| AKS — Workload Identity | https://learn.microsoft.com/azure/aks/workload-identity-overview |
| Azure Content Safety | https://learn.microsoft.com/azure/ai-services/content-safety/overview |

---

### OSS Tools — Official Docs

| Tool | Link |
|------|------|
| LangChain | https://python.langchain.com/docs/introduction |
| LangGraph | https://langchain-ai.github.io/langgraph |
| LlamaIndex | https://docs.llamaindex.ai/en/stable |
| AutoGen (Microsoft) | https://microsoft.github.io/autogen |
| CrewAI | https://docs.crewai.com |
| Haystack (deepset) | https://docs.haystack.deepset.ai/docs/intro |
| Qdrant | https://qdrant.tech/documentation |
| Weaviate | https://weaviate.io/developers/weaviate |
| pgvector | https://github.com/pgvector/pgvector |
| vLLM | https://docs.vllm.ai/en/latest |
| Ollama | https://ollama.com/blog/openai-compatibility |
| Hugging Face TGI | https://huggingface.co/docs/text-generation-inference |
| Apache Airflow | https://airflow.apache.org/docs/apache-airflow/stable/index.html |
| Prefect | https://docs.prefect.io |
| dbt Documentation | https://docs.getdbt.com |
| Apache Kafka | https://kafka.apache.org/documentation |
| Debezium (CDC) | https://debezium.io/documentation/reference/stable |
| Apache Flink | https://nightlies.apache.org/flink/flink-docs-stable |
| MLflow | https://mlflow.org/docs/latest/index.html |
| Langfuse (LLM observability) | https://langfuse.com/docs |
| RAGAS (RAG evaluation) | https://docs.ragas.io |
| Unstructured.io | https://docs.unstructured.io |
| Great Expectations | https://docs.greatexpectations.io |
| sentence-transformers | https://www.sbert.net/docs/sentence_transformer/pretrained_models.html |

---

### Compliance & Regulatory References (Canada)

| Regulation | Link |
|------------|------|
| OSFI B-10 — Third-Party Risk | https://www.osfi-bsif.gc.ca/en/guidance/guidance-library/third-party-risk-management-guideline |
| OSFI E-23 — Model Risk Management | https://www.osfi-bsif.gc.ca/en/guidance/guidance-library/model-risk-management-guideline |
| PIPEDA | https://www.priv.gc.ca/en/privacy-topics/privacy-laws-in-canada/the-personal-information-protection-and-electronic-documents-act-pipeda |
| FINTRAC Compliance | https://www.fintrac-canafe.gc.ca/compliance-conformite/1-eng |
| Quebec Law 25 (privacy) | https://www.cai.quebec.ca/en/law-25 |

---

### Architecture Patterns & Research Papers

| Resource | Link |
|----------|------|
| RAG Survey (Meta / academic) | https://arxiv.org/abs/2312.10997 |
| Agentic AI Patterns (LangGraph) | https://langchain-ai.github.io/langgraph/concepts/agentic_concepts |
| ReAct: Reason + Act paper | https://arxiv.org/abs/2210.03629 |
| HyDE: Hypothetical Document Embeddings | https://arxiv.org/abs/2212.10496 |
| RAGAS evaluation framework paper | https://arxiv.org/abs/2309.15217 |
| Microsoft AutoGen paper | https://arxiv.org/abs/2308.08155 |
| Azure AI Reference Architecture | https://learn.microsoft.com/azure/architecture/ai-ml/architecture/baseline-openai-e2e-chat |

---

### Books & Courses (Mentor Recommendations)

| Resource | Format | Focus |
|----------|--------|-------|
| *Designing Machine Learning Systems* — Chip Huyen | Book | MLOps, production AI systems |
| *Building LLMs for Production* — Louis-François Bouchard | Book | LLM engineering |
| *Fundamentals of Data Engineering* — Joe Reis & Matt Housley | Book | Data platform design |
| Microsoft Learn: AI-102 path | Course (free) | Azure AI Engineer certification |
| Microsoft Learn: DP-203 path | Course (free) | Azure Data Engineer certification |
| DeepLearning.AI: LangChain for LLM Application Dev | Course (free) | LangChain hands-on |
| DeepLearning.AI: Building Agentic RAG with LlamaIndex | Course (free) | Agentic RAG patterns |
| DeepLearning.AI: Multi AI Agent Systems with CrewAI | Course (free) | Multi-agent design |

---

> **Mentor's final note:**  
> The best architecture is not the one with the most impressive tools — it's the one your team can build, deploy, operate, and evolve confidently within your regulatory and budget constraints.  
> When in doubt: start with Azure Native to move fast, identify where you're hitting ceilings or costs, then selectively introduce OSS alternatives for those specific components.  
> Architecture is an evolving conversation, not a one-time decision.

---

*Document version: 1.0 | Created: 2026-05-20 | Author: Abhishek Sinha*  
*Repo: https://github.com/AbhishekSinha02/AzureArchitecture*  
*Related: [UnstructuredData-Migration-RAG-AgenticAI.md](./UnstructuredData-Migration-RAG-AgenticAI.md)*
