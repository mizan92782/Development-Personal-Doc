# Module 3: Backup Types (ব্যাকআপের প্রকারভেদ)

> **Level:** Beginner → Intermediate → Advanced
> **Prerequisite:** Module 1 (Introduction) ও Module 2 (Database Architecture Basics) সম্পন্ন থাকতে হবে।
> **কেন এই Module গুরুত্বপূর্ণ:** Production-এ সঠিক Backup Strategy বেছে নেওয়ার জন্য প্রতিটা Backup Type-এর Internal Working, সুবিধা-অসুবিধা এবং কখন ব্যবহার করা উচিত তা জানা অপরিহার্য।

---

## 3.0 Chapter Roadmap

```
3.1  Backup Type Classification (Overview)
3.2  Full Backup
3.3  Incremental Backup
3.4  Differential Backup
3.5  Logical Backup
3.6  Physical Backup
3.7  Cold Backup
3.8  Warm Backup
3.9  Hot Backup
3.10 Snapshot Backup
3.11 Online Backup
3.12 Offline Backup
3.13 Continuous Backup
3.14 Mirror Backup
3.15 Clone Backup
3.16 কোন Backup Type কখন ব্যবহার করবে (Decision Matrix)
3.17 Best Practices
3.18 Common Mistakes
3.19 Interview Questions
3.20 Summary
3.21 Exercises & Assignments
```

---

## 3.1 Backup Type Classification (Overview)

Backup Type-গুলোকে বোঝার আগে জানতে হবে এগুলোকে বিভিন্ন **Dimension (মাত্রা)** দিয়ে ভাগ করা যায়। একটা Backup একইসাথে একাধিক Category-তে পড়তে পারে।

```
Backup Classification Dimensions
│
├── Dimension 1: ডেটার পরিমাণ অনুযায়ী (By Amount of Data)
│      ├── Full Backup
│      ├── Incremental Backup
│      └── Differential Backup
│
├── Dimension 2: ডেটার ফরম্যাট অনুযায়ী (By Data Format)
│      ├── Logical Backup
│      └── Physical Backup
│
├── Dimension 3: Database এর অবস্থা অনুযায়ী (By DB State)
│      ├── Cold Backup (Offline)
│      ├── Warm Backup
│      └── Hot Backup (Online)
│
├── Dimension 4: পদ্ধতি অনুযায়ী (By Method)
│      ├── Snapshot Backup
│      ├── Mirror Backup
│      ├── Clone Backup
│      └── Continuous Backup
```

### Diagram: একটা Backup কীভাবে একাধিক Category তে পড়ে

```
উদাহরণ: mysqldump দিয়ে নেওয়া Backup

   ┌─────────────────────────────────────┐
   │  Type: Full Backup   (কতটা ডেটা)      │
   │  Type: Logical Backup (কোন ফরম্যাট)   │
   │  Type: Hot Backup     (DB অবস্থা)     │
   └─────────────────────────────────────┘
```

এই Overview মাথায় রাখলে পরের প্রতিটা Section পড়ার সময় বুঝতে সুবিধা হবে যে এগুলো আলাদা আলাদা "প্রতিযোগী" পদ্ধতি না, বরং একে অপরের সাথে **Combine** করে ব্যবহার করা হয়।

---

## 3.2 Full Backup (সম্পূর্ণ ব্যাকআপ)

### সংজ্ঞা

Full Backup মানে হলো Database-এর **সম্পূর্ণ ডেটা** একবারে Backup নেওয়া — প্রতিটা Table, প্রতিটা Row, সব কিছু।

### Internal Working

```
Database (Full Data)
        │
        ▼
Backup Tool পুরো Database Scan করে
        │
        ▼
সম্পূর্ণ ডেটা একটা ফাইলে Export/Copy হয়
        │
        ▼
Backup File (সম্পূর্ণ, Standalone — এটা একাই Restore করা যায়)
```

### বাস্তব উদাহরণ

```bash
# MySQL Full Backup
mysqldump -u root -p --all-databases > full_backup_2026_07_05.sql

# PostgreSQL Full Backup
pg_dump -U postgres mydb > full_backup_2026_07_05.sql
```

### সুবিধা (Advantages)

| সুবিধা | ব্যাখ্যা |
|---|---|
| সহজ Restore | একটা মাত্র ফাইল দিয়ে সম্পূর্ণ Restore করা যায় |
| Standalone | অন্য কোনো Backup-এর উপর নির্ভর করে না |
| সহজে বোঝা যায় | Complexity কম, নতুনদের জন্য সহজ |

### অসুবিধা (Disadvantages)

| অসুবিধা | ব্যাখ্যা |
|---|---|
| সময় বেশি লাগে | ডেটা বড় হলে Backup নিতে ঘণ্টার পর ঘণ্টা লাগতে পারে |
| Storage বেশি লাগে | প্রতিবার সম্পূর্ণ ডেটা Save করতে হয় |
| Server Load বেশি | পুরো Database Scan করতে হয় বলে CPU/Disk I/O বেশি ব্যবহার হয় |

### কখন ব্যবহার করবে

- ছোট থেকে মাঝারি সাইজের Database (কয়েক GB পর্যন্ত)।
- সপ্তাহে একবার বা প্রতিদিন রাতে (Low-traffic সময়ে)।
- Incremental/Differential Backup এর "Base" হিসেবে।

### কখন ব্যবহার করবে না

- খুব বড় Database (কয়েক TB), যেখানে প্রতিদিন Full Backup নেওয়া বাস্তবসম্মত না।
- খুব ঘন ঘন (প্রতি ঘণ্টায়) Backup দরকার হলে — তখন Incremental ভালো।

### Company Usage

Netflix, Amazon-এর মতো কোম্পানি সাধারণত সপ্তাহে একবার Full Backup নেয়, আর প্রতিদিন Incremental Backup নেয় — কারণ তাদের Database এর সাইজ এত বড় যে প্রতিদিন Full Backup নেওয়া Cost এবং Time এর দিক থেকে অসম্ভব।

---

## 3.3 Incremental Backup (ক্রমবর্ধমান ব্যাকআপ)

### সংজ্ঞা

Incremental Backup মানে হলো **শুধুমাত্র শেষ Backup (Full বা Incremental) এর পর থেকে যা পরিবর্তন হয়েছে**, সেটুকুই Backup নেওয়া।

### Internal Working

```
Sunday:    Full Backup           (সম্পূর্ণ ডেটা)
Monday:    Incremental Backup    (শুধু Sunday এর পর যা Change হয়েছে)
Tuesday:   Incremental Backup    (শুধু Monday এর পর যা Change হয়েছে)
Wednesday: Incremental Backup    (শুধু Tuesday এর পর যা Change হয়েছে)
```

```
Diagram:

Full (Sun) ──► Inc (Mon) ──► Inc (Tue) ──► Inc (Wed)
   │               │              │              │
   └───────────────┴──────────────┴──────────────┘
                       │
              Restore করতে হলে
        Full + সব Incremental লাগবে ক্রমানুসারে
```

### Restore Process (গুরুত্বপূর্ণ!)

Wednesday এর ডেটা Restore করতে হলে লাগবে:
```
Full (Sunday) → Incremental (Monday) → Incremental (Tuesday) → Incremental (Wednesday)
```
এই পুরো Chain-টা ক্রমানুসারে Apply করতে হবে। যদি মাঝখানের কোনো একটা Incremental Backup হারিয়ে যায় বা Corrupt হয়, তাহলে তার পরবর্তী সব Incremental Backup অকেজো হয়ে যাবে।

### সুবিধা

| সুবিধা | ব্যাখ্যা |
|---|---|
| Storage কম লাগে | শুধু পরিবর্তিত ডেটাই সংরক্ষণ হয় |
| দ্রুত Backup নেওয়া যায় | পুরো Database Scan করতে হয় না |
| Server Load কম | Resource Usage কম |

### অসুবিধা

| অসুবিধা | ব্যাখ্যা |
|---|---|
| Restore জটিল ও ধীর | পুরো Chain Apply করতে হয়, সময় বেশি লাগে |
| Chain ভাঙলে সমস্যা | একটা Backup নষ্ট হলে তার পরের সব Backup অকেজো |
| Tracking জটিল | কোন Backup কার পরে এসেছে তার হিসাব রাখতে হয় |

### কখন ব্যবহার করবে

- বড় Database, যেখানে প্রতিদিন Full Backup নেওয়া সময়সাপেক্ষ।
- Storage Cost কমাতে চাইলে।
- Change Rate কম হলে (দিনে অল্প পরিমাণ ডেটা পরিবর্তন হয়)।

### Company Usage

বড় Financial Institution বা E-commerce Company (Amazon-এর মতো) সাধারণত Full Backup (সাপ্তাহিক) + Incremental Backup (প্রতিদিন) এর Combination ব্যবহার করে, Storage Cost এবং Recovery Time এর মধ্যে Balance রাখার জন্য।

---

## 3.4 Differential Backup (পার্থক্যমূলক ব্যাকআপ)

### সংজ্ঞা

Differential Backup মানে হলো **শেষ Full Backup এর পর থেকে যত পরিবর্তন হয়েছে**, সব একসাথে Backup নেওয়া (Incremental এর মতো শুধু আগের Backup এর পর থেকে না, বরং সবসময় Full Backup Reference করে)।

### Internal Working ও Incremental এর সাথে পার্থক্য

```
Sunday:    Full Backup

Monday:    Differential Backup  (Sunday থেকে Monday পর্যন্ত সব Change)
Tuesday:   Differential Backup  (Sunday থেকে Tuesday পর্যন্ত সব Change — Monday এরটাও Include)
Wednesday: Differential Backup  (Sunday থেকে Wednesday পর্যন্ত সব Change — Mon+Tue দুটোই Include)
```

```
Diagram (Incremental vs Differential):

Incremental:
Full(Sun) → Inc(Mon: শুধু Sun-Mon) → Inc(Tue: শুধু Mon-Tue) → Inc(Wed: শুধু Tue-Wed)
Restore Wed = Full + Mon + Tue + Wed  (৪টা ফাইল লাগবে)

Differential:
Full(Sun) → Diff(Mon: Sun-Mon) → Diff(Tue: Sun-Tue, বড়) → Diff(Wed: Sun-Wed, আরও বড়)
Restore Wed = Full + Wed শুধু  (২টা ফাইল লাগবে, কারণ Wed এর Diff-এই সব আছে)
```

### Table: Incremental vs Differential

| বৈশিষ্ট্য | Incremental | Differential |
|---|---|---|
| Reference Point | শেষ Backup (যেটাই হোক) | সবসময় শেষ Full Backup |
| Backup সাইজ | ছোট (প্রতিবার) | ক্রমশ বড় হতে থাকে |
| Backup নেওয়ার গতি | দ্রুত | মাঝারি |
| Restore করার গতি | ধীর (অনেকগুলো ফাইল লাগে) | দ্রুত (মাত্র ২টা ফাইল লাগে) |
| জটিলতা | বেশি (Chain দীর্ঘ) | কম |
| Storage Usage | কম | বেশি (ধীরে ধীরে বাড়ে) |

### সুবিধা

- Incremental এর তুলনায় Restore সহজ ও দ্রুত (মাত্র Full + সর্বশেষ Differential লাগে)।
- Chain ভাঙার ঝুঁকি কম (শুধু ২টা ফাইলের উপর নির্ভরশীল)।

### অসুবিধা

- সময়ের সাথে সাথে Backup সাইজ বড় হতে থাকে (Full Backup এর কাছাকাছি চলে যেতে পারে সপ্তাহের শেষে)।
- Incremental এর চেয়ে বেশি Storage লাগে।

### কখন ব্যবহার করবে

- যখন Restore Speed গুরুত্বপূর্ণ (Incremental এর চেয়ে বেশি অগ্রাধিকার)।
- মাঝারি সাইজের Database, মাঝারি Change Rate।

---

## 3.5 Logical Backup (লজিক্যাল ব্যাকআপ)

### সংজ্ঞা

Logical Backup মানে হলো Database-এর ডেটা **SQL Statement বা অন্য কোনো Human-readable ফরম্যাটে** Export করা (যেমন `INSERT INTO` Statements বা CSV), Physical Disk ফাইল Copy করা না।

### Internal Working

```
Database
   │
   ▼
Backup Tool প্রতিটা Table থেকে ডেটা Read করে
   │
   ▼
SQL Statement আকারে রূপান্তর করে
   │
   ▼
Output File (.sql, .csv)

উদাহরণ Output:
CREATE TABLE users (...);
INSERT INTO users VALUES (1, 'Karim', 'karim@example.com');
INSERT INTO users VALUES (2, 'Rahim', 'rahim@example.com');
```

### বাস্তব উদাহরণ

```bash
mysqldump -u root -p mydb > logical_backup.sql
pg_dump -U postgres mydb > logical_backup.sql
```

### সুবিধা

| সুবিধা | ব্যাখ্যা |
|---|---|
| Portable | ভিন্ন Database Version বা এমনকি ভিন্ন Database System-এও (কিছুটা) ব্যবহারযোগ্য |
| Human-readable | ফাইল খুলে দেখা ও বোঝা যায় |
| Selective Restore | নির্দিষ্ট Table বা Row Restore করা সহজ |
| Version Independent | পুরনো Version থেকে নতুন Version-এ Migrate করার জন্য ভালো |

### অসুবিধা

| অসুবিধা | ব্যাখ্যা |
|---|---|
| ধীর গতি | বড় Database-এ Backup/Restore অনেক সময় নেয় |
| বেশি জায়গা লাগে (Uncompressed) | SQL Text Format Physical Format এর চেয়ে বড় হতে পারে |
| Index পুনরায় তৈরি করতে হয় | Restore এর সময় Index আবার Build করতে হয়, যা সময়সাপেক্ষ |

### কখন ব্যবহার করবে

- ছোট থেকে মাঝারি Database।
- Database Migration (এক Version থেকে আরেক Version, বা এক Cloud থেকে আরেক Cloud এ)।
- নির্দিষ্ট কিছু Table Backup/Restore করতে চাইলে।

---

## 3.6 Physical Backup (ফিজিক্যাল ব্যাকআপ)

### সংজ্ঞা

Physical Backup মানে হলো Database এর **Actual Disk ফাইল** (Data Files, Index Files) সরাসরি Copy করা — Byte-by-byte, কোনো SQL Conversion ছাড়াই।

### Internal Working

```
Database Data Directory
   │
   ▼
Backup Tool সরাসরি ফাইল Copy করে
   │
   ▼
Output: .ibd, .frm ফাইল অথবা পুরো Data Directory এর Copy

উদাহরণ:
/var/lib/mysql/mydb/users.ibd  →  /backup/users.ibd (হুবহু Copy)
```

### বাস্তব উদাহরণ

```bash
# MySQL Physical Backup with XtraBackup
xtrabackup --backup --target-dir=/backup/

# PostgreSQL Physical Backup
pg_basebackup -D /backup/ -U postgres -Fp -Xs -P
```

### সুবিধা

| সুবিধা | ব্যাখ্যা |
|---|---|
| দ্রুত Backup ও Restore | Byte-copy হওয়ায় SQL Parse করতে হয় না |
| বড় Database এর জন্য উপযুক্ত | TB সাইজের Database এও কার্যকর |
| Index পুনরায় তৈরি করার দরকার নেই | Restore এর পর সাথে সাথেই ব্যবহারযোগ্য |

### অসুবিধা

| অসুবিধা | ব্যাখ্যা |
|---|---|
| Version-dependent | সাধারণত একই Database Version-এই Restore করতে হয় |
| কম Portable | ভিন্ন OS/Architecture-এ কাজ নাও করতে পারে |
| Human-readable না | ফাইল খুলে বোঝা যায় না, Debug করা কঠিন |

### Table: Logical vs Physical Backup

| বৈশিষ্ট্য | Logical Backup | Physical Backup |
|---|---|---|
| ফরম্যাট | SQL/Text | Binary Disk Files |
| গতি (বড় DB-তে) | ধীর | দ্রুত |
| Portability | বেশি | কম |
| Selective Restore | সহজ | কঠিন |
| Database Size উপযোগিতা | ছোট-মাঝারি | মাঝারি-বড় (TB Scale) |
| Tool উদাহরণ | mysqldump, pg_dump | XtraBackup, pg_basebackup |

### Company Usage

Uber, Netflix এর মতো কোম্পানি যাদের TB Scale-এর Database আছে, তারা মূলত Physical Backup Tool (XtraBackup, pg_basebackup) ব্যবহার করে কারণ Logical Backup দিয়ে এত বড় Database Backup/Restore করা ব্যবহারিকভাবে সম্ভব না (সময় লাগবে অনেক ঘণ্টা থেকে দিন)।

---

## 3.7 Cold Backup (কোল্ড ব্যাকআপ)

### সংজ্ঞা

Cold Backup মানে হলো Database **সম্পূর্ণ বন্ধ (Shutdown)** করে তারপর Backup নেওয়া।

### Internal Working

```
1. Database Server বন্ধ করা (mysqld stop / pg_ctl stop)
2. Data Directory সম্পূর্ণ Copy করা
3. Database আবার চালু করা (mysqld start / pg_ctl start)
```

### সুবিধা

- সবচেয়ে **নিরাপদ ও Consistent** Backup, কারণ Database বন্ধ থাকায় কোনো Active Write হয় না।
- Corruption এর ঝুঁকি সবচেয়ে কম।

### অসুবিধা

- Database বন্ধ থাকার সময় User Application ব্যবহার করতে পারবে না — **Downtime** হয়।
- Production-এ 24/7 Available System-এর জন্য অগ্রহণযোগ্য।

### কখন ব্যবহার করবে

- Development/Test Environment।
- এমন Internal Tool/System যেখানে নির্দিষ্ট Maintenance Window (যেমন রাত ৩টা থেকে ৪টা) নেওয়া সম্ভব।
- ছোট Company/Personal Project যেখানে সামান্য Downtime সমস্যা না।

---

## 3.8 Warm Backup (ওয়ার্ম ব্যাকআপ)

### সংজ্ঞা

Warm Backup মানে হলো Database **আংশিকভাবে সীমিত (Limited/Read-only Mode)** রেখে Backup নেওয়া — সম্পূর্ণ বন্ধ না করেই।

### Internal Working

```
1. Database কে Read-Only Mode এ রাখা (Write বন্ধ, Read চলবে)
2. Backup নেওয়া
3. আবার Normal (Read-Write) Mode এ ফেরত আনা
```

### সুবিধা

- সম্পূর্ণ Downtime এর দরকার নেই, User Read Operation চালাতে পারে।
- Cold Backup এর চেয়ে বেশি Availability।

### অসুবিধা

- Write Operation চলাকালীন সময়ে Block হয়ে যায়, যা User Experience খারাপ করতে পারে।
- এখনও কিছুটা Service Degradation হয়।

### কখন ব্যবহার করবে

- এমন সিস্টেম যেখানে সাময়িক Write বন্ধ রাখা মেনে নেওয়া যায় (যেমন Reporting/Analytics System)।

---

## 3.9 Hot Backup (হট ব্যাকআপ)

### সংজ্ঞা

Hot Backup মানে হলো Database **সম্পূর্ণ Running অবস্থায়** (Read এবং Write উভয়ই চলমান), কোনো Downtime ছাড়াই Backup নেওয়া।

### Internal Working

Hot Backup সম্ভব হয় Module 2-এ শেখা **WAL/Redo Log** এর কারণে। Backup Tool একটা Consistent Snapshot নেয় এবং সাথে সাথে যে Transaction চলমান থাকে সেগুলোর WAL Record Track করে, যাতে Backup শেষ হওয়ার পর সেই WAL ব্যবহার করে একটা সম্পূর্ণ Consistent State তৈরি করা যায়।

```
Diagram: Hot Backup Process

Running Database (Read+Write চলছে)
        │
        ▼
Backup Tool Snapshot নেয় (একটা নির্দিষ্ট মুহূর্তের)
        │
        ▼
সাথে সাথে চলমান WAL/Binlog Record ট্র্যাক করা হয়
        │
        ▼
Backup সম্পূর্ণ হলে, WAL Apply করে Consistency নিশ্চিত করা হয়
        │
        ▼
Consistent Backup তৈরি — Database কখনো বন্ধ হয়নি
```

### বাস্তব উদাহরণ

```bash
# MySQL Hot Backup (Consistent Snapshot দিয়ে)
mysqldump -u root -p --single-transaction --routines --triggers mydb > hot_backup.sql

# XtraBackup দিয়ে Hot Physical Backup
xtrabackup --backup --target-dir=/backup/
```

### সুবিধা

| সুবিধা | ব্যাখ্যা |
|---|---|
| Zero Downtime | Application সবসময় চালু থাকে |
| Production-এ ব্যবহারযোগ্য | 24/7 System-এর জন্য একমাত্র বাস্তবসম্মত সমাধান |

### অসুবিধা

| অসুবিধা | ব্যাখ্যা |
|---|---|
| জটিলতা বেশি | Consistency নিশ্চিত করার জন্য Advanced Logic দরকার |
| সামান্য Performance Impact | Backup চলাকালীন Disk I/O এবং CPU Usage বাড়ে |
| ভুল হলে Inconsistent Backup হওয়ার ঝুঁকি | সঠিক Tool/Flag ব্যবহার না করলে সমস্যা হতে পারে |

### Company Usage

প্রায় সব বড় Production System (Google, Amazon, Netflix, Uber) Hot Backup ব্যবহার করে, কারণ তাদের System Downtime নেওয়ার সুযোগ নেই — এটাই Industry Standard।

---

## 3.10 Snapshot Backup (স্ন্যাপশট ব্যাকআপ)

### সংজ্ঞা

Snapshot Backup মানে হলো একটা নির্দিষ্ট মুহূর্তে সম্পূর্ণ Disk/Volume-এর একটা "ছবি" (Point-in-time Image) তোলা, সাধারণত Storage System/Cloud Provider এর সুবিধা ব্যবহার করে।

### Internal Working: Copy-on-Write (COW) Technique

```
Snapshot নেওয়ার মুহূর্ত (T0)
        │
        ▼
Original Data ব্লক-গুলোর Reference/Pointer রাখা হয় (কপি করা হয় না তখনই)
        │
        ▼
এরপর যদি কোনো Data Block পরিবর্তন হয় (Write হয়):
        │
        ▼
পুরনো Block-টা Snapshot এর জন্য সংরক্ষণ করা হয় (Copy-on-Write)
নতুন পরিবর্তন Original Location-এ লেখা হয়
```

```
Diagram:

Time T0 (Snapshot নেওয়া হলো):
Block A [1] ─┐
Block B [2] ─┼── Snapshot Reference তৈরি (কোনো Actual Copy হয়নি এখনো)
Block C [3] ─┘

Time T1 (Block B পরিবর্তন হলো):
Block A [1]              (অপরিবর্তিত)
Block B [2] → [2-old সংরক্ষিত, নতুন মান 2'] (Copy-on-Write ঘটলো)
Block C [3]              (অপরিবর্তিত)

Snapshot এখনো দেখাবে: A=1, B=2 (পুরনো), C=3
Live Data দেখাবে:      A=1, B=2', C=3
```

### সুবিধা

| সুবিধা | ব্যাখ্যা |
|---|---|
| খুব দ্রুত | কয়েক সেকেন্ডে Snapshot নেওয়া যায় (TB সাইজের Volume এও) |
| কম Storage Overhead | শুরুতে শুধু Reference রাখা হয়, Actual Data Copy না |
| Cloud-Native | AWS EBS Snapshot, GCP Persistent Disk Snapshot এর মতো Native Feature |

### অসুবিধা

| অসুবিধা | ব্যাখ্যা |
|---|---|
| Storage-Dependent | নির্দিষ্ট Storage System/Cloud Provider-এর উপর নির্ভরশীল |
| Application-level Consistency নিশ্চিত না হতে পারে | Database-Aware না হলে Consistency সমস্যা হতে পারে |
| Vendor Lock-in ঝুঁকি | এক Cloud Provider এর Snapshot আরেকটাতে সরাসরি ব্যবহার করা কঠিন |

### বাস্তব উদাহরণ

```bash
# AWS EBS Snapshot তৈরি করা (AWS CLI)
aws ec2 create-snapshot --volume-id vol-0abcd1234efgh5678 \
    --description "Database backup snapshot 2026-07-05"
```

### কখন ব্যবহার করবে

- Cloud Infrastructure (AWS, GCP, Azure) ব্যবহার করলে।
- খুব বড় Database (TB Scale) যেখানে Logical/Physical Backup সময়সাপেক্ষ।
- Disaster Recovery Strategy এর একটা অংশ হিসেবে।

---

## 3.11 Online Backup (অনলাইন ব্যাকআপ)

### সংজ্ঞা

Online Backup বলতে বোঝায় Database চালু (Online) থাকা অবস্থায় Backup নেওয়া। এটা মূলত **Hot Backup** এর সমার্থক শব্দ, তবে কিছু Context এ এটা বোঝাতেও ব্যবহার হয় যে Backup Remote/Cloud Location-এ (Internet এর মাধ্যমে) পাঠানো হচ্ছে।

### দুইটা অর্থ পার্থক্য করা জরুরি

| প্রসঙ্গ | অর্থ |
|---|---|
| Database State প্রসঙ্গে | Hot Backup এর সমার্থক (DB Running অবস্থায় Backup) |
| Storage Location প্রসঙ্গে | Cloud/Internet-based Backup Storage (যেমন AWS S3-তে সরাসরি Upload) |

### বাস্তব উদাহরণ

```bash
# Backup নিয়ে সরাসরি Cloud এ Upload করা (Online Backup Storage অর্থে)
mysqldump -u root -p mydb | gzip | aws s3 cp - s3://my-backup-bucket/backup_$(date +%F).sql.gz
```

---

## 3.12 Offline Backup (অফলাইন ব্যাকআপ)

### সংজ্ঞা

Offline Backup বলতে বোঝায় Database বন্ধ থাকা অবস্থায় Backup নেওয়া (Cold Backup এর সমার্থক), অথবা Backup ফাইল এমন একটা জায়গায় সংরক্ষণ করা যেখানে Internet/Network Access নেই (Air-gapped Storage)।

### Air-gapped Backup — Ransomware Protection এ গুরুত্বপূর্ণ

```
Diagram: Air-gapped Backup Concept

┌────────────────┐        ┌──────────────────┐
│  Production DB   │        │  Online Backup     │
│  (Internet-      │───────►│  (Cloud/Network    │
│   connected)      │        │   Connected)        │
└────────────────┘        └──────────────────┘
                                    │
                        নিয়মিত সময় অন্তর Copy
                                    │
                                    ▼
                          ┌──────────────────┐
                          │  Offline/Air-gapped │
                          │  Backup (Tape/      │
                          │  Disconnected Disk)  │
                          └──────────────────┘
                          (Ransomware এই Backup
                           Access/Encrypt করতে পারবে না)
```

### কখন ব্যবহার করবে

- Ransomware/Cyber Attack থেকে সুরক্ষার জন্য শেষ ভরসা হিসেবে।
- Legal/Compliance প্রয়োজনে দীর্ঘমেয়াদী সংরক্ষণ (Tape Backup)।

---

## 3.13 Continuous Backup (ক্রমাগত ব্যাকআপ)

### সংজ্ঞা

Continuous Backup (একে **Continuous Data Protection - CDP** ও বলা হয়) মানে হলো প্রতিটা পরিবর্তন (প্রতিটা Transaction) সাথে সাথে Backup Location-এ Replicate/Log করা, যাতে যেকোনো মুহূর্তে Restore করা যায় (Near Zero Data Loss)।

### Internal Working

```
প্রতিটা Write Operation
        │
        ▼
তাৎক্ষণিকভাবে WAL/Binlog Stream করা হয় Backup Server-এ
        │
        ▼
Backup Server প্রায় Real-time এ Primary এর সাথে Sync থাকে
```

### সুবিধা

- সবচেয়ে কম Data Loss (RPO প্রায় শূন্যের কাছাকাছি — RPO নিয়ে বিস্তারিত Module 4-এ)।
- যেকোনো মুহূর্ত পর্যন্ত Restore সম্ভব।

### অসুবিধা

- Infrastructure জটিল ও ব্যয়বহুল।
- Continuous Network/Resource Usage প্রয়োজন।

### কখন ব্যবহার করবে

- Financial System, Banking, Stock Trading Platform — যেখানে এক সেকেন্ডের ডেটাও হারানো চলবে না।

---

## 3.14 Mirror Backup (মিরর ব্যাকআপ)

### সংজ্ঞা

Mirror Backup মানে হলো মূল ডেটার একটা **হুবহু, Real-time Copy** রাখা আরেকটা Location এ — কোনো Compression বা History (পুরনো Version) ছাড়াই।

### Internal Working

```
Primary Database  ──── Real-time Sync ────►  Mirror Copy
     (A=1, B=2)                                 (A=1, B=2)

যদি Primary তে A=1 থেকে A=99 হয়ে যায় (এমনকি ভুলে),
Mirror ও সাথে সাথে A=99 হয়ে যাবে!
```

### গুরুত্বপূর্ণ সতর্কতা

> Mirror Backup **Human Error থেকে সুরক্ষা দেয় না!** কারণ ভুল পরিবর্তনও সাথে সাথে Mirror-এ চলে যায়। এটা মূলত **High Availability** এর জন্য ব্যবহার হয়, Backup Strategy হিসেবে এককভাবে যথেষ্ট না (এটা Module 1-এ আলোচিত RAID এর মতো একই সমস্যা)।

### সুবিধা

- তাৎক্ষণিক Failover সম্ভব (Primary নষ্ট হলে সাথে সাথে Mirror ব্যবহার করা যায়)।

### অসুবিধা

- Logical Error/Corruption থেকে সুরক্ষা দেয় না।
- History রাখে না (শুধু সর্বশেষ State)।

---

## 3.15 Clone Backup (ক্লোন ব্যাকআপ)

### সংজ্ঞা

Clone Backup মানে হলো একটা নির্দিষ্ট মুহূর্তে সম্পূর্ণ Database-এর একটা স্বতন্ত্র (Independent) Copy তৈরি করা, যা মূল Database থেকে সম্পূর্ণ বিচ্ছিন্ন এবং আলাদাভাবে ব্যবহারযোগ্য (যেমন Testing/Staging Environment তৈরির জন্য)।

### Mirror vs Clone পার্থক্য

| বৈশিষ্ট্য | Mirror | Clone |
|---|---|---|
| Sync | Real-time, ক্রমাগত | একবার নেওয়া, স্বতন্ত্র |
| উদ্দেশ্য | High Availability/Failover | Testing, Development, নির্দিষ্ট মুহূর্তের Copy |
| Primary এর সাথে সংযোগ | সবসময় সংযুক্ত | বিচ্ছিন্ন, স্বাধীন |

### বাস্তব উদাহরণ

```bash
# MySQL Clone Plugin ব্যবহার করে (MySQL 8.0+)
CLONE INSTANCE FROM 'user'@'source_host':3306
IDENTIFIED BY 'password';
```

### কখন ব্যবহার করবে

- নতুন Developer Environment তৈরি করতে Production এর মতো ডেটা লাগবে।
- Staging Server-এ Real Data দিয়ে Testing করতে চাইলে (Sensitive Data Mask করে)।

---

## 3.16 কোন Backup Type কখন ব্যবহার করবে (Decision Matrix)

| পরিস্থিতি | সুপারিশকৃত Backup Type |
|---|---|
| ছোট Database, Downtime সমস্যা না | Cold Backup + Full Backup |
| 24/7 Production System | Hot Backup + Physical/Logical Backup |
| খুব বড় Database (TB Scale) | Physical Backup + Snapshot Backup |
| Storage Cost কমাতে চাই | Full (সাপ্তাহিক) + Incremental (দৈনিক) |
| দ্রুত Restore চাই | Full + Differential |
| Ransomware Protection দরকার | Offline/Air-gapped Backup |
| Financial/Trading System | Continuous Backup (CDP) |
| High Availability দরকার | Mirror Backup (কিন্তু আলাদা Backup ও রাখতে হবে) |
| Dev/Test Environment তৈরি | Clone Backup |
| Cloud Infrastructure (AWS/GCP) | Snapshot Backup |

---

## 3.17 Best Practices

1. শুধু একটা Backup Type-এর উপর নির্ভর করো না — **Combination** ব্যবহার করো (যেমন Full + Incremental + Snapshot)।
2. Production System-এ সবসময় **Hot Backup** পদ্ধতি বেছে নাও, Downtime এড়াতে।
3. Ransomware Protection এর জন্য অবশ্যই একটা **Offline/Air-gapped Copy** রাখো।
4. Restore Time (RTO) হিসাব করে বুঝে নাও Incremental নাকি Differential তোমার জন্য ভালো।
5. Mirror কে কখনো একমাত্র Backup Strategy হিসেবে ব্যবহার করো না।

---

## 3.18 Common Mistakes

| ভুল | সমস্যা |
|---|---|
| শুধু Mirror/Replication কে Backup মনে করা | Human Error/Logical Corruption থেকে সুরক্ষা নেই |
| Incremental Chain এর মাঝের একটা Backup Delete করে ফেলা | পুরো Chain অকেজো হয়ে যায় |
| Cold Backup Production-এ ব্যবহার করা | অপ্রয়োজনীয় Downtime তৈরি করে |
| Snapshot কে Database-Consistent মনে করা যাচাই ছাড়াই | Restore করার সময় Inconsistent ডেটা পাওয়া যেতে পারে |

---

## 3.19 Interview Questions (Module 3)

1. **Incremental এবং Differential Backup এর মধ্যে মূল পার্থক্য কী?**
   *উত্তর:* Incremental Backup শুধু শেষ Backup (Full বা Incremental) থেকে পরিবর্তিত ডেটা নেয়, আর Differential Backup সবসময় শেষ Full Backup থেকে পরিবর্তিত সব ডেটা নেয় — ফলে Differential Restore এর জন্য মাত্র ২টা ফাইল (Full + সর্বশেষ Differential) লাগে, কিন্তু Incremental এর জন্য পুরো Chain লাগে।

2. **Logical এবং Physical Backup এর মধ্যে পার্থক্য কী?**
   *উত্তর:* Logical Backup SQL Statement আকারে ডেটা Export করে (mysqldump/pg_dump), যা Portable কিন্তু ধীর। Physical Backup সরাসরি Disk ফাইল Copy করে (XtraBackup/pg_basebackup), যা দ্রুত কিন্তু Version-dependent।

3. **Hot Backup কীভাবে সম্ভব হয় Database চলমান অবস্থাতেও?**
   *উত্তর:* WAL/Redo Log এবং Consistent Snapshot Technique ব্যবহার করে — Backup Tool একটা নির্দিষ্ট মুহূর্তের Snapshot নেয় এবং চলমান Transaction গুলোর Log Track করে, পরে সেই Log Apply করে একটা Consistent State তৈরি করে।

4. **Mirror Backup কেন Human Error থেকে সুরক্ষা দেয় না?**
   *উত্তর:* কারণ Mirror Real-time এ Primary এর সব পরিবর্তন Copy করে, ভুল পরিবর্তনও (যেমন ভুল DELETE) সাথে সাথে Mirror-এ চলে যায়।

5. **Snapshot Backup এর Copy-on-Write পদ্ধতি কীভাবে কাজ করে?**
   *উত্তর:* Snapshot নেওয়ার সময় Data Block-এর Reference রাখা হয়, Actual Copy তখনই হয় না। পরে কোনো Block পরিবর্তন হলে, পুরনো Block-টা Snapshot-এর জন্য সংরক্ষণ করে নতুন পরিবর্তন Original Location-এ লেখা হয়।

6. **Continuous Backup (CDP) কোন ধরনের System-এ প্রয়োজন হয়?**
   *উত্তর:* Financial/Banking/Stock Trading System-এর মতো জায়গায়, যেখানে RPO প্রায় শূন্য হতে হবে (এক সেকেন্ডের ডেটাও হারানো চলবে না)।

---

## 3.20 Summary (সারাংশ)

- Backup Type-গুলোকে বিভিন্ন Dimension এ ভাগ করা যায়: ডেটার পরিমাণ (Full/Incremental/Differential), ফরম্যাট (Logical/Physical), Database State (Cold/Warm/Hot), এবং পদ্ধতি (Snapshot/Mirror/Clone/Continuous)।
- **Full Backup** সহজ কিন্তু সময়/জায়গা বেশি লাগে। **Incremental** জায়গা বাঁচায় কিন্তু Restore জটিল। **Differential** এর মাঝামাঝি সমাধান।
- **Logical Backup** Portable কিন্তু ধীর, **Physical Backup** দ্রুত কিন্তু Version-dependent।
- **Cold Backup** সবচেয়ে নিরাপদ কিন্তু Downtime লাগে, **Hot Backup** Production-এর জন্য Standard।
- **Snapshot** Cloud Infrastructure-এ দ্রুত ও সাশ্রয়ী, কিন্তু Application Consistency নিশ্চিত করতে হয়।
- **Mirror** ও **RAID** কখনোই একমাত্র Backup Strategy হতে পারে না — এগুলো High Availability এর জন্য, Backup এর বিকল্প না।
- Production-এ সবসময় একাধিক Backup Type একসাথে (Combination) ব্যবহার করা হয়, একটার উপর নির্ভর করা হয় না।

---

## 3.21 Exercises & Assignments

### Exercise 1
তোমার Local MySQL Database-এ একটা Full Backup নাও (`mysqldump`), তারপর কিছু ডেটা পরিবর্তন করে একটা Incremental ভিত্তিক Backup Chain কল্পনা করে দেখো — Binlog ব্যবহার করে (`SHOW BINARY LOGS`) বুঝার চেষ্টা করো কীভাবে এটা কাজ করবে।

### Exercise 2
একটা Table বানাও যেখানে তুমি তোমার নিজের প্রজেক্টের জন্য প্রতিটা Backup Type (Full, Incremental, Snapshot ইত্যাদি) এর জন্য "হ্যাঁ/না" লিখবে — এটা কি তোমার প্রজেক্টে ব্যবহারযোগ্য?

### Mini Project
একটা Backup Strategy ডকুমেন্ট লেখো যেখানে থাকবে:
- তোমার Database এর সাইজ ও Change Rate অনুমান
- কোন Backup Type/Combination ব্যবহার করবে এবং কেন
- Backup নেওয়ার সময়সূচী (Full কবে, Incremental কবে)
- Storage কোথায় রাখবে (Local + Cloud + Offline)

---

**পরবর্তী চ্যাপ্টার:** `04-Recovery-Concepts.md` — এখানে আমরা শিখবো Restore, Recovery, PITR, Crash Recovery, Disaster Recovery, RPO এবং RTO সম্পর্কে বিস্তারিতভাবে।
