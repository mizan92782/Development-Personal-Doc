# MODULE 10: PostgreSQL Deep Dive
## বাংলায় সম্পূর্ণ PostgreSQL গাইড — Production Level

> **কেন PostgreSQL?**
> Instagram, Uber, Netflix, Spotify, Reddit — সবাই PostgreSQL ব্যবহার করে।
> এটা শুধু একটা database নয় — এটা একটা complete data platform।
> Senior backend engineer হতে হলে PostgreSQL এর internals জানতেই হবে।

---

## 10.1 PostgreSQL Architecture

### Overview

```
Client Application (Django/psycopg2)
          ↓
    [Connection]
          ↓
┌─────────────────────────────────────┐
│         PostgreSQL Server           │
│                                     │
│  Postmaster (main process)          │
│     ↓                               │
│  Backend Process (per connection)   │
│     ↓                               │
│  ┌──────────────────────────────┐   │
│  │    Shared Memory             │   │
│  │  ┌────────────────────────┐  │   │
│  │  │  Shared Buffer Pool    │  │   │
│  │  │  (shared_buffers)      │  │   │
│  │  └────────────────────────┘  │   │
│  │  ┌────────────────────────┐  │   │
│  │  │  WAL Buffer            │  │   │
│  │  └────────────────────────┘  │   │
│  │  ┌────────────────────────┐  │   │
│  │  │  Lock Table            │  │   │
│  │  └────────────────────────┘  │   │
│  └──────────────────────────────┘   │
│                                     │
│  Background Processes:              │
│  - WAL Writer                       │
│  - Checkpointer                     │
│  - Autovacuum                       │
│  - Background Writer                │
│  - Stats Collector                  │
└─────────────────────────────────────┘
          ↓
    [Disk Storage]
    - Data Files (base/)
    - WAL Files (pg_wal/)
    - Config Files
```

### Key Components

```
Shared Buffer Pool:
  - Disk থেকে পড়া pages cache করে রাখে
  - shared_buffers = RAM এর 25% (recommended)
  - Cache hit হলে disk I/O নেই = fast

WAL (Write-Ahead Log):
  - সব changes আগে WAL এ লেখা হয়
  - Crash হলেও WAL থেকে recover করা যায়
  - Replication এর backbone

Background Processes:
  - WAL Writer: WAL buffer → disk
  - Checkpointer: dirty pages → disk (periodic)
  - Autovacuum: dead tuples clean করে
  - Background Writer: proactively dirty pages flush
```

---

## 10.2 MVCC (Multi-Version Concurrency Control)

### কী এবং কেন?

```
Traditional Locking:
  Reader → Writer কে block করে
  Writer → Reader কে block করে
  = Low concurrency

MVCC (PostgreSQL approach):
  Reader → Writer কে block করে না
  Writer → Reader কে block করে না
  = High concurrency

"Readers don't block writers, writers don't block readers"
— PostgreSQL এর সবচেয়ে গুরুত্বপূর্ণ feature
```

### MVCC কীভাবে কাজ করে

```
প্রতিটা row এর hidden columns আছে:
  xmin → এই row কোন transaction তৈরি করেছে
  xmax → এই row কোন transaction delete/update করেছে (0 মানে active)
  ctid  → physical location (tuple id)

Example — UPDATE হলে কী হয়:

Original row:
  id=1, name='Rahim', xmin=100, xmax=0

UPDATE users SET name='Rahim Khan' WHERE id=1;
(Transaction id = 200)

Old row (dead tuple):
  id=1, name='Rahim', xmin=100, xmax=200

New row (live tuple):
  id=1, name='Rahim Khan', xmin=200, xmax=0

DELETE হলে:
  id=1, name='Rahim', xmin=100, xmax=300  ← xmax set হয়, row physically থাকে
```

### Transaction Snapshot

```sql
-- প্রতিটা transaction একটা snapshot দেখে
-- Snapshot = "এই transaction শুরু হওয়ার সময় কোন transactions active ছিল"

-- Transaction A (xid=500):
BEGIN;
SELECT * FROM users WHERE id = 1;
-- Snapshot: xid=500, active=[450, 480]
-- শুধু committed transactions (xid < 500 এবং active নয়) দেখতে পাবে

-- Transaction B (xid=501):
BEGIN;
UPDATE users SET name='New Name' WHERE id = 1;  -- xmin=501
COMMIT;

-- Transaction A আবার query করলে:
SELECT * FROM users WHERE id = 1;
-- REPEATABLE READ: পুরনো snapshot — 'Rahim' দেখবে
-- READ COMMITTED: নতুন snapshot — 'New Name' দেখবে
```

### Dead Tuples সমস্যা

```
MVCC এর side effect:
- UPDATE/DELETE করলে old rows physically থেকে যায় (dead tuples)
- এগুলো disk space নেয়
- Sequential scan এ এগুলো skip করতে হয় → slow
- Index এও dead entries থাকে → index bloat

Solution: VACUUM
```

---

## 10.3 VACUUM এবং AUTOVACUUM

### VACUUM কী?

```
VACUUM এর কাজ:
1. Dead tuples reuse এর জন্য mark করা (space free করা)
2. Transaction ID wraparound prevent করা
3. Table statistics update (ANALYZE সহ)

VACUUM FULL:
- Dead space physically reclaim করে (table rewrite)
- Table এ exclusive lock নেয় — production এ সাবধান!
- Table size physically কমে

VACUUM (without FULL):
- Dead tuples reuse এর জন্য mark করে
- Lock নেয় না (concurrent readers/writers চলতে পারে)
- Table size কমে না কিন্তু space reuse হয়
```

### VACUUM Commands

```sql
-- Basic VACUUM
VACUUM users;

-- VACUUM + statistics update
VACUUM ANALYZE users;

-- VACUUM FULL — table rewrite (production এ সাবধান!)
VACUUM FULL users;

-- VERBOSE output দেখো
VACUUM VERBOSE ANALYZE orders;
-- Output:
-- INFO: vacuuming "public.orders"
-- INFO: scanned index "idx_orders_customer" to remove 15234 row versions
-- INFO: removed 15234 row versions in 892 pages
-- INFO: found 15234 removable, 284521 nonremovable row versions

-- Autovacuum status দেখো
SELECT
    schemaname,
    tablename,
    n_dead_tup,          -- dead tuples count
    n_live_tup,          -- live tuples count
    last_vacuum,
    last_autovacuum,
    last_analyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

### Autovacuum Configuration

```sql
-- postgresql.conf settings
autovacuum = on                          -- enable (default on)
autovacuum_vacuum_threshold = 50         -- minimum dead tuples
autovacuum_vacuum_scale_factor = 0.2     -- 20% of table
-- Trigger: dead_tuples > threshold + scale_factor * reltuples

-- High-write table এর জন্য aggressive autovacuum
-- postgresql.conf বা per-table:
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.05,    -- 5% (more frequent)
    autovacuum_vacuum_threshold = 100,
    autovacuum_analyze_scale_factor = 0.02
);

-- Autovacuum কতটা কাজ করছে দেখো
SELECT
    pid,
    datname,
    query,
    now() - pg_stat_activity.query_start AS duration
FROM pg_stat_activity
WHERE query LIKE 'autovacuum%';
```

### Bloat Detection

```sql
-- Table bloat check
SELECT
    tablename,
    pg_size_pretty(pg_total_relation_size(tablename::regclass)) AS total_size,
    pg_size_pretty(pg_relation_size(tablename::regclass)) AS table_size,
    n_dead_tup,
    ROUND(n_dead_tup::numeric / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 2)
        AS dead_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY dead_pct DESC;
```

---

## 10.4 ANALYZE

### কী এবং কেন?

```
ANALYZE:
- Table এর statistics collect করে
- Query planner এই statistics ব্যবহার করে execution plan বানাতে
- Statistics outdated হলে → wrong plan → slow query

Statistics includes:
- Row count estimates
- Column value distribution (histogram)
- Most common values
- Null fraction
```

```sql
-- ANALYZE চালাও
ANALYZE orders;
ANALYZE;  -- সব tables

-- Statistics দেখো
SELECT
    attname,
    n_distinct,      -- unique values estimate (-1 মানে 100% unique)
    correlation,     -- physical vs logical order
    most_common_vals,
    most_common_freqs
FROM pg_stats
WHERE tablename = 'orders'
  AND attname = 'status';

-- Output:
-- attname: status
-- n_distinct: 4          (pending/confirmed/shipped/delivered)
-- most_common_vals: {delivered, pending, shipped, confirmed}
-- most_common_freqs: {0.65, 0.20, 0.10, 0.05}
-- Planner জানে: 65% rows 'delivered', 20% 'pending'
```

---

## 10.5 WAL (Write-Ahead Log)

### কীভাবে কাজ করে

```
WAL এর Purpose:
1. Durability: Crash হলেও committed data হারাবে না
2. Replication: WAL stream replica তে পাঠানো যায়
3. PITR: Point-in-time recovery

WAL Write Flow:

Transaction COMMIT হলে:
  1. Changes WAL buffer এ লেখো
  2. WAL buffer → WAL file (pg_wal/) flush করো (fsync)
  3. Client কে "committed" confirm করো
  4. পরে (async): dirty pages → data files

এই sequential WAL write অনেক fast (sequential I/O)
Data files এ random write পরে হয়
```

### WAL Configuration

```sql
-- postgresql.conf
wal_level = replica          -- minimal, replica, logical
max_wal_size = 1GB           -- WAL maximum size
min_wal_size = 80MB
checkpoint_completion_target = 0.9  -- gradual checkpoint
synchronous_commit = on      -- wait for WAL write (durability)
-- synchronous_commit = off  -- faster but slight data loss risk

-- WAL statistics
SELECT * FROM pg_stat_bgwriter;

-- WAL location
SELECT pg_current_wal_lsn();        -- current WAL position
SELECT pg_walfile_name(pg_current_wal_lsn());  -- current WAL file
```

### WAL এবং Replication

```
Primary Server:
  WAL → pg_wal/ directory
     ↓ (WAL streaming)
Replica Server:
  WAL receive → replay → replica data updated

Types:
  Streaming Replication: real-time WAL stream
  Logical Replication:   table-level, selective replication
```

---

## 10.6 Replication (রেপ্লিকেশন)

### Streaming Replication Setup

```
Architecture:

[Primary DB] ──── WAL stream ────→ [Replica 1] (Read)
                                 ↘ [Replica 2] (Read)
                                 ↘ [Replica 3] (Standby/Failover)
```

```sql
-- Primary server postgresql.conf:
wal_level = replica
max_wal_senders = 5
wal_keep_size = 1GB

-- Primary server pg_hba.conf:
host replication replicator 192.168.1.0/24 md5

-- Replica server recovery.conf (PostgreSQL 11 এর আগে):
standby_mode = on
primary_conninfo = 'host=primary_host port=5432 user=replicator'
trigger_file = '/tmp/postgresql.trigger'

-- Replica server postgresql.conf (PostgreSQL 12+):
primary_conninfo = 'host=primary_host port=5432 user=replicator'

-- Replication lag monitor করো
SELECT
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replication_lag_bytes
FROM pg_stat_replication;
```

### Django এ Read Replica ব্যবহার

```python
# settings.py
DATABASES = {
    'default': {  # Primary — writes
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        'HOST': 'primary.db.example.com',
    },
    'replica': {  # Replica — reads
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        'HOST': 'replica.db.example.com',
        'TEST': {'MIRROR': 'default'},
    }
}

# Database Router
class PrimaryReplicaRouter:
    def db_for_read(self, model, **hints):
        return 'replica'   # সব reads replica তে

    def db_for_write(self, model, **hints):
        return 'default'   # সব writes primary তে

    def allow_relation(self, obj1, obj2, **hints):
        return True

    def allow_migrate(self, db, app_label, model_name=None, **hints):
        return db == 'default'

# settings.py
DATABASE_ROUTERS = ['myapp.routers.PrimaryReplicaRouter']

# Specific query তে database choose করো
orders = Order.objects.using('replica').filter(status='pending')
Order.objects.using('default').create(...)  # write always primary
```

---

## 10.7 Partitioning (পার্টিশনিং)

### কেন দরকার?

```
Problem:
  orders table এ 1 billion rows
  Query: last month এর orders
  → পুরো 1B rows এর index scan!

Solution: Partitioning
  orders_2024_01 → January 2024 data
  orders_2024_02 → February 2024 data
  ...
  orders_2024_12 → December 2024 data

Query: last month
  → শুধু orders_2024_12 scan করে!
  = "Partition Pruning"
```

### Range Partitioning

```sql
-- Parent table (partitioned)
CREATE TABLE orders (
    id          BIGSERIAL,
    customer_id BIGINT,
    total_amount DECIMAL(10,2),
    created_at  TIMESTAMPTZ NOT NULL,
    status      VARCHAR(20)
) PARTITION BY RANGE (created_at);

-- Monthly partitions
CREATE TABLE orders_2024_01 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE orders_2024_02 PARTITION OF orders
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

CREATE TABLE orders_2024_03 PARTITION OF orders
    FOR VALUES FROM ('2024-03-01') TO ('2024-04-01');

-- Default partition (এর বাইরের values)
CREATE TABLE orders_default PARTITION OF orders DEFAULT;

-- Index প্রতিটা partition এ তৈরি হয়
CREATE INDEX ON orders_2024_01 (customer_id);
CREATE INDEX ON orders_2024_02 (customer_id);

-- Query — partition pruning automatic
EXPLAIN SELECT * FROM orders
WHERE created_at >= '2024-03-01' AND created_at < '2024-04-01';
-- শুধু orders_2024_03 scan করবে!
```

### Auto Partition Creation (pg_partman)

```sql
-- pg_partman extension দিয়ে automatic partition management
CREATE EXTENSION pg_partman;

-- Monthly auto-partitioning setup
SELECT partman.create_parent(
    p_parent_table => 'public.orders',
    p_control => 'created_at',
    p_type => 'range',
    p_interval => 'monthly',
    p_premake => 3  -- 3 মাস আগে partition তৈরি করো
);

-- Maintenance (cron job এ চালাও)
CALL partman.run_maintenance_proc();
```

### Hash Partitioning

```sql
-- User data শার্ড করতে
CREATE TABLE users (
    id   BIGSERIAL,
    name VARCHAR(100),
    email VARCHAR(254)
) PARTITION BY HASH (id);

-- 4টা partition
CREATE TABLE users_p0 PARTITION OF users
    FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE users_p1 PARTITION OF users
    FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE users_p2 PARTITION OF users
    FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE users_p3 PARTITION OF users
    FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

### Django ORM এ Partitioned Tables

```python
# Django directly partitioned tables manage করে না
# কিন্তু query করতে পারে normally

# Raw SQL দিয়ে partition তৈরি করো migration এ
from django.db import migrations

class Migration(migrations.Migration):
    operations = [
        migrations.RunSQL(
            sql="""
            CREATE TABLE orders_2024_01 PARTITION OF orders
            FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
            """,
            reverse_sql="DROP TABLE orders_2024_01;"
        ),
    ]

# Query — ORM automatically correct partition query করে
orders = Order.objects.filter(
    created_at__gte='2024-03-01',
    created_at__lt='2024-04-01'
)
# Partition pruning automatic!
```

---

## 10.8 JSONB (JSON Binary)

### কী এবং কেন?

```
PostgreSQL এ দুটো JSON type:
  json  → text হিসেবে store, query তে parse
  jsonb → binary format, indexed, faster query

JSONB কখন ব্যবহার করবে:
  ✅ Schema flexible হতে হবে (product attributes)
  ✅ Semi-structured data (event logs, metadata)
  ✅ Rapid prototyping
  ✅ Third-party data store

JSONB কখন এড়াবে:
  ❌ Structured relational data (normalization better)
  ❌ Heavy JOIN with JSONB fields
  ❌ Frequent full-field updates
```

### JSONB Operations

```sql
-- JSONB column সহ table
CREATE TABLE products (
    id         BIGSERIAL PRIMARY KEY,
    name       VARCHAR(200),
    attributes JSONB,          -- flexible attributes
    metadata   JSONB
);

-- Insert
INSERT INTO products (name, attributes) VALUES (
    'Samsung TV',
    '{"screen_size": 55, "resolution": "4K", "features": ["HDR", "Smart TV"], "brand": "Samsung"}'
);

-- Query JSONB
SELECT * FROM products
WHERE attributes->>'brand' = 'Samsung';
-- -> returns JSONB, ->> returns text

-- Nested access
SELECT attributes->'specs'->>'processor' FROM products WHERE id = 1;

-- Array element
SELECT attributes->'features'->0 FROM products;  -- first element

-- Contains operator @>
SELECT * FROM products
WHERE attributes @> '{"brand": "Samsung", "screen_size": 55}';

-- Key exists ?
SELECT * FROM products WHERE attributes ? 'discount_price';

-- jsonb_each — row এ expand করো
SELECT key, value
FROM products, jsonb_each(attributes)
WHERE id = 1;

-- Update JSONB field
UPDATE products
SET attributes = jsonb_set(
    attributes,
    '{price}',
    '1200'::jsonb
)
WHERE id = 1;

-- Remove key
UPDATE products
SET attributes = attributes - 'old_key'
WHERE id = 1;
```

### JSONB Index

```sql
-- GIN index — most common for JSONB
CREATE INDEX idx_products_attributes ON products USING GIN(attributes);

-- @> operator এ fast হবে
SELECT * FROM products WHERE attributes @> '{"brand": "Samsung"}';
-- Index ব্যবহার হবে!

-- Specific path এ index
CREATE INDEX idx_products_brand ON products
    ((attributes->>'brand'));
-- WHERE attributes->>'brand' = 'Samsung' fast হবে
```

### Django ORM এ JSONB

```python
from django.contrib.postgres.fields import JSONField  # Django 3.1+
# অথবা models.JSONField  (Django 3.1+)

class Product(models.Model):
    name       = models.CharField(max_length=200)
    attributes = models.JSONField(default=dict)

# Query
# Contains
products = Product.objects.filter(
    attributes__contains={'brand': 'Samsung'}
)
# SQL: WHERE attributes @> '{"brand": "Samsung"}'

# Key-value access
products = Product.objects.filter(
    attributes__brand='Samsung'
)
# SQL: WHERE attributes->>'brand' = 'Samsung'

# Nested
products = Product.objects.filter(
    attributes__specs__processor='Snapdragon'
)

# has_key
products = Product.objects.filter(
    attributes__has_key='discount_price'
)

# Array contains
products = Product.objects.filter(
    attributes__features__contains=['HDR']
)

# Update specific key
from django.db.models import Func, Value

Product.objects.filter(id=1).update(
    attributes=Func(
        'attributes',
        Value(['price']),
        Value(1200),
        function='jsonb_set'
    )
)
```

---

## 10.9 Materialized Views

### কী এবং কেন?

```
Regular View:
  - Query এর alias — প্রতিবার base table query করে
  - আসল data store হয় না

Materialized View:
  - Query result physically store করে
  - Super fast reads (pre-computed)
  - Manual বা scheduled refresh দরকার
  - Stale data হতে পারে

কখন ব্যবহার করবে:
  ✅ Expensive aggregation query (dashboard, reports)
  ✅ Read-heavy, stale data acceptable
  ✅ Complex JOIN results cache করতে
  ✅ Analytics queries
```

### Materialized View তৈরি

```sql
-- Monthly sales report materialized view
CREATE MATERIALIZED VIEW monthly_sales_summary AS
SELECT
    DATE_TRUNC('month', o.created_at) AS month,
    c.name AS category_name,
    COUNT(DISTINCT o.id) AS order_count,
    SUM(oi.quantity) AS total_qty,
    SUM(oi.quantity * oi.unit_price) AS revenue
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON p.id = oi.product_id
JOIN categories c ON c.id = p.category_id
WHERE o.status = 'delivered'
GROUP BY DATE_TRUNC('month', o.created_at), c.name
ORDER BY month DESC, revenue DESC;

-- Index on materialized view
CREATE INDEX ON monthly_sales_summary(month);
CREATE INDEX ON monthly_sales_summary(category_name);

-- Query — instant! (pre-computed)
SELECT * FROM monthly_sales_summary
WHERE month = DATE_TRUNC('month', NOW());
-- milliseconds, কোনো heavy JOIN নেই!

-- Refresh (data update করো)
REFRESH MATERIALIZED VIEW monthly_sales_summary;
-- এটা lock নেয়!

-- CONCURRENTLY — no lock (unique index দরকার)
CREATE UNIQUE INDEX ON monthly_sales_summary(month, category_name);
REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_sales_summary;
-- Production এ সবসময় CONCURRENTLY ব্যবহার করো
```

### Django + Materialized View

```python
# Django তে materialized view unmanaged model হিসেবে define করো

class MonthlySalesSummary(models.Model):
    month         = models.DateTimeField()
    category_name = models.CharField(max_length=100)
    order_count   = models.IntegerField()
    total_qty     = models.IntegerField()
    revenue       = models.DecimalField(max_digits=12, decimal_places=2)

    class Meta:
        managed = False  # Django এই table manage করবে না
        db_table = 'monthly_sales_summary'

# Query করো normally
report = MonthlySalesSummary.objects.filter(
    month=timezone.now().replace(day=1)
).order_by('-revenue')

# Refresh করো (Celery task এ)
from django.db import connection

@shared_task
def refresh_sales_summary():
    with connection.cursor() as cursor:
        cursor.execute(
            "REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_sales_summary"
        )

# Schedule: প্রতি ঘন্টায় বা প্রতিদিন রাতে
```

---

## 10.10 PostgreSQL Performance Tuning

### Key Configuration Parameters

```sql
-- postgresql.conf — important settings

-- Memory
shared_buffers = 4GB              -- RAM এর 25%
effective_cache_size = 12GB       -- RAM এর 75% (OS cache estimate)
work_mem = 64MB                   -- Sort/Hash এর জন্য per-query
maintenance_work_mem = 512MB      -- VACUUM/CREATE INDEX এর জন্য

-- Write Performance
wal_buffers = 64MB
checkpoint_completion_target = 0.9
max_wal_size = 4GB

-- Query Planner
random_page_cost = 1.1           -- SSD এর জন্য (default 4.0 is for HDD)
effective_io_concurrency = 200   -- SSD IOPS capability

-- Connections
max_connections = 200
-- Note: pgBouncer ব্যবহার করলে এটা কম রাখো (50-100)

-- Logging
log_min_duration_statement = 1000   -- 1s এর বেশি queries log করো
log_checkpoints = on
log_lock_waits = on
```

### pg_stat_statements — Top Slow Queries

```sql
-- extension enable করো
CREATE EXTENSION pg_stat_statements;

-- postgresql.conf:
-- shared_preload_libraries = 'pg_stat_statements'
-- pg_stat_statements.max = 10000
-- pg_stat_statements.track = all

-- Top 10 slow queries
SELECT
    LEFT(query, 100) AS query_snippet,
    calls,
    ROUND(total_exec_time::numeric / calls, 2) AS avg_ms,
    ROUND(total_exec_time::numeric, 2) AS total_ms,
    rows
FROM pg_stat_statements
ORDER BY avg_ms DESC
LIMIT 10;

-- Reset statistics
SELECT pg_stat_statements_reset();
```

### Connection Pooling — pgBouncer

```
সমস্যা:
  প্রতিটা PostgreSQL connection = ~10MB RAM + overhead
  1000 connections = 10GB RAM শুধু connections এ!
  Django এর প্রতিটা request এ নতুন connection = slow

Solution: pgBouncer
  Application → pgBouncer → PostgreSQL
  pgBouncer connection pool maintain করে
  Actual PostgreSQL connections অনেক কম
```

```ini
# pgbouncer.ini
[databases]
mydb = host=localhost port=5432 dbname=mydb

[pgbouncer]
pool_mode = transaction        # transaction level pooling (best performance)
max_client_conn = 1000         # app থেকে max connections
default_pool_size = 20         # PostgreSQL এ actual connections
min_pool_size = 5
reserve_pool_size = 5
server_idle_timeout = 600
```

```python
# Django settings — pgBouncer port ব্যবহার করো
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'HOST': 'localhost',
        'PORT': '6432',           # pgBouncer port (PostgreSQL: 5432)
        'NAME': 'mydb',
        'OPTIONS': {
            'pool_timeout': 10,
        },
        # pgBouncer transaction mode এ prepared statements disable করো
        'DISABLE_SERVER_SIDE_CURSORS': True,
        'OPTIONS': {'options': '-c default_transaction_isolation=read\\ committed'},
    }
}
```

---

## 10.11 Useful PostgreSQL Queries (Production Monitoring)

```sql
-- Database size
SELECT
    datname,
    pg_size_pretty(pg_database_size(datname)) AS size
FROM pg_database
ORDER BY pg_database_size(datname) DESC;

-- Table sizes
SELECT
    tablename,
    pg_size_pretty(pg_total_relation_size(tablename::regclass)) AS total,
    pg_size_pretty(pg_relation_size(tablename::regclass)) AS table,
    pg_size_pretty(
        pg_total_relation_size(tablename::regclass) -
        pg_relation_size(tablename::regclass)
    ) AS indexes
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(tablename::regclass) DESC;

-- Active queries
SELECT
    pid,
    now() - query_start AS duration,
    state,
    LEFT(query, 100) AS query
FROM pg_stat_activity
WHERE state != 'idle'
  AND query_start IS NOT NULL
ORDER BY duration DESC;

-- Kill a stuck query
SELECT pg_cancel_backend(pid);    -- graceful
SELECT pg_terminate_backend(pid); -- force kill

-- Cache hit ratio (should be > 99%)
SELECT
    sum(heap_blks_hit) / NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0)
        AS cache_hit_ratio
FROM pg_statio_user_tables;

-- Index usage
SELECT
    relname,
    seq_scan,        -- full table scans
    idx_scan,        -- index scans
    CASE WHEN seq_scan + idx_scan > 0
         THEN ROUND(idx_scan::numeric / (seq_scan + idx_scan) * 100, 2)
         ELSE 0
    END AS idx_usage_pct
FROM pg_stat_user_tables
ORDER BY seq_scan DESC;
```

---

## 10.12 কেন PostgreSQL Production এ এত জনপ্রিয়?

```
Feature                    PostgreSQL          MySQL
─────────────────────────────────────────────────────
ACID compliance            ✅ Full              ✅ InnoDB only
MVCC                       ✅ Excellent         ✅ Good
JSON support               ✅ JSONB (binary)    ✅ JSON (text)
Full-text search           ✅ Built-in          ⚠️  Limited
Partitioning               ✅ Declarative       ✅ Available
Window functions           ✅ Full support      ✅ MySQL 8+
CTEs                       ✅ Full (writable)   ✅ MySQL 8+
Custom types               ✅ Yes               ❌ No
Extensions                 ✅ Rich ecosystem    ❌ Limited
PostGIS (geospatial)       ✅ Best-in-class     ❌ No
Logical replication        ✅ Flexible          ✅ Available
Parallel queries           ✅ Yes               ⚠️  Limited
Licensing                  ✅ Open source       ✅ GPL/Commercial
```

---

## 10.13 Common Mistakes

### ভুল ১: shared_buffers ছোট রাখা

```sql
-- ❌ Default (128MB) — production এ অপর্যাপ্ত
shared_buffers = 128MB

-- ✅ RAM এর 25%
shared_buffers = 4GB  -- 16GB RAM সার্ভারে
```

### ভুল ২: Autovacuum Disable করা

```sql
-- ❌ কখনো করো না!
autovacuum = off
-- Table bloat হবে, queries slow হবে, transaction ID wraparound হবে!

-- ✅ Tune করো, disable করো না
autovacuum_vacuum_scale_factor = 0.05  -- more frequent
```

### ভুল ৩: random_page_cost ঠিক না করা

```sql
-- ❌ Default (4.0) — HDD এর জন্য
-- SSD এ এই value দিলে planner unnecessarily seq scan choose করে

-- ✅ SSD এর জন্য
random_page_cost = 1.1
effective_io_concurrency = 200
```

### ভুল ৪: Connection Pool ছাড়া Production Deploy

```
❌ Django app সরাসরি PostgreSQL এ
   100 concurrent requests = 100 connections = memory problem

✅ pgBouncer → PostgreSQL
   100 concurrent requests → pgBouncer (5-20 actual connections)
```

---

## 10.14 Production Best Practices

```
1.  shared_buffers = RAM এর 25% set করো
2.  random_page_cost = 1.1 (SSD এ)
3.  pgBouncer দিয়ে connection pool করো
4.  Autovacuum tune করো, disable করো না
5.  pg_stat_statements enable করো — slow query monitor করো
6.  log_min_duration_statement = 500ms — slow queries log করো
7.  Partitioning ব্যবহার করো large time-series tables এ
8.  Read replicas set up করো — reads distribute করো
9.  JSONB তে GIN index দাও
10. Materialized view refresh কে CONCURRENTLY করো
11. Regular VACUUM ANALYZE চালাও
12. WAL archiving enable করো PITR এর জন্য
13. Bloat monitor করো pg_stat_user_tables দিয়ে
14. Cache hit ratio > 99% maintain করো
```

---

## 10.15 Senior Engineer Insights

> **War Story — Instagram:**
> Instagram PostgreSQL তে 1.2 billion users handle করে।
> তারা horizontal sharding করেছে — user_id hash দিয়ে thousands of PostgreSQL instances এ।
> কিন্তু শুরুতে single PostgreSQL + read replicas দিয়ে কোটি users handle করেছে।
> Lesson: PostgreSQL অনেক দূর নিয়ে যেতে পারে — premature sharding এড়াও।

> **Uber এর PostgreSQL → MySQL Migration:**
> Uber 2016 তে PostgreSQL থেকে MySQL এ migrate করে।
> কারণ: তাদের specific write pattern এ PostgreSQL MVCC overhead বেশি ছিল।
> কিন্তু এটা edge case — বেশিরভাগ system এ PostgreSQL best choice।

> **Design Decision:**
> JSONB ব্যবহার করো flexible attributes এর জন্য (product variants, user preferences)।
> কিন্তু JSONB কে primary data model বানিও না — relational structure better।
> "JSONB is a tool, not a solution."

---

## 10.16 Interview Questions

### Beginner Level

**Q1: PostgreSQL এবং MySQL এর মূল পার্থক্য কী?**
> PostgreSQL: ACID fully compliant, MVCC excellent, JSONB built-in, custom types, PostGIS, richer SQL standard support। MySQL: simpler, faster for simple queries historically, wider hosting support। Production গ্রেড system এ PostgreSQL preferred।

**Q2: VACUUM কেন দরকার?**
> MVCC এর কারণে DELETE/UPDATE করলে old rows (dead tuples) physically থাকে। VACUUM এগুলো reuse এর জন্য mark করে। না করলে table bloat হয়, queries slow হয়, disk space waste হয়।

**Q3: Materialized view কখন ব্যবহার করবে?**
> Expensive aggregation query যেটা বারবার চলে কিন্তু real-time data দরকার নেই। Dashboard, reports, analytics। Refresh schedule করো (hourly/daily)।

---

### Intermediate Level

**Q4: MVCC কীভাবে কাজ করে?**
> প্রতিটা row এ xmin (created by) এবং xmax (deleted by) hidden column আছে। UPDATE করলে new version তৈরি হয়, old version dead tuple হয়। প্রতিটা transaction snapshot দেখে — শুধু নিজের start এর আগের committed transactions এর rows দেখে। এতে readers writers কে block করে না।

**Q5: Connection pooling কেন দরকার?**
> PostgreSQL এ প্রতিটা connection এর overhead আছে (~10MB RAM, process fork)। High-traffic app এ hundreds of concurrent connections database এ strain দেয়। pgBouncer pool maintain করে — app থেকে 1000 connections আসলেও database এ মাত্র 20-50 actual connections।

**Q6: Partitioning কখন করবে?**
> Table খুব large হলে (100M+ rows), time-based queries common হলে, old data archive করতে হলে। Range partitioning by date সবচেয়ে common। Partition pruning query কে শুধু relevant partition এ limit করে।

---

### Senior / FAANG Level

**Q7: 1 billion rows এর orders table কে কীভাবে design করবে?**
```
Strategy:
1. Range Partitioning by created_at (monthly)
   → প্রতিটা partition ~8M rows (reasonable)
   → Partition pruning date queries কে fast করে

2. Per-partition indexes
   → (customer_id, created_at) composite index
   → Partial index on status='pending'

3. Read Replicas
   → Analytics queries replica তে route করো

4. Archive old partitions
   → 2+ year old data cold storage (S3 + foreign table)

5. Materialized Views
   → Monthly summary pre-compute করো

Schema:
CREATE TABLE orders (
    id          BIGSERIAL,
    customer_id BIGINT NOT NULL,
    status      VARCHAR(20) NOT NULL,
    total_amount DECIMAL(12,2),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- Indexes per partition
CREATE INDEX ON orders_2024_01 (customer_id, created_at DESC);
CREATE INDEX ON orders_2024_01 (status) WHERE status = 'pending';
```

**Q8: PostgreSQL এ high write throughput কীভাবে achieve করবে?**
```
1. WAL Tuning:
   wal_buffers = 64MB
   synchronous_commit = off  (slight durability trade-off)
   commit_delay = 1000       (group commits)

2. bulk_create পরিবর্তে COPY command:
   COPY orders FROM '/tmp/orders.csv' CSV;
   (INSERT এর চেয়ে 10x faster)

3. Unlogged tables (temporary data):
   CREATE UNLOGGED TABLE sessions (...)
   (WAL bypass — faster but no crash recovery)

4. Partitioning reduces lock contention

5. Connection pooling (pgBouncer)

6. SSD storage + tune random_page_cost
```

---

## 10.17 Hands-on Exercises

### Exercise 1
```sql
-- orders table এ MVCC দেখাও:
-- 1. একটা row এর xmin, xmax দেখাও
-- 2. UPDATE করো এবং আবার দেখাও
-- 3. Dead tuple count দেখাও
-- 4. VACUUM চালাও এবং আবার দেখাও
```

### Exercise 2
```sql
-- Materialized view তৈরি করো:
-- Product category wise monthly sales summary
-- CONCURRENTLY refresh করো
-- EXPLAIN দিয়ে দেখাও regular query vs materialized view
```

### Exercise 3
```python
# Django তে:
# 1. Read replica configure করো
# 2. Analytics queries replica তে route করো
# 3. Writes primary তে যাচ্ছে confirm করো
# 4. Replication lag monitor করো
```

### Mini Project
```
Production-ready PostgreSQL setup:
1. Primary + 1 Replica
2. pgBouncer connection pooling
3. Partitioned orders table (monthly)
4. Materialized view for dashboard
5. JSONB product attributes
6. Monitoring queries (slow query, cache hit, bloat)
7. Django database router
```

---

## 10.18 Module Summary & Cheat Sheet

```
POSTGRESQL CHEAT SHEET
=======================

MVCC:
  xmin → row created by transaction
  xmax → row deleted/updated by transaction (0 = live)
  Dead tuple → old version after UPDATE/DELETE
  VACUUM → dead tuples clean করে

VACUUM:
  VACUUM table             → Mark dead tuples (no lock)
  VACUUM ANALYZE table     → + update statistics
  VACUUM FULL table        → Rewrite (lock! avoid in prod)
  VACUUM VERBOSE ANALYZE   → Detailed output
  autovacuum               → Always ON, tune don't disable

WAL:
  wal_level = replica      → Replication enable
  synchronous_commit = on  → Durability (default)
  max_wal_size = 1GB       → WAL size

REPLICATION:
  Streaming → WAL stream real-time
  Logical   → Table-level, selective
  Replica lag: pg_stat_replication

PARTITIONING:
  PARTITION BY RANGE (date)  → Time-series
  PARTITION BY HASH (id)     → Even distribution
  PARTITION BY LIST (status) → Known values
  Partition pruning = automatic query optimization

JSONB:
  ->>  → Text value
  ->   → JSONB value
  @>   → Contains
  ?    → Key exists
  GIN index → Fast @> queries

MATERIALIZED VIEW:
  CREATE MATERIALIZED VIEW name AS SELECT ...
  REFRESH MATERIALIZED VIEW name;
  REFRESH MATERIALIZED VIEW CONCURRENTLY name; → No lock

PERFORMANCE SETTINGS (SSD):
  shared_buffers = RAM * 0.25
  effective_cache_size = RAM * 0.75
  random_page_cost = 1.1
  effective_io_concurrency = 200
  work_mem = 64MB

MONITORING:
  pg_stat_activity          → Active queries
  pg_stat_user_tables       → Table stats, dead tuples
  pg_stat_user_indexes      → Index usage
  pg_stat_statements        → Query performance
  pg_stat_replication       → Replica lag
  pg_statio_user_tables     → Cache hit ratio

USEFUL COMMANDS:
  EXPLAIN (ANALYZE, BUFFERS) query;
  VACUUM ANALYZE table;
  ANALYZE table;
  pg_cancel_backend(pid)    → Cancel query
  pg_terminate_backend(pid) → Kill connection
  pg_size_pretty(size)      → Human readable size
```
