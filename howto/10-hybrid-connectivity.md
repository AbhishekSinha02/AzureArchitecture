# Hybrid Connectivity & Data Transfer — Hands-On How-To

> *"Hybrid is the permanent state of the enterprise.  
>  The data centre doesn't disappear — it becomes one end of a very expensive pipe."*

---

**Q: How do you set up a Site-to-Site VPN between on-premises and Azure?**

You deploy a VPN Gateway in Azure (takes 25–45 minutes to provision — plan for this), configure the Local Network Gateway resource with your on-premises router's public IP and the CIDR blocks of your on-prem network, and set up the Connection resource with the shared key (pre-shared key). On the on-prem side, your router (Cisco, Fortinet, Palo Alto) or Windows RRAS needs the same shared key, the Azure Gateway's public IP, and IKE/IPsec policy to match what Azure negotiates. The most common failure mode: IKE policy mismatch — Azure defaults to IKEv2 with certain cipher suites, and older routers negotiate IKEv1 or different algorithms. Network Watcher's VPN diagnostics tool shows exactly where the handshake breaks.

> 💡 **Deep dive hint:** Active-active VPN Gateway — two public IPs, two tunnels, BGP for dynamic routing, and how it achieves near-99.99% uptime for the VPN connection itself.

---

**Q: What is ExpressRoute and what's the setup process from a practical standpoint?**

ExpressRoute is a private dedicated circuit from your office or data centre to Azure through a carrier partner (Bell, Rogers, Equinix, etc.). You don't order it from Azure — you contact the carrier first, they provision a physical cross-connect at a colocation facility, and give you a Service Key. You enter that Service Key when creating the ExpressRoute Circuit resource in Azure. The carrier then peers BGP with Microsoft's edge routers, and Azure advertises your VNet prefixes over that BGP session. End-to-end provisioning takes 2–8 weeks depending on the carrier. On the Azure side, you create an ExpressRoute Gateway in your hub VNet and link it to the circuit.

> 💡 **Deep dive hint:** ExpressRoute Direct — bypassing carrier partners entirely by connecting at 10/100Gbps directly to Microsoft's edge, used by very large enterprises.

---

**Q: How do you transfer large data volumes on an ongoing basis between on-premises and Azure — not a one-time migration?**

Azure Data Factory with a Self-Hosted Integration Runtime (SHIR) is the production answer. The SHIR is a lightweight agent you install on a Windows server with access to your on-prem data sources. ADF pipelines use the SHIR to run Copy Activities that read from on-prem SQL Server, Oracle, file shares, or SAP — whatever the source is — and write directly to ADLS Gen2 or Azure SQL without the data touching the public internet. The SHIR connects outbound on port 443 (HTTPS) to Azure, so no inbound firewall rules are needed. For high throughput, you install SHIR on multiple machines and ADF load-balances across them.

> 💡 **Deep dive hint:** SHIR high availability setup — two-node cluster, automatic job redistribution on node failure, and the 32-concurrent-job limit per node.

---

**Q: What is Azure Arc and how does it extend Azure governance to on-premises servers?**

Azure Arc projects your on-premises servers, Kubernetes clusters, and databases into Azure as ARM resources — as if they were native Azure resources. You install a lightweight Arc agent (`azcmagent`) on each server, it phones home to Azure, and the server appears in the Azure portal as a resource. From that point: Azure Policy applies to it (enforce antivirus, audit configurations), Defender for Cloud monitors it for threats, Azure Monitor collects its logs and metrics into the same Log Analytics workspace as your cloud resources, and you can run Update Manager or Automation runbooks against it. The governance model becomes uniform across cloud and on-premises.

> 💡 **Deep dive hint:** Arc-enabled Kubernetes — how to apply Azure Policy (via OPA/Gatekeeper), GitOps (Flux), and Azure Monitor Container Insights to an on-premises or multi-cloud Kubernetes cluster.

---

**Q: How do you use Azure Data Box for offline data transfer and what are the gotchas?**

You order the Data Box through the Azure portal, it arrives pre-configured with a network interface and credentials. You copy data to it using NFS or SMB over your local network (gigabit speeds, typically 1–2TB/hour). You lock it via the portal to prevent further writes, ship it back to Microsoft in the prepaid packaging, and within 10 business days the data appears in your storage account. The gotchas: Azure encrypts data on the box with AES-256 and wipes it after upload — you can't recover anything left on the box. The 40TB effective capacity is after encryption overhead. Data Box doesn't do incremental — it's a one-shot bulk copy, so plan your delta sync (AzCopy or ADF) starting from the moment you locked the box.

> 💡 **Deep dive hint:** Data Box Heavy (1PB capacity) vs. Data Box (80TB) — when each is appropriate, and the edge case where multiple Data Boxes in parallel ship simultaneously.

---

**Q: How does Azure File Sync work and when would you use it?**

Azure File Sync turns Azure Files into a sync hub for your on-premises Windows file servers. You install the Storage Sync Agent on the file server, register it with a Storage Sync Service in Azure, and create a sync group linking a server endpoint (a folder on the Windows server) to a cloud endpoint (an Azure Files share). Files sync bidirectionally. The powerful feature is cloud tiering: files not accessed recently get "tiered" — their content moves to Azure Files but a placeholder reparse point stays on the server. When a user opens the tiered file, it rehydrates transparently. This is how you shrink a 10TB file server to a 500GB local cache while keeping the user experience unchanged.

> 💡 **Deep dive hint:** Azure File Sync vs. Azure Files direct mount — when to use each, and the NTFS ACL preservation behaviour that matters when migrating Windows file server permissions.

---

**Q: What is a Hybrid Identity setup and how does Azure AD Connect (Entra Connect) work?**

Hybrid Identity means your on-premises Active Directory and Azure Entra ID are synchronised — users have one identity that works both on-premises and in the cloud. Azure AD Connect (now called Microsoft Entra Connect) is the sync engine: it runs on a Windows server, reads from on-prem AD, and writes user and group objects to Entra ID on a 30-minute sync cycle. Password Hash Sync (PHS) is the simplest mode: it syncs a hash of the password hash so users can authenticate to Azure with their AD credentials even if the on-prem AD is unreachable. Pass-through Authentication (PTA) is stricter: authentication is always forwarded to and validated by on-prem AD — if AD is down, cloud sign-in fails.

> 💡 **Deep dive hint:** Entra Connect Health — monitoring your sync pipeline, catching sync errors before they turn into locked-out users, and the agent that sends sync telemetry to Azure.

---

**Q: How do you troubleshoot a hybrid connectivity issue where on-premises servers can't reach an Azure Private Endpoint?**

The checklist has four stops. First: is the VPN or ExpressRoute connection healthy? Check the gateway resource in Azure — is the tunnel status Connected? Second: is the Private Endpoint DNS resolving correctly from on-premises? Run `nslookup myserver.database.windows.net` from the on-prem server — it should return the private IP (10.x.x.x). If it returns a public IP, your on-prem DNS isn't forwarding Azure DNS queries to the Private DNS Resolver's inbound endpoint. Third: is routing correct? The BGP table on the on-prem router should show the Azure VNet CIDR as reachable via the ExpressRoute/VPN next-hop. Fourth: is the NSG on the Private Endpoint subnet allowing traffic from the on-prem subnet CIDR on the correct port?

> 💡 **Deep dive hint:** Azure Network Watcher's Connection Troubleshoot tool — runs an end-to-end connectivity check between a VM and a target IP/FQDN and tells you exactly which hop or rule blocks it.
