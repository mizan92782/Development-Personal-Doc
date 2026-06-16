# MODULE 7: Query Optimization (কোয়েরি অপ্টিমাইজেশন)
## বাংলায় সম্পূর্ণ Query Optimization গাইড — Production Level

> **কেন এই module জীবন বদলে দেবে?**
> Junior developer রা feature লেখে। Senior developer রা feature লেখে এবং সেটা 10 million users এর জন্য fast রাখে।
> Query optimization জানলে তুমি production incident এ hero হবে।
> Amazon, Google, Netflix — সবার backend engineer দের interview এ এই topic mandatory।

---

## 7.1 EXPLAIN এবং EXPLAIN ANALYZE

### EXPLAIN কী?

EXPLAIN তোমাকে বলে database তোমার query কীভাবে execute করবে — কিন্তু actually execute করে না।
EXPLAIN ANALYZE আসলেই execute করে এবং real time দেখায়।

```
EXPLAIN       → Query plan দেখায় (estimated)
EXPLAIN ANALYZE → Query execute করে real stats দেখায়
EXPLAIN (ANALYZE, BUFFERS) → Memory/disk I/O সহ দেখায়
```

### Basic Usage

```sql
-- Simple EXPLAIN
EXPLAIN SELECT * FROM orders WHERE customer_id = 1234;

-- Output:
-- Index Scan using idx_orders_customer on orders
--   (cost=0.56..125.34 rows=45 width=96)
--   Index Cond: (customer_id = 1234)

-- Full EXPLAIN ANALYZE
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.id, c.name, o.total_amount
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.status = 'pending'
  AND o.created_at >= NOW() - INTERVAL '7 days'
ORDER BY o.created_at DESC
LIMIT 20;
```

### EXPLAIN Output পড়ার সম্পূর্ণ গাইড

```
Seq Scan on orders
  (cost=0.00..98765.43 rows=500000 width=96)
  (actual time=0.023..2345.678 rows=500000 loops=1)
  Buffers: shared hit=1234 read=56789
  
┌─────────────────────────────────────────────────────────────────┐
│ cost=0.00..98765.43                                             │
│   ├── 0.00    = startup cost (first row পাওয়ার আগে খরচ)        │
│   └── 98765.43 = total cost (সব rows পাওয়ার মোট খরচ)          │
│                                                                 │
│ rows=500000  = estimated rows (statistics থেকে)                │
│ width=96     = average row size in bytes                        │
│                                                                 │
│ actual time=0.023..2345.678                                     │
│   ├── 0.023    = actual startup time (ms)                       │
│   └── 2345.678 = actual total time (ms) ← এটাই আসল সময়       │
│                                                                 │
│ Buffers: shared hit=1234  → cache থেকে পড়েছে (fast)          │
│ Buffers: shared read=56789 → disk থেকে পড়েছে (slow)          │
└─────────────────────────────────────────────────────────────────┘
```

### Scan Types — কোনটা কখন আসে

```
┌─────────────────────┬──────────────────────────────────────────┐
│ Scan Type           │ মানে কী                                  │
├─────────────────────┼──────────────────────────────────────────┤
│ Seq Scan            │ Full table scan — index নেই বা useless   │
│ Index Scan          │ Index ব্যবহার করছে                       │
│ Index Only Scan     │ Covering index — table এ যাচ্ছে না      │
│ Bitmap Index Scan   │ Multiple index combine করছে             │
│ Bitmap Heap Scan    │ Bitmap scan এর পরে table read           │
└─────────────────────┴──────────────────────────────────────────┘

Join Types:
┌─────────────────────┬──────────────────────────────────────────┐
│ Nested Loop         │ Small table JOIN — একটা একটা row match  │
│ Hash Join           │ Large table JOIN — hash table তৈরি করে │
│ Merge Join          │ দুটো sorted set JOIN — efficient        │
└─────────────────────┴──────────────────────────────────────────┘
```

### Django ORM এ EXPLAIN

```python
# Django 3.2+
queryset = Order.objects.filter(
    status='pending',
    created_at__gte=timezone.now() - timedelta(days=7)
).select_related('customer')

# EXPLAIN দেখো
print(queryset.explain())

# EXPLAIN ANALYZE দেখো
print(queryset.explain(analyze=True, buffers=True))
```

---

## 7.2 Query Execution Plan বোঝো

### একটা Complex Query এর Plan

```sql
EXPLAIN ANALYZE
SELECT
    c.name,
    COUNT(o.id)      AS order_count,
    SUM(o.total_amount) AS total_spent
FROM customers c
JOIN orders o ON o.customer_id = c.id
WHERE o.created_at >= '2024-01-01'
  AND o.status = 'delivered'
GROUP BY c.id, c.name
ORDER BY total_spent DESC
LIMIT 10;
```

```
Execution Plan (bottom to top পড়তে হয়):

Limit (rows=10)
  └── Sort (cost=...) [total_spent DESC]
        └── HashAggregate (GROUP BY c.id, c.name)
              └── Hash Join (o.customer_id = c.id)
                    ├── Seq Scan on customers
                    └── Bitmap Heap Scan on orders
                          └── Bitmap Index Scan on idx_orders_status_date
                                Index Cond: status='delivered' AND created_at >= '2024-01-01'

Bottom to top পড়ার নিয়ম:
1. Bitmap Index Scan → index দিয়ে matching rows খোঁজো
2. Bitmap Heap Scan  → actual rows table থেকে পড়ো
3. Hash Join         → customers এর সাথে join করো
4. HashAggregate     → GROUP BY করো
5. Sort              → ORDER BY করো
6. Limit             → TOP 10 নাও
```

### Cost Estimation কীভাবে কাজ করে

```sql
-- PostgreSQL statistics দেখো
SELECT
    tablename,
    attname,
    n_distinct,     -- unique values এর estimate
    correlation     -- physical vs logical order (1.0 = perfectly sorted)
FROM pg_stats
WHERE tablename = 'orders';

-- Table statistics
SELECT
    relname,
    reltuples,      -- estimated row count
    relpages        -- disk pages
FROM pg_class
WHERE relname = 'orders';

-- Statistics update করো
ANALYZE orders;
ANALYZE customers;
```

---

## 7.3 N+1 Problem — সবচেয়ে গুরুত্বপূর্ণ Topic

### N+1 কী?

```
N+1 Problem মানে:
- 1টা query চালিয়ে N টা record পাওয়া
- তারপর সেই N টা record এর প্রতিটার জন্য আরেকটা করে query চালানো
- মোট: 1 + N = N+1 queries

Example:
- 100 orders আছে → 1 query
- প্রতিটা order এর customer নাম দেখাতে → 100 queries
- মোট: 101 queries!

10,000 orders হলে: 10,001 queries → database die করবে!
```

### N+1 এর Real Example

```python
# ❌ N+1 Problem — এটা দেখতে innocent কিন্তু ভয়ঙ্কর
def get_order_list(request):
    orders = Order.objects.filter(status='pending')  # Query 1
    
    result = []
    for order in orders:
        result.append({
            'id': order.id,
            'customer': order.customer.name,  # Query 2, 3, 4... N+1!
            'total': order.total_amount
        })
    return result

# যদি 500 pending orders থাকে:
# Query 1: SELECT * FROM orders WHERE status='pending'
# Query 2: SELECT * FROM customers WHERE id=1
# Query 3: SELECT * FROM customers WHERE id=2
# ...
# Query 501: SELECT * FROM customers WHERE id=500
# মোট: 501 queries!
```

```sql
-- Database এ এই queries যাচ্ছে:
SELECT * FROM orders WHERE status = 'pending';
SELECT * FROM customers WHERE id = 1;
SELECT * FROM customers WHERE id = 2;
SELECT * FROM customers WHERE id = 3;
-- ... 500 বার!
```

### N+1 Fix — select_related (INNER JOIN)

```python
# ✅ Fix: select_related — একটাই JOIN query
def get_order_list(request):
    orders = Order.objects.filter(
        status='pending'
    ).select_related('customer')  # JOIN করে আনো
    
    result = []
    for order in orders:
        result.append({
            'id': order.id,
            'customer': order.customer.name,  # No extra query!
            'total': order.total_amount
        })
    return result
```

```sql
-- Generated SQL — মাত্র 1টা query!
SELECT
    orders.id,
    orders.status,
    orders.total_amount,
    orders.customer_id,
    customers.id,
    customers.name,
    customers.email
FROM orders
INNER JOIN customers ON customers.id = orders.customer_id
WHERE orders.status = 'pending';
```

### N+1 Fix — prefetch_related (Separate Query + Python JOIN)

```python
# ✅ prefetch_related — ManyToMany বা reverse FK এর জন্য
def get_order_details(request):
    orders = Order.objects.filter(
        status='pending'
    ).prefetch_related('items__product')  # items আর products prefetch করো
    
    for order in orders:
        for item in order.items.all():  # No extra query!
            print(item.product.name)   # No extra query!
```

```sql
-- Generated SQL — মাত্র 3টা query (N+1 নয়!)

-- Query 1: orders
SELECT * FROM orders WHERE status = 'pending';

-- Query 2: সব order এর items একসাথে
SELECT * FROM order_items
WHERE order_id IN (1, 2, 3, 4, 5, ...all order ids...);

-- Query 3: সব items এর products একসাথে
SELECT * FROM products
WHERE id IN (101, 102, 103, ...all product ids...);

-- Python মেমোরিতে join করে দেয়
```

### select_related vs prefetch_related

```
┌──────────────────────┬──────────────────────┬──────────────────────┐
│ Feature              │ select_related        │ prefetch_related      │
├──────────────────────┼──────────────────────┼──────────────────────┤
│ SQL mechanism        │ JOIN                  │ Separate queries      │
│ Relation type        │ ForeignKey, OneToOne  │ ManyToMany, reverse FK│
│ Depth                │ Nested JOIN           │ Nested prefetch       │
│ Filtering prefetch   │ না                   │ Prefetch() object দিয়ে│
│ Memory usage         │ কম (JOIN result)      │ বেশি (Python মেমোরিতে)│
│ Query count          │ 1                     │ 1 + related table count│
└──────────────────────┴──────────────────────┴──────────────────────┘
```

### Advanced N+1 — Nested Prefetch

```python
from django.db.models import Prefetch

# Deep nested prefetch
orders = Order.objects.filter(
    status='pending'
).select_related(
    'customer'                    # ForeignKey — JOIN
).prefetch_related(
    Prefetch(
        'items',
        queryset=OrderItem.objects.select_related('product__category')
        .filter(quantity__gt=0),  # prefetch এ filter করা যায়!
        to_attr='active_items'    # আলাদা attribute এ store করো
    )
)

for order in orders:
    print(order.customer.name)
    for item in order.active_items:  # filtered prefetch
        print(item.product.name)
        print(item.product.category.name)
```

```sql
-- Generated SQL:
-- Query 1: orders JOIN customers
-- Query 2: order_items WHERE order_id IN (...) AND quantity > 0
-- Query 3: products JOIN categories WHERE id IN (...)
-- মোট: 3 queries — N+1 নয়!
```

### N+1 Detect করার Tool

```python
# django-silk বা django-debug-toolbar install করো
# development এ query count দেখো

# Manual detection
from django.db import connection, reset_queries
import django
django.setup()

reset_queries()

# তোমার code এখানে চালাও
orders = list(Order.objects.filter(status='pending'))
for o in orders:
    _ = o.customer.name

query_count = len(connection.queries)
print(f"Total queries: {query_count}")  # N+1 detect হবে

# nplusone library দিয়ে auto-detect
# pip install nplusone
from nplusone.ext.django import NPlusOneMiddleware
# settings.py তে MIDDLEWARE তে যোগ করো
```

### N+1 — REST API এর Real Example

```python
# ❌ DRF Serializer এ N+1
class OrderSerializer(serializers.ModelSerializer):
    customer_name = serializers.SerializerMethodField()
    
    def get_customer_name(self, obj):
        return obj.customer.name  # প্রতিটা object এ query!
    
    class Meta:
        model = Order
        fields = ['id', 'customer_name', 'total_amount']

class OrderListView(generics.ListAPIView):
    serializer_class = OrderSerializer
    
    def get_queryset(self):
        return Order.objects.filter(status='pending')  # N+1 হবে!

# ✅ Fix
class OrderListView(generics.ListAPIView):
    serializer_class = OrderSerializer
    
    def get_queryset(self):
        return Order.objects.filter(
            status='pending'
        ).select_related('customer')  # Fix!
```

---

## 7.4 Query Rewriting (কোয়েরি রিরাইটিং)

### Technique 1: Subquery → JOIN

```sql
-- ❌ Slow subquery
SELECT * FROM products
WHERE category_id IN (
    SELECT id FROM categories WHERE name = 'Electronics'
);

-- ✅ Faster JOIN
SELECT p.* FROM products p
JOIN categories c ON c.id = p.category_id
WHERE c.name = 'Electronics';
```

### Technique 2: Correlated Subquery → JOIN + GROUP BY

```sql
-- ❌ Correlated subquery — প্রতিটা row এর জন্য subquery চলে
SELECT
    c.id,
    c.name,
    (SELECT COUNT(*) FROM orders WHERE customer_id = c.id) AS order_count
FROM customers c;

-- ✅ Rewrite with JOIN
SELECT
    c.id,
    c.name,
    COUNT(o.id) AS order_count
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
GROUP BY c.id, c.name;
```

### Technique 3: OR → UNION

```sql
-- ❌ OR — index ব্যবহার নাও হতে পারে
SELECT * FROM users
WHERE email = 'a@x.com' OR phone = '01711000001';

-- ✅ UNION — দুটো index ব্যবহার করবে
SELECT * FROM users WHERE email = 'a@x.com'
UNION
SELECT * FROM users WHERE phone = '01711000001';
```

### Technique 4: NOT IN → NOT EXISTS

```sql
-- ❌ NOT IN — NULL এ bug আছে এবং slower
SELECT * FROM customers
WHERE id NOT IN (SELECT customer_id FROM orders);
-- Danger: orders এ customer_id NULL থাকলে সব rows miss হবে!

-- ✅ NOT EXISTS — সঠিক এবং faster
SELECT * FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.id
);
```

### Technique 5: DISTINCT → GROUP BY

```sql
-- ❌ DISTINCT — sort করে deduplicate
SELECT DISTINCT customer_id FROM orders;

-- ✅ GROUP BY — index ব্যবহারে better
SELECT customer_id FROM orders GROUP BY customer_id;
```

---

## 7.5 Pagination Optimization (পেজিনেশন অপ্টিমাইজেশন)

### সমস্যা — OFFSET Pagination

```sql
-- ❌ OFFSET pagination — page যত বাড়ে তত slow!
SELECT * FROM products ORDER BY id LIMIT 20 OFFSET 0;     -- Fast
SELECT * FROM products ORDER BY id LIMIT 20 OFFSET 100;   -- OK
SELECT * FROM products ORDER BY id LIMIT 20 OFFSET 10000; -- Slow
SELECT * FROM products ORDER BY id LIMIT 20 OFFSET 500000;-- Very Slow!

-- কেন slow?
-- OFFSET 500000 মানে: 500020 rows পড়ো, প্রথম 500000 টা ফেলে দাও
-- = 500020 rows waste!
```

### ✅ Solution — Keyset / Cursor Pagination

```sql
-- Keyset pagination — সবসময় fast!
-- প্রথম page
SELECT * FROM products ORDER BY id LIMIT 20;
-- Last id: 20

-- পরের page — id > 20
SELECT * FROM products WHERE id > 20 ORDER BY id LIMIT 20;
-- Last id: 40

-- যেকোনো page — সবসময় O(log n) via index!
SELECT * FROM products WHERE id > :last_id ORDER BY id LIMIT 20;
```

```python
# Django ORM — Cursor Pagination

# ❌ OFFSET pagination (slow for large pages)
def get_products_offset(page, page_size=20):
    offset = (page - 1) * page_size
    return Product.objects.order_by('id')[offset:offset + page_size]

# Generated SQL:
# SELECT * FROM products ORDER BY id LIMIT 20 OFFSET 100000; -- slow!

# ✅ Cursor pagination (always fast)
def get_products_cursor(last_id=None, page_size=20):
    qs = Product.objects.order_by('id')
    if last_id:
        qs = qs.filter(id__gt=last_id)
    return qs[:page_size]

# Generated SQL:
# SELECT * FROM products WHERE id > 100000 ORDER BY id LIMIT 20;
# index ব্যবহার করবে — always fast!
```

### DRF এ Cursor Pagination

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.CursorPagination',
    'PAGE_SIZE': 20,
}

class ProductCursorPagination(CursorPagination):
    page_size = 20
    ordering  = '-created_at'  # index থাকতে হবে এই column এ

class ProductListView(generics.ListAPIView):
    queryset         = Product.objects.all()
    pagination_class = ProductCursorPagination
```

### Count Query Optimization

```sql
-- ❌ Slow count for large tables
SELECT COUNT(*) FROM orders WHERE status = 'pending';
-- Full table scan বা index scan — large table এ slow

-- ✅ Approximate count (analytics এ যথেষ্ট)
SELECT reltuples::BIGINT AS estimated_count
FROM pg_class
WHERE relname = 'orders';

-- ✅ Cache the count (application level)
-- Redis এ total count cache করো, TTL = 60 seconds
```

---

## 7.6 Bulk Operations (বাল্ক অপারেশন)

### সমস্যা — Row-by-Row Insert

```python
# ❌ প্রতিটা row আলাদা INSERT — ভয়ঙ্কর slow
for product_data in product_list:  # 10,000 products
    Product.objects.create(**product_data)
# 10,000 INSERT queries!
# প্রতিটাতে: network round trip + DB overhead
# মোট সময়: ~30 seconds

# ✅ bulk_create — একটাই query!
products = [Product(**data) for data in product_list]
Product.objects.bulk_create(products, batch_size=1000)
# মোট সময়: ~0.5 seconds — 60x faster!
```

```sql
-- Generated SQL by bulk_create:
INSERT INTO products (name, price, category_id)
VALUES
    ('Product 1', 100, 1),
    ('Product 2', 200, 2),
    ('Product 3', 150, 1),
    -- ... 1000 rows at once
;
```

### bulk_create Advanced Options

```python
# ignore_conflicts — duplicate হলে skip করো
Product.objects.bulk_create(
    products,
    ignore_conflicts=True
)

# update_conflicts — duplicate হলে update করো (Django 4.1+)
Product.objects.bulk_create(
    products,
    update_conflicts=True,
    unique_fields=['sku'],
    update_fields=['price', 'stock_qty']
)

# Generated SQL:
# INSERT INTO products (sku, price, stock_qty)
# VALUES (...)
# ON CONFLICT (sku) DO UPDATE
# SET price = EXCLUDED.price, stock_qty = EXCLUDED.stock_qty;
```

### bulk_update

```python
# ❌ Row-by-row update
for product in products:
    product.price *= 1.1
    product.save()  # N queries!

# ✅ bulk_update
for product in products:
    product.price = product.price * Decimal('1.1')

Product.objects.bulk_update(products, ['price'], batch_size=500)
# Generated SQL:
# UPDATE products SET price = CASE
#   WHEN id = 1 THEN 110.00
#   WHEN id = 2 THEN 220.00
#   ...
# END WHERE id IN (1, 2, ...);
```

### F() Expression — Atomic Update

```python
from django.db.models import F

# ❌ Race condition আছে
product = Product.objects.get(id=1)
product.stock_qty -= 1
product.save()
# দুটো request একসাথে আসলে: দুজনেই stock=10 পাবে, দুজনেই 9 করবে!

# ✅ F() expression — database level atomic
Product.objects.filter(id=1).update(stock_qty=F('stock_qty') - 1)
# Generated SQL:
# UPDATE products SET stock_qty = stock_qty - 1 WHERE id = 1;
# Database level atomic — race condition নেই!

# Multiple field update
Product.objects.filter(category_id=5).update(
    price=F('price') * Decimal('1.1'),
    updated_at=timezone.now()
)
```

---

## 7.7 Batch Processing (ব্যাচ প্রসেসিং)

### Large Dataset Process করা

```python
# ❌ সব records একসাথে load — memory overflow!
all_orders = list(Order.objects.filter(status='pending'))
# 1 million orders = RAM শেষ!

# ✅ iterator() — cursor দিয়ে batch এ পড়ো
for order in Order.objects.filter(status='pending').iterator(chunk_size=1000):
    process_order(order)
# মেমোরিতে একসাথে শুধু 1000 rows

# ✅ Batch processing with pagination
def process_in_batches(queryset, batch_size=1000):
    last_id = 0
    while True:
        batch = list(
            queryset.filter(id__gt=last_id)
            .order_by('id')[:batch_size]
        )
        if not batch:
            break
        
        for obj in batch:
            process(obj)
        
        last_id = batch[-1].id
        print(f"Processed up to id: {last_id}")

# Usage
process_in_batches(Order.objects.filter(status='pending'))
```

### Celery দিয়ে Async Batch

```python
# tasks.py
from celery import shared_task

@shared_task
def process_pending_orders_batch(start_id, end_id):
    orders = Order.objects.filter(
        status='pending',
        id__range=(start_id, end_id)
    ).select_related('customer')
    
    for order in orders:
        send_confirmation_email(order)

# Dispatch batches
def dispatch_order_processing():
    max_id = Order.objects.filter(status='pending').order_by('-id').values_list('id', flat=True).first()
    
    batch_size = 1000
    for start in range(0, max_id, batch_size):
        process_pending_orders_batch.delay(start, start + batch_size)
```

---

## 7.8 Real Production Examples

### Example 1 — Uber Surge Pricing Query

```sql
-- ❌ Slow: প্রতিটা request এ এই heavy query
SELECT
    zone_id,
    COUNT(DISTINCT driver_id) AS available_drivers,
    COUNT(DISTINCT ride_id)   AS active_requests
FROM (
    SELECT zone_id, driver_id, NULL AS ride_id
    FROM drivers WHERE status = 'available'
    UNION ALL
    SELECT zone_id, NULL, ride_id
    FROM ride_requests WHERE status = 'waiting'
) combined
GROUP BY zone_id;

-- ✅ Production fix: Materialized view + refresh every 30 seconds
CREATE MATERIALIZED VIEW surge_pricing_data AS
SELECT
    zone_id,
    COUNT(DISTINCT driver_id) AS available_drivers,
    COUNT(DISTINCT ride_id)   AS active_requests,
    NOW() AS refreshed_at
FROM ...
GROUP BY zone_id;

CREATE UNIQUE INDEX ON surge_pricing_data(zone_id);

-- Refresh every 30 seconds (cron/Celery)
REFRESH MATERIALIZED VIEW CONCURRENTLY surge_pricing_data;

-- Fast read:
SELECT * FROM surge_pricing_data WHERE zone_id = 5;
-- 1ms!
```

### Example 2 — Amazon Product Search

```python
# ❌ Slow search
products = Product.objects.filter(
    name__icontains='wireless headphone'
)
# Generated: WHERE name ILIKE '%wireless headphone%'
# Full table scan!

# ✅ Full-text search with GIN index
from django.contrib.postgres.search import SearchVector, SearchQuery, SearchRank

products = Product.objects.annotate(
    search=SearchVector('name', 'description', 'brand'),
    rank=SearchRank(SearchVector('name', 'description', 'brand'),
                    SearchQuery('wireless headphone'))
).filter(
    search=SearchQuery('wireless headphone')
).order_by('-rank')[:20]
```

### Example 3 — Netflix Watch History

```python
# ❌ N+1 in watch history API
class WatchHistoryView(generics.ListAPIView):
    def get_queryset(self):
        return WatchHistory.objects.filter(
            user=self.request.user
        ).order_by('-watched_at')[:50]
    
    # Serializer এ show.title, show.thumbnail access করলে N+1!

# ✅ Optimized
class WatchHistoryView(generics.ListAPIView):
    def get_queryset(self):
        return WatchHistory.objects.filter(
            user=self.request.user
        ).select_related(
            'show',
            'show__genre'
        ).prefetch_related(
            'show__cast_members'
        ).order_by('-watched_at')[:50]
```

---

## 7.9 Common Mistakes

### ভুল ১: QuerySet Evaluate না বুঝে

```python
# Django QuerySet lazy — এখনো query হয়নি
orders = Order.objects.filter(status='pending')  # No SQL yet!

# এখন evaluate হবে:
count = orders.count()    # SELECT COUNT(*)
list_orders = list(orders)  # SELECT *
for o in orders:            # SELECT *
    pass

# ❌ Double evaluation!
count = orders.count()    # Query 1
items = list(orders)       # Query 2
# একই queryset দুইবার database hit!

# ✅ একবার evaluate করো
items = list(orders)       # Query 1
count = len(items)         # No query — Python len()
```

### ভুল ২: values() ছাড়া Unnecessary Columns

```python
# ❌ সব columns fetch করছো কিন্তু দরকার মাত্র 2টা
orders = Order.objects.filter(status='pending')
for o in orders:
    print(o.id, o.total_amount)

# ✅ values() বা only() দিয়ে দরকারি columns নাও
orders = Order.objects.filter(
    status='pending'
).values('id', 'total_amount')

# Generated SQL:
# SELECT id, total_amount FROM orders WHERE status = 'pending';
# SELECT * এর বদলে শুধু দরকারি columns!
```

### ভুল ৩: exists() না জেনে if queryset ব্যবহার

```python
# ❌ Slow — সব rows load করে
if Order.objects.filter(customer_id=1).count() > 0:
    do_something()

# ❌ আরো slow — সব rows load করে Python এ check
if Order.objects.filter(customer_id=1):
    do_something()

# ✅ Fast — শুধু একটা row check করে
if Order.objects.filter(customer_id=1).exists():
    do_something()
# Generated SQL: SELECT 1 FROM orders WHERE customer_id=1 LIMIT 1;
```

---

## 7.10 Optimization Strategies Summary

```
Query Optimization Checklist:
==============================

1. N+1 check করো
   □ select_related (FK, OneToOne)
   □ prefetch_related (M2M, reverse FK)
   □ django-debug-toolbar দিয়ে query count দেখো

2. EXPLAIN ANALYZE চালাও
   □ Seq Scan আছে? → Index দরকার
   □ High cost node? → Optimize করো
   □ Estimated vs Actual rows বিশাল পার্থক্য? → ANALYZE চালাও

3. Pagination
   □ OFFSET ব্যবহার করছো? → Cursor pagination এ migrate করো

4. Bulk Operations
   □ Loop এ create/update? → bulk_create/bulk_update ব্যবহার করো

5. Field selection
   □ values() বা only() দিয়ে দরকারি columns নাও

6. Aggregation
   □ Python এ loop এ calculate না করে DB level এ করো
   □ annotate(), aggregate(), Count(), Sum() ব্যবহার করো

7. Caching
   □ Expensive query? → Redis এ cache করো
   □ Materialized view ব্যবহার করো heavy analytics এ
```

---

## 7.11 Production Best Practices

```
1.  Development এ django-debug-toolbar install করো সবসময়
2.  Query count threshold set করো — 10+ queries হলে alert
3.  Slow query log enable করো (log_min_duration_statement = 500ms)
4.  pg_stat_statements দিয়ে top slow queries monitor করো
5.  Bulk operations এ batch_size দাও (500-1000)
6.  Pagination এ সবসময় cursor-based approach নাও (large datasets)
7.  Analytics query গুলো read replica তে পাঠাও
8.  Heavy query গুলো Celery task এ async করো
9.  Materialized view ব্যবহার করো expensive aggregation এ
10. Production এ query change করার আগে staging এ EXPLAIN দেখো
```

---

## 7.12 Senior Engineer Insights

> **War Story — Instagram:**
> Instagram এর feed API তে একটা endpoint ছিল যেটা 200ms নিচ্ছিল।
> EXPLAIN ANALYZE দেখে বোঝা গেল 47টা query হচ্ছে — classic N+1।
> select_related আর prefetch_related যোগ করার পর 3টা query এ নামে।
> Response time: 200ms → 12ms। User experience dramatically improved।

> **Amazon এর Rule:**
> "Every database query must be justified." মানে — কোনো query unnecessary হবে না।
> Code review এ query count দেখো। N+1 থাকলে PR reject।

> **Design Decision:**
> Read-heavy API তে: QuerySet caching (Redis) + select_related + values()
> Write-heavy API তে: bulk_create + F() expressions + async processing

---

## 7.13 Interview Questions

### Beginner Level

**Q1: N+1 problem কী?**
> N+1 হলো যখন একটা query দিয়ে N records আনা হয় এবং তারপর প্রতিটা record এর জন্য আলাদা query করা হয়। মোট N+1 queries। Solution: select_related (FK) এবং prefetch_related (M2M)।

**Q2: EXPLAIN এবং EXPLAIN ANALYZE এর পার্থক্য কী?**
> EXPLAIN শুধু query plan দেখায়, execute করে না। EXPLAIN ANALYZE আসলেই execute করে real time এবং row count দেখায়।

**Q3: Seq Scan মানে কী?**
> Full table scan — প্রতিটা row পড়া হচ্ছে। Large table এ এটা slow। Index থাকলে এড়ানো যায়।

---

### Intermediate Level

**Q4: select_related এবং prefetch_related কখন কোনটা ব্যবহার করবে?**
> select_related: ForeignKey এবং OneToOne relation এ — SQL JOIN করে।
> prefetch_related: ManyToMany এবং reverse FK এ — আলাদা query করে Python এ merge করে।
> Rule: "Forward FK → select_related, Backward/M2M → prefetch_related"

**Q5: Cursor pagination কেন OFFSET এর চেয়ে better?**
> OFFSET N মানে N rows skip করা — database N rows পড়েই ফেলে দেয়। N বাড়লে আরো slow।
> Cursor pagination WHERE id > last_id ব্যবহার করে — index দিয়ে directly সেই point থেকে শুরু করে। সবসময় O(log n)।

**Q6: F() expression কেন দরকার?**
> F() expression database level এ atomic update করে। Python এ read-then-write করলে race condition আসে। F() দিয়ে single SQL UPDATE query হয় যা atomic।

---

### Senior / FAANG Level

**Q7: 10 million orders আছে। প্রতিদিন রাতে সব pending orders process করতে হবে। কীভাবে করবে?**

```python
# Strategy:
# 1. Iterator + batch processing
# 2. Celery এ async
# 3. Database index ব্যবহার

@shared_task
def process_pending_orders():
    batch_size = 1000
    last_id    = 0

    while True:
        batch = list(
            Order.objects.filter(
                status='pending',
                id__gt=last_id
            ).select_related('customer')
            .order_by('id')[:batch_size]
        )
        if not batch:
            break

        order_ids = [o.id for o in batch]

        # Bulk process করো
        for order in batch:
            process_order(order)

        # Bulk status update
        Order.objects.filter(id__in=order_ids).update(
            status='processed',
            processed_at=timezone.now()
        )

        last_id = batch[-1].id
```

**Q8: একটা API endpoint 5 seconds নিচ্ছে। Debug করার process কী?**

```
Step 1: django-debug-toolbar বা logging দিয়ে query count দেখো
Step 2: EXPLAIN ANALYZE সব queries তে চালাও
Step 3: N+1 আছে? → select_related/prefetch_related
Step 4: Seq Scan আছে? → Index দাও
Step 5: Heavy aggregation? → DB level এ করো, Python loop এ না
Step 6: Pagination issue? → Cursor pagination
Step 7: External API call আছে? → Async করো
Step 8: Result cache করা যায়? → Redis cache যোগ করো
```

---

## 7.14 Hands-on Exercises

### Exercise 1 — N+1 Fix
```python
# এই code তে N+1 খুঁজে বের করো এবং fix করো:
def get_blog_posts():
    posts = BlogPost.objects.filter(published=True)
    return [
        {
            'title':    p.title,
            'author':   p.author.name,       # N+1
            'tags':     [t.name for t in p.tags.all()],  # N+1
            'comments': p.comments.count()   # N+1
        }
        for p in posts
    ]
```

### Exercise 2 — EXPLAIN Analysis
```sql
-- এই query optimize করো:
SELECT DISTINCT u.name
FROM users u
WHERE u.id IN (
    SELECT DISTINCT o.customer_id
    FROM orders o
    WHERE o.created_at > NOW() - INTERVAL '30 days'
      AND o.total_amount > 1000
);
```

### Exercise 3 — Bulk Operations
```python
# 100,000 products এর price 10% বাড়াও
# তিনটা পদ্ধতিতে করো এবং সময় compare করো:
# 1. Loop + save()
# 2. bulk_update()
# 3. update() with F()
```

### Mini Project
```
E-commerce product listing API optimize করো:
- Products, categories, tags, reviews সহ
- Pagination implement করো (cursor-based)
- Filtering add করো (price range, category, rating)
- EXPLAIN ANALYZE দিয়ে prove করো query fast
- Query count 3 এর নিচে রাখো
```

---

## 7.15 Module Summary & Cheat Sheet

```
QUERY OPTIMIZATION CHEAT SHEET
================================

N+1 SOLUTION:
  FK/OneToOne  → .select_related('field')
  M2M/Reverse  → .prefetch_related('field')
  Filtered     → Prefetch('field', queryset=...)
  Detect       → django-debug-toolbar / logging

EXPLAIN KEYWORDS:
  Seq Scan        → No index (bad for large tables)
  Index Scan      → Using index (good)
  Index Only Scan → Covering index (best)
  cost=X..Y       → estimated cost
  actual time=X   → real ms
  Buffers hit=X   → from cache
  Buffers read=X  → from disk (slow)

PAGINATION:
  ❌ OFFSET    → slow for large pages
  ✅ Cursor    → WHERE id > last_id (always fast)

BULK OPERATIONS:
  ❌ Loop + create()   → N queries
  ✅ bulk_create()     → 1 query
  ✅ bulk_update()     → 1 query
  ✅ .update(F())      → atomic SQL UPDATE

FIELD SELECTION:
  .values('id', 'name')  → SELECT id, name only
  .only('id', 'name')    → Model instance, deferred fields
  .defer('big_field')    → Exclude heavy fields

AGGREGATION:
  .count()        → SELECT COUNT(*)
  .exists()       → SELECT 1 LIMIT 1 (fastest!)
  .aggregate()    → Single value
  .annotate()     → Per-row value

DJANGO DEBUG:
  queryset.explain()                 → EXPLAIN
  queryset.explain(analyze=True)     → EXPLAIN ANALYZE
  connection.queries                 → All queries list
  len(connection.queries)            → Query count

PRODUCTION RULES:
  ✅ select_related সব FK এ
  ✅ values() যখন full model দরকার নেই
  ✅ exists() boolean check এ
  ✅ bulk_create/bulk_update batch operations এ
  ✅ F() concurrent updates এ
  ✅ Cursor pagination large datasets এ
  ✅ EXPLAIN ANALYZE before deploying
```
