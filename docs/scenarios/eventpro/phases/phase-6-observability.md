# Phase 6: Event-Driven Observability

## Business Context

**Situation:** EventPro is now handling 100K+ concurrent users during major sales. The architecture
is resilient, but a new challenge has emerged: **visibility**.

**The problems:**

- When a purchase fails, support can't trace what happened
- Cold starts cause latency spikes during sudden traffic surges
- The CEO wants a real-time dashboard showing sales velocity and system health
- Debugging requires checking logs across 10+ services manually

**Your decision:** Implement comprehensive observability with AWS X-Ray for distributed tracing,
CloudWatch for metrics and dashboards, and Lambda Provisioned Concurrency for consistent
performance.

---

## The Three Pillars of Observability

```mermaid
flowchart TB
    subgraph Pillars["Observability"]
        Logs["Logs<br>CloudWatch Logs"]
        Metrics["Metrics<br>CloudWatch Metrics"]
        Traces["Traces<br>AWS X-Ray"]
    end

    subgraph Questions["Questions Answered"]
        Q1["What happened?"]
        Q2["How is the system performing?"]
        Q3["Where did time go?"]
    end

    Logs --> Q1
    Metrics --> Q2
    Traces --> Q3

    style Logs fill:#e3f2fd,color:#000
    style Metrics fill:#c8e6c9,color:#000
    style Traces fill:#fff9c4,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

| Pillar  | Tool               | Purpose                               |
| ------- | ------------------ | ------------------------------------- |
| Logs    | CloudWatch Logs    | Detailed event records, debugging     |
| Metrics | CloudWatch Metrics | Quantitative measurements, alerting   |
| Traces  | AWS X-Ray          | Request flow across services, latency |

---

## AWS X-Ray: Distributed Tracing

### What is X-Ray?

**AWS X-Ray** traces requests as they travel through your distributed application. It shows how time
is spent across services, helping you identify bottlenecks and failures.

### X-Ray Concepts

```mermaid
flowchart LR
    subgraph Trace["Trace (one request)"]
        S1["Segment: API Gateway<br>50ms"]
        S2["Segment: Lambda<br>200ms"]
        S3["Segment: DynamoDB<br>30ms"]

        SS1["Subsegment:<br>Payment API<br>150ms"]
    end

    S1 --> S2
    S2 --> SS1
    S2 --> S3

    style S1 fill:#e3f2fd,color:#000
    style S2 fill:#c8e6c9,color:#000
    style S3 fill:#fff9c4,color:#000
    style SS1 fill:#ffecb3,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

| Concept         | Description                            |
| --------------- | -------------------------------------- |
| **Trace**       | End-to-end journey of a single request |
| **Segment**     | Work done by a single service          |
| **Subsegment**  | Detailed breakdown within a segment    |
| **Annotations** | Indexed key-value pairs for filtering  |
| **Metadata**    | Non-indexed data for context           |

### X-Ray Service Map

X-Ray automatically generates a visual map of your architecture:

```mermaid
flowchart LR
    Client((Client)) --> APIGW["API Gateway<br>Avg: 250ms<br>5xx: 0.1%"]
    APIGW --> Lambda["Lambda<br>Avg: 180ms<br>5xx: 0.05%"]
    Lambda --> DDB["DynamoDB<br>Avg: 15ms<br>Errors: 0%"]
    Lambda --> Payment["Payment API<br>Avg: 120ms<br>5xx: 0.5%"]

    style Payment fill:#ffcdd2,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

**WHY this matters for EventPro:**

- See which service is causing latency
- Identify error rates per service
- Trace a specific failed purchase by order ID

> **SAA Exam Tip:** X-Ray sampling controls what percentage of requests are traced. Default: first
> request each second + 5% of additional requests. Adjust for high-volume applications.

---

## Lambda Performance Optimization

### Understanding Cold Starts

A **cold start** occurs when Lambda creates a new execution environment. This adds latency to the
first request.

```mermaid
sequenceDiagram
    participant User
    participant Lambda as Lambda Service
    participant Env as Execution Environment
    participant Code as Your Code

    Note over Lambda,Code: Cold Start (First Request)
    User->>Lambda: Request
    Lambda->>Env: Create environment
    Note over Env: Download code
    Note over Env: Initialize runtime
    Env->>Code: Initialize handler
    Code-->>User: Response (slow: 500-1000ms)

    Note over Lambda,Code: Warm Start (Subsequent)
    User->>Lambda: Request
    Lambda->>Env: Reuse environment
    Env->>Code: Execute handler
    Code-->>User: Response (fast: 50ms)
```

### Lambda Concurrency Types

Lambda has three concurrency settings. Understanding the differences is critical for the exam and
for avoiding throttling during flash sales:

| Type                        | Description                                      | Default        |
| --------------------------- | ------------------------------------------------ | -------------- |
| **Account concurrency**     | Total concurrent executions across all functions | 1,000 (soft)   |
| **Reserved concurrency**    | Guaranteed capacity for a specific function      | 0 (unreserved) |
| **Provisioned concurrency** | Pre-initialized environments (no cold start)     | 0              |

### Reserved vs Provisioned Concurrency

```mermaid
flowchart TB
    subgraph Reserved["Reserved Concurrency"]
        R1["Guarantees capacity"]
        R2["Still has cold starts"]
        R3["Limits function's max"]
        R4["Free"]
    end

    subgraph Provisioned["Provisioned Concurrency"]
        P1["Pre-warmed environments"]
        P2["No cold starts"]
        P3["Always ready"]
        P4["Costs money 24/7"]
    end

    style Reserved fill:#e3f2fd,color:#000
    style Provisioned fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

**For EventPro:**

- **Reserved:** Set on critical functions to prevent other functions from consuming all capacity
- **Provisioned:** Set on the checkout Lambda during scheduled flash sales

### Lambda Quotas

These limits appear frequently on the SAA exam:

| Limit               | Value                    | Notes                     |
| ------------------- | ------------------------ | ------------------------- |
| Account concurrency | 1,000 (default)          | Soft limit, can increase  |
| Memory              | 128 MB - 10,240 MB       | CPU scales with memory    |
| Timeout             | 15 minutes (900 seconds) | Hard limit                |
| Payload (sync)      | 6 MB request/response    | Use S3 for larger         |
| Payload (async)     | 256 KB event             | Use SQS for larger        |
| /tmp storage        | 512 MB - 10,240 MB       | Ephemeral, per invocation |
| Scaling rate        | 1,000 instances/10 sec   | Per function              |

> **Source:**
> [Lambda Quotas](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html)

> **SAA Exam Tip:** "How do you eliminate Lambda cold starts?" → Provisioned Concurrency. "How do
> you ensure a function always has capacity?" → Reserved Concurrency.

---

## CloudWatch Dashboards

### Building an EventPro Operations Dashboard

A well-designed dashboard shows system health at a glance:

```mermaid
flowchart TB
    subgraph Dashboard["EventPro Operations Dashboard"]
        subgraph Business["Business Metrics"]
            M1["Tickets Sold/min"]
            M2["Revenue/hour"]
            M3["Conversion Rate"]
        end

        subgraph Technical["Technical Metrics"]
            M4["API Latency p99"]
            M5["Error Rate %"]
            M6["Queue Depth"]
        end

        subgraph Capacity["Capacity"]
            M7["Lambda Concurrency"]
            M8["DynamoDB Consumed WCU"]
            M9["SQS Messages in Flight"]
        end
    end

    style Business fill:#c8e6c9,color:#000
    style Technical fill:#fff9c4,color:#000
    style Capacity fill:#e3f2fd,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### Key Metrics to Monitor

| Service     | Metric                        | Alert Threshold        |
| ----------- | ----------------------------- | ---------------------- |
| API Gateway | 5XXError                      | > 1% for 5 minutes     |
| API Gateway | Latency (p99)                 | > 3 seconds            |
| Lambda      | ConcurrentExecutions          | > 80% of limit         |
| Lambda      | Errors                        | > 0 for 5 minutes      |
| Lambda      | Duration (p99)                | > 80% of timeout       |
| SQS         | ApproximateNumberOfMessages   | > 10,000 (building up) |
| SQS         | ApproximateAgeOfOldestMessage | > 300 seconds          |
| DynamoDB    | ConsumedWriteCapacityUnits    | > 80% of provisioned   |
| DynamoDB    | ThrottledRequests             | > 0                    |

### CloudWatch Alarms

```mermaid
flowchart LR
    Metric["CloudWatch Metric<br>5XXError"] --> Alarm["CloudWatch Alarm<br>> 1% for 5 min"]
    Alarm --> SNS["SNS Topic"]
    SNS --> Email["Email Alert"]
    SNS --> Slack["Slack Channel"]
    SNS --> Lambda["Auto-Remediation<br>Lambda"]

    style Alarm fill:#ffcdd2,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

---

## CloudWatch Logs Insights

### Querying Logs

CloudWatch Logs Insights allows SQL-like queries across log groups:

**Find slow Lambda executions:**

```sql
fields @timestamp, @requestId, @duration
| filter @duration > 1000
| sort @duration desc
| limit 20
```

**Count errors by type:**

```sql
fields @message
| filter @message like /ERROR/
| stats count(*) by errorType
```

**Trace a specific order:**

```sql
fields @timestamp, @message
| filter orderId = "ORD-12345"
| sort @timestamp asc
```

> **SAA Exam Tip:** CloudWatch Logs Insights charges per GB scanned. Use time filters to reduce
> costs.

---

## Phase 6 Complete Architecture

```mermaid
flowchart TB
    subgraph Users["Users"]
        User((User))
    end

    subgraph Observability["Observability Layer"]
        XRay["X-Ray<br>Distributed Tracing"]
        CW["CloudWatch<br>Logs + Metrics"]
        Dashboard["Operations<br>Dashboard"]
        Alarms["CloudWatch<br>Alarms"]
    end

    subgraph Application["Application (Instrumented)"]
        CF["CloudFront"]
        APIGW["API Gateway"]
        Lambda["Lambda<br>Provisioned Concurrency"]
        SF["Step Functions"]
        SQS["SQS"]
        EB["EventBridge"]
        DDB["DynamoDB"]
    end

    subgraph Alerts["Alerting"]
        SNS["SNS"]
        Slack["Slack"]
        PagerDuty["PagerDuty"]
    end

    User --> CF --> APIGW --> Lambda
    Lambda --> SF --> SQS
    SF --> EB
    Lambda --> DDB

    APIGW -.-> XRay
    Lambda -.-> XRay
    SF -.-> XRay
    DDB -.-> XRay

    Lambda -.-> CW
    APIGW -.-> CW
    CW --> Dashboard
    CW --> Alarms
    Alarms --> SNS
    SNS --> Slack
    SNS --> PagerDuty

    style XRay fill:#fff9c4,color:#000
    style Dashboard fill:#c8e6c9,color:#000
    style Alarms fill:#ffcdd2,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

---

## EventPro: Final Architecture Summary

After all 6 phases, EventPro has evolved from a crashing monolith to a resilient, observable,
event-driven architecture:

```mermaid
flowchart TB
    subgraph Edge["Edge Layer"]
        CF["CloudFront<br>Global CDN + DDoS"]
    end

    subgraph API["API Layer"]
        APIGW["API Gateway<br>Rate Limiting + Caching"]
    end

    subgraph Compute["Compute Layer"]
        Lambda["Lambda<br>Provisioned Concurrency"]
    end

    subgraph Workflow["Orchestration"]
        SF["Step Functions<br>Saga Pattern"]
    end

    subgraph Messaging["Messaging Layer"]
        SQS["SQS FIFO<br>Traffic Absorption"]
        SNS["SNS<br>Notifications"]
        EB["EventBridge<br>Event Routing"]
    end

    subgraph Data["Data Layer"]
        DDB["DynamoDB<br>Inventory + Orders"]
        S3["S3<br>Tickets"]
    end

    subgraph Observability["Observability"]
        XRay["X-Ray"]
        CW["CloudWatch"]
    end

    CF --> APIGW --> Lambda
    Lambda --> SQS
    SQS --> Lambda
    Lambda --> SF
    SF --> DDB
    SF --> SNS
    SF --> EB
    Lambda --> S3

    Lambda -.-> XRay
    Lambda -.-> CW

    style Edge fill:#e3f2fd,color:#000
    style API fill:#fff9c4,color:#000
    style Messaging fill:#c8e6c9,color:#000
    style Data fill:#bbdefb,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

---

## Key SAA Exam Concepts from Phase 6

### Must-Know Topics

1. **X-Ray Components**
   - Traces, segments, subsegments
   - Annotations (indexed) vs metadata (not indexed)
   - Service map for visualization
   - Sampling to control costs

2. **Lambda Concurrency**
   - Account concurrency: Total across all functions (default 1,000)
   - Reserved: Guarantees capacity, has cold starts
   - Provisioned: Pre-warmed, no cold starts, costs money

3. **Lambda Limits**
   - Timeout: 15 minutes maximum
   - Memory: 128 MB - 10,240 MB
   - Payload sync: 6 MB
   - Scaling: 1,000 instances per 10 seconds

4. **CloudWatch**
   - Metrics: Quantitative data points
   - Logs: Text records
   - Alarms: Threshold-based notifications
   - Dashboards: Visualization

5. **Logs Insights**
   - SQL-like query language
   - Query across log groups
   - Charged per GB scanned

---

## See Also

> **Related Learning:**
>
> - For CloudWatch and CloudTrail in security context, see
>   [MedVault Phase 4: Logging & Monitoring](/scenarios/medvault/phases/phase-4-logging-monitoring.md)
> - For Lambda patterns, see
>   [TechBooks Phase 6: Modernization](/scenarios/techbooks/phases/phase-6-modernization.md)

---

## Scenario Complete!

Congratulations! You've transformed EventPro from a crashing monolith to a scalable, resilient,
event-driven architecture capable of handling 100K+ concurrent users.

### What You've Learned

| Phase | Topic                  | Key Services              |
| ----- | ---------------------- | ------------------------- |
| 1     | Problem identification | -                         |
| 2     | Queue-based decoupling | SQS, DynamoDB             |
| 3     | API protection         | API Gateway, CloudFront   |
| 4     | Workflow orchestration | Step Functions, SNS       |
| 5     | Event routing          | EventBridge, Scheduler    |
| 6     | Observability          | X-Ray, CloudWatch, Lambda |

### SAA Exam Domains Covered

- **Domain 1 (Secure):** API protection, rate limiting, DDoS mitigation
- **Domain 2 (Resilient):** Decoupling, Saga pattern, fault tolerance
- **Domain 3 (High-Performing):** Async processing, caching, provisioned concurrency
- **Domain 4 (Cost-Optimized):** Serverless, on-demand scaling, right-sizing

---

## References

### AWS Documentation

- [Lambda Quotas](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html)
- [Lambda Concurrency](https://docs.aws.amazon.com/lambda/latest/dg/configuration-concurrency.html)
- [Lambda Provisioned Concurrency](https://docs.aws.amazon.com/lambda/latest/dg/provisioned-concurrency.html)
- [X-Ray Concepts](https://docs.aws.amazon.com/xray/latest/devguide/xray-concepts.html)
- [CloudWatch Logs Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AnalyzingLogData.html)

### Architecture Patterns

- [Observability Best Practices](https://docs.aws.amazon.com/prescriptive-guidance/latest/implementing-logging-monitoring-cloudwatch/introduction.html)
