# EC2 Status Checks

---

## Core Objective

**Why does this exist?** EC2 instances can fail in two fundamentally different ways: the **AWS infrastructure** beneath them can fail (hardware faults, network disruption, power loss), or the **guest OS** inside them can fail (kernel panic, full disk, misconfigured network stack). Without a systematic health probe, both failure modes are invisible until an end-user complaint surfaces them.

EC2 Status Checks are an automated, zero-configuration health monitoring system that runs every minute on every instance. They expose two orthogonal dimensions of instance health — **system-level** (AWS's responsibility) and **instance-level** (your responsibility) — through CloudWatch metrics that drive automated remediation via Auto Recovery or operational alerting.

The ultimate goal: **eliminate silent failures** by making EC2 instance health machine-readable and actionable without requiring a monitoring agent or manual SSH-based diagnosis.

> **Exam lens:** The key distinction tested on SAA-C03 is *who owns the fix*: a **System Status Check** failure is AWS's problem (hardware/infrastructure) and is resolved by stopping/starting the instance (migrating to new hardware) or triggering Auto Recovery. An **Instance Status Check** failure is your problem (OS/software) and requires you to SSH/SSM in and diagnose.

**AWS Reference:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/monitoring-system-instance-status-check.html

---

## Fundamental Concepts

### Key Terminology

| Term | Definition |
|---|---|
| **System Status Check** | Automated test that verifies the AWS-managed underlying infrastructure (hardware, host network, power, hypervisor) hosting the instance is operating normally |
| **Instance Status Check** | Automated test that verifies the guest OS and software on the instance can receive and process network traffic |
| **Attached EBS Status Check** | Automated test (for EBS-backed instances) that verifies attached EBS volumes are reachable and functioning |
| `StatusCheckFailed_System` | CloudWatch metric (0 or 1) published every minute; value of 1 indicates system status check failure |
| `StatusCheckFailed_Instance` | CloudWatch metric (0 or 1) published every minute; value of 1 indicates instance status check failure |
| `StatusCheckFailed_AttachedEBS` | CloudWatch metric (0 or 1); value of 1 indicates one or more attached EBS volumes failed their status check |
| `StatusCheckFailed` | Combined metric; value of 1 if **either** the system or instance check fails |
| **Auto Recovery** | EC2 feature (or CloudWatch Alarm action `ec2:recover`) that automatically stops and restarts a failing instance on new underlying hardware when a system status check fails |
| **Instance Reboot** | A restart that keeps the instance on the same hardware; resolves OS-level failures but does **not** migrate to new hardware |
| **Stop/Start** | Terminates the current host assignment and provisions the instance on new hardware; the canonical manual fix for a system status check failure |
| **Scheduled Event** | An AWS-initiated notification of planned maintenance (`system-reboot`, `instance-retirement`) that can precede a status check failure |
| **CloudWatch Alarm** | A CloudWatch construct that watches a single metric over a defined evaluation window and transitions between `OK`, `ALARM`, and `INSUFFICIENT_DATA` states based on a threshold |
| **Evaluation Period** | The number of consecutive `period`-length data point windows CloudWatch uses to determine alarm state; alarm triggers only when the threshold is breached across the required number of periods |
| **`datapoints-to-alarm`** | Optional parameter enabling M-of-N evaluation: the alarm triggers when M out of N evaluation periods breach the threshold — more resilient to transient spikes than requiring N consecutive breaches |
| **`treat-missing-data`** | Controls alarm state when no metric data arrives during an evaluation period; critical for status check alarms where a crashed instance stops publishing metrics entirely |
| **EC2 Alarm Action** | A special ARN (`arn:aws:automate:<region>:ec2:<action>`) that CloudWatch can call directly to perform an EC2 lifecycle operation without requiring SNS or Lambda as an intermediary |

---

### The Three Check Types

| Check | What It Probes | Failure Cause | Resolution Owner | Automated Fix |
|---|---|---|---|---|
| **System Status Check** | AWS host hardware, hypervisor, host network, power supply | Hardware failure, host network disruption, AWS infrastructure event | **AWS** | Stop/Start or `ec2:recover` Auto Recovery |
| **Instance Status Check** | Guest OS network stack, kernel, filesystem, startup configuration | Kernel panic, full root disk, misconfigured startup scripts, exhausted file descriptors | **You** | Reboot (soft failure), SSH/SSM diagnosis |
| **Attached EBS Status Check** | Connectivity and I/O functionality of attached EBS volumes | EBS volume impairment, degraded instance-to-EBS connectivity | **AWS** (volume side) or **You** (config) | Detach/reattach, EBS volume recovery |

---

### CloudWatch Metrics Produced

| Metric | Namespace | Dimension | Values | Granularity |
|---|---|---|---|---|
| `StatusCheckFailed_System` | `AWS/EC2` | `InstanceId` | 0 (pass) / 1 (fail) | Every 1 minute — always |
| `StatusCheckFailed_Instance` | `AWS/EC2` | `InstanceId` | 0 (pass) / 1 (fail) | Every 1 minute — always |
| `StatusCheckFailed_AttachedEBS` | `AWS/EC2` | `InstanceId` | 0 (pass) / 1 (fail) | Every 1 minute — always |
| `StatusCheckFailed` | `AWS/EC2` | `InstanceId` | 0 (pass) / 1 (fail) | Every 1 minute — always |

Status check metrics are **always at 1-minute granularity** regardless of whether Basic or Detailed Monitoring is enabled. They do not require the CloudWatch Agent and are available from the moment the instance enters the `running` state.

---

### Automated Recovery Options

| Option | Trigger | Behavior | Preserves After Recovery |
|---|---|---|---|
| **Auto Recovery (CloudWatch Alarm)** | `StatusCheckFailed_System ≥ 1` | Stops instance → AWS migrates to new hardware → starts instance | Instance ID, private/public IPs, EIPs, EBS volumes, IAM role, placement group |
| **EC2 Reboot (Alarm Action `ec2:reboot`)** | `StatusCheckFailed_Instance ≥ 1` | Reboots OS on same hardware | All persistent state; does **not** migrate hardware |
| **EC2 Auto Recovery (per-instance setting)** | `StatusCheckFailed_System` (built-in) | Equivalent behavior to alarm-triggered `ec2:recover` | Same as CloudWatch alarm-triggered recovery |
| **Manual Stop/Start** | Operator-initiated | Same hardware migration behavior as Auto Recovery | Same as Auto Recovery; in-memory RAM state is lost |

---

### CloudWatch Alarm States

Every CloudWatch Alarm is always in exactly one of three states:

| State | Meaning | Status Check Implication |
|---|---|---|
| `OK` | Metric is within the defined threshold | Status check is passing (metric value = 0) |
| `ALARM` | Metric has breached the threshold for the required number of evaluation periods | Status check is failing — automated action fires (once per state transition) |
| `INSUFFICIENT_DATA` | Not enough data points have arrived to evaluate the threshold | Occurs at alarm creation, or when the instance is stopped/terminated and metric flow stops |

The `INSUFFICIENT_DATA` state is distinct from a `missing data` event during evaluation. A newly created alarm starts in `INSUFFICIENT_DATA` until enough periods have elapsed to make a determination.

---

### EC2 Alarm Actions

CloudWatch alarms on EC2 status check metrics can trigger EC2 lifecycle operations directly via special automate ARNs — no SNS topic or Lambda function required.

| Action ARN | Effect | Supported On | Use Case |
|---|---|---|---|
| `arn:aws:automate:<region>:ec2:recover` | Stops and restarts instance on new hardware (preserves identity) | EBS-backed instances in a VPC only; not supported on bare metal or instance-store | System status check failure — hardware migration |
| `arn:aws:automate:<region>:ec2:reboot` | Sends OS reboot signal (same hardware) | All instance types | Instance status check failure — OS soft reset |
| `arn:aws:automate:<region>:ec2:stop` | Stops the instance (EBS state preserved, instance deallocated) | EBS-backed instances only | Cost control or compliance — stop on sustained failure rather than recover |
| `arn:aws:automate:<region>:ec2:terminate` | Terminates the instance permanently | All instance types | ASG-managed instances where termination triggers replacement |
| `arn:aws:sns:<region>:<account>:<topic>` | Publishes a notification to an SNS topic | All | Human alerting, Lambda invocation, PagerDuty integration |

**Multiple actions can be assigned to a single alarm.** A common pattern: assign both `ec2:recover` and an SNS notification to the system check alarm so Auto Recovery fires *and* the ops team is notified simultaneously.

---

### `treat-missing-data` Options

The `treat-missing-data` parameter controls what happens to alarm state when metric data stops arriving during an evaluation period. For status check alarms, this is critical: a crashed instance stops publishing metrics entirely.

| Value | Alarm State When Data Is Missing | Appropriate For Status Checks? |
|---|---|---|
| `missing` (default) | Alarm retains its current state — does **not** transition | ❌ **Dangerous** — a crashed instance never triggers `ALARM` |
| `breaching` | Treats missing data points as if the threshold was breached — alarm transitions to `ALARM` | ✅ **Correct** for all status check alarms |
| `notBreaching` | Treats missing data points as within threshold — alarm transitions to `OK` | ❌ Actively wrong for health monitoring |
| `ignore` | Alarm state is not evaluated for missing periods — stays in current state | ⚠️ Acceptable only during instance boot/initialization windows |

**Rule**: Every status check alarm must explicitly set `--treat-missing-data breaching`. The default `missing` behavior is indistinguishable from a passing check when the instance has crashed.

---

### Recommended Alarm Matrix

| Metric | Statistic | Period | Evaluation Periods | Datapoints to Alarm | treat-missing-data | Action |
|---|---|---|---|---|---|---|
| `StatusCheckFailed_System` | `Minimum` | 60s | 2 | 2 | `breaching` | `ec2:recover` + SNS |
| `StatusCheckFailed_Instance` | `Maximum` | 60s | 3 | 2 | `breaching` | SNS (pager) |
| `StatusCheckFailed_AttachedEBS` | `Maximum` | 60s | 2 | 2 | `breaching` | SNS (storage team) |
| `StatusCheckFailed` (combined) | `Maximum` | 60s | 2 | 2 | `breaching` | SNS (catch-all fallback) |

**Statistic choice rationale**: `Minimum` for system checks because a single 0-value data point confirms the check passed in that period and should reset evaluation. `Maximum` for instance and EBS checks because a single 1-value means the OS or volume was unresponsive at least once in that period — meaningful even if other data points in the same period were 0.

**AWS Reference:** https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html

---

## Procedure or Logic

### How Status Checks Execute

- **Step 1 — Automatic initiation**: AWS runs both system and instance checks every 60 seconds for every running EC2 instance. No configuration is required; checks begin immediately upon the instance entering the `running` state.

- **Step 2 — System check execution**: The EC2 host infrastructure layer probes hardware components — power supplies, host network interfaces, hypervisor health, and host memory accessible to the hypervisor. If any component reports a degraded state, the system check fails.

- **Step 3 — Instance check execution**: The EC2 service sends ARP requests and network packets to the instance's primary network interface. If the guest OS's network stack responds correctly, the check passes. Failure indicates the OS is unresponsive — either a kernel hang, boot failure, filesystem error, or system resource exhaustion preventing the network stack from processing packets.

- **Step 4 — Metric publication**: Both check results are published to CloudWatch every minute as 0/1 binary data points. `StatusCheckFailed` is the logical OR of both individual checks.

- **Step 5 — Alarm evaluation**: If a CloudWatch Alarm is configured on any status check metric, it evaluates the new data point against its threshold within the same minute window.

- **Step 6 — Automated action (if configured)**: Alarm transitions to `ALARM` state → triggers configured action: `ec2:recover`, `ec2:reboot`, `ec2:stop`, or an SNS notification. Without a configured alarm, the check failure is visible in the console but produces no automated response.

---

### Configuring Status Check Alarms (CLI)

```bash
# 1. Auto Recovery on system status check failure (2-of-2 consecutive failures)
#    Fires ec2:recover AND sends SNS notification simultaneously
aws cloudwatch put-metric-alarm \
  --alarm-name "ec2-auto-recover-i-0abc123" \
  --alarm-description "Recover instance on system status check failure" \
  --metric-name StatusCheckFailed_System \
  --namespace AWS/EC2 \
  --dimensions Name=InstanceId,Value=i-0abc1234567890def \
  --statistic Minimum \
  --period 60 \
  --evaluation-periods 2 \
  --datapoints-to-alarm 2 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions \
    arn:aws:automate:us-east-1:ec2:recover \
    arn:aws:sns:us-east-1:123456789012:ops-alerts \
  --treat-missing-data breaching
```

```bash
# 2. SNS alert on instance status check failure (2-of-3 M-of-N pattern)
#    Avoids false pages from transient OS spikes; triggers on 2 failures within a 3-minute window
aws cloudwatch put-metric-alarm \
  --alarm-name "ec2-instance-check-failed-i-0abc123" \
  --alarm-description "Instance status check failed — OS-level intervention required" \
  --metric-name StatusCheckFailed_Instance \
  --namespace AWS/EC2 \
  --dimensions Name=InstanceId,Value=i-0abc1234567890def \
  --statistic Maximum \
  --period 60 \
  --evaluation-periods 3 \
  --datapoints-to-alarm 2 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:ops-alerts \
  --treat-missing-data breaching
```

```bash
# 3. SNS alert on attached EBS status check failure (2-of-2 consecutive failures)
#    Monitor separately — StatusCheckFailed (combined) does NOT include this check
aws cloudwatch put-metric-alarm \
  --alarm-name "ec2-ebs-check-failed-i-0abc123" \
  --alarm-description "Attached EBS volume impaired — storage team intervention required" \
  --metric-name StatusCheckFailed_AttachedEBS \
  --namespace AWS/EC2 \
  --dimensions Name=InstanceId,Value=i-0abc1234567890def \
  --statistic Maximum \
  --period 60 \
  --evaluation-periods 2 \
  --datapoints-to-alarm 2 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:storage-alerts \
  --treat-missing-data breaching
```

```bash
# 4. Catch-all alarm on the combined StatusCheckFailed metric
#    Provides a single pane of glass alert when either system OR instance check fails
aws cloudwatch put-metric-alarm \
  --alarm-name "ec2-any-check-failed-i-0abc123" \
  --alarm-description "Any EC2 status check (system or instance) has failed" \
  --metric-name StatusCheckFailed \
  --namespace AWS/EC2 \
  --dimensions Name=InstanceId,Value=i-0abc1234567890def \
  --statistic Maximum \
  --period 60 \
  --evaluation-periods 2 \
  --datapoints-to-alarm 2 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:ops-alerts \
  --treat-missing-data breaching
```

---

### Querying Current Status Check State (CLI)

```bash
# Get status check results for a specific instance
aws ec2 describe-instance-status \
  --instance-ids i-0abc1234567890def \
  --query 'InstanceStatuses[*].{
    Instance:InstanceId,
    SystemCheck:SystemStatus.Status,
    InstanceCheck:InstanceStatus.Status
  }' \
  --output table

# List all instances with an impaired instance status check
aws ec2 describe-instance-status \
  --filters "Name=instance-status.status,Values=impaired" \
  --query 'InstanceStatuses[*].InstanceId' \
  --output text

# List all instances with an impaired system status check
aws ec2 describe-instance-status \
  --filters "Name=system-status.status,Values=impaired" \
  --query 'InstanceStatuses[*].InstanceId' \
  --output text

# Check for any scheduled events (reboot, retirement) on an instance
aws ec2 describe-instance-status \
  --instance-ids i-0abc1234567890def \
  --query 'InstanceStatuses[*].Events'
```

---

## Practical Application / Examples

### Scenario: Self-Healing Production Instance with Tiered Alert Response

A stateful application server (`m5.xlarge`, EBS-backed, running in a VPC) hosts a long-lived process. The ops team needs: automatic recovery from hardware failure with no manual intervention, and paged alerts when the OS fails so an engineer can diagnose the root cause.

**Architecture:**

```
EC2 Instance (m5.xlarge, EBS root, VPC)
  │
  ├── [Auto-healed] System Status Check → every minute
  │     └── CloudWatch Alarm: StatusCheckFailed_System ≥ 1 for 2 periods
  │           └── Action: arn:aws:automate:us-east-1:ec2:recover
  │                 └── Instance migrated to new hardware
  │                       Same ID / IPs / EBS volumes / IAM role retained
  │
  ├── [Paged] Instance Status Check → every minute
  │     └── CloudWatch Alarm: StatusCheckFailed_Instance ≥ 1 for 3 periods
  │           └── Action: SNS → PagerDuty (on-call engineer)
  │                 └── Engineer SSMs in, diagnoses OS failure
  │                       (disk full, kernel OOM, misconfigured init)
  │
  └── [Monitored] Attached EBS Status Check → every minute
        └── CloudWatch Alarm: StatusCheckFailed_AttachedEBS ≥ 1 for 2 periods
              └── Action: SNS → ops-alerts (storage team channel)
```

**Key decisions:**

1. **`evaluation-periods: 2` for System Recovery** — a single transient check failure is insufficient to justify a disruptive stop/start cycle. Two consecutive failures confirm genuine infrastructure impairment.
2. **`evaluation-periods: 3` for Instance Alert** — OS-level failures sometimes produce a brief recovery spike before stabilizing; three consecutive failures avoid false pages.
3. **`treat-missing-data: breaching` on all alarms** — a crashed instance stops publishing metrics. Without this, the alarm stays `OK` indefinitely despite the instance being unresponsive.
4. **Auto Recovery over ASG replacement** — for stateful processes, Auto Recovery preserves the instance's private IP and avoids client reconnection storms. ASG replacement creates a new instance with a new private IP.

**Outcome**: A hardware failure self-heals in under 5 minutes with no engineer involvement and no change to the instance's network identity. OS failures page on-call within 3 minutes of the first confirmed failure. EBS impairments alert the storage team independently of instance-level health.

---

## Critical Considerations

- **System check failure requires stop/start — not reboot**: A reboot keeps the instance on the same physical host. If the underlying hardware is the failure source, rebooting does not migrate the instance and will not resolve the system check. Only a **stop/start** (or Auto Recovery) triggers host migration to healthy hardware.

- **Auto Recovery has strict prerequisite constraints**: Auto Recovery requires an **EBS-backed root volume** and an instance running in a **VPC**. It is **not supported** on instance store-backed instances, bare metal instances (`*.metal`), or EC2-Classic. Verify support before relying on it as a HA mechanism.
  - **AWS Reference:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-recover.html

- **`treat-missing-data: breaching` is mandatory for health alarms**: The CloudWatch default is `missing` — the alarm stays in its current state when no data arrives. A crashed OS stops publishing status check metrics, so without `breaching`, the alarm never transitions to `ALARM` and no automated recovery triggers. This default is actively dangerous for status check alarms.

- **Status checks are always 1-minute granularity**: Unlike most EC2 metrics that default to 5-minute intervals under Basic Monitoring, status check metrics are published every minute without any opt-in. Detailed Monitoring and the CloudWatch Agent are both irrelevant to this cadence.

- **Instance check failure during boot is expected**: The instance check shows `initializing` for 1–20 minutes after launch while the OS starts up. Configure alarm evaluation windows to tolerate the initialization period or use the `treat-missing-data: ignore` option during instance launch automation. Do not alert on status check failures in the first few minutes post-launch.

- **Attached EBS failures are a distinct check — the combined metric may not capture them**: `StatusCheckFailed` is the logical OR of system and instance checks only; it does **not** include `StatusCheckFailed_AttachedEBS`. An impaired EBS volume can fail its check while `StatusCheckFailed` remains 0. Monitor all four metrics individually for complete coverage.

- **Auto Recovery preserves instance identity but loses in-memory state**: The recovered instance retains `InstanceId`, private/public IP addresses, Elastic IPs, EBS volumes, and IAM role. All **RAM state** (process memory, in-flight requests, caches) is lost. Applications that depend on in-memory state must either checkpoint to EBS or implement reconnection logic.

- **Status checks themselves are free**: EC2 status checks and the CloudWatch metrics they publish in the `AWS/EC2` namespace incur no charge. Only the CloudWatch Alarms consuming those metrics have a cost (~$0.10/alarm/month).

- **Use `datapoints-to-alarm` for M-of-N evaluation on instance checks**: Setting `--evaluation-periods 3 --datapoints-to-alarm 2` triggers the alarm on 2 failures within a 3-minute window rather than requiring 3 consecutive failures. This is more resilient to transient OS events (a single GC pause or brief CPU saturation can cause one check miss) while still catching genuine failures quickly. For system checks driving Auto Recovery, 2-of-2 is preferred — false recoveries are disruptive, so require strict consecutive confirmation.

- **Assign multiple actions to a single alarm — not multiple alarms**: A CloudWatch Alarm can hold multiple ARNs in `--alarm-actions`. Assigning both `ec2:recover` and an SNS ARN to the system check alarm means Auto Recovery fires *and* the ops team is notified in the same state transition. Creating two separate alarms on the same metric doubles alarm cost and introduces a race condition if one alarm fires before the other evaluates.

- **Alarm actions fire once per state transition, not once per data point**: If an alarm enters `ALARM` and the metric stays breached, the action fires exactly once — at the `OK → ALARM` transition. Auto Recovery fires once. A second recovery attempt only fires if the alarm returns to `OK` and breaches again. Design remediation logic around this: a single Auto Recovery does not loop or retry on its own.

- **Scheduled Events are an early warning signal before check failures**: AWS proactively notifies when underlying hardware is degrading via `describe-instance-status` events (`system-reboot`, `instance-retirement`). Polling for scheduled events — or subscribing to EventBridge events for EC2 Scheduled Events — allows you to act before the check actually fails. Do not rely exclusively on alarm-based detection.
  - **AWS Reference:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/monitoring-instances-status-check_sched.html

---

## Critical Synthesis Note

> **High-level insight — Status checks are the only health signal that does not require the guest OS to cooperate:**
>
> Every other EC2 health mechanism — CloudWatch Agent metrics, application health endpoints, ELB target group health checks — depends on the instance's OS being functional enough to run software. A kernel panic, OOM kill of a critical process, or corrupted init system can silence all of them simultaneously. EC2 Status Checks are the only mechanism that probes instance health from **outside the guest OS**: the system check probes at the hypervisor/hardware layer, and the instance check probes at the network interface layer before packets reach the OS kernel's TCP stack. This makes status checks the **last line of defense** for automated recovery in failure modes where the OS itself is the point of failure.
>
> The architectural implication: status check alarms should be treated as the *floor* of the health monitoring stack — always present, always configured with `treat-missing-data: breaching` and an automated action. Application-level health checks, CloudWatch Agent metrics, and load balancer probes sit *above* this floor and are only trustworthy when the status check confirms the underlying OS layer is functional.

> **Knowledge gap requiring further research**: The interaction between **EC2 Auto Recovery** and **stateful distributed locking** is not well-documented. When Auto Recovery migrates an instance to new hardware, any distributed lock, leader election lease, or cluster membership token held by that instance may not be released before the stop/start cycle completes. For systems using ZooKeeper, etcd, or DynamoDB-based locks, it is unclear whether there is a consistent release window between the stop and the subsequent start — and whether this creates a split-brain condition if a peer node has already acquired the lock during the recovery window. This warrants dedicated testing for any stateful workload that relies on Auto Recovery as its primary HA mechanism.

---

**Primary AWS Documentation:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/monitoring-system-instance-status-check.html

**EC2 Auto Recovery:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-recover.html

**Scheduled Instance Events:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/monitoring-instances-status-check_sched.html

**CloudWatch Alarms — Create and Edit:** https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html

**CloudWatch Alarm Actions for EC2:** https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/UsingAlarmActions.html

**treat-missing-data Reference:** https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html#alarms-and-missing-data

**describe-instance-status CLI Reference:** https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-instance-status.html

**EBS Volume Status Checks:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/monitoring-volume-status.html
