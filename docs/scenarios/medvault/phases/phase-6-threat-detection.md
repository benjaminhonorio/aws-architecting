# Phase 6: Threat Detection & Response

## The Story So Far

MedVault has a mature security posture: strong IAM, encryption, network security, comprehensive
logging, and multi-account isolation. The platform is now serving thousands of healthcare providers.
But security is an ongoing battle.

## Business Trigger

The CISO receives a concerning report from a peer at another healthcare company:

> "We just had a breach. Attackers compromised developer credentials and exfiltrated patient data
> over three weeks before we noticed. They used our own APIs against us."

The board wants assurance:

> "How would we know if someone was trying to breach MedVault? What automated detection and response
> capabilities do we have?"

## Architecture Decision

**Decision**: Deploy AWS security services for continuous threat detection (GuardDuty), centralized
security visibility (Security Hub), data classification (Macie), and automated response
capabilities.

### Defense in Depth

Security requires multiple layers working together. Prevention stops attacks, detection identifies
breaches, response contains damage, and recovery restores operations. MedVault now completes the
detection and response layers:

| Layer      | Control                                  | Phase  |
| ---------- | ---------------------------------------- | ------ |
| Prevention | IAM, encryption, network                 | 1-3    |
| Detection  | CloudTrail, Config, GuardDuty            | 4, 6   |
| Response   | Automated remediation, incident response | 6      |
| Recovery   | Backup, DR (Disaster Recovery)           | Future |

## Key Concepts for SAA Exam

### Amazon GuardDuty

```mermaid
flowchart TB
    subgraph Sources["Data Sources"]
        CT["CloudTrail<br>Events"]
        VPC["VPC Flow<br>Logs"]
        DNS["DNS Logs"]
        S3["S3 Data<br>Events"]
        EKS["EKS (Elastic<br>Kubernetes Service)<br>Audit Logs"]
    end

    subgraph GD["GuardDuty"]
        ML["ML<br>(Machine Learning)"]
        TI["Threat Intelligence"]
        RULES["Anomaly Detection"]
    end

    subgraph Output["Output"]
        FINDINGS["Findings"]
        SEV["Severity:<br>Low/Medium/High"]
    end

    Sources --> GD --> Output

    style Sources fill:#e3f2fd,color:#000
    style GD fill:#fff9c4,color:#000
    style Output fill:#c8e6c9,color:#000
    style CT fill:#fff,color:#000
    style VPC fill:#fff,color:#000
    style DNS fill:#fff,color:#000
    style S3 fill:#fff,color:#000
    style EKS fill:#fff,color:#000
    style ML fill:#fff,color:#000
    style TI fill:#fff,color:#000
    style RULES fill:#fff,color:#000
    style FINDINGS fill:#fff,color:#000
    style SEV fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### GuardDuty Features

GuardDuty uses multiple detection methods to identify threats. It combines threat intelligence feeds
with ML (Machine Learning) anomaly detection to catch both known and novel attacks:

| Feature                  | Description                                           |
| ------------------------ | ----------------------------------------------------- |
| **Threat Intelligence**  | Known malicious IPs, domains                          |
| **ML Anomaly Detection** | Unusual API calls, data access                        |
| **VPC Flow Analysis**    | Port scanning, C2 (Command and Control) communication |
| **DNS Analysis**         | DNS exfiltration, crypto mining                       |

> **Exam Tip**: GuardDuty analyzes CloudTrail, VPC Flow Logs, and DNS automatically. You don't need
> to enable these separately for GuardDuty.

### GuardDuty Finding Types

```mermaid
flowchart TB
    subgraph Findings["GuardDuty Finding Categories"]
        RECON["Recon<br>Port scanning,<br>API probing"]
        INSTANCE["Instance<br>Compromise,<br>crypto mining"]
        ACCOUNT["Account<br>Credential theft,<br>unusual activity"]
        S3["S3<br>Data exfiltration,<br>public access"]
        IAM["IAM<br>Unusual API calls"]
    end

    style Findings fill:#e3f2fd,color:#000
    style RECON fill:#fff9c4,color:#000
    style INSTANCE fill:#fff9c4,color:#000
    style ACCOUNT fill:#ffcdd2,color:#000
    style S3 fill:#ffcdd2,color:#000
    style IAM fill:#fff9c4,color:#000
```

**Example findings**:

- `Recon:EC2/PortProbeUnprotectedPort` - Someone scanning your ports
- `UnauthorizedAccess:IAMUser/ConsoleLoginSuccess.B` - Successful login from unusual location
- `Exfiltration:S3/ObjectRead.Unusual` - Unusual S3 data access pattern

### GuardDuty Organization Integration

```mermaid
flowchart TB
    subgraph Org["AWS Organization"]
        ADMIN["GuardDuty<br>Administrator"]

        subgraph Members["Member Accounts"]
            M1["Account 1"]
            M2["Account 2"]
            M3["Account 3"]
        end
    end

    ADMIN -->|"Centralized<br>findings"| Members

    style Org fill:#e3f2fd,color:#000
    style ADMIN fill:#ffcdd2,color:#000
    style Members fill:#c8e6c9,color:#000
    style M1 fill:#fff,color:#000
    style M2 fill:#fff,color:#000
    style M3 fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

> **Exam Tip**: Designate a delegated administrator account to manage GuardDuty across all
> organization accounts.

### AWS Security Hub

```mermaid
flowchart TB
    subgraph Sources["Security Data Sources"]
        GD["GuardDuty"]
        INSP["Inspector"]
        MACIE["Macie"]
        FM["Firewall Manager"]
        IAM_AA["IAM Access<br>Analyzer"]
        CONFIG["Config"]
    end

    subgraph Hub["Security Hub"]
        AGG["Aggregation"]
        STD["Security Standards"]
        SCORE["Security Score"]
    end

    subgraph Output["Output"]
        DASH["Dashboard"]
        FIND["Findings"]
        ACTION["Automated<br>Response"]
    end

    Sources --> Hub --> Output

    style Sources fill:#e3f2fd,color:#000
    style Hub fill:#fff9c4,color:#000
    style Output fill:#c8e6c9,color:#000
    style GD fill:#fff,color:#000
    style INSP fill:#fff,color:#000
    style MACIE fill:#fff,color:#000
    style FM fill:#fff,color:#000
    style IAM_AA fill:#fff,color:#000
    style CONFIG fill:#fff,color:#000
    style AGG fill:#fff,color:#000
    style STD fill:#fff,color:#000
    style SCORE fill:#fff,color:#000
    style DASH fill:#fff,color:#000
    style FIND fill:#fff,color:#000
    style ACTION fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### Security Hub Standards

Security Hub runs automated compliance checks against industry standards. Each standard includes
dozens of controls that are continuously evaluated. Choose standards based on your compliance
requirements:

| Standard                                     | Focus                 |
| -------------------------------------------- | --------------------- |
| **AWS Foundational Security Best Practices** | AWS-specific security |
| **CIS AWS Foundations Benchmark**            | Industry standard     |
| **PCI DSS**                                  | Payment card security |
| **NIST 800-53**                              | Government security   |

> **Abbreviations**: CIS = Center for Internet Security, PCI DSS = Payment Card Industry Data
> Security Standard, NIST = National Institute of Standards and Technology

> **Exam Tip**: Security Hub provides a unified view of security findings AND runs compliance checks
> against standards.

### Amazon Macie

```mermaid
flowchart LR
    subgraph S3["S3 Buckets"]
        B1["Patient<br>Records"]
        B2["Log<br>Files"]
        B3["Application<br>Data"]
    end

    MACIE["Amazon Macie"]

    subgraph Output["Output"]
        CLASS["Data<br>Classification"]
        PII["PII (Personally<br>Identifiable<br>Information)<br>Detection"]
        ALERT["Sensitive Data<br>Alerts"]
    end

    S3 --> MACIE --> Output

    style S3 fill:#e3f2fd,color:#000
    style MACIE fill:#fff9c4,color:#000
    style Output fill:#c8e6c9,color:#000
    style B1 fill:#fff,color:#000
    style B2 fill:#fff,color:#000
    style B3 fill:#fff,color:#000
    style CLASS fill:#fff,color:#000
    style PII fill:#fff,color:#000
    style ALERT fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### Macie Capabilities

Macie automates sensitive data discovery in S3. For healthcare organizations like MedVault, it can
identify PHI (Protected Health Information) that may have been stored in the wrong location:

| Capability                   | Description                        |
| ---------------------------- | ---------------------------------- |
| **Sensitive Data Discovery** | Scans S3 for PII, PHI, credentials |
| **Data Classification**      | Categorizes data types             |
| **Bucket Inventory**         | S3 security posture                |
| **Policy Findings**          | Public buckets, unencrypted data   |

**Detected data types**:

- Personal: Names, addresses, SSN, passport
- Financial: Credit cards, bank accounts
- Healthcare: PHI, medical records
- Credentials: API keys, passwords

> **Exam Tip**: Macie is specifically for **S3 data classification**. It uses ML to find sensitive
> data.

### Amazon Inspector

```mermaid
flowchart TB
    subgraph Targets["Assessment Targets"]
        EC2["EC2<br>Instances"]
        ECR["ECR (Elastic<br>Container Registry)<br>Images"]
        LAMBDA["Lambda<br>Functions"]
    end

    INSP["Amazon Inspector"]

    subgraph Checks["Vulnerability Checks"]
        CVE["CVE (Common<br>Vulnerabilities and<br>Exposures) Database"]
        NET["Network<br>Reachability"]
        PKG["Package<br>Vulnerabilities"]
    end

    Targets --> INSP --> Checks

    style Targets fill:#e3f2fd,color:#000
    style INSP fill:#fff9c4,color:#000
    style Checks fill:#c8e6c9,color:#000
    style EC2 fill:#fff,color:#000
    style ECR fill:#fff,color:#000
    style LAMBDA fill:#fff,color:#000
    style CVE fill:#fff,color:#000
    style NET fill:#fff,color:#000
    style PKG fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### Security Service Comparison

These four services work together but serve distinct purposes. This comparison helps you choose the
right service for each security question on the exam:

| Service          | What It Does             | Analyzes                  |
| ---------------- | ------------------------ | ------------------------- |
| **GuardDuty**    | Threat detection         | CloudTrail, VPC Flow, DNS |
| **Inspector**    | Vulnerability scanning   | EC2, ECR, Lambda          |
| **Macie**        | Data classification      | S3 contents               |
| **Security Hub** | Aggregation + compliance | All of the above          |

> **Exam Tip**: GuardDuty = threats (who's attacking). Inspector = vulnerabilities (what's
> exploitable). Macie = sensitive data (where's the PII).

### Amazon Detective

For security investigations:

```mermaid
flowchart LR
    FINDING["GuardDuty<br>Finding"]
    DET["Amazon<br>Detective"]
    GRAPH["Behavior<br>Graph"]
    ROOT["Root Cause<br>Analysis"]

    FINDING --> DET --> GRAPH --> ROOT

    style FINDING fill:#ffcdd2,color:#000
    style DET fill:#fff9c4,color:#000
    style GRAPH fill:#e3f2fd,color:#000
    style ROOT fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

> **Exam Tip**: Detective helps **investigate** findings from GuardDuty. It's for post-incident
> analysis.

### Automated Response

```mermaid
flowchart LR
    subgraph Detection["Detection"]
        GD["GuardDuty"]
        CONFIG["Config"]
    end

    EB["EventBridge"]

    subgraph Response["Response"]
        LAMBDA["Lambda"]
        SSM["SSM<br>Automation"]
        SNS["SNS<br>Alert"]
    end

    subgraph Actions["Actions"]
        ISO["Isolate<br>Instance"]
        REVOKE["Revoke<br>Credentials"]
        BLOCK["Block<br>IP"]
    end

    Detection --> EB --> Response --> Actions

    style Detection fill:#ffcdd2,color:#000
    style EB fill:#fff9c4,color:#000
    style Response fill:#e3f2fd,color:#000
    style Actions fill:#c8e6c9,color:#000
    style GD fill:#fff,color:#000
    style CONFIG fill:#fff,color:#000
    style LAMBDA fill:#fff,color:#000
    style SSM fill:#fff,color:#000
    style SNS fill:#fff,color:#000
    style ISO fill:#fff,color:#000
    style REVOKE fill:#fff,color:#000
    style BLOCK fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

**Example: Auto-isolate compromised instance**:

```json
{
  "source": ["aws.guardduty"],
  "detail-type": ["GuardDuty Finding"],
  "detail": {
    "severity": [{ "numeric": [">=", 7] }],
    "type": [{ "prefix": "UnauthorizedAccess:EC2" }]
  }
}
```

## MedVault Threat Detection Architecture

```mermaid
flowchart TB
    subgraph Detection["Detection Layer"]
        GD["GuardDuty<br>Threat Detection"]
        MACIE["Macie<br>PHI Discovery"]
        INSP["Inspector<br>Vulnerabilities"]
    end

    subgraph Aggregation["Security Hub"]
        HUB["Centralized<br>Findings"]
        STD["HIPAA<br>Compliance"]
    end

    subgraph Response["Response Layer"]
        EB["EventBridge"]
        LAMBDA["Lambda<br>Automation"]
        SNS["SNS<br>Alerts"]
    end

    subgraph Actions["Automated Actions"]
        A1["Isolate instance"]
        A2["Revoke credentials"]
        A3["Block IP in WAF"]
        A4["Page security team"]
    end

    Detection --> Aggregation --> Response --> Actions

    style Detection fill:#ffcdd2,color:#000
    style Aggregation fill:#fff9c4,color:#000
    style Response fill:#e3f2fd,color:#000
    style Actions fill:#c8e6c9,color:#000
    style GD fill:#fff,color:#000
    style MACIE fill:#fff,color:#000
    style INSP fill:#fff,color:#000
    style HUB fill:#fff,color:#000
    style STD fill:#fff,color:#000
    style EB fill:#fff,color:#000
    style LAMBDA fill:#fff,color:#000
    style SNS fill:#fff,color:#000
    style A1 fill:#fff,color:#000
    style A2 fill:#fff,color:#000
    style A3 fill:#fff,color:#000
    style A4 fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### MedVault Security Automation

Each security finding type maps to a specific automated response. High-severity threats trigger
immediate containment actions, while lower-severity findings create tickets for human review:

| Trigger                                | Automated Response                |
| -------------------------------------- | --------------------------------- |
| GuardDuty: EC2 compromise (high)       | Isolate instance (security group) |
| GuardDuty: IAM credential exfiltration | Disable access key                |
| GuardDuty: S3 exfiltration             | Block IP in WAF                   |
| Macie: PHI in wrong bucket             | Move to correct bucket, alert     |
| Inspector: Critical CVE                | Create patch ticket               |
| Config: Public S3 bucket               | Auto-remediate (block public)     |

### Compliance Reporting

With Security Hub, MedVault can generate compliance reports:

```mermaid
flowchart TB
    subgraph Standards["Security Standards"]
        HIPAA["HIPAA"]
        CIS["CIS Benchmark"]
        FSBP["AWS Best<br>Practices"]
    end

    subgraph HUB["Security Hub"]
        CHECKS["Automated<br>Checks"]
        SCORE["Compliance<br>Score"]
    end

    subgraph Report["Output"]
        DASH["Dashboard"]
        PDF["Audit<br>Reports"]
    end

    Standards --> CHECKS --> SCORE --> Report

    style Standards fill:#e3f2fd,color:#000
    style HUB fill:#fff9c4,color:#000
    style Report fill:#c8e6c9,color:#000
    style HIPAA fill:#fff,color:#000
    style CIS fill:#fff,color:#000
    style FSBP fill:#fff,color:#000
    style CHECKS fill:#fff,color:#000
    style SCORE fill:#fff,color:#000
    style DASH fill:#fff,color:#000
    style PDF fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

## Congratulations!

You've completed the MedVault security journey. You now understand the AWS security services that
comprise **30% of the SAA exam**:

- **Identity & Access**: IAM, Identity Center, Organizations
- **Data Protection**: KMS, Secrets Manager, encryption options
- **Network Security**: VPC endpoints, WAF, Shield
- **Logging & Monitoring**: CloudTrail, Config, CloudWatch
- **Multi-Account**: Organizations, SCPs, Control Tower
- **Threat Detection**: GuardDuty, Security Hub, Macie, Inspector

### Key Exam Takeaways

1. **IAM roles over users** - Temporary credentials preferred
2. **Encryption with KMS** - CMK for compliance, SSE-KMS for audit
3. **VPC endpoints** - Gateway (S3/DynamoDB free), Interface (everything else)
4. **CloudTrail vs Config** - Activity vs state
5. **SCPs limit, don't grant** - Management account unaffected
6. **GuardDuty for threats** - Inspector for vulnerabilities, Macie for data

## Exam Tips

- **GuardDuty = threat detection** - Uses CloudTrail, VPC Flow, DNS
- **Inspector = vulnerability scanning** - EC2, ECR, Lambda
- **Macie = S3 data classification** - Finds PII, PHI
- **Security Hub = aggregation** - Unified view + compliance standards
- **Detective = investigation** - Post-incident analysis
- **All can integrate with EventBridge** - For automated response

## SAA Exam Concepts

### Must-Know for This Phase

| Concept            | Key Points                                                |
| ------------------ | --------------------------------------------------------- |
| GuardDuty          | Threat detection, CloudTrail/VPC Flow/DNS, ML-based       |
| Inspector          | Vulnerability scanning, EC2/ECR/Lambda, CVE database      |
| Macie              | S3 data classification, PII/PHI detection                 |
| Security Hub       | Aggregates findings, compliance standards, security score |
| Detective          | Investigation, behavior graphs, root cause                |
| Automated Response | EventBridge + Lambda for remediation                      |
