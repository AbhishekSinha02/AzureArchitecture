# Azure High Availability — Hands-On How-To

> *"High availability is not about preventing failure.  
>  It's about making failure so boring that no one notices."*

---

**Q: What is the difference between Availability Sets and Availability Zones?**

An Availability Set spreads VMs across fault domains (separate physical racks, different power and network) and update domains (not rebooted simultaneously during Azure maintenance) within a single datacenter. It protects against hardware failure and planned maintenance. An Availability Zone spreads VMs across physically separate datacenters within a region — separate power, cooling, and networking. A zone outage (a whole datacenter) takes down your Availability Set but not your zone-redundant deployment. The SLA tells the story: single VM = 0%, Availability Set = 99.95%, Availability Zones = 99.99%. In new designs, always use Availability Zones.

> 💡 **Deep dive hint:** Zone-redundant services vs. zone-pinned services — Azure SQL ZRS, Storage ZRS, and how zone-redundancy works differently for managed PaaS vs. IaaS VMs.

---

**Q: How does Azure Traffic Manager work and how is it different from Azure Load Balancer?**

Traffic Manager is a DNS-based global load balancer — it doesn't touch your actual traffic, it just decides which DNS entry to return. When a user queries your domain, Traffic Manager's DNS response points them to the healthiest endpoint based on your routing method (priority, weighted, performance, geographic). Azure Load Balancer distributes actual TCP/UDP packets between VMs within a region. Traffic Manager operates above the transport layer — zero latency overhead, no data path — while Load Balancer is in the data path. For multi-region failover, Traffic Manager is the tool; for multi-VM distribution within a region, Load Balancer is the tool.

> 💡 **Deep dive hint:** Azure Front Door vs Traffic Manager — both do global routing, but Front Door operates at Layer 7 (HTTP) and caches; use Front Door for web apps, Traffic Manager for TCP/UDP or non-HTTP workloads.

---

**Q: How do you make an Azure SQL Database highly available?**

Within a region, Azure SQL Business Critical tier includes an Always On availability group under the hood — three synchronous replicas across availability zones, automatic failover in seconds. You don't configure this; it's built in. For cross-region HA, you enable **Active Geo-Replication** — up to four readable secondary databases in other regions, asynchronous replication. For automatic failover across regions, you use **Auto-failover Groups** — you point your application at the group listener endpoint (a DNS name), and if the primary fails, Azure promotes the secondary and updates the DNS record. Your application reconnects to the same connection string and is back online without any code change.

> 💡 **Deep dive hint:** Active geo-replication RTO vs RPO tradeoffs — the seconds of data loss during async replication and how to design your application to handle it.

---

**Q: How do you design an AKS cluster to survive an availability zone failure?**

You create the AKS cluster with `--zones 1 2 3` and spread node pools across all three zones. For critical system pods (CoreDNS, metrics-server), you use `topologySpreadConstraints` to distribute replicas across zones. For your application deployments, you set a `PodDisruptionBudget` (e.g., min available = 2) and a `topologySpreadConstraint` requiring pods in different zones. Persistent volumes need zone-aware storage class — `managed-csi` with Azure Disk is zone-pinned, so if your pod moves zones, it can't attach the disk from another zone. Use `Azure Files` (ZRS) or CosmosDB for any state that must survive zone failures.

> 💡 **Deep dive hint:** AKS cluster topology spread constraints vs. pod anti-affinity — when each is appropriate and the performance implications of forced zone distribution.

---

**Q: How does CosmosDB achieve 99.999% availability and what do you actually have to configure?**

CosmosDB's SLA is automatic for multi-region accounts: you add a second read region and the SLA jumps from 99.99% to 99.999% for reads. For writes to also hit 99.999%, you enable multi-region writes. In the Azure portal, it's two toggles: add a region, enable multi-region writes. Azure handles all the replication, conflict resolution, and failover logic. The only configuration you own is the **consistency level** — choosing Strong consistency means writes must be acknowledged by all regions before returning, which protects against data loss but increases write latency. Choosing Session or Eventual consistency trades some consistency guarantees for faster writes.

> 💡 **Deep dive hint:** CosmosDB auto-failover priority — how you rank your regions and what happens when the primary fails, including the brief write-unavailability window.

---

**Q: How do you test that your failover actually works before it's needed?**

You run a chaos experiment. For AKS: use Azure Chaos Studio to inject a node pool failure or a zone failure and watch whether your pods reschedule and traffic continues. For database failover: use the Azure portal's "Initiate failover" button on your SQL auto-failover group or CosmosDB account — this is a non-destructive test that promotes the secondary and then fails back. For region-level failover: update Traffic Manager to mark the primary endpoint unhealthy and verify the secondary starts receiving traffic within the health probe interval. The goal is to run these tests quarterly and prove your RTO in writing — OSFI reviewers ask for this evidence.

> 💡 **Deep dive hint:** Azure Chaos Studio — fault library (VM shutdown, network latency, pod kill, AKS node drain) and how to build an experiment that validates your HA design.

---

**Q: What SLA does a single App Service instance give you, and how do you improve it?**

A single instance of Azure App Service gives no SLA — if the underlying host has a hardware failure, your app is down until Azure migrates it. With two or more instances in a Standard or Premium plan, you get 99.95%. With two or more instances in an Availability Zone-enabled Premium v3 plan deployed across zones, you get 99.99%. The jump from one to two instances is the highest-value HA move you can make. After that, adding AZ support is the next jump. Everything else — geo-redundancy, Front Door — is layered on top.

> 💡 **Deep dive hint:** App Service zone redundancy requirements — minimum 3 instances in a Premium v3 plan, and why the billing model changes with zone redundancy enabled.
