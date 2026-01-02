# Phase 2: Data Protection

## The Story So Far

MedVault has established a solid IAM foundation. Employees authenticate via Okta through IAM
Identity Center, applications use IAM roles with temporary credentials, and permission boundaries
prevent privilege escalation.

## Business Trigger

The compliance consultant reviews your IAM setup and moves to the next item:

> "HIPAA requires encryption of Protected Health Information (PHI) at rest and in transit. Show me
> your encryption strategy."

The CTO adds technical requirements:

> "We also need to manage database credentials securely. No hardcoded passwords in code or config
> files. And those credentials need to rotate automatically."

## Architecture Decision

**Decision**: Implement comprehensive encryption using AWS KMS, store secrets in AWS Secrets Manager
with automatic rotation, and enforce TLS for all data in transit.

### Data Protection Layers

| Layer             | Protection     | AWS Service                 |
| ----------------- | -------------- | --------------------------- |
| At rest (storage) | Encryption     | KMS + S3/EBS/RDS encryption |
| In transit        | TLS/SSL        | ACM, ALB, CloudFront        |
| Secrets           | Secure storage | Secrets Manager             |
| Keys              | Management     | KMS                         |

## Key Concepts for SAA Exam

### AWS KMS (Key Management Service)

```mermaid
flowchart TB
    subgraph KMS["AWS KMS"]
        subgraph KeyTypes["Key Types"]
            AWS["AWS Managed Keys<br>aws/s3, aws/ebs"]
            CMK["Customer Managed<br>Keys (CMK)"]
            EXT["External Keys<br>(imported)"]
        end

        subgraph Features["Features"]
            ROT["Automatic Rotation"]
            AUDIT["CloudTrail Logging"]
            POLICY["Key Policies"]
        end
    end

    style KMS fill:#e3f2fd,color:#000
    style KeyTypes fill:#fff9c4,color:#000
    style Features fill:#c8e6c9,color:#000
    style AWS fill:#fff,color:#000
    style CMK fill:#fff,color:#000
    style EXT fill:#fff,color:#000
    style ROT fill:#fff,color:#000
    style AUDIT fill:#fff,color:#000
    style POLICY fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### KMS Key Types Comparison

| Key Type                | Management | Rotation           | Cost                 | Use Case            |
| ----------------------- | ---------- | ------------------ | -------------------- | ------------------- |
| **AWS Managed**         | AWS        | Automatic (yearly) | Free\*               | Default encryption  |
| **Customer Managed**    | You        | Optional (yearly)  | $1/month + API calls | Compliance, control |
| **External (imported)** | You        | Manual             | $1/month + API calls | Key sovereignty     |

> **Exam Tip**: Customer managed keys give you control over the key policy, deletion, and rotation.
> Use them when compliance requires it.

### Envelope Encryption

KMS uses envelope encryption for efficiency:

```mermaid
flowchart LR
    subgraph Process["Envelope Encryption"]
        CMK["Customer<br>Master Key<br>(in KMS)"]
        DEK["Data Encryption<br>Key (DEK)"]
        DATA["Your Data"]
    end

    CMK -->|"1. Generate"| DEK
    DEK -->|"2. Encrypt"| DATA
    CMK -->|"3. Encrypt DEK"| EDEK["Encrypted<br>DEK"]
    EDEK -->|"4. Store with"| EDATA["Encrypted<br>Data"]

    style Process fill:#e3f2fd,color:#000
    style CMK fill:#ffcdd2,color:#000
    style DEK fill:#fff9c4,color:#000
    style DATA fill:#c8e6c9,color:#000
    style EDEK fill:#fff9c4,color:#000
    style EDATA fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

**Why envelope encryption?**

- CMK never leaves KMS (FIPS 140-2 validated)
- Only small DEK is encrypted by CMK (fast)
- Large data encrypted locally by DEK (efficient)

> **Exam Tip**: The CMK encrypts the data key, not the data itself. This is more efficient and
> secure.

### KMS Key Policies

Key policies are resource-based policies for KMS keys:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Enable IAM policies",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::111122223333:root" },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "Allow use by app role",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::111122223333:role/AppRole" },
      "Action": ["kms:Encrypt", "kms:Decrypt", "kms:GenerateDataKey"],
      "Resource": "*"
    }
  ]
}
```

> **Exam Tip**: Unlike other resources, KMS keys **require** a key policy. Without one, no one can
> use the key.

### S3 Encryption Options

```mermaid
flowchart TB
    subgraph ServerSide["Server-Side Encryption"]
        SSE_S3["SSE-S3<br>S3 managed keys"]
        SSE_KMS["SSE-KMS<br>KMS managed keys"]
        SSE_C["SSE-C<br>Customer provided keys"]
    end

    subgraph ClientSide["Client-Side Encryption"]
        CSE["You encrypt before<br>uploading"]
    end

    style ServerSide fill:#c8e6c9,color:#000
    style ClientSide fill:#fff9c4,color:#000
    style SSE_S3 fill:#fff,color:#000
    style SSE_KMS fill:#fff,color:#000
    style SSE_C fill:#fff,color:#000
    style CSE fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

| Option          | Key Management           | Audit Trail         | Use Case          |
| --------------- | ------------------------ | ------------------- | ----------------- |
| **SSE-S3**      | S3 manages               | Limited             | Default, simple   |
| **SSE-KMS**     | You/KMS manage           | Full CloudTrail     | Compliance, audit |
| **SSE-C**       | You provide each request | Your responsibility | Key sovereignty   |
| **Client-side** | You manage entirely      | Your responsibility | Zero-trust        |

> **Exam Tip**: SSE-KMS provides audit trail via CloudTrail. SSE-S3 does not log individual object
> access.

### Default Encryption

Enable default encryption on S3 buckets:

```json
{
  "Rules": [
    {
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "arn:aws:kms:us-east-1:111122223333:key/..."
      },
      "BucketKeyEnabled": true
    }
  ]
}
```

> **Exam Tip**: **Bucket Keys** reduce KMS API calls (and costs) by using a bucket-level key derived
> from your KMS key.

### EBS and RDS Encryption

```mermaid
flowchart TB
    subgraph EBS["EBS Encryption"]
        E1["Encrypted at rest"]
        E2["Encrypted in transit<br>(EBS to EC2)"]
        E3["Snapshots encrypted"]
        E4["Cannot disable once enabled"]
    end

    subgraph RDS["RDS Encryption"]
        R1["Encrypted at rest"]
        R2["TLS in transit"]
        R3["Snapshots encrypted"]
        R4["Read replicas encrypted"]
    end

    style EBS fill:#e3f2fd,color:#000
    style RDS fill:#c8e6c9,color:#000
    style E1 fill:#fff,color:#000
    style E2 fill:#fff,color:#000
    style E3 fill:#fff,color:#000
    style E4 fill:#fff,color:#000
    style R1 fill:#fff,color:#000
    style R2 fill:#fff,color:#000
    style R3 fill:#fff,color:#000
    style R4 fill:#fff,color:#000
```

**Important limitations**:

- Cannot encrypt an existing unencrypted EBS volume (must snapshot and create new)
- Cannot encrypt an existing unencrypted RDS instance (must snapshot and restore)
- Encrypted snapshots can only be copied to encrypted destinations

> **Exam Tip**: To encrypt an unencrypted RDS database: snapshot → copy snapshot with encryption →
> restore from encrypted snapshot.

### AWS Secrets Manager

```mermaid
flowchart LR
    subgraph App["Application"]
        CODE["App Code"]
    end

    subgraph SM["Secrets Manager"]
        SECRET["Secret:<br>db-credentials"]
        ROT["Rotation<br>Lambda"]
    end

    subgraph DB["Database"]
        RDS["RDS MySQL"]
    end

    CODE -->|"1. GetSecretValue"| SECRET
    SECRET -->|"2. Return credentials"| CODE
    CODE -->|"3. Connect"| RDS
    ROT -->|"4. Rotate (scheduled)"| SECRET
    ROT -->|"5. Update password"| RDS

    style App fill:#e3f2fd,color:#000
    style SM fill:#fff9c4,color:#000
    style DB fill:#c8e6c9,color:#000
    style CODE fill:#fff,color:#000
    style SECRET fill:#fff,color:#000
    style ROT fill:#fff,color:#000
    style RDS fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### Secrets Manager vs Parameter Store

| Feature          | Secrets Manager                | Parameter Store                     |
| ---------------- | ------------------------------ | ----------------------------------- |
| **Rotation**     | Built-in, automatic            | Manual (you build it)               |
| **Cost**         | $0.40/secret/month             | Free tier, then $0.05/param         |
| **Cross-region** | Native replication             | Manual                              |
| **Best for**     | Database credentials, API keys | Config values, non-rotating secrets |

> **Exam Tip**: Secrets Manager = automatic rotation. Parameter Store = cheaper, no built-in
> rotation.

### Secrets Manager Rotation

```mermaid
sequenceDiagram
    participant SM as Secrets Manager
    participant Lambda as Rotation Lambda
    participant DB as Database

    Note over SM,DB: Automatic Rotation (e.g., every 30 days)
    SM->>Lambda: Trigger rotation
    Lambda->>SM: Create new secret version (AWSPENDING)
    Lambda->>DB: Set new password
    Lambda->>DB: Test new password
    Lambda->>SM: Promote AWSPENDING to AWSCURRENT
    Note over SM: Old version becomes AWSPREVIOUS
```

### ACM (AWS Certificate Manager)

```mermaid
flowchart TB
    subgraph ACM["ACM"]
        CERT["SSL/TLS<br>Certificate"]
        RENEW["Auto-renewal"]
    end

    subgraph Services["Integrated Services"]
        ALB["Application<br>Load Balancer"]
        CF["CloudFront"]
        APIGW["API Gateway"]
    end

    ACM --> ALB & CF & APIGW

    style ACM fill:#e3f2fd,color:#000
    style Services fill:#c8e6c9,color:#000
    style CERT fill:#fff,color:#000
    style RENEW fill:#fff,color:#000
    style ALB fill:#fff,color:#000
    style CF fill:#fff,color:#000
    style APIGW fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

**Key points**:

- **Free** public certificates
- Auto-renewal for ACM-issued certs
- Cannot be downloaded (use on AWS services only)
- For EC2, use imported certificates

> **Exam Tip**: ACM certificates are **regional** except for CloudFront (must be in us-east-1).

## MedVault Data Protection Architecture

```mermaid
flowchart TB
    subgraph Users["Users"]
        CLIENT["Web Client"]
    end

    subgraph Edge["Edge"]
        CF["CloudFront<br>TLS 1.2+"]
    end

    subgraph VPC["VPC"]
        ALB["ALB<br>ACM Certificate"]

        subgraph App["Application"]
            EC2["EC2<br>EBS Encrypted"]
            SM["Secrets<br>Manager"]
        end

        subgraph Data["Data Tier"]
            RDS[("RDS<br>SSE-KMS")]
            S3["S3<br>SSE-KMS<br>Bucket Key"]
        end
    end

    subgraph KMS["KMS"]
        KEY["medvault-key<br>(CMK)"]
    end

    CLIENT -->|"HTTPS"| CF
    CF -->|"HTTPS"| ALB
    ALB -->|"HTTPS"| EC2
    EC2 -->|"Get credentials"| SM
    EC2 -->|"Query"| RDS
    EC2 -->|"Upload PHI"| S3
    RDS & S3 & SM -.->|"Encryption"| KEY

    style Users fill:#fff,color:#000
    style Edge fill:#bbdefb,color:#000
    style VPC fill:#e3f2fd,color:#000
    style App fill:#fff9c4,color:#000
    style Data fill:#c8e6c9,color:#000
    style KMS fill:#ffcdd2,color:#000
    style CLIENT fill:#fff,color:#000
    style CF fill:#fff,color:#000
    style ALB fill:#fff,color:#000
    style EC2 fill:#fff,color:#000
    style SM fill:#fff,color:#000
    style RDS fill:#fff,color:#000
    style S3 fill:#fff,color:#000
    style KEY fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### MedVault Encryption Decisions

| Data                  | Encryption      | Key Type         | Rationale                |
| --------------------- | --------------- | ---------------- | ------------------------ |
| S3 (PHI documents)    | SSE-KMS         | Customer managed | Audit trail, key control |
| RDS (patient records) | RDS encryption  | Customer managed | Compliance               |
| EBS (app servers)     | EBS encryption  | AWS managed      | Less sensitive           |
| Secrets               | Secrets Manager | AWS managed      | Rotation more important  |
| In transit            | TLS 1.2+        | ACM              | Standard                 |

## What Could Go Wrong?

Data is now encrypted at rest and in transit. Secrets are managed securely with automatic rotation.
But the security auditor has another question:

> "I see you're using S3 and other AWS services. Are those API calls going over the public internet?
> For HIPAA, we'd prefer all traffic stays within AWS."

Time to implement network security controls.

## Exam Tips

- **CMK for compliance** - Customer managed keys when you need audit/control
- **Envelope encryption** - CMK encrypts data key, data key encrypts data
- **SSE-KMS for S3 audit** - CloudTrail logs KMS usage
- **Bucket Keys reduce cost** - Fewer KMS API calls
- **Cannot decrypt unencrypted** - Must copy with encryption
- **Secrets Manager for rotation** - Built-in, automatic
- **Parameter Store is cheaper** - But no automatic rotation
- **ACM is regional** - Except CloudFront (us-east-1 only)

## SAA Exam Concepts

### Must-Know for This Phase

| Concept             | Key Points                                                         |
| ------------------- | ------------------------------------------------------------------ |
| KMS Key Types       | AWS managed (free), Customer managed ($1/mo), External             |
| Envelope Encryption | CMK → Data Key → Data                                              |
| S3 Encryption       | SSE-S3, SSE-KMS (audit trail), SSE-C, Client-side                  |
| Bucket Keys         | Reduce KMS API calls and cost                                      |
| EBS/RDS Encryption  | Cannot enable on existing, must snapshot/restore                   |
| Secrets Manager     | Auto-rotation, $0.40/secret/month, cross-region                    |
| Parameter Store     | Cheaper, no auto-rotation, good for config                         |
| ACM                 | Free public certs, auto-renewal, regional (CloudFront = us-east-1) |
