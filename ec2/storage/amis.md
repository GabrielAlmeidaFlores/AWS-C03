# Amazon Machine Images (AMIs)

---

## Core Objective

**Why does this exist?** Launching an EC2 instance from scratch — installing the OS, configuring software, applying security hardening, installing agents — is slow, error-prone, and inconsistent at scale. AMIs solve this by capturing a complete instance blueprint: the root volume state, block device mappings, launch permissions, and virtualization metadata in a single reusable artifact.

The result: identical, pre-configured instances launch in seconds rather than minutes, with zero configuration drift between them.

> **Exam lens:** Key distinctions tested are **EBS-backed vs. instance store-backed** (can you stop it? can you recover it?), **AMIs are region-specific** (must copy to use elsewhere), the **no-reboot option** (consistency vs. availability tradeoff), and the **Golden AMI pattern** as the correct answer for launching pre-configured fleets consistently.

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

### How an AMI Is Created

- **Step 1 — Configure a source instance**: Launch an instance, install and configure OS, software, agents, and hardening. This is the baseline that will be baked into the AMI.
- **Step 2 — Create the AMI**: AWS snapshots the root volume (and any mapped EBS volumes) and registers the AMI with those snapshot IDs embedded in the block device mapping. The AMI ID is region-specific.
- **Step 3 — Launch instances from the AMI**: Any instance launched from this AMI gets an exact copy of the root volume (via snapshot restore) with the pre-configured software state.
- **Step 4 — Copy to other regions if needed**: AMIs cannot be launched in a region where they don't exist. Copying replicates both the AMI metadata and the underlying snapshots; the AMI gets a new ID in the target region.

---

### The No-Reboot Option

By default, AWS **reboots the instance** before snapshotting it. The reboot flushes all in-memory write buffers and brings the filesystem to a clean, consistent state.

The **no-reboot option** skips this reboot — the snapshot is taken while the instance continues running. It is available in both the **AWS Console** (uncheck "Reboot instance" in the Create Image dialog) and the **AWS CLI**.

| Behavior | Default (reboot) | No-reboot |
|---|---|---|
| **Instance rebooted** | ✅ Yes — before snapshot | ❌ No — snapshot taken live |
| **Filesystem consistency** | ✅ Guaranteed | ⚠️ Not guaranteed |
| **Instance downtime** | Brief (reboot duration) | None |
| **Safe for databases** | ✅ Yes | ❌ No — open transactions may be captured mid-write |
| **Safe for stateless app servers** | ✅ Yes | ✅ Yes |

Use no-reboot only on stateless instances where no in-flight writes exist. Never use it for databases or any instance with open write transactions.

**AWS Reference:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/creating-an-ami-ebs.html

---

### Sharing AMIs Cross-Account

AMI sharing is controlled via **launch permissions**. The owner retains full control; the recipient can launch instances from the AMI without it being copied to their account.

| Permission Level | Who Can Launch | How to Set |
|---|---|---|
| **Private** (default) | Owner only | No action needed |
| **Shared** | Specific AWS account IDs | Console: Edit AMI permissions → add account IDs |
| **Public** | Anyone with an AWS account | Console: Edit AMI permissions → set to Public |

**Critical distinction — AMI sharing ≠ snapshot sharing:**
Sharing an AMI lets the recipient **launch instances** from it. It does **not** grant access to the underlying EBS snapshots. If the recipient needs to **copy** the AMI to their own account, the owner must separately share the underlying snapshots.

**Encrypted AMIs — additional requirement:**
If the AMI's snapshots are encrypted with a **customer-managed KMS key (CMK)**, the owner must also grant the recipient account access to that KMS key via a key policy. Without this, the recipient cannot decrypt the volume during instance launch even if the AMI is shared.

**AWS Reference:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/sharingamis-explicit.html

---

## Practical Application / Examples

### Scenario: Golden AMI Pipeline for a Hardened Application Fleet

A security team requires all EC2 instances to launch with CIS-hardened Amazon Linux 2023, CloudWatch Agent, SSM Agent, and a specific app version baked in. Manual installs are banned.

```
CodePipeline (weekly trigger)
  └── EC2 Image Builder
        ├── Base: AWS-provided Amazon Linux 2023 AMI
        ├── Component: CIS Level 1 hardening
        ├── Component: CloudWatch Agent + config
        ├── Component: App v2.3 install
        └── Output: Golden AMI (us-east-1)
              └── AMI copy → eu-west-1, ap-southeast-1
                    └── SSM Parameter Store: /golden-ami/latest = ami-xxxxxxxx
                          └── Launch Templates reference SSM parameter
                                └── ASGs always launch latest golden AMI
```

**Outcome**: Every instance in every region launches from an identical, tested, hardened image. AMI version is updated weekly with zero changes to Launch Templates.

---

## Critical Considerations

- **Deregistering an AMI does not delete its snapshots**: After deregistration, the underlying EBS snapshots remain and continue to incur storage costs. Delete them explicitly if the AMI is no longer needed.

- **AMIs are region-specific**: A Launch Template in `eu-west-1` cannot reference an AMI from `us-east-1`. The AMI ID changes on copy even if the content is identical.

- **No-reboot risks filesystem inconsistency**: Skipping the reboot means in-flight writes may not be flushed to disk. Safe only for stateless instances with no open write transactions.

- **Public AMIs can expose sensitive data**: If a snapshot contained credentials or application secrets before the AMI was made public, that data is accessible to anyone. Audit AMI content and permissions before sharing.

- **AMI sharing ≠ snapshot sharing**: Sharing an AMI grants launch rights only. The recipient cannot copy the AMI to their account without explicit snapshot sharing. Encrypted AMIs also require KMS key policy grants.

- **EC2 Image Builder is the production-grade AMI pipeline**: For teams building AMIs regularly, Image Builder provides versioning, testing, automated distribution, and compliance scanning. Ad-hoc manual AMI creation does not scale.

- **AMI IDs are not portable**: The same AMI content has a different ID in every region after copy. Hardcoding AMI IDs in infrastructure code causes cross-region launch failures — use SSM Parameter Store to abstract AMI references.

---

## Critical Synthesis Note

> **High-level insight — An AMI is a deployment artifact, not a backup tool:**
>
> The operational role of an AMI is identical to a Docker image or a VM template: it is an immutable, versioned, deployable unit. Treating AMIs as instance backups conflates two distinct concerns — instance recovery (handled by EBS snapshots + Auto Recovery) and configuration management (handled by AMIs). A well-run org maintains a Golden AMI pipeline entirely separate from its backup strategy.

> **Knowledge gap requiring further research**: The behavior of Launch Templates referencing an SSM parameter for AMI ID when the parameter is updated — specifically whether running ASG instances are replaced automatically or only on the next scale-out event — warrants verification for zero-downtime Golden AMI rollout strategies.

---

## SAA-C03 Exam Focus

- **AMIs are region-specific** — always copy before using in another region; the AMI ID changes on copy.
- **EBS-backed = stoppable; instance store-backed = not stoppable** — instance store data is lost on stop or terminate.
- **Deregister ≠ delete snapshots** — deregistering an AMI leaves its snapshots intact; delete them separately to stop incurring costs.
- **Golden AMI pattern** — when the question asks how to ensure all instances launch with the same pre-configured software/hardening, the answer is Golden AMI (not User Data, which runs at boot and is slower/less consistent).
- **No-reboot** — skips the pre-snapshot reboot; speeds up AMI creation but risks filesystem inconsistency; only safe for stateless instances.
- **AMI sharing ≠ snapshot sharing** — recipient can launch but cannot copy the AMI without explicit snapshot sharing; encrypted AMIs also require KMS key access.

---

**Primary AWS Documentation:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html
**EBS-Backed vs Instance Store-Backed:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/RootDeviceStorage.html
**Creating an AMI from an Instance:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/creating-an-ami-ebs.html
**Copying an AMI:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/CopyingAMIs.html
**Sharing an AMI:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/sharingamis-explicit.html
**EC2 Image Builder:** https://docs.aws.amazon.com/imagebuilder/latest/userguide/what-is-image-builder.html
