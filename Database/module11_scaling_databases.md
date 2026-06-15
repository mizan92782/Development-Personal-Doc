# MODULE 11: Scaling Databases (ডাটাবেজ স্কেলিং)
## বাংলায় সম্পূর্ণ Database Scaling গাইড — Production Level

> **কেন এই module জীবন বদলে দেবে?**
> তোমার app এ 100 users থেকে 10 million users হলে কী করবে?
> Database scaling না জানলে production এ app crash করবে।
> Amazon, Netflix, Uber — সবাই এই challenges face করেছে।
> Senior engineer interview এ system design এ এই topic mandatory।

---

## 11.1 কখন Scale করতে হবে?

### Warning Signs

```
Database Scaling দরকার যখন:

1. Query response time > 100ms (simple queries)
2. CPU usage > 70% consistently  
3. Memory usage > 80%
4. Disk I/O wait > 20%
5. Connection pool exhausted (too many clients)
6. Replication lag বাড়ছে
7. VACUUM পিছিয়ে পড়ছে (bloat বাড়ছে)

Monitoring করো:
  - pg_stat_activity (active queries)
  - pg_stat_bgwriter (buffer hit rate)
  - pg_stat_replication (replica lag)
  - OS metrics: CPU, RAM, Disk I/O
```

### Scaling Decision Tree

```
App slow হচ্ছে?
      ↓
Query optimize করা হয়েছে? → না → আগে optimize করো (Module 7)
      ↓ হ্যাঁ
Index ঠিক আছে? → না → Index যোগ করো (Module 6)  
      ↓ হ্যাঁ
Read heavy নাকি Write heavy?
      ↓              ↓
   Read Heavy     Write Heavy
      ↓              ↓
Read Replica    Vertical Scaling আগে
Connection Pool তারপর Sharding
Caching (Redis)
```

---

## 11.2 Vertical Scaling (ভার্টিকাল স্কেলিং)

### কী এবং কেন?

```
Vertical Scaling = একটা server এর resource বাড়াও
  CPU: 4 core → 16 core → 64 core
  RAM: 16GB → 64GB → 256GB
  Storage: HDD → SSD → NVMe SSD
  Network: 1Gbps → 10Gbps

"Scale Up" নামেও পরিচিত
```

### সুবিধা এবং অসুবিধা

```
✅ সুবিধা:
  - Simple — app code change দরকার নেই
  - No distributed complexity
  - Strong consistency maintained
  - Quick fix

❌ অসুবিধা:
  - Expensive (high-end servers costly)
  - Hard limit আছে (max server size)
  - Single point of failure
  - Downtime দরকার হতে পারে upgrade এ
```

### কখন যথেষ্ট?

```
Vertical scaling যথেষ্ট যদি:
  - Read/Write ratio moderate
  - Data size manageable (< 1TB)
  - Traffic predictable
  - Budget আছে

Instagram প্রথম 1 million users পর্যন্ত single PostgreSQL server এ চলেছে।
Stack Overflow এখনো মূলত vertical scaling এ চলে (very optimized queries)।
```

### PostgreSQL Vertical Scaling Config

```sql
-- বড় server এ config update করো (postgresql.conf)

-- RAM 64GB server এর জন্য:
shared_buffers = 16GB           -- RAM এর 25%
effective_cache_size = 48GB     -- RAM এর 75%
work_mem = 256MB                -- Sort/Hash per query
maintenance_work_mem = 2GB      -- VACUUM/INDEX

-- CPU 32 cores এর জন্য:
max_worker_processes = 32
max_parallel_workers = 16
max_parallel_workers_per_gather = 8
max_parallel_maintenance_workers = 4

-- NVMe SSD এর জন্য:
random_page_cost = 1.0
effective_io_concurrency = 300
```

---

## 11.3 Horizontal Scaling (হরিজন্টাল স্কেলিং)

### কী এবং কেন?

```
Horizontal Scaling = একাধিক server যোগ করো
  "Scale Out"

Types:
  1. Read Replicas    → Read load distribute
  2. Sharding         → Write load distribute
  3. Partitioning     → Single server data management

Architecture:

Single Server:
  [App] → [DB]

Horizontal:
  [App] → [Primary DB] (writes)
        ↘ [Replica 1]  (reads)
        ↘ [Replica 2]  (reads)
        ↘ [Replica 3]  (reads)
```

---

## 11.4 Read Replicas (রিড রেপ্লিকা)

### কীভাবে কাজ করে

```
Primary DB:
  - সব writes handle করে
  - WAL generate করে
  - Replicas তে WAL stream করে

Replica DB:
  - WAL apply করে (replay)
  - Read-only queries handle করে
  - Primary এর behind থাকতে পারে (replication lag)

Traffic split:
  Write operations → Primary
  Read operations  → Replica (load balanced)

Instagram এ reads/writes ratio ≈ 99:1
তাই replicas read load dramatically কমায়
```

### Django Read Replica Setup

```python
# settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        'HOST': 'primary.db.internal',
        'PORT': '5432',
        'USER': 'app_user',
        'PASSWORD': 'secret',
    },
    'replica1': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        'HOST': 'replica1.db.internal',
        'PORT': '5432',
        'USER': 'app_user',
        'PASSWORD': 'secret',
        'TEST': {'MIRROR': 'default'},
    },
    'replica2': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        'HOST': 'replica2.db.internal',
        'PORT': '5432',
        'USER': 'app_user',
        'PASSWORD': 'secret',
        'TEST': {'MIRROR': 'default'},
    },
}

DATABASE_ROUTERS = ['myapp.routers.ReplicaRouter']
```

```python
# myapp/routers.py
import random

class ReplicaRouter:
    REPLICA_DATABASES = ['replica1', 'replica2']

    def db_for_read(self, model, **hints):
        # Read replicas এ load balance করো
        return random.choice(self.REPLICA_DATABASES)

    def db_for_write(self, model, **hints):
        return 'default'  # সব writes primary তে

    def allow_relation(self, obj1, obj2, **hints):
        db_set = {'default', 'replica1', 'replica2'}
        if obj1._state.db in db_set and obj2._state.db in db_set:
            return True
        return None

    def allow_migrate(self, db, app_label, model_name=None, **hints):
        return db == 'default'
```

```python
# Views এ ব্যবহার
class OrderListView(generics.ListAPIView):
    def get_queryset(self):
        # Automatically replica তে যাবে (router এর কারণে)
        return Order.objects.filter(status='pending').select_related('customer')

# Manual override — specific DB choose করো
critical_data = Order.objects.using('default').get(id=order_id)
# Write করার পরপরই read করলে primary থেকে পড়ো (replication lag avoid)
```

### Replication Lag Handle করা

```python
# সমস্যা: Write করার পরপরই Read করলে replica lag এর কারণে
# নতুন data দেখা নাও যেতে পারে

# ❌ এই pattern এ replica lag সমস্যা
def create_order_and_return(user_id, items):
    order = Order.objects.create(user_id=user_id)  # Primary write
    return Order.objects.get(id=order.id)           # Replica read → might not exist yet!

# ✅ Write এর পরে primary থেকে পড়ো
def create_order_and_return(user_id, items):
    order = Order.objects.create(user_id=user_id)  # Primary write
    return Order.objects.using('default').get(id=order.id)  # Primary read

# ✅ অথবা: write করা object ই return করো
def create_order_and_return(user_id, items):
    order = Order.objects.create(user_id=user_id)
    return order  # already in memory, no extra query needed
```

---

## 11.5 Sharding (শার্ডিং)

### কী এবং কেন?

```
Sharding = Data কে multiple database servers এ ভাগ করো
(Horizontal Partitioning across multiple machines)

কখন দরকার:
  - Single server এর disk capacity exceed করছে
  - Write throughput single server handle করতে পারছে না
  - Vertical scaling cost-effective না

Example — Instagram User Sharding:
  Shard 1: user_id 1 - 10,000,000
  Shard 2: user_id 10,000,001 - 20,000,000
  Shard 3: user_id 20,000,001 - 30,000,000
  ...

প্রতিটা shard আলাদা PostgreSQL server
```

### Sharding Strategies

```
1. Range-based Sharding:
   user_id 1-1M     → Shard 1
   user_id 1M-2M    → Shard 2
   
   ✅ Simple, range queries efficient
   ❌ Hotspot: নতুন users সব একটা shard এ যায়

2. Hash-based Sharding:
   shard = hash(user_id) % total_shards
   
   ✅ Even distribution
   ❌ Range queries সব shards এ যেতে পারে
   ❌ Shard count বাড়ালে resharding দরকার

3. Directory-based Sharding:
   Lookup table: user_id → shard_id
   
   ✅ Flexible
   ❌ Lookup table bottleneck হতে পারে

4. Geographic Sharding:
   Bangladesh users → Shard BD
   India users      → Shard IN
   
   ✅ Data locality, compliance
   ❌ Cross-region queries কঠিন
```

### Django তে Manual Sharding

```python
# settings.py — multiple shard databases
DATABASES = {
    'default': {...},   # metadata/config
    'shard_0': {'HOST': 'shard0.db.internal', ...},
    'shard_1': {'HOST': 'shard1.db.internal', ...},
    'shard_2': {'HOST': 'shard2.db.internal', ...},
    'shard_3': {'HOST': 'shard3.db.internal', ...},
}

SHARD_COUNT = 4

# Sharding utility
def get_shard(user_id: int) -> str:
    shard_num = user_id % SHARD_COUNT
    return f'shard_{shard_num}'

# Router
class ShardRouter:
    def db_for_read(self, model, **hints):
        if 'user_id' in hints:
            return get_shard(hints['user_id'])
        return 'default'

    def db_for_write(self, model, **hints):
        if 'user_id' in hints:
            return get_shard(hints['user_id'])
        return 'default'

# Usage
def get_user_orders(user_id):
    shard = get_shard(user_id)
    return Order.objects.using(shard).filter(user_id=user_id)

def create_order(user_id, items):
    shard = get_shard(user_id)
    with transaction.atomic(using=shard):
        order = Order.objects.using(shard).create(user_id=user_id)
        OrderItem.objects.using(shard).bulk_create([
            OrderItem(order=order, **item) for item in items
        ])
    return order
```

### Sharding Challenges

```
1. Cross-shard Queries:
   "সব users এর total revenue" → সব shards query করতে হবে
   Solution: Application-level aggregation অথবা analytics DB আলাদা

2. Cross-shard Transactions:
   User A (Shard 1) থেকে User B (Shard 2) তে transfer
   → Distributed transaction দরকার (2PC)
   → অনেক complex, avoid করার চেষ্টা করো

3. Resharding:
   4 shards → 8 shards = data migration nightmare
   Solution: Consistent hashing (minimal resharding)

4. Hot Shards:
   একটা shard এ বেশি traffic
   Solution: Virtual shards + redistribution

Senior Tip: Sharding হলো last resort।
আগে: optimize queries → add indexes → vertical scale → read replicas → caching
তারপর sharding।
```

---

## 11.6 Connection Pooling (কানেকশন পুলিং)

### সমস্যা

```
PostgreSQL এ প্রতিটা connection:
  - ~5-10MB RAM নেয়
  - Process fork overhead আছে
  - Connection setup time ~50ms

Django app এ:
  - প্রতিটা request এ connection নেয়
  - 1000 concurrent requests = 1000 connections
  - 1000 * 10MB = 10GB শুধু connections এ!
  - PostgreSQL max_connections = 200-500 সাধারণত

Solution: Connection Pooling
```

### pgBouncer Deep Dive

```
pgBouncer Modes:

1. Session Mode:
   Client → pgBouncer connection = DB connection (1:1)
   Client disconnect হলে DB connection release
   Simple but not much better than direct connection

2. Transaction Mode (Recommended):
   Client → Transaction চলার সময় DB connection নেয়
   Transaction শেষে DB connection pool এ ফেরত
   1000 clients, কিন্তু মাত্র 20-50 DB connections!

3. Statement Mode:
   প্রতিটা statement এর জন্য connection নেয়/ছাড়ে
   Multi-statement transactions কাজ করে না
   Rarely used
```

```ini
# /etc/pgbouncer/pgbouncer.ini

[databases]
mydb = host=127.0.0.1 port=5432 dbname=mydb

[pgbouncer]
listen_port = 6432
listen_addr = *
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt

# Pool settings
pool_mode = transaction
max_client_conn = 2000         # App থেকে max connections
default_pool_size = 25         # PostgreSQL actual connections per DB
min_pool_size = 5
reserve_pool_size = 10
reserve_pool_timeout = 5

# Timeouts
server_idle_timeout = 600      # Idle server connection timeout
client_idle_timeout = 0        # Disable client idle timeout
server_connect_timeout = 15
query_timeout = 0              # No query timeout (set in app)

# Logging
logfile = /var/log/pgbouncer/pgbouncer.log
pidfile = /var/run/pgbouncer/pgbouncer.pid
```

```python
# Django settings pgBouncer সহ
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        'USER': 'app_user',
        'PASSWORD': 'secret',
        'HOST': 'localhost',
        'PORT': '6432',           # pgBouncer port!
        'CONN_MAX_AGE': 0,        # pgBouncer manages connections
        'OPTIONS': {
            # Transaction mode এ prepared statements কাজ করে না
            'options': '-c default_transaction_isolation=read\\ committed'
        },
        'DISABLE_SERVER_SIDE_CURSORS': True,  # pgBouncer transaction mode
    }
}
```

### pgBouncer Monitoring

```sql
-- pgBouncer admin console (psql -p 6432 pgbouncer)
SHOW POOLS;
-- database | user | cl_active | cl_waiting | sv_active | sv_idle
-- mydb     | app  | 150       | 0          | 20        | 5

SHOW CLIENTS;
SHOW SERVERS;
SHOW STATS;
-- total_requests, total_query_time, avg_query_time

SHOW CONFIG;
```

---

## 11.7 Caching Strategies (ক্যাশিং স্ট্র্যাটেজি)

### কেন Caching দরকার?

```
Database query: 10ms - 100ms
Cache (Redis) read: 0.1ms - 1ms
= 100x faster!

Caching কখন ব্যবহার করবে:
✅ Read-heavy data (product catalog, user profile)
✅ Expensive computation (analytics, reports)
✅ Session data
✅ Rate limiting counters
✅ Frequently accessed, rarely changed data

Caching এড়াবে:
❌ Real-time data (live stock price, chat messages)
❌ User-specific sensitive data (without proper isolation)
❌ Data যেটা প্রতি request এ আলাদা
```

### Cache Patterns

```
1. Cache-Aside (Lazy Loading) — Most Common
   App → Cache check → Hit? Return data
                    → Miss? DB query → Cache store → Return

2. Write-Through
   App → Write to Cache → Write to DB
   Always consistent, write overhead

3. Write-Behind (Write-Back)
   App → Write to Cache → Return (fast!)
   Async: Cache → DB write (later)
   Fast writes, risk of data loss

4. Read-Through
   App → Cache (cache fetches from DB if miss)
   Cache manages DB interaction
```

### Redis Integration — Django

```python
# pip install django-redis redis

# settings.py
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
            'SOCKET_CONNECT_TIMEOUT': 5,
            'SOCKET_TIMEOUT': 5,
            'CONNECTION_POOL_KWARGS': {'max_connections': 100},
            'COMPRESSOR': 'django_redis.compressors.zlib.ZlibCompressor',
        },
        'KEY_PREFIX': 'myapp',
        'TIMEOUT': 300,  # 5 minutes default TTL
    }
}

# Session backend
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
SESSION_CACHE_ALIAS = 'default'
```

```python
# Cache-Aside Pattern
from django.core.cache import cache
from django.utils.functional import cached_property
import hashlib

def get_product(product_id):
    cache_key = f'product:{product_id}'
    
    # Cache check
    product_data = cache.get(cache_key)
    if product_data:
        return product_data  # Cache hit!
    
    # Cache miss → DB query
    product = Product.objects.select_related('category').get(id=product_id)
    product_data = {
        'id':       product.id,
        'name':     product.name,
        'price':    str(product.price),
        'category': product.category.name,
    }
    
    # Cache store
    cache.set(cache_key, product_data, timeout=3600)  # 1 hour TTL
    return product_data

# Cache invalidation — product update হলে cache clear করো
def update_product(product_id, **kwargs):
    Product.objects.filter(id=product_id).update(**kwargs)
    cache.delete(f'product:{product_id}')  # Invalidate cache
```

### Cache Invalidation Strategies

```python
# Strategy 1: TTL based (simple)
cache.set('key', value, timeout=300)  # 5 min TTL
# Data stale হতে পারে কিন্তু auto-expire হবে

# Strategy 2: Event-based invalidation
from django.db.models.signals import post_save, post_delete

@receiver(post_save, sender=Product)
def invalidate_product_cache(sender, instance, **kwargs):
    cache.delete(f'product:{instance.id}')
    cache.delete('product_list')  # List cache ও clear করো

@receiver(post_delete, sender=Product)
def invalidate_product_cache_on_delete(sender, instance, **kwargs):
    cache.delete(f'product:{instance.id}')

# Strategy 3: Cache versioning
def get_user_profile(user_id):
    version = cache.get(f'user_version:{user_id}', 1)
    cache_key = f'user:{user_id}:v{version}'
    
    data = cache.get(cache_key)
    if not data:
        user = User.objects.get(id=user_id)
        data = {'name': user.name, 'email': user.email}
        cache.set(cache_key, data, timeout=3600)
    
    return data

def invalidate_user_cache(user_id):
    # Version bump — old cache key automatically stale
    cache.incr(f'user_version:{user_id}', ignore_key_missing=True)
```

### Cache Warming

```python
# App start হলে বা scheduled task এ important data cache করো

@shared_task
def warm_product_cache():
    """Top 1000 products cache করো"""
    products = Product.objects.select_related('category').order_by(
        '-view_count'
    )[:1000]
    
    pipeline = []
    for product in products:
        cache_key = f'product:{product.id}'
        data = {
            'id': product.id,
            'name': product.name,
            'price': str(product.price),
        }
        cache.set(cache_key, data, timeout=3600)
    
    print(f"Warmed cache for {len(products)} products")
```

### Redis Advanced Patterns

```python
import redis

r = redis.Redis(host='localhost', port=6379, db=0, decode_responses=True)

# Rate Limiting
def check_rate_limit(user_id, limit=100, window=60):
    """Per-minute rate limiting"""
    key = f'rate_limit:{user_id}:{int(time.time() // window)}'
    
    pipe = r.pipeline()
    pipe.incr(key)
    pipe.expire(key, window)
    count, _ = pipe.execute()
    
    return count <= limit

# Distributed Lock
import uuid

def acquire_lock(resource, timeout=10):
    lock_key = f'lock:{resource}'
    lock_value = str(uuid.uuid4())
    
    acquired = r.set(lock_key, lock_value, nx=True, ex=timeout)
    return lock_value if acquired else None

def release_lock(resource, lock_value):
    lock_key = f'lock:{resource}'
    current = r.get(lock_key)
    if current == lock_value:
        r.delete(lock_key)
        return True
    return False

# Usage: Flash sale stock management
def purchase_item(product_id, user_id):
    lock_value = acquire_lock(f'product:{product_id}', timeout=5)
    if not lock_value:
        raise Exception("Could not acquire lock, try again")
    
    try:
        stock = int(r.get(f'stock:{product_id}') or 0)
        if stock <= 0:
            raise Exception("Out of stock")
        
        r.decr(f'stock:{product_id}')
        create_order_async.delay(user_id, product_id)
        return True
    finally:
        release_lock(f'product:{product_id}', lock_value)

# Leaderboard
def update_score(user_id, score):
    r.zadd('leaderboard', {str(user_id): score})

def get_top_users(count=10):
    return r.zrevrange('leaderboard', 0, count - 1, withscores=True)

def get_user_rank(user_id):
    return r.zrevrank('leaderboard', str(user_id))
```

---

## 11.8 Complete Scaling Architecture

### Stage 1: Single Server (0 → 10K users)

```
[Django App] → [PostgreSQL]

Config:
  - Optimize queries
  - Add indexes
  - Connection pooling (pgBouncer)
  - Basic caching (Redis)
```

### Stage 2: Read Replicas (10K → 500K users)

```
[Django App] → [pgBouncer] → [Primary PostgreSQL] (writes)
                           ↘ [Replica 1] (reads, reports)
                           ↘ [Replica 2] (reads)

[Redis Cache] ← [Django App]

Config:
  - Read replica router
  - Redis caching for expensive queries
  - CDN for static assets
```

### Stage 3: Multiple App + Cache Layer (500K → 5M users)

```
[Load Balancer]
     ↓
[App Server 1] [App Server 2] [App Server 3]
     ↓               ↓               ↓
[Redis Cluster] (shared cache, sessions)
     ↓
[pgBouncer]
     ↓
[Primary DB] → [Replica 1] [Replica 2] [Replica 3]

Config:
  - Horizontal app scaling
  - Redis Cluster
  - Analytics DB আলাদা (reporting)
  - Background jobs (Celery + Redis)
```

### Stage 4: Partitioning + Sharding (5M+ users)

```
[Load Balancer]
     ↓
[API Gateway]
     ↓
[App Servers (auto-scaled)]
     ↓
[Redis Cluster]
     ↓
[pgBouncer Pool]
     ↓
┌─────────────────────────────────┐
│  Shard 1  │  Shard 2  │ Shard 3 │
│ (users    │ (users    │ (users  │
│  0-10M)   │ 10M-20M)  │ 20M+)   │
│ Primary   │ Primary   │ Primary │
│ + Replica │ + Replica │+Replica │
└─────────────────────────────────┘
     ↓
[Analytics DB] (separate PostgreSQL/Redshift)
[Search] (Elasticsearch)
[Archive] (S3 + Athena)
```

---

## 11.9 Real-world Scaling Examples

### Uber এর Database Scaling

```
Early days (2009):
  Single MySQL database
  
Growth phase:
  MySQL → MySQL Cluster (sharding)
  Dispatch system: separate DB
  
Current:
  Thousands of microservices
  Each service has its own database
  Mix: PostgreSQL, MySQL, Cassandra, Redis
  
Lesson: Start simple, scale as needed.
Don't over-engineer upfront.
```

### Instagram এর Scaling Journey

```
2010 (Launch): PostgreSQL + PostGIS (single server)
2011 (1M users): Added read replicas
2012 (10M users): Sharding by user_id (Django Sharding)
2013 (100M users): Cassandra for feeds
2016 (500M users): Custom sharding framework

Instagram এর Django sharding code:
  - 12,000 PostgreSQL shards!
  - Each shard = logical unit (not physical server)
  - Multiple logical shards on one physical server
  - Can move shards between servers for rebalancing
```

### WhatsApp এর Approach

```
2M concurrent connections, single Erlang server!
Database: Mnesia (Erlang distributed DB)
PostgreSQL for user data

Lesson: Right tool for right job.
Messaging: in-memory/distributed DB
User data: PostgreSQL
```

---

## 11.10 Trade-offs Summary

```
┌────────────────┬──────────────────┬──────────────────┬──────────────────┐
│ Strategy       │ Complexity       │ When to Use      │ Trade-off        │
├────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Vertical Scale │ Low              │ First step       │ Cost, hard limit │
│ Read Replica   │ Low-Medium       │ Read-heavy       │ Replication lag  │
│ Caching        │ Medium           │ Read-heavy       │ Stale data       │
│ Partitioning   │ Medium           │ Large tables     │ Complex queries  │
│ Sharding       │ High             │ Last resort      │ No cross-shard   │
│ CQRS           │ High             │ Complex domains  │ Eventual consst. │
└────────────────┴──────────────────┴──────────────────┴──────────────────┘
```

---

## 11.11 Common Mistakes

### ভুল ১: Premature Sharding

```
❌ 10,000 users এ sharding implement করা
→ Unnecessary complexity
→ Cross-shard transactions nightmare
→ Debugging কঠিন

✅ Optimize queries → Index → Vertical scale → Read replica → Cache
→ তারপর sharding consider করো
```

### ভুল ২: Cache Invalidation না ভাবা

```python
# ❌ Cache invalidate না করে update
def update_product_price(product_id, new_price):
    Product.objects.filter(id=product_id).update(price=new_price)
    # Cache এ পুরনো price থাকবে!

# ✅ Cache invalidate করো
def update_product_price(product_id, new_price):
    Product.objects.filter(id=product_id).update(price=new_price)
    cache.delete(f'product:{product_id}')
    cache.delete(f'product_list:*')  # List caches ও clear
```

### ভুল ৩: Replication Lag Ignore করা

```python
# ❌ Write করার পরপরই replica থেকে read
order = Order.objects.create(...)   # Primary write
orders = Order.objects.filter(...)  # Replica read → might miss new order!

# ✅ Write এর পরে primary থেকে read অথবা delay দাও
order = Order.objects.create(...)
# Redirect to primary for next request
request.session['use_primary'] = True
```

### ভুল ৪: Redis কে Database হিসেবে ব্যবহার করা

```
❌ সব data Redis এ store করা
→ Redis restart করলে data হারায় (persistence configure না করলে)
→ Memory expensive
→ Complex queries সম্ভব না

✅ Redis শুধু cache, session, queue এর জন্য
Primary data সবসময় PostgreSQL এ
```

---

## 11.12 Production Best Practices

```
1.  Scaling এর আগে সবসময় query optimize করো
2.  pgBouncer connection pooling সবসময় ব্যবহার করো
3.  Read replicas দিয়ে read load কমাও
4.  Redis caching expensive queries এর জন্য
5.  Cache TTL সঠিক set করো — too long = stale, too short = no benefit
6.  Cache invalidation strategy পরিকল্পনা করো আগে থেকে
7.  Replication lag monitor করো — alert set করো > 1 second lag
8.  Sharding last resort — complexity justify করতে হবে
9.  Analytics DB আলাদা রাখো (OLAP vs OLTP)
10. Monitoring: Grafana + Prometheus + pg_stat_statements
11. Load test করো scale করার আগে
12. Gradual rollout করো scaling changes এ
```

---

## 11.13 Senior Engineer Insights

> **War Story — Shopify:**
> Shopify 2015 সালে Black Friday তে database bottleneck এ পড়ে।
> Solution: Read replicas + aggressive caching।
> Product catalog Redis এ cache করার পরে DB load 80% কমে।
> Lesson: Caching সবচেয়ে easy এবং effective first scaling step।

> **Amazon এর Rule:**
> "Each service owns its data." Microservice architecture এ প্রতিটা service এর নিজের database।
> এটা scaling easy করে কিন্তু cross-service queries কঠিন করে।
> Trade-off: autonomy vs query simplicity।

> **Design Decision:**
> Cache করার আগে প্রশ্ন করো:
> "এই data কতবার পরিবর্তন হয়? Stale data acceptable? Cache invalidation কীভাবে হবে?"
> এই তিনটা প্রশ্নের উত্তর না জেনে cache করো না।

---

## 11.14 Interview Questions

### Beginner Level

**Q1: Vertical scaling এবং horizontal scaling এর পার্থক্য কী?**
> Vertical: একটা server এর resource (CPU/RAM) বাড়াও। Simple কিন্তু limit আছে।
> Horizontal: একাধিক server যোগ করো। Unlimited scale কিন্তু complexity বাড়ে।

**Q2: Read replica কেন ব্যবহার করা হয়?**
> Read-heavy application এ read load multiple servers এ distribute করতে। Primary server শুধু writes handle করে। Reporting queries, analytics কে replica তে route করলে primary এর উপর চাপ কমে।

**Q3: Connection pooling কেন দরকার?**
> PostgreSQL এ প্রতিটা connection expensive (RAM, process). pgBouncer pool maintain করে — অনেক app connections কে কম DB connections এ map করে। Performance ও reliability বাড়ে।

---

### Intermediate Level

**Q4: Cache invalidation এর বিভিন্ন strategy কী?**
> TTL-based: সময় পরে auto-expire। Simple কিন্তু stale window আছে।
> Event-based: Data change হলে cache delete। Signal বা post_save দিয়ে।
> Version-based: Key তে version যোগ করো। Old key automatically obsolete।
> Write-through: Write হলেই cache update। Consistent কিন্তু write overhead।

**Q5: Sharding কখন করবে এবং কোন কোন চ্যালেঞ্জ আছে?**
> Single server handle করতে পারছে না তখন। চ্যালেঞ্জ: cross-shard queries কঠিন, distributed transactions complex, resharding painful, hotspot possible। তাই last resort।

**Q6: Replication lag কীভাবে handle করবে?**
> Write এর পরপরই critical read primary থেকে করো (using('default'))। Session তে flag রাখো। Replication lag monitor করো। Lag বেশি হলে alert পাঠাও। Application design করো eventual consistency accept করে।

---

### Senior / FAANG Level

**Q7: 10 million users এর e-commerce platform এর database scaling architecture design করো।**

```
Layer 1 — Application:
  - Django app horizontally scaled (3-10 instances)
  - Stateless (sessions in Redis)
  - Background jobs (Celery + Redis queue)

Layer 2 — Caching:
  - Redis Cluster (6 nodes: 3 primary + 3 replica)
  - Product catalog: 1hr TTL
  - User session: 24hr TTL
  - Shopping cart: Redis (real-time)
  - Rate limiting: Redis counters

Layer 3 — Database:
  Primary PostgreSQL (writes):
    - order creation
    - payment processing
    - inventory updates

  Read Replica × 3 (reads):
    - product listing
    - search
    - user profile
    - order history

  Analytics PostgreSQL (separate):
    - Reports
    - Dashboards
    - Routed separately

Layer 4 — Partitioning:
  - orders: PARTITION BY RANGE (created_at) monthly
  - events: PARTITION BY RANGE (created_at) daily
  - Old partitions → archive to S3

Layer 5 — Search:
  - Elasticsearch for product search
  - Sync from PostgreSQL via Celery

Monitoring:
  - Grafana + Prometheus
  - pg_stat_statements
  - Redis INFO stats
  - Replication lag alerts
```

**Q8: Cache stampede কী এবং কীভাবে prevent করবে?**

```
Cache Stampede (Thundering Herd):
  - Popular cache key expire হলো
  - একসাথে হাজার requests DB তে যায়
  - DB overwhelmed হয়

Solutions:

1. Mutex Lock:
   Cache miss হলে lock নাও।
   একটাই DB query করো।
   বাকিরা wait করো।

2. Probabilistic Early Expiry:
   TTL শেষ হওয়ার আগেই probabilistically refresh করো।

3. Background Refresh:
   TTL এর কাছাকাছি হলে background এ refresh করো।
   User পুরনো data দেখে কিন্তু stampede হয় না।

4. Staggered TTL:
   cache.set(key, value, timeout=300 + random.randint(0, 60))
   সব keys একসাথে expire হয় না।
```

```python
# Mutex Lock implementation
import redis
import time

r = redis.Redis()

def get_with_lock(cache_key, fetch_fn, ttl=300, lock_timeout=10):
    # Try cache first
    data = cache.get(cache_key)
    if data:
        return data

    # Cache miss — try to acquire lock
    lock_key = f'lock:{cache_key}'
    lock_value = str(uuid.uuid4())
    acquired = r.set(lock_key, lock_value, nx=True, ex=lock_timeout)

    if acquired:
        try:
            data = fetch_fn()  # DB query
            cache.set(cache_key, data, timeout=ttl)
            return data
        finally:
            r.delete(lock_key)
    else:
        # Lock না পেলে wait করো
        for _ in range(20):
            time.sleep(0.5)
            data = cache.get(cache_key)
            if data:
                return data
        # Still no data — fetch directly
        return fetch_fn()
```

---

## 11.15 Hands-on Exercises

### Exercise 1
```python
# Read Replica setup করো:
# 1. Django settings এ 2টা replica configure করো
# 2. Router লেখো
# 3. Analytics queries replica তে route করো
# 4. Write এর পরে primary থেকে read করো
```

### Exercise 2
```python
# Redis caching implement করো:
# 1. Product detail cache (TTL: 1hr)
# 2. Product list cache (TTL: 5min)
# 3. post_save signal এ cache invalidation
# 4. Cache hit rate measure করো
```

### Exercise 3
```python
# pgBouncer setup করো:
# 1. pgbouncer.ini configure করো
# 2. Django এ pgBouncer port ব্যবহার করো
# 3. Connection count before/after compare করো
# 4. pgBouncer stats দেখাও
```

### Mini Project
```
Complete Scaling Setup:
1. Primary + 2 Replica PostgreSQL
2. pgBouncer connection pooling
3. Redis cache layer
4. Django router
5. Cache invalidation via signals
6. Monitoring queries (lag, hit rate, slow queries)
7. Load test: 1000 concurrent requests
   before scaling vs after scaling compare করো
```

---

## 11.16 Module Summary & Cheat Sheet

```
DATABASE SCALING CHEAT SHEET
==============================

SCALING ORDER (আগে → পরে):
  1. Query optimize করো
  2. Index যোগ করো
  3. Vertical scaling (more RAM/CPU)
  4. Connection pooling (pgBouncer)
  5. Caching (Redis)
  6. Read replicas
  7. Partitioning (large tables)
  8. Sharding (last resort!)

READ REPLICA:
  - Django DATABASE_ROUTERS ব্যবহার করো
  - db_for_read() → replica
  - db_for_write() → default (primary)
  - Write এর পরে primary থেকে read করো
  - Replication lag monitor করো

SHARDING:
  shard_id = hash(user_id) % SHARD_COUNT
  Cross-shard queries avoid করো
  Resharding plan করো upfront

CONNECTION POOLING (pgBouncer):
  pool_mode = transaction    (best)
  max_client_conn = 2000     (app connections)
  default_pool_size = 25     (DB connections)
  Django port = 6432 (pgBouncer)

REDIS CACHING:
  Cache-aside: try cache → miss → DB → cache store
  TTL: short for frequently changing, long for static
  Invalidation: delete on update (signal/post_save)
  Stampede: mutex lock অথবা staggered TTL

CACHE TTL GUIDELINES:
  User session:       24 hours
  Product catalog:    1 hour
  Product list:       5-10 minutes
  User profile:       30 minutes
  Dashboard stats:    5 minutes
  Real-time data:     No cache!

REDIS PATTERNS:
  cache.get/set/delete     → Basic cache
  r.incr/decr              → Counters (rate limit)
  r.zadd/zrevrange         → Leaderboard (sorted set)
  r.setnx                  → Distributed lock
  r.pipeline()             → Batch commands

MONITORING:
  Replication lag: pg_stat_replication
  Cache hit rate: pg_statio_user_tables
  Slow queries: pg_stat_statements
  Active conns: pg_stat_activity
  Redis: INFO stats, MONITOR

TRADE-OFFS:
  Replica     → Lag, eventual consistency
  Cache       → Stale data, invalidation complexity
  Sharding    → No cross-shard, resharding pain
  Partitioning → Complex cross-partition queries
```
