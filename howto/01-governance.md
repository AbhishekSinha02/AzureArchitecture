# Azure Governance — Hands-On How-To

> *"Governance is the art of saying no to everything that hasn't been approved yet  
>  — and making Azure enforce it for you."*

---

**Q: How do you structure Management Groups for a large enterprise?**

You build a tree that mirrors how the business delegates authority. The root sits above everything — only Global Admins can touch it. Below it you create groups like `mg-platform` (networking, identity, logging subscriptions) and `mg-workloads` (app teams grouped by environment or business unit). The magic is that a policy or RBAC assignment at any node in the tree flows down to every subscription underneath it automatically — you set it once, it applies everywhere.

> 💡 **Deep dive hint:** CAF (Cloud Adoption Framework) Management Group hierarchy — platform vs. landing zone vs. sandbox tiers.

---

**Q: How do you enforce that no one can create resources outside Canada Central or Canada East?**

You create an Azure Policy with the built-in definition `"Allowed locations"`, set the allowed values to `["canadacentral", "canadaeast"]`, and assign it at the Management Group level so every subscription inherits it. The effect is immediate — any ARM deployment targeting another region gets a `RequestDisallowedByPolicy` error before a single resource spins up. For existing out-of-region resources, the policy marks them as *non-compliant* but doesn't delete them — remediation is a separate task.

> 💡 **Deep dive hint:** Policy effects — `Deny` vs `Audit` vs `DeployIfNotExists` — and when to use each without breaking production.

---

**Q: How do you ensure every resource in Azure has a cost-center tag?**

You use the `"Require a tag and its value"` built-in policy assigned at the subscription or management group scope with `effect: Deny`. This blocks any resource creation that omits the tag. For resources created by ARM templates that don't control tags (like AKS auto-created node resource groups), you use a separate `"Inherit a tag from the resource group"` policy with `effect: Modify` — it automatically copies the tag from the parent resource group onto the child resource.

> 💡 **Deep dive hint:** Tag inheritance patterns, policy exemptions, and how to handle tags on resources created by managed services.

---

**Q: What is a Policy Initiative and when do you build one?**

An initiative is a package of multiple policies treated as one assignment. Instead of assigning 15 individual security policies one by one to every subscription, you group them into an initiative — say, `"OSFI Compliance Baseline"` — and assign the initiative once. Azure tracks compliance against the whole set as a single score. You build one when a compliance standard (OSFI, PCI-DSS, NIST) maps to many individual controls that always travel together.

> 💡 **Deep dive hint:** Built-in regulatory compliance initiatives in Azure — `Azure Security Benchmark`, `NIST SP 800-53`, `Canada Federal PBMM`.

---

**Q: How do you prevent teams from creating public storage accounts?**

There's a built-in policy called `"Storage accounts should restrict network access"` — but the stronger one is `"Storage accounts should prevent public blob access"`. Assign it with `effect: Deny` at the management group level. The moment a developer clicks "allow blob public access" in the portal and hits Save, Azure blocks it with a policy denial message that names exactly which policy fired. Teams learn fast.

> 💡 **Deep dive hint:** Combining Defender for Storage + Policy for defence-in-depth on storage security.

---

**Q: How do you audit who changed a policy assignment?**

The Azure Activity Log captures every change to policy assignments as a resource write event under `Microsoft.Authorization/policyAssignments`. You query it in Log Analytics: filter by `ResourceType == "PolicyAssignments"` and `OperationName contains "write"`. For longer retention, route the Activity Log to a Log Analytics Workspace and query with KQL. For compliance reporting, Azure Policy's compliance history shows *when* resources became compliant or non-compliant and what changed.

> 💡 **Deep dive hint:** Routing Activity Logs + Defender alerts into Microsoft Sentinel for a unified audit trail.

---

**Q: How do you handle a subscription that has drifted from your governance baseline?**

First, you run a compliance scan — Azure Policy's compliance dashboard shows exactly which resources violate which policies. For fixable violations (missing tags, wrong SKUs, disabled diagnostics), you use `remediation tasks` — Policy triggers a managed identity to run a DeployIfNotExists or Modify effect at scale across all non-compliant resources. For structural drift (wrong network topology, missing Private Endpoints), you use Terraform plan against the subscription to see what's out of state, then apply to correct it.

> 💡 **Deep dive hint:** Azure Policy remediation tasks — how they use managed identities, and why you need to pre-grant the right RBAC role.

---

**Q: How do you track costs per team or project in Azure?**

Tags are the foundation — every resource gets `costCenter`, `team`, and `project` tags enforced by Policy. Then in Azure Cost Management, you create a cost view filtered by those tag values and pin it to a shared dashboard. For chargeback, you export daily cost data to a storage account and load it into Power BI or Synapse so finance can build their own views. Budgets + alerts round it out: each team gets a budget with 80% and 100% threshold alerts going to their team email.

> 💡 **Deep dive hint:** Azure Cost Management exports, Power BI connector, and building a chargeback model with shared service allocation.
