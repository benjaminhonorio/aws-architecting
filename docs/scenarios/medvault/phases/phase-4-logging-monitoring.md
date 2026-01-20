# Phase 4: Logging & Monitoring

## The Story So Far

MedVault has a robust security foundation: IAM with least privilege, encryption with KMS, and
network security with VPC endpoints and WAF. But the compliance team needs proof that controls are
working.

## Business Trigger

The HIPAA auditor arrives for a mock assessment:

> "Show me who accessed patient record #12345 in the last 30 days. I need to see the user,
> timestamp, source IP, and what action they performed."

The compliance consultant adds:

> "We also need to prove that all S3 buckets are encrypted and all security groups don't allow
> unrestricted access. Can you demonstrate continuous compliance monitoring?"

## Architecture Decision

**Decision**: Implement comprehensive logging with CloudTrail for API (Application Programming
Interface) activity, AWS Config for compliance monitoring, and CloudWatch for metrics and alerting.

### Logging Strategy

Each logging requirement maps to a specific AWS service. Understanding which service answers which
question is critical for the exam - CloudTrail tracks activity, Config tracks state, and CloudWatch
handles metrics and application logs:

| Requirement                 | AWS Service     |
| --------------------------- | --------------- |
| API activity (who did what) | CloudTrail      |
| Resource compliance         | AWS Config      |
| Application logs            | CloudWatch Logs |
| Metrics and alerts          | CloudWatch      |
| Automated response          | EventBridge     |

## Key Concepts for SAA Exam

### AWS CloudTrail

```mermaid
flowchart LR
    subgraph Sources["API Sources"]
        CONSOLE["Console"]
        CLI["CLI<br>(Command Line Interface)"]
        SDK["SDK<br>(Software Development Kit)"]
        SVC["AWS Services"]
    end

    subgraph CT["CloudTrail"]
        TRAIL["Trail"]
    end

    subgraph Destinations["Destinations"]
        S3["S3<br>(required)"]
        CW["CloudWatch<br>Logs"]
        EB["EventBridge"]
    end

    CONSOLE & CLI & SDK & SVC --> CT
    CT --> S3
    CT --> CW
    CT --> EB

    style Sources fill:#e3f2fd,color:#000
    style CT fill:#fff9c4,color:#000
    style Destinations fill:#c8e6c9,color:#000
    style CONSOLE fill:#fff,color:#000
    style CLI fill:#fff,color:#000
    style SDK fill:#fff,color:#000
    style SVC fill:#fff,color:#000
    style TRAIL fill:#fff,color:#000
    style S3 fill:#fff,color:#000
    style CW fill:#fff,color:#000
    style EB fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### CloudTrail Event Types

CloudTrail categorizes events by their scope and impact. Management events are logged by default
because they represent control plane operations. Data events (data plane) generate high volume and
cost extra, so enable them selectively:

| Event Type            | What's Logged                                                | Default | Cost           |
| --------------------- | ------------------------------------------------------------ | ------- | -------------- |
| **Management Events** | API calls that manage resources (CreateBucket, RunInstances) | Yes     | Free (90 days) |
| **Data Events**       | Data operations (S3 GetObject, Lambda Invoke)                | No      | Extra cost     |
| **Insights Events**   | Unusual API activity                                         | No      | Extra cost     |

> **Exam Tip**: Management events are logged by default and retained for 90 days in Event History.
> For longer retention or data events, create a trail to S3.

### Management vs Data Events

```mermaid
flowchart TB
    subgraph Mgmt["Management Events"]
        M1["CreateBucket"]
        M2["PutBucketPolicy"]
        M3["RunInstances"]
        M4["CreateRole"]
    end

    subgraph Data["Data Events"]
        D1["GetObject"]
        D2["PutObject"]
        D3["Invoke (Lambda)"]
        D4["GetItem (DynamoDB)"]
    end

    style Mgmt fill:#c8e6c9,color:#000
    style Data fill:#fff9c4,color:#000
    style M1 fill:#fff,color:#000
    style M2 fill:#fff,color:#000
    style M3 fill:#fff,color:#000
    style M4 fill:#fff,color:#000
    style D1 fill:#fff,color:#000
    style D2 fill:#fff,color:#000
    style D3 fill:#fff,color:#000
    style D4 fill:#fff,color:#000
```

> **Exam Tip**: Data events are high-volume and cost money. Enable selectively (e.g., specific S3
> buckets with PHI).

### CloudTrail Log Integrity

Validate that logs haven't been tampered with:

```mermaid
flowchart LR
    CT["CloudTrail"] -->|"Log file"| S3["S3"]
    CT -->|"Digest file<br>(hourly)"| S3
    S3 -->|"Validate"| CLI["aws cloudtrail<br>validate-logs"]

    style CT fill:#e3f2fd,color:#000
    style S3 fill:#c8e6c9,color:#000
    style CLI fill:#fff9c4,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

**Digest files**:

- Created hourly with hash of all log files
- Signed with AWS private key
- Allows detection of tampering or deletion

> **Exam Tip**: Log file validation ensures integrity - critical for compliance and forensics.

### Organization Trail

```mermaid
flowchart TB
    subgraph Org["AWS Organization"]
        MGMT["Management<br>Account"]
        subgraph OUs["OUs (Organizational Units)"]
            PROD["Prod OU"]
            DEV["Dev OU"]
        end
    end

    subgraph Trail["Organization Trail"]
        OT["Logs ALL accounts<br>automatically"]
    end

    CENTRAL["Central S3<br>Bucket"]

    MGMT --> Trail
    PROD & DEV --> Trail
    Trail --> CENTRAL

    style Org fill:#e3f2fd,color:#000
    style OUs fill:#fff9c4,color:#000
    style Trail fill:#c8e6c9,color:#000
    style MGMT fill:#fff,color:#000
    style PROD fill:#fff,color:#000
    style DEV fill:#fff,color:#000
    style OT fill:#fff,color:#000
    style CENTRAL fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

> **Exam Tip**: Organization trails automatically log all member accounts. Great for centralized
> security monitoring.

### AWS Config

```mermaid
flowchart TB
    subgraph Config["AWS Config"]
        REC["Configuration<br>Recorder"]
        RULES["Config Rules"]
        CONF["Conformance<br>Packs"]
    end

    subgraph Resources["AWS Resources"]
        EC2["EC2"]
        S3["S3"]
        SG["Security<br>Groups"]
    end

    subgraph Output["Output"]
        HIST["Configuration<br>History"]
        COMP["Compliance<br>Status"]
        SNS["SNS<br>Notifications"]
    end

    Resources --> REC
    REC --> RULES --> COMP
    REC --> HIST
    RULES --> SNS
    CONF --> RULES

    style Config fill:#e3f2fd,color:#000
    style Resources fill:#c8e6c9,color:#000
    style Output fill:#fff9c4,color:#000
    style REC fill:#fff,color:#000
    style RULES fill:#fff,color:#000
    style CONF fill:#fff,color:#000
    style EC2 fill:#fff,color:#000
    style S3 fill:#fff,color:#000
    style SG fill:#fff,color:#000
    style HIST fill:#fff,color:#000
    style COMP fill:#fff,color:#000
    style SNS fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### CloudTrail vs Config

These two services complement each other but answer different questions. CloudTrail is verb-focused
(what happened?), while Config is noun-focused (what exists?). For compliance, you typically need
both:

| Feature               | CloudTrail                | AWS Config                   |
| --------------------- | ------------------------- | ---------------------------- |
| **Question answered** | Who did what, when?       | What's the current state?    |
| **Focus**             | API activity (events)     | Resource configuration       |
| **Use case**          | Security investigation    | Compliance monitoring        |
| **Example**           | "Who deleted the bucket?" | "Are all buckets encrypted?" |

> **Exam Tip**: CloudTrail = API activity (verb-focused). Config = resource state (noun-focused).

### Config Rules

```mermaid
flowchart LR
    subgraph Types["Config Rule Types"]
        AWS["AWS Managed<br>Rules"]
        CUSTOM["Custom Rules<br>(Lambda)"]
    end

    subgraph Examples["Example Rules"]
        E1["s3-bucket-ssl-requests-only"]
        E2["encrypted-volumes"]
        E3["restricted-ssh"]
        E4["iam-password-policy"]
    end

    AWS --> E1 & E2 & E3 & E4

    style Types fill:#e3f2fd,color:#000
    style Examples fill:#c8e6c9,color:#000
    style AWS fill:#fff,color:#000
    style CUSTOM fill:#fff,color:#000
    style E1 fill:#fff,color:#000
    style E2 fill:#fff,color:#000
    style E3 fill:#fff,color:#000
    style E4 fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

**Trigger types**:

- **Configuration changes**: Evaluate when resource changes
- **Periodic**: Evaluate on schedule (1hr, 3hr, 6hr, 12hr, 24hr)

### Conformance Packs

Conformance packs bundle multiple Config rules into a single deployment, aligned to specific
compliance frameworks. Instead of configuring 50+ individual rules, deploy one pack that covers your
regulatory requirements:

| Pack                                       | Use Case         |
| ------------------------------------------ | ---------------- |
| **Operational Best Practices for HIPAA**   | Healthcare       |
| **CIS AWS Foundations Benchmark**          | General security |
| **Operational Best Practices for PCI-DSS** | Payment card     |
| **NIST 800-53**                            | Government       |

> **Abbreviations**: CIS = Center for Internet Security, PCI-DSS = Payment Card Industry Data
> Security Standard, NIST = National Institute of Standards and Technology

> **Exam Tip**: Conformance packs simplify compliance - one deployment covers many rules.

### Remediation

Automatically fix non-compliant resources:

```mermaid
flowchart LR
    RESOURCE["S3 Bucket<br>(no encryption)"]
    RULE["Config Rule:<br>s3-bucket-encryption"]
    SSM["SSM (Systems Manager)<br>Automation Document"]
    FIX["Enable<br>Encryption"]

    RESOURCE -->|"Evaluated"| RULE
    RULE -->|"NON_COMPLIANT"| SSM
    SSM -->|"Remediate"| FIX

    style RESOURCE fill:#ffcdd2,color:#000
    style RULE fill:#e3f2fd,color:#000
    style SSM fill:#fff9c4,color:#000
    style FIX fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### CloudWatch Logs

```mermaid
flowchart TB
    subgraph Sources["Log Sources"]
        EC2["EC2<br>(CW Agent)"]
        LAMBDA["Lambda"]
        CT["CloudTrail"]
        VPC["VPC Flow<br>Logs"]
    end

    subgraph CWL["CloudWatch Logs"]
        LG["Log Groups"]
        LS["Log Streams"]
        MF["Metric Filters"]
        SUB["Subscriptions"]
    end

    subgraph Output["Output"]
        ALARM["CloudWatch<br>Alarms"]
        KIN["Kinesis"]
        LAMBDA2["Lambda"]
        ES["OpenSearch"]
    end

    EC2 & LAMBDA & CT & VPC --> LG --> LS
    LG --> MF --> ALARM
    LG --> SUB --> KIN & LAMBDA2 & ES

    style Sources fill:#e3f2fd,color:#000
    style CWL fill:#fff9c4,color:#000
    style Output fill:#c8e6c9,color:#000
    style EC2 fill:#fff,color:#000
    style LAMBDA fill:#fff,color:#000
    style CT fill:#fff,color:#000
    style VPC fill:#fff,color:#000
    style LG fill:#fff,color:#000
    style LS fill:#fff,color:#000
    style MF fill:#fff,color:#000
    style SUB fill:#fff,color:#000
    style ALARM fill:#fff,color:#000
    style KIN fill:#fff,color:#000
    style LAMBDA2 fill:#fff,color:#000
    style ES fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### Metric Filters

Extract metrics from logs:

```
# Filter pattern for failed console logins
{ $.eventName = "ConsoleLogin" && $.errorMessage = "Failed authentication" }
```

> **Exam Tip**: Metric filters convert log data into CloudWatch metrics for alarming.

### CloudWatch Alarms

```mermaid
flowchart LR
    METRIC["Metric"] --> ALARM["Alarm"]
    ALARM -->|"ALARM state"| SNS["SNS Topic"]
    SNS --> EMAIL["Email"]
    SNS --> LAMBDA["Lambda<br>(auto-remediate)"]

    style METRIC fill:#e3f2fd,color:#000
    style ALARM fill:#ffcdd2,color:#000
    style SNS fill:#fff9c4,color:#000
    style EMAIL fill:#fff,color:#000
    style LAMBDA fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

**Alarm states**: OK → ALARM → INSUFFICIENT_DATA

### EventBridge (CloudWatch Events)

```mermaid
flowchart LR
    subgraph Sources["Event Sources"]
        AWS["AWS Services"]
        CT["CloudTrail"]
        CUSTOM["Custom Apps"]
    end

    EB["EventBridge"]

    subgraph Targets["Targets"]
        LAMBDA["Lambda"]
        SNS["SNS"]
        SQS["SQS"]
        SSM["SSM"]
    end

    AWS & CT & CUSTOM --> EB --> LAMBDA & SNS & SQS & SSM

    style Sources fill:#e3f2fd,color:#000
    style EB fill:#fff9c4,color:#000
    style Targets fill:#c8e6c9,color:#000
    style AWS fill:#fff,color:#000
    style CT fill:#fff,color:#000
    style CUSTOM fill:#fff,color:#000
    style LAMBDA fill:#fff,color:#000
    style SNS fill:#fff,color:#000
    style SQS fill:#fff,color:#000
    style SSM fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

**Example: Alert on root login**:

```json
{
  "source": ["aws.signin"],
  "detail-type": ["AWS Console Sign In via CloudTrail"],
  "detail": {
    "userIdentity": {
      "type": ["Root"]
    }
  }
}
```

## MedVault Logging Architecture

```mermaid
flowchart TB
    subgraph Apps["Applications"]
        EC2["EC2 Apps"]
        LAMBDA["Lambda"]
    end

    subgraph Logging["Centralized Logging"]
        CT["CloudTrail<br>Organization Trail"]
        CONFIG["AWS Config<br>HIPAA Pack"]
        CWL["CloudWatch<br>Logs"]
        VPC_FL["VPC Flow<br>Logs"]
    end

    subgraph Storage["Secure Storage"]
        S3_LOG["S3 Log Bucket<br>SSE-KMS<br>Object Lock"]
    end

    subgraph Alerting["Alerting"]
        EB["EventBridge"]
        CW_ALARM["CloudWatch<br>Alarms"]
        SNS["SNS"]
    end

    subgraph Response["Response"]
        TEAM["Security Team"]
        LAMBDA_R["Lambda<br>Auto-remediate"]
    end

    EC2 & LAMBDA --> CWL
    Apps --> CT
    EC2 --> VPC_FL
    CT & CWL & VPC_FL --> S3_LOG
    CT --> EB --> SNS --> TEAM
    CONFIG --> CW_ALARM --> SNS
    EB --> LAMBDA_R

    style Apps fill:#e3f2fd,color:#000
    style Logging fill:#fff9c4,color:#000
    style Storage fill:#c8e6c9,color:#000
    style Alerting fill:#ffcdd2,color:#000
    style Response fill:#bbdefb,color:#000
    style EC2 fill:#fff,color:#000
    style LAMBDA fill:#fff,color:#000
    style CT fill:#fff,color:#000
    style CONFIG fill:#fff,color:#000
    style CWL fill:#fff,color:#000
    style VPC_FL fill:#fff,color:#000
    style S3_LOG fill:#fff,color:#000
    style EB fill:#fff,color:#000
    style CW_ALARM fill:#fff,color:#000
    style SNS fill:#fff,color:#000
    style TEAM fill:#fff,color:#000
    style LAMBDA_R fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### MedVault Logging Decisions

Each logging requirement has a specific configuration tailored to HIPAA compliance. Note that data
events are enabled only for the PHI bucket (not all S3 buckets) to control costs while maintaining
compliance:

| Requirement           | Solution    | Configuration                            |
| --------------------- | ----------- | ---------------------------------------- |
| API audit trail       | CloudTrail  | Management + Data events (S3 PHI bucket) |
| Log integrity         | CloudTrail  | Log file validation enabled              |
| Compliance monitoring | Config      | HIPAA conformance pack                   |
| Real-time alerts      | EventBridge | Root login, policy changes               |
| Log retention         | S3          | 7 years, Object Lock                     |
| Log protection        | KMS         | Customer managed key                     |

### Security Alerts Configured

| Event                      | Response                 |
| -------------------------- | ------------------------ |
| Root account login         | Page security team       |
| IAM policy change          | Alert + auto-review      |
| S3 public access           | Auto-remediate (block)   |
| Security group 0.0.0.0/0   | Auto-remediate (remove)  |
| KMS key deletion scheduled | Alert + require approval |

## What Could Go Wrong?

Logging is comprehensive. The auditor can now trace any access to patient data. But as MedVault
prepares to scale:

> "You're running everything in one account. What happens when you have 50 developers? How do you
> prevent someone in dev from accidentally accessing production patient data?"

Time for multi-account strategy.

## Exam Tips

- **CloudTrail = who did what** - API activity logging
- **Config = what's the state** - Resource compliance
- **Management events free 90 days** - Data events cost extra
- **Organization trail for all accounts** - Central logging
- **Log file validation for integrity** - Digest files
- **Conformance packs for compliance** - HIPAA, PCI, CIS
- **Config can auto-remediate** - SSM Automation documents
- **EventBridge for real-time** - Trigger Lambda/SNS on events

## SAA Exam Concepts

### Must-Know for This Phase

| Concept            | Key Points                                            |
| ------------------ | ----------------------------------------------------- |
| CloudTrail         | Management vs Data events, 90-day free, trails to S3  |
| Log Integrity      | Digest files, validation, tamper detection            |
| Organization Trail | All accounts, central bucket                          |
| Config             | Resource state, rules, conformance packs, remediation |
| Config Rules       | AWS managed, custom (Lambda), triggers                |
| CloudWatch Logs    | Log groups/streams, metric filters, subscriptions     |
| EventBridge        | Event patterns, targets, real-time response           |
