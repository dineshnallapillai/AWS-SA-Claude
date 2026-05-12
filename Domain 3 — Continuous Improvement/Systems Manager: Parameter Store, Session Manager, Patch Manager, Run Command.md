# Systems Manager: Parameter Store, Session Manager, Patch Manager, Run Command

## 1. Core Concepts & Theory

### AWS Systems Manager Overview
- **Unified management service** for AWS and on-premises infrastructure
- **Agent-based**: SSM Agent installed on managed instances (pre-installed on most Amazon Linux/Windows AMIs)
- **Managed Instances**: EC2 instances or on-premises servers registered with SSM
- **Hybrid Activation**: Register on-premises servers as managed instances (prefix `mi-`)
- **IAM Instance Profile**: EC2 instances need `AmazonSSMManagedInstanceCore` policy (or equivalent)
- **Free for most features** (except advanced Parameter Store and on-prem advanced usage)

**SSM Agent:**
- Pre-installed on: Amazon Linux 2, Amazon Linux 2023, Windows Server 2016+, Ubuntu 16.04+
- Communicates outbound to SSM endpoints (HTTPS/443) — no inbound ports required
- **VPC Endpoints** supported: `ssm`, `ssmmessages`, `ec2messages` (for private subnets without internet)
- Agent auto-updates available (via SSM itself)

---

### Parameter Store

**Overview:**
- **Centralized, hierarchical configuration store** for secrets, config data, connection strings
- **Secure**: Supports encryption via KMS (SecureString type)
- **Versioned**: All parameter changes tracked with version numbers
- **Integrated**: Native integration with CloudFormation, ECS, Lambda, CodeBuild, etc.

**Parameter Types:**
| Type | Description | Use Case |
|------|-------------|----------|
| **String** | Plain text value | Config values, AMI IDs, endpoints |
| **StringList** | Comma-separated strings | List of values |
| **SecureString** | Encrypted with KMS | Passwords, API keys, connection strings |

**Parameter Tiers:**
| Feature | Standard | Advanced |
|---------|----------|----------|
| Max parameters | 10,000 | 100,000 |
| Max value size | 4 KB | 8 KB |
| Parameter policies | No | Yes (expiration, notification) |
| Pricing | Free | $0.05/advanced parameter/month |
| Higher throughput | No | Yes (up to 1,000 TPS) |

**Hierarchy & Paths:**
```
/myapp/prod/db/connection-string
/myapp/prod/db/password
/myapp/dev/db/connection-string
```
- Organize with paths (up to 15 levels deep)
- `GetParametersByPath` — retrieve all parameters under a path
- IAM policies can restrict access by path: `arn:aws:ssm:region:account:parameter/myapp/prod/*`

**Parameter Policies (Advanced tier only):**
- **Expiration**: Auto-delete parameter after date (or notify before expiration)
- **ExpirationNotification**: EventBridge event before parameter expires
- **NoChangeNotification**: Alert if parameter hasn't been updated within a period
- Use case: Force rotation of credentials by expiring old parameters

**Parameter Store vs Secrets Manager:**
| Feature | Parameter Store | Secrets Manager |
|---------|----------------|-----------------|
| Pricing | Free (standard) / $0.05/param (advanced) | $0.40/secret/month + $0.05/10K API calls |
| Rotation | Manual (no built-in) | Automatic rotation (Lambda-based) |
| Cross-account sharing | Via RAM or IAM | Native cross-account access |
| Max size | 4 KB (standard) / 8 KB (advanced) | 64 KB |
| Versioning | Yes | Yes |
| CloudFormation dynamic ref | `{{resolve:ssm:name}}` | `{{resolve:secretsmanager:name}}` |
| Best for | Config values, non-rotating secrets | Database credentials, API keys needing rotation |

**Key Limits:**
| Limit | Value |
|-------|-------|
| Standard parameters per account/region | 10,000 |
| Advanced parameters per account/region | 100,000 |
| Max parameter value (standard) | 4 KB |
| Max parameter value (advanced) | 8 KB |
| Max path depth | 15 levels |
| Throughput (standard) | 40 TPS (default), up to 1,000 TPS (adjustable) |
| Throughput (advanced) | Up to 1,000 TPS |
| Parameter history | All versions retained |

---

### Session Manager

**Overview:**
- **Browser-based or CLI-based shell access** to managed instances
- **No SSH keys, no bastion hosts, no inbound ports required**
- Traffic flows through SSM endpoints (HTTPS/443 outbound)
- **Fully auditable**: All session activity logged

**Key Features:**
- **Port forwarding**: Forward local port to instance port (access RDS, internal apps)
- **Tunneling**: SSH-over-SSM (ProxyCommand in SSH config)
- **Logging**: Session data logged to S3 and/or CloudWatch Logs
- **KMS encryption**: Encrypt session data in transit (end-to-end beyond TLS)
- **Idle timeout**: Configurable session timeout
- **Shell profiles**: Customize shell (Linux: bash/sh, Windows: PowerShell)
- **Run As**: Specify OS user to run session as (Linux)
- **Concurrent sessions**: Multiple sessions per instance

**Access Control:**
- IAM policies control who can start sessions to which instances
- Condition keys: `ssm:resourceTag/Environment`, `ssm:SessionDocumentAccessCheck`
- **Session documents**: Define session type and permissions
- No need for security group inbound rules (unlike SSH on port 22)

**Network Requirements (Private Subnet):**
- VPC endpoints needed: `ssm`, `ssmmessages`, `ec2messages`
- OR NAT Gateway for internet access to public SSM endpoints
- No inbound security group rules needed (all communication is outbound HTTPS)

**Session Manager vs SSH/RDP:**
| Feature | Session Manager | SSH/RDP |
|---------|----------------|---------|
| Inbound ports | None (outbound 443 only) | 22 (SSH) or 3389 (RDP) |
| Key management | None (IAM-based) | Key pairs, rotation burden |
| Bastion hosts | Not needed | Often required |
| Audit trail | Full (CloudTrail + session logs) | Limited (OS logs only) |
| MFA | Via IAM | Separate configuration |
| Cross-platform | Yes (browser, CLI, console) | Platform-specific clients |

---

### Patch Manager

**Overview:**
- **Automate OS and application patching** across managed instances
- Supports: Windows, Amazon Linux, Ubuntu, RHEL, SUSE, CentOS, Debian, Oracle Linux, macOS
- **Patch Baselines**: Define which patches are approved/rejected
- **Patch Groups**: Group instances for different patching schedules
- **Maintenance Windows**: Schedule patching during specific time windows

**Core Components:**

**Patch Baselines:**
- **AWS-managed baselines**: Pre-defined per OS (e.g., `AWS-DefaultPatchBaseline` for Windows)
- **Custom baselines**: Define approval rules (auto-approve after X days), approved/rejected patches
- **Approval rules**: Classify by severity (Critical, Important, etc.) and auto-approve after N days
- **Patch sources**: Configure custom patch repositories (Linux)

**Patch Groups:**
- Tag instances with `Patch Group` tag (e.g., `Patch Group: Production-Servers`)
- Associate patch group with a patch baseline
- One patch group → one baseline (but one baseline can serve multiple groups)
- Allows different patching policies per group (dev gets patches immediately, prod waits 7 days)

**Maintenance Windows:**
- Define schedule (cron/rate), duration, cutoff time
- Register targets (instances, resource groups)
- Register tasks (Run Command, Automation, Lambda, Step Functions)
- **Cutoff**: Stop starting new tasks N hours before window end
- Max duration: 24 hours
- Max tasks per window: 20

**Patch Operations:**
- **Scan**: Check compliance (what's missing) — no changes made
- **Install**: Apply approved patches (with optional reboot)
- **Patch compliance**: Reported to SSM Compliance dashboard
- **Pre/Post patch hooks**: Run scripts before/after patching (via SSM documents)

**Patching Workflow:**
```
1. Define Patch Baseline (rules for which patches to approve)
2. Create Patch Group (tag instances)
3. Associate Baseline with Patch Group
4. Create Maintenance Window (schedule)
5. Register "Run Command: AWS-RunPatchBaseline" as window task
6. Patch Manager scans/installs during window
7. Results reported to Compliance dashboard
```

**Key Facts:**
- `AWS-RunPatchBaseline` document: The primary SSM document for patching (Scan or Install mode)
- Reboot behavior: Configurable (NoReboot, RebootIfNeeded)
- **Patch compliance data**: Visible in SSM Compliance, exportable to S3 via Resource Data Sync
- **Integration with AWS Config**: Config rule `ec2-managedinstance-patch-compliance-status-check`

---

### Run Command

**Overview:**
- **Remotely execute commands** on managed instances at scale (no SSH needed)
- Uses **SSM Documents** (JSON/YAML) to define commands
- **No inbound ports required** — agent polls SSM service

**Key Features:**
- **Rate control**: Limit concurrent executions (e.g., 10 at a time, or 25%)
- **Error threshold**: Stop execution if error count/percentage exceeds threshold
- **Targeting**: By instance IDs, tags, resource groups, or all instances
- **Notifications**: SNS integration for command status
- **Output**: S3 bucket and/or CloudWatch Logs for command output
- **Timeout**: Per-command timeout (default 3600 seconds for most documents)
- **CloudWatch Events/EventBridge**: Triggered on command status changes

**Common SSM Documents:**
| Document | Purpose |
|----------|---------|
| `AWS-RunShellScript` | Run shell commands on Linux |
| `AWS-RunPowerShellScript` | Run PowerShell on Windows |
| `AWS-RunPatchBaseline` | Scan/Install patches |
| `AWS-ConfigureAWSPackage` | Install/uninstall AWS packages (e.g., CloudWatch Agent) |
| `AWS-UpdateSSMAgent` | Update SSM Agent |
| `AWS-RunAnsiblePlaybook` | Run Ansible playbooks |
| `AWS-ApplyChefRecipes` | Apply Chef recipes |

**Targeting Strategies:**
- **Instance IDs**: Specific instances (small scale)
- **Tags**: e.g., `Environment=Production` (dynamic, recommended)
- **Resource Groups**: SSM/resource group membership
- **Rate Control**: `MaxConcurrency` (10, 25%, etc.) + `MaxErrors` (stop if too many fail)

**Run Command vs SSH/Remote execution:**
| Aspect | Run Command | SSH + Script |
|--------|-------------|-------------|
| Scale | Thousands of instances simultaneously | Manual or requires orchestration |
| Audit | Full CloudTrail + command history | OS-level logs only |
| Access | IAM-based, no keys | SSH key management |
| Output | S3 + CloudWatch Logs (centralized) | Scattered across instances |
| Error handling | Rate control + error thresholds | Manual |

---

### Other Key SSM Capabilities (Exam-Relevant)

**State Manager:**
- Maintain desired state configuration (auto-remediate drift)
- Associates: Define configuration + target + schedule
- Use case: Ensure CloudWatch Agent is always running, enforce security settings

**Automation:**
- Execute multi-step workflows (runbooks)
- Pre-built runbooks: Restart instance, create AMI, patch and reboot, remediate Config findings
- Supports approvals, branching, looping
- **Change Manager**: Approval workflow for automation (ITSM-style change management)

**Inventory:**
- Collect metadata from managed instances (installed software, network config, Windows updates, services)
- **Resource Data Sync**: Export inventory data to S3 (query with Athena)
- Integrate with AWS Config for compliance

**Distributor:**
- Package and distribute software to managed instances
- Create custom packages or use AWS-provided (e.g., CloudWatch Agent, AWS Inspector agent)
- Version management for packages

---

## 2. Design Patterns & Best Practices

### When to Use

| Feature | Use When | Don't Use When |
|---------|----------|----------------|
| Parameter Store | Configuration management, non-rotating secrets, CloudFormation references | Need automatic rotation (use Secrets Manager) |
| Session Manager | Shell access to instances, port forwarding, compliance-required auditing | Need full desktop (use Fleet Manager or direct RDP) |
| Patch Manager | Automated OS patching at scale | Application-level deployments (use CodeDeploy) |
| Run Command | Ad-hoc or scheduled commands across fleet | Complex multi-step workflows (use Automation) |
| State Manager | Enforce desired state continuously | One-time commands (use Run Command) |
| Automation | Multi-step operational runbooks | Simple single commands (use Run Command) |

### Anti-Patterns
- **Don't store large secrets (>8 KB) in Parameter Store** — use Secrets Manager (64 KB limit) or S3
- **Don't open port 22/3389 when Session Manager is available** — eliminates attack surface
- **Don't patch production without testing** — use dev/staging patch groups first, delay prod approval
- **Don't use Run Command for deployments** — use CodeDeploy (proper deployment strategies, rollback)
- **Don't put actual secret values in CloudFormation templates** — use dynamic references to Parameter Store/Secrets Manager

### Architectural Patterns

**Secure Access Without Bastion:**
```
Developer → IAM Auth (MFA) → Session Manager → Private EC2 instance
                                              → Port Forwarding to RDS (local:3306 → RDS:3306)
```
- No bastion host, no SSH keys, no inbound security group rules
- Full audit trail in CloudWatch Logs + S3

**Centralized Configuration Management:**
```
Parameter Store:
  /shared/prod/db-endpoint → "prod-db.cluster.amazonaws.com"
  /shared/prod/api-key → SecureString (KMS encrypted)
  /app1/prod/feature-flag → "enabled"
  /app2/prod/batch-size → "100"

Applications → GetParameter/GetParametersByPath → Use config at runtime
CloudFormation → {{resolve:ssm:/shared/prod/db-endpoint}} → Deploy-time resolution
Lambda → SDK GetParameter → Runtime resolution
```

**Automated Patching Pipeline:**
```
Week 1: Patch Baseline auto-approves Critical patches
Week 1: Maintenance Window patches DEV instances (Patch Group: Dev)
Week 2: Scan STAGING → if compliant, patch STAGING
Week 3: Scan PRODUCTION → patch PRODUCTION (with pre/post hooks)

Compliance → AWS Config rule → Non-compliant alert → SNS notification
```

**Fleet Management:**
```
SSM Inventory → Collect installed software, OS versions
             → Resource Data Sync → S3 → Athena queries
             → Config rule: ec2-managedinstance-association-compliance-status-check

Run Command → Deploy configurations at scale
State Manager → Ensure CloudWatch Agent running (auto-remediate if stopped)
Automation → Runbook: Create Golden AMI → Update Launch Template → Instance Refresh
```

**Cross-Account Parameter Sharing:**
```
Shared Services Account:
  Parameter Store: /shared/prod/db-endpoint (shared via RAM or resource policy)

Application Accounts:
  Access shared parameters via cross-account IAM role
  OR use AWS RAM to share parameters
```

### Well-Architected Alignment

| Pillar | Practice |
|--------|----------|
| Reliability | Patch Manager ensures security patches applied, State Manager enforces desired state |
| Security | Session Manager eliminates SSH/RDP exposure, Parameter Store SecureString for secrets, KMS encryption |
| Cost | SSM is free (most features), eliminates bastion host cost, Parameter Store standard tier is free |
| Performance | Fast parameter retrieval (cached in SDK), Run Command parallel execution |
| Ops Excellence | Automation runbooks for operational tasks, Maintenance Windows for scheduled operations |

### Integration Points
- **CloudFormation**: Dynamic references (`{{resolve:ssm:...}}`)
- **CloudWatch**: Session logs, Run Command output, custom metrics via Agent
- **KMS**: SecureString encryption, session data encryption
- **S3**: Command output storage, session logs, inventory data sync, patch logs
- **SNS**: Notifications for command completion, maintenance window results
- **EventBridge**: Parameter change events, automation status, command status
- **IAM**: Access control for all SSM features
- **VPC Endpoints**: Private access without internet (3 endpoints: ssm, ssmmessages, ec2messages)
- **Config**: Compliance checking for patch and association compliance
- **Organizations**: Delegated administrator, organization-wide inventory

---

## 3. Security & Compliance

### IAM & Access Control

**SSM Permissions Model:**
```
Instance needs:
  - IAM Instance Profile with AmazonSSMManagedInstanceCore (or equivalent)
  - Allows: ssm:*, ec2messages:*, ssmmessages:* (to communicate with SSM service)

User/Admin needs:
  - ssm:StartSession (for Session Manager)
  - ssm:SendCommand (for Run Command)
  - ssm:GetParameter/PutParameter (for Parameter Store)
  - ssm:StartAutomationExecution (for Automation)
```

**Fine-Grained Access:**
- Restrict sessions to specific instances: Condition on `ssm:resourceTag/Environment`
- Restrict commands to specific documents: Condition on `ssm:DocumentName`
- Restrict parameter access by path: Resource `arn:aws:ssm:*:*:parameter/app/prod/*`
- Restrict KMS usage for SecureString: Condition on `kms:ViaService`

**Session Manager IAM Example:**
```json
{
  "Effect": "Allow",
  "Action": "ssm:StartSession",
  "Resource": "arn:aws:ec2:*:*:instance/*",
  "Condition": {
    "StringEquals": {
      "ssm:resourceTag/Environment": "Development"
    }
  }
}
```

### Encryption

**Parameter Store:**
- SecureString: Encrypted/decrypted with KMS CMK (default `aws/ssm` key or custom CMK)
- Custom CMK allows cross-account access (share key policy)
- Encryption in transit: All API calls over TLS

**Session Manager:**
- **Session data encryption**: Optional KMS encryption (end-to-end, beyond TLS)
- **S3 log encryption**: SSE-S3 or SSE-KMS
- **CloudWatch Logs encryption**: Log group KMS encryption

**Run Command:**
- Output encrypted in S3 (SSE-S3/SSE-KMS)
- Parameters passed to commands: Encrypted in transit

### Auditing & Logging

**CloudTrail:**
- All SSM API calls logged (PutParameter, GetParameter, SendCommand, StartSession, etc.)
- `GetParameter` with `WithDecryption` flag logged (you can audit who decrypted secrets)

**Session Manager Logging:**
- Session transcript → S3 bucket (who ran what commands)
- Session transcript → CloudWatch Logs (real-time)
- **Streaming**: Session data streamed in real-time (not just at end)
- Force logging: Configure in Session Manager preferences (deny sessions if logging fails)

**Run Command Logging:**
- Command output → S3 (per-instance output)
- Command output → CloudWatch Logs
- Command status → EventBridge events

### Compliance Patterns
- **Eliminate SSH/RDP**: Enforce via SCP (deny `ec2:AuthorizeSecurityGroupIngress` on port 22/3389)
- **Mandatory session logging**: Session Manager preferences → require S3/CloudWatch logging
- **Patch compliance**: Patch Manager + Config rule → non-compliant instances flagged
- **Secret rotation**: Parameter policies (expiration notification) or use Secrets Manager
- **Least privilege**: Restrict parameters by path, sessions by tag, commands by document

---

## 4. Cost Optimization

### Pricing

| Feature | Price |
|---------|-------|
| Parameter Store (Standard) | **Free** (up to 10,000 parameters) |
| Parameter Store (Advanced) | $0.05/parameter/month |
| Parameter Store (Higher throughput) | $0.05/10,000 API interactions |
| Session Manager | **Free** |
| Run Command | **Free** |
| Patch Manager | **Free** (EC2 instances) |
| Patch Manager (on-premises) | Free for basic, advanced features additional |
| State Manager | **Free** |
| Automation | **Free** (standard steps) |
| Maintenance Windows | **Free** |
| Inventory | **Free** |
| OpsCenter | Free (OpsItems), charges for OpsItem resolution with Incident Manager |

### Cost-Saving Strategies
1. **Use Standard tier parameters** unless you need >4 KB, >10K params, or parameter policies
2. **Session Manager eliminates bastion hosts** — saves instance cost + management overhead
3. **Patch Manager eliminates third-party patching tools** (WSUS, SCCM licensing)
4. **Run Command replaces SSH jump-box** access patterns
5. **Parameter Store over Secrets Manager** for non-rotating secrets (free vs $0.40/secret/month)
6. **Use path-based GetParametersByPath** instead of individual GetParameter calls (fewer API calls)

### Cost Traps
- **Advanced tier auto-upgrade**: If you exceed 10,000 parameters, account auto-upgrades to advanced (charges apply)
- **Higher throughput charges**: If you exceed default TPS, higher throughput charges ($0.05/10K interactions)
- **Forgetting on-premises**: On-prem managed instances may incur charges for advanced features
- **Automation with Lambda steps**: Lambda execution costs for automation steps using Lambda

---

## 5. High Availability, Disaster Recovery & Resilience

### Built-in HA
- SSM is a **fully managed regional service** — multi-AZ within a region
- Parameter Store: Highly available within a region
- Session Manager: Regional (sessions connected to instances in that region)
- Run Command: Regional (executed in the region where instances reside)

### DR Considerations

**Parameter Store:**
- Parameters are **regional** (not automatically replicated cross-region)
- DR strategy: (1) CloudFormation/CDK to recreate parameters in DR region (2) Custom Lambda to sync parameters cross-region (3) Store parameter values in source control
- For truly global config: Consider DynamoDB Global Tables or custom replication

**Patch Manager:**
- Patch baselines are regional
- DR: Recreate baselines via CloudFormation in DR region
- Maintenance Windows: Recreate in DR region

**Run Command/Session Manager:**
- Regional — instances must be managed in their region
- DR instances automatically managed if SSM Agent is installed and IAM role is configured

### Resilience Patterns
- **VPC Endpoints**: If NAT Gateway fails, instances lose SSM connectivity unless VPC endpoints exist
- **Agent auto-update**: Keeps SSM Agent current (avoids connectivity issues from outdated agents)
- **State Manager**: Auto-remediates if agent stops or configuration drifts
- **Fallback**: If SSM is unreachable, instances continue running (SSM is control plane, not data plane)

---

## 6. Exam-Focused Section

### Straightforward Questions (7 examples)

**Q1:** A company wants to securely store database credentials and reference them in CloudFormation templates without hardcoding. What should they use?
**A:** **SSM Parameter Store (SecureString)** with dynamic reference: `{{resolve:ssm-secure:parameter-name:version}}`. For automatic rotation, use Secrets Manager with `{{resolve:secretsmanager:...}}`.

**Q2:** A security team wants shell access to private EC2 instances without opening inbound ports or using bastion hosts. What should they use?
**A:** **Session Manager** — requires only outbound HTTPS (443) from the instance to SSM endpoints (or VPC endpoints). No inbound rules, no SSH keys, full audit logging.

**Q3:** A company needs to apply security patches to 500 EC2 instances during a maintenance window. What SSM feature handles this?
**A:** **Patch Manager** with a **Maintenance Window**. Define a patch baseline, create patch groups (tag instances), and schedule the `AWS-RunPatchBaseline` document in a maintenance window.

**Q4:** How can a company run a shell script on all EC2 instances tagged `Environment=Production` without SSH access?
**A:** Use **Run Command** with the `AWS-RunShellScript` document, targeting instances by tag `Environment=Production`. Set rate control for safe execution.

**Q5:** A company needs to ensure the CloudWatch Agent is always running on all EC2 instances. If it stops, it should be automatically restarted. What SSM feature achieves this?
**A:** **State Manager** association with the `AWS-ConfigureAWSPackage` document (or custom document) on a schedule. State Manager continuously enforces the desired state.

**Q6:** What VPC endpoints are needed for Session Manager to work in a private subnet with no internet access?
**A:** Three VPC endpoints: `com.amazonaws.region.ssm`, `com.amazonaws.region.ssmmessages`, and `com.amazonaws.region.ec2messages`.

**Q7:** A company stores configuration in Parameter Store using paths like `/app/prod/config`. How can they retrieve all parameters under `/app/prod/` in a single API call?
**A:** Use `GetParametersByPath` with the path `/app/prod/` and set `Recursive=true`. This returns all parameters under that path hierarchy.

---

### Tricky / Scenario-Based Questions (7 examples)

**Q1:** A company uses Parameter Store for database credentials (SecureString with default `aws/ssm` key). They want to share these parameters with another AWS account. The other account gets "AccessDeniedException" even with correct IAM permissions. Why?

**A:** The default `aws/ssm` AWS-managed KMS key **cannot be shared cross-account**. They must re-encrypt the parameters using a **customer-managed CMK** and update the key policy to allow the other account's role to decrypt. Cross-account SecureString access requires a customer CMK.

**Why wrong answers are wrong:**
- "Add a resource policy to the parameter" — Parameter Store doesn't have resource policies (pre-RAM)
- "Use IAM in the source account" — IAM alone doesn't solve KMS cross-account for AWS-managed keys
- "Increase the parameter tier to Advanced" — tier doesn't affect cross-account access
- **Keywords:** "cross-account", "SecureString", "AccessDeniedException" → Must use customer-managed CMK

---

**Q2:** A company has 1,000 instances and wants to patch them in batches of 100 (10% at a time), stopping if more than 5% of any batch fails. How?

**A:** Configure Run Command (via Maintenance Window task) with `AWS-RunPatchBaseline` and set **rate control**: `MaxConcurrency: 10%` (100 instances at a time) and `MaxErrors: 5%` (stop if >5% fail in any batch). This ensures controlled rollout with automatic halt on excessive failures.

**Why wrong answers are wrong:**
- "Use Automation with approval steps" — Automation is for multi-step workflows, not batch execution control
- "Create 10 separate maintenance windows" — unnecessary complexity; rate control handles batching natively
- "Use CodeDeploy" — CodeDeploy is for application deployment, not OS patching
- **Keywords:** "batches", "stop if failures exceed threshold" → Run Command rate control (MaxConcurrency + MaxErrors)

---

**Q3:** A company has EC2 instances in private subnets. After removing the NAT Gateway (cost savings), Session Manager stops working. They add VPC endpoints for `ssm` and `ssmmessages` but it still doesn't work. What's missing?

**A:** The **`ec2messages`** VPC endpoint is also required. Session Manager needs all three endpoints: `ssm` (API calls), `ssmmessages` (session data channel), `ec2messages` (message delivery channel). Missing any one breaks connectivity.

**Why wrong answers are wrong:**
- "Instance profile is incorrect" — worked before NAT removal, so IAM is fine
- "Security group blocks outbound" — VPC endpoint access requires security group on the endpoint allowing inbound from instances (check endpoint security group)
- "SSM Agent needs updating" — unrelated to endpoint connectivity
- **Keywords:** "private subnet", "VPC endpoints added but not working", "Session Manager" → Missing `ec2messages` endpoint (or endpoint security group misconfigured)

---

**Q4:** A company uses Patch Manager and finds that dev instances get patched correctly, but production instances remain non-compliant despite being in the same maintenance window. Investigation shows production instances have a different `Patch Group` tag. What's likely wrong?

**A:** The production **Patch Group** is associated with a **different patch baseline** (or not associated with any baseline at all). Each patch group maps to one baseline. If the production patch group isn't explicitly associated with a baseline, it uses the default baseline which may have different approval rules. Fix: Associate the production patch group with the correct custom baseline.

**Why wrong answers are wrong:**
- "Maintenance window targeting is wrong" — question says instances are in the same window
- "SSM Agent is outdated on production" — would show execution failures, not compliance mismatches
- "IAM permissions differ" — would prevent command execution entirely
- **Keywords:** "different Patch Group tag", "production non-compliant" → Patch Group ↔ Baseline association mismatch

---

**Q5:** A company stores 12,000 parameters in Parameter Store. They notice charges appearing that weren't there before. They haven't changed their usage pattern. What happened?

**A:** Parameter Store **automatically upgrades to Advanced tier** when you exceed 10,000 parameters. Advanced tier costs $0.05/parameter/month. The first 10,000 stay on standard tier but parameters 10,001+ are advanced. To avoid costs: delete unused parameters or use a different account/region to stay under 10,000.

**Why wrong answers are wrong:**
- "Higher throughput was enabled" — that's a separate charge, not triggered by parameter count
- "API call volume increased" — standard API calls are free (only higher throughput has charges)
- "KMS charges for SecureString" — KMS charges exist but aren't new (and are separate from SSM charges)
- **Keywords:** "12,000 parameters", "new charges" → Auto-upgrade to Advanced tier above 10,000

---

**Q6:** A team uses Run Command to execute a script on 200 instances. The command succeeds on 195 instances but fails on 5. They can't see the failure details because the output is truncated (>2500 characters). How to get the full output?

**A:** Configure Run Command to send output to **S3** (specify `OutputS3BucketName` and `OutputS3KeyPrefix`). The console truncates output at 2,500 characters, but S3 stores the complete output per instance. Alternatively, direct output to **CloudWatch Logs** for searchable full output.

**Why wrong answers are wrong:**
- "Increase timeout" — timeout isn't the issue; output is just truncated in console
- "Use Session Manager to check manually" — doesn't scale for debugging across many instances
- "Re-run the command" — same truncation will occur unless output destination is configured
- **Keywords:** "output truncated", "can't see failure details" → Configure S3 or CloudWatch Logs output

---

**Q7:** A company wants developers to use Session Manager for database troubleshooting via port forwarding, but ONLY to development databases (not production). How to enforce this?

**A:** Use IAM policies with **conditions** on Session Manager. Restrict `ssm:StartSession` to instances tagged `Environment=Development`. For port forwarding specifically, use the **Session document** `AWS-StartPortForwardingSessionToRemoteHost` and restrict it with IAM conditions on target host/port. Additionally, use `ssm:resourceTag` conditions.

**Why wrong answers are wrong:**
- "Use security groups" — Session Manager bypasses traditional network access controls (traffic goes through SSM)
- "Separate VPCs" — doesn't prevent Session Manager access if IAM allows it
- "Disable port forwarding globally" — too broad (blocks legitimate dev use)
- **Keywords:** "restrict Session Manager", "only development", "port forwarding" → IAM conditions on resource tags + session documents

---

### Common Exam Traps & Pitfalls

1. **Three VPC endpoints needed** for SSM in private subnets: `ssm`, `ssmmessages`, `ec2messages`. Missing one = broken functionality.

2. **Parameter Store is FREE (standard tier)** — Secrets Manager costs $0.40/secret/month. Use Parameter Store for non-rotating secrets.

3. **Parameter Store does NOT auto-rotate** — only Secrets Manager has built-in rotation. Parameter Store can notify (via parameter policies) but not rotate.

4. **aws/ssm KMS key can't be shared cross-account** — must use customer-managed CMK for cross-account SecureString.

5. **Session Manager requires IAM Instance Profile** — even if the instance has network access, without the right IAM role, SSM Agent can't register.

6. **Run Command ≠ deployment** — Run Command runs scripts, but it doesn't have deployment strategies (canary, blue/green, rollback). Use CodeDeploy for deployments.

7. **Patch Group tag is case-sensitive** and must exactly match the association (`Patch Group` not `patch group` or `PatchGroup`).

8. **Maintenance Windows are regional** — you need one per region for multi-region patching.

9. **Standard parameter throughput is 40 TPS** by default — if your application hammers Parameter Store, you'll get throttled. Request higher throughput (paid) or cache values.

10. **Session Manager works on on-premises too** — not just EC2 (via Hybrid Activation).

---

## 7. Cheat Sheet

### Must-Know Facts

| Fact | Detail |
|------|--------|
| SSM Agent pre-installed | Amazon Linux 2/2023, Windows 2016+ |
| VPC endpoints needed | `ssm`, `ssmmessages`, `ec2messages` |
| Parameter Store free limit | 10,000 standard parameters |
| Standard parameter max size | 4 KB |
| Advanced parameter max size | 8 KB |
| SecureString cross-account | Must use customer-managed CMK |
| Session Manager ports | None inbound; outbound 443 only |
| Session Manager logging | S3 + CloudWatch Logs |
| Patch Group tag | Case-sensitive: `Patch Group` |
| Run Command rate control | MaxConcurrency + MaxErrors |
| Run Command output limit (console) | 2,500 characters |
| Maintenance Window max duration | 24 hours |
| Parameter Store vs Secrets Manager | No rotation vs automatic rotation |
| SSM pricing | Free (most features) |
| Secrets Manager pricing | $0.40/secret/month |

### Decision Flowchart

```
"Store configuration values" → Parameter Store (String)
"Store encrypted secrets without rotation" → Parameter Store (SecureString)
"Store encrypted secrets WITH automatic rotation" → Secrets Manager
"Shell access without SSH/bastion" → Session Manager
"Access RDS from laptop through private instance" → Session Manager port forwarding
"Patch OS at scale on schedule" → Patch Manager + Maintenance Window
"Run ad-hoc commands across fleet" → Run Command
"Ensure desired state continuously" → State Manager
"Multi-step operational workflow" → Automation
"Collect installed software inventory" → Inventory
"Reference secret in CloudFormation" → {{resolve:ssm-secure:...}} or {{resolve:secretsmanager:...}}
"Share parameter cross-account" → Customer-managed CMK + cross-account IAM
```

---

## 8. Additional Deep-Dive

### 10 More Tricky Scenario-Based Questions

**Q1:** A company has 5,000 EC2 instances. They need to update the CloudWatch Agent configuration on all instances without SSH and with minimal risk. How?

**A:** Store the new CloudWatch Agent configuration in **Parameter Store**. Use **State Manager** association with `AmazonCloudWatch-ManageAgent` document targeting all instances. State Manager applies the configuration on schedule, and the agents fetch config from Parameter Store. Rate control ensures gradual rollout.

**Keywords:** "update config at scale", "minimal risk", "no SSH" → State Manager + Parameter Store

---

**Q2:** A company uses Parameter Store to store database endpoints. They deploy the same application in 4 regions. Each region has a different database endpoint. How should they structure this so the application code doesn't need region-specific logic?

**A:** Use the **same parameter path** in each region (e.g., `/app/prod/db-endpoint`) with different values per region. The application always calls `GetParameter("/app/prod/db-endpoint")` — the SSM API returns the value for the current region automatically (since Parameter Store is regional). No code changes needed.

**Keywords:** "same app multiple regions", "different config per region" → Same parameter path, different values per region (SSM is regional)

---

**Q3:** A company runs a maintenance window for patching every Sunday 2-6 AM. A critical zero-day vulnerability is announced on Wednesday. They can't wait until Sunday. What's the fastest way to patch all instances?

**A:** Run `AWS-RunPatchBaseline` via **Run Command** immediately (outside the maintenance window). Run Command doesn't require a maintenance window — it's on-demand. Set the operation to `Install` and target all relevant instances with rate control for safety.

**Keywords:** "can't wait for maintenance window", "patch immediately" → Run Command (on-demand, no window needed)

---

**Q4:** A company's Session Manager sessions are being used to exfiltrate data. They want to detect and prevent unauthorized data transfer during sessions. What controls can they add?

**A:** (1) Enable **session logging to S3/CloudWatch** — review session transcripts for suspicious commands. (2) Use **Session document constraints** — define allowed commands or interactive session types. (3) Implement **session logging with KMS encryption** + alert on specific patterns via metric filters. (4) Restrict IAM to specific instances (limit blast radius). (5) Use **AWS Config** to ensure all sessions are logged.

**Keywords:** "data exfiltration via Session Manager" → Session logging + document constraints + IAM restrictions

---

**Q5:** A company has a Lambda function that reads parameters from Parameter Store on every invocation (10,000 invocations/second). They're getting throttled. What's the fix?

**A:** (1) **Cache parameters** in the Lambda execution environment (outside handler — reused across warm invocations). Use the **AWS Parameters and Secrets Lambda Extension** (sidecar that caches parameters with configurable TTL). (2) Request **higher throughput** for Parameter Store (paid: $0.05/10K interactions beyond default). (3) Increase cache TTL to reduce API calls.

**Keywords:** "Lambda", "Parameter Store throttling", "high invocation rate" → Lambda extension caching + higher throughput

---

**Q6:** A company uses Automation to create a Golden AMI weekly (install patches, configure software, create AMI). They want the automation to stop and wait for manual approval before the AMI is shared with production accounts. How?

**A:** Add an **`aws:approve`** step in the Automation document. This pauses execution and sends notification to approvers (via SNS). Approvers approve/reject via console or API. After approval, automation continues (shares AMI cross-account). **Change Manager** can also be used for formal change approval workflows.

**Keywords:** "automation with manual approval", "pause and wait" → `aws:approve` step in Automation (or Change Manager)

---

**Q7:** A company has hybrid infrastructure (EC2 + on-premises servers). They want to manage both through SSM. The on-premises servers are in a private network (no internet). How to connect them to SSM?

**A:** (1) **Hybrid Activation**: Create SSM activation (provides activation code + ID). (2) Install SSM Agent on on-prem servers with the activation credentials. (3) For connectivity without internet: Use **AWS Direct Connect** or **VPN** from on-prem to the VPC. (4) Create **VPC endpoints** for SSM in the VPC. (5) On-prem servers route SSM traffic through Direct Connect/VPN to VPC endpoints.

**Keywords:** "on-premises", "no internet", "SSM" → Hybrid Activation + Direct Connect/VPN + VPC endpoints

---

**Q8:** A company stores 500 parameters across 3 environments (dev/staging/prod). A developer accidentally overwrites a production parameter. How to prevent this?

**A:** (1) **IAM policies** restricting write access by path — developers can only write to `/app/dev/*`, not `/app/prod/*`. (2) Enable **parameter tagging** + tag-based conditions. (3) Use **parameter policies** (Advanced tier) for change notification. (4) Use **AWS Config** to detect unexpected parameter changes. (5) **Resource-level permissions** on `ssm:PutParameter` with condition `ssm:overwrite` = false for production paths.

**Keywords:** "prevent accidental overwrite", "production parameters" → IAM policy restricting by parameter path

---

**Q9:** A company needs to run different patch baselines for Windows and Linux instances in the same maintenance window. How?

**A:** Create **two patch groups** (e.g., `Patch Group: Windows-Prod` and `Patch Group: Linux-Prod`), each associated with its own OS-specific baseline. Register BOTH patch groups as targets in the same maintenance window with the `AWS-RunPatchBaseline` task. The document automatically uses the baseline associated with each instance's patch group.

**Keywords:** "different baselines, same window", "Windows and Linux" → Separate patch groups with respective baselines, same window

---

**Q10:** A company's SSM-managed instances intermittently show as "Connection Lost" in the SSM console. The instances are running and accessible via other means. What should they investigate?

**A:** (1) **SSM Agent health** — agent may be crashing or stuck (check agent logs at `/var/log/amazon/ssm/`). (2) **Network connectivity** — intermittent issues reaching SSM endpoints (check NAT Gateway, VPC endpoints). (3) **IAM Instance Profile** — if the role's session credentials are expiring or being throttled. (4) **DNS resolution** — instances can't resolve SSM endpoint DNS names. (5) **Agent version** — outdated agents may have connectivity bugs (update via `AWS-UpdateSSMAgent`).

**Keywords:** "intermittently Connection Lost" → Agent health + network connectivity + IAM + DNS

---

### Compare: Parameter Store vs Secrets Manager

| Aspect | Parameter Store | Secrets Manager |
|--------|----------------|-----------------|
| **Use Case** | Config values, flags, non-rotating secrets | Database creds, API keys needing rotation |
| **Rotation** | Manual only (policy notifications) | Automatic (Lambda-based) |
| **Size** | 4 KB / 8 KB | 64 KB |
| **Pricing** | Free (standard) | $0.40/secret/month |
| **Cross-account** | Customer CMK required | Native support |
| **CloudFormation** | `{{resolve:ssm:...}}` / `{{resolve:ssm-secure:...}}` | `{{resolve:secretsmanager:...}}` |
| **Exam Tip** | "Free", "config", "no rotation" | "Rotation", "database credentials", "RDS integration" |

### Compare: Run Command vs Automation vs State Manager

| Aspect | Run Command | Automation | State Manager |
|--------|-------------|------------|---------------|
| **Use Case** | Execute commands on instances | Multi-step operational workflows | Enforce desired state continuously |
| **Execution** | One-time or scheduled | One-time or triggered | Scheduled/continuous (associations) |
| **Scope** | Single step (one document) | Multiple steps (branching, approvals, loops) | Single document (applied repeatedly) |
| **Targets** | Managed instances | Instances, accounts, resources | Managed instances |
| **Exam Tip** | "Run script across fleet" | "Multi-step runbook", "approval", "create AMI" | "Ensure always running", "auto-remediate" |

### Decision Tree: SSM Feature Selection

```
Need to STORE configuration?
  ├── Sensitive + needs rotation → Secrets Manager
  ├── Sensitive + no rotation → Parameter Store (SecureString)
  └── Non-sensitive config → Parameter Store (String)

Need to ACCESS instances remotely?
  ├── Shell/CLI access → Session Manager
  ├── Access internal service (DB) from laptop → Session Manager (port forwarding)
  └── GUI/Desktop → Fleet Manager (or direct RDP over Session Manager tunnel)

Need to RUN commands on fleet?
  ├── One-time execution → Run Command
  ├── Multi-step workflow → Automation
  └── Continuous enforcement → State Manager

Need to PATCH instances?
  └── Patch Manager (baseline + patch group + maintenance window)

Need to COLLECT instance metadata?
  └── Inventory (+ Resource Data Sync for S3 export)

Need CHANGE APPROVAL workflow?
  └── Change Manager (approvals + Automation)
```
