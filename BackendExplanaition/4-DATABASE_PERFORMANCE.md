# LEVEL 4 — Database Performance

This is the level where you stop writing code that "works" and start writing code that **survives real traffic**. Most backend bugs at this stage aren't logic bugs — they're *performance* bugs that only show up once you have real users, real data volume, and real concurrency. This guide covers the N+1 problem in depth, plus the topics that build directly on top of it.

---

## 1. The N+1 Query Problem

This is the single most common performance bug in backend code — and the most important one to understand deeply.

### The Setup

```python
users = User.objects.all()

for user in users:
    print(user.orders.all())
```

This code *looks* completely innocent. It reads like plain English: "for every user, print their orders." But look at what it actually does at the database level.

### What Actually Happens

```
1 query  →  SELECT * FROM users;                          (get all users)

Then, for EACH user returned, a SEPARATE query fires:

+N queries →  SELECT * FROM orders WHERE user_id = 1;     (user 1's orders)
              SELECT * FROM orders WHERE user_id = 2;     (user 2's orders)
              SELECT * FROM orders WHERE user_id = 3;     (user 3's orders)
              ...
              SELECT * FROM orders WHERE user_id = N;     (user N's orders)

Total queries = 1 + N
```

```
        1 query                          N queries (one per user!)
   ┌─────────────┐              ┌───────────────────────────────┐
   │ SELECT       │              │ SELECT orders WHERE user_id=1 │
   │ * FROM users │   ───────►   │ SELECT orders WHERE user_id=2 │
   └─────────────┘              │ SELECT orders WHERE user_id=3 │
                                 │ ...                              │
                                 │ SELECT orders WHERE user_id=N │
                                 └───────────────────────────────┘
```

### Why It's Dangerous

```
10 users    → 11 queries    (barely noticeable)
100 users   → 101 queries   (starting to hurt)
10,000 users → 10,001 queries  (page takes seconds, DB gets hammered)
```

Each query has network round-trip overhead (client ↔ database), even if each individual query is fast. **1 query that returns 10,000 rows is dramatically faster than 10,000 queries that return 1 row each** — the round-trip cost dominates, not the actual data lookup.

```
Round-trip cost per query ≈ 2–5ms (network + parsing + planning)

1 query:      1 × 3ms  =  3ms
10,000 queries: 10,000 × 3ms = 30,000ms = 30 seconds  😱
```

### The Fix — Fetch Related Data in Fewer Queries

Django ORM gives you two tools for this: `select_related()` and `prefetch_related()`. They solve the **same problem** but work completely differently under the hood — knowing *when* to use which is the real skill.

---

## 2. `select_related()` — for Forward Relationships (ForeignKey / OneToOne)

Uses a **SQL JOIN** to fetch the related object **in the same query**.

```python
orders = Order.objects.select_related('user').all()

for order in orders:
    print(order.user.name)   # no extra query — already loaded
```

### SQL Equivalent

```sql
SELECT orders.*, users.*
FROM orders
JOIN users ON orders.user_id = users.id;
```

```
                    1 query, using JOIN
   ┌───────────────────────────────────────────────┐
   │ orders.id | orders.total | users.id | users.name │
   ├───────────────────────────────────────────────┤
   │    1      |    $50       |    2     |  Alice    │
   │    2      |    $20       |    1     |  Bob      │
   │    3      |    $80       |    2     |  Alice    │
   └───────────────────────────────────────────────┘
   All the data arrives already joined — no follow-up query needed.
```

**When to use it:** only works for **ForeignKey** and **OneToOne** relationships — i.e. when the "many" side is looking up the "one" side (each order has exactly one user).

```
Order  ──── user_id ────►  User        ← use select_related()
  (many)                    (one)
```

---

## 3. `prefetch_related()` — for Reverse / Many-to-Many Relationships

Runs a **second separate query** (not a JOIN), then **stitches the results together in Python/application memory**.

```python
users = User.objects.prefetch_related('orders').all()

for user in users:
    print(user.orders.all())   # no extra query — already loaded
```

### SQL Equivalent

```sql
-- Query 1
SELECT * FROM users;

-- Query 2 (fetches ALL matching orders in ONE query, not one per user)
SELECT * FROM orders WHERE user_id IN (1, 2, 3, ..., N);
```

```
Query 1: SELECT * FROM users                Query 2: SELECT * FROM orders
┌────┬───────┐                              WHERE user_id IN (1,2,3)
│ id │ name  │                              ┌────┬─────────┬────────┐
├────┼───────┤                              │ id │ user_id │ total  │
│ 1  │ Alice │                              ├────┼─────────┼────────┤
│ 2  │ Bob   │                              │ 1  │    1    │ $50    │
│ 3  │ Charlie│                             │ 2  │    2    │ $20    │
└────┴───────┘                              │ 3  │    1    │ $80    │
                                             └────┴─────────┴────────┘
                     │
                     ▼  Django stitches these together in memory
        Alice.orders  = [order 1, order 3]
        Bob.orders    = [order 2]
        Charlie.orders = []

Total queries = 2, regardless of how many users there are.
```

**Why not JOIN here?** Because a JOIN between `users` and `orders` would **duplicate the user row for every order they have** (Alice would appear twice if she has 2 orders) — this row explosion gets worse the more nested relationships you prefetch, so Django fetches separately and joins in memory instead.

```
Order  ◄──── user_id ────  User        ← use prefetch_related()
 (many)                    (one, but looking at its MANY children)

Also used for Many-to-Many:
Student ◄──── enrollments ────► Course   ← use prefetch_related()
```

---

## 4. Side-by-Side Comparison

| | `select_related()` | `prefetch_related()` |
|---|---|---|
| Relationship type | ForeignKey, OneToOne | Reverse FK, ManyToMany |
| SQL technique | JOIN | Separate query + `IN (...)` |
| Number of queries | 1 (combined) | 2+ (one per relation prefetched) |
| Where joining happens | Database | Application (Python) |
| Risk of row duplication | Yes, if joining "many" side | No |

```
select_related()                    prefetch_related()
┌─────────────────┐                 ┌─────────────────┐  ┌─────────────────┐
│  1 query (JOIN)   │                │  Query 1: parents │  │ Query 2: children│
└─────────────────┘                 └─────────────────┘  └─────────────────┘
        │                                     │                    │
        ▼                                     └────────┬───────────┘
   Fully joined result                                   ▼
                                          Stitched together in memory
```

### Chaining Both Together

```python
orders = (
    Order.objects
    .select_related('user')           # JOIN: order → user
    .prefetch_related('items')        # Separate query: order → items (many)
)
```

```
Query 1 (JOIN): orders + users combined
Query 2: SELECT * FROM order_items WHERE order_id IN (...)

Total: 2 queries, no matter how many orders or items exist.
```

---

## Additional Topics to Build Real Database Performance Skill

The N+1 problem is the entry point — here's what comes next, roughly in the order you'll run into them.

### 1. Detecting N+1 Problems Before They Bite You

You shouldn't rely on "noticing" slowness — use tools that show you the actual query count.

```
Django Debug Toolbar          → shows query count per page in dev
django-silk                   → query profiling + duration
nplusone (library)            → raises warnings when N+1 detected
Postgres slow query log       → logs queries over a threshold (e.g. >100ms)
```

```
Page load without detection:            Page load with detection:
"Takes 4 seconds, not sure why"    →    "347 queries fired, 340 of them
                                          are duplicate SELECTs on `orders`"
```

### 2. `EXPLAIN ANALYZE` — Reading a Query Plan

Already touched on in Level 3 — but at this level you should be reading these routinely, not just when something is "obviously" slow.

```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 5;
```

```
🚩 Bad sign:
Seq Scan on orders (cost=0.00..2500.00 rows=50000)
  Filter: (user_id = 5)
  Rows Removed by Filter: 49950        ← scanned 50k rows to find 50!

✅ Good sign:
Index Scan using idx_orders_user_id on orders (cost=0.29..8.31 rows=50)
  Index Cond: (user_id = 5)
```

### 3. Query Batching (Bulk Operations)

Looping and issuing one `INSERT`/`UPDATE` per item is its own version of the N+1 problem — applied to writes instead of reads.

```
❌ N individual inserts                  ✅ 1 bulk insert
for item in items:                       Model.objects.bulk_create(items)
    Model.objects.create(**item)
→ N round trips                          → 1 round trip
```

```
1,000 individual INSERTs  →  1,000 round trips  →  slow
1 bulk INSERT (1,000 rows) →  1 round trip        →  fast
```

### 4. Caching Query Results

Not every query needs to hit the database every single time — especially data that doesn't change often (product catalogs, config, user profile basics).

```
Request 1 → Django → Database → Redis (store result, TTL=5min)
Request 2 → Django → Redis (cache hit!) → skip database entirely
```

```
       Request
          │
          ▼
   ┌─────────────┐   hit    ┌───────────────┐
   │    Redis     │────────►│ Return cached   │
   │  (checked     │          │ result instantly│
   │   first)      │          └───────────────┘
   └─────────────┘
          │ miss
          ▼
   ┌─────────────┐
   │  Database     │  →  store result in Redis for next time
   └─────────────┘
```

### 5. Read Replicas

For read-heavy applications, you can offload `SELECT` queries to **copies** of your database, keeping the main (primary) database free for writes.

```
                     Writes (INSERT/UPDATE/DELETE)
                              │
                              ▼
                     ┌─────────────────┐
                     │  Primary DB       │
                     └────────┬─────────┘
                              │ replication
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
     ┌───────────────┐ ┌───────────────┐ ┌───────────────┐
     │  Read Replica 1 │ │  Read Replica 2 │ │  Read Replica 3 │
     └───────────────┘ └───────────────┘ └───────────────┘
              ▲               ▲               ▲
              └───────────────┴───────────────┘
                     Reads (SELECT queries)
```

**Necessity:** a single database server can only handle so many concurrent reads before it becomes a bottleneck — replicas let you scale reads horizontally, while writes still go through one consistent source of truth.

### 6. Database Connection Pool Sizing

Already introduced connection pooling in Level 3 — at this level, you need to understand it's not "set and forget." A pool that's too small causes requests to queue; too large can overwhelm the database itself.

```
Pool size = 10                          Pool size = 500 (too high)
   │                                        │
   ▼                                        ▼
10 concurrent DB connections         500 concurrent connections
Requests beyond 10 wait in queue     Database itself gets overloaded
                                      trying to manage 500 connections
```

### 7. Denormalization (the deliberate opposite of normalization)

Sometimes, for read-heavy hot paths, you intentionally **duplicate** data to avoid expensive JOINs — trading storage and write complexity for read speed.

```
Normalized (always fresh, but needs JOIN):
orders.user_id → users.name

Denormalized (duplicated, but instant read):
orders.user_id, orders.user_name_snapshot   ← copy stored directly on order
```

**Necessity:** useful when a JOIN is genuinely too expensive at scale, or when you need a historical snapshot (e.g. an invoice should show the user's name *at the time of purchase*, even if they later change their name).

### 8. Materialized Views

A materialized view stores the **result of a complex query** physically on disk, refreshed on a schedule — instead of recalculating it live every single time.

```sql
CREATE MATERIALIZED VIEW monthly_sales AS
SELECT category, SUM(total) AS revenue
FROM orders
GROUP BY category;

-- refreshed periodically, e.g. every hour
REFRESH MATERIALIZED VIEW monthly_sales;
```

```
Without materialized view:              With materialized view:
Every dashboard load recalculates       Dashboard reads pre-computed
SUM() over millions of order rows       result — instant, even if the
→ slow, especially with many users      underlying data is huge
```

### 9. Sharding (for extreme scale)

Splitting one large table across **multiple physical databases**, so no single database has to hold or serve all the data.

```
                     users table (100 million rows)
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
     ┌───────────────┐ ┌───────────────┐ ┌───────────────┐
     │ Shard 1          │ │ Shard 2          │ │ Shard 3          │
     │ users id 1–33M   │ │ users id 34–66M  │ │ users id 67–100M │
     └───────────────┘ └───────────────┘ └───────────────┘
```

**Necessity:** at extreme scale, even read replicas aren't enough — a single write-primary can become a bottleneck for both storage and write throughput, so the data itself gets partitioned across independent databases.

### 10. Lazy vs Eager Loading (the general principle behind select/prefetch_related)

```
Lazy loading  → fetch related data only when accessed
                (this is what CAUSES the N+1 problem by default)

Eager loading → fetch related data upfront, in a batch
                (this is what select_related/prefetch_related give you)
```

```
Lazy (default, dangerous in a loop):
user.orders.all()   ← triggers a query EVERY time it's accessed

Eager (deliberate, safe in a loop):
User.objects.prefetch_related('orders')   ← fetched once, upfront,
                                              reused for every user in the loop
```

---

## Quick-Reference: Why Each Concept Exists

| Concept | Necessity in one line |
|---|---|
| N+1 detection | Catch hidden performance bugs before they reach production |
| `select_related()` | Fetch ForeignKey/OneToOne data in a single JOIN query |
| `prefetch_related()` | Fetch reverse/many-to-many data in a batched second query |
| `EXPLAIN ANALYZE` | See exactly how Postgres executes your query, not guess |
| Bulk operations | Avoid N individual write round-trips |
| Caching | Skip the database entirely for data that rarely changes |
| Read replicas | Scale read traffic horizontally without touching writes |
| Connection pool sizing | Balance concurrency against database connection overhead |
| Denormalization | Trade storage/write complexity for faster reads on hot paths |
| Materialized views | Pre-compute expensive aggregations instead of recalculating live |
| Sharding | Split data across databases when a single one can't keep up |
| Lazy vs eager loading | Understand *when* a query fires, so you can control it deliberately |
