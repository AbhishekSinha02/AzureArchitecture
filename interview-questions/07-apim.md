# Azure API Management (APIM) — Interview Questions

> Covers: APIM tiers, policies, versioning, OAuth, rate limiting, APIM with AKS, monetization.
> Levels: Beginner → Intermediate → Advanced

---

## BEGINNER

**Q1. What is Azure API Management and what problems does it solve?**

**Answer:**

Azure API Management (APIM) is a managed API gateway that sits between API consumers and backend services. It provides:

| Problem | APIM solution |
|---------|--------------|
| No authentication on APIs | Subscription keys, OAuth 2.0, client certificates |
| No rate limiting → abuse/DoS | Rate limiting and quota policies |
| API consumers need docs | Developer portal (auto-generated API docs) |
| Backend URLs change | Abstract backends behind stable API URLs |
| Different API versions in use | API versioning + revision management |
| No visibility into API usage | Built-in analytics, Application Insights integration |
| Transforming request/response | Inbound/outbound policy pipelines |

**APIM tiers:**

| Tier | Use case | VNet? | Replicas | Notes |
|------|----------|-------|----------|-------|
| Consumption | Serverless, dev/test, low traffic | No | N/A | Pay per call, scales to zero |
| Developer | Dev/test, not for production | Yes | 1 | No HA SLA |
| Basic | Light production | No | 2 | |
| Standard | Standard production | No | 4 | |
| Premium | Enterprise, multi-region | Yes | Custom | VNet injection, multi-region |
| Basic v2 / Standard v2 | Modern tiers with VNet + faster provisioning | Yes | Custom | New generation — prefer over legacy |

---

**Q2. What is an APIM Policy and how is the policy pipeline structured?**

**Answer:**

APIM Policies are XML snippets that execute on requests and responses. They form a four-phase pipeline:

```xml
<policies>
    <inbound>
        <!-- Runs BEFORE request hits backend -->
        <!-- Auth, rate limiting, request transformation, caching check -->
        <base />
    </inbound>
    <backend>
        <!-- Controls how APIM forwards request to backend -->
        <!-- Load balancing, retry, circuit breaker -->
        <base />
    </backend>
    <outbound>
        <!-- Runs AFTER backend response, BEFORE client receives it -->
        <!-- Response transformation, header stripping, caching store -->
        <base />
    </outbound>
    <on-error>
        <!-- Executes when any other section throws an error -->
        <!-- Custom error responses, logging -->
        <base />
    </on-error>
</policies>
```

**Policy scope (where you can apply):**
1. **Global** — applies to ALL APIs in the APIM instance
2. **Product** — applies to all APIs in a product (e.g., "Banking API Product")
3. **API** — applies to all operations in a specific API
4. **Operation** — applies to one specific endpoint

`<base />` means "execute the parent scope's policy here." Position determines execution order.

---

**Q3. How do you authenticate callers in APIM?**

**Answer:**

**Method 1: Subscription Keys (simplest)**
```http
GET /api/products
Ocp-Apim-Subscription-Key: abc123def456...
```
- Built into APIM — enable on product
- Good for: machine-to-machine, simple auth
- Limitation: key rotation requires coordination, not user-identity-aware

**Method 2: OAuth 2.0 / JWT (most common for production)**
```xml
<inbound>
    <validate-jwt header-name="Authorization" failed-validation-httpcode="401">
        <openid-config url="https://login.microsoftonline.com/{tenant}/.well-known/openid-configuration"/>
        <required-claims>
            <claim name="aud">
                <value>api://your-app-id</value>
            </claim>
        </required-claims>
    </validate-jwt>
</inbound>
```
- APIM validates JWT against Entra ID JWKS endpoint
- Rejects invalid/expired tokens before touching backend
- Backend receives authenticated request (no re-auth needed)

**Method 3: Client Certificates (mutual TLS)**
```xml
<inbound>
    <choose>
        <when condition="@(context.Request.Certificate == null 
            || !context.Request.Certificate.Verify()
            || context.Request.Certificate.Thumbprint != '{{expected-thumbprint}}')">
            <return-response>
                <set-status code="403" reason="Forbidden"/>
            </return-response>
        </when>
    </choose>
</inbound>
```
- Used for: high-security B2B APIs, legacy systems, banking partner integrations

---

## INTERMEDIATE

**Q4. How do you implement rate limiting and quota policies in APIM?**

**Answer:**

**Rate Limit (per time window — prevents bursts):**
```xml
<!-- Allow 100 calls per 60 seconds per subscription key -->
<rate-limit calls="100" renewal-period="60" />

<!-- Per-user rate limit (based on JWT claim) -->
<rate-limit-by-key calls="50"
    renewal-period="60"
    counter-key="@(context.Request.Headers.GetValueOrDefault('Authorization','')
        .Split(' ').Last()
        .AsJwt()?.Claims?.GetValueOrDefault('sub','anonymous'))" />
```

**Quota (total over a period — business-level throttling):**
```xml
<!-- Allow 10,000 calls per 30 days per subscription -->
<quota calls="10000" bandwidth="100" renewal-period="2592000" />
```

**Difference:**
- `rate-limit`: prevent bursts (e.g., 100/min) — returns 429 immediately when exceeded
- `quota`: enforce business usage caps (e.g., 10K/month) — good for API monetization tiers

**Retry-After header (good practice):**
```xml
<on-error>
    <choose>
        <when condition="@(context.Response.StatusCode == 429)">
            <set-header name="Retry-After" exists-action="override">
                <value>60</value>
            </set-header>
        </when>
    </choose>
</on-error>
```

**Granularity options:**
- Per subscription (default)
- Per IP: `counter-key="@(context.Request.IpAddress)"`
- Per user (JWT claim): `counter-key="@(context.Request.Headers['Authorization'].AsJwt().Subject)"`
- Per product, per operation

---

**Q5. How do you implement API versioning in APIM?**

**Answer:**

APIM supports three versioning schemes:

**1. URL path versioning (most visible, most common):**
```
https://api.company.com/v1/orders
https://api.company.com/v2/orders
```

**2. Query string versioning:**
```
https://api.company.com/orders?api-version=2024-01-01
```

**3. Header versioning:**
```
GET /orders
Api-Version: 2024-01-01
```

**APIM versioning concepts:**

| Concept | Definition |
|---------|-----------|
| **Version** | A distinct API version consumers explicitly choose (breaking changes) |
| **Revision** | A non-breaking change to the current version (internal, can be made current) |

**Best practice workflow:**
```
1. Create new revision of API v1 for minor changes (won't affect live consumers)
2. Test the revision → promote to "Current" revision
3. When breaking changes needed:
   → Create new API version (v2) based on current v1
   → Publish both v1 and v2 simultaneously
   → Give consumers a deprecation timeline (e.g., 6 months)
   → Add deprecation notice in Developer Portal
   → After deadline: retire v1
```

**Deprecation policy in APIM:**
```xml
<!-- In outbound policy of deprecated API version -->
<set-header name="Sunset" exists-action="override">
    <value>Sat, 31 Dec 2024 23:59:59 GMT</value>
</set-header>
<set-header name="Deprecation" exists-action="override">
    <value>true</value>
</set-header>
<set-header name="Link" exists-action="override">
    <value>&lt;https://api.company.com/v2/orders&gt;; rel="successor-version"</value>
</set-header>
```

---

**Q6. How do you integrate APIM with an AKS backend securely?**

**Answer:**

**Architecture: APIM (internal VNet mode) → AKS (private cluster)**

```
Internet
  │
  ▼
Azure Front Door (WAF + DDoS)
  │
  ▼
APIM (Premium, internal VNet injection)
  ├── VNet: hub-vnet, subnet: snet-apim (10.0.1.0/24)
  └── Private IP only (no public IP in internal mode)
  │
  ▼
AKS Internal Load Balancer
  ├── VNet: spoke-aks-vnet
  ├── NGINX Ingress Controller (internal LB: 10.1.0.10)
  └── Services exposed on internal IP only
```

**APIM backend configuration:**
```xml
<!-- APIM Backend definition pointing to AKS internal LB -->
<backend id="aks-orders-api">
    <url>http://10.1.0.10/orders</url>
    <tls>
        <validate-certificate-name>false</validate-certificate-name>
    </tls>
    <circuit-breaker>
        <rules>
            <rule name="aks-circuit-breaker" trip-duration="PT30S">
                <conditions>
                    <status-code-range start="500" end="599" />
                </conditions>
                <trip-duration>PT30S</trip-duration>
                <accept-retry-after>true</accept-retry-after>
            </rule>
        </rules>
    </circuit-breaker>
</backend>
```

**APIM → AKS mutual TLS (recommended for production):**
```xml
<!-- APIM presents client certificate to AKS backend -->
<inbound>
    <authentication-certificate thumbprint="{{backend-cert-thumbprint}}" />
</inbound>
```

**AKS Ingress restricts traffic to APIM only:**
```yaml
# NetworkPolicy: only allow traffic from APIM subnet
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-apim-only
spec:
  podSelector:
    matchLabels:
      app: orders-api
  ingress:
  - from:
    - ipBlock:
        cidr: 10.0.1.0/24  # APIM subnet CIDR
    ports:
    - protocol: TCP
      port: 8080
```

---

## ADVANCED

**Q7. Design an APIM architecture for a multi-tenant financial services API platform with 50 enterprise clients.**

**Answer:**

**Requirements:**
- 50 clients (enterprise banks, insurers)
- Each client has different rate limits, SLAs, data access permissions
- Some clients need dedicated capacity
- Full audit trail for regulatory compliance (OSFI)
- API versioning — clients on different versions simultaneously

**Architecture:**

```
APIM Premium (multi-region: Canada Central + Canada East)

PRODUCT TIERS (business-level grouping):
  Product: "Standard" → 10,000 calls/day, 100 calls/min
  Product: "Premium"  → 100,000 calls/day, 1,000 calls/min
  Product: "Enterprise" → unlimited, dedicated throughput

SUBSCRIPTION MANAGEMENT:
  Each client gets:
  → 1 subscription per product tier
  → Subscription key = client identifier for billing/auditing
  → Client-specific JWT audience claim validation

POLICY ARCHITECTURE:
  Global policy:
    → Validate JWT (Entra ID, required on ALL calls)
    → Log all requests to Application Insights (request ID, client ID, operation)
    → Correlation ID propagation

  Product-level policy (per tier):
    → Rate limit per tier
    → Quota per billing cycle

  API-level policy:
    → Request/response transformation
    → Backend circuit breaker
    → Cache-aside for read-heavy operations (5-min TTL)

  Operation-level policy:
    → Fine-grained authorization (specific clients can call specific operations)

CLIENT ISOLATION for top-tier clients:
  → Dedicated APIM Premium unit (capacity units) per large client
  → Private Link from client's Azure VNet to APIM
  → No shared infrastructure

AUDIT LOGGING (OSFI requirement):
  All APIM logs → Event Hub → Log Analytics Workspace
  Log fields: clientId, subscriptionId, operation, timestamp, responseCode, latency
  Retention: 7 years (FINTRAC)
```

**Per-client authorization policy:**
```xml
<inbound>
    <!-- Extract client ID from JWT -->
    <set-variable name="clientId" 
        value="@(context.Request.Headers['Authorization'].AsJwt().Claims['client_id'].FirstOrDefault())" />
    
    <!-- Check if client is authorized for this specific operation -->
    <send-request mode="new" response-variable-name="authCheck" timeout="5">
        <set-url>https://auth-service/api/permissions</set-url>
        <set-method>GET</set-method>
        <set-header name="client-id" exists-action="override">
            <value>@((string)context.Variables["clientId"])</value>
        </set-header>
        <set-header name="operation" exists-action="override">
            <value>@(context.Operation.Id)</value>
        </set-header>
    </send-request>
    
    <choose>
        <when condition="@(((IResponse)context.Variables['authCheck']).StatusCode != 200)">
            <return-response>
                <set-status code="403" reason="Forbidden"/>
            </return-response>
        </when>
    </choose>
</inbound>
```

---

**Q8. How do you implement a circuit breaker pattern in APIM and when does it trigger?**

**Answer:**

**Circuit breaker states:**
- **Closed**: Normal operation, requests flow through
- **Open**: Backend failing, requests fail fast (no backend call)
- **Half-open**: Test period — allow some requests through to check backend recovery

**APIM native circuit breaker (GA in recent versions):**
```xml
<backend id="payment-service-backend">
    <url>https://payment-service.internal</url>
    <circuit-breaker>
        <rules>
            <rule name="payment-circuit-breaker">
                <conditions>
                    <!-- Trip if 50% of requests in 30s window return 5xx -->
                    <percentage-fault threshold="50" interval="PT30S" />
                    <status-code-range start="500" end="599" />
                </conditions>
                <trip-duration>PT1M</trip-duration>   <!-- Open for 1 minute -->
                <accept-retry-after>true</accept-retry-after>
            </rule>
        </rules>
    </circuit-breaker>
</backend>
```

**Custom circuit breaker (policy-based, more control):**
```xml
<inbound>
    <!-- Check circuit state from Redis/Cache -->
    <cache-lookup-value key="circuit:payment-service" variable-name="circuitState" />
    
    <choose>
        <when condition="@((string)context.Variables.GetValueOrDefault("circuitState","closed") == "open")">
            <return-response>
                <set-status code="503" reason="Service temporarily unavailable"/>
                <set-header name="Retry-After" exists-action="override">
                    <value>60</value>
                </set-header>
                <set-body>{"error": "Payment service circuit open. Retry after 60 seconds."}</set-body>
            </return-response>
        </when>
    </choose>
</inbound>

<on-error>
    <!-- If backend fails: increment failure counter, open circuit if threshold exceeded -->
    <cache-store-value key="circuit:payment-service" value="open" duration="60" />
</on-error>
```

**Why this matters in financial services:**
- A slow payment backend is worse than a fast rejection — partial writes are dangerous
- Circuit breaker prevents cascading failures across microservices
- Fail-fast with 503 allows client to queue the request rather than hang
- Retry-After header tells client exactly when to retry (no retry storms)
