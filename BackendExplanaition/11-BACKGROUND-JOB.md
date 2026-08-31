# LEVEL 11 — Background Jobs (Why Celery Exists)

## The Core Problem

An HTTP request should return **fast**. If your endpoint does slow work inline (sending an email,
generating a PDF, calling a flaky third-party API, resizing an image), the client sits there
waiting, the request-handling thread/worker is tied up, and under load your whole API slows down
or times out.

### Without background jobs (BAD)

```
┌────────┐                                                       ┌────────┐
│ Client │──POST /register──▶  [API worker occupied the WHOLE time] ──▶│ Client │
└────────┘                                                       └────────┘
                │
                ▼
        Save user to DB          (fast, ~10ms)
                │
                ▼
        Send welcome email        (SLOW — network call to SMTP/SendGrid, ~5000ms)
                │
                ▼
        Return HTTP response      (client waited 5+ seconds total!)
```

```python
# BAD — blocks the request-response cycle on slow, non-essential work
@app.post("/register")
def register(user_data):
    user = create_user(user_data)
    send_welcome_email(user.email)     # <-- takes 5 seconds, user is just staring at a spinner
    return {"status": "registered"}
```

**Why this is bad:**
1. The user waits far longer than necessary — creating the account itself is fast; sending an
   email is not something they need to wait for.
2. If the email provider is slow or down, your *registration endpoint* fails too, even though
   registration itself succeeded.
3. Under load, every worker/thread stuck waiting on SMTP is a worker that can't handle other
   incoming requests — this can cascade into your whole API becoming unresponsive.

### With background jobs (GOOD)

```
┌────────┐                                              ┌────────┐
│ Client │──POST /register──▶ [fast: ~15ms] ────────────▶│ Client │  gets response immediately
└────────┘                          │
                                     ▼
                             Save user to DB
                                     │
                                     ▼
                     Hand off "send_email" task to Celery
                     (just pushes a message onto a queue — instant)
                                     │
                                     ▼
                          ┌─────────────────┐
                          │  Broker (Redis/  │   <- task sits here until a worker is free
                          │   RabbitMQ)       │
                          └────────┬────────┘
                                     │  (asynchronously, whenever)
                                     ▼
                          ┌─────────────────┐
                          │  Celery Worker    │  picks up the task, sends the email
                          └─────────────────┘
```

```python
# GOOD — hand off the slow part, respond to the client immediately
@app.post("/register")
def register(user_data):
    user = create_user(user_data)
    send_welcome_email_task.delay(user.email)   # <-- returns INSTANTLY, doesn't wait for the email
    return {"status": "registered"}             # client gets response in ~15ms
```

The email still gets sent — just not on the request's critical path. This is the entire reason
Celery (and background job systems generally) exist: **decouple "must happen before I respond"
from "must happen, but the user doesn't need to wait for it."**

---

## The Architecture: Broker → Celery → Workers

```
┌───────────────┐
│  Your App       │
│  (API server)    │
└────────┬────────┘
          │  task.delay(args)  — serializes the task call into a message
          ▼
┌───────────────────────────────┐
│         BROKER                  │   Redis or RabbitMQ
│   (message queue middleman)      │   Just stores messages until
│                                  │   a worker is ready to consume them
└────────┬───────────────────────┘
          │  worker polls / subscribes for new messages
          ▼
┌───────────────────────────────┐
│      CELERY WORKER(S)            │   Separate process(es), can run on
│  process 1  process 2  process 3 │   different machines from your API
└────────┬───────────────────────┘
          │  executes the actual task function
          ▼
   send_welcome_email(user.email)
          │
          ▼
┌───────────────────────────────┐
│   RESULT BACKEND (optional)      │   Redis/DB — stores task result/status
│   "task abc123: SUCCESS"          │   if you need to check on it later
└───────────────────────────────┘
```

### Why a separate Broker?

The broker (Redis or RabbitMQ) is just a **message queue** — its only job is to hold onto task
messages until a worker is available to process them. This decouples producers (your API) from
consumers (workers) completely:

- Your API doesn't need to know how many workers exist, or whether they're busy.
- Workers can be added/removed/scaled independently of the API.
- If all workers are temporarily down, tasks queue up and get processed once a worker comes back
  — nothing is lost (as long as the broker itself persists messages, which RabbitMQ does by
  default and Redis can with proper configuration).

---

## Setting It Up

```python
# celery_app.py
from celery import Celery

app = Celery(
    "myapp",
    broker="redis://localhost:6379/0",        # where tasks are queued
    backend="redis://localhost:6379/1",       # where results are stored (optional)
)

@app.task
def send_welcome_email(email):
    # this function runs on a WORKER process, not on the API process
    smtp_client.send(to=email, subject="Welcome!", body="Thanks for signing up")
    return "sent"
```

```python
# In your FastAPI/Django view
from celery_app import send_welcome_email

@app.post("/register")
def register(user_data):
    user = create_user(user_data)
    send_welcome_email.delay(user.email)     # queues the task, returns immediately
    return {"status": "registered"}
```

Running a worker (a separate long-running process, usually its own container/server):
```bash
celery -A celery_app worker --loglevel=info
```

---

## Core Concepts

### Task

A task is just a Python function decorated with `@app.task`. Calling `.delay()` (or `.apply_async()`
for more control) doesn't run it locally — it **serializes the function name + arguments** into a
message and pushes that message onto the broker.

```python
@app.task
def resize_image(image_path, width, height):
    ...

resize_image.delay("photo.jpg", 800, 600)                  # fire and forget
result = resize_image.apply_async(args=["photo.jpg", 800, 600], countdown=10)  # run in 10s
```

### Queue

Tasks can be routed to **named queues**, letting you dedicate specific workers to specific kinds
of work (e.g., keep a burst of slow "generate PDF" jobs from starving quick "send SMS" jobs).

```python
app.conf.task_routes = {
    "myapp.tasks.send_email": {"queue": "emails"},
    "myapp.tasks.generate_report": {"queue": "reports"},
}
```

```bash
# Dedicated workers per queue — reports won't block emails
celery -A celery_app worker -Q emails --loglevel=info
celery -A celery_app worker -Q reports --loglevel=info
```

```
                  ┌─────────────┐
   task───────────▶│ "emails" queue │───────▶ Worker Pool A (fast, small tasks)
                  └─────────────┘

                  ┌─────────────┐
   task───────────▶│ "reports" queue │──────▶ Worker Pool B (slow, heavy tasks)
                  └─────────────┘
```

### Acknowledgement (ack)

When a worker picks up a task message from the broker, it needs to tell the broker "I've got it
and I'm done" — this is the **acknowledgement**. It's what prevents a task from being silently
lost if a worker crashes mid-execution.

```
Worker picks up task ──▶ starts processing
        │
        ├── SUCCESS ──▶ sends ACK ──▶ broker permanently removes the message
        │
        └── Worker CRASHES before finishing (no ACK sent)
                   │
                   ▼
        Broker (if using `acks_late=True`) re-delivers the task
        to another worker — nothing is silently lost
```

```python
@app.task(acks_late=True)     # only ack AFTER the task finishes successfully
def critical_task(order_id):
    process_payment(order_id)
```

By default Celery acknowledges tasks *as soon as they're received* (before execution) — fast, but
if the worker dies mid-task, that task is lost. `acks_late=True` trades a small risk of
double-execution (if the worker dies right after finishing but before sending the ack) for a
guarantee that a crashed worker never silently drops a task — you should design tasks to be
**idempotent** (safe to run twice) if you use this.

### Retry

Transient failures (a third-party API timing out, a momentary network blip) shouldn't permanently
fail a task — Celery can automatically retry with backoff.

```python
@app.task(bind=True, max_retries=3, default_retry_delay=10)
def call_flaky_api(self, payload):
    try:
        return external_api.post(payload)
    except ConnectionError as exc:
        raise self.retry(exc=exc)   # retries up to 3 times, waiting 10s between attempts

# Exponential backoff variant
@app.task(bind=True, max_retries=5)
def call_flaky_api_backoff(self, payload):
    try:
        return external_api.post(payload)
    except ConnectionError as exc:
        raise self.retry(exc=exc, countdown=2 ** self.request.retries)  # 1s, 2s, 4s, 8s, 16s
```

```
Attempt 1 ──FAIL──▶ wait 1s ──▶ Attempt 2 ──FAIL──▶ wait 2s ──▶ Attempt 3 ──FAIL──▶
wait 4s ──▶ Attempt 4 ──SUCCESS──▶ done
```

### Periodic Task (Celery Beat)

For "run this every N minutes/hours/at a specific time" jobs (cleanup scripts, daily reports,
cache warming), **Celery Beat** is a separate scheduler process that pushes tasks onto the queue
on a schedule — it doesn't run the tasks itself, workers still do that.

```python
from celery.schedules import crontab

app.conf.beat_schedule = {
    "cleanup-expired-sessions-every-hour": {
        "task": "myapp.tasks.cleanup_expired_sessions",
        "schedule": crontab(minute=0),          # every hour, on the hour
    },
    "send-daily-report": {
        "task": "myapp.tasks.send_daily_report",
        "schedule": crontab(hour=8, minute=0),  # every day at 8:00 AM
    },
}
```

```bash
celery -A celery_app beat --loglevel=info      # the scheduler
celery -A celery_app worker --loglevel=info    # the executor (needs to run too)
```

```
┌──────────────┐   "it's 8:00am, run send_daily_report"   ┌───────────┐
│ Celery Beat    │ ─────────────────────────────────────▶ │  Broker     │
│ (scheduler,     │                                          └─────┬─────┘
│  runs on a timer)│                                                │
└──────────────┘                                                ▼
                                                          ┌───────────┐
                                                          │  Worker     │  actually executes it
                                                          └───────────┘
```

### Dead Letter / Failed Tasks

When a task exhausts all its retries (or fails with an exception you don't retry on), it needs to
go *somewhere* so it isn't silently forgotten. This is conceptually the "dead letter queue" idea
from message-queue systems generally.

```python
@app.task(bind=True, max_retries=3)
def process_payment(self, order_id):
    try:
        charge_card(order_id)
    except PaymentError as exc:
        raise self.retry(exc=exc)

@app.task(bind=True)
def process_payment_with_fallback(self, order_id):
    try:
        charge_card(order_id)
    except PaymentError as exc:
        if self.request.retries >= self.max_retries:
            # all retries exhausted — don't let this vanish silently
            log_to_dead_letter_table(order_id, str(exc))
            notify_ops_team(order_id, exc)
        raise self.retry(exc=exc, max_retries=3)
```

```
┌───────┐  fail  ┌───────┐  fail  ┌───────┐  fail  ┌────────────────┐
│ try 1  │──────▶│ try 2  │──────▶│ try 3  │──────▶│ DEAD LETTER      │
└───────┘        └───────┘        └───────┘        │ (logged, alerted, │
                                                     │  needs human review)│
                                                     └────────────────┘
```

RabbitMQ has native Dead Letter Exchange support for this; with Redis-as-broker you typically
implement this pattern yourself (as above) — log failed tasks to a table/separate Redis list so
nothing fails silently into the void.

---

## Full Picture: Redis/RabbitMQ → Celery → Workers

```
┌─────────────────────────────────────────────────────────────────────┐
│                          YOUR APPLICATION                              │
│              (FastAPI / Django — the task PRODUCER)                    │
└──────────────────────────────┬────────────────────────────────────────┘
                                │  task.delay(...)
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      BROKER (Redis or RabbitMQ)                         │
│   Holds queued task messages. Doesn't execute anything itself.           │
│   Queues: [emails] [reports] [default] ...                               │
└──────────────────────────────┬────────────────────────────────────────┘
                                │  workers pull/subscribe
                ┌───────────────┼───────────────┐
                ▼               ▼               ▼
        ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
        │  Worker 1     │ │  Worker 2     │ │  Worker 3     │   <- separate processes,
        │  (executes     │ │  (executes     │ │  (executes     │      can scale independently,
        │   task code)    │ │   task code)    │ │   task code)    │      can run on other machines
        └──────┬──────┘ └──────┬──────┘ └──────┬──────┘
                │               │               │
                ▼               ▼               ▼
        ┌─────────────────────────────────────────────┐
        │        RESULT BACKEND (optional, Redis/DB)      │
        │   task_id → SUCCESS/FAILURE + return value        │
        └─────────────────────────────────────────────┘

                          ┌───────────────┐
                          │  Celery Beat    │  (separate process, optional)
                          │  pushes SCHEDULED│  "every hour, queue cleanup_task"
                          │  tasks onto the   │
                          │  broker on a timer │
                          └───────────────┘
```

**Redis vs RabbitMQ as the broker:** Redis is simpler to run (you probably already have it for
caching) and fast, but has weaker delivery guarantees out of the box. RabbitMQ is a dedicated,
more feature-rich message broker (native dead-lettering, more delivery guarantee options,
better handling of very large queues) at the cost of one more piece of infrastructure to run.
Many teams start with Redis and move to RabbitMQ only if they hit its limits.

---

## Mental Model Summary

| Concept | What it is | Analogy |
|---|---|---|
| **Broker** | Message queue holding pending tasks | The waiting room / mailbox |
| **Task** | A function + its arguments, serialized into a message | A written work order |
| **Worker** | A process that pulls tasks and executes them | The employee doing the job |
| **Queue** | A named channel tasks can be routed to | Different departments' inboxes |
| **Ack** | Confirmation a task was (successfully) handled | Signing off on completed work |
| **Retry** | Re-attempting a failed task | Redoing a job that went wrong |
| **Beat** | Scheduler that queues tasks on a timer | An alarm clock that hands out work orders |
| **Dead letter** | Where permanently-failed tasks end up | The "needs manual review" pile |

**The one-sentence takeaway:** Celery exists so your API can say "someone will handle this soon"
instead of "I will handle this right now, please wait" — turning slow, non-essential work from a
blocker in the request path into an independently-scalable background process.
