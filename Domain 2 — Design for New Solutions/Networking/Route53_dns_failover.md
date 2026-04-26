# Route 53 — Routing Policies, Health Checks & DNS Failover — SAP-C02 Study Notes

---

## 1. Core Concepts & Theory

### What is Amazon Route 53?

Amazon Route 53 is a **highly available, scalable, authoritative DNS service** that also provides **domain registration**, **DNS routing**, and **health checking**. It has a **100% availability SLA** — the only AWS service with this guarantee.

Route 53 is a **global service** (not region-scoped), but health checks run from health checkers in multiple AWS regions.

### DNS Fundamentals for the Exam

| Record Type | Purpose | Example |
|-------------|---------|---------|
| **A** | Maps hostname to IPv4 address | `app.example.com -> 1.2.3.4` |
| **AAAA** | Maps hostname to IPv6 address | `app.example.com -> 2001:db8::1` |
| **CNAME** | Maps hostname to another hostname | `www.example.com -> app.example.com` |
| **Alias** | Route 53-specific. Maps hostname to AWS resource | `example.com -> d1234.cloudfront.net` |
| **MX** | Mail exchange server | Priority + mail server hostname |
| **NS** | Name server for the hosted zone | Delegated NS records |
| **SOA** | Start of Authority — zone metadata | Serial, refresh, retry, expire, TTL |
| **TXT** | Arbitrary text (SPF, DKIM, domain verification) | `"v=spf1 include:..."` |
| **SRV** | Service locator | Protocol, port, priority, weight, target |
| **PTR** | Reverse DNS lookup (IP to hostname) | `4.3.2.1.in-addr.arpa -> app.example.com` |

### Alias Records vs CNAME

This is **heavily tested** on the exam:

| Feature | CNAME | Alias |
|---------|-------|-------|
| Zone apex (naked domain) | **Cannot** use at zone apex (example.com) | **Can** use at zone apex |
| Query charges | Standard DNS query charges | **Free** for queries to AWS resources |
| Target | Any hostname | Only AWS resources (ELB, CloudFront, S3 website, API Gateway, VPC endpoint, Elastic Beanstalk, Global Accelerator, another Route 53 record) |
| Health check association | Via routing policy | Direct association possible |
| TTL | You set it | Automatically set by Route 53 (you cannot override) |
| DNS protocol | Standard CNAME record | Route 53 answers with the target's A/AAAA record directly |

**Exam rule:** If the question involves a **zone apex** (e.g., `example.com` not `www.example.com`), the answer is always **Alias record**, never CNAME.

**Alias targets you must memorize:**
- Elastic Load Balancer (ALB, NLB, CLB)
- CloudFront distribution
- S3 website endpoint (NOT the S3 REST API endpoint)
- API Gateway
- VPC Interface Endpoint
- Elastic Beanstalk environment
- AWS Global Accelerator
- Another Route 53 record in the same hosted zone

**What is NOT an Alias target:**
- EC2 instance public IP (use A record)
- RDS endpoint (use CNAME)
- On-premises hostname (use CNAME)

### Hosted Zones

**Public Hosted Zone:**
- Contains records that define how to route traffic on the internet.
- Created automatically when you register a domain with Route 53.
- Can be used with domains registered elsewhere (update NS records at registrar).
- Cost: $0.50/month per hosted zone.

**Private Hosted Zone (PHZ):**
- Contains records that define how to route traffic within one or more VPCs.
- Must be associated with at least one VPC.
- Can be associated with VPCs in **different accounts** (using RAM or CLI authorization).
- Can be associated with VPCs in **different regions**.
- Requires `enableDnsHostnames` and `enableDnsSupport` on the VPC.
- **Exam tip:** PHZ associated with multiple VPCs allows centralized DNS for private resources across accounts/regions.

### Routing Policies

#### 1. Simple Routing

- Single record, one or more values.
- If multiple values, Route 53 returns **all values in random order**. Client chooses one.
- **No health checks** can be associated.
- Use case: single resource, no failover needed.

#### 2. Weighted Routing

- Distribute traffic across multiple resources by **percentage**.
- Each record has a weight (0-255). Traffic proportion = record weight / sum of all weights.
- Weight of 0 = no traffic sent to that resource (unless all records have weight 0, then equal distribution).
- **Health checks can be associated** — unhealthy records are excluded from responses.
- Use case: A/B testing, gradual migration (blue-green), load distribution.

**Example:**
```
app.example.com  A  1.2.3.4  weight=70  (70% traffic)
app.example.com  A  5.6.7.8  weight=30  (30% traffic)
```

#### 3. Latency-Based Routing

- Routes to the region with **lowest latency** for the end user.
- Route 53 maintains a latency database of approximate latency between user locations and AWS regions.
- Each record is associated with an **AWS region**.
- **Health checks can be associated.**
- Use case: multi-region applications optimizing user experience.
- **Not the same as geographic** — a user in Germany might be routed to us-east-1 if latency is lower than eu-west-1 at that moment.

#### 4. Failover Routing

- **Active-passive** failover. Primary and secondary records.
- Primary record **must** have a health check.
- If primary health check fails, Route 53 returns the secondary record.
- Secondary record can optionally have a health check.
- If both are unhealthy and secondary has a health check, Route 53 returns **no answer**.
- If secondary has no health check, it is always considered healthy (always returned when primary fails).
- Use case: DR failover to a static S3 website, standby region.

#### 5. Geolocation Routing

- Routes based on the **geographic location** of the user (continent, country, or US state).
- **Does NOT measure latency** — purely geographic mapping.
- You must create a **default record** for users whose location doesn't match any specific record. Without a default, those users get **no answer** (NXDOMAIN).
- More specific location wins: US-state > country > continent > default.
- **Health checks can be associated.**
- Use case: content localization, regulatory compliance (serve users in EU from EU region), restricting content by geography.

**Exam distinction:** Geolocation = "users in France must be served from eu-west-1" (legal/compliance). Latency = "serve users from the fastest region" (performance).

#### 6. Geoproximity Routing (Traffic Flow only)

- Routes based on geographic location of users AND resources.
- Supports **bias** values (-99 to +99) to shift traffic toward or away from a resource.
- Positive bias = attract more traffic. Negative bias = repel traffic.
- Works with AWS resources (auto-detected location) and non-AWS resources (you specify latitude/longitude).
- **Requires Route 53 Traffic Flow** (visual editor + traffic policies).
- Use case: gradually shift traffic from one region to another, custom geographic routing.

**Exam tip:** If the question says "shift more traffic to a specific region" or "expand the geographic area served by a region" — think Geoproximity with bias.

#### 7. Multivalue Answer Routing

- Returns **up to 8 healthy records** in response to a DNS query.
- Each record can have its own health check. Unhealthy records are excluded.
- Client-side selection from the healthy set (pseudo-random).
- **Not a substitute for a load balancer** — but provides client-side load balancing with health checking.
- Use case: return multiple IPs for client-side failover, when you need simple health-checked multi-value responses.

**Exam distinction:** Simple routing returns all values (no health checks). Multivalue returns up to 8 **healthy** values (with health checks).

#### Routing Policy Comparison Table

| Policy | Health Check? | Use Case | Key Differentiator |
|--------|--------------|----------|--------------------|
| Simple | No | Single resource | No failover, returns all values |
| Weighted | Yes | A/B testing, migration | Percentage-based distribution |
| Latency | Yes | Multi-region performance | Lowest network latency |
| Failover | Yes (primary required) | Active-passive DR | Primary/secondary with automatic failover |
| Geolocation | Yes | Compliance, localization | User's geographic location (not latency) |
| Geoproximity | Yes | Shift traffic between regions | Bias to expand/shrink region's catchment |
| Multivalue | Yes | Client-side load balancing | Up to 8 healthy IPs returned |

### Health Checks

#### Types of Health Checks

**1. Endpoint Health Check:**
- Monitors an endpoint you specify (IP or domain name).
- Route 53 health checkers send requests from **15+ global locations**.
- Protocols: HTTP, HTTPS, or TCP.
- For HTTP/HTTPS: health check passes if endpoint returns **2xx or 3xx** status code within the timeout.
- Can optionally check for a specific **string in the response body** (first 5,120 bytes).
- Configurable: interval (10s or 30s), failure threshold (1-10, default 3), regions.
- **Important:** Health checkers access your endpoint from the **public internet**. Security groups/NACLs must allow traffic from Route 53 health checker IP ranges.

**2. Calculated Health Check:**
- Combines the results of **multiple child health checks**.
- Logic: AND (all must be healthy), OR (at least one), or **X of N** (at least X out of N must be healthy).
- Can monitor up to **256 child health checks**.
- Use case: consider a service healthy only if both the web server AND database are healthy.

**3. CloudWatch Alarm-based Health Check:**
- Health check status is based on a **CloudWatch alarm** state.
- ALARM state = unhealthy. OK state = healthy. INSUFFICIENT_DATA = configurable (healthy or unhealthy).
- Use case: monitor **private resources** (RDS, internal ALB, DynamoDB) that Route 53 health checkers cannot reach directly.
- **Exam pattern:** "Monitor a private resource for Route 53 failover" = CloudWatch alarm -> Route 53 health check.

#### Health Check Architecture

```
Route 53 Health Checkers (15+ locations worldwide)
         |
         | HTTP/HTTPS/TCP
         v
+-------------------+
| Your Endpoint     |  (must be publicly accessible)
| (ALB, EC2, etc.)  |
+-------------------+

For private resources:
+-------------------+     +-----------------+     +------------------+
| Private Resource  | --> | CloudWatch      | --> | Route 53 Health  |
| (RDS, internal)   |     | Alarm           |     | Check            |
+-------------------+     +-----------------+     +------------------+
```

#### Health Check Configuration Details

| Setting | Options | Default |
|---------|---------|---------|
| Request interval | 10 seconds (fast) or 30 seconds (standard) | 30 seconds |
| Failure threshold | 1-10 consecutive failures to mark unhealthy | 3 |
| Health checker regions | Choose specific or use recommended | Recommended (all) |
| String matching | Check for string in first 5,120 bytes | Disabled |
| Latency measurement | Enable to track endpoint latency | Disabled |
| Invert health check | Swap healthy/unhealthy states | No |
| Health check disabled | Always considered healthy | No |

#### Health Check and Routing Policy Integration

| Routing Policy | Health Check Behavior |
|----------------|----------------------|
| Simple | Health checks **not supported** |
| Weighted | Unhealthy records excluded. If all unhealthy, all returned. |
| Latency | Unhealthy records excluded from latency calculation |
| Failover | Primary must have health check. Failover to secondary when primary unhealthy. |
| Geolocation | Unhealthy records excluded. Falls through to broader location or default. |
| Geoproximity | Unhealthy records excluded |
| Multivalue | Unhealthy records excluded. Up to 8 healthy returned. |

### DNS Failover Patterns

#### Active-Passive Failover

```
                    Route 53 (Failover routing)
                   /                            \
         Primary (health check)          Secondary (S3 static site)
         us-east-1 ALB                   "Service temporarily unavailable"
```

- Primary: active resource with mandatory health check.
- Secondary: standby resource (often S3 static website, standby region, or CloudFront serving cached content).
- Failover happens when the health check for the primary fails.

#### Active-Active Failover

```
                    Route 53 (Weighted / Latency / Multivalue)
                   /                    |                    \
         Region A (health check)  Region B (health check)  Region C (health check)
```

- All resources are active and serving traffic simultaneously.
- Unhealthy resources are automatically removed from DNS responses.
- Use Weighted, Latency, Geolocation, or Multivalue routing — all support health checks.

#### Complex / Nested Routing (Record Trees)

Route 53 supports **combining routing policies** by using Alias records that point to other record sets:

```
example.com (Latency routing)
  |
  +-- us-east-1 (Alias to Weighted records)
  |     +-- 70% -> ALB-blue
  |     +-- 30% -> ALB-green
  |
  +-- eu-west-1 (Alias to Failover records)
        +-- Primary: ALB-eu
        +-- Secondary: S3 static site
```

**How it works:**
1. Top-level record uses Latency routing to pick the best region.
2. Each region's record uses Weighted routing for canary deployment or Failover for DR.
3. Route 53 resolves the tree from top to bottom, evaluating health checks at each level.

### Route 53 Resolver

**Route 53 Resolver** provides DNS resolution between on-premises networks and AWS VPCs:

**Inbound Endpoint:**
- On-premises DNS servers forward queries to Route 53 Resolver.
- Allows on-prem to resolve AWS private hosted zone records.
- Creates ENIs in your VPC subnets that accept DNS queries.

**Outbound Endpoint:**
- Route 53 Resolver forwards queries to on-premises DNS servers.
- Allows AWS resources to resolve on-prem domain names.
- Conditional forwarding rules define which domains to forward.

**Resolver Rules:**
- Forward rules: forward queries for specific domains to target DNS servers.
- System rules: Route 53 Resolver's built-in rules (e.g., resolve private hosted zones).
- Rules can be shared across accounts via **AWS RAM**.

```
On-Premises DNS <--> Route 53 Resolver Endpoints <--> VPC DNS (Private Hosted Zones)
                     (Inbound + Outbound ENIs)
```

### Route 53 Traffic Flow

- Visual editor for creating complex routing configurations.
- Uses **Traffic Policies** (versioned, reusable routing configurations).
- **Traffic Policy Records** apply a policy to a specific hosted zone and record name.
- Supports all routing policies plus rules, branches, and endpoints in a visual tree.
- Cost: $50/month per Traffic Policy Record (expensive).
- Use case: complex routing trees (geoproximity with bias, multi-level failover).

### Limits and Quotas

| Resource | Default Limit |
|----------|--------------|
| Hosted zones per account | 500 (can increase) |
| Records per hosted zone | 10,000 (can increase) |
| Routing policies per record | 1 (but can nest via Alias chains) |
| Health checks per account | 200 (can increase) |
| Domains registered per account | 20 (can increase) |
| VPCs associated per PHZ | 100 |
| Resolver endpoints per VPC | 4 (2 inbound + 2 outbound) |
| IPs per Resolver endpoint | 6 |
| Resolver rules per account (per region) | 1,000 |
| Traffic policies per account | 50 |
| TTL minimum | 0 seconds (not recommended) |
| Alias record TTL | Cannot be set (inherited from target) |
| DNS query response size | 512 bytes UDP / 4,096 bytes TCP |

---

## 2. Design Patterns & Best Practices

### When to Use Each Routing Policy

| Scenario | Routing Policy |
|----------|---------------|
| Single resource, no failover | Simple |
| Blue-green deployment, canary release | Weighted |
| Multi-region app, optimize latency | Latency-based |
| Active-passive DR | Failover |
| Legal/regulatory requirement for geographic routing | Geolocation |
| Gradually shift traffic between regions | Geoproximity (Traffic Flow) |
| Return multiple healthy IPs | Multivalue Answer |

### When NOT to Use Route 53 (Anti-Patterns)

- **Real-time load balancing:** DNS has TTL caching. Use ALB/NLB for real-time load balancing. DNS changes take time to propagate.
- **Session affinity/stickiness:** DNS cannot provide sticky sessions. Use ALB sticky sessions.
- **Sub-second failover:** DNS TTL means failover takes at least TTL seconds. For sub-second failover, use Global Accelerator or multi-region ALB behind CloudFront.
- **Private resource health checks without CloudWatch:** Route 53 health checkers are public. Cannot directly check private IPs. Must use CloudWatch alarm-based health checks.

### Architectural Patterns

#### Pattern 1: Multi-Region Active-Active with Latency Routing

```
Users worldwide
      |
Route 53 (Latency-based routing)
      |
+-----+-----+-----+
|           |           |
us-east-1   eu-west-1   ap-southeast-1
ALB+App     ALB+App     ALB+App
DynamoDB    DynamoDB    DynamoDB
Global Table Global Table Global Table
```

- Latency routing directs users to the lowest-latency region.
- Health checks on each region's ALB. Unhealthy regions removed automatically.
- DynamoDB Global Tables for data replication.
- **RPO near zero** (replication lag). **RTO: DNS TTL** (typically 60s).

#### Pattern 2: Active-Passive Failover with S3 Static Site

```
Route 53 (Failover routing)
    |
Primary: us-east-1 ALB (health check)
Secondary: S3 static website ("We'll be back soon")
```

- Primary region serves all traffic.
- On failure, Route 53 returns S3 static site endpoint.
- S3 website hosting must be enabled. Alias record to S3 website endpoint.
- Cost-effective DR for informational/sorry pages.

#### Pattern 3: Blue-Green Deployment with Weighted Routing

```
Route 53 (Weighted routing)
    |
Blue (weight=90):  ALB -> v1.0 instances (current production)
Green (weight=10): ALB -> v2.0 instances (new version)
```

- Start with 100/0 split. Gradually shift: 90/10, 50/50, 0/100.
- Health checks on both. If green is unhealthy, traffic auto-routes to blue.
- After validation, set blue to weight=0, green to weight=100.
- Rollback: flip weights back.

#### Pattern 4: Geolocation for Compliance

```
Route 53 (Geolocation routing)
    |
EU users -> eu-west-1 (GDPR-compliant region)
US users -> us-east-1
Default  -> us-east-1
```

- EU user data stays in EU regions (GDPR, data residency).
- Must have a default record for unmatched locations.
- Combine with geolocation-based bucket policies and KMS keys for end-to-end compliance.

#### Pattern 5: Hybrid DNS with Route 53 Resolver

```
On-Premises DNS                      AWS VPC
+---------------+    Inbound EP     +------------------+
| corp.local    | <-DNS forward-->  | PHZ: aws.corp    |
| resolver      |    Outbound EP    | records           |
+---------------+                   +------------------+
```

- Inbound Endpoint: on-prem queries AWS PHZ records (e.g., `db.aws.corp.local`).
- Outbound Endpoint: AWS queries on-prem DNS (e.g., `ldap.corp.local`).
- Resolver rules shared via RAM across accounts.

#### Pattern 6: Complex Nested Routing for Global SaaS

```
api.saas.com (Geolocation)
    |
    +-- EU -> eu-west-1 (Failover)
    |         +-- Primary: ALB-EU (health check)
    |         +-- Secondary: us-east-1 ALB (fallback)
    |
    +-- Default -> (Latency)
              +-- us-east-1 ALB (health check)
              +-- ap-southeast-1 ALB (health check)
```

- EU traffic is always served from EU (compliance) with failover to US if EU is down.
- All other traffic uses latency-based routing for best performance.
- Health checks at every level ensure unhealthy resources are excluded.

### Well-Architected Framework Alignment

| Pillar | Route 53 Consideration |
|--------|----------------------|
| **Reliability** | 100% SLA. Health checks enable automatic failover. Multi-region active-active with latency routing. Alias records auto-update when AWS resource IPs change. |
| **Security** | DNSSEC for domain validation. IAM for API access. PHZ for internal DNS. Resolver for hybrid DNS isolation. |
| **Cost** | Alias queries to AWS resources are free. Use low TTL only when needed (more queries = more cost). Avoid Traffic Flow if simple routing suffices ($50/month per policy record). |
| **Performance** | Latency-based routing for global apps. Low TTL for faster failover vs high TTL for fewer queries. CloudFront + Route 53 for edge caching + DNS. |
| **Operational Excellence** | CloudWatch alarms for health check status. SNS notifications on failover. DNS query logging. Version-controlled Traffic Flow policies. |

### Integration Points with Other AWS Services

| Service | Integration |
|---------|------------|
| **ELB** | Alias record to ALB/NLB. Health checks monitor ALB. |
| **CloudFront** | Alias record to distribution. Route 53 Alias queries are free. |
| **S3** | Alias record to S3 website endpoint. Used for static site failover. |
| **API Gateway** | Alias record for custom domain. Edge-optimized or Regional. |
| **Global Accelerator** | Alias record. GA provides static IPs with instant failover (faster than DNS TTL). |
| **Elastic Beanstalk** | Alias record to EB environment URL. |
| **CloudWatch** | Alarm-based health checks for private resources. Metrics for health check status. |
| **SNS** | Notifications when health check status changes. |
| **RAM** | Share Resolver rules and PHZ associations across accounts. |
| **AWS Config** | Track Route 53 record changes for compliance. |
| **CloudTrail** | Log all Route 53 API calls. |
| **Certificate Manager** | DNS validation for ACM certificates (CNAME record in Route 53). |

---

## 3. Security & Compliance

### IAM for Route 53

Route 53 supports fine-grained IAM policies:

```json
{
  "Effect": "Allow",
  "Action": [
    "route53:ChangeResourceRecordSets",
    "route53:GetHostedZone",
    "route53:ListResourceRecordSets"
  ],
  "Resource": "arn:aws:route53:::hostedzone/Z1234567890"
}
```

**Key IAM actions:**
- `route53:CreateHostedZone` / `route53:DeleteHostedZone`
- `route53:ChangeResourceRecordSets` — create/update/delete records
- `route53:CreateHealthCheck` / `route53:UpdateHealthCheck`
- `route53:GetHealthCheckStatus` — view health check results
- `route53domains:RegisterDomain` — domain registration (separate namespace)

### DNSSEC

Route 53 supports **DNSSEC signing** for public hosted zones:

- Protects against DNS spoofing and man-in-the-middle attacks.
- Route 53 manages the Zone Signing Key (ZSK).
- You manage the Key Signing Key (KSK) in **KMS** (must be in **us-east-1**, asymmetric, ECC_NIST_P256).
- To enable: create KSK in KMS -> enable DNSSEC signing -> add DS record at registrar.
- Route 53 automatically rotates ZSK. You manage KSK rotation.

**Exam tip:** DNSSEC = protect DNS integrity (not confidentiality). DNS queries are still plaintext. DNSSEC ensures the response hasn't been tampered with.

### DNS Query Logging

- Log all DNS queries received by a public hosted zone.
- Logs sent to **CloudWatch Logs** (must be in **us-east-1** for public hosted zones).
- Log contents: hosted zone ID, query name, query type, response code, protocol, Route 53 edge location.
- **Not available for private hosted zones** (use VPC DNS query logging via Route 53 Resolver query logging instead).

### Route 53 Resolver Query Logging

- Logs DNS queries from resources within a VPC.
- Destinations: CloudWatch Logs, S3, or Kinesis Data Firehose.
- Captures: queries to Route 53 Resolver, including private hosted zones, AWS internal domains, and forwarded queries.
- Can be shared across accounts via RAM.

### SCP Considerations

Restrict Route 53 actions at the organization level:
```json
{
  "Effect": "Deny",
  "Action": [
    "route53:CreateHostedZone",
    "route53:DeleteHostedZone",
    "route53domains:RegisterDomain"
  ],
  "Resource": "*"
}
```

Use case: prevent individual accounts from creating their own hosted zones or registering domains — centralize DNS management in a shared services account.

### Domain Transfer Lock

- Enable **transfer lock** on registered domains to prevent unauthorized transfers.
- Disable lock only when intentionally transferring to another registrar.
- Route 53 automatically enables transfer lock on new registrations.

---

## 4. Cost Optimization

### Pricing Model

| Component | Price |
|-----------|-------|
| Hosted zone | $0.50/month (first 25 zones), $0.10/month (additional) |
| Standard queries | $0.40 per million queries (first 1B), $0.20 per million (additional) |
| Latency-based queries | $0.60 per million |
| Geo DNS queries | $0.70 per million |
| **Alias queries to AWS resources** | **$0.00 (FREE)** |
| Health checks (basic) | $0.50/month per health check (standard interval) |
| Health checks (fast) | $1.00/month per health check (10-second interval) |
| Health checks + string matching | Additional $1.00/month |
| Health checks + HTTPS | Additional $1.00/month |
| Traffic Flow policy record | $50.00/month per policy record |
| Resolver endpoints | $0.125/hour per ENI per direction |
| Resolver queries | $0.40 per million queries (inbound/outbound) |
| Domain registration | Varies by TLD ($12/year for .com) |

### Cost-Saving Strategies

1. **Use Alias records for AWS resources:** Queries are free. A CNAME to an ALB costs $0.40/M queries. An Alias to the same ALB costs $0.00.

2. **Optimize TTL:**
   - Higher TTL = fewer queries = lower cost. But slower failover.
   - For static resources: TTL 86400 (24 hours).
   - For failover records: TTL 60 seconds (balance between cost and failover speed).
   - Standard: TTL 300 (5 minutes) is a good default.

3. **Minimize health checks:**
   - Each health check costs $0.50-$2.00/month.
   - Don't create health checks for non-critical resources.
   - Use calculated health checks to monitor a group with fewer individual checks.
   - Use 30-second interval (standard) unless fast failover is critical.

4. **Avoid Traffic Flow unless needed:**
   - $50/month per policy record is expensive.
   - Simple nested Alias records can achieve most routing trees without Traffic Flow.
   - Only use Traffic Flow for geoproximity with bias or very complex routing trees.

5. **Consolidate hosted zones:**
   - Don't create one hosted zone per microservice. Use one hosted zone for the domain.
   - Subdomains don't require separate hosted zones.

### Cost Traps

- Using CNAME instead of Alias for AWS resources (paying for queries that could be free).
- Low TTL on high-traffic records (millions of extra queries per month).
- Creating Traffic Flow policy records when simple routing policies suffice.
- Running fast (10-second) health checks when 30-second is sufficient.
- Health checks with string matching + HTTPS (cumulative: $0.50 + $1.00 + $1.00 = $2.50/month each).
- Unused hosted zones still cost $0.50/month.

---

## 5. High Availability, Disaster Recovery & Resilience

### Route 53 Inherent HA

- **100% availability SLA** — the only AWS service with this guarantee.
- Global anycast network of DNS servers.
- No single point of failure.
- Automatic scaling to handle query volume.

### DNS-Based DR Patterns

#### Active-Passive (Pilot Light / Warm Standby)

| Component | Primary Region | DR Region |
|-----------|---------------|-----------|
| Route 53 | Failover routing — Primary record | Failover routing — Secondary record |
| Compute | Full capacity ALB + ASG | Minimal capacity (pilot light) or scaled-down (warm standby) |
| Database | RDS primary / Aurora primary | RDS read replica / Aurora replica (promote on failover) |
| Health check | HTTP check on primary ALB | None (secondary) or health check |

**Failover trigger:** Primary health check fails -> Route 53 returns secondary record -> secondary scales up (ASG policies or manual).

**RTO:** DNS TTL + scale-up time (minutes to hours depending on architecture).
**RPO:** Replication lag (seconds for Aurora, minutes for cross-region RDS).

#### Active-Active (Multi-Region)

| Component | Region A | Region B |
|-----------|----------|----------|
| Route 53 | Latency/Weighted routing with health check | Latency/Weighted routing with health check |
| Compute | Full capacity | Full capacity |
| Database | DynamoDB Global Table / Aurora Global | DynamoDB Global Table / Aurora Global |

**RTO:** DNS TTL (typically 60 seconds). No manual intervention.
**RPO:** Replication lag (milliseconds for DynamoDB GT, seconds for Aurora).

#### Multi-Tier Failover

```
Route 53 Failover
    |
Primary: us-east-1 ALB (health check)
    |
Secondary: Route 53 Failover (nested)
    |
    Primary: eu-west-1 ALB (health check)
    Secondary: S3 static "sorry" page
```

- Three tiers: primary region -> DR region -> static sorry page.
- Requires nested Alias records with failover routing at each level.

### DNS Failover Timing

**Total failover time = Health check detection time + DNS propagation time**

| Factor | Value |
|--------|-------|
| Health check interval | 10 or 30 seconds |
| Failure threshold | Default 3 consecutive failures |
| Detection time (standard, 3 failures) | 3 x 30s = 90 seconds |
| Detection time (fast, 3 failures) | 3 x 10s = 30 seconds |
| DNS TTL | Your configured TTL (e.g., 60 seconds) |
| **Total failover (standard)** | **~150 seconds (90s detection + 60s TTL)** |
| **Total failover (fast)** | **~90 seconds (30s detection + 60s TTL)** |
| Client DNS cache | Variable — some clients ignore TTL |

**To minimize failover time:**
1. Use 10-second health check interval ($1.00/month instead of $0.50).
2. Set failure threshold to 1 or 2 (risk: more false positives).
3. Set low TTL on failover records (60 seconds).
4. For sub-minute failover: use Global Accelerator instead of DNS failover.

### Route 53 + Global Accelerator Comparison for Failover

| Feature | Route 53 Failover | Global Accelerator |
|---------|-------------------|-------------------|
| Failover mechanism | DNS-based (TTL dependent) | Network-layer (instant rerouting) |
| Failover time | TTL + detection (60-150s) | Seconds (health check based) |
| Client caching | DNS TTL may be ignored by clients | Static anycast IPs, no DNS caching issue |
| Cost | Low (queries + health checks) | Higher ($0.025/hr + $0.01-0.035/GB) |
| Protocol | Any (DNS routing) | TCP and UDP |
| Use case | Cost-effective DR with acceptable RTO | Mission-critical, sub-minute failover |

**Exam keyword:** "minimize failover time" or "instant failover" or "DNS caching problem" = Global Accelerator.
"Cost-effective failover" or "DR with acceptable RTO" = Route 53 Failover.

---

## 6. Exam-Focused Section

### Straightforward Questions

**Q1: A company has a web application deployed in us-east-1 and eu-west-1. They want users to be routed to the region with the best performance. Which routing policy should they use?**

**A:** Latency-based routing. It routes users to the AWS region with the lowest network latency, not just the geographically closest.

---

**Q2: Can you create a CNAME record for the zone apex (example.com)?**

**A:** No. CNAME records cannot be used at the zone apex per DNS RFC. Use a Route 53 **Alias record** instead.

---

**Q3: A company uses Route 53 failover routing with a primary ALB in us-east-1 and a secondary S3 static site. The primary fails but traffic is not failing over. What is the most likely cause?**

**A:** The primary record does not have a **health check** configured. Failover routing requires a health check on the primary record to detect failure.

---

**Q4: Which Route 53 routing policy allows you to return multiple IP addresses, each with its own health check?**

**A:** Multivalue Answer routing. It returns up to 8 healthy records, each with its own health check.

---

**Q5: How can Route 53 perform health checks on a private resource like an RDS instance in a private subnet?**

**A:** Create a **CloudWatch alarm** monitoring the private resource (e.g., RDS CPU, connection count). Create a Route 53 health check that monitors the CloudWatch alarm state. Health checkers don't access the resource directly.

---

**Q6: A company wants to route all EU users to their eu-west-1 deployment for GDPR compliance, regardless of latency. Which routing policy?**

**A:** Geolocation routing. It routes based on the user's geographic location (continent/country), not latency. Set records for Europe -> eu-west-1, with a default record for all other users.

---

**Q7: What is the cost of Route 53 Alias queries to an Elastic Load Balancer?**

**A:** $0.00 — free. Alias queries to AWS resources (ELB, CloudFront, S3 website, etc.) are not charged.

---

### Tricky / Scenario-Based Questions

**Q1: A company runs an active-passive architecture with Route 53 failover routing. Primary is in us-east-1, secondary is in eu-west-1. When the primary fails, users report they still reach the old (failed) primary for several minutes. The health check detected the failure within 30 seconds. What is the cause and solution?**

**A:** The issue is **DNS TTL caching**. Even after Route 53 updates the DNS response to the secondary, clients and intermediate resolvers cache the old DNS response for the duration of the TTL.

**Solution:** Lower the TTL on the failover record to 60 seconds. However, some clients and resolvers ignore TTL. For guaranteed fast failover, use **AWS Global Accelerator** which uses static anycast IPs (no DNS caching issue).

**Why wrong answers are wrong:**
- "Increase the health check frequency" — the health check already detected in 30s. The problem is DNS caching, not detection.
- "Use CNAME instead of Alias" — CNAME doesn't solve DNS caching and adds cost.
- "Add a secondary health check" — the secondary health check status doesn't affect DNS caching.

**Exam keywords:** "still reaching old primary after failover" + "minutes delay" = DNS TTL caching problem.

---

**Q2: A company wants to gradually migrate traffic from an on-premises data center (ip: 203.0.113.10) to AWS (ALB in us-east-1). They want to start with 10% to AWS and 90% to on-prem, then gradually increase. Which routing policy?**

**A:** **Weighted routing.** Create two records:
- `app.example.com` A record -> 203.0.113.10, weight=90
- `app.example.com` Alias -> ALB, weight=10

Gradually adjust weights: 90/10 -> 70/30 -> 50/50 -> 0/100.

**Why wrong answers are wrong:**
- "Failover routing" — failover is active-passive, not gradual traffic splitting. It's all-or-nothing.
- "Latency-based routing" — routes based on latency, not percentage. You can't control the split.
- "Geoproximity with bias" — could work theoretically but is overly complex, requires Traffic Flow ($50/month), and bias values don't map to percentages cleanly.

**Exam keywords:** "gradually migrate" + "percentage" + "10% to 90%" = Weighted routing.

---

**Q3: A company has Route 53 Geolocation routing configured for US (-> us-east-1), Europe (-> eu-west-1), and Asia (-> ap-southeast-1). A user in South America reports they cannot access the application (NXDOMAIN). What is the issue?**

**A:** There is **no default record** configured. Geolocation routing only returns records that match the user's location. South America doesn't match US, Europe, or Asia. Without a default record, Route 53 returns no answer (NXDOMAIN).

**Solution:** Add a **default** geolocation record pointing to any region (e.g., us-east-1).

**Why wrong answers are wrong:**
- "Geolocation doesn't support South America" — it does, as a continent. The issue is the missing default.
- "Add a South America geolocation record" — works but doesn't solve for Antarctica, Africa, or any other unmatched location. Default is the best practice.
- "Switch to latency-based routing" — changes the routing behavior. The question implies geolocation is the requirement (possibly for compliance).

**Exam keyword:** "cannot access" + "geolocation" + "some regions" = missing default record.

---

**Q4: A company uses Route 53 with an Alias record pointing to an Application Load Balancer. The ALB has its own health check on the target group. The company also creates a Route 53 health check on the ALB's DNS name. Is the Route 53 health check necessary?**

**A:** For a **simple Alias record** (no routing policy), Route 53 automatically considers the ALB healthy if it has at least one healthy target in at least one AZ. This is called **"Evaluate Target Health" (ETH)**, which is automatically enabled for Alias records to ELB. The separate Route 53 health check is **redundant**.

However, if using a routing policy (weighted, failover, etc.), you might still want a Route 53 health check for **additional criteria** (e.g., check a specific URL path, string matching) beyond basic ALB target health.

**Why wrong answers are wrong:**
- "Route 53 health checks are always required for Alias records" — false. ETH is automatic for ELB Alias records.
- "ALB health checks and Route 53 health checks are the same thing" — false. ALB health checks monitor targets. Route 53 health checks monitor the endpoint from external health checkers.

**Exam concept:** "Evaluate Target Health" for Alias records to ELB = automatic, no separate health check needed.

---

**Q5: A company needs to route DNS for `internal.corp.com` within their VPC. They also need on-premises servers (connected via Direct Connect) to resolve `internal.corp.com`. How should they set this up?**

**A:**
1. Create a **Private Hosted Zone** for `internal.corp.com`, associated with the VPC.
2. Create a **Route 53 Resolver Inbound Endpoint** in the VPC.
3. Configure on-premises DNS to **forward** queries for `internal.corp.com` to the Resolver Inbound Endpoint IP addresses.

**Why wrong answers are wrong:**
- "Use a Public Hosted Zone" — exposes internal DNS records to the internet. Security violation.
- "Use VPC DHCP options to point to on-prem DNS" — this would route ALL DNS from VPC to on-prem, breaking AWS service resolution.
- "Create the records in on-prem DNS and use Resolver Outbound Endpoint" — this routes VPC queries to on-prem. The question asks on-prem to resolve AWS records, so you need Inbound.

**Exam keywords:** "on-premises resolve AWS private DNS" = Resolver Inbound Endpoint. "AWS resolve on-premises DNS" = Resolver Outbound Endpoint.

---

**Q6: A company is deploying a new version of their application. They want to send 5% of traffic to the new version for testing. If the new version becomes unhealthy, all traffic should go to the old version automatically. Which configuration achieves this?**

**A:** **Weighted routing** with health checks:
- Record 1: old version, weight=95, health check enabled
- Record 2: new version, weight=5, health check enabled

If the new version's health check fails, Route 53 excludes it and sends all traffic to the old version. This achieves canary deployment with automatic rollback.

**Why wrong answers are wrong:**
- "Failover routing with primary=old, secondary=new" — this sends all traffic to old, with new as backup. The opposite of what's wanted.
- "Multivalue answer" — returns both IPs but can't control percentage split. ~50/50 distribution.
- "Simple routing with multiple values" — no health checks, no percentage control.

**Exam keywords:** "percentage of traffic" + "automatic rollback if unhealthy" = Weighted + health checks.

---

**Q7: A company wants to expand the geographic area served by their eu-west-1 deployment to attract more traffic from North Africa and Middle East, which are currently routed to ap-south-1. The application is latency-sensitive. Which routing approach?**

**A:** **Geoproximity routing** with a **positive bias** on eu-west-1. Increasing the bias value expands the geographic area from which eu-west-1 attracts traffic. This shifts the boundary so North Africa and Middle East are captured by eu-west-1.

**Why wrong answers are wrong:**
- "Geolocation routing" — maps specific countries/continents to specific regions. You'd have to list every country individually. No concept of "expanding" a region's area.
- "Latency-based routing" — routes by actual latency. You can't control which region serves which area. If ap-south-1 has lower latency for Middle East, it will always win.
- "Weighted routing" — splits traffic by percentage globally, not geographically.

**Exam keywords:** "expand geographic area" + "shift traffic" + "bias" = Geoproximity routing.

---

### Common Exam Traps & Pitfalls

1. **Geolocation vs Latency-Based:**
   - Geolocation = WHERE the user is (compliance, localization). Fixed geographic mapping.
   - Latency = HOW FAST the connection is (performance). Dynamic, can route anywhere.
   - "GDPR compliance" or "data residency" = Geolocation.
   - "Best performance" or "lowest latency" = Latency-based.

2. **Alias vs CNAME:**
   - Zone apex (naked domain) = **must** be Alias. CNAME is invalid at zone apex.
   - Alias to AWS resource = free queries. CNAME = paid queries.
   - Alias TTL is auto-managed. CNAME TTL is user-configurable.

3. **Simple routing has NO health checks:**
   - If the question requires health checks, Simple routing is wrong.
   - Multivalue Answer is the "Simple routing with health checks" equivalent.

4. **Gateway Endpoint vs Failover secondary:**
   - S3 website endpoint as failover secondary needs **Alias record to S3 website endpoint**.
   - The S3 bucket name must match the record name (e.g., `app.example.com` bucket).

5. **Private resource health checks:**
   - Route 53 health checkers cannot access private IPs.
   - "Private RDS" + "health check" = CloudWatch alarm -> Route 53 health check.
   - This is tested repeatedly.

6. **Evaluate Target Health (ETH) for Alias records:**
   - When Alias points to ELB, ETH is **automatic**. No separate health check needed.
   - When Alias points to another Route 53 record, ETH evaluates the target record's health check.
   - When Alias points to S3 or CloudFront, there is no health check integration.

7. **Resolver Inbound vs Outbound:**
   - **Inbound**: on-prem -> AWS. On-prem DNS forwards to Resolver. "On-prem needs to resolve AWS private DNS."
   - **Outbound**: AWS -> on-prem. Resolver forwards to on-prem DNS. "AWS needs to resolve on-prem domains."
   - This is confusing because the names describe the direction **from Route 53 Resolver's perspective**.

8. **Health check + Weighted routing with all unhealthy:**
   - If ALL weighted records are unhealthy, Route 53 returns **all records** (treats all as healthy). This is a safety mechanism to avoid returning no answer.

9. **DNSSEC KSK must be in us-east-1:**
   - The KMS key for KSK must be in us-east-1, must be asymmetric, must be ECC_NIST_P256.
   - Common distractor: "create key in the same region as the hosted zone" — Route 53 is global, KSK must be in us-east-1.

10. **Traffic Flow cost:**
    - $50/month per Traffic Policy Record. Very expensive.
    - Only use for Geoproximity or very complex routing trees.
    - If the question mentions cost optimization, avoid Traffic Flow.

---

## 7. Cheat Sheet

### Must-Know Facts

- Route 53 has a **100% SLA** — only AWS service with this.
- **Alias record**: free queries to AWS resources, works at zone apex, TTL auto-managed.
- **CNAME**: cannot be at zone apex, costs money, TTL you set.
- **Gateway S3/DynamoDB only** (VPC Endpoint). **Alias targets**: ELB, CloudFront, S3 website, API GW, EB, GA, another R53 record.
- **Simple routing**: no health checks. **Multivalue**: up to 8 healthy records with health checks.
- **Failover**: active-passive. Primary MUST have health check.
- **Weighted**: percentage split, supports health checks, used for blue-green/canary.
- **Latency**: lowest latency region. Not geographic, not distance.
- **Geolocation**: compliance/localization. Must have default record.
- **Geoproximity**: bias to shift traffic. Requires Traffic Flow ($50/month).
- **Private resource health check**: CloudWatch Alarm -> Route 53 Health Check.
- **Evaluate Target Health**: automatic for Alias to ELB. No separate health check needed.
- **Resolver Inbound**: on-prem resolves AWS. **Outbound**: AWS resolves on-prem.
- **Failover time**: detection (30-90s) + TTL (60s) = 90-150 seconds. For faster, use Global Accelerator.
- **All weighted unhealthy**: Route 53 returns all records (safety fallback).
- **DNSSEC KSK**: KMS key in us-east-1, asymmetric, ECC_NIST_P256.

### Decision Flowchart

```
What does the question ask about DNS routing?
|
+-- "Zone apex" or "naked domain" (example.com)
|   --> Alias record (never CNAME)
|
+-- "Best performance" or "lowest latency"
|   --> Latency-based routing
|
+-- "Compliance" or "data residency" or "users in specific country"
|   --> Geolocation routing (+ default record!)
|
+-- "Gradually migrate" or "10% traffic to new version"
|   --> Weighted routing
|
+-- "Active-passive" or "DR failover" or "standby region"
|   --> Failover routing (primary must have health check)
|
+-- "Expand region's serving area" or "shift traffic with bias"
|   --> Geoproximity (requires Traffic Flow)
|
+-- "Return multiple healthy IPs" or "client-side failover"
|   --> Multivalue Answer routing
|
+-- "Health check private resource" (RDS, internal ALB)
|   --> CloudWatch Alarm -> Route 53 Health Check
|
+-- "On-prem resolve AWS private DNS"
|   --> Route 53 Resolver Inbound Endpoint
|
+-- "AWS resolve on-prem DNS"
|   --> Route 53 Resolver Outbound Endpoint
|
+-- "Instant failover" or "DNS caching problem"
|   --> Global Accelerator (not Route 53 alone)
|
+-- "Reduce DNS query cost"
|   --> Use Alias records (free for AWS targets)
```

### Routing Policy Quick Reference

| If question says... | Think... |
|---------------------|----------|
| "best performance globally" | Latency-based |
| "GDPR / EU users must stay in EU" | Geolocation |
| "canary deployment / 5% new version" | Weighted |
| "DR / standby / active-passive" | Failover |
| "expand region catchment / bias" | Geoproximity |
| "multiple IPs with health checks" | Multivalue Answer |
| "single resource, no failover" | Simple |
| "on-prem resolve AWS DNS" | Resolver Inbound |
| "AWS resolve on-prem DNS" | Resolver Outbound |
| "private health check / RDS / internal" | CloudWatch Alarm |
| "faster than DNS failover" | Global Accelerator |
| "zone apex / naked domain" | Alias record |
| "free DNS queries" | Alias record |

---

*Generated for AWS SAP-C02 exam preparation. Last updated: 2026-04-26.*
