# Phase 5: Hybrid Storage

## The Story So Far

ShipFast successfully migrated their SQL Server database to Aurora PostgreSQL, saving $150K in
licensing. Direct Connect provides reliable, high-bandwidth connectivity. But there's still 50TB of
file storage sitting in the datacenter.

## Business Trigger

The storage admin sends an alarming ticket:

> "NAS is at 92% capacity. We add about 2TB of shipping documents per month. At this rate, we'll be
> full in 60 days. A new shelf costs $80,000."

The Operations Director adds:

> "Drivers access proof-of-delivery documents via mapped network drives (SMB shares). If we change
> how they access files, we'll need to retrain 500 drivers and update all our mobile apps."

The compliance officer chimes in:

> "Shipping documents must be retained for 7 years. Our current backup system can't handle that
> volume reliably."

## Architecture Decision

**Decision**: Deploy AWS Storage Gateway (File Gateway mode) to extend on-premises storage to S3,
while maintaining SMB access for users.

### Why Storage Gateway?

| Requirement                 | Solution                            |
| --------------------------- | ----------------------------------- |
| Users need SMB shares       | File Gateway presents S3 as SMB/NFS |
| Need unlimited capacity     | S3 scales infinitely                |
| Fast access to recent files | Local cache on gateway              |
| 7-year retention            | S3 lifecycle policies to Glacier    |

## Key Concepts for SAA Exam

### AWS Storage Gateway Types

```mermaid
flowchart TB
    subgraph SGW["Storage Gateway Types"]
        FILE["File Gateway"]
        VOL["Volume Gateway"]
        TAPE["Tape Gateway"]
    end

    subgraph FILE_DESC["File Gateway"]
        F1["NFS/SMB → S3"]
        F2["Files stored as S3 objects"]
        F3["Local cache for hot data"]
    end

    subgraph VOL_DESC["Volume Gateway"]
        V1["iSCSI → EBS Snapshots"]
        V2["Block storage for apps"]
        V3["Cached or Stored mode"]
    end

    subgraph TAPE_DESC["Tape Gateway"]
        T1["VTL → S3/Glacier"]
        T2["Backup software compatible"]
        T3["Replace physical tape"]
    end

    FILE --- FILE_DESC
    VOL --- VOL_DESC
    TAPE --- TAPE_DESC

    style SGW fill:#e3f2fd,color:#000
    style FILE fill:#c8e6c9,color:#000
    style VOL fill:#fff9c4,color:#000
    style TAPE fill:#bbdefb,color:#000
    style FILE_DESC fill:#fff,color:#000
    style VOL_DESC fill:#fff,color:#000
    style TAPE_DESC fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### Detailed Comparison

| Gateway Type                | Protocol    | Backend           | Use Case                          |
| --------------------------- | ----------- | ----------------- | --------------------------------- |
| **File Gateway**            | NFS, SMB    | S3                | File shares, user directories     |
| **Volume Gateway (Cached)** | iSCSI       | S3 + local cache  | Frequently accessed block data    |
| **Volume Gateway (Stored)** | iSCSI       | Local + S3 backup | Full dataset on-prem, DR to cloud |
| **Tape Gateway**            | iSCSI (VTL) | S3, S3 Glacier    | Backup/archive, replace tape      |

### File Gateway Deep Dive

```mermaid
flowchart LR
    subgraph OnPrem["On-Premises"]
        USERS["Users/Apps"]
        CACHE["Local Cache<br>(SSD/RAM)"]
        GW["File Gateway<br>VM/Hardware"]
    end

    subgraph AWS["AWS"]
        S3["S3 Bucket"]
        GLACIER["S3 Glacier"]
    end

    USERS -->|"SMB/NFS"| GW
    GW --- CACHE
    GW -->|"HTTPS"| S3
    S3 -->|"Lifecycle<br>Policy"| GLACIER

    style OnPrem fill:#ffcdd2,color:#000
    style AWS fill:#e3f2fd,color:#000
    style USERS fill:#fff,color:#000
    style CACHE fill:#fff9c4,color:#000
    style GW fill:#fff,color:#000
    style S3 fill:#c8e6c9,color:#000
    style GLACIER fill:#bbdefb,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

**How it works**:

1. User saves file via SMB share
2. File stored in local cache
3. Gateway uploads to S3 asynchronously
4. User reads file - served from cache if available
5. Cache miss - gateway fetches from S3

> **Exam Tip**: Files are stored as S3 objects with 1:1 mapping. Metadata preserved via S3 object
> metadata.

### File Gateway Deployment Options

| Option                 | Where      | Best For                    |
| ---------------------- | ---------- | --------------------------- |
| **VMware ESXi**        | On-prem VM | Existing virtualization     |
| **Hyper-V**            | On-prem VM | Windows environments        |
| **EC2**                | AWS        | Cloud-to-cloud or DR        |
| **Hardware Appliance** | Physical   | No virtualization available |

### Volume Gateway Modes

```mermaid
flowchart TB
    subgraph Cached["Cached Volume"]
        C1["Primary data in S3"]
        C2["Hot data cached locally"]
        C3["Up to 1 PB total"]
        C4["32 volumes max"]
    end

    subgraph Stored["Stored Volume"]
        S1["Primary data on-prem"]
        S2["Async backup to S3"]
        S3["Up to 512 TB total"]
        S4["32 volumes max"]
    end

    style Cached fill:#c8e6c9,color:#000
    style Stored fill:#fff9c4,color:#000
    style C1 fill:#fff,color:#000
    style C2 fill:#fff,color:#000
    style C3 fill:#fff,color:#000
    style C4 fill:#fff,color:#000
    style S1 fill:#fff,color:#000
    style S2 fill:#fff,color:#000
    style S3 fill:#fff,color:#000
    style S4 fill:#fff,color:#000
```

> **Exam Tip**: Cached mode = data in cloud, cache locally. Stored mode = data locally, backup to
> cloud. Know the difference!

### Tape Gateway

Replaces physical tape infrastructure:

```mermaid
flowchart LR
    subgraph OnPrem["On-Premises"]
        BACKUP["Backup Software<br>(Veeam, Veritas)"]
        VTL["Virtual Tape<br>Library"]
    end

    subgraph AWS["AWS"]
        S3["S3<br>(Virtual Tapes)"]
        GLACIER["S3 Glacier<br>(Archived Tapes)"]
    end

    BACKUP -->|"iSCSI"| VTL
    VTL -->|"HTTPS"| S3
    S3 -->|"Archive"| GLACIER

    style OnPrem fill:#ffcdd2,color:#000
    style AWS fill:#e3f2fd,color:#000
    style BACKUP fill:#fff,color:#000
    style VTL fill:#fff,color:#000
    style S3 fill:#c8e6c9,color:#000
    style GLACIER fill:#bbdefb,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

Backup software sees virtual tapes, not cloud storage.

## Amazon FSx Options

For workloads that need true file system features (not just object storage):

### FSx for Windows File Server

```mermaid
flowchart TB
    subgraph FSxW["FSx for Windows File Server"]
        F1["Native Windows<br>SMB protocol"]
        F2["NTFS file system"]
        F3["Active Directory<br>integration"]
        F4["DFS namespaces"]
        F5["Multi-AZ deployment"]
    end

    style FSxW fill:#e3f2fd,color:#000
    style F1 fill:#fff,color:#000
    style F2 fill:#fff,color:#000
    style F3 fill:#fff,color:#000
    style F4 fill:#fff,color:#000
    style F5 fill:#fff,color:#000
```

**Use when**: Windows applications need SMB, AD integration, or Windows-specific features.

### FSx for Lustre

```mermaid
flowchart TB
    subgraph FSxL["FSx for Lustre"]
        L1["High-performance<br>parallel file system"]
        L2["Sub-millisecond<br>latency"]
        L3["100s of GB/s<br>throughput"]
        L4["S3 integration<br>(data repository)"]
    end

    style FSxL fill:#c8e6c9,color:#000
    style L1 fill:#fff,color:#000
    style L2 fill:#fff,color:#000
    style L3 fill:#fff,color:#000
    style L4 fill:#fff,color:#000
```

**Use when**: HPC, ML training, video processing - workloads needing extreme throughput.

### FSx Comparison

| Feature        | FSx Windows        | FSx Lustre      | FSx NetApp ONTAP | FSx OpenZFS     |
| -------------- | ------------------ | --------------- | ---------------- | --------------- |
| Protocol       | SMB                | Lustre          | NFS, SMB, iSCSI  | NFS             |
| Use case       | Enterprise Windows | HPC/ML          | Multi-protocol   | Linux workloads |
| AD integration | Yes                | No              | Yes              | No              |
| S3 integration | No                 | Yes (data repo) | No               | No              |

> **Exam Tip**: FSx for Windows = SMB + AD. FSx for Lustre = HPC + S3 integration.

## AWS Backup

Centralized backup management across AWS services:

```mermaid
flowchart TB
    subgraph Backup["AWS Backup"]
        VAULT["Backup Vault"]
        PLAN["Backup Plan"]
        RULES["Backup Rules"]
    end

    subgraph Resources["Protected Resources"]
        EC2["EC2"]
        RDS["RDS"]
        EFS["EFS"]
        FSX["FSx"]
        DDB["DynamoDB"]
        S3B["S3"]
    end

    PLAN --> RULES
    RULES --> VAULT
    EC2 & RDS & EFS & FSX & DDB & S3B -.->|"Backup to"| VAULT

    style Backup fill:#e3f2fd,color:#000
    style Resources fill:#c8e6c9,color:#000
    style VAULT fill:#fff,color:#000
    style PLAN fill:#fff,color:#000
    style RULES fill:#fff,color:#000
    style EC2 fill:#fff,color:#000
    style RDS fill:#fff,color:#000
    style EFS fill:#fff,color:#000
    style FSX fill:#fff,color:#000
    style DDB fill:#fff,color:#000
    style S3B fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### Backup Features

| Feature                  | Description                                |
| ------------------------ | ------------------------------------------ |
| **Backup Plans**         | Automated schedules and retention policies |
| **Backup Vault**         | Encrypted container for recovery points    |
| **Cross-region copy**    | DR to another region                       |
| **Cross-account backup** | Protect against account compromise         |
| **Vault Lock**           | WORM compliance (immutable backups)        |

## ShipFast Hybrid Storage Architecture

```mermaid
flowchart TB
    subgraph DC["ShipFast Datacenter"]
        USERS["Warehouse<br>Users"]
        DRIVERS["Mobile<br>Drivers"]

        subgraph SGWVM["Storage Gateway VM"]
            FGW["File Gateway"]
            CACHE["500GB<br>SSD Cache"]
        end

        NAS["Legacy NAS<br>(phase out)"]
    end

    subgraph DX["Direct Connect"]
        DXCONN["10 Gbps"]
    end

    subgraph AWS["AWS"]
        subgraph S3Tiers["S3 Storage"]
            S3STD["S3 Standard<br>Hot Documents"]
            S3IA["S3 IA<br>30+ days old"]
            GLACIER["S3 Glacier<br>1+ years old"]
        end

        BACKUP["AWS Backup"]
        AURORA[("Aurora<br>PostgreSQL")]
    end

    USERS -->|"SMB"| FGW
    DRIVERS -->|"API/Mobile"| S3STD
    FGW --- CACHE
    FGW -->|"HTTPS via DX"| DXCONN --> S3STD
    S3STD -->|"Lifecycle"| S3IA
    S3IA -->|"Lifecycle"| GLACIER
    BACKUP -.->|"Backs up"| AURORA

    style DC fill:#ffcdd2,color:#000
    style SGWVM fill:#fff9c4,color:#000
    style AWS fill:#e3f2fd,color:#000
    style S3Tiers fill:#c8e6c9,color:#000
    style DX fill:#fff9c4,color:#000
    style USERS fill:#fff,color:#000
    style DRIVERS fill:#fff,color:#000
    style FGW fill:#fff,color:#000
    style CACHE fill:#fff,color:#000
    style NAS fill:#ffcdd2,color:#000
    style DXCONN fill:#fff,color:#000
    style S3STD fill:#fff,color:#000
    style S3IA fill:#fff,color:#000
    style GLACIER fill:#bbdefb,color:#000
    style BACKUP fill:#fff,color:#000
    style AURORA fill:#bbdefb,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### ShipFast Storage Strategy

| Data Type            | Solution                   | Retention         |
| -------------------- | -------------------------- | ----------------- |
| Active shipping docs | File Gateway → S3 Standard | 30 days hot       |
| Older documents      | S3 lifecycle → S3 IA       | 30 days - 1 year  |
| Archived documents   | S3 lifecycle → Glacier     | 1-7 years         |
| Database backups     | AWS Backup                 | Daily for 30 days |

### S3 Lifecycle Policy

```json
{
  "Rules": [
    {
      "ID": "Shipping-Docs-Lifecycle",
      "Status": "Enabled",
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 365,
          "StorageClass": "GLACIER"
        }
      ],
      "Expiration": {
        "Days": 2555
      }
    }
  ]
}
```

### Cost Comparison

| Storage         | On-Prem NAS          | S3 + File Gateway        |
| --------------- | -------------------- | ------------------------ |
| 50TB capacity   | $80,000 upfront      | ~$1,150/month            |
| 7 years archive | Tape + offsite       | S3 Glacier: $0.004/GB    |
| Scaling         | Buy more shelves     | Automatic                |
| DR              | Manual tape rotation | Cross-region replication |

## DataSync for Initial Migration

To move the existing 50TB from NAS to S3:

```mermaid
flowchart LR
    subgraph OnPrem["On-Premises"]
        NAS["NetApp NAS<br>50TB"]
        AGENT["DataSync<br>Agent"]
    end

    subgraph AWS["AWS"]
        DS["AWS<br>DataSync"]
        S3["S3 Bucket"]
    end

    NAS --> AGENT
    AGENT -->|"10 Gbps via DX"| DS --> S3

    style OnPrem fill:#ffcdd2,color:#000
    style AWS fill:#e3f2fd,color:#000
    style NAS fill:#fff,color:#000
    style AGENT fill:#fff,color:#000
    style DS fill:#fff9c4,color:#000
    style S3 fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

**DataSync vs Storage Gateway**: | Feature | DataSync | Storage Gateway |
|---------|----------|-----------------| | Purpose | Migration, sync | Hybrid access | | Direction |
One-time or scheduled | Continuous | | Protocol | Agent-based | SMB/NFS | | Best for | Moving data |
Ongoing hybrid |

> **Exam Tip**: Use DataSync for migration, Storage Gateway for ongoing hybrid access.

## What Could Go Wrong?

File Gateway is deployed and working. Users access files via SMB just like before. The 50TB
migration completed via DataSync in under 2 days. The NAS is being decommissioned.

But the CTO looks at the calendar:

> "Our datacenter lease ends in 6 months. Database is migrated. Storage is migrated. But we still
> have 4 application servers running .NET in that datacenter. How do we move those?"

Time for the final migration.

## Exam Tips

- **File Gateway for SMB/NFS** - Maps to S3 objects, local cache for hot data
- **Volume Gateway Cached** - Primary data in S3, cache locally
- **Volume Gateway Stored** - Primary data local, async backup to S3
- **Tape Gateway** - Replace physical tape with VTL → S3/Glacier
- **FSx Windows for SMB + AD** - True Windows file server, not S3 object storage
- **FSx Lustre for HPC** - High-throughput, S3 data repository integration
- **DataSync for migration** - One-time or scheduled sync, not for ongoing access
- **AWS Backup** - Centralized, cross-region, cross-account, Vault Lock for compliance

## SAA Exam Concepts

### Must-Know for This Phase

| Concept        | Key Points                                           |
| -------------- | ---------------------------------------------------- |
| File Gateway   | SMB/NFS → S3, local cache, files as objects          |
| Volume Gateway | Cached (S3 primary) vs Stored (local primary)        |
| Tape Gateway   | VTL for backup software, S3/Glacier backend          |
| FSx Windows    | Native SMB, AD integration, Multi-AZ                 |
| FSx Lustre     | HPC, S3 integration, sub-ms latency                  |
| DataSync       | Migration and sync, agent-based, encrypted           |
| AWS Backup     | Centralized, cross-region, cross-account, Vault Lock |

---

## References

Official AWS documentation used to validate this content:

### AWS Storage Gateway

- [What is AWS Storage Gateway?](https://docs.aws.amazon.com/filegateway/latest/files3/WhatIsStorageGateway.html) -
  File, Volume, and Tape Gateway types
- [Volume Gateway](https://docs.aws.amazon.com/storagegateway/latest/vgw/WhatIsStorageGateway.html) -
  Cached mode (up to 1 PB), Stored mode (up to 512 TB), 32 volumes maximum

### Amazon FSx

- [Amazon FSx for Windows File Server](https://docs.aws.amazon.com/fsx/latest/WindowsGuide/what-is.html) -
  Native SMB, Active Directory integration, Multi-AZ
- [Amazon FSx for Lustre](https://docs.aws.amazon.com/fsx/latest/LustreGuide/what-is.html) -
  High-performance parallel file system, S3 data repository integration

### AWS DataSync

- [What is AWS DataSync?](https://docs.aws.amazon.com/datasync/latest/userguide/what-is-datasync.html) -
  Online data transfer service for migrations and scheduled syncs

### AWS Backup

- [What is AWS Backup?](https://docs.aws.amazon.com/aws-backup/latest/devguide/whatisbackup.html) -
  Centralized backup management, cross-region/cross-account copy, Vault Lock
