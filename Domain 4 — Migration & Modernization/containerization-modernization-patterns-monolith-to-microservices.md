# AWS SAP-C02 Study Notes: Containerization & Modernization Patterns (Monolith to Microservices)

---

## 1. Core Concepts & Theory

### Application Modernization Overview

**What it is:** The process of transforming legacy/monolithic applications into modern, cloud-native architectures — typically microservices running in containers or serverless — to improve agility, scalability, resilience, and cost efficiency.

**AWS Modernization Spectrum:**

```
Monolith (EC2) → Lift & Containerize → Decompose to Microservices → Serverless
     ↑                  ↑                        ↑                       ↑
  Least effort     Moderate effort        High effort              Highest effort
  Least benefit    Moderate benefit       High benefit             Highest benefit
```

---

### Monolithic Architecture

**Characteristics:**
- Single deployable unit (one codebase, one artifact)
- Shared database for all modules
- Tightly coupled components
- Vertical scaling (scale the whole app, not individual parts)
- Single point of failure — one bug can crash everything
- Long deployment cycles; small changes require full redeployment

**When monoliths are acceptable:**
- Small teams / early-stage startups
- Simple domain with few bounded contexts
- When time-to-market is more important than scalability
- When the team lacks microservices expertise

---

### Microservices Architecture

**Characteristics:**
- Independently deployable services, each owning a single business capability
- Each service has its own data store (database-per-service pattern)
- Services communicate via well-defined APIs (REST, gRPC, async messaging)
- Independent scaling per service
- Polyglot — each service can use different languages/frameworks
- Fault isolation — one service failure doesn't cascade (if designed correctly)

**Key principles:**
- **Single Responsibility:** One service = one business function
- **Loose Coupling:** Services interact via APIs/events, not shared state
- **High Cohesion:** Related functionality stays within one service
- **Database per Service:** No shared database
- **Design for Failure:** Circuit breakers, retries, bulkheads
- **Decentralized Governance:** Teams own their services end-to-end

---

### Containerization

**What it is:** Packaging an application and its dependencies into a container image that runs consistently across environments.

**AWS Container Services:**

| Service | What It Is | When to Use |
|---------|-----------|-------------|
| **Amazon ECS** | AWS-native container orchestrator | Teams wanting simpler ops, deep AWS integration |
| **Amazon EKS** | Managed Kubernetes | Teams with K8s expertise, multi-cloud portability |
| **AWS Fargate** | Serverless compute for ECS/EKS | No infrastructure management, per-task billing |
| **Amazon ECR** | Container image registry | Store, scan, and manage Docker images |
| **AWS App Runner** | Fully managed container service | Simple web apps/APIs, minimal config |
| **AWS Copilot** | CLI for ECS deployment | Opinionated ECS deployment workflows |

**ECS vs. EKS Decision:**

| Factor | ECS | EKS |
|--------|-----|-----|
| Complexity | Lower (AWS-native abstractions) | Higher (full K8s API) |
| Portability | AWS-only | Multi-cloud / hybrid (EKS Anywhere) |
| Learning curve | Moderate | Steep (K8s ecosystem) |
| Ecosystem | AWS services | Vast K8s ecosystem (Helm, Istio, etc.) |
| Fargate support | Yes | Yes |
| Exam default | "Least operational overhead" for containers | "Kubernetes" / "portability" / "existing K8s" |

**Fargate vs. EC2 Launch Type:**

| Factor | Fargate | EC2 Launch Type |
|--------|---------|-----------------|
| Management | No instances to manage | You manage EC2 fleet |
| Scaling | Per-task, automatic | Cluster auto-scaling required |
| Cost | Higher per-unit cost | Lower with Reserved/Spot instances |
| GPU | Not supported (ECS); supported (EKS on Fargate with limitations) | Supported |
| Persistent storage | EFS only (no EBS for Fargate tasks) | EBS + EFS |
| Exam trigger | "Least operational overhead" / "serverless containers" | "Cost-optimize" / "GPU" / "maximum control" |

---

### AWS App2Container (A2C)

**What it is:** A command-line tool that helps lift-and-shift existing .NET and Java applications running on-premises or on EC2 into containers without code changes.

**Workflow:**
1. **Discover** — Identifies running applications on the server
2. **Analyze** — Generates application profiles
3. **Containerize** — Creates Dockerfile and container image
4. **Deploy** — Generates deployment artifacts for ECS or EKS (CloudFormation/K8s manifests)

**Key facts:**
- Supports Windows (.NET Framework) and Linux (Java) apps
- Generates CI/CD pipeline (CodePipeline) artifacts
- No code changes required (lift-and-containerize)
- Creates optimized Dockerfiles
- Integrates with ECR for image storage

---

### AWS Microservice Extractor for .NET

**What it is:** Tool that helps decompose monolithic .NET applications into microservices. Uses visualization and AI-assisted recommendations to identify service boundaries.

---

### Strangler Fig Pattern

**The most critical modernization pattern for the exam.**

**Concept:** Incrementally replace monolith components with microservices. New features go to microservices; old features are migrated one by one. The monolith "shrinks" over time until it's completely replaced (strangled).

**Implementation on AWS:**
```
[Clients] → [API Gateway / ALB] → Route to:
                                    ├── New microservice (ECS/EKS/Lambda)
                                    ├── New microservice (ECS/EKS/Lambda)
                                    └── Legacy monolith (EC2) ← shrinks over time
```

**Key AWS services enabling Strangler Fig:**
- **API Gateway** or **ALB (path-based routing):** Routes requests to monolith or microservice based on URL path
- **Route 53 weighted routing:** Gradually shift traffic from old to new
- **AWS Migration Hub Refactor Spaces:** Purpose-built for Strangler Fig — manages the routing and incremental cutover

---

### AWS Migration Hub Refactor Spaces

**What it is:** The managed service specifically designed to implement the Strangler Fig pattern at scale.

**Components:**
- **Environment:** A multi-account container for your modernization project
- **Application:** Represents the application being modernized
- **Service:** Each microservice extracted from the monolith
- **Route:** Rules that direct traffic between monolith and microservices

**How it works:**
1. Create an environment spanning multiple AWS accounts
2. Define the monolith as the default service
3. As you build new microservices, add them as services
4. Create routes to direct specific URL paths to new microservices
5. The monolith continues handling un-migrated paths

**Networking:** Uses AWS Transit Gateway and API Gateway under the hood for cross-account routing.

---

### Decomposition Strategies

**How to identify service boundaries:**

| Strategy | Description | Example |
|----------|-------------|---------|
| **By Business Capability** | Align services to business functions | OrderService, PaymentService, InventoryService |
| **By Subdomain (DDD)** | Use Domain-Driven Design bounded contexts | Each bounded context = one microservice |
| **By Data Ownership** | Services own their data exclusively | Each service has its own database |
| **By Team Structure** | Conway's Law — services mirror team structure | Team A owns Service A |
| **By Change Frequency** | Isolate components that change often | Rapidly changing UI vs. stable billing |

---

### Communication Patterns

**Synchronous:**
| Pattern | AWS Service | Use Case |
|---------|------------|----------|
| REST API | API Gateway + ALB | Request/response, CRUD |
| gRPC | ALB (HTTP/2) or App Mesh | High-performance internal calls |
| Service Mesh | AWS App Mesh (Envoy) | Observability, traffic management, retries |
| Service Discovery | AWS Cloud Map | Dynamic service registration and DNS |

**Asynchronous (preferred for loose coupling):**
| Pattern | AWS Service | Use Case |
|---------|------------|----------|
| Event-driven | Amazon EventBridge | Cross-service events, decoupled workflows |
| Message queue | Amazon SQS | Point-to-point, buffering, load leveling |
| Pub/Sub | Amazon SNS | Fan-out to multiple consumers |
| Event streaming | Amazon Kinesis | Ordered, high-throughput event streams |
| Choreography | EventBridge + SQS/SNS | Services react to events independently |
| Orchestration | AWS Step Functions | Centralized workflow coordination |

**Saga Pattern (distributed transactions):**
- **Choreography-based:** Each service publishes events; next service listens and acts
  - AWS: SNS/EventBridge → SQS → Service → publish next event
  - Pros: Loosely coupled; Cons: Hard to track/debug
- **Orchestration-based:** Central coordinator manages the transaction steps
  - AWS: **Step Functions** (ideal — visual workflow, error handling, compensation logic)
  - Pros: Easy to understand/monitor; Cons: Single point of coordination

---

### Data Patterns for Microservices

| Pattern | Description | AWS Implementation |
|---------|-------------|-------------------|
| **Database per Service** | Each service owns its data store | RDS/DynamoDB per service |
| **Shared Database** (anti-pattern) | Multiple services share one DB | Avoid — creates coupling |
| **CQRS** | Separate read/write models | DynamoDB (write) + OpenSearch/ElastiCache (read) |
| **Event Sourcing** | Store events, derive state | Kinesis / DynamoDB Streams |
| **API Composition** | Aggregate data from multiple services | AppSync / API Gateway + Lambda |
| **Change Data Capture (CDC)** | Capture DB changes as events | DynamoDB Streams / RDS → DMS → Kinesis |

---

### Service Mesh — AWS App Mesh

**What it is:** A service mesh that provides application-level networking using Envoy proxies as sidecars.

**Capabilities:**
- Traffic management (routing, retries, timeouts, circuit breakers)
- Observability (metrics, traces, logs via X-Ray integration)
- Security (mTLS between services)
- Works with ECS, EKS, EC2, and Fargate

**Components:**
- **Mesh:** Logical grouping of services
- **Virtual Node:** Pointer to a service (ECS service, K8s deployment)
- **Virtual Service:** Abstraction that routes to virtual nodes
- **Virtual Router:** Routes traffic based on rules (path, header, weight)
- **Virtual Gateway:** Ingress point for external traffic

**When to use:**
- Need consistent observability across all services
- Canary/blue-green deployments at the service level
- mTLS between all services without application changes
- Complex traffic routing (A/B testing, weighted routing)

---

### Service Discovery — AWS Cloud Map

**What it is:** A managed service registry that allows services to discover each other by name.

**Features:**
- DNS-based discovery (services register and resolve via DNS)
- API-based discovery (HTTP API for non-DNS consumers)
- Health checking (integrated with Route 53 health checks)
- Works with ECS (automatic registration), EKS, EC2, Lambda

**ECS integration:** When you enable service discovery on an ECS service, tasks automatically register/deregister in Cloud Map.

---

### Key Limits & Quotas

| Resource | Limit |
|----------|-------|
| ECS clusters per Region | 10,000 |
| ECS services per cluster | 5,000 |
| ECS tasks per service | 5,000 |
| EKS clusters per Region | 100 |
| ECR repositories per Region | 10,000 |
| Fargate task size (CPU) | 16 vCPU max |
| Fargate task size (memory) | 120 GB max |
| App Mesh meshes per account | 15 |
| Cloud Map namespaces per Region | 50 |

---

## 2. Design Patterns & Best Practices

### Modernization Approaches (by effort level)

| Approach | Effort | Description | AWS Services |
|----------|--------|-------------|-------------|
| **Re-host (containerize)** | Low | Wrap monolith in container, deploy as-is | A2C → ECR → ECS/EKS |
| **Re-platform** | Medium | Containerize + use managed services (RDS, ElastiCache) | ECS + RDS + ElastiCache |
| **Refactor (Strangler Fig)** | High | Incrementally extract microservices | Refactor Spaces + ECS/EKS + SQS/SNS |
| **Re-architect** | Very High | Rewrite from scratch as microservices/serverless | Lambda + API Gateway + DynamoDB |

### Best Practices for Decomposition

1. **Start with the monolith** — don't build microservices from day one for a new project (unless domain is well-understood)
2. **Extract the easiest service first** — build team confidence
3. **Use the Strangler Fig pattern** — never attempt a "big bang" rewrite
4. **Define clear API contracts** — use OpenAPI/Swagger or protobuf
5. **Implement asynchronous communication** where possible — reduces coupling
6. **One database per service** — the hardest but most important rule
7. **Centralize observability** — distributed tracing (X-Ray), centralized logging (CloudWatch), metrics
8. **Automate everything** — CI/CD per service (CodePipeline per microservice)

### Anti-Patterns

| Anti-Pattern | Why It's Bad | Fix |
|--------------|-------------|-----|
| **Distributed monolith** | Microservices that are tightly coupled and must deploy together | Ensure loose coupling; async communication |
| **Shared database** | Changes to schema break multiple services | Database per service + events for data sharing |
| **Chatty services** | Too many synchronous calls between services | Aggregate APIs, use async messaging, CQRS |
| **Big Bang rewrite** | High risk, long timeline, no incremental value | Use Strangler Fig pattern |
| **Nano-services** | Services too small — overhead exceeds benefit | Merge related nano-services; align to business capabilities |
| **No service discovery** | Hardcoded endpoints break when services scale/move | Use Cloud Map or DNS-based discovery |
| **No circuit breaker** | One failing service cascades failures | Use App Mesh retries/circuit breakers or application-level (Hystrix pattern) |

### Integration Patterns

**API Gateway Pattern:**
```
[Clients] → [API Gateway] → [Microservice A]
                           → [Microservice B]
                           → [Microservice C]
```
- Single entry point for all clients
- Handles auth, throttling, request routing
- **AWS:** API Gateway (REST/HTTP) or ALB (path-based routing)

**Backend for Frontend (BFF):**
```
[Mobile App] → [Mobile BFF (Lambda)] → [Services]
[Web App]    → [Web BFF (Lambda)]    → [Services]
```
- Separate API layer per client type
- Each BFF aggregates and transforms data for its client
- **AWS:** Separate API Gateway stages or Lambda functions per client type

**Sidecar Pattern:**
- Attach a helper process alongside the main service container
- Used for logging, monitoring, service mesh proxy
- **AWS:** App Mesh Envoy sidecar in ECS task definitions / EKS pods

**Ambassador Pattern:**
- A proxy that handles cross-cutting concerns (auth, retries, TLS)
- Similar to sidecar but specifically for outbound traffic
- **AWS:** App Mesh virtual gateway

---

### Well-Architected Alignment

| Pillar | Guidance |
|--------|----------|
| **Reliability** | Fault isolation per service; circuit breakers; independent scaling; health checks via ALB/Cloud Map |
| **Security** | mTLS via App Mesh; IAM roles per task/pod; secrets in Secrets Manager/Parameter Store; ECR image scanning |
| **Cost** | Right-size containers; use Fargate Spot for non-critical workloads; consolidate nano-services |
| **Performance** | Independent scaling per service; caching at API Gateway; async processing for heavy workloads |
| **Operational Excellence** | CI/CD per service; distributed tracing (X-Ray); centralized logging; Infrastructure as Code |
| **Sustainability** | Right-size tasks; use Graviton (ARM) instances for ECS/EKS; Fargate reduces idle capacity |

---

## 3. Security & Compliance

### Container Security

**Image Security:**
- **ECR image scanning:** Automated vulnerability scanning on push (Basic = Clair; Enhanced = Inspector)
- **Immutable tags:** Prevent image tag overwrites in ECR
- **Image signing:** Use Docker Content Trust or AWS Signer for image integrity
- **Base images:** Use minimal base images (distroless, Alpine); scan regularly
- **No secrets in images:** Never bake credentials into Dockerfiles

**Runtime Security:**
- **IAM Roles per Task (ECS):** Each task definition has its own IAM role — least privilege per service
- **IAM Roles for Service Accounts (EKS — IRSA):** Map K8s service accounts to IAM roles
- **EKS Pod Identity:** Newer, simpler alternative to IRSA for IAM-to-pod mapping
- **Read-only root filesystem:** Set `readonlyRootFilesystem: true` in task/pod definitions
- **Non-root user:** Run containers as non-root
- **Security groups:** Apply at the ENI level (awsvpc network mode in ECS, pod-level in EKS)
- **Network policies (EKS):** Use Calico or VPC CNI network policies for pod-to-pod restrictions

**Secrets Management:**
- **AWS Secrets Manager:** Store and rotate database credentials, API keys
- **Systems Manager Parameter Store:** Store configuration and non-rotating secrets
- **ECS:** Reference secrets in task definitions → injected as environment variables at runtime
- **EKS:** Use External Secrets Operator or Secrets Store CSI Driver to sync AWS secrets into K8s secrets
- **Never** use environment variables in plaintext for sensitive data in task definitions visible in console

**Network Security:**
- **awsvpc network mode (ECS):** Each task gets its own ENI + security group — network isolation at task level
- **Private subnets:** Run containers in private subnets; use NAT Gateway or VPC endpoints for AWS API access
- **VPC endpoints:** PrivateLink for ECR, S3, CloudWatch, Secrets Manager — no internet exposure
- **App Mesh mTLS:** Encrypt service-to-service traffic without application changes

### Compliance Patterns

- **Multi-tenancy isolation:** Separate ECS clusters or namespaces (EKS) per tenant
- **Audit:** CloudTrail logs all ECS/EKS API calls; enable EKS audit logging
- **Compliance scanning:** Use AWS Config rules for container-related resources
- **Runtime protection:** AWS GuardDuty for EKS (detects crypto mining, privilege escalation, etc.)

---

## 4. Cost Optimization

### Container Pricing Models

**ECS on EC2:**
- Pay for the underlying EC2 instances
- Optimize with Reserved Instances, Savings Plans, or **Spot Instances** for fault-tolerant workloads
- Risk: cluster over-provisioning (idle capacity)

**ECS/EKS on Fargate:**
- Pay per vCPU-second and GB-second (per task)
- No idle capacity charges — you pay only for running tasks
- **Fargate Spot:** Up to 70% discount for interruptible tasks
- More expensive per unit than EC2 at high sustained utilization

**EKS Control Plane:**
- $0.10/hour per cluster ($73/month) regardless of worker nodes
- Consolidate workloads into fewer clusters where appropriate

**App Runner:**
- Pay per vCPU-second and GB-second when processing requests
- Minimum instances incur reduced charges when idle
- Simplest pricing but least control

### Cost Optimization Strategies

| Strategy | When to Use |
|----------|-------------|
| Fargate Spot | Non-critical batch processing, dev/test, workers that tolerate interruption |
| EC2 Spot (ECS/EKS) | Large-scale processing, CI/CD runners, stateless workers |
| Graviton (ARM) instances | All workloads that support ARM — ~20% cost reduction |
| Right-sizing tasks | Monitor CPU/memory usage; reduce over-provisioned task definitions |
| Scale to zero (App Runner / Fargate) | Low-traffic services that don't need always-on capacity |
| Cluster consolidation | Merge small EKS clusters; use namespaces for isolation |
| Karpenter (EKS) | Just-in-time node provisioning — minimizes over-provisioning |

### Cost Traps

- **Too many EKS clusters:** $73/month per cluster adds up fast with dozens of clusters
- **Fargate for steady-state high utilization:** More expensive than EC2 Reserved Instances
- **Over-provisioned task definitions:** 4 vCPU / 8 GB when the app uses 0.5 vCPU / 1 GB
- **NAT Gateway data processing charges:** Containers in private subnets pulling images from internet ECR → use VPC endpoints
- **ECR storage:** Old images accumulate — use lifecycle policies to expire untagged/old images
- **Cross-AZ traffic:** Service-to-service calls across AZs incur data transfer — use topology-aware routing in EKS

---

## 5. High Availability, Disaster Recovery & Resilience

### Container HA Patterns

**ECS HA:**
- Deploy services across **multiple AZs** (ECS places tasks across AZs automatically when using multiple subnets)
- Use **ALB health checks** — unhealthy tasks are drained and replaced
- Set **minimum healthy percent** and **maximum percent** in service definition for rolling updates
- **ECS Service Auto Scaling:** Target tracking (CPU/memory), step scaling, or scheduled

**EKS HA:**
- EKS control plane is **multi-AZ by default** (managed by AWS)
- Deploy worker nodes across **multiple AZs** (node groups in each AZ)
- Use **Pod Disruption Budgets (PDB)** to ensure minimum availability during updates
- **Horizontal Pod Autoscaler (HPA):** Scale pods based on metrics
- **Cluster Autoscaler / Karpenter:** Scale nodes to meet pod demand
- **Pod anti-affinity rules:** Spread replicas across AZs/nodes

**Fargate HA:**
- Specify multiple subnets across AZs → Fargate distributes tasks automatically
- No node management — AWS handles underlying infrastructure HA

### Resilience Patterns

| Pattern | Description | AWS Implementation |
|---------|-------------|-------------------|
| **Circuit Breaker** | Stop calling a failing service | App Mesh retry/timeout policies; application-level (Resilience4j) |
| **Bulkhead** | Isolate failures to a subset | Separate ECS services/task definitions; separate thread pools |
| **Retry with backoff** | Retry transient failures | App Mesh retry policies; SDK retries; Step Functions retries |
| **Health checks** | Detect and replace unhealthy instances | ALB health checks; ECS health checks; K8s liveness/readiness probes |
| **Graceful degradation** | Return partial results when a dependency is down | Feature flags; cached responses; fallback responses |
| **Timeout** | Don't wait indefinitely | App Mesh timeout; API Gateway timeout (29s max); ALB idle timeout |

### Multi-Region DR for Containers

| DR Strategy | RTO | RPO | Implementation |
|-------------|-----|-----|----------------|
| **Backup & Restore** | Hours | Hours | ECR replication; RDS snapshots cross-Region |
| **Pilot Light** | 10-30 min | Minutes | ECR cross-Region replication; minimal ECS tasks in DR Region; scale up on failover |
| **Warm Standby** | Minutes | Seconds-Minutes | Running (scaled-down) ECS services in DR; Route 53 failover |
| **Active-Active** | Near-zero | Near-zero | ECS services in both Regions; Global Accelerator or Route 53 latency routing; DynamoDB Global Tables |

**ECR Cross-Region Replication:** Automatically replicate container images to DR Region — essential for all DR strategies.

---

## 6. Exam-Focused Section

### Straightforward Questions (7 examples)

**Q1:** A company wants to containerize their monolithic .NET application running on Windows servers with minimal code changes. Which AWS tool should they use?
**A:** **AWS App2Container (A2C).** It analyzes running applications and generates container artifacts (Dockerfile, ECS/EKS deployment configs) without requiring code changes.

**Q2:** A team is implementing the Strangler Fig pattern to incrementally migrate a monolith to microservices. Which AWS service is purpose-built for managing this migration?
**A:** **AWS Migration Hub Refactor Spaces.** It manages the routing infrastructure (API Gateway + Transit Gateway) to incrementally redirect traffic from monolith to microservices.

**Q3:** A company needs to run containers without managing servers. They want the least operational overhead. What should they use?
**A:** **AWS Fargate** (with ECS or EKS). Serverless compute for containers — no instances to patch, scale, or manage.

**Q4:** What is the recommended approach for data management in a microservices architecture?
**A:** **Database per service.** Each microservice owns its data store exclusively. Services share data via APIs or events, never by direct database access.

**Q5:** A company needs distributed transactions across multiple microservices. Which AWS service is best for orchestration-based sagas?
**A:** **AWS Step Functions.** It coordinates multi-step workflows with built-in error handling, retries, and compensation logic.

**Q6:** How should secrets be provided to containers running on ECS Fargate?
**A:** Reference secrets from **AWS Secrets Manager** or **Systems Manager Parameter Store** in the task definition. ECS injects them as environment variables at runtime — secrets are never stored in the image.

**Q7:** A team wants mutual TLS between all their microservices running on ECS without modifying application code. What should they use?
**A:** **AWS App Mesh** with mTLS enabled. The Envoy sidecar proxy handles TLS termination/origination transparently.

---

### Tricky / Scenario-Based Questions (7 examples)

**Q1:** A company has a monolithic e-commerce application. They want to extract the payment processing module first because it has different scaling requirements. The remaining monolith must continue functioning. They need the LEAST disruption during migration. What approach should they use?

**A:** Use the **Strangler Fig pattern** with **ALB path-based routing**. Deploy the new payment microservice on ECS/Fargate. Configure the ALB to route `/payments/*` to the new service and all other paths to the monolith. No client changes needed.

*Why wrong:*
- Big bang rewrite: High risk, high disruption — violates "least disruption"
- API Gateway only: Works, but ALB is simpler for internal HTTP routing within VPC
- Re-deploy everything as containers first: Unnecessary — extract one service at a time

**Keywords:** "extract one module" + "least disruption" + "continue functioning" → Strangler Fig

---

**Q2:** A company has decomposed their application into 15 microservices on EKS. They're experiencing cascading failures — when the inventory service slows down, the order service and the frontend service also become unresponsive. What should they implement?

**A:** Implement **circuit breakers** and **timeouts** using **AWS App Mesh**. Configure retry policies with exponential backoff and circuit breaker thresholds on the virtual nodes. Additionally, implement **bulkhead isolation** by ensuring each service has its own resource limits (K8s resource requests/limits).

*Why wrong:*
- Just adding more replicas: Doesn't solve cascading failures — more replicas will also hang
- Synchronous → asynchronous: Good long-term fix but not immediate; question asks what to "implement"
- Auto Scaling alone: Scaling won't help if requests are blocked waiting for a slow service

**Keywords:** "cascading failures" + "one service slows down" → Circuit breaker + timeout

---

**Q3:** A company is migrating to microservices and needs to maintain data consistency for their order processing workflow that spans Order, Payment, and Inventory services. They cannot use distributed transactions (2PC). They need full visibility into the workflow state. What should they use?

**A:** **Orchestration-based Saga pattern** using **AWS Step Functions**. Step Functions coordinates the Order → Payment → Inventory sequence, with compensation steps (rollback) if any step fails. The state machine provides full visibility into workflow state.

*Why wrong:*
- Choreography-based saga (EventBridge + SQS): Works but "full visibility into workflow state" is harder — no central view
- DynamoDB Transactions: Only works within a single DynamoDB table/account, not across services
- Two-phase commit (2PC): Question explicitly excludes this; doesn't scale in microservices

**Keywords:** "data consistency across services" + "full visibility" + "no 2PC" → Step Functions (orchestration saga)

---

**Q4:** A company runs 50 microservices on ECS Fargate. Their monthly AWS bill is high. Analysis shows most services have steady-state usage at 80%+ CPU utilization 24/7. How should they reduce costs with MINIMAL architectural changes?

**A:** Migrate from **Fargate to ECS on EC2** with **Compute Savings Plans** or **Reserved Instances**. At 80% steady-state utilization, EC2 with commitments is significantly cheaper than Fargate's per-second billing.

*Why wrong:*
- Fargate Spot: Not suitable for production services requiring high availability
- Right-sizing tasks: Already at 80% utilization — they're well-sized
- Reducing services / consolidating: "Minimal architectural changes" — this would be major

**Keywords:** "steady-state 80% utilization" + "reduce costs" + "minimal changes" → EC2 with commitments (not Fargate)

---

**Q5:** A team is containerizing their application for ECS. The application stores session state in memory. After deploying to ECS with multiple tasks behind an ALB, users report being randomly logged out. What's the issue and fix?

**A:** In-memory session state is lost when requests hit different tasks. Fix: **Externalize session state** to **Amazon ElastiCache (Redis)** or **DynamoDB**. Alternatively, enable **ALB sticky sessions** (but this limits scalability).

*Why wrong:*
- Increase task memory: Doesn't solve the problem — state isn't shared between tasks
- Enable ECS service discovery: This is for inter-service communication, not session management
- Use a single task: Defeats the purpose of scaling

**Keywords:** "randomly logged out" + "multiple tasks" + "session state" → Externalize state to ElastiCache/DynamoDB

---

**Q6:** A company has ECS services in us-east-1 and wants near-zero RTO for container workloads. They use DynamoDB Global Tables for data. What's the optimal multi-Region architecture?

**A:** **Active-Active** multi-Region: Deploy ECS services in both Regions with ECR cross-Region replication. Use **Global Accelerator** or **Route 53 latency-based routing** for traffic distribution. DynamoDB Global Tables handle data replication. If one Region fails, traffic automatically routes to the other.

*Why wrong:*
- Pilot Light: RTO is 10-30 minutes (need to scale up) — not "near-zero"
- Warm Standby: RTO is minutes — close but not "near-zero"
- Single-Region with Multi-AZ: Not multi-Region DR

**Keywords:** "near-zero RTO" + "multi-Region" → Active-Active

---

**Q7:** A company runs 200 microservices across 20 EKS clusters (one per team). Operations team is overwhelmed with cluster management. How should they consolidate while maintaining team isolation?

**A:** Consolidate into **fewer multi-tenant EKS clusters** using **Kubernetes namespaces** for team isolation. Apply **Resource Quotas** and **Network Policies** per namespace. Use **RBAC** to restrict team access to their namespace only. This reduces cluster overhead while maintaining isolation.

*Why wrong:*
- Keep 20 clusters but use managed node groups: Still 20 clusters to manage
- Move everything to Fargate: May not be compatible with all workloads (DaemonSets, etc.)
- Move to ECS: Major re-architecture — not a consolidation strategy

**Keywords:** "too many clusters" + "maintain isolation" → Consolidate clusters + namespaces + RBAC

---

### Common Exam Traps & Pitfalls

1. **"Least operational overhead" for containers = Fargate** — not EC2 launch type, not self-managed K8s

2. **"Kubernetes" or "portability" in the question = EKS** — not ECS. ECS is AWS-only.

3. **Strangler Fig ≠ Big Bang rewrite** — the exam ALWAYS prefers incremental migration over rewrite

4. **App2Container ≠ refactoring** — A2C is lift-and-containerize only. It doesn't decompose monoliths into microservices.

5. **Step Functions for orchestration sagas** — if the question mentions "visibility" or "central coordination" of distributed workflows, it's Step Functions, not choreography

6. **Database per service ≠ impossible** — exam expects this as the correct answer even though it's hard. Shared databases are always wrong in microservices questions.

7. **App Runner vs. Fargate** — App Runner is "simplest" for web apps (no VPC config, no task definitions, auto-scaling built-in). Fargate is for "more control" while still serverless.

8. **ECS Service Connect vs. App Mesh vs. Cloud Map:**
   - Cloud Map = service discovery only
   - App Mesh = full service mesh (traffic management, observability, mTLS)
   - ECS Service Connect = simplified service mesh built into ECS (simpler than App Mesh)

9. **Fargate doesn't support all workloads** — no GPUs (ECS Fargate), no DaemonSets, no privileged containers. If the question mentions these → EC2 launch type.

10. **ECR replication is needed for multi-Region DR** — if you don't replicate images, DR Region can't pull them if the primary Region is down.

---

## 7. Cheat Sheet

### Must-Know Facts

- **Strangler Fig** = incrementally extract microservices using routing (ALB/API Gateway)
- **Refactor Spaces** = managed Strangler Fig service (API Gateway + Transit Gateway under the hood)
- **App2Container** = lift-and-containerize (.NET/Java), no code changes, generates Dockerfile + deployment
- **Fargate** = serverless containers, exam's go-to for "least operational overhead"
- **ECS** = AWS-native, simpler; **EKS** = Kubernetes, portable, richer ecosystem
- **App Mesh** = service mesh (Envoy sidecar), mTLS, traffic management, observability
- **Cloud Map** = service discovery (DNS + API-based)
- **Step Functions** = orchestration-based saga (visibility, compensation, error handling)
- **Database per service** = always the correct microservices data pattern on the exam
- **Circuit breaker** = stops cascading failures (App Mesh or application-level)
- **ECR image scanning** = Enhanced (Inspector-based) or Basic (Clair-based)
- **awsvpc network mode** = task-level security groups in ECS

### Decision Flowchart

```
"Least operational overhead" + containers
  → Fargate (ECS or EKS)

"Simple web app" + "no infrastructure management" + "auto-deploy from source"
  → App Runner

"Kubernetes" OR "portability" OR "existing K8s skills"
  → EKS

"Deep AWS integration" + "simpler than K8s"
  → ECS

"Containerize existing app without code changes"
  → App2Container

"Incrementally replace monolith"
  → Strangler Fig pattern (ALB path routing or Refactor Spaces)

"Distributed workflow" + "coordination" + "visibility"
  → Step Functions (orchestration saga)

"Decouple services" + "event-driven"
  → EventBridge / SQS / SNS (choreography)

"Service-to-service encryption" + "no code changes"
  → App Mesh (mTLS)

"Cascading failures" + "timeout"
  → Circuit breaker (App Mesh / application-level)

"Cost-optimize steady-state containers"
  → EC2 launch type + Savings Plans

"GPU workloads in containers"
  → EC2 launch type (ECS) or EKS managed node groups (not Fargate)
```

### Key Differentiators

| Signal in Question | Answer |
|---|---|
| "Serverless containers" / "no servers" | Fargate |
| "Kubernetes" / "Helm" / "K8s" / "portability" | EKS |
| "Simplest container deployment" / "just a container image" | App Runner |
| "Migrate .NET/Java to containers" | App2Container |
| "Decompose .NET monolith into services" | Microservice Extractor for .NET |
| "Incremental migration" / "route traffic" | Strangler Fig / Refactor Spaces |
| "mTLS" / "service mesh" / "traffic policies" | App Mesh |
| "Service discovery" / "find services by name" | Cloud Map |
| "Distributed transaction" / "saga" / "compensate" | Step Functions |
| "Cascading failure" / "one service brings down others" | Circuit breaker (App Mesh) |
| "Cross-Region container DR" | ECR replication + ECS/EKS in DR Region |

---

## 8. Additional Deep-Dive

### 10 More Tricky Scenario-Based Questions

**Q1:** A company is planning their microservices architecture. The CTO insists on starting with a "big bang" rewrite of their 10-year-old monolith. The monolith handles 50,000 requests/second and any downtime costs $100K/minute. As the solutions architect, what do you recommend?

**A:** Strongly recommend the **Strangler Fig pattern** instead. Use **Refactor Spaces** or **ALB path-based routing** to incrementally extract services while the monolith continues handling production traffic. Extract the lowest-risk, most independent services first. Zero downtime during migration because the monolith stays live until each service is proven in production.

*Why "big bang" is wrong:*
- High risk of failure (10 years of business logic is complex)
- No production traffic during rewrite phase (months of development without validation)
- $100K/min downtime cost makes any cutover risk unacceptable
- Industry data: most big-bang rewrites fail or exceed timeline by 3-5x

---

**Q2:** A company has 30 microservices. Five services are latency-sensitive (need <100ms response) and handle variable traffic (0 to 10,000 RPS). The other 25 services are internal batch processors that run at steady 70% utilization. How should they optimize costs?

**A:** 
- **5 latency-sensitive services:** ECS on **Fargate** with auto-scaling (pay only for active requests, scale to zero possible with App Runner or scale-in with Fargate)
- **25 batch processors:** ECS on **EC2 with Savings Plans** (steady-state high utilization = EC2 wins on cost) + **Spot Instances** for additional batch capacity

*Why wrong:*
- All on Fargate: Over-paying for the steady 70% utilization services
- All on EC2: Over-provisioning for the variable traffic services
- All on Spot: Latency-sensitive services can't tolerate interruptions

---

**Q3:** A company migrated their monolith to 12 microservices but kept the shared PostgreSQL database. Now they can't deploy services independently — schema changes break other services. What should they do?

**A:** Implement **database per service** pattern. Strategy:
1. Identify each service's data ownership (tables it primarily reads/writes)
2. Create separate schemas or databases per service
3. Use **change data capture** (DMS with CDC mode or logical replication) to sync data between services during migration
4. Replace direct DB queries between services with **API calls** or **events** (DynamoDB Streams, EventBridge)
5. Eventually each service owns its database — no shared schema

*Why tricky:* Candidates might suggest "just add database views" (still couples services) or "use a message queue" (doesn't solve the shared schema problem directly).

---

**Q4:** A healthcare company has EKS workloads that process PHI. They need: (a) container images scanned for vulnerabilities before deployment, (b) runtime detection of suspicious behavior, (c) encryption between all pods, and (d) audit trail of all API calls. What combination of services?

**A:**
- (a) **ECR Enhanced Scanning** (Amazon Inspector) — scans images on push
- (b) **Amazon GuardDuty for EKS** — detects runtime threats (crypto mining, privilege escalation, anomalous API calls)
- (c) **AWS App Mesh with mTLS** — encrypts pod-to-pod traffic without application changes
- (d) **EKS Audit Logging** (to CloudWatch Logs) + **CloudTrail** for AWS API calls

*Why wrong alternatives:*
- Third-party scanning tools: Valid but not "AWS-native" answer for the exam
- Network Policies alone: Restrict traffic but don't encrypt it
- VPC Flow Logs: Show network metadata, not API-level audit

---

**Q5:** A startup deploys their API on App Runner. Traffic grows and they now need WebSocket support, VPC access to their RDS database, and custom domain with mutual TLS for B2B clients. Can App Runner handle this?

**A:** **Partially.** App Runner supports VPC access (VPC Connector) and custom domains. However, App Runner does **NOT support WebSockets** or **mutual TLS (mTLS) at the client level**. They should migrate to **ECS Fargate + ALB** (which supports WebSockets and mTLS via custom certificates).

*Why tricky:* Candidates might assume App Runner handles everything since it's "fully managed." Knowing App Runner's limitations is key.

---

**Q6:** A company wants to implement blue-green deployments for their ECS services. They have 10 services that must be deployed together (coordinated release). Which deployment approach works?

**A:** Use **AWS CodeDeploy** with **ECS blue-green deployment**. CodeDeploy manages the blue-green switch at the ALB target group level. For coordinated deployment of 10 services, use **CodePipeline** with a **parallel deployment stage** that triggers CodeDeploy for all 10 services simultaneously, with manual approval gates.

*Why wrong:*
- ECS rolling update: Doesn't support blue-green (only rolling replacement)
- Manual target group switching: Error-prone, not automated
- Each service independently: Question requires coordinated release

**Keywords:** "blue-green" + "ECS" → CodeDeploy (ECS blue-green type)

---

**Q7:** An organization notices their ECS tasks take 3 minutes to start because they pull a 2GB container image from ECR each time. This causes slow scaling. How should they fix this?

**A:** Enable **SOCI (Seekable OCI) indexing** in ECR. SOCI allows lazy-loading of container images — the container starts before the full image is downloaded (only loads what's needed immediately). Additionally:
- Use **multi-stage builds** to reduce image size
- Use ECR in the **same Region** as ECS tasks
- Consider **ECR pull-through cache** if using public images

*Why wrong:*
- Caching on host: Only works for EC2 launch type, not Fargate (no persistent host)
- Larger task CPU: Doesn't speed up network image pull
- Pre-pull with init containers: Not supported in Fargate

---

**Q8:** A company uses Step Functions for their order saga. The workflow processes 1 million orders/day with an average execution time of 30 seconds per order. They're concerned about cost. What's the most cost-effective Step Functions option?

**A:** Use **Step Functions Express Workflows** (not Standard). Express Workflows are priced by duration and executions, optimized for high-volume, short-duration workloads. At 1M executions/day with 30-second duration, Express is significantly cheaper than Standard (which charges per state transition).

*Key difference:*
- **Standard:** Exactly-once execution, up to 1 year, charges per state transition ($0.025 per 1000 transitions)
- **Express:** At-least-once, up to 5 minutes, charges by duration (per 100ms) — much cheaper for high-volume short workflows

**Keywords:** "high volume" + "short duration" + "cost" → Express Workflows

---

**Q9:** A team has 8 microservices on EKS. Each team deploys independently, but they need to test interactions between services before production. How should they implement integration testing?

**A:** Use **EKS namespaces** for isolated test environments. Deploy full service sets in a `staging` namespace with service mesh routing (App Mesh or ECS Service Connect). Use **contract testing** (Pact or similar) for API compatibility. For full integration tests, deploy ephemeral environments per PR using **GitOps** (ArgoCD/Flux) that create temporary namespaces with all 8 services.

*Why wrong:*
- Test in production with feature flags: Risky for integration testing
- Mock all other services: Defeats the purpose of integration testing
- Single shared staging: Conflicts when multiple teams test simultaneously

---

**Q10:** An e-commerce company has a monolithic order processing system that handles 5,000 orders/hour. During Black Friday, it crashes at 50,000 orders/hour. They want to modernize for scalability but the migration must be complete in 6 months. Which parts should they extract first?

**A:** Extract the **most stateless, highest-load** components first:
1. **Product catalog** (read-heavy, stateless) → ECS/Fargate + ElastiCache + DynamoDB
2. **Order intake** (write-heavy, the bottleneck) → API Gateway + SQS + Lambda/ECS (decouple intake from processing)
3. Keep payment, fulfillment in monolith initially (extract later)

Use **SQS** to decouple order intake from processing — this alone solves the Black Friday crash (buffer 50K orders, process at steady rate). The Strangler Fig pattern with ALB routes traffic to new services for extracted paths.

*Why this order:* Highest business impact first (order intake is the crash point); catalog is lowest risk to extract (stateless, read-heavy).

---

### Compare: ECS vs. EKS vs. App Runner vs. Lambda

| Dimension | ECS (Fargate) | EKS (Fargate) | App Runner | Lambda |
|-----------|--------------|---------------|------------|--------|
| **Abstraction** | Task definitions | Pods + K8s API | Container/source → service | Functions |
| **Max execution** | Unlimited | Unlimited | Unlimited | 15 minutes |
| **Scaling unit** | Task | Pod | Instance | Function invocation |
| **Cold start** | Seconds | Seconds | Seconds (kept warm) | ms to seconds |
| **Portability** | AWS only | Multi-cloud (K8s) | AWS only | AWS only |
| **Complexity** | Medium | High | Low | Low-Medium |
| **WebSocket** | Yes (via ALB) | Yes | No | Yes (via API GW) |
| **GPU** | No (Fargate) | Yes (EC2 nodes) | No | No |
| **VPC** | Full control | Full control | VPC Connector | VPC optional |
| **Exam trigger** | "Containers" + "AWS" | "Kubernetes" / "portable" | "Simplest" / "just deploy" | "Event-driven" / "<15 min" |

### When Does the Exam Expect ECS vs. EKS?

**Pick ECS when:**
- "Least operational overhead" + containers (ECS Fargate)
- "Deep AWS integration" (native IAM, CloudWatch, X-Ray)
- "Simple container orchestration"
- Team has no Kubernetes experience
- "AWS-native"

**Pick EKS when:**
- "Kubernetes" mentioned explicitly
- "Multi-cloud" or "hybrid" (EKS Anywhere)
- "Existing K8s manifests" / "Helm charts"
- "Portability" between cloud providers
- Team has Kubernetes expertise
- "Open-source ecosystem" (Istio, Prometheus, etc.)

**Pick App Runner when:**
- "Simplest way to deploy a container"
- "No infrastructure management at all"
- "Web application or API" (simple use case)
- "Source code or container image → running service"

**Pick Lambda when:**
- "Event-driven"
- "Pay only when invoked"
- Execution < 15 minutes
- "Serverless" without container management
- Glue between AWS services

### All "Gotcha" Differences: ECS vs. EKS

| Feature | ECS | EKS |
|---------|-----|-----|
| Task/Pod identity | Task IAM Role | IRSA / Pod Identity |
| Service mesh | App Mesh or Service Connect | App Mesh, Istio, Linkerd |
| Auto-scaling nodes | ECS Capacity Providers | Cluster Autoscaler / Karpenter |
| Spot support | Fargate Spot + EC2 Spot | EC2 Spot node groups + Karpenter |
| DaemonSets | Not supported (use sidecar) | Supported |
| StatefulSets | Not native (use EBS + placement) | Supported |
| Windows containers | Supported | Supported |
| GPU | EC2 launch type only | EC2 node groups |
| Service discovery | Cloud Map (built-in) | CoreDNS + Cloud Map |
| Deployment | Rolling / Blue-Green (CodeDeploy) | Rolling / Blue-Green / Canary (Argo/Flux/CodeDeploy) |
| Privileged containers | Supported (EC2 only) | Supported |
| Multi-tenancy | Clusters or accounts | Namespaces + RBAC + Network Policies |
| Control plane cost | Free | $0.10/hour ($73/month) |
| Max pods per node | N/A (task-based) | Limited by ENI capacity / prefix delegation |

### Decision Tree for Modernization Approach

```
START: Is the application already containerized?
├── Yes → Deploy to ECS/EKS/Fargate (choose by team skills + requirements)
└── No
    ├── Can you modify code?
    │   ├── Yes
    │   │   ├── Want to decompose into microservices?
    │   │   │   ├── Yes → Strangler Fig pattern + Refactor Spaces
    │   │   │   └── No → Containerize monolith (A2C or manual Dockerfile)
    │   │   └── Want serverless?
    │   │       ├── Yes → Refactor to Lambda + API Gateway
    │   │       └── No → Containerize + ECS/EKS
    │   └── No (lift-and-containerize)
    │       ├── .NET or Java → App2Container
    │       └── Other → Manual Dockerfile creation
    └── Time constraint?
        ├── < 3 months → Containerize monolith as-is (no decomposition)
        ├── 3-12 months → Strangler Fig (extract 3-5 key services)
        └── 12+ months → Full microservices decomposition
```

---

*End of study notes for Containerization & Modernization Patterns (Monolith to Microservices)*
