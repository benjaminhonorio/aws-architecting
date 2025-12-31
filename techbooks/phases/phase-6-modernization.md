# Phase 6: Modernization

## Business Context

**Situation:** TechBooks is now profitable! Monthly revenue has grown 5x since launch. The founder wants to reduce AWS costs and add new features. The development team has grown from 1 to 5 engineers.

**New feature requests:**
- User book reviews with image uploads
- Email notifications (order confirmation, shipping updates)
- "Customers also bought" recommendations
- Search autocomplete

**Problems identified:**
- EC2 instances run 24/7 but traffic is spiky (80% idle at night)
- Database queries are slow during traffic spikes
- Session data lost when instances scale down
- Monolithic app is hard to deploy quickly

**The founder asks:** "Can we reduce costs AND ship features faster?"

**Your decision:** Adopt serverless components, add caching, and decouple the architecture.

---

## Step 1: Why Modernize?

### The Current Architecture's Limitations

```mermaid
flowchart TB
    subgraph Current["Current: Monolithic"]
        EC2["EC2 Instances<br>- Web server<br>- Image processing<br>- Email sending<br>- Search<br>- Session storage"]

        Problem1["Problem: Tightly coupled<br>One change = full deploy"]
        Problem2["Problem: Always running<br>Pay for idle capacity"]
        Problem3["Problem: Single technology<br>Everything in one codebase"]
    end

    style Current fill:#ffcdd2,color:#000
    style Problem1 fill:#ffcdd2,color:#000
    style Problem2 fill:#ffcdd2,color:#000
    style Problem3 fill:#ffcdd2,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### The Modernization Approach

We won't rewrite everything. We'll **extract specific functions** to serverless while keeping the core application on EC2.

```mermaid
flowchart TB
    subgraph Modern["Modernized: Hybrid"]
        EC2m["EC2 Instances<br>Core web app only"]
        Lambda["Lambda<br>Image processing<br>Email notifications"]
        Cache["ElastiCache<br>Sessions + DB cache"]
        SQS["SQS<br>Async processing"]
    end

    Benefit1["Benefit: Deploy independently"]
    Benefit2["Benefit: Pay per use"]
    Benefit3["Benefit: Right tool for job"]

    style Modern fill:#c8e6c9,color:#000
    style Benefit1 fill:#c8e6c9,color:#000
    style Benefit2 fill:#c8e6c9,color:#000
    style Benefit3 fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

> **SAA Exam Tip:** Modernization doesn't mean "rewrite in serverless." It means using the right service for each workload. Keep steady-state workloads on EC2, move event-driven workloads to Lambda.

---

## Step 2: AWS Lambda Fundamentals

### What is Lambda?

**AWS Lambda** runs code without provisioning servers. You pay only when code executes.

```mermaid
flowchart LR
    subgraph Traditional["Traditional: EC2"]
        EC2a["EC2 Always Running"]
        Cost1["Pay: 24/7/365"]
        Scale1["Scale: Minutes"]
    end

    subgraph Serverless["Serverless: Lambda"]
        Lambda["Lambda Function"]
        Cost2["Pay: Per invocation"]
        Scale2["Scale: Milliseconds"]
    end

    style Traditional fill:#fff9c4,color:#000
    style Serverless fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### Lambda Key Concepts

| Concept | Description | Limit |
|---------|-------------|-------|
| **Function** | Your code + configuration | N/A |
| **Runtime** | Language (Python, Node.js, etc.) | 12+ supported |
| **Handler** | Entry point function | N/A |
| **Memory** | Allocated RAM (CPU scales with it) | 128 MB - 10 GB |
| **Timeout** | Maximum execution time | 15 minutes |
| **Concurrency** | Parallel executions | 1000 default (soft) |

### Lambda Execution Model

```mermaid
sequenceDiagram
    participant Trigger as Event Trigger
    participant Lambda as Lambda Service
    participant Function as Your Function

    Note over Lambda: Cold Start (first invocation)
    Trigger->>Lambda: Invoke function
    Lambda->>Lambda: Download code
    Lambda->>Lambda: Start runtime
    Lambda->>Function: Initialize handler
    Function-->>Lambda: Response
    Lambda-->>Trigger: Result

    Note over Lambda: Warm Start (subsequent)
    Trigger->>Lambda: Invoke function
    Lambda->>Function: Call handler directly
    Function-->>Lambda: Response
    Lambda-->>Trigger: Result (faster!)
```

### Cold Starts

| Factor | Impact on Cold Start |
|--------|---------------------|
| **Memory size** | More memory = faster init |
| **Code size** | Smaller = faster download |
| **Runtime** | Python/Node.js faster than Java |
| **VPC** | VPC adds 1-2 seconds (improved but still slower) |

> **SAA Exam Tip:** Lambda in VPC used to have significant cold start penalties. AWS improved this with Hyperplane ENIs, but there's still some overhead. Only put Lambda in VPC if it needs to access VPC resources.

---

## Step 3: Lambda Use Cases for TechBooks

### 1. Image Processing for Book Reviews

Users upload review images. We need to resize them for display.

```mermaid
flowchart LR
    User((User)) -->|"Upload image"| S3["S3 Bucket<br>reviews-uploads"]
    S3 -->|"Event trigger"| Lambda["Lambda<br>resize-image"]
    Lambda -->|"Write resized"| S3out["S3 Bucket<br>reviews-processed"]
    Lambda -->|"Update DB"| RDS[("RDS")]

    style Lambda fill:#fff9c4,color:#000
    style S3 fill:#c8e6c9,color:#000
    style S3out fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

**WHY Lambda for this?**
- Event-driven (triggered by upload)
- Variable traffic (depends on reviews)
- Stateless (each image is independent)
- Short-lived (seconds to process)

### 2. Email Notifications

Send order confirmations and shipping updates.

```mermaid
flowchart LR
    App["EC2 App"] -->|"Publish event"| SQS["SQS Queue<br>email-requests"]
    SQS -->|"Trigger"| Lambda["Lambda<br>send-email"]
    Lambda -->|"Send"| SES["Amazon SES"]
    SES -->|"Email"| User((Customer))

    style SQS fill:#fff9c4,color:#000
    style Lambda fill:#fff9c4,color:#000
    style SES fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

**WHY Lambda + SQS?**
- Decoupled (app doesn't wait for email to send)
- Retries built-in (SQS retries on failure)
- Scalable (handles Black Friday spike)
- Cost-effective (no idle servers)

### 3. Search Index Updates

When product data changes, update search index.

```mermaid
flowchart LR
    RDS[("RDS")] -->|"Change event"| DynamoDB["DynamoDB Streams<br>(or SNS)"]
    DynamoDB -->|"Trigger"| Lambda["Lambda<br>update-search"]
    Lambda -->|"Update"| OpenSearch["OpenSearch<br>(or Algolia)"]

    style DynamoDB fill:#fff9c4,color:#000
    style Lambda fill:#fff9c4,color:#000
    style OpenSearch fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### Lambda vs EC2 Decision Framework

| Characteristic | Use Lambda | Use EC2 |
|----------------|-----------|---------|
| Execution time | < 15 minutes | > 15 minutes |
| Traffic pattern | Spiky, unpredictable | Steady, predictable |
| State | Stateless | Stateful |
| Startup time | Tolerant of cold starts | Needs instant response |
| Cost at high volume | Expensive | Cheaper |

> **SAA Exam Tip:** Lambda is cost-effective for variable workloads. For steady, high-volume workloads (millions of requests/hour), EC2 or Fargate may be cheaper.

---

## Step 4: Lambda in VPC

### WHY Put Lambda in VPC?

Some Lambdas need to access VPC resources:

```mermaid
flowchart TB
    subgraph VPC["VPC"]
        subgraph Private["Private Subnet"]
            RDS[("RDS")]
            ElastiCache["ElastiCache"]
        end

        subgraph LambdaVPC["Lambda in VPC"]
            Lambda1["Lambda<br>Needs RDS access"]
        end

        Lambda1 --> RDS
        Lambda1 --> ElastiCache
    end

    subgraph Outside["Outside VPC"]
        Lambda2["Lambda<br>No VPC resources needed"]
        S3["S3"]
        SES["SES"]
    end

    Lambda2 --> S3
    Lambda2 --> SES

    style LambdaVPC fill:#fff9c4,color:#000
    style Private fill:#fff9c4,color:#000
    style Outside fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### Lambda VPC Configuration

| Setting | Purpose |
|---------|---------|
| **Subnets** | Lambda runs ENIs in these subnets |
| **Security Groups** | Controls Lambda's network access |
| **NAT Gateway** | Required for internet access from private subnet |

### Internet Access from VPC Lambda

```mermaid
flowchart LR
    subgraph VPC["VPC"]
        subgraph Private["Private Subnet"]
            Lambda["Lambda"]
        end
        subgraph Public["Public Subnet"]
            NAT["NAT Gateway"]
        end
    end

    Lambda -->|"Route table"| NAT
    NAT --> IGW["Internet Gateway"]
    IGW --> Internet((Internet))

    style Private fill:#fff9c4,color:#000
    style Public fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

> **SAA Exam Tip:** Lambda in VPC needs NAT Gateway for internet access. This adds cost (~$32/month per NAT Gateway). Consider VPC endpoints for AWS services instead.

---

## Step 5: Amazon SQS for Decoupling

### What is SQS?

**Simple Queue Service (SQS)** is a fully managed message queue. It decouples components so they can fail independently.

```mermaid
flowchart LR
    subgraph Coupled["Tightly Coupled"]
        App1["App"] -->|"Direct call"| Service1["Email Service"]
        Problem["If email fails,<br>order fails"]
    end

    subgraph Decoupled["Decoupled with SQS"]
        App2["App"] -->|"Send message"| SQS["SQS Queue"]
        SQS -->|"Pull message"| Service2["Email Service"]
        Benefit["Email can fail,<br>retry later"]
    end

    style Coupled fill:#ffcdd2,color:#000
    style Problem fill:#ffcdd2,color:#000
    style Decoupled fill:#c8e6c9,color:#000
    style Benefit fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### SQS Queue Types

| Type | Use Case | Ordering | Throughput |
|------|----------|----------|------------|
| **Standard** | High throughput, at-least-once | Best effort | Unlimited |
| **FIFO** | Strict order required | Guaranteed | 3000 msg/sec (with batching) |

### SQS Key Concepts

| Concept | Description | Default |
|---------|-------------|---------|
| **Visibility Timeout** | Time message is hidden after read | 30 seconds |
| **Retention Period** | How long messages are kept | 4 days (max 14) |
| **Dead Letter Queue** | Queue for failed messages | Optional |
| **Long Polling** | Wait for messages (reduces API calls) | 0 seconds |

### TechBooks Email Flow with SQS

```mermaid
flowchart TB
    subgraph Producer["Producer: EC2 App"]
        Order["Order placed"]
        Send["Send message to SQS"]
    end

    subgraph Queue["SQS Queue"]
        Message["Message:<br>{orderId, email, type}"]
        DLQ["Dead Letter Queue<br>After 3 failures"]
    end

    subgraph Consumer["Consumer: Lambda"]
        Lambda["Lambda Function"]
        SES["Amazon SES"]
    end

    Order --> Send
    Send --> Message
    Message -->|"Success"| Lambda
    Message -->|"Failure x3"| DLQ
    Lambda --> SES

    style Queue fill:#fff9c4,color:#000
    style DLQ fill:#ffcdd2,color:#000
    style Consumer fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

> **SAA Exam Tip:** Always configure Dead Letter Queues for SQS. They capture failed messages for debugging. Set `maxReceiveCount` to define retry attempts before moving to DLQ.

---

## Step 6: Amazon ElastiCache

### What is ElastiCache?

**ElastiCache** is a managed in-memory cache. It supports Redis and Memcached.

```mermaid
flowchart LR
    subgraph Without["Without Cache"]
        App1["App"] -->|"Every request"| DB1[("RDS<br>~10ms")]
    end

    subgraph With["With Cache"]
        App2["App"] -->|"Check cache"| Cache["ElastiCache<br>~1ms"]
        Cache -->|"Cache miss"| DB2[("RDS")]
    end

    style Without fill:#fff9c4,color:#000
    style With fill:#c8e6c9,color:#000
    style Cache fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### Redis vs Memcached

| Feature | Redis | Memcached |
|---------|-------|-----------|
| **Data structures** | Strings, lists, sets, hashes | Strings only |
| **Persistence** | Yes (snapshots, AOF) | No |
| **Replication** | Yes (read replicas) | No |
| **Multi-AZ** | Yes (automatic failover) | No |
| **Pub/Sub** | Yes | No |
| **Use case** | Complex caching, sessions | Simple caching |

### TechBooks Caching Strategy

```mermaid
flowchart TB
    App["EC2 App"]

    subgraph Cache["ElastiCache Redis"]
        Sessions["Sessions<br>Key: session_id<br>TTL: 30 min"]
        DBCache["DB Query Cache<br>Key: query_hash<br>TTL: 5 min"]
        Catalog["Book Catalog<br>Key: book_id<br>TTL: 1 hour"]
    end

    subgraph Backend["Backend"]
        RDS[("RDS")]
    end

    App -->|"1. Check cache"| Cache
    Cache -->|"2. Miss? Query DB"| RDS
    RDS -->|"3. Populate cache"| Cache

    style Cache fill:#c8e6c9,color:#000
    style Sessions fill:#c8e6c9,color:#000
    style DBCache fill:#c8e6c9,color:#000
    style Catalog fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### Caching Patterns

#### Cache-Aside (Lazy Loading)

```mermaid
sequenceDiagram
    participant App
    participant Cache as ElastiCache
    participant DB as RDS

    App->>Cache: GET book:123
    Cache-->>App: MISS (null)
    App->>DB: SELECT * FROM books WHERE id=123
    DB-->>App: Book data
    App->>Cache: SET book:123 (with TTL)
    App-->>App: Return book data
```

#### Write-Through

```mermaid
sequenceDiagram
    participant App
    participant Cache as ElastiCache
    participant DB as RDS

    App->>DB: UPDATE books SET price=29.99
    DB-->>App: Success
    App->>Cache: SET book:123 (updated data)
    Cache-->>App: Success
```

### Session Storage with ElastiCache

**Problem:** When ASG scales down, sessions on terminated instances are lost.

```mermaid
flowchart TB
    subgraph Problem["Before: Sessions on EC2"]
        User1((User)) --> ALB1["ALB"]
        ALB1 --> EC2a1["EC2-A<br>Session: user1"]
        ALB1 --> EC2b1["EC2-B<br>Session: user2"]
        Terminate["EC2-B terminates<br>User2 logged out!"]
    end

    subgraph Solution["After: Sessions in Redis"]
        User2((User)) --> ALB2["ALB"]
        ALB2 --> EC2a2["EC2-A<br>No local sessions"]
        ALB2 --> EC2b2["EC2-B<br>No local sessions"]
        EC2a2 --> Redis["ElastiCache<br>All sessions"]
        EC2b2 --> Redis
    end

    style Problem fill:#ffcdd2,color:#000
    style Terminate fill:#ffcdd2,color:#000
    style Solution fill:#c8e6c9,color:#000
    style Redis fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

> **SAA Exam Tip:** For stateless applications with Auto Scaling, externalize session storage to ElastiCache Redis or DynamoDB. This enables seamless scale-in/out.

---

## Step 7: ElastiCache Architecture

### ElastiCache in VPC

```mermaid
flowchart TB
    subgraph VPC["VPC: 10.0.0.0/16"]
        subgraph PublicSubnets["Public Subnets"]
            ALB["ALB"]
        end

        subgraph PrivateApp["Private App Subnets"]
            EC2a["EC2"]
            EC2b["EC2"]
        end

        subgraph PrivateCache["Private Cache Subnets"]
            Redis1["Redis Primary<br>cache.t3.micro"]
            Redis2["Redis Replica<br>cache.t3.micro"]
        end

        subgraph PrivateDB["Private DB Subnets"]
            RDS[("RDS Primary")]
            RDSStandby[("RDS Standby")]
        end

        ALB --> EC2a
        ALB --> EC2b
        EC2a --> Redis1
        EC2b --> Redis1
        Redis1 --> Redis2
        EC2a --> RDS
        EC2b --> RDS
    end

    style PrivateCache fill:#c8e6c9,color:#000
    style Redis1 fill:#c8e6c9,color:#000
    style Redis2 fill:#c8e6c9,color:#000
    style PrivateDB fill:#fff9c4,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### ElastiCache Security Group

```mermaid
flowchart LR
    subgraph SG["Security Groups"]
        SGApp["sg-techbooks-app<br>EC2 instances"]
        SGCache["sg-techbooks-cache<br>ElastiCache"]
    end

    SGApp -->|"Outbound: 6379"| SGCache
    SGCache -->|"Inbound: 6379<br>from sg-techbooks-app"| SGApp

    style SGCache fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

| Security Group | Direction | Port | Source/Dest | Purpose |
|----------------|-----------|------|-------------|---------|
| sg-techbooks-cache | Inbound | 6379 | sg-techbooks-app | Redis from EC2 |

> **SAA Exam Tip:** ElastiCache is VPC-only. It has no public endpoints. Always reference Security Groups instead of IP addresses for dynamic environments.

---

## Step 8: S3 for User Uploads

### Upload Flow Design

Users upload book review images. We need secure, scalable uploads.

```mermaid
flowchart TB
    subgraph Upload["Direct S3 Upload with Presigned URL"]
        User((User)) -->|"1. Request upload URL"| App["EC2 App"]
        App -->|"2. Generate presigned URL"| S3["S3"]
        App -->|"3. Return URL"| User
        User -->|"4. Upload directly"| S3
        S3 -->|"5. Trigger"| Lambda["Lambda<br>Process image"]
        Lambda -->|"6. Save metadata"| RDS[("RDS")]
    end

    style S3 fill:#c8e6c9,color:#000
    style Lambda fill:#fff9c4,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### WHY Presigned URLs?

| Direct to EC2 | Presigned URL to S3 |
|---------------|---------------------|
| EC2 handles upload | S3 handles upload |
| Limited bandwidth | Unlimited bandwidth |
| Pay for EC2 time | Pay only S3 storage |
| Complex to scale | Auto-scales |

### Presigned URL Flow

```mermaid
sequenceDiagram
    participant User
    participant App as EC2 App
    participant S3

    User->>App: POST /api/reviews/upload-url
    Note over App: Generate presigned URL<br>Expires in 5 minutes
    App-->>User: {uploadUrl, fields}
    User->>S3: PUT image (presigned URL)
    Note over S3: Validate signature<br>Accept upload
    S3-->>User: 200 OK
    Note over S3: Trigger Lambda<br>via S3 Event
```

### S3 Bucket Configuration

| Setting | Value | Purpose |
|---------|-------|---------|
| **Bucket name** | techbooks-user-uploads | User-uploaded content |
| **Versioning** | Enabled | Recovery from accidental deletes |
| **Encryption** | SSE-S3 | Encrypt at rest |
| **Lifecycle** | Move to IA after 90 days | Cost optimization |
| **CORS** | Allow *.techbooks.com | Browser uploads |

> **SAA Exam Tip:** Presigned URLs allow secure, temporary access to private S3 objects. They can be for upload (PUT) or download (GET). Expiration time is configurable.

---

## Step 9: Event-Driven Architecture

### Putting It Together

```mermaid
flowchart TB
    User((User))

    subgraph Sync["Synchronous: User-Facing"]
        CloudFront["CloudFront"]
        ALB["ALB"]
        EC2["EC2 App"]
        ElastiCache["ElastiCache"]
        RDS[("RDS")]
    end

    subgraph Async["Asynchronous: Background"]
        S3["S3 Uploads"]
        SQS["SQS Queues"]
        Lambda1["Lambda<br>Image Resize"]
        Lambda2["Lambda<br>Send Email"]
        SES["SES"]
    end

    User --> CloudFront --> ALB --> EC2
    EC2 <--> ElastiCache
    EC2 <--> RDS
    EC2 -->|"Presigned URL"| S3
    S3 -->|"S3 Event"| Lambda1
    EC2 -->|"Send message"| SQS
    SQS -->|"Trigger"| Lambda2
    Lambda2 --> SES

    style Sync fill:#e3f2fd,color:#000
    style Async fill:#fff9c4,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### Benefits of Event-Driven

| Benefit | How |
|---------|-----|
| **Resilience** | Failed email doesn't break checkout |
| **Scalability** | Lambda auto-scales with queue depth |
| **Cost** | Pay only when processing |
| **Speed** | User doesn't wait for background tasks |

---

## Step 10: Cost Optimization

### Current Spending Analysis

After Phase 5, TechBooks spends approximately:

| Component | Monthly Cost | Usage Pattern |
|-----------|--------------|---------------|
| EC2 (t3.small x2) | ~$30 | On-Demand, 24/7 |
| RDS Multi-AZ | ~$50 | On-Demand, 24/7 |
| ALB | ~$20 | 24/7 |
| CloudFront | ~$15 | Variable |
| S3 | ~$5 | Variable |
| NAT Gateway | ~$35 | 24/7 |
| **Total** | ~$155 | |

### Savings Opportunities

```mermaid
flowchart TB
    subgraph Savings["Cost Optimization Options"]
        RI["Reserved Instances<br>1-3 year commitment<br>Up to 72% savings"]
        SP["Savings Plans<br>Flexible commitment<br>Up to 66% savings"]
        Spot["Spot Instances<br>For non-critical workloads<br>Up to 90% savings"]
        Right["Right-sizing<br>Match instance to workload<br>Variable savings"]
    end

    style RI fill:#c8e6c9,color:#000
    style SP fill:#c8e6c9,color:#000
    style Spot fill:#fff9c4,color:#000
    style Right fill:#fff9c4,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### Reserved Instances vs Savings Plans

| Feature | Reserved Instances | Savings Plans |
|---------|-------------------|---------------|
| **Commitment** | Specific instance type | $/hour spend |
| **Flexibility** | Limited (RI exchange) | High (any instance) |
| **Services** | EC2, RDS, ElastiCache | EC2, Fargate, Lambda |
| **Discount** | Up to 72% | Up to 66% |
| **Best for** | Predictable workloads | Changing workloads |

### TechBooks Optimization Plan

| Component | Current | Optimized | Savings |
|-----------|---------|-----------|---------|
| EC2 | On-Demand $30 | 1yr RI $15 | 50% |
| RDS | On-Demand $50 | 1yr RI $30 | 40% |
| ElastiCache | On-Demand $25 | 1yr RI $15 | 40% |
| **Compute Total** | $105 | $60 | ~$45/month |

### Additional Optimizations

| Optimization | Potential Savings |
|--------------|-------------------|
| **S3 Intelligent-Tiering** | Auto-moves cold data |
| **S3 Lifecycle policies** | Delete old versions |
| **CloudFront caching** | Reduce origin requests |
| **Lambda right-sizing** | Match memory to needs |
| **NAT Gateway review** | Use VPC endpoints |

> **SAA Exam Tip:** For steady-state workloads, Reserved Instances provide the best savings. For flexible/changing workloads, use Savings Plans. Spot Instances are best for fault-tolerant, stateless workloads.

---

## Step 11: VPC Endpoints (Cost Optimization)

### The NAT Gateway Cost Problem

```mermaid
flowchart LR
    subgraph VPC["VPC"]
        subgraph Private["Private Subnet"]
            Lambda["Lambda"]
            EC2["EC2"]
        end
        NAT["NAT Gateway<br>$32/month + data"]
    end

    Lambda -->|"S3 API call"| NAT
    EC2 -->|"SQS API call"| NAT
    NAT --> Internet((Internet)) --> AWS["S3, SQS, etc."]

    style NAT fill:#ffcdd2,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

**Problem:** Traffic to AWS services goes through NAT Gateway, costing $0.045/GB.

### Solution: VPC Endpoints

```mermaid
flowchart LR
    subgraph VPC["VPC"]
        subgraph Private["Private Subnet"]
            Lambda["Lambda"]
            EC2["EC2"]
        end
        GWEndpoint["Gateway Endpoint<br>S3, DynamoDB<br>FREE"]
        IFEndpoint["Interface Endpoint<br>SQS, SES, etc.<br>~$7/month each"]
    end

    Lambda -->|"S3"| GWEndpoint --> S3["S3"]
    EC2 -->|"SQS"| IFEndpoint --> SQS["SQS"]

    style GWEndpoint fill:#c8e6c9,color:#000
    style IFEndpoint fill:#fff9c4,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### Endpoint Types

| Type | Services | Cost | How It Works |
|------|----------|------|--------------|
| **Gateway** | S3, DynamoDB | Free | Route table entry |
| **Interface** | Most others | ~$7/month | ENI in subnet |

> **SAA Exam Tip:** Gateway endpoints are FREE for S3 and DynamoDB. Always create them to avoid NAT Gateway data charges. Interface endpoints cost money but may still be cheaper than NAT for high-traffic services.

---

## Phase 6 Complete Architecture

```mermaid
flowchart TB
    User((User))

    User --> R53["Route 53<br>techbooks.com"]
    R53 --> CloudFront["CloudFront"]

    CloudFront --> S3Static["S3 Static<br>Assets"]
    CloudFront --> ALB

    subgraph VPC["VPC: 10.0.0.0/16"]
        ALB["ALB"]

        subgraph AppLayer["Application Layer"]
            EC2a["EC2"]
            EC2b["EC2"]
        end

        subgraph CacheLayer["Cache Layer"]
            Redis["ElastiCache<br>Redis"]
        end

        subgraph DataLayer["Data Layer"]
            RDSPrimary[("RDS Primary")]
            RDSStandby[("RDS Standby")]
        end

        subgraph Endpoints["VPC Endpoints"]
            S3EP["S3 Endpoint"]
            SQSEP["SQS Endpoint"]
        end
    end

    ALB --> EC2a & EC2b
    EC2a & EC2b --> Redis
    EC2a & EC2b --> RDSPrimary
    RDSPrimary <--> RDSStandby

    subgraph Serverless["Serverless Components"]
        S3Uploads["S3 Uploads"]
        SQS["SQS Queue"]
        LambdaImg["Lambda<br>Image Processing"]
        LambdaEmail["Lambda<br>Email"]
        SES["SES"]
    end

    EC2a -->|"Presigned URL"| S3Uploads
    S3Uploads -->|"Event"| LambdaImg
    EC2a -->|"Message"| SQS
    SQS --> LambdaEmail
    LambdaEmail --> SES

    style VPC fill:#e3f2fd,color:#000
    style AppLayer fill:#c8e6c9,color:#000
    style CacheLayer fill:#c8e6c9,color:#000
    style DataLayer fill:#fff9c4,color:#000
    style Serverless fill:#fff9c4,color:#000
    style Endpoints fill:#e8f5e9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

---

## Key SAA Exam Concepts from Phase 6

### Lambda

- Runs code without servers, pay per invocation
- 15-minute max timeout, 10GB max memory
- Cold starts affect latency (especially in VPC)
- Use for event-driven, unpredictable workloads
- VPC Lambda needs NAT Gateway for internet access

### SQS

- Managed message queue for decoupling
- Standard (best-effort order, unlimited throughput) vs FIFO (strict order, 3000 msg/sec)
- Dead Letter Queues capture failed messages
- Visibility Timeout hides messages during processing

### ElastiCache

- In-memory cache: Redis (feature-rich) vs Memcached (simple)
- Redis supports persistence, replication, Multi-AZ
- Cache-aside pattern: check cache → miss → query DB → populate cache
- Externalize sessions for stateless applications

### Cost Optimization

- Reserved Instances: up to 72% savings, commit to specific instance
- Savings Plans: up to 66% savings, flexible commitment
- Gateway Endpoints: FREE for S3/DynamoDB
- Interface Endpoints: ~$7/month per service, avoids NAT charges

---

## Final Cost Comparison

| Phase | Monthly Cost | What's Included |
|-------|--------------|-----------------|
| Phase 1 (MVP) | ~$15 | 1 EC2, EIP |
| Phase 2 (RDS) | ~$45 | + RDS, NAT Gateway |
| Phase 3 (HA) | ~$70 | + Multi-AZ RDS |
| Phase 4 (Scaling) | ~$85 | + ALB, ASG |
| Phase 5 (Global) | ~$105 | + CloudFront, S3 |
| Phase 6 (Modern) | ~$130 | + ElastiCache, Lambda, SQS |
| Phase 6 (Optimized) | ~$95 | With Reserved Instances |

---

## TechBooks Journey Complete

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
    style P6 fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### What We Learned

| Phase | Key Services | Key Concepts |
|-------|--------------|--------------|
| 1 | VPC, EC2, SG | Networking, compute basics |
| 2 | RDS, NAT GW | Managed services, private subnets |
| 3 | Multi-AZ | High availability, failover |
| 4 | ALB, ASG | Horizontal scaling, load balancing |
| 5 | CloudFront, Route 53 | CDN, DNS, global distribution |
| 6 | Lambda, SQS, ElastiCache | Serverless, decoupling, caching |

### SAA Exam Domains Covered

| Domain | Topics |
|--------|--------|
| **Secure Architectures** | VPC, Security Groups, OAC, private subnets |
| **Resilient Architectures** | Multi-AZ, Auto Scaling, SQS decoupling |
| **High-Performing** | CloudFront, ElastiCache, Lambda |
| **Cost-Optimized** | Reserved Instances, Savings Plans, VPC Endpoints |

---

## What's Next?

TechBooks is now a modern, scalable, globally distributed e-commerce platform. Future enhancements could include:

- **Multi-Region**: Active-active across regions (DynamoDB Global Tables, Route 53 failover)
- **Containers**: Migrate to ECS/EKS for better deployment consistency
- **Machine Learning**: Personalized recommendations with Amazon Personalize
- **Analytics**: Real-time analytics with Kinesis and QuickSight
- **Security**: AWS WAF, Shield Advanced, GuardDuty

But that's for another scenario!
