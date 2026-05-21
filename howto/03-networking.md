# Azure Networking — Hands-On How-To

> *"In Azure, 90% of connectivity problems are DNS.  
>  The other 10% are also DNS, but they're hiding behind an NSG."*

---

**Q: How do you set up a Hub-Spoke network from scratch?**

You create the hub VNet first — typically a /16 with dedicated subnets for `AzureFirewallSubnet` (/26 minimum, exact name required), `GatewaySubnet` (/27 minimum, exact name required), and `AzureBastionSubnet` (/26 minimum, exact name required). Then you create spoke VNets with non-overlapping CIDRs and create bidirectional VNet peering — hub-to-spoke with `allowForwardedTraffic: true` and `allowGatewayTransit: true`, spoke-to-hub with `useRemoteGateways: true`. Finally, you add a User Defined Route (UDR) on each spoke subnet pointing `0.0.0.0/0` to the hub's Azure Firewall private IP — this forces all traffic through centralized inspection.

> 💡 **Deep dive hint:** Azure Firewall Premium IDPS mode, TLS inspection, and FQDN-based rules vs. IP rules — the tradeoffs.

---

**Q: How do you create a Private Endpoint and why does DNS always trip people up?**

You create the Private Endpoint resource pointing at your target service (e.g., Azure SQL), which drops a private IP into your subnet. Then you create a Private DNS Zone for the right namespace — `privatelink.database.windows.net` for SQL — and create a DNS zone group that links the private endpoint to the zone. The step everyone forgets: **linking the DNS zone to the VNet**. Without that link, VMs in the VNet still resolve the hostname to the service's public IP. After linking, `nslookup` from inside the VNet should return a `10.x.x.x` address, not a `52.x` one.

> 💡 **Deep dive hint:** Centralized Private DNS in Hub-Spoke — why you create all Private DNS Zones in the hub and link them once, rather than per-spoke.

---

**Q: How do NSG rules get evaluated and what's the most common mistake?**

Rules are evaluated by priority — lowest number wins. Azure has five built-in default rules that always exist at the bottom (priorities 65000–65500): allow VNet inbound, allow Azure Load Balancer inbound, deny all inbound, allow VNet outbound, allow internet outbound, deny all outbound. The most common mistake: blocking the `AzureLoadBalancer` service tag on inbound while trying to lock down a subnet, which breaks health probes and makes every VM appear unhealthy to the load balancer. The second most common: forgetting that NSGs on subnets AND NICs stack — both sets of rules must allow the traffic.

> 💡 **Deep dive hint:** NSG Flow Logs + Network Watcher Traffic Analytics — how to visualise allowed/denied flows in Log Analytics.

---

**Q: How do you connect on-premises to Azure and what's the difference between VPN and ExpressRoute?**

A VPN Gateway creates an encrypted IPsec tunnel over the public internet — it takes 30–45 minutes to provision and costs around $150/month for the basic SKU. ExpressRoute is a private dedicated circuit through a carrier (like Bell or Rogers) that never touches the public internet — it takes weeks to provision and starts around $500/month, but gives you consistent latency and up to 100Gbps. In practice: VPN for dev/test and small offices, ExpressRoute for core banking, large data transfers, and anything where latency consistency matters. Many enterprises run both — ExpressRoute as primary, VPN as failover.

> 💡 **Deep dive hint:** ExpressRoute Global Reach — connecting on-premises sites to each other through Azure, and when it makes sense.

---

**Q: How do you control traffic between two spokes — say, between the AKS spoke and the data spoke?**

By default, spoke-to-spoke traffic is blocked — VNet peering is not transitive. To allow it, you route both spokes through the hub Azure Firewall: add a UDR on the AKS subnet pointing the data spoke's CIDR to the Firewall IP, and a UDR on the data subnet pointing the AKS spoke's CIDR back to the Firewall IP. Then create an Azure Firewall Network Rule allowing the specific source/destination/port combination — for example, AKS subnet `10.1.0.0/24` to CosmosDB private endpoint `10.2.0.5` on port 443 only. Everything else between spokes remains blocked.

> 💡 **Deep dive hint:** Azure Virtual WAN as an alternative to Hub-Spoke for large-scale multi-region topologies.

---

**Q: How do you prevent AKS pods from accessing the internet directly?**

You configure the AKS cluster to use a User Defined Route table that sends `0.0.0.0/0` to the hub Azure Firewall. In Azure Firewall, you create application rules that allow only specific FQDNs — like MCR (Microsoft Container Registry), your ACR endpoint, and telemetry endpoints. Any pod trying to call `curl http://evil.com` hits the Firewall, doesn't match an allow rule, and gets dropped. The Firewall logs the denied connection with the source IP, so you can trace it to the exact pod.

> 💡 **Deep dive hint:** AKS egress with Azure Firewall — required FQDN allowlist for AKS control plane communication.

---

**Q: How does Azure DNS Private Resolver work and when do you need it?**

The Private DNS Resolver sits in your hub VNet and acts as a bridge between on-premises DNS and Azure Private DNS Zones. Without it, on-premises servers querying `myserver.database.windows.net` reach out to public DNS and get the public IP — the private endpoint is invisible to them. With an inbound endpoint, on-premises DNS forwards Azure queries to the Resolver's private IP via ExpressRoute or VPN. The Resolver checks the linked Private DNS Zones and returns the correct private IP. The outbound endpoint handles the reverse — Azure VMs resolving on-premises hostnames.

> 💡 **Deep dive hint:** DNS forwarding rules with Private DNS Resolver — conditional forwarders for on-premises domains and hybrid resolution architecture.

---

**Q: What is Azure Front Door and how is it different from Azure Load Balancer?**

Azure Load Balancer operates at Layer 4 (TCP/UDP) within a single region — it load-balances traffic between VMs in an availability set or zone without inspecting HTTP. Azure Front Door operates at Layer 7 (HTTP/HTTPS) globally — it routes traffic to the nearest healthy backend across regions, caches content at the edge, terminates TLS at the PoP (close to the user), and includes a WAF. If your app lives in Canada Central and a user in Tokyo opens it, Load Balancer can't help them — Front Door can route that user to a replica in Japan East and cut their latency from 200ms to 30ms.

> 💡 **Deep dive hint:** Azure Front Door Premium with Private Link origins — how to connect Front Door to your AKS backend without any public IPs on the backend.
