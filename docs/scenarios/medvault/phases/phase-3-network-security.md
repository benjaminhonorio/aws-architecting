# Phase 3: Network Security

## The Story So Far

MedVault has strong IAM controls and comprehensive encryption. Patient data is encrypted at rest
with KMS and in transit with TLS. Database credentials rotate automatically via Secrets Manager.

## Business Trigger

During a security review, the network engineer raises a concern:

> "I traced the network path for our S3 uploads. Even though we're using HTTPS, the traffic goes out
> our NAT Gateway, over the internet, and back into AWS. That's adding latency and costs - plus it's
> technically traversing the public internet."

The compliance consultant adds:

> "Some of our healthcare clients require that PHI never traverses the public internet, even
> encrypted. Can we keep all AWS API traffic within the AWS network?"

The CISO also wants protection at the edge:

> "We're exposing APIs to third-party healthcare systems. We need DDoS protection and the ability to
> block malicious requests."

## Architecture Decision

**Decision**: Implement VPC endpoints for private AWS API access, deploy AWS WAF for application
protection, and enable AWS Shield for DDoS mitigation.

## Key Concepts for SAA Exam

### VPC Endpoints

```mermaid
flowchart TB
    subgraph VPC["VPC"]
        EC2["EC2"]

        subgraph Endpoints["VPC Endpoints"]
            GW["Gateway<br>Endpoint"]
            INT["Interface<br>Endpoint"]
        end
    end

    subgraph AWS["AWS Services"]
        S3["S3"]
        DDB["DynamoDB"]
        KMS["KMS"]
        SM["Secrets<br>Manager"]
    end

    EC2 -->|"Free"| GW --> S3 & DDB
    EC2 -->|"~$0.01/hr"| INT --> KMS & SM

    style VPC fill:#e3f2fd,color:#000
    style Endpoints fill:#fff9c4,color:#000
    style AWS fill:#c8e6c9,color:#000
    style EC2 fill:#fff,color:#000
    style GW fill:#fff,color:#000
    style INT fill:#fff,color:#000
    style S3 fill:#fff,color:#000
    style DDB fill:#fff,color:#000
    style KMS fill:#fff,color:#000
    style SM fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### Gateway vs Interface Endpoints

| Feature            | Gateway Endpoint  | Interface Endpoint                  |
| ------------------ | ----------------- | ----------------------------------- |
| **Services**       | S3, DynamoDB only | 100+ services                       |
| **Cost**           | Free              | ~$0.01/hr + data                    |
| **Implementation** | Route table entry | ENI in subnet                       |
| **DNS**            | Uses S3/DDB DNS   | Private DNS                         |
| **Security**       | Endpoint policies | Security groups + endpoint policies |
| **HA**             | Automatic         | Deploy in multiple AZs              |

> **Exam Tip**: Gateway endpoints are **free** and only work for S3 and DynamoDB. Everything else
> uses interface endpoints.

### Gateway Endpoint Deep Dive

```mermaid
flowchart LR
    subgraph VPC["VPC"]
        subgraph Private["Private Subnet"]
            EC2["EC2"]
        end
        RT["Route Table"]
        GW["Gateway<br>Endpoint"]
    end

    S3["S3"]

    EC2 --> RT
    RT -->|"pl-xxxxx<br>(S3 prefix list)"| GW
    GW -->|"Private AWS<br>backbone"| S3

    style VPC fill:#e3f2fd,color:#000
    style Private fill:#fff9c4,color:#000
    style EC2 fill:#fff,color:#000
    style RT fill:#fff,color:#000
    style GW fill:#c8e6c9,color:#000
    style S3 fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

**Gateway Endpoint Policy** (restrict to specific bucket):

```json
{
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::medvault-patient-data/*"
    }
  ]
}
```

### Interface Endpoint Deep Dive

```mermaid
flowchart LR
    subgraph VPC["VPC"]
        subgraph Private["Private Subnet"]
            EC2["EC2"]
            ENI["ENI<br>10.0.1.50"]
        end
        SG["Security<br>Group"]
    end

    subgraph AWS["AWS"]
        KMS["KMS"]
    end

    EC2 -->|"kms.us-east-1.amazonaws.com<br>resolves to 10.0.1.50"| ENI
    SG --> ENI
    ENI -->|"PrivateLink"| KMS

    style VPC fill:#e3f2fd,color:#000
    style Private fill:#fff9c4,color:#000
    style AWS fill:#c8e6c9,color:#000
    style EC2 fill:#fff,color:#000
    style ENI fill:#fff,color:#000
    style SG fill:#ffcdd2,color:#000
    style KMS fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

**Key points**:

- Creates an ENI in your subnet with a private IP
- Enable **Private DNS** to auto-resolve AWS service DNS
- Apply security groups to control access
- Deploy in multiple AZs for HA

> **Exam Tip**: Interface endpoints use **PrivateLink** technology. The terms are sometimes used
> interchangeably.

### AWS PrivateLink

```mermaid
flowchart LR
    subgraph Consumer["Consumer VPC"]
        APP["Application"]
        INT["Interface<br>Endpoint"]
    end

    subgraph Provider["Provider VPC (or AWS)"]
        NLB["Network<br>Load Balancer"]
        SVC["Service"]
    end

    APP --> INT
    INT -->|"PrivateLink"| NLB --> SVC

    style Consumer fill:#e3f2fd,color:#000
    style Provider fill:#c8e6c9,color:#000
    style APP fill:#fff,color:#000
    style INT fill:#fff,color:#000
    style NLB fill:#fff,color:#000
    style SVC fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

**Use cases**:

- Access AWS services privately
- Expose your services to other VPCs/accounts
- SaaS provider connectivity

### Security Groups vs NACLs

```mermaid
flowchart TB
    subgraph SG["Security Groups"]
        SG1["Instance level"]
        SG2["Stateful"]
        SG3["Allow rules only"]
        SG4["All rules evaluated"]
    end

    subgraph NACL["Network ACLs"]
        N1["Subnet level"]
        N2["Stateless"]
        N3["Allow AND deny rules"]
        N4["Rules processed in order"]
    end

    style SG fill:#c8e6c9,color:#000
    style NACL fill:#fff9c4,color:#000
    style SG1 fill:#fff,color:#000
    style SG2 fill:#fff,color:#000
    style SG3 fill:#fff,color:#000
    style SG4 fill:#fff,color:#000
    style N1 fill:#fff,color:#000
    style N2 fill:#fff,color:#000
    style N3 fill:#fff,color:#000
    style N4 fill:#fff,color:#000
```

| Feature    | Security Groups                     | NACLs                         |
| ---------- | ----------------------------------- | ----------------------------- |
| Level      | Instance (ENI)                      | Subnet                        |
| State      | Stateful (return traffic automatic) | Stateless (must allow return) |
| Rules      | Allow only                          | Allow AND deny                |
| Evaluation | All rules                           | Lowest number first           |
| Default    | Deny all inbound                    | Allow all                     |

> **Exam Tip**: Security groups are **stateful** - if you allow inbound, outbound return traffic is
> automatic. NACLs are **stateless** - you must allow both directions.

### AWS WAF (Web Application Firewall)

```mermaid
flowchart LR
    subgraph Internet["Internet"]
        USERS["Users"]
        ATTACK["Attackers"]
    end

    subgraph AWS["AWS"]
        WAF["AWS WAF"]
        ALB["ALB"]
        APP["Application"]
    end

    USERS -->|"Legitimate"| WAF
    ATTACK -->|"Malicious"| WAF
    WAF -->|"Allow"| ALB --> APP
    WAF -->|"Block"| X["Blocked"]

    style Internet fill:#fff,color:#000
    style AWS fill:#e3f2fd,color:#000
    style USERS fill:#c8e6c9,color:#000
    style ATTACK fill:#ffcdd2,color:#000
    style WAF fill:#fff9c4,color:#000
    style ALB fill:#fff,color:#000
    style APP fill:#fff,color:#000
    style X fill:#ffcdd2,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### WAF Components

| Component         | Description                                                 |
| ----------------- | ----------------------------------------------------------- |
| **Web ACL**       | Container for rules, attached to ALB/CloudFront/API Gateway |
| **Rules**         | Conditions to match (IP, string, regex, rate)               |
| **Rule Groups**   | Reusable collections of rules                               |
| **Managed Rules** | Pre-built rules from AWS and partners                       |

### WAF Rule Types

```mermaid
flowchart TB
    subgraph Rules["WAF Rule Types"]
        IP["IP Match<br>Block known bad IPs"]
        STRING["String Match<br>Block SQL injection"]
        REGEX["Regex Match<br>Pattern matching"]
        SIZE["Size Constraint<br>Block oversized requests"]
        GEO["Geographic<br>Block countries"]
        RATE["Rate-based<br>Block flooding"]
    end

    style Rules fill:#e3f2fd,color:#000
    style IP fill:#fff,color:#000
    style STRING fill:#fff,color:#000
    style REGEX fill:#fff,color:#000
    style SIZE fill:#fff,color:#000
    style GEO fill:#fff,color:#000
    style RATE fill:#fff,color:#000
```

### AWS Managed Rules

Pre-configured rule groups:

| Rule Group              | Protection                   |
| ----------------------- | ---------------------------- |
| **Core Rule Set (CRS)** | OWASP Top 10                 |
| **SQL Database**        | SQL injection                |
| **Known Bad Inputs**    | Common attack patterns       |
| **IP Reputation**       | Known malicious IPs          |
| **Bot Control**         | Bot detection and management |

> **Exam Tip**: Managed rules are the fastest way to protect against common attacks. You can combine
> them with custom rules.

### WAF Rule Evaluation

Rules are evaluated in priority order:

```mermaid
flowchart TB
    REQ["Request"] --> R1{"Rule 1<br>Priority 1"}
    R1 -->|No match| R2{"Rule 2<br>Priority 2"}
    R2 -->|No match| R3{"Rule 3<br>Priority 3"}
    R3 -->|No match| DEF["Default Action"]

    R1 -->|Match: Block| BLOCK["Blocked"]
    R2 -->|Match: Allow| ALLOW["Allowed"]
    R3 -->|Match: Count| COUNT["Counted<br>(continues)"]

    style REQ fill:#e3f2fd,color:#000
    style BLOCK fill:#ffcdd2,color:#000
    style ALLOW fill:#c8e6c9,color:#000
    style COUNT fill:#fff9c4,color:#000
    style DEF fill:#fff,color:#000
    style R1 fill:#fff,color:#000
    style R2 fill:#fff,color:#000
    style R3 fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

> **Exam Tip**: **Count** action is useful for testing rules before blocking.

### AWS Shield

```mermaid
flowchart TB
    subgraph Shield["AWS Shield"]
        STD["Shield Standard<br>Free, automatic"]
        ADV["Shield Advanced<br>$3,000/month"]
    end

    subgraph STD_F["Standard Features"]
        S1["Layer 3/4 protection"]
        S2["Automatic mitigation"]
        S3["All AWS customers"]
    end

    subgraph ADV_F["Advanced Features"]
        A1["Layer 7 protection"]
        A2["DDoS Response Team (DRT)"]
        A3["Cost protection"]
        A4["Real-time metrics"]
        A5["WAF included"]
    end

    STD --- STD_F
    ADV --- ADV_F

    style Shield fill:#e3f2fd,color:#000
    style STD fill:#c8e6c9,color:#000
    style ADV fill:#fff9c4,color:#000
    style STD_F fill:#fff,color:#000
    style ADV_F fill:#fff,color:#000
    style S1 fill:#fff,color:#000
    style S2 fill:#fff,color:#000
    style S3 fill:#fff,color:#000
    style A1 fill:#fff,color:#000
    style A2 fill:#fff,color:#000
    style A3 fill:#fff,color:#000
    style A4 fill:#fff,color:#000
    style A5 fill:#fff,color:#000
```

### Shield Standard vs Advanced

| Feature                | Standard | Advanced                  |
| ---------------------- | -------- | ------------------------- |
| Cost                   | Free     | $3,000/month + data       |
| Layer 3/4 protection   | Yes      | Yes                       |
| Layer 7 protection     | No       | Yes                       |
| DDoS Response Team     | No       | Yes (24/7)                |
| Cost protection        | No       | Yes (refunds for scaling) |
| WAF included           | No       | Yes                       |
| Health-based detection | No       | Yes                       |

> **Exam Tip**: Shield Standard is automatic and free. Shield Advanced is for critical applications
> needing 24/7 DDoS support and cost protection.

### AWS Network Firewall

For VPC-level traffic inspection:

```mermaid
flowchart TB
    subgraph VPC["VPC"]
        subgraph FW_Sub["Firewall Subnet"]
            NFW["Network<br>Firewall"]
        end

        subgraph Private["Private Subnet"]
            EC2["EC2"]
        end

        IGW["Internet<br>Gateway"]
    end

    INET["Internet"] --> IGW --> NFW --> EC2

    style VPC fill:#e3f2fd,color:#000
    style FW_Sub fill:#ffcdd2,color:#000
    style Private fill:#c8e6c9,color:#000
    style NFW fill:#fff,color:#000
    style EC2 fill:#fff,color:#000
    style IGW fill:#fff,color:#000
    style INET fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

**Capabilities**:

- Stateful inspection
- Domain filtering
- IPS (Intrusion Prevention)
- Custom Suricata rules

> **Exam Tip**: Network Firewall is for VPC-level inspection. WAF is for HTTP/HTTPS application
> layer.

## MedVault Network Security Architecture

```mermaid
flowchart TB
    subgraph Internet["Internet"]
        USERS["Healthcare<br>Partners"]
    end

    subgraph Edge["Edge Protection"]
        CF["CloudFront"]
        WAF["AWS WAF"]
        SHIELD["Shield<br>Advanced"]
    end

    subgraph VPC["MedVault VPC"]
        subgraph Public["Public Subnet"]
            ALB["ALB"]
        end

        subgraph Private["Private Subnet"]
            EC2["Application"]
        end

        subgraph Endpoints["VPC Endpoints"]
            GW_S3["Gateway: S3"]
            INT_KMS["Interface: KMS"]
            INT_SM["Interface: Secrets Mgr"]
        end
    end

    subgraph AWS_SVC["AWS Services"]
        S3["S3"]
        KMS["KMS"]
        SM["Secrets Manager"]
    end

    USERS --> CF
    CF --> WAF --> SHIELD
    SHIELD --> ALB --> EC2
    EC2 --> GW_S3 --> S3
    EC2 --> INT_KMS --> KMS
    EC2 --> INT_SM --> SM

    style Internet fill:#fff,color:#000
    style Edge fill:#ffcdd2,color:#000
    style VPC fill:#e3f2fd,color:#000
    style Public fill:#c8e6c9,color:#000
    style Private fill:#fff9c4,color:#000
    style Endpoints fill:#bbdefb,color:#000
    style AWS_SVC fill:#c8e6c9,color:#000
    style USERS fill:#fff,color:#000
    style CF fill:#fff,color:#000
    style WAF fill:#fff,color:#000
    style SHIELD fill:#fff,color:#000
    style ALB fill:#fff,color:#000
    style EC2 fill:#fff,color:#000
    style GW_S3 fill:#fff,color:#000
    style INT_KMS fill:#fff,color:#000
    style INT_SM fill:#fff,color:#000
    style S3 fill:#fff,color:#000
    style KMS fill:#fff,color:#000
    style SM fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### MedVault Security Decisions

| Requirement         | Solution            | Rationale                      |
| ------------------- | ------------------- | ------------------------------ |
| S3 traffic private  | Gateway endpoint    | Free, no internet              |
| KMS/Secrets private | Interface endpoints | PrivateLink, PHI stays private |
| DDoS protection     | Shield Advanced     | Healthcare = critical          |
| Application attacks | WAF + managed rules | OWASP protection               |
| Edge security       | CloudFront + WAF    | Global edge, WAF integration   |

## What Could Go Wrong?

Network security is locked down. All AWS API traffic stays within AWS via VPC endpoints. WAF
protects against application attacks. But the compliance auditor needs more:

> "Great progress. Now I need to see audit logs. Can you prove who accessed what, when, and from
> where? HIPAA requires audit controls."

Time to implement logging and monitoring.

## Exam Tips

- **Gateway = S3/DynamoDB only, free** - Interface = everything else, costs money
- **PrivateLink = Interface endpoints** - Same technology
- **Security groups are stateful** - Return traffic automatic
- **NACLs are stateless** - Must allow both directions
- **WAF attaches to ALB/CloudFront/API Gateway** - Not EC2 directly
- **Shield Standard is free and automatic** - Everyone has it
- **Shield Advanced for DRT and cost protection** - $3K/month
- **Network Firewall for VPC inspection** - Domain filtering, IPS

## See Also

> **Related Learning:** For VPC fundamentals including CIDR planning, subnets, and Security Groups
> basics, see [TechBooks Phase 1: MVP Launch](/scenarios/techbooks/phases/phase-1-mvp-launch.md).

## SAA Exam Concepts

### Must-Know for This Phase

| Concept             | Key Points                                    |
| ------------------- | --------------------------------------------- |
| Gateway Endpoints   | S3, DynamoDB only, free, route table entry    |
| Interface Endpoints | 100+ services, ENI in subnet, security groups |
| PrivateLink         | Technology behind interface endpoints         |
| Security Groups     | Stateful, instance level, allow only          |
| NACLs               | Stateless, subnet level, allow + deny         |
| WAF                 | Web ACL, rules, managed rules, ALB/CF/APIGW   |
| Shield Standard     | Free, L3/L4, automatic                        |
| Shield Advanced     | $3K/mo, L7, DRT, cost protection              |
