# AWS EC2 + Docker + PostgreSQL + S3 — Automated Database Backup System
### (Production-Ready Full Guide — শুরু থেকে শেষ পর্যন্ত)

এই গাইডে আমরা একটি সম্পূর্ণ **automated database backup system** তৈরি করব, যেখানে:

- PostgreSQL চলছে **Docker container**-এ
- Backup নেওয়া হবে **pg_dump** দিয়ে
- Backup **gzip** দিয়ে compress হবে
- **AWS CLI** দিয়ে backup সরাসরি **Amazon S3**-তে upload হবে
- **Cron job** প্রতিদিন automatically backup চালাবে
- **S3 Lifecycle Rule** পুরনো backup নিজে থেকেই delete করে দেবে

---

## 📐 Architecture Overview

```
                    AWS EC2
                       │
        ┌──────────────┴──────────────┐
        │                             │
        ▼                             ▼
 Docker Django                 Docker PostgreSQL
        │                             │
        └──────────────┬──────────────┘
                       │
                   pg_dump
                       │
                       ▼
                    gzip
                       │
                       ▼
                AWS CLI (S3 Upload)
                       │
                       ▼
                  Amazon S3 Bucket
                       │
                       ▼
             Lifecycle Rule (3 Days)
                       │
                       ▼
          Old Backups Deleted Automatically
```

**সংক্ষেপে:** EC2 সার্ভারে Docker-এ চলা PostgreSQL থেকে প্রতিদিন cron job একটি script চালাবে, যেটি ডেটাবেস dump করে, compress করে, এবং সরাসরি S3-তে আপলোড করে দেবে। S3-এর lifecycle rule ৩ দিনের পুরনো backup automatically মুছে ফেলবে, যাতে storage cost বেড়ে না যায়।

---

## ধাপ ১ — S3 Bucket তৈরি করা

AWS Console → **S3** → **Create Bucket**

উদাহরণস্বরূপ bucket-এর নাম:

```
ikonskills-797671034027-ap-southeast-1-an
```

> **টিপ:** Bucket-এর নাম globally unique হতে হয়, তাই সাধারণত account ID এবং region যোগ করে নাম দেওয়া হয়।

---

## ধাপ ২ — EC2-তে AWS CLI ইনস্টল আছে কিনা যাচাই

```bash
aws --version
```

Expected output:

```
aws-cli/2.x.x
```

যদি এই output না আসে, তাহলে AWS CLI ইনস্টল করে নিতে হবে।

---

## ধাপ ৩ — S3 Access টেস্ট করা

```bash
aws s3 ls
```

Expected output:

```
2026-07-02 ikonskills-797671034027-ap-southeast-1-an
```

এর অর্থ হলো, EC2 instance থেকে S3-এ সফলভাবে connect করা যাচ্ছে।

> ⚠️ **Production Tip:** Access Key/Secret Key কখনো সরাসরি সার্ভারে hardcode বা save করে রাখবেন না। এর বদলে EC2 Instance-এর সাথে একটি **IAM Role** attach করুন, যাতে credential leak হওয়ার ঝুঁকি না থাকে।

---

## ধাপ ৪ — PostgreSQL Container-এর নাম বের করা

```bash
docker ps
```

Output-এ container-এর নাম দেখা যাবে, যেমন:

```
lifechoice_postgres_prod
```

---

## ধাপ ৫ — Database তথ্য নোট করা

```
Database Name : lifechoice
Database User : lifechoice
```

এই তথ্যগুলো backup script-এ ব্যবহার হবে।

---

## ধাপ ৬ — Backup Script তৈরি

প্রথমে script file তৈরি করুন:

```bash
nano /home/ubuntu/api/backup.sh
```

তারপর নিচের কোডটি লিখুন:

```bash
#!/bin/bash

set -euo pipefail

CONTAINER_NAME="lifechoice_postgres_prod"
DB_NAME="lifechoice"
DB_USER="lifechoice"

S3_BUCKET="ikonskills-797671034027-ap-southeast-1-an"
S3_PREFIX="database-backups"

TIMESTAMP=$(date +"%Y-%m-%d_%H-%M-%S")
BACKUP_FILE="${DB_NAME}_${TIMESTAMP}.sql.gz"

echo "Backup Started: $(date)"

docker exec "$CONTAINER_NAME" \
    pg_dump -U "$DB_USER" "$DB_NAME" \
| gzip \
| aws s3 cp - "s3://${S3_BUCKET}/${S3_PREFIX}/${BACKUP_FILE}"

echo "Backup Completed: $(date)"
```

---

## ধাপ ৭ — Script-কে Executable করা

```bash
chmod +x /home/ubuntu/api/backup.sh
```

এই command script ফাইলটিকে run করার permission দেয়।

---

## ধাপ ৮ — Manual Test

```bash
bash /home/ubuntu/api/backup.sh
```

Expected output:

```
Backup Started...
upload: ...
Backup Completed...
```

---

## ধাপ ৯ — S3-তে Backup Verify করা

```bash
aws s3 ls s3://ikonskills-797671034027-ap-southeast-1-an/database-backups/
```

এই command দিয়ে দেখা যাবে backup ফাইলটি সত্যিই S3-তে upload হয়েছে কিনা।

---

## ধাপ ১০ — Cron Job Setup

Cron edit করার জন্য:

```bash
crontab -e
```

বাংলাদেশ সময় রাত ২টায় (Asia/Dhaka) backup চালাতে চাইলে, এবং সার্ভার যদি **UTC** timezone-এ থাকে:

```
0 20 * * * /home/ubuntu/api/backup.sh >> /home/ubuntu/api/backup.log 2>&1
```

কারণ: `20:00 UTC` = `02:00 Asia/Dhaka` (পরের দিন)।

যদি সার্ভারের timezone আগে থেকেই **Asia/Dhaka** সেট করা থাকে, তাহলে শুধু এটি যথেষ্ট:

```
0 2 * * *
```

---

## ধাপ ১১ — Cron Entry Verify করা

```bash
crontab -l
```

এই command দিয়ে বর্তমান cron job list দেখা যাবে।

---

## ধাপ ১২ — Cron Service চেক করা

```bash
sudo systemctl status cron
```

Expected output:

```
Active: active (running)
```

এটি নিশ্চিত করে যে cron daemon সচল আছে এবং scheduled task চালাতে পারবে।

---

## ধাপ ১৩ — Log পর্যবেক্ষণ করা

```bash
tail -f /home/ubuntu/api/backup.log
```

এই command দিয়ে backup log real-time-এ দেখা যায় — কোনো সমস্যা হলে সাথে সাথে ধরা পড়ে।

---

## ধাপ ১৪ — S3 Lifecycle Rule (পুরনো Backup Auto-Delete)

প্রথমে `lifecycle.json` নামে একটি ফাইল তৈরি করুন:

```json
{
  "Rules": [
    {
      "ID": "DeleteOldBackups",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "database-backups/"
      },
      "Expiration": {
        "Days": 3
      }
    }
  ]
}
```

এটি apply করার command:

```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket ikonskills-797671034027-ap-southeast-1-an \
  --lifecycle-configuration file://lifecycle.json
```

Verify করার command:

```bash
aws s3api get-bucket-lifecycle-configuration \
  --bucket ikonskills-797671034027-ap-southeast-1-an
```

এই rule-এর মাধ্যমে `database-backups/` prefix-এর যেকোনো ফাইল ৩ দিন পুরনো হয়ে গেলে S3 নিজে থেকেই মুছে ফেলবে — ফলে storage cost নিয়ন্ত্রণে থাকবে।

---

## ধাপ ১৫ — Backup থেকে Restore করা

**১. S3 থেকে Download করা:**

```bash
aws s3 cp \
s3://ikonskills-797671034027-ap-southeast-1-an/database-backups/lifechoice_2026-07-23_08-58-53.sql.gz .
```

**২. Extract (Decompress) করা:**

```bash
gunzip lifechoice_2026-07-23_08-58-53.sql.gz
```

**৩. Database-এ Restore করা:**

```bash
docker exec -i lifechoice_postgres_prod \
psql -U lifechoice lifechoice \
< lifechoice_2026-07-23_08-58-53.sql
```

---

## 🔍 প্রতিটি Command-এর বিস্তারিত ব্যাখ্যা

### `pg_dump`

```bash
pg_dump -U lifechoice lifechoice
```

| অংশ | ব্যাখ্যা |
|---|---|
| `pg_dump` | PostgreSQL-এর নিজস্ব backup utility |
| `-U lifechoice` | Database user নির্দেশ করে |
| `lifechoice` (শেষে) | Database-এর নাম |

### Pipe (`|`)

```bash
pg_dump | gzip
```

এটি একটি **pipe**। `pg_dump`-এর output সরাসরি `gzip`-এ পাঠানো হয় — মাঝখানে কোনো temporary file তৈরি করার প্রয়োজন হয় না।

### `gzip`

Backup file-কে compress করে, ফলে file size অনেক কমে যায়:

```
Before : 50 MB
After  :  8 MB
```

### `aws s3 cp -`

```bash
aws s3 cp - s3://bucket/file.sql.gz
```

এখানে `-` মানে হলো — **STDIN থেকে data নাও**। অর্থাৎ কোনো local file তৈরি না করেই সরাসরি stream আকারে S3-তে upload হয়ে যায়।

### `docker exec`

```bash
docker exec container pg_dump ...
```

এই command container-এর **ভিতরে ঢুকে** কমান্ড execute করে — অর্থাৎ container-এর ভিতরের PostgreSQL-এর ওপর সরাসরি কাজ করে।

### `set -euo pipefail`

| Flag | ব্যাখ্যা |
|---|---|
| `-e` | কোনো command fail করলে পুরো script সাথে সাথে বন্ধ হয়ে যাবে |
| `-u` | কোনো undefined variable ব্যবহার করলে error দেখাবে |
| `pipefail` | Pipeline-এর যেকোনো একটি ধাপ fail করলেও পুরো pipeline fail হিসেবে গণ্য হবে |

এই তিনটি flag একসাথে script-কে অনেক বেশি নিরাপদ এবং reliable করে তোলে — কোনো নীরব (silent) failure ধরা পড়বে না।

---

## 🔄 সম্পূর্ণ Workflow — এক নজরে

```
Cron
 │
 ▼
backup.sh
 │
 ▼
Docker Exec
 │
 ▼
pg_dump
 │
 ▼
gzip
 │
 ▼
AWS CLI
 │
 ▼
Amazon S3
 │
 ▼
Lifecycle Rule
 │
 ▼
Delete Old Backups
```

---

## ✅ Industry Best Practices

1. **IAM Role** ব্যবহার করুন — Access Key/Secret সার্ভারে কখনো hardcode করে রাখবেন না।
2. Backup নেওয়ার পর নিয়মিত **restore test** করুন — backup কাজ করছে কিনা তা নিশ্চিত করার একমাত্র উপায় এটিই।
3. Backup **encrypt** করুন (S3 server-side encryption ব্যবহার করে)।
4. S3 Bucket-এ **public access সম্পূর্ণ বন্ধ** রাখুন।
5. প্রয়োজন অনুযায়ী **Versioning** চালু করবেন কি না তা আপনার recovery policy অনুযায়ী ঠিক করুন।
6. Backup logs **monitor** করুন (CloudWatch বা অন্য কোনো monitoring system দিয়ে)।
7. Cron job fail হলে **notification** (Email বা Slack) পাঠানোর ব্যবস্থা রাখুন, যাতে backup miss হলে সাথে সাথে জানা যায়।

---

## 📌 উপসংহার

এই পুরো প্রক্রিয়াটি ছোট থেকে মাঝারি আকারের **Docker-ভিত্তিক PostgreSQL application**-এর জন্য একটি চমৎকার production pattern।

বড় আকারের PostgreSQL deployment-এর ক্ষেত্রে সাধারণত ব্যবহার করা হয়:

- **Point-in-Time Recovery (PITR)**
- **WAL Archiving**
- **pgBackRest**-এর মতো advanced tool

কিন্তু বর্তমান setup-এর জন্য —

> **pg_dump + gzip + S3 + Cron + Lifecycle Rule**

— এটিই একটি বাস্তবসম্মত, সহজ এবং নির্ভরযোগ্য সমাধান।
