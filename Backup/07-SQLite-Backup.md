# Module 7: SQLite Backup (এসকিউলাইট ব্যাকআপ)

> **Level:** Beginner → Intermediate
> **Prerequisite:** Module 1-4 (বিশেষ করে Transaction, ACID, Consistency ধারণা)।

---

## 7.0 Chapter Roadmap

```
7.1  SQLite এর বিশেষত্ব (কেন এটা আলাদা)
7.2  File Copy পদ্ধতি
7.3  VACUUM INTO
7.4  SQLite Backup API
7.5  Production Considerations
7.6  Django/Flutter Context এ SQLite Backup
7.7  Best Practices
7.8  Common Mistakes
7.9  Interview Questions
7.10 Summary
7.11 Exercises & Assignments
```

---

## 7.1 SQLite এর বিশেষত্ব (কেন এটা আলাদা)

### সংজ্ঞা ও পার্থক্য

MySQL/PostgreSQL এর মতো SQLite একটা **Client-Server Database না** — এটা একটা **Serverless, File-based Database**। সম্পূর্ণ Database একটা মাত্র ফাইলে (`.db`, `.sqlite`, `.sqlite3`) সংরক্ষিত থাকে, এবং কোনো আলাদা Database Server Process চালাতে হয় না।

```
Diagram: SQLite vs MySQL/PostgreSQL Architecture

MySQL/PostgreSQL:
Application ──► Network/Socket ──► Database Server Process ──► Disk Files
                                    (mysqld/postgres চলমান)

SQLite:
Application ──► SQLite Library (একই Process এর ভেতরে) ──► একটা মাত্র .db ফাইল
                (কোনো আলাদা Server Process নেই)
```

### কেন এটা Backup এর ক্ষেত্রে গুরুত্বপূর্ণ পার্থক্য তৈরি করে

যেহেতু পুরো Database একটা মাত্র ফাইল, Backup নেওয়ার সবচেয়ে সহজ উপায় মনে হতে পারে "শুধু ফাইলটা Copy করে ফেলো" — কিন্তু এটা যদি Application চলমান অবস্থায় (Write হচ্ছে এমন সময়) করা হয়, তাহলে Corrupt Backup হতে পারে, ঠিক যেমন Module 2-এ শেখা হয়েছিল যে চলমান Database এর ফাইল সরাসরি Copy করা ঝুঁকিপূর্ণ।

### Django/Flutter এ SQLite এর প্রচলিত ব্যবহার

- **Django:** Development Environment এ Default Database হিসেবে (`db.sqlite3`)।
- **Flutter:** Mobile App এর Local Storage/Offline Data এর জন্য (`sqflite` Package দিয়ে)।

---

## 7.2 File Copy পদ্ধতি

### সাধারণ (কিন্তু ঝুঁকিপূর্ণ) পদ্ধতি

```bash
# সবচেয়ে সহজ পদ্ধতি — কিন্তু সতর্কতা প্রয়োজন
cp db.sqlite3 db_backup_$(date +%Y%m%d).sqlite3
```

### কেন এটা ঝুঁকিপূর্ণ হতে পারে

SQLite এ Write Operation চলাকালীন সময়ে একটা অস্থায়ী **WAL ফাইল** (`db.sqlite3-wal`) বা **Journal ফাইল** (`db.sqlite3-journal`) তৈরি হতে পারে (SQLite এর নিজস্ব WAL Mode থাকে, যা Module 2-এ শেখা WAL এর একই ধারণার প্রয়োগ)। যদি তুমি শুধু মূল `.db` ফাইল Copy করো, কিন্তু `-wal` বা `-journal` ফাইল Copy না করো (বা ভুল সময়ে করো), তাহলে Backup Inconsistent হতে পারে।

```
Diagram: SQLite WAL Mode এ ফাইল Structure

db.sqlite3           ◄── মূল Data ফাইল
db.sqlite3-wal        ◄── এখনো Commit না হওয়া পরিবর্তনসমূহ
db.sqlite3-shm         ◄── Shared Memory Index ফাইল (WAL এর জন্য)

যদি শুধু db.sqlite3 Copy করো এবং -wal ফাইলের ভেতরের
পরিবর্তনগুলো Copy না করো, তাহলে সেই পরিবর্তনগুলো
Backup এ অনুপস্থিত থাকবে (Data Loss)
```

### নিরাপদ File Copy পদ্ধতি (Application বন্ধ রেখে — Cold Backup)

```bash
# ধাপ ১: Application/Process বন্ধ করা (যাতে কোনো Write না হয়)
# ধাপ ২: সব সংশ্লিষ্ট ফাইল একসাথে Copy করা
cp db.sqlite3 db.sqlite3-wal db.sqlite3-shm /backup/ 2>/dev/null

# অথবা, নিরাপদ পদ্ধতি: SQLite এর নিজস্ব .backup Command ব্যবহার করা
sqlite3 db.sqlite3 ".backup /backup/db_backup.sqlite3"
```

### `.backup` Command — সঠিক পদ্ধতি

SQLite CLI এর ভেতরের `.backup` Command আসলে Application চলমান অবস্থাতেও নিরাপদে Backup নিতে পারে, কারণ এটা Internal ভাবে SQLite Backup API ব্যবহার করে (7.4 এ বিস্তারিত)।

```bash
sqlite3 /path/to/db.sqlite3 ".backup '/backup/db_backup.sqlite3'"
```

---

## 7.3 VACUUM INTO

### সংজ্ঞা

`VACUUM INTO` হলো SQLite এর একটা Command (SQLite 3.27.0+ থেকে উপলব্ধ), যা Database কে **Optimize (Vacuum) করে একটা নতুন ফাইলে Copy** করে — এই প্রক্রিয়ায় Backup ফাইল থেকে Deleted/Unused Space সরে যায়, ফলে Backup ফাইল আসল ফাইলের চেয়ে ছোট এবং পরিষ্কার হয়।

### ব্যবহার

```sql
-- SQLite CLI বা Application এর মধ্য থেকে
VACUUM INTO '/backup/db_backup_clean.sqlite3';
```

```bash
# Command Line থেকে
sqlite3 db.sqlite3 "VACUUM INTO '/backup/db_backup_clean.sqlite3'"
```

### VACUUM INTO এর সুবিধা

| সুবিধা | ব্যাখ্যা |
|---|---|
| Consistent Backup | Transaction-Safe, Application চলমান অবস্থাতেও নিরাপদ |
| ছোট Backup সাইজ | Fragmented/Unused Space Remove হয় |
| Optimized Structure | Restore করার পর Database Performance ভালো থাকে |
| একক Command | সহজ, কোনো External Tool লাগে না |

### `.backup` vs `VACUUM INTO` — পার্থক্য

| বৈশিষ্ট্য | `.backup` Command | `VACUUM INTO` |
|---|---|---|
| Output ফাইল সাইজ | মূল ফাইলের সমান (Fragmentation সহ) | ছোট, Optimized |
| গতি | দ্রুত | তুলনামূলক ধীর (Vacuum করতে সময় লাগে) |
| ব্যবহার | নিয়মিত/দ্রুত Backup | সাপ্তাহিক Deep Backup, Storage বাঁচাতে |

---

## 7.4 SQLite Backup API

### সংজ্ঞা

SQLite এর একটা **C-level Backup API** আছে (`sqlite3_backup_init()`, `sqlite3_backup_step()`, `sqlite3_backup_finish()`), যা প্রোগ্রামগতভাবে (Python, C, ইত্যাদি ভাষা থেকে) Safe Backup নেওয়ার সুযোগ দেয়। এই API-ই মূলত `.backup` Command এর ভেতরে ব্যবহৃত হয়।

### Internal Working

```
Diagram: Backup API কীভাবে কাজ করে

sqlite3_backup_init() ── Source ও Destination Database Connect করে
        │
        ▼
sqlite3_backup_step(N) ── ধাপে ধাপে N পৃষ্ঠা (Page) Copy করে
        │                  (একসাথে সব Copy না করে অংশে অংশে করা যায়,
        │                   যাতে বড় Database এ Application Block না হয়)
        ▼
sqlite3_backup_finish() ── Backup সম্পূর্ণ ও Connection বন্ধ করা
```

### Python এ ব্যবহার (Django Context এ প্রাসঙ্গিক)

```python
import sqlite3

def backup_sqlite_db(source_path, backup_path):
    """
    SQLite Backup API ব্যবহার করে নিরাপদে Backup নেওয়া,
    Application চলমান অবস্থাতেও।
    """
    source_conn = sqlite3.connect(source_path)
    backup_conn = sqlite3.connect(backup_path)

    with backup_conn:
        # progress=None দিলে একবারেই সব Copy হবে
        # progress কল-ব্যাক দিলে ধাপে ধাপে (বড় DB এর জন্য ভালো)
        source_conn.backup(backup_conn, pages=100, progress=print_progress)

    source_conn.close()
    backup_conn.close()

def print_progress(status, remaining, total):
    print(f"Backup Progress: {total - remaining}/{total} pages সম্পন্ন হয়েছে")

# ব্যবহার
backup_sqlite_db("db.sqlite3", "/backup/db_backup.sqlite3")
```

### কেন এই পদ্ধতি সবচেয়ে নিরাপদ

Python এর `sqlite3.Connection.backup()` Method (Python 3.7+) সরাসরি SQLite এর C-level Backup API ব্যবহার করে, যা:
1. Application চলমান অবস্থাতেও নিরাপদ (Live Backup)।
2. প্রতিটা পৃষ্ঠা (Page) ধাপে ধাপে Copy করে, ফলে বড় Database এও কম সময়ের জন্য Lock থাকে।
3. `-wal`/`-journal` ফাইলের সমস্যা এড়িয়ে যায়, কারণ SQLite নিজেই সব Internal Consistency সামলায়।

---

## 7.5 Production Considerations (প্রোডাকশন বিবেচনা)

### SQLite কি Production-Grade Database?

এটা একটা গুরুত্বপূর্ণ প্রশ্ন যা Senior Engineer হিসেবে বোঝা দরকার।

| পরিস্থিতি | SQLite উপযুক্ত? |
|---|---|
| Single-user Desktop Application | হ্যাঁ, চমৎকার |
| Mobile App Local Storage (Flutter/Android) | হ্যাঁ, Standard Choice |
| ছোট Website (Low Traffic) | মোটামুটি উপযুক্ত |
| High-Concurrency Web Application | না, উপযুক্ত না (একাধিক Writer এর জন্য ডিজাইন করা না) |
| Multi-server/Distributed System | না, এটা Single-file, Single-machine Database |

### WAL Mode Enable করা (Concurrency উন্নত করতে)

```sql
PRAGMA journal_mode=WAL;
```

WAL Mode Enable করলে একইসাথে একজন Writer এবং একাধিক Reader কাজ করতে পারে, যা Default Rollback Journal Mode এর চেয়ে ভালো Concurrency দেয়। কিন্তু Backup নেওয়ার সময় মনে রাখতে হবে `-wal` ফাইলটাও গুরুত্বপূর্ণ (7.2 এ আলোচিত)।

### Backup Frequency সুপারিশ

- Mobile App (Flutter): প্রতিবার App Close হওয়ার সময় বা নির্দিষ্ট সময় অন্তর Cloud এ Sync করা (Backup API ব্যবহার করে)।
- Django Development: প্রতিটা বড় পরিবর্তনের আগে Manual Backup (Git এ Commit এর মতো Practice)।
- Production SQLite (যদি ব্যবহৃত হয়): নিয়মিত Cron Job দিয়ে Automated Backup।

---

## 7.6 Django/Flutter Context এ SQLite Backup

### Django এ SQLite Backup Script

```python
# management/commands/backup_sqlite.py
# Django Custom Management Command

from django.core.management.base import BaseCommand
from django.conf import settings
import sqlite3
import os
from datetime import datetime

class Command(BaseCommand):
    help = 'SQLite Database এর একটা নিরাপদ Backup নেয়'

    def handle(self, *args, **options):
        db_path = settings.DATABASES['default']['NAME']
        backup_dir = os.path.join(settings.BASE_DIR, 'backups')
        os.makedirs(backup_dir, exist_ok=True)

        timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
        backup_path = os.path.join(backup_dir, f'backup_{timestamp}.sqlite3')

        source_conn = sqlite3.connect(db_path)
        backup_conn = sqlite3.connect(backup_path)

        with backup_conn:
            source_conn.backup(backup_conn)

        source_conn.close()
        backup_conn.close()

        self.stdout.write(
            self.style.SUCCESS(f'Backup সফল হয়েছে: {backup_path}')
        )
```

```bash
# চালানো
python manage.py backup_sqlite
```

### Flutter (sqflite) এ Backup Concept

```dart
// Flutter এ SQLite (sqflite Package) Backup — File Copy পদ্ধতি
import 'dart:io';
import 'package:path_provider/path_provider.dart';
import 'package:sqflite/sqflite.dart';

Future<void> backupDatabase() async {
  // মূল Database এর Path বের করা
  String databasesPath = await getDatabasesPath();
  String dbPath = '$databasesPath/app_database.db';

  // Backup Location (App এর Document Directory তে)
  Directory appDocDir = await getApplicationDocumentsDirectory();
  String backupPath = '${appDocDir.path}/backup_${DateTime.now().millisecondsSinceEpoch}.db';

  // Database বন্ধ করে নিরাপদে Copy করা (গুরুত্বপূর্ণ ধাপ)
  Database db = await openDatabase(dbPath);
  await db.close();  // Copy করার আগে বন্ধ করা, Consistency নিশ্চিত করতে

  File dbFile = File(dbPath);
  await dbFile.copy(backupPath);

  print('Backup সম্পন্ন: $backupPath');
}
```

**গুরুত্বপূর্ণ:** Flutter Context এ Database বন্ধ করে (Close করে) তারপর Copy করা সবচেয়ে নিরাপদ পদ্ধতি, কারণ Mobile App এ সাধারণত High Concurrency থাকে না, তাই এই সাময়িক বন্ধ করাটা User Experience এ তেমন প্রভাব ফেলে না।

---

## 7.7 Best Practices

1. কখনো চলমান SQLite Database এর ফাইল সরাসরি `cp` দিয়ে Copy করো না — `.backup` Command বা Backup API ব্যবহার করো।
2. WAL Mode ব্যবহার করলে, Backup নেওয়ার সময় `-wal` এবং `-shm` ফাইলও বিবেচনা করো (যদিও Backup API/`` .backup` `` Command এই সমস্যা এড়িয়ে যায়)।
3. মাঝে মাঝে `VACUUM INTO` ব্যবহার করে Optimized Backup নাও, Storage বাঁচাতে।
4. Mobile App এ Backup নেওয়ার সময় Database Connection বন্ধ করে তারপর Copy করো।
5. Backup ফাইল Cloud এ (Google Drive/iCloud/Firebase) Sync করার ব্যবস্থা রাখো Mobile Application এ, যাতে Device হারালেও ডেটা সুরক্ষিত থাকে।

---

## 7.8 Common Mistakes

| ভুল | সমস্যা |
|---|---|
| চলমান Application এ সরাসরি ফাইল Copy করা | Corrupt/Incomplete Backup হতে পারে |
| WAL Mode এ শুধু মূল `.db` ফাইল Copy করা, `-wal` ফাইল বাদ দেওয়া | সাম্প্রতিক পরিবর্তন Backup এ অনুপস্থিত থাকতে পারে |
| SQLite কে High-Traffic Production System এ ব্যবহার করা | Concurrency সমস্যা হতে পারে, Backup Strategy আরও জটিল হয়ে যায় |
| Mobile App এ Backup না রেখে শুধু Local Storage এর উপর নির্ভর করা | Device হারালে/নষ্ট হলে সব ডেটা হারিয়ে যায় |

---

## 7.9 Interview Questions (Module 7)

1. **SQLite কেন MySQL/PostgreSQL এর চেয়ে ভিন্নভাবে Backup নিতে হয়?**
   *উত্তর:* কারণ SQLite একটা Serverless, File-based Database — পুরো Database একটা মাত্র ফাইলে থাকে এবং কোনো আলাদা Server Process নেই, তাই সরাসরি ফাইল Copy করার চেষ্টা করলে Consistency সমস্যা হতে পারে যদি সঠিক পদ্ধতি ব্যবহার না করা হয়।

2. **VACUUM INTO এবং `.backup` Command এর মধ্যে পার্থক্য কী?**
   *উত্তর:* `.backup` দ্রুত কিন্তু মূল ফাইলের সমান সাইজের (Fragmentation সহ) Backup তৈরি করে, আর `VACUUM INTO` তুলনামূলক ধীর কিন্তু Optimized, ছোট সাইজের Backup তৈরি করে (Unused Space Remove করে)।

3. **কেন সরাসরি `cp` কমান্ড দিয়ে চলমান SQLite ফাইল Copy করা ঝুঁকিপূর্ণ?**
   *উত্তর:* WAL Mode এ থাকলে কিছু সাম্প্রতিক পরিবর্তন `-wal` ফাইলে থাকতে পারে যা মূল `.db` ফাইলে এখনো Merge হয়নি, তাই শুধু `.db` ফাইল Copy করলে সেই পরিবর্তনগুলো Backup এ অনুপস্থিত থাকবে।

4. **SQLite কি High-Concurrency Production Web Application এর জন্য উপযুক্ত?**
   *উত্তর:* না, কারণ SQLite মূলত একজন Writer কে সমর্থন করার জন্য ডিজাইন করা, High-Concurrency Multi-user Web Application এর জন্য MySQL/PostgreSQL এর মতো Client-Server Database বেশি উপযুক্ত।

---

## 7.10 Summary (সারাংশ)

- SQLite একটা Serverless, File-based Database — পুরো ডেটা একটা মাত্র ফাইলে থাকে।
- সরাসরি File Copy ঝুঁকিপূর্ণ যদি Application চলমান থাকে, বিশেষ করে WAL Mode এ।
- `.backup` Command দ্রুত, নিরাপদ Backup নেয়।
- `VACUUM INTO` Optimized, ছোট সাইজের Backup তৈরি করে।
- SQLite Backup API (Python এর `sqlite3.Connection.backup()`) সবচেয়ে নিরাপদ প্রোগ্রামেটিক পদ্ধতি, ধাপে ধাপে Page Copy করে।
- Django এ Custom Management Command দিয়ে এবং Flutter এ Database বন্ধ করে Copy করার মাধ্যমে Backup Automate করা যায়।
- SQLite High-Concurrency Production System এর জন্য উপযুক্ত না — এটা মূলত Single-user/Mobile/Development Context এ ব্যবহৃত হয়।

---

## 7.11 Exercises & Assignments

### Exercise 1
তোমার Django প্রজেক্টে Section 7.6 এর Management Command তৈরি করে চালিয়ে দেখো, Backup ফাইল সঠিকভাবে তৈরি হচ্ছে কিনা যাচাই করো।

### Exercise 2
`VACUUM INTO` এবং `.backup` দিয়ে একই Database এর দুইটা Backup নিয়ে ফাইল সাইজ তুলনা করো।

### Mini Project
একটা Flutter Sample App এ Section 7.6 এর কোড ব্যবহার করে "Backup Now" একটা Button তৈরি করো, যা Backup নিয়ে Device এর Document Directory তে Save করবে এবং User কে একটা Success Message দেখাবে।

---

**পরবর্তী চ্যাপ্টার:** `08-MongoDB-Backup.md` — এখানে আমরা mongodump, mongorestore, Filesystem Snapshot, Replica Set Backup, Sharding Backup, এবং Cloud Backup নিয়ে শিখবো।
