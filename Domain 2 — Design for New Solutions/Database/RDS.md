# AWS SAP-C02 Study Notes: RDS (Multi-AZ, Read Replicas, Proxy, Custom)

---

## 1. Core Concepts & Theory

### Amazon RDS Fundamentals

Amazon RDS (Relational Database Service) is a **managed relational database service** that handles provisioning, patching, backups, recovery, and scaling for common database engines. You focus on the application; AWS manages the infrastructure.

#### Supported Engines

| Engine | Versions | Key Note |
|--------|----------|----------|
| **MySQL** | 5.7, 8.0 | Most popular open-source engine on RDS |
| **PostgreSQL** | 13–16+ | Advanced features (JSONB, PostGIS, extensions) |
| **MariaDB** | 10.x | MySQL fork with enhancements |
| **Oracle** | 12c, 19c, 21c | License Included (LI) or Bring Your Own License (BYOL) |
| **SQL Server** | 2016–2022 | Express, Web, Standard, Enterprise editions |
| **Amazon Aurora** | MySQL-compatible, PostgreSQL-compatible | **Separate service** — covered in Aurora notes. Not "standard RDS" |

**Exam critical:** Aurora is NOT the same as "RDS MySQL/PostgreSQL." They have different architectures, HA models, pricing, and features. The exam tests this distinction heavily.

---

### RDS Multi-AZ

Multi-AZ provides **high availability** by maintaining a **synchronous standby replica** in a different Availability Zone.

#### Multi-AZ Deployment Types

| Type | Description | Engines | Failover Time |
|------|-------------|---------|---------------|
| **Multi-AZ Instance (classic)** | One primary + one synchronous standby in a different AZ. Standby is **NOT readable** | All engines | **60–120 seconds** |
| **Multi-AZ Cluster** | One primary writer + **two readable standby instances** in different AZs. Uses semisynchronous replication | MySQL, PostgreSQL only | **~35 seconds** (faster) |

#### Multi-AZ Instance (Classic) — Deep Dive

```
AZ-A: Primary DB Instance (read/write)
  ↕ synchronous replication
AZ-B: Standby Instance (NOT readable, NOT accessible)
```

- **Synchronous replication** — every write to the primary is replicated to the standby before being acknowledged to the application.
- Standby is **not accessible** for reads, writes, or any queries. It is purely for failover.
- **Single DNS endpoint** (CNAME). On failover, the DNS CNAME is flipped to the standby (now promoted to primary). Applications using the DNS endpoint are redirected automatically.
- Failover triggers:
  - Primary instance failure / AZ failure
  - Instance type change (maintenance)
  - Manual failover (`RebootDBInstance` with `ForceFailover=true`)
  - OS patching on the primary
  - Storage failure on the primary
- Failover time: typically **60–120 seconds** (DNS propagation + instance startup).
- **No cross-Region** — Multi-AZ is within a single Region.
- Storage: Same storage type/size for primary and standby.

#### Multi-AZ DB Cluster — Deep Dive

```
AZ-A: Writer Instance (read/write)
  ↕ semisynchronous replication
AZ-B: Reader Instance 1 (readable standby)
AZ-C: Reader Instance 2 (readable standby)
```

- **Two readable standby instances** — can serve read traffic (unlike classic Multi-AZ).
- Three endpoints:
  - **Cluster endpoint** (writer) — routes to the current primary.
  - **Reader endpoint** — load-balances across the two reader instances.
  - **Instance endpoints** — direct connection to each instance.
- **Semisynchronous replication** — at least one standby confirms receipt before write is acknowledged. Provides durability without the latency of full synchronous replication.
- Failover time: **~35 seconds** — faster than classic Multi-AZ because standbys are already running and caught up.
- Storage: Uses **optimized local storage** + writes forwarded to the writer. Not shared Aurora-style storage.
- **Supported only for MySQL and PostgreSQL** (not Oracle, SQL Server, or MariaDB).
- **Transaction log-based replication** — more efficient than classic block-level replication.

#### Multi-AZ Instance vs Multi-AZ Cluster

| Feature | Multi-AZ Instance | Multi-AZ Cluster |
|---------|-------------------|------------------|
| Standby readable? | **No** | **Yes** (two readers) |
| Replication method | Synchronous (block-level) | Semisynchronous (transaction log) |
| Failover time | 60–120 seconds | **~35 seconds** |
| Endpoints | Single (writer) | Three (writer, reader, instance) |
| Engines | All 5 engines | MySQL and PostgreSQL only |
| Standby count | 1 | 2 |
| Read scaling | No | Yes (reader endpoint) |

---

### RDS Read Replicas

Read replicas provide **read scaling** by creating asynchronous copies of the primary database.

#### Key Characteristics

| Property | Detail |
|----------|--------|
| **Replication** | **Asynchronous** — there is replication lag (seconds to minutes typically) |
| **Read/Write** | Read replicas are **read-only** (can be promoted to standalone read/write, but this breaks replication) |
| **Max replicas** | **5** per primary for MySQL, MariaDB, PostgreSQL, Oracle. **SQL Server does not support RDS read replicas** (use Always On AG instead) |
| **Cross-Region** | **Yes** — can create read replicas in different Regions (cross-Region read replica) |
| **Cross-Account** | Not natively supported (requires snapshot share + manual setup) |
| **Same AZ** | Yes — can be in the same AZ as primary |
| **Multi-AZ read replica** | Read replica itself can be **Multi-AZ** (has its own standby) |
| **Cascading replicas** | MySQL/MariaDB: replicas can have replicas (chain). PostgreSQL: replicas cannot chain. Oracle: no chaining |
| **Promotion** | Read replica can be **promoted to standalone primary**. This breaks replication permanently. Use for DR or migration |
| **Replication lag metric** | CloudWatch: `ReplicaLag` (seconds) |

#### Cross-Region Read Replicas

- Useful for **disaster recovery** (promote in DR Region) and **global read performance** (serve reads locally).
- Replication is **asynchronous and encrypted in transit** (SSL).
- **Cross-Region data transfer charges** apply.
- If source uses **SSE-KMS encryption**, you must specify a KMS key in the **destination Region** for the replica.
- Cross-Region replicas can be **promoted** to become independent standalone databases in the DR Region.

#### Read Replica vs Multi-AZ

| Aspect | Read Replica | Multi-AZ (Classic) | Multi-AZ Cluster |
|--------|-------------|-------------------|------------------|
| Purpose | **Read scaling** | **High availability** | **HA + read scaling** |
| Readable? | **Yes** | No | **Yes** |
| Replication | Asynchronous | Synchronous | Semisynchronous |
| Cross-Region | **Yes** | No | No |
| Automatic failover | No (manual promotion) | **Yes** | **Yes** |
| Replication lag | Seconds–minutes | None (synchronous) | Minimal |
| Max count | 5 | 1 standby | 2 readers |
| Endpoint | Separate read endpoint | Same DNS endpoint (failover) | Reader endpoint |

**You can combine both:** A Multi-AZ primary with up to 5 read replicas. The replicas replicate from the primary (not the standby).

---

### Amazon RDS Proxy

RDS Proxy is a **fully managed, highly available database proxy** that sits between your application and the RDS database. It pools and shares database connections.

#### Architecture

```
Application → RDS Proxy (connection pooling, failover) → RDS Primary / Read Replicas
```

#### Key Features

| Feature | Detail |
|---------|--------|
| **Connection pooling** | Maintains a pool of database connections. Application connections are multiplexed across pooled connections. Reduces DB connection overhead |
| **Failover acceleration** | RDS Proxy detects failover and redirects connections to the new primary **within seconds** (vs 60–120s for DNS-based failover). Reduces failover impact by **up to 66%** |
| **IAM authentication** | Supports **IAM database authentication** — applications use IAM credentials instead of DB passwords. Proxy handles credential management |
| **Secrets Manager integration** | Stores and rotates database credentials in **AWS Secrets Manager**. Proxy retrieves credentials automatically |
| **Multi-AZ** | Proxy is deployed across multiple AZs by default — highly available |
| **Pinning** | Some SQL features (e.g., session-level variables, prepared statements, temp tables) cause **connection pinning** — the proxy pins the application to a specific DB connection, reducing pooling efficiency |
| **Supported engines** | MySQL, PostgreSQL, MariaDB (and Aurora MySQL/PostgreSQL). **NOT Oracle, NOT SQL Server** |
| **Lambda integration** | Primary use case: **Lambda functions** connecting to RDS. Without proxy, each Lambda invocation opens a new DB connection → connection exhaustion. Proxy solves this |
| **Read/write splitting** | RDS Proxy supports endpoints for the writer AND reader (when using Aurora or Multi-AZ Cluster). Not automatic query routing — application must use the correct endpoint |
| **VPC-only** | RDS Proxy is only accessible from within the VPC (not publicly accessible) |
| **Idle connection timeout** | Configurable. Proxy closes idle connections to the DB, freeing resources |

#### When RDS Proxy Is the Exam Answer

- **Lambda + RDS**: Lambda creates connections per invocation → connection explosion. Proxy pools connections.
- **Frequent failover with minimal downtime**: Proxy reduces failover time from 60–120s to **seconds**.
- **Credential management**: IAM auth + Secrets Manager rotation through the proxy.
- **Applications with many short-lived connections**: Microservices, serverless architectures.

---

### Amazon RDS Custom

RDS Custom provides **managed database infrastructure** while giving you **OS and database-level access** (SSH, RDP, custom patches, specialized configurations).

#### Two Variants

| Variant | Description | OS Access | Engines |
|---------|-------------|-----------|---------|
| **RDS Custom for Oracle** | Full managed RDS with OS access. Install custom Oracle patches, modules, third-party software | **Yes** (SSH/SSM) | Oracle EE |
| **RDS Custom for SQL Server** | Full managed RDS with OS access. Install custom SQL Server components, CLR assemblies | **Yes** (RDP/SSM) | SQL Server EE |

#### Key Details

| Property | Detail |
|----------|--------|
| **OS access** | SSH (Oracle/Linux) or RDP (SQL Server/Windows) via **SSM Session Manager** |
| **Underlying EC2** | RDS Custom runs on EC2 instances you can see (unlike standard RDS). The instance is visible in the EC2 console |
| **Automation paused** | You can **pause RDS automation** (automated patching, snapshots) to make OS-level changes. Must resume automation when done |
| **Customization** | Install custom software, patches, OS configurations, third-party agents. Things not possible on standard RDS |
| **Shared responsibility** | AWS manages: hardware, networking, storage. You manage: OS patches (when automation is paused), custom software, database tuning |
| **Multi-AZ** | Supported for Oracle (not for SQL Server as of now — check latest docs) |
| **Read Replicas** | Supported for Oracle (not SQL Server) |
| **Backups** | Automated snapshots + manual snapshots. Can restore to any point within retention window |

#### When to Use RDS Custom

- **Oracle**: Applications requiring custom Oracle patches, Oracle Spatial, Oracle Text, or third-party tools installed at the OS level.
- **SQL Server**: Applications requiring CLR assemblies, custom SQL Agent jobs, linked servers, or third-party monitoring tools at the OS level.
- **When standard RDS is too restrictive** but you don't want to manage everything on EC2 yourself.
- **When self-managed EC2 is too much work** but you need some OS-level access.

#### RDS Custom vs Standard RDS vs EC2

| Aspect | Standard RDS | RDS Custom | Self-Managed on EC2 |
|--------|-------------|-----------|-------------------|
| OS access | **No** | **Yes** | Yes |
| DB engine management | AWS | Shared (AWS + you) | You |
| Hardware/storage | AWS | AWS | You (EBS choices) |
| Automated backups | Yes | Yes | You (scripts/tools) |
| Multi-AZ failover | Yes | Yes (Oracle) | You (Always On AG, Data Guard) |
| Customization | Limited (parameter groups) | Full OS access | Full control |
| Supported engines | All 5 | Oracle EE, SQL Server EE | Any |
| Use case | Most workloads | Specialized Oracle/SQL Server | Full control needed |

---

### Additional RDS Features (Exam Relevant)

#### Automated Backups

- **Retention period**: 0–35 days (0 disables backups). Default: 7 days.
- **Backup window**: Daily, configurable.
- **Point-in-time recovery (PITR)**: Restore to any second within the retention window. Creates a **new** DB instance (not in-place restore).
- Backups stored in S3 (AWS-managed, not visible to you).
- **Transaction logs** backed up every 5 minutes.
- Automated backups are deleted when you delete the DB instance (unless you take a final snapshot).

#### Manual Snapshots

- User-initiated, retained **indefinitely** until you delete them.
- Can be **shared cross-account** and **copied cross-Region**.
- **Encrypted snapshots** require sharing the KMS key for cross-account sharing.
- Restoring from snapshot creates a **new** DB instance.

#### Parameter Groups and Option Groups

- **Parameter Group**: Database engine configuration parameters (e.g., `max_connections`, `innodb_buffer_pool_size`). Custom groups can be created and applied.
- **Option Group**: Engine-specific features (e.g., Oracle TDE, SQL Server TDE, Oracle APEX). Applied to the DB instance.
- Parameter changes may require a **reboot** depending on the parameter (static vs dynamic).

#### RDS Event Notifications

- Subscribe to events via **Amazon SNS**: failover, backup, maintenance, configuration changes.
- Integrate with EventBridge for more complex routing.

#### Enhanced Monitoring

- **OS-level metrics** at 1-second granularity: CPU, memory, file system, disk I/O, processes.
- Uses a **CloudWatch Agent on the RDS instance** — NOT the standard CloudWatch metrics (which are at 1-minute granularity and DB-level only).
- Costs extra based on data volume.

#### Performance Insights

- **Database performance monitoring** — visualizes database load, top SQL queries, waits.
- Identifies which queries are causing load.
- Available for all engines.
- Free tier: 7 days retention. Paid tier: up to 2 years.

---

### Default Limits Worth Memorizing

| Resource | Limit | Adjustable? |
|----------|-------|-------------|
| DB instances per Region | **40** | Yes |
| Read replicas per primary | **5** (MySQL, PostgreSQL, MariaDB, Oracle) | No |
| SQL Server read replicas | **Not supported** (use Always On AG) | N/A |
| Max storage (General Purpose/Provisioned IOPS) | **64 TiB** (MySQL, PostgreSQL, MariaDB, Oracle). **16 TiB** (SQL Server) | No |
| Max Provisioned IOPS (io1) | **256,000** (on supported instance types) | No |
| Automated backup retention | **0–35 days** | No |
| Transaction log backup frequency | Every **5 minutes** | No |
| Multi-AZ Cluster standby count | **2** (reader instances) | No |
| Multi-AZ Cluster engines | MySQL, PostgreSQL only | N/A |
| RDS Proxy supported engines | MySQL, PostgreSQL, MariaDB | N/A |
| RDS Custom engines | Oracle EE, SQL Server EE | N/A |
| Max manual snapshots per Region | **100** (adjustable) | Yes |
| Max parameter groups per Region | **50** | Yes |
| Cross-Region read replica max | **5** per primary | No |

---

## 2. Design Patterns & Best Practices

### When to Use Which Feature

| Scenario | Feature | Why |
|----------|---------|-----|
| HA with automatic failover (same Region) | **Multi-AZ** | Synchronous standby, automatic DNS failover |
| HA + read scaling (MySQL/PostgreSQL) | **Multi-AZ Cluster** | Two readable standbys, faster failover |
| Read-heavy workloads needing horizontal scaling | **Read Replicas** | Up to 5 asynchronous read replicas |
| Cross-Region read performance | **Cross-Region Read Replicas** | Serve reads from a closer Region |
| Cross-Region DR | **Cross-Region Read Replica** (promote on failover) | RPO: seconds–minutes of replication lag |
| Lambda → RDS connectivity | **RDS Proxy** | Connection pooling prevents Lambda connection explosion |
| Faster failover for applications | **RDS Proxy** | Reduces failover impact to seconds |
| Credential rotation without app changes | **RDS Proxy + Secrets Manager** | Proxy handles credential refresh transparently |
| Oracle/SQL Server with OS access | **RDS Custom** | OS-level customization with managed infrastructure |
| Full database control | **Self-managed on EC2** | Complete control over everything |

### Anti-Patterns

| Anti-Pattern | Why It's Wrong | Correct Approach |
|-------------|---------------|------------------|
| Multi-AZ for read scaling | Classic Multi-AZ standby is NOT readable | Use Read Replicas or Multi-AZ Cluster |
| Read Replicas for HA/failover | Promotion is manual and breaks replication | Use Multi-AZ for automatic failover |
| RDS Proxy for Oracle or SQL Server | Not supported | Use connection pooling at application level or use pgBouncer/ProxySQL |
| Cross-Region Multi-AZ | Multi-AZ is single-Region only | Use cross-Region read replicas for multi-Region DR |
| Standard RDS for custom Oracle patches | No OS access on standard RDS | Use RDS Custom for Oracle |
| RDS for workloads needing >64 TiB storage | RDS max is 64 TiB | Use Aurora (128 TiB) or self-managed on EC2 |
| Using RDS when you need multi-master writes | RDS doesn't support multi-master | Use Aurora Multi-Master (deprecated) or DynamoDB Global Tables |
| Public RDS endpoint in production | Security risk | Deploy in private subnet, access via VPC, use RDS Proxy |

### Common Architectural Patterns

#### Standard Production Setup
```
Application (EC2/ECS/Lambda)
  → RDS Proxy (connection pooling, IAM auth)
    → RDS MySQL Multi-AZ (writer endpoint)
    → Read Replicas (reader endpoint) for reporting/analytics
  → Secrets Manager (credential rotation)
  → CloudWatch (monitoring, alarms)
  → Performance Insights (query analysis)
```

#### Cross-Region DR
```
Primary Region (us-east-1):
  RDS MySQL Multi-AZ (primary)
    → Read Replica 1 (same Region, reporting)
    → Cross-Region Read Replica (eu-west-1)

DR Region (eu-west-1):
  Cross-Region Read Replica
    → On disaster: PROMOTE to standalone primary
    → Applications switch to new primary endpoint
    → RPO: replication lag (seconds–minutes)
    → RTO: promotion time + DNS update + app reconnection
```

#### Lambda + RDS (Most Common Exam Pattern)
```
API Gateway → Lambda Function
  → RDS Proxy (VPC, connection pooling)
    → RDS MySQL Multi-AZ
  → Secrets Manager (DB credentials, auto-rotated)
  → IAM Auth (Lambda role → RDS Proxy → DB)
```

#### Blue-Green Deployment
```
RDS Blue-Green Deployment (native feature):
  Blue: Current production (primary + Multi-AZ + replicas)
  Green: Staging copy (replica promoted, updated schema/version)
  Switchover: AWS swaps DNS endpoints — Blue becomes Green, Green becomes Blue
  Rollback: Switch back to original Blue
```

**RDS Blue-Green Deployments:**
- Creates a staging environment (green) that's a copy of production (blue).
- Green environment stays in sync via replication.
- You can apply changes (engine version upgrade, parameter changes, schema changes) to Green.
- Switchover typically completes in **under a minute** with no data loss.
- Available for MySQL, MariaDB, PostgreSQL.
- **Not available for Oracle, SQL Server, or Multi-AZ Cluster.**

### Well-Architected Framework Alignment

**Reliability**
- Enable Multi-AZ for production workloads — automatic failover.
- Use automated backups with maximum retention (35 days) for PITR.
- Test failover regularly with `RebootDBInstance --ForceFailover`.
- Use cross-Region read replicas for multi-Region DR.
- Use RDS Proxy to minimize failover impact on applications.

**Security**
- Deploy in **private subnets** — no public accessibility.
- Use RDS Proxy with **IAM authentication** — no static passwords in application code.
- Rotate credentials automatically with **Secrets Manager**.
- Enable **encryption at rest** with KMS CMK.
- Enable **SSL/TLS** for in-transit encryption.
- Use **security groups** to restrict network access.

**Performance**
- Use read replicas to offload read-heavy queries.
- Use RDS Proxy for connection pooling (especially with Lambda).
- Use Performance Insights to identify slow queries.
- Use Enhanced Monitoring for OS-level metrics.
- Right-size instances using CloudWatch CPU/memory metrics.
- Use Provisioned IOPS (io1/io2) for consistent database performance.

**Cost Optimization**
- Use Reserved Instances for steady-state workloads (up to 72% savings).
- Right-size instances — don't over-provision.
- Use Multi-AZ Cluster instead of Multi-AZ Instance + read replicas (two birds with one stone: HA + reads).
- Stop development/test instances when not in use (can stop for up to 7 days at a time).
- Delete unused read replicas and snapshots.
- Use gp3 storage (cheaper than io1 for moderate IOPS needs).

**Operational Excellence**
- Use Blue-Green Deployments for safe major version upgrades.
- Use RDS Event Notifications + EventBridge for automated response.
- Use Performance Insights for proactive query optimization.
- Automate snapshot management with AWS Backup.
- Use parameter groups for consistent configuration across environments.

---

## 3. Security & Compliance

### IAM and Access Control

#### IAM Actions

| Action | What It Controls |
|--------|-----------------|
| `rds:CreateDBInstance` | Who can create DB instances |
| `rds:DeleteDBInstance` | Who can delete instances |
| `rds:ModifyDBInstance` | Who can modify (resize, Multi-AZ toggle, etc.) |
| `rds:CreateDBSnapshot` | Who can create snapshots |
| `rds:RestoreDBInstanceFromDBSnapshot` | Who can restore |
| `rds:CreateDBInstanceReadReplica` | Who can create read replicas |
| `rds:RebootDBInstance` | Who can reboot/force failover |

#### IAM Database Authentication

- Supported for **MySQL and PostgreSQL** (and Aurora MySQL/PostgreSQL).
- Application uses an **IAM role/user** to generate an **authentication token** (short-lived, 15-minute expiry).
- Token is used as the password when connecting to the DB.
- **Benefits**: No password management, IAM policy-based access, CloudTrail logging of DB authentication.
- **Limitations**: 200 new connections/second limit per IAM authentication. Not suitable for extremely high connection rates without RDS Proxy.
- **RDS Proxy + IAM auth**: Best combination — Proxy handles IAM token generation and connection pooling.

#### SCPs for RDS

```json
{
  "Effect": "Deny",
  "Action": "rds:CreateDBInstance",
  "Resource": "*",
  "Condition": {
    "Bool": { "rds:StorageEncrypted": "false" }
  }
}
```

- Enforce encryption on all new DB instances.
- Restrict RDS operations to approved Regions.
- Deny public accessibility: condition on `rds:PubliclyAccessible`.
- Restrict instance types (prevent expensive instance launches).

### Encryption

#### At Rest

| Feature | Detail |
|---------|--------|
| **Engine** | AES-256 via AWS KMS |
| **Scope** | Encrypts: DB storage, automated backups, read replicas, snapshots |
| **Enable** | At creation time only. **Cannot encrypt an existing unencrypted DB instance in-place** |
| **Encrypt existing DB** | Snapshot → Copy snapshot with encryption → Restore from encrypted snapshot → Point application to new instance |
| **Read replicas** | If primary is encrypted, replicas MUST be encrypted (same key in same Region, different key allowed cross-Region) |
| **Cross-account snapshot** | Must use CMK (not aws/rds default key). Share KMS key + share snapshot |
| **Cross-Region snapshot copy** | Must specify KMS key in destination Region |
| **Oracle TDE** | Transparent Data Encryption available via Option Group (additional layer, managed by Oracle within the DB) |
| **SQL Server TDE** | Available via Option Group for Enterprise Edition |

#### In Transit

- **SSL/TLS** supported for all engines.
- Can **enforce SSL** via parameter group:
  - MySQL: `require_secure_transport = 1`
  - PostgreSQL: `rds.force_ssl = 1`
  - Oracle: Use Oracle Native Network Encryption (NNE) or SSL
  - SQL Server: `rds.force_ssl = 1`
- RDS provides **SSL certificates** for each instance (downloadable from AWS).

### Logging & Monitoring

| Tool | What It Captures |
|------|-----------------|
| **CloudWatch Metrics** | CPUUtilization, FreeableMemory, ReadIOPS, WriteIOPS, DatabaseConnections, FreeStorageSpace, **ReplicaLag**, DiskQueueDepth, SwapUsage |
| **Enhanced Monitoring** | OS-level metrics at 1-second granularity (CPU per process, memory, disk I/O, threads) |
| **Performance Insights** | DB load, top SQL, wait events, query analysis. **Most important for query-level troubleshooting** |
| **CloudTrail** | RDS API calls (management plane) |
| **Database logs** | Error log, slow query log (MySQL), general log, audit log (varies by engine). Publish to **CloudWatch Logs** |
| **RDS Event Notifications** | DB instance events (failover, backup, maintenance) → SNS → EventBridge |
| **AWS Config** | Rules: `rds-instance-public-access-check`, `rds-storage-encrypted`, `rds-multi-az-support` |
| **GuardDuty RDS Protection** | Detects suspicious login activity (brute force, unusual access patterns) for Aurora (being extended to RDS) |

### Compliance Patterns

- **HIPAA**: Encryption at rest + in transit + CloudTrail + Enhanced Monitoring + private subnets.
- **PCI DSS**: Same as HIPAA + database audit logging + parameter group enforcing SSL.
- **SOC 2**: CloudTrail + Config Rules + Event Notifications for change management.
- **Data residency**: Deploy in specific Region. Use SCPs to prevent RDS in unapproved Regions. Cross-Region replicas only to approved Regions.

---

## 4. Cost Optimization

### Pricing Components

| Component | Pricing Model |
|-----------|--------------|
| **Instance hours** | Per-hour based on instance type. Multi-AZ is ~2× the single-AZ price |
| **Storage** | gp2/gp3: per GB/month. io1/io2: per GB/month + per provisioned IOPS/month. Magnetic: per GB/month |
| **Backup storage** | Free up to 100% of provisioned DB storage. Beyond that: $0.095/GB/month |
| **Snapshot storage** | $0.095/GB/month (manual snapshots, cross-Region copies) |
| **Data transfer** | In: free. Out to internet: standard rates. Cross-Region replication: $0.02/GB |
| **RDS Proxy** | Per vCPU/hour based on the proxy endpoint |
| **Multi-AZ** | ~2× single-AZ instance cost (you pay for the standby instance) |
| **Read Replicas** | Full instance cost (same as primary). Cross-Region replicas also incur data transfer |

### Pricing Models for Instances

| Model | Savings | Commitment |
|-------|---------|-----------|
| **On-Demand** | None (baseline) | None |
| **Reserved Instances (RI)** | Up to **72%** | 1 or 3 years. All Upfront, Partial, No Upfront |
| **On-Demand + Savings Plans** | Compute Savings Plans do **NOT** apply to RDS (only EC2/Fargate/Lambda). Must use **RDS RIs** | N/A |

**Exam critical:** Compute Savings Plans do NOT cover RDS. You MUST purchase **RDS Reserved Instances** specifically for RDS cost savings.

### Storage Pricing (us-east-1)

| Type | $/GB/month | IOPS Cost | Notes |
|------|-----------|-----------|-------|
| **gp2** | $0.115 | Included (3 IOPS/GiB) | Legacy |
| **gp3** | $0.08 | $0.005/IOPS above 3,000 | Recommended |
| **io1** | $0.125 | $0.10/IOPS/month | High IOPS |
| **io2** | $0.125 | Tiered pricing | Highest durability |
| **Magnetic** | $0.10 | $0.10/1M requests | Legacy, avoid |

### Cost-Saving Strategies

1. **RDS Reserved Instances**: Up to 72% savings for steady-state workloads. Purchase for production DBs.

2. **gp3 over gp2**: 20% cheaper storage baseline + independent IOPS provisioning. Migrate existing gp2 volumes to gp3.

3. **Right-size instances**: Monitor CPU/memory with CloudWatch + Enhanced Monitoring. Downsize if under-utilized.

4. **Stop dev/test instances**: RDS allows stopping instances for up to **7 days** at a time (auto-restarts after 7 days). You pay for storage but not compute.

5. **Multi-AZ Cluster instead of Multi-AZ + Read Replica**: If you need HA + moderate read scaling (MySQL/PostgreSQL), Multi-AZ Cluster provides both with 3 instances instead of needing a separate read replica (4 instances).

6. **Delete unused read replicas**: Each replica costs the same as a full instance.

7. **Snapshot cleanup**: Manual snapshots are retained indefinitely. Delete old ones.

8. **Graviton (ARM) instances**: db.m7g, db.r7g — up to 20% cheaper with better performance vs Intel equivalents.

### Common Cost Traps

- **Multi-AZ doubles the instance cost** — standby is a full instance you pay for even though it's not serving traffic (classic Multi-AZ).
- **Read replicas are full instances** — 5 read replicas = 5× instance cost.
- **Cross-Region read replica data transfer**: $0.02/GB for ongoing replication. High-write workloads generate significant transfer costs.
- **Snapshot accumulation**: Manual snapshots beyond free tier cost $0.095/GB/month. Thousands of snapshots = significant charges.
- **Provisioned IOPS (io1/io2) overprovisioning**: Pay for provisioned IOPS whether used or not. Monitor actual IOPS usage.
- **Compute Savings Plans DON'T apply to RDS**: Must use RDS-specific Reserved Instances.
- **Auto-start after 7 days**: Stopped RDS instances automatically restart after 7 days. If you forget, you pay for compute.

---

## 5. High Availability, Disaster Recovery & Resilience

### HA Architecture Patterns

| Pattern | RTO | RPO | Implementation |
|---------|-----|-----|---------------|
| **Multi-AZ Instance** | 60–120 seconds | 0 (synchronous) | Standard production HA |
| **Multi-AZ Cluster** | ~35 seconds | 0 (semisynchronous) | Enhanced HA + read scaling (MySQL/PG) |
| **Multi-AZ + RDS Proxy** | **Seconds** (Proxy detects failover faster) | 0 | Best single-Region HA |
| **Cross-Region Read Replica** | Minutes (promote + DNS + app restart) | Seconds–minutes (replication lag) | Multi-Region DR |
| **Snapshot copy cross-Region** | Hours (restore from snapshot) | Hours (snapshot frequency) | Low-cost DR |
| **Blue-Green Deployment** | Under 1 minute (switchover) | 0 (replication keeps green in sync) | Major upgrades, not DR |

### Failover Behavior

#### Multi-AZ Failover (Classic)

1. Primary becomes unhealthy (or manual failover triggered).
2. AWS promotes the standby to primary.
3. DNS CNAME is updated to point to the new primary.
4. Applications using the endpoint automatically redirect (after DNS cache expires).
5. **Total time: 60–120 seconds** (application should have retry logic with exponential backoff).
6. A new standby is provisioned automatically.

#### Multi-AZ Cluster Failover

1. Primary becomes unhealthy.
2. One of the two reader instances is promoted to writer.
3. Cluster endpoint is updated.
4. **Total time: ~35 seconds** — faster because readers are already running with current data.

#### RDS Proxy Failover

1. Proxy detects primary failure.
2. Proxy redirects connections to the new primary automatically.
3. Application connections are transparently re-routed (may see brief error, but much faster recovery).
4. **Total time: seconds** — significantly faster than DNS-based failover.

### Cross-Region DR Process

```
Normal operation:
  us-east-1: RDS Multi-AZ Primary → async replication → eu-west-1: Cross-Region Read Replica

Disaster (us-east-1 down):
  1. Promote cross-Region read replica in eu-west-1 to standalone primary
  2. Promotion takes a few minutes
  3. Read replica becomes independent read/write DB
  4. Update application configuration to use new endpoint (or Route 53 failover)
  5. Replication from old primary is permanently broken
  6. After recovery: set up replication in reverse direction if needed
```

### Backup & Recovery

| Strategy | RPO | RTO | Notes |
|----------|-----|-----|-------|
| **Automated backup + PITR** | 5 minutes (transaction log frequency) | Minutes–hours (restore creates new instance) | Best RPO for single-Region |
| **Manual snapshots** | At snapshot time | Minutes–hours | User-controlled, indefinite retention |
| **Cross-Region snapshot copy** | Hours (copy frequency) | Hours (restore in DR Region) | Cheapest cross-Region DR |
| **Cross-Region read replica** | Seconds–minutes (replication lag) | Minutes (promote replica) | Best RPO/RTO for cross-Region |
| **AWS Backup** | Configurable (schedule-based) | Hours | Centralized, cross-account |

---

## 6. Exam-Focused Section

### Straightforward Questions (7 examples)

**Q1:** What is the maximum number of read replicas supported per RDS MySQL instance?
**A:** **5.** This applies to MySQL, MariaDB, PostgreSQL, and Oracle.

**Q2:** Can the standby in a classic Multi-AZ deployment be used for read queries?
**A:** **No.** The classic Multi-AZ standby is NOT accessible for any queries. It is exclusively for failover.

**Q3:** Which RDS feature helps Lambda functions avoid exhausting database connections?
**A:** **RDS Proxy.** It pools database connections so Lambda invocations share a managed pool instead of each opening a new connection.

**Q4:** What happens when you promote a read replica?
**A:** The read replica becomes a **standalone read/write DB instance**. Replication from the source is **permanently broken**. The promoted instance is independent.

**Q5:** Can you encrypt an existing unencrypted RDS instance?
**A:** **Not directly.** You must: (1) Create a snapshot, (2) Copy the snapshot with encryption enabled, (3) Restore from the encrypted snapshot to a new instance.

**Q6:** Which engines support RDS Multi-AZ Cluster?
**A:** **MySQL and PostgreSQL only.** Not Oracle, SQL Server, or MariaDB.

**Q7:** What is the maximum automated backup retention period for RDS?
**A:** **35 days.**

---

### Tricky / Scenario-Based Questions (7 examples)

**Q1:** A company runs a production RDS MySQL instance as Multi-AZ. During peak hours, read queries slow down the primary. An architect proposes using the Multi-AZ standby for read queries. Is this feasible?

**A:** **No.** Classic Multi-AZ standby is **NOT readable**.
- **Fix option A:** Add **read replicas** (up to 5) and direct read traffic to them.
- **Fix option B:** Switch to **Multi-AZ Cluster** (MySQL/PostgreSQL only) — the two readers serve read traffic AND provide HA.
- **Key phrases:** "Multi-AZ standby for reads" → Not possible (classic). If the answer mentions Multi-AZ Cluster, that works.

**Q2:** A company has an RDS PostgreSQL Multi-AZ instance. They need cross-Region disaster recovery with an RPO of less than 1 minute. What should they implement?

**A:** Create a **cross-Region read replica** in the DR Region.
- **Why not cross-Region snapshot copy?** Snapshots are point-in-time — RPO would be hours (snapshot frequency). The requirement is <1 minute.
- **Why not Multi-AZ in the DR Region?** Multi-AZ is single-Region only — cannot span Regions.
- **RPO of cross-Region read replica:** Typically seconds of replication lag (asynchronous, but near-real-time).
- **Key phrases:** "cross-Region DR" + "RPO < 1 minute" → Cross-Region read replica.

**Q3:** A company uses AWS Lambda functions that connect to an RDS MySQL database. During traffic spikes, the database reaches its maximum connection limit and new Lambda invocations fail with "Too many connections." What is the best solution?

**A:** Deploy **RDS Proxy** between Lambda and RDS.
- **Why Proxy?** It pools and multiplexes connections. 1,000 Lambda invocations share a pool of (e.g.) 100 actual DB connections instead of each opening its own.
- **Why not increase `max_connections`?** Increasing the parameter is a band-aid — each connection consumes memory. At scale, the DB runs out of memory.
- **Why not increase instance size?** Helps temporarily but doesn't solve the architectural problem. Lambda can scale to thousands of concurrent invocations.
- **Key phrases:** "Lambda" + "Too many connections" + "connection limit" → RDS Proxy.

**Q4:** A company wants to perform a major version upgrade of RDS PostgreSQL from 13 to 16 with minimal downtime. The database is Multi-AZ with 3 read replicas. What approach minimizes risk?

**A:** Use **RDS Blue-Green Deployment**.
- **How it works:** RDS creates a green environment (copy of blue). Green is kept in sync. Apply the version upgrade to green. Test green. Switchover DNS endpoints — blue becomes green, green becomes blue. Switchover takes **under 1 minute**.
- **Why not modify the instance directly?** Major version upgrades can cause extended downtime (minutes to hours) and are risky — no easy rollback.
- **Why not create a read replica and promote it?** That works but requires manual orchestration, DNS changes, and re-creating read replicas. Blue-Green automates this.
- **Key phrases:** "major version upgrade" + "minimal downtime" + "minimize risk" → Blue-Green Deployment.

**Q5:** A company has an RDS Oracle Enterprise Edition database that requires a custom Oracle patch not available through the standard RDS patching process. The DBA needs SSH access to the underlying OS. What should they use?

**A:** **RDS Custom for Oracle.**
- **Why not standard RDS?** No OS access on standard RDS. Cannot install custom patches.
- **Why not EC2 self-managed?** Works, but the question implies they want managed infrastructure (backups, Multi-AZ). RDS Custom provides both OS access AND managed features.
- **Automation pause:** The DBA can pause RDS automation, apply the custom patch via SSH, then resume automation.
- **Key phrases:** "Oracle" + "custom patch" + "SSH/OS access" + "managed" → RDS Custom.

**Q6:** A company's RDS MySQL Multi-AZ instance fails over. The application experiences 2 minutes of downtime because it caches the old DNS endpoint. How can they reduce failover impact?

**A:** Implement **RDS Proxy**.
- **Why Proxy?** Proxy detects failover at the TCP level (not DNS). It redirects connections to the new primary within **seconds**, bypassing DNS caching issues.
- **Why not reduce DNS TTL?** RDS DNS TTL is already set to 5 seconds. The issue is application-level DNS caching or connection pooling holding stale connections.
- **Why not Multi-AZ Cluster?** Multi-AZ Cluster has faster failover (~35s) but doesn't solve application-side DNS caching. Proxy is the definitive solution.
- **Key phrases:** "failover downtime" + "DNS caching" + "reduce failover impact" → RDS Proxy.

**Q7:** A company uses RDS SQL Server Enterprise Edition and needs read replicas for reporting queries in the same Region. They try to create a read replica but get an error. What is wrong?

**A:** **RDS does not support read replicas for SQL Server.** SQL Server read scaling requires **Always On Availability Groups**, which are not managed by RDS (only available on RDS Custom for SQL Server or self-managed on EC2).
- **Workaround on RDS:** Use **snapshot-based reporting**: restore a snapshot to a separate RDS instance for read queries. Not real-time, but functional.
- **Better approach:** If the company needs real-time read replicas for SQL Server, use **RDS Custom for SQL Server** (where they can configure Always On AG manually) or migrate to EC2.
- **Key phrases:** "SQL Server" + "read replica" → Not supported on standard RDS.

---

### Common Exam Traps & Pitfalls

1. **Multi-AZ standby is NOT readable (classic)**: This is the #1 RDS exam trap. Only Multi-AZ Cluster readers are readable.

2. **Read Replicas are NOT for HA**: Promotion is manual and breaks replication. Multi-AZ provides automatic HA failover.

3. **SQL Server does NOT support RDS read replicas**: Must use Always On AG (self-managed on EC2 or RDS Custom).

4. **RDS Proxy does NOT support Oracle or SQL Server**: Only MySQL, PostgreSQL, MariaDB (and Aurora).

5. **RDS Custom is only for Oracle EE and SQL Server EE**: Not for MySQL, PostgreSQL, or MariaDB.

6. **Multi-AZ Cluster is only for MySQL and PostgreSQL**: Not Oracle, SQL Server, or MariaDB.

7. **Cannot encrypt an existing unencrypted RDS instance**: Must go through snapshot → copy (encrypt) → restore workflow. Same as EBS.

8. **Compute Savings Plans do NOT apply to RDS**: Must use RDS Reserved Instances.

9. **Automated backups are deleted when the DB instance is deleted** (unless you take a final snapshot). Manual snapshots persist.

10. **Cross-Region read replica replication is asynchronous**: There IS replication lag. RPO is not zero — it's seconds to minutes.

11. **PITR creates a NEW instance**: Not an in-place restore. You must redirect the application to the new endpoint.

12. **RDS stopped instances auto-start after 7 days**: If you forget, you're billed for compute.

13. **Blue-Green Deployment not available for Oracle, SQL Server, or Multi-AZ Cluster**: Only MySQL, MariaDB, PostgreSQL in standard Multi-AZ or single-AZ.

14. **Read replica can itself be Multi-AZ**: A read replica can have its own standby for HA. This is valid and tested.

15. **Cross-Region snapshot sharing requires CMK**: The default `aws/rds` key cannot be shared cross-account. Must re-encrypt with CMK.

---

## 7. Cheat Sheet

### Must-Know Facts for Exam Day

| Fact | Value |
|------|-------|
| Max read replicas | **5** (MySQL, PG, MariaDB, Oracle) |
| SQL Server read replicas | **Not supported** |
| Multi-AZ classic standby readable | **No** |
| Multi-AZ Cluster standby readable | **Yes** (2 readers) |
| Multi-AZ Cluster engines | **MySQL, PostgreSQL only** |
| Multi-AZ classic failover time | **60–120 seconds** |
| Multi-AZ Cluster failover time | **~35 seconds** |
| RDS Proxy failover time | **Seconds** |
| RDS Proxy engines | **MySQL, PostgreSQL, MariaDB** |
| RDS Custom engines | **Oracle EE, SQL Server EE** |
| Max backup retention | **35 days** |
| Transaction log backup frequency | **5 minutes** |
| Max storage (MySQL/PG/MariaDB/Oracle) | **64 TiB** |
| Max storage (SQL Server) | **16 TiB** |
| PITR restore | **Creates NEW instance** (not in-place) |
| Encrypt existing unencrypted DB | **Not possible in-place** (snapshot → copy encrypted → restore) |
| Cross-Region replication | **Asynchronous** (replication lag) |
| Compute Savings Plans for RDS | **Not applicable** — use RDS RIs |
| Stopped instance auto-restart | **7 days** |
| Blue-Green Deployment engines | **MySQL, MariaDB, PostgreSQL** |

### Key Differentiators

| Multi-AZ (Classic) | Multi-AZ Cluster | Read Replica |
|-------------------|-----------------|--------------|
| 1 standby (not readable) | 2 readers (readable) | Up to 5 (readable) |
| Synchronous | Semisynchronous | Asynchronous |
| Automatic failover | Automatic failover | Manual promotion |
| Same Region only | Same Region only | Cross-Region supported |
| All engines | MySQL, PG only | All except SQL Server |
| HA purpose | HA + read scaling | Read scaling + DR |

| Standard RDS | RDS Custom | Self-Managed EC2 |
|-------------|-----------|-----------------|
| No OS access | OS access (SSH/RDP) | Full access |
| AWS manages everything | Shared management | You manage everything |
| All 5 engines | Oracle EE, SQL Server EE | Any engine |
| Parameter groups | Custom patches, software | Complete control |
| Limited customization | Moderate customization | Full customization |

| RDS Proxy | Application Connection Pool | pgBouncer/ProxySQL |
|-----------|---------------------------|-------------------|
| Fully managed | Application code changes | Self-managed on EC2 |
| IAM auth + Secrets Manager | Application manages credentials | Manual credential config |
| Faster failover | No failover benefit | No failover benefit |
| MySQL, PG, MariaDB | Any engine | Specific to PG/MySQL |
| Best for Lambda | N/A for Lambda | Possible but complex |

### Decision Flowchart

```
Question says "HA" + "automatic failover" (single Region)?
  → Multi-AZ (Classic for all engines, Cluster for MySQL/PG)

Question says "read scaling" or "offload reads"?
  → Read Replicas (up to 5)

Question says "HA + read scaling" + "MySQL or PostgreSQL"?
  → Multi-AZ Cluster

Question says "cross-Region DR" + "low RPO"?
  → Cross-Region Read Replica (promote on failover)

Question says "Lambda" + "database connections" or "Too many connections"?
  → RDS Proxy

Question says "reduce failover time" or "DNS caching during failover"?
  → RDS Proxy

Question says "IAM database authentication" + "credential rotation"?
  → RDS Proxy + Secrets Manager + IAM auth

Question says "Oracle" or "SQL Server" + "OS access" or "custom patch"?
  → RDS Custom

Question says "SQL Server read replica"?
  → Not supported on RDS. Use RDS Custom (Always On AG) or EC2

Question says "major version upgrade" + "minimal downtime"?
  → Blue-Green Deployment (MySQL, MariaDB, PG)

Question says "encrypt existing unencrypted DB"?
  → Snapshot → Copy (encrypted) → Restore new instance

Question says "share snapshot cross-account"?
  → Re-encrypt with CMK + share key + share snapshot

Question says "multi-master writes"?
  → NOT RDS. Use Aurora Multi-Master (limited) or DynamoDB Global Tables

Question says "full database control" + no managed service needed?
  → Self-managed on EC2

Question says "cost savings for steady-state RDS"?
  → RDS Reserved Instances (NOT Compute Savings Plans)

Question says "point-in-time recovery"?
  → Automated backups + PITR (restores to NEW instance, RPO = 5 minutes)

Question says "standby for reads" (classic Multi-AZ)?
  → NOT possible. Multi-AZ Classic standby is not readable.
```

---

*Generated for AWS SAP-C02 exam preparation. Last updated: 2026-04-26.*
