# High-Volume Unstructured Data Migration to Azure
## For Agentic AI & RAG — Principal Architect Reference

> **Scope:** End-to-end design for migrating PB-scale video/audio assets from on-premises to Azure,  
> then building a production-grade RAG pipeline and Agentic AI system on top of that corpus.  
> **Audience:** Principal/Lead Architects, AI Platform Engineers, Enterprise Decision-Makers  
> **Regulatory lens:** OSFI B-10, PIPEDA, FINTRAC (Canadian financial/enterprise context)

---

## Table of Contents

1. [Problem Statement & Scale Signals](#1-problem-statement--scale-signals)
2. [Migration Architecture — Phase by Phase](#2-migration-architecture--phase-by-phase)
3. [Azure Tech Stack — Full Reference](#3-azure-tech-stack--full-reference)
4. [What is RAG? (Definition + Architecture)](#4-what-is-rag-definition--architecture)
5. [RAG Pipeline for Video & Audio](#5-rag-pipeline-for-video--audio)
6. [What is Agentic AI? (Definition + Architecture)](#6-what-is-agentic-ai-definition--architecture)
7. [Agentic AI Design on Top of Migrated Corpus](#7-agentic-ai-design-on-top-of-migrated-corpus)
8. [AI Governance Framework](#8-ai-governance-framework)
9. [Security & Compliance Architecture](#9-security--compliance-architecture)
10. [Cost Model](#10-cost-model)
11. [Interview Q&A Bank](#11-interview-qa-bank)

---

## 1. Problem Statement & Scale Signals

### What We Are Solving

| Dimension | Details |
|-----------|---------|
| **Data type** | Raw video (MP4/MKV/MOV), audio (WAV/MP3/AAC), transcripts, captions, metadata JSON |
| **Origin** | On-premises NAS/SAN arrays, legacy media servers, tape archives |
| **Destination** | Azure Blob Storage → AI-ready knowledge base (AI Search + CosmosDB) |
| **End use** | RAG-powered chatbots, Agentic AI workflows, semantic search, compliance retrieval |

### Enterprise Size Tiers for This Design

| Tier | Volume | Concurrent Users | Migration Window |
|------|--------|-----------------|-----------------|
| **Small** | <10 TB, <5K assets | <50 | Days–weeks, AzCopy |
| **Medium** | 10–500 TB, 5K–500K assets | 50–500 | Weeks, ADF + IR |
| **Large** | 500 TB–PB, 500K+ assets | 500–10K | Months, Data Box + parallel |

---

## 2. Migration Architecture — Phase by Phase

### 2.1 Overall Migration Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         ON-PREMISES                                      │
│  NAS/SAN ──► Media Server ──► Tape Archive                              │
└────────────────────┬────────────────────────────────────────────────────┘
                     │  ExpressRoute (>10TB) / VPN (<10TB)
                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     AZURE INGESTION LAYER                                │
│                                                                          │
│  Azure Data Box ──► Blob Storage (Raw/Landing Zone)                     │
│       OR                                                                 │
│  AzCopy (parallel) ──► Blob Storage (Raw/Landing Zone)                  │
└────────────────────┬────────────────────────────────────────────────────┘
                     │  Event Grid (blob created trigger)
                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                   PROCESSING & ENRICHMENT LAYER                          │
│                                                                          │
│  Azure Media Services / FFmpeg on AKS  ◄── transcoding                 │
│  Azure Video Indexer                   ◄── transcript + scene tags      │
│  Azure AI Speech (Batch Transcription) ◄── audio → text                 │
│  Azure Document Intelligence           ◄── caption/subtitle extraction  │
└────────────────────┬────────────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                  KNOWLEDGE BASE LAYER (AI-Ready)                         │
│                                                                          │
│  Chunks + Embeddings ──► Azure AI Search (vector + keyword hybrid)      │
│  Metadata + Timestamps ──► CosmosDB (fast lookup, session state)        │
│  Original Assets ──► Blob Storage (Hot/Cool/Archive tiered)             │
│  Search Index ──► Azure AI Search (HNSW vector index)                   │
└────────────────────┬────────────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│              RAG + AGENTIC AI SERVING LAYER                              │
│                                                                          │
│  Azure OpenAI (GPT-4o / GPT-4o-mini)                                   │
│  Semantic Kernel / Azure AI Agent Service                               │
│  APIM (throttling, versioning, audit)                                   │
│  Azure Front Door (global routing, WAF)                                 │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### 2.2 Phase 1 — Assessment & Baseline (Weeks 1–2)

**Goal:** Know what you have before you move it.

```
Tools:
  Azure Migrate          → inventory on-prem servers, estimate sizing
  Azure Storage Mover    → scan NAS shares, estimate transfer time
  Custom inventory script → file count, size distribution, format types

Outputs:
  - Total volume (TB) per format (MP4, WAV, MKV…)
  - Age distribution (informs storage tier assignment)
  - Dependency map (which apps read these files?)
  - Network bandwidth assessment (burst capacity for transfer window)
  - Compliance classification (PII in video? FINTRAC relevance?)
```

**Decision gate:** Volume drives transfer method:

| Volume | Method | Estimated Transfer Time (1 Gbps link) |
|--------|---------|---------------------------------------|
| <1 TB | AzCopy direct | Hours |
| 1–10 TB | AzCopy + parallelism flags | 1–3 days |
| 10–40 TB | ADF Self-Hosted IR + AzCopy | 5–10 days |
| >40 TB | Azure Data Box (80 TB device) | Ship + ingest 1–2 weeks |
| >1 PB | Azure Data Box Heavy (1 PB) | Multiple devices in parallel |

---

### 2.3 Phase 2 — Network & Landing Zone Setup (Weeks 2–3)

```hcl
# Terraform: Hub-Spoke VNet for migration workloads
resource "azurerm_virtual_network" "migration_hub" {
  name                = "vnet-migration-hub"
  location            = "canadacentral"
  resource_group_name = azurerm_resource_group.migration.name
  address_space       = ["10.10.0.0/16"]
}

resource "azurerm_subnet" "private_endpoints" {
  name                 = "snet-private-endpoints"
  resource_group_name  = azurerm_resource_group.migration.name
  virtual_network_name = azurerm_virtual_network.migration_hub.name
  address_prefixes     = ["10.10.1.0/24"]
}

# Private Endpoint for Storage — no public access during migration
resource "azurerm_private_endpoint" "storage" {
  name                = "pe-storage-migration"
  location            = "canadacentral"
  resource_group_name = azurerm_resource_group.migration.name
  subnet_id           = azurerm_subnet.private_endpoints.id

  private_service_connection {
    name                           = "psc-storage"
    private_connection_resource_id = azurerm_storage_account.raw.id
    subresource_names              = ["blob"]
    is_manual_connection           = false
  }
}
```

**Landing zone Blob Storage structure:**

```
storageaccount-raw/
  ├── landing/          ← AzCopy / Data Box drops here (raw as-is)
  ├── validated/        ← checksum verified, format confirmed
  ├── processing/       ← actively being transcribed/indexed
  └── quarantine/       ← failed validation, needs review

storageaccount-curated/
  ├── transcripts/      ← plain text output from Video Indexer / Speech
  ├── chunks/           ← chunked text ready for embedding
  ├── embeddings/       ← vector files (backup / reindex source)
  └── metadata/         ← JSON sidecar per asset

storageaccount-archive/
  └── originals/        ← Cool or Archive tier, lifecycle-managed
```

---

### 2.4 Phase 3 — Parallel Transfer (Weeks 3–8, scale-dependent)

**AzCopy optimized command for large transfers:**

```powershell
# Parallel transfer with 128 concurrent connections and 256 MB block size
azcopy copy `
  "\\nas-server\media-library\*" `
  "https://stgrawmigration.blob.core.windows.net/landing?<SAS>" `
  --recursive `
  --block-size-mb 256 `
  --cap-mbps 800 `            # leave 20% headroom on 1Gbps link
  --parallel-level 128 `
  --check-md5 FailIfDifferent ` # checksum validation on every file
  --log-level INFO `
  --output-type json | Tee-Object -FilePath azcopy-log.json

# After transfer: verify with azcopy jobs show <jobID> --with-status Failed
```

**Azure Data Box workflow (>40 TB):**

```
1. Order Data Box via Azure Portal (80TB capacity, ~$320/order)
2. Microsoft ships device (7–10 business days)
3. On-prem team:
   a. Connect to local network via 10GbE
   b. Copy using SMB/NFS mount: \\databox\video-share\
   c. Run local checksum manifest before return
4. Ship back → Azure ingests to Blob (2–3 days ingest at DC)
5. Verify: compare checksum manifest with Blob inventory report
6. Order next device for next batch (pipeline overlapping orders)
```

---

### 2.5 Phase 4 — Validation & Delta Sync

```python
# ADF pipeline: checksum validation activity (Python script)
import hashlib
import json
from azure.storage.blob import BlobServiceClient

def validate_transfer(source_manifest: str, storage_account: str, container: str):
    """
    Compare MD5 checksums between on-prem manifest and Azure Blob.
    Run after every AzCopy job before marking batch as complete.
    """
    blob_client = BlobServiceClient.from_connection_string(
        f"DefaultEndpointsProtocol=https;AccountName={storage_account}..."
    )
    container_client = blob_client.get_container_client(container)

    with open(source_manifest) as f:
        manifest = json.load(f)  # [{filename, md5, size_bytes}, ...]

    failed = []
    for item in manifest:
        blob = container_client.get_blob_client(item["filename"])
        props = blob.get_blob_properties()
        azure_md5 = props.content_settings.content_md5

        if azure_md5 != item["md5"]:
            failed.append({
                "file": item["filename"],
                "expected": item["md5"],
                "actual": azure_md5
            })

    if failed:
        raise Exception(f"Checksum mismatch for {len(failed)} files: {failed}")

    print(f"Validation passed: {len(manifest)} files verified.")
```

**Delta sync for active on-prem systems (zero-downtime pattern):**

```
Source NAS (still active)
    │
    ├── ADF incremental copy (watermark = last_modified timestamp)
    │   Runs every 15 minutes during migration window
    │
    └── Event-based: inotifywait (Linux) / FileSystemWatcher (.NET)
        → push new/modified files to Service Bus queue
        → AKS worker pulls queue, syncs to Azure

Cutover steps:
  1. Set source NAS to read-only (freeze new writes)
  2. Run final delta sync — wait for queue depth = 0
  3. Update app config to point to Azure Blob endpoint
  4. Validate app smoke tests pass
  5. Keep NAS hot (read-only) for 72h rollback window
  6. After 72h: decommission or archive NAS to tape
```

---

### 2.6 Phase 5 — Processing & AI Enrichment Pipeline

This is where raw video/audio becomes AI-queryable knowledge.

```
┌──────────────────────────────────────────────────────────┐
│           EVENT-DRIVEN ENRICHMENT PIPELINE               │
│                                                          │
│  Blob Created (landing/)                                 │
│       │                                                  │
│       ▼ Event Grid trigger                               │
│  Service Bus Queue (enrichment-jobs)                     │
│       │                                                  │
│       ▼ KEDA scale (queue depth → AKS pods)             │
│  AKS Worker Pod                                          │
│       │                                                  │
│       ├── If VIDEO → Azure Video Indexer API            │
│       │     → transcript (timestamped sentences)        │
│       │     → speaker labels                            │
│       │     → scene detection + visual tags             │
│       │     → named entities (people, places, orgs)     │
│       │                                                  │
│       ├── If AUDIO → Azure AI Speech (Batch)            │
│       │     → transcript with confidence scores         │
│       │     → speaker diarization                       │
│       │     → language detection                        │
│       │                                                  │
│       ├── Chunking service                               │
│       │     → split transcript at sentence boundaries   │
│       │     → chunk size: 512 tokens, overlap: 50       │
│       │     → preserve timestamp metadata per chunk     │
│       │                                                  │
│       ├── Embedding service                              │
│       │     → Azure OpenAI text-embedding-3-large       │
│       │     → 3072 dimensions (or ada-002 at 1536)      │
│       │                                                  │
│       └── Index write                                   │
│             → AI Search: chunk + vector + metadata      │
│             → CosmosDB: asset-level metadata + status   │
└──────────────────────────────────────────────────────────┘
```

```python
# AKS worker: video enrichment pipeline
import asyncio
from azure.ai.videoindexer import VideoIndexerClient
from openai import AzureOpenAI
from azure.search.documents import SearchClient
from azure.cosmos import CosmosClient

async def process_video_asset(blob_url: str, asset_id: str):
    """
    Full enrichment pipeline: video → transcript → chunks → embeddings → index.
    Called by AKS worker pulling from Service Bus queue.
    """

    # Step 1: Transcribe via Video Indexer
    vi_client = VideoIndexerClient(account_id="...", subscription_key="...")
    job = await vi_client.upload_video_async(
        video_url=blob_url,
        name=asset_id,
        language="auto-detect",
        indexing_preset="AdvancedAudio"  # includes speaker diarization
    )
    transcript = await vi_client.get_transcript(job.video_id)
    # transcript = [{start_ms, end_ms, text, speaker, confidence}, ...]

    # Step 2: Chunk with timestamp preservation
    chunks = chunk_with_timestamps(transcript, max_tokens=512, overlap_tokens=50)
    # chunks = [{text, start_ms, end_ms, speaker, chunk_index}, ...]

    # Step 3: Generate embeddings
    openai_client = AzureOpenAI(
        azure_endpoint="https://YOUR-RESOURCE.openai.azure.com/",
        api_version="2024-05-01-preview"
    )
    embeddings = []
    for chunk in chunks:
        response = openai_client.embeddings.create(
            model="text-embedding-3-large",
            input=chunk["text"]
        )
        embeddings.append(response.data[0].embedding)

    # Step 4: Write to AI Search
    search_client = SearchClient(endpoint="...", index_name="video-chunks", credential="...")
    documents = [
        {
            "id": f"{asset_id}-chunk-{i}",
            "asset_id": asset_id,
            "blob_url": blob_url,
            "text": chunk["text"],
            "embedding": embeddings[i],
            "start_ms": chunk["start_ms"],
            "end_ms": chunk["end_ms"],
            "speaker": chunk["speaker"],
            "chunk_index": chunk["chunk_index"]
        }
        for i, chunk in enumerate(chunks)
    ]
    await search_client.upload_documents(documents)

    # Step 5: Update asset status in CosmosDB
    cosmos_client = CosmosClient(url="...", credential="...")
    container = cosmos_client.get_database_client("media").get_container_client("assets")
    await container.upsert_item({
        "id": asset_id,
        "status": "indexed",
        "chunk_count": len(chunks),
        "transcript_language": transcript[0].get("language"),
        "indexed_at": datetime.utcnow().isoformat()
    })
```

---

## 3. Azure Tech Stack — Full Reference

### 3.1 Migration Layer

| Component | Azure Service | Purpose | Why This, Not That |
|-----------|--------------|---------|-------------------|
| Bulk transfer <40TB | **AzCopy v10** | Parallel, resumable file copy | Faster than ADF for raw binary; checksum built-in |
| Bulk transfer >40TB | **Azure Data Box** | Offline petabyte-scale shipping | Avoids saturating WAN; $320/device vs months of bandwidth |
| Incremental sync | **Azure Data Factory** | Watermark-based delta copy | Visual pipeline, built-in retry, IR for on-prem connectivity |
| Network backbone | **ExpressRoute 10 Gbps** | Private dedicated circuit | SLA-backed, predictable latency; VPN too slow for this scale |
| Event trigger | **Azure Event Grid** | React to blob creation | Push vs. poll; sub-second latency; native Blob integration |
| Queue buffer | **Azure Service Bus Premium** | Durable job queue | Guaranteed delivery, sessions for ordering, DLQ for failures |

### 3.2 Storage Layer

| Tier | Service | Use Case | Cost Signal |
|------|---------|---------|-------------|
| **Hot** | Blob Hot | Active assets queried daily | ~$0.018/GB/month |
| **Cool** | Blob Cool | Assets accessed <1x/month | ~$0.01/GB/month |
| **Archive** | Blob Archive | Originals, compliance copies | ~$0.00099/GB/month |
| **Metadata** | CosmosDB (NoSQL) | Asset lookup, session state | RU-based autoscale |
| **Search Index** | Azure AI Search | Vector + keyword retrieval | ~$250–$2,500/month depending on tier |

**Lifecycle policy (auto-tiering):**

```json
{
  "rules": [
    {
      "name": "move-to-cool",
      "enabled": true,
      "filters": { "blobTypes": ["blockBlob"], "prefixMatch": ["originals/"] },
      "actions": {
        "baseBlob": { "tierToCool": { "daysAfterLastAccessTimeGreaterThan": 30 } }
      }
    },
    {
      "name": "move-to-archive",
      "enabled": true,
      "filters": { "blobTypes": ["blockBlob"], "prefixMatch": ["originals/"] },
      "actions": {
        "baseBlob": { "tierToArchive": { "daysAfterLastAccessTimeGreaterThan": 180 } }
      }
    }
  ]
}
```

### 3.3 AI/ML Processing Layer

| Component | Service | Config |
|-----------|---------|--------|
| Video transcription + indexing | **Azure Video Indexer** | AdvancedAudio preset, auto-language detect |
| Audio-only transcription | **Azure AI Speech — Batch** | Speaker diarization, word-level timestamps |
| Document/caption extraction | **Azure Document Intelligence** | Prebuilt layout model |
| Embedding generation | **Azure OpenAI — text-embedding-3-large** | 3072 dims; ada-002 for cost-sensitive |
| LLM (generation) | **Azure OpenAI — GPT-4o** | PTU for production, consumption for dev |
| Vector + keyword search | **Azure AI Search S2/S3** | HNSW index, semantic ranker, hybrid BM25+RRF |
| Orchestration | **Semantic Kernel** | Multi-agent, plugin system, memory |
| Compute | **AKS (Standard_D8s_v5 nodes)** | KEDA autoscale on Service Bus queue |

---

## 4. What is RAG? (Definition + Architecture)

### 4.1 Definition

**RAG (Retrieval-Augmented Generation)** is an AI architecture pattern that prevents LLM hallucinations by grounding generated responses in retrieved, factual context from a curated knowledge base.

```
WITHOUT RAG:
  User: "What did the CEO say in the Q3 earnings call?"
  LLM:  Makes up a plausible-sounding answer (hallucination)

WITH RAG:
  User: "What did the CEO say in the Q3 earnings call?"
  RAG:  1. Retrieves actual transcript chunks from Q3 call (AI Search)
        2. Passes those chunks as context to LLM
        3. LLM answers using ONLY the retrieved context
        4. Response is grounded, citable, accurate
```

### 4.2 RAG Architecture — Two Phases

**Phase A: Indexing (offline, runs during/after migration)**

```
Raw Asset (Video/Audio)
    │
    ▼
Transcription (Video Indexer / Speech)
    │
    ▼
Chunking (512 tokens, 50-token overlap, sentence-boundary aware)
    │
    ▼
Embedding (text-embedding-3-large → 3072-dim vector)
    │
    ▼
Index Write → Azure AI Search
              ├── vector field (embedding)
              ├── text field (chunk text)
              ├── metadata fields (asset_id, timestamp, speaker, source_type)
              └── semantic configuration (prioritize text + speaker fields)
```

**Phase B: Query (real-time, every user request)**

```
User Query: "What compliance risks were mentioned in board meetings?"
    │
    ▼
Query Embedding (same model: text-embedding-3-large)
    │
    ▼
Hybrid Search (Azure AI Search)
    ├── Vector search: cosine similarity against all chunk embeddings
    ├── Keyword search: BM25 on text fields
    └── RRF fusion: combines both rankings (Reciprocal Rank Fusion)
    │
    ▼
Semantic Reranking (L2 model re-scores top-50 candidates)
    │
    ▼
Top-K Chunks retrieved (K=5 typically, with citation metadata)
    │
    ▼
Prompt Construction:
    System: "You are a compliance analyst. Answer using ONLY the provided context.
             If the answer is not in the context, say 'I don't know.'"
    Context: [chunk1 text + source citation] [chunk2 text + source citation] ...
    User: "What compliance risks were mentioned in board meetings?"
    │
    ▼
Azure OpenAI GPT-4o → Grounded Response + Citations
```

### 4.3 Why Hybrid Search (not pure vector)?

| Scenario | Best Retrieval Method |
|----------|----------------------|
| Semantic similarity ("What was said about risk?") | Vector |
| Exact term match ("OSFI B-10 paragraph 12") | Keyword (BM25) |
| Combined ("Find B-10 discussions about model risk") | Hybrid RRF |

**Always use hybrid for enterprise.** Pure vector misses exact terminology.  
Pure keyword misses paraphrased meaning. RRF fusion outperforms both individually.

---

## 5. RAG Pipeline for Video & Audio

### 5.1 Video-Specific Enrichment

Video adds richness beyond plain transcription. Azure Video Indexer extracts:

```
Video Asset → Azure Video Indexer
    │
    ├── Transcript (timestamped at sentence level)
    ├── Speaker labels (Speaker 1, Speaker 2 — or named if trained)
    ├── Scene detection (visual chapter breaks)
    ├── Visual labels (objects, actions seen on screen)
    ├── Named entities (people, organizations, locations mentioned)
    ├── Sentiment per utterance
    ├── Key topics per scene
    └── OCR (text visible on slides/whiteboards in frame)
```

**Chunking strategy for video:**

```python
def chunk_video_transcript(transcript: list[dict], max_tokens: int = 512) -> list[dict]:
    """
    Chunk video transcript preserving speaker + timestamp context.
    Chunk at sentence boundaries, never mid-sentence.
    Include 50-token overlap from previous chunk for context continuity.
    Attach start/end timestamps so UI can deep-link to video position.
    """
    chunks = []
    current_chunk = []
    current_tokens = 0

    for sentence in transcript:
        sentence_tokens = len(sentence["text"].split()) * 1.3  # approx token count
        if current_tokens + sentence_tokens > max_tokens and current_chunk:
            chunks.append({
                "text": " ".join([s["text"] for s in current_chunk]),
                "start_ms": current_chunk[0]["start_ms"],
                "end_ms": current_chunk[-1]["end_ms"],
                "speakers": list(set(s["speaker"] for s in current_chunk)),
                "chunk_index": len(chunks)
            })
            # Overlap: keep last 2 sentences in next chunk
            current_chunk = current_chunk[-2:]
            current_tokens = sum(len(s["text"].split()) * 1.3 for s in current_chunk)

        current_chunk.append(sentence)
        current_tokens += sentence_tokens

    return chunks
```

**AI Search index schema for video chunks:**

```json
{
  "name": "video-knowledge-base",
  "fields": [
    { "name": "id", "type": "Edm.String", "key": true },
    { "name": "asset_id", "type": "Edm.String", "filterable": true },
    { "name": "asset_title", "type": "Edm.String", "searchable": true },
    { "name": "text", "type": "Edm.String", "searchable": true },
    { "name": "embedding", "type": "Collection(Edm.Single)", "dimensions": 3072,
      "vectorSearchProfile": "hnsw-profile" },
    { "name": "start_ms", "type": "Edm.Int64", "filterable": true, "sortable": true },
    { "name": "end_ms", "type": "Edm.Int64" },
    { "name": "speakers", "type": "Collection(Edm.String)", "filterable": true },
    { "name": "source_type", "type": "Edm.String", "filterable": true },
    { "name": "date_recorded", "type": "Edm.DateTimeOffset", "filterable": true, "sortable": true },
    { "name": "named_entities", "type": "Collection(Edm.String)", "filterable": true },
    { "name": "sentiment", "type": "Edm.String", "filterable": true },
    { "name": "department", "type": "Edm.String", "filterable": true },
    { "name": "classification", "type": "Edm.String", "filterable": true }
  ],
  "vectorSearch": {
    "profiles": [{ "name": "hnsw-profile", "algorithm": "hnsw-config" }],
    "algorithms": [{
      "name": "hnsw-config", "kind": "hnsw",
      "hnswParameters": { "m": 4, "efConstruction": 400, "efSearch": 500, "metric": "cosine" }
    }]
  },
  "semantic": {
    "configurations": [{
      "name": "semantic-config",
      "prioritizedFields": {
        "contentFields": [{ "fieldName": "text" }],
        "keywordsFields": [{ "fieldName": "named_entities" }]
      }
    }]
  }
}
```

### 5.2 Audio-Specific Pipeline

For pure audio (call recordings, podcasts, earnings calls):

```python
# Azure AI Speech: batch transcription with speaker diarization
from azure.cognitiveservices.speech import SpeechConfig
from azure.cognitiveservices.speech.audio import AudioConfig
import requests

def submit_batch_transcription(blob_sas_url: str, locale: str = "en-CA") -> str:
    """Submit audio file to Speech batch API. Returns transcription job URL."""
    endpoint = "https://canadacentral.api.cognitive.microsoft.com/speechtotext/v3.2/transcriptions"
    headers = {"Ocp-Apim-Subscription-Key": SPEECH_KEY, "Content-Type": "application/json"}

    payload = {
        "contentUrls": [blob_sas_url],
        "locale": locale,
        "displayName": f"transcription-{datetime.utcnow().isoformat()}",
        "properties": {
            "diarizationEnabled": True,        # separate speakers
            "wordLevelTimestampsEnabled": True, # for precise deep-linking
            "punctuationMode": "DictatedAndAutomatic",
            "profanityFilterMode": "None"
        }
    }
    response = requests.post(endpoint, json=payload, headers=headers)
    return response.headers["Location"]  # poll this URL for completion
```

---

## 6. What is Agentic AI? (Definition + Architecture)

### 6.1 Definition

**Agentic AI** is an AI system where an LLM acts as an autonomous reasoning engine that:
1. **Plans** a multi-step sequence of actions to achieve a goal
2. **Uses tools** (APIs, databases, code execution, other agents) to gather information or take action
3. **Observes** results and adjusts its plan based on new information
4. **Iterates** until the goal is achieved or it determines it cannot proceed

```
DIFFERENCE FROM BASIC RAG:

Basic RAG:
  User asks → retrieve → generate → respond (single pass, stateless)

Agentic AI:
  User asks → PLAN → loop {
    choose tool → call tool → observe result → re-plan if needed
  } → synthesize answer → respond (multi-step, stateful)
```

### 6.2 Core Agentic AI Patterns

#### Pattern A: ReAct (Reason + Act)
```
Thought: "I need to find board meeting discussions about risk"
Action:  search_video_knowledge_base(query="risk discussions", filter="source=board-meetings")
Observation: [5 chunks retrieved from Q2 board meeting]
Thought: "I found relevant content. Now I need to check if OSFI B-10 was referenced"
Action:  search_video_knowledge_base(query="OSFI B-10 compliance", date_range="last-12-months")
Observation: [3 chunks referencing OSFI B-10]
Thought: "I have enough context. I can synthesize the compliance risk summary."
Final Answer: [grounded response with citations and video timestamps]
```

#### Pattern B: Multi-Agent (Orchestrator + Specialists)
```
User Request: "Prepare a compliance risk summary from all Q2 board meetings"
    │
    ▼
Orchestrator Agent (GPT-4o)
    ├── delegates to → Video Search Agent (searches video knowledge base)
    ├── delegates to → Document Agent (searches written board minutes)
    ├── delegates to → Compliance Agent (checks against OSFI/PIPEDA rules)
    └── synthesizes all results → Final Report
```

#### Pattern C: Human-in-the-Loop (required for financial/regulated AI)
```
Agent proposes action → confidence > threshold → auto-execute
                     → confidence < threshold → pause → human review → approve/reject → continue
```

### 6.3 Semantic Kernel — Framework of Choice

```csharp
// C# Semantic Kernel: Video Knowledge Agent
using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.Agents;
using Microsoft.SemanticKernel.Connectors.AzureAISearch;

// Build the kernel with tools (plugins)
var kernel = Kernel.CreateBuilder()
    .AddAzureOpenAIChatCompletion(
        deploymentName: "gpt-4o",
        endpoint: "https://YOUR-OPENAI.openai.azure.com/",
        apiKey: keyVaultClient.GetSecret("openai-key")
    )
    .Build();

// Register the video search plugin
kernel.Plugins.AddFromType<VideoKnowledgePlugin>("VideoSearch");
kernel.Plugins.AddFromType<ComplianceCheckPlugin>("Compliance");

// Create the agent
var agent = new ChatCompletionAgent
{
    Kernel = kernel,
    Name = "ComplianceAnalystAgent",
    Instructions = """
        You are a compliance analyst for a Canadian financial institution.
        You have access to a video knowledge base containing board meetings,
        earnings calls, and training videos. 
        
        When answering:
        1. Always search the knowledge base first before answering
        2. Cite the specific video, speaker, and timestamp for every claim
        3. Flag any potential OSFI B-10 or PIPEDA concerns you identify
        4. If you are not confident (confidence < 0.7), escalate to human review
        5. Never speculate beyond what the retrieved content supports
        """,
    Arguments = new KernelArguments(
        new OpenAIPromptExecutionSettings
        {
            ToolCallBehavior = ToolCallBehavior.AutoInvokeKernelFunctions,
            Temperature = 0.1  // low temperature for factual compliance work
        }
    )
};
```

```csharp
// The Video Knowledge Plugin — tools the agent can call
public class VideoKnowledgePlugin
{
    private readonly SearchClient _searchClient;

    [KernelFunction("search_video_knowledge_base")]
    [Description("Search the video/audio knowledge base for relevant content")]
    public async Task<string> SearchAsync(
        [Description("The search query")] string query,
        [Description("Filter by source type: board-meeting, earnings-call, training, all")] 
        string sourceType = "all",
        [Description("Maximum number of results")] int topK = 5)
    {
        var options = new SearchOptions
        {
            QueryType = SearchQueryType.Semantic,
            SemanticSearch = new SemanticSearchOptions
            {
                ConfigurationName = "semantic-config",
                SemanticQuery = query
            },
            VectorSearch = new VectorSearchOptions
            {
                Queries = { new VectorizedQuery(await GetEmbedding(query))
                    { Fields = { "embedding" }, KNearestNeighborsCount = topK * 3 } }
            },
            Filter = sourceType != "all" ? $"source_type eq '{sourceType}'" : null,
            Size = topK,
            Select = { "text", "asset_title", "start_ms", "end_ms", "speakers", "date_recorded" }
        };

        var results = await _searchClient.SearchAsync<VideoChunk>(query, options);
        var chunks = new List<string>();

        await foreach (var result in results.Value.GetResultsAsync())
        {
            var doc = result.Document;
            var timestamp = TimeSpan.FromMilliseconds(doc.StartMs).ToString(@"hh\:mm\:ss");
            chunks.Add($"[{doc.AssetTitle} | {doc.DateRecorded:yyyy-MM-dd} | {timestamp} | " +
                       $"Speaker: {string.Join(", ", doc.Speakers)}]\n{doc.Text}");
        }

        return string.Join("\n\n---\n\n", chunks);
    }
}
```

---

## 7. Agentic AI Design on Top of Migrated Corpus

### 7.1 Full System Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      USER INTERFACE LAYER                                │
│  React Web App / Teams Bot / API Client                                 │
└───────────────────────────┬─────────────────────────────────────────────┘
                            │ HTTPS
                            ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    API MANAGEMENT (APIM)                                 │
│  Rate limiting · Auth (Entra ID) · Audit logging · API versioning       │
└───────────────────────────┬─────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────────┐
│              AGENTIC ORCHESTRATION LAYER (AKS)                           │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │              Orchestrator Agent (GPT-4o)                         │   │
│  │  - Receives user intent                                          │   │
│  │  - Decomposes into sub-tasks                                    │   │
│  │  - Routes to specialist agents                                  │   │
│  │  - Synthesizes final response                                   │   │
│  └──────────┬───────────────┬──────────────────┬───────────────────┘   │
│             │               │                  │                        │
│             ▼               ▼                  ▼                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐     │
│  │ Video Search │  │  Compliance  │  │    Document Agent        │     │
│  │    Agent     │  │    Agent     │  │  (PDFs, reports, policy) │     │
│  │              │  │              │  │                          │     │
│  │ Calls:       │  │ Checks:      │  │ Calls:                   │     │
│  │ AI Search    │  │ OSFI B-10    │  │ AI Search                │     │
│  │ CosmosDB     │  │ PIPEDA rules │  │ Doc Intelligence         │     │
│  └──────────────┘  └──────────────┘  └──────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    KNOWLEDGE BASE LAYER                                  │
│  Azure AI Search (vectors) · CosmosDB (state) · Redis (semantic cache) │
└─────────────────────────────────────────────────────────────────────────┘
```

### 7.2 Memory Architecture (Agents Need Memory)

```
SHORT-TERM (within conversation):
  CosmosDB session container
  → key: session_id, value: {message_history, retrieved_chunks, tool_calls}
  → TTL: 24 hours
  → Max 20 turns before summarization (avoid context window overflow)

LONG-TERM (across conversations, per user):
  CosmosDB user_memory container  
  → key: user_id, value: {preferences, past_queries, cited_assets}
  → Used to personalize future responses (e.g., "user always asks about Q2 risk")
  → 30-day rolling window

SEMANTIC CACHE (repeated queries):
  Redis Cache
  → key: SHA256(query_embedding), value: {response, citations, ttl}
  → TTL: 1 hour for volatile content, 24 hours for archived content
  → Saves ~40% of OpenAI API costs for repeated/similar queries
```

### 7.3 Tool Inventory for Agentic System

| Tool Name | Backing Service | What Agent Can Do |
|-----------|----------------|-------------------|
| `search_videos` | Azure AI Search | Find video chunks by semantic query |
| `get_video_segment` | Azure Blob + CDN | Retrieve playable clip URL for cited timestamp |
| `search_documents` | Azure AI Search | Find PDF/report chunks |
| `get_compliance_rules` | CosmosDB | Look up OSFI/PIPEDA rule details |
| `flag_for_human_review` | Service Bus | Escalate low-confidence or sensitive responses |
| `get_session_history` | CosmosDB | Retrieve prior conversation context |
| `log_audit_event` | Log Analytics | Write immutable audit record (FINTRAC requirement) |
| `check_content_safety` | Azure Content Safety | Screen input/output for PII, harmful content |

---

## 8. AI Governance Framework

### 8.1 Why Governance Matters (Especially in Canada)

```
OSFI B-10 (2023):
  → All AI/ML models in federally regulated financial institutions need:
    • Model risk management framework
    • Independent validation before production
    • Explainability for decisions affecting customers
    • Human oversight for high-impact decisions
    • Ongoing monitoring for model drift

PIPEDA:
  → Any personal information in AI training or retrieval must be:
    • Consented to
    • Minimized (only what's needed)
    • Protected (encryption, access controls)
    • Deletable on request (right to erasure)

FINTRAC:
  → All AI used in AML/transaction monitoring must:
    • Produce immutable audit trails (7 years)
    • Be explainable to regulators on demand
    • Log every model decision with inputs/outputs
```

### 8.2 Governance Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      AI GOVERNANCE PILLARS                               │
├──────────────┬──────────────┬─────────────────┬────────────────────────┤
│  Data        │  Model       │  Runtime         │  Organizational       │
│  Governance  │  Governance  │  Governance      │  Governance           │
├──────────────┼──────────────┼─────────────────┼────────────────────────┤
│ PII detect   │ Model cards  │ Content Safety   │ AI review board       │
│ Data lineage │ Eval suite   │ Output filtering │ Risk tiering          │
│ Consent mgmt │ A/B testing  │ Rate limiting    │ Incident response     │
│ Retention    │ Drift monitor│ Human escalation │ Training/awareness    │
│ Encryption   │ Version ctrl │ Audit logging    │ Vendor assessment     │
└──────────────┴──────────────┴─────────────────┴────────────────────────┘
```

### 8.3 Data Governance for Video/Audio Assets

```python
# PII detection and redaction before indexing
from azure.ai.textanalytics import TextAnalyticsClient

async def detect_and_redact_pii(transcript_chunk: str) -> dict:
    """
    Detect PII in transcripts before writing to AI Search index.
    Required for PIPEDA compliance.
    """
    client = TextAnalyticsClient(endpoint=TEXT_ANALYTICS_ENDPOINT, credential=credential)

    result = client.recognize_pii_entities([transcript_chunk])[0]

    redacted_text = transcript_chunk
    pii_found = []

    for entity in result.entities:
        pii_found.append({
            "category": entity.category,       # e.g., "Person", "PhoneNumber"
            "confidence": entity.confidence_score,
            "offset": entity.offset,
            "length": entity.length
        })
        # Replace PII with category label
        redacted_text = redacted_text.replace(entity.text, f"[{entity.category}]")

    return {
        "original": transcript_chunk,          # stored in Key Vault, not index
        "redacted": redacted_text,             # stored in AI Search index
        "pii_detected": pii_found,
        "pii_count": len(pii_found)
    }
```

### 8.4 Model Governance — LLMOps Pipeline

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        LLMOps LIFECYCLE                                  │
│                                                                          │
│  Prompt Development                                                      │
│    Git-versioned prompts → PromptFlow evaluation → peer review          │
│                                                                          │
│  Pre-Production Evaluation (RAGAS metrics):                             │
│    • Faithfulness       : Is the answer supported by retrieved context? │
│    • Answer Relevance   : Does the answer address the question?         │
│    • Context Precision  : Were retrieved chunks actually useful?        │
│    • Context Recall     : Did we retrieve all needed information?       │
│    Target: Faithfulness > 0.85, Relevance > 0.80 before promotion      │
│                                                                          │
│  Production Monitoring:                                                  │
│    • Log every query + response + retrieved chunks to Log Analytics     │
│    • Weekly automated eval run against golden dataset                   │
│    • Alert if faithfulness drops >5% week-over-week (model drift)      │
│    • Human review queue for flagged/escalated responses                 │
│                                                                          │
│  Model Version Management:                                               │
│    • Keep prior model deployment live (traffic weight = 0, not deleted) │
│    • A/B test new prompts: 10% traffic for 48h → promote if metrics OK  │
│    • Rollback in <5 minutes: just shift traffic weight in APIM          │
└─────────────────────────────────────────────────────────────────────────┘
```

```python
# RAGAS evaluation pipeline (run in PromptFlow / CI)
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_precision

def run_rag_evaluation(test_dataset: list[dict]) -> dict:
    """
    Evaluate RAG quality against golden dataset.
    test_dataset: [{question, answer, contexts, ground_truth}, ...]
    Run this in CI on every prompt change and weekly in production.
    """
    results = evaluate(
        dataset=test_dataset,
        metrics=[faithfulness, answer_relevancy, context_precision],
        llm=AzureChatOpenAI(deployment_name="gpt-4o"),
        embeddings=AzureOpenAIEmbeddings(deployment="text-embedding-3-large")
    )

    # Alert if below threshold
    if results["faithfulness"] < 0.85:
        send_alert(
            severity="HIGH",
            message=f"RAG faithfulness degraded: {results['faithfulness']:.2f} < 0.85"
        )

    return results
```

### 8.5 Runtime Governance — Every Request

```
Request Flow with Governance Controls:
                                          
User Input
    │
    ▼
[1] PII Detection (Azure Content Safety)
    │  → If PII found: redact or block depending on policy
    ▼
[2] Prompt Injection Detection
    │  → Pattern matching + Content Safety classifier
    │  → Block if injection detected, log attempt
    ▼
[3] Rate Limiting (APIM)
    │  → Per-user: 20 requests/minute
    │  → Per-tenant: 200 requests/minute
    ▼
[4] Agent Execution (with tool calls)
    │
    ▼
[5] Output Safety Check (Azure Content Safety)
    │  → Filter harmful content, re-check for PII leakage
    ▼
[6] Confidence Gate
    │  → If agent confidence < 0.70: add disclaimer + offer human review
    ▼
[7] Audit Log (immutable, Log Analytics)
    │  → Log: user_id, session_id, query_hash, response_hash,
    │    retrieved_chunk_ids, latency_ms, confidence_score, timestamp
    │  → WORM storage: cannot be modified or deleted (FINTRAC requirement)
    ▼
Response to User (with citations and confidence indicator)
```

### 8.6 Risk Tiering for AI Use Cases

| Risk Tier | Example Use Case | Required Controls |
|-----------|-----------------|-------------------|
| **Tier 1 — High Risk** | Credit decisions, fraud flags, compliance determinations | Human-in-loop mandatory, full explainability, OSFI model validation |
| **Tier 2 — Medium Risk** | Internal Q&A, meeting summaries, research assist | Content safety + audit log + confidence threshold |
| **Tier 3 — Low Risk** | Internal search, knowledge retrieval, training content | Basic content safety + audit log |

---

## 9. Security & Compliance Architecture

### 9.1 Zero Trust Network Design

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          ZERO TRUST PRINCIPLES                          │
│  • Never trust, always verify                                            │
│  • Assume breach                                                         │
│  • Least privilege access                                                │
└─────────────────────────────────────────────────────────────────────────┘

Network controls:
  ✓ ALL services behind Private Endpoints (no public Blob/OpenAI/Search endpoints)
  ✓ Azure Firewall Premium (L7 inspection, IDPS) in hub VNet
  ✓ NSG on every subnet with explicit deny-all default
  ✓ Azure Bastion for operator access (no SSH/RDP exposed to internet)
  ✓ DDoS Protection Standard on public-facing endpoints

Identity controls:
  ✓ Workload Identity for all AKS pods (no stored secrets)
  ✓ Managed Identity for all Azure services (no service principal secrets)
  ✓ Entra ID with MFA for all human access
  ✓ PIM (Privileged Identity Management) for admin roles — JIT access only
  ✓ Conditional Access: require compliant device + MFA for AI admin roles

Data controls:
  ✓ Customer-managed keys (CMK) via Key Vault for storage encryption
  ✓ Azure Purview for data classification and lineage
  ✓ Blob immutability policies (WORM) for audit logs
  ✓ Private DNS zones linked to all hub/spoke VNets
  ✓ TLS 1.3 enforced for all in-transit data
```

### 9.2 Terraform: Security Baseline

```hcl
# Key Vault with Private Endpoint and RBAC (no access policies)
resource "azurerm_key_vault" "ai_platform" {
  name                       = "kv-ai-platform-prod"
  location                   = "canadacentral"
  resource_group_name        = azurerm_resource_group.ai_platform.name
  tenant_id                  = data.azurerm_client_config.current.tenant_id
  sku_name                   = "premium"
  enable_rbac_authorization  = true
  purge_protection_enabled   = true   # Cannot be deleted for 90 days after soft-delete
  soft_delete_retention_days = 90
  public_network_access_enabled = false  # Private endpoint only

  network_acls {
    default_action = "Deny"
    bypass         = "AzureServices"
  }
}

# RBAC: only specific identities can read secrets
resource "azurerm_role_assignment" "aks_kv_reader" {
  scope                = azurerm_key_vault.ai_platform.id
  role_definition_name = "Key Vault Secrets User"
  principal_id         = azurerm_user_assigned_identity.aks_workload.principal_id
}

# Storage Account: no public access, CMK encryption
resource "azurerm_storage_account" "ai_knowledge_base" {
  name                     = "stgaiknowledgebase"
  resource_group_name      = azurerm_resource_group.ai_platform.name
  location                 = "canadacentral"
  account_tier             = "Standard"
  account_replication_type = "ZRS"  # Zone-redundant within Canada Central
  
  public_network_access_enabled   = false
  allow_nested_items_to_be_public = false
  min_tls_version                 = "TLS1_3"
  
  blob_properties {
    versioning_enabled       = true
    change_feed_enabled      = true  # For audit and recovery
    last_access_time_enabled = true  # Required for lifecycle tiering
  }

  customer_managed_key {
    key_vault_key_id          = azurerm_key_vault_key.storage_cmk.id
    user_assigned_identity_id = azurerm_user_assigned_identity.storage_encryption.id
  }
}
```

---

## 10. Cost Model

### 10.1 Migration Cost Estimate

| Component | Small (<10TB) | Medium (10–500TB) | Large (>500TB) |
|-----------|--------------|------------------|---------------|
| Azure Data Box | N/A | $320 × N devices | $320 × N + Data Box Heavy |
| ExpressRoute (1Gbps, 12mo) | N/A | ~$2,500/month | ~$7,000/month (10Gbps) |
| AzCopy transfer | Free tool | Free tool | Free tool |
| Storage (landing/raw) | ~$100/month | ~$2,000/month | ~$20,000/month |
| Video Indexer (processing) | ~$0.015/min × hours | Per-minute pricing | Commitment tier |
| Speech API (batch) | ~$0.30/hr audio | ~$0.30/hr audio | Volume discount |

### 10.2 Ongoing AI Platform Cost (Monthly)

| Component | Small | Medium | Large |
|-----------|-------|--------|-------|
| Azure OpenAI (GPT-4o, consumption) | $500–$2K | $5K–$20K | PTU: $30K–$100K |
| Azure AI Search (S2) | $250/replica | $1,000+ | $5,000+ (Storage Opt.) |
| AKS (4–16 nodes) | $400 | $2,000 | $10,000+ |
| CosmosDB (autoscale) | $100 | $500 | $3,000+ |
| Blob Storage (ongoing) | $50 | $500 | $5,000+ |
| Redis Cache (C2) | $200 | $500 | $2,000+ |
| **Total estimate** | **~$1.5K/month** | **~$10K/month** | **~$55K+/month** |

### 10.3 Cost Optimization Strategies

```
1. Semantic Cache (Redis):
   Cache embedding queries with TTL — expect 30–40% OpenAI cost reduction
   
2. Embedding model selection:
   text-embedding-ada-002 ($0.0001/1K tokens) vs text-embedding-3-large ($0.00013/1K)
   For 100M chunks: ada-002 saves ~$3K upfront; quality trade-off acceptable for short text

3. Storage tiering (lifecycle policies):
   Move originals to Cool after 30d → Archive after 180d
   At PB scale: saves $500K+/year vs keeping everything on Hot

4. PTU (Provisioned Throughput Units) for OpenAI:
   At >100K tokens/day sustained: PTU is 30–50% cheaper than consumption
   Requires 1-month commitment minimum

5. Spot node pools in AKS:
   Use Spot for batch enrichment jobs (Video Indexer AKS workers): 70% cost savings
   Keep system/user pools on regular nodes for agent serving

6. AI Search: optimize index size:
   Remove redundant fields, compress vectors (quantization if available)
   Avoid storing full original text if it's already in Blob
```

---

## 11. Interview Q&A Bank

### System Design

**Q: "Design an AI system that can answer questions about 5 years of board meeting recordings"**

```
Step 1 — Clarify: How many hours of video? (e.g., 2,000 hours)
          What latency target? (<3 seconds for response)
          Who uses this? (executives, compliance team, auditors)
          Regulatory context? (OSFI B-10, FINTRAC audit trail needed)

Step 2 — Architecture:
  Migration: AzCopy/Data Box → Blob Storage (validated zone)
  Processing: Azure Video Indexer → timestamped transcripts
  Indexing: 512-token chunks → ada-002 embeddings → AI Search (hybrid)
  Serving: Semantic Kernel agent → GPT-4o with citation enforcement
  
Step 3 — Key design decisions:
  Hybrid search (not pure vector): exact OSFI terminology + semantic meaning
  Timestamps preserved: every answer links to specific video moment (audit-ready)
  Confidence threshold: <0.70 escalates to human review
  WORM audit logs: every query + response logged immutably (FINTRAC)

Step 4 — Scale numbers:
  2,000 hours × 60 min × ~130 tokens/min ≈ 15.6M tokens total text
  At 512 tokens/chunk ≈ 30,000 chunks
  AI Search S2: handles this comfortably at <$500/month
  
Step 5 — What breaks first at scale:
  Video Indexer rate limits → batch queue + KEDA scaling
  Embedding calls → cache frequent queries in Redis
  AI Search RU limits → increase replicas
```

**Q: "How do you prevent the agent from hallucinating about compliance rules?"**

```
Four-layer defense:
  1. Retrieval grounding: answer only from retrieved chunks, not model memory
     → System prompt: "If not in context, say 'I don't know'"
  2. Citation enforcement: require source + timestamp for every factual claim
     → Structured output: { answer, citations: [{asset, timestamp, text}] }
  3. Confidence gate: if retrieval score < 0.70, add disclaimer + escalate
  4. RAGAS eval: weekly automated faithfulness check against golden dataset
     → Alert and rollback if faithfulness < 0.85
```

**Q: "How do you handle a video in French for a Quebec-based financial institution?"**

```
  Azure Video Indexer: auto-language detection (supports fr-CA)
  Azure AI Speech: locale = "fr-CA" for batch transcription
  AI Search: language analyzer per-field (fr.microsoft for French text)
  OpenAI: GPT-4o handles French natively
  PII detection: Azure Language service supports French PII recognition
  Consideration: PIPEDA and provincial privacy law (Law 25 in Quebec)
                 → ensure data residency stays in Canada East/Central
```

---

## Architecture Diagram (Text Description for Whiteboard)

```
LEFT SIDE — Migration:
  [On-Prem NAS/SAN] 
    → [ExpressRoute 10Gbps] or [Data Box (>40TB)]
    → [Blob Storage — Landing Zone]
    → [Event Grid] triggers [Service Bus Queue]
    → [AKS Workers w/ KEDA] processes:
        Video → [Video Indexer] → transcript
        Audio → [AI Speech Batch] → transcript
    → Chunks → [OpenAI Embeddings] → [AI Search Index]
    → Metadata → [CosmosDB]

RIGHT SIDE — AI Platform:
  [User / Client App]
    → [APIM] (auth, rate limit, audit)
    → [Orchestrator Agent — GPT-4o on AKS]
        ↔ [Video Search Plugin] → [AI Search]
        ↔ [Compliance Plugin] → [CosmosDB rules]
        ↔ [Audit Logger] → [Log Analytics WORM]
    → [Redis Semantic Cache] (bypass if cache hit)
    → Response with citations and video timestamps

BOTTOM — Governance:
  [Content Safety] wraps all inputs/outputs
  [Azure Policy] enforces no-public-access on all storage
  [Purview] for data classification and lineage
  [Key Vault] holds all secrets and CMK
  [Log Analytics] stores immutable audit trail (7 years)
```

---

*Document version: 1.0 | Created: 2026-05-20 | Author: Abhishek Sinha*  
*Repo: https://github.com/AbhishekSinha02/AzureArchitecture*  
*Next: Add Terraform modules for full deployment, add PromptFlow eval pipeline, add Teams Bot integration pattern*
