# LEVEL 17 — System Design Deep Dive

System design is really just answering one question over and over, at every layer: **"what
happens when this breaks, or when 100x more traffic hits it?"** Everything below is a tool for
answering that question for a specific failure mode or bottleneck.

---

# PART A — The Quality Attributes (What You're Optimizing For)

## 1. Scalability

The ability to handle growth — more users, more data, more traffic — by adding resources, ideally
without a fundamental redesign.

```
Vertical scaling (scale UP):              Horizontal scaling (scale OUT):
┌───────────┐        ┌───────────┐        ┌───────┐   ┌───────┐   ┌───────┐
│  1 server    │  ──▶  │  1 BIGGER    │        │Server1 │   │Server2 │   │Server3 │
│  (4 CPU)      │        │  server        │        └───────┘   └───────┘   └───────┘
│               │        │  (32 CPU)       │           add MORE machines, not bigger ones
└───────────┘        └───────────┘
   simple, but hits a       expensive at the top end,
   hard ceiling               and still a single point of failure
```

**Why it matters:** a system that works fine for 100 users can completely fall over at 100,000 if
it wasn't designed to scale horizontally — this is the reason load balancers, replicas, and
sharding (below) exist.

## 2. Availability

The percentage of time a system is up and able to serve requests. Usually expressed in "nines."

```
99%      ("two nines")   = ~3.65 days of downtime per year
99.9%    ("three nines") = ~8.7 hours of downtime per year
99.99%   ("four nines")  = ~52 minutes of downtime per year
99.999%  ("five nines")  = ~5 minutes of downtime per year
```

**Why it matters:** determines how much redundancy you need. Getting from 99.9% to 99.99% often
requires eliminating every single point of failure — one server, one DB instance, one network
path — since any one of them failing takes the whole system down otherwise.

## 3. Reliability

The system does the **correct thing consistently**, not just "is up" — a server that responds
instantly with wrong data is available but not reliable.

```
Available + WRONG answers   =  NOT reliable
Available + CORRECT answers  =  reliable
DOWN                          =  neither available nor reliable
```

**Why it matters:** reliability is what tests (Level 16), proper error handling, and idempotent
retries protect. An "available" payment system that occasionally double-charges is a disaster
despite perfect uptime.

## 4. Consistency

Whether all parts of a distributed system see the **same data at the same time**. This becomes a
real design decision once you have caches, replicas, or multiple database nodes.

```
Strong consistency:                      Eventual consistency:
Write to DB ──▶ read IMMEDIATELY          Write to primary ──▶ replicas catch up
after gets the new value, always            ASYNCHRONOUSLY — a read from a replica
(often requires waiting for all             right after the write might still see
replicas to agree — slower)                 the OLD value for a short window (faster)
```

**Why it matters:** this is a genuine trade-off, not a solved problem — banking transactions
usually need strong consistency; a social media "like count" is usually fine being eventually
consistent (a few seconds of staleness is imperceptible and lets the system scale far more).

## 5. Latency

How long a **single** request takes, end to end (often measured as p50/p95/p99 — the 95th/99th
percentile matters more than the average, since averages hide bad outlier experiences).

```
p50 (median): 45ms      ← half of requests are faster than this
p95:          180ms     ← 95% of requests are faster than this
p99:          900ms     ← the slowest 1% — often reveals real problems (GC pauses,
                            slow DB queries, cold caches) that the average conceals
```

**Why it matters:** caching (Redis), CDNs, and read replicas all exist primarily to cut latency —
put data physically/logically closer to where it's needed instead of always hitting the slowest
path.

## 6. Throughput

How many requests the system can handle **per unit time** (e.g., requests/second). Distinct from
latency — a system can have low latency but low throughput (fast per-request, but chokes under
concurrent load), or vice versa.

```
Latency:    time for ONE request to complete       (e.g., "180ms")
Throughput: how many requests handled PER SECOND    (e.g., "5,000 req/s")

A single-lane road: cars move fast (low latency) but few can pass per minute (low throughput)
A 10-lane highway: each car takes the same time, but far more pass per minute (high throughput)
```

**Why it matters:** horizontal scaling, load balancing, and connection pooling primarily boost
throughput — they let you serve more requests concurrently, even if each individual request's
latency stays the same.

---

# PART B — The Components (And Why Each One Exists)

## 7. Load Balancer

**Problem it solves:** one server can only handle so many concurrent requests, and if that one
server dies, everything goes down.

```
                        ┌──────────────┐
         requests ────▶ │  Load Balancer  │
                        └──────┬───────┘
                    distributes across many
             ┌───────────────┼───────────────┐
             ▼               ▼               ▼
      ┌───────────┐  ┌───────────┐  ┌───────────┐
      │ Backend 1    │  │ Backend 2    │  │ Backend 3    │
      └───────────┘  └───────────┘  └───────────┘

  If Backend 2 dies, the load balancer detects it (health checks) and
  stops sending it traffic — the other two absorb the load, users never notice.
```

**Why it exists:** turns "one server, one point of failure, one throughput ceiling" into "N
servers, redundant, N times the throughput" — this is the foundational building block of
horizontal scaling and high availability.

## 8. Caching

**Problem it solves:** re-computing/re-fetching the same expensive data on every request wastes
time and database capacity.

```
Without cache: every request ──▶ hits the database ──▶ slow, DB gets hammered
With cache:    most requests  ──▶ hit Redis (fast, in-memory) ──▶ DB only hit on cache misses
```

**Why it exists:** directly attacks both **latency** (memory is orders of magnitude faster than
disk-backed DB queries) and **throughput** (the DB, usually the hardest component to scale
horizontally, sees far less load). Covered in depth in Level 10.

## 9. Database Replication & 10. Read Replica

**Problem it solves:** a single database instance is both a single point of failure AND a
bottleneck — every read and write competes for the same resources.

```
                    ┌──────────────┐
   writes ────────▶ │  Primary DB     │
                    └──────┬───────┘
                            │  replicates changes (usually async)
                ┌───────────┼───────────┐
                ▼           ▼           ▼
        ┌───────────┐┌───────────┐┌───────────┐
        │ Read Replica 1││ Read Replica 2││ Read Replica 3│
        └───────────┘└───────────┘└───────────┘
                ▲           ▲           ▲
              reads       reads       reads   (spread across replicas)
```

```python
# Application-level read/write splitting
def get_user(user_id):
    return read_replica_db.query(f"SELECT * FROM users WHERE id={user_id}")  # goes to a replica

def update_user(user_id, data):
    primary_db.execute(f"UPDATE users SET ... WHERE id={user_id}")            # MUST go to primary
```

**Why it exists:** most applications are heavily **read-skewed** (many more reads than writes) —
replicas let you scale read capacity horizontally by adding more replica servers, while keeping a
single source of truth (the primary) for writes to avoid write-conflict complexity. Also provides
redundancy — a replica can be promoted to primary if the original primary fails.

**The consistency trade-off:** replication is usually asynchronous, so a replica can be
momentarily behind the primary — this is the "eventual consistency" concept from Part A in
practice. A user who just updated their profile and immediately re-reads from a lagging replica
might briefly see stale data.

## 11. Sharding (Partitioning)

**Problem it solves:** replication scales *reads*, but every replica still holds a **full copy**
of all the data — eventually a single database's total data size or write throughput outgrows
what any one machine (even the primary) can handle at all.

```
Without sharding: ONE database holds ALL users (1 through 10,000,000)
                          ┌────────────────────┐
                          │   ALL users 1-10M      │  ← one giant, ever-growing DB
                          └────────────────────┘

With sharding: split the data across multiple databases by some key
      ┌────────────┐   ┌────────────┐   ┌────────────┐
      │ Shard A         │   │ Shard B         │   │ Shard C         │
      │ users 1-3.3M     │   │ users 3.3M-6.6M  │   │ users 6.6M-10M   │
      └────────────┘   └────────────┘   └────────────┘
             ▲                  ▲                  ▲
      user_id % 3 == 0    user_id % 3 == 1    user_id % 3 == 2
      (or a hash of a shard key routes each request to the right shard)
```

**Why it exists:** distributes both storage AND write load across multiple independent database
instances — each shard is a smaller, faster, independently-scalable database. The trade-off:
queries that need to span multiple shards (e.g., "find all users across the whole system matching
X") become much harder — this is the real cost of sharding, and why it's usually a last resort
after replication and caching are maxed out.

## 12. Message Queue

**Problem it solves:** covered in depth in Level 11 — decouples slow/non-essential work from the
request-response cycle, and smooths out traffic spikes.

```
Producer ──▶ [ Queue: buffers messages ] ──▶ Consumer(s)

If a traffic spike sends 10,000 tasks in one second, but workers can only process
1,000/second, the queue absorbs the burst — tasks wait in line instead of overwhelming
the workers or getting dropped entirely.
```

**Why it exists:** decoupling (producers and consumers don't need to know about each other or run
at the same rate) and load-leveling (queues absorb bursts, converting spiky load into a steady
stream workers can keep up with).

## 13. CDN (Content Delivery Network)

**Problem it solves:** if your only server is in one data center (say, Virginia), a user in Tokyo
pays the full round-trip network latency for every request, even for content that never changes
(images, CSS, JS bundles, videos).

```
Without a CDN:                              With a CDN:
User in Tokyo ──────────────────▶            User in Tokyo ──▶ nearest CDN edge server
                    (slow,                                       (in/near Tokyo — FAST)
                  long trip)                                            │
                     │                                          cache miss? then, only
                     ▼                                          ONCE, fetch from origin
              Origin server                                     and cache it at the edge
              (Virginia)                                        for all future Tokyo users
```

**Why it exists:** attacks latency by moving static content **physically closer** to users
worldwide via edge servers, and reduces load on your origin server since most requests for static
assets never even reach it after the first cache fill.

## 14. Microservices vs Monolith

**The trade-off, not a strict "better/worse":**

```
Monolith:                                  Microservices:
┌─────────────────────────┐              ┌──────┐ ┌──────┐ ┌──────┐
│                              │              │ Users   │ │ Orders  │ │ Payments│
│   One codebase, one deploy      │              │ service │ │ service │ │ service │
│   Users + Orders + Payments      │              └──────┘ └──────┘ └──────┘
│   all in ONE application            │                 each independently deployable,
│                              │              scalable, and ownable by a separate team
└─────────────────────────┘
```

| | Monolith | Microservices |
|---|---|---|
| Deployment | One unit, simpler | Many independent deploys, more complex orchestration |
| Scaling | Scale the WHOLE app, even if only one part is under load | Scale just the busy service (e.g., only Payments) |
| Team structure | Works well for small teams / early-stage products | Shines when many teams need to work independently without stepping on each other |
| Failure isolation | A bug/crash can take down the whole app | A failing service can (ideally) degrade gracefully without taking others down |
| Complexity | Lower operational complexity | Higher — network calls between services, distributed tracing, data consistency across services |

**Why the choice exists at all:** most successful products **start as a monolith** (simpler to
build and reason about with a small team) and only split into microservices once a specific
part of the system needs independent scaling or independent team ownership — splitting too early
adds enormous complexity for a problem you don't have yet.

---

# Putting It All Together — Reading the Example Architecture

```
              Load Balancer
                 /    \
                /      \
          Backend 1   Backend 2
                \      /
                 \    /
                 Redis
                   |
              PostgreSQL
```

Walking through **why each piece is there**:

```
              Load Balancer
                 /    \
                /      \
          Backend 1   Backend 2
```
**Why:** solves availability (Backend 1 dying doesn't take the whole app down) and throughput
(two app servers handle roughly 2x the concurrent requests one could). This is horizontal
scaling applied to the application layer.

```
          Backend 1   Backend 2
                \      /
                 \    /
                 Redis
```
**Why:** both backend instances share ONE Redis cache — this matters because if each backend had
its own separate in-memory cache instead, a cache write from Backend 1 wouldn't be visible to
Backend 2, defeating the purpose (this is the same principle from Level 10's "session storage"
use case: shared state must live somewhere both servers can reach). Redis here reduces load on
Postgres and cuts latency for frequently-accessed data.

```
                 Redis
                   |
              PostgreSQL
```
**Why:** Postgres is the durable source of truth (Level 10's Postgres-vs-Redis distinction) — on
a cache miss, Redis defers to Postgres, gets the real answer, and caches it for next time.

### What this diagram is missing (and when you'd add it)

```
              Load Balancer
                 /    \
          Backend 1   Backend 2
                \      /
                 Redis
                   |
              PostgreSQL Primary
                /         \
      Read Replica 1   Read Replica 2     ← add when read load outgrows one Postgres instance

Add a Message Queue + Workers            ← add when some endpoints do slow, non-essential
   for slow background work                  work (emails, PDF generation, etc.)

Add a CDN in front of the Load Balancer  ← add when serving static assets / global users
                                              with latency-sensitive requests

Add Sharding                              ← add only once a SINGLE Postgres primary (even
                                              with replicas) can't hold/write the data volume
```

**The one-sentence takeaway:** every box in a system design diagram exists to solve one specific
failure mode or bottleneck from Part A (availability, latency, throughput, scalability,
consistency, reliability) — good system design is being able to look at a diagram and explain, in
one sentence each, exactly what breaks without that box.
