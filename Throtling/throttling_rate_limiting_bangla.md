# Throttling (Rate Limiting) — FastAPI + Redis (বাংলা টিউটোরিয়াল)

**Throttling / Rate Limiting** মানে হলো — কতগুলো request একজন client একটা নির্দিষ্ট সময়ের মধ্যে করতে পারবে, সেটা নিয়ন্ত্রণ করা। এটা server-কে abuse, spam, এবং overload থেকে রক্ষা করে।

এই ডকুমেন্টে আমরা ৩টা case এবং তার সাথে সময়-ভিত্তিক (time window) কনফিগারেশন দেখব:

1. 👤 **Non-user (anonymous / register / guest)** throttling
2. 👤 **User-based (login user)** throttling
3. 🎯 **Decorator-based reusable system**
4. ⏱ **Different time windows** (1 min, 5 min, 1 hour)
5. 🏗 Senior-level architecture overview

---

## 🧠 ১. NON-USER Throttling (গুরুত্বপূর্ণ)

👉 যখন কোনো user login করা থাকে না (যেমন: register, guest browsing), তখন identity হিসেবে ব্যবহার করতে হবে:

✔ **IP address** (main identity হিসেবে)

📌 **Key idea (Redis key pattern):**
```
rate:anon:{ip}
```

মানে প্রতিটা IP address-এর জন্য আলাদা Redis key থাকবে, যেখানে তার রিকোয়েস্টের timestamp জমা হবে।

### ✅ Step 1: Redis Setup

```python
import redis
import time

redis_client = redis.Redis(host="localhost", port=6379, decode_responses=True)
```

এখানে `decode_responses=True` দেওয়া হয়েছে যাতে Redis থেকে data bytes না এসে সরাসরি Python string আকারে আসে।

### ✅ Step 2: Non-user Throttling Function

```python
def non_user_allowed(ip: str, limit: int = 3, window: int = 300):
    key = f"rate:anon:{ip}"
    now = int(time.time())

    # window-এর বাইরের পুরনো timestamp গুলো মুছে ফেলা হচ্ছে
    redis_client.zremrangebyscore(key, 0, now - window)

    # বর্তমান window-তে এই IP কতবার রিকোয়েস্ট করেছে তা গোনা হচ্ছে
    count = redis_client.zcard(key)

    if count >= limit:
        return False  # limit শেষ, আর অনুমতি নেই

    # নতুন রিকোয়েস্ট timestamp যোগ করা হচ্ছে
    redis_client.zadd(key, {str(now): now})
    # key-টা window শেষে নিজে থেকে মুছে যাবে, memory বাঁচানোর জন্য
    redis_client.expire(key, window)

    return True
```

**ব্যাখ্যা:**
- `zremrangebyscore` দিয়ে পুরনো (expired) রিকোয়েস্ট মুছে দেওয়া হয় — এটাই sliding window-এর মূল কৌশল।
- `zcard` দিয়ে বর্তমানে window-এর ভেতরে কতগুলো রিকোয়েস্ট আছে তা গোনা হয়।
- limit পার হয়ে গেলে `False` রিটার্ন করে, অন্যথায় নতুন রিকোয়েস্ট যোগ করে `True` রিটার্ন করে।

### ✅ Step 3: FastAPI Usage

```python
from fastapi import Request, HTTPException

def get_ip(request: Request):
    return request.client.host

@app.post("/register")
async def register(request: Request):
    ip = get_ip(request)
    if not non_user_allowed(ip):
        raise HTTPException(429, "Too many requests")
    return {"msg": "registered"}
```

**ব্যাখ্যা:** `request.client.host` দিয়ে client-এর IP address বের করা হয়, এরপর সেই IP দিয়ে rate limit চেক করা হয়। limit পার হলে HTTP status `429 Too Many Requests` রিটার্ন হয় — এটাই rate-limiting-এর standard status code।

---

## 🧠 ২. USER-BASED Throttling

👉 এখানে user login করা থাকে, তাই identity হিসেবে IP-এর বদলে ব্যবহার হবে:

✔ **user_id**

📌 **Key idea:**
```
rate:user:{user_id}
```

### ✅ Function

```python
def user_allowed(user_id: str, limit: int = 100, window: int = 60):
    key = f"rate:user:{user_id}"
    now = int(time.time())

    redis_client.zremrangebyscore(key, 0, now - window)
    count = redis_client.zcard(key)

    if count >= limit:
        return False

    redis_client.zadd(key, {str(now): now})
    redis_client.expire(key, window)

    return True
```

লজিক ঠিক আগের মতোই, শুধু key-এর pattern আলাদা — এখন IP-এর বদলে `user_id` দিয়ে আলাদা করা হচ্ছে। ফলে প্রতিটা login user-এর নিজস্ব আলাদা rate limit counter থাকে।

### ✅ Usage

```python
@app.get("/products")
async def products(request: Request):
    user_id = request.headers.get("user-id")
    if not user_allowed(user_id):
        raise HTTPException(429, "limit exceeded")
    return {"msg": "ok"}
```

**ব্যাখ্যা:** এখানে `user-id` request header থেকে নেওয়া হচ্ছে (বাস্তব প্রজেক্টে এটা সাধারণত JWT token decode করে বা authentication dependency থেকে পাওয়া যায়)। প্রতিটা user আলাদাভাবে প্রতি মিনিটে সর্বোচ্চ ১০০টা রিকোয়েস্ট করতে পারবে।

---

## 🧠 ৩. DECORATOR-based Throttling (অত্যন্ত গুরুত্বপূর্ণ)

👉 প্রতিটা API endpoint-এ বার বার একই rate-limiting কোড লেখা senior engineer-রা করে না। এর বদলে একটা **reusable decorator** বানানো হয়, যা যেকোনো endpoint-এ এক লাইনে বসিয়ে দেওয়া যায়।

🎯 **Goal:**
```python
@rate_limit("5/minute")
```

### ✅ Step 1: Limiter Function (Decorator)

```python
def rate_limit(limit: int, window: int):
    def decorator(func):
        async def wrapper(request: Request, *args, **kwargs):
            ip = request.client.host
            key = f"rate:decorator:{func.__name__}:{ip}"
            now = int(time.time())

            redis_client.zremrangebyscore(key, 0, now - window)
            count = redis_client.zcard(key)

            if count >= limit:
                raise HTTPException(429, "Too many requests")

            redis_client.zadd(key, {str(now): now})
            redis_client.expire(key, window)

            return await func(request, *args, **kwargs)
        return wrapper
    return decorator
```

**ব্যাখ্যা:**
- এটা একটা **decorator factory** — মানে `rate_limit(limit, window)` কল করলে এটা একটা `decorator` ফাংশন রিটার্ন করে, যেটা আসল endpoint function-কে wrap করে।
- key বানানোর সময় `func.__name__` ব্যবহার করা হয়েছে, যাতে প্রতিটা আলাদা endpoint-এর জন্য আলাদা rate limit counter থাকে (একই IP হলেও `/search` আর `/login`-এর limit আলাদা হবে)।
- limit পার হলে সরাসরি `HTTPException(429, ...)` raise করে, ফলে আসল function-টা আর call-ই হয় না।

### ✅ Step 2: Usage

```python
@app.get("/search")
@rate_limit(limit=10, window=60)
async def search(request: Request):
    return {"msg": "search result"}
```

এখন `/search` endpoint-এ প্রতি IP থেকে প্রতি মিনিটে সর্বোচ্চ ১০টা রিকোয়েস্ট অনুমোদিত হবে — কোনো manual Redis কোড endpoint-এর ভেতরে লিখতে হলো না।

> ⚠️ **Decorator order নিয়ে সতর্কতা:** FastAPI-তে route decorator (`@app.get(...)`) সবসময় সবচেয়ে উপরে রাখতে হবে এবং `@rate_limit(...)` তার ঠিক নিচে। Python decorator নিচ থেকে উপরে apply হয়, তাই এই order মেনে চলা জরুরি যাতে FastAPI ঠিকভাবে route চিনতে পারে।

---

## 🧠 ৪. সময়-ভিত্তিক (Time-based) Throttling

👉 মূল concept হলো — **window** মানে কত সেকেন্ডের মধ্যে limit count হবে।

| Window (সেকেন্ড) | অর্থ |
|---|---|
| `60` | per minute (প্রতি মিনিটে) |
| `300` | per 5 minutes (প্রতি ৫ মিনিটে) |
| `3600` | per hour (প্রতি ঘণ্টায়) |

### 📌 Example Configs

```python
LOGIN_LIMIT = (5, 60)        # প্রতি মিনিটে ৫টা login চেষ্টা
REGISTER_LIMIT = (3, 300)    # প্রতি ৫ মিনিটে ৩টা register
SEARCH_LIMIT = (50, 60)      # প্রতি মিনিটে ৫০টা search
API_LIMIT = (200, 3600)      # প্রতি ঘণ্টায় ২০০টা সাধারণ API কল
```

এভাবে আলাদা আলাদা endpoint-এর জন্য আলাদা sensitivity অনুযায়ী limit আর window ঠিক করা হয় — যেমন `login`-এর মতো sensitive endpoint-এ tight limit রাখা হয় (brute-force ঠেকাতে), আর `search`-এর মতো সাধারণ endpoint-এ relaxed limit রাখা হয়।

### ✅ Usage Example

```python
@app.post("/login")
@rate_limit(limit=5, window=60)
async def login(request: Request):
    return {"msg": "login"}
```

Tuple unpack করেও ব্যবহার করা যায়, যা কোড আরও পরিষ্কার রাখে:

```python
limit, window = LOGIN_LIMIT

@app.post("/login")
@rate_limit(limit=limit, window=window)
async def login(request: Request):
    return {"msg": "login"}
```

---

## 🧠 ৫. REAL SENIOR ARCHITECTURE (চূড়ান্ত চিত্র)

👉 একটা production-grade system-এ rate limiting সাধারণত layer অনুযায়ী আলাদা আলাদা rule নিয়ে কাজ করে, কিন্তু সবাই একই **shared Redis** ব্যবহার করে যাতে সব instance/server মিলিয়ে সঠিক হিসাব থাকে।

```
            CLIENT
              |
         FASTAPI APP
        /     |      \
   anon     user    admin
    |         |        |
 Redis rate limiting (shared)
```

**ব্যাখ্যা:**
- **anon (anonymous):** IP ভিত্তিক, tight limit (যেমন register, guest browsing)
- **user:** user_id ভিত্তিক, moderate limit (যেমন logged-in user-এর normal activity)
- **admin:** admin_id ভিত্তিক, প্রয়োজনে আলাদা higher/lower limit (admin panel access)
- **Redis shared:** তিনটা layer-ই একই centralized Redis ব্যবহার করে, যাতে FastAPI app multiple server/worker-এ চললেও (horizontal scaling) rate limit হিসাব সবসময় সঠিক এবং consistent থাকে। যদি প্রতিটা server নিজের আলাদা in-memory counter রাখতো, তাহলে limit ভুল হিসাব হতো।

---

## 📋 সারসংক্ষেপ (Summary)

| Case | Identity | Key Pattern | Use-case |
|---|---|---|---|
| Non-user | IP address | `rate:anon:{ip}` | register, guest |
| User-based | user_id | `rate:user:{user_id}` | login user-এর normal activity |
| Decorator | IP / func name | `rate:decorator:{func_name}:{ip}` | reusable, যেকোনো endpoint-এ বসানো যায় |

| Time Window | সেকেন্ড | উদাহরণ Use-case |
|---|---|---|
| 1 minute | `60` | login attempt, search |
| 5 minutes | `300` | register |
| 1 hour | `3600` | general API quota |

---

## ✅ মূল শিক্ষা (Key Takeaways)

1. Rate limiting-এর core technique হলো **Sorted Set (ZSET)** দিয়ে sliding window বানানো — `zadd` + `zremrangebyscore` + `zcard`।
2. **Identity** ঠিকভাবে বেছে নেওয়া জরুরি — user না থাকলে IP, user থাকলে user_id।
3. প্রতিটা endpoint-এ manual কোড না লিখে **decorator** ব্যবহার করলে কোড clean, reusable এবং maintain করা সহজ হয়।
4. Endpoint-এর sensitivity অনুযায়ী আলাদা **limit এবং window** ঠিক করতে হয় — sensitive endpoint-এ tight limit, normal endpoint-এ relaxed limit।
5. Multiple server/worker চললে অবশ্যই **shared Redis** ব্যবহার করতে হবে, নাহলে rate limit হিসাব ভুল হয়ে যাবে।
