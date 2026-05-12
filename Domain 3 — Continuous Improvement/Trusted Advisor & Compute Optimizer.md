# Trusted Advisor & Compute Optimizer

## 1. Core Concepts & Theory

### AWS Trusted Advisor

**Overview:**
- **Best practice recommendation engine** that inspects your AWS environment
- Provides recommendations across **5 categories**: Cost Optimization, Performance, Security, Fault Tolerance, Service Limits
- Different access levels based on **support plan**

**Support Plan Access:**
| Feature | Basic/Developer | Business/Enterprise |
|---------|----------------|-------------------|
| Core security checks | Yes (7 checks) | Yes (all checks) |
| Service limit checks | Yes | Yes |
| Full check categories | No | Yes (all 5 categories, 100+ checks) |
| AWS Support API access | No | Yes |
| CloudWatch integration | No | Yes |
| Programmatic refresh | No | Yes |
| Weekly notification email | No | Yes (configurable) |

**Core Security Checks (Available to ALL):**
1. S3 Bucket Permissions (open access)
2. Security Groups — Unrestricted Access (0.0.0.0/0 on specific ports)
3. IAM Use (is IAM being used vs root)
4. MFA on Root Account
5. EBS Public Snapshots
6. RDS Public Snapshots
7. Service Limits (approaching limits)

**Categories & Example Checks:**

**Cost Optimization:**
- Idle EC2 instances (low CPU over 14 days)
- Underutilized EBS volumes
- Unassociated Elastic IPs
- Idle load balancers
- Reserved Instance optimization (expiring, underutilized)
- Savings Plans recommendations
- Idle RDS instances

**Performance:**
- High utilization EC2 instances
- Overutilized EBS volumes
- CloudFront optimization (alternate domain names, content delivery)
- DNS resolution issues (Route 53 alias vs CNAME)

**Security:**
- Open security groups (unrestricted ports)
- IAM access key rotation (>90 days)
- MFA not enabled
- S3 bucket permissions/policies
- CloudTrail not enabled
- Exposed access keys (leaked on GitHub, etc.)
- Root account usage

**Fault Tolerance:**
- EBS snapshots age (no recent backups)
- EC2 AZ balance
- RDS Multi-AZ not enabled
- RDS backups not enabled
- Load balancer health checks
- Auto Scaling group health check
- VPN tunnel redundancy

**Service Limits:**
- Approaching limits for EC2, EBS, ELB, IAM, RDS, VPC, etc.
- Shows current usage vs limit
- Refresh available to update data

**Trusted Advisor Priority (Enterprise support):**
- Prioritized, context-aware recommendations
- Aggregates across accounts (organization-wide)
- Risk-scored recommendations
- Integrates with AWS Organizations

**Key Facts:**
- Check results: Green (OK), Yellow (investigation recommended), Red (action recommended)
- Data refresh: Manual or automatic (24-hour minimum between manual refreshes for most checks)
- **Notifications**: Weekly email digest (Business/Enterprise)
- **EventBridge integration**: Trusted Advisor check status change events
- **Organizational view**: Requires Business/Enterprise support on all accounts

---

### AWS Compute Optimizer

**Overview:**
- **ML-based service** that analyzes workload patterns and recommends optimal compute resources
- Analyzes: EC2 instances, EBS volumes, Lambda functions, ECS on Fargate, Auto Scaling groups
- Uses **14 days of CloudWatch metrics** (minimum) for recommendations
- Free for basic recommendations; paid for enhanced recommendations

**Supported Resources:**

| Resource | What It Recommends |
|----------|-------------------|
| **EC2 Instances** | Instance type (family, size), over/under-provisioned |
| **Auto Scaling Groups** | Instance type + desired capacity |
| **EBS Volumes** | Volume type (gp2→gp3, io1→io2), size, IOPS, throughput |
| **Lambda Functions** | Memory size (correlates with CPU/network allocation) |
| **ECS on Fargate** | Task size (CPU and memory) |
| **License recommendations** | Third-party license optimization |

**How It Works:**
```
CloudWatch Metrics (14+ days) → Compute Optimizer ML Model → Recommendations
  - EC2: CPU, memory (requires CW Agent), network, disk
  - EBS: IOPS, throughput, volume size utilization
  - Lambda: Duration, memory utilization, invocation count
  - ECS Fargate: CPU and memory utilization per task
```

**Recommendation Types:**
| Finding | Meaning |
|---------|---------|
| **Under-provisioned** | Resource needs MORE capacity (performance risk) |
| **Over-provisioned** | Resource has EXCESS capacity (cost waste) |
| **Optimized** | Current configuration is appropriate |
| **Not optimized** | Resource is not optimal (general recommendation) |

**Enhanced Infrastructure Metrics (Paid):**
- Extends analysis from 14 days to **93 days** (3 months)
- Better captures weekly/monthly patterns
- $0.0003360 per resource per hour (~$0.25/resource/month)
- Opt-in at account or organization level

**Key Features:**
- **Savings estimation**: Shows potential monthly savings per recommendation
- **Performance risk**: Rates risk of under-provisioning (1=low risk to 5=high risk)
- **Migration effort**: Low/Medium/High effort rating
- **External metrics**: Integrates with Datadog, Dynatrace, Instana, New Relic for enhanced recs
- **S3 export**: Export recommendations to S3 for analysis
- **Organization-wide**: Opt-in at org level, view recommendations across accounts

**Requirements:**
- CloudWatch metrics for 14+ days (30+ days recommended)
- For memory-aware EC2 recommendations: CloudWatch Agent publishing memory metrics
- IAM permissions: `compute-optimizer:*` + `cloudwatch:GetMetricData`
- Opt-in required (not enabled by default)

---

### Trusted Advisor vs Compute Optimizer

| Aspect | Trusted Advisor | Compute Optimizer |
|--------|----------------|-------------------|
| **Scope** | Broad (cost, security, performance, fault tolerance, limits) | Narrow (compute right-sizing only) |
| **Depth** | Shallow (basic checks, thresholds) | Deep (ML-based, workload-pattern-aware) |
| **EC2 recommendations** | "Instance is idle" (low CPU 14 days) | "Change from m5.2xlarge to m5.xlarge" (specific) |
| **EBS recommendations** | "Volume is underutilized" | "Change from gp2 to gp3, reduce to 100GB" |
| **Lambda** | No Lambda recommendations | Memory optimization recommendations |
| **Pricing** | Included with support plan | Free (basic) / paid (enhanced) |
| **Data source** | CloudWatch (limited metrics) | CloudWatch (comprehensive) + optional CW Agent |
| **Update frequency** | Manual refresh or 24-hour auto | Continuous (updated as metrics accumulate) |

---

## 2. Design Patterns & Best Practices

### When to Use

| Tool | Use When | Don't Use When |
|------|----------|----------------|
| Trusted Advisor | Broad environment health check, security posture, service limits | Detailed compute right-sizing (use Compute Optimizer) |
| Compute Optimizer | Right-sizing EC2, EBS, Lambda, ECS Fargate, ASG | Non-compute optimization (use Cost Explorer for overall cost) |
| Cost Explorer | Cost analysis, spending trends, forecasting | Instance-level right-sizing (use Compute Optimizer) |
| AWS Budgets | Budget alerts and actions | Recommendations (use TA/CO) |

### Anti-Patterns
- **Don't use Trusted Advisor as your only security tool** — it's a subset of checks; use Security Hub, GuardDuty, Inspector for comprehensive security
- **Don't blindly apply Compute Optimizer recommendations** — verify with application team; ML doesn't know about batch jobs or seasonal patterns
- **Don't rely on Trusted Advisor for real-time monitoring** — it's a periodic check, not continuous monitoring
- **Don't use Compute Optimizer without CloudWatch Agent** — without memory metrics, recommendations may suggest downsizing instances that are actually memory-constrained

### Architectural Patterns

**Cost Optimization Program:**
```
Trusted Advisor (identify idle/underutilized) 
  + Compute Optimizer (specific right-sizing recommendations)
  + Cost Explorer (spending trends and forecasts)
  + AWS Budgets (alerting on overspend)
  = Comprehensive cost management
```

**Continuous Right-Sizing Pipeline:**
```
1. Compute Optimizer identifies over-provisioned instances
2. Export recommendations to S3
3. Lambda processes recommendations → creates Jira tickets
4. Engineering team validates → approves change
5. Systems Manager Automation → resize instance (stop, modify, start)
6. Verify performance (CloudWatch metrics)
```

**Organization-Wide Optimization:**
```
Management Account:
  └── Trusted Advisor Organizational View
       └── Aggregate recommendations across all accounts
  └── Compute Optimizer (org-wide opt-in)
       └── Export all recommendations to central S3 bucket
       └── Athena queries for fleet-wide analysis
```

### Well-Architected Alignment

| Pillar | Trusted Advisor | Compute Optimizer |
|--------|----------------|-------------------|
| Reliability | Fault Tolerance checks (backups, Multi-AZ, redundancy) | Detect under-provisioned instances (performance risk) |
| Security | Security checks (open ports, MFA, key rotation) | N/A (not security-focused) |
| Cost | Idle resources, RI utilization, unassociated EIPs | Specific right-sizing (largest savings potential) |
| Performance | Over-utilized resources, CloudFront optimization | Under-provisioned detection + instance type recommendations |
| Ops Excellence | Service Limits (prevent hitting quotas) | Automated recommendation export + automation |

---

## 3. Security & Compliance

### IAM & Access Control

**Trusted Advisor:**
- `trustedadvisor:Describe*` — view check results
- `trustedadvisor:Refresh*` — refresh check data
- `support:*` — full Support API access (includes Trusted Advisor)
- **Organizational view**: Requires `organizations:*` + trusted advisor permissions in management account

**Compute Optimizer:**
- `compute-optimizer:GetRecommendations` — view recommendations
- `compute-optimizer:GetEnrollmentStatus` — check opt-in status
- `compute-optimizer:UpdateEnrollmentStatus` — opt in/out
- `compute-optimizer:ExportAutoScalingGroupRecommendations` — export to S3
- Cross-account: `compute-optimizer:GetRecommendationPreferences`

### Data Security
- Trusted Advisor: No customer data stored (reads current configuration)
- Compute Optimizer: Processes CloudWatch metrics (retained in Compute Optimizer for 14-93 days)
- S3 exports: Encrypted with SSE-S3 or SSE-KMS
- All API calls: TLS in transit

### Auditing
- **CloudTrail**: Logs Trusted Advisor API calls (via Support API) and Compute Optimizer calls
- **EventBridge**: Trusted Advisor check item refresh events

---

## 4. Cost Optimization

### Pricing

**Trusted Advisor:**
- **Free**: 7 core security checks + service limits (all support plans)
- **Full access**: Included with Business ($100+/month) or Enterprise ($15,000+/month) support plan
- No per-check charge — included in support plan

**Compute Optimizer:**
- **Free tier**: Basic recommendations (14-day lookback)
- **Enhanced recommendations**: $0.0003360/resource/hour (~$0.25/resource/month)
  - EC2, ASG, EBS, Lambda, ECS Fargate
- **External metrics integration**: Additional cost

### Cost-Saving Strategies Using These Tools

**Trusted Advisor savings (common findings):**
| Check | Typical Savings |
|-------|----------------|
| Idle EC2 instances | $50-$500/instance/month |
| Unassociated Elastic IPs | $3.60/IP/month |
| Idle load balancers | $16+/ALB/month |
| Underutilized EBS volumes | $10-$100/volume/month |
| Expiring Reserved Instances | Varies (avoid going back to On-Demand) |

**Compute Optimizer savings (common findings):**
| Resource | Typical Savings |
|----------|----------------|
| Over-provisioned EC2 | 20-40% per instance (right-size) |
| Over-provisioned EBS | 10-50% (gp2→gp3, reduce IOPS) |
| Over-provisioned Lambda | 10-30% (reduce memory allocation) |
| Over-provisioned ECS Fargate | 15-35% (reduce task CPU/memory) |

### Cost Traps
- **Business/Enterprise support plan cost** — Trusted Advisor full access requires expensive support plan
- **Compute Optimizer enhanced metrics** — charges per resource; large fleets can accumulate cost
- **Acting on outdated recommendations** — resource may have changed since last analysis (verify before acting)
- **Ignoring Trusted Advisor RI recommendations** — letting RIs expire and reverting to On-Demand pricing

---

## 5. High Availability, Disaster Recovery & Resilience

### Trusted Advisor for HA

**Fault Tolerance Checks:**
- EBS snapshots: Are backups being taken?
- RDS Multi-AZ: Is it enabled?
- RDS backups: Is automated backup enabled?
- EC2 AZ balance: Are instances distributed across AZs?
- ELB cross-zone load balancing: Is it enabled?
- VPN tunnel redundancy: Are both tunnels up?
- Auto Scaling health check type: EC2 vs ELB
- Route 53 health checks: Are they configured?

**Service Limits for Resilience:**
- If you hit a service limit during a scaling event → availability impact
- Trusted Advisor warns when approaching limits (80%+ utilization)
- Action: Request limit increase before you need it

### Compute Optimizer for Resilience
- **Under-provisioned instances**: Performance risk score helps identify potential bottlenecks
- If instance is under-provisioned → may fail under load → availability impact
- Recommendations include performance risk rating (1-5)
- Helps ensure instances can handle peak load

---

## 6. Exam-Focused Section

### Straightforward Questions (7 examples)

**Q1:** A company wants to identify EC2 instances that are consistently under-utilized and get specific recommendations for right-sizing. What service should they use?
**A:** **AWS Compute Optimizer** — uses ML to analyze CloudWatch metrics and recommends specific instance types (e.g., "change from m5.2xlarge to m5.large"). More detailed than Trusted Advisor's basic "idle instance" check.

**Q2:** A company on the Basic support plan wants to check if their root account has MFA enabled. Can they use Trusted Advisor?
**A:** **Yes** — MFA on Root Account is one of the 7 core security checks available to ALL support plans (including Basic/Developer).

**Q3:** A company wants to be alerted before they hit their EC2 instance limit during peak scaling. What should they use?
**A:** **Trusted Advisor** Service Limits check — it shows current usage vs. limit and warns at 80%+. Combine with EventBridge for automated alerting. Also consider **Service Quotas** service for direct limit monitoring and automatic increase requests.

**Q4:** A Lambda function has 512 MB allocated but CloudWatch shows it uses only 128 MB. What tool recommends the optimal memory size?
**A:** **Compute Optimizer** — analyzes Lambda function memory utilization, duration, and cost to recommend optimal memory allocation (e.g., reduce to 256 MB for cost savings, or increase for faster execution).

**Q5:** What is the minimum amount of CloudWatch metric data needed for Compute Optimizer to generate recommendations?
**A:** **14 days** (minimum). 30+ days is recommended for better accuracy. Enhanced mode extends lookback to 93 days.

**Q6:** A company wants a weekly email summarizing Trusted Advisor recommendations across all 5 categories. What's required?
**A:** **Business or Enterprise support plan** — weekly notification emails are only available with these plans. Configure notification preferences in the Trusted Advisor console.

**Q7:** A company uses Compute Optimizer for EC2 recommendations but notices the recommendations don't account for memory usage (all instances recommended to downsize). Why?
**A:** Compute Optimizer needs **CloudWatch Agent** publishing memory metrics. Without it, only CPU/network/disk metrics are analyzed. Install the CloudWatch Agent to publish memory utilization for memory-aware recommendations.

---

### Tricky / Scenario-Based Questions (7 examples)

**Q1:** A company has 500 EC2 instances. Trusted Advisor shows 50 as "idle" (low CPU for 14 days). The team investigates and finds these are batch processing servers that run heavily for 2 days per month. Should they follow the recommendation?

**A:** **No** — Trusted Advisor uses a simple threshold (CPU < 10% for 14 days). It doesn't understand monthly batch patterns. **Compute Optimizer with Enhanced Infrastructure Metrics** (93-day lookback) would better capture the monthly pattern and provide more accurate recommendations. The team should right-size for peak batch load, possibly using Scheduled Scaling or Spot instances during batch windows instead.

**Why wrong answers are wrong:**
- "Terminate the instances" — would lose batch processing capability
- "Trust Trusted Advisor completely" — it doesn't understand usage patterns beyond its simple threshold
- "Switch to Lambda" — batch processing may need long-running compute, not suitable for Lambda (15-min limit)
- **Keywords:** "batch processing", "runs 2 days/month", "shows idle" → Trusted Advisor too simplistic; use Compute Optimizer enhanced

---

**Q2:** A company wants to implement automated right-sizing: when Compute Optimizer identifies an over-provisioned instance, it should automatically be resized during a maintenance window. How to architect this?

**A:** (1) **Compute Optimizer** exports recommendations to **S3**. (2) **EventBridge scheduled rule** triggers a **Lambda function** weekly. (3) Lambda reads S3 recommendations, identifies over-provisioned instances. (4) Lambda creates **Systems Manager Automation** execution (stop → modify instance type → start) targeted to a **Maintenance Window**. (5) CloudWatch alarm verifies performance post-resize.

**Why wrong answers are wrong:**
- "Use Trusted Advisor API" — TA doesn't provide specific resize recommendations
- "Use Auto Scaling" — ASG manages count, not individual instance types (unless using mixed instances)
- "Resize immediately without window" — causes downtime (instance must stop to change type)
- **Keywords:** "automated right-sizing", "maintenance window" → Compute Optimizer → S3 → Lambda → SSM Automation

---

**Q3:** A company has an Enterprise support plan. Their CTO wants a single dashboard showing Trusted Advisor findings across all 200 member accounts. How?

**A:** Enable **Trusted Advisor Organizational View** (available with Enterprise support). It aggregates Trusted Advisor checks across all accounts in the organization. Accessible from the management account or delegated administrator. Can also use **Trusted Advisor Priority** for risk-prioritized recommendations.

**Why wrong answers are wrong:**
- "Create a custom dashboard aggregating per-account APIs" — unnecessary; org view exists natively
- "Use Config Aggregator" — aggregates Config compliance, not Trusted Advisor checks
- "Run Trusted Advisor in each account and export to S3" — works but org view is built-in and simpler
- **Keywords:** "all 200 accounts", "single dashboard", "Enterprise support" → Trusted Advisor Organizational View

---

**Q4:** A company uses Compute Optimizer for EBS volume recommendations. It recommends changing from gp2 to gp3 for 100 volumes. The team is concerned about performance impact. Is this safe?

**A:** **Generally yes** — gp3 provides the same baseline performance as gp2 (3,000 IOPS, 125 MB/s throughput) at lower cost, with the ability to independently scale IOPS and throughput. However: (1) Verify the volume doesn't need >3,000 baseline IOPS (gp2 scales IOPS with size). (2) If using a gp2 volume >1 TB (which gets >3,000 IOPS from size-based scaling), ensure you provision equivalent IOPS on gp3. (3) EBS modification is **online** — no downtime required.

**Why this is important:**
- gp2 volumes >1 TB get more than 3,000 IOPS (3 IOPS/GB). gp3 starts at 3,000 regardless of size — you must explicitly provision more.
- **Keywords:** "gp2 to gp3", "performance concern" → Safe for most volumes, but verify large gp2 volumes (>1 TB) aren't relying on size-based IOPS scaling

---

**Q5:** A company's Compute Optimizer shows an instance as "under-provisioned" with performance risk 4 (out of 5). But the application team says performance is fine. Who is right?

**A:** **Both could be right.** Compute Optimizer bases recommendations on CPU/memory/network utilization metrics. High utilization (>80%) indicates performance risk statistically, but the application may be designed to run hot (e.g., fully utilizing CPU is expected for compute-intensive workloads). The application team's assessment of actual user experience matters. However, the high risk score means there's **no headroom for traffic spikes** — if load increases, performance will degrade. Consider: keeping current size but adding auto-scaling for spike protection.

**Keywords:** "under-provisioned but performance fine" → Verify with application team; consider headroom for spikes

---

**Q6:** A company has Basic support and wants Trusted Advisor-level checks for cost optimization without upgrading to Business support. What alternatives exist?

**A:** (1) **AWS Cost Explorer** — has right-sizing recommendations (similar to TA idle instance checks). (2) **Compute Optimizer** — free tier provides right-sizing recommendations. (3) **AWS Budgets** — cost alerts and actions. (4) **Cost Anomaly Detection** — detects unusual spending. (5) **Third-party tools** (CloudHealth, Spot.io). These combined provide cost optimization insights without Business support.

**Why wrong answers are wrong:**
- "There's no alternative" — multiple services overlap with TA cost checks
- "Use Config rules" — Config evaluates compliance, not cost optimization
- "Use CloudWatch alarms" — can alert on utilization but doesn't provide recommendations
- **Keywords:** "Basic support", "cost optimization without upgrading" → Cost Explorer right-sizing + Compute Optimizer (both free)

---

**Q7:** A company enables Compute Optimizer across their organization (100 accounts). They want to identify the top 20 instances with the highest savings potential across all accounts. How?

**A:** (1) Enable **organization-wide Compute Optimizer** opt-in from management account. (2) Use **Export recommendations** to S3 (supports cross-account export). (3) Query the exported data with **Athena** — sort by estimated monthly savings, limit to top 20. This gives fleet-wide visibility with cost prioritization.

**Why wrong answers are wrong:**
- "Use Compute Optimizer console" — shows per-account, not easy to compare across 100 accounts
- "Use Trusted Advisor" — doesn't provide specific savings estimates per instance
- "Use Cost Explorer" — has right-sizing recs but less detailed than Compute Optimizer
- **Keywords:** "top 20 highest savings", "across all accounts" → Export to S3 + Athena query

---

### Common Exam Traps & Pitfalls

1. **Trusted Advisor full access requires Business/Enterprise support** — Basic/Developer only gets 7 core security checks + service limits. This is heavily tested.

2. **Compute Optimizer ≠ Trusted Advisor**: CO is deep ML-based right-sizing. TA is broad but shallow. They're complementary, not competitive.

3. **Compute Optimizer needs 14+ days of data** — won't generate recommendations for new instances.

4. **Compute Optimizer needs CloudWatch Agent for memory** — without it, recommendations ignore memory (may suggest downsizing memory-heavy instances).

5. **Trusted Advisor "idle" = simple threshold** — doesn't understand batch, scheduled, or seasonal workloads.

6. **Service Limits ≠ Service Quotas**: Trusted Advisor shows limits (passive). Service Quotas service allows you to REQUEST increases and set alarms.

7. **Compute Optimizer is opt-in** — not enabled by default. Must opt in per account or organization.

8. **Trusted Advisor recommendations are not enforced** — it's advisory only. No auto-fix (unlike Config remediation).

9. **gp2 → gp3 migration**: gp3 is almost always cheaper but large gp2 volumes may have more baseline IOPS (check before migrating).

10. **Compute Optimizer supports ECS Fargate** but NOT ECS on EC2 (only the Fargate tasks are analyzed, not the underlying EC2 instances in ECS EC2 mode — use EC2 recommendations for those).

---

## 7. Cheat Sheet

### Must-Know Facts

| Fact | Detail |
|------|--------|
| TA core checks (all plans) | 7 security checks + service limits |
| TA full access | Business/Enterprise support only |
| TA categories | Cost, Performance, Security, Fault Tolerance, Service Limits |
| TA check result colors | Green (OK), Yellow (investigate), Red (action needed) |
| CO minimum data | 14 days CloudWatch metrics |
| CO enhanced lookback | 93 days ($0.25/resource/month) |
| CO supported resources | EC2, ASG, EBS, Lambda, ECS Fargate |
| CO memory-aware | Requires CloudWatch Agent |
| CO opt-in | Required (not auto-enabled) |
| CO recommendations | Under-provisioned, Over-provisioned, Optimized |
| Service Quotas | View, request increases, set alarms on limits |
| TA org view | Enterprise support required |

### Decision Flowchart

```
"Broad environment health check (security, cost, limits)"
  → Trusted Advisor

"Specific right-sizing recommendation for EC2/EBS/Lambda"
  → Compute Optimizer

"Am I approaching a service limit?"
  → Trusted Advisor Service Limits check + Service Quotas

"What instance type should I use?"
  → Compute Optimizer (specific instance type + size recommendation)

"Which resources are idle/unused?"
  → Trusted Advisor (broad) + Cost Explorer (cost-focused)

"Optimize Lambda memory allocation"
  → Compute Optimizer (or Lambda Power Tuning open-source tool)

"Organization-wide cost/security recommendations"
  → Trusted Advisor Organizational View (Enterprise) + CO org opt-in

"Compliance framework evaluation"
  → NOT TA or CO — use AWS Config + Security Hub

"Real-time security monitoring"
  → NOT TA — use GuardDuty + Security Hub

"Right-size ECS Fargate tasks"
  → Compute Optimizer
```

---

## 8. Additional Deep-Dive

### 10 More Tricky Scenario-Based Questions

**Q1:** A company enables Compute Optimizer and sees recommendations to downsize their RDS instances. Wait — does Compute Optimizer support RDS?

**A:** **No** — Compute Optimizer does NOT support RDS (as of current). It only covers EC2, ASG, EBS, Lambda, and ECS Fargate. For RDS right-sizing, use **Trusted Advisor** (idle RDS check) or **CloudWatch metrics + manual analysis** or **AWS Cost Explorer's right-sizing recommendations** (which include some RDS recommendations).

**Keywords:** "right-size RDS" → NOT Compute Optimizer (it doesn't support RDS). Use TA/Cost Explorer.

---

**Q2:** A company uses Trusted Advisor and sees "Security Groups — Unrestricted Access" flagged as RED for a security group allowing 0.0.0.0/0 on port 443. The team says this is intentional (public web server). Should they fix it?

**A:** **No** — port 443 (HTTPS) on a public-facing ALB/web server is expected to be open to the world. Trusted Advisor flags ALL unrestricted security groups, regardless of whether the port is legitimately public. The team should **suppress or acknowledge** the finding. This is a known false positive for public-facing services.

**Keywords:** "TA flags public port 443", "intentional public web server" → Legitimate configuration; suppress/acknowledge the finding

---

**Q3:** A company wants Compute Optimizer recommendations that account for their seasonal Black Friday traffic (November spike). Standard 14-day lookback misses this pattern. What should they do?

**A:** Enable **Enhanced Infrastructure Metrics** (93-day lookback). If Black Friday was within the last 93 days, it will be captured. If not, the recommendations won't include that pattern. Additionally, they can: (1) Wait until the seasonal data is within the lookback window. (2) Complement CO recommendations with manual analysis of historical CloudWatch metrics. (3) Don't downsize below what's needed for the seasonal peak (use CO as one input, not the sole decision maker).

**Keywords:** "seasonal spike", "standard lookback misses it" → Enhanced Infrastructure Metrics (93-day lookback)

---

**Q4:** A company's Trusted Advisor shows "Amazon EBS Under-utilized Volumes" for volumes with < 1 IOPS per day. But these are backup volumes attached to stopped instances. Should they delete them?

**A:** **Investigate before deleting.** If the instances are intentionally stopped (e.g., dev environments turned off nightly), the volumes are needed when instances restart. Better approach: (1) **Snapshot** the volumes and delete the EBS volumes if instances won't be used for a long time. (2) Use **Warm Pools** or **AMIs** instead of keeping stopped instances with attached volumes. (3) If volumes are genuinely orphaned (instance terminated), delete them.

**Keywords:** "under-utilized EBS", "attached to stopped instances" → Don't blindly delete; evaluate if needed when instances restart

---

**Q5:** A company uses both Trusted Advisor and Compute Optimizer for EC2 right-sizing. They give conflicting recommendations: TA says "instance is idle" (CPU <2%), Compute Optimizer says "over-provisioned, recommend m5.large" (not terminate). Which should they trust?

**A:** **Compute Optimizer** is more nuanced. "Idle" in TA means <2% CPU for 14 days (binary: idle or not). CO analyzes the workload pattern and recommends an appropriate size (it accounts for ALL metrics, not just average CPU). The instance likely has some minimal usage (justifying smaller instance rather than termination). Validate with the application team whether the workload is legitimate.

**Keywords:** "conflicting recommendations from TA and CO" → Trust Compute Optimizer (more detailed ML analysis)

---

**Q6:** A company is hitting their EC2 instance service limit during auto-scaling events, causing scaling failures. Trusted Advisor showed this was approaching the limit last week, but no one acted. How to automate this?

**A:** (1) Use **Service Quotas** service to set a **CloudWatch alarm** when usage approaches the limit (e.g., 80%). (2) Create an **EventBridge rule** on the Service Quotas alarm → trigger **Lambda** → auto-request a limit increase via Service Quotas API. (3) Alternatively, use Trusted Advisor's service limit check with EventBridge integration → SNS alert to the team. The key is automation: don't rely on humans checking Trusted Advisor manually.

**Keywords:** "hitting service limit", "no one acted on TA warning" → Service Quotas CloudWatch alarm + automated increase request

---

**Q7:** A company wants to save money on their Lambda functions. Compute Optimizer recommends reducing memory from 1024 MB to 256 MB for a function. But the function makes API calls to external services and the team is worried about performance. What should they consider?

**A:** In Lambda, **CPU and network scale linearly with memory allocation**. Reducing memory from 1024 MB to 256 MB means getting only 25% of the CPU power and network bandwidth. If the function makes external API calls (network-bound), lower memory = less network throughput = potentially longer duration. Longer duration may **increase cost** (billed per ms × memory). The team should use **AWS Lambda Power Tuning** (open-source tool) to test different memory settings and find the cost-optimal point (not necessarily the lowest memory).

**Keywords:** "Lambda memory reduction", "makes API calls" → Lower memory = less CPU/network; may increase duration and cost. Test with Power Tuning.

---

**Q8:** A company has Trusted Advisor showing 200 unassociated Elastic IPs ($720/month waste). The team says they're "reserved for future use." What's the right approach?

**A:** Elastic IPs cost $0.005/hour ($3.60/month) when NOT associated with a running instance. If they're truly not needed now: (1) **Release them** — you can always allocate new ones later (they're not like static IPs you need to keep). (2) If specific IPs are whitelisted by partners/customers: Keep those but release the rest. (3) Implement a tagging policy: tag EIPs with `Purpose` and `Owner` to justify each allocation. $720/month for unused EIPs is pure waste in most cases.

**Keywords:** "unassociated Elastic IPs", "reserved for future" → Release unused EIPs (can allocate new ones anytime)

---

**Q9:** A company wants to use Compute Optimizer for their ECS Fargate workloads but sees no recommendations. What might be wrong?

**A:** Possible causes: (1) **Opt-in not done** — Compute Optimizer must be explicitly opted in. (2) **Less than 14 days of data** — new tasks won't have recommendations yet. (3) **Fargate tasks not using the right platform version** — check supported platform versions. (4) **CloudWatch Container Insights not enabled** — Compute Optimizer may need Container Insights for ECS metric collection. (5) **Region not supported** — verify Compute Optimizer ECS support in the specific region.

**Keywords:** "no Fargate recommendations" → Check opt-in, data age (14+ days), and Container Insights

---

**Q10:** A company's Trusted Advisor shows 5 exposed access keys (found on public GitHub repositories). This is critical. What's the immediate response?

**A:** **Immediate actions:** (1) **Deactivate the exposed access keys** immediately (IAM → Users → Security Credentials → Deactivate). (2) Check **CloudTrail** for any unauthorized API calls made with those keys. (3) **Rotate all affected credentials** (create new keys, update applications, delete old keys). (4) Review and revoke any actions taken by the compromised keys. (5) Enable **GuardDuty** (detects credential use from unusual locations). (6) Implement **IAM policies** preventing access key creation (use roles instead). (7) Set up **AWS Config rules** and **IAM Access Analyzer** for ongoing monitoring.

**Keywords:** "exposed access keys", "GitHub" → Immediately deactivate + audit CloudTrail + rotate + remediate

---

### Compare: Trusted Advisor vs Compute Optimizer vs Cost Explorer vs Service Quotas

| Aspect | Trusted Advisor | Compute Optimizer | Cost Explorer | Service Quotas |
|--------|----------------|-------------------|---------------|----------------|
| **Purpose** | Broad best-practice checks | ML right-sizing | Cost analysis/forecasting | Limit management |
| **Scope** | Cost, Security, Perf, FT, Limits | EC2, EBS, Lambda, ECS, ASG | All AWS spending | All service limits |
| **Depth** | Shallow (thresholds) | Deep (ML patterns) | Medium (spending patterns) | Specific (quotas) |
| **Output** | Green/Yellow/Red status | Instance type recommendation | Dollar amounts, trends | Usage vs limit % |
| **Pricing** | Business/Enterprise plan | Free / $0.25/resource/month | Free | Free |
| **Exam Tip** | "Broad health check", "security posture" | "Right-size EC2/Lambda" | "Spending trends", "forecast" | "Approaching limit", "request increase" |

### Compare: Cost Optimization Tools

| Tool | Best For | Exam Keyword |
|------|----------|-------------|
| Trusted Advisor | Idle/unused resources | "Idle instance", "unassociated EIP" |
| Compute Optimizer | Right-sizing (specific instance type) | "Over-provisioned", "optimal instance type" |
| Cost Explorer | Spending visibility, RI recommendations | "Cost trends", "forecast spending" |
| AWS Budgets | Spending alerts and enforcement | "Alert when spending exceeds $X" |
| Cost Anomaly Detection | Unexpected spending spikes | "Unusual cost increase" |
| Savings Plans | Commitment-based discounts | "Flexible savings", "compute commitment" |

### Decision Tree: Which Tool to Use

```
"Is my environment following best practices?" (broad)
  └── Trusted Advisor

"What instance type should this EC2 run on?" (specific)
  └── Compute Optimizer

"How much am I spending and where?" (financial)
  └── Cost Explorer

"Alert me if spending exceeds $5,000/month"
  └── AWS Budgets

"Why did my bill spike this month?"
  └── Cost Anomaly Detection

"Am I about to hit a service limit?"
  └── Service Quotas (precise) + Trusted Advisor (broader check)

"Is my Lambda over-allocated?"
  └── Compute Optimizer (or Lambda Power Tuning for testing)

"Are my S3 buckets publicly accessible?"
  └── Trusted Advisor (basic) + Config Rules (comprehensive) + IAM Access Analyzer (deep)

"Should I buy Reserved Instances?"
  └── Cost Explorer RI Recommendations

"What's the optimal EBS volume type?"
  └── Compute Optimizer
```
