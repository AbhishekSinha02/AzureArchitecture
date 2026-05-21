# Azure Security & Zero Trust — Interview Questions

> Covers: Zero Trust principles, Entra ID, RBAC, Private Endpoints, Defender, Key Vault, compliance (OSFI/PIPEDA).
> Levels: Beginner → Intermediate → Advanced

---

## BEGINNER

**Q1. What is Zero Trust and what are its core principles?**

**Answer:**

Zero Trust is a security model based on the principle: **"Never trust, always verify."** It rejects the old perimeter-based model (trust everything inside the network) and assumes breach.

**Three core principles:**

| Principle | Meaning | Azure implementation |
|-----------|---------|---------------------|
| **Verify explicitly** | Authenticate and authorize every request using all available signals | Conditional Access, MFA, device compliance |
| **Least privilege access** | Only the minimum access needed | RBAC, PIM, Just-in-Time access |
| **Assume breach** | Design assuming attackers are already inside | Network segmentation, encryption, audit logs |

**Azure Zero Trust pillars:**
- **Identity**: Entra ID, MFA, Conditional Access
- **Devices**: Intune, Microsoft Defender for Endpoint, device compliance
- **Network**: Private Endpoints, NSGs, Azure Firewall, micro-segmentation
- **Applications**: APIM, App Service auth, mTLS between services
- **Data**: Encryption at rest + in transit, data classification, Purview
- **Infrastructure**: Defender for Cloud, Azure Policy, privileged access workstations

---

**Q2. What is Azure Role-Based Access Control (RBAC) and what are the built-in roles?**

**Answer:**

Azure RBAC controls who can do what to which Azure resources. It uses:
- **Security principal**: User, group, service principal, or managed identity
- **Role definition**: Set of allowed actions (e.g., read, write, delete)
- **Scope**: Management group, subscription, resource group, or resource

**Key built-in roles:**

| Role | Description |
|------|-------------|
| **Owner** | Full access + can grant access to others |
| **Contributor** | Create/manage all resources, can't grant access |
| **Reader** | View resources only |
| **User Access Administrator** | Manage access assignments only |
| **Storage Blob Data Reader** | Read blobs (not storage account management) |
| **Storage Blob Data Contributor** | Read + write + delete blobs |
| **Key Vault Secrets User** | Read secrets |
| **AcrPull** | Pull images from Azure Container Registry |

**Important distinction:** Classic "admin" roles (Owner, Contributor, Reader) control the Azure *management plane*. Data-plane roles (Storage Blob Data Reader, Key Vault Secrets User) control access to *data*. An Owner cannot read Key Vault secrets unless they also have a Key Vault RBAC role.

---

**Q3. What is Conditional Access in Azure Entra ID?**

**Answer:**

Conditional Access is a policy engine that evaluates signals and enforces controls before granting access. Think of it as: **If [conditions], Then [enforce controls].**

**Common policy examples:**

```
Policy 1: MFA for admin roles
  IF user has Global Admin, Privileged Role Admin, or Security Admin role
  THEN require MFA

Policy 2: Block risky sign-ins
  IF sign-in risk = HIGH (leaked credential, impossible travel)
  THEN block access

Policy 3: Compliant device required for email
  IF accessing Exchange Online
  THEN require Intune-compliant device

Policy 4: Location-based access
  IF sign-in from non-Canada location
  AND accessing banking application
  THEN require MFA + block if sign-in risk > Medium

Policy 5: Session controls for sensitive apps
  IF accessing finance application
  THEN limit download of sensitive files (MCAS integration)
```

**Banking context:** Conditional Access is a key OSFI B-10 control for user access management. It replaces static IP allowlists with dynamic, risk-based access decisions.

---

## INTERMEDIATE

**Q4. Explain Private Endpoints vs Service Endpoints. When would you use each?**

**Answer:**

| Feature | Service Endpoint | Private Endpoint |
|---------|-----------------|-----------------|
| **Traffic path** | Azure backbone (not public internet) | Directly into your VNet via private IP |
| **Endpoint IP** | Service retains public IP | Service gets a private IP in your subnet |
| **DNS** | No change needed | Requires Private DNS Zone update |
| **Firewall bypass** | Enabled via service endpoint firewall | No public access needed at all |
| **On-prem access** | No (VPN/ER doesn't benefit) | Yes (traffic travels through VNet to on-prem) |
| **Cost** | Free | ~$7/month per endpoint + data processing |
| **Security** | Better than public internet | Strongest: no public IP at all |

**Private Endpoint is preferred for:**
- Production workloads where zero public exposure is required
- On-premises access to Azure services via ExpressRoute/VPN
- Compliance: OSFI, PCI-DSS, PIPEDA often require private-only access to data services

**Service Endpoint is acceptable for:**
- Dev/test environments with cost sensitivity
- Simple workloads where VNet-only restriction is sufficient
- When on-premises doesn't need direct access to the service

**Most common Private Endpoints in enterprise:**
- Azure SQL Database / SQL Managed Instance
- Azure Storage (Blob, ADLS Gen2, Tables, Queues)
- Azure Key Vault
- Azure CosmosDB
- Azure Service Bus
- Azure Event Hubs

**DNS configuration (critical — most missed step):**
```bash
# Create Private DNS Zone for storage
az network private-dns zone create \
  --resource-group rg-networking \
  --name "privatelink.blob.core.windows.net"

# Link to VNet (without this, DNS still resolves to public IP)
az network private-dns link vnet create \
  --resource-group rg-networking \
  --zone-name "privatelink.blob.core.windows.net" \
  --name link-to-hub-vnet \
  --virtual-network hub-vnet \
  --registration-enabled false

# Verify: from VM in VNet
nslookup mystorageaccount.blob.core.windows.net
# Should return 10.x.x.x (private IP), NOT 52.x.x.x (public IP)
```

---

**Q5. How do you implement the principle of least privilege for an AKS workload accessing Azure services?**

**Answer:**

**Modern approach: Workload Identity (replaces pod service principals with secrets)**

```
Step 1: Enable OIDC issuer on AKS cluster
  az aks update \
    --resource-group rg-aks \
    --name aks-prod \
    --enable-oidc-issuer \
    --enable-workload-identity

Step 2: Create user-assigned managed identity
  az identity create \
    --name id-order-processor \
    --resource-group rg-aks

Step 3: Grant ONLY the required permissions (least privilege)
  # e.g., order-processor needs to: read from Service Bus, write to CosmosDB, read from Key Vault
  az role assignment create \
    --role "Azure Service Bus Data Receiver" \
    --assignee $IDENTITY_CLIENT_ID \
    --scope /subscriptions/$SUB/resourceGroups/rg-data/providers/Microsoft.ServiceBus/...

  az cosmosdb sql role assignment create \
    --account-name prod-cosmos \
    --role-definition-id 00000000-0000-0000-0000-000000000002 \  # Cosmos DB Built-in Data Contributor
    --principal-id $IDENTITY_PRINCIPAL_ID \
    --scope /subscriptions/...

  az keyvault set-policy \
    --name prod-keyvault \
    --object-id $IDENTITY_PRINCIPAL_ID \
    --secret-permissions get list

Step 4: Create Kubernetes service account with annotation
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: order-processor-sa
    namespace: production
    annotations:
      azure.workload.identity/client-id: <IDENTITY_CLIENT_ID>

Step 5: Create federated credential (links K8s SA to Azure Identity)
  az identity federated-credential create \
    --identity-name id-order-processor \
    --name aks-federated-credential \
    --issuer $AKS_OIDC_ISSUER \
    --subject system:serviceaccount:production:order-processor-sa

Step 6: Label pods to use the identity
  spec:
    serviceAccountName: order-processor-sa
    labels:
      azure.workload.identity/use: "true"
```

**Result:** The pod gets a Kubernetes OIDC token → exchanges for Azure AD token → accesses only the specific resources it's been granted. **No secrets stored anywhere.**

---

**Q6. What is Microsoft Defender for Cloud and what does it protect?**

**Answer:**

Defender for Cloud is a unified security management platform combining:
1. **Cloud Security Posture Management (CSPM)**: Identifies misconfigurations and compliance gaps
2. **Cloud Workload Protection (CWP)**: Runtime threat detection per resource type

**Defender plans (per resource type):**

| Plan | Protects | Key detections |
|------|---------|----------------|
| Defender for Servers | VMs, AKS nodes | Malware, fileless attacks, brute force |
| Defender for SQL | Azure SQL, SQL MI | SQL injection attempts, anomalous queries |
| Defender for Storage | Blob, ADLS | Malware upload, suspicious download, unusual access |
| Defender for Key Vault | Key Vault | Unusual secret access, token abuse |
| Defender for Kubernetes | AKS | Crypto mining, privilege escalation, lateral movement |
| Defender for App Service | Web apps, Functions | Web shell, code injection, anomalous activity |

**Secure Score:**
- Numeric score (0–100) representing security posture
- Each recommendation has a score impact
- Useful for prioritizing remediation effort

**Common high-priority recommendations:**
- Enable MFA on all accounts with Contributor or higher
- No storage accounts with public blob access
- All SQL servers should have auditing enabled
- Private endpoints for PaaS services
- Enable Defender plans on all subscriptions

---

## ADVANCED

**Q7. Design a Zero Trust network architecture for a financial services AKS application.**

**Answer:**

**Design principles applied:**
1. No implicit trust based on network location
2. All traffic authenticated and authorized
3. Least privilege at every layer
4. All access logged and auditable

**Architecture:**

```
EXTERNAL USERS
  │
  ▼
Azure Front Door (Premium)
  WAF: OWASP 3.2 + custom financial rules
  Private Link origin: traffic never on public internet to backends
  Bot protection: enabled
  │
  ▼
APIM (internal mode, private VNet)
  OAuth 2.0 validation (Entra ID JWT required on all calls)
  Subscription key enforcement (per-consumer tracking)
  Rate limiting: 100 req/min per user, 1000 req/min per client app
  mTLS: certificate required from internal services
  │
  ▼
AKS CLUSTER (private cluster — API server not internet-accessible)
  Network Policy: Calico
    Default: deny all pod-to-pod
    Allow: specific namespace-to-namespace via NetworkPolicy objects
    
  example NetworkPolicy:
    allow order-processor → payment-service (port 8080 only)
    allow payment-service → cosmosdb-private-endpoint (port 443 only)
    deny everything else
  
  Workload Identity: per-service, minimal permissions (see Q5)
  Pod Security Standards: Restricted profile (no root, no privileged)
  Image scanning: ACR + Defender for Containers (blocks vulnerable images)
  
DATA LAYER (all Private Endpoints, no public access)
  Azure SQL MI: Private Endpoint → Private DNS Zone
  CosmosDB: Private Endpoint → Private DNS Zone
  Key Vault: Private Endpoint → Private DNS Zone
  Service Bus: Private Endpoint → Private DNS Zone

NETWORK
  Hub-Spoke topology:
    Hub: Azure Firewall Premium (IDPS enabled), Azure Bastion, Private DNS Resolver
    Spoke 1: AKS VNet
    Spoke 2: Data VNet (SQL MI, CosmosDB, Storage)
    Spoke 3: Management VNet (DevOps agents, monitoring)
  
  Azure Firewall Premium rules:
    Allow AKS → CosmosDB private endpoint (FQDN-based rule)
    Allow AKS → Key Vault private endpoint
    Block AKS → internet directly (all outbound through Firewall)
    IDPS: alert+deny on financial threat signatures

IDENTITY
  Entra ID:
    Conditional Access: MFA required for all human access
    PIM: no standing Contributor/Owner — JIT admin access only
    Emergency access accounts: 2 break-glass accounts, monitored 24/7
  
  RBAC:
    Dev team: Contributor on dev subscription, Reader on prod
    SRE team: Reader on prod + specific action permissions via custom role
    No direct data access: all data access via application identity only

MONITORING (audit trail for OSFI)
  All diagnostic logs → Log Analytics Workspace (immutable, 1yr hot / 7yr cold)
  Defender for Cloud: all plans enabled
  Microsoft Sentinel: SIEM connected to all log sources
  Alerts: anomaly detection on Key Vault access, storage, SQL queries
```

---

**Q8. How do you detect and respond to a compromised managed identity in Azure?**

**Answer:**

**Detection signals:**

```
Source 1: Defender for Cloud / Microsoft Sentinel
  Alert: "Anomalous activity for managed identity"
  Indicators:
  - Identity accessing resources it has never accessed before
  - Access from unexpected IP (should only come from Azure internal IPs)
  - Unusual volume of reads (exfiltration indicator)
  - Access at unusual hours

Source 2: Entra ID Sign-in Logs
  az monitor log-analytics query \
    --workspace $WORKSPACE_ID \
    --analytics-query "
      AADServicePrincipalSignInLogs
      | where ServicePrincipalId == '<identity-id>'
      | where TimeGenerated > ago(24h)
      | project TimeGenerated, IPAddress, ResourceDisplayName, ResultType
      | order by TimeGenerated desc"

Source 3: Key Vault Audit Logs
  Query for unusual secret access patterns

Source 4: Azure Activity Log
  Look for resource creation/deletion by the identity
```

**Immediate response:**

```
Step 1: Disable the identity immediately
  az identity update --name compromised-identity \
    --resource-group rg-prod \
    --set disableInfrastructure=true  # Via ARM if needed
  
  # Or: remove all RBAC assignments
  az role assignment list --assignee <identity-object-id> --all \
    | jq '.[].id' \
    | xargs -I{} az role assignment delete --ids {}

Step 2: Revoke tokens
  Managed Identity tokens have a 24hr max lifetime
  Cannot be revoked before expiry — rotate secrets it had access to

Step 3: Rotate all secrets the identity could access
  # Key Vault: generate new secret versions
  az keyvault secret set --vault-name prod-kv --name db-password \
    --value "$(openssl rand -base64 32)"
  
  # Storage: rotate account keys (SAS tokens based on old keys invalidated)
  az storage account keys renew --account-name prodstorage --key primary

Step 4: Investigate scope
  What did it access? → Log Analytics queries across all resource logs
  What did it create? → Activity Log
  Did it exfiltrate? → Storage access logs, Network flow logs

Step 5: Create new identity with clean permissions
  Do NOT re-enable compromised identity
  Create new user-assigned managed identity
  Apply least-privilege RBAC from scratch
```

**Prevention:**
- Enable Defender for Cloud's "Anomalous managed identity usage" alert
- Never give managed identities Owner or Contributor at subscription level
- Use Azure Policy to enforce maximum scope of managed identity assignments
- Regular RBAC access reviews (PIM access reviews quarterly)

---

**Q9. What are the OSFI B-10 and PIPEDA compliance requirements and how do they shape Azure architecture in Canada?**

**Answer:**

**OSFI B-10 (Technology and Cyber Risk Management):**
- Applies to: federally regulated financial institutions (banks, insurance, trust companies)
- Key requirements:

| Requirement | Architecture implication |
|-------------|------------------------|
| Data residency | All financial data in Canada (Canada Central + Canada East) |
| Technology risk management | Formal risk assessment for all cloud services |
| Third-party risk | Azure as third party must have OSFI-compliant contracts |
| Incident notification | Report significant incidents to OSFI within 24hr |
| Recovery objectives | RTO/RPO formally defined and tested |
| Concentration risk | Avoid over-reliance on single cloud provider |
| AI/ML model risk | Explainability, validation, monitoring required (OSFI SR 11-7 aligned) |

**PIPEDA (Personal Information Protection and Electronic Documents Act):**
- Applies to: commercial organizations handling personal data of Canadians
- Key principles:

| Principle | Azure control |
|-----------|--------------|
| Accountability | Data classification (Microsoft Purview), DPO designation |
| Identifying purposes | Document why PII is collected (data catalog) |
| Consent | Application-level (not infrastructure) |
| Limiting collection | Only collect what's needed |
| Limiting use | Access controls (RBAC), purpose binding |
| Accuracy | Data quality processes |
| Safeguards | Encryption at rest (SSE + CMK), in transit (TLS 1.2+), access controls |
| Openness | Privacy policy, data subject rights |
| Individual access | Capability to respond to data subject access requests |
| Challenging compliance | Audit trails, Log Analytics |

**FINTRAC (anti-money laundering):**
- Transaction records: 7-year retention
- Azure implementation: ADLS Gen2 WORM (immutable storage) + Lifecycle policy to Archive after 1 year
- Audit logs: Log Analytics (hot 1 year) + Azure Blob Archive (cold 6 years)

**Architecture checklist for Canadian financial services:**
```
✅ All Azure resources deployed to Canada Central or Canada East only
   → Azure Policy: deny resource creation outside these regions

✅ Data encrypted at rest with Customer-Managed Keys (CMK) in Key Vault
   → SQL MI: TDE with CMK
   → Storage: SSE with CMK
   → CosmosDB: encryption at rest with CMK

✅ All PaaS services accessed via Private Endpoints only
   → Azure Policy: deny storage without private endpoint

✅ Audit logs immutable and retained 7 years
   → Log Analytics workspace: 1yr hot retention + export to WORM storage
   → Azure Blob immutability policy: 7-year hold

✅ MFA for all human access + Conditional Access policies
   → Entra ID: Conditional Access requiring MFA for all admin roles

✅ Privileged access managed via PIM (Just-in-Time)
   → No standing Owner/Contributor access in production

✅ Incident response runbook documented and tested
   → OSFI B-10 requires < 24hr notification for significant incidents

✅ AI/ML models have explainability documentation
   → Azure Responsible AI dashboard + SHAP values for lending/fraud models
```
