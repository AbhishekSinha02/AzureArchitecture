# Azure Landing Zones — Hands-On How-To

> *"A landing zone is not a destination — it's a runway.  
>  You build it so every team that comes after you can take off safely."*

---

**Q: What is an Azure Landing Zone and how is it different from just creating a subscription?**

A landing zone is a subscription with opinions baked in — networking, identity, logging, policy, and RBAC are already configured before the first workload team arrives. Creating a subscription alone gives you a blank canvas. A landing zone gives you a canvas with guardrails: a VNet already peered to the Hub, a Log Analytics workspace already receiving diagnostic logs, Azure Policy already enforcing compliance, and RBAC already scoped so the app team can deploy but not change the network. The app team lands and builds — they don't have to think about plumbing.

> 💡 **Deep dive hint:** CAF (Cloud Adoption Framework) conceptual architecture — platform landing zones vs. application landing zones.

---

**Q: How do you actually deploy an Azure Landing Zone — what do you use?**

The fastest production path is the `ALZ-Bicep` or `Terraform Azure Landing Zones` reference implementation maintained by Microsoft. You clone the repo, fill in a parameters file (tenant ID, regions, log workspace settings, hub VNet CIDR), and run the deployment pipeline. It creates Management Groups, deploys platform subscriptions (connectivity, identity, management), configures Policy initiatives, and wires up diagnostics — all in one run. Most teams do it in stages: Management Group hierarchy first, then connectivity (Hub VNet + Firewall), then policy, then hand off spoke subscriptions to app teams.

> 💡 **Deep dive hint:** `ALZ-Bicep` vs `Enterprise-Scale Terraform` — module structure, state management, and how to customize without forking.

---

**Q: What are the platform subscriptions in a landing zone and what lives in each?**

The CAF model uses three platform subscriptions. **Connectivity** holds the hub VNet, Azure Firewall, ExpressRoute/VPN Gateways, and Azure DNS Private Resolver — everything that touches the network edge. **Identity** holds domain controllers, Azure AD DS if needed, and any identity infrastructure that can't run in workload subscriptions. **Management** holds the central Log Analytics workspace, Automation Account, Azure Update Manager, and Defender for Cloud — the observability and operations backbone. These three subscriptions are owned and operated by the platform team, not app teams.

> 💡 **Deep dive hint:** Why platform subscriptions exist in separate subscriptions rather than resource groups — blast radius, billing isolation, and subscription-level Azure Policy.

---

**Q: How do you onboard a new application team to the landing zone?**

You provision a new subscription, move it under the correct management group (e.g., `mg-corp-prod`), deploy a spoke VNet with a pre-approved CIDR, peer it to the hub, and assign the app team the `Contributor` role on their subscription (not the platform subscriptions). A Terraform module or a vending machine pipeline (Logic App or GitHub Actions workflow) handles the subscription creation and wiring automatically — the app team submits a PR with their parameters and the pipeline does the rest. They get a ready-to-use subscription in under 30 minutes.

> 💡 **Deep dive hint:** Subscription vending — automating new landing zone provisioning with GitHub Actions + Bicep or the Azure Landing Zone Accelerator.

---

**Q: How do you handle dev, staging, and production in a landing zone model?**

You create separate management group branches for each environment — `mg-corp-dev`, `mg-corp-prod` — and apply different policy initiatives to each. Production gets stricter policies (deny public endpoints, require Private Endpoints, require CMK). Dev gets softer policies (allow public endpoints, audit rather than deny). Each app team gets one subscription per environment, all peered to the same hub. This means a misconfiguration in dev can never propagate to prod — they're in isolated subscriptions with isolated networks.

> 💡 **Deep dive hint:** Policy inheritance across environment MG tiers — how to apply a strict prod baseline while keeping dev flexible without creating policy exceptions.

---

**Q: What is a "sandbox" subscription in a landing zone and why do you need one?**

A sandbox is a subscription under `mg-sandboxes` with almost no policies — teams can try anything, create any resource type, make it public-facing, break things. It exists so experimentation doesn't happen in a corporate network. The key constraint: sandbox subscriptions have no VNet peering to the hub. They are intentionally isolated — if something goes wrong, the blast radius is that one subscription. Teams prove out ideas in sandbox, then bring production-grade designs to the real landing zone.

> 💡 **Deep dive hint:** Sandbox time-boxing — automatic subscription expiry using Azure Policy + Azure Automation to decommission sandboxes after 90 days.

---

**Q: How do you ensure every new landing zone subscription automatically sends logs to the central workspace?**

You use a `DeployIfNotExists` policy assigned at the management group level. The policy checks whether a subscription's Activity Log diagnostic setting points to the central Log Analytics workspace — if not, it triggers a remediation task that deploys the diagnostic setting automatically using a system-assigned managed identity. The same pattern applies for individual resources: a DeployIfNotExists policy ensures every VM, AKS cluster, and SQL database has its diagnostic settings configured the moment it's created.

> 💡 **Deep dive hint:** Policy-driven governance at scale — DINE (DeployIfNotExists) policies, the managed identity requirement, and how to grant the right role for remediation.
