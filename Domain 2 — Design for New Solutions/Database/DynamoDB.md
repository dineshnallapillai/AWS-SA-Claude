# AWS SAP-C02 Study Notes: DynamoDB (DAX, Global Tables, Streams, Capacity Modes)

---

## 1. Core Concepts & Theory

### Amazon DynamoDB Fundamentals

Amazon DynamoDB is a **fully managed, serverless, key-value and document NoSQL database** delivering single-digit millisecond performance at any scale. It is AWS's flagship NoSQL service.

#### Core Properties

| Property | Detail |
|----------|--------|
| **Type** | NoSQL — key-value and document |
| **Managed** | Fully managed, serverless — no provisioning, patching, or server management |
| **Availability** | Data replicated across **3 AZs** automatically within a Region |
| **Durability** | 11 9s (99.999999999%) |
| **Consistency** | **Eventually consistent** (default) or **strongly consistent** reads |
| **Performance** | Single-digit millisecond latency at any scale. With DAX: **microseconds** |
| **Scale** | Virtually unlimited — tables can grow to any size. No practical limit on items or throughput |
| **Item size** | Max **400 KB** per item |
| **Schema** | Schemaless — only partition key (and optional sort key) are required. All other attributes are flexible per item |

#### Data Model

| Concept | Description |
|---------|-------------|
| **Table** | Collection of items (rows) |
| **Item** | Single data record (row). Collection of attributes |
| **Attribute** | Name-value pair on an item (column equivalent, but schemaless) |
| **Partition Key (PK)** | **Mandatory**. Hash key used to distribute data across partitions. Must be unique if no sort key |
| **Sort Key (SK)** | **Optional**. Combined with PK to form a **composite primary key**. Enables range queries within a partition |
| **Primary Key** | PK alone (simple) or PK + SK (composite). Uniquely identifies each item |
| **Secondary Indexes** | Additional query patterns. **GSI** (Global) and **LSI** (Local) |

#### Secondary Indexes

| Index Type | **GSI (Global Secondary Index)** | **LSI (Local Secondary Index)** |
|-----------|--------------------------------|-------------------------------|
| **Partition Key** | Different from base table | **Same as base table** |
| **Sort Key** | Different from base table | Different from base table |
| **Creation** | Anytime (after table creation) | **Only at table creation** — cannot add later |
| **Max per table** | **20** | **5** |
| **Throughput** | Has its **own provisioned capacity** (independent of base table) | Shares base table's capacity |
| **Consistency** | **Eventually consistent only** | Eventually consistent OR **strongly consistent** |
| **Size limit** | No per-partition limit | **10 GB per partition key value** |
| **Projection** | KEYS_ONLY, INCLUDE, ALL | KEYS_ONLY, INCLUDE, ALL |

**Exam critical:** GSIs have their own throughput capacity. If you under-provision GSI capacity, writes to the base table can be **throttled** even if the base table has capacity. This is a common exam trap.

---

### DynamoDB Capacity Modes

DynamoDB offers two capacity modes that determine how you're charged for read/write throughput.

#### On-Demand Mode

| Property | Detail |
|----------|--------|
| **Pricing** | Pay-per-request. Charged per Read Request Unit (RRU) and Write Request Unit (WRU) |
| **Provisioning** | None — DynamoDB auto-scales instantly |
| **Scaling** | Accommodates up to **2× previous peak** instantly. Beyond that, may throttle briefly while scaling |
| **Best for** | Unpredictable workloads, new tables with unknown traffic, development/test |
| **Switch** | Can switch between On-Demand and Provisioned **once every 24 hours** |

#### Provisioned Mode

| Property | Detail |
|----------|--------|
| **Pricing** | Charged per RCU (Read Capacity Unit) and WCU (Write Capacity Unit) provisioned |
| **Provisioning** | You specify RCUs and WCUs (or use Auto Scaling) |
| **Auto Scaling** | Target tracking auto scaling adjusts provisioned capacity based on utilization (target: typically 70%) |
| **Burst capacity** | DynamoDB reserves a portion of unused capacity for bursts (up to 300 seconds of accumulated unused capacity) |
| **Best for** | Predictable workloads, steady-state traffic, cost optimization at scale |
| **Reserved Capacity** | Purchase 1 or 3-year reservations for additional savings (up to **77%** off) |

#### Capacity Unit Definitions

| Unit | Reads | Writes |
|------|-------|--------|
| **1 RCU** | 1 strongly consistent read/s (up to 4 KB) OR 2 eventually consistent reads/s (up to 4 KB) OR 1 transactional read/s (up to 4 KB, costs 2 RCU) | N/A |
| **1 WCU** | N/A | 1 write/s (up to 1 KB). Transactional write costs 2 WCU |

**Exam math:**
- Strongly consistent read: item_size_in_4KB_blocks × reads_per_second = RCUs needed
- Eventually consistent read: same but ÷ 2 (half the RCUs)
- Write: item_size_in_1KB_blocks × writes_per_second = WCUs needed
- Transactional reads/writes: 2× the normal cost

---

### DynamoDB Accelerator (DAX)

DAX is a **fully managed, in-memory cache** for DynamoDB that delivers **microsecond** read latency.

#### Architecture

```
Application → DAX Cluster (in-memory cache) → DynamoDB Table
              ↑ cache hit: microseconds
              ↓ cache miss: single-digit ms (falls through to DynamoDB)
```

#### Key Details

| Property | Detail |
|----------|--------|
| **Latency** | **Microseconds** for cache hits (vs single-digit ms for DynamoDB) |
| **Protocol** | API-compatible with DynamoDB. **Drop-in replacement** — change endpoint, same SDK calls |
| **Cache types** | **Item cache** (GetItem, BatchGetItem results) and **Query cache** (Query/Scan results) |
| **Write-through** | Writes go through DAX to DynamoDB. DAX caches the item after write |
| **TTL** | Configurable per item cache (default 5 minutes) and query cache (default 5 minutes) |
| **Cluster** | 1–11 nodes. Multi-AZ recommended for HA. Primary + read replicas |
| **Node types** | r-type instances (memory-optimized): dax.r5.large, dax.r5.xlarge, etc. Also t-type for dev/test |
| **VPC only** | DAX clusters run **inside your VPC** — not publicly accessible |
| **Encryption** | At rest (KMS) and in transit (TLS) |
| **Consistency** | DAX serves **eventually consistent reads** from cache. **Strongly consistent reads bypass DAX** and go directly to DynamoDB |
| **Failover** | If primary node fails, DAX promotes a read replica. Automatic failover |

#### When DAX Is the Answer

- **Read-heavy workloads** with repeated reads of the same items (hot keys/items).
- **Microsecond latency** requirement.
- **Reduce DynamoDB read costs** — cache hits don't consume RCUs.
- **Gaming leaderboards, session stores, real-time dashboards** — same data read thousands of times.

#### When DAX Is NOT the Answer

| Scenario | Why Not DAX | Alternative |
|----------|------------|-------------|
| Write-heavy workloads | DAX primarily accelerates reads. Writes pass through to DynamoDB (no write caching benefit) | DynamoDB direct |
| Strongly consistent reads required | DAX only serves eventually consistent from cache. Strong reads bypass cache | DynamoDB direct (strongly consistent) |
| Need caching for non-DynamoDB data | DAX is DynamoDB-specific | ElastiCache (Redis/Memcached) |
| Simple key-value caching (multi-service) | DAX is tied to one DynamoDB table | ElastiCache |

#### DAX vs ElastiCache

| Feature | DAX | ElastiCache |
|---------|-----|-------------|
| Scope | DynamoDB only | Any data source |
| API | DynamoDB API-compatible | Redis/Memcached API |
| Integration | Drop-in (change endpoint) | Application code changes needed |
| Use case | Cache DynamoDB reads | General caching (sessions, API responses, computed results) |
| Exam keyword | "DynamoDB" + "microsecond" or "cache" | "cache" without DynamoDB context, or caching for RDS/API/session |

---

### DynamoDB Global Tables

Global Tables provide **fully managed, active-active, multi-Region replication** for DynamoDB tables.

#### Architecture

```
us-east-1: DynamoDB Table (read/write)
  ↕ asynchronous replication
eu-west-1: DynamoDB Table (read/write)
  ↕ asynchronous replication
ap-southeast-1: DynamoDB Table (read/write)

All replicas are ACTIVE — read AND write from any Region
Conflict resolution: Last-writer-wins (based on timestamp)
```

#### Key Details

| Property | Detail |
|----------|--------|
| **Active-Active** | **All replicas accept reads AND writes** — true multi-Region active-active |
| **Replication** | Asynchronous, typically **<1 second** cross-Region |
| **Conflict resolution** | **Last-writer-wins** (timestamp-based). No manual conflict handling needed |
| **Consistency** | Within a Region: strongly or eventually consistent reads. **Cross-Region: eventually consistent** (replication lag) |
| **Max Regions** | Any number of AWS Regions (no hard limit documented) |
| **Version** | **Version 2019.11.21** (current). Older version (2017) had more limitations |
| **Requirements** | DynamoDB Streams must be **enabled** (New and Old Images). On-Demand or Provisioned with Auto Scaling |
| **Table must be empty or new** when adding to Global Tables (for adding new Regions to existing table, data is auto-replicated) |
| **Encryption** | All replicas must use the same encryption type (AWS-owned, AWS-managed, or CMK) |
| **TTL deletes** | TTL deletes in one Region are **replicated** to all other Regions (the delete propagates) |
| **Capacity** | Each Region has its own capacity settings. For Provisioned mode, **must use Auto Scaling** |
| **Pricing** | Replicated writes charged per Region (rWCU — replicated Write Capacity Unit). Reads are standard |

#### Global Tables vs Aurora Global Database

| Feature | DynamoDB Global Tables | Aurora Global Database |
|---------|----------------------|----------------------|
| **Active-Active writes** | **Yes** (all Regions read/write) | **No** (single writer, read-only secondaries unless write forwarding) |
| **Database type** | NoSQL (key-value/document) | Relational (SQL) |
| **Conflict resolution** | Last-writer-wins (automatic) | Not needed (single writer) or write forwarding (adds latency) |
| **Replication lag** | <1 second (typical) | <1 second (storage-level) |
| **Use case** | Global apps needing multi-Region writes | Global apps with single-writer + multi-Region reads |

---

### DynamoDB Streams

DynamoDB Streams captures a **time-ordered sequence of item-level modifications** (inserts, updates, deletes) to a DynamoDB table.

#### Architecture

```
DynamoDB Table → DynamoDB Stream (24-hour log of changes)
  → Lambda function (event-driven processing)
  → Kinesis Data Streams (alternative destination)
  → Application (KCL consumer)
```

#### Key Details

| Property | Detail |
|----------|--------|
| **What it captures** | Every item-level change (INSERT, MODIFY, REMOVE) |
| **Ordering** | **Per-item ordering guaranteed** — changes to the same item arrive in order. Cross-item ordering NOT guaranteed |
| **Retention** | **24 hours** (then data is deleted from the stream) |
| **Stream views** | KEYS_ONLY, NEW_IMAGE, OLD_IMAGE, **NEW_AND_OLD_IMAGES** (most common) |
| **Consumers** | **Lambda** (event source mapping), KCL application, or **Kinesis Data Streams** (newer option) |
| **Shards** | Stream is divided into shards (managed by AWS). Scales automatically |
| **Exactly-once to stream** | Each change appears **exactly once** in the stream |
| **Lambda processing** | Lambda reads stream in batches. Retries on failure. Use `ReportBatchItemFailures` for partial batch success |
| **Required for** | **Global Tables** (must be enabled with NEW_AND_OLD_IMAGES) |

#### DynamoDB Streams vs Kinesis Data Streams for DynamoDB

| Feature | DynamoDB Streams | Kinesis Data Streams (for DynamoDB) |
|---------|-----------------|-----------------------------------|
| **Retention** | **24 hours** | **Up to 365 days** |
| **Consumers** | Lambda, KCL (max 2 consumers per shard) | **Up to 5 consumers per shard** (enhanced fan-out: 20+) |
| **Fan-out** | Limited | Better fan-out with enhanced fan-out |
| **Integration** | Direct Lambda trigger | Lambda, Kinesis Data Analytics, Kinesis Data Firehose, KCL |
| **Ordering** | Per-item | Per-shard (partition key based) |
| **Cost** | Included (read requests charged) | Kinesis pricing (per shard-hour + per GB) |
| **Use case** | Simple triggers, Global Tables | Complex event processing, long retention, multiple consumers |

#### Common Streams Use Cases

- **Trigger Lambda on data change**: Update a search index (OpenSearch), send notification, update aggregate table.
- **Cross-Region replication**: Global Tables uses Streams internally.
- **Materialized views**: Maintain denormalized views in another table.
- **Audit log**: Capture all changes for compliance (with Kinesis for >24h retention).
- **Event-driven architectures**: DynamoDB change → Lambda → EventBridge → downstream services.

---

### DynamoDB Transactions

DynamoDB supports **ACID transactions** across multiple items and tables.

#### Key Details

| Property | Detail |
|----------|--------|
| **Operations** | `TransactWriteItems` (up to 100 items), `TransactGetItems` (up to 100 items) |
| **Scope** | Single account, single Region. Can span **multiple tables** |
| **ACID** | Atomicity, Consistency, Isolation, Durability — all or nothing |
| **Cost** | **2× the RCU/WCU** of standard operations (transactional reads and writes consume double) |
| **Use case** | Financial transactions, order processing, anything requiring multi-item consistency |
| **Limit** | Max **100 items** per transaction. Max **4 MB** total data per transaction |
| **Idempotency** | Client-side token for idempotent transactions (prevents double-processing) |

---

### Additional DynamoDB Features (Exam Relevant)

#### Time to Live (TTL)

- Automatically **delete expired items** at no cost (no WCU consumed for TTL deletes).
- Define a TTL attribute (Unix epoch timestamp). Items deleted within ~48 hours of expiry.
- TTL deletes are captured in **DynamoDB Streams** (as REMOVE events).
- TTL deletes in Global Tables are **replicated** to all Regions.
- **Use cases**: Session management, log expiry, temporary data cleanup.

#### Point-in-Time Recovery (PITR)

- Continuous backups with **35-day recovery window**.
- Restore to any second within the window. Creates a **new table** (not in-place).
- Must be **enabled** on the table (not on by default).
- Protects against accidental writes/deletes.
- RPO: essentially 0 (continuous). RTO: minutes to hours (depends on table size).

#### On-Demand Backup and Restore

- Manual, full table backups at any time.
- **No impact on performance** — does not consume capacity.
- Retained until explicitly deleted.
- Restore creates a **new table**.
- Can restore cross-Region and cross-account (via AWS Backup).

#### DynamoDB Export to S3

- Export table data to S3 in **DynamoDB JSON or Amazon Ion format**.
- Uses PITR (must be enabled). Does NOT consume table RCUs.
- Exports a point-in-time snapshot.
- Use for: analytics with Athena, data lake ingestion, long-term archival.

#### DynamoDB Import from S3

- Import data from S3 (CSV, DynamoDB JSON, Amazon Ion) to create a new table.
- Does NOT consume WCUs during import.
- Creates a new table — cannot import into an existing table.

#### Conditional Writes

- Write only if a condition is met (e.g., attribute_not_exists, attribute_equals).
- Used for **optimistic locking** — prevent concurrent overwrites.
- **Idempotent operations**: Conditional writes are the DynamoDB pattern for idempotency.

#### Batch Operations

- **BatchGetItem**: Read up to **100 items** from one or more tables in a single call.
- **BatchWriteItem**: Write/delete up to **25 items** in a single call.
- **Not transactional** — individual items can succeed or fail independently.

---

### Default Limits Worth Memorizing

| Resource | Limit | Adjustable? |
|----------|-------|-------------|
| Item max size | **400 KB** | No |
| Partition key max length | **2,048 bytes** | No |
| Sort key max length | **1,024 bytes** | No |
| GSI per table | **20** | Yes (up to 25) |
| LSI per table | **5** | No |
| LSI creation | **At table creation only** | N/A |
| LSI per PK value | **10 GB** | No |
| Projected attributes per index | **100** (for INCLUDE projection) | No |
| Transaction max items | **100** | No |
| Transaction max size | **4 MB** | No |
| BatchGetItem max items | **100** | No |
| BatchWriteItem max items | **25** | No |
| Max table throughput (provisioned, per table) | **40,000 RCU / 40,000 WCU** (default). Can increase to millions | Yes |
| Max table throughput (on-demand, per table) | **40,000 RRU / 40,000 WRU** initially. Auto-scales to previous 2× peak | Yes |
| PITR recovery window | **35 days** | No |
| DynamoDB Streams retention | **24 hours** | No |
| TTL deletion delay | Up to **48 hours** after expiry | N/A |
| DAX cluster max nodes | **11** | No |
| Capacity mode switch frequency | Once every **24 hours** | No |
| Reserved Capacity savings | Up to **77%** | N/A |

---

## 2. Design Patterns & Best Practices

### When to Use DynamoDB

| Scenario | Why DynamoDB |
|----------|-------------|
| Key-value lookups at any scale | Single-digit ms latency, serverless |
| Session management | TTL for auto-expiry, fast read/write |
| Gaming (leaderboards, player state) | Low latency, Global Tables for global games |
| IoT data ingestion | High write throughput, auto-scaling |
| E-commerce (shopping cart, catalog) | Flexible schema, transactions for orders |
| Real-time analytics (counters, aggregates) | Atomic counters, Streams for aggregation |
| Active-active multi-Region writes | Global Tables — true active-active |
| Event sourcing / change data capture | DynamoDB Streams |
| Serverless backends (Lambda) | Native integration, IAM auth, zero server management |

### When NOT to Use DynamoDB (Anti-Patterns)

| Scenario | Why Not | Alternative |
|----------|---------|------------|
| Complex joins, multi-table queries | DynamoDB has no joins — must denormalize | RDS, Aurora |
| Ad-hoc SQL queries, reporting | No SQL query language (PartiQL is limited) | RDS, Redshift, Athena |
| Large objects (>400 KB) | Item size limit 400 KB | S3 (store pointer in DynamoDB) |
| Strong consistency across Regions | Global Tables are eventually consistent cross-Region | Aurora Global Database (single writer) |
| BLOB/document storage | Not designed for large binary data | S3 |
| OLAP / data warehousing | Optimized for OLTP, not analytics | Redshift, Athena |
| Relational integrity (foreign keys, constraints) | NoSQL — no relational constraints | RDS, Aurora |
| Small, infrequently accessed tables where cost matters | DynamoDB has minimum throughput cost. RDS can be cheaper for tiny DBs | RDS (t3.micro, single-AZ) |

### Common Architectural Patterns

#### Serverless API Backend
```
API Gateway → Lambda → DynamoDB (On-Demand)
  → DAX (if read-heavy, microsecond latency needed)
  → DynamoDB Streams → Lambda (post-processing, indexing)
```

#### Global Application (Active-Active)
```
us-east-1: API Gateway → Lambda → DynamoDB (Global Table)
eu-west-1: API Gateway → Lambda → DynamoDB (Global Table)
ap-southeast-1: API Gateway → Lambda → DynamoDB (Global Table)

Route 53 (latency-based routing) → nearest Region
All Regions accept reads AND writes
Replication: <1 second, last-writer-wins conflict resolution
```

#### Event-Driven Architecture
```
DynamoDB Table → DynamoDB Streams (NEW_AND_OLD_IMAGES)
  → Lambda: Update OpenSearch index
  → Lambda: Send SNS notification
  → Lambda: Update aggregate table
  → Kinesis Data Streams → Kinesis Firehose → S3 (archival)
```

#### CQRS (Command Query Responsibility Segregation)
```
Write Path: Application → DynamoDB (primary table, optimized for writes)
  → DynamoDB Streams → Lambda
    → Read-optimized DynamoDB table (denormalized, GSI for query patterns)
    → OpenSearch (full-text search)
    → ElastiCache (hot data cache)
```

#### Large Object Pattern
```
Application → S3 (store large object, get S3 key)
           → DynamoDB (store metadata + S3 key pointer)
           
Read: Get metadata from DynamoDB → Fetch object from S3 using stored key
```

#### Single-Table Design
```
DynamoDB Table:
  PK: "USER#123"    SK: "PROFILE"         → User profile
  PK: "USER#123"    SK: "ORDER#001"       → User's order
  PK: "USER#123"    SK: "ORDER#002"       → Another order
  PK: "ORDER#001"   SK: "ITEM#A"          → Order line item
  PK: "ORDER#001"   SK: "ITEM#B"          → Another line item

GSI1: PK=OrderDate, SK=OrderID → Query orders by date
GSI2: PK=ProductID, SK=OrderDate → Query orders by product

Benefits: Single table, all access patterns, minimal RCU/WCU
```

### Well-Architected Framework Alignment

**Reliability**
- 3 AZs automatic replication. 11 9s durability.
- Global Tables for multi-Region resilience.
- PITR for continuous backup (35 days).
- TTL for automatic data cleanup (prevents unbounded growth).
- DynamoDB handles all scaling, patching, hardware management.

**Security**
- IAM fine-grained access control (item-level, attribute-level conditions).
- Encryption at rest (KMS) — always on.
- VPC endpoints for private access.
- DAX inside VPC for cache security.
- DynamoDB Streams + Lambda for real-time security event processing.

**Performance**
- Single-digit ms latency. DAX for microseconds.
- Partition key design is critical — distribute workload evenly.
- GSI/LSI for multiple query patterns.
- On-Demand for unpredictable traffic. Provisioned + Auto Scaling for steady state.
- Avoid hot partitions (spread reads/writes across partition keys).

**Cost Optimization**
- On-Demand for unpredictable workloads (no over-provisioning).
- Provisioned + Auto Scaling + Reserved Capacity for steady state (up to 77% savings).
- TTL for automatic data cleanup (free deletes).
- Export to S3 for analytics (instead of Scan operations that consume RCUs).
- DAX to reduce DynamoDB read costs (cache hits don't consume RCUs).

**Operational Excellence**
- Fully serverless — zero operational overhead.
- CloudWatch metrics for monitoring.
- DynamoDB Contributor Insights — identify hot keys.
- AWS Backup for centralized backup management.
- DynamoDB import/export for data portability.

---

## 3. Security & Compliance

### IAM Access Control

DynamoDB uses **IAM exclusively** for access control — there are no resource-based policies (no bucket-policy equivalent).

#### IAM Policy Examples

**Fine-grained access (restrict to specific items):**
```json
{
  "Effect": "Allow",
  "Action": ["dynamodb:GetItem", "dynamodb:PutItem", "dynamodb:UpdateItem"],
  "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/Users",
  "Condition": {
    "ForAllValues:StringEquals": {
      "dynamodb:LeadingKeys": ["${aws:PrincipalTag/userId}"]
    }
  }
}
```
This restricts a user to only access items where the partition key matches their own user ID. **Item-level security without application code.**

**Restrict to specific attributes:**
```json
{
  "Condition": {
    "ForAllValues:StringEquals": {
      "dynamodb:Attributes": ["userId", "name", "email"]
    }
  }
}
```

#### VPC Endpoints

- **Gateway endpoint** for DynamoDB (like S3). Free. Routes traffic within the VPC without internet.
- Use VPC endpoint policies to restrict which tables can be accessed from the VPC.
- **Exam critical:** DynamoDB VPC endpoints are **Gateway** type (not Interface). Free, no ENI needed.

### Encryption

| Layer | Detail |
|-------|--------|
| **At rest** | **Always encrypted.** Choose: AWS-owned key (default, free), AWS-managed KMS key (`aws/dynamodb`), or **Customer Managed KMS Key (CMK)** |
| **In transit** | **TLS** (HTTPS) for all DynamoDB API calls |
| **DAX at rest** | KMS encryption |
| **DAX in transit** | TLS encryption (configurable) |
| **Streams** | Encrypted at rest with the same key as the base table |
| **Backups** | Encrypted with same key as source table |
| **Global Table replicas** | Each replica can use a **different KMS key** (per-Region keys) |
| **Cannot change encryption key** | Must create a new table with desired key and migrate data. Or use AWS Backup → restore with different key |

### Logging & Monitoring

| Tool | What It Captures |
|------|-----------------|
| **CloudWatch Metrics** | ConsumedReadCapacityUnits, ConsumedWriteCapacityUnits, ThrottledRequests, ReadThrottleEvents, WriteThrottleEvents, SystemErrors, UserErrors, SuccessfulRequestLatency, **AccountProvisionedReadCapacityUtilization** |
| **CloudTrail** | DynamoDB management API calls (CreateTable, UpdateTable, DeleteTable). **Data plane operations (GetItem, PutItem) are NOT logged by CloudTrail by default** — enable via **CloudTrail Data Events** (additional cost) |
| **DynamoDB Contributor Insights** | Identifies **most-accessed keys** (hot keys) and most-throttled keys. Essential for diagnosing hot partition issues |
| **DAX CloudWatch Metrics** | CacheHitCount, CacheMissCount, ItemCacheHits, QueryCacheHits |
| **AWS Config** | Rules: `dynamodb-table-encrypted-kms`, `dynamodb-pitr-enabled`, `dynamodb-autoscaling-enabled` |

### Compliance Patterns

- **HIPAA/PCI**: Encryption at rest (CMK) + in transit (TLS) + CloudTrail data events + VPC endpoint (no internet exposure) + fine-grained IAM.
- **GDPR**: TTL for automatic data expiry. Fine-grained access control. DynamoDB Streams + Lambda for data subject deletion propagation.
- **Financial (SOX)**: DynamoDB transactions for ACID. PITR for audit trail. CloudTrail data events for access logging.
- **Data residency**: Global Tables only in approved Regions. SCPs to restrict DynamoDB operations by Region.

---

## 4. Cost Optimization

### Pricing Model

#### On-Demand Pricing (us-east-1)

| Resource | Cost |
|----------|------|
| Read Request Unit (RRU) | $0.25 per million |
| Write Request Unit (WRU) | $1.25 per million |
| Replicated WRU (Global Tables) | $1.875 per million |
| Storage | $0.25/GB/month |

#### Provisioned Pricing (us-east-1)

| Resource | Cost |
|----------|------|
| Read Capacity Unit (RCU) | $0.00013/hour ($0.09/month) |
| Write Capacity Unit (WCU) | $0.00065/hour ($0.47/month) |
| Replicated WCU (Global Tables) | $0.000975/hour |
| Storage | $0.25/GB/month |
| Reserved Capacity (1-year) | ~53% savings |
| Reserved Capacity (3-year) | ~**77% savings** |

#### Other Cost Components

| Component | Cost |
|-----------|------|
| DAX (per node-hour) | ~$0.269/hr (dax.r5.large) |
| DynamoDB Streams reads | $0.02 per 100,000 read requests |
| PITR | $0.20/GB/month |
| On-Demand Backup | $0.10/GB/month |
| Restore from backup | $0.15/GB |
| Export to S3 | $0.11/GB |
| Global Table replicated writes | 50% premium over standard writes |
| Data transfer (cross-Region) | Standard AWS data transfer rates |

### Cost-Saving Strategies

1. **Provisioned + Auto Scaling + Reserved Capacity**: For predictable workloads. Auto Scaling handles daily patterns; Reserved Capacity saves up to 77% on the baseline.

2. **On-Demand for unpredictable workloads**: No over-provisioning. But at high, steady throughput, Provisioned is **significantly cheaper**.

3. **DAX to reduce read costs**: Cache hits don't consume RCUs. If you read the same items repeatedly, DAX saves both latency and cost.

4. **TTL instead of delete operations**: TTL deletes are **free** (no WCU consumed). Use TTL for session cleanup, log expiry, temporary data.

5. **Export to S3 for analytics**: Exporting to S3 costs $0.11/GB (uses PITR, doesn't consume RCUs). Much cheaper than Scan operations for analytics.

6. **Avoid Scan operations**: Scans consume massive RCUs. Design with proper GSI/LSI for all query patterns.

7. **GSI projection optimization**: Project only needed attributes (KEYS_ONLY or INCLUDE). ALL projection duplicates all data → 2× storage + write cost.

8. **Single-table design**: One table with well-designed PK/SK patterns instead of multiple tables. Fewer GSIs, less management.

9. **Switch between capacity modes strategically**: Dev/test on On-Demand. Production steady-state on Provisioned. Switch once per 24 hours.

### Common Cost Traps

- **GSI write throttling**: Under-provisioned GSI can throttle base table writes even when the base table has capacity. **GSI has its own capacity** — monitor and provision independently.
- **On-Demand at scale**: At high, steady throughput, On-Demand is 5–7× more expensive than Provisioned with Reserved Capacity.
- **Global Table replicated write cost**: 50% premium per replicated write. Three Regions = 3× write cost (1× base + 2× replicated). Budget carefully.
- **DAX cluster always running**: DAX nodes are charged per hour even with zero requests. Right-size and shut down dev/test clusters.
- **PITR storage cost**: Continuous backups consume storage. $0.20/GB/month on top of base storage.
- **Scan on large tables**: A full table Scan consumes RCUs for every item — can be extremely expensive on large tables.
- **Over-projected GSI**: GSI with ALL projection duplicates all data — doubles storage cost + write cost for the index.
- **Forgotten Auto Scaling cool-down**: Auto Scaling may scale down too slowly (default 5-min cool-down) leaving over-provisioned capacity running longer than needed.

### Cost Comparison: DynamoDB vs Alternatives

| Scenario | DynamoDB | Alternative |
|----------|----------|------------|
| High-scale key-value (>1M requests/day) | Optimal — purpose-built | Redis (ElastiCache) for ultra-low latency but requires management |
| Low-traffic, relational data | More expensive (minimum capacity + storage) | RDS (t3.micro, cheaper for small tables) |
| Multi-Region active-active writes | Global Tables (only NoSQL option with managed active-active) | Aurora Global Database (single writer — not active-active) |
| Large-scale analytics | Expensive (Scan costs) | Redshift, Athena (export to S3 first) |
| Session management | Excellent (TTL, serverless) | ElastiCache Redis (faster but requires management) |

---

## 5. High Availability, Disaster Recovery & Resilience

### DynamoDB Native HA

- Data replicated across **3 AZs** automatically — no configuration needed.
- **99.999% SLA** for Global Tables. **99.99% SLA** for single-Region tables.
- No single point of failure — fully managed, distributed.
- **No downtime for scaling** — capacity changes are seamless.

### DR Patterns

| Pattern | Implementation | RPO | RTO |
|---------|---------------|-----|-----|
| **Single-Region (default)** | 3 AZ replication, automatic | 0 (synchronous within Region) | 0 (automatic failover within Region) |
| **Multi-Region Active-Active** | **Global Tables** | <1 second (replication lag) | **~0** (all Regions active, Route 53 routes to nearest) |
| **Cross-Region backup restore** | AWS Backup cross-Region copy | Hours (backup frequency) | Minutes–hours (restore time) |
| **PITR restore** | Continuous backup, 35-day window | ~0 (continuous) | Minutes–hours (restore creates new table) |
| **Export to S3 + Import** | Export to S3 → replicate S3 cross-Region → import | Hours | Hours |

### Global Tables HA Pattern (Active-Active)

```
Route 53 (latency-based routing)
  → us-east-1: API Gateway → Lambda → DynamoDB Global Table
  → eu-west-1: API Gateway → Lambda → DynamoDB Global Table

If us-east-1 fails:
  - Route 53 detects via health check
  - All traffic routes to eu-west-1
  - eu-west-1 is already active — no promotion needed
  - RPO: <1 second replication lag at time of failure
  - RTO: Route 53 TTL (typically 60 seconds)
  
When us-east-1 recovers:
  - Replication resumes automatically
  - Data is synced, traffic balances again
```

**This is the gold standard for multi-Region HA with NoSQL on AWS.** No other service provides managed active-active multi-Region writes with automatic conflict resolution.

### Backup & Recovery

| Method | Creates New Table? | RCU Impact | Cross-Region? |
|--------|-------------------|-----------|---------------|
| **On-Demand Backup** | Yes (restore) | No impact | Yes (via AWS Backup) |
| **PITR** | Yes (restore) | No impact | No (same Region) |
| **Export to S3** | N/A (S3 data) | No impact (uses PITR) | Yes (S3 replication) |
| **Global Tables** | No (continuous sync) | Replicated writes cost | Yes (built-in) |

---

## 6. Exam-Focused Section

### Straightforward Questions (7 examples)

**Q1:** What is the maximum item size in DynamoDB?
**A:** **400 KB.** For larger objects, store them in S3 and keep a reference/pointer in DynamoDB.

**Q2:** A company needs microsecond read latency for a DynamoDB-based gaming leaderboard with millions of reads per second on the same items. What should they add?
**A:** **DAX (DynamoDB Accelerator).** It provides microsecond latency for cached reads. Drop-in compatible with DynamoDB SDK.

**Q3:** Which DynamoDB capacity mode is best for unpredictable, spiky workloads?
**A:** **On-Demand mode.** No provisioning needed — DynamoDB auto-scales per request. Pay-per-request.

**Q4:** Can you add a Local Secondary Index (LSI) to an existing DynamoDB table?
**A:** **No.** LSIs must be created **at table creation time**. Cannot be added later.

**Q5:** How does DynamoDB Global Tables handle write conflicts when the same item is updated in two Regions simultaneously?
**A:** **Last-writer-wins** (timestamp-based). The most recent write (by timestamp) takes precedence. Automatic conflict resolution.

**Q6:** What is the retention period for DynamoDB Streams?
**A:** **24 hours.** For longer retention, use Kinesis Data Streams for DynamoDB (up to 365 days).

**Q7:** What is required before enabling Global Tables on a DynamoDB table?
**A:** **DynamoDB Streams** must be enabled with **NEW_AND_OLD_IMAGES** stream view type. The table must use On-Demand or Provisioned with Auto Scaling.

---

### Tricky / Scenario-Based Questions (7 examples)

**Q1:** A company has a DynamoDB table with Provisioned capacity. During a flash sale, the application experiences heavy throttling even though average RCU utilization is only 40%. CloudWatch shows `ReadThrottleEvents` spiking. What is the most likely cause?

**A:** **Hot partition.** The traffic is concentrated on a small number of partition keys (popular items), exceeding the per-partition throughput limit — even though total table capacity is under-utilized.
- **Fix:** Redesign the partition key to distribute traffic more evenly. Use **DynamoDB Contributor Insights** to identify the hot keys. Consider adding a random suffix to spread writes. Or use DAX to cache hot reads.
- **Why not just increase RCU?** Increasing total RCU helps somewhat (DynamoDB has adaptive capacity that can borrow from underused partitions), but the root cause is uneven key distribution.
- **Key phrases:** "throttling" + "low average utilization" → Hot partition problem.

**Q2:** A company uses DynamoDB with DAX for a read-heavy application. They require strongly consistent reads for critical operations. An architect places all reads through DAX. Users report stale data on critical reads. What is wrong?

**A:** **DAX only serves eventually consistent reads from cache.** Strongly consistent reads **bypass the DAX cache** and go directly to DynamoDB. But if the application is using the DAX client for all reads, it defaults to eventually consistent.
- **Fix:** For critical reads requiring strong consistency, set `ConsistentRead=true` in the request. DAX will pass the request directly to DynamoDB (bypassing cache). For non-critical reads, continue using DAX's cached eventually consistent reads.
- **Key phrases:** "DAX" + "strongly consistent" + "stale data" → DAX serves eventually consistent only; strong reads bypass cache.

**Q3:** A company has DynamoDB Global Tables in us-east-1 and eu-west-1. A user in the US writes an item, and another user in Europe reads the same item 200ms later. The European user doesn't see the new item. Is this expected?

**A:** **Yes, this is expected.** Global Tables use **asynchronous replication** (typically <1 second). The read in Europe happened during the replication lag window. Cross-Region reads are **eventually consistent** by design.
- **If the European user absolutely needs the latest data:** Route the read to us-east-1 (where the write happened) using the writer endpoint. Or accept eventual consistency.
- **Key phrases:** "Global Tables" + "cross-Region" + "stale read" → Expected behavior (eventually consistent cross-Region).

**Q4:** A company stores session data in DynamoDB with a TTL attribute. The TTL for an item expired 10 minutes ago, but the item is still visible in the table. Is this a bug?

**A:** **No.** DynamoDB TTL deletes are **not instantaneous**. Items are typically deleted within **48 hours** of TTL expiry. The item may appear in reads for up to 48 hours after the TTL timestamp.
- **Workaround:** Filter out expired items in your application logic (check the TTL attribute against current time). Don't rely on TTL for real-time expiry.
- **Key phrases:** "TTL" + "item still visible after expiry" → Expected. Up to 48 hours deletion lag.

**Q5:** A company has a DynamoDB table with 5 GSIs. During high write traffic, writes to the base table are throttled even though the base table has ample WCU capacity. What is the most likely cause?

**A:** **One or more GSIs are under-provisioned.** GSIs have their **own capacity** (independent from the base table). If a GSI's WCU is exhausted, DynamoDB **throttles writes to the base table** to prevent the GSI from falling behind.
- **Fix:** Increase WCU on the throttled GSI(s). Use CloudWatch to check `WriteThrottleEvents` per GSI. Consider using On-Demand capacity mode (auto-scales GSIs too).
- **Key phrases:** "base table throttled" + "base table has capacity" + "GSIs" → GSI is under-provisioned.

**Q6:** A company needs to replicate DynamoDB changes to an Amazon OpenSearch cluster for full-text search. They also need to archive all changes for 1 year for compliance. Using DynamoDB Streams alone, can they achieve both?

**A:** **Partially.** DynamoDB Streams has a **24-hour retention** — insufficient for 1-year archival.
- **Solution:** Use **Kinesis Data Streams for DynamoDB** (up to 365 days retention) instead of native DynamoDB Streams. Consumer 1: Lambda → OpenSearch (real-time indexing). Consumer 2: Kinesis Data Firehose → S3 (long-term archival).
- **Why not DynamoDB Streams + Lambda → S3?** Works for real-time, but if the Lambda fails for >24 hours, data in the stream is lost. Kinesis provides longer retention as a safety buffer.
- **Key phrases:** "DynamoDB changes" + ">24 hours retention" or "archival" → Kinesis Data Streams for DynamoDB.

**Q7:** A company is migrating from Oracle to DynamoDB. They need ACID transactions that span multiple items (e.g., transfer money from one account to another). A team member says DynamoDB doesn't support transactions. Is this correct?

**A:** **No — DynamoDB supports ACID transactions** via `TransactWriteItems` and `TransactGetItems`.
- **Capabilities:** Atomically write up to 100 items across multiple tables. All-or-nothing. Consistent isolation.
- **Cost:** Transactional operations consume **2× the normal WCU/RCU**.
- **Limitation:** Max 100 items, max 4 MB per transaction. Single account, single Region.
- **Key phrases:** "DynamoDB" + "ACID" + "multi-item" → TransactWriteItems / TransactGetItems.

---

### Common Exam Traps & Pitfalls

1. **DAX only caches eventually consistent reads**: Strongly consistent reads bypass DAX entirely. If the question requires strong consistency + microsecond latency, that combination is not possible with DAX.

2. **GSI has its own capacity**: Under-provisioned GSI throttles the BASE table. This is the #1 DynamoDB performance trap.

3. **LSI must be created at table creation**: Cannot add later. If the question mentions adding an LSI to an existing table, it's wrong.

4. **Global Tables = eventually consistent cross-Region**: Reads in a different Region may not reflect the latest write from another Region. Not strongly consistent across Regions.

5. **TTL deletes are not instant**: Up to 48 hours delay. Application must filter expired items.

6. **DynamoDB Streams retention = 24 hours**: For longer retention, use Kinesis Data Streams for DynamoDB (up to 365 days).

7. **On-Demand mode is expensive at sustained high scale**: Provisioned + Reserved Capacity saves up to 77%. On-Demand is best for unpredictable traffic, not steady-state.

8. **Item size limit = 400 KB**: Store large objects in S3, keep a reference in DynamoDB.

9. **Transactions cost 2× the normal WCU/RCU**: A transactional write of a 2 KB item consumes 4 WCUs (2 KB × 2 for transaction).

10. **DynamoDB VPC endpoint is Gateway type (not Interface)**: Free, no ENI. Same as S3 Gateway endpoint.

11. **PITR and On-Demand backups restore to a NEW table**: Not in-place. Must redirect the application to the new table.

12. **Global Tables require DynamoDB Streams enabled**: With NEW_AND_OLD_IMAGES view type. Cannot use Global Tables without Streams.

13. **Capacity mode switch: once per 24 hours**: Cannot rapidly toggle between On-Demand and Provisioned.

14. **Scan consumes massive capacity**: Full table Scan reads every item. Use Query with proper indexes instead. For analytics, export to S3 and use Athena.

15. **Reserved Capacity is NOT the same as Savings Plans**: DynamoDB uses its own Reserved Capacity (not EC2/Compute Savings Plans).

---

## 7. Cheat Sheet

### Must-Know Facts for Exam Day

| Fact | Value |
|------|-------|
| Item max size | **400 KB** |
| Max GSI per table | **20** |
| Max LSI per table | **5** (at creation only) |
| LSI per PK size limit | **10 GB** |
| Max transaction items | **100** |
| Transaction cost | **2× WCU/RCU** |
| RCU: 1 strongly consistent read | **4 KB** per second |
| RCU: 1 eventually consistent read | **4 KB × 2** per second |
| WCU: 1 write | **1 KB** per second |
| DynamoDB Streams retention | **24 hours** |
| Kinesis for DynamoDB retention | **Up to 365 days** |
| TTL deletion delay | **Up to 48 hours** |
| TTL delete WCU cost | **Free** (no WCU consumed) |
| DAX latency | **Microseconds** (cache hit) |
| DAX consistency | **Eventually consistent only** (from cache) |
| DAX max nodes | **11** |
| Global Tables conflict resolution | **Last-writer-wins** |
| Global Tables replication lag | **<1 second** typical |
| Global Tables SLA | **99.999%** |
| PITR recovery window | **35 days** |
| Reserved Capacity max savings | **77%** (3-year) |
| Capacity mode switch | Once per **24 hours** |
| VPC endpoint type | **Gateway** (free) |
| Encryption at rest | **Always on** |
| Data replication | **3 AZs** automatically |

### Key Differentiators

| DAX | ElastiCache |
|-----|-------------|
| DynamoDB only | Any data source |
| DynamoDB API-compatible (drop-in) | Redis/Memcached API (code changes) |
| Microsecond reads for DynamoDB | Microsecond reads for any cached data |
| Eventually consistent from cache | N/A (application manages consistency) |
| Exam keyword: "DynamoDB cache" | Exam keyword: "general caching" or "session store" or "RDS cache" |

| DynamoDB Streams | Kinesis Data Streams (for DynamoDB) |
|-----------------|-----------------------------------|
| 24-hour retention | Up to 365 days |
| 2 consumers per shard | 5+ consumers (enhanced fan-out: 20+) |
| Simple Lambda triggers | Complex event processing pipelines |
| Free (pay for reads) | Kinesis pricing (per shard-hour) |

| On-Demand | Provisioned |
|-----------|------------|
| Pay per request | Pay per RCU/WCU provisioned |
| No capacity planning | Must provision (or Auto Scale) |
| Auto-scales instantly | Auto Scaling (slower, 5-min cooldown) |
| Best for unpredictable traffic | Best for steady-state (cheaper) |
| 5–7× more expensive at scale | Up to 77% cheaper with Reserved Capacity |

| DynamoDB Global Tables | Aurora Global Database |
|----------------------|----------------------|
| **Active-active** (all Regions read/write) | Single writer (secondaries read-only) |
| NoSQL (key-value/document) | Relational (SQL) |
| Last-writer-wins conflicts | No conflicts (single writer) |
| <1s replication | <1s replication |
| Eventually consistent cross-Region | Eventually consistent cross-Region |

### Decision Flowchart

```
Question says "NoSQL" or "key-value" or "serverless database"?
  → DynamoDB

Question says "microsecond read latency" + "DynamoDB"?
  → DAX

Question says "microsecond latency" + NOT DynamoDB-specific?
  → ElastiCache (Redis/Memcached)

Question says "active-active multi-Region writes"?
  → DynamoDB Global Tables

Question says "multi-Region" + "relational" + "single writer"?
  → Aurora Global Database (not DynamoDB)

Question says "unpredictable traffic" + "no capacity planning"?
  → DynamoDB On-Demand

Question says "steady-state" + "cost-optimize DynamoDB"?
  → Provisioned + Auto Scaling + Reserved Capacity

Question says "DynamoDB change capture" + ">24 hours retention"?
  → Kinesis Data Streams for DynamoDB

Question says "DynamoDB change trigger" + "Lambda"?
  → DynamoDB Streams + Lambda

Question says "throttling" + "low utilization" on DynamoDB?
  → Hot partition. Use Contributor Insights, redesign PK

Question says "base table throttled" + "GSIs exist"?
  → Under-provisioned GSI capacity

Question says "strongly consistent" + "DAX"?
  → DAX serves eventually consistent only. Strong reads bypass cache

Question says "items >400 KB"?
  → Store in S3, reference in DynamoDB

Question says "complex joins" or "SQL analytics" on DynamoDB?
  → Wrong service. Use RDS/Aurora for relational, Redshift/Athena for analytics

Question says "auto-expire data" + "no cost for deletes"?
  → DynamoDB TTL

Question says "DynamoDB analytics" + "without consuming RCUs"?
  → Export to S3 → Athena

Question says "ACID transaction" + "DynamoDB"?
  → TransactWriteItems / TransactGetItems (2× cost, max 100 items)

Question says "Global Tables" + "strongly consistent cross-Region reads"?
  → NOT possible. Cross-Region reads are eventually consistent

Question says "add LSI to existing table"?
  → NOT possible. LSI must be created at table creation
```

---

*Generated for AWS SAP-C02 exam preparation. Last updated: 2026-04-26.*
