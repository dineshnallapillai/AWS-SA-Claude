# AWS SAP-C02 Study Notes: AWS Migration Hub, Application Discovery Service & AWS MGN

## 1. Core Concepts & Theory

### AWS Migration Hub

**Purpose:** Single pane of glass to track migration progress across multiple AWS and partner tools.

| Feature | Description |
|---------|-------------|
| **Migration Tracker** | Track status of each application/server through migration phases |
| **Migration Hub Orchestrator** | Predefined and custom workflow templates for orchestrating migrations |
| **Migration Hub Refactor Spaces** | Manage incremental refactoring (strangler fig pattern) |
| **Migration Hub Strategy Recommendations** | Right-strategy recommendations based on discovery data |
| **Home Region** | Must select a home region; all discovery and migration data stored here |

**Key Architecture:**
```
On-Premises ──► Discovery (ADS) ──► Migration Hub (tracking) ──► Migration Tools (MGN, DMS, etc.)
                                           │
                                    Strategy Recommendations
                                           │
                                    Orchestrator (workflows)
```

**Migration Hub Integrations:**

| Tool | What It Reports to Hub |
|------|----------------------|
| AWS MGN | Server migration status |
| AWS DMS | Database migration status |
| AWS SMS (deprecated) | Legacy server migration status |
| Partner tools (CloudEndure, ATADATA, RiverMeadow) | Third-party migration status |

**Migration Hub Orchestrator:**
- Predefined templates for common migration patterns (rehost SAP, rehost SQL Server, etc.)
- Custom templates for unique workflows
- Automates steps: create target infra → replicate → test → cutover → validate
- Integrates with AWS Step Functions under the hood
- Supports manual approval steps (human gates)

**Migration Hub Refactor Spaces:**
- Manages the incremental refactoring of applications (strangler fig)
- Creates infrastructure for routing (API Gateway, NLB) between old and new services
- Supports gradual traffic shifting from monolith to microservices
- Components: Environment → Application → Service → Route
- Can route to Lambda functions or existing endpoints

**Migration Hub Strategy Recommendations:**
- Analyzes data collected by Application Discovery Service
- Recommends migration strategy (R-type) per server/application
- Considers: OS, runtime, database dependencies, utilization
- Provides right-sizing recommendations for target EC2 instances
- Integrates with business priorities (speed vs. optimization)

### AWS Application Discovery Service (ADS)

**Purpose:** Discover on-premises servers, collect configuration/utilization data, and map dependencies.

| Discovery Method | Agentless | Agent-Based |
|-----------------|-----------|-------------|
| **Deployment** | OVA on VMware vCenter | Install agent on each server |
| **Supported platforms** | VMware vCenter 5.5+ ONLY | Windows, Linux (any hypervisor or physical) |
| **Data collected** | VM inventory, configuration, CPU/memory/disk utilization, MAC addresses | Same as agentless PLUS network connections (TCP), running processes, system performance |
| **Network dependencies** | NO | YES (TCP connections between servers) |
| **Process information** | NO | YES (running processes with names) |
| **Performance data** | Basic (CPU, memory, disk) | Detailed (per-process CPU, memory, network I/O) |
| **Update frequency** | Every 60 minutes | Every 15 seconds (processes), every 15 minutes (system) |
| **Use case** | Quick initial inventory of VMware environments | Deep discovery with full dependency mapping |

**Key Facts:**
- Data stored in Migration Hub's home region (encrypted with KMS)
- Data retained for up to 2 years
- Can export data to CSV or query via API
- Integrates directly with Migration Hub for visualization
- Can export to Amazon Athena for complex queries (via S3 export)
- **Import option:** Upload existing inventory data (CSV) if you already have CMDB data

**Agentless Discovery Connector:**
- Deployed as OVA (Open Virtual Appliance) in VMware environment
- Connects to vCenter Server API
- Does NOT require installation on individual VMs
- Cannot see inside the OS (no process/network data)
- Good for initial "what do we have?" inventory
- Supports VMware vCenter 5.5, 6.0, 6.5, 6.7, 7.0

**Discovery Agent:**
- Installed on each server (Windows or Linux)
- Requires root/admin privileges
- Captures TCP network connections → builds dependency map
- Identifies which servers talk to which (critical for wave planning)
- Works on physical servers, VMware, Hyper-V, any platform
- Agent auto-updates itself
- Outbound HTTPS (443) to AWS endpoints required

**Dependency Mapping:**
- Only available with agent-based discovery
- Shows TCP connections between servers
- Visualized in Migration Hub as a network diagram
- Critical for determining migration wave groups
- Identifies "move groups" — servers that must migrate together

### AWS Application Migration Service (AWS MGN)

**Purpose:** Automated lift-and-shift (rehost) migration of servers to AWS.

**How It Works:**
```
Source Server ──► Replication Agent ──► Staging Area (lightweight EC2 + EBS) ──► Target EC2
    │                    │                         │                                  │
    │              Continuous                  Replication                         Launch
    │              block-level                 Subnet                             (cutover)
    │              replication                                                        │
    └──── Initial sync ─────────────────────────────────────────────────────────── Test/Cutover
```

**Core Components:**

| Component | Role |
|-----------|------|
| **Replication Agent** | Installed on source server; captures block-level changes |
| **Staging Area** | Lightweight EC2 instances + EBS volumes in target AWS account; receives replicated data |
| **Replication Server** | The EC2 instance in staging that receives data (t3.small by default) |
| **Launch Template** | Defines target EC2 instance configuration (type, subnet, SGs, IAM role) |
| **Lifecycle states** | Not ready → Ready for testing → Test in progress → Ready for cutover → Cutover in progress → Cutover complete |

**Replication Details:**
- **Block-level replication** (not file-level, not snapshot-based)
- **Continuous** — captures every write to disk in near real-time
- **Initial sync:** Full disk copy (can take hours-days depending on size)
- **Ongoing replication:** Only changed blocks (delta)
- **Compression:** Data compressed during transfer
- **Encryption:** TLS in transit; target EBS can be encrypted with KMS
- **Bandwidth throttling:** Configurable to avoid saturating source network
- **No reboot required** on source server during replication
- **No snapshots** — continuous stream, not periodic snapshots (this is an exam differentiator)

**Supported Source Servers:**

| Platform | Supported |
|----------|-----------|
| Physical servers | Yes |
| VMware | Yes |
| Hyper-V | Yes |
| Azure VMs | Yes (cloud-to-cloud) |
| GCP VMs | Yes (cloud-to-cloud) |
| AWS EC2 (other account/region) | Yes |
| Windows Server 2003+ | Yes |
| Linux (major distros) | Yes |

**Migration Lifecycle:**

1. **Install Agent** → Agent installed on source server
2. **Initial Sync** → Full block-level copy to staging EBS
3. **Continuous Replication** → Ongoing delta sync (seconds of lag)
4. **Launch Test Instance** → Spin up target EC2 for validation (non-disruptive)
5. **Mark Ready for Cutover** → Testing passed
6. **Perform Cutover** → Final sync, launch production target, source can be decommissioned
7. **Finalize Cutover** → Clean up staging resources

**Launch Settings (configurable per server):**

| Setting | Options |
|---------|---------|
| Instance type | Any EC2 type (right-size opportunity) |
| Subnet | Target VPC/subnet |
| Security groups | Target SGs |
| Instance profile | IAM role for the target |
| EBS volume type | gp3, io1, io2, etc. (can change from source) |
| EBS encryption | Enable + specify KMS key |
| Tags | Apply tags to target instances |
| Tenancy | Default, Dedicated, Host |
| License | BYOL or license-included |

**Post-Launch Actions:**
- Automated actions after instance launch (e.g., install CloudWatch agent, join AD domain)
- Configured via SSM documents (Systems Manager Automation)
- Run after test launch and/or cutover launch
- Examples: change hostname, update DNS, install monitoring agents, configure backups

### Key Limits & Quotas

| Resource | Default Limit |
|----------|---------------|
| Source servers per account (MGN) | 3,000 |
| Concurrent replicating servers | 300 |
| Replication servers per staging subnet | Based on subnet IP capacity |
| Discovery agents (ADS) | 1,000 per account |
| Agentless connectors (ADS) | 1 per vCenter (can have multiple vCenters) |
| Data retention (ADS) | 2 years |
| MGN free period | 90 days per source server (from first replication) |
| Supported OS (MGN) | Windows Server 2003+, most Linux distros |

### Networking Requirements

**MGN Replication Traffic:**

| Direction | Port | Protocol | Purpose |
|-----------|------|----------|---------|
| Source → AWS (TCP 443) | 443 | HTTPS | Agent communication with MGN service |
| Source → Staging (TCP 1500) | 1500 | TCP | Data replication to staging replication server |
| Staging → Source | None | N/A | No inbound to source required |

**Connectivity Options:**
- **Public internet:** Source agent connects directly (simplest)
- **VPN:** Agent traffic routes over site-to-site VPN
- **Direct Connect:** Agent traffic routes over DX (recommended for large-scale / sensitive)
- **Private IP replication:** MGN can use private IPs only (no public IP on replication server) — requires VPN/DX

---

## 2. Design Patterns & Best Practices

### Discovery → Planning → Migration Pattern

```
Phase 1: DISCOVER (2-4 weeks)
├── Deploy agentless connector (VMware inventory)
├── Install agents on key servers (dependency mapping)
├── Collect 2+ weeks of utilization data
└── Export to Athena for analysis

Phase 2: PLAN (2-4 weeks)
├── Analyze dependencies → define move groups
├── Assign migration strategy (7 Rs) per application
├── Use Strategy Recommendations for right-sizing
├── Define wave plan (ordered move groups)
└── Set up landing zone (Control Tower)

Phase 3: MIGRATE (ongoing, wave-based)
├── Wave 0: Landing zone, networking, security
├── Wave 1: Simple, low-risk applications (prove process)
├── Wave 2-N: Increasing complexity
└── Final: Mission-critical applications

Phase 4: OPTIMIZE (post-migration)
├── Right-size with Compute Optimizer
├── Implement managed services (replatform opportunities)
├── Purchase Savings Plans / RIs
└── Decommission source infrastructure
```

### Large-Scale Migration Pattern (hundreds/thousands of servers)

1. **Factory model:** Establish repeatable migration process
2. **Wave sizing:** 10-50 servers per wave (manageable cutover windows)
3. **Parallel workstreams:** Multiple waves in different lifecycle stages simultaneously
4. **Automation:** Use MGN API + CloudFormation for consistency
5. **Dedicated team roles:** Wave lead, app owner, test lead, network engineer

### Hybrid Replication Pattern

- **Challenge:** Some servers need private connectivity, others can use public
- **Solution:** Configure MGN with both public and private replication options
  - Private IP replication for sensitive servers (via DX/VPN)
  - Public IP replication for non-sensitive servers (faster setup)
  - Configure per source server in replication settings

### Multi-Account Migration Pattern

- **Source:** Single on-premises environment
- **Target:** Multiple AWS accounts (per application, per team, or per environment)
- MGN replicates into a central "migration" account
- Post-cutover: use AWS RAM or re-replicate to final account
- Alternative: Install agents pointing to different accounts directly

### Dependency-Aware Wave Planning

```
Step 1: Agent-based discovery (2+ weeks TCP data)
Step 2: Export network connections to Athena
Step 3: Query for clusters (servers with heavy mutual communication)
Step 4: Each cluster = one move group (migrate together)
Step 5: Order waves by dependency direction:
        - Downstream services first (databases, shared services)
        - Upstream services second (web tier, user-facing)
Step 6: Validate with application owners
```

### Anti-Patterns

| Anti-Pattern | Why It's Wrong | Better Approach |
|-------------|----------------|-----------------|
| Skipping discovery | Unknown dependencies → broken migrations | Always run ADS for 2+ weeks minimum |
| Agentless only for wave planning | No network dependency data | Use agent-based for dependency mapping |
| Migrating all servers in one wave | Too much risk, too many things to troubleshoot | Wave-based, 10-50 servers per wave |
| Not testing before cutover | Production failures at cutover | Always run test launch, validate, then cutover |
| Using MGN for database migration | MGN copies the full server; you want data-level migration | Use DMS for databases (supports CDC) |
| Running replication for months without cutover | Exceeds 90-day free tier; wastes staging resources | Plan cutover date when starting replication |
| Hardcoding private IPs in migrated apps | Break when IPs change post-migration | Use DNS names, service discovery, or parameter store |
| Not throttling replication bandwidth | Saturates production network on-premises | Configure bandwidth throttling in MGN |

### Well-Architected Alignment

- **Reliability:** Test launch validates target before cutover; continuous replication means near-zero RPO
- **Security:** Private IP replication, encryption in transit/at rest, IAM roles per server, no inbound to source
- **Cost:** 90-day free period, right-size at launch, clean up staging after cutover
- **Performance:** Bandwidth throttling protects source; right-size target instances
- **Operational Excellence:** Migration Hub tracking, post-launch automation (SSM), wave-based approach
- **Sustainability:** Right-size from day one; use discovery data to eliminate waste

---

## 3. Security & Compliance

### IAM Roles & Policies for MGN

| Role | Purpose |
|------|---------|
| `AWSApplicationMigrationReplicationServerRole` | Attached to replication server (staging EC2) |
| `AWSApplicationMigrationConversionServerRole` | Used during conversion (boot volume modification) |
| `AWSApplicationMigrationMGNRole` | Service role for MGN service operations |
| `AWSApplicationMigrationAgentRole` | Credentials used by the agent on source server (via IAM user or role) |
| `AWSApplicationMigrationLaunchInstanceWithDrsRole` | Optional: integrates with Elastic Disaster Recovery |

**Agent Credentials:**
- Agent on source server needs AWS credentials to communicate with MGN
- Options: IAM user access keys (stored on source) or use EC2 instance profile (for cloud-to-cloud)
- **Best practice:** Use dedicated IAM user with minimal permissions; rotate keys

### Encryption

| Data State | Encryption |
|------------|-----------|
| Data in transit (source → staging) | TLS (TCP port 1500) |
| Data at rest (staging EBS) | Optional KMS encryption (recommended) |
| Data at rest (target EBS) | Configurable — can enable with custom KMS key |
| ADS data (stored in Migration Hub) | Encrypted with service-managed KMS key |
| Replication metadata | Encrypted in MGN service |

### Network Security

- **Replication subnet:** Dedicated private subnet; no public access needed if using DX/VPN
- **Staging security group:** Auto-created; allows inbound TCP 1500 from source IPs only
- **No inbound to source:** Agent initiates all outbound connections
- **VPC endpoints:** MGN supports PrivateLink endpoints (no internet traversal for API calls)
- **Private IP mode:** Replication servers have no public IPs; all traffic via DX/VPN

### Compliance Considerations

- **Data residency:** All replication data lands in configured target region — verify compliance
- **ADS data:** Stored in home region only; cannot be moved after selection
- **Audit:** CloudTrail logs all MGN and ADS API calls
- **Access control:** Use SCPs to restrict which accounts can run migrations
- **Sensitive discovery data:** ADS agent collects running processes and network connections — may be sensitive in some environments (discuss with security team)

### Security Best Practices

1. Use private IP replication for regulated workloads (no public internet for data)
2. Enable EBS encryption on all target volumes (KMS CMK preferred)
3. Restrict agent IAM credentials to minimum required permissions
4. Use dedicated replication subnet with restrictive NACLs
5. Rotate agent credentials after migration completes
6. Delete staging resources promptly after successful cutover
7. Audit MGN actions via CloudTrail
8. Use VPC endpoints for MGN API calls in sensitive environments

---

## 4. Cost Optimization

### AWS MGN Pricing

| Component | Cost |
|-----------|------|
| MGN service (first 90 days per server) | **FREE** |
| MGN service (after 90 days) | Per-hour per replicating server |
| Replication server (staging EC2) | Standard EC2 pricing (t3.small default) |
| Staging EBS volumes | Standard EBS pricing (same size as source disks) |
| Data transfer (source → AWS) | **FREE** (inbound to AWS is always free) |
| Target EC2 instances (test/cutover) | Standard EC2 pricing |
| Snapshots (created during cutover) | Standard EBS snapshot pricing |

### Application Discovery Service Pricing

| Component | Cost |
|-----------|------|
| ADS service | **FREE** |
| Agentless connector (OVA) | **FREE** |
| Discovery agents | **FREE** |
| Data storage (in Migration Hub) | **FREE** (up to 2 years) |
| Athena queries (if exported) | Standard Athena pricing |

### Migration Hub Pricing

| Component | Cost |
|-----------|------|
| Migration Hub tracking | **FREE** |
| Strategy Recommendations | **FREE** |
| Refactor Spaces | Charges for underlying resources (API Gateway, NLB) |
| Orchestrator | **FREE** (pay for underlying resources like Step Functions) |

### Cost Traps

| Trap | Impact | Mitigation |
|------|--------|------------|
| Exceeding 90-day free period | Per-hour charges for each server still replicating | Plan cutover within 90 days |
| Staging EBS accumulation | Paying for full-size EBS volumes per server during replication | Minimize time between start and cutover |
| Over-sized replication servers | Paying for larger EC2 than needed in staging | Use default t3.small; only upsize if replication is slow |
| Forgetting to clean up staging | Ongoing costs after cutover complete | Finalize cutover (auto-cleans) or manually terminate |
| Test instances left running | Paying for EC2 instances used for testing | Terminate test instances after validation |
| Right-sizing at launch missed | Running over-provisioned instances post-migration | Use Compute Optimizer data + MGN right-sizing recommendations |
| Source server still running | Paying on-prem costs PLUS AWS costs | Decommission source after validation window |

### Cost Optimization Strategies

1. **Batch waves to stay within 90 days** — plan replication start → cutover within free window
2. **Right-size at launch** — don't replicate 1:1 instance size; use Strategy Recommendations
3. **Finalize cutover immediately** — triggers automatic staging cleanup
4. **Delete test instances** after validation passes
5. **Use Spot for test launches** (non-production validation)
6. **Throttle replication bandwidth** wisely — too slow = longer staging = higher cost
7. **Clean up abandoned migrations** — servers where migration was cancelled

### Bandwidth Cost Calculation

- **Data transfer INTO AWS = FREE** (replication inbound is free)
- **Key insight:** You don't pay for the actual data being replicated to AWS
- You DO pay for: staging infrastructure (EC2 + EBS) that receives and stores data
- For large volumes: faster replication = shorter staging duration = lower cost

---

## 5. High Availability, Disaster Recovery & Resilience

### MGN Replication Resilience

| Scenario | Behavior |
|----------|----------|
| Source server reboots | Agent reconnects automatically; replication resumes |
| Network interruption | Agent retries; resumes from last acknowledged block |
| Replication server failure | MGN auto-replaces replication server; resync from last checkpoint |
| Agent crash | Agent auto-restarts; replication continues |
| Source disk grows (new volume added) | Must add to replication settings manually |

### RPO/RTO with MGN

| Metric | Value | Notes |
|--------|-------|-------|
| **RPO** | Seconds (sub-minute) | Continuous replication; lag is typically seconds |
| **RTO** | Minutes (5-20 min) | Time to launch target EC2 from replicated data |
| **Initial sync RPO** | Hours-days | Until initial sync completes, RPO = total data size |
| **Test launch RTO** | Minutes | Non-disruptive; doesn't affect replication |

### Cutover Resilience

- **Rollback plan:** Source server is NEVER modified by MGN — it's still running and serving traffic until you switch DNS/routing
- **Replication continues during testing** — cutover doesn't stop replication
- **Final sync at cutover:** One last sync to capture any writes between last replication and cutover moment
- **Validation window:** Keep source running for X days after cutover; roll back by switching routing

### Multi-AZ Considerations Post-Migration

- MGN launches single EC2 instance (one AZ) by default
- For HA post-migration, you must:
  - Create AMI from migrated instance
  - Deploy across multiple AZs via ASG
  - Add ELB for traffic distribution
  - Or configure application-level replication (RDS Multi-AZ for databases)
- This is a **post-launch action** — MGN handles rehost, you handle HA design

### Disaster Recovery Integration

- **AWS Elastic Disaster Recovery (DRS):** Same underlying technology as MGN
- MGN = migration (one-time move); DRS = ongoing DR (continuous replication for failover)
- After migration: switch from MGN to DRS for ongoing protection
- DRS provides: continuous replication, point-in-time recovery, automated failover/failback

### Resilience Best Practices

1. Always run a test launch before cutover (validates boot, networking, application)
2. Keep source running for 1-2 weeks post-cutover (rollback safety net)
3. Monitor replication lag before cutover (should be seconds, not minutes)
4. Use multiple DX connections or VPN backup for replication path redundancy
5. Plan for AZ failure post-migration (don't assume single EC2 = HA)

---

## 6. Exam-Focused Section

### Straightforward Questions (7 examples)

**Q1:** A company wants to discover all servers in their VMware environment and understand basic configuration before planning a migration. They want the fastest setup with no agent installation. What should they use?

**A:** Application Discovery Service with the Agentless Connector (OVA deployed to vCenter). Provides VM inventory and utilization without installing agents on individual servers.

**Q2:** A company needs to understand network dependencies between their on-premises servers to plan migration waves. Which discovery method provides this?

**A:** Application Discovery Service with Agent-Based discovery. Only agents capture TCP connection data needed for dependency mapping.

**Q3:** A solutions architect wants a single dashboard to track migration progress across AWS MGN, AWS DMS, and a third-party migration tool. What service provides this?

**A:** AWS Migration Hub. It aggregates migration status from multiple tools into one tracking view.

**Q4:** A company wants to migrate 50 Windows servers from on-premises to EC2 with minimal downtime. The servers are running on Hyper-V. What service should they use?

**A:** AWS MGN (Application Migration Service). It supports Hyper-V source servers and provides continuous replication with minutes of downtime at cutover.

**Q5:** During an MGN migration, a solutions architect wants to automatically install the CloudWatch agent and join the server to Active Directory after the target EC2 instance launches. How should this be configured?

**A:** Configure Post-Launch Actions in MGN using SSM Automation documents. These run automatically after test or cutover launches.

**Q6:** How long is AWS MGN free per source server?

**A:** 90 days from the start of first replication for that server.

**Q7:** A company wants to right-size their EC2 instances during migration rather than replicating the same over-provisioned specs from on-premises. Which Migration Hub feature helps?

**A:** Migration Hub Strategy Recommendations. It analyzes utilization data from ADS and recommends appropriate EC2 instance types.

### Tricky / Scenario-Based Questions (7 examples)

**Q1:** A company has 1,000 servers to migrate. They deployed the Application Discovery Service agentless connector and collected 4 weeks of data. Now they're creating migration waves, but the team cannot determine which servers depend on each other. What's wrong, and what should they do?

**A:** The agentless connector does NOT collect network dependency data — it only provides VM inventory and basic utilization. They need to deploy Discovery Agents on key servers to capture TCP connections for dependency mapping.
- **Wrong:** "Collect more data with agentless" — more time won't add dependency data
- **Wrong:** "Use VPC Flow Logs" — servers are on-premises, not in AWS yet
- **Keywords:** "cannot determine dependencies" + "agentless connector" = missing agent-based discovery

**Q2:** A company is migrating 200 servers using AWS MGN. After 100 days, they notice unexpected charges on their bill related to MGN. The migration team says they haven't performed cutover yet for 150 servers. What is causing the charges?

**A:** MGN is only free for 90 days per server. After day 90, per-hour charges apply for each server still in replication state. 150 servers have been replicating for 100 days (10 days past the free period). They should either accelerate cutover or pause replication for servers not ready.
- **Wrong:** "Data transfer charges" — inbound data transfer to AWS is free
- **Wrong:** "ADS charges" — ADS is free
- **Keywords:** "100 days" > 90-day free period, "150 servers" still replicating

**Q3:** A company wants to migrate a sensitive healthcare application. Regulations prohibit any data from traversing the public internet. They have AWS Direct Connect established. How should they configure MGN replication?

**A:** Enable Private IP replication mode in MGN. Configure replication servers in the staging subnet with NO public IP. Agent traffic routes through the existing Direct Connect connection to the private replication endpoint. Also enable EBS encryption with CMK on staging and target volumes.
- **Wrong:** "Use public replication with VPN" — VPN over internet still technically traverses internet infrastructure for some interpretations
- **Wrong:** "Use S3 as replication target" — MGN replicates to EBS via replication servers, not S3
- **Keywords:** "prohibit any data traversing public internet" (private IP mode), "Direct Connect" (private connectivity), "healthcare" (encryption required)

**Q4:** A company discovers that their main Oracle database server communicates with 30 application servers. They want to migrate the application servers first and keep the database on-premises temporarily. Network latency between on-premises and AWS must be minimal. What should the solutions architect recommend?

**A:** Rehost the 30 application servers using MGN. Maintain connectivity to the on-premises Oracle database via Direct Connect (low latency). Application servers in AWS connect back to on-premises DB over DX. Migrate the database later using DMS with CDC when ready. Consider connection pooling (RDS Proxy pattern) to manage connections across the hybrid boundary.
- **Wrong:** "Migrate everything together" — database may not be ready, and this is a valid phased approach
- **Wrong:** "Use VPN for database connectivity" — question says "minimal latency" which points to Direct Connect
- **Keywords:** "keep database on-premises temporarily" (hybrid state), "minimal latency" (Direct Connect, not VPN)

**Q5:** A company has completed migration of 500 servers using MGN. The migration team finalized cutover for all servers a week ago, but the finance team reports ongoing charges for replication infrastructure. What is the most likely cause?

**A:** The team performed cutover but did NOT finalize (disconnect from service) the servers in MGN. Until you explicitly "Mark as archived" or "Disconnect from service," staging resources (replication servers, EBS volumes) may persist. They need to finalize/archive all migrated servers in the MGN console.
- **Wrong:** "Normal charges for target EC2 instances" — those are expected and separate
- **Wrong:** "90-day charges still applying" — cutover was completed, but cleanup wasn't
- **Keywords:** "finalized cutover" but "ongoing charges for replication infrastructure" = staging not cleaned up

**Q6:** A solutions architect is planning a migration for a VMware environment with 2,000 VMs. They need to collect dependency data but don't want to install agents on all 2,000 servers. What's the most efficient approach?

**A:** Deploy the agentless connector first for full VM inventory. Then identify the critical/complex applications (probably 100-200 servers) and install agents ONLY on those to get dependency mapping where it matters most. Use agentless data for simple, independent workloads that can be migrated without deep dependency analysis.
- **Wrong:** "Install agents on all 2,000" — impractical and unnecessary for simple workloads
- **Wrong:** "Use agentless only" — won't give dependency data for complex apps
- **Keywords:** "don't want to install agents on all 2,000" (hybrid approach), need "dependency data" (agents needed for that specifically)

**Q7:** A company is migrating a Windows Server 2012 R2 application server using MGN. The test launch succeeds, but the application fails to connect to a shared file server that's still on-premises. The security team confirms network connectivity (VPN) is working and ports are open. What should the solutions architect investigate?

**A:** Check DNS resolution. The migrated server likely has hardcoded DNS settings pointing to on-premises DNS servers. After migration, it may be using AWS-provided DNS (VPC resolver) that can't resolve on-premises hostnames. Solution: configure DHCP option sets with on-premises DNS servers, or set up Route 53 Resolver (inbound/outbound endpoints) for hybrid DNS resolution.
- **Wrong:** "MGN corrupted the application" — MGN does block-level copy; application files are identical
- **Wrong:** "Security group issue" — security team confirmed ports are open
- **Keywords:** "application fails to connect" + "connectivity confirmed" + "on-premises file server" = DNS resolution (most common post-migration issue)

### Common Exam Traps & Pitfalls

| Trap | Reality |
|------|---------|
| "Agentless gives dependency maps" | NO — only agent-based provides TCP connection data for dependencies |
| "MGN creates snapshots for replication" | NO — continuous block-level replication, NOT snapshot-based |
| "MGN requires downtime during replication" | NO — replication is non-disruptive; only cutover requires brief downtime |
| "CloudEndure is the current tool" | NO — CloudEndure is deprecated; AWS MGN replaced it |
| "MGN migrates databases optimally" | NO — use DMS for databases; MGN migrates the whole server (overkill for just data) |
| "Migration Hub performs migrations" | NO — Migration Hub only TRACKS; actual migration is done by MGN, DMS, etc. |
| "ADS agentless works on Hyper-V" | NO — agentless ONLY works with VMware vCenter |
| "Data transfer charges during replication" | NO — inbound to AWS is free; you pay for staging infra, not transfer |
| "MGN modifies the source server" | NO — source is never changed; only agent is added (read-only data capture) |
| "You must use ADS before MGN" | NO — ADS is recommended but not required; you can install MGN agent directly |
| "Migration Hub home region can be changed" | NO — once selected, home region is permanent for that account |

---

## 7. Cheat Sheet

### Must-Know Facts

- **Migration Hub:** Tracking only (doesn't perform migration); free; requires home region selection (permanent)
- **ADS Agentless:** VMware only, no dependencies, quick inventory, OVA on vCenter
- **ADS Agent:** Any OS/platform, TCP dependencies, running processes, install per server
- **MGN:** Block-level continuous replication, free 90 days, supports physical/virtual/cloud
- **MGN ports:** TCP 443 (agent→MGN service) + TCP 1500 (agent→replication server)
- **MGN never modifies source** — read-only replication; source keeps running
- **MGN is NOT snapshot-based** — continuous block stream
- **Post-launch actions:** SSM documents for automation after launch
- **Finalize cutover:** Must explicitly archive/disconnect to clean staging resources
- **Refactor Spaces:** Infrastructure for strangler fig pattern (API Gateway + NLB routing)
- **Strategy Recommendations:** Analyzes ADS data → recommends R-type + right-size EC2
- **Orchestrator:** Workflow templates for migration steps (predefined for SAP, SQL Server, etc.)

### Key Differentiators

| MGN vs. DMS | MGN = entire server (OS+apps+data); DMS = database data only |
|---|---|
| **MGN vs. DataSync** | MGN = server migration to EC2; DataSync = file/object transfer to S3/EFS/FSx |
| **MGN vs. DRS** | MGN = one-time migration; DRS = ongoing disaster recovery |
| **MGN vs. Snow Family** | MGN = online replication; Snow = offline physical transport |
| **ADS Agentless vs. Agent** | Agentless = VMware inventory only; Agent = any platform + dependencies |
| **Migration Hub vs. MGN** | Hub = tracking dashboard; MGN = actual migration engine |
| **Refactor Spaces vs. MGN** | Refactor Spaces = incremental refactoring routing; MGN = lift-and-shift |

### Exam Keyword Triggers

| If the question says... | Think... |
|------------------------|----------|
| "Track migration progress" / "single dashboard" | Migration Hub |
| "Discover on-premises servers" / "inventory" | Application Discovery Service |
| "Dependency mapping" / "network connections" / "which servers communicate" | ADS Agent-Based |
| "VMware inventory" / "quick discovery" / "no agent install" | ADS Agentless Connector |
| "Lift and shift" / "rehost" / "server migration" | AWS MGN |
| "Continuous replication" / "minimal downtime" / "block-level" | AWS MGN |
| "Migrate physical servers" | AWS MGN (supports physical) |
| "Cloud-to-cloud migration" (Azure/GCP → AWS) | AWS MGN |
| "Strangler fig" / "incremental refactoring" / "gradual migration to microservices" | Migration Hub Refactor Spaces |
| "Orchestrate migration workflow" / "automate migration steps" | Migration Hub Orchestrator |
| "Right-size during migration" / "recommend strategy" | Strategy Recommendations |
| "After migration, continuous DR" | Switch to Elastic Disaster Recovery (DRS) |

### Decision Flowchart

```
What do you need to do?
│
├── DISCOVER what's on-premises
│   ├── Quick VMware inventory → Agentless Connector
│   └── Deep dependencies + any platform → Discovery Agent
│
├── PLAN the migration
│   ├── Which strategy per app? → Strategy Recommendations
│   ├── Which servers move together? → ADS dependency map
│   └── What order/workflow? → Migration Hub Orchestrator
│
├── TRACK migration progress
│   └── Migration Hub (aggregates MGN, DMS, partner tools)
│
├── MIGRATE servers (lift & shift)
│   └── AWS MGN
│       ├── Physical, VMware, Hyper-V, cloud → All supported
│       ├── Need private connectivity → Private IP replication mode
│       └── Post-launch automation → SSM documents
│
├── MIGRATE databases
│   └── AWS DMS (not MGN)
│
├── MIGRATE file shares
│   └── AWS DataSync (not MGN)
│
└── INCREMENTAL REFACTORING (strangler fig)
    └── Migration Hub Refactor Spaces
```

---

## 8. Additional Deep-Dive

### 10 More Tricky Scenario-Based Questions

**Q1:** A company completed Application Discovery Service agent-based collection for 6 weeks. The data shows Server A communicates with Servers B, C, and D via TCP. Server B also communicates with Server E (which is a database scheduled for DMS migration later). The team wants to migrate Servers A, B, C, and D together. Is this a valid move group?

**A:** Yes, A-B-C-D form a valid move group since they communicate directly. Server E (database) can remain on-premises temporarily if DX/VPN maintains connectivity. However, verify that post-migration latency between B and E (now cross-network) is acceptable. If B's communication with E is latency-sensitive, either include E in the wave or ensure Direct Connect provides adequate latency.
- **Keywords:** "communicates with" (dependency), "database scheduled for DMS later" (phased approach), "together" (move group)

**Q2:** A multinational company has 5,000 servers across 3 data centers in different countries. They want to use Migration Hub to track all migrations. Each data center should migrate to the nearest AWS region. What's the architectural consideration?

**A:** Migration Hub has a single home region per account. All discovery data and tracking aggregates there regardless of target migration regions. MGN can replicate to any target region, and Migration Hub still tracks it. However, ADS data is stored only in the home region. There's no issue migrating to multiple target regions — Migration Hub tracks cross-region migrations centrally.
- **Wrong:** "Need separate Migration Hub per region" — no, one home region tracks everything
- **Wrong:** "ADS must run in each target region" — ADS data stored centrally in home region
- **Keywords:** "different countries" / "nearest AWS region" (multi-region targets), "track all" (single Migration Hub)

**Q3:** A company starts replicating 100 servers with MGN on January 1st. By March 15th (day 74), they've only completed cutover for 30 servers. The remaining 70 are blocked by application testing delays. Budget constraints mean they cannot afford MGN charges after the free period. What should the solutions architect recommend?

**A:** They have 16 days remaining in the free period. Options: (1) Accelerate testing and cutover for as many servers as possible in 16 days. (2) For servers not ready: pause replication (disconnect from service), complete testing offline, then restart replication later (new 90-day window per restart). (3) Accept the small per-hour charge for a few extra weeks if close to ready. Note: stopping and restarting replication requires a full re-sync.
- **Wrong:** "Just continue — MGN charges are minimal" — at scale (70 servers), charges add up
- **Wrong:** "Pause replication with no impact" — pausing requires full re-sync when resumed
- **Keywords:** "day 74" + "70 remaining" + "budget constraints" (timing vs. cost optimization)

**Q4:** A solutions architect is migrating a cluster of 10 application servers using MGN. All 10 must cut over simultaneously because they share state via multicast communication. What MGN feature or approach handles simultaneous cutover?

**A:** Use MGN's "Launch all" or wave/batch cutover capability. Configure all 10 servers in the same wave and initiate cutover together. After launch, place all instances in the same placement group (cluster) for low-latency communication. Note: AWS does not support multicast in VPC by default — they must refactor multicast communication to use Transit Gateway multicast or application-level solution.
- **Wrong:** "Just cut over one at a time" — shared state requires simultaneous migration
- **Wrong:** "MGN handles multicast automatically" — multicast requires Transit Gateway multicast domain or application change
- **Keywords:** "simultaneously" (batch cutover), "multicast" (not natively supported in VPC — needs Transit Gateway multicast or refactoring)

**Q5:** During an MGN migration test launch, the target Windows server boots successfully but cannot join the Active Directory domain. The AD domain controller is on-premises, accessible via Direct Connect. DNS resolution works correctly. What's the most likely issue?

**A:** The target EC2 instance's security group likely doesn't allow the required AD ports (TCP/UDP 389 LDAP, 636 LDAPS, 88 Kerberos, 445 SMB, 135 RPC, 49152-65535 dynamic RPC). Alternatively, the on-premises AD firewall rules don't recognize the new AWS IP range. Solution: Update security groups to allow AD ports to on-premises DC, and update on-premises firewall to allow AWS CIDR.
- **Wrong:** "MGN broke the AD configuration" — MGN does block-level copy; AD config is preserved
- **Wrong:** "Need AWS Managed Microsoft AD" — on-premises AD is fine if network allows it
- **Keywords:** "cannot join AD" + "DNS works" + "Direct Connect exists" = port/firewall issue (not DNS, not connectivity)

**Q6:** A company has an existing CMDB (Configuration Management Database) with detailed server inventory from their ITSM tool. They don't want to deploy ADS agents. Can they still use Migration Hub for planning?

**A:** Yes. Application Discovery Service supports a CSV import feature. They can export their CMDB data into the ADS import template format and upload it directly to Migration Hub. This provides server inventory for tracking and planning. However, imported data won't include real-time utilization metrics or network dependency mapping — those still require agents.
- **Wrong:** "Must use ADS agents — no alternative" — import feature exists
- **Wrong:** "CMDB import gives full dependency data" — only gives static inventory, not live dependencies
- **Keywords:** "existing CMDB" + "don't want to deploy agents" (import feature)

**Q7:** A company is using MGN to migrate 50 Linux servers. Replication is working, but initial sync is taking 5+ days per server because of 2TB disks and limited bandwidth. They need to speed this up without upgrading their internet connection. What options are available?

**A:** Options: (1) Use Snowball Edge to seed the initial data to S3, then use MGN for ongoing delta replication (MGN doesn't natively integrate with Snowball, so this requires EBS snapshot import + switching to MGN). (2) Exclude unnecessary volumes from replication (e.g., temp/scratch disks). (3) Increase replication server size (more network throughput). (4) Use Direct Connect if available (may have more bandwidth than internet). (5) Accept the slow initial sync — once complete, ongoing replication is only deltas (fast).
- **Wrong:** "Use DataSync instead of MGN" — DataSync is for files/objects, not server migration
- **Wrong:** "Increase bandwidth throttling" — throttling limits max, not increases available bandwidth
- **Keywords:** "5+ days" + "2TB" + "limited bandwidth" (initial sync bandwidth challenge)

**Q8:** After migrating 200 servers to AWS using MGN, the operations team wants continuous protection against disasters for the migrated workloads. They liked the continuous replication model of MGN. What should the solutions architect recommend?

**A:** AWS Elastic Disaster Recovery (DRS). It uses the same underlying continuous block-level replication technology as MGN but is designed for ongoing DR (not one-time migration). DRS provides: continuous replication, point-in-time recovery (up to 365 days), sub-second RPO, automated failover and failback. Transition from MGN to DRS post-migration.
- **Wrong:** "Keep using MGN" — MGN is for migration, not ongoing DR; charges per-hour indefinitely
- **Wrong:** "Use AWS Backup" — backup is periodic (RPO = hours); DRS provides continuous replication (RPO = seconds)
- **Keywords:** "continuous protection" + "liked continuous replication" + "post-migration" = DRS

**Q9:** A company migrated their application tier to AWS using MGN but kept their database on-premises. Six months later, they want to migrate the database (Oracle 12c, 5TB) to Aurora PostgreSQL. The application has already been updated to use PostgreSQL-compatible queries. What tool combination should they use?

**A:** AWS SCT (Schema Conversion Tool) to convert Oracle schema to PostgreSQL + AWS DMS with CDC for data migration with minimal downtime. SCT handles stored procedures, triggers, and functions conversion. DMS handles the ongoing data replication during cutover. NOT MGN — you don't want to migrate the entire database server; you want data-level migration with schema transformation.
- **Wrong:** "MGN for the database server" — would give you Oracle on EC2, not Aurora PostgreSQL
- **Wrong:** "DMS only" — heterogeneous migration (Oracle → PostgreSQL) requires SCT for schema conversion
- **Keywords:** "Oracle to Aurora PostgreSQL" (heterogeneous = SCT + DMS), "5TB" (too large for dump/restore = use DMS CDC)

**Q10:** A company is planning migration for a legacy application running on Windows Server 2003. The application requires a specific NIC MAC address for its license validation. After MGN rehost to EC2, the MAC address will be different. What should the solutions architect recommend?

**A:** EC2 instances get new MAC addresses — this cannot be preserved from on-premises. Solutions: (1) Contact the software vendor to re-issue license for new MAC address. (2) Use EC2 ENI which has a fixed MAC address that persists through stop/start (but different from original). (3) If the application allows, configure it to use a different licensing mechanism. (4) Consider this a Replatform or Repurchase if the licensing issue is too complex. This is a common "gotcha" with lift-and-shift.
- **Wrong:** "MGN preserves MAC addresses" — impossible; AWS assigns new MACs
- **Wrong:** "Use Dedicated Host to control MAC" — Dedicated Hosts don't let you choose MAC addresses
- **Keywords:** "specific MAC address" + "license validation" (hardware-locked licensing doesn't survive rehost)

### Compare: ADS Agentless vs. Agent-Based vs. CMDB Import

| Aspect | Agentless Connector | Discovery Agent | CMDB Import |
|--------|-------------------|-----------------|-------------|
| **Setup effort** | Low (1 OVA per vCenter) | Medium (1 agent per server) | Low (CSV upload) |
| **Platforms** | VMware vCenter ONLY | Any (Windows, Linux, physical) | Any (manual data) |
| **VM inventory** | Yes | Yes | Yes |
| **CPU/Memory utilization** | Yes (basic) | Yes (detailed, per-process) | No (static data) |
| **Network dependencies** | NO | YES (TCP connections) | No |
| **Running processes** | NO | YES | No |
| **Real-time updates** | Every 60 min | Every 15 sec (processes) | Manual refresh |
| **Dependency mapping** | NO | YES | NO |
| **Use case** | Quick initial inventory | Wave planning, deep analysis | Already have data, skip discovery |
| **Exam trigger** | "VMware" + "quick" + "no agents" | "dependencies" + "move groups" | "existing CMDB" + "no agents" |

### Compare: MGN vs. DRS (Elastic Disaster Recovery)

| Aspect | AWS MGN | AWS DRS |
|--------|---------|---------|
| **Purpose** | One-time migration to AWS | Ongoing disaster recovery |
| **Duration** | Temporary (migrate then done) | Permanent (continuous protection) |
| **Replication** | Continuous block-level | Continuous block-level (same tech) |
| **Free tier** | 90 days per server | No free tier |
| **Point-in-time recovery** | No | Yes (up to 365 days of recovery points) |
| **Failback** | No (one-way migration) | Yes (automated failback to source) |
| **Recovery drill** | Test launch (manual) | Non-disruptive recovery drill |
| **Lifecycle** | Migrate → Archive → Done | Replicate → Drill → Recover → Failback |
| **When to use** | Moving workloads to AWS permanently | Protecting workloads against regional failure |
| **Post-migration** | Stop using after cutover | START using after migration for ongoing DR |

### When Does the SAP-C02 Exam Expect Me to Pick Each Service?

**Pick Application Discovery Service (Agentless) when:**
- "VMware environment" + "quick inventory" + "no agent installation"
- "How many servers do we have?" (inventory question)
- "Initial assessment" / "first step in migration planning"

**Pick Application Discovery Service (Agent-Based) when:**
- "Dependency mapping" / "which servers talk to each other"
- "Network connections between servers"
- "Plan migration waves" / "determine move groups"
- "Physical servers" (agentless can't do physical)
- "Detailed utilization" / "per-process CPU/memory"

**Pick Migration Hub when:**
- "Track progress" / "single dashboard" / "visibility across tools"
- "Which applications have been migrated?"
- "Orchestrate migration workflow" (Hub Orchestrator)
- "Recommend migration strategy" (Strategy Recommendations)
- "Incremental refactoring" / "strangler fig" (Refactor Spaces)

**Pick AWS MGN when:**
- "Rehost" / "lift and shift" / "server migration"
- "Minimal downtime" + "server migration"
- "Physical server migration"
- "Cloud-to-cloud" (Azure/GCP → AWS)
- "Continuous replication" + "server"
- "Block-level replication"

**Pick DMS (NOT MGN) when:**
- "Database migration" (even if it's on a server)
- "Schema conversion" / "heterogeneous database"
- "Ongoing replication" of database data
- "Oracle to Aurora" / "SQL Server to PostgreSQL"

**Pick DRS (NOT MGN) when:**
- "Ongoing disaster recovery" post-migration
- "Continuous protection" / "failover and failback"
- "Point-in-time recovery" for servers
- "DR drill" / "non-disruptive testing" of recovery

### All Gotcha Differences

| Gotcha | Explanation |
|--------|-------------|
| Agentless ≠ dependencies | Agentless gives inventory ONLY; need agents for TCP dependency data |
| MGN ≠ snapshots | Continuous block-level stream, NOT periodic snapshots |
| MGN ≠ DMS | MGN migrates entire server; DMS migrates database data/schema only |
| Migration Hub ≠ migration tool | Hub tracks; doesn't execute. MGN/DMS execute |
| Home region is permanent | Cannot change Migration Hub home region after selection |
| 90 days ≠ unlimited | MGN free period is exactly 90 days per server from first replication |
| Test launch ≠ cutover | Test launch doesn't stop replication or affect source |
| Cutover ≠ cleanup | Must explicitly finalize/archive to delete staging resources |
| MGN doesn't preserve MAC | New ENI = new MAC address; hardware-locked licenses break |
| MGN agent ≠ ADS agent | Different agents; different purposes; both can coexist on same server |
| Private IP mode ≠ default | Must explicitly enable; default uses public replication server IPs |
| Post-launch ≠ pre-launch | Post-launch actions run AFTER boot; can't modify boot process itself |
