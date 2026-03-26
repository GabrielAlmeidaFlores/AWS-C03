# CloudWatch for EC2

---

## Core Objective

**Why does this exist?** EC2 instances are opaque compute resources by default — without instrumentation, you have no visibility into whether a running instance is healthy, saturated, or silently failing. CloudWatch solves the **observability gap** between AWS infrastructure events and operational awareness by providing metrics collection, log aggregation, threshold-based alerting, and automated remediation — all scoped to EC2 resources.

The ultimate goal: give operators **actionable, real-time visibility** into the performance, health, and resource utilization of EC2 instances — enabling proactive scaling, failure detection, and automated recovery without manual intervention.

> **Exam lens:** A critical distinction tested repeatedly is **what CloudWatch monitors by default vs. what requires the CloudWatch Agent**. Memory utilization, disk space, and swap usage are **not** available as native EC2 metrics — they require the Agent because AWS has no hypervisor-level access to the OS internals of your instance.

**AWS Reference:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/monitoring_ec2.html

---

## Fundamental Concepts

### Key Terminology

| Term | Definition |
|---|---|
| **Metric** | A time-ordered set of data points representing a measurable value (e.g., `CPUUtilization`) published to CloudWatch at a fixed interval |
| **Namespace** | A logical container for metrics; all native EC2 metrics live in the `AWS/EC2` namespace; CloudWatch Agent metrics default to `CWAgent` |
| **Dimension** | A name/value pair that uniquely identifies a metric (e.g., `InstanceId=i-0abc123`); allows filtering metrics to a specific instance, ASG, or AMI |
| **Basic Monitoring** | Default EC2 monitoring; metrics published every **5 minutes**; free of charge |
| **Detailed Monitoring** | Opt-in EC2 monitoring; metrics published every **1 minute**; incurs additional cost; required for responsive Auto Scaling |
| **CloudWatch Agent** | A software agent installed on the EC2 instance OS that collects OS-level metrics (memory, disk, processes) and log files and publishes them to CloudWatch |
| **Alarm** | A CloudWatch construct that watches a single metric and transitions between `OK`, `ALARM`, and `INSUFFICIENT_DATA` states based on defined thresholds |
| **Status Check** | An automated EC2 health test run every minute; produces the `StatusCheckFailed` metric family; distinct from CloudWatch Alarms |
| **Log Group** | A CloudWatch Logs container that holds log streams from one or more EC2 instances (e.g., `/var/log/messages`, application logs) |
| **Log Stream** | A sequence of log events from a single source within a Log Group (typically one stream per instance per log file) |
| **Metric Filter** | A pattern applied to a Log Group that extracts numeric values from log events and publishes them as a CloudWatch metric |
| **Auto Recovery** | A CloudWatch Alarm action that automatically stops and restarts an impaired EC2 instance on new hardware when a system status check fails |

---

### Monitoring Tiers: What CloudWatch Sees

| Metric Category | Source | Available by Default | Requires Agent |
|---|---|---|---|
| CPU Utilization | Hypervisor | ✅ Yes | ❌ No |
| Network In / Out (bytes) | Hypervisor | ✅ Yes | ❌ No |
| Disk Read / Write (ops, bytes) — **instance store only** | Hypervisor | ✅ Yes | ❌ No |
| EBS Read / Write (ops, bytes) | EBS service | ✅ Yes | ❌ No |
| Status Check metrics | EC2 service | ✅ Yes | ❌ No |
| **Memory (RAM) utilization** | OS kernel | ❌ No | ✅ Yes |
| **Disk space used / free** (EBS, NVMe) | OS kernel | ❌ No | ✅ Yes |
| **Swap utilization** | OS kernel | ❌ No | ✅ Yes |
| **Per-process CPU / memory** | OS kernel | ❌ No | ✅ Yes |
| **Application log files** | OS filesystem | ❌ No | ✅ Yes |

---

### EC2 Status Checks — Two Distinct Layers

| Check Type | What It Tests | Failure Cause | Who Fixes It |
|---|---|---|---|
| **System Status Check** (`StatusCheckFailed_System`) | AWS underlying infrastructure (host hardware, network, power) | Hardware failure, AWS network disruption | **AWS** (or Auto Recovery) |
| **Instance Status Check** (`StatusCheckFailed_Instance`) | Guest OS reachability (kernel, filesystem, networking config) | Misconfigured OS, full disk, kernel panic | **You** (reboot, SSH, SSM) |

---

## Procedure or Logic

### Setting Up Full EC2 Observability (End-to-End)

- **Step 1 — Enable Detailed Monitoring**: Enable at instance launch or on a running instance. Required for 1-minute granularity metrics in Auto Scaling policies.
  ```bash
  aws ec2 monitor-instances --instance-ids i-0abc1234567890def
  ```

- **Step 2 — Install and Configure the CloudWatch Agent**: Required for memory, disk, and log collection. Use SSM Run Command or User Data to deploy at scale.
  ```bash
  # Install on Amazon Linux 2 / AL2023
  sudo yum install -y amazon-cloudwatch-agent

  # Generate config interactively
  sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
  ```

- **Step 3 — Apply the Agent Configuration**: Store the config in SSM Parameter Store for reuse across a fleet. Start the agent.
  ```bash
  sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
    -a fetch-config \
    -m ec2 \
    -c ssm:/cloudwatch-agent/config \
    -s
  ```

- **Step 4 — Create CloudWatch Alarms**: Define alarms on key metrics with appropriate thresholds and evaluation periods. Wire alarm actions to SNS, Auto Scaling, or EC2 Auto Recovery.

- **Step 5 — Configure Log Groups**: Ensure the Agent config maps each log file on the instance to a dedicated Log Group with a meaningful retention policy (default: never expire — incurs unbounded cost).

- **Step 6 — Build a Dashboard**: Create a CloudWatch Dashboard grouping all EC2 metrics (CPU, memory, disk, network, status checks) for a fleet or individual instance for at-a-glance operational visibility.

---

### CloudWatch Agent Configuration (Key Metrics Block)

```json
{
  "metrics": {
    "namespace": "CWAgent",
    "metrics_collected": {
      "mem": {
        "measurement": ["mem_used_percent"],
        "metrics_collection_interval": 60
      },
      "disk": {
        "measurement": ["used_percent"],
        "metrics_collection_interval": 60,
        "resources": ["/", "/data"]
      },
      "cpu": {
        "measurement": ["cpu_usage_idle", "cpu_usage_user", "cpu_usage_system"],
        "metrics_collection_interval": 60,
        "totalcpu": true
      },
      "swap": {
        "measurement": ["swap_used_percent"],
        "metrics_collection_interval": 60
      }
    }
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/messages",
            "log_group_name": "/ec2/system/messages",
            "log_stream_name": "{instance_id}",
            "retention_in_days": 30
          },
          {
            "file_path": "/var/log/app/application.log",
            "log_group_name": "/ec2/app/application",
            "log_stream_name": "{instance_id}",
            "retention_in_days": 90
          }
        ]
      }
    }
  }
}
```

---

### Creating an EC2 CloudWatch Alarm via CLI

```bash
# Alarm: trigger if CPU > 80% for 3 consecutive 1-minute periods
aws cloudwatch put-metric-alarm \
  --alarm-name "ec2-high-cpu-i-0abc123" \
  --alarm-description "CPU utilization exceeded 80% for 3 minutes" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --dimensions Name=InstanceId,Value=i-0abc1234567890def \
  --statistic Average \
  --period 60 \
  --evaluation-periods 3 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:ops-alerts \
  --ok-actions arn:aws:sns:us-east-1:123456789012:ops-alerts \
  --treat-missing-data breaching
```

### Creating an EC2 Auto Recovery Alarm via CLI

```bash
# Alarm: automatically recover instance if system status check fails for 2 consecutive minutes
aws cloudwatch put-metric-alarm \
  --alarm-name "ec2-auto-recover-i-0abc123" \
  --alarm-description "Auto-recover instance on system status check failure" \
  --metric-name StatusCheckFailed_System \
  --namespace AWS/EC2 \
  --dimensions Name=InstanceId,Value=i-0abc1234567890def \
  --statistic Minimum \
  --period 60 \
  --evaluation-periods 2 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions arn:aws:automate:us-east-1:ec2:recover
```

---

## Practical Application / Examples

### Scenario: Observability Stack for a Production Web Fleet Behind an Auto Scaling Group

A production web application runs on an ASG of `t3.large` instances behind an ALB. The ops team needs: CPU/memory alerting, automatic instance recovery on hardware failure, log centralization, and a real-time dashboard.

**Architecture:**

```
EC2 Fleet (ASG)
  ├── CloudWatch Agent → CWAgent namespace (memory, disk, swap)
  ├── CloudWatch Agent → Log Groups (/ec2/app/nginx, /ec2/app/application)
  └── AWS/EC2 namespace (CPU, network, status checks) ← native, no agent needed

CloudWatch Alarms:
  ├── CPUUtilization > 70% (3 min) → SNS → PagerDuty
  ├── mem_used_percent > 85% (5 min) → SNS → PagerDuty
  ├── disk used_percent > 90% on "/" → SNS → PagerDuty
  ├── StatusCheckFailed_System ≥ 1 (2 min) → ec2:recover (Auto Recovery)
  └── StatusCheckFailed_Instance ≥ 1 (5 min) → SNS → ops-alerts

CloudWatch Dashboard:
  └── Widgets: CPU%, mem%, disk%, NetworkIn/Out, StatusCheckFailed per instance
```

**Key decisions:**

1. **Detailed Monitoring enabled** on all ASG instances so the Auto Scaling policy responds within 1 minute of a CPU spike — not 5.
2. **Agent config stored in SSM Parameter Store** (`/cloudwatch-agent/config`) and applied via a Launch Template User Data script, ensuring every new ASG instance self-configures on boot.
3. **Log Group retention set to 90 days** for application logs, 30 days for system logs — explicitly configured in the Agent config to prevent indefinite log accumulation.
4. **`treat-missing-data breaching`** on the status check alarms — if an instance stops reporting (e.g., kernel panic prevents metric publication), the alarm immediately triggers rather than staying `OK`.

**Outcome**: An ASG of 20 instances with full CPU, memory, disk, and log observability, zero manual SSH-based diagnosis for hardware failures (Auto Recovery handles it), and centralized logs queryable via CloudWatch Logs Insights.

---

## Critical Considerations

- **Memory and disk are not native metrics — this is the most common production oversight**: New EC2 environments frequently go live without the CloudWatch Agent, leaving operators blind to memory exhaustion and full-disk conditions. Both failure modes are silent at the hypervisor level and produce no native AWS/EC2 metric. Install the Agent as part of the base AMI or Launch Template, not as an afterthought.

- **Detailed Monitoring is required for sub-5-minute Auto Scaling responsiveness**: Basic Monitoring publishes data every 5 minutes. An ASG using Basic Monitoring can take up to **15 minutes** to respond to a CPU event (5-min metric delay + evaluation periods). Enable Detailed Monitoring on any instance participating in a latency-sensitive Auto Scaling group.

- **Auto Recovery has strict instance type constraints**: EC2 Auto Recovery (the `ec2:recover` alarm action) is supported only on instances using **EBS root volumes** running in a **VPC**. It is **not supported** on instance store-backed instances, bare metal instances, or instances in EC2-Classic. Verify support before relying on it as a HA mechanism.
  - **AWS Reference:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-recover.html

- **Auto Recovery preserves instance identity; reboot does not reset state**: A recovered instance retains its `InstanceId`, private/public IP addresses, Elastic IPs, EBS volumes, and IAM role. It is migrated to new host hardware. In-memory state (RAM) is lost, but disk state is preserved.

- **`treat-missing-data` default is `missing` (not `breaching`)**: By default, an alarm stays in its current state if metric data stops arriving. For status check alarms, a crashed instance stops publishing metrics — meaning the alarm will never transition to `ALARM` unless you explicitly set `--treat-missing-data breaching`. This is a dangerous default for health-monitoring alarms.

- **Log Group retention defaults to "Never expire"**: Every log event stored indefinitely is a cost that compounds. Always set an explicit retention policy. Use 30–90 days for operational logs, longer only for compliance-mandated audit logs stored in a cost-optimized tier.

- **CloudWatch Agent IAM permissions**: The EC2 instance profile must include `CloudWatchAgentServerPolicy` (AWS managed) for the Agent to publish metrics and logs. Without it, the Agent silently fails to publish — metrics simply never appear in the console, with no obvious error on the instance.

- **Alarm evaluation periods vs. data points to alarm**: The `--evaluation-periods` parameter defines the window; `--datapoints-to-alarm` (optional, defaults to `evaluation-periods`) defines how many data points within that window must breach the threshold. Setting `datapoints-to-alarm=2` out of `evaluation-periods=3` creates a "2 out of 3" condition — more resilient to transient spikes than requiring 3 consecutive breaches.

- **Per-instance vs. ASG-level alarms**: CloudWatch alarms on `InstanceId` dimension monitor a single instance. For fleet-level visibility, create alarms on the `AutoScalingGroupName` dimension (aggregated across all instances in the ASG) — critical for capacity management decisions.

---

## Critical Synthesis Note

> **High-level insight — CloudWatch is not a monitoring tool for EC2; it is the control plane feedback loop for EC2 automation:**
>
> The operational framing of CloudWatch as "a place to see graphs and set alerts" misses its architectural role. Every EC2 automation primitive — **Auto Scaling** (scale out/in), **Auto Recovery** (self-healing), **EC2 Lifecycle Hooks** (drain before termination), and **EventBridge rules** (event-driven remediation) — is triggered by CloudWatch metric thresholds or state transitions. CloudWatch is not observing EC2; it is the **feedback loop that makes EC2 self-managing**.
>
> The implication for architecture: the quality of your CloudWatch instrumentation directly determines the quality of your automation. An ASG without memory metrics cannot scale on memory pressure. An instance without a status check alarm cannot self-recover. The investment in Agent deployment, metric granularity, and alarm design is not an operational overhead — it is a prerequisite for any EC2 architecture that claims to be self-healing or cost-optimized.

> **Knowledge gap requiring further research**: The interaction between **CloudWatch Metric Math** and ASG scaling policies for composite scaling decisions (e.g., scale out only when `CPUUtilization > 70% AND mem_used_percent > 80%`) is not well-documented with concrete production examples. Target Tracking scaling policies natively support only a single metric. Achieving multi-metric composite scaling requires Step Scaling with a Metric Math expression as the alarm metric — a pattern that warrants dedicated benchmarking to validate scaling responsiveness under real workload conditions.

---

**Primary AWS Documentation:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/monitoring_ec2.html

**CloudWatch Agent Setup for EC2:** https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Install-CloudWatch-Agent.html

**CloudWatch Agent Configuration Reference:** https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Agent-Configuration-File-Details.html

**EC2 Auto Recovery:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-recover.html

**EC2 Status Checks:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/monitoring-system-instance-status-check.html

**CloudWatch Alarms:** https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html

**Detailed Monitoring for EC2:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-cloudwatch-new.html

**CloudWatch Logs with EC2:** https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/QuickStartEC2Instance.html

**IAM Role for CloudWatch Agent:** https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/create-iam-roles-for-cloudwatch-agent.html
