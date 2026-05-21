# Azure Security & Policy — Hands-On How-To

> *"Security in Azure isn't a feature you turn on —  
>  it's a hundred small decisions that add up to a posture."*

---

**Q: What does Defender for Cloud actually do day-to-day — is it just a dashboard?**

Defender for Cloud has two modes. The free CSPM (Cloud Security Posture Management) tier continuously scans your Azure resources and scores them against the Microsoft Cloud Security Benchmark — it tells you that your SQL server has no auditing, your storage account has public access enabled, your VMs have no endpoint protection. That part is always on. When you enable the paid Defender plans (per-service: Servers, SQL, Storage, Kubernetes, Key Vault), you get runtime threat detection — actual alerts like "SQL injection attempt detected on database X" or "Crypto mining process running on VM Y." The dashboard is where you triage; the threat detection is what fires at 2 AM.

> 💡 **Deep dive hint:** Defender for Cloud's attack path analysis — how it models lateral movement paths through your environment and prioritises which fixes close the most paths.

---

**Q: How do you enforce that all Azure SQL databases must have Transparent Data Encryption enabled?**

There's a built-in policy called `"Transparent Data Encryption on SQL databases should be enabled"` with effect `AuditIfNotExists`. You assign it at the management group level. Within 24 hours, Azure Policy scans all SQL databases and marks those without TDE as non-compliant. To go from audit to enforce, you change the effect to `DeployIfNotExists` using a custom policy definition — Azure then auto-deploys TDE on any non-compliant database using a managed identity. The shift from Audit to DeployIfNotExists is the moment governance teeth appear.

> 💡 **Deep dive hint:** The difference between `AuditIfNotExists`, `DeployIfNotExists`, and `Modify` effects — when each is appropriate and the managed identity requirement.

---

**Q: What is the Azure Security Benchmark and how does it map to real configuration?**

The Azure Security Benchmark (ASB) is Microsoft's opinionated set of security best practices, each mapped to controls from NIST, CIS, and ISO 27001. In Defender for Cloud, it's a built-in regulatory compliance initiative — you can see at a glance what percentage of controls you meet and drill into each one to see which specific resources are violating it. Each recommendation links directly to the remediation step. In practice, teams use the ASB as their default compliance baseline and layer on industry-specific initiatives (like OSFI or PCI-DSS) on top of it.

> 💡 **Deep dive hint:** Custom compliance initiatives — how to build a Policy initiative from scratch that maps to your internal security framework or an industry standard not in the built-in list.

---

**Q: How do you detect if someone is accessing your Key Vault in an unusual way?**

Defender for Key Vault monitors access patterns using machine learning. If a managed identity that normally reads one specific secret starts bulk-downloading all secrets, or if an access comes from an unfamiliar IP — it fires an alert. For manual monitoring, Key Vault diagnostic logs go to Log Analytics, and you query them with KQL: filter for `ResultSignature` of `Forbidden` or filter for a service principal that shouldn't be accessing the vault. For production, you also set an Azure Monitor Alert on the metric `ServiceApiResult` where result is not `200` — any access denial shows up within minutes.

> 💡 **Deep dive hint:** Key Vault soft-delete and purge protection — the two settings that prevent ransomware from destroying your secrets and encryption keys.

---

**Q: What is Microsoft Sentinel and when do you need it vs just Log Analytics?**

Log Analytics is a log store with a query engine — it holds your data and lets you run KQL queries. Sentinel is a SIEM (Security Information and Event Management) built on top of Log Analytics, adding: automated detection rules that run continuously against incoming logs, incident management (group related alerts into an investigation), playbooks (Logic Apps that auto-respond to incidents), and built-in connectors to hundreds of data sources (Microsoft 365, Defender, Entra ID, third-party firewalls). You need Sentinel when you need automated threat detection and incident response, not just log storage. For a Canadian bank under OSFI, Sentinel is the SIEM tier of your security program.

> 💡 **Deep dive hint:** Sentinel Analytics rules — Fusion rules (ML-based multi-signal correlation), scheduled rules (KQL on a timer), and NRT (near-real-time) rules.

---

**Q: How do you prevent someone from accidentally creating a public-facing VM?**

Three layers. First, Azure Policy with `Deny` effect on `Microsoft.Network/publicIPAddresses` creation in production management groups — no public IP can be created, period. Second, Azure Firewall in the hub with no outbound NAT rule for unexpected source ranges. Third, Defender for Cloud recommendation: `"Virtual machines should not have public IP addresses"` fires as a High severity alert if any VM somehow gets a public IP. You don't rely on any one of these — all three together mean a public IP can't be created, can't route traffic even if it exists, and gets noticed within minutes if something slips through.

> 💡 **Deep dive hint:** Azure Policy `Deny` for public IPs vs. `DeployIfNotExists` to automatically remove them — and why Deny is almost always better.

---

**Q: How do you handle secrets in a CI/CD pipeline without storing them in the repository?**

The modern answer is: no secrets at all. GitHub Actions supports OIDC federation with Azure — the workflow gets a GitHub-issued OIDC token, exchanges it for an Azure AD access token, and uses that to authenticate. No client secret, no certificate, nothing stored in GitHub Secrets or the repo. For secrets that still need to exist (database passwords, API keys for third-party services), you store them in Key Vault and have the pipeline fetch them at runtime using the OIDC-authenticated identity. The pipeline never has a long-lived credential — every token it uses expires within minutes.

> 💡 **Deep dive hint:** `azure/login` GitHub Action with OIDC — the exact federated credential setup in Entra ID and the GitHub workflow YAML.

---

**Q: What is the difference between encryption at rest and encryption in transit, and how does Azure handle both?**

Encryption at rest protects data stored on disk from physical theft or unauthorized storage access — Azure applies it by default using platform-managed keys (Azure generates and manages the key) on all services including Blob, SQL, CosmosDB, and managed disks. For regulated industries, you upgrade to customer-managed keys (CMK) stored in Key Vault — you control the key, you can revoke it, and Azure can't read your data without your key. Encryption in transit protects data moving over the network — Azure enforces HTTPS/TLS on all PaaS service endpoints, and you enforce minimum TLS version (1.2+) via Azure Policy to stop older clients from negotiating weak cipher suites.

> 💡 **Deep dive hint:** Double encryption — Azure infrastructure encryption layer added on top of service-level encryption, and when OSFI requires it.
