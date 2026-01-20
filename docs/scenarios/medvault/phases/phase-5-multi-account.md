# Phase 5: Multi-Account Strategy

## The Story So Far

MedVault has comprehensive security controls: IAM, encryption, network security, and logging. The
platform is nearing launch, and the team is growing from 5 to 50 engineers.

## Business Trigger

The VP of Engineering raises concerns:

> "We have developers, QA engineers, and DevOps all working in the same AWS account. Last week, a
> junior developer accidentally deleted a DynamoDB table. In production. How do we prevent this as
> we scale?"

The CISO adds:

> "I also need environment isolation. Dev should never touch prod data. And if a developer's
> credentials are compromised, the blast radius should be limited to non-production."

## Architecture Decision

**Decision**: Implement a multi-account strategy using AWS Organizations with SCPs (Service Control
Policies) for guardrails, managed through AWS Control Tower.

### Why Multi-Account?

Multi-account architecture is a best practice for enterprise workloads. The key benefit is blast
radius reduction - a mistake in one account can't affect others. This table compares the trade-offs:

| Single Account            | Multi-Account              |
| ------------------------- | -------------------------- |
| Blast radius = everything | Blast radius = one account |
| Hard to separate billing  | Clear cost attribution     |
| Permission complexity     | Simple, scoped permissions |
| Compliance challenges     | Environment isolation      |

## Key Concepts for SAA Exam

### AWS Organizations

```mermaid
flowchart TB
    subgraph Org["AWS Organization"]
        ROOT["Root"]

        subgraph MGMT["Management Account"]
            ORG["Organizations"]
            BILLING["Consolidated Billing"]
        end

        subgraph OUs["OUs (Organizational Units)"]
            SECURITY["Security OU"]
            WORKLOADS["Workloads OU"]
        end

        subgraph Accounts["Member Accounts"]
            SEC_ACC["Security Account"]
            PROD["Production"]
            DEV["Development"]
        end
    end

    ROOT --> MGMT
    ROOT --> OUs
    SECURITY --> SEC_ACC
    WORKLOADS --> PROD & DEV

    style Org fill:#e3f2fd,color:#000
    style MGMT fill:#ffcdd2,color:#000
    style OUs fill:#fff9c4,color:#000
    style Accounts fill:#c8e6c9,color:#000
    style ROOT fill:#fff,color:#000
    style ORG fill:#fff,color:#000
    style BILLING fill:#fff,color:#000
    style SECURITY fill:#fff,color:#000
    style WORKLOADS fill:#fff,color:#000
    style SEC_ACC fill:#fff,color:#000
    style PROD fill:#fff,color:#000
    style DEV fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### Organization Components

Understanding these components is essential for designing multi-account architectures. The hierarchy
flows from Root → OUs → Accounts, with SCPs applied at any level:

| Component                        | Description                                      |
| -------------------------------- | ------------------------------------------------ |
| **Root**                         | Top of the OU hierarchy                          |
| **Organizational Unit (OU)**     | Container for accounts, can be nested            |
| **Management Account**           | Creates org, manages billing (formerly "master") |
| **Member Account**               | Regular accounts in the organization             |
| **SCP (Service Control Policy)** | Permission guardrails for OUs/accounts           |

> **Exam Tip**: Management account should be used only for organization management, not workloads.

### Service Control Policies (SCPs)

```mermaid
flowchart TB
    subgraph Evaluation["Permission Evaluation"]
        SCP["SCP:<br>What's ALLOWED<br>in this account"]
        IAM["IAM Policy:<br>What this identity<br>CAN DO"]
        RESULT["Effective:<br>Intersection"]
    end

    SCP --> RESULT
    IAM --> RESULT

    style Evaluation fill:#e3f2fd,color:#000
    style SCP fill:#ffcdd2,color:#000
    style IAM fill:#c8e6c9,color:#000
    style RESULT fill:#fff9c4,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### SCP Key Points

- SCPs are **allow lists** or **deny lists** (not grants)
- Don't grant permissions - only **limit** what IAM can grant
- Apply to entire account (all principals except management account)
- Management account is **never affected** by SCPs
- Inherited down the OU hierarchy

> **Exam Tip**: SCPs don't affect the management account. This is why workloads shouldn't run there.

### SCP Inheritance

```mermaid
flowchart TB
    ROOT["Root<br>FullAWSAccess"]
    OU1["Security OU<br>DenyEC2"]
    OU2["Workloads OU"]
    ACC1["Security Account<br>Inherited: DenyEC2"]
    ACC2["Prod Account"]
    ACC3["Dev Account<br>DenyDeleteProd"]

    ROOT --> OU1 & OU2
    OU1 --> ACC1
    OU2 --> ACC2 & ACC3

    style ROOT fill:#e3f2fd,color:#000
    style OU1 fill:#fff9c4,color:#000
    style OU2 fill:#fff9c4,color:#000
    style ACC1 fill:#c8e6c9,color:#000
    style ACC2 fill:#c8e6c9,color:#000
    style ACC3 fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

**Effective permissions** = All SCPs in path must allow + IAM must allow

### Common SCP Patterns

**Deny leaving organization**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "organizations:LeaveOrganization",
      "Resource": "*"
    }
  ]
}
```

**Deny disabling CloudTrail**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": ["cloudtrail:StopLogging", "cloudtrail:DeleteTrail"],
      "Resource": "*"
    }
  ]
}
```

**Restrict regions**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "NotAction": ["iam:*", "organizations:*", "support:*"],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": ["us-east-1", "us-west-2"]
        }
      }
    }
  ]
}
```

> **Exam Tip**: Region restriction SCPs are common exam questions. Note the use of `NotAction` to
> allow global services.

### AWS Control Tower

```mermaid
flowchart TB
    subgraph CT["Control Tower"]
        LZ["Landing Zone"]
        GR["Guardrails"]
        AF["Account Factory"]
        DASH["Dashboard"]
    end

    subgraph Created["Auto-Created"]
        MGMT["Management Account"]
        LOG["Log Archive Account"]
        AUDIT["Audit Account"]
    end

    CT --> Created

    style CT fill:#e3f2fd,color:#000
    style Created fill:#c8e6c9,color:#000
    style LZ fill:#fff,color:#000
    style GR fill:#fff,color:#000
    style AF fill:#fff,color:#000
    style DASH fill:#fff,color:#000
    style MGMT fill:#fff,color:#000
    style LOG fill:#fff,color:#000
    style AUDIT fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### Control Tower Components

Control Tower automates the setup of a secure multi-account environment. Instead of manually
configuring Organizations, SCPs, and accounts, Control Tower provides these building blocks:

| Component           | Description                              |
| ------------------- | ---------------------------------------- |
| **Landing Zone**    | Pre-configured multi-account environment |
| **Guardrails**      | Pre-built SCPs and Config rules          |
| **Account Factory** | Automated account provisioning           |
| **Dashboard**       | Compliance visibility                    |

### Guardrail Types

Guardrails enforce governance at different stages. Preventive guardrails stop actions before they
happen, detective guardrails identify drift after the fact, and proactive guardrails validate
infrastructure-as-code before deployment:

| Type           | Implementation             | Enforcement                      |
| -------------- | -------------------------- | -------------------------------- |
| **Preventive** | SCP                        | Blocks non-compliant actions     |
| **Detective**  | Config rules               | Alerts on non-compliance         |
| **Proactive**  | CFN (CloudFormation) hooks | Blocks non-compliant deployments |

> **Exam Tip**: Preventive = SCP (blocks). Detective = Config (alerts). Know the difference.

### Control Tower Accounts

Control Tower automatically creates these foundational accounts. Each serves a specific purpose in
the security architecture. The management account should never run workloads:

| Account         | Purpose                                 |
| --------------- | --------------------------------------- |
| **Management**  | Organization management (no workloads!) |
| **Log Archive** | Centralized CloudTrail and Config logs  |
| **Audit**       | Security tools, cross-account access    |

### AWS RAM (Resource Access Manager)

Share resources across accounts without copying them:

```mermaid
flowchart LR
    subgraph Owner["Owner Account"]
        VPC["VPC Subnet"]
        TGW["Transit Gateway"]
    end

    RAM["RAM"]

    subgraph Consumer["Consumer Accounts"]
        ACC1["Account 1"]
        ACC2["Account 2"]
    end

    VPC & TGW --> RAM --> ACC1 & ACC2

    style Owner fill:#e3f2fd,color:#000
    style RAM fill:#fff9c4,color:#000
    style Consumer fill:#c8e6c9,color:#000
    style VPC fill:#fff,color:#000
    style TGW fill:#fff,color:#000
    style ACC1 fill:#fff,color:#000
    style ACC2 fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

**Shareable resources**:

- VPC subnets
- Transit Gateway
- Route 53 Resolver rules
- License Manager configurations
- Aurora DB clusters

> **Exam Tip**: RAM enables sharing without copying. The resource stays in the owner account.

### Centralized Logging Architecture

```mermaid
flowchart TB
    subgraph Accounts["All Accounts"]
        PROD["Production"]
        DEV["Development"]
        SEC["Security"]
    end

    subgraph LogArch["Log Archive Account"]
        S3["Central S3<br>Bucket"]
    end

    ORG_TRAIL["Organization<br>Trail"]

    PROD & DEV & SEC --> ORG_TRAIL --> S3

    style Accounts fill:#e3f2fd,color:#000
    style LogArch fill:#c8e6c9,color:#000
    style PROD fill:#fff,color:#000
    style DEV fill:#fff,color:#000
    style SEC fill:#fff,color:#000
    style S3 fill:#fff,color:#000
    style ORG_TRAIL fill:#fff9c4,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### Cross-Account Access Patterns

```mermaid
flowchart LR
    subgraph DevAcc["Dev Account"]
        DEV_ROLE["Developer<br>Role"]
    end

    subgraph ProdAcc["Prod Account"]
        PROD_ROLE["ReadOnly<br>Role"]
        PROD_DB[("Production<br>Database")]
    end

    DEV_ROLE -->|"AssumeRole<br>(read-only)"| PROD_ROLE
    PROD_ROLE -->|"Read"| PROD_DB

    style DevAcc fill:#e3f2fd,color:#000
    style ProdAcc fill:#c8e6c9,color:#000
    style DEV_ROLE fill:#fff,color:#000
    style PROD_ROLE fill:#fff,color:#000
    style PROD_DB fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

## MedVault Multi-Account Architecture

```mermaid
flowchart TB
    subgraph Org["MedVault Organization"]
        ROOT["Root"]

        subgraph Core["Core OU"]
            MGMT["Management<br>Account"]
            LOG["Log Archive"]
            AUDIT["Audit/Security"]
        end

        subgraph Workloads["Workloads OU"]
            subgraph ProdOU["Production OU"]
                PROD["Production"]
            end
            subgraph NonProd["Non-Production OU"]
                STAGING["Staging"]
                DEV["Development"]
            end
        end

        subgraph Sandbox["Sandbox OU"]
            SB1["Developer<br>Sandbox 1"]
            SB2["Developer<br>Sandbox 2"]
        end
    end

    ROOT --> Core & Workloads & Sandbox

    style Org fill:#e3f2fd,color:#000
    style Core fill:#ffcdd2,color:#000
    style Workloads fill:#c8e6c9,color:#000
    style ProdOU fill:#e8f5e9,color:#000
    style NonProd fill:#e8f5e9,color:#000
    style Sandbox fill:#fff9c4,color:#000
    style ROOT fill:#fff,color:#000
    style MGMT fill:#fff,color:#000
    style LOG fill:#fff,color:#000
    style AUDIT fill:#fff,color:#000
    style PROD fill:#fff,color:#000
    style STAGING fill:#fff,color:#000
    style DEV fill:#fff,color:#000
    style SB1 fill:#fff,color:#000
    style SB2 fill:#fff,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### MedVault SCPs

SCPs are applied at different levels based on the security requirements of each OU. Root-level SCPs
apply to all accounts, while OU-specific SCPs target particular environments:

| OU         | SCP                   | Purpose                |
| ---------- | --------------------- | ---------------------- |
| Root       | DenyLeaveOrg          | Prevent account escape |
| Root       | RequireCloudTrail     | Maintain audit trail   |
| Production | DenyDeleteData        | Protect PHI            |
| Production | RequireEncryption     | HIPAA compliance       |
| Sandbox    | RestrictRegions       | Cost control           |
| Sandbox    | DenyExpensiveServices | Budget limits          |

### MedVault Account Strategy

Each account has a clear purpose and access pattern. This separation ensures that developers can
experiment in sandboxes without risking production PHI (Protected Health Information):

| Account     | Purpose             | Access                |
| ----------- | ------------------- | --------------------- |
| Management  | Org admin, billing  | Org admins only       |
| Log Archive | Centralized logs    | Security team (read)  |
| Audit       | Security tools      | Security team         |
| Production  | PHI, patient data   | Limited, audited      |
| Staging     | Integration testing | QA team               |
| Development | Feature development | Dev team              |
| Sandboxes   | Experimentation     | Individual developers |

## What Could Go Wrong?

The multi-account setup is complete. Developers can't accidentally access production. SCPs enforce
security guardrails. But the CISO asks:

> "We've built strong preventive controls. But what if something still gets through? How do we
> detect if someone is trying to breach our systems or if there's unusual activity?"

Time for threat detection.

## Exam Tips

- **Management account not affected by SCPs** - Keep workloads out of it
- **SCPs limit, don't grant** - Still need IAM permissions
- **Effective = SCP intersection with IAM** - Both must allow
- **Control Tower = Landing Zone + Guardrails** - Automated multi-account
- **Preventive = SCP, Detective = Config** - Know the difference
- **RAM for sharing** - Subnets, Transit Gateway, Aurora
- **Organization Trail** - All accounts, central bucket

## SAA Exam Concepts

### Must-Know for This Phase

| Concept            | Key Points                                                       |
| ------------------ | ---------------------------------------------------------------- |
| Organizations      | Root, OUs, management account, member accounts                   |
| SCPs               | Limit permissions, don't grant, inherited down                   |
| SCP Evaluation     | Effective = SCP allows AND IAM allows                            |
| Management Account | Not affected by SCPs, don't run workloads                        |
| Control Tower      | Landing zone, guardrails, account factory                        |
| Guardrails         | Preventive (SCP), Detective (Config), Proactive (CloudFormation) |
| RAM                | Share resources across accounts without copying                  |
