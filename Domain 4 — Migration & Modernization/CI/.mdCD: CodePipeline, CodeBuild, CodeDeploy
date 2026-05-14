# CI/CD: CodePipeline, CodeBuild, CodeDeploy

## 1. Core Concepts & Theory

### AWS CodePipeline
- **Fully managed continuous delivery service** that orchestrates build, test, and deploy phases
- **Pipeline**: A workflow that describes how software changes go through release stages
- **Stage**: A logical unit (e.g., Source, Build, Test, Deploy, Approval) — each stage has one or more action groups
- **Action**: A task performed on an artifact (e.g., source action, build action, deploy action)
- **Action Group**: Actions within a stage that run in parallel (sequential between groups)
- **Artifact**: Files worked on by actions (stored in S3) — input artifacts and output artifacts
- **Transition**: Link between stages (can be disabled to stop pipeline progression)

**Key Limits:**
- Max **50 pipelines per region** (soft limit, can increase)
- Max **50 stages per pipeline** (hard limit)
- Max **50 actions per stage**
- Max **5 parallel actions per action group**
- Artifact size limit: **5 GB** per artifact in S3
- Pipeline execution timeout: **No overall timeout** (individual actions have timeouts)

### AWS CodeBuild
- **Fully managed build service** — compiles source code, runs tests, produces deployable artifacts
- **Build Project**: Configuration (source, environment, buildspec, artifacts, logs)
- **Build Environment**: Docker container where build runs (managed images or custom)
- **Buildspec.yml**: YAML file defining build commands (phases: install, pre_build, build, post_build)
- **Build Phases**: install → pre_build → build → post_build
- **Compute Types**: BUILD_GENERAL1_SMALL (4 GB, 2 vCPU), MEDIUM (8 GB, 4 vCPU), LARGE (16 GB, 8 vCPU), 2XLARGE (145 GB, 72 vCPU)
- **Local caching**: Source cache, Docker layer cache, Custom cache (speeds up repeated builds)

**Key Limits:**
- Max **5,000 build projects per region**
- Max concurrent builds: **60** (default, can increase)
- Build timeout: **5 minutes to 8 hours** (default 60 min)
- Max buildspec file size: **256 KB**
- Environment variables: **200 per project**

### AWS CodeDeploy
- **Fully managed deployment service** for EC2, Lambda, ECS, and on-premises servers
- **Application**: Name/container for deployment configurations
- **Deployment Group**: Set of instances or Lambda/ECS targets
- **Deployment Configuration**: Rules for deployment (e.g., AllAtOnce, HalfAtATime, OneAtATime, custom)
- **AppSpec file**: YAML/JSON defining deployment actions (appspec.yml for EC2/on-prem, appspec.yaml for Lambda/ECS)
- **Revision**: Application content + AppSpec file
- **Agent**: CodeDeploy agent required on EC2/on-prem instances

**Deployment Types:**
| Platform | In-Place | Blue/Green |
|----------|----------|------------|
| EC2/On-prem | Yes | Yes (new ASG) |
| Lambda | N/A | Yes (traffic shifting) |
| ECS | N/A | Yes (task set replacement) |

**Lambda Traffic Shifting Options:**
- **Canary**: X% first, then 100% after interval (e.g., Canary10Percent5Minutes)
- **Linear**: X% every N minutes (e.g., Linear10PercentEvery1Minute)
- **AllAtOnce**: Immediate 100% shift

**ECS Deployment:**
- Blue/Green only (replacement task set created)
- Same traffic shifting as Lambda (Canary, Linear, AllAtOnce)
- Requires ALB or NLB with two target groups

**EC2/On-prem Lifecycle Hooks (in-place):**
ApplicationStop → DownloadBundle → BeforeInstall → Install → AfterInstall → ApplicationStart → ValidateService

**Key Limits:**
- Max **1,000 applications per account per region**
- Max **1,000 deployment groups per application**
- Max **1,000 concurrent deployments per account**
- CodeDeploy agent communicates over **HTTPS (port 443)**

---

## 2. Design Patterns & Best Practices

### When to Use

| Service | Use When | Don't Use When |
|---------|----------|----------------|
| CodePipeline | You need orchestrated, multi-stage release workflow | Simple single-step deploys (use CodeDeploy directly) |
| CodeBuild | You need managed build/test without maintaining servers | You already have Jenkins/TeamCity and don't want to migrate |
| CodeDeploy | You need controlled deployments to EC2, Lambda, or ECS | Deploying to S3 (use S3 deploy action in CodePipeline) or deploying CloudFormation (use CF action) |

### Anti-Patterns
- **Don't use CodePipeline for event-driven, non-linear workflows** — it's strictly linear with stages; use Step Functions for complex branching
- **Don't use CodeBuild for long-running processes** — 8-hour max; use ECS/Fargate for builds > 8 hours
- **Don't use CodeDeploy in-place for zero-downtime requirements** — use Blue/Green instead
- **Don't use AllAtOnce for production** — no rollback safety; use Canary or Linear

### Architectural Patterns

**Multi-Account CI/CD (Cross-Account Pipeline):**
```
Tools Account (Pipeline) → Build Account (CodeBuild) → Staging Account (Deploy) → Prod Account (Deploy)
```
- Pipeline in tools/shared-services account
- Cross-account IAM roles for each target account
- KMS CMK shared across accounts for artifact encryption
- S3 artifact bucket policy allows cross-account access

**Multi-Region Deployment:**
- CodePipeline supports **cross-region actions** natively
- Artifacts replicated to target region's S3 bucket automatically
- Each region needs its own artifact bucket
- Deploy actions specify the region

**GitOps Pattern:**
- Source: CodeCommit/GitHub → CodePipeline trigger
- Build: CodeBuild generates manifests/artifacts
- Deploy: CodeDeploy or CloudFormation deploy action

**Blue/Green with Route 53:**
- CodeDeploy creates new ASG (green)
- Validates green environment (health checks + lifecycle hooks)
- Switches Route 53 DNS or ELB target groups
- Keeps old (blue) for rollback window

### Well-Architected Alignment

| Pillar | Practice |
|--------|----------|
| Reliability | Blue/Green deployments, automated rollback on CloudWatch alarms |
| Security | Cross-account roles, encrypted artifacts (KMS), no secrets in buildspec (use Parameter Store/Secrets Manager) |
| Cost | Use CodeBuild spot instances (SPOT compute type for Linux), right-size compute |
| Performance | Parallel actions in CodePipeline, CodeBuild caching, batch builds |
| Ops Excellence | Pipeline notifications (SNS/EventBridge), CloudWatch metrics, manual approval gates |

### Integration Points
- **CodeCommit / GitHub / Bitbucket / S3** → Source stage
- **ECR** → Source for container image changes
- **CloudFormation** → Deploy stage (create/update stacks, change sets)
- **Elastic Beanstalk** → Deploy action
- **ECS** → Blue/Green via CodeDeploy or standard ECS deploy action
- **Service Catalog** → Deploy constraints
- **EventBridge** → Pipeline state change events
- **SNS** → Notifications, manual approval
- **Parameter Store / Secrets Manager** → Build-time secrets
- **CloudWatch Alarms** → CodeDeploy auto-rollback trigger

---

## 3. Security & Compliance

### IAM & Access Control

**CodePipeline Service Role:**
- Needs access to: S3 (artifacts), CodeBuild (StartBuild), CodeDeploy (CreateDeployment), CloudFormation, IAM PassRole
- Cross-account: AssumeRole to target account roles

**CodeBuild Service Role:**
- Needs: CloudWatch Logs (CreateLogGroup, PutLogEvents), S3 (GetObject, PutObject for artifacts), ECR (if building images), Secrets Manager / SSM (for secrets)
- VPC access: Add EC2 permissions (CreateNetworkInterface, DeleteNetworkInterface, DescribeNetworkInterfaces)

**CodeDeploy Service Role:**
- EC2: Needs Auto Scaling, ELB, EC2 describe permissions
- ECS: Needs ECS, ELB, IAM PassRole
- Lambda: Needs Lambda invoke/update permissions

**Cross-Account Pattern:**
```
Tools Account: Pipeline role → sts:AssumeRole → Target Account Role
Target Account: Trust policy allows Tools Account role
KMS Key: Key policy allows Tools Account + Target Account
S3 Bucket: Bucket policy allows cross-account GetObject
```

### Encryption
- **Artifacts**: Encrypted at rest in S3 using AWS-managed key (default) or customer-managed CMK
- **Cross-account**: Must use customer-managed CMK (aws/s3 key can't be shared)
- **CodeBuild**: Build output encrypted with S3 SSE; environment variables can use SSM SecureString or Secrets Manager
- **In-transit**: All API calls over HTTPS/TLS

### Secrets Management in Builds
- **NEVER put secrets in buildspec.yml** (it's committed to source)
- Use **SSM Parameter Store** (parameter-store type environment variable)
- Use **Secrets Manager** (secrets-manager type environment variable)
- CodeBuild resolves secrets at build time, never exposes plaintext in logs

### Logging & Auditing
- **CloudTrail**: All API calls (CreatePipeline, StartBuild, CreateDeployment)
- **CodeBuild logs**: CloudWatch Logs (default) or S3
- **CodeDeploy logs**: On-instance at `/var/log/aws/codedeploy-agent/` or via CloudWatch Logs agent
- **Pipeline events**: EventBridge for state changes (STARTED, SUCCEEDED, FAILED)

### Compliance Patterns
- **Approval gates**: Manual approval action with SNS notification (segregation of duties)
- **Artifact integrity**: S3 versioning + object lock for immutable artifacts
- **Audit trail**: CloudTrail + Config rules for pipeline configuration changes
- **VPC isolation for builds**: CodeBuild in VPC for accessing private resources (no public internet unless NAT Gateway or VPC endpoints configured)

---

## 4. Cost Optimization

### Pricing Models

**CodePipeline:**
- **$1.00 per active pipeline per month** (first pipeline free in Free Tier for 30 days)
- "Active" = at least one code change processed through the pipeline in a month
- V2 pipelines: $0.002 per action execution minute (after first 100 free minutes)

**CodeBuild:**
- Pay per build minute (rounded up to nearest minute)
- Linux: $0.005/min (small), $0.01/min (medium), $0.02/min (large), $0.20/min (2xlarge)
- ARM: 20% cheaper than x86
- Windows: ~2x Linux pricing
- **Free Tier**: 100 build minutes/month (general1.small)

**CodeDeploy:**
- **Free for EC2/Lambda/ECS** deployments
- **$0.02 per on-premises instance update** (only on-prem costs money)

### Cost-Saving Strategies
- **CodeBuild**: Use `BUILD_GENERAL1_SMALL` unless you need more resources
- **CodeBuild spot**: Set `computeType` to use EC2 Spot instances (Linux only) — not a native feature, but you can use custom fleets
- **Caching**: Enable S3/local caching to reduce build duration (and cost)
- **Pipeline consolidation**: Combine related deployments to fewer pipelines
- **Disable idle pipelines**: Disable transitions or delete unused pipelines
- **Batch builds**: CodeBuild batch builds can share artifacts between build tasks, reducing redundant work
- **Reserved capacity**: CodeBuild reserved capacity for predictable workloads

### Cost Traps
- CodeBuild **timeout left at default (60 min)** with a stuck build = wasted cost
- Leaving **artifact buckets** growing without lifecycle rules
- **Over-provisioned build environments** (using LARGE when SMALL suffices)
- V2 pipelines: action execution minutes can accumulate with many stages/actions

---

## 5. High Availability, Disaster Recovery & Resilience

### High Availability

**CodePipeline:**
- Regional service, fully managed (multi-AZ within region)
- Artifacts stored in S3 (11 9s durability)
- No customer-managed HA configuration needed
- Cross-region actions for multi-region deployments

**CodeBuild:**
- Fully managed, scales automatically
- Concurrent builds up to account limits
- No capacity planning required

**CodeDeploy:**
- Regional service
- Deployment to multiple AZs simultaneously
- Health-check-based validation ensures only healthy instances serve traffic

### Deployment Resilience Patterns

**Rollback Strategies:**
| Platform | Rollback Method |
|----------|----------------|
| EC2 In-Place | Redeploy previous revision |
| EC2 Blue/Green | Reroute traffic back to original (blue) ASG |
| Lambda | Shift traffic back to previous version |
| ECS | Shift traffic back to original task set |

**Automatic Rollback Triggers:**
- Deployment fails (instances fail health checks)
- CloudWatch alarm enters ALARM state during deployment
- Configure rollback on alarm in Deployment Group settings

**Minimum Healthy Hosts:**
- Ensures deployment never takes too many instances offline
- `MinimumHealthyHosts`: Value (e.g., 75%) or count
- If threshold breached → deployment fails → rollback

### DR Patterns

**Multi-Region Pipeline:**
```
Source (us-east-1) → Build (us-east-1) → Deploy us-east-1 → Deploy eu-west-1
```
- Cross-region actions handle artifact replication
- Each region's deployment independent (one region failure doesn't block others)

**Pipeline Recovery:**
- Pipeline definition as code (CloudFormation/CDK) → can recreate in DR region
- Artifacts in S3 (replicated via CRR if needed)
- Source (CodeCommit) supports cross-region replication

---

## 6. Exam-Focused Section

### Straightforward Questions (7 examples)

**Q1:** A company wants to deploy a new application version to EC2 instances with zero downtime. Which CodeDeploy deployment type should they use?
**A:** Blue/Green deployment — creates new instances, validates, then shifts traffic. In-place would cause downtime as instances are updated one by one.

**Q2:** Where should sensitive database credentials be stored for use during a CodeBuild build?
**A:** AWS Systems Manager Parameter Store (SecureString) or AWS Secrets Manager — referenced as environment variables of type `PARAMETER_STORE` or `SECRETS_MANAGER` in the build project.

**Q3:** A pipeline needs human approval before deploying to production. How is this implemented?
**A:** Add a Manual Approval action in CodePipeline (can send SNS notification to approvers with a URL to approve/reject).

**Q4:** What file does CodeDeploy use to define deployment lifecycle hooks on EC2?
**A:** `appspec.yml` — placed in the root of the revision, defines files to copy and lifecycle hook scripts.

**Q5:** How can CodePipeline deploy to multiple AWS regions?
**A:** Use cross-region actions — specify the region in the deploy action configuration. CodePipeline handles artifact replication to the target region automatically.

**Q6:** A CodeBuild project needs to access a private RDS database during integration tests. What must be configured?
**A:** Configure CodeBuild to run in a VPC (specify VPC, subnets, and security groups). Ensure security group allows outbound to RDS. If internet access is also needed, use a NAT Gateway.

**Q7:** What compute type should be used for a simple Node.js build with `npm install` and `npm test`?
**A:** `BUILD_GENERAL1_SMALL` (4 GB RAM, 2 vCPU) — sufficient for most builds, cheapest option.

---

### Tricky / Scenario-Based Questions (7 examples)

**Q1:** A company has a CodePipeline with CodeDeploy deploying to an Auto Scaling group. After deployment, new instances launched by Auto Scaling have the OLD application version. How do you fix this?

**A:** Enable **CodeDeploy Auto Scaling integration** — when you create a deployment group with an ASG, CodeDeploy automatically hooks into ASG lifecycle to deploy the latest successful revision to new instances. If this isn't working, check that the deployment group is associated with the ASG (not just individual EC2 instances tagged manually).

**Why wrong answers are wrong:**
- "Use a launch template with updated AMI" — works but requires AMI rebuild on every deploy (operational overhead)
- "Add a lifecycle hook to ASG that runs a script" — reinvents what CodeDeploy already does natively
- **Keywords:** "Auto Scaling", "new instances", "old version" → think CodeDeploy ASG integration

---

**Q2:** A team needs to deploy a containerized application on ECS Fargate with the ability to test 10% of traffic on the new version before full rollout. Which approach meets this requirement with **least operational overhead**?

**A:** CodeDeploy Blue/Green deployment for ECS with **Canary10Percent5Minutes** configuration. CodeDeploy natively supports ECS Blue/Green with traffic shifting.

**Why wrong answers are wrong:**
- "Use an ALB with weighted target groups manually" — works but high operational overhead (manual traffic management)
- "Use CodePipeline ECS standard deploy action" — this does rolling updates, not canary traffic shifting
- "Use Route 53 weighted routing" — works for multi-region, overkill for single-cluster canary, and requires two services
- **Keywords:** "ECS", "10% traffic", "test before full rollout", "least operational overhead" → CodeDeploy ECS Blue/Green + Canary

---

**Q3:** A company's CodePipeline uses CodeCommit as source, CodeBuild for builds, and deploys via CloudFormation. They want to add automated security scanning of CloudFormation templates BEFORE deployment. Where should this be added?

**A:** Add a **CodeBuild action** in a Test stage (or within the Build stage as a separate action group) that runs cfn-lint, cfn-nag, or similar scanning tools. Place it BEFORE the CloudFormation deploy action.

**Why wrong answers are wrong:**
- "Use AWS Config rules" — Config evaluates AFTER deployment, not before
- "Add a Lambda action" — possible but more operational overhead than CodeBuild for running CLI tools
- "Use CloudFormation Guard in the deploy action" — Guard doesn't run as part of the deploy action itself
- **Keywords:** "BEFORE deployment", "automated scanning" → CodeBuild action in Test/Build stage

---

**Q4:** A company has a cross-account pipeline. The pipeline is in a Tools account and deploys CloudFormation to a Production account. Deployments fail with "Access Denied" on the KMS key. What's the likely issue?

**A:** The **KMS CMK key policy** doesn't grant the Production account's CloudFormation role permission to Decrypt artifacts. Cross-account pipelines require: (1) CMK key policy allows target account, (2) S3 bucket policy allows target account, (3) Target account role has KMS Decrypt permission.

**Why wrong answers are wrong:**
- "The S3 bucket policy is wrong" — possible but question specifically mentions KMS
- "Use the default aws/s3 key" — the default AWS-managed key CANNOT be shared cross-account (this is a key fact)
- "The pipeline service role needs more permissions" — the pipeline role isn't the one accessing artifacts in the target account
- **Keywords:** "cross-account", "KMS", "Access Denied" → CMK key policy + bucket policy + target role

---

**Q5:** A Lambda function needs to be updated with a new version. The team wants to gradually shift traffic: 10% initially, then increase by 10% every 2 minutes until 100%. If errors exceed 1%, roll back immediately. Which configuration achieves this?

**A:** CodeDeploy with deployment configuration **Linear10PercentEvery2Minutes** and a **CloudWatch alarm** (monitoring Lambda errors/invocations > 1%) configured as an automatic rollback trigger.

**Why wrong answers are wrong:**
- "Canary10Percent5Minutes" — Canary only does two steps (X% then 100%), not gradual linear increase
- "Use Lambda aliases with weighted routing manually" — no automatic rollback on alarm
- "Use API Gateway canary release" — this is at the API level, not Lambda level, and doesn't auto-rollback
- **Keywords:** "gradually", "increase by 10% every 2 minutes", "roll back if errors" → Linear + CloudWatch alarm

---

**Q6:** A company runs CodeBuild in a VPC to access private resources. Builds are now failing to download dependencies from the internet. What should be done?

**A:** Add a **NAT Gateway** (or NAT Instance) in a public subnet, and configure the CodeBuild subnet's route table to route `0.0.0.0/0` through the NAT Gateway. CodeBuild in a VPC does NOT get a public IP.

**Why wrong answers are wrong:**
- "Add an Internet Gateway" — IGW alone doesn't help; CodeBuild containers don't get public IPs in VPC mode
- "Use VPC endpoints for everything" — only works for AWS services (S3, ECR), not arbitrary internet dependencies (npm, pip)
- "Remove VPC configuration" — would lose access to private resources
- **Keywords:** "CodeBuild", "VPC", "failing to download from internet" → NAT Gateway

---

**Q7:** A company wants ONE pipeline to deploy the same application to 3 environments (Dev, Staging, Prod) in sequence, with manual approval before Staging and Prod. The pipeline should NOT proceed to Staging if Dev deployment fails. How should this be architected?

**A:** Single CodePipeline with stages: Source → Build → Deploy-Dev → Approval-Staging → Deploy-Staging → Approval-Prod → Deploy-Prod. Stages are sequential by default — if Deploy-Dev fails, the pipeline stops (transitions only happen on success).

**Why wrong answers are wrong:**
- "Use 3 separate pipelines triggered sequentially" — more operational overhead, harder to manage as a single release
- "Use parallel deploy actions for all environments" — violates the sequential requirement
- "Use EventBridge to chain pipelines" — unnecessary complexity when one pipeline handles sequential stages natively
- **Keywords:** "sequence", "manual approval", "should NOT proceed if fails" → Single pipeline with sequential stages + approval actions

---

### Common Exam Traps & Pitfalls

1. **CodeDeploy In-Place vs Blue/Green on Lambda/ECS**: Lambda and ECS ONLY support Blue/Green. In-place is EC2/On-prem only.

2. **Default AWS-managed KMS key (aws/s3) cannot be shared cross-account** — must use customer-managed CMK for cross-account pipelines.

3. **CodeBuild in VPC loses internet access** — requires NAT Gateway. Many candidates forget this.

4. **CodeDeploy is FREE for EC2/Lambda/ECS** — only on-premises instances cost money ($0.02/update).

5. **Canary vs Linear**: Canary = two-step (X% then 100%). Linear = gradual (X% every N minutes). Don't confuse them.

6. **CodePipeline vs CodeDeploy rollback**: CodePipeline doesn't rollback — it re-executes with the previous artifact. CodeDeploy has native rollback (reverts to last successful deployment).

7. **appspec.yml location**: Must be in the ROOT of the deployment revision. Wrong location = deployment fails immediately.

8. **CodePipeline source triggers**: CodeCommit uses EventBridge (default), NOT polling. GitHub uses webhooks (via CodeStar Connections) or polling.

9. **CodeBuild environment variables vs SSM**: Plain text env vars are visible in the console. Use `PARAMETER_STORE` or `SECRETS_MANAGER` type for secrets.

10. **CodeDeploy Agent**: Required on EC2/on-prem. NOT required for Lambda or ECS deployments.

---

## 7. Cheat Sheet

### Must-Know Facts

| Fact | Detail |
|------|--------|
| CodePipeline pricing | $1/active pipeline/month |
| CodeBuild timeout | 5 min to 8 hours (default 60 min) |
| CodeDeploy EC2/Lambda/ECS cost | FREE |
| CodeDeploy on-prem cost | $0.02 per instance update |
| Cross-account KMS | Must use customer-managed CMK |
| CodeBuild in VPC | Loses internet — needs NAT GW |
| Lambda/ECS deploy type | Blue/Green ONLY |
| EC2 deploy types | In-place AND Blue/Green |
| Canary | Two-step: X% → wait → 100% |
| Linear | Gradual: X% every N minutes |
| Pipeline max stages | 50 |
| Build max concurrency | 60 (default) |
| CodeDeploy agent port | 443 (HTTPS) |

### Decision Flowchart

```
Question says "orchestrate multiple stages/steps" → CodePipeline
Question says "build/compile/test code" → CodeBuild
Question says "deploy to EC2/Lambda/ECS with traffic control" → CodeDeploy
Question says "zero downtime deploy to EC2" → Blue/Green (CodeDeploy)
Question says "canary" or "gradual traffic shift" → CodeDeploy (Canary/Linear)
Question says "cross-account deploy" → Cross-account IAM roles + CMK + bucket policy
Question says "on-premises deployment" → CodeDeploy (only service that supports on-prem)
Question says "build needs private DB access" → CodeBuild in VPC + NAT GW for internet
Question says "approval before production" → Manual Approval action in CodePipeline
Question says "rollback on errors" → CodeDeploy + CloudWatch alarm trigger
```

### Key Differentiators

| vs. Comparison | Choose A When | Choose B When |
|----------------|---------------|---------------|
| CodePipeline vs Jenkins | Fully managed, native AWS integration, no servers | Need complex plugin ecosystem, non-AWS sources |
| CodeBuild vs Jenkins | No servers to manage, pay-per-minute, auto-scale | Need persistent build agents, custom hardware |
| CodeDeploy vs Elastic Beanstalk | Fine-grained control, on-prem support, ECS Blue/Green | Want fully managed platform, don't need deploy customization |
| CodeDeploy vs ECS Rolling Update | Need canary/linear traffic shifting, alarm-based rollback | Simple rolling update is sufficient, least config |

---

## 8. Additional Deep-Dive

### 10 More Tricky Scenario-Based Questions

**Q1:** A company's pipeline deploys to ECS using CodeDeploy Blue/Green. They want to run integration tests against the NEW task set BEFORE any production traffic shifts. How?

**A:** Use the **BeforeAllowTraffic** lifecycle hook in the AppSpec file. This triggers a Lambda function that runs integration tests against the replacement task set (accessible via the test listener). If the Lambda returns "Failed", the deployment rolls back.

**Keywords:** "test BEFORE traffic shifts", "ECS Blue/Green" → BeforeAllowTraffic hook

---

**Q2:** A team uses CodePipeline with a GitHub source. They want the pipeline to trigger ONLY when changes occur in the `/backend` directory, ignoring frontend changes. How?

**A:** Use **CodePipeline V2** with trigger filters — configure a push trigger with file path inclusion filter `/backend/**`. V1 pipelines don't support path-based filtering.

**Keywords:** "trigger only when specific directory changes" → Pipeline V2 trigger filters

---

**Q3:** A company needs to build both ARM and x86 container images from the same CodeBuild project. How?

**A:** Use CodeBuild **batch builds** with a build matrix that specifies different environment types (ARM_CONTAINER and LINUX_CONTAINER). Or use two separate CodeBuild actions in parallel within the same pipeline stage.

**Keywords:** "ARM and x86", "same project/pipeline" → Batch builds with environment matrix

---

**Q4:** CodeDeploy is deploying to an ASG behind an ALB. During Blue/Green deployment, the new (green) instances fail health checks but the deployment still shows "Succeeded". Why?

**A:** The deployment configuration's health check settings might be using **EC2 health checks only** (not ELB health checks). Configure the deployment group to use **ELB health checks** by specifying the target group. Also check that the health check grace period is long enough for the application to start.

**Keywords:** "health checks fail but deployment succeeds" → Wrong health check type configured

---

**Q5:** A company wants to enforce that ALL deployments to production go through CodePipeline — no direct console deployments allowed. How?

**A:** Use an **SCP (Service Control Policy)** or IAM policy with a condition: `"Condition": {"StringNotEquals": {"aws:CalledVia": "codepipeline.amazonaws.com"}}` on the Deny statement for deployment actions. This ensures only CodePipeline can trigger deployments.

Alternatively, use `aws:SourceArn` conditions referencing the pipeline ARN.

**Keywords:** "enforce all deployments through pipeline", "prevent direct deployment" → IAM/SCP conditions on CalledVia

---

**Q6:** A build in CodeBuild takes 25 minutes. 15 minutes is spent downloading dependencies from Maven Central. How can this be optimized with **least effort**?

**A:** Enable **S3 caching** (or local caching if builds run frequently on the same host) in the CodeBuild project configuration, with the cache paths pointing to the Maven local repository (`/root/.m2/**/*`). Subsequent builds reuse cached dependencies.

**Why not VPC endpoint?** Maven Central is an external internet repository, not an AWS service.

**Keywords:** "downloading dependencies takes too long", "optimize build time" → CodeBuild caching

---

**Q7:** A company has 500 EC2 instances across 3 regions. They need to deploy the same application version to all regions within a 1-hour window. What architecture minimizes operational overhead?

**A:** Single CodePipeline with a Deploy stage containing **3 parallel CodeDeploy actions** (one per region, using cross-region actions). Each action deploys to the regional deployment group. Parallel actions ensure simultaneous deployment across regions.

**Keywords:** "3 regions", "within 1-hour window", "minimize overhead" → Cross-region parallel deploy actions in one pipeline

---

**Q8:** A company's CodeBuild creates Docker images and pushes to ECR. Builds intermittently fail with "toomanyrequests: Rate exceeded" from Docker Hub (pulling base images). What's the best fix?

**A:** Use **ECR Public Gallery** or **ECR pull-through cache** for base images instead of Docker Hub. Alternatively, store base images in a private ECR repo. ECR pull-through cache (introduced 2021) automatically caches Docker Hub images in your ECR.

**Keywords:** "Docker Hub rate limit", "toomanyrequests" → ECR pull-through cache or private ECR

---

**Q9:** A pipeline needs to deploy a CloudFormation stack AND wait for the stack to complete before running integration tests. How should this be structured?

**A:** Use CloudFormation deploy action with **action mode "CREATE_UPDATE"** (which waits for stack completion before signaling success to the pipeline). Then place the integration test action (CodeBuild) in the next action group or stage. CodePipeline won't proceed until the CloudFormation action completes.

**Why not "CHANGE_SET_EXECUTE"?** This also works but requires two actions (create change set + execute). CREATE_UPDATE is simpler for most cases.

**Keywords:** "wait for stack to complete", "then run tests" → CloudFormation CREATE_UPDATE mode (blocks until complete) + sequential test stage

---

**Q10:** A company uses CodeDeploy with a minimum healthy hosts setting of 75%. They have 4 instances. A deployment starts but fails on the first instance. What happens?

**A:** With 4 instances and 75% minimum healthy hosts, only **1 instance can be down at a time** (4 × 75% = 3 must be healthy). The first instance fails → that's 3/4 healthy (75%) → meets threshold. CodeDeploy attempts the second instance. If that also fails → 2/4 healthy (50%) < 75% → deployment FAILS and triggers rollback.

**Keywords:** "minimum healthy hosts", "fails on instance" → Calculate the math: total × percentage = minimum that must remain healthy

---

### Compare: CodeDeploy Deployment Strategies

| Aspect | In-Place | Blue/Green (EC2) | Blue/Green (Lambda) | Blue/Green (ECS) |
|--------|----------|-----------------|--------------------|--------------------|
| **Use Case** | Simple update, can tolerate brief downtime per instance | Zero-downtime EC2, easy rollback | Serverless traffic shifting | Container traffic shifting |
| **Limits** | EC2/On-prem only | Requires ASG, creates new instances | Lambda aliases required | Requires ALB/NLB + 2 target groups |
| **Pricing** | Free (EC2) / $0.02 (on-prem) | Free + EC2 instance costs during transition | Free | Free + Fargate/EC2 task costs during transition |
| **HA Model** | MinimumHealthyHosts prevents full outage | Two full environments simultaneously | Version-level isolation | Task-set-level isolation |
| **Exam Tip** | "Update existing instances" | "Zero downtime" + "easy rollback" + "EC2/ASG" | "Gradual shift" + "Lambda" | "Canary" + "ECS" + "Fargate" |

### When Does SAP-C02 Expect CodeDeploy over Other Options?

**Pick CodeDeploy when:**
- "Deploy to EC2 instances with rollback capability"
- "Canary deployment to Lambda"
- "Blue/Green for ECS"
- "Deploy to on-premises servers"
- "Traffic shifting with automatic rollback on alarms"
- "Minimize deployment risk"

**Pick ECS Rolling Update (not CodeDeploy) when:**
- "Simple container update"
- "Least configuration"
- "Don't need canary/linear"

**Pick Elastic Beanstalk when:**
- "Least operational overhead for web application"
- "Don't need fine-grained deployment control"
- "All-in-one platform (build + deploy + infra)"

**Pick CloudFormation Deploy Action when:**
- "Infrastructure changes" (not application code)
- "Update stack resources"
- "Create new infrastructure as part of pipeline"

### Gotcha Differences

| Feature | CodePipeline V1 | CodePipeline V2 |
|---------|----------------|----------------|
| Trigger filtering | Not supported (any change triggers) | Path, branch, tag, PR filters |
| Pricing | $1/pipeline/month flat | $0.002/action execution minute (after 100 free) |
| Pipeline types | Only "V1" | Supports "V2" with enhanced features |
| Git tags as trigger | No | Yes |
| Connections | CodeStar Connections | Same but more providers |

| Feature | CodeBuild | GitHub Actions |
|---------|-----------|----------------|
| Integration | Native AWS IAM roles | Needs OIDC federation or access keys |
| VPC access | Native (VPC config) | Needs self-hosted runners in VPC |
| Caching | S3/local cache | Built-in action cache |
| Exam relevance | Always preferred for AWS-native solutions | Not an answer option |

### Decision Tree: CI/CD Service Selection

```
Need to ORCHESTRATE a multi-stage release?
  └── YES → CodePipeline
       ├── Need approval gates? → Manual Approval action
       ├── Need cross-account? → Cross-account IAM + CMK
       └── Need cross-region? → Cross-region actions

Need to BUILD/TEST code?
  └── YES → CodeBuild
       ├── Need private resource access? → VPC mode + NAT GW
       ├── Need to speed up builds? → Caching (S3 or local)
       └── Need multi-arch builds? → Batch builds

Need to DEPLOY with traffic control?
  └── YES → CodeDeploy
       ├── Target is Lambda? → Blue/Green (Canary/Linear/AllAtOnce)
       ├── Target is ECS? → Blue/Green with ALB + target groups
       ├── Target is EC2 + zero downtime? → Blue/Green
       ├── Target is EC2 + simple update OK? → In-Place
       └── Target is on-premises? → In-Place (only option for on-prem Blue/Green is complex)

Need to DEPLOY infrastructure (CloudFormation)?
  └── YES → CloudFormation deploy action in CodePipeline (not CodeDeploy)
```
