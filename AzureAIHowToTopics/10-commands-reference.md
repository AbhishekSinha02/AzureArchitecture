# Azure AI Commands Reference
### All az / Python SDK / REST API / KQL commands with context

> *"Know the command. Know why you're running it. Know what the output means."*

---

## Azure OpenAI — Resource & Deployment Management

```bash
# Create Azure OpenAI resource
az cognitiveservices account create \
  --name aoai-prod-cc \
  --resource-group rg-ai \
  --location canadaeast \
  --kind OpenAI \
  --sku S0 \
  --custom-domain aoai-prod-cc

# List available models in a region
az cognitiveservices account list-models \
  --name aoai-prod-cc \
  --resource-group rg-ai \
  --output table

# Create model deployment (Global Standard — consumption)
az cognitiveservices account deployment create \
  --name aoai-prod-cc \
  --resource-group rg-ai \
  --deployment-name gpt-4o-prod \
  --model-name gpt-4o \
  --model-version "2024-11-20" \
  --model-format OpenAI \
  --sku-name GlobalStandard \
  --sku-capacity 100    # 100K TPM

# List deployments
az cognitiveservices account deployment list \
  --name aoai-prod-cc \
  --resource-group rg-ai \
  --output table

# Show deployment details (quota, model version)
az cognitiveservices account deployment show \
  --name aoai-prod-cc \
  --resource-group rg-ai \
  --deployment-name gpt-4o-prod

# Delete deployment (frees quota)
az cognitiveservices account deployment delete \
  --name aoai-prod-cc \
  --resource-group rg-ai \
  --deployment-name gpt-4o-old

# Get access keys
az cognitiveservices account keys list \
  --name aoai-prod-cc \
  --resource-group rg-ai

# Regenerate key (rotation)
az cognitiveservices account keys regenerate \
  --name aoai-prod-cc \
  --resource-group rg-ai \
  --key-name key1

# Set Private Endpoint (disable public access)
az cognitiveservices account update \
  --name aoai-prod-cc \
  --resource-group rg-ai \
  --custom-domain aoai-prod-cc \
  --public-network-access Disabled
```

---

## Azure OpenAI — Python SDK

```python
from openai import AzureOpenAI
import os

client = AzureOpenAI(
    azure_endpoint=os.getenv("AZURE_OPENAI_ENDPOINT"),
    api_key=os.getenv("AZURE_OPENAI_KEY"),
    api_version="2024-10-21"     # Always pin the version
)

# Chat completion
response = client.chat.completions.create(
    model="gpt-4o-prod",         # Deployment name, not model name
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "Summarise this document: [text]"}
    ],
    temperature=0.1,
    max_tokens=1000,
    seed=42                      # Reproducibility
)
print(response.choices[0].message.content)
print(f"Tokens: {response.usage.prompt_tokens} + {response.usage.completion_tokens}")

# Streaming chat completion
stream = client.chat.completions.create(
    model="gpt-4o-prod",
    messages=[{"role": "user", "content": "Explain quantum computing"}],
    stream=True
)
for chunk in stream:
    if chunk.choices and chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="", flush=True)

# Embeddings
response = client.embeddings.create(
    model="text-embedding-3-small",
    input="The document describes risk management procedures."
)
vector = response.data[0].embedding   # 1536-dim float list
print(f"Embedding dimensions: {len(vector)}")

# Batch embeddings (cost-efficient for many texts)
response = client.embeddings.create(
    model="text-embedding-3-small",
    input=["text one", "text two", "text three"]   # Up to 2048 inputs per call
)
vectors = [item.embedding for item in response.data]

# Function calling (tool use)
response = client.chat.completions.create(
    model="gpt-4o-prod",
    messages=[{"role": "user", "content": "What is the balance on account 12345?"}],
    tools=[{
        "type": "function",
        "function": {
            "name": "get_account_balance",
            "description": "Get account balance",
            "parameters": {
                "type": "object",
                "properties": {"account_id": {"type": "string"}},
                "required": ["account_id"]
            }
        }
    }],
    tool_choice="auto"
)
# Check if model wants to call a tool:
if response.choices[0].finish_reason == "tool_calls":
    tool_call = response.choices[0].message.tool_calls[0]
    print(f"Function: {tool_call.function.name}")
    print(f"Args: {tool_call.function.arguments}")
```

---

## Azure AI Search — Service & Index Management

```bash
# Create AI Search service
az search service create \
  --name srch-prod-cc \
  --resource-group rg-ai \
  --location canadaeast \
  --sku Standard \
  --replica-count 2 \
  --partition-count 1

# Scale search service
az search service update \
  --name srch-prod-cc \
  --resource-group rg-ai \
  --replica-count 3 \
  --partition-count 2

# Get admin key
az search admin-key show \
  --service-name srch-prod-cc \
  --resource-group rg-ai \
  --query primaryKey -o tsv

# Create query key (read-only — use in application code)
az search query-key create \
  --name query-key-app \
  --service-name srch-prod-cc \
  --resource-group rg-ai

# List indexes
az search index list \
  --service-name srch-prod-cc \
  --resource-group rg-ai \
  --output table

# Delete an index
az search index delete \
  --name old-index \
  --service-name srch-prod-cc \
  --resource-group rg-ai

# Run indexer on demand
az search indexer run \
  --name idxr-product-manuals \
  --service-name srch-prod-cc \
  --resource-group rg-ai

# Check indexer status
az search indexer show \
  --name idxr-product-manuals \
  --service-name srch-prod-cc \
  --resource-group rg-ai \
  --query "lastResult.status"
```

---

## Azure AI Search — Python SDK

```python
from azure.search.documents import SearchClient
from azure.search.documents.models import VectorizedQuery, QueryType
from azure.core.credentials import AzureKeyCredential

search_client = SearchClient(
    endpoint=os.getenv("AZURE_SEARCH_ENDPOINT"),
    index_name="policy-docs",
    credential=AzureKeyCredential(os.getenv("AZURE_SEARCH_KEY"))
)

# Keyword search (BM25)
results = search_client.search(
    search_text="OSFI capital requirements",
    top=5,
    select=["id", "content", "source_doc", "@search.score"]
)

# Vector search only
results = search_client.search(
    search_text=None,
    vector_queries=[VectorizedQuery(
        vector=embed("OSFI capital requirements"),
        k_nearest_neighbors=5,
        fields="contentVector"
    )],
    top=5
)

# Hybrid search (BM25 + vector + semantic reranking)
results = search_client.search(
    search_text="OSFI capital requirements",
    vector_queries=[VectorizedQuery(
        vector=embed("OSFI capital requirements"),
        k_nearest_neighbors=50,
        fields="contentVector"
    )],
    query_type=QueryType.SEMANTIC,
    semantic_configuration_name="policy-semantic",
    top=5,
    select=["id", "content", "source_doc"]
)
chunks = list(results)

# Upload documents to index
search_client.upload_documents([
    {
        "id": "doc-001",
        "content": "The maximum exposure limit is $50,000 per counterparty.",
        "source_doc": "policy-031.pdf",
        "page_number": 12,
        "contentVector": embed("The maximum exposure limit is $50,000...")
    }
])

# Delete documents from index
search_client.delete_documents([{"id": "doc-001"}])
```

---

## Azure AI Document Intelligence

```python
from azure.ai.documentintelligence import DocumentIntelligenceClient
from azure.ai.documentintelligence.models import AnalyzeDocumentRequest
from azure.core.credentials import AzureKeyCredential

di_client = DocumentIntelligenceClient(
    endpoint=os.getenv("DOCUMENT_INTELLIGENCE_ENDPOINT"),
    credential=AzureKeyCredential(os.getenv("DOCUMENT_INTELLIGENCE_KEY"))
)

# Analyse with prebuilt invoice model
with open("invoice.pdf", "rb") as f:
    poller = di_client.begin_analyze_document(
        model_id="prebuilt-invoice",
        body=AnalyzeDocumentRequest(bytes_source=f.read()),
        content_type="application/octet-stream"
    )
result = poller.result()

for doc in result.documents:
    vendor = doc.fields.get("VendorName")
    if vendor:
        print(f"Vendor: {vendor.value_string}, Confidence: {vendor.confidence:.2f}")

# Analyse with URL source
poller = di_client.begin_analyze_document(
    model_id="prebuilt-layout",
    body=AnalyzeDocumentRequest(url_source="https://example.com/doc.pdf")
)

# List custom models
poller = di_client.begin_analyze_document(model_id="claims-form-v2", body=...)
```

```bash
# Create Document Intelligence resource
az cognitiveservices account create \
  --name docint-prod-cc \
  --resource-group rg-ai \
  --location canadaeast \
  --kind FormRecognizer \
  --sku S0

# List custom models
az rest --method GET \
  --url "https://docint-prod-cc.cognitiveservices.azure.com/documentintelligence/documentModels?api-version=2024-07-31-preview" \
  --headers "Ocp-Apim-Subscription-Key=$(az cognitiveservices account keys list --name docint-prod-cc -g rg-ai --query key1 -o tsv)"
```

---

## Azure AI Content Safety

```python
from azure.ai.contentsafety import ContentSafetyClient
from azure.ai.contentsafety.models import AnalyzeTextOptions, TextCategory
from azure.core.credentials import AzureKeyCredential

safety_client = ContentSafetyClient(
    endpoint=os.getenv("CONTENT_SAFETY_ENDPOINT"),
    credential=AzureKeyCredential(os.getenv("CONTENT_SAFETY_KEY"))
)

# Analyse text for harmful content
response = safety_client.analyze_text(AnalyzeTextOptions(
    text="Your input text here",
    categories=[TextCategory.HATE, TextCategory.VIOLENCE,
                TextCategory.SEXUAL, TextCategory.SELF_HARM],
    output_type="FourSeverityLevels"
))

for category in response.categories_analysis:
    print(f"{category.category}: severity={category.severity}")
# Severity: 0=safe, 2=low, 4=medium, 6=high
```

```bash
# Create Content Safety resource
az cognitiveservices account create \
  --name content-safety-prod \
  --resource-group rg-ai \
  --location canadaeast \
  --kind ContentSafety \
  --sku S0
```

---

## Azure AI Language (PII Detection)

```python
from azure.ai.textanalytics import TextAnalyticsClient
from azure.core.credentials import AzureKeyCredential

lang_client = TextAnalyticsClient(
    endpoint=os.getenv("AZURE_LANGUAGE_ENDPOINT"),
    credential=AzureKeyCredential(os.getenv("AZURE_LANGUAGE_KEY"))
)

# Detect PII entities
response = lang_client.recognize_pii_entities(
    documents=[{"id": "1", "language": "en", "text": "John Smith SIN 123-456-789"}],
    categories_filter=["Person", "CASocialInsuranceNumber", "CreditCardNumber"]
)

for doc in response:
    for entity in doc.entities:
        print(f"{entity.text}: {entity.category} (confidence: {entity.confidence_score:.2f})")
```

---

## AI Foundry (Azure ML Workspace)

```bash
# Create AI Foundry Hub
az ml workspace create \
  --name hub-ai-prod \
  --resource-group rg-ai \
  --location canadaeast \
  --kind hub

# Create Project under Hub
az ml workspace create \
  --name proj-chatbot \
  --resource-group rg-ai \
  --kind project \
  --hub-id "/subscriptions/<SUB>/resourceGroups/rg-ai/providers/Microsoft.MachineLearningServices/workspaces/hub-ai-prod"

# Deploy PromptFlow as online endpoint
az ml online-endpoint create \
  --name rag-endpoint \
  --workspace-name proj-chatbot \
  --resource-group rg-ai \
  --auth-mode key

az ml online-deployment create \
  --endpoint-name rag-endpoint \
  --name blue \
  --workspace-name proj-chatbot \
  --resource-group rg-ai \
  --flow-path ./rag-pipeline/ \
  --instance-type Standard_DS3_v2 \
  --instance-count 2

# Test endpoint
az ml online-endpoint invoke \
  --name rag-endpoint \
  --workspace-name proj-chatbot \
  --resource-group rg-ai \
  --request-file '{"question": "What is the OSFI capital requirement?"}'

# Get endpoint credentials
az ml online-endpoint get-credentials \
  --name rag-endpoint \
  --workspace-name proj-chatbot \
  --resource-group rg-ai
```

---

## KQL Queries for AI Monitoring

```kusto
// Azure OpenAI: token usage by feature (last 30 days)
AppTraces
| where TimeGenerated > ago(30d)
| where Message == "OpenAI completion"
| extend
    feature           = tostring(Properties["feature"]),
    prompt_tokens     = toint(Properties["prompt_tokens"]),
    completion_tokens = toint(Properties["completion_tokens"]),
    model             = tostring(Properties["model"])
| summarize
    TotalCalls            = count(),
    TotalTokens           = sum(prompt_tokens + completion_tokens),
    EstimatedCostUSD      = (sum(prompt_tokens) / 1000000.0 * 5.0) + (sum(completion_tokens) / 1000000.0 * 15.0)
  by feature, model
| order by EstimatedCostUSD desc

// P95 latency by deployment (alert if > 5s)
AppTraces
| where TimeGenerated > ago(1h)
| where Message == "OpenAI completion"
| extend latency_ms = todouble(Properties["latency_ms"])
| summarize P50=percentile(latency_ms,50), P95=percentile(latency_ms,95), P99=percentile(latency_ms,99)
  by tostring(Properties["model"])
| order by P95 desc

// Content Safety violations by category (security monitoring)
AppTraces
| where TimeGenerated > ago(24h)
| where Message == "Content safety violation"
| extend
    category  = tostring(Properties["category"]),
    severity  = toint(Properties["severity"]),
    context   = tostring(Properties["context"])   // "input" or "output"
| summarize ViolationCount = count() by bin(TimeGenerated, 1h), category, context
| render timechart

// AI audit trail query: find specific decision (compliance investigation)
AppTraces
| where TimeGenerated > ago(548d)   // 18 months back
| where Message == "AI_DECISION_AUDIT"
| extend
    request_id     = tostring(Properties["request_id"]),
    user_id_hash   = tostring(Properties["user_id_hash"]),
    model_version  = tostring(Properties["model_version"]),
    decision       = tostring(Properties["decision"]),
    human_reviewed = tobool(Properties["human_reviewed"]),
    human_override = tobool(Properties["human_override"])
| where user_id_hash == "<hash_of_customer_id>"   // Filter to specific customer
| project TimeGenerated, request_id, model_version, decision, human_reviewed, human_override
| order by TimeGenerated asc

// RAG pipeline: track RAGAS scores over time (drift detection)
AppMetrics
| where TimeGenerated > ago(90d)
| where Name in ("ragas_faithfulness", "ragas_answer_relevancy", "ragas_context_precision")
| summarize AvgScore = avg(Sum/Count) by bin(TimeGenerated, 7d), Name
| render timechart
// Alert if AvgScore drops > 10% week-over-week

// Rate limit errors (429) — helps size PTU commitment
AppDependencies
| where TimeGenerated > ago(7d)
| where Type == "HTTP" and Target contains "openai.azure.com"
| where ResultCode == 429
| summarize ThrottleCount = count() by bin(TimeGenerated, 1h)
| render timechart
// High 429 rate → consider upgrading to PTU
```

---

## Quick Setup Checklist

```bash
# Full AI stack setup sequence (save as setup.sh)

# 1. Azure OpenAI
az cognitiveservices account create --name aoai-prod-cc --resource-group rg-ai --location canadaeast --kind OpenAI --sku S0

# 2. Deploy models
az cognitiveservices account deployment create --name aoai-prod-cc -g rg-ai --deployment-name gpt-4o-prod --model-name gpt-4o --model-version "2024-11-20" --model-format OpenAI --sku-name GlobalStandard --sku-capacity 100
az cognitiveservices account deployment create --name aoai-prod-cc -g rg-ai --deployment-name text-embedding-3-small --model-name text-embedding-3-small --model-version "1" --model-format OpenAI --sku-name GlobalStandard --sku-capacity 350

# 3. AI Search
az search service create --name srch-prod-cc --resource-group rg-ai --location canadaeast --sku Standard --replica-count 2

# 4. Document Intelligence
az cognitiveservices account create --name docint-prod-cc --resource-group rg-ai --location canadaeast --kind FormRecognizer --sku S0

# 5. Content Safety
az cognitiveservices account create --name content-safety-prod --resource-group rg-ai --location canadaeast --kind ContentSafety --sku S0

# 6. Language (PII detection)
az cognitiveservices account create --name lang-prod-cc --resource-group rg-ai --location canadaeast --kind TextAnalytics --sku S

# 7. Log Analytics (Canada East — data residency)
az monitor log-analytics workspace create --workspace-name law-ai-prod --resource-group rg-ai --location canadaeast --retention-in-days 2555

# 8. Store all keys in Key Vault
for service in aoai-prod-cc srch-prod-cc docint-prod-cc content-safety-prod lang-prod-cc; do
  KEY=$(az cognitiveservices account keys list --name $service -g rg-ai --query key1 -o tsv)
  az keyvault secret set --vault-name kv-prod --name "${service}-key" --value "$KEY"
done
```
