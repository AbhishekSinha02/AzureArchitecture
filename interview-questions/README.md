# Azure Interview Question Bank

> Production-grade Q&A covering architecture, data, security, networking, and operations.
> Targeted at **Senior / Lead Cloud / Data Architect** level interviews.

---

## Structure

| File | Topics | Levels |
|------|--------|--------|
| [01-fundamentals.md](01-fundamentals.md) | Core Azure concepts, IaaS/PaaS/SaaS, regions, resource model | Beginner |
| [02-migration.md](02-migration.md) | 6Rs, Azure Migrate, Data Box, CDC, zero-downtime cutover | B / I / A |
| [03-databases.md](03-databases.md) | SQL, CosmosDB, PostgreSQL Flexible, Redis, database selection | B / I / A |
| [04-scalability.md](04-scalability.md) | AKS KEDA, autoscale, partitioning, caching, traffic management | B / I / A |
| [05-disaster-recovery.md](05-disaster-recovery.md) | RTO/RPO, active-active, geo-redundancy, backup, failover | B / I / A |
| [06-security-zero-trust.md](06-security-zero-trust.md) | Zero Trust, Entra ID, RBAC, Private Endpoints, Defender | B / I / A |
| [07-apim.md](07-apim.md) | Policies, tiers, versioning, OAuth, APIM with AKS | B / I / A |
| [08-storage-adls-synapse.md](08-storage-adls-synapse.md) | Blob, ADLS Gen2, Synapse, lakehouse, hot/cold tiering | B / I / A |
| [09-data-factory-checksums.md](09-data-factory-checksums.md) | ADF pipelines, checksums, CDC, Self-hosted IR, data validation | B / I / A |
| [10-networking.md](10-networking.md) | VNet, NSG, Private Endpoints, ExpressRoute, Hub-Spoke | B / I / A |
| [11-howto-logic-apps.md](11-howto-logic-apps.md) | How-to guides: large files, chunking, error handling, patterns | How-To |

**Level Key:** B = Beginner · I = Intermediate · A = Advanced

---

## How to Use This Bank

1. **Self-study**: Read each file end-to-end, cover the answer, try to answer yourself first.
2. **Mock interview**: Use the questions only — have someone read them aloud.
3. **Deep dives**: Each Advanced answer links to follow-up areas worth expanding.
4. **Whiteboard prep**: Every Advanced answer follows the structure from CLAUDE.md §8.3.

---

## Interview Answer Formula (Senior Level)

```
1. Clarify scope — scale, SLA, budget, compliance
2. State assumptions
3. Start simple, layer complexity
4. Lead with WHY: "I chose X over Y because at this scale..."
5. Name failure modes proactively
6. Mention cost + ops burden
7. Close: "This gives us [outcome] at [cost] with [SLA]"
```
