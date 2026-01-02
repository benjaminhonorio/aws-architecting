# Phase 3: Dedicated Connectivity - Direct Connect

## The Story So Far

ShipFast established a Site-to-Site VPN connection to AWS. Testing was successful, but performance
issues emerged during peak hours. The VPN's reliance on the public internet causes inconsistent
latency and occasional packet loss.

## Business Trigger

The Operations Director calls an emergency meeting:

> "Our real-time tracking system needs sub-50ms latency to work properly. The VPN averages 80ms with
> spikes to 200ms. Drivers are seeing stale data and customers are complaining about inaccurate
> ETAs."

The network engineer adds:

> "We're also maxing out the VPN at 1.25 Gbps during database backups. Once we start migrating 50TB
> of files, we'll be saturated for days."

The CFO does the math:

> "Transferring 50TB over the internet at $0.09/GB is $4,500. Plus the bandwidth costs. Direct
> Connect might actually save us money."

## Architecture Decision

**Decision**: Establish AWS Direct Connect for production workloads, keeping VPN as backup.

### Why Direct Connect?

| Requirement                 | VPN                  | Direct Connect        |
| --------------------------- | -------------------- | --------------------- |
| Consistent latency          | No (internet varies) | Yes (dedicated fiber) |
| 10+ Gbps bandwidth          | No (1.25 Gbps limit) | Yes (up to 400 Gbps)  |
| Reduced data transfer costs | No                   | Yes (~50% cheaper)    |
| SLA-backed connection       | No                   | Yes                   |

## Key Concepts for SAA Exam

### How Direct Connect Works

```mermaid
flowchart LR
    subgraph OnPrem["ShipFast Datacenter"]
        ROUTER["Router"]
    end

    subgraph DXLoc["Direct Connect Location"]
        CAGE["AWS Cage"]
        CUST["Customer/Partner<br>Cage"]
        XCONN["Cross<br>Connect"]
    end

    subgraph AWS["AWS Region"]
        DXR["DX<br>Router"]
        VGW["Virtual Private<br>Gateway"]
        VPC["VPC"]
    end

    ROUTER -->|"Fiber from colo<br>or partner"| CUST
    CUST --- XCONN
    XCONN --- CAGE
    CAGE --> DXR
    DXR -->|"Private VIF"| VGW
    VGW --- VPC

    style OnPrem fill:#ffcdd2,color:#000
    style DXLoc fill:#fff9c4,color:#000
    style AWS fill:#e3f2fd,color:#000
    style ROUTER fill:#fff,color:#000
    style CAGE fill:#bbdefb,color:#000
    style CUST fill:#fff,color:#000
    style XCONN fill:#c8e6c9,color:#000
    style DXR fill:#bbdefb,color:#000
    style VGW fill:#fff,color:#000
    style VPC fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### Direct Connect Components

| Component                   | Description                                            |
| --------------------------- | ------------------------------------------------------ |
| **Direct Connect Location** | Physical facility where AWS has presence (colocation)  |
| **Connection**              | Physical port on AWS router (1, 10, or 100 Gbps)       |
| **Cross-Connect**           | Cable linking your cage to AWS cage in the DX location |
| **Virtual Interface (VIF)** | Logical connection over the physical port              |
| **Direct Connect Gateway**  | Enables connections to multiple VPCs/regions           |

### Connection Types

#### Dedicated Connection

- Physical port directly from AWS
- Speeds: **1 Gbps, 10 Gbps, 100 Gbps, or 400 Gbps**
- You manage the cross-connect
- Lead time: weeks to months

#### Hosted Connection

- Through an AWS Partner
- Speeds: **50 Mbps to 25 Gbps**
- Partner manages the cross-connect
- Lead time: days to weeks
- Good when you don't have datacenter in a DX location

```mermaid
flowchart TB
    subgraph Dedicated["Dedicated Connection"]
        D1["1, 10, 100, or 400 Gbps"]
        D2["Direct physical port"]
        D3["You manage cross-connect"]
        D4["Longer lead time"]
    end

    subgraph Hosted["Hosted Connection"]
        H1["50 Mbps - 25 Gbps"]
        H2["Via AWS Partner"]
        H3["Partner manages connection"]
        H4["Faster provisioning"]
    end

    style Dedicated fill:#bbdefb,color:#000
    style Hosted fill:#c8e6c9,color:#000
    style D1 fill:#fff,color:#000
    style D2 fill:#fff,color:#000
    style D3 fill:#fff,color:#000
    style D4 fill:#fff,color:#000
    style H1 fill:#fff,color:#000
    style H2 fill:#fff,color:#000
    style H3 fill:#fff,color:#000
    style H4 fill:#fff,color:#000
```

> **Exam Tip**: Dedicated = 1/10/100/400 Gbps. Hosted = flexible speeds from 50 Mbps to 25 Gbps.

### Virtual Interfaces (VIFs)

You can create multiple VIFs over a single physical connection:

| VIF Type        | Purpose                                   | Connects To                                       |
| --------------- | ----------------------------------------- | ------------------------------------------------- |
| **Private VIF** | Access VPC resources                      | Virtual Private Gateway or Direct Connect Gateway |
| **Public VIF**  | Access AWS public services (S3, DynamoDB) | AWS public endpoints                              |
| **Transit VIF** | Access multiple VPCs via Transit Gateway  | Transit Gateway                                   |

```mermaid
flowchart TB
    subgraph DX["Direct Connect"]
        CONN["1 Physical<br>Connection"]
    end

    subgraph VIFs["Virtual Interfaces"]
        PRIV["Private VIF<br>VLAN 100"]
        PUB["Public VIF<br>VLAN 200"]
        TRANS["Transit VIF<br>VLAN 300"]
    end

    subgraph Targets["Destinations"]
        VPC["VPC<br>(via VGW)"]
        S3["S3, DynamoDB<br>(public endpoints)"]
        TGW["Transit Gateway<br>(multiple VPCs)"]
    end

    CONN --> PRIV & PUB & TRANS
    PRIV --> VPC
    PUB --> S3
    TRANS --> TGW

    style DX fill:#e3f2fd,color:#000
    style VIFs fill:#fff9c4,color:#000
    style Targets fill:#c8e6c9,color:#000
    style CONN fill:#fff,color:#000
    style PRIV fill:#fff,color:#000
    style PUB fill:#fff,color:#000
    style TRANS fill:#fff,color:#000
    style VPC fill:#fff,color:#000
    style S3 fill:#fff,color:#000
    style TGW fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### Direct Connect Gateway

Allows a single Direct Connect to reach multiple VPCs across regions:

```mermaid
flowchart LR
    subgraph OnPrem["Datacenter"]
        ROUTER["Router"]
    end

    subgraph DX["Direct Connect"]
        CONN["Connection"]
        VIF["Private VIF"]
    end

    DXGW["Direct Connect<br>Gateway"]

    subgraph Region1["us-east-1"]
        VGW1["VGW"]
        VPC1["VPC 1"]
    end

    subgraph Region2["eu-west-1"]
        VGW2["VGW"]
        VPC2["VPC 2"]
    end

    ROUTER --> CONN --> VIF --> DXGW
    DXGW --> VGW1 --> VPC1
    DXGW --> VGW2 --> VPC2

    style OnPrem fill:#ffcdd2,color:#000
    style DX fill:#fff9c4,color:#000
    style Region1 fill:#e3f2fd,color:#000
    style Region2 fill:#c8e6c9,color:#000
    style DXGW fill:#bbdefb,color:#000
    style ROUTER fill:#fff,color:#000
    style CONN fill:#fff,color:#000
    style VIF fill:#fff,color:#000
    style VGW1 fill:#fff,color:#000
    style VGW2 fill:#fff,color:#000
    style VPC1 fill:#fff,color:#000
    style VPC2 fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

> **Exam Tip**: Direct Connect Gateway enables multi-region access without needing a DX connection
> in each region.

### Link Aggregation Groups (LAG)

Bundle multiple connections for higher bandwidth and redundancy:

- Combine up to **4 connections** of the same speed
- All connections must terminate at the same DX location
- Uses LACP (Link Aggregation Control Protocol)
- Example: 4 x 10 Gbps = 40 Gbps aggregate

### Resilience Patterns

```mermaid
flowchart TB
    subgraph Pattern1["Development/Non-Critical"]
        P1["Single DX<br>Connection"]
        P1NOTE["No redundancy"]
    end

    subgraph Pattern2["High Resiliency"]
        P2A["DX Connection<br>Location A"]
        P2B["DX Connection<br>Location B"]
        P2NOTE["Survives location failure"]
    end

    subgraph Pattern3["Maximum Resiliency"]
        P3A1["DX 1<br>Location A"]
        P3A2["DX 2<br>Location A"]
        P3B1["DX 1<br>Location B"]
        P3B2["DX 2<br>Location B"]
        P3NOTE["Survives device + location failure"]
    end

    style Pattern1 fill:#ffcdd2,color:#000
    style Pattern2 fill:#fff9c4,color:#000
    style Pattern3 fill:#c8e6c9,color:#000
    style P1 fill:#fff,color:#000
    style P2A fill:#fff,color:#000
    style P2B fill:#fff,color:#000
    style P3A1 fill:#fff,color:#000
    style P3A2 fill:#fff,color:#000
    style P3B1 fill:#fff,color:#000
    style P3B2 fill:#fff,color:#000
    style P1NOTE fill:#ffcdd2,color:#000
    style P2NOTE fill:#fff9c4,color:#000
    style P3NOTE fill:#c8e6c9,color:#000
```

## ShipFast Direct Connect Architecture

```mermaid
flowchart TB
    subgraph DC["ShipFast Datacenter"]
        ROUTER["Border Router"]
    end

    subgraph DXLoc["Direct Connect Location"]
        PARTNER["Equinix<br>Partner Cage"]
    end

    subgraph AWS["AWS"]
        DXGW["Direct Connect<br>Gateway"]

        subgraph VPNB["VPN Backup"]
            VPN["Site-to-Site<br>VPN"]
        end

        subgraph VPCMain["Production VPC"]
            VGW["VGW"]
            PRIV["Private VIF"]
            WEB["Web Tier"]
            APP["App Tier"]
        end
    end

    ROUTER -->|"10 Gbps<br>Hosted Connection"| PARTNER
    PARTNER -->|"Cross Connect"| PRIV
    PRIV --> DXGW --> VGW
    VGW --- WEB & APP

    ROUTER -.->|"Backup Path<br>(lower priority)"| VPN
    VPN -.-> VGW

    style DC fill:#ffcdd2,color:#000
    style DXLoc fill:#fff9c4,color:#000
    style AWS fill:#e3f2fd,color:#000
    style VPNB fill:#ffebee,color:#000
    style VPCMain fill:#e3f2fd,color:#000
    style ROUTER fill:#fff,color:#000
    style PARTNER fill:#fff,color:#000
    style DXGW fill:#bbdefb,color:#000
    style VPN fill:#fff,color:#000
    style VGW fill:#fff,color:#000
    style PRIV fill:#c8e6c9,color:#000
    style WEB fill:#fff,color:#000
    style APP fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### ShipFast's Choices

| Decision               | Choice           | Rationale                                            |
| ---------------------- | ---------------- | ---------------------------------------------------- |
| Connection Type        | Hosted (10 Gbps) | No presence in DX location, partner handles physical |
| VIF Type               | Private VIF      | Access VPC resources, not public AWS services        |
| Redundancy             | DX + VPN backup  | VPN provides failover during DX maintenance          |
| Direct Connect Gateway | Yes              | Future multi-region expansion                        |

### VPN as Backup to Direct Connect

Using BGP, you can configure automatic failover:

```
Route Priority:
1. Direct Connect path (BGP AS-path shorter, preferred)
2. VPN path (BGP AS-path longer, backup only)

If DX fails → BGP routes via VPN automatically
```

> **Exam Tip**: This DX + VPN backup pattern is extremely common in SAA questions.

## Direct Connect vs VPN - Cost Comparison

For ShipFast's 50TB file migration:

| Cost Factor              | VPN                 | Direct Connect           |
| ------------------------ | ------------------- | ------------------------ |
| Data transfer (50TB out) | $4,500 @ $0.09/GB   | $900 @ $0.02/GB          |
| Connection cost          | $36/month           | ~$500/month (hosted 10G) |
| Time to transfer         | 4+ days @ 1.25 Gbps | 12 hours @ 10 Gbps       |

Direct Connect pays for itself in the first migration.

## Encryption Considerations

> **Important**: Direct Connect is **NOT encrypted by default**.

Options for encryption:

1. **VPN over Direct Connect** - IPsec tunnel over private VIF
2. **MACsec** - Layer 2 encryption (10 Gbps and 100 Gbps dedicated only)
3. **Application-level encryption** - TLS/HTTPS for data in transit

```mermaid
flowchart LR
    subgraph Options["Direct Connect Encryption"]
        OPT1["No Encryption<br>(default)"]
        OPT2["VPN over DX<br>(IPsec)"]
        OPT3["MACsec<br>(Layer 2)"]
    end

    OPT1 -->|"Add"| OPT2
    OPT1 -->|"Add"| OPT3

    style Options fill:#e3f2fd,color:#000
    style OPT1 fill:#ffcdd2,color:#000
    style OPT2 fill:#c8e6c9,color:#000
    style OPT3 fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

## What Could Go Wrong?

Direct Connect is up and running. Latency dropped from 80ms to 15ms. The network team is happy.

But now the database team is getting impatient:

> "We've been running SQL Server in the datacenter for 10 years. The license renewal is next month
> and it's $150,000. Can we move the database to AWS before then?"

Time for database migration.

## Exam Tips

- **Not encrypted by default** - Direct Connect uses private connection but no encryption
- **1/10/100/400 Gbps for dedicated** - Know these speeds
- **Hosted for sub-1 Gbps** - Or when you're not in a DX location
- **Private VIF for VPC access** - Most common VIF type
- **Public VIF for public services** - S3, DynamoDB over DX instead of internet
- **Transit VIF for Transit Gateway** - Connects to TGW for multiple VPCs
- **DX Gateway for multi-region** - One DX connection, multiple regions
- **DX + VPN backup** - Very common pattern, know the BGP priority concept
- **Lead time is weeks/months** - Unlike VPN which is hours

## SAA Exam Concepts

### Must-Know for This Phase

| Concept                | Key Points                                                  |
| ---------------------- | ----------------------------------------------------------- |
| Connection Types       | Dedicated (1/10/100/400 Gbps) vs Hosted (50 Mbps - 25 Gbps) |
| VIF Types              | Private (VPC), Public (AWS services), Transit (TGW)         |
| Direct Connect Gateway | Multi-region/multi-VPC over single DX                       |
| LAG                    | Up to 4 connections, same speed, same location              |
| Resilience             | High = 2 locations, Maximum = 2 connections per location    |
| Encryption             | Not default - use VPN over DX or MACsec                     |

---

## References

Official AWS documentation used to validate this content:

### Direct Connect Connections

- [Dedicated Direct Connect Connections](https://docs.aws.amazon.com/directconnect/latest/UserGuide/dedicated_connection.html) -
  Port speeds: 1 Gbps, 10 Gbps, 100 Gbps, 400 Gbps
- [Hosted Direct Connect Connections](https://docs.aws.amazon.com/directconnect/latest/UserGuide/hosted_connection.html) -
  Port speeds: 50 Mbps to 25 Gbps via AWS Partners
- [Link Aggregation Groups (LAGs)](https://docs.aws.amazon.com/directconnect/latest/UserGuide/lags.html) -
  Aggregate up to 4 connections (1/10 Gbps) or 2 connections (100 Gbps)

### Virtual Interfaces

- [Direct Connect Virtual Interfaces](https://docs.aws.amazon.com/directconnect/latest/UserGuide/WorkingWithVirtualInterfaces.html) -
  Private, Public, and Transit VIF types

### Security

- [MAC Security in Direct Connect](https://docs.aws.amazon.com/directconnect/latest/UserGuide/MACsec.html) -
  Layer 2 encryption for dedicated connections
