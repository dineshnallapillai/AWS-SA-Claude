# AWS Application Integration: Step Functions, API Gateway, AppSync

## 1. Core Concepts & Theory

---

### AWS Step Functions

**What it is:** Fully managed serverless orchestration service that coordinates distributed applications and microservices using visual workflows (state machines).

#### Standard Workflows vs Express Workflows

| Feature | Standard | Express |
|---------|----------|---------|
| Duration | Up to 1 year | Up to 5 minutes |
| Execution model | Exactly-once | At-least-once (async) or at-most-once (sync) |
| Execution history | Yes (up to 90 days in console) | CloudWatch Logs only |
| Pricing | Per state transition ($0.025 per 1,000) | Per execution + duration + memory |
| Max executions | 2,000/sec start rate (soft) | 100,000/sec start rate |
| Use case | Long-running, audit trail needed | High-volume, short-duration, idempotent |
| Idempotency | Built-in (exactly-once) | Must implement yourself (at-least-once) |

#### Express Workflow Subtypes
- **Synchronous Express:** Caller waits for result (API Gateway integration, <5 min)
- **Asynchronous Express:** Fire-and-forget, result in CloudWatch Logs

#### State Types

| State | Purpose |
|-------|---------|
| **Task** | Do work (invoke Lambda, ECS, Batch, DynamoDB, SNS, SQS, Glue, SageMaker, etc.) |
| **Choice** | Branching logic (if/else) |
| **Parallel** | Execute branches concurrently |
| **Map** | Iterate over array (Inline Map or Distributed Map) |
| **Wait** | Delay for time or until timestamp |
| **Pass** | Pass input to output (transform/inject data) |
| **Succeed** | Terminal success state |
| **Fail** | Terminal failure state |

#### Error Handling

**Retry:**
```json
"Retry": [
  {
    "ErrorEquals": ["States.TaskFailed", "Lambda.ServiceException"],
    "IntervalSeconds": 2,
    "MaxAttempts": 3,
    "BackoffRate": 2.0,
    "MaxDelaySeconds": 60,
    "JitterStrategy": "FULL"
  }
]
```
- **ErrorEquals:** Which errors to retry
- **IntervalSeconds:** Initial wait before retry
- **MaxAttempts:** Maximum retry count (default 3, set 0 to never retry)
- **BackoffRate:** Multiplier for each subsequent wait
- **JitterStrategy:** FULL or NONE (reduces thundering herd)

**Catch:**
```json
"Catch": [
  {
    "ErrorEquals": ["States.ALL"],
    "Next": "HandleError",
    "ResultPath": "$.error"
  }
]
```
- Routes to a fallback state on failure
- **ResultPath:** Where to place error output in state input

**Predefined Error Codes:**
- `States.ALL` — matches any error
- `States.TaskFailed` — task returned failure
- `States.Timeout` — task/state timed out
- `States.Permissions` — insufficient IAM permissions
- `States.ResultPathMatchFailure` — ResultPath can't be applied
- `States.HeartbeatTimeout` — no heartbeat received within timeout

#### Service Integrations

**Integration Patterns:**
- **Request-Response (.sync not appended):** Call service and move to next state immediately
- **Run a Job (.sync):** Wait for job to complete (Lambda, ECS, Batch, Glue, etc.)
- **Wait for Callback (.waitForTaskToken):** Pause until external system sends token back via `SendTaskSuccess`/`SendTaskFailure`

**SDK Integrations:** Over 200+ AWS services can be called directly (no Lambda glue needed)

#### Distributed Map
- Process large-scale datasets (millions of items from S3)
- Launches up to 10,000 parallel child executions
- Items from: S3 objects, S3 CSV/JSON inventory, or passed-in array
- Supports batching items per child execution
- **Use case:** Large-scale ETL, data processing, batch operations

#### Activity Tasks
- Step Functions sends work to **activity workers** (EC2, on-premises)
- Workers poll for tasks using `GetActivityTask`
- Must send heartbeats and return success/failure
- Enables hybrid workflows with on-premises components

#### Quotas & Limits
- Max state machine definition size: 1 MB
- Max execution history events: 25,000 (Standard)
- Max input/output per state: 256 KB
- Max parallel branches: No hard limit (practical ~40-50)
- Max open executions: 1,000,000 (Standard)
- Execution start rate: 2,000/sec (Standard), 100,000/sec (Express)
- State transition rate: 5,000/sec per account (soft)

---

### Amazon API Gateway

**What it is:** Fully managed service to create, publish, maintain, monitor, and secure APIs at any scale. Supports REST, HTTP, and WebSocket APIs.

#### REST API vs HTTP API vs WebSocket API

| Feature | REST API | HTTP API | WebSocket API |
|---------|----------|----------|---------------|
| Protocol | RESTful | RESTful | Persistent connections |
| Cost | ~$3.50/million requests | ~$1.00/million requests | $1.00/million messages + connection minutes |
| Latency | Higher (~50ms overhead) | Lower (~10ms overhead) | Persistent (no reconnect) |
| Authorization | IAM, Cognito, Lambda authorizer, API keys | IAM, Cognito (JWT), Lambda authorizer | IAM, Lambda authorizer |
| Caching | Yes (built-in) | No | No |
| Usage Plans/API Keys | Yes | No | No |
| Request validation | Yes | No | No |
| Request/Response transformation | Yes (Velocity templates) | Parameter mapping only | Route selection expression |
| WAF integration | Yes | No | Yes |
| Resource policies | Yes | No | No |
| Private endpoints | Yes | Yes (via VPC link) | No |
| Custom domain | Yes | Yes | Yes |
| Mutual TLS | Yes | Yes | No |

#### When to Choose What
- **REST API:** Need caching, WAF, request validation, API keys/usage plans, resource policies, full transformation
- **HTTP API:** Simple proxy to Lambda/HTTP backend, cost-sensitive, low-latency, JWT auth sufficient
- **WebSocket API:** Real-time bidirectional communication (chat, gaming, streaming dashboards)

#### REST API Features In-Depth

**Stages:**
- Named references to a deployment (e.g., `dev`, `staging`, `prod`)
- Stage variables (like environment variables for the stage)
- Canary deployments: Route % of traffic to new deployment
- Each stage has independent settings (throttling, caching, logging)

**Caching:**
- Cache capacity: 0.5 GB to 237 GB
- TTL: Default 300s, range 0–3600s
- Cache per stage (not per method unless configured)
- Cache invalidation: `Cache-Control: max-age=0` header (requires IAM auth or check "Require authorization")
- **Cost:** Per-hour charge based on cache size
- Cache key: Resource path + query string params + headers (configurable)

**Throttling:**
- **Account-level:** 10,000 requests/sec steady-state, 5,000 burst across ALL APIs
- **Stage-level and method-level overrides** available
- **Usage plans:** Associate API keys with throttle limits and quotas
- 429 Too Many Requests returned when throttled
- Token bucket algorithm

**Request Validation:**
- Validate body, query string params, headers
- Returns 400 Bad Request if validation fails
- Reduces unnecessary backend invocations

**API Keys & Usage Plans:**
- API key = alphanumeric string for identifying callers
- Usage plan = throttle + quota settings per API key
- **NOT for authentication** — for tracking and throttling
- Use with `x-api-key` header

**Resource Policies:**
- JSON policy on the API itself
- Control who can invoke the API (cross-account, IP ranges, VPCs)
- Combined with IAM user/role policies using AND logic
- Use case: Restrict API to specific VPCs or IP ranges

#### Integration Types

| Type | Description |
|------|-------------|
| **Lambda (Proxy)** | Passes full request to Lambda, Lambda controls response format |
| **Lambda (Custom)** | Uses mapping templates for request/response transformation |
| **HTTP (Proxy)** | Passes request to HTTP endpoint, returns response as-is |
| **HTTP (Custom)** | Mapping templates for transformation |
| **AWS Service** | Direct integration with AWS services (DynamoDB, SQS, Step Functions, S3, Kinesis) |
| **Mock** | Returns hardcoded response (no backend) |
| **VPC Link** | Connect to resources in private VPC (NLB for REST, ALB/NLB/Cloud Map for HTTP) |

#### WebSocket API

**Connection management:**
- `$connect` route: Triggered on new connection
- `$disconnect` route: Triggered on disconnect
- `$default` route: Fallback for unmatched routes
- Custom routes based on `routeSelectionExpression` (e.g., `$request.body.action`)

**Callback URL:** `https://{api-id}.execute-api.{region}.amazonaws.com/{stage}/@connections/{connection-id}`
- POST: Send message to specific client
- GET: Get connection info
- DELETE: Disconnect client

**Use cases:** Chat, live dashboards, multiplayer games, real-time notifications

#### Endpoint Types
- **Edge-Optimized:** CloudFront distribution in front (default), best for geographically dispersed clients
- **Regional:** No CloudFront, use when clients are in same region or you want your own CloudFront
- **Private:** Accessible only from VPC via VPC Interface Endpoint

#### Quotas
- Payload size: 10 MB (REST/HTTP), 128 KB per frame (WebSocket)
- Integration timeout: 29 seconds (REST/HTTP), 29 seconds (WebSocket idle: 10 minutes)
- Throttle: 10,000 RPS account default (soft limit)
- Max APIs per account per region: 600
- Stages per API: 10 (soft limit)

---

### AWS AppSync

**What it is:** Fully managed GraphQL and Pub/Sub API service that simplifies application development by providing a single endpoint to query, mutate, and subscribe to data from multiple sources.

#### Key Concepts

**Schema-First Design:**
- Define your API using GraphQL SDL (Schema Definition Language)
- Types, Queries, Mutations, Subscriptions
- AppSync generates connection logic based on schema

**Operations:**
- **Query:** Read data (like GET)
- **Mutation:** Write/update data (like POST/PUT/DELETE)
- **Subscription:** Real-time updates via WebSockets (triggered by mutations)

#### Resolvers

**Resolver Types:**
- **Unit Resolver:** Single data source operation
- **Pipeline Resolver:** Chain multiple functions (steps) in sequence
  - Before mapping template → Function 1 → Function 2 → ... → After mapping template
  - Each function can talk to a different data source

**Resolver Runtimes:**
- **APPSYNC_JS:** JavaScript resolvers (recommended, newer)
- **VTL (Velocity Template Language):** Legacy Apache Velocity templates

**Data Sources:**
- DynamoDB
- Lambda
- RDS (Aurora Serverless v2 via Data API)
- OpenSearch
- HTTP endpoints (any REST API)
- EventBridge
- None (local resolver, for Pub/Sub without backend)

#### Real-Time (Subscriptions)
- Pure WebSocket-based (managed by AppSync)
- Client subscribes to specific mutations
- Subscription filter: Server-side filtering to reduce unnecessary data to client
- Enhanced subscription filtering: Filter by arguments at connection time
- Scales to millions of connected clients
- Backed by MQTT over WebSockets
- **Authorization modes apply to subscriptions too**

#### Caching
- Managed caching (ElastiCache under the hood)
- Cache at resolver level or full API level
- TTL configurable (1 second to 3600 seconds)
- Cache key: Based on the query/context
- Cache invalidation: TTL expiry or manual flush
- Instance sizes: t2.small to r4.8xlarge

#### Merged APIs
- Combine multiple source AppSync APIs into a single merged API
- Each team owns their sub-API (schema)
- Merged API provides a unified endpoint
- Enables microservices pattern with GraphQL

#### Conflict Detection & Resolution (for Offline/Sync)
- Built-in conflict detection for offline-first apps (Amplify DataStore)
- Strategies: **Optimistic Concurrency**, **Auto Merge**, **Lambda** (custom)
- Versioned data sources track item versions

#### Authorization Modes (can combine multiple)
- **API_KEY:** Simple key-based (for public APIs, prototyping, TTL max 365 days)
- **AWS_IAM:** IAM roles/policies (for AWS services, authenticated users)
- **AMAZON_COGNITO_USER_POOLS:** JWT from Cognito (group-based access)
- **OPENID_CONNECT:** Any OIDC-compliant provider
- **AWS_LAMBDA:** Custom authorization logic

#### Quotas
- Schema size: 1 MB
- Max nested resolver depth: 10 levels
- Subscription payload: 240 KB
- Request execution timeout: 30 seconds
- Max batch size for BatchGetItem: 100 keys
- Connections: Millions of concurrent WebSocket connections

---

## 2. Design Patterns & Best Practices

### When to Use What

| Scenario | Use |
|----------|-----|
| Orchestrate multi-step workflows | Step Functions (Standard) |
| High-volume event processing pipeline | Step Functions (Express) |
| Human approval workflows | Step Functions (waitForTaskToken) |
| Simple REST API to Lambda | API Gateway HTTP API |
| API with caching, throttling, API keys | API Gateway REST API |
| Real-time bidirectional communication | API Gateway WebSocket or AppSync Subscriptions |
| GraphQL API with multiple data sources | AppSync |
| Mobile/web app with offline sync | AppSync + Amplify DataStore |
| Real-time collaborative features | AppSync Subscriptions |
| Microservices API aggregation | AppSync Merged APIs |

### Anti-Patterns

| Anti-Pattern | Why It's Wrong | Better Approach |
|--------------|---------------|-----------------|
| Lambda chaining (Lambda calls Lambda) | Tight coupling, double billing, error handling nightmare | Step Functions orchestration |
| Step Functions for simple request-response | Overkill, adds latency and cost | Direct Lambda invocation |
| REST API for real-time streaming | Polling is wasteful and high-latency | WebSocket API or AppSync Subscriptions |
| API Gateway caching for personalized data | Cache serves same response to different users | Cache only shared/static data, or use cache keys with user context |
| AppSync for simple CRUD with no relations | Overhead of GraphQL schema for simple APIs | API Gateway + DynamoDB direct integration |
| Standard Step Functions for 100k executions/sec | Standard is limited to 2,000 starts/sec | Express Workflows |
| Express Step Functions for hour-long workflows | Max 5 minutes | Standard Workflows |
| Using API Gateway as a message broker | Not designed for async message buffering | SQS or EventBridge |

### Well-Architected Framework Alignment

**Reliability:**
- Step Functions: Built-in retry/catch, DLQ for failed executions, state durable by default (Standard)
- API Gateway: Multi-AZ by default, throttling protects backends, canary deployments for safe rollouts
- AppSync: Multi-AZ, automatic scaling, resolver error handling

**Security:**
- Step Functions: IAM execution role (least privilege per task), encryption at rest
- API Gateway: Multiple auth options, resource policies, WAF, mutual TLS, private endpoints
- AppSync: Multiple auth modes per API, field-level authorization via `@auth` directives

**Performance:**
- Step Functions: Parallel state for concurrent processing, Express for high-throughput
- API Gateway: Caching (REST), edge-optimized endpoints, payload compression
- AppSync: Resolver-level caching, batch resolvers for N+1 problem, DynamoDB direct access

**Cost:**
- Step Functions: Express cheaper for short, frequent workflows; avoid unnecessary states
- API Gateway: HTTP API 70% cheaper than REST; cache reduces backend calls
- AppSync: Pay per query/mutation, cache reduces resolver invocations

**Operational Excellence:**
- Step Functions: Visual workflow monitoring, execution history, CloudWatch integration
- API Gateway: CloudWatch metrics/logs, X-Ray tracing, access logging
- AppSync: CloudWatch metrics, X-Ray tracing, logging per resolver

### Integration Patterns

```
┌─────────────────────────────────────────────────────────────┐
│                  Common Integration Patterns                  │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  API Gateway → Step Functions (sync via StartSyncExecution)  │
│  API Gateway → SQS → Step Functions (async)                  │
│  API Gateway → Lambda → Step Functions (start execution)     │
│  EventBridge → Step Functions (event-driven orchestration)   │
│  Step Functions → ECS/Fargate (.sync for batch jobs)         │
│  Step Functions → SQS (.waitForTaskToken for callbacks)      │
│  AppSync → Lambda resolver → Step Functions                  │
│  AppSync → DynamoDB (direct resolver, no Lambda)             │
│  CloudFront → API Gateway (custom domain + WAF)              │
│  API Gateway → VPC Link → NLB → ECS (private backend)       │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

**Step Functions + API Gateway Pattern:**
- Synchronous: API Gateway REST → `StartSyncExecution` (Express only, max 29s due to API GW timeout)
- Asynchronous: API Gateway → `StartExecution` → client polls or receives callback

**Saga Pattern (Step Functions):**
- Distributed transaction across microservices
- Each step has a compensating action (undo)
- On failure → execute compensating transactions in reverse
- Step Functions Catch/Retry + Choice states implement this naturally

---

## 3. Security & Compliance

### Step Functions Security

**IAM Execution Role:**
- State machine assumes a role to invoke tasks
- **Least privilege:** Only grant permissions for services the workflow actually calls
- Separate roles per state machine

**Encryption:**
- Execution data encrypted at rest with AWS-owned key (default) or customer-managed KMS key
- Input/output data is logged — be careful with sensitive data in execution history

**Logging:**
- CloudWatch Logs: Enable for Express workflows (required for debugging)
- Execution history: Standard workflows store up to 25,000 events per execution
- X-Ray tracing: Supported for distributed tracing

**VPC Access:**
- Step Functions itself is not in a VPC
- Lambda tasks can run in VPC; ECS tasks can run in VPC
- Use VPC endpoints for private API access to Step Functions

### API Gateway Security

**Authentication & Authorization:**

| Method | Best For |
|--------|----------|
| IAM (Sig v4) | AWS services, cross-account, CLI/SDK calls |
| Cognito User Pools | Web/mobile apps with user sign-up/sign-in |
| Lambda Authorizer (Token) | Bearer token validation (JWT from third-party) |
| Lambda Authorizer (Request) | Multi-header/query param authorization |
| API Keys | Not auth! Only for usage tracking/throttling |
| Mutual TLS (mTLS) | Certificate-based client authentication |

**Resource Policies (REST only):**
- Allow/deny based on: AWS accounts, IP ranges, VPC/VPCE
- Combined with IAM policy (both must allow for cross-account)
- Use case: "Only allow calls from VPC endpoint X" or "Block IPs from country Y"

**WAF Integration (REST & WebSocket only):**
- Protect against SQL injection, XSS, rate-based rules
- IP whitelisting/blacklisting
- Geographic blocking
- Bot control

**Private API:**
- Only accessible from VPC via Interface VPC Endpoint
- Resource policy must allow the VPCE or VPC
- DNS: Use the `execute-api` VPC endpoint with private DNS or custom alias

**Encryption:**
- TLS 1.2 minimum (in transit)
- SSL certificates managed by ACM (custom domains)
- No server-side encryption of cached data — cache is in-memory within the service

### AppSync Security

**Field-Level Authorization:**
- Different auth modes for different fields in the schema
- `@aws_auth`, `@aws_iam`, `@aws_api_key`, `@aws_cognito_user_pools`, `@aws_oidc`, `@aws_lambda`
- A single API can use multiple auth modes simultaneously

**Cognito Groups:**
- Restrict queries/mutations/fields to specific Cognito groups
- `@aws_auth(cognito_groups: ["Admins"])`

**IAM Authorization:**
- Fine-grained: control access to specific GraphQL operations
- Cross-account access via IAM roles

**WAF Integration:**
- Protect AppSync APIs with AWS WAF rules
- Rate limiting, IP blocking, geo-blocking

**Logging:**
- CloudWatch Logs: Per-resolver logging (NONE, ERROR, ALL, INFO, DEBUG)
- X-Ray: Distributed tracing through resolver chain

### Cross-Service Security Patterns

**API Gateway → Lambda → Step Functions:**
- API Gateway authenticates caller (Cognito/IAM)
- Lambda assumes role to start Step Functions execution
- Step Functions assumes role per task invocation
- Each layer has independent IAM boundaries

**Private API Pattern:**
```
Client (VPC) → VPC Endpoint → API Gateway (Private) → VPC Link → NLB → ECS (private subnet)
```
- End-to-end private, never traverses internet

---

## 4. Cost Optimization

### Step Functions Pricing

**Standard Workflows:**
- $0.025 per 1,000 state transitions
- Free tier: 4,000 state transitions/month
- **Trap:** A state machine with 10 states executing 1M times = 10M transitions = $250

**Express Workflows:**
- $1.00 per 1 million requests
- $0.00001667 per GB-second of duration
- Much cheaper for high-volume, short-duration workflows

**Cost Optimization:**
- Use Express for short-lived, high-volume workflows
- Reduce state transitions: combine operations, use Map state instead of loops
- Use SDK integrations directly (skip Lambda when possible) — fewer states
- Avoid Pass states unless needed for transformation
- Distributed Map: Process items in batches to reduce transitions

**When Standard is cheaper:** Long-running workflows with few transitions (e.g., wait states for days)
**When Express is cheaper:** High-frequency, short-duration (e.g., processing each API request)

### API Gateway Pricing

**REST API:**
- $3.50 per million requests (first 333M)
- Cache: $0.02–$3.80/hour depending on size
- Data transfer: Standard AWS rates

**HTTP API:**
- $1.00 per million requests (first 300M)
- No cache cost (no caching feature)
- ~70% cheaper than REST API

**WebSocket API:**
- $1.00 per million connection minutes
- $1.00 per million messages (32KB chunks)

**Cost Optimization:**
- Use HTTP API unless you need REST-specific features (caching, WAF, API keys, request validation)
- Enable caching to reduce backend invocations (but pay for cache hour)
- Use request validation to reject bad requests before hitting Lambda
- Throttle aggressively to protect backend (and budget)
- Payload compression reduces data transfer costs
- Use regional endpoint + your own CloudFront if you need edge caching with more control

**Cost Traps:**
- REST API cache running 24/7 for dev/staging environments
- Not throttling = runaway backend costs on abuse
- Large payloads increase data transfer charges
- WebSocket connections left idle still cost per connection-minute
- Edge-optimized creates a hidden CloudFront distribution (data transfer costs)

### AppSync Pricing

- **Query/Mutation:** $4.00 per million requests
- **Real-time updates (Subscriptions):** $2.00 per million connection-minutes + $2.00 per million messages
- **Caching:** $0.02–$3.80/hour (similar to API GW)
- **Data transfer:** Standard rates

**Cost Optimization:**
- Use DynamoDB direct resolvers instead of Lambda (avoid Lambda invocation cost)
- Enable caching for read-heavy operations
- Batch resolvers to reduce DynamoDB reads (solve N+1 efficiently)
- Use subscription filters to reduce unnecessary message delivery
- Use API keys for unauthenticated/public access (no Cognito overhead)

**Cost Comparison: AppSync vs API Gateway + Lambda + Custom GraphQL:**
- At low scale: AppSync is simpler and similar cost
- At very high scale: API Gateway HTTP API + Lambda can be cheaper for simple operations
- AppSync wins when you need real-time, offline sync, or multi-data-source aggregation

---

## 5. High Availability, Disaster Recovery & Resilience

### Step Functions HA
- **Multi-AZ by default:** State machines and execution history replicated across AZs
- **Regional service:** Not multi-region by default
- **Execution state is durable (Standard):** Survives AZ failure mid-execution
- **Cross-region DR:** Deploy state machines in multiple regions, use Route 53 or EventBridge to route

**Resilience Features:**
- Built-in retry with backoff
- Catch blocks for error isolation
- Heartbeat timeouts detect hung tasks
- No single point of failure within a region

### API Gateway HA
- **Multi-AZ by default:** Fully managed, scales automatically
- **Edge-optimized:** CloudFront provides global edge caching + DDoS protection
- **Regional failover:** Deploy in multiple regions + Route 53 failover routing + custom domain
- **Throttling:** Protects backend from overload (self-preserving)

**Multi-Region API Pattern:**
```
Route 53 (failover) → Region A: API Gateway + Lambda + DynamoDB Global Table
                    → Region B: API Gateway + Lambda + DynamoDB Global Table
```
- Custom domain with ACM certificates in both regions
- Route 53 health checks trigger failover

### AppSync HA
- **Multi-AZ by default**
- **Regional service**
- **Cross-region:** Deploy in multiple regions, use Route 53 custom domain for failover
- **Offline resilience:** Amplify DataStore provides local cache, syncs when reconnected

### Resilience Patterns

**Circuit Breaker (Step Functions):**
```
Check Circuit State (Choice) →
  OPEN → Return cached/default response
  CLOSED → Try operation → 
    Success → Reset failure count
    Failure → Increment failure count →
      Threshold reached → Set circuit OPEN
```

**Bulkhead (API Gateway):**
- Per-method throttling isolates failures
- Usage plans per client prevent noisy neighbors
- Stage-level throttling as overall protection

**Async with Callback (Step Functions):**
```
API Gateway → Start Execution → Return execution ARN to client
Step Functions → Process → SNS/WebSocket notify client of completion
```
- Avoids 29-second API Gateway timeout for long operations

### RPO/RTO

| Service | RPO | RTO | Strategy |
|---------|-----|-----|----------|
| Step Functions | Near-zero (in-region) | Minutes (cross-region redeploy) | Multi-region deployment, IaC |
| API Gateway | Zero (stateless) | Seconds (Route 53 failover) | Multi-region + custom domain |
| AppSync | Near-zero (with Global Tables) | Seconds (Route 53 failover) | Multi-region + DynamoDB Global Tables |

---

## 6. Exam-Focused Section

### Straightforward Questions

**Q1:** A company has a multi-step order processing workflow that takes up to 30 minutes. Each step calls a different microservice. If a step fails, it must retry 3 times. If it still fails, a compensating transaction must execute. Which service?

**A:** AWS Step Functions (Standard Workflow) with Retry configuration (MaxAttempts: 3) and Catch blocks routing to compensating states (Saga pattern).

---

**Q2:** An application exposes a REST API that receives 10,000 requests/second. The responses are the same for identical query parameters and rarely change. How can you reduce backend load?

**A:** Enable API Gateway REST API caching with appropriate TTL. Cache key includes query string parameters. This serves repeated requests from cache without invoking the backend.

---

**Q3:** A mobile app needs real-time updates when data changes in DynamoDB. The app also needs to work offline and sync when reconnected. What service?

**A:** AWS AppSync with Subscriptions for real-time updates and Amplify DataStore for offline support with automatic conflict resolution.

---

**Q4:** A company needs to process 80,000 image thumbnails per second. Each processing takes 2 seconds. Which Step Functions workflow type?

**A:** Express Workflow (supports 100,000 starts/sec, max 5 min duration). Standard would be throttled at 2,000 starts/sec.

---

**Q5:** A company wants to expose an API that directly puts messages into an SQS queue without using Lambda. Which API Gateway integration type?

**A:** AWS Service integration type. API Gateway can directly integrate with SQS (SendMessage action) using request mapping templates (REST API) to construct the SQS API call.

---

**Q6:** A development team needs a low-latency, low-cost API that proxies requests to Lambda. They don't need caching, API keys, or request validation. What should they use?

**A:** API Gateway HTTP API. It's 70% cheaper than REST API, has lower latency (~10ms vs ~50ms overhead), and supports Lambda proxy integration.

---

**Q7:** A workflow needs to pause execution until a human approves a request via email link. Which Step Functions feature?

**A:** Task state with `.waitForTaskToken` integration pattern. Send the task token in the approval email link. When the human clicks approve, the callback endpoint calls `SendTaskSuccess` with the token.

---

### Tricky / Scenario-Based Questions

**Q1:** An API Gateway REST API with caching is returning stale data after a database update. A developer adds `Cache-Control: max-age=0` header to the client request, but it still returns cached data. Why?

**A:** The **"Require authorization"** checkbox for cache invalidation is enabled (default). Only callers with `execute-api:InvalidateCache` IAM permission can invalidate. Without this permission, the header is ignored. **Fix:** Either grant the IAM permission or uncheck "Require authorization" on the stage settings. **Key trap:** Cache invalidation requires IAM permission by default — it's not just a header.

---

**Q2:** A company uses API Gateway with a Lambda authorizer. The authorizer is invoked for every request, causing high latency and cost. What should they do?

**A:** Enable **Lambda authorizer caching** (set TTL > 0). The authorization result is cached based on the token/request parameters, so subsequent requests skip the authorizer invocation. **Default TTL:** 300 seconds. **Wrong answer:** "Use API key" — API keys are not for authentication. **Wrong answer:** "Use Cognito" — valid but changes the auth mechanism entirely; the question asks to optimize current setup.

---

**Q3:** A Step Functions Standard Workflow has 15 sequential Lambda tasks. The company is concerned about cost at scale (1 million executions/month). A colleague suggests switching to Express. Should they?

**A:** It depends on **duration**. If each execution completes in under 5 minutes → Express is cheaper (1M executions × small duration vs. 15M state transitions × $0.025/1000 = $375/month for Standard). But Express provides **at-least-once** semantics — Lambdas must be **idempotent**. If the workflow takes >5 minutes or requires exactly-once semantics → must stay Standard. **Key phrases to watch:** "exactly-once" or "must not duplicate" → Standard. "High volume, short duration" → Express.

---

**Q4:** A WebSocket API on API Gateway needs to send notifications to 100,000 connected clients when data changes. The backend is a Lambda processing DynamoDB Streams. How should the Lambda send messages to all connected clients?

**A:** Lambda must iterate through connections and POST to the `@connections` callback URL for each client. **Problem:** This doesn't scale well (Lambda timeout, connection management). **Better architecture for the exam:** Use AppSync Subscriptions instead — it natively handles fan-out to millions of subscribers. **If they insist on WebSocket API:** Store connection IDs in DynamoDB, use Step Functions or SNS to parallelize message delivery. **Key trap:** API Gateway WebSocket requires you to manage connection IDs and message delivery yourself. AppSync Subscriptions handles this automatically.

---

**Q5:** A company is building a GraphQL API with AppSync. Some queries should be accessible by unauthenticated users (product catalog), while mutations should require Cognito authentication. How?

**A:** Configure AppSync with **multiple authorization modes**: set primary auth to `API_KEY` (for public queries) and add `AMAZON_COGNITO_USER_POOLS` as additional auth. Use `@aws_api_key` directive on public types/fields and `@aws_cognito_user_pools` on protected mutations. **Wrong answer:** "Create two separate AppSync APIs" — unnecessary; multiple auth modes solve this. **Wrong answer:** "Use Lambda authorizer for everything" — overcomplicates when native Cognito integration exists.

---

**Q6:** An API Gateway REST API integrates with a backend that occasionally takes 35 seconds to respond. Clients receive 504 errors. What's wrong and how to fix it?

**A:** API Gateway has a **hard limit of 29 seconds** for integration timeout (cannot be increased). **Fixes:** (1) Asynchronous pattern: API Gateway → SQS/Step Functions, return 202 Accepted, client polls for result. (2) Optimize backend to respond within 29s. (3) Break into smaller operations. **Wrong answer:** "Increase the timeout setting" — 29s is a hard limit, not configurable beyond it. **Key fact:** 29 seconds is one of the most tested limits on the exam.

---

**Q7:** A company uses Step Functions to orchestrate Lambda, ECS, and third-party API calls. An ECS task occasionally hangs indefinitely. How do they prevent the workflow from being stuck?

**A:** Configure **HeartbeatSeconds** on the Task state. The ECS task must send heartbeats periodically. If no heartbeat arrives within the HeartbeatSeconds window → `States.HeartbeatTimeout` error is raised → caught by Catch block. Also set **TimeoutSeconds** as a hard upper bound on task duration. **Key distinction:** HeartbeatSeconds = periodic liveness check. TimeoutSeconds = absolute maximum duration. Both can be set, and heartbeat should be shorter than timeout.

---

### Common Exam Traps & Pitfalls

1. **API Gateway 29-second timeout is HARD.** Cannot be increased. Any question where backend takes >29s → asynchronous pattern required.

2. **HTTP API vs REST API:** HTTP API is cheaper and faster but LACKS: caching, WAF, API keys/usage plans, request validation, resource policies. If the question mentions any of these → REST API.

3. **API Keys are NOT authentication.** They are for usage tracking/throttling only. Never select "use API keys" as an authentication solution.

4. **Step Functions Standard vs Express:** Standard = exactly-once, up to 1 year, audit trail. Express = at-least-once, up to 5 minutes, cheaper at high volume. **Don't confuse duration limits.**

5. **Express Workflows are NOT synchronous by default.** Async Express fires-and-forgets. Sync Express waits for result but is limited to what the caller can wait for (API Gateway = 29s).

6. **Step Functions SDK integrations eliminate Lambda glue.** If the question says "minimize custom code" or "reduce Lambda invocations" → use direct SDK integrations (e.g., Step Functions → DynamoDB PutItem directly).

7. **AppSync vs API Gateway WebSocket for real-time:** AppSync Subscriptions = managed fan-out, automatic scaling, subscription filters. WebSocket API = you manage connections, fan-out, and message delivery yourself.

8. **API Gateway caching is per-stage, not per-API.** Each stage is independently cacheable. Different stages can have different cache sizes or no cache.

9. **Lambda authorizer result caching:** If the exam mentions "reduce authorizer invocations" or "authorizer called too frequently" → enable TTL caching on the authorizer.

10. **Step Functions .waitForTaskToken:** The execution pauses INDEFINITELY until the token is returned (or a timeout fires). If no TimeoutSeconds is set and the callback never comes → execution stays open forever (Standard: counts toward open execution limit of 1M).

11. **VPC Link:** REST API uses NLB only. HTTP API can use ALB, NLB, or Cloud Map. Don't confuse these.

12. **AppSync resolvers can bypass Lambda entirely.** DynamoDB, RDS, OpenSearch, HTTP can be data sources directly. "Reduce latency and cost" + "AppSync" → use direct data source, not Lambda resolver.

---

## 7. Cheat Sheet

### Must-Know Facts

| Fact | Detail |
|------|--------|
| API Gateway timeout | **29 seconds (hard limit)** |
| API Gateway throttle default | 10,000 RPS account-wide, 5,000 burst |
| API Gateway payload limit | 10 MB (REST/HTTP) |
| HTTP API cost savings | ~70% cheaper than REST API |
| Step Functions Standard max duration | 1 year |
| Step Functions Express max duration | 5 minutes |
| Step Functions Standard delivery | Exactly-once |
| Step Functions Express delivery | At-least-once (async) / At-most-once (sync) |
| Step Functions state input/output | 256 KB max |
| Step Functions history events | 25,000 max per execution |
| Step Functions Express start rate | 100,000/sec |
| Step Functions Standard start rate | 2,000/sec |
| AppSync request timeout | 30 seconds |
| AppSync auth modes | API_KEY, IAM, Cognito, OIDC, Lambda |
| WebSocket idle timeout | 10 minutes (API Gateway) |
| WebSocket message size | 128 KB per frame |

### Key Differentiators

| Feature | Step Functions | API Gateway | AppSync |
|---------|---------------|-------------|---------|
| Primary purpose | Orchestration | API management | GraphQL/Real-time |
| Execution model | State machine | Request/response | Query/Mutation/Subscription |
| Real-time | No (use callbacks) | WebSocket API | Subscriptions (managed) |
| Caching | No | REST API only | Yes (resolver level) |
| Offline support | No | No | Yes (with Amplify) |
| Long-running | Yes (1 year) | No (29s timeout) | No (30s timeout) |
| Direct AWS integrations | 200+ services | Yes (AWS proxy) | DynamoDB, RDS, OpenSearch, HTTP |

### Decision Flowchart

```
Question mentions...                                    → Think...
──────────────────────────────────────────────────────────────────────
"orchestrate", "coordinate", "workflow"                 → Step Functions
"long-running process", "wait for approval"            → Step Functions Standard + waitForTaskToken
"high-volume short processing"                         → Step Functions Express
"retry with backoff", "compensating transaction"       → Step Functions (Retry/Catch/Saga)
"REST API", "throttle", "API keys", "usage plans"     → API Gateway REST
"low-cost API proxy to Lambda"                         → API Gateway HTTP API
"real-time bidirectional"                              → WebSocket API or AppSync
"29 seconds", "timeout exceeded"                       → Async pattern (SQS/Step Functions)
"GraphQL", "single endpoint, multiple sources"         → AppSync
"real-time data sync", "offline mobile"               → AppSync + Amplify DataStore
"subscriptions", "push notifications on data change"  → AppSync Subscriptions
"caching API responses"                                → API Gateway REST API caching
"private API only from VPC"                            → API Gateway Private + VPC Endpoint
"reduce Lambda invocations in workflow"                → Step Functions SDK integrations
"fan-out to millions of connected clients"            → AppSync Subscriptions (not WebSocket API)
"backend takes > 29 seconds"                           → Step Functions async, return 202
"minimize custom code in orchestration"               → Step Functions direct integrations
```

### Quick Comparison: "When the question says X, the answer is Y"

- "Coordinate microservices with error handling" → **Step Functions**
- "Wait for human approval" → **Step Functions .waitForTaskToken**
- "Process 100k events/sec, each takes 3 seconds" → **Step Functions Express**
- "Exactly-once workflow execution" → **Step Functions Standard**
- "API with caching and API key management" → **API Gateway REST API**
- "Cheapest API proxy to Lambda" → **API Gateway HTTP API**
- "Chat application with persistent connections" → **API Gateway WebSocket**
- "Backend timeout exceeds 29 seconds" → **Async: API GW → SQS/Step Functions → Callback**
- "GraphQL with multiple data sources" → **AppSync**
- "Mobile offline sync with conflict resolution" → **AppSync + Amplify DataStore**
- "Real-time dashboard for thousands of users" → **AppSync Subscriptions**
- "Restrict API to VPC only" → **API Gateway Private Endpoint + VPC Endpoint**
- "Protect API from DDoS/SQL injection" → **API Gateway REST + WAF**
- "Reduce Lambda glue in workflow" → **Step Functions direct SDK integration**
- "API needs to access private backend (NLB)" → **API Gateway + VPC Link**

---

## 8. Deep Dive: 10 More Tricky Scenario-Based Questions

**Q1:** A company uses API Gateway REST API with Lambda proxy integration. They enable caching with a 5-minute TTL. After deployment, authenticated users see other users' data. What went wrong?

**A:** The cache key does not include user identity. By default, cache key = resource path + query string parameters. If the response is personalized per user, you must add the `Authorization` header (or a user-identifying header) to the cache key parameters. **Fix:** Configure cache key parameters to include the `Authorization` header OR disable caching for personalized endpoints. **Key trap:** API Gateway caching is NOT user-aware by default. Caching personalized responses without proper cache keys = data leakage security vulnerability.

---

**Q2:** A Step Functions workflow uses a Map state to process 50,000 items from S3. The execution fails with "States.DataLimitExceeded" error. What's wrong?

**A:** The standard inline Map state passes all items through the state machine's payload, which is limited to **256 KB per state input/output**. 50,000 items exceeds this. **Fix:** Use **Distributed Map** mode — it reads items directly from S3 (no payload limit) and launches child executions (up to 10,000 concurrent). **Wrong answer:** "Increase the payload limit" — 256 KB is a hard limit. **Key distinction:** Inline Map = items in payload (256 KB limit). Distributed Map = items from S3 (millions of items).

---

**Q3:** A company exposes a REST API via API Gateway. They need to allow access only from their corporate VPN (specific IP range) AND require IAM authentication for AWS SDK clients. They configure a resource policy to allow the IP range, but IAM-authenticated clients outside the VPN are blocked. Why?

**A:** For REST APIs, when BOTH a resource policy and IAM authentication are used, the request must satisfy **both** policies (AND logic for same-account). The resource policy denies requests from outside the IP range, so even valid IAM credentials are rejected. **Fix:** (1) Modify resource policy to allow both the IP range AND IAM principals. (2) Use condition keys in the resource policy that permit IAM-authenticated callers regardless of IP. **Key rule:** Resource policy + IAM auth = both must explicitly ALLOW (implicit deny in resource policy blocks IAM callers).

---

**Q4:** A team deploys an API Gateway HTTP API with a Cognito JWT authorizer. They test with a valid token and get `401 Unauthorized`. The same token works with the Cognito-hosted UI. What's likely wrong?

**A:** Common causes: (1) The JWT `aud` (audience) claim doesn't match the configured audience in the authorizer. HTTP API JWT authorizers require the token's `aud` to match. (2) The `iss` (issuer) URL doesn't match exactly (must include the full Cognito User Pool URL with region). (3) Token is expired. **Key difference:** REST API Cognito authorizer validates tokens automatically. HTTP API JWT authorizer requires explicit `audience` and `issuer` configuration — mismatches silently fail with 401.

---

**Q5:** A Step Functions Standard Workflow calls a third-party API via an HTTP Task (using API Gateway as the target). The API occasionally takes 45 seconds to respond. The workflow times out. How should this be redesigned?

**A:** Don't call the third-party API synchronously through API Gateway (29s hard limit). **Options:** (1) Use Step Functions **HTTP Task** (direct HTTPS endpoint invocation — supports up to the state's TimeoutSeconds, not limited to 29s). (2) Use `.waitForTaskToken` pattern: invoke Lambda that starts the API call, Lambda returns immediately, third-party sends callback with task token when done. (3) Use Lambda with SDK directly (Lambda timeout up to 15 min). **Key nuance:** Step Functions can make direct HTTP calls (HTTPS endpoints) without API Gateway, avoiding the 29s limit. But the task itself still needs `TimeoutSeconds` and `HeartbeatSeconds` configured.

---

**Q6:** An AppSync API uses a DynamoDB resolver to fetch a list of orders. Each order has a `customerId` field, and the client also wants customer details. The naive implementation causes N+1 queries (one query for orders, then one query per customer). How to fix this?

**A:** Use **AppSync Batch Resolvers**. Configure the customer resolver to use `BatchGetItem` — AppSync automatically batches individual resolver calls into a single batch DynamoDB request. Set `maxBatchSize` on the resolver. **Alternative:** Use a Pipeline Resolver that fetches orders first, extracts unique customerIds, then batch-fetches all customers in one DynamoDB call. **Key concept:** AppSync batch resolvers solve N+1 automatically by grouping individual field resolutions into batch operations.

---

**Q7:** A company uses API Gateway with a Lambda function. During a traffic spike, some requests get 502 errors. CloudWatch shows Lambda throttling errors. API Gateway has no throttling configured. What's the fix?

**A:** Lambda is hitting its **concurrency limit** (account default: 1,000 concurrent executions). API Gateway retries throttled Lambda invocations but eventually returns 502 after retries exhaust. **Fixes:** (1) Request Lambda concurrency limit increase. (2) Configure **Provisioned Concurrency** on Lambda for predictable baseline. (3) Set API Gateway **throttle limits** to keep request rate within Lambda's capacity (prevents overwhelming Lambda). (4) Use SQS between API Gateway and Lambda for buffering. **Key trap:** 502 from API Gateway often means the backend (Lambda) failed, not that API Gateway itself has a problem. Check Lambda metrics first.

---

**Q8:** A workflow needs to process items from a DynamoDB table. A developer implements: Lambda → Scan DynamoDB → Pass items to Step Functions Map state. For 10 million items, this fails. What's the architectural issue?

**A:** Multiple problems: (1) Lambda has 15-min timeout — scanning 10M items may exceed it. (2) Lambda payload return limit is 6 MB — 10M items won't fit. (3) Step Functions Map state input is 256 KB max (inline). **Correct architecture:** Use Step Functions **Distributed Map** with a DynamoDB export to S3 as the item source. Or use DynamoDB export → S3 → Distributed Map. **Key learning:** For large-scale data processing, never pass millions of items through Step Functions state payloads. Use Distributed Map with S3 as the item reader.

---

**Q9:** A company has an API Gateway WebSocket API for a chat application. They store connection IDs in DynamoDB. When sending a message to a disconnected client, the POST to `@connections/{connectionId}` returns a 410 GoneException. What should the application do?

**A:** The 410 response means the connection no longer exists. The application should: (1) Catch the 410 error. (2) Delete the stale connection ID from DynamoDB. (3) Continue sending to remaining connections. **Key design pattern:** WebSocket APIs require application-level connection lifecycle management. Always handle 410 GoneException by cleaning up stale connections. `$disconnect` route isn't guaranteed to fire (client may lose connectivity without clean disconnect).

---

**Q10:** A company uses Step Functions to orchestrate a payment workflow: Reserve Inventory → Charge Payment → Ship Order. The "Charge Payment" step succeeds, but "Ship Order" fails. The Catch block routes to a "Refund Payment" compensating action, but the refund also fails. What happens?

**A:** If the compensating action (Refund) also fails and has no further Catch/Retry → the entire execution fails in the FAILED state. **Best practice:** Compensating transactions should have their OWN Retry configuration (with aggressive retries) and a final Catch that routes to a human intervention state (SNS notification, manual queue). **Architecture:** Refund Payment → Retry (MaxAttempts: 5, BackoffRate: 2) → Catch → "Alert Operations Team" (SNS + persist to DLQ). **Key concept:** Saga compensating actions must be as resilient as forward actions. Double failures need human escalation.

---

## 9. Service Comparison Tables

### Step Functions Standard vs Express

| Dimension | Standard | Express |
|-----------|----------|---------|
| **Use Case** | Long workflows, human approval, audit | High-volume event processing, IoT, streaming |
| **Max Duration** | 1 year | 5 minutes |
| **Delivery** | Exactly-once | At-least-once (async) / At-most-once (sync) |
| **Start Rate** | 2,000/sec | 100,000/sec |
| **Pricing** | $0.025 per 1,000 state transitions | ~$1/million executions + duration |
| **History** | Full execution history (90 days) | CloudWatch Logs only |
| **HA Model** | Durable state, Multi-AZ, survives AZ failure | Stateless, Multi-AZ |
| **Exam Tip** | "audit trail", "exactly-once", "long-running" | "high volume", "short duration", "idempotent" |

### API Gateway REST vs HTTP vs WebSocket

| Dimension | REST API | HTTP API | WebSocket API |
|-----------|----------|----------|---------------|
| **Use Case** | Full-featured API management | Simple, cheap Lambda proxy | Real-time bidirectional |
| **Latency** | ~50ms overhead | ~10ms overhead | Persistent (no overhead per msg) |
| **Pricing** | $3.50/million | $1.00/million | $1.00/million msgs + connection mins |
| **Caching** | Yes | No | No |
| **WAF** | Yes | No | Yes |
| **API Keys/Plans** | Yes | No | No |
| **VPC Link Backend** | NLB only | ALB, NLB, Cloud Map | No |
| **HA Model** | Multi-AZ, edge-optimized option | Multi-AZ | Multi-AZ |
| **Exam Tip** | "caching", "WAF", "API keys", "validation" | "cheapest", "lowest latency", "simple proxy" | "chat", "real-time", "persistent connection" |

### API Gateway vs AppSync

| Dimension | API Gateway | AppSync |
|-----------|-------------|---------|
| **Use Case** | REST/HTTP APIs, microservice proxy | GraphQL, real-time, multi-source aggregation |
| **Protocol** | REST, HTTP, WebSocket | GraphQL (HTTP + WebSocket) |
| **Real-Time** | WebSocket (manual management) | Subscriptions (fully managed) |
| **Caching** | REST only (stage-level) | Resolver-level |
| **Offline** | No | Yes (Amplify DataStore) |
| **Data Sources** | Backend (Lambda, HTTP, AWS services) | DynamoDB, RDS, OpenSearch, Lambda, HTTP (direct) |
| **Auth** | IAM, Cognito, Lambda authorizer, API keys | IAM, Cognito, OIDC, Lambda, API Key (multi-mode) |
| **Pricing** | $1.00-$3.50/million | $4.00/million |
| **HA Model** | Multi-AZ, multi-region with Route 53 | Multi-AZ, multi-region with Route 53 |
| **Exam Tip** | "REST", "throttle clients", "WAF" | "GraphQL", "offline", "real-time subscriptions", "multiple data sources" |

### Step Functions vs EventBridge for Orchestration/Coordination

| Dimension | Step Functions | EventBridge |
|-----------|---------------|-------------|
| **Model** | Sequential/parallel state machine | Event-driven routing |
| **Flow Control** | Explicit (Choice, Parallel, Map, Wait) | Rules match events → fire targets |
| **Error Handling** | Retry/Catch built-in | Retry policy + DLQ |
| **Long-Running** | Yes (1 year) | No (immediate delivery) |
| **Human Interaction** | waitForTaskToken | No native support |
| **Exam Tip** | "coordinate steps", "compensating actions", "orchestrate" | "react to events", "route based on content" |

---

## 10. When to Pick Service A over Service B — Exam Keywords

### Pick Step Functions over Lambda-to-Lambda chaining when:
- "coordinate", "orchestrate" multiple services
- "error handling", "retry with backoff"
- "compensating transaction", "saga pattern"
- "wait for human approval", "callback"
- "visual workflow", "audit trail"
- "parallel processing of branches"
- **Exam signal:** Any multi-step workflow with branching/error logic

### Pick API Gateway REST over HTTP API when:
- "caching" (only REST has it)
- "WAF protection" (only REST supports it)
- "API keys", "usage plans", "throttle per client"
- "request validation" (reject bad input before Lambda)
- "request/response transformation" (Velocity templates)
- "resource policy" (IP restriction, cross-account)
- "private API" (both support, but REST has resource policies)
- **Exam signal:** Any advanced API management feature

### Pick HTTP API over REST API when:
- "cost", "cheapest", "lowest cost"
- "low latency", "minimal overhead"
- "simple proxy" to Lambda or HTTP backend
- "JWT/OIDC auth" is sufficient (no Cognito User Pools native needed)
- "no need for caching, WAF, or API keys"
- **Exam signal:** Simple use case + cost optimization

### Pick AppSync over API Gateway when:
- "GraphQL"
- "real-time subscriptions" (managed fan-out to clients)
- "offline mobile application"
- "multiple data sources in one query"
- "reduce over-fetching" (client specifies fields)
- "conflict resolution" (offline sync)
- **Exam signal:** Mobile/web app with real-time + offline needs

### Pick API Gateway WebSocket over AppSync when:
- "custom protocol over WebSocket"
- "server-initiated messages to specific clients" (not broadcast)
- "existing WebSocket infrastructure migration"
- "fine-grained per-connection routing"
- **Exam signal:** Custom bidirectional protocol, not standard pub/sub

### Pick Step Functions Express over Standard when:
- "high volume" (>2,000 executions/sec)
- "short duration" (<5 minutes)
- "cost-sensitive" + high volume
- "idempotent processing" is already in place
- "IoT event processing", "streaming event enrichment"
- **Exam signal:** Volume + short duration + cost

### Pick Step Functions Standard over Express when:
- "exactly-once" semantics required
- "long-running" (>5 minutes)
- "audit trail", "execution history"
- "human approval", "wait days"
- "must not duplicate operations" (financial transactions)
- **Exam signal:** Duration + exactly-once + audit

---

## 11. "Gotcha" Differences the Exam Tests

### API Gateway Gotchas
1. **29-second integration timeout is HARD.** Cannot be changed, not even with support request. Any question with >29s backend → async pattern.
2. **REST API resource policies + IAM auth = AND logic (same account).** Both must allow. Cross-account = AND with explicit Allow required in both.
3. **HTTP API lacks WAF.** If the question mentions WAF + cheapest → REST API (you can't use HTTP API with WAF).
4. **Edge-optimized creates a hidden CloudFront distribution.** You can't customize it. For custom CloudFront config → use Regional + your own CloudFront.
5. **API Gateway throttle is account-wide (10k RPS).** Not per-API. One API at 8k RPS leaves only 2k for all other APIs.
6. **Lambda authorizer caching uses the token/identity source as cache key.** Different tokens = cache miss. Same token = cache hit (won't re-invoke authorizer).
7. **VPC Link for REST = NLB only.** For HTTP API = ALB, NLB, or Cloud Map. If question says "ALB backend" + "API Gateway" → HTTP API with VPC Link.
8. **API Gateway payload limit = 10 MB.** Lambda payload limit = 6 MB (synchronous). If API GW → Lambda → response > 6 MB → Lambda fails, not API GW.
9. **Canary deployments are per-stage.** You promote a canary by updating the stage, not by creating a new stage.
10. **Custom domain names with edge-optimized APIs:** Certificate MUST be in us-east-1. Regional APIs: certificate in same region.

### Step Functions Gotchas
11. **256 KB state input/output limit.** Don't pass large datasets through state machine payload. Use S3 references instead.
12. **25,000 history events per execution (Standard).** Very long workflows with many iterations may hit this — use child executions or Distributed Map.
13. **Standard Workflows charge per state transition.** Even Pass states and Choice states count. Minimize unnecessary states.
14. **Express Workflows have NO execution history API.** Only CloudWatch Logs. Can't query past execution details programmatically like Standard.
15. **`.sync` integration waits for the resource.** Without `.sync`, Step Functions fires and moves on immediately. Common mistake: using Lambda without `.sync` and expecting the result.
16. **Wait state counts as one transition (entering it).** A 1-year Wait state = cheap (1 transition at start, 1 at end).
17. **`States.ALL` in Catch must be LAST.** More specific error matches must come before `States.ALL` or they'll never match.
18. **Activity tasks require polling.** Step Functions doesn't push work — workers must call `GetActivityTask`. If no worker polls within heartbeat timeout → `States.HeartbeatTimeout`.

### AppSync Gotchas
19. **AppSync subscriptions are triggered by MUTATIONS, not direct database changes.** If data changes via direct DynamoDB write (not through AppSync mutation) → no subscription fires.
20. **Multiple auth modes require directives on schema.** Without directives, the default auth mode applies to all fields. Adding a second mode without annotating fields = security gap.
21. **AppSync resolvers timeout at 30 seconds** (vs API Gateway's 29s). Not much difference, but it's a distinct limit.
22. **Pipeline resolvers share a 30-second total timeout.** All functions in the pipeline must complete within 30s combined, not 30s each.
23. **AppSync caching is resolver-level, not response-level.** Different resolvers in the same query can have different cache configs.
24. **Subscription filters are server-side.** Clients don't download all data and filter locally — AppSync filters before sending, saving bandwidth.

---

## 12. Decision Tree: Choosing the Right API/Orchestration Service

```
START: What is the primary need?
│
├─── Need to EXPOSE an API to clients?
│    ├─── GraphQL with multiple data sources? → AppSync
│    │    ├─── Need offline sync? → AppSync + Amplify DataStore
│    │    └─── Need real-time push to clients? → AppSync Subscriptions
│    ├─── REST API with advanced management?
│    │    ├─── Need caching? → REST API
│    │    ├─── Need WAF? → REST API
│    │    ├─── Need API keys/usage plans? → REST API
│    │    └─── Need request validation? → REST API
│    ├─── Simple REST/HTTP proxy (cost priority)? → HTTP API
│    └─── Bidirectional real-time? → WebSocket API or AppSync
│
├─── Need to ORCHESTRATE multi-step workflows?
│    ├─── Long-running (>5 min)? → Step Functions Standard
│    ├─── Need exactly-once? → Step Functions Standard
│    ├─── High-volume, short (<5 min)? → Step Functions Express
│    ├─── Need human approval/callback? → Step Functions + waitForTaskToken
│    └─── Saga/compensation needed? → Step Functions (Retry + Catch)
│
├─── Backend takes LONGER THAN 29 SECONDS?
│    └─── Async pattern: API GW → SQS/Step Functions → Callback/Poll
│
├─── Need to access PRIVATE backend from API?
│    ├─── Backend behind NLB? → REST API + VPC Link
│    ├─── Backend behind ALB? → HTTP API + VPC Link
│    └─── API only accessible from VPC? → Private API + VPC Endpoint
│
├─── Need REAL-TIME push to many clients?
│    ├─── Pub/sub fan-out to millions? → AppSync Subscriptions
│    ├─── Custom per-connection logic? → WebSocket API
│    └─── Simple notification? → SNS → mobile push
│
└─── Need to MINIMIZE custom code in orchestration?
     └─── Step Functions direct SDK integrations (200+ services)
```
