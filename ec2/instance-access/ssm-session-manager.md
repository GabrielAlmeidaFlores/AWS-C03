# AWS Systems Manager — Session Manager

---

## Core Objective

**Why does this exist?** Traditional SSH/RDP access to EC2 instances requires three things that create persistent security and operational risk: open inbound ports (22/3389), a routable network path to the instance, and credential management (SSH keys or Windows passwords). Each of these is an attack surface.

**Session Manager** eliminates all three. It provides interactive shell and port-forwarding access to EC2 instances (and on-premises servers) through a **fully outbound, agent-initiated HTTPS/443 connection** to the AWS Systems Manager service endpoint — no inbound ports, no public IPs, no SSH keys, no bastion hosts required.

Access is governed entirely by **IAM**, and every session is **logged** (CloudWatch Logs, S3) and **auditable** (CloudTrail). This makes Session Manager the preferred access mechanism for compliance-heavy environments (PCI-DSS, HIPAA, SOC 2).

> **Exam lens:** When a scenario asks for *"shell access to an instance with zero open inbound ports"*, *"no SSH key management"*, *"full audit log of every command run"*, or *"access to instances in a private subnet with no internet gateway"* — **Session Manager is the answer**. It is the most secure of the three primary access methods (SSH, EIC, SSM).

**AWS Reference:** https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html

---

## Fundamental Concepts

### Session Manager vs EC2 Instance Connect vs SSH

| Feature | **SSH (classic)** | **EC2 Instance Connect** | **Session Manager** |
|---|---|---|---|
| **Inbound port required** | ✅ TCP/22 | ✅ TCP/22 | ❌ None |
| **Public IP / IGW required** | ✅ (direct) | ✅ (Classic) / ❌ (Endpoint) | ❌ |
| **Key pair management** | ✅ Static keys | ❌ Ephemeral (60s) | ❌ None |
| **Access control** | OS-level (`authorized_keys`) | IAM + ephemeral key | IAM only |
| **Session logging** | ❌ Manual setup | ❌ | ✅ Native (CW Logs / S3) |
| **Command audit trail** | ❌ | ❌ | ✅ CloudTrail + session logs |
| **Works on private subnet, no IGW** | ❌ | ❌ (Classic) / ✅ (Endpoint) | ✅ |
| **Windows RDP support** | ❌ | ✅ (EIC Endpoint only) | ✅ (port forwarding) |
| **On-premises servers** | ✅ | ❌ | ✅ (with SSM Agent + hybrid activation) |
| **Agent required** | ❌ | ✅ `ec2-instance-connect` | ✅ `amazon-ssm-agent` |

---

### Key Terminology

- **SSM Agent (`amazon-ssm-agent`):** A daemon pre-installed on Amazon Linux 2, Amazon Linux 2023, Ubuntu (16.04+), and Windows Server AMIs. It initiates and maintains the outbound HTTPS connection to the SSM service endpoint. This is the fundamental enabler — without it, Session Manager cannot function.
- **SSM Endpoint:** The regional AWS service endpoint (`ssm.<region>.amazonaws.com`, `ssmmessages.<region>.amazonaws.com`, `ec2messages.<region>.amazonaws.com`) that the SSM Agent connects to. In private subnets with no internet access, **VPC Interface Endpoints** must be created for these three services.
- **Session:** An interactive channel between your workstation and an instance, brokered by the SSM service. Each session has a unique Session ID and is fully logged.
- **Hybrid Activation:** A mechanism to register **on-premises servers or VMs** as managed nodes in SSM, enabling Session Manager access to non-EC2 infrastructure.
- **IAM Instance Profile:** The EC2 instance's attached IAM role. It must grant the `AmazonSSMManagedInstanceCore` managed policy (or equivalent) so the SSM Agent can authenticate with the SSM service.
- **Port Forwarding Session:** A Session Manager feature that tunnels a TCP port from a remote instance to a local port on your workstation — enabling RDP, database, or any TCP access without opening inbound firewall rules.

---

### Required IAM Permissions (Two Sides)

Session Manager requires IAM on **both** sides:

**1. Instance side (IAM Role attached to EC2):**
```json
{
  "Effect": "Allow",
  "Action": [
    "ssm:UpdateInstanceInformation",
    "ssmmessages:CreateControlChannel",
    "ssmmessages:CreateDataChannel",
    "ssmmessages:OpenControlChannel",
    "ssmmessages:OpenDataChannel"
  ],
  "Resource": "*"
}
```
> Shortcut: Attach the AWS managed policy **`AmazonSSMManagedInstanceCore`** to the instance role.

**2. User/operator side (IAM User or Role):**
```json
{
  "Effect": "Allow",
  "Action": [
    "ssm:StartSession",
    "ssm:TerminateSession",
    "ssm:ResumeSession",
    "ssm:DescribeSessions",
    "ssm:GetConnectionStatus"
  ],
  "Resource": [
    "arn:aws:ec2:us-east-1:ACCOUNT_ID:instance/i-TARGET_INSTANCE",
    "arn:aws:ssm:*:*:document/AWS-StartSSHSession"
  ]
}
```

**AWS Reference (IAM):** https://docs.aws.amazon.com/systems-manager/latest/userguide/getting-started-restrict-access-quickstart.html

---

### How the Outbound-Only Connection Works

The SSM Agent establishes a **long-polling WebSocket** connection to `ssmmessages.<region>.amazonaws.com` over **HTTPS/443 outbound**. When a user initiates a session via the console or CLI, SSM uses this already-established channel to deliver the interactive shell — no new inbound connection is ever opened. The instance only needs **outbound** internet access (or VPC Interface Endpoints) to reach the SSM endpoints.

```
[Your Workstation]
      │
      │  HTTPS (outbound from workstation to AWS)
      ▼
[SSM Service Endpoint]  ←──── HTTPS/443 outbound (from SSM Agent on EC2)
      │
      ▼
[EC2 Instance — SSM Agent]
      │
      ▼
[Interactive Shell / Port Forward]
```

**Zero inbound ports. Zero public IPs. Zero SSH keys.**

---

## Procedure or Logic

### Setup Procedure (Step by Step)

```
1. Ensure SSM Agent is installed and running on the instance
   └─ Pre-installed on AL2, AL2023, Ubuntu 16.04+, Windows Server
   └─ Manual install: https://docs.aws.amazon.com/systems-manager/latest/userguide/ssm-agent.html

2. Attach IAM Instance Profile to the EC2 instance
   └─ Role must include AmazonSSMManagedInstanceCore managed policy
   └─ Without this, the SSM Agent cannot authenticate with SSM

3. (Private subnet only) Create three VPC Interface Endpoints:
   └─ com.amazonaws.<region>.ssm
   └─ com.amazonaws.<region>.ssmmessages
   └─ com.amazonaws.<region>.ec2messages
   └─ Enable Private DNS on all three endpoints

4. Ensure outbound HTTPS (port 443) is allowed in the Security Group
   └─ The instance initiates outbound — no inbound rules needed for SSM

5. (Optional) Configure session logging in SSM Preferences:
   └─ Enable CloudWatch Logs for session output
   └─ Enable S3 bucket for full session transcript storage
   └─ Enable KMS encryption for session data at rest

6. Start a session:
   └─ Console: EC2 → Instance → Connect → Session Manager tab
   └─ CLI: aws ssm start-session --target i-0abcd1234efgh5678
```

### Port Forwarding (RDP / Database Access):
```bash
# Forward remote port 3389 (RDP) to local port 13389
aws ssm start-session \
  --target i-0abcd1234efgh5678 \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["3389"],"localPortNumber":["13389"]}'

# Then connect your RDP client to localhost:13389
```

### SSH over Session Manager (ProxyCommand):
```bash
# ~/.ssh/config entry — routes SSH through SSM tunnel
Host i-* mi-*
    ProxyCommand sh -c "aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters 'portNumber=%p'"
```
```bash
# Then connect normally:
ssh -i key.pem ec2-user@i-0abcd1234efgh5678
```

**AWS Reference (port forwarding):** https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-sessions-start.html

---

## Practical Application / Examples

### Example 1 — Zero Open Ports: Private Subnet Access
A healthcare company (HIPAA) runs application servers in a private subnet with no internet gateway. Security policy prohibits opening any inbound ports. The team needs shell access for incident response.

**Solution:**
1. Attach `AmazonSSMManagedInstanceCore` to the instance's IAM role.
2. Create VPC Interface Endpoints for `ssm`, `ssmmessages`, `ec2messages` in the private subnet.
3. Ensure Security Group allows **outbound 443** (most SGs allow this by default).

```bash
aws ssm start-session --target i-0private123 --region us-east-1
```

Result: Full interactive shell. No port 22. No public IP. No bastion. **HIPAA-compliant access.**

---

### Example 2 — Full Audit Trail: Compliance Requirement
A financial services firm must log every command executed on production servers, store logs for 7 years, and alert on suspicious activity.

**Solution:**
1. Enable session logging in SSM → Session Manager Preferences:
   - **CloudWatch Logs** for real-time streaming and alerting
   - **S3 bucket** (with lifecycle policy → Glacier after 90 days) for long-term retention
   - **KMS encryption** on session data
2. Create a CloudWatch Metric Filter on the session log group to trigger an alarm on commands like `sudo`, `rm -rf`, `curl`, etc.

Every session now produces:
- A CloudTrail event (`ssm:StartSession`) with IAM principal, instance ID, timestamp
- A full session transcript in S3 (every keystroke and output)

---

### Example 3 — On-Premises Server Access via Hybrid Activation
A company runs 50 on-premises Linux servers that they need to manage alongside their EC2 fleet without VPN.

**Solution:**
1. Create a **Hybrid Activation** in SSM:
```bash
aws ssm create-activation \
  --iam-role SSMServiceRole \
  --registration-limit 50 \
  --region us-east-1
# Returns: ActivationId + ActivationCode
```
2. Install SSM Agent on each on-premises server and register:
```bash
sudo amazon-ssm-agent -register \
  -code "activation-code" \
  -id "activation-id" \
  -region "us-east-1"
```
3. On-premises servers now appear in SSM Fleet Manager as `mi-xxxx` managed instances.
4. Access them identically to EC2:
```bash
aws ssm start-session --target mi-0abcde12345
```

**AWS Reference (hybrid):** https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-managedinstances.html

---

## Critical Considerations

- **SSM Agent must be running:** If the agent crashes or the IAM role is misconfigured, the instance disappears from SSM Fleet Manager. Always monitor SSM Agent health via CloudWatch. Instances show as `Online` or `Connection Lost` in the console.

- **Three VPC endpoints are mandatory for private subnets:** A common misconfiguration is creating only the `ssm` endpoint and forgetting `ssmmessages` and `ec2messages`. The SSM Agent requires **all three** to function in a fully private network.
  - `com.amazonaws.<region>.ssm` — primary API
  - `com.amazonaws.<region>.ssmmessages` — session channel (shell/port-forward)
  - `com.amazonaws.<region>.ec2messages` — Run Command delivery

- **IAM Instance Profile ≠ IAM User permissions:** Both sides must be configured. The instance role governs whether the agent can register with SSM. The user's IAM policy governs whether they can start a session. Forgetting either breaks access.

- **Session Manager does NOT provide OS-level user context by default:** All sessions on Linux default to `ssm-user` (a sudoer created by the agent). You can configure the OS user in SSM Preferences. On Windows, sessions run as `NT AUTHORITY\SYSTEM`.

- **Session duration limit:** Sessions time out after **20 minutes of inactivity** by default. This is configurable in Session Manager Preferences (1–60 minutes, or set to never time out).

- **Not a replacement for Run Command at scale:** Session Manager is designed for interactive access. For running scripts across a fleet of instances, **SSM Run Command** or **SSM Automation** is the correct tool.

- **Data encryption in transit:** Session data is encrypted in transit using **TLS 1.2**. For encryption at rest (session logs in S3/CloudWatch), you must configure a **KMS CMK** explicitly — it is not automatic.

- **Cross-account access:** You can start a session on an instance in Account B from Account A by assuming a cross-account role. The instance role in Account B must still have `AmazonSSMManagedInstanceCore`.

- **Cost:** Session Manager itself is **free**. You pay for: SSM Advanced Instances tier (if used), VPC Interface Endpoints (~$0.01/hr each), CloudWatch Logs ingestion, and S3 storage for session transcripts.

---

## Critical Synthesis Note

> **High-level insight:** Session Manager inverts the traditional security model for remote access. Classical SSH is a **pull model** — the client initiates a connection *to* the server, requiring the server to expose a listening port. Session Manager is a **push model** — the server (SSM Agent) initiates and maintains a persistent outbound channel, and the SSM service *pushes* session data into that channel on demand. This architectural inversion is why Session Manager achieves zero-open-ports access. The same pattern is used by IoT device management platforms (AWS IoT Core, Azure IoT Hub) and modern Zero Trust Network Access (ZTNA) proxies (Cloudflare Access, Zscaler ZPA) — all of which rely on outbound-initiated tunnels to eliminate inbound exposure.

> **Knowledge gap / further research for SAA-C03:** Understand the **cost and tier model** for SSM: by default, EC2 instances use the *Standard* managed instance tier (free). On-premises servers and some advanced features require the *Advanced* tier ($0.00695/instance/hr). The exam may test whether you understand when costs are incurred. Also research **SSM Patch Manager** and **SSM State Manager** — Session Manager is one component of a larger SSM suite, and exam scenarios sometimes require you to distinguish between them.

> **Cross-disciplinary connection:** The session logging capability of Session Manager (full keystroke capture + S3 storage) is functionally equivalent to a **Privileged Access Management (PAM)** system like CyberArk or BeyondTrust — both of which record and audit privileged sessions. AWS positions Session Manager as a cloud-native PAM replacement. For organizations migrating from on-premises PAM solutions, this is a direct architectural substitute that reduces third-party tooling cost and complexity.

---

**Primary AWS Documentation:** https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html

**IAM permissions for Session Manager:** https://docs.aws.amazon.com/systems-manager/latest/userguide/getting-started-restrict-access-quickstart.html

**VPC Endpoints for SSM (private subnet setup):** https://docs.aws.amazon.com/systems-manager/latest/userguide/setup-create-vpc.html

**SSM Agent installation:** https://docs.aws.amazon.com/systems-manager/latest/userguide/ssm-agent.html

**Port forwarding sessions:** https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-sessions-start.html

**Hybrid activations (on-premises):** https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-managedinstances.html

**Session Manager pricing:** https://aws.amazon.com/systems-manager/pricing/
