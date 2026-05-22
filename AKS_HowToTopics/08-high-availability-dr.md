# AKS High Availability & Disaster Recovery
### 🔴 Advanced

> *"HA is not about preventing failure.  
>  It's about designing so that failure is boring."*

---

## Q1. How do you design an AKS cluster to survive an Availability Zone failure?

**Scenario:** Your payment service is in Canada Central (3 AZs). One AZ loses power. You need the cluster to keep serving traffic with no manual intervention and no data loss.

```
AKS Zone-Redundant Architecture (Canada Central):

  Zone 1          Zone 2          Zone 3
  ──────────      ──────────      ──────────
  Node (D4s)      Node (D4s)      Node (D4s)
  Node (D4s)      Node (D4s)      Node (D4s)

  payment-api-0   payment-api-1   payment-api-2
  order-svc-0     order-svc-1     order-svc-2

  Zone 2 FAILS:
  → 4 nodes gone
  → payment-api-1, order-svc-1 gone
  → Kubernetes reschedules to Zones 1 and 3
  → Service still running (minAvailable=2 in PDB) ✅

  Load balancer health probe detects Zone 2 nodes are down
  → stops sending traffic to Zone 2 pods automatically
```

```bash
# Node pools must be created with --zones 1 2 3
az aks nodepool add \
  --cluster-name aks-prod-cc \
  --resource-group rg-spoke-aks \
  --name userpool \
  --zones 1 2 3 \                   # Critical — omitting this puts all nodes in one zone
  --node-count 6 \                  # 2 per zone minimum
  --enable-cluster-autoscaler \
  --min-count 6 \
  --max-count 30
```

**Spread pods evenly across zones using `topologySpreadConstraints`:**
```yaml
spec:
  topologySpreadConstraints:
  - maxSkew: 1                              # Allow max 1 pod difference between zones
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule        # Don't pile pods into one zone
    labelSelector:
      matchLabels:
        app: payment-api
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname     # Also spread across nodes within zone
    whenUnsatisfiable: ScheduleAnyway       # Best-effort for node spread
    labelSelector:
      matchLabels:
        app: payment-api
```

```bash
# Verify pod distribution across zones after deployment
kubectl get pods -n production -o wide \
  -l app=payment-api \
  --sort-by=.spec.nodeName

# Check which zone each node is in
kubectl get nodes \
  -o custom-columns='NAME:.metadata.name,ZONE:.metadata.labels.topology\.kubernetes\.io/zone'
# Expected: nodes spread evenly across zone1, zone2, zone3
```

> ⚠️ **Gotcha:** `--zones 1 2 3` at node pool creation cannot be changed later. A node pool with zones 1 and 2 only cannot be retroactively extended to zone 3. Plan zone coverage upfront — migration requires creating a new node pool.

---

## Q2. How do you configure PodDisruptionBudget to protect workloads during node maintenance?

**Scenario:** AKS auto-upgrade is draining nodes. Your payment API has 3 replicas. During drain, Kubernetes is about to take down 2 pods simultaneously — leaving only 1 pod serving all traffic.

```
Without PDB:                     With PDB (minAvailable: 2):
  Node drain →                     Node drain →
  all pods evicted at once          evict pod 1 ✅ (2 still running)
  1 pod left                        wait for replacement pod
  → traffic spike → timeout         new pod Running ✅
                                    evict pod 2 ✅ (2 still running)
                                    → no impact on traffic
```

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: pdb-payment-api
  namespace: production
spec:
  # Choose ONE of minAvailable or maxUnavailable — not both
  minAvailable: 2              # Always keep at least 2 pods running
  # OR:
  # maxUnavailable: 1          # Allow at most 1 pod to be unavailable at a time
  selector:
    matchLabels:
      app: payment-api
```

**When to use `minAvailable` vs `maxUnavailable`:**
| Workload | Recommended |
|----------|-------------|
| APIs (stateless, 3+ replicas) | `maxUnavailable: 1` (faster drain) |
| Databases (StatefulSet) | `minAvailable: 2` (safety first) |
| Single-replica workloads | `minAvailable: 1` (blocks drain — force it) |
| Job processors | `maxUnavailable: 50%` (drain faster) |

```bash
# Check PDB status — are any disruptions currently allowed?
kubectl get pdb -n production
# NAME              MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS
# pdb-payment-api   2               N/A               1           ← can evict 1 pod safely

# If ALLOWED DISRUPTIONS = 0, the drain will block until a pod becomes available
# Investigate: kubectl describe pdb pdb-payment-api -n production

# Simulate a node drain to test PDB enforcement
kubectl drain <node-name> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --dry-run=client              # --dry-run first to see what would happen
```

> ⚠️ **Gotcha:** A PDB with `minAvailable: 1` on a single-replica Deployment **blocks** all voluntary disruptions — the node drain hangs indefinitely. You must either scale up to 2 replicas before the drain, or use `kubectl drain --force` (skips PDB — risky for stateful workloads).

---

## Q3. How do you implement multi-region DR for AKS with active-passive failover?

**Scenario:** Your AKS cluster is in Canada Central. A full regional outage would take down all services. RTO requirement is 30 minutes, RPO is 1 hour.

```
Active-Passive Multi-Region:

  Canada Central (PRIMARY — active)
  ──────────────────────────────────
  AKS cluster: aks-prod-cc
  CosmosDB: replicated to Canada East
  ACR: geo-replicated
  Traffic Manager → HTTPS health probe (every 10s)

                          ↕ CosmosDB geo-replication (async)
                          ↕ ACR geo-replication (async)

  Canada East (DR — warm standby)
  ─────────────────────────────────
  AKS cluster: aks-dr-ce (running, scaled down to 1 replica)
  Traffic Manager → health probe FAILS → not in rotation
  
  FAILOVER (RTO ~15 min):
  1. Health probe to Canada Central fails (1-2 min)
  2. Traffic Manager routes DNS to Canada East
  3. Scale AKS pods in Canada East to production scale
  4. CosmosDB failover: promote Canada East to primary
```

```bash
# Azure Traffic Manager — routes to healthy region
az network traffic-manager profile create \
  --name tm-payment-api \
  --resource-group rg-global \
  --routing-method Priority \      # Priority = active-passive
  --unique-dns-name payment-api-global

# Add Canada Central as Priority 1 (active)
az network traffic-manager endpoint create \
  --name ep-canada-central \
  --profile-name tm-payment-api \
  --resource-group rg-global \
  --type externalEndpoints \
  --target payment-cc.internal.company.com \
  --priority 1 \
  --endpoint-status enabled

# Add Canada East as Priority 2 (standby)
az network traffic-manager endpoint create \
  --name ep-canada-east \
  --profile-name tm-payment-api \
  --resource-group rg-global \
  --type externalEndpoints \
  --target payment-ce.internal.company.com \
  --priority 2 \
  --endpoint-status enabled

# Test failover: disable primary endpoint
az network traffic-manager endpoint update \
  --name ep-canada-central \
  --profile-name tm-payment-api \
  --resource-group rg-global \
  --endpoint-status disabled
# → Traffic Manager should route to Canada East within 2 TTL cycles (default 30s TTL)
```

**Runbook: manual failover steps:**
```bash
# Step 1 — Scale up DR cluster
az aks nodepool update \
  --cluster-name aks-dr-ce -g rg-dr-ce \
  --name userpool \
  --min-count 10 --max-count 50

# Step 2 — Trigger CosmosDB failover to Canada East
az cosmosdb failover-priority-change \
  --name cosmos-prod \
  --resource-group rg-prod \
  --failover-policies canadaeast=0 canadacentral=1

# Step 3 — Update Traffic Manager DNS TTL to 60s for fast recovery
# (set TTL=60s during normal operations — not during failover)
```

> ⚠️ **Gotcha:** CosmosDB geo-replication is asynchronous — with `Session` or `Eventual` consistency, Canada East may lag behind by seconds. During failover, the last few writes to Canada Central may be lost (RPO ≠ 0). Use `Strong` or `BoundedStaleness` consistency for near-zero RPO in banking scenarios.

> 💡 **Deep dive hint:** Azure Chaos Studio for AKS — inject AZ failures, node crashes, and network latency into your test cluster to validate that your PDBs, topology spread, and failover automation actually work before a real incident.

---

## Q4. How do you test AKS cluster resilience with chaos engineering?

**Scenario:** You claim your cluster handles node failures gracefully. Your manager asks "prove it." You need to inject real failures in a controlled way without impacting production.

```
Chaos Studio experiment targeting AKS:

  Chaos Experiment (scheduled)
       │ fault: kill 1 random node in Zone 2
       │ duration: 10 minutes
       ▼
  AKS node pool: 1 node terminated (VMSS instance deleted)
       │
       ▼
  Expected outcome:
  ✅ Cluster Autoscaler adds replacement node
  ✅ Pods rescheduled to other zones (topology spread)
  ✅ No requests fail (PDB maintains minAvailable)
  ✅ SLO: <0.1% error rate during chaos window
  
  ❌ If outcome fails: PDB missing, topology spread wrong,
     or Cluster Autoscaler misconfigured
```

```bash
# Enable Chaos Studio on AKS cluster (required before running experiments)
az resource update \
  --resource-type "Microsoft.ContainerService/managedClusters" \
  --name aks-prod-cc \
  --resource-group rg-spoke-aks \
  --set properties.securityProfile.imageCleaner.enabled=true

# Create chaos experiment — AKS node pool shutdown
az rest --method PUT \
  --url "https://management.azure.com/subscriptions/<SUB>/resourceGroups/rg-spoke-aks/providers/Microsoft.Chaos/experiments/chaos-node-shutdown?api-version=2024-01-01" \
  --body '{
    "location": "canadacentral",
    "identity": {"type": "SystemAssigned"},
    "properties": {
      "steps": [{
        "name": "Kill Zone 2 Node",
        "branches": [{
          "name": "Branch1",
          "actions": [{
            "name": "urn:csci:microsoft:agent:stressNg/1.0",
            "type": "continuous",
            "duration": "PT10M",
            "parameters": [{"key": "pressureType", "value": "CPU"}],
            "selectorId": "selector1"
          }]
        }]
      }],
      "selectors": [{"type": "List", "id": "selector1", "targets": []}]
    }
  }'

# Simpler: manually test node failure
# (Only in a non-production cluster!)
kubectl cordon <node-name>       # Stop new pods scheduling here
kubectl drain <node-name> \
  --ignore-daemonsets \
  --delete-emptydir-data
# Watch Cluster Autoscaler add a replacement node
kubectl get nodes --watch
```

> ⚠️ **Gotcha:** Run chaos experiments during business hours (not late at night) so the team can observe and respond. A chaos test at 2am that reveals a gap in your PDB configuration is not useful if no one is watching. Chaos engineering requires human oversight — it's not a fire-and-forget job.
