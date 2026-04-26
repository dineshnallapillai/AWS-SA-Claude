# AWS SAP-C02 Study Notes: ECS / EKS / Fargate (When to Use Which)

---

## 1. Core Concepts & Theory

### Container Orchestration on AWS — The Big Picture

AWS offers three container orchestration services and two launch types:

| Service | What It Is | Control Plane |
|---------|-----------|---------------|
| **ECS (Elastic Container Service)** | AWS-native container orchestrator | Managed by AWS (free) |
| **EKS (Elastic Kubernetes Service)** | Managed Kubernetes (upstream K8s) | Managed by AWS ($0.10/hr per cluster) |
| **Fargate** | **Serverless compute engine** — NOT an orchestrator. It's a launch type for ECS and EKS | N/A (used with ECS or EKS) |

**Key distinction:** ECS and EKS are **orchestrators** (they decide where/how to run containers). Fargate and EC2 are **launch types** (they provide the compute). You choose an orchestrator AND a launch type.

```
Orchestrator:   ECS  ─── or ───  EKS
                 │                  │
Launch Type:  EC2 | Fargate      EC2 | Fargate
```

---

### Amazon ECS (Elastic Container Service)

AWS-proprietary container orchestration service. Simpler than Kubernetes, deeply integrated with AWS services.

#### Core Components

| Component | Description |
|-----------|-------------|
| **Cluster** | Logical grouping of tasks/services and the infrastructure they run on |
| **Task Definition** | Blueprint for your application — like a Dockerfile + run config. Specifies containers, CPU/memory, networking mode, volumes, IAM roles, logging |
| **Task** | Running instance of a task definition. One task = one or more containers running together |
| **Service** | Maintains a desired count of tasks. Handles scaling, load balancing, rolling deployments |
| **Container Instance** | EC2 instance registered to an ECS cluster (EC2 launch type only) |
| **Capacity Provider** | Manages the infrastructure (EC2 Auto Scaling group or Fargate). Supports **Capacity Provider Strategies** for mixed EC2/Fargate |
| **ECS Agent** | Software on each EC2 container instance that communicates with the ECS control plane |
| **Service Connect** | Service mesh using AWS Cloud Map for service-to-service communication |
| **ECS Exec** | Interactive shell into a running container (uses SSM Session Manager) |

#### ECS Networking Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| **awsvpc** | Each task gets its **own ENI** with a private IP in the VPC. **Required for Fargate.** | Default for Fargate; recommended for EC2 launch type. Enables security groups per task |
| **bridge** | Docker's default bridge network on the container instance. Dynamic port mapping with ALB | Legacy EC2 workloads |
| **host** | Container uses the host's network namespace directly. No port mapping — container binds directly to host port | High-performance EC2 workloads needing bare-metal networking |
| **none** | No external connectivity | Batch jobs with no network needs |

**Exam critical:** `awsvpc` mode is the only mode that supports **task-level security groups** and is **mandatory for Fargate**.

---

### Amazon EKS (Elastic Kubernetes Service)

Managed Kubernetes service running upstream, conformant Kubernetes. Choose EKS when you need Kubernetes API compatibility, existing K8s tooling, or multi-cloud portability.

#### Core Components

| Component | Description |
|-----------|-------------|
| **Cluster** | Managed K8s control plane (API server, etcd) across 3 AZs. You don't manage control plane nodes |
| **Node Group (Managed)** | AWS manages EC2 instances as K8s worker nodes. Handles AMI updates, draining, patching |
| **Self-Managed Nodes** | You manage EC2 instances yourself and register them with the cluster |
| **Fargate Profile** | Defines which K8s pods run on Fargate (by namespace and labels). No node management |
| **Pod** | Smallest K8s unit — one or more containers sharing network/storage |
| **Service** | K8s service for exposing pods (ClusterIP, NodePort, LoadBalancer) |
| **Ingress** | L7 routing rules, typically backed by AWS ALB Ingress Controller (now AWS Load Balancer Controller) |
| **EKS Add-ons** | Managed operational software: VPC CNI, CoreDNS, kube-proxy, EBS CSI driver, etc. |
| **EKS Anywhere** | Run EKS on your own on-premises infrastructure |
| **EKS Distro** | Same K8s distribution used by EKS, self-managed anywhere |

#### EKS Networking

- Uses the **Amazon VPC CNI plugin** — each pod gets a **real VPC IP address** from the subnet (similar to ECS `awsvpc` mode).
- Pods can communicate directly with other VPC resources using VPC IPs — no overlay network.
- Limitation: Number of pods per node is limited by the number of **ENIs and secondary IPs** the instance type supports.
- **Security groups for pods**: Supported on Nitro instances — assign VPC security groups directly to individual pods.
- **AWS Load Balancer Controller**: Provisions ALBs for K8s Ingress and NLBs for K8s Service type LoadBalancer.

---

### AWS Fargate

**Serverless compute engine for containers.** You define CPU/memory per task (ECS) or pod (EKS), and Fargate provisions and manages the underlying infrastructure.

#### Key Characteristics

- **No EC2 instances to manage** — no patching, no AMIs, no capacity planning.
- Each Fargate task/pod runs in its **own isolated microVM** (Firecracker) — kernel-level isolation between tasks.
- Always uses **awsvpc** networking — each task gets its own ENI.
- Supports **ECS** and **EKS** (Fargate profiles).
- **Ephemeral storage**: 20 GB by default (ECS can configure up to **200 GB**). Shared among all containers in the task.
- **No persistent storage attached directly** — use EFS for persistent shared storage. **EBS is NOT supported on Fargate** (supported on EC2 launch type only).
- **No GPU support** on Fargate. Must use EC2 launch type for GPU workloads.
- **No privileged containers** on Fargate (no `--privileged` flag, no host PID/network namespace).
- Cannot run DaemonSets on EKS Fargate.

#### Fargate Platform Versions (ECS)

- Platform version defines the runtime environment (kernel, container runtime, etc.).
- **Linux**: Latest is `1.4.0` — supports EFS, ephemeral storage config, SYS_PTRACE, container dependencies.
- **Windows**: Latest is `1.0.0`.
- Specify platform version in the task definition or service config.

---

### Head-to-Head Comparison

| Feature | ECS on EC2 | ECS on Fargate | EKS on EC2 | EKS on Fargate |
|---------|-----------|---------------|-----------|---------------|
| **Control plane cost** | Free | Free | $0.10/hr ($73/mo) | $0.10/hr ($73/mo) |
| **Compute billing** | EC2 instance pricing | Per-task (vCPU + memory + storage) | EC2 instance pricing | Per-pod (vCPU + memory) |
| **OS patching** | You manage | AWS manages | You manage (managed node groups help) | AWS manages |
| **GPU support** | Yes | **No** | Yes | **No** |
| **EBS volumes** | Yes | **No** (use EFS) | Yes (EBS CSI driver) | **No** (use EFS) |
| **DaemonSets** | N/A (ECS concept: daemon scheduling strategy) | **No** | Yes | **No** |
| **Privileged containers** | Yes | **No** | Yes | **No** |
| **Max pods/tasks per node** | Depends on instance (ENIs × IPs) | N/A (one task = one microVM) | Depends on instance (ENIs × IPs) | N/A |
| **SSH into host** | Yes | **No** (use ECS Exec) | Yes | **No** |
| **Networking** | awsvpc, bridge, host, none | awsvpc only | VPC CNI (pod-level IPs) | VPC CNI |
| **Security group scope** | Per task (awsvpc) or per instance | Per task | Per pod (Nitro) or per node | Per pod |
| **K8s ecosystem/tools** | No | No | Yes | Yes |
| **Spot support** | EC2 Spot Instances | **Fargate Spot** (ECS only, up to 70% off) | EC2 Spot Instances | **No Fargate Spot for EKS** |
| **Windows containers** | Yes | Yes (limited) | Yes | **No** |

---

### Default Limits, Quotas, and Constraints

| Resource | Limit | Adjustable? |
|----------|-------|-------------|
| ECS clusters per Region | 10,000 | Yes |
| ECS services per cluster | 5,000 | Yes |
| Tasks per service | 5,000 | Yes |
| Containers per task definition | **10** | No |
| ECS task definition max CPU (Fargate) | **16 vCPU** | No |
| ECS task definition max memory (Fargate) | **120 GB** | No |
| ECS Fargate ephemeral storage | 20 GB default, **200 GB max** | No |
| EKS clusters per Region | 100 | Yes |
| EKS managed node groups per cluster | 30 | Yes |
| EKS Fargate profiles per cluster | 10 | Yes |
| EKS pods per Fargate profile selectors | 5 | No |
| EKS control plane cost | **$0.10/hour per cluster** | No |
| Fargate task/pod startup time | ~30–60 seconds typical | N/A |
| Fargate Spot (ECS only) | Capacity-dependent, can be interrupted with 2-min notice | N/A |

---

## 2. Design Patterns & Best Practices

### When to Use Which — The Decision Matrix

| Scenario | Recommended Choice | Why |
|----------|-------------------|-----|
| AWS-native, simple container orchestration | **ECS + Fargate** | Least operational overhead, deep AWS integration, no control plane cost |
| Need Kubernetes API, existing K8s expertise/tooling | **EKS + Managed Node Groups** | Upstream K8s conformance, portability, ecosystem |
| Multi-cloud / hybrid K8s strategy | **EKS** (or EKS Anywhere for on-prem) | K8s provides portability across clouds |
| GPU workloads (ML inference, video processing) | **ECS on EC2** or **EKS on EC2** | Fargate doesn't support GPUs |
| Unpredictable/bursty workloads, minimize ops | **ECS + Fargate** | No capacity management, pay per task |
| Steady-state high utilization | **ECS/EKS on EC2** with RIs or Savings Plans | More cost-effective than Fargate at sustained high utilization |
| Privileged containers / DaemonSets / host networking | **EC2 launch type** (ECS or EKS) | Fargate doesn't support these |
| Persistent block storage per container | **EC2 launch type + EBS** | Fargate doesn't support EBS |
| Shared persistent file storage | **Any + EFS** | EFS works with both EC2 and Fargate |
| Small team, no K8s expertise | **ECS** | Simpler learning curve than K8s |
| Migration from on-prem K8s | **EKS** | Compatible APIs and tooling |
| Windows containers on Fargate | **ECS on Fargate** | EKS Fargate does NOT support Windows |
| Cost-sensitive with interruptible workloads | **ECS + Fargate Spot** | Up to 70% savings. Fargate Spot NOT available on EKS |

### Anti-Patterns

| Anti-Pattern | Why It's Wrong | Correct Approach |
|-------------|---------------|------------------|
| EKS for simple 3-container microservices, team has no K8s experience | Unnecessary complexity and $73/mo control plane cost | ECS + Fargate |
| Fargate for GPU-based ML inference | Fargate has no GPU support | ECS/EKS on EC2 with P/G instances |
| Fargate for containers needing privileged mode | Not supported | EC2 launch type |
| ECS for multi-cloud portability | ECS is AWS-proprietary | EKS (Kubernetes is portable) |
| Running one task per large EC2 instance on ECS | Wasted capacity | Right-size with Fargate or use bin-packing (spread/binpack placement strategies) |
| EKS Fargate for DaemonSet-based monitoring agents | DaemonSets don't run on Fargate | Use sidecar containers or EKS on EC2 |
| Using Fargate when you need EBS persistent volumes | EBS not supported on Fargate | EC2 launch type with EBS CSI driver |

### Common Architectural Patterns

#### Microservices on ECS
```
ALB → ECS Service A (Fargate) → DynamoDB
    → ECS Service B (Fargate) → RDS (private subnet)
    → ECS Service C (Fargate) → SQS → ECS Service D
Service Discovery: AWS Cloud Map or ECS Service Connect
```

#### Hybrid ECS: Fargate + EC2
```
ECS Cluster with Capacity Provider Strategy:
  - Fargate: base=2, weight=1 (guaranteed capacity)
  - Fargate Spot: base=0, weight=3 (cost savings for interruptible tasks)
  - EC2 ASG: base=0, weight=1 (for GPU tasks or specialized needs)
```

#### EKS Multi-Region Active-Active
```
Route 53 (latency routing)
  → Region A: EKS Cluster → ALB (via AWS LB Controller) → Pods → Aurora Global DB
  → Region B: EKS Cluster → ALB (via AWS LB Controller) → Pods → Aurora Global DB
GitOps: ArgoCD / Flux syncs manifests to both clusters
```

#### Migration: On-Prem K8s → EKS
```
Phase 1: EKS Anywhere on-prem (learn the tooling)
Phase 2: EKS in AWS (lift-and-shift with minimal changes)
Phase 3: Fargate profiles for stateless workloads (reduce ops)
```

#### Multi-Account ECS with Shared ECR
```
Shared Services Account: ECR repositories (cross-account access via resource policy)
Workload Account A: ECS Cluster → pulls images from shared ECR
Workload Account B: ECS Cluster → pulls images from shared ECR
```

### Well-Architected Framework Alignment

**Reliability**
- Use Fargate for automatic multi-AZ task placement (tasks placed across AZs automatically).
- ECS/EKS on EC2: spread tasks across AZs using placement strategies and constraints.
- Use ECS service minimum healthy percent and maximum percent for rolling deployments.
- EKS: Use Pod Disruption Budgets (PDBs) to maintain availability during node drains.
- Design containers as stateless — store state in DynamoDB, ElastiCache, or S3.

**Security**
- Use `awsvpc` mode (mandatory on Fargate) for task-level security groups.
- Use separate IAM Task Roles per ECS task / IAM Roles for Service Accounts (IRSA) per EKS pod.
- Use ECR with image scanning, immutable tags, and lifecycle policies.
- Use Secrets Manager / SSM Parameter Store for secrets (NOT environment variables in task definitions).
- Fargate provides kernel-level isolation (Firecracker) — stronger isolation than shared EC2 hosts.

**Performance Efficiency**
- Right-size Fargate tasks (CPU + memory). Don't allocate 4 vCPU for a task using 0.5 vCPU.
- Use EC2 launch type with placement strategies (`binpack` for cost, `spread` for HA).
- Use ECS Service Connect or App Mesh for low-latency service-to-service communication.
- Use **x86_64 or ARM64 (Graviton)** — Fargate supports both; ARM is cheaper and faster for most workloads.

**Cost Optimization**
- Use Fargate Spot (ECS) for fault-tolerant tasks — up to 70% savings.
- Use EC2 Spot Instances with ECS/EKS capacity providers for interruptible workloads.
- Use Compute Savings Plans — apply to Fargate AND EC2 (both ECS and EKS).
- Right-size: monitor actual CPU/memory usage vs. allocated.

**Operational Excellence**
- Use ECS Exec / `kubectl exec` for live debugging.
- Use AWS Copilot CLI for ECS application lifecycle management.
- Use EKS managed add-ons for consistent operational software.
- Implement GitOps (ArgoCD/Flux) for EKS manifest management.
- Use ECR lifecycle policies to clean up old images.

---

## 3. Security & Compliance

### IAM Patterns

#### ECS IAM Roles

| Role | Purpose |
|------|---------|
| **Task Execution Role** | Used by the ECS agent to pull images from ECR, fetch secrets from Secrets Manager/SSM, write logs to CloudWatch. This is the "infrastructure" role |
| **Task Role** | Used by the application container itself to call AWS APIs (DynamoDB, S3, SQS, etc.). This is the "application" role |
| **ECS Instance Role** | (EC2 launch type only) Role assigned to the EC2 container instance — lets the ECS agent communicate with the ECS control plane |
| **Service-Linked Role** | `AWSServiceRoleForECS` — used by ECS to manage ENIs, load balancer registration, etc. |

**Exam critical:** Task Execution Role ≠ Task Role. Execution Role is for ECS infrastructure operations. Task Role is for your application's AWS API calls. Questions often mix these up.

#### EKS IAM

| Mechanism | Description |
|-----------|-------------|
| **IRSA (IAM Roles for Service Accounts)** | Maps K8s service accounts to IAM roles. Pods assume the IAM role via OIDC federation. **Best practice — most granular** |
| **EKS Pod Identity** | Newer, simpler alternative to IRSA. Associates IAM roles with K8s service accounts without managing OIDC providers. AWS recommended going forward |
| **Node IAM Role** | All pods on a node share the node's IAM role. **NOT recommended** — no pod-level isolation |
| **Cluster Role** | IAM role for the EKS control plane (used during cluster creation) |
| **aws-auth ConfigMap** | Maps IAM users/roles to K8s RBAC. Being replaced by **EKS Access Entries** (API-based) |

#### SCPs for Container Security

```json
{
  "Effect": "Deny",
  "Action": "ecs:CreateService",
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "ecs:cluster": "arn:aws:ecs:us-east-1:123456789012:cluster/approved-cluster"
    }
  }
}
```

- Restrict ECS/EKS operations to approved Regions.
- Deny creation of ECS tasks without specific task roles.
- Deny pulling images from non-approved ECR repositories.

### Encryption

| What | ECS | EKS |
|------|-----|-----|
| Data at rest (EBS) | KMS encryption on EC2 launch type | KMS encryption via EBS CSI driver |
| Data at rest (Fargate ephemeral) | Encrypted by default (AWS-managed or CMK) | Encrypted by default |
| Data at rest (EFS) | KMS encryption | KMS encryption |
| Secrets | Secrets Manager / SSM (referenced in task def) | K8s Secrets (optionally encrypted with KMS envelope encryption) + Secrets Manager via CSI driver |
| ECR images | Encrypted at rest by default (KMS) | Same |
| In transit | TLS between services; Service Connect / App Mesh for mTLS | Service mesh (Istio, App Mesh, Linkerd) for mTLS |
| EKS K8s Secrets | N/A | **Envelope encryption with KMS** — must be explicitly enabled. Without it, etcd stores secrets base64-encoded (NOT encrypted) |

### Logging and Monitoring

| Tool | ECS | EKS |
|------|-----|-----|
| **Container logs** | FireLens (Fluent Bit/Fluentd sidecar) or `awslogs` driver → CloudWatch Logs | Fluent Bit DaemonSet → CloudWatch / OpenSearch / S3 |
| **Metrics** | CloudWatch Container Insights | CloudWatch Container Insights + Prometheus |
| **Tracing** | X-Ray sidecar or ADOT (AWS Distro for OpenTelemetry) | X-Ray / ADOT |
| **Audit logs** | CloudTrail (API calls) | CloudTrail + **EKS control plane logging** (API server, authenticator, controller manager, scheduler, audit) |
| **Live debugging** | ECS Exec (SSM) | kubectl exec |

### Compliance Patterns

- **Image security**: ECR image scanning (basic with Clair, enhanced with Amazon Inspector). Use immutable tags to prevent image tampering.
- **Runtime security**: AWS GuardDuty for EKS (detects K8s audit log anomalies and runtime threats). For ECS, use third-party tools or GuardDuty with ECS Runtime Monitoring.
- **Network policy**: EKS supports K8s Network Policies (via Calico or VPC CNI network policy). ECS uses security groups per task.
- **PCI/HIPAA**: Both ECS and EKS are PCI and HIPAA eligible. Use encryption, logging, access controls as required by the standard.

---

## 4. Cost Optimization

### Pricing Models

#### ECS Pricing
- **Control plane**: **Free** (no charge for the ECS orchestrator).
- **EC2 launch type**: Pay for EC2 instances (On-Demand, RI, Savings Plans, Spot).
- **Fargate launch type**: Pay per task based on vCPU and memory per second (1-minute minimum).

#### EKS Pricing
- **Control plane**: **$0.10/hour per cluster** (~$73/month). This is a fixed cost regardless of workload size.
- **EC2 launch type**: Same as ECS — pay for EC2 instances.
- **Fargate launch type**: Same per-task pricing as ECS Fargate.
- **EKS on Fargate Pod cost** = Fargate pricing + EKS cluster fee.

#### Fargate Pricing (us-east-1, Linux x86)

| Resource | Price |
|----------|-------|
| vCPU per second | $0.04048/hr ($0.00001124/s) |
| Memory per GB per second | $0.004445/hr ($0.00000123/s) |
| Ephemeral storage (above 20 GB) | $0.000111/GB/hr |
| **Fargate Spot** (ECS only) | Up to **70% discount** vs On-Demand |
| **ARM64 (Graviton)** | **20% cheaper** than x86 |

#### Compute Savings Plans
- Apply to **Fargate** (ECS and EKS) AND **EC2 instances** (ECS and EKS).
- 1-year or 3-year commitment to $/hr of compute usage.
- Up to **66% savings** on Fargate, up to **66% on EC2**.
- Most flexible savings option across container services.

### Cost-Saving Strategies

1. **Fargate Spot for ECS** — up to 70% off for fault-tolerant tasks (batch processing, queue workers). NOT available for EKS Fargate.
2. **ARM64 / Graviton** — 20% cheaper, often better performance. Supported on both EC2 and Fargate.
3. **Compute Savings Plans** — covers both EC2 and Fargate across ECS and EKS. Preferred over EC2 Instance Savings Plans for container flexibility.
4. **Right-size tasks/pods** — use Container Insights to identify over-provisioned CPU/memory.
5. **EC2 Spot with ECS Capacity Providers** — mixed On-Demand + Spot with automatic draining on interruption.
6. **Binpack placement strategy (ECS on EC2)** — place tasks to maximize utilization per instance, reducing the number of instances needed.
7. **ECR lifecycle policies** — auto-delete old images to reduce storage costs.
8. **Consolidate EKS clusters** — each cluster costs $73/month. Use namespaces for isolation instead of separate clusters when possible.
9. **Karpenter (EKS)** — right-sizes and provisions nodes dynamically based on pod requirements. More efficient than Cluster Autoscaler.

### Common Cost Traps

- **EKS cluster fee ($73/mo) per cluster** — significant for small workloads. Multiple environments (dev/staging/prod) × multiple teams = many clusters = high fixed cost.
- **Over-provisioned Fargate tasks** — allocating 4 vCPU / 8 GB when the container uses 0.25 vCPU / 512 MB.
- **Forgotten idle ECS services** — service maintains desired count even if no traffic. Use Application Auto Scaling to scale to zero (possible with Fargate).
- **CloudWatch Logs costs** — container logs can be massive. Set retention policies and consider filtering before shipping.
- **ECR storage** — old images accumulate. Use lifecycle policies.
- **Fargate vs EC2 break-even**: For sustained high utilization (>50%), EC2 with Savings Plans is typically cheaper than Fargate. Fargate wins for bursty/unpredictable workloads.

### Cost Comparison Table

| Scenario | Cheapest Option | Why |
|----------|----------------|-----|
| Bursty, unpredictable traffic | Fargate | No idle capacity cost |
| Fault-tolerant batch jobs | Fargate Spot (ECS) or EC2 Spot | 70%+ savings |
| Steady-state, high utilization | EC2 with Compute Savings Plans | Lower unit cost at scale |
| Small team, few services | ECS + Fargate | No EKS cluster fee, no EC2 management |
| Multiple K8s workloads | EKS + EC2 (managed node groups) | Cluster fee amortized across many pods; EC2 cheaper at scale |
| GPU workloads | EC2 (P/G instances) | Fargate doesn't support GPUs |

---

## 5. High Availability, Disaster Recovery & Resilience

### HA Architecture

#### ECS HA
- ECS services **automatically distribute tasks across AZs** when using Fargate or EC2 instances in multiple AZs.
- Use ALB/NLB for load balancing across tasks.
- Configure **minimum healthy percent** (e.g., 100%) and **maximum percent** (e.g., 200%) for rolling deployments.
- Use **Circuit Breaker** (ECS deployment circuit breaker) to automatically roll back failed deployments.
- **Capacity Provider Strategies** can mix Fargate + Fargate Spot + EC2 for resilience.

#### EKS HA
- EKS control plane runs across **3 AZs** automatically (managed by AWS).
- Managed node groups: place nodes in multiple AZs via subnets.
- Use **Pod Disruption Budgets** to prevent all replicas from being terminated during node maintenance.
- Use **Pod Topology Spread Constraints** to distribute pods across AZs.
- Use **Karpenter** or **Cluster Autoscaler** for automatic node scaling.
- Use K8s **readiness/liveness probes** for health checking.

#### Fargate HA
- Fargate tasks are placed across AZs automatically when subnets span multiple AZs.
- No instance management — if underlying hardware fails, Fargate replaces the task automatically.
- Fargate Spot: tasks can be interrupted — use for fault-tolerant workloads only. ECS receives a **2-minute SIGTERM warning** before termination.

### Multi-Region Patterns

| Pattern | ECS | EKS |
|---------|-----|-----|
| **Active-Active** | Deploy ECS services in both Regions behind Route 53 / Global Accelerator + shared data store (DynamoDB Global Tables, Aurora Global DB) | Deploy EKS clusters in both Regions, use GitOps to sync manifests, Route 53 / Global Accelerator for traffic |
| **Active-Passive** | Primary Region active, DR Region with ECS services scaled to zero or minimum. Route 53 failover | Primary EKS cluster active, DR cluster with minimal replicas. Route 53 failover |
| **Pilot Light** | ECR cross-Region replication. DR Region has cluster and task definitions ready, no running tasks. Scale up on failover | ECR cross-Region replication. DR cluster ready, deploy workloads on failover |

### Key Resilience Features

| Feature | Description |
|---------|-------------|
| **ECS Deployment Circuit Breaker** | Automatically rolls back a deployment if new tasks fail to stabilize |
| **ECS Service Auto Scaling** | Target tracking, step scaling, or scheduled scaling on CloudWatch metrics (CPU, memory, ALB request count) |
| **EKS Horizontal Pod Autoscaler (HPA)** | Scale pod replicas based on CPU/memory or custom metrics |
| **EKS Karpenter** | Provision right-sized nodes in seconds based on pending pod requirements |
| **ECR Cross-Region Replication** | Replicate container images to DR Region |
| **EFS for persistent storage** | Multi-AZ by default, works with Fargate — no data loss on task replacement |
| **Fargate task replacement** | If a Fargate task fails, ECS/EKS automatically launches a replacement |

### Backup and Recovery

- **Stateless containers + external state**: The primary DR strategy. Containers are disposable; state lives in managed services (DynamoDB, RDS, S3).
- **ECR replication**: Enable cross-Region and/or cross-account replication for container images.
- **Task definitions / K8s manifests in version control**: Infrastructure as code (CloudFormation, CDK, Terraform, Helm charts) enables rapid redeployment.
- **No backup of Fargate ephemeral storage**: It's gone when the task stops. Design accordingly — use EFS or S3 for durable data.

---

## 6. Exam-Focused Section

### Straightforward Questions (7 examples)

**Q1:** A company wants to run containers on AWS with the least operational overhead and no EC2 management. Which combination should they use?
**A:** **ECS + Fargate.** No cluster fee (unlike EKS), no EC2 instances to manage, serverless compute.

**Q2:** What is the maximum ephemeral storage available on an ECS Fargate task?
**A:** **200 GB.** Default is 20 GB, configurable up to 200 GB.

**Q3:** A company needs to run Kubernetes on AWS with full upstream K8s API compatibility. Which service should they use?
**A:** **Amazon EKS.** It runs conformant, upstream Kubernetes.

**Q4:** Which ECS networking mode is required for Fargate?
**A:** **awsvpc.** Each Fargate task gets its own ENI with a VPC IP address.

**Q5:** How does ECS on Fargate provide task-level security isolation?
**A:** Each Fargate task runs in its own **Firecracker microVM**, providing kernel-level isolation between tasks.

**Q6:** What is the cost of the ECS control plane?
**A:** **Free.** There is no charge for the ECS control plane. (EKS charges $0.10/hr.)

**Q7:** A company needs to mount shared, persistent file storage across multiple Fargate tasks. What storage service should they use?
**A:** **Amazon EFS.** EBS is not supported on Fargate. EFS provides shared, persistent, multi-AZ storage compatible with Fargate.

---

### Tricky / Scenario-Based Questions (7 examples)

**Q1:** A startup with a small DevOps team (no Kubernetes experience) needs to deploy 10 microservices. Traffic is bursty and unpredictable. They want minimum operational overhead and cost. An architect proposes EKS with Fargate profiles. Is this the best choice?

**A:** **No. ECS + Fargate is better.**
- **Why not EKS?** EKS has a $73/month control plane fee. The team has no K8s experience — learning curve adds operational overhead. EKS Fargate doesn't support Fargate Spot (no cost savings for interruptible workloads).
- **Why ECS?** Free control plane, simpler API, deep AWS integration, Fargate Spot available for cost savings, lower learning curve.
- **Key phrases:** "small team" + "no Kubernetes experience" + "minimum overhead and cost" → ECS + Fargate.

**Q2:** A company runs a machine learning inference workload that requires GPU access. They want to use containers with minimal management overhead. A team member suggests Fargate. What is the correct approach?

**A:** **Fargate does NOT support GPUs.** Use **ECS on EC2** with GPU-enabled instances (P4, G5, etc.) or **EKS on EC2** with managed node groups containing GPU instances and the NVIDIA device plugin.
- **Why not Fargate?** No GPU support is a hard limitation. No workaround.
- **Key phrases:** "GPU" + "containers" → EC2 launch type (never Fargate).

**Q3:** A company has an ECS cluster with EC2 launch type. They want each task to have its own IAM permissions and VPC security group. Currently, tasks share the EC2 instance's security group. What should they change?

**A:** Switch the networking mode from **bridge** to **awsvpc**.
- **Why `awsvpc`?** Each task gets its own ENI, enabling task-level security groups and task-level IAM roles (Task Role).
- **Why not just change IAM roles?** Task Roles work in any networking mode, but task-level security groups require `awsvpc`.
- **Why not switch to Fargate?** That would also work (Fargate always uses `awsvpc`), but the question says they want to keep EC2 launch type. Changing the networking mode is sufficient.
- **Key phrases:** "own security group per task" → `awsvpc` networking mode.

**Q4:** A company runs EKS with managed node groups. They want to ensure that when a node is drained for maintenance, at least 3 of their 5 replicas remain available. How should they configure this?

**A:** Create a **Pod Disruption Budget (PDB)** with `minAvailable: 3` (or `maxUnavailable: 2`).
- **Why not just increase replicas?** More replicas doesn't prevent all of them from being on the same node or being drained simultaneously. PDB is the K8s mechanism to protect availability during voluntary disruptions.
- **Why not a Deployment strategy?** Deployment strategies control rolling updates, not node drain behavior.
- **Key phrases:** "node drain/maintenance" + "minimum replicas available" → Pod Disruption Budget.

**Q5:** A company uses ECS on Fargate and needs persistent block storage for a database container. They want to use EBS io2 volumes. Can they do this?

**A:** **No. Fargate does not support EBS volumes.** 
- **Options:** (1) Switch to **EC2 launch type** and attach EBS volumes. (2) Use **Amazon EFS** on Fargate (but EFS is a file system, not block storage — may not suit all database needs). (3) Use a **managed database service** (RDS, Aurora) instead of running a database in a container.
- **Exam tip:** The best answer is usually "use a managed database" — running databases in containers is an anti-pattern unless you have specific expertise.
- **Key phrases:** "Fargate" + "EBS" → Not supported. Think EFS or managed database.

**Q6:** A company has ECS services and wants to implement a blue/green deployment strategy. New tasks should receive a portion of traffic for testing before full cutover. Which AWS service should they use?

**A:** **AWS CodeDeploy** with **ECS blue/green deployment**.
- CodeDeploy creates a new (green) task set, shifts traffic via ALB listener rules (canary: 10% then 90%, linear: incremental, all-at-once).
- **Why not ECS rolling update?** Rolling update replaces tasks gradually but doesn't support traffic-weighted canary testing — all new tasks immediately receive traffic.
- **Why not Route 53 weighted routing?** That routes at the DNS level between different endpoints. CodeDeploy with ALB target groups gives precise, instant traffic shifting within the same service.
- **Key phrases:** "blue/green" + "ECS" + "portion of traffic" → CodeDeploy.

**Q7:** A company runs an EKS cluster and uses the `aws-auth` ConfigMap to grant an IAM role access to the cluster. When the ConfigMap is accidentally deleted, all IAM-authenticated users lose access. Only the cluster creator can access the cluster. How should they prevent this in the future?

**A:** Migrate from `aws-auth` ConfigMap to **EKS Access Entries** (API-based access management).
- **Why?** Access Entries are managed via the EKS API (not a fragile ConfigMap inside the cluster). They survive ConfigMap deletion and can be managed with IaC tools.
- **Why not just back up the ConfigMap?** That's a workaround, not a fix. The underlying fragility remains.
- **Key phrases:** "aws-auth ConfigMap" + "accidentally deleted" + "locked out" → EKS Access Entries.

---

### Common Exam Traps & Pitfalls

1. **ECS control plane is free; EKS costs $0.10/hr**: If the question mentions "minimize cost" and there's no K8s requirement, ECS is preferred.

2. **Fargate Spot is ECS only, NOT available on EKS Fargate**: If the question mentions Fargate cost savings on EKS, Fargate Spot is not an option. Use EC2 Spot nodes instead.

3. **Task Execution Role vs Task Role (ECS)**: Execution Role = pull images, get secrets, write logs (infrastructure). Task Role = application AWS API calls. Questions deliberately confuse these.

4. **EBS is NOT supported on Fargate**: Only EFS and ephemeral storage. If the question needs persistent block storage on Fargate, it's a trap — the answer is EC2 launch type or a managed service.

5. **No DaemonSets on Fargate (EKS)**: Use sidecar containers instead. If the question mentions Fargate + DaemonSets, it's wrong.

6. **No privileged containers on Fargate**: If the question requires `--privileged`, the answer is EC2 launch type.

7. **IRSA/Pod Identity vs Node IAM Role (EKS)**: Node IAM Role gives all pods on the node the same permissions — exam expects you to know this is bad practice. IRSA or EKS Pod Identity provides pod-level IAM.

8. **EKS Secrets in etcd are NOT encrypted by default**: Must explicitly enable KMS envelope encryption. Candidates assume managed service = encrypted. Not for K8s secrets.

9. **`awsvpc` mode is the only mode with per-task security groups**: Bridge and host modes share the instance's security group. If the question mentions task-level network isolation, the answer requires `awsvpc`.

10. **Fargate startup time (~30-60s) vs EC2 warm instances**: For latency-sensitive scaling, EC2 with pre-warmed instances (or excess capacity) is faster than Fargate cold start.

11. **Service Connect vs App Mesh**: Service Connect is simpler (built into ECS). App Mesh is more powerful but more complex. For simple ECS service discovery, Service Connect or Cloud Map is the answer. App Mesh is for advanced traffic management.

12. **Windows containers on Fargate**: Supported on ECS Fargate, NOT on EKS Fargate.

---

## 7. Cheat Sheet

### Must-Know Facts for Exam Day

| Fact | Value |
|------|-------|
| ECS control plane cost | **Free** |
| EKS control plane cost | **$0.10/hr (~$73/month)** per cluster |
| EKS control plane AZs | **3 AZs** (automatic) |
| Fargate max vCPU per task | **16 vCPU** |
| Fargate max memory per task | **120 GB** |
| Fargate max ephemeral storage | **200 GB** |
| Fargate networking mode | **awsvpc only** |
| Fargate GPU support | **No** |
| Fargate EBS support | **No** (use EFS) |
| Fargate privileged containers | **No** |
| Fargate DaemonSets (EKS) | **No** |
| Fargate Spot availability | **ECS only** (not EKS) |
| Fargate Spot savings | Up to **70%** |
| Fargate ARM64 savings | **20%** cheaper |
| Containers per ECS task definition | **10 max** |
| ECS deployment types | Rolling update, Blue/Green (CodeDeploy), External |
| EKS pod-level IAM | **IRSA** or **EKS Pod Identity** |
| ECS task-level IAM | **Task Role** (app) + **Task Execution Role** (infra) |
| EKS Secrets encryption | **Not default** — must enable KMS envelope encryption |
| Compute Savings Plans | Apply to **both Fargate and EC2** |

### Key Differentiators

| ECS | EKS |
|-----|-----|
| AWS-proprietary | Open-source Kubernetes |
| Free control plane | $0.10/hr per cluster |
| Simpler | Full K8s ecosystem |
| AWS-only | Multi-cloud portable |
| Task definitions (JSON) | K8s manifests (YAML) |
| Service Connect / Cloud Map | K8s Services / Ingress |
| CodeDeploy blue/green | K8s native + ArgoCD/Flux |
| Fargate Spot supported | Fargate Spot NOT supported |

| EC2 Launch Type | Fargate Launch Type |
|----------------|-------------------|
| You manage instances | Serverless |
| GPU support | No GPU |
| EBS support | No EBS (EFS only) |
| Privileged containers | No privileged |
| DaemonSets (EKS) | No DaemonSets |
| SSH access to host | No SSH (use ECS Exec) |
| Cheaper at scale (steady-state) | Cheaper for bursty workloads |
| More control | Less ops |

| ECS Task Execution Role | ECS Task Role |
|------------------------|--------------|
| Pull images from ECR | Application API calls |
| Fetch secrets from SM/SSM | DynamoDB, S3, SQS access |
| Write to CloudWatch Logs | Business logic permissions |
| Infrastructure role | Application role |

### Decision Flowchart

```
Question says "containers" + "least operational overhead" + no K8s requirement?
  → ECS + Fargate

Question says "Kubernetes" or "K8s" or "existing K8s tooling" or "multi-cloud"?
  → EKS

Question says "GPU" or "privileged containers" or "DaemonSets"?
  → EC2 launch type (NOT Fargate)

Question says "EBS persistent storage" for containers?
  → EC2 launch type (Fargate only supports EFS)

Question says "shared persistent file storage" on Fargate?
  → Amazon EFS

Question says "bursty traffic" + "minimize cost" + "fault-tolerant"?
  → ECS + Fargate Spot

Question says "steady-state" + "high utilization" + "minimize cost"?
  → EC2 launch type + Compute Savings Plans (or RIs)

Question says "task-level security groups"?
  → awsvpc networking mode (mandatory on Fargate, optional on EC2)

Question says "per-pod IAM on EKS"?
  → IRSA or EKS Pod Identity (NOT node IAM role)

Question says "blue/green deployment" + "ECS"?
  → AWS CodeDeploy with ECS

Question says "container image vulnerability scanning"?
  → ECR image scanning (basic) or Amazon Inspector (enhanced)

Question says "on-premises Kubernetes" + "AWS management"?
  → EKS Anywhere

Question says "minimize EKS cluster costs" + multiple environments?
  → Consolidate into fewer clusters + use namespaces for isolation

Question says "secrets in EKS" + "encrypted at rest"?
  → Enable KMS envelope encryption for K8s Secrets

Question says "Windows containers" + "serverless"?
  → ECS on Fargate (NOT EKS on Fargate — not supported)

Question says "migrate from on-prem K8s to AWS"?
  → EKS (compatible APIs and tooling)

Question says "node drain" + "maintain availability"?
  → Pod Disruption Budget (PDB)
```

---

*Generated for AWS SAP-C02 exam preparation. Last updated: 2026-04-26.*
