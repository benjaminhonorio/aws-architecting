# Scale or Fail: The TechBooks Story

## The Business

You're the founding engineer at **TechBooks** - an online bookstore for technical and programming
books. The founder just secured seed funding and needs to launch fast.

## Learning Objectives

This evolving scenario will teach you AWS Solutions Architect concepts through realistic business
decisions:

- [**Phase 1**: MVP Launch](./phases/phase-1-mvp-launch.md) (VPC, EC2, Security Groups)
- [**Phase 2**: First Growth Pains](./phases/phase-2-database-separation.md) (RDS, DB separation,
  backups)
- [**Phase 3**: Going Multi-AZ](./phases/phase-3-high-availability.md) (High Availability, failover)
- [**Phase 4**: Scaling for Success](./phases/phase-4-auto-scaling.md) (ALB, Auto Scaling Groups)
- [**Phase 5**: Going Global](./phases/phase-5-going-global.md) (CloudFront, Route 53, S3)
- [**Phase 6**: Modernization](./phases/phase-6-modernization.md) (Lambda, SQS, ElastiCache)

## Architecture Evolution Map

```mermaid
flowchart LR
    P1["Phase 1<br>Single EC2<br>MVP"] --> P2["Phase 2<br>EC2 + RDS<br>DB Separation"]
    P2 --> P3["Phase 3<br>Multi-AZ<br>HA Setup"]
    P3 --> P4["Phase 4<br>Auto Scaling<br>ALB"]
    P4 --> P5["Phase 5<br>CloudFront<br>Global"]
    P5 --> P6["Phase 6<br>Serverless<br>Modern"]

    style P1 fill:#e1f5fe,color:#000
    style P2 fill:#e1f5fe,color:#000
    style P3 fill:#fff3e0,color:#000
    style P4 fill:#fff3e0,color:#000
    style P5 fill:#e8f5e9,color:#000
    style P6 fill:#e8f5e9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

## How to Use This Guide

Each phase includes:

1. **Business Context** - What's happening with TechBooks
2. **Architecture Decision** - What we're building and WHY
3. **Key Concepts** - SAA exam-relevant knowledge
4. **Diagrams** - Visual representation of the architecture
5. **Exam Tips** - Key points for the SAA certification
