# AWS Security Hub

---

## Core Objective

**Why does this exist?** AWS accounts running at any meaningful scale generate security findings from multiple independent services simultaneously: GuardDuty detects threat activity, Inspector reports software vulnerabilities, Macie flags sensitive data exposure, IAM Access Analyzer surfaces unintended resource access, and AWS Config flags compliance drift. Without a centralized layer, a security team must log into each service console separately, correlate findings manually across accounts and regions, and maintain custom tooling to track remediation status. At multi-account scale, this becomes operationally impossible.

AWS Security Hub solves this by acting as the **regional aggregation and normalization layer for all AWS security findings**. It ingests findings from native AWS security services and third-party products, normalizes them into a single schema (the Amazon Security Finding Format — ASFF), evaluates your environment against automated security standards (CIS, NIST, PCI-DSS, AWS FSBP), and provides a unified compliance posture score. It does not generate findings itself for the most part — it consolidates what other services detect.

Security Hub is also the **routing layer for automated response**: findings flow into Amazon EventBridge, which triggers Lambda functions, SSM Automation documents, or Step Functions workflows for automated remediation — creating a closed-loop security operations pipeline without requiring a SIEM.

> **Exam lens:** SAA-C03 tests Security Hub in two contexts. First, as the **aggregation answer** — when a scenario mentions "centralized view of security findings across multiple accounts and regions," Security Hub is the answer, not GuardDuty or Inspector alone. Second, as the **compliance posture answer** — when a scenario involves automated evaluation against CIS benchmarks or AWS security best practices, Security Hub's security standards feature is what's being tested. Understand that Security Hub *consumes* findings; GuardDuty, Inspector, and Macie *produce* them.

**AWS Reference:** https://docs.aws.amazon.com/securityhub/latest/userguide/what-is-securityhub.html

---

## Fundamental Concepts

### Key Terminology

| Term | Definition |
|---|---|
| **Finding** | A single security observation — a potential threat, vulnerability, or compliance violation. Findings are created by integrated services and normalized into ASFF before ingestion into Security Hub. |
| **ASFF (Amazon Security Finding Format)** | The standardized JSON schema all Security Hub findings conform to. Every finding has a common set of fields: `SchemaVersion`, `ProductArn`, `GeneratorId`, `Types`, `Severity`, `Resources`, `Remediation`. Normalization enables cross-source querying and automated response without per-source parsing logic. |
| **Security Standard** | A curated collection of automated controls evaluated against your AWS environment. Each control maps to one or more AWS Config rules. Security Hub scores your compliance as a percentage of passed controls. |
| **Control** | An individual check within a security standard (e.g., "S3 buckets should have server-side encryption enabled"). A control is either **Passed**, **Failed**, **Not Available**, or **Disabled**. |
| **Security Score** | A percentage (0–100%) representing the fraction of enabled controls that are currently passing. Calculated per standard and rolled up to an overall account score. |
| **Insight** | A saved filter + grouping query over findings. Used to answer recurring operational questions (e.g., "Show me all critical findings grouped by affected resource type"). AWS provides managed insights; you can create custom ones. |
| **Administrator Account** | The central account that aggregates findings from all member accounts in an AWS Organization or manually invited accounts. Has read access to all member findings. |
| **Member Account** | An account whose findings are shared with the Administrator account. Members can view only their own findings. |
| **Aggregation Region** | A designated region that collects findings from all linked regions within one or more accounts. Enables single-pane-of-glass across multi-region environments. |
| **Custom Action** | A user-defined action attached to a finding in the Security Hub console that publishes a specific EventBridge event, triggering an automated or manual response workflow. |
| **Finding Provider** | Any integrated service or product that sends findings to Security Hub in ASFF format. AWS-native providers: GuardDuty, Inspector v2, Macie, IAM Access Analyzer, Firewall Manager, Config, Detective. Third-party: CrowdStrike, Palo Alto, Splunk, and 60+ others via the AWS Partner Network. |

---

### Native AWS Finding Providers

| Service | What It Detects | Finding Category |
|---|---|---|
| **Amazon GuardDuty** | Threat activity: reconnaissance, credential compromise, C2 communication, crypto-mining, lateral movement | Threat Detection |
| **Amazon Inspector v2** | Software vulnerabilities (CVEs) in EC2 AMIs, ECR container images, and Lambda functions | Vulnerability Management |
| **Amazon Macie** | Sensitive data (PII, credentials, financial data) in S3 buckets; bucket access configuration risks | Data Security |
| **IAM Access Analyzer** | S3 buckets, IAM roles, KMS keys, Lambda functions, SQS queues accessible from outside the account or organization | Identity & Access |
| **AWS Firewall Manager** | Policy violations: Security Group compliance, WAF rules, Shield Advanced coverage gaps | Infrastructure Security |
| **AWS Config** | Resource configuration compliance against Config Rules; Security Hub maps Config findings to standard controls | Compliance |
| **Amazon Detective** | Not a finding provider — receives findings *from* Security Hub for deeper investigation | Investigation |

---

### Security Standards Available in Security Hub

| Standard | Governing Body | Controls Count (approx.) | Primary Use Case |
|---|---|---|---|
| **AWS Foundational Security Best Practices (FSBP)** | AWS | ~300 | Baseline AWS security posture for all accounts |
| **CIS AWS Foundations Benchmark v1.4 / v3.0** | Center for Internet Security | ~50–60 | Industry-standard AWS hardening baseline; commonly required by enterprise security teams |
| **PCI DSS v3.2.1 / v4.0** | PCI Security Standards Council | ~100+ | Cardholder data environment compliance |
| **NIST SP 800-53 Rev. 5** | NIST | ~200+ | US federal/DoD compliance requirements |
| **AWS Resource Tagging Standard** | AWS | Varies | Enforce tag policies across resource types |

Each control in a standard maps to one or more AWS Config rules. **Security Hub does not evaluate standards independently** — Config must be enabled in the account for standard controls to produce results.

**AWS Reference (security standards):** https://docs.aws.amazon.com/securityhub/latest/userguide/standards-available.html

---

### Multi-Account and Multi-Region Architecture

```
AWS Organization
├── Management Account
│   └── Security Hub Administrator Account (delegated)
│       ├── Aggregation Region: us-east-1
│       │   ├── Findings from: us-east-1, us-west-2, eu-west-1 (linked regions)
│       │   └── Findings from: all member accounts
│       └── EventBridge → Lambda / SSM Automation (automated response)
│
├── Member Account A  →  findings pushed to Administrator
├── Member Account B  →  findings pushed to Administrator
└── Member Account C  →  findings pushed to Administrator
```

- **Delegated Administrator:** Within an AWS Organization, a designated member account (not the management account) is set as the Security Hub administrator. This is the AWS-recommended pattern — the management account should not be used for operational workloads.
- **Cross-region aggregation:** One region is designated the **Aggregation Region**. Linked regions forward their findings there. You query a single endpoint for your full multi-region picture.
- **Auto-enable:** When an Organization member account is created or joins, Security Hub and its configured standards can be auto-enabled across all new accounts.

**AWS Reference (multi-account):** https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-accounts.html

---

## Procedure or Logic

### Enabling Security Hub and a Security Standard

- **Step 1 — Enable Security Hub in the account/region:** Security Hub is regional and must be enabled per region. GuardDuty and Config must be active in the region for full functionality.

```bash
# Enable Security Hub in us-east-1
aws securityhub enable-security-hub \
  --enable-default-standards \
  --region us-east-1

# Verify enabled standards
aws securityhub get-enabled-standards \
  --region us-east-1
```

- **Step 2 — Enable a specific security standard (if not using defaults):**

```bash
# Enable CIS AWS Foundations Benchmark v1.4.0
aws securityhub batch-enable-standards \
  --standards-subscription-requests \
    '[{"StandardsArn":"arn:aws:securityhub:us-east-1::standards/cis-aws-foundations-benchmark/v/1.4.0"}]' \
  --region us-east-1
```

- **Step 3 — Configure AWS Organizations integration (delegated admin):**

```bash
# From the management account — designate a security account as admin
aws securityhub enable-organization-admin-account \
  --admin-account-id "123456789012" \
  --region us-east-1

# From the delegated admin account — auto-enable for all member accounts
aws securityhub update-organization-configuration \
  --auto-enable \
  --region us-east-1
```

- **Step 4 — Configure cross-region aggregation:**

```bash
# Designate us-east-1 as the aggregation region and link us-west-2, eu-west-1
aws securityhub create-finding-aggregator \
  --region-linking-mode SPECIFIED_REGIONS \
  --regions us-west-2 eu-west-1 \
  --region us-east-1
```

- **Step 5 — Query findings:**

```bash
# Get all CRITICAL findings from the last 24 hours
aws securityhub get-findings \
  --filters '{
    "SeverityLabel": [{"Value": "CRITICAL", "Comparison": "EQUALS"}],
    "RecordState":   [{"Value": "ACTIVE",   "Comparison": "EQUALS"}],
    "WorkflowStatus":[{"Value": "NEW",      "Comparison": "EQUALS"}]
  }' \
  --sort-criteria '[{"Field":"UpdatedAt","SortOrder":"desc"}]' \
  --max-results 50 \
  --region us-east-1
```

---

### Automated Remediation Pipeline via EventBridge

Security Hub findings flow into EventBridge as events. Custom Actions allow you to trigger targeted responses; default event rules can trigger automatic responses for all findings matching a filter.

```json
// EventBridge rule pattern: trigger on any CRITICAL Security Hub finding
{
  "source": ["aws.securityhub"],
  "detail-type": ["Security Hub Findings - Imported"],
  "detail": {
    "findings": {
      "Severity": { "Label": ["CRITICAL"] },
      "RecordState": ["ACTIVE"],
      "WorkflowState": ["NEW"]
    }
  }
}
```

```bash
# Create the EventBridge rule targeting a Lambda remediation function
aws events put-rule \
  --name "SecurityHub-Critical-AutoRemediate" \
  --event-pattern file://critical-finding-pattern.json \
  --state ENABLED \
  --region us-east-1

aws events put-targets \
  --rule "SecurityHub-Critical-AutoRemediate" \
  --targets '[{"Id":"RemediationLambda","Arn":"arn:aws:lambda:us-east-1:123456789012:function:SecurityHubRemediation"}]' \
  --region us-east-1
```

```python
# Lambda handler: parse ASFF finding and route to remediation
import boto3
import json

def lambda_handler(event, context):
    findings = event['detail']['findings']
    for finding in findings:
        resource_type = finding['Resources'][0]['Type']
        generator_id  = finding['GeneratorId']
        resource_arn   = finding['Resources'][0]['Id']

        if resource_type == 'AwsS3Bucket' and 'S3.2' in generator_id:
            # Control S3.2: S3 buckets should prohibit public read access
            s3 = boto3.client('s3')
            bucket_name = resource_arn.split(':::')[1]
            s3.put_public_access_block(
                Bucket=bucket_name,
                PublicAccessBlockConfiguration={
                    'BlockPublicAcls': True,
                    'IgnorePublicAcls': True,
                    'BlockPublicPolicy': True,
                    'RestrictPublicBuckets': True
                }
            )
            print(f"Remediated public S3 bucket: {bucket_name}")
```

**AWS Reference (automated response):** https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-cloudwatch-events.html

---

### Suppressing and Updating Finding Workflow Status

Not every finding requires action — some are accepted risks or false positives. Managing workflow status prevents alert fatigue.

```bash
# Suppress a finding (mark as SUPPRESSED — removes from active dashboard)
aws securityhub batch-update-findings \
  --finding-identifiers '[{"Id":"arn:aws:securityhub:...","ProductArn":"arn:aws:securityhub:..."}]' \
  --workflow '{"Status":"SUPPRESSED"}' \
  --note '{"Text":"Accepted risk: dev account, no production data","UpdatedBy":"security-team"}' \
  --region us-east-1

# Disable a specific control for a standard (e.g., disable a control that does not apply)
aws securityhub update-standards-control \
  --standards-control-arn "arn:aws:securityhub:us-east-1:123456789012:control/cis-aws-foundations-benchmark/v/1.4.0/1.14" \
  --control-status "DISABLED" \
  --disabled-reason "Root account MFA enforced by SCPs, not by IAM policy in this account" \
  --region us-east-1
```

---

## Practical Application / Examples

### Scenario 1 — Centralized Security Posture Across a 50-Account Organization

A company runs 50 AWS accounts (dev, staging, production, sandbox) across three regions (us-east-1, eu-west-1, ap-southeast-1). The CISO requires a single compliance dashboard, automated remediation for critical findings, and a 30-day finding retention audit trail.

**Architecture:**

```
AWS Organizations
├── Security Tooling Account (delegated Security Hub admin)
│   ├── Security Hub — Aggregation Region: us-east-1
│   │   ├── Linked: eu-west-1, ap-southeast-1
│   │   └── Member findings from all 49 accounts (auto-enabled via Org config)
│   ├── EventBridge Rule → Lambda (auto-remediate CRITICAL)
│   ├── EventBridge Rule → SNS → PagerDuty (page on-call for HIGH+)
│   └── S3 bucket (findings export via Firehose for 30-day retention)
└── 49 Member Accounts — findings auto-forwarded, standards auto-enabled
```

**Standards enabled org-wide:**
- AWS Foundational Security Best Practices (FSBP) — baseline posture
- CIS AWS Foundations Benchmark v1.4 — enterprise security requirement

**Outcome:** Single dashboard with aggregate security score across all 50 accounts. Critical findings trigger Lambda within seconds. Monthly compliance reports exported from S3 to the audit team. Zero manual console access to individual accounts for finding review.

---

### Scenario 2 — PCI-DSS Cardholder Data Environment Compliance

A payments company must demonstrate PCI-DSS compliance for its cardholder data environment (CDE), which runs in a dedicated AWS account.

**Implementation:**
```bash
# Enable PCI DSS v3.2.1 standard in the CDE account
aws securityhub batch-enable-standards \
  --standards-subscription-requests \
    '[{"StandardsArn":"arn:aws:securityhub:us-east-1::standards/pci-dss/v/3.2.1"}]' \
  --region us-east-1

# Query only failed PCI controls
aws securityhub get-findings \
  --filters '{
    "ComplianceStatus":    [{"Value":"FAILED","Comparison":"EQUALS"}],
    "ProductName":         [{"Value":"Security Hub","Comparison":"EQUALS"}],
    "GeneratorId":         [{"Value":"pci-dss","Comparison":"PREFIX"}]
  }' \
  --region us-east-1
```

**Outcome:** PCI compliance score is continuously measured. Failed controls generate findings that feed the remediation pipeline. QSA (Qualified Security Assessor) audits reference the Security Hub compliance report as evidence — replacing a manual point-in-time audit checklist with continuous, automated attestation.

---

### Scenario 3 — Third-Party SIEM Integration

A company's SOC team uses Splunk as their primary SIEM. They need all Security Hub findings — including those from GuardDuty, Inspector, and Macie — forwarded to Splunk in real time.

**Architecture:**
```
Security Hub Findings
        │
        ▼
EventBridge (default bus)
        │
        ▼
Kinesis Data Firehose
        │  (ASFF JSON records)
        ▼
Splunk HTTP Event Collector (HEC)
```

```bash
# Create Firehose delivery stream to Splunk HEC
aws firehose create-delivery-stream \
  --delivery-stream-name "SecurityHub-to-Splunk" \
  --delivery-stream-type DirectPut \
  --splunk-destination-configuration '{
    "HECEndpoint": "https://splunk.corp.example.com:8088",
    "HECEndpointType": "Event",
    "HECToken": "splunk-hec-token-here",
    "S3BackupMode": "FailedEventsOnly",
    "S3Configuration": {"RoleARN":"arn:aws:iam::123456789012:role/FirehoseRole","BucketARN":"arn:aws:s3:::securityhub-backup"}
  }' \
  --region us-east-1

# EventBridge rule: all Security Hub findings → Firehose
aws events put-rule \
  --name "SecurityHub-All-to-Firehose" \
  --event-pattern '{"source":["aws.securityhub"],"detail-type":["Security Hub Findings - Imported"]}' \
  --state ENABLED \
  --region us-east-1
```

**Outcome:** Every Security Hub finding (normalized to ASFF) appears in Splunk within seconds. The SOC team queries a single Splunk index for all AWS security events, correlates with on-premises events, and runs SIEM-based detection rules across the unified dataset.

---

## Critical Considerations

- **Security Hub is regional — it must be enabled per region:** Findings generated in `eu-west-1` are not visible in `us-east-1` unless cross-region aggregation is explicitly configured. A common gap: organizations enable Security Hub only in their primary region, leaving other regions blind. Use the **Aggregation Region** feature and enable Security Hub in every active region.

- **AWS Config must be enabled for security standards to function:** Security Hub standards controls are evaluated by AWS Config rules under the hood. If Config is not enabled in a region or account, standard controls will show as `NOT_AVAILABLE` rather than `FAILED` — giving a falsely clean compliance score. Always verify Config is active before interpreting security scores.

- **Security Hub does not remediate findings — it only aggregates and routes:** A common misconception is that enabling Security Hub fixes compliance issues. It does not. Security Hub identifies and centralizes findings; remediation requires separate automation (Lambda, SSM Automation, AWS Systems Manager) triggered via EventBridge. A high security score means findings are passing; it does not mean the environment is secure if findings are being suppressed incorrectly.

- **Finding deduplication is not guaranteed across providers:** The same underlying issue (e.g., an S3 bucket with public access) may generate findings from both Macie and Security Hub's FSBP standard simultaneously. Workflow processes must account for duplicate findings representing the same resource defect. Use finding `GeneratorId` and `Resources[].Id` to correlate and deduplicate before triggering automated remediation.

- **The security score is sensitive to disabled controls:** Disabling a control (even legitimately) improves the score. Organizations sometimes disable noisy or irrelevant controls to inflate their score rather than addressing root causes. The score is a lagging operational indicator, not an audit artifact — treat it with appropriate skepticism and always inspect disabled controls.

- **Cross-region aggregation only works within a single partition:** You cannot aggregate findings from `us-east-1` (aws commercial) and `us-gov-east-1` (aws-us-gov) into a single aggregation region. GovCloud and China regions are separate partitions with isolated Security Hub instances.

- **30-day finding retention is the default — plan for long-term storage:** Security Hub retains findings for **90 days** (not 30) by default, after which they are automatically deleted. For compliance regimes requiring 1–7 year retention, export findings to S3 via EventBridge + Kinesis Data Firehose at ingestion time. Do not rely on Security Hub as a finding archive.

- **Custom insights have a limit of 100 per account:** AWS-managed insights do not count against this quota, but custom insights do. Design insights to be broadly reusable rather than highly specific to avoid hitting the limit in mature security programs with many custom queries.

- **Third-party integrations must explicitly send findings in ASFF format:** Enabling a third-party integration in the Security Hub console does not automatically start receiving findings. The third-party product must be separately configured to send findings to the Security Hub API (`BatchImportFindings`). Verify integration health in Security Hub → Integrations → Check sending status.

- **Pricing scales with finding volume, not account count:** Security Hub is priced per finding ingested per month (first 10,000 findings free, then tiered pricing) and per compliance check per month. In accounts with high GuardDuty event rates, Security Hub finding ingestion costs can become significant — monitor with Cost Explorer and consider filtering low-severity findings at the EventBridge rule level before they reach Security Hub.

**AWS Reference (pricing):** https://aws.amazon.com/security-hub/pricing/

---

## Critical Synthesis Note

> **High-level architectural insight — Security Hub as the security data bus, not a security tool:** The most useful mental model for Security Hub is not "security dashboard" but **security event bus with a normalization layer**. The ASFF schema is the key architectural artifact: by forcing every finding provider — GuardDuty, Inspector, Macie, third-party EDR, CSPM tools — to emit findings in the same schema, Security Hub makes it possible to write a single Lambda remediation function, a single Splunk query, or a single EventBridge rule that works regardless of which service generated the finding. This is the same architectural pattern as Apache Kafka (normalize all events into a topic schema) or OpenTelemetry (normalize all observability data into a common format). Security Hub is not a SIEM replacement — it is the normalization and routing layer that makes the rest of your security toolchain source-agnostic. Organizations that understand this build radically simpler security automation than those that treat it as yet another console to check.

> **Knowledge gap requiring further research:** The relationship between **Security Hub and AWS Security Lake** deserves deeper investigation. Security Lake (built on Amazon S3 and Apache Iceberg, using the OCSF schema) is positioned as the long-term security data lake for multi-source retention and analytics — it ingests from Security Hub, CloudTrail, Route 53, VPC Flow Logs, and third-party sources. The boundary between what Security Hub handles (operational finding routing and compliance scoring) versus what Security Lake handles (long-term retention, cross-source analytics, ML-based detection) is not clearly delineated in SAA-C03 preparation materials. As Security Lake matures, this division of responsibility will likely appear on the exam. Additionally, investigate **AWS Detective** — it reads from Security Hub but performs graph-based investigation; understanding when to route a finding to Detective versus directly to a response workflow is an open architectural question.

---

**Primary AWS Documentation:** https://docs.aws.amazon.com/securityhub/latest/userguide/what-is-securityhub.html
**Security standards available:** https://docs.aws.amazon.com/securityhub/latest/userguide/standards-available.html
**ASFF (Amazon Security Finding Format):** https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-findings-format.html
**Multi-account management:** https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-accounts.html
**Cross-region aggregation:** https://docs.aws.amazon.com/securityhub/latest/userguide/finding-aggregation.html
**Automated response with EventBridge:** https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-cloudwatch-events.html
**Integrations (native + third-party):** https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-integrations.html
**Enabling Security Hub via Organizations:** https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-orgs-delegated-admin.html
**Security Hub pricing:** https://aws.amazon.com/security-hub/pricing/
