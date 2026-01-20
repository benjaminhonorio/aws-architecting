# Phase 4: Auto Scaling & Load Balancing

## Business Context

**Situation:** TechBooks is booming! Traffic has reached 10,000 visitors/day. But problems are
mounting.

**Recent incidents:**

- Site slowed to a crawl during a flash sale
- Deployment caused 5-minute outage (customers complained on Twitter)
- CPU hits 100% during peak hours
- You manually upgraded to t3.small, but it's still not enough

**The founder asks:** "Why can't the site just... handle more traffic automatically? Netflix does
it!"

**Your decision:** Implement Auto Scaling with a Load Balancer for true fault tolerance and
elasticity.

---

## Step 1: Vertical vs Horizontal Scaling

### The Two Approaches

```mermaid
flowchart TB
    subgraph Vertical["Vertical Scaling (Scale Up)"]
        direction TB
        V1["t3.micro"] --> V2["t3.small"]
        V2 --> V3["t3.medium"]
        V3 --> V4["t3.large"]
        Limit["Hardware limit<br>eventually reached"]
    end

    subgraph Horizontal["Horizontal Scaling (Scale Out)"]
        direction TB
        H1["1 instance"]
        H2["2 instances"]
        H3["3 instances"]
        H4["N instances..."]
        H1 --> H2 --> H3 --> H4
        NoLimit["Virtually<br>unlimited"]
    end

    style Vertical fill:#fff3e0,color:#000
    style Horizontal fill:#c8e6c9,color:#000
    style Limit fill:#ffcdd2,color:#000
    style NoLimit fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### Comparison

Understanding these two scaling approaches is fundamental - the exam frequently tests when to use
each:

| Aspect              | Vertical (Scale Up)        | Horizontal (Scale Out)             |
| ------------------- | -------------------------- | ---------------------------------- |
| **How**             | Bigger instance            | More instances                     |
| **Downtime**        | Yes (stop, resize, start)  | No (add instances live)            |
| **Cost efficiency** | Linear (2x size ≈ 2x cost) | Better (smaller instances cheaper) |
| **Limit**           | Max instance size          | Virtually unlimited                |
| **Fault tolerance** | No (still single point)    | Yes (instances can fail)           |
| **Complexity**      | Simple                     | Requires load balancer             |

### WHY Horizontal Scaling for TechBooks

| Problem          | Vertical Solution            | Horizontal Solution            |
| ---------------- | ---------------------------- | ------------------------------ |
| Peak traffic     | Buy biggest instance 24/7    | Add instances during peak      |
| Deployment       | Downtime while restarting    | Rolling update, zero downtime  |
| Instance failure | Site down                    | Other instances handle traffic |
| Cost             | Pay for peak capacity always | Pay for what you need          |

> **SAA Exam Tip:** "Scalable and highly available" almost always means horizontal scaling with Auto
> Scaling and Load Balancer. Vertical scaling has hard limits and causes downtime.

---

## Step 2: Elastic Load Balancing (ELB) Types

### AWS Load Balancer Options

AWS offers four types of load balancers:

| Type                  | Layer     | Use Case                   | Protocol          |
| --------------------- | --------- | -------------------------- | ----------------- |
| **ALB** (Application) | Layer 7   | HTTP/HTTPS, microservices  | HTTP, HTTPS, gRPC |
| **NLB** (Network)     | Layer 4   | Ultra-low latency, TCP/UDP | TCP, UDP, TLS     |
| **GLB** (Gateway)     | Layer 3   | Third-party appliances     | IP                |
| **CLB** (Classic)     | Layer 4/7 | Legacy (don't use for new) | HTTP, HTTPS, TCP  |

```mermaid
flowchart TB
    subgraph OSI["OSI Model (Open Systems Interconnection)"]
        L7["Layer 7: Application<br>HTTP headers, paths, hosts"]
        L4["Layer 4: Transport<br>TCP/UDP ports"]
        L3["Layer 3: Network<br>IP addresses"]
    end

    L7 --> ALB["ALB<br>Content-based routing"]
    L4 --> NLB["NLB<br>Connection-based routing"]
    L3 --> GLB["GLB<br>Appliance routing"]

    style L7 fill:#c8e6c9,color:#000
    style L4 fill:#fff9c4,color:#000
    style L3 fill:#e3f2fd,color:#000
    style ALB fill:#c8e6c9,color:#000
    style NLB fill:#fff9c4,color:#000
    style GLB fill:#e3f2fd,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### WHY ALB for TechBooks

| Requirement                             | ALB Feature                  |
| --------------------------------------- | ---------------------------- |
| Web application (HTTP/HTTPS)            | Layer 7 - understands HTTP   |
| Route `/api/*` to API servers           | Path-based routing           |
| Route `admin.techbooks.com` differently | Host-based routing           |
| Health check on `/health` endpoint      | HTTP health checks           |
| SSL termination                         | HTTPS listener with ACM cert |
| WebSocket support (future chat)         | Native WebSocket support     |

> **SAA Exam Tip:** ALB for HTTP/HTTPS applications. NLB for extreme performance, static IPs, or
> non-HTTP protocols. CLB is legacy - don't choose for new architectures.

---

## Step 3: Application Load Balancer Deep Dive

### ALB Components

```mermaid
flowchart TB
    User((User)) --> DNS["DNS:<br>techbooks.com"]
    DNS --> ALB["Application Load Balancer"]

    subgraph ALBDetail["ALB Components"]
        Listener["Listener<br>Port 443 (HTTPS)"]
        Rules["Rules<br>Path, Host, Headers"]
        TG["Target Group<br>Health checks"]
    end

    ALB --> Listener --> Rules --> TG

    TG --> EC2a["EC2 Instance 1"]
    TG --> EC2b["EC2 Instance 2"]
    TG --> EC2c["EC2 Instance 3"]

    style ALBDetail fill:#e3f2fd,color:#000
    style Listener fill:#fff9c4,color:#000
    style Rules fill:#fff9c4,color:#000
    style TG fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### Component Breakdown

**1. Listener**

- Checks for connection requests on a port/protocol
- Example: HTTPS on port 443

**2. Rules**

- Determine how to route requests
- Conditions: path, host header, HTTP headers, query strings
- Actions: forward, redirect, fixed response

**3. Target Group**

- Group of targets (EC2, Lambda, IP)
- Health check configuration
- Load balancing algorithm

### Path-Based Routing Example

```mermaid
flowchart LR
    ALB["ALB"] --> Rule1{"/api/*"}
    ALB --> Rule2{"/admin/*"}
    ALB --> Rule3{"/* (default)"}

    Rule1 -->|"Yes"| TG1["API Target Group<br>API servers"]
    Rule2 -->|"Yes"| TG2["Admin Target Group<br>Admin servers"]
    Rule3 -->|"Yes"| TG3["Web Target Group<br>Web servers"]

    style ALB fill:#e3f2fd,color:#000
    style TG1 fill:#c8e6c9,color:#000
    style TG2 fill:#fff9c4,color:#000
    style TG3 fill:#bbdefb,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### WHY This Matters for TechBooks

For now, we route everything to one target group. But this architecture allows us to:

- Add separate API servers later
- Route admin traffic to specific instances
- A/B test with weighted routing
- Implement blue-green deployments

> **SAA Exam Tip:** ALB can route based on path (`/api/*`), host (`api.example.com`), HTTP headers,
> query strings, and source IP. Know these routing options!

---

## Step 4: Health Checks

### WHY Health Checks Matter

Without health checks, the load balancer might send traffic to a dead instance:

```mermaid
flowchart LR
    subgraph Bad["Without Health Checks"]
        ALB1["ALB"] --> Dead["Dead Instance<br>Errors!"]
        ALB1 --> Healthy1["Healthy Instance"]
    end

    subgraph Good["With Health Checks"]
        ALB2["ALB"] --> Healthy2["Healthy Instance"]
        ALB2 --> Healthy3["Healthy Instance"]
        Unhealthy["Unhealthy Instance<br>No traffic"]
    end

    style Bad fill:#ffcdd2,color:#000
    style Good fill:#c8e6c9,color:#000
    style Dead fill:#ffcdd2,color:#000
    linkStyle default stroke:#000,stroke-width:2px
    style Unhealthy fill:#f5f5f5,color:#000,stroke-dasharray: 5 5
```

### Health Check Types

Multiple health check mechanisms work together. Understanding which one does what is important for
troubleshooting:

| Type                    | Performed By  | Checks                    | Action on Failure      |
| ----------------------- | ------------- | ------------------------- | ---------------------- |
| **ELB Health Check**    | Load Balancer | HTTP request to `/health` | Stop sending traffic   |
| **EC2 Health Check**    | Auto Scaling  | Instance status           | Terminate and replace  |
| **Custom Health Check** | Your code     | Application logic         | Report to Auto Scaling |

### ELB Health Check Configuration

| Setting                 | Our Value  | WHY                                |
| ----------------------- | ---------- | ---------------------------------- |
| **Protocol**            | HTTP       | App serves HTTP                    |
| **Path**                | `/health`  | Dedicated health endpoint          |
| **Port**                | 80         | Application port                   |
| **Healthy threshold**   | 2          | 2 consecutive successes = healthy  |
| **Unhealthy threshold** | 3          | 3 consecutive failures = unhealthy |
| **Timeout**             | 5 seconds  | Max wait for response              |
| **Interval**            | 30 seconds | Time between checks                |

### What Should `/health` Return?

```javascript
// Good health check endpoint
app.get("/health", async (req, res) => {
  try {
    // Check database connection
    await db.query("SELECT 1");

    // Check critical dependencies
    // await redis.ping();

    res.status(200).json({ status: "healthy" });
  } catch (error) {
    res.status(500).json({ status: "unhealthy", error: error.message });
  }
});
```

**WHY check dependencies:**

- Instance might be running but database connection lost
- Deep health check catches more failures
- ALB routes away from instances with dependency issues

> **SAA Exam Tip:** ELB health checks determine traffic routing. Auto Scaling uses health checks to
> replace instances. Both should be configured for robust HA.

---

## Step 5: Auto Scaling Groups (ASG)

### What is an Auto Scaling Group?

An **Auto Scaling Group** automatically adjusts the number of EC2 instances based on demand or
schedule.

```mermaid
flowchart TB
    subgraph ASG["Auto Scaling Group"]
        Config["Configuration:<br>Min: 2, Desired: 2, Max: 6"]

        subgraph Instances["Current Instances"]
            EC2a["EC2-1<br>AZ-1a"]
            EC2b["EC2-2<br>AZ-1b"]
        end
    end

    Trigger["High CPU<br>Trigger"]
    ScaleOut["Scale Out<br>Add instance"]

    Trigger --> ASG
    ASG --> ScaleOut
    ScaleOut --> EC2c["EC2-3<br>AZ-1a"]

    style ASG fill:#e3f2fd,color:#000
    style Config fill:#fff9c4,color:#000
    style Instances fill:#c8e6c9,color:#000
    style Trigger fill:#ffcdd2,color:#000
    style ScaleOut fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### ASG Components

**1. Launch Template**

- Blueprint for new instances
- AMI, instance type, security groups, user data
- Replaces older "Launch Configuration"

**2. Capacity Settings**

| Setting     | Description               | Our Value          |
| ----------- | ------------------------- | ------------------ |
| **Minimum** | Never go below this       | 2 (HA requirement) |
| **Desired** | Target number to maintain | 2 (normal traffic) |
| **Maximum** | Never exceed this         | 6 (cost control)   |

**3. Scaling Policies**

- When and how to scale
- Target tracking, step scaling, scheduled

### WHY Minimum of 2?

```mermaid
flowchart LR
    subgraph Min1["Minimum: 1"]
        Single["1 Instance<br>AZ-1a"]
        Risk1["AZ failure = Outage<br>Instance failure = Outage"]
    end

    subgraph Min2["Minimum: 2"]
        Multi1["Instance 1<br>AZ-1a"]
        Multi2["Instance 2<br>AZ-1b"]
        Risk2["AZ failure = Still running<br>Instance failure = Still running"]
    end

    style Min1 fill:#ffcdd2,color:#000
    style Min2 fill:#c8e6c9,color:#000
    style Risk1 fill:#ffcdd2,color:#000
    style Risk2 fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

**Rule:** For high availability, minimum should be at least 2, spread across AZs.

> **SAA Exam Tip:** ASG spreads instances across AZs in the subnet configuration. With 2 AZs and
> min=2, you get one instance per AZ automatically.

---

## Step 6: Launch Template

### What Goes in a Launch Template?

```mermaid
flowchart TB
    subgraph LT["Launch Template: techbooks-web-lt"]
        AMI["AMI: ami-techbooks-v1<br>(pre-configured image)"]
        Type["Instance Type: t3.micro"]
        SG["Security Group: sg-techbooks-web"]
        Key["Key Pair: techbooks-key"]
        IAM["IAM Role: TechBooksEC2Role"]
        UserData["User Data:<br>startup script"]
    end

    LT --> ASG["Auto Scaling Group<br>uses this template"]
    ASG --> EC2["New EC2 instances<br>created from template"]

    style LT fill:#e3f2fd,color:#000
    style AMI fill:#fff9c4,color:#000
    style UserData fill:#fff9c4,color:#000
    style ASG fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### User Data Script

The user data script runs on first boot:

```bash
#!/bin/bash
# Update and install dependencies
yum update -y
yum install -y nodejs nginx

# Pull latest application code
aws s3 cp s3://techbooks-deploy/app.zip /opt/app/
unzip /opt/app/app.zip -d /opt/app/

# Configure and start services
systemctl start nginx
cd /opt/app && npm start
```

### AMI Strategy: Golden AMI vs User Data

| Approach       | Pros                  | Cons                            |
| -------------- | --------------------- | ------------------------------- |
| **Golden AMI** | Fast boot, consistent | Must rebuild for updates        |
| **User Data**  | Always latest code    | Slower boot, potential failures |
| **Hybrid**     | Best of both          | More complex                    |

**TechBooks approach:** Golden AMI with base software, User Data pulls latest code.

> **SAA Exam Tip:** Launch Templates are the modern replacement for Launch Configurations. Templates
> support versioning and can be updated without recreating the ASG.

---

## Step 7: Scaling Policies

### Types of Scaling Policies

```mermaid
flowchart TB
    subgraph Policies["Scaling Policy Types"]
        TT["Target Tracking<br>'Keep CPU at 50%'"]
        Step["Step Scaling<br>'If CPU > 80%, add 2'"]
        Sched["Scheduled<br>'Add 3 at 9am'"]
        Pred["Predictive<br>'ML-based forecast'"]
    end

    TT --> Simple["Simplest to configure"]
    Step --> Granular["More control"]
    Sched --> Predictable["Known patterns"]
    Pred --> ML["Machine learning"]

    style TT fill:#c8e6c9,color:#000
    style Step fill:#fff9c4,color:#000
    style Sched fill:#fff9c4,color:#000
    style Pred fill:#e3f2fd,color:#000
    style Simple fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### Target Tracking (Recommended for Most Cases)

"Maintain average CPU at 50%"

```mermaid
flowchart LR
    subgraph Tracking["Target Tracking: CPU 50%"]
        Low["CPU: 30%"] -->|"Scale in"| Target["CPU: 50%"]
        High["CPU: 70%"] -->|"Scale out"| Target
    end

    style Tracking fill:#e3f2fd,color:#000
    style Target fill:#c8e6c9,color:#000
    style Low fill:#bbdefb,color:#000
    style High fill:#ffcdd2,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

**Available target metrics:**

- `ASGAverageCPUUtilization` - Average CPU
- `ASGAverageNetworkIn` - Network bytes in
- `ASGAverageNetworkOut` - Network bytes out
- `ALBRequestCountPerTarget` - Requests per instance

### Step Scaling (More Control)

Define exact actions for specific thresholds:

| CPU Range | Action            |
| --------- | ----------------- |
| 0-40%     | Remove 1 instance |
| 40-60%    | Do nothing        |
| 60-80%    | Add 1 instance    |
| 80-100%   | Add 2 instances   |

### Scheduled Scaling (Known Patterns)

```mermaid
flowchart LR
    subgraph Schedule["TechBooks Traffic Pattern"]
        Morning["9 AM<br>Scale to 4"]
        Peak["12 PM<br>Scale to 6"]
        Evening["6 PM<br>Scale to 3"]
        Night["10 PM<br>Scale to 2"]
    end

    Morning --> Peak --> Evening --> Night

    style Morning fill:#fff9c4,color:#000
    style Peak fill:#ffcdd2,color:#000
    style Evening fill:#fff9c4,color:#000
    style Night fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### TechBooks Scaling Configuration

For TechBooks, we combine multiple policies for comprehensive coverage:

| Policy       | Type            | Configuration                  |
| ------------ | --------------- | ------------------------------ |
| **Primary**  | Target Tracking | CPU at 50%                     |
| **Backup**   | Scheduled       | Min 4 during business hours    |
| **Cooldown** | -               | 300 seconds (prevent flapping) |

> **SAA Exam Tip:** Target tracking is the simplest and recommended for most cases. Step scaling
> provides more control. Scheduled scaling is for predictable patterns. You can combine multiple
> policies.

---

## Step 8: Removing the Elastic IP

### WHY We No Longer Need Elastic IP

```mermaid
flowchart TB
    subgraph Before["Phase 3: Single EC2"]
        User1((User)) --> EIP["Elastic IP<br>54.210.5.100"]
        EIP --> EC2Single["Single EC2"]
    end

    subgraph After["Phase 4: ALB + ASG"]
        User2((User)) --> DNS["techbooks.com"]
        DNS --> ALB["ALB DNS<br>techbooks-alb.us-east-1.elb.amazonaws.com"]
        ALB --> EC2a["EC2-1"]
        ALB --> EC2b["EC2-2"]
    end

    style Before fill:#fff3e0,color:#000
    style After fill:#c8e6c9,color:#000
    style EIP fill:#ffcdd2,color:#000
    style ALB fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

**Changes:**

| Aspect                   | Before (EIP)      | After (ALB)                                    |
| ------------------------ | ----------------- | ---------------------------------------------- |
| **DNS points to**        | Elastic IP        | ALB DNS name                                   |
| **SSL termination**      | EC2 instance      | ALB                                            |
| **Instance replacement** | Must reassign EIP | Automatic                                      |
| **Cost**                 | Free if attached  | ALB hourly + LCU (Load Balancer Capacity Unit) |

### Updating DNS

```
techbooks.com → techbooks-alb-123456.us-east-1.elb.amazonaws.com
```

Use a CNAME or Route 53 Alias record (Alias is better - no charge for queries).

> **SAA Exam Tip:** Route 53 Alias records to ALB are free and support zone apex (naked domain).
> CNAME records don't work for zone apex and incur query charges.

---

## Step 9: Cross-Zone Load Balancing

### The Problem Without Cross-Zone

```mermaid
flowchart TB
    ALB["ALB"]

    subgraph AZ1["AZ-1a: 4 instances"]
        EC2a["EC2"]
        EC2b["EC2"]
        EC2c["EC2"]
        EC2d["EC2"]
    end

    subgraph AZ2["AZ-1b: 1 instance"]
        EC2e["EC2"]
    end

    ALB -->|"50% traffic"| AZ1
    ALB -->|"50% traffic"| AZ2

    Note1["Each instance in AZ-1a<br>gets 12.5% traffic"]
    Note2["Instance in AZ-1b<br>gets 50% traffic!"]

    style AZ1 fill:#c8e6c9,color:#000
    style AZ2 fill:#ffcdd2,color:#000
    style Note2 fill:#ffcdd2,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### With Cross-Zone Load Balancing

```mermaid
flowchart TB
    ALB["ALB<br>Cross-Zone Enabled"]

    subgraph AZ1["AZ-1a"]
        EC2a["EC2: 20%"]
        EC2b["EC2: 20%"]
        EC2c["EC2: 20%"]
        EC2d["EC2: 20%"]
    end

    subgraph AZ2["AZ-1b"]
        EC2e["EC2: 20%"]
    end

    ALB --> EC2a
    ALB --> EC2b
    ALB --> EC2c
    ALB --> EC2d
    ALB --> EC2e

    style ALB fill:#c8e6c9,color:#000
    style AZ1 fill:#c8e6c9,color:#000
    style AZ2 fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

**Cross-zone load balancing:** Distributes traffic evenly across ALL instances in ALL AZs.

| Load Balancer | Cross-Zone Default | Data Transfer Cost |
| ------------- | ------------------ | ------------------ |
| ALB           | Always enabled     | No charge          |
| NLB           | Disabled           | Charged if enabled |
| CLB           | Disabled           | No charge          |

> **SAA Exam Tip:** ALB always has cross-zone load balancing enabled and doesn't charge for cross-AZ
> data transfer. NLB charges for cross-AZ if enabled.

---

## Step 10: Connection Draining (Deregistration Delay)

### WHY It Matters

When an instance is removed (scale in or unhealthy), what happens to in-flight requests?

```mermaid
sequenceDiagram
    participant User
    participant ALB
    participant EC2

    User->>ALB: Start checkout
    ALB->>EC2: Forward request
    Note over EC2: Scale-in triggered!

    alt Without Connection Draining
        EC2->>EC2: Instance terminated
        EC2--xUser: Connection dropped!
        Note over User: Error! Lost cart!
    end

    alt With Connection Draining
        Note over EC2: Stop NEW requests
        EC2->>User: Complete checkout
        Note over EC2: Then terminate
        Note over User: Success!
    end
```

### Configuration

| Setting                  | Value       | WHY                          |
| ------------------------ | ----------- | ---------------------------- |
| **Deregistration delay** | 300 seconds | Allow requests to complete   |
| **For TechBooks**        | 60 seconds  | Most requests finish quickly |

**Trade-off:** Longer delay = safer but slower scaling.

> **SAA Exam Tip:** Connection draining / deregistration delay ensures in-flight requests complete
> before an instance is removed. Default is 300 seconds.

---

## Step 11: Security Group Updates

### New Security Group Architecture

```mermaid
flowchart TB
    Internet((Internet))

    subgraph Public["Public Subnets"]
        ALB["ALB<br>sg-alb"]
    end

    subgraph Private["Could be private"]
        EC2a["EC2<br>sg-web"]
        EC2b["EC2<br>sg-web"]
    end

    RDS["RDS<br>sg-db"]

    Internet -->|"443"| ALB
    ALB -->|"80"| EC2a
    ALB -->|"80"| EC2b
    EC2a -->|"3306"| RDS
    EC2b -->|"3306"| RDS

    style Public fill:#c8e6c9,color:#000
    style Private fill:#fff9c4,color:#000
    style ALB fill:#e3f2fd,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### Security Group Rules

**ALB Security Group (sg-techbooks-alb):**

```
Inbound:
- HTTPS (443) from 0.0.0.0/0    # Public traffic

Outbound:
- HTTP (80) to sg-techbooks-web  # To EC2 instances
```

**EC2 Security Group (sg-techbooks-web):**

```
Inbound:
- HTTP (80) from sg-techbooks-alb  # Only from ALB!
- SSH (22) from your-ip/32         # Admin access

Outbound:
- MySQL (3306) to sg-techbooks-db  # To RDS
- HTTPS (443) to 0.0.0.0/0         # For updates, APIs
```

**Key change:** EC2 no longer accepts traffic from internet directly - only from ALB.

> **SAA Exam Tip:** Reference security groups instead of IPs. EC2 instances should only accept
> traffic from ALB, not directly from the internet.

---

## Phase 4 Complete Architecture

```mermaid
flowchart TB
    User((User)) --> Internet((Internet))
    Internet --> Route53["Route 53<br>techbooks.com"]
    Route53 --> ALB["Application Load Balancer<br>HTTPS:443"]

    subgraph VPC["VPC: 10.0.0.0/16"]
        subgraph AZ1["Availability Zone: us-east-1a"]
            subgraph Public1["Public Subnet: 10.0.1.0/24"]
                ALBNode1["ALB Node"]
            end
            subgraph Private1["Private Subnet: 10.0.10.0/24"]
                EC2a["EC2"]
                RDSPrimary["RDS Primary"]
            end
        end

        subgraph AZ2["Availability Zone: us-east-1b"]
            subgraph Public2["Public Subnet: 10.0.2.0/24"]
                ALBNode2["ALB Node"]
            end
            subgraph Private2["Private Subnet: 10.0.20.0/24"]
                EC2b["EC2"]
                RDSStandby["RDS Standby"]
            end
        end

        ALB --> ALBNode1
        ALB --> ALBNode2
        ALBNode1 --> EC2a
        ALBNode2 --> EC2b
        EC2a --> RDSPrimary
        EC2b --> RDSPrimary
        RDSPrimary <--> RDSStandby
    end

    subgraph ASG["Auto Scaling Group"]
        ASGConfig["Min: 2 | Desired: 2 | Max: 6<br>Target Tracking: CPU 50%"]
    end

    ASG -.->|"Manages"| EC2a
    ASG -.->|"Manages"| EC2b

    style VPC fill:#e3f2fd,color:#000
    style Public1 fill:#c8e6c9,color:#000
    style Public2 fill:#c8e6c9,color:#000
    style Private1 fill:#fff9c4,color:#000
    style Private2 fill:#fff9c4,color:#000
    style ALB fill:#e1f5fe,color:#000
    style RDSPrimary fill:#bbdefb,color:#000
    style RDSStandby fill:#bbdefb,color:#000
    style ASG fill:#e8f5e9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

---

## See Also

> **Related Learning:** For a deep dive into IAM roles and how EC2 instances assume roles securely,
> see
> [MedVault Phase 1: Identity Foundation](/scenarios/medvault/phases/phase-1-identity-foundation.md).

---

## Key SAA Exam Concepts from Phase 4

### Must-Know Topics

1. **Load Balancer Types**
   - ALB: Layer 7, HTTP/HTTPS, content-based routing
   - NLB: Layer 4, ultra-low latency, static IP
   - CLB: Legacy, don't use for new projects

2. **ALB Features**
   - Path-based routing (`/api/*`)
   - Host-based routing (`api.example.com`)
   - Health checks on target groups
   - SSL termination

3. **Auto Scaling**
   - Launch Template (not Launch Configuration)
   - Min/Desired/Max capacity
   - Target Tracking is simplest policy
   - Spreads across AZs automatically

4. **Health Checks**
   - ELB health checks: routing decisions
   - EC2 health checks: instance replacement
   - Both should be configured

5. **Cross-Zone Load Balancing**
   - ALB: Always on, free
   - NLB: Optional, costs for cross-AZ

6. **Connection Draining**
   - Allows in-flight requests to complete
   - Default 300 seconds

---

## Cost Analysis

| Component         | Phase 3 Cost | Phase 4 Cost | Notes           |
| ----------------- | ------------ | ------------ | --------------- |
| EC2 (t3.micro x2) | ~$8/month    | ~$16/month   | Min 2 instances |
| RDS Multi-AZ      | ~$24/month   | ~$24/month   | No change       |
| ALB               | $0           | ~$20/month   | Hourly + LCU    |
| Data Transfer     | ~$2/month    | ~$5/month    | More traffic    |
| Elastic IP        | ~$0          | $0           | Removed!        |
| **Total**         | ~$36/month   | ~$65/month   | +$29            |

**WHY it's worth it:**

- Zero-downtime deployments
- Automatic scaling for traffic spikes
- Fault tolerance (instance failures don't cause outage)
- Foundation for 99.99% availability

---

## What's Coming in Phase 5?

**Business trigger:** TechBooks is going international! Customers in Europe and Asia complain about
slow load times. The founder wants to expand globally.

**Next decisions:**

- CloudFront CDN for static assets and caching
- Route 53 for DNS and geo-routing
- Multi-region considerations
- S3 for static content

---

## Hands-On Challenge

Before moving to Phase 5:

1. Create an Application Load Balancer in your public subnets
2. Create a Target Group with health check on `/health`
3. Create a Launch Template with your EC2 configuration
4. Create an Auto Scaling Group (min: 2, max: 4)
5. Configure Target Tracking policy for 50% CPU
6. Update Route 53 to point to ALB
7. Release the Elastic IP

**Verification:**

- ALB DNS resolves and shows your site
- Terminate one EC2 instance - another should launch automatically
- Run a load test - watch instances scale out

---

## References

Official AWS documentation used to validate this content:

### Elastic Load Balancing

- [How ELB Works](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/how-elastic-load-balancing-works.html) -
  Cross-zone load balancing and load balancer nodes
- [What is an Application Load Balancer?](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html) -
  ALB Layer 7, path/host-based routing
- [Edit Target Group Attributes](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/edit-target-group-attributes.html) -
  Deregistration delay (default 300 seconds)

### Auto Scaling

- [Auto Scaling Launch Templates](https://docs.aws.amazon.com/autoscaling/ec2/userguide/launch-templates.html) -
  Launch Templates vs Launch Configurations
- [Auto Scaling Group Availability Zone Distribution](https://docs.aws.amazon.com/autoscaling/ec2/userguide/ec2-auto-scaling-availability-zone-balanced.html) -
  Instance distribution across AZs

### Route 53

- [Choosing Between Alias and Non-Alias Records](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resource-record-sets-choosing-alias-non-alias.html) -
  Zone apex support for Alias records, CNAME limitations
