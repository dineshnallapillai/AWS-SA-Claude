# AWS Solutions Architect Professional (SAP-C02) — Study Notes Prompt

## How to Use

Copy the prompt below and replace `[TOPIC NAME]` with any topic from the checklist at the bottom. Use one topic per session for best results.

---

## The Prompt

```text
I am preparing for the AWS Solutions Architect Professional (SAP-C02) exam.
Generate exhaustive study notes on the topic: **[TOPIC NAME]**

Structure the notes as follows:

### 1. Core Concepts & Theory
- Detailed explanation of the service/concept
- Architecture patterns and how it fits into the AWS ecosystem
- Key components, features, and terminology
- Default limits, quotas, and constraints worth memorizing

### 2. Design Patterns & Best Practices
- When to use vs. when NOT to use (anti-patterns)
- Common architectural patterns (multi-account, multi-region, hybrid, migration)
- Well-Architected Framework alignment (reliability, security, cost, performance, operational excellence, sustainability)
- Integration points with other AWS services

### 3. Security & Compliance
- IAM policies, resource policies, SCPs relevant to this topic
- Encryption options (at rest, in transit, key management)
- Logging, auditing, and monitoring considerations
- Compliance and governance patterns

### 4. Cost Optimization
- Pricing model and billing nuances
- Cost-saving strategies and common cost traps
- Comparison with alternatives (when is a cheaper option acceptable?)

### 5. High Availability, Disaster Recovery & Resilience
- HA architecture patterns (multi-AZ, multi-region, active-active, active-passive)
- RPO/RTO considerations
- Backup, failover, and recovery strategies

### 6. Exam-Focused Section

#### Straightforward Questions (5-7 examples)
- Questions that directly test knowledge of this topic
- Provide the answer and a one-line explanation for each

#### Tricky / Scenario-Based Questions (5-7 examples)
- Questions with distractors or where two answers seem correct
- Explain WHY the wrong answers are wrong (this is critical)
- Highlight the keywords/phrases in the question that point to the right answer

#### Common Exam Traps & Pitfalls
- Frequently confused services or features
- Subtle differences candidates often miss
- "Sounds right but is wrong" patterns

### 7. Cheat Sheet
- One-page summary of must-know facts for exam day
- Key differentiators from similar services
- Decision flowchart: "If the question says X, think Y"

Use real-world AWS scenarios wherever possible. Be precise about
service names, features, and limitations — do not generalize.
```

---

## SAP-C02 Exam Domains & Topics Checklist

Work through each topic using the prompt above.

### Domain 1 — Organizational Complexity (26%)

- [ ] Multi-Account Strategy & AWS Organizations
- [ ] Service Control Policies (SCPs) & Permission Boundaries
- [ ] AWS Control Tower & Landing Zones
- [ ] Cross-Account Access & Resource Sharing (RAM)
- [ ] Centralized Logging & Security (CloudTrail, Config, Security Hub, GuardDuty)
- [ ] AWS IAM Identity Center (SSO) & Federation (SAML, OIDC)

### Domain 2 — Design for New Solutions (29%)

**Compute**
- [ ] EC2 (placement groups, Nitro, AMIs, instance store vs EBS)
- [ ] Lambda (concurrency, layers, destinations, VPC integration)
- [ ] ECS / EKS / Fargate (when to use which)

**Networking**
- [ ] VPC Design (subnets, NACLs, security groups, flow logs)
- [ ] Transit Gateway & VPC Peering
- [ ] Direct Connect & VPN (BGP, LAG, failover)
- [ ] PrivateLink & VPC Endpoints
- [ ] Route 53 (routing policies, health checks, DNS failover)
- [ ] Global Accelerator vs CloudFront

**Storage**
- [ ] S3 (storage classes, replication, lifecycle, object lock, access points)
- [ ] EBS (volume types, snapshots, encryption)
- [ ] EFS vs FSx (Lustre, Windows, NetApp ONTAP, OpenZFS)
- [ ] Storage Gateway (File, Volume, Tape)

**Databases**
- [ ] RDS (Multi-AZ, read replicas, proxy, custom)
- [ ] Aurora (Global Database, Multi-Master, Serverless v2, cloning)
- [ ] DynamoDB (DAX, Global Tables, streams, capacity modes)
- [ ] ElastiCache (Redis vs Memcached, cluster mode)
- [ ] Redshift (Spectrum, concurrency scaling, RA3 nodes)

**Application Integration**
- [ ] SQS (standard vs FIFO, dead-letter queues, visibility timeout)
- [ ] SNS (fan-out, filtering, FIFO topics)
- [ ] EventBridge (rules, event buses, cross-account)
- [ ] Step Functions (standard vs express, error handling)
- [ ] API Gateway (REST vs HTTP vs WebSocket, caching, throttling)
- [ ] AppSync (GraphQL, real-time, resolvers)

**Analytics**
- [ ] Kinesis (Streams vs Firehose vs Analytics vs Video Streams)
- [ ] Athena & Glue (data catalog, crawlers, ETL)
- [ ] Lake Formation (permissions, data lake)
- [ ] EMR (Spark, Hadoop, managed scaling)
- [ ] OpenSearch (formerly Elasticsearch)

**AI/ML (high-level)**
- [ ] SageMaker, Comprehend, Rekognition, Textract, Transcribe

### Domain 3 — Continuous Improvement (25%)

- [ ] CI/CD: CodePipeline, CodeBuild, CodeDeploy
- [ ] Infrastructure as Code: CloudFormation, CDK, SAM
- [ ] CloudWatch (metrics, logs, alarms, composite alarms, dashboards)
- [ ] X-Ray (tracing, service map)
- [ ] Auto Scaling & Elasticity Patterns
- [ ] Performance Optimization (caching, CDN, read replicas, Global Accelerator)
- [ ] Systems Manager (Parameter Store, Session Manager, Patch Manager, Run Command)
- [ ] AWS Config (rules, conformance packs, remediation)
- [ ] Trusted Advisor & Compute Optimizer

### Domain 4 — Migration & Modernization (20%)

- [ ] Migration Strategies (7 Rs: Rehost, Replatform, Refactor, Repurchase, Retire, Retain, Relocate)
- [ ] AWS Migration Hub & Application Discovery Service
- [ ] AWS MGN (Application Migration Service)
- [ ] DMS & SCT (Database Migration Service & Schema Conversion Tool)
- [ ] Hybrid Architectures (Outposts, Wavelength, Local Zones)
- [ ] Snow Family (Snowcone, Snowball Edge, Snowmobile)
- [ ] DataSync & Transfer Family
- [ ] Containerization & Modernization Patterns (monolith to microservices)

---

## Follow-Up Prompts

After generating notes for a topic, use these to go deeper:

```text
Give me 10 more tricky scenario-based questions on [TOPIC NAME]
```

```text
Compare [Service A] vs [Service B] in a table with columns:
Use Case, Limits, Pricing, HA Model, Exam Tip
```

```text
When does the SAP-C02 exam expect me to pick [Service A] over [Service B]?
Give specific keywords and scenarios.
```

```text
List all the "gotcha" differences between [Service A] and [Service B]
that the exam tests.
```

```text
Create a one-page decision tree for choosing the right [compute/storage/database/networking]
service based on exam question keywords.
```

---

## Study Tips

1. **One topic per session** — depth beats breadth for retention
2. **Focus on the "why wrong" answers** — the exam tests elimination skills
3. **Watch for keywords** — "cost-effective", "least operational overhead", "most secure", "minimize latency" each point to different answers
4. **Multi-service questions dominate** — practice connecting 2-3 services in an architecture
5. **Time management** — 75 questions in 180 minutes = ~2.4 min per question; flag and move on
