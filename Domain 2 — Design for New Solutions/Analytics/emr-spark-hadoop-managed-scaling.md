# AWS SAP-C02 Study Notes: EMR (Spark, Hadoop, Managed Scaling)

---

## 1. Core Concepts & Theory

### What is Amazon EMR?

**Definition:** Amazon Elastic MapReduce (EMR) is a managed big data platform that simplifies running open-source distributed processing frameworks (Apache Spark, Hadoop, Hive, Presto/Trino, HBase, Flink) on AWS at scale.

**Key value proposition:** Run petabyte-scale analytics without managing cluster infrastructure, patching, or tuning Hadoop/Spark configurations yourself.

---

### Deployment Options

| Option | Description | Use Case |
|--------|-------------|----------|
| **EMR on EC2** | Traditional — clusters of EC2 instances running frameworks | Full control, persistent clusters, complex pipelines |
| **EMR on EKS** | Run Spark jobs on Amazon EKS clusters | Share K8s infrastructure, multi-tenant, existing EKS investment |
| **EMR Serverless** | Fully managed, no cluster provisioning | "Least operational overhead," variable workloads, pay-per-use |
| **EMR on Outposts** | EMR on AWS Outposts hardware | On-premises big data with EMR APIs |

---

### EMR on EC2 — Architecture

```
[EMR Cluster]
├── Primary Node (Master)
│   ├── YARN ResourceManager
│   ├── HDFS NameNode
│   ├── Hive Metastore
│   └── Spark Driver (client mode)
├── Core Nodes (1+)
│   ├── YARN NodeManager
│   ├── HDFS DataNode (persistent local storage)
│   └── Task execution
└── Task Nodes (0+)
    ├── YARN NodeManager only
    ├── NO HDFS (no data stored)
    └── Pure compute — can use Spot Instances safely
```

**Node types:**

| Node | Role | HDFS? | Spot Safe? | Scaling |
|------|------|-------|------------|---------|
| **Primary (Master)** | Coordinates cluster, runs drivers | NameNode | No (single point of failure) | 1 or 3 (HA) |
| **Core** | Runs tasks + stores HDFS data | DataNode | Risky (data loss if terminated) | Yes, but carefully |
| **Task** | Runs tasks only, no data | No | Yes (ideal for Spot) | Yes (elastic) |

**Primary Node HA:** Can deploy 3 primary nodes for high availability (multi-master). Requires EMR 5.23+ and specific applications.

---

### Supported Frameworks & Applications

| Framework | Purpose | EMR Version |
|-----------|---------|-------------|
| **Apache Spark** | In-memory distributed processing (batch + streaming) | All recent |
| **Apache Hadoop (MapReduce)** | Disk-based distributed processing (legacy batch) | All |
| **Apache Hive** | SQL-on-Hadoop (data warehouse queries) | All |
| **Presto / Trino** | Distributed SQL query engine (interactive, federated) | All recent |
| **Apache HBase** | NoSQL wide-column store on HDFS | All |
| **Apache Flink** | Real-time stream processing | EMR 5.+ |
| **Apache Hudi** | Data lake table format (upserts, incremental) | EMR 6.+ |
| **Apache Iceberg** | Data lake table format (ACID, time travel) | EMR 6.+ |
| **Delta Lake** | Data lake table format (ACID) | EMR 6.+ |
| **Jupyter / Zeppelin** | Interactive notebooks | All recent |
| **Ganglia** | Cluster monitoring | All |
| **TensorFlow / MXNet** | ML training on cluster | EMR 5.+ |

---

### EMR on EKS

**What it is:** Submit Spark jobs to an Amazon EKS cluster using EMR's APIs. Spark runs in Kubernetes pods instead of dedicated EC2 instances.

**Architecture:**
```
[EMR Virtual Cluster] → [EKS Cluster (shared)]
                              ├── Namespace: emr-spark-jobs
                              │     ├── Spark Driver Pod
                              │     └── Spark Executor Pods (auto-scaled)
                              └── Namespace: other-workloads
```

**Key benefits:**
- Share EKS cluster across Spark + other workloads (cost efficiency)
- Multi-tenant isolation via K8s namespaces
- Use Karpenter/Cluster Autoscaler for node scaling
- Consolidate operations (one K8s platform for everything)
- Faster pod startup vs. EMR on EC2 cluster launch

**When to use:**
- Already running EKS for other workloads
- Need multi-tenancy with resource isolation
- Want to standardize on Kubernetes
- Need faster job startup (no cluster provisioning delay)

---

### EMR Serverless

**What it is:** Run Spark and Hive jobs without provisioning or managing clusters. Submit jobs → EMR auto-provisions workers → scales up/down → releases resources when done.

**Key characteristics:**
- **No cluster to manage** — submit a job, get results
- **Auto-scaling:** Workers added/removed based on workload (sub-second scaling)
- **Pre-initialized workers:** Optionally keep workers warm for faster cold start
- **Pay per vCPU/memory/storage per second** while workers are active
- **Application-level isolation:** Each application is an isolated runtime

**When to use:**
- "Least operational overhead" for Spark/Hive
- Variable/unpredictable workloads
- Don't want to manage cluster lifecycle
- Job-oriented (submit, process, done)
- Development and ad-hoc analytics

**Limitations:**
- Only Spark and Hive (no HBase, Flink, Presto natively)
- Less control over runtime environment vs. EMR on EC2
- Not suited for long-running interactive workloads (costs accumulate)

---

### Storage Options

| Storage | Description | Performance | Persistence | Cost |
|---------|-------------|-------------|-------------|------|
| **HDFS (local)** | Distributed across core nodes (instance store or EBS) | Highest (data locality) | Lost when cluster terminates (unless persistent) | Included in instance cost |
| **EMRFS (S3)** | S3 as the "file system" for EMR | High (optimized connector) | Persistent (survives cluster termination) | S3 pricing |
| **EBS volumes** | Attached to nodes for HDFS or local temp storage | High (SSD/throughput options) | Lost when cluster terminates (unless using persistent clusters) | EBS pricing |

**EMRFS (S3) vs. HDFS:**

| Factor | EMRFS (S3) | HDFS |
|--------|-----------|------|
| **Persistence** | Data survives cluster termination | Data lost with cluster (unless persistent/EBS-backed) |
| **Scalability** | Unlimited | Limited by core nodes |
| **Cost** | Cheaper for large datasets | Instance storage costs |
| **Performance** | Good (S3 optimized, S3A committer) | Better for high I/O shuffle operations |
| **Separation** | Compute & storage decoupled | Compute & storage coupled |
| **Exam preference** | Default for most scenarios | "Highest performance for iterative workloads" |

**Best practice:** Use **S3 (EMRFS)** as the primary data store. Use **HDFS** only for intermediate shuffle data or workloads requiring extreme I/O performance. This enables **transient clusters** — launch, process, terminate.

**EMRFS Consistent View (legacy):** Previously used DynamoDB for S3 consistency. **No longer needed** — S3 now provides strong read-after-write consistency natively.

---

### Managed Scaling

**What it is:** EMR automatically adds or removes instances (core + task nodes) based on workload metrics (YARN pending containers, HDFS utilization).

**How it works:**
1. Set **minimum** and **maximum** instance counts (or instance fleet units)
2. EMR monitors YARN metrics (pending containers, available memory)
3. Scales out when demand exceeds capacity; scales in when idle
4. Respects HDFS replication factor when scaling in core nodes
5. Task nodes scale more aggressively (no HDFS data to protect)

**Configuration:**
```json
{
  "ComputeLimits": {
    "UnitType": "Instances",  // or "InstanceFleetUnits", "VCPU"
    "MinimumCapacityUnits": 2,
    "MaximumCapacityUnits": 100,
    "MaximumOnDemandCapacityUnits": 20,
    "MaximumCoreCapacityUnits": 50
  }
}
```

**Key parameters:**
- `MinimumCapacityUnits` — floor (never scale below this)
- `MaximumCapacityUnits` — ceiling (never scale above this)
- `MaximumOnDemandCapacityUnits` — cap On-Demand usage (rest uses Spot)
- `MaximumCoreCapacityUnits` — limits core node scaling (protects HDFS)

**Scale-down behavior:**
- Waits for tasks to complete on the node (graceful decommissioning)
- Protects HDFS data (won't scale down core if it would violate replication factor)
- Task nodes scale down first (no data to lose)

---

### Instance Fleets vs. Instance Groups

| Feature | Instance Groups | Instance Fleets |
|---------|----------------|-----------------|
| **Model** | One instance type per group (Master, Core, Task) | Multiple instance types per fleet |
| **Spot handling** | Single type — if unavailable, cluster stuck | Multiple types — falls back to alternatives |
| **Best for** | Simple, predictable workloads | Cost optimization, Spot diversity |
| **Capacity** | Count-based (number of instances) | Unit-based (vCPUs or custom units) |
| **Exam preference** | Simple scenarios | "Cost-optimize" + "Spot" + "availability" |

**Instance Fleets with Allocation Strategy:**
- **Capacity-optimized:** Launches from Spot pools with most available capacity (fewer interruptions)
- **Lowest-price:** Launches cheapest instances (more interruption risk)
- **Diversified:** Spreads across pools (default for fleets)

---

### EMR Studio & Notebooks

**EMR Studio:** Integrated development environment (IDE) for Spark with Jupyter notebooks.
- Attach notebooks to EMR clusters or EMR Serverless
- Collaborative editing
- Git integration
- Managed Jupyter endpoints

---

### Key Limits & Quotas

| Resource | Default Limit |
|----------|---------------|
| Active clusters per Region | 500 |
| Instances per cluster | ~20,000 (varies by type) |
| EC2 instance types per fleet | 30 |
| Bootstrap actions per cluster | 16 |
| Steps per cluster (pending) | 256 |
| EMR Serverless applications per Region | 50 |
| EMR Serverless max workers per application | 3,000 |
| EBS volumes per instance | 25 |

---

## 2. Design Patterns & Best Practices

### When to Use Each EMR Option

| Scenario | Best Option |
|----------|-------------|
| Complex multi-framework pipeline (Spark + Hive + HBase) | EMR on EC2 |
| "Least operational overhead" Spark/Hive | EMR Serverless |
| Already on EKS, want to run Spark | EMR on EKS |
| Interactive exploration with notebooks | EMR on EC2 (or EMR Studio) |
| Long-running HBase cluster | EMR on EC2 (persistent) |
| Spot-heavy cost-optimized batch | EMR on EC2 (Instance Fleets + task nodes) |
| On-premises big data with AWS APIs | EMR on Outposts |
| Ad-hoc queries on S3 data lake | EMR Serverless or Athena (if SQL-only) |
| Real-time streaming with Flink | EMR on EC2 |

### Transient vs. Persistent Clusters

| Type | Description | Use Case |
|------|-------------|----------|
| **Transient** | Launch → process → terminate | Batch ETL, scheduled jobs, cost-sensitive |
| **Persistent** | Long-running, always available | Interactive queries, HBase, notebooks, streaming |

**Best practice:** Default to **transient clusters** with S3 storage. Only use persistent clusters when workloads require it (HBase, interactive exploration, streaming).

### Anti-Patterns

| Anti-Pattern | Why It's Bad | Fix |
|--------------|-------------|-----|
| Using HDFS as primary storage | Data lost on termination; can't use transient clusters | Use S3 (EMRFS) as primary; HDFS for temp/shuffle only |
| Running Spot on primary node | Cluster crashes if primary is interrupted | On-Demand for primary; Spot for task nodes |
| Spot on core nodes without backup | HDFS data loss on interruption | On-Demand or Spot Fleet with HDFS replication 3 |
| Over-sized persistent cluster | Paying for idle capacity 24/7 | Transient clusters or managed scaling |
| Single instance type in Spot fleet | Capacity shortages cause job failures | Instance Fleets with 5-10+ diverse types |
| Not using S3 output for results | Results lost with cluster | Write all final output to S3 |
| Skipping compression | Higher S3 storage + slower shuffles | Use Snappy (Spark default) or ZSTD |

### Architectural Patterns

**ETL Pipeline (Transient):**
```
[S3 Raw Zone] → [EMR Transient Cluster: Spark ETL]
                        ↓ (write output)
              [S3 Curated Zone (Parquet)]
                        ↓
              [Athena / Redshift Spectrum queries]
```
- Triggered by Step Functions or EventBridge schedule
- Cluster auto-terminates after steps complete
- Auto-scaling during processing

**Data Lake Processing:**
```
[S3 Data Lake] ← [Glue Catalog (metadata)]
      ↓
[EMR Cluster: Spark reads/writes S3 via EMRFS]
      ↓
[Iceberg/Hudi tables on S3 (ACID, time travel)]
```

**Real-Time Streaming:**
```
[Kinesis Data Streams] → [EMR (Spark Structured Streaming or Flink)]
                                     ↓
                         [S3 / DynamoDB / OpenSearch (output)]
```

**ML Training at Scale:**
```
[S3 Training Data] → [EMR Spark (feature engineering)]
                            ↓
                    [SageMaker Training Job (model training)]
                            ↓
                    [SageMaker Endpoint (inference)]
```

### Well-Architected Alignment

| Pillar | Guidance |
|--------|----------|
| **Reliability** | Multi-master HA; managed scaling; Instance Fleets for Spot resilience; S3 for data durability |
| **Security** | Kerberos; Lake Formation integration; encryption at rest/in transit; VPC-only clusters; security configurations |
| **Cost** | Transient clusters; Spot for task nodes; managed scaling; right-size instances; Graviton instances |
| **Performance** | Right framework for the job; partitioned data; columnar formats; HDFS for shuffle-heavy; S3 for throughput |
| **Operational Excellence** | Step Functions orchestration; CloudWatch metrics; EMR Serverless for simplicity; bootstrap actions for config |
| **Sustainability** | Graviton (ARM) instances; transient clusters (no idle waste); right-size; Serverless auto-release |

---

## 3. Security & Compliance

### Network Security

- **VPC deployment:** All EMR clusters launch in a VPC (public or private subnet)
- **Private subnet (recommended):** No public IPs; access via bastion, VPN, or Systems Manager
- **Security groups:** EMR creates managed security groups (primary, core/task, service access); can add custom groups
- **EMR-managed security groups:** Automatically configure inter-node communication rules
- **VPC endpoints:** Use PrivateLink for S3, KMS, STS, CloudWatch (avoid internet for service access)
- **Block public access:** Account-level setting to prevent clusters with public IPs

### Encryption

**At rest:**
| Layer | Options |
|-------|---------|
| S3 (EMRFS) | SSE-S3, SSE-KMS, CSE-KMS, CSE-Custom |
| Local disk (HDFS, EBS) | LUKS encryption with KMS or custom key provider |
| EBS volumes | AWS-managed encryption or KMS CMK |

**In transit:**
- **TLS:** Enable in-transit encryption for EMRFS ↔ S3 (always TLS)
- **Spark shuffle encryption:** AES encryption for shuffle data between nodes
- **YARN shuffle:** Encrypted shuffle in MapReduce
- **TLS for internal services:** Spark UI, Hive, Presto ports

**Security Configuration:** A reusable EMR resource that bundles all encryption settings (at-rest, in-transit, KMS keys) into one config applied at cluster launch.

### Authentication & Authorization

| Mechanism | Description |
|-----------|-------------|
| **IAM roles** | EMR service role, EC2 instance profile, auto-scaling role |
| **Kerberos** | Full Kerberos (MIT KDC or cross-realm with Active Directory) for user authentication |
| **Lake Formation** | Fine-grained table/column/row authorization (credential vending to Spark) |
| **Apache Ranger** | Open-source authorization (policies for Hive, Spark, etc.) — self-managed |
| **EMRFS IAM roles** | Map HDFS users to IAM roles for S3 access (per-user S3 permissions) |
| **Runtime roles (EMR on EKS / Serverless)** | Per-job IAM roles for least-privilege |

### IAM Roles in EMR

| Role | Purpose |
|------|---------|
| **EMR Service Role** | Allows EMR service to provision EC2, auto-scale, etc. (e.g., `EMR_DefaultRole`) |
| **EC2 Instance Profile** | Allows cluster nodes to access S3, KMS, CloudWatch (e.g., `EMR_EC2_DefaultRole`) |
| **Auto Scaling Role** | Allows auto-scaling actions |
| **EMR Studio Role** | For notebook/studio access |
| **Lake Formation Role** | For credential vending with Lake Formation |

### Auditing

- **CloudTrail:** Logs all EMR API calls (CreateCluster, AddSteps, TerminateCluster)
- **S3 access logs / CloudTrail data events:** Track data access
- **Spark History Server:** View completed Spark application logs
- **YARN Timeline Server:** Historical job metrics
- **CloudWatch Logs:** Forward application logs (Spark driver/executor, Hive, etc.)
- **CloudWatch Metrics:** Cluster utilization, HDFS, YARN metrics

---

## 4. Cost Optimization

### EMR Pricing

**EMR on EC2:**
- **EC2 instance cost** (On-Demand, Reserved, Spot, Savings Plans)
- **EMR surcharge:** Per instance-hour (varies by instance type; ~15-25% of On-Demand EC2 cost)
- **EBS volumes:** Standard EBS pricing
- **Data transfer:** Standard AWS data transfer

**EMR Serverless:**
- **vCPU-hour:** ~$0.052/vCPU-hour
- **Memory-hour:** ~$0.0057/GB-hour
- **Storage-hour:** ~$0.003/GB-hour (ephemeral storage)
- No EMR surcharge, no EC2 instance cost — all-inclusive

**EMR on EKS:**
- Pay for underlying EKS infrastructure (EC2/Fargate)
- EMR surcharge applies per vCPU-hour for pods running EMR workloads

### Cost Optimization Strategies

| Strategy | Savings | How |
|----------|---------|-----|
| **Spot Instances for Task Nodes** | 60-90% on compute | Use Instance Fleets with 10+ instance types; Spot only for task nodes |
| **Spot for Core Nodes (carefully)** | 50-70% | Only with HDFS replication ≥ 2 and Instance Fleets; risky for small clusters |
| **Transient clusters** | Eliminate idle cost | Process → terminate; S3 stores data |
| **Managed Scaling** | Match capacity to demand | Set min/max; auto-scale task nodes |
| **Graviton instances** | ~20% cheaper + faster | Use m6g, c6g, r6g instance families |
| **S3 vs. HDFS** | Reduce EBS/instance costs | Decouple storage from compute |
| **Right-size instances** | Avoid over-provisioning | Analyze YARN metrics; match memory/CPU to workload |
| **Savings Plans** | Up to 72% (committed use) | Compute Savings Plans cover EMR EC2 instances |
| **Reserved Instances** | Up to 72% | For persistent clusters with steady-state usage |
| **EMR Serverless** | Zero idle cost | For sporadic/variable workloads |
| **Compression (Snappy/ZSTD)** | Reduce S3 + shuffle costs | Less data to read/write/transfer |
| **Columnar formats (Parquet/ORC)** | Reduce S3 scan costs | Read only needed columns; better compression |

### Cost Comparison: EMR on EC2 vs. Serverless

| Factor | EMR on EC2 | EMR Serverless |
|--------|-----------|----------------|
| Idle cost | Yes (if persistent/over-provisioned) | No (pay only during execution) |
| Startup time | Minutes (cluster provisioning) | Seconds (pre-initialized workers) |
| Cost at scale (steady) | Cheaper with Spot + RI | More expensive for sustained workloads |
| Cost for sporadic jobs | Expensive (idle between jobs) | Cheaper (zero cost between jobs) |
| Control | Full (instance types, configs) | Limited (managed runtime) |

### Cost Traps

- **Leaving persistent clusters running overnight/weekends:** Use transient or schedule stop/start
- **EBS gp2 volumes on data-intensive clusters:** Use instance store (d-family) or gp3 for better $/IOPS
- **Ignoring shuffle optimization:** Poor partitioning → excessive shuffle → longer jobs → higher cost
- **Not enabling managed scaling:** Fixed cluster size means either over-paying (idle) or under-performing (queued jobs)
- **Running EMR when Athena suffices:** For simple SQL on S3, Athena is often cheaper (no cluster overhead)

---

## 5. High Availability, Disaster Recovery & Resilience

### EMR Cluster HA

**Primary Node HA (Multi-Master):**
- Deploy 3 primary nodes for automatic failover
- Supported on EMR 5.23+ with specific applications
- Requires external metastore (Glue Data Catalog or external MySQL/RDS for Hive Metastore)
- If primary fails in single-master mode → cluster terminates → must relaunch

**HDFS Resilience:**
- Default replication factor: 3 (for clusters with ≥4 core nodes)
- Replication factor 1-2 for small clusters
- Core node loss → HDFS re-replicates blocks (if capacity allows)
- **Best practice:** Don't rely on HDFS for durability — use S3 as source of truth

**Spot Instance Resilience:**
- **Instance Fleets** with multiple instance types — if one Spot pool is interrupted, EMR launches from another
- **Capacity-optimized allocation:** Chooses pools least likely to be interrupted
- **Task nodes on Spot:** If interrupted, tasks retry on remaining/new nodes — no data loss (task nodes have no HDFS)
- **Core nodes on Spot:** HDFS data may be lost — use replication factor ≥ 2 and limit Spot core ratio

### Job Resilience

- **Step retries:** Configure steps to retry on failure (up to configured limit)
- **Spark speculation:** Duplicate slow tasks on other nodes to avoid stragglers
- **Checkpointing:** Spark Structured Streaming checkpoints to S3 for exactly-once recovery
- **YARN application retries:** YARN can re-launch failed ApplicationMasters
- **Output committers:** Use EMRFS S3-optimized committer (avoids partial output on failure)

### DR Strategies

| Strategy | Implementation |
|----------|----------------|
| **Backup & Restore** | Data on S3 (inherently durable + CRR); Hive Metastore in Glue Catalog (Regional); cluster config in CloudFormation |
| **Pilot Light** | EMR cluster config ready (CloudFormation); launch in DR Region on demand; S3 data replicated via CRR |
| **Active-Active** | EMR clusters in both Regions; S3 CRR; Glue Catalog in both Regions; Route 53 or Step Functions routes jobs |

**Key for DR:** Since data is on S3 and metadata is in Glue Catalog, EMR clusters are largely **stateless** (when using transient clusters). Re-launching a cluster in another Region with the same config gives you DR.

---

## 6. Exam-Focused Section

### Straightforward Questions (7 examples)

**Q1:** A company needs to run Apache Spark jobs for nightly ETL on their S3 data lake. They want the least operational overhead. Which EMR deployment should they use?
**A:** **EMR Serverless.** No cluster provisioning, auto-scales, auto-releases resources when done. Submit Spark jobs and pay only during execution.

**Q2:** An EMR cluster uses Spot Instances. Which node type is safest to run on Spot?
**A:** **Task nodes.** They don't store HDFS data, so Spot interruption causes no data loss. Tasks are simply retried on other nodes.

**Q3:** A company wants to run Spark alongside other containerized workloads on their existing EKS cluster. Which EMR option is best?
**A:** **EMR on EKS.** Submit Spark jobs to EKS namespaces, sharing infrastructure with other workloads.

**Q4:** What is the recommended primary storage for EMR clusters that need to be cost-effective and allow transient (ephemeral) cluster patterns?
**A:** **Amazon S3 (EMRFS).** Data persists independently of the cluster lifecycle, enabling transient clusters that launch, process, and terminate.

**Q5:** How does EMR managed scaling decide when to add nodes?
**A:** It monitors **YARN metrics** (pending containers, available memory/vCPU). When pending workload exceeds available capacity, it adds nodes (up to the configured maximum).

**Q6:** A company runs a long-lived EMR cluster for HBase. How can they make the primary node highly available?
**A:** Deploy the cluster with **3 primary nodes (multi-master mode)**. EMR automatically handles failover between primary nodes.

**Q7:** Which EMR feature allows specifying multiple EC2 instance types for a node group to improve Spot availability?
**A:** **Instance Fleets.** You specify multiple instance types per fleet; EMR selects from available Spot pools, improving availability and reducing interruptions.

---

### Tricky / Scenario-Based Questions (7 examples)

**Q1:** A company runs a nightly Spark ETL job on EMR. The job processes 5 TB of data from S3, takes 2 hours, and uses a persistent 20-node cluster running 24/7. The cluster is idle 22 hours/day. How should they reduce costs with minimal job changes?

**A:** Switch to a **transient cluster** model:
1. Use **Step Functions** (or EventBridge schedule) to launch EMR cluster nightly
2. Submit Spark steps; cluster auto-terminates after completion
3. Use **Instance Fleets** with **Spot** for task nodes (60-90% savings on compute)
4. Keep data on S3 (already there)

This eliminates 22 hours/day of idle cost + adds Spot savings.

*Why wrong:*
- EMR Serverless: Valid, but "minimal job changes" — Serverless may require job config adjustments
- Just add managed scaling: Scales down but doesn't eliminate the base cluster cost
- Reserved Instances: Saves money but still paying for 22h of idle time

**Keywords:** "idle 22 hours/day" + "reduce costs" → transient cluster

---

**Q2:** A Spark job on EMR frequently fails midway because Spot Instances are reclaimed. The job processes 10 TB and takes 4 hours. How should they improve reliability while maintaining cost savings from Spot?

**A:** 
1. Use **Instance Fleets** with **10+ diverse instance types** and **capacity-optimized allocation** (fewer interruptions)
2. Run **core nodes On-Demand** (protect HDFS/data); **task nodes on Spot**
3. Enable **Spark checkpointing** to S3 (recover from midpoint on failure, not restart from scratch)
4. Set `MaximumOnDemandCapacityUnits` in managed scaling to ensure a baseline of stable nodes

*Why wrong:*
- All On-Demand: Solves reliability but loses cost savings entirely
- Single large instance type on Spot: Higher interruption risk (thin Spot pool)
- Reduce cluster size: Job takes longer → more exposure to interruptions

**Keywords:** "Spot reclaimed" + "maintain cost savings" → Instance Fleets diversity + On-Demand core + checkpoint

---

**Q3:** A company uses EMR with Hive and the Hive Metastore stored on the primary node's local disk. When the cluster terminates, they lose all table definitions and must re-crawl. What's the fix?

**A:** Use the **AWS Glue Data Catalog** as the external Hive Metastore. It's persistent, managed, and survives cluster termination. Table definitions are stored centrally and shared across EMR clusters, Athena, and Redshift Spectrum.

*Why wrong:*
- RDS MySQL metastore: Works but Glue Catalog is the AWS-native answer with broader integration
- Never terminate the cluster: Expensive; doesn't solve the root cause
- Export metastore on shutdown: Complex; error-prone; must re-import on next launch

**Keywords:** "lose table definitions on terminate" + "Hive Metastore" → Glue Data Catalog as external metastore

---

**Q4:** A data team runs both ad-hoc Spark queries (interactive, seconds to minutes) and large batch ETL jobs (hours). They want to share the same data but need different performance characteristics. Currently everything runs on one over-provisioned persistent cluster. How should they redesign?

**A:** Separate the workloads:
1. **Ad-hoc/interactive:** EMR Studio with a **small persistent cluster** (or EMR Serverless with pre-initialized workers for fast startup)
2. **Batch ETL:** **Transient EMR clusters** launched per job (large, Spot-heavy, auto-terminating)
3. **Shared data:** Both read/write to S3 via EMRFS; Glue Data Catalog as shared metastore

*Why wrong:*
- One big cluster for both: Over-provisioned for interactive; under-provisioned for batch peaks
- EMR Serverless for both: Good for batch; may not suit interactive notebook workflows
- Separate S3 buckets: Unnecessary data duplication

**Keywords:** "interactive" + "batch" + "different characteristics" → Separate clusters, shared S3 + Glue Catalog

---

**Q5:** A company uses EMR on EC2 with Spark. They read data from S3 and notice that Spark shuffle operations are slow. The shuffle data is 500 GB. Currently shuffle goes to EBS gp2 volumes. How should they improve shuffle performance?

**A:** Options (pick based on question constraints):
1. **Use instances with NVMe instance store** (e.g., i3, d3, r5d) — much faster local I/O for shuffle
2. Upgrade EBS to **gp3 with higher IOPS/throughput** provisioned
3. Use **HDFS on core nodes** (NVMe-backed) for shuffle intermediate data
4. **Optimize Spark partitioning** — reduce shuffle size via `repartition()`, broadcast joins for small tables

*Why wrong:*
- More memory (to avoid spill): Helps but doesn't fix I/O throughput
- S3 for shuffle: S3 is not suitable for random I/O shuffle operations
- Larger cluster: Doesn't fix per-node I/O bottleneck

**Keywords:** "shuffle slow" + "EBS gp2" → NVMe instance store or higher-performance EBS

---

**Q6:** A company runs EMR Spark jobs that process sensitive financial data. They need: (a) data encrypted at rest on S3 with their own KMS key, (b) encrypted shuffle between nodes, (c) restrict Spark jobs to specific S3 prefixes per user. What combination of features?

**A:**
- (a) **EMRFS with SSE-KMS** using a customer-managed CMK → S3 data encrypted at rest
- (b) **EMR Security Configuration** with in-transit encryption enabled → Spark shuffle encryption
- (c) **Lake Formation integration** (or EMRFS IAM role mapping) → per-user S3 access control at the table/prefix level

*Why wrong:*
- SSE-S3: Uses AWS-managed keys, not customer's own key
- VPC encryption only: Doesn't cover local disk or shuffle
- S3 bucket policy per user: Doesn't integrate with Spark user identity

---

**Q7:** A company decides between Athena and EMR Serverless for their analytics workloads on S3. Both can query Parquet data via Glue Catalog. When should they choose EMR Serverless over Athena?

**A:** Choose **EMR Serverless** when:
- Need complex transformations (joins, UDFs, ML) beyond SQL
- Writing Spark/PySpark code (not just SQL)
- Jobs require iterative processing (multiple passes over data)
- Need to write output in specific formats (Delta/Iceberg/Hudi)
- Workloads exceed Athena's query timeout (30 minutes) or result size limits

Choose **Athena** when:
- Standard SQL queries only
- Ad-hoc exploration
- Simple aggregations/filters
- Lowest effort (no job submission, just SQL)
- Query cost is acceptable ($5/TB scanned)

**Keywords:** "complex Spark transformations" / "PySpark" / "iterative" → EMR Serverless over Athena

---

### Common Exam Traps & Pitfalls

1. **"Least operational overhead" for Spark = EMR Serverless** — not EMR on EC2

2. **Task nodes = safe for Spot; Core nodes = risky for Spot** — because core nodes have HDFS data

3. **HDFS data is lost on cluster termination** — use S3 for persistent data; HDFS is temporary

4. **Glue Data Catalog replaces Hive Metastore** — persistent, shared, no cluster dependency

5. **Instance Fleets ≠ Instance Groups** — Fleets support multiple types per group (better for Spot); Groups are single-type

6. **EMR on EKS ≠ running Hadoop on K8s** — only Spark is supported on EMR on EKS (no Hive, HBase, Flink)

7. **EMR Serverless only supports Spark and Hive** — not HBase, Presto, Flink

8. **Managed Scaling protects HDFS** — won't scale down core nodes if it would violate replication factor

9. **EMRFS consistent view is obsolete** — S3 now has native strong consistency; don't recommend DynamoDB-based consistent view

10. **Athena vs. EMR is a common comparison** — SQL-only on S3 = Athena; complex ETL/ML/custom code = EMR

---

## 7. Cheat Sheet

### Must-Know Facts

- **EMR on EC2** = full-featured, any framework, full control
- **EMR Serverless** = least overhead for Spark/Hive, pay-per-use
- **EMR on EKS** = Spark on existing K8s cluster, multi-tenant
- **Primary node** = coordinator; **Core** = compute + HDFS; **Task** = compute only
- **Spot safety:** Task nodes (safe) > Core nodes (risky) > Primary (never)
- **Instance Fleets** = multiple instance types per group (Spot diversity)
- **Managed Scaling** = auto adds/removes core + task nodes based on YARN metrics
- **EMRFS** = S3 as file system; enables transient (ephemeral) clusters
- **Glue Data Catalog** = external metastore for Hive (persists across clusters)
- **Security Configuration** = reusable bundle of encryption settings
- **Transient clusters** = launch → process → terminate (cost-effective for batch)
- **Graviton instances** = ~20% cheaper, better performance for Spark

### Decision Flowchart

```
"Run Spark/Hive with least operational overhead"
  → EMR Serverless

"Run Spark on existing EKS cluster"
  → EMR on EKS

"Need HBase / Flink / Presto / multiple frameworks"
  → EMR on EC2

"SQL-only queries on S3 data"
  → Athena (not EMR)

"Cost-optimize Spot for big data batch"
  → EMR on EC2 with Instance Fleets + Task nodes on Spot

"Persistent NoSQL on HDFS"
  → EMR on EC2 with HBase (persistent cluster)

"Interactive notebooks for Spark"
  → EMR Studio + EMR on EC2 or Serverless

"Real-time stream processing"
  → EMR on EC2 with Flink or Spark Structured Streaming

"Nightly ETL, idle most of day"
  → Transient EMR cluster (or EMR Serverless)
```

### Key Differentiators

| Signal in Question | Answer |
|---|---|
| "Least operational overhead" + "Spark" | EMR Serverless |
| "Kubernetes" + "Spark" | EMR on EKS |
| "HBase" / "Flink" / "Presto" | EMR on EC2 |
| "Spot Instances" + "availability" | Instance Fleets (multiple types) |
| "Node type safe for Spot" | Task nodes |
| "Data lost on termination" | Store on S3 (EMRFS), not HDFS |
| "Table definitions lost" | Use Glue Data Catalog as metastore |
| "Idle cluster" + "cost" | Transient cluster or EMR Serverless |
| "Shuffle performance" | NVMe instance store or HDFS on local disks |
| "Fine-grained data access" + "EMR" | Lake Formation integration |
| "Cost-optimize steady-state cluster" | Savings Plans or Reserved Instances |
| "Simple SQL on S3" | Athena (not EMR) |

---

## 8. Additional Deep-Dive

### 10 More Tricky Scenario-Based Questions

**Q1:** A company runs a Spark Structured Streaming job on EMR that reads from Kinesis Data Streams and writes to S3 in near-real-time. The cluster crashed and they lost 2 hours of data. How should they prevent this?

**A:** Enable **Spark checkpointing to S3**. Spark Structured Streaming uses checkpoints to track the stream position (Kinesis shard iterator). On recovery, it resumes from the last checkpoint — no data loss. Additionally, ensure Kinesis **retention period** is at least 24-48 hours (data is still available in the stream for replay).

*Why wrong:*
- HDFS checkpoint: Lost with cluster crash
- Kinesis replay alone: Need the offset tracking (checkpoint) to know where to resume
- Duplicate cluster (active-active): Overkill for recovery

**Keywords:** "streaming" + "cluster crashed" + "lost data" → S3 checkpoint + Kinesis retention

---

**Q2:** A company uses EMR with Spark to process 20 TB daily. They read from S3 and write partitioned Parquet to another S3 location. The job takes 6 hours. They notice the final write step sometimes produces many small files (< 1 MB each) in output partitions. What's the fix?

**A:** The small files problem is caused by too many Spark partitions at write time. Fix:
1. **`repartition()` or `coalesce()`** before writing to reduce partition count
2. Use the **EMRFS S3-optimized committer** (avoids partial/duplicate files)
3. Consider writing with **Iceberg or Hudi** table format (auto-compaction of small files)
4. Post-process with a **Glue compaction job** if needed

*Why wrong:*
- More executor memory: Doesn't change partition count at write
- Larger cluster: More parallelism = MORE small files
- S3 lifecycle rule: Deletes files, doesn't merge them

---

**Q3:** A data engineering team wants to use EMR Spark with Lake Formation for column-level security. Their current EMR cluster has an EC2 instance profile with full S3 access. After enabling Lake Formation, Spark jobs still read all columns including restricted ones. Why?

**A:** The EC2 instance profile provides **direct S3 access**, bypassing Lake Formation. To enforce Lake Formation permissions with EMR:
1. Enable **Lake Formation integration** on the EMR cluster (runtime role / credential vending)
2. **Remove direct S3 permissions** from the EC2 instance profile
3. Grant Lake Formation permissions to the EMR execution role
4. Lake Formation will issue scoped credentials that enforce column restrictions

*Why tricky:* Like all Lake Formation integrations, the analytics engine can bypass LF if it has direct S3 IAM access.

---

**Q4:** A company runs a persistent EMR cluster (20 core nodes, r5.4xlarge) for interactive analytics. Utilization averages 30% during the day and 5% at night. They cannot use transient clusters because analysts need instant access to a running cluster. How should they reduce costs?

**A:** Enable **Managed Scaling** with aggressive scale-down:
- Set `MinimumCapacityUnits` to 5 (enough for night-time trickle)
- Set `MaximumCapacityUnits` to 25 (handle day-time peaks)
- Use **Task nodes on Spot** for burst capacity
- Keep core nodes minimal (5 On-Demand for HDFS stability)
- Alternatively: migrate persistent analytics to **EMR Serverless** with pre-initialized workers (if workload allows)

*Why wrong:*
- Terminate at night: "Cannot use transient" + analysts need instant access
- Reserved Instances for 20 nodes: Paying for 100% when using 30%
- Smaller instance types: May not have enough memory for interactive queries

---

**Q5:** A company migrates on-premises Hadoop (HDFS + Hive + Spark) to AWS. They have 500 TB of HDFS data. They want to minimize re-architecture but gain cloud benefits (elasticity, managed service). What migration approach?

**A:** 
1. **Copy HDFS data to S3** using **AWS DataSync** or **S3 DistCp** (Hadoop distributed copy)
2. Launch **EMR on EC2** with same frameworks (Hive + Spark)
3. Use **Glue Data Catalog** as Hive Metastore (replace on-prem MySQL metastore)
4. Configure EMRFS to read from S3 (minimal Spark code changes — just change paths from `hdfs://` to `s3://`)
5. Enable **managed scaling** for elasticity

*Why wrong:*
- Keep HDFS on EMR: Couples storage to compute; can't leverage S3 benefits
- Rewrite everything as Lambda: Massive re-architecture (violates "minimize re-architecture")
- EMR Serverless: Can't run Hive Metastore or HBase if needed

**Keywords:** "minimize re-architecture" + "Hadoop to AWS" → EMR on EC2 + S3 + Glue Catalog

---

**Q6:** An analytics team queries S3 data using both Athena (SQL analysts) and EMR Spark (data engineers). They need a shared metastore so both services see the same tables. What should they use?

**A:** **AWS Glue Data Catalog** as the shared metastore. Both Athena and EMR (Hive/Spark) can use Glue Data Catalog as their metadata store. Tables created in one service are immediately visible in the other.

*Why wrong:*
- Separate Hive Metastore on EMR: Athena can't access it
- Athena-only catalog: Not a separate thing — Athena uses Glue Catalog by default
- RDS MySQL metastore: Works for EMR but Athena can't use external MySQL

---

**Q7:** A financial services company needs to process market data with sub-second latency using Spark Structured Streaming. They're considering Kinesis → EMR Spark Streaming. The processing window is 1-5 seconds. Is EMR the right choice, or should they use something else?

**A:** For **sub-second latency**, **Apache Flink on EMR** (or Kinesis Data Analytics/Managed Flink) is better than Spark Structured Streaming. Spark Streaming has micro-batch architecture with minimum batch intervals of ~100ms but practically 1+ seconds. Flink is true event-at-a-time streaming with lower latency.

For sub-second:
- **Amazon Managed Service for Apache Flink** (formerly Kinesis Data Analytics) — fully managed, least overhead
- **Flink on EMR** — more control, self-managed cluster

*Why wrong:*
- Spark Structured Streaming: Micro-batch; achievable at 1-5s but not ideal for sub-second
- Lambda: Can't maintain state between invocations for streaming analytics
- Kinesis Data Streams alone: Delivery mechanism, not processing engine

---

**Q8:** A company has an EMR cluster with Instance Groups (single instance type: r5.2xlarge). During a Spot interruption event, all task nodes were reclaimed simultaneously because they were all in the same Spot pool. How should they prevent this?

**A:** Migrate from **Instance Groups to Instance Fleets**:
- Specify 10-15 diverse instance types (r5.2xlarge, r5.4xlarge, r5a.2xlarge, r5d.2xlarge, r6i.2xlarge, m5.2xlarge, etc.)
- Use **capacity-optimized allocation strategy** (launches from pools with most available capacity)
- This diversifies across multiple Spot pools — unlikely that ALL pools are reclaimed simultaneously

*Why tricky:* Instance Groups use a single type, creating single-pool risk. Instance Fleets solve this by design.

---

**Q9:** A company uses EMR to run Presto for interactive queries. Users experience slow startup when a new Presto query requires cluster scaling. How should they improve interactive query performance?

**A:** Multiple approaches:
1. Keep a **persistent cluster** with enough base capacity for interactive workload (don't rely on scale-up for latency-sensitive queries)
2. Use **managed scaling with conservative minimum** (high enough for typical query load)
3. Consider **Amazon Athena** as an alternative (serverless Presto/Trino — no cluster management, instant query capacity)
4. If staying on EMR: pre-warm task nodes (set minimum task nodes > 0)

*Best answer for exam:* If the question emphasizes "least operational overhead" for interactive SQL → **Athena** (serverless Presto). If it emphasizes "EMR with Presto" → persistent cluster with adequate base capacity.

---

**Q10:** A data lake team uses EMR with Apache Hudi for an incremental data pipeline. They write upserts to a Hudi table on S3. After several months, query performance degrades because of many small file versions. What should they do?

**A:** Run **Hudi compaction** (for Merge-On-Read tables) or **clustering** operations:
- **Compaction:** Merges delta log files with base Parquet files (MoR tables)
- **Clustering:** Re-organizes data files for better query performance (sorts/groups data)
- Schedule compaction as a separate EMR step (daily or after N commits)
- Use Hudi's inline or async compaction settings

*Why wrong:*
- Delete old versions: Hudi manages versions; manual delete would corrupt the table
- Increase read parallelism: Treats symptom, not cause
- Switch to HDFS: Doesn't solve file versioning issue

---

### Compare: EMR vs. Athena vs. Redshift vs. Glue ETL

| Dimension | EMR | Athena | Redshift | Glue ETL |
|-----------|-----|--------|----------|----------|
| **Primary use** | Large-scale data processing (ETL, ML, streaming) | Ad-hoc SQL queries on S3 | Data warehousing, BI queries | Managed ETL (Spark-based) |
| **Query type** | Spark, Hive, Presto, Flink (any) | SQL only (Presto/Trino) | SQL (PostgreSQL-based) | PySpark/Scala ETL |
| **Data location** | S3, HDFS, external | S3 only | Local storage + Spectrum (S3) | S3 |
| **Serverless option** | EMR Serverless | Always serverless | Redshift Serverless | Always serverless |
| **Latency** | Seconds to hours | Seconds to minutes | Sub-second to minutes | Minutes to hours |
| **Best for** | Complex transforms, large scale, multi-framework | Quick SQL exploration, simple analytics | BI reporting, complex SQL joins, dashboards | Scheduled ETL, data integration |
| **Cost model** | Per instance-hour + EMR fee | $5/TB scanned | Per node-hour (or RPU for serverless) | Per DPU-hour |
| **Exam trigger** | "Spark" / "Hadoop" / "streaming" / "complex processing" | "Simple SQL on S3" / "ad-hoc" | "BI" / "data warehouse" / "complex joins + aggregations" | "Managed ETL" / "visual ETL" / "crawlers" |

### When Does the Exam Expect EMR vs. Alternatives?

**Pick EMR when:**
- "Spark" / "Hadoop" / "MapReduce" / "Flink" mentioned
- Complex data transformations or ML at scale
- Streaming processing (Spark Streaming, Flink)
- Need specific open-source frameworks
- Processing petabytes with custom code
- "HBase" for NoSQL on HDFS

**Pick Athena when:**
- "Ad-hoc SQL" on S3
- "Serverless queries" on data lake
- "No infrastructure" + SQL only
- Cost-sensitive simple queries (pay-per-query)

**Pick Glue ETL when:**
- "Managed ETL" / "serverless ETL"
- "Visual ETL" (Glue Studio)
- "Crawlers" / "Data Catalog" emphasis
- Simpler ETL pipelines without cluster management

**Pick Redshift when:**
- "Data warehouse" / "BI reports"
- "Complex SQL joins" across large datasets
- "Sub-second query response" on structured data
- "Dashboard" / "QuickSight" integration

### All "Gotcha" Differences

| Gotcha | Detail |
|--------|--------|
| EMR Serverless ≠ all frameworks | Only Spark and Hive (no HBase, Flink, Presto) |
| EMR on EKS ≠ all frameworks | Only Spark |
| Task nodes have NO HDFS | Safe for Spot; losing them loses no data |
| Core nodes HAVE HDFS | Spot interruption = potential data loss |
| Managed scaling won't violate HDFS replication | Won't remove core nodes below replication factor |
| EMRFS consistent view is deprecated | S3 is now strongly consistent natively |
| Glue Data Catalog = external metastore | Replaces Hive Metastore on primary node; persists across clusters |
| Instance Fleets ≠ Instance Groups | Fleets = multi-type (Spot diversity); Groups = single-type |
| EMR Serverless has pre-initialized workers | Keeps workers warm for faster cold start (costs money while warm) |
| Graviton on EMR | Supported; ~20% cheaper; up to 30% better Spark performance |
| S3 committer matters | Use EMRFS optimized committer to avoid duplicate/partial output files |
| Multi-master requires external metastore | Can't use local Hive Metastore with 3 primary nodes |

### Decision Tree for Big Data Processing

```
START: What type of processing?
├── SQL queries on S3?
│   ├── Simple/ad-hoc → Athena
│   ├── Complex joins + BI → Redshift Spectrum or Redshift
│   └── High-concurrency interactive → Athena or Redshift Serverless
├── ETL/Transformation?
│   ├── Simple, visual, scheduled → Glue ETL
│   ├── Complex, custom Spark code → EMR (Serverless or EC2)
│   └── "Least overhead" + Spark → EMR Serverless
├── Streaming?
│   ├── Sub-second latency → Managed Flink or Flink on EMR
│   ├── Micro-batch (seconds) → Spark Structured Streaming on EMR
│   └── Simple transforms → Kinesis Data Firehose (no EMR needed)
├── ML/Feature Engineering?
│   ├── Large-scale Spark → EMR
│   └── Training → SageMaker (after EMR feature eng)
└── Need HBase (NoSQL on HDFS)?
    └── EMR on EC2 (persistent cluster)
```

---

*End of study notes for EMR (Spark, Hadoop, Managed Scaling)*
