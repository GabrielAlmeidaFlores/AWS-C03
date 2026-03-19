# AWS Athena

---

## Core Objective

AWS Athena addresses the operational and financial cost of running ad-hoc analytics against large datasets stored in S3. The fundamental problem it solves is **eliminating the need to provision, manage, or scale dedicated query infrastructure** (e.g., Redshift clusters, EMR jobs) for intermittent analytical workloads.

The ultimate goal: enable teams to derive insights from raw S3 data using standard SQL — paying only per query, with zero infrastructure management.

---

## Fundamental Concepts

### Key Terminology

| Term | Definition |
|---|---|
| **Data Catalog (AWS Glue)** | Metadata repository storing table schemas, column definitions, and data formats that Athena uses to interpret S3 objects |
| **SerDe** | Serializer/Deserializer — a library that tells Athena how to read and write a specific data format (e.g., JSON, CSV, ORC) |
| **Partitioning** | Logical division of data in S3 using path prefixes (e.g., `s3://bucket/year=2024/month=01/`) to restrict query scan scope |
| **Columnar Format** | Storage formats (Parquet, ORC) that organize data by column instead of row, enabling Athena to read only relevant columns and dramatically reduce scan size |
| **Workgroup** | An Athena resource that isolates query execution, enforces cost controls, and manages per-team or per-environment configurations |
| **Query Result Location** | An S3 path where Athena writes `.csv` query output and metadata files after execution |
| **DML / DDL** | Data Manipulation Language (SELECT, INSERT INTO) and Data Definition Language (CREATE TABLE, ALTER TABLE) — both supported in Athena |

### Core Architecture Pillars

- **Compute Layer**: Presto-based distributed query engine (managed by AWS); no servers to configure
- **Storage Layer**: Data lives entirely in S3; Athena does not own or move the data
- **Metadata Layer**: AWS Glue Data Catalog (or a Hive-compatible metastore) defines the logical schema over raw S3 files
- **Pricing Model**: `$5.00 per TB` of data scanned per query; no charge for DDL statements, failed queries, or cancelled queries

---

## Procedure or Logic

### End-to-End Query Workflow

- **Step 1 — Store raw data in S3**: Organize data in a structured prefix hierarchy. Use partitioned paths when data will be filtered by dimension (e.g., date, region).
- **Step 2 — Define a schema in Glue Data Catalog**: Create a database and table using Athena's DDL editor or the Glue console. Map columns to the data's structure and specify the correct SerDe.
- **Step 3 — Configure query result location**: Set an S3 bucket/prefix for query output in the Athena workgroup settings or at query time.
- **Step 4 — Run SQL queries**: Use the Athena Query Editor, AWS SDK, CLI, or JDBC/ODBC driver to submit queries. Athena executes them against the S3 data using the Glue schema.
- **Step 5 — Consume results**: Read the output `.csv` from S3, or integrate directly with QuickSight, SageMaker, Pandas (via `awswrangler`), or any JDBC-compatible BI tool.

### Creating an External Table (DDL Reference)

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS my_database.cloudfront_logs (
  `date`          DATE,
  time            STRING,
  location        STRING,
  bytes           BIGINT,
  request_ip      STRING,
  method          STRING,
  host            STRING,
  uri             STRING,
  status          INT,
  referrer        STRING,
  user_agent      STRING
)
PARTITIONED BY (year STRING, month STRING)
ROW FORMAT DELIMITED
  FIELDS TERMINATED BY '\t'
STORED AS TEXTFILE
LOCATION 's3://my-bucket/cloudfront-logs/'
TBLPROPERTIES ('skip.header.line.count' = '2');
```

### Loading Partitions After Data Arrives

```sql
-- Option A: Auto-discover all partitions (expensive on large datasets)
MSCK REPAIR TABLE my_database.cloudfront_logs;

-- Option B: Add a specific partition manually (preferred for production)
ALTER TABLE my_database.cloudfront_logs
ADD PARTITION (year='2024', month='03')
LOCATION 's3://my-bucket/cloudfront-logs/year=2024/month=03/';
```

---

## Practical Application / Examples

### Scenario: Analyzing VPC Flow Logs for Security Auditing

An AWS security team needs to query rejected traffic across all VPCs over the last 30 days. Raw VPC Flow Logs are delivered to S3, partitioned by account and day.

**1. Create the Glue table** pointing to the Flow Logs prefix, defining columns per the [AWS Flow Log record format](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html).

**2. Query rejected traffic:**

```sql
SELECT
  srcaddr,
  dstaddr,
  dstport,
  protocol,
  COUNT(*) AS rejected_attempts
FROM my_database.vpc_flow_logs
WHERE action = 'REJECT'
  AND partition_date BETWEEN '2024-01-01' AND '2024-01-31'
GROUP BY srcaddr, dstaddr, dstport, protocol
ORDER BY rejected_attempts DESC
LIMIT 50;
```

**3. Cost estimate**: If the filtered partition contains 50 GB of Parquet data, the scan cost is `50 GB × ($5 / 1000 GB) = $0.25` per query execution.

**4. Outcome**: Security team identifies top offending source IPs without standing up any infrastructure, in minutes.

---

## Critical Considerations

- **Columnar formats are non-negotiable for cost efficiency**: Running queries on raw CSV or JSON files at scale results in full-dataset scans. Convert to **Parquet or ORC** and compress with **Snappy or GZIP** to reduce scan size by 60–90%.
- **Partitioning is the primary cost-control lever**: Without partitions, every query scans the entire table. Design partition keys around your most frequent `WHERE` clause filters (e.g., `date`, `region`, `account_id`).
- **Small files degrade performance**: Thousands of small S3 objects (< 128 MB) increase overhead. Use S3 lifecycle policies, Glue ETL, or EMR to compact files before querying.
- **Athena is not ACID-compliant**: It does not support row-level `UPDATE` or `DELETE` natively. Use **Iceberg table format** (supported in Athena v3) if you require ACID transactions, time travel, or upserts.
- **Concurrency limits**: Default limit is 5 concurrent DML queries per account per region. Request a limit increase via AWS Support for high-throughput workloads.
- **Result reuse is available but short-lived**: Athena can reuse query results (up to 7 days) for identical queries. This is useful for dashboards but must be explicitly enabled per workgroup.
- **Schema evolution pitfalls**: Adding columns to existing Parquet files requires schema merging (`'parquet.mr.enable.summary.metadata' = false` and Glue schema updates). Mismatches cause silent nulls or query failures.
- **IAM scope**: Athena requires permissions on S3 (source data + result bucket), Glue (`glue:GetTable`, `glue:GetDatabase`), and Athena itself. Over-permissioned roles are a common security misconfiguration.

---

## Critical Synthesis Note

**Cross-disciplinary insight — Athena as a Lakehouse gateway, not just a query tool:**

Most practitioners treat Athena as a simple ad-hoc SQL interface to S3. The architectural implication frequently missed is that Athena v3's native support for **Apache Iceberg** tables transforms it into a full **Lakehouse** query engine — capable of ACID transactions, time-travel queries, and schema evolution directly over S3 without a dedicated transactional database.

This positions Athena not merely as an analytics convenience but as a credible replacement for operational reporting workloads previously requiring Redshift or Aurora — at a fraction of the cost, with no cluster management.

**Knowledge gap requiring further research**: The behavior of Athena under **high-concurrency, sub-second latency requirements** (< 500ms p99) is not well-documented for production SLAs. Athena is optimized for throughput over latency. If your workload requires low-latency interactive queries (e.g., user-facing dashboards), evaluate **Athena SPICE caching via QuickSight** or consider **Redshift Serverless** as an alternative. The trade-off matrix between these services for mixed workloads (ad-hoc + operational) warrants dedicated benchmarking.
