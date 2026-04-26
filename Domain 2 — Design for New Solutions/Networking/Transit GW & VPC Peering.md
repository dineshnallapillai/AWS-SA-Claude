# AWS SAP-C02 Study Notes: Transit Gateway & VPC Peering

---

## 1. Core Concepts & Theory

### VPC Peering

A **VPC Peering Connection** is a networking connection between two VPCs that enables routing traffic between them privately using AWS's backbone network.

**Key characteristics:**

| Property | Detail |
|----------|--------|
| Scope | Between 2 VPCs — same account or cross-account, same region or cross-region |
| Transitivity | **Non-transitive** — if A↔B and B↔C, A CANNOT reach C through B |
| Encryption | Cross-region peering is **encrypted** (AES-256). Same-region uses AWS private network. |
| Bandwidth | No single point of bottleneck — uses existing VPC infrastructure |
| Gateway required | **No** — it's a direct logical link, no physical hardware or gateway |
| IP overlap | **CIDRs must NOT overlap** — peering will be rejected if they do |
| DNS resolution | Optional — enable DNS resolution to resolve private hostnames across peered VPCs |

**How it works:**
1. **Requester VPC** sends a peering request
2. **Accepter VPC** accepts the request (can be different account/region)
3. **Both sides** must update their route tables to direct traffic to the peering connection (`pcx-xxx`)
4. **Both sides** must update Security Groups / NACLs to allow the traffic

**Peering connection setup:**

```
VPC-A (10.0.0.0/16)                    VPC-B (10.1.0.0/16)
┌─────────────────┐    pcx-abc123     ┌─────────────────┐
│  Route Table:    │◄────────────────►│  Route Table:    │
│  10.1.0.0/16    │                   │  10.0.0.0/16    │
│  → pcx-abc123   │                   │  → pcx-abc123   │
└─────────────────┘                   └─────────────────┘
```

**Non-transitivity (THE most tested concept):**

```
    VPC-A ←──pcx──► VPC-B ←──pcx──► VPC-C

    A can talk to B  ✓
    B can talk to C  ✓
    A can talk to C  ✗  ← NOT possible through B
```

Even if VPC-B has routes to both A and C, AWS will **drop** packets that try to transit through a peering connection to another peering connection. This is enforced at the infrastructure level — no configuration can override it.

**What else is non-transitive through VPC Peering:**
- Cannot route through a peered VPC to reach its **IGW** (no "borrowing" internet access)
- Cannot route through a peered VPC to reach its **NAT Gateway**
- Cannot route through a peered VPC to reach its **VPN/Direct Connect** gateway
- Cannot route through a peered VPC to reach its **VPC Endpoint**

**Edge-to-edge routing is NOT supported:**

```
On-Prem ──VPN──► VPC-A ──pcx──► VPC-B    ✗  (can't transit from VPN through peering)
Internet ──IGW──► VPC-A ──pcx──► VPC-B    ✗  (can't transit from IGW through peering)
VPC-A ──pcx──► VPC-B ──VPCE──► S3         ✗  (can't use B's VPC endpoint from A)
```

### VPC Peering Limits & Quotas

| Resource | Default Limit |
|----------|---------------|
| Active VPC peering connections per VPC | 50 (adjustable to 125) |
| Outstanding peering requests per account | 25 |
| Expiry for pending request | 7 days |
| Routes per route table | 50 (adjustable to 1,000) — critical for peering at scale |

**Scaling problem:** Full mesh peering of N VPCs requires **N(N-1)/2** connections.
- 5 VPCs = 10 connections
- 10 VPCs = 45 connections
- 50 VPCs = 1,225 connections — **unmanageable**

This is where Transit Gateway comes in.

---

### AWS Transit Gateway (TGW)

A **Transit Gateway** is a regional, highly available network transit hub that connects VPCs, VPN connections, Direct Connect gateways, and other Transit Gateways using a **hub-and-spoke model**.

**Think of it as:** A cloud-based network router that sits at the center of your network topology.

```
              ┌──── VPC-A
              │
              ├──── VPC-B
              │
TGW ─────────├──── VPC-C
              │
              ├──── VPN Connection (on-prem)
              │
              ├──── Direct Connect Gateway
              │
              └──── Peered TGW (another region)
```

**Key characteristics:**

| Property | Detail |
|----------|--------|
| Scope | **Regional** — one TGW per region (but can peer cross-region) |
| Transitivity | **Yes — fully transitive** by default. A→TGW→B→TGW→C all works |
| Bandwidth | Up to **50 Gbps** per VPC attachment (burst), sustained varies |
| Attachments | VPCs, VPN, Direct Connect Gateway, TGW Peering, Connect (SD-WAN) |
| Routing | Its own **TGW Route Tables** — supports static and dynamic (BGP) routes |
| Availability | **Highly available** across all AZs in the region (AWS-managed) |
| Multicast | **Supports multicast** (VPC Peering does NOT) |
| IP overlap | Still requires **non-overlapping CIDRs** for routing to work |

### TGW Core Components

#### 1. Attachments

An attachment connects a network resource to the Transit Gateway.

| Attachment Type | What It Connects | Notes |
|----------------|-----------------|-------|
| **VPC** | A VPC to the TGW | Must specify one subnet per AZ. TGW creates an ENI in each selected subnet. |
| **VPN** | Site-to-Site VPN to TGW | Supports ECMP for aggregated bandwidth across multiple VPN tunnels |
| **Direct Connect Gateway** | DX Gateway to TGW | Connects on-prem via dedicated lines |
| **Peering** | Another TGW (same or different region/account) | Static routes only (no dynamic BGP across TGW peering) |
| **Connect** | SD-WAN/third-party appliances | Uses GRE tunnel + BGP for dynamic routing |

**VPC Attachment detail:**
- You specify **one subnet per AZ** for the TGW to place its ENI
- Best practice: use **dedicated small subnets** (e.g., `/28`) for TGW attachments — don't share with workload subnets
- The TGW ENI consumes an IP in each subnet
- Traffic from any subnet in the AZ can route to the TGW through the ENI — not just the attachment subnet
- **Cross-AZ traffic:** If a VPC attachment only has a subnet in AZ-a, traffic from AZ-b must cross AZs to reach TGW (adds latency and cross-AZ cost). Best practice: attach **one subnet per AZ** you use.

#### 2. Route Tables

TGW has its own routing domain with **TGW Route Tables** — separate from VPC route tables.

| Concept | Detail |
|---------|--------|
| **Default route table** | Created automatically. All attachments associated by default (unless you disable auto-accept). |
| **Custom route tables** | Create multiple for **network segmentation** (e.g., prod vs dev isolation) |
| **Associations** | An attachment is associated with **exactly one** TGW route table (determines which table processes its outbound traffic) |
| **Propagations** | An attachment can propagate its routes to **multiple** TGW route tables (advertises its CIDR to those tables) |
| **Static routes** | Manually added (e.g., `0.0.0.0/0 → firewall VPC attachment`) |
| **Blackhole routes** | Drop traffic to specific CIDRs (used for isolation) |

**Association vs Propagation (critical distinction):**

```
Association = "Which route table do I LOOK UP when sending traffic FROM this attachment?"
Propagation = "Which route tables should I ADVERTISE my routes TO?"
```

Example:
- VPC-A is **associated** with RT-Prod → outbound traffic from VPC-A uses RT-Prod
- VPC-A **propagates** to RT-Prod and RT-Shared → both tables learn VPC-A's CIDR (10.0.0.0/16)

#### 3. TGW Route Table Segmentation Pattern

This is the **most important TGW design pattern** for the exam.

**Scenario:** Prod VPCs should talk to each other and to shared services, but NOT to dev VPCs. Dev VPCs should talk to each other and to shared services, but NOT to prod.

```
TGW Route Tables:
┌─────────────────────────────────────────────────────┐
│ RT-Prod                                              │
│   Associations: VPC-Prod-A, VPC-Prod-B              │
│   Propagations: VPC-Prod-A, VPC-Prod-B, VPC-Shared  │
│   (Can see: Prod VPCs + Shared. Cannot see: Dev)    │
├─────────────────────────────────────────────────────┤
│ RT-Dev                                               │
│   Associations: VPC-Dev-A, VPC-Dev-B                │
│   Propagations: VPC-Dev-A, VPC-Dev-B, VPC-Shared    │
│   (Can see: Dev VPCs + Shared. Cannot see: Prod)    │
├─────────────────────────────────────────────────────┤
│ RT-Shared                                            │
│   Associations: VPC-Shared                          │
│   Propagations: VPC-Prod-A, VPC-Prod-B,            │
│                 VPC-Dev-A, VPC-Dev-B                 │
│   (Can see: All VPCs — hub for shared services)     │
└─────────────────────────────────────────────────────┘
```

**Result:**
- Prod-A ↔ Prod-B: **Yes** (both in RT-Prod, both propagated)
- Dev-A ↔ Dev-B: **Yes** (both in RT-Dev, both propagated)
- Prod-A ↔ VPC-Shared: **Yes** (Shared propagated to RT-Prod)
- Dev-A ↔ VPC-Shared: **Yes** (Shared propagated to RT-Dev)
- Prod-A ↔ Dev-A: **No** (Dev routes not propagated to RT-Prod, and vice versa)

### ECMP (Equal-Cost Multi-Path Routing)

TGW supports **ECMP for VPN connections**, allowing you to aggregate bandwidth.

| Feature | Detail |
|---------|--------|
| Without ECMP | 1 VPN = 2 tunnels, ~1.25 Gbps per tunnel |
| With ECMP | Multiple VPNs per TGW, traffic load-balanced across all tunnels |
| Example | 4 VPN connections = 8 tunnels = up to ~10 Gbps aggregate |
| Requirement | BGP must be used (static VPN does not support ECMP) |
| Where | Only on TGW VPN attachments — VGW-based VPN does NOT support ECMP |

**Exam tip:** If a question asks about increasing VPN bandwidth beyond 1.25 Gbps, and Direct Connect is not an option, the answer is **TGW with ECMP across multiple VPN connections**.

### Transit Gateway Multicast

- TGW is the **only AWS service** that supports IP multicast
- Create a **multicast domain** associated with the TGW
- Register sources and group members by ENI
- Use cases: financial trading (market data feeds), media streaming, cluster communications
- VPC Peering does NOT support multicast

### TGW Connect (SD-WAN Integration)

- Uses **GRE tunnels** for higher bandwidth (up to 20 Gbps per Connect peer)
- Supports **BGP** for dynamic route exchange with SD-WAN appliances
- Use case: integrating third-party SD-WAN/NFV appliances (Cisco, Palo Alto, etc.)

### Key Limits & Quotas

| Resource | Default Limit |
|----------|---------------|
| TGWs per region | 5 |
| Attachments per TGW | 5,000 |
| Routes per TGW route table | 10,000 |
| TGW peering attachments per TGW | 50 |
| Max bandwidth per VPC attachment | 50 Gbps (burst) |
| Max bandwidth per VPN tunnel | 1.25 Gbps |
| Max bandwidth per DX Gateway attachment | Limited by DX link (1/10/100 Gbps) |
| Max bandwidth per Connect peer (GRE) | 20 Gbps |
| TGW route tables per TGW | 20 |
| Prefixes advertised from TGW to on-prem (DX) | 20 (important for DX design!) |

---

## 2. Design Patterns & Best Practices

### Pattern 1: Hub-and-Spoke with Centralized Egress

The most common enterprise pattern. A shared services VPC provides internet egress for all spoke VPCs.

```
                    ┌──── VPC-App-1 (Private)
                    │
Internet ◄── NAT ◄──┤
     ▲       GW     ├──── VPC-App-2 (Private)
     │     (Egress  │
     │      VPC)    ├──── VPC-App-3 (Private)
   IGW       │      │
             TGW ───┤
                    └──── VPC-Shared-Services
                          (DNS, AD, logging)
```

**How it works:**
- Spoke VPCs have `0.0.0.0/0 → TGW` in their route tables
- TGW routes `0.0.0.0/0 → Egress VPC attachment`
- Egress VPC routes through NAT Gateway → IGW
- Egress VPC has a return route: spoke CIDRs → TGW

**Benefits:** Single point of egress control, centralized NAT Gateway, consistent outbound IP for whitelisting, Network Firewall can inspect all egress traffic.

### Pattern 2: Hub-and-Spoke with Centralized Inspection (Firewall)

```
                    ┌──── VPC-A
                    │
         AWS        ├──── VPC-B
       Network      │
       Firewall     ├──── VPC-C
       (Inspection  │
        VPC)        ├──── VPN (on-prem)
             │      │
             TGW ───┘
```

**How it works:**
- All inter-VPC traffic and egress traffic routes through the Inspection VPC
- TGW uses **two route tables:**
  - **Spoke RT:** Default route `0.0.0.0/0 → Inspection VPC` (associated with spoke attachments)
  - **Firewall RT:** Specific VPC CIDRs → respective spoke attachments (associated with Inspection VPC attachment)
- AWS Network Firewall in the Inspection VPC inspects all traffic
- **Appliance mode** must be enabled on the TGW VPC attachment to the Inspection VPC to ensure symmetric routing

**Appliance Mode (critical exam concept):**
- By default, TGW selects the AZ for return traffic based on source ENI hashing
- This can cause **asymmetric routing** through firewalls (request goes through FW in AZ-a, response through AZ-b)
- **Appliance mode** ensures traffic flows through the same AZ for both directions
- **Must be enabled** when using firewalls/appliances behind TGW

### Pattern 3: Isolated VPCs with Shared Services

```
TGW Route Table Design:
┌─────────────────────────────────────────┐
│ RT-Isolated (associated with all VPCs)  │
│   Only propagation: VPC-Shared          │
│   Result: VPCs can reach Shared only,   │
│   NOT each other                         │
├─────────────────────────────────────────┤
│ RT-Shared (associated with VPC-Shared)  │
│   Propagations: ALL VPCs               │
│   Result: Shared can reach any VPC      │
└─────────────────────────────────────────┘
```

Use case: multi-tenant SaaS where tenant VPCs must be fully isolated but all need access to shared DNS, logging, or tooling VPCs.

### Pattern 4: Multi-Region with TGW Peering

```
Region: us-east-1              Region: eu-west-1
┌──────────────┐               ┌──────────────┐
│    TGW-1     │◄── Peering ──►│    TGW-2     │
│  ┌─── VPC-A  │               │  ┌─── VPC-D  │
│  ├─── VPC-B  │               │  ├─── VPC-E  │
│  └─── VPC-C  │               │  └─── VPC-F  │
└──────────────┘               └──────────────┘
```

**Key facts about TGW Peering:**
- Uses AWS backbone (encrypted)
- **Static routes only** — no BGP propagation across TGW peering
- You must manually add routes: `10.2.0.0/16 → TGW Peering Attachment` on each side
- Bandwidth: up to 50 Gbps
- Cross-account and cross-region supported

**Exam tip:** If the question says "dynamic routing between regions" → TGW Peering alone won't work (static only). You'd need BGP over VPN or DX for dynamic cross-region routing.

### Pattern 5: Migration — Gradual VPC Peering to TGW Migration

```
Phase 1: Existing VPC Peering mesh
Phase 2: Deploy TGW, attach new VPCs to TGW
Phase 3: Gradually migrate peered VPCs to TGW (both can coexist)
Phase 4: Remove old peering connections
```

**Coexistence:** VPC Peering and TGW can coexist. Route tables use **longest prefix match**, so you can have:
- `10.1.0.0/16 → pcx-xxx` (peering — more specific, takes priority)
- `10.0.0.0/8 → tgw-xxx` (TGW — less specific, catches everything else)

This enables gradual migration without downtime.

### When to Use VPC Peering vs Transit Gateway

| Criteria | VPC Peering | Transit Gateway |
|----------|-------------|-----------------|
| Number of VPCs | **2-5** (manageable mesh) | **5+** (hub-and-spoke scales better) |
| Transitive routing needed | **No** — add direct peering | **Yes** — TGW is transitive |
| Cost sensitivity | **Free** (no hourly charge, only data transfer) | **$0.05/hr per attachment + $0.02/GB** |
| Bandwidth | No bottleneck (AWS infra) | 50 Gbps burst per VPC attachment |
| Cross-region | Supported (encrypted) | Supported (TGW Peering, static routes) |
| Multicast | **Not supported** | **Supported** |
| Centralized egress/firewall | Hard to architect | **Native pattern** |
| Network segmentation | Manual with route tables | **Built-in with TGW route tables** |
| VPN ECMP | **Not supported** | **Supported** |
| On-prem connectivity sharing | **Not possible** (no edge-to-edge routing) | **Built-in** — VPN/DX attached to TGW, all VPCs can reach on-prem |
| Operational complexity | Low (few VPCs), High (many VPCs) | Moderate (centralized management) |

### Anti-Patterns

| Anti-Pattern | Why It's Wrong | Correct Approach |
|-------------|----------------|------------------|
| Full-mesh VPC Peering at scale (50+ VPCs) | O(n^2) connections, route table explosion | Transit Gateway |
| Using TGW for 2 VPCs with no growth plan | Unnecessary cost ($0.05/hr per attachment) | VPC Peering (free) |
| Single TGW route table for all attachments | No isolation between environments | Multiple TGW route tables (segmentation) |
| TGW without Appliance Mode for firewalls | Asymmetric routing breaks stateful inspection | Enable Appliance Mode |
| Expecting BGP over TGW Peering | Only static routes supported across TGW peering | Use static routes or DX/VPN for dynamic |
| Sharing on-prem VPN through VPC Peering | Edge-to-edge routing not supported | Attach VPN to TGW |
| Overlapping CIDRs with TGW | Routing ambiguity, packets dropped | Plan CIDRs with AWS VPC IPAM |

### Well-Architected Framework Alignment

| Pillar | VPC Peering | Transit Gateway |
|--------|-------------|-----------------|
| **Reliability** | Redundant (AWS-managed, no SPOF) | HA across all AZs, auto-failover |
| **Security** | SGs + NACLs, cross-region encrypted | Same + TGW route table segmentation, Network Firewall integration |
| **Cost** | Free (data transfer only) | Hourly per attachment + data processing |
| **Performance** | No bandwidth bottleneck | 50 Gbps burst; ECMP for VPN scaling |
| **Operational Excellence** | Simple at small scale, painful at large | Centralized management, IaC-friendly |
| **Sustainability** | Minimal resources | Shared infrastructure reduces duplication |

---

## 3. Security & Compliance

### Network Segmentation with TGW

**TGW route tables** are the primary mechanism for enforcing network boundaries.

| Technique | How |
|-----------|-----|
| **Isolated environments** | Separate TGW route tables per environment; only propagate shared-services routes |
| **Blackhole routes** | Add `10.99.0.0/16 → blackhole` to explicitly drop traffic to restricted CIDRs |
| **No default route propagation** | Disable auto-propagation; manually control which VPCs can see each other |
| **Separate TGWs** | For maximum isolation (e.g., classified vs. unclassified), use different TGWs entirely |

### Cross-Account Sharing

**TGW can be shared across accounts using AWS Resource Access Manager (RAM).**

| Step | Detail |
|------|--------|
| 1. Share TGW via RAM | Owner account shares TGW with specific accounts or entire Organization |
| 2. Accept share | Target account accepts the RAM share (auto-accept if same Organization) |
| 3. Create attachment | Target account creates a VPC attachment to the shared TGW |
| 4. Route table management | Only the **TGW owner** can manage route tables, associations, and propagations |

**Security implication:** The TGW owner controls all routing. Spoke account owners can create/delete their own attachments but cannot see or modify routes in other accounts' attachments.

### VPC Peering Cross-Account Security

| Control | Detail |
|---------|--------|
| Peering request | Must be explicitly accepted by the other account |
| Route tables | Each account manages its own — must add routes to allow traffic |
| Security Groups | Can reference SGs from peered VPC (same region only) using the SG ID |
| NACLs | Still apply — acts as subnet-level control even for peered traffic |
| DNS | Must explicitly enable DNS resolution across peering connection |

### IAM Policies & SCPs

**Key API actions to control:**

```
ec2:CreateTransitGateway
ec2:CreateTransitGatewayVpcAttachment
ec2:CreateTransitGatewayRouteTable
ec2:CreateTransitGatewayRoute
ec2:CreateVpcPeeringConnection
ec2:AcceptVpcPeeringConnection
ec2:AcceptTransitGatewayVpcAttachment
```

**SCP example — prevent VPC Peering (force all networking through TGW):**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Deny",
    "Action": [
      "ec2:CreateVpcPeeringConnection",
      "ec2:AcceptVpcPeeringConnection"
    ],
    "Resource": "*"
  }]
}
```

**SCP example — restrict TGW attachments to approved TGWs only:**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Deny",
    "Action": "ec2:CreateTransitGatewayVpcAttachment",
    "Resource": "arn:aws:ec2:*:*:transit-gateway/*",
    "Condition": {
      "StringNotEquals": {
        "ec2:TransitGatewayId": "tgw-0abc123approved"
      }
    }
  }]
}
```

### Logging & Monitoring

| Tool | What It Captures |
|------|-----------------|
| **VPC Flow Logs** | Traffic at ENI/subnet/VPC level — includes traffic entering/leaving TGW ENIs |
| **CloudTrail** | All API calls: CreateTransitGateway, CreateVpcPeeringConnection, route changes |
| **AWS Config** | Configuration drift: unauthorized route changes, unapproved peering connections |
| **TGW Network Manager** | Global view of your network topology, monitors SD-WAN integration |
| **CloudWatch Metrics** | TGW metrics: `BytesIn`, `BytesOut`, `PacketsIn`, `PacketsOut`, `PacketDropCountBlackhole` per attachment |
| **TGW Flow Logs** | Available since 2023 — capture traffic metadata at the TGW level (like VPC Flow Logs but for TGW) |

**Key CloudWatch alarms for TGW:**
- `PacketDropCountBlackhole` > 0 → traffic hitting blackhole routes (misconfiguration or expected isolation)
- `BytesOut` per attachment → monitor bandwidth usage, detect anomalies

### Encryption

| Path | Encrypted? |
|------|-----------|
| VPC Peering (same region) | AWS private network, not encrypted at network layer |
| VPC Peering (cross-region) | **Yes** — AES-256, automatic |
| TGW within region | AWS private network, not encrypted at network layer |
| TGW Peering (cross-region) | **Yes** — automatic |
| VPN over TGW | **Yes** — IPsec (AES-256) |
| Direct Connect over TGW | **Not encrypted** — use MACsec or VPN over DX |

---

## 4. Cost Optimization

### VPC Peering Costs

| Component | Cost |
|-----------|------|
| Peering connection | **Free** |
| Data transfer (same AZ) | **Free** |
| Data transfer (cross-AZ) | **$0.01/GB** each direction |
| Data transfer (cross-region) | **$0.02/GB** (varies by region pair) |

**VPC Peering is the cheapest VPC-to-VPC connectivity option.** There are zero hourly charges.

### Transit Gateway Costs

| Component | Cost (us-east-1) |
|-----------|------------------|
| Per VPC/VPN attachment | **$0.05/hour** (~$36/month) |
| Data processing | **$0.02/GB** |
| TGW Peering attachment | **$0.05/hour** |
| TGW Peering data transfer | **$0.02/GB** |

**Cost example:** 20 VPCs attached to TGW, 10 TB/month inter-VPC traffic:
- Attachments: 20 x $0.05/hr x 730 hrs = **$730/month**
- Data processing: 10,000 GB x $0.02 = **$200/month**
- **Total: $930/month**

Same setup with VPC Peering (full mesh, same AZ):
- 20 VPCs = 190 peering connections
- Attachment cost: **$0/month**
- Data: Free (same AZ) to $0.01/GB (cross-AZ)
- But **190 connections** to manage = high operational cost

### Cost Optimization Strategies

| Strategy | Savings |
|----------|---------|
| **Use VPC Peering for high-traffic pairs** | If 2 VPCs exchange massive data, a direct peering bypasses TGW data processing ($0.02/GB) |
| **Combine TGW + Peering** | Use TGW for general connectivity, direct peering for high-bandwidth pairs (longest prefix match handles routing) |
| **Minimize cross-AZ traffic** | Ensure TGW attachment subnets exist in every AZ your workloads use |
| **Consolidate VPCs** | Fewer VPCs = fewer TGW attachments = lower hourly cost |
| **Use VPC Endpoints** | Route S3/DynamoDB traffic through Gateway Endpoints, NOT through TGW to a centralized NAT |
| **Right-size VPN connections** | Don't over-provision VPN tunnels if ECMP isn't fully utilized |
| **Monitor PacketDropCountBlackhole** | Dropped packets = wasted processing charges |

### Cost Comparison Table

| Scenario | Best Option | Why |
|----------|-------------|-----|
| 2 VPCs, high traffic | VPC Peering | Free connectivity, no processing fee |
| 2 VPCs, need on-prem transit | TGW | Peering can't transit to VPN/DX |
| 10+ VPCs | TGW | Management overhead of peering mesh is too high |
| Need multicast | TGW | Only option |
| Budget-constrained, few VPCs | VPC Peering | Zero fixed cost |
| Centralized firewall/egress | TGW | Architecture requires hub-and-spoke |

---

## 5. High Availability, Disaster Recovery & Resilience

### VPC Peering HA

- AWS-managed, **no single point of failure**
- No SLA on peering itself, but rides on AWS backbone
- **Cross-region peering** provides DR connectivity between regions
- If one VPC fails, peering connections to other VPCs are unaffected

### Transit Gateway HA

- **Highly available by design** — AWS deploys across all AZs in the region
- No maintenance windows, no patching, no manual failover
- If an AZ fails, TGW continues routing through remaining AZs
- **VPC attachment subnets in multiple AZs** ensure no single-AZ dependency

### Multi-Region DR Architecture

**Active-Passive with TGW:**

```
Region A (Active)                    Region B (Passive)
┌─────────────────┐                 ┌─────────────────┐
│      TGW-A      │◄── Peering ──► │      TGW-B      │
│  ┌── VPC-App    │                 │  ┌── VPC-App    │
│  ├── VPC-Data   │                 │  ├── VPC-Data   │
│  └── VPC-Shared │                 │  └── VPC-Shared │
│  ┌── VPN/DX     │                 │  ┌── VPN/DX     │
│  └── (On-prem)  │                 │  └── (On-prem)  │
└─────────────────┘                 └─────────────────┘
         │                                   │
    Route 53 ◄─── Failover Routing ──►  Route 53
```

**Failover considerations:**
- TGW Peering = **static routes** → must be pre-configured, but no dynamic failover
- DNS-based failover (Route 53) switches traffic to DR region
- DX/VPN can connect on-prem to both regions for redundancy
- Data replication (Aurora Global DB, S3 CRR, DynamoDB Global Tables) must be in place

### Resilience Patterns

| Failure | VPC Peering | Transit Gateway |
|---------|-------------|-----------------|
| Single AZ | No impact (regional) | No impact (multi-AZ, with proper subnet attachments) |
| Single VPC | Other peerings unaffected | Other attachments unaffected |
| Regional | Cross-region peering to DR | TGW Peering to DR region |
| On-prem link down | N/A (can't attach VPN to peering) | Failover VPN tunnel (or DX backup) |
| Routing misconfiguration | Limited blast radius (2 VPCs) | Potentially wide blast radius (all attached VPCs) — use IaC! |

**Exam tip:** TGW's wider blast radius means you must use **Infrastructure as Code** (CloudFormation/Terraform) and **AWS Config rules** to prevent and detect misconfigurations.

---

## 6. Exam-Focused Section

### Straightforward Questions

**Q1:** A company has 30 VPCs that all need to communicate with each other and with an on-premises data center connected via Direct Connect. What is the MOST operationally efficient solution?

**A:** **AWS Transit Gateway.** Attach all 30 VPCs and the Direct Connect Gateway to a single TGW. This provides transitive routing in a hub-and-spoke model without managing 435 peering connections.
> 30 VPCs in full mesh = 435 connections. TGW = 30 attachments + 1 DX GW attachment = 31 total.

---

**Q2:** Two VPCs in the same region need to communicate. There is no plan to add more VPCs. What is the most cost-effective connectivity option?

**A:** **VPC Peering.** It's free (no hourly charge), simple to set up for 2 VPCs, and has no data processing fee (only standard data transfer).

---

**Q3:** A company needs to ensure that traffic between VPC-A in us-east-1 and VPC-B in eu-west-1 is encrypted. They currently use VPC Peering. Does this meet the requirement?

**A:** **Yes.** Cross-region VPC Peering traffic is **automatically encrypted** using AES-256 over the AWS global backbone.

---

**Q4:** An architect wants to use a centralized AWS Network Firewall to inspect all inter-VPC traffic. What AWS service should act as the network hub?

**A:** **Transit Gateway** with an Inspection VPC. All traffic routes through TGW → Inspection VPC (Network Firewall) → TGW → destination VPC. **Appliance mode** must be enabled.

---

**Q5:** What is the maximum number of VPC peering connections per VPC?

**A:** **50** by default, adjustable to **125**. Each connection also requires a route table entry, and route tables have a default limit of 50 routes (adjustable to 1,000).

---

**Q6:** A company needs IP multicast support for a financial trading application in AWS. Which networking service supports this?

**A:** **Transit Gateway Multicast.** VPC Peering, VPN, and Direct Connect do NOT support multicast.

---

**Q7:** How can an organization share a Transit Gateway across multiple AWS accounts in an Organization?

**A:** Use **AWS Resource Access Manager (RAM)** to share the TGW with specific accounts or the entire Organization. Spoke accounts create VPC attachments; the TGW owner manages routing.

---

### Tricky / Scenario-Based Questions

**Q1:** VPC-A (10.0.0.0/16) is peered with VPC-B (10.1.0.0/16). VPC-B has a NAT Gateway for internet access. The architect wants instances in VPC-A's private subnet to use VPC-B's NAT Gateway for internet access. Is this possible?

**A:** **No.** This is **edge-to-edge routing**, which is NOT supported through VPC Peering. VPC-A cannot route through VPC-B to reach B's NAT Gateway, IGW, VPN, or VPC Endpoints.

> **Why "add a route 0.0.0.0/0 → pcx-xxx in VPC-A" won't work:** Even if you add the route, AWS infrastructure will drop the packets at the peering boundary because edge-to-edge routing is explicitly blocked.
>
> **Correct solution:** Either give VPC-A its own NAT Gateway, or use **Transit Gateway** (which DOES support transitive routing and centralized egress).
>
> **Key phrase:** "use another VPC's NAT/IGW/VPN/endpoint" through peering → **always impossible**

---

**Q2:** A company has a Transit Gateway with 50 VPCs attached. They notice that VPC-Prod cannot communicate with VPC-Dev, even though both are attached to the TGW. All other VPCs can communicate normally. What is the MOST LIKELY cause?

**A:** The VPCs are associated with **different TGW route tables** that do not have each other's routes propagated. This is the TGW **network segmentation pattern** — Prod and Dev are intentionally (or accidentally) isolated via separate TGW route tables.

> **Why "Security Groups" is unlikely:** SGs would block specific ports, not all connectivity
> **Why "NACLs" is unlikely:** Would affect all traffic to/from the subnet, not just TGW traffic
> **Why "VPC CIDR overlap" is unlikely:** The question says other VPCs work fine — overlap would cause issues during attachment creation
>
> **Key phrase:** "specific VPCs can't communicate through TGW" + "others work fine" → **TGW route table segmentation issue**

---

**Q3:** A company wants to increase VPN throughput to their on-premises data center beyond 1.25 Gbps. They currently have a VPN connection to a Virtual Private Gateway (VGW). What should the architect recommend?

**A:** Migrate the VPN from the **VGW to a Transit Gateway** and enable **ECMP (Equal-Cost Multi-Path)**. Create multiple VPN connections to the TGW. Traffic will be load-balanced across all tunnels.

> **Why "add more tunnels to the existing VPN" doesn't work:** A single VPN connection has exactly 2 tunnels (active/passive or active/active), max ~1.25 Gbps per tunnel. You can't add more tunnels to the same connection.
> **Why "use VGW with multiple VPNs" doesn't work:** VGW does NOT support ECMP — it uses only one tunnel per VPN actively.
> **Why "VPC Peering" doesn't work:** Peering connects VPCs, not on-prem.
>
> **Key phrase:** "increase VPN bandwidth beyond 1.25 Gbps" → **TGW + ECMP**

---

**Q4:** A solutions architect is designing a multi-region active-active architecture. They set up TGW Peering between TGW in us-east-1 and TGW in eu-west-1. They expect routes from us-east-1 VPCs to automatically appear in the eu-west-1 TGW route table via BGP propagation. The routes are not appearing. Why?

**A:** **TGW Peering only supports static routes.** BGP route propagation does NOT work across TGW Peering connections. The architect must manually add static routes pointing to the peering attachment on both TGWs.

> **Why this is tricky:** TGW supports BGP for VPN and Connect attachments within a region. Students often assume BGP works for all TGW attachment types. Peering is the exception.
>
> **Key phrase:** "routes not propagating across TGW peering" / "dynamic routing between TGWs" → **static routes only across TGW peering**

---

**Q5:** A company runs a centralized inspection VPC with a third-party firewall appliance behind a Transit Gateway. Users report that some connections are being dropped by the firewall, which logs show as receiving only one direction of traffic (requests but no responses, or vice versa). What is the MOST LIKELY cause?

**A:** **Appliance mode is not enabled** on the TGW VPC attachment for the Inspection VPC. Without Appliance mode, TGW may route request and response traffic through different AZs, causing **asymmetric routing** that breaks stateful firewall inspection.

> **Why "firewall misconfiguration" is less likely:** The question specifies that the firewall sees traffic in only one direction — this is a classic asymmetric routing symptom, not a firewall rule issue.
> **Why "NACL blocking return traffic" is wrong:** NACLs would block consistently, not intermittently with one-directional traffic patterns.
>
> **Key phrase:** "firewall sees only one direction" / "asymmetric" / "inspection VPC" → **enable Appliance Mode**

---

**Q6:** A company has VPC-A peered with VPC-B (same region). An instance in VPC-A needs to reference a Security Group in VPC-B as a source in its SG rules. Is this possible?

**A:** **Yes** — for **same-region** VPC Peering, you can reference Security Groups from the peered VPC using the format `<account-id>/<sg-id>`. This is NOT possible for **cross-region** peering (you must use CIDR blocks instead).

> **Key phrase:** Same-region peering → SG reference works. Cross-region peering → use CIDR only.

---

**Q7:** A company operates 100 spoke VPCs connected to a Transit Gateway. They want to ensure that a compromised VPC cannot be used to attack other spoke VPCs, while still allowing all VPCs to access a shared-services VPC. What is the MOST secure and operationally efficient approach?

**A:** Create **two TGW route tables**:
1. **Spoke RT** (associated with all spoke VPCs): Only propagate the shared-services VPC routes. Add a blackhole route for the overall VPC supernet to drop any inter-spoke traffic.
2. **Shared RT** (associated with shared-services VPC): Propagate all spoke VPC routes.

Result: Spokes → Shared (allowed), Spokes → Spokes (blocked), Shared → Spokes (allowed).

> **Why "100 separate TGW route tables" is wrong:** Massive operational overhead — the 2-table pattern achieves the same isolation.
> **Why "NACLs on every subnet" is wrong:** Doesn't scale, error-prone, and doesn't leverage TGW's built-in segmentation.
>
> **Key phrase:** "isolated spokes" + "shared services" + "operationally efficient" → **TGW route table segmentation**

---

### Common Exam Traps & Pitfalls

| Trap | Reality |
|------|---------|
| "VPC Peering supports transitive routing" | **Wrong.** Peering is non-transitive. A↔B and B↔C does NOT mean A↔C. |
| "You can use VPC-B's NAT Gateway from VPC-A via peering" | **Wrong.** Edge-to-edge routing is not supported through peering. |
| "TGW Peering supports BGP propagation" | **Wrong.** Only static routes across TGW peering. |
| "TGW is free, you only pay for data" | **Wrong.** $0.05/hour per attachment + $0.02/GB data processing. |
| "VPC Peering has a per-hour cost" | **Wrong.** The peering connection itself is free. Only data transfer costs apply. |
| "VGW supports ECMP for VPN" | **Wrong.** Only TGW supports ECMP. VGW uses one tunnel actively per VPN. |
| "All TGW attachments share one route table" | **Misleading.** Default behavior, but best practice is multiple route tables for segmentation. |
| "TGW is single-AZ like NAT Gateway" | **Wrong.** TGW is HA across all AZs in the region. |
| "VPC Peering requires an IGW" | **Wrong.** Peering is a direct logical connection — no gateway, no hardware. |
| "Appliance mode is enabled by default" | **Wrong.** It's disabled by default. Must be explicitly enabled for firewall/inspection VPCs. |
| "Cross-region VPC Peering is unencrypted" | **Wrong.** Cross-region peering IS encrypted (AES-256). Same-region travels on AWS private network. |
| "TGW can connect to VPCs in other regions directly" | **Wrong.** TGW is regional. You need TGW Peering to connect to TGWs in other regions. |
| "SG references work across cross-region peering" | **Wrong.** SG references only work for **same-region** peering. Cross-region must use CIDRs. |

---

## 7. Cheat Sheet

### Must-Know Facts

```
VPC PEERING
├── Non-transitive (A↔B + B↔C ≠ A↔C)
├── No edge-to-edge routing (can't share IGW/NAT/VPN/Endpoint)
├── FREE (no hourly charge)
├── No bandwidth bottleneck
├── Cross-region: encrypted, uses AWS backbone
├── Same-region: can reference peered SGs by ID
├── Cross-region: must use CIDR blocks (no SG reference)
├── Both sides must update route tables + SGs
├── CIDRs must NOT overlap
├── Limit: 50 peerings/VPC (adjustable to 125)
└── Full mesh = N(N-1)/2 connections → unmanageable at scale

TRANSIT GATEWAY
├── Regional, HA across all AZs
├── Fully TRANSITIVE routing
├── Hub-and-spoke model
├── Attachments: VPC, VPN, DX Gateway, TGW Peering, Connect
├── Cost: $0.05/hr per attachment + $0.02/GB processed
├── Own route tables → SEGMENTATION (key exam pattern)
│   ├── Association = which table I look up FROM
│   └── Propagation = which tables I advertise TO
├── ECMP: aggregate VPN bandwidth (TGW only, not VGW)
├── Multicast: TGW only AWS service that supports it
├── Appliance Mode: enable for firewall/inspection VPCs
├── TGW Peering: cross-region, STATIC ROUTES ONLY
├── Shared via RAM across accounts
├── TGW Flow Logs available
├── Max 50 Gbps per VPC attachment (burst)
└── Limit: 5,000 attachments, 10,000 routes per RT
```

### Decision Flowchart

```
How many VPCs need to communicate?
│
├── 2-5 VPCs, no transitive routing needed
│   └── Need on-prem connectivity sharing?
│       ├── No  → VPC Peering (free, simple)
│       └── Yes → Transit Gateway (peering can't transit to VPN/DX)
│
├── 5+ VPCs
│   └── Transit Gateway (hub-and-spoke)
│       ├── Need environment isolation? → Multiple TGW route tables
│       ├── Need centralized firewall? → Inspection VPC + Appliance Mode
│       ├── Need centralized egress? → Egress VPC with NAT GW
│       └── Need VPN >1.25 Gbps? → ECMP with multiple VPN connections
│
├── Cross-region connectivity
│   ├── Simple (2 VPCs) → Cross-region VPC Peering (encrypted, free)
│   └── Complex (many VPCs) → TGW Peering (static routes only)
│
└── Need multicast?
    └── Transit Gateway (only option)
```

### "If the question says X, think Y"

| Question Keyword/Phrase | Answer Direction |
|------------------------|------------------|
| "transitive routing" / "hub-and-spoke" | Transit Gateway |
| "2 VPCs, cost-effective" | VPC Peering |
| "on-prem access for all VPCs" | TGW (attach VPN/DX to TGW) |
| "use another VPC's NAT/IGW" | NOT possible via peering → TGW |
| "VPN bandwidth > 1.25 Gbps" | TGW + ECMP |
| "multicast" | TGW Multicast |
| "isolate prod from dev through same hub" | TGW route table segmentation |
| "full mesh, 50 VPCs" | TGW (not 1,225 peering connections) |
| "firewall drops traffic / asymmetric routing" | TGW Appliance Mode |
| "share TGW across accounts" | AWS RAM |
| "dynamic routing cross-region via TGW" | Not supported — TGW Peering = static only |
| "SG reference cross-region peering" | Not supported — use CIDR blocks |
| "encrypt cross-region VPC traffic" | Already encrypted (VPC Peering & TGW Peering) |
| "minimize operational overhead, many VPCs" | TGW over peering mesh |
| "minimize cost, few VPCs, no transit" | VPC Peering over TGW |
| "centralized egress / single NAT" | TGW + Egress VPC |
| "prevent unauthorized VPC connectivity" | SCP + TGW route table isolation |

---

*Last updated: 2026-04-16 | AWS SAP-C02 Exam Prep*
