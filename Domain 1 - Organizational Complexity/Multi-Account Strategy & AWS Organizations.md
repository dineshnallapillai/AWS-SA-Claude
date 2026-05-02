# Multi-Account Strategy & AWS Organizations

## 1. Core Concepts & Theory

---

### Why Multi-Account?

**Single-account problems at scale:**
- Blast radius: One misconfiguration affects everything
- IAM complexity: Policies become unmanageable
- Service quotas: Shared limits across teams
- Billing: No cost isolation between teams/projects
- Compliance: Different workloads have different regulatory needs
- Security: Lateral movement risk if one workload is compromised

**Multi-account benefits:**
- **Blast radius reduction:** Issues contained to one account
- **Security isolation:** Hard boundary between workloads (account = strongest isolation boundary in AWS)
- **Service quotas:** Per-account limits (independent scaling)
- **Billing separation:** Clear cost attribution per team/project/environment
- **Compliance:** Apply controls at account level (HIPAA accounts, PCI accounts)
- **Autonomy:** Teams manage their own accounts with guardrails

---

### AWS Organizations

**What it is:** Central governance and management service for multiple AWS accounts. Hierarchical structure with policies applied at various levels.

#### Core Structure

```
Management Account (root)
│
├── Root (top of hierarchy)
│   ├── OU: Security
│   │   ├── Log Archive Account
│   │   └── Security Tooling Account
│   ├── OU: Infrastructure
│   │   ├── Networking Account (Transit Gateway, DNS)
│   │   └── Shared Services Account
│   ├── OU: Workloads
│   │   ├── OU: Production
│   │   │   ├── App A Prod Account
│   │   │   └── App B Prod Account
│   │   ├── OU: Staging
│   │   └── OU: Development
│   ├── OU: Sandbox
│   │   └── Developer Sandbox Accounts
│   └── OU: Suspended
│       └── (Accounts pending deletion)
```

#### Management Account

- **Root of the organization** — owns the organization
- **Should NOT host workloads** (best practice)
- **Only use for:** Organization management, billing, creating accounts
- Has OrganizationAccountAccessRole in all member accounts (full admin)
- Cannot be changed — you cannot transfer management account ownership
- **Billing:** All member account charges consolidated here (Consolidated Billing)

#### Organizational Units (OUs)

- Hierarchical containers for accounts
- **Nesting:** Up to 5 levels deep
- Policies attached to OUs inherit downward (to child OUs and accounts)
- An account can belong to only ONE OU at a time
- Move accounts between OUs to change policy application

#### Member Accounts

- Accounts within the organization
- Created by the organization OR invited to join
- **OrganizationAccountAccessRole:** IAM role created automatically in new accounts (allows management account to assume full admin access)
- Can leave the organization (if they have standalone billing configured)

---

### Organization Policies

#### Service Control Policies (SCPs)

**What they are:** Permission guardrails that define the MAXIMUM permissions available to accounts in the organization. They don't grant permissions — they restrict them.

**Key characteristics:**
- Do NOT apply to the management account (critical exam fact)
- Apply to all IAM users, roles, and the root user in member accounts
- Do NOT apply to service-linked roles
- Must have an explicit Allow for actions to work (deny by default if using allow-list strategy)
- Inheritance: Attached at Root → applies to all OUs and accounts below
- **Effective permissions = Intersection of SCP + IAM policy**

**SCP Strategies:**

1. **Deny List (recommended, default):**
   - FullAWSAccess SCP attached at root (allow everything)
   - Attach deny SCPs for specific actions you want to block
   - Easier to manage (only define what's blocked)

2. **Allow List:**
   - Remove FullAWSAccess
   - Explicitly allow only specific services/actions
   - Very restrictive, harder to manage, easy to accidentally lock out

**Common SCP Examples:**
```json
// Deny leaving the organization
{
  "Effect": "Deny",
  "Action": "organizations:LeaveOrganization",
  "Resource": "*"
}

// Deny disabling CloudTrail
{
  "Effect": "Deny",
  "Action": [
    "cloudtrail:StopLogging",
    "cloudtrail:DeleteTrail"
  ],
  "Resource": "*"
}

// Deny root user actions
{
  "Effect": "Deny",
  "Action": "*",
  "Resource": "*",
  "Condition": {
    "StringLike": {
      "aws:PrincipalArn": "arn:aws:iam::*:root"
    }
  }
}

// Restrict to specific regions
{
  "Effect": "Deny",
  "Action": "*",
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "aws:RequestedRegion": ["us-east-1", "eu-west-1"]
    }
  }
}
```

**SCP Limits:**
- Max 5 SCPs per entity (OU or account)
- Max SCP document size: 5,120 characters
- Max SCPs per organization: 1,000

#### Tag Policies

- Enforce standardized tagging across the organization
- Define allowed tag keys, allowed values, case treatment
- Can enforce compliance or just report non-compliance
- Helps with cost allocation and governance

#### Backup Policies

- Centrally define AWS Backup plans
- Apply backup plans to resources across member accounts
- Member accounts cannot override or delete organization backup plans
- Ensures consistent backup strategy (compliance)

#### AI Services Opt-Out Policies

- Control whether AWS AI services can store/use your content for service improvement
- Apply organization-wide opt-out for specific AI services
- Covers: Comprehend, Lex, Polly, Rekognition, Textract, Transcribe, etc.

#### Resource Control Policies (RCPs)

- NEW policy type (2024)
- Control access TO resources (resource-centric), complementing SCPs (identity-centric)
- Restrict who can access resources in your organization's accounts
- Example: Prevent external principals from accessing your S3 buckets

---

### AWS Control Tower

**What it is:** Automated, best-practice multi-account setup service. Orchestrates Organizations, SCPs, AWS Config, IAM Identity Center, CloudTrail, and more.

#### Key Components

**Landing Zone:**
- Pre-configured, well-architected multi-account environment
- Automated setup of: Management account, Log Archive account, Audit account
- Baseline networking, IAM, and logging configuration
- Can be customized with Account Factory Customization (AFC)

**Guardrails (Controls):**
- Pre-built governance rules
- **Types:**
  - **Preventive:** Implemented as SCPs (block non-compliant actions)
  - **Detective:** Implemented as AWS Config Rules (detect non-compliance)
  - **Proactive:** Implemented as CloudFormation Hooks (block non-compliant resource creation)
- **Behaviors:**
  - **Mandatory:** Always on, cannot be disabled (e.g., disallow CloudTrail changes)
  - **Strongly recommended:** Best practices (e.g., enable encryption)
  - **Elective:** Optional (e.g., restrict specific regions)

**Account Factory:**
- Automated provisioning of new accounts
- Pre-configured with guardrails and networking
- **Account Factory for Terraform (AFT):** Terraform-based account provisioning pipeline
- **Account Factory Customization (AFC):** Apply custom templates to new accounts (Service Catalog)

**Dashboard:**
- Central visibility into compliance status across all accounts
- Drift detection (alerts when accounts drift from baseline)

#### Control Tower vs Organizations (alone)

| Aspect | Organizations (alone) | Control Tower |
|--------|----------------------|---------------|
| Setup | Manual | Automated landing zone |
| Guardrails | You write SCPs manually | Pre-built preventive + detective + proactive |
| Account provisioning | CreateAccount API | Account Factory (automated, standardized) |
| Compliance visibility | Build yourself | Built-in dashboard |
| Drift detection | Manual | Automated |
| Best for | Custom, existing orgs | New setups, fast compliance |

---

### Consolidated Billing

**How it works:**
- Management account pays for all member accounts
- Single bill for the entire organization
- **Volume discounts:** Combined usage qualifies for pricing tiers (S3, EC2, etc.)
- **Reserved Instance / Savings Plan sharing:** RIs and SPs apply across the organization (can be disabled per account)
- **Credits:** Applied to the account that generated them first, then shared

**Key features:**
- **Cost Allocation Tags:** Tag resources for reporting (organization-wide activation)
- **Linked account billing:** Each account's charges visible separately in Cost Explorer
- **RI/SP sharing:** Can disable sharing at the account level (payer account controls)

---

### AWS IAM Identity Center (formerly SSO)

**What it is:** Centralized identity and access management for multi-account environments.

**Key features:**
- Single sign-on to all AWS accounts in the organization
- **Permission Sets:** Define IAM roles deployed to accounts (map users/groups to accounts with specific permissions)
- **Identity sources:** Built-in directory, Active Directory (AD Connector or AWS Managed AD), or external IdP (Okta, Azure AD) via SAML 2.0 / SCIM
- **Multi-account permissions:** Assign users/groups to multiple accounts with different permission sets
- **Application assignments:** SSO into SAML-compatible business applications

**Architecture:**
```
Identity Source (AD, Okta, built-in) →
  IAM Identity Center →
    Permission Sets (mapped to users/groups + accounts) →
      Temporary IAM roles in each target account
```

**Key exam facts:**
- Requires AWS Organizations (must be enabled in management account or delegated admin)
- Delegated administrator: Can delegate Identity Center management to a member account
- Permission Sets are deployed as IAM roles in target accounts
- Session duration configurable (1–12 hours)

---

### Key Quotas & Limits

| Resource | Limit |
|----------|-------|
| Accounts per organization | Default 10 (soft limit, increase to thousands) |
| OUs per organization | 1,000 |
| OU nesting depth | 5 levels |
| SCPs per entity (OU/account) | 5 |
| SCP document size | 5,120 characters |
| SCPs per organization | 1,000 |
| Tag policies per entity | 5 |
| Policies per organization (each type) | 1,000 |
| IAM Identity Center permission sets per instance | 2,000 |
| Permission sets per account | 50 |

---

## 2. Design Patterns & Best Practices

### Recommended Account Structure (AWS Best Practice)

#### Core Accounts (always create these)

| Account | Purpose |
|---------|---------|
| **Management** | Organization management only, billing. NO workloads |
| **Log Archive** | Centralized logging (CloudTrail, Config, VPC Flow Logs, access logs). Restricted access |
| **Security Tooling (Audit)** | Security services hub (GuardDuty, Security Hub, Detective, IAM Access Analyzer). Read-only cross-account access |
| **Network** | Transit Gateway, Route 53, Direct Connect, VPN, shared VPCs |
| **Shared Services** | CI/CD pipelines, artifact repositories, Active Directory, shared AMIs |

#### Workload Accounts

| OU | Account Pattern | Purpose |
|----|----------------|---------|
| Production | Per-app or per-team prod accounts | Production workloads (strict controls) |
| Staging | Mirror of prod accounts | Pre-production testing |
| Development | Per-team dev accounts | Development (more permissive) |
| Sandbox | Per-developer accounts | Experimentation (budget-limited, auto-cleanup) |

#### Special-Purpose Accounts

| Account | Purpose |
|---------|---------|
| **Data Lake** | Centralized data analytics (S3, Glue, Athena, Lake Formation) |
| **Backup** | Cross-account backup vault (AWS Backup) |
| **Transit/Perimeter** | Inspection VPC, firewalls (Network Firewall, IDS/IPS) |
| **Identity** | Delegated IAM Identity Center administration |

### OU Design Patterns

**Pattern 1: By Environment (most common)**
```
Root → Workloads → Production / Staging / Development
```

**Pattern 2: By Business Unit**
```
Root → BusinessUnit-A → Prod / Dev
Root → BusinessUnit-B → Prod / Dev
```

**Pattern 3: By Compliance**
```
Root → PCI (strict controls) → Prod / Dev
Root → HIPAA (healthcare controls) → Prod / Dev
Root → Standard → Prod / Dev
```

**Best practice:** Combine patterns — top-level by function (Security, Infrastructure, Workloads), workloads split by environment or business unit.

### Anti-Patterns

| Anti-Pattern | Why It's Wrong | Better Approach |
|--------------|---------------|-----------------|
| Running workloads in management account | Blast radius, security risk, SCPs don't protect it | Dedicated workload accounts |
| Single account for all environments | No isolation, risk of prod accidents | Separate accounts per environment |
| Flat OU structure (all accounts in root) | Can't apply differentiated policies | Hierarchical OUs by function/env |
| Over-nesting OUs (5+ levels) | Complex inheritance, hard to reason about | 2-3 levels is ideal |
| SCPs as primary access control | SCPs are guardrails, not grants | IAM policies grant access; SCPs set boundaries |
| Sharing RI/SP across all accounts blindly | Sandbox/dev accounts consume RI capacity | Disable sharing for non-production |
| Manual account provisioning | Inconsistent configuration, drift | Account Factory / AFT / IaC |
| One security account doing everything | Overloaded, unclear ownership | Separate Log Archive + Security Tooling |
| Not using a Suspended OU | Deleted accounts remain active 90 days | Move to Suspended OU with deny-all SCP |

### Well-Architected Framework Alignment

**Reliability:**
- Account isolation prevents blast radius propagation
- Per-account service quotas prevent resource starvation
- Multi-account disaster recovery (separate DR accounts in different regions)

**Security:**
- Account boundary = strongest isolation primitive in AWS
- SCPs enforce guardrails regardless of IAM permissions
- Centralized security tooling with delegated admin
- Least privilege at the organizational level

**Performance:**
- Dedicated accounts = dedicated service quotas
- No resource contention between teams
- Parallel deployments across accounts

**Cost:**
- Consolidated billing for volume discounts
- Clear cost attribution per account
- RI/SP sharing across organization
- Budget alerts per account/OU

**Operational Excellence:**
- Standardized account provisioning (Account Factory)
- Centralized logging and monitoring
- Consistent guardrails via SCPs
- Drift detection via Control Tower

### Integration Patterns

```
┌────────────────────────────────────────────────────────────────────────┐
│              Multi-Account Architecture Patterns                        │
├────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Centralized Logging:                                                   │
│  All Accounts → CloudTrail → Organization Trail → Log Archive (S3)     │
│  All Accounts → Config → Aggregator → Security Tooling Account         │
│  All Accounts → VPC Flow Logs → S3 (Log Archive) / CloudWatch          │
│                                                                         │
│  Centralized Security:                                                  │
│  GuardDuty → Delegated Admin (Security Account) → All Members          │
│  Security Hub → Delegated Admin → Aggregate findings                    │
│  IAM Access Analyzer → Organization-level analyzer                     │
│                                                                         │
│  Centralized Networking:                                                │
│  Network Account: Transit Gateway (shared via RAM)                     │
│  All VPCs attach to TGW → route through inspection VPC                 │
│  Route 53: Private Hosted Zones shared cross-account                   │
│                                                                         │
│  CI/CD Pipeline:                                                        │
│  Shared Services: CodePipeline → CodeBuild →                           │
│    Deploy to Dev Account → Staging Account → Prod Account              │
│    (Cross-account IAM roles for deployment)                            │
│                                                                         │
│  Identity:                                                              │
│  IAM Identity Center (Management or Delegated Admin) →                 │
│    Permission Sets → All Accounts                                      │
│    External IdP (Okta/Azure AD) → SCIM provisioning                   │
│                                                                         │
│  Backup:                                                                │
│  Organization Backup Policy → All accounts                             │
│  Cross-account backup vault copy (Backup Account)                      │
│                                                                         │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Security & Compliance

### SCP Security Patterns

**Deny root user access in all member accounts:**
```json
{
  "Effect": "Deny",
  "Action": "*",
  "Resource": "*",
  "Condition": {
    "StringLike": {
      "aws:PrincipalArn": "arn:aws:iam::*:root"
    }
  }
}
```

**Prevent disabling security services:**
```json
{
  "Effect": "Deny",
  "Action": [
    "guardduty:DeleteDetector",
    "guardduty:DisassociateFromMasterAccount",
    "securityhub:DisableSecurityHub",
    "config:StopConfigurationRecorder",
    "config:DeleteConfigurationRecorder",
    "cloudtrail:StopLogging",
    "cloudtrail:DeleteTrail"
  ],
  "Resource": "*"
}
```

**Restrict regions:**
```json
{
  "Effect": "Deny",
  "NotAction": [
    "iam:*",
    "organizations:*",
    "support:*",
    "sts:*",
    "budgets:*"
  ],
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "aws:RequestedRegion": ["us-east-1", "eu-west-1"]
    }
  }
}
```
*Note: Exclude global services (IAM, Organizations, STS) from region deny.*

**Deny leaving organization:**
```json
{
  "Effect": "Deny",
  "Action": "organizations:LeaveOrganization",
  "Resource": "*"
}
```

**Require encryption:**
```json
{
  "Effect": "Deny",
  "Action": "s3:PutObject",
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "s3:x-amz-server-side-encryption": "aws:kms"
    }
  }
}
```

### Cross-Account Access Patterns

**Pattern 1: IAM Role Assumption (most common)**
```
Account A (source) → sts:AssumeRole → Account B (target role)
- Target account creates role with trust policy allowing Account A
- Source account needs iam:AssumeRole permission
- Use external ID for third-party access
```

**Pattern 2: Resource-Based Policies**
```
Account B's resource (S3 bucket, KMS key, SNS topic) has policy allowing Account A
- No role assumption needed
- Principal in Account A accesses resource directly
- Supported: S3, SQS, SNS, KMS, Lambda, ECR, Secrets Manager, etc.
```

**Pattern 3: AWS RAM (Resource Access Manager)**
```
Share resources across accounts/OUs without duplicating:
- Transit Gateway, Subnets, Route 53 Resolver Rules
- License Manager, Outposts, CodeBuild projects
- Aurora DB clusters, Resource Groups
- RAM + Organizations: Share within org automatically (no invitation acceptance needed)
```

**Pattern 4: Service Delegated Administrator**
```
Management Account delegates administration of a service to a member account:
- GuardDuty, Security Hub, Config, Firewall Manager
- IAM Identity Center, CloudFormation StackSets, Service Catalog
- Reduces load on management account
- Max delegated admins varies by service (typically 1-5)
```

### Centralized Security Services

| Service | Delegated Admin? | Organization-Wide Feature |
|---------|-----------------|--------------------------|
| GuardDuty | Yes | Auto-enable for new accounts, centralized findings |
| Security Hub | Yes | Aggregate findings from all accounts + regions |
| Config | Yes | Organization Config Rules, Aggregator |
| CloudTrail | Yes (limited) | Organization Trail (logs all accounts) |
| Firewall Manager | Yes | Centralized WAF, SG, Network Firewall policies |
| IAM Access Analyzer | Yes | Organization-level unused access analysis |
| Detective | Yes | Cross-account investigation graphs |
| Macie | Yes | Organization-wide S3 sensitive data scanning |
| Inspector | Yes | Vulnerability scanning across org |
| Audit Manager | Yes | Compliance evidence collection |

### Organization Trail (CloudTrail)

- Created in management account, logs ALL member accounts
- Member accounts CANNOT modify or delete the organization trail
- Logs delivered to Log Archive account S3 bucket
- Can also send to CloudWatch Logs in each account
- Key compliance requirement: Tamper-proof, centralized audit log

### Compliance Patterns

**PCI DSS / HIPAA / FedRAMP:**
- Separate OU with strict SCPs for regulated workloads
- Only approved services allowed (SCP allow-list)
- Mandatory encryption, logging, access controls
- Separate accounts prevent co-mingling of regulated and non-regulated data

**Data Residency:**
- Region-restrict SCPs per OU (data stays in specific regions)
- Separate accounts per region if needed
- VPC endpoints to prevent data leaving region

---

## 4. Cost Optimization

### Consolidated Billing Benefits

**Volume Discounts:**
- S3: Tiered pricing aggregated across all accounts
- EC2 Data Transfer: Combined volume for tier pricing
- Other services with volume pricing: Combined usage

**Reserved Instances & Savings Plans Sharing:**
- RIs and SPs purchased in any account apply across the organization
- Can disable sharing per account (payer account controls this)
- **Best practice:** Purchase RIs/SPs in a dedicated "Billing" or management account

**Free Tier:**
- Free tier is per organization, NOT per account
- Creating multiple accounts does NOT multiply free tier

### Cost Allocation & Attribution

**Tags:**
- Activate cost allocation tags at organization level
- Tag policies enforce consistent tagging
- Use tags for: Team, Environment, Project, Cost Center

**AWS Cost Explorer:**
- Filter by linked account, OU, tag
- Identify cost trends per team/project/environment

**AWS Budgets:**
- Per-account budgets with alerts
- Organization-wide budgets
- Budget actions (auto-stop instances, deny IAM, etc.)

### Cost Optimization Strategies

| Strategy | Detail |
|----------|--------|
| RI/SP centralized purchasing | Buy in one account, share org-wide |
| Sandbox account budget caps | Budget actions to stop spend after limit |
| Dev/test auto-shutdown | Schedule instances off outside business hours |
| Unused account cleanup | Move to Suspended OU → close after 90 days |
| Right-size per account | Smaller instances in dev, larger in prod |
| Disable RI sharing for sandboxes | Prevent sandbox consumption of reserved capacity |

### Cost Traps

- **Management account charges:** Organization-level services (Control Tower, CloudTrail org trail) billed here
- **Cross-account data transfer:** Data between accounts in same region = free (same AZ via VPC peering/TGW). Different AZ/region = charged
- **Transit Gateway:** Per-attachment-hour + per-GB data processing (adds up fast in multi-account)
- **CloudTrail:** Organization trail = one copy of events (good). Enabling per-account trails in addition = double storage cost
- **VPC endpoints per account:** Each account/VPC needs its own endpoints (costs per endpoint-hour). Consider centralized VPC endpoints via shared services
- **AWS Config:** Per-rule, per-account, per-region evaluation cost. Organization Config rules = many evaluations
- **GuardDuty:** Per-account, per-region. 10 accounts × 5 regions = 50 detectors billed
- **Unused accounts:** Even empty accounts incur baseline costs if services (Config, GuardDuty) are enabled

---

## 5. High Availability, Disaster Recovery & Resilience

### Multi-Account Resilience Patterns

**Account-Level Isolation for Blast Radius:**
```
Scenario: Compromised credentials in Dev account
Impact: Limited to Dev account only
Prod accounts: Unaffected (separate IAM, separate resources)
```

**DR Across Accounts:**
```
Primary: Account A (us-east-1)
DR: Account B (eu-west-1)
- Cross-account replication: S3 CRR, RDS cross-account snapshots
- Backup: AWS Backup cross-account vault copy
- AMIs: Share AMIs cross-account
- IaC: Deploy same infrastructure in DR account via CloudFormation StackSets
```

**Resilience of Organizations Itself:**
- AWS Organizations is a global service (managed by AWS, highly available)
- SCPs are eventually consistent (propagation delay: seconds to minutes)
- If Organizations has an issue: IAM still works (accounts function independently)
- Organization management account is a critical dependency → protect it absolutely

### StackSets for Multi-Account Deployment

**AWS CloudFormation StackSets:**
- Deploy CloudFormation stacks across multiple accounts and regions simultaneously
- **Service-managed:** Automatically deploys to new accounts in specified OUs
- **Self-managed:** Deploy to specific account IDs
- Use for: Baseline security resources, IAM roles, Config rules, VPC configuration

**Key features:**
- Automatic drift detection
- Sequential or parallel deployment
- Failure tolerance (continue deploying even if some accounts fail)
- Automatic deployment to new accounts (service-managed + auto-deployment enabled)

### Resilience Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│             Multi-Account DR Architecture                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Primary Region (us-east-1)          DR Region (eu-west-1)       │
│  ┌──────────────┐                    ┌──────────────┐            │
│  │ Prod Account │ ──S3 CRR──────── │ DR Account   │            │
│  │              │ ──RDS snapshot──→ │              │            │
│  │              │ ──DynamoDB GT──→  │              │            │
│  │              │ ──Backup vault──→ │              │            │
│  └──────────────┘                    └──────────────┘            │
│                                                                   │
│  Route 53 (failover routing) → switches to DR account endpoint   │
│  CloudFormation StackSets → same infra in both accounts/regions  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### RPO/RTO Considerations

| Strategy | RPO | RTO | Multi-Account Approach |
|----------|-----|-----|----------------------|
| Backup & Restore | Hours | Hours | Cross-account backup vault, restore in DR account |
| Pilot Light | Minutes | 10-30 min | Minimal infra in DR account, scale up on failover |
| Warm Standby | Seconds-Minutes | Minutes | Scaled-down duplicate in DR account |
| Multi-Site Active-Active | Zero | Zero | Full deployment in multiple accounts/regions simultaneously |

---

## 6. Exam-Focused Section

### Straightforward Questions

**Q1:** A company wants to ensure that no one in any member account can disable CloudTrail logging. How should they enforce this?

**A:** Attach an SCP to the organization Root that denies `cloudtrail:StopLogging` and `cloudtrail:DeleteTrail`. Since SCPs apply to all member accounts (but not the management account), no IAM user or role in member accounts can disable CloudTrail regardless of their IAM permissions.

---

**Q2:** A company is setting up a new multi-account environment and wants automated guardrails, account provisioning, and compliance dashboards with minimal custom development. What should they use?

**A:** AWS Control Tower. It provides an automated landing zone, pre-built guardrails (preventive SCPs + detective Config rules + proactive CloudFormation Hooks), Account Factory for standardized provisioning, and a built-in compliance dashboard.

---

**Q3:** A company needs to share a Transit Gateway with 50 accounts across their organization. What service enables this without individual account invitations?

**A:** AWS Resource Access Manager (RAM). When sharing within an AWS Organization, RAM can share resources (like Transit Gateway) to an entire OU or organization without requiring acceptance by member accounts (trusted access enabled).

---

**Q4:** A company wants all AWS accounts to be billed together and benefit from volume discounts. They also want Reserved Instances purchased by one team to apply across the organization. What feature provides this?

**A:** AWS Organizations Consolidated Billing. All accounts share a single bill, combined usage qualifies for volume pricing tiers, and RI/Savings Plan benefits automatically apply across the organization (unless sharing is disabled for specific accounts).

---

**Q5:** A company needs to centrally manage SSO access for 200 developers across 30 AWS accounts, using their existing Okta identity provider. What service should they use?

**A:** AWS IAM Identity Center (formerly AWS SSO). Configure Okta as an external IdP via SAML 2.0 with SCIM provisioning. Create Permission Sets for different access levels and assign users/groups to accounts. Developers get SSO access to all assigned accounts from a single portal.

---

**Q6:** A company needs to restrict all AWS activity to eu-west-1 and eu-central-1 for GDPR compliance. Some global services (IAM, STS, Organizations) must still work. How?

**A:** Attach an SCP that denies all actions when `aws:RequestedRegion` is not in the allowed regions, BUT exclude global services (IAM, STS, Organizations, Support, Budgets) using `NotAction`. This allows global services to function while restricting regional services to approved regions only.

---

**Q7:** A company creates new accounts frequently and wants security baselines (Config rules, GuardDuty, CloudTrail) automatically deployed to every new account without manual intervention. What should they use?

**A:** CloudFormation StackSets with service-managed permissions and auto-deployment enabled. When a new account is created in the target OU, StackSets automatically deploys the security baseline stack. Alternatively, Control Tower with mandatory guardrails provides this out of the box.

---

### Tricky / Scenario-Based Questions

**Q1:** A company has an SCP attached to the Workloads OU that denies `ec2:TerminateInstances`. A developer in a member account under that OU has an IAM policy with `"Effect": "Allow", "Action": "ec2:*"`. Can the developer terminate instances?

**A:** **No.** SCPs define the maximum permissions available. Even though the IAM policy allows `ec2:*`, the SCP denies `ec2:TerminateInstances`, which takes precedence. Effective permissions = Intersection of SCP (boundary) and IAM policy (grant). The deny in the SCP wins. **Key concept:** SCP Deny > IAM Allow. SCPs are the ceiling, IAM policies are the grant within that ceiling.

---

**Q2:** A company's management account administrator accidentally has their credentials compromised. The attacker uses them to disable GuardDuty and delete CloudTrail. The company has SCPs denying these actions. Why did the SCP fail to prevent this?

**A:** **SCPs do NOT apply to the management account.** This is a critical exam fact. The management account is exempt from all SCPs — it always has full permissions regardless of what SCPs are attached. **Mitigation:** (1) Never run workloads in management account. (2) Use extremely strong MFA on management account root and admin users. (3) Use minimal IAM users/roles in management account. (4) Enable CloudTrail in a Log Archive account where management account can't delete logs. **Exam trap:** If the question involves the management account, SCPs are irrelevant.

---

**Q3:** A company has 500 member accounts organized into OUs. They need to ensure all S3 buckets across all accounts block public access. They've enabled the S3 Block Public Access at the organization level. A month later, they discover some accounts still have public buckets. What went wrong?

**A:** The **S3 Block Public Access organization-level setting only applies to new accounts created AFTER it was enabled** (depending on how it was configured). Existing accounts that already had public buckets are not retroactively changed. **Fix:** (1) Use an SCP to deny `s3:PutBucketPolicy` and `s3:PutBucketAcl` that would make buckets public. (2) Use AWS Config organization rule (`s3-bucket-public-read-prohibited`) to detect non-compliance. (3) Use a remediation action (SSM Automation) to auto-fix. (4) Also consider Firewall Manager or custom Lambda remediation. **Key concept:** Organization-level settings may not be retroactive — combine preventive (SCP) + detective (Config) + corrective (auto-remediation).

---

**Q4:** A company uses Control Tower. A developer uses the AWS CLI to create a VPC in a region that's not in the allowed regions list. Control Tower shows the account as "drifted." How should they resolve this and prevent recurrence?

**A:** (1) **Resolve drift:** Delete the non-compliant resource (VPC in disallowed region). Then in Control Tower, re-register the OU or repair the account to clear the drift status. (2) **Prevent recurrence:** Ensure the "Region deny" guardrail is enabled (preventive SCP). If the developer was able to create the VPC, the guardrail was either not enabled or not applied to that OU. Enable the mandatory/strongly recommended region deny control. **Key concept:** Control Tower drift = configuration has diverged from the expected baseline. You must fix the drift AND prevent recurrence.

---

**Q5:** A company wants to implement a "break glass" procedure where, in emergencies, an admin in a member account can perform actions normally denied by SCPs (e.g., disable a security tool that's causing an outage). How should they design this?

**A:** **You cannot override SCPs from a member account.** Instead: (1) Create a specific IAM role in the member account for break-glass use. (2) Add an SCP **condition** that excludes this role from the deny:
```json
"Condition": {
  "ArnNotLike": {
    "aws:PrincipalArn": "arn:aws:iam::*:role/BreakGlassRole"
  }
}
```
(3) Protect the BreakGlassRole with strict MFA requirements, restricted trust policy, and heavy logging. (4) Alert on any use of the BreakGlassRole. **Alternative:** Require management account intervention (assume OrganizationAccountAccessRole). **Key concept:** SCPs can have conditions to exempt specific roles — this is the "escape hatch" pattern.

---

**Q6:** A company has two OUs: "Production" (strict SCPs) and "Development" (permissive). An application team needs a new account that's initially in Development but will move to Production after testing. When the account moves to Production, existing non-compliant resources (e.g., unencrypted S3 buckets) are not automatically fixed. What's the correct approach?

**A:** Moving an account to a new OU changes its SCPs **going forward** (new actions are evaluated against new SCPs), but **does not remediate existing non-compliant resources.** (1) Before moving: Run a compliance check (Config rules) to verify the account meets Production standards. (2) Remediate all non-compliance BEFORE moving (encrypt buckets, fix security groups, etc.). (3) After moving: Detective guardrails (Config rules) will flag remaining issues. (4) **Best practice:** Use a "Pre-Production" OU with production SCPs but detective-only monitoring for a transition period. **Key concept:** SCPs are preventive (block future actions) not corrective (don't fix existing state).

---

**Q7:** A company has Organization consolidated billing with RI sharing enabled. Team A in Account A purchased Reserved Instances for c5.xlarge. Team B in Account B (sandbox) is running c5.xlarge On-Demand instances and consuming Team A's RI benefits, increasing Team A's effective costs. How should they fix this?

**A:** **Disable RI/SP sharing for Account B (sandbox).** The payer account (management account) can disable RI and Savings Plan discount sharing for specific linked accounts. Once disabled, Account B's instances won't consume Account A's RIs. **Alternative:** Turn off RI sharing at the organization level and enable only for production accounts. **Key concept:** RI/SP sharing is enabled by default in Organizations. You must explicitly disable it for accounts that shouldn't benefit (sandboxes, dev accounts with irregular usage).

---

### Common Exam Traps & Pitfalls

1. **SCPs DO NOT apply to the management account.** If the question involves the management account doing something, SCPs won't prevent it. This is tested heavily.

2. **SCPs do NOT grant permissions.** They only restrict. You still need IAM policies to grant access. SCP + IAM policy = intersection.

3. **SCPs DO NOT apply to service-linked roles.** Service-linked roles can perform their actions regardless of SCPs.

4. **Organization root ≠ account root user.** Organization root is the top of the OU hierarchy. Account root user is the root login for an individual account. Different concepts.

5. **Account closure ≠ immediate deletion.** Closed accounts enter a 90-day suspended period. Move them to a "Suspended" OU with deny-all SCP during this period.

6. **RI/SP sharing is ON by default.** If the question mentions sandbox accounts consuming production RIs, the answer involves disabling sharing.

7. **Control Tower Landing Zone ≠ Organizations.** Control Tower uses Organizations underneath, but adds automation, guardrails, and compliance features. Not all Organizations customers use Control Tower.

8. **Permission Sets are NOT IAM policies.** They're templates that become IAM roles when deployed to accounts. The role is what's assumed.

9. **Tag Policies don't enforce automatically.** By default, they only report non-compliance. You must enable enforcement for specific tag keys/resources.

10. **Cross-account access requires BOTH sides to allow.** Source account needs permission to assume role, AND target account's role trust policy must allow the source. Missing either side = denied.

11. **RAM sharing within Organizations doesn't require acceptance.** But sharing with accounts outside the organization requires invitation acceptance.

12. **Organization trails log all accounts automatically.** Member accounts cannot delete or modify the organization trail. But they CAN create additional trails in their own account (which adds cost).

---

## 7. Cheat Sheet

### Must-Know Facts

| Fact | Detail |
|------|--------|
| SCPs apply to | All member accounts (NOT management account) |
| SCPs don't affect | Management account, service-linked roles |
| SCP max size | 5,120 characters |
| SCPs per entity | 5 max |
| OU nesting depth | 5 levels |
| Default account limit | 10 (soft limit, request increase) |
| Account closure | 90-day suspension, then permanently closed |
| RI/SP sharing | ON by default across organization |
| Identity Center | Requires Organizations, uses Permission Sets |
| Control Tower guardrails | Preventive (SCP), Detective (Config), Proactive (CFN Hooks) |
| Organization Trail | Logs ALL member accounts, members can't delete |
| RAM within org | No acceptance needed (trusted access) |
| StackSets service-managed | Auto-deploys to new accounts in OU |
| Management account best practice | NO workloads, minimal users, strong MFA |
| Consolidated billing | Combined volume = tier discounts |
| Delegated administrator | Offload service management from management account |

### Key Differentiators

| Feature | Organizations | Control Tower | IAM Identity Center |
|---------|--------------|---------------|---------------------|
| Purpose | Account governance & structure | Automated multi-account setup | Centralized user access |
| Scope | Policy-based controls (SCPs) | Guardrails + compliance dashboard | Authentication + authorization |
| Automation | Manual policy management | Automated landing zone | Automated role provisioning |
| Prerequisite | None | Organizations | Organizations |
| Best for | Custom governance | Quick compliant setup | User access management |

### Service Selection: "If the question says X, think Y"

| Question Keywords | Service/Feature |
|-------------------|-----------------|
| "Restrict maximum permissions across accounts" | SCPs |
| "Prevent actions organization-wide" | SCPs (Deny) |
| "Automated multi-account setup, guardrails" | Control Tower |
| "Provision new accounts with standards" | Control Tower Account Factory / AFT |
| "Single sign-on across accounts" | IAM Identity Center |
| "Share resources across accounts (TGW, subnets)" | RAM |
| "Deploy stacks to multiple accounts" | CloudFormation StackSets |
| "Centralized logging across all accounts" | Organization Trail + Log Archive account |
| "Centralized security findings" | Security Hub (delegated admin) |
| "Enforce tagging standards" | Tag Policies |
| "Automated backup across accounts" | Backup Policies |
| "Cross-account IAM access" | IAM role assumption (sts:AssumeRole) |
| "Data sovereignty / restrict regions" | SCP with aws:RequestedRegion condition |
| "Consolidated billing, volume discounts" | Organizations (Consolidated Billing) |
| "Reserve capacity across org" | RI/SP with sharing enabled |
| "Detect non-compliance across accounts" | AWS Config Organization Rules |
| "Delegate security service management" | Delegated Administrator |

### Decision Flowchart

```
Question mentions...                                      → Think...
────────────────────────────────────────────────────────────────────────
"restrict permissions", "prevent actions in accounts"    → SCPs
"management account can't be restricted"                 → SCP doesn't apply to management account
"automated setup", "landing zone", "guardrails"          → Control Tower
"new account auto-configuration"                         → StackSets (auto-deploy) or Control Tower
"SSO", "single sign-on", "one login for all accounts"   → IAM Identity Center
"share Transit Gateway / subnets / resources"            → RAM
"centralized logging / audit trail"                      → Organization Trail → Log Archive
"cross-account access to resources"                      → IAM role assumption or resource-based policy
"enforce regions", "data residency"                      → SCP with region condition
"combined billing", "volume discount"                    → Consolidated Billing
"close account", "offboard account"                      → Move to Suspended OU → Close
"compliance dashboard", "drift detection"                → Control Tower Dashboard
"delegate security management"                           → Delegated Administrator
"break glass / emergency access"                         → SCP condition exempting break-glass role
```

---

## 8. Deep Dive: 10 More Tricky Scenario-Based Questions

**Q1:** A financial institution has a regulatory requirement that certain accounts can ONLY use AWS services that are FedRAMP authorized. They use Organizations with 200+ accounts. Most accounts can use any service, but 15 accounts in the "Regulated" OU must be restricted. How should they implement this?

**A:** Use an **Allow-List SCP** attached to the "Regulated" OU. Remove the default `FullAWSAccess` SCP from that OU and attach a custom SCP that explicitly allows ONLY the FedRAMP-authorized services. All other services are implicitly denied. The remaining OUs keep `FullAWSAccess` and operate normally. **Why not a Deny-list:** The list of non-FedRAMP services is larger and changes frequently. An allow-list of approved services is more secure (default-deny) and easier to audit. **Key concept:** Allow-list SCPs = remove FullAWSAccess, explicitly allow specific services. This is the correct approach for strict compliance OUs.

---

**Q2:** A company has 300 accounts. They want to ensure every account has GuardDuty enabled, findings sent to a central Security account, and cannot be disabled by account administrators. What's the architecture?

**A:** (1) Enable **GuardDuty with delegated administrator** (Security Tooling account). (2) Enable auto-enable for new members. (3) Existing accounts: Use the delegated admin to enable GuardDuty for all current members. (4) Attach an SCP to the organization Root denying `guardduty:DeleteDetector`, `guardduty:DisassociateFromMasterAccount`, `guardduty:UpdateDetector` (to prevent disabling). (5) Findings automatically flow to the delegated admin account. **Key design:** Delegated admin + auto-enable + SCP to prevent disabling = belt-and-suspenders approach the exam loves.

---

**Q3:** A company uses Control Tower. They need to allow one specific account (their CI/CD account) to launch EC2 instances in any region (for global deployments), while all other accounts are restricted to two regions. How?

**A:** Control Tower's region deny guardrail is implemented as an SCP at the OU level. **Solution:** (1) Create a separate OU (e.g., "Global-Infrastructure") with the region deny guardrail NOT enabled. (2) Place the CI/CD account in that OU. (3) Apply other necessary guardrails to that OU. **Alternative (SCP condition):** If using custom SCPs, add a condition excluding the CI/CD account by account ID from the region deny. **Wrong answer:** "Disable the guardrail for the entire org" — this removes protection from all accounts. **Key concept:** OU structure determines policy application. Accounts needing different rules go in different OUs.

---

**Q4:** A company wants developers to be able to create any resource in their sandbox accounts but wants to cap monthly spending at $500 per account. If spending exceeds $500, all resource creation should be automatically blocked. How?

**A:** (1) Create **AWS Budgets** with a $500 monthly budget per sandbox account. (2) Configure a **Budget Action** that applies an SCP or IAM policy to deny resource creation when the threshold is breached. (3) Alternatively: Budget Action that attaches a deny-all IAM policy to the account's users/roles. **Important detail:** Budget Actions can trigger IAM policy attachments or SCP attachments, or run SSM Automation documents. **Limitation:** Budgets are evaluated periodically (not real-time) — spending may slightly exceed the threshold before action triggers. **For exam:** AWS Budgets + Budget Actions = automated cost-based access control.

---

**Q5:** A company migrated to multi-account but some applications need to access DynamoDB tables in another account. They currently use IAM users with long-term access keys. Security team wants to eliminate access keys. What's the best cross-account access pattern?

**A:** (1) In the target account (DynamoDB account): Create an IAM role with a trust policy allowing the source account. Attach a policy granting DynamoDB access. (2) In the source account: Applications assume the cross-account role via `sts:AssumeRole`. (3) For EC2/ECS/Lambda workloads: Attach an instance profile / execution role with permission to assume the cross-account role. (4) Credentials are temporary (STS tokens). **No long-term keys needed.** **Why not resource-based policy:** DynamoDB does NOT support resource-based policies. Cross-account access to DynamoDB always requires role assumption. **Key distinction:** Services supporting resource-based policies (S3, SQS, KMS, Lambda) don't require role assumption. Services without them (DynamoDB, EC2) do.

---

**Q6:** A company discovers that an account they closed 30 days ago still shows as "Suspended" and they need to recover data from an S3 bucket in that account. Can they do this?

**A:** **Yes**, during the 90-day suspension period, the account can be **reopened.** Contact AWS Support or use the Organizations console to reopen the account. Once reopened, access the S3 data. After reopening, either close it again (resets 90-day clock) or keep it active. **After 90 days:** The account is permanently deleted and all data is unrecoverable. **Key facts:** (1) Closed ≠ deleted for 90 days. (2) Accounts can be reopened during suspension. (3) After 90 days, data is gone forever. (4) Move closed accounts to "Suspended OU" with deny-all SCP to prevent any activity during suspension.

---

**Q7:** A company uses IAM Identity Center with Okta as the external IdP. A new employee is added to Okta on Monday, but they can't access their assigned AWS accounts until Tuesday. What's the likely issue?

**A:** **SCIM provisioning delay or not configured.** If SCIM (System for Cross-domain Identity Management) automatic provisioning isn't set up, user/group syncing between Okta and IAM Identity Center may be manual or scheduled (not real-time). **Fixes:** (1) Enable SCIM provisioning for automatic user/group sync. (2) If SCIM is enabled, check if there's a sync interval delay. (3) Verify the user is added to a group in Okta that's mapped to a Permission Set in Identity Center. (4) Manual workaround: Trigger a sync or manually create the user in Identity Center. **Key concept:** External IdP + Identity Center requires SCIM for automatic provisioning. Without it, users must be manually created/synced.

---

**Q8:** A company has a complex OU structure: Root → Corp OU → Regional OU (US, EU, APAC) → Environment OU (Prod, Dev). They attach an SCP at Corp OU allowing all services, an SCP at Regional-EU OU denying non-EU regions, and an SCP at Environment-Prod OU denying deletion of resources. An account is in Root → Corp → Regional-EU → Prod. What's the effective SCP?

**A:** SCPs **inherit through the hierarchy** and the effective permission is the **intersection of all SCPs from Root to the account:** (1) Root: FullAWSAccess (allows all). (2) Corp OU: Allow all. (3) Regional-EU OU: Deny non-EU regions. (4) Prod OU: Deny deletions. **Effective = All allowed, EXCEPT non-EU regions are denied AND deletions are denied.** Both deny SCPs apply (inheritance stacks). An action must be allowed at EVERY level of the hierarchy. **Key concept:** SCPs don't "override" at lower levels — they accumulate. Deny at any level = denied. Allow must exist at every level (for allow-list strategy).

---

**Q9:** A company wants to prevent member accounts from creating IAM users with long-term access keys (enforce use of IAM roles and temporary credentials). But some legacy applications require access keys during a migration period. How do they balance this?

**A:** (1) Attach an SCP denying `iam:CreateAccessKey` to most OUs. (2) Create a "Legacy Migration" OU with this SCP NOT attached (or with an exception condition). (3) Place accounts with legacy applications in the Legacy OU temporarily. (4) Set a timeline to migrate legacy apps to IAM roles. (5) Once migrated, move accounts back to the standard OU (SCP applied). **Alternative (condition-based):** SCP denies `iam:CreateAccessKey` except for a specific "MigrationAdmin" role via condition. **Best practice:** Combine with AWS Config rule to detect existing access keys and report them for remediation.

---

**Q10:** A company has separate AWS accounts for each business unit. Each account has its own VPC. They acquire another company with its own AWS Organization (3 accounts). How do they integrate the acquired company's accounts?

**A:** **Step-by-step migration:** (1) The acquired company's accounts must **leave their current organization** first (one org per account). (2) Send an **invitation** from the acquiring organization to each account. (3) The acquired accounts accept the invitation and join the new organization. (4) Move accounts into appropriate OUs. (5) SCPs immediately apply to joined accounts. **Important considerations:** (a) Consolidated billing switches immediately (pricing may change). (b) RIs/SPs may shift. (c) Existing cross-account trust policies may need updates. (d) Control Tower: Use "Enroll Account" to bring under Control Tower governance. **Key fact:** An account can only belong to ONE organization. Migration requires leave → join sequence. Cannot be in two simultaneously.

---

## 9. Service Comparison Tables

### Multi-Account Governance Tools

| Dimension | SCPs | IAM Policies | Permission Boundaries | Resource Policies |
|-----------|------|-------------|----------------------|-------------------|
| **Scope** | Organization/OU/Account | User/Role/Group | User/Role | Specific resource |
| **Function** | Maximum permissions (ceiling) | Grant permissions | Maximum for a user/role | Grant cross-account access |
| **Denies override?** | Yes (except management account) | Yes | Yes | Yes |
| **Grants permissions?** | No (only restricts) | Yes | No (only restricts) | Yes |
| **Applies to root user?** | Yes (in member accounts) | No (root is unrestricted by IAM) | No | Depends |
| **Cross-account?** | Org-wide | Per-account | Per-account | Cross-account specific |
| **Exam Tip** | "org-wide restriction" | "grant specific access" | "delegate safely" | "cross-account to resource" |

### Account Provisioning Methods

| Dimension | Control Tower Account Factory | AFT (Account Factory for Terraform) | CloudFormation StackSets | Organizations CreateAccount API |
|-----------|------------------------------|--------------------------------------|--------------------------|-------------------------------|
| **Automation** | Service Catalog-based | Terraform pipeline (CodePipeline) | Stack deployment | API call only |
| **Customization** | Blueprint templates | Full Terraform IaC | CloudFormation templates | None (bare account) |
| **Guardrails** | Automatic (Control Tower) | Automatic (Control Tower) | Manual (deploy what you want) | None |
| **Compliance** | Built-in | Built-in | Self-managed | None |
| **Best For** | Console-based teams | Terraform-native teams | Baseline resource deployment | Programmatic, custom pipelines |
| **Exam Tip** | "automated, compliant accounts" | "Terraform multi-account" | "deploy resources to all accounts" | "just create the account" |

### Cross-Account Access Methods

| Dimension | IAM Role Assumption | Resource-Based Policy | RAM | VPC Peering/TGW |
|-----------|--------------------|-----------------------|-----|-----------------|
| **Mechanism** | STS:AssumeRole (temporary creds) | Policy on resource allows external principal | Share resource metadata | Network-level connectivity |
| **Services** | All services | S3, SQS, SNS, KMS, Lambda, Secrets Manager, ECR | TGW, Subnets, Route53, License, Aurora, CodeBuild | VPC network access |
| **Trust Model** | Role trust policy + source IAM policy | Resource policy specifies allowed principals | Share with org/OU/account | Peering request acceptance |
| **Data Stays In** | Target account | Target account | Owning account | Each account's VPC |
| **Exam Tip** | "assume role", "temporary credentials" | "bucket policy allows Account B" | "share Transit Gateway" | "connect VPCs cross-account" |

### Centralized Security Services Deployment

| Service | Deployment Method | Findings Location | Prevention/Detection |
|---------|------------------|-------------------|---------------------|
| GuardDuty | Delegated admin + auto-enable | Centralized in delegated admin | Detection (threat findings) |
| Security Hub | Delegated admin + auto-enable | Aggregated in delegated admin | Detection (compliance + findings) |
| Config | Organization rules + aggregator | Aggregated via aggregator | Detection + auto-remediation |
| CloudTrail | Organization Trail | Log Archive S3 bucket | Audit (API logging) |
| Firewall Manager | Delegated admin | Security account | Prevention (WAF, SG, NF rules) |
| IAM Access Analyzer | Organization analyzer | Analyzer account | Detection (external access, unused access) |
| Macie | Delegated admin | Security account | Detection (sensitive data in S3) |

---

## 10. When to Pick Service A over Service B — Exam Keywords

### Pick SCPs over IAM Policies when:
- "organization-wide restriction"
- "prevent ALL users/roles in an account"
- "even administrators cannot do X"
- "hard guardrail"
- "restrict root user in member accounts"
- **Exam signal:** Broad, account-level restriction that can't be bypassed by any IAM entity

### Pick IAM Policies over SCPs when:
- "grant access to specific users/roles"
- "allow specific actions"
- "fine-grained permissions"
- "conditional access (time, IP, MFA)"
- **Exam signal:** Granting (not restricting), specific users/roles, conditional

### Pick Control Tower over bare Organizations when:
- "automated setup", "quick start"
- "pre-built guardrails", "compliance dashboard"
- "new organization", "landing zone"
- "least operational effort"
- "detect drift"
- **Exam signal:** Automation + compliance + minimal custom work

### Pick Organizations (alone) over Control Tower when:
- "existing complex organization" (migration to Control Tower can be disruptive)
- "highly customized governance" (beyond Control Tower's guardrails)
- "only need SCPs and consolidated billing"
- **Exam signal:** Simple governance or existing setup that works

### Pick RAM over cross-account role assumption when:
- "share infrastructure resources (TGW, subnets)"
- "avoid duplicating resources in each account"
- "resource stays in one account, others use it"
- **Exam signal:** Infrastructure sharing without moving or copying resources

### Pick IAM Identity Center over per-account IAM users when:
- "single sign-on", "centralized access"
- "multiple accounts"
- "external identity provider (Okta, Azure AD)"
- "temporary credentials"
- "manage access centrally"
- **Exam signal:** Multi-account human user access management

### Pick StackSets over Control Tower Account Factory when:
- "deploy specific resources to existing accounts"
- "don't need full Control Tower"
- "custom baseline (not Control Tower's)"
- "deploy to specific accounts/regions selectively"
- **Exam signal:** Resource deployment to existing accounts, not account creation

### Pick Control Tower Account Factory over StackSets when:
- "create new account with full governance"
- "include networking, guardrails, identity automatically"
- "standardized account creation process"
- **Exam signal:** New account provisioning (not just resource deployment)

### Pick Delegated Administrator over Management Account when:
- "reduce risk to management account"
- "dedicated security team manages GuardDuty/Security Hub"
- "separate billing/governance from security operations"
- **Exam signal:** Best practice for running security services (never from management account)

### Pick Organization Backup Policies over per-account backups when:
- "consistent backup across all accounts"
- "compliance requires centralized backup"
- "member accounts cannot modify backup plans"
- **Exam signal:** Mandatory, centralized, tamper-proof backup

---

## 11. "Gotcha" Differences the Exam Tests

### SCP Gotchas
1. **SCPs DO NOT apply to the management account.** This is the #1 trap. If a question asks about restricting the management account → SCPs won't work. Need IAM policies, MFA, or operational procedures.
2. **SCPs do NOT grant permissions.** An SCP with only Allow statements doesn't give access — it just sets the ceiling. Users still need IAM policies to actually do things.
3. **SCPs apply to the ROOT USER in member accounts.** Unlike IAM policies (which don't restrict root), SCPs restrict even the root user in member accounts. This is an account-level control that IAM cannot provide.
4. **SCPs don't affect service-linked roles.** If a service uses a service-linked role to perform actions (e.g., AWS Config creates `AWSServiceRoleForConfig`), SCPs won't block those actions. This is by design.
5. **SCP character limit (5,120) is small.** Complex deny lists can hit this quickly. Workaround: Use multiple SCPs (up to 5 per entity) or use NotAction patterns to reduce size.
6. **SCP inheritance is AND logic (intersection).** An action must be allowed at every level from Root → OU → Account. Deny at any level = denied. This is different from IAM policy evaluation.
7. **Removing FullAWSAccess without replacement locks you out.** If you accidentally remove the allow-all SCP without attaching a new allow SCP → implicit deny = account can't do anything.

### Organizations Gotchas
8. **An account can only belong to ONE organization.** To move between orgs: leave → join. Cannot be members of two simultaneously.
9. **Inviting existing accounts vs creating new:** Invited accounts DON'T automatically get `OrganizationAccountAccessRole`. Only accounts CREATED by the organization get this role. For invited accounts, you must create it manually.
10. **Organization features: "All Features" vs "Consolidated Billing Only."** If you started with "Consolidated Billing Only," enabling "All Features" requires acceptance by all member accounts. Can't be forced.
11. **Account limits start at 10.** New organizations can only have 10 accounts by default. Must request increase (can take days). Plan ahead for large migrations.
12. **Closing an account doesn't immediately free its resources/quotas.** 90-day suspension period — the account ID remains reserved, email can't be reused for new account until permanently deleted.

### Control Tower Gotchas
13. **Control Tower manages specific OUs.** It doesn't automatically govern ALL OUs/accounts in the organization. You must enroll OUs/accounts explicitly.
14. **Drift detection ≠ auto-remediation.** Control Tower detects drift and alerts you, but doesn't automatically fix it (for most guardrails). You must manually resolve.
15. **Control Tower doesn't support all regions at launch.** It governs "home region" + additional governed regions. Resources in non-governed regions aren't monitored by Control Tower.
16. **Account Factory Customization has quotas.** Limited number of custom blueprints and products. Complex customizations may need AFT (Terraform) instead.

### Identity Center Gotchas
17. **Only ONE Identity Center instance per organization.** You cannot have multiple instances (one per region or per OU). It's a single, organization-wide instance.
18. **Permission Sets become IAM roles.** When you update a Permission Set, it must be re-provisioned to target accounts. Updates aren't instantaneous.
19. **Session duration is configurable (1-12 hours).** But if the question says "credentials expire too quickly" → increase session duration in Permission Set configuration.
20. **Delegated admin for Identity Center:** Once delegated, management account retains limited Identity Center capabilities. The delegated admin cannot manage the management account's access.

### Cross-Account Gotchas
21. **External ID is for third-party access, NOT for internal cross-account.** If the question involves cross-account within your own org → external ID is unnecessary. External ID prevents confused deputy in third-party scenarios.
22. **Cross-account S3 access: Object ownership matters.** If Account A puts objects in Account B's bucket, Account B may NOT own those objects (unless Bucket Owner Enforced is set). This affects permissions.
23. **RAM sharing within org = no invitation needed.** But only if trusted access for RAM is enabled in Organizations. Without it, invitations are still required even within the same org.
24. **VPC Peering vs TGW:** Peering = direct 1:1, non-transitive. TGW = hub-and-spoke, transitive routing. For >3 accounts needing connectivity → TGW. But TGW has per-GB data processing charges.

### Billing Gotchas
25. **Free tier is per ORGANIZATION, not per account.** Creating 10 accounts doesn't give you 10× free tier. This trips people up.
26. **RI/SP benefits apply to matching instances anywhere in the org (by default).** A sandbox running the same instance type will consume your production RI. Disable sharing for non-production accounts.
27. **Credits apply to the originating account first.** If Account A earned a credit, it applies to Account A's charges first. Excess credits then apply to the org.

---

## 12. Decision Tree: Choosing the Right Multi-Account Feature

```
START: What is the primary need?
│
├─── Need to RESTRICT what accounts can do?
│    ├─── Restrict all entities in an account (including root)? → SCP
│    │    ├─── Block specific services/actions? → Deny SCP
│    │    ├─── Allow ONLY specific services? → Allow-list SCP (remove FullAWSAccess)
│    │    ├─── Restrict to specific regions? → SCP with aws:RequestedRegion condition
│    │    └─── Prevent security service disablement? → SCP denying delete/stop actions
│    ├─── Restrict specific users/roles only? → IAM Policies / Permission Boundaries
│    └─── Restrict access TO your resources from external? → Resource Control Policies (RCPs)
│
├─── Need to SET UP multi-account environment?
│    ├─── New environment, want best practices fast? → Control Tower
│    │    ├─── Need Terraform-based provisioning? → AFT (Account Factory for Terraform)
│    │    └─── Need custom blueprints per account? → Account Factory Customization
│    ├─── Existing org, just need structure + policies? → Organizations (manual)
│    └─── Deploy baseline resources to all accounts? → CloudFormation StackSets
│
├─── Need to MANAGE USER ACCESS across accounts?
│    ├─── SSO with external IdP (Okta, Azure AD)? → IAM Identity Center
│    ├─── Programmatic cross-account access? → IAM Role Assumption
│    └─── Temporary elevated access (break-glass)? → SCP condition + protected role
│
├─── Need to SHARE RESOURCES across accounts?
│    ├─── Network (TGW, subnets, Route53)? → RAM
│    ├─── Data (S3 buckets, KMS keys)? → Resource-Based Policies
│    ├─── AMIs, snapshots? → AMI/Snapshot sharing (modify permissions)
│    └─── Entire VPC connectivity? → VPC Peering (few) or TGW (many)
│
├─── Need CENTRALIZED SECURITY?
│    ├─── Threat detection across all accounts? → GuardDuty (delegated admin)
│    ├─── Compliance/findings aggregation? → Security Hub (delegated admin)
│    ├─── API audit logging? → Organization Trail → Log Archive
│    ├─── Configuration compliance? → Config (organization rules + aggregator)
│    ├─── Firewall/WAF rules across accounts? → Firewall Manager
│    └─── Detect external access to resources? → IAM Access Analyzer (org-level)
│
├─── Need COST MANAGEMENT?
│    ├─── Combined billing + volume discounts? → Consolidated Billing (automatic)
│    ├─── Per-account cost tracking? → Cost Allocation Tags + Cost Explorer
│    ├─── Budget enforcement with auto-action? → AWS Budgets + Budget Actions
│    ├─── RI/SP shared across accounts? → Enabled by default (disable per-account if needed)
│    └─── Cap spending in sandbox accounts? → Budgets + IAM/SCP actions
│
└─── Need COMPLIANCE GOVERNANCE?
     ├─── Enforce tagging standards? → Tag Policies
     ├─── Enforce backup across accounts? → Backup Policies
     ├─── Opt out of AI service data use? → AI Services Opt-Out Policies
     ├─── Compliance dashboard + drift? → Control Tower
     └─── Custom compliance rules? → Config Organization Rules
```
