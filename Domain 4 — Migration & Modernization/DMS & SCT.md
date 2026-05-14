# AWS SAP-C02 Study Notes: DMS & SCT (Database Migration Service & Schema Conversion Tool)

## 1. Core Concepts & Theory

### AWS Database Migration Service (DMS)

**Purpose:** Migrate databases to AWS with minimal downtime using continuous data replication.

**Key Architecture:**
```
Source Database ──► Replication Instance ──► Target Database
      │                    │                       │
  (endpoint)        (runs migration tasks)    (endpoint)
      │                    │                       │
  Source                Task with              Target
  Endpoint              table mappings         Endpoint
```

**Core Components:**

| Component | Description |
|-----------|-------------|
| **Replication Instance** | EC2 instance (managed) that runs the migration task; processes data flow |
| **Source Endpoint** | Connection to the source database (on-premises, EC2, RDS, Aurora, etc.) |
| **Target Endpoint** | Connection to the target database or data store |
| **Migration Task** | Defines what to migrate, how to migrate, and mapping/transformation rules |
| **Table Mappings** | Rules for selecting tables, filtering rows/columns, and transforming data |

**Migration Types:**

| Type | Description | Use Case |
|------|-------------|----------|
| **Full Load** | Migrate all existing data at a point in time | One-time migration, can tolerate downtime |
| **CDC (Change Data Capture)** | Replicate only ongoing changes | Ongoing replication, no initial data needed |
| **Full Load + CDC** | Full load first, then capture ongoing changes | Most common — minimal downtime migration |

**Change Data Capture (CDC) Mechanisms by Source:**

| Source Engine | CDC Method |
|---------------|-----------|
| Oracle | LogMiner or Binary Reader (Binary Reader preferred for performance) |
| SQL Server | MS-Replication or MS-CDC |
| MySQL/MariaDB | Binary log (binlog) |
| PostgreSQL | Logical replication (test_decoding or pglogical plugin) |
| Amazon Aurora MySQL | Binary log |
| Amazon Aurora PostgreSQL | Logical replication |
| MongoDB | Change streams |
| SAP ASE | Transaction log |

**Supported Sources:**

| Source Type | Examples |
|-------------|---------|
| On-premises databases | Oracle, SQL Server, MySQL, PostgreSQL, MongoDB, SAP ASE, IBM Db2, MariaDB |
| EC2 databases | Any self-managed DB on EC2 |
| RDS databases | All RDS engines (MySQL, PostgreSQL, Oracle, SQL Server, MariaDB) |
| Aurora | MySQL and PostgreSQL compatible |
| Amazon S3 | As source (CSV, Parquet) |
| Azure SQL Database | As source |
| Azure SQL Managed Instance | As source |
| IBM Db2 (LUW, z/OS) | As source |
| DocumentDB | As source |

**Supported Targets:**

| Target Type | Examples |
|-------------|---------|
| RDS engines | MySQL, PostgreSQL, Oracle, SQL Server, MariaDB |
| Aurora | MySQL and PostgreSQL |
| Amazon Redshift | Data warehousing |
| Amazon S3 | Data lake (Parquet, CSV, JSON) |
| DynamoDB | NoSQL |
| Amazon OpenSearch | Search/analytics |
| Amazon Kinesis Data Streams | Real-time streaming |
| Amazon Neptune | Graph database |
| Amazon DocumentDB | MongoDB-compatible |
| Apache Kafka (MSK) | Event streaming |
| Amazon ElastiCache (Redis) | In-memory caching |

### Homogeneous vs. Heterogeneous Migrations

| Type | Definition | SCT Required? | Example |
|------|-----------|---------------|---------|
| **Homogeneous** | Same engine → same engine | NO | MySQL → RDS MySQL, Oracle → RDS Oracle |
| **Heterogeneous** | Different engine → different engine | YES | Oracle → Aurora PostgreSQL, SQL Server → Aurora MySQL |

### AWS Schema Conversion Tool (SCT)

**Purpose:** Convert database schema and code objects from one engine to another for heterogeneous migrations.

**What SCT Converts:**

| Object Type | Examples |
|-------------|---------|
| Schema | Tables, views, indexes, constraints, sequences |
| Stored procedures | PL/SQL → PL/pgSQL, T-SQL → PL/pgSQL |
| Functions | Scalar functions, table-valued functions |
| Triggers | Row-level, statement-level |
| Packages (Oracle) | Decomposed into PostgreSQL functions |
| Synonyms | Mapped to schemas/search paths |
| Materialized views | Converted or flagged for manual review |
| User-defined types | Mapped to target equivalents |

**SCT Architecture:**
```
Source DB ──► SCT (desktop app or DMS SC in cloud) ──► Conversion Report ──► Target DB
                         │                                    │
                   Assessment Report                   Action Items
                   (% auto-converted)                  (manual fixes needed)
```

**SCT Deployment Options:**

| Option | Description | Use Case |
|--------|-------------|----------|
| **Desktop Application** | Install on Windows/Linux/Mac | Full-featured; complex conversions |
| **DMS Schema Conversion (cloud)** | Managed service in DMS console | Simpler migrations; no desktop install needed |

**SCT Assessment Report:**
- Shows percentage of code automatically convertible
- Categories: Green (auto-convert), Yellow (simple manual fix), Red (significant rewrite)
- Identifies incompatible features
- Estimates effort for manual conversion
- Critical for planning and scoping heterogeneous migrations

**SCT Data Extraction Agents:**
- For large-scale data migrations (multi-TB)
- Extract data from source in parallel
- Store in S3 as intermediate staging
- Load into target from S3
- Faster than DMS for very large datasets
- Can use Snowball Edge as intermediate for offline transfer

### Replication Instance

**Sizing Considerations:**

| Factor | Impact |
|--------|--------|
| Number of tasks | More tasks = more CPU/memory needed |
| Data volume | Larger datasets need more memory for sorting/caching |
| Number of tables | More tables = more memory for metadata |
| LOB columns | Large objects consume significant memory |
| Transaction rate | High CDC throughput needs more CPU |
| Complex transformations | Filtering/transformations use CPU |

**Instance Classes:**

| Class | Use Case |
|-------|----------|
| dms.t3.micro/small/medium | Dev/test, small migrations (<10 tables, low throughput) |
| dms.c5.large/xlarge/2xlarge | Production, moderate workloads |
| dms.c5.4xlarge/9xlarge/12xlarge/18xlarge/24xlarge | Large-scale, high-throughput migrations |
| dms.r5.* | Memory-intensive (many LOBs, complex transforms) |

**Multi-AZ Replication Instance:**
- Synchronous standby in different AZ
- Automatic failover if primary AZ has issues
- Recommended for production/critical migrations
- Slight additional cost (2x instance + storage)
- No downtime during AZ failure — migration continues

### Key Limits & Quotas

| Resource | Default Limit |
|----------|---------------|
| Replication instances per account | 60 |
| Endpoints per account | 1,000 |
| Tasks per replication instance | Varies by instance size (no hard limit, but performance degrades) |
| Total storage per replication instance | 6 TB (gp2/gp3) |
| Source databases per task | 1 |
| Target databases per task | 1 |
| LOB max size (limited mode) | Configurable (default 32KB) |
| LOB max size (full mode) | Unlimited (but slower) |
| Free tier (DMS) | dms.t3.micro for 750 hours/month (12 months) |

### DMS Serverless

**AWS DMS Serverless:**
- Automatically provisions and scales capacity
- No need to choose replication instance size
- Scales up/down based on workload
- Pay per capacity unit used (DCU — DMS Capacity Units)
- Supports: Full Load, CDC, Full Load + CDC
- Min/Max DCU configurable (1-384 DCU)
- Ideal for: variable workloads, unknown sizing, burst traffic

**DMS Serverless vs. Provisioned:**

| Aspect | Serverless | Provisioned |
|--------|-----------|-------------|
| Sizing | Automatic | Manual (choose instance class) |
| Scaling | Auto (up/down) | Manual (modify instance) |
| Cost model | Per DCU-hour used | Per instance-hour (always on) |
| Multi-AZ | Built-in | Optional (select at creation) |
| Use case | Variable load, unknown requirements | Predictable, steady-state migrations |
| Supported tasks | Full Load, CDC, Full Load+CDC | All task types |

### Table Mappings & Transformations

**Selection Rules:**
- Include or exclude schemas, tables, columns
- Filter rows using WHERE-like conditions
- Wildcard support (% for multiple characters)

**Transformation Rules:**
- Rename schemas, tables, columns
- Add columns (with expressions)
- Remove columns
- Convert data types
- Add prefix/suffix to table names
- Change case (upper/lower)

**Example (JSON):**
```json
{
  "rules": [
    {
      "rule-type": "selection",
      "rule-action": "include",
      "object-locator": {
        "schema-name": "hr",
        "table-name": "%"
      }
    },
    {
      "rule-type": "transformation",
      "rule-action": "rename",
      "rule-target": "schema",
      "object-locator": {
        "schema-name": "hr"
      },
      "value": "human_resources"
    }
  ]
}
```

### LOB (Large Object) Handling

| Mode | Behavior | Performance | Use Case |
|------|----------|-------------|----------|
| **Don't include LOBs** | Skip LOB columns entirely | Fastest | LOBs not needed in target |
| **Full LOB mode** | Migrate all LOBs regardless of size | Slowest (row-by-row) | Must preserve all LOBs, size unknown |
| **Limited LOB mode** | Truncate LOBs exceeding configured max size | Fast (bulk) | Know max LOB size; acceptable to truncate |
| **Inline LOB mode** | Small LOBs inline (bulk), large LOBs via row-by-row | Balanced | Mixed LOB sizes; best of both worlds |

### DMS Validation

- **Data validation:** Compares source and target row counts and data
- Runs in background after full load or during CDC
- Reports: matching rows, mismatched rows, missing rows
- Identifies specific records with differences
- Can output failures to a control table for investigation
- Adds overhead (~10% performance impact)

---

## 2. Design Patterns & Best Practices

### Migration Pattern: Homogeneous (Same Engine)

```
Source (Oracle on-prem) ──► DMS Replication Instance ──► Target (RDS Oracle)
                                                         
No SCT needed. DMS handles data migration directly.
Schema: Use native tools (mysqldump, pg_dump, Oracle Data Pump) for schema,
        then DMS for data + CDC.
```

**Best Practice:** For homogeneous migrations, use native DB tools for schema migration and DMS only for data replication. Native tools preserve all objects more reliably.

### Migration Pattern: Heterogeneous (Different Engine)

```
Step 1: SCT converts schema
Source (Oracle) ──► SCT ──► Assessment Report ──► Target Schema (Aurora PostgreSQL)
                              │
                        Manual fixes for
                        unconvertible items

Step 2: DMS migrates data
Source (Oracle) ──► DMS (Full Load + CDC) ──► Target (Aurora PostgreSQL)
                                               (schema already exists from Step 1)
```

**Best Practice:** Always run SCT assessment FIRST to understand conversion complexity before committing to a heterogeneous migration.

### Migration Pattern: Large-Scale (Multi-TB)

```
Option A: DMS with multiple tasks (parallel)
Source ──► Task 1 (tables A-F) ──► Target
       ──► Task 2 (tables G-M) ──► Target
       ──► Task 3 (tables N-Z) ──► Target

Option B: SCT Data Extraction Agents
Source ──► SCT Agents (parallel extract) ──► S3 ──► Target (bulk load)

Option C: Snowball + DMS CDC
Source ──► Snowball (initial bulk) ──► S3 ──► Target (bulk load)
       ──► DMS CDC (ongoing changes from day 1) ──► Target (applied after bulk load)
```

### Migration Pattern: Minimal Downtime

```
Timeline:
Day 1: Start DMS Full Load + CDC
Day 1-X: Full load completes; CDC catches up changes
Day X: CDC is caught up (replication lag = 0)
Cutover Window (minutes):
  1. Stop writes to source
  2. Wait for final CDC to drain (seconds)
  3. Verify target data consistency
  4. Switch application connection string to target
  5. Resume writes on target
Total downtime: minutes (time to drain + switch)
```

### Migration Pattern: Database Consolidation

```
Multiple sources → Single target

Source DB 1 (MySQL) ──► DMS Task 1 ──►
Source DB 2 (MySQL) ──► DMS Task 2 ──► Single Aurora MySQL
Source DB 3 (MySQL) ──► DMS Task 3 ──►

Use schema transformation rules to rename schemas to avoid conflicts.
```

### Migration Pattern: Database Fan-Out

```
Single source → Multiple targets

Source (Oracle) ──► DMS Task 1 ──► Aurora PostgreSQL (OLTP)
                ──► DMS Task 2 ──► Redshift (analytics)
                ──► DMS Task 3 ──► S3 (data lake)
                ──► DMS Task 4 ──► Kinesis (real-time streaming)
```

### Anti-Patterns

| Anti-Pattern | Why It's Wrong | Better Approach |
|-------------|----------------|-----------------|
| Using DMS for schema migration (heterogeneous) | DMS doesn't convert stored procedures/functions | Use SCT for schema, DMS for data |
| Single huge task for 500+ tables | Performance bottleneck; hard to troubleshoot | Split into multiple parallel tasks (50-100 tables each) |
| Under-sized replication instance | OOM errors, task failures, slow performance | Right-size based on table count, LOB usage, transaction rate |
| Full LOB mode for all tables | Extremely slow (row-by-row) | Use Limited/Inline LOB mode; set appropriate max size |
| Skipping validation | Silent data corruption/loss | Always enable validation for production migrations |
| Not testing CDC before cutover | Surprise issues during production cutover | Run CDC for days/weeks; verify replication lag = 0 |
| DMS for entire server migration | Migrates data only, not OS/apps | Use MGN for full server; DMS for databases only |
| Ignoring SCT assessment "red" items | Application breaks due to incompatible features | Fix red items before cutover; test thoroughly |

### Well-Architected Alignment

- **Reliability:** Multi-AZ replication instance; DMS auto-restart on failure; validation ensures data integrity
- **Security:** SSL/TLS for endpoints; KMS encryption at rest; IAM roles with least privilege
- **Cost:** Right-size replication instance; use serverless for variable workloads; stop tasks when done
- **Performance:** Parallel tasks; right-size instance; Limited LOB mode; minimize transformations
- **Operational Excellence:** CloudWatch metrics; event notifications; premigration assessment; runbooks

### Integration Points

| Service | Integration with DMS |
|---------|---------------------|
| CloudWatch | Replication metrics, task logs |
| CloudTrail | API audit logging |
| SNS | Event notifications (task state changes, errors) |
| KMS | Encryption of replication instance storage and endpoints |
| IAM | Service roles, endpoint access policies |
| S3 | Target for data lake migrations; source for imports |
| Kinesis | Target for CDC streaming |
| Redshift | Target for analytics migration |
| Secrets Manager | Store endpoint credentials (rotation support) |
| Migration Hub | Reports migration status/progress |

---

## 3. Security & Compliance

### Encryption

| Data State | Mechanism |
|------------|-----------|
| In transit (source → replication instance) | SSL/TLS (configurable per endpoint) |
| In transit (replication instance → target) | SSL/TLS (configurable per endpoint) |
| At rest (replication instance storage) | KMS encryption (enabled at creation; cannot add later) |
| At rest (S3 target) | SSE-S3, SSE-KMS, or SSE-C |
| Credentials storage | Encrypted by DMS; optionally use Secrets Manager |

**Critical:** Replication instance encryption must be enabled at creation time — cannot be added to existing instances. Plan ahead.

### SSL/TLS Configuration

| Engine | SSL Mode Options |
|--------|-----------------|
| MySQL/MariaDB | none, require, verify-ca, verify-full |
| PostgreSQL | none, require, verify-ca, verify-full |
| Oracle | SSL with wallet/certificate |
| SQL Server | SSL with certificate |
| MongoDB | SSL with CA certificate |

**Certificate Management:**
- Upload CA certificates to DMS
- Configure endpoint to use SSL
- Verify certificate chain for sensitive migrations
- Certificates must be imported before endpoint creation (or endpoint modified after)

### IAM Roles & Policies

| Role | Purpose |
|------|---------|
| `dms-vpc-role` | Allows DMS to manage ENIs in your VPC |
| `dms-cloudwatch-logs-role` | Allows DMS to write logs to CloudWatch |
| `dms-access-for-endpoint` | Access to S3, Kinesis, Redshift, or other target services |
| Custom endpoint roles | S3 bucket access, KMS key access, Redshift cluster access |

**IAM for S3 Target:**
```json
{
  "Effect": "Allow",
  "Action": [
    "s3:PutObject",
    "s3:DeleteObject",
    "s3:ListBucket"
  ],
  "Resource": [
    "arn:aws:s3:::target-bucket",
    "arn:aws:s3:::target-bucket/*"
  ]
}
```

### Network Security

- **Replication instance:** Deployed in your VPC (private subnets recommended)
- **Subnet group:** Must specify at least 2 AZs for Multi-AZ support
- **Security groups:** Control inbound/outbound from replication instance
- **Source connectivity:** VPN, Direct Connect, or VPC peering (for EC2/RDS sources)
- **Publicly accessible:** Optional — set to false for production (use VPN/DX for on-prem sources)
- **VPC endpoints (PrivateLink):** Not available for DMS replication instance directly; use VPN/DX

### Compliance Considerations

- **Data in transit:** Always enable SSL for regulated workloads
- **Data at rest:** Enable KMS encryption on replication instance at creation
- **Audit trail:** CloudTrail logs all DMS API calls
- **Data residency:** Replication instance and data remain in configured region
- **PHI/PII:** DMS processes data in memory — ensure replication instance is encrypted and network is private
- **Secrets:** Use Secrets Manager for endpoint credentials; supports auto-rotation

### Security Best Practices

1. Enable KMS encryption on replication instance at creation (can't add later)
2. Use SSL/TLS for all endpoint connections
3. Deploy replication instance in private subnets
4. Restrict security groups: only allow required database ports
5. Use Secrets Manager for credential storage and rotation
6. Enable CloudWatch logging for audit
7. Use Multi-AZ for production migrations (also improves availability)
8. Delete replication instances after migration completes (don't leave running)

---

## 4. Cost Optimization

### DMS Pricing Model

| Component | Pricing |
|-----------|---------|
| Replication instance | Per-hour (like EC2); varies by instance class |
| Storage (attached to instance) | Per GB-month (gp2) |
| Data transfer OUT | Standard AWS data transfer rates |
| Data transfer IN | FREE |
| DMS Serverless | Per DCU-hour consumed |
| Log storage (CloudWatch) | Standard CloudWatch Logs pricing |

### Free Tier

- **dms.t3.micro:** 750 hours/month free for 12 months
- **Storage:** 50 GB free for 12 months
- Enough for testing and small migrations
- After free tier: pay standard rates

### SCT Pricing

- **AWS SCT desktop application:** FREE (download and use)
- **DMS Schema Conversion (cloud):** FREE (part of DMS console)
- You only pay for the resources used (source/target infrastructure)

### Cost Traps

| Trap | Impact | Mitigation |
|------|--------|------------|
| Over-provisioned replication instance | Paying for unused capacity | Start small, monitor metrics, scale up if needed |
| Replication instance left running after migration | Ongoing per-hour charges | Delete instance after final validation |
| Multi-AZ for non-critical migrations | 2x instance cost | Use Multi-AZ only for production/critical migrations |
| Full LOB mode (performance) | Migration takes 10x longer = 10x instance cost | Use Limited/Inline LOB mode |
| Too many separate tasks on one instance | Need larger (expensive) instance | Consolidate related tables into fewer tasks |
| Not using DMS Serverless for spiky workloads | Paying for peak capacity 24/7 | Use Serverless for variable loads |
| Keeping CDC running indefinitely "just in case" | Ongoing charges for instance + storage | Plan cutover date; stop CDC after validation |
| Large CloudWatch log retention | Log costs accumulate | Set appropriate retention period (7-30 days) |

### Cost Optimization Strategies

1. **Right-size replication instance:** Start with c5.large, scale up only if metrics show bottlenecks
2. **Use DMS Serverless** for unpredictable workloads (auto-scales, pay per use)
3. **Consolidate tasks** where possible (fewer tasks = smaller instance needed)
4. **Limited LOB mode:** Dramatically faster = shorter instance runtime = lower cost
5. **Delete immediately** after migration validation passes
6. **Reserve capacity** (DMS RI) for long-running ongoing replications (30-60% savings)
7. **Minimize data transfer OUT** — use within same region; cross-region adds transfer cost
8. **Parallel full load** for faster completion (shorter instance runtime)
9. **Use free tier** for testing migration configurations before production

### Cost Comparison: DMS vs. Native Tools

| Scenario | DMS Cost | Native Tool Cost |
|----------|----------|------------------|
| One-time homogeneous migration | Instance hours during migration | Free (pg_dump, mysqldump) but requires downtime |
| Minimal-downtime migration | Instance hours (longer running CDC) | Expensive — requires application-level solution |
| Ongoing replication (analytics) | Continuous instance cost | Build custom CDC pipeline (dev cost) |
| Heterogeneous | Instance + SCT (free) | Manual conversion (significant dev cost) |

**Key insight:** DMS cost is almost always less than the developer time to build equivalent functionality manually.

---

## 5. High Availability, Disaster Recovery & Resilience

### Multi-AZ Replication Instance

| Feature | Details |
|---------|---------|
| Synchronous standby | Replica in different AZ |
| Automatic failover | If primary AZ fails, standby promotes |
| Failover time | Typically under 1 minute |
| DNS update | Endpoint DNS automatically points to new primary |
| Task impact | Task resumes from last checkpoint (may replay some CDC) |
| Cost | ~2x instance cost |
| When to use | Production migrations, critical databases, long-running CDC |

### Resilience Features

| Feature | Behavior |
|---------|----------|
| **Task auto-restart** | DMS restarts failed tasks automatically (configurable) |
| **CDC checkpointing** | DMS tracks position in source change log; resumes from last checkpoint |
| **Error handling** | Configurable: suspend task, log error and continue, or fail task |
| **Table recovery** | Individual tables can be reloaded without restarting entire task |
| **Network interruption** | DMS retries connection; CDC resumes from checkpoint |
| **Source failover** | If source DB fails over (e.g., RDS Multi-AZ), DMS reconnects to new primary |

### Monitoring for Resilience

**Critical CloudWatch Metrics:**

| Metric | Alert Threshold | Meaning |
|--------|----------------|---------|
| `CDCLatencySource` | >60 seconds | Source is generating changes faster than DMS reads |
| `CDCLatencyTarget` | >60 seconds | DMS is writing to target slower than changes arrive |
| `CDCIncomingChanges` | Growing continuously | Replication falling behind; may need larger instance |
| `FreeableMemory` | <256 MB | Instance running out of memory; scale up |
| `CPUUtilization` | >80% sustained | Instance undersized; scale up |
| `FreeStorageSpace` | <20% | Transaction log caching filling up; increase storage |
| `SwapUsage` | Any swap usage | Instance needs more memory |

### Cutover Resilience

**Minimal-Downtime Cutover Steps:**
1. Monitor `CDCLatencyTarget` = 0 (fully caught up)
2. Stop application writes to source
3. Wait for final CDC events to drain (verify `CDCIncomingChanges` = 0)
4. Run validation (compare row counts + checksums)
5. Switch application connection to target
6. Resume application writes on target
7. Keep DMS task running for N hours (can reverse if needed)
8. Stop DMS task after validation window

**Rollback Strategy:**
- Keep source database running and writable (don't decommission immediately)
- If issues found on target: switch application back to source
- For bidirectional sync (advanced): use DMS to replicate target → source during validation period

### DR Scenarios

| Scenario | DMS Behavior |
|----------|-------------|
| Source AZ failure (RDS Multi-AZ) | RDS fails over; DMS reconnects to new primary; brief CDC gap possible |
| Source region failure | Cannot continue; need source in DR region or separate DMS task |
| Target AZ failure (RDS Multi-AZ) | RDS target fails over; DMS reconnects; brief stall |
| Replication instance AZ failure (Multi-AZ) | Automatic failover to standby; task resumes from checkpoint |
| Replication instance AZ failure (Single-AZ) | Task stops; manual intervention to recreate/restart |
| Network failure | DMS retries; CDC resumes from checkpoint when connectivity restored |

### Best Practices for Resilience

1. **Always use Multi-AZ** for production migration replication instances
2. **Monitor CDCLatencyTarget** — growing latency means you can't cut over safely
3. **Test cutover procedure** on non-production first (practice the exact steps)
4. **Keep source running** for 1-2 weeks post-cutover as rollback safety net
5. **Use validation** to catch data integrity issues before cutover
6. **Set up SNS notifications** for task state changes and errors
7. **Design for resume:** CDC checkpointing means temporary failures don't require full restart

---

## 6. Exam-Focused Section

### Straightforward Questions (7 examples)

**Q1:** A company wants to migrate their on-premises Oracle database to Amazon RDS for Oracle with minimal downtime. What AWS service should they use?

**A:** AWS DMS with Full Load + CDC. Homogeneous migration (Oracle → Oracle) so no SCT needed. Full Load migrates existing data; CDC captures ongoing changes for minimal-downtime cutover.

**Q2:** A company wants to migrate from Oracle to Aurora PostgreSQL. What combination of tools is required?

**A:** AWS SCT (to convert schema, stored procedures, functions from PL/SQL to PL/pgSQL) + AWS DMS (to migrate data with minimal downtime using Full Load + CDC).

**Q3:** What is the purpose of the DMS replication instance?

**A:** The replication instance is a managed EC2 instance that runs migration tasks. It connects to source and target endpoints, reads data from source, and writes to target. It handles caching, transformation, and CDC processing.

**Q4:** A company needs to migrate a 500GB PostgreSQL database to Aurora PostgreSQL with zero schema changes. Do they need SCT?

**A:** No. This is a homogeneous migration (PostgreSQL → Aurora PostgreSQL). Use DMS directly for data migration. Use native pg_dump/pg_restore for schema migration.

**Q5:** How does DMS achieve minimal downtime during migration?

**A:** DMS uses Full Load + CDC. Full Load copies existing data while the source remains operational. CDC continuously captures and replicates changes made during and after full load. At cutover, only seconds of final changes need to drain.

**Q6:** What encryption options does DMS provide?

**A:** SSL/TLS for data in transit (endpoint connections), KMS encryption for data at rest (replication instance storage). Replication instance encryption must be enabled at creation time.

**Q7:** A company wants DMS to automatically scale capacity based on migration workload without managing instance sizes. What feature should they use?

**A:** DMS Serverless. It automatically provisions and scales capacity (DCU) based on workload, eliminates manual instance sizing, and supports Full Load, CDC, and Full Load + CDC.

### Tricky / Scenario-Based Questions (7 examples)

**Q1:** A company is migrating from Oracle (with 200 stored procedures and 50 PL/SQL packages) to Aurora PostgreSQL. The SCT assessment shows 75% auto-conversion rate with 25% requiring manual intervention. The CTO wants to know if they can proceed. What should the solutions architect recommend?

**A:** 75% auto-conversion is typical for Oracle → PostgreSQL. Recommend proceeding with a phased approach: (1) SCT auto-converts 75%. (2) Developers manually convert the 25% (mainly complex PL/SQL packages). (3) Test all converted code against Aurora PostgreSQL. (4) Use DMS for data migration only after schema+code conversion is complete and tested.
- **Wrong:** "25% failure means migration isn't feasible" — 75% is actually a good conversion rate for Oracle
- **Wrong:** "DMS will handle the stored procedures" — DMS migrates DATA, not code objects
- **Keywords:** "stored procedures" + "PL/SQL packages" (code objects = SCT territory), "75% auto-conversion" (normal; 25% needs dev work)

**Q2:** A company is using DMS to replicate an on-premises MySQL database to Aurora MySQL for near-real-time analytics. After running for 3 weeks, they notice CDCLatencyTarget increasing steadily from 5 seconds to 120 seconds. The source database hasn't changed its write pattern. What's the most likely cause and fix?

**A:** The replication instance is likely undersized for the sustained CDC throughput. As transaction logs grow and more changes queue up, the instance can't keep pace. Fix: scale up the replication instance class (e.g., c5.large → c5.xlarge) or optimize the task (reduce transformations, use parallel apply for target).
- **Wrong:** "Source database is the bottleneck" — question says source write pattern hasn't changed
- **Wrong:** "Network issue" — latency increasing gradually (not spiking) suggests processing bottleneck
- **Keywords:** "CDCLatencyTarget increasing steadily" (processing bottleneck, not network), "3 weeks" (sustained load over time)

**Q3:** A healthcare company needs to migrate a 10TB Oracle database to Aurora PostgreSQL. The database contains BLOB columns storing medical images (average 50MB each). The initial DMS full load is estimating 30 days. How can the solutions architect speed this up?

**A:** The slow performance is caused by Full LOB mode processing BLOBs row-by-row. Solutions: (1) Use SCT Data Extraction Agents for parallel extraction to S3, then bulk load into Aurora. (2) If using DMS: split large tables with LOBs into separate tasks; use Limited LOB mode (set max to 50MB) if images are known to be ≤50MB. (3) Consider migrating LOBs to S3 separately and storing S3 references in the database instead.
- **Wrong:** "Just use a larger replication instance" — helps somewhat but Full LOB mode is inherently row-by-row regardless of instance size
- **Wrong:** "Use Snowball for the whole database" — Snowball works for files/objects, not structured database data with ongoing changes
- **Keywords:** "10TB" + "BLOB" + "50MB each" + "30 days" (LOB handling is the bottleneck; size + LOBs = SCT extraction agents or architecture change)

**Q4:** A company is migrating SQL Server to Aurora PostgreSQL. After SCT schema conversion and DMS full load + CDC, the application team reports that queries using SQL Server-specific features (CROSS APPLY, OUTPUT clause, recursive CTEs with FOR XML) produce errors on Aurora PostgreSQL. Who is responsible for fixing this?

**A:** The APPLICATION code must be rewritten. SCT converts database-side code (stored procedures, functions, triggers). Application-side SQL queries embedded in application code must be manually converted by developers. SCT can assess application SQL (using the application SQL conversion feature), but application code changes are a development effort.
- **Wrong:** "SCT should have converted these" — SCT converts DB objects, not application code
- **Wrong:** "DMS should transform queries" — DMS migrates data, not SQL queries
- **Keywords:** "queries" + "application team" (application-side SQL), "SQL Server-specific features" (must be rewritten for PostgreSQL)

**Q5:** A company has an ongoing DMS replication from on-premises PostgreSQL to Aurora PostgreSQL for analytics. The source DBA needs to run ALTER TABLE to add a column to a heavily-used table. What impact will this have on DMS replication?

**A:** DMS handles most DDL changes automatically for homogeneous migrations (PostgreSQL → Aurora PostgreSQL). Adding a column is typically replicated to the target. However: (1) Large ALTER TABLE operations may cause temporary CDC lag. (2) Some DDL changes may require task restart. (3) Best practice: test the DDL change in non-prod DMS first, and monitor `CDCLatencyTarget` during the change. For some engines, DDL replication must be explicitly enabled in task settings.
- **Wrong:** "DMS will break and require full reload" — DDL handling has improved; most basic DDL is handled
- **Wrong:** "DDL is always handled seamlessly" — complex DDL (rename table, change column type) may cause issues
- **Keywords:** "ALTER TABLE" + "ongoing replication" (DDL during CDC scenario), "add a column" (simple DDL — usually handled)

**Q6:** A company wants to migrate their production Oracle database (2TB) to Aurora PostgreSQL with less than 1 hour of downtime. They also need to validate data integrity before cutover. The SCT conversion is already complete. What's the optimal DMS strategy?

**A:** Use DMS Full Load + CDC with validation enabled. Timeline: (1) Start Full Load + CDC (data migration begins, takes ~12-24 hours for 2TB). (2) CDC catches up with ongoing changes. (3) Validation runs in background, comparing source and target. (4) When CDCLatencyTarget = 0 AND validation shows 100% match: initiate cutover window. (5) Stop source writes → drain final CDC → switch to target. Total downtime: minutes (not hours).
- **Wrong:** "Full Load only with 1-hour downtime window" — 2TB full load takes much longer than 1 hour
- **Wrong:** "Skip validation to save time" — question requires data integrity verification
- **Keywords:** "less than 1 hour downtime" (CDC required for minimal downtime), "validate data integrity" (enable DMS validation), "2TB" (hours for full load; need CDC to keep target current)

**Q7:** A company runs DMS replication from an on-premises SQL Server to S3 (data lake). They need the data in S3 to be in Parquet format, partitioned by date, and each file should be approximately 128MB for optimal Athena query performance. Can DMS do this?

**A:** Yes. DMS supports S3 as a target with configurable settings: (1) `DataFormat`: set to "parquet". (2) `DatePartitionEnabled`: set to true for date-based partitioning. (3) `MaxFileSize`: set to 131072 (KB) for ~128MB files. (4) `TimestampColumnName`: specify the date column for partitioning. DMS natively supports Parquet output with date partitioning to S3.
- **Wrong:** "DMS only outputs CSV to S3" — DMS supports CSV, Parquet, and JSON for S3 targets
- **Wrong:** "Need Glue ETL after DMS" — DMS can produce Parquet directly; no post-processing needed
- **Keywords:** "S3" + "Parquet" + "partitioned by date" + "file size" (all configurable in DMS S3 target settings)

### Common Exam Traps & Pitfalls

| Trap | Reality |
|------|---------|
| "DMS converts stored procedures" | NO — DMS migrates DATA only; SCT converts code objects |
| "SCT is needed for all migrations" | NO — only for heterogeneous (different engine) migrations |
| "DMS requires downtime" | NO — Full Load + CDC provides minimal downtime (minutes at cutover) |
| "Replication instance encryption can be added later" | NO — must enable at creation time; cannot modify |
| "DMS handles schema migration" | Partially — can create basic tables, but use native tools or SCT for complete schema |
| "Larger instance always = faster" | Not always — LOB mode, network, and source read speed are often the real bottlenecks |
| "DMS Serverless replaces provisioned completely" | Not yet — some features/configurations only available on provisioned |
| "DMS to S3 only supports CSV" | NO — supports CSV, Parquet, and JSON |
| "CDC requires source downtime to enable" | Depends on engine — some need binlog enabled (may require restart), others don't |
| "DMS validates referential integrity" | NO — validates row counts and data values, not FK relationships |
| "One DMS task = one table" | NO — one task can handle hundreds of tables (use table mappings) |
| "SCT is a paid service" | NO — SCT desktop tool and DMS Schema Conversion are free |

---

## 7. Cheat Sheet

### Must-Know Facts

- **DMS = data migration only** (not schema, not application code, not stored procedures)
- **SCT = schema + code conversion** (free tool, used for heterogeneous migrations)
- **Homogeneous:** Same engine → same engine = DMS only (no SCT)
- **Heterogeneous:** Different engine → different engine = SCT + DMS
- **Full Load + CDC = minimal downtime** (most common pattern on exam)
- **CDC mechanisms:** Oracle (LogMiner/BinaryReader), MySQL (binlog), PostgreSQL (logical replication)
- **Replication instance encryption:** Must enable at creation — CANNOT add later
- **Multi-AZ replication instance:** Automatic failover; recommended for production
- **LOB handling:** Full mode = slow (row-by-row); Limited mode = fast (bulk); Inline = balanced
- **DMS Serverless:** Auto-scales DCU; no instance sizing; pay per use
- **Validation:** Compares source/target data; adds ~10% overhead; catches integrity issues
- **Free tier:** dms.t3.micro 750 hours/month for 12 months; SCT is always free
- **S3 target:** Supports CSV, Parquet, JSON; date partitioning; configurable file size
- **Table mappings:** Selection (include/exclude) + Transformation (rename, filter, convert)

### Key Differentiators

| DMS vs. MGN | DMS = database data; MGN = entire server (OS + apps + data) |
|---|---|
| **DMS vs. DataSync** | DMS = structured database data with CDC; DataSync = files/objects |
| **DMS vs. native tools** | DMS = minimal downtime (CDC); native = simpler but requires downtime |
| **SCT vs. DMS** | SCT = schema/code conversion; DMS = data movement |
| **Full Load vs. CDC** | Full Load = one-time copy; CDC = ongoing changes only |
| **Full Load + CDC** | Both: initial copy + ongoing sync = minimal downtime migration |
| **DMS Provisioned vs. Serverless** | Provisioned = manual sizing, all features; Serverless = auto-scale, simpler |
| **Multi-AZ vs. Single-AZ (DMS)** | Multi-AZ = automatic failover, higher cost; Single-AZ = manual recovery |
| **Full LOB vs. Limited LOB** | Full = all sizes, row-by-row (slow); Limited = truncates, bulk (fast) |

### Exam Keyword Triggers

| If the question says... | Think... |
|------------------------|----------|
| "Minimal downtime" + "database migration" | DMS Full Load + CDC |
| "Oracle to Aurora PostgreSQL" | SCT (schema) + DMS (data) |
| "MySQL to RDS MySQL" | DMS only (homogeneous, no SCT) |
| "Stored procedures need conversion" | SCT (not DMS) |
| "Ongoing replication to data lake" | DMS CDC → S3 (Parquet) |
| "Real-time analytics from production DB" | DMS CDC → Redshift or S3 |
| "Validate data after migration" | DMS Validation feature |
| "Database with large BLOBs (multi-GB)" | Watch for LOB mode discussion; consider SCT extraction agents |
| "Schema conversion assessment" | SCT Assessment Report |
| "Scale migration capacity automatically" | DMS Serverless |
| "Production migration must survive AZ failure" | Multi-AZ replication instance |
| "Convert PL/SQL to PL/pgSQL" | SCT |
| "Encrypt replication data" | KMS (at creation) + SSL/TLS (endpoints) |
| "Stream database changes to Kinesis" | DMS with Kinesis target (CDC mode) |

### Decision Flowchart

```
What are you migrating?
│
├── Entire server (OS + apps + DB) → AWS MGN (not DMS)
│
├── Files/objects (NFS, SMB, S3) → AWS DataSync (not DMS)
│
└── DATABASE
    │
    ├── Same engine? (e.g., MySQL → RDS MySQL)
    │   ├── Schema: Use native tools (pg_dump, mysqldump, Data Pump)
    │   └── Data: Use DMS (Full Load + CDC for minimal downtime)
    │
    └── Different engine? (e.g., Oracle → Aurora PostgreSQL)
        ├── Schema: SCT (assessment → convert → manual fixes)
        ├── Code: SCT (stored procs, functions, triggers)
        ├── Application SQL: Developer manual rewrite
        └── Data: DMS (Full Load + CDC)
            │
            ├── Small DB (<100GB) → Single DMS task
            ├── Medium DB (100GB-1TB) → Multiple parallel tasks
            └── Large DB (>1TB) with LOBs → SCT extraction agents or Snowball + DMS CDC
```

---

## 8. Additional Deep-Dive

### 10 More Tricky Scenario-Based Questions

**Q1:** A company has a 5TB Oracle data warehouse that they want to migrate to Amazon Redshift. The warehouse has complex PL/SQL ETL procedures that run nightly. What's the migration approach?

**A:** Use SCT to convert Oracle schema to Redshift schema (tables, views). Use SCT to assess ETL stored procedures — but note that PL/SQL → Redshift SQL conversion is limited (Redshift doesn't support stored procedures the same way). Recommend: rewrite ETL using AWS Glue (PySpark) or Step Functions + SQL. Use DMS for data migration (Oracle → Redshift is supported). For ongoing ETL: implement in Glue, not Redshift stored procedures.
- **Wrong:** "SCT converts ETL procedures to Redshift" — Redshift has limited procedural SQL support; complex PL/SQL ETL usually requires rewrite
- **Wrong:** "Use DMS for ETL too" — DMS migrates data, doesn't replicate ETL logic
- **Keywords:** "data warehouse" + "Oracle" + "Redshift" (heterogeneous + analytics), "PL/SQL ETL procedures" (must be rewritten, not just converted)

**Q2:** A company is running DMS Full Load + CDC from on-premises PostgreSQL to Aurora PostgreSQL. During the full load phase, they notice that new rows inserted into the source after the full load started for a table are missing from the target. What's happening?

**A:** This shouldn't happen with Full Load + CDC — DMS should capture inserts via CDC even during full load. Most likely cause: logical replication (CDC prerequisite for PostgreSQL) is not properly configured on the source. Check: (1) `wal_level = logical` (requires restart if changed). (2) `max_replication_slots` is sufficient. (3) The DMS user has REPLICATION privilege. (4) Publication exists for the required tables. If CDC isn't capturing, only the point-in-time full load data exists.
- **Wrong:** "This is normal — CDC starts after full load" — No, with Full Load + CDC, DMS captures changes from the moment the task starts, including during full load
- **Wrong:** "DMS doesn't support PostgreSQL CDC" — it does, via logical replication
- **Keywords:** "Full Load + CDC" + "new rows missing" (CDC not working; likely source configuration issue)

**Q3:** A company needs to migrate 50 different MySQL databases (one per tenant) from on-premises to individual Aurora MySQL clusters (one per tenant) in AWS. Each database is 20-50GB. What's the most efficient DMS architecture?

**A:** Create one replication instance (appropriately sized, e.g., c5.2xlarge) with 50 separate migration tasks — one per tenant database. Each task maps one source database to one Aurora target. Run tasks in parallel (staggered to manage initial load). Alternative for even faster: use 2-3 replication instances to distribute load. Use table mappings to select specific schemas per task.
- **Wrong:** "One replication instance per database" (50 instances) — wasteful and hits account limits
- **Wrong:** "One task with all 50 databases" — can't target 50 different Aurora clusters from one task (one target per task)
- **Keywords:** "50 different databases" + "individual Aurora clusters" (one task per tenant, shared replication instance)

**Q4:** A financial company is using DMS to migrate from SQL Server to Aurora PostgreSQL. They have a critical requirement: during cutover, zero transactions can be lost. The application processes financial transactions continuously (no maintenance window). How should they handle the cutover?

**A:** Use DMS Full Load + CDC with the following cutover approach: (1) Wait until CDCLatencyTarget = 0. (2) Instead of stopping the application: enable a "read-only" mode or queue transactions in a message queue (SQS). (3) Final CDC drains in seconds. (4) Validate row counts and checksums. (5) Switch application to target Aurora. (6) Replay any queued transactions. This ensures ZERO data loss with ZERO transaction loss during the brief cutover window.
- **Wrong:** "CDC guarantees zero loss automatically" — CDC can miss the very last transactions if app writes during DNS switch
- **Wrong:** "Use DMS with bidirectional replication" — DMS doesn't support bidirectional for heterogeneous migrations
- **Keywords:** "zero transactions lost" (queue pattern), "no maintenance window" (can't just stop; need queue/read-only approach), "financial transactions" (data integrity critical)

**Q5:** After migrating from Oracle to Aurora PostgreSQL using SCT + DMS, the application team reports that database performance is significantly worse on Aurora. Queries that ran in 100ms on Oracle now take 2+ seconds. DMS replication itself completed successfully. What should the solutions architect investigate?

**A:** Performance issues after heterogeneous migration are commonly caused by: (1) Missing indexes — SCT may not convert all Oracle indexes (especially function-based indexes). (2) Statistics not gathered — Aurora query planner needs ANALYZE run on migrated tables. (3) Oracle hints not applicable — PostgreSQL uses different query planning. (4) Stored procedure conversion inefficiencies (SCT auto-conversion may not be optimal). (5) Different data type mappings (e.g., NUMBER → NUMERIC may have performance differences). (6) Connection pooling differences. Run EXPLAIN ANALYZE on slow queries; add missing indexes; run ANALYZE on all tables.
- **Wrong:** "DMS corrupted the data" — DMS migrated successfully; this is a target performance issue
- **Wrong:** "Aurora is inherently slower than Oracle" — Aurora can match Oracle performance with proper optimization
- **Keywords:** "performance worse on Aurora" + "DMS completed successfully" (post-migration optimization needed), "100ms → 2+ seconds" (missing indexes or stale statistics)

**Q6:** A company has a SQL Server database with 500 tables. They want to migrate to Aurora MySQL but only need 200 specific tables. The remaining 300 tables are historical data that should go to S3 for cold storage. How should this be configured?

**A:** Create two DMS tasks on the same replication instance: Task 1: Selection rules include the 200 operational tables → Target: Aurora MySQL. Task 2: Selection rules include the 300 historical tables → Target: S3 (Parquet format for query efficiency with Athena). Both tasks use SCT for schema conversion. The S3 target can be partitioned for efficient querying.
- **Wrong:** "Need two replication instances" — one instance can run multiple tasks to different targets
- **Wrong:** "DMS can't route different tables to different targets" — separate tasks with different endpoints achieve this
- **Keywords:** "200 tables to Aurora" + "300 tables to S3" (two tasks, different targets), "historical data" + "cold storage" (S3 in Parquet)

**Q7:** A company starts a DMS migration task (Full Load + CDC) for a 1TB MySQL database. After 8 hours, the full load is at 60% but the replication instance runs out of storage (Free Storage Space = 0). The task stops. What happened and how do they fix it?

**A:** During Full Load + CDC, DMS must cache CDC transactions (changes happening during full load) until full load completes and CDC can apply them. For a large, write-heavy database, this cache grows continuously during the long full load. Fix: (1) Increase replication instance storage (modify instance). (2) Use a larger instance class with more storage. (3) Split into multiple tasks (parallel full load = faster = less CDC cache needed). (4) Consider running full load first (without CDC), then starting CDC task separately if acceptable.
- **Wrong:** "The database itself is too large" — 1TB fits; the issue is CDC cache during slow full load
- **Wrong:** "DMS has a 1TB limit" — no such limit; storage is the constraint
- **Keywords:** "out of storage" + "Full Load + CDC" + "8 hours at 60%" (CDC cache growing during long full load)

**Q8:** A company needs to keep an on-premises Oracle database synchronized with Aurora PostgreSQL indefinitely (not a one-time migration). The Oracle source receives ~1 million writes per day. Every month, they also need to add new tables to replication. What architecture should the solutions architect recommend?

**A:** Use DMS with ongoing CDC (heterogeneous ongoing replication is supported). Architecture: SCT converts schema for new tables as they're added. DMS task with Full Load + CDC initially, then ongoing CDC. For adding new tables monthly: modify the existing DMS task's table mappings to include new tables, or create additional tasks. Use Multi-AZ replication instance for resilience. Monitor CDCLatencyTarget continuously. Consider DMS Serverless for the variable load pattern (monthly additions may spike load).
- **Wrong:** "DMS only supports one-time migrations" — ongoing replication is a core DMS use case
- **Wrong:** "Need to restart replication for new tables" — can add tables to existing task (table reload) or add new tasks
- **Keywords:** "indefinitely" + "synchronized" (ongoing replication, not one-time), "add new tables monthly" (dynamic table mappings), "Oracle to Aurora PostgreSQL" (heterogeneous ongoing CDC)

**Q9:** A company is using DMS to migrate from MongoDB to Amazon DocumentDB. The MongoDB source uses replica sets for HA. During the migration, the MongoDB primary fails and a secondary is promoted. What happens to DMS replication?

**A:** DMS supports MongoDB replica set failover. When configured with the replica set connection string (multiple hosts), DMS automatically detects the new primary and reconnects. CDC continues from the last checkpoint. There may be brief latency spike but no data loss. If configured with only the primary host (not replica set string), DMS loses connection and the task fails — requiring manual intervention to point to new primary.
- **Wrong:** "DMS always handles this automatically" — only if configured with replica set connection string
- **Wrong:** "Full reload required after failover" — CDC checkpointing allows resume without full reload
- **Keywords:** "MongoDB" + "replica sets" + "primary fails" (replica set-aware connectivity), "secondary promoted" (DMS must know about replica set topology)

**Q10:** A company completed a DMS migration from Oracle to Aurora PostgreSQL. Post-migration validation shows that 99.8% of rows match, but 0.2% (approximately 10,000 rows) show data mismatches. The mismatched rows all contain Oracle NUMBER columns that were mapped to PostgreSQL NUMERIC. What's the likely cause?

**A:** Oracle NUMBER without precision/scale specification allows up to 38 digits of precision. PostgreSQL NUMERIC also supports this, but the DMS conversion may truncate trailing decimal places during type mapping, especially if the source column has implicit precision behavior. Fix: (1) Check DMS type mapping rules for NUMBER → NUMERIC conversion. (2) Review if custom precision was specified in table mappings. (3) The 10,000 rows likely have values with more decimal places than the configured mapping allows. (4) Adjust data type mapping in DMS task settings or add explicit transformation rules to preserve full precision.
- **Wrong:** "DMS has a bug" — this is a known data type mapping consideration, not a bug
- **Wrong:** "Reload all data" — need to fix the mapping first, then reload only affected tables
- **Keywords:** "NUMBER → NUMERIC" + "data mismatches" + "0.2%" (data type precision mismatch during heterogeneous migration)

### Compare: DMS Migration Modes

| Aspect | Full Load Only | CDC Only | Full Load + CDC |
|--------|---------------|----------|-----------------|
| **Initial data** | Yes (point-in-time copy) | No (assumes target has data) | Yes |
| **Ongoing changes** | No | Yes (from specified point) | Yes (from task start) |
| **Downtime** | Yes (stop writes before start; switch after complete) | N/A (ongoing) | Minimal (minutes at cutover) |
| **Use case** | Dev/test refresh, acceptable downtime | Add replication to already-migrated target | Production migration |
| **Duration** | Hours-days (depends on size) | Indefinite (ongoing) | Initial load + indefinite CDC |
| **Rollback** | Drop target, restart | Stop task | Stop task; source unchanged |
| **Exam default answer** | Rarely (only if "downtime acceptable") | Rarely (special case) | Most common exam answer |

### Compare: DMS vs. Native Database Migration Tools

| Aspect | AWS DMS | Native Tools (pg_dump, mysqldump, Data Pump) |
|--------|---------|----------------------------------------------|
| **Downtime** | Minutes (CDC) | Hours (must stop writes during export/import) |
| **Heterogeneous support** | Yes (any supported → any supported) | No (same engine only) |
| **Schema migration** | Basic (tables only) | Complete (all objects) |
| **Ongoing replication** | Yes (CDC) | No (one-time operation) |
| **Performance (large DB)** | Parallel load + streaming | Sequential (usually) |
| **LOB handling** | Configurable modes | Full (native format) |
| **Validation** | Built-in | Manual (write custom scripts) |
| **Cost** | Instance + storage charges | Free (built into DB) |
| **Complexity** | Medium (setup endpoints, tasks) | Low (single command) |
| **Best for** | Minimal-downtime, heterogeneous, ongoing replication | Simple homogeneous, schema migration, dev refreshes |

### Compare: DMS Target Options for Analytics/Data Lake

| Target | Format | Use Case | Exam Trigger |
|--------|--------|----------|-------------|
| **S3** | CSV, Parquet, JSON | Data lake, Athena queries, archive | "Data lake" / "query with Athena" / "long-term storage" |
| **Redshift** | Native Redshift tables | Data warehouse, complex analytics | "Analytics" / "data warehouse" / "BI dashboards" |
| **Kinesis Data Streams** | JSON records | Real-time streaming, event processing | "Real-time" / "streaming analytics" / "event-driven" |
| **OpenSearch** | JSON documents | Search, log analytics, dashboards | "Full-text search" / "log analytics" / "Kibana" |
| **Kafka (MSK)** | JSON/Avro | Event streaming, microservices | "Event streaming" / "Kafka" / "decouple producers" |

### When Does the SAP-C02 Exam Expect Me to Pick DMS vs. Other Tools?

**Pick DMS when:**
- "Database migration" (any context)
- "Minimal downtime" + database
- "Ongoing replication" from database to another store
- "Oracle/SQL Server → Aurora/RDS" (with or without SCT)
- "Database to S3/Redshift/Kinesis" (data pipeline)
- "Keep source and target in sync" during transition period
- "Heterogeneous database migration" (+ SCT)

**Pick SCT (with DMS) when:**
- "Different database engine" (Oracle → PostgreSQL, SQL Server → MySQL, etc.)
- "Convert stored procedures / functions / triggers"
- "Assessment report" for migration complexity
- "PL/SQL → PL/pgSQL" or "T-SQL → PL/pgSQL"
- "Schema conversion" between engines
- "Multi-TB migration" (SCT extraction agents)

**Pick Native Tools (not DMS) when:**
- "Schema migration" for homogeneous (pg_dump, mysqldump)
- "Simple one-time refresh" where downtime is acceptable
- "Export/import" with full fidelity of all database objects
- "Development environment refresh" (speed over zero-downtime)

**Pick MGN (not DMS) when:**
- "Migrate entire database server" (OS + DB + applications on same host)
- "Keep same database engine on EC2" (not moving to RDS/Aurora)
- "Server migration" (not just database data)

**Pick DataSync (not DMS) when:**
- "File shares" / "NFS" / "SMB" → S3/EFS/FSx
- "Object storage migration" (S3 to S3)
- Not structured database data

### All "Gotcha" Differences

| Gotcha | Explanation |
|--------|-------------|
| DMS ≠ schema migration tool | DMS creates basic target tables, but doesn't convert procedures/functions/triggers; use SCT or native tools |
| SCT ≠ data migration tool | SCT converts schema/code; DMS moves data. They complement each other |
| Homogeneous ≠ needs SCT | Same engine → same engine = DMS only; no conversion needed |
| Full Load ≠ downtime required | Full Load alone requires downtime; Full Load + CDC doesn't |
| LOB mode matters enormously | Full LOB = row-by-row (10x slower); exam questions about performance often hinge on this |
| Encryption at creation only | Cannot add KMS encryption to existing replication instance; must recreate |
| CDC needs source configuration | MySQL needs binlog ON; PostgreSQL needs wal_level=logical; Oracle needs supplemental logging |
| DMS validation ≠ referential integrity | Validates data values and row counts, NOT foreign key relationships |
| Replication instance ≠ source/target | It's a SEPARATE EC2 in YOUR VPC that proxies the data; not on source or target |
| S3 target supports Parquet | DMS can write Parquet directly (exam might test if you think CSV only) |
| DMS Serverless ≠ Lambda | Serverless means auto-sized replication capacity, not Lambda-style execution |
| SCT is free | Both desktop app and cloud DMS Schema Conversion — no license cost |
| Application SQL ≠ SCT's job | SCT converts DB-side code; app-embedded SQL must be manually rewritten by developers |
| Multi-AZ ≠ multi-region | Multi-AZ DMS = HA within region; doesn't replicate cross-region (use separate tasks for that) |
| CDC ≠ instant | CDC has latency (seconds-minutes); it's near-real-time, not synchronous replication |

### Decision Tree for DMS Configuration

```
What's your migration scenario?
│
├── Same engine (e.g., MySQL → MySQL)?
│   ├── Schema: Native tools (pg_dump, mysqldump, Data Pump)
│   └── Data: DMS Full Load + CDC
│
├── Different engine (e.g., Oracle → Aurora PostgreSQL)?
│   ├── Schema: SCT → Assessment → Convert → Manual fixes
│   ├── Code objects: SCT → Convert (may need manual rewrite)
│   ├── Application SQL: Developer rewrite (NOT DMS/SCT)
│   └── Data: DMS Full Load + CDC
│
├── Can you tolerate downtime?
│   ├── YES → Full Load only (simpler) OR native tools
│   └── NO → Full Load + CDC (minimal downtime)
│
├── Database size?
│   ├── Small (<100GB) → Single DMS task, standard instance
│   ├── Medium (100GB-1TB) → Multiple parallel tasks, c5.xlarge+
│   └── Large (>1TB with LOBs) → SCT Extraction Agents or Snowball + CDC
│
├── LOBs present?
│   ├── No LOBs → Standard (fastest)
│   ├── Known max size → Limited LOB mode (set max)
│   ├── Unknown/variable size → Inline LOB mode (balanced)
│   └── Must preserve all, any size → Full LOB mode (slowest)
│
├── Ongoing replication (not one-time)?
│   ├── Production-critical → Multi-AZ instance, monitor latency
│   ├── Variable workload → DMS Serverless
│   └── Analytics feed → CDC to S3 (Parquet) or Kinesis
│
└── Target is not a database?
    ├── Data lake → S3 (Parquet/CSV)
    ├── Search → OpenSearch
    ├── Analytics → Redshift
    ├── Streaming → Kinesis or MSK
    └── Graph → Neptune
```
