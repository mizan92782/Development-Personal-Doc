# Redis সম্পূর্ণ টিউটোরিয়াল (বাংলায়) — FastAPI সহ

এই ডকুমেন্টে Redis-এর প্রতিটা গুরুত্বপূর্ণ command, type এবং function বাংলায় ব্যাখ্যা করা হয়েছে — সাথে FastAPI দিয়ে কিভাবে ব্যবহার করতে হয় তার কোড উদাহরণ। বিশেষ করে rate limiting এবং সাধারণ caching use-case এর উপর focus করা হয়েছে।

---

## 📦 শুরুর আগে — Setup

FastAPI থেকে Redis ব্যবহার করতে হলে `redis` Python লাইব্রেরি লাগবে (async সাপোর্ট সহ)।

```bash
pip install redis fastapi uvicorn
```

একটা reusable Redis connection বানিয়ে নিই, যেটা পুরো টিউটোরিয়ালের সব উদাহরণে ব্যবহার হবে:

```python
# redis_client.py
import redis.asyncio as redis

# Redis সার্ভারের সাথে connection তৈরি করা হচ্ছে
redis_client = redis.Redis(
    host="localhost",
    port=6379,
    db=0,
    decode_responses=True  # bytes এর বদলে string পাওয়ার জন্য
)
```

```python
# main.py
from fastapi import FastAPI
from redis_client import redis_client

app = FastAPI()
```

`decode_responses=True` দিলে Redis থেকে যা ফেরত আসবে সেটা সরাসরি Python string হবে, `b"value"` এর মতো bytes হবে না। তাই প্রতিটা উদাহরণে আমরা এটা ধরে নিয়েই এগোবো।

---

## 🧠 ১. String Operations

👉 **এটা কী:** Redis-এর সবচেয়ে সাধারণ data type। এক **key** এর সাথে এক **value** (string, number, JSON string ইত্যাদি) সংরক্ষণ করা হয়। সাধারণ caching, counter, flag রাখার জন্য ব্যবহার হয়।

### `set(key, value)`
**কাজ:** একটা key-তে value সংরক্ষণ করে। আগে থেকে key থাকলে overwrite হয়ে যায়।

```python
@app.post("/string/set")
async def set_value():
    # "username" নামে key-তে "Rafiq" ভ্যালু সেভ হচ্ছে
    await redis_client.set("username", "Rafiq")
    return {"message": "Value saved"}
```

### `get(key)`
**কাজ:** একটা key-এর value ফেরত দেয়। key না থাকলে `None` রিটার্ন করে।

```python
@app.get("/string/get")
async def get_value():
    value = await redis_client.get("username")
    return {"username": value}
```

### `delete(key)` / `del(key)`
**কাজ:** নির্দিষ্ট key এবং তার value ডিলিট করে দেয়। Python-এ `del` keyword হওয়ায় Redis লাইব্রেরিতে এই function-এর নাম `delete()`।

```python
@app.delete("/string/delete")
async def delete_value():
    deleted_count = await redis_client.delete("username")
    return {"deleted": deleted_count}  # 1 মানে delete হয়েছে, 0 মানে key ছিলোই না
```

### `exists(key)`
**কাজ:** key আছে কিনা চেক করে। থাকলে `1`, না থাকলে `0` রিটার্ন করে।

```python
@app.get("/string/exists")
async def check_exists():
    result = await redis_client.exists("username")
    return {"exists": bool(result)}
```

### `expire(key, seconds)`
**কাজ:** একটা নির্দিষ্ট সময় পর key নিজে থেকেই delete হয়ে যাবে — যেমন session timeout বা temporary cache।

```python
@app.post("/string/expire")
async def set_with_expiry():
    await redis_client.set("otp:1234", "5566")
    # ৬০ সেকেন্ড পরে এই key নিজে থেকেই মুছে যাবে
    await redis_client.expire("otp:1234", 60)
    return {"message": "OTP set for 60 seconds"}
```

### `ttl(key)`
**কাজ:** কোনো key এর expiry হতে আর কত সেকেন্ড বাকি আছে সেটা জানায়। key না থাকলে `-2`, expiry সেট না থাকলে `-1` রিটার্ন করে।

```python
@app.get("/string/ttl")
async def check_ttl():
    seconds_left = await redis_client.ttl("otp:1234")
    return {"ttl_seconds": seconds_left}
```

### `incr(key)`
**কাজ:** value-কে সংখ্যা হিসেবে ধরে ১ বাড়িয়ে দেয়। **rate limiting-এর জন্য সবচেয়ে গুরুত্বপূর্ণ command।**

```python
@app.post("/string/incr")
async def increment_counter():
    # key না থাকলে 0 ধরে নিয়ে +1 করে 1 বানাবে
    new_value = await redis_client.incr("api_call_count")
    return {"count": new_value}
```

### `decr(key)`
**কাজ:** value-কে সংখ্যা হিসেবে ধরে ১ কমিয়ে দেয়। যেমন — available quota কমানো।

```python
@app.post("/string/decr")
async def decrement_counter():
    new_value = await redis_client.decr("api_call_count")
    return {"count": new_value}
```

> 💡 **Rate Limiting উদাহরণ (String দিয়ে):**
> ```python
> @app.get("/rate-limited-endpoint")
> async def rate_limited(user_id: str):
>     key = f"rate_limit:{user_id}"
>     current = await redis_client.incr(key)
>     if current == 1:
>         # প্রথমবার রিকোয়েস্ট করলে ৬০ সেকেন্ডের window সেট হবে
>         await redis_client.expire(key, 60)
>     if current > 10:
>         return {"error": "Too many requests, try again later"}
>     return {"message": "Request allowed", "count": current}
> ```

---

## 🧠 ২. Hash Operations (dict-like)

👉 **এটা কী:** একটা key-এর ভেতরে অনেকগুলো field-value pair রাখা যায় — অনেকটা Python dictionary বা একটা object-এর মতো। যেমন একজন ইউজারের সব তথ্য (নাম, ইমেইল, বয়স) এক key-এর নিচে রাখা।

### `hset(name, key, value)`
**কাজ:** একটা hash-এর ভেতরে নির্দিষ্ট field-এ value বসায়।

```python
@app.post("/hash/set")
async def set_user_field():
    # "user:1" নামে hash-এ "name" field-এ value বসছে
    await redis_client.hset("user:1", "name", "Karim")
    await redis_client.hset("user:1", "email", "karim@example.com")
    return {"message": "User fields saved"}
```

### `hget(name, key)`
**কাজ:** hash থেকে একটা নির্দিষ্ট field-এর value বের করে।

```python
@app.get("/hash/get")
async def get_user_field():
    name = await redis_client.hget("user:1", "name")
    return {"name": name}
```

### `hgetall(name)`
**কাজ:** পুরো hash-এর সব field-value একসাথে dictionary আকারে ফেরত দেয়।

```python
@app.get("/hash/all")
async def get_all_user_fields():
    user_data = await redis_client.hgetall("user:1")
    return {"user": user_data}  # {"name": "Karim", "email": "karim@example.com"}
```

### `hdel(name, key)`
**কাজ:** hash থেকে নির্দিষ্ট একটা field মুছে ফেলে, পুরো hash না।

```python
@app.delete("/hash/delete-field")
async def delete_user_field():
    await redis_client.hdel("user:1", "email")
    return {"message": "Email field removed"}
```

---

## 🧠 ৩. List Operations (queue system)

👉 **এটা কী:** ordered list of items — অনেকটা Python list-এর মতো। Queue (FIFO) বা Stack (LIFO) বানানোর জন্য ব্যবহার হয়, যেমন background job queue।

### `lpush(key, value)`
**কাজ:** list-এর শুরুতে (left side) নতুন item যোগ করে।

```python
@app.post("/list/lpush")
async def push_to_front():
    await redis_client.lpush("task_queue", "send_email")
    return {"message": "Task added to front"}
```

### `rpush(key, value)`
**কাজ:** list-এর শেষে (right side) নতুন item যোগ করে। সাধারণত queue-তে নতুন কাজ যোগ করতে এটা বেশি ব্যবহার হয়।

```python
@app.post("/list/rpush")
async def push_to_back():
    await redis_client.rpush("task_queue", "generate_report")
    return {"message": "Task added to back"}
```

### `lpop(key)`
**কাজ:** list-এর শুরু থেকে একটা item সরিয়ে সেটা রিটার্ন করে।

```python
@app.post("/list/lpop")
async def pop_from_front():
    task = await redis_client.lpop("task_queue")
    return {"task": task}
```

### `rpop(key)`
**কাজ:** list-এর শেষ থেকে একটা item সরিয়ে সেটা রিটার্ন করে।

```python
@app.post("/list/rpop")
async def pop_from_back():
    task = await redis_client.rpop("task_queue")
    return {"task": task}
```

### `lrange(key, start, end)`
**কাজ:** list-এর একটা নির্দিষ্ট অংশ (range) পড়ে, পুরোটা delete না করে। পুরো list দেখতে `0, -1` দিতে হয়।

```python
@app.get("/list/range")
async def get_all_tasks():
    tasks = await redis_client.lrange("task_queue", 0, -1)
    return {"tasks": tasks}
```

---

## 🧠 ৪. Set Operations (unique items)

👉 **এটা কী:** এমন একটা collection যেখানে কোনো item একবারের বেশি থাকতে পারে না (unique), এবং এর কোনো নির্দিষ্ট order থাকে না। যেমন — কোনো পোস্টে কারা লাইক দিয়েছে তার তালিকা।

### `sadd(key, value)`
**কাজ:** set-এ একটা unique item যোগ করে। আগে থেকে থাকলে দ্বিতীয়বার যোগ হবে না।

```python
@app.post("/set/add")
async def add_to_set():
    await redis_client.sadd("post:1:likes", "user_5")
    return {"message": "Like added"}
```

### `srem(key, value)`
**কাজ:** set থেকে নির্দিষ্ট item মুছে দেয়।

```python
@app.delete("/set/remove")
async def remove_from_set():
    await redis_client.srem("post:1:likes", "user_5")
    return {"message": "Like removed"}
```

### `smembers(key)`
**কাজ:** set-এর সব member একসাথে ফেরত দেয়।

```python
@app.get("/set/members")
async def get_all_members():
    likes = await redis_client.smembers("post:1:likes")
    return {"likes": list(likes)}
```

### `sismember(key, value)`
**কাজ:** নির্দিষ্ট item set-এ আছে কিনা চেক করে (True/False)।

```python
@app.get("/set/check")
async def check_member():
    has_liked = await redis_client.sismember("post:1:likes", "user_5")
    return {"has_liked": bool(has_liked)}
```

---

## 🧠 ৫. Sorted Set (ZSET) 🔥

👉 **এটা কী:** Set-এর মতোই unique items রাখে, কিন্তু প্রতিটা item-এর সাথে একটা **score** (সংখ্যা) যুক্ত থাকে, যেটা দিয়ে items automatically sorted থাকে। **এই কারণেই এটা rate limiting এবং leaderboard/ranking system-এর জন্য সবচেয়ে শক্তিশালী tool** — score হিসেবে timestamp ব্যবহার করে sliding-window rate limiting বানানো যায়।

### `zadd(key, {member: score})`
**কাজ:** একটা item-কে নির্দিষ্ট score সহ sorted set-এ যোগ করে।

```python
import time

@app.post("/zset/add")
async def add_request_timestamp(user_id: str):
    current_time = time.time()
    # member হিসেবে timestamp string, score হিসেবেও timestamp ব্যবহার করা হচ্ছে
    await redis_client.zadd(f"requests:{user_id}", {str(current_time): current_time})
    return {"message": "Request logged"}
```

### `zrem(key, member)`
**কাজ:** sorted set থেকে নির্দিষ্ট member মুছে দেয়।

```python
@app.delete("/zset/remove")
async def remove_member():
    await redis_client.zrem("requests:user_1", "1718870400.123")
    return {"message": "Removed"}
```

### `zremrangebyscore(key, min, max)`
**কাজ:** একটা নির্দিষ্ট score range-এর সব item একসাথে মুছে দেয়। **Sliding window rate limiting-এর মূল hero এই function** — পুরনো (expired) timestamp-গুলো এক কমান্ডেই সাফ করে দেওয়া যায়।

```python
@app.post("/zset/cleanup")
async def cleanup_old_requests(user_id: str):
    current_time = time.time()
    window_start = current_time - 60  # গত ৬০ সেকেন্ডের আগের সব মুছে ফেলা হবে
    await redis_client.zremrangebyscore(f"requests:{user_id}", 0, window_start)
    return {"message": "Old requests cleaned"}
```

### `zcard(key)`
**কাজ:** sorted set-এ মোট কতগুলো item আছে তার সংখ্যা জানায়। rate limiting-এ "এই window-তে কতবার রিকোয়েস্ট হয়েছে" বুঝতে এটা ব্যবহার হয়।

```python
@app.get("/zset/count")
async def count_requests(user_id: str):
    total = await redis_client.zcard(f"requests:{user_id}")
    return {"total_requests": total}
```

### `zrange(key, start, end)`
**কাজ:** position (index) অনুযায়ী range-এর item গুলো ফেরত দেয় (score অনুযায়ী sorted থাকে)।

```python
@app.get("/zset/range")
async def get_request_list(user_id: str):
    items = await redis_client.zrange(f"requests:{user_id}", 0, -1)
    return {"requests": items}
```

### `zrangebyscore(key, min, max)`
**কাজ:** score-এর range অনুযায়ী item গুলো ফেরত দেয়। যেমন — গত ৬০ সেকেন্ডে কোন কোন রিকোয়েস্ট হয়েছে সেটা বের করা।

```python
@app.get("/zset/range-by-score")
async def get_recent_requests(user_id: str):
    current_time = time.time()
    window_start = current_time - 60
    recent = await redis_client.zrangebyscore(f"requests:{user_id}", window_start, current_time)
    return {"recent_requests": recent}
```

### `zscore(key, member)`
**কাজ:** নির্দিষ্ট member-এর score (যেমন timestamp) বের করে।

```python
@app.get("/zset/score")
async def get_member_score(member: str):
    score = await redis_client.zscore("requests:user_1", member)
    return {"score": score}
```

> 💡 **সম্পূর্ণ Sliding Window Rate Limiter (ZSET দিয়ে):**
> ```python
> @app.get("/api/protected")
> async def protected_endpoint(user_id: str):
>     key = f"requests:{user_id}"
>     now = time.time()
>     window = 60       # ৬০ সেকেন্ডের window
>     max_requests = 5  # window-তে সর্বোচ্চ ৫টা রিকোয়েস্ট
>
>     # ১. পুরনো timestamp গুলো মুছে ফেলা
>     await redis_client.zremrangebyscore(key, 0, now - window)
>
>     # ২. বর্তমান window-তে কতগুলো রিকোয়েস্ট হয়েছে তা গোনা
>     current_count = await redis_client.zcard(key)
>
>     if current_count >= max_requests:
>         return {"error": "Rate limit exceeded, try again later"}
>
>     # ৩. নতুন রিকোয়েস্ট যোগ করা
>     await redis_client.zadd(key, {str(now): now})
>     # ৪. এই key-টা নিজে থেকে মুছে যাওয়ার জন্য expiry সেট করা (memory বাঁচাতে)
>     await redis_client.expire(key, window)
>
>     return {"message": "Request allowed", "remaining": max_requests - current_count - 1}
> ```

---

## 🧠 ৬. Key Management

👉 **এটা কী:** Redis-এর ভেতরে থাকা key গুলো manage করার জন্য সাধারণ utility commands।

### `keys(pattern)`
**কাজ:** pattern অনুযায়ী match করা সব key খুঁজে বের করে। ⚠️ **Production-এ ব্যবহার এড়িয়ে চলা উচিত**, কারণ এটা পুরো Redis database scan করে, যা বড় ডেটাবেসে server-কে ব্লক করে দিতে পারে।

```python
@app.get("/keys/pattern")
async def find_keys_by_pattern():
    # "user:" দিয়ে শুরু হওয়া সব key খুঁজছে — শুধু development/debug-এর জন্য
    matched_keys = await redis_client.keys("user:*")
    return {"keys": matched_keys}
```

### `scan(cursor)`
**কাজ:** `keys()`-এর নিরাপদ বিকল্প। পুরো ডেটাবেস একসাথে স্ক্যান না করে ধাপে ধাপে (cursor ব্যবহার করে) key খুঁজে বের করে, ফলে server ব্লক হয় না।

```python
@app.get("/keys/scan")
async def scan_keys_safely():
    cursor = 0
    all_keys = []
    while True:
        cursor, keys_batch = await redis_client.scan(cursor, match="user:*", count=100)
        all_keys.extend(keys_batch)
        if cursor == 0:  # cursor 0 মানে scan শেষ
            break
    return {"keys": all_keys}
```

### `rename(key, new_key)`
**কাজ:** একটা key-এর নাম পরিবর্তন করে। নতুন নামে আগে থেকে key থাকলে সেটা overwrite হয়ে যাবে।

```python
@app.post("/keys/rename")
async def rename_key():
    await redis_client.rename("user:temp", "user:1")
    return {"message": "Key renamed"}
```

### `type(key)`
**কাজ:** একটা key কোন data type-এর (string, hash, list, set, zset) সেটা জানায়।

```python
@app.get("/keys/type")
async def get_key_type():
    key_type = await redis_client.type("requests:user_1")
    return {"type": key_type}  # যেমন: "zset"
```

---

## 🧠 ৭. TTL / Expiry System

👉 **এটা কী:** কোনো key কতক্ষণ বেঁচে থাকবে সেটা নিয়ন্ত্রণ করার commands। Session, OTP, temporary cache ইত্যাদির জন্য অপরিহার্য।

### `expire(key, seconds)`
**কাজ:** সেকেন্ড হিসেবে expiry সেট করে (আগেও দেখানো হয়েছে, এখানে আবার context দেওয়া হলো)।

```python
@app.post("/ttl/expire")
async def set_expiry():
    await redis_client.set("session:abc", "active")
    await redis_client.expire("session:abc", 1800)  # ৩০ মিনিট পর session শেষ
    return {"message": "Session expires in 30 minutes"}
```

### `pexpire(key, ms)`
**কাজ:** `expire`-এর মতোই, তবে সময়টা সেকেন্ডের বদলে মিলিসেকেন্ডে দিতে হয়। খুব নিখুঁত (precise) সময় দরকার হলে এটা ব্যবহার হয়।

```python
@app.post("/ttl/pexpire")
async def set_expiry_ms():
    await redis_client.set("lock:resource_1", "locked")
    await redis_client.pexpire("lock:resource_1", 500)  # মাত্র ৫০০ মিলিসেকেন্ড পর expire
    return {"message": "Lock expires in 500ms"}
```

### `persist(key)`
**কাজ:** কোনো key-এর উপর থেকে expiry সরিয়ে দেয়, ফলে key permanent (চিরস্থায়ী) হয়ে যায়।

```python
@app.post("/ttl/persist")
async def remove_expiry():
    await redis_client.persist("session:abc")
    return {"message": "Session is now permanent"}
```

### `ttl(key)`
**কাজ:** key expire হতে আর কত সেকেন্ড বাকি তা জানায় (উপরে String section-এও দেখানো হয়েছে)।

```python
@app.get("/ttl/check")
async def check_remaining_time():
    seconds = await redis_client.ttl("session:abc")
    return {"seconds_left": seconds}
```

---

## 🧠 ৮. Pub/Sub (real-time system)

👉 **এটা কী:** Publisher-Subscriber pattern — এক জায়গা থেকে message পাঠানো (`publish`) আর অন্য জায়গা থেকে সেটা শোনা (`subscribe`)। Real-time notification, chat system, live update-এর জন্য ব্যবহার হয়। FastAPI-তে এটা সাধারণত WebSocket-এর সাথে মিলিয়ে ব্যবহার করা হয়।

### `publish(channel, message)`
**কাজ:** একটা channel-এ message পাঠায়। যারা সেই channel subscribe করে আছে তারা সাথে সাথে message পাবে।

```python
@app.post("/pubsub/publish")
async def publish_notification():
    await redis_client.publish("notifications", "New order received!")
    return {"message": "Notification published"}
```

### `subscribe(channel)`
**কাজ:** নির্দিষ্ট channel শোনা শুরু করে, এবং কেউ সেখানে publish করলেই message পাওয়া যায়। এটা সাধারণত একটা আলাদা background task বা WebSocket-এর ভেতরে চালানো হয়, কারণ এটা continuously listen করে।

```python
import asyncio

async def listen_to_notifications():
    pubsub = redis_client.pubsub()
    await pubsub.subscribe("notifications")
    async for message in pubsub.listen():
        if message["type"] == "message":
            print(f"নতুন বার্তা এসেছে: {message['data']}")

@app.on_event("startup")
async def start_listener():
    # FastAPI app চালু হওয়ার সাথে সাথে background-এ listener চালু হচ্ছে
    asyncio.create_task(listen_to_notifications())
```

---

## 🧠 ৯. Transaction / Pipeline

👉 **এটা কী:** একাধিক Redis command একসাথে, দ্রুত এবং atomic ভাবে execute করার পদ্ধতি। প্রতিটা command আলাদাভাবে network round-trip না করে, সব একসাথে পাঠিয়ে দেওয়া যায় — ফলে performance অনেক বাড়ে।

### `pipeline()`
**কাজ:** একাধিক command একসাথে জমা করে এক ব্যাচে পাঠানোর জন্য ব্যবহার হয় (batch execution), ফলে network overhead কমে।

```python
@app.post("/pipeline/batch")
async def batch_operations():
    pipe = redis_client.pipeline()
    pipe.set("key1", "value1")
    pipe.set("key2", "value2")
    pipe.incr("counter")
    results = await pipe.execute()  # সব command একসাথে এক্সিকিউট হচ্ছে
    return {"results": results}
```

### `multi()` ও `exec()`
**কাজ:** `multi()` দিয়ে transaction শুরু হয় এবং `exec()` দিয়ে সেই transaction-এর সব command একসাথে, atomic ভাবে (মাঝখানে কোনো interruption ছাড়া) execute হয়। `redis-py` লাইব্রেরিতে এটা pipeline-এর `transaction=True` mode দিয়ে হ্যান্ডেল করা হয়:

```python
@app.post("/transaction/run")
async def run_transaction():
    async with redis_client.pipeline(transaction=True) as pipe:
        # multi() স্বয়ংক্রিয়ভাবে শুরু হয়ে যায় এই pipeline ব্লকে
        pipe.incr("balance")
        pipe.decr("pending_balance")
        results = await pipe.execute()  # এখানে exec() এর কাজ হয়ে যাচ্ছে
    return {"results": results}
```

> 💡 **পার্থক্য:** `pipeline()` মূলত speed-এর জন্য (network call কম করা), আর `multi()`/`exec()` মূলত atomicity-এর জন্য — মানে মাঝখানে অন্য কোনো client এসে ডেটা পরিবর্তন করতে পারবে না।

---

## 🧠 ১০. Connection

👉 **এটা কী:** Redis server-এর সাথে সংযোগ পরীক্ষা এবং বন্ধ করার commands। FastAPI app-এর health check বা graceful shutdown-এ এগুলো গুরুত্বপূর্ণ।

### `ping()`
**কাজ:** Redis server সচল (alive) আছে কিনা চেক করে। সাড়া পেলে `True`/`PONG` রিটার্ন করে। সাধারণত health check endpoint-এ ব্যবহার হয়।

```python
@app.get("/health")
async def health_check():
    try:
        is_alive = await redis_client.ping()
        return {"redis_status": "connected" if is_alive else "disconnected"}
    except Exception:
        return {"redis_status": "error"}
```

### `close()`
**কাজ:** Redis-এর সাথে connection বন্ধ করে দেয়। FastAPI app বন্ধ হওয়ার সময় (shutdown event-এ) এটা কল করা ভালো অভ্যাস, যাতে resource leak না হয়।

```python
@app.on_event("shutdown")
async def shutdown_event():
    await redis_client.close()
    print("Redis connection বন্ধ করা হয়েছে")
```

---

## 📋 দ্রুত Summary টেবিল

| Category | মূল Function | কাজ |
|---|---|---|
| String | `set`, `get`, `incr`, `expire` | সাধারণ key-value, counter, cache |
| Hash | `hset`, `hgetall` | object/record store করা |
| List | `lpush`, `rpush`, `lpop` | queue, job processing |
| Set | `sadd`, `smembers` | unique item collection |
| Sorted Set | `zadd`, `zremrangebyscore`, `zcard` | rate limiting, leaderboard |
| Key Mgmt | `keys`, `scan`, `type` | key খোঁজা ও পরিচালনা |
| TTL | `expire`, `ttl`, `persist` | auto-expiry নিয়ন্ত্রণ |
| Pub/Sub | `publish`, `subscribe` | real-time messaging |
| Transaction | `pipeline`, `multi/exec` | batch ও atomic operation |
| Connection | `ping`, `close` | health check ও cleanup |

---

## ✅ শেষ কথা

Rate limiting-এর জন্য বাস্তবে সবচেয়ে বেশি ব্যবহার হয় **String (`incr` + `expire`)** — সহজ fixed-window approach-এর জন্য, এবং **Sorted Set (`zadd` + `zremrangebyscore` + `zcard`)** — নিখুঁত sliding-window approach-এর জন্য। বাকি commands গুলো (Hash, List, Set, Pub/Sub) সাধারণত caching, queue, এবং real-time feature বানাতে কাজে লাগে।
