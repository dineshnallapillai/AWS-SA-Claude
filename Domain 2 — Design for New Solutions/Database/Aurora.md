# AWS SAP-C02 Study Notes: Aurora (Global Database, Multi-Master, Serverless v2, Cloning)

---

## 1. Core Concepts & Theory

### Amazon Aurora Fundamentals

Amazon Aurora is a **cloud-native relational database** built by AWS from the ground up. It is **MySQL-compatible** and **PostgreSQL-compatible** but has a completely different storage architecture from standard RDS.

#### Why Aurora Is Different from RDS MySQL/PostgreSQL

| Aspect | Standard RDS | Aurora |
|--------|-------------|--------|
| Storage architecture | EBS volumes (single AZ, replicated within AZ) | **Distributed, fault-tolerant, self-healing storage** across 3 AZs |
| Storage replication | Within single AZ | **6 copies across 3 AZs** |
| Storage auto-scaling | Must pre-provision or use autoscaling | **Auto-scales** in 10 GB increments up to **128 TiB** |
| Failover time | 60–120 seconds (classic Multi-AZ) | **~30 seconds** (typically <15s with replicas) |
| Read replicas max | 5 | **15** |
| Read replica lag | Seconds–minutes (async over network) | **Typically <20ms** (shared storage, no data replication) |
| Performance | 1× MySQL/PostgreSQL baseline | **Up to 5× MySQL, 3× PostgreSQL** (AWS claim) |
| Backtrack | Not available | **Yes** (MySQL-compatible only) |
| Cloning | Not available | **Yes** (copy-on-write) |
| Global Database | Not available (cross-Region replicas only) | **Yes** (dedicated replication, <1s lag) |
| Serverless | Not available | **Yes** (Serverless v2) |
| Multi-Master | Not available | **Yes** (limited, being deprecated in favor of other patterns) |

---

### Aurora Storage Architecture

This is the foundation of everything Aurora does better than standard RDS.

```
Aurora Writer Instance
  ↕ writes to
Shared Distributed Storage Volume
  → 6 copies of data across 3 AZs (2 copies per AZ)
  → Auto-scales in 10 GB increments up to 128 TiB
  → Self-healing: bad blocks detected and repaired automatically
  ← Aurora Reader Instances read from the same storage (no data replication needed)
```

#### Key Storage Properties

| Property | Detail |
|----------|--------|
| **Replication** | **6 copies across 3 AZs** (2 per AZ) |
| **Write quorum** | Writes succeed with **4 of 6** copies acknowledged |
| **Read quorum** | Reads succeed with **3 of 6** copies |
| **Fault tolerance** | Survives loss of 2 copies (1 AZ) without write impact. Survives loss of 3 copies without read impact |
| **Self-healing** | Continuous background verification. Bad segments automatically repaired from healthy copies |
| **Auto-scaling** | Grows in **10 GB increments** automatically. No pre-provisioning. Max **128 TiB** |
| **Shrink** | Storage does **NOT automatically shrink** when data is deleted (as of current behavior — space is reused internally). This is an exam trap |
| **Encryption** | Encrypted at rest with KMS. All 6 copies, backups, snapshots, and replicas encrypted |

#### Why Replicas Are So Fast

- Aurora replicas read from the **same shared storage volume** — no data replication over the network.
- Replication lag is typically **<20 milliseconds** (vs seconds–minutes for standard RDS read replicas).
- Adding a replica does NOT add storage cost or replication overhead.

---

### Aurora Cluster Architecture

```
Aurora DB Cluster
  ├── Writer Instance (reads + writes)
  ├── Reader Instance 1 (reads only)
  ├── Reader Instance 2 (reads only)
  ├── ... up to 15 readers
  └── Shared Storage Volume (6 copies, 3 AZs, up to 128 TiB)

Endpoints:
  - Cluster (writer) endpoint → routes to current writer
  - Reader endpoint → load-balances across all readers
  - Custom endpoints → user-defined subsets of instances
  - Instance endpoints → direct connection to specific instance
```

#### Endpoint Types

| Endpoint | Routes To | Use Case |
|----------|----------|----------|
| **Cluster (writer) endpoint** | Current primary/writer instance | All write operations, and reads that need latest data |
| **Reader endpoint** | Load-balanced across all reader instances | Read-heavy queries, reporting |
| **Custom endpoint** | User-defined subset of instances | Route analytics queries to larger instances; route OLTP reads to smaller ones |
| **Instance endpoint** | Specific instance | Debugging, targeted testing |

---

### Aurora Global Database

Aurora Global Database provides **cross-Region replication** with **dedicated replication infrastructure** — not standard MySQL/PostgreSQL binlog replication.

#### Architecture

```
Primary Region (us-east-1):
  Aurora Cluster (Writer + Readers)
    ↕ dedicated storage-level replication
Secondary Region (eu-west-1):
  Aurora Cluster (Readers only — all read-only)
    ↕ dedicated storage-level replication
Secondary Region (ap-southeast-1):
  Aurora Cluster (Readers only)
```

#### Key Details

| Property | Detail |
|----------|--------|
| **Max secondary Regions** | **5** secondary Regions per Global Database |
| **Max readers per secondary** | **16** reader instances per secondary Region |
| **Replication lag** | Typically **<1 second** cross-Region (storage-level, not binlog) |
| **RPO** | Typically **<1 second** (replication lag) |
| **RTO** | **<1 minute** (managed planned failover) or **minutes** (unplanned — manual promotion) |
| **Replication mechanism** | Dedicated **storage-level replication** (not MySQL/PG replication). Sub-second because it replicates at the storage layer, not the database engine layer |
| **Secondary Region writes** | **Read-only** by default. To write, must promote secondary to primary (planned failover or detach) |
| **Write forwarding** | **Supported (Aurora MySQL only)**: Secondary Region can accept write queries and forward them to the primary Region. Adds latency (cross-Region round trip) |
| **Engines** | Aurora MySQL and Aurora PostgreSQL |

#### Failover Types

| Type | Scenario | Process | RTO | Data Loss |
|------|----------|---------|-----|-----------|
| **Managed Planned Failover** | Controlled Region migration (e.g., moving primary from us-east-1 to eu-west-1) | AWS demotes current primary, promotes secondary. Replication re-established. Zero-downtime goal | **~1 minute** | **0** (waits for replication to catch up) |
| **Unplanned Failover (Detach)** | Primary Region is down/unreachable | Detach secondary → promote to standalone primary. Manual process | **Minutes** | **Potential data loss** = replication lag at time of failure (typically <1s) |
| **Switchover (headless)** | Planned, but no secondary pre-configured | Create Global Database, add secondary, then switchover | Variable | 0 |

#### Write Forwarding (Aurora MySQL Only)

- Secondary Region readers can accept **write SQL statements**.
- Writes are forwarded to the **primary Region's writer** over the Global Database replication link.
- **Adds latency**: cross-Region round trip for every write.
- **Use case**: Applications in secondary Region can do occasional writes without connecting to the primary Region directly. Read-locally, write-globally.
- **NOT available for Aurora PostgreSQL.**

---

### Aurora Multi-Master

Aurora Multi-Master allows **multiple writer instances** in the same Aurora cluster — all instances can perform reads AND writes.

#### Key Details

| Property | Detail |
|----------|--------|
| **Writer instances** | Up to **4 writer instances** (all read/write) |
| **Scope** | **Single Region only** (cannot combine with Global Database) |
| **Conflict resolution** | Optimistic — writes to different rows succeed in parallel. Conflicting writes to the **same row** → one succeeds, other gets a write conflict error (application must retry) |
| **Replication** | All writers share the same storage volume. Changes propagated via **shared storage + in-memory page cache invalidation** |
| **Latency** | Very low (shared storage, no cross-node replication) |
| **Engine** | **Aurora MySQL only** (not PostgreSQL) |
| **Status** | **Limited adoption** — AWS is not actively promoting this. Most customers use single-writer + readers |
| **Failover** | Continuous availability — if one writer fails, others continue serving writes **with zero downtime** |

#### Multi-Master vs Single-Master

| Feature | Single-Master (Standard) | Multi-Master |
|---------|------------------------|-------------|
| Writers | 1 | Up to 4 |
| Readers | Up to 15 | All nodes are readers AND writers |
| Failover | Promote a reader (~15–30s) | **Zero downtime** (other writers continue) |
| Global Database | **Yes** | **No** |
| Serverless | **Yes** | **No** |
| PostgreSQL | Yes | **No** (MySQL only) |
| Conflict resolution | N/A (single writer) | Application must handle write conflicts |
| Use case | Most workloads | Extreme write availability requirements |

**Exam note:** Multi-Master is rarely the correct answer. The exam is more likely to test Global Database, Serverless v2, and standard Aurora HA patterns. Multi-Master may appear as a distractor.

---

### Aurora Serverless v2

Aurora Serverless v2 provides **automatic, fine-grained scaling** of compute capacity based on application demand.

#### Architecture

```
Application → Aurora Cluster Endpoint → Serverless v2 Instance(s)
  → Scales from min ACU to max ACU in increments of 0.5 ACU
  → Shared Aurora storage volume (same as provisioned Aurora)
```

#### Key Details

| Property | Detail |
|----------|--------|
| **Scaling unit** | **ACU (Aurora Capacity Unit)**: 1 ACU ≈ 2 GiB RAM + proportional CPU + networking |
| **Scale range** | **0.5 ACU to 256 ACU** per instance |
| **Scaling granularity** | **0.5 ACU increments** — fine-grained, not stepped |
| **Scaling speed** | **Instant** — scales in place without disruption (no connection drop, no failover) |
| **Min ACU** | Configurable (minimum: 0.5 ACU). **Cannot scale to zero** (v1 could, v2 cannot) |
| **Max ACU** | Configurable (up to 256 ACU per instance) |
| **Mixing** | Can mix **provisioned and Serverless v2 instances** in the same cluster. E.g., provisioned writer + Serverless v2 readers |
| **Multi-AZ** | **Yes** — Serverless v2 instances can be placed in different AZs |
| **Read replicas** | Serverless v2 instances CAN be Aurora readers |
| **Global Database** | **Yes** — Serverless v2 can participate in Global Database clusters |
| **Engines** | Aurora MySQL and Aurora PostgreSQL |
| **Connections** | No connection pooling needed (unlike v1) — scales compute, not connections |

#### Serverless v2 vs v1

| Feature | Serverless v1 (deprecated) | Serverless v2 |
|---------|--------------------------|---------------|
| **Scale to zero** | Yes | **No** (min 0.5 ACU) |
| **Scaling granularity** | Doubled capacity (2→4→8→16...) | **0.5 ACU increments** |
| **Scaling speed** | 30+ seconds (may drop connections) | **Instant** (in-place, no disruption) |
| **Mixing with provisioned** | No | **Yes** |
| **Global Database** | No | **Yes** |
| **Multi-AZ** | Limited | **Full support** |
| **Reader instances** | No (single instance) | **Yes** (multiple readers) |
| **VPC** | Required public access or VPC | Standard VPC |
| **Status** | **Deprecated** — migrate to v2 | **Current** |

**Exam critical:** Serverless v2 **cannot scale to zero**. If the question requires zero cost during idle, Serverless v2 is not the answer (consider DynamoDB On-Demand or Lambda).

---

### Aurora Cloning

Aurora cloning creates a **fast, space-efficient copy** of a database using **copy-on-write** semantics.

#### How Cloning Works

```
Source Aurora Cluster → Clone (new cluster)
  - Clone initially shares the SAME storage pages as the source
  - No data copied at creation time — instant, regardless of data size
  - When either source or clone modifies a page → new copy of that page is created
  - Over time, clone diverges from source as writes accumulate
```

#### Key Details

| Property | Detail |
|----------|--------|
| **Speed** | **Instant** — regardless of database size (10 GB or 10 TB, same speed) |
| **Storage cost** | Initially **near-zero** additional storage (shared pages). Only divergent pages cost extra |
| **Independence** | Clone is a **fully independent cluster** — can read/write, add/remove readers, delete independently |
| **Same account** | Cloning within the same account: instant |
| **Cross-account** | Supported via **AWS RAM** (Resource Access Manager) sharing. Clone into another account |
| **Encryption** | Clone inherits source encryption. If source is encrypted, clone uses the same KMS key (or you can specify a different one cross-account) |
| **Max clones** | **15 clones per source** (clones of clones count against the source's limit) |
| **Protocol** | Available for Aurora MySQL and Aurora PostgreSQL |

#### Clone vs Snapshot Restore vs Read Replica

| Feature | Clone | Snapshot Restore | Read Replica |
|---------|-------|-----------------|-------------|
| Speed | **Instant** | Minutes–hours (depends on size) | Minutes (creation) |
| Storage cost | Near-zero initially | Full copy | No extra (shared storage) |
| Writable | **Yes** (independent cluster) | **Yes** (new cluster) | **No** (read-only) |
| Independent | **Yes** | **Yes** | No (tied to primary) |
| Up-to-date data | At clone creation time | At snapshot time | Near real-time (<20ms lag) |
| Use case | Dev/test, experiment | DR restore, point-in-time recovery | Read scaling |

#### When to Use Cloning

- **Dev/test environments**: Create production clone for testing — no impact on production, minimal cost.
- **Experimentation**: Try schema changes, query experiments on a clone.
- **Load testing**: Clone production data for realistic load testing.
- **Data analysis**: Analysts work on a clone without affecting production.
- **Pre-migration testing**: Clone database, apply migration, verify, then apply to production.

---

### Aurora Backtrack (MySQL Only)

Backtrack allows you to **rewind** the database to a previous point in time **in place** — without restoring from a snapshot.

#### Key Details

| Property | Detail |
|----------|--------|
| **Engine** | **Aurora MySQL only** (not PostgreSQL) |
| **How it works** | Aurora maintains a change log. Backtrack undoes changes back to the target timestamp |
| **In-place** | **No new cluster created** — the same cluster is rewound. Unlike PITR which creates a new cluster |
| **Speed** | Seconds to minutes (depending on how far back) |
| **Max backtrack window** | **72 hours** |
| **Use case** | Undo accidental `DELETE` or `DROP TABLE`. Quickly recover from human error |
| **Limitation** | Must be enabled **at cluster creation** (or clone from a Backtrack-enabled cluster). Cannot be enabled on an existing cluster |
| **Not for DR** | Backtrack is for operational recovery (human error), not disaster recovery |
| **Global Database** | Backtrack is on the **primary Region cluster** only. Secondary Regions are affected when primary backtracks |
| **Cost** | $0.012 per million change records stored |

---

### Additional Aurora Features (Exam Relevant)

#### Aurora Machine Learning

- Call **SageMaker** and **Amazon Comprehend** directly from SQL queries.
- Use cases: sentiment analysis, fraud detection, recommendations — all via SQL `SELECT` statements.

#### Aurora Parallel Query

- Pushes query processing down to the **storage layer**.
- Speeds up analytical queries on transactional data without ETL.
- Available for Aurora MySQL.

#### Aurora Auto Scaling (for Readers)

- Automatically adds/removes **Aurora Replicas** based on CloudWatch metrics (CPU, connections).
- Uses **Application Auto Scaling** with target tracking or step policies.
- Combined with Serverless v2: readers auto-scale compute AND instance count.

#### Aurora Database Activity Streams

- Near real-time stream of database activity to **Amazon Kinesis Data Streams**.
- Use for compliance auditing, security monitoring.
- Captures all SQL activity (DML, DDL, etc.).
- **Encrypted with KMS** — decoupled from DB admins (auditors manage the KMS key).

---

### Default Limits Worth Memorizing

| Resource | Limit |
|----------|-------|
| Max storage per cluster | **128 TiB** |
| Storage replication | **6 copies, 3 AZs** |
| Max reader instances (standard) | **15** |
| Max reader instances (Global Database secondary) | **16 per secondary Region** |
| Max secondary Regions (Global Database) | **5** |
| Global Database replication lag | Typically **<1 second** |
| Multi-Master max writers | **4** |
| Serverless v2 min ACU | **0.5** |
| Serverless v2 max ACU | **256** per instance |
| Max clones per source | **15** |
| Backtrack max window | **72 hours** |
| Automated backup retention | **1–35 days** |
| Max DB cluster parameter groups | **50** |
| Failover time (with replicas) | **~15–30 seconds** |
| Read replica lag | Typically **<20 ms** |

---

## 2. Design Patterns & Best Practices

### When to Use Aurora vs Standard RDS

| Scenario | Aurora | Standard RDS |
|----------|--------|-------------|
| High availability with fast failover | **Yes** (15–30s, 6-copy storage) | Slower (60–120s classic Multi-AZ) |
| Read-heavy with low replica lag | **Yes** (<20ms, 15 replicas) | Higher lag (seconds–minutes, 5 replicas) |
| Auto-scaling storage | **Yes** (to 128 TiB) | Must pre-provision or manually scale |
| Cross-Region with <1s replication | **Yes** (Global Database) | Cross-Region read replicas (seconds–minutes lag) |
| Serverless auto-scaling compute | **Yes** (Serverless v2) | Not available |
| Instant cloning for dev/test | **Yes** | Not available (use snapshots, slower) |
| Oracle or SQL Server | **No** | Yes (RDS, RDS Custom) |
| Lowest cost for small databases | **No** (Aurora minimum cost higher) | Yes (db.t3.micro, single-AZ) |
| Budget-constrained dev environments | **Maybe** (Serverless v2 min 0.5 ACU) | Yes (stop instances, t3.micro) |

### Anti-Patterns

| Anti-Pattern | Why It's Wrong | Correct Approach |
|-------------|---------------|------------------|
| Aurora for Oracle/SQL Server workloads | Not supported | RDS or RDS Custom |
| Aurora Multi-Master for cross-Region writes | Multi-Master is single-Region only | Global Database with write forwarding |
| Serverless v2 when you need zero cost at idle | Min 0.5 ACU — cannot scale to zero | DynamoDB On-Demand or stop/delete during idle |
| Aurora Backtrack as a DR strategy | Backtrack is for operational recovery (human error), max 72 hours. Not for cross-Region DR | Global Database for DR |
| Using Aurora cloning for ongoing replication | Clone is a one-time copy (diverges). Not a continuous replica | Read replicas for ongoing read scaling |
| Aurora Global Database for active-active multi-Region writes | Secondary Regions are read-only (except write forwarding, which adds latency) | DynamoDB Global Tables for true active-active multi-Region writes |

### Common Architectural Patterns

#### Standard Production Aurora
```
Application → Aurora Cluster (writer endpoint)
            → Aurora Readers (reader endpoint, up to 15)
            → Custom endpoints (analytics → large instances, OLTP → smaller instances)
            → Auto Scaling (add/remove readers based on CPU)

Storage: 6 copies, 3 AZs, auto-scales to 128 TiB
Backups: Automated (35-day retention) + manual snapshots
```

#### Global Application with Aurora Global Database
```
Primary Region (us-east-1):
  Aurora Writer + 3 Readers
    ↕ storage-level replication (<1s lag)
Secondary Region (eu-west-1):
  Aurora 4 Readers (read-only) — serve EU read traffic
    ↕ storage-level replication
Secondary Region (ap-southeast-1):
  Aurora 2 Readers — serve APAC read traffic

Route 53: Latency-based routing
  - Writes → Primary Region writer endpoint
  - Reads → Nearest Region reader endpoint

Failover: Managed planned failover (RTO <1 min, RPO 0)
```

#### Mixed Provisioned + Serverless v2
```
Aurora Cluster:
  Writer: db.r6g.xlarge (provisioned, steady workload)
  Reader 1: Serverless v2 (0.5–64 ACU, auto-scales for bursty reads)
  Reader 2: Serverless v2 (0.5–64 ACU, auto-scales for bursty reads)

Benefits: Stable writer cost + elastic reader capacity
```

#### Dev/Test with Cloning
```
Production Aurora Cluster (100 TB)
  → Clone to Dev (instant, near-zero initial storage cost)
  → Clone to QA (instant)
  → Clone to Load Testing (instant)

Each clone is independent, writable, costs only for divergent data
Total additional cost: minimal until clones diverge significantly
```

#### Lambda + Aurora Serverless v2
```
API Gateway → Lambda
  → RDS Proxy (connection pooling)
    → Aurora Serverless v2 (auto-scales ACU based on load)
  → Secrets Manager (credential rotation)

Scales from near-zero to massive with no manual intervention
```

### Well-Architected Framework Alignment

**Reliability**
- 6 copies across 3 AZs — survives AZ failure.
- Auto-healing storage — detects and repairs corruption.
- Failover to replica in ~15–30 seconds.
- Global Database for cross-Region RPO <1s.
- Backtrack for instant operational recovery.

**Security**
- Encryption at rest (KMS) for all copies, backups, snapshots.
- IAM DB authentication (MySQL/PG compatible).
- Database Activity Streams for compliance auditing.
- VPC isolation, security groups, private subnets.
- Secrets Manager for credential rotation.

**Performance**
- 5× MySQL / 3× PostgreSQL throughput (AWS claim).
- Up to 15 read replicas with <20ms lag.
- Parallel Query for analytical workloads on transactional data.
- Serverless v2 for instant compute scaling.
- Custom endpoints for workload-specific routing.

**Cost Optimization**
- Serverless v2 for variable/unpredictable workloads (pay per ACU-hour).
- Cloning instead of full database copies for dev/test.
- Aurora I/O-Optimized pricing for I/O-heavy workloads (higher storage cost but no per-I/O charge).
- Reserved Instances for provisioned instances.
- Auto Scaling for readers — add during peak, remove during off-peak.

**Operational Excellence**
- Blue-Green Deployments for version upgrades.
- Backtrack for human-error recovery (MySQL).
- Cloning for safe experimentation.
- Performance Insights for query analysis.
- Event Notifications + EventBridge for automated alerting.

---

## 3. Security & Compliance

### Access Control

| Mechanism | Description |
|-----------|-------------|
| **IAM DB Authentication** | Generate short-lived tokens for MySQL/PG connections. Same as RDS |
| **RDS Proxy + IAM** | Proxy handles IAM auth + connection pooling |
| **Database users** | Standard MySQL/PG user management within the DB engine |
| **Security groups** | Network-level access control on the cluster |
| **VPC** | Aurora must be in a VPC. Use private subnets for production |
| **Secrets Manager** | Store and rotate DB credentials. Proxy integration |
| **Database Activity Streams** | Audit all DB activity to Kinesis (encrypted with separate KMS key) |

### Encryption

| Layer | Detail |
|-------|--------|
| **At rest** | AES-256 via KMS. **All 6 storage copies** encrypted. Snapshots, backups, clones inherit encryption |
| **In transit** | SSL/TLS for client connections. Enforced via `require_secure_transport` (MySQL) or `rds.force_ssl` (PG) |
| **Global Database replication** | Encrypted in transit between Regions (AWS-managed) |
| **Cross-account cloning** | Source encryption key must be shared or clone re-encrypted with target account's key |
| **Activity Streams** | Encrypted with a **separate KMS key** — can be managed by security team, not DBA team |
| **Enable at creation** | Like RDS, encryption must be enabled at cluster creation. Cannot encrypt existing unencrypted cluster in-place |

### Logging & Monitoring

| Tool | What It Captures |
|------|-----------------|
| **CloudWatch Metrics** | Same as RDS + Aurora-specific: AuroraReplicaLag, AuroraGlobalDBReplicaLagMillis, ServerlessDatabaseCapacity, ACUUtilization, BufferCacheHitRatio, VolumeBytesUsed |
| **Enhanced Monitoring** | OS-level metrics (1-second granularity) |
| **Performance Insights** | DB load analysis, top SQL, wait events |
| **Database Activity Streams** | All SQL activity → Kinesis. For compliance and security auditing |
| **CloudTrail** | Aurora management API calls |
| **CloudWatch Logs** | DB engine logs: error, slow query, audit, general |
| **AWS Config** | Rules: `aurora-mysql-backtracking-enabled`, `rds-cluster-multi-az-enabled` |

### Compliance Patterns

- **SEC 17a-4 / Financial**: Database Activity Streams (tamper-proof via separate KMS key) + encrypted storage + CloudTrail.
- **HIPAA**: Encryption (rest + transit) + IAM auth + Activity Streams + VPC isolation.
- **GDPR right to erasure**: Aurora supports standard `DELETE` operations. Backtrack can undo deletes (plan retention windows carefully).
- **Data residency**: Global Database secondary Regions must be in approved locations. Use SCPs to restrict.

---

## 4. Cost Optimization

### Aurora Pricing Models

#### Standard (I/O-Inclusive) vs I/O-Optimized

| Pricing | Storage | I/O Cost | Best For |
|---------|---------|----------|----------|
| **Aurora Standard** | $0.10/GB/month | **$0.20/million read I/Os + $0.20/million write I/Os** | Low-to-moderate I/O workloads |
| **Aurora I/O-Optimized** | $0.225/GB/month (2.25×) | **$0 (unlimited I/O included)** | I/O-intensive workloads (>25% of DB cost is I/O) |

**Rule of thumb:** If I/O costs exceed **25% of your total Aurora bill**, switch to I/O-Optimized to save money. You can switch between pricing modes.

#### Compute Pricing

| Type | Pricing |
|------|---------|
| **Provisioned (On-Demand)** | Per instance-hour (same as RDS) |
| **Provisioned (Reserved)** | 1 or 3 year RI — up to **72% savings** |
| **Serverless v2** | Per ACU-hour. 1 ACU ≈ $0.12/hour (us-east-1). Min charge = min ACU × hours |

#### Storage & Backup Pricing

| Component | Cost (us-east-1) |
|-----------|-----------------|
| Storage (Standard) | $0.10/GB/month |
| Storage (I/O-Optimized) | $0.225/GB/month |
| I/O (Standard) | $0.20/million requests |
| Backup storage (beyond free) | $0.021/GB/month |
| Backtrack change records | $0.012/million records |
| Global Database replicated writes | Per replicated write I/O |
| Snapshot export to S3 | $0.01/GB |

### Cost-Saving Strategies

1. **I/O-Optimized for I/O-heavy workloads**: Eliminates per-I/O charges. Break-even is when I/O costs exceed ~25% of total Aurora bill.

2. **Reserved Instances for provisioned instances**: Up to 72% savings for steady-state writer/readers.

3. **Serverless v2 for variable workloads**: Pay only for ACU-hours consumed. Mix with provisioned writer for cost optimization.

4. **Cloning instead of snapshot restore for dev/test**: Clone is instant and costs near-zero initially. Snapshot restore creates a full copy (more storage cost, slower).

5. **Aurora Auto Scaling**: Add readers during peak, remove during off-peak. Don't over-provision readers.

6. **Graviton instances**: db.r7g, db.r6g — up to 20% cheaper with better performance.

7. **Delete unused clusters**: Aurora charges for storage even when instances are stopped/deleted (if storage remains). Delete clusters you no longer need.

8. **Backtrack vs PITR**: Backtrack is cheaper and faster for recent human errors (no new cluster needed). PITR creates a new cluster with full storage cost.

### Common Cost Traps

- **Aurora storage does NOT automatically shrink**: If you load 100 TB then delete 80 TB, you may still be charged for allocated storage (space is internally reused but high-water mark remains). Need to export/reimport or clone to compact.
- **Serverless v2 minimum ACU**: Min 0.5 ACU × 24/7 = ~$43/month minimum. Not free during idle.
- **I/O costs can surprise you**: Standard pricing charges per I/O. A write-heavy workload can generate massive I/O costs. Monitor `VolumeReadIOPs` and `VolumeWriteIOPs` in CloudWatch.
- **Global Database replication writes**: You pay for replicated write I/O in each secondary Region.
- **Backup storage beyond free tier**: Free backup storage = cluster storage size. Beyond that, $0.021/GB/month.
- **Cross-Region data transfer for Global Database**: Ongoing transfer cost for replication traffic.
- **Compute Savings Plans DON'T cover Aurora**: Use Aurora/RDS Reserved Instances.

### Cost Comparison: Aurora vs Standard RDS

| Workload | Aurora | Standard RDS | Notes |
|----------|--------|-------------|-------|
| Small dev/test | More expensive (higher minimum) | Cheaper (db.t3.micro, single-AZ) | RDS wins for small workloads |
| Large production (>1 TB, write-heavy) | Often cheaper overall | I/O costs can be higher on gp2 | Aurora I/O-Optimized can save significantly |
| Read-heavy (many replicas) | 15 replicas, shared storage | 5 replicas, each with full EBS cost | Aurora wins |
| Variable/bursty | Serverless v2 | No serverless option | Aurora wins |
| Cross-Region DR | Global Database (<1s lag) | Cross-Region read replica (seconds lag) | Aurora wins on RPO |

---

## 5. High Availability, Disaster Recovery & Resilience

### Aurora Native HA (Single Region)

- **6 copies across 3 AZs** — no additional configuration needed.
- Write tolerance: lose 2 copies, reads continue. Lose 3 copies, writes continue.
- Self-healing storage repairs damage automatically.
- **Failover to replica: ~15–30 seconds** (DNS update). If no replica exists, Aurora creates a new instance (~10 minutes).
- **Failover priority**: Assign priority tiers (0–15) to reader instances. Lower number = higher priority for promotion. Same priority = largest instance promoted.

### HA Comparison

| Pattern | RTO | RPO | Setup |
|---------|-----|-----|-------|
| **Aurora single-master + 1 reader (same Region)** | ~15–30 seconds | 0 (shared storage) | Standard production |
| **Aurora + RDS Proxy** | **Seconds** | 0 | Best single-Region HA |
| **Aurora Global Database (managed planned failover)** | **<1 minute** | **0** (waits for sync) | Planned Region migration |
| **Aurora Global Database (unplanned failover)** | **Minutes** (manual) | **<1 second** (replication lag) | Region-down disaster |
| **Aurora Multi-Master** | **0** (continuous write availability) | **0** | Extreme write availability (niche) |

### DR Patterns

#### Single-Region DR
```
Aurora Cluster:
  Writer (AZ-a) + Reader (AZ-b) + Reader (AZ-c)
  Storage: 6 copies across 3 AZs
  
Failover: Writer fails → Reader promoted in ~15s
         AZ-a fails → Storage still has 4/6 copies, Readers in AZ-b/c continue
```

#### Cross-Region DR with Global Database
```
us-east-1 (Primary): Writer + 3 Readers
    ↕ <1 second replication
eu-west-1 (Secondary): 2 Readers (read-only)

Planned failover: Managed, <1 min RTO, 0 RPO
Unplanned failover: Detach secondary, promote, minutes RTO, <1s RPO
```

#### Cross-Region DR with Cross-Region Read Replicas (Without Global Database)
```
us-east-1: Aurora Cluster (standard)
  → Cross-Region Aurora Read Replica (eu-west-1)
  
Less capable than Global Database:
  - Uses MySQL/PG binlog replication (higher lag: seconds–minutes)
  - No managed failover
  - Promotion is manual
```

### Backup & Recovery

| Method | RPO | RTO | Notes |
|--------|-----|-----|-------|
| **Automated backup + PITR** | 5 minutes (continuous backup) | Minutes (creates new cluster) | Up to 35 days retention |
| **Manual snapshots** | At snapshot time | Minutes | Indefinite retention, cross-Region copy |
| **Backtrack (MySQL only)** | Seconds (change log) | Seconds–minutes (in-place) | Max 72 hours. In-place — no new cluster |
| **Clone** | At clone time | Instant | For creating test/dev copies |
| **Global Database** | <1 second | <1 minute (planned) / minutes (unplanned) | Best cross-Region DR |

---

## 6. Exam-Focused Section

### Straightforward Questions (7 examples)

**Q1:** How many copies of data does Aurora maintain and across how many AZs?
**A:** **6 copies across 3 AZs** (2 per AZ). This is automatic for all Aurora clusters.

**Q2:** What is the maximum storage capacity of an Aurora cluster?
**A:** **128 TiB.** Storage auto-scales in 10 GB increments.

**Q3:** How many read replicas can an Aurora cluster support?
**A:** **15** (vs 5 for standard RDS).

**Q4:** What is the typical replication lag for Aurora Global Database?
**A:** **Less than 1 second.** Uses dedicated storage-level replication, not binlog.

**Q5:** Can Aurora Serverless v2 scale to zero?
**A:** **No.** Minimum is 0.5 ACU. Serverless v1 could scale to zero but is deprecated.

**Q6:** Which Aurora feature allows rewinding the database to a previous point in time without creating a new cluster?
**A:** **Aurora Backtrack** (MySQL only). In-place rewind up to 72 hours.

**Q7:** How does Aurora cloning work?
**A:** **Copy-on-write.** Clone initially shares the same storage pages as the source. Only divergent pages incur additional storage cost. Creation is instant regardless of database size.

---

### Tricky / Scenario-Based Questions (7 examples)

**Q1:** A company runs Aurora MySQL with a 50 TB database. They need to provide a writable copy of the full production database to the QA team daily. Creating a snapshot and restoring it takes 2 hours and doubles the storage cost. What is a better approach?

**A:** Use **Aurora Cloning**.
- **Why?** Clone is **instant** regardless of database size (50 TB clones in seconds). Initial storage cost is **near-zero** (copy-on-write). QA team gets a fully independent, writable cluster.
- **Why not a read replica?** Read replicas are read-only. QA needs to write.
- **Why not snapshot restore?** Too slow (2 hours) and full storage cost.
- **Key phrases:** "writable copy" + "production data" + "fast" + "minimize cost" → Aurora Clone.

**Q2:** A global SaaS company needs a relational database accessible from us-east-1, eu-west-1, and ap-southeast-1. Reads must be local to each Region. Writes from Europe should update the database without requiring the application to connect to us-east-1. What should they use?

**A:** **Aurora Global Database with Write Forwarding** (Aurora MySQL).
- **How:** Primary writer in us-east-1. Secondary Regions in eu-west-1 and ap-southeast-1 with readers. Enable **write forwarding** on secondary Regions.
- **Why write forwarding?** EU application sends writes to the local secondary. Secondary forwards the write to us-east-1 primary automatically. Application doesn't need cross-Region connectivity logic.
- **Why not DynamoDB Global Tables?** The question says "relational database." DynamoDB is NoSQL.
- **Why not Aurora Multi-Master?** Multi-Master is **single-Region only** — cannot span Regions.
- **Limitation:** Write forwarding adds cross-Region latency. PostgreSQL does NOT support write forwarding.
- **Key phrases:** "multi-Region" + "local reads" + "writes from secondary Region" → Aurora Global Database + Write Forwarding.

**Q3:** A company has an Aurora PostgreSQL cluster with 80 TB of data. A developer accidentally runs `DELETE FROM customers` without a WHERE clause, deleting all customer records. They need to recover the data as quickly as possible. The cluster has automated backups with 7-day retention. What is the fastest recovery?

**A:** **Point-in-Time Recovery (PITR)** — restore to a timestamp just before the DELETE.
- **Why not Backtrack?** Backtrack is **Aurora MySQL only** — not available for PostgreSQL.
- **PITR:** Creates a new Aurora cluster restored to the exact second before the DELETE (within 5 minutes of transaction log granularity). Takes minutes to create, but the data is complete.
- **Why not snapshot restore?** Snapshots are daily — RPO could be up to 24 hours. PITR can restore to within 5 minutes.
- **Key phrases:** "Aurora PostgreSQL" + "accidental DELETE" + "fastest recovery" → PITR (not Backtrack — PG doesn't support it).

**Q4:** A company uses Aurora Global Database with a primary in us-east-1 and secondary in eu-west-1. During a planned migration, they want to make eu-west-1 the new primary with zero data loss. What should they do?

**A:** Use **Managed Planned Failover**.
- **How:** AWS initiates a managed planned failover. The current primary (us-east-1) is demoted. AWS ensures all data is replicated to eu-west-1 before promoting it. Replication is re-established in the reverse direction.
- **RTO:** <1 minute. **RPO:** 0 (AWS waits for replication to complete before switching).
- **Why not detach and promote?** Detach is for **unplanned** failover — it doesn't wait for replication to catch up, so there's potential data loss.
- **Key phrases:** "planned migration" + "zero data loss" + "Global Database" → Managed Planned Failover.

**Q5:** A company needs a database that can continue accepting writes even if one of the database instances fails, with zero downtime. They require this within a single Region. What Aurora feature provides this?

**A:** **Aurora Multi-Master.**
- **Why?** All instances (up to 4) are writers. If one writer fails, others continue serving writes with **zero downtime** — no failover needed.
- **Why not standard Aurora?** Standard Aurora has a single writer. If the writer fails, a reader is promoted (~15–30s downtime).
- **Why not Global Database?** Global Database is for cross-Region. The question specifies single Region.
- **Limitation:** MySQL only, requires application to handle write conflicts, limited feature set (no Global DB, no Serverless).
- **Key phrases:** "zero downtime" + "writer failure" + "single Region" + "continuous writes" → Multi-Master.

**Q6:** A startup has an Aurora PostgreSQL database with unpredictable traffic — near zero at night and spikes of 10,000 connections during product launches. They want to minimize costs during quiet periods and auto-scale during peaks. An architect proposes Aurora Serverless v1. Is this the best choice?

**A:** **No. Use Aurora Serverless v2.**
- **Why not v1?** Serverless v1 is **deprecated**. It scaled in large steps (doubling), took 30+ seconds, and could drop connections during scaling.
- **Why Serverless v2?** Scales in **0.5 ACU increments**, scales **instantly** without disruption, supports **Multi-AZ**, **read replicas**, and **Global Database**.
- **Cost during quiet periods:** Min 0.5 ACU (~$43/month). NOT zero — but much cheaper than a provisioned db.r6g.xlarge (~$300+/month).
- **Connection handling:** For 10,000 connections, use **RDS Proxy** in front of Serverless v2.
- **Key phrases:** "unpredictable traffic" + "auto-scale" + "minimize cost" → Serverless v2 (not v1).

**Q7:** An operations team creates an Aurora clone of a 10 TB production database for load testing. After the clone is created, they notice the clone's storage shows only a few GB. Over the next week of testing, storage grows to 3 TB. Why?

**A:** **Copy-on-write semantics.** The clone initially shares the **same storage pages** as the source — no data is physically copied. The "few GB" is the minimal metadata for the new cluster. As the load test writes data and modifies rows, **only changed pages are allocated as new storage** (divergent pages). After a week of writes, 3 TB of pages have diverged from the source.
- **This is by design** — it's the key benefit of cloning: instant creation, pay only for what changes.
- **Key phrases:** "clone" + "small initial storage" + "grows over time" → Copy-on-write.

---

### Common Exam Traps & Pitfalls

1. **Aurora ≠ RDS MySQL/PostgreSQL**: Different storage architecture, different HA model, different limits (15 vs 5 replicas, 128 TiB vs 64 TiB). Don't confuse them.

2. **Serverless v2 CANNOT scale to zero**: Min 0.5 ACU. V1 could but is deprecated. If the question requires truly zero cost at idle, Aurora Serverless is not the answer.

3. **Backtrack is Aurora MySQL ONLY**: Not PostgreSQL. If the question mentions PostgreSQL + Backtrack, it's wrong.

4. **Multi-Master is single-Region ONLY**: Cannot combine with Global Database. If the question needs cross-Region writes, the answer is Global Database with write forwarding (MySQL only) or DynamoDB Global Tables.

5. **Write Forwarding is Aurora MySQL ONLY**: Not PostgreSQL. Secondary Regions for PostgreSQL are strictly read-only.

6. **Aurora storage does NOT shrink automatically**: High-water mark behavior. If you bulk-delete data, you may need to clone and promote to reclaim space.

7. **Global Database secondary Regions are READ-ONLY**: Cannot write directly to a secondary Region (except via write forwarding for MySQL). Must promote secondary to become writable (breaks Global Database).

8. **Clone is a point-in-time copy, NOT a continuous replica**: Data in the clone diverges from source as soon as either is written to. For ongoing sync, use read replicas.

9. **Managed Planned Failover vs Unplanned (Detach)**: Planned = zero data loss, <1 min. Unplanned (detach) = potential data loss (replication lag at time of failure), manual.

10. **Aurora RI vs Compute Savings Plans**: Compute Savings Plans do NOT cover Aurora/RDS. Must use Aurora/RDS Reserved Instances.

11. **Failover priority tiers (0–15)**: Lower number = higher priority. Assign tier 0 to your largest reader for fastest failover.

12. **Backtrack must be enabled at cluster creation**: Cannot be enabled on an existing cluster (must clone from Backtrack-enabled cluster).

13. **Custom endpoints**: Exam tests whether you know to route analytical queries to large instances and OLTP to smaller ones using custom endpoints. Reader endpoint round-robins all readers — may route analytics to a small OLTP reader.

14. **I/O-Optimized vs Standard pricing**: If the question mentions "I/O-heavy workload" + "unpredictable I/O costs," the answer may be switching to I/O-Optimized pricing.

15. **Aurora storage replication is NOT the same as Multi-AZ**: Aurora always has 6 copies across 3 AZs (storage-level). "Multi-AZ" in Aurora context means having reader instances in multiple AZs for fast compute failover. You still need readers for fast failover — storage redundancy alone doesn't give you fast compute failover.

---

## 7. Cheat Sheet

### Must-Know Facts for Exam Day

| Fact | Value |
|------|-------|
| Storage copies | **6 copies, 3 AZs** |
| Write quorum | **4 of 6** |
| Read quorum | **3 of 6** |
| Max storage | **128 TiB** |
| Storage auto-scaling | **10 GB increments** |
| Max read replicas | **15** |
| Replica lag | **<20 ms** (shared storage) |
| Failover (with replica) | **~15–30 seconds** |
| Global Database max secondary Regions | **5** |
| Global Database readers per secondary | **16** |
| Global Database replication lag | **<1 second** |
| Global Database planned failover RTO | **<1 minute** |
| Write Forwarding | **Aurora MySQL only** |
| Multi-Master max writers | **4** (MySQL only, single-Region) |
| Serverless v2 min ACU | **0.5** |
| Serverless v2 max ACU | **256** |
| Serverless v2 scale to zero | **No** |
| Max clones per source | **15** |
| Backtrack max window | **72 hours** (MySQL only) |
| Backtrack enable timing | **At cluster creation only** |
| Backup retention | **1–35 days** |
| Aurora storage shrinks? | **No** (high-water mark) |
| I/O-Optimized break-even | I/O costs >**25%** of total bill |

### Key Differentiators

| Aurora | Standard RDS (MySQL/PG) |
|--------|------------------------|
| 6 copies, 3 AZs (auto) | EBS replication within AZ |
| 128 TiB max | 64 TiB max |
| 15 replicas, <20ms lag | 5 replicas, seconds–minutes lag |
| Global Database (<1s) | Cross-Region replica (seconds+) |
| Serverless v2 | No serverless |
| Cloning (instant) | No cloning (snapshot restore) |
| Backtrack (MySQL) | No Backtrack |
| Failover ~15–30s | 60–120s (classic Multi-AZ) |
| Higher minimum cost | Lower cost for small DBs |

| Global Database | Multi-Master |
|----------------|-------------|
| Cross-Region (up to 5) | Single-Region only |
| 1 writer, many readers | Up to 4 writers |
| <1s replication lag | Shared storage (near-zero lag) |
| Managed failover | Continuous availability (no failover) |
| Write forwarding (MySQL) | Direct writes on all nodes |
| MySQL + PostgreSQL | MySQL only |

| Serverless v2 | Provisioned |
|--------------|-------------|
| ACU-based scaling (0.5–256) | Fixed instance type |
| Instant scaling | Manual scaling or Auto Scaling (add instances) |
| Pay per ACU-hour | Pay per instance-hour |
| Can't scale to zero | Can stop instance (7-day auto-restart) |
| Mix with provisioned in same cluster | Can mix with Serverless v2 |
| Best for variable workloads | Best for steady-state |

| Clone | Snapshot Restore | Read Replica |
|-------|-----------------|-------------|
| Instant | Minutes–hours | Minutes (creation) |
| Near-zero initial cost | Full copy cost | No extra storage (shared) |
| Writable | Writable | Read-only |
| Point-in-time copy | Point-in-time copy | Continuous replication |
| Independent cluster | Independent cluster | Tied to primary |
| For dev/test/experiment | For DR/recovery | For read scaling |

### Decision Flowchart

```
Question says "relational" + "highest HA" + "auto-scaling storage"?
  → Aurora

Question says "cross-Region DR" + "RPO <1 second" + "relational"?
  → Aurora Global Database

Question says "writes from secondary Region" + "MySQL"?
  → Aurora Global Database + Write Forwarding

Question says "zero downtime on writer failure" + "single Region"?
  → Aurora Multi-Master (MySQL only)

Question says "variable/unpredictable traffic" + "auto-scale compute"?
  → Aurora Serverless v2

Question says "scale to zero" + "relational"?
  → NOT Aurora Serverless v2 (min 0.5 ACU). Consider DynamoDB On-Demand or Lambda

Question says "instant writable copy of production DB" + "dev/test"?
  → Aurora Clone (copy-on-write)

Question says "undo accidental DELETE" + "no new cluster" + "MySQL"?
  → Aurora Backtrack

Question says "undo accidental DELETE" + "PostgreSQL"?
  → PITR (Backtrack not available for PG)

Question says "route analytics queries to specific instances"?
  → Aurora Custom Endpoints

Question says "I/O costs too high" + "Aurora"?
  → Switch to Aurora I/O-Optimized pricing

Question says "Lambda + Aurora" + "connection management"?
  → RDS Proxy + Aurora (Serverless v2 for variable load)

Question says "Oracle or SQL Server"?
  → NOT Aurora. Use RDS or RDS Custom

Question says "active-active multi-Region writes" + "relational"?
  → Very limited with Aurora. Consider DynamoDB Global Tables for true active-active

Question says "cross-Region" + "planned failover" + "zero data loss"?
  → Aurora Global Database Managed Planned Failover

Question says "cross-Region" + "unplanned failover"?
  → Aurora Global Database → detach secondary → promote (RPO <1s, manual)

Question says "15 read replicas with sub-millisecond lag"?
  → Aurora (not standard RDS — RDS has 5 replicas with seconds of lag)

Question says "128 TiB database" + "auto-scaling storage"?
  → Aurora (RDS maxes at 64 TiB, no auto-scaling storage)
```

---

*Generated for AWS SAP-C02 exam preparation. Last updated: 2026-04-26.*
