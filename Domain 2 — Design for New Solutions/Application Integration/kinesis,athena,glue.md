# AWS Data Streaming & Analytics: Kinesis, Athena, Glue

## 1. Core Concepts & Theory

---

### Amazon Kinesis

**What it is:** A family of services for collecting, processing, and analyzing real-time streaming data at any scale.

#### Kinesis Data Streams (KDS)

**What it is:** Real-time data streaming service. You manage consumers, shards, and retention.

**Architecture:**
```
Producers → Kinesis Data Stream (Shards) → Consumers
```

**Key Characteristics:**
- **Real-time:** ~200ms latency (Enhanced Fan-Out) to ~2 second (shared throughput)
- **Retention:** 24 hours (default), up to 365 days
- **Immutable data:** Once written, data cannot be deleted (must expire)
- **Ordering:** Per-shard ordering guaranteed (partition key determines shard)
- **Replay:** Consumers can re-read data within retention period

**Shards:**
- Unit of capacity
- **Write:** 1 MB/sec or 1,000 records/sec per shard
- **Read:** 2 MB/sec per shard (shared across all consumers of that shard)
- **Enhanced Fan-Out:** 2 MB/sec per shard **per consumer** (dedicated throughput)
- Scale by adding/removing shards (resharding)

**Capacity Modes:**
- **Provisioned:** You specify shard count, pay per shard-hour ($0.015/shard-hour)
- **On-Demand:** Auto-scales, pay per GB of data written/read ($0.08/GB written, $0.04/GB read)
  - Default capacity: 4 MB/sec write (scales up to 200 MB/sec)
  - No shard management needed

**Partition Key:**
- Determines which shard receives a record (MD5 hash → shard range)
- Same partition key → same shard → ordering guaranteed
- **Hot shard:** Poor partition key distribution causes one shard to get disproportionate traffic

**Producers:**
- AWS SDK (PutRecord, PutRecords)
- Kinesis Producer Library (KPL): Batching, aggregation, retry, higher throughput
- Kinesis Agent: Pre-built app for log files on servers
- Third-party: Fluentd, Kafka Connect, etc.

**Consumers:**
- **Shared Fan-Out (Classic):** All consumers share 2 MB/sec per shard (pull model, GetRecords API)
- **Enhanced Fan-Out (EFO):** Each consumer gets dedicated 2 MB/sec per shard (push model, SubscribeToShard API, HTTP/2)
- Kinesis Client Library (KCL): Manages shard assignments, checkpointing (uses DynamoDB for coordination)
- Lambda (event source mapping)
- Kinesis Data Firehose
- Kinesis Data Analytics

**KCL (Kinesis Client Library):**
- Each shard is processed by exactly one KCL worker
- Uses DynamoDB table for lease coordination and checkpointing
- If KCL workers < shards → one worker processes multiple shards
- If KCL workers > shards → some workers are idle
- **Recommendation:** Number of KCL instances ≤ number of shards

**Quotas & Limits:**
- Max record size: 1 MB
- Max PutRecords batch: 500 records or 5 MB
- Shard limit per account per region: 500 (on-demand) or 200 (provisioned) — soft limit
- GetRecords: 5 calls/sec per shard, up to 10,000 records or 10 MB per call
- Enhanced Fan-Out: Up to 20 consumers per stream
- IteratorAge: Key metric — time between data written and consumer reading

---

#### Amazon Kinesis Data Firehose (now Amazon Data Firehose)

**What it is:** Fully managed service for loading streaming data into destinations. Near-real-time (not real-time). No consumer management needed.

**Key Characteristics:**
- **Near-real-time:** Minimum buffer time 60 seconds (can be 0 with zero-buffering for some destinations)
- **Fully managed:** No shards, no capacity management, auto-scales
- **Delivery guarantee:** At-least-once
- **No data replay:** Data is delivered and gone (no retention like KDS)

**Buffer Settings:**
- **Buffer size:** 1 MB – 128 MB (varies by destination)
- **Buffer interval:** 60 – 900 seconds
- Whichever condition is met first triggers delivery

**Data Transformation:**
- Lambda function for custom transformation (inline)
- Format conversion: JSON → Parquet/ORC (using Glue schema)
- Compression: GZIP, Snappy, Zip (depends on destination)
- Encryption: SSE-KMS

**Destinations:**
- Amazon S3
- Amazon Redshift (via S3 COPY)
- Amazon OpenSearch Service
- Splunk
- HTTP endpoints (custom)
- Third-party: Datadog, New Relic, MongoDB, etc.

**Error Handling:**
- Failed records → S3 backup bucket (configurable)
- Source record backup: Optionally save all raw records to S3

**Sources:**
- Direct PUT (SDK, Agent, CloudWatch Logs, IoT, EventBridge)
- Kinesis Data Streams → Firehose (KDS as source)

---

#### Amazon Kinesis Data Analytics (now Amazon Managed Service for Apache Flink)

**What it is:** Run Apache Flink applications on streaming data for real-time analytics, transformations, and complex event processing.

**Key Characteristics:**
- **Real-time SQL or Apache Flink (Java/Scala/Python)**
- **Sources:** Kinesis Data Streams, Amazon MSK (Kafka)
- **Destinations:** Kinesis Data Streams, Kinesis Data Firehose, Lambda, S3, custom sinks
- **Serverless scaling:** Parallel processing units (KPUs)
- **Checkpointing:** Automatic state management with snapshots

**Use Cases:**
- Real-time dashboards and metrics
- Anomaly detection on streaming data
- Time-series analytics (windowed aggregations)
- Stream enrichment (join streams with reference data)
- ETL on streaming data before loading to data lake

**SQL vs Flink:**
- **SQL (Legacy):** Simple queries on streams, tumbling/sliding windows
- **Apache Flink:** Full programming model, complex event processing, exactly-once semantics

**Key Concept — Windowing:**
- **Tumbling Window:** Fixed-size, non-overlapping (e.g., every 5 minutes)
- **Sliding Window:** Fixed-size, overlapping (e.g., 5-min window sliding every 1 min)
- **Session Window:** Gap-based (groups events with no gap > threshold)

---

#### Amazon Kinesis Video Streams

**What it is:** Fully managed service to stream video from connected devices to AWS for analytics, ML, and playback.

**Key Characteristics:**
- Ingests video, audio, and time-serialized data from cameras/IoT devices
- Stores data durably (configurable retention: 0 hours to indefinite)
- Integrates with Amazon Rekognition Video, SageMaker, custom consumers
- Supports WebRTC for two-way real-time media streaming (live video calls)
- HLS/DASH for playback
- **NOT for file-based video processing** — designed for live streams

**Use Cases:**
- Smart home cameras (Ring, Nest-style)
- Industrial monitoring
- Computer vision ML pipelines
- Two-way video (WebRTC signaling)

---

### Amazon Athena

**What it is:** Serverless interactive query service that analyzes data directly in Amazon S3 using standard SQL. No infrastructure to manage.

**Key Characteristics:**
- **Serverless:** No servers to provision or manage
- **Pay-per-query:** $5 per TB of data scanned
- **Engine:** Presto/Trino-based (also supports Apache Spark)
- **Data stays in S3:** No data movement or loading required
- **Supports:** CSV, JSON, ORC, Parquet, Avro, and more
- **Integrates with AWS Glue Data Catalog** for metadata/schema

#### Performance Optimization (Critical for Cost & Exam)

| Technique | Benefit |
|-----------|---------|
| **Columnar formats (Parquet/ORC)** | Only reads needed columns, huge scan reduction |
| **Compression (Snappy, GZIP, LZ4, ZSTD)** | Less data to scan = lower cost |
| **Partitioning** | Prune entire folders (e.g., `year=2024/month=03/`) |
| **Bucketing** | Further subdivide within partitions |
| **Use larger files (> 128 MB)** | Reduces S3 request overhead |
| **Avoid many small files** | Causes excessive S3 LIST + GET operations |

#### Partitioning
- **Hive-style:** `s3://bucket/table/year=2024/month=03/day=15/`
- Athena reads only relevant partitions based on WHERE clause
- **Partition Projection:** Define partition pattern in table properties — Athena calculates partitions without scanning the catalog (faster at scale)

#### Federated Query
- Query data beyond S3: DynamoDB, RDS, Redshift, CloudWatch Logs, JDBC sources
- Uses Lambda-based data source connectors
- Enables cross-data-source JOINs without ETL

#### Workgroups
- Isolate queries, costs, and settings per team/project
- Set data usage limits (per-query and per-workgroup)
- Separate query history and results locations
- Enforce encryption settings

#### CTAS (Create Table As Select)
- Materialize query results as a new table in S3
- Useful for: converting formats (CSV → Parquet), pre-aggregating data, partitioning

#### Views
- Saved SQL queries (logical, not materialized unless using CTAS)
- Simplify complex queries for reuse

#### Quotas
- Query timeout: 30 minutes
- Max concurrent queries: 20-25 per account (soft limit, can increase)
- Max query string length: 256 KB
- Result size: No hard limit (written to S3)

---

### AWS Glue

**What it is:** Fully managed serverless ETL (Extract, Transform, Load) service with a centralized data catalog.

#### AWS Glue Data Catalog

**What it is:** Central metadata repository — the "Hive Metastore" for AWS analytics.

**Components:**
- **Databases:** Logical grouping of tables
- **Tables:** Schema definitions (columns, data types, location, format, partitions)
- **Connections:** JDBC connections to RDS, Redshift, on-premises databases
- **Partitions:** Metadata about data partitions (e.g., year/month/day)

**Integration:**
- Used by: Athena, Redshift Spectrum, EMR, Kinesis Data Analytics, Glue ETL
- Single source of truth for data schema across analytics services
- Compatible with Apache Hive Metastore

**Key Point:** Glue Data Catalog is NOT a data store — it only stores metadata (schema, location, format) pointing to data in S3, JDBC, etc.

#### Crawlers

**What they do:** Automatically discover data schema and populate the Data Catalog

**How it works:**
1. Point crawler at S3 path, JDBC, or DynamoDB
2. Crawler scans data, infers schema (column names, types, format)
3. Creates/updates table definitions in the Data Catalog
4. Detects partitions automatically

**Schedules:** On-demand, cron, or event-triggered (S3 event → EventBridge → Crawler)

**Classifiers:** Built-in (CSV, JSON, Parquet, Avro, XML) or custom classifiers for non-standard formats

**Key Behavior:**
- Crawlers can merge schemas across files or create new table versions
- Can add new partitions without re-crawling entire dataset
- Uses IAM role to access data sources

#### Glue ETL Jobs

**Job Types:**
- **Spark:** Apache Spark-based (Python/Scala) — default for large-scale ETL
- **Python Shell:** Lightweight Python scripts (no Spark overhead)
- **Ray:** Python distributed computing (for ML-heavy workloads)
- **Streaming ETL:** Continuously processes streaming data from Kinesis/Kafka

**DynamicFrame:**
- Glue's extension of Spark DataFrame
- Handles semi-structured data (inconsistent schemas)
- **ResolveChoice:** Handle schema inconsistencies (cast, make_struct, project)
- **Relationalize:** Flatten nested JSON into relational tables
- Can convert to/from Spark DataFrame

**Job Bookmarks:**
- Track previously processed data (ETL checkpoint)
- On next run, only processes new data (avoids reprocessing)
- Works with S3 (file timestamps), JDBC (sequential column values)
- **Critical for incremental ETL**

**Development Endpoints & Interactive Sessions:**
- Interactive sessions: Serverless Spark environment for development
- Develop and test scripts before deploying as jobs

**DPU (Data Processing Unit):**
- 1 DPU = 4 vCPUs + 16 GB RAM
- Standard: 10 DPUs min (Spark)
- G.1X: 1 DPU per worker (memory-optimized)
- G.2X: 2 DPU per worker (more memory)
- G.025X: 1/4 DPU per worker (Python Shell)
- **Auto-scaling:** Glue can scale workers based on workload

**FindMatches ML Transform:**
- Identify duplicate/matching records across datasets
- Uses ML (no coding needed, but requires labeling)
- Use case: Customer deduplication, record linkage

#### Glue Workflows
- Orchestrate multiple crawlers and ETL jobs
- Visual DAG (directed acyclic graph) of dependencies
- Trigger-based: schedule, on-demand, or conditional

#### Glue Schema Registry
- Schema versioning and validation for streaming data
- Avro and JSON Schema support
- Integrates with Kinesis, MSK, Glue ETL
- **Schema evolution:** Backward, forward, full compatibility modes
- Reduces schema mismatch errors in streaming pipelines

#### Glue DataBrew
- Visual data preparation tool (no-code)
- 250+ built-in transformations (clean, normalize, pivot)
- Profile data quality automatically
- Output to S3 in any format
- **Not ETL at scale** — more for data analysts preparing datasets

#### Glue Elastic Views (Deprecated → Use Zero-ETL)
- Was for replicating data across stores; replaced by Zero-ETL integrations

#### Quotas & Limits
- Max concurrent job runs: 200 per account (soft)
- Max DPUs per job: Configurable (G.2X workers × count)
- Crawler timeout: 48 hours
- Data Catalog: 1 million tables per account
- Trigger limit: 50 per workflow

---

## 2. Design Patterns & Best Practices

### When to Use What

| Scenario | Use |
|----------|-----|
| Real-time streaming with custom consumers | Kinesis Data Streams |
| Load streaming data to S3/Redshift/OpenSearch | Kinesis Data Firehose |
| Real-time analytics on streams (SQL/Flink) | Kinesis Data Analytics / Managed Flink |
| Live video ingestion and analytics | Kinesis Video Streams |
| Ad-hoc SQL queries on S3 data | Athena |
| Scheduled/batch ETL to transform data | Glue ETL Jobs |
| Discover schema of data in S3 | Glue Crawlers |
| Central metadata for multiple analytics services | Glue Data Catalog |
| Data preparation (no-code) | Glue DataBrew |
| Convert CSV to Parquet at scale | Glue ETL or Athena CTAS |

### Kinesis Streams vs Firehose — The Critical Decision

| Factor | Kinesis Data Streams | Kinesis Data Firehose |
|--------|---------------------|----------------------|
| Latency | Real-time (~200ms EFO, ~2s classic) | Near-real-time (60s+ buffer) |
| Management | You manage shards/consumers | Fully managed, no capacity planning |
| Replay | Yes (within retention) | No |
| Custom consumers | Yes (KCL, Lambda, Flink) | No (fixed destinations) |
| Destinations | Anything your consumer writes to | S3, Redshift, OpenSearch, Splunk, HTTP |
| Ordering | Per-shard (partition key) | No ordering guarantee |
| Transformation | Consumer-side | Built-in Lambda transform |
| Retention | 24h–365 days | None (delivery only) |
| Pricing | Per-shard or on-demand | Per-GB ingested |
| Format conversion | Consumer-side | Built-in (JSON → Parquet/ORC) |

**Decision:** Need custom processing, replay, or <1s latency → KDS. Need simple delivery to a fixed destination → Firehose.

### Anti-Patterns

| Anti-Pattern | Why It's Wrong | Better Approach |
|--------------|---------------|-----------------|
| Athena on many small files (<128 MB) | Excessive S3 overhead, slow, expensive | Compact files with Glue ETL or S3 Object Lambda |
| Athena on unpartitioned data | Full table scan every query | Add partitions (Hive-style or Projection) |
| Athena on CSV/JSON for frequent queries | Reads entire row/file, expensive | Convert to Parquet/ORC |
| Kinesis Firehose for real-time (<1s) | Minimum 60s buffer (0 with specific config) | Kinesis Data Streams + custom consumer |
| Glue ETL for simple S3 → S3 copy | Overkill, expensive | S3 Batch Operations or DataSync |
| Kinesis Streams without monitoring shard utilization | Hot shards cause throttling | Monitor `WriteProvisionedThroughputExceeded`, use good partition keys |
| Using KDS on-demand for predictable stable load | More expensive than provisioned | Use provisioned mode for stable workloads |
| Running Glue Crawlers on every file arrival | Expensive and slow | Use S3 event → Lambda → AddPartition API |

### Well-Architected Framework Alignment

**Reliability:**
- KDS: Multi-AZ data replication, Enhanced Fan-Out for consumer isolation
- Firehose: Auto-retries delivery, S3 backup for failed records
- Athena: Serverless, no infrastructure failure possible
- Glue: Job bookmarks for resumable ETL, retries on failure

**Security:**
- KDS: SSE-KMS, VPC endpoints, IAM policies per stream/consumer
- Firehose: SSE-KMS at rest, TLS in transit, IAM role for delivery
- Athena: Query result encryption (SSE-S3, SSE-KMS, CSE-KMS), workgroup isolation
- Glue: Data Catalog encryption, connection passwords encrypted, job scripts encrypted, VPC for JDBC sources

**Performance:**
- KDS: Scale shards, use Enhanced Fan-Out, optimize partition keys
- Athena: Parquet/ORC, partitioning, CTAS for materialization, workgroup concurrency
- Glue: Scale DPUs, use G.2X workers for memory-intensive, pushdown predicates

**Cost:**
- KDS: On-demand for spiky, provisioned for stable; minimize retention period
- Firehose: Pay per GB — compress data before ingestion
- Athena: Parquet + partitioning = 30-90% cost reduction vs. CSV scan
- Glue: Auto-scaling, job bookmarks (don't reprocess), right-size DPUs

### Integration Patterns

```
┌───────────────────────────────────────────────────────────────────┐
│                  Common Data Pipeline Patterns                      │
├───────────────────────────────────────────────────────────────────┤
│                                                                     │
│  IoT/Clickstream → KDS → Firehose → S3 (Parquet) → Athena        │
│  IoT/Clickstream → KDS → Managed Flink → Real-time dashboard      │
│  CloudWatch Logs → Firehose → S3 → Athena                         │
│  CloudWatch Logs → Firehose → OpenSearch → Kibana                  │
│  S3 (raw) → Glue Crawler → Glue ETL → S3 (curated) → Athena     │
│  S3 (CSV) → Glue ETL → S3 (Parquet, partitioned) → Athena       │
│  KDS → Lambda (transform) → DynamoDB                               │
│  KDS → Firehose → Redshift (COPY via S3)                          │
│  RDS → Glue JDBC Connection → Glue ETL → S3 → Athena             │
│  DynamoDB → Export to S3 → Glue Crawler → Athena                  │
│                                                                     │
└───────────────────────────────────────────────────────────────────┘
```

**Lambda Architecture (Exam Favorite):**
```
                    ┌─→ KDS → Flink → Real-time views (speed layer)
Source → KDS/Firehose ─┤
                    └─→ S3 → Glue ETL → S3 (curated) → Athena (batch layer)
```

**Kappa Architecture (Streaming Only):**
```
Source → KDS → Managed Flink → Serving layer (no batch)
```

---

## 3. Security & Compliance

### Kinesis Security

**KDS Encryption:**
- **At-rest:** Server-side encryption (SSE) with KMS (aws/kinesis or CMK)
- **In-transit:** TLS between producers/consumers and Kinesis endpoints
- Enable via `StartStreamEncryption` API or console

**Access Control:**
- IAM policies: `kinesis:PutRecord`, `kinesis:GetRecords`, `kinesis:DescribeStream`
- **Enhanced Fan-Out consumers:** Register with `RegisterStreamConsumer`, deregister when done
- **VPC Interface Endpoint:** Private access without internet

**KCL DynamoDB Table:**
- Stores shard leases and checkpoints
- Must grant IAM permissions for DynamoDB to KCL consumer role
- Table name matches application name

### Firehose Security
- IAM role for delivery (S3 PutObject, Redshift COPY, OpenSearch index)
- SSE-KMS for data at rest in transit buffer
- Lambda transform function must have appropriate permissions
- VPC endpoint support

### Athena Security

**Encryption:**
- Query results: SSE-S3, SSE-KMS, or CSE-KMS (configured per workgroup)
- Source data encryption: Athena reads SSE-S3, SSE-KMS, CSE-KMS encrypted S3 objects
- **Workgroup enforcement:** Force encryption settings on all queries in a workgroup

**Access Control:**
- IAM: Control who can run queries (`athena:StartQueryExecution`), access workgroups
- **Lake Formation:** Fine-grained access control (column-level, row-level, cell-level)
- **Glue Data Catalog resource policies:** Control cross-account catalog access
- S3 bucket policies: Control access to underlying data

**Workgroup Isolation:**
- Separate teams into different workgroups
- Each workgroup: own query results location, encryption settings, cost limits
- Users can only see queries in their workgroup

### Glue Security

**Data Catalog Encryption:**
- Encrypt entire catalog with KMS
- Connection passwords encrypted
- Enable in Catalog settings (irreversible once enabled)

**Job Security:**
- Jobs run in a Glue-managed VPC (default) or your VPC (for JDBC sources in private subnets)
- Security configurations: S3 encryption, CloudWatch encryption, job bookmark encryption
- IAM role: Least privilege for S3, catalog, JDBC, etc.

**Cross-Account:**
- Glue Data Catalog resource policy allows cross-account access
- Athena in Account A → Glue Catalog in Account B (via resource policy)
- Lake Formation grants for cross-account sharing

### SCP Considerations
- Deny `kinesis:CreateStream` without encryption
- Deny `athena:StartQueryExecution` without specified workgroup
- Enforce Glue catalog encryption at organizational level

---

## 4. Cost Optimization

### Kinesis Data Streams Pricing

**Provisioned Mode:**
- Shard-hour: $0.015/hour ($10.80/month per shard)
- PUT Payload Units: $0.014 per million (25 KB unit)
- Extended retention (>24h): $0.020 per shard-hour
- Long-term retention (>7d): $0.023 per GB stored
- Enhanced Fan-Out: $0.013 per shard-hour per consumer + $0.013/GB data retrieved

**On-Demand Mode:**
- $0.08/GB data written
- $0.04/GB data read
- No shard management
- ~2-5x more expensive than well-utilized provisioned

**Cost Optimization:**
- Right-size shards (monitor `IncomingBytes`/`IncomingRecords` per shard)
- Use provisioned mode for predictable, stable workloads
- Use on-demand for spiky/unpredictable workloads
- Minimize retention period (don't keep 7 days if you only need 24h)
- Use shared fan-out unless consumer isolation is needed (EFO costs extra)
- Aggregate records using KPL (fewer PUT requests)

### Kinesis Firehose Pricing
- Per-GB ingested: $0.029/GB (first 500 TB)
- Format conversion: $0.018/GB
- **No charge for delivery to S3** (but S3 storage costs apply)
- Dynamic partitioning: $0.019/GB

**Cost Optimization:**
- Compress data before sending to Firehose (reduces ingested GB)
- Use format conversion to Parquet (smaller files = cheaper Athena queries later)
- Batch source records to reduce per-record overhead

### Athena Pricing
- **$5.00 per TB of data scanned**
- DDL statements (CREATE, ALTER, DROP): FREE
- Failed queries: Still charged for data scanned before failure
- Cancelled queries: Charged for data scanned before cancellation

**Cost Optimization (HUGE exam topic):**
- **Columnar format (Parquet/ORC):** 30-90% cost reduction (reads only needed columns)
- **Compression:** Further reduces scanned bytes
- **Partitioning:** Skips irrelevant data entirely
- **LIMIT clause:** Does NOT reduce cost (Athena scans first, then limits)
- **SELECT specific columns:** Cheaper than SELECT * (with columnar formats)
- **CTAS:** Pre-aggregate frequently queried data
- **Workgroup quotas:** Set per-query and workgroup-level data scan limits to prevent runaway costs

**Example Cost Comparison:**
- 1 TB CSV, no partitions, SELECT * → $5.00
- Same data as Parquet → ~$0.50-1.50
- Same data as Parquet + partitioned → ~$0.05-0.50

### Glue Pricing
- **Crawler:** $0.44 per DPU-hour (10-second billing increment)
- **ETL Job:** $0.44 per DPU-hour (Spark), $0.44 per DPU-hour (Python Shell at 1/4 DPU)
- **Data Catalog:** Free (first 1M objects), $1/100K objects after
- **Development Endpoints:** $0.44 per DPU-hour (expensive! use Interactive Sessions instead)
- **DataBrew:** $0.48 per session-hour

**Cost Optimization:**
- Use job bookmarks to avoid reprocessing
- Right-size DPUs (start with min, use auto-scaling)
- Avoid Development Endpoints (use Interactive Sessions — charged per session, not per DPU idle)
- Don't over-crawl: use AddPartition API for known partition patterns instead of re-crawling
- Use pushdown predicates to read less data from source
- Compact output files to optimize downstream (Athena) costs

### Cost Traps
- Athena on CSV without partitions = scan everything = expensive
- KDS provisioned shards running 24/7 at low utilization = waste (consider on-demand)
- Glue Development Endpoints left running = burns DPU-hours
- Firehose with unnecessary format conversion when source is already Parquet
- Not setting Athena workgroup quotas → runaway queries scanning petabytes
- KDS extended retention enabled but not needed → extra shard cost
- Glue crawler running hourly on stable data → use event-driven partition updates

---

## 5. High Availability, Disaster Recovery & Resilience

### Kinesis HA
- **KDS:** Multi-AZ by default (data replicated synchronously across 3 AZs)
- **Firehose:** Multi-AZ by default, automatic retry on delivery failure
- **Regional service:** No native cross-region replication
- **Cross-region pattern:** Consumer (Lambda/KCL) in Region A reads and writes to KDS in Region B
- **Data retention as recovery:** Can replay from any point within retention window

**Resilience Patterns:**
- **Checkpointing (KCL):** If consumer fails, new consumer resumes from last checkpoint
- **Iterator age monitoring:** Alarm on `GetRecords.IteratorAgeMilliseconds` to detect falling behind
- **Firehose retry:** Retries delivery for up to 24 hours, backs up failures to S3
- **Hot shard mitigation:** Monitor `WriteProvisionedThroughputExceeded`, add shards or improve partition key

### Athena HA
- **Serverless:** No infrastructure to fail
- **Regional:** Depends on S3 data availability
- **Cross-region:** Query data in S3 cross-region (high latency) or replicate data with S3 CRR
- **Availability:** Service itself is highly available; bottleneck is concurrent query limits

### Glue HA
- **Serverless:** Service handles compute failure internally
- **Job retries:** Configure retry count on failure (0–10)
- **Job bookmarks:** Enable resuming from last processed point
- **Workflow recovery:** Re-run failed stages without re-running successful ones
- **Cross-region DR:** Deploy Glue jobs in secondary region, replicate S3 data with CRR, export/import Data Catalog

### RPO/RTO Considerations

| Service | RPO | RTO | Strategy |
|---------|-----|-----|----------|
| KDS data loss | Retention-period coverage (24h–365d) | Seconds (consumer failover via KCL) | Replay from checkpoint |
| KDS regional failure | Up to last checkpoint | Minutes | Cross-region consumer replication |
| Firehose delivery failure | Zero (retries + S3 backup) | Automatic (up to 24h retry) | S3 error bucket for investigation |
| Athena query failure | N/A (stateless) | Seconds (retry query) | Retry + workgroup failover |
| Glue ETL failure | Last bookmark position | Minutes (job restart) | Job bookmarks + retries |
| Glue Catalog loss | Last backup | Minutes–hours | Catalog export/import, IaC |

---

## 6. Exam-Focused Section

### Straightforward Questions

**Q1:** A company ingests 10 GB/hour of clickstream data that must be stored in S3 in Parquet format for later Athena analysis. They want a fully managed solution with no consumer code. What should they use?

**A:** Kinesis Data Firehose with built-in format conversion (JSON → Parquet using Glue Data Catalog schema) delivering directly to S3.

---

**Q2:** Athena queries on a 5 TB dataset in CSV format cost $25 per full scan. How can the company reduce cost by 90%?

**A:** Convert data to Parquet (columnar format) and partition by commonly filtered columns (e.g., date). Parquet reduces scanned data by 75-90%, and partitioning eliminates scanning irrelevant data entirely.

---

**Q3:** A company has JSON data in S3 and needs to build a data catalog so that both Athena and Redshift Spectrum can query it. What service populates the catalog?

**A:** AWS Glue Crawlers. They scan S3, infer schema, and populate the Glue Data Catalog, which is shared by Athena, Redshift Spectrum, and EMR.

---

**Q4:** A streaming application requires exactly the same records to be processed in order, with the ability to replay data from 3 days ago. Which Kinesis service?

**A:** Kinesis Data Streams with retention set to at least 72 hours. KDS provides per-shard ordering (same partition key = same shard = ordered) and replay within the retention period.

---

**Q5:** A Glue ETL job runs nightly and processes the same data repeatedly, leading to duplicates in the destination. How do you fix this?

**A:** Enable Glue Job Bookmarks. They track previously processed data, ensuring subsequent runs only process new data (incremental ETL).

---

**Q6:** A company needs to run Apache Flink for real-time anomaly detection on data flowing through Kinesis Data Streams. What service runs the Flink application?

**A:** Amazon Managed Service for Apache Flink (formerly Kinesis Data Analytics). It provides a fully managed Flink environment that reads directly from KDS.

---

**Q7:** An analytics team needs to query data across DynamoDB, S3, and an on-premises PostgreSQL database using standard SQL, without moving data. What's the simplest approach?

**A:** Athena Federated Query with Lambda-based data source connectors. Athena can query across multiple sources using pre-built or custom connectors.

---

### Tricky / Scenario-Based Questions

**Q1:** A company uses Kinesis Data Streams with 10 shards and 15 Lambda consumers. Some Lambda invocations process the same records. Why?

**A:** This is a misconception in the question setup. With Lambda event source mapping, each shard gets exactly one concurrent Lambda invocation (parallelization factor = 1 by default). With 10 shards, max 10 concurrent Lambdas are active. If **parallelization factor > 1** is configured (up to 10 per shard), multiple Lambdas process the same shard's data in parallel (but different batches). **Duplicate processing** happens when: (1) Lambda times out and batch retries, (2) checkpoint fails. **Fix:** Make Lambda idempotent, increase timeout, use bisect-on-error to isolate bad records. **Key trap:** More Lambda instances than shards doesn't cause duplicates — Lambda autoscaling is per-shard.

---

**Q2:** A company stores 10 TB of log data partitioned by `year/month/day` in S3. They add a new partition daily. Glue Crawler runs hourly but takes 30 minutes and costs are high. How to optimize?

**A:** **Stop using the crawler for partition discovery.** Since the partition pattern is predictable (`year/month/day`), use one of: (1) **Athena Partition Projection** — defines partition pattern in table DDL, no catalog updates needed, fastest. (2) **`MSCK REPAIR TABLE`** in Athena — adds new partitions. (3) **Lambda triggered by S3 event → `batch_create_partition` Glue API** — adds partitions as data arrives. **Why not crawler:** Crawlers are expensive and slow for predictable schemas. They're best for schema discovery, not ongoing partition maintenance. **Key phrase:** "known partition structure" or "partition pattern is predictable" → don't use crawler.

---

**Q3:** A real-time application uses Kinesis Data Streams. During peak traffic, some producers receive `ProvisionedThroughputExceededException`. The stream has 5 shards and uses `user_id` as the partition key. Monitoring shows one shard at 90% utilization while others are at 20%. What's wrong?

**A:** **Hot shard problem.** A few `user_id` values generate disproportionate traffic, and they hash to the same shard. **Fixes:** (1) Add a random suffix to partition key to spread load (loses per-user ordering). (2) Use a more distributed partition key. (3) Split the hot shard (UpdateShardCount or re-shard). (4) Switch to on-demand capacity mode (auto-scales). **Wrong answer:** "Add more shards" — adding shards helps overall capacity but doesn't fix hot partition. The hash range of the hot shard remains the same unless you specifically split IT. **Key concept:** Partition key distribution determines shard utilization, not just shard count.

---

**Q4:** A company wants to build a data lake. Raw data lands in S3 as JSON. They need to: (1) catalog the schema, (2) convert to Parquet, (3) partition by date, (4) make queryable via Athena. A solutions architect proposes: Glue Crawler → Glue ETL → Athena. Is there a simpler approach?

**A:** For simple format conversion and partitioning, **Athena CTAS (Create Table As Select)** can replace Glue ETL: read JSON, output as Parquet with partitioning, in a single SQL statement. Then either run the Glue Crawler once (or define the table manually) to catalog the Parquet output. **However:** For recurring/scheduled ETL, Glue is better (job bookmarks, scheduling). For a one-time conversion or small-scale recurring, CTAS is simpler. **Key trade-off:** CTAS = simpler for ad-hoc; Glue ETL = better for production pipelines with incremental processing. **Watch for:** "minimize operational overhead for a one-time conversion" → CTAS.

---

**Q5:** A company streams data to Kinesis Data Firehose → S3. They want real-time analytics on the same data with sub-second latency. A developer suggests adding Athena queries on the S3 data. Is this correct?

**A:** **No.** Firehose delivers to S3 with minimum 60-second buffer latency. Athena is a batch query engine (not real-time). **Correct approach:** Use Kinesis Data Streams as the source → feed both Firehose (for S3/Athena batch analytics) AND Managed Service for Apache Flink or Lambda (for real-time sub-second analytics). This is the **Lambda Architecture** pattern. **Key phrases:** "sub-second" or "real-time analytics" + "Athena" = wrong. Athena is for ad-hoc/batch queries, not real-time.

---

**Q6:** A Glue ETL job processes data from an RDS MySQL database in a private subnet. The job fails with connection timeout. What's wrong?

**A:** Glue ETL jobs need network access to the JDBC source. **Fix:** Configure a **Glue Connection** with the VPC, subnet, and security group. The Glue job runs ENIs in that subnet. The security group must allow outbound to RDS, and the RDS security group must allow inbound from Glue's security group. **Also:** The subnet must have a NAT Gateway or VPC endpoints for Glue to access the Glue service (for downloading scripts, logging). **Wrong answer:** "Make RDS public" — violates security best practice. **Key requirement:** Glue in VPC needs: (1) route to source, (2) route to internet/VPC endpoints for service communication.

---

**Q7:** An organization has 500 million records in a Kinesis Data Stream with 100 shards. They want to process each record exactly once and store results in DynamoDB. KCL consumers occasionally produce duplicates during resharding. How do they guarantee exactly-once?

**A:** Kinesis Data Streams provides at-least-once delivery (not exactly-once). **Fix:** Implement **idempotent writes** in the consumer. Use DynamoDB conditional writes (e.g., `attribute_not_exists(record_id)` or compare sequence numbers). Store the KDS sequence number with each record as a deduplication key. **Wrong answer:** "Switch to FIFO SQS" — changes the streaming architecture entirely. **Wrong answer:** "Use Enhanced Fan-Out" — EFO provides dedicated throughput, not exactly-once semantics. **Key concept:** Exactly-once is a consumer responsibility in streaming systems, not a platform guarantee.

---

### Common Exam Traps & Pitfalls

1. **Kinesis Data Streams ≠ Kinesis Data Firehose.** KDS = real-time, you manage consumers, replay possible. Firehose = near-real-time, fully managed delivery, no replay. If the question says "fully managed, no custom code, load to S3" → Firehose.

2. **Athena pricing = data scanned.** LIMIT doesn't reduce cost. Partitioning and columnar formats DO. If the question asks "reduce Athena cost" → Parquet + partitions.

3. **Firehose buffer = minimum 60 seconds.** If the question requires <60s latency → KDS + custom consumer, not Firehose. (Note: zero-buffering exists for some destinations but exam typically tests the 60s minimum.)

4. **Glue Crawler vs. Partition Projection:** Crawlers are expensive and slow for known schemas. If partition pattern is predictable → Partition Projection (Athena) or API-based partition registration.

5. **KDS Enhanced Fan-Out:** Needed when multiple consumers compete for read throughput. Without EFO: 2 MB/s shared per shard. With EFO: 2 MB/s per consumer per shard. "Multiple applications reading same stream with latency issues" → EFO.

6. **Kinesis Data Streams ordering:** Ordering is PER-SHARD only. "Global ordering across all records" is NOT possible with KDS alone (use single shard, which limits throughput).

7. **Glue Data Catalog is NOT Glue ETL.** Catalog = metadata. ETL = processing. Athena uses the Catalog but doesn't need Glue ETL jobs.

8. **Kinesis Data Streams retention ≠ Firehose retention.** KDS retains data (24h-365d). Firehose does NOT retain — it buffers briefly and delivers. No replay with Firehose.

9. **Athena is NOT real-time.** It's a batch query engine. Don't select Athena for real-time dashboards or sub-second analytics. Use Flink or OpenSearch.

10. **KDS On-Demand vs Provisioned:** On-demand auto-scales but costs more per GB. If the question mentions "predictable, steady traffic" → provisioned. "Unpredictable, spiky" → on-demand.

11. **Firehose direct PUT vs KDS → Firehose:** You can PUT directly to Firehose (no KDS needed). Use KDS → Firehose when you also need KDS for other consumers or replay.

12. **Glue Job Bookmarks:** Only work with S3 (file timestamps) and JDBC (sequential primary key/timestamp column). They don't work with arbitrary data sources.

---

## 7. Cheat Sheet

### Must-Know Facts

| Fact | Detail |
|------|--------|
| KDS shard write capacity | 1 MB/sec or 1,000 records/sec |
| KDS shard read capacity | 2 MB/sec (shared), 2 MB/sec per consumer (EFO) |
| KDS max record size | 1 MB |
| KDS default retention | 24 hours (max 365 days) |
| KDS on-demand default write | 4 MB/sec (auto-scales to 200 MB/sec) |
| Firehose buffer minimum | 60 seconds |
| Firehose destinations | S3, Redshift, OpenSearch, Splunk, HTTP, third-party |
| Athena pricing | $5 per TB scanned |
| Athena query timeout | 30 minutes |
| Athena concurrent queries | 20-25 (soft limit) |
| Glue DPU cost | $0.44 per DPU-hour |
| Glue Crawler min billing | 10-second increments |
| Glue max concurrent jobs | 200 (soft) |
| Glue Data Catalog limit | 1 million tables per account |
| KCL coordination | DynamoDB table for leases/checkpoints |
| EFO max consumers | 20 per stream |

### Key Differentiators

| Feature | KDS | Firehose | Managed Flink | Athena | Glue ETL |
|---------|-----|----------|---------------|--------|----------|
| Processing model | Stream (you consume) | Delivery (managed) | Stream analytics | Batch query | Batch ETL |
| Latency | Real-time (ms) | Near-real-time (60s+) | Real-time (ms) | Seconds-minutes | Minutes-hours |
| Managed? | Semi (manage shards or on-demand) | Fully | Fully | Fully | Fully |
| Replay | Yes | No | From source (KDS/Kafka) | Re-query S3 | Re-run job |
| Output | Custom (your consumer) | Fixed destinations | Custom sinks | S3 (query results) | S3, JDBC, Catalog |
| Best for | Custom real-time processing | Simple delivery to stores | Complex stream analytics | Ad-hoc SQL on S3 | Batch transform/load |

### Decision Flowchart

```
Question mentions...                                    → Think...
──────────────────────────────────────────────────────────────────────
"real-time streaming", "custom processing"             → Kinesis Data Streams
"load streaming data to S3/Redshift/OpenSearch"        → Kinesis Data Firehose
"fully managed delivery, no code"                      → Kinesis Data Firehose
"replay streaming data"                                → Kinesis Data Streams (has retention)
"real-time analytics on streams", "Flink"             → Managed Service for Apache Flink
"ad-hoc SQL queries on S3"                            → Athena
"reduce Athena cost"                                   → Parquet + Partitions + Compression
"discover schema in S3"                                → Glue Crawler
"batch ETL, transform at scale"                        → Glue ETL (Spark)
"incremental ETL, don't reprocess"                    → Glue Job Bookmarks
"central metadata, shared catalog"                     → Glue Data Catalog
"convert JSON to Parquet (streaming)"                 → Firehose format conversion
"convert JSON to Parquet (batch)"                     → Glue ETL or Athena CTAS
"multiple consumers on same stream"                   → KDS + Enhanced Fan-Out
"sub-second latency" + "analytics"                    → Managed Flink (NOT Athena)
"hot shard", "throughput exceeded"                    → Fix partition key distribution
"query across S3 + DynamoDB + RDS"                   → Athena Federated Query
"known partition pattern, avoid crawler"              → Partition Projection
"streaming video from cameras"                        → Kinesis Video Streams
"no-code data preparation"                            → Glue DataBrew
```

### Quick Comparison: "When the question says X, the answer is Y"

- "Stream data to S3 with no consumer code" → **Kinesis Data Firehose**
- "Real-time processing with custom consumers" → **Kinesis Data Streams**
- "Replay events from 5 days ago" → **KDS with extended retention**
- "Real-time windowed aggregations" → **Managed Service for Apache Flink**
- "Cheapest way to query S3 data" → **Athena with Parquet + Partitions**
- "Schema discovery for S3 data" → **Glue Crawler**
- "Scheduled nightly batch transform" → **Glue ETL with Job Bookmarks**
- "Multiple analytics services need shared schema" → **Glue Data Catalog**
- "Stream to S3 in Parquet format" → **Firehose with format conversion**
- "Reduce Athena query costs" → **Columnar format + Partitioning + Compression**
- "Consumer falling behind on stream" → **Add shards or use Enhanced Fan-Out**
- "Query DynamoDB with SQL without moving data" → **Athena Federated Query**
- "Live video from IoT cameras to ML" → **Kinesis Video Streams + Rekognition**
- "Prevent Glue job from reprocessing old data" → **Enable Job Bookmarks**
- "Data lake ETL pipeline" → **S3 → Glue Crawler → Glue ETL → S3 (Parquet) → Athena**

---

## 8. Deep Dive: 10 More Tricky Scenario-Based Questions

**Q1:** A company uses Kinesis Data Streams with 20 shards and 5 KCL consumer applications (each with 4 workers). Each application processes all shards. Consumers experience high `GetRecords` throttling. What's wrong?

**A:** Without Enhanced Fan-Out, all 5 consumer applications share the 2 MB/sec read capacity per shard. With 5 consumers polling the same 20 shards, each consumer effectively gets only 400 KB/sec per shard (2 MB / 5). Additionally, `GetRecords` is limited to 5 calls/sec per shard — with 5 consumers, they collectively hit this limit. **Fix:** Enable **Enhanced Fan-Out** for each consumer application. Each registered EFO consumer gets a dedicated 2 MB/sec per shard (push model, no polling). **Key phrase:** "multiple applications reading same stream" + "throttling" = Enhanced Fan-Out.

---

**Q2:** A company runs Athena queries on Parquet data partitioned by `date`. A query with `WHERE date = '2024-03-15'` scans 50 GB but the partition only has 2 GB of data. What's happening?

**A:** Likely causes: (1) The partition column is stored **inside the Parquet files as a regular column** rather than being inferred from the Hive-style path. Athena scans the files to filter. (2) The table definition doesn't have the partition registered in the Glue Data Catalog (Athena reads all partitions). (3) The partition column data type mismatch (string vs date) prevents predicate pushdown. **Fix:** Ensure partitions are registered in the catalog (`MSCK REPAIR TABLE` or `ALTER TABLE ADD PARTITION`), verify the partition column type matches the WHERE clause value type, and confirm data is physically organized in partition paths. **Key trap:** Physical folder structure alone isn't enough — Glue Catalog must know about the partitions.

---

**Q3:** A Kinesis Data Firehose delivery stream is configured with a Lambda transformation function. The function processes records but occasionally returns records with status `ProcessingFailed`. These records end up in S3 but with garbled content. Why?

**A:** When a Lambda transformation returns `ProcessingFailed`, Firehose delivers the **original source record** (not the transformed one) to the configured S3 backup/error bucket. If you're seeing garbled content in the main destination, the issue is likely that the Lambda is returning `Ok` status but with incorrectly base64-encoded data. **Key requirement:** Lambda transformation must return records with: `recordId` (same as input), `result` (Ok/Dropped/ProcessingFailed), and `data` (base64-encoded transformed payload). Incorrect base64 encoding = garbled output with `Ok` status.

---

**Q4:** A data team runs Glue ETL jobs daily at 2 AM. On Monday, the job fails because a source S3 file has a new column that wasn't in the schema. The crawler ran on Sunday and didn't detect it. How should they handle schema evolution?

**A:** Options: (1) **Run the crawler immediately before the ETL job** (but adds latency/cost). (2) Use Glue ETL's **DynamicFrame with `ResolveChoice`** — it handles schema inconsistencies dynamically (new columns become `choice` types that you can cast). (3) **Schema Registry** for streaming sources (validates schemas before processing). (4) Configure crawler to **update table schema** on new columns (vs. create new table version). **Best practice:** Use DynamicFrames (they handle semi-structured data natively), don't rely on the crawler being perfectly in sync. **Key concept:** Glue Crawlers detect schema changes but aren't real-time. DynamicFrames handle schemaless/evolving data gracefully.

---

**Q5:** A company uses Athena to query CloudTrail logs stored in S3. Queries are slow (5+ minutes) and expensive. CloudTrail delivers new log files every 5 minutes, each about 1 MB. How do they optimize?

**A:** The problem is **many small files** (1 MB each, thousands per day). Athena performs poorly with many small files due to S3 LIST + open overhead per file. **Fixes:** (1) **Create a CloudTrail Lake** (managed, queryable, eliminates file management). (2) **Compact files** using a Glue ETL job: periodically merge small files into larger Parquet files partitioned by date/region/event. (3) **Use Athena CTAS** to create a compacted, Parquet version of the data. (4) **CloudTrail → S3 → Firehose (with buffering) → S3 (larger files)**. **Key optimization rule:** Athena works best with files >128 MB in columnar format with partitions.

---

**Q6:** A company has a Kinesis Data Stream in us-east-1. They need the same data available in eu-west-1 for a consumer there, with less than 5 seconds replication lag. What's the best approach?

**A:** No native KDS cross-region replication exists. **Options:** (1) **KDS cross-region consumer**: Lambda/KCL in us-east-1 reads and PutRecords to a KDS stream in eu-west-1 (lowest latency, ~seconds). (2) **Managed Flink application** reading from source KDS and writing to destination KDS (more complex but handles backpressure). (3) **Amazon MSK** with built-in cross-region replication (MirrorMaker 2) if willing to change technology. **Wrong answer:** "Use S3 Cross-Region Replication" — this is for objects, introduces minutes of delay, and isn't streaming. **Key trade-off:** Custom replication adds operational burden; consider whether both regions truly need real-time streaming vs. eventual consistency.

---

**Q7:** A Glue ETL job with Job Bookmarks enabled processes CSV files from S3. The company re-uploads a corrected version of a previously processed file (same filename, new content). The job doesn't pick up the corrected file. Why?

**A:** Glue Job Bookmarks for S3 track files by **file modification timestamp and path**. If you overwrite a file with the same name, the behavior depends on whether the new file has a newer timestamp (it should). However, **if the job bookmark already recorded that path as processed, it won't reprocess it even with a new timestamp** (bookmark tracks max timestamp seen). **Fix:** (1) Reset the job bookmark for that specific run. (2) Use a new filename for corrections. (3) Place corrections in a separate "corrections" prefix processed by a different job. **Key limitation:** Job Bookmarks are forward-only — they don't detect in-place updates to previously processed files.

---

**Q8:** An organization uses Athena with Glue Data Catalog. They want Athena in Account A to query data cataloged in Account B. Both accounts are in the same AWS Organization. What's needed?

**A:** Configure a **Glue Data Catalog resource policy** in Account B that grants Account A access. Alternatively, use **AWS Lake Formation cross-account sharing** (preferred for fine-grained column/row-level control). Then in Account A's Athena, register Account B's catalog as a data source or use the catalog ID in queries. **Requirements:** (1) Catalog resource policy or Lake Formation grants. (2) S3 bucket policy in Account B must allow Account A to read the underlying data. (3) If KMS encrypted, KMS key policy must grant Account A decrypt access. **Key trap:** Three permission layers: Catalog access + S3 data access + KMS (if encrypted). Missing any one = access denied.

---

**Q9:** A company uses Kinesis Data Firehose to deliver data to an S3 bucket partitioned by `year/month/day/hour` using dynamic partitioning. They notice that Athena queries filtering by hour are still scanning the entire day's data. What's wrong?

**A:** Dynamic partitioning creates the S3 prefix structure, but **Athena doesn't automatically know about these partitions unless they're registered in the Glue Data Catalog**. Just having the folder structure isn't enough. **Fix:** (1) Run a Glue Crawler to detect the new partitions. (2) Use **Partition Projection** in the table DDL (best — no crawler needed, automatic). (3) Use `MSCK REPAIR TABLE` in Athena. (4) Use Lambda + `batch_create_partition` API triggered by Firehose delivery notification. **Key misconception:** Physical S3 folder structure ≠ Athena partition awareness. The catalog must be updated.

---

**Q10:** A company uses Managed Service for Apache Flink (Kinesis Data Analytics) to process a KDS stream. They update the Flink application code and redeploy. After deployment, the application starts reading from the latest position in the stream, missing records produced during the deployment window. How do they prevent data loss?

**A:** Configure the Flink application to use **savepoints/checkpoints**. When updating, take a **snapshot** before stopping the old version, then start the new version **from the snapshot**. The snapshot contains the exact shard iterator positions. The Flink application restores consumer positions and processes all records from where it left off. **Key settings:** (1) Enable checkpointing. (2) Use `RESTORE_FROM_LATEST_SNAPSHOT` on application update. (3) Configure `TRIM_HORIZON` as fallback starting position (reads from oldest available if no snapshot). **Wrong answer:** "Set starting position to TRIM_HORIZON" — this reprocesses ALL data in the retention window, not just the gap.

---

## 9. Service Comparison Tables

### Kinesis Data Streams vs Kinesis Data Firehose

| Dimension | Kinesis Data Streams | Kinesis Data Firehose |
|-----------|---------------------|----------------------|
| **Use Case** | Custom real-time processing, multiple consumers | Managed delivery to storage/analytics |
| **Latency** | ~200ms (EFO) to ~2s (shared) | 60s+ (buffer interval) |
| **Consumer Model** | You build/manage (KCL, Lambda, Flink) | No consumer (managed delivery) |
| **Replay** | Yes (within retention) | No |
| **Destinations** | Anything (your code decides) | S3, Redshift, OpenSearch, Splunk, HTTP |
| **Throughput Limit** | 1 MB/s write per shard, unlimited shards | Auto-scales (no shards) |
| **Pricing** | Per shard-hour OR per GB (on-demand) | Per GB ingested ($0.029/GB) |
| **HA Model** | Multi-AZ, 3 AZ replication | Multi-AZ, auto-retry (24h) |
| **Exam Tip** | "custom processing", "replay", "multiple consumers", "sub-second" | "fully managed", "no code", "load to S3/Redshift" |

### Kinesis Data Streams vs SQS

| Dimension | Kinesis Data Streams | SQS |
|-----------|---------------------|-----|
| **Use Case** | Streaming analytics, multiple consumers | Job queue, decoupling |
| **Ordering** | Per-shard (partition key) | Per-Message Group (FIFO only) |
| **Retention** | 24h–365 days | 1 min–14 days |
| **Replay** | Yes | No (message deleted after processing) |
| **Consumers** | Multiple (fan-out) | Single (one consumer per message) |
| **Delivery** | At-least-once | At-least-once (Standard), Exactly-once (FIFO) |
| **Throughput** | 1 MB/s per shard (unlimited shards) | Unlimited (Standard), 70k/s (FIFO) |
| **Pricing** | Per shard-hour + per GB | Per request ($0.40/million) |
| **HA Model** | Multi-AZ, durable | Multi-AZ, durable |
| **Exam Tip** | "streaming", "analytics", "replay", "multiple apps" | "queue", "decouple", "exactly-once", "per-message ack" |

### Athena vs Redshift vs EMR

| Dimension | Athena | Redshift | EMR |
|-----------|--------|----------|-----|
| **Use Case** | Ad-hoc SQL on S3 | Data warehouse, complex analytics | Big data processing (Spark, Hadoop) |
| **Data Location** | S3 (in-place) | Loaded into Redshift | S3/HDFS |
| **Management** | Serverless | Managed cluster (or Serverless) | You manage clusters (or Serverless) |
| **Pricing** | $5/TB scanned | Per-node-hour or Serverless RPU | Per-instance-hour |
| **Best For** | Occasional queries, data exploration | Frequent, complex BI queries | ETL at massive scale, ML |
| **HA Model** | Serverless (inherent) | Multi-AZ (RA3), snapshots | Multi-master, spot instances |
| **Exam Tip** | "serverless SQL", "occasional queries", "data lake" | "BI dashboard", "frequent complex joins", "data warehouse" | "Spark/Hadoop", "custom frameworks", "massive-scale ETL" |

### Glue ETL vs EMR vs Lambda for Data Processing

| Dimension | Glue ETL | EMR | Lambda |
|-----------|----------|-----|--------|
| **Use Case** | Managed Spark ETL | Custom big data frameworks | Small-scale event processing |
| **Max Duration** | No limit (job-based) | No limit (cluster-based) | 15 minutes |
| **Scaling** | DPU auto-scaling | Instance auto-scaling | Automatic |
| **Management** | Serverless (no cluster) | You manage cluster config | Serverless |
| **Framework** | Spark (Python/Scala), Python Shell, Ray | Spark, Hadoop, Hive, Presto, Flink | Custom code |
| **Pricing** | $0.44/DPU-hour | EC2 instance pricing + EMR surcharge | Per-invocation + duration |
| **HA Model** | Managed retries + bookmarks | Multi-AZ, spot instance resilience | Multi-AZ |
| **Exam Tip** | "managed ETL", "no cluster management", "Glue catalog" | "custom Spark", "existing Hadoop", "full control" | "small files", "event-triggered transform" |

### Firehose Format Conversion vs Glue ETL vs Athena CTAS

| Dimension | Firehose Format Conversion | Glue ETL | Athena CTAS |
|-----------|---------------------------|----------|-------------|
| **Use Case** | Convert streaming data on delivery | Batch transform existing data | Ad-hoc conversion via SQL |
| **Input** | Streaming (Firehose source) | S3, JDBC, Catalog tables | S3 (via Athena table) |
| **Output Format** | Parquet, ORC | Any (Parquet, ORC, CSV, JSON) | Parquet, ORC, JSON, CSV, Avro |
| **Scheduling** | Continuous (with stream) | Cron, event-triggered, on-demand | On-demand (SQL query) |
| **Incremental** | Automatic (streaming) | Job Bookmarks | Manual (query new data) |
| **Exam Tip** | "streaming to Parquet" | "batch ETL, recurring, incremental" | "one-time conversion", "ad-hoc" |

---

## 10. When to Pick Service A over Service B — Exam Keywords

### Pick Kinesis Data Streams over Kinesis Firehose when:
- "custom consumer logic", "complex processing"
- "sub-second latency", "real-time" (<60s)
- "replay data", "reprocess from hours ago"
- "multiple applications consuming same data"
- "ordering matters" (partition key)
- "feed to Apache Flink for analytics"
- **Exam signal:** custom + real-time + replay

### Pick Kinesis Firehose over Kinesis Data Streams when:
- "load data to S3/Redshift/OpenSearch"
- "fully managed", "no consumer code", "serverless delivery"
- "near-real-time is acceptable" (60s+ delay OK)
- "convert to Parquet on delivery"
- "minimal operational overhead"
- **Exam signal:** delivery + managed + no code

### Pick Athena over Redshift when:
- "ad-hoc queries", "data exploration", "occasionally"
- "serverless", "no infrastructure"
- "data already in S3", "data lake"
- "pay only when querying"
- "cross-source federated queries" (DynamoDB, RDS, etc.)
- **Exam signal:** occasional + S3 + serverless + ad-hoc

### Pick Redshift over Athena when:
- "frequent complex queries", "BI dashboard"
- "sub-second query performance on large datasets"
- "complex joins across many tables"
- "data warehouse", "OLAP workloads"
- "Redshift Spectrum" (combines Redshift + S3 querying)
- **Exam signal:** frequent + complex + BI + performance

### Pick Glue ETL over EMR when:
- "serverless ETL", "no cluster management"
- "Spark ETL without infrastructure"
- "integrated with Glue Data Catalog"
- "job bookmarks for incremental"
- "DynamicFrames for semi-structured data"
- **Exam signal:** managed + Spark + catalog integration

### Pick EMR over Glue when:
- "custom Hadoop/Spark configuration"
- "existing Spark/Hadoop codebase"
- "need Presto/HBase/Flink natively"
- "full control over cluster"
- "long-running cluster for interactive queries"
- **Exam signal:** custom framework + control + existing code

### Pick Athena over Glue ETL for data conversion when:
- "one-time format conversion"
- "minimize operational overhead for simple transform"
- "SQL-based transformation"
- "CTAS to create Parquet table"
- **Exam signal:** simple + one-time + SQL

### Pick KDS On-Demand over Provisioned when:
- "unpredictable traffic", "spiky load"
- "don't want to manage capacity"
- "new workload, unsure of volume"
- **Exam signal:** variable traffic + operational simplicity

### Pick KDS Provisioned over On-Demand when:
- "predictable, steady throughput"
- "cost optimization", "known capacity"
- "consistent baseline load"
- **Exam signal:** stable + cost-sensitive

---

## 11. "Gotcha" Differences the Exam Tests

### Kinesis Gotchas
1. **KDS Enhanced Fan-Out max 20 consumers per stream.** If you need more → redesign (multiple streams, aggregation layer).
2. **KDS record size max 1 MB.** Firehose can receive up to 1 MB per record too, but buffers up to 128 MB per delivery.
3. **KDS `GetRecords` returns up to 10 MB or 10,000 records per call, max 5 calls/sec per shard.** This is what causes shared throughput contention.
4. **KDS On-Demand starts at 4 MB/sec write capacity.** If your initial burst exceeds this, records may be throttled until it scales up (takes minutes).
5. **KDS shard splitting/merging is not instant.** Resharding takes time and the old shard enters `CLOSED` state — consumers must drain it before moving to new shards.
6. **KCL uses one DynamoDB table per application.** The table name = application name. Multiple apps need multiple tables. DynamoDB costs add to KDS costs.
7. **KDS ordering is per-shard only.** If you split a shard, the two child shards each maintain their own order — cross-shard ordering doesn't exist.
8. **Firehose CANNOT deliver to Kinesis Data Streams.** It goes TO destinations (S3, Redshift, etc.), not back to KDS.
9. **Firehose dynamic partitioning extracts keys from records** using JQ expressions or Lambda. Without dynamic partitioning, all records go to one prefix.
10. **KDS + Lambda: parallelization factor up to 10.** This means up to 10 concurrent Lambda invocations per shard, processing different batches from the same shard in parallel. Increases throughput but records within a batch are still ordered.

### Athena Gotchas
11. **LIMIT does NOT reduce data scanned (cost).** Athena reads all data matching the query, applies LIMIT after scan. Only partitioning and columnar formats reduce scan.
12. **Athena query timeout is 30 minutes.** Queries exceeding this are killed. Complex cross-source federated queries may hit this.
13. **Athena doesn't support UPDATE or DELETE.** It's read-only on S3 data. To modify, use CTAS to recreate or use Iceberg/Hudi table format.
14. **Athena workgroup data scan limits can kill queries.** If a query would exceed the per-query limit, it's cancelled immediately (before scanning). This can cause confusion — "why is my query failing with small data?"
15. **Partition Projection doesn't store partitions in Glue Catalog.** It calculates them on the fly. If other services (Redshift Spectrum, EMR) need the catalog partitions, they won't see Projection-only partitions.
16. **Athena Federated Query uses Lambda connectors.** Lambda has 15-min timeout — very large federated queries may fail.
17. **Athena charges for FAILED queries** up to the point of failure. A query that scans 5 TB and then errors at 3 TB → you pay for 3 TB scanned.

### Glue Gotchas
18. **Glue Job Bookmarks don't work with overwritten files.** They track files by path + timestamp forward. If you overwrite a file in-place, the bookmark may skip it.
19. **Glue Crawlers may create new tables instead of updating.** If schema changes significantly, crawler creates a new table (e.g., `table_1`, `table_2`). Configure this behavior in classifier settings.
20. **Glue ETL in VPC needs NAT Gateway or VPC endpoints.** Without internet access, the Glue job can't download scripts from S3 or communicate with the Glue service.
21. **Glue Data Catalog encryption is irreversible once enabled.** You can't disable it after turning on catalog-level KMS encryption.
22. **Glue Crawlers are NOT free.** They consume DPU-hours. Running crawlers hourly on large datasets is expensive. Use API-based partition management for known patterns.
23. **Glue Schema Registry ≠ Glue Data Catalog.** Schema Registry is for streaming data validation (Avro/JSON Schema). Data Catalog is for table metadata. Don't confuse them.
24. **Glue ETL output file count is determined by DPU parallelism.** More DPUs = more output files (more partitions in Spark). Use `coalesce()` or `repartition()` to control output file count. Too many small files = slow Athena.

### Cross-Service Gotchas
25. **KDS → Firehose → S3 is different from direct Firehose PUT → S3.** When KDS is the source, Firehose doesn't support all features (e.g., some dynamic partitioning limitations). Check current docs.
26. **Athena uses Glue Data Catalog by default in most regions.** But Athena has its own internal catalog in some older regions. Exam assumes Glue Catalog.
27. **Redshift Spectrum and Athena share the Glue Data Catalog.** Table definitions created by crawlers work for both — no need to duplicate.
28. **Firehose → Redshift goes through S3 first.** Firehose writes to S3, then issues a COPY command to Redshift. This means S3 + Redshift permissions + networking must all align.

---

## 12. Decision Tree: Choosing the Right Streaming/Analytics Service

```
START: What is the primary need?
│
├─── Need to INGEST streaming data?
│    ├─── Need custom real-time processing? → Kinesis Data Streams
│    │    ├─── Multiple consumers? → Enable Enhanced Fan-Out
│    │    ├─── Need replay? → Set appropriate retention (24h–365d)
│    │    └─── Feed to analytics? → KDS → Managed Flink
│    ├─── Just deliver to S3/Redshift/OpenSearch? → Kinesis Data Firehose
│    │    ├─── Need Parquet format? → Enable format conversion
│    │    ├─── Need custom partitioning in S3? → Enable dynamic partitioning
│    │    └─── Need transformation? → Add Lambda transform
│    └─── Video/audio from cameras/IoT? → Kinesis Video Streams
│
├─── Need to ANALYZE streaming data in real-time?
│    ├─── Windowed aggregations, anomaly detection? → Managed Flink
│    ├─── Simple transformation before delivery? → Firehose + Lambda
│    └─── Sub-second alerting on specific events? → KDS + Lambda consumer
│
├─── Need to QUERY data at rest (S3)?
│    ├─── Ad-hoc, occasional? → Athena
│    │    ├─── High cost? → Convert to Parquet + Add partitions
│    │    ├─── Need cross-source? → Athena Federated Query
│    │    └─── Many small files? → Compact first (Glue ETL or CTAS)
│    ├─── Frequent, complex BI? → Redshift (+ Spectrum for S3)
│    └─── Custom Spark analytics? → EMR
│
├─── Need to TRANSFORM data (ETL)?
│    ├─── Batch, managed, integrated with catalog? → Glue ETL
│    │    ├─── Incremental? → Enable Job Bookmarks
│    │    ├─── Semi-structured/evolving schema? → Use DynamicFrames
│    │    └─── Need to discover schema first? → Glue Crawler
│    ├─── One-time format conversion? → Athena CTAS
│    ├─── Streaming ETL? → Glue Streaming ETL or Managed Flink
│    └─── Custom Spark with full control? → EMR
│
├─── Need to CATALOG/DISCOVER data schema?
│    ├─── Unknown schema, first time? → Glue Crawler
│    ├─── Known schema, define manually? → Glue API / Athena DDL
│    └─── Known partition pattern? → Athena Partition Projection
│
└─── Need to OPTIMIZE Athena costs?
     ├─── Using CSV/JSON? → Convert to Parquet/ORC
     ├─── No partitions? → Add date/key partitions
     ├─── Many small files? → Compact to >128 MB
     ├─── SELECT *? → Select only needed columns
     └─── No cost controls? → Set workgroup data scan limits
```
