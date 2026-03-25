# AGENTS.md — Documentation Guide for AI Agents

---

## What Is This Project?

This is a personal AWS certification study repository, structured as a **living knowledge base** oriented toward the **AWS Solutions Architect Associate (SAA-C03)** exam and real-world cloud architecture practice.

The goal is **not** to accumulate raw notes. Every document is a synthesized, permanently useful reference — written at a senior practitioner level, optimized for long-term recall, and grounded in real-world applicability. Think of it as a technical handbook, not a transcript.

**Succinctness is a first-class requirement.** Documents must cover what a practitioner needs to know for the SAA-C03 exam and real-world use — not every possible feature variant, API parameter, or edge case. Prefer one sharp, well-chosen example over three overlapping ones. Every sentence must earn its place; if removing it loses nothing, remove it.

---

## Repository Structure

```
AWS-C03/
├── AGENTS.md                     ← this file
├── big-data/                     ← AWS managed data services (Athena, EMR, Glue, etc.)
├── ec2/
│   ├── instance-access/          ← access methods (SSM, EC2 Instance Connect, etc.)
│   ├── instance-launch/          ← launch configuration (placement groups, etc.)
│   └── monitoring/               ← observability (CloudWatch, etc.)
├── general-knowledge/            ← domain-agnostic CS concepts (concurrency, CPU, etc.)
└── vpc/
    └── privatelink-and-lattice/  ← VPC connectivity primitives
```

**When adding new documents:**
- Place the file in the most specific existing subfolder that matches the topic.
- If no subfolder fits, create one with a clear, lowercase, hyphenated name that describes the concern area (e.g., `ec2/storage/`, `vpc/routing/`).
- Never place documents in the root folder. The root is reserved for project-level meta-files like this one.
- File names must be lowercase, hyphenated, and descriptive: `placement-groups.md`, `vpc-endpoints.md`, `cloudwatch.md`.

---

## Mandatory Document Schema

Every document **must** follow this exact section order. Do not add, remove, rename, or reorder sections.

```
# <Service or Topic Name>

---

## Core Objective
## Fundamental Concepts
## Procedure or Logic
## Practical Application / Examples
## Critical Considerations
## Critical Synthesis Note
## SAA-C03 Exam Focus
```

### Section Specifications

**`## Core Objective`**
Define the *why*. What problem does this service or concept solve? What is the ultimate goal? Start with the pain point, then the solution. Keep it to 2–4 tight paragraphs. No bullet lists.

**`## Fundamental Concepts`**
Break down the core pillars. Must include:
- A **terminology table** (`| Term | Definition |`) with all domain-specific vocabulary
- **Architecture pillars** as a bullet list (what are the functional layers of this service?)
- At least one **comparison table** if the topic has distinct variants, types, or modes

**`## Procedure or Logic`**
For processes: a sequential, numbered-style bullet walkthrough (use `**Step N —**` format).
For concepts: a structured chain of reasoning.
Must include at least one **code block** — CLI commands, configuration files, or scripts demonstrating real usage.

**`## Practical Application / Examples`**
At minimum one concrete, real-world scenario. Structure it as:
- A named scenario with a problem statement
- An architecture diagram (use ASCII/text if needed)
- Implementation details with code where relevant
- A measurable outcome (cost, time, scale — make it concrete)

**`## Critical Considerations`**
A bullet list of production pitfalls, edge cases, risks, limitations, and common mistakes. Each bullet must be actionable — not just "be careful with X" but *why* X is dangerous and what to do instead. Minimum 6–8 bullets for any reasonably complex topic.

**`## Critical Synthesis Note`**
Mandatory. Must contain two things:
1. A **high-level architectural insight** — something non-obvious that reframes how the service fits into the broader ecosystem. Not a summary of what was already said.
2. A **documented knowledge gap** — something the input or AWS documentation does not clearly answer, stated as a specific open question for further research.

**`## SAA-C03 Exam Focus`**
Mandatory. A compact, scannable bullet list of the specific facts, distinctions, and decision rules this topic is tested on in the SAA-C03 exam. Each bullet must be self-contained and written as a testable statement — not a topic header. Cover:
- The key **"if X → use Y"** decision rules the exam scenario questions rely on
- The most common **wrong-answer traps** (what the exam tries to trick you into choosing and why it's wrong)
- Any **hard limits, numbers, or constraints** the exam expects you to recall
- The single most important **distinction** for this topic (e.g., System vs. Instance check, Basic vs. Detailed Monitoring)

Keep to **5–8 bullets maximum**. This is a rapid-recall cheat sheet, not a document summary. If a point was already obvious from the rest of the document, omit it — include only what a test-taker is likely to forget or confuse under pressure.

---

## Two Document Patterns

The schema above is universal. The *style* varies by folder:

---

### Pattern A — Service Documentation (used in: `big-data/`)

Used for AWS managed services with no explicit exam-tip orientation. Pure technical depth.

**Characteristics:**
- Critical Synthesis Note is written as **regular bold paragraphs** (no blockquotes)
- No `> **Exam lens:**` callouts
- Inline AWS reference links appear **within the relevant section** as `**AWS Reference:** <url>`
- **No** consolidated links section at the bottom
- Tone: engineering-focused, architecture-first

**Example structure (from `big-data/athena.md`):**

```markdown
# AWS Athena

---

## Core Objective

AWS Athena addresses the operational and financial cost of running ad-hoc analytics...

---

## Fundamental Concepts

### Key Terminology

| Term | Definition |
|---|---|
| **Data Catalog (AWS Glue)** | Metadata repository storing table schemas... |

### Core Architecture Pillars

- **Compute Layer**: Presto-based distributed query engine...

---

## Procedure or Logic

### End-to-End Query Workflow

- **Step 1 — Store raw data in S3**: ...
- **Step 2 — Define a schema in Glue Data Catalog**: ...

### Creating an External Table (DDL Reference)

    ```sql
    CREATE EXTERNAL TABLE IF NOT EXISTS ...
    ```

---

## Practical Application / Examples

### Scenario: Analyzing VPC Flow Logs for Security Auditing

...

---

## Critical Considerations

- **Columnar formats are non-negotiable for cost efficiency**: ...

---

## Critical Synthesis Note

**Cross-disciplinary insight — Athena as a Lakehouse gateway, not just a query tool:**

...

**Knowledge gap requiring further research**: ...

---

## SAA-C03 Exam Focus

- ...
- ...
```

---

### Pattern B — EC2 / VPC Documentation (used in: `ec2/`, `vpc/`)

Used for EC2 and networking topics. Includes exam-oriented callouts and a consolidated links section.

**Characteristics:**
- `> **Exam lens:**` blockquote immediately after the Core Objective — highlights what the SAA-C03 exam specifically tests about this topic
- Inline `**AWS Reference:** <url>` links appear within sections at the point of relevance
- Critical Synthesis Note uses `>` **blockquote format**
- A **consolidated documentation links section** at the very bottom of the file listing all referenced URLs
- Tone: exam-aware + production-grounded

**Example structure (from `ec2/instance-launch/placement-groups.md`):**

```markdown
# EC2 Placement Groups

---

## Core Objective

**Why does this exist?** ...

> **Exam lens:** Placement Groups are a *zero-cost* feature...

**AWS Reference:** https://docs.aws.amazon.com/...

---

## Fundamental Concepts

### The Three Placement Group Types

| Feature | **Cluster** | **Spread** | **Partition** |
|---|---|---|---|
| ... |

---

## Critical Considerations

- **`InsufficientCapacityError`:** ...
  - **AWS Reference:** https://docs.aws.amazon.com/...

---

## Critical Synthesis Note

> **High-level insight — ...**:
>
> ...

> **Knowledge gap requiring further research**: ...

---

## SAA-C03 Exam Focus

- ...
- ...

---

**Primary AWS Documentation:** https://docs.aws.amazon.com/...
**Related Reference 1:** https://docs.aws.amazon.com/...
**Related Reference 2:** https://docs.aws.amazon.com/...
```

---

## External Documentation Links — Non-Negotiable Rule

**Every document must include links to the official AWS documentation pages it references.**

This is not optional. Links serve two purposes:
1. Allow the reader to verify and expand on the documented information
2. Ground the document in a specific, auditable source of truth

**Rules for links:**
- Always link to `https://docs.aws.amazon.com/` — never to blog posts, re:Invent videos, or third-party summaries as primary sources
- In **Pattern A** (big-data): embed links inline as `**AWS Reference:** <url>` at the point of relevance within the section
- In **Pattern B** (ec2/vpc): embed inline `**AWS Reference:** <url>` within sections **and** consolidate all links at the bottom of the document
- Link to the **most specific page possible** — not the service root, but the specific feature or concept page
- If a CLI command, config option, or API parameter is documented, link to it

**Canonical link patterns by topic area:**
```
EC2 general:       https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/<topic>.html
EC2 monitoring:    https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/<topic>.html
VPC:               https://docs.aws.amazon.com/vpc/latest/userguide/<topic>.html
Glue:              https://docs.aws.amazon.com/glue/latest/dg/<topic>.html
Athena:            https://docs.aws.amazon.com/athena/latest/ug/<topic>.html
EMR:               https://docs.aws.amazon.com/emr/latest/ManagementGuide/<topic>.html
IAM:               https://docs.aws.amazon.com/IAM/latest/UserGuide/<topic>.html
```

---

## Formatting Rules

| Rule | Requirement |
|---|---|
| **Section separators** | Use `---` between every section |
| **Emphasis** | Use `**bold**` for terms, warnings, and key distinctions |
| **Code** | Use fenced code blocks with language hint (` ```bash `, ` ```sql `, ` ```json `, ` ```python `) |
| **Tables** | Use for all comparisons with 2+ dimensions; always include a header row |
| **Lists** | Use `-` bullets for unordered; use `**Step N —**` prefix for sequential procedures |
| **File names** | Lowercase, hyphenated: `vpc-endpoints.md`, `cloudwatch.md` |
| **Headings** | `#` for title, `##` for schema sections, `###` for subsections — no deeper than `###` |
| **Tone** | Analytical, authoritative, zero fluff. Every sentence must deliver information. |
| **Length** | Prefer concise over exhaustive. A focused 300-word section beats a bloated 800-word one. Include only what changes how a practitioner thinks or acts — cut the rest. |

---

## What NOT to Do

- **Do not summarize or rephrase** source material. Synthesize — find the underlying logic, hierarchy, and relationships.
- **Do not document exhaustively**. The SAA-C03 exam tests concepts, distinctions, and decision logic — not API encyclopedias. Omit feature variants, parameters, and edge cases that don't change how you reason about a service. One precise example beats three overlapping ones.
- **Do not pad sections to meet a minimum**. The "Minimum 6–8 bullets" for Critical Considerations is a floor for complex topics — not a target. Five high-signal bullets are better than eight where two repeat the same point in different words.
- **Do not omit the Critical Synthesis Note**. It is the highest-value section and the hardest to write. It must contain a non-obvious insight and an open knowledge gap.
- **Do not use marketing language**: "powerful," "robust," "seamlessly," "cutting-edge" — eliminate these entirely.
- **Do not skip code blocks**. Every Procedure section must demonstrate real syntax. Pseudocode is not acceptable.
- **Do not create documents in the root folder** except for project-level meta-files.
- **Do not commit or push changes** unless explicitly instructed by the user.
- **Do not create planning or notes files** (e.g., `TODO.md`, `plan.md`) in the repository. Work in memory.
- **Do not explain out-of-scope features**. A CloudWatch doc placed in `ec2/monitoring/` covers CloudWatch as it applies to EC2 — not CloudWatch for RDS, Lambda, or other services.

---

## Scope Discipline

Each document must be **scoped to its folder context**. If a document lives in `ec2/monitoring/`, it explains the topic as it applies to EC2 — not as a general service reference. If a feature of the service is only relevant outside EC2, omit it or note it briefly as out of scope.

Examples of correct scoping:
- `ec2/monitoring/cloudwatch.md` → covers CloudWatch Agent, EC2 metrics, status checks, alarms, Auto Recovery — **not** CloudWatch for RDS or Lambda
- `vpc/privatelink-and-lattice/vpc-endpoints.md` → covers Gateway and Interface endpoints in VPC context — **not** general PrivateLink architecture for SaaS
- `big-data/glue.md` → covers Glue ETL, Crawlers, Data Catalog — **not** Glue DataBrew or Elastic Views unless directly relevant to ETL pipelines

---

## Quick-Start Checklist for New Documents

Before creating a new document, verify:

- [ ] Correct subfolder identified (or new subfolder created with a logical name)
- [ ] File name is lowercase and hyphenated
- [ ] Pattern A or Pattern B selected based on folder context
- [ ] All 6 schema sections present in the correct order
- [ ] Terminology table included in Fundamental Concepts
- [ ] At least one comparison table included
- [ ] Step-by-step procedure with code block(s)
- [ ] Real-world scenario with measurable outcome in Practical Application
- [ ] Minimum 6 Critical Considerations bullets
- [ ] Critical Synthesis Note contains both a non-obvious insight AND a documented knowledge gap
- [ ] SAA-C03 Exam Focus section present with 5–8 testable bullets (decision rules, traps, hard limits, key distinctions)
- [ ] AWS documentation links present (inline + bottom section for Pattern B)
- [ ] No marketing language or filler words
- [ ] Scope limited to the folder's domain context
