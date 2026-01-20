# Tickets Without Crashes: The EventPro Story

## The Business

You're the Lead Architect at **EventPro** - a ticket sales platform for concerts, sporting events,
and theater. The company has grown to 500K users, but faces a critical problem: **flash crowds**.

When 50,000+ fans try to buy tickets simultaneously (like a Taylor Swift concert sale), the system
collapses under the load. This isn't just a technical problem - it's a business crisis.

## The Trigger

> "We sold Taylor Swift tickets last month. 50,000 fans hit 'Buy' at 10:00 AM. Our API crashed in 30
> seconds. We oversold 200 tickets and had to refund furious customers. The CEO is asking why our
> competitors handle this and we can't."
>
> — CTO, post-incident review

## Learning Objectives

This scenario teaches event-driven architecture and asynchronous processing patterns for
high-traffic applications:

- [**Phase 1**: The Crashing Monolith](scenarios/eventpro/phases/phase-1-crashing-monolith.md) -
  Understanding the problem
- [**Phase 2**: SQS Traffic Absorption](scenarios/eventpro/phases/phase-2-sqs-traffic-absorption.md) -
  Queues and decoupling
- [**Phase 3**: API Gateway Protection](scenarios/eventpro/phases/phase-3-api-gateway-protection.md) -
  Rate limiting and caching
- [**Phase 4**: Step Functions Workflows](scenarios/eventpro/phases/phase-4-step-functions-workflows.md) -
  Orchestration and Saga pattern
- [**Phase 5**: EventBridge Routing](scenarios/eventpro/phases/phase-5-eventbridge-routing.md) -
  Event-driven integration
- [**Phase 6**: Event-Driven Observability](scenarios/eventpro/phases/phase-6-observability.md) -
  Tracing and monitoring

## Architecture Evolution Map

```mermaid
flowchart LR
    P1["Phase 1<br>Monolith<br>Problem"] --> P2["Phase 2<br>SQS<br>Queuing"]
    P2 --> P3["Phase 3<br>API GW<br>Protection"]
    P3 --> P4["Phase 4<br>Step Fn<br>Workflows"]
    P4 --> P5["Phase 5<br>EventBridge<br>Routing"]
    P5 --> P6["Phase 6<br>X-Ray<br>Observable"]

    style P1 fill:#ffcdd2,color:#000
    style P2 fill:#fff9c4,color:#000
    style P3 fill:#fff9c4,color:#000
    style P4 fill:#c8e6c9,color:#000
    style P5 fill:#c8e6c9,color:#000
    style P6 fill:#bbdefb,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

## AWS Services Covered

| Phase | Primary Services          | Key Concepts                              |
| ----- | ------------------------- | ----------------------------------------- |
| 1     | -                         | Race conditions, synchronous bottleneck   |
| 2     | SQS, DynamoDB             | Queue decoupling, DLQ, conditional writes |
| 3     | API Gateway, CloudFront   | Throttling, usage plans, caching          |
| 4     | Step Functions, SNS       | Saga pattern, workflow orchestration      |
| 5     | EventBridge               | Event routing, fan-out, scheduling        |
| 6     | X-Ray, CloudWatch, Lambda | Distributed tracing, observability        |

## SAA Exam Domain Mapping

This scenario primarily covers:

- **Domain 2 (Resilient)**: Decoupling, fault tolerance, Saga pattern
- **Domain 3 (High-Performing)**: Traffic absorption, caching, async processing
- **Domain 1 (Secure)**: API protection, rate limiting, DDoS mitigation

## How to Use This Guide

Each phase includes:

1. **Business Context** - What's happening with EventPro
2. **Architecture Decision** - What we're building and WHY
3. **Key Concepts** - SAA exam-relevant knowledge with verified numbers
4. **Diagrams** - Visual representation of the architecture
5. **Cross-References** - Links to related content in other scenarios
6. **Exam Tips** - Key points for the SAA certification
