# AWS SAP-C02 Study Notes: VPC Design (Subnets, NACLs, Security Groups, Flow Logs)

---

## 1. Core Concepts & Theory

### What is a VPC?

A **Virtual Private Cloud (VPC)** is a logically isolated section of the AWS cloud where you launch resources in a virtual network you define. You have full control over IP addressing, subnets, route tables, gateways, and network access control.

- VPC is **regional** — it spans all Availability Zones in a region
- Every AWS account comes with a **default VPC** in each region (CIDR `172.31.0.0/16`)
- You can create **non-default (custom) VPCs** with your own CIDR blocks

### CIDR Blocks

| Rule | Detail |
|------|--------|
| Primary CIDR | Required at VPC creation. Min `/28` (16 IPs), Max `/16` (65,536 IPs) |
| Secondary CIDRs | Up to **4 additional** CIDRs can be associated (total 5) |
| RFC 1918 ranges | `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` (recommended) |
| Non-RFC 1918 | Allowed but **cannot overlap** with any peered VPC or on-prem range |
| IPv6 | AWS assigns a `/56` CIDR from Amazon's pool (or BYOIP). Subnets get `/64` |

**Exam tip:** You **cannot change** the primary CIDR after creation, but you **can add** secondary CIDRs. If you run out of IPs, add a secondary CIDR — don't recreate the VPC.

### Subnets

A subnet is a range of IP addresses within a VPC, **tied to a single Availability Zone**.

- **Public subnet** — has a route to an Internet Gateway (IGW); instances can have public IPs
- **Private subnet** — no direct route to IGW; outbound internet via NAT Gateway/Instance
- **Isolated subnet** — no internet access at all (no NAT, no IGW route)

**Reserved IPs per subnet (5 addresses):**

| Address | Purpose |
|---------|---------|
| First IP (e.g., `10.0.0.0`) | Network address |
| Second IP (`10.0.0.1`) | VPC router |
| Third IP (`10.0.0.2`) | DNS server (VPC base + 2) |
| Fourth IP (`10.0.0.3`) | Reserved for future use |
| Last IP (`10.0.0.255`) | Broadcast (AWS doesn't support broadcast, but reserves it) |

**Exam favorite:** A `/28` subnet = 16 IPs − 5 reserved = **11 usable IPs**.

### Route Tables

- Every subnet must be associated with **exactly one** route table
- A VPC has a **main route table** (implicit association for subnets without explicit association)
- Routes are evaluated using **longest prefix match**
- Local route (`10.0.0.0/16 → local`) is **always present** and **cannot be deleted**

**Key route targets:**

| Target | Use |
|--------|-----|
| `igw-xxx` | Internet Gateway — public internet |
| `nat-xxx` | NAT Gateway — outbound-only internet for private subnets |
| `vgw-xxx` | Virtual Private Gateway — VPN |
| `tgw-xxx` | Transit Gateway — hub-and-spoke networking |
| `pcx-xxx` | VPC Peering Connection |
| `vpce-xxx` | Gateway VPC Endpoint (S3, DynamoDB) |
| `eni-xxx` | Network Interface (appliance/firewall) |

### Internet Gateway (IGW)

- **Horizontally scaled, redundant, HA** — no bandwidth constraints, no single point of failure
- Performs **1:1 NAT** between private and public/Elastic IPs
- One IGW per VPC; must be **attached** to VPC and **route table** must point to it
- **Does NOT** do any filtering — that's the job of NACLs and Security Groups

### NAT Gateway vs NAT Instance

| Feature | NAT Gateway | NAT Instance |
|---------|-------------|--------------|
| Managed by | AWS | You |
| HA | Single AZ (deploy one per AZ for HA) | Manual (scripts, ASG) |
| Bandwidth | Up to 100 Gbps (scales automatically) | Depends on instance type |
| Security Groups | **Cannot** associate SGs | **Can** associate SGs |
| Bastion/port forwarding | **No** | **Yes** |
| Cost | Hourly + data processing charge | Instance cost only |
| Source/Dest check | N/A | **Must disable** |

**Exam tip:** NAT Gateway is **single AZ**. For HA, deploy one NAT GW per AZ and update each AZ's route table. This is a very common exam question.

### Network ACLs (NACLs)

NACLs are **stateless** firewalls at the **subnet level**.

| Property | Detail |
|----------|--------|
| Scope | Subnet level — applies to all traffic entering/leaving the subnet |
| Stateful? | **No — stateless**. You must explicitly allow both inbound AND outbound (including ephemeral ports) |
| Rule evaluation | Rules processed in **order, lowest number first**. First match wins, rest ignored |
| Default NACL | **Allows all** inbound and outbound traffic |
| Custom NACL | **Denies all** inbound and outbound by default |
| Rule numbers | 1–32766. AWS recommends increments of 10 or 100 |
| Association | A subnet is associated with **exactly one** NACL. A NACL can be associated with **multiple** subnets |
| Supports | **Allow AND Deny** rules |

**Ephemeral ports:** When a client initiates a connection, the response comes back on an ephemeral port (1024–65535 for Linux, 49152–65535 for Windows/NAT GW). Your NACL outbound (or inbound, depending on direction) **must allow this range** or connections will fail silently.

**NACL rule example for web server subnet:**

```
Inbound:
  Rule 100: Allow TCP 443 from 0.0.0.0/0          (HTTPS in)
  Rule 110: Allow TCP 80  from 0.0.0.0/0           (HTTP in)
  Rule 120: Allow TCP 1024-65535 from 10.0.0.0/16   (return traffic from backend)
  Rule *:   DENY all                                 (implicit deny)

Outbound:
  Rule 100: Allow TCP 1024-65535 to 0.0.0.0/0       (ephemeral port responses)
  Rule 110: Allow TCP 3306 to 10.0.2.0/24            (MySQL to DB subnet)
  Rule *:   DENY all                                  (implicit deny)
```

### Security Groups (SGs)

Security Groups are **stateful** firewalls at the **ENI (instance) level**.

| Property | Detail |
|----------|--------|
| Scope | Elastic Network Interface (ENI) — effectively per-instance |
| Stateful? | **Yes**. If inbound is allowed, outbound response is automatic (and vice versa) |
| Rule evaluation | **All rules evaluated**; if any rule allows, traffic is allowed |
| Default SG | Allows **all outbound**, allows inbound **only from same SG** |
| Custom SG | Allows **all outbound**, **denies all inbound** by default |
| Supports | **Allow rules ONLY** — you cannot create deny rules |
| Reference other SGs | Yes — you can reference another SG as a source/destination |

**SG chaining (key exam pattern):**

```
Web SG:     Allow inbound TCP 443 from 0.0.0.0/0
App SG:     Allow inbound TCP 8080 from Web-SG       ← references Web SG
DB SG:      Allow inbound TCP 3306 from App-SG        ← references App SG
```

This is the **most secure and scalable** approach — no need to track IP addresses. Instances can be added/removed and the rules automatically apply.

### NACLs vs Security Groups — The Critical Comparison

| Feature | NACL | Security Group |
|---------|------|----------------|
| Level | Subnet | ENI / Instance |
| Stateful | **No** (must allow return traffic) | **Yes** (return traffic auto-allowed) |
| Rule type | Allow **and** Deny | Allow **only** |
| Rule processing | Ordered (first match) | All rules evaluated (any allow wins) |
| Default (custom) | Deny all | Deny inbound, allow outbound |
| Apply to | All instances in subnet automatically | Only instances assigned the SG |
| Use case | Broad subnet-level blocking, deny lists | Fine-grained instance-level control |

**Exam mantra:** "Need to **DENY** specific IPs or ranges? → NACL. Need to **ALLOW** specific instance-to-instance traffic? → Security Group."

### VPC Flow Logs

Flow Logs capture **IP traffic metadata** (not packet contents) going to/from network interfaces.

**Three levels:**
1. **VPC Flow Logs** — captures all ENIs in the VPC
2. **Subnet Flow Logs** — captures all ENIs in the subnet
3. **ENI Flow Logs** — captures traffic for a specific network interface

**Log record fields (v2 default):**

```
version account-id interface-id srcaddr dstaddr srcport dstport protocol packets bytes start end action log-status
```

**Key fields:**
- `action` = `ACCEPT` or `REJECT`
- `log-status` = `OK`, `NODATA`, or `SKIPDATA`

**What Flow Logs do NOT capture:**
- DNS traffic to Amazon-provided DNS (but captures traffic to custom DNS)
- DHCP traffic
- Traffic to the instance metadata service (`169.254.169.254`)
- Traffic to the Amazon Time Sync Service (`169.254.169.123`)
- Amazon Windows license activation traffic
- Traffic between an endpoint network interface and a Network Load Balancer network interface

**Destinations:**
| Destination | Use Case |
|------------|----------|
| **CloudWatch Logs** | Real-time metric filters, alarms, dashboards |
| **S3** | Long-term storage, Athena queries, cost-effective archival |
| **Kinesis Data Firehose** | Real-time streaming to Splunk, OpenSearch, third-party tools |

**Enhanced/custom format (v3–v5):** Additional fields available including `vpc-id`, `subnet-id`, `az-id`, `pkt-srcaddr`, `pkt-dstaddr` (original packet IPs before NAT), `traffic-path`, `flow-direction`.

**Exam tip:** Flow Logs are **not real-time** — there is a ~10-minute delay. They capture **metadata only**, not packet payloads. For packet inspection, use **Traffic Mirroring** or a third-party appliance.

### Elastic Network Interface (ENI)

- A **virtual network card** attached to an instance
- Has: private IP(s), public IP, Elastic IP, MAC address, security group(s), source/dest check flag
- **Primary ENI** (`eth0`) cannot be detached; secondary ENIs can be moved between instances (same AZ)
- Used for: dual-homed instances, management networks, low-budget HA (move ENI on failover)

### Key Limits & Quotas to Memorize

| Resource | Default Limit |
|----------|---------------|
| VPCs per region | 5 (adjustable) |
| Subnets per VPC | 200 (adjustable) |
| Route tables per VPC | 200 (adjustable) |
| Routes per route table | 50 (adjustable to 1,000) |
| Security groups per VPC | 2,500 (adjustable) |
| Inbound + outbound rules per SG | 60 each (adjustable) |
| SGs per ENI | 5 (adjustable to 16) |
| NACLs per VPC | 200 |
| Rules per NACL | 20 (adjustable to 40) |
| Elastic IPs per region | 5 (adjustable) |
| IGWs per VPC | 1 |
| NAT Gateways per AZ | 5 (adjustable) |
| VPC Peering connections per VPC | 50 (adjustable to 125) |

---

## 2. Design Patterns & Best Practices

### Multi-Tier VPC Design (Standard Pattern)

```
┌─────────────────────── VPC (10.0.0.0/16) ───────────────────────┐
│                                                                   │
│  AZ-a                          AZ-b                              │
│  ┌──────────────┐              ┌──────────────┐                  │
│  │ Public Subnet│              │ Public Subnet│  ← ALB, NAT GW  │
│  │ 10.0.1.0/24  │              │ 10.0.2.0/24  │                  │
│  └──────────────┘              └──────────────┘                  │
│  ┌──────────────┐              ┌──────────────┐                  │
│  │ Private Subnet│             │ Private Subnet│ ← App servers   │
│  │ 10.0.3.0/24  │              │ 10.0.4.0/24  │                  │
│  └──────────────┘              └──────────────┘                  │
│  ┌──────────────┐              ┌──────────────┐                  │
│  │ Isolated Sub  │             │ Isolated Sub  │ ← Databases     │
│  │ 10.0.5.0/24  │              │ 10.0.6.0/24  │                  │
│  └──────────────┘              └──────────────┘                  │
└───────────────────────────────────────────────────────────────────┘
```

**Best practices:**
- Use **/24** subnets for most workloads (251 usable IPs) — easy math, room to grow
- Deploy resources across **at least 2 AZs**
- Place **ALB** in public subnets, **app servers** in private, **databases** in isolated
- One **NAT Gateway per AZ** for HA (each AZ's private route table → its own NAT GW)
- Use **separate route tables** per subnet tier (public, private, isolated)

### CIDR Planning for Enterprise

| Strategy | Approach |
|----------|----------|
| **Non-overlapping CIDRs** | Mandatory if using VPC Peering, Transit Gateway, or Direct Connect |
| **IPAM (IP Address Manager)** | Use AWS VPC IPAM to centrally plan, track, and allocate CIDRs across accounts/regions |
| **Reserve space** | Use `/16` per VPC but only allocate `/24` subnets — leaves room for growth |
| **Separate ranges per environment** | e.g., `10.1.0.0/16` = prod, `10.2.0.0/16` = staging, `10.3.0.0/16` = dev |

### When to Use NACLs vs. Security Groups

| Scenario | Use |
|----------|-----|
| Block a specific malicious IP | **NACL** (deny rule) |
| Allow app tier to talk to DB tier | **Security Group** (SG chaining) |
| Compliance: explicit subnet-level deny logs | **NACL** |
| Dynamic workloads (containers, auto-scaling) | **Security Group** (attached to ENI, follows instance) |
| Defense in depth | **Both** — NACLs as coarse outer layer, SGs as fine inner layer |

### Anti-Patterns

- **Single large subnet** — defeats purpose of isolation and makes NACLs useless
- **All resources in public subnets** — exposes databases and internal services to the internet
- **Overlapping CIDRs** — blocks VPC peering, Direct Connect, Transit Gateway
- **Security Groups with `0.0.0.0/0` on all ports** — negates all security benefits
- **Using only NACLs** — too coarse; ephemeral port management is error-prone without SGs
- **Single NAT Gateway for multi-AZ** — creates cross-AZ dependency and single point of failure

### Integration Points

| Service | VPC Integration |
|---------|----------------|
| **Transit Gateway** | Hub-and-spoke: connect hundreds of VPCs + on-prem via single gateway |
| **VPC Peering** | Point-to-point, non-transitive connection between 2 VPCs (same or cross-region) |
| **PrivateLink / Interface Endpoints** | Access AWS services privately via ENI in your subnet (no IGW needed) |
| **Gateway Endpoints** | Free endpoints for **S3 and DynamoDB only** — route table entry, no ENI |
| **Direct Connect** | Dedicated physical connection from on-prem to AWS VPC via VGW or TGW |
| **AWS Network Firewall** | Managed stateful/stateless firewall deployed in its own subnet, inspects traffic via route table |
| **ALB / NLB** | ALBs need public subnets (minimum 2 AZs); NLBs can have static IPs, support PrivateLink |
| **Lambda** | Can be placed in VPC subnets to access private resources (uses Hyperplane ENIs) |
| **EKS / ECS** | Pods/tasks launched in VPC subnets; use SGs for pod-level security |

---

## 3. Security & Compliance

### Defense-in-Depth Model

```
Internet → WAF → ALB (SG) → NACL → App Instance (SG) → NACL → DB Instance (SG)
                                 ↑                            ↑
                           Subnet boundary              Subnet boundary
```

**Layers:**
1. **AWS WAF** on ALB/CloudFront — Layer 7 filtering (SQL injection, XSS, geo-blocking, rate limiting)
2. **NACLs** — Layer 3/4 subnet boundary (IP/port deny lists)
3. **Security Groups** — Layer 3/4 instance boundary (allow lists, SG chaining)
4. **OS-level firewall** (iptables, Windows Firewall) — host-level hardening
5. **AWS Network Firewall** — managed IDS/IPS, deep packet inspection, domain filtering

### IAM & Resource Policies

- **VPC Endpoint Policies** — restrict which principals/actions/resources can be accessed through a VPC endpoint
  ```json
  {
    "Statement": [{
      "Principal": "*",
      "Action": "s3:GetObject",
      "Effect": "Allow",
      "Resource": "arn:aws:s3:::my-secure-bucket/*"
    }]
  }
  ```
- **S3 Bucket Policy** — can restrict access to a specific VPC or VPC Endpoint using `aws:sourceVpce` or `aws:sourceVpc` condition keys
- **SCPs** — prevent accounts from creating VPCs in unapproved regions, deleting flow logs, or modifying network ACLs

### Encryption

| What | How |
|------|-----|
| Data in transit within VPC | **TLS** between services; traffic between instances in same VPC is **not encrypted by default** |
| Data in transit between VPCs | VPC Peering / TGW traffic is encrypted if cross-region; **same-region is on AWS private network but not encrypted at network layer** |
| Flow Logs at rest | Encrypted in S3 (SSE-S3 or SSE-KMS) or CloudWatch Logs (KMS) |
| VPN | IPsec encryption (AES-256) |
| Direct Connect | **Not encrypted by default** — use VPN over DX or MACsec for encryption |

### Logging & Monitoring

| Tool | What It Captures |
|------|-----------------|
| **VPC Flow Logs** | Network traffic metadata (src/dst IP, port, protocol, accept/reject) |
| **CloudTrail** | API calls (CreateVpc, AuthorizeSecurityGroupIngress, etc.) |
| **AWS Config** | Configuration changes (SG rule modifications, NACL changes) with compliance rules |
| **Traffic Mirroring** | Full packet capture (Layer 3) — send to IDS/IPS appliance |
| **Network Firewall Logs** | Alert and flow logs for stateful rule matches |
| **GuardDuty** | Analyzes Flow Logs + DNS logs + CloudTrail for threat detection (no setup needed) |

**Exam-critical Config rules:**
- `vpc-sg-open-to-public` — flags SGs with `0.0.0.0/0` on sensitive ports
- `vpc-flow-logs-enabled` — ensures Flow Logs are active on all VPCs
- `restricted-ssh` — flags SGs allowing SSH from `0.0.0.0/0`

### Compliance Patterns

| Requirement | Solution |
|-------------|----------|
| PCI-DSS: isolate cardholder data | Separate VPC/subnets with strict NACLs + SG chaining + Flow Logs |
| HIPAA: audit all network access | VPC Flow Logs to S3 (encrypted with KMS) + Athena for querying |
| SOC 2: change management | AWS Config rules + CloudTrail for all VPC API changes |
| Data residency | VPC in specific region + SCP preventing resource creation in other regions |

---

## 4. Cost Optimization

### What's Free

- VPC creation, subnets, route tables, NACLs, security groups, internet gateways
- VPC peering (no charge for the peering connection itself)
- Gateway VPC endpoints (S3, DynamoDB)
- VPC Flow Logs (the feature itself — but you pay for log storage/delivery)

### What Costs Money

| Resource | Cost Model |
|----------|-----------|
| **NAT Gateway** | ~$0.045/hr + $0.045/GB processed (most expensive VPC component) |
| **Interface VPC Endpoints (PrivateLink)** | ~$0.01/hr/AZ + $0.01/GB processed |
| **VPN Connection** | ~$0.05/hr per connection |
| **Elastic IP (unattached)** | ~$0.005/hr when NOT associated with a running instance |
| **Cross-AZ data transfer** | $0.01/GB each way |
| **Cross-Region data transfer** | $0.02/GB (varies by region) |
| **Flow Logs storage** | CloudWatch Logs ingestion: ~$0.50/GB; S3: ~$0.023/GB |
| **Traffic Mirroring** | ~$0.015/hr per ENI source |

### Cost Traps & Optimization

| Trap | Fix |
|------|-----|
| NAT Gateway data processing fees exploding | Use **Gateway VPC endpoints for S3/DynamoDB** (free) — huge savings if workloads pull from S3 |
| Interface endpoints in every AZ | Only deploy in AZs where workloads actually run |
| Cross-AZ traffic from NAT GW | Deploy NAT GW in each AZ to avoid cross-AZ charges |
| Flow Logs to CloudWatch (expensive at scale) | Send to **S3** for bulk storage, query with **Athena** |
| Unused Elastic IPs | AWS now charges for **all** public IPv4 addresses ($0.005/hr) since Feb 2024 — audit and release unused ones |
| VPN when Direct Connect exists | Consolidate to DX; use VPN only as backup |

**Exam tip:** The question "most cost-effective way to access S3 from a private subnet" → **Gateway VPC Endpoint** (free, no data processing charge) — NOT NAT Gateway.

---

## 5. High Availability, Disaster Recovery & Resilience

### Multi-AZ VPC Design (Standard HA)

```
Region: us-east-1
├── AZ: us-east-1a
│   ├── Public Subnet  → NAT GW-a, ALB node
│   ├── Private Subnet → App instances (ASG)
│   └── Isolated Subnet → RDS Primary
├── AZ: us-east-1b
│   ├── Public Subnet  → NAT GW-b, ALB node
│   ├── Private Subnet → App instances (ASG)
│   └── Isolated Subnet → RDS Standby
└── AZ: us-east-1c (optional third AZ for extra resilience)
    ├── Public Subnet  → NAT GW-c, ALB node
    ├── Private Subnet → App instances (ASG)
    └── Isolated Subnet → RDS Read Replica
```

**Key HA principles:**
- **ALB** spans multiple AZs automatically — minimum 2 AZs required
- **NAT Gateway** is AZ-scoped — deploy one per AZ; each AZ's route table points to its own NAT GW
- **Auto Scaling Group** spans subnets across AZs — if one AZ fails, ASG launches instances in surviving AZs
- **RDS Multi-AZ** — synchronous standby in another AZ, automatic failover

### Multi-Region VPC Design

| Pattern | Architecture |
|---------|-------------|
| **Active-Passive** | Primary VPC in Region A; standby VPC in Region B with minimal resources. Route 53 failover routing. |
| **Active-Active** | VPCs in both regions with full workloads. Route 53 latency-based or weighted routing. DynamoDB Global Tables or Aurora Global Database for data. |
| **Pilot Light** | Core infrastructure (DB replicas) always running in DR region; compute scaled up on failure. |
| **Warm Standby** | Scaled-down but fully functional stack in DR region; scale up on failure. |

**Cross-region networking:**
- **VPC Peering (cross-region)** — encrypted, uses AWS backbone, simple but non-transitive
- **Transit Gateway (cross-region peering)** — TGW in each region, peered together; transitive routing
- **CloudWAN** — global network management for complex multi-region topologies

### Resilience Patterns

| Failure Scenario | Mitigation |
|-----------------|------------|
| Single AZ failure | Multi-AZ subnets + ALB + ASG |
| NAT GW failure | One NAT GW per AZ (not shared) |
| Region failure | Multi-region with Route 53 failover |
| DDoS attack | AWS Shield (Standard = free, Advanced = managed), WAF, CloudFront |
| Route table misconfiguration | AWS Config rules, CloudTrail alerts, IaC (CloudFormation/Terraform) |
| SG/NACL lockout | **VPC Reachability Analyzer** to diagnose connectivity; always test changes in non-prod first |

---

## 6. Exam-Focused Section

### Straightforward Questions

**Q1:** A company's security team requires that all SSH access to EC2 instances be blocked from the public internet. Which is the MOST effective way to achieve this?

**A:** Remove any Security Group rule allowing inbound TCP port 22 from `0.0.0.0/0`. Use **Systems Manager Session Manager** for shell access instead — no inbound ports needed.
> SGs are the first line of defense; Session Manager eliminates the need to expose SSH entirely.

---

**Q2:** Which VPC component allows you to explicitly DENY traffic from a specific IP range?

**A:** **Network ACL (NACL)** — the only VPC firewall that supports deny rules. Security Groups only support allow rules.

---

**Q3:** An architect needs to give EC2 instances in a private subnet access to S3 without traversing the internet. What is the most cost-effective solution?

**A:** Create an **S3 Gateway VPC Endpoint** — it's free (no hourly or data processing charges) and routes traffic over the AWS private network.

---

**Q4:** How many IP addresses are available in a `/24` subnet in a VPC?

**A:** **251** — A `/24` = 256 IPs minus 5 reserved by AWS = 251 usable.

---

**Q5:** A solutions architect needs to capture all network traffic metadata for a VPC for security analysis. The data must be queryable and stored cost-effectively for 1 year. What should they use?

**A:** Enable **VPC Flow Logs** and send them to **S3**. Use **Amazon Athena** to query the logs. S3 is cheaper than CloudWatch Logs for long-term storage.

---

**Q6:** An application in a private subnet can reach the internet but external clients cannot initiate connections to it. Which service enables this?

**A:** **NAT Gateway** — allows outbound internet access from private subnets while preventing inbound connections.

---

**Q7:** What happens when you create a new custom NACL and associate it with a subnet?

**A:** All inbound and outbound traffic is **denied by default**. Custom NACLs start with only the default deny-all rule (`*`).

---

### Tricky / Scenario-Based Questions

**Q1:** A company runs a three-tier application. The web tier is in a public subnet, the application tier in a private subnet, and the database in an isolated subnet. Users report intermittent connectivity failures between the app tier and the database. Security groups are correctly configured. What is the MOST LIKELY cause?

**A:** The **NACL** on either the app or database subnet is blocking return traffic on **ephemeral ports** (1024–65535).

> **Why other answers are wrong:**
> - "Security groups are misconfigured" — the question says they're correctly configured
> - "Route table is missing" — within the same VPC, the local route handles intra-VPC traffic automatically
> - "IGW is not attached" — irrelevant for private-to-isolated traffic
>
> **Key phrase:** "intermittent failures" + "SGs correct" → think NACL ephemeral ports (stateless = must explicitly allow return traffic)

---

**Q2:** A company has two VPCs: VPC-A (10.0.0.0/16) peers with VPC-B (10.1.0.0/16), and VPC-B peers with VPC-C (10.2.0.0/16). Instances in VPC-A cannot communicate with instances in VPC-C. What should the architect do?

**A:** Create a **direct VPC Peering connection between VPC-A and VPC-C** (or use **Transit Gateway** for transitive routing). VPC Peering is **non-transitive** — traffic cannot route A→B→C.

> **Why "update route tables in VPC-B" is wrong:** Even with correct routes, VPC Peering does NOT allow transitive routing. This is a hard architectural limitation, not a configuration issue.
>
> **Key phrase:** "VPC-A peers with B, B peers with C, A cannot reach C" → **non-transitive peering**

---

**Q3:** An architect is designing a VPC for a regulated workload. The requirement states: "No EC2 instance should EVER be able to access the public internet, even accidentally." Which combination provides the STRONGEST guarantee?

**A:** Place instances in **isolated subnets** (no route to NAT GW or IGW) AND use an **SCP** to prevent `ec2:CreateRoute` and `ec2:ReplaceRoute` targeting `0.0.0.0/0`. Optionally, use AWS Config rule to detect and auto-remediate.

> **Why "just removing the IGW" is insufficient:** Someone could re-attach it. SCPs prevent the route from being created in the first place.
>
> **Why "NACL deny outbound to 0.0.0.0/0" alone is insufficient:** Someone could modify the NACL.
>
> **Key phrase:** "EVER" + "even accidentally" → need preventive controls (SCP) not just detective/reactive

---

**Q4:** A company needs to migrate 500 microservices to AWS. Each microservice must be network-isolated from every other. The solution must minimize operational overhead. What should the architect recommend?

**A:** Deploy all microservices in a **single VPC** using **Security Groups** for isolation (one SG per microservice). Do NOT create 500 VPCs.

> **Why "one VPC per microservice" is wrong:** Massive operational overhead, CIDR management nightmare, hits VPC limits. Security Groups provide equivalent isolation at L4.
>
> **Key phrase:** "minimize operational overhead" + "network-isolated" → SG-based isolation in shared VPC

---

**Q5:** An organization uses a centralized egress VPC with a NAT Gateway for all outbound internet traffic from 50 spoke VPCs via Transit Gateway. Monthly NAT Gateway data processing costs are $15,000. Most traffic is S3 uploads. How can costs be reduced with MINIMAL architecture changes?

**A:** Add **S3 Gateway VPC Endpoints** in each spoke VPC. S3 traffic will route directly to S3 via the endpoint, bypassing the NAT Gateway entirely. Gateway endpoints are free.

> **Why "use S3 Transfer Acceleration" is wrong:** Doesn't reduce NAT GW costs, adds its own cost
> **Why "switch to NAT instances" is wrong:** Significant operational overhead, doesn't address the root cause
> **Why "use Interface Endpoint for S3" is wrong:** Interface endpoints cost money ($0.01/hr/AZ + data processing). Gateway endpoints are free.
>
> **Key phrase:** "most traffic is S3" + "cost reduction" + "minimal changes" → Gateway VPC Endpoint

---

**Q6:** A security team sees in VPC Flow Logs that traffic from an EC2 instance to a specific IP is showing `ACCEPT` in the Flow Log but the application reports connection timeouts. What could explain this?

**A:** Flow Logs record at the **Security Group and NACL evaluation point**. The `ACCEPT` means AWS network controls allowed it. The issue is likely:
- A **host-based firewall** on the target (OS-level iptables/Windows Firewall)
- The target **application** isn't listening on that port
- A **NACL on the destination subnet** blocking the traffic (Flow Logs on the source ENI only show source-side decisions)

> **Key insight:** Flow Log `ACCEPT` ≠ application-level success. It only means VPC-level security controls (SG + NACL on THAT ENI) allowed the packet.

---

**Q7:** A company wants to log all traffic between two specific EC2 instances for a forensic investigation, including full packet payloads. What should they use?

**A:** **VPC Traffic Mirroring** — it captures actual packet data (not just metadata like Flow Logs). Mirror the source ENI to a target NLB or ENI running a packet analysis tool.

> **Why "VPC Flow Logs" is wrong:** Flow Logs capture **metadata only** (IPs, ports, accept/reject) — NOT packet contents.
>
> **Key phrase:** "full packet payloads" / "packet contents" / "deep packet inspection" → Traffic Mirroring

---

### Common Exam Traps & Pitfalls

| Trap | Reality |
|------|---------|
| "NACLs are stateful" | **Wrong.** NACLs are STATELESS. Security Groups are stateful. This is the #1 tested distinction. |
| "Security Groups can deny traffic" | **Wrong.** SGs only have ALLOW rules. For DENY, use NACLs. |
| "NAT Gateway provides HA across AZs" | **Wrong.** NAT GW is single-AZ. You need one per AZ for HA. |
| "VPC Peering is transitive" | **Wrong.** A↔B and B↔C does NOT mean A↔C. Use Transit Gateway for transitive routing. |
| "Flow Logs capture packet data" | **Wrong.** Flow Logs capture metadata only. Traffic Mirroring captures packets. |
| "Gateway Endpoint works for all AWS services" | **Wrong.** Gateway Endpoints support ONLY S3 and DynamoDB. Everything else needs Interface Endpoints. |
| "Default NACL denies all" | **Wrong.** The DEFAULT NACL allows all. A newly CREATED (custom) NACL denies all. |
| "Default SG allows all inbound" | **Wrong.** Default SG allows inbound only from the SAME SG. Custom SGs deny all inbound. |
| "Larger subnets are always better" | **Wrong.** You lose 5 IPs regardless of size, but overly large subnets waste CIDR space and make isolation harder. |
| "VPC Flow Logs are real-time" | **Wrong.** ~10 minute aggregation delay. Not suitable for real-time threat response. |
| "Direct Connect is encrypted" | **Wrong.** DX is NOT encrypted by default. Use MACsec (Layer 2) or VPN over DX for encryption. |
| "Interface Endpoint = Gateway Endpoint" | **Wrong.** Interface = ENI + hourly cost + data processing. Gateway = route table entry + free. Different mechanisms entirely. |

---

## 7. Cheat Sheet

### Must-Know Facts

```
VPC
├── Regional, spans all AZs
├── CIDR: /16 max, /28 min, 5 CIDRs total
├── 5 IPs reserved per subnet
├── 1 IGW per VPC (HA by design, no bottleneck)
│
├── Subnets (AZ-scoped)
│   ├── Public  = route to IGW
│   ├── Private = route to NAT GW (no IGW)
│   └── Isolated = no internet route at all
│
├── NACLs (STATELESS, subnet-level)
│   ├── Allow + Deny rules
│   ├── Ordered evaluation (lowest # first)
│   ├── Default NACL = allow all
│   ├── Custom NACL = deny all
│   └── MUST allow ephemeral ports (1024-65535)
│
├── Security Groups (STATEFUL, ENI-level)
│   ├── Allow rules ONLY (no deny)
│   ├── All rules evaluated (not ordered)
│   ├── Can reference other SGs (chaining)
│   └── Default SG: all outbound, inbound from same SG only
│
├── Flow Logs
│   ├── VPC / Subnet / ENI level
│   ├── Metadata only (NOT packets)
│   ├── ~10 min delay (not real-time)
│   ├── Destinations: S3, CloudWatch, Kinesis Firehose
│   └── Does NOT capture: DNS to Amazon DNS, DHCP, metadata service
│
├── NAT Gateway
│   ├── Single AZ (one per AZ for HA!)
│   ├── Up to 100 Gbps
│   ├── No Security Group attachment
│   └── Expensive: hourly + per-GB processed
│
└── VPC Endpoints
    ├── Gateway: S3 + DynamoDB only, FREE, route table entry
    └── Interface: all other services, ENI-based, costs money
```

### Key Differentiators

| If the question says... | Think... |
|------------------------|----------|
| "deny specific IP" | NACL |
| "allow between instances/tiers" | Security Group chaining |
| "stateless firewall" | NACL |
| "stateful firewall" | Security Group |
| "packet inspection" / "payload" | Traffic Mirroring (NOT Flow Logs) |
| "network metadata" / "accept/reject logs" | VPC Flow Logs |
| "cost-effective S3 access from private subnet" | S3 Gateway Endpoint (free) |
| "access AWS service privately" (not S3/DDB) | Interface Endpoint (PrivateLink) |
| "transitive routing between VPCs" | Transit Gateway (NOT VPC Peering) |
| "non-transitive, point-to-point" | VPC Peering |
| "no internet access ever" | Isolated subnet + SCP + Config rule |
| "managed firewall / IDS/IPS" | AWS Network Firewall |
| "cross-region VPC connectivity" | TGW peering or VPC peering (cross-region) |
| "minimize data transfer costs" | Same-AZ placement, Gateway Endpoints, avoid cross-AZ NAT |
| "encrypted connection over public internet" | Site-to-Site VPN |
| "dedicated private connection" | Direct Connect (not encrypted by default) |
| "operational overhead" + many VPCs | Transit Gateway (not mesh peering) |
| "ephemeral ports" / "return traffic failing" | NACL misconfiguration |
| "least privilege network access" | SG chaining + isolated subnets + VPC endpoints |

### Decision Flowchart

```
Need to BLOCK an IP?
  └── Yes → NACL deny rule
  └── No  → Continue

Need instance-to-instance access control?
  └── Yes → Security Group (reference other SGs)
  └── No  → Continue

Need to access S3/DynamoDB from private subnet?
  └── Yes → Gateway VPC Endpoint (free!)
  └── No  → Continue

Need to access other AWS services privately?
  └── Yes → Interface VPC Endpoint (PrivateLink)
  └── No  → Continue

Need internet from private subnet?
  └── Yes → NAT Gateway (one per AZ!)
  └── No  → Continue

Need to connect VPCs?
  └── 2-3 VPCs → VPC Peering
  └── Many VPCs or need transitive routing → Transit Gateway
  └── Global multi-region → CloudWAN or TGW inter-region peering

Need to troubleshoot connectivity?
  └── Metadata → VPC Flow Logs
  └── Packet capture → Traffic Mirroring
  └── Path analysis → VPC Reachability Analyzer

Need network-level firewall with IDS/IPS?
  └── AWS Network Firewall
```

---

*Last updated: 2026-04-16 | AWS SAP-C02 Exam Prep*
