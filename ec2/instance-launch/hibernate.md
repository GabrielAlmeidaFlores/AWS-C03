# EC2 Hibernate

---

## Core Objective

**Why does this exist?** Stopping an EC2 instance discards all in-memory state — the OS must reboot, applications must reinitialize, and any in-memory caches or warm JVM/runtime state must be rebuilt from scratch. For workloads with long startup times (large in-memory datasets, pre-warmed caches, slow application bootstraps), this cold-start cost is paid on every stop/start cycle.

Hibernate solves this by writing the entire contents of RAM to the encrypted EBS root volume before suspending the instance. On resume, RAM is restored from disk — the OS does not reboot, processes pick up exactly where they left off, and startup time collapses to seconds regardless of application complexity.

> **Exam lens:** Hibernate is tested on three axes: the **prerequisites** (encrypted root EBS, hibernate enabled at launch, RAM ≤ 150 GB), the **key behavioral difference from Stop** (RAM is preserved on resume vs. lost on stop), and the **use case signal** — any question mentioning "long initialization," "pre-warmed cache," or "resume from exact state" points to Hibernate.

**AWS Reference:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/Hibernate.html

---

## Fundamental Concepts

### Key Terminology

| Term | Definition |
|---|---|
| **Hibernate** | An instance state where RAM contents are saved to the encrypted EBS root volume and the instance is suspended; resumed without an OS reboot |
| **Resume** | Starting a hibernated instance; RAM is restored from EBS, processes continue from their suspended state |
| **Cold start** | A normal stop/start cycle where the OS reboots and applications reinitialize from zero |
| **Hibernation Agent** | A software component (pre-installed on supported AWS AMIs) that handles writing RAM to EBS on hibernate and restoring it on resume |
| **Root volume encryption** | A mandatory prerequisite for hibernate — RAM contents (which may contain secrets) must be encrypted at rest on EBS |

---

### Hibernate vs. Stop vs. Reboot

| Dimension | **Hibernate** | **Stop** | **Reboot** |
|---|---|---|---|
| **RAM state** | ✅ Saved to EBS root volume | ❌ Lost | ✅ Preserved (no suspension) |
| **OS boots on resume** | ❌ No | ✅ Yes | ✅ Yes |
| **EBS root volume** | Persists (+ RAM dump written to it) | Persists | Persists |
| **Instance store** | ❌ Data lost | ❌ Data lost | ✅ Data preserved |
| **Public IP** | Released (same as Stop) | Released | Retained |
| **Private IP** | Retained | Retained | Retained |
| **Billing** | Stops (same as Stop) | Stops | Continues |
| **Use case** | Long-init apps, pre-warmed state | Normal maintenance, cost control | OS-level soft reset |

---

### Prerequisites and Constraints

| Requirement | Detail |
|---|---|
| **Root EBS must be encrypted** | Mandatory — RAM dump contains potentially sensitive data |
| **Root EBS must be large enough** | Must have free space ≥ RAM size to store the memory dump |
| **RAM limit** | ≤ 150 GB |
| **Max hibernation duration** | 60 days |
| **Must be enabled at launch** | Cannot enable hibernate on an already-running or stopped instance |
| **Instance store volumes** | Not supported — instance store-backed instances cannot hibernate |
| **Bare metal instances** | Not supported |
| **Supported instance families** | C3, C4, C5, M4, M5, R3, R4, R5, T2, T3 and variants — verify current list for newer families |

---

## Procedure or Logic

- **Step 1 — Enable hibernation at launch**: Hibernation must be configured in the instance launch options. It cannot be retrofitted onto a running instance.

- **Step 2 — Verify root volume is encrypted**: The EBS root volume must be encrypted (CMK or AWS-managed key). Without encryption, the hibernate request is rejected.

- **Step 3 — Hibernate the instance**: Issue a stop with the hibernate flag. The Hibernation Agent writes RAM to the root EBS volume, then the instance suspends. The instance state transitions: `running` → `stopping` → `stopped` (with hibernation marker).

- **Step 4 — Resume**: Start the instance normally. The hypervisor restores RAM from the EBS snapshot written during hibernate. The OS continues from its suspended state — no boot sequence, no application initialization.

```bash
# Launch an instance with hibernation enabled (requires encrypted root volume)
aws ec2 run-instances \
  --image-id ami-0abc1234567890def \
  --instance-type m5.xlarge \
  --hibernation-options Configured=true \
  --block-device-mappings '[{
    "DeviceName": "/dev/xvda",
    "Ebs": {
      "VolumeSize": 50,
      "Encrypted": true,
      "VolumeType": "gp3"
    }
  }]'

# Hibernate a running instance
aws ec2 stop-instances \
  --instance-ids i-0abc1234567890def \
  --hibernate

# Resume is a standard start — no special flag needed
aws ec2 start-instances \
  --instance-ids i-0abc1234567890def
```

---

## Practical Application / Examples

### Scenario: Pre-Warmed ML Inference Instance

A data science team runs an ML inference service on a `r5.4xlarge` that loads a 40 GB model into RAM on startup. Cold start takes 12 minutes. The service is only needed during business hours (08:00–20:00).

**Without Hibernate:**
```
08:00 — Start instance → 12-minute cold start → service available at 08:12
20:00 — Stop instance (RAM lost)
(12 minutes of dead time + user complaints every morning)
```

**With Hibernate:**
```
Launch once with hibernation enabled + encrypted root volume (≥ 60 GB free for RAM dump)
  └── After model is loaded and service is warm:
        20:00 — Hibernate (RAM written to EBS, billing stops)
        08:00 — Resume (RAM restored, service available in ~30 seconds)
```

**Outcome**: Service available 30 seconds after resume instead of 12 minutes. No billing during off-hours (same as Stop). The 40 GB model never reloads — it persists in RAM across the hibernate/resume cycle.

---

## Critical Considerations

- **Hibernation must be enabled at launch — no retrofit**: If an instance was launched without `--hibernation-options Configured=true`, it can never hibernate. Plan ahead; this cannot be changed on a running or stopped instance.

- **Root volume free space must exceed RAM size**: The Hibernation Agent writes the full RAM contents to the root EBS volume. If the root volume lacks sufficient free space, hibernate fails. Size the root volume as `OS size + RAM size + buffer`.

- **60-day maximum is a hard limit**: An instance cannot remain hibernated for more than 60 days. AWS will terminate it after this limit. Design workflows that rely on long-term suspension to account for this boundary.

- **Instance store data is lost on hibernate**: Like a normal stop, instance store volumes are wiped when the instance suspends. Only EBS volumes retain their state across hibernate/resume.

- **Public IP is released on hibernate**: A hibernated instance behaves like a stopped instance for IP allocation — the public IP is released unless an Elastic IP is attached. Applications depending on a stable public IP must use an EIP.

- **Not a replacement for Auto Scaling**: Hibernate is for preserving warm state on individual long-running instances. It does not help with fleet-level scale-in/out; ASG instances that are terminated lose their RAM state regardless.

---

## Critical Synthesis Note

> **High-level insight — Hibernate shifts the cost model for stateful, long-initializing workloads:**
>
> Stop/start is cost-free between runs (no compute billing) but pays a fixed initialization tax on every resume. Hibernate eliminates that tax at the cost of slightly higher EBS storage (RAM dump persists on the root volume). For any workload where startup time × resume frequency > EBS storage cost, Hibernate is strictly cheaper in total operational cost. The decision is arithmetic, not preference.

> **Knowledge gap requiring further research**: The behavior of distributed coordination mechanisms (e.g., ZooKeeper session timeouts, Kafka consumer group rebalances) when an instance hibernates for an extended period — and whether the session/lease expiry window during hibernation causes cluster membership churn that impacts a resume — is not documented with concrete timeout guidance.

---

## SAA-C03 Exam Focus

- **Hibernate = RAM saved to encrypted EBS root; resume = no OS reboot** — this is the core behavioral distinction from Stop.
- **Three mandatory prerequisites**: encrypted root EBS volume, hibernate enabled **at launch**, RAM ≤ 150 GB. Any question showing a hibernate failure traces to one of these.
- **Signal words in scenario questions**: "long initialization time," "pre-warmed cache," "resume from exact state," "avoid cold start" → Hibernate is the answer.
- **Public IP is released** on hibernate (same behavior as Stop) — use an Elastic IP if a stable public address is required across hibernate/resume cycles.
- **60-day max hibernation** — instances cannot hibernate indefinitely; AWS terminates them after 60 days.
- **Instance store data is lost** on hibernate — only EBS volumes survive the suspend/resume cycle.

---

**Primary AWS Documentation:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/Hibernate.html

**Hibernate Prerequisites:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/hibernating-prerequisites.html

**Instance Lifecycle:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-lifecycle.html
