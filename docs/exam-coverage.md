# SAA-C03 Exam Domain Coverage

Quick reference to find content by exam domain. Use this to target specific areas for review.

---

## Domain 1: Design Secure Architectures (30%)

| Topic                         | Primary                                                                  | Supporting                                                                                                                    |
| ----------------------------- | ------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------- |
| IAM policies & roles          | [MedVault P1](/scenarios/medvault/phases/phase-1-identity-foundation.md) | [TechBooks P4](/scenarios/techbooks/phases/phase-4-auto-scaling.md)                                                           |
| Encryption at rest/in transit | [MedVault P2](/scenarios/medvault/phases/phase-2-data-protection.md)     | [TechBooks P5](/scenarios/techbooks/phases/phase-5-going-global.md)                                                           |
| VPC security & endpoints      | [MedVault P3](/scenarios/medvault/phases/phase-3-network-security.md)    | [TechBooks P1](/scenarios/techbooks/phases/phase-1-mvp-launch.md), [P6](/scenarios/techbooks/phases/phase-6-modernization.md) |
| Logging & compliance          | [MedVault P4](/scenarios/medvault/phases/phase-4-logging-monitoring.md)  | -                                                                                                                             |
| Multi-account governance      | [MedVault P5](/scenarios/medvault/phases/phase-5-multi-account.md)       | -                                                                                                                             |
| Threat detection              | [MedVault P6](/scenarios/medvault/phases/phase-6-threat-detection.md)    | -                                                                                                                             |
| Security groups & NACLs       | [TechBooks P1](/scenarios/techbooks/phases/phase-1-mvp-launch.md)        | [MedVault P3](/scenarios/medvault/phases/phase-3-network-security.md)                                                         |

---

## Domain 2: Design Resilient Architectures (26%)

| Topic                        | Primary                                                                                                                         | Supporting                                                              |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| Multi-AZ & failover          | [TechBooks P3](/scenarios/techbooks/phases/phase-3-high-availability.md)                                                        | [ShipFast P4](/scenarios/shipfast/phases/phase-4-database-migration.md) |
| Auto Scaling & health checks | [TechBooks P4](/scenarios/techbooks/phases/phase-4-auto-scaling.md)                                                             | -                                                                       |
| Backup & recovery (RTO/RPO)  | [TechBooks P2](/scenarios/techbooks/phases/phase-2-database-separation.md)                                                      | [ShipFast P4](/scenarios/shipfast/phases/phase-4-database-migration.md) |
| Hybrid connectivity          | [ShipFast P2](/scenarios/shipfast/phases/phase-2-vpn-connection.md), [P3](/scenarios/shipfast/phases/phase-3-direct-connect.md) | -                                                                       |
| Database replication         | [TechBooks P3](/scenarios/techbooks/phases/phase-3-high-availability.md)                                                        | [ShipFast P4](/scenarios/shipfast/phases/phase-4-database-migration.md) |
| Event-driven resilience      | [TechBooks P6](/scenarios/techbooks/phases/phase-6-modernization.md)                                                            | -                                                                       |

---

## Domain 3: Design High-Performing Architectures (24%)

| Topic                    | Primary                                                                    | Supporting                                                              |
| ------------------------ | -------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| CloudFront & CDN         | [TechBooks P5](/scenarios/techbooks/phases/phase-5-going-global.md)        | -                                                                       |
| Caching (ElastiCache)    | [TechBooks P6](/scenarios/techbooks/phases/phase-6-modernization.md)       | -                                                                       |
| Load balancing (ALB/NLB) | [TechBooks P4](/scenarios/techbooks/phases/phase-4-auto-scaling.md)        | -                                                                       |
| Database performance     | [TechBooks P2](/scenarios/techbooks/phases/phase-2-database-separation.md) | [ShipFast P4](/scenarios/shipfast/phases/phase-4-database-migration.md) |
| Route 53 routing         | [TechBooks P5](/scenarios/techbooks/phases/phase-5-going-global.md)        | -                                                                       |
| Direct Connect bandwidth | [ShipFast P3](/scenarios/shipfast/phases/phase-3-direct-connect.md)        | -                                                                       |

---

## Domain 4: Design Cost-Optimized Architectures (20%)

| Topic                              | Primary                                                                                                                       | Supporting                                                            |
| ---------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------- |
| Reserved Instances & Savings Plans | [TechBooks P6](/scenarios/techbooks/phases/phase-6-modernization.md)                                                          | -                                                                     |
| Right-sizing                       | [TechBooks P1](/scenarios/techbooks/phases/phase-1-mvp-launch.md), [P6](/scenarios/techbooks/phases/phase-6-modernization.md) | -                                                                     |
| VPC Endpoints (cost savings)       | [TechBooks P6](/scenarios/techbooks/phases/phase-6-modernization.md)                                                          | [MedVault P3](/scenarios/medvault/phases/phase-3-network-security.md) |
| Data transfer optimization         | [ShipFast P3](/scenarios/shipfast/phases/phase-3-direct-connect.md)                                                           | [TechBooks P5](/scenarios/techbooks/phases/phase-5-going-global.md)   |
| Serverless cost model              | [TechBooks P6](/scenarios/techbooks/phases/phase-6-modernization.md)                                                          | -                                                                     |
| Storage tiering                    | [ShipFast P5](/scenarios/shipfast/phases/phase-5-hybrid-storage.md)                                                           | [TechBooks P5](/scenarios/techbooks/phases/phase-5-going-global.md)   |

---

## Scenario Focus Summary

| Scenario  | Primary Domains                                    | Best For                                  |
| --------- | -------------------------------------------------- | ----------------------------------------- |
| TechBooks | Resilient (26%), High-Performing (24%), Cost (20%) | Cloud-native scaling patterns             |
| ShipFast  | Resilient (26%), Cost (20%)                        | Migration strategies, hybrid connectivity |
| MedVault  | Secure (30%)                                       | Security-first architecture, compliance   |
