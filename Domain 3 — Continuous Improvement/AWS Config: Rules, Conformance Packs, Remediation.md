# AWS Config: Rules, Conformance Packs, Remediation

## 1. Core Concepts & Theory

### AWS Config Overview
- **Continuous monitoring and recording service** for AWS resource configurations
- Records **configuration changes** over time (who changed what, when)
- Evaluates configurations against **rules** (compliance/non-compliance)
- **Regional service** — must be enabled per region (aggregation available cross-region/account)
- **Configuration Item (CI)**: Point-in-time snapshot of a resource's configuration
- **Configuration Recorder**: Records configuration changes (must be enabled)
- **Delivery Channel**: Delivers CIs and compliance data to S3 (and optionally SNS)

### Configuration Recording

**What's Recorded:**
- Resource properties, relationships, IAM policies, security groups, VPC config, etc.
- Changes are recorded when resources are created, modified, or deleted
- Relationships between resources (e.g., EC2 instance → Security Group → VPC)

**Recording Modes:**
| Mode | Behavior |
|------|----------|
| **Continuous** | Records every change as it happens (default) |
| **Periodic** | Records every 24 hours (cost savings) |
| **Daily** | Same as periodic (once per day) |

**Recording Scope:**
- All supported resources (recommended for full visibility)
- Specific resource types (targeted recording for cost control)
- Resources with specific tags
- **Excluding resource types**: Can exclude high-change resources to reduce noise/cost

**Configuration History:**
- S3 bucket stores configuration history (JSON)
- Retention: Configurable (30 days to 7 years, default 7 years)
- Query with: AWS Config console, Advanced Queries (SQL-like), or Athena on S3 data

---

### Config Rules

**Overview:**
- **Evaluate resource configurations** against desired settings
- Return: COMPLIANT, NON_COMPLIANT, NOT_APPLICABLE, or INSUFFICIENT_DATA
- Two types: **AWS Managed Rules** (~300+) and **Custom Rules**

**AWS Managed Rules (examples):**
| Rule | Checks |
|------|--------|
| `s3-bucket-public-read-prohibited` | S3 buckets not publicly readable |
| `ec2-instance-no-public-ip` | EC2 instances don't have public IPs |
| `rds-instance-public-access-check` | RDS not publicly accessible |
| `encrypted-volumes` | EBS volumes are encrypted |
| `iam-root-access-key-check` | Root account has no access keys |
| `vpc-flow-logs-enabled` | VPC Flow Logs are enabled |
| `cloudtrail-enabled` | CloudTrail is enabled |
| `required-tags` | Resources have required tags |
| `ec2-managedinstance-patch-compliance-status-check` | SSM patch compliance |
| `restricted-ssh` | Security groups don't allow SSH from 0.0.0.0/0 |

**Evaluation Triggers:**
| Trigger Type | When Evaluated |
|-------------|---------------|
| **Configuration change** | When a resource matching the rule scope is created/changed/deleted |
| **Periodic** | Every 1h, 3h, 6h, 12h, or 24h |
| **Both** | On change AND periodically (belt and suspenders) |

**Custom Rules:**

| Type | How It Works | Best For |
|------|-------------|----------|
| **AWS Lambda** | Custom Lambda function evaluates resource (full flexibility) | Complex/custom logic not covered by managed rules |
| **CloudFormation Guard** | Policy-as-code in Guard DSL (declarative) | Template-style validation without Lambda |

**Custom Rule (Lambda):**
- Lambda receives configuration item (CI) from Config
- Evaluates and returns COMPLIANT or NON_COMPLIANT
- Function triggered on config change or periodic schedule
- Must return evaluation result to Config via `PutEvaluations` API

**Rule Scope:**
- All resources of a type (e.g., all EC2 instances)
- Resources with specific tags
- Specific resource IDs
- All changes (configuration change trigger)

**Key Limits:**
| Limit | Value |
|-------|-------|
| Rules per region | 400 (soft, can increase) |
| Conformance packs per account per region | 50 |
| Config rules evaluations per month | Included in pricing (per evaluation) |
| Active Config rules evaluated per resource change | 250 |
| Max evaluation results per `PutEvaluations` call | 100 |

---

### Conformance Packs

**Overview:**
- **Collection of Config rules and remediation actions** packaged as a single entity
- Deployable across an organization (via Organizations integration)
- **Compliance score**: Percentage of rules in the pack that are compliant
- Defined in **YAML templates** (similar to CloudFormation)

**Use Cases:**
- Compliance frameworks: CIS Benchmarks, PCI-DSS, HIPAA, SOC 2, NIST
- Organization-wide standards: Tagging, encryption, logging
- Custom organizational policies

**Deployment:**
| Scope | How |
|-------|-----|
| Single account | Deploy via console/CLI |
| Organization-wide | Deploy via Organizations integration (delegated admin) |
| Specific OUs | Target OUs when deploying organization conformance packs |

**AWS-Authored Conformance Packs:**
- `Operational-Best-Practices-for-CIS-AWS-v1.4-Level1`
- `Operational-Best-Practices-for-PCI-DSS`
- `Operational-Best-Practices-for-HIPAA-Security`
- `Operational-Best-Practices-for-NIST-800-53-rev5`
- `Operational-Best-Practices-for-Amazon-S3`
- Many more pre-built templates

**Conformance Pack Template Structure:**
```yaml
Parameters:
  S3BucketName:
    Type: String
Resources:
  S3BucketPublicReadProhibited:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: s3-bucket-public-read-prohibited
      Source:
        Owner: AWS
        SourceIdentifier: S3_BUCKET_PUBLIC_READ_PROHIBITED
  RemediationForS3PublicRead:
    Type: AWS::Config::RemediationConfiguration
    Properties:
      ConfigRuleName: s3-bucket-public-read-prohibited
      TargetType: SSM_DOCUMENT
      TargetId: AWS-DisableS3BucketPublicReadWrite
      Automatic: true
```

---

### Remediation

**Overview:**
- **Automatically fix non-compliant resources** using SSM Automation documents
- Triggered when a rule evaluates a resource as NON_COMPLIANT
- Two modes: **Automatic** or **Manual** (manual = one-click but requires human trigger)

**Remediation Actions:**
- Backed by **SSM Automation documents** (runbooks)
- AWS pre-built: `AWS-DisableS3BucketPublicReadWrite`, `AWS-EnableS3BucketEncryption`, etc.
- Custom: Write your own SSM Automation document

**Auto-Remediation:**
- Set `Automatic: true` in remediation configuration
- Remediation executes immediately when non-compliance is detected
- **Retry**: Configurable max retries (up to 5) with retry interval
- **Concurrency**: Limit concurrent remediation executions
- **Resource parameters**: Pass the non-compliant resource ID to the remediation document

**Remediation Execution Role:**
- IAM role assumed by Config to execute remediation
- Must have permissions for the actions in the SSM document
- Principle of least privilege: Only permissions needed for that specific fix

**Common Pre-Built Remediation Documents:**
| Document | Action |
|----------|--------|
| `AWS-DisableS3BucketPublicReadWrite` | Disable public access on S3 bucket |
| `AWS-EnableS3BucketEncryption` | Enable default S3 encryption |
| `AWS-ConfigureS3BucketLogging` | Enable S3 access logging |
| `AWS-EnableCloudTrailCloudWatchLogging` | Enable CloudTrail → CloudWatch Logs |
| `AWS-EnableEbsEncryptionByDefault` | Enable account-level EBS encryption |
| `AWSConfigRemediation-EnableSecurityHub` | Enable Security Hub |

---

### Config Aggregator

**Overview:**
- **Cross-account and cross-region aggregation** of Config data
- Central view of compliance across entire organization
- Two authorization methods:
  - **Organization aggregator**: Automatic via AWS Organizations (no manual authorization per account)
  - **Individual accounts**: Each account must authorize the aggregator

**Aggregator Features:**
- View compliance of all accounts from one central dashboard
- Query across all accounts/regions using Advanced Queries
- Does NOT trigger remediation in source accounts (view-only aggregation)
- Aggregates: Configuration data, compliance data, conformance pack compliance

---

### Advanced Queries

- **SQL-like query language** for Config data
- Query current resource configurations across all recorded resources
- Examples:
```sql
SELECT resourceId, resourceType, configuration
WHERE resourceType = 'AWS::EC2::SecurityGroup'
AND configuration.ipPermissions.ipRanges = '0.0.0.0/0'

SELECT resourceId, accountId
WHERE resourceType = 'AWS::S3::Bucket'
AND configuration.status = 'NON_COMPLIANT'
```
- Supports cross-account queries (via aggregator)
- Returns results in JSON
- Useful for: Security audits, inventory queries, compliance reports

---

## 2. Design Patterns & Best Practices

### When to Use

| Feature | Use When | Don't Use When |
|---------|----------|----------------|
| Config Rules | Continuous compliance evaluation, audit | Preventing resource creation (use SCP or IAM) |
| Conformance Packs | Organization-wide compliance frameworks | Single rule checks |
| Auto-Remediation | Fix known non-compliance patterns automatically | Complex fixes needing human judgment |
| Aggregator | Multi-account/region visibility | Single account monitoring |
| Advanced Queries | Ad-hoc resource inventory and compliance queries | Real-time alerting (use EventBridge) |

### Anti-Patterns
- **Don't use Config as a preventive control** — Config is DETECTIVE (evaluates after change). Use SCPs, IAM, or CloudFormation Hooks for prevention.
- **Don't enable recording for all resource types if cost-sensitive** — record only what you need to evaluate
- **Don't auto-remediate without testing** — test remediation in dev first; auto-fix can cause outages
- **Don't use Config for real-time alerting** — evaluation has latency; use EventBridge + CloudTrail for instant alerts
- **Don't create duplicate rules** — one rule per check; use conformance packs to organize

### Architectural Patterns

**Organization-Wide Compliance:**
```
Management Account:
  └── Config Aggregator (Organization-wide)
       ├── Views compliance across all accounts
       └── Advanced Queries for cross-account reporting

Delegated Admin Account (Security/Audit):
  └── Organization Conformance Packs (CIS, PCI, custom)
       ├── Deployed to all member accounts
       └── Auto-remediation on critical findings

Member Accounts:
  └── Config Recorder + Rules (deployed via conformance pack)
       ├── Continuous evaluation
       ├── Remediation executes locally
       └── Results aggregated to central account
```

**Compliance Pipeline:**
```
Resource Change → Config Records CI → Rule Evaluates → NON_COMPLIANT
  → Auto-Remediation (SSM Automation) → Resource Fixed → Re-evaluation → COMPLIANT
  → If remediation fails → EventBridge event → SNS → Security team notified
```

**Multi-Layer Compliance:**
```
Preventive:    SCPs (deny non-compliant actions) + IAM (least privilege)
Detective:     Config Rules (evaluate compliance) + GuardDuty (threat detection)
Corrective:    Config Remediation (auto-fix) + Lambda (custom fix)
Audit:         Config History (who changed what) + CloudTrail (API audit)
```

**Config + Security Hub Integration:**
```
Config Rules → Findings → Security Hub (centralized security view)
Config compliance data feeds into Security Hub security standards
Security Hub can use Config rules as part of its standards (CIS, PCI)
```

### Well-Architected Alignment

| Pillar | Practice |
|--------|----------|
| Reliability | Detect configuration drift from known-good state |
| Security | Enforce encryption, public access blocks, logging enablement |
| Cost | Detect unused/idle resources (via custom rules), enforce tagging |
| Performance | Detect over/under-provisioned resources (combine with Compute Optimizer) |
| Ops Excellence | Organization-wide conformance packs, aggregated compliance dashboards |

### Integration Points
- **Security Hub**: Config findings feed into Security Hub standards
- **SSM Automation**: Remediation actions
- **Organizations**: Organization aggregator, organization conformance packs
- **CloudTrail**: Who made the change (Config shows what changed)
- **S3**: Configuration history storage, compliance snapshots
- **SNS**: Notifications for compliance changes
- **EventBridge**: Config rule compliance change events
- **Lambda**: Custom rule evaluation logic
- **CloudFormation**: Deploy Config rules and conformance packs
- **RAM**: Share custom rules across accounts

---

## 3. Security & Compliance

### IAM & Access Control

**Config Service Role:**
- `AWS-ConfigRole` or custom role
- Needs: Config recording permissions (describe/list for all recorded resource types), S3 delivery, SNS publishing
- If using remediation: Needs to pass role to SSM Automation

**Rule Evaluation Permissions (Custom Lambda Rules):**
- Lambda execution role needs `config:PutEvaluations`
- Needs read access to resources being evaluated
- Config invokes Lambda (Lambda resource policy must allow `config.amazonaws.com`)

**Remediation Role:**
- Separate IAM role for remediation execution
- Only permissions needed for the specific fix
- Config assumes this role when executing remediation

**Cross-Account:**
- Aggregator: Source accounts authorize the aggregator account (or use Organization aggregator)
- Conformance packs (org): Deployed via delegated admin (no per-account authorization needed)

### Encryption
- **Configuration history in S3**: Encrypted with SSE-S3 (default) or SSE-KMS
- **Delivery channel**: S3 bucket for configuration snapshots
- **API calls**: All over TLS

### Auditing
- **CloudTrail**: All Config API calls logged (PutConfigRule, PutRemediationConfigurations, etc.)
- **Config itself**: Records changes to Config rules as configuration items
- **Compliance timeline**: View compliance history per resource over time

### Compliance Patterns

**Preventive + Detective Framework:**
| Layer | Tool | Example |
|-------|------|---------|
| Preventive | SCP | Deny `s3:PutBucketPolicy` with public access |
| Preventive | CloudFormation Hook | Reject templates without encryption |
| Detective | Config Rule | `s3-bucket-public-read-prohibited` |
| Corrective | Config Remediation | `AWS-DisableS3BucketPublicReadWrite` |
| Audit | Config History | Track who/when made bucket public |

---

## 4. Cost Optimization

### Pricing

| Item | Price |
|------|-------|
| Configuration items recorded | $0.003 per CI |
| Config rule evaluations | $0.001 per evaluation (first 100K), decreasing tiers |
| Conformance pack evaluations | $0.001 per evaluation (within pack rules) |
| Custom rule evaluations (Lambda) | Same + Lambda execution cost |
| Advanced queries | Free (no additional charge) |
| Config Aggregator | Free (no additional charge) |
| S3 storage | Standard S3 rates for config history |

### Cost-Saving Strategies
1. **Record only needed resource types** — don't record all 300+ types if you only evaluate 20
2. **Use periodic recording** for resources that rarely change (reduces CI count)
3. **Managed rules over custom Lambda rules** — Lambda has execution cost on top
4. **Consolidate rules** — avoid duplicate rules evaluating the same thing
5. **Use conformance packs** — organized rules, no additional cost beyond evaluations
6. **Exclude high-change resources** — resources that change constantly generate many CIs
7. **Retention policy**: Reduce S3 retention if long history isn't needed

### Cost Traps
- **Recording ALL resource types** in a large account generates millions of CIs ($0.003 each)
- **High-frequency periodic rules** (1-hour evaluation) on thousands of resources = many evaluations
- **Custom Lambda rules** with long execution times (Lambda billing on top)
- **Overly broad change-triggered rules** — evaluates on every change to the resource type
- **Not setting S3 lifecycle** for configuration history — data accumulates indefinitely

---

## 5. High Availability, Disaster Recovery & Resilience

### Built-in HA
- Config is a **fully managed regional service** — multi-AZ within a region
- Configuration data stored durably in S3
- Aggregator provides cross-region visibility (but data stays in source region)

### DR Considerations
- **Config data is regional** — not automatically replicated cross-region
- **S3 cross-region replication**: Replicate configuration history bucket for DR
- **CloudFormation**: Define Config rules and conformance packs as code (redeploy in DR region)
- **Organization conformance packs**: Auto-deploy to all regions (simplifies multi-region)

### Resilience
- If Config service has an issue, resources continue operating (Config is control plane)
- Remediation failures don't affect resources (they stay non-compliant, get retried)
- Aggregator is eventually consistent (slight delay in cross-account/region data)

---

## 6. Exam-Focused Section

### Straightforward Questions (7 examples)

**Q1:** A company wants to ensure all S3 buckets have encryption enabled. If a bucket is found without encryption, it should be automatically encrypted. What service achieves this?
**A:** **AWS Config** with the managed rule `s3-bucket-server-side-encryption-enabled` and **auto-remediation** using `AWS-EnableS3BucketEncryption` SSM document.

**Q2:** A company needs to verify that all EC2 instances across 50 accounts are compliant with security benchmarks. Where can they view consolidated compliance?
**A:** **Config Aggregator** (organization aggregator) — provides a centralized dashboard showing compliance across all accounts and regions.

**Q3:** What is the difference between AWS Config and AWS CloudTrail?
**A:** **Config**: Records WHAT the configuration IS (current and historical state). **CloudTrail**: Records WHO did WHAT (API calls). Config shows resource state; CloudTrail shows actions taken.

**Q4:** A company needs to deploy a set of 20 compliance rules (CIS Benchmark) across all accounts in their organization. What's the best approach?
**A:** **Organization Conformance Pack** — deploy a CIS-aligned conformance pack from the delegated admin account to all member accounts/OUs automatically.

**Q5:** How can a company find all security groups that allow SSH from 0.0.0.0/0 across all their accounts?
**A:** Use Config **Advanced Queries** with the aggregator:
```sql
SELECT resourceId, accountId WHERE resourceType = 'AWS::EC2::SecurityGroup' AND relationships.resourceType = 'AWS::EC2::NetworkInterface'
```
Or use the managed rule `restricted-ssh` and view non-compliant resources.

**Q6:** A company's Config rule detects a non-compliant resource, but the auto-remediation fails with "Access Denied." What's likely wrong?
**A:** The **remediation execution IAM role** doesn't have the necessary permissions to perform the remediation action. The role must have permissions matching what the SSM Automation document needs.

**Q7:** Can AWS Config prevent a non-compliant resource from being created?
**A:** **No** — Config is DETECTIVE, not preventive. It evaluates AFTER creation/change. For prevention, use SCPs, IAM policies, or CloudFormation Hooks.

---

### Tricky / Scenario-Based Questions (7 examples)

**Q1:** A company enables the Config rule `restricted-ssh` (security groups should not allow SSH from 0.0.0.0/0). A developer creates a security group with SSH open to 0.0.0.0/0. The rule shows NON_COMPLIANT but the security group still exists and is usable. The CTO asks why Config didn't prevent this. What should the architect explain?

**A:** Config is a **detective control**, not a preventive control. It evaluates AFTER the resource is created/modified and reports compliance status. To PREVENT the security group from being created, use: (1) **SCP** denying `ec2:AuthorizeSecurityGroupIngress` with condition on CIDR 0.0.0.0/0. (2) **IAM policy** with deny + condition. (3) **CloudFormation Hook** rejecting templates with public SSH.

The architect should implement Config rule (detection) + auto-remediation (correction) + SCP (prevention) for defense-in-depth.

**Keywords:** "Config didn't prevent", "still exists" → Config is detective, not preventive (use SCP for prevention)

---

**Q2:** A company uses Config auto-remediation to remove public access from S3 buckets. A legitimate bucket that MUST be public (static website hosting) keeps getting its public access removed. How to fix this while maintaining the compliance check?

**A:** **Exclude the specific bucket** from the remediation scope. Options: (1) Add a tag to the bucket (e.g., `AllowPublicAccess=true`) and modify the remediation to check for this tag before executing. (2) Use a **custom Lambda remediation** that skips buckets with the exception tag. (3) Modify the Config rule scope to exclude that specific resource. (4) Use separate rules: one for general buckets (with remediation) and one monitoring-only rule for the public bucket.

**Why wrong answers are wrong:**
- "Disable the rule entirely" — loses compliance visibility for all other buckets
- "Remove auto-remediation" — other buckets won't be auto-fixed
- "Move the public bucket to a different account" — overcomplicates; exception handling is simpler
- **Keywords:** "legitimate exception to auto-remediation" → Exception tag + conditional remediation logic

---

**Q3:** A company deploys an Organization Conformance Pack. One member account shows 0% compliance while all others are 95%+. The member account has Config enabled and recording. What's likely wrong?

**A:** The member account likely has **no resources of the types being evaluated** (showing NOT_APPLICABLE or no evaluations), OR the Config rules in the conformance pack haven't finished initial evaluation (can take time). More likely: The member account's Config recorder is recording only specific resource types, and **not recording the types needed by the conformance pack rules**. Fix: Ensure the Config recorder records all resource types evaluated by the conformance pack.

**Why wrong answers are wrong:**
- "IAM permissions are wrong" — conformance packs deploy via Organizations, permissions are automatic
- "Account isn't in the Organization" — wouldn't receive the conformance pack at all
- "Config is disabled" — question states Config is enabled and recording
- **Keywords:** "one account 0% compliance", "Config enabled" → Recorder not recording the needed resource types

---

**Q4:** A company uses Config to detect unencrypted EBS volumes. When a non-compliant volume is found, remediation should: (1) create a snapshot, (2) create an encrypted copy, (3) replace the original volume. This multi-step process fails with the standard remediation. Why?

**A:** Standard Config remediation uses **single SSM Automation documents**. The multi-step process (snapshot → encrypt → swap) requires a **custom SSM Automation document** with multiple steps (or an existing multi-step document like `AWSConfigRemediation-EncryptEBSVolume`). The standard single-step documents can't handle complex multi-step workflows.

**Why wrong answers are wrong:**
- "Config can't remediate EBS volumes" — it can, with the right automation document
- "Use Lambda instead" — Lambda is for rule evaluation, not remediation execution
- "Remediation timeout is too short" — might be a factor but the main issue is the multi-step requirement
- **Keywords:** "multi-step remediation", "fails with standard" → Custom SSM Automation document (or multi-step AWS document)

---

**Q5:** A company has 200 AWS accounts. They want to detect any account where CloudTrail is disabled (non-compliant). They use Config Aggregator but notice a 2-hour delay between when CloudTrail is disabled and when it appears as non-compliant in the aggregator. Is this expected?

**A:** **Yes, this is expected behavior.** Config evaluates on configuration change (detected within minutes in the source account), but the aggregator synchronizes data periodically (not real-time). For faster detection: (1) Set up an **EventBridge rule** on CloudTrail API call `StopLogging` for immediate alert. (2) Use a **Config rule with change trigger** in each account + SNS notification. The aggregator view is for compliance reporting, not real-time alerting.

**Why wrong answers are wrong:**
- "The rule should be periodic, not change-triggered" — change-triggered is faster, not slower
- "The aggregator is broken" — this is normal aggregation latency
- "Use GuardDuty instead" — GuardDuty detects threats, not configuration compliance
- **Keywords:** "delay in aggregator", "real-time detection needed" → Expected behavior; use EventBridge for real-time

---

**Q6:** A company creates a custom Config rule using Lambda to check that all EC2 instances have a specific tag. The rule works for existing instances but doesn't evaluate new instances when they're launched. What's misconfigured?

**A:** The rule's **trigger type** is set to **Periodic** instead of **Configuration Change**. Periodic rules only evaluate on schedule (1h, 3h, etc.), not when resources are created/modified. Fix: Set the trigger to `ConfigurationItemChangeNotification` for the resource type `AWS::EC2::Instance`. This evaluates immediately when a new instance is launched.

**Why wrong answers are wrong:**
- "Lambda timeout is too short" — would cause evaluation failure, not missing evaluation
- "The scope is wrong" — scope defines which resources, but trigger defines when evaluation happens
- "Config recorder isn't recording EC2" — would mean no CIs at all, not delayed evaluation
- **Keywords:** "doesn't evaluate new instances", "works for existing" → Wrong trigger type (periodic instead of change-triggered)

---

**Q7:** A company has auto-remediation for `restricted-ssh`. It keeps triggering remediation → developer re-opens SSH → remediation triggers again → cycle repeats. This causes operational friction. What's the best approach?

**A:** This indicates a **process problem**, not a technical one. The developer legitimately needs SSH access. Best approach: (1) Add an **SCP** that PREVENTS open SSH from being created (preventive) instead of repeatedly remediating. (2) Provide a **secure alternative** (Session Manager for access, or specific IP-restricted SSH). (3) If temporary access is genuinely needed, use a **time-limited exception** process (tag-based exclusion with TTL). The auto-remediation war indicates the preventive control is missing.

**Keywords:** "remediation cycle", "developer re-opens" → Add preventive control (SCP) + provide alternative access method

---

### Common Exam Traps & Pitfalls

1. **Config is DETECTIVE, not preventive** — it evaluates AFTER changes. Cannot prevent non-compliant resource creation. Use SCPs/IAM/Hooks for prevention.

2. **Config ≠ CloudTrail**: Config = resource state (what IS). CloudTrail = actions (what HAPPENED). Both needed for full audit.

3. **Aggregator is view-only** — you cannot trigger remediation from the aggregator. Remediation runs in the source account.

4. **Config must be enabled** (recorder running) for rules to evaluate. No recording = no evaluations.

5. **Remediation uses SSM Automation** — not Lambda. Lambda is for custom rule EVALUATION logic, not for fixing resources.

6. **Organization conformance packs require delegated admin** — can't deploy from every account.

7. **Config rules have evaluation latency** — not instant. For real-time: use EventBridge + CloudTrail.

8. **Auto-remediation can cause issues** — test in non-production first. Blindly fixing resources can break applications.

9. **S3 bucket required for Config** — delivery channel needs S3 bucket. If bucket is deleted, Config delivery fails.

10. **Config rule scope vs trigger**: Scope = which resources. Trigger = when to evaluate (change vs periodic). Both must be correct.

---

## 7. Cheat Sheet

### Must-Know Facts

| Fact | Detail |
|------|--------|
| Config type | Detective control (NOT preventive) |
| Config vs CloudTrail | State (what IS) vs Actions (what HAPPENED) |
| Managed rules | ~300+ pre-built |
| Custom rules | Lambda-based or CloudFormation Guard |
| Rules per region | 400 (soft limit) |
| Conformance packs per account/region | 50 |
| CI pricing | $0.003 per configuration item |
| Rule evaluation pricing | $0.001 per evaluation |
| Remediation engine | SSM Automation documents |
| Aggregator cost | Free |
| Advanced Queries | SQL-like, free |
| Recording retention | Configurable (30 days to 7 years) |
| Organization deployment | Delegated admin + Organization conformance packs |
| Config + Security Hub | Config findings feed into Security Hub |

### Decision Flowchart

```
"Detect non-compliant resources" → Config Rules
"Prevent non-compliant resource creation" → SCP / IAM / CloudFormation Hooks (NOT Config)
"Auto-fix non-compliant resources" → Config Auto-Remediation (SSM Automation)
"Deploy compliance rules across organization" → Organization Conformance Packs
"View compliance across all accounts" → Config Aggregator
"Query resource configurations with SQL" → Advanced Queries
"Audit who changed a resource" → CloudTrail (not Config)
"See configuration history of a resource" → Config Timeline
"Implement CIS/PCI/HIPAA framework" → AWS-authored conformance packs
"Real-time alert on non-compliance" → EventBridge (Config compliance change event)
"Complex multi-step remediation" → Custom SSM Automation document
```

---

## 8. Additional Deep-Dive

### 10 More Tricky Scenario-Based Questions

**Q1:** A company enables Config with auto-remediation for untagged resources. After enabling, 5,000 existing resources are found non-compliant. All 5,000 remediation actions fire simultaneously. This overwhelms the SSM Automation service and some remediations fail. How to prevent this?

**A:** Configure **remediation concurrency** and **retry settings**. Set `MaximumAutomaticAttempts` (max 5) and `RetryAttemptSeconds`. Also set `Concurrency` in the remediation configuration to limit parallel executions. For initial bulk remediation, consider running remediation manually in batches rather than enabling auto-remediation on existing non-compliant resources.

**Keywords:** "bulk non-compliant", "overwhelms", "simultaneous" → Remediation concurrency limits + manual initial cleanup

---

**Q2:** A company uses Config rule `required-tags` to enforce that all EC2 instances have a `CostCenter` tag. A developer launches an instance via Auto Scaling without the tag. Config shows non-compliant. Auto-remediation adds the tag. But next time ASG launches an instance, it again lacks the tag. What's the root fix?

**A:** Fix the **Launch Template** to include the `CostCenter` tag (or configure **ASG tag propagation**). Remediating individual instances is treating symptoms — the root cause is the ASG/launch template missing the tag. Additionally, set up ASG `Tags` configuration with `PropagateAtLaunch: true`.

**Keywords:** "ASG instances repeatedly non-compliant after remediation" → Fix the launch template/ASG tags (root cause)

---

**Q3:** A company's Config rule uses a Lambda function for evaluation. The Lambda function execution takes 5 seconds. With 10,000 resources changing frequently, the Lambda concurrent executions spike and get throttled. How to handle this?

**A:** (1) Increase Lambda **reserved concurrency** or request account concurrency limit increase. (2) Optimize the Lambda function to execute faster. (3) Switch from **change-triggered** to **periodic** evaluation (reduces evaluation frequency). (4) Consider replacing with a **managed rule** if one exists (no Lambda cost/throttling). (5) Narrow the rule scope to reduce the number of resources triggering evaluation.

**Keywords:** "custom Lambda rule", "throttled", "10,000 resources" → Increase concurrency + optimize + consider periodic

---

**Q4:** A company wants to detect when someone modifies a Config rule itself (tampering with compliance monitoring). How?

**A:** (1) **CloudTrail** logs `config:PutConfigRule` and `config:DeleteConfigRule` API calls. (2) Create an **EventBridge rule** triggered on these API calls → alert via SNS. (3) Use **Config itself** to monitor its own rules as resources (yes, Config records Config rule changes). (4) **SCP** to deny `config:DeleteConfigRule` and `config:PutConfigRule` except from specific admin roles.

**Keywords:** "detect Config rule tampering" → CloudTrail + EventBridge + SCP (prevent modification)

---

**Q5:** A company has Config rules evaluating RDS instances. An RDS instance is created in a new region where Config isn't enabled. The instance is never evaluated. How to prevent this gap?

**A:** (1) **Enable Config in all regions** (even unused ones) — use CloudFormation StackSets to deploy Config across all regions. (2) Use **SCP to deny resource creation in non-approved regions** (preventive). (3) Use **Organization conformance packs** that deploy Config rules to all regions automatically.

**Keywords:** "new region without Config", "never evaluated" → Enable Config in all regions (StackSets) + SCP for region restriction

---

**Q6:** A company uses Config remediation that calls an SSM Automation document to encrypt an unencrypted S3 bucket. The remediation succeeds (applies SSE-S3), but the company's policy requires SSE-KMS with a specific CMK. The default remediation document only enables SSE-S3. How to fix?

**A:** Create a **custom SSM Automation document** that applies SSE-KMS encryption with the specific CMK ARN (not the generic `AWS-EnableS3BucketEncryption` which defaults to SSE-S3). Replace the remediation configuration to use the custom document and pass the CMK ARN as a parameter.

**Keywords:** "remediation applies wrong encryption type" → Custom SSM Automation document with correct parameters

---

**Q7:** A company notices Config costs are $5,000/month. They have Config recording ALL resource types in 20 accounts across 4 regions. Most rules only evaluate S3, EC2, and IAM. How to reduce costs?

**A:** (1) **Limit recording to only needed resource types** (S3, EC2, IAM, and dependencies) instead of all types. This dramatically reduces configuration items. (2) **Use periodic recording** for resources that rarely change. (3) **Remove duplicate rules** if any. (4) Set **S3 lifecycle policies** on the Config delivery bucket. The biggest cost driver is likely the sheer volume of CIs from recording everything.

**Keywords:** "Config costs too high", "recording ALL types" → Record only evaluated resource types

---

**Q8:** A company's Config rule shows ALL existing resources as COMPLIANT, but they know some resources are non-compliant. The rule was just created. What's happening?

**A:** New Config rules don't automatically evaluate existing resources unless triggered. If the rule is **change-triggered only**, existing resources won't be evaluated until they change. Fix: (1) Add **periodic trigger** so existing resources are evaluated on schedule. (2) Manually trigger evaluation via `StartConfigRulesEvaluation` API. (3) Wait for periodic evaluation to run.

**Keywords:** "new rule shows all compliant", "known non-compliant exist" → Rule is change-triggered only; add periodic trigger or manually start evaluation

---

**Q9:** A company uses Config with auto-remediation. A remediation for a critical production resource (RDS instance) causes an unplanned restart, creating a brief outage. How to prevent this in the future?

**A:** (1) **Disable auto-remediation for production** — use manual remediation (requires human approval). (2) Use **resource exceptions** (tag production resources to skip auto-remediation). (3) Implement **maintenance window constraints** in the SSM Automation document (only execute during off-hours). (4) Test remediation documents in dev/staging before production.

**Keywords:** "auto-remediation caused production outage" → Disable auto-remediation for prod; use manual + maintenance windows

---

**Q10:** A company's Config Aggregator shows 95% compliance across 100 accounts. The auditor asks: "What was the compliance posture 30 days ago?" Can Config answer this?

**A:** **Partially.** Config maintains a **compliance timeline** per resource (historical compliance states). However, the aggregator shows CURRENT state. For historical aggregate compliance: (1) Use **Config Compliance Change** events sent to S3 via Firehose → query with Athena for historical trends. (2) AWS Config doesn't natively store aggregate compliance history — you must build reporting from the compliance change stream.

**Keywords:** "historical compliance posture", "30 days ago" → Config tracks per-resource timeline but not aggregate history; build custom reporting

---

### Compare: Config vs Security Hub vs GuardDuty

| Aspect | Config | Security Hub | GuardDuty |
|--------|--------|-------------|-----------|
| **Purpose** | Configuration compliance | Centralized security findings | Threat detection |
| **Type** | Detective (configuration) | Aggregator + standards | Detective (threats/anomalies) |
| **Input** | Resource configurations | Findings from Config, GuardDuty, Inspector, etc. | VPC Flow Logs, DNS, CloudTrail |
| **Output** | Compliant/Non-compliant | Finding severity (Critical/High/Medium/Low) | Finding (threat type) |
| **Remediation** | Built-in (SSM Automation) | Custom actions (Lambda) | No built-in remediation |
| **Exam Tip** | "Is resource configured correctly?" | "Centralized security view" | "Is there an active threat?" |

### Compare: Config vs CloudTrail

| Aspect | Config | CloudTrail |
|--------|--------|------------|
| **Records** | Resource STATE (configuration) | API ACTIONS (who did what) |
| **Question answered** | "What is/was the configuration?" | "Who changed it and when?" |
| **Timeline** | Configuration timeline per resource | Event history per API call |
| **Compliance** | Evaluates against rules | No compliance evaluation |
| **Together** | Combine: Config (what changed) + CloudTrail (who changed it) |

### Decision Tree: Compliance Tool Selection

```
Need to EVALUATE resource configuration?
  └── AWS Config Rules

Need to PREVENT non-compliant resource creation?
  └── SCP (account level) / IAM (user level) / CF Hooks (template level)

Need to AUTO-FIX non-compliant resources?
  └── Config Auto-Remediation (SSM Automation)

Need CENTRALIZED security view across services?
  └── Security Hub (aggregates Config, GuardDuty, Inspector findings)

Need to DETECT active threats?
  └── GuardDuty

Need to AUDIT who made changes?
  └── CloudTrail

Need ORGANIZATION-WIDE compliance framework?
  └── Organization Conformance Packs (via delegated admin)

Need REAL-TIME alert on non-compliance?
  └── EventBridge rule on Config compliance change event

Need HISTORICAL configuration of a resource?
  └── Config Configuration Timeline + S3 history
```
