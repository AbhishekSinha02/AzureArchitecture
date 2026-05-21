# Authentication & Authorization — Hands-On How-To

> *"The question is never 'can this thing authenticate?'  
>  The question is 'does it need a password to do it?' — and the answer should be no."*

---

**Q: How do you give an Azure resource (like an AKS pod) access to another Azure resource (like Key Vault) without storing any credentials?**

You use a Managed Identity. For AKS specifically, you enable Workload Identity on the cluster, create a Kubernetes service account annotated with the managed identity's client ID, and create a federated credential that links the Kubernetes identity to the Azure managed identity. The pod gets a projected OIDC token — it exchanges that token with Azure AD for a short-lived access token scoped to Key Vault. No password, no certificate, no connection string stored anywhere. The whole chain is: pod SA → OIDC token → Entra ID → access token → Key Vault.

> 💡 **Deep dive hint:** Workload Identity Federation — how OIDC token exchange works, and how this same pattern extends to GitHub Actions authenticating to Azure without secrets.

---

**Q: What is the difference between a Managed Identity and a Service Principal?**

A service principal is an identity you create manually in Entra ID, and you're responsible for its credentials — you generate a secret or certificate, store it somewhere, and rotate it on a schedule. A managed identity is an identity Azure creates and manages for you — the credential rotation happens automatically, invisibly, and you never see the private key. System-assigned managed identities live and die with the resource they're attached to. User-assigned managed identities are standalone — you can attach the same identity to multiple resources, which is useful when ten microservices all need the same set of permissions.

> 💡 **Deep dive hint:** When service principals are still appropriate — third-party tools, multi-tenant scenarios, and CI/CD systems that don't support OIDC federation.

---

**Q: How does Azure RBAC work and how is it different from Entra ID roles?**

Azure RBAC controls what you can do to Azure *resources* — creating, reading, deleting VMs, storage accounts, AKS clusters. Entra ID roles control what you can do inside *Entra ID itself* — managing users, groups, applications, and conditional access policies. A person can be a `Global Administrator` in Entra ID (god-mode for identity) but have zero access to any Azure subscription. Conversely, a service principal can be `Owner` on a subscription but have no Entra ID role at all. The two systems overlap only at the seam: you need a minimum of `User Access Administrator` in Azure RBAC to assign roles in subscriptions.

> 💡 **Deep dive hint:** Custom RBAC role definitions — when built-in roles are too broad, and how to scope a role to allow only specific actions on specific resource types.

---

**Q: How do you audit who has what access in a subscription?**

You use the Azure Portal's Access Control (IAM) blade to see current assignments, but for audit-grade reporting you query the Azure Resource Graph. The KQL query `AuthorizationResources | where type == "microsoft.authorization/roleassignments"` returns every role assignment in scope. Pair it with the role definitions table to decode the role names. For Entra ID group memberships, you use the Microsoft Graph API or the `az ad group member list` CLI command. In regulated environments, you run this query weekly and diff it against the previous snapshot to catch any unauthorized access grants.

> 💡 **Deep dive hint:** Microsoft Entra ID Access Reviews — automated periodic review of group memberships and role assignments, with auto-removal if reviewers don't respond.

---

**Q: How do you ensure a developer can deploy to dev but not to production?**

You assign the developer `Contributor` on the dev subscription and `Reader` on the prod subscription. That's it structurally. To prevent lateral movement, you also make sure the dev and prod subscriptions are in separate management groups with no cross-subscription network peering. For production deployments, you use a service principal or managed identity tied to the CI/CD pipeline — humans never deploy to prod directly, only the pipeline does, and the pipeline gets triggered by a pull request approval from a senior engineer.

> 💡 **Deep dive hint:** Privileged Identity Management (PIM) for just-in-time production access — how to elevate temporarily instead of maintaining standing permissions.

---

**Q: How do you give an external vendor access to Azure resources without creating them an account in your tenant?**

You use Azure Entra B2B (Business-to-Business) guest access. You invite the vendor's email address — they authenticate with their own identity provider and receive a guest account in your tenant. You assign them RBAC roles just like a regular user, scoped to only the resource group or subscription they need. When the engagement ends, you delete the guest account. No shared passwords, no orphaned service accounts. For more controlled scenarios, you use Conditional Access policies that apply to guests specifically — for example, requiring MFA even if their home tenant doesn't enforce it.

> 💡 **Deep dive hint:** Entra External ID (formerly B2B + B2C) — the difference between guest access for vendors (B2B) and customer-facing identity (B2C/External ID).

---

**Q: How do you rotate a service principal secret without downtime?**

Service principals support multiple simultaneous secrets. You generate a new secret while the old one is still valid, update the application's configuration (Key Vault secret or environment variable) to use the new secret, deploy the updated config, verify the application is authenticating successfully with the new secret, and then delete the old secret. The overlap window — both secrets valid simultaneously — gives you safe deployment without any moment where the application has no valid credential. In Azure Key Vault, you can automate this with a Key Vault rotation policy that generates a new version on a schedule and triggers a Function App to update downstream references.

> 💡 **Deep dive hint:** Key Vault certificate auto-rotation with App Service integration — how to rotate TLS certificates without touching app configuration.
