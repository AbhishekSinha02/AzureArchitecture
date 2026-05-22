# AKS Storage & Persistent Volumes
### 🟡 Intermediate → 🔴 Advanced

> *"A pod is ephemeral. A PersistentVolume is a promise to a pod  
>  that its data will survive the pod's death."*

---

## Q1. What is the difference between Azure Disk and Azure Files for AKS, and when do you use each?

**Scenario:** You're deploying a PostgreSQL database and an NGINX file share for multiple app pods. The DBA asks for dedicated disk, the app team needs shared storage. Which CSI driver do you use for each?

```
Azure Disk (RWO)                    Azure Files (RWX)
─────────────────                   ────────────────────────────
One pod at a time                   Many pods simultaneously
Block storage (LUN)                 SMB/NFS file share
Zoned — disk + pod must match AZ    Zone-redundant or LRS
Best for: DBs, stateful apps        Best for: shared content, config
                                    NFS mounts, legacy apps

         Pod A ──── Disk (zone 1)            Pod A ─┐
                                             Pod B ──┼─── Azure Files Share
                                             Pod C ─┘
```

| Feature | Azure Disk | Azure Files |
|---------|-----------|-------------|
| Access mode | `ReadWriteOnce` (RWO) | `ReadWriteMany` (RWX) |
| Protocol | Block (iSCSI/NVMe) | SMB 3.0 / NFS 4.1 |
| Performance tiers | Standard HDD/SSD, Premium SSD, Ultra | Standard, Premium |
| Zone constraint | Disk must be in same AZ as node | Zone-redundant available |
| Use case | Databases, stateful singletons | Shared file access, media, logs |
| CSI driver | `disk.csi.azure.com` | `file.csi.azure.com` |
| Latency | ~1ms (Premium SSD v2) | ~5ms (Premium NFS) |

```yaml
# StorageClass for Azure Disk (Premium SSD — database use)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-premium-retain
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS
  kind: Managed
reclaimPolicy: Retain          # IMPORTANT: don't delete data when PVC deleted
volumeBindingMode: WaitForFirstConsumer  # Wait until pod scheduled to bind zone
allowVolumeExpansion: true
---
# StorageClass for Azure Files (NFS — shared access)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azurefile-nfs-premium
provisioner: file.csi.azure.com
parameters:
  protocol: nfs                # NFS 4.1 — no password needed, Linux-native
  skuName: Premium_LRS
reclaimPolicy: Retain
volumeBindingMode: Immediate
allowVolumeExpansion: true
```

> ⚠️ **Gotcha:** `volumeBindingMode: Immediate` on Azure Disk creates the disk in a random AZ before knowing which node the pod lands on. If the pod schedules to a different AZ, it fails to mount. Always use `WaitForFirstConsumer` for disk.

---

## Q2. How do you provision persistent storage for a PostgreSQL pod on AKS?

**Scenario:** You're deploying a PostgreSQL StatefulSet in AKS. It needs 100Gi of Premium SSD storage that persists across pod restarts and node replacements.

```
StatefulSet + PVC pattern:

  StatefulSet (postgres)
       │ volumeClaimTemplates: 100Gi Premium SSD
       ▼
  PersistentVolumeClaim (postgres-data-postgres-0)
       │ dynamically provisioned
       ▼
  PersistentVolume (Azure Disk — Premium_LRS, zone 1)
       │ disk ID: /subscriptions/.../disks/pvc-abc123
       ▼
  Pod postgres-0 mounts at /var/lib/postgresql/data
```

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:16
        env:
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata    # Subdirectory required for permissions
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            cpu: "1"
            memory: "2Gi"
          limits:
            cpu: "4"
            memory: "8Gi"
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: managed-premium-retain
      resources:
        requests:
          storage: 100Gi
```

```bash
# Check PVC was provisioned successfully
kubectl get pvc -n production
# Expected:
# NAME                    STATUS   VOLUME          CAPACITY   ACCESS MODES
# postgres-data-postgres-0  Bound  pvc-abc123def   100Gi      RWO

# Expand the disk online (no downtime — AKS CSI supports live expansion)
kubectl patch pvc postgres-data-postgres-0 -n production \
  -p '{"spec": {"resources": {"requests": {"storage": "200Gi"}}}}'

# Verify expansion
kubectl describe pvc postgres-data-postgres-0 -n production
# Look for: "Resizing volume" then "Volume resize successful"
```

> ⚠️ **Gotcha:** The `PGDATA` env var must point to a subdirectory (e.g., `/var/lib/postgresql/data/pgdata`), not directly to the mount point. The Azure Disk CSI driver creates a `lost+found` directory at the root of the disk — PostgreSQL refuses to start if its data directory is not empty.

> 💡 **Deep dive hint:** Azure Disk snapshot for backup — `VolumeSnapshot` objects trigger Azure Disk snapshots, and `VolumeSnapshotContent` allows restoring a PVC from snapshot. Integrate with Velero for full Kubernetes backup.

---

## Q3. How do you mount an Azure Key Vault secret as a file inside a pod?

**Scenario:** Your app reads its TLS private key from `/etc/ssl/app.key` at startup. Storing it as a Kubernetes Secret is not allowed by your security policy. You must read it directly from Key Vault without writing it to etcd.

```
CSI Secret Store Provider flow:

  Key Vault
  kv-prod / secret: tls-private-key
       │
       │ (Workload Identity token exchange)
       ▼
  CSI Secret Store Provider (runs as DaemonSet on each node)
       │ mounts secret as tmpfs volume — never written to disk
       ▼
  Pod at /mnt/secrets/tls-private-key  ← read by app at startup
       │
       │ (optional: sync to Kubernetes Secret for env var access)
       ▼
  Kubernetes Secret: app-tls-secret  ← auto-rotated when KV secret changes
```

```bash
# Step 1 — Install CSI Secret Store Provider for Azure
helm repo add csi-secrets-store-provider-azure \
  https://azure.github.io/secrets-store-csi-driver-provider-azure/charts
helm install csi-secrets-store \
  csi-secrets-store-provider-azure/csi-secrets-store-provider-azure \
  --namespace kube-system \
  --set secrets-store-csi-driver.syncSecret.enabled=true   # Sync to K8s Secrets
```

```yaml
# Step 2 — SecretProviderClass: defines what to pull from Key Vault
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: kv-tls-secrets
  namespace: production
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    clientID: "<MI_CLIENT_ID>"         # Workload Identity managed identity
    keyvaultName: kv-prod
    tenantId: "<TENANT_ID>"
    objects: |
      array:
        - |
          objectName: tls-private-key
          objectType: secret
          objectVersion: ""            # "" = always latest version
        - |
          objectName: db-connection-string
          objectType: secret
  secretObjects:                       # Also sync as K8s Secret
  - secretName: app-tls-secret
    type: Opaque
    data:
    - objectName: tls-private-key
      key: tls.key
---
# Step 3 — Pod mounts the CSI volume
spec:
  serviceAccountName: payment-service-sa   # Must have WI annotation
  volumes:
  - name: secrets-store
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: kv-tls-secrets
  containers:
  - name: app
    volumeMounts:
    - name: secrets-store
      mountPath: /mnt/secrets
      readOnly: true
```

> ⚠️ **Gotcha:** The CSI driver only fetches the secret when the pod starts (and periodically for rotation). If the Key Vault private endpoint is unreachable at pod startup, the volume mount fails and the pod goes into `Error` state with: `failed to mount secrets store objects`. The pod does NOT fall back to a cached copy.

> 💡 **Deep dive hint:** Secret rotation — the CSI provider polls Key Vault every `rotationPollInterval` (default 2min). When a secret rotates in Key Vault, the file on the pod is updated automatically — but the app must re-read the file (reload) to pick up the change.

---

## Q4. How do you solve the AZ affinity problem when using Azure Disk with StatefulSets?

**Scenario:** You have a StatefulSet with 3 replicas, each needing its own Azure Disk. The cluster spans 3 AZs. Sometimes pods get scheduled to a different AZ than their disk, causing mount failures.

```
The Zone Pinning Problem:

  postgres-0 PVC → Disk in Zone 1
  postgres-1 PVC → Disk in Zone 2
  postgres-2 PVC → Disk in Zone 3

  Node failure in Zone 2:
  → Kubernetes tries to reschedule postgres-1 to Zone 1 or Zone 3
  → BUT the disk is in Zone 2 (Azure Disks are zone-locked)
  → Mount fails: "Volume is not available in zone"

  ✅ Fix: Use topology-aware binding + node affinity
```

```yaml
# StorageClass with WaitForFirstConsumer — key fix
# This ensures the disk is created in the SAME zone as the scheduled node
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-premium-zonal
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_ZRS        # ZRS = zone-redundant storage (spans all zones)
  # Alternative to ZRS: use WaitForFirstConsumer + LRS (zonal)
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain
---
# For LRS disks: use pod topology spread to keep pods in correct AZ
# Add to StatefulSet pod spec:
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: postgres
```

```bash
# Check which zone a disk is in
az disk show \
  --name <disk-name> \
  --resource-group MC_rg-spoke-aks_aks-prod-cc_canadacentral \
  --query "zones"
# Expected: ["1"]  ← disk is zoned to AZ 1

# Check which zone a node is in
kubectl get node <node-name> \
  -o jsonpath='{.metadata.labels.topology\.kubernetes\.io/zone}'
# Expected: canadacentral-1

# Premium_ZRS avoids zone pinning entirely — disks accessible from any AZ
# Trade-off: ZRS costs ~35% more than LRS, ~2ms higher write latency
```

> ⚠️ **Gotcha:** `Premium_ZRS` (Zone-Redundant Storage for disks) is not available in all Azure regions. Check `az disk list-skus --location canadacentral` before specifying it. If ZRS is unavailable, use `WaitForFirstConsumer` with LRS and accept that a zone failure may make the pod unschedulable until the zone recovers.
