# MODULE 6: Indexing (ইন্ডেক্সিং)
## বাংলায় সম্পূর্ণ Indexing গাইড — Production Level

> **কেন এই module সবচেয়ে গুরুত্বপূর্ণ?**
> Senior engineer দের interview এ সবচেয়ে বেশি জিজ্ঞেস করা হয় indexing নিয়ে।
> Production slow query এর ৯০% সমস্যা সঠিক index না থাকার কারণে।
> Amazon, Netflix, Uber — সবাই index strategy তে বিশেষ মনোযোগ দেয়।

---

## 6.1 Index কী? (What is an Index?)

### সহজ ভাষায় বোঝো

মনে করো তোমার কাছে একটা বই আছে — ১০০০ পাতার। তুমি "Normalization" শব্দটা খুঁজছো। দুটো উপায়:

```
1. প্রতিটা পাতা উল্টে খোঁজো → O(n) → Table Full Scan
2. বইয়ের শেষে Index দেখো → সরাসরি পাতা নম্বরে যাও → O(log n)
```

Database index ঠিক এভাবেই কাজ করে।

### Database এ Index কীভাবে কাজ করে

```
WITHOUT INDEX:
SELECT * FROM users WHERE email = 'rahim@example.com';

Database: "email = 'rahim@example.com' খুঁজতে হবে..."
→ Row 1 check করো... না
→ Row 2 check করো... না
→ Row 3 check করো... না
...
→ Row 10,000,000 check করো... পেয়েছি!
= 10 million row scan → SLOW!

WITH INDEX on email:
→ B-Tree traverse করো
→ 25-30 steps এ খুঁজে পাও (log₂ 10,000,000 ≈ 23)
= FAST!
```

### Index এর Internal Structure

```
Index = একটা আলাদা data structure যা:
  - Column এর value store করে (sorted)
  - সেই row এর disk location (pointer) store করে
  - Original table থেকে আলাদা থাকে
  - Write হলে automatically update হয়
```

---

## 6.2 B-Tree Index (বি-ট্রি ইন্ডেক্স)

### PostgreSQL এর Default Index

PostgreSQL এ যখন তুমি `CREATE INDEX` লেখো, by default B-Tree index তৈরি হয়।

### B-Tree Structure (Visual)

```
B-Tree for users.last_name column:

                    [M]
                   /   \
             [D, H]     [R, T]
            /  |  \    /  |  \
          [A] [E] [J] [N] [S] [V]
          ...  ...  ...  ...  ...

- Root node থেকে search শুরু
- প্রতিটা level এ comparison করো
- Leaf node এ actual row pointer আছে
- Height সাধারণত 3-4 level (millions of rows এও)
```

### B-Tree এর বৈশিষ্ট্য

```
✅ Supports:  =, <, >, <=, >=, BETWEEN, LIKE 'prefix%'
✅ Range queries তে efficient
✅ Sorted order maintain করে
✅ ORDER BY তে help করে
❌ Supports না: LIKE '%suffix', full-text search
```

### SQL Example

```sql
-- B-Tree index তৈরি
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_created_at ON orders(created_at);
CREATE INDEX idx_products_price ON products(price);

-- Range query — B-Tree index ব্যবহার করবে
SELECT * FROM orders
WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31';

-- LIKE prefix — B-Tree ব্যবহার করবে
SELECT * FROM users WHERE email LIKE 'rahim%';

-- LIKE suffix — B-Tree ব্যবহার করবে না (full scan!)
SELECT * FROM users WHERE email LIKE '%@gmail.com';
```

### Django ORM — Index তৈরি

```python
class User(models.Model):
    email      = models.EmailField(unique=True)  # automatically creates index
    last_name  = models.CharField(max_length=100)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        indexes = [
            models.Index(fields=['last_name']),          # B-Tree (default)
            models.Index(fields=['created_at']),
            models.Index(fields=['last_name', 'email']), # Composite B-Tree
        ]
```

**Generated SQL:**
```sql
CREATE INDEX "myapp_user_last_name" ON "users" ("last_name");
CREATE INDEX "myapp_user_created_at" ON "users" ("created_at");
```

---

## 6.3 Hash Index (হ্যাশ ইন্ডেক্স)

### কীভাবে কাজ করে

```
Hash Function:
  "rahim@example.com" → hash() → bucket #4521 → row pointer

Structure:
  Key (hashed) → Pointer to row

Time Complexity: O(1) for equality lookup
```

### B-Tree vs Hash — পার্থক্য

```
+------------------+------------------+------------------+
| Feature          | B-Tree           | Hash             |
+------------------+------------------+------------------+
| Equality (=)     | ✅ Fast           | ✅ Faster         |
| Range (<, >)     | ✅ Supports       | ❌ Not supported  |
| ORDER BY         | ✅ Helps          | ❌ No help        |
| LIKE 'prefix%'   | ✅ Supports       | ❌ Not supported  |
| Size             | Larger           | Smaller          |
| Use case         | General purpose  | Equality only    |
+------------------+------------------+------------------+
```

### SQL Example

```sql
-- Hash index তৈরি (PostgreSQL)
CREATE INDEX idx_sessions_token ON sessions USING HASH (session_token);

-- Hash index শুধু equality তে কাজ করে
SELECT * FROM sessions WHERE session_token = 'abc123xyz'; -- ✅ Fast
SELECT * FROM sessions WHERE session_token > 'abc123xyz'; -- ❌ Hash index ব্যবহার হবে না
```

### Django ORM — Hash Index

```python
from django.db.models import Index

class Session(models.Model):
    session_token = models.CharField(max_length=255)
    user          = models.ForeignKey(User, on_delete=models.CASCADE)
    expires_at    = models.DateTimeField()

    class Meta:
        indexes = [
            Index(fields=['session_token'], name='idx_session_token_hash',
                  opclasses=['text_pattern_ops']),  # PostgreSQL specific
        ]
```

> **Production Tip:** Hash index খুব কম ব্যবহার হয়। বেশিরভাগ case এ B-Tree যথেষ্ট। Hash index শুধু তখন ব্যবহার করো যখন শুধু equality check করতে হবে এবং B-Tree এর size problem হচ্ছে।

---

## 6.4 Composite Index (কম্পোজিট ইন্ডেক্স)

### কী এবং কেন?

Multiple column মিলিয়ে একটা index — এটাই Composite Index।

### The Golden Rule: Column Order Matters!

```
Index on (last_name, first_name, email)

এই index কাজ করবে:
✅ WHERE last_name = 'Khan'
✅ WHERE last_name = 'Khan' AND first_name = 'Rahim'
✅ WHERE last_name = 'Khan' AND first_name = 'Rahim' AND email = 'r@x.com'

এই index কাজ করবে না (full scan):
❌ WHERE first_name = 'Rahim'            -- leftmost column skip করা
❌ WHERE email = 'r@x.com'              -- leftmost column skip করা
❌ WHERE first_name = 'Rahim' AND email -- leftmost column skip করা
```

এটাকে বলে **"Leftmost Prefix Rule"**।

### Real-world Example — Amazon Orders

```sql
-- Amazon এর order query pattern:
-- "customer এর orders, date range দিয়ে filter করো"
SELECT * FROM orders
WHERE customer_id = 1234
  AND status = 'delivered'
  AND created_at >= '2024-01-01';

-- সঠিক composite index:
CREATE INDEX idx_orders_customer_status_date
ON orders(customer_id, status, created_at);

-- Column order logic:
-- 1. customer_id → highest cardinality filter, সবচেয়ে আগে
-- 2. status      → equality filter
-- 3. created_at  → range filter, সবসময় শেষে
```

### Django ORM

```python
class Order(models.Model):
    customer   = models.ForeignKey(Customer, on_delete=models.CASCADE)
    status     = models.CharField(max_length=20)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        indexes = [
            models.Index(
                fields=['customer', 'status', 'created_at'],
                name='idx_orders_customer_status_date'
            ),
        ]
```

**Query এবং Generated SQL:**
```python
orders = Order.objects.filter(
    customer_id=1234,
    status='delivered',
    created_at__gte='2024-01-01'
).order_by('-created_at')

# Generated SQL:
# SELECT * FROM orders
# WHERE customer_id = 1234
#   AND status = 'delivered'
#   AND created_at >= '2024-01-01'
# ORDER BY created_at DESC;
# -- Uses: idx_orders_customer_status_date ✅
```

---

## 6.5 Covering Index (কভারিং ইন্ডেক্স)

### কী এবং কেন?

Covering Index এমন একটা index যা query র সব column contain করে — তাই database কে main table এ যেতেই হয় না।

```
Normal Index:
  Query → Index (find row pointer) → Table (fetch actual row data)
  = 2 reads

Covering Index:
  Query → Index (সব data এখানেই আছে!) → Done
  = 1 read → FASTER!
```

### Example

```sql
-- Query: customer এর pending orders এর id আর created_at দরকার
SELECT id, created_at FROM orders
WHERE customer_id = 1234 AND status = 'pending';

-- Normal index — table এও যেতে হবে id আর created_at এর জন্য
CREATE INDEX idx_orders_cust_status ON orders(customer_id, status);

-- Covering index — index এই id আর created_at include করা আছে
CREATE INDEX idx_orders_covering
ON orders(customer_id, status)
INCLUDE (id, created_at);  -- PostgreSQL 11+ INCLUDE syntax
```

### EXPLAIN দিয়ে পার্থক্য দেখো

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, created_at FROM orders
WHERE customer_id = 1234 AND status = 'pending';

-- Without covering index:
-- Index Scan on idx_orders_cust_status
-- → Heap Fetches: 500  ← table এ যাচ্ছে

-- With covering index:
-- Index Only Scan on idx_orders_covering
-- → Heap Fetches: 0   ← table এ যেতে হচ্ছে না!
```

### Django ORM

```python
class Order(models.Model):
    class Meta:
        indexes = [
            models.Index(
                fields=['customer', 'status'],
                name='idx_orders_covering',
                include=['id', 'created_at'],  # Django 4.2+
            ),
        ]
```


---

## 6.6 Clustered vs Non-Clustered Index

### Clustered Index

```
Clustered Index মানে: table এর physical data সেই index এর order অনুযায়ী
disk এ সাজানো থাকে।

PostgreSQL এ: Primary Key = Clustered Index (by default HEAP storage,
কিন্তু CLUSTER command দিয়ে physically reorder করা যায়)

MySQL InnoDB তে: Primary Key সবসময় Clustered Index।
```

```
Clustered Index Structure:

Leaf Node তে actual row data থাকে:

        [50]
       /    \
    [25]    [75]
   /    \   /   \
[10,20][30,40][60,70][80,90]
  ↓       ↓      ↓      ↓
actual  actual actual actual
row     row    row    row
data    data   data   data
```

### Non-Clustered Index

```
Non-Clustered Index মানে: index আলাদা structure এ থাকে।
Leaf node এ actual data নেই — শুধু pointer (row location) আছে।

Non-Clustered Index Structure:

        [M]
       /   \
    [D,H]  [R,T]
   ...      ...
    ↓        ↓
 pointer  pointer
    ↓        ↓
[actual  [actual
 table]   table]
```

### পার্থক্য

```
+----------------------+----------------------+----------------------+
| Feature              | Clustered            | Non-Clustered        |
+----------------------+----------------------+----------------------+
| Data storage         | Index = Data         | Index আলাদা          |
| Per table count      | 1 টাই হতে পারে      | অনেকগুলো হতে পারে   |
| Range query          | Very fast            | Slower (extra lookup)|
| Insert/Update        | Slower (reorder)     | Faster               |
| PostgreSQL default   | PK (logical)         | সব বাকি index        |
+----------------------+----------------------+----------------------+
```

### SQL Example

```sql
-- PostgreSQL এ CLUSTER command
CREATE INDEX idx_orders_created ON orders(created_at);
CLUSTER orders USING idx_orders_created;
-- এখন orders table physically created_at order এ সাজানো

-- MySQL InnoDB — PK automatically clustered
CREATE TABLE orders (
    id         BIGINT AUTO_INCREMENT PRIMARY KEY,  -- Clustered!
    customer_id BIGINT,
    created_at DATETIME
);

-- Non-clustered (secondary index in MySQL)
CREATE INDEX idx_customer ON orders(customer_id);
```

---

## 6.7 Partial Index (পার্শিয়াল ইন্ডেক্স)

### কী এবং কেন?

Partial Index = শুধু নির্দিষ্ট rows এর উপর index। পুরো table index করার দরকার নেই।

```
পুরো orders table এ index:
  - 10 million rows index হবে
  - Index size: 500MB
  - Maintenance cost: বেশি

Partial index শুধু 'pending' orders এ:
  - মাত্র 50,000 rows index হবে
  - Index size: 2.5MB
  - Maintenance cost: অনেক কম
  - Query: সমান fast!
```

### Real-world Example — Uber

```sql
-- Uber এর active rides query — সবসময় 'active' status এর rides দরকার
-- কিন্তু 99% rides 'completed' — তাদের index দরকার নেই

-- Full index — waste!
CREATE INDEX idx_rides_status ON rides(status);

-- Partial index — শুধু active rides
CREATE INDEX idx_rides_active
ON rides(driver_id, created_at)
WHERE status = 'active';

-- Query — এই index ব্যবহার করবে
SELECT * FROM rides
WHERE status = 'active' AND driver_id = 5678;
```

### Django ORM

```python
class Order(models.Model):
    status     = models.CharField(max_length=20)
    customer   = models.ForeignKey(Customer, on_delete=models.CASCADE)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        indexes = [
            models.Index(
                fields=['customer', 'created_at'],
                name='idx_pending_orders',
                condition=models.Q(status='pending'),  # Partial index!
            ),
        ]
```

**Generated SQL:**
```sql
CREATE INDEX "idx_pending_orders"
ON "orders" ("customer_id", "created_at")
WHERE "status" = 'pending';
```

### আরেকটা Example — Soft Delete

```sql
-- Soft delete table এ — deleted rows index করার দরকার নেই
CREATE INDEX idx_users_email_active
ON users(email)
WHERE deleted_at IS NULL;

-- এই index শুধু active users এর email lookup এ কাজ করবে
SELECT * FROM users WHERE email = 'x@y.com' AND deleted_at IS NULL;
```

---

## 6.8 Unique Index (ইউনিক ইন্ডেক্স)

```sql
-- Unique constraint automatically unique index তৈরি করে
CREATE UNIQUE INDEX idx_users_email ON users(email);

-- বা constraint দিয়ে
ALTER TABLE users ADD CONSTRAINT uq_users_email UNIQUE (email);

-- Composite unique index
CREATE UNIQUE INDEX idx_enrollments_student_course
ON enrollments(student_id, course_id);
```

```python
# Django ORM
class User(models.Model):
    email = models.EmailField(unique=True)  # unique index auto-created

class Enrollment(models.Model):
    student = models.ForeignKey(Student, on_delete=models.CASCADE)
    course  = models.ForeignKey(Course, on_delete=models.CASCADE)

    class Meta:
        unique_together = ('student', 'course')
        # অথবা:
        constraints = [
            models.UniqueConstraint(
                fields=['student', 'course'],
                name='uq_enrollment_student_course'
            )
        ]
```

---

## 6.9 Full-Text Index (ফুল-টেক্সট ইন্ডেক্স)

### কেন দরকার?

```sql
-- LIKE দিয়ে text search — B-Tree index কাজ করে না
SELECT * FROM products WHERE description LIKE '%wireless headphone%';
-- এটা FULL TABLE SCAN করবে — 10M rows হলে dead slow!

-- Full-text search দরকার
```

### PostgreSQL Full-Text Search

```sql
-- tsvector column যোগ করো
ALTER TABLE products ADD COLUMN search_vector tsvector;

-- Populate করো
UPDATE products
SET search_vector = to_tsvector('english', name || ' ' || description);

-- GIN index তৈরি করো (full-text এর জন্য)
CREATE INDEX idx_products_search ON products USING GIN(search_vector);

-- Search query
SELECT * FROM products
WHERE search_vector @@ to_tsquery('english', 'wireless & headphone');

-- Ranking সহ
SELECT *, ts_rank(search_vector, query) AS rank
FROM products, to_tsquery('english', 'wireless & headphone') query
WHERE search_vector @@ query
ORDER BY rank DESC;
```

### Auto-update trigger

```sql
CREATE FUNCTION update_search_vector() RETURNS trigger AS $$
BEGIN
    NEW.search_vector := to_tsvector('english',
        COALESCE(NEW.name, '') || ' ' || COALESCE(NEW.description, ''));
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER products_search_vector_update
BEFORE INSERT OR UPDATE ON products
FOR EACH ROW EXECUTE FUNCTION update_search_vector();
```

### Django ORM — Full-Text Search

```python
from django.contrib.postgres.search import SearchVector, SearchQuery, SearchRank
from django.contrib.postgres.indexes import GinIndex

class Product(models.Model):
    name        = models.CharField(max_length=200)
    description = models.TextField()

    class Meta:
        indexes = [GinIndex(fields=['name'], name='idx_product_name_gin')]

# Search query
from django.contrib.postgres.search import SearchVector, SearchQuery

results = Product.objects.annotate(
    search=SearchVector('name', 'description')
).filter(search=SearchQuery('wireless headphone'))
```

**Generated SQL:**
```sql
SELECT *, ts_rank(search, query) AS rank
FROM products,
     to_tsquery('wireless & headphone') query
WHERE to_tsvector('english', name || ' ' || description) @@ query;
```

---

## 6.10 Query Execution Impact — EXPLAIN দিয়ে দেখো

### Index আছে vs নেই — তুলনা

```sql
-- Test setup
CREATE TABLE orders (
    id          BIGSERIAL PRIMARY KEY,
    customer_id BIGINT,
    status      VARCHAR(20),
    created_at  TIMESTAMPTZ
);
-- 5 million rows insert করো

-- Index ছাড়া EXPLAIN
EXPLAIN ANALYZE
SELECT * FROM orders WHERE customer_id = 1234;

-- Output:
-- Seq Scan on orders (cost=0.00..98765.00 rows=50 width=64)
--                     (actual time=245.123..892.456 rows=50 loops=1)
-- → 892ms, Full table scan!

-- Index তৈরি করো
CREATE INDEX idx_orders_customer ON orders(customer_id);

-- Index সহ EXPLAIN
EXPLAIN ANALYZE
SELECT * FROM orders WHERE customer_id = 1234;

-- Output:
-- Index Scan using idx_orders_customer on orders
--   (cost=0.56..312.45 rows=50 width=64)
--   (actual time=0.045..1.234 rows=50 loops=1)
-- → 1.2ms! 700x faster!
```

### EXPLAIN output বোঝার গাইড

```
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 1234;

Output terms:
┌─────────────────────────────────────────────────────────────┐
│ Seq Scan          → Full table scan (bad for large tables)  │
│ Index Scan        → Index ব্যবহার করছে (good)              │
│ Index Only Scan   → Covering index (best!)                  │
│ Bitmap Index Scan → Multiple index combine (moderate)       │
│                                                             │
│ cost=0.00..98765  → estimated cost (startup..total)         │
│ rows=50           → estimated rows                          │
│ actual time=0..1  → real time in ms                         │
│ loops=1           → কতবার execute হয়েছে                    │
│ Buffers: hit=X    → cache থেকে পড়েছে (fast)               │
│ Buffers: read=X   → disk থেকে পড়েছে (slow)               │
└─────────────────────────────────────────────────────────────┘
```

---

## 6.11 Slow Query Optimization — Real Production Examples

### Example 1 — N+1 Problem with Index

```python
# ❌ N+1 problem
orders = Order.objects.filter(status='pending')
for order in orders:
    print(order.customer.name)  # প্রতিটা order এ আলাদা query!
# 1 + N queries!

# ✅ Fix with select_related + index
orders = Order.objects.filter(
    status='pending'
).select_related('customer')
# 1 query with JOIN

# Index দরকার:
# CREATE INDEX idx_orders_status ON orders(status);
# CREATE INDEX idx_customers_id ON customers(id); -- PK already indexed
```

### Example 2 — Missing Index on Foreign Key

```sql
-- Common mistake: FK column এ index নেই
SELECT o.*, c.name
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE c.email = 'rahim@example.com';

-- EXPLAIN দেখাবে: Seq Scan on orders ← problem!
-- Fix:
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_customers_email ON customers(email);
```

### Example 3 — Function on Indexed Column (Index Killer!)

```sql
-- ❌ Index কাজ করবে না — column এ function apply হচ্ছে
SELECT * FROM orders WHERE YEAR(created_at) = 2024;
SELECT * FROM users  WHERE LOWER(email) = 'rahim@example.com';
SELECT * FROM orders WHERE DATE(created_at) = '2024-01-15';

-- ✅ Fix — function ছাড়া লেখো
SELECT * FROM orders
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';

-- অথবা functional index তৈরি করো
CREATE INDEX idx_users_email_lower ON users(LOWER(email));
SELECT * FROM users WHERE LOWER(email) = 'rahim@example.com'; -- এখন index ব্যবহার হবে
```

```python
# Django ORM — functional index
from django.db.models.functions import Lower

class User(models.Model):
    email = models.EmailField()

    class Meta:
        indexes = [
            models.Index(Lower('email'), name='idx_user_email_lower'),
        ]
```

---

## 6.12 Index Strategy (ইন্ডেক্স স্ট্র্যাটেজি)

### কোন Column এ Index দেবে?

```
✅ Index দাও:
  - WHERE clause এ বারবার ব্যবহৃত columns
  - JOIN condition এর columns (FK)
  - ORDER BY columns
  - GROUP BY columns
  - High cardinality columns (unique values বেশি)
  - Foreign keys (সবসময়!)

❌ Index দিও না:
  - Low cardinality columns (boolean, gender — শুধু 2-3 values)
  - খুব বেশি write হয় এমন columns
  - ছোট table (< 1000 rows) — full scan faster
  - Rarely queried columns
```

### Cardinality এর গুরুত্ব

```sql
-- Low cardinality — index useless
-- status column এ 4 values: pending/confirmed/shipped/delivered
-- প্রতিটা value এ 25% rows → index কোনো help করে না
CREATE INDEX idx_orders_status ON orders(status);  -- useless for single value query!

-- High cardinality — index very useful
-- email column — প্রতিটা row আলাদা value
CREATE INDEX idx_users_email ON users(email);  -- very useful!

-- Composite: low cardinality + high cardinality
CREATE INDEX idx_orders_status_customer ON orders(status, customer_id);
-- status দিয়ে narrow down করো, customer_id দিয়ে specific rows খোঁজো
```

---

## 6.13 Over-Indexing Problems

### সমস্যা কী?

```
একটা table এ 15টা index আছে।

INSERT একটা row:
→ Main table update
→ 15টা index update
= 16 write operations!

বড় system এ এটা INSERT/UPDATE/DELETE কে 10-20x slow করে দেয়।
```

### Real Production Example

```sql
-- ❌ Over-indexed table
CREATE TABLE events (
    id         BIGSERIAL PRIMARY KEY,
    user_id    BIGINT,
    event_type VARCHAR(50),
    payload    JSONB,
    created_at TIMESTAMPTZ
);
CREATE INDEX idx1 ON events(user_id);
CREATE INDEX idx2 ON events(event_type);
CREATE INDEX idx3 ON events(created_at);
CREATE INDEX idx4 ON events(user_id, event_type);
CREATE INDEX idx5 ON events(user_id, created_at);
CREATE INDEX idx6 ON events(event_type, created_at);
CREATE INDEX idx7 ON events(user_id, event_type, created_at);
-- এটা high-write event table এর জন্য disaster!

-- ✅ Analyze করো — কোন query বেশি হয়?
-- সাধারণত: user_id + created_at range query
CREATE INDEX idx_events_user_date ON events(user_id, created_at);
-- এই একটাই যথেষ্ট!
```

### Unused Index খুঁজে বের করো

```sql
-- PostgreSQL — unused indexes
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,      -- কতবার index ব্যবহার হয়েছে
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
WHERE idx_scan = 0  -- কখনো ব্যবহার হয়নি!
ORDER BY tablename;

-- Index size দেখো
SELECT
    indexname,
    pg_size_pretty(pg_relation_size(indexname::regclass)) AS size
FROM pg_indexes
WHERE tablename = 'orders'
ORDER BY pg_relation_size(indexname::regclass) DESC;
```

---

## 6.14 Common Mistakes (সাধারণ ভুল)

### ভুল ১: FK তে Index না দেওয়া

```sql
-- ❌ FK আছে কিন্তু index নেই
CREATE TABLE order_items (
    order_id   BIGINT REFERENCES orders(id),  -- No index!
    product_id BIGINT REFERENCES products(id) -- No index!
);

-- JOIN query তে full scan হবে

-- ✅ সবসময় FK তে index দাও
CREATE INDEX idx_items_order   ON order_items(order_id);
CREATE INDEX idx_items_product ON order_items(product_id);
```

### ভুল ২: OR condition এ index miss

```sql
-- ❌ OR condition এ single index কাজ নাও করতে পারে
SELECT * FROM users WHERE email = 'x@y.com' OR phone = '01711';

-- ✅ UNION দিয়ে rewrite করো (দুটো index ব্যবহার হবে)
SELECT * FROM users WHERE email = 'x@y.com'
UNION
SELECT * FROM users WHERE phone = '01711';
```

### ভুল ৩: Index সত্ত্বেও Seq Scan

```sql
-- এই situations এ PostgreSQL index ignore করে:
-- 1. Table ছোট (< ~1000 rows)
-- 2. Query অনেক বেশি rows return করে (> 10-20% of table)
-- 3. Statistics outdated — ANALYZE চালাও
-- 4. Column এ function apply হচ্ছে

ANALYZE orders;  -- statistics update করো
```

### ভুল ৪: Index তৈরি করা Table Lock করে

```sql
-- ❌ Production এ এটা table lock করবে!
CREATE INDEX idx_orders_customer ON orders(customer_id);

-- ✅ CONCURRENTLY ব্যবহার করো — no lock!
CREATE INDEX CONCURRENTLY idx_orders_customer ON orders(customer_id);
-- একটু বেশি সময় লাগবে কিন্তু production block হবে না
```

```python
# Django migration এ
from django.db import migrations

class Migration(migrations.Migration):
    atomic = False  # CONCURRENTLY এর জন্য atomic=False দরকার

    operations = [
        migrations.AddIndex(
            model_name='order',
            index=models.Index(
                fields=['customer'],
                name='idx_orders_customer',
                concurrently=True,  # Django 4.1+
            ),
        ),
    ]
```

---

## 6.15 Debugging Techniques

### Step 1: Slow Query চিহ্নিত করো

```sql
-- PostgreSQL slow query log enable করো (postgresql.conf)
-- log_min_duration_statement = 1000  -- 1 second এর বেশি সব log হবে

-- অথবা pg_stat_statements extension
SELECT
    query,
    calls,
    total_time,
    mean_time,
    rows
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 10;
```

### Step 2: EXPLAIN ANALYZE চালাও

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.id, c.name, o.total_amount
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.status = 'pending'
  AND o.created_at >= NOW() - INTERVAL '7 days';
```

### Step 3: Django এ Slow Query Debug

```python
# settings.py — development এ
LOGGING = {
    'loggers': {
        'django.db.backends': {
            'level': 'DEBUG',
            'handlers': ['console'],
        },
    },
}

# Shell এ query count দেখো
from django.db import connection, reset_queries
from django.conf import settings

settings.DEBUG = True
reset_queries()

# তোমার code চালাও
orders = list(Order.objects.filter(status='pending').select_related('customer'))

print(f"Query count: {len(connection.queries)}")
for q in connection.queries:
    print(q['sql'])
    print(f"Time: {q['time']}s")
```

---

## 6.16 Optimization Strategies

### Strategy 1: Index-First Design

```
নতুন feature develop করার সময়:
1. Query pattern আগে ঠিক করো
2. সেই query এর জন্য index design করো
3. তারপর table design করো
```

### Strategy 2: Composite Index Column Order

```
Rule of Thumb:
1. Equality conditions আগে (=)
2. High cardinality columns আগে
3. Range conditions শেষে (<, >, BETWEEN)
4. ORDER BY columns শেষে

Example:
WHERE status = 'active'    -- equality, low cardinality
  AND customer_id = 1234   -- equality, high cardinality
  AND created_at > '2024'  -- range

Index: (customer_id, status, created_at)
       ↑ high card  ↑ eq    ↑ range last
```

### Strategy 3: Periodic Index Maintenance

```sql
-- Index bloat check করো
SELECT
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexname::regclass)) AS index_size
FROM pg_indexes
WHERE schemaname = 'public'
ORDER BY pg_relation_size(indexname::regclass) DESC;

-- REINDEX — index rebuild (bloat কমায়)
REINDEX INDEX CONCURRENTLY idx_orders_customer;
```

---

## 6.17 Trade-offs — কখন Index ব্যবহার করবে না

```
❌ Index এড়াও যখন:
  1. Table ছোট (< 10,000 rows) — full scan faster
  2. Bulk load/ETL — আগে index drop করো, পরে recreate করো
  3. Write-heavy tables এ অতিরিক্ত index
  4. Low cardinality single column (boolean, status with 2 values)
  5. Temporary tables
  6. Columns যেগুলো WHERE/JOIN/ORDER BY তে আসে না

✅ Index রাখো যখন:
  1. Large table এ frequent SELECT
  2. JOIN columns (FK সবসময়)
  3. Unique constraints
  4. ORDER BY + LIMIT pattern (pagination)
  5. Reporting queries
```

---

## 6.18 Production Best Practices

```
1.  সব FK তে index দাও — no exception
2.  Production এ CONCURRENTLY দিয়ে index তৈরি করো
3.  pg_stat_user_indexes দিয়ে unused index খুঁজে drop করো
4.  Composite index এ column order মেনে চলো (leftmost prefix rule)
5.  Partial index দিয়ে index size কমাও
6.  Full-text search এর জন্য GIN index ব্যবহার করো
7.  Index bloat monitor করো এবং REINDEX CONCURRENTLY চালাও
8.  Bulk insert এর আগে index disable করো (ETL jobs)
9.  EXPLAIN ANALYZE দিয়ে index ব্যবহার verify করো
10. Over-indexing avoid করো — প্রতিটা index write কে slow করে
```

---

## 6.19 Senior Engineer Insights

> **War Story — Netflix:**
> Netflix এর একটা recommendation query ছিল যেটা 4 seconds নিচ্ছিল। EXPLAIN দেখে বোঝা গেল একটা JOIN column এ index ছিল না। একটা index যোগ করার পর query 8ms এ নামে। এটাই indexing এর power।

> **Design Decision:**
> Index design করার আগে সবসময় প্রশ্ন করো: "এই query কতবার চলে? কত rows return করে? Table এ কত rows আছে?" এই তিনটা প্রশ্নের উত্তর জানলে সঠিক index strategy নির্ধারণ করা যায়।

> **Google এর Rule:**
> Index = Read কে fast করো, Write কে slow করো। Balance খোঁজো।

---

## 6.20 Interview Questions

### Beginner Level

**Q1: Index কী এবং কেন ব্যবহার করা হয়?**

> Index হলো database এর একটা data structure যা নির্দিষ্ট column এর value এবং সেই row এর location সংরক্ষণ করে। এটা SELECT query কে fast করে কারণ full table scan না করে সরাসরি data খুঁজে পাওয়া যায়। Trade-off হলো write operation slow হয় এবং extra storage লাগে।

**Q2: Primary Key এ কি automatically index তৈরি হয়?**

> হ্যাঁ। PostgreSQL এবং MySQL উভয়েই Primary Key এ automatically unique index তৈরি করে।

**Q3: কোন column এ index দেওয়া উচিত?**

> WHERE clause, JOIN condition, ORDER BY, GROUP BY তে বারবার ব্যবহৃত columns এ। Foreign Key columns এ সবসময় index দেওয়া উচিত।

---

### Intermediate Level

**Q4: Composite index এ column order কেন গুরুত্বপূর্ণ?**

> Leftmost Prefix Rule এর কারণে। Index (A, B, C) তে WHERE A=1 কাজ করবে, কিন্তু WHERE B=1 কাজ করবে না। Equality conditions আগে, range conditions পরে রাখতে হয়।

**Q5: Covering index কী সুবিধা দেয়?**

> Query র সব column index এ থাকলে database main table এ যেতে হয় না — "Index Only Scan" হয়। এতে I/O কমে এবং query significantly faster হয়।

**Q6: EXPLAIN output এ Seq Scan দেখলে কী করবে?**

> ১. WHERE clause এর column এ index আছে কিনা দেখো। ২. Statistics update করতে ANALYZE চালাও। ৩. Query কি বেশি % rows return করছে? তাহলে index কাজ নাও করতে পারে। ৪. Column এ function apply হচ্ছে? Functional index দরকার।

---

### Senior / FAANG Level

**Q7: একটা high-write event tracking system এ index strategy কী হবে?**

> High-write system এ index minimize করতে হবে। Strategy:
> - শুধু mandatory indexes রাখো (PK, critical FK)
> - Partial index ব্যবহার করো (শুধু recent events এ)
> - Write path থেকে index আলাদা করো (CQRS: write DB আলাদা, read DB আলাদা)
> - Time-based partitioning করো এবং old partitions এ index detach করো
> - Batch insert করো (bulk_create) যাতে per-row index update কমে

**Q8: Index bloat কী এবং কীভাবে handle করবে?**

> PostgreSQL এ MVCC এর কারণে deleted/updated rows এর dead tuples index এ থেকে যায়। এটাকে index bloat বলে। Handle করতে:
> - VACUUM ANALYZE নিয়মিত চালাও
> - autovacuum properly configure করো
> - REINDEX CONCURRENTLY দিয়ে index rebuild করো
> - pg_stat_user_indexes দিয়ে bloat monitor করো

**Q9: System Design — Instagram এর photo feed এর জন্য index strategy design করো।**

```sql
-- Feed query pattern:
-- "user X এর following দের posts, last 7 days এর"

CREATE TABLE posts (
    id         BIGSERIAL PRIMARY KEY,
    user_id    BIGINT,
    created_at TIMESTAMPTZ,
    content    TEXT
);

CREATE TABLE follows (
    follower_id  BIGINT,
    following_id BIGINT,
    PRIMARY KEY (follower_id, following_id)
);

-- Index strategy:
-- 1. posts: (user_id, created_at DESC) — user এর recent posts
CREATE INDEX idx_posts_user_date ON posts(user_id, created_at DESC);

-- 2. follows: following_id এ index — "কে কাকে follow করে"
CREATE INDEX idx_follows_following ON follows(following_id);

-- 3. Partial index — শুধু last 30 days এর posts
CREATE INDEX idx_posts_recent ON posts(user_id, created_at)
WHERE created_at > NOW() - INTERVAL '30 days';

-- Feed query:
SELECT p.*
FROM posts p
JOIN follows f ON f.following_id = p.user_id
WHERE f.follower_id = 12345
  AND p.created_at > NOW() - INTERVAL '7 days'
ORDER BY p.created_at DESC
LIMIT 20;
```

> Senior answer: Real Instagram তে এই approach scale করে না (millions of followers)। তারা fan-out on write pattern ব্যবহার করে — post হলে সব followers এর feed table তে push করে। Index strategy তখন feed table এর (user_id, created_at) এ।

---

## 6.21 Hands-on Exercises

### Exercise 1
```sql
-- 1 million rows এর orders table তৈরি করো
-- EXPLAIN ANALYZE দিয়ে দেখো index আগে এবং পরে কতটা fast হয়
```

### Exercise 2
```
Composite index design করো এই query এর জন্য:
SELECT * FROM products
WHERE category_id = 5
  AND price BETWEEN 100 AND 500
  AND in_stock = TRUE
ORDER BY price ASC;
```

### Exercise 3
```python
# Django ORM দিয়ে:
# 1. Unused index খুঁজে বের করো
# 2. N+1 problem fix করো
# 3. Covering index এর সুবিধা দেখাও
```

### Mini Project
```
E-commerce system এর জন্য complete index strategy তৈরি করো:
- products, orders, order_items, customers, categories table
- Top 10 common queries চিহ্নিত করো
- প্রতিটা query এর জন্য index design করো
- EXPLAIN ANALYZE দিয়ে before/after দেখাও
- Over-indexing check করো
```

---

## 6.22 Module Summary & Cheat Sheet

```
INDEX TYPES CHEAT SHEET
========================
B-Tree    → Default, general purpose, =/</>/ BETWEEN/LIKE prefix%
Hash      → Equality only (=), faster than B-Tree for =
Composite → Multiple columns, leftmost prefix rule মেনে চলো
Covering  → All query columns in index, Index Only Scan
Clustered → Data physically sorted (1 per table)
Partial   → WHERE condition সহ index, size কমায়
Unique    → Uniqueness enforce করে
Full-Text → GIN index, tsvector, @@ operator

COLUMN ORDER RULE (Composite Index)
=====================================
1. Equality first  (=)
2. High cardinality first
3. Range last      (<, >, BETWEEN)
4. ORDER BY last

WHEN TO INDEX
==============
✅ FK columns (always!)
✅ WHERE clause frequent columns
✅ JOIN columns
✅ ORDER BY + LIMIT (pagination)
✅ High cardinality columns

WHEN NOT TO INDEX
==================
❌ Small tables (< 10K rows)
❌ Low cardinality (boolean, 2-3 values)
❌ Write-heavy columns (over-indexing)
❌ Never queried columns

PRODUCTION COMMANDS
====================
-- Non-blocking index creation
CREATE INDEX CONCURRENTLY idx_name ON table(col);

-- Unused indexes
SELECT indexname, idx_scan FROM pg_stat_user_indexes
WHERE idx_scan = 0;

-- Index sizes
SELECT indexname, pg_size_pretty(pg_relation_size(indexname::regclass))
FROM pg_indexes WHERE tablename = 'your_table';

-- Rebuild bloated index
REINDEX INDEX CONCURRENTLY idx_name;

-- Update statistics
ANALYZE table_name;

DJANGO INDEX CHEAT SHEET
=========================
# Simple index
models.Index(fields=['column'])

# Composite index
models.Index(fields=['col1', 'col2'])

# Partial index
models.Index(fields=['col'], condition=Q(status='active'))

# Covering index (Django 4.2+)
models.Index(fields=['col1'], include=['col2', 'col3'])

# Functional index
models.Index(Lower('email'), name='idx_email_lower')

# Unique constraint
models.UniqueConstraint(fields=['col1', 'col2'])

# Concurrent migration
class Migration(migrations.Migration):
    atomic = False
```

