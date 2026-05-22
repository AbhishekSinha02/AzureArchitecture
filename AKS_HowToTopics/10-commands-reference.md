# AKS Commands Reference
### All kubectl / az aks / helm commands with context

> *"Know the command. Know why you're running it. Know what the output means."*

---

## Cluster Management

```bash
# Get cluster info and credentials
az aks get-credentials \
  --name aks-prod-cc \
  --resource-group rg-spoke-aks
# Effect: writes kubeconfig to ~/.kube/config

# List clusters in subscription
az aks list --output table
# Columns: Name, Location, KubernetesVersion, ProvisioningState, PowerState

# Show cluster details including OIDC URL, identity, and networking
az aks show \
  --name aks-prod-cc \
  --resource-group rg-spoke-aks \
  --query "{k8sVersion:kubernetesVersion, oidcUrl:oidcIssuerProfile.issuerUrl, \
           networkPlugin:networkProfile.networkPlugin, tier:sku.tier}" \
  --output table

# Get available Kubernetes upgrade versions
az aks get-upgrades \
  --name aks-prod-cc \
  --resource-group rg-spoke-aks \
  --output table

# Upgrade control plane only
az aks upgrade \
  --name aks-prod-cc \
  --resource-group rg-spoke-aks \
  --kubernetes-version 1.29 \
  --control-plane-only
# Duration: 10–20 minutes. Nodes stay on old version until nodepool upgrade.

# Start / Stop cluster (cost saving for non-prod)
az aks stop --name aks-dev --resource-group rg-dev
az aks start --name aks-dev --resource-group rg-dev
```

---

## Node Pool Operations

```bash
# List node pools
az aks nodepool list \
  --cluster-name aks-prod-cc \
  --resource-group rg-spoke-aks \
  --output table

# Add a new user node pool
az aks nodepool add \
  --cluster-name aks-prod-cc \
  --resource-group rg-spoke-aks \
  --name gpupool \
  --node-vm-size Standard_NC6s_v3 \
  --node-count 2 \
  --node-taints "gpu=true:NoSchedule" \
  --zones 1 2 3

# Upgrade node pool to new Kubernetes version
az aks nodepool upgrade \
  --cluster-name aks-prod-cc \
  --resource-group rg-spoke-aks \
  --name userpool \
  --kubernetes-version 1.29 \
  --max-surge 33%
# max-surge: extra nodes added during upgrade = faster + no capacity shortage

# Scale node pool manually (override autoscaler)
az aks nodepool scale \
  --cluster-name aks-prod-cc \
  --resource-group rg-spoke-aks \
  --name userpool \
  --node-count 10

# Update autoscaler min/max
az aks nodepool update \
  --cluster-name aks-prod-cc \
  --resource-group rg-spoke-aks \
  --name userpool \
  --min-count 5 \
  --max-count 50

# Node pool status
az aks nodepool show \
  --cluster-name aks-prod-cc \
  --resource-group rg-spoke-aks \
  --name userpool \
  --query "{version:orchestratorVersion, count:count, state:provisioningState}"
```

---

## kubectl — Pods & Deployments

```bash
# Get all pods across all namespaces with node placement
kubectl get pods --all-namespaces -o wide

# Get pods in a namespace with labels
kubectl get pods -n production --show-labels

# Describe pod (events, volumes, resource limits)
kubectl describe pod <pod-name> -n production

# Logs — current and previous run
kubectl logs <pod-name> -n production
kubectl logs <pod-name> -n production --previous    # Last crash
kubectl logs <pod-name> -n production -f            # Follow (streaming)
kubectl logs <pod-name> -n production --tail=100    # Last 100 lines

# Exec into running pod
kubectl exec -it <pod-name> -n production -- /bin/sh

# Debug crashed pod (copy with debug container)
kubectl debug -it <pod-name> -n production \
  --copy-to=debug-pod \
  --image=busybox:1.28 \
  -- /bin/sh

# Delete pod (restarts it via Deployment controller)
kubectl delete pod <pod-name> -n production

# Restart all pods in a deployment (rolling restart)
kubectl rollout restart deployment/<name> -n production

# Rollback deployment
kubectl rollout undo deployment/<name> -n production
kubectl rollout undo deployment/<name> -n production --to-revision=3

# Rollout history
kubectl rollout history deployment/<name> -n production

# Scale deployment
kubectl scale deployment/<name> --replicas=5 -n production

# Update image (triggers rolling update)
kubectl set image deployment/<name> \
  <container-name>=myacr.azurecr.io/app:v1.2 \
  -n production

# Watch rollout progress
kubectl rollout status deployment/<name> -n production --watch
```

---

## kubectl — Nodes

```bash
# Get nodes with zone and status
kubectl get nodes -o wide
kubectl get nodes \
  -o custom-columns='NAME:.metadata.name,ZONE:.metadata.labels.topology\.kubernetes\.io/zone,STATUS:.status.conditions[-1].type'

# Describe node (capacity, allocatable, running pods)
kubectl describe node <node-name>

# Resource usage per node
kubectl top nodes
kubectl top pods --all-namespaces --sort-by=cpu | head -20

# Cordon — prevent new pods scheduling on node (but keep existing pods)
kubectl cordon <node-name>

# Drain — evict all pods (cordon + graceful eviction, respects PDB)
kubectl drain <node-name> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=60

# Uncordon — allow scheduling again
kubectl uncordon <node-name>

# Check effective resource limits on a node
kubectl get node <node-name> \
  -o jsonpath='{.status.allocatable}'
```

---

## kubectl — Services & Networking

```bash
# Get services with ClusterIP and external IP
kubectl get svc --all-namespaces

# Check endpoints (are pods matched to service?)
kubectl get endpoints <service-name> -n production
# <none> means label selector doesn't match any pod

# Test DNS resolution from inside cluster
kubectl run dns-test --image=busybox:1.28 --rm -it -- \
  nslookup payment-api-svc.production.svc.cluster.local
# Expected: returns ClusterIP

# Test DNS for Private Endpoint
kubectl run dns-test --image=busybox:1.28 --rm -it -- \
  nslookup kv-prod.vault.azure.net
# Expected: 10.x.x.x (private IP), not 52.x (public)

# Test HTTP connectivity from inside cluster
kubectl run curl-test --image=curlimages/curl --rm -it -- \
  curl -s http://payment-api-svc:8080/health
# Expected: {"status":"ok"}

# Port-forward service to local machine
kubectl port-forward svc/payment-api-svc 8080:8080 -n production
# Access: http://localhost:8080

# Get NetworkPolicies
kubectl get networkpolicy -n production
kubectl describe networkpolicy default-deny-all -n production
```

---

## kubectl — Storage

```bash
# List PersistentVolumeClaims
kubectl get pvc -n production
# STATUS: Bound = healthy, Pending = StorageClass issue

# Describe PVC (check events for provisioning errors)
kubectl describe pvc <pvc-name> -n production

# Expand PVC (StorageClass must have allowVolumeExpansion: true)
kubectl patch pvc <pvc-name> -n production \
  -p '{"spec": {"resources": {"requests": {"storage": "200Gi"}}}}'

# List StorageClasses
kubectl get storageclass

# Get PersistentVolumes (cluster-wide, not namespaced)
kubectl get pv

# Check which node a StatefulSet pod is on (for disk zone matching)
kubectl get pod -n production -o wide -l app=postgres
```

---

## kubectl — RBAC & Identity

```bash
# Check what a ServiceAccount can do
kubectl auth can-i get secrets \
  --as=system:serviceaccount:production:payment-service-sa \
  -n production
# yes / no

# Check all permissions for a ServiceAccount
kubectl auth can-i --list \
  --as=system:serviceaccount:production:payment-service-sa \
  -n production

# Get ServiceAccount details (check annotations for Workload Identity)
kubectl describe serviceaccount payment-service-sa -n production
# Look for: Annotations: azure.workload.identity/client-id

# Get ClusterRoleBindings for a subject
kubectl get clusterrolebindings -o json | \
  jq '.items[] | select(.subjects[]?.name=="payment-service-sa")'
```

---

## Workload Identity Setup

```bash
# Full Workload Identity setup sequence
# (see 02-workload-identity-security.md for full context)

# Enable OIDC + Workload Identity on cluster
az aks update \
  --name aks-prod-cc \
  --resource-group rg-spoke-aks \
  --enable-oidc-issuer \
  --enable-workload-identity

# Get OIDC issuer URL
az aks show \
  --name aks-prod-cc \
  --resource-group rg-spoke-aks \
  --query oidcIssuerProfile.issuerUrl \
  --output tsv

# Create federated credential
az identity federated-credential create \
  --identity-name id-payment-service \
  --resource-group rg-spoke-aks \
  --name fed-cred-payment \
  --issuer <OIDC_URL> \
  --subject "system:serviceaccount:production:payment-service-sa" \
  --audience "api://AzureADTokenExchange"

# Verify federated credential
az identity federated-credential list \
  --identity-name id-payment-service \
  --resource-group rg-spoke-aks \
  --output table
```

---

## Managed Identity & ACR

```bash
# Get both managed identity IDs
az aks show \
  --name aks-prod-cc \
  --resource-group rg-spoke-aks \
  --query "{ControlPlanePrincipalId:identity.principalId, KubeletClientId:identityProfile.kubeletidentity.clientId, KubeletObjectId:identityProfile.kubeletidentity.objectId}"

# Grant Kubelet MI permission to pull from ACR
az role assignment create \
  --role AcrPull \
  --assignee <KUBELET_OBJECT_ID> \
  --scope $(az acr show --name myacr -g rg-acr --query id -o tsv)

# Verify AcrPull assignment
az role assignment list \
  --assignee <KUBELET_OBJECT_ID> \
  --query "[?roleDefinitionName=='AcrPull']" \
  --output table

# List images in ACR repository
az acr repository list --name myacr --output table
az acr repository show-tags --name myacr --repository payment --output table
```

---

## Helm Operations

```bash
# Add a Helm chart repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add kedacore https://kedacore.github.io/charts
helm repo update

# Install a chart
helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --values nginx-values.yaml

# Upgrade a release
helm upgrade nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --values nginx-values.yaml \
  --wait --timeout 5m     # Wait for pods to be Ready

# List installed releases
helm list --all-namespaces

# Check release history
helm history nginx-ingress -n ingress-nginx

# Rollback a release
helm rollback nginx-ingress 1 -n ingress-nginx   # Roll back to revision 1

# Render templates without installing (dry run)
helm template nginx-ingress ingress-nginx/ingress-nginx \
  --values nginx-values.yaml | kubectl apply --dry-run=client -f -

# Uninstall a release
helm uninstall nginx-ingress -n ingress-nginx
```

---

## KEDA

```bash
# Check ScaledObject status
kubectl get scaledobject -n production
# READY=True, ACTIVE=True (queue has messages) / False (queue empty, scaled to 0)

# Describe for detailed trigger info
kubectl describe scaledobject order-processor-scaler -n production

# Check TriggerAuthentication
kubectl get triggerauthentication -n production

# View HPA that KEDA created
kubectl get hpa -n production
# KEDA creates one HPA per ScaledObject (name: keda-hpa-<scaledobject-name>)
```

---

## Container Insights & Monitoring

```bash
# Enable Container Insights add-on
az aks enable-addons \
  --addons monitoring \
  --name aks-prod-cc \
  --resource-group rg-spoke-aks \
  --workspace-resource-id <LAW_RESOURCE_ID>

# Check oms-agent DaemonSet is running
kubectl get pods -n kube-system -l component=oms-agent

# Run KQL query from CLI
az monitor log-analytics query \
  --workspace <LAW_WORKSPACE_ID> \
  --analytics-query "KubePodInventory | where ContainerRestartCount > 3 | take 10"

# Flux reconciliation status
flux get all
flux reconcile kustomization flux-system --with-source
```

---

## Quick Diagnostic One-Liners

```bash
# All non-Running pods across cluster
kubectl get pods --all-namespaces | grep -v Running | grep -v Completed

# Pod restart counts (find crashy pods)
kubectl get pods --all-namespaces \
  -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,RESTARTS:.status.containerStatuses[0].restartCount' \
  | sort -k3 -rn | head -20

# Events sorted by time (recent first)
kubectl get events --all-namespaces \
  --sort-by='.metadata.creationTimestamp' | tail -30

# What's using the most CPU right now
kubectl top pods --all-namespaces --sort-by=cpu | head -15

# HPA status all namespaces
kubectl get hpa --all-namespaces

# Check Cluster Autoscaler last action
kubectl -n kube-system logs -l app=cluster-autoscaler --tail=50 | grep -i scale

# All resource requests/limits for a namespace
kubectl get pods -n production -o json | \
  jq -r '.items[].spec.containers[] | [.name, .resources.requests.cpu, .resources.requests.memory, .resources.limits.cpu, .resources.limits.memory] | @csv'
```
