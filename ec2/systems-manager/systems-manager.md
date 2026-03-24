# AWS Systems Manager

---

## Core Objective

**Why does this exist?** Operating a fleet of EC2 instances — or a hybrid estate of EC2 plus on-premises servers — without a unified control plane forces teams into a fragmented set of tools: SSH for ad-hoc commands, cron jobs for patching, hand-rolled scripts for configuration drift detection, and Secrets Manager or environment variables for parameter distribution. Each of these point solutions carries its own access model, audit surface, and operational toil.

AWS Systems Manager (SSM) is the **unified operational control plane for EC2 and hybrid infrastructure**. It replaces this fragmented approach with a single agent-based framework — `amazon-ssm-agent` — that enables secure shell access, remote command execution, automated patching, configuration enforcement, centralized parameter storage, and fleet-wide inventory collection. Every capability shares the same IAM-governed, agent-initiated outbound HTTPS connection model, which means no inbound firewall ports, no SSH key rotation burden, and a single audit trail in CloudTrail.

SSM is not a single service — it is a **suite of capabilities** organized around a common agent and API surface. The core sub-services relevant to EC2 operations are: Session Manager, Run Command, Patch Manager, State Manager, Parameter Store, Inventory, and Automation.

> **Exam lens:** SAA-C03 tests SSM across multiple domains. Session Manager is tested as the zero-open-port access method. Parameter Store is tested against Secrets Manager for configuration data storage. Patch Manager is tested for fleet-wide OS patch automation. Run Command is the answer to "run a script on 500 instances simultaneously without SSH." Know the boundaries between these sub-services — the exam frequently presents scenarios where you must pick the right SSM capability, not just "use SSM."

**AWS Reference:** https://docs.aws.amazon.com/systems-manager/latest/userguide/what-is-systems-manager.html

---

## Fundamental Concepts

### Key Terminology

| Term | Definition |
|---|---|
| **SSM Agent** | Daemon (`amazon-ssm-agent`) pre-installed on Amazon Linux 2, AL2023, Ubuntu 16.04+, Windows Server. Initiates and holds an outbound HTTPS/443 WebSocket to SSM endpoints. The single prerequisite for all SSM capabilities. |
| **Managed Instance** | An EC2 instance or on-premises server registered with SSM. EC2 instances are registered automatically via the IAM Instance Profile. On-premises servers are registered via Hybrid Activations. Identified as `i-xxxx` (EC2) or `mi-xxxx` (non-EC2). |
| **IAM Instance Profile** | The IAM role attached to an EC2 instance. Must include `AmazonSSMManagedInstanceCore` (or equivalent) for the SSM Agent to authenticate with the SSM service. |
| **SSM Document (runbook)** | A JSON or YAML specification that defines a series of actions — shell commands, scripts, AWS API calls, or approval steps. All SSM capabilities (Run Command, Automation, State Manager, Session Manager) are driven by SSM documents. AWS provides hundreds of pre-built documents; you can also author custom ones. |
| **Parameter Store** | A hierarchical key-value store for configuration data and secrets. Supports plaintext (`String`, `StringList`) and encrypted (`SecureString`, backed by KMS) values. Accessible from EC2 user data, Lambda, ECS tasks, and CI/CD pipelines. |
| **Patch Baseline** | A rule set that defines which patches are approved or rejected for a given OS. AWS provides default baselines; custom baselines let you control approval delay, patch classification, and severity filtering. |
| **Maintenance Window** | A scheduled time block during which SSM runs disruptive operations (patching, reboots) on managed instances. Prevents uncontrolled patch application during business hours. |
| **Hybrid Activation** | A registration mechanism for non-EC2 resources (on-premises VMs, edge devices). Generates an `ActivationId` and `ActivationCode` used by the SSM Agent to register as an `mi-xxxx` managed instance. |
| **State Manager Association** | A policy that enforces a desired configuration state on managed instances on a defined schedule. If an instance drifts from the desired state (e.g., a required package is removed), State Manager re-applies it. |
| **Fleet Manager** | The SSM console view that provides a unified dashboard of all managed instances, their SSM Agent versions, OS metadata, and compliance status. |

---

### SSM Sub-Service Map

| Sub-Service | Primary Job | Trigger Model | Typical Use Case |
|---|---|---|---|
| **Session Manager** | Interactive shell / port forwarding | On-demand (user-initiated) | Incident response, debugging, replacing SSH/RDP |
| **Run Command** | Execute scripts/commands fleet-wide | On-demand (API/console/EventBridge) | Bootstrap scripts, log collection, ad-hoc operations at scale |
| **Patch Manager** | OS patch application and compliance | Scheduled (Maintenance Window) or on-demand | Fleet-wide patching without SSH; PCI/SOC2 compliance |
| **State Manager** | Desired-state configuration enforcement | Scheduled (association) | Ensuring packages/config are always present; drift remediation |
| **Parameter Store** | Centralized config and secrets storage | Pull (application runtime) | DB endpoints, API keys, feature flags — replacing hardcoded config |
| **Inventory** | Collect instance metadata (packages, network, services) | Scheduled | Asset tracking, compliance auditing, license management |
| **Automation** | Multi-step workflows with branching and approvals | On-demand, scheduled, or event-driven | AMI creation pipelines, instance remediation runbooks, DR workflows |

---

### The SSM Agent Connection Model

All SSM capabilities share the same underlying transport. The SSM Agent on each managed instance holds a **long-polling WebSocket** (outbound HTTPS/443) to three regional endpoints:

```
com.amazonaws.<region>.ssm          ← primary SSM API endpoint
com.amazonaws.<region>.ssmmessages  ← interactive session channel (Session Manager)
com.amazonaws.<region>.ec2messages  ← Run Command and document delivery
```

In a **private subnet with no internet gateway**, you must create **VPC Interface Endpoints** for all three services. Without them, the SSM Agent cannot reach the SSM service and the instance will not appear as managed.

```
[EC2 Instance — SSM Agent]
        │
        │  HTTPS/443 outbound (agent-initiated)
        ▼
[VPC Interface Endpoint / Internet Gateway]
        │
        ▼
[SSM Service Endpoint — Regional]
        │
        ▼
[Session Manager / Run Command / Patch Manager / ...]
```

**Zero inbound ports required.** The instance never listens for incoming connections from the SSM service.

**AWS Reference (VPC Endpoints):** https://docs.aws.amazon.com/systems-manager/latest/userguide/setup-create-vpc.html

---

### Parameter Store: String vs SecureString

| Attribute | **String / StringList** | **SecureString** |
|---|---|---|
| **Encryption** | None (plaintext) | AWS KMS CMK (aws/ssm default key or custom CMK) |
| **Use case** | Non-sensitive config: URLs, feature flags, instance IDs | Secrets: DB passwords, API tokens, license keys |
| **IAM required** | `ssm:GetParameter` | `ssm:GetParameter` + `kms:Decrypt` |
| **Cross-account access** | With IAM resource policies | Requires sharing the KMS CMK cross-account |
| **Cost (Standard tier)** | Free up to 10,000 parameters | Free (KMS decrypt charges apply per call) |
| **Cost (Advanced tier)** | $0.05/parameter/month | $0.05/parameter/month + KMS decrypt |
| **Max value size** | Standard: 4 KB / Advanced: 8 KB | Standard: 4 KB / Advanced: 8 KB |
| **Version history** | ✅ Up to 100 versions retained | ✅ Up to 100 versions retained |

**AWS Reference (Parameter Store):** https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html

---

## Procedure or Logic

### End-to-End Setup: Making an EC2 Instance SSM-Managed

- **Step 1 — Attach an IAM Instance Profile:** Create or attach an IAM role to the EC2 instance with the `AmazonSSMManagedInstanceCore` managed policy. This policy grants the SSM Agent permissions to register with SSM, send heartbeats, receive documents, and publish session data.

- **Step 2 — Verify SSM Agent is installed and running:** On AL2/AL2023/Windows Server, the agent is pre-installed. For other distributions, install manually.

```bash
# Check agent status on Linux
sudo systemctl status amazon-ssm-agent

# Restart if needed
sudo systemctl restart amazon-ssm-agent

# Check version
amazon-ssm-agent --version
```

- **Step 3 — (Private subnet) Create three VPC Interface Endpoints:** In the VPC console, create Interface Endpoints for `com.amazonaws.<region>.ssm`, `com.amazonaws.<region>.ssmmessages`, and `com.amazonaws.<region>.ec2messages`. Enable **Private DNS** on all three.

- **Step 4 — Verify the instance appears as managed:** In the SSM console → Fleet Manager, the instance should show status `Online`. If it shows `Connection Lost`, the most common causes are: missing Instance Profile, SSM Agent not running, or missing VPC endpoints in private subnets.

- **Step 5 — (Optional) Configure session logging:** In Session Manager Preferences, enable CloudWatch Logs and/or S3 for session transcript capture.

**AWS Reference (setup):** https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-setting-up.html

---

### Run Command: Execute at Scale

Run Command sends an SSM document to one or more managed instances simultaneously, without opening SSH. It supports targeting by instance ID, tag, resource group, or all managed instances.

```bash
# Run a shell script on all instances tagged Env=production
aws ssm send-command \
  --document-name "AWS-RunShellScript" \
  --targets '[{"Key":"tag:Env","Values":["production"]}]' \
  --parameters '{"commands":["yum update -y amazon-ssm-agent","systemctl restart myapp"]}' \
  --timeout-seconds 300 \
  --output-s3-bucket-name "my-ssm-output-bucket" \
  --region us-east-1

# Check command status
aws ssm list-command-invocations \
  --command-id "<CommandId>" \
  --details \
  --region us-east-1
```

**AWS Reference (Run Command):** https://docs.aws.amazon.com/systems-manager/latest/userguide/execute-remote-commands.html

---

### Patch Manager: Fleet-Wide Patching

```bash
# 1. Create a custom patch baseline (Amazon Linux 2, security patches only, auto-approve after 7 days)
aws ssm create-patch-baseline \
  --name "AL2-Security-7DayDelay" \
  --operating-system "AMAZON_LINUX_2" \
  --approval-rules '{"PatchRules":[{"PatchFilterGroup":{"PatchFilters":[{"Key":"CLASSIFICATION","Values":["Security"]},{"Key":"SEVERITY","Values":["Critical","Important"]}]},"AutoApproveAfterDays":7,"ComplianceLevel":"CRITICAL"}]}' \
  --region us-east-1

# 2. Associate the baseline with a patch group (instances tagged with Patch Group key)
aws ssm register-default-patch-baseline \
  --baseline-id "pb-0123456789abcdef0"

# 3. Run patch scan (assess compliance, do not install)
aws ssm send-command \
  --document-name "AWS-RunPatchBaseline" \
  --targets '[{"Key":"tag:PatchGroup","Values":["prod-web"]}]' \
  --parameters '{"Operation":["Scan"]}' \
  --region us-east-1

# 4. Run patch install (install approved patches)
aws ssm send-command \
  --document-name "AWS-RunPatchBaseline" \
  --targets '[{"Key":"tag:PatchGroup","Values":["prod-web"]}]' \
  --parameters '{"Operation":["Install"]}' \
  --region us-east-1
```

**AWS Reference (Patch Manager):** https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-patch.html

---

### Parameter Store: Read at Runtime

```bash
# Store a SecureString parameter
aws ssm put-parameter \
  --name "/prod/db/password" \
  --value "supersecret123" \
  --type "SecureString" \
  --key-id "alias/my-kms-key" \
  --region us-east-1

# Read parameter from an EC2 instance at runtime (decrypted)
aws ssm get-parameter \
  --name "/prod/db/password" \
  --with-decryption \
  --query "Parameter.Value" \
  --output text \
  --region us-east-1

# Read in a shell script (user data or application startup)
DB_PASSWORD=$(aws ssm get-parameter \
  --name "/prod/db/password" \
  --with-decryption \
  --query "Parameter.Value" \
  --output text)
export DB_PASSWORD
```

```python
# Read from Python (boto3) at application startup
import boto3
ssm = boto3.client('ssm', region_name='us-east-1')
response = ssm.get_parameter(Name='/prod/db/password', WithDecryption=True)
db_password = response['Parameter']['Value']
```

---

## Practical Application / Examples

### Scenario 1 — Zero-SSH Fleet Management for a Regulated Workload

A fintech company runs 200 EC2 instances (private subnets, no internet gateway) processing payment data under PCI-DSS. Security policy requires: no open inbound ports, full audit log of every privileged command, automated monthly OS patching with a 7-day delay after vendor release, and centralized storage of all database credentials.

**Architecture:**

```
[Engineer Workstation]
        │ HTTPS (aws ssm start-session)
        ▼
[SSM Service — us-east-1]
        │
        │ (via VPC Interface Endpoints: ssm, ssmmessages, ec2messages)
        ▼
[EC2 Fleet — Private Subnet — No IGW]
   ├─ IAM Role: AmazonSSMManagedInstanceCore + KMS Decrypt
   ├─ SSM Agent (outbound HTTPS only)
   ├─ Patch Group tag: "pci-prod"
   └─ Parameter Store: /prod/payments/db-password (SecureString, CMK)
```

**Implementation:**
1. All instances have IAM Instance Profile with `AmazonSSMManagedInstanceCore` + `kms:Decrypt` on the payments CMK.
2. Three VPC Interface Endpoints (ssm, ssmmessages, ec2messages) with Private DNS enabled.
3. Session Manager Preferences: CloudWatch Logs enabled, S3 bucket with 7-year lifecycle policy for PCI audit retention.
4. Custom Patch Baseline: Security + Critical patches only, 7-day auto-approve delay.
5. Maintenance Window: Sundays 02:00–04:00 UTC. Targets instances tagged `PatchGroup=pci-prod`.
6. All DB credentials stored in Parameter Store as `SecureString` under `/prod/payments/*`. Applications read at startup via `get-parameter --with-decryption`.

**Outcome:** Zero open inbound ports across 200 instances. Every privileged shell command logged to S3. Monthly patching automated with zero manual SSH. Credentials never appear in instance user data, environment files, or source code.

---

### Scenario 2 — Hybrid Fleet: On-Premises Servers + EC2 Unified Management

A media company migrates 60% of workloads to EC2 while retaining 40 on-premises render farm servers. They need a single pane of glass for patching, inventory, and incident response across both environments.

**Solution:**
```bash
# Step 1: Create a Hybrid Activation for on-premises servers
aws ssm create-activation \
  --iam-role "AmazonEC2RunCommandRoleForManagedInstances" \
  --registration-limit 40 \
  --description "Render farm hybrid activation" \
  --region us-east-1
# Returns: ActivationId + ActivationCode

# Step 2: Register SSM Agent on each on-premises server
sudo amazon-ssm-agent -register \
  -code "abc123ActivationCode" \
  -id "de12-ab34-56cd-7890" \
  -region "us-east-1"

# Step 3: Verify registration
aws ssm describe-instance-information \
  --filters "Key=ResourceType,Values=ManagedInstance" \
  --region us-east-1
```

On-premises servers appear as `mi-xxxx` nodes in Fleet Manager alongside EC2 `i-xxxx` instances. Run Command, Patch Manager, Inventory, and State Manager work identically across both.

**Outcome:** Single Maintenance Window covers both EC2 and on-premises servers. Inventory collection feeds a unified CMDB. Patch compliance dashboard covers the full 240-node estate.

---

### Scenario 3 — Parameter Store vs Secrets Manager: Cost-Optimized Config Distribution

A SaaS platform stores 3,000 configuration parameters: 200 are sensitive (DB passwords, API tokens), 2,800 are non-sensitive (feature flags, service URLs, timeout values).

**Decision:**
- **Parameter Store (Standard tier):** Stores all 3,000 parameters free (under 10,000 limit). SecureString for the 200 sensitive ones (KMS decrypt charged per API call at ~$0.03/10,000 calls).
- **Secrets Manager:** Would add $0.40/secret/month × 200 = **$80/month**, plus automatic rotation capability.

**Rule of thumb:**
- Use **Parameter Store SecureString** when: no automatic rotation needed, cost sensitivity matters, parameters are accessed at startup (not per-request).
- Use **Secrets Manager** when: automatic credential rotation is required (RDS, Redshift, DocumentDB have native integrations), or cross-account secret sharing with resource-based policies is needed.

```bash
# Retrieve all parameters under a path prefix (e.g., all prod config at startup)
aws ssm get-parameters-by-path \
  --path "/prod/myapp/" \
  --recursive \
  --with-decryption \
  --region us-east-1
```

---

## Critical Considerations

- **All three VPC endpoints are mandatory in private subnets — not just `ssm`:** A frequent misconfiguration is creating only `com.amazonaws.<region>.ssm` and omitting `ssmmessages` and `ec2messages`. The SSM Agent requires all three: `ssm` for registration and API calls, `ssmmessages` for Session Manager interactive channels, and `ec2messages` for Run Command document delivery. Missing any one causes different SSM capabilities to silently fail.

- **Instance Profile is required — security groups alone are not sufficient:** Outbound port 443 in the Security Group allows the network path, but the SSM Agent still needs IAM credentials from the Instance Profile to authenticate with the SSM service. An instance with no attached role will never appear in Fleet Manager regardless of network configuration.

- **Run Command has a maximum target rate and error threshold — set them explicitly:** By default, `send-command` with tag-based targeting applies to all matching instances at once. For production fleets, always specify `--max-concurrency` (e.g., `"10%"`) and `--max-errors` (e.g., `"5%"`) to implement safe rolling execution and automatic abort on failures.

- **Patch Manager does not handle application-layer patching:** SSM Patch Manager operates at the OS package manager level (yum, apt, Windows Update). It does not patch Java runtimes, Python packages, Node.js modules, Docker images, or custom application binaries. Application-layer patching requires a separate pipeline (CodePipeline, custom SSM Automation documents, or container image rebuilds).

- **Parameter Store Standard tier has a throughput limit of 40 TPS:** If your application reads parameters at high frequency (e.g., per-request), you will hit rate limits. Mitigate by caching parameter values at application startup or using the **Advanced tier** (10,000 TPS). Alternatively, use AWS AppConfig for high-frequency feature flag reads.

- **SecureString parameters require explicit `--with-decryption` — the default is ciphertext:** Calling `get-parameter` without `--with-decryption` returns the KMS-encrypted blob, not the plaintext value. This is a common development-time bug when testing parameter reads. The IAM principal must also have `kms:Decrypt` on the specific CMK used to encrypt the parameter.

- **State Manager associations execute on a schedule, not in real-time:** If an instance drifts from the desired state (e.g., a critical package is removed at 14:00 and the next association run is at 18:00), the drift persists for hours. For real-time configuration enforcement, use a combination of State Manager (remediation) and CloudWatch alarms or AWS Config rules (detection) to trigger immediate Run Command executions via EventBridge.

- **SSM Agent versions must be kept current:** Older SSM Agent versions may not support newer SSM features, and some security patches require an agent update. Use State Manager with the `AWS-UpdateSSMAgent` document on a weekly schedule to keep the agent up to date across the fleet automatically.

- **Maintenance Window tasks have a maximum concurrency and error rate — defaults are aggressive:** The default Maintenance Window task configuration runs on all targets simultaneously with no error abort. For instance patching, always configure `MaxConcurrency` and `MaxErrors` to prevent simultaneous reboots of your entire fleet during a patching window.

- **Parameter Store is not a secrets rotation service:** Unlike Secrets Manager, Parameter Store does not support automatic rotation of `SecureString` values. If you store RDS passwords in Parameter Store, you are responsible for updating the parameter value and redeploying applications when the password changes. For rotation-required secrets, Secrets Manager is the correct service.

---

## Critical Synthesis Note

> **High-level architectural insight — SSM as the EC2 control plane, not just an access tool:** The common framing of SSM as "SSH replacement" significantly undersells its role. In a mature AWS architecture, SSM is the **operational control plane for the compute layer** — the equivalent of what Ansible, Chef, or Puppet provide in traditional infrastructure, but without the need to maintain a separate management server, agent credentials, or network access paths. Session Manager, Run Command, Patch Manager, State Manager, and Automation collectively implement the full **observe → detect → remediate** loop for EC2: Inventory and Fleet Manager provide observability; AWS Config integration and Patch Compliance provide detection; Run Command and Automation provide remediation. Viewed this way, SSM is not a feature — it is the foundation on which you build an autonomous self-healing compute fleet.

> **Knowledge gap requiring further research:** The interaction between **SSM Automation and AWS Config Remediation Actions** is worth deeper investigation. AWS Config can trigger SSM Automation documents as auto-remediation actions when a compliance rule is violated (e.g., an instance is missing required tags, or a security group has port 22 open). The boundary between what is handled by Config Rules vs. State Manager Associations vs. Automation documents for configuration enforcement is not clearly delineated in AWS documentation, and the exam may test scenarios requiring you to select the right combination. Also research **SSM Change Calendar** — a lesser-known sub-service that gates Automation and Maintenance Window execution based on calendar-defined change freeze windows, which is relevant for enterprise change management integrations.

---

**Primary AWS Documentation:** https://docs.aws.amazon.com/systems-manager/latest/userguide/what-is-systems-manager.html
**SSM Agent installation and configuration:** https://docs.aws.amazon.com/systems-manager/latest/userguide/ssm-agent.html
**Setting up SSM (IAM, VPC Endpoints):** https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-setting-up.html
**VPC Endpoints for SSM:** https://docs.aws.amazon.com/systems-manager/latest/userguide/setup-create-vpc.html
**Run Command:** https://docs.aws.amazon.com/systems-manager/latest/userguide/execute-remote-commands.html
**Patch Manager:** https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-patch.html
**Parameter Store:** https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html
**State Manager:** https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-state.html
**Automation:** https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-automation.html
**Hybrid Activations:** https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-managedinstances.html
**SSM Pricing:** https://aws.amazon.com/systems-manager/pricing/
