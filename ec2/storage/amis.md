# Amazon Machine Images (AMIs)

---

## Core Objective

**Why does this exist?** Launching an EC2 instance from scratch — installing the OS, configuring software, applying security hardening, installing agents — is slow, error-prone, and inconsistent at scale. AMIs solve this by capturing a complete instance blueprint: the root volume state, block device mappings, launch permissions, and virtualization metadata in a single reusable artifact.

The result: identical, pre-configured instances launch in seconds rather than minutes, with zero configuration drift between them.

> **Exam lens:** Key distinctions tested are **EBS-backed vs. instance store-backed** (can you stop it? can you recover it?), **AMIs are region-specific** (must copy to use elsewhere), and the **Golden AMI pattern** as the correct answer for launching pre-configured fleets consistently.

**AWS Reference:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html

---

## Fundamental Concepts

### Key Terminology

| Term | Definition |
|---|---|
| **AMI** | A template containing the software configuration (OS, application, data) required to launch an EC2 instance |
| **Root device** | The volume from which the instance boots — either an EBS snapshot or an S3-backed instance store image |
| **Block device mapping** | The specification of all volumes (root + additional EBS) attached to an instance at launch, embedded in the AMI |
| **Launch permission** | Controls who can use the AMI: private (owner only), shared (specific AWS accounts), or public (anyone) |
| **Golden AMI** | An internally hardened, pre-configured AMI used as the standard base for all instance launches in an organization |
| **AMI deregistration** | Removing an AMI from use; does **not** automatically delete the underlying EBS snapshots |
| **AMI copy** | Replicating an AMI to another region; creates a new AMI ID and copies the associated snapshots to the target region |
| **Virtualization type** | `HVM` (hardware virtual machine) — the current standard; `PV` (paravirtual) — legacy, not supported on modern instance types |

---

### EBS-Backed vs. Instance Store-Backed

| Feature | **EBS-Backed** | **Instance Store-Backed** |
|---|---|---|
| **Root volume** | EBS snapshot | S3-stored template (uploaded at launch) |
| **Can be stopped** | ✅ Yes — instance preserves state on EBS | ❌ No — only running or terminated |
| **Data persistence on stop** | ✅ Root volume persists | ❌ All instance store data is lost on stop/terminate |
| **Boot time** | Faster (EBS attach) | Slower (S3 → instance copy) |
| **Snapshot support** | ✅ Yes | ❌ No |
| **Primary use case** | All general workloads | Ephemeral scratch storage, stateless fleets |

The vast majority of modern workloads use EBS-backed AMIs. Instance store-backed AMIs exist primarily for legacy reasons.

---

## Procedure or Logic

- **Step 1 — Configure a source instance**: Launch an instance, install and configure OS, software, agents, and hardening. This is the baseline that will be baked into the AMI.

- **Step 2 — Create the AMI**: AWS stops the instance (for EBS-backed), takes a snapshot of the root volume (and any mapped EBS volumes), and registers the AMI with those snapshot IDs embedded in the block device mapping. The AMI ID is region-specific.

- **Step 3 — Launch instances from the AMI**: Any instance launched from this AMI gets an exact copy of the root volume (via snapshot restore) with the pre-configured software state.

- **Step 4 — Copy to other regions (if needed)**: AMIs cannot be launched in a region where they don't exist. Copy the AMI to replicate both the metadata and the underlying snapshots.

```bash
# Create an AMI from a running or stopped instance
aws ec2 create-image \
  --instance-id i-0abc1234567890def \
  --name "golden-ami-app-v2-$(date +%Y%m%d)" \
  --description "Hardened app server with agent v2.3" \
  --no-reboot   # skip reboot for speed; may affect filesystem consistency

# Copy an AMI to another region for multi-region deployment
aws ec2 copy-image \
  --source-region us-east-1 \
  --source-image-id ami-0abc1234567890def \
  --region eu-west-1 \
  --name "golden-ami-app-v2-eu"

# Share an AMI with another AWS account
aws ec2 modify-image-attribute \
  --image-id ami-0abc1234567890def \
  --launch-permission "Add=[{UserId=123456789012}]"

# Deregister an AMI (does NOT delete snapshots)
aws ec2 deregister-image --image-id ami-0abc1234567890def
# Must separately delete snapshots if cleanup is needed
```

---

## Practical Application / Examples

### Scenario: Golden AMI Pipeline for a Hardened Application Fleet

A security team requires all EC2 instances to launch with: CIS-hardened Amazon Linux 2023, CloudWatch Agent pre-installed and configured, SSM Agent, and a specific app version baked in. Manual installs are banned.

**Architecture:**

```
CodePipeline (weekly trigger)
  └── EC2 Image Builder
        ├── Base: AWS-provided Amazon Linux 2023 AMI
        ├── Component: CIS Level 1 hardening script
        ├── Component: CloudWatch Agent install + config
        ├── Component: App v2.3 install
        └── Output: Golden AMI (us-east-1)
              └── AMI copy → eu-west-1, ap-southeast-1
                    └── SSM Parameter Store: /golden-ami/latest = ami-xxxxxxxx
                          └── Launch Templates reference SSM parameter
                                └── ASGs always launch latest golden AMI
```

**Outcome**: Every instance in every region launches from an identical, tested, hardened image. AMI version is updated weekly with zero changes to Launch Templates (they resolve the SSM parameter at launch time).

---

## Critical Considerations

- **Deregistering an AMI does not delete its snapshots**: After `deregister-image`, the underlying EBS snapshots remain and continue to incur storage costs. Explicitly delete them afterward if the AMI is no longer needed.

- **AMIs are regional — cross-region deployment requires explicit copy**: A Launch Template in `eu-west-1` cannot reference an AMI ID from `us-east-1`. Copy the AMI before deploying to a new region. AMI IDs differ between regions even for the same image content.

- **`--no-reboot` risks filesystem inconsistency**: Skipping the reboot during AMI creation means in-flight writes may not be flushed. Safe for stateless app servers; risky for databases or any instance with open write transactions.

- **Public AMIs can expose data**: If an AMI is made public and its snapshot contained sensitive data (credentials, application secrets), that data becomes accessible to anyone. Audit AMI permissions before sharing.

- **AMI launch permissions do not control snapshot access**: Sharing an AMI with another account lets them launch instances from it. It does not automatically grant access to the underlying snapshots. If they need to copy the AMI, you must also share the snapshots.

- **EC2 Image Builder is the production-grade AMI pipeline**: For teams building AMIs regularly, Image Builder provides versioning, testing, distribution pipelines, and compliance scanning. Ad-hoc `create-image` calls do not scale operationally.

---

## Critical Synthesis Note

> **High-level insight — An AMI is a deployment artifact, not a backup tool:**
>
> The operational role of an AMI is identical to a Docker image or a VM template in on-premises infrastructure: it is an immutable, versioned, deployable unit. Treating AMIs as backups (snapshot the running instance every night) conflates two concerns — instance recovery (handled by snapshots + Auto Recovery) and configuration management (handled by AMIs). A well-run org maintains a Golden AMI pipeline separate from its backup strategy.

> **Knowledge gap requiring further research**: The behavior of Launch Templates referencing an SSM parameter for AMI ID when the parameter is updated — specifically whether running instances in an ASG are replaced automatically or only on the next scale-out event — warrants verification for zero-downtime Golden AMI rollout strategies.

---

## SAA-C03 Exam Focus

- **AMIs are region-specific** — always copy before using in another region; the AMI ID changes on copy.
- **EBS-backed = stoppable; instance store-backed = not stoppable** — this is the most common tested distinction. Instance store data is lost on stop or terminate.
- **Deregister ≠ delete snapshots** — deregistering an AMI leaves its snapshots intact; you must delete them separately to stop incurring costs.
- **Golden AMI pattern** — when the question asks how to ensure all instances launch with the same pre-configured software/hardening, the answer is a Golden AMI (not User Data scripts, which run at boot and are slower/less consistent).
- **`--no-reboot` flag** — skipping reboot speeds up AMI creation but risks filesystem inconsistency; only safe for stateless instances.
- **Sharing an AMI does not share its snapshots** — the recipient can launch instances but cannot copy the AMI without explicit snapshot sharing.

---

**Primary AWS Documentation:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html
**EBS-Backed vs Instance Store-Backed:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/RootDeviceStorage.html
**Creating an AMI from an Instance:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/creating-an-ami-ebs.html
**Copying an AMI:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/CopyingAMIs.html
**EC2 Image Builder:** https://docs.aws.amazon.com/imagebuilder/latest/userguide/what-is-image-builder.html
