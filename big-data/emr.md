# AWS EMR (Elastic MapReduce)

---

## Core Objective

AWS EMR addresses the operational complexity and infrastructure cost of running **large-scale distributed data processing workloads** — ETL pipelines, machine learning preprocessing, log analysis, and interactive analytics — that exceed the capacity of single-node systems and demand horizontal scalability.

The fundamental problem it solves: **provisioning, configuring, and scaling distributed compute clusters** (Hadoop, Spark, Hive, Presto, Flink) without managing raw EC2 fleets, dependency conflicts, or cluster orchestration software manually.

The ultimate goal: run petabyte-scale data processing jobs on a managed, ephemeral or persistent cluster — paying only for the EC2/EBS capacity consumed, with optional Spot Instance integration for up to 90% cost reduction.

---

## Fundamental Concepts

### Key Terminology

| Term | Definition |
|---|---|
| **Cluster** | A collection of EC2 instances (nodes) provisioned by EMR to run a distributed workload |
| **Primary Node** | The single master node that coordinates the cluster: manages YARN ResourceManager, HDFS NameNode, and job scheduling |
| **Core Node** | Worker nodes that both **store data** (HDFS) and **run tasks**; cannot be removed without risk of HDFS data loss |
| **Task Node** | Worker nodes that **only run tasks** — no HDFS storage; safe to add/remove dynamically (ideal for Spot Instances) |
| **EMR Studio** | Managed IDE environment for authoring and debugging Spark/PySpark jobs via Jupyter notebooks |
| **Bootstrap Action** | A script executed on every node **before** the application framework starts; used to install custom packages or configure the OS |
| **Step** | A discrete unit of work submitted to an EMR cluster (e.g., a Spark job JAR, a Hive script); steps execute sequentially by default |
| **EMRFS** | EMR File System — an S3-compatible HDFS implementation that enables clusters to read/write S3 directly as if it were HDFS |
| **Instance Fleet** | A cluster configuration that mixes multiple EC2 instance types to maximize Spot availability and minimize interruption risk |
| **Managed Scaling** | EMR feature that automatically adjusts Core and Task node counts based on YARN metrics (pending containers, cluster load) |

### Core Architecture Pillars

- **Compute Layer**: EC2 instances (On-Demand, Reserved, or Spot) managed by EMR; supports Graviton2/arm64 for cost-optimized workloads
- **Storage Layer**: Decoupled from compute — data lives in **S3 via EMRFS** (recommended) or on-cluster **HDFS** (ephemeral); HDFS is lost when the cluster terminates
- **Application Layer**: Pluggable open-source frameworks installed at cluster launch — Spark, Hadoop, Hive, HBase, Presto, Flink, Livy, JupyterHub
- **Coordination Layer**: YARN (Yet Another Resource Negotiator) schedules and allocates resources across nodes for all submitted jobs
- **Deployment Models**: **Transient clusters** (spin up → run job → terminate; cheapest) vs. **Persistent clusters** (long-running, always available; higher cost)

### Cluster Node Comparison

| Node Type | Runs Tasks | Stores HDFS | Spot-Safe | Scale In/Out |
|---|---|---|---|---|
| **Primary** | Partially (coordination) | NameNode metadata | ❌ No | ❌ Fixed (1 node) |
| **Core** | ✅ Yes | ✅ Yes (DataNode) | ⚠️ Risky | ⚠️ Carefully |
| **Task** | ✅ Yes | ❌ No | ✅ Yes | ✅ Freely |

---

## Procedure or Logic

### Transient Cluster Workflow (Recommended for Batch ETL)

- **Step 1 — Store input data in S3**: Organize raw data in S3 prefixes. EMR reads input via EMRFS; no data needs to be loaded into HDFS before processing.
- **Step 2 — Package your job**: Compile the Spark application into a JAR (Scala/Java) or prepare a PySpark script (`.py`). Upload artifacts to S3.
- **Step 3 — Define cluster configuration**: Select instance types for Primary, Core, and Task nodes. Configure Instance Fleet or Instance Group. Attach an EMR-managed IAM service role (`EMR_DefaultRole`) and EC2 instance profile (`EMR_EC2_DefaultRole`).
- **Step 4 — Add Steps**: Define one or more Steps referencing the job artifact in S3 along with the main class or script entry point.
- **Step 5 — Configure auto-termination**: Set `KeepJobFlowAliveWhenNoSteps=false` (API) so the cluster terminates automatically after all steps complete — eliminating idle cost.
- **Step 6 — Write output to S3**: Direct all job output to an S3 path. Results persist independently of cluster lifecycle.
- **Step 7 — Monitor via CloudWatch / EMR Console**: Track step status, YARN application logs, and node health. Enable logging to S3 for post-mortem debugging.

### Launching a Transient Cluster via AWS CLI

```bash
aws emr create-cluster \
  --name "nightly-etl-cluster" \
  --release-label emr-7.1.0 \
  --applications Name=Spark Name=Hadoop \
  --instance-type m5.xlarge \
  --instance-count 3 \
  --use-default-roles \
  --ec2-attributes KeyName=my-key-pair,SubnetId=subnet-xxxxxxxx \
  --log-uri s3://my-bucket/emr-logs/ \
  --steps Type=Spark,Name="ETL",ActionOnFailure=TERMINATE_CLUSTER,\
Args=[--deploy-mode,cluster,--class,com.example.Main,s3://my-bucket/job.jar] \
  --no-keep-job-flow-alive-when-no-steps
```

### Submitting a Spark Step to a Running Cluster

```bash
aws emr add-steps \
  --cluster-id j-XXXXXXXXXXXX \
  --steps Type=Spark,\
Name="ETL Job",\
ActionOnFailure=CONTINUE,\
Args=[\
  --deploy-mode,cluster,\
  --class,com.example.MainETL,\
  s3://my-bucket/jars/etl-job-1.0.jar,\
  s3://my-bucket/input/,\
  s3://my-bucket/output/\
]
```

### Enabling Managed Scaling

```bash
aws emr put-managed-scaling-policy \
  --cluster-id j-XXXXXXXXXXXX \
  --managed-scaling-policy \
    ComputeLimits='{
      "UnitType": "Instances",
      "MinimumCapacityUnits": 2,
      "MaximumCapacityUnits": 20,
      "MaximumOnDemandCapacityUnits": 5
    }'
```

---

## Practical Application / Examples

### Scenario: Daily Log Aggregation Pipeline at Scale

A platform engineering team needs to process 2 TB of raw application logs per day stored in S3, aggregate error metrics by service and region, and write Parquet output back to S3 for downstream Athena queries.

**Architecture:**
- **Trigger**: EventBridge Scheduler fires at `02:00 UTC` daily
- **Orchestration**: AWS Lambda invokes `aws emr create-cluster` with a pre-baked Step definition
- **Cluster**: 1× `m5.xlarge` Primary + 2× `r5.2xlarge` Core (On-Demand) + up to 10× `r5.2xlarge` Task (Spot, via Instance Fleet)
- **Job**: PySpark script reads raw JSON logs, filters `level=ERROR`, aggregates by `(service, region, hour)`, writes partitioned Parquet to `s3://analytics/errors/`

**PySpark job (simplified):**

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, date_trunc

spark = SparkSession.builder.appName("ErrorAggregation").getOrCreate()

df = spark.read.json("s3://raw-logs/app-logs/date=2024-03-22/")

result = (
    df.filter(col("level") == "ERROR")
      .withColumn("hour", date_trunc("hour", col("timestamp")))
      .groupBy("service", "region", "hour")
      .count()
      .withColumnRenamed("count", "error_count")
)

result.write \
    .mode("overwrite") \
    .partitionBy("service", "region") \
    .parquet("s3://analytics/errors/date=2024-03-22/")

spark.stop()
```

**Outcome**: Cluster spins up, processes 2 TB in ~18 minutes using Spot Task Instances, terminates automatically. Total compute cost: ~$1.40 vs. ~$14.00 on equivalent On-Demand instances — a 90% reduction. Output is immediately queryable via Athena without any additional pipeline steps.

---

## Critical Considerations

- **HDFS is ephemeral — always use S3 as the primary storage layer**: If a transient cluster terminates with data residing only on HDFS, that data is permanently lost. EMRFS (S3-backed) must be the canonical data layer for every production workload.
- **Spot Interruptions on Core nodes corrupt HDFS**: Task nodes are Spot-safe because they hold no HDFS blocks. Never place Core nodes exclusively on Spot without On-Demand fallback in an Instance Fleet; a Spot reclamation mid-job will trigger HDFS block replication failures and abort the entire job.
- **IAM dual-role architecture is mandatory**: EMR requires two distinct IAM entities — the **service role** (`EMR_DefaultRole`, used by the EMR control plane) and the **EC2 instance profile** (`EMR_EC2_DefaultRole`, used by applications running on nodes to access S3, DynamoDB, etc.). Conflating these or assigning insufficient permissions causes `AccessDenied` errors that are difficult to trace at runtime.
- **Bootstrap Actions run before framework initialization**: If a bootstrap action fails, the node is marked unhealthy and the cluster may fail to launch entirely. Wrap bootstrap scripts in robust error handling (`set -e`, explicit exit codes) and validate them independently before embedding in cluster configurations.
- **VPC subnet selection directly impacts Spot availability**: Spot capacity is AZ-specific. Use Instance Fleets with multiple subnets across AZs to maximize the probability of Spot fulfillment. Single-subnet clusters frequently face capacity shortages at scale during peak hours.
- **S3 eventual consistency is no longer a concern (post-2020)**: Amazon S3 provides strong read-after-write consistency. The EMRFS consistency view (a DynamoDB-backed consistency layer from earlier EMR versions) should be explicitly disabled on EMR 5.30+ to avoid unnecessary DynamoDB costs and latency.
- **Log aggregation requires explicit S3 log URI configuration**: Without `--log-uri`, container logs — critical for debugging Spark executor failures — are only accessible via SSH on the Primary node and become inaccessible after cluster termination.
- **EMR Serverless vs. EMR on EC2**: EMR Serverless eliminates cluster management but sacrifices configurability — no custom bootstrap actions, no AMI selection, limited runtime customization. For workloads requiring specific native libraries or OS-level dependencies, EMR on EC2 (or EMR on EKS) remains the correct choice.

---

## Critical Synthesis Note

**Architectural insight — EMR as a stateless transformation engine in a decoupled Lakehouse, not a monolithic data platform:**

The dominant anti-pattern is treating EMR as a self-contained big data platform: storing data on HDFS, running persistent clusters, managing a local Hive Metastore. This reintroduces the operational burden EMR was designed to eliminate and creates tight coupling between compute and storage that destroys cost efficiency.

The architecturally correct mental model is **EMR as an ephemeral, stateless transformation layer** operating over a durable S3-based Lakehouse:

```
S3 (raw)  →  EMR (transform/enrich)  →  S3 (curated, Parquet/Iceberg)  →  Athena / Redshift Spectrum (query)
```

This decoupling means compute scales independently of data volume, clusters carry zero state between runs, and the same curated S3 layer is simultaneously queryable by Athena (serverless ad-hoc SQL), Redshift Spectrum (high-concurrency BI), and SageMaker (ML feature engineering) — without any data duplication. EMR's role becomes narrowly defined: **transform and enrich**; all persistence and querying responsibilities are delegated to purpose-built services.

**Knowledge gap requiring further research**: The operational boundary between **EMR Serverless**, **EMR on EKS**, and **AWS Glue** for Spark workloads is not clearly delineated in AWS documentation in terms of cold-start latency, per-job overhead cost, and maximum job parallelism. All three services execute Apache Spark but differ significantly in startup time (Glue ~1–2 min, EMR Serverless ~15–30s pre-initialized, EMR on EC2 ~5–8 min cold), cost structures, and workload isolation guarantees. A rigorous benchmarking study across these three compute surfaces for standardized ETL workloads at varying data volumes (small/medium/large) is absent from public AWS documentation and represents a critical decision gap for platform architects selecting a managed Spark runtime.
