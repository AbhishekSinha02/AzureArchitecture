# AKS Scaling & KEDA
### 🟡 Intermediate → 🔴 Advanced

> *"Scaling to zero costs nothing. Scaling too slowly costs everything."*

---

## Q1. How does Horizontal Pod Autoscaler (HPA) work and how do you configure it?

**Scenario:** Your payment API handles 50 req/s normally but spikes to 500 req/s during batch settlement windows. You need pods to scale out automatically based on CPU.

```
HPA Control Loop (every 15s):
  ┌──────────────────────────────────────────────────────────┐
  │  Metrics Server                                          │
  │  collects CPU/memory from kubelet every 60s              │
  └──────────────────┬───────────────────────────────────────┘
                     │ current CPU: 82%
                     ▼
  ┌──────────────────────────────────────────────────────────┐
  │  HPA Controller                                          │
  │  desired_replicas = ceil(current × current_metric        │
  │                          ÷ target_metric)                │
  │  = ceil(3 × 82% ÷ 50%) = ceil(4.92) = 5                 │
  └──────────────────┬───────────────────────────────────────┘
                     │ scale to 5 replicas
                     ▼
  Deployment: 3 pods → 5 pods
```

```yaml
# HPA targeting 50% CPU utilisation — must have resource requests set
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-payment-api
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-api
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50      # Scale when average CPU > 50%
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 400Mi         # Scale when average memory > 400Mi
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0     # Scale up immediately
      policies:
      - type: Pods
        value: 4
        periodSeconds: 60               # Add max 4 pods per 60s
    scaleDown:
      stabilizationWindowSeconds: 300   # Wait 5min before scaling down
```

```bash
# Resource requests are REQUIRED for HPA to work — set them on every container
# kubectl apply -f deployment.yaml first, then:
kubectl autoscale deployment payment-api \
  --cpu-percent=50 --min=3 --max=20 -n production

# Check HPA status
kubectl get hpa -n production
# Expected:
# NAME              REFERENCE              TARGETS   MINPODS   MAXPODS   REPLICAS
# hpa-payment-api   Deployment/payment-api   45%/50%   3         20        3

# Watch HPA scale live
kubectl get hpa hpa-payment-api -n production --watch
```

> ⚠️ **Gotcha:** HPA requires `resources.requests.cpu` to be set on the container. Without requests, the metrics server can't calculate utilisation percentage — HPA shows `<unknown>/50%` and never scales.

> 💡 **Deep dive hint:** HPA v2 custom metrics — scaling on application-specific metrics like requests-per-second by publishing to the custom metrics API via Prometheus Adapter, bypassing CPU entirely.

---

## Q2. How do you use KEDA to scale AKS pods based on Azure Service Bus queue depth?

**Scenario:** An order processing service consumes from a Service Bus queue. During off-peak hours the queue is empty and you want zero pods running. During peak, 10,000+ messages queue up and you need to scale fast.

```
Scale-to-zero with KEDA + Service Bus:

  Queue depth = 0         → KEDA scales to 0 pods  (cost: $0)
  Queue depth = 1–999     → KEDA scales 1–10 pods
  Queue depth = 1000–9999 → KEDA scales 10–20 pods
  Queue depth = 10000+    → KEDA scales to maxReplicaCount

  KEDA Operator
       │  polls queue depth every 30s
       │  (via Workload Identity — no connection string stored)
       ▼
  Service Bus Queue (sbq-orders)
       │  messageCount = 5,000
       ▼
  desired_replicas = ceil(5000 ÷ 500) = 10 pods
```

```bash
# Step 1 — Install KEDA via Helm (or use AKS add-on)
helm repo add kedacore https://kedacore.github.io/charts
helm install keda kedacore/keda \
  --namespace keda \
  --create-namespace \
  --set podIdentity.azureWorkload.enabled=true   # Use Workload Identity, not secrets
```

```yaml
# Step 2 — Grant KEDA's identity access to Service Bus
# (done once via az role assignment create — see commands below)

# Step 3 — ScaledObject: tells KEDA how to scale the Deployment
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor-scaler
  namespace: production
spec:
  scaleTargetRef:
    name: order-processor           # Deployment to scale
  minReplicaCount: 0                # Scale to zero when queue empty
  maxReplicaCount: 20
  pollingInterval: 30               # Check queue every 30s
  cooldownPeriod: 300               # Wait 5min before scaling to 0
  triggers:
  - type: azure-servicebus
    metadata:
      queueName: sbq-orders
      namespace: sb-prod-cc         # Service Bus namespace name
      messageCount: "500"           # 1 pod per 500 messages
    authenticationRef:
      name: keda-sb-auth            # TriggerAuthentication below

---
# TriggerAuthentication — uses Workload Identity (no secrets)
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: keda-sb-auth
  namespace: production
spec:
  podIdentity:
    provider: azure-workload
    identityId: <MI_CLIENT_ID>      # KEDA's user-assigned managed identity
```

```bash
# Grant KEDA MI access to read Service Bus queue metrics
SB_ID=$(az servicebus namespace show \
  --name sb-prod-cc -g rg-prod --query id -o tsv)

az role assignment create \
  --role "Azure Service Bus Data Owner" \
  --assignee $KEDA_MI_OBJECT_ID \
  --scope $SB_ID

# Check KEDA is scaling correctly
kubectl get scaledobject -n production
# Expected: READY=True, ACTIVE=True/False (False = queue empty, scaled to 0)

kubectl get hpa -n production   # KEDA creates an HPA internally
```

> ⚠️ **Gotcha:** When `minReplicaCount: 0`, the first message in the queue after a scale-to-zero causes a cold start delay (typically 30–60s while a new pod schedules and starts). For latency-sensitive workloads, set `minReplicaCount: 1` instead.

> 💡 **Deep dive hint:** KEDA ScaledJob for long-running batch tasks — instead of scaling a Deployment, KEDA can create individual Kubernetes Jobs per message batch, with each Job processing a fixed chunk then terminating.

---

## Q3. How does Cluster Autoscaler work and what are its limits?

**Scenario:** Your cluster has 10 nodes and KEDA just scaled pods to 80. New pods are stuck in `Pending` state. You need nodes to scale out automatically without manual intervention.

```
Pod Pending → Cluster Autoscaler kicks in:

  kubectl get pods → payment-api-7f9d4 Pending
                     order-proc-2k8cx Pending (2 pods, no node fits)
       │
       ▼
  Cluster Autoscaler scans unschedulable pods
       │ Simulates: "if I add a node, would these pods schedule?"
       │ YES → trigger node scale-out
       ▼
  Azure VMSS: adds nodes (up to --max-count)
       │ New node joins cluster (~2–5 min for new node to be Ready)
       ▼
  Scheduler places Pending pods on new node ✅

Scale-IN (node removal):
  Node utilisation < 50% for 10min
  + all pods can be rescheduled elsewhere
  → Autoscaler drains and removes node
```

```bash
# Cluster Autoscaler is enabled per node pool — set at creation or update
az aks nodepool update \
  --cluster-name aks-prod-cc \
  --resource-group rg-spoke-aks \
  --name userpool \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 50

# View Cluster Autoscaler activity log
kubectl get events -n kube-system \
  --field-selector reason=TriggeredScaleUp

# Check why a pod is Pending (autoscaler reason)
kubectl describe pod <pending-pod-name> -n production
# Look for: "0/10 nodes are available: 10 Insufficient cpu"
# → Autoscaler will add nodes if max not reached

# Cluster Autoscaler status ConfigMap
kubectl get configmap cluster-autoscaler-status -n kube-system -o yaml
# Shows: lastScaleUpTime, lastScaleDownTime, nodeGroups
```

**Node pool design for autoscaling:**
```yaml
# System pool: fixed size (no autoscaler — system pods must always run)
# User pool: autoscaled (your workloads)

# Separate node pools by workload type:
#   General:    Standard_D4s_v3  (default)
#   Memory:     Standard_E8s_v3  (Redis, stateful apps)
#   GPU:        Standard_NC6s_v3 (ML inference)
#   Spot:       Standard_D4s_v3  spot (batch, non-critical — 90% cheaper)
```

```bash
# Add a Spot node pool for cost savings on batch workloads
az aks nodepool add \
  --cluster-name aks-prod-cc \
  --resource-group rg-spoke-aks \
  --name spotpool \
  --priority Spot \
  --eviction-policy Delete \
  --spot-max-price -1 \           # -1 = pay market price (max Azure Spot price)
  --node-vm-size Standard_D4s_v3 \
  --enable-cluster-autoscaler \
  --min-count 0 \
  --max-count 20

# Pods must tolerate the Spot taint to land on Spot nodes
# AKS automatically taints Spot nodes with:
#   kubernetes.azure.com/scalesetpriority=spot:NoSchedule
```

> ⚠️ **Gotcha:** Cluster Autoscaler will NOT scale down a node if any pod on it has no `PodDisruptionBudget` or if it has `local storage` (hostPath / emptyDir with data). Add PDBs to all stateful workloads or Autoscaler gets stuck with underutilised nodes it can't remove.

> 💡 **Deep dive hint:** AKS node surge during upgrades vs autoscaler — `--max-surge` adds temporary extra nodes during upgrade (not counted against `--max-count`). Understanding the difference prevents "upgrade failed — max node count reached" errors.

---

## Q4. How do you scale AKS pods on a custom Prometheus metric using KEDA?

**Scenario:** Your API Gateway tracks active HTTP connections in Prometheus. You want pods to scale based on `http_active_connections` rather than CPU — CPU stays low even under connection saturation.

```
Custom metric scaling flow:

  Prometheus scrapes pods
       │ http_active_connections{app="api-gateway"} = 850
       ▼
  KEDA Prometheus scaler
       │ queries: sum(http_active_connections{app="api-gateway"})
       │ current = 850, threshold = 100 per pod
       │ desired = ceil(850 ÷ 100) = 9 pods
       ▼
  ScaledObject → HPA → Deployment scales to 9 pods
```

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: api-gateway-scaler
  namespace: production
spec:
  scaleTargetRef:
    name: api-gateway
  minReplicaCount: 2
  maxReplicaCount: 30
  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus-server.monitoring.svc.cluster.local
      metricName: http_active_connections
      query: sum(http_active_connections{app="api-gateway"})
      threshold: "100"              # 1 pod per 100 active connections
```

```bash
# Verify KEDA is reading the metric correctly
kubectl describe scaledobject api-gateway-scaler -n production
# Look for: "External metric ... current value"
# If metric = 0 when it should be non-zero: check Prometheus query syntax

# Port-forward Prometheus to test the query locally
kubectl port-forward svc/prometheus-server 9090:9090 -n monitoring
# Then open: http://localhost:9090 and run the query manually
```

> ⚠️ **Gotcha:** KEDA's Prometheus trigger queries the Prometheus HTTP API — if Prometheus is behind authentication or TLS, you must configure the `TriggerAuthentication` with the appropriate headers or certificate. Unauthenticated queries return a 401 and KEDA scales to `minReplicaCount` silently.
