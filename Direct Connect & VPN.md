# AWS SAP-C02 Study Notes: Direct Connect & VPN (BGP, LAG, Failover)

---

## 1. Core Concepts & Theory

### AWS Direct Connect (DX)

**AWS Direct Connect** is a dedicated, private physical network connection from your on-premises data center to AWS. It bypasses the public internet entirely.

**How it physically works:**

```
On-Premises       Customer/Partner        AWS Direct Connect
Data Center        Router (CPE)             Location (DX PoP)
┌──────────┐      ┌──────────┐            ┌──────────────────┐
│ Corporate│──────│ Customer │── Cross ───│  AWS DX Router   │──── AWS Backbone
│ Network  │      │ Router   │   Connect  │  (Port)          │       │
└──────────┘      └──────────┘            └──────────────────┘       │
                        │                                            │
                   BGP Session                               ┌─────────────────┐
                   (802.1Q VLAN tagging)                     │ AWS Region      │
                                                             │  ├── VPC-A      │
                                                             │  ├── VPC-B      │
                                                             │  └── Public Svcs│
                                                             └─────────────────┘
```

**Steps to set up Direct Connect:**

1. **Request a connection** in the AWS Console (or use a DX Partner)
2. AWS allocates a **port** at a DX Location (1 Gbps or 10 Gbps or 100 Gbps)
3. Establish a **cross-connect** (physical cable) between your router and the AWS DX router at the DX Location
4. Create **Virtual Interfaces (VIFs)** over the connection
5. Configure **BGP** on your router

### DX Connection Types

| Type | Speed | How You Get It |
|------|-------|----------------|
| **Dedicated Connection** | 1 Gbps, 10 Gbps, or 100 Gbps | You request directly from AWS; physical port allocated at DX Location |
| **Hosted Connection** | 50 Mbps to 10 Gbps | Provisioned by an **AWS DX Partner** on their behalf; sub-1G speeds available |
| **Hosted VIF** | Shares partner's connection | Partner creates a VIF on their existing DX connection and assigns it to your account |

**Key distinctions:**

| Feature | Dedicated | Hosted Connection | Hosted VIF |
|---------|-----------|-------------------|------------|
| Port ownership | You own the port | Partner owns, you get a connection | Partner owns, you get a VIF |
| Speed options | 1/10/100 Gbps | 50 Mbps–10 Gbps (increments) | Partner-defined |
| LAG support | **Yes** | **No** | **No** |
| Multiple VIFs | **Yes** (up to 50 per connection) | **1 VIF per hosted connection** | 1 VIF |
| Lead time | **Weeks to months** (physical provisioning) | Days to weeks (partner-dependent) | Days |
| Jumbo Frames | 9001 MTU (private/transit VIF) | Depends on partner | Depends on partner |

**Exam tip:** If a question says "the company needs connectivity within days" → **Hosted Connection via DX Partner** (or VPN as interim). Dedicated connections take weeks/months.

### Virtual Interfaces (VIFs)

VIFs are logical interfaces created over a DX connection. Each VIF is a **VLAN** (802.1Q tagged).

| VIF Type | Purpose | Connects To | BGP | VLAN |
|----------|---------|-------------|-----|------|
| **Private VIF** | Access VPCs via private IP | Virtual Private Gateway (VGW) or Direct Connect Gateway | Private ASN | Customer-assigned |
| **Public VIF** | Access AWS public services (S3, DynamoDB, EC2 public IPs) via public IP | AWS public endpoints directly | Public ASN or private ASN | Customer-assigned |
| **Transit VIF** | Access VPCs via Transit Gateway | Direct Connect Gateway → Transit Gateway | Private ASN | Customer-assigned |

**Critical architecture:**

```
                          ┌── Private VIF ──► DX Gateway ──► VGW ──► VPC (single VPC)
                          │
DX Connection ────────────┼── Public VIF  ──► AWS Public Services (S3, DynamoDB, etc.)
                          │
                          └── Transit VIF ──► DX Gateway ──► TGW ──► Multiple VPCs
```

### Direct Connect Gateway (DXGW)

A **Direct Connect Gateway** is a globally available resource that enables you to connect your DX connection to VPCs in **any region** (except China).

| Property | Detail |
|----------|--------|
| Scope | **Global** — works across regions |
| Connects to | VGWs (via Private VIF) or TGWs (via Transit VIF) |
| Max associations | **20 VGWs** and/or **6 TGWs** (across accounts/regions) |
| Transitivity | **Non-transitive** between VGWs — VPC-A and VPC-B connected via same DXGW cannot communicate with each other through the DXGW |
| Cross-account | **Yes** — DXGW in Account A can associate with VGW/TGW in Account B |

**Without DXGW:** One DX connection → one Private VIF → one VGW → one VPC in one region.
**With DXGW:** One DX connection → one Private VIF → DXGW → multiple VGWs/TGWs in multiple regions.

```
                                    ┌── VGW (VPC in us-east-1)
                                    │
DX ── Private VIF ── DXGW ─────────┼── VGW (VPC in eu-west-1)
                                    │
                                    └── VGW (VPC in ap-southeast-1)

                                    ┌── TGW us-east-1 → 50 VPCs
DX ── Transit VIF ── DXGW ─────────┤
                                    └── TGW eu-west-1 → 30 VPCs
```

**Exam tip:** DXGW + TGW (via Transit VIF) is the **most scalable** pattern for connecting on-prem to hundreds of VPCs across multiple regions through a single DX connection.

### BGP (Border Gateway Protocol)

BGP is **mandatory** for Direct Connect. You cannot use static routing with DX.

| Concept | Detail |
|---------|--------|
| Protocol | **BGP (eBGP)** — exterior BGP between your ASN and AWS's ASN |
| AWS ASN | **7224** (default) for Private VIF; **16509** for Public VIF. You can specify a custom ASN when creating VGW/DXGW. |
| Your ASN | Use your own public ASN (Public VIF) or any private ASN 64512–65534 (Private VIF) |
| BFD | **Bidirectional Forwarding Detection** — recommended for fast failure detection (sub-second) |
| MD5 Authentication | Supported and recommended for BGP session security |
| Prefix limits | **100 prefixes** advertised from on-prem to AWS per BGP session (exceeding causes session teardown) |
| AWS → on-prem prefixes | Private VIF: VPC CIDRs. Public VIF: AWS public IP ranges. Transit VIF: VPCs behind TGW (max **20 prefixes from TGW**) |

**BGP Communities (advanced routing control):**

| Community | Purpose |
|-----------|---------|
| `7224:9100` | **Local preference** — prefer this path over others (higher = preferred) |
| `7224:9200` | Prefer path within a DX location |
| `7224:9300` | Prefer path within a region |
| `7224:8100` | Do NOT propagate beyond local region |
| `7224:8200` | Do NOT propagate beyond continent |
| No community | Propagate globally (default for Public VIF) |

**Exam tip:** BGP communities control **which AWS regions receive your advertised routes** (Public VIF) and allow you to influence **path selection** when you have multiple DX connections.

### BGP Route Advertisement

**On-prem → AWS:**
- You advertise your on-prem prefixes to AWS
- AWS installs them in the VGW/TGW route table
- Max 100 prefixes per BGP session
- If you advertise more than 100, **BGP session is torn down**

**AWS → On-prem:**
- Private VIF: AWS advertises the VPC CIDR(s) associated with the VGW
- Transit VIF: AWS advertises **allowed prefixes** configured on the DXGW-TGW association (max 20)
- Public VIF: AWS advertises all public IP prefixes for the region(s) based on BGP communities

### Link Aggregation Groups (LAGs)

A **LAG** bundles multiple DX connections into a single **logical connection** using LACP (Link Aggregation Control Protocol — 802.3ad).

| Property | Detail |
|----------|--------|
| Purpose | Aggregate bandwidth and provide link-level redundancy |
| Protocol | **LACP** (active mode required on your side) |
| Max connections per LAG | **4** (of the same speed) |
| Same speed required | **Yes** — all connections must be same port speed (all 1G or all 10G or all 100G) |
| Same DX Location | **Yes** — all connections must terminate at the **same DX Location** |
| Same AWS device | **Yes** — all connections in a LAG land on the **same AWS DX device** |
| Minimum links | Configurable — set the minimum number of active links required for the LAG to be operational |
| VIFs | Created on the LAG (not individual connections) |

**Example:** 4 x 10 Gbps connections in a LAG = **40 Gbps** aggregate bandwidth.

```
┌── DX Connection 1 (10G) ──┐
├── DX Connection 2 (10G) ──┤
├── DX Connection 3 (10G) ──┼── LAG (40G logical) ── VIFs
└── DX Connection 4 (10G) ──┘
    (Same DX Location, same AWS device)
```

**Critical LAG limitation (exam favorite):** Because all connections in a LAG terminate on the **same AWS device** at the **same DX Location**, a LAG does NOT protect against:
- DX Location failure
- AWS device failure
- Fiber cut to the DX Location

**For true HA, you need connections at DIFFERENT DX Locations** (not just LAG).

---

### AWS Site-to-Site VPN

A **Site-to-Site VPN** creates an encrypted IPsec tunnel over the **public internet** between your on-premises network and AWS.

**Components:**

```
On-Premises                                            AWS
┌──────────────┐                                ┌──────────────────┐
│  Customer     │        IPsec Tunnels          │  VGW or TGW      │
│  Gateway      │══════════════════════════════│  (VPN Endpoint)   │
│  (CGW)        │   Tunnel 1 (Active)           │                  │
│  Public IP    │   Tunnel 2 (Standby/Active)   │  2 endpoints in  │
│               │                                │  different AZs   │
└──────────────┘                                └──────────────────┘
```

| Component | Detail |
|-----------|--------|
| **Customer Gateway (CGW)** | Representation of your on-prem VPN device in AWS. You specify your device's public IP and ASN. |
| **Virtual Private Gateway (VGW)** | AWS-side VPN endpoint attached to a **single VPC**. Supports both VPN and DX. |
| **Transit Gateway (TGW)** | AWS-side VPN endpoint for **multiple VPCs**. Supports ECMP. |
| **VPN Connection** | Two IPsec tunnels for redundancy (each terminates on a different AWS AZ endpoint) |

### VPN Key Technical Details

| Feature | Detail |
|---------|--------|
| Encryption | **AES-128 or AES-256** (GCM) |
| Integrity | SHA-1 or SHA-256 |
| DH Groups | 2, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24 |
| IKE versions | **IKEv1 and IKEv2** |
| Tunnels per connection | **2** (for redundancy — different AZ endpoints) |
| Bandwidth per tunnel | **Up to 1.25 Gbps** |
| Routing | **Static** or **Dynamic (BGP)** |
| NAT Traversal | Supported (NAT-T) |
| Dead Peer Detection (DPD) | Supported — detects failed peer and triggers failover |
| Accelerated VPN | Uses **AWS Global Accelerator** — routes traffic to nearest AWS edge, reduces latency |

### VPN Routing: Static vs BGP

| Feature | Static Routing | BGP (Dynamic) |
|---------|---------------|---------------|
| Configuration | Manually specify on-prem CIDRs in VPN config | Routes exchanged via BGP |
| Failover | Manual (slower) | Automatic (BGP convergence, ~30-90 seconds) |
| ECMP on TGW | **Not supported** | **Supported** |
| Recommended for | Simple setups, single VPN | Production, multi-VPN, DX backup |
| Active-Active tunnels | Not recommended (no load balancing) | Supported (especially with TGW ECMP) |

**Exam tip:** For any question involving **automatic failover** or **ECMP**, the answer requires **BGP** (not static routing).

### Accelerated Site-to-Site VPN

| Property | Standard VPN | Accelerated VPN |
|----------|-------------|-----------------|
| Path | Public internet end-to-end | Public internet → nearest AWS edge → AWS backbone |
| Latency | Variable (internet-dependent) | More consistent, lower latency |
| Cost | Standard VPN pricing | VPN cost + Global Accelerator hourly + data transfer premium |
| Use case | Normal connectivity | Global/distributed on-prem sites needing consistent performance |
| Compatible with | VGW or TGW | **TGW only** |

**Exam tip:** "Accelerated VPN" = VPN + Global Accelerator. Only works with TGW, NOT VGW.

### AWS Client VPN (Brief — Sometimes Tested)

| Property | Detail |
|----------|--------|
| Type | **Remote access VPN** (user laptops → AWS), NOT site-to-site |
| Protocol | OpenVPN |
| Auth | Active Directory, SAML-based federated, mutual certificate |
| Split tunnel | Supported — only AWS-destined traffic goes through VPN |
| Deployed in | VPC subnet (uses ENIs) |

**Exam tip:** "Remote workers accessing VPC" → Client VPN. "Data center to AWS" → Site-to-Site VPN.

---

## Connectivity Options Comparison (Master Table)

| Feature | Direct Connect | Site-to-Site VPN | DX + VPN (VPN over DX) |
|---------|---------------|-----------------|----------------------|
| Medium | **Dedicated private line** | **Public internet** | **DX with IPsec overlay** |
| Encryption | **Not encrypted** by default | **Encrypted** (IPsec) | **Encrypted** (IPsec over DX) |
| Bandwidth | 50 Mbps – 100 Gbps | Up to 1.25 Gbps/tunnel | Limited by DX speed |
| Latency | **Consistent, low** | **Variable** (internet) | **Consistent, low** |
| Setup time | **Weeks to months** | **Minutes** | Weeks (DX) + minutes (VPN) |
| Cost | **Higher** (port fee + data out) | **Lower** ($0.05/hr per connection) | Highest (DX + VPN) |
| HA | Requires dual connections | 2 tunnels per connection | Both DX + VPN provide HA layers |
| Use case | Production, high throughput, consistent latency | Backup, quick setup, cost-sensitive | Encryption required over private line |

---

## 2. Design Patterns & Best Practices

### Pattern 1: Maximum Resilience (AWS Recommended)

AWS's recommended architecture for **maximum resiliency**:

```
DX Location 1                    DX Location 2
┌────────────────┐              ┌────────────────┐
│ DX Conn 1 (10G)│              │ DX Conn 3 (10G)│
│ DX Conn 2 (10G)│              │ DX Conn 4 (10G)│
└───────┬────────┘              └───────┬────────┘
        │                               │
        └───────────┬───────────────────┘
                    │
              DXGW / VGW / TGW
```

**4 connections at 2 DX Locations:**
- Survives: single connection failure, single device failure, single DX Location failure
- Each location has 2 connections for device-level redundancy
- 2 locations for location-level redundancy
- Connections at the same location can be in a **LAG** for bandwidth aggregation

### Pattern 2: High Resilience

```
DX Location 1              DX Location 2
┌──────────────┐          ┌──────────────┐
│ DX Conn 1    │          │ DX Conn 2    │
└──────┬───────┘          └──────┬───────┘
       │                         │
       └─────────┬───────────────┘
                 │
           DXGW / VGW / TGW
```

**2 connections at 2 DX Locations:**
- Survives: single connection failure, single DX Location failure
- More cost-effective than maximum resilience
- **Minimum for production workloads**

### Pattern 3: DX + VPN Backup (Most Common Exam Pattern)

```
On-Premises
     │
     ├──── Direct Connect ──── DXGW ──── VGW/TGW  (Primary: high bandwidth, low latency)
     │
     └──── Site-to-Site VPN ──────────── VGW/TGW  (Backup: encrypted, over internet)
```

**How failover works with BGP:**
- Both DX and VPN advertise routes to the VGW/TGW
- AWS **prefers DX over VPN** by default (DX has higher Local Preference in BGP)
- If DX fails, BGP converges and traffic shifts to VPN automatically
- When DX recovers, traffic shifts back to DX

**Route preference order (AWS → on-prem direction):**
1. **Longest prefix match** always wins first
2. If prefix lengths are equal:
   - Static routes in VGW route table (highest priority)
   - Routes from DX Private VIF (BGP)
   - Routes from DX Transit VIF (BGP)
   - VPN static routes
   - VPN BGP routes (lowest priority)

**On-prem → AWS direction:**
- Controlled by your router's BGP configuration
- Use **AS-PATH prepending** to make VPN less preferred
- Use **BGP Local Preference** to prefer DX
- Use **MED (Multi-Exit Discriminator)** for path selection between multiple DX connections

### Pattern 4: VPN over Direct Connect (Encrypted DX)

```
On-Premises ──── DX Connection ──── Public VIF ──── Site-to-Site VPN ──── VGW/TGW
```

**Why:** Direct Connect is NOT encrypted. If compliance requires encryption over the private line:
1. Create a **Public VIF** on the DX connection (to reach AWS VPN public endpoints)
2. Create a **Site-to-Site VPN** that routes over the DX Public VIF
3. Result: **IPsec-encrypted traffic over the dedicated DX line**

**Alternative:** **MACsec** (802.1AE) — Layer 2 encryption directly on the DX connection.

| Feature | VPN over DX | MACsec |
|---------|-------------|--------|
| Encryption layer | Layer 3 (IPsec) | Layer 2 (802.1AE) |
| Bandwidth impact | Reduces throughput (encryption overhead) | Minimal impact |
| Supported on | All DX connections (via Public VIF) | **10 Gbps and 100 Gbps Dedicated connections only** |
| Complexity | Moderate (VPN setup required) | Low (enabled on DX connection) |
| Requires | Public VIF + VPN connection | MACsec-capable router, DX port with MACsec |

**Exam tip:** "Encrypt Direct Connect traffic with LEAST operational overhead" + 10G/100G connection → **MACsec**. "Encrypt DX" + 1G or hosted connection → **VPN over DX Public VIF**.

### Pattern 5: TGW with DX and VPN ECMP

```
On-Premises
     │
     ├── DX ── Transit VIF ── DXGW ── TGW  (Primary)
     │
     ├── VPN 1 ──────────────────── TGW     (Backup + ECMP)
     ├── VPN 2 ──────────────────── TGW     (Backup + ECMP)
     ├── VPN 3 ──────────────────── TGW     (Backup + ECMP)
     └── VPN 4 ──────────────────── TGW     (Backup + ECMP)
```

- DX handles primary traffic (bulk data, low latency)
- Multiple VPNs attached to TGW with **ECMP** for aggregate backup bandwidth
- If DX fails: ECMP across 4 VPNs = ~5 Gbps aggregate (4 x 1.25 Gbps per active tunnel)
- BGP required on all connections for ECMP

### Pattern 6: Multi-Region DX Architecture

```
On-Premises
     │
     DX Connection (us-east-1 DX Location)
     │
     ├── Private VIF ── DXGW ──┬── VGW (VPC in us-east-1)
     │                          ├── VGW (VPC in eu-west-1)
     │                          └── VGW (VPC in ap-southeast-1)
     │
     └── Transit VIF ── DXGW ──┬── TGW (us-east-1) → 50 VPCs
                                └── TGW (eu-west-1) → 30 VPCs
```

**Key points:**
- Single DX connection → DXGW → VPCs in multiple regions
- Transit VIF is preferred for scale (TGW supports many VPCs)
- Private VIF works for few VPCs (direct to VGW)
- **DXGW is global** — physical DX location doesn't limit region access

### When to Use What

| Scenario | Solution |
|----------|----------|
| Need high bandwidth, consistent latency to AWS | Direct Connect |
| Need connectivity in < 1 hour | Site-to-Site VPN |
| Need encrypted connection, short-term | VPN |
| Need encrypted connection, long-term, high bandwidth | DX + MACsec (10G/100G) or VPN over DX |
| Need to connect to multiple VPCs in one region | DX → Transit VIF → TGW |
| Need to connect to VPCs in multiple regions | DX → DXGW → multiple VGWs or TGWs |
| Need VPN bandwidth > 1.25 Gbps | Multiple VPNs → TGW with ECMP |
| Production workload, DX as primary | DX primary + VPN backup (BGP for automatic failover) |
| Regulatory: must be private line | Direct Connect (NOT VPN — VPN uses public internet) |
| Budget-constrained, acceptable latency variation | VPN only |

### Anti-Patterns

| Anti-Pattern | Why It's Wrong | Correct Approach |
|-------------|----------------|------------------|
| Single DX connection, no backup | Single point of failure — DX maintenance or failure = total outage | At minimum DX + VPN backup. For production: 2 DX at 2 locations. |
| LAG for HA across locations | LAG requires same DX location + same device | Separate connections at different locations |
| Assuming DX is encrypted | DX is NOT encrypted by default | Add MACsec or VPN over DX |
| VPN for bulk data transfer (TB+) | 1.25 Gbps max per tunnel, variable latency | Direct Connect |
| Static routing with DX backup by VPN | No automatic failover | Use BGP on both for auto-failover |
| 200 prefixes advertised to AWS over BGP | Max 100 — session will be **torn down** | Summarize/aggregate routes to stay under 100 |
| Public VIF to access VPCs | Public VIF is for AWS public services | Use Private VIF (VGW) or Transit VIF (TGW) |
| DX Gateway expecting transitive VPC routing | DXGW to VGW is non-transitive between VPCs | Use DXGW → TGW for inter-VPC routing |
| Accelerated VPN on VGW | Not supported | Use TGW for Accelerated VPN |

---

## 3. Security & Compliance

### Encryption Summary

| Path | Encrypted by Default? | How to Encrypt |
|------|----------------------|----------------|
| Site-to-Site VPN | **Yes** (IPsec AES-256) | Built-in |
| Direct Connect | **No** | MACsec (L2, 10G/100G only) or VPN over DX (L3) |
| DX cross-region via DXGW | **No** (AWS backbone, but not encrypted) | VPN over DX |
| TGW VPN | **Yes** (IPsec) | Built-in |
| Client VPN | **Yes** (TLS/OpenVPN) | Built-in |

### IAM & SCP Controls

**Key API actions:**

```
directconnect:CreateConnection
directconnect:CreatePrivateVirtualInterface
directconnect:CreatePublicVirtualInterface
directconnect:CreateTransitVirtualInterface
directconnect:CreateDirectConnectGateway
directconnect:CreateDirectConnectGatewayAssociation
ec2:CreateVpnConnection
ec2:CreateVpnGateway
ec2:CreateCustomerGateway
```

**SCP — Restrict DX to approved regions only (prevent DXGW association to unapproved regions):**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Deny",
    "Action": "directconnect:CreateDirectConnectGatewayAssociation",
    "Resource": "*",
    "Condition": {
      "StringNotEquals": {
        "aws:RequestedRegion": ["us-east-1", "eu-west-1"]
      }
    }
  }]
}
```

### BGP Security

| Control | Detail |
|---------|--------|
| **MD5 Authentication** | Prevents BGP session hijacking; shared secret between your router and AWS |
| **Prefix limit (100)** | AWS tears down BGP session if exceeded — prevents route table overflow |
| **AS-PATH filtering** | Filter routes on your side to prevent accepting unexpected prefixes |
| **Route filtering (prefix lists)** | On your router: only advertise intended prefixes, only accept expected AWS prefixes |
| **BFD** | Sub-second failure detection for BGP sessions |

### Logging & Monitoring

| Tool | What It Monitors |
|------|-----------------|
| **CloudWatch Metrics (DX)** | `ConnectionState`, `ConnectionBpsIngress/Egress`, `ConnectionPpsIngress/Egress`, `ConnectionLightLevelTx/Rx` (optical signal — detects physical layer issues) |
| **CloudWatch Metrics (VPN)** | `TunnelState` (0=down, 1=up), `TunnelDataIn/Out`, `TunnelCPUUtilization` |
| **CloudTrail** | API calls: CreateConnection, CreateVirtualInterface, CreateVpnConnection |
| **AWS Config** | Configuration changes to VPN and DX resources |
| **Direct Connect Resiliency Toolkit** | Tests failover by bringing down BGP sessions on DX connections |
| **AWS Health Dashboard** | DX maintenance events, scheduled outages |
| **VPC Flow Logs** | Traffic flowing through VGW/TGW (after it enters the VPC) |

**Critical CloudWatch alarms:**

| Alarm | Why |
|-------|-----|
| `ConnectionState != available` | DX physical link is down |
| `ConnectionLightLevelRx < threshold` | Optical signal degrading — fiber issue developing |
| `TunnelState = 0` for both tunnels | VPN fully down — no connectivity |
| BGP session down (from your router) | Route exchange stopped — traffic will fail |

### Compliance Patterns

| Requirement | Solution |
|-------------|----------|
| "All traffic to AWS must be private" | Direct Connect (bypasses public internet) |
| "All traffic must be encrypted" | VPN (IPsec), or DX + MACsec, or VPN over DX |
| "Must have private AND encrypted" | DX + MACsec or VPN over DX |
| "No single point of failure" | Dual DX connections at dual locations + VPN backup |
| "Meet 99.99% SLA" | AWS DX SLA requires **2 connections at 2 DX locations** (maximum resiliency model) |
| "Data residency" | DX Location in specific country + DXGW association limited to approved regions via SCP |

---

## 4. Cost Optimization

### Direct Connect Costs

| Component | Cost |
|-----------|------|
| **Port hourly fee (Dedicated)** | 1 Gbps: ~$0.30/hr. 10 Gbps: ~$1.50/hr. 100 Gbps: ~$13.50/hr |
| **Data Transfer Out (DTO)** | ~$0.02/GB (varies by region; **cheaper than internet DTO**) |
| **Data Transfer In** | **Free** |
| **Cross-connect fee** | Charged by the **DX Location (colocation facility)**, not AWS |
| **DXGW** | **Free** |
| **VIF** | **Free** |

**Exam tip:** DX Data Transfer Out is **cheaper** than internet DTO. At high volumes (10+ TB/month), DX pays for itself in DTO savings alone.

### VPN Costs

| Component | Cost |
|-----------|------|
| **VPN connection** | ~$0.05/hr per connection (~$36/month) |
| **Data Transfer Out** | Standard internet DTO rates (~$0.09/GB) |
| **Data Transfer In** | **Free** |
| **Accelerated VPN** | Additional Global Accelerator charges ($0.015/hr + $0.015/GB) |

### Cost Optimization Strategies

| Strategy | Detail |
|----------|--------|
| **DX for high-volume** | If DTO > 10 TB/month, DX is cheaper than VPN (lower per-GB cost offsets hourly port fee) |
| **Right-size DX port** | Don't over-provision — 1 Gbps is much cheaper than 10 Gbps if utilization is low |
| **Hosted Connection for < 1 Gbps** | Don't pay for a full dedicated 1G port if you only need 200 Mbps |
| **Avoid cross-region DX data charges** | If VPCs are in the same region as the DX Location, data stays local |
| **VPN as backup only** | Hourly VPN cost is small; valuable as failover even if rarely used |
| **Monitor utilization** | CloudWatch `ConnectionBpsEgress/Ingress` — resize if consistently under-utilized |
| **Consider DX pricing models** | Some DX Locations offer 1-year or 3-year agreements for discounts |
| **Gateway Endpoints** | S3/DynamoDB access via Gateway Endpoint avoids sending that traffic over DX/VPN |

### Cost Comparison: DX vs VPN

| Monthly Transfer | DX 1G Cost | VPN Cost | Winner |
|-----------------|-----------|----------|--------|
| 100 GB | ~$221 (port) + $2 (DTO) = $223 | ~$36 + $9 (DTO) = $45 | **VPN** |
| 1 TB | ~$221 + $20 = $241 | ~$36 + $90 = $126 | **VPN** |
| 10 TB | ~$221 + $200 = $421 | ~$36 + $900 = $936 | **DX** |
| 50 TB | ~$221 + $1,000 = $1,221 | ~$36 + $4,500 = $4,536 | **DX** (by far) |

**Crossover point:** Roughly **5-10 TB/month** — above this, DX becomes more cost-effective.

---

## 5. High Availability, Disaster Recovery & Resilience

### AWS DX Resiliency Models (Official AWS Guidance)

#### Maximum Resiliency

```
DX Location A         DX Location B
┌──────────┐          ┌──────────┐
│ Conn 1   │          │ Conn 3   │
│ Conn 2   │          │ Conn 4   │
└────┬─────┘          └────┬─────┘
     └─────────┬───────────┘
               │
         VGW / TGW
```

- **4 connections across 2 locations**
- Survives: any single failure (connection, device, location)
- **Required for 99.99% SLA**
- Each location has 2 connections on **separate AWS devices**

#### High Resiliency

```
DX Location A         DX Location B
┌──────────┐          ┌──────────┐
│ Conn 1   │          │ Conn 2   │
└────┬─────┘          └────┬─────┘
     └─────────┬───────────┘
               │
         VGW / TGW
```

- **2 connections across 2 locations**
- Survives: single connection failure, single location failure
- **Minimum for production**

#### Development / Non-Critical

```
DX Location A
┌──────────┐
│ Conn 1   │ + VPN backup
└────┬─────┘
     │
   VGW / TGW
```

- **1 DX connection + VPN backup**
- DX failure → automatic failover to VPN (if BGP is configured)
- **Not recommended for production**

### VPN HA

Each VPN connection has **2 tunnels** terminating on different AZ endpoints:

```
                    ┌── Tunnel 1 ──► AZ-a endpoint
Customer Gateway ───┤
                    └── Tunnel 2 ──► AZ-b endpoint
```

**For higher HA:**
```
                     ┌── VPN Connection 1 (2 tunnels) ──► VGW/TGW
CGW 1 (DC site A) ──┘
                     ┌── VPN Connection 2 (2 tunnels) ──► VGW/TGW
CGW 2 (DC site B) ──┘
```

- 2 CGWs at different sites → 2 VPN connections → 4 tunnels total
- BGP ensures automatic failover between all paths

### Failover Scenarios & Expected Behavior

| Failure | DX + VPN Backup | Expected Behavior |
|---------|----------------|-------------------|
| Single DX connection fails (of 2) | Traffic shifts to surviving DX connection | BGP convergence: 30-90 seconds |
| All DX connections fail | Traffic shifts to VPN | BGP convergence: 30-90 seconds. BFD: sub-second detection + BGP convergence |
| DX recovers | Traffic shifts back to DX (preferred) | Automatic (DX has higher local preference) |
| VPN Tunnel 1 fails | Traffic shifts to Tunnel 2 | DPD detection + failover: ~10-30 seconds |
| Both VPN tunnels fail | Total VPN failure; only DX remains | DPD detection: ~30 seconds |
| DX Location maintenance | **Pre-announced** by AWS Health Dashboard | Pre-shift traffic by adjusting BGP (AS-PATH prepend) |

### Testing Failover

**AWS Direct Connect Resiliency Toolkit:**
- Bring down BGP sessions on individual DX connections programmatically
- Validates failover behavior without physical intervention
- **Exam tip:** Use this for regular failover testing — don't physically unplug cables

### DR Architecture with DX

```
Region A (Primary)                     Region B (DR)
┌───────────────────────┐             ┌───────────────────────┐
│ DX Location 1 ── TGW  │             │ DX Location 3 ── TGW  │
│ DX Location 2 ── TGW  │             │ VPN backup    ── TGW  │
│                       │             │                       │
│ VPCs (Active)         │◄── TGW ──►  │ VPCs (Passive/Standby)│
│                       │   Peering   │                       │
└───────────────────────┘             └───────────────────────┘
                    │                              │
                    └────── On-Premises ───────────┘
                         (BGP to both regions)
```

---

## 6. Exam-Focused Section

### Straightforward Questions

**Q1:** A company needs a private, dedicated connection to AWS with consistent low latency and 5 Gbps throughput. What should they use?

**A:** **AWS Direct Connect** — dedicated connection with 10 Gbps port. Provides private, consistent latency, and supports the required throughput. VPN cannot guarantee consistent latency or 5 Gbps.

---

**Q2:** A company's Direct Connect connection is currently unencrypted. They need to add encryption with MINIMAL operational overhead. They have a 10 Gbps Dedicated connection. What should they use?

**A:** **MACsec (802.1AE)** — Layer 2 encryption directly on the DX connection. Available on 10 Gbps and 100 Gbps dedicated connections. Lowest overhead compared to VPN over DX.

---

**Q3:** A company needs immediate connectivity to AWS while waiting for their Direct Connect provisioning (6 weeks). What should they use as an interim solution?

**A:** **AWS Site-to-Site VPN** — can be set up in minutes over the public internet. Once DX is ready, configure DX as primary with VPN as backup using BGP.

---

**Q4:** What is the maximum number of prefixes you can advertise from on-premises to AWS over a single BGP session on Direct Connect?

**A:** **100 prefixes.** Exceeding this causes the BGP session to be **torn down**. Summarize/aggregate routes to stay under the limit.

---

**Q5:** A company has a Direct Connect connection with a Private VIF to a single VPC in us-east-1. They now need to connect to VPCs in 3 additional regions. What should they add?

**A:** A **Direct Connect Gateway (DXGW).** Associate the Private VIF with the DXGW, then associate VGWs (or TGWs) from each region with the DXGW. No new physical connections needed.

---

**Q6:** How does AWS determine route preference when both Direct Connect and VPN advertise the same prefix to a VGW?

**A:** **Direct Connect is preferred over VPN.** AWS route preference order: DX BGP routes > VPN static routes > VPN BGP routes. If DX fails, traffic automatically fails over to VPN via BGP convergence.

---

**Q7:** A company needs to aggregate bandwidth across multiple Direct Connect connections for a total of 40 Gbps. What should they use?

**A:** A **Link Aggregation Group (LAG)** with 4 x 10 Gbps Dedicated connections at the same DX Location. LAG uses LACP (802.3ad) to bond them into a single 40 Gbps logical connection.

---

### Tricky / Scenario-Based Questions

**Q1:** A company has two 10 Gbps DX connections in a LAG at DX Location A. They experience a complete outage when the DX Location has a power failure. They thought the LAG provided high availability. What went wrong, and how should they fix it?

**A:** **LAG does NOT provide location-level HA.** All connections in a LAG must be at the **same DX Location on the same AWS device**. A LAG only protects against individual link failure, not device or location failure.

**Fix:** Add a second DX connection (or LAG) at a **different DX Location (Location B)**. This is the **High Resiliency** model. Use BGP for automatic failover between locations.

> **Why "add more connections to the existing LAG" is wrong:** More connections at the same location still fail if the location fails.
> **Why "use DX Resiliency Toolkit" is wrong:** The toolkit tests failover, it doesn't provide HA.
>
> **Key phrase:** "LAG" + "single DX Location" + "complete outage" → LAG doesn't protect against location failure.

---

**Q2:** A company's DX connection uses a Private VIF connected to a VGW on VPC-A. They also have VPC-B peered with VPC-A. They want on-prem servers to reach VPC-B through VPC-A. Traffic is not flowing. Why?

**A:** **Edge-to-edge routing is not supported.** Traffic arriving via DX → VGW → VPC-A cannot transit through a VPC Peering connection to reach VPC-B. This is the same limitation as IGW/NAT through peering.

**Fix:** Either:
1. Create a separate Private VIF and VGW for VPC-B
2. Use a **Transit VIF → DX Gateway → Transit Gateway** with both VPCs attached to TGW (recommended for scale)

> **Key phrase:** "on-prem to VPC through another VPC" + "DX/VPN + Peering" → **edge-to-edge routing not supported**.

---

**Q3:** A solutions architect has a Direct Connect connection and is advertising 120 on-premises prefixes to AWS via BGP. The BGP session keeps dropping. What is the cause?

**A:** The **100-prefix limit** is being exceeded. AWS tears down the BGP session when more than 100 prefixes are advertised on a single BGP session.

**Fix:** **Summarize/aggregate** routes on the on-prem router (e.g., combine `10.1.0.0/24` through `10.1.255.0/24` into `10.1.0.0/16`).

> **Why "increase the prefix limit" is wrong:** The 100-prefix limit is **not adjustable** on the AWS side.
> **Why "switch to static routing" is wrong:** DX requires BGP — static routing is not supported on DX.
>
> **Key phrase:** "BGP session keeps dropping" + "many prefixes" → 100-prefix limit exceeded.

---

**Q4:** A company needs to access AWS S3 from on-premises via Direct Connect, ensuring traffic never traverses the public internet. They currently only have a Private VIF. What should they do?

**A:** Create a **Public VIF** on the existing DX connection. A Public VIF allows access to AWS public services (including S3) over the DX connection — traffic stays on the AWS backbone and **never traverses the public internet**.

**Alternative:** If they have a Transit VIF → TGW, create an **S3 VPC Endpoint** in a VPC attached to the TGW. On-prem traffic → DX → TGW → VPC → S3 Endpoint → S3.

> **Why "Private VIF" alone is wrong:** Private VIF can only access VPC resources (private IPs). S3 is a public service with public endpoints.
> **Why "create an IGW" is wrong:** IGW provides VPC internet access, but doesn't help on-prem reach S3.
>
> **Key phrase:** "Access S3 from on-prem via DX without internet" → **Public VIF** or Transit VIF → TGW → VPC Endpoint.

---

**Q5:** A company has a TGW with 4 VPN connections (BGP-enabled) and ECMP turned on. They expect 5 Gbps aggregate throughput, but they're only seeing 1.25 Gbps. What's wrong?

**A:** ECMP requires traffic to be sent from **multiple source/destination IP pairs** to distribute across tunnels. If all traffic is between a single source and destination IP, it will all hash to the same tunnel (1.25 Gbps).

**Fix:** Ensure traffic is coming from **multiple source IPs** on-prem, or use multiple connections from **different Customer Gateways** to ensure proper distribution.

> **Why "ECMP isn't enabled" is less likely:** The question states ECMP is on.
> **Why "need more VPN connections" is wrong:** 4 connections with ECMP can theoretically deliver 5+ Gbps if traffic is distributed.
>
> **Key phrase:** "ECMP enabled" + "not getting expected throughput" + "single flow" → ECMP is per-flow, not per-packet.

---

**Q6:** A company has a DX Gateway associated with VGWs for VPC-A and VPC-B. An engineer expects VPC-A and VPC-B to communicate through the DX Gateway. They cannot. Why?

**A:** **DX Gateway is non-transitive between VGW associations.** VPC-A and VPC-B cannot communicate through the DXGW — it only facilitates on-prem ↔ VPC traffic, not VPC ↔ VPC traffic.

**Fix:** Use **VPC Peering** or **Transit Gateway** for inter-VPC communication.

> **Key phrase:** "two VPCs communicate through DX Gateway" → **not possible (non-transitive)**.

---

**Q7:** A company has DX as primary and VPN as backup. During a planned DX maintenance window, they want to gracefully shift ALL traffic to VPN BEFORE DX goes down. How?

**A:** Use **BGP AS-PATH prepending** on the DX connection to make the DX routes less preferred. Your on-prem router will shift traffic to VPN (shorter AS-PATH). After maintenance, remove the prepending to shift back.

**Alternative:** Use the **AWS Direct Connect Resiliency Toolkit** to programmatically bring down the DX BGP session, which forces traffic to VPN.

> **Why "just wait for DX to go down" is wrong:** The question says "gracefully" and "BEFORE DX goes down" — they want controlled failover, not reactive.
> **Why "change VPN route to static with higher priority" is wrong:** For VGW, DX BGP always takes priority over VPN regardless of static/dynamic. You must make DX less attractive via BGP manipulation.
>
> **Key phrase:** "graceful failover" + "before maintenance" → BGP AS-PATH prepending or DX Resiliency Toolkit.

---

### Common Exam Traps & Pitfalls

| Trap | Reality |
|------|---------|
| "Direct Connect is encrypted" | **Wrong.** DX is NOT encrypted by default. Use MACsec (10G/100G) or VPN over DX. |
| "LAG provides location-level HA" | **Wrong.** LAG = same location, same device. For location HA, use connections at different locations. |
| "DX Gateway enables VPC-to-VPC communication" | **Wrong.** DXGW is non-transitive between VGW associations. Only on-prem ↔ VPC. |
| "You can use static routing on DX" | **Wrong.** DX **requires BGP**. Static routing is not an option. |
| "VPN can match DX bandwidth" | **Wrong.** VPN max is 1.25 Gbps/tunnel. Even with ECMP, practical limit is ~5-10 Gbps. DX supports up to 100 Gbps. |
| "Advertise unlimited prefixes via BGP" | **Wrong.** Max 100 prefixes per session. Exceeding = session teardown (not adjustable). |
| "Accelerated VPN works on VGW" | **Wrong.** Accelerated VPN requires **TGW**. |
| "ECMP works on VGW" | **Wrong.** VGW does NOT support ECMP. Only TGW supports ECMP for VPN. |
| "Public VIF = internet access" | **Wrong.** Public VIF provides access to **AWS public services only** (S3, DynamoDB endpoints), not general internet. |
| "One DX connection = one region" | **Wrong.** With DXGW, one DX connection can reach VPCs in **any region**. |
| "Transit VIF = Private VIF" | **Wrong.** Transit VIF connects to TGW (many VPCs). Private VIF connects to VGW (single VPC). |
| "DX failover to VPN is instant" | **Wrong.** BGP convergence takes ~30-90 seconds. BFD helps detect faster, but convergence still takes time. |
| "Hosted Connection supports LAG" | **Wrong.** Only Dedicated Connections support LAG. |
| "Cross-connect fee is charged by AWS" | **Wrong.** Cross-connect fee is charged by the **colocation facility**, not AWS. |
| "MACsec works on 1 Gbps connections" | **Wrong.** MACsec requires **10 Gbps or 100 Gbps Dedicated connections** only. |
| "VPN over DX uses Private VIF" | **Wrong.** VPN over DX uses **Public VIF** (to reach AWS VPN public endpoints). |

---

## 7. Cheat Sheet

### Must-Know Facts

```
DIRECT CONNECT
├── Dedicated private connection (NOT encrypted)
├── Speeds: 50 Mbps–100 Gbps
├── Requires BGP (no static routing)
├── Setup: weeks to months
├── 3 VIF types:
│   ├── Private VIF → VGW (single VPC)
│   ├── Transit VIF → TGW (many VPCs via DXGW)
│   └── Public VIF → AWS public services (S3, etc.)
├── DX Gateway: GLOBAL, connects DX to VGWs/TGWs in any region
│   └── Non-transitive between VGW associations
├── LAG: up to 4 connections, same speed, same location, same device
│   └── Does NOT provide location-level HA
├── BGP limits: 100 prefixes max (session torn down if exceeded)
├── Encryption: MACsec (10G/100G only) or VPN over DX Public VIF
├── HA: 2 connections at 2 locations minimum for production
│   └── 4 connections at 2 locations for 99.99% SLA
└── Data Transfer: In=free, Out=~$0.02/GB (cheaper than internet)

SITE-TO-SITE VPN
├── Encrypted (IPsec AES-256) over public internet
├── Setup: minutes
├── 2 tunnels per connection (different AZ endpoints)
├── Max 1.25 Gbps per tunnel
├── Routing: Static or BGP
├── On VGW: no ECMP, no Accelerated VPN
├── On TGW: ECMP (multi-VPN aggregation), Accelerated VPN
├── Automatic failover with BGP (not with static)
├── Cost: $0.05/hr + internet DTO rates
└── Accelerated VPN = VPN + Global Accelerator (TGW only)

ROUTE PREFERENCE (AWS → On-Prem, via VGW)
├── 1. Longest prefix match (always first)
├── 2. DX Private VIF (BGP) — HIGHEST
├── 3. DX Transit VIF (BGP)
├── 4. VPN Static routes
└── 5. VPN BGP routes — LOWEST

BGP KEY FACTS
├── Mandatory for DX, optional for VPN
├── AS-PATH prepending → make a path less preferred
├── BGP Communities → control route propagation scope
├── BFD → sub-second failure detection
├── MD5 auth → secure BGP sessions
└── 100 prefix limit → session torn down if exceeded
```

### Decision Flowchart

```
Need AWS connectivity from on-prem?
│
├── How quickly?
│   ├── Immediately (minutes) → Site-to-Site VPN
│   └── Can wait weeks → Direct Connect
│
├── Bandwidth needs?
│   ├── < 1.25 Gbps → VPN may suffice
│   ├── 1-10 Gbps → DX 1G/10G (or multiple VPN with TGW ECMP as interim)
│   └── 10-100 Gbps → DX 10G/100G (or LAG for aggregation)
│
├── Encryption required?
│   ├── VPN → built-in (IPsec)
│   ├── DX 10G/100G → MACsec (least overhead)
│   └── DX 1G or Hosted → VPN over DX Public VIF
│
├── How many VPCs?
│   ├── 1 VPC → DX → Private VIF → VGW
│   ├── Multiple VPCs, one region → DX → Transit VIF → TGW
│   └── Multiple VPCs, multi-region → DX → DXGW → multiple TGWs/VGWs
│
├── HA requirements?
│   ├── Non-critical → 1 DX + VPN backup
│   ├── Production → 2 DX at 2 locations
│   ├── 99.99% SLA → 4 DX at 2 locations (maximum resiliency)
│   └── VPN-only HA → 2 CGWs + 2 VPN connections + BGP
│
└── VPN bandwidth > 1.25 Gbps?
    └── TGW + ECMP + multiple VPN connections (BGP required)
```

### "If the question says X, think Y"

| Question Keyword/Phrase | Answer Direction |
|------------------------|------------------|
| "dedicated private connection" | Direct Connect |
| "consistent latency" / "predictable performance" | Direct Connect (NOT VPN) |
| "encrypted connection to AWS" | VPN (or DX + MACsec / VPN over DX) |
| "immediate / quick connectivity" | VPN (minutes vs DX weeks) |
| "interim connectivity while DX provisions" | VPN now, DX later, keep VPN as backup |
| "encrypt Direct Connect" + 10G/100G | MACsec |
| "encrypt Direct Connect" + 1G / hosted | VPN over DX (Public VIF) |
| "VPN bandwidth > 1.25 Gbps" | TGW + ECMP + multiple VPN connections |
| "DX to multiple VPCs" | Transit VIF → DXGW → TGW |
| "DX to multiple regions" | DX Gateway (global) |
| "on-prem to S3 over DX, no internet" | Public VIF |
| "LAG" / "aggregate DX bandwidth" | Same location, same speed, max 4, LACP |
| "DX location-level failure" | Need connections at 2 different DX locations |
| "BGP session keeps dropping" | 100-prefix limit exceeded |
| "automatic failover DX ↔ VPN" | BGP on both (not static routing) |
| "graceful failover before maintenance" | AS-PATH prepending or DX Resiliency Toolkit |
| "ECMP" | TGW only (NOT VGW) |
| "Accelerated VPN" | TGW only (NOT VGW) |
| "99.99% DX SLA" | 4 connections, 2 DX locations (maximum resiliency) |
| "DX Gateway + VPC-to-VPC traffic" | Not possible (non-transitive) — use TGW or Peering |
| "private VIF for S3" | Wrong — need Public VIF or Transit VIF → VPC Endpoint |
| "reduce DX cost" | Right-size port, use Hosted Connection if < 1 Gbps, monitor utilization |
| "10+ TB/month to AWS" | DX cheaper than VPN for data transfer |
| "multicast over DX" | Not a DX feature — multicast = TGW multicast domain |

---

*Last updated: 2026-04-16 | AWS SAP-C02 Exam Prep*
