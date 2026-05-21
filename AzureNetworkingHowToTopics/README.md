# Azure Networking — How-To Scenario Library

> Scenario-driven, diagram-first, command-ready.  
> Every question is a real situation. Every answer shows you exactly what to do and why.

---

## Files in This Library

| # | File | What's inside |
|---|------|---------------|
| 01 | [hub-spoke-fundamentals.md](01-hub-spoke-fundamentals.md) | Build Hub-Spoke from zero, peering, UDR, DNS — Beginner |
| 02 | [hub-spoke-advanced.md](02-hub-spoke-advanced.md) | Multi-region, vWAN, Firewall transit, spoke isolation — Advanced |
| 03 | [nsg-allow-deny-access.md](03-nsg-allow-deny-access.md) | NSG rules: allow, deny, exception, service tags, ASGs |
| 04 | [private-endpoints-dns.md](04-private-endpoints-dns.md) | Private Endpoints end-to-end, DNS zones, gotchas |
| 05 | [firewall-and-routing.md](05-firewall-and-routing.md) | Azure Firewall, UDR, forced tunnelling, inspection |
| 06 | [hybrid-scenarios.md](06-hybrid-scenarios.md) | VPN Gateway, ExpressRoute, Arc, File Sync, hybrid DNS |
| 07 | [troubleshooting-playbook.md](07-troubleshooting-playbook.md) | Systematic diagnosis — NSG, DNS, routes, Firewall, BGP |
| 08 | [commands-reference.md](08-commands-reference.md) | Every useful CLI / PowerShell / kubectl command with context |

---

## Levels at a Glance

- 🟢 **Beginner** — What is it, how do you build it for the first time
- 🟡 **Intermediate** — Multi-service interactions, real configurations, common errors
- 🔴 **Advanced** — Large-scale design, edge cases, failure modes, BGP, security hardening

---

## How to Read These Files

Every answer follows this structure:
1. **The scenario** — what situation you're in
2. **The diagram** — ASCII topology so you see the shape
3. **The answer** — bullet points with key services **bolded**
4. **The commands** — runnable CLI snippets
5. **The gotcha** — the one thing that breaks this in production
6. **Deep dive hint** — what to open as a follow-up thread

---

> *"Azure networking is a series of doors.  
>  NSGs are the locks. Routes are the corridors. DNS is the sign on each door.  
>  If you can't get there, one of the three is wrong."*
