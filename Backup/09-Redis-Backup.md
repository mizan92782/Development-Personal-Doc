# Module 9: Redis Backup (রেডিস ব্যাকআপ)

> **Level:** Intermediate → Advanced
> **Prerequisite:** Module 1-4। Redis একটা In-memory Data Store, তাই Persistence (স্থায়িত্ব) ধারণাটা এখানে বিশেষভাবে গুরুত্বপূর্ণ।

---

## 9.0 Chapter Roadmap

```
9.1  Redis কেন আলাদা (In-memory Database এর Backup চ্যালেঞ্জ)
9.2  RDB (Redis Database File)
9.3  AOF (Append Only File)
9.4  Hybrid Persistence (RDB + AOF)
9.5  Restore Process
9.6  Production Backup Strategy
9.7  Best Practices
9.8  Common Mistakes
9.9  Interview Questions
9.10 Summary
9.11 Exercises & Assignments
```

---

## 9.1 Redis কেন আলাদা (In-memory Database এর Backup চ্যালেঞ্জ)

### সংজ্ঞা ও মূল পার্থক্য

Redis হলো একটা **In-memory Data Store** — অর্থাৎ এর সব ডেটা মূলত **RAM**-এ থাকে, Module 2-এ শেখা Disk-based Database (MySQL/PostgreSQL) এর মতো Primarily Disk-এ না। এই কারণে Redis এর Backup Strategy সম্পূর্ণ ভিন্নভাবে কাজ করে।

### কেন এটা একটা চ্যালেঞ্জ

```
Diagram: Redis এর ডেটা কোথায় থাকে

┌─────────────────────────┐
│         RAM (Memory)          │  ◄── সব ডেটা এখানে, খুব দ্রুত Access
│   {key1: value1, key2:...}    │       কিন্তু Volatile (Power গেলে হারায়)
└─────────────────────────┘
              │
      Persistence Mechanism
      (RDB/AOF দিয়ে Disk এ
       নিয়মিত/ক্রমাগত Save করা)
              │
              ▼
┌─────────────────────────┐
│           Disk                  │  ◄── এখানে Save না করলে Power গেলে
│    (dump.rdb / appendonly.aof) │       সব ডেটা স্থায়ীভাবে হারিয়ে যাবে
└─────────────────────────┘
```

Redis Default ভাবে **কোনো Persistence ছাড়াই** চলতেও পারে (শুধু Cache হিসেবে ব্যবহার করলে এটা গ্রহণযোগ্য), কিন্তু যদি Redis-কে একটা **Primary Data Store** (শুধু Cache না) হিসেবে ব্যবহার করা হয়, তাহলে Persistence এবং Backup অপরিহার্য হয়ে যায়।

### Redis এর দুইটা প্রধান Persistence Mechanism

| Mechanism | ধরন | বর্ণনা |
|---|---|---|
| RDB | Snapshot-based | নির্দিষ্ট সময় অন্তর সম্পূর্ণ Dataset এর একটা Snapshot Disk এ Save করে |
| AOF | Log-based | প্রতিটা Write Command ক্রমানুসারে একটা Log ফাইলে Append করে |

---

## 9.2 RDB (Redis Database File)

### সংজ্ঞা

**RDB** হলো Redis এর একটা Persistence পদ্ধতি, যা নির্দিষ্ট সময় অন্তর (বা নির্দিষ্ট শর্ত পূরণ হলে) সম্পূর্ণ Dataset এর একটা **Point-in-time Snapshot** নিয়ে একটা Binary ফাইলে (`dump.rdb`) Save করে (Module 3-এ শেখা Snapshot Backup ধারণার সরাসরি প্রয়োগ)।

### RDB Configuration (redis.conf)

```ini
# redis.conf

# নিম্নলিখিত শর্তে RDB Snapshot নেওয়া হবে:
save 900 1      # ৯০০ সেকেন্ডে (১৫ মিনিট) অন্তত ১টা Key পরিবর্তন হলে
save 300 10     # ৩০০ সেকেন্ডে (৫ মিনিট) অন্তত ১০টা Key পরিবর্তন হলে
save 60 10000   # ৬০ সেকেন্ডে অন্তত ১০,০০০টা Key পরিবর্তন হলে

dbfilename dump.rdb
dir /var/lib/redis/
```

### Manual RDB Snapshot নেওয়া

```bash
# Redis CLI থেকে
redis-cli SAVE       # Blocking — যতক্ষণ Save না হচ্ছে, Redis অন্য Command Process করবে না
redis-cli BGSAVE     # Non-blocking — Background এ Child Process দিয়ে Save করে
```

### `SAVE` vs `BGSAVE` — গুরুত্বপূর্ণ পার্থক্য

```
Diagram: BGSAVE কীভাবে কাজ করে (Copy-on-Write ব্যবহার করে)

BGSAVE Command দেওয়া হলো
        │
        ▼
Redis একটা Child Process Fork করে
        │
        ▼
┌─────────────────────┐    ┌─────────────────────┐
│   Parent Process       │    │   Child Process        │
│   (Normal Client         │    │   (RDB ফাইলে Write     │
│    Request Serve         │    │    করছে, Memory এর     │
│    করতে থাকে)             │    │    একটা Copy থেকে)      │
└─────────────────────┘    └─────────────────────┘
        │                            │
        ▼                            ▼
Client দের কোনো Delay হয় না    RDB ফাইল সম্পূর্ণ হয়
```

`BGSAVE` Linux এর **Copy-on-Write (COW)** Feature ব্যবহার করে (Module 3-এ Snapshot Backup এ আলোচিত একই Technique) — Fork হওয়া মুহূর্তে Child Process Parent এর Memory এর একটা Reference পায় (Actual Copy না), এবং শুধু যেসব Memory Page পরিবর্তন হয় সেগুলোর জন্যই নতুন Copy তৈরি হয়। ফলে Client দের Request Serve করায় কোনো বিঘ্ন ঘটে না।

### সুবিধা ও অসুবিধা

| সুবিধা | অসুবিধা |
|---|---|
| ছোট, Compact Binary ফাইল | Snapshot এর মধ্যে সময়ের ব্যবধানে Data Loss হতে পারে (যেমন প্রতি ৫ মিনিটে Save হলে, শেষ ৫ মিনিটের ডেটা হারাতে পারে) |
| Restore দ্রুত | BGSAVE চলাকালীন Memory Usage সাময়িক বাড়ে (Copy-on-Write এর কারণে) |
| Disaster Recovery এর জন্য উপযুক্ত (Compact Backup) | Real-time Durability দেয় না |

---

## 9.3 AOF (Append Only File)

### সংজ্ঞা

**AOF (Append Only File)** হলো Redis এর আরেকটা Persistence পদ্ধতি, যেখানে প্রতিটা Write Operation (SET, DEL, EXPIRE ইত্যাদি) ক্রমানুসারে একটা Log ফাইলে Append (যোগ) করা হয় — এটা Module 5-এ শেখা MySQL Binlog বা Module 6-এ শেখা PostgreSQL WAL এর সাথে ধারণাগতভাবে সাদৃশ্যপূর্ণ।

### AOF Configuration

```ini
# redis.conf

appendonly yes
appendfilename "appendonly.aof"
appenddirname "appendonlydir"

# fsync Policy (কতটা Aggressive ভাবে Disk এ Write করবে)
appendfsync everysec   # প্রতি সেকেন্ডে Disk এ fsync (Default, ভালো Balance)
# appendfsync always   # প্রতিটা Write এর সাথে সাথে fsync (সবচেয়ে নিরাপদ, কিন্তু ধীর)
# appendfsync no       # OS এর উপর ছেড়ে দেওয়া (দ্রুততম, কিন্তু ঝুঁকিপূর্ণ)
```

### `appendfsync` এর তিনটা Policy — Trade-off

```
Diagram: appendfsync Policy Trade-off

always:     প্রতিটা Write ────► তাৎক্ষণিক Disk এ fsync
            (RPO ≈ 0, কিন্তু সবচেয়ে ধীর Performance)

everysec:   প্রতিটা Write ────► ১ সেকেন্ড পর পর Batch এ fsync
            (RPO ≈ 1 সেকেন্ড, ভালো Performance — Default সুপারিশ)

no:         প্রতিটা Write ────► OS নিজে যখন ইচ্ছা fsync করবে
            (RPO অনির্দিষ্ট, দ্রুততম কিন্তু সবচেয়ে ঝুঁকিপূর্ণ)
```

Module 4-এ শেখা RPO ধারণা এখানে সরাসরি প্রযোজ্য — `appendfsync` Policy বেছে নেওয়াটাই মূলত তোমার Redis Instance এর জন্য RPO নির্ধারণ করে দেয়।

### AOF Rewrite (গুরুত্বপূর্ণ Optimization)

সময়ের সাথে সাথে AOF ফাইল অনেক বড় হয়ে যেতে পারে (একই Key বারবার পরিবর্তন হলে প্রতিটা পরিবর্তন Log হতে থাকে)। **AOF Rewrite** এই Log কে Compact করে — শুধু প্রতিটা Key এর সর্বশেষ State রেখে পুরনো, Redundant Entry সরিয়ে দেয়।

```bash
# Manual AOF Rewrite চালানো
redis-cli BGREWRITEAOF
```

```
Diagram: AOF Rewrite কীভাবে কাজ করে

আগে (AOF ফাইলে):
SET counter 1
SET counter 2
SET counter 3
INCR counter    (এখন counter = 4)

BGREWRITEAOF এর পর:
SET counter 4    (শুধু Final State রাখা হয়, ফাইল অনেক ছোট হয়ে যায়)
```

### RDB vs AOF — বিস্তারিত তুলনা

| বৈশিষ্ট্য | RDB | AOF |
|---|---|---|
| Data Loss ঝুঁকি | বেশি (Snapshot Interval অনুযায়ী) | কম (প্রায় প্রতি সেকেন্ডে Save) |
| ফাইল সাইজ | ছোট, Compact | বড় (Rewrite ছাড়া) |
| Restore গতি | দ্রুত | তুলনামূলক ধীর (সব Command Replay করতে হয়) |
| Performance Impact | Snapshot নেওয়ার সময় সাময়িক | প্রতিটা Write এ কিছুটা Overhead |
| Human-readable | না (Binary) | আংশিক (Command History আকারে) |

---

## 9.4 Hybrid Persistence (RDB + AOF)

### সংজ্ঞা

Redis 4.0+ থেকে **Hybrid Persistence** সমর্থন করে, যেখানে RDB এবং AOF একসাথে ব্যবহার করা হয় — সবচেয়ে ভালো উভয়ের সুবিধা একসাথে পাওয়ার জন্য।

### কীভাবে কাজ করে

```ini
# redis.conf
aof-use-rdb-preamble yes
```

Hybrid Mode এ, AOF Rewrite হওয়ার সময় ফাইলের শুরুতে একটা RDB Format এ (Compact, Binary) সম্পূর্ণ Dataset এর Snapshot রাখা হয়, তারপর তার পরের সব পরিবর্তন AOF (Command Log) Format এ Append হতে থাকে।

```
Diagram: Hybrid Persistence File Structure

┌─────────────────────────────────────────────┐
│           appendonly.aof.1.base.rdb              │  ◄── RDB Format এ
│      (RDB Format এ Compact Snapshot)             │      Compact Base
├─────────────────────────────────────────────┤
│           appendonly.aof.1.incr.aof               │  ◄── এরপর থেকে সব
│      (তারপর থেকে সব Command, AOF Format এ)         │      নতুন Command
└─────────────────────────────────────────────┘

সুবিধা:
- Restore দ্রুত (RDB Base থেকে দ্রুত Load হয়)
- Data Loss কম (তারপরের সব Command AOF এ Log থাকে)
```

### সুবিধা

| সুবিধা | ব্যাখ্যা |
|---|---|
| দ্রুত Restore | RDB Base ব্যবহার করে দ্রুত Load হয় |
| কম Data Loss | AOF অংশ Recent Changes সংরক্ষণ করে |
| ছোট ফাইল সাইজ | RDB Compact Base + শুধু সাম্প্রতিক AOF Entry |

### Production সুপারিশ

Production এ Redis কে যদি শুধু Cache হিসেবে ব্যবহার করো (ডেটা হারালে Database থেকে আবার Regenerate করা যায়), তাহলে শুধু RDB যথেষ্ট। কিন্তু যদি Redis কে Primary Data Store (যেমন Session Store, Queue, বা Real-time Leaderboard) হিসেবে ব্যবহার করো যেখানে Data Loss গুরুতর সমস্যা, তাহলে Hybrid Persistence (RDB + AOF) ব্যবহার করা উচিত।

---

## 9.5 Restore Process

### RDB থেকে Restore করা

```bash
# ধাপ ১: Redis Server বন্ধ করা
systemctl stop redis

# ধাপ ২: dump.rdb ফাইল সঠিক জায়গায় রাখা
cp /backup/dump.rdb /var/lib/redis/dump.rdb
chown redis:redis /var/lib/redis/dump.rdb

# ধাপ ৩: Redis চালু করা (এটা Startup এর সময় স্বয়ংক্রিয়ভাবে dump.rdb Load করবে)
systemctl start redis
```

### AOF থেকে Restore করা

```bash
# ধাপ ১: Redis Server বন্ধ করা
systemctl stop redis

# ধাপ ২: AOF ফাইল সঠিক জায়গায় রাখা
cp -R /backup/appendonlydir /var/lib/redis/

# ধাপ ৩: AOF ফাইলের Integrity Check করা (Corrupt কিনা)
redis-check-aof /var/lib/redis/appendonlydir/appendonly.aof.1.incr.aof

# ধাপ ৪: Redis চালু করা
systemctl start redis
```

### RDB ফাইলের Integrity Check

```bash
redis-check-rdb /var/lib/redis/dump.rdb
```

### Restore Verification

```bash
# Redis চালু হওয়ার পর ডেটা Verify করা
redis-cli DBSIZE          # কতগুলো Key আছে দেখা
redis-cli GET some_key    # নির্দিষ্ট Key এর Value চেক করা
redis-cli INFO persistence  # Persistence সম্পর্কিত তথ্য
```

---

## 9.6 Production Backup Strategy

### সম্পূর্ণ Automation Script

```bash
#!/bin/bash
# /opt/scripts/redis_backup.sh

BACKUP_DIR="/backup/redis"
DATE=$(date +%Y-%m-%d_%H-%M-%S)
RETENTION_DAYS=14
REDIS_DATA_DIR="/var/lib/redis"
LOG_FILE="/var/log/redis_backup.log"

mkdir -p "$BACKUP_DIR"

echo "[$(date)] Redis Backup শুরু হচ্ছে..." >> "$LOG_FILE"

# BGSAVE চালানো (Non-blocking)
redis-cli BGSAVE

# BGSAVE শেষ হওয়া পর্যন্ত অপেক্ষা করা
while [ "$(redis-cli INFO persistence | grep rdb_bgsave_in_progress | cut -d: -f2 | tr -d '\r')" = "1" ]; do
    sleep 1
done

# RDB ফাইল Copy করা
cp "$REDIS_DATA_DIR/dump.rdb" "$BACKUP_DIR/dump_$DATE.rdb"

if [ $? -eq 0 ]; then
    echo "[$(date)] Backup সফল: dump_$DATE.rdb" >> "$LOG_FILE"
    gzip "$BACKUP_DIR/dump_$DATE.rdb"
else
    echo "[$(date)] ERROR: Backup ব্যর্থ হয়েছে!" >> "$LOG_FILE"
    exit 1
fi

# পুরনো Backup Cleanup
find "$BACKUP_DIR" -name "*.rdb.gz" -mtime +$RETENTION_DAYS -delete

echo "[$(date)] Redis Backup সম্পন্ন।" >> "$LOG_FILE"
```

### Cron Job

```bash
# প্রতি ৬ ঘণ্টা পর পর
0 */6 * * * /opt/scripts/redis_backup.sh
```

### Production Architecture উদাহরণ

```
┌───────────────────────────────────────────────┐
│                Redis Production Setup             │
│                                                    │
│  ┌──────────────┐   Replication  ┌──────────────┐ │
│  │  Redis Master    │───────────────►│  Redis Replica   │ │
│  │  (Read + Write)   │                │  (Backup Source) │ │
│  └──────────────┘                └───────┬──────┘ │
│                                             │        │
│                          BGSAVE + AOF এখানে         │
│                          Master এর Load কমাতে       │
└─────────────────────────────────────────────┼──────┘
                                               │
                                    Compress + Cloud Upload
                                               │
                                               ▼
                                  ┌─────────────────────┐
                                  │  Cloud Storage           │
                                  │  (14 দিন Retention)      │
                                  └─────────────────────┘
```

Module 5 ও 8-এ শেখা "Replica থেকে Backup নেওয়া" ধারণাটা এখানেও একইভাবে প্রযোজ্য — Redis Replica থেকে BGSAVE চালালে Master এর Performance এ প্রভাব পড়ে না।

---

## 9.7 Best Practices

1. যদি Redis শুধু Cache হিসেবে ব্যবহার হয়, RDB যথেষ্ট। Primary Data Store হলে Hybrid Persistence (RDB+AOF) ব্যবহার করো।
2. `BGSAVE` ব্যবহার করো, কখনো Production এ `SAVE` (Blocking) চালিও না।
3. Redis Replica থেকে Backup নাও, Master এর Load কমাতে।
4. AOF ব্যবহার করলে নিয়মিত `BGREWRITEAOF` চালাও ফাইল সাইজ নিয়ন্ত্রণে রাখতে।
5. Backup Restore করার আগে সবসময় `redis-check-rdb`/`redis-check-aof` দিয়ে Integrity Verify করো।
6. Memory Usage Monitor করো — `BGSAVE`/Copy-on-Write এর কারণে সাময়িক Memory প্রয়োজন বেড়ে যায়, তাই পর্যাপ্ত RAM Buffer রাখতে হবে।

---

## 9.8 Common Mistakes

| ভুল | সমস্যা |
|---|---|
| Persistence সম্পূর্ণ বন্ধ রাখা Primary Data Store এ | Server Restart হলে সব ডেটা হারিয়ে যায় |
| Production এ `SAVE` (Blocking) Command ব্যবহার করা | বড় Dataset এ Redis সাময়িক সম্পূর্ণ থেমে যেতে পারে |
| Memory এর তুলনায় Disk Space কম রাখা | BGSAVE/AOF Rewrite এর সময় প্রয়োজনীয় Extra Memory/Disk না থাকলে সমস্যা হতে পারে |
| AOF Rewrite কখনো না করা | ফাইল ক্রমশ অনেক বড় হয়ে Disk পূর্ণ করে ফেলতে পারে |
| শুধু RDB ব্যবহার করা Critical Data এর জন্য | Snapshot Interval এর মধ্যে হওয়া সব পরিবর্তন হারিয়ে যেতে পারে |

---

## 9.9 Interview Questions (Module 9)

1. **RDB এবং AOF এর মধ্যে পার্থক্য কী এবং কখন কোনটা ব্যবহার করবে?**
   *উত্তর:* RDB হলো Snapshot-based Persistence (নির্দিষ্ট সময় অন্তর সম্পূর্ণ Dataset Save করে) — Cache Use-case এর জন্য যথেষ্ট। AOF হলো Log-based Persistence (প্রতিটা Write Command Log করে) — Primary Data Store হিসেবে Redis ব্যবহার করলে কম Data Loss এর জন্য প্রয়োজন।

2. **BGSAVE কীভাবে Client দের Block না করে কাজ করে?**
   *উত্তর:* Redis একটা Child Process Fork করে যা Copy-on-Write (COW) Technique ব্যবহার করে Memory এর একটা Reference নিয়ে RDB ফাইলে Write করে, এই সময় Parent Process স্বাভাবিকভাবে Client Request Serve করতে থাকে।

3. **`appendfsync always`, `everysec`, এবং `no` এর মধ্যে Trade-off কী?**
   *উত্তর:* `always` সবচেয়ে নিরাপদ (RPO প্রায় শূন্য) কিন্তু ধীর Performance দেয়। `everysec` ভালো Balance দেয় (RPO প্রায় ১ সেকেন্ড)। `no` দ্রুততম কিন্তু সবচেয়ে বেশি Data Loss এর ঝুঁকি থাকে, কারণ OS নিজে যখন ইচ্ছা fsync করে।

4. **Hybrid Persistence কী এবং এটা কীভাবে RDB ও AOF এর সেরা দিক একত্র করে?**
   *উত্তর:* Hybrid Persistence এ AOF ফাইলের শুরুতে RDB Format এ একটা Compact Snapshot থাকে, তারপর সাম্প্রতিক পরিবর্তনগুলো AOF Command Log আকারে থাকে — ফলে RDB এর দ্রুত Restore ক্ষমতা এবং AOF এর কম Data Loss উভয় সুবিধা পাওয়া যায়।

5. **কেন Redis Replica থেকে Backup নেওয়া উচিত, Master থেকে না?**
   *উত্তর:* যাতে BGSAVE/Backup Process এর কারণে Master Node এর Memory/CPU Usage বেড়ে গিয়ে Live Traffic এর Performance এ প্রভাব না পড়ে।

---

## 9.10 Summary (সারাংশ)

- Redis একটা In-memory Database, তাই Persistence (RDB/AOF) ছাড়া Server Restart হলে সব ডেটা হারিয়ে যেতে পারে।
- **RDB** Snapshot-based, Compact, দ্রুত Restore কিন্তু কিছু Data Loss ঝুঁকি থাকে।
- **AOF** Command Log-based, কম Data Loss কিন্তু ফাইল বড় হতে পারে (Rewrite দিয়ে সমাধান)।
- **Hybrid Persistence** উভয়ের সুবিধা একত্র করে — Production এ Critical Data এর জন্য সুপারিশকৃত।
- `BGSAVE` Copy-on-Write ব্যবহার করে Client দের Block না করেই Backup নেয়।
- `appendfsync` Policy আসলে তোমার Redis Instance এর RPO নির্ধারণ করে দেয়।
- Replica থেকে Backup নেওয়া, Regular Restore Testing, এবং Integrity Check করা — এই Best Practice গুলো Redis Backup Strategy এর মূল ভিত্তি।

---

## 9.11 Exercises & Assignments

### Exercise 1
তোমার Local Redis এ RDB এবং AOF দুটোই Enable করে একটা Hybrid Persistence Setup তৈরি করো, তারপর কিছু ডেটা Insert করে Redis Restart করে দেখো ডেটা টিকে থাকছে কিনা।

### Exercise 2
`BGSAVE` চালিয়ে `redis-cli INFO persistence` দিয়ে Progress Monitor করো, এবং `rdb_last_bgsave_status` ফিল্ড চেক করে বুঝো Backup সফল হয়েছে কিনা।

### Mini Project
Section 9.6 এর Backup Script কে আরও Improve করো:
- AOF Backup ও যোগ করো (শুধু RDB না)
- একটা Health Check যোগ করো যা `redis-check-rdb` দিয়ে প্রতিটা Backup Verify করবে
- Backup Fail হলে Notification পাঠানোর Logic যোগ করো

---

**পরবর্তী চ্যাপ্টার:** `10-Cloud-Database-Backup.md` — এখানে আমরা AWS RDS, Google Cloud SQL, Azure SQL, Supabase, PlanetScale, Neon, Railway, Render, এবং DigitalOcean এর Automatic Backup Feature নিয়ে শিখবো।
