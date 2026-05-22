# AKS Troubleshooting Playbook
### 🟡 Intermediate → 🔴 Advanced

> *"Every Kubernetes error message is a clue.  
>  The trick is knowing which component generated it."*

---

## Q1. How do you diagnose and fix a pod stuck in CrashLoopBackOff?

**Scenario:** A newly deployed pod shows `CrashLoopBackOff`. The deployment was working yesterday. Rollout is blocking and the team is waiting.

```
CrashLoopBackOff diagnosis tree:

  kubectl get pod → STATUS: CrashLoopBackOff
       │
       ├─ kubectl describe pod → Events section
       │       │
       │       ├─ "OOMKilled"         → memory limit too low
       │       ├─ "Error: exit 1"     → app startup error
       │       ├─ "Liveness probe"    → probe config wrong
       │       └─ "Back-off restart"  → check logs
       │
       └─ kubectl logs <pod> --previous → last run's stdout/stderr
              │
              ├─ "Connection refused to db:5432"  → DB not reachable
              ├─ "Failed to read /mnt/secrets"    → CSI mount failed
              ├─ "FATAL: config not found"        → missing ConfigMap/env
              └─ "permission denied"              → security context issue
```

```bash
# Step 1 — Get pod status and recent events
kubectl describe pod <pod-name> -n production
# Look at: "Events" section at the bottom — most valuable field

# Step 2 — Get logs from the crashed container (previous run)
kubectl logs <pod-name> -n production --previous
# --previous shows logs from the LAST crash, not the current restart

# Step 3 — Check if it's OOMKilled
kubectl get pod <pod-name> -n production \
  -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
# "OOMKilled" → increase memory limit

# Step 4 — Temporarily override the command to debug
kubectl debug -it <pod-name> -n production \
  --copy-to=debug-pod \
  --container=<container> \
  -- /bin/sh
# Opens a shell in a copy of the pod so you can inspect the environment

# Step 5 — Check resource usage before crash
kubectl top pod <pod-name> -n production --containers
```

**Common fixes by root cause:**
```yaml
# OOMKilled — increase memory limit
resources:
  requests:
    memory: "256Mi"
  limits:
    memory: "1Gi"    # Was 256Mi — app needed more on startup

# Liveness probe killing healthy app (startup too slow)
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 60    # Was 10 — app takes 45s to start
  periodSeconds: 10
  failureThreshold: 3

# App can't find config — check env vars and ConfigMaps
kubectl get configmap <name> -n production -o yaml
kubectl exec <pod-name> -n production -- env | grep CONFIG
```

> ⚠️ **Gotcha:** `kubectl logs <pod>` without `--previous` shows logs from the CURRENT (running or latest) instance — which may be empty if the pod just restarted. Always use `--previous` for CrashLoopBackOff debugging.

---

## Q2. How do you debug a pod stuck in Pending state?

**Scenario:** A new deployment has 5 pods, 3 are Running, 2 are stuck in `Pending` for 10 minutes. KEDA just scaled but nodes aren't appearing.

```
Pending pod diagnosis:

  kubectl describe pod → Events:
       │
       ├─ "0/10 nodes available: 10 Insufficient cpu"
       │   → Resource requests too high, or cluster at max
       │   → Check: is Cluster Autoscaler maxed out?
       │
       ├─ "0/10 nodes available: pod has unbound PVC"
       │   → PVC not Bound yet → check StorageClass / disk quota
       │
       ├─ "0/10 nodes available: node(s) had taint"
       │   → Node has taint pod doesn't tolerate
       │   → Typical: system pool taints CriticalAddonsOnly
       │
       └─ "0/10 nodes available: topology constraint"
           → topologySpreadConstraints preventing scheduling
           → Not enough nodes in required zone
```

```bash
# Step 1 — See exactly why pod can't schedule
kubectl describe pod <pending-pod> -n production
# Events section: "0/6 nodes are available: 3 Insufficient cpu, 3 node(s) had taint"

# Step 2 — Check Cluster Autoscaler activity
kubectl get events -n kube-system | grep -i scale
# "Successfully added node" → scaling in progress, wait 2–5min
# No event → Autoscaler blocked (check max-count reached)

kubectl get configmap cluster-autoscaler-status -n kube-system -o yaml
# Look for: "lastScaleUpTime" and "errors"

# Step 3 — Check node capacity
kubectl describe nodes | grep -A 5 "Allocated resources"
# Shows: CPU requests, memory requests per node

# Step 4 — Check if max node count is reached
az aks nodepool show \
  --cluster-name aks-prod-cc -g rg-spoke-aks \
  --name userpool \
  --query "{current:count, max:maxCount}"

# Step 5 — Inspect taints and tolerations
kubectl get nodes -o custom-columns='NAME:.metadata.name,TAINTS:.spec.taints'
```

> ⚠️ **Gotcha:** A pod requesting `cpu: "4"` on a `Standard_D4s_v3` node (4 vCPU) will always be Pending — the node has 4 allocatable vCPUs but system daemons consume ~0.3 vCPU, leaving only ~3.7 allocatable. Request `cpu: "3.5"` or use a larger node SKU.

---

## Q3. How do you diagnose ImagePullBackOff and fix ACR authentication?

**Scenario:** Pods fail with `ImagePullBackOff`. The image `myacr.azurecr.io/payment:1.5` was just built and pushed. New pods can't pull it.

```
Image pull failure chain:

  kubelet on node
       │ tries to pull: myacr.azurecr.io/payment:1.5
       ▼
  Azure Container Registry
       │ requires auth: Kubelet Managed Identity must have AcrPull role
       │
       ├─ AcrPull on Kubelet MI → ✅ pull succeeds
       └─ AcrPull missing       → 401 Unauthorized → ImagePullBackOff

  Also fails if:
  ├─ Image tag doesn't exist (typo in tag)
  ├─ ACR behind Private Endpoint — node VNet not linked to Private DNS Zone
  └─ Network policy blocks node → ACR traffic
```

```bash
# Step 1 — Get the exact error
kubectl describe pod <pod-name> -n production | grep -A 5 "Events"
# "Failed to pull image ... 401 Unauthorized" → auth issue
# "Failed to pull image ... not found"        → wrong tag
# "Failed to pull image ... dial tcp: i/o timeout" → network/DNS issue

# Step 2 — Verify Kubelet MI has AcrPull
KUBELET_OID=$(az aks show \
  --name aks-prod-cc -g rg-spoke-aks \
  --query identityProfile.kubeletidentity.objectId -o tsv)

az role assignment list \
  --assignee $KUBELET_OID \
  --query "[?roleDefinitionName=='AcrPull']"
# Empty result → AcrPull not assigned → this is the bug

# Step 3 — Fix: assign AcrPull to Kubelet MI
ACR_ID=$(az acr show --name myacr -g rg-acr --query id -o tsv)
az role assignment create \
  --role AcrPull \
  --assignee $KUBELET_OID \
  --scope $ACR_ID
# Takes 2–5 minutes to propagate

# Step 4 — Verify image tag exists
az acr repository show-tags \
  --name myacr \
  --repository payment \
  --output table

# Step 5 — Test DNS resolution to ACR from a pod
kubectl run dns-test --image=busybox:1.28 --rm -it -- \
  nslookup myacr.azurecr.io
# Expected: private IP 10.x.x.x (not 52.x which would be public)
```

> ⚠️ **Gotcha:** AKS `--attach-acr` flag at cluster creation is a shortcut but only works for ACRs in the same subscription. For cross-subscription ACRs, you must manually assign `AcrPull` to the Kubelet MI. The `--attach-acr` flag silently does nothing if the ACR is in a different subscription.

---

## Q4. How do you diagnose and fix a Service not routing traffic to pods?

**Scenario:** `kubectl get svc` shows a ClusterIP. But `curl http://payment-api-svc:8080/health` from another pod times out.

```
Service routing failure tree:

  Service ClusterIP: 172.16.0.50:8080
       │ iptables / kube-proxy routes to pod IPs
       ▼
  Endpoints: 10.1.2.4:8080, 10.1.2.5:8080  ← check this!
       │
       ├─ Endpoints EMPTY → label selector mismatch between Service and pods
       ├─ Endpoints populated but timeout → Network Policy blocking
       └─ Endpoints populated, no timeout → app not listening on correct port
```

```bash
# Step 1 — Check if Service has endpoints
kubectl get endpoints payment-api-svc -n production
# NAME              ENDPOINTS              AGE
# payment-api-svc   10.1.2.4:8080,...      5m   ← healthy
# payment-api-svc   <none>                 5m   ← no pods matched

# Step 2 — If endpoints empty: check label selector mismatch
kubectl describe svc payment-api-svc -n production | grep Selector
# Selector: app=payment-api
kubectl get pods -n production --show-labels | grep payment
# Verify pods have EXACTLY: app=payment-api (case-sensitive, exact match)

# Step 3 — Test direct pod connectivity (bypass Service)
POD_IP=$(kubectl get pod <payment-pod> -n production \
  -o jsonpath='{.status.podIP}')
kubectl run curl-test --image=curlimages/curl --rm -it -- \
  curl http://$POD_IP:8080/health
# ✅ works → Service selector is broken
# ❌ fails → NetworkPolicy or app issue

# Step 4 — Check NetworkPolicy (is traffic from test pod blocked?)
kubectl get networkpolicy -n production
# If default-deny exists: check if your test pod namespace is allowed

# Step 5 — Verify app is listening on the correct port inside the container
kubectl exec <payment-pod> -n production -- \
  netstat -tlnp | grep 8080
# If nothing on 8080 → app is listening on a different port (common: 80 vs 8080)
```

> ⚠️ **Gotcha:** Service port and `targetPort` are frequently confused. `port: 8080` is the port the Service exposes on its ClusterIP. `targetPort: 8080` is the port the pod container listens on. They can differ. If the container listens on port 80 but `targetPort: 8080`, all traffic to the Service silently fails.

---

## Q5. How do you diagnose Workload Identity authentication failures?

**Scenario:** Your pod has Workload Identity configured. It starts without error, but when it tries to read from Key Vault it gets `DefaultAzureCredential: ClientAuthenticationError`. 

```
Workload Identity failure checklist:

  1. Pod label: azure.workload.identity/use: "true"  ← webhook injects token volume
  2. Pod spec: serviceAccountName: payment-service-sa
  3. ServiceAccount annotation: azure.workload.identity/client-id: <MI_CLIENT_ID>
  4. Federated credential in Azure: subject matches SA + namespace + OIDC URL
  5. MI has correct role on Key Vault
  6. Key Vault network: private endpoint accessible from pod subnet

  If ANY of these are wrong → 401 from Azure AD
```

```bash
# Step 1 — Verify the webhook injected the token volume
kubectl describe pod <pod-name> -n production | grep -A 10 "Volumes:"
# Expected volume: azure-identity-token (projected, serviceAccountToken)
# Missing → pod label azure.workload.identity/use: "true" not set

# Step 2 — Check ServiceAccount annotation
kubectl describe serviceaccount payment-service-sa -n production
# Expected annotation: azure.workload.identity/client-id: <MI_CLIENT_ID>

# Step 3 — Check federated credential matches exactly
OIDC_URL=$(az aks show \
  --name aks-prod-cc -g rg-spoke-aks \
  --query oidcIssuerProfile.issuerUrl -o tsv)
echo "Expected subject: system:serviceaccount:production:payment-service-sa"
echo "Expected issuer: $OIDC_URL"

az identity federated-credential show \
  --identity-name id-payment-service \
  --resource-group rg-spoke-aks \
  --name fed-cred-payment-service \
  --query "{issuer:issuer, subject:subject}"
# Values must match EXACTLY (case-sensitive)

# Step 4 — Check MI has Key Vault access
az keyvault show --name kv-prod --query "properties.accessPolicies" -o table

# Step 5 — Decode the projected token to inspect claims
kubectl exec <pod-name> -n production -- \
  cat /var/run/secrets/azure/tokens/azure-identity-token | \
  cut -d. -f2 | base64 -d 2>/dev/null | python3 -m json.tool
# Check: "sub" matches federated credential subject
```

> ⚠️ **Gotcha:** If the AKS cluster was created with OIDC issuer disabled and you later enabled it (`--enable-oidc-issuer`), the OIDC issuer URL changes. Any existing federated credentials with the old issuer URL stop working. Update all federated credentials with the new `oidcIssuerProfile.issuerUrl`.

---

## Q6. How do you diagnose OOMKill and set correct memory limits?

**Scenario:** A pod is repeatedly OOMKilled at 3am during report generation. The memory limit was set by copy-paste from another service. Nobody knows what the actual memory usage is.

```
OOMKill detection:

  Node kernel kills the container process when:
  container RSS memory > memory limit in pod spec

  kubectl describe pod → Events:
  "OOMKilling: Memory cgroup out of memory: Kill process [app] (memory.limit_in_bytes)"
  
  kubectl get pod → RESTARTS: 5  ← this tells you it keeps happening

  How to find the right limit:
  1. Run pod without limits (dev/staging only)
  2. kubectl top pod — observe peak memory
  3. Set limit = peak × 1.5 (safety margin)
  4. Set request = typical steady-state usage
```

```bash
# Check if OOMKilled (from outside the pod)
kubectl get pod <pod-name> -n production \
  -o jsonpath='{.status.containerStatuses[0].lastState.terminated}'
# Expected: {"exitCode":137, "reason":"OOMKilled", "finishedAt":"..."}
# exit code 137 = 128 + 9 (SIGKILL) = OOM

# Check memory metrics for the pod (requires metrics-server)
kubectl top pod <pod-name> -n production --containers

# KQL query to see memory trend before the kill (Container Insights)
# Perf
# | where CounterName == "memoryRssBytes" and InstanceName contains "payment"
# | where TimeGenerated between (datetime(2024-01-15 02:50) .. datetime(2024-01-15 03:05))
# | project TimeGenerated, MemoryMB = CounterValue / 1048576
# | render timechart

# Set memory limit based on observation
kubectl set resources deployment payment-api \
  -n production \
  --requests=memory=512Mi \
  --limits=memory=1.5Gi
```

> ⚠️ **Gotcha:** Memory requests and limits mean different things. `requests` is the guaranteed allocation (used for scheduling decisions). `limits` is the hard cap (OOMKill threshold). Always set `requests` lower than `limits`. Setting `requests = limits` (Guaranteed QoS) means Kubernetes never OOMKills the pod at the container level — but the node itself may still evict it under memory pressure.
