# Azure AI Foundry (AI Studio)
### 🟡 Intermediate → 🔴 Advanced

> *"AI Foundry is where models become products.  
>  The hub is your AI platform team. The project is your product team."*

---

## Q1. How do you set up Azure AI Foundry Hub and Projects for a team?

**Scenario:** Your organisation has 3 AI teams: fraud detection, customer service chatbot, and document processing. Each needs isolated deployments and budgets, but shared infrastructure (Azure OpenAI quota, AI Search, Key Vault).

```
AI Foundry hierarchy:

  AI Hub (shared platform layer)
  ├── Shared resources:
  │   ├── Azure OpenAI connections
  │   ├── AI Search connections
  │   ├── Key Vault (shared secrets)
  │   └── Storage Account (shared datasets)
  │
  ├── Project: fraud-detection
  │   ├── Own deployments, own budget
  │   ├── Own compute cluster
  │   └── Team: data-science-team (Contributor)
  │
  ├── Project: chatbot-customer-service
  │   ├── Uses hub's OpenAI connection
  │   └── Team: chatbot-team (Contributor)
  │
  └── Project: document-processing
      ├── Uses hub's Document Intelligence connection
      └── Team: document-team (Contributor)

  Hub Admin: AI platform team (Owner)
  Each project team: cannot see other projects' data
```

```bash
# Create AI Foundry Hub
az ml workspace create \
  --name hub-ai-prod \
  --resource-group rg-ai \
  --location canadaeast \
  --kind hub \                        # hub = AI Foundry Hub
  --storage-account adlsprodcc \
  --key-vault kv-prod \
  --container-registry acr-prod

# Create Project under the Hub
az ml workspace create \
  --name proj-chatbot \
  --resource-group rg-ai \
  --kind project \
  --hub-id "/subscriptions/<SUB>/resourceGroups/rg-ai/providers/Microsoft.MachineLearningServices/workspaces/hub-ai-prod"

# Add shared Azure OpenAI connection to Hub (available to all projects)
az ml connection create \
  --name conn-aoai-prod \
  --workspace-name hub-ai-prod \
  --resource-group rg-ai \
  --file - <<EOF
name: conn-aoai-prod
type: azure_open_ai
endpoint: https://aoai-prod-cc.openai.azure.com/
api_key: $(az cognitiveservices account keys list --name aoai-prod-cc -g rg-ai --query key1 -o tsv)
api_version: "2024-10-21"
EOF

# Grant project team access (Contributor = can use resources, not delete hub)
az role assignment create \
  --role "AzureML Data Scientist" \
  --assignee "chatbot-team@company.com" \
  --scope "/subscriptions/<SUB>/resourceGroups/rg-ai/providers/Microsoft.MachineLearningServices/workspaces/proj-chatbot"
```

> ⚠️ **Gotcha:** Azure OpenAI quota is shared at the subscription level, not the Hub level. If your fraud detection project sends 1M tokens/min, it consumes quota that starves the chatbot project. Use separate Azure OpenAI resources per project (or per environment: dev/staging/prod) and separate quota allocations to prevent cross-project throttling.

---

## Q2. How do you fine-tune an Azure OpenAI model in AI Foundry?

**Scenario:** The standard GPT-4o model gives inconsistent output format for your financial report generation task. Despite detailed system prompts, 20% of responses don't follow the required JSON schema. Fine-tuning with 500 examples could fix this.

```
Fine-tuning — when to use vs prompt engineering:

  Use PROMPT ENGINEERING first if:
  ├── < 100 examples available
  ├── Task is general reasoning / Q&A
  └── Budget < $500/month for training

  Use FINE-TUNING when:
  ├── > 200 high-quality examples
  ├── Consistent output FORMAT needed (JSON, specific schema)
  ├── Tone/style consistency across 1000s of outputs
  └── Prompt engineering + few-shot is still inconsistent

  Fine-tuning workflow:
  Prepare JSONL training file (prompt/completion pairs)
       │ min 10 examples, recommended 200+
       ▼
  Upload to Azure OpenAI
       ▼
  Submit fine-tuning job (gpt-4o-mini base recommended — cheaper)
       │ Duration: 30min–4hrs depending on dataset size
       ▼
  Custom model deployment
       ▼
  Evaluate on holdout set vs base model
```

```python
# Step 1 — Prepare training data (JSONL format)
# Each line: {"messages": [system, user, assistant]} — ideal conversation
import jsonlines

training_examples = [
    {
        "messages": [
            {"role": "system", "content": "Extract financial metrics from text. Return valid JSON only."},
            {"role": "user", "content": "Q3 2024: Revenue $45.2M, Net Income $8.1M, EPS $0.32"},
            {"role": "assistant", "content": '{"revenue_millions": 45.2, "net_income_millions": 8.1, "eps": 0.32, "period": "Q3 2024"}'}
        ]
    },
    # ... 499 more examples
]

with jsonlines.open("training_data.jsonl", "w") as writer:
    writer.write_all(training_examples)

# Step 2 — Upload training file
file_response = client.files.create(
    file=open("training_data.jsonl", "rb"),
    purpose="fine-tune"
)
training_file_id = file_response.id
print(f"Training file ID: {training_file_id}")

# Step 3 — Create fine-tuning job
job = client.fine_tuning.jobs.create(
    training_file=training_file_id,
    model="gpt-4o-mini",           # Base model to fine-tune
    hyperparameters={
        "n_epochs": 3,             # 3 passes through training data
        "batch_size": 8,
        "learning_rate_multiplier": 0.1
    }
)
print(f"Fine-tuning job ID: {job.id}")

# Step 4 — Monitor job status
while True:
    job = client.fine_tuning.jobs.retrieve(job.id)
    print(f"Status: {job.status}, Trained tokens: {job.trained_tokens}")
    if job.status in ("succeeded", "failed", "cancelled"):
        break
    time.sleep(60)

# Step 5 — Deploy the fine-tuned model
if job.status == "succeeded":
    print(f"Fine-tuned model ID: {job.fine_tuned_model}")
    # Deploy in Azure Portal or via REST API
    # Use the fine-tuned model ID as the deployment base model
```

```bash
# Deploy fine-tuned model via CLI
az cognitiveservices account deployment create \
  --name aoai-prod-cc \
  --resource-group rg-ai \
  --deployment-name financial-extractor-ft \
  --model-name <fine_tuned_model_id> \
  --model-format OpenAI \
  --sku-name Standard \
  --sku-capacity 10
```

> ⚠️ **Gotcha:** Fine-tuned models are billed separately from base models — and at a higher rate (fine-tuned GPT-4o-mini = ~3× the base model cost per token). Run a cost analysis: fine-tuning reduces token waste from long few-shot examples in every prompt, but the higher per-token rate may offset savings. Fine-tuning is most cost-effective when it replaces 10+ shot examples with a 0-shot request.

---

## Q3. How do you build and evaluate a prompt chain with PromptFlow?

**Scenario:** Your RAG pipeline has 5 steps: embed query → retrieve → rerank → generate → validate. You want to: (1) visually see the flow, (2) run batch evaluation against 100 test cases, (3) compare two versions of the retrieval step.

```
PromptFlow — visual pipeline for LLM chains:

  [Input: user_question]
       │
       ▼
  [Python Node: embed_query]       → calls OpenAI embeddings API
       │ output: query_vector
       ▼
  [Python Node: retrieve_chunks]   → calls AI Search
       │ output: chunks (list)
       ▼
  [LLM Node: generate_answer]      → calls GPT-4o with context
       │ inputs: question + chunks
       │ output: answer_text
       ▼
  [Python Node: validate_groundedness] → calls Content Safety API
       │ output: grounded (bool)
       ▼
  [Output: answer + sources + grounded]

  Batch evaluation: run 100 test cases through the flow
  Compare: flow-v1 (chunk_size=512) vs flow-v2 (chunk_size=1024)
  Metrics: faithfulness, answer_relevancy, latency_p95
```

```bash
# Install PromptFlow CLI
pip install promptflow promptflow-tools

# Create a new flow
pf flow init --flow rag-pipeline --type standard

# Run the flow locally (single test)
pf flow test --flow rag-pipeline \
  --inputs question="What is the OSFI capital requirement?"

# Batch evaluation against test dataset
pf run create \
  --flow rag-pipeline \
  --data evaluation/golden-set.jsonl \
  --column-mapping question='${data.question}' \
  --name "rag-v1-eval-$(date +%Y%m%d)"

# Compare two runs
pf run visualize --name "rag-v1-eval-20240115,rag-v2-eval-20240115"

# Submit batch run to AI Foundry (cloud execution, scalable)
pf run create \
  --flow rag-pipeline \
  --data evaluation/golden-set.jsonl \
  --run-name "rag-v2-eval-cloud" \
  --environment-variables AZURE_OPENAI_KEY=$AOAI_KEY \
  --stream   # Show live output
```

```yaml
# flow.dag.yaml — PromptFlow pipeline definition
inputs:
  question:
    type: string

outputs:
  answer:
    type: string
    reference: ${generate_answer.output.answer}
  grounded:
    type: bool
    reference: ${validate.output.grounded}

nodes:
- name: embed_query
  type: python
  source:
    type: code
    path: embed_query.py
  inputs:
    question: ${inputs.question}

- name: retrieve_chunks
  type: python
  source:
    type: code
    path: retrieve.py
  inputs:
    query_vector: ${embed_query.output}
    top_k: 5

- name: generate_answer
  type: llm
  source:
    type: code
    path: answer.jinja2    # Jinja2 template for the prompt
  inputs:
    question: ${inputs.question}
    context: ${retrieve_chunks.output}
  connection: conn-aoai-prod
  api: chat
  model: gpt-4o-prod
```

> ⚠️ **Gotcha:** PromptFlow batch evaluation runs each test case sequentially by default — 100 test cases × 3s per case = 5 minutes minimum. For fast iteration, set `--max-lines 20` during development. For full CI/CD gate evaluation, use `--stream` with cloud compute and set a time budget — an eval run that blocks a PR for 30 minutes kills developer velocity.

---

## Q4. How do you deploy and serve a model endpoint from AI Foundry?

**Scenario:** Your fine-tuned model and RAG flow are ready. You need to expose them as a secure REST API that your AKS payment portal can call, with authentication, rate limiting, and monitoring.

```
AI Foundry endpoint deployment:

  PromptFlow / Model
       │
       ▼
  Managed Online Endpoint (AI Foundry)
  ├── Deployment: blue (current production)
  ├── Deployment: green (new version — 10% traffic for canary)
  └── Authentication: Entra ID token OR API key
       │
       ▼
  AKS app calls endpoint via:
  POST https://rag-endpoint.canadaeast.inference.ml.azure.com/score
  Authorization: Bearer <token>
       │
       ▼
  Azure Monitor: latency, throughput, errors
  Application Insights: request tracing
```

```bash
# Deploy PromptFlow as a managed online endpoint
az ml online-endpoint create \
  --name rag-endpoint \
  --workspace-name proj-chatbot \
  --resource-group rg-ai \
  --auth-mode key            # key or aml_token or aad_token

# Create deployment (blue — production traffic)
az ml online-deployment create \
  --endpoint-name rag-endpoint \
  --name blue \
  --workspace-name proj-chatbot \
  --resource-group rg-ai \
  --flow-path ./rag-pipeline/ \   # Path to PromptFlow flow directory
  --instance-type Standard_DS3_v2 \
  --instance-count 2 \
  --traffic 100               # 100% traffic to this deployment

# Test the endpoint
az ml online-endpoint invoke \
  --name rag-endpoint \
  --workspace-name proj-chatbot \
  --resource-group rg-ai \
  --request-file test_input.json

# Get endpoint URL and key
az ml online-endpoint show \
  --name rag-endpoint \
  --workspace-name proj-chatbot \
  --resource-group rg-ai \
  --query "{url:scoringUri, key:''}"

az ml online-endpoint get-credentials \
  --name rag-endpoint \
  --workspace-name proj-chatbot \
  --resource-group rg-ai
```

> ⚠️ **Gotcha:** Managed Online Endpoints have a cold start on first request after idle — the container may not be running. Set `--min-instance-count 1` (not 0) to keep at least one instance warm. For production SLAs, use at least 2 instances across availability zones. Scale-to-zero saves cost but adds 30–60 seconds latency on first request.
