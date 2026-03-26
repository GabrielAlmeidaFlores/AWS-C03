# EC2 Placement Groups

---

## Core Objective

**Why does this exist?** By default, AWS distributes EC2 instances across underlying hardware to minimize correlated failures — but this default behavior sacrifices control over **network latency** and **fault tolerance topology**. Placement Groups give you explicit control over how instances are physically placed on the underlying hardware, allowing you to optimize for either:

- **Maximum network throughput / minimum latency** (tightly coupled, HPC workloads), or
- **Maximum resilience / fault isolation** (distributed, failure-resistant architectures).

> **Exam lens:** Placement Groups are a *zero-cost* feature — you pay only for the instances, not the group itself. AWS may ask you to identify the correct group type for a given workload scenario.

**AWS Reference:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/placement-groups.html

---

## Fundamental Concepts

### The Three Placement Group Types

| Feature | **Cluster** | **Spread** | **Partition** |
|---|---|---|---|
| **Physical scope** | Single AZ, same rack (same hardware) | Single or multi-AZ, different racks | Single or multi-AZ, isolated rack partitions |
| **Max instances** | No hard limit (practical: ~hundreds) | **7 instances per AZ per group** | 7 partitions per AZ; hundreds of instances per partition |
| **Network performance** | Up to **25 Gbps** between instances (Enhanced Networking required) | Standard | Standard |
| **Fault domain** | Single point of failure (same rack = same power/network switch) | Each instance on distinct rack = max isolation | Each partition is an independent fault domain |
| **Primary use case** | HPC, tightly coupled apps, low-latency MPI jobs | Small critical instance counts (e.g., 3 HA masters) | Large distributed systems: HDFS, HBase, Cassandra, Kafka |
| **AZ flexibility** | ❌ Single AZ only | ✅ Multi-AZ | ✅ Multi-AZ |

---

### Key Terminology

- **Rack:** A physical unit of hardware in an AWS data center, with its own power and network. Rack-level failure is the standard blast radius model.
- **Partition:** A logical grouping of instances that maps to a **dedicated set of racks**. AWS guarantees no two partitions share racks.
- **Enhanced Networking (ENA):** A prerequisite for achieving Cluster group's low-latency, high-throughput benefits. Uses SR-IOV to reduce CPU overhead and jitter.
- **Capacity reservation:** Launching into a Placement Group increases the probability of `InsufficientCapacityError` because you're constraining placement. Plan accordingly.

---

## Procedure or Logic

### How Each Type Works Internally

#### Cluster Placement Group
- AWS schedules all instances **within the same rack** (or adjacent racks in the same AZ on the same physical spine switch).
- Result: instance-to-instance traffic traverses minimal network hops → sub-millisecond latency, maximum PPS.
- **Risk:** The rack is a shared failure domain. A rack power failure can take down all instances simultaneously.

#### Spread Placement Group
- AWS enforces a **hard constraint**: each instance must land on a **distinct underlying hardware rack**.
- Max 7 instances per AZ is a hard AWS limit (not a soft limit) — it's bounded by the guarantee of distinct racks.
- Ideal for: Active-passive pairs, quorum nodes (e.g., ZooKeeper, etcd), bastion hosts where loss of any one instance is critical.

#### Partition Placement Group
- AWS divides the group into **N partitions** (max 7 per AZ). Each partition = isolated rack set.
- Instances within a partition **may share racks with each other** but are **never** on the same racks as another partition.
- Applications can query the **instance metadata service (IMDS)** to discover their partition number:
  ```
  GET http://169.254.169.254/latest/meta-data/placement/partition-number
  ```
- This allows topology-aware replication: your app can deliberately spread replicas across partitions.

---

### Decision Logic (Which Group to Use?)

```
Is your workload tightly coupled with inter-node communication?
  └─ YES → Is low latency / high throughput the priority?
              └─ YES → Cluster Placement Group
  └─ NO  → How many critical instances do you need to isolate?
              └─ Small count (≤7/AZ), max isolation → Spread Placement Group
              └─ Large distributed system (Kafka, HDFS, Cassandra) → Partition Placement Group
```

---

## Practical Application / Examples

### Example 1 — Cluster: HPC Financial Risk Simulation
A quantitative finance firm runs Monte Carlo simulations on a 64-node MPI cluster. Inter-node communication requires sub-millisecond latency and 10+ Gbps bandwidth.

**Solution:** Deploy all 64 `c5n.18xlarge` instances (Enhanced Networking enabled) into a **Cluster Placement Group** in `us-east-1a`. The tight physical placement ensures MPI collective operations complete without network jitter.

**Tradeoff acknowledged:** All 64 nodes share a failure domain. The firm accepts this risk because the workload is stateless and can be re-run.

---

### Example 2 — Spread: Three-Node HA Database Cluster
A production PostgreSQL primary + 2 standbys must survive any single rack failure without data loss or manual failover.

**Solution:** Place each of the 3 instances in a **Spread Placement Group**. AWS guarantees each node is on a separate rack. A single rack failure takes down at most 1 node, preserving quorum.

**Constraint:** You cannot add an 8th instance to a Spread group in the same AZ — this is a hard limit.

---

### Example 3 — Partition: Apache Kafka Cluster (300 brokers)
A streaming platform runs 300 Kafka brokers. Kafka replication factor is 3; it must ensure no two replicas of the same partition land on the same rack.

**Solution:** Use a **Partition Placement Group** with 3 partitions in `us-east-1a`. Deploy ~100 brokers per partition. Configure Kafka's `broker.rack` property to the partition number (retrieved via IMDS). Kafka's rack-aware replica assignment then guarantees each of the 3 replicas lands in a different partition (rack set).

```bash
# On each broker, retrieve partition number at launch via user data:
PARTITION=$(curl -s http://169.254.169.254/latest/meta-data/placement/partition-number)
echo "broker.rack=partition-${PARTITION}" >> /etc/kafka/server.properties
```

**AWS Reference (Kafka on AWS):** https://docs.aws.amazon.com/whitepapers/latest/building-data-lakes/apache-kafka.html

---

## Critical Considerations

- **`InsufficientCapacityError`:** The tighter the placement constraint (especially Cluster), the more likely you'll hit capacity limits. **Best practice:** Launch all instances in the group simultaneously in a single request to maximize AWS's ability to fulfill the constraint.

- **Instance type homogeneity (Cluster):** While not strictly required, mixing instance families in a Cluster group can reduce the likelihood of achieving the tightest physical placement. Use the same instance type/family when possible.

- **No live migration across groups:** You cannot move a running instance into or between Placement Groups. You must stop → modify → start. Plan for the maintenance window.

- **AMI and instance type compatibility:** Not all instance types support Placement Groups. Older generation types (e.g., `t2`) are excluded. Always verify against the compatibility list.
  - **AWS Reference:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/placement-groups.html#concepts-placement-groups

- **Spread hard limit = 7/AZ:** This is frequently tested. A Spread group cannot be used to isolate 8+ instances per AZ. If you need more, Partition is the answer.

- **Placement Groups are regional, not global:** A single Placement Group cannot span AWS Regions. Multi-AZ is supported only by Spread and Partition, not Cluster.

- **No charge, but no SLA:** AWS does not provide a specific latency or throughput SLA for Placement Groups. The benefits are best-effort based on physical proximity.

- **Dedicated Instances / Dedicated Hosts:** Placement Groups are compatible with Dedicated Instances. For Dedicated Hosts, **only Cluster** Placement Groups are supported.
  - **AWS Reference:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/dedicated-hosts-overview.html

---

## Critical Synthesis Note

> **High-level insight:** Placement Groups expose a fundamental tension in distributed systems design: **the CAP theorem applied to infrastructure topology**. A Cluster group maximizes *performance* (P) by colocating nodes, but sacrifices *partition tolerance* at the hardware level (a rack failure partitions the entire cluster). A Spread group maximizes *availability* (A) through fault domain isolation, but caps horizontal scale. A Partition group is the architectural bridge — it is essentially a rack-aware sharding primitive that lets the *application layer* make intelligent replication decisions, pushing infrastructure topology awareness up the stack.

> **Knowledge gap / further research for SAA-C03:** The exam often conflates Placement Groups with **Capacity Reservations** and **Launch Templates**. Investigate how **On-Demand Capacity Reservations (ODCR)** interact with Placement Groups — specifically, whether reserving capacity in a Cluster group guarantees physical proximity or only logical capacity. Additionally, explore how **AWS Nitro System** underpins the performance claims of Cluster groups, as this context helps eliminate distractor answers on latency-related scenario questions.

> **Cross-disciplinary connection:** The Partition Placement Group's topology-aware metadata pattern mirrors the **rack-awareness** feature in Apache Hadoop's HDFS and Kafka — both of which were designed around the same physical failure model (rack = failure unit). Understanding this convergence helps you reason about *why* Partition groups exist and which AWS-managed services (EMR, MSK) implicitly use this model.

---

**Primary AWS Documentation:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/placement-groups.html

**EC2 Instance Types supporting Placement Groups:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/placement-groups.html#concepts-placement-groups

**Enhanced Networking on EC2:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/enhanced-networking.html

**EC2 Pricing (Placement Groups are free):** https://aws.amazon.com/ec2/pricing/
