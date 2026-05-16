# AWS SAP-C02 Study Notes: OpenSearch (formerly Elasticsearch)

---

## 1. Core Concepts & Theory

### What is Amazon OpenSearch Service?

**Definition:** A managed service for deploying, operating, and scaling OpenSearch (and legacy Elasticsearch) clusters for real-time search, log analytics, application monitoring, and observability workloads. Includes OpenSearch Dashboards (formerly Kibana) for visualization.

**History:** Amazon Elasticsearch Service was renamed to Amazon OpenSearch Service in 2021 when Elasticsearch changed its license. OpenSearch is the AWS-maintained open-source fork of Elasticsearch 7.10 + Kibana (now called OpenSearch Dashboards).

**Key use cases:**
- Log analytics (centralized logging — CloudWatch, CloudTrail, VPC Flow Logs)
- Full-text search (e-commerce product search, document search)
- Application Performance Monitoring (APM) with traces
- Security analytics (SIEM — Security Information and Event Management)
- Clickstream analytics
- Real-time application monitoring and dashboards

---

### Architecture

```
[OpenSearch Domain (Cluster)]
├── Data Nodes (store data + execute queries)
│   ├── Instance types: r6g, r5, m6g, m5, c6g, i3, etc.
│   ├── Storage: EBS (gp3, gp2, io1, io2) or instance store (i3)
│   └── Shards (primary + replica)
├── Dedicated Master Nodes (cluster coordination)
│   ├── Manage cluster state, shard allocation, index metadata
│   ├── Do NOT hold data or serve queries
│   └── Recommended: 3 dedicated masters for production
├── UltraWarm Nodes (warm storage tier)
│   ├── Cost-effective for infrequently accessed data
│   ├── Uses S3-backed storage with local caching
│   └── Read-only (no indexing to warm directly)
└── Cold Storage (cheapest tier)
    ├── S3-based, detached from cluster compute
    ├── Must be "re-attached" (moved to warm) to query
    └── Lowest cost for archival/compliance data
```

**Key terminology:**

| Term | Description |
|------|-------------|
| **Domain** | An OpenSearch cluster (nodes + config + endpoint) |
| **Index** | A collection of documents (like a database table) |
| **Document** | A JSON record (like a row) |
| **Shard** | A partition of an index; distributed across nodes |
| **Primary shard** | The original data partition |
| **Replica shard** | A copy of a primary shard (HA + read throughput) |
| **Dedicated master node** | Manages cluster state (no data) |
| **Data node** | Stores data and handles queries |
| **UltraWarm node** | Warm storage (S3-backed, read-only for old data) |

---

### Deployment Options

| Option | Description | Use Case |
|--------|-------------|----------|
| **Managed Cluster (provisioned)** | You choose instance types, count, storage | Full control, predictable workloads |
| **OpenSearch Serverless** | No cluster management; auto-scales capacity | "Least operational overhead," variable workloads |

---

### OpenSearch Serverless

**What it is:** A serverless deployment option where you don't manage nodes, shards, or scaling. You create a **collection** (equivalent to a domain) and OpenSearch automatically provisions and scales compute/storage.

**Collection types:**
| Type | Use Case | Characteristics |
|------|----------|----------------|
| **Search** | Full-text search applications | Optimized for low-latency queries |
| **Time series** | Log analytics, metrics, traces | Optimized for append-heavy, time-based data |
| **Vector search** | ML embeddings, semantic search, RAG | Optimized for k-NN vector operations |

**Key characteristics:**
- **OCU (OpenSearch Compute Units):** Billing unit; auto-scales based on workload
- **Minimum 4 OCUs** (2 indexing + 2 search) when active
- Automatically replicates data across AZs
- Separate scaling for indexing and search
- Uses **encryption, network, and data access policies** (not IAM resource policies like managed clusters)

**Serverless security model:**
- **Encryption policy:** Which KMS key encrypts the collection
- **Network policy:** VPC or public access; which VPCs/endpoints can connect
- **Data access policy:** Who can read/write which indexes (JSON policy documents, IAM principal-based)

---

### Storage Tiers (Managed Clusters)

```
[Hot]  → Data nodes (EBS/instance store) — active indexing + querying
  ↓ (ISM policy: migrate after N days)
[Warm] → UltraWarm nodes (S3-backed + local SSD cache) — read-only, lower cost
  ↓ (ISM policy: migrate after N days)
[Cold] → Cold storage (S3 only, detached) — lowest cost, must re-attach to query
```

| Tier | Storage Backend | Queryable? | Indexable? | Cost | Latency |
|------|----------------|-----------|-----------|------|---------|
| **Hot** | EBS / instance store | Yes (fastest) | Yes | Highest | Lowest |
| **UltraWarm** | S3 + local SSD cache | Yes (slightly slower) | No (read-only) | ~80% cheaper than hot | Moderate |
| **Cold** | S3 (detached) | No (must move to warm first) | No | ~90% cheaper than hot | High (reattach time) |

**Index State Management (ISM):** Automated lifecycle policies that move indexes between tiers, delete old indexes, or trigger snapshots based on age, size, or doc count.

---

### Multi-AZ Deployment

- **Zone awareness:** Distributes data nodes and replica shards across 2 or 3 AZs
- **3-AZ deployment (recommended for production):** Primary and replica shards spread across 3 AZs → survives one full AZ failure
- **2-AZ deployment:** Acceptable but losing one AZ puts cluster at reduced capacity
- **Dedicated master nodes:** Automatically placed across AZs (3 masters across 3 AZs)

**Minimum production configuration:**
- 3 AZs
- 3 dedicated master nodes
- At least 2 data nodes per AZ (6 total minimum)
- Replica count ≥ 1

---

### Ingestion Patterns

| Source | Ingestion Method | Use Case |
|--------|-----------------|----------|
| CloudWatch Logs | Subscription filter → Lambda → OpenSearch | Centralized log analytics |
| Kinesis Data Firehose | Direct delivery to OpenSearch | High-volume streaming ingestion |
| Kinesis Data Streams | Lambda consumer or Firehose → OpenSearch | Real-time event processing |
| Logstash | Self-managed on EC2 | Complex log parsing/enrichment |
| Fluent Bit / Fluentd | Agent on EC2/containers | Lightweight log collection |
| AWS IoT | IoT Rule → OpenSearch | IoT telemetry |
| DynamoDB Streams | Lambda → OpenSearch | Search over DynamoDB data |
| S3 | Lambda (S3 event) → OpenSearch | Batch/historical data indexing |
| OpenSearch Ingestion | Managed pipeline (replaces Logstash) | Serverless ingestion, no infrastructure |
| CloudTrail | Direct integration or via Firehose | Audit log analytics |

**Amazon OpenSearch Ingestion:**
- Fully managed, serverless data pipeline
- Based on Data Prepper (open-source)
- Supports transformations, enrichment, routing
- Sources: S3, Kinesis, HTTP, OTel
- Replaces self-managed Logstash/Fluent Bit

---

### Key Limits & Quotas

| Resource | Limit |
|----------|-------|
| Data nodes per domain | 80 (can request increase) |
| Dedicated master nodes | 3 or 5 |
| Domains per account per Region | 100 |
| EBS volume size per node | 3 TB (gp3), 35 TB (io1/io2) for certain instance types |
| UltraWarm storage per domain | 3 PB |
| Shard size (recommended) | 10-50 GB per shard |
| Shards per data node (recommended) | ≤25 shards per GB of JVM heap |
| Index name length | 80 characters |
| Bulk request size | 100 MB |
| Serverless: OCU minimum | 4 OCUs (2 index + 2 search) when collection has data |

---

## 2. Design Patterns & Best Practices

### When to Use OpenSearch

**Use OpenSearch when:**
- Full-text search across large document sets (e-commerce, knowledge bases)
- Log analytics / centralized logging (ELK/EFK stack equivalent)
- Real-time dashboards and visualizations (OpenSearch Dashboards)
- SIEM / security analytics (GuardDuty, CloudTrail analysis)
- Application performance monitoring (APM with traces)
- Vector search / semantic search (ML embeddings, RAG for GenAI)
- Clickstream / behavioral analytics

**Don't use OpenSearch when:**
- Need ACID transactions → use RDS/DynamoDB
- Primary data store (OpenSearch is a secondary/search index)
- Simple key-value lookups → DynamoDB or ElastiCache
- SQL analytics on S3 → Athena or Redshift
- Need strict relational integrity → RDS/Aurora

### Anti-Patterns

| Anti-Pattern | Why It's Bad | Fix |
|--------------|-------------|-----|
| **Using as primary database** | Not designed for transactions; no ACID; potential data loss | Use as secondary search index; source of truth in RDS/DynamoDB |
| **Too many small shards** | Cluster overhead; poor performance | Target 10-50 GB per shard |
| **Too few large shards** | Can't distribute evenly; slow recovery | More shards = better parallelism (within reason) |
| **No dedicated masters in production** | Data nodes handle cluster management → instability under load | Always use 3 dedicated masters for production |
| **Single AZ deployment** | Single point of failure | Enable zone awareness (3 AZ preferred) |
| **No replica shards** | Data loss if node fails | At least 1 replica for HA |
| **Indexing directly without buffering** | Bulk API overwhelmed; indexing errors | Buffer with Kinesis Firehose or SQS + Lambda (batch) |
| **Not using ISM policies** | Indexes grow indefinitely; costs explode | Automate hot→warm→cold→delete lifecycle |
| **Sending all logs everywhere** | Massive ingestion cost; noise | Filter at source; only ingest actionable logs |

### Architectural Patterns

**Centralized Logging:**
```
[EC2/ECS/Lambda/CloudWatch/CloudTrail]
          ↓ (Firehose / Subscription Filters)
[OpenSearch Domain]
          ↓
[OpenSearch Dashboards (Kibana)]
          → Alerting → SNS → Ops Team
```

**Search Augmentation (CQRS):**
```
[Application] → writes → [DynamoDB (source of truth)]
                              ↓ (DynamoDB Streams)
                         [Lambda]
                              ↓
                    [OpenSearch (search index)]
                              ↑
[Application] → search queries → [OpenSearch]
```

**SIEM / Security Analytics:**
```
[GuardDuty findings]  ─┐
[CloudTrail logs]     ─┤→ [Firehose] → [OpenSearch Security Analytics]
[VPC Flow Logs]       ─┤                        ↓
[WAF logs]            ─┘               [Dashboards + Alerting]
```

**Vector Search (RAG for GenAI):**
```
[Documents] → [Embedding Model (Bedrock/SageMaker)] → [OpenSearch k-NN index]
                                                              ↑
[User Query] → [Embedding] → [k-NN search] → [Top-K results] → [LLM (Bedrock)] → [Answer]
```

### Shard Strategy

**Sizing rules:**
- Target **10-50 GB per shard** (sweet spot for performance)
- **Shards per node:** ≤25 shards per GB of JVM heap (e.g., 32 GB heap → max ~800 shards on that node)
- **Number of primary shards:** Data size / 30 GB (rough starting point)
- **Replicas:** At least 1 for HA; 2 for critical data (3-AZ deployment)

**Formula:**
```
Total shards = Primary shards × (1 + Replica count)
Storage needed = Data size × (1 + Replica count) × 1.45 (overhead factor)
```

**Example:** 1 TB of data, 1 replica:
- Primary shards: 1000 GB / 30 GB = ~33 primary shards
- Total shards: 33 × 2 = 66
- Storage: 1000 × 2 × 1.45 = 2,900 GB EBS needed across nodes

### Well-Architected Alignment

| Pillar | Guidance |
|--------|----------|
| **Reliability** | 3-AZ zone awareness; 3 dedicated masters; replicas; automated snapshots to S3 |
| **Security** | VPC deployment; fine-grained access control (FGAC); encryption at rest (KMS) + in transit (TLS); SAML/Cognito auth |
| **Cost** | UltraWarm + Cold for old data; ISM policies; right-size instances; Serverless for variable workloads |
| **Performance** | Right shard sizing; gp3/io2 for hot tier; instance store (i3) for highest IOPS; dedicated masters to offload coordination |
| **Operational Excellence** | ISM policies for lifecycle; OpenSearch Ingestion for managed pipelines; slow log monitoring; CloudWatch alarms on cluster health |
| **Sustainability** | Graviton instances (r6g, m6g, c6g); lifecycle old data to cold/delete; avoid over-replication |

---

## 3. Security & Compliance

### Network Security

**VPC deployment (recommended):**
- Place domain in private subnets (2 or 3 AZs)
- Access via VPC endpoint (interface endpoint) or within VPC
- No public endpoint — access only from within VPC or via VPN/Direct Connect
- Security groups control inbound traffic to the domain endpoint

**Public access:**
- Domain gets a public endpoint
- Use IP-based access policies or IAM-based signing (Signature V4) to restrict access
- **Not recommended** for production

**VPC endpoint access (OpenSearch Serverless):**
- Create VPC endpoints to access serverless collections from within a VPC
- Network policies control which VPCs/CIDR ranges can connect

### Authentication & Authorization

**Fine-Grained Access Control (FGAC):**
The recommended security model — provides user-level, index-level, document-level, and field-level security.

| Feature | Description |
|---------|-------------|
| **Internal user database** | Users managed within OpenSearch (username/password) |
| **SAML authentication** | Federated login via SAML IdP (Okta, Azure AD, etc.) |
| **Amazon Cognito** | User pool authentication for Dashboards |
| **IAM role mapping** | Map IAM roles/users to OpenSearch backend roles |
| **Index-level permissions** | Control access per index (read, write, manage) |
| **Document-level security** | Filter documents based on user attributes |
| **Field-level security** | Hide specific fields from certain users |
| **Field masking** | Show masked version of sensitive fields (e.g., `***@email.com`) |

**Access policies (resource-based):**
- JSON policy attached to the domain
- Can use IAM principals, IP addresses, or VPC endpoint conditions
- Works alongside FGAC (both must allow access)

### Encryption

| Layer | Implementation |
|-------|----------------|
| **At rest** | KMS encryption (AWS-managed key or CMK); covers data, automated snapshots, logs, swap files |
| **In transit** | TLS 1.2 enforced for all node-to-node communication and client connections |
| **Snapshot encryption** | Inherits domain encryption settings |

**Important:** Encryption at rest can only be enabled at domain creation time — cannot be added later (must create a new domain and migrate).

### Auditing

- **Audit logs:** Fine-grained access control generates audit logs (who accessed what index, when)
- **CloudTrail:** All OpenSearch configuration API calls logged
- **Slow logs:** Index slow logs (slow indexing operations) and search slow logs (slow queries) → CloudWatch Logs
- **Error logs:** Application-level errors → CloudWatch Logs
- **CloudWatch Metrics:** Cluster health, free storage, CPU, JVM memory pressure, indexing rate

### Compliance

- HIPAA eligible
- PCI DSS compliant
- SOC 1/2/3
- FedRAMP
- ISO 27001

---

## 4. Cost Optimization

### Pricing Model (Managed Clusters)

| Component | Pricing Basis |
|-----------|---------------|
| Data node instances | Per instance-hour (On-Demand or Reserved) |
| EBS storage | Per GB-month (gp3, io1, io2) |
| UltraWarm instances | Per instance-hour (much cheaper than data nodes) |
| UltraWarm storage | Per GB-month (S3 pricing) |
| Cold storage | Per GB-month (cheapest — S3 Glacier-like pricing) |
| Dedicated master nodes | Per instance-hour |
| Data transfer | Standard AWS transfer pricing |
| Automated snapshots | Free (stored in S3, no charge) |
| Manual snapshots | S3 storage pricing |

### Pricing Model (Serverless)

| Component | Pricing Basis |
|-----------|---------------|
| Indexing OCUs | Per OCU-hour (~$0.24/OCU-hour) |
| Search OCUs | Per OCU-hour (~$0.24/OCU-hour) |
| Managed storage | Per GB-month |
| Minimum cost | 4 OCUs when active (can scale to 0 when idle with no active collections) |

### Cost Optimization Strategies

| Strategy | Impact |
|----------|--------|
| **UltraWarm for old data** | 80% cheaper than hot storage for data >7-14 days old |
| **Cold storage for archival** | 90% cheaper than hot; for compliance/audit data rarely queried |
| **ISM policies** | Automate tier transitions; delete old indexes beyond retention |
| **Reserved Instances** | Up to 52% off for 1-year or 3-year commitment (steady-state clusters) |
| **Right-size instances** | Monitor JVM memory pressure; CPU usage; scale horizontally rather than vertically |
| **Graviton instances** | r6g, m6g, c6g — ~20% cheaper, similar or better performance |
| **gp3 over gp2** | Lower cost + configurable IOPS/throughput |
| **Serverless for variable workloads** | Auto-scales; no idle cluster cost between bursts |
| **Filter at source** | Don't ingest all logs — filter/sample before OpenSearch |
| **Reduce replica count** | For non-critical indexes, use 1 replica instead of 2 |
| **Compression** | Use `best_compression` codec for infrequently accessed indexes (smaller storage) |
| **Rollup jobs** | Summarize old detailed data into aggregated rollup indexes (much smaller) |

### Cost Traps

- **Over-provisioned cluster idle 24/7:** If queries are sporadic, consider Serverless
- **All data in hot tier forever:** Massive EBS costs; use ISM → UltraWarm → Cold → Delete
- **Too many replicas:** Each replica doubles storage cost; 1 replica is sufficient for most cases
- **Serverless minimum (4 OCUs):** If collection always has data, you always pay for minimum 4 OCUs (~$700/month)
- **Not using gp3:** gp2 is more expensive and less flexible than gp3
- **Large dedicated master instances:** Masters don't hold data; oversized masters waste money (c6g.large is often sufficient)
- **Ingesting debug-level logs:** Exponentially more data → exponentially more cost

---

## 5. High Availability, Disaster Recovery & Resilience

### HA Architecture

**Production best practices:**
- **3 Availability Zones** with zone awareness enabled
- **3 dedicated master nodes** (one per AZ)
- **At least 1 replica per index** (distributed across AZs)
- **Even number of data nodes across AZs** (e.g., 6 nodes = 2 per AZ)

**Failure scenarios:**

| Failure | Impact | Mitigation |
|---------|--------|-----------|
| Single node failure | Replicas serve reads; cluster rebalances shards | Replicas ≥ 1; multi-node cluster |
| Single AZ failure | Nodes in that AZ lost; replicas in other AZs serve data | 3-AZ deployment; replicas in each AZ |
| Dedicated master failure | One of 3 masters lost; remaining 2 form quorum | 3 masters across 3 AZs |
| EBS volume failure | Node loses data | Replicas; automated snapshots; EBS is already multi-AZ replicated |

### Snapshots & Backup

**Automated snapshots:**
- Taken daily by default (stored in S3 managed by AWS)
- Retention: 14 days (configurable up to 1 year)
- Free of charge
- Can't control destination S3 bucket
- Used for cluster recovery (same Region)

**Manual snapshots:**
- You control destination S3 bucket
- Can be used for cross-Region DR (copy bucket to another Region)
- Charged at S3 storage rates
- Can restore to a different domain or Region

### DR Strategies

| Strategy | Implementation | RPO | RTO |
|----------|----------------|-----|-----|
| **Snapshot + Restore** | Manual snapshots to S3 → S3 CRR → Restore in DR Region | Hours | Hours |
| **Cross-cluster replication** | Real-time async replication to a follower domain in another Region | Seconds | Minutes |
| **Dual-ingest** | Ingest data to domains in both Regions simultaneously | Near-zero | Near-zero (Route 53 failover) |

**Cross-Cluster Replication (CCR):**
- **Leader-follower model:** Primary domain (leader) replicates indexes to secondary domain (follower)
- Follower indexes are **read-only**
- Asynchronous — slight replication lag
- Cross-Region supported
- Use for: DR, geographic read locality, separation of ingestion from querying

### Cluster Health

| Status | Meaning |
|--------|---------|
| **Green** | All primary and replica shards allocated |
| **Yellow** | All primaries allocated; some replicas not allocated (cluster functional but not fully redundant) |
| **Red** | Some primary shards not allocated (data loss possible; queries may fail) |

**Monitoring:**
- `ClusterStatus.green/yellow/red` — CloudWatch metric
- `FreeStorageSpace` — critical; if storage fills, cluster goes read-only
- `JVMMemoryPressure` — above 80% indicates need for more nodes
- `CPUUtilization` — sustained >80% indicates under-provisioned compute
- `MasterReachableFromNode` — masters healthy
- `AutomatedSnapshotFailure` — backup health

---

## 6. Exam-Focused Section

### Straightforward Questions (7 examples)

**Q1:** A company needs real-time full-text search across millions of product listings for their e-commerce site. Which AWS service should they use?
**A:** **Amazon OpenSearch Service.** Purpose-built for full-text search with relevance scoring, faceting, and autocomplete capabilities.

**Q2:** A company ingests application logs at 10 GB/hour into OpenSearch. After 30 days, logs are rarely queried but must be retained for 1 year for compliance. How should they manage storage costs?
**A:** Use **ISM policies** to automatically migrate indexes older than 30 days to **UltraWarm** (80% cheaper). Migrate indexes older than 90 days to **Cold storage** (90% cheaper). Delete after 365 days.

**Q3:** What is the recommended number of dedicated master nodes for a production OpenSearch cluster?
**A:** **3 dedicated master nodes** distributed across 3 AZs. This ensures quorum (2 of 3) during failures and prevents split-brain.

**Q4:** A company wants to use OpenSearch without managing cluster infrastructure. Workloads are unpredictable (sometimes heavy, sometimes no queries for days). What should they use?
**A:** **OpenSearch Serverless.** Auto-scales capacity; no cluster management; scales to zero when idle (no OCU charges when collection is dormant).

**Q5:** What is the recommended shard size range for optimal OpenSearch performance?
**A:** **10-50 GB per shard.** Too small = excessive overhead; too large = slow recovery and unbalanced distribution.

**Q6:** How can a company stream CloudWatch Logs into OpenSearch in near-real-time?
**A:** Use a **CloudWatch Logs subscription filter** with a **Lambda function** that indexes documents into OpenSearch. Alternatively, route via **Kinesis Data Firehose** for buffered delivery.

**Q7:** A company needs to replicate their OpenSearch domain to another Region for DR. Which feature supports this?
**A:** **Cross-cluster replication (CCR).** Set up the DR domain as a follower of the primary (leader) domain for asynchronous replication.

---

### Tricky / Scenario-Based Questions (7 examples)

**Q1:** A company uses OpenSearch for log analytics. Their cluster status is **Yellow**. All queries return results correctly. Should they be concerned?

**A:** Yellow means all **primary shards are allocated** (data is intact and queryable) but some **replica shards are unallocated**. The cluster is functional but NOT fully redundant — if a node fails now, data could be lost. Investigate: likely insufficient nodes to allocate all replicas (add nodes or reduce replica count for non-critical indexes).

*Why tricky:* Candidates may panic at non-green status. Yellow = functional but degraded HA. Red = actual data at risk.

**Keywords:** "Yellow status" + "queries work" → replicas not allocated, not urgent but should fix

---

**Q2:** A security team ingests VPC Flow Logs, CloudTrail, and GuardDuty findings into OpenSearch for their SIEM. During a security incident, they need to search 6 months of historical logs. However, only the last 7 days are in OpenSearch (older data was deleted). How should they redesign for long-term searchable retention at minimal cost?

**A:** Implement the **hot-warm-cold** tiered storage architecture:
1. **Hot (7 days):** Active indexing and querying on data nodes
2. **UltraWarm (7-90 days):** Queryable, read-only, 80% cheaper
3. **Cold (90-365 days):** S3-backed, re-attach when needed for incident investigation
4. Use **ISM policies** to automate transitions

Additionally, send raw logs to **S3** (via Firehose) for long-term archive beyond cold storage (query with Athena if OpenSearch cold is insufficient).

*Why wrong:*
- Keep everything in hot: Extremely expensive for 6 months of data
- Delete after 7 days + use Athena on S3: Loses near-instant search capability for recent data
- Larger cluster: More expensive; doesn't solve the retention design

**Keywords:** "6 months searchable" + "minimal cost" → UltraWarm + Cold storage tiers + ISM policies

---

**Q3:** A company has a DynamoDB table with 50 million products. They need sub-second full-text search with fuzzy matching, autocomplete, and faceted filtering (by category, price range, brand). DynamoDB alone can't support these query patterns. What architecture should they use?

**A:** **DynamoDB + OpenSearch (CQRS pattern):**
1. DynamoDB = source of truth (writes)
2. **DynamoDB Streams** → **Lambda** → indexes documents into OpenSearch
3. Application reads: full-text search queries go to OpenSearch; transactional reads go to DynamoDB
4. OpenSearch provides fuzzy matching, autocomplete (edge-ngram), faceted search

*Why wrong:*
- DynamoDB with GSI: No full-text search, no fuzzy matching, no facets
- RDS with FULLTEXT index: Doesn't scale to 50M documents at sub-second latency
- CloudSearch: Legacy service, less capable than OpenSearch; AWS pushes OpenSearch

**Keywords:** "full-text search" + "fuzzy" + "facets" + "autocomplete" → OpenSearch (with DynamoDB as source via Streams)

---

**Q4:** A company creates an OpenSearch domain without encryption at rest enabled. They now need to enable encryption for compliance. What should they do?

**A:** **Create a new domain with encryption enabled** and migrate data (reindex or restore from snapshot). Encryption at rest **cannot be enabled on an existing domain** after creation — it must be configured at domain creation time.

*Why tricky:* Candidates assume encryption can be toggled like other settings. It cannot — this is a common exam trap.

**Migration approach:**
1. Create new domain with encryption enabled (KMS CMK)
2. Take snapshot of old domain
3. Restore snapshot to new domain
4. Update application endpoints to new domain
5. Delete old domain

---

**Q5:** A company runs OpenSearch Serverless for their search application. They notice that even during zero-traffic periods (weekends), they're billed ~$700/month (4 OCUs). The application has no traffic on weekends. How should they reduce costs?

**A:** OpenSearch Serverless maintains a **minimum of 4 OCUs** when the collection exists with data. Options to reduce cost:
1. If acceptable, use a **provisioned managed cluster** with Reserved Instances (may be cheaper for predictable workloads)
2. For truly zero-traffic collections, the serverless model isn't ideal — the minimum OCU applies regardless
3. If the workload is bursty during weekdays but zero on weekends, the minimum cost is a trade-off vs. managing a cluster

*Why tricky:* "Serverless" implies "pay nothing when idle" — but OpenSearch Serverless has a minimum OCU charge unlike Lambda.

---

**Q6:** A company uses OpenSearch for log analytics with Kinesis Data Firehose delivering logs. During a traffic spike, OpenSearch returns HTTP 429 (Too Many Requests) errors to Firehose. Logs are being lost. How should they fix this?

**A:** **Firehose has built-in retry + S3 backup for failed records.** Ensure:
1. Configure Firehose **S3 backup** for failed records (all failed documents go to S3 for replay)
2. **Scale OpenSearch** — add data nodes or scale to larger instances to handle the ingestion spike
3. Enable **managed scaling** or increase cluster capacity for peak
4. Consider **Firehose buffering** settings (increase buffer interval to batch more efficiently)

*Why wrong:*
- Increase Firehose throughput limit: Doesn't fix the OpenSearch bottleneck
- Drop logs during spike: Unacceptable for production logging
- Switch to direct API ingestion: Loses Firehose's buffering/retry/backup benefits

**Keywords:** "429 errors" + "logs lost" → S3 backup for failures + scale OpenSearch + Firehose buffering

---

**Q7:** A company wants to migrate from a self-managed Elasticsearch 7.10 cluster on EC2 to Amazon OpenSearch Service. They have 5 TB of data in 200 indexes with custom plugins. What's the migration approach?

**A:**
1. **Compatibility check:** OpenSearch is fork of ES 7.10 — API compatible. Custom plugins may need OpenSearch equivalents.
2. **Snapshot-based migration:**
   - Take snapshot of self-managed cluster → S3
   - Create OpenSearch domain (compatible version)
   - Restore snapshot from S3 to OpenSearch domain
3. **Dual-write during cutover:** Point ingestion to both old and new clusters during validation
4. **Testing:** Validate query behavior, custom plugins, and performance on OpenSearch
5. **Cutover:** Switch application to OpenSearch endpoint; decommission old cluster

*Why wrong:*
- Reindex all data: Unnecessary if versions are compatible (snapshot/restore is faster)
- DMS: Doesn't support Elasticsearch/OpenSearch as source/target
- In-place upgrade: Can't "upgrade" self-managed EC2 to AWS managed service

---

### Common Exam Traps & Pitfalls

1. **OpenSearch ≠ primary database** — it's a secondary search/analytics engine. Source of truth should be RDS/DynamoDB/S3.

2. **Encryption at rest = creation-time only** — cannot be added to existing domains. Must create new domain and migrate.

3. **3 dedicated master nodes always** — questions testing HA always expect dedicated masters. Without them, data nodes handle cluster management and become unstable under load.

4. **Yellow ≠ crisis; Red = crisis** — Yellow is degraded HA; Red is data loss risk.

5. **UltraWarm = read-only** — you cannot index new data directly to UltraWarm. Data must go to hot first, then migrate.

6. **OpenSearch Serverless minimum OCUs** — not truly zero-cost when idle (4 OCU minimum ~$700/month with active data).

7. **Cross-cluster replication followers are read-only** — can't write to the follower domain; it's a one-way sync.

8. **Shard count is set at index creation** — can't change primary shard count later (must reindex). Choose wisely.

9. **OpenSearch vs. CloudSearch** — CloudSearch is legacy (limited features, not recommended). Exam always prefers OpenSearch.

10. **Instance store (i3) vs. EBS** — i3 instances use local NVMe (highest performance) but data is lost if instance stops. Only use with replicas + frequent snapshots.

---

## 7. Cheat Sheet

### Must-Know Facts

- **OpenSearch** = managed Elasticsearch fork (full-text search + log analytics + dashboards)
- **OpenSearch Dashboards** = Kibana equivalent (visualization)
- **3 dedicated masters + 3 AZs + replicas** = production minimum
- **UltraWarm** = S3-backed warm tier, read-only, ~80% cheaper than hot
- **Cold storage** = S3 detached, must re-attach to query, ~90% cheaper
- **ISM policies** = automated lifecycle (hot → warm → cold → delete)
- **FGAC** = fine-grained access control (index/document/field-level)
- **Encryption at rest = creation-time only** (cannot add later)
- **Cross-cluster replication (CCR)** = async leader-follower for DR / read locality
- **OpenSearch Serverless** = no cluster management; OCU-based; min 4 OCUs when active
- **OpenSearch Ingestion** = managed pipeline (replaces Logstash)
- **Recommended shard size** = 10-50 GB

### Decision Flowchart

```
"Full-text search" OR "log analytics" OR "SIEM"
  → Amazon OpenSearch Service

"Least operational overhead" + "variable search workload"
  → OpenSearch Serverless

"Full control" + "predictable workload" + "cost-optimize with RIs"
  → OpenSearch managed cluster (provisioned)

"Reduce storage cost for old logs"
  → UltraWarm + Cold storage + ISM policies

"DR for OpenSearch across Regions"
  → Cross-cluster replication (CCR)

"Stream logs to OpenSearch"
  → Kinesis Data Firehose (buffered, retries, S3 backup)

"Search over DynamoDB data"
  → DynamoDB Streams → Lambda → OpenSearch (CQRS)

"Vector search for GenAI/RAG"
  → OpenSearch Serverless (vector search collection) or managed with k-NN plugin
```

### Key Differentiators

| Signal in Question | Answer |
|---|---|
| "Full-text search" / "fuzzy" / "autocomplete" | OpenSearch |
| "Log analytics" / "centralized logging" | OpenSearch + Firehose |
| "SIEM" / "security analytics" | OpenSearch Security Analytics |
| "Dashboards" / "visualizations" for ops | OpenSearch Dashboards |
| "Warm storage" / "tiered storage for logs" | UltraWarm + Cold |
| "Index lifecycle" / "automate retention" | ISM policies |
| "Document-level security" / "field-level masking" | FGAC (Fine-Grained Access Control) |
| "Cross-Region replication" for search | Cross-cluster replication (CCR) |
| "Serverless search" / "no cluster to manage" | OpenSearch Serverless |
| "Vector embeddings" / "semantic search" / "k-NN" | OpenSearch (vector search) |
| "Simple SQL on S3" | Athena (NOT OpenSearch) |
| "Data warehouse" | Redshift (NOT OpenSearch) |

---

## 8. Additional Deep-Dive

### 10 More Tricky Scenario-Based Questions

**Q1:** A media company indexes article content for search. Their index grows by 50 GB/day. After 1 year, their 20-node cluster is constantly at 85% storage capacity, queries are slow, and costs are high. They need articles searchable for 90 days and archived (not searchable) beyond that. What architecture changes?

**A:**
1. Enable **UltraWarm** and create **ISM policy**: hot (14 days) → warm (14-90 days) → delete from OpenSearch
2. For archive beyond 90 days: ingest to **S3 via Firehose** concurrently (raw logs always go to S3)
3. This reduces hot tier from 365 days to 14 days of data → drastically smaller/cheaper cluster
4. UltraWarm handles 14-90 day range at 80% lower cost
5. After 90 days, data only exists in S3 (queryable via Athena if needed)

*Result:* Hot tier goes from ~18 TB to ~700 GB. Cluster can be 3-5 nodes instead of 20.

---

**Q2:** A company has an OpenSearch domain in a VPC. A third-party SaaS application (outside AWS) needs to ingest data into the domain. The domain has no public endpoint. How should they enable access securely?

**A:** Options:
1. **OpenSearch Ingestion pipeline** (managed, serverless) with a public HTTP endpoint — it writes to the VPC domain internally
2. **API Gateway + Lambda** (Lambda in VPC) — third-party calls API Gateway; Lambda indexes to OpenSearch
3. **Kinesis Data Firehose** (VPC delivery) — third-party writes to Firehose; Firehose delivers to VPC domain
4. **Do NOT** make the domain public — keep it in VPC and use one of the above as a secure proxy

*Why wrong:*
- Make domain public: Violates security best practice
- VPN to third-party: Complex, expensive, and gives too much access
- Direct Connect to SaaS provider: Overkill for data ingestion

---

**Q3:** A fintech company uses OpenSearch for fraud detection. They need to search transactions where amount > $10,000 AND the merchant is in a watch list AND the transaction happened in the last hour. Results must return in < 200ms across 500M documents. How should they optimize?

**A:**
1. **Index design:** Use time-based indexes (daily rollover) with aliases — searching "last hour" only hits the current index
2. **Shard routing:** Route by customer_id to minimize cross-shard queries for per-customer fraud checks
3. **Hot tier with SSD storage** (gp3 with high IOPS or i3 instance store) for sub-second performance
4. **Denormalize watch list** into transaction documents at ingest (avoid joins — OpenSearch has no efficient joins)
5. **Adequate replica count** for read parallelism on query-heavy workloads
6. **Filter context** (not query context) for exact-match fields (merchant watchlist = filter; amount > threshold = filter) — filters are cached

*Key insight:* "Last hour" + time-based indexes means OpenSearch only searches one small index, not all 500M docs.

---

**Q4:** A company uses OpenSearch with 3 data nodes (r5.2xlarge, 500 GB EBS each). The cluster status goes **Red** after one node fails during maintenance. Why, and how should they prevent this?

**A:** **Why Red:** With 3 nodes and replica count of 1, some shards have primary + replica on only 2 of 3 nodes. If the node hosting a primary shard fails and that shard's replica was on the same node (or another failed node), the primary becomes unallocated → Red.

**Fix:**
1. **Zone awareness with 3 AZs** — ensures replicas are in different AZs (different physical failure domains)
2. **More data nodes** (at least 2 per AZ = 6 minimum) — more places to allocate shards
3. **Dedicated master nodes** — prevent data nodes from being overwhelmed with cluster management during recovery
4. Avoid having shards that can only fit on specific nodes (right-size shards)

---

**Q5:** A company uses OpenSearch for application search AND log analytics on the same cluster. During log ingestion spikes, search query latency degrades from 50ms to 5 seconds. How should they fix this?

**A:** **Separate workloads:**
1. **Separate clusters:** One for application search (optimized for query performance), one for log analytics (optimized for ingestion throughput)
2. If same cluster: Use **index-level shard allocation filtering** to pin search indexes to dedicated "search" nodes and log indexes to "ingest" nodes (node tagging)
3. Use **search node types** (Serverless has separate search/indexing OCUs by design)

*Why tricky:* Mixing search and analytics on one cluster is common but leads to resource contention. The exam tests whether you recognize workload isolation is needed.

---

**Q6:** A company enables Fine-Grained Access Control (FGAC) on their OpenSearch domain. After enabling, all existing Lambda functions that write to OpenSearch receive HTTP 403 errors. What's wrong?

**A:** After enabling FGAC, all requests must be **authenticated**. The Lambda functions were previously using the open access policy (or IP-based policy). Now they need:
1. **IAM role mapping:** Map the Lambda execution role to an OpenSearch backend role with write permissions
2. Lambda must **sign requests with SigV4** (AWS SDK handles this if using the AWS OpenSearch client)
3. Alternatively: create an internal user for Lambda and authenticate with credentials (less recommended)

*Why tricky:* Enabling FGAC changes the access model fundamentally. Existing unsigned requests are rejected.

---

**Q7:** A company wants to use OpenSearch for vector search (k-NN) to power a RAG application with Amazon Bedrock. They have 10 million document embeddings (1536 dimensions each). Which OpenSearch deployment and configuration is optimal?

**A:**
- **OpenSearch Serverless** with **vector search collection type** (purpose-built for k-NN, auto-scales, simplest to manage)
- OR **Managed cluster** with k-NN plugin enabled on memory-optimized instances (r6g.2xlarge+)

Configuration for managed cluster:
- Use **HNSW algorithm** (best balance of speed + accuracy for high dimensions)
- Instances with high memory (vectors reside in memory for fast search)
- Dedicated nodes for vector indexes (isolate from other workloads)
- `ef_search` and `ef_construction` parameters tuned for latency vs. accuracy trade-off

**Sizing:** 10M vectors × 1536 dims × 4 bytes = ~60 GB just for vectors (plus graph overhead ~2x) = ~120-180 GB memory needed

---

**Q8:** A logging pipeline uses Kinesis Data Firehose → OpenSearch. The company notices duplicate documents in OpenSearch. Why does this happen and how should they fix it?

**A:** **Why duplicates:** Firehose uses at-least-once delivery. If OpenSearch returns a timeout (but actually indexed the document), Firehose retries → duplicate. Also, Lambda retries in the pipeline can cause duplicates.

**Fix:**
1. Use **document IDs** in OpenSearch — index with a deterministic `_id` (e.g., hash of log content + timestamp). Re-indexing the same `_id` overwrites instead of creating a duplicate.
2. If using Lambda transformation in Firehose: ensure idempotent processing
3. Accept minor duplicates for log analytics (many teams consider this acceptable for logs)

---

**Q9:** A company runs OpenSearch in a VPC with private subnets. Developers need to access OpenSearch Dashboards for debugging. They currently SSH tunnel through a bastion host, which is cumbersome. What's a better approach?

**A:** Options:
1. **Amazon Cognito + Application Load Balancer** — place ALB in front of Dashboards endpoint; Cognito authenticates users; ALB routes to VPC OpenSearch (requires public ALB or VPN access to ALB)
2. **AWS Client VPN** — developers connect via VPN; access Dashboards directly in VPC
3. **Nginx reverse proxy** on a public-facing EC2 (with Cognito or OAuth) — proxies to private OpenSearch
4. **Systems Manager Port Forwarding** — `aws ssm start-session --document-name AWS-StartPortForwardingSessionToRemoteHost` → tunnel to OpenSearch from local machine

*Best for exam:* Cognito authentication for Dashboards is the AWS-recommended approach for managed access to OpenSearch Dashboards in VPC.

---

**Q10:** A company uses OpenSearch for e-commerce product search. They want to A/B test a new relevance ranking algorithm without affecting all users. How should they implement this?

**A:**
1. Create a **second index** with the new mapping/settings/ranking configuration
2. Use an **index alias** that points to the production index
3. For A/B testing: application logic routes a percentage of queries to the new index (or use a second alias)
4. Alternatively: use OpenSearch **search pipeline** feature to apply different ranking processors per request (query parameter-based)
5. Compare relevance metrics (click-through rate, conversion) between the two

*Why wrong:*
- Modify production index: Affects all users; can't compare
- Blue-green at cluster level: Overkill for testing relevance changes
- A/B at CDN level: CDN doesn't understand search relevance routing

---

### Compare: OpenSearch vs. CloudSearch vs. Athena vs. Redshift

| Dimension | OpenSearch | CloudSearch | Athena | Redshift |
|-----------|-----------|------------|--------|----------|
| **Primary use** | Full-text search + log analytics | Full-text search (simple) | Ad-hoc SQL on S3 | Data warehousing |
| **Data model** | JSON documents (inverted index) | Structured documents | S3 files (Parquet, CSV, JSON) | Columnar tables |
| **Query language** | OpenSearch DSL + SQL | CloudSearch API | ANSI SQL (Presto/Trino) | PostgreSQL SQL |
| **Real-time ingestion** | Yes (sub-second) | Minutes (batched) | No (query at rest) | Yes (Streaming ingestion) |
| **Log analytics** | Yes (primary use case) | No | Possible but not optimized | Possible but not primary |
| **Full-text search** | Excellent (fuzzy, facets, autocomplete) | Good (basic) | Poor (LIKE only) | Poor |
| **Scalability** | Petabytes | Limited | Unlimited (S3) | Petabytes |
| **Managed?** | Yes | Yes | Serverless | Yes (+ Serverless option) |
| **Exam preference** | Always over CloudSearch | Legacy (avoid) | SQL on S3 | BI/warehouse |

### When Does the Exam Expect OpenSearch vs. Alternatives?

**Pick OpenSearch when:**
- "Full-text search" / "fuzzy matching" / "autocomplete" / "relevance scoring"
- "Log analytics" / "centralized logging" / "ELK stack"
- "SIEM" / "security analytics"
- "Real-time dashboards" for ops data
- "Near-real-time" ingestion + search
- "DynamoDB + search" (CQRS pattern)
- "Vector search" / "k-NN" / "semantic search"

**Pick Athena when:**
- "SQL queries on S3"
- "Ad-hoc analysis"
- "No infrastructure" + SQL-only
- Data already in S3, query infrequently

**Pick Redshift when:**
- "Data warehouse" / "BI" / "QuickSight"
- "Complex SQL joins and aggregations"
- "Sub-second dashboard queries"
- Structured data + business reporting

**Pick CloudSearch when:**
- **Never on the modern exam** — always choose OpenSearch over CloudSearch

### All "Gotcha" Differences

| Gotcha | Detail |
|--------|--------|
| Encryption at rest = creation-time only | Cannot add to existing domain; must create new + migrate |
| UltraWarm is read-only | Cannot index directly to warm; data must flow from hot |
| Cold must be re-attached to query | Can't search cold data directly; move to warm first |
| Serverless minimum = 4 OCUs | Not zero-cost when idle if collection has data |
| Cross-cluster replication = read-only followers | Can't write to follower indexes |
| Primary shard count is immutable | Set at index creation; to change, must reindex |
| i3 instance store = volatile | Data lost on instance stop; MUST have replicas + snapshots |
| FGAC changes access model | Existing unsigned requests will fail after enabling |
| OpenSearch ≠ primary DB | No ACID, no transactions; use as secondary search index |
| Domain name is immutable | Can't rename a domain; must create new + migrate |
| Cluster health Yellow = degraded HA | Functional but not redundant — fix before next failure |
| VPC domain = no public access | Must use VPN, proxy, or Cognito for external access to Dashboards |

### Decision Tree for Search & Analytics Services

```
START: What's the primary requirement?
├── Full-text search (fuzzy, facets, autocomplete)?
│   └── OpenSearch (managed or Serverless)
├── Log analytics / centralized logging?
│   └── OpenSearch + Firehose ingestion
├── SQL queries on S3 data lake?
│   └── Athena
├── BI / data warehouse / complex SQL?
│   └── Redshift
├── Real-time streaming analytics?
│   └── OpenSearch (near-real-time) or Managed Flink (true real-time)
├── Vector/semantic search for GenAI?
│   └── OpenSearch Serverless (vector search collection)
├── Key-value lookup (no search)?
│   └── DynamoDB or ElastiCache (NOT OpenSearch)
└── Need search + transactional source of truth?
    └── DynamoDB (source) + OpenSearch (search) via Streams
```

---

*End of study notes for OpenSearch (formerly Elasticsearch)*
