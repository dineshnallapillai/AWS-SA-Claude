# AWS SAP-C02 Study Notes: EC2 (Placement Groups, Nitro, AMIs, Instance Store vs EBS)

---

## 1. Core Concepts & Theory

### EC2 Fundamentals (SAP-Level Depth)

Amazon EC2 provides resizable virtual servers (instances) running on AWS hypervisors. At the SAP level, you must understand the underlying infrastructure decisions that affect performance, resilience, and cost.

---

### Placement Groups

Placement groups control **how EC2 instances are physically placed** on underlying hardware within an AZ.

#### Three Types

| Type | Placement Logic | Max Instances | Use Case |
|------|----------------|---------------|----------|
| **Cluster** | All instances on same rack / close hardware in a **single AZ** | No hard limit, but best launched in a single request | HPC, low-latency inter-node communication, big data jobs |
| **Spread** | Each instance on **distinct underlying hardware** (different racks, different power/network) | **7 instances per AZ per group** (hard limit) | Critical instances that must be isolated from correlated failure |
| **Partition** | Instances divided into **logical partitions**, each on separate racks | **7 partitions per AZ**, no limit on instances per partition | Large distributed/replicated workloads: HDFS, HBase, Cassandra, Kafka |

#### Key Details for the Exam

- **Cluster placement groups** support **Enhanced Networking** and can achieve **up to 10 Gbps within the group** (25 Gbps with ENA on Nitro instances). Instances MUST be in the **same AZ**.
- **Spread placement groups** can span **multiple AZs within the same Region**. The 7-per-AZ limit is absolute — you cannot request a waiver.
- **Partition placement groups** can span multiple AZs. You can query partition metadata via the instance metadata service to find which partition an instance is in.
- You **cannot merge** placement groups.
- An instance can only be in **one** placement group at a time.
- You can **move an existing instance** into a placement group only if it is **stopped** first (via CLI/API, not console for some types).
- **Dedicated Hosts** are NOT supported in placement groups.
- Not all instance types are supported in all placement group types. **Cluster** placement groups require instances that support Enhanced Networking.
- **Capacity error**: If you get an `InsufficientInstanceCapacity` error in a cluster placement group, stop and restart ALL instances in the group — AWS may need to find a new rack that can fit all of them.

---

### AWS Nitro System

The **Nitro System** is the underlying platform for all modern EC2 instances. It offloads virtualization functions to dedicated hardware and software.

#### Components

| Component | Function |
|-----------|----------|
| **Nitro Cards** | Handle VPC networking, EBS I/O, instance storage, and monitoring — all offloaded from the host CPU |
| **Nitro Security Chip** | Integrated on the motherboard; continuously monitors and protects hardware; prevents any operator (including AWS employees) from accessing instance memory or storage |
| **Nitro Hypervisor** | Lightweight hypervisor (~zero overhead). On bare-metal instances (`.metal`), no hypervisor is present — you get the full hardware |
| **Nitro Enclaves** | Isolated compute environments (virtual machines within a VM) for processing highly sensitive data (PII, healthcare, financial, DRM). Uses cryptographic attestation. No persistent storage, no admin access, no external networking |

#### Why Nitro Matters for the Exam

- **Performance**: Near bare-metal performance because networking/storage I/O is offloaded to Nitro Cards.
- **EBS Optimized by default**: All Nitro-based instances are EBS-optimized at no extra cost.
- **Higher EBS bandwidth**: Nitro instances support up to **64,000 IOPS** and **4,000 MB/s** throughput for io2 Block Express volumes.
- **Enhanced Networking (ENA)**: Nitro instances support Elastic Network Adapter with up to **100 Gbps** networking.
- **Bare metal (.metal)**: Runs directly on Nitro hardware — allows running your own hypervisor (VMware, KVM), needed for license-bound software.
- **Nitro Enclaves** are the answer when the question mentions "isolated processing of sensitive data" on EC2. Not the same as AWS CloudHSM (key management) or AWS Clean Rooms (data sharing).
- Instance families: C5, M5, R5, T3, and all newer generations (C6g, M6i, R6i, etc.) are Nitro-based. **C4, M4, R4, T2 are NOT Nitro**.

---

### AMIs (Amazon Machine Images)

An AMI is a **template** containing the OS, application server, applications, and launch permissions used to launch instances.

#### AMI Components

- **Root volume template** (EBS snapshot or instance store template)
- **Launch permissions** — who can use the AMI
- **Block device mapping** — which volumes to attach at launch

#### AMI Types by Root Device

| Type | Root Volume | Boot Time | Data Persistence | Stop/Start |
|------|-------------|-----------|-------------------|------------|
| **EBS-backed** | EBS snapshot | Faster (~1 min) | Persists after stop/terminate (if DeleteOnTermination=false) | Supported |
| **Instance-store-backed** | Template in S3 | Slower (~5 min) | **Lost on stop or terminate** | **Cannot stop — only terminate or reboot** |

#### AMI Scope and Sharing

- AMIs are **Region-specific**. To use in another Region, you must **copy** the AMI.
- You can share AMIs with specific AWS accounts, make them public, or share with an AWS Organization.
- **Encrypted AMIs** can be shared, but you must also share the **KMS key** used for encryption with the target account.
- You **cannot share** an AMI with an encrypted snapshot if the encryption key is the **default AWS-managed key** (`aws/ebs`). You must re-encrypt with a **Customer Managed Key (CMK)** first.
- **Cross-account copy**: The target account copies the AMI, creating a new AMI owned by that account. They can re-encrypt with their own key.

#### AMI Lifecycle & Deprecation

- AMIs can be **deprecated** (still launchable but shows warning) or **deregistered** (cannot launch new instances).
- EBS snapshots backing a deregistered AMI are NOT automatically deleted — you must delete them manually.
- **EC2 Image Builder** automates AMI creation, testing, and distribution pipelines. Integrates with SSM and can distribute AMIs cross-Region and cross-account.
- **Golden AMI pattern**: Pre-baked AMIs with OS patches, hardening, and application dependencies. Faster boot than bootstrapping with user data. Combine with user data for dynamic config.

#### Key Limits

- Default limit: **50,000 public AMIs** per Region (rarely tested).
- AMI copy is asynchronous — large AMIs with big snapshots take time.

---

### Instance Store vs EBS

This is one of the **highest-yield** topics on the SAP-C02 exam.

#### Instance Store (Ephemeral Storage)

- **Physically attached** to the host machine (local NVMe SSDs or HDDs).
- **Highest IOPS** available on EC2 — up to **3.3 million random read IOPS** on i3en and i4i instances.
- Data is **lost** when: instance is stopped, terminated, or the underlying host fails. Data **survives reboot**.
- **Cannot be detached or reattached** to another instance.
- **Not included in AMI snapshots** — only the block device mapping placeholder is preserved.
- Size is determined by instance type — you cannot add more instance store volumes after launch.
- **No encryption at rest by default** on older instances; **NVMe instance store on Nitro** instances uses **hardware-level AES-256 encryption** by default (keys on the Nitro card, ephemeral, destroyed when instance is terminated/stopped).
- **You MUST specify instance store volumes at launch time** in the block device mapping or they are not attached (except for volumes specified in the AMI).

#### EBS (Elastic Block Store)

| Volume Type | IOPS (max) | Throughput (max) | Size | Use Case |
|-------------|-----------|-----------------|------|----------|
| **gp3** | 16,000 | 1,000 MB/s | 1 GiB – 16 TiB | General purpose, boot volumes, dev/test |
| **gp2** | 16,000 (burst to 3,000) | 250 MB/s | 1 GiB – 16 TiB | Legacy general purpose (gp3 is preferred) |
| **io2 Block Express** | **256,000** | **4,000 MB/s** | 4 GiB – 64 TiB | Mission-critical, largest databases (Nitro instances only) |
| **io2** | 64,000 | 1,000 MB/s | 4 GiB – 16 TiB | Databases requiring sustained IOPS |
| **io1** | 64,000 | 1,000 MB/s | 4 GiB – 16 TiB | Legacy provisioned IOPS |
| **st1** | 500 | 500 MB/s | 125 GiB – 16 TiB | Throughput-intensive: big data, data warehouse, log processing |
| **sc1** | 250 | 250 MB/s | 125 GiB – 16 TiB | Lowest cost cold storage, infrequent access |

#### Critical EBS Details

- EBS volumes are **AZ-scoped** — must be in the same AZ as the instance.
- **Multi-Attach** (io1/io2 only): Attach a single volume to **up to 16 Nitro instances** in the same AZ. Must use a cluster-aware file system (not ext4/xfs). **Does NOT work across AZs**.
- **EBS snapshots** are stored in **S3** (regionally), incremental, and can be copied cross-Region.
- **Fast Snapshot Restore (FSR)**: Eliminates latency on first read from a restored snapshot. Costs extra. Enabled per AZ per snapshot.
- **EBS Encryption**: Uses AWS KMS. If you enable encryption by default in a Region, all new volumes and snapshots are encrypted. **You cannot un-encrypt a volume** — you must create an unencrypted snapshot copy and create a new volume from that.
- **EBS-optimized instances**: Dedicated bandwidth for EBS I/O. All Nitro instances are EBS-optimized by default.
- **DeleteOnTermination**: Default is **true for root volume**, **false for additional volumes**. Critical exam detail — on terminate, root EBS is deleted unless you change this.
- **gp3 vs gp2**: gp3 lets you provision IOPS and throughput **independently of volume size**. gp2 IOPS scales with volume size (3 IOPS per GiB, burst baseline). gp3 is cheaper and more flexible — preferred answer on the exam.
- **io2 Block Express** is the answer when the question mentions >64,000 IOPS, sub-millisecond latency, or >16 TiB volume size. Requires Nitro instances.
- **st1 and sc1 CANNOT be boot volumes**.

#### Instance Store vs EBS Decision Matrix

| Criteria | Instance Store | EBS |
|----------|---------------|-----|
| Persistence | Ephemeral (lost on stop/terminate/failure) | Persistent |
| Performance ceiling | Millions of IOPS | 256K IOPS (io2 Block Express) |
| Boot volume | Yes (but cannot stop instance) | Yes |
| Snapshots | Not supported | Supported |
| Encryption | Hardware encryption on Nitro (NVMe) | KMS encryption |
| Multi-Attach | No | Yes (io1/io2 only) |
| Resize | Fixed per instance type | Can modify volume type, size, IOPS online |
| Cost | Included in instance price | Separate billing |
| Exam keyword | "temporary," "cache," "buffer," "highest IOPS" | "persistent," "durable," "snapshots," "encryption at rest with KMS" |

---

### Default Limits Worth Memorizing

| Resource | Default Limit | Adjustable? |
|----------|--------------|-------------|
| Running On-Demand instances per Region | Varies by instance type (vCPU-based limits) | Yes |
| Spread placement group: instances per AZ | **7** | **No** |
| Partition placement group: partitions per AZ | **7** | No |
| EBS volumes per Region | 5,000 | Yes |
| EBS snapshots per Region | 100,000 | Yes |
| io1/io2 IOPS per instance | 64,000 (256,000 with Block Express) | No |
| AMIs per Region | 50,000 | Yes |
| EBS Multi-Attach: instances per volume | 16 | No |

---

## 2. Design Patterns & Best Practices

### When to Use

| Pattern | Service/Feature | When to Use |
|---------|----------------|-------------|
| **HPC, tightly coupled** | Cluster Placement Group + Enhanced Networking | Low-latency, high-bandwidth inter-node communication (MPI, financial modeling) |
| **Critical isolation** | Spread Placement Group | Small set of critical instances (max 7 per AZ) that must survive correlated hardware failure |
| **Big data distributed** | Partition Placement Group | HDFS, Cassandra, Kafka — need rack-awareness without the 7-instance limit |
| **Fastest boot + consistent config** | Golden AMI | Production deployments requiring fast scaling; minimize user-data bootstrapping time |
| **Highest disk IOPS possible** | Instance Store (i3, i3en, i4i, d3) | Caches, temporary processing, scratch data, buffer/queue storage |
| **Persistent database storage** | EBS io2/io2 Block Express | Mission-critical databases needing durability and snapshots |
| **General purpose workloads** | EBS gp3 | Default for most workloads — boot volumes, app servers, dev/test |
| **Big data throughput** | EBS st1 | Sequential read/write workloads — EMR, data warehousing, log processing |
| **Sensitive data isolation** | Nitro Enclaves | Processing PII, healthcare data, financial data, DRM license validation in isolated environment |
| **Bring your own hypervisor** | Bare Metal (.metal) | VMware Cloud on AWS, licensing restrictions requiring physical hardware visibility |

### When NOT to Use (Anti-Patterns)

| Anti-Pattern | Why It's Wrong | Correct Approach |
|-------------|---------------|------------------|
| Cluster placement group across AZs | Not supported — cluster is single-AZ only | Use partition or spread if multi-AZ needed |
| Instance store for database storage | Data lost on stop/terminate/failure | EBS io2 with Multi-AZ or RDS |
| Spread placement group for 100+ instances | Hard limit of 7 per AZ | Use partition placement group |
| gp2 for high-IOPS small volumes | gp2 IOPS tied to volume size (3 IOPS/GiB) — small volume = low IOPS | Use gp3 (IOPS independent of size) or io2 |
| sc1/st1 as boot volume | Not supported | Use gp3 or io1/io2 |
| Copying AMIs with aws/ebs key cross-account | AWS-managed keys cannot be shared cross-account | Re-encrypt with CMK before sharing |
| EBS Multi-Attach with ext4/xfs | These file systems are not cluster-aware — data corruption will occur | Use a cluster file system (GFS2, OCFS2) or application-level coordination |

### Well-Architected Framework Alignment

**Reliability**
- Use spread placement groups for critical instances needing hardware fault isolation.
- Use EBS with snapshots for backup and recovery.
- Use EBS Multi-Attach for shared storage failover (with cluster file systems).
- Use multiple instance types in ASGs to avoid `InsufficientInstanceCapacity`.

**Performance Efficiency**
- Use cluster placement groups + Enhanced Networking for HPC.
- Use instance store for highest IOPS (caches, temporary processing).
- Use io2 Block Express for demanding database workloads.
- Match instance type to workload: compute-optimized (C-family), memory-optimized (R/X-family), storage-optimized (I/D-family).

**Security**
- Use encrypted EBS volumes with CMKs for data at rest.
- Use Nitro Enclaves for isolated sensitive data processing.
- Use EC2 Image Builder with hardened Golden AMIs for compliance.

**Cost Optimization**
- Use gp3 over gp2 (same or better performance, 20% cheaper).
- Instance store is included in instance price — no separate charge.
- Right-size io2 IOPS — don't over-provision.
- Use st1/sc1 for throughput-oriented cold data instead of gp3.

**Operational Excellence**
- Automate AMI creation with EC2 Image Builder.
- Tag instances in placement groups for operational visibility.
- Enable EBS default encryption at the account/Region level.

---

## 3. Security & Compliance

### IAM and Resource Policies

- **ec2:RunInstances** action can be constrained by condition keys:
  - `ec2:PlacementGroup` — restrict launching into specific placement groups
  - `ec2:InstanceType` — restrict to Nitro-based instance types
  - `ec2:VolumeType` — restrict EBS volume types (enforce io2 over io1)
  - `ec2:Encrypted` — **require** EBS encryption on launch (`"ec2:Encrypted": "true"`)

- **SCP example** — enforce EBS encryption across an Organization:
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

- **KMS key policy** must allow the IAM role to use the key for `kms:CreateGrant`, `kms:Decrypt`, `kms:GenerateDataKey*` for EBS encryption to work.

### Encryption

| What | Mechanism |
|------|-----------|
| EBS volumes at rest | AES-256 via AWS KMS (default or CMK) |
| EBS snapshots | Encrypted if source volume is encrypted |
| Instance store (Nitro NVMe) | Hardware AES-256 encryption; ephemeral keys destroyed on stop/terminate |
| Instance store (non-Nitro) | Not encrypted by default — use OS-level encryption (dm-crypt, LUKS) |
| AMI sharing with encryption | Must share CMK (not aws/ebs) with target account |
| EBS in transit | Encrypted between instance and EBS volume on Nitro instances |
| Nitro Enclaves | Cryptographic attestation; encrypted communication channel with parent instance |

### Logging and Monitoring

- **CloudWatch Metrics**: CPUUtilization, DiskReadOps, DiskWriteOps, NetworkIn/Out, StatusCheckFailed (instance and system).
- **CloudWatch Agent**: Required for memory utilization, disk space (OS-level metrics not available by default).
- **EBS CloudWatch Metrics**: VolumeReadOps, VolumeWriteOps, VolumeQueueLength, BurstBalance (gp2/st1/sc1).
- **CloudTrail**: Logs all EC2 API calls (RunInstances, StopInstances, CreateSnapshot, etc.).
- **VPC Flow Logs**: Network traffic logging for instances.
- **AWS Config Rules**: `encrypted-volumes`, `ec2-instance-no-public-ip`, `approved-amis-by-id`.
- **SSM Inventory/Patch Manager**: Track installed software and patch compliance across instances.

---

## 4. Cost Optimization

### Pricing Models

| Model | Discount vs On-Demand | Commitment | Best For |
|-------|----------------------|-----------|----------|
| **On-Demand** | Baseline | None | Short-term, unpredictable, testing |
| **Reserved Instances (RI)** | Up to 72% | 1 or 3 years | Steady-state, predictable workloads |
| **Savings Plans (Compute)** | Up to 66% | 1 or 3 years ($/hr commitment) | Flexible across instance families, Regions, OS |
| **Savings Plans (EC2 Instance)** | Up to 72% | 1 or 3 years (locked to family + Region) | Known instance family in known Region |
| **Spot Instances** | Up to 90% | None (can be interrupted with 2-min notice) | Fault-tolerant, batch, CI/CD, HPC |
| **Dedicated Hosts** | On-Demand or RI pricing | Optional 1/3-year RI | License compliance (per-socket, per-core) |
| **Dedicated Instances** | ~2% premium over On-Demand | None | Compliance requiring hardware isolation (not license-bound) |
| **Capacity Reservations** | No discount (pay On-Demand rate) | None | Guarantee capacity in specific AZ |

### Cost-Saving Strategies

1. **gp3 over gp2**: 20% cheaper baseline with better performance defaults (3,000 IOPS and 125 MB/s included).
2. **Instance store over EBS for temporary data**: No separate charge — included in instance price.
3. **Delete unattached EBS volumes**: Common cost leak. Use AWS Config rule `ec2-volume-inuse-check`.
4. **Snapshot lifecycle**: Use Amazon Data Lifecycle Manager (DLM) to automate snapshot creation/deletion.
5. **Fast Snapshot Restore costs**: FSR charges per AZ per snapshot per hour. Disable when not needed.
6. **Right-size provisioned IOPS**: Don't provision io2 at 64,000 IOPS if you use only 5,000.
7. **Use Spot for HPC cluster placement groups**: Spot works in cluster placement groups — massive savings for fault-tolerant HPC.
8. **AMI deregistration doesn't delete snapshots**: Orphaned snapshots continue to incur charges. Clean up manually.

### Common Cost Traps

- **EBS snapshots stored indefinitely** — incremental, but they accumulate.
- **Fast Snapshot Restore** — easy to forget it's enabled; $0.75/hr per AZ per snapshot.
- **Provisioned IOPS volumes (io1/io2)** — pay for provisioned IOPS whether you use them or not.
- **Dedicated Hosts** — paying for the whole host even if utilization is low. Use **Dedicated Instances** if you just need hardware isolation without per-core licensing.
- **Stopped EBS-backed instances** — you don't pay for compute, but you **still pay for EBS volumes and Elastic IPs**.

---

## 5. High Availability, Disaster Recovery & Resilience

### HA Architecture Patterns

| Pattern | Implementation |
|---------|---------------|
| **Multi-AZ** | Spread or partition placement groups spanning AZs + ALB + ASG |
| **Multi-Region** | AMI copy to DR Region + ASG + Route 53 failover routing |
| **Active-Active** | ASGs in multiple Regions/AZs behind Global Accelerator or Route 53 weighted/latency routing |
| **Active-Passive** | Primary Region active, DR Region with scaled-down ASG, Route 53 failover |

### RPO / RTO Considerations

| Strategy | RPO | RTO | Cost |
|----------|-----|-----|------|
| **EBS snapshots (periodic)** | Hours (depends on frequency) | Minutes–Hours (restore snapshot → launch instance) | Low |
| **EBS snapshots + FSR** | Hours | Minutes (no first-read penalty) | Medium |
| **EBS Multi-Attach failover** | Near-zero | Seconds–Minutes | High |
| **AMI copy cross-Region + ASG** | Hours–Days (depends on AMI refresh) | Minutes (launch from pre-copied AMI) | Medium |
| **Pilot Light (DR Region)** | Minutes–Hours | Minutes (scale up ASG) | Low–Medium |
| **Warm Standby** | Minutes | Minutes | Medium–High |

### Backup & Recovery Strategies

1. **EBS Snapshots + DLM**: Automate regular snapshots with retention policies. Snapshots are regional but can be copied cross-Region.
2. **Cross-Region AMI Copy**: Automate with EC2 Image Builder. Include AMI in DR launch templates.
3. **Instance Store Data**: **Not backed up by AWS**. Application must handle replication (e.g., HDFS replication factor, Cassandra replication). Consider writing critical data to S3 or EBS as well.
4. **EBS Fast Snapshot Restore**: Enable in DR AZs so restored volumes perform at full speed immediately — critical for low-RTO databases.
5. **ASG with mixed instances**: Use multiple instance types to reduce risk of capacity exhaustion in a single family.

### Resilience Patterns

- **Capacity Reservations + Savings Plans**: For critical workloads, reserve capacity in specific AZ to guarantee launch during regional events.
- **Spread Placement Group + ASG**: Each instance on separate hardware — one rack failure affects at most one instance.
- **EBS volume replacement**: If an EBS volume fails, detach, create new from latest snapshot, attach. Automate with Lambda + CloudWatch alarm on `StatusCheckFailed_AttachedEBS`.

---

## 6. Exam-Focused Section

### Straightforward Questions (7 examples)

**Q1:** A company needs the lowest latency between EC2 instances for an HPC workload. Which placement group type should they use?
**A:** **Cluster placement group.** It places instances on the same underlying hardware/rack in a single AZ for lowest network latency.

**Q2:** What is the maximum number of instances per AZ in a spread placement group?
**A:** **7.** This is a hard limit that cannot be increased.

**Q3:** An EBS-backed instance is terminated. What happens to the root EBS volume by default?
**A:** **It is deleted.** The default `DeleteOnTermination` attribute is `true` for root volumes.

**Q4:** Which EBS volume type supports Multi-Attach?
**A:** **io1 and io2** (Provisioned IOPS volumes only). Up to 16 Nitro instances in the same AZ.

**Q5:** What happens to data on an instance store volume when the instance is stopped?
**A:** **The data is lost.** Instance store is ephemeral; data persists only through reboots, not stops or terminations.

**Q6:** A company wants to share an encrypted AMI with another AWS account. The AMI was encrypted with the default `aws/ebs` key. What must they do?
**A:** **Re-encrypt the AMI with a Customer Managed Key (CMK)**, then share both the AMI and the CMK with the target account. The default AWS-managed key cannot be shared.

**Q7:** Which EC2 feature provides an isolated compute environment for processing sensitive data with cryptographic attestation?
**A:** **AWS Nitro Enclaves.** They create isolated VMs within an instance with no persistent storage, no external networking, and cryptographic attestation.

---

### Tricky / Scenario-Based Questions (7 examples)

**Q1:** A company runs a distributed Cassandra cluster on EC2 with 50 nodes across 3 AZs. They need rack-awareness for data replication but also need hardware fault isolation between groups of nodes. Which placement group type should they use?

**A:** **Partition placement group.**
- **Why not cluster?** Cluster is single-AZ only and doesn't provide rack awareness — all nodes on the same rack.
- **Why not spread?** Spread supports only **7 instances per AZ** — you have 50 nodes. Even across 3 AZs, that's only 21.
- **Key phrase:** "rack-awareness" + "50 nodes" + "distributed" → Partition.

**Q2:** An application requires 200,000 IOPS with sub-millisecond latency on a single EBS volume. The company uses R5 instances. What should the architect recommend?

**A:** **Migrate to a Nitro-based R6i (or similar) instance and use io2 Block Express volumes.**
- **Why not io2 on R5?** io2 standard maxes out at 64,000 IOPS. io2 Block Express supports 256,000 IOPS but requires Nitro instances. R5 IS Nitro, but **io2 Block Express requires specific newer Nitro instance types** (R5b, R6i, etc.) — check compatibility.
- **Why not instance store?** The question says "single EBS volume" — it requires persistence. Also, instance store doesn't support snapshots.
- **Key phrase:** "200,000 IOPS" + "single EBS volume" → io2 Block Express on compatible Nitro instance.

**Q3:** A company has an Auto Scaling group using a cluster placement group. During a scale-out event, instances fail to launch with `InsufficientInstanceCapacity`. What should they do?

**A:** **Stop and restart all existing instances in the placement group, then retry the scale-out.**
- **Why this works:** AWS needs to find a contiguous block of capacity. Restarting all instances lets AWS re-place the entire group on hardware with enough capacity.
- **Better long-term fix:** Launch all desired capacity in a single request, or use multiple instance types in the ASG with an allocation strategy.
- **Why not just retry?** The capacity issue is about contiguous placement — retrying will hit the same constraint.

**Q4:** A solutions architect must design storage for an EC2-based real-time analytics engine that processes streaming data at 2 million IOPS. Data can be regenerated from the source stream. Cost must be minimized. What should they recommend?

**A:** **Instance store volumes on storage-optimized instances (i3en or i4i).**
- **Why not EBS io2 Block Express?** Max 256,000 IOPS per volume. Even with multiple volumes, 2 million IOPS is not achievable with EBS. Instance store can deliver **3.3 million IOPS** on i3en.
- **Why is this acceptable?** "Data can be regenerated" = ephemeral is OK. Instance store is included in instance price = cost-effective.
- **Key phrases:** "2 million IOPS" + "can be regenerated" + "minimize cost" → Instance store.

**Q5:** A company needs to run VMware workloads on AWS and requires access to the underlying physical hardware for VMware ESXi installation. They also need per-socket licensing visibility. Which EC2 option is appropriate?

**A:** **EC2 bare metal instances (.metal) on Dedicated Hosts.**
- **Why bare metal?** VMware ESXi requires direct hardware access — cannot run on a hypervisor.
- **Why Dedicated Hosts?** Per-socket licensing requires visibility into the physical host (sockets, cores). Dedicated Instances provide hardware isolation but NO visibility into physical host topology.
- **Key phrases:** "VMware" + "underlying physical hardware" + "per-socket licensing" → Bare metal + Dedicated Host. (In practice, VMware Cloud on AWS handles this, but the exam may test the underlying concepts.)

**Q6:** An application uses gp2 EBS volumes sized at 200 GiB. Users report inconsistent performance during peak hours. The baseline IOPS for these volumes is 600, but the workload requires a sustained 5,000 IOPS. What is the most cost-effective solution?

**A:** **Switch to gp3 volumes and provision 5,000 IOPS.**
- **Why not increase gp2 size?** You'd need ~1,667 GiB (5,000 / 3 IOPS per GiB) to get 5,000 baseline IOPS. That's paying for 1,467 GiB of unnecessary storage.
- **Why not io2?** Overkill and more expensive when gp3 supports up to 16,000 IOPS at a lower price.
- **Key phrases:** "gp2" + "inconsistent performance" + "sustained IOPS beyond baseline" + "cost-effective" → gp3 with provisioned IOPS.

**Q7:** A company creates an AMI from an instance that has both an EBS root volume and two instance store volumes. They launch a new instance from this AMI. What happens to the instance store data?

**A:** **The instance store data is NOT included in the AMI. New instance store volumes are empty (unformatted).**
- **Why?** AMIs capture EBS snapshots for EBS volumes. Instance store volumes are only recorded as block device mappings (placeholders) — the actual data is never captured.
- **Common trap:** Candidates assume AMI = full disk image of everything. It is not — instance store data must be backed up separately (e.g., to S3).
- **Key phrase:** "AMI" + "instance store" → Data not preserved.

---

### Common Exam Traps & Pitfalls

1. **Dedicated Host vs Dedicated Instance**: Dedicated Host = you control placement, see physical server topology, use for BYOL per-socket/per-core licensing. Dedicated Instance = hardware isolation only, no visibility into physical host, no BYOL licensing control.

2. **gp2 IOPS is size-dependent; gp3 IOPS is independent**: If the question mentions a small volume needing high IOPS, gp3 is the answer. If it mentions gp2 with poor performance, the fix is usually to switch to gp3.

3. **Cluster placement group is single-AZ ONLY**: If the question requires multi-AZ + low latency within a group, there is no perfect answer — cluster gives latency but is single-AZ. The question is testing whether you know this constraint.

4. **Instance store survives reboot but NOT stop**: This is a frequently tested nuance. If the question says the instance was "restarted" (ambiguous), look for other clues. If it says "stopped and started," data is lost.

5. **EBS Multi-Attach requires cluster-aware file system**: If the question mentions ext4 or xfs with Multi-Attach, it's a trap — that will cause data corruption.

6. **st1 and sc1 cannot be boot volumes**: If a question proposes using st1 as a boot volume for a throughput workload, it's wrong regardless of the throughput benefit.

7. **AMI sharing with encrypted snapshots**: Cannot share if encrypted with AWS-managed key. Must use CMK and share the key.

8. **Fast Snapshot Restore is per-AZ and per-snapshot**: If you need FSR in 3 AZs for 2 snapshots, that's 6 FSR activations at $0.75/hr each = $4.50/hr. It adds up.

9. **io2 Block Express vs io2**: Block Express supports up to 256K IOPS and 64 TiB volumes, but only on certain Nitro instances. The standard io2 maxes at 64K IOPS and 16 TiB.

10. **"Highest IOPS" could mean instance store**: If the question asks for absolute highest IOPS and durability is NOT a requirement, the answer is instance store, not io2 Block Express.

---

## 7. Cheat Sheet

### Must-Know Facts for Exam Day

| Fact | Value |
|------|-------|
| Spread placement group: max instances per AZ | **7** (hard limit) |
| Partition placement group: max partitions per AZ | **7** |
| gp3 baseline IOPS | **3,000** (provisioned independently of size) |
| gp3 baseline throughput | **125 MB/s** |
| gp2 IOPS formula | **3 IOPS per GiB**, burst to 3,000 |
| io2 Block Express max IOPS | **256,000** |
| io2 Block Express max throughput | **4,000 MB/s** |
| io2 Block Express max size | **64 TiB** |
| Instance store max IOPS | **3.3 million** (i3en/i4i) |
| EBS Multi-Attach max instances | **16** (Nitro, same AZ, io1/io2 only) |
| Nitro instances: EBS optimized | **Yes, by default** |
| Default DeleteOnTermination (root) | **true** |
| Default DeleteOnTermination (additional) | **false** |
| st1/sc1 as boot volume | **Not supported** |
| AMI scope | **Region-specific** |
| Cluster placement group scope | **Single AZ** |
| Nitro Enclaves | Isolated VM within instance, no persistent storage, no networking, cryptographic attestation |

### Key Differentiators

| Instance Store | EBS |
|---------------|-----|
| Ephemeral | Persistent |
| Millions of IOPS | Up to 256K IOPS |
| Included in instance price | Separate charge |
| Fixed to instance type | Resizable online |
| No snapshots | Snapshot support |

| gp3 | gp2 |
|-----|-----|
| IOPS/throughput independent of size | IOPS tied to volume size |
| 3,000 IOPS baseline included | 3 IOPS/GiB (burst pool) |
| 20% cheaper | Legacy |

| Dedicated Host | Dedicated Instance |
|---------------|-------------------|
| Physical server visibility | No server visibility |
| BYOL per-socket/per-core | No licensing advantage |
| You control placement | AWS controls placement |
| More expensive | Slight premium over On-Demand |

| Nitro Enclaves | CloudHSM | KMS |
|---------------|----------|-----|
| Isolated sensitive data processing | Dedicated HSM for key management | Managed key service |
| Cryptographic attestation | FIPS 140-2 Level 3 | FIPS 140-2 Level 2 (or 3 with CloudHSM backing) |
| Runs custom code in isolation | Crypto operations only | Key management API only |

### Decision Flowchart

```
Question says "highest IOPS" + "data can be regenerated/temporary"?
  → Instance Store

Question says "highest IOPS" + "persistent/durable/snapshots"?
  → io2 Block Express (on Nitro)

Question says "sub-millisecond latency" + ">64K IOPS" on single volume?
  → io2 Block Express

Question says "general purpose" or "boot volume" or "cost-effective"?
  → gp3

Question says "throughput optimized" + "sequential reads" + "big data/logs"?
  → st1

Question says "cold data" + "lowest cost" + "infrequent access"?
  → sc1

Question says "low latency between instances" + "HPC"?
  → Cluster Placement Group

Question says "hardware fault isolation" + ">7 instances"?
  → Partition Placement Group

Question says "hardware fault isolation" + "≤7 instances per AZ"?
  → Spread Placement Group

Question says "VMware" or "bare metal" or "bring your own hypervisor"?
  → .metal instances (+ Dedicated Host if licensing is mentioned)

Question says "process sensitive data in isolation on EC2"?
  → Nitro Enclaves

Question says "per-socket/per-core licensing"?
  → Dedicated Host

Question says "hardware isolation" (but no licensing mention)?
  → Dedicated Instance (cheaper than Dedicated Host)

Question says "share encrypted AMI cross-account"?
  → CMK (not aws/ebs) + share key + share AMI

Question says "inconsistent EBS performance" + "small gp2 volume"?
  → Switch to gp3

Question says "multiple instances sharing one volume"?
  → EBS Multi-Attach (io1/io2) + cluster-aware file system + same AZ
```

---

*Generated for AWS SAP-C02 exam preparation. Last updated: 2026-04-26.*
