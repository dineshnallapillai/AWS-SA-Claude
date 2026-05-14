# AWS SAP-C02 Study Notes: Snow Family, DataSync & Transfer Family

## 1. Core Concepts & Theory

### AWS Snow Family Overview

**Purpose:** Physical devices for offline data transfer, edge computing, and edge storage when network transfer is impractical.

| Device | Storage | Compute | Use Case |
|--------|---------|---------|----------|
| **Snowcone** | 8 TB HDD or 14 TB SSD | 2 vCPUs, 4 GB RAM | Small, portable; edge locations, IoT, tactical environments |
| **Snowball Edge Storage Optimized** | 80 TB usable (210 TB total) | 40 vCPUs, 80 GB RAM | Large data migration, edge storage |
| **Snowball Edge Compute Optimized** | 28 TB usable NVMe (42 TB HDD + 28 TB NVMe) | 104 vCPUs, 416 GB RAM, optional GPU | Edge computing, ML inference, video processing |
| **Snowmobile** | 100 PB per container | N/A | Exabyte-scale data center migration |

### AWS Snowcone

**Form Factor:** Smallest Snow device — 4.5 lbs (2.1 kg), ruggedized, portable

| Feature | Details |
|---------|---------|
| **Snowcone (HDD)** | 8 TB usable storage, HDD |
| **Snowcone SSD** | 14 TB usable storage, NVMe SSD |
| **Compute** | 2 vCPUs, 4 GB RAM |
| **Networking** | 2x 1GbE / 10GbE (Wi-Fi optional) |
| **Power** | USB-C or optional battery |
| **Encryption** | 256-bit encryption (AWS KMS) |
| **DataSync agent** | Pre-installed (can transfer data online if connectivity available) |
| **EC2 instances** | Can run sbe1.small instances (edge compute) |
| **Operating temp** | 0°C to 40°C (32°F to 104°F) |
| **Use cases** | IoT data collection, tactical edge, healthcare, content distribution |

**Key Differentiator:** Only Snow device with built-in DataSync agent — can transfer data back to AWS over the network (online) if bandwidth becomes available, OR ship the device (offline).

### AWS Snowball Edge

**Two Variants:**

**Snowball Edge Storage Optimized:**

| Feature | Details |
|---------|---------|
| **Usable storage** | 80 TB (HDD) + optional 1 TB NVMe for compute cache |
| **Compute** | 40 vCPUs, 80 GB RAM |
| **Networking** | 1x 10GbE (RJ45), 1x 10/25GbE (SFP28), 1x 25/100GbE (QSFP28) |
| **Encryption** | 256-bit (AWS KMS managed) |
| **Clustering** | Up to 15 devices in a local cluster (1.2 PB storage) |
| **S3 compatible** | S3-compatible endpoint on device (S3 adapter) |
| **NFS** | NFS file interface on device |
| **EC2** | Can run EC2 instances (sbe-c, sbe-g instance families) |
| **Use cases** | Large data migration, local storage with S3 interface, edge computing |

**Snowball Edge Compute Optimized:**

| Feature | Details |
|---------|---------|
| **Storage** | 28 TB NVMe usable (+ 42 TB HDD option) |
| **Compute** | 104 vCPUs, 416 GB RAM |
| **GPU** | Optional NVIDIA Tesla V100 GPU |
| **Networking** | Same as Storage Optimized |
| **EC2** | More powerful EC2 instances (up to sbe-c.24xlarge) |
| **Use cases** | ML inference at edge, video analysis, autonomous vehicles, disconnected environments |

**Snowball Edge Features (Both Variants):**
- **S3-compatible endpoint:** Applications use standard S3 APIs against the device
- **NFS mount:** Device appears as NFS file server to on-premises systems
- **EC2 compute:** Run AMIs on the device (deploy with OpsHub)
- **IoT Greengrass:** Run IoT Greengrass Core on device
- **Lambda@Edge:** Run Lambda functions on the device (local processing before transfer)
- **Clustering:** Group up to 15 devices for increased storage (Storage Optimized) or compute (Compute Optimized)
- **OpsHub:** GUI management tool for Snow devices (runs on local laptop)
- **Encryption:** All data encrypted with KMS key before leaving premises; keys never stored on device

### AWS Snowmobile

**Form Factor:** 45-foot shipping container pulled by a semi-trailer truck

| Feature | Details |
|---------|---------|
| **Storage** | Up to 100 PB per Snowmobile |
| **Networking** | High-speed (up to 1 Tbps) via multiple 40 Gbps connections |
| **Security** | GPS tracking, 24/7 video surveillance, escort security, dedicated fiber |
| **Encryption** | 256-bit KMS encryption; optional HSM on container |
| **Transit time** | ~1 week transfer + shipping time |
| **Location** | AWS drives it to your data center; parks it outside |
| **Power** | Requires external power connection |
| **Use cases** | Full data center migration (multi-PB to exabyte scale) |
| **Availability** | Limited regions; requires AWS engagement team |

**When Snowmobile vs. Snowball:**
- **Snowmobile:** >10 PB (rule of thumb); single massive dataset
- **Snowball:** <10 PB; or when you need devices in parallel at multiple locations
- **Break-even:** ~10 Snowball Edge devices ≈ 800 TB; beyond this, Snowmobile may be simpler

### Snow Family Data Transfer Process

```
1. Create Job    → AWS Console/CLI → select device type, S3 bucket, KMS key
2. Receive Device → AWS ships pre-provisioned, encrypted device
3. Connect       → Connect device to local network (10/25/100 GbE)
4. Transfer Data → Copy data to device (S3 API, NFS, or OpsHub)
5. Ship Back     → Use prepaid shipping label; send to AWS
6. Import to S3  → AWS imports data into specified S3 bucket
7. Wipe Device   → AWS securely erases device (NIST 800-88)
```

**Export Process (S3 → Snow → On-premises):**
- Create export job specifying S3 bucket/prefix
- AWS loads data onto device from S3
- Device shipped to you
- Connect and copy data off device to local storage
- Ship device back (AWS wipes it)

### AWS DataSync

**Purpose:** Fully managed data transfer service for automated, accelerated file/object migration between storage systems.

**Architecture:**
```
Source Storage ──► DataSync Agent (on-prem) ──► AWS DataSync Service ──► Target Storage
     │                    │                            │                       │
  NFS/SMB/HDFS     Installed on VM              Managed transfer         S3/EFS/FSx
  Self-managed     (VMware, Hyper-V, KVM)       Parallel streams         AWS storage
  Object storage                                Compression
                                                Encryption
                                                Validation
```

**Supported Sources:**

| Source Type | Protocol/Method |
|-------------|----------------|
| On-premises NFS | NFS v3, v4, v4.1 |
| On-premises SMB | SMB 2.1, 3.0, 3.1.1 |
| On-premises HDFS | Hadoop cluster |
| On-premises object storage | S3-compatible (MinIO, etc.) |
| AWS S3 | Cross-account, cross-region |
| AWS EFS | Cross-account, cross-region |
| AWS FSx (all types) | Cross-account, cross-region |
| AWS Snowcone | Built-in DataSync agent |
| Azure Blob Storage | As source (agentless) |
| Azure Files (SMB) | As source |
| Google Cloud Storage | As source (agentless) |

**Supported Targets:**

| Target Type | Examples |
|-------------|---------|
| Amazon S3 | Any storage class (Standard, IA, Glacier, etc.) |
| Amazon EFS | Regional file system |
| Amazon FSx for Windows | SMB-compatible |
| Amazon FSx for Lustre | HPC workloads |
| Amazon FSx for OpenZFS | Linux workloads |
| Amazon FSx for NetApp ONTAP | Multiprotocol |

**Key Features:**

| Feature | Description |
|---------|-------------|
| **Parallel transfer** | Up to 10 Gbps throughput per task (uses multiple parallel streams) |
| **Incremental transfer** | Only transfers changed files (compares metadata/checksums) |
| **Data integrity** | Automatic verification (checksums during and after transfer) |
| **Bandwidth throttling** | Configurable limit to avoid saturating network |
| **Scheduling** | Built-in scheduler (hourly, daily, weekly, custom cron) |
| **Filtering** | Include/exclude patterns for files/folders |
| **Compression** | In-transit compression for faster transfer |
| **Encryption** | TLS in transit; target encryption (KMS, SSE-S3) |
| **Task execution** | One-time or recurring transfers |
| **Preservation** | Preserves file metadata (permissions, timestamps, ownership, ACLs) |

**DataSync Agent:**
- Deployed on-premises as VMware ESXi, KVM, or Hyper-V virtual machine
- Also available as Amazon EC2 instance (for cloud-to-cloud transfers)
- Reads from source storage via NFS/SMB/HDFS/S3
- Sends data to DataSync service endpoints in AWS
- Agent handles compression, encryption, integrity validation
- One agent can handle multiple tasks
- **Not needed:** For AWS-to-AWS transfers (S3→S3, EFS→EFS) — no agent, service handles directly
- **Not needed:** For Azure Blob or Google Cloud Storage sources — agentless

**DataSync vs. Simple Network Copy:**

| Aspect | DataSync | rsync / robocopy |
|--------|----------|------------------|
| Speed | Up to 10 Gbps (parallel streams) | Limited by single connection |
| Integrity | Automatic checksums | Manual verification |
| Scheduling | Built-in | Must build with cron/task scheduler |
| Incremental | Yes (metadata comparison) | Yes (but slower) |
| Management | Fully managed | Self-managed |
| Encryption | TLS + KMS | Must configure separately |
| Monitoring | CloudWatch metrics | Custom solution |
| Metadata | Preserves all (POSIX, NTFS ACLs) | Varies by tool |

### AWS Transfer Family

**Purpose:** Fully managed SFTP, FTPS, FTP, and AS2 service that provides file transfer directly into/out of S3 or EFS.

**Supported Protocols:**

| Protocol | Port | Use Case |
|----------|------|----------|
| **SFTP** (SSH File Transfer Protocol) | 22 | Secure file exchange; most common |
| **FTPS** (FTP over SSL/TLS) | 21 (control) + dynamic (data) | Legacy systems requiring FTP with encryption |
| **FTP** (unencrypted) | 21 (control) + dynamic (data) | Internal-only; non-sensitive data (VPC-only access) |
| **AS2** (Applicability Statement 2) | 443 (HTTPS) | B2B EDI; supply chain document exchange |

**Architecture:**
```
External Partners / Users ──► AWS Transfer Family Endpoint ──► S3 Bucket or EFS
         │                           │                              │
    SFTP/FTPS/FTP/AS2          Managed server                 Landing zone
    Standard clients           (no infra to manage)           for file processing
         │                           │
    Existing workflows         Identity provider
    No code changes            (Service-managed, AD, custom Lambda)
```

**Key Features:**

| Feature | Description |
|---------|-------------|
| **Endpoint types** | Public (internet-facing), VPC (private, via DX/VPN) |
| **Identity providers** | Service-managed (SSH keys), AWS Directory Service (AD), Custom (Lambda/API Gateway) |
| **Storage backend** | S3 or EFS (transparent to users) |
| **Logical directories** | Map virtual folder structure to different S3 paths/buckets |
| **Managed workflows** | Process files on arrival (copy, tag, decrypt, custom Lambda steps) |
| **IP allowlisting** | Elastic IPs for deterministic client configuration |
| **Monitoring** | CloudWatch metrics + structured JSON logging |
| **Scaling** | Auto-scales; no capacity planning |
| **High availability** | Multi-AZ by default |

**Identity Provider Options:**

| Provider | How It Works |
|----------|-------------|
| **Service-managed** | Store SSH public keys in Transfer Family; username/key pairs |
| **AWS Directory Service** | Authenticate against Managed AD or AD Connector (username/password) |
| **Custom (Lambda)** | Lambda function authenticates against any IdP (LDAP, Okta, DB, etc.) |
| **Custom (API Gateway)** | API Gateway + Lambda for more complex auth flows |

**Managed Workflows:**
- Triggered when file upload completes
- Steps: Copy, Tag, Decrypt (PGP), Delete, Custom (Lambda)
- Use cases: virus scan uploaded files, move to processing bucket, notify downstream, validate format
- Exception handling: on error → move to quarantine bucket

**Endpoint Types:**

| Type | Access | Static IP | Use Case |
|------|--------|-----------|----------|
| **Public** | Internet | Yes (via EIP) | External partners, global access |
| **VPC (Internet-facing)** | Internet via VPC | Yes (EIP on ENI in VPC) | External partners + SG/NACL control |
| **VPC (Internal)** | Private only (DX/VPN) | Yes (private IP) | Internal transfers, on-premises integration |

### Key Limits & Quotas

**Snow Family:**

| Resource | Limit |
|----------|-------|
| Snowcone jobs per account | 1 concurrent (soft limit) |
| Snowball Edge jobs per account | 50 concurrent (soft limit) |
| Snowball Edge cluster size | 5-15 devices |
| Snowcone storage | 8 TB (HDD) / 14 TB (SSD) |
| Snowball Edge Storage Optimized | 80 TB usable |
| Snowball Edge Compute Optimized | 28 TB NVMe usable |
| Snowmobile | 100 PB |
| Device retention | 10 days free; $30/day after (Snowball Edge) |

**DataSync:**

| Resource | Limit |
|----------|-------|
| Tasks per account | 50 (soft limit) |
| Files per task | 25 million (soft limit) |
| Throughput per task | Up to 10 Gbps |
| Agents per account | 50 (soft limit) |
| Locations per account | 100 (soft limit) |
| Minimum file size | No minimum (handles small files) |
| Maximum file size | 50 TB per object (S3 limit) |

**Transfer Family:**

| Resource | Limit |
|----------|-------|
| Servers per account | 50 |
| Users per server | 10,000 |
| Maximum file size | 5 TB (S3 multipart limit) |
| Concurrent sessions | Scales automatically |
| Workflows per server | 10 |
| Workflow steps | 8 per workflow |

### Bandwidth & Time Calculations (Critical for Exam)

**Formula:** Transfer Time = Data Size / Available Bandwidth

| Data Size | 1 Gbps | 10 Gbps | 100 Gbps | Snow Device |
|-----------|--------|---------|----------|-------------|
| 1 TB | ~2.5 hours | ~15 min | ~1.5 min | Not worth it |
| 10 TB | ~1 day | ~2.5 hours | ~15 min | Borderline |
| 100 TB | ~12 days | ~1.2 days | ~3 hours | Snowball Edge |
| 500 TB | ~58 days | ~6 days | ~14 hours | Multiple Snowballs |
| 1 PB | ~116 days | ~12 days | ~28 hours | Multiple Snowballs |
| 10 PB | ~3 years | ~4 months | ~12 days | Snowmobile |

**Rule of Thumb:** If network transfer takes >1 week, consider Snow Family.

**Exam shortcut:** 1 Gbps ≈ 10 TB per day (accounting for overhead).

---

## 2. Design Patterns & Best Practices

### Data Transfer Decision Pattern

```
How much data? + How much bandwidth? + How much time?
│
├── <10 TB, adequate bandwidth → DataSync (online)
│
├── 10-80 TB, limited bandwidth, tight deadline → Snowball Edge (single device)
│
├── 80 TB - 5 PB, limited bandwidth → Multiple Snowball Edge devices (parallel)
│
├── >10 PB → Snowmobile
│
├── Ongoing incremental sync → DataSync (scheduled)
│
├── Partners uploading files (SFTP/FTPS) → Transfer Family
│
└── Edge computing + data collection → Snowcone or Snowball Edge Compute
```

### Large-Scale Migration Pattern (Snowball + DataSync)

```
Phase 1: Bulk transfer (offline)
Snowball Edge ←── 50 TB of historical data ←── On-premises NAS

Phase 2: Incremental sync (online)  
DataSync ←── Changes since Snowball was loaded ←── On-premises NAS

Result: All data in S3/EFS with minimal transfer time
```

**Why this pattern?** Snowball handles the bulk (avoids weeks of network transfer), DataSync handles the delta (catches up changes made while Snowball was in transit).

### Hybrid Storage Pattern (DataSync)

```
On-premises NFS ──► DataSync (scheduled daily) ──► S3 (archive/backup)
                                                    │
                                            Lifecycle policy
                                                    │
                                            S3-IA → Glacier
```

### File Gateway Replacement Pattern

```
Before: Users ──► Storage Gateway File Gateway ──► S3
After:  Users ──► Transfer Family (SFTP/SMB) ──► S3

When: Users already have SFTP/FTP clients; don't need local cache
```

### B2B File Exchange Pattern (Transfer Family)

```
Partner A (SFTP) ──►
Partner B (FTPS) ──► Transfer Family ──► S3 ──► Lambda (process) ──► DynamoDB
Partner C (AS2)  ──►                      │
                                    Managed Workflow
                                    (validate, tag, move)
```

### Edge Computing Pattern (Snow)

```
Remote Location (oil rig, military, factory floor):

Snowball Edge Compute Optimized
├── EC2 instances (local compute)
├── Lambda functions (local processing)  
├── S3 endpoint (local storage)
├── IoT Greengrass (device management)
└── ML inference (GPU model)

When connectivity available: ship device back OR use DataSync
```

### Anti-Patterns

| Anti-Pattern | Why It's Wrong | Better Approach |
|-------------|----------------|-----------------|
| Using network for 50TB with 100Mbps link | Takes ~46 days; deadline risk | Snowball Edge (days vs. weeks) |
| Snowball for 500GB transfer | Overkill; device takes days to ship | DataSync over network (hours) |
| DataSync without agent for on-prem NFS | Service can't reach on-prem directly | Deploy DataSync agent on-premises |
| Transfer Family for internal batch jobs | Designed for external user file exchange | Use DataSync or direct S3 SDK |
| Snowmobile for 5PB | Overkill; high minimum engagement | Multiple Snowball Edge devices |
| Using S3 CLI for ongoing sync | No incremental detection, no validation | DataSync (managed, incremental, validated) |
| Snowball for real-time data | Batch process; days of latency | DataSync or Direct Connect (online) |
| FTP (unencrypted) over internet | Security vulnerability | SFTP or FTPS; or FTP only in VPC |

### Well-Architected Alignment

- **Reliability:** DataSync auto-retry and validation; Transfer Family multi-AZ; Snow devices are encrypted and tamper-evident
- **Security:** KMS encryption on all Snow devices; TLS for DataSync; SFTP/FTPS for Transfer Family; IAM policies
- **Cost:** Right-size transfer method (don't over-provision); DataSync only charges for data moved; Transfer Family per-protocol-hour + per-GB
- **Performance:** DataSync 10 Gbps parallel; Snow 100 GbE connectivity; Transfer Family auto-scales
- **Operational Excellence:** DataSync scheduling + CloudWatch; OpsHub for Snow management; Transfer Family logging

---

## 3. Security & Compliance

### Snow Family Security

| Layer | Mechanism |
|-------|-----------|
| **Physical security** | Tamper-resistant, tamper-evident enclosure; E-ink shipping label |
| **Encryption** | 256-bit AES encryption; KMS customer-managed keys |
| **Key management** | Encryption keys NEVER stored on device; downloaded from KMS at unlock time |
| **Access control** | Device unlock requires credentials + KMS key from console/CLI |
| **Chain of custody** | Tracked by AWS; E-ink label with no visible destination |
| **Data erasure** | NIST 800-88 media sanitization after import; device wiped |
| **TPM** | Trusted Platform Module ensures device integrity |
| **Network isolation** | Devices don't connect to internet during data loading |
| **Snowmobile extras** | GPS tracking, 24/7 video surveillance, armed escort (optional), dedicated fiber, alarm systems |

**Critical Security Facts:**
- If device is tampered with → AWS detects via TPM; data unreadable
- KMS key never leaves AWS → even if device is intercepted, data cannot be decrypted
- AWS cannot access your data on the device (KMS key required from customer)
- After import: data exists only in specified S3 bucket; device is wiped

### DataSync Security

| Aspect | Mechanism |
|--------|-----------|
| **In transit** | TLS 1.2+ encryption for all data |
| **At rest (target)** | Inherits target encryption (S3 SSE, EFS encryption, FSx encryption) |
| **Agent communication** | HTTPS (443) to DataSync service endpoints |
| **VPC endpoints** | PrivateLink support for DataSync (no internet required for control plane) |
| **Agent connectivity** | Agent → DataSync endpoints over internet, VPN, or DX |
| **IAM** | Task execution role defines what DataSync can write to |
| **Source access** | Agent uses NFS/SMB credentials configured at location creation |
| **Audit** | CloudTrail for API calls; CloudWatch for transfer metrics |

**DataSync Network Ports (Agent):**

| Direction | Port | Purpose |
|-----------|------|---------|
| Agent → AWS (TCP 443) | 443 | Control (HTTPS to DataSync service) |
| Agent → AWS (TCP 443) | 443 | Data transfer (TLS to DataSync endpoints) |
| Agent → Source (NFS) | 2049 | Read from NFS source |
| Agent → Source (SMB) | 445 | Read from SMB source |

### Transfer Family Security

| Aspect | Mechanism |
|--------|-----------|
| **SFTP** | SSH key or password authentication; encrypted in transit |
| **FTPS** | TLS/SSL certificate; encrypted in transit |
| **FTP** | VPC endpoint ONLY (no internet-facing FTP allowed); unencrypted |
| **AS2** | Message-level encryption + signing (S/MIME certificates) |
| **At rest** | S3 SSE (SSE-S3, SSE-KMS) or EFS encryption |
| **Network** | Security groups, NACLs (VPC endpoint); EIP allowlisting (public) |
| **Identity** | Service-managed SSH keys, AD integration, custom Lambda IdP |
| **Audit** | CloudTrail + structured CloudWatch logs (JSON) |
| **FIPS** | FIPS 140-2 endpoint available for SFTP |

**Transfer Family IAM:**
- **Execution role:** Per-user role defining S3/EFS permissions
- **Scope-down policy:** Session policy that restricts user to their home directory
- **Logical directories:** Map virtual paths to S3 prefixes (users see simplified structure)

**Example scope-down policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::bucket",
      "Condition": {
        "StringLike": {
          "s3:prefix": ["${transfer:HomeBucket}/*", "${transfer:HomeBucket}"]
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::bucket/${transfer:HomeDirectory}*"
    }
  ]
}
```

### Compliance Considerations

| Requirement | Solution |
|-------------|----------|
| Data residency | Snow devices ship within country; DataSync targets in-region; Transfer Family regional |
| HIPAA | Snow + DataSync + Transfer Family all HIPAA eligible |
| PCI DSS | Supported on all three services |
| FedRAMP | Available in GovCloud regions |
| FIPS 140-2 | Transfer Family SFTP FIPS endpoint; DataSync TLS FIPS |
| Data sovereignty | Snow devices don't cross borders (order in correct region) |

---

## 4. Cost Optimization

### Snow Family Pricing

| Component | Cost |
|-----------|------|
| **Snowcone** | Service fee per job + shipping + storage days |
| **Snowball Edge (on-demand)** | Service fee per job (~$300 per 10-day use) |
| **Snowball Edge (committed)** | 1-3 year commitment (discounted daily rate for edge compute) |
| **Extra days** | ~$30/day per device beyond included 10-15 days |
| **Shipping** | Region-dependent standard shipping cost |
| **Data transfer IN** | FREE (loading data into S3 from device) |
| **Data transfer OUT** | Standard S3 GET + data transfer rates (export jobs) |
| **Snowmobile** | Custom pricing (AWS engagement; ~$0.005/GB typical) |

**Key insight:** Data transfer INTO AWS from Snow devices is FREE. You only pay for the device service fee and shipping.

### DataSync Pricing

| Component | Cost |
|-----------|------|
| **Data transferred** | Per-GB flat rate (~$0.0125/GB for most regions) |
| **No minimum** | Pay only for what you transfer |
| **No agent cost** | Agent software is free |
| **Incremental** | Only pays for CHANGED data on subsequent transfers |
| **Task scheduling** | No extra cost for schedules |
| **Cross-region** | Additional standard data transfer charges apply |

**DataSync cost advantage:** Only charges for data actually transferred. Incremental syncs transfer only changes = minimal ongoing cost after initial migration.

### Transfer Family Pricing

| Component | Cost |
|-----------|------|
| **Endpoint (protocol-hour)** | ~$0.30/hour per protocol enabled per server (~$216/month) |
| **Data transfer** | Per-GB uploaded/downloaded (~$0.04/GB) |
| **Managed workflows** | Per workflow step executed |
| **Multiple protocols** | Each enabled protocol adds per-hour charge |

**Cost trap:** Transfer Family endpoint charges are PER HOUR regardless of usage. A server with SFTP + FTPS enabled = 2x protocol-hour charges continuously. If usage is minimal, this can be expensive per-GB.

### Cost Comparison Matrix

| Scenario | Best Option | Approximate Cost |
|----------|-------------|-----------------|
| 50 TB one-time migration, 100 Mbps link | Snowball Edge | ~$300 (device) + shipping |
| 50 TB one-time migration, 10 Gbps link | DataSync | ~$625 (50,000 GB × $0.0125) |
| 1 TB daily sync | DataSync | ~$12.50/day (incremental = less) |
| External partners uploading via SFTP | Transfer Family | ~$216/month + per-GB |
| 100 PB data center migration | Snowmobile | ~$500,000 (custom pricing) |
| 500 GB occasional file exchange | Transfer Family (if partners need SFTP) OR direct S3 upload | Variable |

### Cost Traps

| Trap | Impact | Mitigation |
|------|--------|------------|
| Snow device kept >10 days | $30/day per device | Load data quickly; ship back immediately |
| Multiple protocols on Transfer Family | 2-4x endpoint charges | Enable only needed protocols |
| Transfer Family for rarely-used SFTP | $216/month even with zero files | Consider S3 presigned URLs or temporary servers |
| DataSync full copy every day | Paying for all data each time | Use incremental (default) — only transfers changes |
| Snowball for small datasets | High fixed cost per device for low data | Use DataSync if ≤10 TB and bandwidth allows |
| Cross-region DataSync | Data transfer charges PLUS DataSync per-GB | Keep source/target in same region if possible |
| Not cleaning up DataSync agents/tasks | No direct cost, but operational overhead | Remove unused agents and tasks |

### Cost Optimization Strategies

1. **Snow devices:** Load data as fast as possible (parallel, 100 GbE); ship back within free days
2. **DataSync:** Use incremental transfers; only move changed data; schedule during low-cost network hours
3. **Transfer Family:** Minimize enabled protocols; consider shutting down dev/test servers off-hours (custom automation)
4. **Right tool selection:** Calculate breakeven — DataSync network cost vs. Snow device cost based on data size and bandwidth
5. **S3 storage class:** DataSync can write directly to S3 Glacier or S3-IA (skip Standard if data is cold)
6. **Combine Snowball + DataSync:** Bulk via Snow, incremental via DataSync (minimizes DataSync per-GB cost on large initial dataset)

---

## 5. High Availability, Disaster Recovery & Resilience

### Snow Family Resilience

| Aspect | Details |
|--------|---------|
| Device failure | Order replacement device; data on failed device is encrypted and unreadable |
| Lost in transit | Data encrypted with KMS; AWS tracks shipping; re-order job |
| Clustering (Snowball Edge) | 5-15 devices; survives individual device failures; data replicated across cluster |
| Edge compute (disconnected) | Devices operate fully offline; no AWS connectivity needed |
| Data durability | ALWAYS keep source copy until confirmed import to S3 (verify S3 objects) |

**Critical:** Snow devices are NOT a backup mechanism. Always maintain source data until confirmed in S3.

### DataSync Resilience

| Feature | Behavior |
|---------|----------|
| **Auto-retry** | Retries failed transfers automatically |
| **Integrity verification** | Checksums verify every byte at source and target |
| **Incremental recovery** | If interrupted, next run only transfers remaining/changed files |
| **Agent failure** | Deploy redundant agents; tasks can be reassigned |
| **Network interruption** | Task pauses; resumes automatically when connectivity restored |
| **Scheduling** | Missed schedule → runs at next opportunity |
| **CloudWatch alarms** | Alert on transfer failures or unexpected metrics |

**DataSync for DR:**
```
Primary (us-east-1)                    DR (us-west-2)
EFS ──► DataSync (scheduled hourly) ──► EFS (replica)
S3  ──► DataSync (scheduled daily)  ──► S3 (backup)
FSx ──► DataSync (scheduled)        ──► FSx (standby)
```

### Transfer Family Resilience

| Feature | Details |
|---------|---------|
| **Multi-AZ** | Built-in; endpoints span multiple AZs automatically |
| **High availability** | Managed by AWS; no single point of failure |
| **Scaling** | Auto-scales concurrent connections |
| **Endpoint failover** | If AZ fails, connections route to healthy AZ |
| **Storage durability** | S3 (11 nines) or EFS (multi-AZ) as backend |
| **No data loss** | Files written to S3/EFS are immediately durable |

### DR Pattern: Cross-Region File Replication

```
Region A (Primary):
Transfer Family endpoint → S3 bucket-A
                              │
                         DataSync (cross-region, scheduled)
                              │
                              ▼
Region B (DR):
S3 bucket-B → Transfer Family endpoint (standby)

Failover: Update DNS to point partners at Region B Transfer Family endpoint
```

---

## 6. Exam-Focused Section

### Straightforward Questions (7 examples)

**Q1:** A company needs to migrate 60 TB of data from on-premises NAS to S3. They have a 1 Gbps internet connection. The project must complete within 1 week. Can they use online transfer?

**A:** No. 60 TB at 1 Gbps ≈ 7 days at 100% utilization (theoretical). With overhead, it would take 8-10 days. Use Snowball Edge (80 TB capacity; load in 2-3 days via 10/25 GbE; ship + import ≈ 5-7 days total).

**Q2:** A company needs to sync files from an on-premises NFS share to Amazon EFS daily, transferring only changed files. What service should they use?

**A:** AWS DataSync. It supports NFS as source, EFS as target, scheduled transfers, and incremental (only changed files) mode.

**Q3:** A company has external partners who upload files via SFTP. They want to move the SFTP server to AWS with files stored in S3 and no infrastructure management. What should they use?

**A:** AWS Transfer Family (SFTP protocol). Partners continue using existing SFTP clients; files land in S3; fully managed (no servers to patch).

**Q4:** What is the maximum storage capacity of a single Snowball Edge Storage Optimized device?

**A:** 80 TB usable storage.

**Q5:** A remote oil rig needs to process sensor data locally with no internet connectivity, then ship the processed data to AWS quarterly. What AWS service should they use?

**A:** Snowball Edge Compute Optimized (provides local EC2 compute for processing; stores results; ships to AWS quarterly for S3 import).

**Q6:** A company needs to transfer 500 TB to AWS. They have adequate time but only 100 Mbps bandwidth. What's the most practical approach?

**A:** Multiple Snowball Edge Storage Optimized devices (500 TB / 80 TB = 7 devices). Order in parallel, load simultaneously if possible.

**Q7:** What is the difference between AWS DataSync and AWS Transfer Family?

**A:** DataSync is for automated data migration/sync between storage systems (bulk transfer, NFS/SMB/HDFS → S3/EFS/FSx). Transfer Family is for user-facing file transfer protocols (SFTP/FTPS/FTP/AS2) enabling external partners to exchange files with S3/EFS.

### Tricky / Scenario-Based Questions (7 examples)

**Q1:** A company has 200 TB on-premises and a 10 Gbps Direct Connect link to AWS. They need to complete the migration within 2 weeks. The data changes at ~500 GB/day during the migration. Should they use Snowball or DataSync?

**A:** DataSync over 10 Gbps DX. Calculation: 200 TB at 10 Gbps ≈ 2-3 days for initial transfer. Then incremental syncs catch up daily changes (500 GB/day at 10 Gbps = ~7 minutes/day). Total well within 2 weeks. Snowball would take longer (shipping time + import time ≈ 1 week just for logistics).
- **Wrong:** "Snowball because it's over 10 TB" — the threshold depends on bandwidth, not just data size
- **Wrong:** "DataSync can't handle 200 TB" — it can; 10 Gbps throughput per task is sufficient
- **Keywords:** "10 Gbps Direct Connect" (fast link = online transfer wins), "2 weeks" (plenty of time for 200 TB at 10 Gbps)

**Q2:** A company orders a Snowball Edge, loads 50 TB of data, and ships it to AWS. While the device is in transit (7 days), an additional 2 TB of data is generated on-premises. How should they handle the delta?

**A:** Use DataSync to transfer the 2 TB delta over the network after the Snowball data is imported to S3. This is the standard "Snowball for bulk + DataSync for delta" pattern. DataSync handles incremental sync by comparing source and target, transferring only files not yet in S3.
- **Wrong:** "Order another Snowball for 2 TB" — overkill; 2 TB transfers quickly over network
- **Wrong:** "Wait and re-copy everything" — Snowball already handled 50 TB; just sync the delta
- **Keywords:** "in transit" + "additional data generated" (classic Snowball + DataSync combo pattern)

**Q3:** A financial services company has 100 external partners uploading reconciliation files daily via SFTP. Currently using an on-premises SFTP server. They want to migrate to AWS with these requirements: (1) Partners must NOT change their SFTP clients or workflows. (2) Files must be encrypted at rest with customer-managed KMS keys. (3) Each partner must only access their own directory. What solution meets all requirements?

**A:** AWS Transfer Family with SFTP protocol. (1) Partners use same SFTP clients — no change needed. (2) S3 backend with SSE-KMS (customer-managed key). (3) Use scope-down IAM policy per user restricting to home directory (${transfer:HomeDirectory} variable). Use logical directories to map each partner to their S3 prefix.
- **Wrong:** "DataSync" — DataSync is for system-to-system transfer, not user-facing SFTP
- **Wrong:** "EC2 with OpenSSH" — not fully managed; doesn't meet "no infrastructure" implicit requirement
- **Keywords:** "SFTP" + "partners must NOT change" (Transfer Family), "each partner only own directory" (scope-down policy)

**Q4:** A media company has 2 PB of video archives on a SAN. They have 1 Gbps internet and need everything in S3 Glacier within 3 months. What's the optimal approach considering cost and timeline?

**A:** Use multiple Snowball Edge Storage Optimized devices (2 PB / 80 TB = 25 devices). Order in batches (can have multiple simultaneously). Load via 10/25/100 GbE from SAN. Ship to AWS → import directly to S3 → apply lifecycle policy to transition to Glacier immediately. Timeline: feasible in 4-6 weeks with parallel device loading.
- **Wrong:** "Snowmobile" — 2 PB is below Snowmobile's sweet spot (~10+ PB); 25 Snowballs is more practical
- **Wrong:** "DataSync over 1 Gbps" — 2 PB at 1 Gbps ≈ 185 days (exceeds 3-month deadline)
- **Wrong:** "Write directly to Glacier" — Snow imports to S3 Standard first; use lifecycle rule to transition
- **Keywords:** "2 PB" + "1 Gbps" + "3 months" (network too slow; need Snow), "S3 Glacier" (lifecycle policy after S3 import)

**Q5:** A company uses DataSync to replicate an on-premises NFS share to EFS daily. One day, the sync task fails midway through transferring 50,000 files (30,000 completed, 20,000 remaining). What happens when the task runs again the next day?

**A:** DataSync performs incremental transfer. On the next run, it compares source and target metadata/checksums, identifies the 20,000 files not yet transferred (plus any new changes), and transfers only those. The 30,000 already-transferred files are NOT re-sent. DataSync's built-in verification ensures already-transferred files match.
- **Wrong:** "Full re-transfer of all 50,000 files" — DataSync is incremental by default
- **Wrong:** "Task permanently fails; needs manual restart" — scheduled tasks run at next scheduled time automatically
- **Keywords:** "fails midway" + "next day" (incremental recovery; only remaining + changed files)

**Q6:** A pharmaceutical company must ship clinical trial data between their facility and AWS. Regulations require that the data never traverses the public internet during transfer, and a complete chain of custody must be maintained. They have no Direct Connect. What's the most compliant approach?

**A:** AWS Snowball Edge. Data is encrypted with KMS on-site (never traverses internet). AWS provides chain of custody tracking (shipping tracking, E-ink label, TPM verification). Device is physically transported (no network = no internet exposure). After import, AWS wipes device per NIST 800-88.
- **Wrong:** "DataSync over VPN" — VPN traverses the public internet (encrypted, but technically on internet infrastructure)
- **Wrong:** "Transfer Family SFTP" — SFTP goes over internet
- **Keywords:** "never traverses public internet" (physical transport only), "chain of custody" (Snow devices tracked), "no Direct Connect" (eliminates private network option)

**Q7:** A company has deployed a Snowball Edge Storage Optimized at a remote factory for 6 months as a local S3 endpoint for IoT data collection. The device is nearing 80% capacity. They need to continue collecting data without interruption. What are their options?

**A:** Options: (1) Order a replacement Snowball Edge; swap devices (return full one, start collecting on new one). (2) If some connectivity available: use DataSync agent on-premises to offload some data to S3 over network, freeing device space. (3) Create a Snowball Edge cluster (up to 15 devices) for expanded capacity. (4) If committed long-term: consider AWS Outposts or dedicated infrastructure. For the exam: swap device is the simplest operational answer.
- **Wrong:** "Expand the device storage" — Snowball Edge storage is fixed at 80 TB; can't expand a single device
- **Wrong:** "Delete old data" — may still be needed and hasn't been shipped to S3 yet
- **Keywords:** "6 months" (long-term edge deployment), "nearing 80% capacity" (needs more space or offload), "without interruption" (swap or cluster)

### Common Exam Traps & Pitfalls

| Trap | Reality |
|------|---------|
| "Snowball can import directly to Glacier" | NO — imports to S3 Standard only; use lifecycle policy to transition |
| "DataSync requires Direct Connect" | NO — works over internet, VPN, or DX (agent connects via HTTPS) |
| "Transfer Family = file migration tool" | NO — it's for user-facing file exchange (SFTP/FTPS/FTP); DataSync is for migration |
| "Snowcone is just smaller Snowball" | Also has built-in DataSync agent (unique); suitable for tactical/portable use |
| "Snow devices store encryption keys" | NO — keys NEVER stored on device; downloaded from KMS at unlock time |
| "DataSync always needs an agent" | NO — AWS-to-AWS transfers (S3→S3, EFS→EFS) are agentless |
| "Snowmobile for 5 PB" | Usually overkill — multiple Snowball Edges more practical at <10 PB |
| "Transfer Family FTP is internet-facing" | NO — FTP (unencrypted) is VPC-only; SFTP/FTPS can be internet-facing |
| "DataSync replaces Storage Gateway" | Different use cases — DataSync is one-time/scheduled sync; Storage Gateway is continuous cache/gateway |
| "Snow = only for import" | NO — Snow supports EXPORT (S3 → device → on-premises) too |
| "DataSync is slow" | Can achieve 10 Gbps per task; often faster than rsync/robocopy |
| "Snowball Edge is just for storage" | Also provides EC2 compute, Lambda, IoT Greengrass, GPU (Compute Optimized) |

---

## 7. Cheat Sheet

### Must-Know Facts

**Snow Family:**
- **Snowcone:** 8 TB HDD / 14 TB SSD; smallest; built-in DataSync agent; portable
- **Snowball Edge Storage Optimized:** 80 TB; S3 endpoint + NFS; EC2 compute; cluster up to 15
- **Snowball Edge Compute Optimized:** 28 TB NVMe; 104 vCPUs; optional GPU; heavy edge compute
- **Snowmobile:** 100 PB; semi-trailer; for >10 PB migrations
- **Encryption:** 256-bit KMS; keys NEVER on device; tamper-resistant
- **Import target:** S3 Standard only (lifecycle to Glacier after)
- **Retention:** ~10 days free per device; $30/day after
- **Data transfer IN:** FREE (Snow → S3)
- **Cluster:** 5-15 Snowball Edge devices (up to 1.2 PB)

**DataSync:**
- **Speed:** Up to 10 Gbps per task (parallel streams)
- **Incremental:** Only transfers changed files (default)
- **Sources:** NFS, SMB, HDFS, S3-compatible, S3, EFS, FSx, Azure Blob, GCS
- **Targets:** S3 (any class), EFS, FSx (all variants)
- **Agent:** Required for on-premises sources (VMware/Hyper-V/KVM VM)
- **No agent:** AWS-to-AWS transfers, Azure Blob, GCS
- **Scheduling:** Built-in (cron or fixed intervals)
- **Validation:** Automatic checksums (source and target)
- **Pricing:** Per-GB transferred only
- **Preserves:** File metadata (permissions, timestamps, ACLs)

**Transfer Family:**
- **Protocols:** SFTP (22), FTPS (21), FTP (21, VPC-only), AS2 (443)
- **Backend:** S3 or EFS (transparent to users)
- **Identity:** Service-managed SSH keys, AD, custom Lambda
- **Endpoint types:** Public, VPC (internet-facing), VPC (internal)
- **Managed workflows:** Process files on arrival (copy, tag, decrypt, Lambda)
- **Multi-AZ:** Built-in by default
- **Cost:** Per-protocol-hour (~$0.30/hr) + per-GB
- **Use case:** Partner file exchange; NOT for bulk data migration

### Key Differentiators

| DataSync vs. Transfer Family | DataSync = system-to-system migration/sync; Transfer Family = user/partner file exchange |
|---|---|
| **DataSync vs. Storage Gateway** | DataSync = transfer/migrate data; Storage Gateway = hybrid storage (cache + gateway) |
| **DataSync vs. S3 Replication** | DataSync = any source to S3; S3 Replication = S3 to S3 only (same/cross-region) |
| **Snow vs. DataSync** | Snow = offline physical transfer; DataSync = online network transfer |
| **Snowcone vs. Snowball Edge** | Snowcone = portable, small (8-14 TB); Snowball Edge = large (80 TB), more compute |
| **Snowball vs. Snowmobile** | Snowball = per-device 80 TB (multiple for PB scale); Snowmobile = single 100 PB container |
| **Transfer Family vs. EC2+SFTP** | Transfer Family = managed, auto-scale; EC2 = self-managed, patch yourself |
| **DataSync vs. rsync** | DataSync = managed, parallel, validated, monitored; rsync = manual, single-stream |

### Exam Keyword Triggers

| If the question says... | Think... |
|------------------------|----------|
| "Offline transfer" / "limited bandwidth" / "no network" | Snow Family |
| "50TB+" + "100 Mbps or less" + "tight deadline" | Snowball Edge |
| "Edge computing" / "disconnected" / "remote location" | Snowball Edge Compute Optimized or Snowcone |
| ">10 PB" / "data center migration" / "exabyte" | Snowmobile |
| "Portable" / "tactical" / "small remote site" | Snowcone |
| "Ongoing sync" / "incremental" / "NFS/SMB to S3/EFS" | DataSync |
| "Migrate file shares" / "NAS migration" | DataSync |
| "Cross-region replication" (files/EFS/FSx) | DataSync |
| "SFTP" / "FTPS" / "partners upload" / "file exchange" | Transfer Family |
| "B2B" / "EDI" / "AS2" / "supply chain" | Transfer Family (AS2 protocol) |
| "No client changes" + "SFTP migration" | Transfer Family |
| "Bulk + delta" / "initial load + incremental" | Snowball + DataSync combination |
| "Built-in DataSync" / "smallest device" | Snowcone |
| "GPU at edge" / "ML inference at edge" | Snowball Edge Compute Optimized |
| "Cluster for edge storage" | Snowball Edge cluster (5-15 devices) |

### Decision Flowchart

```
What do you need to do?
│
├── MIGRATE bulk data (one-time or infrequent)
│   ├── Can network handle it in time?
│   │   ├── YES (adequate bandwidth + timeline) → DataSync
│   │   └── NO (too much data / too slow / no connectivity)
│   │       ├── <80 TB → Snowball Edge (single device)
│   │       ├── 80 TB - 10 PB → Multiple Snowball Edges
│   │       └── >10 PB → Snowmobile
│   └── Need bulk + catch-up?
│       └── Snowball (bulk) + DataSync (delta)
│
├── ONGOING sync (scheduled)
│   ├── File systems (NFS/SMB) → DataSync (scheduled)
│   ├── S3 to S3 (same region) → S3 Replication (if same bucket prefix logic)
│   └── EFS/FSx cross-region → DataSync
│
├── USER/PARTNER file exchange
│   ├── Partners use SFTP → Transfer Family (SFTP)
│   ├── Legacy systems need FTPS → Transfer Family (FTPS)
│   ├── Internal only, unencrypted → Transfer Family (FTP, VPC-only)
│   └── B2B EDI / supply chain → Transfer Family (AS2)
│
├── EDGE COMPUTING (disconnected)
│   ├── Light compute, small data → Snowcone
│   ├── Heavy compute / GPU / ML → Snowball Edge Compute Optimized
│   └── Large edge storage (S3 interface) → Snowball Edge Storage Optimized (or cluster)
│
└── HYBRID STORAGE (continuous access to cloud)
    └── Storage Gateway (NOT Snow/DataSync/Transfer Family)
```

### Bandwidth Quick-Reference (Exam Math)

| Bandwidth | Per Day | Per Week | Per Month |
|-----------|---------|----------|-----------|
| 100 Mbps | ~1 TB | ~7 TB | ~30 TB |
| 1 Gbps | ~10 TB | ~70 TB | ~300 TB |
| 10 Gbps | ~100 TB | ~700 TB | ~3 PB |
| 100 Gbps | ~1 PB | ~7 PB | ~30 PB |

**Shortcut:** 1 Gbps ≈ 10 TB/day. Scale linearly.

---

## 8. Additional Deep-Dive

### 10 More Tricky Scenario-Based Questions

**Q1:** A company needs to migrate 500 TB from an on-premises Hadoop cluster (HDFS) to Amazon S3. They have a 10 Gbps Direct Connect and 6 weeks. The data is actively being processed during migration (new data arriving at 2 TB/day). What's the best approach?

**A:** Use DataSync with HDFS as source and S3 as target. DataSync supports HDFS natively. At 10 Gbps, initial 500 TB takes ~5-6 days. Then schedule daily incremental syncs to capture the 2 TB/day of new data. DataSync's incremental transfer handles ongoing changes efficiently. No need for Snowball since bandwidth and timeline are sufficient.
- **Wrong:** "Snowball for initial + DataSync for incremental" — unnecessary complexity; 10 Gbps handles 500 TB in under a week
- **Wrong:** "Use DMS" — DMS is for databases, not HDFS/file data
- **Keywords:** "HDFS" + "S3" (DataSync supports HDFS source), "10 Gbps" + "6 weeks" (bandwidth sufficient for 500 TB), "actively being processed" (incremental sync needed)

**Q2:** A global company has offices in 15 countries. Each office uses local SFTP servers for partner file exchange. They want to consolidate to a single AWS-based solution while ensuring partners in each country connect to a local endpoint for low latency. What architecture should the solutions architect design?

**A:** Deploy AWS Transfer Family with SFTP protocol in multiple AWS regions (one per country/area). Each regional Transfer Family endpoint uses S3 as backend. Use S3 Cross-Region Replication to consolidate data to a central bucket if needed. Partners connect to their nearest regional endpoint (low latency). Managed workflows in each region process files locally.
- **Wrong:** "Single global Transfer Family endpoint" — high latency for distant partners
- **Wrong:** "Global Accelerator in front of Transfer Family" — Transfer Family doesn't integrate with Global Accelerator for SFTP
- **Keywords:** "15 countries" + "local endpoint" + "low latency" (multi-region deployment), "consolidate" (S3 replication for central view)

**Q3:** A hospital needs to transfer 20 TB of medical imaging data (DICOM files) to S3 weekly from their on-premises PACS system. The PACS exports via NFS. They have 1 Gbps internet. Compliance requires data never traverses public internet unencrypted, and files must be verified for integrity. What should they use?

**A:** AWS DataSync with on-premises agent. Agent reads from NFS (PACS export). DataSync encrypts with TLS in transit (satisfies "never unencrypted on internet"). Built-in integrity verification via checksums (satisfies integrity requirement). Schedule weekly task. 20 TB at 1 Gbps ≈ 2 days — fits within weekly schedule.
- **Wrong:** "Snowball weekly" — impractical for recurring weekly transfer (device logistics)
- **Wrong:** "Transfer Family SFTP" — PACS exports via NFS, not SFTP; DataSync is system-to-system
- **Wrong:** "VPN required" — DataSync TLS encryption satisfies "not unencrypted" (the requirement is encryption, not necessarily VPN)
- **Keywords:** "weekly" + "NFS" + "20 TB" + "1 Gbps" (recurring, feasible online → DataSync), "never unencrypted" (TLS built-in), "verified for integrity" (DataSync auto-verification)

**Q4:** A company ordered 5 Snowball Edge Storage Optimized devices for a 400 TB migration. After loading all 5 devices (400 TB total), they discover that 50 TB of critical data was missed (stored on a different NAS that wasn't included in the original copy). The devices have already been shipped back to AWS. What's the fastest way to get the remaining 50 TB to AWS?

**A:** Transfer the 50 TB over the network using DataSync (if bandwidth allows). Calculate: at 1 Gbps ≈ 5 days; at 10 Gbps ≈ 5 hours. If network transfer is feasible within their timeline, DataSync is fastest (avoids Snowball logistics of 1-2 weeks). If bandwidth is too limited, order another Snowball Edge.
- **Wrong:** "Wait for original devices to import, then re-order for 50 TB" — unnecessary delay
- **Wrong:** "Cancel the shipment" — devices already shipped
- **Keywords:** "already shipped" (can't add to existing devices), "50 TB" (evaluate if online transfer is feasible based on bandwidth), "fastest way" (DataSync if bandwidth allows; new Snowball if not)

**Q5:** A company is using Transfer Family (SFTP) for partner file exchange. A new partner requires AS2 protocol for B2B EDI document exchange with message-level encryption and non-repudiation. Can they support both partners on the same Transfer Family server?

**A:** Yes. Transfer Family supports multiple protocols on the same server. Enable AS2 protocol on the existing server alongside SFTP. AS2 provides message-level encryption (S/MIME) and non-repudiation (digital signatures) as required for B2B EDI. Each protocol has its own endpoint URL. Files from both protocols can land in the same S3 bucket (or different buckets per partner via logical directories).
- **Wrong:** "Need a separate service for AS2" — Transfer Family natively supports AS2
- **Wrong:** "AS2 and SFTP can't coexist" — multiple protocols per server is supported
- **Keywords:** "AS2" + "B2B EDI" + "non-repudiation" (Transfer Family AS2 protocol), "same server" (multi-protocol support)

**Q6:** A media production company generates 10 TB of raw video daily at a remote filming location with no internet connectivity. They need the footage available in S3 within 48 hours for post-production editing in the cloud. Currently, the production runs for 3 months. What solution provides continuous data flow?

**A:** Use multiple Snowcone SSD devices in rotation. Day 1: load 10 TB onto Snowcone SSD (14 TB capacity). Ship via overnight express. Day 2: footage arrives at AWS, imported to S3 within 48 hours. Maintain 3-4 Snowcones in rotation (one loading, one shipping, one at AWS, one returning). Alternatively: Snowball Edge if daily volume exceeds Snowcone capacity or if local processing (transcoding) is needed.
- **Wrong:** "DataSync" — no internet connectivity rules out online transfer
- **Wrong:** "Single Snowball Edge for 3 months" — 10 TB/day × 90 days = 900 TB; single device holds 80 TB max
- **Keywords:** "no internet" (must be offline/physical), "10 TB daily" + "48 hours" (need rapid turnaround; Snowcone rotation), "3 months" (sustained production; rotation logistics)

**Q7:** A company uses DataSync to replicate on-premises NFS data to S3. They need the data in S3 to be stored in S3 Intelligent-Tiering storage class with specific metadata tags applied. They also need to be notified when each sync completes. Can DataSync handle all these requirements?

**A:** Yes. (1) DataSync supports specifying the S3 storage class at the task level — set to S3 Intelligent-Tiering. (2) DataSync doesn't natively apply custom tags to individual objects, but you can use an S3 Event Notification (triggered on object creation) → Lambda function to tag objects after sync. OR use DataSync task-level settings for basic metadata. (3) DataSync integrates with CloudWatch Events / EventBridge — create a rule on "DataSync Task Execution State Change" → SNS notification when task completes.
- **Wrong:** "DataSync only supports S3 Standard" — supports all S3 storage classes as target
- **Wrong:** "Need a separate tool for notifications" — EventBridge integration built-in
- **Keywords:** "S3 Intelligent-Tiering" (storage class setting in DataSync), "tags" (requires post-transfer automation), "notified when sync completes" (EventBridge → SNS)

**Q8:** A government agency needs to migrate 80 TB of classified data from a SCIF (Sensitive Compartmented Information Facility). No network connectivity to the outside is allowed. Personnel cannot bring personal devices into the SCIF. The data must arrive at AWS GovCloud encrypted with their own KMS key. What's the only viable option?

**A:** Snowball Edge in a special secure configuration. The device is provisioned with the agency's KMS key, shipped to the SCIF, loaded with data (air-gapped, no network needed). Data is encrypted on-device with KMS. Device shipped back to AWS GovCloud for import. AWS follows specific handling procedures for classified workloads. Critical: device must be ordered in GovCloud region.
- **Wrong:** "DataSync" — no network connectivity allowed
- **Wrong:** "Snowcone" — 14 TB max is insufficient for 80 TB
- **Wrong:** "Standard Snowball in commercial region" — classified data requires GovCloud
- **Keywords:** "SCIF" + "no network" + "classified" (physical-only, air-gapped), "AWS GovCloud" + "their own KMS key" (GovCloud Snowball with CMK)

**Q9:** A company currently transfers 5 TB daily from their data center to S3 using a custom script that runs `aws s3 sync`. The transfers take 14 hours daily over a 1 Gbps connection, and frequently fail partway through requiring manual restart. They also notice some files are corrupted after transfer. What should they replace the script with?

**A:** AWS DataSync. Benefits over `aws s3 sync`: (1) Parallel streams achieve up to 10 Gbps (faster transfer). (2) Automatic retry on failure (no manual restart). (3) Built-in integrity verification (checksums detect corruption). (4) Incremental transfer efficiently detects changes. (5) Scheduling eliminates cron management. (6) CloudWatch monitoring and alerting. The 5 TB daily at 1 Gbps with DataSync's parallel optimization should complete in 1-2 hours instead of 14.
- **Wrong:** "Just fix the script" — doesn't address fundamental limitations (single-stream, no validation)
- **Wrong:** "Snowball daily" — impractical for daily recurring transfer
- **Keywords:** "aws s3 sync" + "frequently fail" + "corrupted" (DataSync solves all three: parallel, retry, integrity), "14 hours" → DataSync dramatically faster

**Q10:** A company has an on-premises NetApp storage array with 100 TB of data exported via both NFS and SMB. Windows users access via SMB; Linux users access via NFS. They want to migrate to AWS and maintain both access patterns. Target must support both NFS and SMB. What's the migration path?

**A:** Target: Amazon FSx for NetApp ONTAP (supports both NFS and SMB simultaneously — multiprotocol). Migration: Use DataSync with NFS or SMB source (choose one protocol for transfer) → FSx for NetApp ONTAP target. DataSync preserves file permissions and metadata. Post-migration: Windows users connect via SMB, Linux users via NFS (same data, same FSx volume).
- **Wrong:** "Two separate targets (EFS for NFS + FSx Windows for SMB)" — splits the data; doesn't maintain single namespace
- **Wrong:** "S3 as target" — S3 doesn't provide native NFS/SMB access (would need Gateway)
- **Keywords:** "NetApp" + "both NFS and SMB" (FSx for NetApp ONTAP is the multiprotocol answer), "DataSync" for migration, "maintain both access patterns" (multiprotocol filesystem required)

### Compare: Snow Family Devices

| Aspect | Snowcone (HDD) | Snowcone (SSD) | Snowball Edge Storage Opt. | Snowball Edge Compute Opt. | Snowmobile |
|--------|----------------|----------------|---------------------------|---------------------------|------------|
| **Storage** | 8 TB | 14 TB | 80 TB | 28 TB NVMe | 100 PB |
| **Compute** | 2 vCPU, 4 GB | 2 vCPU, 4 GB | 40 vCPU, 80 GB | 104 vCPU, 416 GB | N/A |
| **GPU** | No | No | No | Optional V100 | N/A |
| **Weight** | 4.5 lbs | 4.5 lbs | ~50 lbs | ~50 lbs | N/A (container) |
| **Networking** | 2x 1/10 GbE | 2x 1/10 GbE | 10/25/100 GbE | 10/25/100 GbE | 1 Tbps |
| **Clustering** | No | No | Yes (5-15) | Yes (5-15) | No |
| **DataSync built-in** | YES | YES | No | No | No |
| **EC2** | sbe1.small | sbe1.small | sbe-c/g instances | sbe-c/g (larger) | No |
| **NFS interface** | Yes | Yes | Yes | Yes | No |
| **S3 interface** | Yes | Yes | Yes | Yes | No |
| **Use case** | Portable/tactical | Portable/tactical | Large migration, edge storage | Heavy edge compute, ML | Data center evacuation |
| **Exam trigger** | "Portable" + "small" | "Portable" + "SSD" | "Large transfer" + "edge" | "ML at edge" + "GPU" | ">10 PB" |

### Compare: DataSync vs. Storage Gateway vs. Transfer Family

| Aspect | DataSync | Storage Gateway | Transfer Family |
|--------|----------|-----------------|-----------------|
| **Purpose** | Migrate/sync data between storage systems | Hybrid cloud storage (on-prem cache to cloud) | User/partner file transfer via protocols |
| **Direction** | Bidirectional (any source → any target) | Bidirectional (on-prem ↔ cloud) | Bidirectional (users ↔ S3/EFS) |
| **Protocols (source)** | NFS, SMB, HDFS, S3 | NFS, SMB, iSCSI, VTL | SFTP, FTPS, FTP, AS2 |
| **Backend** | S3, EFS, FSx | S3, EBS, Glacier (Tape) | S3, EFS |
| **Agent** | VM on-premises (or EC2) | VM on-premises (or EC2) | None (managed endpoints) |
| **Caching** | No (transfer-only) | YES (local cache for low-latency access) | No |
| **Use case** | Bulk migration, scheduled sync | Ongoing hybrid access to cloud storage | External file exchange |
| **Users** | System-to-system (no human interaction) | Applications/users via NFS/SMB/iSCSI | External partners via SFTP/FTPS |
| **Pricing** | Per-GB transferred | Per-GB stored + requests | Per-hour + per-GB |
| **Exam trigger** | "Migrate files" / "sync" / "NAS to S3" | "Hybrid" / "local cache" / "extend storage" | "SFTP" / "partners" / "file exchange" |

### Compare: Data Transfer Services — When to Use Each

| Scenario | Best Service | Why |
|----------|-------------|-----|
| Migrate 500 TB, limited bandwidth | Snow Family (Snowball Edge) | Physical transport faster than network |
| Migrate NFS share to EFS | DataSync | Native NFS → EFS support; incremental |
| Partners upload via SFTP | Transfer Family | Managed SFTP; no client changes |
| Ongoing NFS cache with cloud backup | Storage Gateway (File) | Local cache + automatic S3 backup |
| Database migration | DMS | Structured data with CDC — not DataSync |
| Server migration (OS + apps) | MGN | Full server — not Snow/DataSync |
| Cross-region S3 replication (automatic) | S3 Replication | Native S3 feature — not DataSync |
| One-time S3 → S3 cross-account | DataSync | Faster than S3 replication for one-time |
| Edge ML inference, no internet | Snowball Edge Compute Opt. | GPU + storage + offline |
| Tiny remote IoT data collection | Snowcone | Portable; DataSync agent for optional online |

### All "Gotcha" Differences

| Gotcha | Explanation |
|--------|-------------|
| Snow imports to S3 Standard ONLY | Cannot import directly to Glacier/IA; use lifecycle policy after import |
| Snowcone has DataSync built-in | Unique to Snowcone; can transfer online OR ship offline |
| DataSync agent NOT needed for AWS-to-AWS | S3→S3, EFS→EFS, FSx→FSx = service-native, no agent |
| Transfer Family FTP = VPC only | Unencrypted FTP cannot be internet-facing; must be internal |
| DataSync ≠ Storage Gateway | DataSync = transfer; Storage Gateway = hybrid cache + access |
| Transfer Family ≠ DataSync | Transfer Family = user-facing SFTP/FTPS; DataSync = system migration |
| Snow device encryption key ≠ on device | KMS key never stored on device; only in AWS KMS |
| DataSync preserves metadata | Permissions, timestamps, ACLs are maintained (unlike simple S3 copy) |
| Snowball Edge = compute too | Not just storage; runs EC2, Lambda, IoT Greengrass |
| Snowmobile ≠ first choice for <10 PB | Multiple Snowball Edges more practical below ~10 PB |
| Transfer Family per-protocol-hour | Charges even with zero usage; expensive for low-volume |
| DataSync agent ≠ MGN agent ≠ ADS agent | Three different agents; different purposes; can coexist |
| Snow export exists | S3 → Snow device → on-premises (not just import!) |
| DataSync filters | Can include/exclude files by pattern (don't transfer everything) |
| Snowball Edge NFS ≠ S3 interface | Two different interfaces on device; choose based on source system |

### Transfer Time Quick Math (Exam Calculator)

**Formula:** Time (seconds) = Data (bits) / Bandwidth (bits/second)

**Shortcuts:**
- 1 TB at 1 Gbps ≈ 2.5 hours (including protocol overhead)
- 10 TB at 1 Gbps ≈ 1 day
- 100 TB at 1 Gbps ≈ 12 days
- 1 PB at 1 Gbps ≈ 4 months

**Multiply bandwidth by 10 = divide time by 10:**
- 10 TB at 10 Gbps ≈ 2.5 hours
- 100 TB at 10 Gbps ≈ 1 day

**Snow Device Total Time (approximate):**
- Order + ship to you: 3-5 days
- Load data: 1-3 days (depends on data size and local network)
- Ship back: 2-3 days
- Import to S3: 1-2 days
- **Total: ~7-14 days regardless of data size (up to device capacity)**

**Decision Rule:** If network transfer time > 1 week AND Snow device total time < network time → use Snow.
