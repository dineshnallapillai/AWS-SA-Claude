# Cross-Account Access & Resource Sharing (RAM)

## 1. Core Concepts & Theory

---

### Why Cross-Account Access?

In a multi-account environment, workloads in different accounts need to interact:
- Application in Account A reads data from S3 in Account B
- CI/CD pipeline in Shared Services deploys to Production Account
- Centralized monitoring collects metrics from all accounts
- Network infrastructure (Transit Gateway) shared across all accounts
- Security team investigates resources in any account

**Cross-account access mechanisms:**
1. **IAM Role Assumption** (sts:AssumeRole)
2. **Resource-Based Policies** (bucket policies, key policies, etc.)
3. **AWS Resource Access Manager (RAM)** (share resources without duplication)
4. **Service-Specific Sharing** (AMI sharing, snapshot sharing, etc.)
5. **VPC Peering / Transit Gateway** (network-level connectivity)

---

### IAM Role Assumption (sts:AssumeRole)

**What it is:** The primary mechanism for cross-account access. A principal in Account A assumes a role in Account B, receiving temporary credentials scoped to that role's permissions.

#### How It Works

```
┌──────────────────────────────────────────────────────────────────────┐
│                   Cross-Account Role Assumption                        │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Account A (Source)              Account B (Target)                    │
│  ┌────────────────┐             ┌─────────────────────┐              │
│  │ IAM Role/User  │──AssumeRole──→│ Cross-Account Role  │            │
│  │                │             │                     │              │
│  │ Needs:         │             │ Trust Policy:       │              │
│  │ iam:AssumeRole │             │ Allow Account A     │              │
│  │ on target ARN  │             │                     │              │
│  └────────────────┘             │ Permissions Policy: │              │
│                                  │ What the role can do│              │
│  Returns: Temporary             └─────────────────────┘              │
│  credentials (STS tokens)                                             │
│  - AccessKeyId                                                        │
│  - SecretAccessKey                                                    │
│  - SessionToken                                                       │
│  - Expiration                                                         │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

#### Trust Policy (Account B — Target Role)

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::111111111111:root"
    },
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": {
        "sts:ExternalId": "UniqueSecretId123"
      }
    }
  }]
}
```

**Principal options:**
- `"AWS": "arn:aws:iam::111111111111:root"` — Any principal in Account A can assume (if they have permission)
- `"AWS": "arn:aws:iam::111111111111:role/SpecificRole"` — Only that specific role
- `"AWS": "arn:aws:iam::111111111111:user/SpecificUser"` — Only that specific user
- `"Service": "lambda.amazonaws.com"` — AWS service (same-account service role)

#### Identity Policy (Account A — Source)

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "sts:AssumeRole",
    "Resource": "arn:aws:iam::222222222222:role/CrossAccountRole"
  }]
}
```

#### Both Sides Must Allow

```
Access granted when:
  1. Source identity has permission to call sts:AssumeRole on target role ARN
  2. Target role's trust policy allows the source principal
  3. Source account's SCP allows sts:AssumeRole (if in an org)
  4. Target account's SCP allows the actions the role performs (if in an org)
```

#### External ID (Confused Deputy Prevention)

**The confused deputy problem:**
```
Attacker knows your cross-account role ARN →
  Tricks a third-party service into assuming YOUR role →
  Third-party (confused deputy) accesses YOUR resources on attacker's behalf
```

**Solution: External ID**
- A secret value shared between you and the trusted third party
- Added as a condition in the trust policy
- The third party must provide this ID when assuming the role
- Attacker can't guess the External ID → can't exploit the relationship

**When to use External ID:**
- Third-party services assuming roles in YOUR account
- NOT needed for internal cross-account (within your own org)
- NOT a substitute for authentication (it's an anti-confusion measure)

```json
"Condition": {
  "StringEquals": {
    "sts:ExternalId": "customer-specific-secret-123"
  }
}
```

#### Role Chaining

**What it is:** Assuming a role, then from that session, assuming another role.

```
User in Account A → assumes Role-1 in Account B → assumes Role-2 in Account C
```

**Key facts:**
- Each hop creates a new session (previous session's permissions don't carry over)
- Maximum session duration decreases with chaining (1 hour max for chained roles)
- CloudTrail logs each assumption separately
- Used for: Controlled multi-hop access patterns (e.g., jump accounts)

#### Session Duration

| Scenario | Max Duration |
|----------|-------------|
| IAM user assuming role | 1–12 hours (configurable on role) |
| Role chaining (role → role) | 1 hour maximum (cannot extend) |
| SAML/WebIdentity federation | 1–12 hours |
| IAM Identity Center | 1–12 hours (configured in Permission Set) |
| EC2 instance profile | Automatic renewal (no explicit duration) |

---

### Resource-Based Policies

**What they are:** Policies attached directly to resources that define who can access them and what they can do.

#### Services Supporting Resource-Based Policies

| Service | Policy Name | Cross-Account Support |
|---------|------------|----------------------|
| S3 | Bucket Policy | Yes |
| KMS | Key Policy | Yes (required for cross-account) |
| SQS | Queue Policy | Yes |
| SNS | Topic Policy | Yes |
| Lambda | Function Policy | Yes (for invocation) |
| ECR | Repository Policy | Yes |
| Secrets Manager | Resource Policy | Yes |
| API Gateway | Resource Policy | Yes |
| EventBridge | Event Bus Policy | Yes |
| Glacier | Vault Access Policy | Yes |
| Backup | Vault Access Policy | Yes |
| OpenSearch | Domain Access Policy | Yes |
| CodeArtifact | Domain/Repo Policy | Yes |
| MediaStore | Container Policy | Yes |

#### Services NOT Supporting Resource-Based Policies

| Service | Cross-Account Method |
|---------|---------------------|
| DynamoDB | Role assumption required |
| EC2 (instances) | Role assumption required |
| RDS | Role assumption (or snapshot sharing) |
| ECS/EKS | Role assumption required |
| CloudWatch | Role assumption (or cross-account observability) |
| Systems Manager | Role assumption required |
| Step Functions | Role assumption required |

#### Cross-Account Evaluation Differences

**Same-account access:**
```
Resource Policy ALONE can grant access (no identity policy needed)
  OR
Identity Policy ALONE can grant access (no resource policy needed)
Either is sufficient.
```

**Cross-account access (IAM Role):**
```
If resource policy grants access to the ROLE ARN directly:
  → Role can access WITHOUT identity policy granting it
  → But SCP must still allow (if in org)

If resource policy grants access to the ACCOUNT (root):
  → Role ALSO needs identity policy
  → AND SCP must allow
```

**Cross-account access (IAM User):**
```
Always needs BOTH:
  → Resource policy granting access
  → Identity policy granting access
  → SCP must allow
```

#### Key Resource Policy Patterns

**S3 Cross-Account Access:**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "CrossAccountRead",
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::111111111111:role/DataReader"
    },
    "Action": ["s3:GetObject", "s3:ListBucket"],
    "Resource": [
      "arn:aws:s3:::my-bucket",
      "arn:aws:s3:::my-bucket/*"
    ]
  }]
}
```

**KMS Cross-Account Access (required for encrypted resources):**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowCrossAccountDecrypt",
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::111111111111:root"
    },
    "Action": [
      "kms:Decrypt",
      "kms:DescribeKey",
      "kms:GenerateDataKey"
    ],
    "Resource": "*"
  }]
}
```

**SQS Cross-Account (allow SNS to publish):**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Service": "sns.amazonaws.com"
    },
    "Action": "sqs:SendMessage",
    "Resource": "arn:aws:sqs:us-east-1:222222222222:my-queue",
    "Condition": {
      "ArnEquals": {
        "aws:SourceArn": "arn:aws:sns:us-east-1:111111111111:my-topic"
      }
    }
  }]
}
```

---

### AWS Resource Access Manager (RAM)

**What it is:** A service for sharing AWS resources across accounts or within an Organization without duplicating them. The resource stays in the owning account; other accounts get access.

#### How RAM Works

```
┌──────────────────────────────────────────────────────────────────────┐
│                    RAM Resource Sharing                                │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Resource Owner (Account A)         Consumer (Account B)              │
│  ┌─────────────────────┐           ┌─────────────────────┐          │
│  │ Transit Gateway     │           │ Can attach VPCs     │          │
│  │ (actual resource)   │──shared──→│ to the TGW          │          │
│  │                     │           │ (no copy created)   │          │
│  └─────────────────────┘           └─────────────────────┘          │
│                                                                        │
│  RAM Resource Share:                                                   │
│  - Resource(s) to share                                               │
│  - Principals (accounts, OUs, organization)                           │
│  - Permissions (what consumers can do)                                │
│                                                                        │
│  Within Organization (trusted access enabled):                         │
│    → No invitation required (automatic acceptance)                    │
│                                                                        │
│  Outside Organization:                                                 │
│    → Invitation sent → Must be accepted → Then access granted         │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

#### Resources Shareable via RAM

| Resource Type | What Consumer Can Do | Common Use Case |
|--------------|---------------------|-----------------|
| **Transit Gateway** | Attach VPCs, create routes | Centralized networking |
| **Subnets (VPC)** | Launch resources into shared subnet | Shared VPC pattern |
| **Route 53 Resolver Rules** | Associate with their VPCs | Centralized DNS |
| **License Manager Configs** | Use managed licenses | License compliance |
| **Aurora DB Clusters** | Clone the cluster | Cross-account DB cloning |
| **CodeBuild Projects** | View build results | Shared CI/CD |
| **EC2 Capacity Reservations** | Launch instances using reservation | Capacity sharing |
| **AWS Outposts** | Use Outpost resources | Shared on-premises extension |
| **Resource Groups** | View group members | Organized visibility |
| **EC2 Image Builder** | Use image pipelines | Shared AMI building |
| **Glue (Data Catalog)** | Access databases/tables | Shared data catalog |
| **Network Firewall Policies** | Use firewall policies | Centralized security |
| **Prefix Lists** | Reference in security groups/routes | Centralized IP management |
| **Systems Manager Incident Manager** | Use response plans | Shared incident management |
| **Private Certificate Authority** | Issue certificates | Centralized PKI |
| **Verified Access** | Use access policies | Shared zero-trust access |
| **IP Address Manager (IPAM) Pools** | Allocate CIDRs from pool | Centralized IP planning |

#### RAM Within Organizations

**Trusted access enabled:**
- Sharing within the organization requires NO invitation acceptance
- Shares are automatically accepted by member accounts
- Can share to: Entire organization, specific OUs, specific accounts
- Management account enables this: `organizations:EnableAWSServiceAccess` for RAM

**Trusted access NOT enabled:**
- Even within the org, invitations must be accepted
- Behaves like sharing with external accounts
- **Exam trap:** Sharing within org without trusted access = invitations required

#### RAM Permissions

**Managed permissions:**
- RAM defines what consumers can DO with shared resources
- Each resource type has default managed permissions
- You can create custom managed permissions (restrict further)
- Example: Share TGW with "Allow attachment" permission (consumer can attach VPCs but can't modify TGW)

**Key principle:** Resource OWNER retains full control. Consumers get only the permissions defined in the RAM share.

#### RAM vs Resource-Based Policies

| Dimension | RAM | Resource-Based Policies |
|-----------|-----|------------------------|
| **Resource stays in** | Owner's account | Owner's account |
| **Mechanism** | RAM service orchestrates | Policy on resource |
| **Supported resources** | Specific resource types (networking, compute) | S3, KMS, SQS, SNS, Lambda, etc. |
| **Org integration** | Native (no invitation within org) | Manual (specify account IDs) |
| **Permissions model** | Managed permissions (pre-defined actions) | Full IAM policy language |
| **Consumer sees resource** | In their console (as if theirs) | Accessed via ARN |
| **Best for** | Infrastructure sharing (TGW, subnets, DNS) | Data/service sharing (S3, queues, keys) |
| **Exam tip** | "share Transit Gateway", "shared VPC" | "bucket policy allows Account B" |

---

### Shared VPC Pattern (VPC Sharing via RAM)

**What it is:** Share subnets from a central VPC (Network account) with other accounts. Resources in those accounts launch into the shared subnets.

#### Architecture

```
┌───────────────────────────────────────────────────────────────────┐
│                     Shared VPC Pattern                              │
├───────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Network Account (Owner)                                            │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ VPC (10.0.0.0/16)                                           │  │
│  │ ├── Subnet-A (10.0.1.0/24) ──shared via RAM──→ Account B   │  │
│  │ ├── Subnet-B (10.0.2.0/24) ──shared via RAM──→ Account C   │  │
│  │ ├── Subnet-C (10.0.3.0/24) ──shared via RAM──→ Account B+C │  │
│  │ │                                                           │  │
│  │ │ Network Account manages:                                  │  │
│  │ │   Route tables, NACLs, Internet Gateway, NAT Gateway,    │  │
│  │ │   VPC Flow Logs, Transit Gateway attachment               │  │
│  │ │                                                           │  │
│  │ │ Consumer Accounts manage:                                 │  │
│  │ │   Their own resources IN shared subnets                   │  │
│  │ │   Their own Security Groups                               │  │
│  │ │   Their own ENIs                                          │  │
│  │ └───────────────────────────────────────────────────────────│  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
└───────────────────────────────────────────────────────────────────┘
```

#### What Owner Controls vs Consumer Controls

| Owner (Network Account) | Consumer (Workload Accounts) |
|-------------------------|-------------------------------|
| VPC, Subnets, CIDR | Resources launched in shared subnets |
| Route tables | Security Groups (own) |
| NACLs | ENIs |
| Internet Gateway | EC2 instances, RDS, Lambda (in subnet) |
| NAT Gateway | Load Balancers (in shared subnet) |
| VPC Flow Logs | ECS tasks |
| Transit Gateway attachment | Application-level resources |
| VPN/Direct Connect | Their own CloudWatch metrics |
| VPC Endpoints (shared with consumers) | |

#### Benefits of Shared VPC

- **Fewer VPCs** = fewer NAT Gateways, fewer VPC Endpoints, lower cost
- **Centralized network management** (Network team owns VPC, app teams use it)
- **Reduced IP address waste** (one VPC CIDR shared, not duplicated)
- **Simplified connectivity** (no peering/TGW needed for accounts sharing a VPC)
- **Security group cross-referencing** (SGs in same VPC can reference each other)

#### Limitations

- Consumer cannot modify VPC-level resources (route tables, NACLs, etc.)
- Consumer cannot create VPC-level resources (subnets, gateways)
- Consumer's resources in shared subnet use the owner's route table
- Deletion: Owner can unshare subnet → consumers must move resources first
- Some services don't support shared subnets (check service docs)

---

### Service-Specific Cross-Account Sharing

#### AMI Sharing

```
Account A (owner) modifies AMI permissions →
  Add Account B's ID →
  Account B can launch EC2 instances from the AMI

Key facts:
- AMI remains in owner's account
- Consumer can COPY the AMI to their account (for independence)
- Encrypted AMIs: Must also share the KMS key
- Can share to: Specific accounts, organizations, public
```

#### EBS Snapshot Sharing

```
Account A modifies snapshot permissions →
  Add Account B's ID →
  Account B can create volumes from the snapshot

Key facts:
- Encrypted snapshots: Must share KMS key AND grant kms:CreateGrant
- Consumer can copy snapshot to their account (with their own KMS key)
- Cross-region: Copy snapshot to target region first, then share
```

#### RDS Snapshot Sharing

```
Account A shares RDS snapshot →
  Account B can restore a new DB instance from it

Key facts:
- Manual snapshots only (not automated)
- Encrypted: Share KMS key
- Can share with up to 20 AWS accounts
- Aurora: Can share cluster snapshots
```

#### Cross-Account EventBridge

```
Account A's Event Bus has resource policy allowing Account B →
  Account B creates rules targeting Account A's bus →
  Events flow cross-account

Use case: Centralized event bus for organization-wide events
```

#### Cross-Account CloudWatch

**CloudWatch Cross-Account Observability:**
- Source accounts share metrics, logs, and traces with a monitoring account
- Uses Organization-level setup or individual account links
- Monitoring account sees dashboards from all source accounts
- No data duplication (live access to source account data)

---

### AWS PrivateLink (Interface VPC Endpoints for Cross-Account)

**What it is:** Expose a service privately from one VPC to another (same or cross-account) via ENIs, without internet exposure.

```
Provider Account:                    Consumer Account:
  NLB → Target Groups               VPC Endpoint (Interface) →
  ↓                                    ENI in consumer's VPC →
  VPC Endpoint Service                 Private DNS resolves to ENI
  (service name)
```

**Cross-account PrivateLink:**
- Provider creates VPC Endpoint Service (backed by NLB or GWLB)
- Consumer creates Interface VPC Endpoint pointing to the service
- Provider must ALLOW the consumer's account (allowlist)
- Traffic stays on AWS private network (no internet)
- **Use case:** SaaS providers exposing services privately to customers

---

### Key Quotas & Limits

| Resource | Limit |
|----------|-------|
| RAM resource shares per account | 5,000 |
| Principals per resource share | 1,000 |
| Resources per resource share | Varies by type |
| Cross-account role assumption session | 1–12 hours (1 hour if chained) |
| S3 bucket policy size | 20 KB |
| KMS key policy size | 32 KB |
| IAM trust policy size | 2,048 characters |
| RAM shared resources per account | Varies by resource type |
| VPC sharing: subnets shared per VPC | All subnets can be shared |
| AMI sharing: accounts per AMI | Unlimited (or public) |
| RDS snapshot sharing | 20 accounts |

---

## 2. Design Patterns & Best Practices

### Cross-Account Access Patterns

#### Pattern 1: Centralized Networking (RAM + Transit Gateway)

```
Network Account:
  Transit Gateway → Shared via RAM to all workload accounts
  VPC Endpoints → Centralized (shared services VPC)
  Route 53 Resolver Rules → Shared via RAM

All Accounts:
  Create TGW attachments (VPC → TGW)
  Route traffic through Network Account inspection VPC
  Use centralized DNS resolution
```

#### Pattern 2: Hub-and-Spoke Security

```
Security Account:
  GuardDuty (delegated admin) → Receives findings from all accounts
  Security Hub (delegated admin) → Aggregates compliance
  CloudTrail → Organization trail → Log Archive

Workload Accounts:
  Assume read-only roles in Security Account for dashboards
  Security Account assumes investigation roles in Workload Accounts
```

#### Pattern 3: CI/CD Cross-Account Deployment

```
Shared Services Account:
  CodePipeline → CodeBuild →
    Assume deployment role in Dev Account → Deploy
    Assume deployment role in Staging Account → Deploy
    Assume deployment role in Prod Account → Deploy

Each target account:
  Deployment Role with trust policy allowing Shared Services
  Permissions: CloudFormation, ECS, Lambda, S3 (deployment-scoped)
```

#### Pattern 4: Shared Data Lake

```
Data Lake Account:
  S3 buckets → Bucket policies allowing read from analytics accounts
  KMS keys → Key policies allowing decrypt from analytics accounts
  Glue Data Catalog → Shared via RAM (or Lake Formation)
  Lake Formation → Cross-account grants (fine-grained table/column access)

Analytics Accounts:
  Athena queries S3 in Data Lake Account
  EMR processes data cross-account
  QuickSight visualizes cross-account datasets
```

#### Pattern 5: Cross-Account Backup

```
Backup Account:
  AWS Backup vault → Cross-account vault copy policy
  
Workload Accounts:
  AWS Backup → Copy to Backup Account vault
  Organization Backup Policy → Enforces cross-account copy

Result: Backups survive workload account compromise
```

### Anti-Patterns

| Anti-Pattern | Why It's Wrong | Better Approach |
|--------------|---------------|-----------------|
| Long-term access keys for cross-account | Credentials can leak, no expiry | Role assumption (temporary credentials) |
| Wildcard principal in trust policy (`"*"`) | Anyone can assume your role | Specify account IDs or org conditions |
| Sharing KMS key without scoping actions | Over-permissive decryption access | Limit to specific actions (Decrypt, not admin) |
| VPC Peering for 10+ accounts | N² connections, non-transitive | Transit Gateway (hub-and-spoke) |
| Duplicating resources in every account | Waste, inconsistency, cost | RAM sharing (single resource, multiple consumers) |
| Cross-account root user access | No audit trail, maximum permissions | Named roles with specific permissions |
| Not using External ID for third parties | Confused deputy vulnerability | External ID in trust policy condition |
| Object ownership issues (S3 cross-account) | Uploader owns object, bucket owner can't access | Bucket Owner Enforced or bucket-owner-full-control ACL |
| Sharing encrypted resource without key access | Consumer gets "Access Denied" on decrypt | Share KMS key policy alongside resource |
| Unscoped iam:PassRole cross-account | Privilege escalation risk | Restrict PassRole to specific role ARNs |

### Well-Architected Framework Alignment

**Security:**
- Least privilege: Scoped cross-account roles with minimal permissions
- Temporary credentials: Role assumption (no long-term keys)
- External ID: Prevent confused deputy attacks
- SCP enforcement: Cross-account actions still bounded by SCPs
- Audit: CloudTrail logs all cross-account role assumptions

**Cost:**
- RAM sharing: One resource, pay once (vs. duplicating per account)
- Shared VPC: Fewer NAT Gateways, fewer VPC Endpoints
- Centralized Transit Gateway: One TGW vs. N² peering connections
- Cross-account backup: Cheaper than building backup infra per account

**Reliability:**
- Shared networking: Consistent, centrally managed connectivity
- Cross-account DR: Backups in separate accounts survive account compromise
- Service quotas: Shared resources don't duplicate quota consumption

**Operational Excellence:**
- Centralized management: Network team manages TGW, security team manages GuardDuty
- Standardized access patterns: Consistent role naming, trust policies
- Automation: CloudFormation StackSets deploy cross-account roles uniformly

---

## 3. Security & Compliance

### Cross-Account Security Best Practices

#### Trust Policy Hardening

**Bad — Too permissive:**
```json
{
  "Principal": { "AWS": "*" }
}
```

**Better — Account-level:**
```json
{
  "Principal": { "AWS": "arn:aws:iam::111111111111:root" }
}
```

**Best — Role-specific:**
```json
{
  "Principal": { "AWS": "arn:aws:iam::111111111111:role/SpecificRole" }
}
```

**Best with conditions:**
```json
{
  "Principal": { "AWS": "arn:aws:iam::111111111111:role/SpecificRole" },
  "Condition": {
    "StringEquals": { "aws:PrincipalOrgID": "o-myorgid" },
    "Bool": { "aws:MultiFactorAuthPresent": "true" }
  }
}
```

#### SCP Interaction with Cross-Account Access

```
Account A (source, member) → assumes role in Account B (target, member)

Evaluation:
1. Account A's SCP must allow sts:AssumeRole
2. Account A's identity policy must allow sts:AssumeRole on target role
3. Account B's trust policy must allow Account A's principal
4. Account B's SCP must allow the actions the role performs
5. Target role's identity policy determines what actions are possible

Key: BOTH accounts' SCPs apply — source for the assumption, target for the actions
```

#### KMS Cross-Account Security

**Critical for encrypted resources (S3, EBS, RDS, Secrets Manager):**

```json
// KMS Key Policy in Account B (key owner)
{
  "Sid": "AllowCrossAccountUse",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::111111111111:role/DataReader"
  },
  "Action": [
    "kms:Decrypt",
    "kms:DescribeKey"
  ],
  "Resource": "*"
}
```

**Plus** — Account A's role needs IAM policy allowing kms:Decrypt on the key ARN.

**Common mistake:** Sharing an encrypted S3 bucket without sharing KMS key access → "AccessDenied" errors that seem like S3 permission issues but are actually KMS permission issues.

#### S3 Object Ownership (Cross-Account Writes)

**The problem:** When Account A puts objects in Account B's bucket, Account A OWNS those objects by default. Account B (bucket owner) can't read or manage them.

**Solutions:**

1. **Bucket Owner Enforced (recommended):**
```json
// Bucket ownership controls: BucketOwnerEnforced
// Disables ACLs entirely
// All objects owned by bucket owner regardless of uploader
```

2. **bucket-owner-full-control ACL:**
```json
// Require uploader to include ACL granting bucket owner control
{
  "Condition": {
    "StringEquals": {
      "s3:x-amz-acl": "bucket-owner-full-control"
    }
  }
}
```

3. **Cross-account replication (CRR):**
```
Objects replicated cross-account → destination owns the copies
```

#### Confused Deputy Prevention

| Scenario | Solution |
|----------|----------|
| Third-party service assumes your role | External ID condition |
| Cross-service access (service-to-service) | `aws:SourceArn` / `aws:SourceAccount` condition |
| Organization-only access | `aws:PrincipalOrgID` condition |
| Specific VPC only | `aws:SourceVpc` / `aws:SourceVpce` condition |

**Example — Prevent confused deputy with third-party:**
```json
{
  "Effect": "Allow",
  "Principal": { "AWS": "arn:aws:iam::THIRD_PARTY_ACCOUNT:root" },
  "Action": "sts:AssumeRole",
  "Condition": {
    "StringEquals": {
      "sts:ExternalId": "your-unique-id-from-third-party"
    }
  }
}
```

#### IAM Access Analyzer

**What it does for cross-account:** Identifies resources with policies that grant access to external principals (outside your account or organization).

**Types:**
- **External access analyzer:** Finds resources accessible from outside your zone of trust (account or org)
- **Unused access analyzer:** Finds unused permissions in roles (including cross-account roles)

**Findings examples:**
- S3 bucket with policy allowing access from unknown account
- KMS key with policy granting decrypt to external principal
- IAM role with trust policy allowing external assumption
- SQS queue with policy allowing external SendMessage

---

## 4. Cost Optimization

### Cost Impact of Cross-Account Patterns

#### Data Transfer Costs

| Pattern | Data Transfer Cost |
|---------|-------------------|
| Cross-account, same region, same AZ | Free (via VPC Peering or TGW) |
| Cross-account, same region, different AZ | $0.01/GB each direction |
| Cross-account, different region | Standard cross-region rates ($0.02/GB) |
| Transit Gateway data processing | $0.02/GB (on top of transfer) |
| VPC Peering (same region) | Free (same AZ) or $0.01/GB (cross-AZ) |
| PrivateLink (Interface Endpoint) | $0.01/GB + $0.01/hr per AZ |
| S3 cross-account (same region) | Free (no transfer charge) |

#### Cost Optimization Strategies

**RAM Sharing (avoid duplication):**

| Without RAM | With RAM | Savings |
|-------------|----------|---------|
| 10 Transit Gateways (one per account) | 1 TGW shared via RAM | ~90% on TGW costs |
| 10 sets of VPC Endpoints | 1 set in shared VPC | ~90% on endpoint costs |
| 10 NAT Gateways | 1-2 NATs (shared VPC) | ~80-90% on NAT costs |
| Duplicate AMIs per account | 1 AMI shared | Storage savings |

**Shared VPC cost analysis:**
```
Without Shared VPC (10 accounts, 2 AZs each):
  NAT Gateways: 10 × 2 = 20 × $0.045/hr = $648/month
  VPC Endpoints (5 per VPC): 10 × 5 × 2 AZ × $0.01/hr = $720/month
  Total: $1,368/month

With Shared VPC (1 VPC, 2 AZs):
  NAT Gateways: 1 × 2 = 2 × $0.045/hr = $64.80/month
  VPC Endpoints: 1 × 5 × 2 AZ × $0.01/hr = $72/month
  Total: $136.80/month

Savings: ~$1,231/month (90%)
```

**Transit Gateway vs VPC Peering:**
```
10 accounts needing full mesh connectivity:
  VPC Peering: 10×9/2 = 45 peering connections (free, but unmanageable)
  Transit Gateway: 1 TGW + 10 attachments
    Cost: 10 × $0.05/hr attachment + $0.02/GB data = $360/month (base) + data

Decision point: >3-4 accounts needing connectivity → TGW is operationally better
  despite per-GB charge
```

### Cost Traps

- **Transit Gateway data processing:** $0.02/GB in EACH direction. High-throughput cross-account traffic adds up fast
- **VPC Endpoint per account:** If not using shared VPC, each account needs its own endpoints ($0.01/hr/AZ each)
- **Cross-region replication charges:** S3 CRR, RDS cross-region read replicas — both transfer + storage
- **Forgotten cross-account roles:** Roles that exist but aren't used still consume policy evaluation (negligible but audit burden)
- **Encrypted cross-account:** KMS calls for each decrypt operation ($.03 per 10,000 requests)
- **PrivateLink at scale:** $0.01/hr per AZ × many services × many AZs = significant
- **NAT Gateway data processing:** $0.045/GB — if cross-account traffic routes through NAT, expensive

---

## 5. High Availability, Disaster Recovery & Resilience

### Cross-Account DR Patterns

#### Pattern 1: Cross-Account Backup Vault

```
Production Account (us-east-1):
  AWS Backup → Daily backups → Local vault
  AWS Backup → Cross-account copy → Backup Account vault

Backup Account (us-east-1 + eu-west-1):
  Vault in us-east-1 (copy from prod)
  Vault in eu-west-1 (cross-region copy for DR)

DR Scenario:
  Prod account compromised → Restore from Backup Account vault
  us-east-1 down → Restore from eu-west-1 vault in Backup Account
```

#### Pattern 2: Cross-Account Replication

```
Active Account (us-east-1):
  S3 → CRR → DR Account (eu-west-1)
  RDS → Cross-region read replica → DR Account (via snapshot sharing)
  DynamoDB → Global Tables (multi-region, multi-account not native — use backup)
  
Failover:
  Route 53 failover → DR Account endpoint
  Promote read replica to primary
```

#### Pattern 3: Multi-Account Active-Active

```
Account A (us-east-1):          Account B (eu-west-1):
  Application stack               Application stack (identical)
  DynamoDB Global Table            DynamoDB Global Table
  S3 (CRR bi-directional)         S3 (CRR bi-directional)

Route 53 latency-based routing → Closest region
Both accounts fully independent (survive each other's failure)
```

### Resilience of Cross-Account Mechanisms

| Mechanism | Resilience | Failure Mode |
|-----------|-----------|--------------|
| IAM Role Assumption (STS) | Highly available (global) | STS regional endpoint failure → use global endpoint |
| RAM Shared Resources | Depends on underlying resource | If TGW goes down, all consumers affected |
| VPC Peering | Regional, highly available | Regional failure affects peering |
| Transit Gateway | Regional, multi-AZ | Regional failure affects TGW |
| Resource-Based Policies | Same as resource | If S3 goes down, policy is irrelevant |
| PrivateLink | Multi-AZ (per endpoint) | AZ failure → remaining AZs serve |
| Organizations (SCPs) | Global, highly available | Extremely rare failure |

### RPO/RTO with Cross-Account

| Strategy | RPO | RTO | Cross-Account Component |
|----------|-----|-----|------------------------|
| S3 CRR cross-account | Minutes (async replication) | Minutes (data available) | S3 bucket policy granting DR account |
| RDS cross-account snapshot | Hours (snapshot frequency) | Hours (restore time) | KMS key sharing + snapshot permissions |
| DynamoDB Global Tables | Seconds (sync replication) | Seconds (automatic) | Same account multi-region (not cross-account) |
| AWS Backup cross-account | Hours (backup frequency) | Hours (restore time) | Backup vault access policy |
| EBS snapshot cross-account | Hours | Minutes (create volume) | Snapshot permission + KMS sharing |

---

## 6. Exam-Focused Section

### Straightforward Questions

**Q1:** A company has a central Network account with a Transit Gateway. They need to share it with 50 workload accounts in their Organization without each account accepting an invitation. What service and configuration enables this?

**A:** AWS Resource Access Manager (RAM) with Organizations trusted access enabled. Create a RAM resource share containing the Transit Gateway, share it with the entire organization or specific OUs. With trusted access enabled, member accounts automatically receive access without invitation acceptance.

---

**Q2:** A CI/CD pipeline in a Shared Services account needs to deploy CloudFormation stacks to a Production account. What's the secure cross-account mechanism?

**A:** Create a **deployment IAM role** in the Production account with a trust policy allowing the Shared Services account's pipeline role to assume it. The role has permissions for CloudFormation and the resources it needs to create. The pipeline calls `sts:AssumeRole`, receives temporary credentials, and deploys. Both accounts' SCPs must allow the relevant actions.

---

**Q3:** Account A stores sensitive data in an S3 bucket encrypted with a KMS CMK. Account B needs to read this data. What permissions are required?

**A:** Three things needed: (1) **S3 bucket policy** allowing Account B's role `s3:GetObject`. (2) **KMS key policy** allowing Account B's role `kms:Decrypt` and `kms:DescribeKey`. (3) **Account B's role identity policy** allowing `s3:GetObject` on the bucket and `kms:Decrypt` on the key ARN. If either the S3 or KMS permission is missing → Access Denied.

---

**Q4:** A company wants to reduce NAT Gateway costs across their 20-account environment. Currently each account has its own VPC with NAT Gateways. What architectural change reduces cost?

**A:** Implement the **Shared VPC pattern** using RAM. Create a central VPC in the Network account with shared subnets and NAT Gateways. Share subnets via RAM to workload accounts. Workload accounts launch resources into shared subnets, using the central NAT Gateways. Reduces from 20+ NAT Gateways to 2-3 (multi-AZ).

---

**Q5:** A third-party monitoring service needs to assume a role in a customer's AWS account to collect metrics. How should the customer protect against confused deputy attacks?

**A:** Add an **External ID condition** to the IAM role's trust policy. The monitoring service provides a unique External ID per customer. The trust policy requires this specific External ID in the `sts:ExternalId` condition. This prevents other customers of the same monitoring service from tricking it into accessing this customer's resources.

---

**Q6:** An organization uses centralized DNS in a Network account (Route 53 Private Hosted Zones and Resolver Rules). How do they make these available to all workload accounts?

**A:** Share **Route 53 Resolver Rules** via RAM to workload accounts or the organization. Consumer accounts associate the shared resolver rules with their VPCs. For Private Hosted Zones: Associate the PHZ with VPCs in other accounts (using `CreateVPCAssociationAuthorization` + `AssociateVPCWithHostedZone`). RAM handles resolver rules; direct API handles PHZ association.

---

**Q7:** A developer in Account A puts objects into Account B's S3 bucket. Account B's admin discovers they can't access these objects. What's the issue and the fix?

**A:** **Object ownership.** By default, the uploader (Account A) owns the objects. Account B (bucket owner) has no access to objects owned by other accounts. **Fix:** Enable **Bucket Owner Enforced** object ownership on the bucket. This disables ACLs and makes the bucket owner the automatic owner of all objects regardless of who uploaded them. Alternatively, require uploaders to set `bucket-owner-full-control` ACL.

---

### Tricky / Scenario-Based Questions

**Q1:** A company has an SCP on their Production OU that denies `sts:AssumeRole` for any role not matching `arn:aws:iam::*:role/Approved-*`. A DevOps engineer in the Shared Services account tries to assume `ProductionDeployRole` in a production account. The trust policy allows the engineer's role. Why does it fail?

**A:** The SCP denies `sts:AssumeRole` where the target role doesn't match `Approved-*`. **However**, this SCP is on the Production OU — it affects principals IN production accounts (not principals assuming INTO production accounts). **Wait — re-read:** The SCP restricts what principals in Production accounts can do. The assumption call originates from the Shared Services account (different OU). So the Production OU's SCP doesn't block the CALLER. **But:** The actions performed AFTER assumption (inside the production account) ARE subject to the Production OU's SCP. The role `ProductionDeployRole` IS in the production account, and any actions it performs are bounded by the Production SCP. **The actual problem:** If `ProductionDeployRole` doesn't match `Approved-*`, and the SCP is written to deny actions by roles not matching that pattern — then actions BY that role would be denied. **Key concept:** SCPs on the target account affect what the assumed role can DO, not whether it can BE assumed. Assumption is controlled by: source account's SCP + source identity policy + target trust policy.

---

**Q2:** A company shares a Transit Gateway via RAM with their entire organization. A new account is created and enrolled in Control Tower. The account's VPC can't attach to the Transit Gateway. The RAM share shows the organization as the principal. What's wrong?

**A:** Likely causes: (1) The new account might not yet be fully propagated in Organizations (eventual consistency). Wait a few minutes and retry. (2) RAM trusted access might not be enabled (`ram:EnableSharingWithAwsOrganization`). Without it, even org-wide sharing requires invitation acceptance. (3) The account might be in an OU that's excluded from the share (if shared to specific OUs, not the whole org). (4) The account's SCP might deny `ec2:CreateTransitGatewayVpcAttachment`. (5) The account needs IAM permissions to create the TGW attachment. **Most likely exam answer:** RAM trusted access not enabled, OR the account is in an OU not included in the share scope.

---

**Q3:** A company uses cross-account role assumption. An audit reveals a role in Account B with a trust policy allowing `"Principal": {"AWS": "arn:aws:iam::111111111111:root"}`. The security team says this means the root user of Account A can access Account B. Is that correct?

**A:** **Misleading but technically not exactly that.** The `:root` principal means "any authenticated IAM principal in account 111111111111" — it does NOT specifically mean only the root user. Any user, role, or the root user in Account A can assume this role IF they have `sts:AssumeRole` permission in their identity policy. **The root user** of Account A inherently has all permissions (including AssumeRole), so yes, root CAN assume it. But so can any other principal with appropriate IAM permissions. **Key concept:** `:root` in trust policies means "the account" (any principal in it), NOT specifically the root user. It's an account-level trust delegation.

---

**Q4:** A company uses RAM to share subnets from a Network account to a Production account. An application team launches an RDS instance in the shared subnet. Later, the Network team unshares the subnet from RAM. What happens to the RDS instance?

**A:** **The RDS instance continues to run.** Unsharing a subnet via RAM does NOT affect resources already launched in that subnet. The resources remain and function normally. **However:** The Production account can no longer launch NEW resources in that subnet. Existing resources keep their network connectivity (ENIs remain attached). **To fully remove:** You'd need to stop/delete the resources first, then unshare. **Key concept:** RAM unsharing is not destructive to existing resources. It only prevents new resource creation in the shared resource.

---

**Q5:** A Lambda function in Account A needs to process messages from an SQS queue in Account B. The queue is encrypted with a CMK. The Lambda's execution role has the correct SQS and KMS permissions in its identity policy. The SQS queue policy allows Account A. But messages aren't being processed. What's missing?

**A:** **KMS key policy must explicitly allow Account A.** The SQS queue policy grants SQS access, but to decrypt messages, the Lambda needs `kms:Decrypt` on the CMK — and the CMK's key policy must allow Account A's role. KMS key policies are the authoritative control for KMS keys (unlike most resources where identity policy alone can work within the same account). Cross-account KMS ALWAYS requires the key policy to grant access. **Fix:** Add Account A's Lambda execution role to the KMS key policy with `kms:Decrypt` and `kms:GenerateDataKey`. **Key concept:** For cross-account access to encrypted resources, you always need THREE permissions: (1) Resource policy (SQS), (2) KMS key policy, (3) Caller's identity policy (for both SQS and KMS).

---

**Q6:** A company implements cross-account deployment using role assumption. The deployment role in Production has `AdministratorAccess`. A security audit flags this as high risk. What's the least-privilege alternative?

**A:** (1) **Scope the deployment role permissions** to only what's needed: `cloudformation:*`, `s3:GetObject` (artifacts bucket), and specific service permissions for resources being deployed (ecs, lambda, etc.). (2) **Add resource-level restrictions:** Limit to specific CloudFormation stack names, specific S3 bucket, specific Lambda functions. (3) **Add conditions:** `aws:RequestedRegion` (limit to deployment regions), `aws:CalledVia` (only via CloudFormation, not direct). (4) **Use Permission Boundary** on the deployment role to cap maximum permissions. (5) **Limit the trust policy** to only the specific pipeline role (not the entire account). **Key concept:** Cross-account deployment roles should follow least privilege — only the minimum permissions for the deployment to succeed.

---

**Q7:** An organization uses VPC Peering between Account A and Account B. They need to add Account C. Currently A↔B works. After adding A↔C and B↔C peering, traffic from A to C works, but B's resources can't reach C through A (they try routing via A). Why?

**A:** **VPC Peering is non-transitive.** Traffic from B cannot route through A's peering connection to reach C. Each VPC peering is a direct 1:1 connection. Even though A↔B and A↔C exist, B can't reach C via A. B must use its own direct B↔C peering connection. **If B↔C peering exists but routing fails:** Check route tables in B's VPC — they need a route for C's CIDR pointing to the B↔C peering connection (not to A↔B peering). **Fix for complex connectivity:** Use Transit Gateway instead (transitive routing). **Key concept:** VPC Peering = non-transitive. Transit Gateway = transitive. This is an extremely common exam question.

---

### Common Exam Traps & Pitfalls

1. **VPC Peering is non-transitive.** A↔B and B↔C does NOT mean A↔C. Each needs direct peering. If >3 accounts need full mesh → Transit Gateway.

2. **RAM within org needs trusted access.** Without `EnableSharingWithAwsOrganization`, sharing within the org still requires invitation acceptance.

3. **S3 object ownership.** Cross-account uploads create objects owned by uploader, not bucket owner. Use Bucket Owner Enforced.

4. **KMS is always required for encrypted cross-account.** Sharing encrypted S3/EBS/RDS without sharing KMS key = Access Denied.

5. **`:root` in trust policy ≠ root user only.** It means "any principal in that account." It's an account-level trust.

6. **External ID is for third parties, NOT internal.** Within your own org, you don't need External ID. It prevents confused deputy with EXTERNAL services.

7. **Cross-account: SCPs apply on BOTH sides.** Source SCP restricts the assumption call. Target SCP restricts what the role can do.

8. **Role chaining max = 1 hour.** If role A assumes role B assumes role C, the final session is capped at 1 hour regardless of role settings.

9. **RAM unsharing doesn't destroy existing resources.** Consumer's existing resources in shared subnet keep running.

10. **PrivateLink is unidirectional.** Consumer → Provider only. Provider can't initiate connections to consumer.

11. **Transit Gateway has per-GB charges.** VPC Peering (same region) is free for data transfer. TGW charges $0.02/GB — significant at volume.

12. **DynamoDB doesn't have resource-based policies.** Cross-account DynamoDB access ALWAYS requires role assumption.

---

## 7. Cheat Sheet

### Must-Know Facts

| Fact | Detail |
|------|--------|
| Cross-account role assumption | Requires: source identity policy + target trust policy + both SCPs |
| External ID purpose | Prevent confused deputy (third-party access only) |
| Role chaining max duration | 1 hour |
| RAM within org (trusted access) | No invitation needed |
| RAM without trusted access | Invitation required, even within org |
| VPC Peering | Non-transitive, direct 1:1 |
| Transit Gateway | Transitive, hub-and-spoke, $0.02/GB |
| S3 object ownership default | Uploader owns (not bucket owner) |
| Bucket Owner Enforced | Fixes cross-account ownership |
| KMS cross-account | Key policy MUST allow (always required) |
| Resource-based policy (same account) | Can grant access alone (no identity policy needed) |
| Resource-based policy (cross-account, role) | Can grant access alone (role doesn't need identity policy) |
| Resource-based policy (cross-account, user) | User STILL needs identity policy |
| PrivateLink direction | Consumer → Provider only (not bidirectional) |
| Shared VPC (RAM) | Owner manages VPC, consumers launch resources |
| IAM Access Analyzer | Detects external access in resource policies |

### Cross-Account Method Selection

| Scenario | Method |
|----------|--------|
| General cross-account access (any service) | IAM role assumption |
| Share S3 bucket access | Bucket policy (resource-based) |
| Share KMS key | Key policy (always required for cross-account KMS) |
| Share Transit Gateway | RAM |
| Share subnets (Shared VPC) | RAM |
| Share DNS resolver rules | RAM |
| Share AMI | AMI launch permissions |
| Share EBS/RDS snapshot | Snapshot permission modification |
| Expose private service | PrivateLink (VPC Endpoint Service) |
| Connect VPCs (few) | VPC Peering |
| Connect VPCs (many) | Transit Gateway |
| Centralized event bus | EventBridge resource policy |
| Third-party service access | Role assumption + External ID |
| Cross-account monitoring | CloudWatch cross-account observability |
| Fine-grained data access | Lake Formation cross-account grants |

### Decision Flowchart

```
Question mentions...                                        → Think...
──────────────────────────────────────────────────────────────────────────
"share Transit Gateway / subnets / resolver rules"         → RAM
"bucket policy allows another account"                     → Resource-based policy
"assume role in another account"                           → sts:AssumeRole + trust policy
"encrypted resource cross-account"                         → MUST share KMS key policy too
"third-party service accessing our account"                → External ID in trust policy
"connect VPCs, few accounts"                               → VPC Peering (non-transitive)
"connect VPCs, many accounts"                              → Transit Gateway (transitive)
"uploader owns objects in our bucket"                      → Bucket Owner Enforced
"expose service privately cross-account"                   → PrivateLink
"reduce NAT/endpoint costs across accounts"                → Shared VPC via RAM
"non-transitive routing problem"                           → VPC Peering limitation → use TGW
"invitation required within org"                           → RAM trusted access not enabled
"temporary credentials, cross-account"                     → STS AssumeRole
"role assumption max 1 hour"                               → Role chaining
"detect external access to resources"                      → IAM Access Analyzer
"centralized data access, column-level"                    → Lake Formation grants
```

---

## 8. Deep Dive: 10 More Tricky Scenario-Based Questions

**Q1:** A company has Account A (us-east-1) replicating S3 objects via CRR to Account B (eu-west-1). The objects are encrypted with Account A's KMS CMK. Replication fails with Access Denied. What's wrong?

**A:** For cross-account CRR with KMS-encrypted objects, the replication requires: (1) Account A's replication role must have `kms:Decrypt` on Account A's key (to read source objects). (2) Account A's replication role must have `kms:Encrypt` on Account B's KEY (to write encrypted objects in Account B). (3) Account B's bucket policy must allow the replication role to `s3:ReplicateObject`. (4) **Critically:** You must specify a DESTINATION KMS key in Account B for re-encryption. You can't replicate with Account A's key into Account B's bucket — you need Account B's key. (5) Account B's KMS key policy must allow Account A's replication role to encrypt. **Key concept:** Cross-account CRR with KMS requires decryption at source (source key) + re-encryption at destination (destination key). The replication role needs access to BOTH keys.

---

**Q2:** A company uses RAM to share a Transit Gateway with their organization. A workload account creates a TGW attachment (VPC → TGW). But traffic from the VPC can't reach other VPCs attached to the same TGW. The TGW route table shows the attachment. What's missing?

**A:** **Transit Gateway route table associations and routes.** Creating an attachment doesn't automatically enable routing. (1) The attachment must be **associated** with a TGW route table. (2) The TGW route table must have **routes** pointing to the new attachment's CIDR. (3) If using separate route tables (segmentation), the attachment might be associated with a route table that doesn't have routes to other VPCs. (4) The VPC's route table must have a route for destination CIDRs pointing to the TGW attachment. **Key concept:** TGW attachment creation ≠ automatic routing. You need: attachment → association → propagation/static routes (TGW side) + VPC route table entries (VPC side).

---

**Q3:** A company has a centralized Security account that assumes roles in workload accounts for incident investigation. During an incident, the security team assumes `InvestigationRole` in the compromised account. The attacker has modified the trust policy of `InvestigationRole` to remove the Security account. What can the security team do?

**A:** (1) **Use the management account** — assume `OrganizationAccountAccessRole` (created when account was created within org). This role has full admin and its trust policy can only be modified by the management account. (2) **If that role was also compromised:** Attach an SCP denying all actions by the attacker's principal, then use management account access to regain control. (3) **Prevention for the future:** SCP that denies modification of trust policies for investigation/audit roles (deny `iam:UpdateAssumeRolePolicy` on specific role ARNs). **Key concept:** `OrganizationAccountAccessRole` is the last-resort backdoor from management account. Protect it by never running workloads in management account. SCPs should prevent modification of critical cross-account roles.

---

**Q4:** A company uses Lake Formation for cross-account data access. Account A (Data Lake) grants SELECT on a table to Account B. Account B's analyst queries the table using Athena but gets "Access Denied" even though the Lake Formation grant shows active. What's wrong?

**A:** Lake Formation cross-account requires TWO steps: (1) Account A grants permissions to Account B's AWS account (done). (2) **Account B must accept the RAM share** AND then create a **resource link** to the shared table in their own Glue Data Catalog. (3) Account B's analyst must have Lake Formation permissions to access the resource link. (4) Account B may also need to grant itself permissions on the received resource via Lake Formation (re-grant within their account). **Common miss:** Just because Account A shared doesn't mean Account B's users can immediately query. The local Data Catalog resource link and local Lake Formation permissions are required. **Key concept:** Lake Formation cross-account = external grant + local resource link + local permissions.

---

**Q5:** A company has a PrivateLink setup: Account A has a service (NLB → instances) exposed via VPC Endpoint Service. Account B creates an Interface VPC Endpoint to connect. It works. Now Account C also wants access, but Account A hasn't added Account C to the endpoint service's allowed principals list. Can Account C use Account B's endpoint?

**A:** **No.** Each consumer account must create their OWN Interface VPC Endpoint, and the provider must explicitly allow each consumer account in the endpoint service's allowed principals list. Account B's endpoint is private to Account B's VPC — Account C can't use it (they'd need network connectivity to Account B's VPC AND even then, the ENI is in B's VPC). **Fix:** Account A adds Account C to the allowed principals list. Account C creates its own Interface VPC Endpoint. **Key concept:** PrivateLink endpoints are per-consumer. Each consumer needs their own endpoint AND provider approval.

---

**Q6:** A company implements cross-account backup. Production Account backups are copied to a Backup Account vault. The Backup Account vault has a vault access policy allowing only the backup service and a specific admin role. An attacker compromises the Production Account and deletes all local backups. Can they also delete backups in the Backup Account?

**A:** **No.** The Backup Account vault is in a separate account. The attacker compromised Production, not the Backup Account. (1) The attacker would need to assume a role in the Backup Account that has delete permissions on the vault — which they can't because the vault policy only allows specific roles. (2) Even if the attacker could access the backup vault, AWS Backup Vault Lock (if enabled) prevents deletion regardless of permissions. (3) SCPs on the Backup Account can deny deletion actions for added protection. **Key concept:** Cross-account backup isolation = production account compromise cannot affect backup account. This is the primary reason for cross-account backups. Add Vault Lock for immutability.

---

**Q7:** A company uses EventBridge cross-account: Account A's event bus receives events from Account B. Account A has a rule that triggers a Lambda function when events from Account B arrive. After months of working, it suddenly stops. Investigation shows Account B is still sending events but Account A's Lambda isn't triggering. What could have changed?

**A:** Possible causes: (1) **Account A's event bus resource policy** was modified or removed (no longer allows Account B). (2) **Account B's EventBridge rule** was modified (different event bus target or different event pattern). (3) **Account A's Lambda resource policy** was modified (Lambda needs a resource policy allowing EventBridge to invoke it). (4) **IAM permissions** changed for EventBridge's ability to invoke the Lambda. (5) **SCP change** denying EventBridge actions. (6) **Lambda concurrency limit** hit (throttling, not permission). **Most common exam answer:** The event bus resource policy in Account A was modified, removing Account B's permission to PutEvents. **Key concept:** Cross-account EventBridge requires: source's rule → target's bus resource policy → target's rule → target's Lambda policy. Break any link and it stops.

---

**Q8:** A company shares subnets via RAM from Network Account to Workload Account. The Workload Account creates an Application Load Balancer in the shared subnet. The ALB needs a security group that allows traffic from the Network Account's NAT Gateway (for health checks). Can the Workload Account reference the Network Account's security group in their SG rule?

**A:** **Yes — with limitations.** Within a shared VPC, security groups from the participant accounts and the owner account are in the SAME VPC. Security groups in the same VPC can reference each other by security group ID. So the Workload Account's ALB security group CAN reference the Network Account's NAT Gateway security group (if it exists). **However:** The workload account can only reference SGs they can "see" — they can reference their own SGs and owner's SGs by ID. **Key concept:** Shared VPC = same VPC = security groups can cross-reference. This is a benefit of shared VPC over VPC peering (where you can't reference remote SGs — you must use CIDRs).

---

**Q9:** A company's Data Lake account uses AWS Glue Data Catalog tables that reference S3 data. They share Glue resources via RAM with an Analytics account. The Analytics account's EMR cluster tries to query the data but can't read from S3. Glue catalog access works fine. What's missing?

**A:** RAM shares **Glue Data Catalog metadata** (database/table definitions) — but NOT the underlying **S3 data access**. The Analytics account can see the table schema but needs SEPARATE S3 permissions to read the actual data files. **Fix:** (1) S3 bucket policy in Data Lake account must allow the Analytics account's EMR role `s3:GetObject` on the data prefix. (2) If S3 is KMS-encrypted, KMS key policy must also allow the EMR role to decrypt. (3) EMR role's identity policy must allow s3:GetObject and kms:Decrypt. **Key concept:** RAM Glue sharing = metadata only. S3 data access is a separate permission layer. Same applies to Lake Formation grants (which handle both — another reason Lake Formation is preferred for governed data sharing).

---

**Q10:** A company wants to ensure that cross-account roles in their organization can ONLY be assumed by principals within the organization (not by external accounts that might guess the role ARN). What's the best preventive control?

**A:** Add a **condition to all cross-account trust policies** requiring `aws:PrincipalOrgID`:
```json
"Condition": {
  "StringEquals": {
    "aws:PrincipalOrgID": "o-myorgid"
  }
}
```
This ensures only principals from your organization can assume the role, even if external accounts know the role ARN. **Belt-and-suspenders:** Also use an SCP in target accounts denying `sts:AssumeRole` if the calling principal's org ID doesn't match (using Resource Control Policies or conditions on the deny SCP). **IAM Access Analyzer** can detect roles with trust policies that allow external access. **Key concept:** `aws:PrincipalOrgID` in trust policies restricts role assumption to your organization only — regardless of who knows the ARN.

---

## 9. Service Comparison Tables

### Cross-Account Access Methods

| Dimension | Role Assumption | Resource-Based Policy | RAM | PrivateLink | VPC Peering | Transit Gateway |
|-----------|----------------|----------------------|-----|-------------|-------------|-----------------|
| **Type** | Identity-based | Resource-based | Service sharing | Network | Network | Network |
| **Credentials** | Temporary (STS) | Caller's own | N/A | N/A | N/A | N/A |
| **Services** | All | S3, KMS, SQS, SNS, Lambda, etc. | TGW, Subnets, Route53, etc. | Any (via NLB) | VPC-to-VPC | VPC-to-VPC |
| **Transitive?** | Chaining possible | N/A | Depends on resource | No | No | Yes |
| **Data transfer cost** | N/A | N/A | N/A | $0.01/GB + hourly | Free (same AZ) | $0.02/GB |
| **Scale** | Unlimited | Per-resource policy | 5,000 shares/account | Per-endpoint | 125 peerings/VPC | 5,000 attachments |
| **Org integration** | Trust policy + SCP | Policy + SCP | Trusted access (auto-accept) | Manual allowlist | Manual | RAM sharing |
| **Exam use** | General access | Data access (S3/KMS) | Infrastructure sharing | Private service exposure | Few VPCs | Many VPCs |

### Resource Sharing Methods by Service

| Service | Sharing Method | Notes |
|---------|---------------|-------|
| S3 | Bucket policy, Access Points, or role assumption | Bucket Owner Enforced for cross-account writes |
| KMS | Key policy (always required) | Must grant specific crypto actions |
| DynamoDB | Role assumption only | No resource-based policy |
| RDS | Snapshot sharing, Aurora cloning (RAM) | KMS key sharing for encrypted |
| EC2 AMI | Launch permissions | Copy to own account for independence |
| EBS Snapshot | Snapshot permissions | KMS key for encrypted snapshots |
| Lambda | Resource policy (invocation) | For cross-account invocation |
| SQS | Queue policy | Common for SNS→SQS cross-account |
| SNS | Topic policy | Allow publish from other accounts |
| VPC (Subnet) | RAM | Shared VPC pattern |
| Transit Gateway | RAM | Centralized networking |
| Route 53 | RAM (resolver rules), API (PHZ association) | Different mechanisms for rules vs zones |
| Glue Data Catalog | RAM or Lake Formation | RAM = metadata only, LF = governed access |
| Secrets Manager | Resource policy | Cross-account secret access |
| ECR | Repository policy | Cross-account image pull |
| Backup Vault | Vault access policy | Cross-account backup copy |
| EventBridge | Event bus policy | Cross-account event routing |

### VPC Connectivity Options

| Dimension | VPC Peering | Transit Gateway | PrivateLink | Shared VPC (RAM) |
|-----------|-------------|-----------------|-------------|------------------|
| **Topology** | Point-to-point (1:1) | Hub-and-spoke | Point-to-point | Same VPC |
| **Transitive routing** | No | Yes | No | N/A (same VPC) |
| **Cross-region** | Yes | Yes (inter-region peering) | No (same region) | No (same region) |
| **Max connections** | 125 per VPC | 5,000 attachments | Unlimited endpoints | All subnets in VPC |
| **Data cost** | Free (same region/AZ) | $0.02/GB | $0.01/GB + hourly | Free (same VPC) |
| **CIDR overlap** | Not allowed | Not allowed | Allowed (uses ENIs) | Not applicable |
| **Security groups** | Can't reference remote SGs | Can't reference remote SGs | Can reference endpoint SGs | CAN reference (same VPC) |
| **DNS resolution** | Configurable | Via Resolver rules | Private DNS supported | Automatic (same VPC) |
| **Best for** | 2-3 VPCs, simple | Many VPCs, complex routing | Service exposure | Centralized networking, cost |

---

## 10. When to Pick Service A over Service B — Exam Keywords

### Pick Role Assumption over Resource-Based Policy when:
- Service doesn't support resource-based policies (DynamoDB, EC2, ECS)
- "temporary credentials"
- "assume role in another account"
- "multiple services in target account" (one role for all)
- "audit trail of who assumed what" (CloudTrail tracks role assumption)
- **Exam signal:** Non-S3/KMS/SQS services, multiple-action access, auditability

### Pick Resource-Based Policy over Role Assumption when:
- "S3 bucket access from another account"
- "allow Lambda invocation cross-account"
- "allow SQS publish from SNS in another account"
- "simpler (no role to assume)"
- "service-to-service" (SNS → SQS, S3 → Lambda)
- **Exam signal:** S3/KMS/SQS/SNS/Lambda, simple cross-account, service integration

### Pick RAM over Resource-Based Policy when:
- "share Transit Gateway"
- "share subnets" (VPC sharing)
- "share Route 53 resolver rules"
- "share infrastructure without duplication"
- "organization-wide sharing"
- **Exam signal:** Network infrastructure, resource remains with owner, org-level sharing

### Pick RAM over VPC Peering when:
- "share subnet (not just connect VPCs)"
- "resources in same VPC" (Shared VPC pattern)
- "reduce NAT Gateway costs"
- "security group cross-referencing"
- **Exam signal:** Same-subnet deployment, cost reduction, SG referencing

### Pick Transit Gateway over VPC Peering when:
- "many VPCs" (>3 needing connectivity)
- "transitive routing needed"
- "hub-and-spoke architecture"
- "centralized inspection"
- "route traffic through firewall"
- **Exam signal:** Complex routing, many accounts, inspection, transitive

### Pick VPC Peering over Transit Gateway when:
- "two VPCs only"
- "minimize cost" (no per-GB charge)
- "simple, direct connection"
- "high bandwidth, low latency"
- **Exam signal:** Simple 1:1, cost-sensitive, low complexity

### Pick PrivateLink over VPC Peering/TGW when:
- "expose single service privately"
- "SaaS provider to customer"
- "no CIDR overlap concerns"
- "unidirectional access"
- "service endpoint (not full VPC access)"
- **Exam signal:** Service exposure, overlapping CIDRs allowed, provider/consumer model

### Pick Shared VPC (RAM) over Transit Gateway when:
- "reduce costs (NAT, endpoints)"
- "resources need same-VPC behavior"
- "security group cross-referencing"
- "centralized network management"
- "fewer VPCs to manage"
- **Exam signal:** Cost, simplified networking, same-VPC semantics

### Pick External ID over other conditions when:
- "third-party service" accessing your account
- "prevent confused deputy"
- "external vendor assumes role"
- **Exam signal:** Third-party, vendor, SaaS integration (NOT internal cross-account)

### Pick `aws:PrincipalOrgID` over External ID when:
- "restrict to organization only"
- "internal cross-account access"
- "prevent external assumption"
- **Exam signal:** Internal org restriction (not third-party)

---

## 11. "Gotcha" Differences the Exam Tests

### Role Assumption Gotchas

1. **Role chaining caps at 1 hour.** No matter what the role's `MaxSessionDuration` is set to (even 12 hours), if you assumed it via another role (chaining), the session is 1 hour max.

2. **`:root` in trust policy ≠ root user.** It means "any principal in that account with appropriate IAM permissions." It's the account, not the root user specifically.

3. **Trust policy alone isn't enough.** The calling principal also needs `sts:AssumeRole` in their identity policy (unless they're root). Both sides must allow.

4. **Cross-account role has the TARGET account's SCPs applied.** After assumption, you operate under the target account's SCPs, not your source account's.

5. **Session policies can't ADD permissions.** They only RESTRICT the role session further. If the role allows s3:* but session policy allows only s3:GetObject → only GetObject works.

6. **`aws:PrincipalOrgID` vs `aws:PrincipalAccount`:** OrgID covers the entire org (all accounts). PrincipalAccount is a specific account. For org-wide trust, use OrgID.

### Resource-Based Policy Gotchas

7. **Same-account: Resource policy alone grants access.** If a bucket policy allows a role in the same account → the role can access WITHOUT identity policy. But cross-account (for users) needs both.

8. **Cross-account roles vs users differ.** If a resource policy grants a ROLE ARN cross-account → the role can access without identity policy. If it grants a USER → the user still needs identity policy.

9. **KMS key policies are REQUIRED, not optional.** Unlike S3 bucket policies (where identity policy alone works same-account), KMS key policies MUST grant access for any principal to use the key. No key policy statement = no access (even root of the owning account, unless the default key policy is kept).

10. **S3 Access Points per-account can simplify cross-account.** Instead of complex bucket policies, create per-account access points with specific policies.

### RAM Gotchas

11. **RAM trusted access MUST be explicitly enabled.** Just being in the same org doesn't enable auto-accept. The management account must call `EnableSharingWithAwsOrganization`.

12. **RAM doesn't share ALL resource types.** If the exam mentions sharing DynamoDB tables or EC2 instances via RAM → wrong. RAM has a specific supported resource list.

13. **RAM unsharing doesn't destroy existing resources.** Consumer's resources in shared subnets continue running. Only new resource creation is blocked.

14. **Shared VPC: Consumer can't create VPC-level resources.** Can't create subnets, route tables, gateways, peering connections in the shared VPC. Only launch resources INTO shared subnets.

15. **RAM permissions can be customized.** Default managed permissions may be too broad. Create custom managed permissions for least privilege.

### Networking Gotchas

16. **VPC Peering is NON-TRANSITIVE.** A↔B + B↔C ≠ A↔C. Each pair needs direct peering. The exam tests this constantly.

17. **VPC Peering can't overlap CIDRs.** If two VPCs have overlapping IP ranges, they can't be peered. PrivateLink doesn't have this restriction.

18. **Transit Gateway charges per-GB.** VPC Peering (same region) is free for data. High-volume cross-account traffic via TGW can be expensive. Exam may test "most cost-effective" = peering for simple cases.

19. **PrivateLink is unidirectional (consumer → provider).** Provider can't initiate connections to consumer. If bidirectional is needed → VPC Peering or TGW.

20. **Shared VPC security groups can cross-reference.** This is a KEY advantage over peering/TGW where you can't reference remote security groups (must use CIDR blocks in rules).

### Encryption & KMS Gotchas

21. **Encrypted S3 cross-account needs KMS grant.** The #1 missed permission. People share bucket access but forget KMS key access → AccessDenied.

22. **Cross-account KMS requires BOTH key policy AND caller's IAM policy.** Key policy alone isn't sufficient (caller needs IAM policy allowing kms:Decrypt on the key ARN).

23. **Cross-account S3 CRR with KMS: Need destination key.** Can't replicate with source key into destination account. Must specify a destination KMS key and grant replication role access to both keys.

24. **AWS-managed keys (aws/s3, aws/ebs) can't be shared cross-account.** Only customer-managed CMKs can have key policies modified for cross-account. If resources use AWS-managed keys → must re-encrypt with CMK before sharing.

25. **Cross-account EBS snapshot: Must share KMS key AND grant kms:CreateGrant.** CreateGrant is needed for the consumer to attach the volume (EBS needs to create a grant for the instance to use the key).

---

## 12. Decision Tree: Choosing the Right Cross-Account Method

```
START: What type of cross-account access is needed?
│
├─── Need NETWORK CONNECTIVITY between VPCs?
│    ├─── How many VPCs?
│    │    ├─── 2-3 VPCs, simple direct connection? → VPC Peering
│    │    ├─── Many VPCs (4+), need transitive routing? → Transit Gateway (RAM shared)
│    │    └─── Same VPC behavior (SG referencing, cost saving)? → Shared VPC (RAM subnets)
│    │
│    ├─── Need to expose a service (not full VPC access)?
│    │    ├─── Private, one-directional? → PrivateLink (VPC Endpoint Service)
│    │    └─── Cross-region needed? → VPC Peering or TGW inter-region peering
│    │
│    └─── CIDR overlap between VPCs?
│         ├─── Yes → PrivateLink (only option that handles overlap)
│         └─── No → Peering or TGW
│
├─── Need ACCESS TO DATA/SERVICES?
│    ├─── Service supports resource-based policies? (S3, KMS, SQS, SNS, Lambda)
│    │    ├─── Simple one-service access? → Resource-based policy
│    │    ├─── Encrypted? → Resource policy + KMS key policy (BOTH required)
│    │    └─── Cross-account S3 writes? → + Bucket Owner Enforced
│    │
│    ├─── Service doesn't support resource-based policies? (DynamoDB, EC2, ECS, RDS API)
│    │    └─── Role assumption (sts:AssumeRole) → only option
│    │
│    ├─── Need multiple-service access in target account?
│    │    └─── Role assumption (one role, multiple service permissions)
│    │
│    └─── Need fine-grained data access (table/column level)?
│         └─── Lake Formation cross-account grants
│
├─── Need to SHARE INFRASTRUCTURE RESOURCES?
│    ├─── Transit Gateway? → RAM
│    ├─── Subnets (Shared VPC)? → RAM
│    ├─── Route 53 Resolver Rules? → RAM
│    ├─── IPAM Pools? → RAM
│    ├─── License configurations? → RAM
│    ├─── Network Firewall policies? → RAM
│    ├─── Glue Data Catalog? → RAM (metadata) or Lake Formation (governed)
│    └─── Prefix Lists? → RAM
│
├─── Need to SHARE ARTIFACTS?
│    ├─── AMIs? → Modify launch permissions (+ share KMS key if encrypted)
│    ├─── EBS Snapshots? → Modify snapshot permissions (+ KMS + CreateGrant)
│    ├─── RDS Snapshots? → Share snapshot (+ KMS key for encrypted)
│    ├─── ECR Images? → Repository policy allowing cross-account pull
│    └─── Secrets? → Secrets Manager resource policy
│
├─── Need THIRD-PARTY access?
│    ├─── External service assumes role? → Trust policy + External ID
│    ├─── SaaS provider service? → PrivateLink (provider exposes endpoint)
│    └─── Marketplace integration? → Follow vendor's integration guide
│
├─── Need ORG-WIDE access control?
│    ├─── Restrict who accesses your resources? → RCP (Resource Control Policy)
│    ├─── Restrict what your identities access? → SCP
│    ├─── Detect external access? → IAM Access Analyzer
│    └─── Organization-level trust? → aws:PrincipalOrgID condition
│
└─── Need CROSS-ACCOUNT DR/BACKUP?
     ├─── S3 data replication? → CRR (cross-account, cross-region)
     ├─── RDS/Aurora backup? → Cross-account snapshot sharing or Backup vault copy
     ├─── EBS volumes? → Cross-account snapshot + KMS sharing
     ├─── Full backup solution? → AWS Backup cross-account vault copy
     └─── Immutable backups? → Backup Vault Lock in backup account
```
