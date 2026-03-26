# EC2 Image Builder

---

## Core Objective

**Why does this exist?** Building AMIs manually — launching an instance, configuring it, creating the image, copying to regions — is operationally fragile at scale. There is no versioning, no automated testing, no compliance validation, and no audit trail. A single human error produces a bad AMI that propagates to every instance launched from it.

EC2 Image Builder solves this by providing a **managed pipeline** for building, testing, and distributing AMIs automatically. It treats the AMI as a software artifact: versioned, tested before use, and distributed to target accounts and regions through a controlled process.

> **Exam lens:** The SAA-C03 exam positions Image Builder as the correct answer when a question involves **automated, repeatable AMI creation**, **compliance hardening at scale**, or **distributing AMIs to multiple accounts/regions**. It is the production-grade alternative to ad-hoc `create-image` calls.

**AWS Reference:** https://docs.aws.amazon.com/imagebuilder/latest/userguide/what-is-image-builder.html

---

## Fundamental Concepts

### Key Terminology

| Term | Definition |
|---|---|
| **Image Pipeline** | The top-level resource that orchestrates the full build → test → distribute lifecycle on a schedule or on-demand |
| **Image Recipe** | Defines the base image (source AMI) and the ordered list of components to apply; versioned and immutable once published |
| **Component** | A reusable unit of automation (install, configure, validate, clean up) written in YAML; applied in sequence during the build phase |
| **Infrastructure Configuration** | The EC2 instance type, VPC, subnet, IAM role, and SNS topic used to run the build and test instances |
| **Distribution Configuration** | Specifies which regions and AWS accounts receive the finished AMI, and whether it should be encrypted or shared |
| **Image** | The output artifact — a versioned AMI (or container image) produced by a single pipeline execution |
| **Test Component** | A component that runs validation logic (e.g., security scans, service health checks) on the built image before distribution |
| **Build Phase** | The stage where components are applied to the base instance to produce the image |
| **Test Phase** | The stage where the built image is launched into a fresh instance and test components are run; a failed test blocks distribution |

---

### Core Architecture Pillars

- **Image Recipe**: The immutable definition of what goes into the image — source AMI + ordered components. Versioned so you can track exactly what changed between image builds.
- **Components**: Reusable automation blocks in YAML that perform install, configure, validate, and cleanup steps. AWS provides managed components (e.g., CIS hardening, CloudWatch Agent install); you write custom ones for app-specific logic.
- **Infrastructure Configuration**: The temporary EC2 environment used to run the build — instance type, network, IAM role with permissions to write logs and interact with other AWS services.
- **Distribution Configuration**: Where the finished AMI goes — target regions, target accounts, encryption settings, and AMI sharing permissions. This is what makes Image Builder multi-region and multi-account by default.
- **Image Pipeline**: The orchestrator that ties all the above together and runs on a schedule (cron) or on-demand trigger.

---

### Build Lifecycle

| Phase | What Happens |
|---|---|
| **Launch** | Image Builder launches a temporary EC2 instance from the base AMI using the Infrastructure Configuration |
| **Build** | Components are applied in order: download, install, configure, validate, clean up |
| **Stop & Snapshot** | The instance is stopped and an AMI snapshot is taken |
| **Test** | A new temporary instance is launched from the just-built AMI; test components run validation |
| **Distribute** | If tests pass, the AMI is distributed to all target regions and accounts per Distribution Configuration |
| **Terminate** | All temporary instances are terminated automatically |

---

## Procedure or Logic

- **Step 1 — Create an Image Recipe**: Select the base AMI (e.g., latest Amazon Linux 2023 from AWS) and attach components in order — hardening first, then agents, then the application. Pin the recipe to a version.

- **Step 2 — Create an Infrastructure Configuration**: Choose the instance type (e.g., `t3.medium`), VPC/subnet, and an IAM instance profile with `EC2InstanceProfileForImageBuilder` permissions. Optionally configure an S3 bucket and SNS topic for logs and notifications.

- **Step 3 — Create a Distribution Configuration**: Specify target regions, target account IDs, and whether the output AMI should be encrypted (and with which KMS key).

- **Step 4 — Create an Image Pipeline**: Wire the recipe, infrastructure config, and distribution config together. Set a schedule (e.g., weekly) or leave it as manual trigger.

- **Step 5 — Run and monitor**: Trigger the pipeline manually or let the schedule fire. Monitor progress in the Image Builder console or via EventBridge events. On success, the finished AMI is available in all target regions.

```yaml
# Example: Custom component YAML — install and start a service
name: InstallMyApp
description: Installs MyApp v2.3 and enables the service
schemaVersion: 1.0
phases:
  - name: build
    steps:
      - name: InstallPackage
        action: ExecuteBash
        inputs:
          commands:
            - yum install -y myapp-2.3
            - systemctl enable myapp
  - name: validate
    steps:
      - name: CheckService
        action: ExecuteBash
        inputs:
          commands:
            - systemctl is-active myapp || exit 1
```

---

## Practical Application / Examples

### Scenario: Automated Weekly Hardened AMI Across Three Regions

A financial services team must ensure all EC2 instances use a CIS Level 1 hardened image with the latest OS patches, updated weekly, available in `us-east-1`, `eu-west-1`, and `ap-southeast-1`. The process must be fully automated and auditable.

```
Image Pipeline (weekly schedule)
  ├── Image Recipe v1.3
  │     ├── Base: latest aws/amazon-linux-2023/x86_64 (dynamic — resolves at build time)
  │     ├── Component: AWS-managed CIS Level 1 hardening
  │     ├── Component: CloudWatch Agent install + config
  │     └── Component: SSM Agent (latest)
  │
  ├── Infrastructure Config
  │     ├── Instance type: t3.medium
  │     ├── IAM Role: EC2InstanceProfileForImageBuilder
  │     └── SNS: build-notifications@company.com
  │
  └── Distribution Config
        ├── us-east-1  (primary)
        ├── eu-west-1  (copy + encrypt with regional CMK)
        └── ap-southeast-1  (copy + encrypt with regional CMK)
              └── SSM Parameter Store updated: /golden-ami/latest = ami-new
```

**Outcome**: Every Monday at 02:00 UTC, a new hardened AMI is built, tested, and distributed to all three regions. Launch Templates resolve `/golden-ami/latest` at instance launch — no infrastructure code changes required for the weekly update.

---

## Critical Considerations

- **The base AMI in a recipe can be dynamic or static**: Referencing the latest AWS-published AMI (e.g., latest Amazon Linux 2023) means each weekly build picks up OS patches automatically. Pinning a specific AMI ID means you control exactly what base is used — useful for compliance audits but requires manual updates.

- **Test phase failures block distribution entirely**: If a test component returns a non-zero exit code, the pipeline stops before the AMI reaches any target region. This is a feature — bad images never reach production — but it means test components must be authoritative and well-written.

- **Components are versioned and immutable**: Once a component version is published, it cannot be modified. Create a new version instead. This ensures recipe versions are reproducible and auditable.

- **Infrastructure Configuration instances are temporary and billed**: Image Builder launches real EC2 instances during build and test. Instance hours are charged normally. For large instance types or long build processes, this cost can be significant — right-size the build instance to the workload.

- **Distribution to other accounts requires RAM (Resource Access Manager) or AMI sharing**: When distributing to a different AWS account, the target account must be explicitly listed in the Distribution Configuration. Image Builder uses `modify-image-attribute` internally — the target account sees it as a shared AMI.

- **Encrypted distribution requires KMS key grants**: If the Distribution Configuration specifies encryption with a CMK, the Image Builder service role must have `kms:GenerateDataKey` and `kms:Decrypt` permissions on that key in the target region.

- **Image Builder does not manage Launch Template updates**: After a new AMI is built, Launch Templates and ASGs still reference the old AMI unless you update them. The pattern is to write the new AMI ID to SSM Parameter Store after distribution, then have Launch Templates reference the SSM parameter.

---

## Critical Synthesis Note

> **High-level insight — Image Builder enforces the separation of build-time and run-time configuration:**
>
> The fundamental architectural principle behind Image Builder is that everything requiring root access, package installation, or service configuration should happen at build time — not at runtime via User Data. User Data scripts are a runtime patch mechanism, not a configuration management tool. An org that relies on User Data for instance configuration has invisible configuration drift and untested startup sequences. Image Builder shifts that complexity left: the image is tested before it touches production, and every instance is born in a known-good state.

> **Knowledge gap requiring further research**: The interaction between Image Builder's distribution mechanism and AWS Organizations Service Control Policies (SCPs) — specifically whether an SCP blocking `ec2:ModifyImageAttribute` in member accounts would interfere with cross-account AMI distribution — warrants verification for multi-account governance architectures.

---

## SAA-C03 Exam Focus

- **Image Builder = automated, repeatable, tested AMI creation** — whenever a question asks about building AMIs at scale, enforcing compliance, or distributing to multiple regions/accounts automatically, Image Builder is the answer.
- **Recipes are versioned and immutable** — changing a component requires a new recipe version; this ensures auditability and reproducibility.
- **Test phase gates distribution** — a failed test blocks AMI distribution entirely; this is the key differentiator from manual `create-image` which has no built-in testing.
- **Image Builder vs. User Data** — Image Builder bakes configuration into the AMI at build time; User Data runs at boot time. Image Builder is correct for consistency and compliance; User Data is only appropriate for instance-specific dynamic configuration.
- **Distribution Configuration handles multi-region and multi-account** — no manual `copy-image` or `modify-image-attribute` calls needed; Image Builder automates both.

---

**Primary AWS Documentation:** https://docs.aws.amazon.com/imagebuilder/latest/userguide/what-is-image-builder.html

**Image Recipes:** https://docs.aws.amazon.com/imagebuilder/latest/userguide/manage-recipes.html

**Components:** https://docs.aws.amazon.com/imagebuilder/latest/userguide/manage-components.html

**Distribution Configuration:** https://docs.aws.amazon.com/imagebuilder/latest/userguide/manage-distribution-settings.html

**Infrastructure Configuration:** https://docs.aws.amazon.com/imagebuilder/latest/userguide/manage-infra-config.html
