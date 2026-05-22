# Azure Cache for Redis
### 🟢 Beginner → 🟡 Intermediate

> *"A cache is not a database with a TTL.  
>  It's a performance contract between your app and your data tier."*

---

## Q1. How do you implement the cache-aside pattern with Azure Cache for Redis?

**Scenario:** A product catalog API hits CosmosDB on every request. At 5,000 req/s peak load, CosmosDB RU consumption is $2,400/month and latency is 50ms per request. The catalog changes once per hour. How do you cut both cost and latency?

```
Cache-Aside pattern:

  API request: GET /products/category/electronics

  CACHE HIT (95% of requests):
  App → Redis → data found (TTL=5min) → return in <1ms

  CACHE MISS (5% of requests — cold start or TTL expired):
  App → Redis → not found
     → App → CosmosDB → data returned (50ms)
     → App → Redis → SET key data TTL=5min
     → App → return data (50ms)

  Cost comparison:
  Before: 5,000 req/s × 100% CosmosDB = 500K RU/s = $2,400/month
  After:  5,000 req/s × 5% CosmosDB = 25K RU/s = $120/month (95% savings)
  
  Redis cost: C2 Standard = $185/month → net saving = $2,095/month
```

```python
import redis
import json
from azure.cosmos import CosmosClient

# Connect to Redis (TLS required — port 6380, not 6379)
redis_client = redis.Redis(
    host="redis-prod-cc.redis.cache.windows.net",
    port=6380,
    password="<ACCESS_KEY>",
    ssl=True,                     # Always use TLS in production
    decode_responses=True,
    socket_connect_timeout=2,
    socket_timeout=2,
    retry_on_timeout=True
)

def get_products_by_category(category: str) -> list:
    cache_key = f"products:category:{category}"

    # 1. Try cache first
    cached = redis_client.get(cache_key)
    if cached:
        return json.loads(cached)     # Cache hit — return immediately

    # 2. Cache miss: fetch from CosmosDB
    cosmos = CosmosClient(COSMOS_URL, COSMOS_KEY)
    container = cosmos.get_database_client("catalog").get_container_client("products")
    products = list(container.query_items(
        query="SELECT * FROM c WHERE c.category = @cat",
        parameters=[{"name": "@cat", "value": category}],
        enable_cross_partition_query=True
    ))

    # 3. Write to cache with TTL
    redis_client.setex(
        cache_key,
        300,                          # TTL = 300 seconds (5 minutes)
        json.dumps(products)
    )

    return products

# Invalidate cache when product catalog changes
def update_product(product: dict):
    container.upsert_item(product)
    redis_client.delete(f"products:category:{product['category']}")  # Bust cache
```

> ⚠️ **Gotcha:** Never store the entire cache payload as a single Redis key that all 5,000 req/s read simultaneously. On a cache miss, multiple requests hit CosmosDB concurrently before the first response is written back — this is called the **thundering herd problem**. Fix: use a distributed lock (`SET key:lock "" NX EX 5`) to allow only one request to refresh the cache.

> 💡 **Deep dive hint:** Redis semantic caching for Azure OpenAI — embedding-based cache where similar (not identical) prompts return cached responses, dramatically reducing Azure OpenAI token spend for repeated question patterns in chatbots.

---

## Q2. How do you choose the right Azure Cache for Redis tier?

**Scenario:** Your team is evaluating Redis tiers. The basic tier is cheapest but your apps need HA. The developer asks what the difference is between Standard and Premium.

```
Tier comparison:

  Basic (C0–C6)
  ─────────────
  Single node — NO HA
  0.5GB – 53GB memory
  No replication
  ~$16–$500/month
  Use: dev/test ONLY — a node failure = data loss

  Standard (C0–C6)
  ─────────────────
  Two nodes (primary + replica)
  Automatic failover ~30 seconds
  0.5GB – 53GB memory
  ~$32–$1,000/month
  Use: production apps that can tolerate 30s failover

  Premium (P1–P5)
  ────────────────
  Two nodes + Redis persistence (AOF/RDB snapshots)
  Geo-replication to secondary region
  Redis Cluster (shard across multiple nodes)
  Private Endpoint support
  VNet injection
  26GB – 530GB memory
  ~$370–$18,000/month
  Use: mission-critical, large datasets, compliance requirements

  Enterprise / Enterprise Flash
  ─────────────────────────────
  Redis Enterprise (not open-source Redis)
  RediSearch, RedisJSON, RedisBloom modules
  Active geo-replication (multi-master writes)
  99.99% SLA (Standard = 99.9%)
  Use: large financial platforms, search-enriched caching
```

```bash
# Create Premium tier Redis with geo-replication + Private Endpoint
az redis create \
  --name redis-prod-cc \
  --resource-group rg-data \
  --location canadacentral \
  --sku Premium \
  --vm-size P2 \              # 13GB memory, 4,000 connections
  --enable-non-ssl-port false \   # Force TLS only
  --minimum-tls-version 1.2 \
  --subnet-id $SUBNET_ID          # VNet injection (Premium feature)

# Enable geo-replication to Canada East (Premium+)
az redis geo-replication link \
  --name redis-prod-cc \
  --resource-group rg-data \
  --server-to-link redis-dr-ce   # Secondary in Canada East (must be same SKU)

# Enable Redis persistence (RDB snapshot every 60 minutes)
az redis patch-schedule create \
  --name redis-prod-cc \
  --resource-group rg-data \
  --schedule-entries '[{"dayOfWeek":"Everyday","startHourUtc":2,"maintenanceWindow":"PT5H"}]'

az redis update \
  --name redis-prod-cc \
  --resource-group rg-data \
  --redis-configuration '{"rdb-backup-enabled":"true","rdb-backup-frequency":"60","rdb-backup-max-snapshot-count":"1","rdb-storage-connection-string":"<STORAGE_CONNECTION>"}'
```

> ⚠️ **Gotcha:** Standard and Premium tiers require the secondary node to be in the same region (for HA within the region). Geo-replication to a second region is a separate feature available only in Premium. Don't confuse HA (same-region replica) with geo-DR (cross-region replica) — they require different tiers and configuration.

---

## Q3. How do you use Redis for distributed session state in an AKS application?

**Scenario:** Your AKS payment portal runs 20 replicas. Users log in, add items to cart, but when requests route to a different pod, their session is gone. The app stores session in pod memory.

```
Sticky session (broken) → Redis session (correct):

  BEFORE:                           AFTER:
  User → Pod A (session: cart)      User → Pod A → write session to Redis
  Retry → Pod B (no session)        User → Pod B → reads session from Redis
  → user sees empty cart ❌         → cart still intact ✅

  Redis session key pattern:
  sessions:{session-id}  →  {userId, cart, lastActivity, csrfToken}
  TTL: 30 minutes (sliding — reset on each request)
```

```csharp
// .NET 8 — configure Redis session store
// Program.cs

builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration["Redis:ConnectionString"];
    // Format: redis-prod-cc.redis.cache.windows.net:6380,password=<key>,ssl=True,abortConnect=False
    options.InstanceName = "payment-portal:";   // Prefix all keys
});

builder.Services.AddSession(options =>
{
    options.IdleTimeout = TimeSpan.FromMinutes(30);   // Session TTL
    options.Cookie.HttpOnly = true;
    options.Cookie.IsEssential = true;
    options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
});

// In controller: read/write session
HttpContext.Session.SetString("cart", JsonSerializer.Serialize(cartItems));
var cart = HttpContext.Session.GetString("cart");
```

```python
# Python/Flask — redis-py session store
from flask import Flask, session
from flask_session import Session
import redis

app = Flask(__name__)
app.config["SECRET_KEY"] = os.getenv("SESSION_SECRET")
app.config["SESSION_TYPE"] = "redis"
app.config["SESSION_PERMANENT"] = False
app.config["SESSION_USE_SIGNER"] = True
app.config["SESSION_REDIS"] = redis.Redis(
    host="redis-prod-cc.redis.cache.windows.net",
    port=6380, ssl=True,
    password=os.getenv("REDIS_KEY")
)

Session(app)

@app.route("/cart/add", methods=["POST"])
def add_to_cart():
    if "cart" not in session:
        session["cart"] = []
    session["cart"].append(request.json)
    session.modified = True   # Force Redis write even for mutable objects
    return jsonify({"status": "ok"})
```

> ⚠️ **Gotcha:** Redis session TTL is absolute by default — a user active for 35 minutes loses their session at minute 30. Implement **sliding expiration**: reset the TTL on every request. In .NET this is automatic with `Session.IdleTimeout`. In Python/Redis, call `redis.expire(session_key, 1800)` on every request handler.

---

## Q4. How do you diagnose Redis memory pressure and eviction issues?

**Scenario:** Your API starts returning cache misses at a high rate at 3pm every day. Redis logs show `maxmemory-policy allkeys-lru` evictions. Some items that should be cached for 1 hour are being evicted after 5 minutes.

```
Redis memory pressure diagnosis:

  redis-cli INFO memory
  ─────────────────────
  used_memory: 12,884,901,888  (12GB)   ← approaching limit
  maxmemory:   13,421,772,800  (12.5GB) ← max configured
  maxmemory_human: 12.50G
  maxmemory_policy: allkeys-lru         ← evicts least-recently-used keys

  redis-cli INFO stats
  ─────────────────────
  evicted_keys: 24,502             ← keys being evicted = memory full
  keyspace_hits: 890,234
  keyspace_misses: 45,678
  hit rate = 890K / (890K + 46K) = 95.1%  ← still healthy despite eviction

  3pm pattern:
  → daily batch job caches 2GB of product data at 3pm
  → pushes Redis over limit → evicts session data (LRU) instead
  → users lose sessions ❌
```

```bash
# Connect to Redis and inspect memory usage
az redis list-keys --name redis-prod-cc --resource-group rg-data
# Use redis-cli with the primary key:

redis-cli -h redis-prod-cc.redis.cache.windows.net \
  -p 6380 --tls -a <access-key> \
  INFO memory | grep -E "used_memory_human|maxmemory_human|mem_fragmentation_ratio"

# Find the largest keys consuming memory (sample-based)
redis-cli -h redis-prod-cc.redis.cache.windows.net \
  -p 6380 --tls -a <access-key> \
  --bigkeys
# Expected: shows top 10 largest keys per data type

# Check eviction rate in Azure Monitor (CLI)
az monitor metrics list \
  --resource "/subscriptions/<SUB>/resourceGroups/rg-data/providers/Microsoft.Cache/Redis/redis-prod-cc" \
  --metric "evictedkeys" \
  --interval PT5M \
  --output table

# Fix 1: Increase Redis tier (more memory)
az redis update \
  --name redis-prod-cc \
  --resource-group rg-data \
  --sku Premium \
  --vm-size P3          # 26GB instead of P2 (13GB)

# Fix 2: Use volatile-lru policy (only evict keys WITH TTL — protects sessions without TTL)
az redis update \
  --name redis-prod-cc \
  --resource-group rg-data \
  --redis-configuration '{"maxmemory-policy":"volatile-lru"}'
# Now: keys WITHOUT TTL (like persistent data) are NEVER evicted
# Keys WITH TTL (cache items) are evicted LRU when memory full
```

> ⚠️ **Gotcha:** `allkeys-lru` (default in Azure) evicts ANY key including session state that has no TTL. If session data sits alongside cached data in the same Redis instance, sessions get evicted during memory pressure. Either: (a) use `volatile-lru` so only TTL-bearing keys are evicted, or (b) use separate Redis instances for sessions vs cache.
