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

### 3. [CLAUDE.md — Architect Context](./CLAUDE.md)
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
| Prepare for an architecture interview | [Interview Q&A](./UnstructuredData-Migration-RAG-AgenticAI.md#11-interview-qa-bank) |

---

## Repo Structure

```
AzureArchitecture/
├── README.md                                          ← this file
├── CLAUDE.md                                          ← architect context + patterns
├── UnstructuredData-Migration-RAG-AgenticAI.md        ← migration + RAG + Agentic AI design
└── TechStack-Comparison-DataMigration-RAG-AgenticAI.md ← Azure Native vs OSS comparison
```

---

*Last updated: 2026-05-20*
