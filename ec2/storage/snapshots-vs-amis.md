# EBS Snapshots vs. AMIs

---

## Core Objective

**Why does this distinction matter?** Snapshots and AMIs are deeply related — an AMI is built on top of snapshots — but they serve different purposes and are the right answer to different problems. Confusing the two leads to using AMIs as backup tools (expensive, operationally wrong) or trying to launch instances from snapshots (not possible).

The core rule: **a snapshot is a storage backup; an AMI is an instance blueprint.** One restores data. The other launches machines.

> **Exam lens:** SAA-C03 scenario questions test whether you know which primitive to use for a given goal: recovering data after volume corruption → snapshot. Launching pre-configured instances consistently → AMI. Moving an instance to another region → AMI copy (which copies its underlying snapshots). The exam also tests that an AMI wraps snapshots — deregistering an AMI does not delete its snapshots.

**AWS Reference:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html

**AWS Reference:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSSnapshots.html

---

## Fundamental Concepts

### Key Terminology

| Term | Definition |
|---|---|
| **EBS Snapshot** | A point-in-time backup of a single EBS volume, stored in AWS-managed S3; can be used to create new volumes or as the source for an AMI root volume |
| **AMI** | A complete instance launch template containing one or more snapshot references (block device mapping), launch permissions, and virtualization metadata |
| **Block device mapping** | The part of an AMI that specifies which snapshot maps to which volume at instance launch — the structural link between AMI and snapshot |
| **AMI deregistration** | Removing an AMI's registration; the underlying snapshots are **not** deleted and continue to exist independently |
| **Root snapshot** | The EBS snapshot of the root volume that an AMI references; this is what gets cloned into the root volume of every instance launched from that AMI |

---

### Side-by-Side Comparison

| Dimension | **EBS Snapshot** | **AMI** |
|---|---|---|
| **What it captures** | State of a single EBS volume | OS + configuration + block device layout of an entire instance |
| **Primary purpose** | Data backup and recovery | Instance launch template |
| **Can launch an instance** | ❌ No — must create a volume first | ✅ Yes — directly |
| **Region scope** | Regional (copyable) | Regional (copyable) |
| **Relationship** | Standalone artifact | Wraps one or more snapshots via block device mapping |
| **Incremental storage** | ✅ Yes — only changed blocks after the first | ❌ No — each AMI creation takes a new full snapshot |
| **Typical use case** | Nightly backups, DR replication, volume cloning | Fleet standardization, Golden AMI, cross-region deployment |
| **Lifecycle on delete** | Data deleted (with incremental chain management) | AMI deregistered; underlying snapshots remain until explicitly deleted |
| **Cost driver** | Storage per GB of changed data | Storage of full root snapshot per AMI version |

---

### The Containment Relationship

```
AMI
├── Metadata (virtualization type, architecture, launch permissions)
├── Block Device Mapping
│     ├── /dev/xvda → snap-aaaa1111  ← root volume snapshot
│     └── /dev/xvdb → snap-bbbb2222  ← additional EBS volume snapshot
└── (launch permissions: private / shared / public)

snap-aaaa1111 (EBS Snapshot)
└── Can exist and be used independently of the AMI
      └── e.g., create a volume from it for data recovery
            without ever launching an instance
```

An AMI **references** snapshots — it does not contain them. If you deregister the AMI, the snapshots remain. If you delete the snapshots while the AMI still references them, the AMI breaks.

---

## Procedure or Logic

### When to Use a Snapshot

- **Step 1 — Identify the need**: You need to recover a specific volume to a past state, copy volume data to a new AZ/region, or back up a database volume nightly.
- **Step 2 — Create a snapshot** of the volume directly.
- **Step 3 — Restore**: Create a new EBS volume from the snapshot in any AZ, attach it to an instance.

### When to Use an AMI

- **Step 1 — Identify the need**: You need to launch multiple identical instances, standardize a fleet, or move a complete instance configuration to another region.
- **Step 2 — Create an AMI** from a configured instance (which internally creates snapshots of all attached volumes).
- **Step 3 — Launch**: Use the AMI ID in a Launch Template or directly in `run-instances`.

```bash
# Snapshot use case: recover a corrupted data volume
aws ec2 create-volume \
  --snapshot-id snap-0abc1234 \
  --availability-zone us-east-1b \
  --volume-type gp3
# Attach the new volume to the instance and mount it

# AMI use case: launch 10 identical pre-configured web servers
aws ec2 run-instances \
  --image-id ami-0abc1234567890def \
  --instance-type t3.medium \
  --count 10 \
  --subnet-id subnet-xxxxxxxx
```

---

## Practical Application / Examples

### Scenario: Choosing the Right Tool After a Ransomware Event

A ransomware attack encrypts data on two `data` EBS volumes attached to a production application server. The root volume (OS + application) is unaffected.

**What to use:**

```
Root volume (OS + app) — UNAFFECTED
  └── No action needed on the instance itself

Data volumes (vol-data-1, vol-data-2) — ENCRYPTED BY RANSOMWARE
  └── Use: EBS Snapshots (taken last night by DLM)
        └── Create new volumes from the clean snapshots
              └── Detach corrupted volumes → attach restored volumes
                    └── Mount and verify data integrity
```

**Why not use an AMI here?** The AMI would restore the entire instance — including the OS and application — to a past state, losing any configuration changes or software updates made since the last AMI was created. A snapshot restore targets only the affected volumes, leaving the healthy root untouched.

**Outcome**: Data restored to last night's state in under 20 minutes with zero impact to the OS or application layer.

---

## Critical Considerations

- **Never delete AMI snapshots while the AMI is still registered**: An AMI references its snapshots by ID in the block device mapping. Deleting a referenced snapshot leaves the AMI in a broken state — instances launched from it will fail to start. Always deregister the AMI before deleting its snapshots.

- **AMI creation always produces a full root snapshot**: Unlike standalone snapshots (which are incremental after the first), each `create-image` call takes a fresh snapshot of the root volume. Frequent AMI creation is storage-expensive; use DLM for volume backups and AMIs only for configuration versioning.

- **Snapshot restores are the right DR primitive; AMIs are the right deployment primitive**: Using AMIs as nightly backups is a common anti-pattern — it's expensive (full snapshot per AMI) and restoring from an AMI to recover data is slower and more disruptive than restoring a volume from a snapshot.

- **Cross-region movement works for both, but via different mechanisms**: Snapshots use `copy-snapshot`; AMIs use `copy-image` (which copies the snapshots internally). The result of `copy-image` is a new AMI ID in the target region backed by new snapshot IDs — fully independent of the source.

---

## Critical Synthesis Note

> **High-level insight — AMIs and snapshots form a two-layer abstraction: storage state vs. compute configuration:**
>
> Snapshots answer "what was on this disk at time T?" AMIs answer "how do I reproduce this machine?" They are complementary, not interchangeable. A mature EC2 operational model maintains both: a snapshot backup strategy (DLM, nightly, cross-region copy) for data recovery, and a Golden AMI pipeline (Image Builder, weekly bake, SSM parameter distribution) for instance standardization. Conflating the two creates either expensive AMI sprawl or a lack of reproducible instance configuration.

> **Knowledge gap requiring further research**: When an AMI is shared between AWS accounts and the recipient copies it, the copy operation creates new snapshots in the recipient's account. It is unclear whether the original account's snapshot costs are affected (i.e., does sharing trigger any S3 replication charges) or whether costs are entirely isolated post-copy.

---

## SAA-C03 Exam Focus

- **Snapshot = restore data; AMI = launch instance** — this is the root decision rule for every scenario question involving these two primitives.
- **AMI wraps snapshots** — an AMI is not self-contained; it references EBS snapshots. Deregister the AMI first, then delete the snapshots, or the snapshots remain and incur cost.
- **Never delete snapshots an AMI still references** — the AMI will be broken and instances launched from it will fail.
- **Cross-region instance portability = copy-image** — copying an AMI to another region copies its snapshots automatically. The new AMI gets a new ID in the target region.
- **Use snapshots for recovery, AMIs for deployment** — the exam frequently presents a scenario where both seem applicable; the presence of "launch identical instances" or "pre-configured fleet" signals AMI; "recover data" or "restore volume" signals snapshot.

---

**EBS Snapshots Documentation:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSSnapshots.html

**AMI Documentation:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html

**Copying AMIs:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/CopyingAMIs.html

**Copying Snapshots:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-copy-snapshot.html

**Deregistering an AMI:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/deregister-ami.html
