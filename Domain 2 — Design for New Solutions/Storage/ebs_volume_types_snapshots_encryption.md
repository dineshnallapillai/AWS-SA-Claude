# AWS SAP-C02 Study Notes: EBS (Volume Types, Snapshots, Encryption)

---

## 1. Core Concepts & Theory

### Amazon EBS Fundamentals

Amazon Elastic Block Store (EBS) provides **persistent block storage** for EC2 instances. An EBS volume behaves like a raw, unformatted block device that you can mount as a file system or use as raw storage (databases).

#### Key Properties

| Property | Detail |
|----------|--------|
| **Scope** | **AZ-scoped** — a volume exists in a single AZ and can only attach to instances in that same AZ |
| **Persistence** | Data persists independently of instance lifecycle (survives stop/start/terminate if DeleteOnTermination=false) |
| **Replication** | Automatically replicated **within its AZ** for durability (not across AZs) |
| **Durability** | 99.8%–99.999% AFR (Annual Failure Rate) depending on volume type — io2 offers 99.999% |
| **Attach model** | One volume to one instance (default). **Multi-Attach** for io1/io2 only (up to 16 Nitro instances, same AZ) |
| **Live modification** | Can change volume type, size, and IOPS **without detaching** (Elastic Volumes) |
| **Protocol** | NVMe or Xen block device, depending on instance generation |

---

### EBS Volume Types

There are **6 volume types** in two categories: SSD-backed (IOPS-focused) and HDD-backed (throughput-focused).

#### SSD-Backed Volumes

| Volume Type | **gp3** | **gp2** | **io2 Block Express** | **io2** | **io1** |
|-------------|---------|---------|----------------------|---------|---------|
| Category | General Purpose SSD | General Purpose SSD | Provisioned IOPS SSD | Provisioned IOPS SSD | Provisioned IOPS SSD |
| Max IOPS | **16,000** | **16,000** | **256,000** | **64,000** | **64,000** |
| Max throughput | **1,000 MB/s** | **250 MB/s** | **4,000 MB/s** | **1,000 MB/s** | **1,000 MB/s** |
| Size range | 1 GiB – 16 TiB | 1 GiB – 16 TiB | 4 GiB – **64 TiB** | 4 GiB – 16 TiB | 4 GiB – 16 TiB |
| Baseline IOPS | **3,000 included** | 3 IOPS/GiB (min 100, burst to 3,000) | Provisioned | Provisioned | Provisioned |
| Baseline throughput | **125 MB/s included** | Scales with IOPS | Provisioned | Provisioned | Provisioned |
| IOPS:GiB ratio | Up to 500:1 | 3:1 | **1,000:1** | **500:1** | 50:1 |
| Durability (AFR) | 0.1%–0.2% | 0.1%–0.2% | **0.001% (99.999%)** | **0.001% (99.999%)** | 0.1%–0.2% |
| Multi-Attach | No | No | **Yes** (up to 16) | **Yes** (up to 16) | **Yes** (up to 16) |
| Boot volume | Yes | Yes | Yes | Yes | Yes |
| Latency | Single-digit ms | Single-digit ms | **Sub-millisecond** | Single-digit ms | Single-digit ms |
| Instance requirement | Any | Any | **Nitro (specific types)** | Nitro recommended | Any |

#### HDD-Backed Volumes

| Volume Type | **st1** | **sc1** |
|-------------|---------|---------|
| Category | Throughput Optimized HDD | Cold HDD |
| Max IOPS | **500** | **250** |
| Max throughput | **500 MB/s** | **250 MB/s** |
| Size range | **125 GiB – 16 TiB** | **125 GiB – 16 TiB** |
| Baseline throughput | 40 MB/s per TiB | 12 MB/s per TiB |
| Burst throughput | 250 MB/s per TiB | 80 MB/s per TiB |
| Boot volume | **No** | **No** |
| Use case | Big data, data warehouse, log processing, streaming | Coldest data, infrequent access, lowest cost |

#### gp3 vs gp2 — Deep Comparison

| Aspect | gp3 | gp2 |
|--------|-----|-----|
| IOPS provisioning | **Independent of size** — 3,000 IOPS included, provision up to 16,000 | **Tied to size** — 3 IOPS/GiB. Need 5,334 GiB for max 16,000 IOPS |
| Throughput provisioning | **Independent of size** — 125 MB/s included, provision up to 1,000 MB/s | Tied to IOPS (max 250 MB/s) |
| Price (us-east-1) | $0.08/GiB/month | $0.10/GiB/month |
| Burst credits | **No burst model** — consistent provisioned performance | Burst credit pool (I/O credits). Volumes <1 TiB burst to 3,000 IOPS; deplete credits under sustained load |
| Exam default | **Preferred for new workloads** | Legacy — still valid but gp3 is better in almost all cases |

**Exam critical:** If a gp2 volume is showing inconsistent performance, the answer is almost always "switch to gp3" or "increase gp2 volume size to get higher baseline IOPS."

#### io2 Block Express — When to Use

- Requires **specific Nitro instance types** (R5b, R6i, X2idn, etc. — not all Nitro instances)
- Supports volumes up to **64 TiB** (vs 16 TiB for standard io2)
- Up to **256,000 IOPS** per volume (vs 64,000 for standard io2)
- Up to **4,000 MB/s** throughput
- **Sub-millisecond latency**
- **99.999% durability (0.001% AFR)** — highest among all EBS volume types
- Use for: SAP HANA, Oracle, SQL Server, and other large mission-critical databases

---

### EBS Snapshots

EBS snapshots are **point-in-time, incremental backups** stored in Amazon S3 (managed by AWS — you don't see the S3 bucket).

#### Snapshot Characteristics

| Property | Detail |
|----------|--------|
| **Storage** | Stored in S3 (AWS-managed, Region-scoped) |
| **Incremental** | Only changed blocks since last snapshot are stored. Deleting intermediate snapshots is safe — AWS manages block references |
| **Scope** | **Region-scoped** — created in the same Region as the volume |
| **Cross-Region** | Can be **copied** to another Region (for DR) |
| **Cross-Account** | Can be **shared** with specific accounts or made public |
| **Consistency** | Crash-consistent (not application-consistent). For app consistency, quiesce I/O or stop the instance before snapshot |
| **Availability during snapshot** | Volume is fully usable during snapshot creation |
| **Snapshot creation** | Asynchronous — returns immediately. Data is captured at the point-in-time of the API call |

#### Snapshot Restore Performance

- **New volumes from snapshots** load data lazily from S3 — first read of each block has higher latency.
- **EBS Fast Snapshot Restore (FSR)**: Pre-warms snapshot data so restored volumes perform at full provisioned IOPS/throughput immediately. No first-read penalty.
  - Enabled **per snapshot per AZ**
  - Costs: **$0.75/hour per AZ per snapshot** with FSR enabled
  - Up to **10 volumes per AZ** created simultaneously from an FSR-enabled snapshot at full performance
- **Alternative to FSR**: Initialize the volume by reading all blocks with `dd` or `fio` (free but time-consuming).

#### EBS Snapshots Archive

- Move snapshots to an **archive tier** — **75% cheaper** than standard snapshot storage.
- Retrieval time: **24–72 hours** to restore from archive to standard tier.
- Minimum archive duration: **90 days** (charged for 90 days even if retrieved earlier).
- Use for: Long-term retention snapshots that rarely need restoration.

#### Snapshot Lifecycle Management

- **Amazon Data Lifecycle Manager (DLM)**: Automates snapshot creation, retention, and cross-Region/cross-account copy.
  - Policies based on tags, schedules (every N hours/days), and retention rules.
  - Can create **application-consistent snapshots** using pre/post scripts (via SSM).
  - Supports EBS volumes and EC2 instance snapshots (all attached volumes).
- **AWS Backup**: Centralized backup service that manages EBS snapshots alongside RDS, DynamoDB, EFS, etc. Policy-based, cross-account, cross-Region.

#### Recycle Bin for EBS Snapshots

- Protects against accidental deletion.
- Deleted snapshots go to **Recycle Bin** for a configurable retention period (1 day – 1 year).
- Snapshots in Recycle Bin can be recovered.
- Must create **retention rules** before deletion occurs (rules don't retroactively protect already-deleted snapshots).

---

### EBS Encryption

EBS encryption provides at-rest and in-transit encryption using **AWS KMS** keys.

#### How It Works

- Uses **AES-256** algorithm.
- Encryption is handled by the EC2 host — **transparent to the instance** (no application changes needed).
- Encrypts:
  - Data at rest on the volume
  - Data in transit between the instance and the volume
  - All snapshots created from the volume
  - All volumes created from those snapshots

#### Encryption Keys

| Key Type | Description |
|----------|-------------|
| **AWS-managed key** (`aws/ebs`) | Default. AWS creates and manages. Free. **Cannot be shared cross-account** |
| **Customer Managed Key (CMK)** | You create in KMS. Full control over key policy, rotation, grants. **Can be shared cross-account**. Costs $1/month + API calls |

#### Encryption Constraints & Behaviors

- **Default encryption**: You can enable **EBS encryption by default** at the account level per Region. All new volumes and snapshots in that Region will be encrypted automatically.
- **Cannot un-encrypt a volume**: To remove encryption, create an unencrypted snapshot copy (only if the snapshot was encrypted with a key you can access), then create a new unencrypted volume. In practice, the exam expects you to go encrypted → snapshot → copy (unencrypted) → new volume.
- **Cannot un-encrypt a snapshot**: You CAN copy an encrypted snapshot as unencrypted only if it was encrypted with a CMK you control (not cross-account scenarios with restricted keys).
- **Encrypt an existing unencrypted volume**:
  1. Create a snapshot of the unencrypted volume
  2. Copy the snapshot with encryption enabled (specify KMS key)
  3. Create a new volume from the encrypted snapshot
  4. Attach the new encrypted volume, detach the old one
- **Cross-account snapshot sharing**: Encrypted snapshots can be shared only if encrypted with a **CMK** (not the AWS-managed `aws/ebs` key). You must also share the KMS key with the target account.
- **Cross-Region snapshot copy**: When copying to another Region, you must specify a KMS key in the **destination Region** (KMS keys are Region-specific).
- **Re-encryption on copy**: You can re-encrypt a snapshot with a different key when copying.

#### Encryption and Performance

- **No performance impact** on modern (Nitro) instances — encryption/decryption is offloaded to the Nitro card.
- Older (Xen-based) instance types may see minor CPU impact.
- **Encrypted volumes are EBS-optimized by default** on Nitro instances.

---

### Default Limits Worth Memorizing

| Resource | Default Limit | Adjustable? |
|----------|--------------|-------------|
| EBS volumes per Region | **5,000** | Yes |
| EBS snapshots per Region | **100,000** | Yes |
| gp3 max IOPS | **16,000** per volume | No |
| gp3 max throughput | **1,000 MB/s** per volume | No |
| io2 Block Express max IOPS | **256,000** per volume | No |
| io2 Block Express max throughput | **4,000 MB/s** per volume | No |
| io2 Block Express max size | **64 TiB** | No |
| io1/io2 max IOPS per instance | **160,000** (aggregate across all attached volumes) | No |
| io2 Block Express max IOPS per instance | **260,000** | No |
| gp3 max IOPS per instance | **260,000** (aggregate) | No |
| Multi-Attach: max instances per volume | **16** (io1/io2 on Nitro only) | No |
| st1/sc1 min volume size | **125 GiB** | No |
| st1/sc1 max volume size | **16 TiB** | No |
| Max volumes per instance | Depends on instance type (~28 for most) | No |
| Fast Snapshot Restore: max volumes per snapshot per AZ | **10** simultaneously at full performance | Yes |
| Snapshot archive min retention | **90 days** | No |
| DLM max policies per Region | **100** | Yes |

---

## 2. Design Patterns & Best Practices

### When to Use Which Volume Type

| Scenario | Volume Type | Why |
|----------|------------|-----|
| General purpose boot volume | **gp3** | 3,000 IOPS + 125 MB/s included, 20% cheaper than gp2 |
| Dev/test environments | **gp3** | Cost-effective, consistent performance |
| Production database (moderate) | **gp3** (up to 16K IOPS) or **io2** (up to 64K) | gp3 if ≤16K IOPS sufficient; io2 for higher |
| Mission-critical database (SAP HANA, Oracle RAC) | **io2 Block Express** | 256K IOPS, sub-ms latency, 99.999% durability, up to 64 TiB |
| Shared volume across instances | **io1/io2 with Multi-Attach** | Only provisioned IOPS volumes support Multi-Attach |
| Big data, data warehouse, log processing | **st1** | Optimized for large sequential throughput |
| Coldest data, lowest cost block storage | **sc1** | Cheapest EBS volume type |
| Highest possible IOPS on EC2 | **Instance Store** (not EBS) | Up to 3.3M IOPS. EBS maxes at 256K |

### Anti-Patterns

| Anti-Pattern | Why It's Wrong | Correct Approach |
|-------------|---------------|------------------|
| Using gp2 small volumes for sustained high IOPS | gp2 IOPS tied to size; small volume = low baseline + burst depletion | gp3 with independently provisioned IOPS |
| Using st1/sc1 as boot volume | **Not supported** — HDD cannot be boot volumes | gp3 or io2 for boot |
| Using io2 for workloads needing <3,000 IOPS | Over-provisioned and expensive | gp3 (3,000 IOPS included free) |
| Using EBS Multi-Attach with ext4/xfs | Data corruption — not cluster-aware file systems | Use cluster-aware FS (GFS2, OCFS2) or application-level locking |
| Storing temporary/cache data on EBS | Paying for persistence you don't need | Use instance store (free, highest IOPS) |
| Creating manual snapshots without DLM or AWS Backup | Operational burden, forgotten snapshots, no lifecycle | Automate with DLM or AWS Backup |
| Leaving FSR enabled after initial restore | $0.75/hr/AZ/snapshot adds up silently | Disable FSR once volumes are initialized |
| Sharing encrypted snapshots with aws/ebs key | AWS-managed key **cannot** be shared cross-account | Re-encrypt with a CMK before sharing |

### Common Architectural Patterns

#### Database HA with Multi-Attach
```
io2 Volume (Multi-Attach enabled)
  ├── EC2 Instance A (primary writer)
  └── EC2 Instance B (standby reader)
Both in same AZ, cluster-aware file system (GFS2)
Failover: Instance B becomes writer if A fails
```

#### Cross-Region DR with Snapshots
```
Production (us-east-1):
  EC2 + EBS io2 volume → DLM policy: daily snapshots → auto-copy to eu-west-1

DR (eu-west-1):
  Encrypted snapshot copy → restore to new io2 volume → attach to standby EC2
  Optional: FSR enabled in DR AZs for instant restore performance
```

#### Migration: Unencrypted to Encrypted
```
Unencrypted Volume → Snapshot → Copy Snapshot (enable encryption, specify CMK)
  → New Encrypted Volume → Attach to instance → Detach/Delete old volume
```

#### Cost-Optimized Data Lake Storage
```
Hot data:     EBS gp3 (attached to EMR instances)
Warm data:    EBS st1 (throughput-optimized, sequential reads)
Archive:      EBS snapshots → Snapshot Archive tier (75% cheaper)
Long-term:    S3 Glacier Deep Archive (via data pipeline)
```

### Well-Architected Framework Alignment

**Reliability**
- EBS volumes are replicated within their AZ automatically.
- Use DLM or AWS Backup for automated snapshot schedules.
- Use cross-Region snapshot copies for DR.
- Enable Recycle Bin to protect against accidental snapshot deletion.
- io2/io2 Block Express: 99.999% durability — 10× more durable than gp3/io1.
- Use Elastic Volumes to resize/retype without downtime.

**Security**
- Enable EBS encryption by default at the account level.
- Use CMKs for cross-account sharing and granular key policies.
- Use SCPs to enforce encryption.
- Encrypt snapshots before sharing.
- Use IAM policies to restrict snapshot sharing scope.

**Performance**
- gp3: provision IOPS and throughput independently for right-sizing.
- io2 Block Express for mission-critical sub-millisecond workloads.
- Use FSR for time-sensitive restores from snapshots.
- Ensure EBS-optimized instances (all Nitro by default).
- Monitor `VolumeQueueLength` and `BurstBalance` CloudWatch metrics.

**Cost Optimization**
- gp3 over gp2: 20% cheaper storage, better performance defaults.
- Don't over-provision io2 IOPS — monitor actual usage.
- Use st1/sc1 for throughput/cold workloads instead of gp3.
- Use Snapshot Archive for long-term retention (75% savings).
- Delete orphaned volumes and unused snapshots.
- Disable FSR when not needed.

**Operational Excellence**
- Automate snapshot management with DLM.
- Use AWS Backup for cross-service, cross-account, cross-Region policies.
- Tag volumes for cost allocation and lifecycle management.
- Use CloudWatch alarms on `VolumeQueueLength` > 1 (indicates bottleneck).
- Use Elastic Volumes for online modifications.

---

## 3. Security & Compliance

### IAM Policies

#### Key IAM Actions

| Action | What It Controls |
|--------|-----------------|
| `ec2:CreateVolume` | Who can create volumes |
| `ec2:AttachVolume` | Who can attach volumes to instances |
| `ec2:DetachVolume` | Who can detach volumes |
| `ec2:DeleteVolume` | Who can delete volumes |
| `ec2:CreateSnapshot` | Who can create snapshots |
| `ec2:DeleteSnapshot` | Who can delete snapshots |
| `ec2:CopySnapshot` | Who can copy snapshots (including cross-Region) |
| `ec2:ModifySnapshotAttribute` | Who can share snapshots with other accounts |
| `ec2:ModifyVolume` | Who can modify volume type/size/IOPS (Elastic Volumes) |
| `ec2:EnableEbsEncryptionByDefault` | Who can enable default encryption |

#### Condition Keys for EBS

```json
{
  "Effect": "Deny",
  "Action": "ec2:CreateVolume",
  "Resource": "arn:aws:ec2:*:*:volume/*",
  "Condition": {
    "Bool": { "ec2:Encrypted": "false" }
  }
}
```

| Condition Key | Purpose |
|--------------|---------|
| `ec2:Encrypted` | Enforce encryption on new volumes |
| `ec2:VolumeType` | Restrict to specific volume types |
| `ec2:VolumeSize` | Restrict maximum volume size |
| `ec2:VolumeIops` | Restrict provisioned IOPS |
| `ec2:VolumeThroughput` | Restrict provisioned throughput |
| `kms:ViaService` | Restrict KMS key usage to specific services (e.g., `ec2.us-east-1.amazonaws.com`) |

#### SCP: Enforce Encryption Across Organization

```json
{
  "Effect": "Deny",
  "Action": "ec2:CreateVolume",
  "Resource": "*",
  "Condition": {
    "Bool": { "ec2:Encrypted": "false" }
  }
}
```

#### SCP: Prevent Sharing Snapshots Publicly

```json
{
  "Effect": "Deny",
  "Action": "ec2:ModifySnapshotAttribute",
  "Resource": "*",
  "Condition": {
    "StringEquals": { "ec2:Add/group": "all" }
  }
}
```

### Encryption Details

| Scenario | Encryption Behavior |
|----------|-------------------|
| Create new volume | Uses default key (`aws/ebs`) or specified CMK |
| Snapshot of encrypted volume | Always encrypted with same key |
| Volume from encrypted snapshot | Always encrypted (same key or specify different) |
| Copy snapshot (same Region) | Can change encryption key |
| Copy snapshot (cross-Region) | **Must specify key in destination Region** (KMS keys are Regional) |
| Share snapshot cross-account | Must use **CMK** (not aws/ebs). Must share KMS key AND snapshot |
| Unencrypted → Encrypted | Snapshot → Copy (encrypt) → New volume |
| Encrypted → Unencrypted | Snapshot → Copy (decrypt) → New volume (only with CMK you control) |
| Enable encryption by default | All new volumes in the Region are automatically encrypted |
| Instance with encrypted root | **Can launch from encrypted AMI** or enable default encryption |

#### KMS Key Policy Requirements for EBS

The IAM principal (user/role) must have these KMS permissions:
- `kms:CreateGrant` — needed for EBS to use the key for data key generation
- `kms:Decrypt` — decrypt the encrypted data key
- `kms:DescribeKey` — verify the key exists and is enabled
- `kms:GenerateDataKeyWithoutPlaintext` — generate data keys for encryption
- `kms:ReEncrypt*` — for cross-key operations (copy with re-encryption)

### Logging and Monitoring

| Tool | What It Provides |
|------|-----------------|
| **CloudWatch EBS Metrics** | VolumeReadOps, VolumeWriteOps, VolumeReadBytes, VolumeWriteBytes, **VolumeQueueLength** (>1 means bottleneck), **BurstBalance** (gp2/st1/sc1), VolumeThroughputPercentage, VolumeConsumedReadWriteOps |
| **CloudTrail** | All EBS API calls (CreateVolume, DeleteSnapshot, ModifyVolume, etc.) |
| **AWS Config Rules** | `encrypted-volumes`, `ec2-volume-inuse-check`, `ebs-snapshot-public-restorable-check`, `ebs-optimized-instance` |
| **AWS Backup audit** | Compliance reporting for backup policies |
| **S3 Storage Lens** | N/A for EBS (S3 only) |

#### Critical CloudWatch Metrics

| Metric | Warning Threshold | Meaning |
|--------|------------------|---------|
| `VolumeQueueLength` | >1 sustained | I/O requests queuing — volume is a bottleneck |
| `BurstBalance` | <20% | gp2/st1/sc1 burst credits depleting — switch to gp3 or increase size |
| `VolumeReadLatency` / `VolumeWriteLatency` | >10ms for SSD | Abnormal latency |
| `VolumeThroughputPercentage` | >80% sustained | Approaching throughput limit |
| `VolumeConsumedReadWriteOps` | Close to provisioned | io1/io2 IOPS nearing limit |

---

## 4. Cost Optimization

### Pricing Model (us-east-1)

| Volume Type | Storage ($/GiB/month) | IOPS ($/IOPS/month) | Throughput ($/MB/s/month) |
|-------------|----------------------|---------------------|--------------------------|
| **gp3** | $0.08 | $0.005 (above 3,000) | $0.04 (above 125 MB/s) |
| **gp2** | $0.10 | Included (tied to size) | Included |
| **io2** | $0.125 | $0.065 (up to 32K), $0.046 (32K–64K) | Included |
| **io2 Block Express** | $0.125 | $0.065 (tiered pricing) | Included |
| **io1** | $0.125 | $0.065 | Included |
| **st1** | $0.045 | Included | Included |
| **sc1** | $0.015 | Included | Included |

#### Snapshot Pricing

| Type | Price (us-east-1) |
|------|-------------------|
| Standard snapshot storage | $0.05/GB/month |
| Snapshot Archive | **$0.0125/GB/month** (75% cheaper) |
| Snapshot Archive retrieval | $0.03/GB |
| Fast Snapshot Restore | $0.75/snapshot/AZ/hour |
| Snapshot copy (cross-Region) | Data transfer charges + storage in destination |

### Cost-Saving Strategies

1. **gp3 over gp2**: 20% cheaper base storage ($0.08 vs $0.10/GiB) with same or better performance. **Single most common cost optimization on the exam.**

2. **Right-size provisioned IOPS**: Monitor actual IOPS usage with CloudWatch. Don't provision 64,000 IOPS on io2 if you only use 10,000. Consider gp3 for workloads needing ≤16,000 IOPS.

3. **Delete unattached volumes**: Use AWS Config rule `ec2-volume-inuse-check` to find orphaned volumes. These are charged even when not attached.

4. **Snapshot Archive for old snapshots**: Move snapshots accessed less than once per 90 days to Archive tier — 75% savings.

5. **DLM retention policies**: Avoid accumulating thousands of snapshots. Set max retention count or age-based expiration.

6. **Disable Fast Snapshot Restore**: FSR costs $0.75/hr per AZ per snapshot. For 3 AZs × 2 snapshots = $4.50/hr = $3,240/month. Enable only during DR exercises or initial deployments.

7. **st1 for throughput workloads**: $0.045/GiB vs $0.08/GiB for gp3. If the workload is sequential throughput (logs, data warehouse), st1 saves 44%.

8. **sc1 for cold data**: $0.015/GiB — cheapest block storage. Use for archival data that still needs block-level access.

9. **Instance store for temporary data**: Free (included in instance price). Use for caches, scratch, buffers. Eliminates EBS cost entirely for ephemeral data.

10. **Elastic Volumes for right-sizing**: Modify volume type/size/IOPS online — no need to over-provision from the start. Start with gp3 3,000 IOPS, scale up only if needed.

### Common Cost Traps

- **Orphaned EBS volumes**: Instance terminated but volumes with `DeleteOnTermination=false` remain. Charged until deleted.
- **Orphaned snapshots**: AMI deregistered but underlying snapshots not deleted. Manual cleanup required.
- **FSR left enabled**: $0.75/hr/AZ/snapshot adds up quickly. Easy to forget.
- **Over-provisioned io2**: Paying $0.065/IOPS/month for IOPS you don't use. 64,000 IOPS = $4,160/month in IOPS charges alone.
- **gp2 "hidden capacity cost"**: To get 16,000 IOPS on gp2, you need a 5,334 GiB volume ($533/month) even if you only use 100 GiB of data.
- **Snapshot storage growth**: Incremental, but long-running volumes with high churn generate large snapshot chains. Monitor with Cost Explorer.
- **Cross-Region snapshot copies**: Data transfer ($0.02/GB) + storage in destination Region. A 1 TB snapshot copied daily = ~$600/month in transfer alone.

### Cost Comparison: When to Use What

| Need | Cheapest Option |
|------|----------------|
| Boot volume, general purpose | gp3 |
| Database ≤16K IOPS | gp3 with provisioned IOPS |
| Database >16K IOPS | io2 (cost justified by performance need) |
| Database >64K IOPS | io2 Block Express |
| Sequential throughput workload | st1 |
| Cold block storage | sc1 |
| Temporary cache data | Instance Store (free) |
| Long-term snapshot retention | Snapshot Archive |

---

## 5. High Availability, Disaster Recovery & Resilience

### EBS Availability Model

- EBS volumes are **AZ-scoped** and replicated within the AZ. **Single AZ failure = volume unavailable.**
- EBS does NOT natively span multiple AZs (unlike S3 or EFS).
- For multi-AZ resilience, you must use **snapshots + restore** or **application-level replication**.

### HA Patterns

| Pattern | Implementation | RPO | RTO |
|---------|---------------|-----|-----|
| **Snapshot-based DR (same Region)** | DLM daily snapshots → restore in different AZ | Hours | Minutes (+ FSR) to Hours |
| **Cross-Region DR** | DLM snapshots → auto-copy cross-Region → restore in DR | Hours | Minutes (+ FSR) |
| **Multi-AZ with application replication** | Two EC2 instances in different AZs, each with own EBS, app-level sync replication (e.g., DRBD, MySQL replication) | Near-zero | Minutes (failover) |
| **Multi-Attach within AZ** | io1/io2 volume shared by up to 16 Nitro instances in same AZ. Cluster-aware FS | Near-zero | Seconds (within AZ only) |
| **AWS Backup cross-account** | Centralized backup account with cross-account vault → restore to any account | Hours | Minutes–Hours |
| **Pilot Light** | Snapshots in DR Region. Standby EC2 stopped. On failover: restore volumes, start EC2 | Hours | Minutes–Hours |

### Snapshot-Based Recovery Steps

1. **Identify latest snapshot** (DLM or AWS Backup managed)
2. **Create volume from snapshot** (specify AZ, optionally different volume type/size)
3. **Enable FSR** if low-RTO required (eliminates first-read penalty)
4. **Attach to EC2 instance** in the target AZ
5. **Mount and verify** data integrity

### Resilience Features

| Feature | Benefit |
|---------|---------|
| **Elastic Volumes** | Modify type/size/IOPS without downtime — respond to changing needs |
| **DLM** | Automated snapshot lifecycle with cross-Region/cross-account copy |
| **AWS Backup** | Centralized, policy-based backup with compliance reporting |
| **Recycle Bin** | Recover accidentally deleted snapshots |
| **Fast Snapshot Restore** | Eliminate first-read latency for time-sensitive restores |
| **Snapshot Archive** | Cost-effective long-term retention for compliance |
| **io2 99.999% durability** | 10× more durable than gp3/io1 — reduces volume failure risk |

### EBS vs Other Storage for DR

| Criteria | EBS | EFS | S3 |
|----------|-----|-----|-----|
| AZ scope | Single AZ | Multi-AZ (Regional) | Regional (multi-AZ) |
| Cross-Region replication | Snapshot copy | EFS Replication | S3 CRR |
| RPO achievable | Hours (snapshot frequency) | Minutes (EFS replication) | Minutes (CRR with RTC) |
| Block storage | Yes | No (file system) | No (object storage) |
| Database use | Primary choice | Not for databases | Not for databases |

---

## 6. Exam-Focused Section

### Straightforward Questions (7 examples)

**Q1:** What is the maximum IOPS supported by a single io2 Block Express volume?
**A:** **256,000 IOPS.** Requires specific Nitro instance types. Standard io2 maxes at 64,000.

**Q2:** A company needs the cheapest EBS volume for infrequently accessed log data that will be read sequentially during monthly audits. Which volume type should they use?
**A:** **sc1 (Cold HDD).** Cheapest EBS volume at $0.015/GiB/month. Suitable for infrequent sequential access. (st1 if they need better throughput during reads.)

**Q3:** Can st1 or sc1 volumes be used as boot volumes?
**A:** **No.** Only SSD-backed volumes (gp2, gp3, io1, io2) can be boot volumes.

**Q4:** What happens to data on an EBS root volume when the instance is terminated?
**A:** **Deleted by default.** The `DeleteOnTermination` attribute is `true` for root volumes. Additional volumes default to `false`.

**Q5:** A company has an unencrypted EBS volume and needs to encrypt it. Can they encrypt it in place?
**A:** **No.** They must snapshot the volume, copy the snapshot with encryption enabled, then create a new encrypted volume from the snapshot.

**Q6:** What is the minimum volume size for an st1 volume?
**A:** **125 GiB.** Both st1 and sc1 have a minimum of 125 GiB (HDD volumes cannot be smaller).

**Q7:** How many instances can an io2 Multi-Attach volume be attached to simultaneously?
**A:** **16 Nitro instances** in the same AZ.

---

### Tricky / Scenario-Based Questions (7 examples)

**Q1:** A company runs a database on EC2 with a 100 GiB gp2 volume. Users report sporadic slowdowns during peak hours. CloudWatch shows `BurstBalance` dropping to 0% during peaks. The database requires sustained 5,000 IOPS. What is the MOST cost-effective solution?

**A:** **Switch to gp3 and provision 5,000 IOPS.**
- **Why not increase gp2 size?** To get 5,000 baseline IOPS on gp2, you need ~1,667 GiB (5,000 ÷ 3 IOPS/GiB). That's paying for 1,567 GiB of unnecessary storage at $0.10/GiB = $156.70/month wasted.
- **gp3 cost:** 100 GiB × $0.08 + 2,000 extra IOPS × $0.005 = $8 + $10 = **$18/month**. (3,000 IOPS included free, so only provision the additional 2,000.)
- **Why not io2?** Overkill. io2 costs $0.125/GiB + $0.065/IOPS — far more expensive for only 5,000 IOPS.
- **Key phrases:** "gp2" + "BurstBalance dropping" + "sustained IOPS" + "cost-effective" → gp3.

**Q2:** A solutions architect needs to share an encrypted EBS snapshot with another AWS account in a different Region. The snapshot is encrypted with the default `aws/ebs` key. What steps must they take?

**A:** (1) **Copy the snapshot with a CMK** (cannot share aws/ebs key cross-account). (2) **Share the CMK** with the target account via KMS key policy. (3) **Share the re-encrypted snapshot** with the target account. (4) Target account **copies the snapshot to their Region** (specifying their own KMS key in the destination Region, since KMS keys are Regional).
- **Why can't they share directly?** The `aws/ebs` key is AWS-managed and cannot be shared with other accounts.
- **Why copy to destination Region?** KMS keys are Region-specific. The shared CMK only works in the source Region. The target account needs to copy and re-encrypt with a key in their Region.
- **Key phrases:** "aws/ebs" + "cross-account" + "cross-Region" → Re-encrypt with CMK, share key, copy cross-Region with destination key.

**Q3:** A company restores an EBS volume from a snapshot for a production database. Immediately after launching, the database experiences extremely high read latency on the first queries. Subsequent reads are fast. What should they have done?

**A:** **Enabled Fast Snapshot Restore (FSR)** for the snapshot in the target AZ before creating the volume.
- **Why this happens:** Volumes restored from snapshots load blocks lazily from S3. First read of each block incurs high latency. FSR pre-warms the snapshot data.
- **Why not just use a bigger volume?** Volume size doesn't affect first-read latency — the issue is lazy loading from S3.
- **Alternative:** Initialize the volume by reading all blocks with `dd` or `fio` before production use (free but time-consuming).
- **Key phrases:** "restore from snapshot" + "first read latency" + "subsequent reads fast" → Fast Snapshot Restore.

**Q4:** A database workload requires 200,000 IOPS with sub-millisecond latency and 99.999% volume durability. The team proposes RAID 0 across four io2 volumes (50,000 IOPS each). Is there a better approach?

**A:** **Use a single io2 Block Express volume with 200,000 IOPS provisioned.** No RAID needed.
- **Why not RAID 0?** Adds operational complexity, risk (one volume failure = total data loss), and complicates snapshots (must coordinate across 4 volumes). io2 Block Express supports 256,000 IOPS on a single volume.
- **Why io2 Block Express specifically?** Standard io2 maxes at 64,000 IOPS per volume. Only Block Express reaches 256,000. Requires specific Nitro instance types (R5b, R6i, etc.).
- **Key phrases:** "200,000 IOPS" + "sub-millisecond" + "99.999% durability" → io2 Block Express (single volume).

**Q5:** A company has 500 EC2 instances each with a gp3 volume. They want automated daily snapshots with 30-day retention and automatic cross-Region copy for DR. What is the best approach?

**A:** **Amazon Data Lifecycle Manager (DLM)** with a snapshot lifecycle policy. Configure: tag-based targeting, daily schedule, 30-day retention, cross-Region copy rule.
- **Why not AWS Backup?** AWS Backup also works, but DLM is simpler for EBS-only snapshot lifecycle and is the native EBS solution. For multi-service backup policies, AWS Backup is better.
- **Why not a Lambda-based solution?** Over-engineering. DLM is purpose-built for this exact use case.
- **Key phrases:** "automated snapshots" + "retention" + "cross-Region copy" → DLM (or AWS Backup).

**Q6:** An architect enables EBS encryption by default in a Region. An existing unencrypted volume in that Region is still unencrypted. Why?

**A:** **Encryption by default only applies to NEW volumes.** Existing volumes are NOT retroactively encrypted. The architect must manually encrypt existing volumes using the snapshot-copy-restore process.
- **Exam trap:** Candidates assume "encryption by default" encrypts everything. It only applies to volumes created after the setting is enabled.
- **Key phrases:** "encryption by default" + "existing volume still unencrypted" → Only applies to new volumes.

**Q7:** A company uses io2 volumes with Multi-Attach for a clustered database across 3 instances. They want to extend this cluster to a 4th instance in a different AZ for HA. Can they?

**A:** **No.** Multi-Attach only works within a **single AZ**. All attached instances must be in the same AZ as the volume. EBS volumes are AZ-scoped.
- **For cross-AZ HA:** Use application-level replication (database replication, DRBD), or use a managed service like RDS Multi-AZ or Aurora.
- **Key phrases:** "Multi-Attach" + "different AZ" → Not supported. EBS is AZ-scoped.

---

### Common Exam Traps & Pitfalls

1. **gp2 IOPS is size-dependent; gp3 is independent**: The most tested EBS fact. If a gp2 volume has performance issues, the answer is almost always gp3.

2. **DeleteOnTermination defaults**: Root volume = **true** (deleted). Additional volumes = **false** (retained). Questions test this frequently.

3. **st1/sc1 cannot be boot volumes**: Any answer proposing HDD as a boot volume is wrong.

4. **EBS is AZ-scoped**: Cannot attach cross-AZ. Cannot Multi-Attach cross-AZ. Must snapshot and restore in the target AZ.

5. **Multi-Attach requires cluster-aware file system**: ext4, xfs, NTFS will cause **data corruption** with Multi-Attach. Must use GFS2, OCFS2, or application-level coordination.

6. **Encryption by default is per Region, not per account globally**: Must be enabled in each Region separately.

7. **aws/ebs key cannot be shared cross-account**: Must re-encrypt with a CMK before sharing snapshots.

8. **KMS keys are Regional**: Cross-Region snapshot copy requires specifying a key in the destination Region.

9. **Snapshots are incremental but you can delete any snapshot safely**: AWS manages block references. Deleting snapshot #2 of #5 doesn't lose data — required blocks are preserved in remaining snapshots.

10. **Fast Snapshot Restore costs $0.75/hr per AZ per snapshot**: Not per volume — per snapshot enablement. Costs accumulate quickly across multiple AZs and snapshots.

11. **io2 Block Express requires specific Nitro instances**: Not all Nitro instances support Block Express. Check compatibility (R5b, R6i, X2idn, etc.).

12. **io2 vs io1 durability**: io2 = 99.999% (0.001% AFR). io1 = 99.8%–99.9% (0.1%–0.2% AFR). 100× difference. If the question mentions "highest durability," the answer is io2.

13. **Elastic Volumes can change type/size/IOPS online**: No need to detach, stop instance, or create new volume. But there's a **6-hour cooldown** between modifications.

14. **Instance store vs EBS**: Highest possible IOPS = instance store (3.3M). If the question says "highest IOPS" and data is ephemeral, the answer is NOT EBS.

15. **Snapshot Archive 90-day minimum**: Charged for 90 days even if restored after 10 days. Don't archive snapshots you'll need soon.

---

## 7. Cheat Sheet

### Must-Know Facts for Exam Day

| Fact | Value |
|------|-------|
| gp3 baseline IOPS | **3,000** (independent of size) |
| gp3 baseline throughput | **125 MB/s** (independent of size) |
| gp3 max IOPS | **16,000** |
| gp3 max throughput | **1,000 MB/s** |
| gp2 IOPS formula | **3 IOPS/GiB** (burst to 3,000) |
| gp3 price | **$0.08/GiB/month** (20% cheaper than gp2) |
| io2 max IOPS | **64,000** (standard) / **256,000** (Block Express) |
| io2 Block Express max size | **64 TiB** |
| io2 Block Express latency | **Sub-millisecond** |
| io2 durability | **99.999%** (0.001% AFR) |
| st1/sc1 boot volume | **No** |
| st1/sc1 min size | **125 GiB** |
| Multi-Attach | **io1/io2 only, 16 Nitro instances, same AZ** |
| DeleteOnTermination (root) | **true** by default |
| DeleteOnTermination (additional) | **false** by default |
| Volumes per Region | **5,000** default |
| Snapshots per Region | **100,000** default |
| FSR cost | **$0.75/snapshot/AZ/hour** |
| Snapshot Archive savings | **75%** cheaper |
| Snapshot Archive min retention | **90 days** |
| Elastic Volume modification cooldown | **6 hours** |
| Encryption by default scope | **Per Region, new volumes only** |
| aws/ebs key sharing | **Cannot share cross-account** |

### Key Differentiators

| gp3 | gp2 |
|-----|-----|
| IOPS independent of size | IOPS tied to size (3/GiB) |
| 3,000 IOPS baseline | Burst credits model |
| $0.08/GiB | $0.10/GiB |
| 1,000 MB/s max throughput | 250 MB/s max throughput |
| Preferred for new workloads | Legacy |

| io2 | io2 Block Express |
|-----|-------------------|
| 64,000 IOPS | 256,000 IOPS |
| 16 TiB max | 64 TiB max |
| Single-digit ms latency | Sub-millisecond latency |
| Most Nitro instances | Specific Nitro types only |
| 500:1 IOPS:GiB | 1,000:1 IOPS:GiB |

| EBS | Instance Store |
|-----|---------------|
| Persistent | Ephemeral |
| 256K IOPS max | 3.3M IOPS max |
| Snapshots supported | No snapshots |
| AZ-scoped, attachable | Fixed to instance |
| Separate cost | Included in instance price |
| Resizable (Elastic Volumes) | Fixed size |

| DLM | AWS Backup |
|-----|-----------|
| EBS-native snapshot lifecycle | Multi-service backup (EBS, RDS, EFS, DynamoDB, etc.) |
| Tag-based, schedule-based | Policy-based, cross-account vault |
| Cross-Region copy | Cross-Region, cross-account |
| Simpler for EBS-only | Better for centralized backup strategy |

### Decision Flowchart

```
Question says "general purpose" or "boot volume" or "cost-effective"?
  → gp3

Question says "gp2 performance issues" or "BurstBalance dropping"?
  → Switch to gp3 (provision IOPS independently)

Question says "sustained IOPS ≤16,000"?
  → gp3 with provisioned IOPS

Question says "sustained IOPS >16,000 and ≤64,000"?
  → io2

Question says ">64,000 IOPS" or "sub-millisecond latency" or ">16 TiB volume"?
  → io2 Block Express

Question says "99.999% volume durability"?
  → io2 or io2 Block Express

Question says "shared volume across multiple instances"?
  → io1/io2 Multi-Attach + cluster-aware file system + SAME AZ

Question says "sequential throughput" + "big data / logs / data warehouse"?
  → st1

Question says "lowest cost block storage" + "cold data"?
  → sc1

Question says "highest possible IOPS" + "ephemeral/temporary data"?
  → Instance Store (NOT EBS)

Question says "automate snapshot lifecycle"?
  → DLM (EBS-only) or AWS Backup (multi-service)

Question says "restore from snapshot" + "immediate full performance"?
  → Fast Snapshot Restore (FSR)

Question says "long-term snapshot retention" + "cheapest"?
  → EBS Snapshot Archive (75% savings, 24-72hr restore)

Question says "protect snapshots from accidental deletion"?
  → Recycle Bin

Question says "encrypt existing unencrypted volume"?
  → Snapshot → Copy (encrypted) → New volume

Question says "share snapshot cross-account" + "encrypted"?
  → Must use CMK (not aws/ebs) + share KMS key + share snapshot

Question says "share snapshot cross-Region"?
  → Copy snapshot to destination Region (specify destination KMS key)

Question says "Multi-Attach" + "different AZs"?
  → NOT possible. EBS is AZ-scoped. Use application replication.

Question says "volume type change without downtime"?
  → Elastic Volumes (6-hour cooldown between changes)
```

---

*Generated for AWS SAP-C02 exam preparation. Last updated: 2026-04-26.*
