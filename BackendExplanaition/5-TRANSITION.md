# LEVEL 5 — Transactions (Crystal Clear Concept)

This is the level where "it works on my machine" stops being good enough. Transactions are what stand between your application and **silent data corruption** the moment two things happen at the same time — which, in production, is *constantly*. This guide builds the concept from the ground up, ending with the exact overselling scenario you described.

---

## 1. What Is a Transaction?

A transaction groups multiple queries into **one atomic unit** — either **all of them** succeed, or **none of them** do. There is no in-between state visible to the outside world.

```
BEGIN
   ↓
Query 1   (e.g. debit account A)
   ↓
Query 2   (e.g. credit account B)
   ↓
Query 3   (e.g. log the transfer)
   ↓
COMMIT     ← all 3 changes become permanent, together
```

If anything goes wrong before `COMMIT`:

```
BEGIN
   ↓
Query 1   ✓
   ↓
Query 2   ✗  (fails — e.g. constraint violation, crash, network drop)
   ↓
ROLLBACK   ← Query 1 is UNDONE too, as if nothing happened
```

```
                     BEGIN
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
     Query 1        Query 2        Query 3
        │              │              │
        └──────────────┼──────────────┘
                       │
              ┌────────┴────────┐
              ▼                 ▼
          All succeed        Any one fails
              │                 │
              ▼                 ▼
           COMMIT            ROLLBACK
        (all saved)      (all undone, DB
                          looks untouched)
```

### Real Example — Bank Transfer

```sql
BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- debit Alice
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- credit Bob
COMMIT;
```

```
Before:  Alice = $500     Bob = $200

If the server crashes right here ↓ (between the two UPDATEs)
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;  ✓ done
  ⚡ CRASH ⚡
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;  ✗ never ran

Without a transaction:  Alice = $400   Bob = $200   → $100 vanished!
With a transaction:     ROLLBACK on restart → Alice = $500  Bob = $200
                         (as if the transfer never started)
```

**This is the entire point of transactions:** money (or any related pair of writes) should never exist in a "half-done" state.

---

## 2. ACID — The Four Guarantees

```
A → Atomicity     "All or nothing"
C → Consistency   "Valid state to valid state"
I → Isolation     "Concurrent transactions don't interfere"
D → Durability    "Once committed, it survives anything"
```

### Atomicity

```
BEGIN
  Query 1 ✓
  Query 2 ✓
  Query 3 ✗  ← fails
ROLLBACK

Result: Query 1 and Query 2 are undone too.
There is no world where "2 out of 3" queries stick.
```

### Consistency

The database moves from one **valid** state to another valid state — all constraints, foreign keys, and rules are still satisfied after the transaction.

```
Constraint: balance >= 0

BEGIN
  UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
  -- would make balance = -500, violating the constraint
COMMIT  → ❌ REJECTED, automatic ROLLBACK
         (database refuses to enter an invalid state)
```

### Isolation

Concurrent transactions shouldn't see each other's **uncommitted** changes.

```
Transaction A                      Transaction B
BEGIN                               BEGIN
UPDATE stock = 5                    
                                     SELECT stock  → sees 10, NOT 5
                                     (A hasn't committed yet — B is
                                      "isolated" from A's in-progress work)
COMMIT
```

### Durability

Once `COMMIT` succeeds, the change is **permanent** — even if the server crashes 1 millisecond later.

```
COMMIT  →  written to disk (WAL: Write-Ahead Log)
              │
              ▼
        ⚡ Power loss ⚡
              │
              ▼
        Server restarts → change is STILL there
        (Postgres replays the WAL to recover it)
```

---

## 3. Transaction Isolation Levels

Isolation isn't all-or-nothing — SQL defines **levels** of isolation, each preventing different problems, at different cost to performance.

```
Weakest ────────────────────────────────────────► Strongest
(faster, more anomalies)              (slower, safest)

READ UNCOMMITTED → READ COMMITTED → REPEATABLE READ → SERIALIZABLE
```

### The problems each level solves

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| Read Uncommitted | ❌ Possible | ❌ Possible | ❌ Possible |
| Read Committed *(Postgres default)* | ✅ Prevented | ❌ Possible | ❌ Possible |
| Repeatable Read | ✅ Prevented | ✅ Prevented | ❌ Possible |
| Serializable | ✅ Prevented | ✅ Prevented | ✅ Prevented |

### Dirty Read (worst case, rarely allowed)

```
Transaction A                      Transaction B
BEGIN                               
UPDATE balance = 0                  BEGIN
                                     SELECT balance  → reads 0 (UNCOMMITTED!)
ROLLBACK  (A undoes it)             
                                     ...but B already acted on balance=0,
                                     which never actually existed
```

### Non-Repeatable Read

```
Transaction A                      Transaction B
BEGIN                               
SELECT balance → $500               
                                     BEGIN
                                     UPDATE balance = $300
                                     COMMIT
SELECT balance → $300   ← same transaction, DIFFERENT result!
                            (the row changed underneath A, mid-transaction)
```

### Phantom Read

```
Transaction A                      Transaction B
BEGIN
SELECT COUNT(*) FROM orders
  WHERE status='pending'  → 5
                                     BEGIN
                                     INSERT INTO orders (status) VALUES ('pending')
                                     COMMIT
SELECT COUNT(*) FROM orders
  WHERE status='pending'  → 6   ← a new "phantom" row appeared
```

**Postgres default is `READ COMMITTED`** — good enough for most apps, but for financial/inventory logic you often need stronger guarantees, which is where explicit locking comes in.

---

## 4. Race Conditions

A race condition happens when the **outcome depends on timing** — two operations "race" each other, and whichever happens to interleave in the wrong order produces a wrong result.

```
Two processes, same shared value, no coordination:

Time →  T1                  T2
        read stock=1
                             read stock=1     ← both read the SAME value
        stock = 1-1 = 0                       before either writes back
        write stock=0
                             stock = 1-1 = 0
                             write stock=0

Expected: 2 purchases → stock should be -1 (oversold, caught)
Actual:   stock = 0     → looks fine, but 2 items were sold with only 1 in stock!
```

This is a "race" because **if the timing were slightly different** (T2 reads *after* T1 writes), the result would correctly be `-1` or a rejected purchase. The bug only appears under concurrent load — which is exactly why it's so dangerous: it won't show up in normal manual testing.

---

## 5. The Classic Example — Overselling the Last Item

Let's trace your exact scenario in detail.

```
Product: "Limited Edition Sneakers"
stock = 1
```

```
Time    User A                          User B
─────   ──────────────────────────      ──────────────────────────
t0      SELECT stock FROM products
        WHERE id=1  → stock = 1
t1                                       SELECT stock FROM products
                                          WHERE id=1  → stock = 1
t2      if stock > 0: proceed ✓
t3                                        if stock > 0: proceed ✓
t4      UPDATE stock = stock - 1
        (stock: 1 → 0)
t5                                        UPDATE stock = stock - 1
                                           (stock: 0 → -1)   ❌
t6      COMMIT (order created)
t7                                        COMMIT (order created)

RESULT: 2 orders created, stock = -1, but only 1 item existed!
```

```
              stock = 1
                  │
        ┌─────────┴─────────┐
        ▼                   ▼
    User A reads         User B reads
    stock = 1            stock = 1
    (before A writes)    (before A writes)
        │                   │
        ▼                   ▼
    "Available!"         "Available!"
        │                   │
        ▼                   ▼
    Order created        Order created
        │                   │
        └─────────┬─────────┘
                  ▼
         Both think they succeeded
         Physical stock: only 1 item
         Database stock: -1
```

**Why this happens:** the `SELECT` (read) and `UPDATE` (write) are **two separate steps**, and nothing stops another transaction from reading the *same* stale value in between. Without extra protection, "check then act" is inherently unsafe under concurrency.

---

## 6. Row Locking — The Core Fix

A **row lock** prevents other transactions from reading/modifying a specific row until the current transaction finishes.

```
User A                              User B
BEGIN
SELECT ... FOR UPDATE  ← locks the row
(stock = 1)
                                     BEGIN
                                     SELECT ... FOR UPDATE
                                     ← BLOCKED, must WAIT for A to finish
UPDATE stock = 0
COMMIT  (lock released)
                                     ← now proceeds
                                     SELECT ... FOR UPDATE → stock = 0
                                     if stock > 0: ❌ false → reject purchase
                                     COMMIT
```

```
        Row: stock = 1
              │
              ▼
      ┌─────────────────┐
      │   🔒 LOCKED       │  ← User A holds the lock
      │   by User A       │
      └─────────────────┘
              │
      User B tries to lock same row
              │
              ▼
      ┌─────────────────┐
      │   ⏳ User B WAITS  │
      └─────────────────┘
              │
      User A commits → lock released
              │
              ▼
      User B's lock granted → sees updated stock (0)
      → correctly rejects the purchase
```

---

## 7. `SELECT FOR UPDATE`

This is the actual SQL mechanism that creates a row lock, used *inside* a transaction.

```sql
BEGIN;
  SELECT stock FROM products WHERE id = 1 FOR UPDATE;  -- locks this row
  -- ... application checks stock > 0 ...
  UPDATE products SET stock = stock - 1 WHERE id = 1;
COMMIT;  -- lock released here
```

```
FOR UPDATE says:
"I'm about to change this row — nobody else may read it for the
 purpose of updating it until I'm done."

Without FOR UPDATE:              With FOR UPDATE:
SELECT just reads a snapshot     SELECT reserves the row
→ another transaction can        → another transaction attempting
  read the same stale value        FOR UPDATE on the same row
  freely                           must wait in line
```

### Applied to the sneakers example

```sql
BEGIN;
  SELECT stock FROM products WHERE id = 1 FOR UPDATE;
  -- User A gets stock = 1, User B is now BLOCKED and waits

  -- User A's application logic:
  IF stock > 0 THEN
    UPDATE products SET stock = stock - 1 WHERE id = 1;
    INSERT INTO orders (...) VALUES (...);
  END IF;
COMMIT;
-- User B's SELECT FOR UPDATE now unblocks, reads stock = 0
-- User B's application logic correctly rejects the purchase
```

```
Result now:  stock = 0  (correct!)
             1 order created (correct!)
             User B sees "Out of stock" instead of an oversold order
```

---

## 8. Deadlocks

A deadlock happens when two transactions are each **waiting on a lock the other one holds** — neither can proceed, forever, unless something intervenes.

```
Transaction A                      Transaction B
BEGIN                               BEGIN
LOCK row 1 (products)               LOCK row 2 (orders)
   ↓                                    ↓
Try to LOCK row 2                   Try to LOCK row 1
   ↓                                    ↓
WAITING for B to release row 2      WAITING for A to release row 1

           ┌─────────┐         ┌─────────┐
           │  Txn A   │ ─wants→│  Row 2   │ ← held by Txn B
           └─────────┘         └─────────┘
                ▲                    │
                │                    │ Txn B wants
             held by                 ▼
           ┌─────────┐         ┌─────────┐
           │  Row 1   │ ◄──────│  Txn B   │
           └─────────┘         └─────────┘

Neither can ever finish — a circular wait. 🔒🔁🔒
```

**How databases handle it:** Postgres automatically **detects** this cycle and forcibly `ROLLBACK`s one of the two transactions (the "deadlock victim"), returning an error the application must catch and typically retry.

```sql
ERROR: deadlock detected
DETAIL: Process 123 waits for ShareLock on transaction 456;
        blocked by process 456.
```

**How to avoid deadlocks in practice:** always lock rows/tables in the **same consistent order** across your entire codebase (e.g. always lock `products` before `orders`, never the reverse) — this removes the possibility of a circular wait.

---

## 9. Optimistic vs Pessimistic Locking

Two fundamentally different **strategies** for handling concurrent writes to the same row.

### Pessimistic Locking (what `SELECT FOR UPDATE` does)

**Assumption:** conflicts are likely, so prevent them upfront by locking before anyone else can touch the row.

```
User A: "I'm locking this row RIGHT NOW, before I even
          decide what to do with it."

SELECT ... FOR UPDATE   ← lock acquired immediately
[...think, validate, decide...]
UPDATE ...
COMMIT                    ← lock released
```

```
Pros: guarantees correctness, simple to reason about
Cons: other transactions must WAIT (blocked), can hurt throughput
      under heavy contention, risk of deadlocks
```

### Optimistic Locking (check at the end, not the start)

**Assumption:** conflicts are rare, so don't lock anything upfront — just verify at the moment of writing that nothing changed since you read it. Usually implemented with a `version` column.

```sql
-- Step 1: read the row, note its version
SELECT stock, version FROM products WHERE id = 1;
-- → stock = 1, version = 7

-- Step 2: update ONLY if version hasn't changed
UPDATE products
SET stock = stock - 1, version = version + 1
WHERE id = 1 AND version = 7;

-- If another transaction already updated it (version is now 8),
-- this UPDATE affects 0 rows → application detects failure → retry
```

```
User A reads: stock=1, version=7
User B reads: stock=1, version=7    ← both read the same version

User A writes: WHERE version=7 → ✓ succeeds, version becomes 8
User B writes: WHERE version=7 → ✗ 0 rows affected (version is now 8!)
                                    → app detects conflict, retries,
                                      re-reads stock=0, correctly
                                      rejects the purchase
```

```
Pros: no waiting/blocking, better performance under low contention
Cons: requires retry logic in the application, wasted work if
      conflicts DO happen often (defeats the "optimism")
```

### When to use which

```
High contention (many users fighting over the SAME row)
  → e.g. flash sale, last item in stock
  → Pessimistic locking (SELECT FOR UPDATE) is usually safer

Low contention (conflicts are rare, most updates don't collide)
  → e.g. editing a user profile, a blog post
  → Optimistic locking scales better, avoids unnecessary blocking
```

---

## 10. Full Picture — Solving the Sneakers Problem, Two Ways

```
❌ NO PROTECTION                    ✅ PESSIMISTIC (FOR UPDATE)
SELECT stock → 1                    BEGIN
UPDATE stock = stock-1                SELECT stock FOR UPDATE → 1
(race condition possible)             (User B BLOCKS here)
                                       UPDATE stock = stock-1
                                     COMMIT (lock released, B proceeds
                                             safely, sees stock=0)

✅ OPTIMISTIC (version column)
SELECT stock, version → 1, v7
UPDATE ... WHERE version=v7
  → succeeds for A, fails for B (0 rows affected)
  → B retries, sees stock=0, correctly blocked
```

---

## Quick-Reference: Why Each Concept Exists

| Concept | Necessity in one line |
|---|---|
| Transaction (BEGIN/COMMIT/ROLLBACK) | Group multiple writes so they succeed or fail as one unit |
| Atomicity | No "half-done" state ever visible |
| Consistency | Database never lands in a state that violates its own rules |
| Isolation | Concurrent transactions don't see each other's unfinished work |
| Durability | A committed change survives even a crash |
| Isolation levels | Trade correctness guarantees against performance/concurrency |
| Race condition | Explains *why* concurrent read-then-write logic breaks |
| Row locking | Prevents two transactions from acting on stale data simultaneously |
| `SELECT FOR UPDATE` | The actual SQL tool that creates a pessimistic row lock |
| Deadlock | Two transactions waiting on each other forever — must be detected/avoided |
| Pessimistic locking | Best for high-contention, "last item in stock"-style conflicts |
| Optimistic locking | Best for low-contention updates, avoids blocking, needs retry logic |
