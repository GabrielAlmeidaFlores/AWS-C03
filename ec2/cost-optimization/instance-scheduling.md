# EC2 Instance Scheduling

---

## Core Objective

**Why does this exist?** EC2 instances accrue compute charges every hour they run — including nights, weekends, and holidays when no one is using them. For non-production environments (dev, test, staging), this idle time is pure waste. A `t3.large` running 24/7 costs roughly 3× more than the same instance running only during business hours.

Instance scheduling automates start/stop cycles so instances run only when needed. The mechanism is straightforward: a time-based trigger invokes the EC2 `StartInstances` or `StopInstances` API on a defined schedule. No agents, no application changes, no instance modification required.

> **Exam lens:** SAA-C03 tests scheduling primarily through the lens of cost optimization. The native AWS mechanism is **EventBridge Scheduler** invoking EC2 API calls directly. For larger fleets with complex schedule requirements, **AWS Instance Scheduler** (an AWS Solutions Library deployment) is the managed answer. The exam also distinguishes this from Auto Scaling — scheduling is for predictable on/off cycles, not demand-based capacity management.

**AWS Reference:** https://docs.aws.amazon.com/scheduler/latest/UserGuide/what-is-scheduler.html

---

## Fundamental Concepts

### Key Terminology

| Term | Definition |
|---|---|
| **EventBridge Scheduler** | A fully managed AWS scheduler that triggers actions (API calls, Lambda, etc.) on cron or rate expressions; the native mechanism for EC2 start/stop automation |
| **AWS Instance Scheduler** | An AWS Solutions Library CloudFormation deployment that provides tag-based, timezone-aware scheduling for EC2 and RDS instances at fleet scale |
| **Cron expression** | A time pattern (`cron(0 8 ? * MON-FRI *)`) specifying exact times for schedule triggers; supports days of week, months, and specific times |
| **Rate expression** | A simpler interval pattern (`rate(1 hour)`); less useful for business-hours scheduling but suitable for periodic tasks |
| **Flexible time window** | An EventBridge Scheduler option that distributes invocations across a window (e.g., within 15 minutes of the target time) to reduce API call bursts across large fleets |
| **IAM execution role** | The role EventBridge Scheduler assumes to call the EC2 API; must have `ec2:StartInstances` and `ec2:StopInstances` permissions |

---

### Scheduling Mechanisms Compared

| Mechanism | Best For | Setup Complexity | Fleet Scale | Timezone Support |
|---|---|---|---|---|
| **EventBridge Scheduler** | Individual instances or small groups; native, no extra infrastructure | Low — console or CLI | Moderate (one schedule per target group) | ✅ Yes |
| **AWS Instance Scheduler** | Large fleets with varied schedules; tag-driven | Medium — CloudFormation stack | ✅ High — tag any instance to add it | ✅ Yes |
| **Lambda + EventBridge Rules** | Custom logic (conditional start, health checks before stop) | High — custom code | Any | ✅ Yes |

For most teams: **EventBridge Scheduler** for small setups, **AWS Instance Scheduler** for org-wide fleet management.

---

## Procedure or Logic

### EventBridge Scheduler: Start/Stop a Dev Instance on Business Hours

- **Step 1 — Create an IAM role** that EventBridge Scheduler can assume, with permission to call `ec2:StartInstances` and `ec2:StopInstances` on the target instances.

- **Step 2 — Create a schedule to start the instance** at the beginning of the business day using a cron expression scoped to weekdays.

- **Step 3 — Create a schedule to stop the instance** at the end of the business day.

- **Step 4 — Verify**: Check that the instance state transitions on the expected schedule. Use EventBridge Scheduler's invocation metrics to confirm delivery.

```bash
# Step 1 — Create the IAM role for EventBridge Scheduler
aws iam create-role \
  --role-name EventBridgeSchedulerEC2Role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "scheduler.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

aws iam put-role-policy \
  --role-name EventBridgeSchedulerEC2Role \
  --policy-name EC2StartStopPolicy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": ["ec2:StartInstances", "ec2:StopInstances"],
      "Resource": "arn:aws:ec2:us-east-1:123456789012:instance/i-0abc1234567890def"
    }]
  }'

# Step 2 — Schedule: start at 08:00 UTC Mon-Fri
aws scheduler create-schedule \
  --name "dev-instance-start" \
  --schedule-expression "cron(0 8 ? * MON-FRI *)" \
  --schedule-expression-timezone "America/New_York" \
  --target '{
    "Arn": "arn:aws:scheduler:::aws-sdk:ec2:startInstances",
    "RoleArn": "arn:aws:iam::123456789012:role/EventBridgeSchedulerEC2Role",
    "Input": "{\"InstanceIds\": [\"i-0abc1234567890def\"]}"
  }' \
  --flexible-time-window '{"Mode": "OFF"}'

# Step 3 — Schedule: stop at 20:00 UTC Mon-Fri
aws scheduler create-schedule \
  --name "dev-instance-stop" \
  --schedule-expression "cron(0 20 ? * MON-FRI *)" \
  --schedule-expression-timezone "America/New_York" \
  --target '{
    "Arn": "arn:aws:scheduler:::aws-sdk:ec2:stopInstances",
    "RoleArn": "arn:aws:iam::123456789012:role/EventBridgeSchedulerEC2Role",
    "Input": "{\"InstanceIds\": [\"i-0abc1234567890def\"]}"
  }' \
  --flexible-time-window '{"Mode": "OFF"}'
```

---

### AWS Instance Scheduler: Tag-Based Fleet Scheduling

For large fleets, deploy the Instance Scheduler stack and apply schedules via tags:

```bash
# After deploying the Instance Scheduler CloudFormation stack:

# Tag an instance to apply a schedule (schedule name defined in DynamoDB)
aws ec2 create-tags \
  --resources i-0abc1234567890def \
  --tags Key=Schedule,Value=office-hours

# The "office-hours" schedule is defined in the Instance Scheduler DynamoDB table:
# Mon-Fri, 08:00-20:00, timezone: America/New_York
# Any instance tagged Schedule=office-hours follows this schedule automatically
```

---

## Practical Application / Examples

### Scenario: Dev/Test Fleet Cost Reduction

A company runs 20 `t3.large` dev instances and 10 `m5.xlarge` test instances in `us-east-1`. All run 24/7 but are only used Mon–Fri, 08:00–18:00 (10 hours/day).

**Current cost (24/7):**
```
t3.large:  $0.0832/hr × 24hr × 365 days × 20 instances = ~$14,576/year
m5.xlarge: $0.192/hr  × 24hr × 365 days × 10 instances = ~$16,819/year
Total: ~$31,395/year
```

**Scheduled cost (50 hours/week):**
```
Running hours: 10hr/day × 5 days × 52 weeks = 2,600 hr/year (vs. 8,760)
t3.large:  $0.0832 × 2,600 × 20 = ~$4,326/year
m5.xlarge: $0.192  × 2,600 × 10 = ~$4,992/year
Total: ~$9,318/year  →  savings: ~$22,000/year (70% reduction)
```

**Implementation:** Deploy AWS Instance Scheduler via CloudFormation. Define one `office-hours` schedule (Mon–Fri, 08:00–18:00, US/Eastern). Tag all 30 instances with `Schedule=office-hours`. Done — no per-instance configuration, new instances automatically inherit the schedule on tagging.

---

## Critical Considerations

- **Scheduled stop does not protect unsaved work**: If a developer has an uncommitted database transaction or an unsaved in-memory state at stop time, it is lost. Communicate schedules clearly and consider a 5-minute warning via SNS before the stop fires.

- **EventBridge Scheduler requires an IAM execution role**: Unlike CloudWatch Events (predecessor), EventBridge Scheduler does not use a resource-based policy — it requires an explicit IAM role with `sts:AssumeRole` from `scheduler.amazonaws.com`. Missing or misconfigured roles cause silent schedule failures.

- **Scheduling is not Auto Scaling**: Scheduled stop/start manages a fixed instance on a time-based trigger. It does not add or remove capacity based on load. If the question involves variable demand, the answer is Auto Scaling with scheduled actions — not instance scheduling.

- **EBS costs continue while the instance is stopped**: Stopping an instance eliminates compute charges but EBS volumes, Elastic IPs (unattached), and data transfer charges continue. EBS is ~10% of typical instance cost — savings are still significant.

- **Instances with pending patching or running jobs need protection**: A naive stop schedule can terminate a long-running batch job mid-execution. Add a pre-stop check (Lambda inspecting a CloudWatch metric or a custom tag `DoNotStop=true`) if instances run variable-duration workloads.

- **Timezone handling is critical for global teams**: A schedule defined in UTC with no timezone context behaves unexpectedly for teams in other regions. Both EventBridge Scheduler and AWS Instance Scheduler support explicit timezone configuration — always set it explicitly.

---

## Critical Synthesis Note

> **High-level insight — Instance scheduling is the highest-ROI cost optimization for non-production environments:**
>
> Reserved Instances and Savings Plans reduce per-hour cost. Scheduling reduces billable hours. For a dev fleet that genuinely runs only 30% of the time it's provisioned, scheduling delivers a 70% cost reduction with zero architectural change — far exceeding the 30–40% discount of a 1-year RI. The two approaches compose: schedule + RI/Savings Plan on the running hours compounds the savings further.

> **Knowledge gap requiring further research**: The interaction between AWS Instance Scheduler and EC2 instances that are part of an Auto Scaling Group — specifically whether the scheduler's stop action conflicts with ASG desired capacity and triggers an immediate replacement — is not clearly documented for all ASG configurations.

---

## SAA-C03 Exam Focus

- **EventBridge Scheduler = native mechanism** for automated EC2 start/stop; uses cron expressions and an IAM execution role targeting the EC2 API directly.
- **AWS Instance Scheduler = tag-based fleet solution** — when the question describes many instances with varying schedules managed centrally, this is the answer.
- **Scheduling ≠ Auto Scaling** — scheduling is for predictable time-based on/off; Auto Scaling is for demand-based capacity. Don't confuse scheduled scaling actions (ASG feature) with instance scheduling (stop/start).
- **Stopped instances still incur EBS charges** — compute billing stops, but storage does not.
- **IAM execution role is required** for EventBridge Scheduler — it cannot call EC2 APIs without one.
- **70% cost reduction is achievable** for dev/test fleets running only business hours — this arithmetic is the justification the exam expects you to recognize as correct for this pattern.

---

**Primary AWS Documentation (EventBridge Scheduler):** https://docs.aws.amazon.com/scheduler/latest/UserGuide/what-is-scheduler.html

**AWS Instance Scheduler Solution:** https://aws.amazon.com/solutions/implementations/instance-scheduler-on-aws/

**EventBridge Scheduler Targets (EC2):** https://docs.aws.amazon.com/scheduler/latest/UserGuide/managing-targets-universal.html

**EC2 Instance Lifecycle:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-lifecycle.html
