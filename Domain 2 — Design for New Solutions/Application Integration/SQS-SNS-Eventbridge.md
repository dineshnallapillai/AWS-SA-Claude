# AWS Messaging & Event-Driven Services: SQS, SNS, EventBridge

## 1. Core Concepts & Theory

---

### Amazon SQS (Simple Queue Service)

**What it is:** Fully managed message queuing service that decouples producers from consumers. Messages are stored until a consumer processes and deletes them.

#### Standard Queue
- **Ordering:** Best-effort ordering (no guarantee)
- **Delivery:** At-least-once delivery (duplicates possible)
- **Throughput:** Nearly unlimited TPS (transactions per second)
- **Use case:** High throughput where occasional duplicates/out-of-order is acceptable

#### FIFO Queue
- **Ordering:** Strict first-in-first-out within a **Message Group ID**
- **Delivery:** Exactly-once processing (deduplication within 5-minute window)
- **Throughput:** 300 messages/sec without batching, 3,000 messages/sec with batching (soft limit, can request increase up to 70,000 msg/sec with high-throughput mode)
- **Naming:** Queue name must end in `.fifo`
- **Message Group ID:** Enables parallel processing of ordered groups — messages within the same group are ordered, different groups are processed in parallel
- **Message Deduplication ID:** Prevents duplicate messages within 5-minute deduplication interval (can use content-based deduplication as alternative)

#### Key Components & Settings

| Setting | Description | Default | Range |
|---------|-------------|---------|-------|
| Visibility Timeout | Time a message is hidden after being read | 30 seconds | 0 sec – 12 hours |
| Message Retention | How long messages are kept | 4 days | 1 minute – 14 days |
| Delay Queue | Time before message becomes visible | 0 seconds | 0 – 15 minutes |
| Max Message Size | Maximum payload | 256 KB | 1 byte – 256 KB |
| Long Polling Wait | WaitTimeSeconds for ReceiveMessage | 0 (short poll) | 0 – 20 seconds |
| Receive Message Wait Time | Queue-level long polling default | 0 | 0 – 20 seconds |

#### Dead-Letter Queue (DLQ)
- A separate SQS queue where messages that fail processing are sent
- **maxReceiveCount:** Number of times a message can be received before being sent to DLQ (1–1000)
- **Redrive Policy:** Configures source queue to send failed messages to DLQ
- **Redrive Allow Policy:** Controls which source queues can use a given DLQ
- DLQ must be the **same type** as source queue (Standard → Standard, FIFO → FIFO)
- DLQ must be in the **same AWS account and region** as the source queue
- **DLQ Redrive to Source:** Allows moving messages from DLQ back to source queue for reprocessing (console feature)
- Set DLQ retention to **14 days** (maximum) to avoid losing messages

#### Visibility Timeout Deep Dive
- When a consumer reads a message, it becomes invisible to other consumers
- If consumer processes successfully → it deletes the message
- If consumer crashes/times out → message becomes visible again after visibility timeout expires
- **ChangeMessageVisibility API:** Consumer can extend visibility timeout if processing takes longer
- **Anti-pattern:** Setting visibility timeout too low → same message processed by multiple consumers
- **Anti-pattern:** Setting visibility timeout too high → long delays before retry on failure

#### Quotas & Limits
- In-flight messages: 120,000 (Standard), 20,000 (FIFO)
- Batch operations: Max 10 messages per batch
- Message attributes: Max 10 metadata attributes per message
- Long polling: Max 20 seconds
- SQS Extended Client Library: For messages > 256 KB (stores payload in S3)

---

### Amazon SNS (Simple Notification Service)

**What it is:** Fully managed pub/sub messaging service. Publishers send messages to Topics, and all subscribers receive a copy (fan-out pattern).

#### Key Characteristics
- **Push-based:** SNS pushes messages to subscribers (unlike SQS which is pull-based)
- **No message persistence:** If subscriber is unavailable, message may be lost (use SQS for durability)
- **Up to 12.5 million subscriptions per topic**
- **Up to 100,000 topics per account** (soft limit)
- **Message size:** Up to 256 KB (use SNS Extended Client Library with S3 for larger)

#### Supported Subscription Protocols
- SQS
- Lambda
- HTTP/HTTPS endpoints
- Email / Email-JSON
- SMS
- Kinesis Data Firehose
- Platform application endpoints (mobile push)

#### Fan-Out Pattern (SNS + SQS)
- SNS topic → multiple SQS queues subscribed
- Each queue gets a copy of every message
- Enables parallel processing by different services
- **Fully decoupled, no data loss** (SQS persists messages)
- Cross-account fan-out is supported via SQS queue policy

#### Message Filtering
- **Subscription Filter Policy:** JSON policy attached to each subscription
- Filters on **message attributes** (not message body) — unless you enable payload-based filtering
- **Payload-based filtering:** Can filter on message body content (JSON)
- Unmatched messages are simply dropped (not sent to that subscriber)
- If no filter policy → subscriber receives ALL messages
- Filter policy applies per-subscription (different subscribers can filter differently)

#### SNS FIFO Topics
- Pairs with SQS FIFO queues for ordered, deduplicated fan-out
- **Supports only SQS FIFO queues as subscribers** (not Lambda, HTTP, etc.)
- Ordering by Message Group ID
- Deduplication by Message Deduplication ID or content-based
- Throughput: 300 publishes/sec (3,000 with batching)

#### SNS Message Delivery Retry Policies
- Different retry policies per protocol (HTTP, Lambda, SQS have different defaults)
- HTTP/S: Configurable retry with backoff (up to 100,015 attempts over 23 days)
- For Lambda, SQS — delivery is highly reliable (within AWS network)

#### Dead-Letter Queue for SNS
- SNS supports DLQs for **subscriptions** (not topics)
- If message delivery to a subscription fails all retries → sent to the subscription's DLQ (an SQS queue)
- Must be configured per-subscription

---

### Amazon EventBridge (formerly CloudWatch Events)

**What it is:** Serverless event bus that connects applications using events from AWS services, SaaS applications, and custom sources.

#### Key Components

**Event Bus:**
- **Default event bus:** Receives events from AWS services automatically
- **Custom event bus:** For your own applications' events
- **Partner event bus:** For SaaS partner integrations (Zendesk, Datadog, Salesforce, etc.)
- **Cross-account event bus:** Send/receive events across accounts

**Rules:**
- Match incoming events and route to targets
- A single event bus can have up to **300 rules** (soft limit)
- A rule can have up to **5 targets**
- Rules use **event patterns** (content-based filtering) or **schedules** (cron/rate)

**Event Pattern Matching:**
- Match on any field in the event JSON
- Supports: exact match, prefix, numeric comparisons, IP address, exists/not-exists, anything-but, wildcard
- **Content-based filtering is FREE** (no cost for rules)

**Targets:**
- Lambda, Step Functions, SQS, SNS, Kinesis Data Streams, Kinesis Firehose, ECS tasks, SSM, API Gateway, Redshift, CodePipeline, CodeBuild, Batch, EC2 (reboot, stop, terminate), Inspector, and more
- **API Destinations:** Send events to any HTTP endpoint (third-party APIs)
- **Input Transformers:** Modify the event before delivering to target

#### EventBridge vs CloudWatch Events
- EventBridge is the evolution of CloudWatch Events (same underlying service)
- EventBridge adds: custom event buses, SaaS integrations, schema registry, archive & replay
- CloudWatch Events rules continue to work and appear in EventBridge console

#### Schema Registry & Discovery
- Auto-discovers and stores event schemas
- Generates code bindings (Java, Python, TypeScript)
- Useful for development — know what events look like

#### Archive & Replay
- Archive events (all or filtered) sent to an event bus
- Replay archived events for debugging, recovery, or testing
- **Retention:** Indefinite or set number of days
- Replay to the same or different event bus

#### Cross-Account & Cross-Region
- Send events to another account's event bus via resource policy
- Cross-region event routing: Put events on a bus in another region
- **Pattern:** Central event bus in a management account for organization-wide events

#### EventBridge Pipes
- Point-to-point integrations with optional filtering, enrichment, and transformation
- Source → (Filter) → (Enrichment) → Target
- Sources: SQS, Kinesis, DynamoDB Streams, Managed Kafka, self-managed Kafka, MQ
- Reduces custom glue code

#### Quotas
- PutEvents: 10,000 entries per request
- Event size: 256 KB max
- Rules per event bus: 300 (soft limit, can increase to 2,000)
- Targets per rule: 5
- Invocations: Soft limit varies by region (default varies, can be millions/sec)

---

## 2. Design Patterns & Best Practices

### When to Use What

| Scenario | Use |
|----------|-----|
| Decouple producer/consumer, buffer load | SQS |
| Fan-out same message to multiple consumers | SNS (+ SQS for durability) |
| Ordered processing | SQS FIFO or SNS FIFO + SQS FIFO |
| React to AWS service state changes | EventBridge |
| Route events based on content to multiple targets | EventBridge |
| Simple notification (email, SMS) | SNS |
| Schedule-based triggers (cron) | EventBridge Scheduler |
| SaaS event integration | EventBridge |
| Point-to-point with enrichment | EventBridge Pipes |
| Cross-account event routing | EventBridge |
| Replay/audit past events | EventBridge (Archive & Replay) |

### Anti-Patterns

| Anti-Pattern | Why It's Wrong | Better Approach |
|--------------|---------------|-----------------|
| Using SNS alone when you need durability | SNS doesn't persist; if subscriber is down, message lost | SNS + SQS fan-out |
| Using SQS when you need fan-out to many consumers | SQS is point-to-point (one consumer per message) | SNS → multiple SQS queues |
| Using EventBridge for high-throughput streaming | EventBridge has event size/rate limits | Kinesis Data Streams |
| Polling SQS with short polling | Wastes API calls, increases cost | Enable long polling (WaitTimeSeconds = 20) |
| Using Standard SQS when order matters | No ordering guarantee | SQS FIFO with Message Group ID |
| Setting visibility timeout = message processing time | No buffer for retries/spikes | Set to 6x average processing time |

### Well-Architected Framework Alignment

**Reliability:**
- SQS DLQ for poison messages
- SNS + SQS fan-out for decoupled resilience
- EventBridge retry policies (up to 185 retries over 24 hours) and DLQs

**Security:**
- SQS: queue policies + IAM + encryption (SSE-SQS or SSE-KMS)
- SNS: topic policies + IAM + encryption (SSE-KMS)
- EventBridge: resource-based policies + IAM for cross-account

**Performance:**
- SQS: Scale consumers horizontally based on `ApproximateNumberOfMessagesVisible`
- Use batch APIs (SendMessageBatch, DeleteMessageBatch) — up to 10 messages
- Long polling to reduce empty receives

**Cost:**
- SQS long polling reduces API calls
- EventBridge content filtering reduces unnecessary Lambda invocations
- SNS filtering avoids delivering messages subscribers don't need

**Operational Excellence:**
- CloudWatch metrics: `ApproximateAgeOfOldestMessage`, `NumberOfMessagesSent`
- EventBridge schema registry for documentation
- DLQ monitoring and alarms

### Integration Patterns

```
┌─────────────────────────────────────────────────────┐
│              Common Integration Patterns              │
├─────────────────────────────────────────────────────┤
│                                                       │
│  S3 Event → EventBridge → Step Functions             │
│  S3 Event → SNS → Multiple Lambdas                   │
│  API Gateway → SQS → Lambda (async processing)      │
│  CloudTrail → EventBridge → SNS → Alert             │
│  EC2 State Change → EventBridge → Lambda             │
│  SNS → SQS (fan-out) → Lambda consumers             │
│  EventBridge Scheduler → Lambda (cron)               │
│  DynamoDB Streams → EventBridge Pipes → SQS         │
│                                                       │
└─────────────────────────────────────────────────────┘
```

---

## 3. Security & Compliance

### SQS Security

**Encryption:**
- **In-transit:** HTTPS endpoints (enforced via `aws:SecureTransport` condition)
- **At-rest:** SSE-SQS (AWS managed, free) or SSE-KMS (customer managed CMK)
- KMS key policy must grant access to any service sending messages (e.g., SNS, S3)

**Access Control:**
- **IAM policies:** Identity-based (who can call SQS APIs)
- **SQS Queue Policy (Resource-based):** Required for cross-account access or allowing AWS services (S3, SNS) to send messages
- **VPC Endpoint (Interface):** Private access from VPC without internet

**Key IAM Actions:**
- `sqs:SendMessage`, `sqs:ReceiveMessage`, `sqs:DeleteMessage`
- `sqs:ChangeMessageVisibility`, `sqs:GetQueueAttributes`

### SNS Security

**Encryption:**
- SSE-KMS for topics at rest
- HTTPS enforced via condition key `aws:SecureTransport`
- Messages to SQS subscribers: SQS queue must have KMS permissions if encrypted

**Access Control:**
- **SNS Topic Policy:** Cross-account publish/subscribe, allow AWS services
- **IAM policies:** Control who can publish, subscribe, manage topics
- **Subscription confirmation:** HTTP/email endpoints require opt-in (prevents spam)

### EventBridge Security

**Access Control:**
- **Resource-based policy on event bus:** Cross-account PutEvents, Manage rules
- **IAM roles for targets:** EventBridge assumes a role to invoke targets
- **Organization-level access:** `aws:PrincipalOrgID` condition for org-wide access

**Encryption:**
- Events encrypted at rest using AWS-owned keys by default
- Can use customer-managed KMS keys for custom/partner event buses

### SCP Considerations
- SCP can deny `sqs:*` or `sns:*` in specific accounts
- Deny `events:PutEvents` to prevent accounts from sending events to central bus
- Enforce encryption: deny `sqs:CreateQueue` without `SqsManagedSseEnabled` or `KmsMasterKeyId`

### Logging & Monitoring

| Service | CloudTrail | CloudWatch Metrics | CloudWatch Logs |
|---------|------------|-------------------|-----------------|
| SQS | API calls logged | Yes (message counts, age, size) | No native (use Lambda consumer logging) |
| SNS | API calls logged | Yes (publishes, deliveries, failures) | Yes (delivery status logging) |
| EventBridge | API calls logged | Yes (invocations, failed invocations, throttles) | Yes (can log matched events) |

---

## 4. Cost Optimization

### SQS Pricing
- **Standard:** ~$0.40 per million requests (first 1M free/month)
- **FIFO:** ~$0.50 per million requests (first 1M free/month)
- **Request = any API action** (Send, Receive, Delete, ChangeVisibility all count)
- A single request can batch up to 10 messages (batch = 1 request)
- 64KB chunk pricing: A 256KB message = 4 requests

**Cost Optimization:**
- Use long polling (reduces empty ReceiveMessage calls)
- Batch operations (10 messages/request)
- Delete messages promptly (avoid re-delivery billing)
- Use Standard over FIFO when strict ordering not needed

### SNS Pricing
- Publishes: ~$0.50 per million
- Deliveries vary by protocol: SQS = free, HTTP = $0.60/million, Lambda = free, SMS = per-message
- Data transfer charges apply for cross-region
- **Message filtering is free** — filter at SNS, not at consumer

### EventBridge Pricing
- **Custom events:** $1.00 per million events published
- **AWS service events:** FREE
- **Cross-account events:** $1.00 per million (paid by sender)
- **Schema discovery:** Free for AWS events, $0.10 per million custom events
- **Archive:** Storage cost + replay cost
- **Pipes:** $0.40 per million events
- **Scheduler:** $1.00 per million invocations (first 14M free/month)

**Cost Optimization:**
- Use EventBridge rules filtering to reduce downstream invocations
- Archive only what you need (set retention)
- EventBridge Scheduler is cheaper than CloudWatch Events for scheduled tasks
- AWS service events are free — prefer EventBridge over polling for state changes

### Cost Traps
- SQS short polling with fast poll loops = massive API costs
- Large messages (> 64KB) count as multiple requests in SQS
- SNS SMS charges can explode (use spend limits)
- Not deleting processed SQS messages = unnecessary re-receives = cost
- EventBridge archive without retention policy = growing S3 costs

---

## 5. High Availability, Disaster Recovery & Resilience

### SQS HA
- **Multi-AZ by default:** Messages stored redundantly across multiple AZs in a region
- **No configuration needed** for HA within a region
- **Cross-region:** No native replication — implement with Lambda reading from source and sending to destination queue
- **Queue is regional** — if region goes down, queue is unavailable

### SNS HA
- **Multi-AZ by default:** Messages stored across multiple AZs
- **Cross-region fan-out:** SNS in Region A → SQS in Region B (subscriber)
- Topic is regional

### EventBridge HA
- **Multi-AZ by default**
- **Global Endpoints:** Route events to secondary region during failover
  - Uses Route 53 health checks to detect failure
  - Active/passive or active/active event ingestion
  - Automatic failover and failback
  - Events replicated to secondary region
- **Cross-region event routing:** Rules can target event buses in other regions

### Resilience Patterns

**Pattern: Guaranteed Processing with DLQ**
```
Producer → SQS Queue → Consumer (Lambda)
              ↓ (after maxReceiveCount failures)
           DLQ → CloudWatch Alarm → Ops team
```

**Pattern: Multi-Region Event-Driven**
```
Region A: EventBridge → (Global Endpoint replicates) → Region B: EventBridge
                                                              ↓
                                                         Same Rules & Targets
```

**Pattern: Poison Message Handling**
```
SNS Topic → SQS Queue (maxReceiveCount=3) → Lambda
                 ↓ (failures)
              DLQ → Lambda (error handler) → Alert + Store for analysis
```

### RPO/RTO Considerations

| Scenario | RPO | RTO | Strategy |
|----------|-----|-----|----------|
| SQS regional failure | Messages in-flight lost | Minutes (failover to DR queue) | Cross-region with application-level replication |
| EventBridge regional failure | Near-zero (with Global Endpoints) | Seconds-minutes | Global Endpoints with health check |
| SNS topic failure | N/A (no persistence) | Minutes | Recreate topic, re-subscribe |

### Retry & Error Handling

| Service | Retry Behavior |
|---------|---------------|
| SQS | Consumer controls retries (visibility timeout → reappear → retry) |
| SNS | Protocol-specific retry policy (HTTP: exponential backoff, up to 23 days) |
| EventBridge | Up to 185 retries over 24 hours with exponential backoff, then DLQ |

---

## 6. Exam-Focused Section

### Straightforward Questions

**Q1:** A company wants to decouple their order processing system. Orders must be processed exactly once in the sequence they were placed. Which SQS configuration should they use?

**A:** SQS FIFO queue with a single Message Group ID for all orders. FIFO guarantees ordering and exactly-once processing.

---

**Q2:** An application publishes events to an SNS topic. Three microservices need to independently process each event. What pattern should be used?

**A:** SNS fan-out pattern — subscribe 3 SQS queues to the SNS topic. Each queue receives a copy of every message.

---

**Q3:** A team wants to trigger a Lambda function whenever an EC2 instance changes state (running, stopped, terminated). What service should they use?

**A:** Amazon EventBridge with a rule matching EC2 Instance State-change Notification events, targeting the Lambda function. EC2 state changes automatically go to the default event bus.

---

**Q4:** Messages in an SQS queue are being processed multiple times. The visibility timeout is set to 30 seconds and processing takes 45 seconds on average. What should be done?

**A:** Increase the visibility timeout to at least 270 seconds (6x processing time) or use ChangeMessageVisibility API to extend it during processing.

---

**Q5:** A company wants certain SNS subscribers to only receive messages about orders over $1000. How can this be achieved without modifying publishers?

**A:** Use SNS subscription filter policies that filter on a message attribute (e.g., `order_value`) with a numeric condition `>= 1000`.

---

**Q6:** An application needs to capture AWS API calls and trigger remediation Lambda functions when specific actions occur. What should be used?

**A:** CloudTrail → EventBridge rule (matching specific API calls) → Lambda function. CloudTrail events are delivered to the default event bus.

---

**Q7:** A FIFO queue has a DLQ. After investigating failures, the team wants to reprocess DLQ messages in order. What's the simplest approach?

**A:** Use the SQS DLQ Redrive feature to move messages back to the source FIFO queue, which will maintain ordering via Message Group ID.

---

### Tricky / Scenario-Based Questions

**Q1:** A company processes 50,000 financial transactions per second. Transactions must be processed exactly once. A solutions architect recommends SQS FIFO. Is this correct?

**A:** **No.** SQS FIFO supports up to 70,000 msg/sec with high-throughput mode, BUT exactly-once + ordering at that scale requires careful Message Group ID design. However, the real issue: if ALL transactions must be globally ordered (single Message Group), throughput drops to 300-3,000/sec. If they can be partitioned by customer/account (multiple Message Groups), FIFO can work. **Key phrase to watch:** "must be processed in exact sequence" (single group = low throughput) vs. "each customer's transactions in order" (partitioned = higher throughput).

---

**Q2:** An application uses SNS to fan out to 3 Lambda functions. During peak load, one Lambda is throttled and messages are lost. How do you prevent message loss?

**A:** Insert SQS queues between SNS and Lambda (SNS → SQS → Lambda for each). SQS provides durability; if Lambda is throttled, messages persist in the queue. **Wrong answer:** "Increase Lambda concurrency" — this helps but doesn't guarantee no loss during extreme throttles. **Wrong answer:** "Enable SNS DLQ" — this helps after retries exhaust, but SNS → SQS is the robust pattern.

---

**Q3:** A company has EventBridge rules routing events to Lambda. They notice that some events are processed twice. Why, and how do they fix it?

**A:** EventBridge guarantees at-least-once delivery, so duplicates are expected. **Fix:** Make the Lambda function idempotent (use DynamoDB conditional writes or idempotency tokens). **Wrong answer:** "Switch to SQS FIFO" — this changes the architecture and FIFO only helps if you control the source. **Key concept:** At-least-once is a feature of EventBridge, not a bug.

---

**Q4:** A team needs to send S3 event notifications to both an SQS queue and a Lambda function. The S3 bucket notification configuration only allows one destination per event type. What's the best approach?

**A:** Configure S3 to send events to SNS topic (or EventBridge), then subscribe both SQS and Lambda to the topic/bus. **Key detail:** S3 can actually send the same event type to different destinations IF the prefixes/suffixes differ. But if they truly need the SAME event to go to both, use SNS fan-out or EventBridge. **Alternative (best in 2024+):** Enable S3 Event Notifications to EventBridge (setting on bucket) — then create two EventBridge rules.

---

**Q5:** An organization has 20 AWS accounts. They want a central account to receive all EC2 termination events from member accounts. Which approach is most scalable?

**A:** Use EventBridge cross-account event routing. In each member account, create a rule on the default event bus that sends EC2 termination events to the central account's custom event bus. Use an organization-level resource policy (`aws:PrincipalOrgID`) on the central bus. **Wrong answer:** "Use SNS topics in each account publishing to a central SQS queue" — this works but requires managing 20 SNS topics and cross-account subscriptions. EventBridge is purpose-built for this. **Wrong answer:** "Use CloudWatch Events" — CloudWatch Events IS EventBridge, but the answer should reference EventBridge for cross-account.

---

**Q6:** A company uses SQS with a maxReceiveCount of 5 and visibility timeout of 60 seconds. A consumer occasionally takes 90 seconds to process. Messages appear in the DLQ that were successfully processed. Why?

**A:** The visibility timeout (60s) expires before processing completes (90s). The message becomes visible again, gets picked up by another consumer, and the receive count increments. After 5 receives (not 5 failures), it goes to DLQ. **Fix:** Increase visibility timeout OR use ChangeMessageVisibility API. **Key trap:** maxReceiveCount counts receives, not failures. A message can be "successfully" processed but still hit DLQ if visibility timeout is wrong.

---

**Q7:** A team is choosing between EventBridge and SNS for an event-driven architecture. Events come from 5 microservices and need to be routed to different targets based on event content. Some targets are third-party HTTP APIs. Which should they choose?

**A:** EventBridge. **Reasons:** (1) Content-based filtering/routing built-in, (2) API Destinations for third-party HTTP APIs with auth management, (3) Schema registry for discoverability. **Why not SNS:** SNS filter policies work for simple attribute filtering, but EventBridge offers richer pattern matching on body content, and API Destinations handle auth/rate-limiting to external APIs. **Key phrases:** "content-based routing" + "third-party APIs" = EventBridge.

---

### Common Exam Traps & Pitfalls

1. **SQS Standard "at-least-once" vs FIFO "exactly-once":** The exam loves testing whether you know Standard can deliver duplicates. If the question says "must not process duplicates" → FIFO.

2. **FIFO throughput limits:** If the question mentions high throughput (>3,000/sec without batching) AND strict ordering → check if Message Group partitioning is acceptable. If not, FIFO may not work.

3. **SNS alone ≠ durable:** If a question asks about guaranteed delivery/no message loss, SNS alone is wrong. It needs SNS + SQS or a DLQ.

4. **EventBridge vs CloudWatch Events:** They're the same service. If an answer says "migrate from CloudWatch Events to EventBridge for cross-account," that's valid — but they share the same API under the hood.

5. **DLQ is NOT automatic:** You must configure it. And a DLQ for SQS is on the source queue, while a DLQ for SNS is on the subscription, and for EventBridge it's on the rule/target.

6. **Visibility Timeout vs Delay Queue:** Visibility timeout = after message is received. Delay Queue = before message is available the first time. Don't confuse them.

7. **SQS FIFO + SNS FIFO:** SNS FIFO can ONLY deliver to SQS FIFO queues. If the question mentions Lambda subscribers with SNS FIFO → wrong answer.

8. **EventBridge rules limit targets to 5:** If you need more than 5 targets for one event pattern, use multiple rules or fan-out through SNS/SQS.

9. **Message Group ID in FIFO:** All messages with the same Message Group ID are ordered. Different Group IDs = parallel processing. If the question says "different customers can be processed in parallel" → use customer ID as Message Group ID.

10. **S3 → EventBridge vs S3 → SNS/SQS:** S3 can send events directly to SNS, SQS, or Lambda (bucket notification). It can ALSO send events to EventBridge (must be enabled on the bucket). EventBridge gives richer filtering and more targets.

---

## 7. Cheat Sheet

### Must-Know Facts

| Fact | Detail |
|------|--------|
| SQS max message size | 256 KB (use Extended Client + S3 for larger) |
| SQS max retention | 14 days |
| SQS FIFO throughput | 300/sec (3,000 batched, 70,000 high-throughput) |
| SQS in-flight limit | 120,000 Standard, 20,000 FIFO |
| SQS visibility timeout range | 0 seconds – 12 hours |
| SNS max subscriptions/topic | 12.5 million |
| SNS FIFO subscribers | SQS FIFO only |
| EventBridge event size | 256 KB max |
| EventBridge retry | 185 times over 24 hours |
| EventBridge rules/bus | 300 (soft), targets/rule = 5 |
| EventBridge AWS events cost | FREE |
| EventBridge custom events cost | $1/million |

### Key Differentiators

| Feature | SQS | SNS | EventBridge |
|---------|-----|-----|-------------|
| Model | Pull (poll) | Push (pub/sub) | Push (event-driven) |
| Persistence | Yes (up to 14 days) | No | Archive & Replay |
| Ordering | FIFO only | FIFO topics only | No guarantee |
| Filtering | No (consumer-side) | Attribute-based filter policies | Rich content-based rules |
| Fan-out | No (single consumer) | Yes (all subscribers) | Yes (multiple targets) |
| Cross-account | Queue policy | Topic policy | Event bus policy |
| DLQ location | Source queue config | Subscription config | Rule/target config |
| Max throughput | Nearly unlimited (Standard) | Soft limits | Soft limits |
| Third-party targets | No | HTTP endpoints | API Destinations (managed) |
| Scheduling | No | No | Yes (Scheduler) |

### Decision Flowchart

```
Question mentions...                           → Think...
─────────────────────────────────────────────────────────────
"decouple", "buffer", "queue"                  → SQS
"exactly-once", "ordered", "FIFO"              → SQS FIFO
"fan-out", "multiple subscribers"              → SNS (+ SQS for durability)
"filter by attribute"                          → SNS filter policy
"AWS service event", "state change"            → EventBridge
"cron", "schedule"                             → EventBridge Scheduler
"cross-account events"                         → EventBridge
"content-based routing"                        → EventBridge rules
"SaaS integration"                             → EventBridge partner bus
"replay events", "audit trail"                 → EventBridge Archive & Replay
"message lost", "subscriber unavailable"       → Add SQS between SNS and consumer
"processing duplicates"                        → SQS FIFO or idempotent consumer
"messages going to DLQ unexpectedly"           → Check visibility timeout vs processing time
"high throughput + ordering"                   → Partition with Message Group IDs
"third-party HTTP endpoint"                    → EventBridge API Destinations
"global/multi-region events"                   → EventBridge Global Endpoints
```

### Quick Comparison: "When the question says X, the answer is Y"

- "Decouple microservices, one producer one consumer" → **SQS Standard**
- "Process each order exactly once in order" → **SQS FIFO**
- "Notify multiple services of the same event" → **SNS fan-out to SQS**
- "React to EC2/S3/RDS state changes" → **EventBridge**
- "Route events to different targets based on content" → **EventBridge**
- "Send events to a third-party API with rate limiting" → **EventBridge API Destinations**
- "Schedule a Lambda every 5 minutes" → **EventBridge Scheduler**
- "Guaranteed no message loss with multiple consumers" → **SNS + SQS**
- "Process failed messages later" → **Dead-Letter Queue**
- "Cross-account event aggregation" → **EventBridge cross-account**

---

## 8. Deep Dive: 10 More Tricky Scenario-Based Questions

**Q1:** A company uses SNS to deliver messages to an SQS queue in another account. Messages are encrypted with a customer-managed KMS key (CMK) in the SNS account. The SQS queue in the target account cannot decrypt the messages. What's wrong?

**A:** The KMS key policy in the SNS account must grant `kms:Decrypt` and `kms:GenerateDataKey` to the SQS service principal in the target account. Additionally, the SQS queue's KMS key (if encrypted) must allow SNS to generate data keys. **Key trap:** Cross-account encrypted messaging requires KMS key policies in BOTH accounts to allow the respective services. Simply sharing the SNS topic isn't enough when KMS is involved.

---

**Q2:** An application sends 5,000 messages/sec to an SQS FIFO queue using a single Message Group ID. The queue processes messages slowly and a backlog builds. Adding more consumers doesn't help. Why?

**A:** With a single Message Group ID, all messages must be processed **sequentially by one consumer** (FIFO ordering guarantee). Additional consumers are idle because only one consumer can process messages from a single message group at a time. **Fix:** Redesign to use multiple Message Group IDs (e.g., per customer, per category) to enable parallel processing while maintaining order within each group. **Key concept:** Parallelism in FIFO = number of unique Message Group IDs, NOT number of consumers.

---

**Q3:** A team configures S3 event notifications to send to both an SNS topic and an SQS queue for the same event type (`s3:ObjectCreated:*`) on the same prefix. Configuration fails. Why?

**A:** S3 event notification configuration does NOT allow the same combination of event type + prefix + suffix to be sent to multiple destinations. **Fix:** (1) Send to SNS topic, then fan out to SQS + other consumers. (2) Enable EventBridge on the bucket (all S3 events go to EventBridge), then create multiple rules. (3) Use different prefix/suffix filters for each destination. **Key rule:** One event type + prefix + suffix combination = one destination in S3 bucket notification config.

---

**Q4:** A company uses EventBridge to trigger a Lambda function when an IAM policy is changed. The rule works in the us-east-1 region but not in eu-west-1 where the company operates. Why?

**A:** IAM is a **global service** and its API calls are logged by CloudTrail only in **us-east-1**. Therefore, IAM events appear on the EventBridge default bus only in us-east-1. **Fix:** Create the rule in us-east-1, or configure cross-region event routing from us-east-1 to eu-west-1. **Key trap:** Global service events (IAM, STS, CloudFront) are only available in us-east-1 EventBridge.

---

**Q5:** An application sends messages to SQS with a Delay Queue set to 15 minutes. A consumer reads the message and sets the visibility timeout to 5 minutes. The consumer fails after 3 minutes. When will the message become visible again?

**A:** The message becomes visible 5 minutes after it was received (not after failure). The delay queue only applies to the **initial delivery**. Once the message has been received, the visibility timeout governs reappearance. So the message reappears **2 minutes after the consumer fails** (5 min visibility timeout - 3 min processing = 2 min remaining). **Key distinction:** Delay Queue = initial wait before first visibility. Visibility Timeout = wait after each receive.

---

**Q6:** A company has an EventBridge rule that targets 5 Lambda functions. They need to add a 6th target. What are their options?

**A:** EventBridge limits each rule to **5 targets**. Options: (1) Create a second rule with the same event pattern targeting the 6th Lambda. (2) Have one target be SNS or SQS, then fan out to additional consumers from there. (3) Target a Step Functions state machine that orchestrates all 6 Lambdas. **Key fact:** Multiple rules can match the same event — the event is delivered to all matching rules' targets. Duplicate rules with the same event pattern is perfectly valid.

---

**Q7:** A microservices architecture uses SNS topics for inter-service communication. During deployment of Service B, its SQS subscription is deleted and recreated. Messages published during the gap are lost. How do you prevent this?

**A:** **Option 1:** Never delete the subscription — update the endpoint instead. **Option 2:** Use infrastructure as code (CloudFormation/CDK) to ensure subscription always exists. **Option 3:** Configure SNS delivery retry policy (but SNS only retries for active subscriptions, not deleted ones). **Best fix:** Ensure subscriptions are persistent resources managed via IaC, not recreated during deployments. **Key concept:** If a subscription doesn't exist, messages published during that window are permanently lost (SNS has no persistence).

---

**Q8:** A company uses SQS with Lambda event source mapping. The Lambda function processes batches of 10 messages. If one message in the batch fails, all 10 messages return to the queue and are reprocessed. How do they avoid reprocessing the 9 successful messages?

**A:** Enable **Partial Batch Response** (`ReportBatchItemFailures`). Lambda returns only the failed message IDs in `batchItemFailures`. SQS only returns those specific messages to the queue; successfully processed messages are deleted. **Without this feature:** The entire batch fails or succeeds as a unit. **Key config:** Set `FunctionResponseTypes` to include `ReportBatchItemFailures` on the event source mapping.

---

**Q9:** An organization uses EventBridge with a custom event bus. Events from Account A must be processed by rules in Account B's custom event bus. Events are arriving at Account B's bus but no rules are matching. What's likely wrong?

**A:** When events are forwarded cross-account, the **event `source` and `detail-type` fields are preserved**, but the rule in Account B must match the event pattern exactly as it arrives (including the original source). Check: (1) The rule's event pattern matches the incoming event structure. (2) The event isn't wrapped in an additional envelope. **Common mistake:** Rules expecting `source: "aws.events"` (the forwarding service) instead of the original source (e.g., `source: "custom.myapp"`). Cross-account forwarding preserves the original event structure.

---

**Q10:** A company uses SQS Standard queue with a Lambda consumer. They notice that `ApproximateAgeOfOldestMessage` is increasing steadily. The Lambda function is not erroring. What's happening?

**A:** Possible causes: (1) **Lambda concurrency limit reached** — messages queue up because Lambda can't scale to meet demand. (2) **Batch window configured** — Lambda waits to fill batches before processing. (3) **Reserved concurrency set too low** for the Lambda function. (4) **Event source mapping disabled or throttled.** **Fix:** Check Lambda concurrent executions metric, increase reserved concurrency, or increase Lambda scaling configuration (MaximumConcurrency on SQS event source). **Key metric:** `ApproximateAgeOfOldestMessage` growing = consumers aren't keeping up, not necessarily errors.

---

## 9. Service Comparison Tables

### SQS vs SNS vs EventBridge

| Dimension | SQS | SNS | EventBridge |
|-----------|-----|-----|-------------|
| **Use Case** | Decouple, buffer, async processing | Fan-out notifications | Event routing, AWS service reactions |
| **Max Message/Event Size** | 256 KB | 256 KB | 256 KB |
| **Throughput Limit** | Unlimited (Standard), 70k/s (FIFO) | Soft limits (varies by region) | Soft limits (varies by region) |
| **Pricing** | $0.40/million requests | $0.50/million publishes | $1.00/million custom events; AWS events FREE |
| **HA Model** | Multi-AZ, regional | Multi-AZ, regional | Multi-AZ, regional + Global Endpoints |
| **Ordering** | FIFO queues only | FIFO topics only | No guarantee |
| **Exam Tip** | "decouple", "buffer", "exactly-once" | "fan-out", "multiple subscribers", "notify" | "AWS event", "cross-account", "content routing", "schedule" |

### SQS Standard vs SQS FIFO

| Dimension | Standard | FIFO |
|-----------|----------|------|
| **Use Case** | High throughput, duplicates OK | Ordered, exactly-once critical |
| **Throughput** | Unlimited | 300/s (3,000 batched, 70k high-throughput) |
| **Delivery** | At-least-once | Exactly-once |
| **Ordering** | Best-effort | Strict per Message Group ID |
| **Pricing** | $0.40/million | $0.50/million |
| **HA Model** | Same (Multi-AZ) | Same (Multi-AZ) |
| **Exam Tip** | "high volume", "order not critical" | "must not duplicate", "sequential", "financial" |

### SNS Standard vs SNS FIFO

| Dimension | Standard Topics | FIFO Topics |
|-----------|----------------|-------------|
| **Subscribers** | SQS, Lambda, HTTP, Email, SMS, Firehose | SQS FIFO only |
| **Throughput** | Very high (soft limits) | 300/s (3,000 batched) |
| **Ordering** | No | Yes (per Message Group ID) |
| **Deduplication** | No | Yes (5-min window) |
| **Exam Tip** | "fan-out to Lambda/HTTP/email" | "ordered fan-out to multiple FIFO queues" |

### EventBridge vs SNS for Event Routing

| Dimension | EventBridge | SNS |
|-----------|-------------|-----|
| **Filtering** | Rich content-based (body + attributes, nested JSON) | Attribute-based filter policies (simpler) |
| **Targets** | 28+ AWS service targets + API Destinations | Protocol-based (SQS, Lambda, HTTP, email, SMS) |
| **Cross-Account** | Native (event bus policies) | Requires cross-account subscription setup |
| **Schema** | Schema Registry + Discovery | No |
| **Replay** | Archive & Replay | No |
| **Scheduling** | Yes (Scheduler) | No |
| **Exam Tip** | "content routing", "third-party API", "AWS state change" | "simple fan-out", "email/SMS notification" |

---

## 10. When to Pick Service A over Service B — Exam Keywords

### Pick SQS over SNS when:
- "single consumer", "one processing application"
- "buffer requests", "decouple producer from consumer"
- "batch processing", "worker queue"
- "retain messages", "process later"
- "visibility timeout", "processing time varies"
- "exactly-once processing" (FIFO)

### Pick SNS over SQS when:
- "multiple subscribers", "fan-out"
- "notify", "alert", "email", "SMS"
- "push-based delivery"
- "different consumers need same message"
- "real-time notification to mobile"

### Pick EventBridge over SNS when:
- "react to AWS service event" (EC2, S3, RDS state changes)
- "content-based routing" (filter on event body content)
- "cross-account event aggregation"
- "third-party SaaS integration" (partner event bus)
- "schedule", "cron job"
- "archive events", "replay events"
- "send to third-party API" (API Destinations)
- "many different targets for one event"

### Pick SNS over EventBridge when:
- "simple notification" (email, SMS, mobile push)
- "existing SNS infrastructure", "fan-out to SQS already set up"
- "FIFO ordered fan-out" (SNS FIFO → SQS FIFO)
- "extremely high throughput fanout" (SNS has higher soft limits)
- Cost-sensitive (SNS = $0.50/million vs EventBridge = $1.00/million custom events)

### Pick SQS FIFO over Kinesis Data Streams when:
- "exactly-once" (SQS FIFO guarantees this; KDS is at-least-once)
- "message-level acknowledgment" (SQS deletes per-message; KDS checkpoints per-shard)
- "lower throughput, strict ordering" (<70k messages/sec)
- "no need for replay" (don't need retention/replay capability)

### Pick Kinesis Data Streams over SQS when:
- "replay", "reprocess historical data"
- "multiple consumers on same data" (Enhanced Fan-Out)
- "real-time analytics" (Flink integration)
- "very high throughput streaming" (unlimited with enough shards)
- "time-based retention" (24h–365d)

---

## 11. "Gotcha" Differences the Exam Tests

### SQS Gotchas
1. **maxReceiveCount counts ALL receives, not just failures.** A message read 5 times (even if processed successfully 4 times but visibility timeout expired) hits DLQ.
2. **SQS DLQ must be same type as source.** Standard → Standard, FIFO → FIFO. Cross-type is not allowed.
3. **SQS FIFO queue name MUST end in `.fifo`.** This is enforced — you can't create one without the suffix.
4. **Long polling returns immediately if messages exist.** WaitTimeSeconds only applies when the queue is empty.
5. **In-flight message limit (120k Standard / 20k FIFO) is per-queue.** Exceeding this causes `OverLimit` errors.
6. **Purging a queue takes up to 60 seconds.** Messages may still be visible briefly after purge.

### SNS Gotchas
7. **SNS FIFO topics ONLY support SQS FIFO subscribers.** Not Lambda, not HTTP, not email. If the question mentions Lambda + FIFO ordering → wrong.
8. **SNS filter policy on message attributes by default.** Payload-based filtering must be explicitly enabled (newer feature).
9. **SNS → SQS cross-account requires BOTH topic policy AND queue policy.** Missing either = delivery fails silently.
10. **SNS message delivery is NOT guaranteed for HTTP endpoints.** If endpoint is down beyond retry policy → message lost (unless DLQ configured on subscription).
11. **SNS raw message delivery:** By default, SNS wraps messages in JSON envelope. Enable "Raw Message Delivery" to pass payload as-is to SQS/HTTP.

### EventBridge Gotchas
12. **AWS service events are FREE. Custom events cost $1/million.** The exam may test whether you know the cost difference.
13. **EventBridge rules execute independently.** If two rules match the same event, both fire — no deduplication between rules.
14. **EventBridge Global Endpoints need Route 53 health checks.** Without health checks, failover doesn't work.
15. **EventBridge Pipes sources are limited:** SQS, Kinesis, DynamoDB Streams, MSK, self-managed Kafka, MQ. Not SNS, not S3, not HTTP.
16. **`PutEvents` supports up to 10 entries per API call** (not 10,000 — that's entries per second throughput, often confused).
17. **Cross-account EventBridge:** The SENDER account creates the rule that forwards. The RECEIVER account sets the resource policy on its bus.

### Cross-Service Gotchas
18. **S3 → EventBridge must be explicitly enabled** per bucket. It's not automatic like S3 → default event bus.
19. **DLQ configuration differs:** SQS DLQ = on source queue. SNS DLQ = on subscription. EventBridge DLQ = on target/rule.
20. **Message size is 256 KB for ALL three services** — but they count differently (SQS counts in 64KB chunks for billing).

---

## 12. Decision Tree: Choosing the Right Messaging/Event Service

```
START: What is the primary need?
│
├─── Need to DECOUPLE producer from consumer?
│    ├─── Single consumer processing? → SQS
│    │    ├─── Need ordering? → SQS FIFO
│    │    ├─── Need exactly-once? → SQS FIFO
│    │    └─── High throughput, duplicates OK? → SQS Standard
│    └─── Multiple consumers need same message? → SNS + SQS fan-out
│
├─── Need to REACT to an AWS service event?
│    └─── EventBridge (free for AWS events)
│         ├─── Need to route to multiple targets? → EventBridge rules
│         ├─── Need cross-account? → EventBridge bus policy
│         └─── Need to archive/replay? → EventBridge Archive
│
├─── Need to NOTIFY humans or systems?
│    ├─── Email/SMS/Mobile push? → SNS
│    ├─── Third-party HTTP API with auth? → EventBridge API Destinations
│    └─── Webhook (simple HTTP)? → SNS HTTP subscription
│
├─── Need CONTENT-BASED ROUTING?
│    ├─── Simple attribute filtering? → SNS filter policies
│    └─── Rich body/nested JSON filtering? → EventBridge rules
│
├─── Need SCHEDULING?
│    └─── EventBridge Scheduler (cron/rate)
│
├─── Need CROSS-ACCOUNT event aggregation?
│    └─── EventBridge (central bus + org policy)
│
├─── Need ORDERED FAN-OUT?
│    └─── SNS FIFO → SQS FIFO queues
│
└─── Need REAL-TIME STREAMING with replay?
     └─── Kinesis Data Streams (not SQS, SNS, or EventBridge)
```
