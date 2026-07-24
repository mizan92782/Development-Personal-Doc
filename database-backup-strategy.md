# Database Backup Strategy
### PostgreSQL (Docker) on AWS EC2 — with Server Time (UTC) Reference

---

## 1. Overview

This document defines the backup strategy for the production PostgreSQL database running inside a Docker container on our AWS EC2 instance. It covers scheduling, storage, retention, security, monitoring, and disaster recovery — with explicit attention to **server time**, since the server operates on **UTC**, not local Bangladesh time.

---

## 2. Server Time Reference

| Item | Value |
|---|---|
| Server time zone | **UTC** (AWS EC2 default) |
| Target local backup time | 2:00 AM Bangladesh Time (Asia/Dhaka, UTC+6) |
| Equivalent server (cron) time | **20:00 UTC** (previous day) |
| Verify server time | `date` or `timedatectl` |

> **Why this matters:** Every timestamp in cron, backup logs, and S3 filenames is recorded in UTC. A backup file timestamped `2026-07-23_20-00-01` was actually taken at **2:00 AM Dhaka time on July 24**. This offset must be accounted for whenever reading logs or reasoning about "which night's backup is this."

**Time conversion table (for quick reference):**

| Bangladesh Time (local) | Server Time (UTC) |
|---|---|
| 12:00 AM | 18:00 (previous day) |
| 2:00 AM | 20:00 (previous day) |
| 6:00 AM | 00:00 (same day) |
| 12:00 PM (noon) | 06:00 |
| 6:00 PM | 12:00 |

---

## 3. Backup Schedule

Backups run **once daily**, automatically, via cron.

```
0 20 * * * /home/ubuntu/api/backup.sh >> /home/ubuntu/api/backup.log 2>&1
```

- **Cron time:** `20:00 UTC` daily
- **Real-world time:** `2:00 AM Asia/Dhaka` daily
- **Frequency:** 1x per day (sufficient for current data-change volume; can be increased if needed — see Section 8)

If the server's time zone is ever changed to `Asia/Dhaka`, this cron line must be updated to `0 2 * * *` to preserve the same real-world backup time.

---

## 4. Backup Process

| Step | Action | Tool |
|---|---|---|
| 1 | Dump live database from inside the PostgreSQL Docker container | `pg_dump` via `docker exec` |
| 2 | Compress the dump to reduce size/cost | `gzip` |
| 3 | Stream-upload directly to S3 (no local file left on server) | `aws s3 cp -` |
| 4 | Log start/completion for auditing | `backup.sh` → `backup.log` |

**Core command:**

```bash
docker exec lifechoice_postgres_prod \
    pg_dump -U lifechoice lifechoice \
| gzip \
| aws s3 cp - s3://ikonskills-797671034027-ap-southeast-1-an/database-backups/lifechoice_$(date +%Y-%m-%d_%H-%M-%S).sql.gz
```

Note: the `$(date ...)` in the filename is also **server time (UTC)** — a file named `lifechoice_2026-07-23_20-00-01.sql.gz` corresponds to `2:00 AM Dhaka time on July 24`.

---

## 5. Storage Location

| Item | Value |
|---|---|
| Storage service | Amazon S3 |
| Bucket | `ikonskills-797671034027-ap-southeast-1-an` |
| Path (prefix) | `database-backups/` |
| Access | Private bucket, no public access |
| Server → S3 authentication | IAM Role attached to EC2 instance (no hardcoded credentials) |

---

## 6. Retention Policy

Retention is enforced via an **S3 Lifecycle Rule**, not a fixed-count rule:

```json
{
  "Rules": [
    {
      "ID": "DeleteOldBackups",
      "Status": "Enabled",
      "Filter": { "Prefix": "database-backups/" },
      "Expiration": { "Days": 3 }
    }
  ]
}
```

- Any object under `database-backups/` is **automatically deleted 3 days after creation**.
- This is **age-based**, not count-based — the bucket may briefly hold more than 3 files (e.g. during manual test runs) or, if backups are ever interrupted for more than 3 days, temporarily hold zero.
- This approach was chosen intentionally for simplicity and low storage cost; see Section 8 for options if stricter "always keep exactly N backups" behavior is needed later.

---

## 7. Restore Procedure

In the event of data loss or corruption, restore using the most recent backup:

```bash
# 1. Download from S3
aws s3 cp s3://ikonskills-797671034027-ap-southeast-1-an/database-backups/<backup-file>.sql.gz .

# 2. Decompress
gunzip <backup-file>.sql.gz

# 3. Restore into the running container
docker exec -i lifechoice_postgres_prod \
    psql -U lifechoice lifechoice < <backup-file>.sql
```

**Recommended:** periodically test this restore process on a non-production database to confirm backups are actually usable, not just present.

---

## 8. Monitoring & Alerting

**Current state:** backup success/failure is recorded in `backup.log` on the server, and must be checked manually:

```bash
tail -30 /home/ubuntu/api/backup.log
```

**Recommended next step (not yet implemented):** add automated failure notification (email or Slack) so a failed backup is flagged immediately instead of discovered later. This is the single highest-value improvement to this strategy going forward.

---

## 9. Security Practices in Place

- ✅ IAM Role used for S3 access (no access keys stored on server)
- ✅ S3 bucket is private, public access blocked
- ✅ Backups are compressed before leaving the server
- ⬜ Server-side encryption on S3 objects — recommended, not yet confirmed enabled
- ⬜ S3 versioning — optional, depends on desired recovery granularity

---

## 10. Known Limitations & Future Improvements

| Limitation | Impact | Suggested Fix |
|---|---|---|
| Retention is age-based, not count-based | Manual/test backups can temporarily exceed "3 backups" | Add script-side cleanup to always retain exactly the newest N backups |
| No failure alerting | A failed backup may go unnoticed until a restore is needed | Add Slack/email notification on script failure |
| Single daily backup | Up to ~24 hours of data could be lost in a worst-case failure | Increase frequency (e.g. every 6–12 hours) if data-change volume justifies it |
| No PITR (Point-in-Time Recovery) | Can only restore to the last daily snapshot, not to a specific moment | Consider WAL archiving or `pgBackRest` if finer recovery granularity becomes necessary |

For the current scale of the application, the existing strategy (`pg_dump` + `gzip` + S3 + Cron + Lifecycle Rule) is a solid, low-maintenance, production-appropriate solution. The items above are optional hardening steps, not urgent gaps.

---

## 11. Quick Reference Summary

| Question | Answer |
|---|---|
| When does backup run? | 20:00 UTC daily = 2:00 AM Bangladesh time |
| Where is it stored? | `s3://ikonskills-797671034027-ap-southeast-1-an/database-backups/` |
| How long is it kept? | 3 days, then auto-deleted |
| How do I check it ran? | `tail -30 /home/ubuntu/api/backup.log` |
| How do I restore? | See Section 7 |
