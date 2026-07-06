# Module 8: MongoDB Backup (মঙ্গোডিবি ব্যাকআপ)

> **Level:** Intermediate → Advanced
> **Prerequisite:** Module 1-4। MongoDB একটা NoSQL, Document-based Database, তাই Relational Database থেকে কিছু ধারণা ভিন্নভাবে প্রয়োগ হয়।

---

## 8.0 Chapter Roadmap

```
8.1  MongoDB Architecture সংক্ষেপে (Backup বোঝার জন্য প্রয়োজনীয়)
8.2  mongodump
8.3  mongorestore
8.4  Filesystem Snapshot
8.5  Replica Set Backup
8.6  Sharding Backup
8.7  Cloud Backup (MongoDB Atlas)
8.8  Best Practices
8.9  Common Mistakes
8.10 Interview Questions
8.11 Summary
8.12 Exercises & Assignments
```

---

## 8.1 MongoDB Architecture সংক্ষেপে

### কেন এটা জানা দরকার

MongoDB একটা **Document-Oriented NoSQL Database**, যেখানে ডেটা JSON-এর মতো (BSON Format এ) Document আকারে থাকে, Table/Row এর বদলে **Collection/Document** ব্যবহার হয়। এছাড়া MongoDB এর নিজস্ব Storage Engine (Default: **WiredTiger**, যা Module 2-এ উল্লেখিত হয়েছিল) আছে, যেখানে নিজস্ব Journal (WAL এর সমতুল্য) ব্যবস্থা আছে।

### MongoDB এর মূল স্থাপত্যিক ধারণা

```
Database
   │
   ▼
Collection  (MySQL এর Table এর সমতুল্য)
   │
   ▼
Document    (MySQL এর Row এর সমতুল্য, কিন্তু JSON/BSON আকারে)

উদাহরণ Document:
{
  "_id": ObjectId("64f1..."),
  "name": "Karim",
  "email": "karim@example.com",
  "orders": [
    { "product": "Laptop", "price": 50000 }
  ]
}
```

### Replica Set ও Sharding — প্রাথমিক ধারণা (বিস্তারিত পরের Section এ)

MongoDB Production এ সাধারণত একা চলে না — এটা **Replica Set** (একাধিক Copy, High Availability এর জন্য) বা **Sharded Cluster** (ডেটা একাধিক Server এ ভাগ করে রাখা, বড় Scale এর জন্য) হিসেবে চলে। এই দুটো ধারণা Backup Strategy কে সরাসরি প্রভাবিত করে।

---

## 8.2 mongodump

### সংজ্ঞা

`mongodump` হলো MongoDB এর Built-in **Logical Backup Tool**, যা Database/Collection কে BSON Format এ Export করে (Module 3 এর Logical Backup ধারণার MongoDB প্রয়োগ)।

### মূল ব্যবহার

```bash
# সম্পূর্ণ Database Backup
mongodump --uri="mongodb://localhost:27017" --db=mydb --out=/backup/

# নির্দিষ্ট Collection Backup
mongodump --uri="mongodb://localhost:27017" --db=mydb \
    --collection=users --out=/backup/

# Authentication সহ
mongodump --uri="mongodb://user:password@localhost:27017" \
    --db=mydb --out=/backup/

# Compressed Backup (gzip)
mongodump --uri="mongodb://localhost:27017" --db=mydb \
    --gzip --out=/backup/

# Archive Format এ (একটা মাত্র ফাইলে সব কিছু)
mongodump --uri="mongodb://localhost:27017" --db=mydb \
    --archive=/backup/mydb_backup.archive --gzip
```

### Output Structure

```
/backup/
└── mydb/
    ├── users.bson          # Document ডেটা (Binary JSON)
    ├── users.metadata.json # Index/Schema Information
    ├── orders.bson
    └── orders.metadata.json
```

### Consistency নিয়ে গুরুত্বপূর্ণ বিষয়: `--oplog` Flag

Single Server এ `mongodump` চালানোর সময়, Backup সম্পূর্ণ হতে যদি সময় লাগে (বড় Database), তাহলে শুরুর Collection এবং শেষের Collection এর মধ্যে Consistency সমস্যা হতে পারে (একটার ডেটা আগের সময়ের, আরেকটার পরের সময়ের)। এই সমস্যা সমাধানে `--oplog` Flag ব্যবহার করা হয় (Replica Set এ)।

```bash
mongodump --uri="mongodb://localhost:27017" --oplog --out=/backup/
```

`--oplog` চালু থাকলে, mongodump Backup চলাকালীন সময়ের সব Operation ও Oplog (Operation Log — MongoDB এর নিজস্ব WAL সদৃশ Log) থেকে সংগ্রহ করে, এবং Restore এর সময় সেগুলো Apply করে একটা Point-in-time Consistent Snapshot তৈরি করে — অনেকটা Module 5-এ শেখা MySQL Binlog এর মতোই কাজ করে।

```
Diagram: --oplog কীভাবে Consistency নিশ্চিত করে

mongodump শুরু (T0)
        │
        ▼
Collection A Backup হচ্ছে ── (এই সময়ে নতুন Write হতে থাকে)
        │
        ▼
Collection B Backup হচ্ছে
        │
        ▼
mongodump শেষ (T1)
        │
        ▼
T0 থেকে T1 পর্যন্ত সব Oplog Entry ও Backup এর সাথে সংগ্রহ করা হয়
        │
        ▼
Restore এর সময় এই Oplog Apply করে সব Collection কে
একই (T1) মুহূর্তের Consistent State এ আনা হয়
```

---

## 8.3 mongorestore

### সংজ্ঞা

`mongorestore` হলো mongodump দিয়ে নেওয়া BSON Backup থেকে ডেটা ফিরিয়ে আনার Tool।

### মূল ব্যবহার

```bash
# সম্পূর্ণ Database Restore
mongorestore --uri="mongodb://localhost:27017" --db=mydb /backup/mydb/

# Archive Format থেকে Restore
mongorestore --uri="mongodb://localhost:27017" \
    --archive=/backup/mydb_backup.archive --gzip

# Oplog সহ Restore করা (Consistent Restore এর জন্য)
mongorestore --uri="mongodb://localhost:27017" \
    --oplogReplay /backup/

# একটা নির্দিষ্ট Collection শুধু Restore করা
mongorestore --uri="mongodb://localhost:27017" \
    --db=mydb --collection=users /backup/mydb/users.bson

# Existing Data Drop করে তারপর Restore (সতর্কতার সাথে ব্যবহার করতে হবে)
mongorestore --uri="mongodb://localhost:27017" \
    --db=mydb --drop /backup/mydb/
```

### `--drop` Flag নিয়ে সতর্কতা

`--drop` Flag ব্যবহার করলে Restore করার আগে Existing Collection সম্পূর্ণ Delete করে দেওয়া হয়, তারপর নতুন করে ডেটা বসানো হয়। Production এ এটা ব্যবহার করার আগে অবশ্যই নিশ্চিত হতে হবে যে এটাই তোমার উদ্দেশ্য, কারণ ভুলবশত Existing Live ডেটা Delete হয়ে যেতে পারে (Module 1-এ শেখা Human Error এর একটা ক্লাসিক উদাহরণ)।

---

## 8.4 Filesystem Snapshot

### সংজ্ঞা

MongoDB এর ডেটা Physically Disk এ যেভাবে থাকে (WiredTiger এর Data ফাইল), সেই পুরো Disk/Volume এর একটা **Snapshot** নেওয়া যায় (Module 3-এ শেখা Snapshot Backup এর MongoDB প্রয়োগ) — এটা `mongodump` এর চেয়ে অনেক দ্রুত, বিশেষ করে বড় Database এ।

### কেন Filesystem Snapshot ব্যবহার করবে

| বিষয় | mongodump (Logical) | Filesystem Snapshot (Physical) |
|---|---|---|
| গতি | ধীর (বড় DB তে) | দ্রুত |
| Restore গতি | ধীর (Index পুনরায় Build) | দ্রুত |
| Consistency নিশ্চিত করা | `--oplog` লাগে | Storage-level এ নিজে থেকেই দ্রুত, কিন্তু Database-Aware Snapshot Tool ব্যবহার করা ভালো |

### Consistent Snapshot নেওয়ার সঠিক পদ্ধতি (গুরুত্বপূর্ণ ধাপ)

```bash
# ধাপ ১: MongoDB কে সাময়িকভাবে Write Lock করা (fsync + lock)
mongosh --eval "db.fsyncLock()"

# ধাপ ২: Filesystem/Cloud Snapshot নেওয়া
# (উদাহরণ: AWS EBS Snapshot)
aws ec2 create-snapshot --volume-id vol-0abcd1234efgh5678 \
    --description "MongoDB consistent snapshot"

# ধাপ ৩: Lock ছেড়ে দেওয়া
mongosh --eval "db.fsyncUnlock()"
```

### কেন `fsyncLock` প্রয়োজন

`db.fsyncLock()` চালালে MongoDB সব Pending Write Disk এ Flush করে দেয় এবং নতুন Write সাময়িকভাবে Block করে দেয়, যাতে Snapshot নেওয়ার মুহূর্তে Disk এর ডেটা সম্পূর্ণ Consistent থাকে (Module 2-এ শেখা Memory থেকে Disk এ Flush হওয়ার ধারণা মনে করো — এখানে সেটাই জোরপূর্বক করানো হচ্ছে)।

```
Diagram: fsyncLock এর ভূমিকা

fsyncLock() চালানো হলো
        │
        ▼
সব Pending Write Memory থেকে Disk এ Flush হলো
        │
        ▼
নতুন Write Block হয়ে গেলো (সাময়িকভাবে)
        │
        ▼
এখন Disk এর অবস্থা সম্পূর্ণ Consistent — Snapshot নেওয়ার
জন্য নিরাপদ সময়
        │
        ▼
fsyncUnlock() চালিয়ে আবার Normal Operation চালু করা
```

---

## 8.5 Replica Set Backup

### সংজ্ঞা

**Replica Set** হলো MongoDB এর High Availability Architecture, যেখানে একটা **Primary** Node (Write Handle করে) এবং একাধিক **Secondary** Node (Primary এর ডেটা Replicate করে রাখে) থাকে।

### Architecture Diagram

```
┌───────────────┐
│    Primary       │  ◄── সব Write এখানে হয়
│  (Read + Write)   │
└───────┬───────┘
        │  Oplog Replication
   ┌────┴────┐
   ▼           ▼
┌────────┐ ┌────────┐
│Secondary 1│ │Secondary 2│  ◄── Backup নেওয়া হয় এখান থেকে,
│(Read-only) │ │(Read-only) │      Primary এর Load কমাতে
└────────┘ └────────┘
```

### Secondary থেকে Backup নেওয়া (Module 5-এ MySQL Replica এর মতো একই ধারণা)

```bash
# একটা Secondary Node থেকে Backup নেওয়া
mongodump --uri="mongodb://secondary_host:27017" \
    --oplog --out=/backup/
```

### গুরুত্বপূর্ণ বিবেচনা: Read Preference

```bash
# নির্দিষ্টভাবে Secondary থেকে Read করার জন্য
mongodump --uri="mongodb://replica_set/mydb?readPreference=secondary" \
    --out=/backup/
```

### Replica Set এর জন্য Best Practice: Hidden Member ব্যবহার করা

Production এ প্রায়ই একটা **Hidden Replica Member** রাখা হয়, যা Application এর Read Traffic এর জন্য দৃশ্যমান না, শুধুমাত্র Backup নেওয়ার জন্য ব্যবহৃত হয়। এতে Backup Process কোনোভাবেই User-facing Traffic কে প্রভাবিত করে না।

```javascript
// rs.conf() এ একটা Member কে Hidden + Priority 0 করা
{
  "_id": 2,
  "host": "backup_node:27017",
  "priority": 0,
  "hidden": true
}
```

---

## 8.6 Sharding Backup

### সংজ্ঞা

**Sharding** হলো MongoDB এর একটা Technique, যেখানে বড় Dataset কে একাধিক Server (Shard) এ ভাগ করে রাখা হয়, যাতে Horizontal Scaling সম্ভব হয়।

### Sharded Cluster Architecture

```
┌─────────────────────────────────────────────┐
│                 mongos (Router)                  │
│         (Application এখানে Connect করে)          │
└───────────┬───────────┬───────────┬────────────┘
            │             │             │
      ┌─────▼─────┐ ┌─────▼─────┐ ┌─────▼─────┐
      │  Shard 1     │ │  Shard 2     │ │  Shard 3     │
      │ (Replica Set)│ │ (Replica Set)│ │ (Replica Set)│
      └───────────┘ └───────────┘ └───────────┘
            │             │             │
            └──────┬──────┴──────┬──────┘
                    ▼             ▼
              ┌─────────────────────┐
              │   Config Servers        │
              │ (Metadata: কোন ডেটা      │
              │  কোন Shard এ আছে)         │
              └─────────────────────┘
```

### কেন Sharding Backup জটিল

Sharding এ ডেটা একাধিক Server এ ছড়ানো থাকে, তাই সব Shard কে একই মুহূর্তে Consistent ভাবে Backup নেওয়া একটা চ্যালেঞ্জ — যদি একটা Shard Backup নেওয়ার সময় অন্য Shard এ নতুন Write হয়, তাহলে Cross-Shard Transaction/Relationship Inconsistent হতে পারে।

### সঠিক Sharding Backup Process

```
1. Balancer বন্ধ করা (Data Migration সাময়িক থামানো)
        │
        ▼
2. প্রতিটা Shard কে আলাদাভাবে Backup নেওয়া (তাদের নিজস্ব
   Secondary/Replica থেকে, একই সময়ে Parallel ভাবে)
        │
        ▼
3. Config Server ডেটা Backup নেওয়া (Metadata সংরক্ষণের জন্য)
        │
        ▼
4. Balancer আবার চালু করা
```

```bash
# ধাপ ১: Balancer বন্ধ করা
mongosh --eval "sh.stopBalancer()"

# ধাপ ২: প্রতিটা Shard এর Backup (Parallel ভাবে চালানো যায়)
mongodump --uri="mongodb://shard1_secondary:27017" --oplog --out=/backup/shard1/
mongodump --uri="mongodb://shard2_secondary:27017" --oplog --out=/backup/shard2/
mongodump --uri="mongodb://shard3_secondary:27017" --oplog --out=/backup/shard3/

# ধাপ ৩: Config Server Backup
mongodump --uri="mongodb://config_server:27019" --out=/backup/config/

# ধাপ ৪: Balancer আবার চালু করা
mongosh --eval "sh.startBalancer()"
```

### Production Note

Balancer বন্ধ রাখার সময় যত কম রাখা যায় তত ভালো, কারণ এই সময়ে Data Migration/Rebalancing হয় না, যা দীর্ঘ সময় বন্ধ থাকলে Cluster এর Load Distribution অসামঞ্জস্যপূর্ণ হয়ে যেতে পারে।

---

## 8.7 Cloud Backup (MongoDB Atlas)

### সংজ্ঞা

**MongoDB Atlas** হলো MongoDB এর Official Managed Cloud Service, যেখানে Backup ব্যবস্থাপনা অনেকাংশে স্বয়ংক্রিয় ও সহজ।

### Atlas এর Backup Feature সমূহ

| Feature | ব্যাখ্যা |
|---|---|
| Continuous Backup | প্রতি কয়েক মিনিটে Snapshot + Oplog নিয়ে প্রায় যেকোনো মুহূর্তে PITR সম্ভব |
| Cloud Provider Snapshot | নির্দিষ্ট Schedule এ (দৈনিক/সাপ্তাহিক) Full Snapshot |
| Cross-Region Backup Copy | Disaster Recovery এর জন্য অন্য Region এ Backup রাখা |
| On-Demand Snapshot | যেকোনো মুহূর্তে Manual Snapshot নেওয়ার সুবিধা |

### Atlas এ PITR এর সুবিধা

Atlas এর Continuous Backup Feature Enable থাকলে, তুমি Module 4-এ শেখা PITR ধারণা ব্যবহার করে যেকোনো নির্দিষ্ট Second পর্যন্ত Database Restore করতে পারবে, কোনো Manual Oplog Management ছাড়াই — Atlas নিজেই এটা Manage করে।

### Atlas UI/CLI দিয়ে Restore (Concept)

```bash
# Atlas CLI ব্যবহার করে একটা নির্দিষ্ট মুহূর্তে Restore করা
atlas backups restores start pointInTime \
    --clusterName myCluster \
    --pointInTimeUTCMillis 1720180800000 \
    --targetProjectId <project_id> \
    --targetClusterName myRestoredCluster
```

### Self-hosted vs Atlas — Trade-off

| বিষয় | Self-hosted MongoDB | MongoDB Atlas |
|---|---|---|
| Backup Setup | Manual (mongodump/Snapshot Script) | স্বয়ংক্রিয়, Built-in |
| PITR | নিজে Configure করতে হয় | Built-in Feature |
| Cost | Infrastructure Cost কম, কিন্তু Engineer সময় বেশি লাগে | Subscription Cost বেশি, কিন্তু Setup সহজ |
| নিয়ন্ত্রণ | সম্পূর্ণ নিয়ন্ত্রণ | Managed Service এর সীমাবদ্ধতা |

---

## 8.8 Best Practices

1. Production এ সবসময় Secondary/Hidden Member থেকে Backup নাও, Primary কে বিরক্ত করো না।
2. `--oplog` Flag ব্যবহার করো Consistent Backup এর জন্য।
3. Sharded Cluster এ Backup নেওয়ার আগে Balancer বন্ধ করো, এবং যত কম সময় সম্ভব বন্ধ রাখো।
4. যদি Budget থাকে, MongoDB Atlas এর Continuous Backup ব্যবহার করো — এটা PITR সহজ করে দেয়।
5. Filesystem Snapshot নেওয়ার আগে `fsyncLock()` চালাও Consistency নিশ্চিত করতে।

---

## 8.9 Common Mistakes

| ভুল | সমস্যা |
|---|---|
| `--oplog` ছাড়া বড় Database এ mongodump চালানো | Collection গুলোর মধ্যে Inconsistency হতে পারে |
| `mongorestore --drop` ভুলবশত Production এ চালানো | Existing Live ডেটা Delete হয়ে যায় |
| Sharded Cluster এ Balancer বন্ধ না করে Backup নেওয়া | Cross-shard Data Inconsistency |
| fsyncLock ছাড়া সরাসরি Filesystem Snapshot নেওয়া | Inconsistent Physical Backup হতে পারে |
| Primary Node থেকে সরাসরি ভারী mongodump চালানো | Production Performance এ প্রভাব পড়ে |

---

## 8.10 Interview Questions (Module 8)

1. **mongodump এ `--oplog` Flag এর কাজ কী এবং কেন এটা গুরুত্বপূর্ণ?**
   *উত্তর:* এটা Backup চলাকালীন সময়ের Oplog Entry সংগ্রহ করে, যা Restore এর সময় Apply করে সব Collection কে একই Consistent মুহূর্তে নিয়ে আসে, যেহেতু একাধিক Collection ধারাবাহিকভাবে (একে একে) Backup নেওয়া হয় এবং এর মাঝে নতুন Write হতে পারে।

2. **Replica Set এ Backup কোথা থেকে নেওয়া উচিত এবং কেন?**
   *উত্তর:* Secondary (বিশেষত Hidden Priority-0) Node থেকে, কারণ এতে Primary Node এর উপর কোনো Extra Load পড়ে না এবং Live Traffic Performance অপ্রভাবিত থাকে।

3. **Sharded Cluster Backup নেওয়ার সময় Balancer কেন বন্ধ করতে হয়?**
   *উত্তর:* কারণ Balancer চলমান থাকলে Data Migration হতে পারে Shard এর মধ্যে, যা Backup চলাকালীন Cross-shard Consistency নষ্ট করতে পারে — একই ডেটা দুইবার বা কোনো ডেটা একবারও Backup না হওয়ার ঝুঁকি থাকে।

4. **fsyncLock() কীভাবে Filesystem Snapshot এর Consistency নিশ্চিত করে?**
   *উত্তর:* এটা সব Pending Write Memory থেকে Disk এ Flush করে দেয় এবং নতুন Write সাময়িক Block করে, ফলে Snapshot নেওয়ার মুহূর্তে Disk এর ডেটা সম্পূর্ণ Consistent থাকে।

5. **mongodump (Logical) এবং Filesystem Snapshot (Physical) এর মধ্যে কখন কোনটা বেছে নেবে?**
   *উত্তর:* ছোট-মাঝারি Database এবং Selective Restore প্রয়োজন হলে mongodump, আর বড় Database এ দ্রুত Backup/Restore প্রয়োজন হলে Filesystem Snapshot বেছে নেওয়া উচিত।

---

## 8.11 Summary (সারাংশ)

- **mongodump/mongorestore** হলো MongoDB এর Logical Backup/Restore Tool, `--oplog` Flag Consistency নিশ্চিত করে।
- **Filesystem Snapshot** দ্রুত Physical Backup পদ্ধতি, `fsyncLock()`/`fsyncUnlock()` দিয়ে Consistency নিশ্চিত করতে হয়।
- **Replica Set Backup** সবসময় Secondary/Hidden Node থেকে নেওয়া উচিত, Primary এর Load কমাতে।
- **Sharding Backup** জটিল — Balancer বন্ধ করে প্রতিটা Shard আলাদা আলাদা Backup নিয়ে Config Server ও Backup নিতে হয়।
- **MongoDB Atlas** Cloud Managed Service ব্যবহার করলে Continuous Backup ও PITR অনেক সহজ হয়ে যায়, Manual Configuration এর দরকার কমে যায়।
- Production এ সবসময় Secondary Node ব্যবহার + Oplog + নিয়মিত Restore Testing — এই তিনটা মূলনীতি অনুসরণ করা উচিত।

---

## 8.12 Exercises & Assignments

### Exercise 1
তোমার Local MongoDB এ (Single Server বা Docker Container) একটা `mongodump --oplog` চালিয়ে Output Structure দেখো, তারপর `mongorestore --oplogReplay` দিয়ে একটা নতুন Database এ Restore করো।

### Exercise 2
একটা ৩-Node Replica Set (Docker Compose দিয়ে সহজে তৈরি করা যায়) সেটআপ করে, Secondary Node থেকে Backup নেওয়ার Process Practice করো।

### Mini Project
তোমার নিজের প্রজেক্টের জন্য একটা সম্পূর্ণ MongoDB Backup Automation Script লেখো, যা:
- Secondary Node থেকে Backup নেবে
- Compress করবে (`--gzip`)
- একটা নির্দিষ্ট Cloud Storage এ Upload করবে
- পুরনো Backup Cleanup করবে

---

**পরবর্তী চ্যাপ্টার:** `09-Redis-Backup.md` — এখানে আমরা Redis এর RDB, AOF, Hybrid Persistence, এবং Production Backup Strategy নিয়ে শিখবো।
