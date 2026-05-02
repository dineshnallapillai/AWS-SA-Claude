# AWS IAM Identity Center (SSO) & Federation (SAML, OIDC)

---

## 1. Core Concepts & Theory

### AWS IAM Identity Center (formerly AWS SSO)

- **What it does:** Centralized access management for multiple AWS accounts and cloud applications using a single sign-on experience.
- **Key components:**
  - **Identity Source:** Where users/groups are stored — options: Identity Center directory (built-in), Active Directory (AD Connector or AWS Managed Microsoft AD), External IdP (SAML 2.0 — Okta, Azure AD, Ping, etc.).
  - **Permission Sets:** Collections of IAM policies that define access levels. Assigned to users/groups for specific AWS accounts.
  - **AWS Access Portal:** Web portal where users log in once and see all accounts/applications they have access to.
  - **Multi-Account Permissions:** Map permission sets to users/groups across multiple AWS accounts in the organization.
  - **Application Assignments:** Assign users/groups to pre-integrated or custom SAML/OIDC applications.
- **Deployment model:** Lives in ONE region (home region) in the management account or delegated admin account. Organization-wide scope.
- **Credential types issued:**
  - Short-term credentials via AWS Access Portal (console access).
  - Temporary credentials via CLI/SDK (`aws sso login` → cached token).
  - SAML assertions for integrated applications.
- **Limits & Quotas:**
  - 1 Identity Center instance per organization.
  - 2,000 permission sets per instance.
  - 100,000 users, 50,000 groups (Identity Center directory).
  - 50 inline policies + 20 managed policies per permission set (including AWS managed + customer managed).
  - Session duration: 1-12 hours (configurable per permission set).
  - 3,000 account assignments per permission set.

### SAML 2.0 Federation

- **What it does:** Enables federated SSO between an external Identity Provider (IdP) and AWS as a Service Provider (SP).
- **How it works (IdP-initiated & SP-initiated):**
  1. User authenticates with corporate IdP (Okta, ADFS, Azure AD).
  2. IdP generates SAML assertion (XML) containing user attributes and roles.
  3. User's browser POSTs assertion to AWS STS endpoint (`https://signin.aws.amazon.com/saml`).
  4. STS validates assertion, calls `AssumeRoleWithSAML`, returns temporary credentials.
  5. User is redirected to AWS Console (or receives CLI credentials).
- **Key elements:**
  - **Trust relationship:** IAM role trusts `saml-provider` principal.
  - **SAML metadata XML:** Exchanged between IdP and AWS (describes endpoints, certificates).
  - **Attribute mapping:** Maps IdP attributes to AWS roles (`https://aws.amazon.com/SAML/Attributes/Role`).
  - **Session duration:** Max 12 hours (configured via `DurationSeconds` or SAML `SessionDuration` attribute).
- **IAM Identity Provider entity:** Created in IAM to represent the external IdP (stores metadata document).
- **Limits:**
  - 1 SAML assertion max size: 100KB.
  - Max roles in assertion: Any number, but user must choose ONE to assume.
  - SAML providers per account: 128.
  - Session duration: 15 min to 12 hours.

### OIDC (OpenID Connect) Federation

- **What it does:** Allows authentication via web identity providers (Google, Facebook, Amazon Cognito, any OIDC-compliant IdP) without storing long-term credentials.
- **How it works:**
  1. User authenticates with OIDC provider → receives ID token (JWT).
  2. Application calls `AssumeRoleWithWebIdentity` with the token.
  3. STS validates token against the OIDC provider's JWKS endpoint.
  4. Returns temporary AWS credentials scoped to the assumed IAM role.
- **Key elements:**
  - **IAM OIDC Identity Provider:** Entity in IAM representing the external OIDC provider (thumbprint + client ID).
  - **Audience (aud claim):** Must match the client ID registered in the IAM OIDC provider.
  - **Subject (sub claim):** Can be used in IAM policy conditions for fine-grained access.
- **Common use cases:**
  - Mobile/web apps authenticating via social providers.
  - CI/CD systems (GitHub Actions OIDC → AWS, GitLab CI OIDC → AWS).
  - EKS Pod Identity (IRSA — IAM Roles for Service Accounts — uses projected OIDC tokens).
- **Limits:**
  - OIDC providers per account: 128.
  - Client IDs per OIDC provider: 100.
  - Thumbprints per OIDC provider: 5.

### Amazon Cognito (Federation Context)

- **What it does:** Adds authentication, authorization, and user management to web/mobile apps. Acts as a federation broker.
- **Key components for federation:**
  - **User Pools:** User directory with sign-in (can federate to external SAML/OIDC providers). Issues JWTs.
  - **Identity Pools (Federated Identities):** Exchange tokens (Cognito, SAML, OIDC, social) for temporary AWS credentials. Supports authenticated AND unauthenticated access.
  - **Enhanced (simplified) AuthFlow:** User Pool token → Identity Pool → AWS credentials.
- **Key differentiator from direct OIDC/SAML:**
  - Cognito handles the federation complexity (token exchange, credential vending).
  - Supports unauthenticated (guest) access with limited IAM roles.
  - Supports attribute-based access control (ABAC) via token claims mapped to session tags.
- **Limits:**
  - User Pools: 50 per account (soft).
  - Identity Pools: 60 per account (soft).
  - External IdPs per User Pool: 300.

### Key Differences: Identity Center vs. Direct SAML/OIDC Federation

| Aspect | IAM Identity Center | Direct IAM Federation (SAML/OIDC) |
|--------|--------------------|------------------------------------|
| Scope | Organization-wide, multi-account | Per-account IAM role setup |
| Management | Centralized (one config) | Decentralized (configure each account) |
| User experience | Single portal, all accounts | Per-account console URLs |
| Permission model | Permission Sets (reusable) | IAM Roles per account (manual) |
| Best for | Multi-account AWS access | Single account, legacy, or non-org setups |
| CLI/SDK | `aws sso login` | `AssumeRoleWithSAML` / custom scripts |
| Effort to scale | Low (assign permission set to new account) | High (create role + trust in each account) |

---

## 2. Design Patterns & Best Practices

### Pattern 1: Enterprise Multi-Account SSO (Recommended)

```
Corporate IdP (Okta/Azure AD/ADFS)
         │
         ▼ (SAML 2.0 / SCIM)
IAM Identity Center (Home Region)
         │
         ├── Permission Set: AdminAccess → OU: Production (Security Team)
         ├── Permission Set: ReadOnlyAccess → OU: All (Auditors)
         ├── Permission Set: DeveloperAccess → OU: Development (Dev Team)
         └── Permission Set: DataScientist → Account: Analytics (Data Team)
         
Users see all assigned accounts in AWS Access Portal
```

### Pattern 2: Workload Identity (CI/CD with OIDC)

```
GitHub Actions / GitLab CI
         │
         ▼ (OIDC Token - JWT)
IAM OIDC Identity Provider
         │
         ▼ (AssumeRoleWithWebIdentity)
IAM Role (conditions: repo, branch, environment)
         │
         ▼
Deploy to AWS (no stored secrets)
```

### Pattern 3: Customer-Facing App Federation (Cognito)

```
End Users → Cognito User Pool ← External IdPs (Google, SAML corporate)
                    │
                    ▼ (JWT tokens)
              Cognito Identity Pool
                    │
                    ▼ (Temporary AWS credentials)
              S3, DynamoDB, etc. (scoped by sub/claims)
```

### Pattern 4: Legacy Per-Account SAML (Pre-Identity Center)

```
Corporate IdP → SAML Assertion → AWS STS (AssumeRoleWithSAML)
                                         │
                                         ▼
                               IAM Role in specific account
                               
(Each account needs its own role + trust + IdP entity)
```

### Best Practices

| Area | Best Practice | Why |
|------|--------------|-----|
| Identity Source | Use external IdP (Okta/Azure AD) as source of truth | Single directory, automatic deprovisioning via SCIM |
| SCIM Provisioning | Enable automatic provisioning (SCIM) in Identity Center | Users/groups sync automatically — no manual creation |
| Permission Sets | Use AWS managed policies where possible | Less maintenance, auto-updated by AWS |
| Permission Sets | Apply least privilege — avoid `AdministratorAccess` broadly | Blast radius reduction |
| Session Duration | Set appropriate session durations per permission set | Sensitive accounts: short (1h). Dev accounts: longer (8h) |
| MFA | Enforce MFA at the IdP level (not IAM) | Centralized MFA policy, consistent experience |
| CI/CD | Use OIDC federation, NOT access keys | No long-term secrets to rotate; ephemeral credentials |
| ABAC | Use session tags from SAML/OIDC attributes for dynamic policies | Scale without per-user policies; tag-based access |
| Cognito | Use User Pools for app auth + Identity Pools for AWS access | Separation of concerns; User Pools handle auth logic |
| Delegated Admin | Delegate Identity Center admin to a security account | Management account stays clean |

### When to Use / Not Use

| Scenario | Use | Don't Use |
|----------|-----|-----------|
| Multi-account human access | IAM Identity Center | Per-account SAML roles |
| Single account, no Organization | Direct SAML federation with IAM | Identity Center (requires org) |
| CI/CD pipeline needs AWS access | OIDC federation (no secrets) | IAM user access keys |
| Mobile app needs AWS resources | Cognito Identity Pools | Direct IAM user credentials |
| B2C customer sign-in | Cognito User Pools | IAM Identity Center |
| EKS pods need AWS API access | IRSA (OIDC-based) or EKS Pod Identity | Node-level instance profile |
| Temporary contractors | Identity Center + short session + group | IAM users |
| Machine-to-machine (no human) | IAM Roles (service roles) | Identity Center |

---

## 3. Security & Compliance

### Security Controls

| Control | Implementation |
|---------|---------------|
| MFA enforcement | Configure at IdP level (Okta MFA policy, Azure AD Conditional Access). Identity Center also supports built-in MFA if using built-in directory |
| Session duration | Permission set setting: 1-12 hours. Shorter for privileged access |
| IP restrictions | IdP-level conditional access policies (not natively in Identity Center) |
| Least privilege | Use permission boundaries + minimal permission sets |
| Access review | Identity Center integrates with AWS Access Analyzer for unused permissions |
| SCIM deprovisioning | When user removed from IdP group, access automatically revoked |
| Emergency access | Break-glass IAM user in management account (MFA + strict policy + monitored) |
| Credential exposure | OIDC/SAML = no long-term credentials. Eliminate IAM user access keys |

### IAM Policy Conditions for Federation

```json
{
  "Condition": {
    "StringEquals": {
      "saml:aud": "https://signin.aws.amazon.com/saml",
      "aws:RequestedRegion": "us-east-1"
    },
    "StringLike": {
      "aws:PrincipalTag/department": "engineering"
    },
    "ForAnyValue:StringLike": {
      "token.actions.githubusercontent.com:sub": "repo:myorg/myrepo:ref:refs/heads/main"
    }
  }
}
```

### ABAC (Attribute-Based Access Control) with Federation

- SAML attributes or OIDC claims can be passed as **session tags**.
- IAM Identity Center supports session tags from identity source attributes.
- Enables dynamic policies: "Users can only access resources tagged with their department."
- Example: `aws:PrincipalTag/CostCenter` = `${saml:Attribute/CostCenter}`

### Key SCPs for Federation Security

```json
{
  "Effect": "Deny",
  "Action": [
    "sso:DeletePermissionSet",
    "sso:DeleteAccountAssignment",
    "iam:CreateUser",
    "iam:CreateAccessKey"
  ],
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "aws:PrincipalOrgMasterAccountId": "${aws:PrincipalAccount}"
    }
  }
}
```

### Compliance Considerations

- **No long-term credentials:** SAML/OIDC federation eliminates IAM access keys → satisfies credential rotation requirements automatically.
- **Centralized audit:** Identity Center logs all authentication events to CloudTrail (`sso.amazonaws.com`).
- **SOC 2 / ISO 27001:** Identity Center supports compliance by centralizing access reviews and enforcing least privilege.
- **PCI DSS:** MFA requirement satisfied at IdP level + session duration controls.

---

## 4. Cost Optimization

### Pricing Model Summary

| Service | Pricing |
|---------|---------|
| IAM Identity Center | **FREE** — no charge for SSO, permission sets, or account assignments |
| IAM Federation (SAML/OIDC) | **FREE** — STS AssumeRole calls are free |
| Cognito User Pools | Free tier: 50,000 MAU (monthly active users). Then $0.0055/MAU (tiered down) |
| Cognito Identity Pools | **FREE** — no charge for credential exchange. Cost is in downstream AWS services |
| External IdP (Okta, Azure AD) | Vendor licensing (typically per-user/month) |

### Cost Optimization Strategies

1. **Use Identity Center (free) instead of third-party SSO-to-AWS solutions** — eliminates per-seat costs for AWS access specifically.
2. **Cognito User Pools:** Stay within the 50K MAU free tier by using Cognito only for active authentication (not as a general user directory).
3. **Avoid IAM users with access keys** — access keys require rotation solutions (Secrets Manager = $0.40/secret/month), while federation is free.
4. **OIDC for CI/CD** — eliminates need for Secrets Manager to store/rotate pipeline credentials.
5. **Cognito Identity Pools** — use `getCredentialsForIdentity` (enhanced flow) not `getOpenIdToken` + `AssumeRoleWithWebIdentity` (classic flow) — fewer API calls.

### Cost Traps

- Cognito User Pools with advanced security features (adaptive authentication): adds $0.050/MAU.
- Using Cognito when a simple OIDC direct integration would suffice (over-engineering).
- Maintaining BOTH Identity Center and per-account IAM federation (redundant complexity, not direct cost but operational cost).
- Ignoring IdP vendor licensing costs when evaluating total cost of ownership.

---

## 5. High Availability, Disaster Recovery & Resilience

### Service Resilience

| Service | HA Model | DR Consideration |
|---------|----------|-----------------|
| IAM Identity Center | Regional (single home region) | If home region is down, SSO is unavailable. Break-glass IAM users as backup |
| IAM (SAML/OIDC) | Global service | Highly available across all regions. STS endpoints per region |
| STS | Regional endpoints (recommended) + global endpoint | Use regional endpoints for lower latency and regional isolation |
| Cognito User Pools | Regional | No native cross-region replication. Use backup IdP or multi-region deployment |
| Cognito Identity Pools | Regional | Same as User Pools — regional scope |

### Resilience Patterns

1. **Break-glass access:** Maintain 1-2 IAM users in the management account with strong MFA + IP restrictions + CloudTrail alerting. Used when Identity Center or IdP is down.
2. **Regional STS endpoints:** Always use regional STS endpoints (`sts.us-east-1.amazonaws.com`) instead of global (`sts.amazonaws.com`) — regional endpoints survive global endpoint issues.
3. **IdP redundancy:** Enterprise IdPs (Okta, Azure AD) have their own HA/DR. Understand your IdP's SLA — it's now on the critical path for ALL AWS access.
4. **Cognito multi-region:** Deploy User Pools in multiple regions with a routing layer (Route 53/CloudFront) for customer-facing apps requiring high availability.
5. **OIDC certificate rotation:** Monitor IdP certificate expiration — expired certs = broken federation. Automate renewal alerts.

### Key Risk: Identity Center Home Region Failure

- Identity Center runs in ONE region. If that region has an outage, users cannot authenticate via SSO.
- **Mitigation:** Pre-provisioned break-glass roles + direct console access via IAM federation (bypassing Identity Center).
- **Mitigation:** Document emergency access procedures and test regularly (game days).

---

## 6. Exam-Focused Section

### 6.1 Straightforward Questions

**Q1:** A company with 30 AWS accounts uses AWS Organizations. They want to provide their 500 employees access to appropriate accounts using their corporate Active Directory credentials. What is the most operationally efficient solution?

**A:** Configure AWS IAM Identity Center with AWS Managed Microsoft AD (or AD Connector) as the identity source. Create permission sets and assign them to AD groups for specific accounts. Users access the AWS Access Portal to reach their authorized accounts.

---

**Q2:** A startup wants their GitHub Actions pipelines to deploy to AWS without storing any long-term credentials. What should they configure?

**A:** Create an IAM OIDC Identity Provider for GitHub Actions (`token.actions.githubusercontent.com`). Create an IAM role with a trust policy that allows `AssumeRoleWithWebIdentity` with conditions on the repository and branch (`sub` claim). Configure the GitHub Actions workflow to use the `aws-actions/configure-aws-credentials` action with the role ARN.

---

**Q3:** What's the difference between a Cognito User Pool and a Cognito Identity Pool?

**A:** **User Pool** = user directory + authentication (sign-up, sign-in, federation to IdPs, issues JWTs). **Identity Pool** = authorization (exchanges tokens for temporary AWS credentials). User Pools answer "who are you?" Identity Pools answer "what can you access in AWS?"

---

**Q4:** A company is migrating from per-account IAM SAML federation to IAM Identity Center. What key benefit do they gain?

**A:** Centralized, reusable permission sets that can be assigned across all accounts from one place — eliminating the need to create and maintain individual IAM roles and IdP configurations in every account.

---

**Q5:** An application running on EKS needs to access an S3 bucket with pod-level permissions (not node-level). What federation mechanism enables this?

**A:** IAM Roles for Service Accounts (IRSA) or EKS Pod Identity. IRSA uses a projected OIDC token from the EKS cluster's OIDC provider. The pod assumes an IAM role via `AssumeRoleWithWebIdentity` using the projected service account token. Each pod gets its own scoped credentials.

---

**Q6:** How does SCIM provisioning work with IAM Identity Center?

**A:** SCIM (System for Cross-domain Identity Management) automatically synchronizes users and groups from the external IdP to Identity Center. When a user is added/removed from a group in Okta/Azure AD, that change propagates to Identity Center, updating their AWS access assignments without manual intervention.

---

**Q7:** A company wants to enforce that federated users can only access resources tagged with their department. What mechanism should they use?

**A:** ABAC (Attribute-Based Access Control) using session tags. Configure the IdP to pass the department as a SAML attribute. Map this to an AWS session tag in Identity Center (or via SAML role trust policy). Write IAM policies that condition on `aws:PrincipalTag/department` matching `aws:ResourceTag/department`.

---

### 6.2 Tricky/Scenario-Based Questions

**Q1:** A company uses IAM Identity Center with Okta as the external IdP. During an Okta outage, NO employees can access AWS. The CTO demands a contingency plan. What should the architect recommend?

**A:** Implement **break-glass IAM users** in the management account — minimum 2 IAM users with MFA hardware tokens, strict IP-based policy, and CloudTrail alerting on any usage. Store credentials in a physical safe. These are ONLY used when the IdP or Identity Center is unavailable. Also consider: configure Identity Center's built-in MFA as a backup authenticator.

**Why other options are wrong:**
- "Switch identity source to built-in directory during outage": Identity Center only supports ONE identity source at a time — switching resets all user assignments.
- "Create IAM users for all employees": Defeats the purpose of federation; massive operational burden.
- "Use a backup IdP": Identity Center supports only one external IdP. You'd need per-account SAML roles pointing to a backup IdP.

---

**Q2:** A developer runs `aws sso login` and gets temporary credentials. The permission set has a 4-hour session duration. After 4 hours, their CLI session stops working. How should they handle this without interrupting long-running scripts?

**A:** Re-run `aws sso login` to get new credentials (session refresh). For long-running scripts, consider: (1) Increase the permission set session duration (up to 12h). (2) Use `credential_process` in AWS config for automatic token refresh. (3) For truly long processes, use an IAM role assumed by the workload (not user credentials).

**Why other options are wrong:**
- "Create an IAM access key": Anti-pattern — long-term credentials.
- "Use the global STS endpoint for longer sessions": STS session duration is bounded by the permission set configuration, not the endpoint.
- "Use `aws sts get-session-token`": Doesn't work with Identity Center SSO tokens.

---

**Q3:** An organization migrating to Identity Center has existing cross-account roles that automation tools assume. They want to keep automation working while migrating humans to Identity Center. What's the best approach?

**A:** Migrate human access to Identity Center (permission sets) while keeping existing IAM roles for automation/service accounts. Add conditions to the automation roles to prevent human assumption (e.g., `aws:PrincipalIsAWSService` or source IP conditions). Over time, migrate automation to OIDC federation or service roles. This is a **phased migration** — humans first, machines later.

---

**Q4:** A company configures OIDC federation for GitHub Actions. A malicious actor forks their repo and runs a pipeline. Can the fork assume their IAM role?

**A:** **No** — if the trust policy properly restricts the `sub` claim. The `sub` claim from GitHub includes the repository owner and name (e.g., `repo:myorg/myrepo:ref:refs/heads/main`). A fork has a different owner (`repo:attacker/myrepo:...`), which won't match the condition. **Exam trap:** If the trust policy only checks `aud` but NOT `sub`, then YES, any repo using that GitHub OIDC provider could assume the role.

---

**Q5:** A security audit reveals that an IAM role used for SAML federation has a trust policy allowing ANY SAML provider in the account to assume it. What's the risk and fix?

**A:** **Risk:** Any SAML provider entity configured in the account (even a rogue one created by a compromised admin) could issue assertions for that role. **Fix:** Restrict the trust policy's `Principal.Federated` to the SPECIFIC SAML provider ARN (`arn:aws:iam::123456789012:saml-provider/CorporateIdP`) AND add conditions on `saml:iss` (issuer) and `saml:aud`.

---

**Q6:** A company uses Cognito User Pools with federation to a corporate SAML IdP. Users report that after their corporate password is reset, they can still access the application using their Cognito session. Why?

**A:** Cognito User Pool tokens (JWTs) are stateless and valid until expiration. A password reset at the IdP doesn't invalidate existing Cognito tokens. **Fix:** Use Cognito's Global Sign Out (revokes refresh tokens) or reduce access token expiration. For enterprise apps requiring immediate revocation: validate the access token against the User Pool on each request or use shorter token lifetimes.

**Why other options are wrong:**
- "Use ID tokens instead of access tokens": Both are JWTs with expiration — same problem.
- "Reduce User Pool session duration": This only affects the hosted UI session, not issued tokens.

---

**Q7:** An architect is choosing between two approaches for a B2B application where enterprise customers bring their own IdP: (A) Direct SAML federation with IAM roles per customer, or (B) Cognito User Pool with multiple SAML providers. Which is better and why?

**A:** **(B) Cognito User Pool** — it supports up to 300 external IdPs per pool, handles the SAML assertion processing, normalizes user attributes, issues standardized JWTs, and provides attribute mapping. Direct IAM SAML federation would require creating IAM roles per customer (doesn't scale) and is designed for AWS Console/API access, not application-level authentication.

---

### 6.3 Common Exam Traps & Pitfalls

| Trap | Reality |
|------|---------|
| "Use IAM users for employees" | ALWAYS prefer federation (Identity Center/SAML/OIDC) — IAM users are for legacy or break-glass only |
| "Identity Center works without Organizations" | Identity Center REQUIRES AWS Organizations — can't use standalone |
| "SAML and OIDC do the same thing" | SAML = enterprise/browser-based SSO (XML assertions). OIDC = modern/API/mobile (JWT tokens). Different protocols, different use cases |
| "Identity Center supports multiple IdPs" | Only ONE identity source at a time (built-in, AD, or ONE external IdP). To change, you must delete and recreate assignments |
| "Cognito User Pools give AWS credentials" | User Pools issue JWTs for application auth. IDENTITY POOLS issue temporary AWS credentials |
| "AssumeRoleWithSAML can be called by the application" | It's called by the USER'S BROWSER (POST to STS). The IdP doesn't call AWS directly |
| "OIDC federation requires an IdP running in your account" | No — the IdP is EXTERNAL (GitHub, Google, etc.). You just register it in IAM |
| "Session duration in trust policy overrides permission set" | Permission set session duration takes precedence for Identity Center. Direct SAML: max of role & assertion values |
| "SCIM is required for Identity Center" | SCIM is OPTIONAL (auto-sync). You can manually create users in Identity Center directory without SCIM |
| "Identity Center permission sets ARE IAM roles" | Permission sets CREATE IAM roles in each assigned account, but they're managed centrally — you don't create them manually |

---

## 7. Cheat Sheet

### One-Page Summary

| Service/Mechanism | Purpose | Key Differentiator |
|-------------------|---------|-------------------|
| **IAM Identity Center** | Multi-account SSO for humans | Centralized, org-wide, permission sets = reusable |
| **SAML 2.0 Federation** | Enterprise SSO (browser-based) | XML assertions, corporate IdPs, console access |
| **OIDC Federation** | Modern/API/mobile/CI-CD auth | JWT tokens, web identity, no stored secrets |
| **Cognito User Pools** | App-level user directory + auth | B2C/B2B, supports multiple IdPs, issues JWTs |
| **Cognito Identity Pools** | Exchange tokens for AWS credentials | Credential vending machine, supports guest access |
| **STS AssumeRoleWithSAML** | API call for SAML → AWS creds | Per-account, legacy approach |
| **STS AssumeRoleWithWebIdentity** | API call for OIDC → AWS creds | Used by IRSA, GitHub Actions, mobile apps |

### Decision Flowchart

```
Question: Human multi-account access? ──► IAM Identity Center
Question: CI/CD needs AWS access? ──► OIDC Federation (no secrets)
Question: Mobile app needs AWS resources? ──► Cognito Identity Pools
Question: Customer-facing app sign-in? ──► Cognito User Pools
Question: Single account + corporate IdP? ──► Direct SAML federation
Question: EKS pod needs AWS API? ──► IRSA or Pod Identity (OIDC)
Question: Temporary access, no stored creds? ──► Any federation (SAML/OIDC)
Question: Machine-to-machine (no human)? ──► IAM Role (not federation)
```

### Key Numbers to Remember

| Item | Value |
|------|-------|
| Identity Center: identity sources | 1 at a time |
| Permission sets per instance | 2,000 |
| Permission set session duration | 1-12 hours |
| SAML assertion max size | 100KB |
| SAML providers per account | 128 |
| OIDC providers per account | 128 |
| Cognito User Pool free tier | 50,000 MAU |
| STS session duration (SAML/OIDC) | 15 min to 12 hours |
| Identity Center requires | AWS Organizations |
| Cognito external IdPs per pool | 300 |

---

## 8. Additional Tricky Scenario-Based Questions

**Q8:** A company has 5,000 employees across 10 departments. They want each department to only access resources tagged with their department name, across 50 AWS accounts. What's the most scalable solution?

**A:** IAM Identity Center + ABAC. Configure the IdP to pass the department attribute via SCIM. Enable session tags in Identity Center (map department attribute → `aws:PrincipalTag/department`). Create ONE permission set with a policy that conditions on `aws:PrincipalTag/department` == `aws:ResourceTag/department`. Assign this single permission set to all users. The policy dynamically scopes access per user without per-department permission sets.

---

**Q9:** During a security review, an auditor finds that `AssumeRoleWithSAML` calls are appearing in CloudTrail from an unrecognized SAML provider ARN. What are the immediate and long-term remediation steps?

**A:** **Immediate:** Disable/delete the unrecognized SAML provider entity in IAM. Revoke any active sessions assumed via that provider (`aws sts revoke-security-credentials` isn't available — instead, add an explicit deny policy on the roles that were assumed). **Long-term:** SCP preventing `iam:CreateSAMLProvider` except from authorized accounts. Config rule detecting unauthorized SAML providers. Review trust policies on all roles.

---

**Q10:** A company migrating from ADFS-based SAML federation to Identity Center asks: "Can we run both in parallel during migration?" What's the answer?

**A:** **Yes.** Identity Center and per-account SAML federation are independent mechanisms. During migration: (1) Set up Identity Center with the same AD as source. (2) Create permission sets mirroring existing SAML roles. (3) Have users test via Identity Center. (4) Once validated, remove per-account SAML roles and IAM IdP entities. They run in parallel without conflict.

---

**Q11:** A company uses GitHub Actions OIDC to deploy to AWS. They have 200 repos, each needing a different level of AWS access. Creating 200 IAM roles is unsustainable. What's a better pattern?

**A:** Use a smaller set of IAM roles scoped by **repository pattern** or **environment** in the trust policy conditions. Example: one role for "deploy" environment (`sub: repo:org/*:environment:production`), one for "staging" (`sub: repo:org/*:environment:staging`). Use IAM policy conditions on the `sub` claim to further restrict (e.g., `StringLike` on repo name). Or use session tags from OIDC claims for ABAC-style access.

---

**Q12:** A security engineer notices that after removing a user from all groups in Okta, that user can still access AWS for up to 1 hour via Identity Center. Why?

**A:** The user's existing SSO session hasn't expired yet. SCIM deprovisioning removes future access, but currently active sessions persist until their session duration expires. **Fix:** Reduce session duration on sensitive permission sets. For immediate revocation, use IAM policies with an `aws:TokenIssueTime` condition or delete the user's assignments in Identity Center directly.

---

**Q13:** An architect needs to provide 10,000 external contractors temporary AWS Console access. Contractors authenticate via their own company's IdP. The solution must NOT require any configuration per contractor company. What works?

**A:** **Cognito User Pool with SAML federation** won't work here (requires per-IdP configuration). The best approach is to use **Cognito User Pool with OIDC federation** to a broker IdP that supports multi-tenant federation (like Auth0 or Azure AD B2C). Alternatively, if contractors can use social login (Google Workspace accounts tied to their company), configure OIDC with Google as the provider. The key constraint "no per-company config" eliminates most enterprise SAML approaches.

---

**Q14:** What happens when you change the identity source in IAM Identity Center from the built-in directory to an external IdP?

**A:** All existing users and groups in the Identity Center directory are deleted. All permission set assignments associated with those users/groups are removed. You must reconfigure all assignments using the new identity source's users/groups. **This is destructive and cannot be undone.** Always plan migration carefully — this is a common exam scenario testing understanding of Identity Center limitations.

---

**Q15:** A company uses EKS with IRSA (IAM Roles for Service Accounts). A developer reports that pods in a new node group can't assume their IAM role. The OIDC provider is configured correctly. What's likely wrong?

**A:** The new node group's nodes don't have the required IAM permission to call `sts:AssumeRoleWithWebIdentity`? **No** — IRSA doesn't need node permissions. More likely: the projected service account token isn't being mounted (check pod spec for `serviceAccountName`) or the IAM role's trust policy condition on `sub` doesn't match the service account namespace/name (`system:serviceaccount:namespace:sa-name`). Also check: EKS cluster's OIDC issuer URL must match the IAM OIDC provider's URL exactly (including trailing path).

---

**Q16:** A regulated financial institution needs to ensure that federated users cannot escalate their own permissions. Their concern: a developer with Identity Center access could create an IAM admin user in their assigned account. How to prevent this?

**A:** Use **permission boundaries** in the permission set. Attach a permission boundary to the permission set that limits the maximum effective permissions. Even if the developer has `iam:CreateUser` + `iam:AttachUserPolicy`, the boundary prevents them from creating users with more permissions than they have. Additionally: SCP denying `iam:CreateUser` and `iam:CreateAccessKey` in member accounts.

---

**Q17:** A company discovers that their SAML-federated IAM role allows session durations up to 12 hours, but their security policy requires sessions to be max 1 hour. They've set `DurationSeconds` to 3600 in the role. Users still get 12-hour sessions. Why?

**A:** The SAML assertion from the IdP includes a `SessionDuration` attribute that can override the `DurationSeconds` parameter. If the IdP sends `<Attribute Name="https://aws.amazon.com/SAML/Attributes/SessionDuration"><AttributeValue>43200</AttributeValue></Attribute>`, the session will be 12 hours. **Fix:** Configure the IdP to send 3600 (or remove the attribute entirely) AND set the IAM role's max session duration to 3600. The effective session is the MINIMUM of: role's max, SAML attribute value, and API parameter.

---

## 9. Service Comparison Table

| Dimension | IAM Identity Center | Direct SAML (IAM) | OIDC (IAM) | Cognito User Pools | Cognito Identity Pools |
|-----------|--------------------|--------------------|------------|-------------------|----------------------|
| **Primary Use Case** | Multi-account human SSO | Single-account enterprise SSO | Workload/app identity | App user authentication | App AWS credential vending |
| **Protocol** | SAML 2.0 (backend) | SAML 2.0 | OpenID Connect | OIDC + SAML + Social | Token exchange |
| **Credential Output** | Temp creds (console + CLI) | Temp creds (console + CLI) | Temp creds (API) | JWT (ID + Access + Refresh) | Temp AWS credentials |
| **Scope** | AWS Organization (all accounts) | Per account | Per account | Per application | Per application |
| **Scalability** | 2,000 permission sets | 1 role per use case per account | 1 role per use case per account | 50K MAU free, millions supported | Millions of identities |
| **Cost** | Free | Free | Free | Per MAU (free tier: 50K) | Free |
| **Best For** | Enterprise AWS access | Legacy/no-org setups | CI/CD, mobile, EKS pods | B2C/B2B apps | App-to-AWS access |
| **Exam Tip** | "centralized", "multiple accounts", "SSO portal" | "single account", "existing SAML setup" | "GitHub Actions", "no secrets", "IRSA", "mobile" | "user sign-up", "app authentication" | "AWS credentials for app users" |

---

## 10. When to Pick Service A over Service B

| Exam Keyword/Scenario | Pick This | Not That |
|----------------------|-----------|----------|
| "500 employees, 30 accounts, single sign-on" | IAM Identity Center | Per-account SAML |
| "CI/CD, no stored credentials" | OIDC federation | IAM access keys |
| "Mobile app needs S3 access" | Cognito (User Pool + Identity Pool) | IAM users |
| "EKS pod needs DynamoDB access" | IRSA (OIDC) or Pod Identity | Node instance profile |
| "Customer-facing app sign-in + social login" | Cognito User Pools | IAM Identity Center |
| "Requires AWS Organizations" | IAM Identity Center | Direct SAML/OIDC |
| "One account, no org, corporate IdP" | Direct SAML federation | Identity Center |
| "Centralized permission management" | Identity Center (permission sets) | Per-account IAM roles |
| "Temporary contractors, short access" | Identity Center + short session | IAM users with expiry |
| "Cross-account access for services" | IAM Roles (AssumeRole) | Identity Center (human only) |
| "Unauthenticated guest access to AWS" | Cognito Identity Pools | Nothing else supports this |
| "Active Directory integration for AWS" | Identity Center + AD Connector/Managed AD | Creating IAM users mirroring AD |
| "Attribute-based dynamic permissions" | ABAC with session tags (any federation) | Static per-user policies |
| "Immediate access revocation" | Short session duration + IdP disable | Long-lived tokens |
| "Break-glass emergency access" | Dedicated IAM users (MFA + monitored) | Relying solely on federation |

---

## 11. "Gotcha" Differences

| Gotcha | Details |
|--------|---------|
| Identity Center vs. IAM roles | Identity Center CREATES IAM roles behind the scenes (one per permission set per account). But you manage them via permission sets, not IAM directly. |
| Identity Center identity source change | Changing from built-in to external IdP (or vice versa) DELETES all users/groups and assignments. Irreversible. Plan carefully. |
| SAML vs. OIDC token format | SAML = XML assertion (bulky, browser-based POST). OIDC = JWT (compact, API-friendly). Not interchangeable. |
| Cognito User Pool token vs. Identity Pool credential | User Pool JWT = proves identity (for your app). Identity Pool credential = temporary IAM key/secret/token (for AWS APIs). Different purposes. |
| `AssumeRoleWithSAML` vs. `AssumeRoleWithWebIdentity` | SAML = enterprise IdP (ADFS, Okta). WebIdentity = OIDC providers (Google, GitHub, Cognito). Different API calls, different trust policy principals. |
| Identity Center session vs. IdP session | Two separate sessions. IdP session can be valid (SSO cookie) even after Identity Center permission set session expires. User re-authenticates to AWS, not to IdP. |
| SCIM provisioning vs. JIT (Just-In-Time) provisioning | SCIM = async background sync of users/groups. JIT = user created on first login via SAML attributes. Identity Center supports SCIM. Direct SAML uses JIT. |
| Permission set boundary vs. IAM permission boundary | Permission set boundary (Identity Center) = limits what the permission set can grant. IAM permission boundary = limits what an IAM entity can do. Both are "maximum permissions" but configured differently. |
| OIDC audience (`aud`) vs. subject (`sub`) | `aud` = who the token is intended for (client ID). `sub` = who the token represents (user/repo). Trust policies should check BOTH for security. |
| Identity Center + Control Tower | Control Tower uses Identity Center for account factory access. They're integrated but separate — disabling one doesn't disable the other. |
| Regional STS vs. Global STS | Global endpoint (`sts.amazonaws.com`) routes to us-east-1. If us-east-1 is down, global endpoint fails. ALWAYS use regional endpoints. |
| Cognito "sign-in" vs. "federation" | A Cognito User Pool user can be native (username/password in pool) OR federated (redirects to external IdP). The app doesn't need to know which — Cognito handles both. |

---

## 12. One-Page Decision Tree

```
┌─ "The exam question asks about..."
│
├── MULTI-ACCOUNT HUMAN SSO / "CENTRALIZED ACCESS"
│   └── IAM Identity Center
│       ├── Identity source? → External IdP (Okta/Azure AD) via SAML + SCIM
│       ├── Permission model? → Permission Sets (reusable across accounts)
│       ├── CLI access? → `aws sso login`
│       ├── Emergency access? → Break-glass IAM users
│       ├── Dynamic permissions? → ABAC with session tags
│       └── Requires? → AWS Organizations
│
├── WORKLOAD / CI-CD / "NO LONG-TERM CREDENTIALS"
│   └── OIDC Federation
│       ├── GitHub Actions? → IAM OIDC provider + role (sub claim condition)
│       ├── GitLab CI? → IAM OIDC provider + role
│       ├── EKS pods? → IRSA or Pod Identity
│       ├── Security? → Restrict `sub`, `aud`, `repo`, `branch` in conditions
│       └── Benefit? → Zero stored secrets
│
├── SINGLE ACCOUNT / LEGACY / "CORPORATE IdP CONSOLE ACCESS"
│   └── Direct SAML Federation
│       ├── How? → IAM SAML provider + IAM role with trust policy
│       ├── Initiated by? → IdP (redirect) or SP (AWS sign-in URL)
│       ├── Session? → SAML `SessionDuration` attribute or API parameter
│       └── Scaling issue? → Need role per account (use Identity Center instead)
│
├── APPLICATION AUTH / "USER SIGN-IN" / "B2C or B2B"
│   └── Cognito User Pools
│       ├── Social login? → Google, Facebook, Apple federation
│       ├── Enterprise customers? → SAML/OIDC per-IdP federation
│       ├── Output? → JWTs (ID token, access token, refresh token)
│       ├── MFA? → Built-in (SMS, TOTP, email)
│       └── Customization? → Hosted UI, Lambda triggers, custom domains
│
├── APP NEEDS AWS RESOURCES / "CREDENTIALS FOR APP USERS"
│   └── Cognito Identity Pools
│       ├── Input? → User Pool JWT, SAML, OIDC, social tokens
│       ├── Output? → Temporary AWS credentials (key + secret + token)
│       ├── Guest access? → Unauthenticated role supported
│       └── Scope? → IAM role per authenticated/unauthenticated
│
└── ACCESS CONTROL MODEL
    ├── Static (by group/role)? → Permission Sets or IAM Roles
    ├── Dynamic (by attribute)? → ABAC + Session Tags
    ├── Per-resource? → Resource policies (bucket policy, KMS key policy)
    └── Maximum permissions? → Permission Boundaries or SCPs
```

---

Generated for AWS SAP-C02 exam preparation. The exam heavily tests federation scenarios — understand when to use Identity Center vs. direct federation, and know the boundaries of Cognito (app auth vs. AWS credentials).
