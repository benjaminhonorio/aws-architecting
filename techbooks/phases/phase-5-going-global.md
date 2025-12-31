# Phase 5: Going Global

## Business Context

**Situation:** TechBooks is expanding internationally! The marketing team ran campaigns in Europe and Asia, and orders are coming in from around the world.

**Problems reported:**
- European customers: "Your site takes 4 seconds to load"
- Asian customers: "Images load slowly, checkout times out"
- US customers: "Site is fine for me" (origin is in us-east-1)

**Data you gathered:**
| Region | Latency to us-east-1 | User Experience |
|--------|---------------------|-----------------|
| US East | ~20ms | Great |
| US West | ~80ms | Good |
| Europe | ~120ms | Acceptable |
| Asia | ~250ms | Poor |

**The founder asks:** "Can we make the site fast for everyone, everywhere?"

**Your decision:** Implement CloudFront CDN and optimize DNS with Route 53.

---

## Step 1: Why Is It Slow?

### The Physics Problem

Every HTTP request travels to your origin in Virginia (us-east-1). The speed of light is fast, but not instant.

```mermaid
flowchart LR
    subgraph Users["Users Around the World"]
        Tokyo["Tokyo User<br>~250ms round trip"]
        London["London User<br>~120ms round trip"]
        NYC["NYC User<br>~20ms round trip"]
    end

    subgraph Origin["Origin: us-east-1"]
        ALB["ALB + EC2"]
    end

    Tokyo -->|"14,000 km"| ALB
    London -->|"6,000 km"| ALB
    NYC -->|"400 km"| ALB

    style Tokyo fill:#ffcdd2,color:#000
    style London fill:#fff9c4,color:#000
    style NYC fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### The Real Impact

A typical page load:
- HTML document: 1 request
- CSS files: 3 requests
- JavaScript: 5 requests
- Images: 20 requests
- Fonts: 2 requests
- **Total: ~30 requests**

Each request from Tokyo = ~250ms. Even with parallel requests, this adds up to seconds of latency.

> **SAA Exam Tip:** Latency-sensitive applications need content served from locations close to users. This is the core problem CDNs solve.

---

## Step 2: Content Delivery Network (CDN) Concept

### What is a CDN?

A **Content Delivery Network** caches content at edge locations around the world, serving users from the nearest location.

```mermaid
flowchart TB
    subgraph WithoutCDN["Without CDN"]
        U1["Tokyo User"] -->|"250ms"| O1["Origin<br>us-east-1"]
        U2["London User"] -->|"120ms"| O1
    end

    subgraph WithCDN["With CDN"]
        U3["Tokyo User"] -->|"10ms"| E1["Edge<br>Tokyo"]
        U4["London User"] -->|"10ms"| E2["Edge<br>London"]
        E1 -.->|"Cache miss only"| O2["Origin<br>us-east-1"]
        E2 -.->|"Cache miss only"| O2
    end

    style WithoutCDN fill:#ffcdd2,color:#000
    style WithCDN fill:#c8e6c9,color:#000
    style E1 fill:#c8e6c9,color:#000
    style E2 fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### How Caching Works

```mermaid
sequenceDiagram
    participant User
    participant Edge as Edge Location
    participant Origin

    Note over User,Origin: First Request (Cache Miss)
    User->>Edge: GET /logo.png
    Edge->>Edge: Cache check: MISS
    Edge->>Origin: GET /logo.png
    Origin-->>Edge: logo.png + Cache headers
    Edge->>Edge: Store in cache
    Edge-->>User: logo.png

    Note over User,Origin: Second Request (Cache Hit)
    User->>Edge: GET /logo.png
    Edge->>Edge: Cache check: HIT
    Edge-->>User: logo.png (from cache)
    Note over Origin: Not contacted!
```

**Benefits:**
- Lower latency (content closer to users)
- Reduced origin load (cached content doesn't hit origin)
- Better availability (edge can serve cached content if origin is down)
- DDoS protection (distributed across edge locations)

---

## Step 3: Amazon CloudFront

### What is CloudFront?

**Amazon CloudFront** is AWS's CDN service with 450+ edge locations in 90+ cities across 49 countries.

```mermaid
flowchart TB
    subgraph Global["CloudFront Global Network"]
        subgraph NA["North America"]
            Edge1["Edge: NYC"]
            Edge2["Edge: LA"]
            Edge3["Edge: Toronto"]
        end
        subgraph EU["Europe"]
            Edge4["Edge: London"]
            Edge5["Edge: Frankfurt"]
            Edge6["Edge: Paris"]
        end
        subgraph Asia["Asia Pacific"]
            Edge7["Edge: Tokyo"]
            Edge8["Edge: Singapore"]
            Edge9["Edge: Sydney"]
        end
    end

    subgraph Regional["Regional Edge Caches"]
        REC1["REC: Virginia"]
        REC2["REC: Oregon"]
    end

    Origin["Origin: us-east-1"]

    Edge1 & Edge2 & Edge3 --> REC1
    Edge4 & Edge5 & Edge6 --> REC1
    Edge7 & Edge8 & Edge9 --> REC2
    REC1 --> Origin
    REC2 --> Origin

    style Global fill:#e3f2fd,color:#000
    style NA fill:#c8e6c9,color:#000
    style EU fill:#c8e6c9,color:#000
    style Asia fill:#c8e6c9,color:#000
    style Regional fill:#fff9c4,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### Edge Locations vs Regional Edge Caches

| Component | Count | Purpose |
|-----------|-------|---------|
| **Edge Locations** | 450+ | Serve content to end users |
| **Regional Edge Caches** | 13 | Middle tier, larger cache |

**Flow:** User → Edge Location → Regional Edge Cache → Origin

> **SAA Exam Tip:** Regional Edge Caches have larger storage and sit between edge locations and origin. They reduce origin load for less popular content.

---

## Step 4: CloudFront Distribution

### What is a Distribution?

A **Distribution** is a CloudFront configuration that defines:
- Origins (where content comes from)
- Cache behaviors (how to handle different paths)
- Settings (SSL, logging, restrictions)

```mermaid
flowchart TB
    subgraph Distribution["CloudFront Distribution"]
        Domain["Domain:<br>d1234.cloudfront.net"]

        subgraph Behaviors["Cache Behaviors"]
            B1["/*.html<br>TTL: 5 min"]
            B2["/static/*<br>TTL: 1 year"]
            B3["/api/*<br>No cache"]
            B4["Default (*)<br>TTL: 1 day"]
        end

        subgraph Origins["Origins"]
            O1["S3: static assets"]
            O2["ALB: dynamic content"]
        end
    end

    B1 --> O2
    B2 --> O1
    B3 --> O2
    B4 --> O2

    style Distribution fill:#e3f2fd,color:#000
    style Behaviors fill:#fff9c4,color:#000
    style Origins fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### Origin Types

| Origin Type | Use Case | Example |
|-------------|----------|---------|
| **S3 Bucket** | Static files | Images, CSS, JS |
| **ALB** | Dynamic content | API, HTML pages |
| **Custom Origin** | Any HTTP server | On-premises, other clouds |
| **MediaStore** | Video streaming | Live/on-demand video |

### TechBooks Distribution Design

| Path Pattern | Origin | Cache | WHY |
|--------------|--------|-------|-----|
| `/static/*` | S3 bucket | 1 year | Images, CSS, JS rarely change |
| `/api/*` | ALB | No cache | Dynamic, personalized |
| `/*.html` | ALB | 5 minutes | Semi-dynamic pages |
| `*` (default) | ALB | 1 hour | Fallback |

> **SAA Exam Tip:** Multiple origins and cache behaviors let you optimize caching per content type. Static content should have long TTLs, dynamic content should bypass cache.

---

## Step 5: S3 for Static Assets

### WHY Move Static Assets to S3?

```mermaid
flowchart LR
    subgraph Before["Before: All on EC2"]
        EC2a["EC2 serves everything:<br>HTML, API, images, CSS, JS"]
        Problem["Problems:<br>- EC2 bandwidth limited<br>- EC2 CPU for static files<br>- Scales poorly"]
    end

    subgraph After["After: S3 + CloudFront"]
        S3["S3: static assets<br>- Unlimited bandwidth<br>- No servers<br>- $0.023/GB storage"]
        EC2b["EC2: dynamic only<br>- HTML, API<br>- Lighter load"]
    end

    style Before fill:#ffcdd2,color:#000
    style After fill:#c8e6c9,color:#000
    style Problem fill:#ffcdd2,color:#000
    style S3 fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### S3 Bucket Structure

```
techbooks-static/
├── images/
│   ├── books/
│   │   ├── book-001.jpg
│   │   └── book-002.jpg
│   └── ui/
│       ├── logo.png
│       └── icons.svg
├── css/
│   ├── main.css
│   └── vendor.css
├── js/
│   ├── app.js
│   └── vendor.js
└── fonts/
    └── opensans.woff2
```

### S3 vs EC2 for Static Content

| Aspect | EC2 | S3 + CloudFront |
|--------|-----|-----------------|
| **Cost (storage)** | EBS: $0.10/GB | S3: $0.023/GB |
| **Cost (transfer)** | $0.09/GB | CloudFront: $0.085/GB |
| **Scalability** | Limited by instance | Unlimited |
| **Availability** | Depends on instance | 99.99% |
| **Management** | Server maintenance | None |

> **SAA Exam Tip:** S3 + CloudFront is the most cost-effective and scalable solution for static content. Don't serve static files from EC2.

---

## Step 6: Origin Access Control (OAC)

### The Problem: Direct S3 Access

If S3 bucket is public, users can bypass CloudFront:

```mermaid
flowchart LR
    User((User))

    subgraph Bad["Without OAC"]
        User -->|"Bypass!"| S3a["S3<br>Public"]
        User --> CF1["CloudFront"] --> S3a
        Note1["Problems:<br>- No caching benefit<br>- No CloudFront security<br>- Direct S3 costs"]
    end

    style Bad fill:#ffcdd2,color:#000
    style Note1 fill:#ffcdd2,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### The Solution: Origin Access Control

```mermaid
flowchart LR
    User((User))

    subgraph Good["With OAC"]
        User --> CF2["CloudFront"]
        CF2 -->|"OAC credentials"| S3b["S3<br>Private"]
        User -.->|"Blocked!"| S3b
    end

    style Good fill:#c8e6c9,color:#000
    style S3b fill:#fff9c4,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

**How it works:**
1. S3 bucket is private (no public access)
2. CloudFront uses OAC to authenticate to S3
3. Users can only access through CloudFront

### S3 Bucket Policy with OAC

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::techbooks-static/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::123456789:distribution/EDFDVBD6EXAMPLE"
        }
      }
    }
  ]
}
```

> **SAA Exam Tip:** OAC (Origin Access Control) is the modern replacement for OAI (Origin Access Identity). OAC supports more features including SSE-KMS encryption. Always use OAC for new distributions.

---

## Step 7: Cache Behaviors and TTL

### Time To Live (TTL)

**TTL** determines how long CloudFront caches content before checking the origin.

```mermaid
flowchart LR
    subgraph TTL["TTL Timeline"]
        Fetch["Fetch from origin<br>Cache content"]
        Serve["Serve from cache"]
        Expire["TTL expires"]
        Revalidate["Revalidate with origin"]
    end

    Fetch --> Serve
    Serve -->|"TTL period"| Expire
    Expire --> Revalidate
    Revalidate -->|"If unchanged"| Serve

    style Serve fill:#c8e6c9,color:#000
    style Expire fill:#fff9c4,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### TTL Settings

| Setting | Description | Default |
|---------|-------------|---------|
| **Minimum TTL** | CloudFront won't cache shorter | 0 seconds |
| **Maximum TTL** | CloudFront won't cache longer | 31536000 (1 year) |
| **Default TTL** | Used if origin doesn't specify | 86400 (1 day) |

### Origin Headers Override CloudFront

If origin sends `Cache-Control` header, it overrides default TTL:

| Origin Header | CloudFront Behavior |
|---------------|---------------------|
| `Cache-Control: max-age=3600` | Cache for 1 hour |
| `Cache-Control: no-cache` | Don't cache |
| `Cache-Control: no-store` | Don't cache |
| No header | Use CloudFront default TTL |

### TechBooks TTL Strategy

| Content | TTL | WHY |
|---------|-----|-----|
| `/static/js/app.v2.js` | 1 year | Versioned filename, immutable |
| `/static/images/*` | 1 year | Rarely change |
| `/products/*.html` | 5 minutes | Product data changes |
| `/api/*` | 0 (no cache) | Personalized, real-time |

**Versioning strategy:** Include hash in filename (`app.a1b2c3.js`). Change hash when content changes. Set TTL to 1 year.

> **SAA Exam Tip:** Long TTLs for static, versioned content. Short or no TTL for dynamic content. Use versioned filenames to force cache invalidation.

---

## Step 8: Cache Invalidation

### WHY Invalidation?

Sometimes you need to clear cached content immediately:
- Fixed a bug in JavaScript
- Updated product images
- Security issue in cached content

### Invalidation Methods

| Method | When to Use | Cost |
|--------|-------------|------|
| **Versioned URLs** | Best practice, avoid invalidation | Free |
| **Invalidation API** | Emergency, unversioned content | First 1000 free/month |
| **Low TTL** | Frequently changing content | Free, but less caching |

```mermaid
flowchart TB
    subgraph Methods["Cache Invalidation Strategies"]
        subgraph Best["Best: Versioned URLs"]
            V1["app.v1.js"] -->|"Deploy"| V2["app.v2.js"]
            Note1["Different URL = fresh cache"]
        end

        subgraph OK["OK: Invalidation API"]
            API["aws cloudfront create-invalidation"]
            Note2["Clears /path/* from all edges"]
        end

        subgraph Avoid["Avoid: Short TTL"]
            Short["TTL: 60 seconds"]
            Note3["Constant origin requests"]
        end
    end

    style Best fill:#c8e6c9,color:#000
    style OK fill:#fff9c4,color:#000
    style Avoid fill:#ffcdd2,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### Invalidation Example

```bash
# Invalidate specific files
aws cloudfront create-invalidation \
    --distribution-id E1234567890 \
    --paths "/static/css/main.css" "/static/js/app.js"

# Invalidate entire directory
aws cloudfront create-invalidation \
    --distribution-id E1234567890 \
    --paths "/static/*"

# Invalidate everything (expensive!)
aws cloudfront create-invalidation \
    --distribution-id E1234567890 \
    --paths "/*"
```

> **SAA Exam Tip:** Invalidation takes time to propagate to all edges (minutes). Versioned URLs provide instant "invalidation" without waiting or paying.

---

## Step 9: Route 53 DNS

### What is Route 53?

**Route 53** is AWS's DNS service with advanced routing capabilities.

```mermaid
flowchart TB
    User((User)) -->|"techbooks.com"| R53["Route 53"]

    subgraph Routing["Routing Options"]
        Simple["Simple<br>One IP/record"]
        Weighted["Weighted<br>% traffic split"]
        Latency["Latency<br>Lowest latency"]
        Failover["Failover<br>Primary/backup"]
        Geo["Geolocation<br>By user location"]
    end

    R53 --> Simple
    R53 --> Weighted
    R53 --> Latency
    R53 --> Failover
    R53 --> Geo

    style R53 fill:#e3f2fd,color:#000
    style Routing fill:#fff9c4,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### Routing Policies

| Policy | Use Case | Example |
|--------|----------|---------|
| **Simple** | Single resource | One ALB |
| **Weighted** | A/B testing, migration | 90% v1, 10% v2 |
| **Latency** | Multi-region, lowest latency | Route to nearest region |
| **Failover** | Disaster recovery | Primary + backup |
| **Geolocation** | Compliance, localization | EU users → EU servers |
| **Geoproximity** | Fine-tuned geo control | Bias traffic to regions |
| **Multivalue** | Simple load balancing | Up to 8 healthy records |

### TechBooks DNS Configuration

For now (single region), we use **Simple routing** with an **Alias record**:

```
techbooks.com → CloudFront distribution (d1234.cloudfront.net)
```

**WHY Alias record?**
- Works with zone apex (naked domain: techbooks.com)
- No charge for DNS queries to AWS resources
- Automatically handles IP changes

> **SAA Exam Tip:** Alias records are Route 53-specific. They're free for queries to AWS resources and work with zone apex. Use Alias instead of CNAME for AWS resources.

---

## Step 10: SSL/TLS with ACM

### AWS Certificate Manager (ACM)

**ACM** provides free SSL/TLS certificates for AWS services.

```mermaid
flowchart LR
    subgraph SSL["SSL/TLS Termination Options"]
        subgraph Edge["CloudFront Edge"]
            CF["CloudFront"]
            Cert1["ACM Cert<br>*.techbooks.com"]
        end

        subgraph Origin["Origin (ALB)"]
            ALB["ALB"]
            Cert2["ACM Cert<br>(optional)"]
        end
    end

    User((User)) -->|"HTTPS"| CF
    CF -->|"HTTP or HTTPS"| ALB

    style Edge fill:#c8e6c9,color:#000
    style Origin fill:#fff9c4,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### SSL Configuration Options

| From User to CloudFront | From CloudFront to Origin | Use Case |
|-------------------------|---------------------------|----------|
| HTTPS (required) | HTTP | Common, simple |
| HTTPS | HTTPS (ACM cert) | End-to-end encryption |
| HTTPS | HTTPS (self-signed) | Custom requirements |

### ACM Key Points

| Feature | Detail |
|---------|--------|
| **Cost** | Free for AWS services |
| **Renewal** | Automatic |
| **Validation** | DNS or Email |
| **CloudFront region** | Must be **us-east-1** |
| **ALB region** | Same region as ALB |

> **SAA Exam Tip:** ACM certificates for CloudFront must be created in us-east-1 (N. Virginia). This is a common exam question!

---

## Step 11: CloudFront Security Features

### Additional Security Options

| Feature | What It Does | Use Case |
|---------|--------------|----------|
| **AWS WAF** | Web Application Firewall | Block SQL injection, XSS |
| **Geo Restriction** | Block/allow by country | Compliance, licensing |
| **Signed URLs** | Time-limited, secure URLs | Paid content, downloads |
| **Signed Cookies** | Multiple file access | Premium subscriptions |
| **Field-Level Encryption** | Encrypt specific fields | Credit cards, PII |

```mermaid
flowchart TB
    User((User)) --> WAF["AWS WAF<br>Block malicious requests"]
    WAF --> Geo["Geo Restriction<br>Allowed countries only"]
    Geo --> CF["CloudFront"]

    subgraph Protected["Protected Content"]
        Signed["Signed URLs<br>Premium e-books"]
    end

    CF --> Protected

    style WAF fill:#ffcdd2,color:#000
    style Geo fill:#fff9c4,color:#000
    style Protected fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### TechBooks Security Configuration

| Feature | Status | WHY |
|---------|--------|-----|
| AWS WAF | Optional (later) | Cost vs threat level |
| Geo Restriction | None | Selling globally |
| Signed URLs | None (yet) | No premium content yet |
| HTTPS | Required | Security best practice |

> **SAA Exam Tip:** Signed URLs for single files, Signed Cookies for multiple files. WAF integrates with CloudFront for Layer 7 protection.

---

## Phase 5 Complete Architecture

```mermaid
flowchart TB
    User((User)) --> R53["Route 53<br>techbooks.com"]
    R53 --> CF["CloudFront<br>Edge Locations"]

    subgraph CloudFront["CloudFront Distribution"]
        Behavior1["/static/*<br>→ S3"]
        Behavior2["/*<br>→ ALB"]
    end

    CF --> Behavior1 --> S3["S3 Bucket<br>techbooks-static<br>Private + OAC"]
    CF --> Behavior2 --> ALB

    subgraph VPC["VPC: 10.0.0.0/16"]
        ALB["Application<br>Load Balancer"]

        subgraph AZ1["AZ: us-east-1a"]
            EC2a["EC2"]
            RDSPrimary["RDS Primary"]
        end

        subgraph AZ2["AZ: us-east-1b"]
            EC2b["EC2"]
            RDSStandby["RDS Standby"]
        end

        ALB --> EC2a
        ALB --> EC2b
        EC2a --> RDSPrimary
        EC2b --> RDSPrimary
        RDSPrimary <--> RDSStandby
    end

    style CloudFront fill:#e3f2fd,color:#000
    style S3 fill:#c8e6c9,color:#000
    style VPC fill:#e3f2fd,color:#000
    style AZ1 fill:#c8e6c9,color:#000
    style AZ2 fill:#c8e6c9,color:#000
    style RDSPrimary fill:#bbdefb,color:#000
    style RDSStandby fill:#bbdefb,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

---

## Key SAA Exam Concepts from Phase 5

### Must-Know Topics

1. **CloudFront Basics**
   - 450+ edge locations globally
   - Regional Edge Caches between edges and origin
   - Distribution = configuration for CDN

2. **Origins**
   - S3 bucket (use OAC for private buckets)
   - ALB, EC2, custom HTTP servers
   - Multiple origins with cache behaviors

3. **Caching**
   - TTL controls cache duration
   - Origin headers can override TTL
   - Versioned URLs > invalidation

4. **Security**
   - OAC for private S3 access (not OAI)
   - ACM certificates (us-east-1 for CloudFront!)
   - Signed URLs/Cookies for private content

5. **Route 53**
   - Alias records for AWS resources (free, zone apex)
   - Routing policies: simple, weighted, latency, failover, geolocation

---

## Performance Impact

| Metric | Before (Phase 4) | After (Phase 5) | Improvement |
|--------|------------------|-----------------|-------------|
| Tokyo latency | ~250ms | ~30ms | 88% faster |
| London latency | ~120ms | ~20ms | 83% faster |
| NYC latency | ~20ms | ~10ms | 50% faster |
| Origin load | 100% | ~30% | 70% reduction |
| Static file cost | EC2 transfer | CloudFront | ~40% cheaper |

---

## Cost Analysis

| Component | Phase 4 Cost | Phase 5 Cost | Notes |
|-----------|--------------|--------------|-------|
| EC2 (t3.micro x2) | ~$16/month | ~$16/month | No change |
| RDS Multi-AZ | ~$24/month | ~$24/month | No change |
| ALB | ~$20/month | ~$20/month | No change |
| CloudFront | $0 | ~$15/month | Data transfer |
| S3 | $0 | ~$3/month | Static assets |
| Route 53 | ~$0.50 | ~$0.50 | Hosted zone |
| **Total** | ~$65/month | ~$83/month | +$18 |

**WHY it's worth it:**
- 80%+ latency reduction for global users
- Reduced origin load
- Better user experience globally
- DDoS protection included
- Foundation for global expansion

---

## What's Coming in Phase 6?

**Business trigger:** TechBooks is now profitable! The founder wants to reduce costs and add new features. The development team has grown and wants to ship features faster.

**Next decisions:**
- Serverless components (Lambda) for specific features
- ElastiCache for session/database caching
- Consider S3 for user uploads
- Optimize costs with Reserved Instances

---

## Hands-On Challenge

Before moving to Phase 6:

1. Create an S3 bucket for static assets (`techbooks-static`)
2. Upload your CSS, JS, and images to S3
3. Create a CloudFront distribution with:
   - S3 origin for `/static/*`
   - ALB origin for everything else
   - OAC for S3 access
4. Request an ACM certificate in us-east-1
5. Update Route 53 to point to CloudFront (Alias record)
6. Test from different geographic locations (use VPN or online tools)

**Verification:**
- `techbooks.com/static/logo.png` serves from S3 via CloudFront
- `techbooks.com/` serves from ALB via CloudFront
- Check CloudFront cache hit rate in CloudWatch
- Verify HTTPS works with your domain
