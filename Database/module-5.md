# MODULE 5: Database Design
## বাংলায় ডাটাবেজ ডিজাইন — সম্পূর্ণ গাইড

---

## 5.1 Normalization (নরমালাইজেশন)

### কেন শিখব?

তুমি যদি একটা e-commerce সাইট বানাও এবং database design ঠিকমতো না করো, তাহলে:
- একই data বারবার store হবে
- data update করলে inconsistency আসবে
- query করতে গেলে performance খারাপ হবে
- bugs ধরা কঠিন হবে

Senior engineer রা সবসময় বলে: **"Bad schema design is the root of 80% of backend performance problems."**

---

### Normalization কী?

Normalization হলো database table গুলোকে এমনভাবে organize করা যাতে:
1. **Data redundancy** কমে
2. **Data anomalies** না আসে
3. **Data integrity** বজায় থাকে

```
Data Anomalies ৩ ধরনের:

1. Insert Anomaly  → নতুন data insert করতে গেলে সমস্যা
2. Update Anomaly  → এক জায়গায় update করলে অন্য জায়গায় আপডেট হয় না
3. Delete Anomaly  → একটা record delete করলে অন্য দরকারি data মুছে যায়
```

---

### 1NF — First Normal Form

**Rule:**
- প্রতিটি column এ atomic (অবিভাজ্য) value থাকবে
- কোনো repeating group থাকবে না
- প্রতিটি row unique হবে (Primary Key থাকবে)

**❌ 1NF Violation — খারাপ Design:**

```
orders table:
| order_id | customer_name | products                        |
|----------|---------------|---------------------------------|
| 1        | Rahim         | "iPhone, AirPods, Case"         |
| 2        | Karim         | "Samsung TV"                    |
```

এখানে `products` column এ multiple value আছে — এটা atomic না।

**✅ 1NF Correct Design:**

```sql
CREATE TABLE orders (
    order_id    INT,
    customer_name VARCHAR(100),
    product     VARCHAR(100),
    PRIMARY KEY (order_id, product)
);

-- Data:
| order_id | customer_name | product   |
|----------|---------------|-----------|
| 1        | Rahim         | iPhone    |
| 1        | Rahim         | AirPods   |
| 1        | Rahim         | Case      |
| 2        | Karim         | Samsung TV|
```

**Django ORM Example:**

```python
# Bad model (1NF violation)
class Order(models.Model):
    customer_name = models.CharField(max_length=100)
    products = models.TextField()  # "iPhone, AirPods" — WRONG!

# Good model (1NF compliant)
class Order(models.Model):
    customer_name = models.CharField(max_length=100)

class OrderItem(models.Model):
    order = models.ForeignKey(Order, on_delete=models.CASCADE)
    product = models.CharField(max_length=100)
```

---

### 2NF — Second Normal Form

**Rule:**
- 1NF মেনে চলতে হবে
- প্রতিটি non-key column কে **full primary key** এর উপর depend করতে হবে
- Partial dependency থাকবে না (composite key এর ক্ষেত্রে প্রযোজ্য)

**❌ 2NF Violation:**

```sql
-- Composite PK: (order_id, product_id)
CREATE TABLE order_items (
    order_id      INT,
    product_id    INT,
    quantity      INT,
    product_name  VARCHAR(100),  -- শুধু product_id এর উপর depend করে!
    customer_name VARCHAR(100),  -- শুধু order_id এর উপর depend করে!
    PRIMARY KEY (order_id, product_id)
);
```

`product_name` শুধু `product_id` এর উপর depend করে — এটা partial dependency।
`customer_name` শুধু `order_id` এর উপর depend করে — এটাও partial dependency।

**✅ 2NF Correct Design:**

```sql
CREATE TABLE products (
    product_id   INT PRIMARY KEY,
    product_name VARCHAR(100)
);

CREATE TABLE orders (
    order_id      INT PRIMARY KEY,
    customer_name VARCHAR(100)
);

CREATE TABLE order_items (
    order_id   INT REFERENCES orders(order_id),
    product_id INT REFERENCES products(product_id),
    quantity   INT,
    PRIMARY KEY (order_id, product_id)
);
```

**Django ORM:**

```python
class Product(models.Model):
    name = models.CharField(max_length=100)

class Order(models.Model):
    customer_name = models.CharField(max_length=100)

class OrderItem(models.Model):
    order   = models.ForeignKey(Order, on_delete=models.CASCADE)
    product = models.ForeignKey(Product, on_delete=models.CASCADE)
    quantity = models.PositiveIntegerField()

    class Meta:
        unique_together = ('order', 'product')
```

---

### 3NF — Third Normal Form

**Rule:**
- 2NF মেনে চলতে হবে
- কোনো **transitive dependency** থাকবে না
- Non-key column অন্য non-key column এর উপর depend করবে না

**❌ 3NF Violation:**

```sql
CREATE TABLE employees (
    emp_id       INT PRIMARY KEY,
    emp_name     VARCHAR(100),
    dept_id      INT,
    dept_name    VARCHAR(100),  -- dept_id এর উপর depend করে, emp_id এর উপর না!
    dept_manager VARCHAR(100)   -- dept_id এর উপর depend করে!
);
```

এখানে: `emp_id → dept_id → dept_name` — এটা transitive dependency।

**✅ 3NF Correct Design:**

```sql
CREATE TABLE departments (
    dept_id      INT PRIMARY KEY,
    dept_name    VARCHAR(100),
    dept_manager VARCHAR(100)
);

CREATE TABLE employees (
    emp_id   INT PRIMARY KEY,
    emp_name VARCHAR(100),
    dept_id  INT REFERENCES departments(dept_id)
);
```

**Real-world Example — Amazon:**

Amazon এর product catalog এ:
```
product → category_id → category_name (transitive dependency)
```

তাই Amazon আলাদা `categories` table রাখে।

**Django ORM:**

```python
class Department(models.Model):
    name    = models.CharField(max_length=100)
    manager = models.CharField(max_length=100)

class Employee(models.Model):
    name       = models.CharField(max_length=100)
    department = models.ForeignKey(Department, on_delete=models.SET_NULL, null=True)
```

**Generated SQL by Django ORM:**
```sql
SELECT "employees"."id", "employees"."name", "employees"."department_id"
FROM "employees"
INNER JOIN "departments" ON ("employees"."department_id" = "departments"."id");
```

---

### BCNF — Boyce-Codd Normal Form

**Rule:**
- 3NF এর একটু শক্তিশালী version
- প্রতিটি functional dependency X → Y তে X কে superkey হতে হবে

**❌ BCNF Violation Example:**

```
একটা university তে:
- একজন student একটা subject এ একজনই teacher পায়
- একজন teacher শুধু একটাই subject পড়ায়

Table: (student, subject, teacher)
PK: (student, subject)

student → subject (via teacher: student → teacher → subject)
```

```sql
-- Problematic table
CREATE TABLE student_subject (
    student VARCHAR(50),
    subject VARCHAR(50),
    teacher VARCHAR(50),
    PRIMARY KEY (student, subject)
);
```

**✅ BCNF Fix:**

```sql
CREATE TABLE teacher_subject (
    teacher VARCHAR(50) PRIMARY KEY,
    subject VARCHAR(50)
);

CREATE TABLE student_teacher (
    student VARCHAR(50),
    teacher VARCHAR(50) REFERENCES teacher_subject(teacher),
    PRIMARY KEY (student, teacher)
);
```

---

### 4NF — Fourth Normal Form

**Rule:**
- BCNF মেনে চলতে হবে
- কোনো **multi-valued dependency** থাকবে না

**❌ 4NF Violation:**

```sql
-- একজন employee এর multiple skills এবং multiple languages আছে
CREATE TABLE emp_skills_languages (
    emp_id   INT,
    skill    VARCHAR(50),
    language VARCHAR(50)
    -- emp_id →→ skill এবং emp_id →→ language (multi-valued)
);

-- Data:
| emp_id | skill   | language |
|--------|---------|----------|
| 1      | Python  | English  |
| 1      | Python  | Bengali  |
| 1      | Java    | English  |
| 1      | Java    | Bengali  |
-- Redundancy! Python আর Java দুইবার করে আসছে
```

**✅ 4NF Fix:**

```sql
CREATE TABLE emp_skills (
    emp_id INT,
    skill  VARCHAR(50),
    PRIMARY KEY (emp_id, skill)
);

CREATE TABLE emp_languages (
    emp_id   INT,
    language VARCHAR(50),
    PRIMARY KEY (emp_id, language)
);
```

---

### 5NF — Fifth Normal Form

5NF (PJNF — Project-Join Normal Form) খুবই advanced। Production এ খুব কম দেখা যায়।

**Rule:** Join dependency eliminate করতে হবে।

> **Senior Engineer Tip:** 5NF বেশিরভাগ production system এ দরকার হয় না। 3NF বা BCNF পর্যন্ত গেলেই যথেষ্ট। Performance এর জন্য কখনো কখনো denormalize করতে হয়।


---

## 5.2 Denormalization (ডিনরমালাইজেশন)

### কেন দরকার?

Normalization data redundancy কমায়, কিন্তু **JOIN** এর সংখ্যা বাড়িয়ে দেয়। বড় system এ অনেক JOIN মানে slow query।

**Netflix, Amazon, Uber** এর মতো company রা intentionally denormalize করে read performance বাড়ায়।

---

### Denormalization কখন করবে?

```
✅ করবে যখন:
- Read > Write (read-heavy system)
- JOIN অনেক বেশি হচ্ছে এবং query slow
- Reporting/Analytics system
- Data warehouse

❌ করবে না যখন:
- Write-heavy system
- Data consistency খুব জরুরি (banking, payment)
- Data size ছোট
```

---

### Real-world Example — Facebook Post Feed

**Normalized (3NF):**

```sql
-- 4টা table JOIN করতে হয় feed দেখাতে
SELECT
    p.content,
    u.name,
    u.profile_pic,
    COUNT(l.id) AS likes
FROM posts p
JOIN users u ON p.user_id = u.id
JOIN likes l ON l.post_id = p.id
JOIN follows f ON f.following_id = p.user_id
WHERE f.follower_id = 123
GROUP BY p.id, u.name, u.profile_pic;
```

প্রতিটা feed request এ এই query চললে millions of users এর জন্য database die করবে।

**Denormalized (Production approach):**

```sql
-- feed_items table — precomputed, denormalized
CREATE TABLE feed_items (
    id           BIGSERIAL PRIMARY KEY,
    user_id      INT,         -- যার feed এ দেখাবে
    post_id      INT,
    post_content TEXT,
    author_name  VARCHAR(100), -- denormalized from users
    author_pic   VARCHAR(255), -- denormalized from users
    like_count   INT,          -- denormalized, periodically updated
    created_at   TIMESTAMPTZ
);

-- এখন feed query:
SELECT * FROM feed_items
WHERE user_id = 123
ORDER BY created_at DESC
LIMIT 20;
-- কোনো JOIN নেই!
```

---

### Denormalization Techniques

**1. Storing Derived/Computed Data:**

```sql
-- orders table এ total_amount store করা
CREATE TABLE orders (
    id           BIGSERIAL PRIMARY KEY,
    customer_id  INT,
    total_amount DECIMAL(10,2),  -- computed, denormalized
    item_count   INT             -- computed, denormalized
);
```

```python
# Django ORM
class Order(models.Model):
    customer    = models.ForeignKey(Customer, on_delete=models.CASCADE)
    total_amount = models.DecimalField(max_digits=10, decimal_places=2)
    item_count  = models.PositiveIntegerField(default=0)

    def recalculate_totals(self):
        from django.db.models import Sum, Count
        result = self.items.aggregate(
            total=Sum('price'),
            count=Count('id')
        )
        self.total_amount = result['total'] or 0
        self.item_count   = result['count'] or 0
        self.save(update_fields=['total_amount', 'item_count'])
```

**Generated SQL:**
```sql
UPDATE "orders"
SET "total_amount" = 1500.00, "item_count" = 3
WHERE "orders"."id" = 1;
```

---

**2. Duplicating Columns (Column Redundancy):**

```sql
-- order_items তে product_name copy করা রাখা
-- কারণ: product name পরে বদলালেও order history ঠিক থাকবে
CREATE TABLE order_items (
    id           BIGSERIAL PRIMARY KEY,
    order_id     INT,
    product_id   INT,
    product_name VARCHAR(100),  -- denormalized snapshot
    unit_price   DECIMAL(10,2), -- denormalized snapshot
    quantity     INT
);
```

> **Production Insight:** E-commerce এ এটা mandatory। একটা order place হওয়ার পর product এর price বা name বদলালে old order এ সেই পুরনো price দেখাতে হবে। তাই snapshot store করতে হয়।

---

**3. Summary Tables:**

```sql
-- Daily sales summary — analytics এর জন্য
CREATE TABLE daily_sales_summary (
    date         DATE PRIMARY KEY,
    total_orders INT,
    total_revenue DECIMAL(12,2),
    avg_order_value DECIMAL(10,2)
);

-- Cron job বা trigger দিয়ে populate করা হয়
```

---

### Denormalization Trade-offs

```
+---------------------------+----------------------------------+
| সুবিধা                    | অসুবিধা                          |
+---------------------------+----------------------------------+
| Read query faster          | Data redundancy বাড়ে            |
| Fewer JOINs                | Update করলে multiple place update|
| Simple queries             | Storage বেশি লাগে               |
| Better cache hit rate      | Inconsistency risk বাড়ে        |
+---------------------------+----------------------------------+
```

---

## 5.3 ER Diagram (Entity-Relationship Diagram)

### কী এবং কেন?

ER Diagram হলো database এর visual blueprint। Code লেখার আগে schema design করতে এটা ব্যবহার হয়।

**Components:**
```
Entity (Table)     → Rectangle box
Attribute (Column) → Oval/এলিপস
Relationship       → Diamond
PK                 → Underlined attribute
FK                 → Arrow/connection line
```

---

### Cardinality (সম্পর্কের ধরন)

```
1. One-to-One (1:1)
   একজন User এর একটাই Profile
   User ──── Profile

2. One-to-Many (1:N)
   একটা Order এ অনেক OrderItem
   Order ────< OrderItems

3. Many-to-Many (M:N)
   একজন Student অনেক Course নিতে পারে
   একটা Course এ অনেক Student থাকতে পারে
   Student >────< Course
   (Junction table দরকার: student_courses)
```

---

### Real-world ER Diagram — E-Commerce System

```
+------------+         +------------+         +------------+
|  customers |         |   orders   |         | order_items|
+------------+         +------------+         +------------+
| PK id      |1      N | PK id      |1      N | PK id      |
| name       |─────────| FK cust_id |─────────| FK order_id|
| email      |         | status     |         | FK prod_id |
| phone      |         | total_amt  |         | quantity   |
| created_at |         | created_at |         | unit_price |
+------------+         +------------+         +-------+----+
                                                       |N
                                                       |
                                               +-------+----+
                                               |  products  |
                                               +------------+
                                               | PK id      |
                                               | name       |
                                               | FK cat_id  |──N──+
                                               | price      |     |1
                                               | stock_qty  |  +--+--------+
                                               +------------+  | categories |
                                                               +------------+
                                                               | PK id      |
                                                               | name       |
                                                               +------------+
```

**SQL Schema:**

```sql
CREATE TABLE customers (
    id         BIGSERIAL PRIMARY KEY,
    name       VARCHAR(100) NOT NULL,
    email      VARCHAR(150) UNIQUE NOT NULL,
    phone      VARCHAR(20),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE categories (
    id   SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

CREATE TABLE products (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(200) NOT NULL,
    category_id INT REFERENCES categories(id),
    price       DECIMAL(10,2) NOT NULL CHECK (price >= 0),
    stock_qty   INT NOT NULL DEFAULT 0
);

CREATE TABLE orders (
    id          BIGSERIAL PRIMARY KEY,
    customer_id BIGINT REFERENCES customers(id),
    status      VARCHAR(20) DEFAULT 'pending',
    total_amount DECIMAL(12,2),
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE order_items (
    id         BIGSERIAL PRIMARY KEY,
    order_id   BIGINT REFERENCES orders(id) ON DELETE CASCADE,
    product_id BIGINT REFERENCES products(id),
    quantity   INT NOT NULL CHECK (quantity > 0),
    unit_price DECIMAL(10,2) NOT NULL  -- snapshot at purchase time
);
```

**Django ORM Models:**

```python
class Category(models.Model):
    name = models.CharField(max_length=100)

class Product(models.Model):
    name      = models.CharField(max_length=200)
    category  = models.ForeignKey(Category, on_delete=models.SET_NULL, null=True)
    price     = models.DecimalField(max_digits=10, decimal_places=2)
    stock_qty = models.PositiveIntegerField(default=0)

class Customer(models.Model):
    name       = models.CharField(max_length=100)
    email      = models.EmailField(unique=True)
    phone      = models.CharField(max_length=20, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)

class Order(models.Model):
    STATUS_CHOICES = [
        ('pending', 'Pending'),
        ('confirmed', 'Confirmed'),
        ('shipped', 'Shipped'),
        ('delivered', 'Delivered'),
    ]
    customer     = models.ForeignKey(Customer, on_delete=models.PROTECT)
    status       = models.CharField(max_length=20, choices=STATUS_CHOICES, default='pending')
    total_amount = models.DecimalField(max_digits=12, decimal_places=2, null=True)
    created_at   = models.DateTimeField(auto_now_add=True)

class OrderItem(models.Model):
    order      = models.ForeignKey(Order, on_delete=models.CASCADE, related_name='items')
    product    = models.ForeignKey(Product, on_delete=models.PROTECT)
    quantity   = models.PositiveIntegerField()
    unit_price = models.DecimalField(max_digits=10, decimal_places=2)
```


---

## 5.4 Junction Tables (জাংশন টেবিল)

### কী এবং কেন?

Many-to-Many relationship handle করতে junction table (associative table / bridge table) ব্যবহার হয়।

---

### Example 1 — Students & Courses

```
Student >────< Course
```

```sql
CREATE TABLE students (
    id   BIGSERIAL PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE courses (
    id    BIGSERIAL PRIMARY KEY,
    title VARCHAR(200)
);

-- Junction table
CREATE TABLE enrollments (
    student_id  BIGINT REFERENCES students(id) ON DELETE CASCADE,
    course_id   BIGINT REFERENCES courses(id)  ON DELETE CASCADE,
    enrolled_at TIMESTAMPTZ DEFAULT NOW(),
    grade       CHAR(2),
    PRIMARY KEY (student_id, course_id)
);
```

**Django ManyToManyField with through:**

```python
class Student(models.Model):
    name    = models.CharField(max_length=100)
    courses = models.ManyToManyField('Course', through='Enrollment')

class Course(models.Model):
    title = models.CharField(max_length=200)

class Enrollment(models.Model):
    student     = models.ForeignKey(Student, on_delete=models.CASCADE)
    course      = models.ForeignKey(Course, on_delete=models.CASCADE)
    enrolled_at = models.DateTimeField(auto_now_add=True)
    grade       = models.CharField(max_length=2, blank=True)

    class Meta:
        unique_together = ('student', 'course')
```

**Query Example:**

```python
# একজন student এর সব courses
student = Student.objects.get(id=1)
courses = student.courses.all()

# Generated SQL:
# SELECT courses.* FROM courses
# INNER JOIN enrollments ON courses.id = enrollments.course_id
# WHERE enrollments.student_id = 1;

# Grade সহ query
enrollments = Enrollment.objects.filter(
    student_id=1
).select_related('course')
```

---

### Example 2 — Products & Tags (Real E-commerce)

```sql
CREATE TABLE tags (
    id   SERIAL PRIMARY KEY,
    name VARCHAR(50) UNIQUE
);

CREATE TABLE product_tags (
    product_id BIGINT REFERENCES products(id) ON DELETE CASCADE,
    tag_id     INT    REFERENCES tags(id)     ON DELETE CASCADE,
    PRIMARY KEY (product_id, tag_id)
);
```

```python
class Tag(models.Model):
    name = models.CharField(max_length=50, unique=True)

class Product(models.Model):
    name = models.CharField(max_length=200)
    tags = models.ManyToManyField(Tag, blank=True)

# Generated junction table: product_tags (automatic)
```

**Query:**
```python
# "electronics" tag এর সব products
products = Product.objects.filter(tags__name='electronics')

# Generated SQL:
# SELECT products.* FROM products
# INNER JOIN product_tags ON products.id = product_tags.product_id
# INNER JOIN tags ON product_tags.tag_id = tags.id
# WHERE tags.name = 'electronics';
```

---

### Example 3 — Uber: Driver & Vehicle Assignment

```sql
CREATE TABLE drivers (
    id   BIGSERIAL PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE vehicles (
    id           BIGSERIAL PRIMARY KEY,
    license_plate VARCHAR(20) UNIQUE
);

-- Junction: একজন driver একাধিক vehicle চালাতে পারে
-- একটা vehicle একাধিক driver চালাতে পারে (shift system)
CREATE TABLE driver_vehicle_assignments (
    driver_id  BIGINT REFERENCES drivers(id),
    vehicle_id BIGINT REFERENCES vehicles(id),
    start_date DATE,
    end_date   DATE,
    PRIMARY KEY (driver_id, vehicle_id, start_date)
);
```

---

## 5.5 Schema Design Mistakes (সাধারণ ভুলগুলো)

### ভুল ১: Generic/EAV Table Design

```sql
-- ❌ খুব খারাপ design (EAV - Entity Attribute Value)
CREATE TABLE properties (
    entity_id  INT,
    attr_name  VARCHAR(50),
    attr_value TEXT
);

-- Data:
| entity_id | attr_name | attr_value |
|-----------|-----------|------------|
| 1         | name      | iPhone 15  |
| 1         | price     | 1200       |
| 1         | color     | Black      |
```

**সমস্যা:** Type safety নেই, query করা কঠিন, index ঠিকমতো কাজ করে না।

**✅ সঠিক পদ্ধতি:** Proper columns অথবা PostgreSQL JSONB ব্যবহার করো।

---

### ভুল ২: Storing Comma-Separated Values

```sql
-- ❌ ভুল
CREATE TABLE posts (
    id       SERIAL PRIMARY KEY,
    tag_ids  VARCHAR(200)  -- "1,5,8,12"
);

-- ✅ সঠিক — junction table ব্যবহার করো
```

---

### ভুল ৩: না ভেবে NULL ব্যবহার

```sql
-- ❌ অর্থহীন NULL
CREATE TABLE users (
    id           SERIAL PRIMARY KEY,
    name         VARCHAR(100),
    phone        VARCHAR(20),      -- NULL মানে কী? Unknown নাকি Not Applicable?
    deleted_at   TIMESTAMPTZ       -- soft delete এর জন্য NULL OK
);
```

> Rule: NULL এর meaning স্পষ্ট হওয়া উচিত। "Unknown" এবং "Not applicable" আলাদা।

---

### ভুল ৪: Over-normalization

```sql
-- ❌ অতিরিক্ত normalization — 10টা JOIN লাগছে
-- একটা simple report এর জন্য 8-10 টা table join করতে হচ্ছে
-- Production এ এটা slow হবে
```

> **Balance খোঁজো:** 3NF পর্যন্ত normalize করো, তারপর performance দেখে denormalize করো।

---

### ভুল ৫: VARCHAR(MAX) সর্বত্র ব্যবহার

```sql
-- ❌ ভুল
CREATE TABLE users (
    name  VARCHAR(5000),  -- নামের জন্য 5000?
    email VARCHAR(5000)
);

-- ✅ সঠিক — realistic size দাও
CREATE TABLE users (
    name  VARCHAR(100),
    email VARCHAR(254)   -- RFC 5321 standard
);
```

---

### ভুল ৬: Soft Delete ভুলভাবে

```sql
-- ❌ is_deleted = TRUE দিয়ে সব query তে WHERE is_deleted = FALSE দিতে ভুলে যাওয়া

-- ✅ Django এ সঠিক পদ্ধতি
class SoftDeleteManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(deleted_at__isnull=True)

class BaseModel(models.Model):
    deleted_at = models.DateTimeField(null=True, blank=True)
    objects    = SoftDeleteManager()
    all_objects = models.Manager()

    def soft_delete(self):
        from django.utils import timezone
        self.deleted_at = timezone.now()
        self.save(update_fields=['deleted_at'])

    class Meta:
        abstract = True
```

---

## 5.6 Performance Implications

### Normalization এর Performance Impact

```
Query Type    | Normalized | Denormalized
--------------+------------+--------------
Simple SELECT | Fast        | Fast
JOIN queries  | Slower      | Faster
INSERT/UPDATE | Faster      | Slower (update multiple places)
Storage       | Less        | More
Consistency   | High        | Lower risk
```

### Index Strategy with Normalized Schema

```sql
-- Foreign key গুলোতে সবসময় index দাও
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_order_items_order_id ON order_items(order_id);
CREATE INDEX idx_order_items_product_id ON order_items(product_id);

-- Junction table এর FK গুলোতে index
CREATE INDEX idx_enrollments_student_id ON enrollments(student_id);
CREATE INDEX idx_enrollments_course_id  ON enrollments(course_id);
```

---

## 5.7 Production Best Practices

```
1. সবসময় surrogate PK (BIGSERIAL/UUID) ব্যবহার করো
2. FK তে সবসময় index দাও
3. created_at, updated_at সব table এ রাখো
4. soft delete এর জন্য deleted_at ব্যবহার করো
5. Enum এর পরিবর্তে lookup table ব্যবহার করো (বড় system এ)
6. CHECK constraint দিয়ে data validation করো
7. NOT NULL constraint যথাসম্ভব ব্যবহার করো
8. Schema change এর জন্য migration ব্যবহার করো (Django migrations)
9. Production এ schema deploy করার আগে EXPLAIN দিয়ে check করো
10. Denormalize করলে sync strategy ঠিক করো (trigger/celery task)
```

---

## 5.8 Interview Questions

### Beginner Level

**Q1: Normalization কী এবং কেন দরকার?**

> Normalization হলো database table গুলোকে সুসংগঠিত করার প্রক্রিয়া যাতে data redundancy কমে এবং data integrity বাড়ে। এটা insert, update, delete anomaly প্রতিরোধ করে।

**Q2: 1NF, 2NF, 3NF এর মূল পার্থক্য কী?**

> - 1NF: Atomic values, no repeating groups
> - 2NF: 1NF + no partial dependency on composite PK
> - 3NF: 2NF + no transitive dependency

**Q3: Junction table কখন লাগে?**

> Many-to-many relationship handle করতে। যেমন: Student-Course, Product-Tag, User-Role।

---

### Intermediate Level

**Q4: Denormalization কখন করবে?**

> Read-heavy system এ, যেখানে JOIN অনেক বেশি এবং query slow হচ্ছে। Analytics, reporting, feed systems এ সাধারণত denormalize করা হয়। Trade-off হলো data consistency এর risk বাড়ে।

**Q5: BCNF vs 3NF পার্থক্য কী?**

> 3NF তে non-prime attribute অন্য non-prime attribute এর উপর depend করতে পারবে না। BCNF তে প্রতিটা functional dependency X → Y তে X কে superkey হতে হবে। BCNF slightly more strict।

**Q6: একটা e-commerce system design করো যেখানে product এর multiple variants আছে (size, color)।**

```sql
CREATE TABLE products (
    id    BIGSERIAL PRIMARY KEY,
    name  VARCHAR(200)
);

CREATE TABLE product_variants (
    id         BIGSERIAL PRIMARY KEY,
    product_id BIGINT REFERENCES products(id),
    sku        VARCHAR(50) UNIQUE,
    price      DECIMAL(10,2),
    stock_qty  INT DEFAULT 0
);

CREATE TABLE variant_attributes (
    variant_id BIGINT REFERENCES product_variants(id),
    attr_name  VARCHAR(50),   -- 'color', 'size'
    attr_value VARCHAR(100),  -- 'Red', 'XL'
    PRIMARY KEY (variant_id, attr_name)
);
```

---

### Senior / FAANG Level

**Q7: Uber এর ride system এর schema design করো।**

```sql
CREATE TABLE users (
    id         BIGSERIAL PRIMARY KEY,
    name       VARCHAR(100),
    email      VARCHAR(254) UNIQUE,
    user_type  VARCHAR(10) CHECK (user_type IN ('rider', 'driver'))
);

CREATE TABLE vehicles (
    id            BIGSERIAL PRIMARY KEY,
    driver_id     BIGINT REFERENCES users(id),
    license_plate VARCHAR(20) UNIQUE,
    vehicle_type  VARCHAR(20)
);

CREATE TABLE rides (
    id           BIGSERIAL PRIMARY KEY,
    rider_id     BIGINT REFERENCES users(id),
    driver_id    BIGINT REFERENCES users(id),
    vehicle_id   BIGINT REFERENCES vehicles(id),
    status       VARCHAR(20) DEFAULT 'requested',
    pickup_lat   DECIMAL(10,8),
    pickup_lng   DECIMAL(11,8),
    dropoff_lat  DECIMAL(10,8),
    dropoff_lng  DECIMAL(11,8),
    fare         DECIMAL(10,2),
    started_at   TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at   TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE ride_locations (
    id         BIGSERIAL PRIMARY KEY,
    ride_id    BIGINT REFERENCES rides(id),
    latitude   DECIMAL(10,8),
    longitude  DECIMAL(11,8),
    recorded_at TIMESTAMPTZ DEFAULT NOW()
);
```

**Q8: High-write system এ normalization vs denormalization strategy কী হবে?**

> High-write system এ (payment, banking):
> - 3NF বজায় রাখো — data consistency critical
> - Denormalization করলে write কমবে কিন্তু inconsistency risk বাড়বে
> - Summary table গুলো async update করো (Celery/queue)
> - CQRS pattern use করো: write model normalized, read model denormalized

**Q9: Schema migration এ production এ কী সমস্যা হতে পারে?**

> - ADD COLUMN: Safe, কিন্তু DEFAULT সহ হলে পুরো table lock হতে পারে (PostgreSQL 11+ এ নেই)
> - ADD NOT NULL COLUMN: Table rewrite হয় — avoid করো, আগে nullable add করো তারপর backfill করো
> - Large table এ index creation: CONCURRENTLY ব্যবহার করো
> - Django migration তে atomic=False দিতে হতে পারে long-running migration এ

---

## 5.9 Hands-on Exercises

### Exercise 1
নিচের table টা normalize করো (1NF → 3NF):

```
employee_projects:
| emp_id | emp_name | dept | dept_head | project_id | project_name | hours |
```

### Exercise 2
একটা Hospital Management System এর ER diagram বানাও এবং SQL schema লেখো।
Entities: Doctor, Patient, Appointment, Ward, Prescription

### Exercise 3
Django ORM দিয়ে একটা Blog system বানাও:
- Post, Author, Tag (many-to-many), Comment
- Post এর tag count denormalize করে রাখো
- Author change হলে post এ author_name snapshot রাখো

### Mini Project
একটা simple e-commerce এর পূর্ণ schema design করো:
1. ER diagram (text format)
2. SQL DDL
3. Django models
4. 5টা common query লেখো ORM দিয়ে
5. EXPLAIN দিয়ে query plan দেখাও

---

## 5.10 Module Summary & Cheat Sheet

```
NORMALIZATION CHEAT SHEET
==========================

1NF  → Atomic values + Primary Key
2NF  → 1NF + No partial dependency (composite PK এ)
3NF  → 2NF + No transitive dependency
BCNF → Every determinant is a superkey
4NF  → BCNF + No multi-valued dependency
5NF  → 4NF + No join dependency

WHEN TO DENORMALIZE
====================
✅ Read-heavy (analytics, feeds, reports)
✅ JOIN count > 4-5 for common queries
✅ Data warehouse / OLAP
❌ Banking / payment systems
❌ Write-heavy systems

JUNCTION TABLE RULES
=====================
- M:N relationship এর জন্য mandatory
- Composite PK (fk1, fk2) ব্যবহার করো
- Extra attributes থাকলে through model করো
- উভয় FK তে index দাও

COMMON MISTAKES
================
1. Comma-separated values store করা
2. EAV anti-pattern ব্যবহার করা
3. FK তে index না দেওয়া
4. Over-normalization (10+ JOINs)
5. created_at/updated_at না রাখা
6. Soft delete strategy না ভাবা
7. VARCHAR সাইজ না ভাবা

PRODUCTION CHECKLIST
=====================
□ সব FK তে index আছে?
□ created_at, updated_at আছে?
□ PK BIGSERIAL বা UUID?
□ CHECK constraints দেওয়া?
□ NOT NULL যথাস্থানে?
□ Junction table এ composite PK?
□ Soft delete strategy ঠিক?
□ Migration safe? (zero downtime)
```

---

> **Senior Engineer Insight:**
> Schema design হলো সবচেয়ে important decision যা পরে পরিবর্তন করা সবচেয়ে কঠিন। Code refactor করা সহজ, কিন্তু production database এ schema change করা — বিশেষত millions of rows এ — অনেক risky এবং সময়সাপেক্ষ। তাই শুরুতেই ভালো design করো।

