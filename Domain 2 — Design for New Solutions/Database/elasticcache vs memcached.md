# AWS SAP-C02 Study Notes: ElastiCache (Redis vs Memcached, Cluster Mode)

---

## 1. Core Concepts & Theory

### Amazon ElastiCache Overview

Amazon ElastiCache is a **fully managed in-memory data store and cache** service. It supports two open-source engines: **Redis** and **Memcached**. ElastiCache delivers **microsecond latency** for real-time applications by keeping data in memory.

#### Why In-Memory Caching Matters

- Relational databases (RDS, Aurora) deliver single-digit **millisecond** latency.
- DynamoDB delivers single-digit **millisecond** latency (microseconds with DAX).
- ElastiCache delivers **sub-millisecond (microsecond)** latency — 10–100× faster than disk-based databases.
- Use cases: reduce database load, accelerate application performance, store session data, real-time analytics.

---

### Redis vs Memcached — The Complete Comparison

This is one of the **highest-yield** SAP-C02 topics. Know this table cold.

| Feature | **ElastiCache for Redis** | **ElastiCache for Memcached** |
|---------|--------------------------|-------------------------------|
| **Data structures** | Strings, hashes, lists, sets, sorted sets, bitmaps, HyperLogLogs, geospatial, streams | **Strings only** (simple key-value) |
| **Persistence** | **Yes** — RDB snapshots + AOF (Append Only File) | **No** — pure cache, data lost on restart |
| **Replication** | **Yes** — primary + up to 5 read replicas per shard | **No** replication |
| **Multi-AZ** | **Yes** — automatic failover with Multi-AZ | **No** automatic failover |
| **Cluster mode** | **Yes** — data sharding (partitioning) across multiple shards | **Yes** — auto-discovery, multi-node, but no replication |
| **Backup/Restore** | **Yes** — automatic and manual snapshots | **No** |
| **Encryption at rest** | **Yes** (KMS) | **No** |
| **Encryption in transit** | **Yes** (TLS) | **Yes** (TLS) |
| **AUTH** | **Yes** — Redis AUTH token (password) + **IAM authentication** (Redis 7+) | **No** AUTH. SASL authentication available |
| **Pub/Sub** | **Yes** — built-in publish/subscribe messaging | **No** |
| **Lua scripting** | **Yes** | **No** |
| **Geospatial** | **Yes** — native geospatial indexing and queries | **No** |
| **Sorted sets (leaderboards)** | **Yes** — native sorted sets with rank operations | **No** |
| **Streams** | **Yes** — Redis Streams (log data structure, like Kafka-lite) | **No** |
| **Transactions** | **Yes** — MULTI/EXEC atomic operations | **No** |
| **Multi-threaded** | Single-threaded (per shard). But I/O threads in Redis 6+ | **Multi-threaded** — better utilization of multi-core nodes |
| **Max data per node** | Up to **~500 GB** (r7g.16xlarge) | Up to **~500 GB** per node |
| **Global Datastore** | **Yes** — cross-Region replication | **No** |
| **Auto Scaling** | **Yes** — scale replicas and shards | **No** native auto scaling (must add/remove nodes manually) |

**Exam rule of thumb:**
- **Redis**: Choose when you need persistence, replication, Multi-AZ, complex data structures, pub/sub, sorted sets, geospatial, or cross-Region.
- **Memcached**: Choose when you need **simplest possible caching**, multi-threaded performance on a single node, or the application handles all fault tolerance (cache-aside pattern with no persistence needed).

---

### ElastiCache for Redis — Deep Dive

#### Architecture: Non-Cluster Mode (Cluster Mode Disabled)

```
Redis Replication Group:
  Primary Node (read/write)
    ↕ asynchronous replication
  Replica 1 (read-only, AZ-b)
  Replica 2 (read-only, AZ-c)
  ... up to 5 replicas

Single shard — all data fits on one node
Primary endpoint: writes
Reader endpoint: load-balanced reads across replicas
```

- **One shard** (one primary + up to 5 replicas).
- All data must fit in a **single node's memory**.
- Read scaling: add replicas (up to 5).
- Write scaling: **vertical only** (bigger node type).
- **Multi-AZ with automatic failover**: If primary fails, a replica is promoted.

#### Architecture: Cluster Mode Enabled

```
Redis Cluster:
  Shard 1: Primary (slots 0-5460)     + Replica 1a, Replica 1b
  Shard 2: Primary (slots 5461-10922) + Replica 2a, Replica 2b
  Shard 3: Primary (slots 10923-16383)+ Replica 3a, Replica 3b

Data partitioned across shards using hash slots (16,384 total)
Each shard: 1 primary + 0–5 replicas
```

- **Multiple shards** — data partitioned across shards using **16,384 hash slots**.
- Each shard has 1 primary + up to 5 replicas.
- **Horizontal write scaling**: Add more shards to distribute write load.
- **Horizontal read scaling**: Add replicas per shard.
- **Max**: **500 nodes per cluster** (e.g., 250 shards × 2 nodes each, or 83 shards × 6 nodes each).
- **Online resharding**: Add/remove shards without downtime (data redistributed).
- **Slot rebalancing**: Redistribute hash slots across shards for even distribution.

#### Cluster Mode Disabled vs Enabled

| Feature | Cluster Mode Disabled | Cluster Mode Enabled |
|---------|---------------------|---------------------|
| Shards | **1** | **1–500** |
| Data size | Limited to **one node** | **Distributed** across shards (TB-scale) |
| Write scaling | Vertical only (bigger node) | **Horizontal** (add shards) |
| Read scaling | Add replicas (up to 5) | Add replicas per shard |
| Hash slots | N/A | 16,384 slots distributed |
| Multi-key operations | Supported (single shard) | Must use **hash tags** for multi-key operations on same shard |
| Online resharding | N/A | **Yes** |
| Endpoint | Single primary + reader endpoint | **Configuration endpoint** (client discovers shards) |
| Typical use case | Smaller data sets, simpler apps | Large data sets, high write throughput |

#### Redis Global Datastore

Cross-Region replication for Redis (Cluster Mode Enabled).

| Property | Detail |
|----------|--------|
| **Regions** | 1 primary Region + up to **2 secondary Regions** |
| **Replication** | Asynchronous, typically **<1 second** cross-Region |
| **Secondary Region** | **Read-only**. Promotes to primary on failover |
| **Failover** | Manual promotion of secondary Region |
| **Use case** | Cross-Region DR and low-latency global reads |
| **Requires** | Cluster Mode Enabled, encryption in transit |

#### Redis Auto Scaling

- **Replica Auto Scaling**: Add/remove read replicas based on CloudWatch metrics (CPU, connections, etc.).
- **Shard Auto Scaling** (Cluster Mode Enabled): Add/remove shards based on metrics. Redistributes data automatically.
- Uses **Application Auto Scaling** with target tracking or scheduled scaling.

---

### ElastiCache for Memcached — Deep Dive

#### Architecture

```
Memcached Cluster:
  Node 1 (partition of data)
  Node 2 (partition of data)
  Node 3 (partition of data)
  ... up to 300 nodes per cluster

No replication — each node holds a unique partition
Auto-discovery: client discovers all nodes automatically
```

- **Multi-node cluster** — data distributed across nodes using consistent hashing.
- **No replication**: If a node fails, its data is **lost**. The application must handle cache misses.
- **No persistence**: All data is in memory only. Node restart = data loss.
- **No failover**: If a node fails, ElastiCache replaces it with an empty node. No data recovery.
- **Multi-threaded**: Memcached utilizes multiple cores per node — better per-node throughput for simple operations.
- **Auto-discovery**: ElastiCache provides a configuration endpoint. Clients use auto-discovery to find all nodes.

#### When Memcached Is the Right Answer

The exam rarely picks Memcached. But when it does, look for:
- "Simplest caching" + "no persistence needed" + "multi-threaded"
- "Application already handles cache misses" + "object caching"
- "HTML fragment caching" + "no complex data structures"
- "Multi-core utilization" on a single node

---

### Caching Strategies

| Strategy | How It Works | Use Case |
|----------|-------------|----------|
| **Lazy Loading (Cache-Aside)** | Application checks cache first. Cache miss → read from DB → write to cache → return. Cache only populated on read | General purpose. Only caches data that's actually read. Risk: stale data (set TTL) |
| **Write-Through** | Every DB write also writes to cache. Cache always has latest data | Low-latency reads with consistent data. Risk: cache churn (writes to data never read) |
| **Write-Behind (Write-Back)** | Write to cache first. Asynchronously flush to DB | High write throughput. Risk: data loss if cache fails before flush |
| **TTL (Time to Live)** | Each cached item has an expiration time. After TTL, item is evicted/refreshed | Prevents stale data. Balance: short TTL = more cache misses, long TTL = staler data |
| **Session Store** | Store user session data in ElastiCache (Redis). Shared across app instances | Stateless application tier. All instances read/write sessions from Redis |

---

### Additional Features (Exam Relevant)

#### ElastiCache Serverless

- **Launched 2023** — fully serverless ElastiCache (Redis and Memcached).
- No node selection, no cluster management.
- Auto-scales compute and memory.
- Pay per data stored (GB-hours) and ECPUs (ElastiCache Processing Units) consumed.
- Multi-AZ by default.
- **Not the same as traditional provisioned clusters** — different pricing model.

#### Data Tiering (Redis)

- **Extends memory** by using SSDs in addition to RAM.
- Frequently accessed data stays in memory. Less-accessed data moves to SSD.
- Up to **500 GB total** (RAM + SSD) per node on r6gd node types.
- Lower cost than using larger RAM-only nodes.
- Slightly higher latency for SSD-resident data (still microseconds, but slower than pure RAM).

#### Redis Sorted Sets (Leaderboards)

- **Native sorted set** data structure: each member has a score.
- Operations: ZADD, ZRANK, ZRANGE, ZREVRANGE — all O(log N).
- Perfect for: gaming leaderboards, real-time rankings, rate limiting, priority queues.
- This is a unique Redis capability — no equivalent in Memcached or DynamoDB (without complex design).

#### Redis Pub/Sub

- Built-in **publish/subscribe** messaging.
- Publishers send messages to channels. Subscribers receive messages from channels.
- **Not persistent** — if subscriber is offline, messages are lost.
- Use case: real-time notifications, chat, live updates.
- For persistent messaging, use **Redis Streams** or SQS/SNS.

#### Redis Streams

- **Append-only log** data structure (similar to Kafka).
- Persistent (unlike Pub/Sub). Supports consumer groups.
- Use case: event sourcing, activity feeds, IoT data ingestion.

---

### Default Limits Worth Memorizing

| Resource | Limit |
|----------|-------|
| Redis replicas per shard | **5** |
| Redis shards per cluster (Cluster Mode Enabled) | **500** (across all nodes) |
| Max nodes per cluster (Redis CME) | **500** |
| Memcached nodes per cluster | **300** |
| Redis Global Datastore secondary Regions | **2** |
| Redis snapshot retention | **35 days** (automated) |
| Memcached snapshots | **Not supported** |
| Redis AUTH token length | Up to **128 characters** |
| Redis parameter groups per Region | **150** |
| ElastiCache clusters per Region | **50** (adjustable) |
| Subnet groups per Region | **150** |
| Redis hash slots | **16,384** |
| Node types | r-type (memory), m-type (general), t-type (burstable) |
| Data tiering max | **~500 GB per node** (r6gd) |

---

## 2. Design Patterns & Best Practices

### When to Use Which

| Scenario | Choice | Why |
|----------|--------|-----|
| General-purpose caching with HA | **Redis (Cluster Mode Disabled)** | Simple, Multi-AZ failover, persistence |
| Large data set (>node memory), high write throughput | **Redis (Cluster Mode Enabled)** | Horizontal sharding for data + writes |
| Session store for web application | **Redis** | Persistence, Multi-AZ, TTL, shared across instances |
| Real-time leaderboard / ranking | **Redis** | Native sorted sets (ZADD, ZRANK) |
| Pub/Sub messaging | **Redis** | Built-in Pub/Sub |
| Geospatial queries (nearby stores, etc.) | **Redis** | Native geospatial data type |
| Cross-Region read cache | **Redis Global Datastore** | <1s replication, read from local Region |
| Simplest possible object cache, no persistence | **Memcached** | Multi-threaded, simple, no frills |
| Cache for DynamoDB specifically | **DAX** (not ElastiCache) | API-compatible, drop-in |
| Rate limiting / token bucket | **Redis** | Atomic increment, TTL, Lua scripting |

### Anti-Patterns

| Anti-Pattern | Why It's Wrong | Correct Approach |
|-------------|---------------|------------------|
| Memcached for session store | No replication, no persistence — node failure = sessions lost | Redis (replication + persistence + Multi-AZ) |
| Memcached for cross-Region DR | No Global Datastore, no snapshots | Redis with Global Datastore |
| ElastiCache for DynamoDB-specific caching | ElastiCache requires code changes; DAX is drop-in | DAX for DynamoDB caching |
| Redis (Cluster Mode Disabled) for >100 GB data | Single node has memory limits | Redis (Cluster Mode Enabled) for sharding |
| Using ElastiCache as primary database | In-memory = data at risk. Not designed for durability | Use as cache/accelerator in front of a primary database |
| Pub/Sub for reliable messaging | Redis Pub/Sub drops messages if subscriber is offline | Use SQS/SNS for reliable messaging, or Redis Streams |

### Common Architectural Patterns

#### Database Cache (Cache-Aside)
```
Application → Check ElastiCache Redis
  Hit → Return data (microseconds)
  Miss → Read from RDS/Aurora → Store in Redis (TTL) → Return data
```

#### Session Store (Stateless App Tier)
```
ALB → EC2 Instance 1 → ElastiCache Redis (session data)
    → EC2 Instance 2 → ElastiCache Redis (session data)
    → EC2 Instance 3 → ElastiCache Redis (session data)

All instances share session state via Redis
If an instance fails, sessions are preserved in Redis
User is not affected by instance replacement
```

#### Real-Time Leaderboard
```
Game Server → Redis ZADD (player_id, score)
Leaderboard Service → Redis ZREVRANGE (top 100 players)
                    → Redis ZRANK (player_id) → player's rank

Sorted sets: O(log N) for all operations
Millions of players, real-time rankings
```

#### Write-Through with Read Replicas
```
Application writes:
  → Write to RDS (primary)
  → Write to Redis (cache updated immediately)

Application reads:
  → Read from Redis (always fresh)
  → Redis read replicas for scaling

Benefits: Cache always consistent, low read latency
Cost: Every write goes to both DB and cache
```

#### Cross-Region with Global Datastore
```
us-east-1: Redis Cluster (primary, read/write)
  ↕ async replication (<1s)
eu-west-1: Redis Cluster (secondary, read-only)

Application in eu-west-1 reads from local Redis (microsecond latency)
Writes go to us-east-1 primary
Failover: promote eu-west-1 to primary (manual)
```

#### Lambda + ElastiCache
```
API Gateway → Lambda → ElastiCache Redis (VPC)

Lambda MUST be in a VPC to access ElastiCache
Use Redis as a shared cache/state store across Lambda invocations
Connection: Lambda establishes connection per invocation (consider connection pooling)
```

### Well-Architected Framework Alignment

**Reliability**
- Redis Multi-AZ with automatic failover — primary fails, replica promoted.
- Redis persistence (RDB + AOF) — data survives node restart.
- Global Datastore for cross-Region resilience.
- Cluster Mode Enabled for data distribution — no single-shard bottleneck.
- Application should handle cache misses gracefully (cache-aside pattern).

**Security**
- Deploy in **private subnets** — ElastiCache is VPC-only (no public endpoints).
- Redis AUTH token for authentication. IAM authentication for Redis 7+.
- Encryption at rest (KMS) and in transit (TLS) for Redis.
- Security groups to restrict network access.
- No direct internet access — access via VPN, Direct Connect, or bastion.

**Performance**
- Right-size nodes: monitor memory usage, CPU, cache hit rate.
- Use Cluster Mode Enabled for horizontal scaling (shards for writes, replicas for reads).
- Use reader endpoints to distribute read load.
- Monitor `CacheHits` vs `CacheMisses` — tune TTL for optimal hit rate.
- Data tiering (r6gd) for cost-effective large data sets.

**Cost Optimization**
- Reserved Nodes for steady-state (up to **72% savings**).
- Data tiering (r6gd) to extend capacity without expensive large-memory nodes.
- Right-size: monitor `FreeableMemory` and `EngineCPUUtilization`.
- Use Cluster Mode Enabled to add smaller shards vs one large node.
- ElastiCache Serverless for unpredictable/variable workloads.
- Delete development/test clusters when not in use.

**Operational Excellence**
- Automated backups (Redis) with configurable retention.
- CloudWatch metrics: EngineCPUUtilization, CacheHits, CacheMisses, CurrConnections, Evictions.
- Slow log (Redis) for query optimization.
- SNS notifications for cluster events (failover, maintenance, etc.).
- Parameter groups for consistent configuration.

---

## 3. Security & Compliance

### Access Control

#### Network Security
- ElastiCache clusters run **inside a VPC** — no public endpoint.
- Access controlled via **security groups**.
- No IAM resource-based policies (unlike S3 or Lambda).
- Must be in the same VPC as the application (or use VPC peering/Transit Gateway).

#### Authentication

| Method | Redis | Memcached |
|--------|-------|-----------|
| **Redis AUTH** | Password-based (set AUTH token). Single token per cluster | N/A |
| **Redis ACL** (Redis 6+) | User-based access control lists. Multiple users with different permissions | N/A |
| **IAM Authentication** (Redis 7+) | IAM users/roles authenticate directly. Short-lived tokens | N/A |
| **SASL** | N/A | SASL authentication (username/password) |
| **No auth** | Supported (not recommended for production) | Default |

#### IAM Authentication (Redis 7+)

- Applications use **IAM credentials** to generate a short-lived auth token.
- No need to manage Redis passwords.
- Integrates with IAM roles — EC2, Lambda, ECS task roles can authenticate directly.
- **Best practice** for production: IAM auth + encryption in transit.

### Encryption

| Layer | Redis | Memcached |
|-------|-------|-----------|
| **At rest** | **Yes** — KMS (AWS-managed or CMK) | **No** |
| **In transit** | **Yes** — TLS | **Yes** — TLS |
| **Enable timing** | Must be enabled at **cluster creation** — cannot add later | Must be enabled at **cluster creation** |
| **Performance impact** | Minor (~25% overhead for TLS) | Minor |

**Exam critical:** Encryption at rest is **Redis only**. Memcached does NOT support encryption at rest. If the question requires encryption at rest, Memcached is eliminated.

### Logging & Monitoring

| Tool | What It Captures |
|------|-----------------|
| **CloudWatch Metrics** | CPUUtilization, EngineCPUUtilization, FreeableMemory, CacheHits, CacheMisses, CurrConnections, NewConnections, Evictions, ReplicationLag, BytesUsedForCache |
| **CloudWatch Alarms** | Evictions > 0 (cache too small), CacheMisses > threshold, ReplicationLag > threshold |
| **Redis Slow Log** | Queries exceeding configurable latency threshold. Diagnose slow operations |
| **ElastiCache Events** | Cluster events: failover, maintenance, node replacement → SNS notifications |
| **CloudTrail** | ElastiCache management API calls (CreateCacheCluster, ModifyReplicationGroup, etc.) |
| **AWS Config** | Rules: `elasticache-redis-cluster-automatic-backup-check`, `elasticache-repl-grp-auto-failover-enabled` |

#### Critical Metrics

| Metric | Warning | Meaning |
|--------|---------|---------|
| `Evictions` | > 0 | Cache is full and evicting items. Increase node size or add shards |
| `CacheHitRate` | < 80% | Poor cache effectiveness. Review TTL, caching strategy, key design |
| `EngineCPUUtilization` | > 90% | Redis process saturated. Scale up or add shards |
| `FreeableMemory` | < 10% of total | Running out of memory. Scale up or enable data tiering |
| `ReplicationLag` | > 1 second | Replica falling behind. Check network, node size |
| `CurrConnections` | Near `maxclients` | Connection exhaustion. Increase limit or use connection pooling |

### Compliance Patterns

- **HIPAA**: Redis with encryption at rest (KMS CMK) + in transit (TLS) + AUTH/IAM + VPC + CloudTrail.
- **PCI DSS**: Same as HIPAA + Redis ACLs for granular access + security group restrictions.
- **Data residency**: ElastiCache is Regional. Use Global Datastore only to approved Regions.
- **Memcached is unsuitable for compliance workloads** requiring encryption at rest or persistence.

---

## 4. Cost Optimization

### Pricing Model

#### Provisioned (Traditional)

| Component | Cost Factor |
|-----------|------------|
| **Node-hours** | Per hour per node, based on node type. Multi-AZ: pay for primary + replicas |
| **Backup storage** (Redis) | Free up to cluster size. Beyond: $0.085/GB/month |
| **Data transfer** | Cross-AZ: $0.01/GB each way. Cross-Region (Global Datastore): standard data transfer rates |

#### Node Type Pricing Examples (us-east-1, Redis)

| Node Type | vCPUs | Memory | On-Demand $/hr | Use Case |
|-----------|-------|--------|----------------|----------|
| cache.t3.micro | 2 | 0.5 GB | ~$0.017 | Dev/test |
| cache.r7g.large | 2 | 13.07 GB | ~$0.252 | Production |
| cache.r7g.xlarge | 4 | 26.32 GB | ~$0.504 | Large production |
| cache.r7g.16xlarge | 64 | 419 GB | ~$8.064 | Largest workloads |
| cache.r6gd.xlarge | 4 | 26.32 GB (+ SSD) | ~$0.552 | Data tiering |

#### Reserved Nodes

- 1-year or 3-year reservations.
- Up to **72% savings** vs On-Demand.
- Payment: All Upfront, Partial Upfront, No Upfront.
- **NOT covered by Compute Savings Plans** — must purchase ElastiCache Reserved Nodes specifically.

#### ElastiCache Serverless Pricing

| Component | Cost |
|-----------|------|
| Data stored | $0.125/GB-hour |
| ECPUs (reads) | $0.0034 per million ECPUs |
| ECPUs (writes) | $0.0068 per million ECPUs |

### Cost-Saving Strategies

1. **Reserved Nodes for steady-state**: Up to 72% savings. Purchase for production clusters.

2. **Right-size nodes**: Monitor `FreeableMemory` and `EngineCPUUtilization`. Don't over-provision memory.

3. **Data tiering (r6gd)**: Extend data capacity with SSD. Cheaper than upgrading to larger RAM-only nodes for warm data.

4. **Cluster Mode Enabled with smaller shards**: Many small shards can be cheaper than one massive node (and better distributed).

5. **Graviton nodes (r7g, m7g)**: Up to 20% cheaper and better performance than Intel equivalents.

6. **ElastiCache Serverless for variable workloads**: Pay per use. No idle cost beyond minimum data storage.

7. **Delete dev/test clusters**: ElastiCache charges per node-hour. Stop paying when not in use. (Take a Redis snapshot before deleting for data preservation.)

8. **Reduce replicas in non-production**: Dev/test doesn't need 5 replicas. Use 0–1.

### Common Cost Traps

- **Multi-AZ replicas are full instances**: 1 primary + 5 replicas = 6 nodes billed. Each replica costs the same as the primary.
- **Global Datastore cross-Region transfer**: Ongoing data transfer cost for replication.
- **Backup storage beyond free tier**: If backup size exceeds cluster data size, additional charges apply.
- **Forgotten dev/test clusters**: ElastiCache nodes charge 24/7 until deleted.
- **Evictions indicate undersized cache**: Evictions mean you're losing cached data. Increasing node size costs money, but reduces expensive DB round-trips.
- **Reserved Nodes DON'T cover ElastiCache Serverless**: Different pricing models. Reserved Nodes are for provisioned clusters only.

### Cost Comparison

| Need | ElastiCache Redis | DAX | DynamoDB On-Demand |
|------|------------------|-----|-------------------|
| DynamoDB cache | More expensive + code changes | Cheaper, drop-in | No caching (baseline) |
| General cache (RDS, API) | Best option | Not applicable | Not applicable |
| Session store | Good (feature-rich) | Not applicable | Works but slower |
| Microsecond reads | Yes | Yes (DynamoDB only) | No (milliseconds) |

---

## 5. High Availability, Disaster Recovery & Resilience

### Redis HA Architecture

#### Multi-AZ Automatic Failover (Cluster Mode Disabled)

```
AZ-a: Primary (read/write)
  ↕ async replication
AZ-b: Replica 1 (read-only)
AZ-c: Replica 2 (read-only)

Primary fails →
  1. ElastiCache detects failure
  2. Promotes replica with least replication lag
  3. DNS endpoint updated to new primary
  4. Old primary replaced with new replica
  Failover time: ~30 seconds–few minutes
```

#### Multi-AZ (Cluster Mode Enabled)

```
Shard 1: Primary (AZ-a) + Replica (AZ-b)
Shard 2: Primary (AZ-b) + Replica (AZ-c)
Shard 3: Primary (AZ-c) + Replica (AZ-a)

Shard 1 primary fails →
  1. Shard 1 replica (AZ-b) promoted to primary
  2. New replica created
  Other shards unaffected
```

- Shards fail over **independently** — partial cluster failure doesn't affect other shards.
- Distribute primary and replica nodes across AZs for maximum resilience.

#### Global Datastore (Cross-Region DR)

```
us-east-1: Redis Cluster (primary, read/write)
  ↕ async replication (<1s)
eu-west-1: Redis Cluster (secondary, read-only)
ap-southeast-1: Redis Cluster (secondary, read-only)

us-east-1 fails →
  1. Manually promote eu-west-1 to primary
  2. Update application endpoints
  3. eu-west-1 becomes read/write
  RPO: <1 second (replication lag)
  RTO: Minutes (manual promotion)
```

### Memcached HA (Limited)

- **No replication, no failover, no persistence**.
- If a node fails, data is **lost**. ElastiCache replaces the node with an empty one.
- Application MUST handle cache misses gracefully (cache-aside pattern).
- **No cross-Region** capability.
- HA strategy: Application resilience, not infrastructure redundancy.

### HA Comparison

| Feature | Redis (CMD) | Redis (CME) | Memcached |
|---------|------------|------------|-----------|
| Multi-AZ failover | **Yes** (automatic) | **Yes** (per-shard) | **No** |
| Data persistence | **Yes** (RDB/AOF) | **Yes** | **No** |
| Backup/restore | **Yes** | **Yes** | **No** |
| Cross-Region DR | **Yes** (Global Datastore requires CME) | **Yes** (Global Datastore) | **No** |
| Node failure impact | Failover to replica | Failover within shard | **Data lost** |

### Backup & Recovery (Redis Only)

| Type | Mechanism | Retention | Impact |
|------|-----------|-----------|--------|
| **Automated backup** | RDB snapshot at scheduled time | **1–35 days** | Brief increase in latency during snapshot |
| **Manual snapshot** | User-triggered RDB snapshot | **Indefinite** | Same as automated |
| **AOF (Append Only File)** | Continuous write logging | Runtime only | Faster recovery, more durable than RDB alone |
| **Restore** | Create new cluster from snapshot | N/A | Minutes (depends on data size) |
| **Export snapshot** | Copy to S3 for archival or cross-account sharing | Indefinite (in S3) | One-time operation |

### RPO/RTO Summary

| Pattern | RPO | RTO |
|---------|-----|-----|
| **Redis Multi-AZ (same Region)** | ~seconds (async replication lag) | ~30 seconds–minutes (automatic failover) |
| **Redis Global Datastore** | <1 second (cross-Region lag) | Minutes (manual promotion) |
| **Redis snapshot restore** | Hours (snapshot frequency) | Minutes (create new cluster from snapshot) |
| **Memcached** | **Total data loss** (no persistence) | Minutes (node replacement, but cache cold) |

---

## 6. Exam-Focused Section

### Straightforward Questions (7 examples)

**Q1:** A company needs a managed in-memory cache that supports data persistence, replication, and automatic Multi-AZ failover. Which ElastiCache engine should they use?
**A:** **Redis.** It supports persistence (RDB/AOF), replication (up to 5 replicas), and Multi-AZ automatic failover. Memcached does not support any of these.

**Q2:** What is the maximum number of read replicas per shard in an ElastiCache Redis cluster?
**A:** **5.** Each shard can have 1 primary + up to 5 read replicas.

**Q3:** Can Memcached encrypt data at rest?
**A:** **No.** Encryption at rest is supported only for Redis (KMS). Memcached supports encryption in transit (TLS) but NOT at rest.

**Q4:** A company needs a real-time gaming leaderboard that ranks millions of players by score. Which AWS service and data structure is best?
**A:** **ElastiCache for Redis with Sorted Sets.** Redis sorted sets (ZADD, ZRANK, ZREVRANGE) provide O(log N) ranking operations natively.

**Q5:** Which ElastiCache for Redis feature enables cross-Region replication for disaster recovery?
**A:** **Global Datastore.** It provides asynchronous cross-Region replication with <1 second lag. Requires Cluster Mode Enabled.

**Q6:** What happens when a Memcached node fails?
**A:** **Data on that node is lost.** ElastiCache replaces the node with an empty one. There is no replication, no failover, no recovery.

**Q7:** Can ElastiCache be accessed from the public internet?
**A:** **No.** ElastiCache clusters are **VPC-only** with no public endpoint. Access requires being in the same VPC (or connected via peering/TGW/VPN).

---

### Tricky / Scenario-Based Questions (7 examples)

**Q1:** A company runs a web application with session data stored in ElastiCache Memcached. During a node failure, thousands of users are logged out because their sessions are lost. How should the architect fix this with MINIMUM application changes?

**A:** **Migrate to ElastiCache for Redis** with Multi-AZ enabled.
- **Why Redis?** Redis supports persistence (data survives restarts) and Multi-AZ replication (if primary fails, replica has the data). Sessions are preserved during failover.
- **Why not "fix Memcached"?** Memcached has no replication, no persistence, no failover. There is no way to make Memcached resilient to node failure.
- **Why "minimum changes"?** Redis supports the same key-value get/set operations as Memcached. Session libraries (e.g., Spring Session, Express-session) support both. Endpoint change + minor config.
- **Key phrases:** "Memcached" + "sessions lost on node failure" → Migrate to Redis.

**Q2:** A company uses ElastiCache Redis (Cluster Mode Disabled) with a single primary and 2 replicas. The dataset has grown to 150 GB, exceeding the node's 128 GB memory. The application is experiencing evictions. What should they do?

**A:** **Enable Cluster Mode** (migrate to Cluster Mode Enabled) to shard the data across multiple nodes.
- **Why not just upgrade to a bigger node?** This works short-term, but Cluster Mode Disabled limits you to one node's memory. The data will keep growing.
- **How to migrate:** Create a new Cluster Mode Enabled cluster from a backup of the existing cluster. Or use online migration.
- **Alternative:** Use **data tiering** (r6gd node types) to extend memory with SSD — fits more data per node. But if data keeps growing, sharding is the sustainable answer.
- **Key phrases:** "data exceeds node memory" + "evictions" → Cluster Mode Enabled (sharding).

**Q3:** A company uses ElastiCache Redis as a cache for RDS Aurora. They also want to cache DynamoDB reads with microsecond latency. An architect proposes using the same Redis cluster for both. Is this optimal?

**A:** **Not optimal.** Use **DAX for DynamoDB caching** and **Redis for Aurora caching**.
- **Why DAX?** DAX is API-compatible with DynamoDB — drop-in with no code changes (just change the endpoint). It's purpose-built for DynamoDB with optimized caching logic.
- **Why not Redis for DynamoDB?** Redis requires application code to implement caching logic (serialize/deserialize, key management, TTL, invalidation). DAX handles all of this transparently.
- **Redis for Aurora:** Redis is the correct choice for caching relational data — DAX only works with DynamoDB.
- **Key phrases:** "DynamoDB cache" → DAX. "RDS/Aurora cache" → Redis/ElastiCache.

**Q4:** A company needs cross-Region disaster recovery for their ElastiCache Redis cluster. They currently use Cluster Mode Disabled. They try to enable Global Datastore but get an error. Why?

**A:** **Global Datastore requires Cluster Mode Enabled.** It does not work with Cluster Mode Disabled.
- **Fix:** Migrate to Cluster Mode Enabled (create a new CME cluster from a backup), then enable Global Datastore.
- **Additional requirement:** Encryption in transit must be enabled for Global Datastore.
- **Key phrases:** "Global Datastore" + "Cluster Mode Disabled" + "error" → Requires Cluster Mode Enabled.

**Q5:** A Lambda function needs to access an ElastiCache Redis cluster. The function is not in a VPC. The developer gets connection timeouts. What is the issue?

**A:** **ElastiCache is VPC-only.** The Lambda function must be **deployed in the same VPC** as the ElastiCache cluster (or a connected VPC) to access it.
- **Fix:** Configure the Lambda function to run in the VPC where ElastiCache resides. Add the appropriate subnet and security group configurations.
- **Side effect:** VPC-attached Lambda loses default internet access. If it also needs internet, configure a NAT Gateway.
- **Key phrases:** "Lambda" + "ElastiCache" + "connection timeout" → Lambda must be in VPC.

**Q6:** A company stores user preferences in ElastiCache Redis with a 24-hour TTL. The underlying data in RDS changes frequently (every few minutes). Users complain about seeing stale preferences. What should they change?

**A:** Switch from **lazy loading (cache-aside)** to a **write-through** pattern with a shorter TTL.
- **Write-through:** Every time the application updates RDS, also update Redis. Cache is always current.
- **Shorter TTL:** If write-through is too complex, reduce TTL from 24 hours to 5–10 minutes. More cache misses, but fresher data.
- **Why not just invalidate on write?** That's essentially write-through (or cache invalidation). Either approach works — the key is not relying on a 24-hour TTL for frequently changing data.
- **Key phrases:** "stale data" + "long TTL" + "frequently changing source" → Shorter TTL or write-through.

**Q7:** A company evaluates ElastiCache Serverless vs provisioned Redis for a production workload with predictable, steady traffic of 100,000 reads/second. Which is more cost-effective?

**A:** **Provisioned Redis with Reserved Nodes** is more cost-effective for predictable, steady workloads.
- **Why not Serverless?** Serverless pricing is per-use (ECPU-based). At sustained high throughput, provisioned nodes with Reserved pricing (up to 72% off) are significantly cheaper.
- **When IS Serverless better?** Unpredictable traffic, development environments, workloads with dramatic peaks and valleys.
- **Key phrases:** "predictable, steady traffic" + "cost-effective" → Provisioned + Reserved Nodes.

---

### Common Exam Traps & Pitfalls

1. **Memcached has NO persistence, NO replication, NO Multi-AZ failover**: If the question needs any of these, Memcached is wrong. Always Redis.

2. **Memcached has NO encryption at rest**: Only Redis supports encryption at rest (KMS). If compliance requires encryption at rest, Memcached is eliminated.

3. **DAX is for DynamoDB only; ElastiCache is for everything else**: If the question mentions DynamoDB caching, the answer is DAX. If it mentions RDS/Aurora/API caching, the answer is ElastiCache.

4. **Global Datastore requires Cluster Mode Enabled + encryption in transit**: Cannot use Global Datastore with Cluster Mode Disabled.

5. **ElastiCache is VPC-only — no public endpoint**: Lambda must be VPC-attached to access ElastiCache. No public internet access to clusters.

6. **Redis encryption must be enabled at creation**: Cannot add encryption at rest or in transit to an existing cluster. Must create a new cluster.

7. **Cluster Mode Disabled = 1 shard (single node memory limit)**: If data exceeds one node's memory, you need Cluster Mode Enabled.

8. **Multi-key operations in Cluster Mode Enabled require hash tags**: Operations spanning multiple keys (MGET, transactions) must target keys on the same shard. Use hash tags `{tag}` in key names to co-locate.

9. **Redis Pub/Sub is NOT persistent**: If a subscriber is offline, messages are lost. For reliable messaging, use SQS/SNS or Redis Streams.

10. **Failover is automatic for Redis Multi-AZ, manual for Global Datastore**: Within-Region failover is automatic (~30s). Cross-Region promotion (Global Datastore) is manual.

11. **Reserved Nodes ≠ Compute Savings Plans**: ElastiCache has its own Reserved Nodes. Not covered by EC2/Compute Savings Plans.

12. **Sorted Sets = leaderboard**: If the question mentions "leaderboard," "ranking," or "scoring," the answer is Redis Sorted Sets. No equivalent in Memcached or DynamoDB.

13. **Redis is single-threaded per shard; Memcached is multi-threaded**: For CPU-bound simple operations on a single large node, Memcached may have better per-node throughput. But Redis Cluster Mode Enabled distributes across shards (each single-threaded but parallel across shards).

14. **ElastiCache Serverless is different from provisioned**: Different pricing model, different management model. Reserved Nodes don't apply to Serverless.

---

## 7. Cheat Sheet

### Must-Know Facts for Exam Day

| Fact | Value |
|------|-------|
| Redis replicas per shard | **5** |
| Redis max nodes per cluster (CME) | **500** |
| Memcached max nodes | **300** |
| Redis hash slots | **16,384** |
| Redis persistence | **Yes** (RDB + AOF) |
| Memcached persistence | **No** |
| Redis Multi-AZ failover | **Yes** (automatic) |
| Memcached Multi-AZ | **No** |
| Redis encryption at rest | **Yes** (KMS) |
| Memcached encryption at rest | **No** |
| Redis Global Datastore max secondary Regions | **2** |
| Global Datastore requires | **Cluster Mode Enabled + TLS** |
| Redis snapshot retention | **35 days** (automated) |
| Memcached snapshots | **Not supported** |
| Redis AUTH | Yes (token, ACL, IAM in Redis 7+) |
| Memcached AUTH | SASL only |
| ElastiCache public endpoint | **No** (VPC-only) |
| Redis failover time | **~30 seconds–few minutes** |
| Global Datastore replication lag | **<1 second** |
| Reserved Node savings | Up to **72%** |
| Encryption enable timing | **At creation only** |

### Key Differentiators

| Redis | Memcached |
|-------|-----------|
| Rich data structures (sorted sets, lists, hashes, streams, geo) | Strings only |
| Persistence (RDB/AOF) | No persistence |
| Replication (up to 5 replicas) | No replication |
| Multi-AZ automatic failover | No failover |
| Encryption at rest (KMS) | No encryption at rest |
| Pub/Sub, Streams | None |
| Global Datastore (cross-Region) | No cross-Region |
| Backups/snapshots | No backups |
| Single-threaded per shard | Multi-threaded |
| More features, choose for almost all use cases | Simplest caching only |

| DAX | ElastiCache Redis |
|-----|-------------------|
| DynamoDB only | Any data source |
| DynamoDB API-compatible (drop-in) | Redis API (code changes) |
| Microsecond DynamoDB reads | Microsecond any-data reads |
| Eventually consistent only | Application-controlled consistency |
| Managed by AWS inside VPC | Managed by AWS inside VPC |

| Cluster Mode Disabled | Cluster Mode Enabled |
|---------------------|---------------------|
| 1 shard | 1–500 shards |
| Data fits in one node | Data sharded across nodes |
| Vertical write scaling | Horizontal write scaling |
| Up to 5 replicas | Up to 5 replicas per shard |
| No online resharding | Online resharding supported |
| No Global Datastore | Global Datastore supported |
| Simpler | More complex, more scalable |

### Decision Flowchart

```
Question says "in-memory cache" + "persistence" or "replication" or "Multi-AZ"?
  → Redis

Question says "in-memory cache" + "simplest" + "no persistence needed" + "multi-threaded"?
  → Memcached (rare exam answer)

Question says "DynamoDB" + "cache" or "microsecond"?
  → DAX (NOT ElastiCache)

Question says "RDS/Aurora cache" or "API cache" or "session store"?
  → ElastiCache Redis

Question says "leaderboard" or "ranking" or "sorted scores"?
  → Redis Sorted Sets

Question says "pub/sub" or "real-time messaging" (in-memory)?
  → Redis Pub/Sub (or Streams for persistence)

Question says "geospatial queries" + "in-memory"?
  → Redis geospatial data type

Question says "cross-Region cache replication"?
  → Redis Global Datastore (requires Cluster Mode Enabled)

Question says "data exceeds single node memory"?
  → Redis Cluster Mode Enabled (sharding)

Question says "Lambda" + "ElastiCache" + "cannot connect"?
  → Lambda must be in VPC

Question says "session store" + "node failure" + "data lost"?
  → Migrate from Memcached to Redis

Question says "encryption at rest" + "in-memory cache"?
  → Redis (Memcached doesn't support encryption at rest)

Question says "cache" + "unpredictable traffic" + "serverless"?
  → ElastiCache Serverless

Question says "cache" + "steady traffic" + "cost-optimize"?
  → Provisioned Redis + Reserved Nodes

Question says "stale cache data"?
  → Reduce TTL, switch to write-through, or implement cache invalidation

Question says "evictions" in ElastiCache?
  → Cache too small. Increase node size, add shards, or enable data tiering
```

---

*Generated for AWS SAP-C02 exam preparation. Last updated: 2026-04-26.*
