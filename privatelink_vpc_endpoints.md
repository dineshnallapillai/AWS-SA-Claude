# AWS PrivateLink & VPC Endpoints — SAP-C02 Study Notes

---

## 1. Core Concepts & Theory

### What is AWS PrivateLink?

AWS PrivateLink is a networking technology that provides **private connectivity between VPCs, AWS services, and on-premises networks without exposing traffic to the public internet**. Traffic stays on the Amazon backbone network. It uses ENIs (Elastic Network Interfaces) with private IPs in your VPC as the entry point.

**Key distinction:** PrivateLink is not a standalone service you "create" — it is the underlying technology powering Interface VPC Endpoints and Endpoint Services.

### VPC Endpoint Types

| Feature | Gateway Endpoint | Interface Endpoint (PrivateLink) |
|---------|-----------------|----------------------------------|
| Technology | Route table entry pointing to a prefix list | ENI with private IP in your subnet |
| Supported services | **S3 and DynamoDB ONLY** | 100+ AWS services + custom services |
| Cost | **Free** (no hourly or data charges) | $0.01/hr per AZ + $0.01/GB data processed |
| Availability | Per-region, not per-AZ | Per-AZ (ENI deployed in chosen subnets) |
| DNS | No private DNS by default | Private DNS resolves public service endpoint to private IP |
| Security | Controlled via endpoint policy + route table | Controlled via endpoint policy + security groups |
| Access from on-premises | **NOT directly accessible** from on-prem via VPN/DX | **Accessible** from on-prem via VPN/DX (through private IP) |
| Cross-region | No | No (same region only, but can be reached cross-region via peering/TGW) |
| Protocol | TCP only | TCP (and now UDP for some services) |

### Gateway Endpoints (S3 and DynamoDB)

- Adds a **prefix list entry** (pl-xxxxxxxx) to the route table of specified subnets.
- Traffic to S3/DynamoDB is routed through the VPC endpoint instead of an internet gateway or NAT gateway.
- **No ENI created, no private IP assigned.**
- You can have multiple gateway endpoints in a VPC (e.g., one per route table).
- **Cannot be accessed from outside the VPC** (on-prem, peered VPCs). This is the #1 exam trap.
- Endpoint policy controls which S3 buckets/DynamoDB tables can be accessed.
- **Free** — no hourly charge, no data processing charge.

### Interface Endpoints (PrivateLink)

- Creates an **ENI with a private IP** in each subnet you specify.
- Supports **100+ AWS services**: CloudWatch, KMS, SNS, SQS, API Gateway, Secrets Manager, SSM, ECS, ECR, Kinesis, STS, etc.
- **S3 also supports Interface Endpoints** (in addition to Gateway) — use this when you need on-prem access to S3 via DX/VPN.
- **Private DNS option:** When enabled, the default public DNS name (e.g., `sqs.us-east-1.amazonaws.com`) resolves to the private ENI IP inside the VPC. No application code changes needed.
- **Security groups** attached to the ENI control access.
- Can be accessed from on-prem via Direct Connect or VPN (traffic flows to the ENI private IP).

### VPC Endpoint Services (Custom PrivateLink)

You can create your own PrivateLink service to expose your application to other AWS accounts:

```
Service Provider                    Service Consumer
+-----------------+                 +-----------------+
| NLB / GWLB     |<--- PrivateLink ---|  Interface     |
| (your app)     |    connection    |  Endpoint       |
+-----------------+                 +-----------------+
```

**Requirements for creating an Endpoint Service:**
1. **Network Load Balancer (NLB)** or **Gateway Load Balancer (GWLB)** in the provider account.
2. Create a VPC Endpoint Service pointing to the NLB/GWLB.
3. Consumer creates an Interface Endpoint pointing to the service.
4. Provider can require **manual acceptance** of connection requests.
5. Supports **cross-account** access (consumer and provider in different accounts).

**Key detail:** The consumer sees only an ENI IP. They CANNOT see the provider's VPC CIDR, instance IPs, or any network details. This is **unidirectional** — consumer initiates connections to provider, not the other way.

### Gateway Load Balancer Endpoints (GWLBE)

- Used specifically with **Gateway Load Balancer** for inline traffic inspection (firewall, IDS/IPS).
- Creates a GWLB Endpoint in the consumer VPC that routes traffic to a GWLB in the security VPC.
- Traffic flow: Consumer VPC -> GWLBE -> GWLB -> Security appliances -> GWLB -> GWLBE -> Destination.
- Used in **centralized inspection architectures** with Transit Gateway.

### Key Terminology

| Term | Meaning |
|------|---------|
| **Endpoint (consumer side)** | The ENI or route entry in your VPC that connects to a service |
| **Endpoint Service (provider side)** | The NLB/GWLB-backed service you expose via PrivateLink |
| **Endpoint Policy** | IAM resource policy attached to the endpoint controlling access |
| **Private DNS** | Enables the public service hostname to resolve to the private endpoint IP |
| **Prefix List** | Managed list of IP ranges for S3/DynamoDB used in Gateway Endpoint route tables |
| **Service Name** | Format: `com.amazonaws.<region>.<service>` (e.g., `com.amazonaws.us-east-1.s3`) |

### Limits and Quotas

| Resource | Default Limit |
|----------|--------------|
| Gateway endpoints per VPC | 255 |
| Interface endpoints per VPC | 50 (can increase) |
| Endpoint services per account (per region) | 10 (can increase) |
| Subnets per Interface Endpoint | 1 per AZ |
| Security groups per Interface Endpoint | 10 |
| IP addresses per Interface Endpoint (per AZ) | 1 ENI = 1 primary private IP (+ optional IPv6) |
| Bandwidth per endpoint ENI | Up to 10 Gbps per AZ (bursts to 40 Gbps) |

---

## 2. Design Patterns & Best Practices

### When to Use

| Scenario | Solution |
|----------|----------|
| Private access to S3 from VPC (no on-prem need) | **Gateway Endpoint** (free) |
| Private access to S3 from on-premises via DX/VPN | **Interface Endpoint for S3** (PrivateLink) |
| Private access to DynamoDB from VPC | **Gateway Endpoint** (free, only option) |
| Private access to SQS, SNS, KMS, Secrets Manager, etc. | **Interface Endpoint** |
| Expose your SaaS app to customers in other accounts | **VPC Endpoint Service** (NLB + PrivateLink) |
| Centralized firewall/IDS inspection | **GWLB Endpoint** |
| Private API Gateway access | **Interface Endpoint** for `execute-api` |
| Private ECR image pulls from ECS/EKS | Interface Endpoints for `ecr.api`, `ecr.dkr`, and `s3` |

### When NOT to Use (Anti-Patterns)

- **Cross-region private connectivity:** PrivateLink is same-region only. Use VPC Peering, Transit Gateway, or CloudFront for cross-region.
- **Bidirectional communication:** PrivateLink is unidirectional (consumer -> provider). If you need bidirectional, use VPC Peering or TGW.
- **High-bandwidth bulk data transfer:** PrivateLink has per-ENI bandwidth limits. For bulk S3 transfers, Gateway Endpoint is free and has no bandwidth cap.
- **Simple VPC-to-VPC connectivity:** Use VPC Peering (free data transfer within AZ) or TGW instead of building custom PrivateLink services.
- **Public-facing services:** PrivateLink is for private access. Use ALB/CloudFront for public endpoints.

### Architectural Patterns

#### Pattern 1: Hub-and-Spoke with Centralized Endpoints

```
Spoke VPC A --\
Spoke VPC B ---+-- Transit Gateway ---> Hub VPC (Interface Endpoints)
Spoke VPC C --/                              |
                                        AWS Services (S3, SQS, KMS, etc.)
```

- All spoke VPCs route to centralized Interface Endpoints in the hub VPC via TGW.
- **Cost saving:** One set of endpoints shared across many VPCs instead of per-VPC endpoints.
- **Requires:** Private DNS disabled on the endpoint. Use Route 53 Private Hosted Zone (PHZ) with alias records to the endpoint DNS, associated with all spoke VPCs.
- **Exam tip:** The question will say "reduce cost of VPC endpoints across 50 VPCs" — answer is centralized endpoints in a shared services VPC.

#### Pattern 2: On-Premises Access to AWS Services

```
On-Premises --> Direct Connect / VPN --> VPC (Interface Endpoint) --> AWS Service
```

- On-prem resolves the AWS service DNS to the Interface Endpoint's private IP.
- Requires Route 53 Resolver Inbound Endpoint to forward DNS queries from on-prem to VPC.
- **Gateway Endpoints do NOT work here** — they are route-table based and only work within the VPC.
- **Exam keyword:** "on-premises access to S3 privately" = Interface Endpoint for S3 (not Gateway).

#### Pattern 3: Multi-Account SaaS via PrivateLink

```
SaaS Provider Account              Customer Account A
+-------------------+              +-------------------+
| NLB -> App Servers| <--Private   | Interface         |
| Endpoint Service  |    Link----> | Endpoint          |
+-------------------+              +-------------------+

                                   Customer Account B
                                   +-------------------+
                                   | Interface         |
                                   | Endpoint          |
                                   +-------------------+
```

- Provider exposes service via NLB + Endpoint Service.
- Each customer creates an Interface Endpoint in their VPC.
- **No VPC peering, no shared CIDRs, no IP overlap concerns.**
- Provider can whitelist accounts or require manual acceptance.
- Provider sees only the NLB target health, not customer VPC details.

#### Pattern 4: Private API Gateway

```
VPC (Interface Endpoint for execute-api) --> Private API Gateway --> Lambda/Backend
```

- API Gateway can be configured as **Private** endpoint type.
- Only accessible via Interface Endpoint for `execute-api`.
- API Gateway resource policy specifies which VPC Endpoints or VPCs can invoke it.
- **Exam keyword:** "API accessible only from within VPC" = Private API Gateway + Interface Endpoint.

#### Pattern 5: ECS/EKS Private Image Pull

For ECS tasks in private subnets (no NAT Gateway) to pull images from ECR:
- Interface Endpoint for `com.amazonaws.<region>.ecr.api`
- Interface Endpoint for `com.amazonaws.<region>.ecr.dkr`
- Gateway Endpoint for S3 (ECR stores layers in S3)
- Interface Endpoint for `com.amazonaws.<region>.logs` (for CloudWatch Logs if needed)

### Well-Architected Framework Alignment

| Pillar | PrivateLink Consideration |
|--------|--------------------------|
| **Security** | No public internet exposure. Data never leaves AWS network. Endpoint policies as additional access control. Security groups on ENIs. |
| **Reliability** | Deploy Interface Endpoints across multiple AZs. One ENI per AZ. If an AZ fails, other AZ endpoint ENIs still function. |
| **Cost** | Use free Gateway Endpoints for S3/DynamoDB when possible. Centralize Interface Endpoints via TGW for multi-VPC. Monitor data processing charges. |
| **Performance** | Traffic stays on AWS backbone (low latency). Consider AZ affinity — traffic to endpoint ENI in same AZ avoids cross-AZ data charges. |
| **Operational Excellence** | Enable VPC Flow Logs on endpoint ENIs. Use CloudWatch metrics for endpoint. Use CloudTrail for endpoint API calls. |
| **Sustainability** | Eliminates NAT Gateway for AWS service traffic (reduces unnecessary compute). |

---

## 3. Security & Compliance

### Endpoint Policies

Endpoint policies are **IAM resource policies** attached to VPC Endpoints that control which principals can use the endpoint and which resources they can access:

```json
{
  "Statement": [
    {
      "Sid": "AllowSpecificBucket",
      "Principal": "*",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::my-bucket/*"
    }
  ]
}
```

**Key rules:**
- Default endpoint policy allows **full access** (all actions, all resources, all principals).
- Endpoint policy is an **additional filter**, not a replacement for IAM. Both IAM policy AND endpoint policy must allow the action.
- Endpoint policy **does not grant permissions** — it only restricts what the endpoint allows.
- For S3 Gateway Endpoints: endpoint policy can restrict to specific buckets/prefixes.
- For Interface Endpoints: endpoint policy can restrict to specific actions/resources.

**Exam trap:** "Restrict S3 access to only a specific bucket through the VPC endpoint" — use endpoint policy (not bucket policy alone).

### S3 Bucket Policy with VPC Endpoint Condition

Restrict a bucket to only accept traffic from a specific VPC Endpoint:

```json
{
  "Statement": [
    {
      "Sid": "DenyNotFromVPCE",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": ["arn:aws:s3:::my-bucket", "arn:aws:s3:::my-bucket/*"],
      "Condition": {
        "StringNotEquals": {
          "aws:sourceVpce": "vpce-1234567890abcdef0"
        }
      }
    }
  ]
}
```

**Condition keys:**
- `aws:sourceVpce` — the specific VPC Endpoint ID
- `aws:sourceVpc` — the VPC ID (works with Gateway Endpoints)
- These conditions work in S3 bucket policies, SQS queue policies, SNS topic policies, etc.

### Security Groups on Interface Endpoints

- Every Interface Endpoint ENI has a **security group**.
- Inbound rules control who can access the endpoint (e.g., allow TCP 443 from 10.0.0.0/16).
- **Exam tip:** If the question mentions "control which instances can access the endpoint" — answer is security groups on the Interface Endpoint.
- Gateway Endpoints do NOT have security groups (they are route-table entries).

### Encryption

- Traffic to Interface Endpoints uses **TLS** (HTTPS) — encrypted in transit by default for all AWS service endpoints.
- PrivateLink traffic between consumer and provider is encrypted using AWS internal mechanisms.
- No additional encryption configuration needed at the PrivateLink layer.
- Application-level encryption (e.g., KMS for S3 objects) is orthogonal.

### Logging and Monitoring

| Mechanism | What it captures |
|-----------|-----------------|
| **VPC Flow Logs** | Network flow records for endpoint ENIs (src IP, dst IP, bytes, packets, accept/reject) |
| **CloudTrail** | API calls to create/delete/modify endpoints. API calls made THROUGH the endpoint appear as the service's API calls. |
| **CloudWatch Metrics** | ActiveConnections, BytesProcessed, NewConnections, PacketsProcessed, RstPacketsReceived for Interface Endpoints |
| **AWS Config** | Track endpoint configuration changes and compliance rules |
| **Access Advisor** | Shows which services were accessed through the endpoint (IAM-level) |

### SCP (Service Control Policy) Considerations

- SCPs can restrict the creation of VPC Endpoints:
```json
{
  "Effect": "Deny",
  "Action": "ec2:CreateVpcEndpoint",
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "ec2:VpceServiceName": [
        "com.amazonaws.us-east-1.s3",
        "com.amazonaws.us-east-1.dynamodb"
      ]
    }
  }
}
```
- This restricts accounts to only create endpoints for approved services — prevents data exfiltration via unauthorized endpoints.

### Data Exfiltration Prevention

**Problem:** A malicious insider creates a VPC Endpoint to a personal S3 bucket in another account.

**Solution layers:**
1. **SCP:** Restrict which endpoint services can be created.
2. **Endpoint Policy:** Restrict which S3 buckets the endpoint can access.
3. **S3 Bucket Policy:** Restrict access to `aws:sourceVpce` condition.
4. **VPC Endpoint Service allow-listing:** For custom PrivateLink services, provider controls who can connect.

---

## 4. Cost Optimization

### Pricing Model

**Gateway Endpoints (S3 and DynamoDB):**
- **$0.00** — completely free. No hourly charge, no data charge.
- This makes Gateway Endpoints the default choice for S3/DynamoDB access from within a VPC.

**Interface Endpoints:**
- **$0.01/hour per AZ** (~$7.20/month per AZ).
- **$0.01/GB** data processed (in both directions).
- Deploying in 3 AZs = $0.03/hour = ~$21.60/month base cost before data.
- Each AWS service needs its own endpoint (separate charges per service).

**PrivateLink (Endpoint Service — provider side):**
- **$0.01/hour per AZ** for the NLB (standard NLB pricing applies separately).
- **$0.01/GB** data processed through the endpoint.

### Cost Comparison: PrivateLink vs NAT Gateway for AWS Service Access

| Component | NAT Gateway | Interface Endpoint |
|-----------|-------------|-------------------|
| Hourly | $0.045/hr per AZ | $0.01/hr per AZ per service |
| Data | $0.045/GB | $0.01/GB |
| Per month (3 AZ, 1TB) | $32.40 + $45.00 = $77.40 | $21.60 + $10.00 = $31.60 |

**Exam insight:** If the question mentions "reduce data transfer costs for AWS API calls" or "eliminate NAT Gateway charges for AWS service traffic," the answer is VPC Endpoints.

### Cost-Saving Strategies

1. **Gateway Endpoint for S3/DynamoDB:** Always free. No reason not to use one.
2. **Centralized endpoints via TGW:** Instead of 50 endpoints in 50 VPCs ($1,080/month), use 1 endpoint set in a shared VPC ($21.60/month) + TGW cost.
3. **Minimize AZ deployment:** If HA is not critical for dev/test, deploy in fewer AZs.
4. **Consolidate traffic paths:** One Interface Endpoint per service per VPC handles all traffic for that service.
5. **Remove unused endpoints:** Endpoint charges accumulate even with zero traffic.
6. **Use Gateway Endpoint for S3 when no on-prem access is needed:** Avoid the Interface Endpoint charge.

### Cost Traps

- Deploying Interface Endpoints for S3 when Gateway Endpoint suffices (free vs $21.60+/month).
- Deploying endpoints in every spoke VPC instead of centralizing.
- Forgetting per-GB data processing charges on high-volume endpoints.
- Creating endpoints for services you rarely call (e.g., Interface Endpoint for STS but only authenticate once/hour).

---

## 5. High Availability, Disaster Recovery & Resilience

### HA Architecture

**Interface Endpoints:**
- Deploy an ENI in **each AZ** where your workload runs.
- If one AZ fails, the endpoint ENIs in other AZs continue to function.
- DNS resolves to all healthy endpoint IPs (round-robin within AZ-aware resolution).
- Best practice: deploy in all AZs of your VPC.

**Gateway Endpoints:**
- Managed by AWS. Route table-based. Highly available by default.
- No AZ-specific deployment needed — AWS handles redundancy.

**Endpoint Services (provider side):**
- NLB must be in multiple AZs for HA.
- If NLB target is unhealthy in one AZ, traffic is routed to healthy AZ targets.
- Enable **cross-zone load balancing** on the NLB for better distribution.

### Failover and Resilience Patterns

1. **Multi-AZ endpoint deployment:** Always deploy Interface Endpoints in all AZs where your workload exists.
2. **Health checks:** Interface Endpoints have built-in health monitoring. If an AZ endpoint ENI becomes unhealthy, DNS stops returning its IP.
3. **Route table failover (Gateway Endpoints):** If you need to switch from Gateway Endpoint to Interface Endpoint (e.g., for on-prem migration), update the route table and DNS.

### Cross-Region Considerations

- PrivateLink does NOT work cross-region natively.
- For cross-region private access: use **Inter-Region VPC Peering** or **Transit Gateway Inter-Region Peering** to route to an endpoint in the target region.
- **Exam scenario:** "Access S3 in us-west-2 from a VPC in eu-west-1 privately" — Interface Endpoint in us-west-2 VPC + inter-region peering or TGW peering.

### DR Patterns

| Scenario | Solution |
|----------|----------|
| Primary region fails | Pre-create endpoints in DR region VPC. Route via Route 53 failover. |
| Provider Endpoint Service fails | Consumer endpoint connection goes unhealthy. Consumer must switch to backup service or region. |
| AZ failure | ENIs in other AZs handle traffic automatically. |

---

## 6. Exam-Focused Section

### Straightforward Questions

**Q1: A company wants to access S3 from EC2 instances in a private subnet without internet access. What is the most cost-effective solution?**

**A:** S3 Gateway Endpoint. It is free (no hourly or data charges) and routes S3 traffic through the VPC endpoint via the route table.

---

**Q2: Which VPC endpoint type creates an ENI in your subnet?**

**A:** Interface Endpoint. Gateway Endpoints modify route tables; Interface Endpoints deploy ENIs with private IPs.

---

**Q3: A company needs to expose an internal microservice to hundreds of customer accounts without VPC peering. What should they use?**

**A:** AWS PrivateLink with an NLB + VPC Endpoint Service. Each customer creates an Interface Endpoint. No CIDR overlap concerns, no VPC peering needed.

---

**Q4: Which services support Gateway Endpoints?**

**A:** Only **Amazon S3** and **Amazon DynamoDB**. All other services use Interface Endpoints. This is a top memorization item.

---

**Q5: An Interface Endpoint has private DNS enabled. What happens when an application calls `sqs.us-east-1.amazonaws.com`?**

**A:** The DNS name resolves to the **private IP of the endpoint ENI** inside the VPC, not the public SQS endpoint. Traffic stays private. No code changes needed.

---

**Q6: Can a VPC endpoint policy grant permissions that an IAM policy does not?**

**A:** No. Endpoint policies are restrictive only. Both the IAM policy AND the endpoint policy must allow the action. Endpoint policies filter; they do not grant.

---

**Q7: A company uses a NAT Gateway for EC2 instances to call AWS APIs (KMS, SQS, SNS). They want to reduce costs. What should they do?**

**A:** Create Interface VPC Endpoints for KMS, SQS, and SNS. Remove the NAT Gateway if no other internet access is needed. Interface Endpoints cost ~$0.01/hr per AZ vs NAT Gateway at $0.045/hr per AZ, plus lower data charges.

---

### Tricky / Scenario-Based Questions

**Q1: A company has on-premises servers connected via Direct Connect. They need to access S3 privately from on-prem. They already have an S3 Gateway Endpoint. Will this work?**

**A: No.** Gateway Endpoints are route-table based and only work for traffic originating **within the VPC**. On-prem traffic entering via DX/VPN does not traverse the VPC route table for S3 prefixes.

**Solution:** Create an **Interface Endpoint for S3** (PrivateLink). On-prem DNS resolves S3 hostname to the private ENI IP. Traffic from on-prem -> DX -> VPC -> ENI -> S3.

**Why wrong answers are wrong:**
- "Use the existing Gateway Endpoint with a route propagation to on-prem" — Gateway Endpoints don't propagate to on-prem. The prefix list is for VPC route tables only.
- "Use a VPN to the S3 public endpoint" — exposes traffic to the internet, violates "privately" requirement.

**Exam keyword:** "on-premises" + "privately" + "S3" = Interface Endpoint (not Gateway).

---

**Q2: A company has 100 VPCs connected via Transit Gateway. They want private access to AWS services. The current approach of creating Interface Endpoints in every VPC is too expensive. What should they recommend?**

**A:** Create Interface Endpoints in a **centralized shared services VPC**. Route traffic from all spoke VPCs via Transit Gateway to the shared VPC. Use **Route 53 Private Hosted Zones** associated with all VPCs to resolve service DNS to the endpoint ENI IPs.

**Why wrong answers are wrong:**
- "Create Gateway Endpoints in each VPC" — Gateway Endpoints only support S3 and DynamoDB, not all services. Also they are VPC-local.
- "Use a NAT Gateway in the shared VPC" — traffic goes to public internet, not private. More expensive.
- "Enable Private DNS on the shared endpoint" — Private DNS only works within the VPC where the endpoint is created. For cross-VPC, you need Route 53 PHZ.

**Exam keywords:** "100 VPCs" + "reduce cost" + "private access" = centralized endpoints + TGW + R53 PHZ.

---

**Q3: A SaaS provider wants to offer their service to customers in different AWS accounts. The customers require that traffic never traverses the public internet, and there should be no IP address overlap concerns. Which approach should they use?**

**A:** AWS PrivateLink. Provider creates an **NLB** in front of their application and a **VPC Endpoint Service**. Each customer creates an **Interface Endpoint** in their VPC. Traffic is private, unidirectional, and there are no CIDR overlap concerns (no peering).

**Why wrong answers are wrong:**
- "VPC Peering" — does not scale (max 125 peering connections per VPC), CIDR overlap is a problem, and full network visibility is a security concern.
- "Transit Gateway with RAM sharing" — requires the provider to manage routing complexity, potential CIDR overlap, and gives customers more network visibility than needed.
- "ALB with HTTPS" — traffic goes over the internet (even with TLS, it traverses public network).

**Exam keywords:** "SaaS" + "multiple accounts" + "no internet" + "no IP overlap" = PrivateLink.

---

**Q4: An application in a private subnet needs to pull container images from ECR. The VPC has no NAT Gateway and no internet access. What endpoints are required?**

**A:** Three endpoints minimum:
1. Interface Endpoint for `com.amazonaws.<region>.ecr.api` (ECR API calls)
2. Interface Endpoint for `com.amazonaws.<region>.ecr.dkr` (Docker registry API)
3. **Gateway Endpoint for S3** (ECR stores image layers in S3)

Optionally: Interface Endpoint for `com.amazonaws.<region>.logs` (CloudWatch Logs for container logging).

**Why wrong answers are wrong:**
- "Only ECR Interface Endpoint" — misses the S3 dependency. ECR image layers are stored in S3. Without S3 access, image pulls fail.
- "Interface Endpoint for S3" — works but costs money. Gateway Endpoint for S3 is free and sufficient for VPC-internal access.

**Exam keyword:** "ECR" + "private subnet" + "no NAT" = 3 endpoints (ecr.api + ecr.dkr + S3 gateway).

---

**Q5: A company has an S3 Gateway Endpoint with a restrictive endpoint policy that allows access to only `my-bucket`. Users report they can no longer access the AWS Management Console's S3 page. Why?**

**A:** The S3 console makes API calls to the S3 service (ListBuckets, etc.) that route through the Gateway Endpoint. The restrictive endpoint policy blocks these calls because they are not scoped to `my-bucket`. The ListBuckets API call is not bucket-specific and is denied.

**Solution:** Either add ListBuckets permission to the endpoint policy, or access the console from outside the VPC (where traffic doesn't go through the gateway endpoint).

**Exam insight:** Endpoint policies affect ALL traffic routing through that endpoint, including console and CLI operations from within the VPC.

---

**Q6: A company wants to restrict an S3 bucket so that it can ONLY be accessed from a specific VPC. They add a bucket policy with `aws:sourceVpc` condition. Users accessing the bucket from the AWS console (from their corporate network) are now denied. Is this expected?**

**A: Yes.** The `aws:sourceVpc` condition means only requests originating from that VPC (via a VPC Endpoint) are allowed. Console access from a corporate network does not originate from the VPC. The Deny policy blocks it.

**Why wrong answers are wrong:**
- "Add an IAM policy to override the bucket policy" — explicit Deny in bucket policy cannot be overridden by IAM Allow.
- "Use aws:sourceIp to also allow console access" — this works but changes the security model. The question is asking whether the behavior is expected.

**Exam trap:** `aws:sourceVpc` and `aws:sourceVpce` conditions block ALL non-VPC traffic, including console, CLI from laptops, and cross-account access not going through the endpoint.

---

**Q7: A company uses PrivateLink to connect to a third-party SaaS vendor. The vendor is in a different AWS region. How should they establish this connection?**

**A:** PrivateLink is **same-region only**. The options are:
1. Ask the vendor to deploy their endpoint service in the customer's region.
2. Use **Inter-Region VPC Peering** or **Transit Gateway Inter-Region Peering** to route from the customer's VPC to a VPC in the vendor's region that has the endpoint.

**Why wrong answers are wrong:**
- "Create a cross-region Interface Endpoint" — this does not exist. PrivateLink is regional.
- "Use a VPN over the internet to the vendor's endpoint" — defeats the purpose of private connectivity.

**Exam keyword:** "different region" + "PrivateLink" = not supported natively. Need peering.

---

### Common Exam Traps & Pitfalls

1. **Gateway Endpoint vs Interface Endpoint for S3:**
   - S3 supports BOTH. Gateway is free, Interface costs money.
   - Gateway: VPC-internal only. Interface: accessible from on-prem.
   - **If question mentions on-prem access to S3 -> Interface Endpoint.**
   - **If question mentions cost optimization for S3 access from VPC -> Gateway Endpoint.**

2. **"Private DNS" confusion:**
   - Private DNS on Interface Endpoints only works within the VPC where the endpoint is created.
   - For centralized endpoints accessed via TGW/Peering, you must use Route 53 PHZ instead.
   - If Private DNS is enabled, you cannot have both a Gateway Endpoint and an Interface Endpoint for S3 in the same VPC (DNS conflict).

3. **Gateway Endpoints and on-prem:**
   - Gateway Endpoints are NEVER accessible from on-premises. Period.
   - This is the #1 most-tested PrivateLink concept.

4. **PrivateLink is one-directional:**
   - Consumer initiates connections to provider. Provider cannot initiate connections back.
   - If bidirectional communication is needed, PrivateLink is wrong — use VPC Peering or TGW.

5. **NLB is required for Endpoint Services:**
   - You cannot create an Endpoint Service with an ALB. Must be NLB or GWLB.
   - **Exception:** As of 2024, PrivateLink now supports ALB as a target, but this is very new. Exam likely still expects NLB.

6. **Endpoint Policy is not a substitute for IAM:**
   - Endpoint policy restricts. IAM grants. Both must allow.
   - A permissive endpoint policy (default) + restrictive IAM = IAM controls.
   - A restrictive endpoint policy + permissive IAM = endpoint policy controls.

7. **Interface Endpoint for DynamoDB does NOT exist:**
   - DynamoDB only supports Gateway Endpoints.
   - If the question asks about private DynamoDB access from on-prem, the answer involves **DAX** or routing through a VPC proxy, not a direct Interface Endpoint.

8. **Security groups vs Endpoint policies:**
   - Security groups: network-level (IP, port, protocol).
   - Endpoint policies: IAM-level (actions, resources, principals).
   - Gateway Endpoints: endpoint policy only (no security groups).
   - Interface Endpoints: both endpoint policy AND security groups.

---

## 7. Cheat Sheet

### Must-Know Facts

- **Gateway Endpoints:** S3 + DynamoDB only. Free. Route table based. VPC-internal only.
- **Interface Endpoints:** 100+ services. $0.01/hr/AZ + $0.01/GB. ENI-based. Accessible from on-prem.
- **S3 supports both.** Gateway = free, VPC-only. Interface = paid, on-prem accessible.
- **DynamoDB only supports Gateway.** No Interface Endpoint option.
- **PrivateLink requires NLB (or GWLB)** on the provider side. Not ALB (exam assumption).
- **Private DNS = auto-resolve public hostname to private IP.** Only works in the endpoint's VPC.
- **Centralized endpoints + TGW + R53 PHZ** = cost optimization for multi-VPC.
- **Endpoint policies restrict, never grant.** Both IAM + endpoint policy must allow.
- **PrivateLink is same-region only.**
- **Gateway Endpoints are NOT accessible from on-premises.**

### Key Differentiators

| If the question says... | Think... |
|------------------------|----------|
| "On-premises access to S3 privately" | Interface Endpoint for S3 |
| "Cost-effective S3 access from VPC" | Gateway Endpoint (free) |
| "Expose service to other AWS accounts without peering" | PrivateLink (NLB + Endpoint Service) |
| "Reduce NAT Gateway costs for AWS API calls" | Interface Endpoints for those services |
| "100 VPCs need private access, reduce cost" | Centralized endpoints in shared VPC + TGW |
| "Private API Gateway" | Interface Endpoint for execute-api + Private API type |
| "ECR pulls from private subnet" | ecr.api + ecr.dkr Interface Endpoints + S3 Gateway Endpoint |
| "Restrict S3 bucket to VPC only" | Bucket policy with aws:sourceVpce or aws:sourceVpc condition |
| "No IP overlap between provider and consumer" | PrivateLink (no CIDR sharing) |
| "Bidirectional communication between VPCs" | NOT PrivateLink. Use VPC Peering or TGW. |
| "Cross-region private access" | NOT PrivateLink alone. Need inter-region peering. |
| "Inline traffic inspection" | GWLB + GWLB Endpoints |

### Decision Flowchart

```
Need private access to an AWS service from VPC?
|
+-- Is it S3?
|   +-- Need on-prem access? --> Interface Endpoint (PrivateLink)
|   +-- VPC-only access? --> Gateway Endpoint (FREE)
|
+-- Is it DynamoDB?
|   +-- Gateway Endpoint (only option)
|
+-- Is it any other AWS service (SQS, SNS, KMS, etc.)?
|   +-- Interface Endpoint
|
+-- Need to expose YOUR service to other accounts?
|   +-- NLB + VPC Endpoint Service (PrivateLink)
|
+-- Need inline traffic inspection (firewall/IDS)?
|   +-- GWLB + GWLB Endpoint
|
+-- Need to reduce cost across many VPCs?
|   +-- Centralized endpoints in shared VPC + TGW + R53 PHZ
```

### Comparison: Gateway Endpoint vs Interface Endpoint vs VPC Peering vs TGW

| Feature | Gateway Endpoint | Interface Endpoint | VPC Peering | Transit Gateway |
|---------|-----------------|-------------------|-------------|-----------------|
| Use case | S3/DynamoDB | AWS services | VPC-to-VPC | Hub-and-spoke |
| Cost | Free | $0.01/hr/AZ | Free (same AZ data) | $0.05/hr + data |
| Cross-region | No | No | Yes | Yes |
| On-prem access | No | Yes | No | Yes (with VPN/DX) |
| Transitive routing | No | N/A | No | Yes |
| CIDR overlap | N/A | N/A | Not allowed | Not allowed |
| Max connections | 255/VPC | 50/VPC | 125/VPC | 5000 attachments |
| Security | Endpoint policy | SG + Endpoint policy | SG + NACL | SG + NACL + RT |

---

*Generated for AWS SAP-C02 exam preparation. Last updated: 2026-04-26.*
