# AWS SAP-C02 Study Notes: Migration Strategies (7 Rs)

## 1. Core Concepts & Theory

### The 7 Rs of Migration

| Strategy | Definition | Effort | Risk | Cost Impact |
|----------|-----------|--------|------|-------------|
| **Rehost** (Lift & Shift) | Move as-is to AWS with no code changes | Low | Low | Moderate savings (20-30%) |
| **Replatform** (Lift, Tinker & Shift) | Minor optimizations without changing core architecture | Low-Medium | Low-Medium | Better savings than rehost |
| **Refactor / Re-architect** | Redesign using cloud-native features | High | High | Maximum long-term savings |
| **Repurchase** (Drop & Shop) | Move to a different product (often SaaS) | Medium | Medium | Variable (license vs subscription) |
| **Retire** | Decommission applications no longer needed | None | None | Immediate savings |
| **Retain** (Revisit) | Keep in current environment (not ready to migrate) | None | None | No change |
| **Relocate** | Move to AWS without purchasing new hardware (VMware Cloud on AWS) | Low | Low | Operational savings |

### Rehost (Lift & Shift)

- Move applications to AWS without modifications
- Use AWS Application Migration Service (AWS MGN) for automated lift-and-shift
- Typically the fastest migration path
- Good for large-scale migrations where speed matters more than optimization
- Applications can be optimized AFTER migration (two-phase approach)
- Supports both physical and virtual servers
- **Common targets:** EC2 instances with same OS, same configuration
- **Typical savings:** 20-30% TCO reduction from infrastructure consolidation alone

### Replatform (Lift, Tinker & Shift)

- Make a few cloud optimizations without changing core architecture
- Examples:
  - Migrate on-premises MySQL to Amazon RDS (no schema changes)
  - Move from self-managed Redis to ElastiCache
  - Replace local file storage with S3
  - Switch to Elastic Load Balancer instead of hardware load balancers
  - Migrate to managed containers (ECS) without refactoring the app
- **Key distinction from Refactor:** Application code stays largely the same; you're swapping infrastructure components for managed equivalents

### Refactor / Re-architect

- Fundamentally redesign the application using cloud-native services
- Driven by strong business need: scalability, agility, performance
- Examples:
  - Monolith to microservices
  - Move from relational DB to DynamoDB for specific workloads
  - Implement event-driven architecture with Lambda + EventBridge
  - Decouple with SQS/SNS
  - Replace batch processing with streaming (Kinesis)
- **Most expensive and time-consuming** but yields greatest long-term benefits
- Consider when: current architecture can't meet future requirements

### Repurchase (Drop & Shop)

- Replace existing application with a SaaS or marketplace solution
- Examples:
  - On-premises CRM → Salesforce
  - Self-hosted email → Amazon WorkMail or Microsoft 365
  - Custom HR system → Workday
  - Self-managed CMS → SaaS CMS
  - On-premises databases → Amazon RDS or purpose-built databases
- **Key consideration:** Data migration, user training, integration rewiring
- Often involves moving from perpetual licensing to subscription model

### Retire

- Identify applications that are no longer useful and can be turned off
- Typically 10-20% of an enterprise application portfolio can be retired
- Discovery process reveals:
  - Duplicate applications
  - Applications with no active users
  - Applications replaced by other systems
- **Immediate cost savings** — stop paying for hardware, licenses, maintenance
- Must verify no dependencies before decommissioning

### Retain (Revisit)

- Keep applications in their current environment
- Reasons to retain:
  - Recently upgraded (not worth migrating again)
  - Compliance or regulatory constraints
  - Unresolved dependencies
  - Low priority / limited business value to migrate now
  - Requires significant refactoring that isn't justified yet
- **Not a permanent decision** — revisit periodically
- Still part of the migration plan (explicitly decided, not forgotten)

### Relocate (VMware Cloud on AWS)

- Move VMware-based infrastructure to VMware Cloud on AWS
- No need to purchase new hardware, rewrite apps, or modify operations
- Maintains VMware tooling (vSphere, vSAN, NSX)
- Useful for:
  - Data center evacuation with tight timelines
  - Organizations heavily invested in VMware
  - Maintaining operational consistency while gaining AWS proximity
- **Key distinction:** Infrastructure-level move, not application-level

### Migration Process & Supporting Services

| Phase | AWS Service | Purpose |
|-------|-------------|---------|
| Assess | Migration Hub, Application Discovery Service | Portfolio analysis, dependency mapping |
| Mobilize | Migration Hub Orchestrator | Planning, setting up landing zone |
| Migrate | MGN, DMS, DataSync, Snow Family | Execute migration |
| Optimize | Compute Optimizer, Trusted Advisor, Cost Explorer | Right-size, optimize |

### AWS Cloud Adoption Framework (CAF) Perspectives

| Perspective | Focus Area |
|-------------|-----------|
| Business | IT aligns with business outcomes |
| People | Organizational change management |
| Governance | Orchestrating cloud initiatives |
| Platform | Building enterprise-grade cloud platform |
| Security | Achieving confidentiality, integrity, availability |
| Operations | Ensuring cloud services meet business needs |

### Key Limits & Facts

- AWS MGN supports continuous replication (block-level, not snapshot-based)
- MGN supports Windows and Linux source servers
- MGN replication uses lightweight agent on source servers
- Migration Hub supports tracking across multiple AWS tools and partner tools
- Application Discovery Service: Agentless (VMware only, basic info) vs. Agent-based (detailed, any OS)
- Large migrations: AWS recommends wave-based approach (groups of 10-50 servers)
- Typical large enterprise migration: 6-18 months for full portfolio

---

## 2. Design Patterns & Best Practices

### Migration Patterns by Workload Type

| Workload Type | Recommended Strategy | Why |
|---------------|---------------------|-----|
| Legacy monolith (stable, low change) | Rehost | Fast, low risk, optimize later |
| Database with managed equivalent | Replatform | Reduce operational burden, same schema |
| High-growth application | Refactor | Need cloud-native scalability |
| COTS with SaaS equivalent | Repurchase | Vendor manages everything |
| Unused legacy system | Retire | Stop paying for it |
| Recently upgraded ERP | Retain | ROI doesn't justify migration now |
| VMware cluster, tight deadline | Relocate | Preserve existing ops model |

### Wave-Based Migration Pattern

1. **Wave 0:** Build landing zone, networking, security baseline
2. **Wave 1:** Low-risk, simple applications (proving the process)
3. **Wave 2-N:** Increasing complexity, using lessons learned
4. **Final wave:** Complex, mission-critical applications

### Two-Phase Optimization Pattern

1. **Phase 1:** Rehost everything quickly (reduce data center costs, exit lease)
2. **Phase 2:** Optimize on AWS (right-size, refactor where needed, adopt managed services)
- **Why:** Speed of migration is sometimes more important than optimization (e.g., data center lease expiration)

### Strangler Fig Pattern (for Refactoring)

- Gradually replace components of a monolith with microservices
- Route traffic through API Gateway or ALB
- Incrementally migrate functionality until monolith is fully replaced
- Reduces risk compared to big-bang rewrite

### Anti-Patterns

| Anti-Pattern | Why It's Wrong | Better Approach |
|-------------|----------------|-----------------|
| Refactoring everything | Too expensive, too slow, unnecessary for stable workloads | Use 7 Rs assessment — only refactor where business justifies it |
| Rehosting everything permanently | Misses optimization opportunities | Plan Phase 2 optimization |
| Ignoring application dependencies | Creates broken migrations | Use Application Discovery Service |
| Big-bang migration | All-or-nothing risk | Wave-based incremental approach |
| Migrating without a landing zone | Security gaps, no governance | Control Tower / Landing Zone first |
| Skipping the Retire analysis | Wasting effort migrating dead applications | Portfolio assessment first |

### Well-Architected Framework Alignment

- **Reliability:** Replatform to managed services for built-in HA; use multi-AZ after rehosting
- **Security:** Establish security baseline BEFORE migration (landing zone); encrypt in transit during replication
- **Cost Optimization:** Right-size after rehost; retire unused apps; consider Reserved Instances post-stabilization
- **Performance Efficiency:** Refactor bottlenecks; use ElastiCache, read replicas post-migration
- **Operational Excellence:** Automate with MGN; use Migration Hub for tracking; document runbooks
- **Sustainability:** Consolidate servers; use managed services to improve utilization

### Integration Points

| Service | Role in Migration |
|---------|-------------------|
| AWS MGN | Automated lift-and-shift replication |
| AWS DMS | Database migration with minimal downtime |
| AWS SCT | Schema conversion for heterogeneous DB migrations |
| AWS DataSync | File/object data transfer |
| AWS Transfer Family | SFTP/FTPS/FTP endpoint migration |
| AWS Snow Family | Offline bulk data transfer |
| CloudEndure (legacy) | Predecessor to MGN (deprecated) |
| Migration Hub | Single pane of glass for migration tracking |
| Application Discovery Service | Discover on-premises inventory and dependencies |
| Migration Hub Orchestrator | Automate and orchestrate migration workflows |
| Control Tower | Set up secure multi-account landing zone |

---

## 3. Security & Compliance

### Data Security During Migration

- **In-transit encryption:** MGN uses TLS for replication data
- **At-rest encryption:** Target EBS volumes can be encrypted with KMS
- **Network isolation:** Replication traffic can traverse VPN or Direct Connect (private connectivity)
- **DMS encryption:** Supports SSL/TLS for source and target endpoints; KMS for replication instance storage

### IAM & Access Control

- **MGN:** Requires `AWSApplicationMigrationServiceRole` and `AWSApplicationMigrationConversionServerRole`
- **Cross-account migration:** Use IAM roles with cross-account trust policies
- **Least privilege:** Separate roles for replication, launch, cutover
- **Service-linked roles:** MGN creates service-linked roles automatically

### Compliance Considerations

- **Data residency:** Ensure target region meets regulatory requirements BEFORE migration
- **Audit trail:** CloudTrail logs all MGN/DMS API calls
- **Data classification:** Identify sensitive data pre-migration; apply appropriate controls in target
- **PCI/HIPAA/SOC:** Validate that migration process doesn't break compliance chain
- **GDPR:** Cross-region data transfer may require additional safeguards

### Security Best Practices

1. Establish landing zone with security guardrails (Control Tower + SCPs) BEFORE migration
2. Enable AWS Config rules in target accounts from day one
3. Use VPN or Direct Connect for replication — avoid public internet for sensitive workloads
4. Encrypt all EBS volumes in target (can't encrypt existing unencrypted volumes — must copy)
5. Implement Security Hub and GuardDuty in target accounts before cutover
6. Rotate credentials post-migration (embedded passwords, API keys, certificates)
7. Review and tighten security groups post-rehost (don't just replicate on-prem firewall rules)

### Network Security During Migration

- **Replication subnet:** Dedicated subnet for staging/replication servers (isolate from production)
- **Security groups:** Restrict replication server access to source IPs only
- **NACLs:** Consider blocking all inbound except replication ports
- **VPN/DX:** Required for sensitive/regulated workloads
- **PrivateLink:** Use for DMS endpoints when possible

---

## 4. Cost Optimization

### Cost Model by Strategy

| Strategy | Upfront Cost | Ongoing Cost | Break-Even |
|----------|-------------|--------------|------------|
| Rehost | Low (tooling) | Moderate (un-optimized) | 3-6 months |
| Replatform | Low-Medium | Lower (managed services) | 6-12 months |
| Refactor | High (development) | Lowest (optimized) | 12-24 months |
| Repurchase | Medium (licensing) | Variable (subscription) | Depends on contract |
| Retire | None | Immediate savings | Immediate |
| Retain | None | Same as current | N/A |
| Relocate | Low | Similar (VMware licensing) | Quick if avoiding HW refresh |

### AWS MGN Pricing

- **Free for 90 days** per source server (from first replication)
- After 90 days: per-hour charge per replicating server
- **Exam trap:** MGN itself is free for 90 days, but you pay for:
  - Replication instances (lightweight EC2)
  - EBS volumes (staging area)
  - Data transfer
  - Target EC2 instances during testing

### Cost Traps

| Trap | Impact | Mitigation |
|------|--------|------------|
| Over-provisioning after rehost | Paying for on-prem-sized instances | Right-size with Compute Optimizer within 2 weeks |
| Running staging servers too long | Unnecessary EBS/EC2 costs | Clean up staging immediately after cutover |
| Forgetting to terminate source | Paying for both environments | Decommission source after validation period |
| Migrating unused applications | Wasted effort and ongoing costs | Retire analysis first |
| Not using Reserved Instances post-migration | Missing 30-60% savings | Commit after workloads stabilize (1-3 months) |
| Data transfer costs during migration | Can be significant for large datasets | Use Snow Family for >10TB; use DX for ongoing |

### Cost Optimization Strategies

1. **Portfolio rationalization FIRST** — retire 10-20% of apps before spending on migration
2. **Right-size within 2 weeks** of rehost using Compute Optimizer recommendations
3. **Savings Plans / RIs** — purchase after 1-3 months of stable usage patterns
4. **Spot Instances** — for non-production migrated workloads
5. **Use managed services** (replatform) to eliminate patching/admin labor costs
6. **Consolidate licenses** — BYOL where beneficial, or switch to license-included

### Comparison: Migration Cost vs. Staying On-Premises

- **Data center lease:** Often the forcing function (lease expiry = hard deadline)
- **Hardware refresh:** Avoided entirely with cloud migration
- **Staffing:** Managed services reduce ops headcount needs
- **Licensing:** May increase (some licenses don't transfer) or decrease (SaaS model)
- **TCO calculation:** Include people, power, cooling, real estate — not just compute

---

## 5. High Availability, Disaster Recovery & Resilience

### HA Considerations by Strategy

| Strategy | HA After Migration |
|----------|-------------------|
| Rehost | Must explicitly configure (Multi-AZ, ASG, ELB) — not automatic |
| Replatform | Managed services often include HA (RDS Multi-AZ, ElastiCache replication) |
| Refactor | Design for HA from scratch (serverless, multi-region) |
| Repurchase | HA is vendor's responsibility (SaaS) |
| Relocate | VMware HA/DRS within VMware Cloud on AWS |

### Migration Cutover Strategies

| Strategy | Downtime | Risk | Use Case |
|----------|----------|------|----------|
| **Big-bang cutover** | Hours-days | High | Simple apps, acceptable downtime window |
| **Phased cutover** | Minutes per phase | Medium | Complex apps with separable components |
| **Blue-green** | Seconds-minutes | Low | Critical apps, near-zero downtime required |
| **Canary** | None (gradual) | Lowest | High-traffic apps, need to validate at scale |

### Cutover Best Practices

1. **Rehearse cutover** — run full test cutover before production
2. **Rollback plan** — always have ability to switch back to source
3. **DNS TTL** — lower TTL well in advance (24-48 hours before cutover)
4. **Replication lag** — verify zero lag before cutover (MGN continuous replication)
5. **Validation window** — keep source running for defined period post-cutover
6. **Data consistency** — freeze writes or use DMS CDC for databases during cutover

### RPO/RTO by Approach

| Approach | RPO | RTO |
|----------|-----|-----|
| MGN continuous replication | Seconds (near-zero) | Minutes (launch target) |
| DMS with CDC | Seconds | Minutes-hours (depends on validation) |
| Backup & restore (S3/snapshots) | Hours (last backup) | Hours |
| Snow Family (offline) | Days-weeks | Days-weeks |
| Pilot Light post-migration | Minutes | 10-30 minutes |
| Warm Standby post-migration | Seconds | Minutes |
| Multi-site Active-Active | Zero | Zero (automatic failover) |

### Resilience During Migration (Hybrid State)

- **Challenge:** During migration, workloads span on-premises and AWS
- **DNS routing:** Route 53 weighted/failover routing between environments
- **Data synchronization:** DMS CDC or DataSync for ongoing sync during transition
- **Network resilience:** Redundant VPN or DX connections
- **Monitoring:** CloudWatch + existing on-prem monitoring in parallel

### Disaster Recovery Post-Migration

- After rehost: immediately configure:
  - Multi-AZ for databases
  - Auto Scaling Groups for compute
  - Cross-region backups for critical data
  - Route 53 health checks and failover
- Don't assume cloud = HA — you must design for it

---

## 6. Exam-Focused Section

### Straightforward Questions (7 examples)

**Q1:** A company wants to move its on-premises applications to AWS as quickly as possible to exit an expiring data center lease. Applications should remain unchanged. Which migration strategy should they use?
**A:** Rehost (Lift & Shift). Speed is the priority, no code changes needed.

**Q2:** A company is migrating a self-managed PostgreSQL database and wants to reduce operational overhead without changing the application schema. Which strategy is this?
**A:** Replatform. Moving to RDS PostgreSQL is a managed equivalent with no schema change.

**Q3:** During portfolio analysis, a team discovers 15% of applications have zero active users. What should they recommend?
**A:** Retire. Decommission unused applications to save costs immediately.

**Q4:** A company decides to replace its on-premises CRM with Salesforce as part of its cloud migration. Which strategy is this?
**A:** Repurchase (Drop & Shop). Replacing with a SaaS product.

**Q5:** An organization has invested heavily in VMware and wants to migrate to AWS while maintaining their existing VMware tooling and operations. Which strategy is best?
**A:** Relocate. VMware Cloud on AWS preserves VMware operations.

**Q6:** A company wants to break its monolithic application into microservices to improve scalability. Which migration strategy applies?
**A:** Refactor / Re-architect. Fundamental redesign for cloud-native benefits.

**Q7:** A company's legacy ERP was upgraded 6 months ago and has regulatory constraints preventing cloud migration currently. What strategy should be applied?
**A:** Retain. Keep in current environment, revisit later when constraints are resolved.

### Tricky / Scenario-Based Questions (7 examples)

**Q1:** A company needs to migrate 500 servers to AWS within 3 months due to a data center contract expiration. The CTO also wants to optimize costs. Some applications haven't been used in over a year. What combination of strategies should a solutions architect recommend?

**A:** Retire unused applications first, then Rehost the remainder. Optimize (right-size) after migration.
- **Wrong:** Refactor everything — too slow for a 3-month deadline
- **Wrong:** Rehost all 500 — wastes effort migrating dead applications
- **Keywords:** "3 months" (time constraint), "haven't been used" (retire candidates)

**Q2:** A company is migrating a three-tier web application. The web tier is stateless, the app tier has session affinity, and the database is Oracle with PL/SQL stored procedures. Which strategy should be applied to each tier?

**A:** Web tier: Rehost (stateless, easy). App tier: Replatform (move sessions to ElastiCache). Database: Rehost to EC2 with Oracle OR Refactor to Aurora (depends on business justification for rewriting PL/SQL).
- **Wrong:** Replatform Oracle to RDS — PL/SQL stored procedures may not be compatible without SCT conversion
- **Keywords:** "PL/SQL stored procedures" (indicates complexity), "stateless" (easy rehost), "session affinity" (needs managed session store)

**Q3:** A large enterprise is migrating 2,000 applications. They completed discovery using Application Discovery Service and categorized applications into groups. 200 applications are legacy with no active development, 1,500 are standard web apps, and 300 are critical real-time trading systems. How should they approach migration?

**A:** Retire or Retain the 200 legacy apps. Rehost the 1,500 standard apps in waves. Refactor the 300 trading systems for low-latency cloud-native architecture.
- **Wrong:** Rehost the trading systems — "real-time trading" requires performance optimization that lift-and-shift won't provide
- **Wrong:** Refactor the 1,500 standard apps — unnecessary cost and delay
- **Keywords:** "no active development" (retire/retain), "standard web apps" (rehost), "real-time trading" (refactor for performance)

**Q4:** A company wants to migrate their on-premises Microsoft Exchange server to the cloud. They want minimal ongoing maintenance. Which migration strategy should they choose?

**A:** Repurchase — move to Amazon WorkMail or Microsoft 365 (SaaS).
- **Wrong:** Rehost Exchange on EC2 — still requires patching, licensing, maintenance
- **Wrong:** Replatform — no managed AWS equivalent of Exchange
- **Keywords:** "minimal ongoing maintenance" → SaaS (Repurchase)

**Q5:** A company has been using AWS MGN to replicate servers for migration. After 85 days of replication, they realize they need 30 more days for testing before cutover. What cost implication should the solutions architect highlight?

**A:** MGN is free for 90 days per server. After day 90, per-hour charges apply for each replicating server. The additional 25 days beyond the free period will incur MGN charges (plus ongoing EBS and replication instance costs).
- **Wrong:** No additional cost because MGN is free — only free for 90 days
- **Keywords:** "85 days" + "30 more days" = 115 days total (exceeds 90-day free tier)

**Q6:** A company is migrating a legacy application that uses Windows Server 2008 with a SQL Server 2008 database. The application requires minimal changes and must maintain its architecture. Both the OS and database are out of support. What should the solutions architect recommend?

**A:** Rehost to EC2 (maintains architecture as required), but flag that both OS and DB are end-of-life. Consider Replatform the database to RDS SQL Server (newer version) if minor application changes are acceptable.
- **Wrong:** Refactor — requirement says "minimal changes" and "maintain architecture"
- **Wrong:** Retain — keeping unsupported software on-premises doesn't solve the EOL problem
- **Keywords:** "minimal changes" (rehost/replatform), "maintain architecture" (rehost), "out of support" (security risk to address)

**Q7:** During a migration, a solutions architect discovers that an application on-premises communicates with 15 other applications via direct IP addresses hardcoded in configuration files. What should they do before migrating this application?

**A:** Map all dependencies using Application Discovery Service (agent-based). Plan to migrate dependent applications together in the same wave or update configurations to use DNS names / service discovery before cutover.
- **Wrong:** Migrate it independently — will break communication with dependent apps
- **Wrong:** Just update the IP addresses after migration — need DNS/service discovery for resilience
- **Keywords:** "hardcoded IP addresses" (dependency risk), "communicates with 15 other applications" (complex dependencies)

### Common Exam Traps & Pitfalls

| Trap | Reality |
|------|---------|
| "Replatform = code changes" | No — replatform means infrastructure changes with minimal/no code changes |
| "Rehost means no optimization" | Rehost first, optimize after — it's a two-phase approach |
| "Refactor is always the best answer" | Only when business justifies the cost/time; most apps should rehost |
| "Relocate = Rehost" | No — Relocate specifically means VMware Cloud on AWS (infrastructure-level) |
| "Repurchase = buying new AWS services" | No — Repurchase means replacing with a different product (usually SaaS) |
| "MGN is always free" | Free for 90 days only; then per-hour charges |
| "CloudEndure is the migration tool" | CloudEndure is deprecated; AWS MGN replaced it |
| "Retain means the app stays forever" | Retain means "not now" — revisit periodically |
| "All apps should be migrated" | Retire 10-20% of portfolio first |
| "Migration is one strategy per app" | Apps can combine strategies across tiers |

---

## 7. Cheat Sheet

### Must-Know Facts for Exam Day

- **7 Rs:** Rehost, Replatform, Refactor, Repurchase, Retire, Retain, Relocate
- **Rehost:** Fastest, lowest risk, lift-and-shift, use AWS MGN
- **Replatform:** Minor optimizations (e.g., DB → RDS), no code changes
- **Refactor:** Cloud-native redesign, highest cost, best long-term ROI
- **Repurchase:** Switch to SaaS (e.g., Exchange → WorkMail)
- **Retire:** Kill unused apps (typically 10-20% of portfolio)
- **Retain:** Not ready yet, revisit later
- **Relocate:** VMware Cloud on AWS only
- **MGN free tier:** 90 days per source server
- **Application Discovery Service:** Agentless (VMware only) vs. Agent-based (any OS, more detail)
- **Wave-based migration:** Groups of 10-50 servers, increasing complexity
- **Strangler Fig:** Gradual monolith decomposition pattern
- **Landing Zone FIRST:** Control Tower + SCPs before any migration

### Key Differentiators

| Rehost vs. Replatform | Rehost = exact same config on EC2; Replatform = swap component for managed service |
|---|---|
| **Replatform vs. Refactor** | Replatform = same architecture, managed infra; Refactor = new architecture entirely |
| **Repurchase vs. Refactor** | Repurchase = buy someone else's solution; Refactor = rebuild your own |
| **Retain vs. Retire** | Retain = still needed, not ready; Retire = no longer needed at all |
| **Rehost vs. Relocate** | Rehost = per-application to EC2; Relocate = VMware cluster to VMware Cloud on AWS |

### Decision Flowchart

```
Is the application still needed?
├── NO → RETIRE
└── YES → Can it move to cloud now?
    ├── NO → RETAIN
    └── YES → Is there a SaaS replacement that's better?
        ├── YES → REPURCHASE
        └── NO → Does it need cloud-native redesign?
            ├── YES (business justifies) → REFACTOR
            └── NO → Can you swap infra for managed service easily?
                ├── YES → REPLATFORM
                └── NO → Is it VMware-based (bulk move)?
                    ├── YES → RELOCATE
                    └── NO → REHOST
```

### Exam Keyword Triggers

| If the question says... | Think... |
|------------------------|----------|
| "Fastest migration" / "tight deadline" | Rehost |
| "Minimal changes" / "as-is" | Rehost |
| "Reduce operational overhead" (for DB/cache) | Replatform |
| "Cloud-native" / "microservices" / "serverless" | Refactor |
| "Scalability limitations of current architecture" | Refactor |
| "Replace with SaaS" / "third-party solution" | Repurchase |
| "No longer needed" / "zero users" | Retire |
| "Not ready" / "recently upgraded" / "compliance blocks" | Retain |
| "VMware" / "vSphere" / "maintain existing operations" | Relocate |
| "Data center lease expiring" | Rehost (speed priority) + Retire (unused) |
| "Least effort" | Rehost or Retire |
| "Most cost-effective long-term" | Refactor (if justified) or Retire |

---

## 8. Additional Deep-Dive

### 10 More Tricky Scenario-Based Questions

**Q1:** A financial services company is migrating 800 applications. Their compliance team requires that all migrated applications must be encrypted at rest and in transit, and no data can traverse the public internet during migration. The current environment is 60% VMware, 30% physical servers, and 10% containers. What migration approach should the solutions architect recommend?

**A:** Use Direct Connect for private connectivity. For VMware (60%): Relocate to VMware Cloud on AWS OR rehost using MGN with agent. For physical servers (30%): Rehost using MGN (supports physical). For containers (10%): Replatform to ECS/EKS. Enable EBS encryption by default in target accounts. Configure MGN to use private IP for replication (not public).
- **Wrong:** Use VPN only — may not provide sufficient bandwidth for 800 servers
- **Wrong:** Relocate everything — only works for VMware; physical servers and containers need different approaches
- **Keywords:** "no data can traverse public internet" (Direct Connect + private replication), "60% VMware" (Relocate candidate), "physical servers" (MGN agent)

**Q2:** A retail company has a peak-season e-commerce platform running on-premises. They plan to migrate, but the next peak season is in 4 months. They cannot risk any performance degradation during peak. The application is a Java monolith with Oracle database. What should the solutions architect recommend?

**A:** Rehost to EC2 immediately (within 2 months). Run both environments in parallel during peak as a safety net. After peak season, replatform the Oracle DB to Aurora PostgreSQL using DMS+SCT. Eventually refactor the monolith to microservices.
- **Wrong:** Refactor before peak — 4 months is not enough time to redesign a monolith
- **Wrong:** Wait until after peak to start — loses migration momentum and time
- **Wrong:** Replatform Oracle immediately — schema conversion risk too close to peak season
- **Keywords:** "4 months" (tight timeline), "cannot risk performance degradation" (conservative approach), "Java monolith with Oracle" (complex, multi-phase)

**Q3:** A healthcare company must migrate patient records to AWS. Regulations require that data never leaves the country, and the migration must maintain an audit trail showing chain of custody for all data movement. They currently use a mix of SQL Server databases and file shares. What approach minimizes compliance risk?

**A:** Choose an AWS region within the country. Replatform SQL Server to RDS SQL Server (maintains schema, adds managed HA). Use DataSync for file shares → S3/EFS with verification enabled. Enable CloudTrail for API-level audit. Use AWS Artifact to document compliance. DMS with validation to verify row counts and data integrity.
- **Wrong:** Rehost SQL Server to EC2 — misses opportunity to reduce operational overhead while maintaining compliance
- **Wrong:** Use Snow Family — doesn't provide real-time audit trail of data movement (offline process)
- **Keywords:** "data never leaves the country" (region selection), "audit trail" (CloudTrail + DMS validation), "chain of custody" (logging every data movement)

**Q4:** A media company is migrating 50TB of video assets plus their transcoding pipeline. The video assets are stored on NAS, and the transcoding pipeline runs on a dedicated GPU cluster. They need to complete migration within 6 weeks but have only 100 Mbps internet connectivity. What should the solutions architect recommend?

**A:** Use Snowball Edge for the 50TB of video assets (100 Mbps would take ~46 days for 50TB — too slow). Replatform the transcoding pipeline to AWS Elemental MediaConvert (or rehost to GPU EC2 instances). Use DataSync for incremental changes after Snowball delivery.
- **Wrong:** Transfer over the internet — 50TB at 100 Mbps ≈ 46 days (exceeds 6-week window accounting for overhead)
- **Wrong:** Rehost NAS to EC2 — NAS should become S3 or EFS (replatform)
- **Keywords:** "50TB" + "100 Mbps" + "6 weeks" (bandwidth math points to Snow Family), "GPU cluster" + "transcoding" (MediaConvert = replatform)

**Q5:** A company's migration team discovers that their SAP system has dependencies on 40 other applications, many of which are not scheduled for migration. The SAP system must continue communicating with on-premises applications after it moves to AWS. What should the solutions architect recommend?

**A:** Rehost SAP to AWS using SAP-certified EC2 instances. Maintain hybrid connectivity via Direct Connect (primary) + VPN (backup). Use DNS-based service discovery instead of hardcoded IPs. Keep dependent applications communicating via the DX/VPN link until they are also migrated. Consider AWS Backint Agent for SAP HANA backups.
- **Wrong:** Wait until all 40 apps migrate first — could delay SAP migration indefinitely
- **Wrong:** Refactor SAP — SAP is not typically refactored; certified instance types exist for rehosting
- **Keywords:** "dependencies on 40 other applications" (hybrid connectivity needed), "SAP" (specialized migration, certified instances), "not scheduled for migration" (long-term hybrid state)

**Q6:** A startup acquired a competitor and needs to consolidate their AWS environments. The competitor has 200 EC2 instances across 5 accounts with no tagging strategy, unknown dependencies, and suspected unused resources. What should the solutions architect recommend FIRST?

**A:** Use Application Discovery Service (agent-based) to map dependencies and identify usage patterns. Use AWS Migration Hub to create a unified inventory. Identify retire candidates (unused instances). Then plan consolidation waves based on dependency groups. This is a cloud-to-cloud migration — MGN supports AWS-to-AWS.
- **Wrong:** Just move everything into the acquirer's accounts — unknown dependencies will break things
- **Wrong:** Start migrating immediately — discovery must come first
- **Keywords:** "unknown dependencies" (discovery first), "suspected unused resources" (retire candidates), "consolidate" (multi-account to fewer accounts)

**Q7:** A government agency needs to migrate a classified workload. The data is rated at Impact Level 5 (IL5). They require dedicated infrastructure with no multi-tenancy. Current infrastructure is physical (not virtualized). What AWS strategy and services are appropriate?

**A:** Rehost to AWS GovCloud (US) on dedicated hosts (no multi-tenancy). Use MGN agent on physical servers for migration. Dedicated Hosts provide single-tenant hardware. Consider Nitro Enclaves for additional isolation. All storage on encrypted dedicated EBS volumes.
- **Wrong:** Standard AWS commercial region — doesn't meet IL5 requirements
- **Wrong:** Relocate — not VMware-based (physical servers)
- **Wrong:** Outposts — question implies moving TO cloud, not keeping on-premises
- **Keywords:** "classified" + "IL5" (GovCloud), "no multi-tenancy" (Dedicated Hosts), "physical" (MGN agent, not agentless)

**Q8:** A company is migrating a real-time bidding platform that processes 500,000 requests per second with <10ms latency requirements. The current architecture uses custom-built in-memory caching and event processing. What strategy should be used?

**A:** Refactor. The latency and throughput requirements demand cloud-native optimization. Use ElastiCache for Redis (in-memory caching), Kinesis Data Streams or MSK (event processing), and EC2 with enhanced networking (placement groups, ENA) or Lambda@Edge for edge processing. Cannot simply rehost — on-premises optimizations won't translate to cloud networking.
- **Wrong:** Rehost — network latency characteristics differ between on-prem and cloud; custom optimizations won't work as-is
- **Wrong:** Replatform — "minor changes" won't achieve <10ms at 500K RPS in cloud
- **Keywords:** "500,000 requests per second" + "<10ms latency" (performance-critical = refactor), "custom-built" (needs redesign for cloud)

**Q9:** A multinational company has identical application stacks in 8 countries, each running independently in local data centers. They want to consolidate into AWS while maintaining data residency requirements. What should they recommend?

**A:** Rehost each country's stack into the nearest AWS region (maintaining data residency). Use a global architecture pattern: single codebase deployed per-region, with Route 53 geolocation routing. DynamoDB Global Tables or Aurora Global Database for read performance (if regulations allow read replicas cross-region). Control Tower with per-region OUs for governance.
- **Wrong:** Consolidate all into one region — violates data residency requirements
- **Wrong:** Refactor into a single global application — data residency laws prevent centralization
- **Keywords:** "8 countries" + "data residency" (multi-region mandatory), "identical stacks" (repeatable rehost pattern), "consolidate" (single management plane, not single region)

**Q10:** A company completed rehosting 300 servers two months ago. The CFO reports that AWS costs are 40% higher than the previous on-premises infrastructure costs. The solutions architect needs to reduce costs without application changes. What should they recommend?

**A:** This is a post-migration optimization problem. Steps: (1) Right-size using Compute Optimizer (most EC2 instances are over-provisioned after lift-and-shift). (2) Purchase Savings Plans for stable workloads. (3) Use Spot Instances for fault-tolerant workloads. (4) Schedule start/stop for non-production instances. (5) Delete unattached EBS volumes and unused EIPs. (6) Enable S3 Intelligent-Tiering for migrated data.
- **Wrong:** Refactor applications — question says "without application changes"
- **Wrong:** Move back on-premises — not a valid exam answer
- **Keywords:** "40% higher" (typical post-rehost before optimization), "without application changes" (infrastructure-level optimization only), "two months ago" (enough data for right-sizing recommendations)

### Compare: AWS MGN vs. DMS vs. DataSync

| Aspect | AWS MGN | AWS DMS | AWS DataSync |
|--------|---------|---------|--------------|
| **Use Case** | Server migration (OS + apps) | Database migration | File/object data transfer |
| **What it moves** | Entire server (block-level) | Database contents + schema | Files, objects, NFS/SMB shares |
| **Replication type** | Continuous block-level | CDC (change data capture) | Scheduled or one-time transfer |
| **Source** | Physical, virtual, or cloud servers | Any supported database | NFS, SMB, HDFS, S3, self-managed storage |
| **Target** | EC2 instances | RDS, Aurora, DynamoDB, S3, Redshift | S3, EFS, FSx |
| **Downtime** | Minutes (at cutover only) | Minutes (with CDC) | Depends on data volume |
| **Agent required** | Yes (on source server) | No (connects to DB endpoint) | Yes (on-premises agent for on-prem sources) |
| **Schema conversion** | N/A | Use SCT for heterogeneous | N/A |
| **Free tier** | 90 days per server | Free for 6 months (t2/t3.micro) | No free tier |
| **Exam tip** | "Migrate servers" | "Migrate databases" | "Migrate file shares/NFS/SMB" |

### When Does the SAP-C02 Exam Expect Me to Pick One Strategy Over Another?

**Pick REHOST when:**
- "Tight deadline" / "data center lease expiring" / "fastest migration"
- "No application changes" / "maintain current architecture"
- "Large-scale migration" (hundreds/thousands of servers)
- "Optimize after migration" / "two-phase approach"

**Pick REPLATFORM when:**
- "Reduce operational overhead" + database/cache workload
- "Move to managed service" without schema changes
- "Self-managed [X] to Amazon [X]" (e.g., MySQL → RDS MySQL)
- "Minor optimizations" / "few cloud optimizations"

**Pick REFACTOR when:**
- "Cloud-native" / "microservices" / "serverless" / "event-driven"
- "Current architecture limits scalability"
- "Long-term cost optimization" (with budget and time available)
- "Performance requirements that current architecture can't meet"
- Application needs to be "fundamentally redesigned"

**Pick REPURCHASE when:**
- "Replace with SaaS" / "third-party solution"
- "Minimal maintenance" for commodity applications (email, CRM, HR)
- "Commercial off-the-shelf" replacement available

**Pick RETIRE when:**
- "No active users" / "unused" / "duplicate functionality"
- "Portfolio rationalization" identifies unnecessary applications
- "Reduce costs immediately" without migration effort

**Pick RETAIN when:**
- "Recently upgraded" / "just purchased" / "new hardware"
- "Compliance prevents migration" currently
- "Unresolved dependencies" / "not ready"
- "Low priority" / "revisit later"

**Pick RELOCATE when:**
- "VMware" / "vSphere" / "vCenter" / "vSAN"
- "Maintain existing operations" / "preserve VMware tooling"
- "Data center evacuation" + VMware environment
- "No operational change" during migration

### All "Gotcha" Differences Between Migration Strategies

| Gotcha | Explanation |
|--------|-------------|
| Replatform ≠ code changes | Replatform changes infrastructure (e.g., self-managed DB → RDS), NOT application code |
| Refactor ≠ always best | Most expensive, longest timeline — only when business case is strong |
| Relocate ≠ Rehost | Relocate is VMware-specific bulk infrastructure move; Rehost is per-application to EC2 |
| Repurchase ≠ buying AWS services | Repurchase means replacing with a DIFFERENT product (usually SaaS from third party) |
| Retain ≠ Ignore | Retain is a conscious decision with a planned revisit date — not "we forgot about it" |
| Retire ≠ Retain | Retire = decommission permanently; Retain = keep running, migrate later |
| MGN ≠ CloudEndure | CloudEndure deprecated; MGN is the current tool (same underlying tech) |
| "Lift and shift then optimize" = valid | Two-phase: Rehost now (speed), optimize later (cost). This IS a recommended pattern |
| Multi-tier apps use multiple strategies | Web tier might rehost, DB tier might replatform, frontend might repurchase |

### Decision Tree for Choosing the Right Service

```
What are you migrating?
│
├── ENTIRE SERVERS (OS + applications + data)
│   └── AWS MGN (Application Migration Service)
│       ├── Physical servers → Agent-based
│       ├── Virtual servers → Agent-based  
│       └── Cloud servers → Agent-based (cloud-to-cloud)
│
├── DATABASES ONLY
│   └── AWS DMS (+ SCT if heterogeneous)
│       ├── Same engine (MySQL → RDS MySQL) → DMS only
│       ├── Different engine (Oracle → Aurora PostgreSQL) → SCT + DMS
│       └── Need ongoing replication → DMS with CDC
│
├── FILE SHARES / OBJECT STORAGE
│   └── AWS DataSync
│       ├── NFS/SMB → S3, EFS, or FSx
│       ├── HDFS → S3
│       └── S3 → S3 (cross-account/cross-region)
│
├── BULK DATA (offline, >10TB, limited bandwidth)
│   └── AWS Snow Family
│       ├── Up to 8TB → Snowcone
│       ├── Up to 80TB per device → Snowball Edge
│       └── Up to 100PB → Snowmobile
│
├── VMware ENVIRONMENT (bulk)
│   └── VMware Cloud on AWS (Relocate strategy)
│
└── DISCOVERY / PLANNING
    └── Application Discovery Service + Migration Hub
        ├── VMware only, basic info → Agentless
        └── Any OS, detailed dependencies → Agent-based
```
