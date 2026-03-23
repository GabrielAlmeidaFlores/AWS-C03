# AWS Glue

---

## Core Objective

AWS Glue addresses the operational overhead of building and maintaining **data integration infrastructure** — the pipelines that discover, catalog, clean, transform, and move data between sources and analytical targets (S3, Redshift, RDS, DynamoDB, Kafka).

The fundamental problem it solves: **eliminating the need to provision and manage ETL servers**, write boilerplate data format conversion code, or manually maintain a central metadata catalog across a heterogeneous data ecosystem.

The ultimate goal: provide a fully managed, serverless platform where data engineers define transformation logic — and Glue handles execution, scaling, schema inference, job scheduling, and metadata governance — reducing pipeline development cycles from weeks to hours.

---

## Fundamental Concepts

### Key Terminology

| Term | Definition |
|---|---|
| **Data Catalog** | A centralized, persistent Apache Hive-compatible metadata repository storing database/table definitions, schemas, and partition information; queryable by Athena, EMR, and Redshift Spectrum |
| **Crawler** | An automated agent that scans a data source (S3, RDS, DynamoDB, JDBC), infers schema, detects partitions, and populates or updates the Data Catalog |
| **Classifier** | A rule (built-in or custom) used by Crawlers to identify the format of a data source (CSV, JSON, Parquet, Avro, ORC, XML) and extract its schema |
| **Glue Job** | A serverless execution unit that runs an ETL script (PySpark, Spark Scala, or Python Shell); billed per DPU-hour consumed |
| **DPU (Data Processing Unit)** | A unit of compute capacity for Glue jobs; 1 DPU = 4 vCPU + 16 GB RAM; jobs are allocated a configurable number of DPUs |
| **Glue Studio** | Visual drag-and-drop interface for building ETL pipelines; generates PySpark code that can be further customized |
| **DynamicFrame** | A Glue-native distributed data abstraction (analogous to a Spark DataFrame) that supports schema flexibility — fields can have inconsistent types across records without causing job failure |
| **Bookmark** | A state-tracking mechanism that records which data has already been processed, enabling incremental job runs without reprocessing previously seen records |
| **Trigger** | A scheduling or event-based mechanism that starts a Glue Job or Workflow; supports cron schedules, on-demand invocation, and event-based chaining |
| **Workflow** | An orchestration construct that chains multiple Glue Jobs and Crawlers with conditional branching and dependency tracking |
| **Glue DataBrew** | A visual data preparation tool for non-engineers; applies transformations (normalization, deduplication, format conversion) without writing code |
| **Glue Elastic Views** | Materializes and replicates data across stores (DynamoDB → S3 → Redshift) using SQL-defined views; keeps targets automatically synchronized |

### Core Architecture Pillars

- **Metadata Layer (Data Catalog)**: Central schema registry; a single source of truth for table definitions shared across Athena, EMR, and Redshift Spectrum — eliminating per-service schema duplication
- **Discovery Layer (Crawlers)**: Automated schema inference over S3 paths, JDBC sources, and NoSQL stores; runs on a schedule or on-demand
- **Transformation Layer (Glue Jobs)**: Serverless Apache Spark (PySpark / Scala) or Python Shell execution environment; scales from 2 to hundreds of DPUs dynamically
- **Orchestration Layer (Workflows / Triggers)**: DAG-based pipeline orchestration native to Glue; integrates with EventBridge and Step Functions for cross-service coordination
- **Pricing Model**: Billed per **DPU-hour** for ETL jobs (`$0.44/DPU-hour`); Crawlers billed per DPU-hour as well; Data Catalog charges apply per million objects stored/accessed beyond the free tier

### Glue Job Types Comparison

| Job Type | Runtime | Use Case | Min DPUs | Max DPUs |
|---|---|---|---|---|
| **Spark (PySpark / Scala)** | Apache Spark | Large-scale distributed ETL, complex transformations | 2 | 100+ |
| **Spark Streaming** | Structured Streaming | Near-real-time ETL from Kinesis or Kafka | 2 | 100+ |
| **Python Shell** | CPython 3.9 | Lightweight tasks, API calls, small file processing | 0.0625 (1/16) | 1 |
| **Ray** | Ray framework | ML preprocessing, parallelized Python workloads | 2 | 100+ |

---

## Procedure or Logic

### End-to-End ETL Pipeline Workflow

- **Step 1 — Define the source**: Identify the data source (S3 prefix, RDS table, JDBC endpoint, Kafka topic). Ensure the Glue IAM role has read access to that source.
- **Step 2 — Run a Crawler**: Configure a Crawler pointing to the source. Set a schedule (or run on-demand). The Crawler infers schema, detects partitions, and registers a table in the Glue Data Catalog.
- **Step 3 — Author the ETL Job**: Use Glue Studio (visual) or write a PySpark script directly. Read source data as a `DynamicFrame` via `glueContext.create_dynamic_frame.from_catalog()`. Apply transformations.
- **Step 4 — Apply transformations**: Use built-in Glue transforms (`ApplyMapping`, `Filter`, `Join`, `ResolveChoice`) or standard Spark DataFrame operations after converting with `.toDF()` / `.fromDF()`.
- **Step 5 — Write to target**: Write the output `DynamicFrame` to S3 (Parquet/ORC recommended), Redshift, RDS, or another catalog table via `glueContext.write_dynamic_frame.from_options()`.
- **Step 6 — Enable Job Bookmarks**: Activate bookmarks in job parameters (`--job-bookmark-option=job-bookmark-enable`) to ensure subsequent runs process only new or modified data.
- **Step 7 — Schedule via Trigger or Workflow**: Set a cron Trigger or wire the Job into a Workflow that chains Crawlers → Jobs → downstream Crawlers for schema refresh.
- **Step 8 — Monitor with CloudWatch**: Track `glue.driver.aggregate.elapsedTime`, DPU consumption, and record counts. Set alarms on job failures via CloudWatch Events → SNS.

### PySpark ETL Script Structure (Glue Job)

```python
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job

args = getResolvedOptions(sys.argv, ["JOB_NAME"])
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args["JOB_NAME"], args)

# Step 1: Read from Data Catalog
source_dyf = glueContext.create_dynamic_frame.from_catalog(
    database="raw_db",
    table_name="app_events",
    transformation_ctx="source_dyf"
)

# Step 2: Resolve ambiguous types (e.g., a field that is sometimes string, sometimes int)
resolved_dyf = ResolveChoice.apply(
    frame=source_dyf,
    choice="cast:string",
    transformation_ctx="resolved_dyf"
)

# Step 3: Map and rename columns
mapped_dyf = ApplyMapping.apply(
    frame=resolved_dyf,
    mappings=[
        ("event_id",   "string", "event_id",    "string"),
        ("user_id",    "string", "user_id",      "string"),
        ("event_type", "string", "event_type",   "string"),
        ("ts",         "string", "event_ts",     "timestamp"),
    ],
    transformation_ctx="mapped_dyf"
)

# Step 4: Write to S3 as Parquet, partitioned
glueContext.write_dynamic_frame.from_options(
    frame=mapped_dyf,
    connection_type="s3",
    connection_options={
        "path": "s3://curated-bucket/app_events/",
        "partitionKeys": ["event_type"]
    },
    format="parquet",
    transformation_ctx="sink"
)

job.commit()
```

### Creating a Crawler via AWS CLI

```bash
aws glue create-crawler \
  --name "raw-events-crawler" \
  --role "arn:aws:iam::123456789012:role/GlueServiceRole" \
  --database-name "raw_db" \
  --targets '{"S3Targets": [{"Path": "s3://raw-bucket/app_events/"}]}' \
  --schedule "cron(0 1 * * ? *)" \
  --schema-change-policy '{"UpdateBehavior":"UPDATE_IN_DATABASE","DeleteBehavior":"LOG"}'
```

---

## Practical Application / Examples

### Scenario: Centralizing Multi-Source Customer Data for a Data Warehouse Load

A data engineering team needs to consolidate customer records from three sources — an RDS PostgreSQL (transactional CRM), a DynamoDB table (behavioral events), and raw CSV uploads in S3 — into a unified Parquet dataset in S3, then load it into Redshift for BI reporting. The pipeline must run nightly and process only new records.

**Architecture:**

```
RDS (CRM)   ──┐
DynamoDB    ──┼──► Glue Crawlers ──► Data Catalog ──► Glue ETL Job ──► S3 (Parquet) ──► Redshift COPY
S3 (CSVs)   ──┘                                           ▲
                                                    Job Bookmarks
                                                  (incremental only)
```

**Key implementation decisions:**

1. **Crawlers** run at `00:30 UTC` to refresh schemas before the ETL job starts at `01:00 UTC` (chained via a Glue Workflow).
2. **`ResolveChoice`** handles the DynamoDB source where the `customer_id` field is inconsistently typed (`string` vs `number`) across records — a scenario that would crash a standard Spark job.
3. **Job Bookmarks** track the high-water mark on the RDS JDBC source and the S3 CSV prefix, ensuring only records ingested since the last run are processed.
4. **Output**: Partitioned Parquet by `ingestion_date`, written to `s3://curated/customers/`. A Redshift `COPY` command (triggered by a downstream Lambda) loads the new partition.

**Outcome**: Nightly pipeline processes only the delta (typically 50K–200K records vs. 40M total), completing in under 4 minutes on 10 DPUs — a 95% reduction in processing time and cost compared to a full-table scan approach.

---

## Critical Considerations

- **Crawlers do not guarantee schema correctness**: Crawlers infer schema from a sample of files. If data quality is inconsistent (mixed types, nullable fields, evolving schemas), the inferred schema will be wrong or incomplete. Always validate Crawler output against the actual data contract and consider defining tables manually via DDL for production sources.
- **DynamicFrame vs. DataFrame — know when to convert**: `DynamicFrame` tolerates schema inconsistencies but has a smaller API surface. Convert to a Spark `DataFrame` (`.toDF()`) for complex transformations (window functions, UDFs, multi-column joins) and convert back (`.fromDF()`) before writing. Avoid staying in `DynamicFrame` for logic that Spark handles natively.
- **Job Bookmarks have edge cases**: Bookmarks track state by job run, not by record timestamp. If a job fails mid-run and is restarted, bookmarks may skip or reprocess records depending on the source connector. Always validate idempotency of your write target (use `mode("overwrite")` on partitioned S3 paths or upsert logic in Redshift).
- **DPU allocation directly determines cost and performance**: Over-provisioning DPUs is the most common source of runaway Glue costs. Start with the minimum (`--number-of-workers 2`) and profile via the Glue job metrics dashboard before scaling up. Use **auto-scaling** (`--enable-auto-scaling`) for jobs with variable data volumes.
- **Cold start latency is non-trivial**: Glue Spark jobs incur a ~1–2 minute startup overhead for container provisioning and Spark context initialization. Glue is unsuitable for sub-minute latency requirements. For low-latency ETL, evaluate **EMR Serverless** or **Lambda** (for lightweight transformations).
- **JDBC connections to VPC-hosted databases require a Glue Connection**: RDS, Redshift, or self-managed databases inside a VPC are not directly reachable by Glue workers. A **Glue Connection** (specifying VPC, subnet, and security group) must be configured, and the target security group must allow inbound traffic from the Glue ENI on the database port.
- **Data Catalog is region-scoped**: Tables registered in the Glue Data Catalog in `us-east-1` are not visible to Athena or EMR in `eu-west-1`. For multi-region architectures, either replicate Catalog metadata or use a shared Glue Data Catalog via cross-account resource policies.
- **Python Shell jobs are not Spark**: Python Shell jobs run on a single node with no distributed compute. They are appropriate for metadata operations, API calls, or small file manipulation — not for processing GBs of data. Mistakenly using Python Shell for large datasets produces silent slowness and timeouts.
- **Sensitive data handling**: Glue jobs run with the permissions of the attached IAM role. If the job processes PII, ensure the role follows least-privilege, enable Glue job encryption (at-rest via KMS, in-transit via TLS), and consider using **Glue Sensitive Data Detection** to identify and mask PII fields automatically.

---

## Critical Synthesis Note

**Architectural insight — The Glue Data Catalog as the gravitational center of the AWS analytical ecosystem:**

Most practitioners treat the Glue Data Catalog as a side effect of running Crawlers — a metadata store that exists to make Athena queries work. The deeper architectural reality is that the Data Catalog is the **single, shared schema contract** across the entire AWS analytical stack. Athena, EMR, Redshift Spectrum, and Lake Formation all read from the same Catalog. This means a schema change registered in the Catalog propagates instantly to every consumer — without any per-service reconfiguration.

This positions the Data Catalog not as a Glue-specific component but as the **schema governance layer of the AWS Lakehouse**. Teams that treat it as such — enforcing schema versioning, tagging tables with data classification labels, and governing access via Lake Formation policies at the Catalog level — achieve a degree of data governance that would otherwise require a dedicated third-party catalog tool (Apache Atlas, Collibra, Alation).

The implication for architecture: **invest in Data Catalog hygiene early**. Naming conventions, partition strategies, and data classification tags applied at Catalog creation time pay compounding dividends as the number of consumers grows. Retrofitting governance onto a sprawling, poorly-named Catalog is one of the most expensive and disruptive data engineering tasks in a mature AWS environment.

**Knowledge gap requiring further research**: The behavior of Glue Job Bookmarks under **concurrent job executions** targeting overlapping S3 prefixes is not clearly documented. If two Glue jobs (e.g., a backfill job and the nightly incremental job) run simultaneously against the same source, bookmark state contention can produce duplicate records or missed data. The safe concurrency model for Glue Bookmarks — and whether bookmark state is lock-protected — warrants dedicated investigation before deploying parallel Glue job patterns in production.
