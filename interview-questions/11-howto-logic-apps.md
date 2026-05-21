# Azure Logic Apps — How-To Guides

> Step-by-step guides for common Logic Apps scenarios.
> Includes: large file handling, chunking, error handling, approvals, database triggers, retry patterns.

---

## Guide 1: How to Send Large Files Using Logic Apps

### Problem
Logic Apps has message size limits:
- Standard tier: 100MB per message
- Consumption tier: 100MB per message (connector-dependent)
- Email attachments: typically 25MB for Outlook connector
- HTTP action body: 100MB default

For files larger than these limits, you must use chunking.

---

### Solution A: Blob Storage + SAS URL (Best for large files)

Instead of attaching the file, store it in Azure Blob and share a link.

**Trigger:** HTTP Request, Schedule, or any event

**Steps:**

```
Step 1: Upload file to Azure Blob Storage
  Action: "Create blob (V2)" — Azure Blob Storage connector
    Storage account: prod-storage
    Container: outgoing-files
    Blob name: @{triggerBody()?['filename']}
    Content: @{triggerBody()?['fileContent']}

Step 2: Generate SAS URL (time-limited access)
  Action: "Create SAS URI by path" — Azure Blob Storage connector
    Storage account: prod-storage
    Blob path: @{body('Create_blob')?['Path']}
    Permissions: Read
    Expiry time: @{addHours(utcNow(), 72)}    -- Link valid for 72 hours

Step 3: Send email with download link (not attachment)
  Action: "Send an email (V2)" — Office 365 Outlook connector
    To: @{triggerBody()?['recipientEmail']}
    Subject: "Your file is ready for download"
    Body: |
      Hello,
      
      Your file "@{triggerBody()?['filename']}" is ready.
      
      Download link (valid for 72 hours):
      @{body('Create_SAS_URI_by_path')?['WebUrl']}
      
      If the link has expired, please request a new one.
```

**Why this is better than attaching large files:**
- No size limit on email body (just a URL)
- Recipient doesn't need to download immediately
- You control access (SAS expiry + revocation)
- Audit trail: Blob access logs show who downloaded and when

---

### Solution B: Chunked HTTP Transfer (for streaming large files)

When you must transfer large files via HTTP (e.g., to an external API), use chunked transfer encoding.

**Logic Apps Standard (Stateful workflow) — enables large file streaming:**

```json
// Enable chunked transfer in HTTP action settings
{
  "type": "Http",
  "inputs": {
    "method": "POST",
    "uri": "https://partner-api.company.com/upload",
    "body": "@triggerBody()",
    "headers": {
      "Transfer-Encoding": "chunked",
      "Content-Type": "application/octet-stream"
    }
  },
  "runtimeConfiguration": {
    "contentTransfer": {
      "transferMode": "Chunked"
    }
  }
}
```

**Enabling chunked transfer on triggers and actions:**
- In Logic Apps Standard: enable in action settings → "Allow chunking"
- Maximum supported chunk size: 50MB
- Logic Apps orchestrates the chunking automatically

---

### Solution C: Split Large File into Parts (Manual Chunking)

When the recipient API does not support chunked transfer, split the file into parts.

```
[Variables initialization]
Initialize variable: chunkSize = 5242880   -- 5MB in bytes
Initialize variable: offset = 0
Initialize variable: partNumber = 1
Initialize variable: totalSize = @{length(triggerBody()?['fileContent'])}

[Until loop: send all chunks]
Until: @{greaterOrEquals(variables('offset'), variables('totalSize'))}

  Step 1: Calculate chunk end
    Set variable: chunkEnd = @{min(add(variables('offset'), variables('chunkSize')), variables('totalSize'))}

  Step 2: Extract chunk
    Set variable: currentChunk = 
      @{substring(base64ToString(triggerBody()?['fileContent']), 
                  variables('offset'), 
                  sub(variables('chunkEnd'), variables('offset')))}

  Step 3: Upload chunk to external API
    Action: HTTP POST
    URI: https://external-api.com/upload/chunks
    Headers:
      X-Chunk-Index: @{variables('partNumber')}
      X-Total-Chunks: @{div(add(variables('totalSize'), sub(variables('chunkSize'), 1)), variables('chunkSize'))}
      X-Upload-Id: @{guid()}
    Body: @{variables('currentChunk')}

  Step 4: Advance offset
    Set variable: offset = @{variables('chunkEnd')}
    Set variable: partNumber = @{add(variables('partNumber'), 1)}

[After loop] Complete upload
  Action: HTTP POST
  URI: https://external-api.com/upload/complete
  Body: { "uploadId": "...", "totalParts": @{variables('partNumber')} }
```

---

## Guide 2: How to Implement Retry Logic with Exponential Backoff

### Problem
External APIs fail intermittently. Logic Apps default retry is fixed interval — better to use exponential backoff.

### Solution

**Configure retry policy on any action:**

```json
// In action "Run After" settings → Retry Policy
{
  "retryPolicy": {
    "type": "exponential",
    "count": 5,
    "interval": "PT5S",       // Initial interval: 5 seconds
    "minimumInterval": "PT5S", // Min wait: 5 seconds
    "maximumInterval": "PT1H"  // Max wait: 1 hour
  }
}
```

**Exponential backoff intervals:**
- Attempt 1: 5 seconds
- Attempt 2: ~10 seconds
- Attempt 3: ~20 seconds
- Attempt 4: ~40 seconds
- Attempt 5: ~80 seconds

**Dead letter pattern for permanent failures:**

```
HTTP Action (with retry policy)
  ├── On Success → Continue workflow
  └── On Failure (after all retries) → 
        Send failure event to Service Bus dead-letter queue
        Log failure details to Log Analytics
        Send PagerDuty/Teams alert
        Terminate workflow gracefully (not with error)
```

---

## Guide 3: How to Build a Human Approval Workflow

### Use case
A document arrives → AI classifies it → if confidence < 80%, send to human for review.

### Steps

```
Trigger: HTTP Request (document received)

Step 1: Call Azure Document Intelligence
  Action: HTTP POST to Document Intelligence endpoint
  Output: extracted fields + confidence scores

Step 2: Check confidence threshold
  Condition: @{less(body('DocIntelligence')?['confidence'], 0.80)}
  
  [If confidence < 80% — needs human review]
  
    Step 3: Send approval request
      Action: "Send approval email" — Office 365 connector
        To: approver@company.com
        Subject: "Document review required: @{triggerBody()?['documentName']}"
        User options: Approve | Reject | Request more info
        Approval type: First to respond
        Timeout: PT48H   -- 48 hour timeout
      
      Step 4a: If Approved
        Set variable: reviewOutcome = "approved"
        Proceed to processing pipeline
        
      Step 4b: If Rejected
        Send notification to document submitter
        Store rejection reason in CosmosDB
        
      Step 4c: If Timed out (48hr no response)
        Escalate to manager
        Repeat approval request with higher-priority email
  
  [If confidence >= 80% — auto-approve]
    Proceed directly to processing pipeline

Step 5: Audit log
  Action: Insert document review record to CosmosDB
    Fields: documentId, reviewType (auto/manual), outcome, reviewer, timestamp, confidence
```

---

## Guide 4: How to Trigger Logic Apps from Azure Data Factory

### Use case
ADF pipeline completes (success or failure) → Logic App sends notification or triggers downstream workflow.

### Step 1: Create Logic App with HTTP Trigger

```json
// Logic App workflow: DataPipelineNotification
Trigger: "When a HTTP request is received"
  Request Body JSON Schema:
  {
    "type": "object",
    "properties": {
      "pipelineName": { "type": "string" },
      "runId": { "type": "string" },
      "status": { "type": "string" },
      "errorMessage": { "type": "string" },
      "rowsCopied": { "type": "number" },
      "duration": { "type": "string" }
    }
  }
```

### Step 2: Add Web Activity in ADF

```json
// ADF Web Activity (on pipeline success/failure)
{
  "name": "NotifyLogicApp",
  "type": "WebActivity",
  "dependsOn": [
    {
      "activity": "CopyData",
      "dependencyConditions": ["Succeeded", "Failed"]
    }
  ],
  "typeProperties": {
    "url": "@{linkedService('LogicAppEndpoint').properties.typeProperties.url}",
    "method": "POST",
    "headers": { "Content-Type": "application/json" },
    "body": {
      "pipelineName": "@pipeline().Pipeline",
      "runId": "@pipeline().RunId",
      "status": "@activity('CopyData').Status",
      "errorMessage": "@activity('CopyData').Error.Message",
      "rowsCopied": "@activity('CopyData').output.rowsCopied",
      "duration": "@activity('CopyData').Duration"
    }
  }
}
```

### Step 3: Logic App notification actions

```
Parse JSON (from HTTP trigger body)

Condition: @{equals(triggerBody()?['status'], 'Succeeded')}

If Succeeded:
  → Send Teams notification: "Pipeline @{triggerBody()?['pipelineName']} completed. 
                              @{triggerBody()?['rowsCopied']} rows copied."
  → Insert success audit record to SQL

If Failed:
  → Send PagerDuty alert (HTTP action to PagerDuty Events API)
  → Send Teams alert with @mention to on-call DBA
  → Create ServiceNow incident (HTTP action)
  → Insert failure record to SQL audit table
```

---

## Guide 5: How to Process Messages from Azure Service Bus in Logic Apps

### Use case
Orders arrive in Service Bus → Logic App processes each order → calls ERP API → updates database.

### Steps

```
Trigger: "When a message is received in a queue (peek-lock)"
  Service Bus namespace: prod-servicebus
  Queue: orders-queue
  Session enabled: false

Step 1: Parse order message
  Action: Parse JSON
  Content: @{base64ToString(triggerBody()?['ContentData'])}
  Schema: { "orderId": string, "customerId": string, "items": array, "amount": number }

Step 2: Check for duplicate processing (idempotency)
  Action: HTTP GET to idempotency check service
  URI: https://orders-api.internal/api/orders/@{body('Parse_JSON')?['orderId']}/status
  
  Condition: @{equals(body('IdempotencyCheck')?['status'], 'processed')}
  If already processed → skip to Step 5 (complete message)

Step 3: Call ERP API
  Action: HTTP POST
  URI: https://erp.company.com/api/orders
  Retry policy: exponential, count: 3
  Body: @{body('Parse_JSON')}
  
  On failure → DO NOT complete message → let it return to queue (or dead-letter after max deliveries)

Step 4: Update database
  Action: Execute stored procedure (SQL Server)
  Procedure: sp_MarkOrderProcessed
  Parameters: orderId, erpReference, processedAt

Step 5: Complete message (remove from queue — prevents reprocessing)
  Action: "Complete the message in a queue"
  Lock token: @{triggerBody()?['LockToken']}

[Error handling]
  If any step fails (Step 3 or 4):
    → Abandon message (returns to queue for retry)
    → After max deliveries (default 10): Service Bus moves to dead-letter queue
    → Dead-letter queue monitored by separate Logic App → alert + investigation
```

---

## Guide 6: How to Call Azure Functions from Logic Apps and Handle Errors

### When to use Azure Functions instead of built-in actions

| Task | Use Logic App Action | Use Azure Function |
|------|---------------------|-------------------|
| Send email | Office 365 action | No |
| Simple HTTP call | HTTP action | No |
| Complex calculation or transformation | No | Yes |
| Calling .NET library | No | Yes |
| Custom checksum validation | No | Yes |
| Parsing complex file formats | No | Yes |

### Steps

```
Step 1: Create Azure Function (HTTP trigger)

[C# example — validates CSV file integrity]
[HttpTrigger(AuthorizationLevel.Function, "post")]
public static async Task<IActionResult> ValidateFile(HttpRequest req)
{
    var body = await new StreamReader(req.Body).ReadToEndAsync();
    var request = JsonConvert.DeserializeObject<ValidationRequest>(body);
    
    var checksum = ComputeMD5(request.FileContent);
    var rowCount = CountRows(request.FileContent);
    
    return new OkObjectResult(new {
        checksum = checksum,
        rowCount = rowCount,
        isValid = checksum == request.ExpectedChecksum
    });
}

Step 2: Call Function from Logic App
  Action: "Azure Functions — Call an Azure function"
    Function App: prod-functions
    Function: ValidateFile
    Request Body:
    {
      "fileContent": "@{body('ReadFile')?['content']}",
      "expectedChecksum": "@{triggerBody()?['expectedChecksum']}"
    }

Step 3: Handle function response
  Condition: @{equals(body('Call_Azure_Function')?['isValid'], true)}
  If valid → continue processing
  If invalid → 
    Send alert
    Store failure record
    Return HTTP 422 to caller
```

---

## Guide 7: How to Implement a Scheduled Data Export with Logic Apps

### Use case
Every weekday at 6 AM, export previous day's transactions from SQL Database to ADLS Gen2 as CSV, then trigger a downstream ADF pipeline.

```
Trigger: Recurrence
  Frequency: Week
  Interval: 1
  Days: Monday, Tuesday, Wednesday, Thursday, Friday
  At: 06:00
  Timezone: Eastern Standard Time

Step 1: Calculate date range
  Initialize variable: startDate = @{formatDateTime(addDays(utcNow(), -1), 'yyyy-MM-dd')}
  Initialize variable: endDate   = @{formatDateTime(utcNow(), 'yyyy-MM-dd')}

Step 2: Query database
  Action: Execute SQL Query (SQL Server connector)
  Query: |
    SELECT TransactionId, AccountId, Amount, Currency, TransactionDate, Description
    FROM dbo.Transactions
    WHERE CAST(TransactionDate AS DATE) = '@{variables('startDate')}'
    ORDER BY TransactionDate ASC
  
  Returns: array of rows

Step 3: Convert to CSV
  Action: Create CSV table
    From: @{body('Execute_SQL_Query')?['resultsets']?['Table1']}
    Columns: Automatic (uses SQL column names as headers)

Step 4: Upload to ADLS Gen2
  Action: Create blob (V2) — Azure Blob Storage
    Storage account: prod-adls
    Container: exports
    Blob name: transactions/@{variables('startDate')}/transactions_@{formatDateTime(utcNow(), 'yyyyMMdd_HHmmss')}.csv
    Content: @{body('Create_CSV_table')}
    Content type: text/csv

Step 5: Trigger ADF pipeline
  Action: HTTP POST (ADF REST API)
  URI: https://management.azure.com/subscriptions/{sub}/resourceGroups/{rg}/
       providers/Microsoft.DataFactory/factories/{factory}/pipelines/{pipeline}/createRun?api-version=2018-06-01
  Authentication: Managed Identity
  Body: { "exportDate": "@{variables('startDate')}" }

Step 6: Send completion notification
  Action: Send Teams message
  Body: "Daily transaction export complete. 
         Date: @{variables('startDate')}
         Rows: @{length(body('Execute_SQL_Query')?['resultsets']?['Table1'])}
         File: transactions/@{variables('startDate')}/"
```

---

## Guide 8: How to Handle Long-Running Operations with Logic Apps Async Pattern

### Problem
Some operations (e.g., large document processing, video transcoding) take minutes to hours. Logic Apps HTTP timeout = 120 seconds. For long-running backends, use the async polling pattern.

### Solution: 202 Accepted + Polling

```
Step 1: Start the long-running operation
  Action: HTTP POST
  URI: https://processing-api.company.com/jobs
  Body: { "documentUrl": "...", "processingType": "ocr" }
  
  Backend responds immediately with:
  HTTP 202 Accepted
  Location: https://processing-api.company.com/jobs/job-123/status
  Retry-After: 30

Step 2: Store job ID
  Initialize variable: jobStatusUrl = @{outputs('Start_Job')['headers']['Location']}
  Initialize variable: jobStatus = "pending"

Step 3: Poll until complete
  Until: @{not(equals(variables('jobStatus'), 'pending'))}
  Limit: Count: 60, Timeout: PT2H   -- Max 2 hours
  
    3a: Wait
      Delay: 30 seconds (match Retry-After header)
    
    3b: Check status
      Action: HTTP GET
      URI: @{variables('jobStatusUrl')}
      
    3c: Update status variable
      Set variable: jobStatus = @{body('Check_Status')?['status']}
      -- Values: "pending", "processing", "completed", "failed"

Step 4: Handle result
  Condition: @{equals(variables('jobStatus'), 'completed')}
  
  If completed:
    → Download result: HTTP GET to result URL
    → Process result data
    → Notify requester
  
  If failed:
    → Log error: @{body('Check_Status')?['errorMessage']}
    → Send alert
    → Trigger retry or escalation

Step 5: Timeout handling (if Until loop exits due to timeout)
  → Send timeout alert
  → Log job ID for manual investigation
  → Return timeout response to original caller
```

**Logic Apps Standard (Durable): preferred for very long operations**

Logic Apps Standard supports Durable Functions patterns natively:
- Checkpoint and replay state
- Unlimited timeout (no 2-hour limit)
- Built-in external event waiting (human approval, webhook callbacks)
- Use for: multi-day approval workflows, long ETL jobs, complex orchestrations
