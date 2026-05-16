# AWS SAP-C02 Study Notes: Hybrid Architectures (Outposts, Wavelength, Local Zones)

---

## 1. Core Concepts & Theory

### AWS Outposts

**What it is:** Fully managed infrastructure that extends AWS services, APIs, and tools to your on-premises data center or co-location facility. AWS owns, operates, and manages the hardware — you just provide power, cooling, and network connectivity.

**Two form factors:**
| | Outposts Rack | Outposts Server |
|---|---|---|
| **Size** | 42U full rack (80" tall) | 1U or 2U server |
| **Capacity** | Up to 96 racks per site | 1-2 servers per site |
| **Use Case** | Large-scale on-prem workloads | Small/remote locations (retail, branch offices) |
| **Services** | EC2, EBS, S3, RDS, ECS, EKS, EMR, ElastiCache | EC2, EBS only |
| **Networking** | Service Link (encrypted connection back to parent Region) | Same |
| **S3 on Outposts** | Yes (S3 API-compatible, local storage) | No |

**Key characteristics:**
- Outposts is an **extension of a specific AWS Region** — you select a parent Region and AZ
- All control plane operations route back to the Region via **Service Link** (encrypted VPN or Direct Connect)
- Data plane can operate locally (compute, storage) even if connectivity to Region is temporarily lost (limited functionality)
- **Subnet model:** You create Outpost subnets within your VPC — they appear as a "logical extension" of your VPC
- **Local Gateway (LGW):** Routes traffic between Outposts and your on-premises network
- **Hardware maintenance:** AWS proactively monitors and replaces failed components (you provide physical access)
- **Minimum commitment:** 3-year term

**Services available on Outposts Rack:**
- EC2 (select instance types: C5, M5, R5, G4dn, C6i, M6i, R6i, etc.)
- EBS (gp2, io1)
- S3 on Outposts (separate API namespace: `s3-outposts`)
- RDS (MySQL, PostgreSQL)
- ECS, EKS
- ElastiCache
- EMR
- Application Load Balancer
- AWS IoT Greengrass

**Limits & quotas:**
- Up to 96 racks per site
- EBS volumes: limited to local Outpost capacity
- S3 on Outposts: up to 48 TB per Outpost (expandable by adding more capacity)
- No multi-AZ for RDS on Outposts (single-AZ only, use the Region for DR)
- Instance types are a subset of what's available in-Region

---

### AWS Local Zones

**What it is:** An extension of an AWS Region placed in a large population center, providing single-digit millisecond latency to end users in that metro area. Think of it as a "mini AZ" closer to users.

**Key characteristics:**
- Managed entirely by AWS (you don't host hardware)
- Connected back to the parent Region via AWS's private network (high-bandwidth, redundant)
- You **opt in** to a Local Zone, then create subnets in it
- Appears as an additional AZ in your VPC (e.g., `us-east-1-bos-1a` for Boston)
- **Not all services are available** — primarily EC2, EBS, ECS, EKS, ALB, FSx, ElastiCache, RDS (subset)
- **No free tier** in Local Zones
- Internet egress routes through the Local Zone (local internet breakout possible)

**Available services (varies by Local Zone):**
- EC2 (wide range of instance families)
- EBS (gp2, gp3, io1)
- Amazon ECS, EKS
- Application Load Balancer
- Amazon FSx
- Amazon RDS (select engines)
- Amazon ElastiCache
- AWS Direct Connect (at select Local Zones)

**Limits & quotas:**
- Not all instance types available everywhere
- No S3 hosted in Local Zone (use Region S3 or S3 Transfer Acceleration)
- Availability: individual Local Zone, not multi-AZ (unless multiple Local Zones exist in the metro)
- VPC subnets can span into Local Zones, but Route Tables and Security Groups are per-AZ/LZ

---

### AWS Wavelength

**What it is:** AWS infrastructure deployed at the edge of 5G carrier networks (e.g., Verizon, Vodafone, KDDI, SK Telecom, Bell Canada). Provides ultra-low latency to mobile devices connected via 5G.

**Key characteristics:**
- **Wavelength Zone** = AWS compute/storage embedded in a telecom carrier's 5G network
- Traffic from 5G devices never leaves the carrier network to reach Wavelength resources → **single-digit millisecond latency**
- You create **Wavelength subnets** in your VPC
- Instances in Wavelength Zones get **carrier IP addresses** (routable on the carrier network)
- To reach the public internet or AWS Region services, traffic goes through the **carrier gateway**
- Wavelength Zones are tied to a specific carrier in a specific city

**Available services:**
- EC2 (limited instance types: t3, r5, g4dn)
- EBS (gp2)
- ECS
- EKS
- VPC (subnets, route tables, security groups)
- IAM, CloudWatch (control plane from Region)

**Limits & quotas:**
- Very limited instance families
- No S3, RDS, or most managed services directly in Wavelength
- One carrier per Wavelength Zone
- Must opt in per Wavelength Zone
- Applications must be designed to tolerate limited local capacity

---

### Comparison Summary

| Feature | Outposts | Local Zones | Wavelength |
|---------|----------|-------------|------------|
| **Location** | Your data center | AWS-managed, near metro areas | Telecom carrier's 5G edge |
| **Who hosts hardware** | You provide facility; AWS manages HW | AWS entirely | Carrier facility; AWS manages HW |
| **Primary use case** | Data residency, low-latency to on-prem | Low latency to metro end users | Ultra-low latency for 5G mobile |
| **Latency target** | Sub-ms to on-prem workloads | Single-digit ms to metro users | Single-digit ms to 5G devices |
| **Service breadth** | Broad (EC2, EBS, S3, RDS, ECS, EKS) | Moderate (EC2, EBS, ECS, ALB, RDS) | Narrow (EC2, EBS, ECS, EKS) |
| **Networking** | Service Link + Local Gateway | AWS backbone to Region | Carrier gateway |
| **VPC integration** | Outpost subnet in your VPC | Local Zone subnet in your VPC | Wavelength subnet in your VPC |
| **S3** | S3 on Outposts (local) | No (use Region) | No (use Region) |
| **Commitment** | 3-year term | On-demand | On-demand |
| **Data sovereignty** | Yes (data stays on-prem) | No (AWS facility, may be in same country) | No |

---

### AWS Outposts vs. Alternatives

| Need | Solution |
|------|----------|
| Run AWS services on-prem with same APIs | **Outposts** |
| Extend on-prem to cloud (VMware-centric) | **VMware Cloud on AWS** (or Outposts with VMware) |
| Just need connectivity to AWS | **Direct Connect / Site-to-Site VPN** |
| Edge computing for IoT | **Greengrass, Snowball Edge, Outposts Server** |
| Kubernetes on-prem connected to AWS | **EKS Anywhere** (or EKS on Outposts) |
| Run containers on-prem, connect to AWS | **ECS Anywhere** |

---

## 2. Design Patterns & Best Practices

### When to Use Each

**Use Outposts when:**
- Data residency / data sovereignty requirements (data must not leave premises)
- Need full AWS APIs on-premises (same tooling, same CloudFormation templates)
- Low-latency access to on-premises systems (legacy databases, mainframes)
- Migration factory: lift-and-shift while maintaining on-prem footprint during transition
- Local data processing before sending results to the cloud
- Regulated industries (healthcare, finance, government) with strict data locality

**Use Local Zones when:**
- End users need single-digit millisecond latency (real-time gaming, media streaming, live video)
- Workloads need to be closer to a specific metro but don't require on-prem
- Content creation / rendering (VFX studios near talent pools)
- Real-time multiplayer gaming servers
- Ad-tech and financial trading at the metro level

**Use Wavelength when:**
- 5G mobile application with ultra-low latency requirement
- Connected vehicle / autonomous driving data processing
- AR/VR streaming to mobile devices
- Real-time game streaming to phones
- IoT on 5G networks with edge inference (ML at the edge)

### Anti-Patterns

| Don't Use | When |
|-----------|------|
| Outposts | You just need a VPN to AWS — use Direct Connect instead |
| Outposts | For short-term or bursty workloads (3-year commit) |
| Local Zones | For data sovereignty — data is in an AWS facility |
| Local Zones | When Regional latency (10-50ms) is acceptable |
| Wavelength | Users are on WiFi/wired connections (no 5G benefit) |
| Wavelength | You need broad AWS service access |

### Architectural Patterns

**Hybrid with Outposts:**
```
On-Prem DC ←→ [Outposts (compute, storage, DB)] ←Service Link→ [AWS Region (control plane, DR, analytics)]
                     ↕ Local Gateway
            [On-Prem Applications / Users]
```

**Multi-tier with Local Zones:**
```
[End Users in Metro] → [ALB + App Tier in Local Zone] → [Database / Backend in Region]
```
- Place latency-sensitive tiers (web servers, caches) in Local Zone
- Keep stateful services (RDS, S3) in Region for durability
- Use Local Zone for compute, Region for storage/data

**Edge with Wavelength:**
```
[5G Device] → [Carrier 5G Network] → [Wavelength Zone (inference, processing)] → [Region (storage, training)]
```

### Well-Architected Alignment

| Pillar | Guidance |
|--------|----------|
| **Reliability** | Outposts: plan for Service Link disconnection; Local Zones: not multi-AZ by default — pair with Region for HA |
| **Security** | All three inherit IAM, VPC security groups, encryption. Outposts: physical security is your responsibility |
| **Cost** | Outposts: significant upfront commitment; Local Zones & Wavelength: on-demand but limited pricing optimization |
| **Performance** | Match the right extension to the latency requirement |
| **Operational Excellence** | Same APIs/tools as Region — use CloudFormation, CDK across all |
| **Sustainability** | Local Zones/Wavelength reduce data transfer distances |

---

## 3. Security & Compliance

### Outposts Security

- **Physical security:** Customer responsibility (your data center, your locks)
- **Nitro-based:** Runs on AWS Nitro System — same hardware security as in-Region
- **Encryption:** All data on Outposts is encrypted at rest using AWS-managed keys stored in the Region (via Service Link)
- **Service Link:** Encrypted connection; if severed, instances continue running but can't launch new ones or access control plane
- **IAM:** Standard IAM policies apply; Outposts-specific condition keys (`aws:RequestedRegion`, resource ARNs with outpost ID)
- **VPC:** Outpost subnets use the same security groups, NACLs, and VPC policies
- **Compliance:** Outposts helps meet data residency requirements (PCI DSS, HIPAA, GDPR)
- **Audit:** CloudTrail logs API calls; Config rules apply

**Key IAM condition keys:**
```json
{
  "Condition": {
    "StringEquals": {
      "ec2:OutpostArn": "arn:aws:outposts:us-east-1:123456789012:outpost/op-1234567890abcdef"
    }
  }
}
```

### Local Zones & Wavelength Security

- Same security model as Region (IAM, security groups, NACLs, encryption)
- Data in Local Zones resides in an AWS-managed facility
- **Local Zones do NOT satisfy data sovereignty** for most compliance regimes (AWS facility, not your premises)
- Wavelength: carrier provides physical facility security; AWS manages logical security
- All control plane calls route to the parent Region (CloudTrail captures them)

### Encryption

| Layer | Outposts | Local Zones | Wavelength |
|-------|----------|-------------|------------|
| EBS at rest | Yes (keys in Region KMS) | Yes (standard KMS) | Yes (standard KMS) |
| In transit (Service Link) | Encrypted by default | N/A (AWS backbone) | Carrier network → AWS |
| S3 on Outposts | SSE-S3, SSE-C | N/A | N/A |

### Service Link Disconnection (Outposts)

When Service Link is lost:
- **Running instances continue** to run
- **Local EBS operations** continue
- **Cannot:** Launch new instances, create/delete volumes, authenticate new IAM sessions, access KMS for new encryption operations
- **S3 on Outposts:** Continues serving local requests
- **Best practice:** Pre-cache IAM credentials, use instance profiles with long-lived tokens for critical workloads

---

## 4. Cost Optimization

### Outposts Pricing

- **Capacity pricing:** Choose EC2 and EBS capacity configuration; pay a fixed monthly/3-year fee regardless of utilization
- **No per-instance charges** — you pay for the rack capacity, not individual usage
- **S3 on Outposts:** Charged per GB-month stored
- **Data transfer:** No charge for data transfer between Outposts and parent Region over Service Link
- **Networking:** Direct Connect/VPN costs for Service Link connectivity still apply
- **Support:** Enterprise Support recommended (included hardware replacement SLA)
- **Power & cooling:** Customer cost
- **Facility:** Customer cost

**Cost traps:**
- Underutilization: you pay for full rack capacity even if using 20% of instances
- No Spot or Savings Plans for Outposts instances
- S3 on Outposts is more expensive per GB than standard S3
- Must factor in power, cooling, space, and network costs

### Local Zones Pricing

- **On-demand pricing:** Typically 10-20% higher than equivalent Region pricing
- Standard EC2 pricing model (On-Demand, Reserved, Savings Plans available)
- Data transfer: charges between Local Zone and Region (within the same Region, lower than cross-Region)
- **Cost trap:** Not all instance types available → may need to use larger/more expensive types

### Wavelength Pricing

- **On-demand pricing:** Similar to Region pricing (varies by carrier/zone)
- Data transfer: charges for traffic exiting the carrier network to internet/Region
- **Traffic within carrier network:** No data transfer charges between 5G device and Wavelength Zone
- **Cost trap:** Limited instance types may force over-provisioning

### Cost Comparison Decision

| Scenario | Most Cost-Effective |
|----------|-------------------|
| Need AWS on-prem for 3+ years, high utilization | Outposts |
| Short-term low-latency need in a metro | Local Zone (On-Demand) |
| Burst 5G processing | Wavelength (On-Demand) |
| Just need hybrid connectivity | Direct Connect (cheapest) |

---

## 5. High Availability, Disaster Recovery & Resilience

### Outposts HA/DR

- **Single site risk:** An Outpost is a single physical location — no multi-AZ within Outposts
- **HA pattern:** Run primary workloads on Outposts, replicate to Region for DR
- **RDS on Outposts:** Single-AZ only — use RDS in-Region as standby via replication
- **Service Link failure:** Instances keep running but management is degraded → design for autonomous operation during disconnections
- **Multi-rack:** Deploy multiple racks for local redundancy within the same site
- **RPO/RTO:** Depends on replication strategy to Region (async replication = potential data loss)

**DR architectures:**
```
[Outposts - Primary] ──async replication──→ [Region - DR]
   (Active)                                    (Passive / Pilot Light / Warm Standby)
```

### Local Zones HA/DR

- A Local Zone is a **single point of presence** — not inherently HA
- **HA pattern:** Deploy across Local Zone + parent Region AZs
- **Best practice:** Use Local Zone for the latency-sensitive tier, Region AZs for everything else
- Auto Scaling can span Local Zone + Region AZs (but be aware of latency differences)
- ALB can route to targets in both Local Zone and Region

### Wavelength HA/DR

- Single Wavelength Zone per carrier per city
- **HA pattern:** Multi-carrier or multi-city Wavelength deployment
- Failover to Region if Wavelength Zone becomes unavailable (accept higher latency)
- Design applications to be stateless at the edge → state stored in Region

### Resilience Summary

| Component | HA Strategy | DR Strategy |
|-----------|-------------|-------------|
| Outposts | Multi-rack, redundant networking | Replicate to Region |
| Local Zones | Spread across LZ + Region AZs | Failover to Region |
| Wavelength | Multi-carrier / multi-city | Failover to Region |

---

## 6. Exam-Focused Section

### Straightforward Questions (7 examples)

**Q1:** A company requires AWS services in their on-premises data center due to data sovereignty regulations. Which service should they use?
**A:** AWS Outposts. It runs AWS infrastructure on-premises, keeping data local while using native AWS APIs.

**Q2:** A gaming company needs single-digit millisecond latency for players in Los Angeles. They don't have a data center there. What should they use?
**A:** AWS Local Zones. Deploy game servers in the `us-west-2-lax-1` Local Zone for low latency without owning infrastructure.

**Q3:** A mobile app provider wants to deliver ultra-low latency AR experiences to 5G users. What should they use?
**A:** AWS Wavelength. It embeds compute at the 5G carrier edge, keeping traffic on the carrier network.

**Q4:** What is the minimum commitment for AWS Outposts?
**A:** 3-year term.

**Q5:** How does an Outpost communicate with the parent AWS Region?
**A:** Via the Service Link — an encrypted connection (over Direct Connect or internet VPN) that carries control plane traffic.

**Q6:** Can you run S3 locally on AWS Outposts?
**A:** Yes. S3 on Outposts provides local object storage with S3 APIs (`s3-outposts` namespace).

**Q7:** Which AWS service places infrastructure at 5G carrier locations?
**A:** AWS Wavelength.

---

### Tricky / Scenario-Based Questions (7 examples)

**Q1:** A financial services company must process trades with sub-millisecond latency to their on-premises matching engine AND meet regulatory requirements that data never leaves their premises. They also need to run analytics in the cloud. What architecture should they use?

**A:** Deploy **AWS Outposts** in their data center for the trade processing (sub-ms to matching engine, data stays on-prem). Use the **Service Link** to send aggregated/anonymized analytics data to the Region for processing.

*Why other answers are wrong:*
- Local Zones: Data would be in an AWS facility, not on-premises → fails data sovereignty
- Wavelength: For 5G mobile, not on-prem connectivity
- Direct Connect alone: Doesn't provide AWS compute on-prem

**Keywords:** "data never leaves premises" + "AWS services" → Outposts

---

**Q2:** A media company needs to provide real-time video editing for editors in New York City. The editors are on corporate wired networks. Latency to us-east-1 (Virginia) is ~15ms which causes lag. What should they use?

**A:** **Local Zones** in New York (`us-east-1-nyc-1`). Place the video editing compute instances in the Local Zone for single-digit ms latency to NYC users.

*Why other answers are wrong:*
- Wavelength: Users are on wired/WiFi, not 5G — no benefit
- Outposts: Company doesn't need on-prem infrastructure; they need proximity to a city
- CloudFront: CDN is for content delivery, not interactive compute workloads

**Keywords:** "editors in [city]" + "wired/corporate network" + "need lower latency" → Local Zone

---

**Q3:** A company deploys Outposts and loses Service Link connectivity. What continues to work?

**A:** Running EC2 instances continue operating. Local EBS reads/writes continue. S3 on Outposts serves local requests. **However:** cannot launch new instances, create new EBS volumes, or make new IAM authentication calls.

*Why tricky:* Candidates often think everything stops. The data plane continues; the control plane is impacted.

---

**Q4:** An autonomous vehicle company needs to process LIDAR data from 5G-connected vehicles with minimal latency. The processing requires GPU instances. Which solution minimizes latency?

**A:** **AWS Wavelength** with G4dn instances. Traffic stays on the carrier network, providing ultra-low latency for 5G-connected vehicles.

*Why other answers are wrong:*
- Local Zones with GPU: Lower latency than Region but traffic still exits carrier network
- Outposts: Vehicles aren't at a fixed data center
- Edge locations (CloudFront): No compute capability for ML inference

**Keywords:** "5G-connected" + "minimal latency" + "mobile devices/vehicles" → Wavelength

---

**Q5:** A company runs production workloads on Outposts and needs disaster recovery. They have an RPO of 1 hour and RTO of 15 minutes. What DR architecture is most cost-effective?

**A:** **Warm Standby** in the parent Region. Replicate data asynchronously (meets 1-hour RPO), maintain scaled-down but running infrastructure in-Region (meets 15-min RTO).

*Why other answers are wrong:*
- Pilot Light: RTO would likely exceed 15 minutes (need to scale up)
- Active-Active: Meets requirements but NOT "most cost-effective"
- Backup & Restore: RTO would far exceed 15 minutes

**Keywords:** "cost-effective" + RPO 1h + RTO 15min → Warm Standby

---

**Q6:** A retail company wants to run a small local application at 500 branch stores using AWS APIs. Each store has limited space (no room for a full rack). What should they use?

**A:** **AWS Outposts Servers** (1U/2U form factor). Provides EC2 and EBS at small sites without requiring a full 42U rack.

*Why other answers are wrong:*
- Outposts Rack: Requires full rack space — too large for branch stores
- Local Zones: AWS-managed location, not at the customer's branches
- Snow Family: For data transfer/edge compute, not persistent production workloads with AWS APIs

**Keywords:** "limited space" + "branch stores" + "AWS APIs" → Outposts Servers

---

**Q7:** A company uses Local Zones for a latency-sensitive web application. They need the database tier to be highly available with Multi-AZ. Where should they place the database?

**A:** Place the database (RDS Multi-AZ) in the **parent Region**, not in the Local Zone. Local Zones don't support Multi-AZ deployments. The application tier runs in the Local Zone; the data tier is in the Region.

*Why tricky:* Candidates may think all tiers should be in the Local Zone. But Multi-AZ requires multiple AZs, which Local Zones don't provide.

---

### Common Exam Traps & Pitfalls

1. **Outposts ≠ data sovereignty by default** — it enables it, but you must verify no data is replicated to Region unintentionally (e.g., CloudWatch logs flow to Region)

2. **Local Zones ≠ data sovereignty** — they're AWS-managed facilities in metro areas, NOT on your premises

3. **Wavelength requires 5G** — if the question mentions WiFi, wired, or broadband users, Wavelength gives no benefit → Local Zone instead

4. **Outposts Rack vs. Outposts Servers** — don't confuse the form factors. "Small locations" / "limited space" → Servers. "Full production workloads" / "broad services" → Rack

5. **Service Link disconnection** — running instances DON'T stop. This is a common exam trap. Data plane continues; control plane is impacted.

6. **S3 on Outposts is a separate namespace** — `s3-outposts`, not standard S3. Access points are mandatory. Can't use standard S3 features like Cross-Region Replication directly.

7. **Local Zone subnet ≠ AZ subnet** — they look similar in VPC config, but have different service availability and are not considered an AZ for Multi-AZ deployments

8. **Outposts has NO Spot instances** — capacity is dedicated and pre-provisioned

---

## 7. Cheat Sheet

### Must-Know Facts

- **Outposts** = AWS in YOUR data center (data sovereignty, same APIs)
- **Local Zones** = AWS closer to a CITY (single-digit ms to metro users)
- **Wavelength** = AWS at the 5G CARRIER EDGE (ultra-low latency for mobile)
- All three are extensions of your VPC (create subnets in them)
- Outposts requires **3-year commitment** and **Service Link** to Region
- Local Zones and Wavelength are **opt-in** and **on-demand**
- **Service Link down?** Running instances continue; no new launches
- **Outposts Rack** = full 42U rack, broad services (EC2, EBS, S3, RDS, ECS, EKS)
- **Outposts Server** = 1U/2U, limited services (EC2, EBS only), for small/remote sites
- S3 on Outposts uses **access points** (mandatory) and `s3-outposts` endpoint
- No Multi-AZ anything on Outposts or in Local Zones → DR to Region

### Decision Flowchart

```
Question says "data must remain on-premises" or "data sovereignty"
  → AWS Outposts

Question says "low latency to users in [city]" + users on wired/WiFi
  → Local Zones

Question says "5G" or "mobile edge" or "carrier network"
  → Wavelength

Question says "small/remote location" + "limited space" + "AWS APIs"
  → Outposts Servers

Question says "run AWS services on-prem" + "same APIs and tools"
  → Outposts Rack

Question says "hybrid connectivity" only (no on-prem compute needed)
  → Direct Connect / VPN (NOT Outposts)
```

### Key Differentiators

| Signal in Question | Answer |
|---|---|
| "Data cannot leave the country/premises" | Outposts |
| "Same AWS APIs on-premises" | Outposts |
| "5G", "carrier", "mobile edge" | Wavelength |
| "City name" + "end users" + "wired/WiFi" | Local Zones |
| "500 remote branches", "small form factor" | Outposts Servers |
| "Sub-millisecond to on-prem systems" | Outposts |
| "Single-digit ms to metro users" | Local Zones |
| "Ultra-low latency for connected vehicles" | Wavelength |

---

## 8. Additional Deep-Dive

### 10 More Tricky Scenario-Based Questions

**Q1:** A healthcare organization stores PHI (Protected Health Information) and is required by regulation to keep all patient data within their own facilities. They want to modernize using containers and AWS EKS. What should they recommend?

**A:** **EKS on AWS Outposts.** Deploy Outposts in their data center to keep PHI on-premises while leveraging managed EKS. The control plane runs in the Region; worker nodes and data stay on the Outpost.

*Why wrong:*
- EKS Anywhere: Also valid but doesn't have the same integration level (no native EBS, no automatic updates from AWS)
- ECS Anywhere: Containers but not Kubernetes
- Local Zones: Data would be in an AWS facility → fails regulation

**Keywords:** "PHI" + "must stay in their facilities" + "EKS" → EKS on Outposts

---

**Q2:** A game studio in Austin wants to offer low-latency multiplayer gaming. Players are across the US. Currently all servers are in us-east-1. What architecture reduces latency for West Coast and Central US players while remaining cost-effective?

**A:** Deploy game servers in **Local Zones** in Los Angeles and Dallas/Denver. Use Route 53 latency-based routing to direct players to the nearest game servers. Keep backend (matchmaking, player database) in us-east-1 or us-west-2.

*Why wrong:*
- Wavelength: Players are on wired/WiFi, not 5G-specific
- Outposts: Studio doesn't have data centers across the US
- Multiple Regions: More expensive and complex than Local Zones for this use case

---

**Q3:** A manufacturing company has IoT sensors on their factory floor sending telemetry every 100ms. They need sub-10ms processing and the factory is in a rural area with no nearby AWS Region or Local Zone. What should they use?

**A:** **AWS Outposts Server** at the factory (small form factor for a single site). Process IoT data locally with minimal latency, aggregate and send to Region asynchronously.

*Why wrong:*
- Local Zones: Not available in rural areas
- Wavelength: Requires 5G carrier presence (unlikely in rural factory)
- Outposts Rack: Potentially overkill for IoT processing at one factory (unless scale requires it)

**Keywords:** "rural" + "no nearby Region/LZ" + "factory" + "sub-10ms" → Outposts Server

---

**Q4:** A media streaming company wants to deploy a CDN origin closer to users in Mumbai. They currently use CloudFront with an origin in ap-south-1. Some premium users in Mumbai need even lower latency for live streams. Which combination minimizes latency for premium users?

**A:** Deploy encoding/origin servers in the **Mumbai Local Zone** (if available). CloudFront edge locations serve cached content globally, but the Local Zone origin reduces first-byte latency for live (uncacheable) content for premium users in Mumbai.

*Why wrong:*
- Wavelength: Users aren't specifically on 5G mobile
- Outposts: Company doesn't have a data center in Mumbai
- Just CloudFront: Already in use; adding a closer origin reduces origin-fetch latency

---

**Q5:** A company has Outposts and wants to use Amazon S3 on Outposts for local object storage. They need to replicate critical objects back to S3 in the Region for backup. How should they achieve this?

**A:** Use **DataSync** to replicate objects from S3 on Outposts to S3 in the Region. S3 on Outposts does not support S3 Cross-Region Replication (CRR) or Same-Region Replication (SRR) natively.

*Why wrong:*
- S3 Replication (CRR/SRR): Not supported for S3 on Outposts
- AWS Backup: Does not support S3 on Outposts
- Custom Lambda on Outposts: Overly complex; DataSync is purpose-built

**Keywords:** "S3 on Outposts" + "replicate to Region" → DataSync

---

**Q6:** An application running in a Wavelength Zone needs to access an Amazon DynamoDB table. How does this traffic flow?

**A:** Traffic from the Wavelength Zone routes through the **carrier gateway** to the AWS Region where DynamoDB resides. DynamoDB is not available within Wavelength Zones — all AWS managed service calls go to the Region.

*Why tricky:* Candidates might think there's a shortcut or VPC endpoint in Wavelength. There isn't — only EC2, EBS, ECS, and EKS run locally.

---

**Q7:** A financial institution runs trading algorithms on Outposts for co-location with their exchange connectivity. They need exactly the same EC2 instance types they use in us-east-1 (including p4d.24xlarge for ML). Is this possible?

**A:** **No.** Outposts supports a subset of instance types. p4d instances are NOT available on Outposts. They would need to run ML inference on supported GPU instances (g4dn) or process in-Region.

*Why tricky:* Candidates assume all instance types are available on Outposts. Only a subset is supported.

---

**Q8:** A company uses Outposts and has configured subnets. On-premises applications outside the Outpost need to access the EC2 instances running on Outposts. How is this traffic routed?

**A:** Through the **Local Gateway (LGW)**. The Local Gateway provides a target for routes connecting Outpost subnets to the local on-premises network. You configure routes in the VPC route table pointing on-prem CIDRs to the Local Gateway.

*Why wrong:*
- Internet Gateway: For public internet access, not on-prem routing
- NAT Gateway: For outbound internet from private subnets
- Transit Gateway: For inter-VPC routing, not Outpost-to-local-network

**Keywords:** "on-premises network" + "reach Outposts instances" → Local Gateway

---

**Q9:** A company needs to choose between deploying a video processing pipeline on Local Zones vs. a second Region closer to their users. The pipeline processes user uploads and serves results within 2 seconds. What should they choose?

**A:** **Local Zones.** Benefits: simpler architecture (single VPC spanning Local Zone + Region AZs), lower operational overhead (one Region to manage), and sufficient for single-digit ms latency improvement. A second Region adds complexity (cross-region replication, global load balancing, separate IAM/config).

*Why wrong:*
- Second Region: "Least operational overhead" and "simpler" → Local Zone wins
- Wavelength: No 5G requirement mentioned

**Keywords:** "simpler" + "single VPC" + "latency to [city] users" → Local Zone over multi-Region

---

**Q10:** An organization is evaluating EKS Anywhere vs. EKS on Outposts for running containers on-premises. When should they choose EKS on Outposts over EKS Anywhere?

**A:** Choose **EKS on Outposts** when you want:
- AWS-managed infrastructure (hardware, OS, patching handled by AWS)
- Native EBS volumes for persistent storage
- Tight integration with other AWS services on Outposts (RDS, ElastiCache, S3 on Outposts)
- Same billing model as in-Region EKS

Choose **EKS Anywhere** when:
- You don't want to commit to Outposts hardware (use your own servers)
- You have existing on-prem infrastructure
- You want flexibility across bare metal, VMware, or other environments
- Lower cost (no Outpost hardware commitment)

---

### Compare: Outposts vs. Local Zones vs. Wavelength

| Dimension | Outposts | Local Zones | Wavelength |
|-----------|----------|-------------|------------|
| **Use Case** | Data residency, on-prem AWS | Metro-area low latency | 5G mobile edge |
| **Location** | Customer facility | AWS metro facility | Carrier PoP |
| **Instance Breadth** | Moderate (C5, M5, R5, G4dn, etc.) | Broad (most families) | Narrow (t3, r5, g4dn) |
| **Storage** | EBS + S3 on Outposts | EBS only | EBS only |
| **Database** | RDS (single-AZ) | RDS (select engines) | None locally |
| **Pricing** | Fixed capacity (3-yr) | On-Demand/RI/SP | On-Demand |
| **HA** | Single-site | Single-LZ | Single-carrier-zone |
| **VPC** | Subnet (Outpost) | Subnet (LZ AZ) | Subnet (Wavelength) |
| **Gateway** | Local Gateway | Internet Gateway | Carrier Gateway |
| **Exam Trigger** | "on-premises" + "AWS API" | "[city]" + "end users" | "5G" + "mobile" |

### When Does the Exam Expect Outposts vs. Local Zones vs. Wavelength?

**Pick Outposts when the question says:**
- "Data must remain on-premises / in customer's facility"
- "Same AWS APIs and tools on-prem"
- "Data sovereignty / data residency"
- "Low latency to on-premises legacy systems"
- "Regulated industry + local compute"
- "Hybrid but must use native AWS services (not just connectivity)"

**Pick Local Zones when the question says:**
- "Users in [specific city]"
- "Single-digit millisecond latency to end users"
- "Real-time gaming / media / content creation"
- "Closer to population center"
- Users are on **wired or WiFi** connections

**Pick Wavelength when the question says:**
- "5G" (explicit mention)
- "Mobile edge computing"
- "Connected vehicles / IoT over cellular"
- "Carrier network"
- "Ultra-low latency to mobile devices"
- "AR/VR on mobile"

**Pick Direct Connect / VPN (not Outposts) when:**
- Question only needs **connectivity** to AWS, not on-prem compute
- "Consistent network performance to AWS"
- No mention of running AWS services locally

### All "Gotcha" Differences

| Gotcha | Detail |
|--------|--------|
| Outposts needs Service Link | Encrypted backchannel to Region — without it, control plane fails |
| Outposts RDS = single-AZ only | No Multi-AZ on Outposts → replicate to Region for HA |
| S3 on Outposts ≠ standard S3 | Different namespace (`s3-outposts`), requires access points, no replication features |
| Local Zone ≠ AZ for Multi-AZ | RDS Multi-AZ won't span into a Local Zone |
| Wavelength carrier IP ≠ public IP | Instances get carrier IPs routable on the carrier network only |
| Outposts Servers have no S3 | Only Rack form factor supports S3 on Outposts |
| Local Zones cost more | Typically 10-20% premium over Region pricing |
| Wavelength → Region = extra latency | Accessing Region services from Wavelength adds carrier gateway hop |
| EKS on Outposts control plane in Region | Worker nodes on Outpost, masters in Region → Service Link required |
| No Spot on Outposts | Capacity is pre-allocated — no spot market |

### Decision Tree for Choosing the Right Service

```
START: Does the workload require AWS services?
├── No → Use Direct Connect/VPN for connectivity only
└── Yes
    ├── Must data physically stay on customer premises?
    │   ├── Yes → AWS Outposts
    │   │   ├── Large site, broad services needed? → Outposts Rack
    │   │   └── Small/remote site, EC2+EBS sufficient? → Outposts Servers
    │   └── No
    │       ├── Are end users on 5G mobile and need <5ms latency?
    │       │   ├── Yes → AWS Wavelength
    │       │   └── No
    │       │       ├── Are end users in a specific metro needing <10ms?
    │       │       │   ├── Yes → AWS Local Zones
    │       │       │   └── No → Standard Region deployment
    │       │       └── Is Regional latency (10-50ms) acceptable?
    │       │           ├── Yes → Standard Region
    │       │           └── No → Local Zones or multi-Region
    └── Needs "least operational overhead"?
        ├── Yes → Local Zones (no hardware to manage) or Region
        └── No → Evaluate Outposts if on-prem is required
```

---

*End of study notes for Hybrid Architectures (Outposts, Wavelength, Local Zones)*
