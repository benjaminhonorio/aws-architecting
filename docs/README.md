# AWS Architecture Learning Scenarios

A collection of hands-on learning scenarios for AWS Solutions Architect certification preparation.

## Available Scenarios

### [TechBooks: Scale or Fail](scenarios/techbooks/00-overview.md)

Learn AWS architecture through the journey of an online bookstore that evolves from a simple MVP to
a modern, globally-distributed application.

**Covers**: VPC, EC2, RDS, Multi-AZ, Auto Scaling, ALB, CloudFront, Route 53, Lambda, SQS,
ElastiCache

```mermaid
flowchart LR
    P1["Phase 1<br>MVP"] --> P2["Phase 2<br>DB Sep"]
    P2 --> P3["Phase 3<br>HA"]
    P3 --> P4["Phase 4<br>Scaling"]
    P4 --> P5["Phase 5<br>Global"]
    P5 --> P6["Phase 6<br>Modern"]

    style P1 fill:#e1f5fe,color:#000
    style P2 fill:#e1f5fe,color:#000
    style P3 fill:#fff3e0,color:#000
    style P4 fill:#fff3e0,color:#000
    style P5 fill:#e8f5e9,color:#000
    style P6 fill:#e8f5e9,color:#000
```

---

### [ShipFast Logistics: Migrate or Die](scenarios/shipfast/00-overview.md)

Learn AWS migration and hybrid architecture through the journey of a logistics company migrating
from an on-premises datacenter to AWS.

**Covers**: Site-to-Site VPN, Direct Connect, DMS, SCT, Storage Gateway, FSx, DataSync, MGN, Snow
Family

```mermaid
flowchart LR
    P1["Phase 1<br>On-Prem"] --> P2["Phase 2<br>VPN"]
    P2 --> P3["Phase 3<br>DX"]
    P3 --> P4["Phase 4<br>DB Mig"]
    P4 --> P5["Phase 5<br>Storage"]
    P5 --> P6["Phase 6<br>Workload"]

    style P1 fill:#ffcdd2,color:#000
    style P2 fill:#fff9c4,color:#000
    style P3 fill:#fff9c4,color:#000
    style P4 fill:#c8e6c9,color:#000
    style P5 fill:#c8e6c9,color:#000
    style P6 fill:#bbdefb,color:#000
```

---

### [MedVault: Secure by Design](scenarios/medvault/00-overview.md)

Learn AWS security and compliance through the journey of a healthcare startup building a
HIPAA-compliant patient records platform.

**Covers**: IAM, KMS, Secrets Manager, VPC Endpoints, WAF, Shield, CloudTrail, Config,
Organizations, SCPs, GuardDuty, Security Hub, Macie

```mermaid
flowchart LR
    P1["Phase 1<br>Identity"] --> P2["Phase 2<br>Encryption"]
    P2 --> P3["Phase 3<br>Network"]
    P3 --> P4["Phase 4<br>Logging"]
    P4 --> P5["Phase 5<br>Multi-Acct"]
    P5 --> P6["Phase 6<br>Detection"]

    style P1 fill:#e8eaf6,color:#000
    style P2 fill:#e8eaf6,color:#000
    style P3 fill:#fff3e0,color:#000
    style P4 fill:#fff3e0,color:#000
    style P5 fill:#e8f5e9,color:#000
    style P6 fill:#ffebee,color:#000
```

---

### [EventPro: Tickets Without Crashes](scenarios/eventpro/00-overview.md)

Learn event-driven architecture through the journey of a ticket sales platform that evolves from a
crashing monolith to handling 100K+ concurrent users during flash sales.

**Covers**: SQS, DynamoDB, API Gateway, CloudFront, Step Functions, SNS, EventBridge, X-Ray,
CloudWatch, Lambda

```mermaid
flowchart LR
    P1["Phase 1<br>Monolith"] --> P2["Phase 2<br>SQS"]
    P2 --> P3["Phase 3<br>API GW"]
    P3 --> P4["Phase 4<br>Step Fn"]
    P4 --> P5["Phase 5<br>Events"]
    P5 --> P6["Phase 6<br>Observe"]

    style P1 fill:#ffcdd2,color:#000
    style P2 fill:#fff9c4,color:#000
    style P3 fill:#fff9c4,color:#000
    style P4 fill:#c8e6c9,color:#000
    style P5 fill:#c8e6c9,color:#000
    style P6 fill:#bbdefb,color:#000
```

---

### [DataLake Corp: Data at Scale](scenarios/datalake/00-overview.md)

Learn AWS analytics and data services through the journey of a retail analytics company building a
modern cloud data platform from Excel reports to real-time fraud detection.

**Covers**: S3 Data Lake, Athena, Glue, Redshift, Redshift Spectrum, AQUA, Kinesis, Lake Formation,
ElastiCache, Step Functions

```mermaid
flowchart LR
    P1["Phase 1<br>S3 Lake"] --> P2["Phase 2<br>Athena"]
    P2 --> P3["Phase 3<br>Glue ETL"]
    P3 --> P4["Phase 4<br>Redshift"]
    P4 --> P5["Phase 5<br>Kinesis"]
    P5 --> P6["Phase 6<br>Governance"]

    style P1 fill:#e3f2fd,color:#000
    style P2 fill:#e3f2fd,color:#000
    style P3 fill:#fff3e0,color:#000
    style P4 fill:#fff3e0,color:#000
    style P5 fill:#e8f5e9,color:#000
    style P6 fill:#e8f5e9,color:#000
```
