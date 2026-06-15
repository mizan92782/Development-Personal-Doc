# MODULE 2: SQL Fundamentals

## 📌 2.1 Aliases (AS)

### ১. কেন শিখবো?

Production SQL-এ complex query-তে alias ছাড়া কোড unreadable হয়ে যায়। Code review-এ reject হবে।

### ২. সহজ ভাষায়

Alias মানে সাময়িক নাম। Table বা column-কে সহজ নামে ডাকা।

```sql
-- Column alias
SELECT
    first_name || ' ' || last_name AS full_name,
    salary * 12                    AS annual_salary,
    department_id                  AS dept
FROM employees;

-- Table alias (JOIN-এ অপরিহার্য)
SELECT
    u.id,
    u.name,
    o.created_at AS order_date,
    o.total
FROM users AS u
JOIN orders AS o ON u.id = o.user_id;

-- Subquery alias
SELECT dept_name, avg_salary
FROM (
    SELECT
        d.name AS dept_name,
        AVG(e.salary) AS avg_salary
    FROM departments d
    JOIN employees e ON d.id = e.department_id
    GROUP BY d.name
) AS department_stats
WHERE avg_salary > 50000;
```

### ৩. Django ORM Example

```python
from django.db.models import F, Value, CharField
from django.db.models.functions import Concat

# Column alias → annotate() দিয়ে
users = User.objects.annotate(
    full_name=Concat('first_name', Value(' '), 'last_name',
                     output_field=CharField()),
    annual_salary=F('salary') * 12
).values('full_name', 'annual_salary')

# Generated SQL:
# SELECT
#   first_name || ' ' || last_name AS full_name,
#   salary * 12 AS annual_salary
# FROM users;
```

---

## 📌 2.2 Aggregate Functions

### ১. কী কী আছে?

```sql
COUNT() → row গণনা করে
SUM()   → যোগফল বের করে
AVG()   → গড় বের করে
MAX()   → সর্বোচ্চ মান
MIN()   → সর্বনিম্ন মান
```

### ২. Real-world Example: E-commerce Analytics

```sql
-- Amazon-এর মতো dashboard query
SELECT
    COUNT(*)                            AS total_orders,
    COUNT(DISTINCT user_id)             AS unique_customers,
    SUM(total_amount)                   AS revenue,
    AVG(total_amount)                   AS avg_order_value,
    MAX(total_amount)                   AS largest_order,
    MIN(total_amount)                   AS smallest_order,
    SUM(total_amount) / COUNT(*)        AS aov  -- Average Order Value
FROM orders
WHERE
    created_at >= NOW() - INTERVAL '30 days'
    AND status = 'completed';
```

```sql
-- NULL handling — এটা জানা জরুরি!
SELECT
    COUNT(*)            AS total_rows,      -- NULL সহ সব গণনা করে
    COUNT(discount)     AS rows_with_discount,  -- NULL বাদ দেয়
    AVG(discount)       AS avg_discount,    -- NULL ignore করে
    COALESCE(AVG(discount), 0) AS safe_avg  -- NULL হলে 0 দেখাবে
FROM orders;
```

### ৩. Django ORM — Aggregate

```python
from django.db.models import Count, Sum, Avg, Max, Min, F
from django.db.models.functions import Coalesce

from datetime import timedelta
from django.utils import timezone

stats = Order.objects.filter(
    created_at__gte=timezone.now() - timedelta(days=30),
    status='completed'
).aggregate(
    total_orders=Count('id'),
    unique_customers=Count('user_id', distinct=True),
    revenue=Sum('total_amount'),
    avg_order_value=Avg('total_amount'),
    largest_order=Max('total_amount'),
    smallest_order=Min('total_amount'),
)

print(stats)
# {'total_orders': 1500, 'revenue': Decimal('750000.00'), ...}

# Generated SQL:
# SELECT
#   COUNT(id) AS total_orders,
#   COUNT(DISTINCT user_id) AS unique_customers,
#   SUM(total_amount) AS revenue,
#   AVG(total_amount) AS avg_order_value,
#   MAX(total_amount) AS largest_order,
#   MIN(total_amount) AS smallest_order
# FROM orders
# WHERE created_at >= '...' AND status = 'completed';
```

---

## 📌 2.3 GROUP BY

### ১. কেন শিখবো?

Data analytics, reporting, business intelligence — সব জায়গায় GROUP BY দরকার। Facebook-এর engagement stats, Uber-এর trip analytics সব GROUP BY দিয়ে।

### ২. Real-world Example: Sales Report

```sql
-- প্রতি category-তে sales কত?
SELECT
    c.name          AS category,
    COUNT(o.id)     AS order_count,
    SUM(oi.total)   AS total_sales,
    AVG(oi.total)   AS avg_sale,
    ROUND(SUM(oi.total) * 100.0 / SUM(SUM(oi.total)) OVER (), 2) AS pct_of_total
FROM order_items oi
JOIN products p     ON oi.product_id = p.id
JOIN categories c   ON p.category_id = c.id
JOIN orders o       ON oi.order_id   = o.id
WHERE o.created_at >= NOW() - INTERVAL '7 days'
GROUP BY c.id, c.name
ORDER BY total_sales DESC;
```

```sql
-- প্রতিদিনের revenue (Netflix-এর subscription analytics)
SELECT
    DATE_TRUNC('day', created_at) AS day,
    COUNT(*)                       AS new_subscriptions,
    SUM(amount)                    AS daily_revenue
FROM subscriptions
WHERE created_at >= NOW() - INTERVAL '30 days'
GROUP BY DATE_TRUNC('day', created_at)
ORDER BY day;
```

### ৩. GROUP BY Rule — সবচেয়ে Common Mistake

```sql
-- ❌ এই query কাজ করবে না:
SELECT name, department, COUNT(*)  -- name SELECT-এ আছে কিন্তু GROUP BY-তে নেই
FROM employees
GROUP BY department;

-- Error: column "employees.name" must appear in the GROUP BY clause

-- ✅ সঠিক:
SELECT department, COUNT(*) AS employee_count
FROM employees
GROUP BY department;

-- অথবা name-ও GROUP BY-তে দাও:
SELECT name, department, COUNT(*) OVER (PARTITION BY department)
FROM employees;  -- Window function দিয়ে সমাধান
```

### ৪. Django ORM — GROUP BY

```python
from django.db.models import Count, Sum, Avg
from django.db.models.functions import TruncDay

# Category-wise sales
sales = OrderItem.objects.values(
    'product__category__name'  # → GROUP BY category
).annotate(
    order_count=Count('order', distinct=True),
    total_sales=Sum('total'),
    avg_sale=Avg('total')
).order_by('-total_sales')

# Generated SQL:
# SELECT categories.name, COUNT(DISTINCT order_id), SUM(total), AVG(total)
# FROM order_items
# JOIN products ON ...
# JOIN categories ON ...
# GROUP BY categories.name
# ORDER BY SUM(total) DESC;

# Daily revenue
daily = Subscription.objects.annotate(
    day=TruncDay('created_at')
).values('day').annotate(
    count=Count('id'),
    revenue=Sum('amount')
).order_by('day')
```

---

## 📌 2.4 HAVING

### ১. GROUP BY vs HAVING

```
WHERE  → raw data filter করে (GROUP BY এর আগে)
HAVING → grouped data filter করে (GROUP BY এর পরে)
```

```sql
-- ভুল বোঝাপড়া:
-- ❌ এটা কাজ করবে না:
SELECT department, COUNT(*) AS count
FROM employees
WHERE COUNT(*) > 5  -- Error! aggregate WHERE-এ ব্যবহার করা যায় না
GROUP BY department;

-- ✅ HAVING ব্যবহার করো:
SELECT department, COUNT(*) AS count
FROM employees
GROUP BY department
HAVING COUNT(*) > 5;
```

### ২. Real-world Example: VIP Customer Detection

```sql
-- Shopee/Daraz-এর মতো: ৩ মাসে ৫+ order দেওয়া customers
SELECT
    u.id,
    u.name,
    u.email,
    COUNT(o.id)     AS order_count,
    SUM(o.total)    AS total_spent
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE o.created_at >= NOW() - INTERVAL '90 days'
  AND o.status = 'completed'
GROUP BY u.id, u.name, u.email
HAVING
    COUNT(o.id) >= 5
    AND SUM(o.total) >= 10000  -- কমপক্ষে ১০,০০০ টাকা খরচ করেছে
ORDER BY total_spent DESC;
```

### ৩. Django ORM — HAVING

```python
from django.db.models import Count, Sum, Q
from datetime import timedelta
from django.utils import timezone

cutoff = timezone.now() - timedelta(days=90)

vip_customers = User.objects.filter(
    orders__created_at__gte=cutoff,
    orders__status='completed'
).annotate(
    order_count=Count('orders'),
    total_spent=Sum('orders__total')
).filter(  # ← এটাই HAVING generate করে
    order_count__gte=5,
    total_spent__gte=10000
).order_by('-total_spent')

# Generated SQL:
# SELECT users.*, COUNT(orders.id) AS order_count, SUM(orders.total) AS total_spent
# FROM users
# JOIN orders ON users.id = orders.user_id
# WHERE orders.created_at >= '...' AND orders.status = 'completed'
# GROUP BY users.id
# HAVING COUNT(orders.id) >= 5 AND SUM(orders.total) >= 10000
# ORDER BY total_spent DESC;
```

---

## 📌 2.5 CASE Statement

### ১. কেন শিখবো?

Conditional logic SQL-এর মধ্যে রাখা। Programming-এর if-else-এর মতো।

### ২. Real-world Example: Order Status Classification

```sql
-- E-commerce order status রিপোর্ট
SELECT
    order_id,
    total_amount,
    created_at,
    CASE status
        WHEN 'pending'    THEN '⏳ Pending'
        WHEN 'processing' THEN '🔄 Processing'
        WHEN 'shipped'    THEN '🚚 Shipped'
        WHEN 'delivered'  THEN '✅ Delivered'
        WHEN 'cancelled'  THEN '❌ Cancelled'
        ELSE '❓ Unknown'
    END AS status_label,

    CASE
        WHEN total_amount < 500   THEN 'Small'
        WHEN total_amount < 2000  THEN 'Medium'
        WHEN total_amount < 10000 THEN 'Large'
        ELSE 'Enterprise'
    END AS order_tier,

    CASE
        WHEN EXTRACT(DOW FROM created_at) IN (0, 6) THEN 'Weekend'
        ELSE 'Weekday'
    END AS day_type
FROM orders
WHERE created_at >= NOW() - INTERVAL '30 days';
```

```sql
-- Conditional aggregation (advanced CASE usage)
-- প্রতি category-তে weekday vs weekend sales comparison
SELECT
    category,
    SUM(CASE WHEN EXTRACT(DOW FROM o.created_at) IN (0,6)
             THEN oi.total ELSE 0 END) AS weekend_sales,
    SUM(CASE WHEN EXTRACT(DOW FROM o.created_at) NOT IN (0,6)
             THEN oi.total ELSE 0 END) AS weekday_sales,
    COUNT(CASE WHEN o.status = 'cancelled' THEN 1 END) AS cancellations
FROM order_items oi
JOIN orders o ON oi.order_id = o.id
JOIN products p ON oi.product_id = p.id
GROUP BY category;
```

### ৩. Django ORM — CASE/WHEN

```python
from django.db.models import Case, When, Value, CharField, IntegerField, Sum

orders = Order.objects.annotate(
    status_label=Case(
        When(status='pending',    then=Value('⏳ Pending')),
        When(status='processing', then=Value('🔄 Processing')),
        When(status='shipped',    then=Value('🚚 Shipped')),
        When(status='delivered',  then=Value('✅ Delivered')),
        default=Value('❓ Unknown'),
        output_field=CharField()
    ),
    order_tier=Case(
        When(total_amount__lt=500,   then=Value('Small')),
        When(total_amount__lt=2000,  then=Value('Medium')),
        When(total_amount__lt=10000, then=Value('Large')),
        default=Value('Enterprise'),
        output_field=CharField()
    )
).values('order_id', 'status_label', 'order_tier')
```

---

# MODULE 3: SQL Joins

## 📌 3.1 JOIN-এর মূল ধারণা

```
JOIN = দুই বা তার বেশি table-এর data একসাথে আনা

সাধারণ ভিজ্যুয়াল:

Table A          Table B
[1, 2, 3, 4]    [3, 4, 5, 6]

INNER JOIN:      [3, 4]         (উভয়ে আছে)
LEFT JOIN:       [1, 2, 3, 4]  (A-এর সব + match)
RIGHT JOIN:      [3, 4, 5, 6]  (B-এর সব + match)
FULL OUTER JOIN: [1, 2, 3, 4, 5, 6] (সব)
CROSS JOIN:      [1×3, 1×4, 1×5...] (Cartesian product)
```

### Sample Data Setup

```sql
-- এই data দিয়ে সব JOIN explain করবো
CREATE TABLE customers (
    id   INT,
    name VARCHAR(50)
);

INSERT INTO customers VALUES
(1, 'Rafi'),
(2, 'Karim'),
(3, 'Sumon'),
(4, 'Mitu');  -- কোনো order নেই

CREATE TABLE orders (
    id          INT,
    customer_id INT,
    amount      DECIMAL
);

INSERT INTO orders VALUES
(101, 1, 500),   -- Rafi-র order
(102, 1, 300),   -- Rafi-র আরেকটি order
(103, 2, 750),   -- Karim-এর order
(104, 5, 200);   -- customer_id=5 exists না
```

---

## 📌 3.2 INNER JOIN

```
শুধু matching rows দেখায়।
```

```sql
SELECT
    c.name AS customer,
    o.id   AS order_id,
    o.amount
FROM customers c
INNER JOIN orders o ON c.id = o.customer_id;

-- ফলাফল:
-- customer | order_id | amount
-- ---------|----------|-------
-- Rafi     | 101      | 500
-- Rafi     | 102      | 300
-- Karim    | 103      | 750
-- Mitu → নেই (order নেই)
-- order 104 → নেই (customer নেই)
```

**Real-world:** User-দের সাথে তাদের orders দেখাতে — শুধু যাদের order আছে।

---

## 📌 3.3 LEFT JOIN (LEFT OUTER JOIN)

```
বাম table-এর সব row + matching right rows।
Match না থাকলে right side NULL।
```

```sql
SELECT
    c.name      AS customer,
    o.id        AS order_id,
    o.amount,
    CASE WHEN o.id IS NULL THEN 'No Orders' ELSE 'Has Orders' END AS status
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id;

-- ফলাফল:
-- customer | order_id | amount | status
-- ---------|----------|--------|----------
-- Rafi     | 101      | 500    | Has Orders
-- Rafi     | 102      | 300    | Has Orders
-- Karim    | 103      | 750    | Has Orders
-- Sumon    | NULL     | NULL   | No Orders  ← LEFT JOIN-এর কারণে
-- Mitu     | NULL     | NULL   | No Orders  ← LEFT JOIN-এর কারণে
```

**Real-world:** সব user দেখাও, যাদের order নেই তারাও (email campaign-এর জন্য)।

### ❗ Common Mistake: LEFT JOIN + WHERE clause

```sql
-- ❌ LEFT JOIN কে INNER JOIN-এ পরিণত করে ফেলছ!
SELECT c.name, o.amount
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE o.amount > 100;  -- NULL rows filter হয়ে যাবে!

-- ✅ সঠিক:
SELECT c.name, o.amount
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE o.amount > 100 OR o.id IS NULL;  -- NULL rows রেখে দাও
-- অথবা condition JOIN condition-এ দাও:
LEFT JOIN orders o ON c.id = o.customer_id AND o.amount > 100
```

---

## 📌 3.4 RIGHT JOIN

```
ডান table-এর সব row + matching left rows।
(LEFT JOIN-এর opposite)
(Production-এ rarely used — LEFT JOIN দিয়ে পুনর্লিখন সহজ)
```

```sql
SELECT
    c.name     AS customer,
    o.id       AS order_id,
    o.amount
FROM customers c
RIGHT JOIN orders o ON c.id = o.customer_id;

-- ফলাফল:
-- customer | order_id | amount
-- ---------|----------|-------
-- Rafi     | 101      | 500
-- Rafi     | 102      | 300
-- Karim    | 103      | 750
-- NULL     | 104      | 200   ← orphan order (customer নেই)
```

> 💡 **Senior Insight:** RIGHT JOIN কে সবসময় LEFT JOIN দিয়ে rewrite করো — readable।
> `A RIGHT JOIN B` = `B LEFT JOIN A`

---

## 📌 3.5 FULL OUTER JOIN

```
উভয় table-এর সব row।
Match না থাকলে opposite side NULL।
```

```sql
SELECT
    c.name     AS customer,
    o.id       AS order_id,
    o.amount
FROM customers c
FULL OUTER JOIN orders o ON c.id = o.customer_id;

-- ফলাফল:
-- customer | order_id | amount
-- ---------|----------|-------
-- Rafi     | 101      | 500
-- Rafi     | 102      | 300
-- Karim    | 103      | 750
-- Sumon    | NULL     | NULL   ← order নেই
-- Mitu     | NULL     | NULL   ← order নেই
-- NULL     | 104      | 200   ← orphan order
```

**Real-world:** Data reconciliation — দুটো system-এর data compare করতে।

---

## 📌 3.6 CROSS JOIN (Cartesian Product)

```
প্রতিটি row-কে অন্য table-এর প্রতিটি row-এর সাথে combine করে।
```

```sql
-- Size: A rows × B rows
SELECT
    color.name AS color,
    size.name  AS size
FROM colors CROSS JOIN sizes;

-- colors: (Red, Blue)  → 2 rows
-- sizes:  (S, M, L)    → 3 rows
-- Result: 2×3 = 6 rows
-- Red-S, Red-M, Red-L, Blue-S, Blue-M, Blue-L
```

**Real-world Use Case:**
- Product variant generation (color × size)
- Report matrix (month × product)
- Test data generation

⚠️ **Warning:** বড় table-এ CROSS JOIN = performance disaster! 1000 × 1000 = 1,000,000 rows!

---

## 📌 3.7 SELF JOIN

```
একই table-কে নিজের সাথে JOIN করা।
Hierarchical বা relationship data-তে ব্যবহার।
```

```sql
-- Employee-Manager hierarchy
CREATE TABLE employees (
    id         INT PRIMARY KEY,
    name       VARCHAR(50),
    manager_id INT REFERENCES employees(id)  -- self reference!
);

INSERT INTO employees VALUES
(1, 'CEO',     NULL),
(2, 'CTO',     1),
(3, 'CFO',     1),
(4, 'Dev Lead',2),
(5, 'Engineer',4);

-- প্রতিটি employee-এর manager কে?
SELECT
    e.name        AS employee,
    m.name        AS manager,
    COALESCE(m.name, 'No Manager') AS reports_to
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;

-- employee  | manager  | reports_to
-- ----------|----------|----------
-- CEO       | NULL     | No Manager
-- CTO       | CEO      | CEO
-- CFO       | CEO      | CEO
-- Dev Lead  | CTO      | CTO
-- Engineer  | Dev Lead | Dev Lead
```

**Real-world:** Facebook friends (user SELF JOIN user), organizational hierarchy, category tree।

---

## 📌 3.8 Multi-table Joins

### Real-world Example: E-commerce Full Order Details

```sql
-- Amazon-এর মতো order detail page
SELECT
    o.id            AS order_id,
    u.name          AS customer_name,
    u.email,
    p.name          AS product_name,
    p.sku,
    c.name          AS category,
    oi.quantity,
    oi.unit_price,
    oi.quantity * oi.unit_price AS line_total,
    o.total_amount,
    o.status,
    o.created_at,
    addr.street     AS delivery_address,
    addr.city,
    courier.name    AS delivery_partner
FROM orders o
JOIN users u            ON o.user_id     = u.id
JOIN order_items oi     ON o.id          = oi.order_id
JOIN products p         ON oi.product_id = p.id
JOIN categories c       ON p.category_id = c.id
LEFT JOIN addresses addr ON o.address_id  = addr.id
LEFT JOIN couriers courier ON o.courier_id = courier.id
WHERE o.id = 12345;
```

### Performance Implication

```
Multi-table JOIN-এ sequence গুরুত্বপূর্ণ:
1. Smallest filtered table প্রথমে
2. Index আছে এমন column-এ JOIN করো
3. EXPLAIN ANALYZE দিয়ে check করো

Query Planner আপনাই optimize করে,
কিন্তু hint দিতে পারো JOIN order দিয়ে।
```

### Django ORM — Multi-table Join

```python
# select_related() → INNER/LEFT JOIN generate করে
order_details = Order.objects.select_related(
    'user',           # JOIN users
    'user__profile',  # JOIN user_profiles
    'address',        # LEFT JOIN addresses
).prefetch_related(
    'items__product__category',  # Separate optimized queries
).get(id=12345)

# Accessing data (no extra queries):
print(order_details.user.name)
print(order_details.user.profile.phone)
print(order_details.address.city)
for item in order_details.items.all():
    print(item.product.name, item.product.category.name)
```

---

## 📌 3.9 JOIN Performance Best Practices

```sql
-- ✅ JOIN column-এ index থাকা জরুরি
CREATE INDEX idx_orders_user_id    ON orders(user_id);
CREATE INDEX idx_order_items_order ON order_items(order_id);
CREATE INDEX idx_products_category ON products(category_id);

-- ✅ Filter আগে করো, JOIN পরে
-- ❌ ভুল: সব join করে তারপর filter
SELECT * FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.created_at > '2024-01-01';

-- ✅ সঠিক: subquery দিয়ে filter আগে করো (large table-এ)
SELECT * FROM (
    SELECT * FROM orders WHERE created_at > '2024-01-01'
) o
JOIN users u ON o.user_id = u.id;
```

### JOIN Interview Questions

**Beginner:**
> Q: INNER JOIN এবং LEFT JOIN-এর পার্থক্য কী?
> A: INNER JOIN শুধু matching rows দেয়। LEFT JOIN বাম table-এর সব rows দেয়, right side-এ match না থাকলে NULL।

**Intermediate:**
> Q: LEFT JOIN করার পর WHERE clause-এ right table-এর column filter করলে কী হয়?
> A: Effectively INNER JOIN হয়ে যায় কারণ NULL rows filter হয়ে যায়।

**Senior/FAANG:**
> Q: তোমার query 5টি table JOIN করছে এবং 10 seconds নিচ্ছে। কীভাবে optimize করবে?
> A: 
> 1. EXPLAIN ANALYZE দিয়ে execution plan দেখবো
> 2. Sequential scan আছে কিনা দেখবো → Index যোগ করবো
> 3. JOIN column-এ index আছে কিনা confirm করবো
> 4. Filter condition-কে subquery-তে push down করবো
> 5. Materialized view বিবেচনা করবো
> 6. Denormalization বিবেচনা করবো

---

# MODULE 4: Advanced SQL

## 📌 4.1 Subqueries

### ১. Subquery কী?

Query-এর মধ্যে আরেকটি query।

```sql
-- সহজ উদাহরণ: Average-এর বেশি salary কাদের?
SELECT name, salary
FROM employees
WHERE salary > (
    SELECT AVG(salary) FROM employees  -- ← subquery
);
```

### ২. WHERE-এ Subquery (Scalar)

```sql
-- Uber-এর মতো: গত মাসে সবচেয়ে বেশি trip করা driver
SELECT
    d.name,
    d.phone,
    (SELECT COUNT(*) FROM trips t
     WHERE t.driver_id = d.id
     AND t.created_at >= NOW() - INTERVAL '30 days') AS monthly_trips
FROM drivers d
WHERE id = (
    SELECT driver_id
    FROM trips
    WHERE created_at >= NOW() - INTERVAL '30 days'
    GROUP BY driver_id
    ORDER BY COUNT(*) DESC
    LIMIT 1
);
```

### ৩. FROM-এ Subquery (Derived Table)

```sql
-- প্রতি user-এর first order (Shopify analytics)
SELECT u.name, first_orders.created_at AS first_order_date
FROM users u
JOIN (
    SELECT user_id, MIN(created_at) AS created_at
    FROM orders
    GROUP BY user_id
) AS first_orders ON u.id = first_orders.user_id;
```

### ৪. SELECT-এ Subquery (Correlated — সাবধান!)

```sql
-- প্রতিটি product-এর total sold
SELECT
    p.name,
    p.price,
    (SELECT COALESCE(SUM(quantity), 0)
     FROM order_items oi
     WHERE oi.product_id = p.id) AS total_sold
FROM products p;

-- ⚠️ এটা correlated subquery — N+1 problem!
-- প্রতিটি product-এর জন্য আলাদা subquery চলে
-- 1000 product = 1001 queries!
-- Solution: JOIN বা Window Function ব্যবহার করো
```

---

## 📌 4.2 Correlated Subqueries

```
Outer query-এর প্রতিটি row-এর জন্য inner query চলে।
Performance-critical — সাবধানে ব্যবহার করো।
```

```sql
-- প্রতি department-এ যারা department average-এর বেশি earn করে
SELECT name, department, salary
FROM employees e1
WHERE salary > (
    SELECT AVG(salary)
    FROM employees e2
    WHERE e2.department = e1.department  -- outer query reference!
);

-- ⚠️ Performance: প্রতিটি employee-এর জন্য subquery চলছে

-- ✅ Better approach with Window Function:
SELECT name, department, salary
FROM (
    SELECT
        name, department, salary,
        AVG(salary) OVER (PARTITION BY department) AS dept_avg
    FROM employees
) t
WHERE salary > dept_avg;
```

---

## 📌 4.3 CTE (Common Table Expression) — WITH clause

### ১. কেন CTE?

```
1. Complex query-কে readable steps-এ ভাগ করা যায়
2. Same subquery বারবার reference করা যায়
3. Recursive query লেখা যায়
4. Temporary result set হিসেবে ব্যবহার করা যায়
```

### ২. Basic CTE

```sql
-- E-commerce: Top 10 customers and their order details
WITH top_customers AS (
    SELECT
        user_id,
        COUNT(*) AS order_count,
        SUM(total) AS total_spent
    FROM orders
    WHERE status = 'completed'
    GROUP BY user_id
    ORDER BY total_spent DESC
    LIMIT 10
),
customer_details AS (
    SELECT
        tc.user_id,
        u.name,
        u.email,
        tc.order_count,
        tc.total_spent
    FROM top_customers tc
    JOIN users u ON tc.user_id = u.id
)
SELECT *
FROM customer_details
ORDER BY total_spent DESC;
```

### ৩. Multiple CTEs

```sql
-- Netflix-এর মতো: User engagement analysis
WITH
watch_stats AS (
    SELECT
        user_id,
        COUNT(DISTINCT content_id) AS unique_content,
        SUM(watch_minutes) AS total_minutes
    FROM watch_history
    WHERE watched_at >= NOW() - INTERVAL '30 days'
    GROUP BY user_id
),
subscription_info AS (
    SELECT user_id, plan_type, expires_at
    FROM subscriptions
    WHERE status = 'active'
),
user_segments AS (
    SELECT
        ws.user_id,
        ws.total_minutes,
        si.plan_type,
        CASE
            WHEN ws.total_minutes > 3000 THEN 'Heavy User'
            WHEN ws.total_minutes > 1000 THEN 'Regular User'
            ELSE 'Light User'
        END AS segment
    FROM watch_stats ws
    JOIN subscription_info si ON ws.user_id = si.user_id
)
SELECT segment, plan_type, COUNT(*) AS user_count
FROM user_segments
GROUP BY segment, plan_type
ORDER BY segment, plan_type;
```

### ৪. Recursive CTE

```sql
-- Organizational hierarchy দেখাও (সব level)
WITH RECURSIVE org_tree AS (
    -- Base case: CEO (top level)
    SELECT id, name, manager_id, 1 AS level, name::TEXT AS path
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive case: reports করা employees
    SELECT
        e.id,
        e.name,
        e.manager_id,
        ot.level + 1,
        ot.path || ' → ' || e.name
    FROM employees e
    JOIN org_tree ot ON e.manager_id = ot.id
)
SELECT
    REPEAT('  ', level - 1) || name AS org_chart,
    level,
    path
FROM org_tree
ORDER BY path;

-- ফলাফল:
-- org_chart         | level | path
-- ------------------|-------|------------------------
-- CEO               | 1     | CEO
--   CTO             | 2     | CEO → CTO
--     Dev Lead      | 3     | CEO → CTO → Dev Lead
--       Engineer    | 4     | CEO → CTO → Dev Lead → Engineer
--   CFO             | 2     | CEO → CFO
```

```sql
-- Facebook-এর মতো: Friend suggestion (mutual connections)
WITH RECURSIVE friend_network AS (
    SELECT friend_id AS user_id, 1 AS degree
    FROM friendships
    WHERE user_id = 1001  -- আমার friends

    UNION ALL

    SELECT f.friend_id, fn.degree + 1
    FROM friendships f
    JOIN friend_network fn ON f.user_id = fn.user_id
    WHERE fn.degree < 3  -- 3 degrees of separation
)
SELECT DISTINCT user_id, MIN(degree) AS connection_degree
FROM friend_network
WHERE user_id != 1001
GROUP BY user_id
ORDER BY connection_degree;
```

### ৫. Django ORM — CTE

```python
# Django-তে native CTE support নেই।
# django-cte package ব্যবহার করো অথবা raw SQL:

from django.db import connection

def get_org_chart():
    with connection.cursor() as cursor:
        cursor.execute("""
            WITH RECURSIVE org_tree AS (
                SELECT id, name, manager_id, 1 AS level
                FROM employees
                WHERE manager_id IS NULL

                UNION ALL

                SELECT e.id, e.name, e.manager_id, ot.level + 1
                FROM employees e
                JOIN org_tree ot ON e.manager_id = ot.id
            )
            SELECT id, name, level
            FROM org_tree
            ORDER BY level, name
        """)
        return cursor.fetchall()
```

---

## 📌 4.4 Window Functions

### ১. কেন Window Functions?

```
সবচেয়ে powerful SQL feature।
GROUP BY aggregate করে কিন্তু row কমায়।
Window function aggregate করে কিন্তু সব row রাখে।
```

```sql
-- তুলনা:
-- GROUP BY:
SELECT department, AVG(salary) FROM employees GROUP BY department;
-- Result: 3 rows (3 departments)

-- Window Function:
SELECT name, department, salary,
    AVG(salary) OVER (PARTITION BY department) AS dept_avg
FROM employees;
-- Result: সব rows, কিন্তু প্রতিটিতে department average আছে!
```

### ২. Window Function Syntax

```sql
function_name() OVER (
    PARTITION BY column1, column2   -- গ্রুপ করো
    ORDER BY column3                -- গ্রুপের মধ্যে order
    ROWS/RANGE BETWEEN ... AND ...  -- frame define করো
)
```

### ৩. Ranking Functions

```sql
-- E-commerce: প্রতি category-তে top products
SELECT
    product_name,
    category,
    sales_count,
    ROW_NUMBER() OVER (PARTITION BY category ORDER BY sales_count DESC) AS row_num,
    RANK()       OVER (PARTITION BY category ORDER BY sales_count DESC) AS rank,
    DENSE_RANK() OVER (PARTITION BY category ORDER BY sales_count DESC) AS dense_rank,
    PERCENT_RANK() OVER (PARTITION BY category ORDER BY sales_count DESC) AS pct_rank
FROM product_sales;

-- পার্থক্য:
-- Data: [100, 100, 80, 70]
-- ROW_NUMBER: [1, 2, 3, 4]  → কোনো tie নেই
-- RANK:       [1, 1, 3, 4]  → tie হলে same rank, gap হয়
-- DENSE_RANK: [1, 1, 2, 3]  → tie হলে same rank, gap নেই
```

```sql
-- Top 3 products per category
SELECT * FROM (
    SELECT
        product_name, category, sales,
        ROW_NUMBER() OVER (PARTITION BY category ORDER BY sales DESC) AS rn
    FROM products
) ranked
WHERE rn <= 3;
```

### ৪. LEAD/LAG — পরের/আগের row access

```sql
-- Revenue month-over-month growth (Netflix, Amazon analytics)
SELECT
    month,
    revenue,
    LAG(revenue) OVER (ORDER BY month) AS prev_month_revenue,
    revenue - LAG(revenue) OVER (ORDER BY month) AS growth,
    ROUND(
        (revenue - LAG(revenue) OVER (ORDER BY month)) * 100.0
        / NULLIF(LAG(revenue) OVER (ORDER BY month), 0),
        2
    ) AS growth_pct
FROM monthly_revenue
ORDER BY month;

-- Output:
-- month      | revenue | prev_month | growth | growth_pct
-- -----------|---------|------------|--------|----------
-- 2024-01    | 100000  | NULL       | NULL   | NULL
-- 2024-02    | 120000  | 100000     | 20000  | 20.00
-- 2024-03    | 110000  | 120000     | -10000 | -8.33
```

```sql
-- LEAD: পরের row-এর value
SELECT
    user_id,
    action,
    created_at,
    LEAD(action) OVER (PARTITION BY user_id ORDER BY created_at) AS next_action,
    LEAD(created_at) OVER (PARTITION BY user_id ORDER BY created_at) - created_at
        AS time_to_next_action
FROM user_events;
-- Session analysis, funnel analysis-এ ব্যবহার
```

### ৫. Running Total / Cumulative Sum

```sql
-- Daily cumulative revenue
SELECT
    order_date,
    daily_revenue,
    SUM(daily_revenue) OVER (
        ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS cumulative_revenue,
    AVG(daily_revenue) OVER (
        ORDER BY order_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW  -- 7-day moving average
    ) AS moving_avg_7day
FROM daily_sales;
```

### ৬. PARTITION BY বিস্তারিত

```sql
-- প্রতি customer-এর order-গুলো rank করো এবং running total রাখো
SELECT
    customer_id,
    order_id,
    order_date,
    amount,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date) AS order_seq,
    SUM(amount)  OVER (PARTITION BY customer_id ORDER BY order_date) AS cumulative_spend,
    AVG(amount)  OVER (PARTITION BY customer_id) AS avg_order_value
FROM orders;
```

### ৭. Django ORM — Window Functions

```python
from django.db.models import Window, F, Sum, Avg
from django.db.models.functions import Rank, RowNumber, Lag

# Month-over-month revenue
from django.db.models import Window

revenue_trend = MonthlyRevenue.objects.annotate(
    prev_revenue=Window(
        expression=Lag('revenue', offset=1),
        order_by=F('month').asc()
    ),
    cumulative=Window(
        expression=Sum('revenue'),
        order_by=F('month').asc()
    )
).values('month', 'revenue', 'prev_revenue', 'cumulative')

# Product ranking per category
from django.db.models.functions import DenseRank

ranked_products = Product.objects.annotate(
    category_rank=Window(
        expression=DenseRank(),
        partition_by=[F('category')],
        order_by=F('sales_count').desc()
    )
).filter(category_rank__lte=3)
```

---

## 📌 4.5 UNION vs UNION ALL

```sql
-- UNION: duplicate remove করে (DISTINCT)
-- UNION ALL: সব রাখে (faster!)

-- উদাহরণ: সব transaction history
SELECT user_id, amount, 'purchase' AS type, created_at FROM purchases
UNION ALL
SELECT user_id, amount, 'refund'   AS type, created_at FROM refunds
UNION ALL
SELECT user_id, amount, 'transfer' AS type, created_at FROM transfers
ORDER BY created_at DESC;

-- ✅ UNION ALL prefer করো (no dedup step → faster)
-- ❌ UNION শুধু তখন যখন duplicate সত্যিই remove করতে হবে
```

---

## 📌 4.6 EXISTS vs IN vs ANY vs ALL

```sql
-- EXISTS: row exist করে কিনা check করে (fastest for large datasets)
SELECT * FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.customer_id = c.id
    AND o.total > 1000
);

-- IN: value list-এ আছে কিনা
SELECT * FROM products
WHERE category_id IN (1, 2, 3);

-- IN with subquery (সাবধান!)
SELECT * FROM customers
WHERE id IN (
    SELECT DISTINCT customer_id FROM orders WHERE total > 1000
);

-- ANY: subquery-র যেকোনো value match করলে
SELECT name FROM employees
WHERE salary > ANY (SELECT salary FROM managers);  -- কোনো manager-এর চেয়ে বেশি

-- ALL: subquery-র সব value match করলে
SELECT name FROM employees
WHERE salary > ALL (SELECT salary FROM managers);  -- সব manager-এর চেয়ে বেশি
```

### EXISTS vs IN — কোনটা কখন?

```
EXISTS:
✅ Large subquery result-এ faster (short-circuit করে)
✅ NULL handling safe
✅ Correlated subquery-তে best

IN:
✅ Small, fixed list-এ fast
✅ Simpler to read
❌ NULL-এ সমস্যা: IN (1, 2, NULL) → unexpected results
❌ Large subquery-তে slow

Performance Rule:
- NOT IN → বিপজ্জনক (NULL issue) → NOT EXISTS use করো
- EXISTS → large subquery-তে
- IN → small fixed list বা small subquery-তে
```

```sql
-- ❌ ভুল: NOT IN with possible NULLs
SELECT * FROM orders WHERE customer_id NOT IN (
    SELECT id FROM customers WHERE country = 'BD'
    -- যদি কোনো id NULL হয়, সব query empty return করবে!
);

-- ✅ সঠিক: NOT EXISTS
SELECT * FROM orders o WHERE NOT EXISTS (
    SELECT 1 FROM customers c
    WHERE c.id = o.customer_id AND c.country = 'BD'
);
```

---

## 📊 MODULE 2-4 Summary Sheet

```
╔══════════════════════════════════════════════════════════════╗
║           MODULE 2-4: SQL FUNDAMENTALS & ADVANCED           ║
╠══════════════════════════════════════════════════════════════╣
║ AGGREGATE FUNCTIONS                                          ║
║  COUNT(*) vs COUNT(col) → NULL handling!                     ║
║  GROUP BY → aggregate করো                                   ║
║  HAVING → grouped result filter করো                         ║
╠══════════════════════════════════════════════════════════════╣
║ JOINS                                                        ║
║  INNER → only matches                                        ║
║  LEFT  → all left + matches                                  ║
║  FULL  → all from both                                       ║
║  CROSS → cartesian (dangerous with large tables!)            ║
║  SELF  → hierarchy, relationships                            ║
╠══════════════════════════════════════════════════════════════╣
║ ADVANCED SQL                                                 ║
║  Subquery → simple but beware correlated = N+1               ║
║  CTE (WITH) → readable, reusable                             ║
║  Recursive CTE → hierarchy, graphs                           ║
║  Window Functions → most powerful feature                    ║
║    ROW_NUMBER, RANK, DENSE_RANK                              ║
║    LEAD, LAG → adjacent rows                                 ║
║    PARTITION BY → group without losing rows                  ║
╠══════════════════════════════════════════════════════════════╣
║ PERFORMANCE RULES                                            ║
║  UNION ALL > UNION (no dedup)                                ║
║  EXISTS > IN for large subqueries                            ║
║  NOT EXISTS > NOT IN (NULL safety)                           ║
║  Window Functions > Correlated Subqueries                    ║
╚══════════════════════════════════════════════════════════════╝
```

## 🎯 MODULE 2-4 Exercises

```sql
-- Exercise 1: Amazon Sales Dashboard
-- গত ৭ দিনের daily sales, cumulative revenue, 
-- এবং day-over-day growth % বের করো

-- Exercise 2: Top N per Group
-- প্রতি category-তে top 3 best-selling products বের করো
-- Window function ব্যবহার করো

-- Exercise 3: Customer Segmentation
-- RFM Analysis (Recency, Frequency, Monetary):
-- R: সর্বশেষ order কত দিন আগে
-- F: মোট কতটি order
-- M: মোট কত টাকার order
-- প্রতিটিতে 1-5 score দাও এবং segment করো

-- Exercise 4: Hierarchy Query
-- Employee table-এ recursive CTE দিয়ে
-- সম্পূর্ণ org chart বের করো (সব levels)

-- Exercise 5: Django ORM
-- উপরের ৪টি exercise Django ORM-এ করো
-- Generated SQL EXPLAIN দিয়ে দেখো
```

---
