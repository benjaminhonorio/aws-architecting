# Phase 7: Container Deployment

## Business Context

**Situation:** EventPro's microservices architecture is working well, but the DevOps team is
struggling. They manage 15 different Lambda functions, each with different runtimes and
dependencies. Deployment takes hours and debugging is painful.

**The DevOps lead's frustration:** "Lambda cold starts are killing our P99 latency. Some functions
need 2GB memory. We're spending more time managing deployment packages than writing code."

**Requirements:**

- Consistent deployment across all microservices
- Reduce cold start latency for critical paths
- Run some workloads on-premises (venue kiosks)
- Secure service-to-service communication
- Private API access for internal services

---

## Step 1: ECS vs EKS Decision

### Container Orchestration Options

```mermaid
flowchart TB
    subgraph ECS["Amazon ECS"]
        ECSF["Fargate<br>Serverless containers"]
        ECSE["EC2<br>Self-managed instances"]
        ECSA["ECS Anywhere<br>On-premises"]
    end

    subgraph EKS["Amazon EKS"]
        EKSF["Fargate<br>Serverless pods"]
        EKSE["EC2<br>Managed node groups"]
        EKSA["EKS Anywhere<br>On-premises"]
    end

    style ECS fill:#e3f2fd,color:#000
    style EKS fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### ECS vs EKS Comparison

| Aspect             | ECS                  | EKS                          |
| ------------------ | -------------------- | ---------------------------- |
| **Complexity**     | Simpler, AWS-native  | Complex, Kubernetes standard |
| **Portability**    | AWS-only             | Multi-cloud, on-prem         |
| **Learning curve** | Lower                | Higher                       |
| **Ecosystem**      | AWS integrations     | Kubernetes ecosystem         |
| **Cost**           | No control plane fee | $0.10/hour per cluster       |
| **Best for**       | AWS-first teams      | K8s expertise, multi-cloud   |

> **SAA Exam Tip:** "Simple container orchestration on AWS" = ECS. "Kubernetes with portability" =
> EKS. "Serverless containers" = Fargate (works with both).

---

## Step 2: ECS Anywhere

### Run ECS On-Premises

EventPro's venue kiosks run locally but need the same deployment model:

```mermaid
flowchart LR
    subgraph AWS["AWS Region"]
        ECS["ECS Control Plane"]
        ECR["ECR<br>Container Images"]
    end

    subgraph Venue["Venue (On-Premises)"]
        Agent["ECS Agent"]
        Kiosk["Kiosk App<br>Container"]
    end

    subgraph Cloud["AWS Cloud"]
        API["Ticket API"]
        DB["DynamoDB"]
    end

    ECS -->|"Task management"| Agent
    ECR -->|"Pull images"| Agent
    Agent --> Kiosk
    Kiosk --> API
    API --> DB

    style AWS fill:#e3f2fd,color:#000
    style Venue fill:#fff9c4,color:#000
    style Cloud fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### ECS Anywhere Features

| Feature              | Description                         |
| -------------------- | ----------------------------------- |
| **Same APIs**        | Use ECS APIs for on-prem and cloud  |
| **SSM integration**  | Systems Manager for connectivity    |
| **Task definitions** | Same task defs work everywhere      |
| **ECR integration**  | Pull images from ECR                |
| **Pricing**          | $0.01025/hour per external instance |

### Registration Process

```bash
# Generate activation key in AWS
aws ssm create-activation \
    --iam-role ECSAnywhereRole \
    --registration-limit 10

# On the external instance
curl -o ecs-anywhere-install.sh \
    https://amazon-ecs-agent.s3.amazonaws.com/ecs-anywhere-install.sh
sudo bash ecs-anywhere-install.sh \
    --region us-east-1 \
    --cluster eventpro-kiosks \
    --activation-id $ACTIVATION_ID \
    --activation-code $ACTIVATION_CODE
```

> **SAA Exam Tip:** "Run containers on-premises with ECS management" = ECS Anywhere. "Kubernetes
> on-premises" = EKS Anywhere. "AWS infrastructure on-premises" = Outposts.

---

## Step 3: EKS and IRSA

### IAM Roles for Service Accounts

**IRSA (IAM Roles for Service Accounts)** provides fine-grained IAM permissions to Kubernetes pods:

```mermaid
flowchart TB
    subgraph EKS["EKS Cluster"]
        subgraph Pods["Kubernetes Pods"]
            P1["Ticket Pod<br>SA: ticket-sa"]
            P2["Payment Pod<br>SA: payment-sa"]
        end
        OIDC["OIDC Provider"]
    end

    subgraph IAM["AWS IAM"]
        R1["TicketRole<br>DynamoDB access"]
        R2["PaymentRole<br>Secrets Manager access"]
    end

    subgraph Services["AWS Services"]
        DDB["DynamoDB"]
        SM["Secrets Manager"]
    end

    P1 --> OIDC
    P2 --> OIDC
    OIDC --> R1
    OIDC --> R2
    R1 --> DDB
    R2 --> SM

    style EKS fill:#e3f2fd,color:#000
    style IAM fill:#fff9c4,color:#000
    style Services fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### Why IRSA Over Node Roles

| Approach                 | Scope            | Security        |
| ------------------------ | ---------------- | --------------- |
| **EC2 Instance Profile** | All pods on node | Over-privileged |
| **IRSA**                 | Individual pods  | Least privilege |

### Configuring IRSA

```yaml
# Service Account with IAM role annotation
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ticket-service
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/TicketServiceRole

---
# Pod using the service account
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ticket-service
spec:
  template:
    spec:
      serviceAccountName: ticket-service
      containers:
        - name: ticket-app
          image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/ticket-service:v1
```

> **SAA Exam Tip:** "Fine-grained IAM for Kubernetes pods" = IRSA. "All pods share same IAM role" =
> Instance profile (avoid for security).

---

## Step 4: API Gateway Private Integration

### VPC-Private APIs

EventPro's internal services should not be exposed to the internet:

```mermaid
flowchart LR
    subgraph Internet["Internet"]
        Client["Public Client"]
    end

    subgraph VPC["EventPro VPC"]
        subgraph Public["Public Subnet"]
            ALB["Application<br>Load Balancer"]
        end

        subgraph Private["Private Subnet"]
            APIGW["API Gateway<br>Private Endpoint"]
            VPCLink["VPC Link"]
            NLB["Network<br>Load Balancer"]
            ECS["ECS Tasks"]
        end
    end

    Client --> ALB
    ALB --> ECS
    APIGW --> VPCLink
    VPCLink --> NLB
    NLB --> ECS

    style Internet fill:#ffcdd2,color:#000
    style VPC fill:#e3f2fd,color:#000
    style Public fill:#c8e6c9,color:#000
    style Private fill:#fff9c4,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### Private API Gateway Types

| Type                           | Access              | Use Case               |
| ------------------------------ | ------------------- | ---------------------- |
| **Regional + Resource Policy** | IP/VPC restrictions | Limited public access  |
| **Private**                    | VPC endpoints only  | Internal microservices |

### VPC Link Configuration

```yaml
# Create VPC Link for private integration
Resources:
  VPCLink:
    Type: AWS::ApiGatewayV2::VpcLink
    Properties:
      Name: eventpro-internal
      SubnetIds:
        - !Ref PrivateSubnet1
        - !Ref PrivateSubnet2
      SecurityGroupIds:
        - !Ref VPCLinkSecurityGroup

  InternalAPI:
    Type: AWS::ApiGatewayV2::Api
    Properties:
      Name: eventpro-internal-api
      ProtocolType: HTTP

  Integration:
    Type: AWS::ApiGatewayV2::Integration
    Properties:
      ApiId: !Ref InternalAPI
      IntegrationType: HTTP_PROXY
      IntegrationUri: !Ref ALBListener
      IntegrationMethod: ANY
      ConnectionType: VPC_LINK
      ConnectionId: !Ref VPCLink
```

> **SAA Exam Tip:** "API Gateway to private resources" = VPC Link. "Private API accessible only from
> VPC" = Private API with VPC endpoint.

---

## Step 5: Container Security Best Practices

### ECR Image Scanning

```mermaid
flowchart LR
    subgraph Dev["Development"]
        Build["Build Image"]
    end

    subgraph ECR["Amazon ECR"]
        Push["Push to ECR"]
        Scan["Vulnerability Scan"]
        Sign["Image Signing"]
    end

    subgraph Deploy["Deployment"]
        Policy["Admission Policy<br>Block critical vulns"]
        ECS["ECS/EKS"]
    end

    Build --> Push
    Push --> Scan
    Scan --> Sign
    Sign --> Policy
    Policy -->|"Passed"| ECS

    style Dev fill:#fff9c4,color:#000
    style ECR fill:#e3f2fd,color:#000
    style Deploy fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### Security Features

| Feature               | Service         | Description                 |
| --------------------- | --------------- | --------------------------- |
| **Image scanning**    | ECR             | CVE vulnerability detection |
| **Image signing**     | Signer          | Cryptographic signature     |
| **Secrets injection** | Secrets Manager | Inject secrets at runtime   |
| **Task IAM roles**    | ECS/EKS         | Per-task IAM permissions    |
| **Network policies**  | EKS             | Pod-to-pod traffic control  |

> **SAA Exam Tip:** "Scan container images for vulnerabilities" = ECR image scanning. "Prevent
> unsigned images" = Container image signing with admission policies.

---

## Step 6: ECS Task Networking

### awsvpc Network Mode

```mermaid
flowchart TB
    subgraph VPC["EventPro VPC"]
        subgraph Subnet["Private Subnet"]
            ENI1["ENI<br>10.0.1.10"]
            ENI2["ENI<br>10.0.1.11"]
            ENI3["ENI<br>10.0.1.12"]
        end

        subgraph Tasks["ECS Tasks"]
            T1["Task 1<br>→ ENI1"]
            T2["Task 2<br>→ ENI2"]
            T3["Task 3<br>→ ENI3"]
        end

        SG["Security Group<br>Controls task traffic"]
    end

    ENI1 --> T1
    ENI2 --> T2
    ENI3 --> T3
    SG --> ENI1
    SG --> ENI2
    SG --> ENI3

    style VPC fill:#e3f2fd,color:#000
    style Subnet fill:#c8e6c9,color:#000
    style Tasks fill:#fff9c4,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### Network Mode Comparison

| Mode       | IP Address        | Security Group | Use Case             |
| ---------- | ----------------- | -------------- | -------------------- |
| **awsvpc** | Own ENI, own IP   | Per-task       | **Recommended**      |
| **bridge** | Host port mapping | Host-level     | Legacy EC2           |
| **host**   | Host network      | Host-level     | Performance-critical |
| **none**   | No networking     | N/A            | Batch jobs           |

> **SAA Exam Tip:** "ECS task with its own security group" = awsvpc network mode. Fargate REQUIRES
> awsvpc mode.

---

## Step 7: Service Mesh with App Mesh

### Microservices Communication

For complex service-to-service communication, use **AWS App Mesh**:

```mermaid
flowchart LR
    subgraph Mesh["AWS App Mesh"]
        subgraph ServiceA["Ticket Service"]
            A["App"]
            EA["Envoy Proxy"]
        end

        subgraph ServiceB["Payment Service"]
            B["App"]
            EB["Envoy Proxy"]
        end

        subgraph ServiceC["Notification Service"]
            C["App"]
            EC["Envoy Proxy"]
        end
    end

    EA <-->|"mTLS"| EB
    EB <-->|"mTLS"| EC
    EA <-->|"mTLS"| EC

    style Mesh fill:#e3f2fd,color:#000
    style ServiceA fill:#c8e6c9,color:#000
    style ServiceB fill:#c8e6c9,color:#000
    style ServiceC fill:#c8e6c9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### App Mesh Features

| Feature                | Description                      |
| ---------------------- | -------------------------------- |
| **Traffic control**    | Routing rules, retries, timeouts |
| **Observability**      | Metrics, logs, traces via X-Ray  |
| **Security**           | mTLS between services            |
| **Canary deployments** | Weighted routing for rollouts    |

> **SAA Exam Tip:** "Service mesh for microservices on AWS" = App Mesh. "mTLS between containers" =
> App Mesh with Envoy.

---

## Step 8: EventPro Container Architecture

### Complete Solution

```mermaid
flowchart TB
    subgraph Internet["Internet"]
        Users["Ticket Buyers"]
    end

    subgraph VPC["EventPro VPC"]
        subgraph Public["Public Subnet"]
            ALB["Application<br>Load Balancer"]
        end

        subgraph Private["Private Subnet"]
            subgraph EKS["EKS Cluster"]
                P1["Ticket Pod<br>IRSA: TicketRole"]
                P2["Payment Pod<br>IRSA: PaymentRole"]
            end
            subgraph ECS["ECS Cluster"]
                T1["Notification Task"]
                T2["Email Task"]
            end
        end
    end

    subgraph Venue["On-Premises"]
        Kiosk["ECS Anywhere<br>Venue Kiosks"]
    end

    subgraph Services["AWS Services"]
        DDB["DynamoDB"]
        SQS["SQS"]
        ECR["ECR"]
    end

    Users --> ALB
    ALB --> P1
    P1 --> DDB
    P1 --> P2
    P2 --> SQS
    SQS --> T1
    T1 --> T2
    Kiosk --> ALB
    ECR --> EKS
    ECR --> ECS
    ECR --> Kiosk

    style Internet fill:#ffcdd2,color:#000
    style VPC fill:#e3f2fd,color:#000
    style EKS fill:#c8e6c9,color:#000
    style ECS fill:#c8e6c9,color:#000
    style Venue fill:#fff9c4,color:#000
    style Services fill:#bbdefb,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

### Architecture Decisions

| Component           | Choice           | Reason                      |
| ------------------- | ---------------- | --------------------------- |
| **Core APIs**       | EKS with Fargate | Team has K8s expertise      |
| **Background jobs** | ECS with Fargate | Simpler for async tasks     |
| **Venue kiosks**    | ECS Anywhere     | Consistent deployment model |
| **IAM**             | IRSA             | Least privilege per pod     |
| **Networking**      | awsvpc           | Per-task security groups    |

---

## Exam Tips Summary

| Topic            | Key Point                                             |
| ---------------- | ----------------------------------------------------- |
| **ECS vs EKS**   | ECS = simpler, AWS-native; EKS = Kubernetes, portable |
| **Fargate**      | Serverless containers, works with ECS and EKS         |
| **ECS Anywhere** | Run ECS on-premises, same APIs and task definitions   |
| **EKS Anywhere** | Kubernetes on-premises with EKS management            |
| **IRSA**         | Fine-grained IAM for individual Kubernetes pods       |
| **VPC Link**     | API Gateway to private VPC resources                  |
| **Private API**  | API Gateway accessible only from VPC                  |
| **awsvpc mode**  | Each task gets its own ENI and security group         |
| **App Mesh**     | Service mesh for mTLS and traffic control             |
| **ECR scanning** | Container image vulnerability detection               |

---

**[← Back to EventPro Overview](../00-overview.md)**
