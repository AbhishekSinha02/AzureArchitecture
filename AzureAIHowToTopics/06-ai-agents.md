# AI Agents & Agentic Patterns
### 🔴 Advanced

> *"An agent that can only answer questions is a chatbot.  
>  An agent that can take actions is a system — and systems need guardrails."*

---

## Q1. How do you build a multi-step AI agent using Azure OpenAI Assistants API?

**Scenario:** A financial analyst needs an AI assistant that can: (1) retrieve stock data from an API, (2) run a Python calculation for portfolio risk, (3) generate a Word report. Each step depends on the previous — it's too complex for a single prompt.

```
Assistants API — stateful multi-step agent:

  User: "Generate a risk report for my portfolio: AAPL 40%, MSFT 30%, GOOG 30%"
       │
       ▼
  Assistant (Thread: persistent state)
       │ Step 1: tool_call → get_stock_data(symbols=["AAPL","MSFT","GOOG"])
       │ Tool result: prices, volatility, correlation matrix
       │
       │ Step 2: tool_call → code_interpreter
       │ Python code: calculate portfolio VaR, Sharpe ratio
       │ Code output: {"VaR_95": 0.045, "sharpe": 1.82}
       │
       │ Step 3: file_output → risk_report.docx
       │ Generated Word report with charts
       ▼
  User receives: download link to risk_report.docx ✅

  Key advantage: multi-step reasoning with state between steps
  Assistant manages the loop — you just poll for completion
```

```python
from openai import AzureOpenAI
import time, json

client = AzureOpenAI(
    azure_endpoint=os.getenv("AZURE_OPENAI_ENDPOINT"),
    api_key=os.getenv("AZURE_OPENAI_KEY"),
    api_version="2024-05-01-preview"    # Assistants API requires this version
)

# Create a persistent assistant (once — reuse the ID)
assistant = client.beta.assistants.create(
    name="Portfolio Risk Analyst",
    instructions=(
        "You are a quantitative finance analyst. "
        "When given portfolio weights, retrieve market data, "
        "calculate risk metrics (VaR, Sharpe ratio, max drawdown), "
        "and produce a formatted report. "
        "Always show your calculation methodology."
    ),
    model="gpt-4o-prod",
    tools=[
        {"type": "code_interpreter"},     # Runs Python code securely
        {
            "type": "function",
            "function": {
                "name": "get_market_data",
                "description": "Get historical prices and volatility for given stock symbols",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "symbols": {"type": "array", "items": {"type": "string"}},
                        "period_days": {"type": "integer", "default": 252}
                    },
                    "required": ["symbols"]
                }
            }
        }
    ]
)
print(f"Assistant ID: {assistant.id}")   # Save this ID — reuse across sessions

def run_portfolio_analysis(user_request: str, portfolio: dict) -> str:
    # Create a new thread (conversation)
    thread = client.beta.threads.create()

    # Add user message
    client.beta.threads.messages.create(
        thread_id=thread.id,
        role="user",
        content=f"{user_request}\nPortfolio: {json.dumps(portfolio)}"
    )

    # Start the run
    run = client.beta.threads.runs.create(
        thread_id=thread.id,
        assistant_id=assistant.id
    )

    # Poll until complete (handle tool calls in the loop)
    while run.status not in ("completed", "failed", "cancelled"):
        time.sleep(2)
        run = client.beta.threads.runs.retrieve(thread_id=thread.id, run_id=run.id)

        if run.status == "requires_action":
            tool_outputs = []
            for tool_call in run.required_action.submit_tool_outputs.tool_calls:
                if tool_call.function.name == "get_market_data":
                    args = json.loads(tool_call.function.arguments)
                    data = fetch_market_data(args["symbols"], args.get("period_days", 252))
                    tool_outputs.append({
                        "tool_call_id": tool_call.id,
                        "output": json.dumps(data)
                    })

            # Submit tool results back to the assistant
            run = client.beta.threads.runs.submit_tool_outputs(
                thread_id=thread.id,
                run_id=run.id,
                tool_outputs=tool_outputs
            )

    # Retrieve the final response
    messages = client.beta.threads.messages.list(thread_id=thread.id)
    return messages.data[0].content[0].text.value
```

> ⚠️ **Gotcha:** Assistants API runs are asynchronous — polling adds 2–30 seconds of latency per step depending on model and tool complexity. For user-facing features requiring streaming, use SSE to stream run events (`client.beta.threads.runs.create(..., stream=True)`) and send intermediate thinking steps to the UI so users see progress rather than a blank screen.

---

## Q2. How do you build a multi-step agent using Semantic Kernel?

**Scenario:** You want a claims processing agent that: reads a claim PDF, extracts key fields, validates against policy rules, calculates payout, and writes the decision to a database. The logic is too complex for a single tool call.

```
Semantic Kernel planner — automatic step decomposition:

  User goal: "Process claim #CLM-2024-001 and determine payout"
       │
       ▼
  Planner (GPT-4o) decomposes into steps:
  1. DocumentPlugin.ExtractClaim(claim_id="CLM-2024-001")
  2. PolicyPlugin.ValidateCoverage(policy_id, claim_type)
  3. CalculationPlugin.ComputePayout(coverage_amount, damage_assessment)
  4. DatabasePlugin.WriteClaim(claim_id, payout_amount, decision)
       │
       ▼
  SK executes each step, passing outputs to next step
  Final result: claim written to DB with payout decision ✅
```

```python
import asyncio
from semantic_kernel import Kernel
from semantic_kernel.connectors.ai.open_ai import AzureChatCompletion
from semantic_kernel.functions import kernel_function

# Set up kernel
kernel = Kernel()
kernel.add_service(AzureChatCompletion(
    deployment_name="gpt-4o-prod",
    endpoint=os.getenv("AZURE_OPENAI_ENDPOINT"),
    api_key=os.getenv("AZURE_OPENAI_KEY")
))

# Define plugins (groups of related functions)
class DocumentPlugin:
    @kernel_function(description="Extract structured fields from a claim document")
    async def extract_claim(self, claim_id: str) -> str:
        pdf_bytes = await storage.download(f"claims/{claim_id}.pdf")
        result = await document_intelligence.extract(pdf_bytes)
        return json.dumps(result)

class PolicyPlugin:
    @kernel_function(description="Validate whether a claim type is covered under the policy")
    async def validate_coverage(self, policy_id: str, claim_type: str) -> str:
        policy = await cosmos.get_policy(policy_id)
        covered = claim_type in policy["covered_perils"]
        return json.dumps({"covered": covered, "limit": policy.get("limit", 0)})

class CalculationPlugin:
    @kernel_function(description="Calculate insurance payout given coverage and damage")
    async def compute_payout(self, coverage_limit: float, damage_amount: float,
                              deductible: float = 500.0) -> str:
        payout = min(coverage_limit, max(0, damage_amount - deductible))
        return json.dumps({"payout": payout, "deductible_applied": deductible})

# Register plugins
kernel.add_plugin(DocumentPlugin(), plugin_name="Document")
kernel.add_plugin(PolicyPlugin(), plugin_name="Policy")
kernel.add_plugin(CalculationPlugin(), plugin_name="Calculation")

# Run agent with auto-planner
async def process_claim(claim_id: str):
    from semantic_kernel.planners import FunctionCallingStepwisePlanner

    planner = FunctionCallingStepwisePlanner(service_id="default")
    result = await planner.invoke(
        kernel=kernel,
        question=f"Process insurance claim {claim_id}: extract fields, validate coverage, compute payout, and return the final payout amount with justification."
    )
    return result.final_answer
```

> ⚠️ **Gotcha:** Semantic Kernel planners make multiple LLM calls for planning + each step execution. A 4-step plan costs 5+ LLM calls. Token costs accumulate fast. For known, fixed workflows (claims processing always has the same 4 steps), use a hardcoded sequential pipeline instead of a dynamic planner — faster, cheaper, and more predictable.

---

## Q3. How do you implement persistent agent memory across sessions?

**Scenario:** A customer calls your AI support agent on Monday. On Thursday they call back: "I have a question about that issue we discussed." The agent has no memory of Monday's conversation — the user has to explain everything again.

```
Agent memory architecture:

  Session 1 (Monday)
  User: "My mortgage payment failed due to NSF"
  Agent: resolves issue → WRITES to memory store:
  {
    "customer_id": "cust-001",
    "memory_type": "episode",
    "summary": "NSF on mortgage Jan 15. Resolved by waiving $35 fee. Advised to increase auto-transfer.",
    "date": "2024-01-15",
    "tags": ["mortgage", "NSF", "fee_waived"]
  }

  Session 2 (Thursday)
  User: "I have a question about that issue from Monday"
  Agent: READS memory → retrieves Jan 15 episode
  → "I see on January 15th your mortgage payment had an NSF. We waived the $35 fee.
     Would you like to discuss that further?"
  → User: "Yes exactly!" ← seamless experience ✅
```

```python
from azure.search.documents import SearchClient
from azure.search.documents.models import VectorizedQuery

# Memory stored in AI Search (searchable + vector retrieval)
memory_client = SearchClient(
    endpoint=os.getenv("AZURE_SEARCH_ENDPOINT"),
    index_name="agent-memory",
    credential=AzureKeyCredential(os.getenv("AZURE_SEARCH_KEY"))
)

def save_episode_memory(customer_id: str, conversation: list):
    """Summarise and save a completed conversation to memory."""
    # Ask LLM to summarise the episode
    summary = openai_client.chat.completions.create(
        model="gpt-4o-prod",
        messages=[
            {"role": "system", "content": "Summarise this customer service conversation in 2-3 sentences. Include: issue, resolution, and any commitments made."},
            {"role": "user", "content": json.dumps(conversation)}
        ],
        max_tokens=150,
        temperature=0
    ).choices[0].message.content

    # Embed summary for semantic search
    embedding = embed(summary)

    # Store in AI Search
    memory_client.upload_documents([{
        "id":           f"{customer_id}-{int(time.time())}",
        "customer_id":  customer_id,
        "summary":      summary,
        "date":         datetime.utcnow().isoformat(),
        "memory_type":  "episode",
        "contentVector": embedding,
        "raw_conversation": json.dumps(conversation[-5:])  # Last 5 turns only
    }])

def retrieve_relevant_memories(customer_id: str, current_query: str, top_k: int = 3) -> list:
    """Find memories relevant to the current conversation."""
    query_vector = embed(current_query)

    results = memory_client.search(
        search_text=current_query,
        filter=f"customer_id eq '{customer_id}'",   # Only this customer's memories
        vector_queries=[VectorizedQuery(
            vector=query_vector,
            k_nearest_neighbors=10,
            fields="contentVector"
        )],
        top=top_k,
        select=["summary", "date", "memory_type"]
    )

    return [{"summary": r["summary"], "date": r["date"]} for r in results]

def agent_with_memory(customer_id: str, user_message: str, current_session: list) -> str:
    # Retrieve relevant past memories
    memories = retrieve_relevant_memories(customer_id, user_message)

    memory_context = ""
    if memories:
        memory_context = "Relevant past interactions:\n" + \
            "\n".join([f"- {m['date'][:10]}: {m['summary']}" for m in memories])

    messages = [
        {"role": "system", "content": f"You are a helpful banking support agent.\n\n{memory_context}"},
        *current_session[-6:],     # Last 6 turns of current session
        {"role": "user", "content": user_message}
    ]

    return instrumented_chat_completion(messages, feature="support_agent_with_memory")
```

> ⚠️ **Gotcha:** Storing raw conversation history as agent memory creates privacy and compliance risk — you're keeping detailed PII records of every interaction. Follow your data retention policy: summarise conversations (removing specific amounts, account numbers) before storing, set TTL on memory records (e.g., 90 days), and ensure memory storage is covered by your PIPEDA data processing agreement.

---

## Q4. How do you build a multi-agent workflow where specialized agents collaborate?

**Scenario:** A complex loan application requires: document extraction agent, credit risk scoring agent, regulatory compliance agent, and a supervisor agent that coordinates them and makes the final decision.

```
Multi-agent orchestration pattern:

  User submits loan application (docs + form data)
       │
       ▼
  Supervisor Agent (GPT-4o)
       │ "I need 3 analyses before I can decide"
       ├──► Document Agent    → extracts fields from uploaded docs
       │                         → returns: {income, assets, liabilities, employment}
       │
       ├──► Risk Agent        → scores credit risk
       │                         → returns: {risk_score: 72, risk_tier: "medium"}
       │
       └──► Compliance Agent  → checks OSFI/PIPEDA requirements
                                 → returns: {compliant: true, flags: []}
       │
       ▼
  Supervisor: synthesises all 3 outputs → decision
  "Approved with conditions: 4.2% rate, 25yr amortization"
  → Written to CosmosDB → Notification → Applicant
```

```python
import asyncio
from dataclasses import dataclass

@dataclass
class AgentResult:
    agent: str
    result: dict
    success: bool
    error: str = None

async def document_agent(application_id: str) -> AgentResult:
    try:
        docs = await storage.list_docs(application_id)
        extracted = {}
        for doc in docs:
            pdf = await storage.download(doc["path"])
            result = await di_client.extract(pdf)
            extracted.update(result)
        return AgentResult("document", extracted, True)
    except Exception as e:
        return AgentResult("document", {}, False, str(e))

async def risk_agent(extracted_data: dict) -> AgentResult:
    try:
        prompt = f"Score the credit risk for this applicant data: {json.dumps(extracted_data)}. Return JSON with risk_score (0-100) and risk_tier (low/medium/high)."
        response = await async_openai_call(prompt, temperature=0)
        return AgentResult("risk", json.loads(response), True)
    except Exception as e:
        return AgentResult("risk", {}, False, str(e))

async def compliance_agent(extracted_data: dict, application_id: str) -> AgentResult:
    try:
        checks = {
            "age_of_majority": extracted_data.get("age", 0) >= 18,
            "income_verified": bool(extracted_data.get("income_proof_doc")),
            "id_verified":     bool(extracted_data.get("government_id_doc")),
            "pep_screening":   await check_pep_list(extracted_data.get("name", ""))
        }
        return AgentResult("compliance", {"compliant": all(checks.values()), "checks": checks}, True)
    except Exception as e:
        return AgentResult("compliance", {}, False, str(e))

async def supervisor_agent(application_id: str) -> dict:
    # Step 1: Run document extraction first (others depend on it)
    doc_result = await document_agent(application_id)
    if not doc_result.success:
        return {"decision": "error", "reason": "Document extraction failed"}

    # Step 2: Run risk + compliance in parallel (independent)
    risk_result, compliance_result = await asyncio.gather(
        risk_agent(doc_result.result),
        compliance_agent(doc_result.result, application_id)
    )

    # Step 3: Supervisor synthesises all results
    synthesis_prompt = f"""
    Loan application {application_id} analysis:

    Extracted data: {json.dumps(doc_result.result)}
    Risk assessment: {json.dumps(risk_result.result)}
    Compliance check: {json.dumps(compliance_result.result)}

    Based on our lending policy (min credit score 650, max DTI 43%, must be OSFI compliant),
    make a lending decision. Return JSON: {{decision: approved/declined/pending_review,
    conditions: [], reason: str, interest_rate: float|null}}
    """

    decision = await async_openai_call(synthesis_prompt, temperature=0)
    return json.loads(decision)
```

> ⚠️ **Gotcha:** Multi-agent systems amplify failures — if any agent returns incorrect data, the supervisor makes a wrong decision based on bad inputs. For high-stakes decisions (loan approval), never rely on a single LLM call per agent. Add a validation layer: the supervisor should request human review when any agent returns low-confidence results or when agents' outputs contradict each other.
