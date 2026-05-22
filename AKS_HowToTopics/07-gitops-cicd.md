# AKS GitOps & CI/CD
### 🟡 Intermediate → 🔴 Advanced

> *"If you can't deploy from a git commit alone,  
>  you don't have GitOps — you have a pipeline that touches git."*

---

## Q1. How does Flux v2 work for GitOps on AKS and how do you set it up?

**Scenario:** Your team manually runs `kubectl apply` to deploy changes. This caused two incidents where someone applied the wrong manifest. You need deployments fully controlled by Git — no manual kubectl in production.

```
Flux v2 reconciliation loop:

  Developer pushes manifest to Git
       │ git push origin main
       ▼
  GitHub repo: k8s-manifests/production/payment-api.yaml
       │
       │ Flux source-controller polls every 1m
       ▼
  GitRepository object (in-cluster)
       │ detects new commit abc123
       ▼
  Kustomization controller applies manifests
       │ kubectl apply (internally)
       ▼
  AKS cluster state matches Git state ✅

  Drift: someone runs kubectl edit → Flux reverts it next reconcile
```

```bash
# Step 1 — Install Flux CLI
curl -s https://fluxcd.io/install.sh | sudo bash

# Step 2 — Bootstrap Flux to AKS + GitHub repo
flux bootstrap github \
  --owner=<GITHUB_ORG> \
  --repository=k8s-manifests \
  --branch=main \
  --path=clusters/aks-prod-cc \   # Flux looks for manifests here
  --personal \                    # Use personal access token (or use OIDC below)
  --token-auth

# What this does:
#  - Creates GitHub repo if not exists
#  - Installs Flux components in kube-system
#  - Creates a GitRepository source pointing at the repo
#  - Creates a Kustomization targeting clusters/aks-prod-cc

# Step 3 — Verify Flux components are running
kubectl get pods -n flux-system
# Expected: source-controller, kustomize-controller, helm-controller, notification-controller

# Check reconciliation status
flux get all
# NAME                      READY  STATUS
# gitrepository/flux-system True   stored artifact for revision 'main/abc123'
# kustomization/flux-system True   Applied revision: main/abc123
```

**Directory structure in the Git repo:**
```
k8s-manifests/
  clusters/
    aks-prod-cc/
      flux-system/          ← Flux's own components (auto-managed)
      apps/
        production/
          payment-api.yaml  ← your application manifests
          kustomization.yaml
  infrastructure/
    monitoring/
      prometheus.yaml
```

> ⚠️ **Gotcha:** Flux bootstrap creates a `deploy key` on the GitHub repo (read-only by default). If you need Flux to push (e.g., image automation), the deploy key needs write access. Set `--read-write-key` in the bootstrap command or add a write deploy key manually.

> 💡 **Deep dive hint:** Flux image automation — `ImageRepository` + `ImagePolicy` + `ImageUpdateAutomation` objects let Flux automatically commit a new image tag back to Git when a new image is pushed to ACR, completing a full push-to-deploy loop without manual YAML edits.

---

## Q2. How do you deploy to AKS from GitHub Actions without storing Azure credentials?

**Scenario:** Your CI/CD pipeline uses a Service Principal secret stored in GitHub Secrets. Security flagged it — secrets can expire, leak, or be over-permissioned. You need secretless deployment.

```
GitHub Actions OIDC → Azure (no secrets):

  GitHub Actions job starts
       │ requests short-lived OIDC token from GitHub
       │ token claims: repo=org/repo, environment=production
       ▼
  Azure Entra ID
       │ validates token against federated credential:
       │   issuer: https://token.actions.githubusercontent.com
       │   subject: repo:org/repo:environment:production
       │ matches → issues short-lived Azure access token (~1h)
       ▼
  GitHub Actions uses token to run az aks get-credentials
       │ then kubectl apply / helm upgrade
       ▼
  AKS cluster ✅  (no password ever stored)
```

```yaml
# .github/workflows/deploy.yml
name: Deploy to AKS Production

on:
  push:
    branches: [main]

permissions:
  id-token: write    # Required for OIDC token request
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production    # Must match federated credential subject
    steps:
    - uses: actions/checkout@v4

    - name: Azure Login (OIDC — no secrets)
      uses: azure/login@v2
      with:
        client-id: ${{ vars.AZURE_CLIENT_ID }}          # Not a secret — it's public
        tenant-id: ${{ vars.AZURE_TENANT_ID }}
        subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}

    - name: Get AKS credentials
      run: |
        az aks get-credentials \
          --name aks-prod-cc \
          --resource-group rg-spoke-aks \
          --overwrite-existing

    - name: Deploy via Helm
      run: |
        helm upgrade payment-api ./helm/payment-api \
          --namespace production \
          --set image.tag=${{ github.sha }} \
          --wait --timeout 5m
```

```bash
# One-time setup: create federated credential linking GitHub → Azure MI
az identity federated-credential create \
  --identity-name id-github-deployer \
  --resource-group rg-spoke-aks \
  --name github-actions-production \
  --issuer "https://token.actions.githubusercontent.com" \
  --subject "repo:your-org/your-repo:environment:production" \
  --audience "api://AzureADTokenExchange"

# Grant the MI access to the AKS cluster
az role assignment create \
  --role "Azure Kubernetes Service Cluster User Role" \
  --assignee $MI_OBJECT_ID \
  --scope $(az aks show --name aks-prod-cc -g rg-spoke-aks --query id -o tsv)
```

> ⚠️ **Gotcha:** The `subject` in the federated credential must exactly match the GitHub Actions token claim. `repo:org/repo:environment:production` works only if the workflow uses `environment: production`. For branch-based deployments (no environment), use `repo:org/repo:ref:refs/heads/main`.

---

## Q3. How do you implement a blue-green deployment on AKS?

**Scenario:** You need to deploy a new version of `payment-api` but want zero downtime and a quick rollback path if the new version has issues. The team wants to validate the new version before cutting all traffic over.

```
Blue-Green on AKS:

  BEFORE:                          AFTER traffic cut:
  Service (payment-api-svc)        Service (payment-api-svc)
       │                                  │
       ▼                                  ▼
  Blue Deployment (v1.0) ✅        Green Deployment (v1.1) ✅
  3 pods — LIVE                    3 pods — LIVE

  Green Deployment (v1.1) ⬜        Blue Deployment (v1.0) ⬜
  3 pods — WARM (no traffic)        3 pods — STANDBY (30min)
                                    → delete after confidence

  Rollback: patch service selector back to blue — instant
```

```bash
# Step 1 — Deploy green (new version) alongside live blue
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-api-green
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment-api
      version: green
  template:
    metadata:
      labels:
        app: payment-api
        version: green
    spec:
      containers:
      - name: payment
        image: myacr.azurecr.io/payment:v1.1
EOF

# Step 2 — Verify green pods are healthy before cutting traffic
kubectl rollout status deployment/payment-api-green -n production
kubectl get pods -n production -l version=green
# All Running → proceed

# Step 3 — Patch service to point to green (instant traffic switch)
kubectl patch service payment-api-svc -n production \
  -p '{"spec": {"selector": {"version": "green"}}}'

# Step 4 — Monitor for 30 minutes, then delete blue if stable
kubectl delete deployment payment-api-blue -n production

# ROLLBACK (instant — just patch the selector back)
kubectl patch service payment-api-svc -n production \
  -p '{"spec": {"selector": {"version": "blue"}}}'
```

> ⚠️ **Gotcha:** Blue-green requires 2× compute during the transition window. With KEDA scale-to-zero, the warm standby (blue) still needs to hold its replicas — it doesn't scale to zero until explicitly deleted. Budget for double capacity during deployments.

---

## Q4. How do you implement canary deployment using NGINX Ingress on AKS?

**Scenario:** You want to send 10% of traffic to `payment-api v1.1` while 90% goes to `v1.0`, and monitor error rates before promoting to 100%.

```
Canary traffic split via NGINX Ingress annotations:

  Ingress (payment-api-canary)
  annotation: nginx.ingress.kubernetes.io/canary-weight: "10"
       │
       ├── 90% → payment-api-stable (v1.0)
       └── 10% → payment-api-canary (v1.1)

  Monitoring:
  - Error rate canary vs stable
  - P99 latency canary vs stable
  - After 24h: bump weight to 50%, then 100%
```

```yaml
# Stable deployment Service (existing)
# payment-api-stable-svc → payment-api v1.0 pods

# Canary Ingress — NGINX routes % of traffic here
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: payment-api-canary
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"   # 10% of requests
spec:
  ingressClassName: nginx
  rules:
  - host: payment.internal.company.com
    http:
      paths:
      - path: /api/payments
        pathType: Prefix
        backend:
          service:
            name: payment-api-canary-svc     # Points to v1.1 pods
            port:
              number: 8080
```

```bash
# Gradually increase canary weight as confidence builds
# Week 1: 10%
kubectl annotate ingress payment-api-canary -n production \
  nginx.ingress.kubernetes.io/canary-weight=10 --overwrite

# Week 2: 50%
kubectl annotate ingress payment-api-canary -n production \
  nginx.ingress.kubernetes.io/canary-weight=50 --overwrite

# Full promotion: delete canary ingress, update stable deployment to v1.1
kubectl delete ingress payment-api-canary -n production
kubectl set image deployment/payment-api-stable \
  payment=myacr.azurecr.io/payment:v1.1 -n production
```

> ⚠️ **Gotcha:** NGINX canary `canary-weight` is a percentage of all requests reaching the stable Ingress, not a percentage split between stable and canary independently. If you have multiple canary Ingress objects for the same host/path, the weights are summed — having two canaries at `50` each results in both receiving 50% each (100% canary, 0% stable), which is not what you want.
