# Performance Optimization: Caching, CDN, Read Replicas, Global Accelerator

## 1. Core Concepts & Theory

### Performance Optimization Overview
Performance optimization on AWS involves reducing latency, increasing throughput, and improving user experience through:
- **Caching** (reduce repeated computation/retrieval)
- **CDN** (move content closer to users)
- **Read Replicas** (offload read traffic from primary database)
- **Global Accelerator** (optimize network path)

---

### Caching

#### Amazon ElastiCache

**ElastiCache for Redis:**
- **In-memory data store** — sub-millisecond latency
- **Data structures**: Strings, hashes, lists, sets, sorted sets, bitmaps, HyperLogLog, streams, geospatial
- **Persistence**: RDB snapshots + AOF (append-only file) — survives restarts
- **Replication**: Primary + up to 5 read replicas (async replication)
- **Cluster Mode Disabled**: Single shard, 1 primary + replicas (max 6 nodes, up to ~340 GB)
- **Cluster Mode Enabled**: Multiple shards (partitions), each with primary + replicas (up to 500 nodes, 340 TB data)
- **Multi-AZ**: Automatic failover to replica (seconds)
- **Global Datastore**: Cross-region replication (< 1 second lag, read replicas in up to 2 secondary regions)
- **Auth**: Redis AUTH token (password) + TLS encryption
- **Pub/Sub**: Built-in messaging
- **Lua scripting**: Server-side script execution
- **Transactions**: MULTI/EXEC (atomic operations)

**ElastiCache for Memcached:**
- **Simple caching** — key-value only (strings)
- **Multi-threaded**: Better for simple caching with multiple cores
- **No persistence**: Data lost on restart
- **No replication**: No failover (must handle cache miss in application)
- **Auto Discovery**: Client auto-detects cluster nodes
- **Partitioning**: Data sharded across nodes
- **Max node size**: 300+ GB (same as Redis per node)

**Redis vs Memcached Decision:**
| Feature | Redis | Memcached |
|---------|-------|-----------|
| Data structures | Complex (lists, sets, sorted sets) | Simple (key-value strings) |
| Persistence | Yes (RDB + AOF) | No |
| Replication/HA | Yes (Multi-AZ failover) | No |
| Cluster mode | Yes (sharding + replication) | Yes (sharding only) |
| Pub/Sub | Yes | No |
| Transactions | Yes | No |
| Multi-threaded | No (single-threaded per shard) | Yes |
| Backup/Restore | Yes | No |
| Use case | Complex caching, sessions, leaderboards, queues | Simple key-value caching, disposable data |

**Caching Strategies:**
| Strategy | How It Works | Best For |
|----------|-------------|----------|
| **Lazy Loading (Cache-Aside)** | Read: check cache → miss → read DB → write cache | Read-heavy, tolerates stale data |
| **Write-Through** | Every DB write also writes to cache | Always fresh cache, write penalty |
| **Write-Behind (Write-Back)** | Write to cache immediately, async write to DB | Write-heavy, risk of data loss |
| **TTL (Time-To-Live)** | Cache entries expire after set duration | Balances freshness and cache hit rate |

**Lazy Loading:**
```
Read: Cache hit? → Return cached data
      Cache miss? → Read from DB → Store in cache → Return data
```
- Pro: Only caches data that's actually requested
- Con: Cache miss penalty (extra latency), stale data until TTL expires

**Write-Through:**
```
Write: Write to DB → Write to cache
Read: Always hits cache (fresh data)
```
- Pro: Cache always fresh
- Con: Write latency (two writes), caches data that may never be read

#### Amazon DynamoDB Accelerator (DAX)
- **In-memory cache** purpose-built for DynamoDB
- **Microsecond latency** for reads (10x faster than DynamoDB alone)
- **Write-through cache**: Writes go through DAX to DynamoDB
- **API-compatible**: Same API as DynamoDB (drop-in replacement in code)
- **Cluster**: Primary + read replicas (up to 10 nodes, Multi-AZ)
- **Item cache**: Caches individual GetItem/BatchGetItem results
- **Query cache**: Caches Query/Scan results
- **TTL**: Configurable (default 5 minutes for items, 5 minutes for queries)
- **Encryption**: At rest (KMS) and in transit (TLS)
- **VPC only**: DAX cluster lives in VPC (not accessible over internet)

**DAX vs ElastiCache for DynamoDB:**
| Aspect | DAX | ElastiCache |
|--------|-----|-------------|
| API compatibility | Drop-in (same DynamoDB API) | Custom code required |
| Write caching | Write-through (automatic) | Must implement manually |
| Use case | Simple DynamoDB acceleration | Complex caching logic, shared cache across services |
| Latency | Microseconds | Sub-millisecond |
| Exam tip | "DynamoDB read-heavy, minimal code change" | "Multiple data sources, custom caching logic" |

#### API Gateway Caching
- Cache API responses at the stage level
- Capacity: 0.5 GB to 237 GB
- TTL: 0-3600 seconds (default 300)
- Per-method override: Enable/disable per resource/method
- Cache key: Include headers, query strings, or path parameters
- **Invalidation**: Client sends `Cache-Control: max-age=0` (requires IAM permission)
- Cost: $0.02-$3.80/hour depending on cache size

#### CloudFront Caching
- **Edge Location caching** — content cached at 400+ edge locations globally
- **Regional Edge Cache**: Additional caching layer between origin and edge locations
- **Cache behaviors**: Rules for how different URL patterns are cached
- **TTL**: Minimum (0), Default (86400s = 24h), Maximum (31536000s = 1 year)
- **Cache Policy**: Controls which headers/cookies/query strings are included in cache key
- **Origin Request Policy**: Controls which values are forwarded to origin (separate from cache key)
- **Invalidation**: Remove objects from all edge caches (first 1000/month free, then $0.005 each)
- **Versioned URLs**: Better than invalidation (e.g., `/v2/style.css` or `/style.css?v=2`)

---

### CDN (CloudFront)

**Core Concepts:**
- **Distribution**: CloudFront configuration (contains origins, behaviors, settings)
- **Origin**: Source of content (S3, ALB, EC2, API Gateway, custom HTTP server, MediaStore)
- **Edge Location**: Cache location closest to users (~400+ globally)
- **Regional Edge Cache**: Larger cache between edge and origin (13 locations)
- **Behavior**: Rules matching URL pattern to origin + cache settings
- **Origin Group**: Two origins for origin failover (primary + secondary)

**Distribution Types:**
- **Web Distribution**: HTTP/HTTPS content (websites, APIs, streaming)
- RTMP distribution: Deprecated (was for Adobe Flash streaming)

**Key Features:**
- **Lambda@Edge**: Run Lambda at edge locations (viewer-request, origin-request, origin-response, viewer-response)
- **CloudFront Functions**: Lightweight JavaScript at edge (viewer-request, viewer-response only) — 1/6 cost of Lambda@Edge
- **Origin Access Control (OAC)**: Restrict S3 access to CloudFront only (replaces Origin Access Identity)
- **Field-Level Encryption**: Encrypt specific POST fields at edge (only specific services can decrypt)
- **Signed URLs/Cookies**: Restrict access to content (time-limited, IP-restricted)
- **Geo-Restriction**: Block/allow countries
- **Price Classes**: Limit edge locations to reduce cost (All, 200, 100)

**CloudFront + S3:**
- OAC restricts S3 bucket access to CloudFront only
- S3 as origin: Static website hosting (HTTP) or S3 REST endpoint (OAC compatible)
- Transfer Acceleration (S3) vs CloudFront: CloudFront for reads, Transfer Acceleration for writes

**CloudFront + ALB/EC2:**
- ALB/EC2 must be publicly accessible (CloudFront connects over internet)
- Custom headers: Verify requests come from CloudFront (shared secret header + ALB rule)
- Or use AWS WAF on ALB to restrict to CloudFront IPs

**CloudFront + API Gateway:**
- API Gateway has built-in edge-optimized endpoints (uses CloudFront internally)
- Custom CloudFront distribution in front of Regional API Gateway gives more control (caching, WAF, Lambda@Edge)

**Key Limits:**
| Limit | Value |
|-------|-------|
| Distributions per account | 200 |
| Origins per distribution | 25 |
| Behaviors per distribution | 25 |
| Alternate domain names (CNAMEs) | 100 per distribution |
| Cache invalidation paths | 3,000 in progress |
| Lambda@Edge concurrent executions | 10,000 per region |
| CloudFront Functions max size | 10 KB |
| Lambda@Edge max size | 1 MB (viewer), 50 MB (origin) |

---

### Read Replicas

#### RDS Read Replicas
- **Asynchronous replication** from primary to replica
- **Read traffic offloaded** — application reads from replica, writes go to primary
- Up to **15 replicas** (Aurora) or **5 replicas** (RDS MySQL/PostgreSQL/MariaDB)
- **Cross-region replicas**: For global read performance and DR
- **Can be promoted**: To standalone DB (breaks replication — for DR)
- **Independent endpoint**: Each replica has its own DNS endpoint (application must manage routing)
- **Replication lag**: Milliseconds to seconds (depends on write volume)

**RDS Replica Features by Engine:**
| Feature | MySQL/MariaDB | PostgreSQL | Oracle | SQL Server |
|---------|---------------|------------|--------|------------|
| Max replicas | 5 | 5 | 5 (Active Data Guard) | 5 |
| Cross-region | Yes | Yes | No | No |
| Multi-AZ replica | Yes | Yes | Yes | Yes |
| Cascade replicas | Yes (MySQL) | No | No | No |
| Different instance class | Yes | Yes | Yes | Yes |

#### Aurora Read Replicas
- Up to **15 read replicas** (in-region, same cluster)
- **Shared storage**: All replicas read from same distributed storage (not replication lag from writes)
- **Replica lag**: Typically < 20ms (much lower than RDS replicas)
- **Auto Scaling**: Aurora Auto Scaling adds/removes replicas based on load
- **Reader Endpoint**: Load-balanced across all replicas (single connection string)
- **Custom Endpoints**: Route specific queries to specific replicas (e.g., analytics queries to large replicas)
- **Aurora Global Database**: Cross-region read replicas (< 1 second lag, up to 5 secondary regions)
- **Failover priority**: Tiers 0-15 (lower = higher priority for promotion)

#### Aurora Auto Scaling for Replicas
- Target Tracking: Maintain CPU or connections at target
- Min/Max replicas: Define bounds
- Scales in/out based on demand
- Different from RDS (which has no built-in replica auto scaling)

---

### AWS Global Accelerator

**Core Concepts:**
- **Anycast IPs**: Two static IPv4 addresses that route to your applications globally
- **AWS Global Network**: Traffic enters AWS edge locations and travels over AWS backbone (not public internet)
- **Endpoint Groups**: Regional groupings of endpoints (ALBs, NLBs, EC2, Elastic IPs)
- **Endpoints**: The actual targets (with weights)
- **Listener**: Protocol/port configuration (TCP, UDP)
- **Health Checks**: Continuously monitors endpoint health

**How It Works:**
```
User → Nearest Edge Location (anycast) → AWS Global Network → Endpoint Group (Region) → Endpoint
```

**Key Features:**
- **Static IP addresses**: 2 anycast IPs (simplifies DNS/firewall, never changes)
- **Instant failover**: < 30 seconds (health check failure → traffic rerouted)
- **Traffic dials**: Percentage of traffic to each endpoint group (for blue/green, migrations)
- **Endpoint weights**: Within a group, weight traffic between endpoints
- **Client affinity**: Sticky sessions by source IP (for stateful apps)
- **Custom routing**: Direct specific users to specific endpoints (gaming, VoIP)
- **DDoS protection**: AWS Shield Standard included automatically

**Global Accelerator vs CloudFront:**
| Aspect | Global Accelerator | CloudFront |
|--------|-------------------|------------|
| **Purpose** | Network-level acceleration (TCP/UDP) | Content caching + HTTP acceleration |
| **Caching** | NO caching | YES (edge caching) |
| **Protocols** | TCP, UDP | HTTP, HTTPS, WebSocket |
| **Use case** | Non-HTTP (gaming, VoIP, IoT), failover, static IPs | Web content, APIs, streaming |
| **Static IPs** | Yes (2 anycast) | No (uses DNS CNAME) |
| **Failover speed** | Seconds (health check) | Depends on TTL (DNS) |
| **DDoS** | Shield Standard + Advanced | Shield Standard + Advanced |
| **Pricing** | $0.025/hour + data transfer premium | Per-request + data transfer |

**Key Limits:**
| Limit | Value |
|-------|-------|
| Accelerators per account | 20 |
| Listeners per accelerator | 10 |
| Endpoint groups per listener | 10 (one per region) |
| Endpoints per endpoint group | 10 |
| Static IPs per accelerator | 2 |

---

## 2. Design Patterns & Best Practices

### When to Use Which

| Solution | Use When | Don't Use When |
|----------|----------|----------------|
| ElastiCache Redis | Complex caching, sessions, leaderboards, pub/sub | Simple DynamoDB reads (use DAX) |
| ElastiCache Memcached | Simple key-value, multi-threaded, disposable | Need persistence, replication, complex data types |
| DAX | DynamoDB read acceleration, minimal code change | Need cross-service cache, non-DynamoDB sources |
| CloudFront | Static/dynamic web content, global users, S3 distribution | Non-HTTP protocols, real-time gaming |
| Global Accelerator | TCP/UDP acceleration, static IPs, instant failover | HTTP caching (CloudFront better), cost-sensitive |
| RDS Read Replicas | Read-heavy SQL workloads, cross-region reads | Strong consistency required (replicas are async) |
| Aurora Replicas | High read throughput, auto-scaling reads, low replication lag | Cross-engine compatibility (vendor lock-in concern) |
| API Gateway Cache | Reduce backend calls for API responses | Highly dynamic/personalized responses |

### Anti-Patterns
- **Don't cache everything** — cache only data that's read frequently and changes infrequently
- **Don't use CloudFront for non-cacheable, highly personalized content** — cache hit ratio will be near zero, adds latency
- **Don't use Global Accelerator when CloudFront suffices** — GA is more expensive and doesn't cache
- **Don't use read replicas for strong consistency** — replication is async; use primary for consistency-critical reads
- **Don't use DAX for write-heavy DynamoDB** — DAX excels at reads; writes still go to DynamoDB
- **Don't create cross-region read replicas just for DR** — consider Aurora Global Database (faster failover, managed)

### Architectural Patterns

**Multi-Layer Caching:**
```
Client → CloudFront (edge cache, static content)
       → API Gateway (API response cache)
       → ElastiCache Redis (application data cache)
       → RDS/DynamoDB (database)
```
- Each layer reduces load on the next
- Different TTLs per layer (shorter for dynamic, longer for static)

**Global Low-Latency Architecture:**
```
Users worldwide → CloudFront (content) + Global Accelerator (APIs)
                → Regional deployments:
                    └── ALB → ECS/EC2 (compute)
                    └── Aurora Global Database (write-primary in one region, read-replicas in all regions)
                    └── ElastiCache Global Datastore (cross-region session cache)
                    └── DynamoDB Global Tables (multi-region active-active)
```

**Read-Heavy Workload Pattern:**
```
Application →  Is in cache? 
               ├── YES → Return (ElastiCache, <1ms)
               └── NO → Read Replica (<10ms) → Store in cache → Return
                         ├── Replica lag acceptable? → Use replica
                         └── Need latest? → Read from primary
```

**Session Management:**
```
User → ALB (sticky sessions OFF) → EC2 instances (stateless)
                                        └── Session stored in ElastiCache Redis
                                            └── Any instance can serve any user
```
- Enables true horizontal scaling (no sticky sessions needed)
- Session survives instance failure
- Redis persistence ensures session survival across restarts

**CloudFront + S3 + Lambda@Edge (Serverless Website):**
```
User → CloudFront → Edge Function (auth, redirect, A/B test)
     → S3 (static assets, OAC protected)
     → API Gateway/ALB (dynamic content)
     → Origin failover (S3 backup if primary fails)
```

**Database Read Scaling Pattern:**
```
Aurora Cluster:
  Writer Endpoint → Primary instance (writes + consistency-critical reads)
  Reader Endpoint → Auto-Scaled replicas (0-15, load balanced)
  Custom Endpoint → Analytics replicas (large instance, separate workload)
```

### Well-Architected Alignment

| Pillar | Practice |
|--------|----------|
| Reliability | Origin failover (CloudFront), multi-AZ caching, Global Accelerator failover, replica promotion for DR |
| Security | OAC for S3, signed URLs/cookies, field-level encryption, encryption at rest/transit for all caches |
| Cost | CloudFront Price Classes, right-sized cache nodes, Reserved Nodes for ElastiCache, efficient invalidation |
| Performance | Multi-layer caching, edge computing (Lambda@Edge), read replicas for read-heavy, Global Accelerator for latency |
| Ops Excellence | Cache hit ratio monitoring, replication lag monitoring, origin health checks, auto-scaling replicas |

### Integration Points
- **Route 53**: Alias records for CloudFront distributions, latency-based routing to regions
- **WAF**: Attached to CloudFront or ALB for security
- **Shield Advanced**: DDoS protection for CloudFront and Global Accelerator
- **ACM**: SSL/TLS certificates for CloudFront (must be in us-east-1)
- **S3**: Origin for CloudFront, Transfer Acceleration for uploads
- **Lambda@Edge / CloudFront Functions**: Edge compute for customization
- **Auto Scaling**: Aurora replica auto scaling, ElastiCache scaling
- **CloudWatch**: Cache hit ratio, replication lag, latency metrics
- **Secrets Manager**: Database credentials for read replicas

---

## 3. Security & Compliance

### Caching Security

**ElastiCache:**
- **VPC only**: Cannot be accessed outside VPC (use VPN/Direct Connect for on-prem access)
- **Security groups**: Control which clients can connect
- **Redis AUTH**: Password-based authentication (single token)
- **Redis RBAC**: Role-based access control (multiple users with different permissions) — Redis 6+
- **Encryption at rest**: KMS (AES-256)
- **Encryption in transit**: TLS (must be enabled at cluster creation, cannot add later)
- **Subnet groups**: Control which subnets the cluster uses
- **Automatic backups**: Redis only (daily snapshots, retained 1-35 days)

**DAX:**
- VPC only (subnet group required)
- IAM policies control access (standard DynamoDB IAM + DAX-specific permissions)
- Encryption at rest (KMS)
- Encryption in transit (TLS)
- Cluster endpoint accessible only within VPC

### CDN Security

**CloudFront:**
- **OAC (Origin Access Control)**: Restricts S3 bucket access to CloudFront (S3 bucket policy)
- **Signed URLs**: Time-limited access to specific files (individual file restriction)
- **Signed Cookies**: Time-limited access to multiple files (multiple file restriction)
- **Geo-Restriction**: Allow/block by country
- **AWS WAF**: Attach WAF WebACL to distribution (rate limiting, SQL injection, XSS protection)
- **Field-Level Encryption**: Encrypt specific POST fields at edge with public key (only destination can decrypt with private key)
- **SSL/TLS**: Custom SSL certificate (ACM in us-east-1), TLS 1.2 minimum, SNI or dedicated IP
- **Security headers**: Add via CloudFront Functions (HSTS, X-Frame-Options, etc.)

**CloudFront Signed URLs vs Cookies:**
| Feature | Signed URLs | Signed Cookies |
|---------|-------------|----------------|
| Scope | Single file | Multiple files (entire path) |
| Use case | Individual file downloads | Streaming site, protected area |
| Client change | URL itself is different | Cookie added to requests |
| RTMP | Supported (legacy) | Not supported |

### Global Accelerator Security
- **AWS Shield Standard**: Automatic DDoS protection
- **Shield Advanced**: Enhanced DDoS protection + 24/7 DRT support
- **Security groups**: On endpoints (ALB, EC2)
- **Static IPs**: Easier to whitelist in firewalls than DNS-based solutions
- **Client IP preservation**: Original client IP visible to application (for ALB/EC2 endpoints)

### Read Replica Security
- **Encryption**: If primary is encrypted, replicas are encrypted (same KMS key for same-region)
- **Cross-region replicas**: Can use different KMS key in target region
- **Network**: Replicas in same VPC (or peered VPC for cross-region)
- **IAM Authentication**: Works with replicas (separate IAM-to-DB user mapping)
- **SSL/TLS**: Enforce with `rds.force_ssl` parameter

---

## 4. Cost Optimization

### Pricing Summary

**ElastiCache:**
| Item | Price (example: cache.r6g.large, us-east-1) |
|------|------|
| On-Demand | ~$0.182/hour per node |
| Reserved (1yr, no upfront) | ~30% savings |
| Reserved (3yr, all upfront) | ~55% savings |
| Data transfer (same AZ) | Free |
| Data transfer (cross-AZ) | Standard EC2 data transfer rates |
| Backup storage | Free (up to cluster size), $0.085/GB beyond |

**CloudFront:**
| Item | Price |
|------|-------|
| Data transfer (first 10 TB) | $0.085/GB (US/Europe) |
| HTTP requests | $0.0075/10,000 |
| HTTPS requests | $0.01/10,000 |
| Lambda@Edge | $0.60/million requests + $0.00005001/GB-sec |
| CloudFront Functions | $0.10/million invocations |
| Invalidation | First 1,000/month free, then $0.005 each |
| Origin Shield | $0.0090/10,000 (additional layer) |

**Global Accelerator:**
| Item | Price |
|------|-------|
| Fixed fee | $0.025/hour per accelerator (~$18/month) |
| Data Transfer Premium (DTP) | $0.015-$0.035/GB (on top of standard transfer) |

**RDS Read Replicas:**
- Same pricing as primary instance (per-hour instance cost)
- Cross-region replication: Data transfer charges apply ($0.02/GB)
- No additional licensing (included in RDS pricing)

**Aurora Read Replicas:**
- Same pricing as Aurora instances
- Shared storage (no additional storage cost per replica)
- Auto-scaling replicas: Pay only when running

### Cost-Saving Strategies

**Caching:**
1. **Reserved Nodes** for ElastiCache (30-55% savings for predictable workloads)
2. **Right-size cache nodes** — monitor memory utilization, scale down if underutilized
3. **Use TTL aggressively** — prevent cache from growing unboundedly
4. **DAX vs ElastiCache**: DAX is simpler for DynamoDB-only caching (less operational cost)
5. **Serverless cache (ElastiCache Serverless)**: Pay per GB/hour stored + ECPUs processed (better for variable workloads)

**CDN:**
1. **CloudFront Price Class 100/200**: Limit to cheaper regions if users are concentrated
2. **Maximize cache hit ratio**: Include only necessary values in cache key
3. **Use Origin Shield**: Additional caching layer reduces origin requests (saves origin costs)
4. **Versioned URLs over invalidations**: Free (new version = new cache entry) vs $0.005/invalidation
5. **Compress content**: Enable Gzip/Brotli (smaller data transfer = lower cost)
6. **CloudFront Functions over Lambda@Edge**: 1/6 the cost for viewer-request/response logic

**Read Replicas:**
1. **Right-size replicas**: Replicas don't need same size as primary (can be smaller for read-only)
2. **Aurora Auto Scaling**: Scale replicas in/out based on demand (don't over-provision)
3. **Use Reader Endpoint**: Load balance across replicas efficiently
4. **Consider Aurora Serverless v2**: Scales to zero readers when no read load

**Global Accelerator:**
1. **Only use when needed**: If CloudFront solves the problem, it's cheaper (no hourly fee)
2. **Remove unused accelerators**: $18/month even with no traffic
3. **Traffic dials**: Route traffic away from expensive regions during testing

### Cost Traps
- **ElastiCache without Reserved Nodes** for stable workloads (3yr RI = 55% savings left on table)
- **CloudFront with low cache hit ratio** — paying for CDN but still hitting origin constantly (misconfigured cache key)
- **Invalidations instead of versioned URLs** — costs $0.005 each, versioning is free
- **Global Accelerator for HTTP-only workloads** — CloudFront would cache AND accelerate, GA only accelerates
- **Cross-region read replicas you don't read from** — paying for instance + data transfer
- **Over-provisioned read replicas** — 5 replicas when 2 would suffice for the read load

---

## 5. High Availability, Disaster Recovery & Resilience

### Caching HA

**ElastiCache Redis:**
- **Multi-AZ with Auto-Failover**: Primary fails → replica promoted (seconds)
- **Cluster Mode Enabled**: Data sharded + replicated (tolerates individual node failures without data loss)
- **Global Datastore**: Cross-region replication for DR (promote secondary region to primary in < 1 minute)
- **Automatic backups**: Point-in-time recovery within retention period
- **Snapshot before maintenance**: Can restore if maintenance causes issues

**DAX:**
- Multi-AZ cluster (primary + replicas in different AZs)
- Automatic failover if primary fails
- At least 3 nodes recommended for production HA

### CDN HA

**CloudFront:**
- **Globally distributed**: No single point of failure (400+ edge locations)
- **Origin Failover (Origin Group)**: Primary origin fails → CloudFront automatically uses secondary origin
  - Failover criteria: 5xx errors, timeout, connection failed from primary
  - Use case: S3 primary origin → S3 secondary in different region
- **Regional Edge Cache**: Additional resilience layer (survives edge failures)
- **Always-on**: CloudFront is one of the most resilient AWS services

### Read Replica HA

**RDS:**
- **Multi-AZ + Read Replicas**: Primary (Multi-AZ sync replica for HA) + Read Replicas (async for read scaling)
- **Promote replica for DR**: Cross-region replica → promote to standalone primary in DR region
- **Replication lag monitoring**: CloudWatch `ReplicaLag` metric + alarm

**Aurora:**
- **Shared distributed storage**: 6 copies across 3 AZs (tolerates AZ+1 failure for reads, AZ failure for writes)
- **Auto-failover**: Promotes highest-priority replica (< 30 seconds)
- **Aurora Global Database**: Cross-region failover < 1 minute (managed RPO < 1 second)
- **Reader Endpoint**: Auto-routes away from failed replicas

### Global Accelerator HA
- **Instant failover**: Health checks fail → traffic routed to next-closest healthy endpoint (< 30 seconds)
- **Multi-region**: Endpoints across multiple regions for full regional failure tolerance
- **No DNS propagation delay**: Anycast IPs mean failover happens at network level (no TTL wait)
- **Health check interval**: As low as 10 seconds (vs 30-60s typical)

### DR Patterns

| Solution | RPO | RTO | DR Approach |
|----------|-----|-----|-------------|
| ElastiCache Global Datastore | < 1 second | < 1 minute | Promote secondary region |
| CloudFront + S3 CRR | Near-zero (CRR replication) | Automatic (origin failover) | Origin group failover |
| Aurora Global Database | < 1 second | < 1 minute | Promote secondary cluster |
| RDS Cross-Region Replica | Seconds (replication lag) | Minutes (promotion time) | Promote replica |
| Global Accelerator | N/A | < 30 seconds | Automatic endpoint failover |

---

## 6. Exam-Focused Section

### Straightforward Questions (7 examples)

**Q1:** A company's DynamoDB-backed application has 90% reads. Read latency is too high. What's the simplest way to improve read performance with minimal code changes?
**A:** **DAX (DynamoDB Accelerator)** — drop-in replacement for DynamoDB API calls, provides microsecond read latency, requires minimal code change (just change the client endpoint).

**Q2:** A company needs to serve static website content to users globally with low latency. The content is stored in S3. What's the best approach?
**A:** **CloudFront distribution with S3 origin** and OAC (Origin Access Control) to restrict direct S3 access. CloudFront caches content at edge locations globally.

**Q3:** A company's RDS MySQL database is overwhelmed by read queries from a reporting application. How can they offload the reads?
**A:** Create **RDS Read Replicas** and point the reporting application to the replica endpoint. This offloads read traffic from the primary.

**Q4:** A company needs static IP addresses for their global application (firewall whitelisting requirement) with automatic failover between regions. What service provides this?
**A:** **AWS Global Accelerator** — provides 2 static anycast IP addresses with automatic health-check-based failover between regional endpoints.

**Q5:** What caching strategy should be used when data changes frequently and the application cannot tolerate stale data?
**A:** **Write-Through** caching — every write updates both the database AND the cache simultaneously, ensuring the cache is always fresh.

**Q6:** A company wants to restrict access to CloudFront-distributed premium video content to paying subscribers. What mechanism should they use?
**A:** **CloudFront Signed Cookies** — allow access to multiple files (video segments) for authenticated/paying users for a limited time period.

**Q7:** What is the maximum number of read replicas for an Aurora cluster?
**A:** **15 read replicas** in a single Aurora cluster, with automatic load balancing via the Reader Endpoint.

---

### Tricky / Scenario-Based Questions (7 examples)

**Q1:** A company has a global application with users in US, Europe, and Asia. They need sub-second failover between regions for a TCP-based custom protocol (not HTTP). CloudFront is not suitable. What should they use?

**A:** **AWS Global Accelerator** — it supports TCP and UDP protocols (not just HTTP), provides anycast static IPs, and failover is < 30 seconds (based on health checks). Traffic routes over AWS backbone for lower latency.

**Why wrong answers are wrong:**
- "CloudFront" — only supports HTTP/HTTPS/WebSocket, not arbitrary TCP protocols
- "Route 53 failover routing" — DNS-based, subject to TTL delays (not sub-second)
- "Multi-region ALB with Route 53 latency routing" — ALB is HTTP-only, and DNS failover is slow
- **Keywords:** "TCP-based custom protocol", "sub-second failover", "not HTTP" → Global Accelerator

---

**Q2:** A company uses ElastiCache Redis with Lazy Loading. They notice that after a cache flush, the application experiences extreme latency for several minutes. How can they prevent this "thundering herd" problem?

**A:** Implement **cache warming** (pre-populate the cache with hot data before/after flush). Additionally, use **TTL with jitter** (add random variation to TTL values so entries don't all expire simultaneously). Another approach: **staggered expiration** — set different TTLs for different keys to avoid mass cache misses.

**Why wrong answers are wrong:**
- "Switch to Write-Through" — helps for individual keys but doesn't prevent the mass invalidation issue
- "Increase cache node size" — doesn't help; the issue is all requests hitting the database simultaneously
- "Add more read replicas" — reduces database impact but doesn't fix the caching issue
- **Keywords:** "cache flush", "extreme latency", "thundering herd" → Cache warming + TTL with jitter

---

**Q3:** A company serves an API through CloudFront + API Gateway. Some API responses are personalized (user-specific), while others are shared (product catalog). Currently, nothing is cached because personalization headers are in the cache key. How to optimize?

**A:** Create **multiple cache behaviors** in CloudFront: (1) Cache behavior for `/catalog/*` — exclude personalization headers from cache key (high cache hit ratio). (2) Cache behavior for `/user/*` — don't cache or use short TTL with the personalization header in cache key. Use **Cache Policy** and **Origin Request Policy** separately — forward headers to origin (for personalization) without including them in the cache key.

**Why wrong answers are wrong:**
- "Disable CloudFront caching entirely" — loses performance benefit for cacheable content
- "Cache all responses with short TTL" — personalized content shouldn't be cached (security + correctness)
- "Use Lambda@Edge for all personalization" — could work but over-engineers the caching issue
- **Keywords:** "mix of personalized and shared content", "cache hit ratio" → Separate cache behaviors + origin request policy

---

**Q4:** A company uses Aurora Global Database with primary in us-east-1 and read replica in eu-west-1. European users read from eu-west-1 replica. The team wants European users to write locally too (for lowest latency). What options exist?

**A:** Aurora Global Database supports **write forwarding** — secondary regions forward writes to the primary region transparently. Latency is higher than local writes (cross-region round trip) but lower than if the application had to manage it. For true multi-region writes, switch to **DynamoDB Global Tables** (active-active) if the data model allows.

**Why wrong answers are wrong:**
- "Create a second Aurora primary in eu-west-1" — Aurora doesn't support multi-primary across regions (only Global Database with one write region)
- "Use read replicas for writes" — read replicas can't accept writes
- "Promote eu-west-1 as primary" — loses us-east-1 as primary (can only have one writer)
- **Keywords:** "write locally in secondary region", "Aurora Global Database" → Write forwarding (or DynamoDB Global Tables for true multi-write)

---

**Q5:** A company uses CloudFront with an ALB origin. They notice that during an origin outage, ALL users get errors immediately. They want to serve stale cached content during origin failures. How?

**A:** Configure **Origin Failover** using an **Origin Group** with the ALB as primary and an S3 bucket (with cached/stale content) as secondary. Alternatively, use **stale-while-revalidate** behavior — CloudFront can serve stale objects when origin returns errors if `custom-error-response` is configured with `ErrorCachingMinTTL` (cache error responses briefly while retrying origin).

Also consider: CloudFront **custom error pages** — return a static error page from S3 instead of the raw error.

**Why wrong answers are wrong:**
- "Increase CloudFront TTL" — helps only if content was already cached (cache hit); doesn't help for cache misses
- "Add Global Accelerator" — doesn't cache, just routes to healthy endpoints (same origin is still down)
- "Use Route 53 health checks" — Route 53 could failover DNS but this is slower than origin groups
- **Keywords:** "serve stale during origin outage", "CloudFront" → Origin Group failover + custom error responses

---

**Q6:** A company has a leaderboard application requiring real-time ranking of millions of players with sub-millisecond reads. Which caching solution is most appropriate?

**A:** **ElastiCache Redis** with **Sorted Sets** — Redis sorted sets maintain elements ordered by score. Operations like `ZADD`, `ZRANGE`, `ZRANK` run in O(log N) with sub-millisecond latency. This is the purpose-built data structure for leaderboards.

**Why wrong answers are wrong:**
- "DAX" — DAX caches DynamoDB queries but doesn't provide sorted set operations
- "Memcached" — no sorted sets or complex data structures
- "DynamoDB with GSI" — possible but won't achieve sub-millisecond reads, and real-time ranking at scale is complex
- **Keywords:** "leaderboard", "real-time ranking", "sorted", "sub-millisecond" → Redis Sorted Sets

---

**Q7:** A company uses Global Accelerator with endpoints in us-east-1 and eu-west-1. They want to do a blue/green deployment, gradually shifting 10% of global traffic to the new version in eu-west-1. How?

**A:** Use **endpoint weights** within the eu-west-1 endpoint group (if blue/green are in same region) OR use **traffic dials** on endpoint groups to shift traffic percentage between regions. Set us-east-1 traffic dial to 90% and eu-west-1 to 100% (or adjust weights within a single endpoint group for same-region blue/green).

**Why wrong answers are wrong:**
- "Use Route 53 weighted routing" — adds DNS TTL delay, Global Accelerator's traffic dial is immediate
- "CloudFront with origin groups" — origin groups are for failover (not weighted traffic splitting)
- "ALB weighted target groups" — works within a region but doesn't handle global traffic splitting
- **Keywords:** "Global Accelerator", "gradually shift traffic", "blue/green" → Traffic dials (between groups) or endpoint weights (within group)

---

### Common Exam Traps & Pitfalls

1. **Global Accelerator does NOT cache** — it only optimizes the network path. For caching, use CloudFront.

2. **CloudFront is HTTP/HTTPS only** — for TCP/UDP, use Global Accelerator.

3. **CloudFront ACM certificates MUST be in us-east-1** — regardless of where the origin is.

4. **DAX is VPC-only** — cannot be accessed from outside VPC (Lambda must be in same VPC).

5. **Read replicas are EVENTUALLY consistent** — never use for reads that MUST return latest write.

6. **Aurora Reader Endpoint is not a load balancer with smart routing** — it's DNS round-robin. For connection pooling, use RDS Proxy.

7. **ElastiCache encryption in transit can't be added after creation** — must be specified at cluster creation time.

8. **CloudFront signed URLs vs signed cookies**: Signed URL = individual file. Signed cookie = multiple files/path.

9. **Origin Failover triggers on 5xx/timeout from origin** — NOT on CloudFront edge errors or cache misses.

10. **Global Accelerator traffic dial 0% ≠ no traffic** — it only applies to new connections; existing connections continue.

11. **Redis Cluster Mode Enabled vs Disabled**: Cluster Mode = sharding (more data capacity). Disabled = single shard (simpler, still has replicas for HA).

12. **CloudFront Functions vs Lambda@Edge**: Functions are cheaper and faster but limited (10 KB, no network calls, viewer events only). Lambda@Edge for origin events or complex logic.

---

## 7. Cheat Sheet

### Must-Know Facts

| Fact | Detail |
|------|--------|
| ElastiCache Redis max replicas | 5 per shard |
| ElastiCache Redis Cluster Mode max | 500 nodes |
| Aurora max read replicas | 15 |
| Aurora replica lag | < 20ms (shared storage) |
| RDS max read replicas | 5 |
| DAX latency | Microseconds |
| ElastiCache latency | Sub-millisecond |
| CloudFront edge locations | 400+ |
| CloudFront default TTL | 24 hours (86400s) |
| Global Accelerator IPs | 2 static anycast IPv4 |
| Global Accelerator failover | < 30 seconds |
| CloudFront cert region | us-east-1 (always) |
| CloudFront Functions size | 10 KB |
| Lambda@Edge viewer size | 1 MB |
| Lambda@Edge origin size | 50 MB |
| Redis AUTH | Single token (or RBAC in Redis 6+) |
| DAX VPC | Required (VPC-only) |
| ElastiCache encrypt in-transit | Must set at creation |

### Decision Flowchart

```
"Reduce DynamoDB read latency, minimal code change"
  → DAX

"Complex caching with data structures (leaderboards, sessions, pub/sub)"
  → ElastiCache Redis

"Simple disposable cache, multi-threaded"
  → ElastiCache Memcached

"Serve static content globally"
  → CloudFront + S3

"Accelerate dynamic HTTP API globally"
  → CloudFront (with caching where possible)

"Accelerate TCP/UDP non-HTTP protocol globally"
  → Global Accelerator

"Need static IP addresses for firewall whitelisting"
  → Global Accelerator

"Instant failover (seconds) between regions"
  → Global Accelerator (network-level) — faster than DNS

"Offload read traffic from SQL database"
  → Read Replicas (RDS/Aurora)

"Auto-scale read capacity for Aurora"
  → Aurora Auto Scaling (replica count)

"Global active-active database"
  → DynamoDB Global Tables (not Aurora — single write region only)

"Cross-region read acceleration for Aurora"
  → Aurora Global Database

"Restrict S3 access to CloudFront only"
  → OAC (Origin Access Control)

"Edge compute for request manipulation"
  → CloudFront Functions (simple/cheap) or Lambda@Edge (complex)

"Protect content with time-limited access"
  → CloudFront Signed URLs (single file) or Signed Cookies (multiple files)
```

---

## 8. Additional Deep-Dive

### 10 More Tricky Scenario-Based Questions

**Q1:** A company has a REST API behind CloudFront. Cache hit ratio is only 5% despite most responses being cacheable. What's likely wrong?

**A:** The **cache key** probably includes unnecessary values (all headers, all cookies, all query strings). Most requests differ by Authorization header or cookie, creating unique cache keys for each user. Fix: Use a **Cache Policy** that excludes Authorization header and user-specific cookies from the cache key. Forward them to origin via **Origin Request Policy** (separate from cache key).

**Keywords:** "low cache hit ratio", "responses are cacheable" → Cache key includes too many varying values

---

**Q2:** A company uses ElastiCache Redis Cluster Mode Enabled with 3 shards. They try to run a Lua script that accesses keys across multiple shards. The operation fails. Why?

**A:** In Redis Cluster Mode, **multi-key operations (including Lua scripts) must operate on keys in the SAME shard** (same hash slot). Keys are distributed across shards by hash slot. Fix: Use **hash tags** `{tag}` in key names to ensure related keys land on the same shard (e.g., `user:{123}:profile`, `user:{123}:sessions`).

**Keywords:** "Cluster Mode", "multi-key operation fails", "Lua script across shards" → Hash tags to co-locate keys

---

**Q3:** A company uses CloudFront with an ALB origin. Their application sets `Set-Cookie` headers on every response. CloudFront is caching the Set-Cookie responses and serving other users' cookies. How to fix?

**A:** Configure the cache behavior to **NOT cache responses that contain Set-Cookie headers** — use a response headers policy or configure CloudFront to forward cookies to origin BUT also include cookies in the cache key. Best fix: For dynamic responses with Set-Cookie, set `Cache-Control: no-cache` or `max-age=0` from the origin, or use a **separate cache behavior** with caching disabled for authenticated paths.

**Keywords:** "Set-Cookie cached and served to other users" → Cache key must include cookies, or don't cache responses with Set-Cookie

---

**Q4:** A company migrates from self-managed Redis to ElastiCache Redis. Their application uses Redis `KEYS *` command for scanning. In production (10M+ keys), this command causes multi-second latency spikes affecting all users. How to fix?

**A:** Replace `KEYS` with **`SCAN`** command — SCAN is cursor-based and non-blocking (returns results incrementally without blocking the single-threaded Redis). `KEYS *` blocks the entire Redis instance while scanning all keys. This is a well-known Redis anti-pattern.

**Keywords:** "KEYS command", "latency spikes", "Redis single-threaded blocking" → Use SCAN instead of KEYS

---

**Q5:** A company uses Aurora with 5 read replicas behind the Reader Endpoint. One replica has much higher CPU than others (80% vs 30%). Connections seem unevenly distributed. Why?

**A:** The Aurora **Reader Endpoint uses DNS round-robin** — it doesn't actively balance connections. Long-lived connections (connection pooling) concentrate on whichever node the DNS resolved to initially. Fix: Use **RDS Proxy** for intelligent connection multiplexing and load balancing, or use **Aurora Custom Endpoints** with application-level routing. Alternatively, implement shorter connection TTL in the connection pool.

**Keywords:** "uneven load across replicas", "Reader Endpoint", "connection pooling" → DNS round-robin doesn't rebalance long connections (use RDS Proxy)

---

**Q6:** A company uses Global Accelerator with endpoints in us-east-1 (weight 100) and us-west-2 (weight 100). They set the us-west-2 traffic dial to 0% for maintenance. Users in the western US report increased latency but no errors. Why?

**A:** Traffic dial at 0% means **no new connections** go to us-west-2, so western US users are routed to us-east-1 (farther away = higher latency). This is expected behavior — traffic dials shift traffic to other endpoint groups. Existing connections to us-west-2 may continue until they close. The latency increase is the cross-region distance to us-east-1.

**Keywords:** "traffic dial 0%", "increased latency" → Traffic rerouted to farther region (expected behavior)

---

**Q7:** A company wants to accelerate S3 uploads from global users. Should they use CloudFront or S3 Transfer Acceleration?

**A:** **S3 Transfer Acceleration** — specifically designed for accelerating uploads to S3 using AWS edge locations. CloudFront is primarily for downloads/reads (PUT support exists but Transfer Acceleration is optimized for writes with direct-to-S3 acceleration). Transfer Acceleration uses the same edge network but is purpose-built for S3 writes.

**Keywords:** "accelerate S3 uploads" → S3 Transfer Acceleration (not CloudFront)

---

**Q8:** A company uses ElastiCache Redis Global Datastore for cross-region session caching. They notice reads in the secondary region are stale by 1-2 seconds. The business requires sessions to be consistent within 200ms. What's the solution?

**A:** Global Datastore replication lag is typically < 1 second but can spike. For stronger consistency: (1) Read from primary region always (defeats the purpose of global read). (2) Use **DynamoDB Global Tables** instead (last-writer-wins, typically < 1 second). (3) Accept eventual consistency and design the application to handle stale session reads (most practical). (4) If 200ms is hard requirement, keep sessions in each region's local Redis and use sticky sessions at the DNS/GA level.

**Keywords:** "Global Datastore stale reads", "consistency requirement" → Accept eventual consistency or use different architecture (DynamoDB Global Tables, sticky sessions)

---

**Q9:** A company uses CloudFront with a custom origin (EC2 fleet). They want to reduce the number of requests reaching the origin during traffic spikes. Currently, each edge location makes separate requests to origin for the same content. What feature helps?

**A:** **Origin Shield** — an additional centralized caching layer in a designated region. All edge locations check Origin Shield before going to origin. This collapses multiple edge requests into one origin request (especially during cold-cache scenarios). It reduces origin load and improves cache hit ratio.

**Keywords:** "reduce origin requests", "multiple edges requesting same content" → CloudFront Origin Shield

---

**Q10:** A company has an application with both read-heavy queries (product catalog) and latency-sensitive writes (order placement). The RDS MySQL primary is overwhelmed. They add 3 read replicas. Reads improve but write latency increases. Why?

**A:** Read replicas cause **replication overhead on the primary** — the primary must send binary log events to all 3 replicas. More replicas = more replication work on primary = higher write latency. Fix: (1) Reduce replica count. (2) Migrate to **Aurora** (shared storage architecture — replicas don't burden the primary with replication). (3) Use ElastiCache for reads instead of replicas (zero replication overhead on primary).

**Keywords:** "replicas added", "write latency increases" → Replication overhead on primary (migrate to Aurora or use caching instead)

---

### Compare: CloudFront vs Global Accelerator vs S3 Transfer Acceleration

| Aspect | CloudFront | Global Accelerator | S3 Transfer Acceleration |
|--------|-----------|-------------------|--------------------------|
| **Use Case** | Cache + deliver HTTP content | Accelerate TCP/UDP, failover | Accelerate S3 uploads |
| **Caching** | YES (primary purpose) | NO | NO |
| **Protocols** | HTTP, HTTPS, WebSocket | TCP, UDP | HTTPS (S3 API) |
| **Static IPs** | No (DNS CNAME) | Yes (2 anycast) | No (bucket endpoint) |
| **Failover** | Origin groups (5xx-triggered) | Health check (< 30s) | N/A |
| **Pricing** | Per-request + data transfer | $0.025/hr + DTP | $0.04-$0.08/GB |
| **Exam Tip** | "Cache", "web content", "static" | "TCP/UDP", "static IPs", "instant failover" | "Upload to S3 faster" |

### Compare: ElastiCache Redis vs DAX vs CloudFront Cache

| Aspect | ElastiCache Redis | DAX | CloudFront Cache |
|--------|------------------|-----|-----------------|
| **Use Case** | General-purpose caching, sessions, pub/sub | DynamoDB read acceleration | HTTP response caching at edge |
| **Latency** | Sub-millisecond | Microseconds | Varies (edge proximity) |
| **Data Source** | Any (app-managed) | DynamoDB only | HTTP origin |
| **Code Change** | Significant (cache logic) | Minimal (same DynamoDB API) | None (CDN config) |
| **Persistence** | Yes (RDB + AOF) | No (through-cache to DDB) | TTL-based |
| **Exam Tip** | "Sessions", "leaderboard", "complex cache" | "DynamoDB reads", "minimal change" | "Static content", "global delivery" |

### When Does SAP-C02 Expect Which Performance Solution?

**Pick CloudFront when:**
- "Global users accessing web content"
- "Reduce latency for static/dynamic HTTP"
- "S3 distribution"
- "Edge computing (Lambda@Edge)"
- "Signed URLs/Cookies"

**Pick Global Accelerator when:**
- "TCP/UDP non-HTTP protocol"
- "Static IP addresses needed"
- "Instant (sub-second) failover"
- "Gaming, VoIP, IoT"
- "No caching needed, just faster network path"

**Pick ElastiCache Redis when:**
- "Session management"
- "Leaderboards / sorted data"
- "Pub/sub"
- "Rate limiting"
- "Complex caching with data structures"

**Pick DAX when:**
- "DynamoDB reads too slow"
- "Minimal code change"
- "Microsecond latency"

**Pick Read Replicas when:**
- "Database read-heavy"
- "Offload reporting/analytics queries"
- "Cross-region read performance"

**Pick Aurora when:**
- "Need more than 5 read replicas"
- "Lower replication lag needed"
- "Auto-scaling read capacity"
- "Global Database (cross-region)"

### Decision Tree: Performance Solution Selection

```
Is the content STATIC and served over HTTP?
  └── YES → CloudFront (+ S3 origin)

Is it a DYNAMIC API response (HTTP)?
  └── YES → Is it cacheable?
            ├── YES → CloudFront (short TTL) + API Gateway Cache
            └── NO → Global Accelerator (network optimization only)

Is it a NON-HTTP protocol (TCP/UDP)?
  └── YES → Global Accelerator

Is it a DATABASE read bottleneck?
  └── YES → What database?
            ├── DynamoDB → DAX (first choice) or ElastiCache
            ├── Aurora → Add Read Replicas (up to 15) + Auto Scaling
            ├── RDS MySQL/PostgreSQL → Add Read Replicas (up to 5)
            └── Any → ElastiCache (application-level caching)

Need GLOBAL LOW-LATENCY WRITES?
  └── DynamoDB Global Tables (active-active multi-region)

Need CROSS-REGION READ acceleration?
  └── Aurora Global Database or RDS Cross-Region Replicas

Need SESSION STORE with HA?
  └── ElastiCache Redis (Multi-AZ, persistence)

Need INSTANT REGIONAL FAILOVER?
  └── Global Accelerator (< 30s) > Route 53 (depends on TTL)

Need to UPLOAD files faster to S3?
  └── S3 Transfer Acceleration
```
