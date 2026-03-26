# EC2 Instance Access Methods — Comparison: EC2 Instance Connect vs SSM Session Manager

---

## Core Objective

**Why does this document exist?** Both EC2 Instance Connect (EIC) and AWS Systems Manager Session Manager (SSM) solve the same surface-level problem — *accessing an EC2 instance without managing static SSH keys* — but they do so through fundamentally different architectures, with different security postures, network requirements, and operational tradeoffs.

Choosing the wrong tool for a given scenario is one of the most common mistakes on the SAA-C03 exam and in production architectures. This document does **not** re-explain how each service works internally. Instead, it focuses exclusively on the **decision boundary** between them.

> **Full service documentation:**
> - EC2 Instance Connect: [ec2-instance-connect.md](https://github.com/GabrielAlmeidaFlores/AWS-C03/blob/main/ec2/instance-access/ec2-instance-connect.md)
> - SSM Session Manager: [ssm-session-manager.md](https://github.com/GabrielAlmeidaFlores/AWS-C03/blob/main/ec2/instance-access/ssm-session-manager.md)

---

## Fundamental Concepts

### Side-by-Side Comparison

| Dimension | **EC2 Instance Connect** | **EC2 Instance Connect Endpoint** | **SSM Session Manager** |
|---|---|---|---|
| **Inbound port required** | ✅ TCP/22 | ✅ TCP/22 (from endpoint SG) | ❌ **None** |
| **Public IP / IGW required** | ✅ Yes | ❌ No | ❌ No |
| **Agent on instance** | `ec2-instance-connect` pkg | `ec2-instance-connect` pkg | `amazon-ssm-agent` |
| **Pre-installed on AL2/AL2023** | ✅ | ✅ | ✅ |
| **Access credential** | Ephemeral key (60s TTL) | Ephemeral key (60s TTL) | None — IAM only |
| **Access control layer** | IAM + ephemeral key | IAM + ephemeral key | IAM only |
| **Session logging (commands)** | ❌ | ❌ | ✅ Native (CloudWatch / S3) |
| **CloudTrail auditability** | ✅ `SendSSHPublicKey` event | ✅ `OpenTunnel` event | ✅ `StartSession` event + full transcript |
| **Works in private subnet, no IGW** | ❌ | ✅ | ✅ |
| **Windows RDP support** | ❌ | ✅ (port 3389) | ✅ (port forwarding) |
| **On-premises servers** | ❌ | ❌ | ✅ (hybrid activation) |
| **VPC Endpoint required (private subnet)** | N/A | Creates its own | 3 Interface Endpoints needed |
| **Max endpoints per VPC** | N/A | **1 per VPC** (hard limit) | Unlimited (3 endpoint types, reused) |
| **Cost** | Free | Free (data transfer only) | Free (SSM) + VPC endpoint cost |
| **Port forwarding** | ❌ | ❌ | ✅ |
| **Protocol** | SSH only | SSH + RDP | Any TCP (via port forwarding) |

---

### The Core Architectural Difference

This is the single most important concept for the exam:

```
EC2 Instance Connect                    SSM Session Manager
─────────────────────────────────────   ───────────────────────────────────────
INBOUND model                           OUTBOUND model

[You] ──────────────────► [EC2]         [You] ◄──────────── [EC2 SSM Agent]
       TCP/22 must be open                      Agent initiates outbound HTTPS/443
                                                No inbound port ever opened
```

**EIC** is a *client-to-server* connection — the network path must exist from your machine to the instance, requiring an open inbound port.

**Session Manager** is a *server-to-service* connection — the SSM Agent dials out to AWS, and your session is delivered through that pre-established channel. The instance is never reachable directly.

---

## Procedure or Logic

### Decision Framework

```
Does the instance have a public IP and is in a public subnet?
  ├─ YES → Is your only concern eliminating static SSH key management?
  │          └─ YES → EC2 Instance Connect (Classic) — simplest, zero cost
  │          └─ NO, need command logging too → SSM Session Manager
  │
  └─ NO (private subnet, no public IP)
       ├─ Do you need SSH/RDP specifically (not just shell access)?
       │    ├─ SSH → EIC Endpoint OR SSM + ProxyCommand
       │    └─ RDP (Windows) → EIC Endpoint (simpler) OR SSM port forwarding
       │
       ├─ Is there only 1 VPC to worry about?
       │    └─ EIC Endpoint works (1 per VPC limit is not a constraint)
       │
       ├─ Do you need full command audit logs / session transcripts?
       │    └─ SSM Session Manager (EIC has no session logging)
       │
       ├─ Do you need access to on-premises servers too?
       │    └─ SSM Session Manager (EIC cannot reach non-EC2 targets)
       │
       └─ Is zero open inbound ports a hard security requirement?
            └─ SSM Session Manager — EIC Endpoint still requires TCP/22
```

---

### When Each Tool "Wins"

**Choose EC2 Instance Connect when:**
- Instance is in a **public subnet** with a public IP
- Team needs standard SSH access without `.pem` file distribution
- Simplicity is the priority — no agents to configure, no VPC Endpoints to create
- Budget is constrained — EIC is completely free

**Choose EIC Endpoint when:**
- Instance is in a **private subnet** and the team is already using SSH workflows (IDE plugins, SCP, SFTP)
- You need **RDP access to Windows** without opening port 3389 publicly
- You want to **eliminate the bastion host** without changing SSH tooling
- There is only **one VPC** (hard limit of 1 endpoint per VPC is acceptable)

**Choose SSM Session Manager when:**
- **Zero open inbound ports** is a hard requirement (PCI-DSS, HIPAA, SOC 2)
- You need **full command-level audit logs** (every keystroke logged to S3/CloudWatch)
- You need to access **on-premises servers** alongside EC2
- You need **port forwarding** to databases, RDP, or internal services
- You have **multiple VPCs** (no per-VPC endpoint limits)
- The instance has **no internet access at all** (using VPC Interface Endpoints for SSM)

---

## Practical Application / Examples

### Example 1 — Exam Scenario: "Zero Open Ports + Full Audit Trail"
> *"A security team requires that no inbound ports be open on production EC2 instances. All access must be logged at the command level for compliance. Which access method should be used?"*

**Answer: SSM Session Manager.**

EIC (even with Endpoint) requires TCP/22 open. Session Manager requires zero inbound ports. Session transcripts provide the command-level audit log.

---

### Example 2 — Exam Scenario: "Replace Bastion Host, Keep SSH Tooling"
> *"A team uses SSH heavily in their IDEs and scripts. They want to eliminate their bastion host without rewriting their tooling. Instances are in private subnets."*

**Answer: EIC Endpoint.**

The EIC Endpoint acts as a transparent SSH proxy. Existing SSH commands work unchanged by adding an `~/.ssh/config` `ProxyCommand` entry. Session Manager would work too, but requires more tooling changes (SSM plugin for CLI) and doesn't support native SCP/SFTP without additional configuration.

---

### Example 3 — Exam Scenario: "On-Premises + EC2 Unified Access"
> *"A company wants a single access mechanism for both EC2 instances and their on-premises VMware servers."*

**Answer: SSM Session Manager with Hybrid Activation.**

EIC is EC2-only. Session Manager with Hybrid Activation registers on-premises servers as SSM managed nodes (`mi-xxxxx`), giving a unified access model across all infrastructure.

---

### Example 4 — Exam Scenario: "Private Subnet, Multiple VPCs, Minimum Cost"
> *"A company has 8 VPCs, all with instances in private subnets. They need a scalable access solution."*

**Answer: SSM Session Manager.**

EIC Endpoint is limited to **1 per VPC**. Deploying one per VPC is possible but requires managing 8 separate endpoints with independent security groups and access controls. SSM Session Manager scales across all VPCs using the same IAM model, with 3 Interface Endpoints per VPC (or a shared Transit Gateway path to centralized VPC endpoints).

---

## Critical Considerations

- **EIC Endpoint still opens a port:** The most common exam trap. EIC Endpoint eliminates the public IP requirement and the internet path, but TCP/22 (or 3389) must still be open **from the endpoint's security group** to the instance's security group. Session Manager requires no such rule.

- **Session Manager has no native SCP/SFTP:** For file transfers, Session Manager requires either the SSM `send-command` with S3, or configuring SSH-over-SSM (ProxyCommand). EIC and EIC Endpoint support native SCP/SFTP out of the box.

- **EIC key TTL is 60 seconds — non-negotiable:** If the SSH handshake doesn't complete within 60 seconds of `SendSSHPublicKey`, the connection fails. Network latency or slow client machines can cause spurious failures. Session Manager has no equivalent timeout on session initiation.

- **Session Manager default OS user is `ssm-user`, not `ec2-user`:** Sessions don't authenticate as the standard instance user by default. This can cause permission issues if scripts or tools assume `ec2-user` home directory paths.

- **Both require IAM:** Neither tool bypasses IAM. Always verify both the instance role (agent auth) and the operator's IAM policy (session initiation) are correctly configured.

- **CloudTrail covers both, but depth differs:** Both emit CloudTrail events on session/connection initiation. Only Session Manager provides **session content logging** (the actual commands typed and output returned). EIC CloudTrail records prove *that* a connection was established, not *what was done*.

---

## Critical Synthesis Note

> **High-level insight:** The EIC vs SSM choice maps directly onto the **defense-in-depth** spectrum. EIC operates at the *network + identity* layer: it enforces IAM authentication but still requires a TCP connection path to the instance. SSM operates at the *identity-only* layer: it eliminates the network path entirely, making the instance unreachable by definition — there is no port to scan, no connection to intercept, no IP to target. This is the architectural definition of a **Zero Trust** access model applied to infrastructure: "never expose, always authenticate." For the exam, the presence of *any* of these keywords in a scenario — compliance, audit logs, zero open ports, on-premises, PCI/HIPAA — should immediately trigger SSM as the answer.

> **Knowledge gap / further research for SAA-C03:** Study how **AWS Systems Manager Fleet Manager** integrates with Session Manager to provide a GUI-based RDP experience for Windows instances entirely through the AWS Console — without opening port 3389 or installing an RDP client. This is a high-signal differentiator from EIC Endpoint's RDP support, which requires a local RDP client and port-forwarding setup.

> **Cross-disciplinary connection:** The EIC-vs-SSM tradeoff mirrors the **VPN vs ZTNA** debate in enterprise network security. Traditional VPN (analogous to EIC) establishes a network tunnel and then relies on application-layer controls — the network path itself is still exposed. ZTNA (analogous to SSM) never exposes the network path; every connection is brokered through an identity-aware proxy. The industry shift from VPN to ZTNA (Cloudflare Access, Zscaler ZPA, BeyondCorp) is the same architectural shift AWS made when building Session Manager as a successor to bastion-host patterns.

---

**EC2 Instance Connect full documentation:** https://github.com/GabrielAlmeidaFlores/AWS-C03/blob/main/ec2/instance-access/ec2-instance-connect.md

**SSM Session Manager full documentation:** https://github.com/GabrielAlmeidaFlores/AWS-C03/blob/main/ec2/instance-access/ssm-session-manager.md

**AWS official comparison guidance:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connect-to-linux-instance.html

**EIC AWS docs:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connect-linux-inst-eic.html

**SSM Session Manager AWS docs:** https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html
