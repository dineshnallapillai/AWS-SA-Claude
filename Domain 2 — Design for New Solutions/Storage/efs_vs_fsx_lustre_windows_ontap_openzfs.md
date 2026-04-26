# AWS SAP-C02 Study Notes: EFS vs FSx (Lustre, Windows, NetApp ONTAP, OpenZFS)

---

## 1. Core Concepts & Theory

### The AWS File Storage Landscape

AWS offers **five managed file storage services**. Choosing the right one is a high-frequency SAP-C02 topic.

| Service | Protocol | OS Support | Key Use Case |
|---------|----------|-----------|-------------|
| **Amazon EFS** | NFSv4.1 | Linux | Shared POSIX file system for Linux workloads |
| **Amazon FSx for Windows File Server** | SMB (+ NFS via ONTAP) | Windows (+ Linux via SMB) | Windows-native shared storage, Active Directory, DFS |
| **Amazon FSx for Lustre** | Lustre (POSIX) | Linux | HPC, ML training, high-throughput compute |
| **Amazon FSx for NetApp ONTAP** | NFS, SMB, iSCSI | Linux, Windows, macOS | Multi-protocol, enterprise features, hybrid cloud |
| **Amazon FSx for OpenZFS** | NFS | Linux, macOS, Windows (NFS client) | ZFS migration, low-latency NFS, snapshots, compression |

---

### Amazon EFS (Elastic File System)

Fully managed, serverless, elastic **NFS file system** for Linux workloads.

#### Architecture

- **Regional service** — data stored across **multiple AZs** automatically.
- Access via **mount targets** — one ENI per AZ in your VPC subnets.
- Supports **thousands of concurrent EC2/Fargate/Lambda connections**.
- Elastic — grows and shrinks automatically. **No capacity provisioning needed** (Standard). Provisioned throughput mode available.

#### Storage Classes

| Class | Availability | Cost (us-east-1) | Access |
|-------|-------------|-------------------|--------|
| **EFS Standard** | Multi-AZ | $0.30/GB/month | Immediate |
| **EFS Standard-IA** | Multi-AZ | $0.025/GB/month (+ $0.01/GB retrieval) | Immediate |
| **EFS One Zone** | Single AZ | $0.16/GB/month | Immediate |
| **EFS One Zone-IA** | Single AZ | $0.0133/GB/month (+ $0.01/GB retrieval) | Immediate |
| **EFS Archive** | Multi-AZ | $0.008/GB/month (+ $0.06/GB retrieval) | Immediate |

#### Performance Modes

| Mode | Latency | Throughput | Use Case |
|------|---------|-----------|----------|
| **General Purpose** (default) | Lowest (sub-ms) | Up to 10+ GB/s (elastic) | Web serving, CMS, home directories, dev environments |
| **Max I/O** | Slightly higher | Highest aggregate throughput | Large-scale HPC, big data analytics (legacy — Elastic Throughput mostly replaces this) |

#### Throughput Modes

| Mode | How It Works | Best For |
|------|-------------|----------|
| **Elastic** (default, recommended) | Automatically scales throughput. Pay per data transferred. Up to 10 GB/s read, 3 GB/s write | Unpredictable/spiky workloads |
| **Provisioned** | You specify throughput (1 MiB/s – 3+ GiB/s). Pay for provisioned amount | Steady, predictable throughput needs |
| **Bursting** | Throughput scales with storage size (50 MiB/s per TiB). Burst credits | Small file systems that occasionally need higher throughput (legacy) |

#### EFS Lifecycle Management

- **Intelligent-Tiering** automatically moves files between Standard → Standard-IA → Archive based on access patterns.
- Configurable: move to IA after 1/7/14/30/60/90 days without access.
- Move to Archive after 90/180/270/365 days without access.
- **First-byte latency** is the same across all classes — all are immediately accessible.

#### EFS + Lambda / Fargate

- Lambda can mount EFS via VPC (function must be VPC-attached).
- ECS/Fargate tasks can mount EFS as persistent shared storage.
- **This is the answer when Fargate needs persistent shared storage** (EBS is not supported on Fargate).

---

### Amazon FSx for Windows File Server

Fully managed **Windows-native** file system built on Windows Server. Provides **SMB protocol**, **NTFS**, and full **Active Directory integration**.

#### Key Features

| Feature | Detail |
|---------|--------|
| **Protocol** | SMB 2.0–3.1.1 |
| **File system** | NTFS |
| **AD integration** | AWS Managed Microsoft AD or self-managed AD. **Required** — no anonymous/local-user access |
| **DFS (Distributed File System)** | Supports DFS Namespaces for file share grouping across multiple FSx file systems |
| **Shadow Copies** | Windows VSS-based point-in-time snapshots accessible by end users (right-click → Previous Versions) |
| **Data deduplication** | Built-in Windows Server dedup — can save 50–80% for general-purpose file shares |
| **Quotas** | User-level storage quotas |
| **Access** | EC2 (Windows + Linux via SMB), WorkSpaces, AppStream 2.0, VMware Cloud on AWS |
| **On-premises access** | Via **VPN or Direct Connect** (private connectivity required) |
| **Storage type** | SSD (low-latency) or HDD (throughput-optimized, cheaper) |
| **Deployment** | Single-AZ or **Multi-AZ** (active/standby with automatic failover) |

#### Performance

| Metric | SSD | HDD |
|--------|-----|-----|
| Latency | Sub-millisecond | Single-digit ms |
| Max throughput | Up to 2 GB/s | Up to 12 GB/s (with in-memory caching) |
| Max IOPS | Hundreds of thousands | N/A (throughput-optimized) |

---

### Amazon FSx for Lustre

Fully managed **Lustre file system** — the world's most popular high-performance parallel file system. Designed for compute-intensive workloads.

#### Key Features

| Feature | Detail |
|---------|--------|
| **Protocol** | Lustre (POSIX-compatible) |
| **OS** | Linux only |
| **S3 integration** | **Native** — can link to an S3 bucket. Objects appear as files; files written back to S3 (lazy load + writeback) |
| **Performance** | Hundreds of GB/s throughput, millions of IOPS, sub-millisecond latency |
| **Deployment types** | **Scratch** (temporary, no replication, highest performance) and **Persistent** (data replicated within AZ, longer-term workloads) |
| **Compression** | LZ4 compression — reduces storage costs by up to 50% |

#### Deployment Types

| Type | Data Durability | Use Case | AZs |
|------|----------------|----------|-----|
| **Scratch** | NOT replicated. Data lost if file server fails | Short-term processing, temp compute (HPC burst) | Single AZ |
| **Persistent 1** | Replicated within AZ | Longer-term compute, ML training | Single AZ |
| **Persistent 2** | Replicated within AZ, higher durability | Longer-term, sensitive workloads | Single AZ |

**Exam critical:** FSx for Lustre is **always single-AZ**. For multi-AZ HA, it's NOT the right choice. Use EFS or FSx for ONTAP.

#### S3 Data Repository Integration

- **Linked S3 bucket**: FSx Lustre file system can be linked to an S3 bucket.
- **Lazy loading**: Data loaded from S3 on first access (not pre-loaded at creation).
- **Auto-import**: New/changed S3 objects automatically appear in the file system.
- **Auto-export**: Changed files written back to S3 automatically (or on-demand via `hsm_archive`).
- Use case: Process S3 data at HPC speed → results written back to S3 → delete the Lustre file system.

---

### Amazon FSx for NetApp ONTAP

Fully managed NetApp ONTAP file system — the most **feature-rich** and **multi-protocol** option.

#### Key Features

| Feature | Detail |
|---------|--------|
| **Protocols** | **NFS, SMB, and iSCSI** — all simultaneously on the same data |
| **OS** | Linux, Windows, macOS |
| **Multi-AZ** | **Yes** — active/standby with automatic failover |
| **Storage efficiency** | Deduplication, compression, compaction, thin provisioning — can save 60–70% |
| **Snapshots** | Instant, space-efficient, user-accessible |
| **Cloning** | **FlexClone** — instant, zero-copy clones of volumes (for dev/test) |
| **Tiering** | Automatic tiering to capacity pool storage (cheaper, S3-backed). Configurable policies: Auto, Snapshot-Only, All, None |
| **SnapMirror** | Cross-Region replication for DR |
| **On-premises** | Accessible from on-prem via VPN/Direct Connect. Also supports **NetApp Cloud Volumes ONTAP** on-prem for hybrid |
| **iSCSI** | Block storage via iSCSI — can serve as SAN storage for databases (Oracle, SQL Server) |
| **Multi-account** | Can be shared across accounts via AWS RAM |

#### Architecture

- **File System**: Contains one or more **Storage Virtual Machines (SVMs)**.
- **SVM**: Isolated virtual file server with its own DNS name, protocols, and access.
- **Volume**: Logical container within an SVM. Volumes have individual policies (tiering, snapshots, quotas).
- **Capacity Pool**: Low-cost S3-backed tier for infrequently accessed data. Data moves automatically.

#### Performance

| Metric | Value |
|--------|-------|
| Latency | Sub-millisecond (SSD tier), 10s of ms (capacity pool) |
| Throughput | Up to 4 GB/s per file system |
| IOPS | Hundreds of thousands |
| Scale-out | Up to 6 HA pairs per file system (scale-out architecture) |

---

### Amazon FSx for OpenZFS

Fully managed **OpenZFS file system** — optimized for latency-sensitive NFS workloads with rich ZFS data management features.

#### Key Features

| Feature | Detail |
|---------|--------|
| **Protocol** | NFS (v3, v4, v4.1, v4.2) |
| **OS** | Linux, macOS, Windows (with NFS client) |
| **Multi-AZ** | **Yes** (since 2023) — active/standby automatic failover |
| **Snapshots** | Instant, space-efficient ZFS snapshots |
| **Cloning** | Instant, zero-copy clones from snapshots |
| **Compression** | Z-Standard, LZ4 — up to 3–4× compression |
| **Copy-on-write** | ZFS COW semantics — data integrity, no journaling overhead |
| **NFS delegation** | Supports NFS delegations for improved client-side caching |
| **User quotas** | Per-user and per-group storage quotas |
| **On-premises** | Accessible via VPN/Direct Connect |

#### Performance

| Metric | Value |
|--------|-------|
| Latency | **~100 microseconds** (sub-millisecond) — lowest latency among FSx family |
| Throughput | Up to 12.5 GB/s |
| IOPS | Up to 1 million |

#### Primary Use Cases

- **Migration from on-prem ZFS/NFS** — drop-in replacement with same semantics.
- **Low-latency NFS workloads** — databases, media processing, financial analytics.
- **Dev/test cloning** — instant clones for test environments.

---

### Head-to-Head Comparison

| Feature | EFS | FSx Windows | FSx Lustre | FSx ONTAP | FSx OpenZFS |
|---------|-----|-------------|-----------|-----------|-------------|
| **Protocol** | NFSv4.1 | SMB | Lustre (POSIX) | NFS + SMB + iSCSI | NFS |
| **OS** | Linux | Windows (+Linux SMB) | Linux | Linux + Windows + macOS | Linux + macOS (+Windows NFS) |
| **Multi-AZ** | **Yes** (default) | Yes (optional) | **No** | **Yes** | **Yes** |
| **Max throughput** | 10 GB/s | 12 GB/s (HDD) | Hundreds of GB/s | 4 GB/s | 12.5 GB/s |
| **Max IOPS** | 500K+ | Hundreds of K | Millions | Hundreds of K | 1 million |
| **Latency** | Sub-ms | Sub-ms (SSD) | Sub-ms | Sub-ms (SSD tier) | **~100 µs** |
| **S3 integration** | No | No | **Yes (native)** | No (manual) | No |
| **AD integration** | No | **Yes (required)** | No | Yes (optional) | No |
| **Deduplication** | No | **Yes** | No | **Yes** | No |
| **Compression** | No | No | LZ4 | **Yes** | **Yes** (Z-Std, LZ4) |
| **iSCSI (block)** | No | No | No | **Yes** | No |
| **Cloning** | No | No | No | **FlexClone** | **Yes** |
| **Auto-tiering** | **Yes** (IA/Archive) | No | No | **Yes** (capacity pool) | No |
| **Serverless/Elastic** | **Yes** | No (fixed capacity) | No (fixed capacity) | No (fixed SSD, elastic capacity pool) | No (fixed capacity) |
| **Lambda/Fargate** | **Yes** | No | No | No | No |
| **Pricing model** | Pay per GB stored | Fixed capacity provisioned | Fixed capacity provisioned | Fixed SSD + capacity pool | Fixed capacity provisioned |
| **Cost (starting)** | $0.30/GB/month (Std) | ~$0.013/GB/month (HDD) | ~$0.14/GB/month (Persistent) | ~$0.125/GB/month (SSD) | ~$0.09/GB/month |

---

### Default Limits Worth Memorizing

| Resource | Limit |
|----------|-------|
| EFS file systems per account per Region | **1,000** |
| EFS max file size | **52 TiB** |
| EFS mount targets per file system per AZ | **1** |
| EFS max throughput (Elastic) | **10 GB/s read, 3 GB/s write** |
| FSx for Windows max file system size | **64 TiB (SSD) / 512 TiB (HDD)** |
| FSx for Lustre max file system size | **Hundreds of PB** |
| FSx for Lustre max throughput | **Hundreds of GB/s** |
| FSx for ONTAP max SSD storage per FS | **192 TiB** (scale-out: 1 PiB) |
| FSx for ONTAP capacity pool | Virtually unlimited (S3-backed) |
| FSx for OpenZFS max storage | **512 TiB** |
| FSx for OpenZFS max IOPS | **1 million** |
| All FSx: file systems per account per Region | **100** (adjustable) |

---

## 2. Design Patterns & Best Practices

### When to Use Which — The Decision Matrix

| Scenario | Service | Why |
|----------|---------|-----|
| Linux shared file system, elastic, serverless | **EFS** | NFSv4.1, auto-scaling, Lambda/Fargate compatible |
| Windows file shares, AD/GPO, DFS namespaces | **FSx for Windows** | Only service with SMB + native AD integration |
| HPC, ML training, genomics, video rendering | **FSx for Lustre** | Highest throughput, S3 integration, purpose-built for HPC |
| Process S3 data at HPC speed, then discard file system | **FSx for Lustre (Scratch)** | Cheapest, linked S3 bucket, no persistence needed |
| Multi-protocol (NFS + SMB + iSCSI) on same data | **FSx for ONTAP** | Only service offering all three protocols simultaneously |
| Migrate from on-prem NetApp to AWS | **FSx for ONTAP** | Compatible APIs, SnapMirror, same management paradigm |
| Migrate from on-prem ZFS/NFS to AWS | **FSx for OpenZFS** | Drop-in ZFS replacement with same semantics |
| Lowest NFS latency (sub-100µs) | **FSx for OpenZFS** | Lowest latency in the FSx family |
| Dev/test: instant zero-copy clones | **FSx for ONTAP** or **FSx for OpenZFS** | FlexClone / ZFS clones — instant, space-efficient |
| Enterprise SAN replacement (iSCSI) | **FSx for ONTAP** | Only managed service with iSCSI block access |
| Shared storage for Fargate or Lambda | **EFS** | Only file storage service compatible with serverless compute |
| Home directories for WorkSpaces | **FSx for Windows** (Windows) or **EFS** (Linux) | Depends on OS |
| Hybrid cloud storage with on-prem access | **FSx for ONTAP** | NetApp hybrid cloud story, SnapMirror, all protocols |
| Cost-optimized archival file storage | **EFS** (IA/Archive tiers) | Intelligent-Tiering to $0.008/GB Archive tier |

### Anti-Patterns

| Anti-Pattern | Why It's Wrong | Correct Approach |
|-------------|---------------|------------------|
| EFS for Windows workloads | EFS is NFS only — Windows doesn't natively support NFS well | FSx for Windows (SMB) or FSx for ONTAP (SMB + NFS) |
| FSx for Lustre for multi-AZ HA requirements | Lustre is **single-AZ only** — no multi-AZ option | EFS (multi-AZ default) or FSx for ONTAP (multi-AZ) |
| FSx for Windows without Active Directory | AD is **mandatory** for FSx for Windows — cannot operate without it | If no AD available: EFS (Linux) or FSx for OpenZFS/ONTAP |
| EFS for HPC with hundreds of GB/s throughput | EFS maxes at ~10 GB/s. HPC needs more | FSx for Lustre (hundreds of GB/s) |
| FSx for Lustre for persistent long-term storage | Scratch type is ephemeral. Even Persistent is single-AZ | Use for compute, store results in S3 or EFS for persistence |
| EFS for workloads needing block storage (database) | EFS is a file system, not block storage | EBS or FSx for ONTAP (iSCSI) |
| FSx for Windows for Linux-only environment | Unnecessary complexity and cost when no Windows/AD needed | EFS or FSx for OpenZFS |
| EFS with Provisioned Throughput for spiky workloads | Over-provisioning for peaks wastes money | EFS Elastic Throughput (pay per use) |

### Common Architectural Patterns

#### Shared Linux Storage for Containers
```
ECS/Fargate Tasks (multi-AZ)
  → EFS mount targets (one per AZ)
    → EFS Standard (Elastic Throughput)
      → Lifecycle: Standard-IA (30d) → Archive (90d)
```

#### Windows Enterprise File Shares
```
Active Directory (AWS Managed Microsoft AD or self-managed)
  → FSx for Windows (Multi-AZ, SSD)
    → DFS Namespace (multiple FSx file systems under one namespace)
    → Shadow Copies (end-user self-service restore)
    → Deduplication enabled (50-80% savings)
  ← On-premises access via Direct Connect
```

#### HPC Pipeline with S3
```
S3 Bucket (raw data: genomics, simulation inputs)
  → FSx for Lustre (linked to S3, Scratch type)
    → EC2 HPC cluster (Cluster Placement Group)
      → Process at 200+ GB/s
    → Results auto-exported back to S3
  → Delete Lustre file system (temporary compute)
```

#### Multi-Protocol Enterprise Hybrid
```
On-premises NetApp (SnapMirror DR to AWS)
  → FSx for ONTAP (Multi-AZ)
    → NFS: Linux application servers
    → SMB: Windows file shares (AD integrated)
    → iSCSI: Oracle database (block storage)
    → Capacity pool tiering (cold data → S3-backed, auto)
  ← Direct Connect (private connectivity)
```

#### Dev/Test Environment Cloning
```
FSx for ONTAP Production Volume
  → FlexClone → Dev Volume (instant, zero-copy, writable)
  → FlexClone → Test Volume (instant, zero-copy, writable)
  → FlexClone → QA Volume (instant, zero-copy, writable)
Each clone uses storage only for changed blocks
```

### Well-Architected Framework Alignment

**Reliability**
- EFS: Multi-AZ by default — highest HA among file services.
- FSx for Windows / ONTAP / OpenZFS: Multi-AZ deployment option with automatic failover.
- FSx for Lustre: Single-AZ only — back up data to S3 for durability.
- Use AWS Backup for all services (EFS, FSx) for centralized backup.
- FSx for ONTAP SnapMirror for cross-Region DR.

**Security**
- All services support **encryption at rest (KMS)** and **in transit**.
- EFS: IAM authorization + POSIX permissions + security groups on mount targets.
- FSx for Windows: AD-based ACLs (NTFS), Kerberos authentication.
- FSx for ONTAP: AD (SMB), Kerberos (NFS), CHAP (iSCSI).
- All: VPC security groups for network access control.

**Performance**
- EFS Elastic: auto-scales throughput (up to 10 GB/s).
- FSx for Lustre: hundreds of GB/s for HPC.
- FSx for OpenZFS: ~100µs latency — lowest in the family.
- FSx for ONTAP: sub-ms on SSD tier, auto-tiering offloads cold data.
- FSx for Windows: SSD for IOPS, HDD for throughput.

**Cost Optimization**
- EFS Lifecycle (IA + Archive) for auto-tiering — up to 92% savings vs Standard.
- FSx for ONTAP capacity pool tiering — cold data on S3-backed storage.
- FSx for Windows HDD for throughput workloads (cheaper than SSD).
- FSx for Lustre Scratch for short-lived compute (cheapest, no replication overhead).
- EFS One Zone for dev/test (47% cheaper than Standard).

**Operational Excellence**
- All services support AWS Backup.
- FSx for ONTAP: FlexClone for instant dev/test environments.
- EFS: fully serverless — zero capacity management.
- FSx for Windows: Shadow Copies for end-user self-service restore.

---

## 3. Security & Compliance

### Access Control Comparison

| Service | Authentication | Authorization | Network Access |
|---------|---------------|---------------|---------------|
| **EFS** | IAM policies + access points | POSIX user/group IDs | Security groups on mount targets, VPC |
| **FSx Windows** | Active Directory (Kerberos) | NTFS ACLs | Security groups, VPC, AD trust |
| **FSx Lustre** | None (network-level) | POSIX permissions | Security groups, VPC |
| **FSx ONTAP** | AD (SMB), Kerberos (NFS), CHAP (iSCSI) | NTFS ACLs (SMB), POSIX/NFSv4 ACLs (NFS) | Security groups, VPC, route tables |
| **FSx OpenZFS** | None (NFS-level) | POSIX permissions, NFSv4 ACLs | Security groups, VPC |

#### EFS Access Points

- Create **application-specific entry points** into EFS.
- Enforce a POSIX user/group ID (override whatever the client sends).
- Enforce a root directory (chroot-like behavior).
- Use with **IAM authorization** — IAM policy determines which access point a role can use.
- **Exam use case:** Multi-tenant applications where each tenant gets a separate EFS access point with enforced directory and UID/GID.

#### EFS IAM Authorization

```json
{
  "Effect": "Allow",
  "Action": [
    "elasticfilesystem:ClientMount",
    "elasticfilesystem:ClientWrite"
  ],
  "Resource": "arn:aws:elasticfilesystem:us-east-1:123456789012:file-system/fs-abc123",
  "Condition": {
    "StringEquals": {
      "elasticfilesystem:AccessPointArn": "arn:aws:elasticfilesystem:us-east-1:123456789012:access-point/fsap-xyz789"
    }
  }
}
```

### Encryption

| Service | At Rest | In Transit | Key Management |
|---------|---------|-----------|----------------|
| **EFS** | KMS (AES-256). Must enable at creation — cannot enable later | TLS via EFS mount helper (`-o tls`) | AWS-managed or CMK |
| **FSx Windows** | KMS (AES-256). Enabled at creation | SMB 3.x encryption (Kerberos) | AWS-managed or CMK |
| **FSx Lustre** | KMS (AES-256). Enabled at creation | In-transit encryption (Lustre client 2.10+) | AWS-managed or CMK |
| **FSx ONTAP** | KMS (AES-256). Enabled at creation | NFS/SMB/iSCSI in-transit encryption | AWS-managed or CMK |
| **FSx OpenZFS** | KMS (AES-256). Enabled at creation | NFS in-transit encryption | AWS-managed or CMK |

**Exam critical:** EFS encryption at rest must be enabled **at file system creation**. Cannot be enabled on an existing file system. To encrypt an existing EFS, create a new encrypted file system and migrate data (AWS DataSync recommended).

### Logging & Monitoring

| Tool | Coverage |
|------|----------|
| **CloudWatch Metrics** | All services: throughput, IOPS, latency, storage used, credit balance |
| **CloudTrail** | All services: API calls (create/delete/modify file systems) |
| **CloudWatch Logs** | FSx for Windows: audit logs (file access, share access). EFS: no native file-level audit logging |
| **VPC Flow Logs** | Network-level traffic to mount targets / file system ENIs |
| **AWS Config** | Rules for encryption validation, backup compliance |

### Compliance Patterns

- **HIPAA / PCI**: All five services are eligible. Use encryption at rest + in transit + CloudTrail + appropriate access controls.
- **Windows audit requirements**: FSx for Windows file access auditing → CloudWatch Logs or Kinesis Firehose.
- **Multi-tenant isolation**: EFS access points enforce per-tenant root directories and POSIX IDs.
- **Data residency**: All services are Regional. Use SCPs to restrict to approved Regions.

---

## 4. Cost Optimization

### Pricing Comparison (us-east-1, approximate)

| Service | Storage (SSD/Standard) | Storage (Lower Tier) | Throughput | Key Cost Driver |
|---------|----------------------|---------------------|-----------|----------------|
| **EFS Standard** | $0.30/GB/month | $0.025 (IA), $0.008 (Archive) | Elastic: per-data-transferred | Storage × access pattern |
| **EFS One Zone** | $0.16/GB/month | $0.0133 (One Zone-IA) | Same as Standard | Cheaper for non-critical data |
| **FSx Windows SSD** | ~$0.13/GB/month | N/A | Included in provisioned capacity | Provisioned capacity |
| **FSx Windows HDD** | ~$0.013/GB/month | N/A | $0.12/MBps/month (above baseline) | Provisioned capacity + throughput |
| **FSx Lustre Scratch** | ~$0.14/GB/month | N/A | Included (200 MB/s per TiB) | Provisioned capacity |
| **FSx Lustre Persistent** | ~$0.145–$0.21/GB/month | N/A | Varies by throughput tier | Provisioned capacity × throughput tier |
| **FSx ONTAP SSD** | ~$0.125/GB/month | ~$0.025 (capacity pool) | Included | SSD + capacity pool usage |
| **FSx OpenZFS** | ~$0.09/GB/month | N/A | Included | Provisioned capacity |

### Cost-Saving Strategies

1. **EFS Lifecycle Management (IA + Archive)**: Move cold data automatically. Archive tier is $0.008/GB — 97% cheaper than Standard. Biggest savings for infrequently accessed file data.

2. **EFS One Zone for dev/test**: 47% cheaper than Standard. Single-AZ risk is acceptable for non-production.

3. **EFS Elastic Throughput**: Pay only for data transferred (not provisioned capacity). Much cheaper than Provisioned Throughput for spiky workloads.

4. **FSx for Windows HDD**: 10× cheaper than SSD for throughput-oriented workloads (home directories with large files).

5. **FSx for Windows Deduplication**: Saves 50–80% on storage for file shares with many duplicate files (VM images, software packages).

6. **FSx for Lustre Scratch**: No replication overhead — cheapest Lustre option for temporary HPC. Delete after job completion.

7. **FSx for ONTAP Capacity Pool**: Cold data auto-tiers to S3-backed storage at $0.025/GB. Keep SSD provisioned only for hot data.

8. **FSx for ONTAP Deduplication + Compression**: Can reduce effective storage by 60–70%. You pay for logical size, savings are on reduced capacity needs.

9. **FSx for OpenZFS Compression**: Z-Standard compression can achieve 3–4× reduction. Effective cost drops significantly.

### Common Cost Traps

- **EFS Standard with no lifecycle policy**: All data at $0.30/GB even if 80% is cold. Enable lifecycle immediately.
- **EFS Provisioned Throughput for spiky workloads**: Paying for peak even during idle. Switch to Elastic.
- **FSx for Windows SSD for large-file throughput**: SSD is optimized for IOPS. Use HDD for large sequential access (cheaper + better throughput for this pattern).
- **Over-provisioned FSx capacity**: All FSx services (except EFS) require pre-provisioned storage. Over-provisioning wastes money.
- **Forgetting FSx ONTAP capacity pool**: Without enabling auto-tiering, all data sits on expensive SSD tier.
- **FSx for Lustre left running after HPC job**: Lustre is expensive. Delete Scratch file systems immediately after job completion.

---

## 5. High Availability, Disaster Recovery & Resilience

### HA Comparison

| Service | Multi-AZ | Cross-Region Replication | Backup |
|---------|---------|------------------------|--------|
| **EFS** | **Default** (Standard class) | **EFS Replication** (async, one destination) | AWS Backup |
| **FSx Windows** | **Optional** (active-standby, auto failover) | Not native — use AWS Backup cross-Region or DFS-R | AWS Backup, Shadow Copies |
| **FSx Lustre** | **No** (single-AZ always) | Not supported — back up to S3 | AWS Backup, S3 export |
| **FSx ONTAP** | **Yes** (active-standby, auto failover) | **SnapMirror** (cross-Region, cross-account) | AWS Backup, ONTAP snapshots |
| **FSx OpenZFS** | **Yes** (active-standby, auto failover) | Not native — use AWS Backup cross-Region | AWS Backup, ZFS snapshots |

### DR Patterns

| Pattern | Service | Implementation |
|---------|---------|---------------|
| **Active-Active multi-AZ** | EFS | Default — data replicated across AZs, mount targets in each AZ |
| **Active-Passive multi-AZ** | FSx Windows, ONTAP, OpenZFS | Multi-AZ deployment with automatic DNS failover to standby |
| **Cross-Region DR** | EFS | EFS Replication to another Region (RPO: minutes) |
| **Cross-Region DR** | FSx ONTAP | SnapMirror to DR Region (RPO: configurable, minutes to hours) |
| **Cross-Region DR** | FSx Windows / OpenZFS | AWS Backup cross-Region copy (RPO: hours) |
| **HPC DR** | FSx Lustre | Data backed up in S3 (durable). Create new Lustre FS in DR Region linked to S3. RPO depends on S3 state |

### EFS Replication

- Replicate an EFS file system to **another Region or same Region** (different AZ set).
- **Asynchronous** — RPO is typically minutes.
- **One replication destination** per file system.
- Destination file system is **read-only** until you promote it (during failover).
- Supports cross-account replication via AWS RAM.

### FSx for ONTAP SnapMirror

- Cross-Region, cross-account replication.
- Configurable RPO (schedule-based).
- Can replicate individual volumes (not entire file system).
- Integrates with NetApp's SnapMirror technology — familiar to enterprise NetApp admins.

### Backup (AWS Backup)

All five services integrate with AWS Backup:
- Automated, policy-based backup schedules.
- Cross-Region and cross-account vault copy.
- Backup compliance auditing.
- Point-in-time restore for EFS.
- FSx ONTAP: AWS Backup + native ONTAP snapshots (complementary).

---

## 6. Exam-Focused Section

### Straightforward Questions (7 examples)

**Q1:** A company needs shared file storage for a fleet of Linux EC2 instances across 3 AZs. The storage must grow and shrink automatically. Which service should they use?
**A:** **Amazon EFS.** It's the only fully elastic, multi-AZ NFS file system. No capacity provisioning needed.

**Q2:** A company needs to provide Windows file shares with Active Directory integration, NTFS permissions, and DFS Namespaces. Which service?
**A:** **FSx for Windows File Server.** It's the only AWS service with native SMB, AD integration, and DFS support.

**Q3:** An HPC workload needs to process 50 TB of data stored in S3 at hundreds of GB/s throughput. Which service?
**A:** **FSx for Lustre.** Native S3 integration (linked bucket) + hundreds of GB/s throughput, purpose-built for HPC.

**Q4:** A company migrating from on-premises NetApp needs NFS, SMB, and iSCSI access on the same data set. Which service?
**A:** **FSx for NetApp ONTAP.** Only managed service supporting all three protocols simultaneously with NetApp feature compatibility.

**Q5:** A Lambda function needs persistent, shared file storage. Which service?
**A:** **Amazon EFS.** It's the only file storage service that Lambda (and Fargate) can mount natively.

**Q6:** Which FSx service provides the lowest NFS latency (~100 microseconds)?
**A:** **FSx for OpenZFS.** Sub-100µs latency, purpose-built for latency-sensitive NFS workloads.

**Q7:** Is FSx for Lustre multi-AZ?
**A:** **No.** FSx for Lustre is always single-AZ (both Scratch and Persistent deployment types).

---

### Tricky / Scenario-Based Questions (7 examples)

**Q1:** A media company processes video files stored in S3. The processing requires POSIX file system access at 100+ GB/s aggregate throughput. After processing, results must be written back to S3. The processing takes 4 hours and runs weekly. Which storage solution minimizes cost?

**A:** **FSx for Lustre (Scratch type)** linked to the S3 bucket.
- **Why Scratch?** Data is temporary (4 hours/week). Scratch is cheapest — no replication overhead. The source data is durable in S3, and results are exported back to S3.
- **Why not EFS?** EFS maxes at ~10 GB/s. The workload needs 100+ GB/s — only Lustre can deliver this.
- **Why not FSx for ONTAP?** ONTAP maxes at ~4 GB/s and doesn't have native S3 linking.
- **Why not keep data on S3?** S3 is object storage, not POSIX. The application requires file system access.
- **Key phrases:** "S3 data" + "100+ GB/s" + "POSIX" + "temporary processing" + "minimize cost" → FSx for Lustre Scratch + S3 link.

**Q2:** A financial services company runs both Linux (NFS) and Windows (SMB) workloads that need access to the **same data** set. They require multi-AZ HA, iSCSI for an Oracle database, and cross-Region DR. Which service?

**A:** **FSx for NetApp ONTAP.**
- **Why ONTAP?** Only service supporting NFS + SMB + iSCSI simultaneously on the same data. Multi-AZ HA with automatic failover. SnapMirror for cross-Region DR.
- **Why not EFS + FSx Windows?** Two separate services = two separate data copies. The requirement is "same data" accessible via both protocols.
- **Why not FSx for OpenZFS?** OpenZFS supports NFS only — no SMB or iSCSI.
- **Key phrases:** "NFS + SMB on same data" + "iSCSI" + "multi-AZ" + "cross-Region DR" → FSx for ONTAP.

**Q3:** A company has an EFS file system with 100 TB of data. 90 TB hasn't been accessed in 6 months. They want to reduce costs without changing application behavior. What should they do?

**A:** Enable **EFS Lifecycle Management** to transition files to **Standard-IA** (after configurable days) and **Archive** (after 90+ days).
- **Cost savings:** 90 TB at Standard = $27,000/month. 90 TB at Archive = $720/month. **Saves ~$26,280/month.**
- **Why not One Zone?** Existing Standard file system cannot be converted to One Zone. Would need to create new FS and migrate.
- **Why not move to S3?** Application requires file system (POSIX) access. S3 is object storage.
- **Application impact:** First-byte latency is the same across all EFS classes. Application behavior is transparent — no code changes.
- **Key phrases:** "EFS" + "90% cold data" + "reduce costs" + "no application changes" → Lifecycle to IA/Archive.

**Q4:** A company's EFS file system was created without encryption at rest. The security team now requires all data to be encrypted with a CMK. Can they enable encryption on the existing file system?

**A:** **No.** EFS encryption at rest must be enabled **at creation time**. Cannot be added later.
- **Fix:** Create a **new encrypted EFS file system**, then migrate data using **AWS DataSync** (or `rsync`). Update mount targets and application config.
- **Why not EFS replication?** EFS replication creates a destination FS, but the source must be encrypted for the destination to be encrypted. Doesn't solve the problem.
- **Key phrases:** "existing EFS" + "enable encryption" → Cannot. Must create new encrypted FS + migrate.

**Q5:** A DevOps team needs to create 20 independent copies of a 10 TB production data set for parallel testing. Each copy must be writable. Storage cost must be minimized. Which service?

**A:** **FSx for NetApp ONTAP with FlexClone** (or FSx for OpenZFS with ZFS clones).
- **Why?** FlexClone creates **instant, zero-copy, writable clones**. 20 clones of 10 TB = virtually 0 additional storage (only changed blocks consume space). Creating 20 copies with EBS or EFS would require 200 TB of storage.
- **Why not EBS snapshots?** EBS snapshots create new volumes — each would consume space for writes. No shared dedup across clones.
- **Key phrases:** "multiple writable copies" + "minimize storage cost" + "instant" → FlexClone (ONTAP) or ZFS clone (OpenZFS).

**Q6:** A company needs to provide shared file storage for ECS Fargate tasks running in multiple AZs. The data must persist across task restarts. An architect proposes FSx for Lustre. Is this correct?

**A:** **No. Use Amazon EFS.**
- **Why not FSx for Lustre?** (1) Fargate **does not support** FSx for Lustre mounting. (2) FSx for Lustre is single-AZ, but the tasks are multi-AZ. EFS is the **only file storage** supported by Fargate.
- **Why EFS?** Native Fargate/Lambda integration, multi-AZ, persistent, elastic.
- **Key phrases:** "Fargate" + "shared persistent storage" + "multi-AZ" → EFS. Always EFS for serverless compute.

**Q7:** A company has an on-premises Windows file server with 500 TB of data using DFS Namespaces and Windows ACLs. They want to migrate to AWS with minimal disruption to end users and maintain the same namespace/permissions model. Which service and migration approach?

**A:** **FSx for Windows File Server** (Multi-AZ, SSD or HDD) + **AWS DataSync** for migration + **DFS Namespaces** to redirect clients.
- **Why FSx for Windows?** Only service supporting SMB + NTFS ACLs + DFS Namespaces + Active Directory — matches the on-prem capabilities exactly.
- **Migration:** AWS DataSync preserves NTFS permissions, timestamps, and ownership. DFS Namespaces allow gradual cutover (redirect folders one at a time).
- **Why not FSx for ONTAP?** ONTAP supports SMB and AD, but does NOT support DFS Namespaces natively. The question specifically mentions DFS.
- **Key phrases:** "Windows file server" + "DFS Namespaces" + "Windows ACLs" + "migrate" → FSx for Windows + DataSync.

---

### Common Exam Traps & Pitfalls

1. **EFS encryption cannot be enabled after creation**: Must be at creation time. This is the #1 EFS trap.

2. **FSx for Lustre is ALWAYS single-AZ**: Both Scratch and Persistent. If the question requires multi-AZ file storage, Lustre is wrong. Use EFS or FSx for ONTAP.

3. **FSx for Windows REQUIRES Active Directory**: No AD = no FSx for Windows. If the scenario has no AD, FSx for Windows is not an option.

4. **EFS is the only option for Lambda/Fargate**: No other file storage integrates with serverless compute. This is tested frequently.

5. **Multi-protocol (NFS + SMB + iSCSI) = FSx for ONTAP**: No other service provides all three. If the question mentions accessing the same data via multiple protocols, the answer is always ONTAP.

6. **FSx for Lustre S3 integration is NATIVE; EFS has no S3 integration**: If the question says "process S3 data as files at high throughput," the answer is FSx for Lustre — not EFS.

7. **EFS is elastic; all FSx services require provisioned capacity**: If the question says "no capacity management" or "scales automatically," the answer is EFS.

8. **FlexClone = ONTAP, ZFS Clones = OpenZFS**: These are zero-copy, instant, writable clones. EFS and FSx for Windows/Lustre don't have this capability.

9. **DFS Namespaces = FSx for Windows only**: If the question mentions DFS, the answer is FSx for Windows.

10. **Shadow Copies (Previous Versions) = FSx for Windows only**: End-user self-service restore via right-click → Previous Versions.

11. **FSx for ONTAP Capacity Pool ≠ S3**: The capacity pool is S3-backed internally, but users don't interact with S3 directly. It's transparent auto-tiering.

12. **EFS One Zone ≠ One Zone-IA**: One Zone is a deployment type (single AZ, cheaper). IA is a storage class (infrequent access, even cheaper). They can be combined: One Zone-IA is the cheapest EFS option.

13. **EFS max throughput (Elastic)**: 10 GB/s read, 3 GB/s write. If the question needs hundreds of GB/s, the answer is FSx for Lustre — not EFS.

14. **Cross-Region replication**: EFS has native EFS Replication. FSx for ONTAP has SnapMirror. FSx for Windows/Lustre/OpenZFS rely on AWS Backup for cross-Region copy.

15. **iSCSI (block over network) = FSx for ONTAP only**: If the question mentions SAN replacement or block storage over network, the answer is ONTAP.

---

## 7. Cheat Sheet

### Must-Know Facts for Exam Day

| Fact | Value |
|------|-------|
| EFS protocol | **NFSv4.1** |
| EFS multi-AZ | **Default** (Standard class) |
| EFS max throughput (Elastic) | **10 GB/s read, 3 GB/s write** |
| EFS encryption after creation | **Not possible** — must enable at creation |
| EFS + Lambda/Fargate | **Yes** — only file storage option for serverless |
| EFS cheapest tier | **Archive: $0.008/GB/month** |
| FSx Windows protocol | **SMB** |
| FSx Windows AD | **Required** |
| FSx Windows DFS | **Supported** |
| FSx Windows deduplication | **Built-in (50-80% savings)** |
| FSx Lustre multi-AZ | **No** (always single-AZ) |
| FSx Lustre S3 integration | **Native (linked bucket)** |
| FSx Lustre max throughput | **Hundreds of GB/s** |
| FSx ONTAP protocols | **NFS + SMB + iSCSI** (all three simultaneously) |
| FSx ONTAP multi-AZ | **Yes** |
| FSx ONTAP cloning | **FlexClone** (instant, zero-copy) |
| FSx ONTAP tiering | **Auto to capacity pool (S3-backed)** |
| FSx ONTAP DR | **SnapMirror** (cross-Region) |
| FSx OpenZFS latency | **~100 µs** (lowest) |
| FSx OpenZFS multi-AZ | **Yes** |
| FSx OpenZFS compression | **Z-Standard, LZ4 (3-4×)** |

### Key Differentiators Table

| Feature | EFS | FSx Windows | FSx Lustre | FSx ONTAP | FSx OpenZFS |
|---------|-----|-------------|-----------|-----------|-------------|
| Protocol | NFS | SMB | Lustre | NFS+SMB+iSCSI | NFS |
| Multi-AZ | Yes (default) | Optional | **No** | Yes | Yes |
| S3 link | No | No | **Yes** | No | No |
| AD required | No | **Yes** | No | Optional | No |
| Serverless | **Yes** (Lambda/Fargate) | No | No | No | No |
| Elastic | **Yes** | No | No | No | No |
| Cloning | No | No | No | **FlexClone** | **Yes** |
| Dedup | No | **Yes** | No | **Yes** | No |
| iSCSI | No | No | No | **Yes** | No |

### Decision Flowchart

```
Question says "Linux shared file storage" + "elastic/serverless"?
  → EFS

Question says "Lambda" or "Fargate" + "persistent shared storage"?
  → EFS (only option)

Question says "Windows file shares" or "SMB" or "Active Directory" or "DFS"?
  → FSx for Windows File Server

Question says "HPC" or "ML training" or "hundreds of GB/s" or "process S3 data as files"?
  → FSx for Lustre

Question says "S3 data" + "file system access" + "high performance"?
  → FSx for Lustre (native S3 link)

Question says "NFS + SMB" or "multi-protocol" or "iSCSI" or "SAN replacement"?
  → FSx for NetApp ONTAP

Question says "NetApp migration" or "SnapMirror" or "FlexClone"?
  → FSx for NetApp ONTAP

Question says "ZFS migration" or "lowest NFS latency" or "ZFS snapshots"?
  → FSx for OpenZFS

Question says "instant writable clones" + "dev/test"?
  → FSx for ONTAP (FlexClone) or FSx for OpenZFS (ZFS clones)

Question says "multi-AZ file storage" + NOT Windows/AD?
  → EFS (default multi-AZ, elastic) or FSx ONTAP/OpenZFS (if enterprise features needed)

Question says "reduce EFS costs" + "cold data"?
  → EFS Lifecycle (Standard-IA → Archive)

Question says "temporary HPC compute" + "data safe in S3"?
  → FSx for Lustre Scratch (cheapest, delete after job)

Question says "on-premises access" + "Windows" + "Direct Connect"?
  → FSx for Windows (Multi-AZ) via Direct Connect

Question says "on-premises access" + "multi-protocol" + "hybrid"?
  → FSx for ONTAP (SnapMirror, all protocols, Direct Connect)

Question says "EFS" + "encrypt existing file system"?
  → Cannot. Create new encrypted EFS + migrate with DataSync.

Question says "file system" + "multi-AZ" + "Lustre"?
  → Trap! Lustre is single-AZ. Choose EFS or ONTAP instead.

Question says "shared storage" + "Fargate" + "Lustre"?
  → Trap! Fargate only supports EFS.
```

---

*Generated for AWS SAP-C02 exam preparation. Last updated: 2026-04-26.*
