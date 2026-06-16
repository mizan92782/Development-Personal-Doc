# MODULE 18: Practice (প্র্যাকটিস)
## বাংলায় সম্পূর্ণ Practice গাইড — SQL থেকে System Design

> **কেন Practice করবে?**
> Theory জানা আর করতে পারা — দুটো আলাদা জিনিস।
> Production engineer হতে হলে হাতে-কলমে practice ছাড়া কোনো উপায় নেই।
> এই module এ প্রতিটা module এর exercises, real interview assignments, এবং mini projects আছে।

---

## 18.1 Setup — Practice Environment

```bash
# Docker দিয়ে PostgreSQL setup (সবচেয়ে সহজ)
docker run --name practice-db \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_DB=practice \
  -p 5432:5432 \
  -d postgres:15

# Connect করো
psql -h localhost -U postgres -d practice

# Redis setup
docker run --name practice-redis \
  -p 6379:6379 \
  -d redis:7

# Django project setup
pip install django psycopg2-binary redis django-debug-toolbar
django-admin startproject practice_project
cd practice_project
python manage.py startapp shop
```

### Practice Database — Seed Data

```sql
-- সব exercises এর জন্য এই schema ব্যবহার করো

CREATE TABLE users (
    id         BIGSERIAL PRIMARY KEY,
    name       VARCHAR(200) NOT NULL,
    email      VARCHAR(255) UNIQUE NOT NULL,
    country    VARCHAR(100) DEFAULT 'Bangladesh',
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE categories (
    id        BIGSERIAL PRIMARY KEY,
    name      VARCHAR(100) NOT NULL,
    parent_id BIGINT REFERENCES categories(id)
);

CREATE TABLE products (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(300) NOT NULL,
    category_id BIGINT REFERENCES categories(id),
    price       DECIMAL(12,2) NOT NULL,
    stock       INT DEFAULT 0,
    is_active   BOOLEAN DEFAULT TRUE,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE orders (
    id          BIGSERIAL PRIMARY KEY,
    user_id     BIGINT REFERENCES users(id),
    status      VARCHAR(30) DEFAULT 'pending',
    total       DECIMAL(12,2) NOT NULL,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE order_items (
    id         BIGSERIAL PRIMARY KEY,
    order_id   BIGINT REFERENCES orders(id),
    product_id BIGINT REFERENCES products(id),
    qty        INT NOT NULL,
    unit_price DECIMAL(12,2) NOT NULL
);

CREATE TABLE reviews (
    id         BIGSERIAL PRIMARY KEY,
    product_id BIGINT REFERENCES products(id),
    user_id    BIGINT REFERENCES users(id),
    rating     SMALLINT CHECK (rating BETWEEN 1 AND 5),
    comment    TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Seed data
INSERT INTO categories (name, parent_id) VALUES
  ('Electronics', NULL), ('Clothing', NULL), ('Books', NULL),
  ('Phones', 1), ('Laptops', 1), ('Shirts', 2), ('Pants', 2);

INSERT INTO users (name, email, country) VALUES
  ('Alice Rahman', 'alice@example.com', 'Bangladesh'),
  ('Bob Islam', 'bob@example.com', 'Bangladesh'),
  ('Charlie Das', 'charlie@example.com', 'India'),
  ('Diana Khan', 'diana@example.com', 'Bangladesh'),
  ('Eve Ahmed', 'eve@example.com', 'Pakistan');

INSERT INTO products (name, category_id, price, stock) VALUES
  ('Samsung Galaxy A54', 4, 35000, 50),
  ('iPhone 15', 4, 130000, 20),
  ('Dell Laptop', 5, 75000, 15),
  ('MacBook Pro', 5, 200000, 10),
  ('Blue Shirt', 6, 800, 200),
  ('Black Pants', 7, 1200, 150),
  ('Clean Code Book', 3, 2500, 100),
  ('Xiaomi Phone', 4, 18000, 80);

INSERT INTO orders (user_id, status, total) VALUES
  (1, 'completed', 35000),
  (1, 'completed', 130000),
  (2, 'pending', 75000),
  (3, 'completed', 800),
  (4, 'cancelled', 1200),
  (1, 'completed', 18000),
  (2, 'completed', 2500),
  (5, 'pending', 200000);

INSERT INTO order_items (order_id, product_id, qty, unit_price) VALUES
  (1, 1, 1, 35000), (2, 2, 1, 130000), (3, 3, 1, 75000),
  (4, 5, 1, 800), (5, 6, 1, 1200), (6, 8, 1, 18000),
  (7, 7, 1, 2500), (8, 4, 1, 200000);

INSERT INTO reviews (product_id, user_id, rating, comment) VALUES
  (1, 1, 5, 'Great phone!'), (1, 2, 4, 'Good value'),
  (2, 1, 5, 'Excellent!'), (3, 3, 3, 'Average'),
  (7, 4, 5, 'Must read'), (8, 5, 4, 'Good phone');
```

---

## 18.2 Module 1-2 Exercises — SQL Fundamentals

### Exercise Set A: Basic Queries

```sql
-- Exercise 1: সব active products এর নাম এবং price দেখাও, price অনুযায়ী sort করো।
-- Expected: 8 products, ascending price

-- Exercise 2: Bangladesh থেকে কতজন user আছে?
-- Expected: 3

-- Exercise 3: প্রতিটা category তে কতটা product আছে?
-- Expected: categories with product counts

-- Exercise 4: Average order value কত?
-- Expected: single number

-- Exercise 5: সবচেয়ে দামী product কোনটা?
-- Expected: MacBook Pro, 200000
```

### Solutions A

```sql
-- Exercise 1
SELECT name, price FROM products
WHERE is_active = TRUE
ORDER BY price ASC;

-- Exercise 2
SELECT COUNT(*) FROM users WHERE country = 'Bangladesh';

-- Exercise 3
SELECT c.name, COUNT(p.id) AS product_count
FROM categories c
LEFT JOIN products p ON p.category_id = c.id
GROUP BY c.id, c.name
ORDER BY product_count DESC;

-- Exercise 4
SELECT ROUND(AVG(total), 2) AS avg_order_value FROM orders;

-- Exercise 5
SELECT name, price FROM products ORDER BY price DESC LIMIT 1;
```

---

### Exercise Set B: GROUP BY, HAVING, CASE

```sql
-- Exercise 6: প্রতিটা user এর total spending দেখাও (completed orders only)
-- Exercise 7: 2+ orders আছে এমন users দেখাও
-- Exercise 8: প্রতিটা product এ average rating দেখাও (reviewed products only)
-- Exercise 9: Order কে price range এ categorize করো:
--   0-10000: 'Budget', 10001-50000: 'Mid Range', 50001+: 'Premium'
-- Exercise 10: প্রতিটা country থেকে কতজন user, কিন্তু শুধু 2+ users এর country
```

### Solutions B

```sql
-- Exercise 6
SELECT u.name, SUM(o.total) AS total_spent
FROM users u
JOIN orders o ON o.user_id = u.id
WHERE o.status = 'completed'
GROUP BY u.id, u.name
ORDER BY total_spent DESC;

-- Exercise 7
SELECT u.name, COUNT(o.id) AS order_count
FROM users u
JOIN orders o ON o.user_id = u.id
GROUP BY u.id, u.name
HAVING COUNT(o.id) >= 2;

-- Exercise 8
SELECT p.name, ROUND(AVG(r.rating), 2) AS avg_rating, COUNT(r.id) AS review_count
FROM products p
JOIN reviews r ON r.product_id = p.id
GROUP BY p.id, p.name
ORDER BY avg_rating DESC;

-- Exercise 9
SELECT id, total,
    CASE
        WHEN total <= 10000  THEN 'Budget'
        WHEN total <= 50000  THEN 'Mid Range'
        ELSE                      'Premium'
    END AS price_range
FROM orders;

-- Exercise 10
SELECT country, COUNT(*) AS user_count
FROM users
GROUP BY country
HAVING COUNT(*) >= 2;
```

---

## 18.3 Module 3 Exercises — JOINs

```sql
-- Exercise 11: সব users দেখাও — order আছে বা না থাকুক। Order count দেখাও।
-- Exercise 12: কোনো order নেই এমন users দেখাও।
-- Exercise 13: প্রতিটা order এর user name, product names, quantities দেখাও।
-- Exercise 14: Self join: categories এর parent name সহ দেখাও।
-- Exercise 15: সব products দেখাও যেগুলো কখনো order হয়নি।
```

### Solutions

```sql
-- Exercise 11
SELECT u.name, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id, u.name;

-- Exercise 12
SELECT u.name
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE o.id IS NULL;

-- Exercise 13
SELECT u.name AS customer, o.id AS order_id,
       p.name AS product, oi.qty
FROM orders o
JOIN users u ON o.user_id = u.id
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON oi.product_id = p.id
ORDER BY o.id;

-- Exercise 14
SELECT c.name AS category, parent.name AS parent_category
FROM categories c
LEFT JOIN categories parent ON c.parent_id = parent.id;

-- Exercise 15
SELECT p.name
FROM products p
LEFT JOIN order_items oi ON oi.product_id = p.id
WHERE oi.id IS NULL;
```

---

## 18.4 Module 4 Exercises — Advanced SQL

```sql
-- Exercise 16: প্রতিটা user এর most expensive order দেখাও (subquery)
-- Exercise 17: Average এর উপরে price আছে এমন products দেখাও
-- Exercise 18: প্রতিটা product এর sales rank দেখাও (window function)
-- Exercise 19: LEAD/LAG দিয়ে প্রতিটা order এর আগের order এর total দেখাও
-- Exercise 20: Recursive CTE দিয়ে category hierarchy দেখাও (full path)
```

### Solutions

```sql
-- Exercise 16
SELECT u.name, o.total AS max_order
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.total = (
    SELECT MAX(o2.total) FROM orders o2 WHERE o2.user_id = o.user_id
);

-- Exercise 17
SELECT name, price
FROM products
WHERE price > (SELECT AVG(price) FROM products);

-- Exercise 18
SELECT
    p.name,
    COALESCE(SUM(oi.qty), 0) AS units_sold,
    RANK() OVER (ORDER BY COALESCE(SUM(oi.qty), 0) DESC) AS sales_rank
FROM products p
LEFT JOIN order_items oi ON oi.product_id = p.id
GROUP BY p.id, p.name;

-- Exercise 19
SELECT
    id,
    total,
    LAG(total) OVER (PARTITION BY user_id ORDER BY created_at) AS prev_order_total
FROM orders;

-- Exercise 20
WITH RECURSIVE cat_path AS (
    SELECT id, name, parent_id, name::TEXT AS path
    FROM categories WHERE parent_id IS NULL

    UNION ALL

    SELECT c.id, c.name, c.parent_id, cp.path || ' > ' || c.name
    FROM categories c
    JOIN cat_path cp ON c.parent_id = cp.id
)
SELECT path FROM cat_path ORDER BY path;
```

---

## 18.5 Module 6 Exercises — Indexing

```sql
-- Exercise 21: EXPLAIN এ Seq Scan দেখো, তারপর index যোগ করো, আবার দেখো
EXPLAIN SELECT * FROM orders WHERE user_id = 1;
-- Seq Scan দেখবে

CREATE INDEX idx_orders_user ON orders(user_id);

EXPLAIN SELECT * FROM orders WHERE user_id = 1;
-- Index Scan দেখবে

-- Exercise 22: Composite index তৈরি করো এবং test করো
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

EXPLAIN ANALYZE
SELECT * FROM orders
WHERE user_id = 1 AND status = 'completed';

-- Exercise 23: Covering index তৈরি করো
-- Query: SELECT total FROM orders WHERE user_id = 1 AND status = 'completed'
-- এমন index তৈরি করো যাতে Index Only Scan হয়

CREATE INDEX idx_orders_covering
ON orders(user_id, status) INCLUDE (total);

EXPLAIN SELECT total FROM orders
WHERE user_id = 1 AND status = 'completed';
-- Index Only Scan দেখবে!

-- Exercise 24: Unused indexes identify করো
SELECT indexname, idx_scan
FROM pg_stat_user_indexes
WHERE relname = 'orders'
ORDER BY idx_scan;

-- Exercise 25: Partial index তৈরি করো
-- শুধু pending orders এ index (active subset)
CREATE INDEX idx_orders_pending
ON orders(user_id, created_at)
WHERE status = 'pending';
```

---

## 18.6 Module 7 Exercises — Query Optimization

### N+1 Detection and Fix

```python
# Django project এ এই কাজ করো:

# Exercise 26: N+1 detect করো
# settings.py তে DEBUG=True রাখো

from django.db import connection, reset_queries

reset_queries()

# ❌ N+1 code চালাও:
orders = Order.objects.all()
for order in orders:
    print(order.user.name)  # N+1!

print(f"Queries: {len(connection.queries)}")
# Expected: 1 + N queries

# ✅ Fix করো:
reset_queries()
orders = Order.objects.select_related('user').all()
for order in orders:
    print(order.user.name)

print(f"Queries: {len(connection.queries)}")
# Expected: 1 query

# Exercise 27: prefetch_related fix
reset_queries()
orders = Order.objects.prefetch_related('items', 'items__product').all()
for order in orders:
    for item in order.items.all():
        print(item.product.name)
print(f"Queries: {len(connection.queries)}")
# Expected: 3 queries (orders + items + products)
```

### Pagination Exercise

```python
# Exercise 28: OFFSET pagination এর problem দেখাও

import time

# ❌ OFFSET pagination — large offset এ slow
start = time.time()
orders = Order.objects.order_by('-created_at')[10000:10020]
list(orders)  # force evaluation
print(f"OFFSET: {time.time()-start:.3f}s")

# ✅ Keyset pagination implement করো
def get_orders_keyset(cursor_id=None, limit=20):
    qs = Order.objects.order_by('-id')
    if cursor_id:
        qs = qs.filter(id__lt=cursor_id)
    return qs[:limit]

start = time.time()
page = get_orders_keyset(cursor_id=10021)
list(page)
print(f"Keyset: {time.time()-start:.3f}s")
# Keyset always fast!
```

---

## 18.7 Module 8 Exercises — Transactions & Concurrency

```python
# Exercise 29: Atomic transaction practice

from django.db import transaction
from django.db.models import F

# Scenario: order place করো with inventory check
@transaction.atomic
def place_order(user_id, product_id, qty):
    # select_for_update দিয়ে lock নাও
    product = Product.objects.select_for_update().get(id=product_id)

    if product.stock < qty:
        raise ValueError(f"Insufficient stock: {product.stock} available")

    # Stock deduct করো
    Product.objects.filter(id=product_id).update(
        stock=F('stock') - qty
    )

    # Order create করো
    order = Order.objects.create(
        user_id=user_id,
        total=product.price * qty,
        status='pending'
    )
    OrderItem.objects.create(
        order=order,
        product=product,
        qty=qty,
        unit_price=product.price
    )
    return order

# Test করো:
try:
    order = place_order(user_id=1, product_id=2, qty=5)
    print(f"Order created: {order.id}")
except ValueError as e:
    print(f"Failed: {e}")

# Exercise 30: Deadlock simulate করো (2 threads)
import threading

def thread1():
    with transaction.atomic():
        p1 = Product.objects.select_for_update().get(id=1)
        time.sleep(1)
        p2 = Product.objects.select_for_update().get(id=2)

def thread2():
    with transaction.atomic():
        p2 = Product.objects.select_for_update().get(id=2)  # reverse order!
        time.sleep(1)
        p1 = Product.objects.select_for_update().get(id=1)

# ✅ Fix: consistent lock order
def thread_fixed():
    ids = sorted([1, 2])  # always ascending order
    with transaction.atomic():
        products = Product.objects.select_for_update().filter(
            id__in=ids
        ).order_by('id')
```

---

## 18.8 Module 9 Exercises — Django ORM Deep Dive

```python
# Exercise 31: annotate vs aggregate practice

from django.db.models import Count, Sum, Avg, Max, F, Q

# annotate: প্রতিটা object এ extra field যোগ করে
users_with_orders = User.objects.annotate(
    order_count=Count('orders'),
    total_spent=Sum('orders__total', filter=Q(orders__status='completed'))
).filter(order_count__gt=0)

for u in users_with_orders:
    print(f"{u.name}: {u.order_count} orders, spent {u.total_spent}")

# aggregate: single value return করে
stats = Order.objects.aggregate(
    total_orders=Count('id'),
    total_revenue=Sum('total'),
    avg_order=Avg('total'),
    max_order=Max('total')
)
print(stats)

# Exercise 32: Complex Q objects
from django.db.models import Q

# Bangladesh users অথবা 50000+ এর orders
results = Order.objects.filter(
    Q(user__country='Bangladesh') | Q(total__gte=50000)
).select_related('user')

# Completed এবং (Bangladesh অথবা India)
results = Order.objects.filter(
    Q(status='completed') &
    (Q(user__country='Bangladesh') | Q(user__country='India'))
)

# Exercise 33: bulk_create performance test
import time

# ❌ Slow: individual creates
start = time.time()
for i in range(1000):
    Review.objects.create(
        product_id=1, user_id=1, rating=5, comment=f'Review {i}'
    )
print(f"Individual: {time.time()-start:.2f}s")

# ✅ Fast: bulk_create
start = time.time()
Review.objects.bulk_create([
    Review(product_id=1, user_id=1, rating=5, comment=f'Review {i}')
    for i in range(1000)
], batch_size=500)
print(f"Bulk: {time.time()-start:.2f}s")
# Bulk is 10-50x faster!

# Exercise 34: Raw SQL when ORM is not enough
from django.db import connection

def get_product_sales_report():
    with connection.cursor() as cursor:
        cursor.execute("""
            SELECT
                p.name,
                COUNT(oi.id) AS times_ordered,
                SUM(oi.qty) AS total_units,
                SUM(oi.qty * oi.unit_price) AS revenue
            FROM products p
            LEFT JOIN order_items oi ON oi.product_id = p.id
            LEFT JOIN orders o ON oi.order_id = o.id AND o.status = 'completed'
            GROUP BY p.id, p.name
            ORDER BY revenue DESC NULLS LAST
        """)
        columns = [col[0] for col in cursor.description]
        return [dict(zip(columns, row)) for row in cursor.fetchall()]
```

---

## 18.9 Module 10 Exercises — PostgreSQL

```sql
-- Exercise 35: JSONB practice

ALTER TABLE products ADD COLUMN attributes JSONB;

UPDATE products SET attributes = '{"color": "black", "brand": "Samsung", "warranty_years": 1}'
WHERE id = 1;

UPDATE products SET attributes = '{"color": "silver", "brand": "Apple", "warranty_years": 2}'
WHERE id = 2;

-- Query JSONB
SELECT name, attributes->>'brand' AS brand
FROM products
WHERE attributes IS NOT NULL;

-- Filter by JSONB value
SELECT name FROM products
WHERE attributes->>'brand' = 'Samsung';

-- JSONB index
CREATE INDEX idx_products_brand ON products USING gin(attributes);

-- Contains operator
SELECT name FROM products
WHERE attributes @> '{"brand": "Apple"}';

-- Exercise 36: Materialized View তৈরি করো
CREATE MATERIALIZED VIEW product_sales_summary AS
SELECT
    p.id,
    p.name,
    COUNT(oi.id) AS order_count,
    COALESCE(SUM(oi.qty), 0) AS total_units_sold,
    COALESCE(SUM(oi.qty * oi.unit_price), 0) AS total_revenue,
    ROUND(AVG(r.rating), 2) AS avg_rating
FROM products p
LEFT JOIN order_items oi ON oi.product_id = p.id
LEFT JOIN orders o ON oi.order_id = o.id AND o.status = 'completed'
LEFT JOIN reviews r ON r.product_id = p.id
GROUP BY p.id, p.name;

CREATE UNIQUE INDEX ON product_sales_summary(id);

-- Query (fast!)
SELECT * FROM product_sales_summary ORDER BY total_revenue DESC;

-- Refresh (when data changes)
REFRESH MATERIALIZED VIEW CONCURRENTLY product_sales_summary;

-- Exercise 37: Partitioning practice
CREATE TABLE events (
    id         BIGSERIAL,
    user_id    BIGINT,
    event_type VARCHAR(50),
    data       JSONB,
    created_at TIMESTAMPTZ NOT NULL
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2024_01 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE events_2024_02 PARTITION OF events
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Insert
INSERT INTO events (user_id, event_type, data, created_at)
VALUES (1, 'page_view', '{"page": "/home"}', '2024-01-15');

-- Partition pruning verify করো
EXPLAIN SELECT * FROM events WHERE created_at >= '2024-01-01'
                               AND created_at <  '2024-02-01';
-- শুধু events_2024_01 scan হবে
```

---

## 18.10 Module 11 Exercises — Scaling

```python
# Exercise 38: Redis caching implement করো

import redis
import json
from django.conf import settings

cache = redis.Redis(host='localhost', port=6379, decode_responses=True)

def get_product_cached(product_id):
    key = f'product:{product_id}'
    data = cache.get(key)

    if data:
        print("Cache HIT")
        return json.loads(data)

    print("Cache MISS")
    product = Product.objects.select_related('category').get(id=product_id)
    payload = {
        'id': product.id,
        'name': product.name,
        'price': str(product.price),
        'category': product.category.name,
    }
    cache.setex(key, 300, json.dumps(payload))
    return payload

# Test:
print(get_product_cached(1))  # MISS
print(get_product_cached(1))  # HIT

# Exercise 39: Rate limiting with Redis
def is_rate_limited(user_id, limit=10, window=60):
    key = f'rate:{user_id}:{int(time.time() // window)}'
    count = cache.incr(key)
    if count == 1:
        cache.expire(key, window)
    return count > limit

# Test:
for i in range(12):
    if is_rate_limited(user_id=1):
        print(f"Request {i+1}: RATE LIMITED")
    else:
        print(f"Request {i+1}: OK")

# Exercise 40: Redis leaderboard
def update_leaderboard(user_id, score):
    cache.zadd('leaderboard', {f'user:{user_id}': score})

def get_top_10():
    return cache.zrevrange('leaderboard', 0, 9, withscores=True)

update_leaderboard(1, 9500)
update_leaderboard(2, 8200)
update_leaderboard(3, 9800)
print(get_top_10())
```

---

## 18.11 Mini Projects

### Mini Project 1 — E-commerce Backend (Django)

```
Build করো: Simple E-commerce API

Models:
  Category, Product, User, Order, OrderItem

Features:
  1. Product listing with filtering (category, price range)
  2. Product detail with average rating
  3. Place order (with inventory check — atomic)
  4. Order history for a user
  5. Sales report (top products by revenue)

Requirements:
  - select_related / prefetch_related সব জায়গায়
  - Redis cache product detail (5 min TTL)
  - Index on all foreign keys and filter columns
  - EXPLAIN ANALYZE করো প্রতিটা major query তে
  - bulk_create ব্যবহার করো order items create তে

Bonus:
  - Keyset pagination implement করো
  - Materialized view দিয়ে sales report করো
```

### Mini Project 2 — Banking Transaction System

```
Build করো: Simple Banking API

Models:
  Customer, Account, Transaction

Features:
  1. Create account
  2. Deposit money
  3. Withdraw money (balance check)
  4. Transfer between accounts (atomic, deadlock safe)
  5. Account statement (last 30 days)
  6. Balance verification (sum of transactions == balance)

Requirements:
  - All operations ACID compliant
  - Transactions table immutable (no UPDATE/DELETE)
  - select_for_update() for concurrent safety
  - Consistent lock order for transfer (prevent deadlock)
  - SERIALIZABLE isolation for transfer

Test:
  - Concurrent transfers (threading) — no race condition
  - Insufficient funds → rollback
  - Running balance always correct
```

### Mini Project 3 — Query Optimization Challenge

```
Problem: এই slow queries optimize করো

Setup:
  -- Large data generate করো
  INSERT INTO orders (user_id, status, total, created_at)
  SELECT
    (random() * 5 + 1)::INT,
    (ARRAY['pending','completed','cancelled'])[floor(random()*3+1)],
    (random() * 200000)::DECIMAL(12,2),
    NOW() - (random() * 365 * interval '1 day')
  FROM generate_series(1, 500000);

  INSERT INTO order_items (order_id, product_id, qty, unit_price)
  SELECT
    (random() * 500000 + 1)::INT,
    (random() * 8 + 1)::INT,
    (random() * 5 + 1)::INT,
    (random() * 200000)::DECIMAL(12,2)
  FROM generate_series(1, 1500000);

Slow Queries to Optimize:
  1. SELECT * FROM orders WHERE status = 'pending' ORDER BY created_at;
  2. SELECT * FROM orders o JOIN users u ON o.user_id = u.id
     WHERE u.country = 'Bangladesh';
  3. SELECT user_id, COUNT(*), SUM(total)
     FROM orders WHERE created_at > NOW() - INTERVAL '30 days'
     GROUP BY user_id HAVING COUNT(*) > 3;

For each query:
  Step 1: EXPLAIN ANALYZE দেখো (before)
  Step 2: Problem identify করো
  Step 3: Index তৈরি করো
  Step 4: EXPLAIN ANALYZE দেখো (after)
  Step 5: Improvement document করো
```

### Mini Project 4 — Real-time Dashboard

```
Build করো: Product Analytics Dashboard

Features:
  1. Real-time counters (page views, add-to-cart)
  2. Top 10 products today
  3. Revenue by hour (last 24 hours)
  4. Slow-moving inventory (stock > 50, no orders in 30 days)

Technical Requirements:
  - Redis: real-time counters (INCR)
  - PostgreSQL: historical data
  - Materialized view: hourly revenue (refresh every 15 min)
  - Celery task: flush Redis counters to DB every 5 minutes

Django Management Commands:
  python manage.py flush_redis_counters
  python manage.py refresh_dashboard_views
  python manage.py find_slow_moving_inventory
```

---

## 18.12 Debugging Exercises

```sql
-- Exercise 41: এই query টা slow কেন? Fix করো।
-- (500K orders table তে)
SELECT * FROM orders
WHERE DATE(created_at) = '2024-06-15';
-- Problem: DATE() function index ব্যবহার করতে পারে না!

-- Fix:
SELECT * FROM orders
WHERE created_at >= '2024-06-15'
  AND created_at <  '2024-06-16';
-- Range query — index ব্যবহার করবে।

-- Exercise 42: এই query তে কী সমস্যা?
SELECT * FROM products WHERE name LIKE '%phone%';
-- Problem: Leading wildcard — index skip হয়!
-- Fix: Full-text search ব্যবহার করো

CREATE INDEX idx_products_fts
ON products USING gin(to_tsvector('english', name));

SELECT * FROM products
WHERE to_tsvector('english', name) @@ plainto_tsquery('phone');

-- Exercise 43: এই JOIN slow কেন?
SELECT o.*, u.*
FROM orders o
JOIN users u ON UPPER(o.user_email) = UPPER(u.email);
-- Problem: Function on join column → index skip!
-- Fix: Store email in same case, use direct equality

-- Exercise 44: এই update কী সমস্যা করতে পারে?
UPDATE products SET price = price * 1.1;
-- Problem: No WHERE clause — সব rows update!
-- Table lock, very slow on large table।
-- Fix: Batch update with WHERE
UPDATE products SET price = price * 1.1
WHERE id BETWEEN 1 AND 1000;
-- Repeat in batches

-- Exercise 45: এই query inefficient কেন?
SELECT DISTINCT user_id FROM orders WHERE status = 'completed';
-- DISTINCT = sort + dedup = expensive
-- Fix: EXISTS অথবা GROUP BY
SELECT user_id FROM orders WHERE status = 'completed' GROUP BY user_id;
-- অথবা:
SELECT id FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id AND o.status = 'completed'
);
```

---

## 18.13 Interview Assignment Practice

### Assignment 1 (Junior Level — 2 hours)

```
Task: Design এবং implement করো একটা simple blog system।

Requirements:
  - Users can write posts
  - Posts have tags (many-to-many)
  - Comments on posts
  - Like posts

Deliverables:
  1. ER diagram (text format)
  2. SQL schema (CREATE TABLE statements)
  3. Django models
  4. Queries:
     a. Top 5 most liked posts this week
     b. Users with most posts (last 30 days)
     c. Posts with their comment count and like count
  5. Index strategy (justify each index)
```

### Assignment 2 (Mid Level — 4 hours)

```
Task: Optimize করো একটা slow Django API endpoint।

Given code:
  def get_order_report(request):
      orders = Order.objects.filter(status='completed')
      result = []
      for order in orders:
          items = []
          for item in order.orderitem_set.all():
              items.append({
                  'product': item.product.name,
                  'qty': item.qty,
                  'price': str(item.unit_price),
              })
          result.append({
              'order_id': order.id,
              'customer': order.user.name,
              'total': str(order.total),
              'items': items,
          })
      return JsonResponse({'orders': result})

Tasks:
  1. সমস্যাগুলো identify করো (কয়টা queries হচ্ছে?)
  2. Optimize করো (minimal queries)
  3. Redis cache যোগ করো (1 hour TTL)
  4. Before/after query count দেখাও
  5. EXPLAIN ANALYZE output দেখাও
```

### Assignment 3 (Senior Level — 1 day)

```
Task: Design করো একটা "Flash Sale System"।

Requirements:
  - Flash sale: 1000 units, 10 minutes, 50% discount
  - 100,000 concurrent users try to buy
  - Each user can buy max 1 unit
  - No overselling (exactly 1000 units sold)

Challenges to solve:
  1. Race condition — multiple users buying last item
  2. Performance — 100K concurrent requests
  3. Fairness — first come first served

Deliverables:
  1. Schema design
  2. Redis-based solution (fast path)
  3. PostgreSQL fallback (reliable path)
  4. Django implementation
  5. How to test concurrency (threading/locust)
  6. Trade-offs discussion
```

---

## 18.14 Complete Roadmap — Expert Database Engineer

```
ROADMAP: Junior → Senior Database Engineer
===========================================

PHASE 1: Foundation (Month 1-2)
  ✅ SQL Fundamentals (SELECT, JOIN, GROUP BY, HAVING)
  ✅ ACID properties
  ✅ Basic normalization (1NF, 2NF, 3NF)
  ✅ Primary/Foreign keys, constraints
  ✅ Django ORM basics (filter, exclude, get)
  
  Practice: LeetCode Easy SQL problems (20 problems)
  Project: Simple blog database

PHASE 2: Intermediate (Month 3-4)
  ✅ Advanced SQL (CTEs, Window Functions, Subqueries)
  ✅ Index types and strategy
  ✅ EXPLAIN ANALYZE reading
  ✅ N+1 problem and solutions
  ✅ Django ORM: select_related, prefetch_related, annotate, F()
  ✅ Basic transactions and isolation levels
  
  Practice: LeetCode Medium SQL problems (20 problems)
  Project: E-commerce backend with optimized queries

PHASE 3: Advanced (Month 5-6)
  ✅ PostgreSQL internals (MVCC, VACUUM, WAL)
  ✅ Partitioning and sharding
  ✅ Connection pooling (pgBouncer)
  ✅ Redis integration (cache, queues, leaderboards)
  ✅ Deadlock prevention
  ✅ Optimistic vs pessimistic locking
  
  Practice: LeetCode Hard SQL problems (10 problems)
  Project: Banking system with concurrent transactions

PHASE 4: Production Level (Month 7-8)
  ✅ Backup and recovery (PITR, WAL-G)
  ✅ Replication and failover (Patroni)
  ✅ Zero-downtime migrations
  ✅ Database security (SQL injection, roles, encryption)
  ✅ Monitoring (pg_stat_statements, slow query log)
  ✅ NoSQL (MongoDB, Cassandra when to use)
  
  Practice: Real production incident simulations
  Project: Full system with HA setup

PHASE 5: Senior/Architect Level (Month 9-12)
  ✅ System design with database decisions
  ✅ Polyglot persistence
  ✅ Scaling strategies (when to shard)
  ✅ Real-world case studies (design Uber, Twitter)
  ✅ Database evolution and migration strategies
  ✅ Interview preparation (all levels)
  
  Practice: Mock system design interviews
  Project: Design and document a production system

RESOURCES:
  Books:
    "Designing Data-Intensive Applications" — Martin Kleppmann (must read!)
    "PostgreSQL: Up and Running" — Regina Obe
    "High Performance MySQL" — Baron Schwartz
    "Database Internals" — Alex Petrov

  Online:
    use-the-index-luke.com (indexing bible)
    pganalyze.com/blog (PostgreSQL tips)
    LeetCode Database section
    SQLZoo.net

  Tools to master:
    psql (PostgreSQL CLI)
    pgAdmin / DBeaver (GUI)
    pgBouncer (connection pooling)
    WAL-G (backup)
    Patroni (HA)
    Redis CLI

DAILY HABITS:
  Read one pg_stat_statements slow query and optimize it
  Review EXPLAIN ANALYZE of your slowest endpoint
  Check index usage weekly (unused indexes?)
  Monitor replication lag
  Test backup restore monthly
```

---

## 18.15 Module Summary & Cheat Sheet

```
PRACTICE CHEAT SHEET
======================

SQL PRACTICE ORDER:
  1. Basic SELECT, WHERE, ORDER BY
  2. JOINs (INNER, LEFT, RIGHT, SELF)
  3. GROUP BY, HAVING, aggregate functions
  4. Subqueries, EXISTS, IN
  5. CTEs, Recursive CTEs
  6. Window Functions (ROW_NUMBER, RANK, LAG/LEAD)

ORM PRACTICE ORDER:
  1. filter(), exclude(), get(), all()
  2. select_related(), prefetch_related()
  3. annotate(), aggregate()
  4. F() expressions, Q() objects
  5. bulk_create(), bulk_update()
  6. select_for_update(), transaction.atomic()
  7. Raw SQL for complex cases

OPTIMIZATION CHECKLIST:
  □ EXPLAIN ANALYZE দেখো
  □ Seq Scan on large table → add index
  □ Function on WHERE column → rewrite
  □ N+1 detected → select_related/prefetch_related
  □ OFFSET large → keyset pagination
  □ Repeated expensive query → Redis cache
  □ Aggregate on large table → materialized view

INDEX CHECKLIST:
  □ Foreign key columns → always index
  □ WHERE clause columns → index
  □ ORDER BY columns → index
  □ Composite: most selective first
  □ Partial index for frequent subset
  □ CONCURRENTLY for production

TRANSACTION CHECKLIST:
  □ Money operations → SERIALIZABLE
  □ Inventory → select_for_update
  □ Multiple rows → consistent lock order
  □ Long transactions → avoid
  □ Deadlock → retry logic

DJANGO ORM RULES:
  □ Never loop with ORM queries
  □ select_related for FK/O2O
  □ prefetch_related for M2M/reverse FK
  □ F() for atomic counter updates
  □ bulk_create for batch inserts
  □ only() / defer() for large models

MINI PROJECT PRIORITY:
  1. E-commerce (covers most concepts)
  2. Banking (covers transactions, ACID)
  3. Query optimization challenge
  4. Real-time dashboard (covers Redis)
```
