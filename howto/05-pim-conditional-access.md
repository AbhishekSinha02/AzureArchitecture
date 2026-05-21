# PIM & Conditional Access — Hands-On How-To

> *"Standing admin access is like leaving your front door unlocked because  
>  you might need to come home in a hurry. PIM is the lock with a smart key."*

---

**Q: What is Privileged Identity Management (PIM) and how does it work in practice?**

PIM turns permanent role assignments into *eligible* ones. Instead of a DBA being permanently `Contributor` on the production database subscription, they're *eligible* — they have no access until they need it. When they need access, they go to PIM in the portal, click Activate, enter a justification, and optionally wait for an approver to click Approve. Azure assigns the role for a defined window (say, 4 hours), then automatically removes it when the window expires. Every activation is logged — who, when, why, how long — which is exactly what OSFI auditors ask for.

> 💡 **Deep dive hint:** PIM for groups — assigning PIM eligibility to an Entra ID group rather than individual users, so one activation can grant multiple resources at once.

---

**Q: How do you set up PIM for a production subscription so no one has standing Owner access?**

You go to PIM → Azure Resources → select the subscription → Role Settings → Owner → configure: activation duration 2 hours max, require justification, require approval, require MFA on activation. Then you go to Assignments → remove any permanent Owner assignments → add those users as "Eligible" instead. From that point, the subscription has zero standing Owners. The break-glass account (a separate emergency account) keeps a permanent Owner role but is monitored by an alert that fires if it ever signs in.

> 💡 **Deep dive hint:** Break-glass emergency access accounts — why you need two, how to store the credentials offline, and how to monitor them with Azure Monitor alerts.

---

**Q: What is Conditional Access and how is it different from MFA enforcement in user settings?**

Legacy per-user MFA is a blunt instrument — on or off, applies to everything, no context awareness. Conditional Access is a policy engine that evaluates *signals* — who is the user, what device are they on, what's the sign-in risk level, what app are they accessing, from which location — and then enforces the *right* control for that specific context. You can require MFA for admins but not for low-risk users on compliant corporate devices accessing a non-sensitive app. You can block access entirely if the sign-in risk is High. Conditional Access replaces legacy MFA and should be used instead, not alongside it.

> 💡 **Deep dive hint:** Conditional Access report-only mode — how to deploy a new policy in "Report Only" for two weeks to see its impact before enforcing it.

---

**Q: How do you block access to Azure from all countries except Canada?**

In Conditional Access, you create a Named Location for Canada (select country from the built-in list), then create a policy: `Assignments: All users` → `Conditions: Locations: All locations except Canada` → `Access controls: Block`. Now any sign-in from outside Canada is blocked. The exception is your break-glass accounts — you create a separate group for them and exclude that group from this policy, so emergency access is never geo-blocked. You also exclude your service principals from this policy since they authenticate from Azure datacenters, not physical locations.

> 💡 **Deep dive hint:** Named Locations with trusted IP ranges for VPN exit nodes — how to mark your corporate VPN as a trusted location and relax MFA requirements for it.

---

**Q: What is a Conditional Access Sign-In Risk policy and how does Azure know if a sign-in is risky?**

Entra ID Protection runs machine learning against every sign-in — it evaluates things like impossible travel (logging in from Toronto and then from Singapore 10 minutes later), leaked credentials (Microsoft monitors dark web dumps), unfamiliar sign-in properties (new device, new location, new browser), and anonymous IP address (Tor exit node). Each sign-in gets a risk score: Low, Medium, or High. A risk-based Conditional Access policy intercepts High-risk sign-ins and forces an MFA challenge or blocks them entirely — the user either proves they're legit or gets locked out.

> 💡 **Deep dive hint:** Entra ID Protection's User Risk vs Sign-In Risk — the difference, and how to build a self-remediation policy where users can unblock themselves with MFA.

---

**Q: How does Conditional Access interact with service principals and CI/CD pipelines?**

By default, Conditional Access policies with `All users` in scope apply to human users, not service principals — service principals have their own Conditional Access workload identity policies (a newer feature). In practice, most teams explicitly exclude service principals from user-facing policies by adding them to an exclusion group. For CI/CD pipelines using service principals, the recommended path is to migrate them to Workload Identity Federation (OIDC) so they never hold a credential and can't be targeted by credential-stuffing attacks.

> 💡 **Deep dive hint:** Workload Identity Conditional Access policies — new capability for applying controls specifically to service principals and managed identities.

---

**Q: How do you use Hybrid Access to extend Conditional Access to on-premises apps?**

Azure Entra Application Proxy publishes your on-premises web app to the internet through an Entra ID endpoint. Users access the app through a `*.msappproxy.net` URL, authenticate against Entra ID (subject to all your Conditional Access policies — MFA, device compliance, location), and the Application Proxy connector (a lightweight agent running on your on-prem network) forwards the authenticated request to the internal app. The internal app never needs to be publicly exposed. This is how you get MFA and Conditional Access on an old on-premises app that was built before modern auth existed.

> 💡 **Deep dive hint:** Entra Application Proxy with Kerberos Constrained Delegation (KCD) — enabling SSO for Windows-integrated authentication apps through the proxy.

---

**Q: What happens when Conditional Access blocks a legitimate user and they call the help desk?**

The sign-in logs in the Entra ID portal are your first stop — filter by username and the approximate time, find the blocked sign-in, and the entry tells you exactly which Conditional Access policy blocked it and why. Common causes: user signed in from a personal device (not Intune-enrolled), user's account was flagged as High risk, user tried from a blocked country while on travel. The fix depends on the cause: for a travelling employee blocked by geo-policy, you temporarily add them to the policy exclusion group. For a risk-flagged account, you reset their risk level in Entra ID Protection and require a password change.

> 💡 **Deep dive hint:** Entra ID Sign-In Diagnostic tool — the built-in wizard that walks through a specific sign-in and tells you exactly which policy applied and why.
