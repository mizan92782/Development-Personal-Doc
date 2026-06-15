# MODULE 12: NoSQL Fundamentals
## বাংলায় সম্পূর্ণ NoSQL গাইড — Production Level

> **কেন এই module দরকার?**
> "SQL নাকি NoSQL?" — এটা senior engineer interview এর সবচেয়ে common প্রশ্ন।
> সঠিক database choose না করলে পরে migrate করা nightmare।
> Netflix, Facebook, Amazon — সবাই SQL এবং NoSQL দুটোই ব্যবহার করে।
> জানতে হবে কখন কোনটা।

---

## 12.1 কেন NoSQL এর জন্ম হলো?

### SQL এর সীমাবদ্ধতা

```
2000s এর শেষে internet explosion:
  - Facebook: billions of users
  - Google: petabytes of data
  - Amazon: millions of products

SQL database এর সমস্যা:
  1. Fixed schema — নতুন field যোগ করতে ALTER TABLE (expensive!)
  2. Horizontal scaling কঠিন — JOIN across servers জটিল
  3. Specific data types এ inefficient (graph, document, time-series)
  4. Write throughput limit (single master bottleneck)

Solution: NoSQL
  - Schema-less (flexible)
  - Horizontally scalable by design
  - Specific use cases এ optimized
  - High write throughput
```

### NoSQL এর ধরন

```
┌──────────────────┬───────────────────┬──────────────────────────────┐
│ Type             │ Examples          │ Best For                     │
├──────────────────┼───────────────────┼──────────────────────────────┤
│ Key-Value        │ Redis, DynamoDB   │ Cache, Session, Config       │
│ Document         │ MongoDB, CouchDB  │ Flexible schema, catalog     │
│ Column-Family    │ Cassandra, HBase  │ Time-series, wide-column     │
│ Graph            │ Neo4j, Amazon     │ Relationships, social network│
│                  │ Neptune           │                              │
│ Time-Series      │ InfluxDB, TimescaleDB│ Metrics, IoT, monitoring │
│ Search           │ Elasticsearch     │ Full-text search             │
└──────────────────┴───────────────────┴──────────────────────────────┘
```

---

## 12.2 CAP Theorem (ক্যাপ থিওরেম)

### কী এবং কেন গুরুত্বপূর্ণ?

```
CAP Theorem (Eric Brewer, 2000):
Distributed system এ একসাথে তিনটা guarantee দেওয়া সম্ভব না।
যেকোনো দুটো বেছে নিতে হবে।

C — Consistency:
  সব nodes এ same time এ same data দেখায়।
  একটা write হলে সব nodes তাৎক্ষণিক updated।

A — Availability:
  System সবসময় response দেবে (error দিলেও)।
  কোনো request unanswered থাকবে না।

P — Partition Tolerance:
  Network partition (nodes এর মধ্যে communication failure) 
  হলেও system চলতে থাকবে।

Reality: Network partition সবসময় হতে পারে।
তাই P তে সবসময় compromise করা যায় না।
মূলত choice: CP নাকি AP।
```

### CAP Visual

```
         Consistency (C)
              △
             / \
            /   \
           /     \
          /  CA   \
         / (SQL DB)\
        /___________\
       /      |      \
      /   CP  |  AP   \
     /        |        \
    ◁─────────┼─────────▷
Partition      |      Availability
Tolerance(P)   |         (A)

CP Systems (Consistent + Partition Tolerant):
  PostgreSQL, MongoDB (with majority writes)
  "Network failure হলে unavailable হবো কিন্তু stale data দেবো না"

AP Systems (Available + Partition Tolerant):
  Cassandra, DynamoDB, CouchDB
  "Network failure তেও respond করবো কিন্তু data stale হতে পারে"

CA Systems (Consistent + Available):
  Single-node SQL databases
  "Network partition impossible — single server"
```

### PACELC Extension

```
CAP এর limitation: Partition rare, কিন্তু Latency সবসময় আছে।

PACELC:
  If Partition → C or A trade-off
  Else → Latency or Consistency trade-off

Example:
  DynamoDB: PA/EL (Available during partition, Low latency normally)
  PostgreSQL: PC/EC (Consistent during partition, Consistent normally)
```

---

## 12.3 MongoDB (ডকুমেন্ট ডাটাবেজ)

### কী এবং কেন?

```
MongoDB:
  - Document store (JSON-like BSON)
  - Schema-less (flexible)
  - Horizontal scaling built-in (sharding)
  - Rich query language
  - Good for: catalogs, CMS, user profiles, logs

Document Example:
{
  "_id": ObjectId("..."),
  "name": "Samsung Galaxy S24",
  "category": "smartphone",
  "price": 1200,
  "specs": {
    "ram": "12GB",
    "storage": "256GB",
    "camera": "200MP"
  },
  "tags": ["samsung", "android", "flagship"],
  "variants": [
    {"color": "Black", "stock": 50},
    {"color": "White", "stock": 30}
  ],
  "reviews": [
    {"user": "Rahim", "rating": 5, "comment": "Great phone!"}
  ]
}
```

### MongoDB কখন ব্যবহার করবে

```
✅ MongoDB ভালো:
  - Flexible/evolving schema (product catalog with varied attributes)
  - Nested/embedded data (blog post with comments)
  - Rapid prototyping
  - Document-oriented data (CMS, e-commerce catalog)
  - Horizontal scaling needed
  - Geo-spatial queries

❌ MongoDB এড়াও:
  - Complex JOINs দরকার
  - ACID transactions critical (banking, payment)
  - Strong consistency needed
  - Relational data (normalized)
  - Reporting/Analytics with complex aggregations
```

### MongoDB vs PostgreSQL JSONB

```
PostgreSQL JSONB:
  ✅ Full ACID transactions
  ✅ SQL + JSON combined
  ✅ Better JOIN support
  ✅ Mature tooling
  ❌ Horizontal scaling complex

MongoDB:
  ✅ Native document model
  ✅ Built-in sharding
  ✅ Rich aggregation pipeline
  ✅ Better for pure document workloads
  ❌ No true multi-document ACID (until v4.0, limited)

Senior Tip: যদি তুমি PostgreSQL ব্যবহার করছো,
JSONB দিয়ে flexible schema handle করো।
MongoDB এ migrate করার আগে চিন্তা করো।
```

### Django + MongoDB (Djongo/MongoEngine)

```python
# pip install mongoengine

from mongoengine import (
    Document, StringField, FloatField,
    ListField, DictField, connect
)

connect('ecommerce_db', host='mongodb://localhost:27017/')

class Product(Document):
    name       = StringField(required=True, max_length=200)
    category   = StringField(required=True)
    price      = FloatField(required=True)
    specs      = DictField()       # flexible attributes
    tags       = ListField(StringField())
    
    meta = {
        'collection': 'products',
        'indexes': ['name', 'category', ('price', 1)]
    }

# Create
product = Product(
    name='Samsung Galaxy S24',
    category='smartphone',
    price=1200,
    specs={'ram': '12GB', 'storage': '256GB'},
    tags=['samsung', 'android']
)
product.save()

# Query
phones = Product.objects(category='smartphone', price__lte=1500)
samsung = Product.objects(tags='samsung').order_by('-price')

# Aggregation
pipeline = [
    {'$match': {'category': 'smartphone'}},
    {'$group': {'_id': '$category', 'avg_price': {'$avg': '$price'}}},
    {'$sort': {'avg_price': -1}}
]
result = Product.objects.aggregate(pipeline)
```

---

## 12.4 Redis (কী-ভ্যালু ডাটাবেজ)

### Redis Data Structures

```
Redis শুধু key-value নয় — rich data structures:

String:    "counter" → "1234"
Hash:      "user:1" → {name: "Rahim", email: "r@x.com"}
List:      "queue" → [job1, job2, job3]  (FIFO/LIFO)
Set:       "online_users" → {user1, user2, user3}  (unique)
Sorted Set:"leaderboard" → {user1: 100, user2: 95, user3: 90}
Bitmap:    "active_days" → bit array
HyperLogLog: approximate unique count
Streams:   event log (like Kafka-lite)
Geo:       location data
```

### Redis Use Cases

```python
import redis
r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# 1. Caching
r.setex('product:1', 3600, json.dumps(product_data))
data = r.get('product:1')

# 2. Session Storage
r.hset(f'session:{session_id}', mapping={
    'user_id': '123',
    'created_at': str(datetime.now()),
    'role': 'admin'
})
r.expire(f'session:{session_id}', 86400)  # 24hr

# 3. Rate Limiting
def is_rate_limited(user_id, limit=100, window=60):
    key = f'ratelimit:{user_id}:{int(time.time() // window)}'
    count = r.incr(key)
    if count == 1:
        r.expire(key, window)
    return count > limit

# 4. Real-time Counter
r.incr('page_views:homepage')
r.incr(f'product_views:{product_id}')

# 5. Pub/Sub (real-time notifications)
# Publisher
r.publish('order_updates', json.dumps({'order_id': 1, 'status': 'shipped'}))

# Subscriber
pubsub = r.pubsub()
pubsub.subscribe('order_updates')
for message in pubsub.listen():
    if message['type'] == 'message':
        data = json.loads(message['data'])
        print(f"Order {data['order_id']} is {data['status']}")

# 6. Leaderboard (Sorted Set)
r.zadd('game_scores', {'user:1': 1500, 'user:2': 2300, 'user:3': 1800})
top_10 = r.zrevrange('game_scores', 0, 9, withscores=True)
rank = r.zrevrank('game_scores', 'user:1')

# 7. Distributed Queue
r.lpush('email_queue', json.dumps({'to': 'r@x.com', 'subject': 'Welcome'}))
job = r.brpop('email_queue', timeout=30)  # blocking pop

# 8. Bloom Filter (probabilistic set membership)
# pip install redis-py-bloom
from redisbloom.client import Client
rb = Client()
rb.bfCreate('email_seen', 0.001, 1000000)  # 0.1% false positive
rb.bfAdd('email_seen', 'user@example.com')
rb.bfExists('email_seen', 'user@example.com')  # True/False
```

### Redis Persistence

```
Redis Persistence Options:

1. RDB (Redis Database Snapshot):
   - Periodic snapshot to disk
   - Fast recovery
   - Some data loss possible (since last snapshot)
   
2. AOF (Append Only File):
   - Every write operation logged
   - More durable
   - Larger file, slower recovery
   
3. No persistence:
   - Pure cache (data loss on restart OK)
   - Fastest

# redis.conf
save 900 1      # 900 seconds এ 1 change হলে save
save 300 10     # 300 seconds এ 10 changes হলে save
appendonly yes  # AOF enable
appendfsync everysec  # Every second sync
```

---

## 12.5 Cassandra (কলাম-ফ্যামিলি ডাটাবেজ)

### কী এবং কেন?

```
Apache Cassandra:
  - Wide-column store
  - Masterless architecture (no single point of failure)
  - Linear horizontal scaling
  - High write throughput
  - Eventual consistency (tunable)
  - Best for: time-series, IoT, messaging, activity feeds

Facebook এ Cassandra তৈরি হয়েছিল।
Netflix, Twitter, Instagram ব্যবহার করে।

Data Model:
  Keyspace → Table → Partition Key + Clustering Key + Columns

Example — Uber ride history:
CREATE TABLE rides_by_driver (
    driver_id  UUID,
    ride_date  DATE,
    ride_id    UUID,
    pickup     TEXT,
    dropoff    TEXT,
    fare       DECIMAL,
    PRIMARY KEY ((driver_id, ride_date), ride_id)
    -- (driver_id, ride_date) = Partition Key
    -- ride_id = Clustering Key
);
-- driver + date দিয়ে query = very fast (same partition)
-- Cross-partition query = slow/expensive
```

### Cassandra কখন ব্যবহার করবে

```
✅ Cassandra ভালো:
  - Write-heavy workloads (millions writes/second)
  - Time-series data (IoT sensor data, logs, metrics)
  - Always-on, no downtime required
  - Geographically distributed
  - Linear scalability needed
  - Messaging/activity feeds

❌ Cassandra এড়াও:
  - Complex queries (JOINs, aggregations)
  - ACID transactions needed
  - Small datasets
  - Ad-hoc queries
  - Strong consistency required
```

### Cassandra vs PostgreSQL

```
┌──────────────┬────────────────────┬────────────────────┐
│ Feature      │ Cassandra          │ PostgreSQL         │
├──────────────┼────────────────────┼────────────────────┤
│ Write speed  │ Extremely fast     │ Good               │
│ Read speed   │ Fast (known key)   │ Very fast          │
│ Consistency  │ Eventual (tunable) │ Strong (ACID)      │
│ Schema       │ Flexible           │ Strict             │
│ JOINs        │ Not supported      │ Excellent          │
│ Aggregation  │ Limited            │ Excellent          │
│ Scaling      │ Linear horizontal  │ Complex            │
│ Ops          │ Complex            │ Simpler            │
└──────────────┴────────────────────┴────────────────────┘
```

---

## 12.6 Graph Database — Neo4j

### কী এবং কেন?

```
Graph Database:
  - Data = Nodes (entities) + Edges (relationships)
  - Relationship-first data model
  - Traversal queries efficient
  
SQL এ Friend-of-Friend query:
  -- 3 levels deep friendship
  SELECT DISTINCT u3.*
  FROM users u1
  JOIN friendships f1 ON f1.user1_id = u1.id
  JOIN users u2 ON u2.id = f1.user2_id
  JOIN friendships f2 ON f2.user1_id = u2.id
  JOIN users u3 ON u3.id = f2.user2_id
  WHERE u1.id = 1;
  -- N levels deep = N JOINs = exponentially slow!

Neo4j Cypher query:
  MATCH (u:User {id: 1})-[:FRIEND*1..3]-(friend)
  RETURN DISTINCT friend
  -- Depth N = same performance!
```

### Graph DB Use Cases

```
✅ Graph DB ভালো:
  - Social networks (friend suggestions)
  - Recommendation engines ("users who bought X also bought Y")
  - Fraud detection (suspicious transaction patterns)
  - Knowledge graphs
  - Network/IT infrastructure mapping
  - Identity and access management

Real Examples:
  - LinkedIn: connection graph
  - Facebook: social graph
  - Amazon: product recommendations
  - PayPal: fraud detection graph
```

### Django + Neo4j

```python
# pip install neomodel

from neomodel import (
    StructuredNode, StringProperty,
    RelationshipTo, RelationshipFrom, config
)

config.DATABASE_URL = 'bolt://neo4j:password@localhost:7687'

class User(StructuredNode):
    uid      = StringProperty(unique_index=True)
    name     = StringProperty()
    friends  = RelationshipTo('User', 'FRIEND')
    follows  = RelationshipTo('User', 'FOLLOWS')

# Create
rahim = User(uid='1', name='Rahim').save()
karim = User(uid='2', name='Karim').save()
rahim.friends.connect(karim)

# Query — friends of friends
friends_of_friends = db.cypher_query("""
    MATCH (u:User {uid: $uid})-[:FRIEND*2]-(fof)
    WHERE NOT (u)-[:FRIEND]-(fof) AND fof.uid <> $uid
    RETURN DISTINCT fof
""", {'uid': '1'})
```

---

## 12.7 Time-Series Database

### কী এবং কেন?

```
Time-Series Data:
  - Timestamp + metric value
  - High frequency writes (millions/second)
  - Range queries by time
  - Aggregation over time windows

Examples:
  - Server CPU/memory metrics
  - IoT sensor data
  - Stock prices
  - Application performance monitoring (APM)
  - User activity tracking
```

### TimescaleDB (PostgreSQL Extension)

```sql
-- TimescaleDB = PostgreSQL + Time-series optimization
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- Regular PostgreSQL table
CREATE TABLE metrics (
    time        TIMESTAMPTZ NOT NULL,
    device_id   UUID NOT NULL,
    temperature FLOAT,
    humidity    FLOAT
);

-- Convert to hypertable (automatic time-based partitioning)
SELECT create_hypertable('metrics', 'time', chunk_time_interval => INTERVAL '1 day');

-- Insert — same as PostgreSQL
INSERT INTO metrics (time, device_id, temperature)
VALUES (NOW(), 'device-1', 25.5);

-- Time-series queries
SELECT
    time_bucket('1 hour', time) AS hour,
    AVG(temperature)            AS avg_temp,
    MAX(temperature)            AS max_temp
FROM metrics
WHERE device_id = 'device-1'
  AND time > NOW() - INTERVAL '7 days'
GROUP BY hour
ORDER BY hour;

-- Continuous aggregates (materialized, auto-refresh)
CREATE MATERIALIZED VIEW hourly_metrics
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS hour,
    device_id,
    AVG(temperature) AS avg_temp
FROM metrics
GROUP BY hour, device_id;

-- Compression (old data)
SELECT add_compression_policy('metrics', INTERVAL '7 days');
-- 7 days এর পুরনো data automatically compress হবে → 90% storage savings
```

---

## 12.8 Elasticsearch (সার্চ ইঞ্জিন)

### কী এবং কেন?

```
Elasticsearch:
  - Distributed search and analytics engine
  - Full-text search optimized
  - Real-time indexing
  - Horizontal scaling built-in
  - RESTful API

Use Cases:
  - Product search (Amazon, Daraz)
  - Log analysis (ELK Stack)
  - Application search
  - Autocomplete
  - Geo search
```

### Django + Elasticsearch

```python
# pip install elasticsearch django-elasticsearch-dsl

# documents.py
from django_elasticsearch_dsl import Document, fields
from django_elasticsearch_dsl.registries import registry

@registry.register_document
class ProductDocument(Document):
    category_name = fields.TextField()
    
    class Index:
        name     = 'products'
        settings = {
            'number_of_shards': 2,
            'number_of_replicas': 1
        }
    
    class Django:
        model  = Product
        fields = ['id', 'name', 'description', 'price', 'stock']
    
    def prepare_category_name(self, instance):
        return instance.category.name if instance.category else ''

# Search
from elasticsearch_dsl import Q as ESQ

def search_products(query, category=None, min_price=None, max_price=None):
    search = ProductDocument.search()
    
    # Full-text search
    search = search.query(
        'multi_match',
        query=query,
        fields=['name^3', 'description', 'category_name^2'],
        fuzziness='AUTO'  # Typo tolerance
    )
    
    # Filters
    if category:
        search = search.filter('term', category_name=category)
    if min_price:
        search = search.filter('range', price={'gte': min_price})
    if max_price:
        search = search.filter('range', price={'lte': max_price})
    
    # In-stock only
    search = search.filter('range', stock={'gt': 0})
    
    # Sort by relevance, then price
    search = search.sort('_score', 'price')
    
    return search[:20].execute()
```

---

## 12.9 SQL vs NoSQL — কখন কোনটা?

### Decision Framework

```
SQL ব্যবহার করো যখন:
  ✅ Data relational এবং structured
  ✅ ACID transactions critical (banking, payment, inventory)
  ✅ Complex queries, JOINs, aggregations
  ✅ Data integrity constraints দরকার
  ✅ Schema stable (বেশি পরিবর্তন হবে না)
  ✅ Reporting/Analytics
  ✅ Team SQL জানে

NoSQL ব্যবহার করো যখন:
  ✅ Schema flexible বা unknown
  ✅ Massive scale horizontal sharding দরকার
  ✅ Specific data model (graph, time-series, document)
  ✅ High write throughput (Cassandra)
  ✅ Simple access patterns (key-value lookup)
  ✅ Eventual consistency acceptable
```

### Real-world Database Choices

```
Company / Use Case → Database Choice:

Facebook:
  User data, posts       → MySQL (sharded)
  Messages               → HBase (Cassandra-like)
  Social graph           → TAO (custom graph DB over MySQL)
  Cache                  → Memcached/Redis

Netflix:
  User profiles          → MySQL + Cassandra
  Viewing history        → Cassandra
  Content metadata       → MySQL
  Real-time recommendations → Apache Flink + Cassandra
  Billing                → MySQL

Uber:
  Trip data              → PostgreSQL → MySQL (sharded)
  Real-time location     → Redis
  Analytics              → Hadoop + Hive
  Dispatch               → Redis

Instagram:
  Posts, users           → PostgreSQL (sharded)
  Feed                   → Redis + Cassandra
  Media metadata         → Cassandra
  Search                 → Elasticsearch

Twitter:
  Tweets                 → MySQL → Manhattan (custom)
  Timeline               → Redis (cache)
  Graph (followers)      → FlockDB (custom graph)
  Search                 → Elasticsearch

E-commerce (Daraz/Amazon style):
  Products catalog       → PostgreSQL + Elasticsearch
  Orders/payments        → PostgreSQL (ACID critical)
  Cart                   → Redis
  Sessions               → Redis
  Search                 → Elasticsearch
  Analytics              → Redshift/BigQuery
  Recommendations        → Graph DB / ML
```

---

## 12.10 Polyglot Persistence

### কী এবং কেন?

```
Polyglot Persistence:
  একটা application এ multiple databases ব্যবহার করো
  প্রতিটা database তার নিজের strength এ ব্যবহার করো

Example — E-commerce:
  PostgreSQL → Orders, payments, users (ACID, relational)
  Redis      → Cart, sessions, cache, rate limiting
  Elasticsearch → Product search, autocomplete
  MongoDB    → Product catalog (flexible attributes)
  Cassandra  → User activity, event logs
  S3         → Media files, backups

Django এ এটা implement করা:
  - Multiple DATABASES settings
  - Database routers
  - Direct DB clients (redis-py, pymongo, elasticsearch-py)
```

### Django Polyglot Setup

```python
# settings.py
DATABASES = {
    'default': {  # PostgreSQL — primary relational data
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'myapp_db',
        'HOST': 'postgres.internal',
    },
    'analytics': {  # Read replica for analytics
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'myapp_db',
        'HOST': 'postgres-replica.internal',
    },
}

# Redis — via django-redis
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://redis.internal:6379/0',
    }
}

# MongoDB — via mongoengine (separate connection)
import mongoengine
mongoengine.connect('product_catalog', host='mongodb://mongo.internal:27017/')

# Elasticsearch — via elasticsearch-py
from elasticsearch import Elasticsearch
es = Elasticsearch(['elasticsearch.internal:9200'])
```

---

## 12.11 Eventual Consistency (ইভেন্চুয়াল কনসিস্টেন্সি)

### কী?

```
Strong Consistency:
  Write হলে সব nodes তাৎক্ষণিক same data দেখে।
  SQL databases, ZooKeeper

Eventual Consistency:
  Write হলে eventually সব nodes same data দেখবে।
  কিন্তু তাৎক্ষণিক নয়।
  Cassandra, DynamoDB, DNS

Example:
  Facebook এ post করলে:
  - তোমার নিজের feed এ তাৎক্ষণিক দেখা যায়
  - বন্ধুর feed এ কিছুক্ষণ পরে দেখা যায়
  - এটা eventual consistency — acceptable!

Banking এ:
  Account balance wrong দেখানো → Not acceptable!
  Strong consistency দরকার।
```

### BASE vs ACID

```
ACID (SQL):
  Atomicity, Consistency, Isolation, Durability
  Strong consistency guarantee

BASE (NoSQL):
  Basically Available  → সবসময় respond করে
  Soft state           → Data always in flux
  Eventually consistent → Eventually সব nodes agree করে

কোনটা ভালো নয় — use case এর উপর নির্ভর করে।
```

---

## 12.12 Common Mistakes

### ভুল ১: NoSQL = Silver Bullet ভাবা

```
❌ "NoSQL faster, তাই সব MongoDB তে নিয়ে যাই"
→ Complex reporting nightmare
→ Data consistency problems
→ Transaction support নেই (v4 এর আগে)

✅ Use case analyze করো। PostgreSQL অনেক দূর নিয়ে যেতে পারে।
```

### ভুল ২: MongoDB তে Unbounded Arrays

```javascript
// ❌ একটা document এ হাজার হাজার nested items
{
  "_id": "post:1",
  "comments": [
    // 10,000 comments এখানে!
    // Document size limit 16MB
    // Query slow হবে
  ]
}

// ✅ Comments আলাদা collection এ
{
  "_id": ObjectId(),
  "post_id": "post:1",
  "user": "Rahim",
  "text": "Great post!"
}
```

### ভুল ৩: Cassandra তে Wrong Partition Key

```sql
-- ❌ Low cardinality partition key
CREATE TABLE events (
    event_type TEXT,   -- শুধু 5 types! Hot partition!
    event_id   UUID,
    data       TEXT,
    PRIMARY KEY (event_type, event_id)
);
-- সব 'click' events একটা partition এ → Hot partition!

-- ✅ High cardinality partition key
CREATE TABLE events (
    user_id    UUID,
    event_date DATE,
    event_id   UUID,
    event_type TEXT,
    data       TEXT,
    PRIMARY KEY ((user_id, event_date), event_id)
);
-- user_id + date দিয়ে partition — even distribution
```

### ভুল ৪: Redis কে Primary Database বানানো

```
❌ সব data Redis এ store করা
→ Memory expensive
→ Persistence configure না করলে data loss
→ Complex queries possible নয়

✅ Redis = Cache + Session + Queue + Real-time counters
PostgreSQL = Primary data store
```

---

## 12.13 Production Best Practices

```
1.  PostgreSQL দিয়ে শুরু করো — যথেষ্ট powerful
2.  NoSQL যোগ করো specific need এর জন্য (polyglot)
3.  Redis সবসময় cache/session/queue এর জন্য — excellent
4.  Search দরকার হলে Elasticsearch যোগ করো
5.  Time-series → TimescaleDB (PostgreSQL extension, easy migration)
6.  Graph data → Neo4j অথবা AWS Neptune
7.  CAP theorem মাথায় রেখে database choose করো
8.  Eventual consistency acceptable কিনা validate করো
9.  NoSQL schema design আগে ভাবো (query-first design)
10. Operational complexity consider করো (MongoDB ops != PostgreSQL ops)
```

---

## 12.14 Senior Engineer Insights

> **War Story — Discord:**
> Discord শুরু করেছিল MongoDB তে messages store করে।
> Growth এর সাথে সাথে MongoDB slow হয়ে যায়।
> Cassandra তে migrate করে → billions of messages, millisecond reads।
> Lesson: Messaging/time-series → Cassandra excellent choice।

> **Airbnb এর Approach:**
> Airbnb শুরু করেছিল single MySQL তে।
> Gradually added: Elasticsearch (search), Redis (cache), Druid (analytics)।
> Core data এখনো relational database এ।
> Lesson: Start simple, add databases as specific needs arise।

> **Design Decision:**
> "Choose boring technology." — Dan McKinley (Etsy)
> PostgreSQL boring, কিন্তু extremely powerful।
> Team জানে, tooling mature, debugging easy।
> NoSQL এ migrate করার আগে PostgreSQL এর limit reach করো।

---

## 12.15 Interview Questions

### Beginner Level

**Q1: SQL এবং NoSQL এর মূল পার্থক্য কী?**
> SQL: Fixed schema, ACID transactions, relational data, JOINs সমর্থন করে, vertical scaling সহজ।
> NoSQL: Flexible schema, eventual consistency (mostly), horizontal scaling সহজ, specific data models (document/graph/column)।

**Q2: CAP theorem কী?**
> Distributed system এ Consistency, Availability, Partition Tolerance — তিনটা একসাথে guarantee দেওয়া সম্ভব না। Network partition সবসময় হতে পারে, তাই P বাদ দেওয়া যায় না। Choice হলো CP (consistent but might be unavailable) নাকি AP (available but might be inconsistent)।

**Q3: Redis কেন এত popular?**
> In-memory → microsecond latency। Rich data structures (string, hash, list, set, sorted set)। Persistence options। Pub/Sub। Atomic operations। Perfect for cache, session, rate limiting, leaderboard, queue।

---

### Intermediate Level

**Q4: MongoDB এবং PostgreSQL এ কখন কোনটা choose করবে?**
> PostgreSQL: ACID transactions critical, complex JOINs, stable schema, reporting/analytics।
> MongoDB: Flexible/evolving schema, document-oriented data, horizontal scaling needed, rapid prototyping। তবে PostgreSQL JSONB দিয়ে অনেক MongoDB use case handle হয়।

**Q5: Cassandra কোন use cases এ best?**
> High write throughput (IoT, logs, metrics), time-series data, always-on availability needed, geographically distributed, linear scaling। Write-heavy এবং known access patterns এ excellent।

**Q6: Eventual consistency কখন acceptable?**
> Social media feeds, product view counts, analytics dashboards, recommendation systems — এখানে কিছুক্ষণ stale data acceptable। Banking, payment, inventory — এখানে strong consistency mandatory।

---

### Senior / FAANG Level

**Q7: Instagram এর feed system design করো — কোন database ব্যবহার করবে?**

```
Feed System Requirements:
  - User follow করলে তাদের posts দেখাবে
  - Real-time (near real-time)
  - 500M+ users
  - Each user 200+ followers

Approach 1 — Pull Model (Fan-out on read):
  - Post table (PostgreSQL): post_id, author_id, content, created_at
  - Follow table (PostgreSQL): follower_id, following_id
  - Feed query: SELECT posts WHERE author_id IN (following_ids)
  - সমস্যা: 200 following × 10M followers = expensive query

Approach 2 — Push Model (Fan-out on write):
  - Post হলে সব followers এর feed table এ insert করো
  - feed_items table (Cassandra): user_id, post_id, created_at
  - Read: SELECT FROM feed_items WHERE user_id = X LIMIT 20
  - Very fast reads!
  - সমস্যা: Celebrity problem (10M followers = 10M writes per post)

Instagram এর Actual Approach:
  - Regular users: Fan-out on write (Cassandra)
  - Celebrities (>1M followers): Fan-out on read (merge at read time)
  - Cache: Redis (hot feeds)
  - Media: S3 + CDN

Database Stack:
  PostgreSQL → User data, relationships
  Cassandra  → Feed items, activity
  Redis      → Hot feed cache, sessions
  S3         → Photos, videos
```

**Q8: তুমি কীভাবে SQL থেকে NoSQL migration করবে?**

```
Migration Strategy:

1. Identify the use case:
   কী data NoSQL তে যাবে? কেন?
   Access patterns কী?

2. Dual write:
   কিছুদিন SQL এবং NoSQL দুটোতেই write করো
   Verify consistency

3. Read migration:
   Gradually traffic NoSQL এ shift করো
   Monitor performance, errors

4. Cutover:
   SQL কে read-only করো
   সব traffic NoSQL তে
   
5. Backfill:
   Historical data SQL → NoSQL migrate করো

6. Decommission:
   SQL table drop করো (backup রাখো)

Rollback plan সবসময় থাকতে হবে।
Feature flag দিয়ে gradually migrate করো।
```

---

## 12.16 Hands-on Exercises

### Exercise 1
```python
# Redis দিয়ে implement করো:
# 1. Product view counter (atomic increment)
# 2. Top 10 viewed products (sorted set)
# 3. User session store (hash)
# 4. Rate limiter (5 requests/minute per user)
```

### Exercise 2
```python
# MongoDB দিয়ে:
# 1. Product catalog তৈরি করো (flexible attributes)
# 2. Category-wise aggregation করো
# 3. Full-text search implement করো
# 4. GIN index compare করো PostgreSQL JSONB এর সাথে
```

### Exercise 3
```
CAP theorem practice:
1. PostgreSQL এ: network partition simulate করো
2. Redis Cluster এ: node failure simulate করো
3. Observe: কোনটা CP, কোনটা AP behave করে
```

### Mini Project
```
Polyglot E-commerce:
1. PostgreSQL → Orders, users, payments
2. Redis → Cart, sessions, product cache
3. Elasticsearch → Product search
4. MongoDB → Product catalog (flexible specs)

Django ORM দিয়ে সব integrate করো।
Search করলে Elasticsearch query যাবে।
Product detail → Redis cache → PostgreSQL fallback।
Order create → PostgreSQL (ACID)।
```

---

## 12.17 Module Summary & Cheat Sheet

```
NOSQL CHEAT SHEET
==================

DATABASE TYPES:
  Key-Value    → Redis, DynamoDB  (cache, session, queue)
  Document     → MongoDB          (flexible schema, catalog)
  Column-Family→ Cassandra        (time-series, high write)
  Graph        → Neo4j            (social, recommendations)
  Search       → Elasticsearch    (full-text, analytics)
  Time-Series  → TimescaleDB      (metrics, IoT)

CAP THEOREM:
  C = Consistency (all nodes same data)
  A = Availability (always responds)
  P = Partition Tolerance (survives network split)
  Can only guarantee 2 of 3.
  P usually required → Choose CP or AP.

  CP: PostgreSQL, MongoDB (majority writes)
  AP: Cassandra, DynamoDB, CouchDB

WHEN SQL:
  ✅ ACID transactions (banking, payment)
  ✅ Complex JOINs and aggregations
  ✅ Stable, relational schema
  ✅ Strong consistency required

WHEN NOSQL:
  ✅ Flexible/evolving schema
  ✅ Massive horizontal scale
  ✅ Specific data model (graph, time-series)
  ✅ High write throughput
  ✅ Simple access patterns
  ✅ Eventual consistency OK

REDIS DATA STRUCTURES:
  String    → Cache, counters
  Hash      → Object storage (session, user)
  List      → Queue (FIFO/LIFO)
  Set       → Unique members (online users)
  Sorted Set→ Leaderboard, ranked data
  Stream    → Event log

COMMON PATTERNS:
  Cache-aside    → App manages cache + DB
  Write-through  → Write cache + DB together
  Pub/Sub        → Real-time messaging
  Distributed Lock → Redis SETNX
  Rate Limiting  → Redis INCR + EXPIRE

POLYGLOT PERSISTENCE:
  PostgreSQL  → Core relational data
  Redis       → Cache, session, queue
  Elasticsearch→ Search
  MongoDB     → Flexible document data
  Cassandra   → Time-series, activity feeds
  S3          → Files, backups

MISTAKES TO AVOID:
  ❌ NoSQL = always faster (not true)
  ❌ MongoDB unbounded arrays
  ❌ Cassandra low-cardinality partition key
  ❌ Redis as primary database
  ❌ Premature NoSQL migration
  ✅ Start with PostgreSQL, add NoSQL as needed
```
