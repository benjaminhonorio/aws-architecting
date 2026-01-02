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

_More scenarios coming soon..._
