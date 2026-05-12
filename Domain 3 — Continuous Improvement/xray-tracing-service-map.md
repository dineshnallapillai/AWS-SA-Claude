# X-Ray: Tracing, Service Map

## 1. Core Concepts & Theory

### AWS X-Ray Overview
- **Distributed tracing service** for analyzing and debugging production applications
- Visualizes request flow across microservices, identifies bottlenecks, and pinpoints errors
- Supports: Lambda, ECS, EKS, EC2, Elastic Beanstalk, API Gateway, AppSync, SNS, SQS

### Core Components

**Trace:**
- End-to-end path of a single request through your application
- Composed of one or more **segments** (one per service)
- Unique **Trace ID** propagated via `X-Amzn-Trace-Id` header

**Segment:**
- Represents work done by a single service for a request
- Contains: service name, request/response data, timestamps, subsegments
- Types: computed (inferred from downstream calls) or recorded (sent by instrumented service)

**Subsegment:**
- Granular timing for downstream calls within a segment
- Examples: AWS SDK call, HTTP call, SQL query, custom
- Provides: timing, errors, metadata for individual operations within a service

**Annotations:**
- **Indexed** key-value pairs for filtering traces
- Use for: customer ID, order ID, environment, tenant
- Max 50 annotations per trace
- **Filterable** in X-Ray console (unlike metadata)

**Metadata:**
- **Non-indexed** key-value pairs for additional data
- Use for: request bodies, response payloads, debug info
- NOT filterable — only visible when viewing individual trace details
- No size limit (within segment document limit of 64 KB)

**Groups:**
- Define a filter expression to categorize traces
- Each group can have its own CloudWatch metric (latency, fault rate)
- Example: Group "slow-orders" with filter `responsetime > 3 AND service("order-service")`
- Used for focused alarming on subsets of traffic

**Sampling:**
- Controls WHICH requests are traced (reduces cost and noise)
- **Default rule**: 1 request/second + 5% of additional requests
- **Custom rules**: Define sampling based on service, HTTP method, URL path, etc.
- **Reservoir**: Fixed number of traces per second guaranteed
- **Fixed rate**: Percentage of additional traces beyond reservoir
- Sampling rules can be configured centrally (no app restart needed)

**Service Map:**
- Auto-generated visualization of your application architecture
- Shows services as nodes, connections as edges
- Displays: latency, request counts, error rates, fault rates per service
- Color-coded: green (healthy), yellow (errors), red (faults)
- Updated in near-real-time

### X-Ray Daemon
- Background process that collects segments from instrumented applications
- Listens on **UDP port 2000** (default)
- Batches and sends segments to X-Ray API
- Required on: EC2 instances, ECS (sidecar container), EKS (DaemonSet or sidecar)
- NOT required on: Lambda (built-in), Elastic Beanstalk (pre-installed if enabled)

### X-Ray SDK
- **Instrumentation libraries** for application code
- Available for: Java, Node.js, Python, Go, .NET, Ruby
- Auto-instruments: AWS SDK calls, HTTP requests, SQL queries
- **Patches/wrappers**: Intercept outgoing calls to add subsegments
- Manual instrumentation: Custom subsegments for application logic

### X-Ray Integration by Service

| Service | How to Enable | Daemon Needed? |
|---------|--------------|----------------|
| Lambda | Enable "Active Tracing" in function config | No (built-in) |
| API Gateway | Enable tracing in Stage settings | No |
| ECS/Fargate | X-Ray daemon as sidecar container | Yes (sidecar) |
| EKS | DaemonSet or sidecar | Yes |
| EC2 | Install daemon + SDK in app | Yes |
| Elastic Beanstalk | Enable in EB configuration | No (auto-installed) |
| AppSync | Enable in AppSync settings | No |
| SNS/SQS | Automatic (trace header propagated) | No |
| ALB | Adds trace ID header (no full segment) | No |

### AWS Distro for OpenTelemetry (ADOT)
- **Alternative to X-Ray SDK** — vendor-neutral instrumentation
- Based on OpenTelemetry (CNCF project)
- Can send traces to X-Ray, Jaeger, Zipkin, or other backends
- **AWS recommends ADOT** for new applications (X-Ray SDK for existing)
- Supports auto-instrumentation (Java, Python, Node.js)

### X-Ray Insights
- **Automated anomaly detection** for traces
- Identifies: latency anomalies, error rate spikes, fault increases
- Generates **insights** with root cause analysis
- Integrates with EventBridge for automated response
- No manual threshold configuration needed

### Key Limits

| Limit | Value |
|-------|-------|
| Trace document size | 64 KB |
| Annotations per trace | 50 |
| Segment size | 64 KB |
| Trace retention | 30 days |
| Sampling rules per account | 25 |
| Groups per account | 25 |
| Segment sending rate (daemon) | Batched, 1 MB or 10 segments per batch |
| GetTraceSummaries max | 1,000 per page |

---

## 2. Design Patterns & Best Practices

### When to Use

| Use When | Don't Use When |
|----------|----------------|
| Debug latency issues across microservices | Single monolithic application (simple logging suffices) |
| Identify which service causes errors in a chain | Need log aggregation/search (use CloudWatch Logs) |
| Understand service dependencies | Need infrastructure monitoring (use CloudWatch Metrics) |
| Performance optimization (find bottlenecks) | Need security event monitoring (use GuardDuty) |
| Validate architecture assumptions (actual call graph) | Need real-time alerting on metrics (use CloudWatch Alarms) |

### Anti-Patterns
- **Don't trace 100% of requests in production** — use sampling to control cost and reduce noise
- **Don't use X-Ray for logging** — X-Ray is for tracing; use CloudWatch Logs for detailed log data
- **Don't store sensitive data in annotations** — annotations are visible in the console and filterable
- **Don't rely on X-Ray alone for monitoring** — combine with CloudWatch for metrics/alarms and Logs for debugging
- **Don't use X-Ray SDK for new projects** — AWS recommends ADOT (OpenTelemetry) for new applications

### Architectural Patterns

**Full Observability Stack:**
```
X-Ray (traces) + CloudWatch Metrics (KPIs) + CloudWatch Logs (details)
     ↓                    ↓                          ↓
Service Map       Dashboards/Alarms          Log Insights queries
     ↓                    ↓                          ↓
              CloudWatch ServiceLens (unified view)
```

**Microservices Tracing Pattern:**
```
Client → API Gateway (trace start) → Lambda A → SQS → Lambda B → DynamoDB
                                               → SNS → Lambda C → S3
```
- Trace ID propagated through all services via headers
- Each service adds a segment
- Async services (SQS, SNS) propagate trace context in message attributes

**Cross-Account Tracing:**
- X-Ray supports cross-account traces via CloudWatch cross-account observability (OAM)
- Monitoring account sees unified service map spanning multiple source accounts
- Each source account sends traces to X-Ray in their own account
- Monitoring account aggregates view

**ECS/EKS Sidecar Pattern:**
```
Task/Pod:
  ├── Application Container (instrumented with X-Ray SDK)
  │       └── Sends UDP segments to localhost:2000
  └── X-Ray Daemon Container (sidecar)
          └── Forwards segments to X-Ray API
```

**Lambda Tracing Pattern:**
- Enable Active Tracing (no daemon needed)
- Lambda runtime sends segment automatically (cold start, invocation, overhead)
- SDK adds subsegments for downstream calls
- Trace header passed to downstream services

### Well-Architected Alignment

| Pillar | Practice |
|--------|----------|
| Reliability | Identify failure points in service chain; trace error propagation |
| Security | IAM policies restrict who can view traces; don't trace sensitive data |
| Cost | Sampling rules reduce trace volume; adjust reservoir/fixed rate |
| Performance | Identify latency bottlenecks per service/subsegment |
| Ops Excellence | Service map validates architecture; X-Ray Insights automates anomaly detection |

### Integration Points
- **CloudWatch ServiceLens**: Unified view of metrics + traces + logs
- **CloudWatch Logs**: Correlated log groups shown alongside traces
- **CloudWatch Metrics**: X-Ray publishes service metrics (latency, errors, faults, throttles)
- **EventBridge**: X-Ray Insights generate events for automation
- **API Gateway**: Automatic trace ID injection and segment generation
- **Lambda**: Built-in active tracing support
- **SQS/SNS**: Trace header propagation through messages
- **AppSync**: GraphQL resolver tracing
- **Elastic Beanstalk**: One-click X-Ray daemon deployment
- **ECS/EKS**: Sidecar/DaemonSet daemon patterns

---

## 3. Security & Compliance

### IAM Permissions

**Application (sending traces):**
- `xray:PutTraceSegments` — send segment documents
- `xray:PutTelemetryRecords` — send telemetry (daemon health)
- `xray:GetSamplingRules` — fetch centralized sampling rules
- `xray:GetSamplingTargets` — get sampling decision quotas

**Console/Analysis (viewing traces):**
- `xray:GetTraceSummaries` — list traces
- `xray:BatchGetTraces` — get full trace details
- `xray:GetServiceGraph` — view service map
- `xray:GetInsight` — view X-Ray Insights
- `xray:GetGroups` — list groups

**Cross-Account:**
- Source accounts need IAM role with `xray:PutTraceSegments` (or OAM link)
- Monitoring account uses OAM sink for cross-account visibility
- X-Ray console can display traces from multiple accounts via cross-account observability

### Encryption
- **Traces encrypted at rest** by default (AWS-managed key)
- **Optional**: Customer-managed KMS CMK for trace encryption (`PutEncryptionConfig`)
- **In-transit**: All X-Ray API calls over TLS/HTTPS
- **Daemon-to-API**: HTTPS
- **App-to-Daemon**: UDP (local only — within same host/container, not exposed externally)

### Data Sensitivity
- **Annotations**: Visible in console, filterable — do NOT put PII
- **Metadata**: Visible in trace details — avoid sensitive data
- **Segment content**: SQL queries captured by default — sanitize if needed (`sql.sanitized_query`)
- **Sampling**: Reduce exposure by sampling fewer traces in sensitive environments

### Auditing
- **CloudTrail**: Logs X-Ray API calls (GetTraceSummaries, PutEncryptionConfig, etc.)
- **X-Ray Insights**: Automated detection of anomalies (audit trail of performance issues)
- **Service map history**: Not natively stored — screenshot/export for compliance

---

## 4. Cost Optimization

### Pricing Model

| Item | Price |
|------|-------|
| Traces recorded | $5.00 per 1 million traces |
| Traces retrieved/scanned | $0.50 per 1 million traces |
| X-Ray Insights traces stored | $1.00 per 1 million traces |
| X-Ray Insights event generation | Free (from Insights-stored traces) |
| **Free Tier** | 100,000 traces recorded/month, 1,000,000 traces retrieved/month |

### Cost-Saving Strategies

1. **Sampling rules**: Adjust reservoir and fixed rate to trace only what you need
   - Production: Low rate (1/s + 1-5%)
   - Staging: Higher rate for debugging
   - Specific paths: Higher sampling for critical endpoints
2. **Targeted sampling**: Create rules for specific services/URLs that need detailed tracing
3. **Remove SDK from non-critical services**: Only instrument services you need to trace
4. **Disable active tracing on high-volume Lambda**: If a function runs millions of times/day, the trace cost adds up
5. **Use groups wisely**: Groups create CloudWatch metrics — each group adds metric cost
6. **Trace retention**: Traces auto-expire after 30 days (no configuration needed, but be aware when planning)

### Cost Traps
- **100% sampling in production** — high-volume services generate millions of traces ($5/million)
- **Instrumented batch jobs** that run thousands of invocations per minute
- **Forgetting Lambda active tracing** is enabled on dev functions that never get cleaned up
- **X-Ray Insights** enabled on high-traffic services (additional $1/million for stored traces)

---

## 5. High Availability, Disaster Recovery & Resilience

### Built-in HA
- X-Ray is a **fully managed regional service** — multi-AZ within a region
- No customer-managed infrastructure for HA
- Traces stored durably for 30 days
- Service map auto-regenerates from incoming traces

### Limitations
- **Regional**: Traces only visible in the region where they were recorded
- **Cross-region**: A request that spans regions creates separate trace segments per region
  - Must use cross-account observability or correlate via trace ID manually
- **30-day retention**: Cannot extend — export to S3 if longer retention needed

### DR Considerations
- X-Ray data is **not replicated** across regions
- DR strategy: Instrumented services in DR region automatically send traces to X-Ray in that region
- Service map rebuilds itself as traffic flows in DR region
- **Export traces**: Use `BatchGetTraces` API to export trace data to S3 for long-term retention

### Resilience Patterns
- **X-Ray daemon failure**: If daemon crashes, traces are lost (UDP fire-and-forget), but application is NOT affected (non-blocking)
- **SDK failure**: Instrumented to be non-blocking — if X-Ray API is unavailable, application continues normally
- **Sampling centralization**: If X-Ray API is unreachable for sampling rules, SDK falls back to default rule (1/s + 5%)

---

## 6. Exam-Focused Section

### Straightforward Questions (7 examples)

**Q1:** A company wants to identify which microservice is causing high latency in their order processing pipeline. What should they use?
**A:** **AWS X-Ray** — it provides distributed tracing showing latency at each service in the request chain, plus a service map visualizing the architecture and bottlenecks.

**Q2:** How do you enable X-Ray tracing for a Lambda function?
**A:** Enable **Active Tracing** in the Lambda function configuration (console or `TracingConfig: {Mode: Active}` in CloudFormation). No daemon is needed — Lambda has X-Ray built in.

**Q3:** A team wants to filter X-Ray traces by customer ID to debug a specific customer's issue. How?
**A:** Add the customer ID as an **annotation** (not metadata) in the trace. Annotations are indexed and can be used in filter expressions: `annotation.customer_id = "12345"`.

**Q4:** What port does the X-Ray daemon listen on by default?
**A:** **UDP port 2000**.

**Q5:** A company runs ECS on Fargate and wants to add X-Ray tracing. How should the daemon be deployed?
**A:** Deploy the X-Ray daemon as a **sidecar container** in the same task definition. The application container sends trace segments to `localhost:2000` (UDP).

**Q6:** What is the default X-Ray sampling rule?
**A:** **1 request per second (reservoir) + 5% of additional requests (fixed rate)**. This applies when no custom sampling rules match.

**Q7:** How long does X-Ray retain trace data?
**A:** **30 days**. After that, trace data is automatically deleted.

---

### Tricky / Scenario-Based Questions (7 examples)

**Q1:** A company has a Lambda → SQS → Lambda architecture. The second Lambda function's traces show as disconnected from the first Lambda's traces (two separate traces instead of one connected trace). How to fix?

**A:** Ensure the **trace header is propagated through SQS**. When using the X-Ray SDK, the trace header (`X-Amzn-Trace-Id`) must be passed as a **message attribute** in the SQS message. The consuming Lambda should extract the trace header and continue the trace. With the AWS SDK's built-in X-Ray instrumentation, this should happen automatically — verify the SDK is instrumented on the sending side.

**Why wrong answers are wrong:**
- "Enable active tracing on both Lambdas" — active tracing is necessary but doesn't solve the SQS header propagation issue
- "Use SNS instead of SQS" — both support trace propagation, switching services doesn't fix instrumentation
- "Increase sampling rate" — sampling is about whether to trace, not about trace continuity
- **Keywords:** "disconnected traces", "async (SQS/SNS)" → Trace header propagation through message attributes

---

**Q2:** A company has 50 microservices and wants tracing but is concerned about performance impact and cost. Which configuration minimizes impact while still providing useful debugging data?

**A:** Configure **custom sampling rules** with low fixed rate for high-volume services (e.g., 1/s reservoir + 1% fixed rate) and higher rates for critical/low-volume services. This ensures every service has SOME traces (reservoir guarantees 1/s) without overwhelming cost. The X-Ray SDK adds <1ms overhead per traced request.

**Why wrong answers are wrong:**
- "Disable tracing in production" — loses all production visibility
- "Trace 100% and filter later" — extremely expensive at scale ($5/million traces × millions of requests)
- "Only trace the first and last service" — loses visibility into middle services where bottlenecks often occur
- **Keywords:** "minimize impact", "50 microservices", "cost concern" → Custom sampling rules per service

---

**Q3:** A company uses X-Ray and notices the service map shows a downstream DynamoDB call taking 500ms, but when they test DynamoDB directly, it responds in 5ms. What's happening?

**A:** The 500ms likely includes **cold start time, connection setup, or retry time** — not just DynamoDB latency. Check the subsegment breakdown: the X-Ray segment for a DynamoDB call includes SDK initialization, connection pool establishment, retries, and serialization. Also check if **VPC NAT Gateway** or **VPC endpoint** is adding network latency. Look at the subsegment-level detail to identify whether it's network vs. actual DynamoDB time.

**Why wrong answers are wrong:**
- "X-Ray is inaccurate" — X-Ray measures actual elapsed time, including overhead
- "DynamoDB is throttling" — would show as errors, not just latency (and subsegments would show retry)
- "Sampling is skewing results" — sampling doesn't affect timing measurement
- **Keywords:** "service map shows high latency", "direct test shows low latency" → Look at subsegment breakdown (SDK overhead, network, retries)

---

**Q4:** A team deploys the X-Ray daemon as a DaemonSet on EKS. Some pods' traces are not appearing. Application logs show "unable to send segment to X-Ray daemon." What's the issue?

**A:** The application pod is likely configured to send to `localhost:2000`, but the daemon runs as a DaemonSet (on the node), not as a sidecar. The daemon's UDP port must be accessible at the **node's IP:2000** (or use `AWS_XRAY_DAEMON_ADDRESS` environment variable set to the node IP). Alternatively, switch to a **sidecar pattern** where the daemon is in the same pod (accessible via localhost).

**Why wrong answers are wrong:**
- "IAM permissions" — would show permission errors, not connection errors
- "Sampling rules dropping traces" — would be sampling-related, not connection errors
- "X-Ray is throttling" — would show throttle errors, not connection failures
- **Keywords:** "DaemonSet", "unable to send", EKS → Daemon address configuration (localhost vs node IP)

---

**Q5:** A company wants to use X-Ray tracing AND send traces to their existing Jaeger backend for correlation with on-premises services. What approach works?

**A:** Use **AWS Distro for OpenTelemetry (ADOT)** — it supports multiple trace exporters. Configure ADOT to send traces to BOTH X-Ray AND Jaeger simultaneously. ADOT is the vendor-neutral alternative to the X-Ray SDK.

**Why wrong answers are wrong:**
- "Use X-Ray SDK with Jaeger" — X-Ray SDK only sends to X-Ray, not Jaeger
- "Export traces from X-Ray API to Jaeger" — complex, high-latency, not real-time
- "Run Jaeger agent alongside X-Ray daemon" — requires dual-instrumentation in the app
- **Keywords:** "X-Ray AND other tracing backend", "vendor-neutral" → ADOT (OpenTelemetry)

---

**Q6:** A company uses API Gateway → Lambda → DynamoDB. They enable X-Ray on API Gateway and Lambda. The service map shows API Gateway → Lambda correctly, but DynamoDB calls from Lambda don't appear. Why?

**A:** The **AWS SDK must be instrumented with the X-Ray SDK** (or ADOT) in the Lambda function code. Active Tracing on Lambda only captures the Lambda invocation segment — downstream calls (DynamoDB, S3, etc.) require SDK-level instrumentation to generate subsegments. In Node.js: `AWSXRay.captureAWS(require('aws-sdk'))`.

**Why wrong answers are wrong:**
- "DynamoDB doesn't support X-Ray" — DynamoDB fully supports X-Ray, but the SDK must be wrapped
- "Enable active tracing on DynamoDB" — there's no such setting on DynamoDB itself
- "The Lambda IAM role is missing permissions" — would cause errors, not missing segments
- **Keywords:** "downstream calls not appearing", "Lambda active tracing enabled" → Must instrument AWS SDK in code

---

**Q7:** A company has X-Ray running with default sampling. During a production incident, they want to temporarily capture 100% of requests to debug the issue. How to do this without redeploying?

**A:** Update the **X-Ray sampling rules** in the X-Ray console or API — change the rule's fixed rate to 100% (or set reservoir to a very high number). Sampling rules are **centralized and dynamic** — the SDK fetches updated rules without application restart. After debugging, revert the rule.

**Why wrong answers are wrong:**
- "Redeploy with environment variable" — unnecessary downtime, X-Ray sampling is centrally managed
- "It's not possible without redeployment" — false, centralized sampling rules update in real-time
- "Use CloudWatch Logs instead" — doesn't provide distributed tracing
- **Keywords:** "temporarily 100%", "without redeploying" → Update centralized sampling rules (dynamic, no restart)

---

### Common Exam Traps & Pitfalls

1. **X-Ray daemon NOT needed on Lambda** — Lambda has built-in X-Ray. Daemon is for EC2, ECS, EKS.

2. **Active Tracing ≠ full instrumentation** — Active Tracing on Lambda captures the invocation segment, but you MUST instrument AWS SDK calls to see downstream services.

3. **Annotations are indexed, metadata is NOT** — only annotations can be used in filter expressions. This is heavily tested.

4. **UDP port 2000** — X-Ray daemon listens on UDP (not TCP). Fire-and-forget = no impact if daemon is down.

5. **Sampling rules are centralized** — managed via X-Ray API, fetched by SDK dynamically. No app restart needed to change sampling.

6. **X-Ray ≠ CloudWatch Logs** — X-Ray traces requests across services. CloudWatch Logs stores log events. They complement each other.

7. **ADOT vs X-Ray SDK** — AWS recommends ADOT for new applications. X-Ray SDK is legacy but still supported. If question mentions "vendor-neutral" or "OpenTelemetry," answer is ADOT.

8. **Trace retention = 30 days only** — cannot extend. For long-term, export via API.

9. **Service map is auto-generated** — you don't draw it manually. It builds from trace data.

10. **Cross-account tracing** — requires CloudWatch cross-account observability (OAM). Not automatic.

---

## 7. Cheat Sheet

### Must-Know Facts

| Fact | Detail |
|------|--------|
| X-Ray daemon port | UDP 2000 |
| Trace retention | 30 days |
| Max annotations per trace | 50 |
| Segment max size | 64 KB |
| Default sampling | 1/s reservoir + 5% fixed rate |
| Lambda X-Ray | Active Tracing (no daemon) |
| ECS/EKS X-Ray | Sidecar or DaemonSet |
| Annotations | Indexed, filterable |
| Metadata | NOT indexed, NOT filterable |
| ADOT | OpenTelemetry-based, multi-backend, recommended for new apps |
| Pricing | $5/million traces recorded |
| Free tier | 100K traces/month |
| Sampling rules | Centralized, dynamic (no restart) |
| Cross-account | Via CloudWatch OAM |

### Decision Flowchart

```
"Distributed tracing across microservices" → X-Ray
"Identify latency bottlenecks in request chain" → X-Ray Service Map
"Filter traces by business attribute" → X-Ray Annotations (NOT metadata)
"Debug specific customer issue" → Annotations + filter expression
"New application, vendor-neutral tracing" → ADOT (OpenTelemetry)
"ECS/Fargate tracing" → X-Ray daemon as sidecar
"EKS tracing" → DaemonSet (node-level) or sidecar (pod-level)
"Lambda tracing" → Active Tracing (no daemon)
"Trace through SQS/SNS" → Trace header propagation via message attributes
"Downstream AWS calls not appearing" → Instrument AWS SDK with X-Ray SDK
"Reduce tracing cost" → Custom sampling rules (lower fixed rate)
"Monitor for trace anomalies automatically" → X-Ray Insights
"Combine metrics + traces + logs" → CloudWatch ServiceLens
"Send traces to X-Ray AND Jaeger/Zipkin" → ADOT (multi-exporter)
"Temporarily increase trace capture" → Update centralized sampling rules
```

### Key Differentiators

| vs. Comparison | X-Ray | Alternative |
|----------------|-------|-------------|
| X-Ray vs CloudWatch | Request-level tracing (WHERE in the chain) | Aggregate metrics (WHAT is happening) |
| X-Ray vs CloudWatch Logs | Service dependencies and latency breakdown | Detailed application log messages |
| X-Ray SDK vs ADOT | AWS-specific, mature | Vendor-neutral, multi-backend, recommended for new |
| X-Ray vs Jaeger/Zipkin | Fully managed, AWS-integrated | Self-managed, multi-cloud |
| Annotations vs Metadata | Indexed, filterable, 50 max | Not indexed, no limit, detail only |
| Active Tracing vs SDK | Lambda/API GW invocation capture | Downstream call subsegments |

---

## 8. Additional Deep-Dive

### 10 More Tricky Scenario-Based Questions

**Q1:** A company's service map shows a "client" node making calls to their API, but they didn't instrument any client application. Where is this "client" node coming from?

**A:** The "client" node is an **inferred node** that X-Ray automatically creates when it sees the first segment in a trace (typically from API Gateway or ALB). It represents the caller that initiated the request. This is normal behavior — X-Ray infers downstream and upstream nodes from trace data.

**Keywords:** "unknown client node on service map" → Inferred node (automatically created)

---

**Q2:** A company instruments their Node.js application with the X-Ray SDK but finds that only the first request is traced, and subsequent requests are missing. What's the issue?

**A:** Likely a **sampling issue** — the default sampling rule is 1/s reservoir + 5% fixed rate. If the app receives fewer than 20 requests per second, after the first second's single traced request, only 5% of remaining requests get traced. Increase the reservoir or fixed rate in sampling rules for development/debugging.

**Keywords:** "only first request traced", "subsequent missing" → Default sampling rate (1/s + 5%)

---

**Q3:** A company uses Step Functions to orchestrate Lambda functions. They want to see the entire state machine execution as one trace. How?

**A:** Enable **X-Ray tracing on the Step Functions state machine**. Step Functions natively supports X-Ray — when enabled, it creates a trace that spans the entire execution, with each state as a subsegment. Lambda functions invoked by the state machine add their own segments to the same trace.

**Keywords:** "Step Functions", "single trace for execution" → Enable X-Ray on the state machine

---

**Q4:** A company's X-Ray traces show "Throttle" errors on DynamoDB calls, but DynamoDB CloudWatch metrics show no throttling. What explains this discrepancy?

**A:** The X-Ray "Throttle" subsegment may be capturing **SDK-level retries** that succeed. The AWS SDK retries throttled requests automatically — DynamoDB's CloudWatch `ThrottledRequests` metric only counts requests that ultimately return a throttle error to the caller. If SDK retries succeed, DynamoDB metrics won't show them, but X-Ray captures all attempts (including retried ones).

**Keywords:** "X-Ray shows throttle but CloudWatch doesn't" → SDK retries succeeding (X-Ray captures all attempts)

---

**Q5:** A company uses API Gateway (REST API) with Lambda integration. They enable X-Ray on both. The service map shows API Gateway → Lambda, but the trace shows 200ms total with 50ms in Lambda and 150ms unaccounted for. What's the 150ms?

**A:** The 150ms is **API Gateway overhead**: request validation, mapping templates, authorization, throttle checking, integration setup, and response transformation. X-Ray shows this in the API Gateway segment (not subsegment). Check the segment details for breakdown of API Gateway processing time vs. integration latency.

**Keywords:** "unaccounted latency between API GW and Lambda" → API Gateway internal processing overhead

---

**Q6:** A company has a Java application on EC2 instrumented with X-Ray SDK. After a VPC migration, traces stop appearing in the X-Ray console. The X-Ray daemon is running and the application logs show no errors. What's the likely cause?

**A:** After VPC migration, the X-Ray daemon may not have outbound internet access to reach the X-Ray API endpoint. The daemon needs to reach `xray.REGION.amazonaws.com` (HTTPS/443). Check: (1) Security group allows outbound 443. (2) Route table has internet access (NAT GW) or a **VPC endpoint for X-Ray** (`com.amazonaws.REGION.xray`).

**Keywords:** "VPC migration", "traces stop", "daemon running, no errors" → Network access (NAT GW or VPC endpoint for X-Ray)

---

**Q7:** A company wants per-API-endpoint latency metrics (e.g., p99 for /orders vs /users) without creating custom CloudWatch metrics. Can X-Ray provide this?

**A:** Yes — create **X-Ray Groups** with filter expressions for each endpoint (e.g., `http.url CONTAINS "/orders"`). Each group automatically generates CloudWatch metrics (latency, fault rate, error rate) for traces matching that filter. You can then alarm on these metrics per endpoint.

**Keywords:** "per-endpoint latency metrics", "without custom metrics" → X-Ray Groups (auto-publish CloudWatch metrics)

---

**Q8:** A company instruments a Python Lambda function and sees this error: "Segment already exists. This is usually caused by trying to create a segment when one already exists." What's wrong?

**A:** With **Active Tracing** enabled on Lambda, the Lambda runtime automatically creates a segment (called a "facade segment"). The application code should create **subsegments** (not segments). If the code calls `xray_recorder.begin_segment()`, it conflicts with the Lambda-created segment. Fix: Only use `xray_recorder.begin_subsegment()` in Lambda.

**Keywords:** "segment already exists", "Lambda" → Don't create segments in Lambda; use subsegments only

---

**Q9:** A company wants to trace requests that flow through an Application Load Balancer to EC2 targets. They notice ALB doesn't appear as a full segment on the service map. Why?

**A:** ALB **adds the `X-Amzn-Trace-Id` header** to requests but does NOT send full trace segments to X-Ray. ALB appears as an inferred node on the service map (based on the trace header), not as a fully instrumented service. To get ALB-level timing, use ALB access logs or CloudWatch metrics.

**Keywords:** "ALB not full segment", "inferred node" → ALB adds trace header but doesn't report segments

---

**Q10:** A company has 100 microservices but only wants to trace requests that result in errors (5XX) to reduce cost. Can X-Ray do this?

**A:** Not directly — X-Ray sampling decisions are made at the **beginning** of the request, before the outcome is known. You cannot sample based on response status. Workaround: (1) Use a low sampling rate generally. (2) When errors occur, use annotations/metadata plus filter expressions to quickly find error traces. (3) Use X-Ray Insights (automatically detects error spikes and analyzes contributing traces).

**Keywords:** "only trace errors", "sample based on result" → Not possible (sampling is at request start). Use filters/Insights instead.

---

### Compare: X-Ray vs CloudWatch vs Third-Party

| Aspect | X-Ray | CloudWatch | Datadog APM / New Relic |
|--------|-------|------------|------------------------|
| **Use Case** | Distributed tracing, service maps | Metrics, logs, alarms | Full-stack observability, multi-cloud |
| **Limits** | 30-day retention, 64KB segment | 455-day metrics, unlimited logs | Varies by plan |
| **Pricing** | $5/million traces | $0.50/GB logs, $0.30/metric | Per-host pricing |
| **HA Model** | Fully managed regional | Fully managed regional | SaaS (provider-managed) |
| **Exam Tip** | "Trace requests", "service map", "bottlenecks" | "Metrics", "alarms", "logs" | Only if "multi-cloud" or "existing tooling" |

### Compare: X-Ray SDK vs ADOT (OpenTelemetry)

| Aspect | X-Ray SDK | ADOT |
|--------|-----------|------|
| **Backend** | X-Ray only | X-Ray + Jaeger + Zipkin + OTLP |
| **Languages** | Java, Python, Node, Go, .NET, Ruby | Java, Python, Node, Go, .NET |
| **Auto-instrumentation** | Manual SDK wrapping | Available (Java agent, Python) |
| **AWS recommendation** | Existing applications | New applications |
| **Vendor lock-in** | AWS-specific | Vendor-neutral (CNCF standard) |
| **Exam Tip** | "X-Ray tracing" (default) | "OpenTelemetry", "multi-backend", "vendor-neutral" |

### When Does SAP-C02 Expect X-Ray vs Other Observability?

**Pick X-Ray when:**
- "Distributed tracing"
- "Service map"
- "Identify latency bottlenecks across services"
- "Debug request flow in microservices"
- "Which service is causing failures?"
- "Trace ID"

**Pick CloudWatch when:**
- "Monitor metrics" / "alarms" / "dashboards"
- "Aggregate performance data"
- "Log analysis"
- "Auto Scaling triggers"

**Pick CloudWatch ServiceLens when:**
- "Unified view of metrics AND traces"
- "Combine X-Ray and CloudWatch"

**Pick ADOT when:**
- "OpenTelemetry"
- "Send traces to multiple backends"
- "Vendor-neutral instrumentation"
- "Migrate from Jaeger/Zipkin to X-Ray"

**Pick Third-party (Datadog, etc.) when:**
- "Multi-cloud monitoring"
- "Existing APM investment"
- (Rarely correct on SAP-C02)

### Gotcha Differences: X-Ray Annotations vs Metadata

| Feature | Annotations | Metadata |
|---------|------------|----------|
| Indexed | YES | NO |
| Filterable in console | YES | NO |
| Max per trace | 50 | No limit (within 64KB segment) |
| Use for | Searchable business context (customer ID, order ID) | Debug data, request/response bodies |
| Cost impact | Included in trace cost | Included in trace cost |
| Exam trap | "Filter traces by X" = must be annotation | "Store additional data" = metadata is fine |

### Decision Tree: Observability Tool Selection

```
Need to understand REQUEST FLOW across services?
  └── X-Ray (tracing + service map)
       ├── New application? → ADOT (OpenTelemetry)
       └── Existing application? → X-Ray SDK

Need to ALERT on metric thresholds?
  └── CloudWatch Alarms (NOT X-Ray)

Need to SEARCH/QUERY logs?
  └── CloudWatch Log Insights (or OpenSearch for complex full-text)

Need COMBINED view (metrics + traces + logs)?
  └── CloudWatch ServiceLens

Need to IDENTIFY top contributors to issues?
  └── CloudWatch Contributor Insights

Need to DETECT anomalies in traces?
  └── X-Ray Insights

Need to MONITOR endpoints proactively?
  └── CloudWatch Synthetics

Need PER-ENDPOINT latency metrics?
  └── X-Ray Groups (auto-publishes CW metrics per group)

Need to trace ASYNC workflows (SQS, SNS, Step Functions)?
  └── X-Ray with trace header propagation
       ├── SQS → Message attributes
       ├── SNS → Automatic
       └── Step Functions → Native X-Ray support
```
