# Auto Scaling & Elasticity Patterns

## 1. Core Concepts & Theory

### Elasticity vs Scalability
- **Scalability**: Ability to handle increased load by adding resources (vertical or horizontal)
- **Elasticity**: Ability to automatically scale OUT and IN based on demand (add AND remove resources)
- **Vertical Scaling (Scale Up/Down)**: Change instance size (e.g., t3.medium → t3.xlarge) — requires downtime for EC2
- **Horizontal Scaling (Scale Out/In)**: Add/remove instances — no downtime, preferred for elasticity

### Amazon EC2 Auto Scaling

**Components:**
- **Launch Template** (preferred) / Launch Configuration (legacy): Defines WHAT to launch (AMI, instance type, key pair, security groups, user data)
- **Auto Scaling Group (ASG)**: Defines WHERE and HOW MANY (VPC, subnets, min/max/desired capacity)
- **Scaling Policy**: Defines WHEN to scale (conditions that trigger scaling)
- **Cooldown Period**: Wait time after a scaling activity before another can start (default 300s)
- **Warm-up Period**: Time for a new instance to start contributing to metrics (used with step/target tracking)

**Launch Template vs Launch Configuration:**
| Feature | Launch Template | Launch Configuration |
|---------|----------------|---------------------|
| Versioning | Yes (multiple versions) | No (immutable) |
| Mixed instances | Yes | No |
| Spot + On-Demand | Yes | Limited |
| T2/T3 Unlimited | Yes | No |
| Multiple instance types | Yes | Single type only |
| AWS recommendation | Yes (preferred) | Deprecated (no new features) |

**ASG Key Settings:**
- **Minimum capacity**: Floor (never fewer than this)
- **Maximum capacity**: Ceiling (never more than this)
- **Desired capacity**: Target count (ASG maintains this unless scaling policy changes it)
- **Health check type**: EC2 (default) or ELB (recommended with load balancer)
- **Health check grace period**: Time before health checks start (default 300s) — prevents premature termination of initializing instances
- **Default termination policy**: Oldest launch config → closest to billing hour → random (within AZ with most instances)

**Scaling Policy Types:**

| Policy Type | How It Works | Best For |
|-------------|--------------|----------|
| **Target Tracking** | Maintain a metric at a target value (like a thermostat) | Simplest, most common (e.g., "keep CPU at 50%") |
| **Step Scaling** | Scale by different amounts based on alarm threshold bands | Variable scaling amounts based on severity |
| **Simple Scaling** | Scale by fixed amount when alarm triggers, then cooldown | Legacy, avoid for new designs |
| **Scheduled Scaling** | Scale at specific date/time | Predictable load patterns (business hours) |
| **Predictive Scaling** | ML-based forecast from historical patterns, pre-provisions capacity | Recurring cyclical patterns |

**Target Tracking Scaling:**
- Specify metric + target value (e.g., ASGAverageCPUUtilization = 50%)
- ASG creates and manages CloudWatch alarms automatically
- Built-in metrics: CPU, Network In/Out, ALB Request Count Per Target
- Custom metrics supported (via CloudWatch)
- **Scale-in protection**: Can disable scale-in (only scales out) — useful with multiple policies
- **Warm-up time**: New instances excluded from metric calculation during warm-up

**Step Scaling:**
```
CPU 50-70% → Add 1 instance
CPU 70-90% → Add 2 instances
CPU > 90%  → Add 3 instances
```
- Multiple step adjustments based on alarm breach size
- No cooldown (uses warm-up period instead)
- More granular than target tracking for complex scenarios

**Predictive Scaling:**
- Uses ML to analyze historical patterns (daily/weekly cycles)
- Pre-provisions capacity BEFORE demand arrives
- Scheduling mode: "Forecast only" (observe) or "Forecast and scale" (take action)
- Needs 24 hours of data minimum; 14 days for best accuracy
- Combined with dynamic scaling (predictive handles baseline, dynamic handles unexpected spikes)

**Instance Refresh:**
- Replace instances in ASG with updated launch template (rolling replacement)
- **Minimum healthy percentage**: How many instances must remain running during refresh (default 90%)
- **Instance warmup**: Time before a new instance is considered healthy
- **Checkpoint**: Pause at a percentage to validate before continuing
- Replaces: manual rolling deployments, custom scripts

**Warm Pools:**
- Pre-initialized instances kept in a "stopped" or "running" state
- Pulled into ASG much faster than launching new instances from scratch
- Lifecycle: Launch → Pre-warm → Warm Pool (Stopped/Running/Hibernated) → InService
- Reduces scale-out time from minutes to seconds
- **Use case**: Applications with long initialization (JVM warmup, large config downloads)

**Lifecycle Hooks:**
- Pause instance launch/termination for custom actions
- **Pending:Wait** → custom action (install software, register DNS) → Complete → InService
- **Terminating:Wait** → custom action (deregister, backup logs) → Complete → Terminated
- Timeout: 1 hour default (max 48 hours, extendable via heartbeat)
- Targets: CloudWatch Events, SNS, SQS, Lambda

**Key Limits:**
| Limit | Value |
|-------|-------|
| ASGs per region | 500 (soft) |
| Launch templates per region | 5,000 |
| Scaling policies per ASG | 50 |
| Scheduled actions per ASG | 125 |
| Lifecycle hooks per ASG | 50 |
| Max instances per ASG | 2,000 (soft, can increase to 5,000+) |
| Health check grace period max | 7,200 seconds (2 hours) |
| Cooldown default | 300 seconds |

### Application Auto Scaling

**Supports (NOT EC2 — that's EC2 Auto Scaling):**
- ECS Services (task count)
- DynamoDB (read/write capacity units)
- Aurora Replicas (replica count)
- EMR Clusters (instance count)
- AppStream 2.0 Fleets
- SageMaker Endpoints
- Custom Resources (via CloudWatch + API Gateway)
- Lambda Provisioned Concurrency
- Comprehend endpoints
- Neptune Clusters

**Same policy types:** Target Tracking, Step Scaling, Scheduled Scaling

### AWS Auto Scaling (Unified Console)
- **Single dashboard** for creating scaling plans across multiple services
- Uses **scaling plans** with recommendations
- Combines EC2 Auto Scaling + Application Auto Scaling
- Can optimize for availability, cost, or balanced
- Supports predictive scaling across services

### ECS Auto Scaling

**Service Auto Scaling (Task Level):**
- Target tracking on: CPU, Memory, ALB request count per target
- Step scaling with custom CloudWatch metrics
- Scheduled scaling for known patterns

**Cluster Auto Scaling (Capacity Provider):**
- **Capacity Providers**: Link ASG to ECS cluster
- **Managed Scaling**: ECS manages the ASG based on task placement needs
- `CapacityProviderReservation` metric: Target = 100% (means cluster capacity matches demand)
- **Managed termination protection**: Prevents scaling in instances with running tasks

**Fargate Auto Scaling:**
- Only task-level scaling (no infrastructure to manage)
- Target tracking on CPU/memory utilization or custom metrics
- Simplest auto scaling model (no capacity providers needed)

### Lambda Scaling

- **Concurrent executions** scale automatically (no ASG needed)
- **Reserved Concurrency**: Guarantees AND caps concurrency for a function
- **Provisioned Concurrency**: Pre-initialized execution environments (eliminates cold starts)
  - Can be auto-scaled using Application Auto Scaling (target tracking on utilization)
- **Account concurrency limit**: 1,000 (default, can increase to tens of thousands)
- **Burst limit**: 500-3000 depending on region (initial burst)
- Scale rate: 500 additional instances/minute after burst

### DynamoDB Auto Scaling
- Uses Application Auto Scaling
- Target tracking on consumed read/write capacity vs provisioned
- Target: 50-90% utilization (default 70%)
- Reacts in minutes — not suitable for sudden spikes (use on-demand for unpredictable)
- **On-Demand mode**: Instant scaling, no capacity planning — but 2.5x more expensive

---

## 2. Design Patterns & Best Practices

### When to Use Which Scaling Approach

| Scenario | Approach |
|----------|----------|
| Steady, predictable growth | Target Tracking |
| Known daily/weekly patterns | Scheduled + Predictive Scaling |
| Spiky, unpredictable load | Target Tracking + Predictive as baseline |
| Very spiky, millisecond response needed | Pre-warmed capacity (Warm Pools / Provisioned Concurrency) |
| Event-driven (concert tickets, flash sale) | Scheduled scaling (pre-provision) + Target Tracking |
| Cost-sensitive with variable load | Mixed instances (Spot + On-Demand) |
| Serverless workloads | Lambda auto-scaling (automatic) + Provisioned Concurrency for latency-sensitive |

### Anti-Patterns
- **Don't scale on CPU alone if the app is I/O bound** — use custom metrics (queue depth, request latency)
- **Don't use Simple Scaling** — Step or Target Tracking is always better (no cooldown gap)
- **Don't set health check grace period too low** — instances terminated before initialization completes
- **Don't use min = max = desired** — defeats the purpose of auto scaling (use for fixed-size fleets only)
- **Don't scale on 1-minute metrics with 1-period alarm** — too reactive; use 3+ periods to avoid flapping
- **Don't use Launch Configurations** — deprecated, use Launch Templates
- **Don't rely solely on reactive scaling for large applications** — add Predictive Scaling for baseline

### Architectural Patterns

**Multi-AZ Auto Scaling:**
- ASG spans multiple AZs (specify subnets in different AZs)
- ASG automatically balances instances across AZs
- If one AZ fails: ASG launches replacements in remaining AZs
- **AZ Rebalancing**: Launches new instance in underrepresented AZ, then terminates from over-represented

**Mixed Instances Policy (Spot + On-Demand):**
```
ASG:
  - Base capacity: 2 On-Demand (always running)
  - Above base: 70% Spot, 30% On-Demand
  - Instance types: [m5.large, m5a.large, m4.large, c5.large]  # diversified
  - Allocation strategy: capacity-optimized (fewer interruptions)
```
- **Capacity-optimized**: Chooses Spot pools with most available capacity (fewer interruptions)
- **Lowest-price**: Cheapest Spot pool (more interruptions)
- **Price-capacity-optimized**: Balance of lowest price AND available capacity (recommended)

**Blue/Green with ASG:**
- Two ASGs behind same ALB (different target groups)
- Scale up green ASG → shift traffic (weighted target groups) → scale down blue ASG
- Or: Route 53 weighted routing between two ALBs

**Queue-Based Scaling:**
```
SQS Queue → CloudWatch Metric (ApproximateNumberOfMessagesVisible)
         → Custom metric: Backlog per Instance = Messages / Running Instances
         → Target Tracking: Backlog per Instance = 0 (or acceptable target)
```
- Best pattern for decoupled, asynchronous workloads
- AWS recommendation: `BacklogPerInstance` as custom metric for target tracking
- Scales based on actual work, not CPU (which may be low during I/O waits)

**Scheduled + Dynamic Combination:**
```
Scheduled: Every weekday 8 AM → min=10, desired=10
Scheduled: Every weekday 6 PM → min=2, desired=2
Dynamic: Target Tracking CPU 60% (handles unexpected spikes within the window)
```

**Predictive + Dynamic Combination:**
```
Predictive: Handles daily/weekly pattern (pre-provisions 10 minutes before expected spike)
Dynamic: Target Tracking handles unexpected spikes beyond prediction
Result: No cold-start scaling lag for predicted demand + safety net for unpredicted
```

### Well-Architected Alignment

| Pillar | Practice |
|--------|----------|
| Reliability | Multi-AZ ASG, health checks (ELB), AZ rebalancing, lifecycle hooks for graceful shutdown |
| Security | Launch templates with IMDSv2 required, encrypted EBS, security groups, IAM instance profiles |
| Cost | Spot instances, right-sizing, scheduled scale-in, predictive scaling (avoid over-provisioning) |
| Performance | Warm Pools for fast scale-out, target tracking for optimal utilization, custom metrics for accurate scaling |
| Ops Excellence | Instance refresh for updates, lifecycle hooks for automation, predictive scaling reduces manual intervention |

### Integration Points
- **ELB (ALB/NLB)**: Health checks, target group registration/deregistration
- **CloudWatch**: Metrics for scaling policies, alarms
- **SNS**: Scaling notifications
- **SQS**: Queue-based scaling pattern
- **EventBridge**: Lifecycle hook events, scaling events
- **Systems Manager**: Parameter Store for config, Run Command for fleet management
- **CodeDeploy**: Deploy to ASG with auto-scaling integration
- **CloudFormation**: `UpdatePolicy` for rolling updates, `CreationPolicy` for cfn-signal
- **EC2 Spot**: Mixed instances policy, capacity-optimized allocation

---

## 3. Security & Compliance

### IAM & Access Control

**ASG Service-Linked Role** (`AWSServiceRoleForAutoScaling`):
- Automatically created
- Allows ASG to: describe/launch/terminate instances, manage ELB target groups, manage lifecycle hooks
- Cannot be modified (service-linked)

**Custom Service Role:**
- Needed if ASG must use customer-managed CMK for EBS encryption
- Allows KMS:CreateGrant on the specific CMK

**Launch Template Security:**
- Specify IAM instance profile (role for EC2 instances)
- Require IMDSv2 (metadata service hardening): `HttpTokens: required`
- Specify security groups (least privilege)
- Encrypted EBS volumes (specify KMS key)
- No SSH keys in production (use SSM Session Manager instead)

**Scaling Policy Permissions:**
- `autoscaling:PutScalingPolicy` — create/update scaling policies
- `autoscaling:SetDesiredCapacity` — manual scaling
- `autoscaling:UpdateAutoScalingGroup` — change min/max/desired
- `cloudwatch:PutMetricAlarm` — target tracking creates alarms automatically

**SCP Considerations:**
- Prevent modification of production ASG min/max (`autoscaling:UpdateAutoScalingGroup` deny on specific ASG)
- Enforce minimum instance count for compliance
- Require specific launch template versions

### Encryption
- **EBS volumes**: Specify KMS key in launch template (all instances get encrypted volumes)
- **Default encryption**: Enable account-level EBS default encryption (all new volumes encrypted)
- **User data**: Can contain secrets (visible in metadata) — use SSM Parameter Store references instead
- **Instance metadata**: Require IMDSv2 in launch template to prevent SSRF attacks

### Auditing
- **CloudTrail**: All Auto Scaling API calls (CreateAutoScalingGroup, SetDesiredCapacity, etc.)
- **ASG Activity History**: Records all scaling activities (launch, terminate, failed)
- **CloudWatch Events/EventBridge**: Real-time scaling events
- **Scaling notifications**: SNS notifications for launch, terminate, fail

### Compliance Patterns
- **Minimum instance count**: Set ASG min ≥ compliance requirement (e.g., min=2 for HA)
- **Instance compliance**: Use lifecycle hooks to validate instance compliance before InService
- **Patch compliance**: Instance refresh to replace instances with updated AMIs (patched)
- **Tagging**: ASG propagates tags to instances (cost allocation, compliance tracking)
- **Termination protection**: Scale-in protection on specific instances (prevent accidental termination)

---

## 4. Cost Optimization

### Pricing Model
- **EC2 Auto Scaling**: FREE (no additional charge — pay only for EC2 instances)
- **AWS Auto Scaling (unified plans)**: FREE
- **Application Auto Scaling**: FREE
- **Cost = EC2 instance costs + any ELB + data transfer**

### Cost-Saving Strategies

**1. Spot Instances in ASG:**
- Up to 90% savings vs On-Demand
- Use mixed instances policy (multiple instance types + AZs)
- Allocation strategy: **price-capacity-optimized** (best balance)
- Handle interruptions: 2-minute warning → lifecycle hooks for graceful drain

**2. Right-Sizing:**
- Use Compute Optimizer recommendations for instance types
- Target tracking at higher utilization (60-70% instead of default 40-50%)
- Avoid over-provisioning by trusting auto scaling to handle spikes

**3. Scheduled Scaling:**
- Scale down during off-hours (nights, weekends)
- Scale up before business hours (avoid reactive cold-start lag)
- Dev/test environments: Schedule min=0 outside work hours

**4. Predictive Scaling:**
- Avoids over-provisioning from aggressive reactive policies
- More efficient than large static capacity "just in case"
- Combined with lower min capacity (predictive handles the baseline)

**5. Warm Pools (cost consideration):**
- Stopped instances: Pay only for EBS volumes (not compute)
- Running instances: Pay full compute (but ready instantly)
- Trade-off: Small EBS storage cost vs. fast scaling response

**6. Savings Plans / Reserved Instances:**
- RIs/Savings Plans apply to the On-Demand portion of mixed instances
- Reserve base capacity; let Spot handle the variable portion
- EC2 Instance Savings Plans: Flexible across instance sizes within family

### Cost Traps
- **ASG with min=max (no scaling)** — still paying for fixed capacity during low usage
- **Health check grace period too short** — instances cycling (launch → fail health check → terminate → launch)
- **Scaling too aggressively** — adds instances for brief spikes that resolve before instances initialize
- **Spot without diversification** — single instance type = higher interruption rate = frequent replacement
- **Not using scale-in policies** — scales out but never scales in (ratcheting up cost)
- **Cooldown too long** — keeps launching at old scale while demand drops (over-provisioned during wind-down)

---

## 5. High Availability, Disaster Recovery & Resilience

### Multi-AZ HA Pattern
```
ASG Configuration:
  - VPCZoneIdentifier: [subnet-az1, subnet-az2, subnet-az3]
  - MinSize: 3 (one per AZ minimum)
  - DesiredCapacity: 6 (two per AZ)
  - HealthCheckType: ELB (more accurate than EC2)
  - AZ Rebalancing: Enabled (default)
```
- ALB distributes traffic across AZs
- If AZ fails: ASG launches replacement instances in remaining AZs
- Cross-zone load balancing ensures even distribution

### AZ Failure Handling
1. AZ goes down → instances become unhealthy (ELB health check fails)
2. ASG terminates unhealthy instances
3. ASG launches replacements in other AZs (to maintain desired count)
4. AZ Rebalancing may trigger additional launches/terminations to re-even distribution
5. When AZ recovers: AZ Rebalancing redistributes instances

### Multi-Region HA (Active-Active)
- Separate ASGs in each region
- Route 53 latency-based or failover routing
- Each region independently scales
- Global Accelerator for instant failover (faster than DNS TTL)

### Resilience Patterns

**Graceful Scale-In:**
- Lifecycle hook (Terminating:Wait) → drain connections, finish in-flight requests
- ELB connection draining (deregistration delay): 300s default
- Custom logic: Deregister from service discovery, flush caches, upload logs

**Spot Interruption Handling:**
- 2-minute warning via instance metadata or EventBridge
- Lifecycle hook captures the termination → complete in-flight work
- Mixed instances policy ensures On-Demand base capacity remains
- **Capacity Rebalancing**: ASG proactively launches replacement before interruption

**Instance Recovery vs ASG:**
| Scenario | Solution |
|----------|----------|
| Hardware failure (single instance) | EC2 Auto Recovery (CloudWatch alarm) — same instance, same IP |
| Application failure | ASG terminates and launches NEW instance (new IP unless using Elastic IP) |
| AZ failure | ASG replaces across remaining AZs |
| Region failure | Separate ASG in DR region + DNS failover |

### RPO/RTO for Auto Scaling
- **RTO for scale-out**: Instance launch time (1-5 min typical) + health check grace period
- **Reduce RTO**: Warm Pools (seconds instead of minutes) or Predictive Scaling (pre-provision)
- **RTO for AZ failure**: Same as scale-out (ASG detects via health checks → replaces)
- **RPO**: N/A for stateless instances; for stateful, depends on data replication strategy

---

## 6. Exam-Focused Section

### Straightforward Questions (7 examples)

**Q1:** A company wants their EC2 fleet to maintain average CPU utilization at 60%. What scaling policy should they use?
**A:** **Target Tracking** scaling policy with `ASGAverageCPUUtilization` metric and target value of 60. ASG automatically creates and manages alarms.

**Q2:** An ASG has min=2, max=10, desired=4. A scaling policy adds 2 instances. What happens if there are already 9 instances?
**A:** Only **1 instance** is added (max=10 is the ceiling). ASG never exceeds the maximum capacity, even if the scaling policy requests more.

**Q3:** What is the difference between EC2 health checks and ELB health checks for an ASG?
**A:** EC2 health checks only verify the instance is running (system/instance status checks). ELB health checks verify the APPLICATION is responding (HTTP health check on specified path). ELB is more accurate for detecting application failures.

**Q4:** A company needs to run custom software installation on new ASG instances before they start receiving traffic. What feature should they use?
**A:** **Lifecycle Hook** on instance launch (Pending:Wait state). Custom action installs software, then signals `CompleteLifecycleAction` to transition instance to InService.

**Q5:** Which Auto Scaling policy type uses machine learning to forecast demand?
**A:** **Predictive Scaling** — analyzes 14 days of historical data and pre-provisions capacity before anticipated demand.

**Q6:** How do you scale ECS tasks automatically based on ALB request count?
**A:** Use **Application Auto Scaling** on the ECS Service with a Target Tracking policy on the `ALBRequestCountPerTarget` metric.

**Q7:** A company wants to replace all instances in an ASG with a new AMI without downtime. What feature achieves this?
**A:** **Instance Refresh** — performs a rolling replacement of instances with the updated launch template, maintaining minimum healthy percentage throughout.

---

### Tricky / Scenario-Based Questions (7 examples)

**Q1:** A company has an ASG with Target Tracking at 50% CPU. During a traffic spike, new instances take 5 minutes to initialize (JVM warmup). During this warmup, the instances report 100% CPU, causing the ASG to launch even MORE instances unnecessarily. How to fix?

**A:** Set the **warm-up period** in the Target Tracking policy (e.g., 300 seconds). During warm-up, new instances are excluded from the aggregate metric calculation, preventing false scaling signals. Also consider using **Warm Pools** with pre-initialized instances to eliminate the initialization spike entirely.

**Why wrong answers are wrong:**
- "Increase cooldown period" — cooldown is for Simple Scaling; Target Tracking uses warm-up
- "Scale on a different metric" — CPU is correct, just needs warm-up exclusion
- "Increase health check grace period" — prevents premature termination but doesn't fix the extra scaling
- **Keywords:** "new instances high CPU during initialization", "launches too many" → Warm-up period (or Warm Pools)

---

**Q2:** A company processes messages from an SQS queue using an ASG of worker instances. CPU utilization stays at 20% even when the queue backs up (I/O-bound workload). The ASG doesn't scale out. How to fix?

**A:** Create a **custom metric**: `BacklogPerInstance = ApproximateNumberOfMessagesVisible / NumberOfRunningInstances`. Use **Target Tracking** on this custom metric with target = acceptable backlog per instance (e.g., 10 messages). This scales based on actual queue depth, not CPU.

**Why wrong answers are wrong:**
- "Use Step Scaling with CPU > 50%" — CPU won't reach 50% for I/O-bound workloads
- "Increase instance size" — vertical scaling doesn't help if the bottleneck is parallelism
- "Switch to Lambda" — may work but question asks about fixing the ASG
- **Keywords:** "SQS queue", "CPU stays low", "queue backs up" → Custom metric (backlog per instance)

---

**Q3:** A company runs a stateless web application on an ASG with mixed instances (Spot + On-Demand). They receive a Spot interruption notice. 30 seconds later, active user sessions are dropped. How to prevent session loss?

**A:** Enable **Capacity Rebalancing** on the ASG — it proactively launches a replacement instance when a Spot instance receives a rebalance recommendation (before the 2-minute interruption). Also use **ELB connection draining** (deregistration delay) to finish in-flight requests. For sessions: move session state to **ElastiCache or DynamoDB** (stateless architecture).

**Why wrong answers are wrong:**
- "Increase the deregistration delay to 5 minutes" — helps with in-flight but doesn't prevent session loss if sessions are on-instance
- "Use Reserved Instances instead" — eliminates Spot savings
- "Handle interruption in application code" — too late for active sessions; must externalize state
- **Keywords:** "Spot interruption", "session loss" → Capacity Rebalancing + externalize session state

---

**Q4:** A company has an ASG spanning 3 AZs (2 instances each = 6 total). One AZ fails completely. The ASG minimum is 4. What happens?

**A:** The 2 instances in the failed AZ are detected as unhealthy. ASG terminates them and launches **2 new instances distributed across the remaining 2 AZs** (to maintain desired=6, not just min=4). The AZ rebalancing ensures even distribution: each remaining AZ gets 3 instances.

**Why wrong answers are wrong:**
- "ASG scales down to min=4" — ASG maintains DESIRED capacity (6), not minimum (4). Min is just the floor.
- "ASG only launches 2 instances in one remaining AZ" — AZ rebalancing distributes across remaining AZs
- "Nothing happens until health check grace period expires" — instances in a failed AZ fail immediately (host unreachable)
- **Keywords:** "AZ fails", "min vs desired" → ASG maintains desired count (not min), rebalances across remaining AZs

---

**Q5:** A company's ASG uses Target Tracking at 70% CPU with scale-in protection disabled. After a brief 2-minute traffic spike, the ASG scaled from 4 to 8 instances. Now traffic is back to normal, but the ASG stays at 8 for much longer than expected before scaling in. Why?

**A:** Target Tracking **intentionally scales in conservatively** to avoid flapping. It waits for the metric to remain below the target for a sustained period (approximately 15 minutes of data showing over-provisioning). Additionally, scale-in has a **built-in cooldown of ~300 seconds** after scale-out. This is by design — AWS prioritizes availability over cost in scale-in decisions.

**Why wrong answers are wrong:**
- "Cooldown period is too long" — Target Tracking manages its own cooldown (not configurable like Simple Scaling)
- "Instances are unhealthy" — unhealthy instances would be terminated regardless of scaling
- "Alarm is in INSUFFICIENT_DATA state" — would prevent scale-in, but the issue is intentional conservative behavior
- **Keywords:** "stays scaled out longer than expected", "Target Tracking" → Intentional conservative scale-in behavior

---

**Q6:** A company has a gaming platform that gets massive traffic at 8 PM daily and almost zero at 4 AM. They need to pre-provision by 7:50 PM (before the spike) and scale to zero at 4 AM. Current Target Tracking reacts too slowly (users experience latency during scale-out). What's the best approach?

**A:** Use **Predictive Scaling** (learns the daily pattern and pre-provisions capacity ~10 minutes before the spike) COMBINED with **Scheduled Scaling** as a safety net (set min capacity increase at 7:50 PM, decrease at 4 AM). Keep Target Tracking for handling unpredictable variations above the predicted baseline.

**Why wrong answers are wrong:**
- "Only use Scheduled Scaling" — doesn't adapt if the actual spike is larger than scheduled
- "Only use Target Tracking with lower target" — still reactive (users experience latency during scale-out)
- "Use Warm Pools" — helps with speed but doesn't eliminate the reaction delay for detecting the spike
- **Keywords:** "predictable daily pattern", "pre-provision BEFORE spike" → Predictive Scaling + Scheduled as fallback

---

**Q7:** A company uses ECS Fargate with Application Auto Scaling (Target Tracking on CPU 50%). During peak hours, tasks scale from 10 to 50, but many tasks fail to start with "unable to place task" errors. What's the issue?

**A:** **Fargate quotas** — the account has a concurrent Fargate task limit (varies by region, typically starts at a few hundred but can be lower for new accounts). Request a **service quota increase** for Fargate tasks. Also check: (1) VPC has enough IP addresses in the subnets. (2) If using ENI trunking (awsvpc mode), check ENI limits.

**Why wrong answers are wrong:**
- "Add more AZs" — helps with placement across AZs but doesn't fix quota limits
- "Switch to EC2 launch type" — works but question asks about fixing the Fargate issue
- "Increase desired count manually" — same error would occur regardless of how tasks are requested
- **Keywords:** "unable to place task", "Fargate", "scaling" → Service quota limits or subnet IP exhaustion

---

### Common Exam Traps & Pitfalls

1. **Desired vs Minimum**: ASG maintains DESIRED count, not minimum. Min is just the floor that desired/scaling can't go below.

2. **Target Tracking creates its own alarms** — don't create separate alarms for the same metric (conflict).

3. **Cooldown only applies to Simple Scaling** — Target Tracking and Step Scaling use warm-up periods instead.

4. **EC2 health check vs ELB health check**: EC2 only checks instance status (running/not). ELB checks application health. Always use ELB type when behind a load balancer.

5. **Health check grace period is critical** — if too short, instances terminated before initialization; if too long, unhealthy instances serve traffic too long.

6. **Launch Configuration is immutable** — can't update it, must create new one. Launch Template supports versions.

7. **AZ Rebalancing can terminate instances** — ASG may terminate healthy instances to rebalance. This catches candidates who think ASG only adds.

8. **Predictive Scaling needs 24h minimum data** — but 14 days for accuracy. Don't expect instant predictions.

9. **Application Auto Scaling ≠ EC2 Auto Scaling** — different services! Application Auto Scaling handles ECS, DynamoDB, Lambda provisioned concurrency, etc.

10. **Scale-in protection** doesn't protect from: manual termination, health check replacement, Spot interruption, or AZ Rebalancing.

11. **Instance Refresh ≠ CodeDeploy**: Instance Refresh replaces instances (new launch template). CodeDeploy deploys application code to existing instances.

---

## 7. Cheat Sheet

### Must-Know Facts

| Fact | Detail |
|------|--------|
| ASG pricing | Free (pay only for instances) |
| Default cooldown | 300 seconds |
| Health check grace period default | 300 seconds |
| Max instances per ASG | 2,000 (soft, can increase) |
| Predictive Scaling data needed | 24h min, 14 days ideal |
| Warm-up period | Excludes new instances from metric calculation |
| Warm Pool states | Stopped, Running, Hibernated |
| Lifecycle hook timeout | 1 hour default (max 48 hours) |
| Instance Refresh min healthy | 90% default |
| Subscription filter max | 2 per log group |
| Lambda burst concurrency | 500-3000 (region-dependent) |
| Lambda scale rate (after burst) | 500/minute |
| Spot interruption notice | 2 minutes |
| Capacity Rebalancing | Proactive replacement before interruption |

### Decision Flowchart

```
"Maintain metric at target value" → Target Tracking
"Different scale amounts for different severities" → Step Scaling
"Known daily/weekly pattern" → Predictive Scaling + Scheduled
"Event at known time (game launch)" → Scheduled Scaling (pre-provision)
"Spiky serverless workload" → Lambda (auto) + Provisioned Concurrency
"Scale ECS tasks" → Application Auto Scaling
"Scale DynamoDB" → Application Auto Scaling (or use On-Demand mode)
"Queue-based workers" → Custom metric (backlog/instance) + Target Tracking
"Long instance initialization" → Warm Pools
"Spot cost savings" → Mixed Instances Policy (price-capacity-optimized)
"Zero-downtime AMI update" → Instance Refresh
"Custom action before instance serves traffic" → Lifecycle Hook (Pending:Wait)
"Graceful shutdown" → Lifecycle Hook (Terminating:Wait) + ELB deregistration delay
"Multi-AZ resilience" → ASG across 3+ AZs + ELB health checks
```

### Key Differentiators

| Comparison | Choose A | Choose B |
|-----------|----------|----------|
| Target Tracking vs Step | Simple, set-and-forget, single target | Multi-level response, complex logic |
| Predictive vs Scheduled | Pattern varies slightly day-to-day | Exact same capacity needed every time |
| Warm Pools vs Larger ASG | Cost-sensitive, long init time | Simple, fast init, cost not primary concern |
| EC2 Auto Scaling vs App Auto Scaling | EC2 instances in ASG | ECS, DynamoDB, Lambda, Aurora, etc. |
| Instance Refresh vs CodeDeploy | Replace instances (new AMI/template) | Deploy code to existing instances |
| Spot + On-Demand vs Pure On-Demand | Cost savings, fault-tolerant app | Stateful, can't handle interruptions |

---

## 8. Additional Deep-Dive

### 10 More Tricky Scenario-Based Questions

**Q1:** A company uses an ASG with Target Tracking (CPU 50%). They also add a Scheduled Scaling action to set min=20 every Friday for a weekly sale. On Friday, the ASG has 15 instances at 30% CPU. What happens?

**A:** Scheduled Scaling sets min=20, so ASG increases desired to 20 (launches 5 more instances). Target Tracking won't scale IN below the new min=20, even though CPU is below target. When the scheduled action resets min back (e.g., to 5 on Saturday), Target Tracking will then scale in because CPU is below 50%.

**Keywords:** "scheduled + target tracking conflict" → Scheduled sets the floor, Target Tracking operates within min/max bounds

---

**Q2:** An ASG has 10 instances across 3 AZs (4-3-3 distribution). A new scaling policy adds 2 instances. Where will they be placed?

**A:** ASG places instances in the AZs with the **fewest instances** first (AZ rebalancing). The 2 new instances go to the two AZs with 3 instances each, resulting in 4-4-4 distribution.

**Keywords:** "where instances are placed" → AZ with fewest instances first (rebalancing)

---

**Q3:** A company's ASG terminates instances during scale-in, but users report dropped connections. ELB deregistration delay is set to 300 seconds. Why are connections still dropping?

**A:** The ASG might be terminating instances BEFORE the ELB deregistration delay completes. Check if the ASG's **lifecycle hook** allows enough time. Without a terminating lifecycle hook, ASG can terminate immediately. Solution: Add a **Terminating:Wait lifecycle hook** with heartbeat timeout ≥ deregistration delay, or ensure the ASG's default termination behavior waits for ELB deregistration (which it does by default when registered with a target group — but verify the target group deregistration delay value).

**Keywords:** "dropped connections during scale-in" → ELB deregistration delay + lifecycle hook timing

---

**Q4:** A company uses Predictive Scaling. On an unusual day (unplanned flash sale), traffic is 5x higher than the predicted pattern. What happens?

**A:** Predictive Scaling provisions capacity for the normal predicted pattern, but the flash sale exceeds it. The **dynamic scaling policy** (Target Tracking) kicks in reactively to handle the excess demand. This is why AWS recommends combining Predictive + Dynamic scaling — predictive handles the baseline, dynamic handles the unexpected.

**Keywords:** "unexpected spike beyond prediction" → Dynamic scaling (Target Tracking) handles overshoot

---

**Q5:** A company has an ASG with mixed instances (3 instance types across 3 AZs). They notice that one specific instance type gets interrupted much more than others. How to reduce interruptions?

**A:** Change the Spot allocation strategy from `lowest-price` to **`price-capacity-optimized`** (or `capacity-optimized`). This selects Spot pools with the most available capacity rather than just the cheapest, reducing interruption frequency. Also add more instance types to the diversification list.

**Keywords:** "frequent Spot interruptions", "one type more than others" → price-capacity-optimized allocation + more instance type diversity

---

**Q6:** A company uses DynamoDB with Auto Scaling (target 70% utilization). During a sudden burst of reads (100x normal), reads get throttled for 5-10 minutes before auto scaling catches up. How to eliminate the throttling?

**A:** Switch to **DynamoDB On-Demand** capacity mode — it handles sudden spikes instantly without auto scaling delay. If On-Demand is too expensive long-term, use **DAX** (in-memory caching) to absorb read bursts. Another option: increase the provisioned capacity's maximum in the scaling policy so it can scale higher, and set a higher target (e.g., 50% instead of 70%) so there's more headroom.

**Keywords:** "DynamoDB", "sudden burst", "throttled for minutes" → On-Demand mode (instant) or DAX (caching)

---

**Q7:** A company has an ECS cluster with Capacity Providers (managed scaling enabled). They notice EC2 instances in the ASG are at 90% memory utilization but the cluster won't add more instances. Why?

**A:** Capacity Provider managed scaling uses the **CapacityProviderReservation** metric (based on task placement needs), NOT instance-level memory. If all tasks are placed successfully (no pending tasks), the capacity provider sees no need to scale out — even if instances are highly utilized. Fix: Set `targetCapacity` in managed scaling to a lower value (e.g., 80%) to maintain headroom, or create a CloudWatch alarm on memory to trigger ASG scaling policy directly.

**Keywords:** "ECS Capacity Provider", "high utilization but not scaling" → CapacityProviderReservation metric ≠ instance utilization

---

**Q8:** A Lambda function has provisioned concurrency of 100. During peak, 150 concurrent requests arrive. What happens to the extra 50?

**A:** The extra 50 requests use **on-demand (standard) Lambda concurrency** — they CAN experience cold starts. Provisioned concurrency guarantees 100 pre-warmed environments; additional requests are served by the regular Lambda scaling mechanism (up to the account/function concurrency limit). No requests are rejected unless the total concurrent executions hit the function's reserved concurrency or account limit.

**Keywords:** "provisioned concurrency exceeded" → Extra requests use standard concurrency (possible cold starts, NOT rejected)

---

**Q9:** A company sets an ASG health check type to ELB, but instances keep getting terminated immediately after launch (never reaching InService). CloudWatch shows the application takes 4 minutes to start. What's wrong?

**A:** The **health check grace period** is too short (default 300s = 5 min, but if set lower). If the ELB health check runs before the application is ready, it reports unhealthy → ASG terminates → launches new → cycle repeats. Fix: Increase the health check grace period to exceed application startup time (e.g., 360 seconds for a 4-minute startup).

**Keywords:** "instances terminated immediately after launch", "ELB health check", "app takes time to start" → Increase health check grace period

---

**Q10:** A company uses an ASG with both a Target Tracking policy (CPU 60%) and a Step Scaling policy (based on custom metric: request queue depth). Both policies want to scale out simultaneously. What happens?

**A:** When multiple scaling policies instruct to scale out simultaneously, ASG uses the policy that provides the **largest capacity increase** (most instances). It doesn't add them together. For scale-in, ASG uses the most conservative (least reduction). This ensures availability is prioritized.

**Keywords:** "multiple scaling policies", "both want to scale" → ASG honors the largest increase (not additive)

---

### Compare: Scaling Policy Types

| Aspect | Target Tracking | Step Scaling | Simple Scaling | Scheduled | Predictive |
|--------|----------------|-------------|----------------|-----------|------------|
| **Use Case** | Maintain metric at value | Variable response by severity | Basic (legacy) | Known time-based patterns | ML-forecasted patterns |
| **Complexity** | Low (set target) | Medium (define steps) | Low | Low (set schedule) | Low (enable + wait) |
| **Reactivity** | Moderate (waits for data) | Fast (alarm-driven) | Slow (cooldown) | Proactive (time-based) | Proactive (predicted) |
| **Scale-in** | Conservative (built-in) | As aggressive as alarm allows | After cooldown | At scheduled time | At predicted decrease |
| **Exam Tip** | "Maintain at X%", "simplest" | "Scale more for bigger breach" | Don't choose (legacy) | "Every Monday", "business hours" | "Daily pattern", "forecast" |

### Compare: EC2 Auto Scaling vs Application Auto Scaling

| Aspect | EC2 Auto Scaling | Application Auto Scaling |
|--------|-----------------|------------------------|
| **Targets** | EC2 instances (ASG) | ECS, DynamoDB, Aurora, Lambda, EMR, etc. |
| **Infrastructure** | Manages EC2 instances directly | Manages service capacity (tasks, RCU/WCU, replicas) |
| **Policies** | Target Tracking, Step, Simple, Scheduled, Predictive | Target Tracking, Step, Scheduled |
| **Unique Features** | Launch templates, warm pools, lifecycle hooks, instance refresh | Custom resources API for any scalable target |
| **Exam Tip** | "EC2 fleet", "ASG", "instances" | "ECS tasks", "DynamoDB capacity", "Aurora replicas" |

### When Does SAP-C02 Expect Which Scaling Approach?

**Pick Target Tracking when:**
- "Simplest scaling policy"
- "Maintain at target percentage"
- "Set-and-forget"
- Default recommendation if no specific need mentioned

**Pick Step Scaling when:**
- "Different response for different thresholds"
- "Scale more aggressively as metric worsens"
- "Custom metric with variable response"

**Pick Scheduled Scaling when:**
- "Known times" (business hours, weekly events)
- "Minimum capacity during peak hours"
- "Scale to zero at night"

**Pick Predictive Scaling when:**
- "Daily/weekly pattern"
- "Pre-provision capacity"
- "Reduce scaling latency for predictable traffic"

**Pick Warm Pools when:**
- "Long initialization time"
- "Reduce scale-out time"
- "Pre-warmed instances"

**Pick DynamoDB On-Demand when:**
- "Unpredictable/spiky traffic"
- "Zero throttling tolerance"
- "New application with unknown patterns"

### Decision Tree: Scaling Strategy Selection

```
Is the workload pattern PREDICTABLE (same daily/weekly)?
  ├── YES → Predictive Scaling (baseline) + Target Tracking (spikes)
  │         ├── Exact known time? → Add Scheduled Scaling
  │         └── Need pre-warmed capacity? → Warm Pools
  └── NO → Is it REACTIVE (unknown patterns)?
            ├── Single metric to maintain? → Target Tracking
            ├── Multi-level response? → Step Scaling
            ├── Queue-based work? → Custom metric (backlog/instance) + Target Tracking
            └── Completely unpredictable (serverless)? → Lambda + DynamoDB On-Demand

Is the app STATELESS?
  ├── YES → Spot instances OK (mixed policy, cost savings)
  └── NO → On-Demand only (or externalize state first)

Long initialization time?
  ├── YES → Warm Pools (Stopped or Running state)
  └── NO → Standard ASG (launch on demand)

Need custom actions during launch/termination?
  ├── YES → Lifecycle Hooks
  └── NO → Standard ASG behavior
```
