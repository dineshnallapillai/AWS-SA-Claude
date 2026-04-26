# Global Accelerator & CloudFront — SAP-C02 Study Notes

---

## 1. Core Concepts & Theory

### Amazon CloudFront

CloudFront is a **Content Delivery Network (CDN)** that caches content at **450+ edge locations** and **13 regional edge caches** worldwide. It accelerates delivery of static content, dynamic content, APIs, and video streaming by serving from the location closest to the user.

**Key architecture:**
```
User --> Edge Location (cache hit? return) --> Regional Edge Cache --> Origin (S3, ALB, EC2, custom)
```

#### CloudFront Components

| Component | Description |
|-----------|-------------|
| **Distribution** | The top-level CloudFront configuration. Types: Web or RTMP (deprecated). |
| **Origin** | Where CloudFront fetches content: S3 bucket, ALB, EC2, API Gateway, MediaStore, custom HTTP server, Lambda@Edge origin. |
| **Edge Location** | The CDN point of presence. Caches content. 450+ worldwide. |
| **Regional Edge Cache** | Larger caches between edge locations and origin. Holds less-popular content longer. |
| **Behavior** | Path-pattern rules that determine how CloudFront handles requests (e.g., `/api/*` vs `/images/*`). Each behavior maps to an origin. |
| **Cache Policy** | Controls what is included in the cache key (headers, query strings, cookies). |
| **Origin Request Policy** | Controls what is forwarded to the origin (independent of cache key). |
| **Response Headers Policy** | Adds/modifies HTTP headers in the response (CORS, security headers). |
| **Origin Access Control (OAC)** | Restricts S3 bucket access to CloudFront only. Replaces legacy OAI. |
| **Function associations** | Lambda@Edge or CloudFront Functions at viewer/origin request/response events. |

#### Origin Types

| Origin | Details |
|--------|---------|
| **S3 bucket** | Static content. Use OAC to restrict direct S3 access. Can serve S3 website endpoints too. |
| **ALB** | Dynamic content. Must be public-facing (CloudFront connects over internet). Security group must allow CloudFront IPs. |
| **EC2** | Must have public IP. Security group must allow CloudFront IPs. |
| **API Gateway** | Edge-optimized or Regional API. Use for API caching and DDoS protection. |
| **Custom origin** | Any HTTP/HTTPS endpoint (on-premises server, other cloud). |
| **MediaStore / MediaPackage** | Video streaming origins. |
| **Origin Group** | Primary + secondary origin for origin failover (HA). |

#### Cache Behaviors

Each distribution has a **default behavior** (`*`) and can have up to 25 additional behaviors with path patterns:

```
Path: /api/*     --> Origin: ALB (no cache, forward all headers)
Path: /images/*  --> Origin: S3 bucket (cache 1 day, no cookies)
Path: /*         --> Default: S3 bucket (cache 1 hour)
```

**Cache key** determines what makes a cached object unique. Includes: URL path + optionally: query strings, headers, cookies (configured via Cache Policy).

**Important:** Forwarding headers/cookies/query strings to origin AND including them in the cache key are **separate decisions** (Cache Policy vs Origin Request Policy). This is a relatively new separation that the exam tests.

#### CloudFront Functions vs Lambda@Edge

| Feature | CloudFront Functions | Lambda@Edge |
|---------|---------------------|-------------|
| Runtime | JavaScript only | Node.js, Python |
| Execution location | Edge location (450+) | Regional edge cache (13) |
| Triggers | Viewer request, Viewer response ONLY | Viewer request/response, Origin request/response |
| Execution time | < 1 ms | Up to 5s (viewer) / 30s (origin) |
| Memory | 2 MB | 128-3,008 MB (viewer) / 128-10,240 MB (origin) |
| Network access | No | Yes |
| File system access | No | Yes (via /tmp) |
| Request body access | No | Yes (origin triggers only) |
| Pricing | ~$0.10 per million invocations | ~$0.60 per million + duration |
| Use cases | URL rewrites, header manipulation, simple redirects, cache key normalization | Auth with external service, dynamic content generation, origin selection, image transformation |

**Exam rule:** If the question says "simple header/URL manipulation" or "high volume, low latency" = CloudFront Functions. If "network call," "auth check against external API," or "origin request manipulation" = Lambda@Edge.

#### Origin Access Control (OAC) vs Origin Access Identity (OAI)

| Feature | OAC (new, recommended) | OAI (legacy) |
|---------|----------------------|--------------|
| S3 SSE-KMS support | Yes | No |
| S3 in all regions | Yes | Yes |
| POST/PUT/DELETE to S3 | Yes | Limited |
| Dynamic origins | Yes | No |
| Signing protocol | SigV4 | Custom |

**Exam tip:** OAC is the current best practice. If the question mentions SSE-KMS + CloudFront + S3, the answer involves OAC (not OAI).

#### Signed URLs and Signed Cookies

| Feature | Signed URL | Signed Cookie |
|---------|-----------|---------------|
| Access scope | Single file/object | Multiple files (entire path or site) |
| Use case | Individual file download | HLS/DASH video streams (multiple segment files), entire premium section |
| URL change | URL is different for each user/expiry | URL stays the same, cookie differs |
| Client support | Works everywhere (even no cookie support) | Requires cookie support |

**Trusted signers/key groups:** CloudFront uses RSA key pairs for signing. You manage public keys in CloudFront (key group) and use the private key in your application to generate signatures.

### AWS Global Accelerator

Global Accelerator is a **networking service** that provides **two static anycast IPv4 addresses** that serve as a fixed entry point to your application. Traffic enters the AWS global network at the nearest edge location and is routed over the AWS backbone to your endpoints.

**Key distinction from CloudFront:** Global Accelerator does **NOT cache content**. It is a network-layer (Layer 4) proxy that improves performance by using the AWS backbone instead of the public internet.

```
User --> Anycast IP --> Nearest Edge Location --> AWS Backbone --> Endpoint (ALB, NLB, EC2, EIP)
```

#### Global Accelerator Components

| Component | Description |
|-----------|-------------|
| **Accelerator** | Top-level resource. Gets 2 static anycast IPv4 addresses. |
| **Listener** | Processes inbound connections. Protocol: TCP, UDP, or both. Port range(s). |
| **Endpoint Group** | One per AWS region. Contains endpoints. Has traffic dial (0-100%). Has health check configuration. |
| **Endpoint** | ALB, NLB, EC2 instance, or Elastic IP. Each has a weight (0-255). |
| **Static IP** | Two anycast IPs that never change. Can be brought-your-own (BYOIP). |

#### Global Accelerator Features

**Client Affinity:**
- None (default): requests distributed across endpoints.
- Source IP: all requests from the same client IP go to the same endpoint.
- Useful for stateful applications (but prefer ALB sticky sessions when possible).

**Traffic Dial:**
- Per endpoint group (per region). Range: 0-100%.
- Controls what percentage of traffic goes to that region.
- Used for blue-green deployment, gradual migration, or dialing down a region for maintenance.

**Endpoint Weights:**
- Per endpoint within an endpoint group. Range: 0-255.
- Controls traffic distribution among endpoints in the same region.

**Health Checks:**
- Built-in health checks for endpoints.
- For ALB/NLB: uses the load balancer's health checks.
- For EC2/EIP: configurable health check (TCP, HTTP, HTTPS) with interval, threshold, path.
- Unhealthy endpoints automatically removed. Traffic rerouted to other healthy endpoints.
- If all endpoints in a region are unhealthy, traffic reroutes to the next closest healthy region.

**Instant Failover:**
- When an endpoint or region becomes unhealthy, traffic reroutes in **seconds** (not minutes like DNS).
- No DNS propagation delay. The anycast IPs don't change — the routing behind them changes.
- This is the #1 differentiator vs Route 53 failover.

#### Standard Accelerator vs Custom Routing Accelerator

| Feature | Standard | Custom Routing |
|---------|----------|----------------|
| Endpoints | ALB, NLB, EC2, EIP | VPC subnets (maps to specific EC2 instances) |
| Traffic routing | GA decides based on health, proximity | Application controls mapping of users to specific instances |
| Use case | General web/API applications | Gaming (map players to specific game servers), VoIP, real-time |
| Health checks | Yes | Application manages |
| Protocols | TCP, UDP | TCP, UDP |

### CloudFront vs Global Accelerator — The Core Exam Comparison

| Feature | CloudFront | Global Accelerator |
|---------|------------|-------------------|
| **Layer** | Layer 7 (HTTP/HTTPS/WebSocket) | Layer 4 (TCP/UDP) |
| **Caching** | Yes (primary purpose) | No caching |
| **Content type** | HTTP/HTTPS content (static, dynamic, streaming) | Any TCP/UDP traffic (HTTP, non-HTTP, gaming, IoT, VoIP) |
| **IP addresses** | Dynamic (DNS-based, IPs change) | 2 static anycast IPs (never change) |
| **DDoS protection** | AWS Shield Standard (free) | AWS Shield Standard (free) |
| **Edge locations** | 450+ | 100+ (same AWS edge network) |
| **Failover** | Origin failover with Origin Groups | Instant endpoint/region failover |
| **SSL/TLS termination** | At edge (viewer protocol) + origin protocol | Pass-through or terminate at endpoint |
| **WebSocket** | Supported | Supported |
| **Non-HTTP** | No (HTTP/HTTPS only) | Yes (TCP/UDP — gaming, IoT, VoIP) |
| **Static IP** | No (use Global Accelerator if needed) | Yes (2 static anycast IPs) |
| **WAF integration** | Yes (AWS WAF at edge) | No |
| **Lambda@Edge** | Yes | No |
| **Client IP preservation** | X-Forwarded-For header | Client IP preservation (configurable) |
| **Pricing model** | Data transfer + requests | Fixed hourly + data transfer premium |

---

## 2. Design Patterns & Best Practices

### When to Use CloudFront

| Scenario | Why CloudFront |
|----------|---------------|
| Static website hosting (S3) | Cache at edge, low latency, OAC for security |
| Dynamic API acceleration | Edge presence reduces first-byte latency, TLS termination at edge |
| Video streaming (HLS/DASH) | Segment caching, signed URLs for DRM |
| DDoS protection for HTTP apps | Shield Standard free, WAF at edge |
| Global web application | Cache static assets, pass dynamic requests to origin |
| API Gateway caching | CloudFront in front of API GW for additional caching and WAF |
| Field-level encryption | Encrypt sensitive form fields at the edge |

### When to Use Global Accelerator

| Scenario | Why Global Accelerator |
|----------|----------------------|
| Non-HTTP traffic (gaming, IoT, VoIP) | Layer 4 — supports TCP/UDP natively |
| Static IP requirement (IP whitelisting) | 2 fixed anycast IPs |
| Instant failover (not DNS-dependent) | Failover in seconds, no TTL delay |
| Multi-region active-active HTTP apps | Automatic health-based routing + instant failover |
| IP-based client routing / firewall rules | Partners whitelist your static IPs |
| Blue-green deployment with traffic dial | Regional traffic dial (0-100%) |
| DNS caching issues | Anycast IPs solve client DNS caching problems |

### When NOT to Use (Anti-Patterns)

**CloudFront anti-patterns:**
- Non-HTTP protocols (gaming UDP, MQTT for IoT) — use Global Accelerator.
- Need for static IP addresses — CloudFront IPs are dynamic.
- Latency-insensitive, infrequent access — caching benefit is minimal, adds cost.
- Internal/private-only content — CloudFront is public-facing (use VPC endpoints for private access).

**Global Accelerator anti-patterns:**
- Content that benefits from caching — GA doesn't cache. Use CloudFront.
- WAF protection needed — GA doesn't integrate with WAF. Put CloudFront in front.
- Cost-sensitive applications — GA has a premium for data transfer.
- Simple static website — overkill and expensive. Use CloudFront + S3.

### Architectural Patterns

#### Pattern 1: CloudFront + S3 Static Website

```
Users --> CloudFront (edge cache) --> S3 Bucket (origin)
                                       |
                                  OAC restricts direct access
```

- S3 bucket is private. OAC allows only CloudFront to access.
- Cache static assets (HTML, CSS, JS, images) at edge.
- HTTPS termination at CloudFront (ACM certificate).
- Custom error pages (e.g., 404 -> /index.html for SPA routing).

#### Pattern 2: CloudFront + ALB + Origin Failover

```
Users --> CloudFront --> Origin Group
                           |
                    Primary: ALB us-east-1
                    Secondary: ALB eu-west-1 (failover)
```

- Origin Group: primary + secondary origin.
- If primary returns 5xx or times out, CloudFront automatically tries secondary.
- Configurable failover criteria: specific HTTP status codes (500, 502, 503, 504).
- Used for multi-region HA without Route 53 failover complexity.

#### Pattern 3: Global Accelerator for Multi-Region Active-Active

```
Users --> Static Anycast IP --> AWS Edge --> GA
                                              |
                               +--- Endpoint Group: us-east-1 (ALB)
                               +--- Endpoint Group: eu-west-1 (ALB)
                               +--- Endpoint Group: ap-southeast-1 (ALB)
```

- Health checks per endpoint group.
- Instant failover if a region goes down (seconds, not DNS TTL).
- Traffic dial controls regional traffic distribution.
- Client affinity (source IP) for stateful apps.

#### Pattern 4: CloudFront + Global Accelerator Together

```
Users --> CloudFront (edge cache, WAF) --> Global Accelerator (static IP, instant failover)
                                              --> ALB us-east-1
                                              --> ALB eu-west-1
```

- CloudFront provides caching + WAF + SSL offload.
- Global Accelerator provides static IPs + instant failover.
- Use when you need ALL of: caching, WAF, static IPs, and instant failover.
- GA is the custom origin for CloudFront.

#### Pattern 5: CloudFront for API Gateway Acceleration

```
Users --> CloudFront --> API Gateway (Regional) --> Lambda/Backend
```

- API Gateway "Edge-optimized" already uses CloudFront internally (but you can't customize it).
- Using CloudFront + Regional API Gateway gives you full control: custom cache behaviors, WAF rules, custom domain management.
- Cache API responses at edge (with appropriate cache policy).
- **Exam tip:** "Custom WAF rules at the edge for API Gateway" = CloudFront + Regional API GW (not Edge-optimized API GW).

#### Pattern 6: Global Accelerator for Gaming / Real-Time

```
Game Clients --> Anycast IP --> AWS Edge --> GA (UDP listener)
                                              --> NLB us-east-1 --> Game Servers
                                              --> NLB eu-west-1 --> Game Servers
```

- UDP support for real-time gaming protocols.
- Static IPs for client configuration (no DNS lookup needed).
- Instant failover between regions.
- Custom Routing Accelerator for mapping specific players to specific game server instances.

#### Pattern 7: Migration — On-Prem to AWS with Global Accelerator

```
Clients (IP whitelisted 1.2.3.4, 5.6.7.8)
    |
Global Accelerator (static IPs: 1.2.3.4, 5.6.7.8 via BYOIP)
    |
Phase 1: Traffic dial 100% --> On-prem NLB/EC2
Phase 2: Traffic dial 50/50 --> On-prem + AWS ALB
Phase 3: Traffic dial 100% --> AWS ALB
```

- BYOIP: bring your existing IPs to Global Accelerator.
- No client-side changes (same IPs).
- Gradual migration via traffic dial.

### Well-Architected Framework Alignment

| Pillar | CloudFront | Global Accelerator |
|--------|------------|-------------------|
| **Reliability** | Origin Groups for failover. Multi-origin architectures. 450+ edge locations. | Instant failover. Multi-region endpoint groups. Health checks. |
| **Security** | WAF at edge, Shield Standard/Advanced, OAC for S3, field-level encryption, signed URLs/cookies, geo-restriction. | Shield Standard/Advanced, security groups on endpoints, DDoS protection at edge. |
| **Cost** | Free tier (1TB/month, 10M requests). Reserved capacity pricing for high volume. Minimize origin fetches via caching. | No free tier. $0.025/hr per accelerator + data transfer premium. Use only where value justifies cost. |
| **Performance** | Edge caching reduces latency for cacheable content. TLS termination at edge reduces connection time. | AWS backbone routing reduces latency for all traffic. Eliminates public internet hops. |
| **Operational Excellence** | Real-time logs (Kinesis), standard logs (S3), CloudWatch metrics, invalidation API. | Flow logs (S3), CloudWatch metrics, endpoint health monitoring. |

---

## 3. Security & Compliance

### CloudFront Security

#### OAC (Origin Access Control)

Restricts S3 origin access to CloudFront only:

1. Create OAC in CloudFront.
2. Associate OAC with the distribution's S3 origin.
3. Update S3 bucket policy:
```json
{
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "cloudfront.amazonaws.com"},
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-bucket/*",
    "Condition": {
      "StringEquals": {
        "AWS:SourceArn": "arn:aws:cloudfront::123456789012:distribution/E1234567890"
      }
    }
  }]
}
```

#### AWS WAF Integration (CloudFront Only)

- Attach a WAF Web ACL to a CloudFront distribution.
- WAF evaluates requests at the edge before they reach origin.
- Rules: IP-based blocking, geo-blocking, rate limiting, SQL injection, XSS protection, bot management.
- WAF for CloudFront must be created in **us-east-1** (global scope).
- **Global Accelerator does NOT support WAF.**

#### Geo-Restriction (CloudFront)

- **Allowlist:** only specified countries can access content.
- **Blocklist:** specified countries are blocked.
- Returns 403 Forbidden for restricted locations.
- Uses GeoIP database (country-level).
- For more granular control (city, state): use Lambda@Edge with a third-party GeoIP database.

#### Field-Level Encryption (CloudFront)

- Encrypt specific POST form fields at the edge using a public key.
- Data remains encrypted through to the origin application.
- Only the application with the private key can decrypt.
- Protects sensitive data (credit card, SSN) even if HTTPS is terminated at intermediate points.

#### SSL/TLS

**CloudFront:**
- Viewer Protocol: HTTP, HTTPS, or redirect HTTP to HTTPS.
- Origin Protocol: HTTP only, HTTPS only, or match viewer.
- Custom SSL certificate via **ACM** (must be in **us-east-1**).
- SNI (Server Name Indication): free, default. Dedicated IP: $600/month (for old clients that don't support SNI).
- TLS versions: TLSv1.0 to TLSv1.3. Configurable security policy.

**Global Accelerator:**
- TLS pass-through to the endpoint (GA does not terminate TLS by default).
- For HTTP endpoints (ALB), TLS terminates at the ALB.
- Certificate management is at the endpoint level, not GA.

#### AWS Shield Integration

Both CloudFront and Global Accelerator include **Shield Standard** (free DDoS protection).

**Shield Advanced** (paid, $3,000/month):
- Available for both CloudFront and Global Accelerator.
- Cost protection: refund for scaling charges during DDoS.
- DRT (DDoS Response Team) access.
- Advanced metrics and visibility.
- Automatic application-layer DDoS mitigation (for CloudFront + ALB).

### Logging

| Feature | CloudFront | Global Accelerator |
|---------|------------|-------------------|
| Access logs | Standard logs to S3 | Flow logs to S3 |
| Real-time logs | Kinesis Data Streams | Not available |
| CloudTrail | API calls logged | API calls logged |
| CloudWatch | Metrics: requests, bytes, errors, cache hit ratio | Metrics: processed bytes, healthy endpoints |

**CloudFront real-time logs:** Sent to Kinesis Data Streams. Can then go to Kinesis Data Firehose -> S3/Redshift/OpenSearch. Use for real-time analytics and monitoring.

---

## 4. Cost Optimization

### CloudFront Pricing

| Component | Price (varies by region) |
|-----------|------------------------|
| Data transfer out to internet | $0.085 - $0.170 / GB (tiered) |
| Data transfer to origin | $0.02 / GB (all regions) |
| HTTP requests | $0.0075 - $0.016 per 10K |
| HTTPS requests | $0.01 - $0.022 per 10K |
| Invalidation requests | 1,000/month free, then $0.005 each |
| Lambda@Edge | $0.60 per million + $0.00005001 per GB-second |
| CloudFront Functions | $0.10 per million invocations |
| Real-time logs | Kinesis Data Streams pricing |
| Field-level encryption | $0.02 per 10K requests |
| **Free tier** | **1 TB data out + 10M requests/month (12 months)** |

**Data transfer from S3 to CloudFront is FREE.** This is critical for cost optimization.

### Global Accelerator Pricing

| Component | Price |
|-----------|-------|
| Fixed fee | $0.025/hour per accelerator (~$18/month) |
| Data transfer (DT) premium | Varies by region: $0.015 - $0.091 per GB |
| DT premium is ON TOP of standard AWS data transfer |  |
| **No free tier** |  |

**Key cost insight:** GA charges a data transfer **premium** on top of standard data transfer rates. For 1TB/month from US: ~$85 (standard DT) + ~$15 (GA premium) + ~$18 (fixed) = ~$118/month total.

### Cost Comparison

| Scenario: 1TB/month, global users | CloudFront | Global Accelerator | Route 53 + ALB |
|------------------------------------|------------|-------------------|----------------|
| Monthly cost (approx) | ~$85 (most cached from S3 free transfer) | ~$118 | ~$90 (ALB + DT) |
| Caching benefit | Yes — reduces origin transfer | None | None |
| DNS query cost | Alias = free | N/A (static IPs) | $0.40/M queries |
| Additional features | WAF, Lambda@Edge, geo-restriction | Static IP, instant failover | Routing policies |

### Cost-Saving Strategies

**CloudFront:**
1. **Maximize cache hit ratio:** Proper cache policy (minimize cache key complexity). More cache hits = fewer origin fetches = lower origin transfer cost.
2. **Use S3 as origin:** S3 -> CloudFront transfer is free. EC2/ALB -> CloudFront transfer is not free.
3. **Price Class:** Restrict edge locations to cheaper regions if global coverage isn't needed.
   - Price Class All: all edge locations (most expensive but best performance)
   - Price Class 200: excludes most expensive regions (South America, Australia)
   - Price Class 100: US, Canada, Europe only (cheapest)
4. **Reserved capacity pricing:** For >10 TB/month, commit for 12 months for discounts.
5. **Invalidation management:** Use versioned file names (`app.v2.js`) instead of invalidation requests.
6. **CloudFront Functions over Lambda@Edge:** 6x cheaper for simple transformations.
7. **Compress content:** Enable automatic compression (gzip, Brotli). Reduces data transfer.

**Global Accelerator:**
1. **Only use where value justifies cost:** Static IPs or instant failover are the main reasons.
2. **Use endpoint weights and traffic dial** to control costs during migration.
3. **Don't use for cacheable content** — CloudFront is cheaper and provides caching.

### Cost Traps

- Using Global Accelerator when CloudFront would suffice (paying GA premium for HTTP content that could be cached).
- CloudFront Price Class All when users are only in US/Europe (Price Class 100 is cheaper).
- High invalidation volume (use versioned URLs instead).
- Lambda@Edge for simple header manipulation (CloudFront Functions is 6x cheaper).
- Not compressing content through CloudFront (paying for unnecessary bytes).
- Using Edge-optimized API Gateway (includes built-in CloudFront you can't control) when Regional API GW + your own CloudFront is more cost-effective and flexible.

---

## 5. High Availability, Disaster Recovery & Resilience

### CloudFront HA

**Built-in resilience:**
- 450+ edge locations worldwide. If one edge fails, traffic routes to another.
- Regional edge caches provide a second layer of caching.
- AWS-managed infrastructure, no customer HA configuration needed.

**Origin failover with Origin Groups:**
```
CloudFront Distribution
    --> Origin Group
         Primary:   ALB us-east-1
         Secondary: ALB eu-west-1
         Failover on: 500, 502, 503, 504, or connection timeout
```

- If primary origin returns a configured error code, CloudFront automatically retries on secondary.
- **No health check delay** — failover happens per-request.
- Origin failover does NOT use Route 53 — it is built into CloudFront.
- Secondary origin can be a different region, S3 bucket, or any valid origin.
- **Exam tip:** "Multi-region failover without DNS change" = CloudFront Origin Groups.

### Global Accelerator HA

**Built-in resilience:**
- Traffic enters AWS at the nearest of 100+ edge locations.
- If an edge fails, anycast routing sends to the next nearest edge.
- **Health-based routing:** unhealthy endpoints automatically excluded.

**Multi-region failover:**
```
Endpoint Group us-east-1 (ALB) --- health check --> healthy? route here
Endpoint Group eu-west-1 (ALB) --- health check --> healthy? fallback
Endpoint Group ap-southeast-1   --- health check --> healthy? fallback
```

- If all endpoints in us-east-1 are unhealthy, traffic automatically routes to eu-west-1.
- Failover time: **seconds** (health check failure detection -> immediate reroute).
- No DNS propagation delay (anycast IPs don't change).

**Traffic dial for DR testing:**
- Set traffic dial to 0% for a region to simulate a regional outage.
- Verify traffic routes to remaining healthy regions.
- Set back to 100% when done.

### Failover Time Comparison

| Method | Detection Time | Propagation Time | Total |
|--------|---------------|------------------|-------|
| Route 53 Failover | 30-90s (health check) | 60s+ (DNS TTL) | 90-150+ seconds |
| CloudFront Origin Group | Per-request (origin returns error) | Immediate (next request tries secondary) | ~seconds |
| Global Accelerator | ~10-30s (health check) | Immediate (anycast routing) | ~10-30 seconds |

### DR Architecture Comparison

| Pattern | CloudFront | Global Accelerator |
|---------|------------|-------------------|
| Active-Active | Not applicable (CDN, not routing layer) | Yes — multiple endpoint groups with health checks |
| Active-Passive | Origin Groups (primary/secondary) | Traffic dial (100% primary, 0% secondary until failover) |
| Warm Standby | Secondary origin always available | Secondary region always running, GA routes if needed |
| Pilot Light | Secondary origin (S3 static site for sorry page) | Secondary region minimal instances, scale on failover |

---

## 6. Exam-Focused Section

### Straightforward Questions

**Q1: A company needs to serve a static website globally with the lowest latency. The content is stored in S3. What service should they use?**

**A:** CloudFront distribution with S3 origin. CloudFront caches content at 450+ edge locations, delivering with lowest latency. Use OAC to restrict direct S3 access.

---

**Q2: A gaming company needs to route UDP traffic to game servers in multiple regions with instant failover. Which service?**

**A:** AWS Global Accelerator. It supports UDP at Layer 4, provides instant failover (not DNS-dependent), and routes to the nearest healthy endpoint.

---

**Q3: A company needs static IP addresses for their application that can be whitelisted by partners' firewalls. The application is behind an ALB. Which service provides this?**

**A:** Global Accelerator. It provides 2 static anycast IPv4 addresses. CloudFront IPs are dynamic and cannot be whitelisted.

---

**Q4: What is the difference between CloudFront Functions and Lambda@Edge?**

**A:** CloudFront Functions run at 450+ edge locations, support JavaScript only, execute in < 1ms, and handle viewer request/response only. Lambda@Edge runs at 13 regional edge caches, supports Node.js/Python, executes up to 30s, handles viewer AND origin request/response, and can make network calls.

---

**Q5: How does CloudFront Origin failover work?**

**A:** Using an Origin Group with primary and secondary origins. If the primary returns a configured error status (5xx) or times out, CloudFront automatically routes the request to the secondary origin. No DNS change required.

---

**Q6: A CloudFront distribution serves content from an S3 bucket encrypted with SSE-KMS. Which access control should be used?**

**A:** Origin Access Control (OAC). OAI (legacy) does not support SSE-KMS. OAC uses SigV4 signing which is compatible with KMS-encrypted S3 objects.

---

**Q7: Where must an ACM certificate be created for use with CloudFront?**

**A:** **us-east-1 (N. Virginia)**. CloudFront is a global service and requires certificates in us-east-1 regardless of where your origin is.

---

### Tricky / Scenario-Based Questions

**Q1: A company has a multi-region application with ALBs in us-east-1 and eu-west-1. They need HTTP caching at the edge, WAF protection, AND instant failover between regions (seconds, not minutes). Which architecture?**

**A:** **CloudFront + Global Accelerator together.**
- CloudFront provides HTTP caching and WAF at the edge.
- Global Accelerator provides static IPs and instant failover.
- CloudFront's origin is the Global Accelerator endpoint.
- GA routes to healthy ALB endpoints across regions.

**Why wrong answers are wrong:**
- "CloudFront alone with Origin Groups" — Origin Groups provide per-request failover for origin errors, but don't provide static IPs or the full instant regional failover like GA.
- "Global Accelerator alone" — GA doesn't cache content or support WAF.
- "Route 53 failover + CloudFront" — Route 53 failover has DNS TTL delay (minutes), not seconds.

**Exam keywords:** "caching" + "WAF" + "instant failover" + "static IP" = CloudFront + Global Accelerator.

---

**Q2: A company serves dynamic content from an ALB. They want to improve global performance. The content cannot be cached (personalized per user). Should they use CloudFront or Global Accelerator?**

**A:** **Both can help, but for different reasons:**
- **CloudFront** still helps with dynamic content: TLS termination at edge reduces connection latency, persistent connections to origin (connection reuse), HTTP/2 and HTTP/3 at the edge.
- **Global Accelerator** helps by routing traffic over AWS backbone instead of public internet.

If the question specifies **HTTP-only and no static IP requirement**, CloudFront is preferred (cheaper, more features like WAF). If the question mentions **non-HTTP, static IP, or instant failover**, GA is the answer.

**Exam keyword:** "dynamic content" does NOT automatically mean "not CloudFront." CloudFront accelerates dynamic content via edge TLS offload and persistent connections.

---

**Q3: A company uses CloudFront with an S3 origin. Users report accessing S3 directly (bypassing CloudFront) by using the S3 URL. How do they prevent this?**

**A:** Configure **Origin Access Control (OAC)** on the CloudFront distribution and update the S3 **bucket policy** to allow access only from the CloudFront distribution (using `aws:SourceArn` condition). Remove any public access grants on the bucket.

**Why wrong answers are wrong:**
- "Use signed URLs" — signed URLs prevent unauthorized access through CloudFront, but don't prevent direct S3 access if the bucket is public.
- "Enable S3 Block Public Access" — this alone blocks public access but also blocks CloudFront if using the S3 REST endpoint without OAC.
- "Use a VPC Endpoint for S3" — this is for VPC-to-S3 private access, not CloudFront-to-S3.

**Exam keywords:** "bypass CloudFront" + "direct S3 access" = OAC + bucket policy.

---

**Q4: A company needs to migrate from an on-premises application to AWS without changing the public IP addresses that clients use. Partners have whitelisted these IPs in their firewalls. Which service helps?**

**A:** **Global Accelerator with BYOIP (Bring Your Own IP).** Import the existing IP address range into AWS and assign to the GA accelerator. Clients and partners continue using the same IPs.

**Why wrong answers are wrong:**
- "Elastic IP" — you can't bring arbitrary IPs to EIP (only specific supported ranges to GA BYOIP).
- "CloudFront" — doesn't support static IPs or BYOIP.
- "Route 53" — DNS can point the domain to new IPs, but doesn't solve the IP whitelisting problem. Partners have whitelisted IPs, not domains.

**Exam keywords:** "same IP addresses" + "migration" + "firewall whitelist" = Global Accelerator BYOIP.

---

**Q5: A company uses CloudFront with an ALB origin in us-east-1. They want the ALB to see the client's real IP address, not CloudFront's IP. How?**

**A:** CloudFront adds the **X-Forwarded-For** header containing the client's IP address. The ALB can read this header. No additional configuration needed — CloudFront adds it automatically.

**Why wrong answers are wrong:**
- "Use Global Accelerator for client IP preservation" — GA can preserve client IP, but the question is about CloudFront. Adding GA is unnecessary complexity.
- "Disable CloudFront proxy" — you can't. CloudFront is always a proxy. X-Forwarded-For is the standard solution.
- "Use Lambda@Edge to add the client IP" — unnecessary. CloudFront adds X-Forwarded-For automatically.

**With Global Accelerator:** Client IP preservation is configurable per endpoint. For ALB endpoints, client IP preservation is **enabled by default**. For NLB/EC2, it depends on the configuration. GA uses a different mechanism than X-Forwarded-For (it preserves the actual source IP at the network level for ALBs).

---

**Q6: A company has a CloudFront distribution serving an e-commerce website. They want to show different content to users in different countries (e.g., language, currency). They don't want to route to different origins. What is the best approach?**

**A:** Use the **CloudFront-Viewer-Country** header. Add it to the cache key (via Cache Policy) and forward it to the origin (via Origin Request Policy). The origin application reads this header and returns localized content. CloudFront caches different versions per country.

**Alternatively:** Use **Lambda@Edge** (origin request trigger) to modify the request based on the `CloudFront-Viewer-Country` header before it reaches the origin — e.g., rewrite the URL to a country-specific path.

**Why wrong answers are wrong:**
- "Route 53 Geolocation routing" — works for routing to different endpoints/regions, but the question says "don't want to route to different origins."
- "CloudFront geo-restriction" — only allows or blocks entire countries. Cannot customize content.
- "CloudFront Functions for origin request" — CloudFront Functions only support viewer request/response, not origin request/response.

**Exam keywords:** "different content per country" + "same origin" = CloudFront-Viewer-Country header + cache policy.

---

**Q7: A company serves video content via CloudFront. They want paying subscribers to access premium content while blocking non-subscribers. The premium content is thousands of video segment files (HLS). What is the best approach?**

**A:** **Signed Cookies.** Set a cookie for authenticated/paying users. All HLS segment requests (.ts files) automatically include the cookie. CloudFront validates the cookie signature for each request.

**Why wrong answers are wrong:**
- "Signed URLs" — works for single file access but impractical for HLS with thousands of segments (each would need its own signed URL).
- "S3 presigned URLs" — bypass CloudFront entirely. No edge caching benefit.
- "WAF with IP blocking" — can't distinguish paying vs non-paying users by IP.
- "Lambda@Edge auth check" — works but adds latency and cost for every segment request. Signed cookies are simpler and faster.

**Exam keywords:** "HLS/DASH video" + "multiple segment files" + "restrict access" = Signed Cookies.

---

### Common Exam Traps & Pitfalls

1. **CloudFront vs Global Accelerator — caching:**
   - CloudFront = caching CDN. GA = network accelerator (no caching).
   - If question mentions "cacheable content" = CloudFront. "Non-HTTP" or "UDP" = GA.

2. **Static IPs:**
   - CloudFront = dynamic IPs (DNS-based). Cannot be whitelisted.
   - GA = 2 static anycast IPs. Can be whitelisted. Support BYOIP.
   - "IP whitelist" or "static IP" = Global Accelerator.

3. **ACM certificate region for CloudFront:**
   - Must be in **us-east-1**. Even if your origin is in ap-southeast-1.
   - For ALB, certificate must be in the ALB's region.

4. **OAC vs OAI:**
   - OAC is newer, supports SSE-KMS, supports all S3 features. Exam default.
   - OAI is legacy. If question mentions "SSE-KMS + S3 + CloudFront" = OAC.

5. **WAF is CloudFront only (at the edge):**
   - GA does NOT support WAF. If question needs WAF at edge = CloudFront.
   - WAF Web ACL for CloudFront must be in us-east-1.

6. **Lambda@Edge region:**
   - Lambda@Edge functions must be authored in **us-east-1**. They are then replicated globally.
   - Common distractor: "create Lambda in the same region as your origin."

7. **CloudFront Functions vs Lambda@Edge triggers:**
   - CloudFront Functions: **viewer only** (request + response).
   - Lambda@Edge: viewer AND origin (request + response).
   - "Modify origin request" or "select origin dynamically" = Lambda@Edge (not CF Functions).

8. **CloudFront origin failover is NOT DNS-based:**
   - Origin Groups failover per-request (CloudFront detects error, retries secondary).
   - This is different from Route 53 failover which changes DNS.
   - "Failover without DNS change" = CloudFront Origin Groups or Global Accelerator.

9. **Global Accelerator endpoint types:**
   - ALB, NLB, EC2 instance, Elastic IP. **NOT S3, NOT CloudFront, NOT API Gateway.**
   - If question mentions S3 origin = CloudFront (not GA).

10. **Price Class restriction:**
    - Restricting edge locations reduces cost but increases latency for excluded regions.
    - If question says "reduce CloudFront costs, users mostly in US and Europe" = Price Class 100.

---

## 7. Cheat Sheet

### Must-Know Facts

- **CloudFront:** CDN, Layer 7, HTTP/HTTPS, caches content, 450+ edge locations, WAF integration, dynamic IPs.
- **Global Accelerator:** Network accelerator, Layer 4, TCP/UDP, NO caching, 2 static anycast IPs, instant failover.
- **CloudFront + S3:** OAC (not OAI) for SSE-KMS. S3-to-CloudFront transfer is free.
- **ACM for CloudFront:** must be in **us-east-1**.
- **Lambda@Edge:** must be authored in **us-east-1**. Supports origin triggers.
- **CloudFront Functions:** viewer triggers only. JavaScript. < 1ms. Cheaper.
- **WAF:** CloudFront only (not GA). WAF ACL must be in us-east-1 for CloudFront.
- **Origin Groups:** primary + secondary origin. Per-request failover. No DNS delay.
- **GA failover:** seconds. No DNS caching issue. Static IPs never change.
- **GA BYOIP:** bring your own IPs for migration without IP changes.
- **GA endpoints:** ALB, NLB, EC2, EIP only. Not S3, not API GW.
- **Signed URLs:** single file. **Signed Cookies:** multiple files (HLS video).
- **Price Class:** All (global), 200 (excludes most expensive), 100 (US/Canada/Europe only).

### Decision Flowchart

```
What does the question need?
|
+-- Caching at the edge?
|   --> CloudFront
|
+-- Non-HTTP traffic (UDP/TCP gaming, IoT)?
|   --> Global Accelerator
|
+-- Static IP address (whitelist, BYOIP)?
|   --> Global Accelerator
|
+-- Instant failover (seconds, no DNS delay)?
|   --> Global Accelerator
|       (or CloudFront Origin Groups for per-request origin failover)
|
+-- WAF at the edge?
|   --> CloudFront (GA doesn't support WAF)
|
+-- Content caching + WAF + instant failover?
|   --> CloudFront + Global Accelerator together
|
+-- Dynamic content, HTTP only, no static IP needed?
|   --> CloudFront (still helps with TLS offload, connection reuse)
|
+-- S3 static site, global?
|   --> CloudFront + S3 with OAC
|
+-- HLS video with subscriber auth?
|   --> CloudFront + Signed Cookies
|
+-- Simple header/URL rewrite at edge?
|   --> CloudFront Functions
|
+-- Auth against external API at edge?
|   --> Lambda@Edge
|
+-- SSE-KMS S3 origin?
|   --> CloudFront + OAC (not OAI)
```

### Quick Comparison

| If the question says... | Think... |
|------------------------|----------|
| "Cache static content globally" | CloudFront |
| "UDP / gaming / IoT / non-HTTP" | Global Accelerator |
| "Static IP / IP whitelist" | Global Accelerator |
| "Instant failover / seconds" | Global Accelerator |
| "DNS caching problem / TTL" | Global Accelerator |
| "WAF at edge" | CloudFront |
| "S3 origin + SSE-KMS" | CloudFront + OAC |
| "Zone apex + CDN" | CloudFront + Route 53 Alias |
| "HLS video segments + auth" | CloudFront + Signed Cookies |
| "Origin failover without DNS" | CloudFront Origin Groups |
| "Edge function, simple rewrite" | CloudFront Functions |
| "Edge function, network call needed" | Lambda@Edge |
| "ACM certificate for CDN" | Must be in us-east-1 |
| "Reduce CDN cost, US/EU users" | CloudFront Price Class 100 |
| "BYOIP / keep same IPs during migration" | Global Accelerator BYOIP |
| "Multi-region active-active, HTTP" | CloudFront or GA (depends on caching/static IP need) |

---

*Generated for AWS SAP-C02 exam preparation. Last updated: 2026-04-26.*
