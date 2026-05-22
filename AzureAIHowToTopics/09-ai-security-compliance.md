# AI Security & Compliance (Canadian Financial Services)
### 🔴 Advanced

> *"An AI system that cannot explain its decisions  
>  is a liability, not an asset, in a regulated industry."*

---

## Q1. How do you build an OSFI B-10 compliant AI governance framework for a banking chatbot?

**Scenario:** Your bank is deploying an AI chatbot for retail customers. OSFI Guideline B-10 (Technology and Cyber Risk Management) requires documented model risk management, explainability, and human oversight for AI systems. The AI team has none of this in place.

```
OSFI B-10 AI compliance requirements → technical controls:

  Requirement                   Technical Control
  ─────────────────────────────────────────────────────────────
  Model inventory               → AI Foundry: register every model deployment
  Risk classification           → Tag models: low/medium/high risk (per B-10)
  Model validation              → RAGAS eval suite + human review 20% sample
  Explainability                → Citation enforcement + confidence scoring
  Human oversight               → Human-in-the-loop for high-risk decisions
  Audit trail                   → Immutable log: every LLM call → Log Analytics
  Model change management       → Git + PR approval for prompt changes
  Third-party risk              → Azure OpenAI = third party → vendor risk assessment
  Incident response             → Runbook: what to do when AI produces wrong output
  Board reporting               → Monthly AI risk dashboard → senior management
```

```python
# OSFI B-10 compliant AI request wrapper
import hashlib
from datetime import datetime

class OSFICompliantAIClient:
    """Wraps every Azure OpenAI call with B-10-required audit logging."""

    def __init__(self, model_id: str, risk_tier: str):
        self.model_id = model_id
        self.risk_tier = risk_tier    # "low", "medium", "high"
        self.client = AzureOpenAI(
            azure_endpoint=os.getenv("AZURE_OPENAI_ENDPOINT"),
            api_key=os.getenv("AZURE_OPENAI_KEY"),
            api_version="2024-10-21"
        )

    def complete(self, messages: list, user_id: str, session_id: str,
                 feature: str, require_human_review: bool = False) -> dict:

        request_id = hashlib.sha256(
            f"{session_id}{datetime.utcnow().isoformat()}".encode()
        ).hexdigest()[:16]

        response = self.client.chat.completions.create(
            model=self.model_id,
            messages=messages,
            temperature=0.1,
            user=user_id    # Passed to Azure OpenAI for abuse monitoring
        )

        answer = response.choices[0].message.content

        # Immutable audit log entry (OSFI B-10 Section 5.3 — audit trail)
        audit_entry = {
            "timestamp":       datetime.utcnow().isoformat() + "Z",
            "request_id":      request_id,
            "user_id":         user_id,           # Pseudonymised in production
            "session_id":      session_id,
            "feature":         feature,
            "model_id":        self.model_id,
            "risk_tier":       self.risk_tier,
            "prompt_tokens":   response.usage.prompt_tokens,
            "completion_tokens": response.usage.completion_tokens,
            "finish_reason":   response.choices[0].finish_reason,
            # Do NOT log full prompt/response for PII reasons — log hash only
            "prompt_hash":     hashlib.sha256(str(messages).encode()).hexdigest(),
            "response_hash":   hashlib.sha256(answer.encode()).hexdigest(),
        }

        # Log to Log Analytics (WORM via Immutable Storage — 7yr FINTRAC retention)
        logger.info("AI_AUDIT", extra={"custom_dimensions": audit_entry})

        # High-risk tier: queue for human review sample (20%)
        import random
        if self.risk_tier == "high" and random.random() < 0.20:
            human_review_queue.send_message(json.dumps({
                "request_id": request_id,
                "session_id": session_id,
                "answer_hash": audit_entry["response_hash"]
            }))

        return {"answer": answer, "request_id": request_id}
```

```bash
# Tag Azure OpenAI resource with risk classification (OSFI model inventory)
az resource tag \
  --ids "/subscriptions/<SUB>/resourceGroups/rg-ai/providers/Microsoft.CognitiveServices/accounts/aoai-prod-cc" \
  --tags \
    "osfi_risk_tier=high" \
    "model_purpose=customer_facing_chatbot" \
    "model_owner=ai-team@bank.com" \
    "last_validated=$(date +%Y-%m-%d)" \
    "next_validation=$(date -d '+6 months' +%Y-%m-%d)" \
    "human_oversight=required"
```

> ⚠️ **Gotcha:** OSFI B-10 does not prescribe specific technical standards — it requires that you can demonstrate your AI governance process to an examiner. Document every decision: why you chose this model, how you validated it, what controls are in place, who owns it, and what happens when it fails. The technical controls matter less than the documented evidence that you have them.

---

## Q2. How do you ensure Azure OpenAI data stays in Canada for PIPEDA compliance?

**Scenario:** Your legal team says all customer data processed by AI must stay within Canada's borders (PIPEDA data residency). Azure OpenAI is available in Canada East. What else do you need to configure?

```
Data residency requirements:

  Data at rest:
  ├── Azure OpenAI: Canada East region → data processed + stored in Canada ✅
  ├── AI Search: Canada East → index stored in Canada ✅
  ├── Document Intelligence: Canada East → processed in Canada ✅
  └── Application Insights logs: must be in Canada East workspace ← often missed

  Data in transit:
  ├── All API calls via Private Endpoint → never leaves Azure backbone ✅
  └── No public endpoint → no data exits Canada through internet ✅

  Prompt + response logging:
  ├── Azure OpenAI abuse monitoring: Microsoft may process logs ← read ToS carefully
  ├── Opt-out of data processing for model training:
  │   Request "Data Protection Addendum" via Microsoft account team
  └── Log Analytics workspace: must be Canada East (not East US default) ← common miss

  Exceptions to review:
  - Global Standard deployments may route traffic globally → use Standard tier
  - Azure CDN and Front Door have global PoPs → don't put AI behind global CDN
  - Third-party embeddings (e.g., Cohere) → data leaves Azure → may violate PIPEDA
```

```bash
# Create Log Analytics workspace in Canada East (NOT default East US)
az monitor log-analytics workspace create \
  --workspace-name law-ai-prod \
  --resource-group rg-ai \
  --location canadaeast \           # Must be Canada East for data residency
  --retention-in-days 2555          # 7 years for FINTRAC compliance

# Enable diagnostic settings on Azure OpenAI → Canada East workspace
az monitor diagnostic-settings create \
  --name "aoai-diagnostics" \
  --resource "/subscriptions/<SUB>/resourceGroups/rg-ai/providers/Microsoft.CognitiveServices/accounts/aoai-prod-cc" \
  --workspace "/subscriptions/<SUB>/resourceGroups/rg-ai/providers/Microsoft.OperationalInsights/workspaces/law-ai-prod" \
  --logs '[{"category":"Audit","enabled":true},{"category":"RequestResponse","enabled":false}]'
  # RequestResponse=false to avoid logging actual prompt/response content (PII risk)

# Verify all AI resources are in Canada
az resource list \
  --resource-group rg-ai \
  --query "[?kind=='OpenAI' || contains(type, 'search') || contains(type, 'FormRecognizer')].{name:name, location:location}" \
  --output table
# Expected: all locations = "canadaeast"
```

> ⚠️ **Gotcha:** Azure OpenAI **Global Standard** deployment routes requests globally to the lowest-latency region — this means your prompts may be processed in the US or Europe, violating Canadian data residency. Use **Standard** (regional) deployment with `canadaeast` endpoint to guarantee data stays in Canada. Global Standard is 40% cheaper — but not for regulated financial data.

---

## Q3. How do you implement human-in-the-loop for high-stakes AI decisions?

**Scenario:** Your AI model recommends mortgage approval/decline. Regulators require that any AI-driven credit decision has a human review path — especially for declined applications (OSFI model risk management, FCAC fairness requirements).

```
Human-in-the-loop pattern:

  AI Decision Engine
       │ score = 72 (threshold: approve > 70, decline < 60, review 60-70)
       │
       ├── Score > 70 → AUTO APPROVE
       │    → Loan offer sent to customer
       │    → Logged: AI decision, model version, score
       │
       ├── Score 60–70 → HUMAN REVIEW QUEUE
       │    → Service Bus: sbq-loan-review
       │    → Loan officer reviews in portal
       │    → Decision: approve / decline / escalate
       │    → SLA: 4 business hours
       │    → Logged: AI recommendation + human final decision
       │
       └── Score < 60 → HUMAN CONFIRMS DECLINE
            → Decline requires human confirmation (not AI-only)
            → Adverse action notice sent (FCAC requirement)
            → Customer has right to request explanation
            → Logged: reason codes + SHAP values for explainability
```

```python
from azure.servicebus import ServiceBusClient, ServiceBusMessage
import json

def loan_decision_with_hitl(application_id: str, ai_score: float,
                             model_version: str, shap_values: dict) -> dict:

    audit_log = {
        "application_id": application_id,
        "ai_score":        ai_score,
        "model_version":   model_version,
        "shap_values":     shap_values,      # Top 5 factors for explainability
        "decision_time":   datetime.utcnow().isoformat()
    }

    if ai_score >= 70:
        # Auto-approve
        decision = {"outcome": "approved", "review_type": "ai_auto"}
        cosmos.write_decision(application_id, {**audit_log, **decision})
        notify_customer(application_id, "approved")

    elif ai_score >= 60:
        # Queue for human review
        message = ServiceBusMessage(
            json.dumps({
                "application_id":  application_id,
                "ai_score":        ai_score,
                "ai_recommendation": "approve" if ai_score >= 65 else "decline",
                "shap_values":     shap_values,
                "sla_deadline":    (datetime.utcnow() + timedelta(hours=4)).isoformat()
            }),
            subject="human_review_required"
        )
        sb_client.get_queue_sender("sbq-loan-review").send_messages(message)
        decision = {"outcome": "pending_human_review", "review_type": "human_required"}
        cosmos.write_decision(application_id, {**audit_log, **decision})

    else:
        # Decline requires human confirmation — never auto-decline alone
        message = ServiceBusMessage(
            json.dumps({
                "application_id": application_id,
                "ai_score":       ai_score,
                "action_required": "confirm_decline",
                "reason_codes":   [k for k, v in sorted(shap_values.items(),
                                   key=lambda x: abs(x[1]), reverse=True)[:5]],
                "sla_deadline":   (datetime.utcnow() + timedelta(hours=4)).isoformat()
            }),
            subject="confirm_decline"
        )
        sb_client.get_queue_sender("sbq-loan-review").send_messages(message)
        decision = {"outcome": "pending_decline_confirmation", "review_type": "human_required"}
        cosmos.write_decision(application_id, {**audit_log, **decision})

    return decision
```

> ⚠️ **Gotcha:** "Human-in-the-loop" doesn't satisfy regulators if the human rubber-stamps every AI recommendation without actually reviewing. Track the human override rate — if loan officers approve 99.9% of AI recommendations, the "review" is theatre. Set expectations: if the override rate drops below 5%, investigate whether reviewers have sufficient information and time. OSFI examiners look at override rates.

---

## Q4. How do you build an immutable AI audit trail for 7-year compliance retention?

**Scenario:** A customer disputes an AI-driven credit decision from 18 months ago. The regulator asks for the exact prompt sent to the model, the model version, the output, and every human decision in the chain. You have no audit trail.

```
Immutable audit trail architecture:

  Every AI interaction
       │
       ▼
  Azure Monitor / Application Insights
       │ structured log (JSON): request_id, user_id_hash,
       │ model_version, feature, input_hash, output_hash,
       │ decision, human_override, timestamp
       │
       ▼
  Log Analytics Workspace (Canada East)
       │ Retention: 2 years hot
       │
       ▼
  Azure Storage (WORM — Write Once Read Many)
  Blob Immutability Policy: LOCKED, 7 years
  Container: ai-audit-archive/
       │ exported from Log Analytics monthly via Data Export
       ▼
  Lifecycle policy: Hot → Cool after 1yr → Archive after 3yr
  Deletion: 7 years exactly (FINTRAC requirement)

  Why WORM matters:
  Regular blob: can be deleted by admin → audit trail tampered
  WORM locked: even subscription owners cannot delete → tamper-proof ✅
```

```bash
# Set WORM (immutability) policy on audit storage container — LOCKED = permanent
az storage container immutability-policy create \
  --account-name stauditprodcc \
  --container-name ai-audit-archive \
  --period 2555 \                  # 7 years in days
  --resource-group rg-compliance

# LOCK the policy (irreversible — cannot be changed or shortened after locking)
# ⚠️ Only run this after verifying the policy is correct — CANNOT BE UNDONE
az storage container immutability-policy lock \
  --account-name stauditprodcc \
  --container-name ai-audit-archive \
  --if-match "<etag-from-create-command>"

# Set up Log Analytics data export to archive storage (monthly)
az monitor log-analytics workspace data-export create \
  --name export-ai-audit \
  --workspace-name law-ai-prod \
  --resource-group rg-ai \
  --tables "AppTraces,AppDependencies" \
  --destination "/subscriptions/<SUB>/resourceGroups/rg-compliance/providers/Microsoft.Storage/storageAccounts/stauditprodcc"
```

```python
# Application: log every AI decision in structured, queryable format
def log_ai_decision(request: dict, response: dict, decision: dict):
    """Structured audit log compliant with OSFI B-10 record-keeping requirements."""

    logger.info("AI_DECISION_AUDIT", extra={
        "custom_dimensions": {
            # Identity (pseudonymised — no raw PII)
            "user_id_hash":    hashlib.sha256(request["user_id"].encode()).hexdigest(),
            "session_id":      request["session_id"],
            "request_id":      request["request_id"],

            # Model traceability
            "model_id":        request["model"],
            "model_version":   request["model_version"],
            "deployment_name": request["deployment"],
            "api_version":     "2024-10-21",

            # Decision audit
            "feature":         request["feature"],
            "decision":        decision["outcome"],
            "ai_score":        decision.get("score"),
            "human_reviewed":  decision.get("human_reviewed", False),
            "human_override":  decision.get("human_override", False),

            # Explainability
            "top_factors":     json.dumps(decision.get("shap_top5", {})),

            # Data hashes (not content — protects PII while enabling audit)
            "input_hash":      hashlib.sha256(json.dumps(request["messages"]).encode()).hexdigest(),
            "output_hash":     hashlib.sha256(response["answer"].encode()).hexdigest(),

            # Timestamp
            "event_time":      datetime.utcnow().isoformat() + "Z"
        }
    })
```

> ⚠️ **Gotcha:** Locking a WORM policy on Azure Storage is **permanent and irreversible** — you cannot shorten or remove the immutability period after locking. Test with a short-period policy first (e.g., 1 day) in a dev environment to confirm your data export pipeline works correctly before applying and locking the 7-year policy in production.
