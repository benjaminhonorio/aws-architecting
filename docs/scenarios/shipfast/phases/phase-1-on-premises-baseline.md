# Phase 1: On-Premises Baseline

## The Story So Far

ShipFast Logistics has been running its own datacenter for 15 years. The infrastructure works, but
cracks are showing. The CTO has asked you - the new Cloud Architect - to assess the situation and
develop a migration strategy.

## Business Trigger

The CFO drops by your desk with concerning news:

> "Our datacenter lease is up in 18 months. Renewing costs $2M, plus we need $500K in hardware
> refresh. The board wants to know: can we move to the cloud instead?"

You also hear from Operations:

> "Last month's power outage took us down for 6 hours. We don't have a real DR (Disaster Recovery)
> site - just tape backups stored off-site. If this building floods, we're done."

## Current Architecture

```mermaid
flowchart TB
    subgraph DC["ShipFast Datacenter"]
        subgraph AppTier["Application Tier"]
            LB["F5 Load<br>Balancer"]
            APP1["App Server 1<br>Windows/.NET"]
            APP2["App Server 2<br>Windows/.NET"]
            APP3["App Server 3<br>Windows/.NET"]
            APP4["App Server 4<br>Windows/.NET"]
        end

        subgraph DataTier["Data Tier"]
            SQL1[("SQL Server<br>Primary<br>2TB")]
            SQL2[("SQL Server<br>Standby")]
            NAS["NetApp NAS<br>50TB"]
        end

        subgraph Infra["Infrastructure"]
            AD["Active<br>Directory"]
            DNS["Internal<br>DNS"]
            BACKUP["Veeam<br>Backup"]
        end
    end

    subgraph WH["Warehouses (12 sites)"]
        WH1["Warehouse 1"]
        WH2["Warehouse 2"]
        WHN["Warehouse N..."]
    end

    Internet((Internet)) --> LB
    LB --> APP1 & APP2 & APP3 & APP4
    APP1 & APP2 & APP3 & APP4 --> SQL1
    APP1 & APP2 & APP3 & APP4 --> NAS
    SQL1 -.->|Replication| SQL2
    WH1 & WH2 & WHN -->|Site VPN| DC

    style DC fill:#ffcdd2,color:#000
    style AppTier fill:#ffebee,color:#000
    style DataTier fill:#ffebee,color:#000
    style Infra fill:#ffebee,color:#000
    style WH fill:#fff9c4,color:#000
    style LB fill:#fff,color:#000
    style APP1 fill:#fff,color:#000
    style APP2 fill:#fff,color:#000
    style APP3 fill:#fff,color:#000
    style APP4 fill:#fff,color:#000
    style SQL1 fill:#bbdefb,color:#000
    style SQL2 fill:#bbdefb,color:#000
    style NAS fill:#c8e6c9,color:#000
    style AD fill:#fff,color:#000
    style DNS fill:#fff,color:#000
    style BACKUP fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

## The Migration Assessment

Before jumping to solutions, you need to assess the current state. AWS provides tools for this:

### AWS Application Discovery Service

Collects configuration and usage data from on-premises servers:

- **Agentless discovery** - Uses VMware vCenter to collect VM inventory
- **Agent-based discovery** - Installs agents for deeper data (processes, connections, performance)
- Feeds into **AWS Migration Hub** for tracking

### AWS Migration Evaluator (formerly TSO Logic)

Analyzes on-premises workloads and provides:

- Right-sizing recommendations
- Cost projections for AWS
- Business case for migration

## Key Concepts for SAA Exam

### The 7 Rs of Migration

This framework appears constantly on the SAA exam. Know each strategy and when to use it:

| Strategy       | Description                              | When to Use                       | ShipFast Example          |
| -------------- | ---------------------------------------- | --------------------------------- | ------------------------- |
| **Rehost**     | "Lift and shift" - move as-is to EC2     | Quick migration, minimal changes  | App servers to EC2        |
| **Replatform** | "Lift and reshape" - minor optimizations | Easy wins without rewriting       | SQL Server to RDS         |
| **Repurchase** | Replace with SaaS                        | Commodity workloads               | Move to Salesforce        |
| **Refactor**   | Re-architect for cloud-native            | Strategic apps needing scale      | Future: containerize apps |
| **Relocate**   | Move to AWS with minimal changes         | VMware workloads, rapid migration | VMware Cloud on AWS       |
| **Retire**     | Decommission                             | Unused or redundant systems       | Legacy reporting server   |
| **Retain**     | Keep on-premises                         | Compliance, latency, or not ready | (none for ShipFast)       |

### Migration Strategies Comparison

```mermaid
flowchart LR
    subgraph Speed["Migration Speed"]
        direction TB
        FAST["Fastest"]
        SLOW["Slowest"]
    end

    subgraph Strategies["The 7 Rs"]
        R1["Rehost<br>'Lift & Shift'"]
        R2["Replatform<br>'Lift & Reshape'"]
        R3["Repurchase<br>'Drop & Shop'"]
        R4["Refactor<br>'Re-architect'"]
    end

    subgraph Optimization["Cloud Optimization"]
        direction TB
        LOW["Least Optimized"]
        HIGH["Most Optimized"]
    end

    FAST --> R1
    R1 --> R2
    R2 --> R3
    R3 --> R4
    R4 --> SLOW

    R1 --> LOW
    R2 -.-> |partial| LOW
    R3 -.-> |partial| HIGH
    R4 --> HIGH

    style FAST fill:#c8e6c9,color:#000
    style SLOW fill:#ffcdd2,color:#000
    style LOW fill:#fff9c4,color:#000
    style HIGH fill:#c8e6c9,color:#000
    style R1 fill:#e3f2fd,color:#000
    style R2 fill:#e3f2fd,color:#000
    style R3 fill:#e3f2fd,color:#000
    style R4 fill:#e3f2fd,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### ShipFast Migration Decisions

After assessment, here's the proposed strategy for each component:

| Component        | Strategy                 | Rationale                                 |
| ---------------- | ------------------------ | ----------------------------------------- |
| App Servers      | **Rehost** (Phase 6)     | Quick win, minimal risk, optimize later   |
| SQL Server       | **Replatform** (Phase 4) | Move to RDS, reduce licensing costs       |
| File Storage     | **Replatform** (Phase 5) | Storage Gateway + S3, keep NFS/SMB access |
| Active Directory | **Retain then Extend**   | AWS Managed AD later                      |
| Backup           | **Repurchase** (Phase 5) | Replace Veeam with AWS Backup             |

## Architecture Decision

**Decision**: Hybrid-first approach with phased migration

We won't do a "big bang" migration. Instead:

1. **Connect first** - Establish VPN, then Direct Connect
2. **Migrate data** - Database and files (highest risk, do early)
3. **Migrate compute** - Applications last (can always fall back)

### Why Hybrid First?

| Approach          | Pros                        | Cons                       |
| ----------------- | --------------------------- | -------------------------- |
| **Big Bang**      | Single cutover, clean break | High risk, long planning   |
| **Phased/Hybrid** | Lower risk, incremental     | Complexity of running both |

For 24/7 operations like ShipFast, phased migration is almost always the right answer.

## Migration Phases Overview

```mermaid
flowchart TB
    subgraph Phase1["Phase 1-3: Connectivity"]
        VPN["VPN<br>Connection"] --> DX["Direct<br>Connect"]
    end

    subgraph Phase2["Phase 4-5: Data Migration"]
        DB["Database<br>DMS"] --> STORAGE["Storage<br>Gateway"]
    end

    subgraph Phase3["Phase 6: Compute Migration"]
        APPS["Application<br>Migration"]
    end

    Phase1 --> Phase2
    Phase2 --> Phase3

    style Phase1 fill:#fff9c4,color:#000
    style Phase2 fill:#c8e6c9,color:#000
    style Phase3 fill:#bbdefb,color:#000
    style VPN fill:#fff,color:#000
    style DX fill:#fff,color:#000
    style DB fill:#fff,color:#000
    style STORAGE fill:#fff,color:#000
    style APPS fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

## What Could Go Wrong?

You present the migration plan to leadership. The CTO asks:

> "This all sounds great on paper, but how do we actually connect our datacenter to AWS? We can't
> just migrate data over the public internet - it's too slow and not secure enough for our shipping
> data."

Good question. That's exactly what we'll solve in Phase 2.

## Exam Tips

- **7 Rs appear frequently** - Know the difference between Rehost (no changes), Replatform (minor
  changes), and Relocate (VMware Cloud on AWS)
- **Migration Hub** - Central place to track migration progress across tools
- **Application Discovery Service** - Two modes: agentless (VMware) and agent-based (deeper data)
- **Hybrid is usually the answer** - SAA rarely asks about big-bang migrations
- **Assess before migrating** - Questions often test whether you'd assess first or just start
  migrating

## SAA Exam Concepts

### Must-Know for This Phase

| Concept               | Key Points                                                          |
| --------------------- | ------------------------------------------------------------------- |
| 7 Rs of Migration     | Rehost, Replatform, Repurchase, Refactor, Relocate, Retire, Retain  |
| AWS Migration Hub     | Centralized tracking, integrates with discovery tools               |
| Application Discovery | Agentless (vCenter) vs Agent-based (installed on servers)           |
| Migration Evaluator   | Business case, TCO (Total Cost of Ownership) analysis, right-sizing |

---

## References

Official AWS documentation used to validate this content:

### Migration Strategies

- [About the Migration Strategies](https://docs.aws.amazon.com/prescriptive-guidance/latest/large-migration-guide/migration-strategies.html) -
  The 7 Rs: Rehost, Replatform, Repurchase, Refactor, Relocate, Retire, Retain

### AWS Application Discovery Service

- [What is AWS Application Discovery Service?](https://docs.aws.amazon.com/application-discovery/latest/userguide/what-is-appdiscovery.html) -
  Agentless discovery (VMware vCenter), agent-based discovery, and Migration Hub integration

### AWS Migration Hub

- [AWS Migration Hub Dashboard](https://docs.aws.amazon.com/application-discovery/latest/userguide/dashboard.html) -
  Centralized tracking and migration status visualization
