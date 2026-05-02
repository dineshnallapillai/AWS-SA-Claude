# Service Control Policies (SCPs) & Permission Boundaries

## 1. Core Concepts & Theory

---

### The AWS Authorization Model

AWS evaluates permissions through multiple policy layers. Understanding how they interact is critical for the SAP-C02 exam.

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    AWS Authorization Decision Flow                         │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  Request arrives →                                                         │
│                                                                            │
│  1. Is there an explicit DENY in any policy?     → YES → DENY             │
│  2. Is there an SCP that allows? (org member)    → NO  → DENY             │
│  3. Is there a Resource Policy that allows?      → YES → might ALLOW*      │
│  4. Is there a Permission Boundary?              → NO allow → DENY         │
│  5. Is there a Session Policy?                   → NO allow → DENY         │
│  6. Is there an Identity Policy that allows?     → NO  → DENY             │
│  7. All checks pass                             → ALLOW                    │
│                                                                            │
│  *Resource policy can grant access without identity policy (same account)  │
│   For cross-account: both sides must allow (except resource policy alone)  │
│                                                                            │
│  Effective Permissions = Intersection of ALL applicable policy types        │
│                                                                            │
└──────────────────────────────────────────────────────────────────────────┘
```

**Policy types that SET BOUNDARIES (restrict, don't grant):**
- Service Control Policies (SCPs)
- Permission Boundaries
- Session Policies

**Policy types that GRANT ACCESS:**
- Identity-based policies (IAM user/role/group policies)
- Resource-based policies (bucket policies, KMS key policies, etc.)

---

### Service Control Policies (SCPs) — Deep Dive

**What they are:** Organization-level policies that define the MAXIMUM available permissions for IAM entities in member accounts. They are a guardrail — they restrict what's possible, never grant access.

#### How SCPs Work

**Evaluation logic:**
- SCPs are evaluated BEFORE IAM policies
- If an SCP doesn't allow an action → DENIED regardless of IAM permissions
- If an SCP explicitly denies an action → DENIED regardless of IAM permissions
- SCPs set the ceiling; IAM policies grant within that ceiling
- **Effective permissions = SCP ∩ IAM Policy** (intersection)

**What SCPs affect:**
- All IAM users in member accounts
- All IAM roles in member accounts
- The ROOT USER in member accounts
- Federated users (via SAML, Identity Center)

**What SCPs DO NOT affect:**
- The management account (NEVER — even if SCP attached to Root)
- Service-linked roles (SLRs)
- Actions performed by the Organizations service itself
- Resource-based policies granting access to principals outside the org (the policy still applies, but the action evaluation in the member account is bound by SCP)

#### SCP Inheritance Model

```
Root (FullAWSAccess)
│
├── OU-A (SCP: Deny ec2:TerminateInstances)
│   ├── OU-A1 (SCP: Deny s3:DeleteBucket)
│   │   └── Account-1
│   │       Effective: Deny TerminateInstances + Deny DeleteBucket
│   └── Account-2
│       Effective: Deny TerminateInstances only
│
├── OU-B (SCP: Allow only s3:*, ec2:*, iam:*)
│   └── Account-3
│       Effective: ONLY s3, ec2, iam allowed (all others denied)
│
└── Account-4 (directly under Root)
    Effective: FullAWSAccess (no additional restrictions)
```

**Inheritance rules:**
1. SCPs attached to a parent apply to all children (OUs and accounts below)
2. An account's effective SCP = intersection of ALL SCPs from Root down to the account
3. For **allow-list strategy**: Action must be allowed at EVERY level (Root → OU → Account)
4. For **deny-list strategy**: A deny at ANY level = denied (denies cascade down)
5. Multiple SCPs on the same entity: Evaluated together (union of allows, any deny wins)

#### SCP Strategies Deep Dive

**Strategy 1: Deny List (Recommended)**

```json
// FullAWSAccess (default, attached at Root)
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "*",
    "Resource": "*"
  }]
}

// Additional deny SCP (attached to specific OU)
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyDangerousActions",
    "Effect": "Deny",
    "Action": [
      "organizations:LeaveOrganization",
      "cloudtrail:StopLogging",
      "guardduty:DeleteDetector"
    ],
    "Resource": "*"
  }]
}
```

**Advantages:**
- Simple to understand and manage
- Default-allow means new services work immediately
- Only define what you specifically want to block
- Less risk of accidentally locking out accounts

**Strategy 2: Allow List (Strict Compliance)**

```json
// Step 1: REMOVE FullAWSAccess from the OU/account
// Step 2: Attach explicit allow

{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowApprovedServicesOnly",
    "Effect": "Allow",
    "Action": [
      "s3:*",
      "ec2:*",
      "rds:*",
      "lambda:*",
      "iam:*",
      "sts:*",
      "cloudwatch:*",
      "logs:*"
    ],
    "Resource": "*"
  }]
}
```

**Advantages:**
- Maximum security (only explicitly allowed services work)
- New services are blocked by default (prevents shadow IT)
- Meets strict compliance requirements (FedRAMP, PCI)

**Disadvantages:**
- Must update SCP every time a new service is approved
- Easy to accidentally lock out accounts (including basic operations)
- 5,120 character limit makes comprehensive allow-lists challenging
- Multiple SCPs may be needed (max 5 per entity)

#### Advanced SCP Patterns

**Condition-based exemptions (break-glass):**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyAllExceptBreakGlass",
    "Effect": "Deny",
    "Action": [
      "guardduty:DeleteDetector",
      "config:StopConfigurationRecorder"
    ],
    "Resource": "*",
    "Condition": {
      "ArnNotLike": {
        "aws:PrincipalArn": [
          "arn:aws:iam::*:role/BreakGlassRole",
          "arn:aws:iam::*:role/aws-reserved/sso.amazonaws.com/*"
        ]
      }
    }
  }]
}
```

**Require specific tags on resource creation:**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyEC2WithoutCostCenter",
    "Effect": "Deny",
    "Action": "ec2:RunInstances",
    "Resource": "arn:aws:ec2:*:*:instance/*",
    "Condition": {
      "Null": {
        "aws:RequestTag/CostCenter": "true"
      }
    }
  }]
}
```

**Deny actions outside specific VPC (data perimeter):**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyOutsideVPC",
    "Effect": "Deny",
    "Action": [
      "s3:GetObject",
      "s3:PutObject"
    ],
    "Resource": "*",
    "Condition": {
      "StringNotEquals": {
        "aws:SourceVpc": "vpc-abc123"
      },
      "ArnNotLike": {
        "aws:PrincipalArn": "arn:aws:iam::*:role/AdminRole"
      }
    }
  }]
}
```

**Restrict instance types (cost control):**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyExpensiveInstances",
    "Effect": "Deny",
    "Action": "ec2:RunInstances",
    "Resource": "arn:aws:ec2:*:*:instance/*",
    "Condition": {
      "ForAnyValue:StringNotLike": {
        "ec2:InstanceType": [
          "t3.*",
          "t3a.*",
          "m5.large",
          "m5.xlarge"
        ]
      }
    }
  }]
}
```

**Prevent S3 public access:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyPublicBucketACL",
      "Effect": "Deny",
      "Action": "s3:PutBucketAcl",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "s3:x-amz-acl": ["public-read", "public-read-write", "authenticated-read"]
        }
      }
    },
    {
      "Sid": "DenyPublicObjectACL",
      "Effect": "Deny",
      "Action": "s3:PutObjectAcl",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "s3:x-amz-acl": ["public-read", "public-read-write", "authenticated-read"]
        }
      }
    }
  ]
}
```

#### SCP Limits & Quotas

| Resource | Limit |
|----------|-------|
| SCPs per organization | 1,000 |
| SCPs per entity (OU or account) | 5 |
| SCP document size | 5,120 characters (including whitespace) |
| SCP nesting (OU depth) | 5 levels |
| Condition keys per statement | No hard limit (bounded by doc size) |
| Statements per SCP | No hard limit (bounded by doc size) |

---

### Permission Boundaries — Deep Dive

**What they are:** An advanced IAM feature that sets the MAXIMUM permissions an IAM entity (user or role) can have. Like an SCP but at the individual identity level.

#### How Permission Boundaries Work

```
┌─────────────────────────────────────────────────────────┐
│              Permission Boundary Evaluation               │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  IAM User/Role has:                                       │
│    Identity Policy: Allow s3:*, ec2:*, rds:*, lambda:*   │
│    Permission Boundary: Allow s3:*, ec2:*, iam:Get*      │
│                                                           │
│  Effective permissions = Intersection:                     │
│    s3:* ✓ (in both)                                       │
│    ec2:* ✓ (in both)                                      │
│    rds:* ✗ (NOT in boundary → denied)                    │
│    lambda:* ✗ (NOT in boundary → denied)                 │
│    iam:Get* ✗ (in boundary but NOT in identity policy)   │
│                                                           │
│  Result: Only s3:* and ec2:* are effective                │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

**Key principle:** Effective permissions = Identity Policy ∩ Permission Boundary

- Permission boundary does NOT grant access (just like SCP)
- Identity policy alone doesn't work if action isn't in boundary
- Boundary alone doesn't grant if action isn't in identity policy
- Both must allow for the action to be effective

#### Permission Boundary Use Cases

**Use Case 1: Delegated Administration (Primary Exam Use Case)**

Allow developers to create IAM roles for their applications WITHOUT being able to escalate privileges:

```json
// Permission Boundary policy (attached to roles devs create)
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "s3:*",
      "dynamodb:*",
      "lambda:*",
      "logs:*",
      "sqs:*",
      "sns:*"
    ],
    "Resource": "*"
  }]
}
```

```json
// Developer's IAM policy (allows creating roles WITH boundary attached)
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "CreateRolesWithBoundary",
      "Effect": "Allow",
      "Action": [
        "iam:CreateRole",
        "iam:AttachRolePolicy",
        "iam:PutRolePolicy"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "iam:PermissionsBoundary": "arn:aws:iam::123456789012:policy/DevBoundary"
        }
      }
    },
    {
      "Sid": "DenyRemovingBoundary",
      "Effect": "Deny",
      "Action": [
        "iam:DeleteRolePermissionsBoundary",
        "iam:PutRolePermissionsBoundary"
      ],
      "Resource": "*"
    }
  ]
}
```

**What this achieves:**
- Developers can create IAM roles for their Lambda functions, etc.
- Every role they create MUST have the DevBoundary attached
- Even if they attach AdministratorAccess to the role, the boundary limits it
- They can't remove or change the boundary
- **Result:** Self-service IAM without privilege escalation risk

**Use Case 2: Contractor/Partner Access Limitation**

```json
// Boundary: Restrict contractors to specific resources
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:*", "ec2:Describe*"],
    "Resource": "*",
    "Condition": {
      "StringEquals": {
        "aws:RequestedRegion": "eu-west-1"
      }
    }
  }]
}
```

**Use Case 3: Limiting Service Roles Created by Automation**

CI/CD pipelines that create execution roles for Lambda/ECS tasks — boundary prevents the pipeline from creating over-privileged roles.

#### Permission Boundary vs SCP

| Dimension | SCP | Permission Boundary |
|-----------|-----|---------------------|
| **Scope** | Entire account (all identities) | Single IAM user or role |
| **Applies to** | Member accounts in org | Any IAM user/role (any account) |
| **Requires** | AWS Organizations | Just IAM (no Organizations needed) |
| **Affects root user?** | Yes (member accounts) | No (root is never bounded) |
| **Set by** | Organization admins | Account admins (or automation) |
| **Primary use case** | Org-wide guardrails | Delegated IAM admin, privilege escalation prevention |
| **Management** | Per-OU or per-account | Per-user or per-role |
| **Inheritance** | OU hierarchy (cascades down) | None (explicit per-entity) |
| **Max per entity** | 5 SCPs | 1 boundary per user/role |
| **Policy type** | Organization policy | IAM managed policy |

#### Permission Boundary Evaluation with SCPs

When BOTH SCP and Permission Boundary apply:

```
Effective = SCP ∩ Permission Boundary ∩ Identity Policy

Example:
  SCP allows:        s3:*, ec2:*, rds:*, lambda:*
  Boundary allows:   s3:*, ec2:*, dynamodb:*
  Identity allows:   s3:*, dynamodb:*, rds:*

  Effective = s3:* only
  (only action allowed by ALL THREE)
```

**Evaluation order:**
1. Explicit Deny (in any policy type) → DENY
2. SCP must allow → otherwise DENY
3. Permission Boundary must allow → otherwise DENY
4. Identity policy must allow → otherwise DENY
5. All allow → ALLOW

---

### Session Policies

**What they are:** Policies passed during role assumption (`sts:AssumeRole`) or federation that limit the session's permissions. Another boundary mechanism.

```json
// When assuming a role, pass a session policy:
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/MyRole \
  --role-session-name MySession \
  --policy '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":"s3:GetObject","Resource":"arn:aws:s3:::my-bucket/*"}]}'
```

**Evaluation:**
- Session Policy sets a boundary on the role session
- Effective = Role's Identity Policies ∩ Session Policy
- Used for: Temporary further restriction of assumed roles (e.g., vending machine pattern)

**Key facts for exam:**
- Session policies are optional (if not provided, role has full role permissions)
- Session policies CANNOT exceed the role's permissions (only restrict further)
- Max session policy size: 2,048 characters (inline) or use managed policy ARNs
- Applies to: AssumeRole, AssumeRoleWithSAML, AssumeRoleWithWebIdentity, GetFederationToken

---

### Resource-Based Policies & Cross-Account Interaction

**Critical cross-account evaluation difference:**

**Same-account access:**
```
Resource policy OR identity policy can grant access independently
(Either one alone is sufficient)
```

**Cross-account access:**
```
BOTH resource policy AND identity policy must allow
(Exception: If resource policy specifies the IAM principal directly)
```

**Cross-account with SCP:**
```
If caller is in an org member account:
  SCP must also allow the action
  
Effective = SCP (caller's account) ∩ Identity Policy (caller) ∩ Resource Policy (target)
```

**Important cross-account resource policy behavior:**
- S3 bucket policy granting access to Account B's role: Account B's SCP MUST also allow s3 actions
- KMS key policy granting access cross-account: Same rule applies
- **If SCP denies s3:* for Account B, then even a resource policy granting Account B access won't work**

---

### Resource Control Policies (RCPs)

**What they are (2024):** New organization policy type that restricts who can access resources in your organization's accounts. Complementary to SCPs.

**SCP vs RCP:**
- **SCP:** Controls what identities IN your org can DO (identity-centric, outbound)
- **RCP:** Controls who can ACCESS resources in your org (resource-centric, inbound)

**Example RCP — Deny external principals from accessing S3:**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyExternalS3Access",
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:*",
    "Resource": "*",
    "Condition": {
      "StringNotEquals": {
        "aws:PrincipalOrgID": "o-myorgid"
      }
    }
  }]
}
```

**Key facts:**
- Apply at OU or account level (like SCPs)
- Prevent resource-based policies from granting access to external principals
- Useful for data perimeter (prevent data exfiltration)
- Complement SCPs — SCPs restrict your identities, RCPs restrict access to your resources

---

## 2. Design Patterns & Best Practices

### SCP Design Patterns

#### Pattern 1: Layered Deny List (Most Common)

```
Root: FullAWSAccess + Base Deny (leave org, disable logging)
├── Security OU: + Deny modifying security resources
├── Workloads OU: + Deny expensive instances
│   ├── Production OU: + Deny delete actions, require encryption
│   └── Development OU: (more permissive, fewer denies)
└── Sandbox OU: + Deny non-approved regions only
```

#### Pattern 2: Compliance Zones (Allow-List for Regulated OUs)

```
Root: FullAWSAccess
├── Standard OU: FullAWSAccess + deny list
└── Regulated OU: REMOVE FullAWSAccess, attach explicit allow-list
    ├── PCI OU: Allow only PCI-approved services
    └── HIPAA OU: Allow only HIPAA-eligible services
```

#### Pattern 3: Data Perimeter (SCP + RCP combination)

```
SCP: Prevent identities from accessing external resources
  - Deny s3:* unless target bucket is in org
  - Deny kms:* unless key is in org

RCP: Prevent external identities from accessing our resources
  - Deny access to our S3 unless principal is in org
  - Deny access to our KMS unless principal is in org

Result: Data can't leave OR enter the org boundary
```

### Permission Boundary Design Patterns

#### Pattern 1: Developer Self-Service IAM

```
Admin creates:
  1. Permission Boundary policy (max permissions for dev-created roles)
  2. Developer IAM policy that:
     a. Allows iam:CreateRole WITH condition requiring boundary
     b. Allows iam:AttachRolePolicy
     c. Denies iam:DeleteRolePermissionsBoundary
     d. Denies creating roles WITHOUT boundary
```

#### Pattern 2: CI/CD Pipeline Boundary

```
Pipeline role creates execution roles for deployed services:
  1. Pipeline has permission to create roles
  2. Condition: Must attach "ServiceBoundary" to created roles
  3. ServiceBoundary limits what deployed services can do
  4. Pipeline cannot exceed its own boundary when creating roles
```

#### Pattern 3: Multi-Tenant Isolation

```
Tenant-A role: Identity Policy allows dynamodb:* on table-A
  + Boundary limits to dynamodb:* on table-A only

Even if someone misconfigures the identity policy to allow table-B:
  Boundary prevents access to table-B
```

### Anti-Patterns

| Anti-Pattern | Why It's Wrong | Better Approach |
|--------------|---------------|-----------------|
| Using SCPs as primary access control | SCPs don't grant — users still need IAM policies | SCPs = guardrails, IAM = grants |
| SCP allow-list for all OUs | Too restrictive, high maintenance, breaks things | Allow-list only for compliance OUs, deny-list everywhere else |
| No break-glass escape hatch in SCPs | Locked out during emergencies | SCP condition exempting break-glass role |
| Permission boundary without enforcing attachment | Developers skip boundary when creating roles | Condition key requiring boundary on iam:CreateRole |
| Overly broad SCPs at Root level | Blocks legitimate use across entire org | Apply restrictive SCPs at lower OUs where they're needed |
| Not testing SCPs before applying | Can lock out entire OUs | Test in Sandbox OU first, use AWS Policy Simulator |
| Single SCP trying to do everything | Hits 5,120 char limit | Multiple focused SCPs (up to 5 per entity) |
| Boundary that's too permissive (Allow *) | Defeats the purpose | Scope to specific services needed for the use case |
| Forgetting about service-linked roles | SLRs bypass SCPs, unexpected behavior | Understand which services use SLRs (Config, Org, etc.) |

### Well-Architected Framework Alignment

**Security (primary pillar):**
- SCPs: Defense-in-depth at organizational boundary
- Permission boundaries: Prevent privilege escalation
- Both: Enforce least privilege at scale
- Data perimeter: SCP + RCP prevents unauthorized data movement

**Operational Excellence:**
- SCPs: Consistent governance across all accounts
- Permission boundaries: Enable self-service without admin bottleneck
- Version control SCPs in Git, deploy via pipeline (IaC)

**Cost:**
- SCPs: Restrict expensive instance types, regions, services
- Permission boundaries: Prevent developers from creating costly resources
- Both: Proactive cost control (preventive, not reactive)

**Reliability:**
- SCPs: Prevent accidental deletion of critical resources
- SCPs: Prevent disabling of monitoring/alerting services
- Permission boundaries: Prevent over-provisioned roles that increase attack surface

---

## 3. Security & Compliance

### Privilege Escalation Prevention

**The problem:** A user with `iam:CreateRole` + `iam:AttachRolePolicy` can create a role with AdministratorAccess and assume it → full admin escalation.

**Solution with Permission Boundaries:**

```json
// Step 1: Create a boundary policy
// arn:aws:iam::123456789012:policy/DeveloperBoundary
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "s3:*",
      "dynamodb:*",
      "lambda:*",
      "sqs:*",
      "sns:*",
      "logs:*",
      "cloudwatch:*",
      "xray:*"
    ],
    "Resource": "*"
  }]
}
```

```json
// Step 2: Developer's policy — can create roles but MUST attach boundary
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCreateRoleWithBoundary",
      "Effect": "Allow",
      "Action": "iam:CreateRole",
      "Resource": "arn:aws:iam::123456789012:role/app-*",
      "Condition": {
        "StringEquals": {
          "iam:PermissionsBoundary": "arn:aws:iam::123456789012:policy/DeveloperBoundary"
        }
      }
    },
    {
      "Sid": "AllowAttachPolicy",
      "Effect": "Allow",
      "Action": [
        "iam:AttachRolePolicy",
        "iam:PutRolePolicy"
      ],
      "Resource": "arn:aws:iam::123456789012:role/app-*"
    },
    {
      "Sid": "DenyBoundaryModification",
      "Effect": "Deny",
      "Action": [
        "iam:DeleteRolePermissionsBoundary",
        "iam:PutRolePermissionsBoundary",
        "iam:CreatePolicyVersion"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyCreatingRolesWithoutBoundary",
      "Effect": "Deny",
      "Action": "iam:CreateRole",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "iam:PermissionsBoundary": "arn:aws:iam::123456789012:policy/DeveloperBoundary"
        }
      }
    }
  ]
}
```

**Security chain:**
1. Developer creates `app-myfunction-role` with DeveloperBoundary attached ✓
2. Developer attaches `AdministratorAccess` to that role
3. Role effectively can ONLY do s3, dynamodb, lambda, sqs, sns, logs, cloudwatch, xray
4. Developer cannot remove or change the boundary
5. Developer cannot modify the boundary policy itself

### Data Perimeter Pattern (SCP + RCP + VPC Endpoint Policies)

**Three controls for a complete data perimeter:**

```
1. SCP (identity perimeter): "My identities can only access MY resources"
   → Deny access to resources outside the org

2. RCP (resource perimeter): "My resources can only be accessed by MY identities"
   → Deny access from principals outside the org

3. VPC Endpoint Policy (network perimeter): "Traffic from my VPCs can only 
   reach MY resources, and only from MY identities"
   → Restrict what can pass through VPC endpoints
```

**SCP for identity perimeter:**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyAccessToExternalS3",
    "Effect": "Deny",
    "Action": "s3:*",
    "Resource": "*",
    "Condition": {
      "StringNotEquals": {
        "aws:ResourceOrgID": "${aws:PrincipalOrgID}"
      },
      "ArnNotLike": {
        "aws:PrincipalArn": "arn:aws:iam::*:role/ExternalIntegrationRole"
      }
    }
  }]
}
```

### Compliance Frameworks & Policy Mapping

| Compliance Requirement | SCP Implementation |
|----------------------|-------------------|
| Data residency (GDPR) | Deny actions outside approved regions |
| Encryption at rest | Deny creating unencrypted resources (S3, EBS, RDS) |
| No public exposure | Deny public S3 ACLs, public security groups |
| Audit trail integrity | Deny CloudTrail/Config modification |
| Approved services only (FedRAMP) | Allow-list SCP for compliant services |
| Prevent data exfiltration | Deny access to external S3/KMS (identity perimeter) |
| Restrict root user | Deny all actions when principal is root |
| MFA enforcement | Deny actions unless MFA is present |

### IAM Policy Evaluation with All Boundary Types

**Complete evaluation for a request from an org member account:**

```
1. Explicit Deny check (any policy type) → DENY wins immediately
2. SCP check → must allow (or implicit deny)
3. RCP check (if applicable) → must not deny
4. Resource-based policy → may allow (same-account only)
5. Permission Boundary → must allow
6. Session Policy (if in assumed role session) → must allow
7. Identity Policy → must allow
8. All checks pass → ALLOW
```

**Exam scenario:** User in member account, with permission boundary, in a role session with session policy, accessing a cross-account resource:

```
Effective = SCP ∩ Permission Boundary ∩ Session Policy ∩ Identity Policy
(AND the target resource's policy must allow cross-account access)
```

---

## 4. Cost Optimization

### Cost Control via SCPs

**Restrict expensive instance types:**
```json
{
  "Effect": "Deny",
  "Action": "ec2:RunInstances",
  "Resource": "arn:aws:ec2:*:*:instance/*",
  "Condition": {
    "ForAnyValue:StringNotLike": {
      "ec2:InstanceType": ["t3.*", "t3a.*", "m5.large", "m5.xlarge"]
    }
  }
}
```

**Deny expensive services in sandbox:**
```json
{
  "Effect": "Deny",
  "Action": [
    "redshift:*",
    "es:*",
    "sagemaker:CreateNotebookInstance",
    "sagemaker:CreateTrainingJob",
    "ec2:RequestSpotFleet"
  ],
  "Resource": "*"
}
```

**Prevent RI/SP purchases outside billing account:**
```json
{
  "Effect": "Deny",
  "Action": [
    "ec2:PurchaseReservedInstancesOffering",
    "savingsplans:CreateSavingsPlan",
    "rds:PurchaseReservedDBInstancesOffering"
  ],
  "Resource": "*"
}
```

**Deny creating resources without cost allocation tags:**
```json
{
  "Effect": "Deny",
  "Action": [
    "ec2:RunInstances",
    "rds:CreateDBInstance",
    "lambda:CreateFunction"
  ],
  "Resource": "*",
  "Condition": {
    "Null": {
      "aws:RequestTag/CostCenter": "true"
    }
  }
}
```

### Cost of Policies Themselves

| Item | Cost |
|------|------|
| SCPs | Free (no charge for SCP evaluation) |
| Permission Boundaries | Free (IAM policies are free) |
| AWS Organizations | Free |
| Control Tower | Free (underlying services may charge: Config, CloudTrail, S3) |
| AWS Config Rules (detective guardrails) | Per-rule, per-evaluation charge |
| CloudTrail (org trail) | Per-event charge for management events, S3 storage |

### Cost Optimization Best Practices

- Use SCPs to prevent cost overruns proactively (cheaper than reactive alerts)
- Restrict expensive instance families in dev/sandbox OUs
- Deny direct RI/SP purchases (centralize in billing account)
- Require cost tags via SCP → enables accurate cost attribution
- Restrict regions to reduce operational surface (fewer resources to manage = less cost)
- Permission boundaries prevent dev-created roles from accessing expensive services

---

## 5. High Availability, Disaster Recovery & Resilience

### SCP Availability Characteristics

**SCP evaluation availability:**
- SCPs are evaluated by the AWS IAM authorization service
- IAM is a global service with extremely high availability
- SCP evaluation does NOT add latency to API calls (evaluated server-side)
- If Organizations service is temporarily unavailable: **Existing SCPs continue to be enforced** (cached)

**SCP propagation:**
- Changes to SCPs propagate with **eventual consistency**
- Typical propagation time: Seconds to a few minutes
- During propagation: Some requests may be evaluated against old SCP
- **Not instantaneous** — don't rely on instant enforcement

### Resilience Concerns

**Accidental lockout scenarios:**
1. Removing FullAWSAccess without replacement → account locked out
2. Overly restrictive SCP attached to wrong OU → affects unintended accounts
3. Deny SCP that blocks sts:AssumeRole → breaks cross-account access
4. Allow-list SCP missing critical services (CloudWatch, CloudTrail) → breaks baseline

**Mitigation strategies:**
- Always test SCPs in sandbox OU first
- Use AWS Policy Simulator to validate
- Maintain a break-glass role exempted from deny SCPs
- Keep management account available for emergency access (OrganizationAccountAccessRole)
- Version-control SCPs in Git with PR review
- Use Control Tower proactive guardrails to catch issues before deployment

**Recovery from lockout:**
1. Management account is NEVER affected by SCPs → can always intervene
2. Management account assumes `OrganizationAccountAccessRole` in affected account
3. Detach or modify the problematic SCP from the OU/account
4. Propagation delay: Recovery is also eventually consistent

### Multi-Region Considerations

- SCPs are **global** (apply across all regions)
- A region-restrict SCP affects all regions simultaneously
- No per-region SCP application (it's org-wide)
- If you need different policies per region → use IAM policy conditions within accounts

### Resilience of Permission Boundaries

- Permission boundaries are IAM managed policies → stored in IAM (global, highly available)
- If the boundary policy is accidentally deleted: **The user/role loses all effective permissions** (boundary reference becomes invalid → implicit deny on everything)
- Mitigation: Use policy versioning, protect boundary policies with deny-delete conditions

---

## 6. Exam-Focused Section

### Straightforward Questions

**Q1:** A company uses AWS Organizations with SCPs. They need to ensure no IAM entity in member accounts can terminate EC2 instances tagged as "Critical". Which policy type should they use?

**A:** An SCP with an explicit Deny on `ec2:TerminateInstances` with a condition checking the resource tag `aws:ResourceTag/Classification: Critical`. Attach this SCP to the relevant OU or Root. Since SCPs restrict all IAM entities in member accounts (including root), no one can terminate instances with that tag.

---

**Q2:** An admin wants developers to create IAM roles for their Lambda functions but prevent them from creating roles with AdministratorAccess that they could then assume. What feature prevents this privilege escalation?

**A:** Permission Boundaries. Create a boundary policy that limits allowed actions to specific services (S3, DynamoDB, Lambda, etc.). Require developers to attach this boundary when creating roles (via IAM policy condition `iam:PermissionsBoundary`). Even if they attach AdministratorAccess to the role, effective permissions are bounded.

---

**Q3:** An SCP attached to an OU allows `ec2:*` and `s3:*` only (allow-list). A user in an account under that OU has an IAM policy granting `rds:*`. Can they access RDS?

**A:** **No.** The SCP uses an allow-list strategy, meaning only ec2 and s3 are permitted. Since rds is not in the SCP's allow list, it's implicitly denied by the SCP. Even though the IAM policy grants rds:*, effective permissions = SCP ∩ IAM = only ec2 and s3 actions the IAM policy also grants.

---

**Q4:** A company wants to prevent all member accounts from creating VPC peering connections outside the organization. Which approach is most effective?

**A:** An SCP that denies `ec2:CreateVpcPeeringConnection` when the `ec2:AccepterVpc` doesn't belong to the organization. Use condition `StringNotEquals` with `aws:PrincipalOrgID` on the accepter account. This prevents peering with external accounts while allowing internal peering.

---

**Q5:** A company uses session policies when vending temporary credentials via a broker. The role has broad S3 access, but session policies restrict each session to a specific S3 prefix based on the user. How does this work?

**A:** Session policies are passed during `sts:AssumeRole` and further restrict the session's permissions. The role's identity policy allows broad S3 access, but the session policy limits each session to `s3:*` on `arn:aws:s3:::bucket/user-prefix/*` only. Effective = Role Policy ∩ Session Policy. Each user gets credentials limited to their prefix.

---

**Q6:** Can an SCP prevent the root user of a member account from performing actions?

**A:** **Yes.** SCPs apply to ALL principals in member accounts, INCLUDING the root user. This is unique to SCPs — IAM policies cannot restrict root. However, SCPs do NOT apply to the root user of the management account. This is why SCPs are the only way to restrict root user access in member accounts.

---

**Q7:** A company has a permission boundary attached to a role. They also have an SCP on the account. A resource-based policy on an S3 bucket in the same account grants access to this role. Does the role have access?

**A:** **For same-account access with resource-based policies:** The resource-based policy can grant access independently of identity policies, BUT the SCP and permission boundary still apply. If the SCP or boundary denies the action, it's denied regardless of the resource policy. For the access to work: SCP must allow, boundary must allow, AND (identity policy OR resource policy) must grant.

---

### Tricky / Scenario-Based Questions

**Q1:** A developer creates a Lambda execution role and attaches the `DeveloperBoundary` permission boundary as required. They then attach the AWS managed policy `AmazonS3FullAccess`. Later, the security team updates the `DeveloperBoundary` to remove S3 write access (only allows `s3:Get*`). Without modifying the Lambda role's identity policy, what happens to S3 write operations?

**A:** **S3 writes are immediately denied.** Permission boundaries are evaluated at request time (not at policy attachment time). When the boundary is updated to only allow `s3:Get*`, the effective permissions change immediately: Role has `s3:*` (from identity policy) ∩ `s3:Get*` (from boundary) = only `s3:Get*`. Write operations fail. **Key concept:** Updating a boundary policy propagates to all entities using it without reattachment. This is powerful for centralized control.

---

**Q2:** Account A (member) has an SCP that denies `s3:PutObject` to external buckets. Account B (also a member in the same org) has an S3 bucket with a resource policy allowing Account A's role to PutObject. Can Account A's role write to Account B's bucket?

**A:** **Yes.** The SCP denies access to "external" buckets (outside the org). Since Account B is in the same org, the `aws:ResourceOrgID` condition matches. The SCP allows access within the org. Cross-account evaluation: Account A's SCP allows (same org), Account A's identity policy must allow `s3:PutObject`, Account B's bucket policy allows Account A's role. All conditions met → access granted. **Key concept:** "External" in data perimeter SCPs typically means outside the org (checked via `aws:ResourceOrgID` or `aws:PrincipalOrgID`).

---

**Q3:** A company attaches an SCP to an OU that denies `iam:*`. A service-linked role (`AWSServiceRoleForAutoScaling`) in an account under that OU needs to call `iam:CreateServiceLinkedRole`. Does the SCP block this?

**A:** **No.** SCPs do NOT affect service-linked roles (SLRs). Service-linked roles perform actions on behalf of AWS services and are explicitly exempt from SCP evaluation. Auto Scaling can still create/use its service-linked role regardless of the `iam:*` deny SCP. **Key concept:** SLRs are designed to always work for their respective services. If the exam mentions a service-linked role and an SCP, the SCP won't block the SLR.

---

**Q4:** A company uses permission boundaries for their development team. A developer creates a role with the required boundary. The role's identity policy allows `dynamodb:*`. The boundary allows `dynamodb:*`. An SCP on the OU denies `dynamodb:DeleteTable`. Can the role delete a DynamoDB table?

**A:** **No.** Even though both the identity policy and permission boundary allow `dynamodb:*`, the SCP explicitly denies `dynamodb:DeleteTable`. Evaluation: (1) Explicit deny in SCP → DENY. The SCP supersedes both the permission boundary and identity policy. **Hierarchy of evaluation:** Explicit Deny (any policy type) > any Allow. SCPs, boundaries, and identity policies all contribute to deny evaluation. **Key concept:** Deny always wins, regardless of which policy type it's in.

---

**Q5:** An organization has an SCP that restricts regions to `us-east-1` and `eu-west-1`. A developer needs to create a CloudFront distribution (which is a global service with API endpoint in `us-east-1`). The developer is also trying to create an ACM certificate in `us-east-1` for CloudFront. Will the SCP block either action?

**A:** **Neither action is blocked** if the SCP is correctly written using `aws:RequestedRegion` condition. CloudFront API calls go through `us-east-1` (global services route there), and ACM in `us-east-1` is explicitly allowed. **HOWEVER**, if the SCP denies based on `aws:RequestedRegion` for ALL actions without excluding global services, CloudFront creation would be allowed (it uses `us-east-1`) but other global services might be affected. **Best practice:** Region-deny SCPs should exclude global services via `NotAction` (IAM, STS, Support, Organizations, CloudFront, Route 53, WAF Global). **Key concept:** Region SCPs must account for global services.

---

**Q6:** A security team deploys a permission boundary across all developer roles. Six months later, they realize the boundary includes `iam:PassRole` without a resource condition. A developer discovers they can pass their role (which has broad permissions due to the boundary) to an EC2 instance, then SSH to that instance and act with the role's permissions. Is this a vulnerability?

**A:** **Yes, potentially.** If the boundary allows `iam:PassRole` and the identity policy also allows it, the developer can pass a role to a service (EC2, Lambda). The passed role's permissions are limited by its OWN boundary, but if the developer's role IS the one passed (or another broad role), this enables indirect privilege escalation. **Fix:** (1) Restrict `iam:PassRole` to specific role ARNs in the developer's identity policy. (2) Add condition `iam:PassedToService` to restrict which services can receive the role. (3) Ensure passed roles also have appropriate boundaries. **Key concept:** `iam:PassRole` is a privilege escalation vector even with boundaries — it must be scoped.

---

**Q7:** A company moves an account from "Dev OU" (SCP allows all) to "Prod OU" (SCP denies delete/terminate actions). Existing EC2 instances in the account were launched without tags. The Prod OU SCP requires all RunInstances to have a `CostCenter` tag. What happens to existing instances and future operations?

**A:** (1) **Existing instances are unaffected.** SCPs are evaluated at request time — they don't retroactively modify resources. Untagged instances continue to run. (2) **New instances** must have the `CostCenter` tag (SCP enforces on RunInstances). (3) **Terminating existing instances** is denied (Prod SCP denies terminate). (4) **Problem:** Untagged instances can't be terminated AND new ones require tags → creates zombie untagged instances. **Fix:** Either (a) tag existing instances before moving account, or (b) temporarily exempt the account from the terminate-deny SCP while cleaning up. **Key concept:** SCPs are preventive (future actions), not corrective (existing state).

---

### Common Exam Traps & Pitfalls

1. **SCPs don't grant permissions.** An SCP with `"Effect": "Allow"` just sets the ceiling — users still need IAM policies. "Allow" in SCP ≠ "Grant."

2. **Permission boundary doesn't grant permissions either.** Same principle — it sets a maximum. Both boundary AND identity policy must allow.

3. **Explicit Deny wins everywhere.** Deny in SCP, boundary, identity policy, resource policy, or session policy → DENY. Doesn't matter where the allow is.

4. **SCPs don't apply to management account.** If the question involves protecting the management account → SCPs are wrong. Use IAM policies + MFA + operational controls.

5. **SCPs don't affect service-linked roles.** If a service uses an SLR, the SCP won't block it. The exam tests this with Config, Auto Scaling, Organizations itself.

6. **Permission boundary limit: ONE per user/role.** You can't stack multiple boundaries. Use one comprehensive boundary policy.

7. **Session policies only restrict (never expand).** If a role has limited permissions, a session policy can't add more. It can only narrow further.

8. **Cross-account: Resource policy alone can grant (same account).** But cross-account requires both sides OR explicit principal ARN in resource policy.

9. **`iam:PermissionsBoundary` condition key** is how you enforce boundary attachment. Without this condition on CreateRole, developers can skip the boundary.

10. **SCP + Permission Boundary + Identity Policy = triple intersection.** Effective permissions = what's allowed by ALL THREE simultaneously. Action must be in the intersection.

11. **Root user in member accounts IS affected by SCPs.** This is unique — IAM can't restrict root, but SCPs can. Management account root is exempt.

12. **`NotAction` in SCPs ≠ `Deny Action`.** `NotAction` with Deny means "deny everything EXCEPT these actions." Common in region-restrict SCPs to exclude global services.

---

## 7. Cheat Sheet

### Must-Know Facts

| Fact | Detail |
|------|--------|
| SCP grants permissions? | NO — only restricts (sets ceiling) |
| Permission boundary grants? | NO — only restricts (sets ceiling) |
| SCP applies to management account? | NO — NEVER |
| SCP applies to root user (member)? | YES |
| SCP applies to service-linked roles? | NO |
| Permission boundary applies to root? | NO |
| Boundaries per user/role | 1 maximum |
| SCPs per entity | 5 maximum |
| SCP size limit | 5,120 characters |
| SCP propagation | Eventually consistent (seconds-minutes) |
| Effective permissions | SCP ∩ Boundary ∩ Session Policy ∩ Identity Policy |
| Deny wins? | YES — explicit deny in ANY policy type = denied |
| Session policy adds permissions? | NO — only restricts role session further |
| Who sets SCPs? | Organization admins (management or delegated) |
| Who sets boundaries? | Account admins (on users/roles they manage) |
| Allow-list SCP requires? | Removing FullAWSAccess + explicit allow |
| Break-glass pattern | SCP condition exempting specific role |

### Policy Type Comparison

| Dimension | SCP | Permission Boundary | Session Policy | Identity Policy | Resource Policy |
|-----------|-----|---------------------|----------------|-----------------|-----------------|
| **Purpose** | Org guardrail | Identity guardrail | Session guardrail | Grant access | Grant access (resource) |
| **Grants?** | No | No | No | Yes | Yes |
| **Scope** | Account-wide | Per user/role | Per session | Per user/role | Per resource |
| **Requires Org?** | Yes | No | No | No | No |
| **Affects root?** | Yes (member) | No | No | No | Depends |
| **Max per entity** | 5 | 1 | 1 | 10+ (managed) | 1 (per resource) |
| **Exam keyword** | "org-wide restrict" | "delegated admin" | "temporary restrict" | "grant access" | "cross-account" |

### Decision Flowchart

```
Question mentions...                                          → Think...
──────────────────────────────────────────────────────────────────────────
"restrict all users in account, including root"              → SCP
"organization-wide restriction"                              → SCP
"management account can't be restricted by policy"           → SCP limitation (doesn't apply to mgmt)
"developer creates roles, prevent privilege escalation"      → Permission Boundary
"delegate IAM administration safely"                         → Permission Boundary
"limit assumed role session"                                 → Session Policy
"grant access to S3 from another account"                    → Resource Policy (bucket policy)
"what's the maximum the user can do?"                        → Intersection of all boundaries
"service-linked role still works despite SCP"                → SLRs exempt from SCPs
"deny always wins regardless of allows"                      → Explicit deny in any policy
"new org policy type for resource access"                    → Resource Control Policies (RCPs)
"prevent external access to our resources"                   → RCP
"prevent our identities accessing external resources"        → SCP (identity perimeter)
"data perimeter, prevent exfiltration"                       → SCP + RCP + VPC Endpoint Policy
"restrict to specific regions (org-wide)"                    → SCP with aws:RequestedRegion
"allow-list vs deny-list"                                    → Deny-list = recommended. Allow-list = compliance
```

---

## 8. Deep Dive: 10 More Tricky Scenario-Based Questions

**Q1:** A company has an SCP denying all actions for the root user in member accounts. An account's root user needs to perform a specific task that ONLY root can do (e.g., change account's support plan, close the account, or restore permissions on S3 with Bucket Owner Enforced). How can they accomplish this?

**A:** **They cannot from the member account if the SCP blocks root.** The SCP denying root user actions is absolute in member accounts. **Options:** (1) Temporarily modify the SCP to add an exception (e.g., time-limited condition). (2) Have the management account perform the action (management account root is not affected by SCPs). (3) For most tasks "only root can do" — the management account can perform them or the Organizations API can handle them. (4) Add a condition to the SCP allowing root for ONLY those specific actions that require root. **Key insight:** Some tasks genuinely require account root (like restoring S3 bucket policy access on Bucket Owner Enforced), so a blanket root-deny SCP needs exceptions or management account intervention.

---

**Q2:** A company uses permission boundaries for all Lambda execution roles. A developer writes a Lambda function that calls `sts:AssumeRole` to assume a role in another account. The boundary allows `sts:AssumeRole`. The cross-account role has AdministratorAccess with no boundary. Once the Lambda assumes the cross-account role, is it still bounded?

**A:** **No — once the Lambda assumes the cross-account role, the boundary on the original role no longer applies.** The new session operates under the cross-account role's permissions (subject to THAT role's policies, boundary, and THAT account's SCPs). **The risk:** A developer could bypass their boundary by assuming an over-privileged role elsewhere. **Mitigation:** (1) SCP in the developer's account restricting which roles can be assumed (`sts:AssumeRole` with resource condition). (2) Cross-account role's trust policy should restrict who can assume it. (3) SCP in the target account should also bound what's possible. **Key concept:** Permission boundaries are per-session. A new `AssumeRole` creates a new session under the target role's policies.

---

**Q3:** An organization has the following SCP hierarchy: Root (FullAWSAccess) → OU-A (Allow: s3:\*, ec2:\*) → Account-1 (Allow: s3:\*, rds:\*). What can a user in Account-1 do if their IAM policy allows `s3:*`, `ec2:*`, and `rds:*`?

**A:** **Only `s3:*`.** SCP evaluation with allow-list requires the action to be allowed at EVERY level. Root allows all (✓). OU-A allows s3 and ec2 only (ec2 passes, rds fails here). Account-1 allows s3 and rds only (ec2 fails here, rds was already blocked at OU-A). Only `s3:*` is in the intersection of all levels (Root ∩ OU-A ∩ Account-1 ∩ IAM). **Key concept:** In allow-list SCP strategy, the effective permission is the intersection across all levels. Each level can only narrow (never broaden) what was allowed above.

---

**Q4:** A company has an SCP denying `ec2:RunInstances` unless the `aws:RequestTag/Environment` tag is present. A developer uses Terraform to launch an EC2 instance with the tag "Environment: prod". Terraform also creates a security group in the same apply. The security group creation doesn't include the Environment tag. Does the security group creation fail?

**A:** **No — the SCP condition applies specifically to `ec2:RunInstances` on the instance resource.** Security group creation (`ec2:CreateSecurityGroup`) is a different action and the SCP condition only applies to RunInstances. However, if the SCP were broadly written as "Deny ec2:* unless tag present" → then yes, CreateSecurityGroup would fail. **Key concept:** SCP conditions are per-statement. Check exactly which actions and resources the condition applies to. Well-written SCPs scope conditions to specific actions and resource ARNs, not broad wildcards.

---

**Q5:** A company attaches a permission boundary to a role that allows `s3:*`. The role's identity policy also allows `s3:*`. An S3 bucket has a bucket policy with an explicit Deny for this role's ARN. Can the role access the bucket?

**A:** **No.** An explicit Deny in ANY policy (including resource-based policies) overrides all allows. The bucket policy's explicit Deny means the role cannot access that bucket, regardless of what the boundary and identity policy allow. **Evaluation:** Step 1 — Is there an explicit Deny? Yes (bucket policy denies this role) → DENY. **Key concept:** Explicit Deny wins everywhere — not just in SCPs and boundaries, but in resource policies too. This is the fundamental IAM principle.

---

**Q6:** A company updates their SCP to deny `lambda:CreateFunction` unless the function uses a specific VPC configuration. An existing Lambda function (created before the SCP) is running without VPC. Does it stop working?

**A:** **No.** The existing function continues to work. SCPs are evaluated at REQUEST time (API calls). The Lambda function was already created — it won't be re-evaluated retroactively. However, if someone tries to UPDATE the function configuration, or if the function is deleted and needs recreation, the SCP will apply to those new API calls. Invocations of the existing function are NOT API calls blocked by this SCP (invocation uses `lambda:InvokeFunction`, not `CreateFunction`). **Key concept:** SCPs are preventive (block new actions), not corrective (don't affect existing resources/configurations).

---

**Q7:** A developer has a permission boundary that allows `iam:CreateRole`. Their identity policy also allows `iam:CreateRole`. They create a role WITHOUT attaching a boundary to it. Is this possible?

**A:** **Yes, unless there's a separate control preventing it.** A permission boundary only limits what the bounded entity (the developer) can do. The boundary allows `iam:CreateRole` and the identity policy allows it → they can create roles. The boundary does NOT automatically force created roles to have boundaries. **To prevent this:** The developer's identity policy must include a condition requiring `iam:PermissionsBoundary` on CreateRole AND a deny for CreateRole without boundary. The boundary alone doesn't enforce boundary propagation. **Key concept:** Boundaries are not transitive. They limit the developer's actions but don't control properties of resources the developer creates — that requires conditions in the identity policy.

---

**Q8:** An account is in an OU with an SCP denying `kms:Decrypt` unless `kms:ViaService` is `s3.amazonaws.com`. A Lambda function in this account tries to decrypt data using KMS directly (SDK call, not through S3). Does it work?

**A:** **No.** The SCP denies `kms:Decrypt` unless the call comes via the S3 service (`kms:ViaService` condition). A direct KMS SDK call from Lambda won't satisfy this condition — there's no "via service" context. The decrypt is denied. **Use case:** This SCP pattern ensures KMS keys can only be used through specific services (preventing direct data decryption by users/roles while allowing S3 SSE-KMS to function). **Key concept:** `kms:ViaService` is a powerful condition for controlling how encryption keys are used. SCP can enforce it org-wide.

---

**Q9:** A company implements both an SCP (denying access outside eu-west-1) and a permission boundary (allowing only s3:* and dynamodb:*). A user in a member account has an identity policy allowing all actions. They try to call `s3:PutObject` in `us-east-1`. What happens?

**A:** **Denied by the SCP** (region restriction). Even though the permission boundary allows s3:* and the identity policy allows all actions, the SCP denies actions outside eu-west-1. Evaluation: (1) SCP check — `s3:PutObject` in us-east-1 → region not allowed → DENY. Never reaches boundary or identity policy evaluation. **If it were in eu-west-1:** SCP allows (✓), Boundary allows s3:* (✓), Identity allows * (✓) → ALLOW. **Key concept:** SCP is the outermost boundary. If it denies, nothing else matters.

---

**Q10:** A company wants to implement a "separation of duties" pattern: Security admins can manage security services but NOT application services. App admins can manage application services but NOT security services. Both are in the same account. Can they use permission boundaries for this?

**A:** **Permission boundaries alone are insufficient for separation of duties.** Boundaries set maximums but a user might have LESS than the boundary. The correct approach: (1) **Identity policies** grant what each admin CAN do (security admin gets security service access, app admin gets app service access). (2) **Permission boundaries** set maximums as a safety net (security admin boundary = security services only, app admin boundary = app services only). (3) **SCP** at the OU level can add additional guardrails. **Why boundary alone isn't enough:** If both admins' identity policies were granted `*` (which you'd fix with proper policies), the boundary would enforce separation. But it's belt AND suspenders — identity policies should be scoped too. **Key concept:** Defense in depth. Boundaries prevent escalation; identity policies grant appropriate access. Use both together.

---

## 9. Service Comparison Tables

### Authorization Mechanisms Compared

| Dimension | SCP | Permission Boundary | Session Policy | IAM Identity Policy | Resource Policy | RCP |
|-----------|-----|---------------------|----------------|---------------------|-----------------|-----|
| **Grants access?** | No | No | No | Yes | Yes | No |
| **Restricts?** | Yes (ceiling) | Yes (ceiling) | Yes (ceiling) | Yes (deny statements) | Yes (deny) | Yes (ceiling) |
| **Scope** | Account-level | User/role-level | Session-level | User/role-level | Resource-level | Account/OU-level |
| **Affects root?** | Yes (member) | No | No | No | Depends | TBD |
| **Cross-account?** | Org-wide | Per-account only | Per-session | Per-account | Grants cross-acct | Org-wide |
| **Set by** | Org admin | Account admin | Caller (AssumeRole) | Account admin | Resource owner | Org admin |
| **Required?** | No (FullAWSAccess default) | No (optional) | No (optional) | Yes (for access) | Varies by service | No |
| **Max per entity** | 5 | 1 | 1 | 10 managed + inline | 1 per resource | 5 |
| **Size limit** | 5,120 chars | 6,144 chars | 2,048 chars (inline) | 6,144 chars (managed) | Varies (20KB S3) | 5,120 chars |

### SCP Strategy Comparison

| Dimension | Deny List | Allow List |
|-----------|-----------|------------|
| **Base** | FullAWSAccess (allow all) | Remove FullAWSAccess |
| **Approach** | Block specific things | Allow only specific things |
| **Default posture** | Permissive (new services work) | Restrictive (new services blocked) |
| **Complexity** | Low (only write denies) | High (must list everything allowed) |
| **Risk** | Lower (hard to accidentally lock out) | Higher (easy to miss needed services) |
| **Maintenance** | Low (new services auto-allowed) | High (update for each new service) |
| **Size pressure** | Lower (fewer statements) | Higher (many allow statements) |
| **Best for** | General workload OUs | Compliance-regulated OUs (PCI, FedRAMP) |
| **Exam signal** | "block specific actions" | "only approved services" |

### Privilege Escalation Vectors & Mitigations

| Vector | How It Works | Mitigation |
|--------|-------------|------------|
| CreateRole + AttachPolicy | Create admin role, assume it | Permission boundary + condition |
| PassRole to service | Pass broad role to EC2/Lambda | Restrict iam:PassRole resource ARN |
| CreatePolicyVersion | Create new version of existing policy | Deny CreatePolicyVersion or restrict |
| SetDefaultPolicyVersion | Activate a previous broader version | Deny SetDefaultPolicyVersion |
| AddUserToGroup | Add self to admin group | Deny for non-admins |
| CreateAccessKey (other users) | Create keys for an admin | Restrict to own user ARN |
| AssumeRole (cross-account) | Assume over-privileged external role | SCP restricting target role ARNs |
| UpdateFunctionCode (Lambda) | Inject code into admin Lambda | Restrict UpdateFunctionCode resource |
| CreateEC2 with instance profile | Launch EC2 with broad role, SSH in | Restrict PassRole + instance profiles |

### When to Use Which Boundary Type

| Scenario | Use | Why |
|----------|-----|-----|
| Restrict entire accounts org-wide | SCP | Account-level, org-wide, includes root |
| Prevent privilege escalation for devs | Permission Boundary | Per-role, enforceable via conditions |
| Restrict specific session temporarily | Session Policy | Per-session, dynamic |
| Limit third-party contractor access | Permission Boundary | Limits regardless of policy attached |
| Data perimeter (org-wide) | SCP + RCP | Both identity and resource sides |
| Region restriction | SCP | Must be org-wide, include root user |
| Compliance (approved services only) | SCP (allow-list) | Account-level, non-bypassable |
| CI/CD pipeline restricting deployed roles | Permission Boundary | Roles created by pipeline are bounded |
| Vending machine pattern (per-tenant creds) | Session Policy | Different restrictions per session |
| Emergency override | SCP condition exemption | Break-glass role excluded from deny |

---

## 10. When to Pick Service A over Service B — Exam Keywords

### Pick SCP over Permission Boundary when:
- "organization-wide"
- "all accounts", "all member accounts"
- "restrict root user"
- "account-level guardrail"
- "compliance across org"
- "no one in the account can do X"
- **Exam signal:** Broad organizational restriction, includes root, no exceptions (except via condition)

### Pick Permission Boundary over SCP when:
- "delegate IAM administration"
- "developers create roles without escalation"
- "limit what a specific user/role can achieve"
- "prevent privilege escalation"
- "self-service IAM"
- "sandbox user permissions"
- **Exam signal:** Per-identity restriction, delegation, escalation prevention

### Pick Session Policy over Permission Boundary when:
- "temporary restriction for a specific session"
- "credential vending machine"
- "different permissions per assumed session"
- "broker issues restricted credentials"
- "per-request permission scoping"
- **Exam signal:** Dynamic, per-session, vending machine pattern

### Pick SCP (Deny List) over SCP (Allow List) when:
- "block specific dangerous actions"
- "general workload accounts"
- "minimal maintenance"
- "don't want to disrupt normal operations"
- "new services should work by default"
- **Exam signal:** Standard governance, specific blocks

### Pick SCP (Allow List) over SCP (Deny List) when:
- "only approved services"
- "FedRAMP", "PCI", "strict compliance"
- "new services must be explicitly approved"
- "default deny posture"
- **Exam signal:** Compliance, maximum restriction, audit-driven approval

### Pick RCP over SCP when:
- "prevent external access TO our resources"
- "resource-centric control"
- "even if resource policy allows external access"
- "data perimeter (inbound)"
- **Exam signal:** Controlling who accesses your resources (not what your identities do)

### Pick SCP over RCP when:
- "control what our identities can DO"
- "restrict our users from accessing external resources"
- "identity-centric control"
- "data perimeter (outbound)"
- **Exam signal:** Controlling your identities' actions (not who accesses your resources)

### Pick Permission Boundary + IAM Condition over just IAM Policy when:
- "admin creates roles for others"
- "ensure created roles can't exceed X permissions"
- "defense in depth"
- "non-bypassable maximum"
- **Exam signal:** Preventing the creator from giving the created entity more than intended

### Pick IAM Policy over Permission Boundary when:
- "grant access" (boundaries don't grant)
- "fine-grained access rules"
- "conditional access (MFA, time, IP)"
- "standard access management"
- **Exam signal:** Actually giving permissions to users/roles

---

## 11. "Gotcha" Differences the Exam Tests

### SCP Gotchas

1. **SCP "Allow" ≠ Grant.** An SCP allow statement only sets the ceiling. A user with an SCP allowing `s3:*` but NO IAM policy granting s3 → can't access S3. This confuses people who think "Allow" means "Grant."

2. **SCPs affect root in member accounts but NOT management account.** If the exam says "root user in Account X can't be stopped" — check if it's the management account (SCP doesn't apply) or member account (SCP applies).

3. **Service-linked roles bypass SCPs.** An SCP denying `config:*` won't prevent AWS Config from functioning via its SLR. The exam may present this as "Config still works despite the SCP — is it broken?"

4. **SCP deny at OU level blocks the entire subtree.** You can't "un-deny" at a child OU. Once denied in a parent, it's denied for all children. The only escape is a condition exclusion in the deny itself.

5. **5,120 character limit includes whitespace.** Minify your SCP JSON. Remove comments, extra spaces. This limit is surprisingly small for complex allow-lists.

6. **NotAction ≠ Deny Action.** `"NotAction": ["iam:*"], "Effect": "Deny"` means "deny everything EXCEPT IAM." It does NOT deny IAM. The exam LOVES testing this with region-restrict SCPs.

7. **Condition key `aws:PrincipalOrgID` vs `aws:ResourceOrgID`:** PrincipalOrgID = who's making the request. ResourceOrgID = what resource is being accessed. Confused in data perimeter SCPs.

8. **SCPs don't block organizations:LeaveOrganization from management account.** Only from member accounts. To prevent a member from leaving, attach the SCP to the member (not management).

### Permission Boundary Gotchas

9. **Only ONE boundary per entity.** You can't attach multiple boundaries to a role. If you need complex boundaries, create one comprehensive policy.

10. **Boundary is NOT transitive.** If User A (with boundary) creates Role B, Role B does NOT inherit User A's boundary. You must ENFORCE boundary on creation via IAM policy conditions.

11. **Deleting a boundary policy = instant lockout.** If you delete the policy used as a boundary (and it's still referenced by roles), those roles lose all effective permissions. The boundary reference becomes invalid → implicit deny everything.

12. **Boundary doesn't protect against direct resource policy grants (same account).** If a bucket policy grants a role s3:GetObject in the same account, the access is allowed even if it's not in the boundary. **EXCEPTION:** This is debated — AWS documentation says boundary IS checked even for resource-based policy grants in same account for IAM users (but NOT for IAM roles in some cases). **Safest exam answer:** Boundaries always apply for IAM users. For roles, resource-based policies in the same account can bypass boundaries.

13. **`iam:PermissionsBoundary` condition key exists ONLY for CreateRole and CreateUser.** You can't condition other IAM actions (AttachRolePolicy, PutRolePolicy) on the target role having a boundary. The enforcement is at creation time.

14. **Boundary scope vs identity policy scope:** A boundary allowing `s3:*` on `Resource: *` plus an identity policy allowing `s3:GetObject` on `arn:aws:s3:::my-bucket/*` → effective = `s3:GetObject` on `my-bucket/*` only. The boundary's broad resource doesn't expand the identity policy's narrow scope.

### Cross-Account Gotchas

15. **Cross-account with resource policy (role):** If Account B's bucket policy grants Account A's ROLE directly → Account A's role can access WITHOUT an identity policy. But Account A's SCP must still allow it.

16. **Cross-account with resource policy (user):** If Account B's bucket policy grants Account A's USER → Account A's user ALSO needs identity policy. Users require identity policy even with cross-account resource policy.

17. **Assumed role in Account B = Account B's SCPs apply.** If you assume a cross-account role, the session is subject to the TARGET account's SCPs. Your source account's SCPs applied during the AssumeRole call.

18. **Session policy on AssumeRole cross-account:** The session is bounded by: target role's policies ∩ session policy ∩ target account's SCP. Source account's SCP only evaluated during the initial AssumeRole API call.

### Evaluation Logic Gotchas

19. **Resource policy Deny + Identity Allow ≠ Allow.** Explicit deny in resource policy wins. Many think "but my identity policy allows it" — deny always wins.

20. **Resource policy Allow + Identity Deny ≠ Allow.** Same principle reversed. Deny anywhere = denied.

21. **No policy statement matching = implicit deny.** If no policy explicitly allows an action → denied by default. Absence of deny is NOT an allow.

22. **Multiple policies on same entity = union of all.** A user with Policy A (allow s3) and Policy B (allow ec2) can do both. Policies combine — but if Policy C denies s3, the deny wins over Policy A's allow.

---

## 12. Decision Tree: Choosing the Right Boundary Mechanism

```
START: What boundary/restriction is needed?
│
├─── Need to restrict ACROSS THE ORGANIZATION?
│    ├─── Restrict what identities CAN DO? → SCP
│    │    ├─── Block specific actions? → Deny-list SCP
│    │    ├─── Only allow specific services? → Allow-list SCP
│    │    ├─── Restrict to specific regions? → SCP + aws:RequestedRegion + NotAction for global
│    │    ├─── Restrict root user (member)? → SCP (only way to restrict root in member accts)
│    │    └─── Emergency override needed? → SCP with condition exempting break-glass role
│    │
│    └─── Restrict who can ACCESS our resources? → RCP
│         ├─── Prevent external principals? → RCP with aws:PrincipalOrgID condition
│         └─── Combined with SCP? → Full data perimeter (SCP + RCP + VPC endpoint policy)
│
├─── Need to restrict SPECIFIC USERS/ROLES (per-identity)?
│    ├─── Prevent privilege escalation? → Permission Boundary
│    │    ├─── Developers create roles? → Boundary + iam:PermissionsBoundary condition
│    │    ├─── Contractors limited scope? → Boundary limiting services/regions
│    │    └─── CI/CD creates execution roles? → Boundary on pipeline-created roles
│    │
│    ├─── Limit access to specific resources? → Identity Policy (scoped Resource ARN)
│    │
│    └─── Temporary session restriction? → Session Policy
│         ├─── Credential vending machine? → Session policy per-tenant
│         ├─── Federated user session? → Session policy at federation
│         └─── Broker-mediated access? → Session policy on AssumeRole call
│
├─── Need to restrict ACCESS TO A RESOURCE?
│    ├─── Same-account access control? → Resource Policy (bucket policy, key policy)
│    ├─── Cross-account grant? → Resource Policy + Identity Policy (both sides)
│    └─── Org-wide resource protection? → RCP
│
├─── Need DEFENSE IN DEPTH (multiple layers)?
│    │
│    Use combination:
│    ├─── Org-level: SCP (outermost boundary)
│    ├─── Identity-level: Permission Boundary (middle boundary)
│    ├─── Session-level: Session Policy (innermost boundary)
│    ├─── Identity grant: Identity Policy (what's actually allowed)
│    └─── Resource grant: Resource Policy (resource-side control)
│    │
│    Effective = SCP ∩ Boundary ∩ Session ∩ (Identity OR Resource policy)
│
└─── Need to GRANT ACCESS (not restrict)?
     ├─── To a user/role in same account? → Identity Policy
     ├─── To a principal in another account? → Resource Policy + Identity Policy
     ├─── To a service (Lambda, EC2, etc.)? → Identity Policy (execution role)
     └─── To everyone/public? → Resource Policy (Principal: "*") — usually avoid!
```
