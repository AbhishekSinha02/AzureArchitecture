# AKS How-To — Scenario Library

> *"Kubernetes is infrastructure as code. AKS is infrastructure as a managed opinion.  
>  The opinion is good — until the day you need to override it."*

### 🟢 Beginner → 🟡 Intermediate → 🔴 Advanced

---

## Files

| # | File | What's inside |
|---|------|---------------|
| 01 | [cluster-setup.md](01-cluster-setup.md) | Cluster creation, node pools, networking modes, identity |
| 02 | [workload-identity-security.md](02-workload-identity-security.md) | Workload Identity, pod security, RBAC, image scanning |
| 03 | [networking-ingress.md](03-networking-ingress.md) | Azure CNI, Ingress, Internal LB, Network Policy, DNS |
| 04 | [scaling-keda.md](04-scaling-keda.md) | HPA, KEDA, Cluster Autoscaler, node pool design |
| 05 | [storage-persistent-volumes.md](05-storage-persistent-volumes.md) | PVC, Azure Disk, Azure Files, CSI drivers, zone-pinning |
| 06 | [monitoring-observability.md](06-monitoring-observability.md) | Container Insights, Prometheus, Grafana, KQL alerts |
| 07 | [gitops-cicd.md](07-gitops-cicd.md) | Flux v2, ArgoCD, GitHub Actions OIDC, image automation |
| 08 | [high-availability-dr.md](08-high-availability-dr.md) | AZs, PDB, topology spread, node surge, chaos testing |
| 09 | [troubleshooting-playbook.md](09-troubleshooting-playbook.md) | Pod crashes, networking, auth, OOMKill, CrashLoopBackOff |
| 10 | [commands-reference.md](10-commands-reference.md) | Every kubectl / az aks / helm command with context |

---

## How to Read

Every answer follows the validated pattern:
1. **Scenario** — the real situation you're in
2. **Diagram** — ASCII topology or flow
3. **Answer** — bullet points, **services bolded**
4. **Commands** — runnable, annotated with expected output
5. **⚠️ Gotcha** — the one thing that breaks this in production
6. **💡 Deep dive hint** — follow-up thread topic
