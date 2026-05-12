# CloudWatch: Metrics, Logs, Alarms, Composite Alarms, Dashboards

## 1. Core Concepts & Theory

### Amazon CloudWatch Overview
- **Unified monitoring and observability service** for AWS resources and applications
- Collects metrics, logs, and events — enables alarms, dashboards, and automated actions
- **Regional service** (metrics/logs are per-region), but cross-account and cross-region dashboards are supported

### CloudWatch Metrics

**Core Concepts:**
- **Namespace**: Container for metrics (e.g., `AWS/EC2`, `AWS/RDS`, custom: `MyApp/Backend`)
- **Metric**: Time-ordered set of data points (e.g., CPUUtilization)
- **Dimension**: Name-value pair that uniquely identifies a metric (e.g., InstanceId=i-123)
- **Statistic**: Aggregation over a period (Sum, Average, Min, Max, SampleCount)
- **Percentile**: p99, p95, p50 — useful for latency metrics
- **Period**: Length of time for one data point (minimum 1 second for high-resolution, default 60s or 300s)
- **Resolution**: Standard (60-second) or High-Resolution (1-second)
- **Data Retention**:
  - 1-second data points: retained for **3 hours**
  - 60-second data points: retained for **15 days**
  - 5-minute data points: retained for **63 days**
  - 1-hour data points: retained for **455 days** (15 months)

**Default vs Detailed Monitoring:**
| Feature | Basic (Free) | Detailed (Paid) |
|---------|-------------|-----------------|
| EC2 metrics | Every 5 minutes | Every 1 minute |
| Cost | Free | $2.10/instance/month (first 10 free) |
| Use case | General monitoring | Auto Scaling responsiveness |

**Custom Metrics:**
- Published via `PutMetricData` API or CloudWatch Agent
- Can be standard (60s) or high-resolution (1s)
- Max 30 dimensions per metric
- Max **150 transactions per second (TPS)** per `PutMetricData` call (soft limit)
- Metric math: perform calculations across metrics (e.g., error rate = errors/requests × 100)

**Embedded Metric Format (EMF):**
- Publish custom metrics from Lambda/ECS via structured JSON log lines
- No `PutMetricData` API call needed — metrics extracted from logs automatically
- Lower cost than PutMetricData for high-volume metrics

**Key EC2 Metrics (memory/disk NOT included by default):**
| Included by Default | Requires CloudWatch Agent |
|--------------------|--------------------------|
| CPUUtilization | MemoryUtilization |
| NetworkIn/Out | DiskSpaceUtilization |
| DiskReadOps/WriteOps | Swap usage |
| StatusCheckFailed | Per-process metrics |
| EBSReadOps/WriteOps | Custom application metrics |

**Key Limits:**
| Limit | Value |
|-------|-------|
| Custom metrics per account per region | 10,000 (soft) |
| Dimensions per metric | 30 |
| PutMetricData data points per call | 1,000 |
| Metric data batch size | 1 MB |
| Dashboard limit | 5,000 per account |
| Metrics per dashboard widget | 500 |
| Metric retention (1-hour) | 455 days |

### CloudWatch Logs

**Core Concepts:**
- **Log Group**: Collection of log streams sharing retention, monitoring, and access settings
- **Log Stream**: Sequence of log events from a single source (e.g., one EC2 instance, one Lambda invocation)
- **Log Event**: Single record (timestamp + message)
- **Metric Filter**: Extract metric values from log data (e.g., count ERROR occurrences)
- **Subscription Filter**: Real-time stream of log events to destinations (Lambda, Kinesis, Firehose, OpenSearch)
- **Log Insights**: Interactive query language for analyzing log data
- **Contributor Insights**: Identify top-N contributors (e.g., top IP addresses, top error sources)
- **Log Class**: Standard (full features) or Infrequent Access (cheaper, limited features)

**Log Sources:**
- CloudWatch Agent (EC2, on-prem)
- Lambda (automatic — /aws/lambda/function-name)
- API Gateway (access logs, execution logs)
- VPC Flow Logs
- Route 53 DNS query logs
- ECS/Fargate (awslogs driver)
- CloudTrail (management/data events)
- RDS/Aurora (error, slow query, general, audit logs)
- Custom applications (SDK, CLI)

**Log Retention:**
- Default: **Never expire** (indefinite)
- Configurable: 1 day to 10 years, or indefinite
- Important: You MUST set retention to control costs

**Subscription Filters:**
- Max **2 subscription filters per log group**
- Destinations: Lambda, Kinesis Data Streams, Kinesis Data Firehose, OpenSearch
- Cross-account: Can subscribe to another account's log group (requires destination policy)

**Log Insights Query Language:**
```
fields @timestamp, @message
| filter @message like /ERROR/
| stats count(*) as errorCount by bin(5m)
| sort errorCount desc
| limit 20
```

**Export to S3:**
- One-time export task (`CreateExportTask`) — NOT real-time (up to 12 hours delay)
- For real-time: Use subscription filter → Kinesis Firehose → S3

**Key Limits:**
| Limit | Value |
|-------|-------|
| Log groups per account per region | 1,000,000 |
| Subscription filters per log group | 2 |
| Log event size | 256 KB |
| PutLogEvents batch size | 1 MB or 10,000 events |
| Metric filters per log group | 100 |
| Log Insights query timeout | 60 minutes |
| Log Insights max results | 10,000 |
| Export to S3 concurrency | 1 task at a time per account |

### CloudWatch Alarms

**Core Concepts:**
- **Metric Alarm**: Watches a single metric or metric math expression
- **States**: OK → ALARM → INSUFFICIENT_DATA
- **Evaluation Period**: Number of consecutive periods to evaluate
- **Datapoints to Alarm**: How many data points within evaluation period must breach threshold (M of N)
- **Treat Missing Data**: `missing` (maintain state), `notBreaching`, `breaching`, `ignore`
- **Actions**: SNS notification, Auto Scaling, EC2 actions (stop/terminate/reboot/recover), Systems Manager

**Alarm Resolution:**
- Standard alarms: Evaluate at the metric's period (60s, 300s, etc.)
- High-resolution alarms: Can evaluate at 10-second or 30-second periods (for high-res metrics)

**Alarm Actions:**
| Action Type | Examples |
|-------------|---------|
| SNS | Notify team, trigger Lambda via SNS |
| Auto Scaling | Scale out/in policy |
| EC2 | Stop, Terminate, Reboot, Recover instance |
| Systems Manager | OpsItem, Incident Manager |

**EC2 Instance Recovery:**
- `recover` action: Migrates instance to new hardware (same instance ID, IP, metadata)
- Works only for instances with EBS-backed root (NOT instance store)
- Preserves: instance ID, private IP, Elastic IP, metadata, placement group

### CloudWatch Composite Alarms

- **Combines multiple alarms** using Boolean logic (AND, OR, NOT)
- Reduces **alarm noise** — only triggers when multiple conditions are met simultaneously
- **Rule expression**: `ALARM(alarm1) AND ALARM(alarm2)` or `ALARM(alarm1) OR ALARM(alarm2)`
- Can suppress actions during deployment windows (using `ActionsSuppressor`)
- **Use case**: "Alert only when BOTH CPU is high AND memory is high" (avoids false positives from single-metric spikes)

**Actions Suppressor:**
- Suppress composite alarm actions while a specified "suppressor alarm" is in ALARM state
- Use case: Suppress alerts during planned maintenance or deployments
- `ActionsSuppressorWaitPeriod`: How long to wait before suppressing
- `ActionsSuppressorExtensionPeriod`: How long to continue suppressing after suppressor returns to OK

**Key Facts:**
- Composite alarms can include other composite alarms (nesting)
- Max **5 levels of nesting**
- Composite alarms do NOT take actions on `INSUFFICIENT_DATA` (only OK/ALARM)
- Can be used as targets for EventBridge rules

### CloudWatch Dashboards

- **Customizable pages** in CloudWatch console for monitoring
- **Widgets**: Line graphs, stacked area, numbers, text, logs, alarms, metric explorer
- **Cross-account dashboards**: View metrics from multiple accounts in one dashboard
- **Cross-region dashboards**: View metrics from multiple regions in one dashboard
- **Automatic dashboards**: AWS auto-generates dashboards for each service
- **Sharing**: Dashboards can be shared publicly (with SSO, Cognito, or unauthenticated)

**Dashboard Pricing:**
- First **3 dashboards** (up to 50 metrics each): **Free**
- Additional: **$3.00/dashboard/month**

**Key Features:**
- Annotations: Horizontal (threshold lines) and vertical (event markers)
- Variables: Dynamic dashboards using filter parameters
- Anomaly detection bands: Display expected ranges
- Live mode: Auto-refresh every 10 seconds
- Full-screen mode: TV/wall display

### CloudWatch Agent

- **Unified agent** for collecting metrics AND logs from EC2 and on-premises
- Replaces legacy CloudWatch Logs agent and custom scripts
- Configured via **JSON configuration file** (wizard available: `amazon-cloudwatch-agent-config-wizard`)
- Can collect: system metrics (memory, disk, swap, netstat), custom application metrics, log files
- Supports **StatsD** and **collectd** protocols for custom metrics
- Stored in **SSM Parameter Store** for centralized configuration management

### CloudWatch Anomaly Detection
- ML-based model that learns expected metric patterns (daily/weekly seasonality)
- Creates an **anomaly detection band** (expected range)
- Can be used as alarm threshold: alarm triggers when metric is outside the band
- Training period: ~2 weeks for accurate model
- Supports exclusion windows (e.g., exclude Black Friday from normal patterns)

### CloudWatch Evidently
- **Feature flags** and **A/B testing** service
- Launch features gradually with traffic splitting
- Measure impact with metrics
- Different from CodeDeploy canary (Evidently is for feature experimentation, not deployment)

### CloudWatch Synthetics (Canaries)
- **Synthetic monitoring** — scripts that run on schedule to check endpoints
- Written in Node.js or Python (runs on Lambda)
- Monitors: availability, latency, UI screenshots, broken links, API responses
- **Canary blueprints**: Heartbeat, API, broken link checker, visual monitoring, GUI workflow

---

## 2. Design Patterns & Best Practices

### When to Use

| Feature | Use When | Don't Use When |
|---------|----------|----------------|
| CloudWatch Metrics | Monitoring AWS resources, custom app metrics | Need distributed tracing (use X-Ray) |
| CloudWatch Logs | Centralized logging, log analysis | Need real-time streaming analytics (use Kinesis) |
| Metric Alarms | Alert on single metric threshold | Need multi-condition alerting (use Composite) |
| Composite Alarms | Multi-condition alerting, reduce noise | Simple single-metric threshold |
| Dashboards | Operational visibility, NOC displays | Ad-hoc analysis (use Log Insights) |
| Anomaly Detection | Unpredictable baselines, seasonal patterns | Known static thresholds |
| Synthetics | Monitor customer-facing endpoints proactively | Internal-only services (use health checks) |
| Embedded Metric Format | High-volume custom metrics from Lambda/containers | Low-volume metrics (PutMetricData is simpler) |

### Anti-Patterns
- **Don't use CloudWatch Logs for real-time streaming** — use Kinesis Data Streams for sub-second processing
- **Don't export logs to S3 for real-time analysis** — export is batch (12h delay); use subscription filters → Firehose → S3
- **Don't create one alarm per instance** — use composite alarms or metric math to aggregate
- **Don't use CloudWatch as a SIEM** — use Security Hub, GuardDuty, or third-party SIEM
- **Don't publish high-cardinality dimensions** (e.g., request ID as dimension) — creates millions of metrics, very expensive

### Architectural Patterns

**Centralized Logging (Multi-Account):**
```
Member Accounts → CloudWatch Logs → Subscription Filter → 
  Kinesis Data Firehose (central account) → S3 (log archive) + OpenSearch (analysis)
```
- Cross-account log subscriptions require destination access policy
- Alternative: CloudWatch cross-account observability (newer feature)

**CloudWatch Cross-Account Observability:**
- **Monitoring account**: Views metrics/logs/traces from source accounts
- **Source accounts**: Share observability data with monitoring account
- Uses **OAM (Observability Access Manager)** links and sinks
- Source accounts create OAM links → Monitoring account has OAM sink
- Supports: Metrics, Logs, Traces (X-Ray), Application Insights

**Operational Dashboard Pattern:**
- Cross-account + cross-region dashboard in central ops account
- Composite alarms for service-level indicators (SLIs)
- Anomaly detection for auto-adapting thresholds
- Synthetics for external user experience monitoring

**Event-Driven Alerting:**
```
CloudWatch Alarm → SNS → Lambda (enrichment) → Slack/PagerDuty
                 → EventBridge → Incident Manager (automated response)
```

**Log Analytics Pipeline:**
```
Application → CloudWatch Logs → Metric Filters (KPIs)
                               → Subscription Filter → OpenSearch (full-text search)
                               → Log Insights (ad-hoc queries)
                               → S3 (long-term archive via Firehose)
```

### Well-Architected Alignment

| Pillar | Practice |
|--------|----------|
| Reliability | Alarms on key metrics, composite alarms for SLIs, auto-recovery actions |
| Security | Log all API calls (CloudTrail → CloudWatch Logs), metric filters for security events, encrypted log groups |
| Cost | Set log retention, use Infrequent Access log class, avoid high-cardinality custom metrics |
| Performance | Detailed monitoring for ASG, anomaly detection for latency, Synthetics for endpoint performance |
| Ops Excellence | Cross-account observability, standardized dashboards, runbook integration via SSM |

### Integration Points
- **EventBridge**: Alarm state changes generate events; schedule-based actions
- **SNS**: Primary alarm notification mechanism
- **Lambda**: Triggered by alarms (via SNS), subscription filters, Synthetics
- **Auto Scaling**: Alarm-based scaling policies (target tracking, step, simple)
- **EC2**: Instance actions (stop, terminate, reboot, recover)
- **Systems Manager**: OpsItems from alarms, parameter storage for agent config, Incident Manager
- **S3**: Log export, Firehose delivery, metric stream delivery
- **OpenSearch**: Real-time log analysis via subscription filters
- **Kinesis**: Subscription filter destination, metric streams
- **X-Ray**: Cross-account traces via CloudWatch observability
- **Grafana (Amazon Managed Grafana)**: CloudWatch as data source

---

## 3. Security & Compliance

### IAM & Access Control

**CloudWatch Permissions:**
- `cloudwatch:PutMetricData` — publish custom metrics
- `cloudwatch:GetMetricData` / `GetMetricStatistics` — read metrics
- `cloudwatch:PutMetricAlarm` / `DeleteAlarms` — manage alarms
- `cloudwatch:PutDashboard` — create/update dashboards
- `cloudwatch:DescribeAlarms` — view alarm state

**CloudWatch Logs Permissions:**
- `logs:CreateLogGroup` / `CreateLogStream` — create groups/streams
- `logs:PutLogEvents` — write logs
- `logs:GetLogEvents` / `FilterLogEvents` — read logs
- `logs:PutSubscriptionFilter` — set up subscriptions
- `logs:StartQuery` — run Log Insights queries

**Resource Policies (for cross-account):**
- Log group resource policy: Allows another account's service to write logs
- Destination policy: Allows cross-account subscription filter delivery
- Example: Route 53 query logs require resource policy allowing `route53.amazonaws.com`

**Cross-Account Access:**
- **Cross-account IAM roles**: Traditional approach (assume role to access CloudWatch in another account)
- **CloudWatch cross-account observability (OAM)**: Modern approach — source accounts share data to monitoring account
- **Sharing permissions**: Can share specific telemetry types (metrics, logs, traces)

### Encryption

**CloudWatch Logs:**
- Encrypted at rest by default (AWS-managed keys)
- **Optional**: Customer-managed KMS CMK per log group (`associate-kms-key`)
- Once set, all new log data encrypted with CMK
- KMS key policy must allow `logs.REGION.amazonaws.com` service principal

**CloudWatch Metrics:**
- Encrypted at rest (AWS-managed, not configurable)
- In-transit: All API calls over TLS

**Metric Streams:**
- Data delivered to Firehose is encrypted per Firehose encryption settings
- Can specify KMS key for Firehose delivery

### Logging & Auditing
- **CloudTrail** logs all CloudWatch/Logs API calls
- **Alarm history**: 2 weeks of alarm state change history
- **Log data protection**: Detect and mask sensitive data in logs (PII, credit cards)
  - Uses managed data identifiers or custom patterns
  - Can audit findings without exposing data
  - Masking in query results

### Compliance Patterns
- **Log retention enforcement**: Use AWS Config rule to ensure log groups have retention set
- **Centralized security logging**: All accounts stream CloudTrail/VPC Flow Logs to central account
- **Data protection**: Enable log data protection policies for PII masking
- **Access control**: Restrict who can run Log Insights queries (data exfiltration risk)
- **Encryption audit**: Config rule to verify KMS encryption on log groups

---

## 4. Cost Optimization

### Pricing Model

**CloudWatch Metrics:**
| Item | Price |
|------|-------|
| First 10 custom metrics | Free |
| Custom metrics | $0.30/metric/month (first 10K), decreasing tiers |
| API requests (GetMetricData) | $0.01 per 1,000 metrics requested |
| Detailed monitoring (EC2) | $2.10/instance/month |
| Metric Streams | $0.003 per 1,000 metric updates |
| Anomaly Detection | Free (included with alarm cost) |

**CloudWatch Logs:**
| Item | Price |
|------|-------|
| Ingestion (Standard) | $0.50/GB |
| Ingestion (Infrequent Access) | $0.25/GB |
| Storage | $0.03/GB/month |
| Log Insights queries | $0.005 per GB scanned |
| Vended logs (VPC Flow, Route53) | $0.10/GB (ingestion discount) |
| Cross-account delivery | Standard ingestion rate |

**CloudWatch Alarms:**
| Item | Price |
|------|-------|
| Standard alarm | $0.10/alarm/month |
| High-resolution alarm | $0.30/alarm/month |
| Composite alarm | $0.50/alarm/month |
| Anomaly detection alarm | $0.30/alarm/month + model training |

**Dashboards:**
- First 3 (50 metrics each): Free
- Additional: $3.00/dashboard/month

### Cost-Saving Strategies

1. **Set log retention** — default is "never expire" which accumulates cost indefinitely
2. **Use Infrequent Access log class** — 50% cheaper ingestion for logs you rarely query
3. **Vended logs discount** — VPC Flow Logs, Route 53 query logs get $0.10/GB vs $0.50/GB
4. **Avoid high-cardinality dimensions** — each unique dimension combination is a separate metric ($0.30/month)
5. **Use EMF instead of PutMetricData** — extract metrics from logs (cheaper for high-volume)
6. **Filter logs before ingestion** — use CloudWatch Agent's log filtering to drop irrelevant lines
7. **Use metric math** instead of publishing derived metrics — compute at query time, zero storage cost
8. **Consolidate alarms** — use composite alarms instead of many individual alarms
9. **Archive to S3** — for long-term storage, stream to S3 via Firehose (S3 storage: $0.023/GB vs CW Logs: $0.03/GB)
10. **Metric Streams to third-party** — if using Datadog/Splunk, stream directly instead of API polling (cheaper at scale)

### Cost Traps
- **Unlimited log retention** is the #1 CloudWatch cost trap
- **Detailed monitoring on all EC2** when only ASG instances need it
- **Excessive custom metrics** with high-cardinality dimensions (e.g., user ID as dimension = millions of metrics)
- **Log Insights queries** on large log groups without time-range filter (scans everything)
- **Cross-region metric widget** API calls accumulate charges
- **Metric Streams** sending metrics you don't need (filter by namespace)

---

## 5. High Availability, Disaster Recovery & Resilience

### Built-in HA
- CloudWatch is a **fully managed regional service** — multi-AZ within a region
- No customer-managed HA required
- Metrics stored durably (455 days retention for 1-hour data)
- Logs stored across multiple AZs within a region
- Alarms evaluate continuously — no single point of failure

### Multi-Region Observability

**Cross-Region Dashboards:**
- Single dashboard showing metrics from multiple regions
- Each widget specifies its source region
- API calls made to each region (no data replication)

**Multi-Region Alarming:**
- Alarms are regional — must create alarm in each region
- Use EventBridge cross-region event routing to centralize alarm events
- Composite alarm can reference alarms in same region only

**Log Replication:**
- CloudWatch Logs does NOT natively replicate cross-region
- Pattern: Subscription filter → Kinesis Firehose → S3 (with CRR) for cross-region log availability
- Or: Subscription filter → Kinesis Data Streams (cross-region delivery)

### DR Patterns

**Monitoring DR:**
- CloudFormation/CDK for dashboard and alarm definitions (recreate in DR region)
- Alarms auto-resume once metrics flow in DR region
- Agent configuration in SSM Parameter Store (replicate or deploy per-region)

**Log Availability During DR:**
- If primary region fails, logs in that region are inaccessible
- Best practice: Stream critical logs to S3 (cross-region replication) or to another region via Firehose
- CloudWatch cross-account observability: monitoring account loses visibility if source region fails

### Resilience Patterns
- **Synthetic canaries** in multiple regions detect regional failures
- **Composite alarms** combining metrics from multiple services detect cascading failures
- **Auto-recovery action** on StatusCheckFailed_System alarm (moves EC2 to healthy hardware)
- **Contributor Insights** identifies anomalous traffic patterns during incidents

---

## 6. Exam-Focused Section

### Straightforward Questions (7 examples)

**Q1:** An application running on EC2 needs to publish memory utilization metrics to CloudWatch. What is required?
**A:** Install and configure the **CloudWatch Agent** on the EC2 instance. Memory metrics are NOT collected by default — the agent is required.

**Q2:** A company wants to be notified when the error count exceeds 100 in a 5-minute period in their application logs. How?
**A:** Create a **Metric Filter** on the CloudWatch Log Group that counts "ERROR" occurrences, then create a **CloudWatch Alarm** on that metric with threshold > 100 and period 5 minutes.

**Q3:** How can logs from multiple AWS accounts be collected into a single account for analysis?
**A:** Use **cross-account log subscriptions** — subscription filters in source accounts deliver logs to a destination (Kinesis/Lambda) in the central account. Or use **CloudWatch cross-account observability** (OAM) for the modern approach.

**Q4:** A company needs real-time log delivery to S3 for compliance. Which approach works?
**A:** Subscription filter → **Kinesis Data Firehose** → S3. NOT `CreateExportTask` (which is batch with up to 12-hour delay).

**Q5:** What is the maximum retention period for CloudWatch metrics?
**A:** **455 days (15 months)** for 1-hour aggregated data. Higher-resolution data expires sooner (3 hours for 1-second, 15 days for 1-minute, 63 days for 5-minute).

**Q6:** A team wants to alert only when BOTH CPU utilization is high AND available memory is low, to reduce false alarms from single-metric spikes. What should they use?
**A:** **Composite Alarm** with rule expression: `ALARM(HighCPUAlarm) AND ALARM(LowMemoryAlarm)`.

**Q7:** An EC2 instance becomes unreachable due to underlying hardware failure. How can CloudWatch automatically recover it?
**A:** Create an alarm on `StatusCheckFailed_System` metric with the **EC2 recover action**. The instance migrates to new hardware retaining its instance ID and private IP.

---

### Tricky / Scenario-Based Questions (7 examples)

**Q1:** A company has a Lambda function that processes 1 million requests per hour. They need to track custom business metrics (order count, revenue) with minimal latency and cost. What approach is best?

**A:** Use **Embedded Metric Format (EMF)** — write structured JSON log lines that CloudWatch automatically extracts as metrics. No `PutMetricData` API calls needed, and it's more cost-effective at high volume than calling the API for each invocation.

**Why wrong answers are wrong:**
- "PutMetricData from each Lambda invocation" — 1M API calls/hour would be throttled (150 TPS limit) and expensive
- "Write to DynamoDB, then query" — not a monitoring solution, doesn't integrate with alarms
- "Publish to Kinesis, process with Lambda" — unnecessary complexity for metric publishing
- **Keywords:** "high volume", "Lambda", "custom metrics", "minimal cost" → EMF

---

**Q2:** A company needs to monitor application logs for specific patterns (failed login attempts > 50 in 5 minutes) and trigger an automated security response. Which combination of services achieves this with least operational overhead?

**A:** **Metric Filter** (count failed login patterns) → **CloudWatch Alarm** (threshold > 50 in 5 min) → **SNS** → **Lambda** (automated response: block IP in WAF, create Security Hub finding). Alternatively, alarm → EventBridge → Systems Manager Incident Manager.

**Why wrong answers are wrong:**
- "Use GuardDuty" — GuardDuty doesn't analyze application logs (it monitors VPC Flow Logs, DNS, CloudTrail)
- "Stream logs to OpenSearch and build alerts" — much higher operational overhead than metric filter + alarm
- "Use CloudWatch Contributor Insights" — identifies top contributors but doesn't trigger automated actions
- **Keywords:** "application logs", "pattern matching", "automated response", "least overhead" → Metric Filter + Alarm + Lambda

---

**Q3:** A company has 200 microservices. During deployments, many individual alarms trigger (CPU spikes, latency increases) creating excessive notifications. They want to suppress deployment-related alerts without disabling monitoring. What's the best approach?

**A:** Use **Composite Alarms with ActionsSuppressor**. Create a "deployment in progress" alarm (e.g., triggered by a custom metric published during deployments). Set this as the `ActionsSuppressor` on composite alarms. While the deployment alarm is in ALARM state, other alarm actions are suppressed.

**Why wrong answers are wrong:**
- "Disable alarms during deployment" — lose monitoring during the most risky period
- "Increase alarm thresholds during deployment" — complex to coordinate, easy to forget resetting
- "Use maintenance windows in Incident Manager" — doesn't suppress CloudWatch alarm actions directly
- **Keywords:** "suppress during deployment", "don't disable monitoring" → Composite Alarm + ActionsSuppressor

---

**Q4:** A company's CloudWatch Logs bill has doubled in 3 months. They have VPC Flow Logs enabled across 50 accounts. What's the most cost-effective fix while retaining the log data?

**A:** Switch VPC Flow Logs to deliver directly to **S3** instead of CloudWatch Logs. S3 delivery costs $0.023/GB (storage) vs CloudWatch Logs at $0.50/GB (ingestion) + $0.03/GB (storage). If they MUST stay in CloudWatch Logs, use the **Infrequent Access log class** ($0.25/GB ingestion) or use **Vended Logs** pricing ($0.10/GB) which applies automatically to VPC Flow Logs.

**Why wrong answers are wrong:**
- "Reduce retention period" — helps with storage ($0.03/GB) but ingestion ($0.50/GB) is the bigger cost
- "Sample flow logs" — loses data visibility
- "Use traffic mirroring instead" — completely different feature, doesn't replace flow logs
- **Keywords:** "cost", "VPC Flow Logs", "doubled" → Deliver to S3 directly (or confirm vended logs pricing is applied)

---

**Q5:** A company has an Auto Scaling group. They want to scale out when the AVERAGE CPU across all instances exceeds 70% for 3 consecutive 1-minute periods. The alarm keeps firing on brief spikes and scaling too aggressively. How to fix?

**A:** Change the alarm evaluation to **5 out of 5 data points** (or increase the period to 5 minutes) OR use **Metric Math with a moving average** to smooth out spikes. The current "3 of 3 × 1 minute" is too sensitive. Another option: change from **Step Scaling** to **Target Tracking** scaling policy (which manages the alarm internally with built-in dampening).

**Why wrong answers are wrong:**
- "Add a cooldown period" — cooldowns prevent scaling actions AFTER a scale event, but don't prevent the alarm from firing
- "Use high-resolution alarm" — this would make it MORE sensitive, not less
- "Treat missing data as not breaching" — doesn't address the spike sensitivity issue
- **Keywords:** "brief spikes", "too aggressively", "smooth" → Increase evaluation period or use target tracking

---

**Q6:** A company needs to monitor a multi-step order processing workflow across Lambda, SQS, and DynamoDB. They want a single view showing how orders flow through the system and where bottlenecks occur. What should they use?

**A:** **AWS X-Ray** for distributed tracing (shows the flow and latency at each step) combined with a **CloudWatch Dashboard** for aggregate metrics. X-Ray provides the service map and per-request tracing; CloudWatch provides aggregate KPIs.

**Why wrong answers are wrong:**
- "CloudWatch Logs Insights" — can query logs but doesn't show flow/dependencies between services
- "CloudWatch Contributor Insights" — shows top-N contributors, not request flow
- "CloudWatch ServiceLens" — correct answer IF listed (ServiceLens combines X-Ray + CloudWatch), but typically the answer is X-Ray for the tracing component
- **Keywords:** "flow through system", "bottlenecks", "multi-service" → X-Ray (tracing + service map)

---

**Q7:** A company uses CloudWatch cross-account observability. The monitoring account suddenly shows no metrics from one source account. CloudTrail shows no permission changes. What's the most likely cause?

**A:** The **OAM link** in the source account may have been deleted, or the source account was removed from the organization OU that has the sharing configuration. Check the OAM link status in the source account and verify the OAM sink in the monitoring account still lists the source.

**Why wrong answers are wrong:**
- "KMS key rotation" — OAM doesn't use customer-managed encryption for the link itself
- "CloudWatch service outage in source region" — would affect all accounts in that region, not just one
- "IAM role expired" — OAM uses service-linked roles, which don't expire
- **Keywords:** "cross-account observability", "one source account stops" → OAM link issue

---

### Common Exam Traps & Pitfalls

1. **Memory/disk metrics require CloudWatch Agent** — not available by default on EC2. #1 most-tested fact.

2. **Log export to S3 is NOT real-time** — `CreateExportTask` can take up to 12 hours. For real-time, use subscription filter → Firehose → S3.

3. **Subscription filter limit = 2 per log group** — if you need more destinations, fan out through Kinesis.

4. **Metric retention auto-aggregates** — 1-second data becomes 1-minute after 3 hours. You cannot keep 1-second resolution long-term.

5. **Composite alarms ≠ metric math alarms** — Composite alarms combine ALARM states (Boolean). Metric math combines metric VALUES (arithmetic).

6. **Vended logs pricing** — VPC Flow Logs, Route 53 query logs are automatically cheaper ($0.10/GB vs $0.50/GB) when sent to CloudWatch Logs.

7. **PutMetricData TPS limit (150)** — high-volume apps should batch data points or use EMF instead.

8. **CloudWatch Alarms can directly stop/terminate/reboot/recover EC2** — no Lambda required for these basic actions.

9. **Cross-region alarms don't exist** — you must create alarms in each region. Centralize via EventBridge cross-region routing.

10. **Anomaly Detection needs 2 weeks of data** — don't expect accurate baselines on new metrics immediately.

11. **Log group encryption** — associate KMS key AFTER creation (can't specify at creation in console, but can in API/CF). Key policy must allow `logs.REGION.amazonaws.com`.

---

## 7. Cheat Sheet

### Must-Know Facts

| Fact | Detail |
|------|--------|
| EC2 default metrics interval | 5 minutes (basic), 1 minute (detailed) |
| Memory/disk metrics | Require CloudWatch Agent |
| High-resolution metrics | 1-second interval, $0.30/metric/month |
| Metric retention (1-hour) | 455 days |
| Metric retention (1-second) | 3 hours |
| Log subscription filters max | 2 per log group |
| Log event max size | 256 KB |
| Log export to S3 | NOT real-time (up to 12h delay) |
| Real-time logs to S3 | Subscription filter → Firehose → S3 |
| Composite alarm logic | AND, OR, NOT on alarm states |
| ActionsSuppressor | Suppress alarm actions during maintenance |
| EC2 recover action | Same ID, IP, metadata — EBS-only |
| Embedded Metric Format | Custom metrics via log lines (high volume) |
| Anomaly Detection training | ~2 weeks for accurate model |
| Dashboard pricing | First 3 free, then $3/month |
| Alarm pricing (standard) | $0.10/alarm/month |
| CW Logs ingestion | $0.50/GB (standard), $0.25/GB (IA) |
| Vended logs | $0.10/GB (VPC Flow, Route 53) |

### Decision Flowchart

```
"Monitor memory/disk on EC2" → CloudWatch Agent
"Custom metrics from Lambda at scale" → Embedded Metric Format (EMF)
"Real-time logs to S3" → Subscription Filter → Firehose → S3
"Log pattern alerting" → Metric Filter → Alarm
"Multi-condition alarm" → Composite Alarm
"Suppress alerts during deployment" → Composite Alarm + ActionsSuppressor
"Reduce alarm noise" → Composite Alarms (AND logic)
"Auto-recover EC2 from hardware failure" → Alarm on StatusCheckFailed_System + Recover action
"Cross-account monitoring" → CloudWatch Cross-Account Observability (OAM)
"Distributed tracing" → X-Ray (not CloudWatch alone)
"Top contributors to high traffic" → Contributor Insights
"Monitor endpoint availability" → CloudWatch Synthetics (Canaries)
"Anomalous metric behavior" → Anomaly Detection
"Cost reduction for logs" → Set retention + Infrequent Access class + S3 archival
"Cross-region dashboard" → Cross-region widgets (API calls per region)
```

---

## 8. Additional Deep-Dive

### 10 More Tricky Scenario-Based Questions

**Q1:** A company has 10 microservices each writing logs to CloudWatch. They want to stream ALL logs to a central OpenSearch domain in a different account for full-text search. But each log group already has 2 subscription filters. How do they add the OpenSearch destination?

**A:** Since the limit is 2 subscription filters per log group, they must consolidate. Options: (1) Remove one existing filter and replace with a **Kinesis Data Stream** that fans out to multiple consumers (OpenSearch + whatever the removed filter was doing). (2) Use **account-level subscription filter** (available for organization-level log delivery) if applicable. (3) Combine the OpenSearch delivery into an existing Kinesis Firehose subscription and add a second delivery stream.

**Keywords:** "2 subscription filters already used", "add more destinations" → Fan out through Kinesis or consolidate filters

---

**Q2:** A company wants to alert when their API error rate (errors/total requests × 100) exceeds 5%, not when absolute error count exceeds a threshold. How?

**A:** Use **Metric Math** in the alarm definition: create an expression `m1/m2*100` where m1 = 5XX count and m2 = total request count. Set the alarm threshold on this expression > 5. This calculates the percentage at alarm evaluation time without needing a custom metric.

**Keywords:** "percentage/ratio alarm", "error rate" → Metric Math expression in alarm

---

**Q3:** A startup wants to monitor their production application but has a very tight budget. They need logs, metrics, and alarms. How should they minimize CloudWatch costs?

**A:** (1) Use **Infrequent Access log class** ($0.25/GB vs $0.50/GB). (2) Set aggressive **log retention** (7-14 days). (3) Use **EMF** from Lambda instead of PutMetricData (free metrics from logs you're already paying for). (4) Use **standard resolution alarms** ($0.10 vs $0.30). (5) Stay within **10 free custom metrics**. (6) Use the **3 free dashboards**. (7) For long-term log storage, stream to **S3** ($0.023/GB vs $0.03/GB).

**Keywords:** "tight budget", "minimize cost" → IA log class + retention + EMF + free tier maximization

---

**Q4:** A company's CloudWatch alarm is flapping (rapidly switching between OK and ALARM states), causing notification spam. How to stabilize it?

**A:** Increase the **evaluation periods** and/or **datapoints to alarm** (e.g., change from "1 of 1" to "3 of 5" — alarm only triggers if 3 out of 5 consecutive periods breach the threshold). This filters out brief spikes. Additionally, consider using **Anomaly Detection** alarm instead of a static threshold if the metric has natural variance.

**Keywords:** "alarm flapping", "notification spam", "brief spikes" → Increase datapoints-to-alarm (M of N evaluation)

---

**Q5:** A company deploys a new feature and wants to detect if error logs increase compared to the previous week's baseline, not against a static threshold. Which CloudWatch feature supports this?

**A:** **CloudWatch Anomaly Detection** — it builds a model based on historical patterns (including weekly seasonality). Create an alarm using the anomaly detection band as the threshold. If errors exceed the learned normal range for that day/hour, the alarm triggers.

**Keywords:** "compared to previous baseline", "not static threshold", "seasonal" → Anomaly Detection

---

**Q6:** A company needs to ensure that CloudWatch log data containing PII (social security numbers, credit card numbers) is automatically detected and masked before anyone can view it. What feature provides this?

**A:** **CloudWatch Logs Data Protection** — configure a data protection policy on the log group with managed data identifiers (SSN, credit card, etc.). Matches are automatically masked in log query results and subscriptions. Audit findings are sent to CloudWatch Logs, S3, or Firehose.

**Keywords:** "PII in logs", "automatically masked", "data protection" → CloudWatch Logs Data Protection policy

---

**Q7:** A company has an ALB and wants to create a dashboard showing request count, latency (p99), and 5XX errors in a single graph with different Y-axes. Is this possible in CloudWatch?

**A:** Yes — use a **dashboard widget with multiple metrics** and enable **second Y-axis** for metrics with different scales. Request count uses left Y-axis, latency uses right Y-axis. 5XX count can share an axis with request count or use its own widget.

**Keywords:** "single graph", "different scales" → Multi-metric widget with dual Y-axes

---

**Q8:** A Lambda function uses X-Ray for tracing. The team also wants to see Lambda invocation metrics (duration, errors, throttles) alongside traces. What service combines both views?

**A:** **CloudWatch ServiceLens** — it integrates CloudWatch metrics, logs, and X-Ray traces into a single view. ServiceLens provides a service map (from X-Ray) annotated with CloudWatch metrics, plus correlated logs for each trace.

**Keywords:** "combine metrics and traces", "single view" → CloudWatch ServiceLens

---

**Q9:** A company's CloudWatch Agent on EC2 instances is configured using an SSM Parameter Store parameter. They update the parameter but agents don't pick up the new configuration. What's needed?

**A:** Updating the SSM parameter doesn't automatically restart the agent. You must use **SSM Run Command** to restart the CloudWatch Agent on all instances (`amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -c ssm:parameter-name -s`). Or use a State Manager association for ongoing configuration enforcement.

**Keywords:** "updated SSM parameter", "agent not picking up changes" → Run Command to restart/refetch agent config

---

**Q10:** A company receives too many emails from CloudWatch Alarms across 20 accounts. They want a single notification channel with alarm aggregation and deduplication. What's the best architecture?

**A:** Route all alarm state-change events via **EventBridge** to a **central monitoring account** (using EventBridge cross-account event bus). In the central account, use EventBridge rules to route events to **Systems Manager Incident Manager** (for deduplication, aggregation, and runbook automation) or to a single SNS topic with Lambda for custom aggregation logic.

**Keywords:** "too many notifications", "aggregation", "deduplication", "multiple accounts" → EventBridge cross-account → Incident Manager

---

### Compare: CloudWatch Logs vs Kinesis Data Streams vs Kinesis Firehose

| Aspect | CloudWatch Logs | Kinesis Data Streams | Kinesis Data Firehose |
|--------|----------------|---------------------|----------------------|
| **Use Case** | Centralized logging, metric extraction, ad-hoc queries | Real-time streaming, custom consumers, sub-second processing | Near-real-time delivery to S3/OpenSearch/Redshift |
| **Latency** | Seconds (ingestion), instant (Log Insights) | Milliseconds (real-time) | 60-900 seconds buffer (near-real-time) |
| **Pricing** | $0.50/GB ingestion + storage | $0.015/shard-hour + $0.014/million PUT | $0.029/GB ingested |
| **HA Model** | Fully managed, multi-AZ | 3-AZ replication per shard | Fully managed, multi-AZ |
| **Exam Tip** | "Store and query logs", "metric filters" | "Real-time processing", "multiple consumers", "custom code" | "Deliver to S3/OpenSearch", "near-real-time", "no code" |

### Compare: Metric Filter vs Contributor Insights vs Log Insights

| Aspect | Metric Filter | Contributor Insights | Log Insights |
|--------|---------------|---------------------|--------------|
| **Use Case** | Create CW metric from log pattern | Top-N contributors in real-time | Ad-hoc log querying/analysis |
| **Output** | CloudWatch metric (for alarms) | Dashboard showing top contributors | Query results (tables, visualizations) |
| **Real-time** | Yes (as logs arrive) | Yes (continuously updated) | No (on-demand query) |
| **Exam Tip** | "Count errors in logs and alarm" | "Find top IP addresses", "identify heavy hitters" | "Search across logs", "ad-hoc analysis" |

### When Does SAP-C02 Expect CloudWatch vs Third-Party?

**Pick CloudWatch when:**
- "AWS-native monitoring"
- "Least operational overhead for AWS resources"
- "Unified metrics and logs"
- "Integrated with Auto Scaling, EC2 actions"

**Pick Amazon Managed Grafana when:**
- "Existing Grafana dashboards"
- "Multiple data sources (Prometheus, CloudWatch, etc.)"
- "Advanced visualization"

**Pick Amazon OpenSearch when:**
- "Full-text search on logs"
- "Complex log analytics with Kibana"
- "Near-real-time log analysis at scale"

**Pick Third-party (Datadog, Splunk, etc.) when:**
- "Multi-cloud monitoring"
- "Existing tool investment"
- These are rarely correct on the exam

### Decision Tree: Choosing the Right Monitoring Approach

```
Need to COLLECT metrics from EC2 (memory/disk)?
  → CloudWatch Agent

Need to EXTRACT metrics from logs?
  → Metric Filter (for alarms) or EMF (for Lambda)

Need REAL-TIME log processing with custom code?
  → Subscription Filter → Kinesis Data Streams → Lambda/App

Need logs DELIVERED to S3 in near-real-time?
  → Subscription Filter → Kinesis Data Firehose → S3

Need AD-HOC log analysis/search?
  → CloudWatch Log Insights (simple) or OpenSearch (complex/full-text)

Need to ALERT on metric threshold?
  → CloudWatch Alarm (single metric) or Composite Alarm (multiple conditions)

Need to ALERT on unexpected behavior (no known threshold)?
  → Anomaly Detection alarm

Need to TRACE requests across services?
  → X-Ray

Need AUTOMATED RESPONSE to alarms?
  → Alarm → SNS → Lambda (custom) or Alarm → EC2 action (built-in)

Need to monitor EXTERNAL endpoints?
  → CloudWatch Synthetics (Canaries)

Need to REDUCE alarm noise?
  → Composite Alarms + ActionsSuppressor
```
