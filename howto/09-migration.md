# Azure Migration — Hands-On How-To

> *"A migration without a rollback plan is not a migration.  
>  It's a leap of faith with other people's data."*

---

**Q: How do you use Azure Migrate to assess an on-premises environment?**

You deploy the Azure Migrate appliance — a lightweight OVA (for VMware) or Hyper-V VM — inside your on-premises environment. It connects to vCenter or Hyper-V Manager, discovers VMs automatically, and spends 1–2 weeks profiling CPU, memory, and disk utilisation (not just peak, but over time). You then open Azure Migrate in the portal, run an assessment, and get a report: which VMs are "Ready" for Azure (and which target SKU fits), which have compatibility issues, and an estimated monthly cost. The sizing recommendation is the key output — without it, you're guessing VM sizes and either over-provisioning (expensive) or under-provisioning (performance problems).

> 💡 **Deep dive hint:** Azure Migrate dependency mapping — how to discover which VMs talk to each other before you migrate, so you move dependent workloads together.

---

**Q: How do you migrate a VM from on-premises to Azure with minimal downtime?**

You set up replication using Azure Migrate Server Migration (or Azure Site Recovery for existing workloads). The appliance replicates the initial disk image to Azure — this is the bulk copy, takes hours or days depending on VM size and network. After the initial sync, it continuously replicates delta changes (every few seconds). When ready, you run a test migration first — a non-disruptive failover to a test VNet where you verify the VM boots and the application works. On cutover day, you stop the source VM, let the final delta sync complete (seconds to minutes of data), migrate, update DNS and connection strings, and the source VM becomes read-only standby for 72 hours before you decommission it.

> 💡 **Deep dive hint:** Azure Migrate for physical servers (not just VMware/Hyper-V) — the agent-based replication approach for bare-metal and cloud-to-cloud migrations.

---

**Q: How do you copy 50TB of files from on-premises NAS to Azure Blob Storage?**

AzCopy is the right tool — it parallelises transfers, supports resume on failure, and can validate MD5 checksums. You install it on a server with network access to the NAS, authenticate to Azure using a SAS token or Managed Identity, and run: `azcopy copy '/nas/data/*' 'https://account.blob.core.windows.net/container?<SAS>' --recursive --check-md5=FailIfDifferent`. For 50TB, the transfer time depends on bandwidth: at 1Gbps it's roughly 4.5 days. If that's too slow, you order an Azure Data Box (80TB capacity), copy the data locally to the device, ship it back, and Microsoft uploads it at their datacenter's internal speed — typical turnaround is 10–15 days end-to-end including shipping.

> 💡 **Deep dive hint:** AzCopy performance tuning — `--block-size-mb`, `--cap-mbps`, `--concurrency` flags, and how to saturate a 10Gbps link.

---

**Q: How do you migrate an on-premises SQL Server database to Azure with less than 1 hour of downtime?**

You use Azure Database Migration Service (DMS) in online migration mode. DMS reads the transaction log continuously from the source SQL Server and applies changes to the Azure SQL target in near-real-time — the replication lag is typically seconds. You let this run for days while the source stays live. When the lag reaches near zero, you initiate the "cutover" window: stop writes to the source (maintenance page or connection draining), wait for lag to hit zero, DMS completes the final log restore, you update your application's connection string to point at Azure SQL, and restore writes. Total downtime: the time it takes to drain connections and verify the target — typically 10–30 minutes, sometimes less.

> 💡 **Deep dive hint:** DMS prerequisites that trip people up — CDC must be enabled on the source database, SQL Server Agent must be running, and the DMS service needs inbound access to the source on port 1433.

---

**Q: What is the first thing you do when a migration fails mid-cutover?**

You execute the rollback plan you wrote before starting. The source system should still be running in read-only or paused-write mode — not stopped and decommissioned. You flip the DNS entry or load balancer target back to the source, restore writes, and the application is back online. Then you investigate what failed, fix it, and reschedule the cutover. The golden rule: keep the source hot for at least 72 hours after a successful cutover. Decommissioning it before that window passes is the single most common cause of irreversible migration failures.

> 💡 **Deep dive hint:** Migration runbook structure — what every production migration document should include: go/no-go checklist, cutover sequence with timestamps, rollback sequence, smoke test list, and escalation contacts.

---

**Q: How do you validate that a migrated database has all its data intact?**

Three-layer validation. First, row counts: `SELECT COUNT(*) FROM each_table` on both source and target should match exactly. Second, checksum: `SELECT CHECKSUM_AGG(BINARY_CHECKSUM(*)) FROM each_table` — if a row was silently modified, the checksum differs even if counts match. Third, business validation: run the key reports and transactions that matter to the business — the daily transaction total, the account balance query, the most-used stored procedure — and compare results. Automated tooling like `tablediff` (SQL Server) or a Python script that samples records from source and target is faster than manual comparison at scale.

> 💡 **Deep dive hint:** Great Expectations — open-source Python library for defining and running data quality assertions across source and target during and after migration.

---

**Q: How does the Azure Database Migration Service handle large databases that take days to transfer initially?**

DMS breaks the migration into two phases. In the full load phase, it reads the entire source database and copies it to the target — for large databases, this runs concurrently across multiple tables using parallel workers. While the full load is running, DMS also starts capturing changes from the transaction log so no changes are lost. Once full load completes, the apply phase starts: all the log changes captured during full load are applied to the target, and then new changes are applied continuously. By the time you want to cut over, the target is seconds behind the source regardless of how long the initial copy took.

> 💡 **Deep dive hint:** DMS vs Debezium for CDC — when the self-hosted open-source approach gives you more flexibility, especially for non-Microsoft source databases.
