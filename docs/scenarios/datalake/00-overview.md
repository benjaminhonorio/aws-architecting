# Data Lake Journey: The DataLake Corp Story

## The Business

You're the lead data engineer at **DataLake Corp**, a retail analytics company that helps e-commerce
businesses understand their customers through data. The company started as a small consulting firm
doing Excel reports, but rapid growth demands a modern cloud data platform.

## Learning Objectives

This evolving scenario teaches you AWS analytics and data services through realistic business
decisions:

- [**Phase 1**: Data Lake Foundation](scenarios/datalake/phases/phase-1-data-lake-foundation.md) (S3
  Data Lake, Storage Classes, Lifecycle)
- [**Phase 2**: Query Without Servers](scenarios/datalake/phases/phase-2-query-without-servers.md)
  (Athena, Glue Catalog)
- [**Phase 3**: ETL Pipelines](scenarios/datalake/phases/phase-3-etl-pipelines.md) (AWS Glue, Step
  Functions)
- [**Phase 4**: Enterprise Warehouse](scenarios/datalake/phases/phase-4-enterprise-warehouse.md)
  (Redshift, Redshift Spectrum, AQUA)
- [**Phase 5**: Real-Time Analytics](scenarios/datalake/phases/phase-5-real-time-analytics.md)
  (Kinesis Streams, Firehose, Analytics)
- [**Phase 6**: Data Governance](scenarios/datalake/phases/phase-6-data-governance.md) (Lake
  Formation, Fine-Grained Access)

## Architecture Evolution Map

```mermaid
flowchart LR
    P1["Phase 1<br>S3 Data Lake<br>Foundation"] --> P2["Phase 2<br>Athena<br>Serverless SQL"]
    P2 --> P3["Phase 3<br>Glue ETL<br>Pipelines"]
    P3 --> P4["Phase 4<br>Redshift<br>Warehouse"]
    P4 --> P5["Phase 5<br>Kinesis<br>Real-Time"]
    P5 --> P6["Phase 6<br>Lake Formation<br>Governance"]

    style P1 fill:#e3f2fd,color:#000
    style P2 fill:#e3f2fd,color:#000
    style P3 fill:#fff3e0,color:#000
    style P4 fill:#fff3e0,color:#000
    style P5 fill:#e8f5e9,color:#000
    style P6 fill:#e8f5e9,color:#000
    linkStyle default stroke:#000,stroke-width:2px
```

## SAA-C03 Exam Coverage

This scenario covers critical analytics topics that appear frequently on the exam:

| Service                        | Exam Weight | Phase |
| ------------------------------ | ----------- | ----- |
| Amazon S3 (Data Lake patterns) | High        | 1-6   |
| Amazon Athena                  | High        | 2, 4  |
| AWS Glue (ETL, Catalog)        | High        | 2, 3  |
| Amazon Redshift / Spectrum     | High        | 4     |
| Amazon Kinesis Family          | High        | 5     |
| AWS Lake Formation             | Medium      | 6     |
| Amazon ElastiCache             | Medium      | 4     |

## How to Use This Guide

Each phase includes:

1. **Business Context** - What's happening at DataLake Corp
2. **Architecture Decision** - What we're building and WHY
3. **Key Concepts** - SAA exam-relevant knowledge
4. **Diagrams** - Visual representation of the architecture
5. **Exam Tips** - Key points for the SAA certification
