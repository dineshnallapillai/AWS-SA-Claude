# Centralized Logging & Security (CloudTrail, Config, Security Hub, GuardDuty)

---

## 1. Core Concepts & Theory

### AWS CloudTrail

- **What it does:** Records API calls (management events, data events, Insights events) across AWS accounts.
- **Trail types:**
  - **Management events:** Control-plane operations (CreateBucket, RunInstances, etc.) — enabled by default for 90 days (Event History).
  - **Data events:** Data-plane operations (S3 object-level: GetObject/PutObject, Lambda invocations, DynamoDB streams). NOT logged by default — must be explicitly enabled.
  - **Insights events:** Detects unusual API activity (e.g., spike in TerminateInstances calls).
- **Organization Trail:** Single trail in the management account that automatically collects events from all member accounts. Members cannot modify or delete the org trail.
- **Log delivery:** JSON files → S3 bucket (typically ~5-15 min delay). Can also send to CloudWatch Logs for real-time alerting.
- **Log file integrity validation:** SHA-256 digest files delivered hourly — proves logs haven't been tampered with.
- **Limits & Quotas:**
  - 5 trails per region (can be increased).
  - Event History: free, 90 days, management events only, non-configurable.
  - S3 delivery: first copy of management events free; additional copies & data events charged.
  - Maximum 250 data event selectors per trail.

### AWS Config

- **What it does:** Continuously records resource configuration state and evaluates compliance against rules.
- **Key components:**
  - **Configuration Recorder:** Captures resource configuration changes (creates Configuration Items).
  - **Config Rules:** Evaluate whether resources comply (managed rules, custom Lambda rules, CloudFormation Guard rules).
  - **Conformance Packs:** Collection of Config rules + remediation actions deployed as a single entity.
  - **Aggregator:** Multi-account, multi-region view — collects data from source accounts (does NOT need to be in an org, but org-level aggregator is simpler).
- **Organization Config Rules & Conformance Packs:** Deploy from delegated admin or management account to all member accounts.
- **Advanced Queries:** SQL-like queries against current configuration state using AWS Config's data model.
- **Limits:**
  - 500 Config rules per account per region (soft limit).
  - Aggregator: max 10,000 accounts.
  - Configuration Recorder: 1 per region per account.
  - Conformance Packs: 50 per account per region.

### AWS Security Hub

- **What it does:** Central dashboard for security findings; aggregates, normalizes, and prioritizes findings from multiple AWS services and third-party tools.
- **Finding format:** AWS Security Finding Format (ASFF) — standard JSON schema for all findings.
- **Standards:** CIS AWS Foundations Benchmark, AWS Foundational Security Best Practices, PCI DSS, NIST 800-53.
- **Integrations:** GuardDuty, Inspector, Macie, Firewall Manager, IAM Access Analyzer, third-party (Prowler, Qualys, etc.).
- **Cross-region aggregation:** Designate a single aggregation region to view findings from all regions.
- **Organization integration:** Automatically enable Security Hub in all member accounts from the delegated admin account.
- **Automated response:** Finding → EventBridge rule → Lambda/Step Functions/SSM Automation for auto-remediation.
- **Limits:**
  - 10,000 member accounts per admin.
  - Findings retained for 90 days after last update.
  - 1,000 custom actions per account per region.
  - Batch import: 100 findings per request.

### AWS GuardDuty

- **What it does:** Intelligent threat detection using ML, anomaly detection, and integrated threat intelligence.
- **Data sources:**
  - **Foundational:** VPC Flow Logs, DNS logs, CloudTrail management events (always analyzed — no need to enable these separately for GuardDuty).
  - **Optional/Protection plans:** S3 data events, EKS audit logs, RDS login activity, Lambda network activity, EBS malware protection, Runtime Monitoring (EKS/ECS/EC2).
- **Finding types:** Recon, initial access, persistence, privilege escalation, defense evasion, credential access, data exfiltration, impact, crypto mining.
- **Organization integration:** Delegated administrator auto-enables GuardDuty for all org accounts.
- **Multi-account:** Invitation-based (standalone) or Organizations-based (recommended).
- **Malware Protection:** Triggered automatically on certain findings (e.g., crypto-mining), scans EBS volumes (creates snapshot → scans → deletes snapshot).
- **Limits:**
  - 5,000 member accounts per detector (org-based).
  - 10,000 trusted IP addresses, 100,000 threat IPs in threat lists.
  - Findings retained for 90 days.

---

## 2. Design Patterns & Best Practices

### Centralized Logging Architecture (Multi-Account)

```
Member Accounts ──► Organization Trail ──► Central S3 Bucket (Log Archive Account)
                                                    │
                                                    ├── Glacier lifecycle for cost
                                                    ├── S3 Object Lock (WORM) for compliance
                                                    └── Athena / OpenSearch for analysis

Member Accounts ──► Config Aggregator ──► Delegated Admin Account
Member Accounts ──► Security Hub ──► Delegated Admin (aggregation region)
Member Accounts ──► GuardDuty ──► Delegated Admin Account
```

### Best Practices

| Service | Best Practice | Why |
|---------|--------------|-----|
| CloudTrail | Use Organization Trail + dedicated Log Archive account | Single pane, member accounts cannot tamper |
| CloudTrail | Enable log file integrity validation | Prove no tampering for audits |
| CloudTrail | Enable data events selectively (not all S3 buckets) | Cost control — data events are expensive |
| CloudTrail | Send to CloudWatch Logs + create metric filters | Real-time alerting on critical API calls |
| Config | Use Organization Conformance Packs | Consistent compliance across all accounts |
| Config | Use Aggregator in dedicated Security/Audit account | Cross-account visibility without per-account login |
| Config | Use remediation actions (SSM Automation) | Auto-fix non-compliant resources |
| Security Hub | Enable cross-region aggregation | Single region view of all findings |
| Security Hub | Use delegated admin (not management account) | Least-privilege; management account stays clean |
| GuardDuty | Auto-enable for new accounts via org integration | No gaps in threat detection coverage |
| GuardDuty | Export findings to S3 for long-term retention | 90-day limit in console; need longer for forensics |
| GuardDuty | Suppress expected findings (suppression rules) | Reduce noise, focus on real threats |

### When to Use / Not Use

| Scenario | Use | Don't Use |
|----------|-----|-----------|
| "Who called this API?" | CloudTrail | Config |
| "Is this resource compliant now?" | Config | CloudTrail |
| "Is there an active threat?" | GuardDuty | Config |
| "Show me all security findings in one place" | Security Hub | Individual service consoles |
| "Prove logs haven't been tampered" | CloudTrail integrity validation | Config |
| "Auto-remediate non-compliant resources" | Config Rules + SSM remediation | GuardDuty |
| "Detect unusual API patterns" | CloudTrail Insights or GuardDuty | Config |
| "Track resource configuration over time" | Config (timeline) | CloudTrail alone |

### Well-Architected Alignment (Security Pillar)

- **SEC04 — Detect and investigate events:** CloudTrail + GuardDuty + Security Hub
- **SEC01 — Operate securely:** Config rules enforce secure baselines
- **SEC07 — Classify data:** Macie (feeds into Security Hub)
- **SEC08 — Protect data at rest:** Config rules check encryption settings

---

## 3. Security & Compliance

### Protecting the Logging Infrastructure

| Control | Implementation |
|---------|---------------|
| Prevent log deletion | S3 Object Lock (Governance or Compliance mode), bucket policy denying s3:DeleteObject |
| Prevent trail modification | SCP denying `cloudtrail:StopLogging`, `cloudtrail:DeleteTrail` in member accounts |
| Encrypt logs | SSE-S3 (default) or SSE-KMS (recommended for cross-account access control) |
| Restrict access to logs | Dedicated Log Archive account, bucket policy with explicit deny for non-authorized principals |
| Detect tampering | CloudTrail log file integrity validation (digest files) |
| Ensure logging is always on | Config rule: `cloud-trail-enabled`, `multi-region-cloud-trail-enabled` |

### IAM Considerations

- **CloudTrail:** Trail's S3 bucket policy must allow `cloudtrail.amazonaws.com` to write. For org trail, the management account owns the trail.
- **Config:** Requires a service-linked role `AWSServiceRoleForConfig`. Cross-account aggregator needs authorization from source accounts (or org-level aggregator avoids this).
- **Security Hub:** Delegated admin can view/manage findings for all member accounts. Uses service-linked role.
- **GuardDuty:** Delegated admin can enable/disable protections for all members. Uses service-linked role.

### Key SCPs for Security Guardrails

```json
{
  "Effect": "Deny",
  "Action": [
    "cloudtrail:StopLogging",
    "cloudtrail:DeleteTrail",
    "cloudtrail:UpdateTrail",
    "config:StopConfigurationRecorder",
    "config:DeleteConfigurationRecorder",
    "guardduty:DisableOrganizationAdminAccount",
    "guardduty:DeleteDetector",
    "securityhub:DisableSecurityHub"
  ],
  "Resource": "*"
}
```

### Compliance Frameworks

- **CloudTrail + S3 Object Lock** → Satisfies audit trail requirements (SOC 2, PCI DSS, HIPAA)
- **Config Conformance Packs** → Pre-built packs for PCI DSS, HIPAA, NIST
- **Security Hub Standards** → Automated scoring against CIS, PCI DSS, NIST 800-53

---

## 4. Cost Optimization

### Pricing Model Summary

| Service | Free Tier / Included | Paid |
|---------|---------------------|------|
| CloudTrail | First copy of management events free; 90-day Event History free | Additional trail copies: $2/100K management events; Data events: $0.10/100K events; Insights: $0.35/100K events analyzed |
| Config | None (pay per rule evaluation + configuration items recorded) | $0.003/config item recorded; $0.001/rule evaluation/region |
| Security Hub | 30-day free trial | Finding ingestion events: first 10K free/account/region/month, then tiered ($0.00003-$0.000015/event); Security checks: $0.0010/check/region |
| GuardDuty | 30-day free trial (all features) | Per volume of data analyzed: VPC Flow Logs, DNS, CloudTrail events — tiered pricing |

### Cost Optimization Strategies

1. **CloudTrail:**
   - Use ONE organization trail (not per-account trails) — avoids duplicate charges.
   - Be selective with data events — only enable for critical buckets/functions.
   - Lifecycle S3 logs to Glacier Deep Archive after 90 days.
   - Use S3 Intelligent-Tiering for log buckets with unpredictable access.

2. **Config:**
   - Only record resource types you care about (not all).
   - Use periodic evaluation mode for rules that don't need real-time (reduces evaluations).
   - Remove unused/duplicate rules.

3. **Security Hub:**
   - Disable standards/controls you don't need (each enabled control = checks = cost).
   - Use cross-region aggregation in one region, not Security Hub enabled in all regions with separate admin views.

4. **GuardDuty:**
   - Review optional protection plans — disable those not needed (e.g., RDS protection if you don't use RDS).
   - VPC Flow Log analysis is the biggest cost driver — optimize VPC design to reduce noisy flows.

### Cost Traps

- Enabling data events for ALL S3 buckets in CloudTrail (very expensive in data-heavy accounts).
- Running Config recorder for all resource types when you only need a few.
- Enabling all GuardDuty protection plans without reviewing cost.
- Duplicate trails (each additional copy charges for management events).
- Security Hub: enabling all standards across all regions with no consolidation.

---

## 5. High Availability, Disaster Recovery & Resilience

### Service Resilience

| Service | HA Model | DR Consideration |
|---------|----------|-----------------|
| CloudTrail | Regional service, logs to S3 (11 9s durability) | Multi-region trail ensures capture even if one region is impaired; S3 cross-region replication for DR |
| Config | Regional service | Aggregator provides cross-region view; no native DR — relies on S3 durability for data |
| Security Hub | Regional (cross-region aggregation available) | Aggregation region provides consolidated view; findings replicated to aggregation region |
| GuardDuty | Regional | Enable in all active regions; no cross-region aggregation natively (use Security Hub for that) |

### Resilience Patterns

1. **Multi-region trail:** Captures events in all regions, including regions you don't actively use (detect unauthorized activity in unused regions).
2. **S3 cross-region replication:** Replicate log bucket to a second region for DR.
3. **S3 Object Lock:** Prevents deletion even by root — critical for compliance.
4. **Dedicated Log Archive account:** Isolated from blast radius of any single account compromise.
5. **Enable services in ALL regions:** GuardDuty and Config should run in all regions to detect unauthorized resource usage (common exam scenario).

### RPO/RTO for Logging

- **RPO:** Near-zero for CloudTrail (events captured within minutes); Config records continuously.
- **RTO:** N/A for logging (passive collection). For alerting/response: EventBridge rules trigger in seconds.
- **Key risk:** If CloudTrail logging is stopped, there's a gap. SCP prevents this. Even so, CloudTrail delivers with ~5-15 min delay — real-time monitoring needs CloudWatch Logs integration.

---

## 6. Exam-Focused Section

### 6.1 Straightforward Questions

**Q1:** A company wants to ensure that API activity is logged across all 50 accounts in their AWS Organization. What is the most operationally efficient solution?

**A:** Create an Organization Trail in the management account. It automatically applies to all member accounts and cannot be modified by them.

---

**Q2:** A security team needs to receive alerts within 5 minutes of any root user login across all accounts. What should they configure?

**A:** Organization Trail → CloudWatch Logs → Metric Filter for `ConsoleLogin` where `userIdentity.type = Root` → CloudWatch Alarm → SNS notification.

---

**Q3:** Which service should be used to automatically remediate an S3 bucket that becomes public?

**A:** AWS Config rule (`s3-bucket-public-read-prohibited`) with an automatic remediation action using SSM Automation document (`AWS-DisableS3BucketPublicReadWrite`).

---

**Q4:** A company needs to view all security findings from GuardDuty, Inspector, and Macie in a single dashboard. What service provides this?

**A:** AWS Security Hub — it aggregates findings in ASFF format from all these services.

---

**Q5:** How can you prove that CloudTrail log files have not been modified after delivery?

**A:** Enable CloudTrail log file integrity validation. CloudTrail creates SHA-256 digest files that can be used to verify log file integrity using the AWS CLI `validate-logs` command.

---

**Q6:** A company uses AWS Organizations and needs to track compliance of all resources against CIS benchmarks across all accounts. What's the best approach?

**A:** Enable Security Hub with the CIS AWS Foundations Benchmark standard across all member accounts using the delegated admin. Use cross-region aggregation for a single-pane view.

---

**Q7:** What data source does GuardDuty use that does NOT require you to enable VPC Flow Logs manually?

**A:** GuardDuty analyzes VPC Flow Logs, DNS logs, and CloudTrail events independently — it uses its own internal data pipeline. You do NOT need to enable VPC Flow Logs or DNS logging yourself for GuardDuty to work.

---

### 6.2 Tricky/Scenario-Based Questions

**Q1:** A company has an organization trail logging to a central S3 bucket in the Log Archive account. A junior admin in a member account accidentally creates a trail pointing to a different S3 bucket in their account. What happens?

**A:** Both trails log events in that member account. The org trail continues to work. The junior admin's trail is an additional trail, and management events logged to it will be charged ($2/100K events). **Exam trap:** The org trail is NOT disrupted — it's additive. The concern is cost and potential log sprawl.

---

**Q2:** A security team notices that GuardDuty is generating findings for expected traffic from a vulnerability scanner. What is the BEST approach?

**A:** Create a **suppression rule** in GuardDuty to auto-archive findings matching the scanner's IP/behavior. Do NOT add the scanner IP to the trusted IP list — trusted IPs only suppress findings where that IP is the actor, and the list has limitations. Suppression rules are more flexible and purpose-built for this.

**Why other options are wrong:**
- Trusted IP list: Only works for certain finding types; max 10,000 IPs; less granular.
- Disabling the finding type entirely: Loses detection for actual threats of that type.
- Creating a Config rule: Wrong service — Config doesn't manage GuardDuty findings.

---

**Q3:** An architect needs to ensure that deleted CloudTrail logs can be recovered for 7 years to meet regulatory requirements. The solution must prevent even the root user from deleting logs. What should they implement?

**A:** S3 Object Lock in **Compliance mode** with a 7-year retention period. Compliance mode prevents deletion by ANY user including root and cannot be shortened. Combined with a bucket policy denying `s3:DeleteObject`.

**Why other options are wrong:**
- Governance mode: Can be overridden by users with `s3:BypassGovernanceRetention` permission.
- Versioning alone: Doesn't prevent deletion of versions by root.
- Glacier Vault Lock: Works but more complex and less integrated with CloudTrail S3 delivery.

---

**Q4:** A company wants Config to evaluate compliance of resources across 200 accounts. They are NOT using AWS Organizations. How should they set up the Config Aggregator?

**A:** Create a Config Aggregator in the central account, then each source account must explicitly authorize the aggregator account. Without Organizations, you cannot use an organization-level aggregator — individual account authorization is required.

---

**Q5:** An organization uses Security Hub with the AWS Foundational Security Best Practices standard. They notice the `Config.1` finding is FAILED. What does this mean and how do they fix it?

**A:** `Config.1` checks that AWS Config is enabled and recording all resource types with a global recorder in at least one region. Fix: Enable Config in that region with the configuration recorder running and recording all supported resource types. **Exam trap:** Security Hub relies on Config rules for many of its checks — if Config is not enabled, many Security Hub findings will fail.

---

**Q6:** A company needs near-real-time threat detection in their EKS cluster. They already have GuardDuty enabled. What additional step is needed?

**A:** Enable **EKS Runtime Monitoring** (or EKS Protection) in GuardDuty. Basic GuardDuty only analyzes EKS audit logs for control-plane threats. Runtime Monitoring deploys a security agent to detect container-level threats (process execution, file access, network connections).

**Why other options are wrong:**
- Enabling EKS audit log monitoring alone: Only detects control-plane anomalies (e.g., suspicious kubectl commands), not runtime threats inside containers.
- CloudTrail data events: Doesn't monitor inside containers.
- Third-party agent: Works but not the MOST integrated/operationally efficient AWS-native answer.

---

**Q7:** After enabling GuardDuty across all accounts, findings from member accounts are only visible in each individual member account console, not the delegated admin. What is likely misconfigured?

**A:** The member accounts were likely added via invitation (standalone) BEFORE the organization integration was set up, and haven't been migrated to org-based management. With invitation-based membership, the admin sees findings; but if the accounts were never properly associated (accepted the invitation), the admin won't see them. **Resolution:** Migrate to organization-based GuardDuty management via the delegated admin.

---

### 6.3 Common Exam Traps & Pitfalls

| Trap | Reality |
|------|---------|
| "Enable VPC Flow Logs for GuardDuty" | GuardDuty has its own independent data source — you DON'T need to manually enable VPC Flow Logs |
| "CloudTrail records everything by default" | Only MANAGEMENT events are recorded. Data events (S3 object, Lambda invoke) require explicit opt-in |
| "Config Aggregator gives you control over member accounts" | Aggregator is READ-ONLY — it aggregates data but cannot push rules or remediate in source accounts |
| "Security Hub detects threats" | Security Hub AGGREGATES findings — it doesn't detect. GuardDuty detects threats |
| "Organization Trail can be modified by member accounts" | Members CANNOT modify, stop, or delete an organization trail — only the management account can |
| "CloudTrail delivers logs in real time" | ~5-15 minute delay to S3. For real-time, stream to CloudWatch Logs |
| "Config rules auto-remediate by default" | Remediation must be explicitly configured — it's not automatic |
| "GuardDuty can block threats" | GuardDuty only DETECTS and generates findings — it doesn't block. Use Lambda/Step Functions for response |
| "Security Hub needs GuardDuty to work" | They're independent — Security Hub can work without GuardDuty, and vice versa (but best together) |
| "S3 Object Lock Governance mode is tamper-proof" | Only Compliance mode is truly tamper-proof — Governance can be overridden with `s3:BypassGovernanceRetention` |

---

## 7. Cheat Sheet

### One-Page Summary

| Service | Purpose | Key Differentiator |
|---------|---------|-------------------|
| **CloudTrail** | API activity logging (who did what, when) | Audit trail — the "security camera" |
| **Config** | Resource compliance & configuration history | Configuration state + rules — the "compliance officer" |
| **Security Hub** | Aggregated security posture dashboard | Normalizes findings — the "single pane of glass" |
| **GuardDuty** | Intelligent threat detection | ML-based anomaly detection — the "security guard" |

### Decision Flowchart

```
Question: What happened? ──► CloudTrail (API calls)
Question: Is it compliant? ──► Config (rules & state)
Question: Is it under attack? ──► GuardDuty (threat detection)
Question: Show me everything ──► Security Hub (aggregator)
Question: Fix it automatically ──► Config remediation OR EventBridge + Lambda
Question: Prove no tampering ──► CloudTrail log integrity validation
Question: Store forever, can't delete ──► S3 Object Lock Compliance mode
```

### Key Numbers to Remember

| Item | Value |
|------|-------|
| CloudTrail Event History retention | 90 days |
| CloudTrail delivery delay to S3 | ~5-15 minutes |
| GuardDuty finding retention | 90 days |
| Security Hub finding retention | 90 days (after last update) |
| Config rules per account/region | 500 (soft limit) |
| Max trails per region | 5 |
| Organization Trail modification by members | NOT POSSIBLE |
| GuardDuty - need to enable Flow Logs? | NO |

---

## 8. Additional Tricky Scenario-Based Questions

**Q8:** A financial company must retain ALL CloudTrail logs (management + data events) for 10 years with the ability to query them. The current approach using S3 + Athena is too costly for long-term storage. What's the best solution?

**A:** Use **CloudTrail Lake** — a managed data lake for CloudTrail events. It stores events in Apache ORC format, supports SQL queries, and allows retention up to 2,557 days (7 years) or longer with custom configurations. For 10 years, combine with S3 export from CloudTrail Lake + Glacier Deep Archive for cold storage beyond 7 years.

---

**Q9:** A company discovers that someone disabled GuardDuty in the eu-west-1 region. By the time they re-enable it, what historical data is lost?

**A:** ALL historical findings and learned baseline are lost. GuardDuty doesn't persist data after being disabled — the detector is deleted. When re-enabled, it starts fresh with no learned baseline (more false positives initially). This is why SCPs preventing `guardduty:DeleteDetector` are critical.

---

**Q10:** An organization has Config enabled but an architect notices that compliance scores in Security Hub don't match the Config dashboard. Why?

**A:** Security Hub uses its OWN set of Config rules (prefixed with `securityhub-`) that it creates and manages independently. These are separate from any custom Config rules the organization created. The two dashboards evaluate different rule sets, so scores differ.

---

**Q11:** A company wants to detect when an IAM user's access keys are used from an unusual geographic location. Which service and finding type applies?

**A:** **GuardDuty** — finding type `UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration` or `Discovery:IAMUser/AnomalousBehavior`. GuardDuty's ML baseline detects API calls from unusual locations. CloudTrail alone only records the event but doesn't flag anomalies.

---

**Q12:** An architect is designing a solution where Config rules must evaluate compliance differently in production vs. development accounts. What's the best approach?

**A:** Use **Organization Conformance Packs with template parameters** or deploy different conformance packs to different OUs. You can also use Config rule input parameters that vary by account (e.g., `maxAccessKeyAge` = 90 for prod, 180 for dev).

---

**Q13:** After setting up an Organization Trail, the security team notices that some accounts' CloudTrail logs are missing from the central S3 bucket. What should they check?

**A:** Check the **S3 bucket policy** — it must allow `cloudtrail.amazonaws.com` to write from all member accounts. Also verify there are no **SCPs blocking CloudTrail's ability to call `s3:PutObject`** on the bucket. Additionally, check if those accounts were added to the organization AFTER the trail was created (they should auto-enroll, but verify).

---

**Q14:** A company uses Security Hub and wants to automatically create JIRA tickets for HIGH severity findings. What architecture should they use?

**A:** Security Hub finding → **EventBridge rule** (filter: `severity.label = HIGH`) → Lambda function → JIRA API. Use a custom action in Security Hub for on-demand ticket creation, or the EventBridge integration for automatic. **Do NOT** use SNS for this — EventBridge provides native filtering on ASFF fields.

---

**Q15:** The CISO asks: "If an attacker compromises the Log Archive account, can they delete all our audit logs?" What controls prevent this?

**A:** S3 Object Lock in **Compliance mode** — even the root user of the Log Archive account cannot delete objects before the retention period expires. Additionally: MFA Delete on the bucket, IAM policies denying delete operations, and ideally the bucket uses a KMS key managed in a separate account (so even with bucket access, decryption requires cross-account KMS permissions).

---

**Q16:** A company is streaming CloudTrail logs to CloudWatch Logs for alerting. They notice that their CloudWatch costs have increased significantly. What's the most cost-effective optimization?

**A:** Only stream **management events** to CloudWatch Logs (not data events). Use CloudWatch metric filters ONLY for the specific API calls you need to alert on. For analysis and querying, use S3 + Athena or CloudTrail Lake instead of CloudWatch Logs Insights. Consider reducing the retention period in the CloudWatch Log group.

---

**Q17:** What's the difference between GuardDuty's Trusted IP List and a Suppression Rule, and when would you use each?

**A:**
- **Trusted IP List:** GuardDuty won't generate findings where the trusted IP is the source. Limited to IP addresses only. Max 10,000 IPs. Applies globally to all finding types.
- **Suppression Rule:** Auto-archives findings matching specific criteria (IP, instance, finding type, severity, etc.). More granular. Findings are still generated but auto-archived.
- **Use Trusted IP List:** For corporate IP ranges you NEVER want flagged (offices, VPN endpoints).
- **Use Suppression Rule:** For known patterns (vulnerability scanners, expected behaviors) where you want fine-grained control.

---

## 9. Service Comparison Table

| Dimension | CloudTrail | Config | Security Hub | GuardDuty |
|-----------|-----------|--------|--------------|-----------|
| **Primary Use Case** | API audit logging | Compliance & configuration tracking | Security posture aggregation | Threat detection |
| **Data Source** | API calls | Resource state changes | Findings from other services | Flow Logs, DNS, CloudTrail, runtime |
| **Scope** | Who did what | Is it configured correctly | What findings exist | Is there a threat |
| **Auto-Remediation** | No (event source only) | Yes (SSM Automation) | No (triggers EventBridge) | No (findings only) |
| **Org Deployment** | Organization Trail | Org Config Rules + Aggregator | Delegated admin auto-enable | Delegated admin auto-enable |
| **Cross-Region** | Multi-region trail | Aggregator | Cross-region aggregation | No native (use Security Hub) |
| **Retention** | Event History: 90 days; S3: unlimited | Unlimited (in S3) | 90 days | 90 days |
| **Real-time** | ~5-15 min to S3; near-real-time to CW Logs | Near-real-time (on change) | Near-real-time (on finding) | Near-real-time |
| **Pricing Model** | Per event (management/data) | Per config item + rule evaluation | Per finding + per check | Per data volume analyzed |
| **Exam Tip** | "audit trail", "who called", "prove" | "compliant", "resource state", "history" | "aggregated view", "single pane" | "detect threat", "anomaly", "ML" |

---

## 10. When to Pick Service A over Service B

| Exam Keyword/Scenario | Pick This | Not That |
|----------------------|-----------|----------|
| "Track all API calls for compliance" | CloudTrail | Config |
| "Know current configuration state" | Config | CloudTrail |
| "Know configuration 6 months ago" | Config (timeline) | CloudTrail |
| "Detect cryptocurrency mining" | GuardDuty | CloudTrail Insights |
| "Single dashboard for all security" | Security Hub | Individual service consoles |
| "Auto-fix non-compliant resources" | Config + SSM Automation | Security Hub |
| "Detect compromised EC2 instance" | GuardDuty | Config |
| "Verify encryption is enabled" | Config rule | GuardDuty |
| "Unusual API call volume" | CloudTrail Insights | GuardDuty |
| "Unusual API call location/pattern" | GuardDuty | CloudTrail Insights |
| "CIS Benchmark scoring" | Security Hub | Config alone |
| "Prove audit logs untampered" | CloudTrail integrity validation | S3 versioning alone |
| "Prevent log deletion" | S3 Object Lock (Compliance mode) | IAM policy alone |
| "Alert on root login" | CloudTrail → CloudWatch → Alarm | GuardDuty (also works but slower) |
| "Inventory all resources" | Config (advanced queries) | CloudTrail |

---

## 11. "Gotcha" Differences

| Gotcha | Details |
|--------|---------|
| CloudTrail Insights vs. GuardDuty anomaly detection | Insights = unusual API VOLUME (spike in calls). GuardDuty = unusual API BEHAVIOR (new IP, impossible travel, known bad actor). |
| Config Aggregator vs. Organization Config Rules | Aggregator = read-only data collection. Org Rules = push rules to member accounts. They serve different purposes. |
| Security Hub controls vs. Config rules | Security Hub creates its own `securityhub-*` Config rules. These are separate from your custom rules. Disabling a Security Hub control doesn't delete the underlying Config rule immediately. |
| GuardDuty "uses VPC Flow Logs" vs. "enable VPC Flow Logs" | GuardDuty has its own independent telemetry — it does NOT read your VPC Flow Log configurations. You don't need Flow Logs enabled. |
| Organization Trail vs. per-account trail | Org trail: owned by management account, applies everywhere, members can't modify. Per-account trail: owned by member, additional cost if org trail also exists. |
| Config "recording" vs. "evaluating" | Recording = capturing config items. Evaluating = running rules. You can record without rules, but you can't evaluate without recording. |
| Security Hub "aggregation region" vs. "admin account" | Aggregation region = where you view consolidated findings from ALL regions. Admin account = the account that manages member accounts. They can be the same or different. |
| GuardDuty disabling vs. suspending | Suspending: Stops monitoring but retains findings and configuration. Disabling: DELETES everything — detector, findings, learned baseline. Irreversible. |
| CloudTrail "management events" in free tier | Only FIRST COPY is free. Second trail logging management events = charged. Data events are ALWAYS charged. |
| Config rule "compliance" status | A rule can show "Not applicable" if no matching resources exist — this is NOT the same as "Compliant". |

---

## 12. One-Page Decision Tree

```
┌─ "The exam question asks about..."
│
├── LOGGING / AUDIT / "WHO DID WHAT"
│   └── CloudTrail
│       ├── API calls only? → Management events (free first copy)
│       ├── S3 object access / Lambda invocations? → Data events ($$)
│       ├── Unusual call volume? → CloudTrail Insights
│       ├── Real-time alert? → CloudWatch Logs + Metric Filter
│       ├── Tamper-proof? → Log file integrity validation
│       └── Multi-account? → Organization Trail
│
├── COMPLIANCE / "IS IT CONFIGURED RIGHT" / "RESOURCE STATE"
│   └── Config
│       ├── Check compliance? → Config Rule (managed or custom)
│       ├── Auto-fix? → Remediation action (SSM Automation)
│       ├── Multi-account view? → Aggregator
│       ├── Bulk deploy rules? → Organization Conformance Pack
│       └── Query current state? → Advanced Queries (SQL)
│
├── THREAT DETECTION / "IS IT UNDER ATTACK" / "ANOMALY"
│   └── GuardDuty
│       ├── EC2 compromise? → UnauthorizedAccess findings
│       ├── Crypto mining? → CryptoCurrency finding type
│       ├── Container threats? → EKS/ECS Runtime Monitoring
│       ├── S3 data exfil? → S3 Protection
│       ├── Known expected traffic? → Suppression Rule
│       └── Multi-account? → Delegated admin (org-based)
│
├── AGGREGATED VIEW / "SINGLE PANE" / "PRIORITIZE FINDINGS"
│   └── Security Hub
│       ├── Multi-service findings? → ASFF normalization
│       ├── Compliance scoring? → Security standards (CIS, PCI)
│       ├── Auto-response? → EventBridge integration
│       ├── Cross-region view? → Aggregation region
│       └── Multi-account? → Delegated admin
│
└── PROTECTION / "PREVENT DELETION" / "IMMUTABLE"
    └── S3 Object Lock
        ├── Even root can't delete? → Compliance mode
        ├── Admin override possible? → Governance mode
        └── Combined with? → Bucket policy + MFA Delete + SCP
```

---

Generated for AWS SAP-C02 exam preparation. Focus on understanding the boundaries between these services — the exam tests whether you know which service to apply for a given scenario.
