# AWS SAP-C02 Study Notes: S3 (Storage Classes, Replication, Lifecycle, Object Lock, Access Points)

---

## 1. Core Concepts & Theory

### Amazon S3 Fundamentals

Amazon S3 is an **object storage service** offering virtually unlimited storage with **11 9s (99.999999999%) durability** across all storage classes. Objects are stored in **buckets** within a **flat namespace** (the "/" in key names is a logical prefix, not a directory).

#### Key Properties

| Property | Detail |
|----------|--------|
| Object max size | **5 TB** |
| Single PUT max | **5 GB** |
| Multipart upload required | >**100 MB recommended**, >**5 GB mandatory** |
| Max object key length | **1,024 bytes** |
| Bucket namespace | **Globally unique** across all AWS accounts |
| Data consistency | **Strong read-after-write consistency** for all operations (PUT, DELETE, LIST) since Dec 2020 |
| Availability SLA | Varies by storage class (99.5% – 99.99%) |
| Durability | **99.999999999% (11 9s)** for all classes except One Zone-IA |

---

### Storage Classes

S3 offers **7 storage classes**. Knowing the exact numbers is critical for the exam.

| Storage Class | Durability | Availability SLA | AZs | Min Storage Duration | Min Object Size | Retrieval Fee | Use Case |
|--------------|-----------|-----------------|-----|---------------------|----------------|--------------|----------|
| **S3 Standard** | 11 9s | 99.99% | ≥3 | None | None | None | Frequently accessed, general purpose |
| **S3 Intelligent-Tiering** | 11 9s | 99.9% | ≥3 | None | None | None (monitoring fee: $0.0025/1K objects/mo) | Unknown or changing access patterns |
| **S3 Standard-IA** | 11 9s | 99.9% | ≥3 | **30 days** | **128 KB** | Per-GB retrieval | Infrequent access, rapid retrieval needed |
| **S3 One Zone-IA** | 99.999999999% *in one AZ* | 99.5% | **1** | **30 days** | **128 KB** | Per-GB retrieval | Recreatable data, infrequent access |
| **S3 Glacier Instant Retrieval** | 11 9s | 99.9% | ≥3 | **90 days** | **128 KB** | Per-GB retrieval | Archive, needs millisecond access |
| **S3 Glacier Flexible Retrieval** | 11 9s | 99.99% | ≥3 | **90 days** | N/A | Per-GB + per-request | Archive, minutes-to-hours retrieval |
| **S3 Glacier Deep Archive** | 11 9s | 99.99% | ≥3 | **180 days** | N/A | Per-GB + per-request | Long-term archive, 12-48 hour retrieval |

#### Intelligent-Tiering Access Tiers (Automatic)

| Tier | When Moved | Retrieval |
|------|-----------|-----------|
| **Frequent Access** | Default tier | Immediate |
| **Infrequent Access** | After 30 days no access | Immediate |
| **Archive Instant Access** | After 90 days no access | Immediate (milliseconds) |
| **Archive Access** (opt-in) | After 90–730 days configurable | 3–5 hours |
| **Deep Archive Access** (opt-in) | After 180–730 days configurable | 12 hours |

**Key exam detail:** Intelligent-Tiering has **no retrieval fee** and **no minimum storage duration charge**. The only cost is a small monitoring/automation fee per object ($0.0025 per 1,000 objects/month). Objects <128 KB are never moved from Frequent Access tier but are still charged at Frequent Access rate.

#### Glacier Retrieval Options

| Class | Expedited | Standard | Bulk |
|-------|-----------|----------|------|
| **Glacier Flexible Retrieval** | 1–5 minutes | 3–5 hours | 5–12 hours |
| **Glacier Deep Archive** | N/A | 12 hours | 48 hours |

**Provisioned capacity** for Glacier Flexible Retrieval guarantees Expedited retrieval availability ($100/unit/month).

---

### S3 Replication

S3 Replication copies objects between buckets **asynchronously**.

#### Replication Types

| Type | Description |
|------|-------------|
| **CRR (Cross-Region Replication)** | Between buckets in **different Regions** |
| **SRR (Same-Region Replication)** | Between buckets in the **same Region** |

#### Replication Requirements

- **Versioning must be enabled** on both source and destination buckets.
- Source bucket owner must have permissions to replicate to the destination.
- IAM role with `s3:ReplicateObject`, `s3:ReplicateDelete`, `s3:GetReplicationConfiguration` permissions.
- If buckets are in **different accounts**, the destination bucket policy must grant the source account permissions.

#### Replication Scope & Filtering

- Replicate **all objects** or filter by **prefix** and/or **tag**.
- Multiple replication rules with priority ordering.
- Can replicate to **multiple destination buckets** (multi-destination replication).

#### What IS Replicated

- New objects created after replication is enabled (by default).
- Object metadata, tags, ACLs.
- Objects encrypted with SSE-S3 or SSE-KMS (must explicitly enable KMS replication and grant key permissions).
- Object Lock retention/legal hold (if destination bucket has Object Lock enabled).

#### What IS NOT Replicated (by default)

- **Pre-existing objects** — use **S3 Batch Replication** to replicate existing objects.
- **Objects in Glacier/Deep Archive** — must restore first. (Exception: lifecycle-transitioned objects can be replicated if using S3 Replication Time Control.)
- **Delete markers** — NOT replicated by default. You must explicitly enable **delete marker replication**. Even then, **version-specific deletes (permanent deletes) are NEVER replicated** — this prevents malicious cross-account deletion.
- **Objects already replicated from another source** (no chaining — A→B→C is NOT automatic. You need explicit A→B and A→C or B→C rules).
- **SSE-C encrypted objects** (customer-provided keys).
- **Bucket-level configurations** (lifecycle rules, notifications, etc.).

#### S3 Replication Time Control (S3 RTC)

- SLA: **99.99% of objects replicated within 15 minutes**.
- Enables CloudWatch metrics for replication monitoring.
- Required for compliance use cases needing predictable replication times.
- **Additional cost** over standard replication.

#### S3 Batch Replication

- Replicate **existing objects** that were created before replication was enabled.
- Replicate objects that previously failed replication.
- Replicate objects across accounts.
- Uses S3 Batch Operations under the hood.

---

### S3 Lifecycle Policies

Lifecycle policies automate transitioning objects between storage classes or expiring (deleting) them.

#### Lifecycle Actions

| Action | Description |
|--------|-------------|
| **Transition** | Move objects to a different storage class after N days |
| **Expiration** | Delete objects after N days |
| **NoncurrentVersionTransition** | Transition noncurrent versions (versioned bucket) |
| **NoncurrentVersionExpiration** | Delete noncurrent versions after N days |
| **AbortIncompleteMultipartUpload** | Clean up incomplete multipart uploads after N days |
| **ExpiredObjectDeleteMarker** | Remove delete markers when they are the only version remaining |

#### Transition Waterfall (Allowed Directions)

```
S3 Standard
  → Standard-IA (min 30 days after creation)
  → Intelligent-Tiering
  → One Zone-IA (min 30 days after creation)
  → Glacier Instant Retrieval (min 90 days)
  → Glacier Flexible Retrieval (min 90 days)
  → Glacier Deep Archive (min 180 days)
```

**Rules:**
- Transitions flow **downward only** (you cannot lifecycle-transition from Glacier back to Standard — you must restore + copy).
- **30-day minimum** before transitioning from Standard to any IA class.
- **30-day minimum** in Standard-IA before transitioning to Glacier classes (so minimum 60 days from creation to Glacier if going through IA).
- **Direct transition** from Standard to Glacier/Deep Archive is allowed (skip IA).
- Objects **smaller than 128 KB** are NOT transitioned to IA or Glacier Instant Retrieval classes (they stay in Standard; you're charged at the Standard rate).
- You CAN have lifecycle rules that transition objects to Intelligent-Tiering from any class.

#### Lifecycle + Versioning Interaction

- On versioned buckets, lifecycle rules apply to the **current version** (Transition/Expiration actions) and **noncurrent versions** separately.
- **Expiration on current version** = creates a **delete marker** (doesn't permanently delete).
- **NoncurrentVersionExpiration** = permanently deletes noncurrent versions after N days.
- Common pattern: Keep current version in Standard, transition noncurrent versions to Glacier after 30 days, delete noncurrent versions after 365 days.

---

### S3 Object Lock

Object Lock provides **WORM (Write Once Read Many)** protection for objects. Once locked, an object version **cannot be deleted or overwritten** for the retention period.

#### Prerequisites

- **Versioning must be enabled** (Object Lock requires versioning).
- Object Lock must be enabled **at bucket creation time** — **cannot be enabled on an existing bucket** (you can enable it via support ticket or by creating a new bucket and copying objects).

#### Retention Modes

| Mode | Who Can Delete/Overwrite? | Use Case |
|------|--------------------------|----------|
| **Governance Mode** | Users with `s3:BypassGovernanceRetention` permission CAN override or delete. Regular users cannot | Internal compliance; flexibility for admins |
| **Compliance Mode** | **Nobody** — not even the root account. Cannot be shortened or removed during the retention period | Regulatory compliance (SEC 17a-4, FINRA, HIPAA) |

#### Legal Hold

- **Separate from retention** — can be applied independently.
- No expiration date — remains until explicitly removed.
- Any user with `s3:PutObjectLegalHold` can apply or remove.
- An object can have BOTH a retention period AND a legal hold.
- Object is protected if **either** the retention period is active OR the legal hold is in place.

#### Object Lock + Replication

- Retention and legal hold metadata ARE replicated (if destination bucket has Object Lock enabled).
- The destination bucket must have Object Lock enabled for replication to preserve WORM protection.

#### Default Retention

- You can set a **default retention configuration** on a bucket — all new objects automatically get that retention.
- Can be overridden per-object at upload time (with longer retention only in Compliance mode; Governance mode allows override with permission).

---

### S3 Access Points

Access Points simplify managing access to shared S3 buckets. Each access point has its own **DNS name**, **IAM access point policy**, and **network controls**.

#### Key Features

| Feature | Description |
|---------|-------------|
| **Dedicated DNS endpoint** | Each access point gets its own URL: `https://<access-point-name>-<account-id>.s3-accesspoint.<region>.amazonaws.com` |
| **Access point policy** | JSON policy (like a bucket policy) scoped to the access point. Must be a **subset** of permissions granted by the bucket policy |
| **VPC restriction** | An access point can be restricted to a **specific VPC** — requests from outside that VPC are denied |
| **Block Public Access** | Each access point has its own Block Public Access settings (independent of the bucket's BPA) |
| **Per prefix/team access** | Create separate access points for different teams/applications, each with tailored permissions |
| **Multi-Region Access Points** | Route S3 requests to the nearest bucket across Regions using AWS Global Accelerator routing |

#### S3 Multi-Region Access Points (MRAP)

- A **single global endpoint** that routes requests to the **nearest S3 bucket** across up to 20 Regions.
- Uses **AWS Global Accelerator** network for optimized routing.
- Supports **active-active** configuration with S3 Cross-Region Replication.
- Supports **failover controls** — promote/demote Regions as active/passive.
- For the exam: MRAP + CRR = multi-Region active-active storage with automatic failover.

#### Access Points + Delegated Access

- In multi-account setups, you can delegate access point creation to other accounts via SCPs and bucket policies.
- Bucket policy can reference access points using condition keys: `s3:AccessPointNetworkOrigin` (Internet or VPC).

#### Object Lambda Access Points

- Transform data as it's retrieved from S3 using a Lambda function.
- Use case: Redact PII, convert formats, resize images on read.
- The Lambda function sits between the access point and S3.
- Original data in S3 is unchanged — transformation happens at read time.

---

### Additional S3 Features (Exam Relevant)

#### S3 Event Notifications

- Trigger on object events (Created, Removed, Restore, Replication).
- Targets: **SNS, SQS, Lambda, EventBridge**.
- **EventBridge** destination supports all S3 events, filtering, and routing to 18+ AWS services. Preferred for complex event routing.

#### S3 Select & Glacier Select

- Query objects using SQL (CSV, JSON, Parquet).
- Returns only the matching data — reduces data transfer and processing time.
- Works on Standard, IA, and Glacier (Glacier Select for archived data).

#### S3 Transfer Acceleration

- Uses **CloudFront edge locations** to accelerate uploads to S3.
- Must be enabled on the bucket.
- Use the `<bucket>.s3-accelerate.amazonaws.com` endpoint.
- Best for long-distance, large object uploads.

#### S3 Batch Operations

- Perform bulk operations on billions of objects: copy, invoke Lambda, restore from Glacier, replace tags, replace ACLs, apply Object Lock retention.
- Uses an **inventory report** or **CSV manifest** as input.
- Integrates with S3 Inventory for generating object lists.

#### S3 Inventory

- Generates daily or weekly CSV/ORC/Parquet reports listing all objects and metadata.
- Use for auditing encryption status, replication status, storage class, etc.
- Feeds into S3 Batch Operations, Athena queries, or analytics.

#### S3 Storage Lens

- Organization-wide storage analytics and recommendations.
- Aggregates metrics across accounts, Regions, buckets.
- Free tier: 28 metrics. Advanced tier: 35+ metrics with recommendations ($0.20/million objects monitored).

---

### Default Limits Worth Memorizing

| Resource | Limit | Adjustable? |
|----------|-------|-------------|
| Buckets per account | **100** (soft limit) | Yes (up to 1,000) |
| Object size | **5 TB** max | No |
| Single PUT | **5 GB** max | No |
| Multipart upload parts | **10,000** max parts | No |
| Lifecycle rules per bucket | **1,000** | No |
| Replication rules per bucket | **1,000** | No |
| Access points per account per Region | **10,000** | Yes |
| S3 request rate per prefix | **5,500 GET/HEAD/s**, **3,500 PUT/COPY/POST/DELETE/s** | No (but scales per prefix) |
| Bucket policy max size | **20 KB** | No |
| Access point policy max size | **20 KB** | No |
| Object tags per object | **10** | No |
| S3 notifications per bucket | No hard limit (but one event can only go to one target per configuration) | N/A |

---

## 2. Design Patterns & Best Practices

### When to Use Which Storage Class

| Scenario | Storage Class | Why |
|----------|--------------|-----|
| Frequently accessed data, performance-sensitive | **S3 Standard** | Lowest latency, no retrieval fees |
| Access pattern unknown or changing | **S3 Intelligent-Tiering** | Automatic optimization, no retrieval fees |
| Infrequent access (<1/month), but need immediate retrieval | **S3 Standard-IA** | 45% cheaper than Standard, millisecond access |
| Infrequent access, data is recreatable (thumbnails, replicas) | **S3 One Zone-IA** | 20% cheaper than Standard-IA, single AZ risk acceptable |
| Archive data, need instant access for occasional compliance queries | **Glacier Instant Retrieval** | 68% cheaper than Standard, millisecond access |
| Archive data, can tolerate minutes-to-hours retrieval | **Glacier Flexible Retrieval** | Very low cost, flexible retrieval speeds |
| Long-term archive (7+ years), regulatory retention | **Glacier Deep Archive** | Cheapest storage (~$1/TB/month), 12-48hr retrieval |

### Anti-Patterns

| Anti-Pattern | Why It's Wrong | Correct Approach |
|-------------|---------------|------------------|
| Using Standard for rarely accessed compliance logs | Massive cost waste | Intelligent-Tiering or lifecycle to Glacier |
| Using One Zone-IA for critical, non-recreatable data | Single AZ failure = data loss | Standard-IA or Glacier Instant Retrieval (multi-AZ) |
| Using Glacier for data needed in <1 minute | Glacier Flexible Retrieval takes minutes-hours | Glacier Instant Retrieval or Standard-IA |
| Lifecycle transition objects <128 KB to IA/Glacier Instant | They won't be transitioned; min size applies | Keep in Standard or use Intelligent-Tiering |
| Relying on S3 replication for backup without testing restore | Replication is async; accidental deletes may replicate | Enable versioning + Object Lock for true protection |
| Using bucket policies for 50+ teams/applications on shared bucket | Policy becomes unmanageable (20 KB limit) | Use S3 Access Points — one per team/app |
| Cross-Region Replication without enabling KMS replication | SSE-KMS encrypted objects silently fail to replicate | Explicitly enable KMS replication in the replication config |

### Common Architectural Patterns

#### Data Lake on S3
```
Raw Zone (S3 Standard) → Processed Zone (S3 Standard) → Curated Zone (S3 Standard/IA)
     ↓ lifecycle                ↓ lifecycle                    ↓ lifecycle
Glacier Flexible (90d)    Intelligent-Tiering          Glacier Deep Archive (365d)

Query: Athena / Redshift Spectrum / EMR
Catalog: AWS Glue Data Catalog
```

#### Multi-Account Data Sharing
```
Central Data Account:
  S3 Bucket → Access Point A (Team-Analytics account, /analytics/* prefix)
            → Access Point B (Team-ML account, /ml-data/* prefix)
            → Access Point C (VPC-restricted, /sensitive/* prefix)
```

#### Multi-Region Active-Active Storage
```
S3 Multi-Region Access Point (global endpoint)
  → us-east-1 bucket (active) ←CRR→ eu-west-1 bucket (active)
  
Client → MRAP → nearest bucket (auto-routed via Global Accelerator)
Failover: MRAP failover controls demote unhealthy Region
```

#### Compliance Archive
```
S3 Bucket (Object Lock: Compliance Mode, 7-year retention)
  → Glacier Deep Archive (lifecycle after 90 days)
  → S3 Inventory → Athena (audit queries)
  → CloudTrail Data Events (access logging)
  → S3 Replication → DR Region (Object Lock on destination too)
```

#### Hybrid: On-Prem to S3
```
On-premises data center
  → AWS Storage Gateway (File Gateway) → S3 Standard
  → AWS DataSync (scheduled sync) → S3 Standard
  → S3 Transfer Acceleration (long-distance uploads)
  → AWS Snow Family (large-scale offline migration)
```

### Well-Architected Framework Alignment

**Reliability**
- S3 provides 11 9s durability across ≥3 AZs (except One Zone-IA).
- Use CRR for cross-Region resilience.
- Enable versioning to protect against accidental deletes/overwrites.
- Use Object Lock (Compliance mode) for regulatory immutability.
- Use MRAP with failover controls for multi-Region availability.

**Security**
- Enable S3 Block Public Access at account AND bucket level.
- Use bucket policies + access points for fine-grained access.
- Enable SSE-S3 (default), SSE-KMS, or SSE-C for encryption.
- Enable CloudTrail data events + S3 server access logging.
- Use VPC endpoints (Gateway) for private access without internet.
- Use Object Lambda access points for PII redaction on read.

**Performance**
- S3 scales per prefix: 5,500 GET/s, 3,500 PUT/s per prefix.
- Use random/hashed prefixes to distribute requests (less critical now but still good practice).
- Use Transfer Acceleration for long-distance uploads.
- Use S3 Select to reduce data transferred.
- Use multipart upload for large objects (parallel parts).
- Use byte-range fetches for parallel downloads.

**Cost Optimization**
- Use Intelligent-Tiering for unpredictable access.
- Use lifecycle policies to transition to cheaper classes.
- Use S3 Storage Lens to identify optimization opportunities.
- Use S3 Inventory + Athena to audit storage class distribution.
- Abort incomplete multipart uploads (lifecycle rule).
- Use Requester Pays for shared datasets.

**Operational Excellence**
- Use S3 Inventory for compliance auditing.
- Use S3 Storage Lens for dashboard visibility.
- Use EventBridge for event-driven automation.
- Use S3 Batch Operations for bulk remediation (encrypt, tag, transition).

---

## 3. Security & Compliance

### Access Control Hierarchy

S3 evaluates access using a **combination of policies** (union with explicit deny taking precedence):

1. **IAM policies** (identity-based)
2. **Bucket policies** (resource-based)
3. **Access Point policies** (scoped resource-based)
4. **ACLs** (legacy — AWS recommends disabling via Bucket Owner Enforced)
5. **S3 Block Public Access** (overrides everything — if BPA denies, nothing can make it public)
6. **VPC Endpoint policies** (restricts access from within the VPC)
7. **SCPs** (Organization-level guardrails)

#### S3 Block Public Access Settings

| Setting | What It Blocks |
|---------|---------------|
| `BlockPublicAcls` | Blocks PUT calls with public ACLs |
| `IgnorePublicAcls` | Ignores existing public ACLs |
| `BlockPublicPolicy` | Blocks bucket policies that grant public access |
| `RestrictPublicBuckets` | Restricts access to AWS services and authorized users only for buckets with public policies |

- Can be set at **account level** (applies to all buckets) or **bucket level**.
- **Best practice**: Enable all four at the account level. Only disable per-bucket when explicitly needed (e.g., static website hosting).

#### Cross-Account Access Patterns

| Method | Use Case |
|--------|----------|
| **Bucket policy** | Grant specific account/role access to the bucket |
| **Access Points** | Per-team/app access with dedicated policies |
| **IAM role assumption** | Source account assumes role in destination account |
| **S3 Object Ownership: Bucket Owner Enforced** | Ensures bucket owner owns all uploaded objects (disables ACLs). **Required for cross-account uploads to avoid ownership confusion** |

**Exam critical:** By default, when Account B uploads an object to Account A's bucket, **Account B owns the object** and Account A cannot access it. Fix: Enable **Bucket Owner Enforced** on the bucket (disables ACLs, bucket owner automatically owns all objects).

### Encryption

#### Encryption at Rest

| Method | Key Management | Use Case |
|--------|---------------|----------|
| **SSE-S3** (AES-256) | AWS manages keys entirely. **Default since Jan 2023** — all new objects encrypted by default | General purpose. No KMS costs |
| **SSE-KMS** | AWS KMS manages keys (default or CMK). Separate permissions for key usage | Regulatory compliance, audit trail (CloudTrail logs every key usage), fine-grained access control via KMS key policy |
| **SSE-C** | Customer provides the encryption key with each request. AWS does NOT store the key | Customer controls keys completely; must manage key distribution |
| **Client-side encryption** | Encrypt before upload. AWS never sees plaintext | Maximum security; complex to manage |

**Exam details:**
- **SSE-KMS has a request quota** (5,500–30,000 decrypt requests/second per Region depending on Region). High-throughput workloads may need SSE-S3 instead, or request a KMS quota increase.
- **SSE-KMS with S3 Bucket Keys** reduces KMS API calls (and costs) by generating a bucket-level key. **Reduces cost by up to 99%** for KMS-encrypted buckets.
- You can **enforce encryption** via bucket policy:
```json
{
  "Effect": "Deny",
  "Action": "s3:PutObject",
  "Resource": "arn:aws:s3:::my-bucket/*",
  "Condition": {
    "StringNotEquals": { "s3:x-amz-server-side-encryption": "aws:kms" }
  }
}
```

#### Encryption in Transit

- S3 supports **TLS 1.2+** for all API calls.
- Enforce HTTPS via bucket policy:
```json
{
  "Effect": "Deny",
  "Action": "s3:*",
  "Resource": ["arn:aws:s3:::my-bucket", "arn:aws:s3:::my-bucket/*"],
  "Condition": { "Bool": { "aws:SecureTransport": "false" } }
}
```

### Logging & Auditing

| Method | What It Logs | Cost |
|--------|-------------|------|
| **S3 Server Access Logging** | Detailed request logs (requester, bucket, key, operation, response). Delivered to a target S3 bucket | Free (pay for log storage) |
| **CloudTrail Data Events** | API-level logging for S3 operations (GetObject, PutObject). Integrates with CloudWatch, EventBridge | Paid ($0.10/100K events) |
| **S3 Inventory** | Scheduled reports of all objects with metadata (encryption status, storage class, replication) | Paid (per million objects listed) |
| **S3 Storage Lens** | Account/Org-wide dashboards, anomaly detection, recommendations | Free (basic) / Paid (advanced) |
| **CloudWatch Metrics** | Bucket-level metrics: BucketSizeBytes, NumberOfObjects. Request metrics (optional): 4xx/5xx errors, latency | Free (storage metrics) / Paid (request metrics) |

**Exam trap:** S3 Server Access Logging vs CloudTrail Data Events — Server Access Logs are best-effort (may miss some records) and are delivered to an S3 bucket. CloudTrail provides guaranteed delivery and integrates with CloudWatch/EventBridge for alerts. For compliance auditing, **CloudTrail** is the recommended answer.

### Compliance Patterns

- **SEC 17a-4 / FINRA**: Object Lock Compliance mode + Glacier Deep Archive.
- **HIPAA**: SSE-KMS encryption + CloudTrail data events + S3 Block Public Access + VPC endpoint access.
- **GDPR right to erasure**: If Object Lock Governance mode is used, authorized admins can delete. Compliance mode prevents deletion — plan retention periods carefully.
- **Data residency**: Use SCPs to restrict S3 operations to approved Regions. Use S3 bucket policies with `aws:RequestedRegion` condition.

---

## 4. Cost Optimization

### Pricing Components

| Component | S3 Standard (us-east-1) | Notes |
|-----------|------------------------|-------|
| Storage | $0.023/GB/month | First 50 TB |
| PUT/COPY/POST/LIST | $0.005/1,000 requests | |
| GET/SELECT | $0.0004/1,000 requests | |
| Data transfer OUT | $0.09/GB (first 10 TB) | To internet. Transfer to CloudFront is free |
| Data transfer to same Region | Free | |
| Data transfer cross-Region | $0.02/GB | Replication costs this |
| Retrieval (IA/Glacier) | Varies by class | Per-GB fee |
| Lifecycle transition | $0.01/1,000 requests | Per transition request |

### Storage Class Cost Comparison (us-east-1, per GB/month)

| Class | Storage $/GB | Retrieval $/GB | Min Duration Charge |
|-------|-------------|---------------|-------------------|
| Standard | $0.023 | $0 | None |
| Intelligent-Tiering | $0.023 (Frequent) / $0.0125 (Infrequent) / $0.004 (Archive Instant) | $0 | None |
| Standard-IA | $0.0125 | $0.01 | 30 days |
| One Zone-IA | $0.01 | $0.01 | 30 days |
| Glacier Instant Retrieval | $0.004 | $0.03 | 90 days |
| Glacier Flexible Retrieval | $0.0036 | $0.01 (Standard) | 90 days |
| Glacier Deep Archive | **$0.00099** | $0.02 (Standard) | 180 days |

### Cost-Saving Strategies

1. **Intelligent-Tiering for unknown patterns**: No retrieval fees, automatic optimization. Monitoring fee is negligible for objects >128 KB.

2. **Lifecycle policies**: Transition logs from Standard → Standard-IA (30 days) → Glacier Flexible (90 days) → Deep Archive (365 days). Expire after 2,555 days (7 years) if regulatory allows.

3. **Abort incomplete multipart uploads**: Create a lifecycle rule to abort after 7 days. These fragments accumulate silently and cost money.

4. **S3 Bucket Keys with SSE-KMS**: Reduces KMS API calls by up to 99%, saving significant KMS costs on high-throughput buckets.

5. **Requester Pays**: Shared data sets where consumers should bear transfer costs.

6. **S3 Storage Lens**: Identify buckets with no lifecycle policies, objects in wrong storage classes, orphaned multipart uploads.

7. **One Zone-IA for recreatable data**: 20% cheaper than Standard-IA. Use for thumbnails, transcoded media, replicated data.

8. **S3 Select / Glacier Select**: Query inside objects instead of downloading the full object — reduces data transfer costs.

9. **VPC Gateway Endpoint for S3**: Free. Eliminates NAT Gateway data processing charges for S3 access from private subnets ($0.045/GB saved).

### Common Cost Traps

- **Minimum storage duration charges**: Deleting an object from Standard-IA after 10 days = charged for 30 days. From Deep Archive after 30 days = charged for 180 days.
- **Minimum object size (128 KB)**: Objects <128 KB in Standard-IA are charged as 128 KB. Store small objects in Standard or Intelligent-Tiering.
- **Glacier retrieval fees on bulk restore**: Restoring 10 TB from Deep Archive Standard retrieval = $200 in retrieval fees alone.
- **Lifecycle transition request fees**: Transitioning millions of small objects incurs $0.01/1,000 transition requests. May cost more than the storage savings.
- **Cross-Region Replication data transfer**: CRR incurs $0.02/GB cross-Region transfer + storage in destination Region. Budget for both.
- **Intelligent-Tiering monitoring fee**: $0.0025/1,000 objects/month. For billions of tiny objects, this adds up. Best for fewer, larger objects.
- **CloudTrail S3 data events**: At $0.10/100K events, high-volume buckets can generate significant CloudTrail costs.

---

## 5. High Availability, Disaster Recovery & Resilience

### S3 Inherent Resilience

- **11 9s durability** — objects stored across **≥3 AZs** (except One Zone-IA: 1 AZ).
- **99.99% availability** for Standard class.
- S3 automatically handles hardware failures, bit rot (checksums), and facility-level events.
- **No action needed for single AZ failure** — S3 Standard automatically survives AZ loss.

### DR Patterns

| Pattern | Implementation | RPO | RTO |
|---------|---------------|-----|-----|
| **Same-Region backup** | SRR to a backup bucket + versioning | Minutes (async replication) | Seconds (redirect to backup bucket) |
| **Cross-Region DR** | CRR to DR Region + versioning | Minutes (15 min with RTC) | Seconds–Minutes (switch endpoint) |
| **Multi-Region Active-Active** | MRAP + bidirectional CRR | Near-zero | Near-zero (automatic routing) |
| **Ransomware protection** | Object Lock (Compliance) + versioning + CRR | Zero (immutable) | Seconds (versions intact) |
| **Accidental deletion protection** | Versioning + MFA Delete + lifecycle for noncurrent versions | Zero (version preserved) | Seconds (restore previous version) |

### MFA Delete

- Requires MFA to **permanently delete** an object version or change bucket versioning state.
- Can only be enabled by the **root account** via the CLI (not console).
- Adds protection against compromised credentials.

### S3 Versioning for Resilience

- Every overwrite/delete creates a new version — previous versions are preserved.
- Delete = adds a **delete marker** (current version becomes the marker; previous version is still accessible).
- Permanent delete = deleting a specific version ID.
- **Cost implication**: All versions count toward storage costs. Use lifecycle rules to expire noncurrent versions.

### Multi-Region Access Points (MRAP) for DR

```
MRAP Global Endpoint
  ├── us-east-1 bucket (ACTIVE)
  ├── eu-west-1 bucket (ACTIVE)
  └── ap-southeast-1 bucket (ACTIVE)

Bidirectional CRR between all buckets
MRAP routes to nearest/healthiest bucket
Failover: promote/demote Regions via MRAP failover controls
```

---

## 6. Exam-Focused Section

### Straightforward Questions (7 examples)

**Q1:** What is the durability of S3 Standard?
**A:** **99.999999999% (11 9s).** Applies to all storage classes except One Zone-IA (11 9s in one AZ — if the AZ is destroyed, data is lost).

**Q2:** A company needs to store compliance data for 7 years with immutable protection that even the root user cannot override. Which S3 feature should they use?
**A:** **S3 Object Lock in Compliance mode.** No one, including root, can delete or shorten the retention period.

**Q3:** What must be enabled on both source and destination buckets for S3 replication?
**A:** **Versioning.** Replication requires versioning on both buckets.

**Q4:** What is the minimum storage duration charge for S3 Glacier Deep Archive?
**A:** **180 days.** If you delete an object before 180 days, you are still charged for 180 days.

**Q5:** How can a company prevent public access to all S3 buckets across an AWS Organization?
**A:** Enable **S3 Block Public Access at the account level** for all member accounts, enforced via an **SCP** that denies `s3:PutBucketPublicAccessBlock` modifications.

**Q6:** A bucket policy is 18 KB and the team needs to add more access rules but is approaching the 20 KB limit. What should they do?
**A:** Use **S3 Access Points.** Create separate access points with their own policies (each up to 20 KB), reducing the bucket policy size.

**Q7:** What is the retrieval time for S3 Glacier Instant Retrieval?
**A:** **Milliseconds.** Same as Standard/Standard-IA — the name is confusing, but "Instant" means immediate.

---

### Tricky / Scenario-Based Questions (7 examples)

**Q1:** A company enables Cross-Region Replication from us-east-1 to eu-west-1. The bucket has 50 TB of existing data and 1 TB of daily new data. After enabling CRR, they notice only new objects are replicating. The existing 50 TB is not replicated. What should they do?

**A:** Use **S3 Batch Replication** to replicate the existing 50 TB.
- **Why doesn't CRR handle this?** CRR only replicates **new objects** created after the rule is enabled. It does NOT retroactively replicate existing objects.
- **Why not copy via CLI (`aws s3 cp`)?** Works but is slower, not monitored/managed, and doesn't preserve replication metadata. Batch Replication is the purpose-built solution.
- **Key phrases:** "existing objects" + "not replicated" → S3 Batch Replication.

**Q2:** A company stores financial records in S3 with Object Lock in Compliance mode and a 7-year retention period. After 2 years, the legal team discovers that some records were stored with incorrect data and need to be deleted. Can they delete these objects?

**A:** **No.** In Compliance mode, **nobody** — not even the root account — can delete, overwrite, or shorten the retention period for the remaining 5 years.
- **Why not switch to Governance mode?** You **cannot change Compliance mode** to Governance mode. And even if it were Governance mode, you'd need `s3:BypassGovernanceRetention`.
- **What can they do?** They can upload a corrected version (which will also be locked for 7 years). The incorrect version remains until its retention expires.
- **Key phrases:** "Compliance mode" + "delete before retention expires" → Impossible.

**Q3:** A company uses S3 Standard-IA for log files. They create a lifecycle rule to transition objects to Glacier Flexible Retrieval after 15 days. The rule appears to not work — objects remain in Standard-IA. Why?

**A:** There is a **30-day minimum** before transitioning from Standard-IA to any Glacier class. The 15-day rule is invalid because objects must be in Standard-IA for at least 30 days before they can transition further.
- **Fix:** Set the Glacier transition to ≥60 days from object creation (30 days in Standard-IA + 30 days minimum retention) OR transition directly from Standard to Glacier (skipping IA — direct transition has no 30-day IA constraint, just a 1-day minimum from creation).
- **Key phrases:** "Standard-IA" + "lifecycle to Glacier" + "15 days" → 30-day minimum in IA.

**Q4:** A company has CRR enabled with SSE-KMS encryption. Objects in the source bucket are encrypted with a CMK, but replication status shows "FAILED" for all objects. What is the most likely cause?

**A:** **KMS replication was not explicitly enabled in the replication configuration**, and/or the **replication IAM role lacks `kms:Decrypt` on the source CMK and `kms:Encrypt` on the destination CMK**.
- **Why this happens:** By default, CRR does NOT replicate SSE-KMS encrypted objects. You must: (1) Explicitly enable KMS replication in the replication rule. (2) Specify the destination KMS key. (3) Grant the replication role `kms:Decrypt` on source key and `kms:Encrypt` on destination key.
- **Why not SSE-S3?** SSE-S3 encrypted objects replicate without any special configuration.
- **Key phrases:** "CRR" + "SSE-KMS" + "replication FAILED" → KMS replication not enabled/configured.

**Q5:** A company has a single S3 bucket accessed by 200 applications across 50 AWS accounts. The bucket policy has grown to 19.5 KB and they need to add permissions for 10 more applications. What is the best approach?

**A:** Use **S3 Access Points.** Create one access point per application or per team, each with its own policy. The bucket policy can delegate to access points.
- **Why not just increase the bucket policy?** Max size is **20 KB** — hard limit, not adjustable. Adding more entries will exceed it.
- **Why not IAM policies only?** IAM policies are on the identity side — for cross-account access, you still need a resource-based policy (bucket or access point).
- **Key phrases:** "bucket policy size limit" + "many applications/accounts" → Access Points.

**Q6:** An architect uses S3 Intelligent-Tiering for a data lake. They notice tiny objects (50 bytes each — metadata records) are charged at the Frequent Access tier rate even after months of no access. Why aren't they moving to the Infrequent Access tier?

**A:** Objects **smaller than 128 KB are NOT moved to lower tiers** in Intelligent-Tiering. They remain in the Frequent Access tier permanently.
- **Fix:** Aggregate small objects into larger files (e.g., combine metadata records into batch files). Or accept the Standard-rate cost for small objects.
- **Why is this a trap?** Candidates assume Intelligent-Tiering optimizes ALL objects. It doesn't — the 128 KB minimum applies.
- **Key phrases:** "Intelligent-Tiering" + "small objects" + "not transitioning" → 128 KB minimum.

**Q7:** A company enables S3 replication with delete marker replication enabled. An employee permanently deletes an object version in the source bucket. The company checks the destination bucket and finds the object version is still present. Is this expected?

**A:** **Yes, this is expected and by design.** S3 replication does NOT replicate **permanent deletes (version-specific deletes)**. Only **delete markers** are replicated (when enabled). This is a **safety feature** to prevent malicious or accidental permanent deletion from propagating cross-Region/cross-account.
- **Exam trap:** Candidates confuse "delete marker replication" with "delete replication." They are different. Delete markers (soft delete) can be replicated. Permanent deletes (specific version ID) are NEVER replicated.
- **Key phrases:** "permanent delete" + "version-specific delete" + "destination still has object" → By design, not replicated.

---

### Common Exam Traps & Pitfalls

1. **One Zone-IA durability vs availability**: Durability is still 11 9s *within that AZ*. But if the AZ is destroyed, data is gone. Standard-IA has 11 9s across multiple AZs. Questions test whether candidates confuse durability (within AZ) with AZ resilience.

2. **Glacier Instant Retrieval vs Glacier Flexible Retrieval**: "Instant" = milliseconds. "Flexible" = minutes to hours. The word "Instant" is the differentiator. If the question says "must access archived data immediately," the answer is Glacier Instant Retrieval, NOT Glacier Flexible with Expedited.

3. **S3 Replication does NOT replicate existing objects**: Only new objects after rule creation. Batch Replication handles existing. This is tested frequently.

4. **Permanent deletes are NEVER replicated**: Delete markers can be. This is the most common replication confusion on the exam.

5. **SSE-KMS replication requires explicit configuration**: SSE-S3 just works. SSE-KMS needs special setup. SSE-C objects are NEVER replicated.

6. **Lifecycle 30-day minimum for IA classes**: Cannot transition from Standard to Standard-IA in less than 30 days. Cannot transition from Standard-IA to Glacier in less than 30 days after arriving in IA (60 days total from creation if going Standard→IA→Glacier).

7. **128 KB minimum for IA and Glacier Instant**: Small objects are charged as 128 KB in IA classes. In Intelligent-Tiering, small objects stay in Frequent Access tier.

8. **Object Lock only at bucket creation**: Cannot enable on existing bucket (without contacting AWS Support). Must plan ahead.

9. **Compliance mode is truly immutable**: Not even root can delete. Governance mode allows bypass with permissions. Questions test whether candidates know the difference.

10. **S3 Block Public Access overrides bucket policies**: Even if a bucket policy allows public access, BPA blocks it. BPA at the account level overrides bucket-level BPA settings.

11. **S3 request rate is PER PREFIX**: 5,500 GET/s and 3,500 PUT/s per prefix. Total bucket throughput scales with the number of prefixes. Not a per-bucket limit.

12. **Cross-account object ownership**: By default, the uploader owns the object, not the bucket owner. Enable **Bucket Owner Enforced** to change this. ACLs are then disabled.

13. **S3 Transfer Acceleration vs CloudFront**: Transfer Acceleration speeds up uploads TO S3. CloudFront speeds up downloads FROM S3. Different directions.

14. **Intelligent-Tiering has NO retrieval fee**: Unlike Standard-IA and Glacier classes. Only a monitoring fee. If the question says "no retrieval fee" + "unknown access pattern" → Intelligent-Tiering.

---

## 7. Cheat Sheet

### Must-Know Facts for Exam Day

| Fact | Value |
|------|-------|
| S3 durability (all multi-AZ classes) | **11 9s (99.999999999%)** |
| S3 Standard availability | **99.99%** |
| One Zone-IA availability | **99.5%** |
| Max object size | **5 TB** |
| Max single PUT | **5 GB** |
| Multipart upload required | >**5 GB** |
| Buckets per account | **100** (soft, up to 1,000) |
| Bucket policy max size | **20 KB** |
| Request rate per prefix | **5,500 GET/s**, **3,500 PUT/s** |
| Lifecycle rules per bucket | **1,000** |
| Standard-IA min duration | **30 days** |
| Glacier Instant Retrieval min duration | **90 days** |
| Glacier Deep Archive min duration | **180 days** |
| IA / Glacier Instant min object size | **128 KB** |
| Glacier Flexible Expedited retrieval | **1–5 minutes** |
| Deep Archive Standard retrieval | **12 hours** |
| CRR replicates existing objects? | **No** (use Batch Replication) |
| Permanent deletes replicated? | **Never** |
| Default encryption (since Jan 2023) | **SSE-S3 (AES-256)** |
| Object Lock mode that root can't override | **Compliance** |

### Key Differentiators Table

| Standard-IA | One Zone-IA | Glacier Instant | Glacier Flexible | Deep Archive |
|-------------|-------------|-----------------|------------------|-------------|
| Multi-AZ | Single AZ | Multi-AZ | Multi-AZ | Multi-AZ |
| 30-day min | 30-day min | 90-day min | 90-day min | 180-day min |
| ms retrieval | ms retrieval | ms retrieval | min–hours | 12–48 hours |
| $0.0125/GB | $0.01/GB | $0.004/GB | $0.0036/GB | $0.00099/GB |

| CRR | SRR |
|-----|-----|
| Cross-Region | Same-Region |
| DR, compliance, latency | Log aggregation, cross-account backup, compliance copies |
| Data transfer cost | No cross-Region transfer cost |

| Object Lock Compliance | Object Lock Governance |
|----------------------|---------------------|
| Nobody can delete/shorten | Users with bypass permission can override |
| For regulatory (SEC, FINRA) | For internal policies |
| Truly immutable | Flexible immutability |

| Bucket Policy | Access Point Policy |
|--------------|-------------------|
| One per bucket, 20 KB max | One per access point, 20 KB each |
| All traffic through one policy | Segmented by team/app/VPC |
| Becomes unwieldy at scale | Scales to thousands of access points |

### Decision Flowchart

```
Question says "unknown/changing access patterns" + "no retrieval fees"?
  → S3 Intelligent-Tiering

Question says "infrequent access" + "millisecond retrieval" + "multi-AZ"?
  → S3 Standard-IA (if accessed monthly+) or Glacier Instant Retrieval (if accessed quarterly)

Question says "infrequent access" + "recreatable data"?
  → S3 One Zone-IA

Question says "archive" + "must access in milliseconds"?
  → S3 Glacier Instant Retrieval (NOT Glacier Flexible Retrieval)

Question says "long-term archive" + "7+ years" + "cheapest"?
  → S3 Glacier Deep Archive

Question says "regulatory immutability" + "cannot delete"?
  → S3 Object Lock Compliance Mode

Question says "WORM" + "admin flexibility to override"?
  → S3 Object Lock Governance Mode

Question says "replicate existing objects"?
  → S3 Batch Replication

Question says "replication SLA" + "15 minutes"?
  → S3 Replication Time Control (RTC)

Question says "multi-Region active-active S3" + "single endpoint"?
  → S3 Multi-Region Access Points (MRAP) + bidirectional CRR

Question says "bucket policy too large" + "many teams"?
  → S3 Access Points

Question says "restrict S3 access to specific VPC"?
  → VPC Gateway Endpoint + endpoint policy OR Access Point with VPC restriction

Question says "transform data on read" + "redact PII"?
  → S3 Object Lambda Access Point

Question says "accelerate uploads from far away"?
  → S3 Transfer Acceleration

Question says "bulk operations on millions of objects"?
  → S3 Batch Operations

Question says "audit storage class / encryption across all buckets"?
  → S3 Inventory (per bucket) or S3 Storage Lens (org-wide)

Question says "SSE-KMS" + "replication failing"?
  → Enable KMS replication in replication rule + grant replication role KMS permissions

Question says "cross-account upload" + "bucket owner can't access objects"?
  → Enable Bucket Owner Enforced (S3 Object Ownership setting)

Question says "high request rate" + "performance"?
  → Distribute across multiple prefixes (5,500 GET/s per prefix)

Question says "prevent accidental permanent delete"?
  → Versioning + MFA Delete
```

---

*Generated for AWS SAP-C02 exam preparation. Last updated: 2026-04-26.*
