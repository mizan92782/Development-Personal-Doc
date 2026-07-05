# Module 4: Recovery Concepts (রিকভারি ধারণাসমূহ)

> **Level:** Intermediate → Advanced
> **Prerequisite:** Module 1, 2, 3 সম্পন্ন থাকতে হবে।
> **কেন এই Module গুরুত্বপূর্ণ:** Backup নেওয়া অর্ধেক কাজ — বাকি অর্ধেক হলো সঠিকভাবে Recovery/Restore করতে পারা। এই Module-এ আমরা শিখবো কীভাবে ঠিক করা যায় "কতটুকু ডেটা হারানো মেনে নেওয়া যায়" (RPO) এবং "কত দ্রুত System আবার চালু করতে হবে" (RTO) — যা যেকোনো Production Backup Strategy ডিজাইনের ভিত্তি।

---

## 4.0 Chapter Roadmap

```
4.1  Restore কী?
4.2  Recovery কী? (এবং Restore vs Recovery পার্থক্য)
4.3  PITR (Point-In-Time Recovery)
4.4  Crash Recovery
4.5  Disaster Recovery (DR)
4.6  RPO (Recovery Point Objective)
4.7  RTO (Recovery Time Objective)
4.8  RPO এবং RTO এর সম্পর্ক ও Trade-off
4.9  Recovery Testing / Restore Drill
4.10 Best Practices
4.11 Common Mistakes
4.12 Interview Questions
4.13 Summary
4.14 Exercises & Assignments
```

---

## 4.1 Restore কী? (What is Restore?)

### সংজ্ঞা

**Restore** হলো একটা Backup ফাইল থেকে ডেটা নিয়ে Database-এ পুনরায় বসানোর প্রক্রিয়া। এটা Backup Process এর সরাসরি বিপরীত দিক (Reverse Direction)।

```
Backup:   Production Database  ──────►  Backup File
Restore:  Backup File          ──────►  Production Database
```

### বাস্তব উদাহরণ

```bash
# MySQL Restore
mysql -u root -p mydb < full_backup_2026_07_05.sql

# PostgreSQL Restore
psql -U postgres mydb < full_backup_2026_07_05.sql
```

### Restore এর ধাপসমূহ (Internal Working)

```
1. Backup ফাইল Verify করা (Corrupt কিনা যাচাই)
        │
        ▼
2. প্রয়োজনে নতুন/খালি Database তৈরি করা
        │
        ▼
3. Schema (Table Structure) তৈরি করা
        │
        ▼
4. ডেটা Insert করা (Backup থেকে)
        │
        ▼
5. Index, Constraint, Trigger পুনরায় তৈরি করা
        │
        ▼
6. Application দিয়ে Test করে নিশ্চিত হওয়া সব ঠিক আছে
```

### গুরুত্বপূর্ণ সতর্কতা

Restore Backup এর চেয়ে অনেক বেশি ঝুঁকিপূর্ণ একটা Operation, কারণ:
- এটা সাধারণত Emergency/Incident পরিস্থিতিতে চালানো হয়, যেখানে চাপ (Pressure) বেশি থাকে।
- ভুল Database-এ Restore করলে Existing ডেটা হারিয়ে যেতে পারে।
- Restore চলাকালীন সময়ে Application সাধারণত Downtime এ থাকে।

---

## 4.2 Recovery কী? (What is Recovery?)

### সংজ্ঞা

**Recovery** একটা বিস্তৃত (Broader) ধারণা — এটা শুধু Backup থেকে ডেটা ফিরিয়ে আনা না, বরং Database-কে একটা **সম্পূর্ণ কার্যকরী (Fully Functional), Consistent State** এ ফিরিয়ে আনার সম্পূর্ণ প্রক্রিয়া। Recovery-এর মধ্যে Restore একটা অংশ মাত্র।

### Restore vs Recovery — পার্থক্য

| বিষয় | Restore | Recovery |
|---|---|---|
| সংজ্ঞা | Backup ফাইল থেকে ডেটা ফিরিয়ে আনা | Database কে সম্পূর্ণ কার্যকরী State এ ফিরিয়ে আনা (Restore + আরও কিছু) |
| পরিধি | সংকীর্ণ (Narrow) — শুধু ডেটা কপি করা | ব্যাপক (Broad) — Restore + Log Apply + Verification + Application Reconnect |
| উদাহরণ ধাপ | `mysql < backup.sql` চালানো | Restore + Binlog Apply (PITR) + Data Consistency Check + Application Restart + User Notification |

### Diagram: Recovery এর সম্পূর্ণ প্রক্রিয়া (Restore এর চেয়ে বড় ধারণা)

```
┌─────────────────────────────────────────────────────────┐
│                      RECOVERY PROCESS                       │
│                                                              │
│  ┌──────────┐   ┌──────────────┐   ┌──────────────────┐   │
│  │  Restore   │──►│ Log Apply     │──►│ Consistency Check │   │
│  │ (Backup    │   │ (WAL/Binlog   │   │ (Verify Data is   │   │
│  │  থেকে ডেটা)  │   │  থেকে PITR)   │   │  Correct)          │   │
│  └──────────┘   └──────────────┘   └────────┬─────────┘   │
│                                              │              │
│                                              ▼              │
│                                   ┌─────────────────────┐  │
│                                   │ Application Reconnect  │  │
│                                   │ & Service Restart      │  │
│                                   └─────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

---

## 4.3 PITR (Point-In-Time Recovery)

### সংজ্ঞা

**PITR (Point-In-Time Recovery)** মানে হলো Database কে অতীতের **যেকোনো নির্দিষ্ট মুহূর্তের** অবস্থায় ফিরিয়ে আনার ক্ষমতা — শুধুমাত্র Last Full Backup এর মুহূর্তে না, বরং তার পরের যেকোনো সেকেন্ডে।

### কেন PITR দরকার? (বাস্তব সমস্যা)

ধরো তোমার Backup Schedule এরকম:
```
রাত ২:০০টা — Full Backup নেওয়া হয়
```

এখন যদি দুপুর ১টায় কেউ ভুলে একটা গুরুত্বপূর্ণ Table Drop করে ফেলে, তাহলে শুধু রাত ২টার Backup Restore করলে তুমি রাত ২টা থেকে দুপুর ১টা পর্যন্ত হওয়া সব ডেটা (Orders, Payments, User Registration) হারিয়ে ফেলবে।

PITR ব্যবহার করে তুমি Database-কে ঠিক দুপুর ১২:৫৯:৫৯ (Table Drop হওয়ার ঠিক আগের মুহূর্ত) পর্যন্ত ফিরিয়ে আনতে পারবে — কোনো ডেটা না হারিয়ে।

### Internal Working

```
রাত ২:০০টা                                    দুপুর ১:০০টা
Full Backup                                    ভুল করে Table Drop
      │                                              │
      ▼                                              ▼
┌──────────────┐   ┌─────────────────────────────────┐
│  Full Backup   │   │  WAL/Binlog (রাত ২টা থেকে সব       │
│  (রাত ২টার      │   │  পরিবর্তনের Record রাখা আছে)         │
│   Snapshot)     │   │                                     │
└──────┬─────────┘   └─────────────┬───────────────────┘
       │                            │
       └──────────┬─────────────────┘
                   │
                   ▼
        PITR Process: Full Backup Restore করো,
        তারপর WAL/Binlog Apply করো ঠিক দুপুর ১২:৫৯:৫৯
        পর্যন্ত (Table Drop এর আগ পর্যন্ত), তারপর থামো
                   │
                   ▼
        Database এখন দুপুর ১২:৫৯:৫৯ এর অবস্থায়
        (Table Drop হয়নি এমন অবস্থায়) ফিরে এসেছে
```

### বাস্তব উদাহরণ: MySQL এ PITR

```bash
# ধাপ ১: Full Backup Restore করা
mysql -u root -p mydb < full_backup_night2am.sql

# ধাপ ২: Binlog থেকে নির্দিষ্ট সময় পর্যন্ত Apply করা
mysqlbinlog --stop-datetime="2026-07-05 12:59:59" \
    /var/lib/mysql/mysql-bin.000123 | mysql -u root -p mydb
```

### বাস্তব উদাহরণ: PostgreSQL এ PITR

```bash
# recovery.signal ফাইল তৈরি করা (PostgreSQL 12+)
touch /var/lib/postgresql/14/main/recovery.signal

# postgresql.conf এ Recovery Target সেট করা
recovery_target_time = '2026-07-05 12:59:59'
restore_command = 'cp /wal_archive/%f %p'
```

### সুবিধা

- Exact মুহূর্ত পর্যন্ত ডেটা ফিরিয়ে আনা যায় — খুবই কম Data Loss।
- Human Error দ্রুত Recover করা যায় (কখন ভুল হয়েছে জানলে)।

### অসুবিধা

- WAL/Binlog ফাইল সংরক্ষণ ও Manage করা জটিল ও জায়গা লাগে বেশি।
- Restore Process ধীর হতে পারে (Full Backup + বড় পরিমাণ Log Apply করতে হয়)।
- সঠিক Timestamp জানা দরকার (কখন সমস্যা শুরু হয়েছিল)।

---

## 4.4 Crash Recovery (ক্র্যাশ রিকভারি)

### সংজ্ঞা

**Crash Recovery** হলো Database Server হঠাৎ বন্ধ হয়ে যাওয়ার (Crash/Power Failure) পর, Database Engine নিজে থেকেই (স্বয়ংক্রিয়ভাবে) নিজেকে একটা Consistent State এ ফিরিয়ে আনার প্রক্রিয়া — এখানে কোনো Backup ফাইলের দরকার হয় না, বরং Module 2-এ শেখা WAL/Redo Log ব্যবহার হয়।

### Internal Working

```
Database Server Crash হলো (হঠাৎ Power বন্ধ)
        │
        ▼
Database পুনরায় Start হচ্ছে
        │
        ▼
WAL/Redo Log পড়া শুরু হয়
        │
        ▼
   ┌─────────────────┬──────────────────┐
   │                                       │
   ▼                                       ▼
REDO Phase                          UNDO Phase
(যেসব Transaction Commit           (যেসব Transaction Commit
হয়েছিল কিন্তু Data File-এ           হয়নি, সেগুলো Rollback
পুরোপুরি Reflect হয়নি,              করে বাদ দেওয়া হয়)
সেগুলো আবার Apply করা হয়)
   │                                       │
   └─────────────────┬──────────────────┘
                      ▼
          Database একটা Consistent
          State এ চলে আসে, স্বয়ংক্রিয়ভাবে
```

### বাস্তব উদাহরণ: MySQL InnoDB Crash Recovery Log

```
InnoDB Server Startup এর সময় Log-এ দেখা যায়:

[Note] InnoDB: Starting crash recovery.
[Note] InnoDB: Reading tablespace information from the .ibd files...
[Note] InnoDB: Restoring possible half-written data pages
[Note] InnoDB: 128 transaction(s) which must be rolled back or cleaned up
[Note] InnoDB: Crash recovery finished.
```

### Crash Recovery vs PITR vs Backup Restore — পার্থক্য

| বিষয় | Crash Recovery | PITR | Backup Restore |
|---|---|---|---|
| কখন হয় | Server Crash/Restart হলে, স্বয়ংক্রিয় | Human Error/Disaster এর পর, Manual | Full Data Loss এর পর, Manual |
| কী লাগে | শুধু WAL/Redo Log (Server নিজেই করে) | Full Backup + WAL/Binlog | শুধু Backup ফাইল |
| ব্যবহারকারীর হস্তক্ষেপ | দরকার নেই | দরকার (Command চালাতে হয়) | দরকার (Command চালাতে হয়) |
| সময় লাগে | সেকেন্ড থেকে মিনিট | মিনিট থেকে ঘণ্টা | মিনিট থেকে ঘণ্টা (Size অনুযায়ী) |

### Production Note

Crash Recovery প্রতিদিনের একটা স্বাভাবিক প্রক্রিয়া — যেকোনো ভালো Database Engine (InnoDB, PostgreSQL) এটা নিজে থেকেই পরিচালনা করে। কিন্তু Engineer হিসেবে তোমার জানা দরকার এটা কীভাবে কাজ করে, কারণ যদি Crash Recovery ব্যর্থ হয় (যেমন WAL ফাইল নিজেই Corrupt হয়ে যায়), তখনই তোমাকে Backup থেকে Restore করার প্রয়োজন হবে।

---

## 4.5 Disaster Recovery (DR) — দুর্যোগ পুনরুদ্ধার

### সংজ্ঞা

**Disaster Recovery (DR)** হলো একটা সামগ্রিক (Comprehensive) পরিকল্পনা ও প্রক্রিয়া, যা একটা সম্পূর্ণ বড় Disaster (যেমন পুরো Data Center ধ্বংস হওয়া, দীর্ঘ Regional Outage) এর পর ব্যবসা/সার্ভিস পুনরায় চালু করতে ব্যবহৃত হয়। এটা Single Database Restore এর চেয়ে অনেক বড় পরিসরের ধারণা — এখানে পুরো Infrastructure, Network, Application, এবং Database সবকিছু জড়িত।

### Disaster Recovery Plan (DRP) এর উপাদান

```
Disaster Recovery Plan
│
├── 1. Risk Assessment (কী কী Disaster হতে পারে)
├── 2. Backup Strategy (Data কীভাবে সুরক্ষিত রাখা হবে)
├── 3. Failover Plan (কোন Server/Region এ সরে যাবে)
├── 4. Communication Plan (কাকে কখন জানানো হবে)
├── 5. Recovery Procedure (ধাপে ধাপে কী করতে হবে - Runbook)
├── 6. Testing Schedule (কত ঘন ঘন DR Drill করা হবে)
└── 7. RPO/RTO Targets (কতটুকু Data Loss ও কত সময় মেনে নেওয়া যায়)
```

### DR Architecture উদাহরণ: Multi-Region Setup

```
┌─────────────────────────┐         ┌─────────────────────────┐
│   Primary Region (US)      │         │   DR Region (Europe)      │
│                             │         │                            │
│  ┌───────────────────┐    │         │  ┌───────────────────┐   │
│  │  Active Database     │    │ Replication │  Standby Database    │   │
│  │  (Read + Write)       │────┼─────────►│  (Read-only / Idle)   │   │
│  └───────────────────┘    │         │  └───────────────────┘   │
│                             │         │                            │
│  Application Servers        │         │  Application Servers        │
│  (Handling Live Traffic)    │         │  (Ready but Idle)            │
└─────────────────────────┘         └─────────────────────────┘
        │
        │  Disaster হলে (Region ধ্বংস/Outage)
        ▼
     Failover — DNS/Traffic DR Region এ Redirect করা হয়
```

### DR এর প্রকারভেদ (DR Site Types)

| Type | ব্যাখ্যা | Cost | Recovery Time |
|---|---|---|---|
| Cold Site | শুধু Infrastructure Ready, ডেটা নেই | কম | ধীর (দিন/সপ্তাহ) |
| Warm Site | Infrastructure + মাঝে মাঝে Sync হওয়া ডেটা | মাঝারি | মাঝারি (ঘণ্টা) |
| Hot Site | সম্পূর্ণ Ready, Real-time Sync ডেটা | বেশি | দ্রুত (মিনিট/সেকেন্ড) |

### Company Example

Netflix তাদের Multi-Region Architecture ব্যবহার করে এমনভাবে ডিজাইন করেছে যে একটা পুরো AWS Region Down হয়ে গেলেও (যেমন ২০১২ সালে AWS US-EAST-1 Outage), তাদের Service অন্য Region থেকে চলতে থাকে — এটাই একটা বাস্তব Hot Site DR এর উদাহরণ।

---

## 4.6 RPO (Recovery Point Objective)

### সংজ্ঞা

**RPO (Recovery Point Objective)** হলো একটা সময়ের পরিমাপ, যা বলে দেয় Disaster হওয়ার সময় **সর্বোচ্চ কতটুকু ডেটা হারানো মেনে নেওয়া যায়** (Backup এর মধ্যে ব্যবধান)।

> সহজ ভাষায়: "শেষ কতক্ষণ আগের ডেটা পর্যন্ত আমরা ফিরে যেতে রাজি আছি?"

### RPO এর ব্যাখ্যা Diagram দিয়ে

```
Timeline:  ────────────────────────────────────────►
              │                              │
        Last Backup                    Disaster ঘটলো
        (রাত ২টা)                       (দুপুর ১টা)

RPO = 1 ঘণ্টা হলে অর্থ:
"আমরা সর্বোচ্চ ১ ঘণ্টার ডেটা হারাতে রাজি আছি"
তার মানে আমাদের Backup Frequency (কতক্ষণ পর পর Backup নেওয়া হয়)
কমপক্ষে প্রতি ১ ঘণ্টায় একবার হতে হবে, না হলে RPO Target পূরণ হবে না।
```

### RPO নির্ধারণ কীভাবে Backup Strategy ঠিক করে

| RPO Target | প্রয়োজনীয় Backup Strategy |
|---|---|
| ২৪ ঘণ্টা | দৈনিক Full Backup যথেষ্ট |
| ১ ঘণ্টা | ঘন ঘন Incremental Backup বা WAL/Binlog Archiving |
| কয়েক সেকেন্ড | Continuous Backup (CDP) বা Synchronous Replication |
| শূন্য (Zero Data Loss) | Synchronous Multi-node Replication (ব্যয়বহুল) |

### বাস্তব উদাহরণ

- একটা Blog Website এর RPO ২৪ ঘণ্টা হলেও চলবে (দিনে একবার Backup যথেষ্ট)।
- একটা Banking System এর RPO প্রায় শূন্য হতে হবে (প্রতিটা Transaction গুরুত্বপূর্ণ)।

---

## 4.7 RTO (Recovery Time Objective)

### সংজ্ঞা

**RTO (Recovery Time Objective)** হলো Disaster হওয়ার পর System পুনরায় সম্পূর্ণ কার্যকরী (Functional) হতে **সর্বোচ্চ কত সময় লাগতে পারে** তার লক্ষ্যমাত্রা।

> সহজ ভাষায়: "Disaster হওয়ার পর সিস্টেম আবার চালু হতে সর্বোচ্চ কতক্ষণ সময় নেওয়া যাবে?"

### RTO এর ব্যাখ্যা Diagram দিয়ে

```
Timeline:  ────────────────────────────────────────►
              │                              │
        Disaster ঘটলো                  System আবার
        (দুপুর ১টা)                     চালু হলো

RTO = 4 ঘণ্টা হলে অর্থ:
"Disaster হওয়ার পর সর্বোচ্চ ৪ ঘণ্টার মধ্যে System
আবার সম্পূর্ণ কার্যকরী করতে হবে"
```

### RTO Target অনুযায়ী প্রয়োজনীয় Infrastructure

| RTO Target | প্রয়োজনীয় Approach |
|---|---|
| কয়েকদিন | Cold Backup + Manual Restore Process |
| কয়েক ঘণ্টা | Automated Restore Script + Physical Backup |
| কয়েক মিনিট | Hot Standby Server + Automated Failover |
| প্রায় তাৎক্ষণিক (সেকেন্ড) | Active-Active Multi-Region Architecture, Load Balancer Auto-failover |

---

## 4.8 RPO এবং RTO এর সম্পর্ক ও Trade-off

### একসাথে বোঝা

```
              Disaster ঘটলো
                    │
      ◄─────────────┼─────────────►
        RPO                    RTO
   (কতটুকু ডেটা              (কত সময়ে System
    হারালাম, পেছনের             আবার চালু হলো,
    দিকে পরিমাপ)                সামনের দিকে পরিমাপ)

Timeline:
─────●───────────────●───────────────●──────────►
  Last Backup      Disaster      System Restored
     (RPO এখান           (এখানে ঘটলো)      (RTO এখানে
      থেকে পরিমাপ                          শেষ হলো)
      শুরু)
```

### Cost vs RPO/RTO Trade-off

RPO এবং RTO যত কম (কঠোর) হবে, Infrastructure তত বেশি জটিল ও ব্যয়বহুল হবে।

```
Diagram: Cost বনাম RPO/RTO সম্পর্ক

উচ্চ Cost  │                                    ●  (RPO≈0, RTO≈0)
           │                          ●  (Hot Standby)
           │                ●  (Warm Standby)
           │      ●  (Daily Backup)
নিম্ন Cost  │  ●  (Weekly Backup)
           └──────────────────────────────────────►
             ধীর RPO/RTO                দ্রুত RPO/RTO
             (বেশি Data Loss             (কম Data Loss
              সহনীয়, বেশি Downtime       সহনীয়, কম Downtime
              সহনীয়)                     সহনীয়)
```

### Table: বিভিন্ন Business এর জন্য উপযুক্ত RPO/RTO

| Business Type | সহনীয় RPO | সহনীয় RTO | কারণ |
|---|---|---|---|
| Personal Blog | ২৪-৪৮ ঘণ্টা | কয়েক ঘণ্টা-দিন | কম গুরুত্বপূর্ণ ডেটা |
| E-commerce (মাঝারি) | ১ ঘণ্টা | ৩০ মিনিট-১ ঘণ্টা | Order/Payment গুরুত্বপূর্ণ |
| Banking System | কয়েক সেকেন্ড | কয়েক মিনিট | প্রতিটা Transaction অতি গুরুত্বপূর্ণ |
| Healthcare System | কয়েক মিনিট | কয়েক মিনিট | জীবন-মৃত্যু সম্পর্কিত তথ্য |

### কীভাবে RPO/RTO ঠিক করবে (Practical Approach)

1. Business Stakeholder দের সাথে আলোচনা করো — "কতটুকু ডেটা হারানো মেনে নেওয়া যাবে?" এবং "কত সময় System বন্ধ থাকলে ব্যবসার ক্ষতি হবে?"
2. Cost হিসাব করো — কঠোর RPO/RTO এর জন্য কত Infrastructure Cost লাগবে।
3. একটা Balance বের করো যা Business Requirement এবং Budget দুটোই মেটায়।

---

## 4.9 Recovery Testing / Restore Drill (রিকভারি পরীক্ষা)

### কেন এটা সবচেয়ে গুরুত্বপূর্ণ কিন্তু সবচেয়ে অবহেলিত অংশ

Module 1-এ GitLab এর উদাহরণ মনে আছে? তাদের ৫টা Backup Mechanism ছিল, কিন্তু ৪টাই কাজ করেনি কারণ তারা কখনো Test করেনি। এই ভুল এড়ানোর জন্যই **Restore Drill** করতে হয়।

### Restore Drill Process

```
1. একটা Isolated/Test Environment তৈরি করা (Production থেকে আলাদা)
        │
        ▼
2. সর্বশেষ Backup ফাইল ব্যবহার করে সেখানে Restore করা
        │
        ▼
3. Restore হওয়া ডেটা Verify করা:
   - সব Table আছে কিনা?
   - Row Count মিলছে কিনা?
   - Application সঠিকভাবে Connect হচ্ছে কিনা?
   - Sample Query চালিয়ে ডেটা সঠিক কিনা যাচাই করা
        │
        ▼
4. সময় মাপা — কত সময় লাগলো সম্পূর্ণ Restore করতে (RTO এর সাথে তুলনা করা)
        │
        ▼
5. ফলাফল Document করা এবং কোনো সমস্যা পেলে Fix করা
```

### Restore Drill এর Frequency

| Environment Type | সুপারিশকৃত Frequency |
|---|---|
| Critical Production System | মাসে অন্তত ১ বার |
| সাধারণ Production System | প্রতি ৩ মাসে ১ বার |
| Development/Internal Tool | প্রতি ৬ মাসে ১ বার |

### Production Note

কোনো Senior DBA বা Backend Engineer কখনো বলবে না "আমাদের Backup আছে, তাই আমরা নিরাপদ।" তারা বলবে "আমাদের Backup আছে **এবং আমরা গত মাসে সফলভাবে সেটা Restore করে পরীক্ষা করেছি**।" এই দুইটা বাক্যের মধ্যে বিশাল পার্থক্য।

---

## 4.10 Best Practices

1. প্রতিটা Database/Service এর জন্য স্পষ্টভাবে RPO এবং RTO নির্ধারণ করো, Business এর সাথে আলোচনা করে।
2. PITR সক্ষম রাখতে WAL/Binlog সবসময় Enable রাখো এবং Archive করো।
3. নিয়মিত Restore Drill করো — Calendar এ Reminder সেট করে রাখো।
4. Disaster Recovery Plan Document লিখে রাখো (Runbook আকারে), যাতে Emergency এর সময় কেউ Step-by-step অনুসরণ করতে পারে।
5. Crash Recovery Log নিয়মিত Monitor করো — যদি বারবার Crash Recovery ঘটে, সেটা একটা বড় সমস্যার লক্ষণ হতে পারে।

---

## 4.11 Common Mistakes

| ভুল | সমস্যা |
|---|---|
| RPO/RTO নির্ধারণ না করেই Backup Strategy বানানো | Backup Frequency ভুল হতে পারে, Business Need এর সাথে না মিলতে পারে |
| Restore Drill কখনো না করা | Backup আসলে কাজ করে কিনা জানা যায় না (GitLab এর মতো ঘটনা ঘটতে পারে) |
| PITR এর জন্য WAL/Binlog সংরক্ষণ না করা | শুধু Full Backup Point পর্যন্তই Restore সম্ভব হয় |
| Disaster Recovery Plan লিখিতভাবে না রাখা | Emergency এর সময় বিভ্রান্তি ও ভুল সিদ্ধান্ত হতে পারে |
| Restore কে Backup এর মতো সহজ ভাবা | Restore আসলে অনেক বেশি ঝুঁকিপূর্ণ ও জটিল Process |

---

## 4.12 Interview Questions (Module 4)

1. **RPO এবং RTO এর মধ্যে পার্থক্য কী?**
   *উত্তর:* RPO বলে দেয় Disaster হলে সর্বোচ্চ কতটুকু ডেটা হারানো মেনে নেওয়া যায় (Backup Frequency নির্ধারণ করে), আর RTO বলে দেয় Disaster এর পর সর্বোচ্চ কত সময়ে System আবার চালু করতে হবে (Recovery Speed নির্ধারণ করে)।

2. **PITR কীভাবে কাজ করে?**
   *উত্তর:* PITR-এ প্রথমে একটা Full Backup Restore করা হয়, তারপর সেই Backup নেওয়ার পর থেকে সংরক্ষিত WAL/Binlog ফাইল থেকে নির্দিষ্ট একটা সময় (Timestamp) পর্যন্ত পরিবর্তনগুলো Apply করা হয়।

3. **Crash Recovery এবং Backup Restore এর মধ্যে পার্থক্য কী?**
   *উত্তর:* Crash Recovery স্বয়ংক্রিয়ভাবে ঘটে Database Server Restart হওয়ার সময়, WAL/Redo Log ব্যবহার করে — এখানে কোনো Backup ফাইল লাগে না। Backup Restore হলো Manual Process যেখানে একটা আলাদা Backup ফাইল থেকে ডেটা ফিরিয়ে আনা হয়।

4. **Restore Drill কেন গুরুত্বপূর্ণ?**
   *উত্তর:* কারণ Backup নেওয়া হয়েছে মানেই এই না যে সেটা সফলভাবে Restore করা যাবে। নিয়মিত Test না করলে GitLab এর মতো ঘটনা ঘটতে পারে, যেখানে সব Backup ব্যর্থ হয়েছিল Emergency মুহূর্তে।

5. **একটা Banking System এর জন্য RPO/RTO কেমন হওয়া উচিত এবং কেন?**
   *উত্তর:* RPO প্রায় শূন্য (কয়েক সেকেন্ড) এবং RTO কয়েক মিনিট হওয়া উচিত, কারণ প্রতিটা Financial Transaction অত্যন্ত গুরুত্বপূর্ণ এবং সামান্য Data Loss বা দীর্ঘ Downtime বড় আর্থিক ও Trust এর ক্ষতি করতে পারে।

6. **Cold, Warm, এবং Hot DR Site এর মধ্যে পার্থক্য কী?**
   *উত্তর:* Cold Site এ শুধু Infrastructure Ready থাকে, ডেটা থাকে না (ধীর Recovery)। Warm Site এ Infrastructure + মাঝে মাঝে Sync হওয়া ডেটা থাকে। Hot Site এ সম্পূর্ণ Ready অবস্থা ও Real-time Sync ডেটা থাকে (দ্রুততম Recovery, কিন্তু সবচেয়ে ব্যয়বহুল)।

---

## 4.13 Summary (সারাংশ)

- **Restore** হলো Backup থেকে ডেটা ফিরিয়ে আনা, আর **Recovery** হলো এর চেয়ে বিস্তৃত — Database কে সম্পূর্ণ কার্যকরী State এ ফিরিয়ে আনার পুরো প্রক্রিয়া।
- **PITR** ব্যবহার করে Full Backup + WAL/Binlog মিলিয়ে যেকোনো নির্দিষ্ট মুহূর্তে Restore করা যায়।
- **Crash Recovery** স্বয়ংক্রিয়ভাবে ঘটে Server Restart হওয়ার সময়, WAL/Redo Log ব্যবহার করে — Manual Backup এর দরকার হয় না।
- **Disaster Recovery (DR)** হলো বড় পরিসরের একটা পরিকল্পনা যা পুরো Data Center/Region ধ্বংস হলেও ব্যবসা চালু রাখে।
- **RPO** নির্ধারণ করে কতটুকু ডেটা হারানো মেনে নেওয়া যায়, **RTO** নির্ধারণ করে কত সময়ে System চালু হতে হবে।
- RPO/RTO যত কঠোর (কম), Infrastructure তত জটিল ও ব্যয়বহুল হবে — এটা একটা Cost vs Reliability Trade-off।
- **Restore Drill/Testing** নিয়মিত করা Backup Strategy এর সবচেয়ে গুরুত্বপূর্ণ কিন্তু প্রায়ই অবহেলিত অংশ।

---

## 4.14 Exercises & Assignments

### Exercise 1
তোমার নিজের একটা Test Database এ একটা Full Backup নাও, তারপর কিছু ডেটা পরিবর্তন করো, তারপর একটা "ভুল করে টেবিল Drop" এর পরিস্থিতি তৈরি করো। এরপর PITR টেকনিক ব্যবহার করে (Binlog/WAL এর সাহায্যে) সেই Drop হওয়ার ঠিক আগের মুহূর্ত পর্যন্ত Restore করার চেষ্টা করো।

### Exercise 2
তোমার নিজের প্রজেক্টের জন্য RPO এবং RTO লিখে ফেলো — কারণসহ ব্যাখ্যা করো কেন তুমি এই নির্দিষ্ট মান বেছে নিয়েছো।

### Mini Project
একটা সম্পূর্ণ **Disaster Recovery Runbook** লেখো (Markdown ফাইল আকারে), যাতে থাকবে:
- কোন কোন পরিস্থিতিতে DR Activate হবে
- ধাপে ধাপে কী করতে হবে (Restore Command সহ)
- কাকে কখন জানাতে হবে
- RPO/RTO Target
- Restore Drill এর Schedule ও গত ফলাফল

---

**পরবর্তী চ্যাপ্টার:** `05-MySQL-Backup.md` — এখানে আমরা MySQL এর জন্য নির্দিষ্ট Backup Technique (mysqldump, mysqlpump, Binary Logs, GTID, XtraBackup, Clone Plugin, Automation) নিয়ে গভীরভাবে শিখবো, Module 1-4 এ শেখা তত্ত্বীয় ভিত্তি ব্যবহার করে।
