# MODULE 17: Interview Preparation (ইন্টারভিউ প্রস্তুতি)
## বাংলায় সম্পূর্ণ Database Interview গাইড — Beginner to FAANG Level

> **কেন এই module দরকার?**
> তুমি সব theory জানো — কিন্তু interview এ কীভাবে present করবে?
> FAANG companies এ database interview এ শুধু SQL জানলে হয় না।
> System design, trade-offs, real scenarios — সব জানতে হয়।
> এই module এ প্রতিটা topic এর interview questions + model answers আছে।

---

## 17.1 Database Fundamentals — Interview Questions

### Beginner Level

**Q1: Primary Key এবং Unique Key এর পার্থক্য কী?**
```
Primary Key:
  - একটা table এ শুধু একটাই থাকে
  - NULL value হতে পারে না
  - Clustered index তৈরি করে (PostgreSQL এ heap based, MySQL InnoDB তে clustered)
  - একটা table এর identity

Unique Key:
  - একটা table এ একাধিক থাকতে পারে
  - NULL হতে পারে (এবং multiple NULLs allowed)
  - Non-clustered index তৈরি করে
  - Duplicate prevent করে কিন্তু primary identifier না

Example:
  users(id PRIMARY KEY, email UNIQUE, phone UNIQUE)
  id = primary key (identity)
  email, phone = unique keys (business constraints)
```

**Q2: ACID কী? প্রতিটা property explain করো।**
```
A — Atomicity:
  Transaction সম্পূর্ণ হবে অথবা কিছুই হবে না।
  Bank transfer: debit + credit — দুটোই হবে অথবা দুটোই rollback।

C — Consistency:
  Transaction এর পরেও database valid state এ থাকবে।
  Constraints, rules সবসময় valid থাকবে।
  Account balance negative হবে না (CHECK constraint)।

I — Isolation:
  Concurrent transactions একে অপরকে affect করবে না।
  একটা transaction এর intermediate state অন্যটা দেখবে না।

D — Durability:
  Commit হওয়া transaction permanent।
  Server crash হলেও data হারাবে না।
  WAL (Write-Ahead Log) এটা নিশ্চিত করে।
```

**Q3: Index কী এবং কীভাবে কাজ করে?**
```
Index = database এর book এর index এর মতো।
Without index: full table scan — O(n)
With index: B-tree traversal — O(log n)

B-Tree Index internally:
  Root node → Branch nodes → Leaf nodes (actual row pointers)
  
  Balance tree, তাই সবসময় O(log n) search।
  
  CREATE INDEX idx_users_email ON users(email);
  
  SELECT * FROM users WHERE email = 'test@test.com';
  -- Without index: scan all 1M rows
  -- With index: traverse B-tree, 20 comparisons for 1M rows

Cost:
  ✅ Fast reads
  ❌ Slow writes (index update করতে হয়)
  ❌ Extra storage
```

---

### Intermediate Level

**Q4: Normalization কী? 1NF, 2NF, 3NF explain করো।**
```
Normalization = redundancy কমানো, data integrity বাড়ানো।

1NF (First Normal Form):
  - প্রতিটা column এ atomic value (no arrays, no multiple values)
  - প্রতিটা row unique (primary key আছে)
  
  ❌ Violates 1NF:
  orders(id, items="phone,laptop,tablet")
  
  ✅ 1NF:
  order_items(order_id, item_name)

2NF (Second Normal Form):
  - 1NF satisfied
  - Non-key columns সম্পূর্ণ primary key এর উপর dependent (no partial dependency)
  
  ❌ Violates 2NF:
  order_items(order_id, product_id, product_name, qty)
  product_name depends only on product_id (partial dependency)
  
  ✅ 2NF:
  order_items(order_id, product_id, qty)
  products(product_id, product_name)

3NF (Third Normal Form):
  - 2NF satisfied
  - Non-key columns শুধু primary key এর উপর dependent (no transitive dependency)
  
  ❌ Violates 3NF:
  employees(id, dept_id, dept_name)
  dept_name → dept_id → id (transitive)
  
  ✅ 3NF:
  employees(id, dept_id)
  departments(dept_id, dept_name)
```

**Q5: Transaction Isolation Levels কী কী এবং কোনটা কখন use করবে?**
```
4 Isolation Levels (weakest to strongest):

READ UNCOMMITTED:
  Dirty reads possible।
  T1 এর uncommitted change T2 দেখতে পারে।
  Use: Almost never। Performance critical analytics যেখানে dirty read OK।

READ COMMITTED (PostgreSQL default):
  Dirty reads নেই।
  Non-repeatable reads possible।
  Same row দুবার read করলে different result পেতে পারো।
  Use: Most applications — good balance।

REPEATABLE READ:
  Non-repeatable reads নেই।
  Phantom reads possible (MySQL InnoDB তে নেই কিন্তু)।
  Transaction এ same query twice করলে same result।
  Use: Financial reports, balance calculations।

SERIALIZABLE:
  সবচেয়ে strict। সব anomalies বন্ধ।
  Transactions যেন sequential এ চলছে।
  Use: Banking critical operations, inventory।
  Cost: Slowest, more lock contention।

Django তে:
  from django.db import transaction
  
  with transaction.atomic():
      # Default: READ COMMITTED
      pass
  
  # SERIALIZABLE:
  from django.db import connection
  with transaction.atomic():
      connection.cursor().execute('SET TRANSACTION ISOLATION LEVEL SERIALIZABLE')
      # ... your code
```

**Q6: Deadlock কী? কীভাবে prevent করবে?**
```
Deadlock:
  T1 locks row A, waits for row B
  T2 locks row B, waits for row A
  Both wait forever → deadlock!

Detection:
  PostgreSQL automatically detects deadlocks।
  ERROR: deadlock detected

Prevention Strategy 1 — Consistent Lock Order:
  সবসময় same order এ resources lock করো।
  
  ❌ T1: lock account 101, then 202
     T2: lock account 202, then 101  → DEADLOCK!
  
  ✅ Always lock by ascending ID:
  accounts = Account.objects.select_for_update().filter(
      id__in=[101, 202]
  ).order_by('id')  # consistent order!

Prevention Strategy 2 — Short Transactions:
  Transaction যত ছোট রাখো, deadlock এর chance কম।
  Long running transactions avoid করো।

Prevention Strategy 3 — Lock Timeout:
  SET lock_timeout = '5s';
  5 seconds lock না পেলে error throw করবে।

Prevention Strategy 4 — Optimistic Locking:
  Lock না নিয়ে version check করো।
  version field: update করার সময় check করো।
```

---

### Senior Level

**Q7: MVCC (Multi-Version Concurrency Control) কীভাবে কাজ করে?**
```
MVCC = locks ছাড়া concurrency।
PostgreSQL, Oracle ব্যবহার করে।

How it works:
  প্রতিটা row এর multiple versions থাকে।
  
  Row: {id:1, name:"Alice", xmin:100, xmax:null}
  xmin = transaction id যে এই version তৈরি করেছে
  xmax = transaction id যে এই version delete/update করেছে

  T1 (txid=200) updates row 1:
    Old version: {id:1, name:"Alice", xmin:100, xmax:200}
    New version: {id:1, name:"Alice updated", xmin:200, xmax:null}

  T2 (txid=150, started before T1):
    Sees old version! (xmin:100 < 150 < xmax:200)
    T2 doesn't get blocked।

Benefits:
  ✅ Readers don't block writers
  ✅ Writers don't block readers
  ✅ High concurrency

Cost:
  ❌ Dead tuples accumulate (old versions)
  ❌ VACUUM needed to clean up dead tuples
  ❌ Table bloat if VACUUM not running
```

**Q8: B-Tree vs Hash Index — কখন কোনটা?**
```
B-Tree Index:
  Supports: =, <, >, <=, >=, BETWEEN, LIKE 'abc%'
  Use for: Most cases — range queries, sorting
  
  CREATE INDEX idx_orders_date ON orders(created_at);
  SELECT * FROM orders WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31';
  -- B-Tree সবচেয়ে effective

Hash Index:
  Supports: = only (exact match)
  Faster than B-Tree for exact lookups
  Cannot do range queries, sorting
  
  CREATE INDEX idx_sessions_token ON sessions USING HASH (token);
  SELECT * FROM sessions WHERE token = 'abc123xyz';
  -- Hash এ perfect — exact match only

Real usage:
  99% of the time: B-Tree
  Hash: session tokens, UUID lookups, exact match only columns
  
  PostgreSQL এ Hash index WAL logged হয় না (before PG10)
  PG10+ এ WAL logged — production safe
```

---

## 17.2 SQL — Interview Questions

### Beginner Level

**Q9: JOIN types এর পার্থক্য explain করো।**
```sql
-- INNER JOIN: দুই table এ matching rows only
SELECT o.id, u.name
FROM orders o
INNER JOIN users u ON o.user_id = u.id;
-- শুধু order আছে এমন users

-- LEFT JOIN: left table এর সব rows + matching right rows
SELECT u.name, o.id
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;
-- সব users, order না থাকলে NULL

-- RIGHT JOIN: right table এর সব rows (rarely used, use LEFT JOIN instead)

-- FULL OUTER JOIN: দুই table এর সব rows
SELECT u.name, o.id
FROM users u
FULL OUTER JOIN orders o ON u.id = o.user_id;
-- সব users এবং সব orders

-- CROSS JOIN: Cartesian product (m × n rows)
SELECT u.name, p.name
FROM users u CROSS JOIN products p;
-- Every user with every product combination

-- SELF JOIN: same table এর সাথে join
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```

**Q10: GROUP BY এবং HAVING এর পার্থক্য কী?**
```sql
-- WHERE: individual rows filter করে (before grouping)
-- HAVING: groups filter করে (after grouping)

-- ❌ Cannot use aggregate in WHERE
SELECT user_id, COUNT(*) FROM orders
WHERE COUNT(*) > 5  -- ERROR!
GROUP BY user_id;

-- ✅ Use HAVING for aggregate conditions
SELECT user_id, COUNT(*) AS order_count
FROM orders
WHERE status = 'completed'  -- row level filter (before group)
GROUP BY user_id
HAVING COUNT(*) > 5;         -- group level filter (after group)
-- completed orders যাদের 5+ আছে তাদের দেখাও
```

---

### Intermediate Level

**Q11: Window Function কী? ROW_NUMBER, RANK, DENSE_RANK এর পার্থক্য?**
```sql
-- Window Function = GROUP BY ছাড়া aggregate

-- ROW_NUMBER: unique sequential number (ties ভাঙে arbitrarily)
-- RANK: same value = same rank, next rank skipped (1,1,3)
-- DENSE_RANK: same value = same rank, next rank not skipped (1,1,2)

SELECT
    name,
    score,
    ROW_NUMBER() OVER (ORDER BY score DESC) AS row_num,
    RANK()       OVER (ORDER BY score DESC) AS rank,
    DENSE_RANK() OVER (ORDER BY score DESC) AS dense_rank
FROM students;

-- name    score  row_num  rank  dense_rank
-- Alice   95     1        1     1
-- Bob     95     2        1     1
-- Charlie 90     3        3     2   ← RANK skips 2, DENSE_RANK doesn't
-- David   85     4        4     3

-- Practical use: top N per group
SELECT * FROM (
    SELECT
        user_id, product_id, amount,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY amount DESC) AS rn
    FROM orders
) ranked
WHERE rn <= 3;
-- প্রতিটা user এর top 3 orders
```

**Q12: CTE এবং Subquery এর মধ্যে কখন কোনটা use করবে?**
```sql
-- Subquery: inline, simple cases
SELECT * FROM orders
WHERE user_id IN (
    SELECT id FROM users WHERE country = 'Bangladesh'
);

-- CTE: complex logic, reusable, readable
WITH bangladeshi_users AS (
    SELECT id FROM users WHERE country = 'Bangladesh'
),
recent_orders AS (
    SELECT * FROM orders WHERE created_at > NOW() - INTERVAL '30 days'
)
SELECT o.*
FROM recent_orders o
JOIN bangladeshi_users u ON o.user_id = u.id;

-- কখন CTE:
--   1. Same subquery একাধিকবার use করতে হলে
--   2. Complex logic readable করতে
--   3. Recursive query (tree traversal)

-- কখন Subquery:
--   1. Simple, one-time use
--   2. Correlated subquery (outer query reference করে)
--   3. EXISTS check

-- Recursive CTE (category tree):
WITH RECURSIVE category_tree AS (
    SELECT id, name, parent_id, 1 AS level
    FROM categories WHERE parent_id IS NULL  -- root

    UNION ALL

    SELECT c.id, c.name, c.parent_id, ct.level + 1
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT * FROM category_tree ORDER BY level, name;
```

**Q13: N+1 Problem কী? কীভাবে solve করবে?**
```python
# N+1 Problem:
orders = Order.objects.all()  # 1 query
for order in orders:
    print(order.user.name)    # N queries (one per order!)
# Total: 1 + N queries

# SQL generated:
# SELECT * FROM orders;
# SELECT * FROM users WHERE id = 1;
# SELECT * FROM users WHERE id = 2;
# ... (N times)

# Solution 1: select_related (INNER/LEFT JOIN)
orders = Order.objects.select_related('user').all()
# SQL: SELECT orders.*, users.* FROM orders JOIN users ON ...
# Total: 1 query

# Solution 2: prefetch_related (separate query + Python join)
orders = Order.objects.prefetch_related('items').all()
# SQL:
# SELECT * FROM orders;
# SELECT * FROM order_items WHERE order_id IN (1,2,3,...);
# Total: 2 queries

# Rule:
# select_related  → ForeignKey, OneToOne (JOIN)
# prefetch_related → ManyToMany, reverse FK (separate query)
```

---

### Senior Level

**Q14: EXPLAIN ANALYZE output কীভাবে read করবে?**
```sql
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE user_id = 101
ORDER BY created_at DESC
LIMIT 20;

-- Output:
-- Limit (cost=0.43..8.45 rows=20 width=120) (actual time=0.05..0.12 rows=20)
--   -> Index Scan Backward using idx_orders_user_created on orders
--      (cost=0.43..4523.21 rows=21847 width=120)
--      (actual time=0.04..0.11 rows=20)
--      Index Cond: (user_id = 101)
-- Planning Time: 0.3 ms
-- Execution Time: 0.15 ms

-- Reading the output:
-- cost=0.43..8.45
--   0.43 = startup cost (first row পেতে)
--   8.45 = total cost (সব rows পেতে)
-- actual time=0.05..0.12
--   0.05ms = first row, 0.12ms = total
-- rows=20 = 20 rows returned

-- Red flags:
-- Seq Scan on large table → index missing!
-- Hash Join on large tables → might need index
-- actual rows >> estimated rows → ANALYZE needed (statistics stale)
-- Nested Loop with large rows → bad join order

-- Good signs:
-- Index Scan → index being used ✅
-- Index Only Scan → covering index ✅
-- Bitmap Index Scan → multiple conditions ✅
```

**Q15: Pagination এর সমস্যা এবং solution কী?**
```sql
-- ❌ OFFSET pagination (bad for large tables)
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 1000000;
-- 1M rows skip করতে হবে! Very slow।
-- Page 50000 load করতে 30+ seconds!

-- ✅ Keyset pagination (cursor-based)
-- First page:
SELECT * FROM orders
ORDER BY created_at DESC, id DESC
LIMIT 20;
-- Last row: created_at='2024-06-15', id=5050

-- Next page (use last row's values as cursor):
SELECT * FROM orders
WHERE (created_at, id) < ('2024-06-15', 5050)
ORDER BY created_at DESC, id DESC
LIMIT 20;
-- Always fast regardless of page number! O(log n)

-- Django implementation:
def get_orders_page(cursor_created_at=None, cursor_id=None):
    qs = Order.objects.order_by('-created_at', '-id')
    if cursor_created_at and cursor_id:
        qs = qs.filter(
            models.Q(created_at__lt=cursor_created_at) |
            models.Q(created_at=cursor_created_at, id__lt=cursor_id)
        )
    return qs[:20]

-- Tradeoff:
-- OFFSET: jump to any page, but slow for large offsets
-- Keyset: always fast, but no random page access
```

---

## 17.3 PostgreSQL — Interview Questions

### Intermediate Level

**Q16: PostgreSQL VACUUM কেন দরকার?**
```
MVCC এর কারণে dead tuples accumulate হয়।

যখন row UPDATE হয়:
  Old version → dead tuple (space নেয় কিন্তু কাজে লাগে না)
  New version → live tuple

VACUUM:
  Dead tuples mark করে reusable হিসেবে।
  Space free করে (OS কে return করে না — autovacuum FULL ছাড়া)।
  Table statistics update করে (query planner এর জন্য)।

VACUUM FULL:
  Dead tuples পুরোপুরি সরায়।
  Space OS কে return করে।
  ⚠️ Table lock নেয় — production এ সাবধান!

AUTOVACUUM:
  PostgreSQL automatically VACUUM চালায়।
  Configure করো:
    autovacuum_vacuum_scale_factor = 0.1  (10% dead tuples এ trigger)
    autovacuum_analyze_scale_factor = 0.05

Check vacuum status:
  SELECT tablename, n_dead_tup, last_autovacuum
  FROM pg_stat_user_tables
  ORDER BY n_dead_tup DESC;
```

**Q17: WAL (Write-Ahead Log) কী?**
```
WAL = প্রতিটা change database এ apply হওয়ার আগে log file এ লেখা হয়।

কেন?
  Server crash হলে:
  1. WAL থেকে uncommitted transactions rollback হয়
  2. Committed কিন্তু disk এ না লেখা transactions replay হয়
  
  Data loss ছাড়া crash recovery।

WAL এর uses:
  1. Crash recovery (durability)
  2. Replication (WAL streaming to replicas)
  3. PITR (Point-in-Time Recovery)
  4. Logical replication (CDC)

WAL configuration:
  wal_level = replica    (replication এর জন্য)
  wal_level = logical    (logical replication এর জন্য)
  archive_mode = on      (WAL archiving এর জন্য)
```

---

### Senior Level

**Q18: PostgreSQL Partitioning কখন করবে এবং কীভাবে?**
```sql
-- কখন partition করবে:
-- Table > 100M rows এবং mostly range queries
-- Time-series data (logs, events, metrics)
-- Archival needed (old partitions drop করবে)

-- Types:
-- RANGE: date ranges (most common)
-- LIST: specific values (country, status)
-- HASH: even distribution

-- Example: orders table partition by month
CREATE TABLE orders (
    id          BIGSERIAL,
    user_id     BIGINT,
    total       DECIMAL(12,2),
    created_at  TIMESTAMPTZ NOT NULL
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2024_01 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE orders_2024_02 PARTITION OF orders
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Query automatically hits correct partition:
SELECT * FROM orders WHERE created_at >= '2024-01-01';
-- Only scans orders_2024_01, not all partitions!

-- Partition pruning verify করো:
EXPLAIN SELECT * FROM orders WHERE created_at >= '2024-01-01';
-- Partitions: orders_2024_01  ← only 1 partition scanned

-- Benefits:
-- Fast queries on date ranges
-- Easy archival: DROP TABLE orders_2021_01 (instant!)
-- Parallel query across partitions

-- Gotchas:
-- Primary key must include partition key
-- Foreign keys to partitioned tables limited
-- Cross-partition queries slower
```

---

## 17.4 NoSQL — Interview Questions

**Q19: Redis কোন কোন use case এ ব্যবহার করবে?**
```
Redis Data Structures and Use Cases:

STRING:
  Cache: product details, API responses
  Counter: page views, rate limiting
  Session: user session storage
  
  SET product:101 '{"name":"Phone","price":999}' EX 3600
  INCR page_views:homepage
  SETEX session:abc123 1800 user_data

LIST:
  Queue: job queue, notification queue
  Recent activity: last 10 actions
  
  LPUSH notification_queue '{"user_id":1,"msg":"Order shipped"}'
  RPOP notification_queue  # worker processes

SET:
  Unique visitors, tags, permissions
  
  SADD online_users 101 202 303
  SCARD online_users  # count
  SISMEMBER online_users 101  # check membership

SORTED SET (ZSET):
  Leaderboard, rate limiting, priority queue
  
  ZADD leaderboard 9500 "user:101"
  ZADD leaderboard 8200 "user:202"
  ZREVRANGE leaderboard 0 9  # top 10
  ZRANK leaderboard "user:202"  # rank

HASH:
  User sessions, driver locations, config
  
  HSET driver:101 lat 23.8103 lng 90.4125
  HGETALL driver:101

When NOT to use Redis:
  ❌ Primary database for important data
  ❌ Complex relational queries
  ❌ Data that must survive Redis restart (unless persistence enabled)
  ❌ Data > RAM size
```

**Q20: CAP Theorem — real interview answer কী হওয়া উচিত?**
```
CAP Theorem:
  Consistency, Availability, Partition Tolerance — তিনটার মধ্যে
  network partition এর সময় শুধু দুটো guarantee করা সম্ভব।

Network partition real world এ inevitable।
তাই real choice: CP vs AP।

CP Systems (Consistency + Partition Tolerance):
  PostgreSQL, MySQL, MongoDB (strong consistency mode)
  Network split হলে availability sacrifice।
  "Stale data দেখানোর চেয়ে error দেওয়া better।"
  Use: Banking, payments, inventory।

AP Systems (Availability + Partition Tolerance):
  Cassandra, DynamoDB, CouchDB
  Network split হলে stale data serve করে।
  Eventually consistent।
  "Error দেওয়ার চেয়ে slightly old data better।"
  Use: Social media feed, product catalog, DNS।

Interview tip:
  "CAP theorem simplifies things — real world এ
  consistency এবং availability এর spectrum আছে।
  Tunable consistency (Cassandra) দিয়ে
  per-operation trade-off করা যায়।"
```

---

## 17.5 System Design — Interview Questions

**Q21: Design করো একটা "Design Twitter/X Database" (system design interview)**
```
Step 1: Requirements Clarify করো
  Functional:
    - Post tweets (280 chars)
    - Follow/unfollow users
    - News feed
    - Like, retweet
    - Search tweets

  Non-functional:
    - 500M users, 200M DAU
    - 100M tweets/day
    - Read:Write = 100:1 (read heavy)
    - Feed latency < 200ms

Step 2: Scale Estimation
  Tweets: 100M/day = 1200/sec writes
  Feed reads: 100M DAU × 5 reads = 500M/day = 6000/sec
  Storage: 100M tweets × 300 bytes = 30GB/day

Step 3: Schema Design
  users(id, username, follower_count, tweet_count)
  tweets(id, user_id, content, like_count, retweet_count, created_at)
  follows(follower_id, followee_id, created_at)
  likes(user_id, tweet_id, created_at)
  feed(user_id, tweet_id, created_at)  -- pre-computed!

Step 4: Database Selection
  PostgreSQL: users, tweets, follows (relational, ACID)
  Redis: feed cache, online users, rate limiting
  Elasticsearch: tweet search
  Cassandra: tweet timelines for celebrities (high write)

Step 5: Feed Generation
  Fan-out on write (normal users < 1M followers):
    When tweet posted → insert into all followers' feed table
    Redis: cache latest 800 tweets per user
    
  Fan-out on read (celebrities > 1M followers):
    Feed read time এ celebrity tweets merge করো
    
  Hybrid:
    Normal users: push model (pre-computed)
    Celebrities: pull model (real-time merge)

Step 6: Scaling
  Read heavy → Redis cache + read replicas
  Write scaling → PostgreSQL primary + Cassandra for timelines
  Global → Multi-region with eventual consistency
```

**Q22: Design করো "Ride Sharing Matching System" (Uber/Pathao)**
```
Problem: Driver খুঁজে বের করো passenger এর কাছে।

Data:
  Drivers: 1M active drivers
  Passengers: 5M rides/day
  Location updates: 500K/second (drivers every 4 sec)

Approach 1 — Naive (wrong):
  SELECT * FROM drivers
  WHERE status = 'available'
  ORDER BY distance(lat, lng, driver_lat, driver_lng)
  LIMIT 5;
  -- 1M drivers এ full scan — too slow!

Approach 2 — Geohash + Redis:
  Driver location → Redis Geohash
  
  # Driver updates location:
  GEOADD available_drivers 90.4125 23.8103 "driver:101"
  
  # Find nearby drivers (within 2km):
  GEORADIUS available_drivers 90.4150 23.8120 2 km ASC COUNT 10
  
  # O(N+log M) where N=nearby drivers, M=total
  # Much faster than SQL distance calculation!

  Architecture:
    Driver app → location update → Redis GEO (real-time)
    Matching service → GEORADIUS → candidate drivers
    Assign closest available driver
    
    PostgreSQL: permanent ride records
    Redis: real-time locations (TTL 30s — offline drivers expire)
    Kafka: location event stream

Step by step matching:
  1. Passenger requests ride
  2. GEORADIUS → nearest 10 drivers
  3. Filter: available, vehicle type match, rating threshold
  4. Offer to nearest driver (30s timeout)
  5. Driver accepts → ride starts
  6. Driver rejects → offer to next driver
```

---

## 17.6 Django ORM — Interview Questions

**Q23: select_related এবং prefetch_related কীভাবে কাজ করে?**
```python
# select_related — SQL JOIN ব্যবহার করে
# ForeignKey এবং OneToOneField এর জন্য

orders = Order.objects.select_related('user', 'user__profile').all()
# Generated SQL:
# SELECT orders.*, users.*, profiles.*
# FROM orders
# INNER JOIN users ON orders.user_id = users.id
# LEFT JOIN profiles ON users.id = profiles.user_id

# prefetch_related — আলাদা query, Python এ join করে
# ManyToMany এবং reverse ForeignKey এর জন্য

orders = Order.objects.prefetch_related('items', 'items__product').all()
# Generated SQL:
# SELECT * FROM orders;
# SELECT * FROM order_items WHERE order_id IN (1,2,3,...);
# SELECT * FROM products WHERE id IN (101,202,303,...);
# Total: 3 queries — regardless of how many orders!

# Custom prefetch:
from django.db.models import Prefetch

orders = Order.objects.prefetch_related(
    Prefetch(
        'items',
        queryset=OrderItem.objects.filter(qty__gt=0).select_related('product'),
        to_attr='valid_items'
    )
).all()
# order.valid_items → filtered items list

# When to use which:
# select_related:  FK/O2O, forward relation, small related objects
# prefetch_related: M2M, reverse FK, large number of related objects
```

**Q24: F() expression কেন ব্যবহার করবে?**
```python
# ❌ Race condition:
product = Product.objects.get(id=1)
product.view_count += 1  # Python এ read করে increment
product.save()
# Problem: দুটো requests একই সময়ে → both read 100 → both save 101 → lost update!

# ✅ F() expression — database level atomic update:
from django.db.models import F

Product.objects.filter(id=1).update(view_count=F('view_count') + 1)
# SQL: UPDATE products SET view_count = view_count + 1 WHERE id = 1
# Atomic! No race condition।

# F() অন্য কাজেও:
# Column comparison:
orders = Order.objects.filter(discount__gt=F('total') * 0.5)
# WHERE discount > total * 0.5

# Annotation:
orders = Order.objects.annotate(
    profit=F('total') - F('cost')
)

# Avoiding extra queries:
Order.objects.filter(id=1).update(
    updated_at=Now(),
    status=F('next_status')  # column থেকে value নাও
)
```

---

## 17.7 Scenario-Based Questions

**Q25: Production এ হঠাৎ database slow হয়ে গেছে। কী করবে?**
```
Step 1: Immediate triage (5 minutes)
  -- Active queries দেখো:
  SELECT pid, now() - query_start AS duration, query, state
  FROM pg_stat_activity
  WHERE state != 'idle'
  ORDER BY duration DESC;
  
  -- Long running queries kill করো (if blocking):
  SELECT pg_terminate_backend(pid)
  FROM pg_stat_activity
  WHERE duration > interval '5 minutes'
    AND state = 'active';

Step 2: Lock contention check
  SELECT blocked.query, blocking.query
  FROM pg_stat_activity blocked
  JOIN pg_stat_activity blocking
      ON blocking.pid = ANY(pg_blocking_pids(blocked.pid));

Step 3: Slow queries identify করো
  SELECT query, calls, mean_exec_time, total_exec_time
  FROM pg_stat_statements
  ORDER BY mean_exec_time DESC LIMIT 10;

Step 4: Resource check
  -- Connections:
  SELECT COUNT(*) FROM pg_stat_activity;
  
  -- Cache hit rate (should be > 99%):
  SELECT
      sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) AS cache_hit_ratio
  FROM pg_statio_user_tables;

Step 5: Fix
  Missing index → CREATE INDEX CONCURRENTLY
  Slow query → EXPLAIN ANALYZE → rewrite
  Too many connections → pgBouncer, kill idle
  Lock contention → identify blocking query, optimize
  Cache miss → Redis cache add করো
```

**Q26: Migration চালানোর সময় site down হয়ে গেছে। কী করবে?**
```
Immediate:
  1. Migration rollback করো:
     python manage.py migrate myapp 0045  # previous migration
  
  2. Site উঠলো কিনা check করো
  
  3. যদি না ওঠে:
     pg_restore থেকে last known good state restore করো
     (Migration আগে backup নেওয়া উচিত ছিল!)

Root Cause Analysis:
  কোন migration টা problem করলো?
  Large table এ lock নিয়েছিল?
  Syntax error ছিল?

Prevention (future):
  Migration এর আগে backup নাও
  Staging এ আগে test করো
  Zero-downtime migration patterns follow করো
  Large table এ CREATE INDEX CONCURRENTLY ব্যবহার করো
  django-pg-zero-downtime-migrations library ব্যবহার করো
```

---

## 17.8 Quick Fire Round — Common Questions

```
Q: NULL এবং empty string এর পার্থক্য?
A: NULL = unknown/missing value। Empty string = known empty value।
   NULL = NULL is FALSE (use IS NULL)। '' = '' is TRUE।

Q: DELETE, TRUNCATE, DROP এর পার্থক্য?
A: DELETE: rows delete, WHERE possible, rollback possible, slow (logged)।
   TRUNCATE: সব rows delete, fast (minimal logging), rollback possible।
   DROP: table structure সহ সব delete, irreversible।

Q: CHAR এবং VARCHAR এর পার্থক্য?
A: CHAR(10): fixed length, always 10 chars (padded with spaces)।
   VARCHAR(10): variable length, max 10 chars (no padding)।
   Use VARCHAR almost always। CHAR শুধু fixed length codes এ (country code, status)।

Q: Composite index (a, b) কি query (WHERE b = ?) use করবে?
A: না। Leading column rule: (a, b) index — WHERE a = ? ✅, WHERE a = ? AND b = ? ✅,
   WHERE b = ? ❌ (index skip হবে, full scan)।

Q: Covering index কী?
A: Index এ সব needed columns আছে — table access দরকার নেই।
   CREATE INDEX idx_cov ON orders(user_id, status, created_at);
   SELECT status, created_at FROM orders WHERE user_id = 101;
   → Index Only Scan (table touch করে না!) — fastest possible।

Q: Materialized View কী?
A: Precomputed query result যা disk এ store হয়।
   Regular view = virtual table (query every time)।
   Materialized view = stored result (REFRESH MATERIALIZED VIEW তে update)।
   Use: expensive aggregation queries, reporting।

Q: Connection pool এর pool size কত হওয়া উচিত?
A: Formula: (CPU cores × 2) + effective_spindles
   4 core server: (4 × 2) + 1 = 9 connections per PostgreSQL instance।
   "More is not better" — too many connections = more context switching।

Q: Sharding এবং Partitioning এর পার্থক্য?
A: Partitioning: একটা DB server এ table কে ভাগ করা।
   Sharding: multiple DB servers এ data ভাগ করা (horizontal scaling)।
   Partitioning সহজ, Sharding জটিল কিন্তু unlimited scale।
```

---

## 17.9 Interview Preparation Tips

### Before the Interview

```
Technical Preparation:
  ✅ SQL queries practice করো (LeetCode Database section)
  ✅ EXPLAIN ANALYZE output read করতে পারো?
  ✅ Index types এবং কখন কোনটা?
  ✅ Transaction isolation levels মুখস্ত করো
  ✅ N+1 problem detect এবং fix করতে পারো?
  ✅ Sharding vs Partitioning clearly explain করতে পারো?

System Design Preparation:
  ✅ Practice: Design Twitter, Uber, WhatsApp database
  ✅ Scale estimation করতে পারো?
  ✅ Trade-offs clearly articulate করতে পারো?
  ✅ "Why not X?" প্রশ্নের উত্তর দিতে পারো?

Django ORM:
  ✅ select_related vs prefetch_related এর generated SQL জানো?
  ✅ F() expression কেন ব্যবহার করবে?
  ✅ Q() objects দিয়ে complex queries?
  ✅ Raw SQL কখন ব্যবহার করবে?
```

### During the Interview

```
Approach:
  1. Requirements clarify করো (functional + non-functional)
  2. Scale estimate করো (users, requests/sec, storage)
  3. Schema design করো (step by step)
  4. Database selection justify করো
  5. Trade-offs discuss করো
  6. Scaling strategy explain করো

Communication:
  "আমি X approach নেবো কারণ Y।
   Alternative Z হতো কিন্তু সেক্ষেত্রে W সমস্যা।"
  
  Trade-offs সবসময় mention করো।
  Perfect solution নেই — interviewer জানতে চায় তুমি কীভাবে ভাবো।

Red flags to avoid:
  ❌ "NoSQL সবসময় SQL থেকে faster"
  ❌ "Redis is just a cache" (Redis is a data structure server)
  ❌ "Sharding করো" (without explaining when and why)
  ❌ Index দাও সব column এ
```

---

## 17.10 LeetCode-Style SQL Problems

### Problem 1: Second Highest Salary
```sql
-- Table: employees(id, salary)
-- Find the second highest salary. Return NULL if not exists.

SELECT MAX(salary) AS SecondHighestSalary
FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);

-- Alternative with DENSE_RANK:
SELECT salary AS SecondHighestSalary
FROM (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM employees
) ranked
WHERE rnk = 2
LIMIT 1;
```

### Problem 2: Employees Earning More Than Manager
```sql
-- Table: employees(id, name, salary, manager_id)
-- Find employees earning more than their manager.

SELECT e.name AS Employee
FROM employees e
JOIN employees m ON e.manager_id = m.id
WHERE e.salary > m.salary;
```

### Problem 3: Consecutive Available Seats
```sql
-- Table: cinema(seat_id, free)
-- Find all consecutive available seats.

SELECT DISTINCT c1.seat_id
FROM cinema c1
JOIN cinema c2 ON ABS(c1.seat_id - c2.seat_id) = 1
WHERE c1.free = 1 AND c2.free = 1
ORDER BY c1.seat_id;
```

### Problem 4: Department Top 3 Salaries
```sql
-- Find top 3 salaries per department.

SELECT department, name, salary
FROM (
    SELECT
        d.name AS department,
        e.name,
        e.salary,
        DENSE_RANK() OVER (PARTITION BY d.id ORDER BY e.salary DESC) AS rnk
    FROM employees e
    JOIN departments d ON e.department_id = d.id
) ranked
WHERE rnk <= 3;
```

### Problem 5: Cumulative Sum
```sql
-- Table: orders(order_date, amount)
-- Running total per day.

SELECT
    order_date,
    amount,
    SUM(amount) OVER (ORDER BY order_date) AS running_total
FROM orders
ORDER BY order_date;
```

---

## 17.11 Module Summary & Cheat Sheet

```
INTERVIEW PREPARATION CHEAT SHEET
====================================

MUST KNOW TOPICS (every interview):
  ACID properties
  Index types (B-Tree, Hash, Composite, Covering)
  JOIN types
  GROUP BY vs HAVING
  Window Functions (ROW_NUMBER, RANK, DENSE_RANK)
  N+1 problem and solutions
  Transaction isolation levels
  Deadlock prevention
  EXPLAIN ANALYZE reading

SENIOR LEVEL TOPICS:
  MVCC internals
  Sharding vs Partitioning
  CAP theorem with real examples
  Zero-downtime migration
  Cache invalidation strategies
  Connection pooling
  Read replica routing

DJANGO ORM MUST KNOW:
  select_related vs prefetch_related (and generated SQL)
  F() expression (atomic updates)
  Q() objects (complex filters)
  annotate() vs aggregate()
  bulk_create() / bulk_update()
  select_for_update() (pessimistic lock)

SYSTEM DESIGN FRAMEWORK:
  1. Clarify requirements
  2. Estimate scale
  3. Design schema
  4. Choose databases with justification
  5. Caching strategy
  6. Scaling plan
  7. Trade-offs discussion

COMMON MISTAKES IN INTERVIEWS:
  ❌ Jump to solution without requirements
  ❌ "NoSQL is always faster"
  ❌ Forget to mention trade-offs
  ❌ Over-engineer from day 1
  ❌ Ignore consistency requirements
  ❌ Forget index on foreign keys

KEY ANSWERS TO MEMORIZE:
  N+1 fix: select_related / prefetch_related
  Race condition fix: F() expression / select_for_update
  Pagination fix: Keyset (cursor) pagination
  Deadlock fix: Consistent lock order
  Slow query fix: EXPLAIN ANALYZE → missing index
  Too many connections: pgBouncer connection pooler
```
