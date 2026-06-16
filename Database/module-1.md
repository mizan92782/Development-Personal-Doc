# 🗄️ Database Engineering: বিগিনার থেকে অ্যাডভান্সড লেভেল
## সম্পূর্ণ বাংলা টিউটোরিয়াল — Backend Engineer দের জন্য

> **লেখক:** Senior Staff Software Engineer (Google, Amazon, Netflix অভিজ্ঞতা সম্পন্ন)
> **লক্ষ্য:** তোমাকে industry-level database engineer বানানো
> **ভাষা:** বাংলা (Bengali)
> **Stack:** Django ORM + PostgreSQL + SQL

---

# 📋 সম্পূর্ণ রোডম্যাপ

```
MODULE 1:  Database Fundamentals (Keys, ACID, Transactions)
MODULE 2:  SQL Fundamentals (Aliases, Aggregates, GROUP BY, HAVING, CASE)
MODULE 3:  SQL Joins (INNER, LEFT, RIGHT, FULL, CROSS, SELF, Multi-table)
MODULE 4:  Advanced SQL (Subqueries, CTE, Window Functions, UNION)
MODULE 5:  Database Design (Normalization, ER Diagram, Junction Tables)
MODULE 6:  Indexing (B-Tree, Hash, Composite, Covering, Clustered)
MODULE 7:  Query Optimization (EXPLAIN, N+1, Pagination, Bulk Operations)
MODULE 8:  Transactions & Concurrency (Isolation, Locks, Deadlocks)
MODULE 9:  ORM Deep Dive (Django ORM, Lazy/Eager Loading, F/Q Objects)
MODULE 10: PostgreSQL Deep Dive (MVCC, WAL, Partitioning, JSONB)
MODULE 11: Scaling Databases (Sharding, Replication, Caching)
MODULE 12: NoSQL Fundamentals (CAP, MongoDB, Redis)
MODULE 13: Database Security (SQL Injection, Encryption)
MODULE 14: Backup and Recovery (PITR, Disaster Recovery)
MODULE 15: Real-World Case Studies
MODULE 16: System Design Decisions
MODULE 17: Interview Preparation (Beginner to FAANG)
MODULE 18: Practice Exercises
```

---

# MODULE 1: Database Fundamentals

## 📌 1.1 Candidate Key

### ১. কেন শিখবো?

Candidate key হলো database design-এর ভিত্তি। তুমি যদি এটা না জানো, তাহলে তোমার schema design-এ এমন ভুল হবে যা production-এ গিয়ে disaster ঘটাবে। Amazon, Netflix সব জায়গায় senior engineer হতে হলে এটা crystal clear থাকতে হবে।

### ২. এই topic কী সমস্যা সমাধান করে?

একটি table-এ কোন column (বা column-এর combination) দিয়ে প্রতিটি row-কে **uniquely identify** করা যাবে — এটা নির্ধারণ করে।

### ৩. না জানলে কী হয়?

- Duplicate data insert হবে
- Data integrity নষ্ট হবে
- Query করলে ভুল result আসবে
- Production-এ customer complaint আসবে

### ৪. সহজ ভাষায় বোঝা

ধরো একটি student table:

```
| student_id | email              | national_id   | name  |
|------------|-------------------|---------------|-------|
| 1          | rafi@gmail.com    | BD-1234567    | Rafi  |
| 2          | karim@gmail.com   | BD-7654321    | Karim |
```

এখানে:
- `student_id` → প্রতিটি student-কে uniquely identify করে ✅
- `email` → প্রতিটি student-এর email unique ✅
- `national_id` → প্রতিটির national ID unique ✅

সুতরাং **Candidate Keys** হলো: `student_id`, `email`, `national_id`

এদের মধ্যে থেকে যেটাকে primary key হিসেবে choose করা হয় সেটাই **Primary Key**।

### ৫. নিয়ম (Rules)

```
Candidate Key হতে হলে দুটো property থাকতে হবে:
1. UNIQUENESS — প্রতিটি row-কে uniquely identify করতে পারবে
2. IRREDUCIBILITY (Minimality) — column কমালে আর unique থাকবে না
```

### ৬. Minimality উদাহরণ

```
| order_id | product_id | customer_id | quantity |
|----------|------------|-------------|----------|
| 1        | 101        | 201         | 5        |
| 2        | 101        | 202         | 3        |
```

`(order_id, product_id)` → unique হতে পারে, কিন্তু minimal না কারণ শুধু `order_id` দিয়েই unique।
`order_id` → minimal ✅ → Candidate Key

### ৭. SQL Example

```sql
-- Candidate key ব্যবহার করে table তৈরি
CREATE TABLE students (
    student_id   SERIAL,
    email        VARCHAR(255) NOT NULL,
    national_id  CHAR(12) NOT NULL,
    name         VARCHAR(100) NOT NULL,

    -- সব candidate key-তে UNIQUE constraint দাও
    CONSTRAINT uq_email       UNIQUE (email),
    CONSTRAINT uq_national_id UNIQUE (national_id),

    -- একটিকে Primary Key বানাও
    PRIMARY KEY (student_id)
);
```

### ৮. Django ORM Example

```python
from django.db import models

class Student(models.Model):
    email       = models.EmailField(unique=True)   # Candidate Key
    national_id = models.CharField(max_length=12, unique=True)  # Candidate Key
    name        = models.CharField(max_length=100)

    class Meta:
        db_table = 'students'

# Django এর পিছনে যা SQL generate করে:
# CREATE TABLE students (
#     id          SERIAL PRIMARY KEY,  ← Django auto-adds this
#     email       VARCHAR(254) UNIQUE NOT NULL,
#     national_id VARCHAR(12)  UNIQUE NOT NULL,
#     name        VARCHAR(100) NOT NULL
# );
```

> 💡 **Django Insight:** Django automatically `id` field add করে primary key হিসেবে, যদি তুমি explicitly না দাও।

### ৯. Common Mistakes

❌ **ভুল:** `name` field-কে candidate key মনে করা
```sql
-- ভুল! দুইজনের নাম এক হতে পারে
ALTER TABLE students ADD UNIQUE (name);
```

✅ **সঠিক:** Only uniquely identifying columns-কে candidate key treat করো

### ১০. Interview Questions

**Beginner:**
> Q: Candidate key কী?
> A: যে column বা column-combination দিয়ে table-এর প্রতিটি row uniquely identify করা যায় এবং কোনো column বাদ দিলে আর unique থাকে না।

**Intermediate:**
> Q: Candidate key এবং Primary key-এর পার্থক্য কী?
> A: একটি table-এ একাধিক candidate key থাকতে পারে। তাদের মধ্যে থেকে একটিকে primary key হিসেবে choose করা হয়। বাকিগুলো alternate key।

**Senior/FAANG:**
> Q: তোমার e-commerce system-এ `orders` table design করতে হবে। Candidate key নির্বাচনে কী কী বিষয় বিবেচনা করবে?
> A: 
> - `order_id` (surrogate key) — simple, stable, fast indexing
> - Business logic দিয়ে কোনো natural candidate key আছে কিনা দেখবো (যেমন `order_number` যদি globally unique হয়)
> - Email + timestamp combination candidate key হতে পারে, কিন্তু composite key indexing complex করে
> - Production-এ surrogate key prefer করি কারণ business rule পরিবর্তন হলে natural key-তে migration nightmare হয়

---

## 📌 1.2 Composite Key

### ১. কেন শিখবো?

Many-to-many relationship handle করতে composite key অপরিহার্য। Amazon-এর order-product, Facebook-এর user-friend সব জায়গায় এটা আছে।

### ২. সহজ ভাষায় বোঝা

যখন **দুই বা তার বেশি column মিলে** একটি unique identifier তৈরি করে, তখন সেটা Composite Key।

### ৩. Real-world Example: Amazon Order Items

```
একটি order-এ একই product একবারই থাকবে।
তাহলে (order_id, product_id) মিলে unique।
কিন্তু আলাদা আলাদাভাবে unique না।
```

```sql
CREATE TABLE order_items (
    order_id    INTEGER NOT NULL,
    product_id  INTEGER NOT NULL,
    quantity    INTEGER NOT NULL DEFAULT 1,
    unit_price  DECIMAL(10, 2) NOT NULL,

    -- Composite Primary Key
    PRIMARY KEY (order_id, product_id),

    FOREIGN KEY (order_id)   REFERENCES orders(id),
    FOREIGN KEY (product_id) REFERENCES products(id)
);
```

```
order_id | product_id | quantity | unit_price
---------|------------|----------|----------
1001     | 501        | 2        | 599.00     ✅ unique (1001, 501)
1001     | 502        | 1        | 1299.00    ✅ unique (1001, 502)
1002     | 501        | 3        | 599.00     ✅ unique (1002, 501)
```

### ৪. Django ORM Example

```python
class OrderItem(models.Model):
    order   = models.ForeignKey('Order',   on_delete=models.CASCADE)
    product = models.ForeignKey('Product', on_delete=models.CASCADE)
    quantity   = models.PositiveIntegerField(default=1)
    unit_price = models.DecimalField(max_digits=10, decimal_places=2)

    class Meta:
        # Composite Primary Key (Django 5.2+ supports this natively)
        # পুরোনো Django-তে unique_together ব্যবহার করতে হতো:
        unique_together = [('order', 'product')]
        # অথবা:
        constraints = [
            models.UniqueConstraint(
                fields=['order', 'product'],
                name='uq_order_product'
            )
        ]

# Generated SQL:
# ALTER TABLE order_items
#     ADD CONSTRAINT uq_order_product UNIQUE (order_id, product_id);
```

### ৫. Composite Key-এর Performance Implication

```
✅ সুবিধা:
- Junction table-এ extra surrogate key avoid করা যায়
- Index আপনা-আপনি তৈরি হয় (primary key = clustered index in some DBs)

❌ অসুবিধা:
- Foreign key reference করা complex হয়
- যদি child table reference করতে হয়, উভয় column দিতে হবে
- Query লেখা verbose হয়
```

### ৬. Common Mistake

```sql
-- ❌ ভুল: surrogate id add করে composite key ignore করা
CREATE TABLE order_items (
    id         SERIAL PRIMARY KEY,  -- unnecessary!
    order_id   INTEGER,
    product_id INTEGER
    -- duplicate insert হওয়ার সম্ভাবনা!
);

-- ✅ সঠিক:
CREATE TABLE order_items (
    order_id   INTEGER NOT NULL,
    product_id INTEGER NOT NULL,
    quantity   INTEGER NOT NULL,
    PRIMARY KEY (order_id, product_id)  -- constraint enforce করছে
);
```

---

## 📌 1.3 Surrogate Key vs Natural Key

### ১. সংজ্ঞা

| | Surrogate Key | Natural Key |
|---|---|---|
| **কী?** | Artificially generated ID (1, 2, 3...) | Real-world data থেকে আসা (email, NID) |
| **উদাহরণ** | `user_id = 1001` | `email = 'rafi@gmail.com'` |
| **Control** | আমরা তৈরি করি | Business domain থেকে আসে |

### ২. Real-world War Story 🔥

> **Netflix-এর একটি বাস্তব সমস্যা:**
> 
> Netflix প্রথমে `movie_title` কে natural key হিসেবে ব্যবহার করেছিল। পরে যখন একই নামের দুটো movie license করতে হলো (দুটো আলাদা region-এর), তখন পুরো database migration nightmare হয়ে গেল। লক্ষাধিক foreign key update করতে হলো।
>
> **শিক্ষা:** Business rule পরিবর্তন হয়। Surrogate key stable থাকে।

### ৩. তুলনামূলক বিশ্লেষণ

```
SURROGATE KEY (id SERIAL):
✅ Simple, fast integer comparison
✅ Business logic পরিবর্তন হলেও key পরিবর্তন হয় না
✅ JOIN করা সহজ
✅ URL-এ expose করা যায় (/users/1234)
❌ Real-world meaning নেই
❌ Extra column দরকার

NATURAL KEY (email, NID, ISBN):
✅ Human-readable
✅ Business দিক থেকে meaningful
✅ Extra column দরকার নেই
❌ পরিবর্তন হতে পারে (email change, NID correct)
❌ String comparison slow
❌ Foreign key-তে large data copy হয়
❌ Composite হলে JOIN জটিল
```

### ৪. Production Best Practice

```sql
-- ✅ Production-grade approach: Surrogate + Natural key উভয়ই রাখো
CREATE TABLE users (
    -- Surrogate key (internal use, JOIN করতে)
    id          BIGSERIAL PRIMARY KEY,

    -- Natural key (business logic, human-readable)
    email       VARCHAR(255) UNIQUE NOT NULL,

    -- Opaque public ID (API expose করতে, UUID)
    public_id   UUID DEFAULT gen_random_uuid() UNIQUE NOT NULL,

    name        VARCHAR(100) NOT NULL,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);
```

> 💡 **Senior Engineer Insight:**
> - Internal JOIN-এ: `id` (integer, fast)
> - API response-এ: `public_id` (UUID, security)
> - Human communication-এ: `email` (meaningful)
> - তিনটার আলাদা purpose আছে!

### ৫. Django ORM Example

```python
import uuid
from django.db import models

class User(models.Model):
    # id → Django auto-adds BigAutoField (surrogate key)
    public_id = models.UUIDField(default=uuid.uuid4, unique=True, editable=False)
    email     = models.EmailField(unique=True)  # natural key
    name      = models.CharField(max_length=100)

    class Meta:
        db_table = 'users'

# API view-এ public_id use করো:
# GET /api/users/550e8400-e29b-41d4-a716-446655440000/
# এতে user enumeration attack prevent হয়
```

---

## 📌 1.4 Constraints (Database Constraints)

### ১. কেন শিখবো?

Constraint হলো database-এর "রক্ষী"। এটা data quality নিশ্চিত করে। Application layer-এ bug থাকলেও database-এ garbage data ঢুকতে পারবে না।

### ২. Constraints-এর প্রকারভেদ

```
1. NOT NULL      — column empty থাকতে পারবে না
2. UNIQUE        — duplicate value থাকবে না
3. PRIMARY KEY   — NOT NULL + UNIQUE
4. FOREIGN KEY   — অন্য table-এর valid reference
5. CHECK         — custom validation rule
6. DEFAULT       — value না দিলে default value
7. EXCLUSION     — PostgreSQL-specific (overlap prevent)
```

### ৩. SQL Example — E-commerce Products Table

```sql
CREATE TABLE products (
    id            BIGSERIAL PRIMARY KEY,
    sku           VARCHAR(50) NOT NULL,
    name          VARCHAR(255) NOT NULL,
    price         DECIMAL(10, 2) NOT NULL,
    stock_qty     INTEGER NOT NULL DEFAULT 0,
    category_id   INTEGER,
    is_active     BOOLEAN NOT NULL DEFAULT TRUE,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- Constraints
    CONSTRAINT uq_sku           UNIQUE (sku),
    CONSTRAINT chk_price_positive CHECK (price > 0),
    CONSTRAINT chk_stock_non_neg  CHECK (stock_qty >= 0),
    CONSTRAINT fk_category
        FOREIGN KEY (category_id)
        REFERENCES categories(id)
        ON DELETE SET NULL
        ON UPDATE CASCADE
);
```

### ৪. Foreign Key Actions বোঝা

```sql
-- ON DELETE এর option:
-- CASCADE    → parent delete হলে child-ও delete হবে
-- SET NULL   → parent delete হলে child column NULL হবে
-- RESTRICT   → parent delete করতে দেবে না (child থাকলে)
-- NO ACTION  → RESTRICT-এর মতো (default)
-- SET DEFAULT → default value set হবে

-- উদাহরণ: User account delete হলে orders কী হবে?
-- ❌ CASCADE — order history হারিয়ে যাবে (banking nightmare!)
-- ✅ SET NULL বা soft delete করো

ALTER TABLE orders
    ADD CONSTRAINT fk_user
    FOREIGN KEY (user_id)
    REFERENCES users(id)
    ON DELETE SET NULL;  -- user delete হলে order থাকবে, user_id NULL হবে
```

### ৫. Django ORM Example

```python
class Product(models.Model):
    sku      = models.CharField(max_length=50, unique=True)
    name     = models.CharField(max_length=255)
    price    = models.DecimalField(max_digits=10, decimal_places=2)
    stock    = models.IntegerField(default=0)
    category = models.ForeignKey(
        'Category',
        on_delete=models.SET_NULL,  # FK action
        null=True,
        blank=True
    )
    is_active = models.BooleanField(default=True)

    class Meta:
        constraints = [
            models.CheckConstraint(
                check=models.Q(price__gt=0),
                name='chk_price_positive'
            ),
            models.CheckConstraint(
                check=models.Q(stock__gte=0),
                name='chk_stock_non_negative'
            ),
        ]

# Generated SQL:
# ALTER TABLE products ADD CONSTRAINT chk_price_positive CHECK (price > 0);
# ALTER TABLE products ADD CONSTRAINT chk_stock_non_negative CHECK (stock >= 0);
```

### ৬. Production Best Practice

```sql
-- সবসময় constraint-এ নাম দাও (unnamed constraint debug কঠিন)
-- ❌ ভুল:
ALTER TABLE products ADD CHECK (price > 0);

-- ✅ সঠিক:
ALTER TABLE products ADD CONSTRAINT chk_product_price_positive CHECK (price > 0);

-- Error message হবে: "violates check constraint "chk_product_price_positive""
-- Debug সহজ!
```

### ৭. Real-world Problem: Inventory System

```sql
-- Uber Eats-এর মতো food delivery system
-- একটি restaurant-এ item-এর stock negative হতে পারবে না

CREATE TABLE menu_items (
    id          BIGSERIAL PRIMARY KEY,
    restaurant_id INTEGER NOT NULL,
    name        VARCHAR(255) NOT NULL,
    price       DECIMAL(8, 2) NOT NULL,
    available_count INTEGER NOT NULL DEFAULT 0,

    CONSTRAINT chk_price         CHECK (price > 0),
    CONSTRAINT chk_available     CHECK (available_count >= 0),
    CONSTRAINT uq_restaurant_item UNIQUE (restaurant_id, name),

    FOREIGN KEY (restaurant_id)
        REFERENCES restaurants(id)
        ON DELETE CASCADE  -- restaurant delete হলে সব item delete
);
```

---

## 📌 1.5 ACID Properties

### ১. কেন এটা সবচেয়ে গুরুত্বপূর্ণ?

Banking, payment, e-commerce — সব critical system ACID-এর উপর নির্ভরশীল। এটা না জানলে তুমি কখনো production-ready system বানাতে পারবে না।

### ২. ACID মানে কী?

```
A — Atomicity    (সব হবে, নয়তো কিছুই হবে না)
C — Consistency  (database সবসময় valid state-এ থাকবে)
I — Isolation    (concurrent transaction একে অপরকে affect করবে না)
D — Durability   (commit হলে power fail হলেও data থাকবে)
```

### ৩. বাস্তব উদাহরণ: Bank Transfer

#### Scenario: রাফি → করিম-কে ১০০০ টাকা পাঠাচ্ছে

```
রাফির balance:  5000 টাকা
করিমের balance: 2000 টাকা
Transfer:       1000 টাকা
```

**ACID ছাড়া কী হতে পারে?**

```
Step 1: রাফির account থেকে 1000 কাটো  → রাফি: 4000
Step 2: *** SYSTEM CRASH ***
Step 3: করিমের account-এ 1000 যোগ করো → NEVER EXECUTED

ফলাফল: রাফির 1000 টাকা গায়েব! Bank account balance mismatch!
```

**ACID সহ:**

```sql
BEGIN TRANSACTION;  -- শুরু

UPDATE accounts SET balance = balance - 1000 WHERE user_id = 'rafi';
UPDATE accounts SET balance = balance + 1000 WHERE user_id = 'karim';

-- দুটোই সফল হলে:
COMMIT;

-- যেকোনো একটি fail হলে:
ROLLBACK;  -- সব পূর্বাবস্থায় ফিরে যাবে
```

### ৪. Atomicity বিস্তারিত

```
"All or Nothing" — হয় সব operation হবে, নয়তো একটাও হবে না

উদাহরণ: E-commerce order place
1. Order create করো
2. Payment charge করো
3. Inventory কমাও
4. Notification পাঠাও

যদি step 3 fail করে:
❌ ACID ছাড়া: Order created, Payment charged, Inventory not reduced
✅ ACID সহ:   সব rollback → Customer charge হয়নি, order নেই
```

### ৫. Consistency বিস্তারিত

```
Database সবসময় valid state-এ থাকবে।
Constraint, Rule সব মেনে চলবে।

উদাহরণ:
- balance কখনো negative হবে না (CHECK constraint)
- foreign key সবসময় valid reference দেখাবে
- একটি transaction-এর পরে database consistent থাকবে
```

### ৬. Isolation বিস্তারিত

```
দুটো transaction simultaneously চললে
একটি আরেকটিকে "দেখতে" পাবে না (intermediate state)

উদাহরণ: Flight booking
User A: Seat 14A check করছে
User B: Seat 14A book করছে (almost same time)

ACID Isolation নিশ্চিত করে:
- User A কে দেখাবে seat available
- User B book করলে commit হবে
- User A যদি এরপর book করতে চায়, সে দেখবে seat taken

দুইজনকে একই seat দেওয়া হবে না!
```

### ৭. Durability বিস্তারিত

```
COMMIT হলে data permanent।
Power cut, crash — কিছুই হবে না।

এটা নিশ্চিত করে:
- WAL (Write-Ahead Log)
- fsync
- Disk write

PostgreSQL প্রতিটি commit-এ disk-এ লেখে।
এজন্য PostgreSQL production-এ trusted।
```

### ৮. SQL Example — Complete Bank Transfer

```sql
-- PostgreSQL-এ production-ready bank transfer
CREATE OR REPLACE FUNCTION transfer_money(
    p_from_user_id  INTEGER,
    p_to_user_id    INTEGER,
    p_amount        DECIMAL(10,2)
) RETURNS VOID AS $$
DECLARE
    v_from_balance DECIMAL(10,2);
BEGIN
    -- Atomicity: এই block-এর সব কিছু একসাথে হবে বা rollback হবে

    -- Step 1: Sender-এর balance check (FOR UPDATE = row lock)
    SELECT balance INTO v_from_balance
    FROM accounts
    WHERE user_id = p_from_user_id
    FOR UPDATE;  -- অন্য transaction এই row change করতে পারবে না

    -- Step 2: Consistency check
    IF v_from_balance < p_amount THEN
        RAISE EXCEPTION 'Insufficient balance: % < %', v_from_balance, p_amount;
    END IF;

    -- Step 3: Deduct from sender
    UPDATE accounts
    SET balance = balance - p_amount,
        updated_at = NOW()
    WHERE user_id = p_from_user_id;

    -- Step 4: Add to receiver
    UPDATE accounts
    SET balance = balance + p_amount,
        updated_at = NOW()
    WHERE user_id = p_to_user_id;

    -- Step 5: Audit log
    INSERT INTO transaction_logs (from_user, to_user, amount, created_at)
    VALUES (p_from_user_id, p_to_user_id, p_amount, NOW());

    -- সব ঠিক থাকলে commit (caller করবে)
END;
$$ LANGUAGE plpgsql;

-- Call:
BEGIN;
SELECT transfer_money(1, 2, 1000.00);
COMMIT;
```

### ৯. Django ORM — Transaction Management

```python
from django.db import transaction
from django.core.exceptions import ValidationError
from decimal import Decimal

def transfer_money(from_user_id: int, to_user_id: int, amount: Decimal):
    """
    Bank transfer with full ACID compliance using Django ORM
    """
    with transaction.atomic():  # ← এটাই ACID transaction শুরু করে
        # SELECT FOR UPDATE — row lock দেয় (Isolation)
        from_account = Account.objects.select_for_update().get(
            user_id=from_user_id
        )
        to_account = Account.objects.select_for_update().get(
            user_id=to_user_id
        )

        # Consistency check
        if from_account.balance < amount:
            raise ValidationError("Insufficient balance")

        # Atomic operations
        from_account.balance -= amount
        from_account.save()

        to_account.balance += amount
        to_account.save()

        # Audit log
        TransactionLog.objects.create(
            from_account=from_account,
            to_account=to_account,
            amount=amount
        )
        # Exception হলে সব auto rollback হবে!

# Generated SQL:
# BEGIN;
# SELECT ... FROM accounts WHERE user_id=1 FOR UPDATE;
# SELECT ... FROM accounts WHERE user_id=2 FOR UPDATE;
# UPDATE accounts SET balance=balance-1000 WHERE user_id=1;
# UPDATE accounts SET balance=balance+1000 WHERE user_id=2;
# INSERT INTO transaction_logs (...) VALUES (...);
# COMMIT;
```

### ১০. ACID vs NoSQL

```
MongoDB (default): ACID নেই (single document-এ আছে, cross-document নেই)
PostgreSQL:        Full ACID ✅
MySQL InnoDB:      Full ACID ✅
Redis:             Limited (single command atomic, transaction weak)

কখন ACID দরকার?
→ Banking, payments, inventory, booking systems
→ যেখানে data loss বা inconsistency fatal

কখন ACID ছাড়া চলে?
→ Social media likes, view counts, analytics
→ যেখানে eventual consistency acceptable
```

### ১১. Interview Questions

**Beginner:**
> Q: ACID মানে কী?
> A: Atomicity (all or nothing), Consistency (valid state), Isolation (concurrent isolation), Durability (permanent after commit)

**Intermediate:**
> Q: Bank transfer-এ ACID কীভাবে help করে?
> A: Atomicity নিশ্চিত করে debit এবং credit দুটোই হবে বা কোনোটাই হবে না। Isolation নিশ্চিত করে দুটো concurrent transfer একে অপরকে corrupt করবে না।

**Senior/FAANG:**
> Q: Amazon-এর payment system-এ ACID কীভাবে implement করবে যেখানে multiple microservices আছে?
> A: 
> - Single database-এ ACID native
> - Distributed system-এ SAGA pattern use করি
> - প্রতিটি service local transaction করে
> - Failure হলে compensating transaction চালাই
> - Two-Phase Commit (2PC) avoid করি (performance bottleneck)
> - Event sourcing দিয়ে audit trail রাখি

---

## 📌 1.6 Transactions

### ১. Transaction কী?

Transaction হলো এক বা একাধিক SQL operation-এর একটি logical unit যা ACID property মেনে চলে।

```sql
-- Simple transaction structure:
BEGIN;          -- transaction শুরু
    SQL_1;
    SQL_2;
    SQL_3;
COMMIT;         -- সফল হলে save
-- অথবা
ROLLBACK;       -- fail হলে undo
```

### ২. Savepoint — Advanced Transaction

```sql
-- Complex scenario: Amazon order with multiple steps
BEGIN;

-- Step 1: Order তৈরি করো
INSERT INTO orders (user_id, total) VALUES (101, 5000.00);
SAVEPOINT order_created;  -- checkpoint

-- Step 2: Payment try করো
INSERT INTO payments (order_id, amount, status)
VALUES (LASTVAL(), 5000.00, 'processing');
SAVEPOINT payment_initiated;

-- Step 3: Inventory update
UPDATE products SET stock = stock - 1 WHERE id = 501;

-- যদি inventory update fail করে:
ROLLBACK TO SAVEPOINT payment_initiated;
-- শুধু inventory step undo হবে, order ও payment থাকবে

COMMIT;
```

### ৩. Django ORM — Savepoint

```python
from django.db import transaction

def place_order(user_id, cart_items):
    with transaction.atomic():
        # Order create
        order = Order.objects.create(user_id=user_id)
        sid = transaction.savepoint()  # savepoint

        try:
            # Payment process
            payment = Payment.objects.create(
                order=order,
                amount=order.total
            )
        except PaymentError:
            transaction.savepoint_rollback(sid)
            # Payment fail, but order still exists
            order.status = 'payment_failed'
            order.save()
            raise

        # Inventory update
        for item in cart_items:
            Product.objects.filter(id=item.product_id).update(
                stock=F('stock') - item.quantity
            )

        transaction.savepoint_commit(sid)
```

### ৪. Transaction Isolation Levels (Preview — বিস্তারিত Module 8-এ)

```
READ UNCOMMITTED  → Dirty reads possible (সবচেয়ে কম strict)
READ COMMITTED    → No dirty reads (PostgreSQL default)
REPEATABLE READ   → No dirty reads, no non-repeatable reads
SERIALIZABLE      → সবচেয়ে strict, full isolation
```

### ৫. Common Mistakes

```python
# ❌ ভুল: Transaction ছাড়া multiple operations
def transfer():
    from_account.balance -= 1000
    from_account.save()
    # *** CRASH HERE ***
    to_account.balance += 1000
    to_account.save()

# ✅ সঠিক: atomic() দিয়ে wrap করো
def transfer():
    with transaction.atomic():
        from_account.balance -= 1000
        from_account.save()
        to_account.balance += 1000
        to_account.save()
```

```python
# ❌ ভুল: Exception কে transaction-এর বাইরে handle করা
try:
    with transaction.atomic():
        do_something()
except Exception:
    pass  # transaction already rolled back, কিন্তু side effects বাকি

# ✅ সঠিক:
with transaction.atomic():
    try:
        do_something()
    except SpecificError:
        # handle করো, কিন্তু atomic() block-এর মধ্যে থাকো
        handle_error()
        raise  # re-raise করলে rollback হবে
```

---

## 📊 MODULE 1 — Summary Sheet

```
╔══════════════════════════════════════════════════════════════╗
║              MODULE 1: DATABASE FUNDAMENTALS                 ║
╠══════════════════════════════════════════════════════════════╣
║ CANDIDATE KEY                                                ║
║  • Uniquely identifies each row                              ║
║  • Must be minimal (irreducible)                             ║
║  • A table can have multiple candidate keys                  ║
╠══════════════════════════════════════════════════════════════╣
║ COMPOSITE KEY                                                ║
║  • Two or more columns together form a unique identifier     ║
║  • Used in junction/bridge tables (many-to-many)             ║
║  • PRIMARY KEY (col1, col2)                                  ║
╠══════════════════════════════════════════════════════════════╣
║ SURROGATE vs NATURAL KEY                                     ║
║  • Surrogate: auto-generated (id, uuid) — stable, simple     ║
║  • Natural: business data (email, NID) — meaningful          ║
║  • Production: use surrogate + natural both                  ║
╠══════════════════════════════════════════════════════════════╣
║ CONSTRAINTS                                                  ║
║  • NOT NULL, UNIQUE, PRIMARY KEY, FOREIGN KEY                ║
║  • CHECK (custom validation), DEFAULT                        ║
║  • Always name your constraints!                             ║
╠══════════════════════════════════════════════════════════════╣
║ ACID PROPERTIES                                              ║
║  A → Atomicity: All or Nothing                               ║
║  C → Consistency: Valid state always                         ║
║  I → Isolation: Concurrent transactions don't interfere      ║
║  D → Durability: Committed data is permanent                 ║
╠══════════════════════════════════════════════════════════════╣
║ TRANSACTIONS                                                 ║
║  • BEGIN / COMMIT / ROLLBACK                                 ║
║  • Django: with transaction.atomic():                        ║
║  • Savepoints for partial rollback                           ║
║  • select_for_update() for row locking                       ║
╚══════════════════════════════════════════════════════════════╝
```

## 🎯 MODULE 1 — Hands-on Exercises

```sql
-- Exercise 1: Banking System Schema
-- নিচের table গুলো সঠিক constraints দিয়ে তৈরি করো:
-- 1. banks (id, name, routing_number)
-- 2. accounts (id, bank_id, user_id, account_number, balance, type)
-- 3. transactions (id, from_account, to_account, amount, type, status)

-- Exercise 2: Constraint Testing
-- একটি account-এর balance negative করার চেষ্টা করো
-- CHECK constraint কীভাবে prevent করে দেখো

-- Exercise 3: ACID Transaction
-- দুটো account-এর মধ্যে transfer করো
-- মাঝপথে error inject করো এবং rollback confirm করো

-- Exercise 4: Django ORM
-- উপরের schema Django models-এ convert করো
-- transfer_money() function লেখো transaction.atomic() দিয়ে
```

---
