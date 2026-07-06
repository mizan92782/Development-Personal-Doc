# Module 6: PostgreSQL Backup (পোস্টগ্রেসকিউএল ব্যাকআপ)

> **Level:** Intermediate → Advanced (Production-Level)
> **Prerequisite:** Module 1-5 সম্পন্ন থাকতে হবে (বিশেষ করে WAL, PITR, ACID ধারণা)।

---

## 6.0 Chapter Roadmap

```
6.1  pg_dump — Logical Backup
6.2  pg_dumpall — সব Database একসাথে
6.3  pg_basebackup — Physical Base Backup
6.4  WAL বিস্তারিত (PostgreSQL Context এ)
6.5  Archive Mode
6.6  Streaming Replication
6.7  PITR (PostgreSQL এ বাস্তব প্রয়োগ)
6.8  Recovery Process
6.9  Restore Process (সম্পূর্ণ)
6.10 Automation
6.11 Production Example
6.12 Best Practices
6.13 Common Mistakes
6.14 Interview Questions
6.15 Summary
6.16 Exercises & Assignments
```

---

## 6.1 pg_dump — Logical Backup

### সংজ্ঞা

`pg_dump` হলো PostgreSQL এর প্রধান **Logical Backup Tool**, যা একটা নির্দিষ্ট Database কে SQL Script বা Custom Binary Format এ Export করে।

### মূল ব্যবহার

```bash
# Plain SQL Format এ Backup (Human-readable)
pg_dump -U postgres -d mydb -f backup.sql

# Custom Format এ Backup (Compressed, Restore এ Flexible)
pg_dump -U postgres -d mydb -F c -f backup.dump

# Directory Format এ Backup (Parallel Dump/Restore সমর্থন করে)
pg_dump -U postgres -d mydb -F d -j 4 -f backup_dir/

# শুধু Schema
pg_dump -U postgres -d mydb --schema-only -f schema.sql

# শুধু Data
pg_dump -U postgres -d mydb --data-only -f data.sql

# নির্দিষ্ট Table
pg_dump -U postgres -d mydb -t users -f users_backup.sql
```

### pg_dump এর Format সমূহ — বিস্তারিত পার্থক্য

| Format | Flag | বৈশিষ্ট্য |
|---|---|---|
| Plain SQL | `-F p` (Default) | Human-readable, `psql` দিয়ে সরাসরি Restore করা যায় |
| Custom | `-F c` | Compressed, Selective Restore সম্ভব, `pg_restore` লাগে |
| Directory | `-F d` | Parallel Backup/Restore সমর্থন করে (বড় DB তে সেরা) |
| Tar | `-F t` | Tape Archive Format, কম ব্যবহৃত হয় |

### Consistency: pg_dump কীভাবে কাজ করে (MVCC এর মাধ্যমে)

PostgreSQL এর **MVCC (Multi-Version Concurrency Control)** পদ্ধতির কারণে `pg_dump` স্বয়ংক্রিয়ভাবে একটা Consistent Snapshot পায়, MySQL এর মতো আলাদা কোনো `--single-transaction` Flag লাগে না।

```
Diagram: MVCC কীভাবে pg_dump কে Consistent রাখে

pg_dump শুরু হলো
      │
      ▼
একটা Transaction শুরু হয় (Repeatable Read Isolation Level এ)
      │
      ▼
এই মুহূর্তের একটা Snapshot নেওয়া হয় (MVCC ব্যবহার করে)
      │
      ▼
Dump চলাকালীন অন্য Session নতুন ডেটা লিখলেও,
এই Transaction সবসময় শুরুর মুহূর্তের ডেটাই দেখবে
(পুরনো Row Version গুলো এখনো উপলব্ধ থাকে MVCC এর কারণে)
      │
      ▼
Dump সম্পূর্ণ হলে Transaction শেষ হয়
```

### বাস্তব উদাহরণ: Restore করা

```bash
# Plain SQL Restore
psql -U postgres -d mydb -f backup.sql

# Custom Format Restore (pg_restore ব্যবহার করে)
pg_restore -U postgres -d mydb backup.dump

# Directory Format থেকে Parallel Restore
pg_restore -U postgres -d mydb -j 4 backup_dir/

# শুধু নির্দিষ্ট Table Restore করা (Custom Format থেকে Selective)
pg_restore -U postgres -d mydb -t users backup.dump
```

---

## 6.2 pg_dumpall — সব Database একসাথে

### সংজ্ঞা

`pg_dumpall` পুরো PostgreSQL Server এর **সব Database, Role (User), এবং Tablespace** একসাথে Backup নেয় — যেখানে `pg_dump` শুধু একটা নির্দিষ্ট Database Backup নেয়।

### মূল ব্যবহার

```bash
# সব Database Backup
pg_dumpall -U postgres -f all_databases_backup.sql

# শুধু Role/User Information Backup (কোনো Data ছাড়া)
pg_dumpall -U postgres --roles-only -f roles_backup.sql

# শুধু Tablespace Information
pg_dumpall -U postgres --tablespaces-only -f tablespaces_backup.sql
```

### pg_dump vs pg_dumpall

| বৈশিষ্ট্য | pg_dump | pg_dumpall |
|---|---|---|
| পরিসর | একটা নির্দিষ্ট Database | সব Database + Global Object |
| Format | Plain/Custom/Directory/Tar | শুধু Plain SQL |
| User/Role Backup | না | হ্যাঁ |
| Parallel Support | হ্যাঁ (Directory Format এ) | না |
| ব্যবহার | নিয়মিত Backup, Selective Restore | সম্পূর্ণ Server Migration |

### Production Note

Production এ সাধারণত `pg_dumpall --roles-only` দিয়ে শুধু User/Role Backup নেওয়া হয় (এটা ছোট), আর প্রতিটা Database আলাদাভাবে `pg_dump` দিয়ে Backup নেওয়া হয় (কারণ এতে Parallel এবং Selective Restore এর সুবিধা পাওয়া যায়)।

---

## 6.3 pg_basebackup — Physical Base Backup

### সংজ্ঞা

`pg_basebackup` হলো PostgreSQL এর Built-in **Physical Backup Tool**, যা সম্পূর্ণ Data Directory (সব Data File) Copy করে — Database Running অবস্থাতেই, কোনো Downtime ছাড়া (Hot Backup)।

### মূল ব্যবহার

```bash
# Plain Format এ Base Backup
pg_basebackup -h localhost -U replication_user \
    -D /backup/base/ -Fp -Xs -P

# Compressed Tar Format এ (Space বাঁচাতে)
pg_basebackup -h localhost -U replication_user \
    -D /backup/base/ -Ft -z -Xs -P
```

### Flag সমূহের ব্যাখ্যা

| Flag | ব্যাখ্যা |
|---|---|
| `-D` | Backup Destination Directory |
| `-Fp` | Plain Format (সরাসরি Directory Structure) |
| `-Ft` | Tar Format (Compressed) |
| `-Xs` | WAL Streaming Mode — Backup চলাকালীন WAL ও সাথে সাথে সংগ্রহ করে |
| `-P` | Progress দেখায় |
| `-z` | Compress করে (gzip দিয়ে) |

### Internal Working

```
Diagram: pg_basebackup কীভাবে Consistency নিশ্চিত করে

pg_basebackup শুরু
      │
      ▼
"Start Backup" Signal পাঠানো হয় (একটা Checkpoint তৈরি হয়)
      │
      ▼
সব Data File Copy করা শুরু হয় (Database চলমান অবস্থায়)
      │
      ▼
Copy চলাকালীন যেসব WAL তৈরি হয়, তা -Xs Flag দিয়ে
সাথে সাথে Stream করে সংগ্রহ করা হয়
      │
      ▼
"Stop Backup" Signal পাঠানো হয়
      │
      ▼
Backup সম্পূর্ণ — Data File + প্রয়োজনীয় সব WAL একসাথে আছে
(Restore এর সময় এই WAL Apply করে Consistency নিশ্চিত হবে)
```

### কেন শুধু Data File Copy যথেষ্ট না (WAL ছাড়া)

যেহেতু Backup চলাকালীন Database Live থাকে, শুরুতে Copy হওয়া একটা File এবং শেষে Copy হওয়া আরেকটা File ভিন্ন ভিন্ন সময়ের অবস্থায় থাকতে পারে। `-Xs` (WAL Stream) Flag এই ব্যবধান পূরণ করার জন্য প্রয়োজনীয় WAL Record ও সংগ্রহ করে রাখে, যাতে Restore এর সময় সব ফাইল একটা নির্দিষ্ট, Consistent মুহূর্তে নিয়ে আসা যায়।

---

## 6.4 WAL বিস্তারিত (PostgreSQL Context এ)

Module 2-এ WAL এর প্রাথমিক ধারণা শেখা হয়েছিল। এখানে PostgreSQL-এ এর Practical ব্যবহার শিখবো।

### WAL ফাইল Location ও Structure

```bash
# WAL ফাইল থাকে এখানে
ls -la /var/lib/postgresql/14/main/pg_wal/

# উদাহরণ Output:
# 000000010000000000000001
# 000000010000000000000002
# 000000010000000000000003
```

প্রতিটা WAL ফাইল সাধারণত ১৬ MB (Default), এবং এগুলো ক্রমানুসারে তৈরি হয়।

### WAL সংক্রান্ত গুরুত্বপূর্ণ Configuration

```ini
# postgresql.conf ফাইলে

wal_level = replica          # WAL এ কতটুকু তথ্য রাখা হবে (minimal/replica/logical)
max_wal_size = 1GB           # সর্বোচ্চ WAL সাইজ Checkpoint এর আগে
min_wal_size = 80MB          # সর্বনিম্ন WAL সাইজ রাখা হবে
wal_keep_size = 512MB        # Replica এর জন্য কতটুকু পুরনো WAL রাখা হবে
```

### wal_level এর বিভিন্ন মান

| মান | ব্যাখ্যা |
|---|---|
| minimal | শুধু Crash Recovery এর জন্য যথেষ্ট তথ্য (Replication/Archiving সমর্থন করে না) |
| replica | Replication এবং WAL Archiving এর জন্য পর্যাপ্ত (সবচেয়ে বেশি ব্যবহৃত) |
| logical | Logical Replication এর জন্য অতিরিক্ত তথ্য রাখে |

---

## 6.5 Archive Mode (আর্কাইভ মোড)

### সংজ্ঞা

**Archive Mode** Enable করলে PostgreSQL প্রতিটা পূর্ণ হওয়া WAL Segment একটা নির্দিষ্ট Location এ Copy/Archive করে রাখে, যা পরবর্তীতে PITR এর জন্য ব্যবহার করা যায় — Module 5-এ MySQL এ যেভাবে Binlog Archive করা হয়, একই ধারণা।

### Archive Mode Enable করা

```ini
# postgresql.conf ফাইলে

archive_mode = on
archive_command = 'cp %p /archive/wal/%f'

# অথবা Cloud Storage এ সরাসরি (উদাহরণ - S3 এ)
archive_command = 'aws s3 cp %p s3://my-wal-archive-bucket/%f'
```

`%p` মানে হলো WAL ফাইলের Full Path, এবং `%f` মানে হলো শুধু ফাইলের নাম।

### Diagram: Archive Mode Working

```
WAL Segment পূর্ণ হলো (16MB)
        │
        ▼
PostgreSQL archive_command চালায়
        │
        ▼
WAL ফাইল Archive Location এ Copy হয়
(Local Disk / NFS / S3 / অন্য কোনো Storage)
        │
        ▼
Archive Location এ WAL জমতে থাকে ক্রমানুসারে
        │
        ▼
এই সংগৃহীত WAL গুলো ব্যবহার করে PITR করা সম্ভব হয়
```

### Archive Command Verify করা

```bash
# psql এ চেক করা
SHOW archive_mode;
SHOW archive_command;

# Archive Status দেখা
SELECT * FROM pg_stat_archiver;
```

---

## 6.6 Streaming Replication

### সংজ্ঞা

**Streaming Replication** হলো PostgreSQL এর একটা Feature, যেখানে WAL Record Real-time এ Primary Server থেকে একটা Replica (Standby) Server এ পাঠানো হয়, যাতে Replica সবসময় Primary এর প্রায় সমান্তরাল (Near Real-time) অবস্থায় থাকে।

### Architecture Diagram

```
┌────────────────┐                  ┌────────────────┐
│  Primary Server   │   WAL Stream     │  Standby Server   │
│  (Read + Write)    │─────────────────►│  (Read-only,       │
│                     │   (Real-time)     │   Backup Purpose)   │
└────────────────┘                  └────────────────┘
```

### Setup করা (সংক্ষিপ্ত)

```bash
# Primary Server এ postgresql.conf
wal_level = replica
max_wal_senders = 5
wal_keep_size = 1GB

# pg_hba.conf এ Replica Server কে অনুমতি দেওয়া
host replication replicator_user standby_ip/32 md5

# Standby Server এ Base Backup নিয়ে সেটআপ করা
pg_basebackup -h primary_ip -U replicator_user \
    -D /var/lib/postgresql/14/main -Fp -Xs -P -R
```

`-R` Flag স্বয়ংক্রিয়ভাবে `standby.signal` ফাইল এবং প্রয়োজনীয় Connection Information তৈরি করে দেয়।

### Streaming Replication থেকে Backup নেওয়া

Module 5-এ MySQL Replica থেকে Backup নেওয়ার মতোই, PostgreSQL Standby Server থেকে Backup নিলে Primary Server এর Performance এ প্রভাব পড়ে না।

```bash
# Standby Server এ Backup নেওয়া
pg_basebackup -h standby_server -U backup_user -D /backup/ -Fp -Xs
```

---

## 6.7 PITR (PostgreSQL এ বাস্তব প্রয়োগ)

### সম্পূর্ণ PITR Setup ও Process

```
Diagram: PostgreSQL PITR এর সম্পূর্ণ Flow

Base Backup (pg_basebackup)  ──►  Archive Mode Enable
        │                              │
        │                              ▼
        │                    WAL ফাইল ক্রমাগত Archive হচ্ছে
        │                              │
        ▼                              ▼
    ধরো দুপুর ১টায়            WAL Archive এ সব
    ভুল হলো (Table Drop)      পরিবর্তনের Record আছে
        │                              │
        └──────────────┬───────────────┘
                        ▼
        Recovery Target Time সেট করে PITR চালানো
        (Table Drop এর ঠিক আগ পর্যন্ত)
```

### PITR Restore করার ধাপ

```bash
# ধাপ ১: PostgreSQL বন্ধ করা
systemctl stop postgresql

# ধাপ ২: পুরনো Data Directory সরানো
mv /var/lib/postgresql/14/main /var/lib/postgresql/14/main_old

# ধাপ ৩: Base Backup Restore করা
cp -R /backup/base/* /var/lib/postgresql/14/main/

# ধাপ ৪: recovery.signal ফাইল তৈরি করা (PostgreSQL 12+)
touch /var/lib/postgresql/14/main/recovery.signal

# ধাপ ৫: postgresql.conf এ Recovery Configuration যোগ করা
cat >> /var/lib/postgresql/14/main/postgresql.conf << EOF
restore_command = 'cp /archive/wal/%f %p'
recovery_target_time = '2026-07-05 12:59:59'
recovery_target_action = 'promote'
EOF

# ধাপ ৬: Permission ঠিক করা এবং PostgreSQL চালু করা
chown -R postgres:postgres /var/lib/postgresql/14/main
systemctl start postgresql

# PostgreSQL স্বয়ংক্রিয়ভাবে recovery_target_time পর্যন্ত
# WAL Apply করবে, তারপর নিজেই Normal Operation এ চলে আসবে
```

### recovery_target এর বিভিন্ন Option

| Option | ব্যাখ্যা |
|---|---|
| `recovery_target_time` | নির্দিষ্ট Timestamp পর্যন্ত Restore |
| `recovery_target_xid` | নির্দিষ্ট Transaction ID পর্যন্ত Restore |
| `recovery_target_lsn` | নির্দিষ্ট WAL LSN (Log Sequence Number) পর্যন্ত |
| `recovery_target_name` | একটা পূর্বে তৈরি করা Named Restore Point পর্যন্ত |

### Named Restore Point তৈরি করা (উন্নত Technique)

```sql
-- একটা গুরুত্বপূর্ণ মুহূর্তে Named Restore Point তৈরি করা
-- (যেমন বড় একটা Deployment এর আগে)
SELECT pg_create_restore_point('before_major_deployment');
```

এই পদ্ধতিতে ভবিষ্যতে ঠিক Timestamp মনে রাখার দরকার নেই, শুধু এই নামটা ব্যবহার করে সেই মুহূর্তে ফিরে যাওয়া যাবে।

---

## 6.8 Recovery Process (রিকভারি প্রসেস)

### PostgreSQL Recovery এর Internal States

```
Recovery শুরু (Standby Mode বা PITR)
        │
        ▼
    ┌───────────────────┐
    │  Redo শুরু হয়        │  ◄── WAL থেকে সব Change Apply হয়
    │  (Base Backup থেকে   │       Recovery Target পর্যন্ত
    │   Target পর্যন্ত)      │
    └─────────┬───────────┘
              │
              ▼
    ┌───────────────────┐
    │  Recovery Target     │
    │  এ পৌঁছালে             │
    └─────────┬───────────┘
              │
    ┌─────────┴──────────┐
    │                       │
    ▼                       ▼
recovery_target_action    Default: Server Promote
= 'pause'                 হয়ে Normal Read-Write
(Manual Review করার        Mode এ চলে আসে
সুযোগ দেয়)
```

### Recovery Verify করা

```sql
-- Recovery চলছে কিনা চেক করা
SELECT pg_is_in_recovery();

-- বর্তমান Recovery Progress দেখা (Log ফাইলেও পাওয়া যায়)
SELECT pg_last_wal_replay_lsn();
```

---

## 6.9 Restore Process (সম্পূর্ণ)

### Logical Backup থেকে Restore

```bash
# Plain SQL
psql -U postgres -d mydb -f backup.sql

# Custom Format (একটা নতুন Database তৈরি করে তারপর Restore)
createdb -U postgres new_mydb
pg_restore -U postgres -d new_mydb backup.dump

# শুধু নির্দিষ্ট Table Restore
pg_restore -U postgres -d mydb -t orders backup.dump
```

### Physical Backup থেকে Restore (pg_basebackup)

```bash
systemctl stop postgresql
rm -rf /var/lib/postgresql/14/main/*
cp -R /backup/base/* /var/lib/postgresql/14/main/
chown -R postgres:postgres /var/lib/postgresql/14/main
systemctl start postgresql
```

### Restore Verification Checklist

```
□ pg_restore/psql Command Error ছাড়া শেষ হয়েছে কিনা
□ SELECT COUNT(*) দিয়ে গুরুত্বপূর্ণ Table এর Row Count Verify করা
□ Sequence (Auto-increment) Value ঠিক আছে কিনা চেক করা
□ Foreign Key Constraint সব ঠিক আছে কিনা
□ Application Connect করে Basic Function Test করা
```

---

## 6.10 Automation

### Bash Script: সম্পূর্ণ PostgreSQL Backup Automation

```bash
#!/bin/bash
# /opt/scripts/postgres_backup.sh

DB_NAME="mydb"
DB_USER="backup_user"
BACKUP_DIR="/backup/postgres"
DATE=$(date +%Y-%m-%d_%H-%M-%S)
RETENTION_DAYS=30
LOG_FILE="/var/log/postgres_backup.log"

mkdir -p "$BACKUP_DIR"

echo "[$(date)] Backup শুরু হচ্ছে..." >> "$LOG_FILE"

pg_dump -U "$DB_USER" -F c -d "$DB_NAME" \
    -f "$BACKUP_DIR/backup_$DATE.dump"

if [ $? -eq 0 ]; then
    echo "[$(date)] Backup সফল: backup_$DATE.dump" >> "$LOG_FILE"
else
    echo "[$(date)] ERROR: Backup ব্যর্থ হয়েছে!" >> "$LOG_FILE"
    exit 1
fi

# পুরনো Backup Cleanup
find "$BACKUP_DIR" -name "*.dump" -mtime +$RETENTION_DAYS -delete

echo "[$(date)] Backup Process সম্পন্ন।" >> "$LOG_FILE"
```

### Cron Job

```bash
# প্রতিদিন রাত ৩টায়
0 3 * * * /opt/scripts/postgres_backup.sh
```

### WAL Archive Cleanup Automation (pg_archivecleanup)

```bash
# পুরনো, আর প্রয়োজন নেই এমন WAL ফাইল Cleanup করা
pg_archivecleanup /archive/wal/ $(pg_controldata /var/lib/postgresql/14/main | grep "REDO WAL file" | awk '{print $NF}')
```

---

## 6.11 Production Example

### বাস্তব একটা SaaS Company এর PostgreSQL Backup Architecture

```
┌───────────────────────────────────────────────────────┐
│                  Production Environment                    │
│                                                             │
│  ┌───────────────┐  Streaming   ┌──────────────────┐      │
│  │  Primary          │  Replication │  Standby (Backup     │      │
│  │  PostgreSQL        │──────────────►│  + Failover Ready)   │      │
│  └───────┬───────┘              └──────────────────┘      │
│          │                                                 │
│    Archive Mode ON (WAL প্রতি ১৬MB Segment এ Archive হয়)     │
│          │                                                 │
└──────────┼─────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────┐
│  WAL Archive Storage            │  ◄── PITR এর জন্য
│  (S3/NFS, প্রতিদিন pg_basebackup   │
│   + ক্রমাগত WAL Stream)            │
└─────────────────────────────┘
           │
   দৈনিক pg_dump (Logical Backup, Selective Restore এর জন্য)
           │
           ▼
┌─────────────────────────────┐
│  Cloud Storage (Encrypted)       │
│  30 দিন Retention + Glacier       │
└─────────────────────────────┘

মাসিক Restore Drill: Full PITR Test করা হয় একটা Staging Server এ
```

---

## 6.12 Best Practices

1. Production এ সবসময় `archive_mode = on` রাখো, PITR এর সুযোগ হারিও না।
2. `pg_basebackup` কে Standby Server থেকে চালাও, Primary এর Load কমাতে।
3. WAL Archive Storage নিয়মিত Monitor করো — Disk Full হয়ে গেলে Archive Fail হবে এবং WAL জমতে জমতে Primary Server এর Disk ও ভরে যেতে পারে।
4. `pg_dump` Custom Format (`-F c`) ব্যবহার করো, কারণ এটা Compressed এবং Selective Restore সমর্থন করে।
5. Named Restore Point তৈরি করো বড় Deployment এর আগে।

---

## 6.13 Common Mistakes

| ভুল | সমস্যা |
|---|---|
| archive_command ভুল লেখা বা Test না করা | WAL Archive হয় না, PITR সম্ভব হয় না |
| WAL Archive Storage Full হয়ে যাওয়া | Primary Server এ pg_wal Directory ভরে গিয়ে Database Down হতে পারে |
| Recovery Target Time ভুল Timezone এ দেওয়া | ভুল সময় পর্যন্ত Restore হয়ে যায় |
| Standby Server কে সরাসরি Write করার চেষ্টা | Standby সাধারণত Read-only, Error হবে |
| pg_basebackup এর WAL Streaming Flag (`-Xs`) বাদ দেওয়া | Backup Inconsistent হতে পারে |

---

## 6.14 Interview Questions (Module 6)

1. **pg_dump কীভাবে Lock ছাড়াই Consistent Backup নেয়?**
   *উত্তর:* PostgreSQL এর MVCC (Multi-Version Concurrency Control) ব্যবহার করে — Dump শুরু হওয়ার মুহূর্তে একটা Snapshot নেওয়া হয়, এবং পুরনো Row Version গুলো MVCC এর কারণে তখনো উপলব্ধ থাকে বলে অন্য Session এর নতুন পরিবর্তন Dump কে প্রভাবিত করে না।

2. **pg_dump এবং pg_dumpall এর মধ্যে পার্থক্য কী?**
   *উত্তর:* pg_dump একটা নির্দিষ্ট Database Backup নেয় বিভিন্ন Format এ (Plain/Custom/Directory), আর pg_dumpall পুরো Server এর সব Database + Role/User Information একসাথে Plain SQL Format এ Backup নেয়।

3. **Archive Mode কী এবং এটা কেন প্রয়োজন?**
   *উত্তর:* Archive Mode Enable করলে PostgreSQL প্রতিটা পূর্ণ হওয়া WAL Segment একটা নির্দিষ্ট Location এ Copy করে রাখে, যা পরবর্তীতে PITR (Point-in-Time Recovery) করার জন্য অপরিহার্য।

4. **pg_basebackup এর `-Xs` Flag এর কাজ কী?**
   *উত্তর:* এটা Backup চলাকালীন তৈরি হওয়া WAL Record একইসাথে Stream করে সংগ্রহ করে, যাতে Backup শেষে সব Data File একটা Consistent, নির্দিষ্ট মুহূর্তে নিয়ে আসা যায়।

5. **Named Restore Point কী এবং এটা কীভাবে সাহায্য করে?**
   *উত্তর:* `pg_create_restore_point()` দিয়ে তৈরি করা একটা নামযুক্ত Marker, যা ব্যবহার করে ভবিষ্যতে সঠিক Timestamp মনে না রেখেও সেই নির্দিষ্ট মুহূর্তে PITR করা যায় — বিশেষ করে বড় Deployment এর আগে ব্যবহার করা হয়।

---

## 6.15 Summary (সারাংশ)

- **pg_dump** নির্দিষ্ট Database এর Logical Backup নেয়, MVCC এর কারণে স্বয়ংক্রিয়ভাবে Consistent।
- **pg_dumpall** পুরো Server (সব Database + Role) Backup নেয়।
- **pg_basebackup** Physical Hot Backup নেয়, `-Xs` Flag দিয়ে WAL Streaming সহ।
- **Archive Mode** WAL Segment সংরক্ষণ করে PITR সক্ষম করে।
- **Streaming Replication** Real-time এ Standby Server আপডেট রাখে, Backup Load কমাতে ব্যবহৃত হয়।
- **PITR** Base Backup + Archived WAL মিলিয়ে যেকোনো নির্দিষ্ট মুহূর্তে Restore করার ক্ষমতা দেয়।
- Named Restore Point বড় Deployment এর আগে একটা "Safe Point" তৈরি করে রাখতে সাহায্য করে।
- Production এ Standby Server থেকে Backup, Archive Mode Enable, এবং Monthly Restore Drill — এই তিনটাই অপরিহার্য।

---

## 6.16 Exercises & Assignments

### Exercise 1
তোমার Local PostgreSQL এ Archive Mode Enable করো এবং একটা Full pg_basebackup নিয়ে দেখো WAL ফাইল কীভাবে Archive Directory তে জমা হচ্ছে।

### Exercise 2
একটা Named Restore Point তৈরি করো (`pg_create_restore_point`), তারপর কিছু ডেটা পরিবর্তন করে সেই Point পর্যন্ত PITR করার চেষ্টা করো।

### Mini Project
Section 6.11 এর Production Architecture নিজের প্রজেক্টের জন্য Customize করে লিখে ফেলো — কোথায় Archive রাখবে, কত ঘন ঘন Base Backup নেবে, RPO/RTO কত রাখবে (Module 4 থেকে সংযুক্ত করে)।

---

**পরবর্তী চ্যাপ্টার:** `07-SQLite-Backup.md` — এখানে আমরা SQLite এর Backup Technique (File Copy, VACUUM INTO, Backup API) নিয়ে শিখবো।
