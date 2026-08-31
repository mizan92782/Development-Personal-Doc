# LEVEL 3 — Database & SQL (PostgreSQL Focus)

This is the layer where weak understanding hides for a while... and then breaks everything at scale. A backend developer who treats the database as a black box will eventually write code that's *technically correct* but *practically unusable* under real traffic. This guide covers everything you listed, plus the topics you'll need next.

---

## 1. SQL Fundamentals

Every SQL query follows a mental pipeline. Understanding the **order things actually execute in** (not the order you type them) is the single biggest unlock.

```
SELECT column_list
FROM table
JOIN other_table ON condition
WHERE row_condition
GROUP BY column
HAVING group_condition
ORDER BY column
LIMIT n OFFSET m;
```

### The real execution order (this is the key insight)

```
You WRITE it like this:          It EXECUTES like this:

SELECT ...                       1. FROM        (get base table)
FROM ...                         2. JOIN        (attach related tables)
JOIN ...                         3. WHERE        (filter individual rows)
WHERE ...                        4. GROUP BY     (bucket rows into groups)
GROUP BY ...                     5. HAVING       (filter groups)
HAVING ...                       6. SELECT       (pick final columns)
ORDER BY ...                     7. ORDER BY     (sort results)
LIMIT ...                        8. LIMIT/OFFSET (cut down to a page)
```

**Why this matters:** this is exactly why you can't filter on an aggregate using `WHERE` — at the point `WHERE` runs, groups don't exist yet. That's what `HAVING` is for.

### Each command, one line each

| Command | Purpose |
|---|---|
| `SELECT` | Read/retrieve rows |
| `INSERT` | Add new row(s) |
| `UPDATE` | Modify existing row(s) |
| `DELETE` | Remove row(s) |
| `JOIN` | Combine rows from related tables |
| `GROUP BY` | Bucket rows into groups (e.g. per user, per category) |
| `HAVING` | Filter groups (after aggregation) |
| `ORDER BY` | Sort the result set |
| `LIMIT` | Cap how many rows are returned |
| `OFFSET` | Skip N rows before returning results |

### WHERE vs HAVING (common confusion)

```
WHERE  → filters individual ROWS, before grouping
HAVING → filters GROUPS, after aggregation

Example: "Users who placed more than 5 orders"

SELECT user_id, COUNT(*) AS order_count
FROM orders
GROUP BY user_id
HAVING COUNT(*) > 5;      ← can't use WHERE here, COUNT(*) doesn't
                             exist yet at the WHERE stage
```

### Example query, visualized

```sql
SELECT category, COUNT(*) AS total
FROM products
WHERE in_stock = true
GROUP BY category
HAVING COUNT(*) > 10
ORDER BY total DESC
LIMIT 5;
```

```
All Products (1000 rows)
        │
        ▼ WHERE in_stock = true
   In-stock products (700 rows)
        │
        ▼ GROUP BY category
   [Electronics: 150] [Clothing: 8] [Books: 40] [Toys: 5] ...
        │
        ▼ HAVING COUNT(*) > 10
   [Electronics: 150] [Books: 40]        ← "Clothing" and "Toys" dropped
        │
        ▼ ORDER BY total DESC
   [Electronics: 150] [Books: 40]
        │
        ▼ LIMIT 5
   Final result (top 5, here just 2 groups qualify)
```

---

## 2. Primary Key

A **primary key** uniquely identifies each row in a table. No two rows can share one, and it can never be `NULL`.

```
users table
┌────┬───────────┬──────────────────┐
│ id │   name    │      email       │
├────┼───────────┼──────────────────┤
│ 1  │ Alice     │ alice@mail.com   │  ← id is the PRIMARY KEY
│ 2  │ Bob       │ bob@mail.com     │     unique, not null, one per row
│ 3  │ Charlie   │ charlie@mail.com │
└────┴───────────┴──────────────────┘
```

**Necessity:** without a primary key, there's no reliable way to say "update *this exact* row" — you'd have to match on other fields, which might not be unique or might change over time.

---

## 3. Foreign Key

A **foreign key** is a column in one table that **references the primary key of another table**, creating a link between them.

```
orders table                          users table
┌────┬─────────┬──────────┐          ┌────┬───────┐
│ id │ user_id │  total   │          │ id │ name  │
├────┼─────────┼──────────┤          ├────┼───────┤
│ 1  │    2    │  $50     │  ───►    │ 1  │ Alice │
│ 2  │    1    │  $20     │  ───►    │ 2  │ Bob   │
│ 3  │    2    │  $80     │  ───►    │ 3  │ Charlie│
└────┴─────────┴──────────┘          └────┴───────┘
        ▲
        └── user_id → users.id  (Foreign Key)
```

```sql
CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  user_id INT REFERENCES users(id),  -- foreign key
  total NUMERIC
);
```

**Necessity:** foreign keys enforce **referential integrity** — the database itself will *refuse* to insert an order with `user_id = 999` if no user with `id = 999` exists, preventing orphaned/broken data at the source instead of relying on application code to remember to check.

---

## 4. Relationships

### One-to-One

Each row in Table A relates to **exactly one** row in Table B, and vice versa.

```
users                          user_profiles
┌────┬───────┐                ┌────┬─────────┬──────────────┐
│ id │ name  │                │ id │ user_id │   bio        │
├────┼───────┤                ├────┼─────────┼──────────────┤
│ 1  │ Alice │  ◄────────────►│ 1  │    1    │ "Loves cats" │
└────┴───────┘                └────┴─────────┴──────────────┘
```
Example: `users` ↔ `user_profiles` (each user has exactly one profile).

### One-to-Many

One row in Table A relates to **many** rows in Table B, but each row in B relates to only one row in A.

```
users                          orders
┌────┬───────┐                ┌────┬─────────┬────────┐
│ id │ name  │                │ id │ user_id │ total  │
├────┼───────┤                ├────┼─────────┼────────┤
│ 1  │ Alice │  ◄──────┬─────►│ 1  │    1    │ $50    │
└────┴───────┘         │      ├────┼─────────┼────────┤
                        └─────►│ 2  │    1    │ $20    │
                               └────┴─────────┴────────┘
```
Example: one user → many orders. This is the **most common** relationship in real systems.

### Many-to-Many

Many rows in Table A relate to many rows in Table B. This **requires a third "junction" table** in between, since a foreign key column can only point to one row.

```
students                junction table              courses
┌────┬───────┐          (enrollments)              ┌────┬─────────┐
│ id │ name  │        ┌───────────┬──────────┐     │ id │  name   │
├────┼───────┤        │ student_id│ course_id│     ├────┼─────────┤
│ 1  │ Alice │◄──────►│     1     │    1     │◄───►│ 1  │ Math    │
│ 2  │ Bob   │◄──────►│     1     │    2     │◄───►│ 2  │ Physics │
└────┴───────┘        │     2     │    1     │     └────┴─────────┘
                       └───────────┴──────────┘
```
Example: students ↔ courses. Alice takes Math + Physics; Bob takes Math — you can't model this with a single foreign key on either side, so `enrollments` sits in the middle with two foreign keys.

```sql
CREATE TABLE enrollments (
  student_id INT REFERENCES students(id),
  course_id INT REFERENCES courses(id),
  PRIMARY KEY (student_id, course_id)
);
```

---

## 5. Indexes

An index is a **separate, sorted data structure** the database maintains alongside your table, so it can find rows **without scanning every single row**.

```sql
CREATE INDEX idx_user_email
ON users(email);
```

### Without an index — full table scan

```
SELECT * FROM users WHERE email = 'bob@mail.com';

┌────┬──────────────────┐
│ id │ email             │
├────┼──────────────────┤
│ 1  │ alice@mail.com    │  ← check
│ 2  │ zoe@mail.com      │  ← check
│ 3  │ bob@mail.com      │  ← check → MATCH (but had to check every row!)
│ 4  │ dan@mail.com      │  ← check
│ .. │ ... (1 million rows)│  ← check every single one
└────┴──────────────────┘
Time complexity: O(n) — gets slower as the table grows
```

### With an index — like a book's index page

```
Index on `email` (sorted, like a phone book)

  alice@mail.com  → row 1
  bob@mail.com    → row 3      ← jump straight here
  dan@mail.com    → row 4
  zoe@mail.com    → row 2

Time complexity: O(log n) — barely slows down even with millions of rows
```

```
                    B-Tree Index Structure (simplified)

                          [ m ]
                        /       \
                 [ b, d ]        [ p, z ]
                /    |            |    \
          [a,b]  [b,d]      [d,p]     [p,z]
                    ▲
             "bob@mail.com" found in
             3 hops instead of scanning
             1,000,000 rows
```

### The downside — indexes aren't free

```
More indexes
     ↓
Faster reads   (SELECT queries jump straight to the row)
     ↓
More storage   (each index is its own copy of sorted data)
     ↓
Slower writes  (every INSERT/UPDATE/DELETE must also update
                every index on that table)
```

```
INSERT INTO users (name, email) VALUES ('Eve', 'eve@mail.com');

Without indexes:              With 5 indexes on this table:
  1. Write row                  1. Write row
                                 2. Update index 1
                                 3. Update index 2
                                 4. Update index 3
                                 5. Update index 4
                                 6. Update index 5
  → Fast write                  → Slower write (6 operations instead of 1)
```

**Rule of thumb:** index columns you frequently `WHERE`, `JOIN`, or `ORDER BY` on (like `email`, `user_id`, `created_at`) — but don't index every column "just in case," especially on tables with heavy write traffic.

---

## Topics You Should Learn Next (beyond what you listed)

These are the natural next steps once the above feels automatic — skipping them is exactly where "backend developers hit a wall" as you put it.

### 1. Transactions & ACID

A transaction groups multiple queries so they **all succeed or all fail together** — critical for anything involving money or multi-step writes.

```
BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- debit
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- credit
COMMIT;

If the server crashes between the two UPDATEs:
  → Without a transaction: money vanishes (debited, never credited)
  → With a transaction: ROLLBACK happens, balance = 0 net change
```

```
ACID
A — Atomicity    → all steps succeed, or none do
C — Consistency  → database moves from one valid state to another
I — Isolation    → concurrent transactions don't corrupt each other
D — Durability   → once committed, it survives even a crash
```

### 2. Database Normalization

Organizing tables to **reduce duplicate data** and avoid update anomalies.

```
❌ Unnormalized (duplicated data)          ✅ Normalized
┌────┬───────┬──────────────┐             users              orders
│ id │ user  │  user_email  │             ┌────┬───────┐    ┌────┬─────────┐
├────┼───────┼──────────────┤             │ id │ email │    │ id │ user_id │
│ 1  │ Alice │alice@mail.com│             └────┴───────┘    └────┴─────────┘
│ 2  │ Alice │alice@mail.com│  ← duplicated
│ 3  │ Alice │alice@mail.com│  ← if email changes, must update everywhere
└────┴───────┴──────────────┘
```

### 3. N+1 Query Problem

A very common performance bug where fetching a list triggers **one extra query per row**, instead of one combined query.

```
❌ N+1 problem                              ✅ Fixed with JOIN or eager loading
1 query: SELECT * FROM users (10 users)     1 query:
Then, for EACH user:                        SELECT * FROM users
  SELECT * FROM orders WHERE user_id = 1    JOIN orders ON orders.user_id = users.id
  SELECT * FROM orders WHERE user_id = 2
  ... (10 more queries)                     → Total: 1 query instead of 11
Total: 11 queries for 10 users
```

### 4. EXPLAIN / Query Optimization

PostgreSQL can show you **exactly how it plans to execute a query** — essential for diagnosing slow queries.

```sql
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'bob@mail.com';
```

```
Seq Scan on users  (cost=0.00..180.00 rows=1)   ← 🚩 full table scan, slow!
   Filter: (email = 'bob@mail.com')

vs. after adding an index:

Index Scan using idx_user_email on users  (cost=0.29..8.31 rows=1)  ← ✅ fast
   Index Cond: (email = 'bob@mail.com')
```

### 5. Constraints (beyond keys)

```sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email TEXT UNIQUE NOT NULL,        -- UNIQUE + NOT NULL
  age INT CHECK (age >= 0),          -- CHECK constraint
  status TEXT DEFAULT 'active'       -- DEFAULT value
);
```

These push validation **into the database itself**, as a last line of defense even if application-level validation has a bug.

### 6. Migrations

Version-controlled, incremental changes to your database schema — so schema changes are tracked, reversible, and reproducible across dev/staging/production.

```
migrations/
  001_create_users_table.sql
  002_add_email_index.sql
  003_add_orders_table.sql
  004_add_foreign_key_orders_users.sql
```

```
Dev DB ──► migration 001,002,003,004 ──► Dev DB (up to date)
Prod DB ──► same migrations, same order ──► Prod DB (identical schema)
```

### 7. Connection Pooling

Opening a new database connection per request is expensive — a **pool** keeps a set of ready-to-use connections and hands them out on demand.

```
❌ Without pooling                      ✅ With pooling
Request → open connection → query        Request → borrow from pool → query
        → close connection                       → return to pool
(slow: connection setup every time)     (fast: connection reused)
```

### 8. Isolation Levels & Race Conditions

Determines how much concurrent transactions can "see" of each other's uncommitted work — important once multiple users hit the same rows simultaneously (e.g. two people buying the last item in stock).

```
Transaction A                    Transaction B
BEGIN                             BEGIN
SELECT stock FROM products         SELECT stock FROM products
  WHERE id=1  → stock = 1            WHERE id=1  → stock = 1
UPDATE stock = 0                   UPDATE stock = 0
COMMIT                             COMMIT
                                   
Result: Both think they got the last item → oversold! 
        (Isolation level / row locking prevents this)
```

---

## Quick-Reference: Why Each Concept Exists

| Concept | Necessity in one line |
|---|---|
| SELECT/INSERT/UPDATE/DELETE | The four basic ways to interact with data |
| JOIN | Combine related data living in separate tables |
| GROUP BY / HAVING | Aggregate and filter data by category |
| ORDER BY / LIMIT / OFFSET | Control result order and size (pagination) |
| Primary Key | Uniquely and reliably identify a row |
| Foreign Key | Enforce valid links between related tables |
| One-to-One / One-to-Many / Many-to-Many | Model how real-world entities relate |
| Index | Avoid full table scans — trade storage/write speed for read speed |
| Transactions / ACID | Guarantee multi-step writes succeed or fail together |
| Normalization | Avoid duplicated, inconsistent data |
| N+1 problem | Avoid accidentally issuing hundreds of extra queries |
| EXPLAIN | Diagnose *why* a query is slow, not just guess |
| Constraints | Enforce data correctness at the database level |
| Migrations | Track and reproduce schema changes safely |
| Connection pooling | Avoid the overhead of opening a DB connection per request |
| Isolation levels | Prevent race conditions when multiple users act at once |
