# AWS SAP-C02 Study Notes: Lambda (Concurrency, Layers, Destinations, VPC Integration)

---

## 1. Core Concepts & Theory

### AWS Lambda Fundamentals

AWS Lambda is a **serverless, event-driven compute service** that runs code in response to triggers without provisioning or managing servers. You pay only for the compute time consumed (per-invocation + duration).

#### Execution Model

1. **Event source** triggers invocation (API Gateway, S3, SQS, EventBridge, DynamoDB Streams, etc.)
2. Lambda service receives the event and routes it to an **execution environment**
3. If no warm environment exists → **cold start**: download code, initialize runtime, run init code
4. Handler function executes, returns response
5. Execution environment stays **warm** for reuse (typically 5–15 minutes, not guaranteed)

#### Key Components

| Component | Description |
|-----------|-------------|
| **Function** | Your code + configuration (runtime, handler, memory, timeout, env vars) |
| **Runtime** | Language environment (Python, Node.js, Java, .NET, Go, Ruby, custom via Runtime API) |
| **Handler** | Entry point function that Lambda calls (e.g., `lambda_function.lambda_handler`) |
| **Execution environment** | Isolated container (Firecracker microVM) with your code, runtime, and extensions |
| **Layer** | Reusable package of libraries, custom runtimes, or data — mounted as `/opt` |
| **Alias** | Named pointer to a function version (e.g., `prod` → v5). Supports weighted traffic shifting |
| **Version** | Immutable snapshot of function code + config. `$LATEST` is the only mutable version |
| **Extension** | Companion process running alongside your function (for monitoring, security, governance) |
| **Destination** | Target for async invocation results (success or failure) — SQS, SNS, Lambda, EventBridge |

---

### Concurrency

Concurrency is the **number of function instances processing events simultaneously**.

#### Concurrency Types

| Type | Description | Scope |
|------|-------------|-------|
| **Unreserved concurrency** | Default pool shared by all functions in the account/Region | Account-wide |
| **Reserved concurrency** | Guarantees a set number of concurrent executions for a function; also acts as a **max cap** | Per-function |
| **Provisioned concurrency** | Pre-initializes a set number of execution environments — **eliminates cold starts** | Per-function version or alias |

#### How Concurrency Works

- Each concurrent invocation requires **one execution environment**.
- If all environments are busy, Lambda either:
  - **Scales up** (creates new environment → cold start)
  - **Throttles** (returns 429 TooManyRequestsException) if at the concurrency limit

#### Burst Concurrency

- Lambda scales up in **bursts** — initial burst depends on Region:
  - **3,000** immediate concurrent executions: US East (N. Virginia), US West (Oregon), Europe (Ireland)
  - **1,000** immediate: Asia Pacific (Tokyo), Europe (Frankfurt), US East (Ohio)
  - **500** immediate: All other Regions
- After the initial burst, scales at **500 additional instances per minute** until the account limit is reached.

#### Reserved Concurrency Deep Dive

- Setting reserved concurrency to **0** effectively **disables** the function (all invocations throttled).
- Reserved concurrency is **subtracted** from the account's unreserved pool. If your account limit is 1,000 and you reserve 200 for Function A, all other functions share 800.
- **No additional cost** for reserved concurrency.
- Acts as both a **floor** (guaranteed) and a **ceiling** (cannot exceed) for that function.

#### Provisioned Concurrency Deep Dive

- **Eliminates cold starts** by keeping environments pre-initialized.
- Applied to a **specific version or alias** — NOT to `$LATEST`.
- Can use **Application Auto Scaling** to schedule or target-track provisioned concurrency (e.g., scale up before known peak, scale based on `ProvisionedConcurrencyUtilization` metric).
- **Costs money** even when not processing requests — you pay for provisioned environments.
- If invocations exceed provisioned concurrency, Lambda falls back to standard (on-demand) scaling with possible cold starts.
- **Use with aliases and weighted routing** for canary/linear deployments (CodeDeploy integration).

---

### Lambda Layers

A **Layer** is a ZIP archive containing libraries, a custom runtime, configuration files, or data. Layers let you separate shared dependencies from function code.

#### Key Details

| Fact | Detail |
|------|--------|
| Max layers per function | **5** |
| Max unzipped deployment package (function + all layers) | **250 MB** |
| Layer mount point | `/opt` directory in the execution environment |
| Layer versioning | Each publish creates an **immutable version**. Functions reference a specific layer version |
| Layer sharing | Can be shared with specific accounts, entire AWS Organization, or made public |
| Layer scope | **Region-specific** — must exist in the same Region as the function |
| Supported by | All Lambda runtimes including custom runtimes |

#### Layer Use Cases

- **Shared libraries**: Common SDK packages used by multiple functions (e.g., `boto3` custom version, `pandas`, `numpy`).
- **Custom runtimes**: Package a runtime (e.g., PHP, Rust, COBOL) as a layer using the Runtime API.
- **Configuration/data files**: Shared config, ML model files (within size limits), certificate bundles.

#### Layer Precedence

- If a function includes code at the same path as a layer, the **function's code takes precedence**.
- When multiple layers provide the same file, the **last layer in the list wins**.

---

### Destinations

**Destinations** route the results of **asynchronous invocations** to downstream services without writing custom routing code.

#### Destination Targets

| Event | Supported Targets |
|-------|-------------------|
| **On Success** | SQS, SNS, Lambda, EventBridge |
| **On Failure** | SQS, SNS, Lambda, EventBridge |

#### Destinations vs DLQ (Dead Letter Queue)

| Feature | Destinations | DLQ |
|---------|-------------|-----|
| Trigger type | Async invocations only | Async invocations only |
| Supports success routing | **Yes** | No (failure only) |
| Payload | Full invocation record (request + response + metadata) | Original event only |
| Targets | SQS, SNS, Lambda, EventBridge | SQS, SNS only |
| Recommendation | **Preferred** (newer, richer) | Legacy — use Destinations instead |

#### Important Nuances

- Destinations work only for **asynchronous invocations** (S3 events, SNS, EventBridge, `InvokeAsync`). They do NOT apply to synchronous invocations (API Gateway, ALB) or stream-based invocations (DynamoDB Streams, Kinesis).
- For **stream-based event sources** (DynamoDB Streams, Kinesis), use the **On-Failure Destination** configured on the **event source mapping** — not on the function itself. Targets: SQS or SNS only.
- Destinations and DLQ can coexist on the same function, but **Destinations take precedence** for failure routing if both are configured.
- SQS-triggered Lambda does NOT use Destinations or DLQ on the function — configure the **source SQS queue's DLQ** (redrive policy) instead.

---

### VPC Integration

By default, Lambda functions run in an **AWS-managed VPC** with internet access. You can configure a function to connect to your VPC to access private resources (RDS, ElastiCache, internal APIs).

#### How VPC-Connected Lambda Works (Current Architecture — Hyperplane ENI)

1. Lambda creates a **Hyperplane Elastic Network Interface (ENI)** in each subnet you specify.
2. ENIs are shared across execution environments for the same function (not one ENI per invocation — this changed in 2019).
3. The ENI is created once and **reused** — subsequent cold starts in VPC-attached functions are fast (~1 second, not minutes as in the old model).
4. The function's traffic flows through the ENI into your VPC.

#### Critical VPC Networking Rules

| Scenario | Internet Access? | AWS Service Access? |
|----------|-----------------|-------------------|
| Lambda NOT in VPC | Yes (default) | Yes (public endpoints) |
| Lambda in **public subnet** | **No** — Lambda ENIs don't get public IPs even in public subnets | Only via VPC Endpoints |
| Lambda in **private subnet + NAT Gateway** | **Yes** (via NAT GW) | Yes (via NAT GW or VPC Endpoints) |
| Lambda in **private subnet + VPC Endpoints** | No | Yes (only for services with configured endpoints) |

**The single most important fact:** A Lambda function in a VPC **loses internet access**. To restore it, place the function in a **private subnet** with a route to a **NAT Gateway** in a public subnet. Placing it in a public subnet does NOT work — the ENI does not receive a public IP.

#### Subnet and Security Group Configuration

- Specify **at least 2 subnets in different AZs** for HA (Lambda distributes ENIs across them).
- Lambda functions use the **security group** you assign — control outbound rules to restrict access.
- The ENI must have sufficient IP addresses — Lambda consumes IPs from the subnet. For high-concurrency functions, ensure subnets have large enough CIDR blocks.
- Use `/19` or larger subnets for high-concurrency Lambda functions in VPC.

#### IP Address Consumption (Hyperplane Model)

- Under the Hyperplane ENI model, Lambda creates **far fewer ENIs** than the old model (roughly **one ENI per unique security-group + subnet combination**, not per concurrent execution).
- **Formula (approximate):** ENIs ≈ number of unique (subnet × security group) combinations for the function, NOT proportional to concurrency.
- Still, each ENI consumes one IP from the subnet.

---

### Default Limits, Quotas, and Constraints

| Resource | Default Limit | Adjustable? |
|----------|--------------|-------------|
| **Concurrent executions per Region** | **1,000** | Yes (can request increase to tens of thousands) |
| **Burst concurrency** | 500–3,000 (Region-dependent) | No |
| **Function timeout** | Max **15 minutes (900 seconds)** | No |
| **Memory** | 128 MB – **10,240 MB** (10 GB) in 1 MB increments | No |
| **vCPU** | Proportional to memory: 1,769 MB = 1 vCPU, 10,240 MB ≈ 6 vCPUs | No |
| **Ephemeral storage (`/tmp`)** | 512 MB – **10,240 MB** (10 GB) | No (configurable per function) |
| **Deployment package (zipped)** | **50 MB** (direct upload) / **250 MB** (unzipped, including layers) | No |
| **Container image size** | **10 GB** | No |
| **Layers per function** | **5** | No |
| **Environment variables** | **4 KB** total | No |
| **Invocation payload (sync)** | **6 MB** | No |
| **Invocation payload (async)** | **256 KB** | No |
| **Reserved concurrency minimum** | 0 (disables function) | N/A |
| **Provisioned concurrency** | Up to account concurrency limit | Yes |

---

## 2. Design Patterns & Best Practices

### When to Use Lambda

| Pattern | When Lambda Is the Right Choice |
|---------|-------------------------------|
| **Event-driven processing** | S3 uploads, DynamoDB changes, SQS messages, API requests |
| **Microservices backend** | Stateless API behind API Gateway or ALB |
| **Data transformation** | ETL with Glue triggers, Kinesis stream processing, S3 event processing |
| **Scheduled tasks** | Cron-like jobs via EventBridge Scheduler (≤15 min per execution) |
| **Real-time file processing** | Thumbnail generation, log processing, virus scanning on S3 upload |
| **Orchestration steps** | Step Functions state machine tasks |
| **ChatOps / automation** | Slack bots, auto-remediation on CloudWatch alarms |

### When NOT to Use Lambda (Anti-Patterns)

| Anti-Pattern | Why | Alternative |
|-------------|-----|-------------|
| Long-running processes (>15 min) | Hard timeout limit | ECS/Fargate, Step Functions with ECS tasks, EC2 |
| High-throughput, steady-state compute | More expensive than reserved EC2/Fargate at sustained high volume | ECS/Fargate with Savings Plans, EC2 RIs |
| Workloads requiring GPUs | Lambda doesn't support GPUs | EC2 P/G instances, SageMaker |
| Stateful applications | Lambda is stateless by design; `/tmp` is per-invocation and not shared | ECS, EC2, ElastiCache for state |
| Latency-sensitive startup | Cold starts (especially Java/.NET, VPC-attached) can add 100ms–10s | Provisioned concurrency (mitigates but costs more), Fargate/EC2 |
| Large deployment packages (>10 GB) | Container image limit is 10 GB, ZIP is 250 MB unzipped | ECS/Fargate for very large applications |
| WebSocket long-lived connections | Lambda invocations are request-response, not persistent connections | API Gateway WebSocket API (still uses Lambda but manages connections) or ECS |

### Common Architectural Patterns

#### API Backend
```
Client → API Gateway (REST/HTTP) → Lambda → DynamoDB/RDS
                                  ↓
                           Cognito (auth)
```

#### Event Processing Pipeline
```
S3 PUT → Lambda (transform) → Destination: SQS (success) / SNS (failure)
                             ↓
                       S3 (processed output)
```

#### Fan-Out
```
SNS Topic → Lambda Function A (process)
          → Lambda Function B (index)
          → Lambda Function C (notify)
```

#### Stream Processing
```
Kinesis Data Stream → Lambda (event source mapping, batch of records)
                    → DynamoDB (store results)
                    → On-failure destination: SQS (failed batch)
```

#### Step Functions Orchestration
```
Step Functions → Lambda A (validate) → Lambda B (process) → Lambda C (notify)
              → Catch → Lambda D (error handler)
```

#### Multi-Region Active-Active
```
Route 53 (latency routing)
  → Region A: API Gateway → Lambda → DynamoDB Global Table
  → Region B: API Gateway → Lambda → DynamoDB Global Table
```

### Well-Architected Framework Alignment

**Reliability**
- Use reserved concurrency to protect critical functions from starvation.
- Set reserved concurrency on non-critical functions to prevent them from consuming the entire account pool.
- Configure Destinations or DLQs for async invocations to avoid losing failed events.
- Use multiple subnets across AZs for VPC-attached functions.
- Implement idempotency — Lambda may retry and deliver events more than once.

**Security**
- Use execution roles with least-privilege IAM policies.
- Store secrets in SSM Parameter Store or Secrets Manager — NOT in environment variables (even though they're encrypted, they're visible in the console).
- Use VPC integration only when needed (accessing private resources). Don't put Lambda in VPC unnecessarily.
- Use resource-based policies to control who can invoke the function.
- Enable AWS X-Ray tracing for distributed tracing.

**Performance Efficiency**
- Right-size memory (more memory = more CPU = faster execution; find the cost-performance sweet spot using AWS Lambda Power Tuning).
- Minimize deployment package size — use layers for large dependencies.
- Use provisioned concurrency for latency-sensitive functions.
- Initialize SDK clients and DB connections outside the handler (in the init phase) to reuse across invocations.
- Use `/tmp` for caching between invocations on the same execution environment (but don't depend on it).

**Cost Optimization**
- Don't over-provision memory. Use Lambda Power Tuning to find the optimal setting.
- Use Graviton2 (ARM64) — up to 20% cheaper with better performance for most workloads.
- Avoid unnecessary VPC attachment (adds no cost but can cause throttling from ENI limits).
- Use reserved concurrency (free) instead of provisioned concurrency when cold start is acceptable.
- Batch records from SQS/Kinesis/DynamoDB Streams to reduce number of invocations.

**Operational Excellence**
- Use Lambda versions and aliases for safe deployments.
- Use CodeDeploy with alias traffic shifting (canary/linear) for gradual rollouts.
- Use Lambda extensions for monitoring/security agents.
- Use structured logging (JSON) with CloudWatch Logs.
- Set appropriate timeout — don't use the max 15 minutes for a function that should complete in 5 seconds.

---

## 3. Security & Compliance

### IAM Policies

#### Execution Role (what Lambda CAN do)

The execution role is an IAM role assumed by the function. It defines what AWS services/resources the function can access.

```json
{
  "Effect": "Allow",
  "Action": [
    "dynamodb:PutItem",
    "dynamodb:GetItem"
  ],
  "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/MyTable"
}
```

Required managed policies:
- `AWSLambdaBasicExecutionRole` — CloudWatch Logs (logs:CreateLogGroup, CreateLogStream, PutLogEvents)
- `AWSLambdaVPCAccessExecutionRole` — Required for VPC-attached functions (ec2:CreateNetworkInterface, DescribeNetworkInterfaces, DeleteNetworkInterface)
- `AWSLambdaKinesisExecutionRole` — For Kinesis stream event source mappings

#### Resource-Based Policy (who can INVOKE Lambda)

Controls which AWS accounts, services, or organizations can invoke the function.

```json
{
  "Effect": "Allow",
  "Principal": { "Service": "s3.amazonaws.com" },
  "Action": "lambda:InvokeFunction",
  "Resource": "arn:aws:lambda:us-east-1:123456789012:function:MyFunction",
  "Condition": {
    "ArnLike": { "AWS:SourceArn": "arn:aws:s3:::my-bucket" }
  }
}
```

#### SCP Examples

- **Deny Lambda outside approved Regions:**
```json
{
  "Effect": "Deny",
  "Action": "lambda:*",
  "Resource": "*",
  "Condition": {
    "StringNotEquals": { "aws:RequestedRegion": ["us-east-1", "eu-west-1"] }
  }
}
```

- **Deny Lambda without VPC configuration** (force all functions into VPC):
```json
{
  "Effect": "Deny",
  "Action": ["lambda:CreateFunction", "lambda:UpdateFunctionConfiguration"],
  "Resource": "*",
  "Condition": {
    "Null": { "lambda:VpcIds": "true" }
  }
}
```

### Encryption

| What | Mechanism |
|------|-----------|
| Environment variables | Encrypted at rest with AWS KMS (default service key or CMK). Optionally encrypted in transit with **Lambda encryption helpers** (encrypt before deploy, decrypt in function) |
| Deployment package | Encrypted at rest in S3/Lambda storage |
| `/tmp` ephemeral storage | Encrypted at rest (AES-256) |
| In-transit (invocation payload) | TLS 1.2+ between caller and Lambda service |
| In-transit (VPC) | Traffic within VPC follows VPC encryption rules; ENI traffic is encrypted if using Nitro-based networking |
| Layers | Encrypted at rest in Lambda storage |

### Logging, Auditing, and Monitoring

| Tool | What It Captures |
|------|-----------------|
| **CloudWatch Logs** | Function stdout/stderr, log groups auto-created per function |
| **CloudWatch Metrics** | Invocations, Duration, Errors, Throttles, ConcurrentExecutions, UnreservedConcurrentExecutions, ProvisionedConcurrencyUtilization, ProvisionedConcurrencySpilloverInvocations |
| **CloudTrail** | API calls (CreateFunction, UpdateFunctionCode, Invoke via API, etc.) |
| **X-Ray** | Distributed tracing across Lambda, API Gateway, DynamoDB, SQS, etc. |
| **CloudWatch Lambda Insights** | Enhanced monitoring: CPU, memory, disk, network per invocation (via Lambda extension) |
| **CloudWatch Embedded Metrics Format** | Custom metrics emitted from function code as structured log lines |

### Compliance Patterns

- **Data residency**: Use SCPs to restrict Lambda to specific Regions.
- **Network isolation**: VPC-attach functions + VPC endpoints for AWS service access (no internet traversal).
- **Audit trail**: CloudTrail + CloudWatch Logs. Enable log group encryption with KMS CMK for sensitive workloads.
- **Code signing**: **Lambda Code Signing** — ensures only trusted code is deployed. Uses AWS Signer signing profiles. The function can be configured to warn or enforce on unsigned/untrusted code.

---

## 4. Cost Optimization

### Pricing Model

| Dimension | Price (us-east-1, x86) | Notes |
|-----------|----------------------|-------|
| **Requests** | $0.20 per 1M requests | First 1M requests/month free |
| **Duration** | $0.0000166667 per GB-second | 128 MB × 100ms = 0.0000125 GB-s |
| **Provisioned concurrency** | ~$0.0000041667 per GB-second (idle) + $0.0000097222 per GB-second (active) | Paid even when idle |
| **Ephemeral storage** | $0.0000000309 per GB-second (above 512 MB) | 512 MB included free |
| **ARM64 (Graviton2)** | 20% cheaper on duration | Same request pricing |

### Cost-Saving Strategies

1. **Right-size memory**: Use [AWS Lambda Power Tuning](https://github.com/alexcasalboni/aws-lambda-power-tuning) to find optimal memory. More memory = more CPU = faster execution. Sometimes doubling memory halves duration → same cost but faster.

2. **Use ARM64/Graviton2**: 20% cheaper and often faster. Change the architecture in function config — most Python/Node.js code works without changes. Java/Go/.NET compiled code must be rebuilt for ARM.

3. **Batch processing**: Configure SQS event source mapping with `BatchSize` up to 10,000 and `MaximumBatchingWindowInSeconds` to process more records per invocation.

4. **Avoid unnecessary VPC attachment**: VPC Lambda has no extra cost but can cause throttling if subnet IPs are exhausted, leading to failed invocations and retries.

5. **Use reserved concurrency (free)** instead of provisioned concurrency when cold start latency is acceptable.

6. **Minimize provisioned concurrency**: Use Application Auto Scaling to schedule provisioned concurrency only during peak hours.

7. **Optimize package size**: Smaller packages = faster cold starts = less billed duration during init.

8. **Use Lambda for event-driven, not steady-state**: At ~1M requests/day sustained, EC2/Fargate with Savings Plans becomes cheaper. Lambda's value is in bursty, unpredictable workloads.

### Common Cost Traps

- **Provisioned concurrency always charges** — even at 2 AM when no one is calling the function. Schedule scaling.
- **Timeout misconfig**: Function timeout set to 15 minutes but normally completes in 2 seconds. An error hangs for 15 minutes, burning money. Set timeouts to just above expected p99 duration.
- **Recursive invocations**: Lambda triggers S3, which triggers the same Lambda → infinite loop → massive bill. Use Lambda recursive loop detection (auto-stops after 16 invocations).
- **Over-provisioned memory**: 3 GB allocated but function uses 256 MB — paying 12× more than needed.
- **Forgotten provisioned concurrency**: Set up for a one-time event, never removed.
- **CloudWatch Logs storage**: Lambda logs can accumulate rapidly. Set log group retention policies (default is **Never Expire**).

### Comparison with Alternatives

| Criteria | Lambda | Fargate | EC2 |
|----------|--------|---------|-----|
| Best for | Bursty, event-driven, short-lived | Steady containers, longer tasks | Steady-state, GPU, custom OS |
| Max duration | 15 min | Unlimited | Unlimited |
| Scaling speed | Milliseconds | Seconds–minutes | Minutes |
| Min cost | $0 (idle) | Task-level billing (min 1 min) | Instance-level billing (min 60 sec) |
| Break-even | <1M req/day typically cheaper | >1M steady req/day | Very high steady utilization |

---

## 5. High Availability, Disaster Recovery & Resilience

### HA Architecture

- Lambda is **inherently multi-AZ** — AWS runs your function across multiple AZs within the Region automatically. **You don't manage AZ placement** (unless VPC-attached, where you specify subnets).
- For VPC-attached Lambda: specify **subnets in at least 2 AZs**. Lambda distributes ENIs and invocations across them.
- Lambda itself is a **Regional service** — if the entire Region fails, your functions are unavailable.

### Multi-Region Patterns

| Pattern | Architecture | RPO/RTO |
|---------|-------------|---------|
| **Active-Active** | API Gateway + Lambda in both Regions behind Route 53 latency/failover routing + DynamoDB Global Tables | RPO: near-zero, RTO: seconds (DNS TTL) |
| **Active-Passive** | Primary Region active. DR Region has Lambda deployed (via CI/CD) but not receiving traffic. Route 53 failover with health checks | RPO: depends on data replication, RTO: minutes |
| **Pilot Light** | Lambda code deployed to DR Region. No traffic until failover. DynamoDB Global Tables or Aurora Global Database for data | RPO: seconds–minutes, RTO: minutes |

### Resilience Patterns

#### Async Invocation Retries
- Lambda **automatically retries async invocations twice** (3 total attempts) with delays between retries.
- Configure **Maximum Event Age** (discard events older than X) and **Maximum Retry Attempts** (0, 1, or 2).
- After retries exhausted → sent to **Destination (on-failure)** or **DLQ**.

#### Stream-Based Event Source Retry
- For Kinesis/DynamoDB Streams: Lambda retries the **entire batch** on failure, blocking the shard.
- Configure:
  - **BisectBatchOnFunctionError**: Splits the batch in half to isolate the failing record.
  - **MaximumRetryAttempts**: Number of retries before moving on (default: unlimited → blocks shard forever).
  - **MaximumRecordAgeInSeconds**: Skip records older than this.
  - **On-failure destination**: Send failed batch info to SQS/SNS.
- **Tumbling windows** and **checkpointing** allow partial batch success reporting.

#### SQS Event Source
- Lambda deletes messages from SQS only after **successful processing**.
- On failure: message becomes visible again after the **visibility timeout** expires.
- After `maxReceiveCount` failures → message sent to **SQS DLQ** (configured on the queue, not on Lambda).
- **Partial batch response**: Enable `ReportBatchItemFailures` — Lambda reports which messages failed, only those return to the queue.

#### Idempotency
- Lambda guarantees **at-least-once** delivery for async invocations. Your function MUST be idempotent.
- Use DynamoDB conditional writes, idempotency keys, or the **Powertools for AWS Lambda** idempotency utility.

---

## 6. Exam-Focused Section

### Straightforward Questions (7 examples)

**Q1:** What is the maximum execution timeout for a Lambda function?
**A:** **15 minutes (900 seconds).** This is a hard limit that cannot be increased.

**Q2:** How many layers can a Lambda function use?
**A:** **5 layers.** The total unzipped size of the function + all layers cannot exceed 250 MB.

**Q3:** A Lambda function in a VPC needs to call the DynamoDB API. What should the architect configure?
**A:** **A VPC Gateway Endpoint for DynamoDB** (or a NAT Gateway for internet access, but the gateway endpoint is free and preferred).

**Q4:** What is the default concurrent execution limit for Lambda per Region?
**A:** **1,000.** This is a soft limit that can be increased via a service quota request.

**Q5:** A company wants to eliminate Lambda cold starts for a latency-sensitive API. What feature should they use?
**A:** **Provisioned concurrency.** It pre-initializes execution environments so they are ready to respond immediately.

**Q6:** Where are Lambda Layers mounted inside the execution environment?
**A:** **`/opt` directory.**

**Q7:** What happens to data in Lambda's `/tmp` storage when the execution environment is reused?
**A:** **Data persists** between invocations on the **same** execution environment. It is **not shared** across different environments and is deleted when the environment is terminated.

---

### Tricky / Scenario-Based Questions (7 examples)

**Q1:** A company runs a Lambda function triggered by S3 PUT events. The function processes images and writes results to the same S3 bucket in a different prefix. Occasionally, the function runs thousands of times concurrently and generates a massive bill. What is the MOST likely cause and fix?

**A:** **Recursive invocation loop.** The output write triggers another S3 event that re-invokes the function.
- **Fix:** Use a different output bucket, or configure the S3 event notification with a **prefix filter** that excludes the output prefix. Also enable **Lambda recursive loop detection** (Lambda automatically stops recursive invocations between Lambda, SQS, and SNS after 16 recursions).
- **Why not reserved concurrency?** That would throttle the function but not fix the root cause — events would queue up or be lost.
- **Key phrases:** "same bucket" + "massive bill" + "thousands of invocations" → recursive loop.

**Q2:** A VPC-attached Lambda function cannot reach the internet to call a third-party API. The function is deployed in a public subnet with an Internet Gateway. What is wrong?

**A:** **Lambda ENIs do not receive public IP addresses, even in public subnets.** The function cannot route through the IGW.
- **Fix:** Move the function to a **private subnet** and route traffic through a **NAT Gateway** in the public subnet.
- **Why not Elastic IP?** You cannot assign an EIP to a Lambda ENI.
- **Why not just remove VPC?** If the function needs VPC access for private resources (RDS, ElastiCache), it must stay in the VPC.
- **Key phrases:** "VPC Lambda" + "public subnet" + "cannot reach internet" → NAT Gateway in private subnet.

**Q3:** A company has an account-level Lambda concurrency limit of 1,000. Function A (critical payment processing) occasionally gets throttled because Function B (non-critical analytics) consumes most of the concurrency during peak. How should the architect solve this with MINIMUM cost?

**A:** **Set reserved concurrency on Function A** (e.g., 400) to guarantee it always has capacity. Optionally also set reserved concurrency on Function B to cap it.
- **Why not provisioned concurrency?** Provisioned concurrency eliminates cold starts but costs money. The question asks for minimum cost, and reserved concurrency is **free**.
- **Why not increase account limit?** That works but takes time (quota request) and doesn't protect Function A from Function B in the future.
- **Key phrases:** "throttled" + "another function consuming concurrency" + "minimum cost" → reserved concurrency (free).

**Q4:** A Lambda function processes messages from a Kinesis Data Stream. When one record in a batch causes an error, the entire shard is blocked because Lambda retries the full batch. What TWO configurations should the architect enable?

**A:** **BisectBatchOnFunctionError** (splits the batch to isolate the bad record) and **MaximumRetryAttempts** (limits retries to prevent infinite blocking). Also configure an **on-failure destination** (SQS) to capture the failed record for later analysis.
- **Why not Destinations (on the function)?** Destinations on the function apply to async invocations. Kinesis is a stream-based event source mapping — use the **event source mapping's** on-failure destination.
- **Why not DLQ on the function?** Same reason — DLQ on the function is for async, not stream-based invocations.
- **Key phrase:** "Kinesis" + "entire shard blocked" + "one bad record" → BisectBatchOnFunctionError.

**Q5:** A company needs to process sensitive financial data on Lambda. They want to encrypt environment variables with a customer-managed KMS key and ensure the variables are decrypted only at runtime, not visible in the console. What should they do?

**A:** Use **Lambda encryption helpers** to **encrypt the environment variables at deployment** with a CMK, then **decrypt them in the function code** using the KMS Decrypt API at runtime.
- **Why not just set a CMK on the function config?** That encrypts at rest with the CMK, but variables are still **decrypted and visible in plaintext in the Lambda console**. The encryption helpers add an additional client-side encryption layer.
- **Better alternative for the exam:** Store secrets in **AWS Secrets Manager** or **SSM Parameter Store (SecureString)** and retrieve them at runtime. This is often the preferred exam answer for "secure secret management."
- **Key phrases:** "not visible in console" + "encrypted environment variables" → encryption helpers or Secrets Manager.

**Q6:** A company deploys a Lambda function with provisioned concurrency of 100 on alias `prod`. During a traffic spike, CloudWatch shows `ProvisionedConcurrencySpilloverInvocations` increasing. What does this metric indicate?

**A:** Invocations are **exceeding the provisioned concurrency** and falling back to **on-demand (standard) execution** — which means those requests experience **cold starts**.
- **Fix:** Increase provisioned concurrency or configure **Application Auto Scaling** with a target tracking policy on `ProvisionedConcurrencyUtilization`.
- **Why not an error?** Spillover invocations are NOT errors — they succeed but with potential cold-start latency.
- **Key metric:** `ProvisionedConcurrencySpilloverInvocations` > 0 → provisioned concurrency is insufficient.

**Q7:** A Lambda function processes SQS messages. Some messages repeatedly fail. The architect configures a DLQ on the Lambda function, but failed messages never appear in the DLQ. Why?

**A:** **For SQS-triggered Lambda, the DLQ must be configured on the SQS source queue** (via redrive policy), NOT on the Lambda function. Lambda's DLQ setting applies only to **async invocations**. SQS → Lambda is a **synchronous poll** (event source mapping), not async.
- **Fix:** Configure a redrive policy on the source SQS queue with `maxReceiveCount` and a DLQ. Also enable `ReportBatchItemFailures` on the event source mapping for partial batch success.
- **Key phrases:** "SQS trigger" + "DLQ on Lambda" + "messages don't appear" → DLQ must be on the SQS queue.

---

### Common Exam Traps & Pitfalls

1. **Lambda DLQ vs SQS Redrive Policy**: Lambda's DLQ/Destinations apply to **async invocations only**. For SQS-triggered Lambda, configure the DLQ on the **SQS queue itself**. For Kinesis/DynamoDB Streams, use the **event source mapping's on-failure destination**.

2. **VPC Lambda in public subnet ≠ internet access**: Lambda ENIs never get public IPs. Must use NAT Gateway from a private subnet. This is tested repeatedly.

3. **Reserved concurrency is free; Provisioned concurrency costs money**: Reserved = guarantees AND caps concurrency. Provisioned = pre-warms environments (eliminates cold starts) but charges even when idle.

4. **Provisioned concurrency requires a version or alias**: You CANNOT set provisioned concurrency on `$LATEST`. Must publish a version or create an alias.

5. **Destinations vs DLQ**: Destinations are the newer, richer option (support success routing, more targets, full invocation record). DLQ is legacy. Both work for async only.

6. **Lambda timeout vs API Gateway timeout**: API Gateway has a **29-second hard timeout**. If Lambda takes 30 seconds, the client gets a 504 even if Lambda succeeds. The 15-minute Lambda timeout is irrelevant when invoked via API Gateway.

7. **Layer size counts toward the 250 MB limit**: Function code + all layers combined must be ≤ 250 MB unzipped. Candidates forget that layers count.

8. **Cold start factors**: Largest cold start contributors: VPC attachment (improved but still adds ~1s), runtime (Java/.NET > Python/Node.js > Go/Rust), package size, init code. Provisioned concurrency eliminates ALL of these.

9. **Concurrency ≠ requests per second**: 100 concurrent executions at 1-second duration = 100 requests/second. At 100ms duration, the same 100 concurrency handles 1,000 requests/second. Exam may test understanding of this relationship.

10. **Async invocation payload limit is 256 KB, not 6 MB**: Synchronous is 6 MB. If the question describes large payloads with async invocation, this is a constraint.

11. **Lambda@Edge vs Lambda**: Lambda@Edge has stricter limits (128 MB memory for viewer triggers, 5-second timeout for viewer, 30-second for origin). Don't confuse Lambda@Edge limits with standard Lambda limits.

12. **SQS visibility timeout must be ≥ 6× Lambda timeout**: If Lambda timeout is 60s, visibility timeout should be ≥ 360s. Otherwise, SQS makes the message visible again while Lambda is still processing, causing duplicate processing.

---

## 7. Cheat Sheet

### Must-Know Facts for Exam Day

| Fact | Value |
|------|-------|
| Max timeout | **15 minutes** |
| Max memory | **10,240 MB (10 GB)** |
| Max `/tmp` storage | **10,240 MB (10 GB)** |
| Default concurrency limit | **1,000 per Region** |
| Max layers per function | **5** |
| Max deployment package (unzipped) | **250 MB** (including layers) |
| Max container image | **10 GB** |
| Sync payload limit | **6 MB** |
| Async payload limit | **256 KB** |
| API Gateway timeout | **29 seconds** (hard limit) |
| Async retries | **2** (3 total attempts) |
| Provisioned concurrency target | Version or alias only (NOT `$LATEST`) |
| Reserved concurrency cost | **Free** |
| VPC Lambda internet access | Private subnet + NAT Gateway |
| ARM64 discount | **20% cheaper** |
| Burst concurrency (us-east-1) | **3,000** immediate |
| Post-burst scaling | **500/minute** |

### Key Differentiators

| Lambda Destinations | Lambda DLQ |
|--------------------|-----------|
| Success + failure routing | Failure only |
| SQS, SNS, Lambda, EventBridge | SQS, SNS only |
| Full invocation record | Original event only |
| Preferred (newer) | Legacy |

| Reserved Concurrency | Provisioned Concurrency |
|---------------------|----------------------|
| Free | Paid |
| Guarantees + caps concurrency | Pre-warms environments |
| Does NOT eliminate cold starts | Eliminates cold starts |
| Set on function | Set on version/alias |

| Lambda | Fargate | EC2 |
|--------|---------|-----|
| 15-min max | Unlimited | Unlimited |
| Scales in ms | Scales in seconds | Minutes |
| $0 idle | Task-level billing | Instance billing |
| Event-driven | Container workloads | Full control |

| SQS Trigger Failure | Async Invocation Failure | Stream (Kinesis/DDB) Failure |
|--------------------|------------------------|---------------------------|
| DLQ on SQS queue | DLQ/Destination on function | On-failure destination on event source mapping |
| Redrive policy | MaxRetryAttempts (0-2) | BisectBatchOnFunctionError + MaxRetryAttempts |
| ReportBatchItemFailures | N/A | Checkpointing |

### Decision Flowchart

```
Question says "eliminate cold starts"?
  → Provisioned Concurrency (on a version or alias, NOT $LATEST)

Question says "throttling" + "critical function starved by others"?
  → Reserved Concurrency (free, guarantees + caps)

Question says "Lambda in VPC cannot reach internet"?
  → Private subnet + NAT Gateway (NOT public subnet)

Question says "Lambda needs to access DynamoDB/S3 from VPC"?
  → VPC Gateway Endpoint (DynamoDB, S3 — free)
  → VPC Interface Endpoint (all other services — costs money)

Question says "async invocation failure handling"?
  → Destination (on-failure) — preferred over DLQ

Question says "SQS messages failing repeatedly"?
  → DLQ configured on the SQS QUEUE (not Lambda function)
  → ReportBatchItemFailures for partial batch success

Question says "Kinesis/DynamoDB Stream blocked by bad record"?
  → BisectBatchOnFunctionError + MaximumRetryAttempts + On-failure destination on ESM

Question says "shared libraries across multiple functions"?
  → Lambda Layer (max 5 per function, mounted at /opt)

Question says "gradual deployment / canary"?
  → Alias + weighted traffic shifting + CodeDeploy

Question says "process > 15 minutes"?
  → NOT Lambda. Use Fargate, ECS, EC2, or Step Functions with ECS tasks

Question says "API Gateway + Lambda" + "timeout > 29 seconds"?
  → Cannot work. API Gateway hard limit is 29 seconds. Use ALB (no hard timeout) or async pattern.

Question says "sensitive data processing on Lambda"?
  → Secrets in Secrets Manager/SSM SecureString (NOT env vars)
  → For extreme isolation: Nitro Enclaves (but that's EC2, not Lambda)

Question says "lowest cost" + "event-driven" + "unpredictable traffic"?
  → Lambda (pay per invocation)

Question says "steady high throughput" + "cost-effective compute"?
  → Fargate or EC2 with Savings Plans (NOT Lambda)

Question says "recursive invocation loop"?
  → Separate input/output buckets, prefix filters, Lambda recursive loop detection
```

---

*Generated for AWS SAP-C02 exam preparation. Last updated: 2026-04-26.*
