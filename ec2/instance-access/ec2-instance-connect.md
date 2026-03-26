# EC2 Instance Connect

---

## Core Objective

**Why does this exist?** Traditional SSH access to EC2 instances requires managing long-lived SSH key pairs — generating them, distributing them securely, rotating them, and revoking access when team members leave. This creates operational overhead and a persistent attack surface (leaked `.pem` files, stale authorized keys, etc.).

**EC2 Instance Connect (EIC)** eliminates the need for static SSH keys by replacing them with **ephemeral, one-time-use public keys** pushed via the AWS API. Access is controlled entirely through **IAM policies**, meaning you can grant, audit, and revoke SSH access without ever touching the instance's filesystem.

> **Exam lens:** EC2 Instance Connect is a favored answer for scenarios asking *"how do you give SSH access without managing SSH keys?"* or *"how do you access an instance in a private subnet without a bastion host?"* (the latter via EIC Endpoint).

**AWS Reference:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connect-linux-inst-eic.html

---

## Fundamental Concepts

### The Two Delivery Modes

| Feature | **EC2 Instance Connect (Classic)** | **EC2 Instance Connect Endpoint (EIC Endpoint)** |
|---|---|---|
| **Launched** | 2019 | 2023 |
| **Instance location** | Public subnet (needs public IP) | **Private subnet** (no public IP needed) |
| **Internet Gateway required** | ✅ Yes | ❌ No |
| **SSH traffic path** | Internet → IGW → EC2 | AWS internal network → EIC Endpoint → EC2 |
| **Cost** | Free | Free (you pay for data transfer only) |
| **Supported protocols** | SSH (port 22) | SSH (port 22) **and RDP (port 3389)** |
| **Replaces bastion host?** | ❌ No | ✅ Yes |
| **VPC Endpoint required** | ❌ | ✅ (EIC creates one automatically) |

---

### Key Terminology

- **Ephemeral key pair:** A temporary RSA/ED25519 key pair generated locally by the AWS CLI or console. The public key is pushed to the instance for **60 seconds** only. After that, it is automatically removed from the instance's `authorized_keys`. The private key never leaves your machine.
- **Instance Metadata Service (IMDS):** EC2 Instance Connect injects the temporary public key via the instance metadata endpoint, which the `aws:ec2-instance-connect` agent reads.
- **EIC Endpoint:** A VPC-level resource that acts as a managed proxy, tunneling SSH/RDP traffic from your workstation through the AWS network fabric into private subnets — without traversing the public internet.
- **`ec2-instance-connect` agent:** A small daemon pre-installed on Amazon Linux 2, Amazon Linux 2023, and Ubuntu (16.04+). It polls IMDS for newly pushed public keys and temporarily appends them to `~/.ssh/authorized_keys`.
- **IAM Principal:** The entity (user, role, assumed role) whose permissions govern whether the `SendSSHPublicKey` or `OpenTunnel` API call is authorized.

---

### How IAM Controls Access

EC2 Instance Connect does **not** use SSH key-based authorization as the trust mechanism — it uses **IAM**. The SSH key is only the transport credential; IAM is the access control layer.

Two critical IAM actions:

| IAM Action | Purpose |
|---|---|
| `ec2-instance-connect:SendSSHPublicKey` | Allows pushing an ephemeral key to a specific instance (Classic mode) |
| `ec2-instance-connect:OpenTunnel` | Allows opening a tunnel through an EIC Endpoint (Endpoint mode) |

You can scope both actions to specific instances via condition keys:
```json
{
  "Effect": "Allow",
  "Action": "ec2-instance-connect:SendSSHPublicKey",
  "Resource": "arn:aws:ec2:us-east-1:123456789012:instance/i-0abcd1234efgh5678",
  "Condition": {
    "StringEquals": {
      "ec2:osuser": "ec2-user"
    }
  }
}
```

**AWS Reference (IAM):** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/permissions-for-ec2-instance-connect.html

---

## Procedure or Logic

### Classic EC2 Instance Connect — How It Works (Step by Step)

```
1. Developer runs: aws ec2-instance-connect send-ssh-public-key ...
   └─ AWS CLI generates a temporary ED25519 key pair locally
   └─ Calls the EC2 Instance Connect API (SendSSHPublicKey)
   └─ IAM evaluates the request → authorized or denied

2. AWS pushes the temporary public key to the instance via IMDS
   └─ Key is available at: http://169.254.169.254/latest/meta-data/managed-ssh-keys/active-keys/<os-user>/

3. The ec2-instance-connect agent (running on the instance) reads the key from IMDS
   └─ Appends it to ~/.ssh/authorized_keys for the specified OS user

4. Developer's SSH client connects using the temporary private key
   └─ Standard SSH handshake succeeds

5. After 60 seconds, the key is automatically invalidated
   └─ No cleanup required; the key simply disappears from IMDS
```

### EIC Endpoint — How It Works (Step by Step)

```
1. Admin creates an EIC Endpoint in the target VPC/subnet (one-time setup)
   └─ aws ec2 create-instance-connect-endpoint --subnet-id subnet-xxxx

2. Developer runs: aws ec2-instance-connect open-tunnel ...
   └─ IAM evaluates OpenTunnel permission
   └─ AWS establishes an encrypted WebSocket tunnel from the developer's machine
      through the EIC Endpoint into the private subnet

3. Developer's SSH client connects to localhost:<local_port>
   └─ Traffic is forwarded through the tunnel to the instance's private IP

4. The connection is auditable via CloudTrail (OpenTunnel API call is logged)
```

### Single CLI command to connect (Classic, convenience wrapper):
```bash
aws ec2-instance-connect ssh \
  --instance-id i-0abcd1234efgh5678 \
  --os-user ec2-user \
  --region us-east-1
```

### Single CLI command to connect via EIC Endpoint (private subnet):
```bash
aws ec2-instance-connect ssh \
  --instance-id i-0abcd1234efgh5678 \
  --connection-type eice \
  --region us-east-1
```

**AWS Reference (CLI):** https://docs.aws.amazon.com/cli/latest/reference/ec2-instance-connect/

---

## Practical Application / Examples

### Example 1 — Classic: Developer Access in a Public Subnet
A startup runs web servers in a public subnet. The DevOps team needs SSH access for debugging but refuses to manage `.pem` files or shared key pairs.

**Solution:** Install `ec2-instance-connect` agent (pre-installed on AL2/AL2023). Attach an IAM policy to the developer's IAM user granting `ec2-instance-connect:SendSSHPublicKey` scoped to the target instance. The developer connects via:
```bash
aws ec2-instance-connect ssh --instance-id i-0abc123 --os-user ec2-user
```
No `.pem` file. No `~/.ssh/authorized_keys` management. Access revoked instantly by removing the IAM policy.

---

### Example 2 — EIC Endpoint: Replacing a Bastion Host
A financial services company runs all application servers in private subnets with no internet-facing exposure. Previously, they maintained a bastion EC2 instance (patching overhead, cost, attack surface).

**Solution:** Deploy a single **EIC Endpoint** in the private subnet. Remove the bastion host entirely. Developers connect using:
```bash
aws ec2-instance-connect ssh \
  --instance-id i-0private123 \
  --connection-type eice
```
Traffic never touches the public internet. Every connection is logged in **CloudTrail** as an `OpenTunnel` event, providing a full audit trail. The bastion host is decommissioned — eliminating its patching burden and associated cost.

---

### Example 3 — IAM-Scoped Access per Environment
A team needs read-only SSH access to production instances but full access to staging.

**Solution:** Create two IAM policies:
```json
// prod-readonly-ssh.json — scoped to specific prod instance
{
  "Effect": "Allow",
  "Action": "ec2-instance-connect:SendSSHPublicKey",
  "Resource": "arn:aws:ec2:us-east-1:ACCOUNT:instance/i-PROD",
  "Condition": { "StringEquals": { "ec2:osuser": "readonly-user" } }
}
```
```json
// staging-full-ssh.json — scoped to staging instance, admin OS user
{
  "Effect": "Allow",
  "Action": "ec2-instance-connect:SendSSHPublicKey",
  "Resource": "arn:aws:ec2:us-east-1:ACCOUNT:instance/i-STAGING",
  "Condition": { "StringEquals": { "ec2:osuser": "ec2-user" } }
}
```
Access is enforced at the IAM layer — no SSH config changes on the instances required.

---

## Critical Considerations

- **Agent must be installed and running:** The `ec2-instance-connect` package must be present. It is pre-installed on **Amazon Linux 2, Amazon Linux 2023, and Ubuntu 16.04+** official AMIs. Custom or hardened AMIs may not include it — install manually via `yum install ec2-instance-connect` or `apt install ec2-instance-connect`.

- **Security Group must allow SSH inbound:**
  - **Classic:** SG must allow TCP/22 from the AWS IP range for EC2 Instance Connect in your region (not from `0.0.0.0/0`). Use the managed prefix list `com.amazonaws.<region>.ec2-instance-connect` or check the IP ranges at: https://ip-ranges.amazonaws.com/ip-ranges.json
  - **EIC Endpoint:** SG must allow TCP/22 only from the EIC Endpoint's security group — no public IP range needed.

- **60-second key TTL is non-configurable:** The ephemeral key validity window is fixed by AWS. You cannot extend it. If the SSH handshake doesn't complete within 60 seconds of `SendSSHPublicKey`, the connection will fail.

- **EIC Endpoint limitations:**
  - One EIC Endpoint per VPC (hard limit).
  - Supports only TCP traffic (SSH/RDP). Cannot tunnel arbitrary protocols.
  - The endpoint lives in a specific subnet but can reach **any instance in the VPC** (not just the subnet it's deployed in), subject to routing and SG rules.

- **Not a replacement for Session Manager in all scenarios:** AWS Systems Manager Session Manager also eliminates SSH key management and doesn't require any open inbound ports (works over HTTPS/443). For instances that **cannot** have the SSM agent installed, EIC is the right answer. For maximum security (zero open ports), **Session Manager** may be preferable.
  - **AWS Reference (Session Manager):** https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html

- **Windows instances:** Classic EIC does **not** support Windows. EIC Endpoint supports RDP (port 3389) for Windows instances.

- **CloudTrail auditability:** Every `SendSSHPublicKey` and `OpenTunnel` call is logged in CloudTrail, including the IAM principal, instance ID, OS user, and timestamp. This is a key compliance argument for EIC over static key pairs.

- **IMDS v2 (IMDSv2) compatibility:** EC2 Instance Connect is fully compatible with IMDSv2 (token-required mode). No special configuration needed.

---

## Critical Synthesis Note

> **High-level insight:** EC2 Instance Connect is an architectural manifestation of the **zero standing privilege** principle from Zero Trust security models. Static SSH keys are a form of *standing access* — they exist and are valid 24/7 regardless of whether they're in use. EIC enforces *just-in-time (JIT) access*: credentials are minted at the moment of need and expire automatically. This is the same pattern used by Vault's dynamic secrets, AWS STS temporary credentials, and certificate-based SSH (e.g., HashiCorp Vault SSH CA). Understanding this design principle helps you reason about *why* EIC is architecturally superior to key pairs for compliance-heavy environments.

> **Knowledge gap / further research for SAA-C03:** The exam frequently juxtaposes EIC with **AWS Systems Manager Session Manager**. The key differentiator is the network model: Session Manager requires **no open inbound ports** (the SSM agent initiates an outbound HTTPS connection), while EIC Endpoint still requires TCP/22 open from the endpoint's SG. For a question asking "how to SSH to an instance with **zero open inbound ports**," Session Manager is the correct answer — not EIC. Map out the decision boundary between these two services before the exam.

> **Cross-disciplinary connection:** The EIC Endpoint's WebSocket tunnel model is architecturally identical to **SSH ProxyJump** (`-J` flag) and **AWS CloudShell's** underlying transport. Both use a managed intermediate node to relay traffic through a trust boundary without exposing the destination directly. Recognizing this pattern helps you understand how AWS PrivateLink, VPC endpoints, and service-side proxies all solve the same class of problem: *controlled cross-boundary access without route exposure*.

---

**Primary AWS Documentation:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connect-linux-inst-eic.html

**EIC Endpoint announcement:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connect-using-eice.html

**IAM permissions reference:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/permissions-for-ec2-instance-connect.html

**AWS IP ranges (for SG rules):** https://ip-ranges.amazonaws.com/ip-ranges.json

**EC2 Instance Connect vs Session Manager:** https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html
