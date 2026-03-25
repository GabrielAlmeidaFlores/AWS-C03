# EBS Snapshots

---

## Core Objective

**Why does this exist?** EBS volumes are tied to a single Availability Zone — they cannot natively survive an AZ failure or be moved across regions. Snapshots solve this by capturing the state of an EBS volume as a durable, incremental backup stored in AWS-managed S3, decoupled from any AZ.

The result: point-in-time recovery, cross-region disaster recovery, volume cloning, and encryption migration — all driven from a single primitive.

> **Exam lens:** The most tested snapshot concepts are: incremental mechanics (only changed blocks are stored after the first snapshot), the encryption chain (encrypted volume → encrypted snapshot → only encrypted volumes), and cross-region copy as the DR mechanism for EBS data.

**AWS Reference:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSSnapshots.html

---

## Fundamental Concepts

### Key Terminology

| Term | Definition |
|---|---|
| **Snapshot** | A point-in-time copy of an EBS volume stored in AWS-managed S3; the source volume remains available during snapshot creation |
| **Incremental backup** | Each snapshot after the first stores only the blocks that changed since the previous snapshot — not a full copy each time |
| **Fast Snapshot Restore (FSR)** | An opt-in feature that pre-warms snapshot data so volumes restored from it deliver full performance immediately, eliminating the lazy-loading I/O penalty |
| **Data Lifecycle Manager (DLM)** | AWS service that automates snapshot creation, retention, and deletion on a schedule using tag-based policies |
| **Snapshot Archive** | A lower-cost storage tier (~75% cheaper) for snapshots you rarely need; restoring an archived snapshot takes 24–72 hours |
| **Recycle Bin** | A retention policy feature that holds deleted snapshots for a configurable period before permanent deletion, enabling accidental-deletion recovery |
| **Multi-volume snapshot** | A coordinated, crash-consistent snapshot of all volumes attached to an EC2 instance taken simultaneously |
| **Snapshot encryption** | Snapshots of encrypted volumes are always encrypted; unencrypted snapshots can be encrypted during copy |

---

### Snapshot Incremental Chain

| Snapshot | What Is Stored | What You Pay For |
|---|---|---|
| **First snapshot** | All used blocks on the volume | All used data |
| **Subsequent snapshots** | Only blocks changed since the previous snapshot | Only the changed delta |
| **After deleting intermediate snapshots** | Unchanged blocks are merged into remaining snapshots | AWS handles consolidation automatically — no data loss |

---

## Procedure or Logic

- **Step 1 — Create a snapshot**: Snapshots can be taken while the volume is in use. For data consistency on databases, flush writes or use `fsfreeze` before initiating.

- **Step 2 — Snapshot stored in S3**: AWS manages the S3 bucket; the snapshot is not visible in your S3 console. It is referenced by snapshot ID (`snap-xxxxxxxx`).

- **Step 3 — Restore to a new volume**: Create a new EBS volume from the snapshot. The volume is available immediately but data is loaded lazily from S3 in the background — first-access I/O is slow until blocks are loaded (unless FSR is enabled).

- **Step 4 — Copy across regions (for DR)**: Use snapshot copy to replicate a snapshot to another region. Encryption and KMS key can be changed during copy.

```bash
# Create a snapshot
aws ec2 create-snapshot \
  --volume-id vol-0abc1234567890def \
  --description "pre-deployment backup $(date +%Y-%m-%d)"

# Copy snapshot to another region (enables cross-region DR)
aws ec2 copy-snapshot \
  --source-region us-east-1 \
  --source-snapshot-id snap-0abc1234567890def \
  --destination-region eu-west-1 \
  --description "DR copy"

# Create a volume from a snapshot in a specific AZ
aws ec2 create-volume \
  --snapshot-id snap-0abc1234567890def \
  --availability-zone us-east-1a \
  --volume-type gp3
```

---

### Automating Snapshots with DLM

```json
{
  "Description": "Daily snapshots with 7-day retention",
  "State": "ENABLED",
  "PolicyDetails": {
    "ResourceTypes": ["VOLUME"],
    "TargetTags": [{"Key": "Backup", "Value": "daily"}],
    "Schedules": [{
      "Name": "DailyBackup",
      "CreateRule": {"Interval": 24, "IntervalUnit": "HOURS", "Times": ["03:00"]},
      "RetainRule": {"Count": 7}
    }]
  }
}
```

---

## Practical Application / Examples

### Scenario: Cross-Region Disaster Recovery for a Production Database Volume

A production PostgreSQL instance runs on a `r6g.xlarge` with a 500 GB `io2` EBS volume in `us-east-1`. RTO is 1 hour; RPO is 24 hours. The team needs automated daily backups replicated to `eu-west-1`.

**Architecture:**

```
us-east-1 (primary)
  EC2 (r6g.xlarge) + io2 volume (500 GB)
    └── DLM policy: snapshot daily at 03:00 UTC, retain 7 days
          └── EventBridge rule: on snapshot completion
                └── Lambda: copy snapshot to eu-west-1

eu-west-1 (DR region)
  snap-xxxxxxxx (replicated daily)
    └── On DR activation: create volume → attach to new EC2 instance
```

**Outcome**: RPO of ~24 hours. RTO under 30 minutes (create volume + launch instance). Snapshot storage cost: ~$0.05/GB/month × 500 GB × 7 snapshots (incremental — actual cost far lower after day 1).

---

## Critical Considerations

- **Consistency requires application cooperation**: Snapshots are crash-consistent by default — safe for most use cases, but databases may have in-flight transactions. Quiesce writes (`fsfreeze`, `FLUSH TABLES WITH READ LOCK`) before snapshotting or use multi-volume snapshots for instance-level consistency.

- **Lazy loading causes first-access latency on restored volumes**: A volume restored from a snapshot delivers full performance only after all blocks are pulled from S3. For production restores where latency matters, enable **Fast Snapshot Restore** on the target AZ before restoring.

- **Encryption is one-way and inherited**: Snapshots of encrypted volumes are always encrypted. You cannot create an unencrypted snapshot from an encrypted volume. To encrypt an unencrypted snapshot, copy it with encryption enabled — this creates a new, separate encrypted snapshot.

- **Deleting snapshots is safe even in the middle of a chain**: AWS automatically manages block references between snapshots. Deleting an intermediate snapshot never causes data loss — referenced blocks are merged into adjacent snapshots.

- **DLM does not snapshot instance store volumes**: DLM manages only EBS volumes. Instance store data is ephemeral and cannot be snapshotted.

- **Snapshot copy does not create a live volume**: Copying a snapshot to another region produces another snapshot, not a usable volume. You must explicitly create a volume from it in the target region to use it.

- **Recycle Bin must be configured before deletion**: The Recycle Bin only retains snapshots that were deleted after a retention rule was applied. Snapshots deleted before a rule existed are gone permanently.

---

## Critical Synthesis Note

> **High-level insight — Snapshots are the portability primitive for EBS data, not just a backup mechanism:**
>
> Every cross-AZ, cross-region, and cross-account movement of EBS data flows through snapshots. Volume encryption migration, AMI creation, and DR replication all use snapshots as the underlying transport. Understanding this makes it clear why snapshot management (retention, encryption, replication) deserves the same operational rigor as the volumes themselves.

> **Knowledge gap requiring further research**: The performance characteristics of FSR across different volume types (`gp3` vs `io2`) and whether FSR pre-warming is instant or probabilistic under concurrent access patterns from multiple restored instances is not clearly documented.

---

## SAA-C03 Exam Focus

- **Incremental after the first**: Only the first snapshot is a full copy. Deleting intermediate snapshots is safe — AWS consolidates blocks automatically.
- **Cross-region copy = the EBS DR mechanism**: Snapshots cannot span regions natively; explicit copy is required. This is the answer whenever a question asks how to make EBS data available in another region.
- **Encrypted volume → encrypted snapshot → encrypted volume only**: You cannot break the encryption chain. Unencrypted → encrypted is possible only via a copy operation.
- **Lazy loading trap**: Restored volumes are immediately available but slow until blocks load from S3. FSR is the fix — but it costs extra and must be enabled per AZ.
- **DLM = automated snapshot lifecycle**: If the question asks about automating backups with retention policies for EBS, the answer is DLM (not Lambda, not CloudWatch Events alone).
- **Snapshots live in S3 but are not in your S3 bucket**: You cannot see or manage them via the S3 console; use the EC2 API/console only.

---

**Primary AWS Documentation:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSSnapshots.html
**Fast Snapshot Restore:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-fast-snapshot-restore.html
**Data Lifecycle Manager:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/snapshot-lifecycle.html
**Snapshot Recycle Bin:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/recycle-bin.html
**Copying Snapshots:** https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-copy-snapshot.html
