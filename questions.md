# EC2 Practice Questions

---

## Question 1

**How do you change the EC2 instance type in the AWS console?**

| # | Answer |
|---|---|
| A | By doing a right-click and select "Instance settings" then select "Change instance type" |
| B | By stopping the EC2 instance, then doing a right-click and select "Instance settings" then select "Change instance type". Finally, start the EC2 instance |
| C | By terminating the instance and launching a new one with the desired instance type |
| D | By using the AWS CLI command `aws ec2 modify-instance-attribute --instance-type` while the instance is running |

<details>
<summary><strong>Answer</strong></summary>

---

✅ **Correct: B** — By stopping the EC2 instance, then doing a right-click and select "Instance settings" then select "Change instance type". Finally, start the EC2 instance.

EC2 does not allow instance type changes while the instance is running. The hypervisor must release the current hardware allocation before AWS can reassign the instance to a different hardware class or size. The mandatory sequence is: **Stop → change type → Start**. Attempting to skip the stop step is not permitted — the console greys out the option entirely if the instance is in a running state.

---

❌ **A — Why it's wrong:**
The same navigation path (`Instance settings → Change instance type`) exists, but it is only accessible when the instance is in a **stopped** state. Attempting this on a running instance will either block the action or return an error. The instance type is a static configuration bound to the underlying host — it cannot be hot-swapped like network interfaces or security groups.

❌ **C — Why it's wrong:**
Terminating an instance is a **destructive and irreversible action** — the instance and its ephemeral storage are permanently deleted. While launching a replacement with a different type is technically possible, it is not "changing the instance type"; it creates an entirely new instance with a new ID, new private IP, and potentially different configuration. The correct approach preserves the existing instance.

❌ **D — Why it's wrong:**
`aws ec2 modify-instance-attribute --instance-type` is a valid CLI command but it has the same requirement as the console: **the instance must be stopped first**. Running it against a live instance returns an error (`IncorrectInstanceState`). The CLI does not bypass the stop requirement; it enforces the same constraint as the console.

</details>

---

## Question 4

**You have installed the Unified CloudWatch Agent on an EC2 instance to collect custom metrics. You want to monitor individual processes running on your EC2 instance and their system utilization. What would you use?**

| # | Answer |
|---|---|
| A | Configure Unified CloudWatch Agent with StatsD protocol |
| B | Configure Unified CloudWatch Agent with collectd protocol |
| C | Configure Unified CloudWatch Agent with procstat plugin |
| D | Configure Unified CloudWatch Agent with Embedded Metrics Format (EMF) |

<details>
<summary><strong>Answer</strong></summary>

---

✅ **Correct: C** — Configure Unified CloudWatch Agent with procstat plugin.

The **procstat plugin** is the CloudWatch Agent feature purpose-built for **per-process metrics**. It identifies processes by name, PID file, or pattern match, then emits metrics such as `cpu_usage`, `memory_rss`, `read_bytes`, and `write_bytes` scoped to each process. This is the only option in the CloudWatch Agent that delivers individual process-level visibility.

**AWS Reference:** https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Agent-procstat-process-metrics.html

---

❌ **A — Why it's wrong:**
The **StatsD protocol** enables applications to push arbitrary custom metrics to the CloudWatch Agent over UDP (default port 8125). It is an application-level instrumentation protocol — your code calls `statsd.increment("orders.processed")` and the agent forwards it to CloudWatch. It has no awareness of OS processes and cannot report system-level utilization.

❌ **B — Why it's wrong:**
The **collectd protocol** allows the CloudWatch Agent to receive metrics from the collectd daemon, which collects system-wide OS statistics (CPU, memory, network, disk). While collectd has its own process plugin, it is not a native CloudWatch feature — it requires a separately installed and configured collectd daemon. For process monitoring directly through the CloudWatch Agent without additional tooling, procstat is the correct and AWS-native choice.

❌ **D — Why it's wrong:**
**Embedded Metrics Format (EMF)** is a JSON specification that allows applications to embed CloudWatch metric data inside structured log events. It is a logging-based instrumentation pattern for application-emitted metrics — not a mechanism for observing OS-level process activity. It cannot introspect running processes or report their CPU or memory consumption.

</details>

---

## Question 6

**A company has EC2 instances running in a private subnet in a VPC without an Internet Gateway or NAT Gateway. They want secure SSH access without exposing instances to the Internet. Which AWS service should they use?**

| # | Answer |
|---|---|
| A | SSM Session Manager |
| B | EC2 Instance Connect Endpoint |
| C | Client VPN |
| D | AWS Direct Connect |

<details>
<summary><strong>Answer</strong></summary>

---

✅ **Correct: B** — EC2 Instance Connect Endpoint.

**EC2 Instance Connect Endpoint** creates a private, VPC-resident endpoint that tunnels SSH/RDP connections from the AWS console or CLI to instances in private subnets — with no IGW, NAT Gateway, bastion host, or public IP required. The connection travels entirely over the AWS network, and IAM policies control who can initiate sessions. It is the purpose-built solution for SSH access to fully private instances.

**AWS Reference:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connect-using-eice.html

---

❌ **A — Why it's wrong:**
**SSM Session Manager** provides shell access through the SSM service but does **not** use SSH protocol — it opens a browser-based or CLI-proxied shell session over HTTPS. Additionally, SSM requires the SSM Agent to reach the SSM service endpoints, which either needs an IGW/NAT or three VPC Interface Endpoints (`ssm`, `ssmmessages`, `ec2messages`). Without those endpoints already in place, SSM will not work in a fully isolated VPC. The question specifies SSH access, which SSM Session Manager does not provide.

❌ **C — Why it's wrong:**
**Client VPN** establishes a TLS-encrypted tunnel between a client device and the VPC, but it requires: (1) a Client VPN endpoint backed by an internet-facing infrastructure, (2) certificate provisioning, and (3) the VPN client software on each workstation. It is operationally heavy and designed for broad network access, not targeted SSH connectivity to private instances.

❌ **D — Why it's wrong:**
**AWS Direct Connect** is a dedicated physical network connection between an on-premises data center and AWS — provisioned in weeks, costing hundreds to thousands of dollars per month. It is a network connectivity product for hybrid architectures, not a mechanism for SSH access to individual EC2 instances.

</details>

---

**AWS Reference — EC2 Instance Connect Endpoint:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connect-using-eice.html
**AWS Reference — CloudWatch Agent procstat plugin:** https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Agent-procstat-process-metrics.html
**AWS Reference — Changing instance type:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/resize-instances.html
