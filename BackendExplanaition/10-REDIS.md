# LEVEL 10 — Redis Deep Dive

Redis is an **in-memory data structure store**. That one fact explains almost everything about
when to use it: it's blazing fast (RAM, not disk) but not durable/relational the way Postgres is.
Everything below builds on that single idea.

```
┌─────────────────────────────────────────────────────────┐
│                    YOUR APPLICATION                       │
└───────────┬───────────────────────────┬───────────────────┘
            │                           │
            ▼                           ▼
   ┌─────────────────┐         ┌──────────────────┐
   │     Redis        │         │    PostgreSQL      │
   │  (in-memory,      │         │  (disk-backed,      │
   │   key-value)       │         │   relational)         │
   │                    │         │                       │
   │  fast, ephemeral,   │         │  durable, complex       │
   │  simple data model  │         │  queries, transactions  │
   └─────────────────┘         └──────────────────┘
```

---

## 1. Redis as a CACHE

The most common use: store the result of an expensive computation/query so the next request
doesn't have to redo the work.

```
┌────────┐   1. GET user:42        ┌─────────┐
│ Client │ ───────────────────────▶│  Redis   │
└────────┘                          └────┬────┘
                                          │ 2. MISS (not cached)
                                          ▼
                                    ┌─────────┐
                                    │ Postgres │  3. SELECT * FROM users WHERE id=42
                                    └────┬────┘
                                          │ 4. row returned
                                          ▼
                                    ┌─────────┐
                                    │  Redis   │  5. SET user:42 <data> EX 300
                                    └─────────┘
                                          │ 6. return data
                                          ▼
                                    ┌────────┐
                                    │ Client │
                                    └────────┘
```

```python
import redis, json

r = redis.Redis(host="localhost", port=6379, db=0)

def get_user(user_id):
    key = f"user:{user_id}"
    cached = r.get(key)
    if cached:
        return json.loads(cached)           # cache HIT — no DB hit at all

    user = db.query(f"SELECT * FROM users WHERE id={user_id}")  # cache MISS
    r.setex(key, 300, json.dumps(user))     # cache it for 300s (5 min)
    return user
```

---

## 2. Redis as a QUEUE

Redis Lists (`LPUSH`/`RPOP`) or the more robust **Streams** (`XADD`/`XREAD`) act as a lightweight
message queue between producers and consumers.

```
Producer                    Redis LIST "task_queue"                 Worker(s)
   │                        ┌───┬───┬───┬───┐                          │
   │  LPUSH task_queue job ─▶│ 3 │ 2 │ 1 │   │◀─ RPOP task_queue ───────┤
   │                        └───┴───┴───┴───┘                          │
```

```python
# Producer
r.lpush("task_queue", json.dumps({"task": "send_email", "to": "a@x.com"}))

# Worker (blocking pop — waits until a job appears, avoids polling)
while True:
    _, job = r.brpop("task_queue")
    process(json.loads(job))
```

For production job queues you'd typically reach for **Celery** or **RQ** (both use Redis as
their broker under the hood) rather than hand-rolling this.

---

## 3. Redis as PUB/SUB

One-to-many, fire-and-forget messaging. Unlike a queue, messages are **not stored** — if no one
is subscribed at the moment of publish, the message is lost.

```
                     ┌──────────────┐
        publish ────▶│  Redis        │──── fan-out ────▶ Subscriber A
        "chat:room1" │  Channel:      │──── fan-out ────▶ Subscriber B
                     │  chat:room1    │──── fan-out ────▶ Subscriber C
                     └──────────────┘
```

```python
# Publisher
r.publish("chat:room1", json.dumps({"user": "Alice", "msg": "hello"}))

# Subscriber
pubsub = r.pubsub()
pubsub.subscribe("chat:room1")
for message in pubsub.listen():
    if message["type"] == "message":
        print(json.loads(message["data"]))
```

Use case: real-time chat, live notifications, broadcasting cache-invalidation events across
multiple app servers.

---

## 4. Redis as SESSION STORAGE

Instead of storing session data in the app server's memory (which breaks when you scale to
multiple servers), store it in Redis — any server can read/write it.

```
┌────────┐          ┌──────────────┐          ┌─────────┐
│ Server1 │ ◀──────▶│               │◀───────▶ │ Server2  │
└────────┘          │  Redis         │          └─────────┘
                     │  session:abc123 │
┌────────┐          │  → {user_id:5}   │          ┌─────────┐
│ Server3 │ ◀──────▶│               │◀───────▶ │ Server4  │
└────────┘          └──────────────┘          └─────────┘

Any server can serve the request — session isn't tied to one machine.
```

```python
def create_session(user_id):
    session_id = str(uuid.uuid4())
    r.setex(f"session:{session_id}", 3600, json.dumps({"user_id": user_id}))
    return session_id

def get_session(session_id):
    data = r.get(f"session:{session_id}")
    return json.loads(data) if data else None
```

---

## 5. Redis as a RATE LIMITER

`INCR` is atomic — perfect for counting requests per time window without race conditions.

```python
def is_rate_limited(user_id, limit=100, window_seconds=60):
    key = f"ratelimit:{user_id}"
    current = r.incr(key)          # atomic: increments AND returns new value
    if current == 1:
        r.expire(key, window_seconds)   # only set TTL on the FIRST request in the window
    return current > limit

if is_rate_limited(user.id):
    raise HTTPException(429, "Too many requests")
```

```
Timeline for user "42", limit=3 per 10s:

t=0s   INCR ratelimit:42 → 1   (EXPIRE set to 10s)
t=2s   INCR ratelimit:42 → 2
t=4s   INCR ratelimit:42 → 3
t=5s   INCR ratelimit:42 → 4   → BLOCKED (over limit)
t=10s  key expires, counter resets to 0
```

For smoother limiting (avoiding bursts right at window boundaries), use a **sliding window**
via sorted sets (`ZADD`/`ZREMRANGEBYSCORE`) instead of a fixed counter.

---

## 6. Redis as a DISTRIBUTED LOCK

When multiple app servers/workers must not run the same critical section simultaneously (e.g.,
"only one worker should process this specific order"), Redis's atomic `SET key val NX EX ttl`
gives you a lock that self-expires even if the lock holder crashes.

```
Worker A                     Redis                      Worker B
   │  SET lock:order42 A NX EX 10 ──▶ OK (lock acquired)     │
   │                                                          │  SET lock:order42 B NX EX 10
   │                                                          │  ──▶ nil (already locked, fails)
   │  ... does the work ...                                   │  ... backs off / retries ...
   │  DEL lock:order42 (release) ──▶ 1                        │
```

```python
import uuid

def acquire_lock(lock_name, ttl=10):
    token = str(uuid.uuid4())
    acquired = r.set(f"lock:{lock_name}", token, nx=True, ex=ttl)
    return token if acquired else None

def release_lock(lock_name, token):
    # Only release if we still own it (avoid releasing someone else's lock
    # if ours already expired and was re-acquired by another worker)
    lua_script = """
    if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
    else
        return 0
    end
    """
    r.eval(lua_script, 1, f"lock:{lock_name}", token)

token = acquire_lock("order:42")
if token:
    try:
        process_order(42)
    finally:
        release_lock("order:42", token)
else:
    print("Another worker is already processing this order")
```

`NX` = "set only if key doesn't already exist" — this is what makes acquisition atomic. The
release script checks ownership via the random token before deleting, to avoid a subtle bug where
Worker A's expired lock gets deleted by A after Worker B has already legitimately acquired it.

---

## Core Commands

```python
r.set("key", "value")            # SET — store a value
r.get("key")                     # GET — retrieve a value
r.setex("key", 60, "value")      # SET + EXPIRE combined — expires in 60s
r.expire("key", 60)              # EXPIRE — set/reset TTL on existing key
r.ttl("key")                     # check remaining seconds ( -1 = no expiry, -2 = doesn't exist)
r.del("key")                     # DEL — remove a key
r.incr("counter")                # INCR — atomic increment (creates key at 0 first if missing)
r.decr("counter")                # DECR — atomic decrement
r.exists("key")                  # check existence, returns 1 or 0
```

---

## Key Naming (Prefixing) — the "how do I organize keys" question

Redis is a **flat global namespace** — there's no concept of tables/schemas like SQL. Everyone's
keys live in the same space, so a **consistent prefix convention** is what keeps things sane and
lets you group, scan, and bulk-invalidate related keys.

### Convention: `namespace:entity:id:field`

```python
# Good, structured, greppable keys
"user:42"                     # a whole user object
"user:42:profile"             # sub-resource
"user:42:orders"              # user's orders list
"session:abc123"
"cache:products:page:2"
"ratelimit:api:user:42"
"lock:order:42"
```

```python
def cache_key(*parts) -> str:
    """Central helper — every cache key in the app goes through this,
    so the naming convention is enforced in ONE place."""
    return ":".join(str(p) for p in parts)

cache_key("user", 42, "profile")   # -> "user:42:profile"
```

**Why prefixing matters:**

1. **Avoids collisions** between different features/teams sharing one Redis instance.
2. **Enables pattern-based operations** — you can find/delete all keys belonging to one entity.
3. **Makes multi-tenant setups possible** — e.g. `tenant:acme:user:42` isolates data per client
   without needing separate Redis instances.

```python
# Multi-tenant prefixing
def tenant_key(tenant_id, *parts):
    return ":".join([f"tenant:{tenant_id}"] + [str(p) for p in parts])

tenant_key("acme_corp", "user", 42)   # -> "tenant:acme_corp:user:42"
```

### Finding keys by prefix

```python
# SCAN is the safe way (non-blocking, cursor-based) — never use KEYS in production,
# it blocks the whole server while it scans everything.
for key in r.scan_iter(match="user:42:*"):
    print(key)
```

---

## Cache Invalidation — the hard part

> "There are only two hard things in Computer Science: cache invalidation and naming things."

The core problem: your cache and your source of truth (Postgres) can drift out of sync. There
are three main strategies:

### Strategy 1 — TTL (let it expire naturally)

Simplest approach: every cached value has an expiry; stale data self-heals after the TTL passes.

```python
r.setex("user:42", 300, json.dumps(user_data))   # auto-expires in 5 minutes
```

**Trade-off:** up to 5 minutes of staleness after an update. Fine for data that doesn't need to
be instantly fresh (product listings, view counts). Bad for data that must be accurate right
after a write (account balance).

### Strategy 2 — Explicit invalidation on write

When you update the source of truth, immediately delete (or update) the corresponding cache key.

```
┌────────┐   UPDATE users SET name=... ┌──────────┐
│ Client │ ────────────────────────────▶│ Postgres │
└────────┘                              └────┬─────┘
                                              │ write succeeded
                                              ▼
                                        ┌──────────┐
                                        │  DEL      │  invalidate the stale cache entry
                                        │  user:42  │  (next GET will be a cache miss →
                                        └──────────┘   refetch fresh data from Postgres)
```

```python
def update_user(user_id, new_data):
    db.execute("UPDATE users SET name = %s WHERE id = %s", (new_data["name"], user_id))
    r.delete(f"user:{user_id}")     # invalidate immediately — next read repopulates fresh
```

**Pattern-based invalidation** — when one write affects many cached keys (e.g., updating a
product invalidates several cached listing pages):

```python
def invalidate_product_caches(product_id):
    r.delete(f"product:{product_id}")
    for key in r.scan_iter(match=f"cache:products:*"):   # invalidate all listing pages
        r.delete(key)
```

### Strategy 3 — Cache-Aside (a.k.a. Lazy Loading)

This is the **most common pattern** in real backends, and it's what the first "cache" example
above already showed. The application code (not Redis itself) is responsible for the read/write
logic:

```
READ path:
  1. Check cache
  2. If HIT → return cached value
  3. If MISS → read from DB → write result into cache → return it

WRITE path:
  1. Write to DB
  2. Invalidate (delete) the corresponding cache key — do NOT try to update the cache in place,
     that's a common source of bugs. Just delete it and let the next read repopulate it.
```

```python
def get_user(user_id):
    key = f"user:{user_id}"
    cached = r.get(key)
    if cached:
        return json.loads(cached)
    user = db.get_user(user_id)
    r.setex(key, 300, json.dumps(user))
    return user

def update_user(user_id, data):
    db.update_user(user_id, data)
    r.delete(f"user:{user_id}")     # cache-aside: invalidate, don't update in place
```

**Why "delete" and not "update the cache directly on write"?** If two writes race, updating the
cache directly can leave it holding the result of the *older* write while the DB has the newer
one. Deleting and letting the next read repopulate from the DB avoids that race entirely.

---

## Cache Stampede (a.k.a. Dogpile Effect)

**The problem:** a popular cache key expires. In the same instant, 1,000 concurrent requests all
get a cache MISS, and all 1,000 hammer the database simultaneously trying to regenerate the same
value — potentially taking the DB down right when load is highest.

```
Cache key "trending_products" expires at t=0
                                                                     ┌──────────┐
Request 1  ──MISS──┐                                             ┌─▶│           │
Request 2  ──MISS──┼── ALL 1000 requests hit the DB at once ─────┼─▶│ Postgres  │  💥 overloaded
Request 3  ──MISS──┤    to recompute the SAME expensive query    ├─▶│           │
   ...     ──MISS──┘                                             └─▶│           │
Request 1000 ──MISS─                                                └──────────┘
```

### Fix 1 — Locking (only one request regenerates, others wait)

```python
def get_trending_products():
    key = "trending_products"
    cached = r.get(key)
    if cached:
        return json.loads(cached)

    lock_token = acquire_lock(f"lock:{key}", ttl=10)
    if lock_token:
        try:
            # double-check — someone else might have just filled it while we waited for the lock
            cached = r.get(key)
            if cached:
                return json.loads(cached)
            data = expensive_db_query()
            r.setex(key, 300, json.dumps(data))
            return data
        finally:
            release_lock(f"lock:{key}", lock_token)
    else:
        time.sleep(0.05)                 # someone else is already regenerating — wait briefly
        return get_trending_products()   # then retry (will likely hit cache now)
```

### Fix 2 — Early / probabilistic expiration (recompute slightly before actual expiry)

Refresh the cache proactively, staggered, *before* it goes stale — so it almost never
actually hits zero TTL under load.

```python
import random

def get_with_early_refresh(key, ttl, fetch_fn):
    value, remaining_ttl = r.get(key), r.ttl(key)
    if value and remaining_ttl > 0:
        # small random chance to refresh early, spreading refreshes over time
        # instead of all keys expiring in one synchronized burst
        if random.random() < 0.01:
            fresh = fetch_fn()
            r.setex(key, ttl, json.dumps(fresh))
        return json.loads(value)

    fresh = fetch_fn()
    r.setex(key, ttl, json.dumps(fresh))
    return fresh
```

### Fix 3 — Never let popular keys expire; refresh via a background job instead

For truly hot keys, don't rely on TTL-triggered regeneration at all — a scheduled job
(cron/Celery beat) recomputes and re-writes the key every N seconds, so user-facing reads
*never* trigger a DB hit.

---

## When should I use PostgreSQL vs Redis?

```
┌───────────────────────────────────────────────────────────┐
│                     Decision Flow                            │
└───────────────────────────────────────────────────────────┘

Does the data need to survive a crash/restart with full durability?
   │
   ├── YES ──▶ PostgreSQL (Redis persistence — RDB/AOF — exists but is
   │                        secondary to its design goal; it's not built
   │                        to be your durable system of record)
   │
   └── NO ──▶ Is it relational, needs joins, complex filtering,
              transactions across tables, or must be queryable in
              ways you can't predict in advance?
                │
                ├── YES ──▶ PostgreSQL
                │
                └── NO ──▶ Is it about SPEED — hot data accessed
                           extremely often, or ephemeral coordination
                           (locks, rate limits, sessions, pub/sub)?
                              │
                              └── YES ──▶ Redis
```

| Use Case | Choose | Why |
|---|---|---|
| User accounts, orders, payments | **PostgreSQL** | Needs ACID transactions, relationships, durability — losing this data is unacceptable |
| Product catalog with complex filtering/search | **PostgreSQL** | Relational queries, joins across categories/inventory/pricing |
| Caching expensive query results | **Redis** | Speed matters, data is disposable/regenerable from Postgres |
| User sessions | **Redis** | Ephemeral, needs to be fast, shared across app servers, TTL built-in |
| Rate limiting | **Redis** | Needs atomic increments at very high frequency; losing counters on restart is acceptable |
| Distributed locks | **Redis** | Needs atomicity + auto-expiry; not meant to be permanent state |
| Leaderboards / real-time rankings | **Redis** | Sorted Sets give O(log n) ranked inserts/reads — Postgres can do it but far less naturally |
| Job/task queue | **Redis** (or a dedicated broker) | Needs fast push/pop; jobs are transient, not permanent records |
| Real-time chat / pub-sub notifications | **Redis** | Built-in Pub/Sub primitive; low latency fan-out |
| Financial ledger, audit trail | **PostgreSQL** | Must never lose data, must be queryable/auditable years later |
| Full-text search with ranking | **PostgreSQL** (or dedicated search engine like Elasticsearch) | Redis isn't built for this; Postgres has `tsvector`, or offload to a search-specific tool |

### The mental shortcut

- **Postgres = the source of truth.** If losing this data would be a disaster, it belongs here.
- **Redis = the accelerator + coordination layer.** If this data disappeared right now, your app
  should be able to regenerate it or simply degrade gracefully (cache miss → slower response,
  not data loss).

Most real backends use **both together**: Postgres as the durable system of record, Redis in
front of it (cache-aside) and alongside it (sessions, locks, rate limits, pub/sub) to keep things
fast and coordinated — exactly the split shown in the diagram at the top of this document.
