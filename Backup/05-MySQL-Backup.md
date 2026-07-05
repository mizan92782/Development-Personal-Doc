# Module 5: MySQL Backup (মাইএসকিউএল ব্যাকআপ)

> **Level:** Intermediate → Advanced (Production-Level)
> **Prerequisite:** Module 1-4 সম্পন্ন থাকতে হবে (বিশেষ করে Storage Engine, WAL/Binlog, ACID, PITR ধারণা)।

---

## 5.0 Chapter Roadmap

```
5.1  mysqldump — Logical Backup Tool
5.2  mysqlpump — Parallel Logical Backup
5.3  Binary Logs (Binlog) বিস্তারিত
5.4  GTID (Global Transaction Identifier)
5.5  Replication-based Backup
5.6  XtraBackup — Physical Hot Backup
5.7  MySQL Clone Plugin
5.8  Automation (Cron Job দিয়ে)
5.9  Compression
5.10 Encryption
5.11 Restore Process (সব পদ্ধতি একসাথে)
5.12 Production Example: সম্পূর্ণ Backup Architecture
5.13 Best Practices
5.14 Common Mistakes
5.15 Interview Questions
5.16 Summary
5.17 Exercises & Assignments
```

---

## 5.1 mysqldump — Logical Backup Tool

### সংজ্ঞা

`mysqldump` হলো MySQL এর সবচেয়ে জনপ্রিয় এবং Built-in **Logical Backup Tool**, যা Database কে SQL Statement আকারে Export করে (Module 3-এ শেখা Logical Backup এর বাস্তব প্রয়োগ)।

### মূল ব্যবহার (Basic Usage)

```bash
# একটা নির্দিষ্ট Database Backup নেওয়া
mysqldump -u root -p mydb > mydb_backup.sql

# একাধিক Database একসাথে
mysqldump -u root -p --databases db1 db2 db3 > multi_backup.sql

# সব Database (System Database সহ)
mysqldump -u root -p --all-databases > all_backup.sql

# শুধু Schema (Structure), ডেটা ছাড়া
mysqldump -u root -p --no-data mydb > schema_only.sql

# শুধু ডেটা, Schema ছাড়া
mysqldump -u root -p --no-create-info mydb > data_only.sql

# একটা নির্দিষ্ট Table
mysqldump -u root -p mydb users > users_table_backup.sql
```

### গুরুত্বপূর্ণ Flag: `--single-transaction` (Hot Backup এর জন্য অপরিহার্য)

```bash
mysqldump -u root -p --single-transaction --routines --triggers \
    --events mydb > consistent_backup.sql
```

**কেন এই Flag গুরুত্বপূর্ণ:** Module 2-এ শেখা Transaction ও Isolation ধারণা মনে করো — `--single-transaction` Flag ব্যবহার করলে `mysqldump` একটা একক Transaction শুরু করে, যা InnoDB এর **Repeatable Read Isolation Level** ব্যবহার করে একটা **Consistent Snapshot** নেয়। ফলে Backup চলাকালীন যদি অন্য কেউ ডেটা পরিবর্তন করে, সেই পরিবর্তন Backup-এ প্রভাব ফেলবে না — Backup শুরুর মুহূর্তের ডেটাই Consistently Export হবে, কোনো Table Lock ছাড়াই।

```
Diagram: --single-transaction কীভাবে কাজ করে

mysqldump শুরু
      │
      ▼
BEGIN TRANSACTION (Repeatable Read)
      │
      ▼
এই মুহূর্তের একটা Consistent View তৈরি হলো (Snapshot)
      │
      ▼
একে একে সব Table Export হচ্ছে ── (এই সময়ে অন্য Client নতুন
      │                            ডেটা Insert করলেও, সেটা এই
      │                            Snapshot-এ দেখা যাবে না)
      ▼
COMMIT (Transaction শেষ)
      │
      ▼
Backup সম্পূর্ণ — পুরোপুরি Consistent
```

### গুরুত্বপূর্ণ Flag সমূহের Table

| Flag | ব্যাখ্যা |
|---|---|
| `--single-transaction` | InnoDB Table এর জন্য Lock ছাড়া Consistent Backup |
| `--routines` | Stored Procedure/Function ও Include করে |
| `--triggers` | Trigger ও Include করে (Default এ Enable থাকে) |
| `--events` | Scheduled Event ও Include করে |
| `--master-data=2` | Backup নেওয়ার মুহূর্তের Binlog Position Comment আকারে যোগ করে (PITR এর জন্য গুরুত্বপূর্ণ) |
| `--flush-logs` | Backup নেওয়ার আগে নতুন Binlog ফাইলে Switch করে |
| `--lock-tables=false` | MyISAM Table এর জন্য Lock এড়ানো (তবে Consistency ঝুঁকিপূর্ণ হতে পারে) |

### সুবিধা ও অসুবিধা

| সুবিধা | অসুবিধা |
|---|---|
| সহজ, Built-in, সব জায়গায় পাওয়া যায় | Single-threaded — বড় Database এ ধীর |
| Human-readable Output | Restore এর সময় Index পুনরায় Build করতে হয়, সময়সাপেক্ষ |
| Selective Backup সহজ (নির্দিষ্ট Table/DB) | Memory Usage বেশি হতে পারে বড় Table এ |

### কখন ব্যবহার করবে

- ছোট থেকে মাঝারি Database (কয়েক GB পর্যন্ত)।
- Development/Staging Environment।
- Migration/Version Upgrade এর সময়।

---

## 5.2 mysqlpump — Parallel Logical Backup

### সংজ্ঞা

`mysqlpump` হলো MySQL 5.7+ থেকে আসা একটা নতুন Logical Backup Tool, যা `mysqldump` এর মতোই কাজ করে কিন্তু **Parallel Processing** (একাধিক Thread ব্যবহার করে একসাথে একাধিক Database/Table Backup) সমর্থন করে, ফলে দ্রুততর হয়।

### মূল ব্যবহার

```bash
# Parallel Backup, ৪টা Thread ব্যবহার করে
mysqlpump -u root -p --default-parallelism=4 \
    --databases mydb > mydb_backup.sql

# নির্দিষ্ট Database Exclude করা
mysqlpump -u root -p --exclude-databases=test,mysql \
    > backup_without_system_db.sql
```

### mysqldump vs mysqlpump — পার্থক্য

| বৈশিষ্ট্য | mysqldump | mysqlpump |
|---|---|---|
| Processing | Single-threaded | Multi-threaded (Parallel) |
| গতি (বড় DB তে) | ধীর | দ্রুততর |
| Progress Indicator | নেই | আছে (কত% হয়েছে দেখায়) |
| User Account Backup | না | হ্যাঁ (User/Privilege সহ Export করতে পারে) |
| Compatibility | সব জায়গায় সমর্থিত, বেশি পুরনো Tool | তুলনামূলক নতুন |
| Community ব্যবহার | সবচেয়ে বেশি ব্যবহৃত (Standard) | কম জনপ্রিয়, তবে বড় DB তে ভালো |

### একটা গুরুত্বপূর্ণ সীমাবদ্ধতা

`mysqlpump` এ `--single-transaction` এর সমতুল্য Behavior থাকলেও, এটা `mysqldump` এর মতো ততটা Battle-tested (দীর্ঘদিন ধরে ব্যবহৃত ও পরীক্ষিত) না, তাই অনেক Production System এখনো `mysqldump` কেই বেশি বিশ্বাস করে, বিশেষ করে Consistency এর ক্ষেত্রে।

---

## 5.3 Binary Logs (Binlog) বিস্তারিত

Module 2-এ Binlog এর প্রাথমিক ধারণা শেখানো হয়েছিল। এখানে আমরা Backup এর প্রেক্ষাপটে এটা Practically কীভাবে ব্যবহার করতে হয় শিখবো।

### Binlog Enable করা

```ini
# my.cnf বা my.ini ফাইলে
[mysqld]
log_bin = /var/log/mysql/mysql-bin
binlog_format = ROW
expire_logs_days = 7
max_binlog_size = 100M
server_id = 1
```

### Binlog Management Commands

```bash
# Binlog Enable আছে কিনা চেক করা
SHOW VARIABLES LIKE 'log_bin';

# সব Binlog ফাইলের তালিকা
SHOW BINARY LOGS;

# বর্তমান Binlog Position
SHOW MASTER STATUS;

# একটা নির্দিষ্ট Binlog ফাইলের Content পড়া
mysqlbinlog /var/log/mysql/mysql-bin.000045

# পুরনো Binlog Purge (Delete) করা
PURGE BINARY LOGS BEFORE '2026-06-01 00:00:00';
```

### Binlog দিয়ে PITR (Module 4 এর ব্যবহারিক প্রয়োগ)

```bash
# ধাপ ১: সর্বশেষ Full Backup Restore করা
mysql -u root -p mydb < full_backup.sql

# ধাপ ২: Full Backup নেওয়ার সময়ের Binlog Position জানা
# (mysqldump --master-data=2 দিয়ে নিলে backup.sql ফাইলের ভেতরেই এই তথ্য থাকে)
head -30 full_backup.sql | grep "CHANGE MASTER"

# ধাপ ৩: সেই Position থেকে নির্দিষ্ট সময় পর্যন্ত Binlog Apply করা
mysqlbinlog --start-position=1547 \
    --stop-datetime="2026-07-05 12:59:59" \
    /var/log/mysql/mysql-bin.000045 | mysql -u root -p mydb
```

### Diagram: Binlog Retention Strategy

```
প্রতিদিন Full Backup (রাত ২টা)
        │
        ▼
প্রতিদিনের Binlog ফাইলগুলো Archive Server এ Copy হয়
        │
        ▼
কমপক্ষে শেষ ২টা Full Backup Cycle এর Binlog রাখা হয়
(যদি একটা Full Backup Corrupt হয়, আগেরটা দিয়েও PITR করা যায়)

Full(Day1) ── Binlog(Day1) ── Full(Day2) ── Binlog(Day2) ── Full(Day3)
     │             │              │             │
     └─── রাখা ─────┴─── রাখা ─────┘             │
                                                  └── এখনো তৈরি হচ্ছে
```

---

## 5.4 GTID (Global Transaction Identifier)

### সংজ্ঞা

**GTID** হলো প্রতিটা Transaction-কে একটা **Unique, Global Identifier** দেওয়ার একটা পদ্ধতি, যা MySQL 5.6+ থেকে চালু হয়েছে। এটা প্রতিটা Transaction-কে Server ও Binlog Position নির্বিশেষে সনাক্ত করতে সাহায্য করে।

### GTID এর Format

```
GTID = source_id:transaction_id

উদাহরণ:
3E11FA47-71CA-11E1-9E33-C80AA9429562:23

এখানে:
- 3E11FA47-71CA-11E1-9E33-C80AA9429562  → Server এর Unique UUID
- 23                                     → সেই Server এ ২৩তম Transaction
```

### GTID ছাড়া বনাম GTID সহ (কেন এটা দরকার)

**Traditional Binlog Position ভিত্তিক পদ্ধতির সমস্যা:** পুরনো পদ্ধতিতে (Binlog Filename + Position) Replica Server Track করতো ঠিক কোন ফাইলের কোন Position পর্যন্ত ডেটা Apply হয়েছে। কিন্তু Failover এর সময় (Primary Server পাল্টে গেলে) এই Position Number গুলো এলোমেলো হয়ে যেতো, ফলে সঠিক Point থেকে Replication চালিয়ে যাওয়া কঠিন হতো।

**GTID এর সমাধান:** প্রতিটা Transaction এর একটা Global Unique ID থাকায়, যেকোনো Server থেকে Replication শুরু/চালিয়ে যাওয়া সহজ হয় — কারণ MySQL নিজেই বুঝতে পারে কোন GTID পর্যন্ত Apply হয়েছে এবং কোনটা বাকি আছে।

### GTID Enable করা

```ini
# my.cnf ফাইলে
[mysqld]
gtid_mode = ON
enforce_gtid_consistency = ON
log_bin = mysql-bin
log_slave_updates = ON
```

### GTID দিয়ে PITR করার সুবিধা

```bash
# GTID ব্যবহার করে নির্দিষ্ট Transaction পর্যন্ত Restore করা
mysqlbinlog --include-gtids='3E11FA47-71CA-11E1-9E33-C80AA9429562:1-50' \
    /var/log/mysql/mysql-bin.000045 | mysql -u root -p mydb

# একটা নির্দিষ্ট Transaction (যেমন ভুল করে চালানো একটা) Exclude করা
mysqlbinlog --exclude-gtids='3E11FA47-71CA-11E1-9E33-C80AA9429562:51' \
    /var/log/mysql/mysql-bin.000045 | mysql -u root -p mydb
```

এই শেষ Command-টা খুবই শক্তিশালী — যদি তুমি জানো ঠিক কোন GTID (কোন Transaction) ভুল ছিল (যেমন ভুল DELETE), তাহলে সেই একটামাত্র Transaction Skip করে বাকি সব ঠিকভাবে Apply করতে পারবে, PITR এর মতো পুরো সময় ধরে Cut করার দরকার নেই।

---

## 5.5 Replication-based Backup

### সংজ্ঞা

Replication-based Backup মানে হলো একটা **Replica (Slave) Server** থেকে Backup নেওয়া, Primary (Master) Server থেকে না — যাতে Primary Server এর উপর কোনো Load/Performance Impact না পড়ে।

### Architecture Diagram

```
┌─────────────────┐         ┌─────────────────┐
│  Primary (Master)  │  Replication │  Replica (Slave)   │
│  (Live Traffic       │─────────────►│  (শুধু Backup       │
│   Handle করছে)       │              │   নেওয়ার জন্য)       │
└─────────────────┘         └────────┬────────┘
                                       │
                                       ▼
                              Backup Tool চলছে এখানে
                              (mysqldump/XtraBackup)
                              Primary Server এ কোনো
                              প্রভাব পড়ছে না
```

### Replica সেটআপ করা (সংক্ষিপ্ত)

```sql
-- Primary Server এ
CREATE USER 'repl_user'@'%' IDENTIFIED BY 'strong_password';
GRANT REPLICATION SLAVE ON *.* TO 'repl_user'@'%';

-- Replica Server এ
CHANGE MASTER TO
    MASTER_HOST='primary_server_ip',
    MASTER_USER='repl_user',
    MASTER_PASSWORD='strong_password',
    MASTER_AUTO_POSITION=1;  -- GTID ব্যবহার করে

START SLAVE;
```

### Replica থেকে Backup নেওয়া

```bash
# Replica তে Replication সাময়িক থামিয়ে Consistent Backup নেওয়া
mysql -u root -p -e "STOP SLAVE SQL_THREAD;"
mysqldump -u root -p --single-transaction --all-databases > backup.sql
mysql -u root -p -e "START SLAVE SQL_THREAD;"
```

### সুবিধা

| সুবিধা | ব্যাখ্যা |
|---|---|
| Primary Server এ কোনো Load পড়ে না | Backup এর কারণে Production Performance কমে না |
| Backup চলাকালীন সময় বেশি নিলেও সমস্যা নেই | Replica Backup এর জন্য Dedicated |
| Delayed Replica ব্যবহার করলে Human Error থেকেও সুরক্ষা | কিছু সেকেন্ড/মিনিট পিছিয়ে থাকা Replica রাখলে ভুল Command Replica তে পৌঁছানোর আগেই থামানো যায় |

### অসুবিধা

- Replica Server এর জন্য অতিরিক্ত Infrastructure Cost লাগে।
- Replication Lag (Primary ও Replica এর মধ্যে সময়ের ব্যবধান) থাকলে Backup সামান্য পুরনো হতে পারে।

---

## 5.6 XtraBackup — Physical Hot Backup

### সংজ্ঞা

**Percona XtraBackup** হলো একটা Open-source, Free Tool যা MySQL/InnoDB এর জন্য **Physical Hot Backup** নেয় — অর্থাৎ Database চলমান অবস্থায় সরাসরি Data File Copy করে, কোনো Downtime বা Locking ছাড়াই (Module 3-এ শেখা Physical + Hot Backup এর বাস্তব প্রয়োগ)।

### কীভাবে এটা Consistency নিশ্চিত করে (Internal Working)

```
XtraBackup শুরু
      │
      ▼
InnoDB Data File গুলো সরাসরি Copy করা শুরু হয় (Byte-level)
      │
      ▼
Copy চলাকালীন সময়েও নতুন Transaction চলতে থাকে
      │
      ▼
XtraBackup একই সাথে InnoDB এর Redo Log ও Track করে
      │
      ▼
Copy শেষ হলে, Backup এর মধ্যে থাকা Redo Log Apply করে
Backup কে একটা Consistent State এ আনা হয় ("Prepare" ধাপ)
      │
      ▼
সম্পূর্ণ, Consistent, Restore-ready Backup তৈরি
```

### বাস্তব ব্যবহার

```bash
# ধাপ ১: Full Backup নেওয়া
xtrabackup --backup --target-dir=/backup/full/ \
    --user=root --password=yourpassword

# ধাপ ২: Backup কে "Prepare" করা (Consistency নিশ্চিত করা)
xtrabackup --prepare --target-dir=/backup/full/

# ধাপ ৩: Incremental Backup নেওয়া (Full এর পর থেকে পরিবর্তন)
xtrabackup --backup --target-dir=/backup/inc1/ \
    --incremental-basedir=/backup/full/ \
    --user=root --password=yourpassword

# ধাপ ৪: Restore এর জন্য Full + Incremental একসাথে Prepare করা
xtrabackup --prepare --apply-log-only --target-dir=/backup/full/
xtrabackup --prepare --target-dir=/backup/full/ \
    --incremental-dir=/backup/inc1/

# ধাপ ৫: Restore করা
xtrabackup --copy-back --target-dir=/backup/full/ \
    --datadir=/var/lib/mysql/
```

### mysqldump vs XtraBackup — কখন কোনটা

| বিষয় | mysqldump | XtraBackup |
|---|---|---|
| Type | Logical | Physical |
| গতি (বড় DB) | ধীর | দ্রুত |
| Downtime | নেই (single-transaction দিয়ে) | নেই |
| Database Size উপযোগিতা | ছোট-মাঝারি (কয়েক GB) | বড়-বিশাল (কয়েক GB - TB) |
| Incremental Support | না (সরাসরি) | হ্যাঁ (Native Support) |
| Restore Speed | ধীর (Index পুনরায় Build) | দ্রুত (সরাসরি Copy) |
| Portability | বেশি | কম (একই MySQL Version এ ভালো কাজ করে) |

### Company Usage

XtraBackup Facebook, Booking.com, এবং অসংখ্য বড় Tech Company তাদের বড় MySQL Cluster Backup নিতে ব্যবহার করে, কারণ এটা Free, Open-source, এবং TB Scale Database এ চমৎকার Performance দেয়।

---

## 5.7 MySQL Clone Plugin

### সংজ্ঞা

MySQL 8.0.17+ থেকে আসা **Clone Plugin** একটা Built-in Feature যা এক MySQL Server থেকে আরেক MySQL Server-এ সরাসরি (Network এর মাধ্যমে) সম্পূর্ণ Data Clone করার সুবিধা দেয় — এক্সটার্নাল Tool ছাড়াই।

### Setup ও ব্যবহার

```sql
-- উভয় Server এ Plugin Install করা
INSTALL PLUGIN clone SONAME 'mysql_clone.so';

-- Donor (Source) Server এ প্রয়োজনীয় Privilege
GRANT BACKUP_ADMIN ON *.* TO 'clone_user'@'%';

-- Recipient (Target) Server এ Clone চালানো
SET GLOBAL clone_valid_donor_list = 'donor_host:3306';

CLONE INSTANCE FROM 'clone_user'@'donor_host':3306
IDENTIFIED BY 'password';
```

### কখন ব্যবহার করবে

- নতুন Replica Server দ্রুত সেটআপ করতে (Manual Backup+Restore এর ঝামেলা ছাড়াই)।
- Development/Testing Environment এ Production এর মতো ডেটা আনতে।

### সীমাবদ্ধতা

- শুধুমাত্র MySQL 8.0.17+ এর মধ্যে কাজ করে (একই বা সামঞ্জস্যপূর্ণ Version দরকার)।
- Point-in-time Backup Archive হিসেবে ব্যবহারের জন্য উপযুক্ত না (এটা মূলত তাৎক্ষণিক Data Copy এর জন্য)।

---

## 5.8 Automation (Cron Job দিয়ে)

### কেন Automation অপরিহার্য

Module 1-এ শেখা হয়েছিল Manual Backup এর উপর নির্ভর করা একটা Common Mistake, কারণ মানুষ ভুলে যায়। তাই Production এ Backup সবসময় **Automated** হতে হবে।

### সম্পূর্ণ Backup Script উদাহরণ

```bash
#!/bin/bash
# /opt/scripts/mysql_backup.sh

# ===== Configuration =====
DB_USER="backup_user"
DB_PASS="strong_password_here"
BACKUP_DIR="/backup/mysql"
DATE=$(date +%Y-%m-%d_%H-%M-%S)
RETENTION_DAYS=30
LOG_FILE="/var/log/mysql_backup.log"

# ===== Backup Directory তৈরি করা =====
mkdir -p "$BACKUP_DIR"

# ===== Backup নেওয়া =====
echo "[$(date)] Backup শুরু হচ্ছে..." >> "$LOG_FILE"

mysqldump -u "$DB_USER" -p"$DB_PASS" \
    --single-transaction --routines --triggers --events \
    --all-databases | gzip > "$BACKUP_DIR/full_backup_$DATE.sql.gz"

# ===== Success/Failure যাচাই =====
if [ $? -eq 0 ]; then
    echo "[$(date)] Backup সফল হয়েছে: full_backup_$DATE.sql.gz" >> "$LOG_FILE"
else
    echo "[$(date)] ERROR: Backup ব্যর্থ হয়েছে!" >> "$LOG_FILE"
    # এখানে Alert পাঠানোর Code যোগ করা যায় (Email/Slack)
    exit 1
fi

# ===== পুরনো Backup Delete করা (Retention Policy) =====
find "$BACKUP_DIR" -name "*.sql.gz" -mtime +$RETENTION_DAYS -delete

echo "[$(date)] Backup Process সম্পন্ন হলো।" >> "$LOG_FILE"
```

### Cron Job সেটআপ করা

```bash
# Crontab এ যোগ করা (crontab -e দিয়ে)
chmod +x /opt/scripts/mysql_backup.sh

# প্রতিদিন রাত ২টায় চালানো
0 2 * * * /opt/scripts/mysql_backup.sh

# প্রতি ৬ ঘণ্টা পর পর চালানো
0 */6 * * * /opt/scripts/mysql_backup.sh
```

### Crontab সময়ের ব্যাখ্যা

```
 ┌───────────── মিনিট (0 - 59)
 │ ┌───────────── ঘণ্টা (0 - 23)
 │ │ ┌───────────── দিন (1 - 31)
 │ │ │ ┌───────────── মাস (1 - 12)
 │ │ │ │ ┌───────────── সপ্তাহের দিন (0 - 6) (রবি=0)
 │ │ │ │ │
 0 2 * * *   →  প্রতিদিন রাত ২:০০ টায়
```

### Python দিয়ে Automation (Django Integration সহ)

```python
# backup_script.py
import subprocess
import os
import gzip
import shutil
from datetime import datetime, timedelta

BACKUP_DIR = "/backup/mysql"
DB_USER = "backup_user"
DB_PASS = os.environ.get("MYSQL_BACKUP_PASSWORD")
RETENTION_DAYS = 30

def run_backup():
    timestamp = datetime.now().strftime("%Y-%m-%d_%H-%M-%S")
    backup_file = f"{BACKUP_DIR}/full_backup_{timestamp}.sql"
    gz_file = f"{backup_file}.gz"

    cmd = [
        "mysqldump", "-u", DB_USER, f"-p{DB_PASS}",
        "--single-transaction", "--routines", "--triggers",
        "--all-databases"
    ]

    try:
        with open(backup_file, "w") as f:
            result = subprocess.run(cmd, stdout=f, stderr=subprocess.PIPE, check=True)

        # Compress করা
        with open(backup_file, 'rb') as f_in:
            with gzip.open(gz_file, 'wb') as f_out:
                shutil.copyfileobj(f_in, f_out)
        os.remove(backup_file)

        print(f"Backup সফল: {gz_file}")
        cleanup_old_backups()

    except subprocess.CalledProcessError as e:
        print(f"Backup ব্যর্থ: {e.stderr.decode()}")
        # এখানে Alert System Integrate করা যায়
        raise

def cleanup_old_backups():
    cutoff = datetime.now() - timedelta(days=RETENTION_DAYS)
    for filename in os.listdir(BACKUP_DIR):
        filepath = os.path.join(BACKUP_DIR, filename)
        if os.path.getmtime(filepath) < cutoff.timestamp():
            os.remove(filepath)
            print(f"পুরনো Backup Delete করা হলো: {filename}")

if __name__ == "__main__":
    run_backup()
```

---

## 5.9 Compression (কম্প্রেশন)

### কেন Compression দরকার

Backup ফাইল সাধারণত অনেক বড় হয়, তাই Storage Cost কমাতে এবং Network Transfer দ্রুত করতে Compression ব্যবহার করা হয়।

### বিভিন্ন Compression পদ্ধতি

```bash
# gzip দিয়ে Compress করা (সবচেয়ে জনপ্রিয়)
mysqldump -u root -p mydb | gzip > backup.sql.gz

# gzip দিয়ে Decompress করে Restore করা
gunzip < backup.sql.gz | mysql -u root -p mydb

# zstd দিয়ে (আরও দ্রুত এবং ভালো Compression Ratio)
mysqldump -u root -p mydb | zstd > backup.sql.zst

# Parallel Compression (pigz - Multi-core ব্যবহার করে দ্রুত)
mysqldump -u root -p mydb | pigz > backup.sql.gz
```

### Compression Tool তুলনা

| Tool | গতি | Compression Ratio | Multi-core Support |
|---|---|---|---|
| gzip | মাঝারি | ভালো | না |
| pigz | দ্রুত | ভালো (gzip এর সমান) | হ্যাঁ |
| zstd | খুব দ্রুত | চমৎকার | হ্যাঁ |
| bzip2 | ধীর | খুব ভালো | না |

### Production Tip

খুব বড় Database এর জন্য `zstd` বা `pigz` ব্যবহার করা ভালো, কারণ এগুলো Multi-core CPU ব্যবহার করে দ্রুত Compress করতে পারে, ফলে Backup Window (Backup নেওয়ার সময়কাল) কমে যায়।

---

## 5.10 Encryption (এনক্রিপশন)

### কেন Encryption অপরিহার্য

Backup ফাইলে সাধারণত Sensitive User Data (Password Hash, Personal Information, Payment Info) থাকে। যদি এই Backup ফাইল কোনোভাবে চুরি হয় বা ভুল জায়গায় পৌঁছায়, Encryption না থাকলে যে কেউ সরাসরি ডেটা পড়ে ফেলতে পারবে।

### OpenSSL দিয়ে Encryption

```bash
# Backup নিয়ে সাথে সাথে Encrypt করা
mysqldump -u root -p mydb | gzip | \
    openssl enc -aes-256-cbc -salt -pbkdf2 \
    -out backup_encrypted.sql.gz.enc -k "YourStrongPassphrase"

# Decrypt করে Restore করা
openssl enc -aes-256-cbc -d -pbkdf2 \
    -in backup_encrypted.sql.gz.enc -k "YourStrongPassphrase" | \
    gunzip | mysql -u root -p mydb
```

### GPG দিয়ে Encryption (আরও নিরাপদ, Key-based)

```bash
# Public Key দিয়ে Encrypt করা
mysqldump -u root -p mydb | gzip | \
    gpg --encrypt --recipient admin@company.com > backup.sql.gz.gpg

# Private Key দিয়ে Decrypt করা
gpg --decrypt backup.sql.gz.gpg | gunzip | mysql -u root -p mydb
```

### Encryption Key ম্যানেজমেন্ট Best Practices

| নিয়ম | ব্যাখ্যা |
|---|---|
| Key কখনো Backup ফাইলের সাথে একই জায়গায় রাখবে না | Key চুরি হলে Encryption অর্থহীন হয়ে যায় |
| AWS KMS/HashiCorp Vault এর মতো Key Management System ব্যবহার করো | Manual Key Handling এর ঝুঁকি কমায় |
| Key Rotation নিয়মিত করো | পুরনো Key Compromise হলেও ক্ষতি সীমিত থাকে |

---

## 5.11 Restore Process (সব পদ্ধতি একসাথে)

### Logical Backup (mysqldump) থেকে Restore

```bash
# Compressed Backup থেকে Restore
gunzip < backup.sql.gz | mysql -u root -p

# Encrypted + Compressed Backup থেকে Restore
openssl enc -aes-256-cbc -d -pbkdf2 -in backup.sql.gz.enc \
    -k "YourStrongPassphrase" | gunzip | mysql -u root -p
```

### Physical Backup (XtraBackup) থেকে Restore

```bash
# ধাপ ১: MySQL Server বন্ধ করা
systemctl stop mysql

# ধাপ ২: পুরনো Data Directory সরিয়ে রাখা (Safety এর জন্য)
mv /var/lib/mysql /var/lib/mysql_old

# ধাপ ৩: Backup Copy Back করা
xtrabackup --copy-back --target-dir=/backup/full/ \
    --datadir=/var/lib/mysql/

# ধাপ ৪: Permission ঠিক করা
chown -R mysql:mysql /var/lib/mysql

# ধাপ ৫: MySQL চালু করা
systemctl start mysql
```

### Restore Checklist (Production Level)

```
□ Backup ফাইল Corrupt কিনা যাচাই করা (Checksum verify)
□ পর্যাপ্ত Disk Space আছে কিনা নিশ্চিত করা
□ Restore করার আগে Existing ডেটার একটা Backup নেওয়া (Safety Net)
□ Maintenance Mode Enable করা (Application থেকে)
□ Restore চালানো
□ Restore এর পর Row Count/Sample Data Verify করা
□ Application Connect করে Test করা
□ Maintenance Mode বন্ধ করে Normal Traffic চালু করা
□ পুরো Incident/Restore Process Document করা
```

---

## 5.12 Production Example: সম্পূর্ণ Backup Architecture

### বাস্তব একটা E-commerce Company এর MySQL Backup Architecture

```
┌────────────────────────────────────────────────────────────┐
│                    Production Environment                     │
│                                                                │
│  ┌──────────────┐   Replication   ┌──────────────────┐      │
│  │ Primary MySQL   │────────────────►│ Replica MySQL       │      │
│  │ (Live Traffic)   │                 │ (Backup-only Node)  │      │
│  └──────────────┘                 └─────────┬────────────┘      │
│                                              │                   │
│                          Daily 2AM: XtraBackup Full Backup       │
│                          Every 6hr: Binlog Archive                │
│                                              │                   │
└──────────────────────────────────────────────┼───────────────────┘
                                               │
                          Compress (zstd) + Encrypt (GPG)
                                               │
                                               ▼
                          ┌──────────────────────────────┐
                          │   Local Backup Storage          │
                          │   (7 দিনের জন্য রাখা হয়)          │
                          └──────────────┬────────────────┘
                                          │
                          Upload to Cloud (দৈনিক)
                                          │
                                          ▼
                          ┌──────────────────────────────┐
                          │   AWS S3 (30 দিনের Retention)   │
                          │   + Glacier (1 বছর Archive)     │
                          └──────────────┬────────────────┘
                                          │
                          Cross-Region Replication
                                          │
                                          ▼
                          ┌──────────────────────────────┐
                          │  DR Region S3 Bucket             │
                          │  (Disaster Recovery এর জন্য)      │
                          └──────────────────────────────┘

মাসিক Restore Drill: Test Environment এ Restore করে Verify করা হয়
Monitoring: Backup Fail হলে Slack/PagerDuty এ Alert যায়
```

### এই Architecture এর প্রতিটা সিদ্ধান্তের কারণ

| সিদ্ধান্ত | কারণ |
|---|---|
| Replica থেকে Backup নেওয়া | Primary Server এর Performance এ প্রভাব না ফেলার জন্য |
| XtraBackup ব্যবহার | বড় Database এ দ্রুত Physical Backup এর জন্য |
| Binlog Archive প্রতি ৬ ঘণ্টায় | PITR এর জন্য RPO কম রাখতে |
| Compression + Encryption | Storage Cost কমানো ও Security নিশ্চিত করা |
| Local + Cloud + Cross-Region (3-2-1 Rule) | Module 1 এ শেখা 3-2-1 Rule মেনে চলা |
| Monthly Restore Drill | GitLab এর মতো ঘটনা এড়ানো |

---

## 5.13 Best Practices

1. Production এ সবসময় `--single-transaction` Flag ব্যবহার করো InnoDB Table এর জন্য।
2. বড় Database এ XtraBackup ব্যবহার করো, mysqldump শুধু ছোট DB/Migration এ।
3. Replica Server থেকে Backup নাও, Primary কে বিরক্ত করো না।
4. GTID Enable রাখো — এটা Replication ও PITR উভয়ই সহজ করে।
5. Backup সবসময় Compress এবং Encrypt করো।
6. Automation ব্যবহার করো — Cron/Systemd Timer দিয়ে, Manual Backup কখনো নয়।
7. Retention Policy স্পষ্টভাবে নির্ধারণ করো এবং Automated Cleanup রাখো।

---

## 5.14 Common Mistakes

| ভুল | সমস্যা |
|---|---|
| `--single-transaction` ছাড়া mysqldump চালানো Production এ | Inconsistent Backup হওয়ার ঝুঁকি |
| Primary Server থেকে সরাসরি ভারী Backup নেওয়া | Production Performance এ প্রভাব পড়ে |
| Binlog Purge করে ফেলা PITR এর কথা না ভেবে | PITR Chain ভেঙে যায় |
| Backup Encrypt না করা | Data Breach এর ঝুঁকি |
| XtraBackup এর Prepare ধাপ ভুলে যাওয়া | Restore করার সময় Inconsistent ডেটা পাওয়া যায় |

---

## 5.15 Interview Questions (Module 5)

1. **`--single-transaction` Flag কীভাবে কাজ করে এবং কেন এটা গুরুত্বপূর্ণ?**
   *উত্তর:* এটা InnoDB এর Repeatable Read Isolation Level ব্যবহার করে একটা Consistent Snapshot নেয়, যাতে Backup চলাকালীন অন্য Transaction এর পরিবর্তন Backup-কে প্রভাবিত না করে এবং Table Lock এর দরকার না হয়।

2. **XtraBackup mysqldump এর চেয়ে কেন বড় Database এ ভালো?**
   *উত্তর:* কারণ XtraBackup Physical Backup নেয় (সরাসরি Byte-level File Copy), যা mysqldump এর SQL Generation পদ্ধতির চেয়ে অনেক দ্রুত, বিশেষ করে বড় Database এ Restore করার সময় (Index পুনরায় Build করার দরকার হয় না)।

3. **GTID কীভাবে Traditional Binlog Position ভিত্তিক Replication এর চেয়ে ভালো?**
   *উত্তর:* GTID প্রতিটা Transaction কে একটা Global Unique ID দেয়, যা Server/Failover নির্বিশেষে সহজে Track করা যায়, যেখানে Traditional পদ্ধতিতে Binlog Filename+Position Failover এর পর এলোমেলো হয়ে যেতে পারতো।

4. **কেন Replica Server থেকে Backup নেওয়া ভালো Practice?**
   *উত্তর:* কারণ এতে Primary Server-এ কোনো Extra Load পড়ে না, ফলে Live Traffic এর Performance অপরিবর্তিত থাকে।

5. **XtraBackup এর "Prepare" ধাপ কেন প্রয়োজন?**
   *উত্তর:* Backup নেওয়ার সময় Copy চলাকালীনও নতুন Transaction চলতে থাকে, তাই Backup এর মধ্যে থাকা Redo Log Apply করে একটা সম্পূর্ণ Consistent State তৈরি করতে "Prepare" ধাপ প্রয়োজন হয়, নাহলে Restore করা ডেটা Inconsistent থাকবে।

---

## 5.16 Summary (সারাংশ)

- **mysqldump** সহজ, Logical Backup Tool — ছোট-মাঝারি Database এর জন্য উপযুক্ত, `--single-transaction` দিয়ে Consistency নিশ্চিত করা হয়।
- **mysqlpump** Parallel Processing সমর্থন করে দ্রুততর, তবে কম ব্যবহৃত।
- **Binlog** PITR এর ভিত্তি — প্রতিটা পরিবর্তনের Record রাখে।
- **GTID** Transaction Tracking সহজ করে, Failover-friendly Replication সম্ভব করে।
- **Replication-based Backup** Primary Server এর Load কমায়।
- **XtraBackup** বড় Database এর জন্য সেরা Physical Hot Backup Tool — Full ও Incremental উভয়ই সমর্থন করে।
- **Clone Plugin** দ্রুত Server-to-Server Data Clone করতে সাহায্য করে।
- Backup সবসময় **Automated, Compressed, এবং Encrypted** হওয়া উচিত।
- Production-এ Full Architecture এ Replica Backup + XtraBackup + Binlog Archive + Cloud Storage + Monthly Restore Drill একসাথে ব্যবহার করা হয়।

---

## 5.17 Exercises & Assignments

### Exercise 1
তোমার Local MySQL এ `--single-transaction` Flag দিয়ে এবং ছাড়া দুইটা Backup নাও, তারপর একটা Long-running Transaction চালিয়ে দেখো পার্থক্য বোঝার চেষ্টা করো।

### Exercise 2
XtraBackup Install করো (যদি সম্ভব হয় একটা Test VM/Container এ) এবং একটা Full + Incremental Backup Cycle সম্পূর্ণ করে Restore করে দেখো।

### Mini Project
Section 5.8 এর Bash Script টা নিজের মতো করে Improve করো:
- Slack/Email Notification যোগ করো Backup Fail হলে
- Backup Size Log করো
- Checksum Verification যোগ করো

---

**পরবর্তী চ্যাপ্টার:** `06-PostgreSQL-Backup.md` — এখানে আমরা PostgreSQL এর জন্য pg_dump, pg_dumpall, pg_basebackup, WAL Archiving, Streaming Replication, এবং PITR নিয়ে বিস্তারিতভাবে শিখবো।
