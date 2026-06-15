# MODULE 16: System Design and Database Decisions
## বাংলায় সম্পূর্ণ System Design গাইড — Senior Engineer Level

> **কেন এই module সবচেয়ে গুরুত্বপূর্ণ?**
> Junior engineer code লেখে। Senior engineer সিদ্ধান্ত নেয়।
> "কোন database ব্যবহার করবো?" — এই প্রশ্নের উত্তর দিতে পারাটাই senior engineer এর কাজ।
> Google, Amazon, Netflix এর system design interview এ এটাই জিজ্ঞেস করা হয়।
> ভুল database choice = বছরের পর বছর technical debt।

---

## 16.1 কীভাবে Senior Engineer Database Choose করে?

### Framework — Database Selection

```
প্রশ্ন করো এই ক্রমে:

1. Data কেমন?
   - Structured (rows, columns)?     → SQL
   - Semi-structured (JSON, nested)? → PostgreSQL JSONB / MongoDB
   - Key-Value pairs?                → Redis / DynamoDB
   - Graph relationships?            → Neo4j
   - Time series?                    → TimescaleDB / InfluxDB
   - Full-text search?               → Elasticsearch

2. Consistency requirement কতটুকু?
   - Strong consistency (money, inventory) → PostgreSQL, MySQL
   - Eventual consistency OK?             → Cassandra, DynamoDB

3. Scale কত?
   - < 10M rows, < 1000 req/sec → Single PostgreSQL
   - > 100M rows, > 10K req/sec → Sharding / NoSQL
   - Write-heavy?               → Cassandra
   - Read-heavy?                → Read replicas + Redis

4. Query pattern কেমন?
   - Complex joins, aggregations → SQL
   - Simple key lookup           → Redis / DynamoDB
   - Range queries               → SQL / Cassandra
   - Graph traversal             → Neo4j

5. Team expertise কী আছে?
   - নতুন technology শেখার cost কত?
   - Operations cost (managed vs self-hosted)?
```

### Decision Matrix

```
USE CASE                  RECOMMENDED DB         WHY
─────────────────────────────────────────────────────────────
User accounts, orders     PostgreSQL             ACID, joins, complex queries
Session storage           Redis                  Fast TTL-based key-value
Real-time leaderboard     Redis (ZSET)           O(log N) sorted set
Product catalog           PostgreSQL + Redis      SQL + cache
Activity feed             Cassandra / Redis       High write, append-only
Search                    Elasticsearch           Full-text, facets
IoT sensor data           TimescaleDB             Time series optimization
Social graph              Neo4j / PostgreSQL      Graph traversal
File metadata             PostgreSQL              Structured + JSONB
Analytics / OLAP          BigQuery / Redshift     Columnar, large scans
Chat messages             Cassandra               High write, time-ordered
Config / feature flags    Redis / DynamoDB        Fast read, small data
```

---

## 16.2 Trade-offs — Senior Engineer যেভাবে ভাবে

### CAP Theorem Real-World Decisions

```
CAP Theorem:
  C = Consistency  (সব nodes এ same data)
  A = Availability (সবসময় respond করে)
  P = Partition Tolerance (network split হলেও চলে)

  Network partition real world এ হয়।
  তাই choice: CP অথবা AP।

CP (Consistency + Partition Tolerance):
  PostgreSQL, MySQL, MongoDB (default)
  Network split এ availability sacrifice করে।
  Banking, payments — সঠিক।

AP (Availability + Partition Tolerance):
  Cassandra, DynamoDB, CouchDB
  Network split এ stale data দিতে পারে।
  Social media feed, product catalog — acceptable।

Example:
  bKash money transfer → CP (PostgreSQL)
    "Wrong balance দেখানো acceptable না"

  Facebook news feed → AP (Cassandra)
    "5 seconds পুরনো feed দেখানো acceptable"
```

### SQL vs NoSQL — কখন কোনটা?

```
SQL চাও যখন:
  ✅ Complex queries (joins, aggregations, CTEs)
  ✅ Strong ACID guarantees
  ✅ Relationships between entities
  ✅ Schema validation important
  ✅ Reporting / analytics
  ✅ Team already knows SQL

NoSQL চাও যখন:
  ✅ Massive write throughput (> 100K writes/sec)
  ✅ Horizontal sharding necessary
  ✅ Schema-less / dynamic attributes
  ✅ Simple access patterns (key lookup)
  ✅ Eventual consistency acceptable
  ✅ Specific data types (graph, time series, document)

❌ Common Mistake:
  "NoSQL is faster than SQL" — FALSE।
  Properly indexed PostgreSQL beats MongoDB in most cases।
  NoSQL এর advantage = horizontal scale, not raw speed।
```

---

## 16.3 Scaling Decisions

### কখন Scale করবে?

```
Scaling Triggers:

Database CPU > 70% sustained     → Scale up or optimize
Query response time > 100ms P99  → Index, cache, or scale
Disk I/O > 80% capacity          → Scale up storage
Connections > 80% of max         → Connection pooling
Replication lag > 5 seconds      → Scale write path

"Don't scale prematurely."
  First: optimize queries and indexes
  Then: add caching (Redis)
  Then: read replicas
  Then: vertical scaling
  Last: horizontal sharding
```

### Scaling Strategy Decision Tree

```
START: System is slow
         │
         ▼
    Slow queries?
    YES → Add indexes, rewrite queries
    NO  ↓
         ▼
    Read-heavy (> 80% reads)?
    YES → Add read replicas + Redis cache
    NO  ↓
         ▼
    Write-heavy?
    YES → Connection pooling → Vertical scale → Sharding
    NO  ↓
         ▼
    Data too large for one server?
    YES → Partitioning → Sharding
    NO  ↓
         ▼
    Specific bottleneck?
         → Profile and fix
```

### Vertical vs Horizontal

```
Vertical Scaling (Scale Up):
  Bigger server: more CPU, RAM, faster disk
  
  ✅ Simple — no application change
  ✅ ACID maintained
  ❌ Expensive after a point
  ❌ Single point of failure
  ❌ Hardware limit আছে

  When: first line of defense।
  Example: db.r6g.large → db.r6g.4xlarge

Horizontal Scaling (Scale Out):
  More servers: sharding, replicas

  ✅ Theoretically unlimited scale
  ✅ No single point of failure
  ❌ Complex application logic
  ❌ Cross-shard queries কঠিন
  ❌ Distributed transactions expensive

  When: vertical limit hit করার পরে।
```

### Read Replica Strategy

```
Architecture:

App Server
    │
    ├── Writes → Primary DB
    │
    └── Reads  → Read Replica 1
                 Read Replica 2

Django settings.py:
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'HOST': 'primary.db.internal',
    },
    'replica': {
        'ENGINE': 'django.db.backends.postgresql',
        'HOST': 'replica.db.internal',
        'TEST': {'MIRROR': 'default'},
    }
}

# Database router
class PrimaryReplicaRouter:
    def db_for_read(self, model, **hints):
        return 'replica'

    def db_for_write(self, model, **hints):
        return 'default'

    def allow_relation(self, obj1, obj2, **hints):
        return True

    def allow_migrate(self, db, app_label, model_name=None, **hints):
        return db == 'default'

# settings.py
DATABASE_ROUTERS = ['myapp.routers.PrimaryReplicaRouter']

# Specific query on primary (after write):
User.objects.using('default').get(id=1)
```

### Sharding Strategy

```
Sharding = data কে multiple databases এ ভাগ করা।

Shard Key Selection (critical decision!):

Option 1: User ID based
  user_id % 4 = shard number
  ✅ Even distribution (if user IDs are uniform)
  ❌ Hot shard if some users are more active
  
  shard_0: user_id 0,4,8,12...
  shard_1: user_id 1,5,9,13...
  shard_2: user_id 2,6,10,14...
  shard_3: user_id 3,7,11,15...

Option 2: Geographic
  Bangladesh users → shard_bd
  India users      → shard_in
  ✅ Data locality, compliance
  ❌ Uneven if one region much larger

Option 3: Range based
  user_id 1-1M → shard_1
  user_id 1M-2M → shard_2
  ❌ Hot shard at end (new users)

Option 4: Consistent Hashing
  Hash(user_id) → ring position → shard
  ✅ Adding/removing shards easy
  Used by: Cassandra, DynamoDB

Problems with Sharding:
  ❌ Cross-shard JOIN impossible
  ❌ Cross-shard transactions complex
  ❌ Schema changes harder
  ❌ Rebalancing expensive

Rule: শেষ option হিসেবে sharding করো।
```

---

## 16.4 Caching Strategy

### Cache-Aside (Most Common)

```python
# Cache-Aside Pattern
import redis
import json

cache = redis.Redis(host='localhost', port=6379)

def get_product(product_id):
    cache_key = f'product:{product_id}'

    # 1. Cache check
    cached = cache.get(cache_key)
    if cached:
        return json.loads(cached)  # Cache hit

    # 2. DB query (cache miss)
    product = Product.objects.select_related('category').get(id=product_id)
    data = {
        'id': product.id,
        'name': product.name,
        'price': str(product.price),
    }

    # 3. Cache store
    cache.setex(cache_key, 3600, json.dumps(data))  # 1 hour TTL
    return data

def update_product(product_id, data):
    Product.objects.filter(id=product_id).update(**data)
    # Cache invalidate করো
    cache.delete(f'product:{product_id}')
```

### Cache Strategies Comparison

```
Cache-Aside (Lazy Loading):
  App checks cache → miss → DB → store in cache
  ✅ Only caches what's needed
  ❌ First request always slow (cold cache)
  ❌ Cache invalidation manual
  Use: Product pages, user profiles

Write-Through:
  Write to DB + cache simultaneously
  ✅ Cache always fresh
  ❌ Write latency increases
  ❌ Caches data that may never be read
  Use: Session data, user preferences

Write-Behind (Write-Back):
  Write to cache → async flush to DB
  ✅ Fast writes
  ❌ Data loss if cache crashes before flush
  Use: View counters, like counts

Read-Through:
  Cache library handles DB fallback
  ✅ Transparent to application
  Use: With caching libraries (django-cacheops)
```

### Cache Invalidation Strategies

```python
# Strategy 1: TTL (simplest)
cache.setex('product:101', 300, data)  # 5 min TTL

# Strategy 2: Event-based invalidation
from django.db.models.signals import post_save

@receiver(post_save, sender=Product)
def invalidate_product_cache(sender, instance, **kwargs):
    cache.delete(f'product:{instance.id}')
    cache.delete(f'category_products:{instance.category_id}')

# Strategy 3: Cache versioning
def get_cache_key(product_id):
    version = cache.get('product_cache_version') or 1
    return f'product:{product_id}:v{version}'

def invalidate_all_products():
    cache.incr('product_cache_version')  # All old keys become stale
```

---

## 16.5 Migration Strategies

### Zero-Downtime Migration

```
Goal: Production database schema change করো
      কিন্তু app downtime হবে না।

Expand-Contract Pattern (Blue-Green Migration):

Phase 1 — Expand:
  নতুন column যোগ করো (nullable)
  নতুন code deploy: old column + new column দুটোই লেখে
  Old code: old column পড়ে
  New code: new column পড়ে (fallback to old)

Phase 2 — Migrate:
  Background job: old data → new column এ copy করো
  Verify: data correct?

Phase 3 — Contract:
  New code only: new column পড়ে
  Old column drop করো (পরের deployment এ)

Example:
  users.name (single column) → users.first_name + users.last_name

  Step 1: Add first_name, last_name (nullable)
  Step 2: Deploy code that writes to both
  Step 3: Backfill: UPDATE users SET first_name = split_part(name,' ',1)
  Step 4: Verify all rows filled
  Step 5: Deploy code that reads first_name/last_name
  Step 6: Drop name column
```

### Large Table Migration

```python
# ❌ Bad: migrate entire table at once (locks table for hours!)
# ALTER TABLE orders ADD COLUMN tracking_url VARCHAR(500);
# For 100M rows — this locks for 30+ minutes!

# ✅ Good: Online schema change tools

# Option 1: PostgreSQL concurrent index
CREATE INDEX CONCURRENTLY idx_orders_status ON orders(status);
-- No table lock! Takes longer but doesn't block.

# Option 2: pt-online-schema-change (MySQL)
# pt-online-schema-change --alter "ADD COLUMN tracking_url VARCHAR(500)" D=mydb,t=orders

# Option 3: gh-ost (GitHub's tool for MySQL)
# gh-ost --alter "ADD COLUMN tracking_url VARCHAR(500)" --table=orders

# Option 4: Django + django-pg-zero-downtime-migrations
# settings.py
ZERO_DOWNTIME_MIGRATIONS_RAISE_FOR_UNSAFE = True
ZERO_DOWNTIME_MIGRATIONS_LOCK_TIMEOUT = '2s'
ZERO_DOWNTIME_MIGRATIONS_STATEMENT_TIMEOUT = '2s'

# Batch data migration
class Command(BaseCommand):
    def handle(self, *args, **options):
        batch_size = 1000
        last_id = 0

        while True:
            batch = Order.objects.filter(
                id__gt=last_id,
                tracking_url=None
            ).order_by('id')[:batch_size]

            if not batch:
                break

            for order in batch:
                order.tracking_url = generate_tracking_url(order)

            Order.objects.bulk_update(batch, ['tracking_url'])
            last_id = batch[-1].id
            time.sleep(0.1)  # DB কে breathe করতে দাও
```

### Blue-Green Deployment

```
Blue-Green Database Migration:

Current state:
  App v1 → Blue DB (production)

Migration steps:
  1. Green DB = Blue DB এর replica তৈরি করো
  2. Schema migration চালাও Green DB তে
  3. Green DB কে Blue DB এর replica হিসেবে রাখো (sync)
  4. App v2 deploy করো (Green DB এ point করে)
  5. Traffic switch করো: Blue → Green
  6. Monitor করো
  7. Blue DB backup রাখো (rollback এর জন্য)

Risk: Step 4-5 এর মাঝে Blue DB তে যে writes হয়েছে
      সেগুলো Green DB তে sync হতে হবে।
      Replication lag minimize করো।
```

---

## 16.6 Database Evolution

### Schema Evolution Patterns

```
Pattern 1: Additive Changes (safe)
  - Column add (nullable)
  - Table add
  - Index add (CONCURRENTLY)
  - Constraint add (NOT VALID first, then VALIDATE)

Pattern 2: Destructive Changes (dangerous!)
  - Column rename     → Add new + migrate + drop old
  - Column type change → Add new + migrate + drop old
  - Column drop        → Make nullable first, then drop later
  - Table rename       → Create view with old name

Pattern 3: Constraint Changes
  -- Add NOT NULL safely:
  -- Step 1: Add nullable
  ALTER TABLE orders ADD COLUMN status VARCHAR(20);
  
  -- Step 2: Fill data
  UPDATE orders SET status = 'completed' WHERE status IS NULL;
  
  -- Step 3: Add constraint (not validated yet — no full scan!)
  ALTER TABLE orders ADD CONSTRAINT orders_status_not_null
      CHECK (status IS NOT NULL) NOT VALID;
  
  -- Step 4: Validate (background scan, no lock)
  ALTER TABLE orders VALIDATE CONSTRAINT orders_status_not_null;
  
  -- Step 5: Set NOT NULL
  ALTER TABLE orders ALTER COLUMN status SET NOT NULL;
```

### Versioning Your Schema

```sql
-- Django migrations = version control for schema
-- প্রতিটা migration = একটা version

-- Good migration practices:
-- 1. One concern per migration file
-- 2. Always reversible (provide reverse operation)
-- 3. Test on staging before production
-- 4. Never edit existing migrations (create new ones)

-- Django squashmigrations (clean up old migrations)
python manage.py squashmigrations myapp 0001 0050
-- 50টা migration কে 1টাতে compress করে

-- Check migration safety
python manage.py migrate --check  -- pending migrations আছে?
python manage.py showmigrations   -- migration status দেখো
```

---

## 16.7 Multi-Database Architecture

### Polyglot Persistence

```
Real production systems একাধিক database ব্যবহার করে।
প্রতিটা database তার best use case এ।

Example — E-commerce platform:

PostgreSQL:
  - Users, orders, payments, products
  - Reason: ACID, complex queries, relationships

Redis:
  - Sessions, cart, cache, rate limiting
  - Reason: Fast in-memory, TTL support

Elasticsearch:
  - Product search, log analysis
  - Reason: Full-text search, faceted search

MongoDB (optional):
  - Product reviews, user activity logs
  - Reason: Flexible schema, document model

S3 (object storage):
  - Product images, files, backups
  - Reason: Cheap, durable, scalable

Architecture:
  ┌─────────────────────────────────────────┐
  │              App Server                 │
  │                                         │
  │  ┌──────────┐  ┌──────┐  ┌───────────┐ │
  │  │PostgreSQL│  │Redis │  │Elasticsearch│ │
  │  │(primary) │  │(cache│  │(search)    │ │
  │  └──────────┘  └──────┘  └───────────┘ │
  └─────────────────────────────────────────┘
```

### Django Multi-DB Setup

```python
# settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'myapp_primary',
        'HOST': 'postgres.internal',
    },
    'analytics': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'myapp_analytics',
        'HOST': 'analytics-postgres.internal',
    },
}

# Analytics model — always uses analytics DB
class PageView(models.Model):
    path       = models.CharField(max_length=500)
    user_id    = models.BigIntegerField(null=True)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        app_label = 'analytics'

# Router
class AnalyticsRouter:
    def db_for_read(self, model, **hints):
        if model._meta.app_label == 'analytics':
            return 'analytics'
        return 'default'

    def db_for_write(self, model, **hints):
        if model._meta.app_label == 'analytics':
            return 'analytics'
        return 'default'

    def allow_migrate(self, db, app_label, **hints):
        if app_label == 'analytics':
            return db == 'analytics'
        return db == 'default'
```

---

## 16.8 Monitoring and Observability

### Key Metrics to Monitor

```
DATABASE HEALTH METRICS:
  Query response time (P50, P95, P99)
  Queries per second (QPS)
  Active connections
  Connection pool utilization
  Replication lag
  Disk I/O utilization
  Cache hit rate
  Lock wait time
  Deadlock frequency
  Slow query count (> 100ms)

ALERT THRESHOLDS (production):
  P99 query time > 500ms    → Warning
  P99 query time > 2s       → Critical
  Connections > 80% max     → Warning
  Replication lag > 30s     → Critical
  Disk > 80% full           → Warning
  Cache hit rate < 80%      → Warning
  Deadlocks > 10/hour       → Warning
```

### PostgreSQL Monitoring Queries

```sql
-- Slow queries (pg_stat_statements extension)
SELECT
    query,
    calls,
    ROUND(mean_exec_time::numeric, 2) AS avg_ms,
    ROUND(total_exec_time::numeric, 2) AS total_ms,
    rows
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;

-- Active connections
SELECT
    state,
    COUNT(*) AS count,
    MAX(NOW() - query_start) AS max_duration
FROM pg_stat_activity
WHERE datname = 'myapp_db'
GROUP BY state;

-- Lock waits
SELECT
    blocked.pid,
    blocked.query AS blocked_query,
    blocking.pid AS blocking_pid,
    blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
    ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE blocked.cardinality(pg_blocking_pids(blocked.pid)) > 0;

-- Index usage
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan AS times_used,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;
-- idx_scan = 0 মানে index কখনো ব্যবহার হয়নি — drop করো!

-- Table bloat
SELECT
    tablename,
    pg_size_pretty(pg_total_relation_size(tablename::regclass)) AS total_size,
    n_dead_tup AS dead_tuples,
    last_vacuum,
    last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

### Django Debug Toolbar + Logging

```python
# settings.py — Query logging
LOGGING = {
    'version': 1,
    'handlers': {
        'console': {'class': 'logging.StreamHandler'},
    },
    'loggers': {
        'django.db.backends': {
            'handlers': ['console'],
            'level': 'DEBUG',  # সব queries log করো (development only!)
        },
    },
}

# Query counting middleware
class QueryCountMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        from django.db import connection, reset_queries
        reset_queries()

        response = self.get_response(request)

        if settings.DEBUG:
            queries = len(connection.queries)
            if queries > 10:  # N+1 alert
                import logging
                logger = logging.getLogger(__name__)
                logger.warning(f'{request.path} made {queries} DB queries!')

        return response
```

---

## 16.9 Production Incident Patterns

### War Story 1 — Missing Index (Real Scenario)

```
Incident:
  E-commerce site — Black Friday।
  Morning এ সব ঠিক ছিল।
  Afternoon এ traffic 10x বাড়লো।
  Database CPU 100%, site slow।

Root Cause:
  orders table এ 50M rows।
  ORDER BY created_at — কিন্তু index নেই!
  Full table scan on every page load।

Detection:
  pg_stat_statements দেখো:
  SELECT * FROM orders WHERE user_id=? ORDER BY created_at DESC LIMIT 20
  avg_time: 4500ms (should be < 10ms)

Fix:
  CREATE INDEX CONCURRENTLY idx_orders_user_created
      ON orders(user_id, created_at DESC);
  -- 20 minutes to build — no downtime
  -- After: avg_time 3ms

Lesson:
  Load test করো। EXPLAIN ANALYZE করো production data দিয়ে।
  Staging এ 50M rows নিয়ে test করো — 100 rows দিয়ে না।
```

### War Story 2 — N+1 in Production

```
Incident:
  API endpoint — "Get all orders with items"
  Dev environment এ: 50ms
  Production এ (10K orders): 45 seconds!

Root Cause:
  views.py:
    orders = Order.objects.all()
    for order in orders:
        items = order.items.all()  # N+1!
  
  10K orders = 10K+1 queries = 45 seconds

Fix:
  orders = Order.objects.prefetch_related('items').all()
  1+1 = 2 queries = 200ms

Lesson:
  Django Debug Toolbar দিয়ে query count check করো।
  Never loop করে ORM query করো।
  prefetch_related / select_related সবসময় consider করো।
```

### War Story 3 — Connection Pool Exhaustion

```
Incident:
  Production site — suddenly all requests timeout।
  Database সুস্থ, app server সুস্থ।
  কিন্তু সব requests hang করছে।

Root Cause:
  PostgreSQL max_connections = 100
  Django apps: 5 servers × 20 workers = 100 connections
  Plus: admin tools, monitoring = 110 connections!
  New connections: "FATAL: sorry, too many clients"

Fix (immediate):
  Kill idle connections:
  SELECT pg_terminate_backend(pid)
  FROM pg_stat_activity
  WHERE state = 'idle'
    AND query_start < NOW() - INTERVAL '5 minutes';

Fix (permanent):
  Add pgBouncer (connection pooler):
    pgBouncer max_client_conn = 1000
    pgBouncer pool_size = 20 (actual DB connections)
  
  1000 app connections → 20 actual PostgreSQL connections

Lesson:
  Always use connection pooler in production।
  Monitor active connections।
  Alert when > 80% of max_connections।
```

---

## 16.10 Interview Questions

### Beginner Level

**Q1: SQL এবং NoSQL এর মধ্যে পার্থক্য কী?**
> SQL: structured schema, ACID, complex queries with JOINs, vertical scaling। NoSQL: flexible schema, horizontal scaling, eventual consistency (mostly), specific data models। SQL ব্যবহার করো যখন relationships, transactions, complex queries দরকার। NoSQL ব্যবহার করো যখন massive scale, simple access patterns, flexible schema দরকার।

**Q2: Read replica কী এবং কখন ব্যবহার করবে?**
> Read replica = primary DB এর copy যেখানে শুধু reads যায়। Read-heavy application এ (80%+ reads) primary এর load কমায়। Analytics queries, reports, search এ ব্যবহার করো। Writes সবসময় primary তে যাবে। Replication lag সম্পর্কে সচেতন থাকো।

---

### Intermediate Level

**Q3: Zero-downtime migration কীভাবে করবে?**
> Expand-Contract pattern: প্রথমে nullable column যোগ করো, background এ data migrate করো, তারপর old column drop করো। CREATE INDEX CONCURRENTLY ব্যবহার করো। Large table এ batch update করো (1000 rows at a time)। Never ALTER TABLE on large table without a plan।

**Q4: Cache invalidation কীভাবে করবে?**
> TTL based (simplest) — data stale হওয়ার acceptable time set করো। Event based — data change হলে cache delete করো (Django signals)। Cache versioning — global version increment করলে সব old keys stale হয়। Write-through — write এ DB এবং cache দুটোতেই লেখো।

**Q5: Connection pool কেন দরকার?**
> PostgreSQL প্রতিটা connection এ ~5-10MB RAM ব্যবহার করে এবং process fork করে। 1000 app connections = 1000 processes = expensive। pgBouncer 1000 app connections কে 20 real DB connections এ multiplex করে। Production এ সবসময় pgBouncer বা RDS Proxy ব্যবহার করো।

---

### Senior / FAANG Level

**Q6: Design করো একটা URL shortener এর database (bit.ly level — 1B URLs, 10B clicks/day)।**

```
Requirements:
  Write: 1000 URLs/sec (new short URLs)
  Read:  100K redirects/sec
  Storage: 1B URLs × ~200 bytes = 200GB

Schema:
  urls (
    id          BIGSERIAL PRIMARY KEY,
    short_code  VARCHAR(8) UNIQUE NOT NULL,
    long_url    TEXT NOT NULL,
    user_id     BIGINT,
    click_count BIGINT DEFAULT 0,
    created_at  TIMESTAMPTZ DEFAULT NOW()
  )

  clicks (
    id          BIGSERIAL PRIMARY KEY,
    url_id      BIGINT,
    ip          INET,
    referrer    TEXT,
    country     CHAR(2),
    clicked_at  TIMESTAMPTZ DEFAULT NOW()
  ) PARTITION BY RANGE (clicked_at);  -- monthly partitions

Scaling Strategy:
  Redirect (100K/sec reads):
    Redis cache: short_code → long_url (TTL: 24h)
    Cache hit rate: 95%+ (popular URLs)
    Only 5% goes to PostgreSQL

  Click tracking (10B/day = 115K writes/sec):
    Async: write to Redis queue first
    Batch flush to PostgreSQL every second
    Cassandra alternative for click events

  URL storage:
    Single PostgreSQL শুরুতে
    Read replicas যোগ করো
    Shard by short_code prefix if needed

  Short code generation:
    Option 1: Base62(auto-increment id)
    Option 2: Random 8-char + unique check
    Option 3: Distributed ID (Snowflake)
```

**Q7: একটা বড় table এর schema কীভাবে safely migrate করবে (500M rows, 24/7 production)?**

```
Table: orders (500M rows)
Change: VARCHAR(50) status → ENUM type

Step 1: Add new column
  ALTER TABLE orders ADD COLUMN status_new order_status_enum;
  -- Instant! NULL by default।

Step 2: Dual-write
  Deploy code that writes to BOTH status AND status_new।
  Old code reads from status।

Step 3: Backfill (batch job)
  UPDATE orders
  SET status_new = status::order_status_enum
  WHERE id BETWEEN :start AND :end
    AND status_new IS NULL;
  -- 1000 rows at a time, sleep 100ms between batches
  -- 500M ÷ 1000 = 500K batches × 100ms = ~14 hours

Step 4: Verify
  SELECT COUNT(*) FROM orders WHERE status_new IS NULL;
  -- Should be 0

Step 5: Switch reads
  Deploy code that reads status_new।
  Fallback to status if NULL।

Step 6: Remove old column
  ALTER TABLE orders DROP COLUMN status;
  ALTER TABLE orders RENAME COLUMN status_new TO status;

Total downtime: 0
Total duration: ~14 hours backfill (background)
Risk: Minimal — each step reversible
```

---

## 16.11 Hands-on Exercises

### Exercise 1 — Database Selection
```
নিচের systems এর জন্য database choose করো এবং কারণ দাও:
1. Real-time stock price tracking (1M updates/sec)
2. Hospital patient records
3. Twitter-like social media (500M users)
4. E-commerce product catalog with search
5. Gaming leaderboard (top 100 players)
6. IoT temperature sensors (10K devices, 1 reading/sec)
```

### Exercise 2 — Migration Plan
```
users table (10M rows):
  Current: full_name VARCHAR(200)
  Target:  first_name VARCHAR(100) + last_name VARCHAR(100)

Write:
1. Step-by-step migration plan
2. Django migration files
3. Backfill script
4. Rollback plan
```

### Exercise 3 — Caching
```python
# Implement করো:
# 1. get_product_detail() with Redis cache
# 2. Cache invalidation on product update
# 3. Cache warming on app startup
# 4. Cache hit/miss metrics logging
```

### Mini Project
```
Design করো একটা "URL Analytics System" (Google Analytics lite):
  - Page views tracking
  - Unique visitors
  - Top pages report
  - Real-time visitor count

Requirements:
  - 10K page views/second
  - Reports up to 1 year old data
  - Real-time dashboard

Deliverables:
  1. Database selection with justification
  2. Schema design
  3. Django models
  4. Caching strategy
  5. Scaling plan
```

---

## 16.12 Module Summary & Cheat Sheet

```
SYSTEM DESIGN & DATABASE DECISIONS CHEAT SHEET
================================================

DATABASE SELECTION:
  Structured + ACID + JOINs     → PostgreSQL
  Fast key-value + cache + TTL  → Redis
  Full-text search + facets     → Elasticsearch
  Massive writes + time series  → Cassandra / TimescaleDB
  Graph traversal               → Neo4j
  Document + flexible schema    → MongoDB
  Managed + serverless          → DynamoDB

SCALING ORDER:
  1. Optimize queries + indexes
  2. Redis cache (read reduction)
  3. Read replicas
  4. Vertical scaling (bigger server)
  5. Partitioning
  6. Sharding (last resort)

CACHING PATTERNS:
  Cache-Aside   → App controls cache, lazy load
  Write-Through → Write to cache + DB together
  Write-Behind  → Write cache, async flush to DB
  TTL           → Simplest, set expiry time

MIGRATION STRATEGY:
  Expand-Contract → Add new + migrate + drop old
  CONCURRENTLY    → Index without table lock
  Batch update    → 1000 rows, sleep between batches
  Dual-write      → Write old + new during transition
  Blue-Green      → Switch traffic after full migration

CONNECTION POOLING:
  Always use pgBouncer / RDS Proxy
  Pool size = (CPU cores × 2) + disk spindles
  Max connections alert at 80%

MONITORING MUST-HAVES:
  pg_stat_statements  → Slow query detection
  pg_stat_activity    → Active connections
  Replication lag     → Alert > 30s
  Index usage         → Drop unused indexes
  Dead tuples         → Monitor VACUUM

KEY RULES:
  "Scale only when needed — optimize first"
  "Every NoSQL problem can be solved in SQL until 10M+ rows"
  "Cache what you read often, invalidate when you write"
  "Test migrations on production-size data"
  "Zero-downtime migrations are always possible"
  "Monitor before you're in crisis"
  "Connection pooler is not optional in production"
```
