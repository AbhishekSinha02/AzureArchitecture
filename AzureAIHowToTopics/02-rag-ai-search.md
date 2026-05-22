# RAG & Azure AI Search
### 🟡 Intermediate → 🔴 Advanced

> *"RAG without good chunking is like a library where every book  
>  has been cut into random paragraphs and reshuffled."*

---

## Q1. How do you build an end-to-end RAG pipeline with Azure OpenAI and AI Search?

**Scenario:** A financial services company has 50,000 policy documents. A compliance chatbot must answer questions grounded in those documents, cite the source, and never make up an answer.

```
RAG pipeline — two phases:

  INDEXING (offline, run once then incrementally):
  PDF/Word docs
       │ Azure Document Intelligence (extract text + structure)
       ▼
  Chunker (512 tokens, 50-token overlap)
       │ produces chunk objects with metadata
       ▼
  Azure OpenAI (text-embedding-3-small)
       │ embed each chunk → 1536-dim vector
       ▼
  Azure AI Search index
       │ stores: chunk text + vector + metadata (doc_id, page, section)

  QUERY (real-time, per user question):
  User question
       │ embed question → query vector
       ▼
  AI Search: hybrid search (BM25 keyword + vector)
       │ top-5 chunks retrieved
       ▼
  GPT-4o: "Answer using ONLY these chunks. Cite the source."
       │ grounded answer + citations
       ▼
  User: "Per Section 4.2 of Policy-031, the limit is $50,000."
```

```python
from openai import AzureOpenAI
from azure.search.documents import SearchClient
from azure.search.documents.models import VectorizedQuery
from azure.core.credentials import AzureKeyCredential

# Clients
openai_client = AzureOpenAI(
    azure_endpoint=os.getenv("AZURE_OPENAI_ENDPOINT"),
    api_key=os.getenv("AZURE_OPENAI_KEY"),
    api_version="2024-10-21"
)

search_client = SearchClient(
    endpoint=os.getenv("AZURE_SEARCH_ENDPOINT"),
    index_name="policy-docs",
    credential=AzureKeyCredential(os.getenv("AZURE_SEARCH_KEY"))
)

def embed(text: str) -> list[float]:
    response = openai_client.embeddings.create(
        model="text-embedding-3-small",
        input=text
    )
    return response.data[0].embedding

def rag_query(user_question: str) -> dict:
    # Step 1: Embed the question
    query_vector = embed(user_question)

    # Step 2: Hybrid search (BM25 + vector)
    results = search_client.search(
        search_text=user_question,              # BM25 keyword search
        vector_queries=[VectorizedQuery(
            vector=query_vector,
            k_nearest_neighbors=10,
            fields="contentVector"              # Vector field name in index
        )],
        query_type="semantic",
        semantic_configuration_name="policy-semantic",
        top=5,
        select=["id", "content", "source_doc", "page_number", "section"]
    )

    # Step 3: Build context from retrieved chunks
    chunks = list(results)
    context = "\n\n".join([
        f"[Source: {c['source_doc']}, Page {c['page_number']}]\n{c['content']}"
        for c in chunks
    ])

    # Step 4: Generate grounded answer
    response = openai_client.chat.completions.create(
        model="gpt-4o-prod",
        messages=[
            {
                "role": "system",
                "content": (
                    "You are a compliance assistant. Answer ONLY from the provided context. "
                    "If the answer is not in the context, say 'I cannot find this in the policy documents.' "
                    "Always cite the source document and page number."
                )
            },
            {
                "role": "user",
                "content": f"Context:\n{context}\n\nQuestion: {user_question}"
            }
        ],
        temperature=0.0,    # Zero temp for factual compliance answers
        max_tokens=1000
    )

    return {
        "answer": response.choices[0].message.content,
        "sources": [{"doc": c["source_doc"], "page": c["page_number"]} for c in chunks],
        "tokens_used": response.usage.total_tokens
    }
```

> ⚠️ **Gotcha:** GPT-4o will hallucinate if the prompt doesn't explicitly forbid it. "Answer ONLY from the provided context" alone is insufficient — add "If the answer is not in the context, say [specific phrase]." Then programmatically check the output for that phrase to catch unanswered questions and route them to a human.

> 💡 **Deep dive hint:** Agentic RAG — instead of a fixed single-retrieval loop, the LLM decides whether to retrieve more documents, reformulate the query, or answer directly. Uses function calling to trigger additional search steps when initial retrieval is insufficient.

---

## Q2. How do you chunk documents optimally for RAG retrieval quality?

**Scenario:** Your RAG chatbot returns irrelevant chunks. Investigation shows that 50-page policy documents are being split randomly every 500 characters — cutting mid-sentence, mid-table, mid-clause. Retrieval quality is poor.

```
Chunking strategy comparison:

  Fixed-size (naive — bad for documents)
  ─────────────────────────────────────
  "The maximum exposure limit is $50,000 per counterpar"  ← sentence cut
  "ty per quarter. This applies to..."                    ← context lost
  → Vector: encodes incomplete thought → poor retrieval

  Sentence-aware with overlap (better)
  ─────────────────────────────────────
  Chunk 1: "Section 4.2: Counterparty Limits. The maximum exposure limit
            is $50,000 per counterparty per quarter."
  Chunk 2 (50-token overlap): "is $50,000 per counterparty per quarter.
            This applies to all tier-2 counterparties..."
  → Complete semantic units → better retrieval ✅

  Hierarchical / parent-child (best for long docs)
  ─────────────────────────────────────────────────
  Parent: full section (2000 tokens) — provides full context to LLM
  Child: individual sentences (100 tokens) — used for retrieval precision
  → Retrieve by child, return parent context to LLM
```

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter
import tiktoken

def chunk_document(text: str, source_metadata: dict) -> list[dict]:
    """
    Chunk using recursive character splitter — respects sentence boundaries.
    Returns list of chunk dicts ready for indexing.
    """
    enc = tiktoken.encoding_for_model("gpt-4o")

    splitter = RecursiveCharacterTextSplitter(
        chunk_size=512,             # Target chunk size in tokens
        chunk_overlap=50,           # Overlap to preserve context at boundaries
        length_function=lambda t: len(enc.encode(t)),   # Count tokens, not chars
        separators=[
            "\n## ", "\n### ",      # Markdown headers — strong break
            "\n\n",                 # Paragraph breaks — medium break
            "\n",                   # Line breaks
            ". ", "! ", "? ",       # Sentence ends — weak break
            " ", ""                 # Word/char breaks — last resort
        ]
    )

    chunks = splitter.split_text(text)

    return [
        {
            "id": f"{source_metadata['doc_id']}-chunk-{i}",
            "content": chunk,
            "source_doc": source_metadata["filename"],
            "page_number": source_metadata.get("page", 0),
            "section": source_metadata.get("section", ""),
            "doc_id": source_metadata["doc_id"],
            "chunk_index": i,
            "total_chunks": len(chunks),
            "token_count": len(enc.encode(chunk))
        }
        for i, chunk in enumerate(chunks)
    ]

# For tables: don't split mid-table — detect and keep whole
def chunk_with_table_preservation(text: str, metadata: dict) -> list[dict]:
    # Split on table boundaries first (markdown tables start with |)
    import re
    table_pattern = r'(\|[^\n]+\|[\s\S]*?)(?=\n[^|]|\Z)'
    sections = re.split(r'(\|[^\n]+\|[\s\S]*?(?=\n[^|]|\Z))', text)
    # Tables become single chunks regardless of size
    chunks = []
    for section in sections:
        if section.strip().startswith("|"):
            chunks.append(section)    # Keep table whole
        else:
            chunks.extend([c.content for c in
                          RecursiveCharacterTextSplitter(chunk_size=512, chunk_overlap=50)
                          .create_documents([section])])
    return chunks
```

> ⚠️ **Gotcha:** Chunk size measured in characters (default in most splitters) ≠ tokens. GPT-4o has a 128K context window measured in tokens. A "512 character" chunk is ~128 tokens. A "512 token" chunk is ~2,000 characters. Always use `tiktoken` to measure chunk size in tokens — passing oversized chunks silently truncates them at the API boundary.

---

## Q3. How do you implement hybrid search combining BM25 keyword + vector with RRF?

**Scenario:** Vector-only search finds semantically similar documents but misses exact terms (regulation code "OSFI-B10-2024-Section-4"). BM25-only misses paraphrased questions. You need both.

```
Hybrid search with Reciprocal Rank Fusion (RRF):

  Query: "What are the AI model validation requirements under OSFI?"

  BM25 (keyword) results:          Vector (semantic) results:
  Rank 1: doc about "OSFI"         Rank 1: doc about "model governance"
  Rank 2: doc about "validation"   Rank 2: doc about "AI risk management"
  Rank 3: doc about "requirements" Rank 3: doc about "algorithmic auditing"

  RRF fusion score = Σ 1/(rank + k) across all result sets
  k=60 (constant to reduce impact of low ranks)

  Fused result:
  Rank 1: "OSFI B-10 model risk management framework" ← appears in BOTH lists
  Rank 2: "AI validation checklist per OSFI guidelines"
  → Better than either search alone ✅
```

```python
from azure.search.documents.models import VectorizedQuery, QueryType

def hybrid_search(query: str, top_n: int = 5) -> list[dict]:
    query_vector = embed(query)

    results = search_client.search(
        search_text=query,                  # BM25 leg
        vector_queries=[
            VectorizedQuery(
                vector=query_vector,
                k_nearest_neighbors=50,     # Retrieve more candidates for RRF
                fields="contentVector",
                exhaustive=False            # Use ANN (faster) not exact KNN
            )
        ],
        query_type=QueryType.SEMANTIC,      # Add semantic re-ranking on top
        semantic_configuration_name="policy-semantic",
        query_language="en-us",
        top=top_n,
        select=["id", "content", "source_doc", "page_number", "@search.score",
                "@search.reranker_score"]
    )

    return [
        {
            "content": r["content"],
            "source": r["source_doc"],
            "page": r["page_number"],
            "bm25_score": r.get("@search.score", 0),
            "semantic_score": r.get("@search.reranker_score", 0)
        }
        for r in results
    ]
```

```bash
# Create AI Search index with vector field
az search index create \
  --service-name srch-prod-cc \
  --resource-group rg-ai \
  --name policy-docs \
  --fields '[
    {"name":"id","type":"Edm.String","key":true,"filterable":true},
    {"name":"content","type":"Edm.String","searchable":true,"analyzer":"en.microsoft"},
    {"name":"source_doc","type":"Edm.String","filterable":true,"facetable":true},
    {"name":"page_number","type":"Edm.Int32","filterable":true},
    {"name":"contentVector","type":"Collection(Edm.Single)","searchable":true,
     "vectorSearchProfile":"hnsw-profile","dimensions":1536}
  ]' \
  --vector-search '{
    "algorithms":[{"name":"hnsw","kind":"hnsw","hnswParameters":{"m":4,"efConstruction":400,"metric":"cosine"}}],
    "profiles":[{"name":"hnsw-profile","algorithm":"hnsw"}]
  }' \
  --semantic-search '{
    "configurations":[{"name":"policy-semantic","prioritizedFields":{"contentFields":[{"fieldName":"content"}]}}]
  }'
```

> ⚠️ **Gotcha:** Semantic ranking (Azure AI Search semantic re-ranker) is billed separately — $1 per 1,000 semantic queries. It adds ~100–200ms latency. Do NOT enable semantic ranking on every query if your index has millions of documents and high QPS. Use it selectively for compliance/regulatory queries where precision matters more than speed.

---

## Q4. How do you handle multi-turn conversation with context in a RAG chatbot?

**Scenario:** User asks "What is the OSFI capital requirement?" — retrieval works. Next message: "How does that compare to Basel III?" — the word "that" refers to the previous answer. RAG retrieves completely unrelated documents because it doesn't know what "that" means.

```
Multi-turn context problem:

  Turn 1: "What is the OSFI capital requirement?"
  → Retrieve + answer: "OSFI requires 10.5% CET1 capital ratio"

  Turn 2: "How does THAT compare to Basel III?"
  ↑ "that" = OSFI CET1 requirement — but the search index doesn't know this

  WITHOUT standalone question rewriting:
  Search query: "How does that compare to Basel III?"
  → "that" finds nothing → poor retrieval → bad answer

  WITH standalone question rewriting:
  History fed to GPT: rewrite question →
  "How does the OSFI CET1 capital ratio requirement compare to Basel III?"
  → Excellent retrieval → accurate comparison ✅
```

```python
from collections import deque

def rewrite_standalone_question(question: str, history: list) -> str:
    """Rewrite question to be self-contained given the conversation history."""
    if not history:
        return question

    history_text = "\n".join([
        f"User: {h['user']}\nAssistant: {h['assistant'][:500]}..."
        for h in history[-3:]    # Last 3 turns only — keep context tight
    ])

    response = openai_client.chat.completions.create(
        model="gpt-4o-prod",
        messages=[
            {
                "role": "system",
                "content": (
                    "Given a conversation history and a follow-up question, "
                    "rewrite the follow-up question to be standalone and self-contained. "
                    "Output ONLY the rewritten question. No explanation."
                )
            },
            {
                "role": "user",
                "content": f"History:\n{history_text}\n\nFollow-up: {question}"
            }
        ],
        temperature=0,
        max_tokens=100
    )
    return response.choices[0].message.content

class RAGChatbot:
    def __init__(self):
        self.history = deque(maxlen=10)   # Keep last 10 turns

    def chat(self, user_message: str) -> dict:
        # Step 1: Rewrite question with context
        standalone_q = rewrite_standalone_question(user_message, list(self.history))

        # Step 2: Retrieve with rewritten question
        chunks = hybrid_search(standalone_q, top_n=5)
        context = "\n\n".join([f"[{c['source']} p.{c['page']}]\n{c['content']}" for c in chunks])

        # Step 3: Build messages with conversation history
        messages = [
            {"role": "system", "content": "You are a compliance assistant. Answer only from context. Cite sources."}
        ]
        for turn in list(self.history)[-3:]:    # Last 3 turns in messages
            messages.append({"role": "user", "content": turn["user"]})
            messages.append({"role": "assistant", "content": turn["assistant"]})

        messages.append({"role": "user", "content": f"Context:\n{context}\n\nQuestion: {user_message}"})

        # Step 4: Generate answer
        response = openai_client.chat.completions.create(
            model="gpt-4o-prod",
            messages=messages,
            temperature=0,
            max_tokens=1000
        )
        answer = response.choices[0].message.content

        # Step 5: Update history
        self.history.append({"user": user_message, "assistant": answer})

        return {"answer": answer, "sources": chunks, "standalone_question": standalone_q}
```

> ⚠️ **Gotcha:** Storing full conversation history in the LLM context grows token consumption linearly. A 20-turn conversation with 1,000 tokens per turn = 20,000 tokens of history in every request. For banking chatbots with strict cost controls, limit history to 3–5 turns and use the standalone question rewrite to preserve context without passing the full history.
