# Module 10: Cloud Database Backup (ক্লাউড ডেটাবেজ ব্যাকআপ)

> **Level:** Intermediate → Advanced
> **Prerequisite:** Module 1-9। এই Module এ আমরা দেখবো কীভাবে Cloud Provider গুলো Backup ব্যবস্থাপনা সহজ করে দেয়, এবং কখন Managed Service এর উপর নির্ভর করা যায়, কখন নিজে Control নেওয়া উচিত।

---

## 10.0 Chapter Roadmap

```
10.1  Managed Database Service এর মূল ধারণা
10.2  AWS RDS
10.3  Google Cloud SQL
10.4  Azure SQL
10.5  Supabase
10.6  PlanetScale
10.7  Neon
10.8  Railway
10.9  Render
10.10 DigitalOcean Managed Databases
10.11 Managed vs Self-hosted — Decision Framework
10.12 Best Practices
10.13 Common Mistakes
10.14 Interview Questions
10.15 Summary
10.16 Exercises & Assignments
```

---

## 10.1 Managed Database Service এর মূল ধারণা

### সংজ্ঞা

**Managed Database Service** মানে হলো Cloud Provider নিজে Database Server Setup, Maintenance, Patching, এবং **Backup** এর দায়িত্ব নেয় — Developer কে শুধু Database ব্যবহার করতে হয়, Infrastructure নিয়ে চিন্তা করতে হয় না।

### Self-hosted vs Managed — মূল পার্থক্য

```
Diagram: Responsibility ভাগাভাগি

Self-hosted (নিজে EC2/VM এ MySQL Install করে চালানো):
┌─────────────────────────────────────┐
│  তোমার দায়িত্ব:                        │
│  - OS Patch                            │
│  - MySQL Install/Update                │
│  - Backup Script লেখা ও চালানো          │
│  - Monitoring Setup করা                │
│  - Replication Configure করা           │
│  - Security Configure করা              │
└─────────────────────────────────────┘

Managed (AWS RDS/Google Cloud SQL):
┌─────────────────────────────────────┐
│  Cloud Provider এর দায়িত্ব:              │
│  - OS Patch                            │
│  - Database Software Update            │
│  - Automatic Backup                    │
│  - Automatic Failover                  │
│                                         │
│  তোমার দায়িত্ব:                        │
│  - Backup Retention Policy সেট করা      │
│  - Restore Testing করা                 │
│  - Application Level Configuration      │
└─────────────────────────────────────┘
```

### গুরুত্বপূর্ণ সতর্কতা: Managed মানে "Backup নিয়ে ভাবতে হবে না" — এটা ভুল ধারণা

এটা একটা Critical ভুল ধারণা যা অনেক Junior Engineer করে থাকে। Managed Service Backup **নেয়** ঠিকই, কিন্তু:
- Retention Period সীমিত হতে পারে (Default Setting এ)।
- Restore Testing এখনো তোমার দায়িত্ব।
- Ransomware/Human Error থেকে সুরক্ষার জন্য Additional Backup Strategy (যেমন Manual Snapshot, Cross-region Copy) এখনো প্রয়োজন হতে পারে।

---

## 10.2 AWS RDS (Relational Database Service)

### সংজ্ঞা

**AWS RDS** হলো Amazon এর Managed Relational Database Service, যা MySQL, PostgreSQL, MariaDB, Oracle, SQL Server সমর্থন করে।

### Automatic Backup Feature

```
RDS Automatic Backup এর মূল বৈশিষ্ট্য:

├── Automated Snapshot   — প্রতিদিন একটা নির্দিষ্ট Backup Window এ নেওয়া হয়
├── Transaction Log Backup — প্রতি ৫ মিনিটে (PITR এর জন্য)
├── Retention Period      — ০ থেকে ৩৫ দিন (Configurable)
└── Manual Snapshot        — যেকোনো সময় Manual ভাবে নেওয়া যায়, কোনো Expiry নেই
```

### RDS এ PITR (Module 4-এ শেখা ধারণার Practical প্রয়োগ)

RDS Automatic Backup Enable থাকলে, Transaction Log প্রতি ৫ মিনিটে Backup হয়, ফলে তুমি শেষ ৫ মিনিটের মধ্যে যেকোনো মুহূর্তে (Retention Period এর মধ্যে) Restore করতে পারবে — এটা মূলত AWS নিজেই Full Backup + WAL/Binlog Management (Module 5, 6 এ যা Manual ভাবে করা শেখানো হয়েছিল) স্বয়ংক্রিয়ভাবে করে দেয়।

```bash
# AWS CLI দিয়ে PITR Restore করা
aws rds restore-db-instance-to-point-in-time \
    --source-db-instance-identifier mydb-instance \
    --target-db-instance-identifier mydb-restored \
    --restore-time 2026-07-05T12:59:59Z
```

### Manual Snapshot নেওয়া ও Cross-Region Copy

```bash
# Manual Snapshot তৈরি করা
aws rds create-db-snapshot \
    --db-instance-identifier mydb-instance \
    --db-snapshot-identifier mydb-manual-snapshot-2026-07-05

# Snapshot অন্য Region এ Copy করা (Disaster Recovery এর জন্য)
aws rds copy-db-snapshot \
    --source-db-snapshot-identifier arn:aws:rds:us-east-1:123456789:snapshot:mydb-manual-snapshot-2026-07-05 \
    --target-db-snapshot-identifier mydb-dr-copy \
    --source-region us-east-1 \
    --region eu-west-1
```

### RDS Backup সম্পর্কিত গুরুত্বপূর্ণ সীমাবদ্ধতা

| বিষয় | সীমাবদ্ধতা |
|---|---|
| Automatic Backup Storage | সাধারণত Database Storage সাইজ পর্যন্ত Free, তার বেশি হলে অতিরিক্ত Cost |
| Restore করলে নতুন Instance তৈরি হয় | সরাসরি Existing Instance এর উপর Overwrite হয় না — নতুন Endpoint আসে |
| Backup Window এ সামান্য Performance Impact | I/O Suspend হতে পারে Multi-AZ না হলে |

---

## 10.3 Google Cloud SQL

### সংজ্ঞা

**Google Cloud SQL** হলো Google Cloud এর Managed Database Service (MySQL, PostgreSQL, SQL Server সমর্থন করে)।

### Automatic Backup ও PITR

```bash
# gcloud CLI দিয়ে Automated Backup Configure করা
gcloud sql instances patch mydb-instance \
    --backup-start-time=02:00 \
    --enable-bin-log \
    --retained-backups-count=30 \
    --retained-transaction-log-days=7

# Manual On-demand Backup
gcloud sql backups create --instance=mydb-instance

# PITR Restore করা (নতুন Instance এ)
gcloud sql instances clone mydb-instance mydb-restored \
    --point-in-time='2026-07-05T12:59:59Z'
```

### `--enable-bin-log` এর গুরুত্ব

Module 5-এ শেখা Binlog ধারণা এখানে সরাসরি প্রাসঙ্গিক — Cloud SQL এ PITR সক্ষম করতে হলে Binary Logging Enable করতে হয় (MySQL এর জন্য), যা Google এর Infrastructure এ Automatically Manage হয়, কিন্তু Enable করাটা তোমাকেই করতে হবে।

### Cross-Region Backup (Disaster Recovery)

Google Cloud SQL এ Automated Backup Default ভাবে Multi-region এ সংরক্ষিত হতে পারে (Configuration Option অনুযায়ী), যা Module 4-এ শেখা DR Site ধারণার একটা Built-in বাস্তবায়ন।

---

## 10.4 Azure SQL

### সংজ্ঞা

**Azure SQL Database** হলো Microsoft Azure এর Managed SQL Server Database Service।

### Automatic Backup বৈশিষ্ট্য

```
Azure SQL এর Backup System:

├── Full Backup       — প্রতি সপ্তাহে
├── Differential Backup — প্রতি ১২ ঘণ্টায় (Module 3-এ শেখা Differential Backup এর বাস্তব প্রয়োগ)
├── Transaction Log Backup — প্রতি ৫-১০ মিনিটে
└── Retention Period    — ৭ থেকে ৩৫ দিন (Point-in-time Restore এর জন্য)
```

লক্ষ্য করো, Azure SQL সরাসরি Module 3-এ শেখা **Full + Differential + Log Backup** Combination ব্যবহার করে — এটা তত্ত্বীয় ধারণা কীভাবে বাস্তব Production Cloud System এ প্রয়োগ হয় তার একটা চমৎকার উদাহরণ।

### PITR করা (Azure Portal/CLI)

```bash
# Azure CLI দিয়ে PITR Restore
az sql db restore \
    --dest-name mydb-restored \
    --name mydb \
    --resource-group myResourceGroup \
    --server myserver \
    --time "2026-07-05T12:59:59"
```

### Long-term Retention (LTR)

Azure SQL এ **Long-term Retention Policy** সেট করা যায়, যেখানে Weekly/Monthly/Yearly Backup ১০ বছর পর্যন্ত সংরক্ষণ করা সম্ভব — এটা Legal/Compliance প্রয়োজনে (Module 1 এ উল্লেখিত) খুবই গুরুত্বপূর্ণ Feature।

```bash
az sql db ltr-policy set \
    --resource-group myResourceGroup \
    --server myserver --database mydb \
    --weekly-retention "P12W" \
    --monthly-retention "P12M" \
    --yearly-retention "P5Y" --week-of-year 1
```

---

## 10.5 Supabase

### সংজ্ঞা

**Supabase** হলো একটা Open-source Firebase-বিকল্প, যা মূলত PostgreSQL এর উপর ভিত্তি করে তৈরি — Backend-as-a-Service (BaaS) হিসেবে জনপ্রিয়, বিশেষত Django/React/Flutter Developer দের কাছে।

### Backup Feature

| Plan | Backup Feature |
|---|---|
| Free Tier | Daily Backup, ৭ দিন Retention (সীমিত) |
| Pro Plan | Daily Backup + PITR সমর্থন (আলাদা Add-on) |
| Enterprise | Custom Retention, Cross-region Backup |

### Manual Backup নেওয়া (Supabase এ Extra নিরাপত্তার জন্য)

যেহেতু Supabase মূলত PostgreSQL, Module 6-এ শেখা `pg_dump` সরাসরি ব্যবহার করা যায় Extra Manual Backup এর জন্য (Managed Backup এর পাশাপাশি)।

```bash
pg_dump "postgresql://postgres:[PASSWORD]@[HOST]:5432/postgres" \
    -F c -f supabase_manual_backup.dump
```

### Production Note

Supabase এর Free/Pro Tier এ PITR না থাকলে, নিজে থেকে Cron Job দিয়ে নিয়মিত `pg_dump` Backup নেওয়া এবং Cloud Storage এ Upload করে রাখা একটা গুরুত্বপূর্ণ Extra Safety Layer।

---

## 10.6 PlanetScale

### সংজ্ঞা

**PlanetScale** হলো একটা Serverless MySQL-Compatible Database Platform, যা Vitess (YouTube এর তৈরি একটা MySQL Scaling System) এর উপর ভিত্তি করে তৈরি।

### বিশেষ বৈশিষ্ট্য: Branching (Git-এর মতো Database Branch)

PlanetScale এর একটা Unique Feature হলো **Database Branching** — এটা সরাসরি Backup না, কিন্তু এটা Module 3-এ শেখা Clone Backup ধারণার সাথে সম্পর্কিত।

```
Diagram: PlanetScale Branching Concept

main branch (Production)
      │
      ├── dev branch (Clone করে তৈরি, Testing এর জন্য)
      │
      └── feature-x branch (আরেকটা Clone, নতুন Feature Test করতে)
```

### Backup Feature

PlanetScale স্বয়ংক্রিয়ভাবে Automated Backup নেয় (Plan অনুযায়ী Frequency ভিন্ন), এবং তাদের Underlying Vitess Architecture এ প্রতিটা Shard এর জন্য পৃথক Backup ব্যবস্থাপনা থাকে — Module 8-এ শেখা Sharding Backup এর ধারণার সাথে সাদৃশ্যপূর্ণ।

---

## 10.7 Neon

### সংজ্ঞা

**Neon** হলো একটা Serverless PostgreSQL Platform, যার একটা মূল বৈশিষ্ট্য হলো **Storage এবং Compute আলাদা** (Separation of Storage and Compute), যা Backup/Branching কে অনেক সহজ ও দ্রুত করে তোলে।

### Instant Branching with Point-in-Time

Neon এ যেকোনো মুহূর্তের একটা Branch তৈরি করা যায় প্রায় তাৎক্ষণিকভাবে (কারণ এটা Copy-on-Write Storage ব্যবহার করে — Module 3-এ শেখা Snapshot Backup এর Copy-on-Write ধারণার সরাসরি বাস্তবায়ন)।

```
Diagram: Neon এর Copy-on-Write Branching

Main Database (Timeline)
      │
      ├──── এখানে Branch তৈরি (গতকালের একটা মুহূর্ত থেকে)
      │     (তাৎক্ষণিক, কারণ Actual Data Copy হয় না,
      │      শুধু Reference তৈরি হয়)
      │
      └──── Continue হচ্ছে (Present সময় পর্যন্ত)
```

এই Technique ব্যবহার করে তুমি Production এর যেকোনো পুরনো মুহূর্তে সরাসরি একটা নতুন, Independent Branch তৈরি করতে পারবে — যা কার্যকরভাবে একটা তাৎক্ষণিক PITR-based Clone।

---

## 10.8 Railway

### সংজ্ঞা

**Railway** হলো একটা Modern Deployment Platform, যেখানে PostgreSQL, MySQL, Redis, MongoDB সহজে Deploy করা যায় — মূলত Small-to-Medium Project এবং Startup দের জন্য জনপ্রিয়।

### Backup Feature

Railway এ Built-in Automated Backup তুলনামূলক সীমিত (Platform এবং Plan এর উপর নির্ভরশীল), তাই Railway ব্যবহার করলে Module 5/6 এ শেখা Manual Backup Script (mysqldump/pg_dump + Cron) নিজে থেকে Setup করাটা বিশেষভাবে গুরুত্বপূর্ণ।

```bash
# Railway এর Database এ External Connection দিয়ে Manual Backup
pg_dump "$DATABASE_URL" -F c -f railway_backup.dump
```

---

## 10.9 Render

### সংজ্ঞা

**Render** হলো আরেকটা Modern Cloud Platform, যেখানে PostgreSQL Managed Database হিসেবে Deploy করা যায়।

### Backup Feature

Render এ PostgreSQL Instance এর জন্য Automated Daily Backup থাকে, Retention Period Plan অনুযায়ী ভিন্ন হয় (Free Tier এ সীমিত, Paid Plan এ বেশি দিন)।

```bash
# Render Dashboard থেকে "Recovery" ট্যাবে গিয়ে Point-in-time Restore করা যায়
# অথবা CLI দিয়ে Manual Backup
pg_dump "$RENDER_DATABASE_URL" -F c -f render_backup.dump
```

---

## 10.10 DigitalOcean Managed Databases

### সংজ্ঞা

**DigitalOcean Managed Databases** MySQL, PostgreSQL, MongoDB, Redis সমর্থন করে, এবং ছোট থেকে মাঝারি Startup দের কাছে জনপ্রিয় তুলনামূলক সাশ্রয়ী মূল্যের কারণে।

### Automatic Backup

```
DigitalOcean Managed Database Backup:

├── Daily Automated Backup — Backup Window নির্বাচন করা যায়
├── Retention Period          — ৭ দিন (Standard)
└── Point-in-time Recovery     — নির্দিষ্ট Plan এ উপলব্ধ
```

### CLI দিয়ে Backup Management

```bash
# doctl (DigitalOcean CLI) দিয়ে Database Backup তালিকা দেখা
doctl databases backups list <database-cluster-id>

# একটা Backup থেকে নতুন Cluster তৈরি করা (Restore)
doctl databases create restored-db \
    --backup-restore-database-cluster-id <cluster-id> \
    --backup-restore-created-at <timestamp>
```

---

## 10.11 Managed vs Self-hosted — Decision Framework

### Comparison Table

| বিষয় | Self-hosted (নিজে Setup) | Managed Service (RDS/Cloud SQL ইত্যাদি) |
|---|---|---|
| Backup Setup | নিজে করতে হয় (Module 5-9 এ শেখা পদ্ধতি) | স্বয়ংক্রিয়, Built-in |
| PITR | নিজে Configure করতে হয় | সাধারণত Built-in (Config Enable করতে হতে পারে) |
| Cost | Infrastructure Cost কম, Engineer সময় বেশি | Subscription Cost বেশি, Setup সহজ |
| নিয়ন্ত্রণ | সম্পূর্ণ | সীমিত (Provider এর সীমাবদ্ধতার মধ্যে) |
| Restore Testing দায়িত্ব | সম্পূর্ণ নিজের | তবুও নিজের (এটা কখনোই Provider এর দায়িত্ব না) |
| Ransomware Protection (Air-gapped) | নিজে Setup করতে হয় | নিজে Extra ব্যবস্থা নিতে হয় (Cross-account/Region Copy) |

### কখন কোনটা বেছে নেবে

```
Decision Flow:

তোমার Team এ ডেডিকেটেড DBA/DevOps আছে?
    │
    ├── হ্যাঁ, বড় Scale, খরচ কমাতে চাই ──► Self-hosted (XtraBackup/pg_basebackup)
    │
    └── না, দ্রুত Development, Small Team ──► Managed Service (RDS/Cloud SQL/Supabase)
```

### সবচেয়ে গুরুত্বপূর্ণ শিক্ষা এই Module থেকে

> Managed Service ব্যবহার করলেও, **Backup Retention Policy Review করা, PITR Feature Enable করা, নিয়মিত Restore Test করা, এবং Cross-region/Cross-account Copy রাখা** — এই দায়িত্বগুলো কখনোই সম্পূর্ণভাবে Provider এর উপর ছেড়ে দেওয়া উচিত না। Module 1-এ শেখা 3-2-1 Rule এখানেও প্রযোজ্য।

---

## 10.12 Best Practices

1. Managed Service ব্যবহার করলেও PITR/Automated Backup Feature Explicitly Enable এবং Verify করো।
2. Retention Period Business Requirement (RPO/RTO, Module 4) অনুযায়ী Configure করো, Default এ ছেড়ে দিও না।
3. Critical Data এর জন্য Managed Backup এর পাশাপাশি নিজস্ব Manual Backup (pg_dump/mysqldump) ও রাখো, বিশেষ করে Free/ছোট Plan ব্যবহার করলে।
4. Cross-region Snapshot Copy রাখো Disaster Recovery এর জন্য।
5. মাসিক Restore Drill করো Managed Service এও — Provider Backup নিয়েছে মানেই Restore সফল হবে এই ধারণা যাচাই না করে বিশ্বাস করো না।

---

## 10.13 Common Mistakes

| ভুল | সমস্যা |
|---|---|
| Managed Service মানেই সম্পূর্ণ নিরাপদ ভাবা | Provider Outage/Account Compromise এ ঝুঁকি থেকে যায় |
| Default Retention Period পরীক্ষা না করা | Business Requirement এর সাথে না মিলতে পারে |
| Free Tier এ PITR না থাকা সত্ত্বেও Manual Backup না রাখা | RPO Target পূরণ না হতে পারে |
| শুধু একই Account/Region এ Backup রাখা | 3-2-1 Rule ভঙ্গ হয়, একক ঝুঁকির বিন্দু তৈরি হয় |
| Restore Process কখনো Test না করা | Managed Service হলেও Restore Fail করার সম্ভাবনা থাকে |

---

## 10.14 Interview Questions (Module 10)

1. **Managed Database Service ব্যবহার করলেও কেন নিজস্ব Backup Strategy দরকার হতে পারে?**
   *উত্তর:* কারণ Managed Backup এর Retention Period সীমিত হতে পারে, Ransomware/Account Compromise থেকে সুরক্ষার জন্য Cross-account/Region Copy দরকার, এবং Restore Testing এখনো নিজের দায়িত্ব — Provider এটা যাচাই করে দেয় না।

2. **AWS RDS এ Automatic Backup কীভাবে PITR সক্ষম করে?**
   *উত্তর:* RDS প্রতিদিন একটা Full Snapshot নেয় এবং প্রতি ৫ মিনিটে Transaction Log Backup নেয়, ফলে Retention Period এর মধ্যে যেকোনো ৫-মিনিট Granularity তে Restore করা সম্ভব হয়।

3. **Azure SQL এর Long-term Retention (LTR) Feature কী এবং কেন এটা গুরুত্বপূর্ণ?**
   *উত্তর:* এটা Weekly/Monthly/Yearly Backup ১০ বছর পর্যন্ত সংরক্ষণ করতে দেয়, যা Legal/Compliance প্রয়োজনে (যেমন নির্দিষ্ট সময় পর্যন্ত ডেটা রাখা বাধ্যতামূলক এমন আইনি প্রয়োজনে) গুরুত্বপূর্ণ।

4. **Neon এর Copy-on-Write Branching কীভাবে Backup/Restore ধারণার সাথে সম্পর্কিত?**
   *উত্তর:* এটা Module 3-এ শেখা Snapshot Backup এর Copy-on-Write Technique ব্যবহার করে, ফলে অতীতের যেকোনো মুহূর্তের একটা Independent Branch তাৎক্ষণিকভাবে তৈরি করা যায় — যা কার্যকরভাবে একটা তাৎক্ষণিক PITR-based Clone হিসেবে কাজ করে।

5. **Self-hosted এবং Managed Database Service এর মধ্যে Backup Responsibility কীভাবে ভাগ হয়?**
   *উত্তর:* Self-hosted এ Backup Setup, Automation, Monitoring সবকিছু নিজের দায়িত্ব। Managed Service এ Provider Automatic Backup নেয়, কিন্তু Retention Configuration, PITR Enable করা, এবং সবচেয়ে গুরুত্বপূর্ণভাবে Restore Testing এখনো ব্যবহারকারীর দায়িত্ব থেকে যায়।

---

## 10.15 Summary (সারাংশ)

- Managed Database Service (AWS RDS, Google Cloud SQL, Azure SQL) Backup এর অনেক কাজ স্বয়ংক্রিয় করে দেয়, কিন্তু সম্পূর্ণ দায়িত্বমুক্তি দেয় না।
- **AWS RDS** এ Automated Snapshot + ৫ মিনিট Transaction Log Backup দিয়ে PITR সম্ভব।
- **Google Cloud SQL** এ Binary Logging Enable করে PITR সক্ষম করতে হয়।
- **Azure SQL** এ Full+Differential+Log Backup Combination এবং Long-term Retention Feature আছে।
- **Supabase, Railway, Render, DigitalOcean** এর মতো ছোট/Modern Platform এ Backup Feature Plan-নির্ভর, তাই প্রায়ই Manual Backup (pg_dump/mysqldump) দিয়ে Extra Safety যোগ করা প্রয়োজন।
- **PlanetScale ও Neon** এর মতো Modern Platform Branching/Copy-on-Write Technique ব্যবহার করে Backup ও Development Workflow কে একীভূত করেছে।
- সবচেয়ে গুরুত্বপূর্ণ শিক্ষা: Managed Service ব্যবহার করলেও Retention Policy Review, Manual Backup Layer, Cross-region Copy, এবং নিয়মিত Restore Testing — এই দায়িত্ব থেকে ব্যবহারকারী কখনো সম্পূর্ণ মুক্ত না।

---

## 10.16 Exercises & Assignments

### Exercise 1
তোমার ব্যবহৃত (বা পরিকল্পিত) Cloud Provider এর Documentation পড়ে বের করো — তাদের Default Retention Period কত, এবং PITR সমর্থন করে কিনা।

### Exercise 2
একটা Free-tier Supabase/Railway/Render Database তৈরি করো এবং সেখান থেকে `pg_dump` দিয়ে একটা Manual Backup নিয়ে দেখো External Connection কীভাবে কাজ করে।

### Mini Project
একটা Comparison Document লেখো যেখানে তোমার প্রজেক্টের জন্য ৩টা Cloud Provider (যেমন AWS RDS vs Supabase vs Railway) এর Backup Feature, Cost, এবং Restore Process তুলনা করবে, এবং শেষে একটা সুপারিশ (Recommendation) দেবে কারণসহ।

---

**পরবর্তী চ্যাপ্টার:** `11-Docker-Backup.md` — এখানে আমরা Docker Volumes, Containers, Images, এবং Persistent Storage Backup/Restore নিয়ে শিখবো।
