# Azure OpenAI Service
### 🟢 Beginner → 🟡 Intermediate → 🔴 Advanced

> *"Azure OpenAI is not a chatbot. It's an inference engine.  
>  How you deploy it determines whether it's a toy or a platform."*

---

## Q1. How do you deploy Azure OpenAI and choose the right deployment mode?

**Scenario:** You're building a banking document summarisation service. The AI team debates between Pay-As-You-Go (consumption) and Provisioned Throughput Units (PTU). Which do you choose and why?

```
Two deployment modes:

  Global Standard (Pay-As-You-Go / Consumption)
  ──────────────────────────────────────────────
  Billed per token (input + output)
  Throughput: shared — may be throttled (429) at high load
  Latency: variable (depends on Azure's global load)
  Best for: dev, test, variable/unpredictable workloads
  Cost: GPT-4o = $5/1M input tokens, $15/1M output tokens

  Provisioned Throughput Units (PTU)
  ───────────────────────────────────
  Reserved compute — dedicated capacity for your deployment
  Throughput: guaranteed (no 429 throttling)
  Latency: consistent and predictable
  Minimum commitment: 50 PTUs (hourly billing)
  1 PTU for GPT-4o ≈ 2,500 RPM (requests per minute)
  Best for: production, SLA-bound, high-volume
  Cost: ~$2/PTU/hour = $1,440/month for 50 PTUs

  Decision matrix:
  ─────────────────────────────────────────────────────
  < 100K tokens/day    → Global Standard
  100K–2M tokens/day   → Global Standard + retry logic
  > 2M tokens/day      → PTU (economics and reliability)
  Banking SLA required → PTU always (no throttling risk)
```

```bash
# Create Azure OpenAI resource (Canada East has GPT-4o availability)
az cognitiveservices account create \
  --name aoai-prod-cc \
  --resource-group rg-ai \
  --location canadaeast \
  --kind OpenAI \
  --sku S0 \
  --custom-domain aoai-prod-cc    # Required for Private Endpoint

# Deploy GPT-4o model (Global Standard — consumption)
az cognitiveservices account deployment create \
  --name aoai-prod-cc \
  --resource-group rg-ai \
  --deployment-name gpt-4o-prod \
  --model-name gpt-4o \
  --model-version "2024-11-20" \
  --model-format OpenAI \
  --sku-name GlobalStandard \
  --sku-capacity 100    # 100K TPM (tokens per minute) quota

# Deploy text-embedding-ada-002 for RAG embeddings
az cognitiveservices account deployment create \
  --name aoai-prod-cc \
  --resource-group rg-ai \
  --deployment-name text-embedding-3-small \
  --model-name text-embedding-3-small \
  --model-version "1" \
  --model-format OpenAI \
  --sku-name GlobalStandard \
  --sku-capacity 350    # 350K TPM for embedding workloads

# List deployments and check quota
az cognitiveservices account deployment list \
  --name aoai-prod-cc \
  --resource-group rg-ai \
  --output table
```

```python
# Basic GPT-4o chat completion
from openai import AzureOpenAI

client = AzureOpenAI(
    azure_endpoint="https://aoai-prod-cc.openai.azure.com/",
    api_key=os.getenv("AZURE_OPENAI_KEY"),
    api_version="2024-10-21"        # Always pin the API version
)

response = client.chat.completions.create(
    model="gpt-4o-prod",            # Deployment name, not model name
    messages=[
        {"role": "system", "content": "You are a financial document analyst. Be precise and cite sources."},
        {"role": "user", "content": "Summarise the key risks in this document: [document_text]"}
    ],
    temperature=0.1,                # Low temp = more deterministic for document tasks
    max_tokens=1000,
    seed=42                         # Reproducible outputs for testing
)

print(response.choices[0].message.content)
print(f"Tokens used: {response.usage.total_tokens}")
```

> ⚠️ **Gotcha:** Model names and deployment names are different in Azure OpenAI. The `model` parameter in the API call takes the **deployment name** you set (e.g., `gpt-4o-prod`), NOT the model name (`gpt-4o`). Passing the model name directly returns a 404 "model not found" — a confusing error for newcomers.

> 💡 **Deep dive hint:** PTU sizing for banking — how to calculate required PTUs from your peak RPM, average tokens per request, and target utilisation headroom, accounting for burst patterns in financial workloads.

---

## Q2. How do you implement retry logic and handle Azure OpenAI rate limit errors?

**Scenario:** Your production pipeline processes 500 documents per hour through Azure OpenAI. At peak load, 10–15% of requests return HTTP 429 (rate limit exceeded). The pipeline crashes and documents are lost.

```
Rate limit error handling:

  API Request → Azure OpenAI
       │
       ├── 200 OK → process response ✅
       ├── 429 Too Many Requests
       │    → Retry-After header: "30" seconds
       │    → Exponential backoff: 1s → 2s → 4s → 8s → 16s
       │    → Max retries: 5
       │    → After 5 retries: send to dead-letter queue for reprocessing
       │
       └── 503 Service Unavailable
            → Same exponential backoff
            → Different cause: Azure capacity issue (rare)

  Throughput management:
  Semaphore: limit concurrent requests to 80% of TPM capacity
  Token counting: count tokens BEFORE sending to avoid 429
```

```python
import time
import random
import asyncio
from openai import AzureOpenAI, RateLimitError, APIStatusError
import tiktoken

client = AzureOpenAI(
    azure_endpoint=os.getenv("AZURE_OPENAI_ENDPOINT"),
    api_key=os.getenv("AZURE_OPENAI_KEY"),
    api_version="2024-10-21",
    max_retries=0       # Disable SDK retry — we implement our own
)

def count_tokens(text: str, model: str = "gpt-4o") -> int:
    enc = tiktoken.encoding_for_model("gpt-4o")
    return len(enc.encode(text))

def chat_with_retry(messages: list, max_retries: int = 5) -> str:
    base_delay = 1.0

    for attempt in range(max_retries):
        try:
            response = client.chat.completions.create(
                model="gpt-4o-prod",
                messages=messages,
                temperature=0.1,
                max_tokens=2000
            )
            return response.choices[0].message.content

        except RateLimitError as e:
            if attempt == max_retries - 1:
                raise    # Final attempt — let caller handle

            # Read Retry-After from header if present
            retry_after = e.response.headers.get("Retry-After", None)
            if retry_after:
                wait = int(retry_after) + random.uniform(0, 1)
            else:
                # Exponential backoff with jitter
                wait = (base_delay * (2 ** attempt)) + random.uniform(0, 1)

            print(f"Rate limited (attempt {attempt+1}/{max_retries}). Waiting {wait:.1f}s...")
            time.sleep(wait)

        except APIStatusError as e:
            if e.status_code in (500, 503) and attempt < max_retries - 1:
                wait = base_delay * (2 ** attempt)
                time.sleep(wait)
            else:
                raise

# Async batch processing with concurrency control
async def process_documents_batch(documents: list, concurrency: int = 10):
    semaphore = asyncio.Semaphore(concurrency)   # Max 10 concurrent requests

    async def process_one(doc):
        async with semaphore:
            return chat_with_retry([
                {"role": "user", "content": f"Summarise: {doc['text']}"}
            ])

    tasks = [process_one(doc) for doc in documents]
    return await asyncio.gather(*tasks, return_exceptions=True)
```

> ⚠️ **Gotcha:** Azure OpenAI rate limits are measured in **tokens per minute (TPM)**, not requests per minute. A single request with a 4,000-token prompt counts as 4,000 TPM — you can hit the limit with just 25 requests/min if each has a large prompt. Count tokens before sending (`tiktoken`) and implement a token-bucket rate limiter, not just a request-count limiter.

---

## Q3. How do you use function calling to connect GPT-4o to your own APIs?

**Scenario:** A customer support bot must answer account balance queries. The answer is in your core banking API, not in the LLM's training data. The LLM must decide when to call the API and pass the right parameters.

```
Function calling flow:

  User: "What's the balance on my savings account?"
       │
       ▼
  GPT-4o (with tools defined)
       │ "I need to call get_account_balance with account_type=savings"
       │ Returns: finish_reason="tool_calls", tool call JSON
       ▼
  Your application code
       │ Receives tool call → calls real banking API
       │ GET /api/accounts/savings/balance → {"balance": 12450.00}
       ▼
  GPT-4o (second call — with tool result)
       │ "Your savings account balance is $12,450.00 as of today."
       ▼
  User receives accurate, real-time answer ✅
```

```python
from openai import AzureOpenAI
import json

client = AzureOpenAI(
    azure_endpoint=os.getenv("AZURE_OPENAI_ENDPOINT"),
    api_key=os.getenv("AZURE_OPENAI_KEY"),
    api_version="2024-10-21"
)

# Define tools (functions) the model can call
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_account_balance",
            "description": "Get the current balance for a customer account. Use when customer asks about their balance.",
            "parameters": {
                "type": "object",
                "properties": {
                    "customer_id": {
                        "type": "string",
                        "description": "Customer ID from the authenticated session"
                    },
                    "account_type": {
                        "type": "string",
                        "enum": ["chequing", "savings", "investment"],
                        "description": "Type of account"
                    }
                },
                "required": ["customer_id", "account_type"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "get_recent_transactions",
            "description": "Get recent transactions for an account. Use when customer asks about transaction history.",
            "parameters": {
                "type": "object",
                "properties": {
                    "customer_id": {"type": "string"},
                    "account_type": {"type": "string", "enum": ["chequing", "savings"]},
                    "days": {"type": "integer", "description": "Number of days back to fetch", "default": 30}
                },
                "required": ["customer_id", "account_type"]
            }
        }
    }
]

def run_banking_agent(user_message: str, customer_id: str) -> str:
    messages = [
        {"role": "system", "content": "You are a helpful banking assistant. You have access to real-time account data via tools. Never guess balances — always use tools."},
        {"role": "user", "content": user_message}
    ]

    while True:
        response = client.chat.completions.create(
            model="gpt-4o-prod",
            messages=messages,
            tools=tools,
            tool_choice="auto"    # Model decides when to call tools
        )

        choice = response.choices[0]

        # If model wants to call a tool
        if choice.finish_reason == "tool_calls":
            messages.append(choice.message)   # Append assistant message with tool calls

            for tool_call in choice.message.tool_calls:
                fn_name = tool_call.function.name
                fn_args = json.loads(tool_call.function.arguments)
                fn_args["customer_id"] = customer_id   # Inject from session — never from LLM

                # Call the real API
                if fn_name == "get_account_balance":
                    result = banking_api.get_balance(fn_args["account_type"], fn_args["customer_id"])
                elif fn_name == "get_recent_transactions":
                    result = banking_api.get_transactions(fn_args["account_type"], fn_args["customer_id"], fn_args.get("days", 30))
                else:
                    result = {"error": "Unknown function"}

                messages.append({
                    "role": "tool",
                    "tool_call_id": tool_call.id,
                    "content": json.dumps(result)
                })
        else:
            # No more tool calls — return final answer
            return choice.message.content
```

> ⚠️ **Gotcha:** Never trust `customer_id` or any sensitive identifier from the LLM's function arguments. The LLM may hallucinate or be prompt-injected to pass a different customer ID. Always inject security context (customer ID, permissions) from your authenticated session server-side, not from the LLM's output.

---

## Q4. How do you implement streaming responses to improve perceived latency?

**Scenario:** Your document summarisation API takes 8–12 seconds to complete. Users see a blank screen for 10 seconds then get all the text at once. UX team says the experience feels broken. Users abandon after 3 seconds.

```
Without streaming (current):          With streaming (target):
  Request sent                          Request sent
  ....wait 10 seconds....               Word 1 appears (0.3s)
  Full response appears                 Word 2, 3, 4... (continuous)
  → User sees nothing for 10s          → User sees text flowing immediately
  → 60% drop-off rate                  → Engagement maintained ✅

  SSE (Server-Sent Events) — how streaming works:
  API sends: data: {"delta": {"content": "The"}}
             data: {"delta": {"content": " document"}}
             data: {"delta": {"content": " describes"}}
             data: [DONE]
```

```python
# Server-side: stream from Azure OpenAI
from openai import AzureOpenAI

client = AzureOpenAI(
    azure_endpoint=os.getenv("AZURE_OPENAI_ENDPOINT"),
    api_key=os.getenv("AZURE_OPENAI_KEY"),
    api_version="2024-10-21"
)

def stream_summary(document_text: str):
    stream = client.chat.completions.create(
        model="gpt-4o-prod",
        messages=[
            {"role": "system", "content": "Summarise financial documents concisely."},
            {"role": "user", "content": f"Summarise: {document_text}"}
        ],
        stream=True     # Enable streaming
    )

    for chunk in stream:
        if chunk.choices and chunk.choices[0].delta.content:
            yield chunk.choices[0].delta.content   # Generator: yield each token
```

```python
# FastAPI: expose streaming endpoint (SSE)
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()

@app.post("/summarise")
async def summarise(request: SummaryRequest):
    def generate():
        for token in stream_summary(request.document_text):
            yield f"data: {json.dumps({'token': token})}\n\n"   # SSE format
        yield "data: [DONE]\n\n"

    return StreamingResponse(
        generate(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "X-Accel-Buffering": "no"   # Disable nginx buffering — critical for SSE
        }
    )
```

```javascript
// Frontend: consume SSE stream
const response = await fetch('/summarise', {
    method: 'POST',
    body: JSON.stringify({ document_text: docText }),
    headers: { 'Content-Type': 'application/json' }
});

const reader = response.body.getReader();
const decoder = new TextDecoder();

while (true) {
    const { value, done } = await reader.read();
    if (done) break;

    const lines = decoder.decode(value).split('\n');
    for (const line of lines) {
        if (line.startsWith('data: ') && line !== 'data: [DONE]') {
            const data = JSON.parse(line.slice(6));
            document.getElementById('output').textContent += data.token;
        }
    }
}
```

> ⚠️ **Gotcha:** NGINX and Azure API Management (APIM) buffer responses by default — SSE tokens accumulate in the proxy buffer and are released in bursts, defeating the point of streaming. For NGINX: set `proxy_buffering off`. For APIM: disable response buffering in the policy with `<forward-request buffer-response="false"/>`.
