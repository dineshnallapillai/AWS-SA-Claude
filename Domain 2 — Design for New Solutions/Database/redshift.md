# Amazon Redshift — Spectrum, Concurrency Scaling, RA3 Nodes

---

## 1. Core Concepts & Theory

### What Is Amazon Redshift?
- **Fully managed, petabyte-scale data warehouse** based on PostgreSQL (heavily modified — not a drop-in replacement)
- **Columnar storage** + **massively parallel processing (MPP)** architecture
- Designed for **OLAP** (Online Analytical Processing), NOT OLTP
- Supports SQL queries against structured and semi-structured data (JSON, Parquet, ORC)
- Integrates deeply with the AWS analytics ecosystem (S3, Glue, Athena, Lake Formation, QuickSight, EMR)

### Architecture

```
              ┌──────────────────────────────────┐
              │         Leader Node               │
              │  (SQL parsing, query planning,    │
              │   aggregation of results)         │
              └──────────┬───────────────────────┘
                         │
          ┌──────────────┼──────────────────┐
          │              │                  │
   ┌──────▼──────┐ ┌────▼────────┐ ┌───────▼─────┐
   │ Compute Node│ │Compute Node │ │Compute Node │
   │  (Slices)   │ │  (Slices)   │ │  (Slices)   │
   └─────────────┘ └─────────────┘ └─────────────┘
```

- **Leader Node**: Receives queries, develops execution plans, coordinates compute nodes, aggregates results — clients connect ONLY to the leader node
- **Compute Nodes**: Execute query plans in parallel; each node divided into **slices** (each slice processes a portion of data)
- **Slices**: Each slice is allocated a portion of memory and disk; number of slices depends on node type

### Node Types

| Node Family | Type | Storage | Key Characteristic |
|-------------|------|---------|-------------------|
| **DC2** | Dense Compute | Local SSD | Fast local storage, fixed capacity |
| **RA3** | Managed Storage | S3-backed (Redshift Managed Storage — RMS) | Separate compute from storage, scale independently |
| **DS2** | Dense Storage | Local HDD | **Legacy — replaced by RA3** |

### RA3 Nodes (Exam Critical)
- **Decouple compute from storage** — compute nodes use large local SSD cache + automatically tier data to S3
- **Redshift Managed Storage (RMS)**: Data stored in S3, transparently managed — you don't interact with S3 directly
- **Scale compute and storage independently** — add nodes for more compute, storage scales automatically
- **Node sizes**: ra3.xlplus (4 vCPU, 32 GB), ra3.4xlarge (12 vCPU, 96 GB), ra3.16xlarge (48 vCPU, 384 GB)
- **Minimum 2 nodes** for ra3.xlplus and ra3.4xlarge; minimum 2 nodes for ra3.16xlarge
- **Cross-AZ data sharing** — RA3 enables data sharing between clusters (because data is in S3)
- **Automatic data tiering**: Hot data cached on local NVMe SSD; cold data on S3 — transparent to queries
- **Exam tip**: RA3 is the **default recommendation** for new clusters; DS2 is legacy

### Redshift Spectrum
- Query **exabytes of data directly in S3** without loading it into Redshift
- Uses **external tables** defined in an **external schema** backed by **AWS Glue Data Catalog** or **Hive Metastore**
- Pushes computation to a **dedicated Spectrum layer** (thousands of nodes) — scales independently of your Redshift cluster
- Supports: Parquet, ORC, JSON, CSV, Avro, Grok, Ion, and more
- **You must have a running Redshift cluster** to use Spectrum (the leader node coordinates Spectrum queries)
- Spectrum can read data from S3 that is **encrypted with SSE-S3, SSE-KMS, or CSE-KMS**
- **Partitioning** in S3 (e.g., `s3://bucket/year=2024/month=01/`) dramatically improves Spectrum performance
- Spectrum is billed per **TB of data scanned** (like Athena) — columnar formats reduce cost

### Concurrency Scaling
- Automatically adds **transient clusters** (burst capacity) when query queues build up
- Queries routed to concurrency scaling clusters are **read queries** (SELECT) and also **write queries** (INSERT, DELETE, UPDATE, COPY, UNLOAD — since 2023)
- **Free credits**: Each cluster earns **1 hour of free concurrency scaling credits per day** (per 24-hour period)
- Beyond free credits, billed **per-second** at on-demand compute rates
- Enabled at the **WLM (Workload Management) queue level** — you enable it per queue, not globally
- Concurrency scaling clusters have **full access** to your cluster's data (current, consistent snapshot)
- **Exam tip**: Concurrency scaling = handle **unpredictable spikes** in read/write workloads without over-provisioning

### Workload Management (WLM)
- Controls how queries are routed and prioritized
- **Automatic WLM** (recommended): Redshift manages memory allocation and concurrency — up to **8 queues**
- **Manual WLM**: You define queues, concurrency levels, memory allocation — up to **8 queues** (+ 1 superuser queue always exists)
- **Short Query Acceleration (SQA)**: Automatically detects short-running queries and runs them in a dedicated fast lane (bypassing queued long-running queries)
- **Query Priority**: Within Automatic WLM, set queue priority: HIGHEST, HIGH, NORMAL, LOW, LOWEST
- **Queue hopping / QMR (Query Monitoring Rules)**: Define rules (e.g., if query runs > 5 minutes, abort or move to different queue)

### Redshift Data Sharing
- Share **live, transactionally consistent data** between Redshift clusters **without copying**
- Works across clusters in the **same or different AWS accounts**, same or different Regions
- Requires **RA3 node types** (not DC2)
- **Producer cluster** creates a datashare → **Consumer cluster** reads from it
- Consumers see **real-time data** — no ETL needed
- **Cross-Region data sharing** supported (producer and consumer in different Regions)
- **Exam tip**: Data sharing + RA3 = the modern alternative to ETL pipelines between data warehouses

### Redshift Serverless
- Run analytics **without managing clusters**
- Automatically provisions and scales compute capacity (measured in **RPUs — Redshift Processing Units**)
- Billed per **RPU-hour** (charged per second) — only when queries are running
- Base capacity: 8-512 RPUs (default 128 RPUs)
- Supports Spectrum, data sharing, Federated Query
- **Exam tip**: Use Redshift Serverless for **intermittent/unpredictable** analytics workloads; provisioned clusters for **steady-state**

### Redshift Federated Query
- Query data in **live operational databases** (RDS PostgreSQL, Aurora PostgreSQL, RDS MySQL, Aurora MySQL) directly from Redshift
- No need to ETL data into Redshift
- Creates **external schemas** pointing to the operational database
- Best for: joining operational data with warehouse data for enrichment
- **Exam tip**: Different from Spectrum — Federated Query targets databases, Spectrum targets S3

### Key Limits & Quotas

| Limit | Value |
|-------|-------|
| Max nodes per cluster | 128 (RA3) |
| Max databases per cluster | 60 |
| Max schemas per database | 9,900 |
| Max tables per cluster | 100,000 (DC2), 100,000 (RA3) |
| Max concurrent user connections | 500 |
| Max concurrency per WLM queue (manual) | 50 |
| Max WLM queues | 8 + 1 superuser |
| Snapshot retention | 1-35 days (automated), manual = indefinite |
| Single SQL statement max | 16 MB |
| COPY single file max | Unlimited (but 1 MB – 1 GB chunks recommended) |
| Max columns per table | 1,600 |
| Leader node only functions | Current-session functions, certain system tables |

---

## 2. Design Patterns & Best Practices

### When to Use Redshift

| Use Redshift When | Don't Use Redshift When |
|---|---|
| Petabyte-scale analytics / data warehousing | OLTP workloads (use RDS/Aurora) |
| Complex joins across large datasets | Simple key-value lookups (use DynamoDB) |
| Predictable, heavy analytical workloads | Ad-hoc, infrequent queries on S3 data only (use Athena) |
| Need sub-second query performance on warehoused data | Real-time streaming analytics (use Kinesis Analytics/Flink) |
| BI dashboards with complex aggregations | Unstructured data / ML workloads (use EMR/SageMaker) |
| Data lakehouse pattern (Spectrum + local tables) | < 100 GB datasets (overkill — use RDS + Athena) |

### Anti-Patterns
- **Small datasets**: Redshift has overhead; for datasets < 100 GB, Athena or RDS is more cost-effective
- **OLTP workloads**: Redshift is columnar — row-level inserts/updates are expensive
- **Unstructured data processing**: Use EMR, Glue, or SageMaker
- **BLOB storage**: Redshift doesn't store binary large objects; use S3

### Data Lakehouse Pattern (Exam Favorite)
```
S3 Data Lake ──── Glue Catalog ──── Redshift Spectrum (query in place)
                                     │
                        Redshift Local Tables (hot/frequent data)
```
- Store all raw data in S3 (data lake)
- Use Spectrum for ad-hoc queries on cold/historical data
- Load frequently accessed data into Redshift local tables for performance
- Single query can join local tables + Spectrum external tables

### Distribution Styles (Performance Critical)

| Style | How It Works | When to Use |
|-------|-------------|-------------|
| **KEY** | Rows distributed by hash of chosen column | Large fact tables joined to dimension tables on that key |
| **EVEN** | Round-robin across slices | No obvious join key; want balanced distribution |
| **ALL** | Full copy on every node | Small dimension tables (< 1M rows) joined frequently |
| **AUTO** | Redshift chooses (starts ALL → switches to EVEN/KEY) | Default; good for most tables |

### Sort Keys
- **Compound Sort Key**: Columns sorted in order defined — good for queries that filter on leading columns (like a composite index)
- **Interleaved Sort Key**: Equal weight to each column — good for ad-hoc queries filtering on any column combination (higher maintenance cost via VACUUM)
- **Auto Sort Key**: Redshift automatically determines the sort key — default recommendation
- **Exam tip**: Compound sort key = queries always filter on same leading column(s); Interleaved = varied filter patterns

### Data Loading Best Practices
- **COPY command** is the fastest way to load data (parallel from S3, DynamoDB, EMR, remote hosts)
- Split files into **multiples of the number of slices** for maximum parallelism
- Use **compressed, columnar formats** (Parquet, ORC) for Spectrum
- Use **manifest files** to control exactly which S3 files to load
- **Avoid single large files** — split into 1 MB – 1 GB chunks
- **UNLOAD** to export data from Redshift to S3 (parallel, compressed)

### Integration Points

| Service | Integration |
|---------|-------------|
| **S3** | COPY/UNLOAD, Spectrum external tables, RMS storage (RA3) |
| **Glue** | Data Catalog for Spectrum, ETL jobs to load Redshift |
| **Athena** | Both query S3 via Glue Catalog; complement each other |
| **QuickSight** | BI visualization directly connected to Redshift |
| **Lake Formation** | Fine-grained access control for Spectrum tables |
| **Kinesis Data Firehose** | Stream data directly into Redshift (via S3 staging + COPY) |
| **DynamoDB** | COPY from DynamoDB tables into Redshift |
| **SageMaker** | Export data for ML training; use Redshift ML for in-database ML |
| **CloudWatch** | Performance metrics, alarms |
| **SNS** | Event notifications (e.g., cluster events) |
| **Secrets Manager** | Manage Redshift credentials |
| **IAM** | Cluster IAM roles for S3/Glue/DynamoDB access |

---

## 3. Security & Compliance

### Network Security
- **VPC-based**: Clusters launch in a VPC (private by default)
- **Enhanced VPC Routing**: Forces all COPY/UNLOAD traffic through VPC (enables VPC security features: Security Groups, NACLs, VPC Endpoints, VPC Flow Logs)
- Without Enhanced VPC Routing, COPY/UNLOAD traffic goes over the internet
- **Exam trap**: Enhanced VPC Routing is **NOT enabled by default** — you must enable it
- **Publicly accessible** option can be toggled (default: not publicly accessible)
- Access via **VPC Endpoints** (Gateway endpoint for S3, Interface endpoints for Redshift API)

### Authentication & Authorization
- **IAM roles** attached to the cluster for accessing S3, Glue, DynamoDB, etc.
- **Database users/groups** with GRANT/REVOKE SQL permissions
- **IAM authentication**: Generate temporary database credentials using IAM (GetClusterCredentials API)
- **Federated access**: SAML 2.0 / OIDC federation → IAM role → temporary Redshift credentials
- **Lake Formation integration**: Fine-grained column/row-level security for Spectrum tables
- **Redshift role-based access control (RBAC)**: CREATE ROLE, GRANT ROLE for simplified management

### Encryption
- **At rest**:
  - **KMS** (AWS-managed or customer-managed CMK) — AES-256
  - **CloudHSM** — for clusters requiring HSM-managed keys
  - Encryption is set at **cluster creation** — to encrypt an existing unencrypted cluster, you must **create a new encrypted cluster and migrate data** (snapshot → restore encrypted is also an option now)
  - 4-tier key hierarchy: master key → cluster encryption key (CEK) → database encryption key (DEK) → data blocks
- **In transit**: SSL/TLS connections enforced via `require_ssl` parameter in parameter group
- **Spectrum**: Can read SSE-S3, SSE-KMS, CSE-KMS encrypted S3 data

### Logging & Auditing
- **Audit logging**: Connection logs, user activity logs, user logs → delivered to **S3**
- **CloudTrail**: API-level activity (CreateCluster, ModifyCluster, etc.)
- **System tables**: STL_QUERY, STL_WLM_QUERY, SVL_QUERY_REPORT for query-level auditing
- **CloudWatch metrics**: CPU, disk space, read/write latency, query performance

### Compliance
- PCI DSS, HIPAA eligible, SOC 1/2/3, ISO, FedRAMP
- **Column-level access control**: GRANT SELECT (col1, col2) ON table TO user
- **Row-level security (RLS)**: CREATE RLS POLICY to filter rows per user/role
- **Dynamic Data Masking**: Mask sensitive columns based on user privileges (e.g., show last 4 digits of SSN)

---

## 4. Cost Optimization

### Pricing Models

| Component | Pricing |
|-----------|---------|
| **On-Demand** | Per-node, per-hour (leader node free for multi-node clusters) |
| **Reserved Instances** | 1-year or 3-year commitment; up to **75% savings** |
| **Spectrum** | $5 per TB of data scanned |
| **Concurrency Scaling** | Per-second at on-demand rates (1 hr/day free) |
| **Redshift Serverless** | Per RPU-hour (charged per second) |
| **RMS (RA3)** | Managed storage: per GB-month (currently ~$0.024/GB-month) |
| **Snapshots** | Automated snapshots: free up to cluster storage; manual/cross-region: per GB-month |

### Cost-Saving Strategies
1. **Reserved Instances**: For steady-state workloads — up to 75% off
2. **Pause/Resume clusters**: For dev/test clusters not used 24/7 (no compute charges while paused — still pay for storage)
3. **RA3 + Managed Storage**: Only pay for storage you use (vs. DC2 where you pay for provisioned SSD)
4. **Concurrency Scaling free credits**: 1 hr/day free — often enough for periodic bursts
5. **Spectrum for cold data**: Keep historical data in S3, query with Spectrum at $5/TB instead of storing in Redshift
6. **Columnar formats for Spectrum**: Parquet/ORC reduce data scanned → lower Spectrum costs
7. **Redshift Serverless**: For intermittent workloads — no cost when idle
8. **Elastic resize**: Add/remove nodes quickly (minutes) to match workload — better than over-provisioning
9. **Data compression**: ENCODE on columns — Redshift recommends optimal encoding via ANALYZE COMPRESSION
10. **Snapshot management**: Delete old manual snapshots; set appropriate retention for automated ones

### Cost Traps
- **DC2 over-provisioning**: You pay for all provisioned SSD even if unused — switch to RA3
- **Spectrum on unpartitioned, uncompressed CSV**: Full table scan = maximum cost — use Parquet + partitioning
- **Leaving clusters running 24/7 for dev/test**: Use pause/resume or Serverless
- **Cross-Region snapshot copies**: Storage costs in the destination Region
- **Single-node clusters**: Leader node IS the compute node — you still pay for it

### Redshift vs. Athena (Cost Comparison)

| Factor | Redshift | Athena |
|--------|----------|--------|
| Pricing model | Per-node-hour (or RPU-hour serverless) | Per TB scanned |
| Best for | Heavy, repeated, complex queries | Ad-hoc, infrequent queries |
| Break-even | Cheaper at high query volumes | Cheaper at low query volumes |
| Data must be in | Redshift tables (or S3 via Spectrum) | S3 only |
| Performance | Sub-second on local tables | Seconds to minutes |

---

## 5. High Availability, Disaster Recovery & Resilience

### High Availability
- Redshift clusters run in a **single AZ** (not multi-AZ by default)
- **Multi-AZ deployment** (GA since 2023): Active cluster in one AZ, standby in another — **automatic failover in minutes**
  - Supported only for **RA3 node types**
  - RPO = 0 (shared storage via RMS)
  - RTO = minutes
- **Node recovery**: If a node fails, Redshift automatically provisions a replacement from the most recent snapshot
- **Disk failure**: Data replicated within the cluster; automatic re-replication on failure

### Snapshots & Backup
- **Automated snapshots**: Every 8 hours or every 5 GB of data changes (whichever comes first); retained 1-35 days (default 1 day)
- **Manual snapshots**: User-initiated; retained indefinitely until deleted
- Snapshots stored in **S3** (internally managed — you don't see them in your S3 console)
- **Cross-Region snapshot copy**: Configure automatic copy of snapshots to another Region for DR
  - If source cluster is encrypted with KMS, you must configure a **snapshot copy grant** in the destination Region with a KMS key
- **Restore**: Creates a **new cluster** from snapshot (not in-place restore)
- **Table-level restore**: Restore individual tables from a snapshot to a running cluster

### Disaster Recovery Patterns

| Pattern | RPO | RTO | How |
|---------|-----|-----|-----|
| **Multi-AZ (RA3)** | 0 | Minutes | Automatic failover |
| **Cross-Region snapshots** | Hours (8h default) | 1-2 hours (restore time) | Automated cross-Region copy + restore |
| **Cross-Region data sharing** | Near-zero | Minutes (consumer already running) | RA3 data sharing across Regions |
| **Redshift Serverless + S3** | Near-zero | Minutes | Spectrum reads S3 (replicated with CRR) |

### Elastic Resize vs. Classic Resize

| Feature | Elastic Resize | Classic Resize |
|---------|---------------|----------------|
| Speed | Minutes | Hours |
| Availability | Brief unavailability (connections dropped) | Cluster read-only during resize |
| Node type change | Same node type only | Can change node type |
| Node count | Add/remove nodes (limited — usually 2x/0.5x) | Any node count/type |
| Exam tip | Use for quick scaling | Use for node type migration |

---

## 6. Exam-Focused Section

### Straightforward Questions

**Q1**: A company needs to analyze petabytes of historical data stored in S3 alongside terabytes of frequently queried data in their data warehouse. Which approach minimizes data movement?

**A**: Use Amazon Redshift with **Spectrum** for S3 historical data and **local Redshift tables** for frequently queried data. A single SQL query can join both. No data movement needed for S3 data.

---

**Q2**: A Redshift cluster experiences performance degradation during peak hours when many analysts run concurrent queries. What is the most operationally efficient solution?

**A**: Enable **Concurrency Scaling** on the relevant WLM queues. Redshift automatically adds transient clusters to handle burst read/write workloads. First hour each day is free.

---

**Q3**: A company wants to separate compute and storage costs in their Redshift cluster and avoid paying for unused disk. Which node type should they choose?

**A**: **RA3 nodes** with Redshift Managed Storage (RMS). Compute and storage scale independently; you pay only for storage used (in S3) plus compute nodes.

---

**Q4**: A data team needs to share live, up-to-date data between two Redshift clusters in different AWS accounts without ETL. What feature should they use?

**A**: **Redshift Data Sharing** — the producer cluster creates a datashare; consumer cluster in another account reads live data. Requires **RA3 node types**.

---

**Q5**: An organization must ensure all COPY and UNLOAD traffic from Redshift passes through their VPC (not over the internet) for security compliance. What must they enable?

**A**: Enable **Enhanced VPC Routing** on the Redshift cluster. This forces all COPY/UNLOAD data flow through the VPC, enabling use of VPC Endpoints, NACLs, and Flow Logs.

---

**Q6**: A company runs Redshift for weekly reporting (queries run every Saturday for 4 hours). How can they minimize costs?

**A**: Use **Redshift Serverless** — charges only per RPU-hour when queries are running. For 4 hours/week, this is far cheaper than a provisioned cluster running 24/7 (even with pause/resume).

---

**Q7**: A company needs to join Redshift warehouse data with live transactional data in Aurora PostgreSQL without ETL. What feature should they use?

**A**: **Redshift Federated Query** — create an external schema pointing to the Aurora PostgreSQL database and join with local Redshift tables in a single SQL query.

---

### Tricky / Scenario-Based Questions

**Q1**: A company stores 500 TB in Redshift (DC2 nodes) and queries mostly recent data (last 3 months). Storage costs are growing rapidly but they don't want to lose query access to historical data. What should they do?

**A**: Migrate to **RA3 nodes**. Recent data automatically cached on local SSD; historical data tiers to S3-backed Managed Storage at lower cost. Query performance for recent data stays fast. Alternative: offload historical data to S3 and use Spectrum — but this requires schema changes and application updates.

**Why not just add DC2 nodes?** DC2 couples compute and storage — adding nodes for storage alone wastes compute budget.

**Why not Athena?** Athena requires data in S3. Data is currently in Redshift local tables; migrating 500 TB to S3 is a large effort and loses Redshift's performance for recent queries.

---

**Q2**: A multinational company has Redshift clusters in us-east-1 and eu-west-1. They want analysts in Europe to query US data without replicating the entire dataset. The solution must show real-time data. What should they recommend?

**A**: Use **Cross-Region Data Sharing** (RA3 required). The US cluster creates a datashare; the EU cluster consumes it. Data is live and transactionally consistent — no ETL or replication lag.

**Why not cross-Region snapshot restore?** Snapshots are point-in-time, not real-time. Analysts would see stale data.

**Why not Redshift Spectrum on cross-Region S3?** S3 cross-Region access adds latency, and Spectrum queries S3 — not Redshift local tables. If data is in Redshift local tables, Spectrum can't see it.

---

**Q3**: A company runs Redshift with Manual WLM. Short-running dashboard queries are stuck behind long-running ETL queries, causing timeouts. They want to fix this with minimal configuration changes.

**A**: Enable **Short Query Acceleration (SQA)**. SQA automatically detects short-running queries and runs them in a prioritized space ahead of long-running queries. Minimal configuration — just enable it.

**Why not just add more WLM queues?** That works but requires manual configuration: define queue criteria, assign user groups, tune concurrency — not "minimal configuration."

**Why not Concurrency Scaling?** Concurrency scaling adds capacity for burst workloads, but the problem here is priority, not capacity. Long ETL queries still monopolize slots unless SQA or priority-based WLM redirects short queries.

**Keywords that point to the answer**: "short-running queries stuck behind long-running queries" + "minimal configuration" → **SQA**.

---

**Q4**: A company uses Redshift Spectrum to query 200 TB of CSV files in S3. Spectrum costs are $1,000/TB-scanned per month. They want to reduce Spectrum costs by at least 80% without loading data into Redshift. What should they do?

**A**: Convert CSV files to **Parquet (columnar format)** and **partition** the data in S3 by commonly filtered columns (e.g., year/month/day). Parquet typically reduces data scanned by 90%+ due to columnar pruning. Partitioning eliminates scanning irrelevant partitions entirely.

**Why not just compress CSVs?** Compression (gzip) reduces file size but Spectrum still scans entire rows. Columnar formats allow column pruning — only reading the columns needed.

**Why not switch to Athena?** Athena also charges per TB scanned at $5/TB — same pricing as Spectrum. Format conversion helps Athena too, but switching alone doesn't reduce cost.

**Keywords**: "reduce Spectrum costs" + "without loading into Redshift" → optimize S3 data format (Parquet + partitioning).

---

**Q5**: A startup has unpredictable analytics workloads — some weeks heavy queries, other weeks nothing. They need the full power of Redshift (joins, stored procedures, ML). They want to minimize cost and operational overhead. What should they recommend?

**A**: **Redshift Serverless**. It provides full Redshift SQL capabilities including stored procedures, ML, Spectrum, and Federated Query. Charges per RPU-hour with per-second billing — zero cost when no queries are running. No infrastructure management.

**Why not Athena?** Athena doesn't support stored procedures, full SQL transaction semantics, or Redshift ML. The question specifies "full power of Redshift."

**Why not provisioned Redshift with pause/resume?** Pause/resume has a minimum cluster cost when running and requires manual or scheduled pause — more operational overhead than Serverless.

**Keywords**: "unpredictable workloads" + "full Redshift capabilities" + "minimize cost and operational overhead" → **Redshift Serverless**.

---

**Q6**: A financial services company has an encrypted Redshift cluster in us-east-1 (KMS-encrypted). They need automated DR snapshots in eu-west-1 that they can restore within 2 hours. What must they configure?

**A**: Configure **Cross-Region snapshot copy** with a **snapshot copy grant** in eu-west-1 using a KMS CMK in that Region. Since the source cluster is KMS-encrypted, you MUST create a KMS key in the destination Region and configure the snapshot copy grant — Redshift cannot use the source Region's KMS key.

**Why can't they use the same KMS key?** KMS keys are **Region-specific**. You cannot use a us-east-1 KMS key to encrypt data in eu-west-1.

**Exam trap**: Forgetting the snapshot copy grant for encrypted clusters is a common wrong answer pattern. Without it, cross-Region copy fails silently or returns an error.

**Keywords**: "encrypted" + "cross-Region snapshot" → **snapshot copy grant** with destination-Region KMS key.

---

**Q7**: A company runs two workloads on Redshift: (1) real-time dashboard queries from analysts, and (2) nightly batch ETL jobs. During ETL, dashboard queries slow down significantly. They want to ensure dashboards stay responsive during ETL with minimal cost impact.

**A**: Use **Automatic WLM with query priorities**. Create two WLM queues: dashboard queries with **HIGH priority**, ETL jobs with **LOW priority**. Redshift will allocate more resources to high-priority queries. Additionally, enable **Concurrency Scaling** for the dashboard queue to handle any overflow.

**Why not separate clusters?** Running two clusters doubles cost. WLM + Concurrency Scaling achieves workload isolation on a single cluster.

**Why not just Concurrency Scaling alone?** Concurrency Scaling adds capacity but doesn't prioritize — both workloads compete equally. Priority ensures dashboards get resources first.

**Keywords**: "two workloads on same cluster" + "one must stay responsive" + "minimal cost" → **WLM priority** + optional Concurrency Scaling.

---

### Common Exam Traps & Pitfalls

| Trap | Reality |
|------|---------|
| "Redshift is multi-AZ by default" | Single-AZ by default. Multi-AZ is optional and only for RA3 |
| "Spectrum doesn't need a Redshift cluster" | **It does** — the leader node is required to coordinate Spectrum queries |
| "Athena and Spectrum are the same" | Both query S3, but Spectrum requires a Redshift cluster, supports joins with local tables, and has different performance characteristics |
| "Enhanced VPC Routing is on by default" | **Off by default** — must be explicitly enabled |
| "You can encrypt an existing unencrypted cluster in-place" | You must create a new encrypted cluster and migrate data (or snapshot → restore encrypted) |
| "DS2 nodes are still recommended" | **DS2 is legacy** — RA3 replaces DS2 for all use cases |
| "Concurrency Scaling adds permanent nodes" | Transient clusters — spun up on demand, removed when not needed |
| "Cross-Region snapshots work automatically for encrypted clusters" | Need a **snapshot copy grant** with a KMS key in the destination Region |
| "Redshift Federated Query works with any database" | Only **PostgreSQL and MySQL** (RDS/Aurora) are supported |
| "RA3 data is stored in your S3 bucket" | Stored in **Redshift Managed Storage** (AWS-managed S3) — you don't see or manage these buckets |
| "Data Sharing works with DC2" | **RA3 only** |
| "Single-node cluster has no leader node" | Single-node = leader + compute combined in one node |
| "Redshift Serverless scales to zero" | Scales down to minimum RPUs; you pay nothing when idle (no queries running), but the namespace/endpoint persists |

---

## 7. Cheat Sheet

### Must-Know Facts

- **RA3 = decouple compute + storage** — always the answer for "scale storage independently" or "reduce storage cost"
- **Spectrum = query S3 in place** — needs a running Redshift cluster + Glue Catalog
- **Concurrency Scaling = burst read/write capacity** — 1 free hr/day, enabled per WLM queue
- **Data Sharing = live data between clusters** — RA3 only, cross-account, cross-Region
- **Federated Query = query live RDS/Aurora** — PostgreSQL and MySQL only
- **Serverless = intermittent workloads** — RPU-based billing, full Redshift features
- **Enhanced VPC Routing = COPY/UNLOAD through VPC** — OFF by default
- **SQA = short queries bypass long queries** — minimal config
- **Snapshots restore to a NEW cluster** — not in-place
- **Cross-Region encrypted snapshots need a snapshot copy grant** with KMS key in destination Region
- **Multi-AZ = RA3 only** — RPO 0, RTO minutes

### Key Differentiators

| vs. Service | Redshift Wins When | Other Service Wins When |
|------------|-------------------|----------------------|
| **Athena** | Complex joins, heavy repeated queries, sub-second perf, need stored procedures | Ad-hoc/infrequent queries, no infra management needed |
| **EMR (Hive/Presto)** | SQL-centric warehouse workloads, BI dashboards | ETL/ML on unstructured data, Spark/Hadoop ecosystem |
| **DynamoDB** | Complex analytics, aggregations, joins | Key-value lookups, OLTP, millisecond latency |
| **Aurora** | Analytical queries on large datasets | OLTP, transactional workloads |
| **Redshift Spectrum vs Athena** | Already have Redshift cluster, need to join with local tables | Pure S3 querying, no Redshift cluster |

### Decision Flowchart

```
Question mentions "data warehouse" or "OLAP" or "complex analytics"?
  └─ YES → Think REDSHIFT
       │
       ├─ "Query data in S3 without loading"?
       │    └─ Already have Redshift? → SPECTRUM
       │    └─ No Redshift cluster? → ATHENA
       │
       ├─ "Scale compute and storage independently"?
       │    └─ RA3 NODES
       │
       ├─ "Unpredictable/intermittent workloads"?
       │    └─ REDSHIFT SERVERLESS
       │
       ├─ "Handle query spikes / burst concurrency"?
       │    └─ CONCURRENCY SCALING
       │
       ├─ "Share live data between clusters"?
       │    └─ DATA SHARING (RA3 required)
       │
       ├─ "Join with live RDS/Aurora data"?
       │    └─ FEDERATED QUERY
       │
       ├─ "Short queries stuck behind long queries"?
       │    └─ SQA (Short Query Acceleration)
       │
       ├─ "Ensure COPY/UNLOAD stays in VPC"?
       │    └─ ENHANCED VPC ROUTING
       │
       ├─ "DR across Regions with encrypted cluster"?
       │    └─ Cross-Region snapshots + SNAPSHOT COPY GRANT
       │
       └─ "Reduce Spectrum costs"?
            └─ PARQUET + PARTITIONING in S3
```

### Keyword → Service Mapping

| Exam Keyword/Phrase | Think... |
|--------------------|----------|
| "petabyte-scale analytics" | Redshift |
| "data warehouse" | Redshift |
| "columnar storage" | Redshift |
| "query S3 data from Redshift" | Spectrum |
| "data lakehouse" | Redshift + Spectrum |
| "burst query capacity" | Concurrency Scaling |
| "separate compute and storage" | RA3 nodes |
| "managed storage" | RA3 / RMS |
| "share live data between clusters" | Data Sharing |
| "query RDS from Redshift" | Federated Query |
| "intermittent analytics" | Redshift Serverless |
| "COPY through VPC" | Enhanced VPC Routing |
| "cross-Region encrypted snapshot" | Snapshot copy grant |
| "short queries slow because of ETL" | SQA |
| "distribution key" or "sort key" | Redshift performance tuning |
| "Parquet + partitioning" | Spectrum cost optimization |
