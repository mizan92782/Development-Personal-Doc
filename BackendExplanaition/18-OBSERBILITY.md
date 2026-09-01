# LEVEL 18 — Observability Deep Dive

## Why Observability Exists

Once your system has multiple servers, a cache, a database, and background workers, "just look at
the code" stops being enough to answer "why did this break?" — the failure could be in any of a
dozen moving parts, most of which you can't attach a debugger to in production. Observability is
the practice of instrumenting your system so that, when something goes wrong, you can answer
*what happened, where, and why* using data the system already emitted — instead of guessing.

```
Debugging locally:        set a breakpoint, step through code, inspect variables
Debugging in production:  NO breakpoints. Only what you already logged/measured/traced
                           BEFORE the incident happened. This is why observability has
                           to be built in ahead of time, not added after something breaks.
```

---

## The Three Pillars

```
┌───────────────────────────────────────────────────────────────────┐
│                                                                       │
│   LOGGING              METRICS              TRACING                    │
│   "what happened,       "how much/how many,   "where did THIS specific   │
│    in detail, at         over time"             request's time go,        │
│    this moment"                                 across every service       │
│                                                  it touched"                │
│                                                                       │
│   discrete events        numeric time-series    a request's full journey    │
│   ("user 42 login         ("CPU: 45%,             (API → cache → DB →        │
│    failed: bad password")  requests/sec: 230,      queue → back)              │
│                             p99 latency: 800ms)                             │
└───────────────────────────────────────────────────────────────────┘
```

### Logging

```python
import logging

logger = logging.getLogger(__name__)

def process_order(order_id):
    logger.info(f"Processing order {order_id}")
    try:
        charge_payment(order_id)
        logger.info(f"Order {order_id} payment succeeded")
    except PaymentError as e:
        logger.error(f"Order {order_id} payment FAILED: {e}", exc_info=True)
        raise
```

**Structured logging** (JSON instead of free text) is what makes logs actually searchable/
aggregatable at scale — instead of grepping for a substring, you query fields.

```python
import structlog
logger = structlog.get_logger()

logger.info("order_processed", order_id=42, amount=99.99, status="success")
# outputs: {"event": "order_processed", "order_id": 42, "amount": 99.99, "status": "success", "timestamp": "..."}
```

### Metrics

Numeric measurements sampled over time — cheap to store and query even at huge volume, because
they're just numbers, not full text blobs.

```python
from prometheus_client import Counter, Histogram

REQUEST_COUNT = Counter("http_requests_total", "Total HTTP requests", ["method", "endpoint", "status"])
REQUEST_LATENCY = Histogram("http_request_duration_seconds", "Request latency", ["endpoint"])

@REQUEST_LATENCY.labels(endpoint="/users").time()
def get_users():
    REQUEST_COUNT.labels(method="GET", endpoint="/users", status="200").inc()
    ...
```

### Tracing

Follows **one request** across every service/component it touches, showing exactly where time
was spent — invaluable once "which one of my 5 microservices is slow" becomes the question.

```
Trace ID: abc-123  (the SAME id is passed along through every hop)

├─ API Gateway              [0ms   ────▶ 250ms]  total request time
│   ├─ Auth Service          [0ms  ──▶ 20ms]      "checking token"
│   ├─ Orders Service         [20ms ──▶ 230ms]     "fetching order" ← most of the time is HERE
│   │   └─ Postgres query      [30ms ──▶ 220ms]     "SELECT ... slow query!"
│   └─ Response serialization   [230ms ──▶ 250ms]
```

```python
# OpenTelemetry example
from opentelemetry import trace
tracer = trace.get_tracer(__name__)

def get_order(order_id):
    with tracer.start_as_current_span("get_order"):
        with tracer.start_as_current_span("db_query"):
            return db.query(f"SELECT * FROM orders WHERE id={order_id}")
```

### Monitoring vs Alerting

**Monitoring** is continuously collecting and visualizing the above (dashboards you can look at).
**Alerting** is defining rules on top of that data that page a human when something crosses a
threshold — monitoring is passive/observational, alerting is active/actionable.

```
Monitoring: "here's a graph of error rate over the last 24 hours" (you have to look at it)
Alerting:   "error rate exceeded 5% for 5 minutes → send a page to on-call NOW"
```

```yaml
# Prometheus alerting rule
groups:
  - name: api_alerts
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
        for: 5m
        annotations:
          summary: "Error rate above 5% for 5 minutes"
```

---

## The Standard Stack: Prometheus + Grafana + Loki

```
┌─────────────────────────────────────────────────────────────┐
│                        Application                              │
│                                                                    │
│   ├── Logs   ──────────────────────▶  Loki       (log storage/search)│
│   ├── Metrics ─────────────────────▶  Prometheus  (metrics storage)   │
│   └── (both queried by) ──────────▶  Grafana     (dashboards, alerts)  │
└─────────────────────────────────────────────────────────────┘
```

- **Prometheus** — pulls (scrapes) numeric metrics from your app at regular intervals and stores
  them as time-series data; has its own query language (PromQL) for aggregating/graphing them.
- **Loki** — like Prometheus but for logs; indexes only metadata (labels) rather than full text,
  making it cheap to run at scale compared to full-text log indexing (e.g., Elasticsearch).
- **Grafana** — the unified visualization layer on top of both (and other data sources) — one
  place to build dashboards mixing metrics and logs, and to configure alerts.

```
Application exposes /metrics endpoint (Prometheus scrapes this every N seconds)
Application writes logs to stdout (a log-shipping agent like Promtail forwards these to Loki)
                                            │
                              ┌─────────────┴─────────────┐
                              ▼                             ▼
                        Prometheus                        Loki
                       (metrics DB)                    (log storage)
                              │                             │
                              └──────────────┬──────────────┘
                                             ▼
                                        Grafana
                              (dashboards querying BOTH — e.g.
                               "show error rate AND the actual
                               error logs for that time window,
                               side by side")
```

---

# The Production Debugging Playbook

For each question: what tools to check, in what order, and what you're looking for.

## 1. Why is the API slow?

```
Step 1: Check METRICS first — is it slow everywhere, or one endpoint?
   → Grafana dashboard: p50/p95/p99 latency, broken down by endpoint
   → "everywhere" suggests infra-wide (DB, network); "one endpoint" suggests that code path

Step 2: Check TRACING — for a slow request, where did the time actually go?
   → Is it the DB query? An external API call? Serialization? Queueing at the app server?

Step 3: Check LOGS around that timeframe — anything unusual logged (retries, timeouts, warnings)?

Step 4: Check the DEPENDENCY layer — is Redis/Postgres/an external API itself slow right now?
   → If Redis/DB latency spiked at the same time, the bottleneck is there, not your app code

Step 5: Check for RESOURCE exhaustion on the app server itself (see CPU/memory questions below)
```

```bash
# Quick manual checks
curl -w "@curl-format.txt" -o /dev/null -s https://yourapp.com/api/endpoint   # time the request
ss -tulpn | grep 8000                          # is the app server even accepting connections fast?
```

**Common root causes:** N+1 database queries, missing indexes, a slow downstream API call with no
timeout, cache misses on a hot key, connection pool exhaustion, GC pauses, too few worker
processes for the current load.

## 2. Why did the server return 500?

```
Step 1: Check LOGS immediately — a 500 should always have a stack trace logged. Find it.
   → grep/Loki query for the request's timestamp + trace ID (if you have tracing)

Step 2: Identify the EXCEPTION TYPE — this usually tells you the category immediately:
   → DatabaseError/OperationalError → DB connection/query issue
   → KeyError/AttributeError        → unhandled edge case in your code, bad assumption about data
   → Timeout                        → a downstream dependency didn't respond in time
   → MemoryError                    → resource exhaustion (see memory question below)

Step 3: Reproduce — can you trigger it with the same input in a non-prod environment?

Step 4: Check if it's WIDESPREAD or ISOLATED
   → Metrics: what % of requests are 500ing? All of them (something is fundamentally broken,
     e.g. DB is down) or a tiny %, correlated with specific input/user/endpoint (an edge-case bug)?
```

```python
# Make sure 500s are ALWAYS logged with full context — this is the #1 prerequisite
@app.exception_handler(Exception)
async def unhandled_exception_handler(request, exc):
    logger.error(
        "unhandled_exception",
        path=request.url.path,
        method=request.method,
        exc_info=True,   # includes the full traceback
    )
    return JSONResponse(status_code=500, content={"detail": "Internal server error"})
```

**Common root causes:** unhandled exceptions on edge-case input, a downstream service being down,
a database connection pool exhausted, a bug introduced in the latest deploy (check: did this
start right after a deploy?).

## 3. Why is CPU high?

```
Step 1: WHICH process is using the CPU?
   → top / htop  — is it your app, the database, or something unexpected entirely?

Step 2: Is it high because of LEGITIMATE increased load, or an inefficiency?
   → Check request-rate metrics for the same time window — did traffic actually go up?
   → If traffic is flat but CPU spiked, something changed in the CODE, not the load

Step 3: PROFILE the actual hot code path
   → py-spy (can attach to a running Python process without restarting it!)
```

```bash
py-spy top --pid 1234          # live view of which functions are consuming CPU right now
py-spy dump --pid 1234           # one-time stack trace snapshot of all threads
```

```
Step 4: Common CPU culprits to check for specifically:
   - An inefficient algorithm/loop that's O(n²) hit with more data than it was tested against
   - Regex catastrophic backtracking on unexpected input
   - JSON/serialization of huge payloads
   - A retry loop with no backoff, hammering a failing dependency
   - Too much synchronous CPU-bound work inside an async event loop (see Level 8!),
     blocking OTHER requests too, which can look like "everything is slow" not just "CPU is high"
```

## 4. Why is memory increasing (a "memory leak")?

```
Step 1: Is it a LEAK (grows forever, never comes back down) or just normal usage under load
   (grows with traffic, comes back down when traffic drops)?
   → Grafana: graph memory over a long window (hours/days) — does it plateau or climb indefinitely?

Step 2: In Python specifically, "leaks" are usually about things that are still REFERENCED
   somewhere, so the garbage collector can never free them:
   - A global list/dict that things get appended to but never cleared
   - Circular references involving objects with __del__ (harder for the GC to collect)
   - An unbounded cache (e.g., a plain dict used as a cache with no eviction policy/TTL)
   - Database connections/file handles never closed, accumulating over time
   - Event listeners/callbacks registered repeatedly without being unregistered

Step 3: TOOLS to find WHAT is accumulating
```

```python
import tracemalloc
tracemalloc.start()

# ... let the app run for a while, handling requests ...

snapshot = tracemalloc.take_snapshot()
top_stats = snapshot.statistics("lineno")
for stat in top_stats[:10]:
    print(stat)   # shows exactly which lines of code are holding the most memory
```

```bash
py-spy dump --pid 1234 --locals    # inspect live object state in a running process
```

```
Step 4: Check for the classic mutable-default-argument bug from Level 8 —
   a module-level or default-argument list/dict that keeps growing across requests
   is one of the most common accidental "leaks" in Python web apps.
```

## 5. Why is the DB slow?

```
Step 1: Is ONE query slow, or is the whole database under load?
   → Postgres: check pg_stat_activity for currently running queries and their duration

Step 2: Find the slow query specifically
```

```sql
-- Currently running queries, ordered by how long they've been running
SELECT pid, now() - query_start AS duration, query
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY duration DESC;

-- Enable slow query logging to catch this automatically going forward
-- postgresql.conf: log_min_duration_statement = 500   (log anything over 500ms)
```

```
Step 3: For a specific slow query, check its EXECUTION PLAN
```

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 42 AND status = 'pending';
```

```
Look for:
  "Seq Scan" on a large table  →  missing an index (should likely be "Index Scan")
  Nested Loop with huge row counts → possible N+1 pattern or bad join
  High "actual time" vs "estimated" → stale table statistics (run ANALYZE)

Step 4: Check for RESOURCE contention at the DB level, not just query logic
   - Connection pool exhausted (too many app instances/threads opening connections)
   - Lock contention — one long transaction holding a lock that blocks others
   - Disk I/O saturated (check iostat) — especially after data grew past what fits in RAM cache
   - Replication lag, if reading from a replica (Level 17) — it may just be behind
```

```bash
# Check for blocking locks
SELECT blocked_locks.pid AS blocked_pid, blocking_locks.pid AS blocking_pid
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_locks blocking_locks
  ON blocking_locks.locktype = blocked_locks.locktype
  AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
WHERE NOT blocked_locks.granted;
```

**Common root causes:** missing index, N+1 queries from the ORM, a long-running transaction
holding locks, connection pool too small (or too large — overwhelming the DB), table statistics
out of date, a sudden data growth pattern that outgrew the current query plan's assumptions.

---

## More Questions to Build Your Debugging Instincts

### Why are requests timing out (but not erroring)?
Distinguish between: the app never received the request (network/load balancer issue — check LB
logs and health checks), the app received it but never responded (check if the request is stuck
waiting on a downstream call with no timeout configured — a classic async footgun from Level 8:
a blocking call inside an async handler can freeze the whole event loop, making *other* unrelated
requests time out too, not just the slow one).

### Why did throughput suddenly drop even though CPU/memory look normal?
Check connection pool limits (app-to-DB, app-to-Redis) — if the pool is exhausted, new requests
queue up waiting for a connection, tanking throughput without any single resource looking
"maxed out." Also check for a downstream dependency that got slower (its latency eating into your
worker availability) even if it hasn't started erroring yet.

### Why is disk I/O high?
Check what's writing: verbose logging at DEBUG level in production, a database doing heavy
writes/WAL flushing, swap usage (if memory is under pressure, the OS starts swapping to disk,
which is catastrophically slower than RAM and often the REAL cause of a mysterious slowdown that
initially looks CPU or DB related).

### Why did error rate spike right after a deploy?
Check the diff between the last known-good version and the new one first — this is almost always
the fastest path to root cause. Check for: a missing environment variable/config in the new
environment, a database migration that hasn't finished running yet (code expects a new column
that doesn't exist in prod yet), a dependency version bump with a breaking change.

### Why is one server in the load-balanced pool behaving differently from the others?
Check for configuration drift (did a deploy only partially roll out?), a stuck/zombie process
that a restart would fix, that server having a different resource allocation, or it holding a
larger share of long-lived connections/sessions due to load-balancer stickiness.

### Why is the message queue backlog growing?
Compare the rate tasks are being PRODUCED vs the rate they're being CONSUMED (Prometheus metrics
for both) — either producers sped up (traffic spike) or consumers slowed down (workers crashing,
a task type suddenly taking much longer, not enough worker instances for current load).

### Why did memory usage jump suddenly (not gradually)?
Look for a specific large batch job, a huge unpaginated query loading an entire table into
memory at once, or a large file/upload being processed entirely in memory instead of streamed.

### Why are we seeing intermittent connection errors to the database?
Check connection pool exhaustion again, but also: DB server restarts/failovers, network blips
between app and DB (especially across availability zones/regions), and DB-side connection limits
(`max_connections` in Postgres) being hit by the total across all app instances combined.

### Why does the same endpoint work fine in staging but is slow/broken in production?
Check for data volume differences (a query that's fast on a 1,000-row staging table can be
catastrophic on a 50-million-row production table if it's missing an index), differing
configuration (connection pool sizes, cache settings, feature flags), and load differences
(concurrency exposes race conditions that never show up with a single test user).

```
General debugging instinct, restated as one flow:

    Something's wrong
          │
          ▼
    Is it EVERYWHERE or ISOLATED?  (one endpoint? one server? one user? all traffic?)
          │
          ▼
    Did something CHANGE recently?  (deploy, config change, traffic pattern shift, data growth)
          │
          ▼
    Follow the request: Logs (what happened) → Metrics (how much/how often) →
    Tracing (where exactly did the time/failure occur) → Dependency-level checks
    (DB/cache/queue/external API health) → Resource-level checks (CPU/memory/disk/network)
```

**The one-sentence takeaway:** observability isn't about collecting more data for its own sake —
it's about having, ahead of time, exactly the logs/metrics/traces you'll need to answer "why" the
one time production breaks at 2am, without needing to reproduce the problem from scratch.
