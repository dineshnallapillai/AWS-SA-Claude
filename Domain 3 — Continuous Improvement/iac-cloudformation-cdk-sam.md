# Infrastructure as Code: CloudFormation, CDK, SAM

## 1. Core Concepts & Theory

### AWS CloudFormation
- **Declarative IaC service** — define infrastructure in JSON/YAML templates, CloudFormation provisions and manages resources
- **Stack**: A collection of AWS resources managed as a single unit (create/update/delete together)
- **Template**: JSON/YAML file describing resources, parameters, outputs, mappings, conditions
- **Change Set**: Preview of changes before executing a stack update (safe update workflow)
- **Stack Set**: Deploy stacks across multiple accounts and regions from a single template
- **Drift Detection**: Identifies resources that have been modified outside CloudFormation
- **Nested Stacks**: Stacks within stacks (reusable components via `AWS::CloudFormation::Stack`)
- **Cross-Stack References**: Export/Import values between stacks (via Outputs + `Fn::ImportValue`)

**Template Anatomy:**
```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: ...
Metadata: ...
Parameters: ...       # Input values (runtime)
Mappings: ...         # Static key-value lookups
Conditions: ...       # Conditional resource creation
Transform: ...        # Macros (e.g., AWS::Serverless-2016-10-31)
Resources: ...        # REQUIRED — AWS resources to create
Outputs: ...          # Return values (can be exported)
```

**Key Limits:**
| Limit | Value |
|-------|-------|
| Resources per template | 500 |
| Parameters per template | 200 |
| Outputs per template | 200 |
| Mappings per template | 200 |
| Template size (S3) | 1 MB |
| Template size (direct) | 51,200 bytes |
| Stacks per account | 2,000 (soft) |
| Stack Sets per admin account | 100 |
| Stack instances per Stack Set | 2,000 |
| Nested stack depth | 5 levels |
| Exports per account per region | 5,000 |

**Intrinsic Functions (must-know):**
- `Fn::Ref` / `!Ref` — Reference parameter or resource logical ID
- `Fn::GetAtt` — Get attribute of a resource (e.g., ARN, DNS name)
- `Fn::Sub` — String substitution
- `Fn::Join` — Concatenate strings
- `Fn::Select` — Select from list
- `Fn::Split` — Split string into list
- `Fn::ImportValue` — Import exported value from another stack
- `Fn::FindInMap` — Lookup from Mappings
- `Fn::If` / `Fn::Equals` — Conditional logic
- `Fn::GetAZs` — Get list of AZs

**Resource Attributes:**
- `DependsOn` — Explicit dependency ordering
- `DeletionPolicy` — Retain, Snapshot, or Delete on stack deletion
- `UpdateReplacePolicy` — Retain or Snapshot when resource is replaced during update
- `CreationPolicy` — Wait for signals (e.g., cfn-signal from EC2)
- `UpdatePolicy` — How Auto Scaling groups handle updates (rolling, replacing)

### AWS CDK (Cloud Development Kit)
- **Imperative IaC framework** — define infrastructure using programming languages (TypeScript, Python, Java, C#, Go)
- **Synthesizes** to CloudFormation templates (CDK → CloudFormation → AWS)
- **Construct**: Basic building block (like a class)
  - **L1 (Cfn)**: 1:1 mapping to CloudFormation resources (e.g., `CfnBucket`)
  - **L2 (Curated)**: Higher-level, opinionated defaults (e.g., `Bucket` with sensible defaults)
  - **L3 (Patterns)**: Multi-resource patterns (e.g., `ApplicationLoadBalancedFargateService`)
- **App**: Root of the construct tree
- **Stack**: Unit of deployment (maps to a CloudFormation stack)
- **Stage**: Group of stacks deployed together (for pipeline stages)
- **cdk synth**: Generate CloudFormation template
- **cdk deploy**: Deploy stack (synth + create/update CloudFormation stack)
- **cdk diff**: Show pending changes
- **cdk bootstrap**: Set up CDK toolkit stack (S3 bucket + IAM roles for deployment)
- **CDK Pipelines**: L3 construct for self-mutating CI/CD pipelines

**Key Facts:**
- CDK uses **assets** (local files/Docker images) — uploaded to S3/ECR during deployment
- **Context values** (`cdk.json`) — cached lookups (AZs, AMIs) for deterministic synthesis
- **Aspects**: Apply operations across all constructs (e.g., add tags, validate)
- **Escape hatches**: Access L1 construct from L2 to override specific properties
- Bootstrap stack: `CDKToolkit` (per account/region) — S3 bucket, ECR repo, IAM roles

### AWS SAM (Serverless Application Model)
- **CloudFormation extension** specifically for serverless applications
- **Transform**: `AWS::Serverless-2016-10-31` (macro that expands SAM → CloudFormation)
- **Simplified syntax** for Lambda, API Gateway, DynamoDB, Step Functions, EventBridge
- **SAM CLI**: Local testing, build, package, deploy
  - `sam init` — scaffold new project
  - `sam build` — compile and prepare deployment artifacts
  - `sam local invoke` — test Lambda locally (Docker)
  - `sam local start-api` — local API Gateway emulator
  - `sam deploy` — package and deploy (wraps `aws cloudformation deploy`)
  - `sam sync` — fast incremental deployments (skip CloudFormation for code-only changes)

**SAM Resource Types:**
| SAM Resource | Expands To |
|-------------|------------|
| `AWS::Serverless::Function` | Lambda + IAM Role + optional API Gateway/EventBridge/SQS trigger |
| `AWS::Serverless::Api` | API Gateway REST API + Stage + Deployment |
| `AWS::Serverless::HttpApi` | API Gateway HTTP API |
| `AWS::Serverless::SimpleTable` | DynamoDB table (simple key schema) |
| `AWS::Serverless::LayerVersion` | Lambda Layer |
| `AWS::Serverless::Application` | Nested serverless app (from SAR) |
| `AWS::Serverless::StateMachine` | Step Functions state machine |

**SAM Policy Templates** (simplified IAM):
- `DynamoDBCrudPolicy` — full CRUD on a table
- `S3ReadPolicy` — read from S3 bucket
- `SQSPollerPolicy` — read from SQS queue
- `KMSDecryptPolicy` — decrypt with KMS key

---

## 2. Design Patterns & Best Practices

### When to Use Which

| Criteria | CloudFormation | CDK | SAM |
|----------|---------------|-----|-----|
| Team expertise | Ops/YAML-comfortable | Developers (TypeScript/Python) | Serverless developers |
| Complexity | Any complexity | Complex, multi-stack apps with logic | Serverless-focused apps |
| Abstraction level | Low (declarative) | High (imperative + patterns) | Medium (serverless-specific shortcuts) |
| Testing | Limited (cfn-lint, TaskCat) | Full unit/integration testing in code | SAM CLI local testing |
| Reuse | Nested stacks, modules | Constructs (npm/PyPI packages) | SAR (Serverless App Repository) |
| Learning curve | Moderate | Steeper (but powerful) | Low (for serverless) |

### Anti-Patterns

**CloudFormation:**
- Don't put ALL resources in one giant template (>500 resources) — use nested stacks or cross-stack references
- Don't hardcode values — use Parameters, Mappings, SSM dynamic references
- Don't ignore Change Sets for production — always preview before updating

**CDK:**
- Don't use CDK for simple, static infrastructure (overhead of language setup for 5 resources)
- Don't mix L1 and L2 arbitrarily — prefer L2, use L1 only when L2 doesn't expose what you need (escape hatch)
- Don't forget to commit `cdk.json` context — otherwise synth becomes non-deterministic

**SAM:**
- Don't use SAM for non-serverless workloads (EC2, RDS, VPCs) — use CloudFormation/CDK
- Don't use SAM when you need complex conditional logic — SAM templates are still declarative YAML

### Architectural Patterns

**Multi-Account IaC:**
```
Management Account (StackSets admin) → Organization StackSets → Member Accounts
```
- **Service-managed StackSets**: Auto-deploy to new accounts via AWS Organizations
- **Self-managed StackSets**: Explicit account list + IAM roles

**Landing Zone Pattern:**
- Control Tower uses CloudFormation StackSets under the hood
- Baseline stacks: VPC, IAM roles, logging, security config
- Customizations: Custom StackSets for org-specific resources

**Multi-Region with CloudFormation:**
- StackSets with multiple regions specified
- Region-specific parameters via `ParameterOverrides`
- Custom resources or macros for region-aware logic

**CI/CD for IaC:**
```
CodeCommit → CodePipeline → CodeBuild (cdk synth / sam build) → CloudFormation Deploy (Change Set → Execute)
```

**CDK Pipelines (Self-Mutating):**
- Pipeline defined in CDK
- Pipeline updates ITSELF on every commit (adds/removes stages automatically)
- Stages map to environments (Dev, Staging, Prod)

### Well-Architected Alignment

| Pillar | Practice |
|--------|----------|
| Reliability | Use CloudFormation for consistent provisioning; rollback on failure; drift detection |
| Security | SSM/Secrets Manager dynamic references; SCPs on CloudFormation actions; IAM boundary on stack roles |
| Cost | Detect unused resources via drift; StackSets ensure consistent tagging for cost allocation |
| Performance | Parallel resource creation (CloudFormation handles dependency graph); CDK assets for optimized Lambda bundles |
| Ops Excellence | Change Sets for safe updates; Stack policies to prevent accidental deletes; CloudFormation Registry for custom types |

### Integration Points
- **Service Catalog** — CloudFormation templates as products (governed self-service)
- **AWS Config** — Detect drift, enforce compliance on stack resources
- **Systems Manager** — Dynamic references (`{{resolve:ssm:name}}`, `{{resolve:secretsmanager:name}}`)
- **CodePipeline** — CloudFormation deploy action (CREATE_UPDATE, CHANGE_SET_CREATE, CHANGE_SET_EXECUTE)
- **Organizations** — StackSets with service-managed permissions
- **Control Tower** — Account Factory uses CloudFormation
- **CloudFormation Registry** — Custom resource types, modules, hooks

---

## 3. Security & Compliance

### IAM & Access Control

**CloudFormation Service Role:**
- If specified, CloudFormation assumes THIS role to create/modify/delete resources
- If not specified, CloudFormation uses the caller's permissions
- **Best practice**: Always use a stack role — separates "who can update the stack" from "what the stack can do"
- Developers get `cloudformation:*` but NOT the resource permissions — the stack role has resource permissions

**Stack Policy:**
- JSON document that protects stack resources from unintended updates
- Applies DURING stack updates only (not creation or deletion)
- Default: all resources can be updated
- Once set, ALL resources are protected (must explicitly Allow updates)
- Can be temporarily overridden during update with `--stack-policy-during-update-body`

```json
{
  "Statement": [{
    "Effect": "Deny",
    "Action": "Update:Replace",
    "Principal": "*",
    "Resource": "LogicalResourceId/ProductionDatabase"
  }]
}
```

**CDK Bootstrap Roles:**
- `CloudFormationExecutionRole` — used by CloudFormation to create resources
- `DeploymentActionRole` — used to initiate deployments
- `FilePublishingRole` — used to upload assets to S3
- `ImagePublishingRole` — used to push images to ECR
- `LookupRole` — used during synthesis to look up context

**SCPs for CloudFormation:**
- Prevent `cloudformation:DeleteStack` on production stacks
- Require specific stack roles (condition on `iam:PassedToService`)
- Restrict regions where stacks can be created

### Encryption & Secrets

**Dynamic References (resolved at deploy time, never stored in template):**
- `{{resolve:ssm:parameter-name:version}}` — SSM Parameter Store (plaintext)
- `{{resolve:ssm-secure:parameter-name:version}}` — SSM SecureString (only for supported resources)
- `{{resolve:secretsmanager:secret-id:json-key:version-stage:version-id}}` — Secrets Manager

**Restrictions on dynamic references:**
- `ssm-secure` only works with specific resource properties (e.g., RDS MasterUserPassword, not all resources)
- Cannot use dynamic references in `Conditions`, `Metadata`, or `Outputs`
- Max 60 dynamic references per template

### Logging & Auditing
- **CloudTrail**: All CloudFormation API calls (CreateStack, UpdateStack, DeleteStack)
- **Stack Events**: Detailed timeline of resource creation/update/deletion within a stack
- **Drift Detection**: Scheduled or on-demand detection of out-of-band changes
- **CloudFormation Registry** hooks: Run validation logic before/after resource operations

### Compliance Patterns
- **Preventive**: Stack policies + SCPs + CloudFormation hooks (validate before create/update)
- **Detective**: AWS Config rules + drift detection
- **Corrective**: Config auto-remediation + StackSets for re-baseline
- **Service Catalog**: Constrained CloudFormation templates for governed provisioning
- **CloudFormation Guard**: Policy-as-code rules for templates (e.g., "all S3 buckets must have encryption enabled")

---

## 4. Cost Optimization

### Pricing

**CloudFormation:**
- **Free** for AWS resources (you pay only for the resources created)
- **Third-party resources** (from CloudFormation Registry): $0.0009 per handler operation (create/update/delete/read/list)
- **Handler operation duration**: $0.00008 per second beyond 30 seconds

**CDK:**
- **Free** (open-source tool) — cost is CloudFormation + resources
- CDK bootstrap stack costs: S3 bucket (minimal storage), ECR repo (if using Docker assets)

**SAM:**
- **Free** — cost is CloudFormation + serverless resources

### Cost-Saving Strategies
- **Delete development stacks** when not in use (re-create from template)
- **Use `DeletionPolicy: Snapshot`** on databases — save money by deleting instances but keeping snapshots
- **StackSets**: Enforce cost tags across all accounts
- **CDK Aspects**: Automatically add cost allocation tags to all resources
- **Avoid circular dependencies** that require custom resources (Lambda-backed) — these add Lambda cost
- **SAM**: Use `AutoPublishAlias` + `DeploymentPreference` instead of managing aliases manually

### Cost Traps
- **Forgetting to delete test stacks** — resources keep running
- **CloudFormation rollback** on failed update may leave additional resources (e.g., new security groups created before failure)
- **Nested stacks** with `DeletionPolicy: Retain` — orphaned resources after parent deletion
- **CDK bootstrap ECR** images accumulate — configure lifecycle policy
- **Third-party resource types** in Registry — per-operation charges can add up

---

## 5. High Availability, Disaster Recovery & Resilience

### Stack Resilience

**Stack Update Strategies:**
| Strategy | Risk | Use When |
|----------|------|----------|
| Direct update | Higher (immediate) | Dev/test environments |
| Change Set | Lower (preview first) | Production (always) |
| Rolling update (ASG) | Medium | Gradual instance replacement |
| Stack replacement (Blue/Green) | Lowest | Critical stacks needing zero-downtime |

**Rollback Behavior:**
- **Create failure**: All resources rolled back (deleted) by default
- **Update failure**: Reverts to previous successful state
- **Disable rollback**: Can be configured (useful for debugging, NOT for production)
- **Continue rollback**: Resume a failed rollback by skipping problematic resources
- **Rollback triggers**: CloudWatch alarms that trigger automatic rollback during stack operations

**DeletionPolicy:**
- `Retain` — resource NOT deleted when stack is deleted (useful for databases, S3 buckets)
- `Snapshot` — take snapshot before deleting (RDS, EBS, ElastiCache Redis, Redshift, Neptune)
- `Delete` — default behavior (resource deleted with stack)

### DR Patterns

**Multi-Region IaC:**
- Store templates in version-controlled repository (CodeCommit with cross-region replication)
- Use StackSets for multi-region deployment
- **DR recovery**: Deploy template in DR region from same source
- **Parameters**: Region-specific parameters via SSM Parameter Store (each region has its own)

**Stack Import:**
- Import existing resources into a stack (`resource-import` operation)
- Useful for DR: rebuild stack around existing resources that survived

**CDK for DR:**
- Same CDK app deploys to multiple regions (parameterized by environment)
- CDK Pipelines can deploy to DR region as a pipeline stage

### StackSets for HA
- Deploy same infrastructure across multiple regions simultaneously
- Failure tolerance: Configure how many accounts/regions can fail before StackSet operation fails
- Max concurrent: Control parallelism of deployments
- Auto-deployment: Automatically deploy to new accounts added to OU

---

## 6. Exam-Focused Section

### Straightforward Questions (7 examples)

**Q1:** A company wants to ensure their RDS database is not accidentally deleted when the CloudFormation stack is deleted. What should they do?
**A:** Set `DeletionPolicy: Snapshot` (takes a final snapshot) or `DeletionPolicy: Retain` (keeps the database running) on the RDS resource.

**Q2:** What is the maximum number of resources in a single CloudFormation template?
**A:** 500 resources per template. Use nested stacks to exceed this limit.

**Q3:** A team uses CDK and wants to deploy a load-balanced Fargate service with minimal code. Which construct level should they use?
**A:** L3 pattern construct — `ApplicationLoadBalancedFargateService` creates ALB, ECS Service, task definition, and Fargate configuration in one construct.

**Q4:** How can you reference an SSM Parameter Store value in a CloudFormation template without hardcoding it?
**A:** Use dynamic references: `{{resolve:ssm:parameter-name:version}}`. The value is resolved at deploy time.

**Q5:** A company needs to deploy the same baseline VPC configuration to all accounts in their AWS Organization automatically. What should they use?
**A:** CloudFormation StackSets with **service-managed permissions** and **auto-deployment** enabled for the Organization/OU.

**Q6:** What command runs a Lambda function locally for testing before deploying a SAM application?
**A:** `sam local invoke FunctionName` — uses Docker to emulate the Lambda runtime locally.

**Q7:** How do you preview changes before updating a production CloudFormation stack?
**A:** Create a **Change Set** (`aws cloudformation create-change-set`) — it shows which resources will be added, modified, or replaced without executing the changes.

---

### Tricky / Scenario-Based Questions (7 examples)

**Q1:** A company updates a CloudFormation stack and the update fails midway. Some resources were updated successfully before the failure. What happens?

**A:** CloudFormation automatically **rolls back ALL changes** to the previous known good state. Resources that were successfully updated are reverted. The stack returns to its previous configuration. If rollback also fails, the stack enters `UPDATE_ROLLBACK_FAILED` state and you must use `continue-update-rollback` (possibly skipping problematic resources).

**Why wrong answers are wrong:**
- "Only the failed resource is reverted" — CloudFormation rolls back the ENTIRE update, not individual resources
- "The stack is deleted" — update failures roll back, they don't delete
- "The stack stays in mixed state" — only happens if rollback itself fails
- **Keywords:** "update fails midway" → automatic rollback of ALL changes

---

**Q2:** A team has two CloudFormation stacks. Stack A creates a VPC and exports the VPC ID. Stack B imports the VPC ID. The team tries to delete Stack A but gets an error. Why?

**A:** You **cannot delete a stack whose exports are being imported by another stack**. Stack B must be updated to remove the `Fn::ImportValue` reference (or deleted) before Stack A can be deleted.

**Why wrong answers are wrong:**
- "Add DeletionPolicy: Retain" — DeletionPolicy is per-resource, doesn't affect export dependency
- "Delete Stack B first then Stack A" — technically works, but question asks "why the error"
- "Use Fn::GetAtt instead of ImportValue" — you can't reference resources across stacks without export/import
- **Keywords:** "cannot delete", "exports", "imported by another stack" → cross-stack reference dependency

---

**Q3:** A company wants to use CloudFormation to deploy resources but wants to prevent anyone from directly updating the production database resource through CloudFormation updates. What mechanism provides this protection?

**A:** **Stack Policy** — define a Deny rule for `Update:Replace` and `Update:Delete` on the database logical resource. Stack policies protect specific resources from being modified during stack updates.

**Why wrong answers are wrong:**
- "DeletionPolicy" — only protects on stack deletion, not during updates
- "IAM policy denying cloudformation:UpdateStack" — too broad, prevents ALL stack updates
- "SCP denying RDS modifications" — prevents direct RDS API calls but CloudFormation uses its own role
- **Keywords:** "prevent update through CloudFormation", "specific resource protection" → Stack Policy

---

**Q4:** A company uses CDK and wants to ensure all S3 buckets created by any team have encryption enabled and public access blocked, without modifying each team's code. How?

**A:** Use **CDK Aspects** — create an Aspect that visits all S3 Bucket constructs in the tree and adds/enforces encryption and public access block configuration. Apply the Aspect at the App level so it affects all stacks.

**Why wrong answers are wrong:**
- "Use an SCP" — SCPs can deny unencrypted bucket creation but can't inject CloudFormation properties
- "Use Service Catalog constraints" — works but requires teams to use Service Catalog products
- "Modify each team's stack" — the question says "without modifying each team's code"
- **Keywords:** "all buckets", "any team", "without modifying code" → CDK Aspects

---

**Q5:** A team has a CloudFormation stack with an EC2 instance. They change the instance type from t3.medium to t3.large. What type of update occurs?

**A:** This is an **Update with No Interruption** (or brief interruption for instance stop/start). CloudFormation stops the instance, changes the type, and starts it. The instance is NOT replaced (same physical ID).

But if they change the **AMI ID**, that's a **Replacement** — new instance is created, old one deleted.

**Why this is tricky:**
- Candidates confuse "which property changes cause replacement" — Instance Type = no replacement, AMI = replacement, Availability Zone = replacement
- **Keywords:** "change instance type" → no replacement (stop/start). "Change AMI" → replacement (new instance)

---

**Q6:** A company deploys Lambda functions using SAM. They need to gradually shift traffic from the old Lambda version to the new one, rolling back if the error rate exceeds 5%. What SAM configuration achieves this?

**A:** In the SAM template, use `AutoPublishAlias` with `DeploymentPreference`:
```yaml
AutoPublishAlias: live
DeploymentPreference:
  Type: Linear10PercentEvery1Minute
  Alarms:
    - !Ref ErrorAlarm
```
SAM automatically creates the CodeDeploy application, deployment group, and alias traffic shifting.

**Why wrong answers are wrong:**
- "Manually configure CodeDeploy" — SAM does this automatically with DeploymentPreference
- "Use Lambda weighted aliases manually" — no automatic rollback on alarm
- "Use API Gateway canary" — canary at API level, not Lambda level
- **Keywords:** "SAM", "gradually shift traffic", "Lambda", "rollback on error" → DeploymentPreference + Alarms

---

**Q7:** A company has a CloudFormation stack that takes 45 minutes to create because of an RDS instance. During creation, the EC2 instances launch but the application fails because the database isn't ready yet. How should this be fixed?

**A:** Add a `DependsOn` attribute on the EC2 instances (or their Auto Scaling group) pointing to the RDS resource. This ensures CloudFormation doesn't create EC2 instances until RDS creation is COMPLETE. Additionally, use `CreationPolicy` with `cfn-signal` on the ASG to ensure instances are fully configured before the stack reports success.

**Why wrong answers are wrong:**
- "Add a wait condition" — WaitCondition is for external signals, not resource ordering
- "Use Fn::GetAtt on RDS endpoint in EC2 UserData" — this creates an implicit dependency (which actually DOES work), but explicit DependsOn is clearer. However, if this is an answer choice, it's valid because implicit dependencies from Ref/GetAtt work too.
- "Increase the stack timeout" — doesn't solve the ordering issue, just allows more time
- **Keywords:** "application fails because database isn't ready" → DependsOn (or implicit dependency via Ref/GetAtt)

---

### Common Exam Traps & Pitfalls

1. **CDK deploys via CloudFormation** — it does NOT directly call AWS APIs. CDK synth → CF template → CF deploys. This means CF quotas and behaviors still apply.

2. **SAM is a CloudFormation extension** (Transform macro) — a SAM template IS a CloudFormation template with extra syntax. You can mix standard CF resources with SAM resources.

3. **DeletionPolicy vs UpdateReplacePolicy**: DeletionPolicy = stack deletion. UpdateReplacePolicy = when a resource is REPLACED during update. Both are needed for full protection.

4. **Stack Policy only applies during updates** — it does NOT prevent stack deletion. Use IAM/SCP to prevent `DeleteStack`.

5. **Cross-stack references create hard dependencies** — you cannot delete the exporting stack while imports exist. This catches many candidates.

6. **StackSets ≠ Nested Stacks**: StackSets = same template across accounts/regions. Nested Stacks = stack composition within one account/region.

7. **CloudFormation does NOT detect drift on ALL resource types** — only supported resources. Check documentation.

8. **Change Set creation is FREE and non-destructive** — always use for production. Executing it is what makes changes.

9. **`Fn::ImportValue` cannot be used with `Ref` or `Fn::GetAtt`** — the imported value must be a string literal or `Fn::Sub` result.

10. **CDK bootstrap is per-account-per-region** — must bootstrap EVERY target account/region before deploying there.

11. **CloudFormation custom resources** (Lambda-backed): If the Lambda fails, the stack hangs for 1 hour (default timeout) waiting for a response. Always implement error handling that sends SUCCESS/FAILED.

12. **SAM Globals section**: Default properties for all functions (Runtime, Timeout, MemorySize) — reduces repetition but candidates forget it exists.

---

## 7. Cheat Sheet

### Must-Know Facts

| Fact | Detail |
|------|--------|
| CF max resources per template | 500 |
| CF max parameters | 200 |
| CF max template size (S3) | 1 MB |
| CF max nested depth | 5 levels |
| CF exports per region | 5,000 |
| CF pricing | Free (pay for resources only) |
| CF third-party resources | $0.0009/operation |
| CDK construct levels | L1 (Cfn), L2 (curated), L3 (patterns) |
| CDK synth output | CloudFormation template |
| SAM Transform | `AWS::Serverless-2016-10-31` |
| SAM CLI local test | `sam local invoke` (Docker) |
| DeletionPolicy options | Retain, Snapshot, Delete |
| StackSets deployment | Cross-account + cross-region |
| Dynamic reference (SSM) | `{{resolve:ssm:name:version}}` |
| Dynamic reference (Secrets) | `{{resolve:secretsmanager:id}}` |
| Stack Policy | Protects during UPDATE only |
| Change Set | Preview only, free, non-destructive |

### Decision Flowchart

```
Question says "deploy same template to multiple accounts/regions"
  → StackSets

Question says "prevent accidental resource modification during update"
  → Stack Policy

Question says "keep database when stack is deleted"
  → DeletionPolicy: Retain or Snapshot

Question says "preview changes before applying"
  → Change Set

Question says "developers want to use TypeScript/Python for infrastructure"
  → CDK

Question says "serverless application with Lambda + API Gateway"
  → SAM (or CDK with serverless constructs)

Question says "enforce standards across all constructs without modifying code"
  → CDK Aspects

Question says "reference secrets at deploy time without hardcoding"
  → Dynamic references (ssm-secure / secretsmanager)

Question says "resources modified outside CloudFormation"
  → Drift Detection

Question says "share values between stacks"
  → Cross-stack references (Exports + Fn::ImportValue)

Question says "reuse template components"
  → Nested Stacks (single account) or Modules

Question says "auto-deploy to new accounts in organization"
  → StackSets with service-managed permissions + auto-deployment
```

### Key Differentiators

| Comparison | CloudFormation | CDK | SAM |
|-----------|---------------|-----|-----|
| Format | YAML/JSON (declarative) | TypeScript/Python/etc (imperative) | YAML (declarative, serverless-focused) |
| Learning curve | Moderate | Higher (need programming) | Lower (for serverless) |
| Testing | cfn-lint, TaskCat | jest/pytest (unit test constructs) | sam local (integration) |
| Reusability | Nested stacks, modules | npm/PyPI construct libraries | SAR (Serverless App Repo) |
| Local dev | None | `cdk synth` + watch mode | `sam local invoke/start-api` |
| Best for | Any AWS workload, ops teams | Complex apps, dev teams | Serverless-first apps |

---

## 8. Additional Deep-Dive

### 10 More Tricky Scenario-Based Questions

**Q1:** A company manages 200 AWS accounts with AWS Organizations. They need to deploy a security baseline (Config rules, GuardDuty, CloudTrail) to all current AND future accounts automatically. What's the best approach?

**A:** Use **CloudFormation StackSets** with **service-managed permissions** targeting the Organization root or specific OUs. Enable **auto-deployment** — when a new account is added to the OU, the StackSet automatically deploys the baseline stack.

**Keywords:** "all current AND future accounts", "automatically" → StackSets + service-managed + auto-deploy

---

**Q2:** A CloudFormation stack update replaces an RDS instance (because the engine version changed). The team wants to keep the old database's data. What configuration prevents data loss?

**A:** Set `UpdateReplacePolicy: Snapshot` on the RDS resource. When CloudFormation replaces the resource, it takes a snapshot of the OLD instance before deleting it. Note: `DeletionPolicy` alone doesn't help here because the resource is being REPLACED, not stack-deleted.

**Keywords:** "update replaces resource", "keep data" → UpdateReplacePolicy: Snapshot (NOT just DeletionPolicy)

---

**Q3:** A team uses CDK Pipelines for deployment. They add a new deployment stage for EU region. After pushing the code, the pipeline does NOT deploy to EU. Why?

**A:** CDK Pipelines is **self-mutating** — the pipeline updates itself from the source. BUT: the EU account/region must be **bootstrapped** first (`cdk bootstrap aws://ACCOUNT/eu-west-1 --trust PIPELINE_ACCOUNT`). Without bootstrapping, the pipeline can't deploy there. Also, cross-account bootstrapping must trust the pipeline account.

**Keywords:** "CDK Pipelines", "new region/account not working" → Must bootstrap target first

---

**Q4:** A CloudFormation custom resource (Lambda-backed) is used to create a DNS record in an external DNS provider. The stack creation hangs for an hour and then fails. What's the root cause?

**A:** The Lambda function didn't send a response (SUCCESS or FAILED) to the CloudFormation **pre-signed S3 URL**. Custom resources MUST send a response — if the Lambda crashes or times out without responding, CloudFormation waits for the default 1-hour timeout. Fix: implement try/catch that always sends a response, and set a shorter Lambda timeout.

**Keywords:** "custom resource", "hangs", "timeout" → Lambda failed to send response to CF callback URL

---

**Q5:** A company wants to enforce that all CloudFormation stacks must include specific tags (CostCenter, Environment) and all S3 buckets must have versioning enabled. They want this enforced BEFORE resources are created. What should they use?

**A:** **CloudFormation Hooks** (part of CloudFormation Registry) — write a hook that inspects resource configurations before creation (preCreate handler). Hooks can FAIL the operation if compliance rules aren't met, preventing non-compliant resources from being created.

**Why not Config Rules?** Config Rules are detective (after creation), not preventive.
**Why not SCPs?** SCPs deny API actions but can't inspect resource configurations/properties.

**Keywords:** "enforce BEFORE creation", "inspect properties" → CloudFormation Hooks

---

**Q6:** A team has a CloudFormation stack with an ASG. During stack update, they want instances replaced using a rolling update (replace 2 at a time, wait for health check). What attribute configures this?

**A:** **UpdatePolicy** attribute on the `AWS::AutoScaling::AutoScalingGroup` resource with `AutoScalingRollingUpdate`:
```yaml
UpdatePolicy:
  AutoScalingRollingUpdate:
    MaxBatchSize: 2
    MinInstancesInService: 2
    PauseTime: PT10M
    WaitOnResourceSignals: true
```

**Why wrong answers are wrong:**
- "DependsOn" — controls creation order, not update behavior
- "CreationPolicy" — controls initial creation signals, not updates
- "Use CodeDeploy" — works but not CloudFormation-native for ASG rolling updates
- **Keywords:** "ASG", "rolling update", "replace N at a time" → UpdatePolicy: AutoScalingRollingUpdate

---

**Q7:** A team wants to use CloudFormation to deploy to production but ensure that the DynamoDB table is NEVER deleted or replaced, even if someone accidentally removes it from the template. What combination of protections is needed?

**A:** Three layers: (1) `DeletionPolicy: Retain` — survives stack deletion, (2) `UpdateReplacePolicy: Retain` — survives replacement during updates, (3) **Stack Policy** denying `Update:Replace` and `Update:Delete` on the table — prevents the update from even being attempted.

**Keywords:** "NEVER deleted or replaced", "even if removed from template" → All three: DeletionPolicy + UpdateReplacePolicy + Stack Policy

---

**Q8:** A company uses SAM and wants to run integration tests AFTER deployment but BEFORE traffic shifts to the new Lambda version. How is this configured?

**A:** In the SAM template's `DeploymentPreference`, use **Hooks**:
```yaml
DeploymentPreference:
  Type: Canary10Percent5Minutes
  Hooks:
    PreTraffic: !Ref PreTrafficTestFunction
    PostTraffic: !Ref PostTrafficTestFunction
```
The `PreTraffic` hook Lambda runs integration tests against the new version before any traffic shifts.

**Keywords:** "after deployment, before traffic shift", "SAM" → DeploymentPreference Hooks (PreTraffic)

---

**Q9:** A company has a CloudFormation stack that creates a Lambda function. The function code is in an S3 bucket. They update the code in S3 but the CloudFormation stack update shows "No updates to perform." Why?

**A:** CloudFormation tracks the **template**, not the content of referenced artifacts. If the S3 key/version hasn't changed in the template, CloudFormation sees no difference. Fix: Use **S3 object versioning** and reference the `S3ObjectVersion` property, OR change the S3 key name (e.g., include hash), OR use SAM/CDK (which handle this automatically with asset hashing).

**Keywords:** "code changed in S3", "no updates to perform" → Template hasn't changed (S3 key is the same)

---

**Q10:** A company is migrating from Terraform to CloudFormation. They have 100 existing AWS resources. They don't want to recreate them. What's the approach?

**A:** Use **CloudFormation resource import** (`--change-set-type IMPORT`). Create a template describing the existing resources with their physical IDs, then import them into a new stack. Resources are adopted by CloudFormation without being recreated or modified.

**Keywords:** "existing resources", "don't recreate", "adopt/import" → Resource Import

---

### Compare: CloudFormation vs CDK vs SAM

| Aspect | CloudFormation | CDK | SAM |
|--------|---------------|-----|-----|
| **Use Case** | Any AWS workload, ops-driven | Complex multi-service apps, dev-driven | Serverless (Lambda, API GW, DynamoDB) |
| **Limits** | 500 resources/template | Same (generates CF) | Same (is CF with Transform) |
| **Pricing** | Free | Free | Free |
| **HA Model** | Built-in rollback, StackSets multi-region | Same as CF | Same as CF |
| **Exam Tip** | Default IaC answer for AWS | "Developer-friendly", "programming language", "reusable constructs" | "Serverless", "Lambda local testing", "simplified deployment" |

### Compare: Nested Stacks vs StackSets vs Cross-Stack References

| Aspect | Nested Stacks | StackSets | Cross-Stack References |
|--------|---------------|-----------|----------------------|
| **Use Case** | Reusable components in ONE account/region | Same template across MANY accounts/regions | Share values between independent stacks |
| **Limits** | 5 levels deep | 2000 instances per StackSet | 5000 exports per region |
| **Dependency** | Parent manages lifecycle | Admin manages, target accounts receive | Loose coupling (export/import) |
| **Exam Tip** | "Reuse VPC template as component" | "Deploy to all accounts in org" | "Stack A provides VPC ID to Stack B" |

### When Does SAP-C02 Expect CloudFormation vs CDK vs SAM?

**Pick CloudFormation when:**
- "Infrastructure as Code" (generic)
- "Declarative template"
- "Stack", "nested stack", "StackSet"
- "Drift detection"
- "Resource import"
- "Stack policy"

**Pick CDK when:**
- "Developer-friendly IaC"
- "Programming language" / "TypeScript" / "Python for infrastructure"
- "Reusable constructs" / "share infrastructure patterns via npm"
- "Self-mutating pipeline"
- "Unit test infrastructure"

**Pick SAM when:**
- "Serverless application"
- "Local Lambda testing"
- "Simplified Lambda deployment"
- "Traffic shifting for Lambda" (DeploymentPreference)
- "sam local invoke" / "sam deploy"

**Pick Terraform when:**
- "Multi-cloud" / "not AWS-only"
- "Provider-agnostic"
- (Terraform is rarely the right answer on SAP-C02 — it's an AWS exam)

### Gotcha Differences

| Feature | CloudFormation | Terraform |
|---------|---------------|-----------|
| State | Managed by AWS (no state file) | State file (S3 + DynamoDB lock) |
| Drift | Built-in drift detection | `terraform plan` shows drift |
| Rollback | Automatic on failure | Manual (no auto-rollback) |
| Scope | AWS only (+some third-party via Registry) | Multi-cloud |
| Exam relevance | Primary IaC answer | Only for "multi-cloud" scenarios |

| Feature | Nested Stacks | Cross-Stack References |
|---------|---------------|----------------------|
| Coupling | Tight (parent owns child lifecycle) | Loose (independent stacks) |
| Update | Update parent → updates children | Update independently |
| Delete | Delete parent → deletes children | Must remove imports before deleting exporter |
| Use when | Reusable components, single ownership | Independent teams sharing values |

### Decision Tree: IaC Tool Selection

```
Is it serverless-focused (Lambda + API GW + DynamoDB)?
  ├── YES → Do you need local testing?
  │         ├── YES → SAM
  │         └── NO → SAM or CDK (both work)
  └── NO → Is the team developer-heavy (TypeScript/Python)?
            ├── YES → Do you need complex logic/conditions?
            │         ├── YES → CDK
            │         └── NO → CDK or CloudFormation (either works)
            └── NO → CloudFormation (YAML/JSON, ops-friendly)

Need to deploy across accounts/regions?
  → StackSets

Need to reuse template components in same account?
  → Nested Stacks

Need to share values between independent stacks?
  → Cross-Stack References (Exports + ImportValue)

Need to prevent resource modification during updates?
  → Stack Policy

Need to protect resource on stack deletion?
  → DeletionPolicy: Retain or Snapshot

Need to enforce compliance before resource creation?
  → CloudFormation Hooks (or Guard for template validation)

Need to detect out-of-band changes?
  → Drift Detection

Need multi-cloud IaC?
  → Terraform (not CloudFormation)
```
