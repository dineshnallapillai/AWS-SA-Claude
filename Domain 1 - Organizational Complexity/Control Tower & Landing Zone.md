# AWS Control Tower & Landing Zones

## 1. Core Concepts & Theory

---

### What is a Landing Zone?

**Definition:** A well-architected, multi-account AWS environment that serves as the starting point for deploying workloads and applications. It provides a baseline of identity, access, governance, security, and networking.

**A landing zone includes:**
- Multi-account structure (Organizations, OUs)
- Identity and access management (IAM Identity Center)
- Centralized logging and auditing (CloudTrail, Config)
- Network connectivity (VPC design, DNS, connectivity)
- Security baselines (guardrails, detective controls)
- Governance automation (account provisioning, policy enforcement)

**Landing zone approaches:**
1. **AWS Control Tower** — Fully managed, opinionated, automated
2. **Custom Landing Zone** — Built with Organizations, CloudFormation, Terraform (more flexible, more effort)
3. **AWS Landing Zone Solution (deprecated)** — Older AWS solution, replaced by Control Tower

---

### AWS Control Tower

**What it is:** An AWS managed service that automates the setup and governance of a secure, multi-account AWS environment (landing zone) based on AWS best practices.

#### Core Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Control Tower Landing Zone                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  Management Account (Control Tower home)                                  │
│  ├── AWS Organizations (All Features enabled)                            │
│  ├── AWS Control Tower service                                           │
│  ├── AWS Service Catalog (Account Factory)                               │
│  ├── AWS CloudFormation StackSets                                        │
│  └── AWS IAM Identity Center                                             │
│                                                                           │
│  Security OU (Core OU)                                                    │
│  ├── Log Archive Account                                                  │
│  │   ├── CloudTrail logs (all accounts) → S3                            │
│  │   ├── Config logs (all accounts) → S3                                │
│  │   └── Access logging for the log buckets                              │
│  └── Audit Account (Security Tooling)                                    │
│      ├── SNS topics for security notifications                           │
│      ├── Cross-account audit roles (read-only to all accounts)           │
│      ├── AWS Config aggregator                                           │
│      └── Programmatic audit access to all accounts                       │
│                                                                           │
│  Sandbox OU (optional, created during setup)                              │
│                                                                           │
│  Custom OUs (you create these)                                            │
│  ├── Workloads OU → Production / Staging / Development                   │
│  ├── Infrastructure OU → Network / Shared Services                       │
│  └── Policy Staging OU (test guardrails before production)               │
│                                                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

#### Foundational Accounts

| Account | Purpose | Created By |
|---------|---------|-----------|
| **Management Account** | Hosts Control Tower, Organizations, billing. NO workloads | You (existing or new) |
| **Log Archive** | Centralized, immutable log storage (CloudTrail, Config). Highly restricted access | Control Tower (automatic) |
| **Audit (Security Tooling)** | Security team access, cross-account read-only roles, SNS alerts, Config aggregator | Control Tower (automatic) |

**Key facts about foundational accounts:**
- Log Archive: S3 buckets with lifecycle policies, bucket policies preventing deletion, no direct user access in normal operations
- Audit: Has `AWSControlTowerExecution` role in all member accounts (cross-account access)
- Management: Should NEVER run workloads (blast radius, SCP exemption)

---

### Controls (Guardrails)

**What they are:** Pre-packaged governance rules that Control Tower applies across your landing zone. They enforce policies or detect non-compliance.

#### Control Types (Implementation)

| Type | Mechanism | When Evaluated | Effect |
|------|-----------|----------------|--------|
| **Preventive** | SCP | At request time | Blocks non-compliant actions before they occur |
| **Detective** | AWS Config Rules | Continuous / periodic | Detects and flags non-compliant resources |
| **Proactive** | CloudFormation Hooks | At resource creation (IaC) | Blocks non-compliant resource provisioning via CloudFormation |

#### Control Behaviors (Enforcement Level)

| Behavior | Description | Can Disable? |
|----------|-------------|-------------|
| **Mandatory** | Essential for landing zone operation | No — always enabled, cannot be disabled |
| **Strongly Recommended** | AWS best practices for well-architected environments | Yes — enabled by default, can be disabled |
| **Elective** | Optional governance rules | Yes — disabled by default, opt-in |

#### Examples of Key Controls

**Mandatory (cannot disable):**
- Disallow changes to CloudTrail configuration
- Disallow deletion of Log Archive resources
- Disallow changes to IAM roles set up by Control Tower
- Disallow changes to AWS Config rules
- Enable CloudTrail in all accounts

**Strongly Recommended:**
- Disallow internet connection through RDP (port 3389)
- Disallow internet connection through SSH (port 22)
- Enable encryption for EBS volumes attached to EC2 instances
- Disallow public read access to S3 buckets
- Enable AWS Config in all accounts
- Disallow changes to S3 bucket policies for CloudTrail logs

**Elective:**
- Disallow MFA delete for S3 buckets
- Disallow cross-region networking
- Restrict regions for AWS services
- Disallow VPN connections
- Detect whether public IP addresses are auto-assigned to EC2

#### Control Application

- Controls are applied at the **OU level** (not individual accounts)
- All accounts within the OU inherit the controls
- Different OUs can have different sets of controls
- A control enabled on a parent OU applies to child OUs (inheritance)

#### Proactive Controls (CloudFormation Hooks)

- Evaluate CloudFormation templates BEFORE resource creation
- Block non-compliant resources from being created via IaC
- Examples: "Require encryption on RDS instances", "Require specific VPC configuration"
- Only applies to CloudFormation-managed resources (not CLI/Console direct creation — that's handled by preventive SCPs)
- **Key distinction:** Proactive = pre-deployment check. Preventive = runtime block. Detective = post-deployment detection.

---

### Account Factory

**What it is:** Automated, standardized account provisioning within Control Tower.

#### Account Factory via Service Catalog

- Uses AWS Service Catalog to provision accounts
- Pre-configured account templates (blueprints)
- Users request accounts via Service Catalog portal
- Accounts automatically enrolled in Control Tower governance
- Networking pre-configured (VPC, subnets, internet access options)

**Account Factory configuration options:**
- Account name, email, IAM Identity Center user
- OU placement
- VPC configuration (CIDR, regions, subnets, internet connectivity)
- Enable/disable VPC creation

#### Account Factory for Terraform (AFT)

**What it is:** Terraform-based account provisioning pipeline for Control Tower, deployed as a Terraform module.

**Architecture:**
```
Developer commits account request (Terraform) →
  CodePipeline → CodeBuild →
    Create account (Organizations API) →
    Enroll in Control Tower →
    Apply account customizations (Terraform) →
    Apply global customizations →
    Account ready
```

**Key features:**
- GitOps workflow (account definitions in Git)
- Customization pipeline (per-account and global customizations)
- Feature flags for customization stages
- Supports Terraform modules for post-creation configuration
- Runs in a dedicated AFT management account

**AFT pipeline stages:**
1. **Account provisioning** — Creates the AWS account
2. **Control Tower enrollment** — Enrolls account under CT governance
3. **Global customizations** — Applied to ALL accounts (baseline)
4. **Account customizations** — Per-account specific configurations
5. **Account provisioning customizations** — One-time setup

#### Account Factory Customization (AFC)

- Uses Service Catalog blueprints (CloudFormation templates)
- Applied when accounts are created or updated
- Simpler than AFT but less flexible
- Good for: Standardized VPC, IAM roles, security baseline via CloudFormation

#### Account Factory Comparison

| Dimension | Account Factory (Service Catalog) | AFT (Terraform) | AFC (Blueprints) |
|-----------|-----------------------------------|------------------|-------------------|
| **IaC Tool** | Service Catalog / Console | Terraform | CloudFormation (Service Catalog) |
| **Automation** | Console or API-driven | Full CI/CD pipeline (GitOps) | Template-driven |
| **Customization** | VPC config, OU, user | Full Terraform (any resource) | CloudFormation templates |
| **Complexity** | Low | High (requires AFT infrastructure) | Medium |
| **Best for** | Simple, console-based teams | Terraform-native, large-scale automation | CloudFormation teams wanting blueprints |
| **Pipeline** | None (direct provisioning) | CodePipeline + CodeBuild | Service Catalog provisioning |

---

### Landing Zone Configuration

#### Region Selection

- **Home Region:** Region where Control Tower is deployed (cannot be changed after setup)
- **Governed Regions:** Additional regions where guardrails are enforced
- Resources in non-governed regions are NOT monitored by Control Tower
- **Region deny control:** Optional control that blocks activity in non-governed regions

#### VPC Configuration (Default)

Control Tower's Account Factory can create VPCs with:
- Configurable CIDR range
- Public and private subnets across AZs
- NAT Gateway (optional)
- Internet Gateway (optional)
- VPC can be opted out entirely (for centralized networking patterns)

**Best practice for enterprise:** Disable Account Factory VPC creation and use a centralized networking account with Transit Gateway (shared via RAM).

#### IAM Identity Center Integration

- Control Tower sets up IAM Identity Center automatically
- Creates default Permission Sets:
  - `AWSAdministratorAccess` — Full admin access
  - `AWSPowerUserAccess` — Power user (no IAM)
  - `AWSReadOnlyAccess` — Read-only
  - `AWSServiceCatalogEndUserAccess` — Account Factory access
- Users/groups assigned to accounts via Identity Center
- SSO portal for accessing all governed accounts

---

### Drift Detection & Resolution

**What is drift?** When your landing zone configuration diverges from the expected baseline established by Control Tower.

#### Types of Drift

| Drift Type | Example | Impact |
|-----------|---------|--------|
| **SCP drift** | SCP modified, detached, or deleted | Preventive controls may not function |
| **Account drift** | Account moved out of registered OU | Controls may not apply to account |
| **Control Tower role drift** | CT-managed IAM roles modified | CT operations may fail |
| **OU drift** | OU deleted that was registered with CT | CT can't govern accounts in that OU |
| **CloudTrail drift** | Org trail modified or deleted | Logging gaps |
| **Config drift** | Config rules modified or deleted | Detective controls may not function |

#### Drift Detection

- Control Tower continuously monitors for drift
- Drift detected → **SNS notification** to Audit account
- Dashboard shows drift status per OU/account
- Some drift auto-detected, some requires periodic checks

#### Drift Resolution

- **Automatic:** Some drift types can be resolved via "Re-register OU" or "Repair account"
- **Manual:** Some drift requires you to fix the underlying issue first, then reset
- Landing Zone repair/update may be needed for infrastructure drift
- **Critical:** Drift does NOT auto-remediate in most cases — you must take action

---

### Customizations for Control Tower (CfCT)

**What it is:** An AWS solution (not a service) that extends Control Tower with custom CloudFormation templates and SCPs deployed via a CI/CD pipeline.

**Architecture:**
```
S3 bucket (manifest + templates) →
  CodePipeline →
    CloudFormation StackSets →
      Deploy to target OUs/accounts
```

**Use cases:**
- Deploy custom security baselines beyond CT's built-in controls
- Deploy network configurations (Transit Gateway attachments)
- Deploy additional SCPs beyond CT's preventive controls
- Apply custom Config rules
- Deploy monitoring/alerting (CloudWatch alarms, dashboards)

**Key features:**
- Manifest file defines what to deploy where (YAML)
- Supports both StackSets (multi-account) and SCPs
- Triggers on S3 upload or scheduled
- Integrates with Control Tower lifecycle events

---

### Control Tower Lifecycle Events

**What they are:** CloudTrail events emitted when Control Tower operations occur. Used for automation.

| Event | Trigger |
|-------|---------|
| `CreateManagedAccount` | New account created via Account Factory |
| `UpdateManagedAccount` | Account configuration updated |
| `EnableGuardrail` | Control enabled on an OU |
| `DisableGuardrail` | Control disabled on an OU |
| `RegisterOrganizationalUnit` | OU enrolled in Control Tower |
| `DeregisterOrganizationalUnit` | OU removed from Control Tower |
| `SetupLandingZone` | Landing zone created |
| `UpdateLandingZone` | Landing zone updated to new version |

**Automation pattern:**
```
Control Tower lifecycle event →
  EventBridge rule →
    Lambda function →
      Custom automation (SSM, CloudFormation, API calls)
```

**Common automations triggered by lifecycle events:**
- Add new account to VPC sharing (RAM)
- Configure Route 53 DNS for new account
- Create baseline IAM roles
- Subscribe to centralized monitoring
- Add account to backup policies
- Register with third-party security tools

---

### Key Quotas & Limits

| Resource | Limit |
|----------|-------|
| OUs that can be registered | 300+ (previously limited) |
| Accounts per org (CT managed) | Thousands (depends on org limit) |
| Controls per OU | No hard limit (but applied individually) |
| Regions governed | Up to all commercial regions |
| Nested OU depth (CT) | 5 levels |
| Account Factory concurrent provisioning | ~5 accounts concurrently |
| Landing zone versions | Must keep updated (mandatory updates) |
| Custom controls | Unlimited (via CfCT or AFT) |
| AFT customization pipelines | Per-account + global |

---

## 2. Design Patterns & Best Practices

### Landing Zone Design Patterns

#### Pattern 1: Centralized Networking (Hub-and-Spoke)

```
Network Account (Infrastructure OU):
  Transit Gateway (shared via RAM to all accounts)
  DNS (Route 53 Private Hosted Zones, Resolver Rules)
  Direct Connect / VPN termination
  Network Firewall / IDS (inspection VPC)

All workload accounts:
  VPCs attach to shared TGW
  No direct internet access (route through inspection VPC)
  Account Factory VPC creation DISABLED
```

**Best practice:** Disable Account Factory VPC and use centralized networking for enterprise environments.

#### Pattern 2: Segmented Compliance Zones

```
Control Tower Landing Zone:
├── Security OU (mandatory) — Log Archive, Audit
├── Infrastructure OU — Network, Shared Services
├── Regulated OU (PCI/HIPAA)
│   ├── Strict controls (encryption, region-restrict, approved services)
│   ├── Production sub-OU
│   └── Non-production sub-OU
├── Standard Workloads OU
│   ├── Production sub-OU
│   └── Non-production sub-OU
└── Sandbox OU — Budget limits, auto-cleanup, permissive
```

#### Pattern 3: Progressive Governance (Policy Staging)

```
1. Test new controls in "Policy Staging OU" (non-production accounts)
2. If no issues → enable on Development OU
3. Monitor for 1-2 weeks → enable on Staging OU
4. If stable → enable on Production OU

This prevents accidentally breaking production workloads with new guardrails.
```

#### Pattern 4: Multi-Region with Region Deny

```
Control Tower Home Region: eu-west-1
Governed Regions: eu-west-1, eu-central-1

Region Deny Control: ENABLED
  → All other regions blocked (except global services)
  → Data stays in EU (GDPR compliance)
  → Reduces attack surface
```

### Anti-Patterns

| Anti-Pattern | Why It's Wrong | Better Approach |
|--------------|---------------|-----------------|
| Running workloads in management account | SCPs don't protect it, blast radius | Dedicated workload accounts |
| Modifying CT-managed resources manually | Causes drift, CT may overwrite changes | Use CfCT or lifecycle events for customization |
| Using Account Factory VPC for complex networking | Limited configuration, doesn't support TGW | Centralized Network account + RAM |
| Not updating Landing Zone versions | Miss security patches, new features | Keep landing zone up-to-date (mandatory updates) |
| Enrolling all OUs at once without testing | May break existing workloads | Enroll one OU at a time, test controls |
| Ignoring drift notifications | Controls may not be functioning | Resolve drift immediately |
| Using Control Tower without customization | Out-of-box doesn't cover everything | CfCT + lifecycle events for full governance |
| Creating accounts outside Account Factory | Accounts miss baselines and controls | Always use Account Factory (or AFT) |
| Disabling strongly recommended controls | Weakens security posture | Only disable with explicit justification |
| Not using a Sandbox OU | Developers use production-governed accounts for experiments | Sandbox OU with budget controls and permissive policies |

### Well-Architected Framework Alignment

**Security:**
- Centralized logging (tamper-proof in Log Archive)
- Preventive controls (SCPs blocking dangerous actions)
- Detective controls (Config rules identifying drift)
- Cross-account audit access (Audit account)
- Region restriction (reduce attack surface)
- Mandatory CloudTrail + Config across all accounts

**Operational Excellence:**
- Automated account provisioning (Account Factory/AFT)
- Drift detection and notification
- Lifecycle events for automation
- Dashboard for compliance visibility
- Version-controlled landing zone updates

**Reliability:**
- Multi-AZ by default (managed service)
- Account isolation prevents cross-workload failures
- Standardized baselines reduce configuration errors
- Automated rollback on failed account provisioning

**Cost:**
- Sandbox accounts with budget caps
- Region restriction reduces resource sprawl
- Standardized VPCs prevent over-provisioning
- Cost allocation tags enforced via controls
- Control Tower itself is free (pay for underlying services)

**Performance:**
- Per-account service quotas (no contention)
- Consistent architecture across accounts (predictable behavior)
- Centralized networking (optimized routing via TGW)

### Integration Patterns

```
┌────────────────────────────────────────────────────────────────────────┐
│              Control Tower Integration Patterns                          │
├────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Account Vending:                                                        │
│  Request → Account Factory/AFT → Account Created →                      │
│  Lifecycle Event → EventBridge → Lambda →                                │
│    RAM share (TGW), DNS setup, Baseline IAM, Monitoring enrollment      │
│                                                                          │
│  Security Integration:                                                   │
│  Control Tower → CloudTrail org trail → Log Archive S3                  │
│  Control Tower → Config rules → Audit Account (aggregator)              │
│  Lifecycle Event → Enable GuardDuty, Security Hub, Inspector            │
│                                                                          │
│  Customization Pipeline:                                                 │
│  CfCT: S3 manifest → CodePipeline → StackSets → All accounts          │
│  AFT: Git commit → CodePipeline → Terraform → Account customization    │
│                                                                          │
│  Compliance Reporting:                                                   │
│  Config (all accounts) → Aggregator (Audit) →                           │
│  Custom dashboard / QuickSight / Third-party SIEM                        │
│                                                                          │
│  Network Automation:                                                     │
│  New Account → Lifecycle Event → Lambda →                                │
│    Create VPC → Attach to TGW → Update route tables → DNS registration  │
│                                                                          │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Security & Compliance

### Control Tower Security Architecture

**Defense in Depth layers provided by Control Tower:**

```
Layer 1: Organization boundary (account isolation)
Layer 2: SCPs (preventive controls — block disallowed actions)
Layer 3: CloudFormation Hooks (proactive — block bad IaC)
Layer 4: Config Rules (detective — identify non-compliance)
Layer 5: CloudTrail (audit — complete API history)
Layer 6: IAM Identity Center (centralized access control)
Layer 7: Log Archive (immutable evidence)
```

### Preventive Security Controls (SCP-based)

**What Control Tower enforces with mandatory SCPs:**
- Protect CloudTrail configuration
- Protect Config configuration
- Protect IAM roles created by Control Tower
- Protect Log Archive S3 buckets
- Ensure CloudTrail is enabled
- Protect SNS topics used by Control Tower

**Strongly recommended SCPs:**
- Deny root user access in member accounts
- Deny creation of access keys for root
- Require MFA for specific actions
- Deny leaving the organization
- Deny modification of specific IAM roles

### Detective Security Controls (Config-based)

**Examples of detective controls:**
- S3 buckets with public access
- EBS volumes not encrypted
- RDS instances not encrypted
- Root user has MFA enabled
- IAM users have MFA enabled
- Security groups allow unrestricted access
- CloudTrail enabled and configured correctly
- VPC flow logs enabled

### Proactive Security Controls (Hook-based)

**Examples of proactive controls:**
- Require RDS encryption in CloudFormation templates
- Require S3 bucket encryption in CloudFormation templates
- Require VPC flow logs in CloudFormation templates
- Block creation of public subnets via CloudFormation
- Require specific instance types in CloudFormation

### Log Archive Security

**S3 bucket protections:**
- Bucket policy denies deletion by ALL principals (including root in Log Archive account)
- Only Control Tower service can manage these buckets
- Versioning enabled
- Lifecycle policies for retention
- Access logging enabled
- SSE-S3 or SSE-KMS encryption

**Access controls:**
- Only specific roles can read logs (Audit account cross-account role)
- Log Archive account admin cannot delete/modify logs
- SCP prevents disabling logging or modifying bucket policies

### Audit Account Security

**Cross-account access:**
- `AWSControlTowerExecution` role exists in ALL member accounts
- Audit account can assume this role for investigation
- Read-only by default, admin if needed for incident response
- All assumption events logged in CloudTrail

**SNS notifications:**
- Config rule violations → SNS → Security team
- Drift detection → SNS → Operations team
- Aggregate findings for compliance reporting

### Compliance Framework Alignment

| Framework | Control Tower Feature |
|-----------|---------------------|
| SOC 2 | Centralized logging, access controls, monitoring, change management |
| PCI DSS | Region restriction, encryption enforcement, access auditing, network segmentation |
| HIPAA | Encryption, audit trails, access controls, approved services only |
| GDPR | Region restriction (data residency), access logging, PII controls |
| FedRAMP | Approved services (allow-list SCP), encryption, continuous monitoring |
| NIST 800-53 | Full mapping: AC (Identity Center), AU (CloudTrail), CM (Config), SC (SCPs) |

### Security Best Practices with Control Tower

1. **Enable all strongly recommended controls** (unless explicit business reason not to)
2. **Delegate security service administration** to the Audit account (GuardDuty, Security Hub)
3. **Add custom detective controls** via CfCT for organization-specific requirements
4. **Use lifecycle events** to auto-enable GuardDuty/Security Hub in new accounts
5. **Implement data perimeter** via custom SCPs (beyond CT's built-in controls)
6. **Monitor drift** and resolve immediately (drift = potential security gap)
7. **Restrict Log Archive access** to absolute minimum (ideally automated access only)
8. **Review Control Tower audit logs** regularly (management account CloudTrail)
9. **Enable region deny** to minimize attack surface
10. **Use proactive controls** for IaC-heavy environments (catch issues before deployment)

---

## 4. Cost Optimization

### Control Tower Pricing

**Control Tower itself: FREE**

**Costs come from underlying services:**

| Service | Cost Driver |
|---------|-------------|
| AWS Config | Per-rule evaluation, per-configuration item recorded (biggest cost) |
| CloudTrail | Organization trail: Per-event (management events free for first trail, data events charged) |
| S3 (Log Archive) | Storage for CloudTrail + Config logs (grows over time) |
| IAM Identity Center | Free |
| Service Catalog | Free (Account Factory) |
| CloudFormation | Free (StackSets deployment) |
| SNS | Per-notification (negligible) |
| VPC (Account Factory) | NAT Gateway per account/AZ if enabled ($0.045/hr + data) |

### Cost Drivers & Optimization

**AWS Config (typically the largest cost):**
- Each Config rule evaluated per-account, per-region
- 10 detective controls × 10 accounts × 3 regions = 300 rule evaluations
- Cost: ~$1/rule/region/month for periodic, $0.003 per evaluation for change-triggered
- **Optimization:**
  - Only govern necessary regions (reduce region count)
  - Use periodic evaluation (not continuous) where possible
  - Disable optional detective controls not needed
  - Consider which regions truly need full Config coverage

**CloudTrail (Log Archive storage):**
- Organization trail captures ALL management events across ALL accounts
- Storage grows linearly with number of accounts and API activity
- **Optimization:**
  - Use S3 lifecycle policies (transition to Glacier after 90 days, delete after 1-7 years based on compliance)
  - Don't enable data events unless specifically required (very high volume)
  - Avoid creating per-account trails IN ADDITION to the organization trail (double logging)

**NAT Gateway (Account Factory VPC):**
- If Account Factory creates VPCs with NAT Gateways: $0.045/hr × AZs × accounts
- 50 accounts × 2 AZs = 100 NAT Gateways = $4.50/hr = $3,240/month
- **Optimization:**
  - Disable Account Factory VPC creation
  - Use centralized NAT (shared services or network account)
  - Use VPC endpoints instead of NAT for AWS service access
  - Use centralized egress VPC via TGW

**Accounts sitting idle:**
- Even empty accounts incur baseline costs if Config/GuardDuty enabled
- ~$10-50/month per account for baseline governance services
- **Optimization:**
  - Close unused accounts (move to Suspended OU → close)
  - Don't provision accounts speculatively

### Cost Optimization Strategies

| Strategy | Savings |
|----------|---------|
| Reduce governed regions to minimum needed | Proportional to region reduction (Config, GuardDuty) |
| Disable Account Factory VPC (use centralized) | Eliminates per-account NAT Gateway costs |
| S3 lifecycle on Log Archive | 60-80% storage reduction (Glacier) |
| Avoid per-account CloudTrail trails | Eliminates duplicate logging costs |
| Close unused accounts | $10-50/month per account saved |
| Use periodic Config rules (not continuous) | Reduces evaluation charges |
| Consolidate NAT egress | Saves $32/month per NAT Gateway eliminated |

### Cost Traps

- **Config rules multiply:** Each control × accounts × regions = many evaluations
- **NAT Gateways per account:** Account Factory default creates per-account NATs (expensive at scale)
- **Landing Zone updates can add controls:** New mandatory controls = new Config rules = new cost
- **Not implementing lifecycle policies on Log Archive:** Logs grow indefinitely
- **Enabling all elective controls "just in case":** More detective controls = more Config evaluations
- **GuardDuty + Security Hub + Inspector across all accounts/regions:** Combined cost can be significant
- **CloudTrail data events:** Enabling data events (S3 object-level, Lambda invoke) is very expensive at scale

---

## 5. High Availability, Disaster Recovery & Resilience

### Control Tower Resilience

**Control Tower as a service:**
- Regional service (deployed in home region)
- Highly available within the home region (managed by AWS)
- If home region has an outage: Control Tower management unavailable
- **But:** Existing controls (SCPs, Config rules) continue to function (they're independent of CT)

**What keeps working during Control Tower outage:**
- SCPs (managed by Organizations, which is global)
- IAM Identity Center (global service)
- Config rules (regional, run independently)
- CloudTrail (regional, runs independently)
- Account access (IAM is global)

**What doesn't work during Control Tower outage:**
- Account Factory provisioning (new accounts)
- Control enablement/disablement
- Drift detection (CT-specific)
- Landing zone updates
- Dashboard

### Multi-Region Resilience

**Control Tower home region:**
- Cannot be changed after setup
- Choose a region close to your operations team
- Consider: Region stability, service availability, compliance requirements

**Governed regions strategy:**
- All governed regions get Config rules and CloudTrail coverage
- If one governed region goes down: Other regions unaffected
- Non-governed regions: No Control Tower coverage (wild west)

### DR Strategy for Landing Zones

**Landing Zone DR is different from workload DR:**

| Concern | Strategy |
|---------|----------|
| SCPs unavailable | Won't happen — Organizations is global, highly available |
| CloudTrail logs lost | Log Archive S3 bucket: Enable CRR to another region |
| Config data lost | Config data stored per-region; use aggregator for central view |
| Identity Center down | Extremely rare (global); fallback: direct IAM role assumption |
| Management account compromised | Break-glass procedure: AWS Support, root user recovery |
| Control Tower service down | Existing controls unaffected; can't provision/change until recovered |

### Resilience Best Practices

1. **Cross-region replication for Log Archive S3** — Critical for compliance (logs must survive region failure)
2. **Avoid single-region dependency** — Don't put all workloads in the CT home region
3. **Document break-glass procedures** — What to do if management account access is lost
4. **Backup critical SCPs and policies** — Store in Git (can redeploy if corrupted)
5. **Test DR regularly** — Verify you can operate without Control Tower console access
6. **Multiple org admins** — Don't rely on a single person for management account access
7. **IAM Identity Center recovery** — Document process for IdP failure scenarios

---

## 6. Exam-Focused Section

### Straightforward Questions

**Q1:** A company is setting up a new AWS environment and needs automated multi-account governance with pre-built security controls, centralized logging, and a compliance dashboard. They want minimal custom development. What should they use?

**A:** AWS Control Tower. It provides an automated landing zone with pre-built controls (preventive SCPs + detective Config rules + proactive CloudFormation Hooks), centralized logging to Log Archive, Account Factory for provisioning, and a built-in compliance dashboard showing control status across all accounts.

---

**Q2:** A company using Control Tower needs to deploy custom Config rules and IAM roles to every account in their organization automatically, including new accounts created in the future. What's the best approach?

**A:** Use **Customizations for Control Tower (CfCT)**. Define a manifest specifying the resources to deploy and target OUs. CfCT uses CodePipeline and CloudFormation StackSets to deploy to all existing and new accounts. Alternatively, use **Control Tower lifecycle events** with EventBridge + Lambda for event-driven automation.

---

**Q3:** A company's Control Tower dashboard shows an OU in "Drift" status. The security team discovers that someone manually modified an SCP that Control Tower manages. How should they resolve this?

**A:** (1) Revert the SCP to its original state (or let Control Tower fix it). (2) In the Control Tower console, select the drifted OU and choose "Re-register OU" or "Repair." Control Tower will reapply the expected baseline. (3) Investigate who made the change (CloudTrail) and prevent recurrence (SCP protecting CT-managed resources is already mandatory, so this indicates an IAM policy issue in the management account).

---

**Q4:** A company needs to provision 50 new AWS accounts with standardized networking, IAM roles, and security baselines. Their platform team uses Terraform. What Control Tower feature should they use?

**A:** **Account Factory for Terraform (AFT)**. It provides a Terraform-based pipeline (CodePipeline + CodeBuild) for provisioning accounts with full customization. Account definitions are stored in Git (GitOps). AFT handles account creation, Control Tower enrollment, and post-creation customization via Terraform modules.

---

**Q5:** A company using Control Tower wants to block any CloudFormation stack from creating an unencrypted RDS instance. Preventive SCPs only block API calls, not IaC declarations. What type of control should they use?

**A:** A **proactive control** (CloudFormation Hook). Proactive controls evaluate CloudFormation templates before resources are created and block non-compliant configurations. This catches unencrypted RDS at the IaC level before the API call happens. Unlike detective controls (which detect after creation), proactive controls prevent creation entirely.

---

**Q6:** A company has Control Tower deployed in eu-west-1 (home region). They need to ensure no resources can be created in any other region. What Control Tower feature enforces this?

**A:** Enable the **"Region deny" control** (elective, preventive). This implements an SCP that denies all actions in non-governed regions (while excluding global services like IAM, STS, Organizations, CloudFront). Only the home region and explicitly governed regions are allowed.

---

**Q7:** A company wants their security team to investigate potential breaches across all accounts without giving them admin access to workload accounts. How does Control Tower support this?

**A:** The **Audit account** has cross-account read-only roles (`AWSControlTowerExecution` or custom roles) in all member accounts. The security team operates from the Audit account and can assume these roles to investigate any account. Additionally, all CloudTrail logs and Config data flow to the Log Archive and Audit accounts respectively, allowing investigation without entering workload accounts directly.

---

### Tricky / Scenario-Based Questions

**Q1:** A company has used Control Tower for 2 years. They need to add a new governed region (ap-southeast-1) because they're expanding to Asia. After adding the region, they notice existing accounts don't have Config rules or CloudTrail coverage in the new region. What happened?

**A:** Adding a governed region to an existing landing zone does NOT automatically retroactively deploy controls to existing accounts in that region. **Fix:** (1) Update the landing zone version (this may trigger re-registration). (2) Re-register existing OUs — this reapplies controls to all accounts including the new region. (3) For accounts created before the region was governed, you may need to update them through Account Factory. **Key concept:** Adding a governed region is a landing zone configuration change that requires propagation to existing accounts. It's not automatic for previously-provisioned accounts.

---

**Q2:** A company uses Control Tower with AFT. A developer requests a new account via the AFT pipeline. The account is created and enrolled in Control Tower, but the Terraform customization step fails (due to a bug in the Terraform code). The account exists but has no custom configuration. What happens?

**A:** The account EXISTS in the organization and IS enrolled in Control Tower (mandatory controls apply). However, the custom resources (VPCs, IAM roles, etc.) are missing. **Resolution:** (1) Fix the Terraform code in Git. (2) Re-trigger the AFT pipeline for that account (customization re-run). (3) The account is already governed by CT controls — it's just missing custom infrastructure. **Key concept:** AFT separates account creation/enrollment from customization. A customization failure doesn't roll back the account — the account is created but in a partially configured state.

---

**Q3:** A company's management account administrator accidentally deletes the `AWSControlTowerExecution` role in a member account. Control Tower reports drift. What impact does this have and how do they fix it?

**A:** **Impact:** Control Tower cannot manage that account (can't deploy Config rules, can't apply StackSets, can't update controls). The account's existing controls still function (SCPs still apply, existing Config rules still run), but CT can't make changes. **Fix:** (1) Recreate the `AWSControlTowerExecution` role manually in the member account (matching the original trust policy allowing the management account). (2) Alternatively, use the management account's `OrganizationAccountAccessRole` to recreate it. (3) Then repair the drift in Control Tower. **Key concept:** CT-managed roles are critical for CT operations. Drift here means CT is blind to the account for management purposes.

---

**Q4:** A company using Control Tower decides to implement a custom SCP that's more restrictive than CT's built-in controls. They attach it directly to an OU in Organizations (not through Control Tower). A month later, Control Tower shows drift. Why?

**A:** Control Tower **does NOT cause drift just because you add custom SCPs.** Drift occurs if you MODIFY or DETACH SCPs that Control Tower manages. Adding additional custom SCPs alongside CT-managed ones is supported and expected. **If drift IS showing:** The custom SCP may have inadvertently modified a CT-managed SCP (e.g., exceeded the 5-SCP-per-entity limit, forcing detachment). **Key concept:** You CAN add custom SCPs alongside CT controls. Drift only occurs if CT-managed resources are modified. Check if attaching the new SCP caused a CT-managed one to be detached (due to the 5-SCP limit).

---

**Q5:** A company uses Control Tower with 200 accounts. They want to migrate from their external IdP (Okta) to Microsoft Entra ID (formerly Azure AD) for IAM Identity Center. During the migration, will accounts lose access?

**A:** **Potential disruption.** Changing the identity source in IAM Identity Center deletes all existing user/group assignments (permission set assignments remain, but the identities they're mapped to are removed). **Migration approach:** (1) Plan carefully — document all user/group → account → permission set mappings. (2) Switch identity source (Identity Center supports only ONE source at a time). (3) Provision users/groups from new IdP via SCIM. (4) Reassign users/groups to accounts and permission sets. (5) During transition: Users cannot access accounts via SSO. **Mitigation:** Use direct IAM role assumption as break-glass during migration. **Key concept:** IdP migration in Identity Center is disruptive. All identity-based mappings are lost when switching sources.

---

**Q6:** A company enabled a detective control "Detect whether S3 buckets have public access" across their Production OU. After enabling, the dashboard shows 50 non-compliant buckets. Does Control Tower automatically remediate them?

**A:** **No.** Detective controls only DETECT and REPORT non-compliance — they do NOT auto-remediate. The 50 buckets remain public until someone fixes them. **To remediate:** (1) Manually fix each bucket. (2) Set up auto-remediation via Config rules (SSM Automation document that disables public access). (3) Add a PREVENTIVE control (SCP) to prevent NEW public buckets from being created going forward. **Key concept:** Detective = visibility only. Preventive = blocks new violations. Neither fixes existing non-compliance. You need remediation (manual or automated via Config + SSM) for existing resources.

---

**Q7:** A company uses Control Tower. An application team needs an account that's exempt from the region-deny control because their application requires us-west-2 (not a governed region). Other accounts must remain restricted. How should they handle this?

**A:** (1) Create a separate OU (e.g., "Global Workloads OU") that is registered with Control Tower but does NOT have the region-deny control enabled. (2) Place the application's account in this OU. (3) Apply all other controls normally to this OU. (4) The region-deny control remains enabled on the other OUs. **Why this works:** Controls are applied at the OU level. Different OUs can have different control configurations. **Wrong answer:** "Disable region-deny for the entire organization" — this removes protection from all accounts. **Key concept:** OU structure is how you differentiate control application. Exceptions = separate OU.

---

### Common Exam Traps & Pitfalls

1. **Control Tower ≠ Organizations.** Control Tower USES Organizations but adds automation, guardrails, Account Factory, and dashboards. They're not synonyms.

2. **Control Tower is a REGIONAL service.** Home region cannot be changed. If that region goes down, CT management is unavailable (but existing controls still work).

3. **Drift ≠ auto-remediation.** Control Tower detects drift and notifies. It does NOT fix drift automatically (for most cases). You must intervene.

4. **Detective controls don't remediate.** They detect non-compliance but don't fix it. For remediation, you need Config auto-remediation or manual action.

5. **Proactive controls only work with CloudFormation.** If resources are created via CLI/Console (not IaC), proactive controls won't catch them. Use preventive (SCP) for runtime protection.

6. **Account Factory VPC is optional.** You can (and often should) disable it for enterprise patterns with centralized networking.

7. **Adding governed regions doesn't auto-deploy to existing accounts.** You must re-register OUs or update accounts for retroactive coverage.

8. **Control Tower manages specific SCPs/roles/Config rules.** Don't modify them manually — it causes drift. Add CUSTOM ones alongside them.

9. **Mandatory controls cannot be disabled.** They're non-negotiable, even if they seem restrictive.

10. **5 SCP limit per entity still applies.** Control Tower uses some of those 5 slots. If you add custom SCPs and exceed 5, something gets detached → drift.

11. **AFT vs Account Factory vs CfCT — different tools for different purposes:**
    - Account Factory = create accounts (console/Service Catalog)
    - AFT = create accounts + customize with Terraform (GitOps)
    - CfCT = customize existing accounts with CloudFormation (post-creation)

12. **Log Archive account ≠ Audit account.** Log Archive = storage (S3 logs). Audit = investigation (cross-account access, alerting). Common confusion.

---

## 7. Cheat Sheet

### Must-Know Facts

| Fact | Detail |
|------|--------|
| Control Tower cost | Free (underlying services cost: Config, CloudTrail, S3) |
| Home region | Cannot be changed after setup |
| Foundational accounts | Management + Log Archive + Audit (Security OU) |
| Control types | Preventive (SCP), Detective (Config), Proactive (CFN Hook) |
| Control behaviors | Mandatory (always on), Strongly Recommended, Elective |
| Drift | Detected but NOT auto-remediated |
| Controls applied at | OU level (not individual accounts) |
| Account Factory uses | AWS Service Catalog |
| AFT uses | Terraform + CodePipeline + CodeBuild |
| CfCT uses | CloudFormation StackSets + CodePipeline |
| Lifecycle events | CloudTrail events → EventBridge → Automation |
| Log Archive protection | S3 bucket policies prevent deletion by all principals |
| Region deny | Elective control, blocks non-governed regions |
| Max OU depth | 5 levels |
| Proactive controls apply to | CloudFormation only (not CLI/Console) |
| SCPs managed by CT | Don't modify manually (causes drift) |
| Adding custom SCPs | Allowed alongside CT-managed SCPs (no drift) |

### Key Differentiators

| Feature | Control Tower | Organizations (alone) | CfCT | AFT |
|---------|--------------|----------------------|------|-----|
| **Purpose** | Automated landing zone | Account structure/policies | Extend CT with custom resources | Terraform account provisioning |
| **Automation** | Full (setup, baselines) | Manual | Post-creation customization | Full account lifecycle |
| **Controls** | Pre-built + custom | Manual SCPs only | Custom CloudFormation | Custom Terraform |
| **Account creation** | Account Factory | CreateAccount API | Not for account creation | Full GitOps pipeline |
| **Dashboard** | Built-in compliance view | None | None | None |
| **Drift detection** | Automatic | Manual | N/A | N/A |
| **Best for** | New environments, compliance | Existing complex orgs | Extending CT governance | Terraform-native teams |

### Service Selection: "If the question says X, think Y"

| Question Keywords | Answer |
|-------------------|--------|
| "Automated landing zone", "best-practice setup" | Control Tower |
| "Compliance dashboard", "drift detection" | Control Tower |
| "Pre-built guardrails", "out-of-box governance" | Control Tower Controls |
| "Block before creation via IaC" | Proactive Control (CloudFormation Hook) |
| "Detect non-compliance in existing resources" | Detective Control (Config Rule) |
| "Block API calls organization-wide" | Preventive Control (SCP) |
| "Automated account creation with Terraform" | AFT (Account Factory for Terraform) |
| "Deploy custom baselines to all accounts" | CfCT (Customizations for Control Tower) |
| "New account triggers automation" | Lifecycle Events + EventBridge |
| "Centralized immutable logs" | Log Archive Account |
| "Cross-account security investigation" | Audit Account |
| "Restrict all activity to specific regions" | Region Deny Control (Elective) |
| "Standardized VPC in every account" | Account Factory VPC config (or CfCT for custom) |
| "Account drifted from baseline" | Control Tower Drift Detection → Repair |
| "Extend Control Tower with custom SCPs" | CfCT (manifest with SCP templates) |

### Decision Flowchart

```
Question mentions...                                         → Think...
─────────────────────────────────────────────────────────────────────────
"new environment", "landing zone", "automated setup"        → Control Tower
"existing org, add governance on top"                       → Control Tower (enroll existing OUs)
"pre-built security controls"                               → Control Tower Controls
"block before CloudFormation creates resource"              → Proactive Control
"detect existing non-compliance"                            → Detective Control
"prevent actions org-wide"                                  → Preventive Control (SCP)
"Terraform account pipeline"                                → AFT
"custom CloudFormation baselines to all accounts"           → CfCT
"account provisioned → auto-configure"                     → Lifecycle Events
"immutable audit logs"                                      → Log Archive Account
"security investigation"                                    → Audit Account
"someone changed CT-managed SCP"                            → Drift → Re-register OU
"dashboard shows non-compliance"                            → Detective control detected it → fix manually
"restrict to approved regions"                              → Region Deny (elective control)
"account needs exception to a control"                      → Move to different OU without that control
```

---

## 8. Deep Dive: 10 More Tricky Scenario-Based Questions

**Q1:** A company uses Control Tower and has enabled 15 detective controls on their Production OU (50 accounts, 4 governed regions). The AWS Config bill is unexpectedly high. How should they optimize costs without losing critical compliance visibility?

**A:** (1) **Reduce governed regions** to only regions with production resources (if some are unused). Going from 4→2 regions halves Config cost. (2) **Consolidate detective controls** — some may overlap. Review which are truly needed for compliance vs. nice-to-have. (3) **Switch periodic evaluation** for non-critical controls (reduces evaluation count). (4) **Disable elective controls** that aren't compliance-mandated. (5) **Use AWS Config aggregator** for visibility without per-region full evaluation. **Cost math:** 15 controls × 50 accounts × 4 regions = 3,000 rule evaluations. Reducing to 10 controls × 2 regions = 1,000 evaluations (67% reduction). **Key concept:** Config costs scale with controls × accounts × regions. Reduce any dimension to reduce cost.

---

**Q2:** A company has a Control Tower landing zone with CfCT deploying custom security baselines. They need to update the landing zone to the latest version. After the update, some of their custom CfCT-deployed StackSets show drift. What happened?

**A:** Landing zone updates may modify OU structures, SCPs, or Config configurations that CfCT deployments depend on. If CfCT deployed resources that reference CT-managed resources (like specific SCP names, Config rule names, or IAM role ARNs), the update may have renamed or modified these dependencies. **Fix:** (1) Review Control Tower update release notes for breaking changes. (2) Update CfCT manifests to reference new resource names/ARNs if changed. (3) Re-run CfCT pipeline to reapply customizations. **Prevention:** Design CfCT customizations to be independent of CT-managed resource names where possible. Use parameters/lookups rather than hardcoded references. **Key concept:** Landing zone updates can affect customizations. Always review release notes and test in a staging OU before updating production.

---

**Q3:** A company's Audit account administrator has been compromised. The attacker assumes the `AWSControlTowerExecution` role into production accounts and begins exfiltrating data. How could this have been prevented, and how should they respond?

**A:** **Prevention:** (1) Enable MFA on all Audit account access. (2) Add SCP condition on `sts:AssumeRole` requiring MFA for the execution role. (3) Limit the execution role's permissions (by default it's broad). (4) Monitor cross-account role assumption with CloudWatch alerts. (5) Restrict who in the Audit account can assume cross-account roles. **Response:** (1) Immediately revoke Audit account compromised credentials (rotate keys, disable users). (2) Restrict the `AWSControlTowerExecution` role trust policy. (3) Review CloudTrail for all actions taken via this role. (4) Assess data exposure in affected accounts. (5) Implement short-session durations and continuous monitoring. **Key concept:** The Audit account has powerful cross-account access. It needs the STRONGEST identity controls (MFA, monitoring, least privilege on who can operate within it).

---

**Q4:** A company wants to use Control Tower but they already have 100 accounts in an existing Organization with custom SCPs and Config rules. They're worried that enabling Control Tower will break existing workloads. What's the recommended migration approach?

**A:** (1) **Enable Control Tower in the management account** — this initially only sets up the Security OU (Log Archive + Audit). It does NOT immediately affect existing accounts. (2) **Register OUs incrementally** — start with a non-production OU. Registering an OU deploys CT controls to it. (3) **Test:** Verify existing workloads in the registered OU still function with CT's mandatory controls. (4) **Resolve conflicts:** If existing custom SCPs conflict with CT-managed SCPs (5-SCP limit), consolidate them. (5) **Gradually register remaining OUs** (Production last). (6) **Existing Config rules are preserved** — CT adds its own alongside yours. **Key concept:** Control Tower enrollment is incremental. You don't have to register everything at once. Test with non-critical OUs first.

---

**Q5:** A company uses Control Tower with Account Factory. A team creates a new account through the Service Catalog product. During creation, they specify a custom VPC CIDR (10.0.0.0/16) that overlaps with an existing account's VPC. The account is created successfully. Why didn't Control Tower prevent the overlap?

**A:** **Control Tower does NOT validate VPC CIDR overlap across accounts.** Account Factory creates VPCs with the specified CIDR regardless of what other accounts use. There's no cross-account CIDR management built into CT. **Fix:** (1) Use an external IPAM solution (AWS VPC IPAM) to manage CIDR allocations. (2) Implement a custom Lambda triggered by lifecycle events that checks CIDR allocation against a central registry before allowing VPC creation. (3) Disable Account Factory VPC creation entirely and manage networking centrally. **Best practice:** Centralized IPAM + centralized networking account. **Key concept:** Control Tower provides governance (security, compliance) — not network management. CIDR planning is your responsibility.

---

**Q6:** A company enables a new elective control (region deny) on their Workloads OU. Several hours later, a team reports that their CI/CD pipeline in us-west-2 (a non-governed region) has stopped deploying. Lambda functions in us-west-2 can't be updated. What happened and how should they fix it without disabling the control for everyone?

**A:** The region deny control (SCP) blocks all actions in non-governed regions. Since us-west-2 isn't governed, the CI/CD pipeline's IAM roles in that OU can no longer perform actions there. **Fix:** (1) **Option A:** Add us-west-2 as a governed region (requires landing zone update — Config and CloudTrail will be deployed there). (2) **Option B:** Move the CI/CD account to a different OU where region deny is NOT enabled (e.g., a "Global Infrastructure" OU). (3) **Option C:** Don't use Control Tower's built-in region deny — implement a custom SCP with conditions exempting the CI/CD role. **Key concept:** The built-in region deny control is binary (all-or-nothing for the OU). For granular exceptions, use custom SCPs instead.

---

**Q7:** A company uses Control Tower with lifecycle events to auto-configure new accounts. They create an EventBridge rule that triggers a Lambda function when `CreateManagedAccount` succeeds. The Lambda function shares a Transit Gateway with the new account via RAM. Occasionally, the sharing fails because the account isn't fully ready. What's the issue?

**A:** **Timing issue.** The `CreateManagedAccount` lifecycle event fires when the account is created in Organizations, but the account may not be fully ready for resource operations (IAM roles may still be propagating, resources may still be deploying). **Fix:** (1) Add a delay/retry mechanism in the Lambda (wait 60-120 seconds before attempting RAM share). (2) Use Step Functions with a Wait state before the sharing action. (3) Check for specific account readiness indicators (verify the target IAM role exists via STS before proceeding). (4) Implement exponential backoff on failures. **Key concept:** Lifecycle events fire when the CT operation completes, but downstream dependencies (IAM propagation, StackSets deployment) may still be in progress. Build resilience into your automation.

---

**Q8:** A company implemented Control Tower 18 months ago. AWS releases a new mandatory landing zone version with critical security fixes. The company's platform team is in a change freeze for Q4. Can they delay the update?

**A:** **Technically yes, but it's risky.** Mandatory landing zone updates have deadlines — AWS communicates required update windows. Failing to update may result in: (1) Loss of certain CT functionality. (2) Security vulnerabilities in the landing zone baseline. (3) Eventually, CT may stop functioning correctly with the outdated version. **Recommendation:** (1) Review the update's release notes for breaking changes. (2) Test the update in a non-production OU. (3) Prioritize the security fix over the change freeze (coordinate with change management). (4) Most updates are non-disruptive (backward compatible). **Key concept:** Landing zone version updates are mandatory over time. Plan for them. They rarely break existing workloads but may modify CT-managed infrastructure.

---

**Q9:** A company uses Control Tower and wants to implement "account tagging" — tagging each account with cost center, team, and environment metadata at creation time. Account Factory doesn't have native account-level tagging during provisioning. How should they implement this?

**A:** (1) **Use AFT** — AFT supports account-level tags in account request definitions (Terraform). (2) **Lifecycle events approach:** EventBridge rule on `CreateManagedAccount` → Lambda that calls `organizations:TagResource` to tag the account. (3) **Service Catalog tag options:** Add tag constraints to the Account Factory Service Catalog product (tags inherited during provisioning). (4) **Post-provisioning:** CfCT or Lambda applies tags after account creation. **Best approach for exam:** AFT if using Terraform (native support). Lifecycle events + Lambda for Service Catalog-based Account Factory. **Key concept:** Account-level tagging is separate from resource tagging. Use Organizations API (`TagResource`) for account tags.

---

**Q10:** A company has Control Tower managing 300 accounts. They need to report compliance status to their board quarterly — showing which accounts comply with each control and which don't. Control Tower's dashboard shows real-time status but can't generate historical reports. How should they build this?

**A:** (1) **Config Aggregator** (in Audit account) collects compliance data from all accounts. (2) **AWS Config Advanced Query** to pull compliance status per rule/account. (3) **Export to S3** (Config snapshots or Config stream → Firehose → S3). (4) **QuickSight** for visualization and quarterly reports. (5) **Alternative:** Use **Security Hub** (aggregates findings including Config rule evaluations) → export to S3 → QuickSight. (6) For time-series compliance tracking: **Config Compliance Timeline** (shows compliance state over time per resource). **Architecture:**
```
Config (all accounts) → Aggregator (Audit) → S3 export → Athena/QuickSight → Board report
```
**Key concept:** Control Tower dashboard is real-time only. For historical/trend reporting, export Config/Security Hub data to S3 and use analytics services.

---

## 9. Service Comparison Tables

### Landing Zone Approaches Compared

| Dimension | Control Tower | Custom (CloudFormation) | Custom (Terraform) | AWS Landing Zone (deprecated) |
|-----------|--------------|------------------------|--------------------|-----------------------------|
| **Setup effort** | Low (automated) | High (build yourself) | High (build yourself) | Medium (template-based) |
| **Guardrails** | Pre-built (300+) | Write your own | Write your own | Pre-built (limited) |
| **Account provisioning** | Account Factory / AFT | StackSets + custom | Terraform modules | Account Vending Machine |
| **Drift detection** | Built-in | Build yourself | Build yourself | Limited |
| **Dashboard** | Built-in | Build yourself | Build yourself | None |
| **Flexibility** | Opinionated (limited customization) | Unlimited | Unlimited | Limited |
| **Maintenance** | AWS maintains (mandatory updates) | You maintain everything | You maintain everything | Deprecated (no updates) |
| **Best for** | Most organizations | Highly custom requirements | Terraform-native teams | Don't use (migrate to CT) |
| **Exam answer** | Default recommendation | "already have complex custom setup" | "Terraform requirement" | Wrong answer (deprecated) |

### Control Types Compared

| Dimension | Preventive | Detective | Proactive |
|-----------|-----------|-----------|-----------|
| **Mechanism** | SCP | AWS Config Rule | CloudFormation Hook |
| **When evaluated** | At API request time | After resource exists | Before CFN creates resource |
| **Scope** | All API calls (CLI, Console, SDK, IaC) | All existing resources | CloudFormation only |
| **Effect** | Blocks action | Reports non-compliance | Blocks CFN deployment |
| **Remediation** | N/A (prevented) | Manual or auto-remediation | N/A (prevented) |
| **Cost** | Free (SCP) | Per-rule evaluation | Free (Hook) |
| **Latency** | None (immediate) | Minutes (evaluation period) | Adds to CFN deploy time |
| **Exam signal** | "block", "prevent", "deny" | "detect", "identify", "report" | "IaC", "CloudFormation", "before creation" |

### Account Provisioning Methods Compared

| Dimension | Account Factory (Console) | Account Factory (API) | AFT | AFC |
|-----------|--------------------------|----------------------|-----|-----|
| **Interface** | Service Catalog console | Service Catalog API | Git + Terraform | Service Catalog blueprints |
| **IaC** | No | Possible (API call) | Full Terraform | CloudFormation |
| **Customization** | VPC, OU, user | Same | Full lifecycle (Terraform) | Blueprints (CFN templates) |
| **Pipeline** | None | None | CodePipeline + CodeBuild | Service Catalog provisioning |
| **GitOps** | No | No | Yes | No |
| **Scale** | Low (manual) | Medium | High (fully automated) | Medium |
| **Team fit** | Console-comfortable | API-driven | Terraform experts | CloudFormation teams |
| **Exam signal** | "simple, few accounts" | "programmatic provisioning" | "Terraform", "GitOps", "at scale" | "CloudFormation blueprints" |

### Customization Approaches Compared

| Dimension | CfCT | AFT Customizations | StackSets (direct) | Lifecycle Events + Lambda |
|-----------|------|-------------------|--------------------|-----------------------------|
| **Scope** | Deploy to existing OUs/accounts | Per-account (during provisioning) | Any accounts/OUs | Event-triggered per-account |
| **IaC Tool** | CloudFormation | Terraform | CloudFormation | Custom (any) |
| **Trigger** | S3 upload / Schedule | Account creation | Manual / Scheduled | CT lifecycle event |
| **New account** | Deploys to OU members | Part of provisioning | Auto-deploy to new (service-managed) | Event-driven automation |
| **Best for** | Org-wide custom baselines | Per-account Terraform config | Direct StackSet deployment | Light automation (RAM, DNS, etc.) |
| **Integrates with CT?** | Yes (designed for CT) | Yes (CT enrollment) | Yes (can target CT OUs) | Yes (CT lifecycle events) |

---

## 10. When to Pick Service A over Service B — Exam Keywords

### Pick Control Tower over bare Organizations when:
- "new multi-account environment"
- "automated landing zone"
- "pre-built guardrails / controls"
- "compliance dashboard"
- "minimal operational effort"
- "best practices"
- "Account Factory"
- "drift detection"
- **Exam signal:** New setup + automation + compliance + least effort

### Pick Organizations (without CT) over Control Tower when:
- "already have complex existing org with custom governance"
- "Control Tower too opinionated"
- "need full flexibility in SCP design"
- "only need consolidated billing"
- "specific region not supported by CT"
- **Exam signal:** Existing complex setup + maximum flexibility + simple needs

### Pick AFT over Account Factory (Service Catalog) when:
- "Terraform"
- "GitOps", "version-controlled account definitions"
- "automated pipeline for account provisioning"
- "large scale (50+ accounts)"
- "custom post-provisioning automation"
- **Exam signal:** Terraform + automation + scale

### Pick Account Factory over AFT when:
- "simple account creation"
- "console-based teams"
- "few accounts (< 20)"
- "don't use Terraform"
- "least complexity"
- **Exam signal:** Simple + few accounts + console-comfortable

### Pick CfCT over StackSets (direct) when:
- "extend Control Tower governance"
- "deploy custom baselines alongside CT controls"
- "maintain relationship with CT lifecycle"
- "manifest-driven deployment"
- **Exam signal:** Custom governance layered on top of CT

### Pick StackSets (direct) over CfCT when:
- "not using Control Tower"
- "simple resource deployment to multiple accounts"
- "don't need CfCT pipeline complexity"
- "ad-hoc multi-account deployment"
- **Exam signal:** Simple StackSet deployment without CT dependency

### Pick Proactive Controls over Preventive when:
- "CloudFormation", "IaC"
- "validate before creation"
- "check template configuration"
- "property-level validation" (e.g., encryption flag on RDS)
- **Exam signal:** IaC validation, pre-deployment checks

### Pick Preventive Controls over Proactive when:
- "block API calls regardless of how they're made"
- "CLI, Console, AND CloudFormation"
- "runtime restriction"
- "organization-wide hard guardrail"
- **Exam signal:** Universal block (all paths: CLI/Console/IaC)

### Pick Detective Controls over both when:
- "identify existing non-compliance"
- "visibility into current state"
- "continuous monitoring"
- "can't block (need to allow but report)"
- "compliance reporting"
- **Exam signal:** Monitor + report + existing resources

### Pick Lifecycle Events over CfCT when:
- "lightweight automation on account creation"
- "single action (RAM share, DNS registration)"
- "event-driven (not baseline)"
- "don't need full StackSet deployment"
- **Exam signal:** Simple trigger-based automation, not resource deployment

---

## 11. "Gotcha" Differences the Exam Tests

### Control Tower Gotchas

1. **Control Tower is REGIONAL.** Home region locked at setup. CT management is unavailable if home region goes down. But existing controls (SCPs, Config rules) still function independently.

2. **Controls apply at OU level, not account level.** You can't enable a control for one specific account — it's all-or-nothing per OU. Exception = move the account to a different OU.

3. **Mandatory controls cannot be disabled.** Period. Even if they seem inconvenient. They protect CT's own infrastructure.

4. **Drift detection ≠ auto-remediation.** CT tells you something is wrong. YOU must fix it. Common misconception that CT auto-fixes.

5. **Proactive controls ONLY apply to CloudFormation.** If someone creates resources via CLI or Console, proactive controls don't fire. You STILL need preventive (SCP) or detective (Config) for non-IaC paths.

6. **Adding a governed region doesn't retroactively apply to existing accounts.** You must re-register OUs or update accounts.

7. **Account Factory VPC is per-account by default.** Each account gets its own VPC — not shared. This creates NAT Gateway sprawl. Enterprise pattern: Disable and use centralized networking.

8. **5 SCP limit still applies with CT.** CT uses SCP slots. If you add custom SCPs, you may hit the limit. If the 5th CT-managed SCP can't be attached → drift.

9. **CT-managed resources are off-limits.** Modifying them manually causes drift. Add your own ALONGSIDE them, don't modify them.

10. **Log Archive account admin cannot delete logs.** The bucket policy prevents deletion even by the account root. Only CT (via management account automation) manages these buckets.

### AFT Gotchas

11. **AFT requires a dedicated management account (AFT account).** It's not installed in the CT management account — it gets its own account in an Infrastructure OU.

12. **AFT customization failure doesn't roll back account creation.** The account exists but may be partially configured. You must re-run the pipeline.

13. **AFT pipeline stages are sequential.** Global customizations run before account customizations. Dependency order matters.

14. **AFT account requests are in Terraform.** Non-Terraform teams can't easily use it (learning curve).

### Landing Zone Gotchas

15. **You cannot change the home region.** Choose wisely at setup. If you need to change it, you'd need to decommission and rebuild (major undertaking).

16. **Landing zone updates are mandatory (eventually).** AWS may deprecate old versions. Plan for periodic updates.

17. **Existing accounts need enrollment.** Just having accounts in the org doesn't mean CT governs them. You must explicitly register the OU or enroll the account.

18. **Nested OU support:** CT supports nested OUs up to 5 levels, but controls may not inherit intuitively. Test inheritance behavior.

### Identity Center Gotchas

19. **Changing IdP in Identity Center is disruptive.** All user/group assignments are lost when switching identity sources. Plan migration carefully.

20. **Permission Sets are deployed as IAM roles.** Updating a Permission Set requires re-provisioning to target accounts (not instant).

21. **Only ONE Identity Center instance per organization.** Can't have regional instances or per-OU instances.

### Compliance Gotchas

22. **Detective controls show NON-COMPLIANT but don't prevent creation.** A new public S3 bucket CAN be created — the detective control just flags it afterward.

23. **Config rule evaluation has a delay.** Resources may exist for minutes before being flagged. Not real-time prevention.

24. **Proactive controls add deployment time.** CloudFormation Hooks evaluate during stack operations — adds latency to deployments.

25. **Not all AWS Config rules are available as CT controls.** Some Config rules must be deployed separately via CfCT or direct Config setup.

---

## 12. Decision Tree: Choosing the Right Control Tower Feature

```
START: What do you need?
│
├─── Need to SET UP a landing zone?
│    ├─── New environment, want best practices fast? → Control Tower
│    ├─── Already have complex org? → Control Tower (enroll incrementally)
│    ├─── Need maximum flexibility (non-standard)? → Custom (Organizations + IaC)
│    └─── Using deprecated AWS Landing Zone? → Migrate to Control Tower
│
├─── Need to PROVISION accounts?
│    ├─── Simple, console-based? → Account Factory (Service Catalog)
│    ├─── Terraform, GitOps, at scale? → AFT
│    ├─── CloudFormation blueprints? → AFC (Account Factory Customization)
│    └─── Programmatic (API-driven)? → Account Factory API or AFT
│
├─── Need to ENFORCE governance?
│    ├─── Block API calls (any method)? → Preventive Control (SCP)
│    ├─── Block non-compliant IaC? → Proactive Control (CFN Hook)
│    ├─── Detect existing non-compliance? → Detective Control (Config Rule)
│    ├─── Custom controls beyond CT built-in? → CfCT (deploy custom SCPs/Config rules)
│    └─── Need auto-remediation? → Detective Control + Config Remediation (SSM)
│
├─── Need to CUSTOMIZE the landing zone?
│    ├─── Deploy resources to all accounts? → CfCT (CloudFormation StackSets)
│    ├─── Per-account Terraform customization? → AFT customization stage
│    ├─── Event-driven light automation? → Lifecycle Events + EventBridge + Lambda
│    └─── Custom SCPs beyond CT's built-in? → CfCT manifest (SCP section)
│
├─── Need to handle EXCEPTIONS?
│    ├─── Account needs different controls? → Move to separate OU
│    ├─── Role needs exemption from SCP? → SCP condition (ArnNotLike)
│    ├─── Region exception for one account? → Separate OU without region deny
│    └─── Temporary exception during migration? → Temporary OU with relaxed controls
│
├─── Need COMPLIANCE REPORTING?
│    ├─── Real-time compliance view? → Control Tower Dashboard
│    ├─── Historical compliance trend? → Config → S3 → Athena/QuickSight
│    ├─── Aggregated findings? → Security Hub (aggregator in Audit account)
│    └─── Audit trail? → CloudTrail → Log Archive S3
│
├─── Need to DETECT AND FIX problems?
│    ├─── CT configuration changed unexpectedly? → Drift Detection → Re-register/Repair
│    ├─── Resource non-compliant? → Detective Control → Manual fix or Config auto-remediation
│    ├─── Who made the change? → CloudTrail (Log Archive)
│    └─── Automated alert on violation? → Config Rule → SNS (Audit account)
│
└─── Need to RESPOND TO INCIDENTS?
     ├─── Investigate specific account? → Audit account cross-account roles
     ├─── Review API history? → CloudTrail (Log Archive)
     ├─── Contain blast radius? → Move account to Quarantine OU (deny-all SCP)
     └─── Lock out compromised user? → IAM Identity Center (revoke session)
```
