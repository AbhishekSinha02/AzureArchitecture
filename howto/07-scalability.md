# Azure Scalability — Hands-On How-To

> *"Scaling is not about handling more traffic.  
>  It's about handling more traffic while the person on-call stays asleep."*

---

**Q: How do you configure autoscale on an Azure App Service?**

In the App Service Plan (not the App Service itself — the plan is what you scale), you go to Scale Out → Custom autoscale → add a rule: `if Average CPU > 70% over 10 minutes, increase instance count by 2`. You add a scale-in rule too: `if Average CPU < 30% over 15 minutes, decrease by 1`. Set a minimum of 2 (never run with one instance in production) and a maximum of 20. The cooldown period on scale-in should be longer than scale-out — you want to scale up fast and scale down slowly to avoid thrashing. Schedule-based rules layer on top: scale to 10 instances at 8 AM weekdays regardless of CPU.

> 💡 **Deep dive hint:** App Service autoscale vs. Azure Functions Premium plan autoscale — why Functions has a different cold-start model and how to pre-warm instances.

---

**Q: How does KEDA scale AKS pods based on a Service Bus queue?**

You install KEDA in AKS (helm install or the KEDA add-on), then create a `ScaledObject` resource pointing at your Service Bus queue. KEDA polls the queue every 30 seconds and adjusts pod replicas based on queue depth — you set a `messageCount` trigger (e.g., 1 pod per 10 messages). The beauty is `minReplicaCount: 0` — when the queue is empty, all pods scale to zero and you pay nothing. When a burst of messages arrives, KEDA scales from zero to N pods within about 30 seconds. No custom code, no HPA limitations — KEDA brings external event sources into the Kubernetes scaling model natively.

> 💡 **Deep dive hint:** KEDA ScaledJob (vs ScaledObject) — for batch workloads where each job processes one message and exits, rather than a long-running consumer.

---

**Q: How do you handle a 10x traffic spike that you know is coming — like Black Friday?**

Two days before: scale up your minimum instance counts (no waiting for autoscale to react), increase CosmosDB autoscale max RU/s, pre-warm your Redis cache with the product catalog, and set APIM cache policies for high-read endpoints. The day before: run a load test against a staging environment at projected peak — find the bottleneck before it finds you. During the spike: watch queue depths and cache hit rates, not CPU — CPU is a lagging indicator. After the spike: scale back down (autoscale handles this), reduce CosmosDB max RU/s to stop paying for unused capacity.

> 💡 **Deep dive hint:** Azure Load Testing — running a JMeter-based load test in Azure at scale, and reading the test report to identify the first resource to saturate.

---

**Q: How do you scale a CosmosDB container for a workload that spikes during business hours?**

You enable autoscale on the container and set the maximum RU/s based on your peak estimate. CosmosDB automatically scales from 10% of max to 100% — so a max of 10,000 RU/s means it idles at 1,000 RU/s at night and bursts to 10,000 during business hours. You're billed at the highest RU/s used per hour. To size the max, estimate your peak write rate: a 1KB document write costs about 5 RU, so 1,000 writes/sec = 5,000 RU/s for writes alone. Add your read RU/s, multiply by 1.3 for headroom, and set that as your max.

> 💡 **Deep dive hint:** CosmosDB serverless vs. autoscale vs. manual — the cost model for each and when serverless makes sense despite its 1TB and RU/s limits.

---

**Q: How does Azure Front Door help with scalability for a globally distributed app?**

Front Door puts your app behind 100+ edge PoPs (Points of Presence) worldwide. Static content — images, JS, CSS — is cached at the PoP closest to the user and never hits your origin. Dynamic content goes to the nearest healthy backend region. This means a user in Tokyo accessing your Canada-based app might get their product images from a Tokyo PoP with 5ms latency instead of routing all the way to Canada at 180ms. The origin sees only cache misses and dynamic requests — a fraction of total traffic. For scale events, this edge caching is your first line of defence before the traffic ever touches your AKS cluster.

> 💡 **Deep dive hint:** Front Door rules engine — routing rules based on URL path, query string, and headers to split traffic between app versions or backends.

---

**Q: What is the cache-aside pattern and how do you implement it with Azure Redis?**

Cache-aside means the application is responsible for cache management — it checks Redis first, fetches from the database on a miss, and writes the result to Redis for the next caller. In C# with StackExchange.Redis: `var value = await cache.StringGetAsync(key)` — if null, query the database, then `await cache.StringSetAsync(key, value, TimeSpan.FromMinutes(5))`. The TTL is the most important tuning knob: too short and you hammer the database; too long and you serve stale data. For product catalogs: 5 minutes. For user session data: 30 minutes. For reference data that rarely changes: 1 hour with explicit invalidation on write.

> 💡 **Deep dive hint:** Cache stampede (thundering herd) — what happens when a popular key expires and 1,000 requests hit the database simultaneously, and how to prevent it with probabilistic early expiration or a Redis lock.

---

**Q: How do you right-size AKS node pools for a workload?**

You start with resource requests and limits on every pod — without these, the scheduler can't make placement decisions and cluster autoscaler can't reason about capacity. Profile the actual CPU and memory usage of your pods under load using `kubectl top pods` and Prometheus. Then set requests at the P75 of actual usage (scheduler guarantee), limits at the P99 (safety ceiling). For node pool sizing: pick nodes where your largest pod uses ~25% of the node's resources — this gives you 4 pods per node naturally, and the cluster autoscaler can pack nodes efficiently. Avoid mixing large and small workloads in the same node pool — use separate pools with taints and tolerations.

> 💡 **Deep dive hint:** VPA (Vertical Pod Autoscaler) — how it recommends and optionally adjusts resource requests/limits based on actual usage history.
