# Module 2: Database Architecture Basics (ডেটাবেজ আর্কিটেকচারের মৌলিক ধারণা)

> **Level:** Beginner → Intermediate
> **Prerequisite:** Module 1 (Introduction to Database Backup) সম্পন্ন করা থাকতে হবে।
> **কেন এই Module গুরুত্বপূর্ণ:** Backup Strategy সঠিকভাবে বোঝার জন্য প্রথমে জানতে হবে Database ভেতরে কীভাবে কাজ করে — ডেটা কীভাবে Disk-এ লেখা হয়, Transaction কীভাবে কাজ করে, এবং কেন WAL/Binlog এর মতো জিনিস দরকার হয়।

---

## 2.0 Chapter Roadmap

```
2.1 Storage Engine কী?
2.2 Disk এবং Memory এর ভূমিকা
2.3 Transaction কী?
2.4 Commit কী?
2.5 Rollback কী?
2.6 ACID Properties
2.7 WAL (Write Ahead Log)
2.8 Binlog (Binary Log)
2.9 এই সব কিছু কীভাবে Backup এর সাথে সম্পর্কিত
2.10 Best Practices
2.11 Common Mistakes
2.12 Interview Questions
2.13 Summary
2.14 Exercises & Assignments
```

---

## 2.1 Storage Engine কী? (What is a Storage Engine?)

### সংজ্ঞা

**Storage Engine** হলো Database Management System (DBMS)-এর সেই অংশ, যেটা প্রকৃতপক্ষে ঠিক করে ডেটা কীভাবে Disk-এ সংরক্ষণ (Store), পড়া (Read), এবং লেখা (Write) হবে।

তুমি MySQL ব্যবহার করলে সাধারণত **InnoDB** নামের Storage Engine ব্যবহার হয় (Default)। এছাড়া MyISAM নামেও একটা পুরনো Engine আছে।

PostgreSQL-এ আলাদা করে Storage Engine বলে কিছু নেই বলা যায় (এটার নিজস্ব একটাই Storage System আছে, যাকে বলে **Heap Storage**), কিন্তু ধারণাগতভাবে কাজ একই রকম।

### কেন Storage Engine এর ধারণা গুরুত্বপূর্ণ?

কারণ Backup নেওয়ার সময় তুমি জানতে চাইবে:
- ডেটা কোন ফাইলে Physically সংরক্ষিত থাকে?
- সেই ফাইল Copy করলেই কি Backup হয়ে যাবে, নাকি আরও কিছু লাগবে?

### Diagram: Storage Engine Layer

```
┌───────────────────────────────────┐
│         Application Layer          │  (Django, Spring Boot)
└───────────────┬────────────────────┘
                 │  SQL Query
                 ▼
┌───────────────────────────────────┐
│      Database Server (MySQL)       │
│  ┌───────────────────────────────┐│
│  │      SQL Parser & Optimizer    ││
│  └───────────────┬───────────────┘│
│                   ▼                │
│  ┌───────────────────────────────┐│
│  │   Storage Engine (InnoDB)      ││ ◄── এখানে ডেটা কীভাবে
│  └───────────────┬───────────────┘│     Disk এ যাবে তা ঠিক হয়
└──────────────────┼─────────────────┘
                    ▼
         ┌─────────────────────┐
         │   Physical Disk       │
         │  (.ibd, .frm files)   │
         └─────────────────────┘
```

### MySQL InnoDB এর ফাইল স্ট্রাকচার (উদাহরণ)

```bash
/var/lib/mysql/
├── ibdata1              # System Tablespace
├── ib_logfile0           # Redo Log (WAL এর মতো কাজ করে)
├── ib_logfile1
├── my_database/
│   ├── users.ibd         # প্রতিটা Table এর জন্য আলাদা .ibd ফাইল
│   ├── orders.ibd
│   └── db.opt
```

### Table: জনপ্রিয় Storage Engine তুলনা

| Storage Engine | Database | Transaction Support | Crash Recovery | ব্যবহার |
|---|---|---|---|---|
| InnoDB | MySQL | হ্যাঁ (ACID) | হ্যাঁ | Default, Production-এ সবচেয়ে বেশি ব্যবহৃত |
| MyISAM | MySQL | না | সীমিত | পুরনো System, Read-heavy কাজে |
| Heap Storage | PostgreSQL | হ্যাঁ (ACID) | হ্যাঁ (WAL দিয়ে) | Default |
| WiredTiger | MongoDB | হ্যাঁ | হ্যাঁ | Default MongoDB Engine |

### Production Note

Backup নেওয়ার সময় যদি তুমি না জানো তোমার Database কোন Storage Engine ব্যবহার করছে, তাহলে ভুল Backup Method বেছে নেওয়ার ঝুঁকি থাকে। যেমন, MyISAM-এর জন্য যে Backup Method কাজ করবে (Simple File Copy), InnoDB-তে সেটা করলে Corrupt Backup হতে পারে যদি সঠিকভাবে না করা হয় (কারণ InnoDB-তে Active Transaction চলমান থাকতে পারে)।

---

## 2.2 Disk এবং Memory এর ভূমিকা (Role of Disk & Memory)

### কেন এটা জানা দরকার?

Backup বোঝার জন্য সবচেয়ে গুরুত্বপূর্ণ ধারণা হলো — **ডেটা সবসময় সাথে সাথে Disk-এ লেখা হয় না**। এটা প্রথমে Memory (RAM)-তে থাকে, তারপর একটা নির্দিষ্ট সময় পর Disk-এ যায়। এই ধারণাটা না বুঝলে Backup কেন Consistency (সামঞ্জস্যতা) নিয়ে সমস্যা তৈরি করতে পারে সেটা বোঝা যাবে না।

### Diagram: Memory থেকে Disk পর্যন্ত ডেটার যাত্রা

```
Application: INSERT INTO orders VALUES (...)
        │
        ▼
┌─────────────────────────────┐
│     Buffer Pool (RAM)         │  ◄── ডেটা প্রথমে এখানে যায় (Fast, কিন্তু Volatile)
│  (InnoDB Buffer Pool /        │
│   PostgreSQL Shared Buffers)  │
└───────────────┬───────────────┘
                │
        একটা নির্দিষ্ট সময় পর
        (Checkpoint/Flush)
                │
                ▼
┌─────────────────────────────┐
│      Physical Disk             │  ◄── এখানে ডেটা স্থায়ীভাবে থাকে (Slow, কিন্তু Persistent)
│   (.ibd / .frm / Data Files)   │
└─────────────────────────────┘
```

### গুরুত্বপূর্ণ প্রশ্ন: যদি Disk-এ লেখার আগেই Power চলে যায়?

এখানেই **WAL (Write Ahead Log)** বা **Redo Log** এর ভূমিকা আসে (2.7 এ বিস্তারিত)। সংক্ষেপে বললে — Database প্রথমে একটা ছোট Log ফাইলে লিখে রাখে "আমি এই পরিবর্তনটা করতে যাচ্ছি", তারপরই Actual Data File-এ পরিবর্তন করে। এভাবে Power চলে গেলেও, Log ফাইল দেখে Database Restart হওয়ার সময় বুঝতে পারে কোন Change গুলো Complete হয়নি এবং সেগুলো আবার সম্পন্ন করে (Crash Recovery)।

### Memory vs Disk তুলনা

| বৈশিষ্ট্য | Memory (RAM) | Disk (HDD/SSD) |
|---|---|---|
| গতি | অনেক দ্রুত (Nanoseconds) | তুলনামূলক ধীর (Microseconds-Milliseconds) |
| স্থায়িত্ব | Volatile (Power গেলে হারিয়ে যায়) | Persistent (Power গেলেও থাকে) |
| দাম | বেশি (per GB) | কম |
| Backup Relevance | Backup নেওয়ার সময় RAM-এর ডেটা বিবেচনা করতে হয় | Backup মূলত Disk-এর ডেটা নিয়ে কাজ করে |

---

## 2.3 Transaction কী? (What is a Transaction?)

### সংজ্ঞা

**Transaction** হলো একটা বা একাধিক Database Operation-এর (INSERT, UPDATE, DELETE) একটা গ্রুপ, যেগুলো একসাথে **সব সফল হবে, নয়তো সব বাতিল হবে** — মাঝামাঝি কোনো অবস্থা থাকবে না।

### বাস্তব উদাহরণ: Bank Transfer

ধরো তুমি একজনের Account থেকে আরেকজনের Account-এ টাকা Transfer করছো। এটা আসলে দুইটা আলাদা Operation:

```sql
BEGIN TRANSACTION;

UPDATE accounts SET balance = balance - 1000 WHERE id = 'A';  -- টাকা কমানো
UPDATE accounts SET balance = balance + 1000 WHERE id = 'B';  -- টাকা যোগ করা

COMMIT;
```

যদি প্রথম Query সফল হয় কিন্তু দ্বিতীয় Query চালানোর সময় Server Crash করে, তাহলে কী হবে? A এর টাকা কমে গেছে, কিন্তু B এর টাকা বাড়েনি — এটা একটা **Data Inconsistency (অসামঞ্জস্যতা)**।

Transaction এর কাজ হলো এই সমস্যা প্রতিরোধ করা — হয় দুইটা Query-ই সফল হবে (Commit), নয়তো দুইটাই বাতিল হয়ে যাবে (Rollback), যেন কখনোই "A এর টাকা কমলো কিন্তু B এর টাকা বাড়লো না" — এই অবস্থা তৈরি না হয়।

### Diagram: Transaction Lifecycle

```
        BEGIN TRANSACTION
                │
                ▼
      ┌───────────────────┐
      │  Execute Queries    │
      │  (INSERT/UPDATE/    │
      │   DELETE)           │
      └─────────┬───────────┘
                │
       ┌────────┴─────────┐
       │                    │
       ▼                    ▼
   সব ঠিক আছে         কোনো সমস্যা হলো
       │                    │
       ▼                    ▼
   COMMIT             ROLLBACK
  (Permanent হবে)    (সব বাতিল হবে)
```

### কেন এটা Backup এর সাথে সম্পর্কিত?

Backup নেওয়ার সময় যদি একটা Transaction "Half-complete" অবস্থায় থাকে (অর্থাৎ Commit বা Rollback কোনোটাই হয়নি), তাহলে সেই Backup নিয়ে Restore করলে ডেটা Inconsistent হতে পারে। এই কারণেই Modern Backup Tools (mysqldump --single-transaction, pg_dump) এমনভাবে ডিজাইন করা হয়েছে যাতে Backup নেওয়ার সময় একটা **Consistent Snapshot** নেওয়া যায় — অর্থাৎ Backup শুধু সম্পূর্ণ Commit হওয়া Transaction-গুলোই Include করবে।

---

## 2.4 Commit কী? (What is Commit?)

### সংজ্ঞা

**Commit** মানে হলো একটা Transaction-এর সব পরিবর্তন স্থায়ীভাবে (Permanently) Database-এ সংরক্ষণ করা। একবার Commit হয়ে গেলে, সেই পরিবর্তন আর ফেরত নেওয়া যায় না (স্বাভাবিক উপায়ে) — নতুন কোনো Transaction দিয়ে আবার পরিবর্তন করতে হবে।

```sql
BEGIN;
INSERT INTO users (name, email) VALUES ('Rahim', 'rahim@example.com');
COMMIT;  -- এখন এই ডেটা স্থায়ী হয়ে গেলো
```

### Commit এর Internal Working

যখন তুমি `COMMIT` চালাও, তখন Database Engine নিশ্চিত করে যে:
1. সব পরিবর্তন WAL/Redo Log-এ লেখা হয়ে গেছে (Disk-এ, Durable ভাবে)।
2. একটা "Commit Marker" Log-এ লেখা হয়।
3. এরপরই Client-কে "Success" জানানো হয়।

এই প্রক্রিয়াটাকেই বলে **Durability** (ACID এর "D") — একবার Commit হয়ে গেলে, সেই ডেটা Server Crash হলেও হারাবে না, কারণ সেটা ইতিমধ্যে Log-এ Persist (স্থায়ী) হয়ে গেছে।

---

## 2.5 Rollback কী? (What is Rollback?)

### সংজ্ঞা

**Rollback** মানে হলো একটা Transaction-এর সব পরিবর্তন বাতিল করে দেওয়া, যেন সেই Transaction কখনো ঘটেইনি।

```sql
BEGIN;
UPDATE accounts SET balance = balance - 5000 WHERE id = 'A';
-- হঠাৎ বুঝলে A এর Balance যথেষ্ট না, ভুল হয়ে গেছে
ROLLBACK;  -- আগের অবস্থায় ফিরে যাবে
```

### কখন Rollback হয়?

1. Application কোড থেকে ইচ্ছাকৃতভাবে Rollback করা হলে।
2. কোনো Constraint Violation হলে (যেমন Duplicate Primary Key)।
3. Deadlock হলে (দুইটা Transaction একে অপরের জন্য অপেক্ষা করলে)।
4. Server Crash হলে — Restart হওয়ার সময় Database সব Uncommitted Transaction Rollback করে দেয় (Crash Recovery এর অংশ হিসেবে)।

---

## 2.6 ACID Properties (ACID বৈশিষ্ট্যসমূহ)

Database Transaction-কে নির্ভরযোগ্য করতে ৪টা মূলনীতি অনুসরণ করা হয়, যাকে বলে **ACID**।

### A — Atomicity (অবিভাজ্যতা)

একটা Transaction-এর সব Operation হয় সম্পূর্ণভাবে হবে, নয়তো একদমই হবে না। মাঝামাঝি কিছু হবে না।

### C — Consistency (সামঞ্জস্যতা)

Transaction শেষ হওয়ার পর Database সবসময় একটা বৈধ (Valid) অবস্থায় থাকবে — কোনো Rule/Constraint ভঙ্গ হবে না (যেমন, Bank Total Balance সবসময় সঠিক থাকবে)।

### I — Isolation (বিচ্ছিন্নতা)

একইসাথে একাধিক Transaction চললেও, একটা Transaction অন্যটার Intermediate (মাঝামাঝি) অবস্থা দেখতে পারবে না। যেন প্রতিটা Transaction একা একা চলছে (Isolated)।

### D — Durability (স্থায়িত্ব)

একবার Commit হয়ে গেলে, সেই পরিবর্তন স্থায়ী থাকবে — এমনকি Server Crash হলেও।

### Table: ACID Summary

| অক্ষর | নাম | ব্যাখ্যা | Backup এর সাথে সম্পর্ক |
|---|---|---|---|
| A | Atomicity | সব বা কিছুই না | Backup-এ Half-complete Transaction থাকা উচিত না |
| C | Consistency | সবসময় Valid State | Backup Restore করার পর Data Consistent থাকতে হবে |
| I | Isolation | একে অপরকে প্রভাবিত করে না | Backup নেওয়ার সময় Running Transaction থেকে আলাদা View দরকার (Snapshot Isolation) |
| D | Durability | Commit হলে স্থায়ী | WAL/Binlog এই Durability নিশ্চিত করে, এবং Backup এর ভিত্তি তৈরি করে |

### Diagram: ACID এর ভূমিকা Backup Consistency-তে

```
   Transaction 1 (Running)      Transaction 2 (Committed)
          │                              │
          │  Backup শুরু হলো             │
          ▼                              ▼
   ┌─────────────────────────────────────────┐
   │   Consistent Snapshot (Backup Point)      │
   │   শুধু Committed ডেটা Include হবে,        │
   │   Running/Uncommitted ডেটা বাদ যাবে       │
   └─────────────────────────────────────────┘
```

---

## 2.7 WAL (Write Ahead Log)

### সংজ্ঞা

**WAL (Write-Ahead Log)** হলো একটা Logging পদ্ধতি, যেখানে Database কোনো Data File-এ পরিবর্তন করার **আগেই**, সেই পরিবর্তনের একটা Record একটা আলাদা Log ফাইলে লিখে রাখে।

> নামের মধ্যেই উত্তর আছে: **"Write AHEAD"** — অর্থাৎ, Actual Data পরিবর্তনের আগে Log-এ লেখা।

### কেন WAL দরকার?

ধরো তুমি একটা বড় Table Update করছো, আর মাঝপথে হঠাৎ Power চলে গেলো। Data File আংশিক পরিবর্তন হয়ে থাকতে পারে — এটা একটা Corrupt অবস্থা। কিন্তু WAL এ যেহেতু আগে থেকেই লেখা আছে ঠিক কী পরিবর্তন হওয়ার কথা ছিল, তাই Database Restart হওয়ার সময় সেই WAL পড়ে বুঝতে পারে:

1. কোন Transaction Commit হয়ে গিয়েছিল কিন্তু Data File-এ পুরোপুরি Reflect হয়নি → সেটা আবার Apply (REDO) করে।
2. কোন Transaction Commit হয়নি → সেটা Rollback (UNDO) করে।

এই পুরো প্রক্রিয়াকে বলে **Crash Recovery**।

### Diagram: WAL Working

```
Application Change Request
          │
          ▼
┌─────────────────────┐
│  1. WAL-এ প্রথমে      │  ◄── আগে Log এ লেখা হয় (ছোট, দ্রুত Write)
│     Log Entry লেখা    │
└──────────┬────────────┘
           │
           ▼
┌─────────────────────┐
│ 2. WAL Disk-এ Flush   │  ◄── Durability নিশ্চিত হয়
│    (fsync)            │
└──────────┬────────────┘
           │
           ▼
┌─────────────────────┐
│ 3. পরে Actual Data     │  ◄── আসল Data File পরিবর্তন
│    File Update হয়     │      (এটা পরে ব্যাচে করা যায়, ধীরে)
│    (Checkpoint সময়ে)  │
└─────────────────────┘
```

### PostgreSQL এ WAL

PostgreSQL-এ WAL ফাইল থাকে `pg_wal/` Directory-তে (আগে এটার নাম ছিল `pg_xlog`)। এই WAL ফাইলগুলো ব্যবহার করেই PostgreSQL এর বিখ্যাত Feature **PITR (Point-In-Time Recovery)** কাজ করে, যা আমরা Module 4 এবং Module 6-এ বিস্তারিত শিখবো।

```bash
# PostgreSQL WAL ফাইল দেখা
ls -la /var/lib/postgresql/14/main/pg_wal/
```

### MySQL এ সমতুল্য: Redo Log

MySQL-এর InnoDB Storage Engine-এ WAL এর সমতুল্য জিনিসটাকে বলা হয় **Redo Log** (ib_logfile0, ib_logfile1)।

---

## 2.8 Binlog (Binary Log)

### সংজ্ঞা

**Binlog (Binary Log)** হলো MySQL-এর একটা Log সিস্টেম, যেখানে Database-এ হওয়া সব **Data-changing Operation** (INSERT, UPDATE, DELETE, Schema Change) ক্রমানুসারে (Sequential) রেকর্ড করা হয়।

### WAL এবং Binlog এর পার্থক্য

এটা একটা খুবই গুরুত্বপূর্ণ পার্থক্য যা অনেকে গুলিয়ে ফেলে:

| বিষয় | WAL / Redo Log | Binlog |
|---|---|---|
| উদ্দেশ্য | Crash Recovery (Storage Engine Level) | Replication এবং Point-in-Time Recovery (Server Level) |
| Level | Storage Engine Internal (InnoDB) | MySQL Server Level (Engine-independent) |
| Format | Physical/Low-level Page changes | Logical/Statement বা Row-based changes |
| ব্যবহার | Server Restart এর পর Recovery | Replica Server-এ ডেটা পাঠানো, Backup এর সাথে মিলিয়ে PITR করা |

### Diagram: Binlog কীভাবে কাজ করে

```
┌────────────────┐
│  MySQL Server    │
│  (Primary/Master)│
└────────┬─────────┘
         │  প্রতিটা Write Operation
         ▼
┌────────────────────┐
│   Binary Log (Binlog) │  ◄── mysql-bin.000001, mysql-bin.000002...
└────────┬─────────────┘
         │
   ┌─────┴──────┐
   │             │
   ▼             ▼
Replica       Backup System
Server        (PITR এর জন্য
(Replication) Full Backup + Binlog
              একসাথে ব্যবহার করে)
```

### Binlog এর Format Types

| Format | ব্যাখ্যা |
|---|---|
| STATEMENT | আসল SQL Query রেকর্ড করে (যেমন `UPDATE users SET...`) |
| ROW | প্রতিটা Row-এর Before/After ডেটা রেকর্ড করে (বেশি নির্ভরযোগ্য) |
| MIXED | পরিস্থিতি অনুযায়ী দুইটাই ব্যবহার করে |

### Production Example: Binlog দেখা

```bash
# Binlog Enable করা আছে কিনা চেক করা
SHOW VARIABLES LIKE 'log_bin';

# বর্তমান Binlog ফাইলগুলোর তালিকা
SHOW BINARY LOGS;

# একটা Binlog ফাইলের ভেতরের Content দেখা
mysqlbinlog /var/lib/mysql/mysql-bin.000005
```

### কেন Binlog Backup এর জন্য গুরুত্বপূর্ণ?

Full Backup সাধারণত দিনে একবার নেওয়া হয় (যেমন রাত ২টায়)। কিন্তু যদি দুপুর ২টায় কোনো সমস্যা হয়, তাহলে শুধু আগের রাতের Backup দিয়ে দুপুর পর্যন্ত ডেটা Restore করা যাবে না। এখানেই Binlog কাজে আসে — Full Backup + তারপরের সব Binlog Apply করে ঠিক সেই মুহূর্ত পর্যন্ত (দুপুর ২টা পর্যন্ত) ডেটা Restore করা যায়। এই টেকনিককে বলে **PITR (Point-In-Time Recovery)**, যা আমরা Module 4-এ বিস্তারিত শিখবো।

---

## 2.9 এই সব কিছু কীভাবে Backup এর সাথে সম্পর্কিত (How This All Connects to Backup)

```
┌──────────────────────────────────────────────────────────┐
│                     সম্পূর্ণ চিত্র (Full Picture)            │
└──────────────────────────────────────────────────────────┘

Storage Engine  →  ঠিক করে ডেটা কীভাবে Disk-এ থাকবে
                    (Backup করার সময় কোন ফাইল Copy করতে হবে তা জানতে সাহায্য করে)

Transaction     →  ডেটার একটা Logical Unit
                    (Backup-কে অবশ্যই Transaction-Consistent হতে হবে)

Commit/Rollback →  ঠিক করে কোন ডেটা স্থায়ী, কোনটা না
                    (Backup শুধু Committed ডেটা নেবে)

ACID            →  ডেটার নির্ভরযোগ্যতার মূলনীতি
                    (Backup/Restore করার পরেও এই নীতি বজায় থাকতে হবে)

WAL/Redo Log    →  Crash Recovery এর ভিত্তি
                    (Hot Backup এবং PITR এর মূল ভিত্তি)

Binlog          →  পরিবর্তনের ইতিহাস
                    (Full Backup + Binlog = যেকোনো মুহূর্তে Restore করার ক্ষমতা)
```

এই Module 2 না বুঝলে Module 5 (MySQL Backup) এবং Module 6 (PostgreSQL Backup)-এ যে **Hot Backup**, **PITR**, **Replication-based Backup** শেখানো হবে, সেগুলোর "কেন এভাবে কাজ করে" এই প্রশ্নের উত্তর বোঝা যাবে না।

---

## 2.10 Best Practices

1. সবসময় জেনে রাখো তোমার Database কোন Storage Engine ব্যবহার করছে (InnoDB নাকি MyISAM ইত্যাদি)।
2. WAL/Binlog সবসময় Enable রাখো Production-এ, কখনো বন্ধ করো না (এটা PITR এর জন্য অপরিহার্য)।
3. Backup নেওয়ার সময় Transaction-Consistent Snapshot নিশ্চিত করো (যেমন MySQL-এ `--single-transaction` Flag ব্যবহার করা)।
4. Binlog/WAL ফাইলগুলোও নিয়মিত Backup/Archive করো, শুধু Full Backup যথেষ্ট না।
5. Disk Space Monitor করো — Binlog/WAL ফাইল জমতে জমতে Disk ভরে যেতে পারে যদি ঠিকমতো Purge/Archive না করা হয়।

---

## 2.11 Common Mistakes

| ভুল | সমস্যা |
|---|---|
| Binlog/WAL বন্ধ রাখা Performance এর জন্য | PITR করার সুযোগ হারিয়ে যায় |
| Transaction চলাকালীন সরাসরি Data File Copy করা | Corrupt/Inconsistent Backup তৈরি হয় |
| WAL/Binlog ফাইল Delete করে দেওয়া পুরনো ভেবে | PITR Chain ভেঙে যায়, শুধু Full Backup Point পর্যন্ত Restore করা যাবে |
| Storage Engine না জেনে Backup Method বেছে নেওয়া | ভুল Backup Strategy প্রয়োগ হতে পারে |

---

## 2.12 Interview Questions (Module 2)

1. **WAL এবং Binlog এর মধ্যে পার্থক্য কী?**
   *উত্তর:* WAL হলো Storage Engine Level-এর Log যা Crash Recovery এর জন্য ব্যবহৃত হয় (PostgreSQL/InnoDB-তে থাকে), আর Binlog হলো MySQL Server Level-এর Log যা Replication এবং PITR এর জন্য ব্যবহৃত হয়।

2. **ACID এর প্রতিটা অক্ষরের অর্থ কী এবং Backup-এর সাথে এর সম্পর্ক কী?**
   *উত্তর:* Atomicity, Consistency, Isolation, Durability। এগুলো নিশ্চিত করে Backup নেওয়ার সময় শুধু সঠিক, সম্পূর্ণ (Committed) ডেটা Include হবে এবং Restore এর পরেও Database একটা Valid অবস্থায় থাকবে।

3. **Transaction চলাকালীন যদি Server Crash করে, তাহলে কী হবে?**
   *উত্তর:* Database Restart হওয়ার সময় WAL/Redo Log পড়ে বুঝবে কোন Transaction Commit হয়েছিল (সেগুলো REDO করবে) এবং কোনটা হয়নি (সেগুলো Rollback/UNDO করবে) — এই প্রক্রিয়াকে Crash Recovery বলে।

4. **শুধু Full Backup নিয়ে কি একদম Present মুহূর্ত (এই মুহূর্ত) পর্যন্ত Restore করা সম্ভব?**
   *উত্তর:* না, Full Backup শুধু সেই মুহূর্ত পর্যন্তই ডেটা দেয় যখন সেটা নেওয়া হয়েছিল। Present মুহূর্ত পর্যন্ত Restore করতে হলে Full Backup + তার পরের সব WAL/Binlog Apply করতে হয় (PITR)।

5. **কেন সরাসরি Database ফাইল Copy করা (`cp` কমান্ড দিয়ে) একটা ভালো Backup Method না, যদি Database চলমান থাকে?**
   *উত্তর:* কারণ Database চলমান অবস্থায় ফাইলে কোনো Transaction-এর আংশিক পরিবর্তন থাকতে পারে, ফলে Copy করা ফাইলটা Inconsistent/Corrupt হতে পারে। এজন্য Hot Backup Tool (mysqldump, pg_dump, XtraBackup) ব্যবহার করা উচিত যেগুলো Consistency নিশ্চিত করে।

---

## 2.13 Summary (সারাংশ)

- **Storage Engine** ঠিক করে ডেটা Disk-এ কীভাবে থাকবে (InnoDB, MyISAM, WiredTiger ইত্যাদি)।
- ডেটা প্রথমে **Memory (RAM)**-তে থাকে, পরে **Disk**-এ Flush হয় — এই ব্যবধানটাই Backup Consistency-র মূল চ্যালেঞ্জ তৈরি করে।
- **Transaction** হলো একাধিক Operation-এর একটা Group যা Atomic ভাবে সফল বা ব্যর্থ হয়।
- **Commit** পরিবর্তন স্থায়ী করে, **Rollback** পরিবর্তন বাতিল করে।
- **ACID** (Atomicity, Consistency, Isolation, Durability) হলো Reliable Transaction-এর ৪টা মূলনীতি।
- **WAL/Redo Log** হলো Crash Recovery এর ভিত্তি — Data File পরিবর্তনের আগেই Log-এ লেখা হয়।
- **Binlog** হলো MySQL-এর পরিবর্তনের ইতিহাস, যা Replication এবং PITR এর জন্য ব্যবহৃত হয়।
- এই সব ধারণা একসাথে বুঝলেই Module 5-20 এ শেখানো বিভিন্ন Backup কৌশল (Hot Backup, PITR, Replication Backup) এর "কেন" এবং "কীভাবে" স্পষ্ট হবে।

---

## 2.14 Exercises & Assignments

### Exercise 1
তোমার MySQL Server-এ লগইন করে চেক করো Binlog Enable আছে কিনা:
```sql
SHOW VARIABLES LIKE 'log_bin';
SHOW BINARY LOGS;
```
যদি বন্ধ থাকে, `my.cnf` ফাইলে কী পরিবর্তন লাগবে তা খুঁজে বের করো (এখনই Enable করার দরকার নেই, শুধু জানো কীভাবে করতে হয়)।

### Exercise 2
PostgreSQL-এ `pg_wal` Directory খুঁজে বের করো এবং সেখানে কতগুলো WAL ফাইল আছে দেখো:
```bash
ls -la /var/lib/postgresql/*/main/pg_wal/
```

### Mini Project
একটা ছোট Diagram (কাগজে বা টুলে) আঁকো যেখানে দেখাবে — তোমার নিজের Application-এর একটা Order Placement Transaction (Order তৈরি + Inventory কমানো + Payment রেকর্ড) কীভাবে Atomic ভাবে Commit বা Rollback হবে। এই Diagram পরবর্তী Module-গুলোতে Backup Strategy ডিজাইন করার সময় কাজে লাগবে।

---

**পরবর্তী চ্যাপ্টার:** `03-Backup-Strategies.md` — এখানে আমরা সব ধরনের Backup Type (Full, Incremental, Differential, Logical, Physical, Hot, Cold ইত্যাদি) নিয়ে গভীরভাবে আলোচনা করবো।
