# MODULE 8: Transactions and Concurrency (ট্রানজেকশন এবং কনকারেন্সি)
## বাংলায় সম্পূর্ণ গাইড — Production Level

> **কেন এই module critical?**
> Banking system এ টাকা transfer হচ্ছে। Payment gateway এ charge হচ্ছে। Inventory তে stock কমছে।
> এই সব জায়গায় একটা ভুল মানে: টাকা দুইবার কাটা, stock negative হওয়া, data corruption।
> Transactions এবং Concurrency না জানলে তুমি financial system এ কাজ করতে পারবে না।
> FAANG interview এ এই topic থেকে scenario-based questions সবচেয়ে বেশি আসে।

---

## 8.1 Transaction কী? (পুনরালোচনা + Deep Dive)

### সহজ সংজ্ঞা

Transaction হলো একগুচ্ছ SQL operation যেগুলো একসাথে হয় — অথবা কিছুই হয় না।

```
উদাহরণ — ব্যাংক Transfer:
Rahim এর account থেকে Karim এর account এ ১০০০ টাকা পাঠাবে।

Step 1: Rahim এর balance থেকে ১০০০ বাদ দাও
Step 2: Karim এর balance তে ১০০০ যোগ করো

যদি Step 1 হয় কিন্তু Step 2 হওয়ার আগে server crash করে?
→ Rahim এর ১০০০ টাকা গেছে, Karim পায়নি → টাকা উধাও!

Transaction দিয়ে:
→ দুটো step একসাথে commit হবে
→ অথবা দুটোই rollback হবে
→ মাঝখানে কোনো state নেই
```

### ACID Properties (বিস্তারিত)

```
A — Atomicity (অ্যাটমিসিটি)
    সব operations হবে অথবা কিছুই হবে না।
    "All or Nothing"

C — Consistency (কনসিস্টেন্সি)
    Transaction এর আগে এবং পরে database valid state এ থাকবে।
    Constraints, rules সব মানতে হবে।
    (balance negative হবে না, FK valid থাকবে)

I — Isolation (আইসোলেশন)
    একটা transaction চলার সময় অন্য transaction তার
    intermediate state দেখতে পাবে না।

D — Durability (ডিউরেবিলিটি)
    Commit হয়ে গেলে server crash করলেও data থাকবে।
    WAL (Write-Ahead Log) এটা নিশ্চিত করে।
```

### Basic Transaction SQL

```sql
-- Manual transaction
BEGIN;

UPDATE accounts SET balance = balance - 1000 WHERE id = 1;  -- Rahim
UPDATE accounts SET balance = balance + 1000 WHERE id = 2;  -- Karim

-- সব ঠিক থাকলে
COMMIT;

-- কোনো সমস্যা হলে
ROLLBACK;
```

### Django ORM Transaction

```python
from django.db import transaction

# Method 1: atomic() decorator
@transaction.atomic
def transfer_money(from_account_id, to_account_id, amount):
    from_acc = Account.objects.select_for_update().get(id=from_account_id)
    to_acc   = Account.objects.select_for_update().get(id=to_account_id)

    if from_acc.balance < amount:
        raise ValueError("Insufficient balance")

    from_acc.balance -= amount
    to_acc.balance   += amount

    from_acc.save()
    to_acc.save()
    # Exception হলে automatically rollback হবে

# Method 2: context manager
def transfer_money_v2(from_id, to_id, amount):
    with transaction.atomic():
        from_acc = Account.objects.select_for_update().get(id=from_id)
        to_acc   = Account.objects.select_for_update().get(id=to_id)

        from_acc.balance -= amount
        to_acc.balance   += amount

        from_acc.save()
        to_acc.save()
```

```sql
-- Generated SQL:
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
SELECT * FROM accounts WHERE id = 2 FOR UPDATE;
UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
UPDATE accounts SET balance = balance + 1000 WHERE id = 2;
COMMIT;
```

---

## 8.2 Isolation Levels (আইসোলেশন লেভেল)

### কেন দরকার?

Multiple transactions একসাথে চললে বিভিন্ন সমস্যা আসতে পারে। Isolation level নির্ধারণ করে কোন সমস্যা allow করবে আর কোনটা করবে না।

```
৪টা Isolation Level (weak → strong):

READ UNCOMMITTED  →  সবচেয়ে কম isolation
READ COMMITTED    →  PostgreSQL এর default
REPEATABLE READ   →  MySQL এর default
SERIALIZABLE      →  সবচেয়ে বেশি isolation
```

### ৩টা Concurrency সমস্যা

```
1. Dirty Read       — uncommitted data পড়া
2. Non-Repeatable Read — একই query দুইবার চালালে different result
3. Phantom Read     — একই range query তে নতুন rows দেখা
```

---

## 8.3 Dirty Read (ডার্টি রিড)

### কী সমস্যা?

```
Transaction A এর uncommitted change Transaction B পড়তে পারছে।
A যদি পরে rollback করে → B ভুল data পেয়েছে!
```

### Real Example — Payment System

```
Timeline:

Transaction A (Payment processing):          Transaction B (Balance check):
                                             
t1: BEGIN                                    
t2: UPDATE balance SET amount = 5000         
    WHERE user_id = 1                        
    (not committed yet)                      
                                    t3:      BEGIN
                                    t4:      SELECT balance FROM accounts
                                             WHERE user_id = 1
                                             → 5000 দেখাচ্ছে (dirty read!)
t5: ROLLBACK (payment failed!)               
                                    t6:      "Balance: 5000" দেখাচ্ছে
                                             কিন্তু actual balance 4000!
```

```sql
-- READ UNCOMMITTED এ dirty read হয় (PostgreSQL এ এই level নেই effectively)
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;

-- READ COMMITTED এ dirty read হয় না (PostgreSQL default)
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

---

## 8.4 Non-Repeatable Read (নন-রিপিটেবল রিড)

### কী সমস্যা?

```
একটা transaction এর মধ্যে একই query দুইবার চালালে
ভিন্ন result পাওয়া — কারণ মাঝে অন্য transaction data পরিবর্তন করেছে।
```

### Real Example — Inventory System

```
Transaction A (Order processing):           Transaction B (Stock update):

t1: BEGIN
t2: SELECT stock FROM products              
    WHERE id = 1 → stock = 10              
                                   t3:      BEGIN
                                   t4:      UPDATE products SET stock = 0
                                            WHERE id = 1
                                   t5:      COMMIT
t6: SELECT stock FROM products
    WHERE id = 1 → stock = 0  ← different!
t7: (confused — which value is correct?)
```

```sql
-- READ COMMITTED এ non-repeatable read হয়
-- REPEATABLE READ এ হয় না
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

BEGIN;
SELECT stock FROM products WHERE id = 1;  -- 10
-- অন্য transaction update করলেও...
SELECT stock FROM products WHERE id = 1;  -- এখনো 10 (snapshot)
COMMIT;
```

```python
# Django ORM
from django.db import transaction

def process_order(product_id, quantity):
    with transaction.atomic():
        # REPEATABLE READ isolation
        # Django তে manually set করতে হয়
        from django.db import connection
        connection.cursor().execute(
            "SET TRANSACTION ISOLATION LEVEL REPEATABLE READ"
        )
        
        product = Product.objects.get(id=product_id)
        first_check = product.stock  # 10
        
        # ... কিছু processing ...
        
        product.refresh_from_db()
        second_check = product.stock  # REPEATABLE READ এ এখনো 10
```

---

## 8.5 Phantom Read (ফ্যান্টম রিড)

### কী সমস্যা?

```
একটা transaction range query দুইবার করলে
দ্বিতীয়বার নতুন rows দেখা যাচ্ছে (phantom rows) —
কারণ অন্য transaction নতুন rows insert করেছে।
```

### Real Example — Banking Report

```
Transaction A (Monthly Report):              Transaction B (New Account):

t1: BEGIN
t2: SELECT COUNT(*) FROM accounts
    WHERE balance > 10000 → 150 accounts
                                    t3:      BEGIN
                                    t4:      INSERT INTO accounts
                                             (balance) VALUES (15000)
                                    t5:      COMMIT
t6: SELECT COUNT(*) FROM accounts
    WHERE balance > 10000 → 151 accounts ← Phantom!
    (নতুন একটা row এসে গেছে!)
```

```sql
-- SERIALIZABLE isolation এ phantom read হয় না
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

BEGIN;
SELECT COUNT(*) FROM accounts WHERE balance > 10000;  -- 150
-- অন্য transaction insert করলেও...
SELECT COUNT(*) FROM accounts WHERE balance > 10000;  -- এখনো 150
COMMIT;
```

---

## 8.6 Isolation Levels Summary Table

```
┌──────────────────────┬──────────────┬─────────────────────┬──────────────┐
│ Isolation Level      │ Dirty Read   │ Non-Repeatable Read │ Phantom Read │
├──────────────────────┼──────────────┼─────────────────────┼──────────────┤
│ READ UNCOMMITTED     │ ✅ হতে পারে  │ ✅ হতে পারে         │ ✅ হতে পারে  │
│ READ COMMITTED       │ ❌ হবে না    │ ✅ হতে পারে         │ ✅ হতে পারে  │
│ REPEATABLE READ      │ ❌ হবে না    │ ❌ হবে না           │ ✅ হতে পারে  │
│ SERIALIZABLE         │ ❌ হবে না    │ ❌ হবে না           │ ❌ হবে না    │
└──────────────────────┴──────────────┴─────────────────────┴──────────────┘

Performance cost: READ UNCOMMITTED < READ COMMITTED < REPEATABLE READ < SERIALIZABLE
                  (fast)                                                    (slow)

PostgreSQL default: READ COMMITTED
MySQL default:      REPEATABLE READ
```

### Django তে Isolation Level Set করা

```python
# settings.py — globally set করো
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'OPTIONS': {
            'isolation_level': 'read committed',  # default
            # 'isolation_level': 'repeatable read',
            # 'isolation_level': 'serializable',
        }
    }
}

# Per-transaction set করো
import psycopg2.extensions

def critical_operation():
    with transaction.atomic():
        from django.db import connection
        with connection.cursor() as cursor:
            cursor.execute(
                "SET TRANSACTION ISOLATION LEVEL SERIALIZABLE"
            )
        # এখন এই transaction SERIALIZABLE
        do_critical_work()
```

---

## 8.7 Locks (লক)

### কেন দরকার?

```
দুটো transaction একই row update করতে চাইছে:

Transaction A: SELECT balance → 1000, তারপর UPDATE balance = 500
Transaction B: SELECT balance → 1000, তারপর UPDATE balance = 800

কোনটা জিতবে? Last write wins → data loss!
Lock দিয়ে এটা prevent করা হয়।
```

### Row-Level Locking

```sql
-- SELECT FOR UPDATE — row lock করো
-- অন্য transaction এই row update করতে পারবে না যতক্ষণ lock release না হয়

BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
-- এখন এই row locked। অন্য কেউ এই row এ SELECT FOR UPDATE করলে wait করবে।
UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
COMMIT;
-- Lock release হলো

-- SELECT FOR SHARE — read lock (অন্যরা read করতে পারবে কিন্তু write না)
SELECT * FROM accounts WHERE id = 1 FOR SHARE;
```

```python
# Django ORM — select_for_update()
def transfer_money(from_id, to_id, amount):
    with transaction.atomic():
        # Row lock করো — অন্য transaction wait করবে
        accounts = Account.objects.select_for_update().filter(
            id__in=[from_id, to_id]
        ).order_by('id')  # Deadlock avoid করতে consistent order!

        from_acc = next(a for a in accounts if a.id == from_id)
        to_acc   = next(a for a in accounts if a.id == to_id)

        if from_acc.balance < amount:
            raise InsufficientFundsError()

        from_acc.balance -= amount
        to_acc.balance   += amount

        Account.objects.bulk_update([from_acc, to_acc], ['balance'])
```

```sql
-- Generated SQL:
BEGIN;
SELECT * FROM accounts WHERE id IN (1, 2) ORDER BY id FOR UPDATE;
UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
UPDATE accounts SET balance = balance + 1000 WHERE id = 2;
COMMIT;
```

### NOWAIT এবং SKIP LOCKED

```python
# NOWAIT — lock না পেলে immediately error throw করো (wait করো না)
try:
    account = Account.objects.select_for_update(nowait=True).get(id=1)
except OperationalError:
    raise Exception("Account is being processed by another transaction")

# SKIP LOCKED — locked rows skip করো (queue processing এ useful)
# Celery worker দিয়ে orders process করার সময়
pending_orders = Order.objects.select_for_update(
    skip_locked=True
).filter(status='pending')[:10]
# অন্য worker যে orders process করছে সেগুলো skip করবে
```

```sql
-- Generated SQL:
SELECT * FROM orders WHERE status = 'pending'
LIMIT 10
FOR UPDATE SKIP LOCKED;
```

---

## 8.8 Optimistic Locking (অপ্টিমিস্টিক লকিং)

### কী এবং কেন?

```
Pessimistic Locking: "ধরে নিচ্ছি conflict হবে, তাই lock করে রাখো"
Optimistic Locking:  "ধরে নিচ্ছি conflict হবে না, শেষে check করো"

Optimistic Locking কখন ভালো:
- Read বেশি, Write কম
- Lock এর overhead avoid করতে চাই
- Distributed system এ

Mechanism:
- Table এ version বা updated_at column রাখো
- Read করার সময় version নোট করো
- Update করার সময় WHERE version = old_version দাও
- 0 rows update হলে → conflict! retry করো
```

### SQL Implementation

```sql
-- version column সহ table
CREATE TABLE products (
    id      BIGSERIAL PRIMARY KEY,
    name    VARCHAR(200),
    price   DECIMAL(10,2),
    stock   INT,
    version INT DEFAULT 1
);

-- Optimistic update
UPDATE products
SET
    stock   = stock - 1,
    version = version + 1
WHERE id = 1
  AND version = 5;  -- আমরা version 5 পেয়েছিলাম

-- Rows affected check করো
-- 0 rows → conflict (অন্য কেউ আগে update করেছে)
-- 1 row  → success
```

### Django ORM Implementation

```python
from django.db import transaction
from django.core.exceptions import ValidationError

class Product(models.Model):
    name    = models.CharField(max_length=200)
    stock   = models.IntegerField()
    version = models.IntegerField(default=1)

def purchase_product(product_id, quantity, max_retries=3):
    for attempt in range(max_retries):
        # Read current state
        product = Product.objects.get(id=product_id)

        if product.stock < quantity:
            raise ValidationError("Insufficient stock")

        # Optimistic update — version check
        updated = Product.objects.filter(
            id=product_id,
            version=product.version  # conflict check
        ).update(
            stock=product.stock - quantity,
            version=product.version + 1
        )

        if updated == 1:
            return True  # Success!
        
        # Conflict — retry
        if attempt == max_retries - 1:
            raise Exception("Could not complete purchase after retries")
    
    return False
```

```sql
-- Generated SQL:
SELECT * FROM products WHERE id = 1;
-- version = 5, stock = 10

UPDATE products
SET stock = 9, version = 6
WHERE id = 1 AND version = 5;
-- 1 row updated → success!
-- 0 rows updated → conflict → retry
```

### Django Built-in: select_for_update vs Optimistic

```python
# Pessimistic (select_for_update) — high contention এ deadlock risk
with transaction.atomic():
    product = Product.objects.select_for_update().get(id=1)
    product.stock -= 1
    product.save()

# Optimistic — high read, low write এ better
updated = Product.objects.filter(
    id=1, version=current_version
).update(stock=F('stock') - 1, version=F('version') + 1)
if not updated:
    raise ConflictError("Retry needed")
```

---

## 8.9 Pessimistic Locking (পেসিমিস্টিক লকিং)

```
কখন ব্যবহার করবে:
✅ Write contention বেশি (banking, payment)
✅ Critical sections (inventory checkout)
✅ Short transactions

কখন এড়াবে:
❌ Long-running transactions (deadlock risk)
❌ Read-heavy system (unnecessary overhead)
❌ Distributed transactions (lock management complex)
```

### Banking Example — Full Implementation

```python
# accounts/services.py
from django.db import transaction
from django.db.models import F
from decimal import Decimal

class BankingService:
    
    @staticmethod
    @transaction.atomic
    def transfer(from_account_id, to_account_id, amount: Decimal):
        """
        Thread-safe money transfer with pessimistic locking.
        Deadlock avoid করতে সবসময় consistent order এ lock করো।
        """
        # Consistent ordering — always lock smaller id first
        ids = sorted([from_account_id, to_account_id])
        
        accounts = {
            acc.id: acc
            for acc in Account.objects.select_for_update().filter(id__in=ids)
        }
        
        from_acc = accounts[from_account_id]
        to_acc   = accounts[to_account_id]
        
        # Validations
        if from_acc.balance < amount:
            raise InsufficientFundsError(
                f"Balance {from_acc.balance} < transfer amount {amount}"
            )
        if from_acc.status != 'active':
            raise AccountInactiveError()
        
        # Atomic updates
        Account.objects.filter(id=from_account_id).update(
            balance=F('balance') - amount
        )
        Account.objects.filter(id=to_account_id).update(
            balance=F('balance') + amount
        )
        
        # Audit log
        Transaction.objects.create(
            from_account=from_acc,
            to_account=to_acc,
            amount=amount,
            type='transfer'
        )
        
        return True
```

---

## 8.10 Deadlocks (ডেডলক)

### কী এবং কেন হয়?

```
Deadlock: দুটো transaction পরস্পরের lock এর জন্য অপেক্ষা করছে।

Transaction A:                    Transaction B:
  Lock Row 1 ✅                     Lock Row 2 ✅
  Wait for Row 2... 🔄              Wait for Row 1... 🔄
  (B already has Row 2)             (A already has Row 1)
  
= DEADLOCK! কেউ এগোতে পারছে না।

PostgreSQL automatically detect করে এবং একটাকে kill করে।
Error: "ERROR: deadlock detected"
```

### Real Example — Payment System Deadlock

```sql
-- Transaction A: Rahim → Karim (Row 1 then Row 2)
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;  -- Lock Rahim (Row 1)
-- এখন Karim lock করতে চাই কিন্তু B এর কাছে আছে...
SELECT * FROM accounts WHERE id = 2 FOR UPDATE;  -- WAIT!

-- Transaction B: Karim → Rahim (Row 2 then Row 1) - same time!
BEGIN;
SELECT * FROM accounts WHERE id = 2 FOR UPDATE;  -- Lock Karim (Row 2)
-- এখন Rahim lock করতে চাই কিন্তু A এর কাছে আছে...
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;  -- WAIT!

-- DEADLOCK! PostgreSQL একটাকে rollback করবে।
```

### Deadlock Prevention — Consistent Lock Ordering

```python
# ❌ Deadlock prone
def transfer_bad(from_id, to_id, amount):
    with transaction.atomic():
        from_acc = Account.objects.select_for_update().get(id=from_id)
        to_acc   = Account.objects.select_for_update().get(id=to_id)
        # Thread A: lock 1 then 2
        # Thread B: lock 2 then 1 → DEADLOCK!

# ✅ Deadlock safe — always lock in consistent order
def transfer_safe(from_id, to_id, amount):
    with transaction.atomic():
        # সবসময় ছোট id আগে lock করো
        first_id  = min(from_id, to_id)
        second_id = max(from_id, to_id)

        accounts = Account.objects.select_for_update().filter(
            id__in=[first_id, second_id]
        ).order_by('id')  # consistent order!

        accs = {a.id: a for a in accounts}
        from_acc = accs[from_id]
        to_acc   = accs[to_id]

        from_acc.balance -= amount
        to_acc.balance   += amount
        Account.objects.bulk_update([from_acc, to_acc], ['balance'])
```

### Deadlock Handling — Retry Logic

```python
import time
import random
from django.db import transaction, OperationalError

def with_retry(func, max_retries=3, base_delay=0.1):
    """Deadlock retry decorator"""
    for attempt in range(max_retries):
        try:
            return func()
        except OperationalError as e:
            if 'deadlock' in str(e).lower() and attempt < max_retries - 1:
                # Exponential backoff with jitter
                delay = base_delay * (2 ** attempt) + random.uniform(0, 0.1)
                time.sleep(delay)
                continue
            raise

# Usage
def process_payment(order_id):
    def _process():
        with transaction.atomic():
            order = Order.objects.select_for_update().get(id=order_id)
            # ... payment logic
    
    with_retry(_process)
```

---

## 8.11 Real Production Systems

### Inventory System — Flash Sale (Daraz/Shajgoj style)

```python
# Flash sale: 100টা product, 10,000 জন একসাথে কিনতে চাইছে

# ❌ Race condition — overselling হবে!
def purchase_flash_sale_bad(product_id, user_id):
    product = Product.objects.get(id=product_id)
    if product.stock > 0:
        product.stock -= 1
        product.save()
        Order.objects.create(product=product, user_id=user_id)
        return True
    return False
# দুজন একসাথে stock=1 পাবে, দুজনেই order করবে → stock=-1!

# ✅ Pessimistic lock দিয়ে oversell prevention
@transaction.atomic
def purchase_flash_sale(product_id, user_id):
    try:
        product = Product.objects.select_for_update(
            nowait=True  # অন্য transaction lock এ থাকলে immediately fail
        ).get(id=product_id)
    except OperationalError:
        raise Exception("Product is being purchased by someone else, try again")

    if product.stock <= 0:
        raise Exception("Out of stock")

    product.stock -= 1
    product.save()

    return Order.objects.create(
        product=product,
        user_id=user_id,
        price=product.price
    )

# ✅ আরো better — F() + conditional update (no explicit lock)
@transaction.atomic
def purchase_flash_sale_v2(product_id, user_id):
    updated = Product.objects.filter(
        id=product_id,
        stock__gt=0  # stock > 0 হলেই update করো
    ).update(stock=F('stock') - 1)

    if not updated:
        raise Exception("Out of stock")

    product = Product.objects.get(id=product_id)
    return Order.objects.create(product=product, user_id=user_id)
```

```sql
-- Generated SQL (v2):
UPDATE products
SET stock = stock - 1
WHERE id = 1 AND stock > 0;
-- 1 row → success, 0 rows → out of stock
-- Database level atomic — no race condition!
```

### Payment System — Stripe Style Idempotency

```python
# Payment duplicate হওয়া prevent করো
class Payment(models.Model):
    idempotency_key = models.CharField(max_length=255, unique=True)
    order           = models.ForeignKey(Order, on_delete=models.PROTECT)
    amount          = models.DecimalField(max_digits=10, decimal_places=2)
    status          = models.CharField(max_length=20, default='pending')
    created_at      = models.DateTimeField(auto_now_add=True)

@transaction.atomic
def process_payment(order_id, amount, idempotency_key):
    # Idempotency check — same key দিয়ে দুইবার call করলে same result
    payment, created = Payment.objects.get_or_create(
        idempotency_key=idempotency_key,
        defaults={
            'order_id': order_id,
            'amount':   amount,
            'status':   'pending'
        }
    )
    
    if not created:
        return payment  # Already processed, return existing

    try:
        # Charge করো (Stripe API call etc.)
        charge_result = payment_gateway.charge(amount)
        payment.status = 'completed'
        payment.save()
    except Exception:
        payment.status = 'failed'
        payment.save()
        raise
    
    return payment
```

### E-commerce Order Placement — Full Transaction

```python
@transaction.atomic
def place_order(user_id, cart_items):
    """
    Full order placement with:
    - Stock check and reservation
    - Order creation
    - Payment initiation
    - Stock deduction
    """
    # Step 1: Lock all products (consistent order to avoid deadlock)
    product_ids = sorted([item['product_id'] for item in cart_items])
    products = {
        p.id: p
        for p in Product.objects.select_for_update().filter(id__in=product_ids)
    }
    
    # Step 2: Validate stock
    for item in cart_items:
        product = products[item['product_id']]
        if product.stock < item['quantity']:
            raise InsufficientStockError(
                f"Only {product.stock} units available for {product.name}"
            )
    
    # Step 3: Create order
    order = Order.objects.create(
        user_id=user_id,
        status='pending',
        total_amount=sum(
            products[i['product_id']].price * i['quantity']
            for i in cart_items
        )
    )
    
    # Step 4: Create order items + deduct stock
    order_items = []
    for item in cart_items:
        product = products[item['product_id']]
        order_items.append(OrderItem(
            order=order,
            product=product,
            quantity=item['quantity'],
            unit_price=product.price
        ))
        product.stock -= item['quantity']
    
    OrderItem.objects.bulk_create(order_items)
    Product.objects.bulk_update(
        list(products.values()), ['stock']
    )
    
    return order
```

---

## 8.12 Savepoints (সেভপয়েন্ট)

```python
# Nested transaction — partial rollback
@transaction.atomic  # outer transaction
def complex_operation():
    create_order()  # committed if success

    try:
        with transaction.atomic():  # savepoint তৈরি হবে
            send_notification()     # এটা fail হলে...
    except Exception:
        pass  # শুধু notification rollback, order ঠিক থাকবে

    update_inventory()  # এটা চলবে
```

```sql
-- SQL Savepoint:
BEGIN;
INSERT INTO orders ...;          -- success

SAVEPOINT before_notification;
INSERT INTO notifications ...;   -- fail হলে...
ROLLBACK TO SAVEPOINT before_notification;  -- শুধু notification rollback

UPDATE inventory ...;            -- এটা চলবে
COMMIT;
```

---

## 8.13 Common Mistakes

### ভুল ১: Transaction এর বাইরে External API Call

```python
# ❌ ভয়ঙ্কর ভুল!
@transaction.atomic
def place_order(user_id, cart):
    order = Order.objects.create(user_id=user_id, ...)
    
    # External API call transaction এর ভেতরে!
    stripe.charge(amount)  # 5 seconds নিচ্ছে → DB connection held!
    # এই 5 seconds এ DB connection block থাকবে → connection pool শেষ!
    
    order.status = 'paid'
    order.save()

# ✅ সঠিক পদ্ধতি
def place_order(user_id, cart):
    # DB operations একসাথে করো
    with transaction.atomic():
        order = Order.objects.create(user_id=user_id, status='pending', ...)
        OrderItem.objects.bulk_create([...])
    
    # Transaction এর বাইরে external API call
    try:
        charge = stripe.charge(amount)
        Order.objects.filter(id=order.id).update(
            status='paid',
            stripe_charge_id=charge.id
        )
    except stripe.error.CardError:
        Order.objects.filter(id=order.id).update(status='payment_failed')
        raise
```

### ভুল ২: Long Transaction — Connection Pool শেষ

```python
# ❌ দীর্ঘ transaction — connection আটকে থাকে
@transaction.atomic
def bad_batch_processing():
    for order in Order.objects.filter(status='pending'):
        process_order(order)     # সময় নেয়
        send_email(order)        # আরো সময়
        call_external_api(order) # আরো বেশি সময়
    # পুরো সময় DB connection locked!

# ✅ Short transactions
def good_batch_processing():
    order_ids = list(
        Order.objects.filter(status='pending').values_list('id', flat=True)
    )
    
    for order_id in order_ids:
        with transaction.atomic():  # প্রতিটার জন্য short transaction
            order = Order.objects.select_for_update().get(id=order_id)
            process_order_db_operations(order)
        
        # Transaction এর বাইরে
        send_email(order)
        call_external_api(order)
```

### ভুল ৩: on_commit() না জানা

```python
# ❌ Transaction rollback হলেও email চলে যাবে!
@transaction.atomic
def create_user(email):
    user = User.objects.create(email=email)
    send_welcome_email(user)  # Transaction fail হলেও email যাবে!
    return user

# ✅ on_commit — commit হওয়ার পরই email পাঠাও
@transaction.atomic
def create_user(email):
    user = User.objects.create(email=email)
    
    transaction.on_commit(
        lambda: send_welcome_email.delay(user.id)  # Celery task
    )
    return user
```

---

## 8.14 Debugging Techniques

```sql
-- Active locks দেখো
SELECT
    pid,
    usename,
    wait_event_type,
    wait_event,
    state,
    query,
    now() - pg_stat_activity.query_start AS duration
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC;

-- Lock conflicts দেখো
SELECT
    blocked.pid     AS blocked_pid,
    blocked.query   AS blocked_query,
    blocking.pid    AS blocking_pid,
    blocking.query  AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
    ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE blocked.wait_event_type = 'Lock';

-- Deadlock log দেখো (postgresql.conf)
-- log_lock_waits = on
-- deadlock_timeout = 1s
```

```python
# Django তে transaction debug
import logging
logger = logging.getLogger('django.db.backends')

# Long transaction detect করো
from django.db import connection

class TransactionTimeMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        start = time.time()
        response = self.get_response(request)
        duration = time.time() - start
        
        if duration > 1.0:  # 1 second এর বেশি
            logger.warning(f"Slow transaction: {request.path} took {duration:.2f}s")
        
        return response
```

---

## 8.15 Production Best Practices

```
1.  Transaction যতটা সম্ভব ছোট রাখো
2.  Transaction এর ভেতরে external API call করো না
3.  Transaction commit এর পর side effects (email/webhook) পাঠাও — on_commit()
4.  Deadlock prevent করতে consistent lock ordering maintain করো
5.  select_for_update() তে NOWAIT বা SKIP LOCKED ব্যবহার করো
6.  Optimistic locking high-read, low-write system এ prefer করো
7.  Payment/banking এ SERIALIZABLE isolation ব্যবহার করো
8.  Idempotency key দিয়ে duplicate payment prevent করো
9.  Retry logic implement করো deadlock এর জন্য
10. pg_stat_activity দিয়ে long-running transactions monitor করো
11. Connection pool size ঠিক করো (pgBouncer)
12. Bulk operations এ transaction wrap করো
```

---

## 8.16 Senior Engineer Insights

> **War Story — Flipkart Big Billion Day:**
> Flash sale এ 50,000 users একসাথে একটা product কিনতে চেষ্টা করে।
> Pessimistic locking এ deadlock হয়। Database connection pool শেষ।
> Fix: Redis based distributed lock + database এ F() expression update।
> Lesson: High concurrency system এ database lock এড়িয়ে application-level locking ব্যবহার করো।

> **Amazon এর Rule:**
> "Keep transactions short. Never hold a lock while doing I/O."
> Database connection একটা precious resource — যত কম সময় hold করবে তত ভালো।

> **Design Decision:**
> - Banking/Payment → SERIALIZABLE + Pessimistic lock
> - E-commerce inventory → Optimistic lock + F() expression
> - Feed/Social media → No transaction needed (eventual consistency OK)
> - Queue processing → SKIP LOCKED pattern

---

## 8.17 Interview Questions

### Beginner Level

**Q1: ACID কী?**
> Atomicity (সব বা কিছুই না), Consistency (valid state maintain), Isolation (transactions একে অপরকে দেখে না), Durability (commit হলে permanently saved)।

**Q2: Transaction কখন rollback হয়?**
> Exception/error হলে, explicitly ROLLBACK call করলে, অথবা connection crash করলে।

**Q3: Dirty read কী?**
> একটা transaction অন্য uncommitted transaction এর data পড়তে পারছে। সেই transaction rollback করলে ভুল data পড়া হয়েছে।

---

### Intermediate Level

**Q4: Deadlock কীভাবে prevent করবে?**
> সবসময় consistent order এ lock করো (ছোট id আগে)। select_for_update() এর সাথে order_by() ব্যবহার করো। Transaction ছোট রাখো। NOWAIT ব্যবহার করো যেখানে wait করা ঠিক না।

**Q5: Optimistic vs Pessimistic locking কখন কোনটা?**
> Read বেশি, write কম, conflict rare → Optimistic (version column)।
> Write contention বেশি, critical section, short transaction → Pessimistic (SELECT FOR UPDATE)।

**Q6: select_for_update(skip_locked=True) কোথায় ব্যবহার হয়?**
> Queue/job processing এ। Multiple workers একই queue থেকে jobs নিচ্ছে — locked jobs skip করে unlocked jobs নেয়। Celery, background task processing এ common।

---

### Senior / FAANG Level

**Q7: Flash sale system design করো যেখানে 100,000 concurrent users একটা product কিনতে চাইছে।**

```
Strategy (Layer by layer):

Layer 1: Rate limiting (Nginx/API Gateway)
  → প্রতি user এ request throttle করো

Layer 2: Redis distributed lock
  → product_id এর উপর Redis SETNX lock
  → lock পেলে proceed, না পেলে "sold out" দেখাও

Layer 3: Redis counter
  → DECR stock_counter atomically
  → 0 হলে sold out
  → negative হলে rollback

Layer 4: Async order creation (Celery)
  → DB write async করো
  → User কে immediately response দাও

Layer 5: Database (final source of truth)
  → F() expression দিয়ে atomic update
  → Idempotency key দিয়ে duplicate prevent

Code:
```
```python
import redis
r = redis.Redis()

def flash_purchase(user_id, product_id):
    # Step 1: Redis atomic decrement
    remaining = r.decr(f'stock:{product_id}')
    
    if remaining < 0:
        r.incr(f'stock:{product_id}')  # rollback counter
        raise SoldOutError()
    
    # Step 2: Async DB write
    create_order_async.delay(user_id, product_id)
    
    return {"status": "success", "message": "Order placed!"}

@shared_task
def create_order_async(user_id, product_id):
    with transaction.atomic():
        updated = Product.objects.filter(
            id=product_id, stock__gt=0
        ).update(stock=F('stock') - 1)
        
        if updated:
            Order.objects.create(user_id=user_id, product_id=product_id)
        else:
            r.incr(f'stock:{product_id}')  # compensate Redis counter
```

**Q8: Payment system এ double charge prevent করার complete strategy কী?**

```
1. Idempotency Key: Client প্রতিটা payment request এ unique key পাঠাবে।
   Same key দিয়ে দুইবার request আসলে same response return করো।

2. Database unique constraint: idempotency_key column এ UNIQUE index।
   get_or_create() দিয়ে race condition handle করো।

3. Payment state machine:
   pending → processing → completed/failed
   State transition atomic করো।

4. Webhook idempotency: Payment gateway webhook duplicate আসতে পারে।
   stripe_charge_id unique constraint দিয়ে handle করো।

5. Distributed lock: Payment processing শুরু করার আগে
   Redis lock নাও (order_id key তে)।
```

---

## 8.18 Hands-on Exercises

### Exercise 1
```python
# Bank transfer function লেখো যেটা:
# - Deadlock safe
# - Insufficient balance handle করে
# - Audit log রাখে
# - on_commit() দিয়ে notification পাঠায়
```

### Exercise 2
```sql
-- pg_stat_activity দিয়ে দেখাও:
-- 1. কোন queries 5 seconds এর বেশি চলছে
-- 2. কোন queries lock wait করছে
-- 3. Deadlock হয়েছে কিনা
```

### Exercise 3
```python
# Flash sale system implement করো:
# - Redis counter দিয়ে stock track করো
# - F() expression দিয়ে DB update করো
# - Idempotency key implement করো
# - Retry logic যোগ করো
```

### Mini Project
```
E-commerce Checkout System:
1. Cart → Order conversion (with stock lock)
2. Payment processing (with idempotency)
3. Order confirmation (on_commit email)
4. Inventory update (atomic)
5. Simulate concurrent purchases এবং দেখাও কোনো overselling হচ্ছে না
```

---

## 8.19 Module Summary & Cheat Sheet

```
TRANSACTION CHEAT SHEET
=========================

ISOLATION LEVELS:
  READ COMMITTED  → No dirty read (PostgreSQL default)
  REPEATABLE READ → No dirty + non-repeatable read
  SERIALIZABLE    → No dirty + non-repeatable + phantom read
  (Use SERIALIZABLE for banking/payment)

CONCURRENCY PROBLEMS:
  Dirty Read          → Uncommitted data পড়া
  Non-Repeatable Read → Same query, different result
  Phantom Read        → Same range, new rows appear

LOCKING:
  select_for_update()              → Pessimistic row lock
  select_for_update(nowait=True)   → Fail immediately if locked
  select_for_update(skip_locked=True) → Skip locked rows (queue)
  F('field') in update()           → Atomic update, no lock needed

OPTIMISTIC LOCKING:
  version column + WHERE version = old_version
  0 rows updated → conflict → retry

DEADLOCK PREVENTION:
  ✅ Consistent lock order (always lock smaller id first)
  ✅ Short transactions
  ✅ NOWAIT where appropriate
  ✅ Retry logic with exponential backoff

DJANGO PATTERNS:
  @transaction.atomic          → Function level transaction
  with transaction.atomic():   → Block level transaction
  transaction.on_commit(func)  → Run after commit (emails, tasks)
  Savepoint = nested atomic()  → Partial rollback

PRODUCTION RULES:
  ❌ No external API calls inside transaction
  ❌ No long transactions (hold connection)
  ✅ on_commit() for side effects
  ✅ Idempotency key for payments
  ✅ Retry for deadlocks
  ✅ F() for concurrent numeric updates
  ✅ SKIP LOCKED for queue processing

MONITORING:
  pg_stat_activity  → Active transactions, locks
  pg_blocking_pids  → Who is blocking whom
  log_lock_waits=on → Log slow lock waits
  deadlock_timeout  → Deadlock detection time
```
