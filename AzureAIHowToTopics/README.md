# Azure AI How-To — Scenario Library

> *"An AI system that works in a demo but fails in production  
>  is not an AI system. It's a proof of concept with ambitions."*

### 🟢 Beginner → 🟡 Intermediate → 🔴 Advanced

---

## Files

| # | File | What's inside |
|---|------|---------------|
| 01 | [azure-openai.md](01-azure-openai.md) | Deployments, PTU vs consumption, function calling, streaming, rate limit handling |
| 02 | [rag-ai-search.md](02-rag-ai-search.md) | RAG pipeline, chunking strategy, hybrid BM25+vector, multi-turn chat |
| 03 | [document-intelligence.md](03-document-intelligence.md) | Prebuilt + custom models, high-volume pipeline, OCR validation |
| 04 | [content-safety.md](04-content-safety.md) | Content filtering, prompt injection, PII redaction, output validation |
| 05 | [llmops-monitoring.md](05-llmops-monitoring.md) | Usage monitoring, RAGAS evals, A/B testing via APIM, drift detection |
| 06 | [ai-agents.md](06-ai-agents.md) | Assistants API, Semantic Kernel, multi-agent workflows, agent memory |
| 07 | [ai-studio-foundry.md](07-ai-studio-foundry.md) | AI Foundry hub/project, fine-tuning, model deployment, PromptFlow |
| 08 | [vector-search.md](08-vector-search.md) | AI Search index, hybrid search + RRF, semantic ranking, large-scale indexing |
| 09 | [ai-security-compliance.md](09-ai-security-compliance.md) | OSFI B-10, data residency, human-in-the-loop, audit logging |
| 10 | [commands-reference.md](10-commands-reference.md) | Every az / Python SDK / REST API / KQL command with context |

---

## How to Read

Every answer follows the validated pattern:
1. **Scenario** — the real situation you're in
2. **Diagram** — ASCII topology or flow
3. **Answer** — bullet points, **services bolded**
4. **Commands** — runnable, annotated with expected output
5. **⚠️ Gotcha** — the one thing that breaks this in production
6. **💡 Deep dive hint** — follow-up thread topic
