# AKS Networking & Ingress
### 🟡 Intermediate → 🔴 Advanced

> *"A Kubernetes Service is not a network address.  
>  It's a promise that something will be reachable at that address — eventually."*

---

## Q1. How do you expose an AKS application internally (not to the internet)?

**Scenario:** A payment API runs in AKS. It must be reachable from other Azure services and on-premises, but must never have a public IP. How do you expose it?

```
Internet                    Azure (Canada Central)
────────                    ─────────────────────────────────────────
  ✗                         Hub VNet ──── Spoke VNet (AKS)
  (blocked)                              │
                                    Internal Load Balancer
                                    Private IP: 10.1.6.10
                                         │
                                    NGINX Ingress Controller
                                         │
                              ┌──────────┴──────────┐
                         payment-api svc        orders-api svc
                         10.1.2.x pods           10.1.2.x pods
```

**Step 1 — Deploy NGINX Ingress Controller with internal annotation:**
```bash
# values.yaml for NGINX with internal LB
cat > nginx-values.yaml <<EOF
controller:
  service:
    annotations:
      service.beta.kubernetes.io/azure-load-balancer-internal: "true"
      service.beta.kubernetes.io/azure-load-balancer-internal-subnet: "snet-aks-ingress"
    loadBalancerIP: "10.1.6.10"   # Static private IP in ingress subnet
  replicaCount: 2
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - topologyKey: kubernetes.io/hostname   # One replica per node
EOF

helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --values nginx-values.yaml
```

**Step 2 — Ingress resource routing by hostname/path:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: payment-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - payment.internal.company.com
    secretName: payment-tls-secret
  rules:
  - host: payment.internal.company.com
    http:
      paths:
      - path: /api/payments
        pathType: Prefix
        backend:
          service:
            name: payment-api-svc
            port:
              number: 8080
```

> ⚠️ **Gotcha:** The annotation `azure-load-balancer-internal-subnet` must name a subnet that exists in the **same VNet** as the AKS node pool. If AKS uses a different VNet than where you want the LB subnet, the annotation silently falls back to the node subnet.

> 💡 **Deep dive hint:** Application Gateway Ingress Controller (AGIC) — when to use Azure Application Gateway instead of NGINX, WAF integration, and the AKS add-on vs Helm deployment difference.

---

## Q2. How does Kubernetes Network Policy work and how do you implement default-deny?

**Scenario:** Pods in the `production` namespace are currently able to communicate with pods in the `dev` namespace. Security requires strict isolation — production pods must only accept traffic from within `production` and from the ingress controller.

```
BEFORE (no NetworkPolicy — all pods can talk to all pods):
dev/pod-a ──────────────────────────────────► production/payment-api  ✗ allowed!

AFTER (NetworkPolicy — default deny + specific allow):
dev/pod-a ──────────────────────────────────► production/payment-api  ✗ BLOCKED
ingress-nginx/controller ───────────────────► production/payment-api  ✅ ALLOWED
production/order-service ───────────────────► production/payment-api  ✅ ALLOWED
```

**Step 1 — Default deny all ingress and egress for production namespace:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}          # Applies to ALL pods in the namespace
  policyTypes:
  - Ingress
  - Egress
  # No ingress/egress rules = deny everything
```

**Step 2 — Allow ingress from NGINX and same namespace:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-controller
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: payment-api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx   # From ingress namespace
    - podSelector: {}                                   # From same namespace
    ports:
    - protocol: TCP
      port: 8080
```

**Step 3 — Allow egress to Key Vault private endpoint and DNS:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-keyvault
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: payment-api
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 10.1.0.22/32    # Key Vault private endpoint IP
    ports:
    - protocol: TCP
      port: 443
  - to:                        # Allow DNS (required for any DNS resolution)
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

> ⚠️ **Gotcha:** Always include a DNS egress rule when using default-deny. Without UDP/TCP port 53, pods can't resolve any hostname — including `kv-prod.vault.azure.net`. The pod crashes with a DNS resolution error that looks like a connectivity issue.

---

## Q3. How do you configure AKS to use Azure CNI with proper subnet sizing for 100 nodes?

**Scenario:** You're building an AKS cluster that will grow to 100 nodes, each running up to 30 pods. Your network team asks you what CIDR they need to allocate.

```
Azure CNI IP allocation:
  Every pod gets a real IP from the subnet (no overlay)

Calculation:
  100 nodes × 30 pods/node  = 3,000 pod IPs
  + 100 node IPs
  + 5 Azure reserved per subnet
  ─────────────────────────────
  Total needed: ~3,105 IPs

  /21 = 2,046 usable IPs  ← TOO SMALL
  /20 = 4,094 usable IPs  ← CORRECT ✅
  /19 = 8,190 usable IPs  ← Safe with room to grow
```

```bash
# Create correctly-sized subnet before cluster creation
az network vnet subnet create \
  --name snet-aks-user \
  --vnet-name vnet-spoke-aks \
  --resource-group rg-spoke-aks \
  --address-prefix 10.1.0.0/20      # 4,094 usable IPs — fits 100 nodes × 30 pods

# Create cluster with explicit max-pods setting
az aks create \
  --name aks-prod-cc \
  --resource-group rg-spoke-aks \
  --network-plugin azure \
  --vnet-subnet-id $SUBNET_ID \
  --max-pods 30 \                   # Controls per-node pod density
  --node-count 10 \
  --min-count 10 \
  --max-count 100                   # Cluster Autoscaler maximum
```

**Azure CNI Overlay — alternative for large clusters (no subnet exhaustion):**
```bash
az aks create \
  --network-plugin azure \
  --network-plugin-mode overlay \   # Pods use overlay IPs, not VNet IPs
  --pod-cidr 192.168.0.0/16 \       # Overlay range — doesn't consume VNet subnet IPs
  --vnet-subnet-id $SUBNET_ID       # Subnet only needs IPs for nodes
# Benefit: subnet only needs ~100 IPs for 100 nodes, not 3,100
```

> ⚠️ **Gotcha:** If you create the cluster with a subnet that's too small, `kubectl get nodes` shows nodes in `NotReady` state and the error is `failed to allocate IP address` in the IPAM controller logs — not an obvious message for a subnet sizing problem.

---

## Q4. How do you configure CoreDNS in AKS for custom DNS resolution?

**Scenario:** Your AKS pods need to resolve hostnames from your on-premises domain (`corp.internal`) which is served by your corporate DNS servers at `192.168.0.10`. Azure's DNS doesn't know about these names.

```
AKS Pod
  │ DNS query: sqlserver.corp.internal
  ▼
CoreDNS (in-cluster DNS server — 172.16.0.10)
  │ Check ConfigMap: is there a stub zone for corp.internal?
  │ YES → forward to 192.168.0.10
  ▼
Corporate DNS (192.168.0.10)
  │ Returns: 192.168.1.50
  ▼
Pod connects to 192.168.1.50 (via VPN/ExpressRoute)
```

```yaml
# Custom CoreDNS ConfigMap (AKS-specific — use azure-coredns-custom)
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: kube-system
data:
  corp-internal.server: |
    corp.internal:53 {
        errors
        cache 30
        forward . 192.168.0.10 192.168.0.11   # On-prem DNS servers (primary + secondary)
        }
  azure-blob.override: |
    blob.core.windows.net:53 {
        errors
        cache 30
        forward . 168.63.129.16                # Azure internal DNS — resolves PE hostnames
    }
```

```bash
# Apply and restart CoreDNS to pick up changes
kubectl apply -f coredns-custom.yaml

kubectl rollout restart deployment/coredns -n kube-system

# Verify from a pod
kubectl run dns-test --image=busybox:1.28 --rm -it -- \
  nslookup sqlserver.corp.internal
# Expected: returns 192.168.1.50
```

> ⚠️ **Gotcha:** Use the ConfigMap name `coredns-custom` (not `coredns`) in `kube-system`. AKS CoreDNS is configured to look for this specific ConfigMap name for custom overrides — editing the main `coredns` ConfigMap is wiped on cluster upgrades.

> 💡 **Deep dive hint:** CoreDNS health monitoring — how to check `kubectl logs -n kube-system -l k8s-app=kube-dns` for DNS query failures, and how to increase CoreDNS replicas when DNS becomes a cluster-wide bottleneck.
