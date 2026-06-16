# MODULE 9: ORM Deep Dive (Django ORM)
## বাংলায় সম্পূর্ণ Django ORM গাইড — Production Level

> **কেন এই module?**
> তুমি Django দিয়ে কাজ করো — কিন্তু ORM এর ভেতরে কী হচ্ছে সেটা না জানলে
> production এ mysterious bugs, slow APIs, এবং data corruption দেখবে।
> Senior Django developer হতে হলে ORM কে SQL এর মতোই ভালো বুঝতে হবে।

---

## 9.1 ORM Architecture (ORM এর ভেতরে কী হয়?)

### ORM কীভাবে কাজ করে

```
তোমার Python Code
      ↓
Django ORM (QuerySet API)
      ↓
SQL Compiler (Django এর ভেতরে)
      ↓
Database Backend (psycopg2 for PostgreSQL)
      ↓
PostgreSQL Server
      ↓
Result → Python Objects
```

### QuerySet Lifecycle

```
QuerySet তৈরি হওয়া ≠ Query চলা

qs = Order.objects.filter(status='pending')
# এখনো কোনো SQL নেই! Lazy evaluation।

# এগুলো করলে SQL চলবে (evaluation):
list(qs)           # সব rows
qs[0]              # প্রথম row
for o in qs:       # iteration
qs.count()         # COUNT query
qs.exists()        # EXISTS query
bool(qs)           # EXISTS check
str(qs)            # repr — evaluation
```

### QuerySet Chaining

```python
# প্রতিটা filter/exclude নতুন QuerySet return করে — original unchanged
base_qs    = Order.objects.filter(status='pending')
recent_qs  = base_qs.filter(created_at__gte=timezone.now() - timedelta(days=7))
ordered_qs = recent_qs.order_by('-created_at')
limited_qs = ordered_qs[:20]

# সবগুলো chain করা যায়
final_qs = Order.objects.filter(
    status='pending'
).filter(
    created_at__gte=timezone.now() - timedelta(days=7)
).select_related('customer').order_by('-created_at')[:20]

# Generated SQL:
# SELECT orders.*, customers.*
# FROM orders
# INNER JOIN customers ON customers.id = orders.customer_id
# WHERE orders.status = 'pending'
#   AND orders.created_at >= '2024-06-01'
# ORDER BY orders.created_at DESC
# LIMIT 20;
```

---

## 9.2 Lazy Loading vs Eager Loading

### Lazy Loading (ডিফল্ট — সমস্যার কারণ)

```python
# Django ORM default: lazy loading
order = Order.objects.get(id=1)
# SQL: SELECT * FROM orders WHERE id = 1

print(order.customer.name)
# এখন আরেকটা SQL: SELECT * FROM customers WHERE id = X
# এটাই lazy loading — দরকার হলে তখন load করে

# Loop এ এটা N+1 তৈরি করে!
orders = Order.objects.filter(status='pending')
for o in orders:
    print(o.customer.name)  # প্রতিটা iteration এ 1 query!
```

### Eager Loading — select_related (JOIN)

```python
# ForeignKey এবং OneToOne এর জন্য
order = Order.objects.select_related('customer').get(id=1)
# SQL: SELECT orders.*, customers.*
#      FROM orders
#      JOIN customers ON customers.id = orders.customer_id
#      WHERE orders.id = 1
# একটাই query — customer আগে থেকেই loaded

# Nested select_related
order = Order.objects.select_related(
    'customer',
    'customer__address',    # customer এর address
    'assigned_staff'        # order এর staff
).get(id=1)

# Generated SQL:
# SELECT orders.*, customers.*, addresses.*, staff.*
# FROM orders
# JOIN customers ON customers.id = orders.customer_id
# JOIN addresses ON addresses.id = customers.address_id
# LEFT JOIN staff ON staff.id = orders.assigned_staff_id
# WHERE orders.id = 1;
```

### Eager Loading — prefetch_related (Separate Query)

```python
# ManyToMany এবং reverse ForeignKey এর জন্য
order = Order.objects.prefetch_related('items').get(id=1)
# SQL 1: SELECT * FROM orders WHERE id = 1
# SQL 2: SELECT * FROM order_items WHERE order_id = 1
# Python এ join করে দেয়

# Nested prefetch
orders = Order.objects.prefetch_related(
    'items',
    'items__product',
    'items__product__category'
).filter(status='pending')
# SQL 1: orders
# SQL 2: order_items WHERE order_id IN (...)
# SQL 3: products WHERE id IN (...)
# SQL 4: categories WHERE id IN (...)
# মোট 4 queries — N+1 নয়!
```

### Prefetch Object — Filtered Prefetch

```python
from django.db.models import Prefetch

# Prefetch করার সময় filter এবং ordering করা যায়
orders = Order.objects.prefetch_related(
    Prefetch(
        'items',
        queryset=OrderItem.objects.filter(
            quantity__gt=0
        ).select_related('product').order_by('-quantity'),
        to_attr='active_items'  # আলাদা attribute এ store
    )
).filter(status='pending')

for order in orders:
    for item in order.active_items:  # filtered, pre-loaded
        print(item.product.name, item.quantity)
```

---

## 9.3 annotate() — Row-Level Calculation

### কী এবং কেন?

```python
# annotate() = প্রতিটা row এ একটা calculated field যোগ করো
# Database level এ calculate হয় — Python loop এর দরকার নেই

from django.db.models import Count, Sum, Avg, Max, Min, F, Value
from django.db.models.functions import Coalesce

# প্রতিটা customer এর order count
customers = Customer.objects.annotate(
    order_count=Count('orders')
)

# Generated SQL:
# SELECT customers.*, COUNT(orders.id) AS order_count
# FROM customers
# LEFT OUTER JOIN orders ON orders.customer_id = customers.id
# GROUP BY customers.id;

for c in customers:
    print(c.name, c.order_count)  # No extra query!

# Multiple annotations
customers = Customer.objects.annotate(
    order_count=Count('orders'),
    total_spent=Sum('orders__total_amount'),
    avg_order=Avg('orders__total_amount'),
    last_order=Max('orders__created_at')
)

# Generated SQL:
# SELECT customers.*,
#   COUNT(orders.id) AS order_count,
#   SUM(orders.total_amount) AS total_spent,
#   AVG(orders.total_amount) AS avg_order,
#   MAX(orders.created_at) AS last_order
# FROM customers
# LEFT OUTER JOIN orders ON orders.customer_id = customers.id
# GROUP BY customers.id;
```

### annotate() + filter() — Powerful Combination

```python
# Top customers যারা 5+ orders করেছে এবং total spent > 10000
top_customers = Customer.objects.annotate(
    order_count=Count('orders'),
    total_spent=Sum('orders__total_amount')
).filter(
    order_count__gte=5,
    total_spent__gt=10000
).order_by('-total_spent')

# Generated SQL:
# SELECT customers.*,
#   COUNT(orders.id) AS order_count,
#   SUM(orders.total_amount) AS total_spent
# FROM customers
# LEFT JOIN orders ON orders.customer_id = customers.id
# GROUP BY customers.id
# HAVING COUNT(orders.id) >= 5
#    AND SUM(orders.total_amount) > 10000
# ORDER BY total_spent DESC;
```

### Conditional Annotation

```python
from django.db.models import Case, When, IntegerField

# প্রতিটা product এর জন্য: এই মাসে কতটা বিক্রি হয়েছে
from django.utils import timezone
this_month = timezone.now().replace(day=1)

products = Product.objects.annotate(
    monthly_sales=Coalesce(
        Sum(
            'orderitem__quantity',
            filter=Q(orderitem__order__created_at__gte=this_month)
        ),
        0  # NULL হলে 0
    ),
    is_low_stock=Case(
        When(stock__lte=10, then=Value(True)),
        default=Value(False),
    )
)
```

---

## 9.4 aggregate() — Table-Level Calculation

```python
from django.db.models import Count, Sum, Avg, Max, Min

# Single value return করে (QuerySet নয়)
result = Order.objects.aggregate(
    total_orders=Count('id'),
    total_revenue=Sum('total_amount'),
    avg_order_value=Avg('total_amount'),
    max_order=Max('total_amount'),
    min_order=Min('total_amount')
)

# Result: dictionary
# {'total_orders': 15000, 'total_revenue': Decimal('2500000.00'), ...}

# Generated SQL:
# SELECT
#   COUNT(id) AS total_orders,
#   SUM(total_amount) AS total_revenue,
#   AVG(total_amount) AS avg_order_value,
#   MAX(total_amount) AS max_order,
#   MIN(total_amount) AS min_order
# FROM orders;

print(result['total_revenue'])  # 2500000.00
```

---

## 9.5 F() Expressions — Database Level Operations

### কী এবং কেন?

```python
from django.db.models import F

# F() = database এর actual column value refer করো
# Python এ read না করেই database level এ operate করো

# ❌ Race condition আছে
product = Product.objects.get(id=1)
product.views += 1    # Python: read 100, add 1 = 101
product.save()        # Write 101
# দুটো request একসাথে আসলে: দুজনেই 100 পাবে, দুজনেই 101 করবে!

# ✅ F() — atomic database operation
Product.objects.filter(id=1).update(views=F('views') + 1)
# SQL: UPDATE products SET views = views + 1 WHERE id = 1;
# Database level atomic!

# Multiple F() operations
Product.objects.filter(id=1).update(
    views=F('views') + 1,
    score=F('views') * F('rating'),  # columns এর মধ্যে calculation
    updated_at=timezone.now()
)

# F() দিয়ে column compare করো
# price > original_price যে products গুলো
discounted = Product.objects.filter(
    original_price__gt=F('price')
)
# SQL: WHERE original_price > price

# Date arithmetic
from django.db.models import ExpressionWrapper, DurationField
overdue_orders = Order.objects.filter(
    due_date__lt=F('created_at') + timedelta(days=7)
)
```

### F() in annotate()

```python
# প্রতিটা product এর profit margin calculate করো
products = Product.objects.annotate(
    profit=F('price') - F('cost'),
    margin_pct=ExpressionWrapper(
        (F('price') - F('cost')) * 100.0 / F('price'),
        output_field=FloatField()
    )
)

# Generated SQL:
# SELECT *, (price - cost) AS profit,
#   ((price - cost) * 100.0 / price) AS margin_pct
# FROM products;
```

---

## 9.6 Q() Objects — Complex Queries

```python
from django.db.models import Q

# Q() = complex WHERE conditions

# OR condition
orders = Order.objects.filter(
    Q(status='pending') | Q(status='processing')
)
# SQL: WHERE status = 'pending' OR status = 'processing'

# AND + OR combination
orders = Order.objects.filter(
    Q(customer__city='Dhaka') &
    (Q(status='pending') | Q(status='processing'))
)
# SQL: WHERE customer.city = 'Dhaka'
#       AND (status = 'pending' OR status = 'processing')

# NOT condition
orders = Order.objects.filter(
    ~Q(status='cancelled')
)
# SQL: WHERE NOT (status = 'cancelled')

# Dynamic filter building
def search_orders(status=None, city=None, min_amount=None):
    filters = Q()
    
    if status:
        filters &= Q(status=status)
    if city:
        filters &= Q(customer__city=city)
    if min_amount:
        filters &= Q(total_amount__gte=min_amount)
    
    return Order.objects.filter(filters).select_related('customer')

# Usage
orders = search_orders(status='pending', city='Dhaka', min_amount=1000)
```

---

## 9.7 bulk_create() এবং bulk_update()

### bulk_create — Massive Insert Performance

```python
# ❌ Slow: N queries
for data in product_data_list:
    Product.objects.create(**data)

# ✅ Fast: 1 query
products = [
    Product(
        name=d['name'],
        price=d['price'],
        category_id=d['category_id']
    )
    for d in product_data_list
]

created = Product.objects.bulk_create(
    products,
    batch_size=500,         # 500 rows per INSERT
    ignore_conflicts=True   # Duplicate হলে skip
)

# Generated SQL (per batch):
# INSERT INTO products (name, price, category_id)
# VALUES ('A', 100, 1), ('B', 200, 2), ...  -- 500 rows at once
# ON CONFLICT DO NOTHING;

# Return ids (PostgreSQL only, Django 4.1+)
created_with_ids = Product.objects.bulk_create(
    products,
    update_conflicts=True,
    unique_fields=['sku'],
    update_fields=['price', 'stock']
)
# SQL: INSERT ... ON CONFLICT (sku) DO UPDATE SET price=..., stock=...
```

### bulk_update — Massive Update Performance

```python
# ❌ Slow: N queries
for product in products:
    product.price = product.price * Decimal('1.1')
    product.save()

# ✅ Fast: 1 query
for product in products:
    product.price = product.price * Decimal('1.1')

Product.objects.bulk_update(
    products,
    fields=['price'],
    batch_size=500
)

# Generated SQL (per batch):
# UPDATE products SET price = CASE
#   WHEN id = 1 THEN 110.00
#   WHEN id = 2 THEN 220.00
#   WHEN id = 3 THEN 165.00
#   ...
# END WHERE id IN (1, 2, 3, ...);
```

---

## 9.8 values() এবং values_list()

```python
# values() — dictionary return করে (Model instance নয়)
orders = Order.objects.filter(
    status='pending'
).values('id', 'total_amount', 'customer__name')
# [{'id': 1, 'total_amount': 500, 'customer__name': 'Rahim'}, ...]

# SQL: SELECT orders.id, orders.total_amount, customers.name
#      FROM orders JOIN customers ON ...
#      WHERE status = 'pending';

# values_list() — tuple return করে
order_ids = Order.objects.filter(
    status='pending'
).values_list('id', flat=True)  # flat=True → list of values (not tuples)
# [1, 2, 3, 4, 5, ...]

# SQL: SELECT id FROM orders WHERE status = 'pending';

# Named tuples
orders = Order.objects.values_list(
    'id', 'total_amount', named=True
)
for o in orders:
    print(o.id, o.total_amount)
```

### কখন values() ব্যবহার করবে?

```python
# ✅ values() ব্যবহার করো যখন:
# - Full model instance দরকার নেই
# - API response এ শুধু কিছু fields দরকার
# - Aggregation এর পর data দরকার
# - Memory optimize করতে হবে

# ❌ values() এড়াও যখন:
# - Model methods call করতে হবে (save, delete, custom methods)
# - Related objects access করতে হবে
```

---

## 9.9 Raw SQL — কখন এবং কীভাবে

### Manager.raw()

```python
# Complex query যেটা ORM এ express করা কঠিন
products = Product.objects.raw(
    """
    SELECT p.*, 
           COUNT(oi.id) AS sales_count,
           COALESCE(SUM(oi.quantity), 0) AS total_sold
    FROM products p
    LEFT JOIN order_items oi ON oi.product_id = p.id
    LEFT JOIN orders o ON o.id = oi.order_id
        AND o.status = 'delivered'
    GROUP BY p.id
    ORDER BY total_sold DESC
    LIMIT 10
    """
)

for p in products:
    print(p.name, p.total_sold)  # annotated fields accessible
```

### cursor() — Arbitrary SQL

```python
from django.db import connection

def get_sales_by_region():
    with connection.cursor() as cursor:
        cursor.execute("""
            SELECT
                r.name AS region,
                COUNT(o.id) AS orders,
                SUM(o.total_amount) AS revenue
            FROM regions r
            JOIN customers c ON c.region_id = r.id
            JOIN orders o ON o.customer_id = c.id
            WHERE o.created_at >= %s
            GROUP BY r.name
            ORDER BY revenue DESC
        """, [timezone.now() - timedelta(days=30)])
        
        columns = [col[0] for col in cursor.description]
        return [dict(zip(columns, row)) for row in cursor.fetchall()]

# SQL Injection safe: %s placeholder ব্যবহার করো
# NEVER: f"WHERE name = '{user_input}'"  ← SQL Injection!
```

### Raw SQL কখন ব্যবহার করবে?

```
✅ করবে:
  - Window functions (ORM এ limited support)
  - Complex recursive CTEs
  - Database-specific functions
  - Performance-critical queries
  - Reporting/Analytics

❌ করবে না:
  - Simple CRUD operations
  - যেখানে ORM যথেষ্ট
  - User input directly pass করে (SQL injection risk)
```

---

## 9.10 Transactions in Django ORM

```python
from django.db import transaction

# 1. Decorator
@transaction.atomic
def create_order(user_id, items):
    order = Order.objects.create(user_id=user_id)
    OrderItem.objects.bulk_create([
        OrderItem(order=order, **item) for item in items
    ])
    return order

# 2. Context Manager
def create_order_v2(user_id, items):
    with transaction.atomic():
        order = Order.objects.create(user_id=user_id)
        OrderItem.objects.bulk_create([...])
    return order

# 3. Savepoint (nested atomic)
@transaction.atomic
def complex_flow(user_id):
    user = User.objects.create(...)
    
    try:
        with transaction.atomic():  # Savepoint
            send_to_crm(user)       # External call — might fail
    except Exception:
        pass  # CRM fail হলেও user create হবে
    
    # on_commit — transaction commit হওয়ার পর
    transaction.on_commit(
        lambda: send_welcome_email.delay(user.id)
    )

# 4. Manual transaction control
from django.db import connection

def manual_transaction():
    with transaction.atomic():
        connection.cursor().execute("SAVEPOINT my_sp")
        try:
            risky_operation()
        except Exception:
            connection.cursor().execute("ROLLBACK TO SAVEPOINT my_sp")
```

---

## 9.11 ORM Anti-Patterns এবং Fixes

### Anti-Pattern 1: Python Loop এ Aggregation

```python
# ❌ Python এ calculate করছো
total = 0
for order in Order.objects.filter(customer_id=1):
    total += order.total_amount  # N queries + Python loop

# ✅ Database এ calculate করো
from django.db.models import Sum
total = Order.objects.filter(
    customer_id=1
).aggregate(total=Sum('total_amount'))['total'] or 0
# 1 query, database level
```

### Anti-Pattern 2: QuerySet তৈরি করে Count

```python
# ❌ সব objects load করে count করছো
count = len(Order.objects.filter(status='pending'))
# SQL: SELECT * FROM orders WHERE status='pending' → সব rows load!

# ✅ COUNT query
count = Order.objects.filter(status='pending').count()
# SQL: SELECT COUNT(*) FROM orders WHERE status='pending'
```

### Anti-Pattern 3: get() এ Exception Handle না করা

```python
# ❌ DoesNotExist বা MultipleObjectsReturned crash করবে
order = Order.objects.get(id=order_id)

# ✅ Proper handling
try:
    order = Order.objects.get(id=order_id)
except Order.DoesNotExist:
    return Response({'error': 'Not found'}, status=404)

# অথবা
order = Order.objects.filter(id=order_id).first()
if not order:
    return Response({'error': 'Not found'}, status=404)
```

### Anti-Pattern 4: Update এ Full Object Load

```python
# ❌ Full object load করছো শুধু একটা field update এর জন্য
order = Order.objects.get(id=1)
order.status = 'shipped'
order.save()
# SQL: UPDATE orders SET status='shipped', total_amount=..., customer_id=...,
#      created_at=..., updated_at=... WHERE id=1;
# সব fields update হচ্ছে!

# ✅ শুধু দরকারি field update করো
Order.objects.filter(id=1).update(
    status='shipped',
    updated_at=timezone.now()
)
# SQL: UPDATE orders SET status='shipped', updated_at=NOW() WHERE id=1;

# অথবা save() তে update_fields দাও
order = Order.objects.get(id=1)
order.status = 'shipped'
order.save(update_fields=['status', 'updated_at'])
```

### Anti-Pattern 5: Queryset Reuse After Evaluation

```python
# ❌ Cache aware না হওয়া
orders = Order.objects.filter(status='pending')
first_count = orders.count()  # DB hit
# কিছু orders status change হলো...
second_count = orders.count()  # এখনো cached? না! আবার DB hit
# কিন্তু iteration করলে cache হয়:
list(orders)  # DB hit, cached
list(orders)  # cache থেকে (DB hit নেই)

# ✅ সচেতন থাকো
orders = list(Order.objects.filter(status='pending'))  # evaluate once
count = len(orders)  # No DB hit
```

---

## 9.12 Django ORM — Generated SQL Examples

### Complex Real-world Query

```python
# E-commerce dashboard: category wise sales last 30 days
from django.db.models import Sum, Count, F, Q
from django.db.models.functions import TruncDate

report = OrderItem.objects.filter(
    order__created_at__gte=timezone.now() - timedelta(days=30),
    order__status='delivered'
).values(
    'product__category__name'
).annotate(
    total_qty=Sum('quantity'),
    total_revenue=Sum(F('quantity') * F('unit_price')),
    order_count=Count('order', distinct=True)
).order_by('-total_revenue')

# Generated SQL:
# SELECT
#   categories.name AS product__category__name,
#   SUM(order_items.quantity) AS total_qty,
#   SUM(order_items.quantity * order_items.unit_price) AS total_revenue,
#   COUNT(DISTINCT order_items.order_id) AS order_count
# FROM order_items
# INNER JOIN products ON products.id = order_items.product_id
# INNER JOIN categories ON categories.id = products.category_id
# INNER JOIN orders ON orders.id = order_items.order_id
# WHERE orders.created_at >= '2024-05-01'
#   AND orders.status = 'delivered'
# GROUP BY categories.name
# ORDER BY total_revenue DESC;
```

---

## 9.13 Performance Optimization Techniques

### Technique 1: only() এবং defer()

```python
# only() — শুধু এই fields load করো
users = User.objects.only('id', 'name', 'email')
# SQL: SELECT id, name, email FROM users;
# Other fields access করলে lazy load হবে (additional query!)

# defer() — এই fields বাদ দিয়ে বাকি সব load করো
users = User.objects.defer('bio', 'profile_picture')
# SQL: SELECT id, name, email, phone, created_at FROM users;
# (bio এবং profile_picture বাদ)
```

### Technique 2: iterator() — Memory Optimization

```python
# Large queryset process করতে
# iterator() ব্যবহার করলে সব rows একসাথে memory তে load হয় না

for order in Order.objects.filter(
    status='pending'
).iterator(chunk_size=1000):
    process_order(order)
# প্রতিবার 1000 rows memory তে — পুরো table নয়

# Warning: iterator() তে prefetch_related কাজ করে না
```

### Technique 3: exists() vs count() vs bool(queryset)

```python
# Fastest: শুধু existence check
if Order.objects.filter(customer_id=1).exists():
    # SQL: SELECT 1 FROM orders WHERE customer_id=1 LIMIT 1
    pass

# count() — exact number দরকার হলে
count = Order.objects.filter(status='pending').count()
# SQL: SELECT COUNT(*) FROM orders WHERE status='pending'

# ❌ Slowest: bool(queryset) — সব rows load করে
if Order.objects.filter(customer_id=1):  # Evaluates queryset!
    pass
```

### Technique 4: Database Functions

```python
from django.db.models.functions import (
    TruncMonth, TruncDate, TruncYear,
    Lower, Upper, Length, Substr,
    Coalesce, Cast, Now
)

# Monthly sales report
monthly = Order.objects.annotate(
    month=TruncMonth('created_at')
).values('month').annotate(
    revenue=Sum('total_amount'),
    orders=Count('id')
).order_by('month')

# Generated SQL:
# SELECT DATE_TRUNC('month', created_at) AS month,
#        SUM(total_amount) AS revenue,
#        COUNT(id) AS orders
# FROM orders
# GROUP BY DATE_TRUNC('month', created_at)
# ORDER BY month;

# String functions
users = User.objects.annotate(
    name_lower=Lower('name')
).filter(name_lower='rahim khan')
```

---

## 9.14 ORM এ Window Functions (Django 2.0+)

```python
from django.db.models import Window, F
from django.db.models.functions import Rank, RowNumber, Lag, Lead

# প্রতিটা category তে price অনুযায়ী rank
products = Product.objects.annotate(
    price_rank=Window(
        expression=Rank(),
        partition_by=[F('category_id')],
        order_by=F('price').desc()
    )
)

# Generated SQL:
# SELECT *,
#   RANK() OVER (
#     PARTITION BY category_id
#     ORDER BY price DESC
#   ) AS price_rank
# FROM products;

# LAG — আগের row এর value
monthly_sales = Sale.objects.annotate(
    month=TruncMonth('created_at'),
    prev_month_revenue=Window(
        expression=Lag('total_amount', offset=1),
        order_by=TruncMonth('created_at').asc()
    )
).values('month', 'total_amount', 'prev_month_revenue')
```

---

## 9.15 Real-world ORM Examples

### Example 1 — Instagram-style Feed

```python
def get_user_feed(user_id, last_post_id=None, limit=20):
    following_ids = Follow.objects.filter(
        follower_id=user_id
    ).values_list('following_id', flat=True)

    qs = Post.objects.filter(
        author_id__in=following_ids
    ).select_related(
        'author'
    ).prefetch_related(
        Prefetch(
            'likes',
            queryset=Like.objects.filter(user_id=user_id),
            to_attr='user_likes'
        ),
        'media_files'
    ).annotate(
        like_count=Count('likes'),
        comment_count=Count('comments')
    ).order_by('-created_at')

    if last_post_id:
        qs = qs.filter(id__lt=last_post_id)  # cursor pagination

    return qs[:limit]
```

### Example 2 — E-commerce Product Search

```python
from django.contrib.postgres.search import SearchVector, SearchQuery, SearchRank
from django.db.models import Q

def search_products(query, category_id=None, min_price=None,
                    max_price=None, in_stock=True, sort='relevance'):
    
    search_query = SearchQuery(query, config='english')
    search_vector = SearchVector('name', weight='A') + \
                    SearchVector('description', weight='B') + \
                    SearchVector('brand', weight='C')

    qs = Product.objects.annotate(
        rank=SearchRank(search_vector, search_query)
    ).filter(rank__gte=0.1)

    if category_id:
        qs = qs.filter(category_id=category_id)
    if min_price:
        qs = qs.filter(price__gte=min_price)
    if max_price:
        qs = qs.filter(price__lte=max_price)
    if in_stock:
        qs = qs.filter(stock__gt=0)

    sort_map = {
        'relevance': '-rank',
        'price_asc': 'price',
        'price_desc': '-price',
        'newest': '-created_at',
    }
    qs = qs.order_by(sort_map.get(sort, '-rank'))

    return qs.select_related('category').only(
        'id', 'name', 'price', 'stock', 'thumbnail', 'category__name'
    )
```

### Example 3 — Analytics Dashboard Query

```python
from django.db.models import Sum, Count, Avg, F, Q
from django.db.models.functions import TruncDate

def get_dashboard_stats(start_date, end_date):
    base_qs = Order.objects.filter(
        created_at__range=(start_date, end_date),
        status__in=['delivered', 'completed']
    )

    # Daily revenue
    daily_revenue = base_qs.annotate(
        date=TruncDate('created_at')
    ).values('date').annotate(
        revenue=Sum('total_amount'),
        orders=Count('id'),
        avg_value=Avg('total_amount')
    ).order_by('date')

    # Top products
    top_products = OrderItem.objects.filter(
        order__in=base_qs
    ).values(
        'product__name'
    ).annotate(
        total_qty=Sum('quantity'),
        total_revenue=Sum(F('quantity') * F('unit_price'))
    ).order_by('-total_revenue')[:10]

    # Customer segments
    customer_stats = Customer.objects.annotate(
        order_count=Count('orders', filter=Q(
            orders__created_at__range=(start_date, end_date)
        )),
        total_spent=Sum('orders__total_amount', filter=Q(
            orders__created_at__range=(start_date, end_date)
        ))
    ).filter(order_count__gt=0).aggregate(
        new_customers=Count('id', filter=Q(order_count=1)),
        repeat_customers=Count('id', filter=Q(order_count__gt=1)),
        avg_customer_value=Avg('total_spent')
    )

    return {
        'daily_revenue': list(daily_revenue),
        'top_products': list(top_products),
        'customer_stats': customer_stats
    }
```

---

## 9.16 Common Mistakes

```python
# ✅ DO                                  # ❌ DON'T
Order.objects.filter(id=1).update(...)  # Order.objects.get().save()
.exists()                               # if queryset:
.count()                                # len(queryset)
.values('id', 'name')                   # full model when not needed
.select_related('fk')                   # access fk in loop
.bulk_create(items)                     # loop + create()
F('col') in update()                    # read → modify → save
transaction.on_commit(task.delay)       # task.delay() inside atomic
.iterator()                             # list() on huge queryset
.only('needed_fields')                  # loading all fields
```

---

## 9.17 Production Best Practices

```
1.  সব FK/OneToOne এ select_related দাও
2.  Reverse FK / M2M তে prefetch_related দাও
3.  API response এ values() দিয়ে দরকারি fields নাও
4.  Numeric update এ F() ব্যবহার করো
5.  Bulk insert/update এ bulk_create/bulk_update ব্যবহার করো
6.  Transaction এ on_commit() দিয়ে side effects handle করো
7.  Large dataset এ iterator() ব্যবহার করো
8.  exists() দিয়ে boolean check করো
9.  Django Debug Toolbar দিয়ে query count monitor করো
10. Raw SQL শেষ resort — ORM আগে try করো
11. QuerySet এর explain() method ব্যবহার করো
12. update_fields=['field1'] দিয়ে targeted save করো
```

---

## 9.18 Interview Questions

### Beginner Level

**Q1: Django ORM এ lazy loading মানে কী?**
> QuerySet তৈরি হলেই SQL execute হয় না। প্রথমবার iterate, list(), count() বা অন্যভাবে evaluate করলে তখন SQL চলে। এটা performance friendly কারণ chain করতে পারো।

**Q2: select_related এবং prefetch_related এর পার্থক্য কী?**
> select_related SQL JOIN করে এক query তে data আনে — ForeignKey/OneToOne এর জন্য। prefetch_related আলাদা queries করে Python এ merge করে — ManyToMany এবং reverse FK এর জন্য।

**Q3: F() expression কেন ব্যবহার করা হয়?**
> Python এ read-modify-write race condition এড়াতে। F() database level এ atomic SQL operation করে। Stock update, counter increment এ mandatory।

---

### Intermediate Level

**Q4: annotate() এবং aggregate() এর পার্থক্য কী?**
> annotate() প্রতিটা row এ calculated field যোগ করে QuerySet return করে। aggregate() পুরো queryset এর উপর single calculation করে dictionary return করে।

**Q5: Q() object কোন সমস্যা solve করে?**
> Complex OR, AND, NOT conditions। Dynamic filter building। ORM এর default filter AND conditions chain করে, কিন্তু OR করতে Q() দরকার।

**Q6: bulk_create() এ batch_size কেন দিতে হয়?**
> Database এর maximum query size limit আছে। 100,000 rows একসাথে insert করলে "too many parameters" error হতে পারে। batch_size=500 মানে 500 rows per INSERT query।

---

### Senior / FAANG Level

**Q7: একটা high-traffic API তে ORM optimize করার strategy কী?**
```
1. django-debug-toolbar দিয়ে query count measure করো
2. N+1 fix: select_related/prefetch_related
3. Field selection: values() বা only()
4. exists() for boolean checks
5. Database level aggregation (annotate/aggregate)
6. Bulk operations (bulk_create/bulk_update)
7. Covering index ensure করো
8. Queryset result cache করো (Redis)
9. Read-heavy: read replica তে route করো
10. Heavy computation: background task (Celery)
```

**Q8: Django ORM দিয়ে window function ব্যবহার করে প্রতিটা category তে সবচেয়ে বেশি বিক্রিত product খুঁজে বের করো।**

```python
from django.db.models import Window, Sum, F
from django.db.models.functions import Rank

top_per_category = Product.objects.annotate(
    total_sales=Sum('orderitem__quantity'),
    sales_rank=Window(
        expression=Rank(),
        partition_by=[F('category_id')],
        order_by=Sum('orderitem__quantity').desc()
    )
).filter(sales_rank=1).select_related('category')

# Generated SQL:
# SELECT *, SUM(order_items.quantity) AS total_sales,
#   RANK() OVER (
#     PARTITION BY category_id
#     ORDER BY SUM(order_items.quantity) DESC
#   ) AS sales_rank
# FROM products
# LEFT JOIN order_items ON order_items.product_id = products.id
# GROUP BY products.id
# HAVING RANK() OVER (...) = 1;
```

---

## 9.19 Hands-on Exercises

### Exercise 1
```python
# এই code optimize করো — কতটা queries কমাতে পারো?
def get_blog_feed():
    posts = BlogPost.objects.filter(published=True).order_by('-created_at')[:20]
    return [
        {
            'title':    p.title,
            'author':   p.author.name,
            'avatar':   p.author.profile.avatar_url,
            'tags':     [t.name for t in p.tags.all()],
            'likes':    p.likes.count(),
            'comments': p.comments.count(),
        }
        for p in posts
    ]
```

### Exercise 2
```python
# annotate() দিয়ে এই report তৈরি করো:
# প্রতিটা customer এর:
# - Total orders
# - Total spent
# - Average order value
# - Last order date
# - "VIP" label (total_spent > 50000)
```

### Exercise 3
```python
# bulk_create() দিয়ে 100,000 products insert করো
# এবং সময় measure করো:
# 1. Loop + create()
# 2. bulk_create(batch_size=100)
# 3. bulk_create(batch_size=1000)
# কোনটা fastest?
```

### Mini Project
```
E-commerce Analytics API:
1. /api/dashboard/ — daily revenue, top products, customer stats
2. /api/products/search/ — full-text search with filters
3. /api/customers/{id}/summary/ — customer lifetime value
সব queries EXPLAIN ANALYZE দিয়ে verify করো
Query count 5 এর নিচে রাখো প্রতিটা endpoint এ
```

---

## 9.20 Module Summary & Cheat Sheet

```
DJANGO ORM CHEAT SHEET
========================

LOADING RELATED DATA:
  .select_related('fk', 'fk__nested')   → JOIN (FK/O2O)
  .prefetch_related('m2m', 'reverse_fk') → Separate query (M2M/reverse)
  Prefetch('field', queryset=..., to_attr='name') → Filtered prefetch

FIELD OPERATIONS:
  F('col')                    → DB column reference (atomic)
  F('col') + 1                → Atomic increment
  F('col1') * F('col2')       → Column arithmetic

COMPLEX CONDITIONS:
  Q(a=1) | Q(b=2)             → OR
  Q(a=1) & Q(b=2)             → AND
  ~Q(a=1)                     → NOT

AGGREGATION:
  .aggregate(total=Sum('price'))    → Single dict result
  .annotate(count=Count('items'))   → Per-row calculation
  .values('cat').annotate(...)      → GROUP BY cat

BULK OPERATIONS:
  .bulk_create(objs, batch_size=500)
  .bulk_update(objs, ['field'], batch_size=500)
  .update(field=F('field') + 1)     → Atomic update

FIELD SELECTION:
  .values('id', 'name')             → Dict queryset
  .values_list('id', flat=True)     → List of values
  .only('id', 'name')               → Partial model
  .defer('big_field')               → Exclude field

PERFORMANCE:
  .exists()                  → Fastest boolean check
  .count()                   → COUNT(*)
  .iterator(chunk_size=1000) → Memory-efficient iteration
  .explain(analyze=True)     → EXPLAIN ANALYZE

TRANSACTIONS:
  @transaction.atomic                → Decorator
  with transaction.atomic():         → Context manager
  transaction.on_commit(fn)          → Post-commit hook
  nested atomic() = Savepoint

ANTI-PATTERNS TO AVOID:
  ❌ Loop + create()      → bulk_create()
  ❌ Loop + save()        → bulk_update() or .update()
  ❌ if queryset:         → .exists()
  ❌ len(queryset)        → .count()
  ❌ FK access in loop    → select_related()
  ❌ M2M access in loop   → prefetch_related()
  ❌ .save() all fields   → .save(update_fields=[...])
  ❌ External API in @atomic → Move outside transaction
```
