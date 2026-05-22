# Azure AI Document Intelligence
### 🟡 Intermediate → 🔴 Advanced

> *"OCR reads characters. Document Intelligence reads meaning.  
>  The difference is the gap between a scanner and an analyst."*

---

## Q1. How do you extract structured data from invoices using the prebuilt Invoice model?

**Scenario:** Your accounts payable team manually keys 500 invoices per day from PDFs. Each invoice has vendor name, line items, amounts, dates, and PO numbers. You need to automate extraction with > 95% accuracy.

```
Prebuilt Invoice model — what it extracts:

  Invoice PDF
       │ Azure Document Intelligence (prebuilt-invoice model)
       ▼
  Structured JSON output:
  {
    "VendorName":    "Contoso Supplies Ltd.",     confidence: 0.99
    "InvoiceDate":   "2024-01-15",               confidence: 0.98
    "InvoiceTotal":  {"amount": 4520.00, "currencyCode": "CAD"},
    "PurchaseOrder": "PO-2024-00123",            confidence: 0.97
    "Items": [
      {"Description": "Office chairs × 10", "UnitPrice": 350.00, "Amount": 3500.00},
      {"Description": "Delivery",            "UnitPrice": 120.00, "Amount": 120.00}
    ]
  }
  → No training needed — prebuilt model works out of the box
```

```python
from azure.ai.documentintelligence import DocumentIntelligenceClient
from azure.ai.documentintelligence.models import AnalyzeDocumentRequest
from azure.core.credentials import AzureKeyCredential

client = DocumentIntelligenceClient(
    endpoint=os.getenv("DOCUMENT_INTELLIGENCE_ENDPOINT"),
    credential=AzureKeyCredential(os.getenv("DOCUMENT_INTELLIGENCE_KEY"))
)

def extract_invoice(pdf_bytes: bytes) -> dict:
    poller = client.begin_analyze_document(
        model_id="prebuilt-invoice",
        body=AnalyzeDocumentRequest(bytes_source=pdf_bytes),
        content_type="application/octet-stream"
    )
    result = poller.result()

    invoices = []
    for invoice in result.documents:
        fields = invoice.fields

        def get_field(name):
            field = fields.get(name)
            if not field:
                return None, 0.0
            return field.value_string or field.value_date or field.value_number, field.confidence

        vendor_name, vendor_conf      = get_field("VendorName")
        invoice_date, date_conf       = get_field("InvoiceDate")
        invoice_total, total_conf     = get_field("InvoiceTotal")
        purchase_order, po_conf       = get_field("PurchaseOrder")

        items = []
        if "Items" in fields:
            for item in fields["Items"].value_array or []:
                item_fields = item.value_object or {}
                items.append({
                    "description": (item_fields.get("Description", {}).value_string or ""),
                    "quantity":    (item_fields.get("Quantity", {}).value_number or 1),
                    "unit_price":  (item_fields.get("UnitPrice", {}).value_number or 0),
                    "amount":      (item_fields.get("Amount", {}).value_number or 0),
                    "confidence":  min([f.confidence for f in item_fields.values() if f.confidence])
                })

        invoices.append({
            "vendor_name":    vendor_name,
            "invoice_date":   str(invoice_date) if invoice_date else None,
            "invoice_total":  invoice_total,
            "purchase_order": purchase_order,
            "line_items":     items,
            "low_confidence_fields": [
                k for k, v in {
                    "vendor_name": vendor_conf,
                    "invoice_date": date_conf,
                    "invoice_total": total_conf
                }.items() if v and v < 0.8   # Flag for human review
            ]
        })
    return invoices
```

```bash
# Test prebuilt model with a URL (no code — quick validation)
az rest --method POST \
  --url "https://<endpoint>.cognitiveservices.azure.com/documentintelligence/documentModels/prebuilt-invoice:analyze?api-version=2024-07-31-preview" \
  --headers "Ocp-Apim-Subscription-Key=<key>" "Content-Type=application/json" \
  --body '{"urlSource": "https://raw.githubusercontent.com/Azure-Samples/cognitive-services-REST-api-samples/master/curl/form-recognizer/sample-invoice.pdf"}'
# Returns operation-location header with job ID for polling
```

> ⚠️ **Gotcha:** Confidence scores below 0.8 indicate the model is uncertain — common for handwritten corrections, non-standard layouts, or poor scan quality. Always check `confidence` per field and flag low-confidence extractions for human review rather than auto-approving them. Blindly trusting all extractions causes costly payment errors.

> 💡 **Deep dive hint:** Prebuilt models vs custom models — prebuilt covers invoices, receipts, W2s, ID documents. For proprietary forms (insurance claims, loan applications, regulatory filings) with non-standard layouts, use custom extraction models trained on your own labeled examples.

---

## Q2. How do you train a custom Document Intelligence model for proprietary forms?

**Scenario:** Your insurance company processes 300 types of claims forms. Prebuilt models don't understand your custom fields: "Claimant Policy Number", "Adjuster Code", "Peril Category". You need a custom extraction model.

```
Custom model training pipeline:

  Step 1: Label (Azure Document Intelligence Studio)
  ─────────────────────────────────────────────────
  Upload 5–10 sample forms to ADLS Gen2
  Label fields: draw bounding box → assign field name
  Min: 5 labeled documents; Recommended: 50+ for robustness

  Step 2: Train
  ─────────────────────────────────────────────────
  az rest → POST /documentModels:build
  Takes: 5–30 minutes depending on dataset size
  Output: custom model ID (e.g., "claims-form-v1")

  Step 3: Evaluate
  ─────────────────────────────────────────────────
  Test with 20 holdout labeled documents
  Check: field-level accuracy, confidence distribution
  Target: > 90% accuracy on all critical fields

  Step 4: Deploy + Monitor
  ─────────────────────────────────────────────────
  Use custom model ID just like prebuilt models
  Monitor confidence trends — retrain when accuracy drops
```

```bash
# Step 1 — Create Document Intelligence resource
az cognitiveservices account create \
  --name docint-prod-cc \
  --resource-group rg-ai \
  --location canadaeast \
  --kind FormRecognizer \
  --sku S0

# Step 2 — Store labeled training data in ADLS Gen2
# Structure:
# training-data/
#   form-001.pdf
#   form-001.pdf.labels.json   ← label file (from DI Studio)
#   form-002.pdf
#   form-002.pdf.labels.json
#   fields.json                ← field definitions

# Grant Document Intelligence resource access to training storage
DOCINT_MI=$(az cognitiveservices account show \
  --name docint-prod-cc -g rg-ai \
  --query identity.principalId -o tsv)
az role assignment create \
  --role "Storage Blob Data Reader" \
  --assignee $DOCINT_MI \
  --scope $ADLS_RESOURCE_ID

# Step 3 — Build custom model
az rest --method POST \
  --url "https://docint-prod-cc.cognitiveservices.azure.com/documentintelligence/documentModels:build?api-version=2024-07-31-preview" \
  --headers "Ocp-Apim-Subscription-Key=<KEY>" "Content-Type=application/json" \
  --body '{
    "modelId": "claims-form-v2",
    "description": "Insurance claims form extraction model v2",
    "buildMode": "neural",
    "azureBlobSource": {
      "containerUrl": "https://adlsprodcc.blob.core.windows.net/training-data",
      "prefix": "claims-forms/"
    }
  }'
# Poll operation-location header for completion
```

```python
# Step 4 — Use custom model identically to prebuilt
poller = client.begin_analyze_document(
    model_id="claims-form-v2",    # Your custom model ID
    body=AnalyzeDocumentRequest(bytes_source=pdf_bytes),
    content_type="application/octet-stream"
)
result = poller.result()

for doc in result.documents:
    # Access your custom-labeled fields
    policy_number = doc.fields.get("ClaimantPolicyNumber")
    adjuster_code = doc.fields.get("AdjusterCode")
    peril_category = doc.fields.get("PerilCategory")

    print(f"Policy: {policy_number.value_string}, "
          f"Confidence: {policy_number.confidence:.2f}")
```

> ⚠️ **Gotcha:** `buildMode: "neural"` (recommended) requires minimum 10 training documents with 200 labeled field values across the set. `buildMode: "template"` needs only 5 but only works for fixed-layout forms where fields are always in the same position. Neural mode handles varied layouts — use it for forms that come from multiple vendors or templates.

---

## Q3. How do you process 10,000 documents per day through Document Intelligence at scale?

**Scenario:** An ADF pipeline drops batches of scanned insurance claims into a blob container. Peak volume is 10,000 PDFs per day. Sequential processing takes 28 hours. You need it done in under 4 hours.

```
High-volume Document Intelligence pipeline:

  ADLS Gen2 (raw/claims/)
       │ Blob trigger → Event Grid → Service Bus queue
       │ (one message per PDF)
       ▼
  Service Bus Queue (sbq-claims-processing)
       │ depth = number of unprocessed documents
       ▼
  AKS + KEDA
       │ KEDA scales pods based on queue depth
       │ 0 docs → 0 pods | 10K docs → 50 pods
       ▼
  Worker pods (each pod processes 1 document at a time)
       │ Document Intelligence API (parallel, up to 15 concurrent per resource)
       │ Rate: ~200 pages/min per resource
       ▼
  Results → CosmosDB (extracted fields)
          → ADLS Gen2 (original + JSON output, side-by-side)
          → Service Bus (success/failure notification)

  For 10K docs × avg 5 pages = 50K pages:
  50K ÷ 200 pages/min = 250 min → need 2 Document Intelligence resources
  2 resources × 50 pods = 4,500 pages/min = ~11 minutes ✅
```

```python
# Worker pod: process one document from Service Bus
import asyncio
from azure.servicebus.aio import ServiceBusClient
from azure.ai.documentintelligence.aio import DocumentIntelligenceClient

async def process_claims_worker():
    async with ServiceBusClient.from_connection_string(SB_CONNECTION_STRING) as sb_client:
        async with sb_client.get_queue_receiver(
            queue_name="sbq-claims-processing",
            max_wait_time=30
        ) as receiver:
            async for message in receiver:
                try:
                    payload = json.loads(str(message))
                    blob_url = payload["blob_url"]

                    # Download PDF from ADLS
                    pdf_bytes = await download_blob(blob_url)

                    # Extract via Document Intelligence (async client)
                    async with DocumentIntelligenceClient(
                        endpoint=DI_ENDPOINT,
                        credential=AzureKeyCredential(DI_KEY)
                    ) as di_client:
                        poller = await di_client.begin_analyze_document(
                            model_id="claims-form-v2",
                            body=AnalyzeDocumentRequest(bytes_source=pdf_bytes),
                            content_type="application/octet-stream"
                        )
                        result = await poller.result()

                    # Write results to CosmosDB
                    await write_results(payload["claim_id"], result)

                    # Acknowledge message (remove from queue)
                    await receiver.complete_message(message)

                except Exception as e:
                    # Dead-letter after 3 delivery attempts
                    if message.delivery_count >= 3:
                        await receiver.dead_letter_message(message, reason=str(e))
                    else:
                        await receiver.abandon_message(message)   # Retry
```

```bash
# Monitor Document Intelligence throttling (check if hitting rate limits)
az monitor metrics list \
  --resource "/subscriptions/<SUB>/resourceGroups/rg-ai/providers/Microsoft.CognitiveServices/accounts/docint-prod-cc" \
  --metric "TotalErrors,SuccessfulCalls" \
  --interval PT5M \
  --output table
# If TotalErrors spikes with 429s: add a second DI resource and load-balance
```

> ⚠️ **Gotcha:** Document Intelligence S0 tier allows **15 concurrent requests** per resource. Sending 50 concurrent requests results in 35 immediate 429 errors. Implement a semaphore per Document Intelligence endpoint (`asyncio.Semaphore(12)` — leave headroom) and distribute load across multiple DI resources if processing at high concurrency.

---

## Q4. How do you validate OCR accuracy and handle poor-quality scans?

**Scenario:** 8% of uploaded claim forms are faxed or photographed at low resolution. Document Intelligence returns confidence 0.2–0.4 on these. Auto-approving them causes claims to be processed with wrong amounts.

```
Confidence-based routing:

  Document Intelligence output
       │
       ├── All fields confidence > 0.85
       │    → Auto-approve → CosmosDB → downstream processing ✅
       │
       ├── Some fields confidence 0.6–0.85
       │    → Flag for review → queue to human review portal
       │    → Reviewer confirms or corrects flagged fields
       │
       └── Any field confidence < 0.6 OR page_quality = "low"
            → Reject with reason → notify submitter
            → "Please resubmit a higher quality scan"
            → SLA: resubmit within 2 business days

  Confidence thresholds (adjust per field risk):
  ClaimAmount:      > 0.90  (financial — high stakes)
  PolicyNumber:     > 0.88  (lookup key — must match)
  ClaimantName:     > 0.80  (important but correctable)
  AdjusterNotes:    > 0.70  (descriptive — lower stakes)
```

```python
def route_by_confidence(extracted: dict, thresholds: dict) -> dict:
    """Route extraction result based on per-field confidence thresholds."""

    low_confidence_fields = []
    for field_name, threshold in thresholds.items():
        field = extracted.get(field_name)
        if field and field.confidence < threshold:
            low_confidence_fields.append({
                "field": field_name,
                "value": field.value_string or field.value_number,
                "confidence": round(field.confidence, 3),
                "threshold": threshold
            })

    if not low_confidence_fields:
        return {"routing": "auto_approve", "issues": []}

    # Check if any critical fields are low confidence
    critical_fields = {"ClaimAmount", "PolicyNumber"}
    has_critical_issue = any(
        f["field"] in critical_fields for f in low_confidence_fields
    )

    return {
        "routing": "reject" if has_critical_issue else "human_review",
        "issues": low_confidence_fields,
        "action": (
            "Resubmit high-quality scan" if has_critical_issue
            else "Human reviewer required"
        )
    }

# Check overall page quality
def assess_scan_quality(result) -> str:
    pages = result.pages or []
    avg_word_confidence = sum(
        word.confidence
        for page in pages
        for word in (page.words or [])
    ) / max(sum(len(page.words or []) for page in pages), 1)

    if avg_word_confidence < 0.6:
        return "poor"
    elif avg_word_confidence < 0.8:
        return "medium"
    return "high"
```

> ⚠️ **Gotcha:** Document Intelligence returns confidence per word/field but not a single overall document confidence score. A document with 98 high-confidence fields and 1 zero-confidence field (e.g., a handwritten amount) can pass a naive average-confidence check. Always check the **minimum** confidence among critical fields, not the average.
