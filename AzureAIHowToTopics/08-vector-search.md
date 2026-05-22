# Azure AI Search — Vector & Hybrid Search
### 🟡 Intermediate → 🔴 Advanced

> *"Full-text search finds the word. Vector search finds the meaning.  
>  Hybrid search finds what your user actually wanted."*

---

## Q1. How do you create an Azure AI Search index with vector fields?

**Scenario:** You're building a knowledge base for 50,000 product manuals. Users search with natural language ("how do I reset my device") but manuals use technical terms ("factory reset procedure"). Keyword search misses the connection — you need semantic similarity.

```
Azure AI Search — index with vector fields:

  Document chunk:
  {
    "id": "manual-001-chunk-42",
    "content": "To restore factory settings, press and hold the reset button for 10 seconds.",
    "product_id": "PRD-001",
    "contentVector": [0.0234, -0.1823, 0.0091, ...]  ← 1536-dim float array
  }

  Query: "how do I reset my device"
  query_vector = embed("how do I reset my device")
         → [0.0198, -0.1744, 0.0103, ...]
  Cosine similarity search finds semantically similar chunks
  → "factory reset procedure" document scores high despite no keyword match ✅
```

```bash
# Create AI Search service (Standard tier for vector search)
az search service create \
  --name srch-prod-cc \
  --resource-group rg-ai \
  --location canadaeast \
  --sku Standard \             # Standard2 or higher for large indexes
  --replica-count 2 \          # HA: 2 replicas
  --partition-count 1          # 1 partition = 25GB vector storage

# Create index with vector field via REST (CLI doesn't support vector field config)
az rest --method PUT \
  --url "https://srch-prod-cc.search.windows.net/indexes/product-manuals?api-version=2024-07-01" \
  --headers "api-key=$(az search admin-key show --service-name srch-prod-cc -g rg-ai --query primaryKey -o tsv)" \
             "Content-Type=application/json" \
  --body '{
    "name": "product-manuals",
    "fields": [
      {"name": "id",         "type": "Edm.String", "key": true, "filterable": true},
      {"name": "content",    "type": "Edm.String", "searchable": true, "analyzer": "en.microsoft"},
      {"name": "product_id", "type": "Edm.String", "filterable": true, "facetable": true},
      {"name": "chunk_index","type": "Edm.Int32",  "filterable": true, "sortable": true},
      {
        "name": "contentVector",
        "type": "Collection(Edm.Single)",
        "searchable": true,
        "vectorSearchProfile": "hnsw-cosine",
        "dimensions": 1536
      }
    ],
    "vectorSearch": {
      "algorithms": [{
        "name": "hnsw-algo",
        "kind": "hnsw",
        "hnswParameters": {
          "m": 4,
          "efConstruction": 400,
          "efSearch": 500,
          "metric": "cosine"
        }
      }],
      "profiles": [{
        "name": "hnsw-cosine",
        "algorithm": "hnsw-algo"
      }]
    }
  }'
```

```python
from azure.search.documents import SearchClient
from azure.search.documents.indexes import SearchIndexClient
from azure.core.credentials import AzureKeyCredential

# Index documents with vectors
search_client = SearchClient(
    endpoint=os.getenv("AZURE_SEARCH_ENDPOINT"),
    index_name="product-manuals",
    credential=AzureKeyCredential(os.getenv("AZURE_SEARCH_KEY"))
)

def index_chunks(chunks: list):
    documents = []
    for chunk in chunks:
        vector = embed(chunk["content"])   # 1536-dim float list
        documents.append({
            "id":             chunk["id"],
            "content":        chunk["content"],
            "product_id":     chunk["product_id"],
            "chunk_index":    chunk["chunk_index"],
            "contentVector":  vector
        })
        if len(documents) >= 100:
            search_client.upload_documents(documents)
            documents = []

    if documents:
        search_client.upload_documents(documents)
```

| HNSW Parameter | What it controls | Default | Recommendation |
|---------------|-----------------|---------|----------------|
| `m` | Connections per node (index graph density) | 4 | 4–16; higher = better recall, more memory |
| `efConstruction` | Build-time search depth | 400 | 400–800; higher = better quality, slower indexing |
| `efSearch` | Query-time search depth | 500 | 500–1000; higher = better recall, slower query |
| `metric` | Distance function | cosine | `cosine` for text embeddings |

> ⚠️ **Gotcha:** Vector dimensions must match exactly between the index definition and the embedding model. `text-embedding-3-small` produces 1536 dimensions, `text-embedding-3-large` produces 3072. Indexing 3072-dim vectors into a 1536-dim field silently truncates them, destroying retrieval quality. Verify dimensions before indexing.

---

## Q2. How do you implement hybrid search combining keyword BM25 and vector with RRF?

**Scenario:** Vector search excels at semantic similarity but fails on exact terms (regulation codes, product SKUs). Keyword search excels on exact terms but fails on paraphrasing. You need both.

```
Reciprocal Rank Fusion (RRF) — how it works:

  Query: "OSFI B-10 artificial intelligence model validation"

  BM25 results (keyword):          Vector results (semantic):
  Rank 1: "OSFI B-10 FAQ"         Rank 1: "AI model governance guide"
  Rank 2: "B-10 Appendix"         Rank 2: "Model risk management policy"
  Rank 3: "OSFI press release"    Rank 3: "OSFI B-10 FAQ"  ← also in BM25!
  ...                             ...

  RRF score formula: score = Σ 1/(rank_i + 60)
  "OSFI B-10 FAQ": 1/(1+60) + 1/(3+60) = 0.0164 + 0.0154 = 0.0318  ← highest!
  → "OSFI B-10 FAQ" wins because it appears highly in BOTH result sets ✅

  Result: best of both worlds
  - Exact regulation code "B-10" found via BM25
  - Related content "model governance" found via vector
  - Overlapping result gets RRF boost
```

```python
from azure.search.documents.models import (
    VectorizedQuery, QueryType, QueryLanguage
)

def hybrid_search_with_filter(
    query: str,
    product_filter: str = None,
    top_n: int = 5
) -> list[dict]:

    query_vector = embed(query)

    # Build filter expression (optional)
    filter_expr = f"product_id eq '{product_filter}'" if product_filter else None

    results = search_client.search(
        search_text=query,                    # BM25 leg — keyword search
        filter=filter_expr,                   # Pre-filter by metadata
        vector_queries=[
            VectorizedQuery(
                vector=query_vector,          # Vector leg — semantic search
                k_nearest_neighbors=50,       # Retrieve top 50 candidates for RRF
                fields="contentVector",
                exhaustive=False              # ANN (approximate) — faster than exact KNN
            )
        ],
        query_type=QueryType.SEMANTIC,        # Optional: add semantic re-ranking
        semantic_configuration_name="product-semantic",
        query_language=QueryLanguage.EN_US,
        top=top_n,
        select=["id", "content", "product_id", "chunk_index"],
        scoring_statistics="global"           # Better scoring across shards
    )

    return [
        {
            "id":         r["id"],
            "content":    r["content"],
            "product_id": r["product_id"],
            "rrf_score":  r["@search.score"]   # RRF fused score
        }
        for r in results
    ]
```

```bash
# Add semantic configuration to existing index
az rest --method PATCH \
  --url "https://srch-prod-cc.search.windows.net/indexes/product-manuals?api-version=2024-07-01" \
  --headers "api-key=<KEY>" "Content-Type=application/json" \
  --body '{
    "semantic": {
      "configurations": [{
        "name": "product-semantic",
        "prioritizedFields": {
          "contentFields": [{"fieldName": "content"}],
          "keywordsFields": [{"fieldName": "product_id"}]
        }
      }]
    }
  }'
```

> ⚠️ **Gotcha:** Semantic ranking (`QueryType.SEMANTIC`) is applied AFTER vector search retrieves candidates. If your `k_nearest_neighbors` is too small (e.g., 5), the semantic re-ranker has only 5 candidates to reorder — it cannot surface a relevant document that wasn't in the initial top-5. Set `k_nearest_neighbors` to at least 3–5× your final `top` value to give the re-ranker enough candidates to work with.

---

## Q3. How do you handle large-scale indexing for 100 million documents?

**Scenario:** You're migrating a document search platform from Elasticsearch to Azure AI Search. The corpus is 100M documents, 500GB of text. Bulk indexing naively takes weeks. You need it in 48 hours.

```
Large-scale indexing strategy:

  100M docs → parallelize across:
  - Partitions: Storage + compute for vector index
  - Indexers: multiple ADF pipelines feeding search
  - Batch size: 1,000 docs per upload batch (SDK maximum)

  Throughput estimate:
  - 1 Standard2 service: ~50 doc/s with vectors
  - 100M docs ÷ 50 doc/s = 23 days → too slow
  
  Scale-up strategy:
  1. Use Standard3 tier (10 partitions) → 10× throughput
  2. Run 20 parallel Databricks notebook tasks
  3. Each task: processes 5M docs (shard of ADLS data)
  4. Total: 20 × 500 doc/s = 10,000 doc/s
  5. 100M ÷ 10,000 doc/s = ~2.8 hours ✅
```

```python
from azure.search.documents import SearchClient
from azure.search.documents.models import IndexingResult
import concurrent.futures

# Databricks: parallel indexing across partitions
def index_partition(partition_path: str, partition_id: int):
    """Index one shard of documents. Run as separate Databricks task per shard."""

    client = SearchClient(
        endpoint=os.getenv("AZURE_SEARCH_ENDPOINT"),
        index_name="product-manuals",
        credential=AzureKeyCredential(os.getenv("AZURE_SEARCH_KEY"))
    )

    # Read chunk from ADLS
    df = spark.read.parquet(partition_path)
    records = df.collect()

    batch_size = 1000      # AI Search max batch size
    success_count = 0
    error_count = 0

    for i in range(0, len(records), batch_size):
        batch = records[i:i + batch_size]

        # Embed batch in parallel (OpenAI batch embedding)
        texts = [r["content"] for r in batch]
        embed_response = openai_client.embeddings.create(
            model="text-embedding-3-small",
            input=texts
        )
        vectors = [item.embedding for item in embed_response.data]

        documents = [
            {
                "id":            r["id"],
                "content":       r["content"],
                "product_id":    r["product_id"],
                "contentVector": vectors[j]
            }
            for j, r in enumerate(batch)
        ]

        # Upload batch to AI Search
        results = client.upload_documents(documents)
        success_count += sum(1 for r in results if r.succeeded)
        error_count   += sum(1 for r in results if not r.succeeded)

    print(f"Partition {partition_id}: {success_count} success, {error_count} errors")
    return {"partition_id": partition_id, "success": success_count, "errors": error_count}

# Databricks: run 20 partition tasks in parallel via Databricks Workflow
# Each task calls index_partition with a different ADLS path shard
```

```bash
# Scale AI Search to handle indexing throughput
# Standard3 (S3) with 12 partitions = max indexing throughput
az search service update \
  --name srch-prod-cc \
  --resource-group rg-ai \
  --partition-count 12 \     # 12 partitions = 12 × 25GB = 300GB vector storage
  --replica-count 3          # 3 replicas = HA + read throughput

# After bulk indexing complete: scale back down to save cost
az search service update \
  --name srch-prod-cc \
  --resource-group rg-ai \
  --partition-count 2 \      # 2 partitions sufficient for 100M docs at steady state
  --replica-count 2
```

> ⚠️ **Gotcha:** AI Search charges are based on the **maximum** search units (SU = replicas × partitions) used during any hour — not an average. Scaling up to 12 partitions for 48 hours of bulk indexing costs 12 SU × 48hr. Scale back down immediately after bulk indexing or you pay for idle compute.

---

## Q4. How do you use AI Search Integrated Vectorisation to skip manual embedding?

**Scenario:** Your team wants to add documents without writing Python to embed and upload. Every time a PDF lands in ADLS Gen2, it should automatically appear in the search index — no pipeline code.

```
Integrated vectorisation — zero-code indexing:

  ADLS Gen2: upload PDF
       │ (blob trigger)
       ▼
  AI Search Indexer (scheduled / change-tracked)
       │ reads new/modified blobs
       ▼
  Built-in skillset:
  ├── Document cracking (OCR, PDF text extraction)
  ├── Text split skill (chunking — no custom code)
  └── Azure OpenAI embedding skill (automatic vectorisation)
       │
       ▼
  AI Search index (vectors populated automatically)
  → Searchable within minutes of upload ✅
```

```bash
# Create datasource pointing to ADLS Gen2
az rest --method PUT \
  --url "https://srch-prod-cc.search.windows.net/datasources/ds-product-manuals?api-version=2024-07-01" \
  --headers "api-key=<KEY>" "Content-Type=application/json" \
  --body '{
    "name": "ds-product-manuals",
    "type": "azureblob",
    "credentials": {"connectionString": "ResourceId=/subscriptions/<SUB>/resourceGroups/rg-data/providers/Microsoft.Storage/storageAccounts/adlsprodcc;"},
    "container": {"name": "raw", "query": "manuals/"}
  }'

# Create skillset with integrated vectorisation
az rest --method PUT \
  --url "https://srch-prod-cc.search.windows.net/skillsets/ss-vectorise?api-version=2024-07-01" \
  --headers "api-key=<KEY>" "Content-Type=application/json" \
  --body '{
    "name": "ss-vectorise",
    "skills": [
      {
        "@odata.type": "#Microsoft.Skills.Text.SplitSkill",
        "name": "split",
        "textSplitMode": "pages",
        "maximumPageLength": 2000,
        "inputs": [{"name":"text","source":"/document/content"}],
        "outputs": [{"name":"textItems","targetName":"pages"}]
      },
      {
        "@odata.type": "#Microsoft.Skills.Text.AzureOpenAIEmbeddingSkill",
        "name": "embed",
        "resourceUri": "https://aoai-prod-cc.openai.azure.com",
        "deploymentId": "text-embedding-3-small",
        "modelName": "text-embedding-3-small",
        "inputs": [{"name":"text","source":"/document/pages/*"}],
        "outputs": [{"name":"embedding","targetName":"contentVector"}]
      }
    ],
    "cognitiveServices": {"@odata.type": "#Microsoft.Azure.Search.DefaultCognitiveServices"}
  }'

# Create indexer that runs the skillset
az rest --method PUT \
  --url "https://srch-prod-cc.search.windows.net/indexers/idxr-product-manuals?api-version=2024-07-01" \
  --headers "api-key=<KEY>" "Content-Type=application/json" \
  --body '{
    "name": "idxr-product-manuals",
    "dataSourceName": "ds-product-manuals",
    "targetIndexName": "product-manuals",
    "skillsetName": "ss-vectorise",
    "schedule": {"interval": "PT5M"},
    "parameters": {"configuration": {"dataToExtract": "contentAndMetadata"}}
  }'
```

> ⚠️ **Gotcha:** Integrated vectorisation calls Azure OpenAI embedding API from inside AI Search — billing goes to your Azure OpenAI resource. The indexer processes documents sequentially with built-in throttling. For bulk indexing (>10K docs at initial load), run one explicit indexer trigger after setting a high `batchSize`. Relying solely on the 5-minute scheduled indexer means initial indexing of large corpora takes days.
