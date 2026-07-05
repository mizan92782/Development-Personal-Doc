# Module 1: Introduction to Database Backup (ডেটাবেজ ব্যাকআপের ভূমিকা)

> **Level:** Beginner → Intermediate
> **Prerequisite:** কোনো প্রকার প্রাথমিক জ্ঞান লাগবে না। ধরে নেওয়া হচ্ছে তুমি Backup সম্পর্কে কিছুই জানো না।

---

## 1.0 এই চ্যাপ্টারে কী শিখবে (Chapter Roadmap)

```
1.1 Backup আসলে কী?
1.2 কেন Backup দরকার (Why Backup)?
1.3 Data Loss কী এবং কীভাবে হয়
1.4 Disaster (দুর্যোগ) - Database এর প্রেক্ষাপটে
1.5 Corruption (ডেটা নষ্ট হয়ে যাওয়া)
1.6 Human Error (মানুষের ভুল)
1.7 Hardware Failure (হার্ডওয়্যার নষ্ট হওয়া)
1.8 Ransomware (মুক্তিপণ ম্যালওয়্যার)
1.9 Backup vs Restore পার্থক্য
1.10 Real Production Case Studies
1.11 Best Practices
1.12 Common Mistakes
1.13 Interview Questions
1.14 Summary
1.15 Exercises & Assignments
```

---

## 1.1 Backup আসলে কী? (What is Backup?)

### সংজ্ঞা (Definition)

**Backup** মানে হলো তোমার Database-এর ডেটার একটা **কপি (copy)** তৈরি করে রাখা, যাতে মূল ডেটা কোনো কারণে হারিয়ে গেলে, নষ্ট হয়ে গেলে বা Corrupt (নষ্ট) হয়ে গেলে সেই কপি থেকে ডেটা আবার ফিরিয়ে আনা (Restore) যায়।

সহজ ভাষায় বলতে গেলে —

> Backup হলো "তোমার ডেটার Insurance (বীমা)"।

তুমি যেমন গাড়ির Insurance করো এই ভেবে যে "যদি অ্যাক্সিডেন্ট হয়, তাহলে ক্ষতিপূরণ পাবো" — ঠিক তেমনই Database Backup রাখা হয় এই ভেবে যে "যদি ডেটা হারিয়ে যায়, তাহলে ফিরিয়ে আনতে পারবো।"

### Backup এর প্রযুক্তিগত সংজ্ঞা (Technical Definition)

Backup হলো একটি **point-in-time (নির্দিষ্ট সময়ের)** ডেটার Snapshot (ছবি) যা মূল Production Database থেকে আলাদা একটি Storage Location-এ সংরক্ষণ করা হয় — যাতে:

1. মূল Database নষ্ট হলেও Backup অক্ষত থাকে।
2. প্রয়োজনে সেই Backup থেকে Database-কে আগের অবস্থায় ফিরিয়ে আনা যায় (Restore)।

### Diagram: Backup এর মূল ধারণা

```
                 ┌─────────────────────┐
                 │   Production DB      │
                 │  (Live, Active Data)  │
                 └──────────┬───────────┘
                            │
                     Backup Process
                            │
                            ▼
                 ┌─────────────────────┐
                 │   Backup File/Copy    │
                 │  (Snapshot at Time T) │
                 └──────────┬───────────┘
                            │
                    Stored Separately
                            │
                            ▼
              ┌───────────────────────────┐
              │  Backup Storage             │
              │  (Local Disk / Cloud / Tape)│
              └───────────────────────────┘
```

### বাস্তব উদাহরণ (Real World Example)

ধরো তুমি একটি e-commerce Application বানিয়েছো Django দিয়ে, যেখানে:

- User রেজিস্ট্রেশন করে
- Order দেয়
- Payment করে

এখন যদি হঠাৎ MySQL Server-এর Hard Disk নষ্ট হয়ে যায়, তাহলে তোমার সব User, Order, Payment তথ্য মুহূর্তেই হারিয়ে যাবে। কিন্তু যদি তোমার প্রতিদিন রাত ২টায় একটা Automated Backup System থাকে (যেমন mysqldump দিয়ে), তাহলে তুমি সেই আগের রাতের Backup থেকে ডেটা Restore করতে পারবে।

---

## 1.2 কেন Backup দরকার? (Why Backup?)

### প্রধান কারণসমূহ

| ক্রম | কারণ | ব্যাখ্যা |
|---|---|---|
| 1 | Data Loss প্রতিরোধ | হার্ডওয়্যার/সফটওয়্যার সমস্যার কারণে ডেটা হারিয়ে যাওয়া থেকে সুরক্ষা |
| 2 | Human Error Recovery | ভুলে `DROP TABLE` বা `DELETE FROM` চালালে ফিরিয়ে আনার উপায় |
| 3 | Disaster Recovery | আগুন, বন্যা, Data Center ধ্বংস হলে ব্যবসা চালু রাখা |
| 4 | Compliance & Legal | অনেক দেশে আইন অনুযায়ী নির্দিষ্ট সময় পর্যন্ত ডেটা রাখা বাধ্যতামূলক (যেমন GDPR, HIPAA) |
| 5 | Business Continuity | ব্যবসা যেন থেমে না যায় |
| 6 | Testing/Staging Environment তৈরি | Production ডেটার কপি নিয়ে Test করা |
| 7 | Migration | এক Server থেকে আরেক Server-এ ডেটা সরানো |
| 8 | Ransomware Protection | হ্যাকার ডেটা Encrypt করে ফেললে Backup থেকে উদ্ধার |

### কেন এটা এত গুরুত্বপূর্ণ? (The "WHY" behind Backup)

একটা কথা মনে রাখবে:

> **"It's not a matter of IF you will lose data, it's a matter of WHEN."**
> (তুমি ডেটা হারাবে কিনা এটা প্রশ্ন না, কবে হারাবে সেটাই প্রশ্ন।)

কোম্পানিগুলোতে যারা Senior Backend Engineer বা DBA হিসেবে কাজ করে, তারা জানে যে সিস্টেম যতই ভালো হোক না কেন, একদিন না একদিন কোনো না কোনো সমস্যা হবেই — Server Crash, Disk Failure, Bug, Human Mistake, Cyber Attack। তাই Backup একটা **Optional Feature না, বরং Mandatory Requirement**।

### Cost of NOT having Backup (Backup না থাকার মূল্য)

বাস্তব একটা উদাহরণ দিই:

**GitLab.com Incident (2017)** — GitLab একটি Production Database ভুলবশত Delete করে ফেলেছিল, এবং তাদের ৫টা Backup Strategy-র মধ্যে ৪টাই কাজ করেনি (কারণ সেগুলো ঠিকমতো Test করা হয়নি)। শেষমেশ তারা প্রায় ৬ ঘণ্টার ডেটা হারিয়ে ফেলে। এটা থেকে Industry-level শিক্ষা হলো:

> শুধু Backup রাখলেই হবে না — Backup **Restore করে পরীক্ষা** করতে হবে নিয়মিত।

---

## 1.3 Data Loss কী এবং কীভাবে হয়? (What is Data Loss & How it Happens)

### Data Loss এর সংজ্ঞা

Data Loss মানে হলো তোমার Database-এ থাকা তথ্য আংশিক বা সম্পূর্ণভাবে হারিয়ে যাওয়া, যা স্বাভাবিক উপায়ে আর ফিরিয়ে আনা যায় না।

### Data Loss এর কারণসমূহ

```
Data Loss Causes
│
├── 1. Hardware Failure (হার্ডওয়্যার সমস্যা)
│      ├── Hard Disk Crash
│      ├── SSD Failure
│      ├── RAM Corruption
│      └── Power Supply Failure
│
├── 2. Software Bugs (সফটওয়্যার বাগ)
│      ├── Database Engine Bug
│      ├── Application Logic Error
│      └── ORM Migration Bug
│
├── 3. Human Error (মানুষের ভুল)
│      ├── Wrong DELETE/UPDATE Query
│      ├── Wrong Server-এ Deploy
│      └── Accidental Table Drop
│
├── 4. Malicious Attack (দুর্বৃত্তায়ন আক্রমণ)
│      ├── Ransomware
│      ├── SQL Injection
│      └── Insider Threat
│
├── 5. Natural Disaster (প্রাকৃতিক দুর্যোগ)
│      ├── Fire
│      ├── Flood
│      └── Earthquake
│
└── 6. Power Failure / Sudden Shutdown
       └── Unclean Shutdown → Data Corruption
```

### Internal Working: ডেটা কীভাবে আসলে Disk-এ থাকে?

এটা বোঝা জরুরি কারণ Backup Strategy বুঝতে হলে জানতে হবে ডেটা আসলে কীভাবে সংরক্ষিত থাকে।

```
Application (Django/Spring Boot)
        │
        ▼
Database Engine (MySQL/PostgreSQL)
        │
        ▼
Memory Buffer (RAM - Cache)
        │
        ▼
Disk Write (Physical Storage - .ibd, .frm ফাইল)
```

যখন তুমি `INSERT INTO users VALUES(...)` চালাও, তখন ডেটা প্রথমে RAM-এর একটা Buffer-এ যায়, তারপর নির্দিষ্ট সময় পর Disk-এ Flush (লেখা) হয়। যদি এই Flush হওয়ার আগেই Power চলে যায়, তাহলে সেই ডেটা হারিয়ে যেতে পারে — এই কারণেই WAL (Write Ahead Log) এবং Binlog এর মতো প্রযুক্তি দরকার হয় (এগুলো আমরা Module 2-এ বিস্তারিত শিখবো)।

---

## 1.4 Disaster (দুর্যোগ) — Database প্রেক্ষাপটে

### Disaster কী?

Database Disaster বলতে বোঝায় এমন একটি ঘটনা যা পুরো Database System বা Data Center-কে সাময়িক বা স্থায়ীভাবে অকার্যকর করে দেয়।

### প্রকারভেদ (Types)

| Disaster Type | উদাহরণ | প্রভাব |
|---|---|---|
| Natural Disaster | ভূমিকম্প, বন্যা, আগুন | পুরো Data Center ধ্বংস |
| Infrastructure Failure | Power Grid Failure, Network Outage | সাময়িক Downtime |
| Cyber Attack | Ransomware, DDoS | ডেটা Encrypt/Inaccessible |
| Regional Cloud Outage | AWS Region Down | পুরো Service বন্ধ |

### Real World Example

**AWS S3 Outage (2017, US-EAST-1)** — একটা ভুল Command-এর কারণে বিশাল একটা অংশ Internet Service কয়েক ঘণ্টার জন্য বন্ধ হয়ে গিয়েছিল। যেসব কোম্পানি একাধিক Region-এ Backup রেখেছিল, তারা দ্রুত Recover করতে পেরেছিল (একে বলে **Cross-Region Backup**, যা আমরা Module 15-এ শিখবো)।

---

## 1.5 Corruption (ডেটা নষ্ট হয়ে যাওয়া)

### Corruption কী?

Data Corruption মানে হলো Database-এর ভেতরে থাকা ডেটা এমনভাবে নষ্ট হয়ে যাওয়া যে সেটা আর সঠিকভাবে পড়া বা ব্যবহার করা যায় না। ডেটা হারিয়ে যায় না ঠিকই, কিন্তু সেটা "Garbage" (অকেজো) হয়ে যায়।

### Corruption এর কারণ

1. **Unclean Shutdown** — Database চলাকালীন হঠাৎ Power বন্ধ হয়ে গেলে।
2. **Disk Bad Sector** — Physical Disk-এর কোনো অংশ নষ্ট হয়ে গেলে।
3. **Filesystem Bug** — Operating System-এর File System-এ সমস্যা।
4. **RAM Corruption (Bit Flip)** — খুবই বিরল, কিন্তু ঘটে।
5. **Buggy Query/Transaction** — কোনো Transaction Half-complete অবস্থায় থেমে গেলে।

### Diagram: Corruption কীভাবে সনাক্ত করা হয়

```
Database Startup
      │
      ▼
Checksum Verification (প্রতিটা Page/Block-এর জন্য)
      │
      ├── Match হলে ──► Normal Operation
      │
      └── Mismatch হলে ──► "Corruption Detected" Error
                                  │
                                  ▼
                         Backup থেকে Restore প্রয়োজন
```

### Production Note

MySQL InnoDB এবং PostgreSQL উভয়েই প্রতিটা Data Page-এর একটা **Checksum** রাখে, যাতে পড়ার সময় বোঝা যায় ডেটা Corrupt হয়েছে কিনা। এই Checksum Verification হলো তোমার প্রথম Alarm System — যদি এই Error দেখো, সাথে সাথে Backup Restore Plan Activate করতে হবে।

---

## 1.6 Human Error (মানুষের ভুল)

### সবচেয়ে সাধারণ কারণ

ইন্ডাস্ট্রিতে গবেষণায় দেখা গেছে, Database Incident-এর একটা বিশাল অংশ (প্রায় ৩০-৪০%) হয় **মানুষের ভুলের কারণে**, হার্ডওয়্যার সমস্যার কারণে না।

### সাধারণ ভুলসমূহ

```sql
-- ভুল ১: WHERE clause ছাড়া DELETE
DELETE FROM users;  -- এটা পুরো টেবিল খালি করে দেবে!

-- সঠিক
DELETE FROM users WHERE id = 105;

-- ভুল ২: WHERE clause ছাড়া UPDATE
UPDATE orders SET status = 'cancelled';  -- সব অর্ডার Cancel হয়ে যাবে!

-- সঠিক
UPDATE orders SET status = 'cancelled' WHERE id = 2045;

-- ভুল ৩: ভুল Database-এ কমান্ড চালানো
DROP DATABASE production_db;  -- Production-এ ভুলে চালিয়ে ফেলা!
```

### বাস্তব ঘটনা

**GitLab (2017)** এর ঘটনায় একজন Engineer ভুলে Production Database Server-এ একটা Command চালিয়ে ফেলেন যা Replica Server-কে টার্গেট করার কথা ছিল, কিন্তু ভুলবশত Primary Server-এ চলে যায় এবং প্রায় ৩০০ GB ডেটা মুছে যায়।

### প্রতিরোধের উপায় (Prevention)

| পদ্ধতি | ব্যাখ্যা |
|---|---|
| Soft Delete | সরাসরি DELETE না করে `is_deleted = true` ফ্ল্যাগ ব্যবহার করা |
| Confirmation Prompt | Production-এ কমান্ড চালানোর আগে Confirmation চাওয়া |
| Least Privilege Access | সবাইকে Full Access না দিয়ে প্রয়োজন অনুযায়ী Access দেওয়া |
| Staging Environment | সরাসরি Production-এ Test না করে আলাদা Staging Server ব্যবহার |
| Automated Backup | যেকোনো ভুল হলেও Backup থেকে ফিরিয়ে আনার ব্যবস্থা রাখা |

---

## 1.7 Hardware Failure (হার্ডওয়্যার নষ্ট হওয়া)

### সাধারণ Hardware Failure এর ধরন

```
Hardware Components That Fail
│
├── Hard Disk Drive (HDD) — যান্ত্রিক অংশ থাকায় সবচেয়ে বেশি নষ্ট হয়
├── Solid State Drive (SSD) — সীমিত Write Cycle থাকে
├── RAM — বিরল কিন্তু Bit Errors হতে পারে
├── Power Supply Unit (PSU) — হঠাৎ বন্ধ হয়ে Corruption তৈরি করতে পারে
├── Network Card — Data Transfer এর সময় সমস্যা
└── Motherboard/CPU — সম্পূর্ণ Server Failure
```

### MTBF (Mean Time Between Failures) ধারণা

Industry-তে Hard Disk-এর একটা পরিসংখ্যান থাকে যাকে বলে **MTBF** — অর্থাৎ গড়ে কতদিন পর একটা Hardware নষ্ট হতে পারে। বড় বড় Data Center (Google, Amazon) এই পরিসংখ্যান ব্যবহার করে বুঝতে পারে কবে Hardware পরিবর্তন করতে হবে, কিন্তু তারপরেও Backup রাখা আবশ্যক কারণ এটা কখনোই ১০০% নিশ্চিত না।

### সমাধান: RAID কি Backup এর বিকল্প?

**গুরুত্বপূর্ণ ভুল ধারণা:** অনেকে মনে করে RAID (Redundant Array of Independent Disks) থাকলে Backup লাগে না। এটা সম্পূর্ণ ভুল।

```
RAID এর কাজ:  Hardware Failure থেকে সুরক্ষা (একটা Disk নষ্ট হলে অন্য Disk থেকে ডেটা পাওয়া)
Backup এর কাজ: Logical Error, Human Error, Disaster থেকে সুরক্ষা

RAID ≠ Backup
```

RAID যদি একটা Disk-এর Failure থেকে বাঁচায়, কিন্তু যদি তুমি ভুলে `DROP TABLE` করো, তাহলে RAID এর সব Disk থেকেই সেই টেবিল মুছে যাবে — কারণ RAID শুধু একই ডেটা একাধিক Disk-এ Mirror করে রাখে।

---

## 1.8 Ransomware (মুক্তিপণ ম্যালওয়্যার)

### Ransomware কী?

Ransomware হলো এক ধরনের Malicious Software (ক্ষতিকর সফটওয়্যার) যা তোমার Database বা Files-কে Encrypt করে ফেলে, এবং সেই Encryption খুলতে (Decrypt করতে) টাকা দাবি করে।

### কীভাবে কাজ করে

```
Hacker Attack
      │
      ▼
Database/Files Encrypted (মূল ডেটা Access করা যায় না)
      │
      ▼
Ransom Note (মুক্তিপণের দাবি — সাধারণত Bitcoin/Crypto তে)
      │
      ▼
  ┌──────────────────────┬─────────────────────────┐
  │                       │                          │
  ▼                       ▼                          ▼
টাকা দেওয়া          Backup থেকে Restore       ডেটা স্থায়ীভাবে হারানো
(ঝুঁকিপূর্ণ,          (সবচেয়ে নিরাপদ ও          (Backup না থাকলে)
নিশ্চয়তা নেই)         সুপারিশকৃত পদ্ধতি)
```

### বিখ্যাত ঘটনা

**MongoDB Ransom Attack (2017)** — হাজার হাজার Unprotected (কোনো Password ছাড়া, Internet-এ Open) MongoDB Server আক্রান্ত হয়েছিল, যেখানে হ্যাকাররা ডেটা মুছে ফেলে টাকা দাবি করেছিল। যাদের Backup ছিল তারা সহজে Recover করতে পেরেছিল, বাকিরা স্থায়ীভাবে ডেটা হারিয়েছে।

### Ransomware থেকে সুরক্ষার Best Practices

1. **Offline/Air-gapped Backup** রাখা (যা Internet থেকে সরাসরি Access করা যায় না)।
2. **Immutable Backup** ব্যবহার করা (যা একবার লেখার পর পরিবর্তন করা যায় না — যেমন AWS S3 Object Lock)।
3. Database-কে Internet-এ সরাসরি Expose না করা (Firewall/VPC ব্যবহার করা)।
4. নিয়মিত Security Patch আপডেট রাখা।
5. Multiple Backup Copies রাখা (3-2-1 Rule — যা আমরা পরে বিস্তারিত শিখবো)।

---

## 1.9 Backup vs Restore — পার্থক্য বোঝা

এই দুইটা শব্দ প্রায়ই গুলিয়ে ফেলে অনেকে, তাই স্পষ্টভাবে বোঝা দরকার।

| বিষয় | Backup | Restore |
|---|---|---|
| সংজ্ঞা | ডেটার একটা কপি তৈরি করে রাখা | Backup থেকে ডেটা ফিরিয়ে Database-এ বসানো |
| দিক | Production → Backup Storage (বাইরে যাওয়া) | Backup Storage → Production (ভেতরে আসা) |
| কখন হয় | নিয়মিত, Scheduled ভাবে (প্রতিদিন/প্রতি ঘণ্টায়) | শুধুমাত্র প্রয়োজন হলে (Incident এর পর) |
| ঝুঁকি | কম ঝুঁকিপূর্ণ | বেশি ঝুঁকিপূর্ণ (ভুল Restore পুরো System নষ্ট করে দিতে পারে) |
| Frequency | High (প্রতিদিন) | Low (কালেভদ্রে, শুধু জরুরি অবস্থায়) |
| Testing প্রয়োজন | হ্যাঁ | হ্যাঁ (এটাই সবচেয়ে গুরুত্বপূর্ণ, কিন্তু সবচেয়ে অবহেলিত) |

### Diagram: সম্পূর্ণ Cycle

```
┌────────────┐    Backup Process    ┌──────────────┐
│ Production │ ────────────────────►│ Backup Store  │
│  Database  │                       │ (Disk/Cloud)  │
└────────────┘                       └──────────────┘
      ▲                                     │
      │                                     │
      │          Restore Process            │
      └─────────────────────────────────────┘
              (শুধু Incident/Disaster হলে)
```

### গুরুত্বপূর্ণ Principle

> **"A backup is only as good as its restore."**
> (একটা Backup ততটাই ভালো, যতটা ভালোভাবে সেটা Restore করা যায়।)

তুমি যদি প্রতিদিন Backup নাও, কিন্তু কখনো Restore করে পরীক্ষা না করো, তাহলে তুমি জানোই না যে তোমার Backup আসলে কাজ করে কিনা। তাই Industry-তে একটা নিয়ম আছে — **Restore Drill / Restore Testing**, যা নিয়মিত (মাসে একবার বা তার বেশি) করা উচিত।

---

## 1.10 Real Production Case Studies (বাস্তব প্রোডাকশন কেস স্টাডি)

### Case Study 1: GitLab.com Data Loss (2017)

- **সমস্যা:** একজন Engineer ভুলে Production Database-এর একটা Directory Delete করে ফেলেন।
- **Backup Status:** তাদের ৫ ধরনের Backup Mechanism ছিল, কিন্তু ৪টাই কাজ করছিল না (কেউ Test করেনি)।
- **ফলাফল:** প্রায় ৬ ঘণ্টার User Data স্থায়ীভাবে হারিয়ে যায়।
- **শিক্ষা:** Backup থাকাই যথেষ্ট না, নিয়মিত Restore Test করা বাধ্যতামূলক।

### Case Study 2: MongoDB Ransomware Wave (2017)

- **সমস্যা:** হাজারো MongoDB Instance ইন্টারনেটে Password ছাড়া Open ছিল।
- **ফলাফল:** ডেটা মুছে ফেলে মুক্তিপণ দাবি করা হয়।
- **শিক্ষা:** Security Configuration + Offline Backup দুটোই দরকার।

### Case Study 3: Amazon S3 Outage (2017)

- **সমস্যা:** একটা ভুল কমান্ডের কারণে US-EAST-1 Region-এর বিশাল অংশ Down হয়ে যায়।
- **শিক্ষা:** যারা Multi-Region Backup রেখেছিল, তারা দ্রুত Recover করতে পেরেছিল।

---

## 1.11 Best Practices (Production-Level)

1. **3-2-1 Backup Rule** অনুসরণ করো:
   - কমপক্ষে **3টা** কপি রাখো
   - **2টা** ভিন্ন Storage Media-তে রাখো (যেমন Local Disk + Cloud)
   - **1টা** কপি Off-site (অন্য জায়গায়/Region-এ) রাখো

2. Backup নেওয়ার পর অবশ্যই **Verify** করো (Checksum/Test Restore দিয়ে)।

3. Backup **Automate** করো — Manual Backup ভুলে যাওয়ার সম্ভাবনা থাকে।

4. Backup **Encrypt** করো, বিশেষ করে যদি এতে Sensitive User Data থাকে।

5. **Retention Policy** নির্ধারণ করো — কতদিনের Backup রাখবে (যেমন ৩০ দিন Daily, ১২ মাস Monthly)।

6. Backup Storage-এ **Access Control** রাখো — যাতে যে কেউ Backup Access/Delete করতে না পারে।

7. **Monitoring & Alerting** সেটআপ করো — Backup Fail হলে সাথে সাথে যেন Notification যায়।

---

## 1.12 Common Mistakes (সাধারণ ভুলসমূহ)

| ভুল | সমস্যা | সমাধান |
|---|---|---|
| Backup নিয়ে কখনো Restore Test না করা | Backup আসলে কাজ করে কিনা জানা যায় না | নিয়মিত Restore Drill করা |
| শুধু একটা কপি রাখা | সেই কপিও নষ্ট হলে সব শেষ | 3-2-1 Rule মেনে চলা |
| Backup Encrypt না করা | Data Breach এর ঝুঁকি | Encryption বাধ্যতামূলক করা |
| RAID-কেই Backup মনে করা | Logical Error থেকে সুরক্ষা নেই | আলাদা Backup System রাখা |
| একই Server-এ Backup রাখা | পুরো Server নষ্ট হলে Backup ও হারায় | Off-site/Cloud Backup রাখা |
| Manual Backup-এর উপর নির্ভর করা | মানুষ ভুলে যেতে পারে | Automation ব্যবহার করা |

---

## 1.13 Interview Questions (Module 1)

1. **Backup কী এবং কেন এটা গুরুত্বপূর্ণ?**
   *উত্তর:* Backup হলো ডেটার একটা কপি যা Data Loss, Corruption, Disaster থেকে সুরক্ষার জন্য রাখা হয়, যাতে প্রয়োজনে Restore করা যায়।

2. **RAID কি Backup এর বিকল্প হতে পারে? কেন না?**
   *উত্তর:* না, কারণ RAID শুধু Hardware Failure থেকে সুরক্ষা দেয়, কিন্তু Human Error বা Logical Corruption (যেমন ভুল DELETE) থেকে সুরক্ষা দেয় না, কারণ RAID একই ডেটা একাধিক Disk-এ Mirror করে রাখে।

3. **3-2-1 Backup Rule কী?**
   *উত্তর:* ৩টা কপি, ২টা ভিন্ন Media-তে, ১টা Off-site।

4. **Backup আর Restore এর মধ্যে পার্থক্য কী?**
   *উত্তর:* Backup হলো ডেটার কপি তৈরি করা (Production → Storage), আর Restore হলো সেই কপি থেকে ডেটা ফিরিয়ে আনা (Storage → Production)।

5. **কেন শুধু Backup নেওয়া যথেষ্ট না?**
   *উত্তর:* কারণ Backup যদি Corrupt হয় বা সঠিকভাবে কাজ না করে, সেটা তখনই জানা যাবে যখন Restore করার প্রয়োজন হবে। তাই নিয়মিত Restore Test করা জরুরি।

6. **GitLab 2017 Incident থেকে কী শিক্ষা পাওয়া যায়?**
   *উত্তর:* Backup থাকাই যথেষ্ট না, সবগুলো Backup Mechanism নিয়মিতভাবে Test করে নিশ্চিত হতে হবে যে সেগুলো আসলেই কাজ করে।

7. **Ransomware থেকে বাঁচার জন্য কী ধরনের Backup দরকার?**
   *উত্তর:* Offline/Air-gapped এবং Immutable Backup, যাতে Ransomware Backup-কেও Encrypt করতে না পারে।

---

## 1.14 Summary (সারাংশ)

- Backup হলো Database-এর ডেটার একটা Point-in-time কপি, যা Data Loss থেকে সুরক্ষা দেয়।
- Data Loss হতে পারে Hardware Failure, Human Error, Corruption, Disaster, বা Ransomware এর কারণে।
- RAID Backup এর বিকল্প না — এটা শুধু Hardware Redundancy দেয়।
- Backup নেওয়ার পাশাপাশি Restore Testing করা সমান গুরুত্বপূর্ণ।
- 3-2-1 Rule একটি Industry Standard Best Practice।
- Production-এ Backup Automation, Encryption, Monitoring অপরিহার্য।

---

## 1.15 Exercises & Assignments

### Exercise 1
তোমার নিজের একটা Local MySQL/PostgreSQL Database তৈরি করো, তারপর সেখান থেকে ভুলে `DELETE FROM table_name;` (কোনো WHERE ছাড়া) চালিয়ে দেখো কী হয়। এরপর চিন্তা করো — যদি Backup না থাকতো, কী হতো?

### Exercise 2
একটা লিস্ট বানাও — তোমার বর্তমান প্রজেক্টে (Django/Spring Boot) কী কী ধরনের Data Loss Risk আছে (Human Error, Hardware, ইত্যাদি)।

### Mini Project
একটা ছোট Documentation লেখো (নিজের ভাষায়) — "যদি আমার Production Database আজ রাতে সম্পূর্ণ নষ্ট হয়ে যায়, আমার কাছে বর্তমানে Recovery করার কী উপায় আছে?" — এটা একে বলে **Disaster Recovery Readiness Assessment**, যা প্রতিটা Senior Engineer করে থাকে।

---

**পরবর্তী চ্যাপ্টার:** `02-Database-Architecture-Basics.md` — এখানে আমরা শিখবো Database এর ভেতরে আসলে কী ঘটে (Storage Engine, WAL, Transaction, ACID) যা Backup বোঝার জন্য অপরিহার্য ভিত্তি।
