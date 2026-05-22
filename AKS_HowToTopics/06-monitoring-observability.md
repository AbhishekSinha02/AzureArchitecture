# AKS Monitoring & Observability
### 🟡 Intermediate → 🔴 Advanced

> *"A cluster you can't observe is a cluster you can't trust.  
>  Metrics tell you something is wrong. Traces tell you why."*

---

## Q1. How do you enable and query Azure Monitor Container Insights for AKS?

**Scenario:** Ops is getting paged for "AKS is slow" with no details. You need visibility into pod CPU/memory, node saturation, and container restarts — without deploying any custom tooling.

```
Container Insights data flow:

  AKS nodes / pods
       │ (omsagent DaemonSet collects metrics + logs)
       ▼
  Log Analytics Workspace (law-aks-prod)
       │
       ├─── ContainerLog     (stdout/stderr from all containers)
       ├─── KubePodInventory (pod state, restarts, node placement)
       ├─── Perf             (CPU, memory per container/node)
       └─── KubeEvents       (Kubernetes events: OOMKill, Eviction, etc.)
       │
       ▼
  Azure Monitor Workbooks (pre-built AKS dashboards)
  + Custom KQL alerts
```

```bash
# Enable Container Insights on existing cluster
az aks enable-addons \
  --addons monitoring \
  --name aks-prod-cc \
  --resource-group rg-spoke-aks \
  --workspace-resource-id \
    "/subscriptions/<SUB>/resourceGroups/rg-monitoring/providers/Microsoft.OperationalInsights/workspaces/law-aks-prod"

# Verify agent is running on every node
kubectl get pods -n kube-system -l component=oms-agent
# Expected: one oms-agent pod per node, all Running
```

**KQL queries for common operational questions:**

```kusto
// Top 5 pods by CPU usage in the last hour
Perf
| where ObjectName == "K8SContainer" and CounterName == "cpuUsageNanoCores"
| where TimeGenerated > ago(1h)
| summarize AvgCPU = avg(CounterValue) by bin(TimeGenerated, 5m), Computer, InstanceName
| top 5 by AvgCPU desc

// Pods that restarted more than 3 times today
KubePodInventory
| where TimeGenerated > startofday(now())
| where ContainerRestartCount > 3
| project TimeGenerated, Namespace, PodName = Name, 
          ContainerRestartCount, PodStatus
| order by ContainerRestartCount desc

// OOMKill events in the last 24 hours
KubeEvents
| where TimeGenerated > ago(24h)
| where Reason == "OOMKilling" or Message contains "OOMKilled"
| project TimeGenerated, Namespace, Name, Message
```

```bash
# Quick check without KQL — from kubectl
kubectl top pods --all-namespaces --sort-by=cpu | head -20
kubectl top nodes
```

> ⚠️ **Gotcha:** Container Insights collects logs at a default sample rate — by default it does NOT collect all container logs to control cost. Check `ContainerLogV2` schema (newer, lower cost) vs `ContainerLog`. If logs are missing, check the `ConfigMap` named `container-azm-ms-agentconfig` in `kube-system`.

> 💡 **Deep dive hint:** Container Insights cost control — at scale, Log Analytics ingestion costs dominate. Set up `Basic` log tier for verbose/debug logs, and `Analytics` tier only for operational logs you actively query or alert on.

---

## Q2. How do you deploy Prometheus + Grafana on AKS and scrape custom application metrics?

**Scenario:** The Container Insights pre-built dashboards aren't enough. Your team wants Grafana dashboards for business metrics (orders-per-second, payment success rate) alongside infrastructure metrics.

```
Prometheus scraping architecture in AKS:

  App pods (expose /metrics on port 9090)
       │ HTTP GET /metrics every 30s
       ▼
  Prometheus Server (stateful pod + PVC for TSDB)
       │ stores metrics for 15 days
       ▼
  Grafana (queries Prometheus via DataSource)
       │ renders dashboards
       ▼
  Users / On-call engineers

  Optional: Azure Managed Grafana ──► replaces self-hosted Grafana
            Azure Monitor Managed Prometheus ──► replaces self-hosted Prometheus
```

```bash
# Install kube-prometheus-stack (Prometheus + Grafana + AlertManager + node-exporter)
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts

helm install kube-prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.adminPassword="<STRONG_PASSWORD>" \
  --set prometheus.prometheusSpec.retention="15d" \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage="50Gi"
```

**Annotate pods to enable Prometheus scraping:**
```yaml
# Add annotations to your deployment pod spec
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    prometheus.io/path: "/metrics"
```

**Or use ServiceMonitor (recommended — no annotations needed):**
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: payment-api-monitor
  namespace: monitoring
  labels:
    release: kube-prometheus   # Must match Prometheus operator selector
spec:
  selector:
    matchLabels:
      app: payment-api         # Selects Services with this label
  namespaceSelector:
    matchNames:
    - production
  endpoints:
  - port: metrics              # Named port in the Service spec
    interval: 30s
    path: /metrics
```

```bash
# Access Grafana (port-forward for testing)
kubectl port-forward svc/kube-prometheus-grafana 3000:80 -n monitoring
# Open: http://localhost:3000  admin / <password set above>

# Check Prometheus targets (are your pods being scraped?)
kubectl port-forward svc/kube-prometheus-kube-prometheus 9090:9090 -n monitoring
# Open: http://localhost:9090/targets
```

> ⚠️ **Gotcha:** `ServiceMonitor` labels must match the Prometheus operator's `serviceMonitorSelector`. Check the selector with: `kubectl get prometheus -n monitoring -o yaml | grep serviceMonitorSelector`. A ServiceMonitor that doesn't match is silently ignored — Prometheus shows no error.

> 💡 **Deep dive hint:** Azure Managed Prometheus + Azure Managed Grafana — Microsoft-managed versions of both, eliminating the need to manage Prometheus storage or Grafana upgrades. Data flows to Azure Monitor workspace. Costs more than self-hosted but no ops overhead.

---

## Q3. How do you create an alert when a pod crashes more than 3 times in 5 minutes?

**Scenario:** The on-call team is not being notified when `payment-api` enters CrashLoopBackOff. By the time someone notices, the queue has backed up. You need an automated alert.

```
Alert pipeline:

  KubePodInventory table (Log Analytics)
       │ KQL query runs every 5 minutes
       │ Count pod restarts in last 5 min
       │ threshold: > 3 restarts
       ▼
  Alert fires → Action Group
       │
       ├─── Email to on-call DL
       ├─── PagerDuty webhook
       └─── Teams channel notification
```

```bash
# Create alert rule via Azure CLI
az monitor scheduled-query create \
  --name "AKS-CrashLoopAlert-payment-api" \
  --resource-group rg-monitoring \
  --scopes "/subscriptions/<SUB>/resourceGroups/rg-monitoring/providers/Microsoft.OperationalInsights/workspaces/law-aks-prod" \
  --condition-query "KubePodInventory
    | where TimeGenerated > ago(5m)
    | where Namespace == 'production'
    | where Name contains 'payment-api'
    | where ContainerRestartCount > 3
    | summarize Restarts = max(ContainerRestartCount) by Name
    | where Restarts > 3" \
  --condition-threshold 0 \
  --condition-operator GreaterThan \
  --evaluation-frequency 5m \
  --window-size 5m \
  --severity 1 \
  --action-groups "/subscriptions/<SUB>/resourceGroups/rg-monitoring/providers/microsoft.insights/actionGroups/ag-oncall"
```

**PrometheusRule for Alertmanager (if using Prometheus stack):**
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: pod-crash-alerts
  namespace: monitoring
  labels:
    release: kube-prometheus
spec:
  groups:
  - name: pod.rules
    rules:
    - alert: PodCrashLooping
      expr: rate(kube_pod_container_status_restarts_total{namespace="production"}[5m]) * 60 > 3
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "Pod {{ $labels.pod }} is crash looping"
        description: "{{ $labels.pod }} in namespace {{ $labels.namespace }} has restarted {{ $value | printf \"%.0f\" }} times per minute"
```

```bash
# Test alert routing (Alertmanager)
kubectl port-forward svc/kube-prometheus-alertmanager 9093:9093 -n monitoring
# Open: http://localhost:9093/#/alerts
```

> ⚠️ **Gotcha:** Log Analytics alert queries run against data that has a 2–5 minute ingestion delay. Set `--window-size 10m` (not 5m) to avoid alerts that never fire because the window is too tight to catch the ingested data.

---

## Q4. How do you implement distributed tracing in AKS workloads?

**Scenario:** A request to `order-api` calls `inventory-api` which calls `payment-api`. One of them is slow but logs show "all requests processed." You need end-to-end trace visibility to find the bottleneck.

```
Distributed trace flow:

  Client
    │ HTTP POST /orders
    │ TraceID: abc123 (injected by APIM or app)
    ▼
  order-api (span 1: 350ms)
    │ calls inventory-api
    │ passes TraceID: abc123 + SpanID: s1
    ▼
  inventory-api (span 2: 290ms)   ← SLOW — this is the bottleneck
    │ calls payment-api
    │ passes TraceID: abc123 + SpanID: s2
    ▼
  payment-api (span 3: 12ms)

  All spans collected → Application Insights (or Jaeger/Zipkin)
  → Flame chart shows: 290ms is in inventory-api DB call
```

```bash
# Azure Application Insights — auto-instrumentation via OpenTelemetry

# For .NET apps — add SDK (no code change needed with auto-instrumentation)
# In Dockerfile or deployment: set env var
```

```yaml
env:
- name: APPLICATIONINSIGHTS_CONNECTION_STRING
  valueFrom:
    secretKeyRef:
      name: appinsights-secret
      key: connection-string
- name: OTEL_SERVICE_NAME
  value: "order-api"
- name: OTEL_RESOURCE_ATTRIBUTES
  value: "deployment.environment=production,service.version=1.0.0"
```

```bash
# For Python / Node.js — use OpenTelemetry SDK + Azure Monitor exporter
# pip install azure-monitor-opentelemetry

# Query traces in Application Insights (KQL)
az monitor app-insights query \
  --app ai-aks-prod \
  --analytics-query "
    requests
    | where timestamp > ago(1h)
    | where operation_Name == 'POST /orders'
    | where duration > 1000    -- requests taking > 1 second
    | project timestamp, operation_Id, duration, cloud_RoleName
    | order by duration desc
    | take 20"
```

> ⚠️ **Gotcha:** Distributed tracing only works end-to-end if all services propagate the `traceparent` header (W3C standard) or `x-b3-traceid` (Zipkin). One service in the chain that doesn't forward the header breaks the trace — you see orphaned spans that can't be correlated.
