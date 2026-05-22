# LLMOps & Monitoring
### 🔴 Advanced

> *"A model deployed without monitoring is a risk deployed without visibility.  
>  LLMOps is the discipline of knowing when your AI is wrong before your users do."*

---

## Q1. How do you monitor Azure OpenAI usage, latency, and cost with Azure Monitor?

**Scenario:** Your CTO asks "How much did we spend on AI last month? Which feature consumed the most tokens? Is our latency SLA of < 3s p95 being met?" You have no answers because there's no monitoring.

```
Azure OpenAI built-in metrics (via Azure Monitor):

  azure_openai_resource
       │
       ├── AzureOpenAIRequests       → total API calls per model/deployment
       ├── AzureOpenAITokens         → prompt + completion tokens (cost driver)
       ├── AzureOpenAIProcessedPromptTokens
       ├── AzureOpenAIGeneratedCompletionTokens
       ├── AzureOpenAILatency        → ms per request (p50, p95, p99)
       ├── AzureOpenAIRateLimitErrors → count of 429 responses
       └── AzureOpenAIServerErrors   → count of 5xx responses
```

```bash
# Query token usage by deployment (last 7 days)
az monitor metrics list \
  --resource "/subscriptions/<SUB>/resourceGroups/rg-ai/providers/Microsoft.CognitiveServices/accounts/aoai-prod-cc" \
  --metric "TokenTransaction" \
  --dimension ModelDeploymentName \
  --interval PT1H \
  --start-time "$(date -u -d '7 days ago' +%Y-%m-%dT%H:%M:%SZ)" \
  --output table

# Create alert: latency p95 > 5 seconds
az monitor metrics alert create \
  --name "aoai-latency-alert" \
  --resource-group rg-ai \
  --scopes "/subscriptions/<SUB>/resourceGroups/rg-ai/providers/Microsoft.CognitiveServices/accounts/aoai-prod-cc" \
  --condition "avg AzureOpenAILatency > 5000" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --severity 2 \
  --action-groups "/subscriptions/<SUB>/resourceGroups/rg-monitoring/providers/microsoft.insights/actionGroups/ag-oncall"
```

```kusto
// KQL: Token consumption by feature (requires custom dimension in app logs)
// Application sends custom property "feature" with each call
AppTraces
| where TimeGenerated > ago(30d)
| where Message == "OpenAI completion"
| extend
    feature        = tostring(Properties["feature"]),
    prompt_tokens  = toint(Properties["prompt_tokens"]),
    completion_tokens = toint(Properties["completion_tokens"]),
    model          = tostring(Properties["model"])
| summarize
    TotalCalls = count(),
    TotalPromptTokens = sum(prompt_tokens),
    TotalCompletionTokens = sum(completion_tokens),
    AvgLatencyMs = avg(todouble(Properties["latency_ms"]))
  by feature, model
| extend EstimatedCostUSD = (TotalPromptTokens / 1000000.0 * 5.0)
                          + (TotalCompletionTokens / 1000000.0 * 15.0)
  // GPT-4o pricing: $5/1M input, $15/1M output
| order by EstimatedCostUSD desc

// KQL: P95 latency by hour
AppTraces
| where TimeGenerated > ago(7d)
| where Message == "OpenAI completion"
| extend latency_ms = todouble(Properties["latency_ms"])
| summarize P50=percentile(latency_ms,50), P95=percentile(latency_ms,95), P99=percentile(latency_ms,99)
  by bin(TimeGenerated, 1h)
| render timechart
```

```python
# Instrument every OpenAI call with structured logging
import time
import logging

logger = logging.getLogger(__name__)

def instrumented_chat_completion(messages: list, feature: str) -> str:
    start_ms = time.monotonic()

    response = openai_client.chat.completions.create(
        model="gpt-4o-prod",
        messages=messages,
        temperature=0.1
    )

    latency_ms = (time.monotonic() - start_ms) * 1000

    # Structured log → Application Insights → queryable via KQL above
    logger.info("OpenAI completion", extra={
        "custom_dimensions": {
            "feature":             feature,
            "model":               "gpt-4o-prod",
            "prompt_tokens":       response.usage.prompt_tokens,
            "completion_tokens":   response.usage.completion_tokens,
            "latency_ms":          round(latency_ms, 1),
            "finish_reason":       response.choices[0].finish_reason
        }
    })
    return response.choices[0].message.content
```

> ⚠️ **Gotcha:** Azure OpenAI's native `AzureOpenAITokens` metric aggregates all deployments together — you can't see which feature (RAG vs agent vs document processing) is driving cost without custom application-level logging. Instrument every call with a `feature` tag from day one.

---

## Q2. How do you evaluate RAG pipeline quality using RAGAS metrics?

**Scenario:** You improved your chunking strategy. The team says it "feels better." Your manager asks for numbers. You need objective metrics to compare v1 vs v2 of the RAG pipeline.

```
RAGAS evaluation framework — 4 key metrics:

  1. Faithfulness (0–1): Is the answer factually consistent with the context?
     Low = model is hallucinating beyond the retrieved chunks

  2. Answer Relevancy (0–1): Is the answer relevant to the question?
     Low = model went off-topic

  3. Context Precision (0–1): Are the retrieved chunks actually relevant?
     Low = retrieval is returning noise

  4. Context Recall (0–1): Were all relevant chunks retrieved?
     Low = retrieval is missing important context

  A/B comparison:
  ┌──────────────────────┬──────────┬──────────┐
  │ Metric               │ RAG v1   │ RAG v2   │
  ├──────────────────────┼──────────┼──────────┤
  │ Faithfulness         │ 0.72     │ 0.91 ✅  │
  │ Answer Relevancy     │ 0.68     │ 0.87 ✅  │
  │ Context Precision    │ 0.61     │ 0.79 ✅  │
  │ Context Recall       │ 0.55     │ 0.74 ✅  │
  └──────────────────────┴──────────┴──────────┘
  → v2 is clearly better — ship it ✅
```

```python
from ragas import evaluate
from ragas.metrics import (
    faithfulness,
    answer_relevancy,
    context_precision,
    context_recall
)
from datasets import Dataset

def run_ragas_evaluation(test_cases: list) -> dict:
    """
    test_cases: list of {question, ground_truth, answer, contexts}
    ground_truth: the known correct answer (from human-labeled eval set)
    contexts: list of chunks retrieved by your RAG pipeline
    """

    # Build evaluation dataset
    eval_dataset = Dataset.from_list([
        {
            "question":     tc["question"],
            "answer":       tc["answer"],           # RAG pipeline output
            "contexts":     tc["contexts"],         # Retrieved chunks (list of strings)
            "ground_truth": tc["ground_truth"]      # Human-verified correct answer
        }
        for tc in test_cases
    ])

    # Run RAGAS evaluation (uses GPT-4o to score — costs tokens)
    results = evaluate(
        dataset=eval_dataset,
        metrics=[faithfulness, answer_relevancy, context_precision, context_recall],
        llm=openai_client,
        embeddings=embedding_model
    )

    print(results)
    return results.to_pandas().to_dict()

# Build a golden eval set (50–100 Q&A pairs) from your domain experts
# Store in ADLS Gen2: evaluation/golden-set-v1.json
# Re-run evaluation on every pipeline change — track scores in Azure ML or MLflow
```

```bash
# Log RAGAS scores to Azure ML for tracking
az ml run log-artifact \
  --path ragas_results.json \
  --run-id $RUN_ID \
  --experiment-name rag-evaluation
```

> ⚠️ **Gotcha:** RAGAS uses GPT-4o (or another LLM) to score responses — this means evaluation itself has an LLM-in-the-loop with its own error rate. RAGAS scores are estimates, not ground truth. For regulated industries (banking compliance chatbot), supplement RAGAS with a human review of 20–30% of flagged cases each sprint to catch systematic errors RAGAS misses.

---

## Q3. How do you A/B test two Azure OpenAI models via APIM?

**Scenario:** GPT-4o is expensive. The team wants to test if GPT-4o-mini handles 70% of queries adequately, routing only complex ones to GPT-4o. You need to split traffic and measure quality difference in production.

```
A/B testing via Azure API Management:

  Client sends request to APIM (single endpoint)
       │
       ▼
  APIM Inbound Policy:
  Random number 0–100:
  ├── 0–70  → Backend: gpt-4o-mini deployment    (70% traffic)
  └── 71–100 → Backend: gpt-4o deployment        (30% traffic)

  Both backends log:
  - model used
  - tokens consumed
  - latency
  - user feedback signal (thumbs up/down)

  Analysis (2 weeks):
  gpt-4o-mini: 70% of traffic, $800/month, user satisfaction 82%
  gpt-4o:      30% of traffic, $2,100/month, user satisfaction 91%
  → Decision: route "complex" queries (>500 tokens input) to GPT-4o
             route "simple" queries to GPT-4o-mini
```

```xml
<!-- APIM Inbound Policy: random traffic split -->
<policies>
  <inbound>
    <base />
    <!-- Generate random 0-100 and store in variable -->
    <set-variable name="randomSplit" value="@(new Random().Next(0, 100))" />

    <!-- Route based on random value -->
    <choose>
      <when condition="@(int.Parse(context.Variables.GetValueOrDefault<string>("randomSplit")) <= 70)">
        <set-backend-service
          base-url="https://aoai-prod-cc.openai.azure.com/openai/deployments/gpt-4o-mini" />
        <set-header name="X-AB-Variant" exists-action="override">
          <value>gpt-4o-mini</value>
        </set-header>
      </when>
      <otherwise>
        <set-backend-service
          base-url="https://aoai-prod-cc.openai.azure.com/openai/deployments/gpt-4o-prod" />
        <set-header name="X-AB-Variant" exists-action="override">
          <value>gpt-4o</value>
        </set-header>
      </otherwise>
    </choose>

    <!-- Log variant to Application Insights for analysis -->
    <trace source="ab-test" severity="information">
      <message>@($"AB variant: {context.Variables.GetValueOrDefault<string>("X-AB-Variant")}")</message>
    </trace>
  </inbound>
</policies>
```

```kusto
// Analyse A/B test results after 2 weeks
AppTraces
| where TimeGenerated > ago(14d)
| where Message == "OpenAI completion"
| extend
    variant      = tostring(Properties["ab_variant"]),
    latency_ms   = todouble(Properties["latency_ms"]),
    total_tokens = toint(Properties["prompt_tokens"]) + toint(Properties["completion_tokens"]),
    thumbs_up    = tobool(Properties["user_feedback_positive"])
| summarize
    RequestCount  = count(),
    AvgLatency    = avg(latency_ms),
    P95Latency    = percentile(latency_ms, 95),
    AvgTokens     = avg(total_tokens),
    SatisfactionPct = countif(thumbs_up == true) * 100.0 / count()
  by variant
| order by variant asc
```

> ⚠️ **Gotcha:** Random traffic splitting means the same user may get GPT-4o on one request and GPT-4o-mini on the next. For multi-turn conversations, this causes inconsistent quality within a session. Use session-sticky routing: hash the session ID to determine the variant, and keep that variant for the entire conversation.

---

## Q4. How do you detect model quality drift over time?

**Scenario:** The RAG chatbot scored 0.87 faithfulness in January. It's now August. No code changed. But users report more wrong answers. You have no way to know if the model's behaviour has drifted.

```
Drift detection — three signals:

  1. Metric drift (automated):
  Run RAGAS on a fixed 100-question golden set weekly
  Track faithfulness, answer_relevancy over time
  Alert if any metric drops > 10% week-over-week

  2. Implicit feedback drift (from users):
  Track: thumbs down rate, session abandonment rate, escalation to human rate
  Baseline (week 1): 4% negative feedback
  Alert threshold: > 8% → potential drift

  3. Topic drift in queries (queries your RAG can't answer):
  Track "I cannot find this in documents" responses
  Spike → new topics emerging that aren't in the index
  Fix: add new documents to index, not retrain the model
```

```python
import json
from datetime import datetime, timedelta
from azure.storage.blob import BlobServiceClient

def weekly_drift_check(golden_set_path: str, history_blob_path: str) -> dict:
    """Run RAGAS weekly and compare to previous week's scores."""

    # Load golden set
    with open(golden_set_path) as f:
        golden_set = json.load(f)

    # Run current evaluation
    test_cases = []
    for item in golden_set:
        answer_result = rag_chatbot.query(item["question"])
        test_cases.append({
            "question":     item["question"],
            "ground_truth": item["ground_truth"],
            "answer":       answer_result["answer"],
            "contexts":     [c["content"] for c in answer_result["sources"]]
        })

    current_scores = run_ragas_evaluation(test_cases)

    # Load previous week's scores
    blob_client = BlobServiceClient.from_connection_string(STORAGE_CONN)
    try:
        prev_blob = blob_client.get_blob_client("monitoring", history_blob_path)
        prev_scores = json.loads(prev_blob.download_blob().readall())
    except Exception:
        prev_scores = None    # First run — no baseline

    # Detect drift
    drift_alerts = []
    if prev_scores:
        for metric in ["faithfulness", "answer_relevancy", "context_precision"]:
            current = current_scores.get(metric, 0)
            previous = prev_scores.get(metric, 0)
            drop_pct = (previous - current) / previous * 100 if previous > 0 else 0

            if drop_pct > 10:
                drift_alerts.append({
                    "metric": metric,
                    "previous": round(previous, 3),
                    "current": round(current, 3),
                    "drop_pct": round(drop_pct, 1)
                })

    # Save current scores for next week
    current_scores["timestamp"] = datetime.utcnow().isoformat()
    blob_client.get_blob_client("monitoring", history_blob_path) \
               .upload_blob(json.dumps(current_scores), overwrite=True)

    if drift_alerts:
        send_alert_to_teams(f"RAG quality drift detected: {drift_alerts}")

    return {"current_scores": current_scores, "drift_alerts": drift_alerts}
```

> ⚠️ **Gotcha:** Model quality drift is often NOT caused by the model changing — Azure OpenAI model versions are pinned at deployment. Common real causes: (1) the document index has become stale (new regulations added, old ones removed from index), (2) users are asking new types of questions the system wasn't designed for, (3) the system prompt was edited by someone without running evals. Always run evals after any change to prompts, index, or chunking strategy.
