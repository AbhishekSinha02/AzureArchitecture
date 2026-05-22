# AKS Cluster Setup
### 🟢 Beginner → 🟡 Intermediate

> *"An AKS cluster is not a VM you SSH into.  
>  It is a contract between your workloads and Azure's managed control plane."*

---

## Q1. What are the components of an AKS cluster and who manages what?

**Scenario:** Your team is new to AKS. Before writing any `az aks create` command, you need to understand what Azure manages vs what you own.

```
AZURE MANAGES (free, SLA-backed with paid tier)
┌─────────────────────────────────────────────────┐
│              CONTROL PLANE                      │
│  ┌──────────┐ ┌──────────┐ ┌─────────────────┐ │
│  │ API      │ │ etcd     │ │ Controller Mgr  │ │
│  │ Server   │ │ (state)  │ │ Scheduler       │ │
│  └──────────┘ └──────────┘ └─────────────────┘ │
└─────────────────────────────────────────────────┘
                      │ kubelet talks up to API server
YOU MANAGE (billed as VMs in your subscription)
┌─────────────────────────────────────────────────┐
│                DATA PLANE                       │
│  ┌──────────────┐     ┌──────────────┐          │
│  │ System Pool  │     │ User Pool(s) │          │
│  │ (AKS system  │     │ (your apps)  │          │
│  │  pods only)  │     │              │          │
│  └──────────────┘     └──────────────┘          │
└─────────────────────────────────────────────────┘
```

**What Azure manages:**
- ✅ **API server** — Kubernetes control plane endpoint (`<cluster>.hcp.<region>.azmk8s.io`)
- ✅ **etcd** — cluster state database, backed up by Azure
- ✅ **Control plane OS patching** — automatic
- ✅ **Control plane scaling** — automatic based on cluster size

**What you manage:**
- 🔑 **Node pools** — VMSS instances, OS patching, node image upgrades
- 🔑 **Workloads** — Deployments, Services, Ingress, ConfigMaps
- 🔑 **Networking** — VNet, subnets, NSGs, Private DNS Zones
- 🔑 **Identity** — Managed Identity assignments, Workload Identity federation
- 🔑 **Add-ons** — KEDA, Flux, Azure Policy, NGINX Ingress

**SLA tiers:**
| Tier | SLA | Cost |
|------|-----|------|
| Free | No SLA | Free |
| Standard | 99.9% uptime on API server | ~$73/month |
| Premium | 99.95% + LTS Kubernetes versions | ~$146/month |

> ⚠️ **Gotcha:** Free tier has no SLA — the API server can go offline during Azure maintenance. Always use Standard or Premium for production.

> 💡 **Deep dive hint:** AKS node image upgrade vs cluster upgrade — two separate upgrade channels and why you should automate node image upgrades independently of Kubernetes version upgrades.

---

## Q2. How do you create a production-ready AKS cluster from scratch?

**Scenario:** You're provisioning a new AKS cluster for a financial services app in Canada Central. It needs a private API server, system/user node pool separation, managed identity, and integration with an existing VNet.

```
Existing Hub VNet (10.0.0.0/16)
         │
         │ Peering
         ▼
Spoke VNet: vnet-spoke-aks (10.1.0.0/16)
  ├── snet-aks-system   10.1.1.0/24   ← system node pool
  ├── snet-aks-user     10.1.2.0/22   ← user node pool (needs large CIDR for pods)
  └── snet-aks-ingress  10.1.6.0/24   ← internal ingress LB
```

```bash
# Step 1 — Variables
RG="rg-spoke-aks"
CLUSTER="aks-prod-cc"
LOCATION="canadacentral"
VNET="vnet-spoke-aks"
SYSTEM_SUBNET="snet-aks-system"
USER_SUBNET="snet-aks-user"

SYSTEM_SUBNET_ID=$(az network vnet subnet show \
  --name $SYSTEM_SUBNET --vnet-name $VNET -g $RG --query id -o tsv)

USER_SUBNET_ID=$(az network vnet subnet show \
  --name $USER_SUBNET --vnet-name $VNET -g $RG --query id -o tsv)

# Step 2 — Create cluster
az aks create \
  --name $CLUSTER \
  --resource-group $RG \
  --location $LOCATION \
  --kubernetes-version 1.29 \
  --tier Standard \                         # SLA-backed control plane
  --enable-private-cluster \                # API server private — no public endpoint
  --network-plugin azure \                  # Azure CNI — routable pod IPs
  --vnet-subnet-id $SYSTEM_SUBNET_ID \
  --service-cidr 172.16.0.0/16 \            # Must not overlap VNet or on-prem
  --dns-service-ip 172.16.0.10 \
  --enable-managed-identity \               # System-assigned MI for control plane
  --enable-workload-identity \              # OIDC issuer for pod identity
  --enable-oidc-issuer \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 10 \
  --node-vm-size Standard_D4s_v3 \
  --node-count 3 \
  --zones 1 2 3 \                           # Spread nodes across all AZs
  --nodepool-name systempool \
  --nodepool-taints CriticalAddonsOnly=true:NoSchedule \  # Reserve for system pods
  --os-sku AzureLinux \                     # Lightweight, Microsoft-maintained OS
  --generate-ssh-keys \
  --no-wait

# Step 3 — Add user node pool (workloads go here, not system pool)
az aks nodepool add \
  --cluster-name $CLUSTER \
  --resource-group $RG \
  --name userpool \
  --vnet-subnet-id $USER_SUBNET_ID \
  --node-vm-size Standard_D8s_v3 \
  --node-count 3 \
  --min-count 3 \
  --max-count 50 \
  --enable-cluster-autoscaler \
  --zones 1 2 3 \
  --mode User \
  --os-sku AzureLinux
```

**Key flags explained:**
- `--enable-private-cluster` → API server gets a private IP only — accessible from VNet/peered VNets, not the internet
- `--network-plugin azure` → each pod gets a real VNet IP (Azure CNI) — required for Private Endpoints, NSG targeting
- `--nodepool-taints CriticalAddonsOnly=true:NoSchedule` → prevents your app pods from landing on system nodes
- `--zones 1 2 3` → nodes spread across three Availability Zones — survives one zone outage

> ⚠️ **Gotcha:** Private cluster API server requires a Private DNS Zone (`privatelink.<region>.azmk8s.io`) linked to every VNet that needs `kubectl` access — including your DevOps agent's VNet. Forget this and `kubectl` times out silently.

> 💡 **Deep dive hint:** AKS private cluster with custom DNS — when you run your own DNS servers (not Azure DNS), the private DNS zone auto-link fails and you must manually configure DNS forwarding for the private API FQDN.

---

## Q3. What is the difference between Azure CNI and Kubenet and which should you use?

**Scenario:** You're designing an AKS cluster for a bank. The network team asks which CNI plugin to use and why it matters for their NSG and Private Endpoint policies.

```
KUBENET (basic)                         AZURE CNI (routable)
───────────────                         ────────────────────
Node IP:  10.1.0.4   (VNet IP)          Node IP:  10.1.0.4   (VNet IP)
Pod IPs:  10.244.x.x (overlay, NAT'd)   Pod IPs:  10.1.2.x   (VNet IPs)
                │                                       │
         NAT at node NIC                    No NAT — pods are VNet citizens
                │
Pod → Node NAT → VNet → destination     Pod → VNet → destination (direct)
```

| Feature | Kubenet | Azure CNI |
|---------|---------|-----------|
| Pod IPs | Overlay (not routable) | Real VNet IPs |
| Subnet size needed | Small (nodes only) | Large (nodes + max pods × nodes) |
| NSG rules on pods | Not possible (NAT hides pod IP) | Possible (pod IP is known) |
| Private Endpoint from pod | Works (via NAT) | Works (direct) |
| Network Policy | Calico only | Azure NPM or Calico |
| On-prem can reach pods | No (overlay) | Yes (routable) |
| AKS recommended for prod | ❌ | ✅ |

**Azure CNI subnet sizing — the most common mistake:**
```bash
# Azure CNI allocates: max_pods_per_node × node_count IPs from the subnet
# Default max_pods = 30 per node

# Example: 50 nodes × 30 pods = 1,500 pod IPs needed
# Plus node IPs: 50
# Total: ~1,550 IPs → need at minimum a /21 (2,046 usable IPs)

# Safe formula: (max_nodes × max_pods_per_node) + max_nodes + 5 (buffer)
# For 50-node cluster with 30 pods: (50×30) + 50 + 5 = 1,555 → use /21
```

**Azure CNI Overlay (newer, recommended for large clusters):**
```bash
# Best of both worlds: pod IPs are overlay but nodes get VNet IPs
# Pods still appear routable within the cluster
# Subnet only needs IPs for nodes — not for every pod
az aks create \
  --network-plugin azure \
  --network-plugin-mode overlay \    # Azure CNI Overlay
  --pod-cidr 10.244.0.0/16           # Overlay range — doesn't consume VNet IPs
```

> ⚠️ **Gotcha:** You cannot change the CNI plugin after cluster creation. Switching from Kubenet to Azure CNI requires a new cluster. Plan this upfront.

---

## Q4. How do you configure two managed identities for AKS correctly?

**Scenario:** You set up AKS with managed identity but the cluster can't pull images from ACR and the Kubelet can't read Key Vault secrets. You hear "there are two managed identities" — what does that mean?

```
AKS CLUSTER — TWO IDENTITIES, TWO JOBS
┌──────────────────────────────────────────────────────┐
│                                                      │
│  Identity 1: CONTROL PLANE MI                        │
│  ─────────────────────────────                       │
│  Created: automatically with cluster                 │
│  Used for: ARM operations                            │
│  Needs:  - Contributor on MC_ resource group         │
│          - Network Contributor on VNet subnet         │
│          - Managed Identity Operator (for kubelet MI) │
│                                                      │
│  Identity 2: KUBELET MI (node identity)              │
│  ──────────────────────────────────────              │
│  Created: automatically (or bring your own)          │
│  Used for: every operation FROM the nodes            │
│  Needs:  - AcrPull on Azure Container Registry       │
│          - Key Vault Secrets User (if pods read KV)  │
│          - Storage Blob Data Reader (if pods read SA) │
└──────────────────────────────────────────────────────┘
```

```bash
# Get both identity IDs after cluster creation
az aks show \
  --name aks-prod-cc \
  --resource-group rg-spoke-aks \
  --query "{ControlPlaneMI:identity.principalId, KubeletMI:identityProfile.kubeletidentity.clientId}"

# Grant Kubelet MI permission to pull from ACR
ACR_ID=$(az acr show --name myacr --resource-group rg-acr --query id -o tsv)
KUBELET_MI=$(az aks show \
  --name aks-prod-cc -g rg-spoke-aks \
  --query identityProfile.kubeletidentity.objectId -o tsv)

az role assignment create \
  --role AcrPull \
  --assignee $KUBELET_MI \
  --scope $ACR_ID

# Grant Kubelet MI access to Key Vault (for CSI Secret Store)
az keyvault set-policy \
  --name kv-prod \
  --object-id $KUBELET_MI \
  --secret-permissions get list
```

> ⚠️ **Gotcha:** A common error — granting `AcrPull` on the Control Plane MI instead of the Kubelet MI. The Control Plane MI provisions infrastructure; the Kubelet MI runs on nodes and pulls images. They are different objects with different object IDs.

> 💡 **Deep dive hint:** Bring Your Own (BYO) Kubelet identity — why you'd pre-create a user-assigned MI and pass it at cluster creation time instead of letting AKS auto-create one, and the RBAC implications.

---

## Q5. How do you upgrade an AKS cluster safely without downtime?

**Scenario:** Your cluster is on Kubernetes 1.28. You need to upgrade to 1.29. You have stateful workloads and can't afford more than 1 pod unavailable at a time per Deployment.

```
UPGRADE ORDER (always control plane first, then node pools)

Step 1: Upgrade Control Plane
  1.28 → 1.29  (can skip patch versions, not minor versions)
  API server updated — workloads keep running on 1.28 nodes
  ↓ (nodes stay on 1.28 during this step)

Step 2: Upgrade Node Pools (one at a time)
  System pool: cordon → drain → replace node → uncordon
  User pool:   cordon → drain → replace node → uncordon
  ↓ (surge node added before old node drained — no downtime)

Step 3: Verify
  kubectl get nodes  → all nodes on 1.29
  kubectl get pods   → all pods Running
```

```bash
# Step 1 — Check available versions
az aks get-upgrades \
  --name aks-prod-cc \
  --resource-group rg-spoke-aks \
  --output table

# Step 2 — Upgrade control plane only
az aks upgrade \
  --name aks-prod-cc \
  --resource-group rg-spoke-aks \
  --kubernetes-version 1.29 \
  --control-plane-only \             # Upgrade CP first, leave nodes on 1.28
  --no-wait

# Step 3 — Upgrade node pools individually
az aks nodepool upgrade \
  --cluster-name aks-prod-cc \
  --resource-group rg-spoke-aks \
  --name userpool \
  --kubernetes-version 1.29 \
  --max-surge 33%                    # Add 1/3 extra nodes during upgrade (faster, safe)

# Monitor progress
az aks nodepool show \
  --cluster-name aks-prod-cc -g rg-spoke-aks --name userpool \
  --query "{Version:orchestratorVersion, State:provisioningState}"
```

**PodDisruptionBudget — protect stateful workloads during drain:**
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: pdb-payment-api
spec:
  minAvailable: 2          # Always keep at least 2 pods up during node drain
  selector:
    matchLabels:
      app: payment-api
```

> ⚠️ **Gotcha:** You cannot skip Kubernetes minor versions. Going from 1.27 to 1.29 directly is blocked — you must go 1.27 → 1.28 → 1.29. Check the upgrade path before scheduling a maintenance window.
