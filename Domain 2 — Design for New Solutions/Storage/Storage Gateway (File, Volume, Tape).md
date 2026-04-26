# AWS SAP-C02 Study Notes: Storage Gateway (File, Volume, Tape)

---

## 1. Core Concepts & Theory

### AWS Storage Gateway Overview

AWS Storage Gateway is a **hybrid cloud storage service** that provides on-premises applications with virtually unlimited cloud storage. It deploys a **gateway appliance** (VM or hardware) in your data center that connects local applications to AWS storage services using standard protocols (NFS, SMB, iSCSI, iSCSI-VTL).

#### The Core Idea

```
On-Premises Applications
  → Standard storage protocols (NFS, SMB, iSCSI)
    → Storage Gateway Appliance (on-prem VM or hardware)
      → AWS Cloud (S3, EBS snapshots, S3 Glacier)
```

The gateway bridges on-premises environments with AWS cloud storage, providing **local caching** for low-latency access while storing primary data in AWS.

---

### Gateway Types

AWS Storage Gateway has **four gateway types** (three major categories):

| Gateway Type | Protocol | Backend Storage | Local Cache | Use Case |
|-------------|----------|----------------|-------------|----------|
| **S3 File Gateway** | NFS, SMB | **S3** (Standard, IA, One Zone-IA, Intelligent-Tiering) | Yes (recently accessed files) | File shares backed by S3 |
| **FSx File Gateway** | SMB | **FSx for Windows File Server** | Yes (recently accessed files) | Windows file shares with local cache |
| **Volume Gateway — Cached** | iSCSI | **S3** (primary data in S3, cache on-prem) | Yes (frequently accessed data) | Primary data in cloud, local cache for low latency |
| **Volume Gateway — Stored** | iSCSI | **S3** (async backup of on-prem data) | No (full data set on-prem) | Primary data on-prem, async snapshots to AWS |
| **Tape Gateway** | iSCSI-VTL | **S3 + S3 Glacier/Deep Archive** | Yes (recently accessed tapes) | Virtual tape library replacing physical tape infrastructure |

---

### S3 File Gateway

Exposes an **S3 bucket as an NFS or SMB file share** to on-premises applications.

#### Architecture

```
On-Prem Application → NFS/SMB → S3 File Gateway (local cache) → S3 Bucket
```

#### Key Details

| Property | Detail |
|----------|--------|
| **Protocol** | NFS v3/v4.1, SMB 2/3 |
| **Backend** | Amazon S3 (objects). Each file = one S3 object. File metadata stored in S3 object metadata |
| **Cache** | Local disk cache for recently accessed files — low-latency reads |
| **Object mapping** | Files map 1:1 to S3 objects. File path = S3 key (`/share/dir/file.txt` → `s3://bucket/dir/file.txt`) |
| **S3 storage classes** | Can target: S3 Standard, Standard-IA, One Zone-IA, Intelligent-Tiering. Lifecycle policies on the bucket apply normally |
| **Notifications** | **RefreshCache API** — must be called (or automated via EventBridge/Lambda) when objects are added to S3 outside the gateway (e.g., by another gateway or direct S3 upload) |
| **Access** | NFS: POSIX permissions. SMB: Active Directory integration for Windows ACL-like access |
| **Max file size** | **5 TB** (S3 object size limit) |
| **Max objects per share** | No hard limit (but performance degrades beyond ~10 million objects per prefix — use multiple shares/prefixes) |

#### Important Behaviors

- **Write-through**: Files written to the gateway are uploaded to S3 **asynchronously** but confirmed to the client only after data is durable in the cache.
- **Read-through cache**: First read fetches from S3; subsequent reads served from local cache.
- Objects modified directly in S3 are **NOT automatically visible** to NFS/SMB clients. You must call `RefreshCache` or use automated refresh (CloudWatch Events / EventBridge + Lambda).
- **S3 lifecycle policies** work normally on objects created by the gateway — transition to Glacier, expire, etc.
- **S3 versioning, replication, Object Lock** all work — the gateway creates standard S3 objects.
- **Cannot** directly access Glacier-transitioned objects via the file gateway — must restore in S3 first.

---

### FSx File Gateway

Provides a **local cache** in front of **FSx for Windows File Server** for on-premises Windows clients.

#### Architecture

```
On-Prem Windows Clients → SMB → FSx File Gateway (local cache) → FSx for Windows File Server (in AWS)
```

#### Key Details

| Property | Detail |
|----------|--------|
| **Protocol** | SMB |
| **Backend** | FSx for Windows File Server (not S3) |
| **Cache** | Local cache for frequently accessed files |
| **AD integration** | Required — same AD as the FSx for Windows file system |
| **DFS Namespace** | Supported — FSx File Gateway can participate in DFS |
| **Use case** | On-prem users need low-latency access to FSx for Windows shares without relying solely on Direct Connect/VPN bandwidth |
| **vs. direct FSx access** | Direct access over VPN/DX works, but FSx File Gateway adds local caching = lower latency for frequently accessed files |

#### When to Use FSx File Gateway vs S3 File Gateway

| Need | Gateway |
|------|---------|
| Files backed by S3 objects, Linux or Windows | S3 File Gateway |
| Windows file shares with AD/NTFS/DFS backed by FSx | FSx File Gateway |
| Need to access S3 features (lifecycle, versioning, analytics) | S3 File Gateway |
| Need Windows-native features (shadow copies, dedup, DFS) | FSx File Gateway |

---

### Volume Gateway

Presents cloud-backed **iSCSI block storage volumes** to on-premises applications. Two modes:

#### Cached Volumes

```
On-Prem Application → iSCSI → Volume Gateway (cache disk) → S3 (primary data)
                                                            → EBS Snapshots (backup)
```

| Property | Detail |
|----------|--------|
| **Data location** | **Primary data in S3**. Frequently accessed data cached locally |
| **Volume size** | Up to **32 TiB per volume** |
| **Volumes per gateway** | Up to **32 cached volumes** (total: **1 PiB per gateway**) |
| **Snapshots** | EBS snapshots stored in S3. Can create EBS volumes from snapshots in AWS |
| **Use case** | Need full data set available but most data is infrequently accessed. Local cache provides low-latency access for active subset |
| **Low on-prem storage** | Only need enough local disk for cache — not the entire data set |

#### Stored Volumes

```
On-Prem Application → iSCSI → Volume Gateway (full data on-prem) → S3 (async snapshots)
                                                                   → EBS Snapshots
```

| Property | Detail |
|----------|--------|
| **Data location** | **Full data set stored on-premises**. Async backup (snapshots) to S3 |
| **Volume size** | Up to **16 TiB per volume** |
| **Volumes per gateway** | Up to **32 stored volumes** (total: **512 TiB per gateway**) |
| **Snapshots** | Point-in-time EBS snapshots uploaded asynchronously to S3 |
| **Use case** | Need full data set on-prem for lowest latency. AWS used for disaster recovery (snapshots) |
| **Full local storage** | Must have enough local disk for the entire data set |

#### Volume Gateway Snapshots

- Both cached and stored volumes create **EBS snapshots** (stored in S3, managed by AWS).
- Snapshots can be used to:
  - **Create new EBS volumes** in AWS (attach to EC2) — useful for DR or migration.
  - **Create new Volume Gateway volumes** — clone on-prem volumes.
- Snapshots are **incremental**.
- Scheduled snapshots configurable via the console or API.

---

### Tape Gateway (Virtual Tape Library — VTL)

Replaces **physical tape infrastructure** with cloud-backed virtual tapes. Existing backup software (Veeam, Veritas NetBackup, Commvault, Microsoft DPM, etc.) works unchanged.

#### Architecture

```
Backup Application (Veeam, NetBackup) → iSCSI-VTL → Tape Gateway (local cache)
  → Virtual Tapes in S3 (tape shelf)
  → Archived Tapes in S3 Glacier Flexible Retrieval or Deep Archive (tape vault)
```

#### Key Details

| Property | Detail |
|----------|--------|
| **Protocol** | iSCSI-VTL (Virtual Tape Library) |
| **Virtual tape size** | 100 GiB to **5 TiB** per tape |
| **Max tapes per gateway** | **1,500 virtual tapes** |
| **Max total storage** | **1 PiB** per gateway (across all tapes) |
| **Tape shelf** | Active tapes stored in S3 (accessible for read/write) |
| **Tape archive (Glacier)** | Ejected/archived tapes move to **S3 Glacier Flexible Retrieval** — retrieval: **3–5 hours** |
| **Tape archive (Deep Archive)** | Alternatively, archive to **S3 Glacier Deep Archive** — retrieval: **12 hours** |
| **Tape retrieval** | To read an archived tape, issue a retrieval request → tape returns to the VTL shelf (S3) → backup software reads it |
| **Backup software compatibility** | Works with all major backup applications (transparent VTL interface) |
| **WORM tapes** | Supports **tape retention lock** — WORM-compliant tapes that cannot be deleted during retention period |

#### Virtual Tape Lifecycle

```
Create tape → Write data (backup application) → Eject tape
  → Tape moves to S3 Glacier Flexible Retrieval or Deep Archive
  → Retrieve archived tape → Tape returns to VTL shelf (S3)
  → Read data (restore operation)
```

---

### Gateway Deployment Options

| Option | Description | Use Case |
|--------|-------------|----------|
| **VM on-premises** | VMware ESXi, Microsoft Hyper-V, Linux KVM | Standard on-premises deployment |
| **Amazon EC2** | Gateway VM running in AWS | Cloud-to-cloud gateway, testing, or migration staging |
| **Hardware appliance** | AWS-provided Dell PowerEdge hardware | Sites where VM infrastructure is unavailable or impractical |

#### Hardware Appliance

- Pre-configured Dell PowerEdge server ordered from AWS.
- Designed for **branch offices** or **small data centers** without existing VM infrastructure.
- Supports all gateway types (File, Volume, Tape).
- Includes local SSDs for cache.

---

### Gateway Local Disk Requirements

| Gateway Type | Upload Buffer | Cache Storage |
|-------------|--------------|--------------|
| **S3 File Gateway** | N/A | **≥150 GiB** recommended. Larger cache = more data served locally |
| **FSx File Gateway** | N/A | **≥150 GiB** recommended |
| **Volume Gateway (Cached)** | Yes (upload buffer) | Yes (cache for frequently accessed data) |
| **Volume Gateway (Stored)** | Yes (upload buffer) | Full data set stored locally |
| **Tape Gateway** | Yes (upload buffer) | Yes (tape cache) |

**Upload buffer**: Temporary staging area for data being uploaded to AWS. If network is slow, upload buffer prevents application blocking.

**Cache**: Serves frequently accessed data locally. Sizing affects performance — undersized cache causes frequent round-trips to S3/AWS.

---

### Default Limits Worth Memorizing

| Resource | Limit |
|----------|-------|
| Cached volume max size | **32 TiB** per volume |
| Stored volume max size | **16 TiB** per volume |
| Volumes per gateway (cached) | **32** (1 PiB total) |
| Volumes per gateway (stored) | **32** (512 TiB total) |
| Virtual tape size | 100 GiB – **5 TiB** |
| Virtual tapes per gateway | **1,500** |
| Max tape storage per gateway | **1 PiB** |
| File share max objects per share | No hard limit (performance degrades >10M per prefix) |
| File max size (S3 File Gateway) | **5 TiB** (S3 object size limit) |
| Gateways per account per Region | **50** |
| Max bandwidth per gateway | Depends on network/hardware — typically 100–500 MB/s to S3 |

---

## 2. Design Patterns & Best Practices

### When to Use Which Gateway Type

| Scenario | Gateway Type | Why |
|----------|------------|-----|
| On-prem Linux/Windows apps need file access to S3 | **S3 File Gateway** | NFS/SMB interface, objects stored in S3 |
| On-prem Windows apps need cached access to FSx for Windows | **FSx File Gateway** | SMB, AD, DFS, local cache for FSx |
| On-prem apps need iSCSI block storage, most data in cloud | **Volume Gateway — Cached** | Low on-prem storage footprint, data in S3 |
| On-prem apps need iSCSI block storage, all data local, cloud DR | **Volume Gateway — Stored** | Full data on-prem, async snapshots for DR |
| Replace physical tape with cloud-backed VTL | **Tape Gateway** | Drop-in VTL replacement, Glacier archival |
| Migrate on-prem data to S3 incrementally | **S3 File Gateway** | Continuous sync as files are written |
| Lift-and-shift block storage to AWS | **Volume Gateway (either)** | EBS snapshots from volumes → create EBS volumes in AWS → attach to EC2 |
| Regulatory tape retention (WORM) | **Tape Gateway** with tape retention lock | Compliance-ready WORM tapes |

### Anti-Patterns

| Anti-Pattern | Why It's Wrong | Correct Approach |
|-------------|---------------|------------------|
| S3 File Gateway for HPC data processing | Gateway throughput limited by network/appliance. Not designed for HPC | FSx for Lustre with S3 integration |
| Volume Gateway for primary cloud-native workloads | Volume Gateway is for hybrid bridging, not cloud-native | Use EBS directly on EC2 |
| Tape Gateway for fast, frequent restores | Archived tapes take 3–12 hours to retrieve from Glacier | Use Volume Gateway or S3 File Gateway for frequent access |
| S3 File Gateway to access Glacier-transitioned objects | Gateway cannot read Glacier objects directly | Restore objects to S3 first, then access via gateway |
| Storage Gateway for one-time large data migration | Gateway is for ongoing hybrid. One-time migration is better served by DataSync, Snow Family, or Transfer Family | AWS DataSync for online migration, Snow Family for offline |
| Using many file shares with tens of millions of objects each | Metadata lookup slows — gateway performance degrades | Use multiple shares with fewer objects per prefix; or use DataSync for bulk migration |

### When to Use Storage Gateway vs Alternatives

| Need | Storage Gateway | Alternative | When to Use Alternative |
|------|----------------|-------------|----------------------|
| Ongoing hybrid file access to S3 | S3 File Gateway | N/A | Gateway is the right choice |
| One-time migration to S3 | Can work, but slow | **AWS DataSync** | DataSync is 10× faster, purpose-built for migration |
| Offline migration (TB–PB scale) | Not suitable | **AWS Snow Family** | No network bandwidth or unreliable connectivity |
| On-prem access to FSx for Windows | FSx File Gateway | Direct access via VPN/DX | Direct access if bandwidth is sufficient; gateway adds caching |
| HPC processing of S3 data | Not suitable | **FSx for Lustre** | Lustre provides hundreds of GB/s — gateway cannot |
| Cloud-native block storage | Not ideal | **EBS** | EBS for cloud-native; gateway for hybrid bridge |
| Real-time data transfer and replication | Limited by upload buffer/network | **AWS DataSync** | DataSync has scheduling, validation, bandwidth throttling |

### Common Architectural Patterns

#### Hybrid File Storage (Most Common Exam Pattern)
```
Branch Office A → S3 File Gateway (NFS/SMB) → S3 Bucket (us-east-1)
Branch Office B → S3 File Gateway (NFS/SMB) → Same S3 Bucket
  → RefreshCache triggered by EventBridge when files are uploaded
  → S3 Lifecycle: Standard → IA (30d) → Glacier (90d)
  → S3 Replication: CRR to eu-west-1 for DR
```

#### Disaster Recovery with Volume Gateway
```
On-Prem Production → Volume Gateway (Stored) → Async EBS Snapshots
  → Scheduled snapshots (hourly)
  → Cross-Region snapshot copy (DLM)
  → DR: Restore EBS volumes from snapshots → Attach to EC2 in DR Region
  → Or: Create new Volume Gateway from snapshot (clone on-prem)
```

#### Tape Replacement
```
On-Prem Backup Software (Veeam/NetBackup)
  → Tape Gateway (iSCSI-VTL)
    → Active tapes: S3 (VTL shelf)
    → Archive tapes: S3 Glacier Deep Archive (tape vault)
    → Tape retention lock for WORM compliance
  → Cost: ~$0.00099/GB/month (Deep Archive) vs $500+/month for physical tape library
```

#### Migration Pipeline: On-Prem to AWS
```
Phase 1: S3 File Gateway → Data flows to S3 as files are accessed/written
Phase 2: AWS DataSync → Bulk migration of remaining data
Phase 3: Cutover → Applications point directly to S3/EFS/FSx
Phase 4: Decommission gateway
```

#### Multi-Site File Sharing via S3
```
Site A (California) → S3 File Gateway → S3 Bucket
Site B (New York)   → S3 File Gateway → Same S3 Bucket
Site C (London)     → S3 File Gateway → Same S3 Bucket

All sites share data via S3 bucket as the single source of truth
RefreshCache automation ensures consistency across gateways
```

### Well-Architected Framework Alignment

**Reliability**
- Volume Gateway stored mode keeps full data on-prem — survives AWS outage.
- Volume Gateway cached mode depends on S3 — local cache provides partial resilience.
- Use Volume Gateway snapshots for DR — restore as EBS volumes in AWS.
- S3 File Gateway benefits from S3's 11 9s durability.
- Monitor gateway health via CloudWatch (CachePercentUsed, UploadBufferPercentUsed).

**Security**
- All data encrypted in transit (SSL/TLS) and at rest (SSE-S3 or SSE-KMS).
- SMB shares support AD integration for access control.
- NFS shares use IP-based access control.
- Deploy gateway in private subnet; control access via security groups.
- Use VPC endpoint for S3 to keep traffic off the internet.

**Performance**
- Size the cache appropriately — undersized cache = poor read performance.
- Size the upload buffer — undersized buffer = write throttling.
- Use Direct Connect for dedicated bandwidth between on-prem and AWS.
- Consider read replicas: multiple S3 File Gateways reading from the same S3 bucket.
- NFS performs better than SMB for large file transfers (less protocol overhead).

**Cost Optimization**
- Tape Gateway with Glacier Deep Archive: ~$1/TB/month — far cheaper than physical tape.
- S3 File Gateway + S3 lifecycle policies: automatic transition to IA/Glacier.
- Volume Gateway cached: less on-prem storage needed than stored mode.
- Use S3 Intelligent-Tiering via File Gateway for unknown access patterns.
- Decommission gateways after migration is complete.

**Operational Excellence**
- Monitor gateway health: CacheHitPercent, CachePercentUsed, UploadBufferPercentUsed, CloudBytesUploaded.
- Set CloudWatch alarms on CachePercentUsed > 80% (need larger cache or faster upload).
- Automate RefreshCache calls for multi-site S3 File Gateway deployments.
- Use AWS Backup for Volume Gateway snapshot management.

---

## 3. Security & Compliance

### Access Control

#### S3 File Gateway
- **NFS**: Client access controlled by IP address in the file share configuration. No user-level authentication for NFS (standard NFS limitation).
- **SMB**: Active Directory (AWS Managed AD or on-prem AD via AD Connector). Users authenticate via AD, and file-level permissions apply.
- **S3 bucket policy**: Controls who can access the objects in S3 (independent of gateway permissions).
- **IAM role**: The gateway assumes an IAM role to read/write to S3. This role must have appropriate S3 permissions.

#### FSx File Gateway
- **SMB**: Active Directory (same AD as FSx for Windows). Full Windows ACL support.
- **IAM role**: Gateway needs permissions to access FSx.

#### Volume Gateway
- **iSCSI**: CHAP (Challenge-Handshake Authentication Protocol) authentication between the initiator and gateway.
- **IAM role**: Gateway role needs permissions for S3 (volume data) and EBS (snapshots).

#### Tape Gateway
- **iSCSI-VTL**: CHAP authentication.
- **IAM role**: Gateway role needs S3 and Glacier permissions.

### Encryption

| Layer | Mechanism |
|-------|-----------|
| **In transit (on-prem → gateway)** | Application-level (NFS/SMB/iSCSI). SMB 3.0 supports encryption. NFS and iSCSI are **not encrypted by default** — use IPsec VPN if needed |
| **In transit (gateway → AWS)** | **SSL/TLS** (HTTPS). All data uploaded to AWS is encrypted in transit |
| **At rest (S3)** | **SSE-S3** (default) or **SSE-KMS** (configurable per file share/volume) |
| **At rest (local cache)** | **Not encrypted by default** by the gateway. Use host-level encryption (OS/hypervisor) or encrypted disks |
| **At rest (EBS snapshots)** | Encrypted with KMS if the volume was configured with KMS encryption |
| **At rest (Glacier)** | Encrypted by default (S3 Glacier encryption) |

**Exam critical:** Local cache/upload buffer data on the gateway appliance is **NOT encrypted at rest by the gateway itself**. You must rely on the underlying hypervisor/hardware encryption. Questions may test this nuance.

### Logging & Monitoring

| Tool | What It Captures |
|------|-----------------|
| **CloudWatch Metrics** | CacheHitPercent, CachePercentUsed, CachePercentDirty, UploadBufferPercentUsed, CloudBytesUploaded, CloudBytesDownloaded, ReadBytes, WriteBytes, TotalCacheSize |
| **CloudWatch Alarms** | Critical: CachePercentUsed > 80%, UploadBufferPercentUsed > 80% |
| **CloudTrail** | Gateway API calls (CreateGateway, CreateNFSFileShare, CreateSMBFileShare, etc.) |
| **S3 access logs / CloudTrail data events** | Track S3-level access to objects created by the gateway |
| **Health notifications** | Gateway sends health notifications to CloudWatch Events / EventBridge |
| **Audit log (SMB)** | S3 File Gateway supports file access audit logging for SMB shares → CloudWatch Logs |

### Compliance Patterns

- **WORM / Regulatory retention**: Tape Gateway with **tape retention lock** — tapes cannot be deleted during the retention period. Archived to Glacier for long-term compliance.
- **HIPAA / PCI**: Use SSE-KMS encryption, CHAP authentication, VPN/DX for transit, CloudTrail for audit.
- **Data sovereignty**: Gateway sends data to a specific AWS Region. Use S3 bucket policies and SCPs to enforce Region restrictions.
- **SEC 17a-4**: Tape Gateway with retention lock + Glacier Deep Archive. Alternatively, S3 File Gateway → S3 Object Lock (Compliance mode).

---

## 4. Cost Optimization

### Pricing Model

Storage Gateway pricing has multiple components:

| Component | Cost |
|-----------|------|
| **Gateway usage** | S3 File/FSx File: Free. Volume/Tape: **$0.01/GB/month** for data stored via gateway |
| **S3 storage** | Standard S3 pricing for objects created by File Gateway |
| **EBS snapshots** | Standard EBS snapshot pricing ($0.05/GB/month) for Volume Gateway snapshots |
| **Tape storage (VTL)** | $0.024/GB/month (virtual tapes in S3) |
| **Tape archive (Glacier)** | $0.0036/GB/month (Glacier Flexible Retrieval) or $0.00099/GB/month (Deep Archive) |
| **Tape retrieval** | $0.01/GB (Glacier Flexible) or $0.02/GB (Deep Archive) |
| **Data transfer** | Standard AWS data transfer pricing (in is free; out to internet is charged) |
| **Hardware appliance** | ~$12,000–$20,000 one-time purchase |

#### Free Tier

- **100 GB/month free** for the first 12 months of Volume/Tape Gateway usage.
- S3 File Gateway and FSx File Gateway have no per-GB gateway charge (you pay only for underlying S3/FSx storage).

### Cost-Saving Strategies

1. **S3 Lifecycle policies with File Gateway**: Objects created by the gateway follow normal S3 lifecycle rules. Transition to IA (30d) → Glacier (90d) → Deep Archive (365d) for massive savings.

2. **Tape Gateway + Deep Archive**: Replace physical tapes at ~$1/TB/month vs hundreds of dollars/month for tape library maintenance.

3. **Cached Volume instead of Stored Volume**: Less on-prem storage needed (only cache, not full data set). On-prem storage is expensive.

4. **S3 Intelligent-Tiering via File Gateway**: Set the file share to target S3 Intelligent-Tiering — automatic cost optimization.

5. **Right-size cache**: Monitor CacheHitPercent. If >90%, cache may be larger than needed. If <70%, increase cache.

6. **Decommission after migration**: If using gateway only for migration, decommission it once data is in AWS. Ongoing gateway cost is unnecessary.

7. **Use DataSync for bulk migration instead of gateway**: DataSync is faster for large initial migration. Use gateway for ongoing hybrid access.

### Common Cost Traps

- **Tape retrieval from Deep Archive**: $0.02/GB. Retrieving 100 TB = $2,000. Plan retrieval needs before archiving.
- **Volume Gateway stored mode data growth**: All data on-prem + snapshots in AWS. Paying for both local storage and S3 snapshots.
- **Undersized upload buffer**: Causes upload retries and potential data staging issues. Don't skimp.
- **Forgotten gateway after migration**: Gateway VM continues consuming local resources (compute, storage, electricity) even if no longer needed.
- **S3 request costs from File Gateway**: Many small files = many S3 PUT/GET requests. S3 request charges can be significant at scale.
- **Cross-Region snapshot copy**: Volume Gateway snapshots copied cross-Region incur data transfer + storage costs.

### Cost Comparison: Gateway vs Alternatives

| Scenario | Storage Gateway Cost | Alternative | Alternative Cost |
|----------|---------------------|------------|-----------------|
| Ongoing hybrid NFS to S3 | Free (gateway) + S3 storage | No real alternative (DataSync is for migration, not ongoing NFS) | N/A |
| One-time migration | Slow + ongoing overhead | DataSync ($0.0125/GB transferred) | Faster, one-time cost |
| Tape replacement | ~$1/TB/month (Deep Archive) | Physical tape | $100–500+/month (hardware, media, offsite) |
| On-prem block storage DR | EBS snapshot cost | AWS Backup + EBS | Similar, but Backup adds cross-service management |

---

## 5. High Availability, Disaster Recovery & Resilience

### Gateway HA

The gateway appliance itself is a **single point of failure**. AWS provides several HA strategies:

| Strategy | Description |
|----------|-------------|
| **VMware HA** | Deploy gateway VM on VMware with HA enabled — auto-restart on host failure |
| **Multiple gateways** | Deploy two gateways to the same S3 bucket. Active-passive or active-active with load distribution |
| **Hardware appliance redundancy** | Deploy two hardware appliances for failover |
| **EC2 gateway** | Run gateway on EC2 for AWS-level HA (multi-AZ via ASG not supported — but can script failover) |

#### S3 File Gateway HA

- Multiple gateways can point to the **same S3 bucket**.
- If one gateway fails, redirect clients to another gateway.
- Must call `RefreshCache` on the surviving gateway to see latest state.
- Objects in S3 are durable (11 9s) — gateway failure doesn't lose data.

#### Volume Gateway DR

- **Stored mode**: Full data on-prem. Snapshots in AWS.
  - If on-prem fails: Create EBS volumes from snapshots → attach to EC2.
  - If AWS fails: Data is safe on-prem.
- **Cached mode**: Primary data in S3.
  - If on-prem fails: Deploy new gateway → point to same volumes in S3.
  - If AWS fails: Local cache has frequently accessed data only. Full data access lost until AWS recovers.

#### Tape Gateway DR

- Virtual tapes in S3 (and archived tapes in Glacier) are durable.
- If gateway fails: Deploy new Tape Gateway → existing tapes are accessible.
- Tape data is immutable once written — inherently durable.

### DR Patterns

| Pattern | Gateway Type | Implementation | RPO | RTO |
|---------|-------------|---------------|-----|-----|
| **File DR via S3** | S3 File Gateway | S3 CRR to DR Region. Deploy new gateway in DR pointing to replicated bucket | Minutes (CRR lag) | Hours (deploy new gateway) |
| **Block DR via snapshots** | Volume Gateway | EBS snapshots copied cross-Region. In DR: create EBS volumes from snapshots → attach to EC2 | Hours (snapshot frequency) | Hours (restore + mount) |
| **Tape DR** | Tape Gateway | Tapes in S3/Glacier are Regional but durable. Deploy new gateway in DR Region | Near-zero (tapes in S3) | Hours (deploy gateway + retrieve tapes) |
| **Volume cloud migration** | Volume Gateway | Create EBS volume from Volume Gateway snapshot → attach to EC2 → migrate application to AWS | N/A (migration) | Hours |

### Gateway to EC2 Migration Path (Volume Gateway)

This is a **frequently tested exam pattern**:

```
1. On-prem Volume Gateway (Stored) → create snapshot
2. Snapshot stored as EBS snapshot in S3
3. Create new EBS volume from snapshot (in any AZ in the Region)
4. Attach EBS volume to EC2 instance
5. Application now runs fully in AWS on EBS
```

This is the primary **lift-and-shift** path for block storage workloads from on-prem to AWS.

---

## 6. Exam-Focused Section

### Straightforward Questions (7 examples)

**Q1:** A company wants to give on-premises Linux applications NFS access to S3. Which service should they use?
**A:** **S3 File Gateway.** It presents S3 buckets as NFS (or SMB) file shares with local caching.

**Q2:** Which Storage Gateway type stores the complete data set on-premises and asynchronously backs up to AWS?
**A:** **Volume Gateway — Stored mode.** Full data on-prem, async EBS snapshots to S3.

**Q3:** A company wants to replace its physical tape library with cloud-backed virtual tapes. Existing backup software must continue to work unchanged. Which service?
**A:** **Tape Gateway.** Provides an iSCSI-VTL interface compatible with existing backup software. Tapes archive to S3 Glacier.

**Q4:** What is the maximum size of a single virtual tape in Tape Gateway?
**A:** **5 TiB.**

**Q5:** How does data written through S3 File Gateway appear in S3?
**A:** Each file maps **1:1 to an S3 object**. File metadata is stored as S3 object metadata. File path becomes the S3 key.

**Q6:** An on-premises application uses Volume Gateway with cached volumes. Where is the primary copy of the data?
**A:** **In S3.** Cached volumes store primary data in S3 with a local cache for frequently accessed data.

**Q7:** Can S3 File Gateway read objects that have been transitioned to S3 Glacier via a lifecycle policy?
**A:** **No.** Glacier objects must be **restored to S3** first before the File Gateway can access them.

---

### Tricky / Scenario-Based Questions (7 examples)

**Q1:** A company has two offices (New York and London) that need to share files. Files are written in one office and read by both. They want to use S3 as the backend and need low-latency access at both sites. After a file is written in New York, London users report they don't see the new file for hours. What is the problem and fix?

**A:** The London S3 File Gateway's **cache metadata is stale**. S3 File Gateway doesn't automatically detect objects added to S3 by another gateway.
- **Fix:** Automate **RefreshCache API calls** using S3 Event Notifications → EventBridge → Lambda → RefreshCache on the London gateway whenever objects are written to the bucket.
- **Why not just "it's S3 eventual consistency"?** S3 has strong read-after-write consistency since 2020. The issue is the gateway's local metadata cache, not S3.
- **Key phrases:** "two sites" + "S3 File Gateway" + "don't see new files" → RefreshCache.

**Q2:** A company needs to migrate 500 TB of block storage data from on-premises to AWS EC2. The data is currently on a SAN. They want to minimize downtime during cutover. Which approach should they use?

**A:** **Volume Gateway (Stored mode)** → async snapshots to AWS → EBS volumes from snapshots → attach to EC2 → final sync → cutover.
- **Process:** (1) Deploy Volume Gateway stored mode on-prem. (2) Present iSCSI volumes to applications (replaces SAN). (3) Data is stored locally AND continuously snapshotted to AWS. (4) At cutover: take final snapshot, create EBS volumes, attach to EC2, redirect applications.
- **Why not DataSync?** DataSync is for file/object migration, not block storage (SAN/iSCSI).
- **Why not Snowball?** Snowball works but doesn't provide continuous sync — only a point-in-time copy. The question says "minimize downtime" which requires the final delta to be small.
- **Key phrases:** "block storage" + "SAN" + "migrate to EC2" + "minimize downtime" → Volume Gateway stored → snapshots → EBS.

**Q3:** A company uses S3 File Gateway with SMB shares backed by an S3 bucket. They configure S3 lifecycle to transition objects to S3 Glacier after 90 days. After 90 days, users try to open old files through the gateway and get errors. What is wrong?

**A:** **S3 File Gateway cannot read objects in S3 Glacier.** The lifecycle policy transitioned the objects to Glacier, making them inaccessible via the gateway.
- **Fix:** (1) Change lifecycle to transition to **S3 Standard-IA or Intelligent-Tiering** instead of Glacier (maintains instant access). (2) Or if Glacier is required for cost, use **S3 Glacier Instant Retrieval** (millisecond access — but verify gateway compatibility). (3) Or implement a process to restore objects from Glacier before accessing via gateway.
- **Key phrases:** "File Gateway" + "lifecycle to Glacier" + "cannot access old files" → Gateway can't read Glacier objects.

**Q4:** A company has on-premises applications using iSCSI storage. They want to move the primary data to AWS while maintaining low-latency access for the most frequently used data on-premises. On-prem storage capacity is limited. Which gateway mode?

**A:** **Volume Gateway — Cached mode.**
- **Why cached?** Primary data in S3 (unlimited). Local cache stores only frequently accessed data = low on-prem storage footprint.
- **Why not stored?** Stored mode requires the full data set on-prem. The question says "on-prem storage capacity is limited."
- **Key phrases:** "iSCSI" + "primary data in cloud" + "low-latency for frequent data" + "limited on-prem storage" → Volume Gateway Cached.

**Q5:** A compliance officer requires that backup tapes cannot be deleted for 7 years. The company currently uses Tape Gateway. How can they enforce this?

**A:** Enable **tape retention lock** on virtual tapes.
- **What it does:** Sets a retention period (up to decades). Tapes cannot be deleted until the retention period expires. This provides WORM compliance for virtual tapes.
- **Storage tier:** Archive tapes to **S3 Glacier Deep Archive** for lowest cost over 7 years (~$1/TB/month).
- **Why not S3 Object Lock?** Tape Gateway manages tapes internally — you don't directly access S3 objects. Tape retention lock is the VTL-specific WORM mechanism.
- **Key phrases:** "tape" + "cannot be deleted" + "7 years" + "compliance" → Tape retention lock + Deep Archive.

**Q6:** A company uses Volume Gateway (Cached) with 50 TB of data. During an on-premises disaster, the gateway hardware is destroyed. The local cache is lost. Can they recover their data? How?

**A:** **Yes.** The primary data is in **S3** (cached volumes store primary data in S3, not on-prem). 
- **Recovery:** (1) Deploy a new Volume Gateway (VM or hardware). (2) Attach existing cached volumes to the new gateway. The cache will rebuild as data is accessed. OR (3) Create EBS volumes from the latest snapshots → attach to EC2 instances in AWS.
- **What's lost?** Only the local cache contents (will be rebuilt). No data loss because primary data is in S3.
- **What if it were Stored mode?** Then the full data set would be on-prem — and destroyed. You'd recover from the latest EBS snapshot (with RPO = time since last snapshot).
- **Key phrases:** "cached mode" + "on-prem disaster" + "data recovery" → Data safe in S3, deploy new gateway.

**Q7:** A company is evaluating S3 File Gateway vs AWS DataSync for an ongoing need to sync on-premises files to S3 daily. Files are written by applications using NFS. After sync, applications continue to access the files locally. Which service is more appropriate?

**A:** **S3 File Gateway.**
- **Why?** File Gateway provides an **ongoing NFS mount** that applications use continuously. Files written locally are automatically uploaded to S3. Local cache provides continued low-latency read access.
- **Why not DataSync?** DataSync is a **data transfer service** — it runs as scheduled/one-time jobs. It doesn't provide a persistent NFS mount for applications. After a DataSync job, applications would need a separate NFS server for local access.
- **When IS DataSync better?** For one-time migrations, periodic bulk syncs where applications don't need ongoing local NFS access, or migrating between AWS storage services.
- **Key phrases:** "ongoing" + "applications use NFS" + "continue to access locally" → S3 File Gateway (not DataSync).

---

### Common Exam Traps & Pitfalls

1. **Cached vs Stored Volume Gateway**: Cached = primary data in S3, cache on-prem. Stored = primary data on-prem, snapshots to S3. If the question mentions "limited on-prem storage," the answer is Cached. If it mentions "lowest latency for full data set," the answer is Stored.

2. **S3 File Gateway ≠ automatic Glacier access**: Objects transitioned to Glacier via lifecycle are NOT readable through the gateway. Must restore to S3 first.

3. **RefreshCache requirement**: When multiple gateways or external processes write to the same S3 bucket, other gateways do NOT automatically see new objects. Must call RefreshCache.

4. **File Gateway creates S3 OBJECTS, not EBS volumes**: This is different from Volume Gateway, which creates EBS snapshots. File Gateway data is accessible directly in S3 (can be queried by Athena, processed by Lambda, etc.).

5. **Volume Gateway creates EBS SNAPSHOTS, not S3 objects**: You can create EBS volumes from these snapshots (for EC2 migration or DR). You cannot access this data via S3 APIs.

6. **Tape Gateway → Glacier retrieval takes hours**: Expedited: N/A for Deep Archive. Standard: 3–5 hours (Flexible) or 12 hours (Deep Archive). Don't expect instant tape restores.

7. **Local cache is NOT encrypted at rest by the gateway**: Must use underlying OS/hypervisor encryption or encrypted disks. Questions may test this.

8. **Storage Gateway vs DataSync**: Gateway = ongoing hybrid storage with local access. DataSync = data transfer and migration tool. If the question mentions "ongoing NFS access on-prem," the answer is Gateway. If it says "migrate 100 TB once," the answer is DataSync.

9. **Storage Gateway vs Transfer Family**: Transfer Family provides SFTP/FTPS/FTP access to S3. Gateway provides NFS/SMB/iSCSI. Different protocols, different use cases.

10. **Hardware appliance for no-VM-infrastructure sites**: If the question mentions a remote office without VMware/Hyper-V, the answer is the hardware appliance.

11. **Volume Gateway stored → EBS migration path**: This is a frequently tested DR/migration pattern. Stored volumes → snapshots → EBS volumes → EC2. Know this flow cold.

12. **S3 File Gateway file size limit = 5 TB**: Because each file = one S3 object, and S3 object max = 5 TB.

13. **Cached volume max = 32 TiB; Stored volume max = 16 TiB**: Cached supports larger individual volumes because primary data is in S3 (not bounded by local disk).

14. **FSx File Gateway vs S3 File Gateway**: FSx File Gateway caches FSx for Windows (SMB, AD, NTFS). S3 File Gateway caches S3. Different backends, different features.

---

## 7. Cheat Sheet

### Must-Know Facts for Exam Day

| Fact | Value |
|------|-------|
| S3 File Gateway protocols | **NFS v3/v4.1, SMB 2/3** |
| S3 File Gateway backend | **S3** (1:1 file-to-object mapping) |
| S3 File Gateway max file size | **5 TB** (S3 object limit) |
| FSx File Gateway protocol | **SMB** |
| FSx File Gateway backend | **FSx for Windows File Server** |
| Volume Gateway — Cached data location | **S3** (primary), local cache (hot data) |
| Volume Gateway — Stored data location | **On-prem** (primary), async snapshots to S3 |
| Cached volume max size | **32 TiB** per volume |
| Stored volume max size | **16 TiB** per volume |
| Volumes per gateway | **32** |
| Tape max size | **5 TiB** |
| Tapes per gateway | **1,500** |
| Tape archive targets | **S3 Glacier Flexible Retrieval** or **S3 Glacier Deep Archive** |
| Glacier Flexible Retrieval time | **3–5 hours** (Standard) |
| Deep Archive retrieval time | **12 hours** (Standard) |
| Volume Gateway backup format | **EBS Snapshots** |
| Gateway encryption in transit to AWS | **SSL/TLS** |
| Gateway encryption at rest in S3 | **SSE-S3** or **SSE-KMS** |
| Local cache encryption | **Not encrypted by gateway** — use host-level encryption |
| RefreshCache needed? | **Yes** — when S3 objects modified outside the gateway |
| File Gateway reads Glacier objects? | **No** — must restore to S3 first |
| Tape retention lock | **WORM compliance for virtual tapes** |
| Hardware appliance | Dell PowerEdge for sites without VM infrastructure |

### Key Differentiators

| S3 File Gateway | FSx File Gateway |
|----------------|-----------------|
| Backend: S3 | Backend: FSx for Windows |
| NFS + SMB | SMB only |
| S3 features (lifecycle, versioning, analytics) | Windows features (DFS, dedup, Shadow Copies) |
| Any AD or no AD (NFS) | AD required |

| Volume Cached | Volume Stored |
|--------------|--------------|
| Primary data: S3 | Primary data: On-prem |
| Low on-prem storage | Full data on-prem |
| Max 32 TiB/volume | Max 16 TiB/volume |
| On-prem failure: data safe in S3 | On-prem failure: recover from snapshots (RPO = snapshot interval) |

| Storage Gateway | DataSync | Snow Family |
|----------------|----------|-------------|
| Ongoing hybrid access (NFS/SMB/iSCSI) | Data transfer/migration | Offline bulk migration |
| Local cache for low latency | Scheduled jobs, no persistent mount | Physical device, no network needed |
| Long-term coexistence | Short-term migration | One-time migration |

### Decision Flowchart

```
Question says "on-prem NFS/SMB access to S3" + "ongoing"?
  → S3 File Gateway

Question says "on-prem Windows file shares" + "AD" + "FSx backend"?
  → FSx File Gateway

Question says "on-prem iSCSI block storage" + "limited on-prem storage"?
  → Volume Gateway — Cached

Question says "on-prem iSCSI block storage" + "full data on-prem" + "DR to AWS"?
  → Volume Gateway — Stored

Question says "replace physical tape library" or "VTL"?
  → Tape Gateway

Question says "migrate block storage to EC2"?
  → Volume Gateway Stored → EBS Snapshots → EBS Volumes → EC2

Question says "WORM tapes" or "tape retention compliance"?
  → Tape Gateway with tape retention lock

Question says "on-prem users can't see files uploaded by another site"?
  → RefreshCache API on the S3 File Gateway

Question says "files transitioned to Glacier not accessible via gateway"?
  → Restore from Glacier to S3 first (or change lifecycle to avoid Glacier)

Question says "one-time migration" or "bulk data transfer"?
  → AWS DataSync (NOT Storage Gateway)

Question says "no VM infrastructure on-prem" + "need gateway"?
  → Storage Gateway Hardware Appliance

Question says "migrate on-prem data to S3" + "incrementally"?
  → S3 File Gateway (for ongoing) or DataSync (for scheduled jobs)

Question says "HPC" + "process S3 data"?
  → FSx for Lustre (NOT S3 File Gateway — insufficient throughput)

Question says "on-prem to S3" + "SFTP/FTPS"?
  → AWS Transfer Family (NOT Storage Gateway)
```

---

*Generated for AWS SAP-C02 exam preparation. Last updated: 2026-04-26.*
