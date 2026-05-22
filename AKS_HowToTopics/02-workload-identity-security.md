# AKS Workload Identity & Security
### 🟡 Intermediate → 🔴 Advanced

> *"The moment a Kubernetes secret contains an Azure password,  
>  you've moved the problem, not solved it."*

---

## Q1. How do you give an AKS pod access to Azure Key Vault without storing any credentials?

**Scenario:** A payment microservice pod needs to read a database connection string from Key Vault at startup. The current approach stores an SP secret in a Kubernetes Secret — security wants it removed.

```
BEFORE (insecure — secret stored in K8s)        AFTER (Workload Identity)
────────────────────────────────────            ────────────────────────────────
Kubernetes Secret                               Pod SA token (OIDC)
  clientId: abc123                                    │
  clientSecret: s3cr3t  ◄── rotates,                 │ token exchange
                             can leak                  ▼
                                               Entra ID validates token
                                                    │
                                               Short-lived access token
                                                    │
                                               Key Vault (no password ever stored)
```

**Step-by-step setup:**

```bash
# Step 1 — Enable OIDC issuer + Workload Identity on cluster
az aks update \
  --name aks-prod-cc \
  --resource-group rg-spoke-aks \
  --enable-oidc-issuer \
  --enable-workload-identity

# Get OIDC issuer URL (needed for federated credential)
OIDC_URL=$(az aks show \
  --name aks-prod-cc -g rg-spoke-aks \
  --query oidcIssuerProfile.issuerUrl -o tsv)

# Step 2 — Create user-assigned managed identity
az identity create \
  --name id-payment-service \
  --resource-group rg-spoke-aks

MI_CLIENT_ID=$(az identity show \
  --name id-payment-service -g rg-spoke-aks --query clientId -o tsv)
MI_OBJECT_ID=$(az identity show \
  --name id-payment-service -g rg-spoke-aks --query principalId -o tsv)

# Step 3 — Grant identity access to Key Vault (least privilege)
az keyvault set-policy \
  --name kv-prod \
  --object-id $MI_OBJECT_ID \
  --secret-permissions get       # Only 'get' — not list, set, delete

# Step 4 — Create Kubernetes service account annotated with MI client ID
kubectl create serviceaccount payment-service-sa -n production

kubectl annotate serviceaccount payment-service-sa \
  -n production \
  azure.workload.identity/client-id=$MI_CLIENT_ID

# Step 5 — Create federated credential (links K8s SA to Azure MI)
az identity federated-credential create \
  --identity-name id-payment-service \
  --resource-group rg-spoke-aks \
  --name fed-cred-payment-service \
  --issuer $OIDC_URL \
  --subject "system:serviceaccount:production:payment-service-sa" \
  --audience api://AzureADTokenExchange
```

**Pod manifest using Workload Identity:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: production
spec:
  template:
    metadata:
      labels:
        azure.workload.identity/use: "true"    # Required label
    spec:
      serviceAccountName: payment-service-sa   # SA with the annotation
      containers:
      - name: payment
        image: myacr.azurecr.io/payment:1.0
        # No env vars with secrets — SDK reads the projected token automatically
        # Azure SDK: new DefaultAzureCredential() works without configuration
```

> ⚠️ **Gotcha:** The `azure.workload.identity/use: "true"` label on the pod is mandatory. Without it, the mutating webhook does not inject the OIDC token volume — the pod starts but `DefaultAzureCredential` fails silently with a 401.

> 💡 **Deep dive hint:** CSI Secret Store Provider for AKS — mounting Key Vault secrets as files or environment variables inside pods, with automatic rotation when the secret changes in Key Vault.

---

## Q2. How do you enforce Pod Security Standards across all namespaces?

**Scenario:** A developer accidentally deployed a pod running as root with `hostPath` volume mounts in production. Security wants guardrails that prevent this cluster-wide.

```
Pod Security Standards — three levels:
┌────────────────┬───────────────────────────────────────────────┐
│ Level          │ What it blocks                                │
├────────────────┼───────────────────────────────────────────────┤
│ privileged     │ Nothing — unrestricted (dev/test only)        │
│ baseline       │ Blocks hostNetwork, hostPID, privileged mode  │
│ restricted     │ + blocks root user, requires securityContext  │
└────────────────┴───────────────────────────────────────────────┘

Modes per namespace:
  enforce  → reject the pod immediately (admission error)
  audit    → allow but log to audit log
  warn     → allow but show kubectl warning to developer
```

**Apply restricted policy to production namespace:**
```bash
# Label the namespace — PSA webhook enforces automatically
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted
```

**Pod that FAILS restricted policy (and why):**
```yaml
spec:
  containers:
  - name: app
    image: myapp:1.0
    securityContext:
      runAsRoot: true          # ❌ BLOCKED — restricted disallows root
      privileged: true         # ❌ BLOCKED — restricted disallows privileged
```

**Pod that PASSES restricted policy:**
```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault     # Required by restricted
  containers:
  - name: app
    image: myapp:1.0
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]          # Drop all Linux capabilities
      readOnlyRootFilesystem: true
```

> ⚠️ **Gotcha:** Applying `enforce=restricted` to the `kube-system` namespace breaks AKS system pods — they run as root. Always label your own namespaces only, never `kube-system`, `kube-public`, or the Azure add-on namespaces.

---

## Q3. How do you restrict which container images pods can run from?

**Scenario:** A pod was deployed with `image: alpine` pulled from Docker Hub. Your company policy requires all images to come only from your ACR instance. How do you enforce this?

```
BEFORE (any registry allowed)              AFTER (ACR only)
─────────────────────────────              ────────────────
kubectl apply → pod created                kubectl apply → BLOCKED by policy
image: alpine (Docker Hub)                 image: alpine
                                           Error: image not from myacr.azurecr.io
                                           
                                           kubectl apply → ALLOWED
                                           image: myacr.azurecr.io/alpine:3.18 ✅
```

**Method 1: Azure Policy add-on (cluster-level enforcement):**
```bash
# Enable Azure Policy add-on
az aks enable-addons \
  --addons azure-policy \
  --name aks-prod-cc \
  --resource-group rg-spoke-aks

# Assign built-in policy: "Kubernetes cluster containers should only use allowed images"
az policy assignment create \
  --name "restrict-aks-images" \
  --policy "febd0533-8e55-448f-b798-9e8afe5d06d8" \  # Built-in policy ID
  --scope "/subscriptions/<SUB>/resourceGroups/rg-spoke-aks" \
  --params '{"allowedContainerImagesRegex": {"value": "^myacr\\.azurecr\\.io/.+$"}}'
```

**Method 2: OPA Gatekeeper (more flexible):**
```yaml
# ConstraintTemplate — defines the rule logic
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: allowedregistries
spec:
  crd:
    spec:
      names:
        kind: AllowedRegistries
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package allowedregistries
      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        not startswith(container.image, "myacr.azurecr.io/")
        msg := sprintf("Image '%v' not from allowed registry", [container.image])
      }
---
# Constraint — applies the template
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: AllowedRegistries
metadata:
  name: prod-acr-only
spec:
  match:
    namespaces: ["production", "staging"]
  parameters: {}
```

> ⚠️ **Gotcha:** Azure Policy add-on uses the `audit` effect initially — pods that violate policy are created but flagged. Change to `deny` effect in the assignment to block at admission. Many teams run audit for a sprint first to catch existing violations before switching to deny.

---

## Q4. How do you scan container images for vulnerabilities before they reach production?

**Scenario:** A CVE is discovered in a base image used by 8 of your microservices. You need to know which ACR images are affected and block new images with critical CVEs from deploying.

```
CI/CD Pipeline with image scanning gate:

  Code push
      │
      ▼
  Build image
      │
      ▼
  Push to ACR (non-prod tag: myacr.azurecr.io/app:pr-123)
      │
      ▼
  Microsoft Defender for Containers scans image
      │
      ├─── Critical CVEs found? ──► Block pipeline / fail PR check
      │
      └─── Clean / Low/Medium only ──► Promote tag to :latest / :prod-1.0
                                             │
                                             ▼
                                       AKS pulls from ACR ✅
```

**Enable Defender for Containers (scans all ACR images):**
```bash
az security pricing create \
  --name Containers \
  --tier Standard        # Enables Defender for Containers = ACR image scanning
```

**Query scan results in Log Analytics:**
```kusto
ContainerRegistryRepositoryEvents
| where OperationName == "Push"
| join kind=leftouter (
    SecurityRecommendation
    | where RecommendationName contains "vulnerabilities"
    | extend ImageName = tostring(ResourceDetails["imageDigest"])
  ) on $left.ImageDigest == $right.ImageName
| project TimeGenerated, Repository, Tag, CriticalSeverityCount, HighSeverityCount
| where CriticalSeverityCount > 0
| order by TimeGenerated desc
```

**Block deployment via admission webhook (Ratify + Gatekeeper):**
```bash
# Ratify verifies image signatures and vulnerability attestations at admission time
helm install ratify ratify/ratify \
  --namespace gatekeeper-system \
  --set featureFlags.RATIFY_CERT_ROTATION=true \
  --set azureWorkloadIdentity.clientId=$MI_CLIENT_ID
```

> 💡 **Deep dive hint:** ACR image signing with Notation + Azure Key Vault — how to sign images at build time and verify signatures at admission using Ratify, so only signed images from your pipeline can run in production.
