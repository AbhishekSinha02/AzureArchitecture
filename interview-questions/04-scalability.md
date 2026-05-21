# Azure Scalability — Interview Questions

> Covers: Autoscaling, KEDA, AKS node pools, partitioning, caching strategies, traffic management, load testing.
> Levels: Beginner → Intermediate → Advanced

---

## BEGINNER

**Q1. What is the difference between vertical and horizontal scaling in Azure?**

**Answer:**

| | Vertical (Scale Up) | Horizontal (Scale Out) |
|--|--------------------|-----------------------|
| **What** | Bigger machine (more CPU/RAM) | More instances of same size |
| **Example** | VM: D2s_v3 → D8s_v3 | 2 App Service instances → 10 |
| **Ceiling** | Hard limit per VM size | Essentially unlimited |
| **Downtime** | Often requires restart | No downtime (load balanced) |
| **Cost** | Exponential at high tiers | Linear |
| **Requirement** | None | Stateless application design |

**Cloud-native preference:** Horizontal scaling. Design stateless services — externalize session to Redis, avoid in-memory state.

**When vertical is appropriate:** Databases (SQL) where horizontal sharding is expensive, or legacy monoliths that can't scale horizontally.

---

**Q2. What is Azure Autoscale and how does it work on App Service?**

**Answer:**

Azure Autoscale automatically adds or removes instances based on rules you define. On App Service:

**Scale-out triggers:**
- **Metric-based**: CPU > 70% for 10 minutes → add 2 instances
- **Schedule-based**: Scale to 10 instances every weekday at 9 AM
- **Custom metrics**: Queue depth, HTTP queue length, Application Insights metrics

**Configuration example:**
```json
{
  "scaleOut": {
    "condition": "Average CPU > 70%",
    "window": "10 minutes",
    "cooldown": "5 minutes",
    "increment": 2
  },
  "scaleIn": {
    "condition": "Average CPU < 30%",
    "window": "15 minutes",
    "cooldown": "20 minutes",
    "decrement": 1
  },
  "minimum": 2,
  "maximum": 20,
  "default": 3
}
```

**Important nuances:**
- Scale-in cooldown should be longer than scale-out to avoid thrashing
- Minimum 2 instances for production (single instance = no HA)
- App Service instances are added to the load balancer automatically — no config needed

---

**Q3. What is KEDA and why is it preferred over standard HPA in AKS?**

**Answer:**

**HPA (Horizontal Pod Autoscaler):** Built-in Kubernetes autoscaler based on CPU and memory.

**KEDA (Kubernetes Event-driven Autoscaling):** Extends HPA to scale on external event sources.

**Why KEDA is better for event-driven workloads:**

| Scenario | HPA | KEDA |
|----------|-----|------|
| Scale on CPU | Yes | Yes |
| Scale on queue depth | No | Yes (Azure Service Bus, Event Hubs) |
| Scale on database row count | No | Yes |
| Scale to zero | No | Yes |
| Scale up before CPU spikes | No | Yes (proactive) |

**KEDA scale-to-zero** is critical for batch workloads — no pods running when queue is empty = zero cost.

**Example KEDA ScaledObject:**
```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor-scaler
spec:
  scaleTargetRef:
    name: order-processor
  minReplicaCount: 0     # Scale to zero when idle
  maxReplicaCount: 50
  triggers:
  - type: azure-servicebus
    metadata:
      queueName: orders
      messageCount: "10"  # 1 pod per 10 messages in queue
```

---

## INTERMEDIATE

**Q4. Explain the Cache-Aside pattern with Redis and when it breaks down.**

**Answer:**

**Cache-Aside (Lazy Loading):**

```
1. App checks Redis for key
2. Cache HIT  → return cached value
3. Cache MISS → query database → write to Redis → return value

Code pattern:
  var product = await redis.GetAsync<Product>($"product:{id}");
  if (product == null)
  {
      product = await db.Products.FindAsync(id);
      await redis.SetAsync($"product:{id}", product, TimeSpan.FromMinutes(5));
  }
  return product;
```

**Failure scenarios and mitigations:**

| Problem | Symptom | Fix |
|---------|---------|-----|
| **Cache stampede** | Redis miss for popular key → 1000 DB queries simultaneously | Use Redis SETNX lock + single-flight (only 1 thread populates cache) |
| **Cold start** | After Redis restart, all keys missing → DB overwhelmed | Pre-warm cache on startup; use Redis persistence (RDB/AOF) |
| **Stale data** | Product price updated but cached version returned | Explicit cache invalidation on write + short TTL (belt and suspenders) |
| **Cache poisoning** | Bug writes wrong data to Redis | Hash/version the cached value; validate on read |
| **Redis connection failure** | Circuit open → all reads hit DB | Circuit breaker (Polly): fail open (bypass cache on Redis timeout) |

**Write-through vs Cache-Aside:**
- Write-through: write to cache AND DB simultaneously — cache always fresh but higher write latency
- Cache-aside: simpler, more common — accepts brief staleness

---

**Q5. How does Azure CosmosDB autoscale work and how do you size it for variable workloads?**

**Answer:**

**CosmosDB Autoscale:**
- You set a **maximum RU/s** (e.g., 10,000 RU/s)
- CosmosDB scales from 10% to 100% of max automatically (1,000 → 10,000 RU/s)
- You're billed at the **highest RU/s reached per hour**
- No manual scaling, no throttling within the range

**Manual vs Autoscale decision:**

| | Manual provisioned | Autoscale |
|--|-------------------|-----------| 
| Load pattern | Predictable, steady | Variable, spiky |
| Cost (steady) | Cheaper | 1.5× more expensive |
| Cost (spiky) | Expensive (over-provision) | Cheaper (pays for peaks only) |
| SLA | Same | Same |

**Sizing formula:**
```
Peak RU/s estimate:
  1KB document read by partition key = 1 RU
  1KB document write = 5 RU
  Query across partitions = 10-100 RU (depends on results scanned)

Example: 1,000 orders/sec, each order = 2KB
  → Write: 1,000 × 2KB × 5 RU/KB = 10,000 RU/s write
  → Reads: 5,000 lookups/sec × 1 RU = 5,000 RU/s read
  → Total: ~15,000 RU/s peak
  → Autoscale max: 20,000 RU/s (25% headroom)
```

**Cost optimization:**
- Use **serverless** for dev/test (pay per request, no minimum)
- Use **autoscale** for prod with variable load
- Use **manual + reserved capacity** for steady high-throughput (cheapest for predictable)

---

**Q6. How do you design an AKS cluster for a 10x Black Friday traffic spike?**

**Answer:**

**Architecture layers:**

```
Layer 1 — Edge (pre-cluster)
  Azure Front Door:
  - WAF rules: block known attack patterns during high load
  - Rate limiting: 1000 req/s per IP
  - CDN: static assets cached at edge (product images, JS)
  - Global load balancing: route to nearest healthy region

Layer 2 — API Gateway
  APIM:
  - Rate limiting per subscription key (prevent API abuse)
  - Response caching for product catalog (TTL 60s)
  - Circuit breaker policy (fail fast when downstream overloaded)

Layer 3 — AKS
  System node pool: 3 nodes (Standard_D4s_v3) — reserved, not autoscaled
  User node pools:
    - web-pool:   min 5, max 50 nodes (Standard_D4s_v3) — Cluster Autoscaler
    - worker-pool: min 0, max 30 nodes (Standard_D8s_v3) — Spot nodes, KEDA
  
  Pod autoscaling:
    - HPA on frontend pods: CPU > 60% → scale out
    - KEDA on order workers: Service Bus queue depth / 50 messages per pod
    - KEDA: scale-to-zero on worker-pool during off-peak

Layer 4 — Data
  Redis: pre-warm product catalog cache 30 min before event
  CosmosDB: autoscale, max 500K RU/s (set 2h before event)
  Service Bus: pre-create queues, message TTL = 24h (no lost orders)
```

**Pre-scaling runbook:**
```bash
# 2 hours before event:
# 1. Scale minimum nodes in user pool
kubectl annotate hpa frontend-hpa \
  cluster-autoscaler.kubernetes.io/safe-to-evict="false"

# 2. Pre-warm Redis cache
kubectl create job cache-warmer --image=cache-warmer:latest

# 3. Set CosmosDB autoscale max higher
az cosmosdb sql container throughput update \
  --account-name prod-cosmos \
  --database-name orders \
  --name order-container \
  --max-throughput 500000

# 4. Verify KEDA ScaledObjects are configured
kubectl get scaledobjects -n production
```

---

## ADVANCED

**Q7. Design a system that handles 1 million concurrent users for a product launch. Cost must be defensible to a CFO.**

**Answer:**

**Requirement validation:**
- 1M concurrent users ≠ 1M RPS — users browsing at 1 req/3s → ~333K RPS at peak
- Assume 80/20 rule: 80% browse, 20% buy → 267K read RPS + 67K write RPS

**Architecture with cost breakdown:**

```
TIER 1 — STATIC CONTENT (handles 70% of load)
  Azure Front Door + CDN
  → Product images, HTML, JS, CSS cached at 100+ PoPs globally
  → Cost: ~$0.01/GB data transfer + $0.009/10K requests
  → Handles: ~500K of the 700K "browsing" requests without hitting origin

TIER 2 — API LAYER
  APIM (Standard tier):
  → Rate limiting, response caching (product catalog: TTL 60s)
  → 1M calls/month included in Standard tier
  
  AKS (3 user node pools):
  → Product API: 20–100 pods (D4s_v3, 4 vCPU each)
  → Cart API: 10–50 pods
  → Order API: 5–30 pods (KEDA on Service Bus)
  
  Spot nodes for scale: 60-70% savings vs on-demand
  Reserved instances for base load: 40% savings

TIER 3 — CACHING
  Redis Premium P5 (26GB, cluster)
  → Product catalog: TTL 60s, ~5ms reads
  → Cart sessions: TTL 30min
  → Cost: ~$800/month, handles 250K ops/sec

TIER 4 — DATA
  CosmosDB autoscale 0–500K RU/s
  → Orders: strong consistency (financial)
  → Product views: eventual (metrics)
  Cost burst: ~$4,000 for event day peak
  
  Azure SQL Hyperscale (8 vCores + 2 read replicas)
  → Customer accounts, inventory
  → Cost: ~$1,200/month

TIER 5 — ASYNC PROCESSING
  Service Bus Premium (1 messaging unit)
  → Order processing, inventory updates, notification fanout
  → Cost: ~$700/month

TOTAL ESTIMATED COST (event day):
  Normal month: ~$8,000/month
  Launch day spike: +$3,000 (CosmosDB RU burst, extra nodes)
  CFO narrative: "10x traffic for $3K extra = highly efficient"
```

**Reliability mechanisms:**
- Circuit breakers (Polly) on all service-to-service calls
- Graceful degradation: if inventory service down, show "check availability" instead of error
- Queue orders immediately (Service Bus) even if downstream slow — user gets "order received"

---

**Q8. What is the thundering herd problem in distributed systems and how do you prevent it in Azure?**

**Answer:**

**Thundering herd:** When many concurrent processes/threads compete for the same resource simultaneously — typically after a cache miss, a service restart, or a traffic spike.

**Scenario 1: Cache miss stampede**
```
Redis TTL expires for popular product (Black Friday deal)
→ 10,000 requests hit simultaneously
→ All see cache miss
→ All query database simultaneously
→ Database overwhelmed → connection pool exhausted → all fail

Prevention:
  1. Probabilistic early expiration (PER):
     if (ttl_remaining < threshold && random() < probability):
         refresh_now()  # only some threads trigger refresh early
  
  2. Redis lock (mutex):
     lock_key = f"lock:product:{id}"
     if redis.setnx(lock_key, "1", ex=10):   # get the lock
         data = db.query(id)
         redis.set(f"product:{id}", data, ex=300)
         redis.delete(lock_key)
     else:
         time.sleep(0.05)
         return redis.get(f"product:{id}")   # wait and retry
  
  3. Staggered TTL: add random jitter to TTL
     ttl = base_ttl + random.randint(0, 60)  # avoid synchronized expiry
```

**Scenario 2: Service restart stampede**
```
10 AKS pods restart simultaneously (rolling update)
→ All pods connect to database on startup
→ Connection pool spike → database overwhelmed

Prevention:
  - Stagger pod startups: maxSurge=1, maxUnavailable=0 in rolling update
  - Add startup delay jitter in app code
  - Database connection pool: min=2, max=20 per pod (not max=100)
  - Use connection proxy (PgBouncer for PostgreSQL) to buffer connections
```

**Scenario 3: Retry storm**
```
Service B is slow → Service A times out → retries immediately
→ Retry makes Service B even slower → more timeouts → more retries

Prevention (Polly exponential backoff + jitter):
  services.AddHttpClient("service-b")
      .AddPolicyHandler(Policy<HttpResponseMessage>
          .Handle<HttpRequestException>()
          .WaitAndRetryAsync(3, retryAttempt =>
              TimeSpan.FromSeconds(Math.Pow(2, retryAttempt))  // 2s, 4s, 8s
              + TimeSpan.FromMilliseconds(Random.Shared.Next(0, 100))));  // jitter
```

---

**Q9. How do you implement geo-distributed scalability for a global retail platform on Azure?**

**Answer:**

**Requirements:**
- Users in North America, Europe, Asia-Pacific
- Product catalog: read-heavy, slightly stale acceptable
- Orders: write-heavy, must be consistent, region-specific
- Compliance: EU data must stay in EU (GDPR)

**Architecture:**

```
Global traffic layer:
  Azure Front Door (global):
  → Routes users to nearest healthy region
  → Anycast DNS: user in Tokyo → Japan East, user in London → UK South
  → WAF at global edge
  → Static content cached at 100+ PoPs

Regional deployments (Canada Central, UK South, Japan East):
  Each region has:
  → AKS cluster (workloads)
  → CosmosDB write region
  → Redis Cache (regional product cache)
  → Service Bus (regional order queue)

Data strategy:
  Product catalog:
    → CosmosDB multi-region read
    → Consistency: Consistent Prefix (fast, ordered, ~1s staleness acceptable)
    → All regions can read the latest catalog
  
  User sessions:
    → CosmosDB Session consistency
    → User's writes visible to themselves immediately
  
  Orders:
    → Region-local CosmosDB partition (orderId + region suffix)
    → Compliance: EU orders stored only in UK South + West Europe
    → Strong consistency for financial accuracy
  
  Inventory:
    → Eventual consistency (slight oversell handled by reservation + cancellation)
    → Central inventory service in primary region; replicated read cache to others

GDPR compliance enforcement:
  → APIM policy: inspect JWT → extract user region → route to correct backend
  → CosmosDB: separate container per data residency zone
  → Azure Policy: deny cross-region data copy for EU customers
```

**Latency targets:**
- Product page load: <200ms globally (CDN for static, CosmosDB nearby replica for data)
- Add to cart: <300ms (regional Redis + CosmosDB)
- Checkout: <500ms (regional primary write, synchronous confirmation)
