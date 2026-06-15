# MODULE 15: Real-World Case Studies (বাস্তব জীবনের Case Studies)
## বাংলায় সম্পূর্ণ Database Design গাইড — Production Level

> **কেন এই module সবচেয়ে গুরুত্বপূর্ণ?**
> এতদিন তুমি theory শিখেছো। এখন সেটা real system এ apply করার সময়।
> Senior engineer হওয়ার মানে হলো — blank page থেকে production-ready schema design করতে পারা।
> এই module এ তুমি দেখবে Amazon, Uber, Facebook, Pathao কীভাবে database design করে।

---

## 15.1 E-Commerce System (Amazon / Daraz / Shajgoj)

### কেন E-commerce Database জটিল?

```
চ্যালেঞ্জ গুলো:
  1. Product catalog — লক্ষ লক্ষ products, হাজার attributes
  2. Inventory — real-time stock tracking
  3. Orders — complex state machine
  4. Pricing — discount, coupon, flash sale
  5. Search — fast full-text search
  6. Reviews — user generated content
  7. Multiple sellers (marketplace)
```

### Core Schema Design

```sql
-- Users
CREATE TABLE users (
    id          BIGSERIAL PRIMARY KEY,
    email       VARCHAR(255) UNIQUE NOT NULL,
    phone       VARCHAR(20) UNIQUE,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Sellers (marketplace)
CREATE TABLE sellers (
    id          BIGSERIAL PRIMARY KEY,
    user_id     BIGINT REFERENCES users(id),
    shop_name   VARCHAR(255) NOT NULL,
    rating      DECIMAL(3,2) DEFAULT 0.00,
    is_verified BOOLEAN DEFAULT FALSE
);

-- Categories (self-referential tree)
CREATE TABLE categories (
    id          BIGSERIAL PRIMARY KEY,
    parent_id   BIGINT REFERENCES categories(id),
    name        VARCHAR(100) NOT NULL,
    slug        VARCHAR(100) UNIQUE NOT NULL,
    level       INT DEFAULT 0
    -- level 0 = Electronics, level 1 = Phones, level 2 = Smartphones
);

-- Products
CREATE TABLE products (
    id          BIGSERIAL PRIMARY KEY,
    seller_id   BIGINT REFERENCES sellers(id),
    category_id BIGINT REFERENCES categories(id),
    name        VARCHAR(500) NOT NULL,
    description TEXT,
    base_price  DECIMAL(12,2) NOT NULL,
    status      VARCHAR(20) DEFAULT 'active', -- active, inactive, deleted
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Product Variants (size, color combinations)
CREATE TABLE product_variants (
    id          BIGSERIAL PRIMARY KEY,
    product_id  BIGINT REFERENCES products(id),
    sku         VARCHAR(100) UNIQUE NOT NULL,
    price       DECIMAL(12,2) NOT NULL,
    attributes  JSONB,  -- {"color": "red", "size": "XL"}
    stock_qty   INT DEFAULT 0,
    CONSTRAINT positive_stock CHECK (stock_qty >= 0)
);

-- Inventory (warehouse level tracking)
CREATE TABLE inventory (
    id          BIGSERIAL PRIMARY KEY,
    variant_id  BIGINT REFERENCES product_variants(id),
    warehouse   VARCHAR(100) NOT NULL,
    qty         INT NOT NULL DEFAULT 0,
    reserved    INT NOT NULL DEFAULT 0,  -- qty reserved in pending orders
    CONSTRAINT chk_qty CHECK (qty >= 0 AND reserved >= 0 AND qty >= reserved)
);

-- Addresses
CREATE TABLE addresses (
    id          BIGSERIAL PRIMARY KEY,
    user_id     BIGINT REFERENCES users(id),
    label       VARCHAR(50),  -- Home, Office
    street      TEXT NOT NULL,
    city        VARCHAR(100),
    country     VARCHAR(100) DEFAULT 'Bangladesh',
    is_default  BOOLEAN DEFAULT FALSE
);

-- Orders
CREATE TABLE orders (
    id              BIGSERIAL PRIMARY KEY,
    user_id         BIGINT REFERENCES users(id),
    address_id      BIGINT REFERENCES addresses(id),
    status          VARCHAR(30) DEFAULT 'pending',
    -- pending → confirmed → shipped → delivered → completed
    -- pending → cancelled
    subtotal        DECIMAL(12,2) NOT NULL,
    discount_amount DECIMAL(12,2) DEFAULT 0,
    shipping_fee    DECIMAL(12,2) DEFAULT 0,
    total_amount    DECIMAL(12,2) NOT NULL,
    coupon_code     VARCHAR(50),
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Order Items
CREATE TABLE order_items (
    id          BIGSERIAL PRIMARY KEY,
    order_id    BIGINT REFERENCES orders(id),
    variant_id  BIGINT REFERENCES product_variants(id),
    seller_id   BIGINT REFERENCES sellers(id),
    qty         INT NOT NULL,
    unit_price  DECIMAL(12,2) NOT NULL,  -- price at time of order (snapshot!)
    total_price DECIMAL(12,2) NOT NULL
);

-- Payments
CREATE TABLE payments (
    id              BIGSERIAL PRIMARY KEY,
    order_id        BIGINT REFERENCES orders(id),
    method          VARCHAR(50),  -- bkash, card, cod
    status          VARCHAR(30) DEFAULT 'pending',
    amount          DECIMAL(12,2) NOT NULL,
    transaction_id  VARCHAR(255) UNIQUE,
    paid_at         TIMESTAMPTZ
);

-- Coupons
CREATE TABLE coupons (
    id              BIGSERIAL PRIMARY KEY,
    code            VARCHAR(50) UNIQUE NOT NULL,
    type            VARCHAR(20),  -- percentage, fixed
    value           DECIMAL(10,2) NOT NULL,
    min_order_amount DECIMAL(12,2) DEFAULT 0,
    max_uses        INT,
    used_count      INT DEFAULT 0,
    expires_at      TIMESTAMPTZ
);

-- Reviews
CREATE TABLE reviews (
    id          BIGSERIAL PRIMARY KEY,
    product_id  BIGINT REFERENCES products(id),
    user_id     BIGINT REFERENCES users(id),
    order_id    BIGINT REFERENCES orders(id),  -- verified purchase only!
    rating      SMALLINT CHECK (rating BETWEEN 1 AND 5),
    comment     TEXT,
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE (product_id, user_id, order_id)  -- একটা order এ একবারই review
);
```

### Critical Design Decisions

```sql
-- ১. unit_price snapshot — product price পরে change হলেও order history সঠিক থাকবে
-- order_items.unit_price = price AT THE TIME of purchase
-- product_variants.price = CURRENT price

-- ২. Inventory reservation — race condition avoid করতে
-- যখন order place হয়: reserved += qty
-- যখন order confirm হয়: qty -= reserved, reserved -= qty
-- যখন order cancel হয়: reserved -= qty

-- Inventory deduction with lock
UPDATE inventory
SET reserved = reserved + 2
WHERE variant_id = 101
  AND (qty - reserved) >= 2;  -- available stock check
-- rowcount = 0 মানে out of stock

-- ৩. Category tree query (recursive CTE)
WITH RECURSIVE category_tree AS (
    SELECT id, name, parent_id, 0 AS depth
    FROM categories WHERE id = 1  -- Electronics

    UNION ALL

    SELECT c.id, c.name, c.parent_id, ct.depth + 1
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT * FROM category_tree ORDER BY depth;
```

### Django ORM

```python
from django.db import models, transaction, F

class ProductVariant(models.Model):
    product    = models.ForeignKey('Product', on_delete=models.CASCADE)
    sku        = models.CharField(max_length=100, unique=True)
    price      = models.DecimalField(max_digits=12, decimal_places=2)
    attributes = models.JSONField(default=dict)
    stock_qty  = models.PositiveIntegerField(default=0)

class Inventory(models.Model):
    variant   = models.ForeignKey(ProductVariant, on_delete=models.CASCADE)
    warehouse = models.CharField(max_length=100)
    qty       = models.IntegerField(default=0)
    reserved  = models.IntegerField(default=0)

    def available(self):
        return self.qty - self.reserved

# Atomic order placement
@transaction.atomic
def place_order(user, cart_items, address_id):
    order = Order.objects.create(
        user=user,
        address_id=address_id,
        subtotal=sum(i['price'] * i['qty'] for i in cart_items),
        total_amount=0,
    )

    for item in cart_items:
        # Reserve inventory with select_for_update
        inv = Inventory.objects.select_for_update().get(
            variant_id=item['variant_id']
        )
        if inv.available() < item['qty']:
            raise ValueError(f"Insufficient stock for variant {item['variant_id']}")

        inv.reserved = F('reserved') + item['qty']
        inv.save(update_fields=['reserved'])

        OrderItem.objects.create(
            order=order,
            variant_id=item['variant_id'],
            qty=item['qty'],
            unit_price=item['price'],  # snapshot!
            total_price=item['price'] * item['qty'],
        )

    order.total_amount = order.subtotal
    order.save()
    return order
```

### Indexes

```sql
-- Product search
CREATE INDEX idx_products_category   ON products(category_id);
CREATE INDEX idx_products_seller     ON products(seller_id);
CREATE INDEX idx_products_status     ON products(status);

-- Variant lookup
CREATE INDEX idx_variants_product    ON product_variants(product_id);
CREATE UNIQUE INDEX idx_variants_sku ON product_variants(sku);

-- Order queries
CREATE INDEX idx_orders_user         ON orders(user_id);
CREATE INDEX idx_orders_status       ON orders(status);
CREATE INDEX idx_order_items_order   ON order_items(order_id);
CREATE INDEX idx_order_items_variant ON order_items(variant_id);

-- Full-text search on products
CREATE INDEX idx_products_fts ON products
    USING gin(to_tsvector('english', name || ' ' || COALESCE(description, '')));

-- Search query:
SELECT * FROM products
WHERE to_tsvector('english', name || ' ' || COALESCE(description,''))
    @@ plainto_tsquery('english', 'samsung phone')
ORDER BY ts_rank(
    to_tsvector('english', name), plainto_tsquery('english', 'samsung phone')
) DESC;
```

---

## 15.2 Banking System (DBBL / bKash / Nagad)

### কেন Banking Database সবচেয়ে Critical?

```
Requirements:
  ✅ Zero data loss — RPO = 0
  ✅ ACID guarantees — money must not disappear
  ✅ Audit trail — every transaction logged
  ✅ Concurrency — millions of transactions
  ✅ Regulatory compliance — Bangladesh Bank rules
  ✅ Fraud detection — real-time checks
```

### Core Schema

```sql
-- Customers
CREATE TABLE customers (
    id              BIGSERIAL PRIMARY KEY,
    national_id     VARCHAR(20) UNIQUE NOT NULL,  -- NID
    full_name       VARCHAR(200) NOT NULL,
    date_of_birth   DATE NOT NULL,
    kyc_status      VARCHAR(20) DEFAULT 'pending',  -- pending, verified, rejected
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Accounts
CREATE TABLE accounts (
    id              BIGSERIAL PRIMARY KEY,
    customer_id     BIGINT REFERENCES customers(id),
    account_number  VARCHAR(20) UNIQUE NOT NULL,
    account_type    VARCHAR(20) NOT NULL,  -- savings, current, loan
    currency        CHAR(3) DEFAULT 'BDT',
    balance         DECIMAL(18,2) NOT NULL DEFAULT 0.00,
    status          VARCHAR(20) DEFAULT 'active',  -- active, frozen, closed
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    CONSTRAINT positive_balance CHECK (balance >= 0)
);

-- Transactions (immutable ledger — never UPDATE or DELETE!)
CREATE TABLE transactions (
    id              BIGSERIAL PRIMARY KEY,
    txn_ref         VARCHAR(50) UNIQUE NOT NULL,  -- unique reference
    account_id      BIGINT REFERENCES accounts(id),
    type            VARCHAR(20) NOT NULL,  -- credit, debit
    amount          DECIMAL(18,2) NOT NULL CHECK (amount > 0),
    balance_before  DECIMAL(18,2) NOT NULL,  -- snapshot
    balance_after   DECIMAL(18,2) NOT NULL,  -- snapshot
    description     TEXT,
    channel         VARCHAR(30),  -- atm, mobile, branch, online
    ip_address      INET,
    status          VARCHAR(20) DEFAULT 'completed',
    created_at      TIMESTAMPTZ DEFAULT NOW()
);
-- CRITICAL: transactions table এ কখনো UPDATE/DELETE করো না!
-- Correction করতে হলে reversal transaction তৈরি করো।

-- Transfers
CREATE TABLE transfers (
    id              BIGSERIAL PRIMARY KEY,
    from_account_id BIGINT REFERENCES accounts(id),
    to_account_id   BIGINT REFERENCES accounts(id),
    amount          DECIMAL(18,2) NOT NULL CHECK (amount > 0),
    debit_txn_id    BIGINT REFERENCES transactions(id),
    credit_txn_id   BIGINT REFERENCES transactions(id),
    status          VARCHAR(20) DEFAULT 'pending',
    initiated_at    TIMESTAMPTZ DEFAULT NOW(),
    completed_at    TIMESTAMPTZ
);

-- Audit Log (every sensitive action)
CREATE TABLE audit_log (
    id          BIGSERIAL PRIMARY KEY,
    table_name  VARCHAR(100) NOT NULL,
    record_id   BIGINT NOT NULL,
    action      VARCHAR(10) NOT NULL,  -- INSERT, UPDATE, DELETE
    old_data    JSONB,
    new_data    JSONB,
    changed_by  VARCHAR(100),
    changed_at  TIMESTAMPTZ DEFAULT NOW()
);
```

### Transfer — ACID Transaction

```sql
-- Money Transfer: double-entry bookkeeping
BEGIN;

-- Step 1: Lock both accounts (always lock in consistent order to avoid deadlock)
SELECT id, balance FROM accounts
WHERE id IN (101, 202)
ORDER BY id  -- consistent order!
FOR UPDATE;

-- Step 2: Check balance
-- (application layer checks result)

-- Step 3: Debit sender
UPDATE accounts
SET balance = balance - 5000
WHERE id = 101 AND balance >= 5000;
-- rowcount = 0 মানে insufficient funds

-- Step 4: Credit receiver
UPDATE accounts
SET balance = balance + 5000
WHERE id = 202;

-- Step 5: Log transactions
INSERT INTO transactions (txn_ref, account_id, type, amount, balance_before, balance_after, description)
VALUES ('TXN-001', 101, 'debit',  5000, 50000, 45000, 'Transfer to 202');

INSERT INTO transactions (txn_ref, account_id, type, amount, balance_before, balance_after, description)
VALUES ('TXN-002', 202, 'credit', 5000, 10000, 15000, 'Transfer from 101');

-- Step 6: Record transfer
INSERT INTO transfers (from_account_id, to_account_id, amount, debit_txn_id, credit_txn_id, status)
VALUES (101, 202, 5000, lastval()-1, lastval(), 'completed');

COMMIT;
```

### Django ORM — Transfer

```python
from django.db import transaction, models
from django.db.models import F
import uuid

@transaction.atomic
def transfer_money(from_account_id, to_account_id, amount):
    # Lock in consistent order (prevent deadlock)
    ids = sorted([from_account_id, to_account_id])
    accounts = {
        a.id: a
        for a in Account.objects.select_for_update().filter(id__in=ids).order_by('id')
    }

    sender   = accounts[from_account_id]
    receiver = accounts[to_account_id]

    if sender.balance < amount:
        raise ValueError("Insufficient funds")

    if sender.status != 'active' or receiver.status != 'active':
        raise ValueError("Account not active")

    txn_ref = str(uuid.uuid4())

    # Debit
    debit_txn = Transaction.objects.create(
        txn_ref        = f'D-{txn_ref}',
        account        = sender,
        type           = 'debit',
        amount         = amount,
        balance_before = sender.balance,
        balance_after  = sender.balance - amount,
    )
    Account.objects.filter(id=from_account_id).update(balance=F('balance') - amount)

    # Credit
    credit_txn = Transaction.objects.create(
        txn_ref        = f'C-{txn_ref}',
        account        = receiver,
        type           = 'credit',
        amount         = amount,
        balance_before = receiver.balance,
        balance_after  = receiver.balance + amount,
    )
    Account.objects.filter(id=to_account_id).update(balance=F('balance') + amount)

    Transfer.objects.create(
        from_account = sender,
        to_account   = receiver,
        amount       = amount,
        debit_txn    = debit_txn,
        credit_txn   = credit_txn,
        status       = 'completed',
    )
```

### Account Statement Query

```sql
-- Last 30 days statement
SELECT
    t.created_at,
    t.type,
    t.amount,
    t.balance_after,
    t.description,
    t.channel
FROM transactions t
WHERE t.account_id = 101
  AND t.created_at >= NOW() - INTERVAL '30 days'
ORDER BY t.created_at DESC;

-- Running balance verification (should match account.balance)
SELECT
    SUM(CASE WHEN type = 'credit' THEN amount ELSE 0 END) -
    SUM(CASE WHEN type = 'debit'  THEN amount ELSE 0 END) AS calculated_balance
FROM transactions
WHERE account_id = 101;
```

---

## 15.3 Food Delivery System (Foodpanda / Shohoz Food)

### Schema Design

```sql
-- Restaurants
CREATE TABLE restaurants (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(200) NOT NULL,
    address     TEXT,
    lat         DECIMAL(9,6),
    lng         DECIMAL(9,6),
    cuisine     VARCHAR(100)[],  -- array: ['Bengali', 'Chinese']
    rating      DECIMAL(3,2) DEFAULT 0.00,
    is_open     BOOLEAN DEFAULT TRUE,
    opens_at    TIME,
    closes_at   TIME
);

-- Menu Categories
CREATE TABLE menu_categories (
    id              BIGSERIAL PRIMARY KEY,
    restaurant_id   BIGINT REFERENCES restaurants(id),
    name            VARCHAR(100) NOT NULL,
    sort_order      INT DEFAULT 0
);

-- Menu Items
CREATE TABLE menu_items (
    id              BIGSERIAL PRIMARY KEY,
    restaurant_id   BIGINT REFERENCES restaurants(id),
    category_id     BIGINT REFERENCES menu_categories(id),
    name            VARCHAR(200) NOT NULL,
    description     TEXT,
    price           DECIMAL(10,2) NOT NULL,
    is_available    BOOLEAN DEFAULT TRUE,
    prep_time_mins  INT DEFAULT 15
);

-- Delivery Zones
CREATE TABLE delivery_zones (
    id              BIGSERIAL PRIMARY KEY,
    restaurant_id   BIGINT REFERENCES restaurants(id),
    zone_name       VARCHAR(100),
    delivery_fee    DECIMAL(8,2) DEFAULT 0,
    min_order       DECIMAL(10,2) DEFAULT 0,
    est_minutes     INT DEFAULT 30
);

-- Riders
CREATE TABLE riders (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(200) NOT NULL,
    phone       VARCHAR(20) UNIQUE NOT NULL,
    status      VARCHAR(20) DEFAULT 'offline',  -- offline, available, busy
    lat         DECIMAL(9,6),
    lng         DECIMAL(9,6),
    rating      DECIMAL(3,2) DEFAULT 5.00,
    updated_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Orders
CREATE TABLE food_orders (
    id              BIGSERIAL PRIMARY KEY,
    user_id         BIGINT REFERENCES users(id),
    restaurant_id   BIGINT REFERENCES restaurants(id),
    rider_id        BIGINT REFERENCES riders(id),
    status          VARCHAR(30) DEFAULT 'placed',
    -- placed → confirmed → preparing → ready → picked_up → delivered
    -- placed → cancelled
    delivery_address TEXT NOT NULL,
    delivery_lat    DECIMAL(9,6),
    delivery_lng    DECIMAL(9,6),
    subtotal        DECIMAL(10,2) NOT NULL,
    delivery_fee    DECIMAL(8,2) DEFAULT 0,
    total           DECIMAL(10,2) NOT NULL,
    placed_at       TIMESTAMPTZ DEFAULT NOW(),
    confirmed_at    TIMESTAMPTZ,
    delivered_at    TIMESTAMPTZ,
    estimated_at    TIMESTAMPTZ  -- estimated delivery time
);

-- Order Items
CREATE TABLE food_order_items (
    id          BIGSERIAL PRIMARY KEY,
    order_id    BIGINT REFERENCES food_orders(id),
    item_id     BIGINT REFERENCES menu_items(id),
    qty         INT NOT NULL,
    unit_price  DECIMAL(10,2) NOT NULL,  -- snapshot
    customization JSONB  -- {"extra_spicy": true, "no_onion": true}
);

-- Order Status History (event log)
CREATE TABLE order_status_history (
    id          BIGSERIAL PRIMARY KEY,
    order_id    BIGINT REFERENCES food_orders(id),
    status      VARCHAR(30) NOT NULL,
    note        TEXT,
    changed_at  TIMESTAMPTZ DEFAULT NOW()
);
```

### Nearby Restaurant Query

```sql
-- Nearby open restaurants (within 5km using Haversine)
SELECT
    r.id,
    r.name,
    r.rating,
    ROUND(
        6371 * acos(
            cos(radians(23.8103)) * cos(radians(r.lat)) *
            cos(radians(r.lng) - radians(90.4125)) +
            sin(radians(23.8103)) * sin(radians(r.lat))
        )::numeric, 2
    ) AS distance_km
FROM restaurants r
WHERE r.is_open = TRUE
HAVING distance_km <= 5
ORDER BY distance_km ASC
LIMIT 20;

-- Production note: PostGIS extension ব্যবহার করো real geo queries এ
-- CREATE EXTENSION postgis;
-- SELECT * FROM restaurants ORDER BY location <-> ST_Point(90.4125, 23.8103) LIMIT 20;
```

### Django ORM — Order Flow

```python
@transaction.atomic
def place_food_order(user_id, restaurant_id, items, delivery_address):
    restaurant = Restaurant.objects.get(id=restaurant_id, is_open=True)

    order = FoodOrder.objects.create(
        user_id          = user_id,
        restaurant       = restaurant,
        status           = 'placed',
        delivery_address = delivery_address,
        subtotal         = sum(i['price'] * i['qty'] for i in items),
        total            = 0,
    )
    order.total = order.subtotal + order.delivery_fee
    order.save()

    FoodOrderItem.objects.bulk_create([
        FoodOrderItem(
            order      = order,
            item_id    = i['item_id'],
            qty        = i['qty'],
            unit_price = i['price'],
        ) for i in items
    ])

    OrderStatusHistory.objects.create(order=order, status='placed')
    return order
```

---

## 15.4 Ride Sharing System (Pathao / Uber / Shohoz)

### Schema Design

```sql
-- Drivers
CREATE TABLE drivers (
    id              BIGSERIAL PRIMARY KEY,
    name            VARCHAR(200) NOT NULL,
    phone           VARCHAR(20) UNIQUE NOT NULL,
    license_number  VARCHAR(50) UNIQUE NOT NULL,
    vehicle_type    VARCHAR(30),  -- bike, car, cng
    vehicle_number  VARCHAR(30),
    rating          DECIMAL(3,2) DEFAULT 5.00,
    status          VARCHAR(20) DEFAULT 'offline',  -- offline, available, on_trip
    current_lat     DECIMAL(9,6),
    current_lng     DECIMAL(9,6),
    location_updated_at TIMESTAMPTZ
);

-- Rides
CREATE TABLE rides (
    id              BIGSERIAL PRIMARY KEY,
    rider_id        BIGINT REFERENCES users(id),
    driver_id       BIGINT REFERENCES drivers(id),
    status          VARCHAR(20) DEFAULT 'requested',
    -- requested → driver_assigned → driver_arrived → started → completed
    -- requested → cancelled
    pickup_lat      DECIMAL(9,6) NOT NULL,
    pickup_lng      DECIMAL(9,6) NOT NULL,
    pickup_address  TEXT,
    drop_lat        DECIMAL(9,6) NOT NULL,
    drop_lng        DECIMAL(9,6) NOT NULL,
    drop_address    TEXT,
    distance_km     DECIMAL(8,2),
    duration_mins   INT,
    fare            DECIMAL(10,2),
    vehicle_type    VARCHAR(30),
    requested_at    TIMESTAMPTZ DEFAULT NOW(),
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ
);

-- Ride Events (GPS tracking log)
CREATE TABLE ride_events (
    id          BIGSERIAL PRIMARY KEY,
    ride_id     BIGINT REFERENCES rides(id),
    driver_id   BIGINT REFERENCES drivers(id),
    lat         DECIMAL(9,6),
    lng         DECIMAL(9,6),
    speed_kmh   DECIMAL(5,1),
    recorded_at TIMESTAMPTZ DEFAULT NOW()
) PARTITION BY RANGE (recorded_at);
-- GPS data অনেক বেশি — partition করা জরুরি

CREATE TABLE ride_events_2024_06 PARTITION OF ride_events
    FOR VALUES FROM ('2024-06-01') TO ('2024-07-01');

-- Surge Pricing Zones
CREATE TABLE surge_zones (
    id          BIGSERIAL PRIMARY KEY,
    zone_name   VARCHAR(100),
    multiplier  DECIMAL(4,2) DEFAULT 1.00,  -- 1.5x, 2x
    active_from TIMESTAMPTZ,
    active_to   TIMESTAMPTZ
);

-- Ratings
CREATE TABLE ride_ratings (
    id          BIGSERIAL PRIMARY KEY,
    ride_id     BIGINT REFERENCES rides(id) UNIQUE,
    rider_rating    SMALLINT CHECK (rider_rating BETWEEN 1 AND 5),
    driver_rating   SMALLINT CHECK (driver_rating BETWEEN 1 AND 5),
    rider_comment   TEXT,
    driver_comment  TEXT,
    rated_at    TIMESTAMPTZ DEFAULT NOW()
);
```

### Fare Calculation

```sql
-- Fare calculation query
WITH trip_data AS (
    SELECT
        r.id,
        r.distance_km,
        r.vehicle_type,
        r.requested_at,
        COALESCE(sz.multiplier, 1.0) AS surge_multiplier
    FROM rides r
    LEFT JOIN surge_zones sz ON
        r.pickup_lat BETWEEN -90 AND 90  -- simplified zone check
        AND sz.active_from <= r.requested_at
        AND sz.active_to   >= r.requested_at
    WHERE r.id = 1001
)
SELECT
    id,
    distance_km,
    CASE vehicle_type
        WHEN 'bike' THEN (30 + distance_km * 12) * surge_multiplier
        WHEN 'car'  THEN (50 + distance_km * 20) * surge_multiplier
        WHEN 'cng'  THEN (40 + distance_km * 15) * surge_multiplier
    END AS calculated_fare
FROM trip_data;
```

### Driver Location Update (High Frequency)

```python
# Driver location — Redis এ store করো (high frequency updates)
import redis
import json

r = redis.Redis(host='localhost', port=6379)

def update_driver_location(driver_id, lat, lng):
    # Redis এ real-time location
    r.hset(f'driver:{driver_id}', mapping={
        'lat': lat, 'lng': lng,
        'updated_at': datetime.now().isoformat()
    })
    r.expire(f'driver:{driver_id}', 300)  # 5 min TTL

    # PostgreSQL এ periodic write (every 30 seconds)
    # Background task দিয়ে batch update করো
    r.lpush('location_updates', json.dumps({
        'driver_id': driver_id, 'lat': lat, 'lng': lng
    }))

# Celery task: batch flush to PostgreSQL every 30s
@shared_task
def flush_location_updates():
    updates = []
    while True:
        data = r.rpop('location_updates')
        if not data: break
        updates.append(json.loads(data))

    if updates:
        Driver.objects.bulk_update_or_create(updates, ...)
```

---

## 15.5 Social Media System (Facebook / Instagram)

### Schema Design

```sql
-- Users
CREATE TABLE social_users (
    id          BIGSERIAL PRIMARY KEY,
    username    VARCHAR(50) UNIQUE NOT NULL,
    display_name VARCHAR(200),
    bio         TEXT,
    avatar_url  TEXT,
    follower_count  INT DEFAULT 0,  -- denormalized for performance!
    following_count INT DEFAULT 0,
    post_count      INT DEFAULT 0,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Follows (friendship graph)
CREATE TABLE follows (
    follower_id BIGINT REFERENCES social_users(id),
    followee_id BIGINT REFERENCES social_users(id),
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (follower_id, followee_id),
    CONSTRAINT no_self_follow CHECK (follower_id != followee_id)
);

-- Posts
CREATE TABLE posts (
    id          BIGSERIAL PRIMARY KEY,
    user_id     BIGINT REFERENCES social_users(id),
    content     TEXT,
    media_urls  TEXT[],  -- array of S3 URLs
    like_count  INT DEFAULT 0,  -- denormalized!
    comment_count INT DEFAULT 0,
    share_count INT DEFAULT 0,
    visibility  VARCHAR(20) DEFAULT 'public',  -- public, followers, private
    created_at  TIMESTAMPTZ DEFAULT NOW()
) PARTITION BY RANGE (created_at);
-- Posts partition by month — old posts কম access হয়

CREATE TABLE posts_2024_06 PARTITION OF posts
    FOR VALUES FROM ('2024-06-01') TO ('2024-07-01');

-- Likes
CREATE TABLE likes (
    user_id     BIGINT REFERENCES social_users(id),
    post_id     BIGINT NOT NULL,
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (user_id, post_id)
);

-- Comments
CREATE TABLE comments (
    id          BIGSERIAL PRIMARY KEY,
    post_id     BIGINT NOT NULL,
    user_id     BIGINT REFERENCES social_users(id),
    parent_id   BIGINT REFERENCES comments(id),  -- nested comments
    content     TEXT NOT NULL,
    like_count  INT DEFAULT 0,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Notifications
CREATE TABLE notifications (
    id          BIGSERIAL PRIMARY KEY,
    user_id     BIGINT REFERENCES social_users(id),  -- recipient
    actor_id    BIGINT REFERENCES social_users(id),  -- who caused it
    type        VARCHAR(30),  -- like, comment, follow, mention
    entity_type VARCHAR(30),  -- post, comment
    entity_id   BIGINT,
    is_read     BOOLEAN DEFAULT FALSE,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_notif_user_unread ON notifications(user_id, is_read, created_at DESC);

-- News Feed (fan-out on write — pre-computed)
CREATE TABLE feed_items (
    user_id     BIGINT REFERENCES social_users(id),
    post_id     BIGINT NOT NULL,
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (user_id, post_id)
);
-- Instagram/Twitter approach: যখন কেউ post করে,
-- তার সব followers এর feed_items এ insert করো
```

### News Feed — Fan-out on Write

```python
# Post create করার সাথে সাথে followers এর feed এ push করো
from celery import shared_task

@shared_task
def fanout_post_to_followers(post_id, author_id):
    # Author এর সব followers fetch করো
    follower_ids = list(
        Follow.objects.filter(followee_id=author_id)
        .values_list('follower_id', flat=True)
    )

    # Celebrity problem: 10M followers? Redis তে store করো!
    if len(follower_ids) > 10_000:
        # Fan-out on read (pull model) for celebrities
        redis_client.sadd(f'celeb_posts:{author_id}', post_id)
        return

    # Normal users: fan-out on write
    FeedItem.objects.bulk_create([
        FeedItem(user_id=uid, post_id=post_id)
        for uid in follower_ids
    ], ignore_conflicts=True)

# Feed read
def get_user_feed(user_id, page=1, per_page=20):
    offset = (page - 1) * per_page
    return (
        Post.objects
        .filter(feeditem__user_id=user_id)
        .select_related('user')
        .order_by('-created_at')
        [offset:offset + per_page]
    )
```

### Counter Denormalization

```sql
-- Like করলে counter update করো (denormalized)
-- ❌ Slow: SELECT COUNT(*) FROM likes WHERE post_id = 101
-- ✅ Fast: SELECT like_count FROM posts WHERE id = 101

-- Like এ counter increment
UPDATE posts SET like_count = like_count + 1 WHERE id = 101;
INSERT INTO likes (user_id, post_id) VALUES (55, 101);

-- Unlike এ decrement
UPDATE posts SET like_count = GREATEST(like_count - 1, 0) WHERE id = 101;
DELETE FROM likes WHERE user_id = 55 AND post_id = 101;
```

---

## 15.6 Chat Application (WhatsApp / Messenger)

### Schema Design

```sql
-- Conversations
CREATE TABLE conversations (
    id          BIGSERIAL PRIMARY KEY,
    type        VARCHAR(20) DEFAULT 'direct',  -- direct, group
    name        VARCHAR(200),  -- group name
    created_by  BIGINT REFERENCES social_users(id),
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Conversation Members
CREATE TABLE conversation_members (
    conversation_id BIGINT REFERENCES conversations(id),
    user_id         BIGINT REFERENCES social_users(id),
    role            VARCHAR(20) DEFAULT 'member',  -- admin, member
    joined_at       TIMESTAMPTZ DEFAULT NOW(),
    last_read_at    TIMESTAMPTZ DEFAULT NOW(),  -- unread count এর জন্য
    PRIMARY KEY (conversation_id, user_id)
);

-- Messages
CREATE TABLE messages (
    id              BIGSERIAL PRIMARY KEY,
    conversation_id BIGINT REFERENCES conversations(id),
    sender_id       BIGINT REFERENCES social_users(id),
    type            VARCHAR(20) DEFAULT 'text',  -- text, image, video, audio
    content         TEXT,
    media_url       TEXT,
    reply_to_id     BIGINT REFERENCES messages(id),
    is_deleted      BOOLEAN DEFAULT FALSE,
    sent_at         TIMESTAMPTZ DEFAULT NOW()
) PARTITION BY RANGE (sent_at);
-- Messages partition by month — very high volume

-- Message Status (delivery receipts)
CREATE TABLE message_status (
    message_id  BIGINT REFERENCES messages(id),
    user_id     BIGINT REFERENCES social_users(id),
    status      VARCHAR(20) DEFAULT 'sent',  -- sent, delivered, read
    updated_at  TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (message_id, user_id)
);
```

### Unread Count Query

```sql
-- Unread messages per conversation for a user
SELECT
    cm.conversation_id,
    COUNT(m.id) AS unread_count
FROM conversation_members cm
JOIN messages m ON
    m.conversation_id = cm.conversation_id
    AND m.sent_at > cm.last_read_at
    AND m.sender_id != cm.user_id
    AND m.is_deleted = FALSE
WHERE cm.user_id = 101
GROUP BY cm.conversation_id;

-- Mark messages as read
UPDATE conversation_members
SET last_read_at = NOW()
WHERE conversation_id = 55 AND user_id = 101;
```

---

## 15.7 Hospital Management System

### Schema Design

```sql
-- Patients
CREATE TABLE patients (
    id              BIGSERIAL PRIMARY KEY,
    patient_no      VARCHAR(20) UNIQUE NOT NULL,
    full_name       VARCHAR(200) NOT NULL,
    date_of_birth   DATE NOT NULL,
    gender          CHAR(1) CHECK (gender IN ('M', 'F', 'O')),
    blood_group     VARCHAR(5),
    phone           VARCHAR(20),
    emergency_contact_name  VARCHAR(200),
    emergency_contact_phone VARCHAR(20),
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Doctors
CREATE TABLE doctors (
    id              BIGSERIAL PRIMARY KEY,
    name            VARCHAR(200) NOT NULL,
    specialization  VARCHAR(100),
    bmdc_number     VARCHAR(50) UNIQUE NOT NULL,  -- Bangladesh Medical & Dental Council
    department_id   BIGINT REFERENCES departments(id),
    consultation_fee DECIMAL(10,2)
);

-- Departments
CREATE TABLE departments (
    id      BIGSERIAL PRIMARY KEY,
    name    VARCHAR(100) NOT NULL,  -- Cardiology, Orthopedics
    floor   INT
);

-- Appointments
CREATE TABLE appointments (
    id          BIGSERIAL PRIMARY KEY,
    patient_id  BIGINT REFERENCES patients(id),
    doctor_id   BIGINT REFERENCES doctors(id),
    scheduled_at TIMESTAMPTZ NOT NULL,
    status      VARCHAR(20) DEFAULT 'scheduled',  -- scheduled, completed, cancelled
    type        VARCHAR(20) DEFAULT 'consultation',
    notes       TEXT,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Medical Records
CREATE TABLE medical_records (
    id              BIGSERIAL PRIMARY KEY,
    patient_id      BIGINT REFERENCES patients(id),
    doctor_id       BIGINT REFERENCES doctors(id),
    appointment_id  BIGINT REFERENCES appointments(id),
    chief_complaint TEXT,
    diagnosis       TEXT,
    treatment_plan  TEXT,
    vitals          JSONB,  -- {"bp": "120/80", "temp": 98.6, "weight": 70}
    recorded_at     TIMESTAMPTZ DEFAULT NOW()
);

-- Prescriptions
CREATE TABLE prescriptions (
    id              BIGSERIAL PRIMARY KEY,
    record_id       BIGINT REFERENCES medical_records(id),
    patient_id      BIGINT REFERENCES patients(id),
    doctor_id       BIGINT REFERENCES doctors(id),
    issued_at       TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE prescription_items (
    id              BIGSERIAL PRIMARY KEY,
    prescription_id BIGINT REFERENCES prescriptions(id),
    medicine_name   VARCHAR(200) NOT NULL,
    dosage          VARCHAR(100),
    frequency       VARCHAR(100),  -- "1+0+1", "twice daily"
    duration_days   INT,
    instructions    TEXT
);

-- Lab Tests
CREATE TABLE lab_tests (
    id              BIGSERIAL PRIMARY KEY,
    patient_id      BIGINT REFERENCES patients(id),
    doctor_id       BIGINT REFERENCES doctors(id),
    test_name       VARCHAR(200) NOT NULL,
    status          VARCHAR(20) DEFAULT 'ordered',  -- ordered, sample_collected, resulted
    ordered_at      TIMESTAMPTZ DEFAULT NOW(),
    resulted_at     TIMESTAMPTZ,
    result          JSONB,  -- flexible result format per test type
    reference_range TEXT
);

-- Bills
CREATE TABLE bills (
    id          BIGSERIAL PRIMARY KEY,
    patient_id  BIGINT REFERENCES patients(id),
    total       DECIMAL(12,2) NOT NULL,
    paid        DECIMAL(12,2) DEFAULT 0,
    status      VARCHAR(20) DEFAULT 'unpaid',
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE bill_items (
    id          BIGSERIAL PRIMARY KEY,
    bill_id     BIGINT REFERENCES bills(id),
    description VARCHAR(200),
    amount      DECIMAL(10,2),
    item_type   VARCHAR(50)  -- consultation, lab_test, medicine, procedure
);
```

---

## 15.8 Inventory Management System

### Schema Design

```sql
-- Warehouses
CREATE TABLE warehouses (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(200) NOT NULL,
    address     TEXT,
    capacity    INT
);

-- Products (simplified)
CREATE TABLE inv_products (
    id          BIGSERIAL PRIMARY KEY,
    sku         VARCHAR(100) UNIQUE NOT NULL,
    name        VARCHAR(300) NOT NULL,
    unit        VARCHAR(20),  -- pcs, kg, liter
    reorder_level INT DEFAULT 10,  -- alert when stock < this
    reorder_qty   INT DEFAULT 100
);

-- Stock Levels
CREATE TABLE stock (
    id              BIGSERIAL PRIMARY KEY,
    product_id      BIGINT REFERENCES inv_products(id),
    warehouse_id    BIGINT REFERENCES warehouses(id),
    qty_on_hand     INT DEFAULT 0,
    qty_reserved    INT DEFAULT 0,
    qty_on_order    INT DEFAULT 0,  -- PO এ আছে but not received yet
    UNIQUE (product_id, warehouse_id)
);

-- Stock Movements (immutable ledger — like banking transactions)
CREATE TABLE stock_movements (
    id              BIGSERIAL PRIMARY KEY,
    product_id      BIGINT REFERENCES inv_products(id),
    warehouse_id    BIGINT REFERENCES warehouses(id),
    type            VARCHAR(30) NOT NULL,
    -- purchase_receipt, sale, transfer_in, transfer_out, adjustment, return
    qty             INT NOT NULL,  -- positive = in, negative = out
    reference_no    VARCHAR(100),
    note            TEXT,
    moved_by        BIGINT REFERENCES users(id),
    moved_at        TIMESTAMPTZ DEFAULT NOW()
);

-- Purchase Orders
CREATE TABLE purchase_orders (
    id              BIGSERIAL PRIMARY KEY,
    supplier_id     BIGINT REFERENCES suppliers(id),
    status          VARCHAR(20) DEFAULT 'draft',
    -- draft → sent → partially_received → received → cancelled
    ordered_at      TIMESTAMPTZ,
    expected_at     DATE,
    total_amount    DECIMAL(12,2)
);

CREATE TABLE po_items (
    id              BIGSERIAL PRIMARY KEY,
    po_id           BIGINT REFERENCES purchase_orders(id),
    product_id      BIGINT REFERENCES inv_products(id),
    ordered_qty     INT NOT NULL,
    received_qty    INT DEFAULT 0,
    unit_cost       DECIMAL(10,2)
);
```

### Low Stock Alert Query

```sql
-- Products below reorder level
SELECT
    p.sku,
    p.name,
    p.reorder_level,
    p.reorder_qty,
    SUM(s.qty_on_hand - s.qty_reserved) AS available_stock,
    SUM(s.qty_on_order) AS incoming_stock
FROM inv_products p
JOIN stock s ON s.product_id = p.id
GROUP BY p.id, p.sku, p.name, p.reorder_level, p.reorder_qty
HAVING SUM(s.qty_on_hand - s.qty_reserved) < p.reorder_level
ORDER BY available_stock ASC;
```

### Stock Movement with Django ORM

```python
@transaction.atomic
def receive_stock(po_item_id, received_qty, warehouse_id):
    po_item = POItem.objects.select_for_update().get(id=po_item_id)

    # Update PO item
    po_item.received_qty = F('received_qty') + received_qty
    po_item.save(update_fields=['received_qty'])

    # Update stock
    stock, _ = Stock.objects.get_or_create(
        product_id=po_item.product_id,
        warehouse_id=warehouse_id,
    )
    Stock.objects.filter(id=stock.id).update(
        qty_on_hand=F('qty_on_hand') + received_qty,
        qty_on_order=F('qty_on_order') - received_qty,
    )

    # Log movement (immutable)
    StockMovement.objects.create(
        product_id   = po_item.product_id,
        warehouse_id = warehouse_id,
        type         = 'purchase_receipt',
        qty          = received_qty,
        reference_no = f'PO-{po_item.po_id}',
    )
```

---

## 15.9 Common Design Patterns

### Pattern 1: Immutable Ledger

```
Banking transactions, stock movements, order history —
এগুলো কখনো UPDATE বা DELETE করো না।
Mistake হলে reversal/adjustment entry দাও।

কেন?
  ✅ Audit trail বজায় থাকে
  ✅ Regulatory compliance
  ✅ Bug investigation সহজ
  ✅ Data integrity
```

### Pattern 2: Snapshot Pricing

```sql
-- order_items তে unit_price snapshot রাখো
-- product এর price পরে change হলেও
-- পুরনো order এর price ঠিক থাকবে

order_items.unit_price = products.price AT TIME OF ORDER
```

### Pattern 3: Denormalized Counters

```
likes.count, followers.count, posts.count —
এগুলো real-time COUNT(*) query না করে
denormalized column এ রাখো।

UPDATE posts SET like_count = like_count + 1 WHERE id = X;

কিন্তু: counter inconsistency হতে পারে।
Periodic reconciliation job চালাও।
```

### Pattern 4: Soft Delete

```sql
-- Production এ DELETE করো না — soft delete করো
ALTER TABLE products ADD COLUMN deleted_at TIMESTAMPTZ;
ALTER TABLE products ADD COLUMN is_deleted BOOLEAN DEFAULT FALSE;

-- Delete
UPDATE products SET is_deleted = TRUE, deleted_at = NOW() WHERE id = X;

-- Query (always filter)
SELECT * FROM products WHERE is_deleted = FALSE;

-- Django: use django-soft-delete অথবা custom manager
```

### Pattern 5: Status State Machine

```
Order status: pending → confirmed → shipped → delivered
                                  ↘ cancelled

কখনো arbitrary status jump করতে দিও না।
State transition validate করো।

def transition_order(order, new_status):
    valid_transitions = {
        'pending':   ['confirmed', 'cancelled'],
        'confirmed': ['shipped', 'cancelled'],
        'shipped':   ['delivered'],
        'delivered': [],
        'cancelled': [],
    }
    if new_status not in valid_transitions[order.status]:
        raise ValueError(f"Cannot go from {order.status} to {new_status}")
```

---

## 15.10 Interview Questions

### Beginner Level

**Q1: E-commerce এ order_items এ unit_price কেন store করা হয়?**
> Product এর price যেকোনো সময় change হতে পারে। Order place করার সময়ের price snapshot রাখলে historical order report সঠিক থাকে। পুরনো invoice তে সেই সময়ের price দেখাবে।

**Q2: Banking transaction table এ কেন UPDATE করা হয় না?**
> Transaction হলো immutable financial record। Correction করতে হলে reversal transaction তৈরি করতে হয়। এতে audit trail বজায় থাকে, regulatory compliance হয়, এবং যেকোনো dispute এ প্রমাণ থাকে।

**Q3: Social media তে follower_count কেন separate table থেকে COUNT না করে column এ রাখা হয়?**
> COUNT(*) query expensive — millions of rows এ slow। Denormalized counter fast read দেয়। Counter increment/decrement atomic UPDATE দিয়ে করা হয়।

---

### Intermediate Level

**Q4: Food delivery app এ restaurant nearby search কীভাবে করবে?**
> Haversine formula দিয়ে lat/lng থেকে distance calculate করা যায়। Production এ PostGIS extension ব্যবহার করো — spatial index দিয়ে অনেক দ্রুত। Redis GEO commands (GEOADD, GEODIST, GEORADIUS) ও ব্যবহার করা হয় real-time driver tracking এ।

**Q5: Ride sharing app এ driver location কোথায় store করবে?**
> Real-time location (high frequency) → Redis HASH — fast read/write, TTL set করো।
> Permanent storage (trip history) → PostgreSQL — partitioned by date।
> Batch write: every 30 seconds Redis থেকে PostgreSQL এ flush করো।

**Q6: Chat app এ unread count কীভাবে efficiently করবে?**
> conversation_members table এ last_read_at column রাখো। Unread count = messages WHERE sent_at > last_read_at। Message read করলে last_read_at = NOW() update করো। COUNT(*) avoid করতে Redis এ unread count cache করা যায়।

---

### Senior / FAANG Level

**Q7: Social media news feed design করো — 500M users, 50M posts/day।**

```
Fan-out on Write (Push Model):
  Post করলে সব followers এর feed table তে insert।
  Fast read — pre-computed feed।
  Problem: Celebrity with 50M followers — 50M inserts per post!

Fan-out on Read (Pull Model):
  Feed read করার সময় followings এর posts merge করো।
  No write amplification।
  Problem: Slow read — many queries।

Hybrid (Instagram/Twitter approach):
  Normal users (< 1M followers): fan-out on write
  Celebrities (> 1M followers):  fan-out on read

  Feed read করার সময়:
    - Pre-computed feed (normal users)
    - + Celebrity posts (real-time merge)
    = Final feed

  Redis: feed cache, celebrity post list
  PostgreSQL: permanent storage
  Cassandra: high-write feed storage
```

**Q8: E-commerce inventory এ race condition কীভাবে handle করবে?**

```
Problem:
  100 users একই সময়ে last 1 item কিনতে চাইছে।
  Naive approach: read qty → check > 0 → update qty - 1
  Race condition: সবাই qty=1 পড়বে, সবাই success পাবে!
  Result: qty = -99!

Solution 1 — Database lock:
  SELECT qty FROM inventory WHERE id = X FOR UPDATE;
  -- Lock নিয়ে check এবং update করো

Solution 2 — Optimistic locking:
  UPDATE inventory
  SET qty = qty - 1, version = version + 1
  WHERE id = X AND qty > 0 AND version = :expected_version;
  -- rowcount = 0 হলে retry

Solution 3 — Atomic check-and-decrement:
  UPDATE inventory
  SET reserved = reserved + :qty
  WHERE id = X AND (qty - reserved) >= :qty;
  -- Single atomic operation — most efficient

Solution 4 — Redis (for very high traffic):
  DECRBY inventory:product:101 1
  -- Atomic, fast। Periodic sync to PostgreSQL।
```

---

## 15.11 Hands-on Exercises

### Exercise 1 — E-commerce
```sql
-- এই queries লেখো:
-- 1. Top 10 best-selling products (last 30 days)
-- 2. Revenue by category
-- 3. Orders with pending payment > 24 hours
-- 4. Customer lifetime value (total spend per user)
```

### Exercise 2 — Banking
```sql
-- এই queries লেখো:
-- 1. Daily transaction volume report
-- 2. Accounts with suspicious activity (> 10 transactions/hour)
-- 3. Balance sheet (total credits vs debits per account)
```

### Exercise 3 — Django ORM
```python
# Implement করো:
# 1. place_order() with inventory reservation
# 2. transfer_money() with deadlock prevention
# 3. get_user_feed() with pagination
# 4. low_stock_alert() query
```

### Mini Project
```
Choose করো যেকোনো একটা:
  Option A: E-commerce (Django + PostgreSQL)
    - Product catalog with variants
    - Cart and order placement
    - Inventory management
    - Order status tracking

  Option B: Food Delivery
    - Restaurant and menu management
    - Order placement
    - Rider assignment
    - Status tracking

  Deliverables:
    - ER diagram (text format)
    - Django models
    - Key business logic (atomic transactions)
    - Basic API endpoints
    - Index strategy
```

---

## 15.12 Module Summary & Cheat Sheet

```
REAL-WORLD DATABASE DESIGN CHEAT SHEET
========================================

E-COMMERCE:
  products → product_variants → inventory
  orders → order_items (price snapshot!)
  Inventory reservation: SELECT FOR UPDATE
  Full-text search: GIN index on tsvector

BANKING:
  accounts.balance — always ACID
  transactions — immutable ledger
  Transfer: lock both accounts (consistent order)
  Correction: reversal transaction, never UPDATE

FOOD DELIVERY:
  restaurant + menu_items + food_orders
  GPS: Redis real-time, PostgreSQL batch
  Order status: state machine (validate transitions)

RIDE SHARING:
  drivers.status: offline/available/on_trip
  ride_events: partitioned table (high volume GPS)
  Location: Redis HASH + periodic DB flush
  Fare: dynamic calculation with surge

SOCIAL MEDIA:
  follows (follower_id, followee_id) — graph
  Counters: denormalized (like_count, follower_count)
  Feed: fan-out on write (normal) + pull (celebrities)
  Posts: partitioned by month

CHAT:
  conversations → messages (partitioned)
  last_read_at for unread count
  message_status for delivery receipts

HOSPITAL:
  patients → appointments → medical_records
  prescriptions → prescription_items
  lab_tests: flexible result JSONB
  bills: itemized bill_items

INVENTORY:
  stock (qty_on_hand, qty_reserved, qty_on_order)
  stock_movements — immutable ledger
  Low stock: HAVING qty < reorder_level

KEY PATTERNS:
  ✅ Immutable ledger (transactions, movements)
  ✅ Price snapshot (order_items.unit_price)
  ✅ Denormalized counters (like_count)
  ✅ Soft delete (is_deleted, deleted_at)
  ✅ State machine (order status transitions)
  ✅ Partition high-volume tables (events, messages, posts)
  ✅ Redis for real-time data (location, counters)
  ✅ SELECT FOR UPDATE for concurrent writes
  ✅ Consistent lock order (prevent deadlock)
  ✅ JSONB for flexible/dynamic attributes

INDEXES (always needed):
  Foreign keys → index করো
  Status columns → index করো
  created_at → index করো (range queries)
  Search columns → GIN/GiST index
  Unique constraints → automatic index
```
