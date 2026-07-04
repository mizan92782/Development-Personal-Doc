# 🚦 MODULE 9: RATE LIMITING — DEEP DIVE (FAANG Level)

> Module 5.3-এ Rate Limiting-এর basic concept covered হয়েছিল। এই মডিউলে আমরা algorithm-গুলোর গভীরে যাবো — math, distributed implementation, এবং production-grade design।

---

## 9.1 Token Bucket Algorithm

### 1. Topic Name
Token Bucket Rate Limiting

### 2. কেন শিখবে?
Token Bucket সবচেয়ে বেশি ব্যবহৃত algorithm কারণ এটা **burst traffic** allow করে কিন্তু **sustained rate** নিয়ন্ত্রণ করে — বাস্তব traffic pattern-এ user মাঝে মাঝে দ্রুত কয়েকটা request পাঠায় (burst), তারপর থেমে যায়। AWS, Stripe, GitHub সবাই এটা ব্যবহার করে কারণ এটা user-friendly এবং implement করাও তুলনামূলক সহজ।

### 3. Problem এটা কী সমাধান করে
Fixed rate limiting (যেমন "প্রতি সেকেন্ডে ঠিক ১০টা") legitimate burst-কেও block করে দেয়, যা user experience খারাপ করে। Token Bucket burst-friendly থাকে কারণ bucket-এ token জমা থাকলে হঠাৎ কয়েকটা request একসাথে allow হয়।

### 4. Production Failure যদি Ignore করো
Rate limiter ছাড়া বা fixed-window limiter ব্যবহার করলে window boundary-তে "double burst" সমস্যা হয় — যেমন মিনিটের শেষ সেকেন্ডে ১০০ request, পরের মিনিটের প্রথম সেকেন্ডে আরও ১০০ request — কার্যত ২ সেকেন্ডে ২০০ request পার হয়ে যায় limit থাকা সত্ত্বেও।

### 5. Real-world উদাহরণ
- **AWS API Gateway**: token bucket দিয়ে burst limit + steady-state rate limit দুটোই configure করা যায়
- **Stripe API**: token bucket-ভিত্তিক rate limiting, প্রতিটা account-এর নিজস্ব bucket
- **Shopify API**: leaky bucket ভিত্তিক, কিন্তু docs-এ "token bucket-এর মতো আচরণ" describe করে

### 6. API Concept Explanation — Math
```
bucket_capacity = 10        # সর্বোচ্চ ১০টা token জমা থাকতে পারে
refill_rate = 2 tokens/sec  # প্রতি সেকেন্ডে ২টা token যোগ হয়

Request আসলে:
  if current_tokens >= 1:
      current_tokens -= 1
      allow request
  else:
      reject request (429)
```
Bucket পূর্ণ থাকলে ১০টা request একসাথে (burst) allow হবে, তারপর প্রতি সেকেন্ডে মাত্র ২টা request pass হবে (sustained rate)।

### 7. Django REST Framework Implementation
```python
import time
import redis

r = redis.Redis()

def token_bucket_allow(user_id, capacity=10, refill_rate=2):
    key = f"rate_limit:{user_id}"
    now = time.time()
    bucket = r.hgetall(key)

    if not bucket:
        tokens, last_refill = capacity, now
    else:
        tokens = float(bucket[b'tokens'])
        last_refill = float(bucket[b'last_refill'])
        elapsed = now - last_refill
        tokens = min(capacity, tokens + elapsed * refill_rate)

    if tokens >= 1:
        tokens -= 1
        r.hset(key, mapping={"tokens": tokens, "last_refill": now})
        r.expire(key, 60)
        return True
    return False
```

### 8. Spring Boot Equivalent (Bucket4j + Redis)
```java
BucketConfiguration config = BucketConfiguration.builder()
    .addLimit(Bandwidth.classic(10, Refill.greedy(2, Duration.ofSeconds(1))))
    .build();

ProxyManager<String> proxyManager = ... ; // Redis-backed
Bucket bucket = proxyManager.builder().build(userId, config);

if (bucket.tryConsume(1)) {
    // process request
} else {
    return ResponseEntity.status(429).header("Retry-After", "1").build();
}
```

### 9. Internal Request Lifecycle
Request আসে → user/API-key অনুযায়ী bucket lookup (Redis) → elapsed time অনুযায়ী token refill calculate → token থাকলে consume করে proceed, না থাকলে `429` + `Retry-After` header রিটার্ন।

### 10. Architecture Diagram (Text)
```
             ┌─────────────────────┐
Request -->  │  Token Bucket Check  │ --> tokens available? --> Yes --> Forward to Service
             │  (Redis: HGETALL)    │                       --> No  --> 429 Too Many Requests
             └─────────────────────┘
```

### 11. Performance Impact
Redis-এ atomic operation (Lua script দিয়ে check+decrement একসাথে) না করলে race condition হয় — দুইটা concurrent request একই সাথে token চেক করে দুটোই pass করে যেতে পারে (over-limit)। এটাই সবচেয়ে common production bug rate limiter-এ।

### 12. Common Mistakes (Junior vs Senior)
- Junior: read-then-write আলাদা আলাদা Redis call করে (race condition তৈরি করে)
- Senior: Redis Lua script দিয়ে atomic check-and-decrement করে
- Junior: single global bucket ব্যবহার করে সব user-এর জন্য
- Senior: per-user/per-API-key আলাদা bucket রাখে

### 13. Debugging Strategy
`429` response rate spike হলে rate limit config এবং Redis latency চেক করা। Bucket state log করে দেখা token refill সঠিকভাবে হচ্ছে কিনা।

### 14. Optimization Techniques
Redis Lua script ব্যবহার করে atomic operation (network round-trip কমায় এবং race condition এড়ায়):
```lua
-- atomic token bucket Lua script (conceptual)
local tokens = tonumber(redis.call('HGET', KEYS[1], 'tokens') or capacity)
-- refill logic + consume logic এখানে atomic ভাবে হয়
```

### 15. Trade-offs (কখন ব্যবহার করবে না)
যদি strict smooth rate দরকার হয় (কোনো burst allow না করা), Leaky Bucket ভালো choice — Token Bucket সবসময় কিছুটা burst allow করে যা কিছু use case-এ (যেমন hardware protection) অনাকাঙ্ক্ষিত।

### 16. Production Best Practices
Rate limit response-এ `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `Retry-After` header দেয়া client-কে informed রাখতে। Per-tier rate limit রাখা (free vs paid user)।

### 17. Security Considerations
IP-based rate limiting শুধু যথেষ্ট না কারণ attacker easily IP rotate করতে পারে — account/API-key based limiting-এর সাথে সাথে IP-based limiting layer করা উচিত (defense in depth)।

### 18. Senior Engineer Design Mindset
সিনিয়র ইঞ্জিনিয়ার ভাবে: "distributed environment-এ (multiple gateway instance) এই counter কীভাবে consistent থাকবে?" — এই প্রশ্ন থেকেই centralized Redis-backed counter design আসে, local in-memory counter নয়।

---

## 9.2 Leaky Bucket Algorithm

### 1-2. Topic ও কেন শিখবে
Leaky Bucket output rate-কে সম্পূর্ণ **smooth/constant** রাখে — এটা downstream system-কে protect করার জন্য উপযোগী যেখানে হঠাৎ burst একদমই সহ্য করা যায় না (যেমন hardware-limited legacy system-এর সাথে integration)।

### 3-4. Problem ও Failure
Token bucket-এর burst allowance কিছু downstream system-এর জন্য বিপজ্জনক (যেমন printer queue, legacy mainframe integration) — Leaky Bucket সেখানে queue করে ধীরে ধীরে process করে, burst থেকে সম্পূর্ণ রক্ষা দেয়।

### 5. Real-world উদাহরণ
নেটওয়ার্ক traffic shaping-এ Leaky Bucket ব্যাপকভাবে ব্যবহৃত হয় (network router-এ QoS management)। কিছু payment gateway integration-এও এটা ব্যবহার হয় যেখানে downstream bank system fixed rate-এর বেশি নিতে পারে না।

### 6. API Concept — Math
```
Request queue-তে ঢোকে (bucket-এ পানি পড়ার মতো)
Fixed rate-এ queue থেকে process হয় (bucket থেকে leak হওয়ার মতো)
Queue পূর্ণ হয়ে গেলে নতুন request drop হয় (bucket overflow)
```
Token Bucket-এর সাথে মূল পার্থক্য: Token Bucket burst allow করে (output rate variable), কিন্তু Leaky Bucket output rate সবসময় constant রাখে queue ব্যবহার করে।

### 7. DRF Implementation (Conceptual, Celery queue দিয়ে)
```python
from celery import shared_task
import time

REQUEST_QUEUE_KEY = "leaky_bucket_queue"

def enqueue_request(payload):
    if r.llen(REQUEST_QUEUE_KEY) >= MAX_QUEUE_SIZE:
        raise Exception("Queue full - request dropped")
    r.rpush(REQUEST_QUEUE_KEY, json.dumps(payload))

@shared_task
def process_queue_at_fixed_rate():
    # fixed rate-এ (e.g. প্রতি ২০০ms-এ ১টা) queue থেকে item process করে
    item = r.lpop(REQUEST_QUEUE_KEY)
    if item:
        process(json.loads(item))
```

### 8. Spring Boot Equivalent
```java
// Guava RateLimiter দিয়ে smooth rate simulate করা যায় (leaky-bucket-এর মতো আচরণ)
RateLimiter rateLimiter = RateLimiter.create(5.0); // ৫ permits/second, smooth

public void handleRequest(Request req) {
    rateLimiter.acquire(); // fixed rate না মেলা পর্যন্ত block করে
    process(req);
}
```

### 9-10. Lifecycle ও Diagram
```
Requests -->  [Queue (Bucket)] --> Fixed-Rate Worker (Leak) --> Downstream Service
                    |
              Queue Full? --> Drop/Reject New Requests
```

### 11. Performance Impact
Queue ব্যবহারের কারণে latency যোগ হয় (request সাথে সাথে process হয় না, queue-তে wait করে) — যা real-time API-এর জন্য ideal না, কিন্তু background/batch processing-এর জন্য perfect।

### 12. Common Mistakes
Junior: Leaky Bucket ব্যবহার করে real-time user-facing API-তে যেখানে latency critical — ফলে user experience খারাপ হয়।
Senior: Leaky Bucket শুধু ব্যবহার করে যেখানে smooth output rate essential, latency tolerance আছে এমন জায়গায় (batch job, downstream protection)।

### 13-18. Debugging, Optimization, Trade-off, Best Practices
Trade-off: Token Bucket vs Leaky Bucket সিদ্ধান্ত নেয়ার key প্রশ্ন — "burst allow করা ঠিক আছে, নাকি output rate একদম smooth হওয়া দরকার?" প্রথমটার উত্তর হ্যাঁ হলে Token Bucket, না হলে Leaky Bucket।

---

## 9.3 Throttling in DRF (Practical Deep Dive)

### 1-2. Topic ও কেন শিখবে
DRF-এর built-in throttling system production-ready এবং customizable — বেশিরভাগ ক্ষেত্রে custom implementation না করে DRF-এর throttling class ব্যবহার করাই ভালো (reinventing the wheel এড়ানো)।

### 6-7. API Concept ও Implementation
```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.ScopedRateThrottle',
    ],
}

class LoginThrottle(ScopedRateThrottle):
    scope = 'login'

REST_FRAMEWORK['DEFAULT_THROTTLE_RATES'] = {
    'login': '5/minute',      # brute-force login attack prevent করতে কম rate
    'search': '30/minute',
    'upload': '10/hour',
}

class LoginView(APIView):
    throttle_classes = [LoginThrottle]
    throttle_scope = 'login'
```

Custom throttle — subscription tier অনুযায়ী:
```python
class SubscriptionRateThrottle(UserRateThrottle):
    def get_rate(self):
        user = self.request.user
        if user.subscription_tier == 'premium':
            return '1000/hour'
        return '100/hour'
```

### 12. Common Mistakes
Junior: সব endpoint-এ একই global rate limit বসায়।
Senior: sensitive endpoint (login, password reset, OTP verify)-এ অনেক কম, low-risk endpoint (public read)-এ বেশি rate রাখে — endpoint-এর risk profile অনুযায়ী scope আলাদা করে।

### 16-18. Best Practices, Security, Mindset
Login/OTP endpoint-এ throttling brute-force attack ঠেকানোর প্রথম defense layer — কিন্তু এটাই একমাত্র defense না, CAPTCHA ও account lockout policy-র সাথে combine করা উচিত।

### 🔹 Interview Questions (Module 9 সম্পূর্ণ)

**Beginner**
1. Token Bucket ও Leaky Bucket-এর মূল পার্থক্য কী?
2. DRF-এ throttling কীভাবে configure করে?
3. `429 Too Many Requests` response কখন পাঠানো উচিত?

**Intermediate**
4. Token Bucket-এ "burst" কীভাবে কাজ করে ব্যাখ্যা করো একটা numeric example দিয়ে।
5. `Retry-After` header কেন গুরুত্বপূর্ণ?
6. Login endpoint-এ কেন আলাদা (কম) rate limit রাখা উচিত অন্য endpoint-এর তুলনায়?

**Advanced/Senior**
7. Distributed system-এ (multiple API gateway instance) rate limiter-এ race condition কীভাবে prevent করবে?
8. Sliding Window Log ও Sliding Window Counter algorithm-এর মধ্যে trade-off কী?
9. Multi-region deployment-এ globally-consistent rate limiting design করলে কী কী challenge আসবে (network latency, Redis replication lag)?

**FAANG System Design**
10. Design a distributed rate limiter for an API Gateway serving 10 million requests/second globally, with per-user and per-IP limits.
11. Stripe-এর মতো একটা payment API-তে rate limit design করো যেখানে different endpoint-এর জন্য different sensitivity আছে (charge creation vs balance check)।

**Scenario-Based**
12. একটা client complain করছে তাদের legitimate traffic মাঝে মাঝে 429 পাচ্ছে যদিও তারা limit-এর নিচে আছে বলে মনে করে — root cause analysis কীভাবে করবে? (উত্তর সম্ভাবনা: clock skew, multiple gateway instance-এ inconsistent counter, retry logic নিজেই extra request তৈরি করছে)

---

# 🧠 MODULE 10: SYSTEM DESIGN (API FOCUS) — FAANG Level

> প্রতিটা case study-তে: Requirements → Core APIs → Data Model → High-Level Architecture → Scaling Strategy → Trade-offs → Interview Questions

---

## 10.1 Amazon (E-commerce) — Product & Order System Design

### Requirements (Functional)
- Product browse/search, cart management, order placement, inventory tracking, payment processing

### Requirements (Non-Functional)
- High availability (৯৯.৯৯%), low latency product page (<200ms), strong consistency for inventory/payment, eventual consistency ঠিক আছে recommendation/review-এর জন্য

### Core APIs
```
GET  /api/products/{id}
GET  /api/products/search?q=laptop&category=electronics
POST /api/cart/items
POST /api/orders                  (checkout)
GET  /api/orders/{id}/status
POST /api/inventory/reserve       (internal service call)
```

### Data Model (simplified)
```
Product(id, name, price, category, seller_id)
Inventory(product_id, warehouse_id, quantity)  -- strongly consistent
Order(id, user_id, items[], total, status, created_at)
Payment(id, order_id, status, gateway_txn_id)
```

### High-Level Architecture
```
Client --> API Gateway --> Product Service --> Product DB (read-heavy, cached)
                        --> Cart Service --> Redis (session cart)
                        --> Order Service --> Order DB
                                          --> Inventory Service (reserve stock, must be consistent!)
                                          --> Payment Service --> External Payment Gateway
                                          --> Notification Service (async via Kafka)
```

### কেন এভাবে ডিজাইন?
Product browsing read-heavy এবং high-traffic — এখানে **eventual consistency + heavy caching (CDN, Redis)** ব্যবহার করা হয়, কারণ product info একটু stale হলেও বড় সমস্যা না। কিন্তু **Inventory reservation** এবং **Payment** অংশে **strong consistency** দরকার — দুইজন customer একই শেষ item একসাথে কিনে ফেললে সমস্যা (overselling), তাই এখানে **distributed lock** বা **atomic decrement** ব্যবহার করা হয়:
```python
# Atomic inventory decrement (race condition free)
UPDATE inventory SET quantity = quantity - 1
WHERE product_id = %s AND quantity > 0;
-- affected rows == 0 হলে "out of stock" error
```

### Scaling Strategy
- Product catalog: read replica + CDN caching + Elasticsearch for search
- Order processing: async event-driven (Order placed → Kafka event → Inventory, Payment, Notification service independently consume)
- Database sharding: user_id বা order_id দিয়ে horizontal shard করা millions of orders handle করতে

### Trade-offs
Strong consistency (inventory) latency বাড়ায় কিন্তু overselling prevent করে — এটা business-critical, তাই performance-এর চেয়ে correctness priority পায় এখানে। Eventually consistent recommendation engine latency কম রাখে কারণ accuracy-তে সামান্য delay acceptable।

### 🔹 Interview Questions
- Overselling (same item দুইজন কিনে ফেলা) কীভাবে prevent করবে high-concurrency-তে?
- Order placement-এর সময় payment fail হলে inventory reservation কীভাবে rollback করবে (Saga pattern)?
- Flash sale (Black Friday)-এর মতো ১০০x traffic spike কীভাবে handle করবে?

---

## 10.2 Netflix (Streaming) — Video Delivery & Recommendation System Design

### Requirements
- Video streaming (adaptive bitrate), personalized recommendations, watch history sync, offline download

### Non-Functional
- Extremely high availability, global low-latency video delivery, eventual consistency acceptable বেশিরভাগ জায়গায়

### Core APIs
```
GET  /api/videos/{id}/manifest        (adaptive bitrate streaming manifest)
GET  /api/recommendations?user_id=X
POST /api/watch-history
GET  /api/videos/{id}/metadata
```

### Data Model
```
Video(id, title, genre, duration, available_bitrates[])
WatchHistory(user_id, video_id, position_seconds, timestamp)
Recommendation(user_id, video_ids[], generated_at)  -- precomputed, cached
```

### High-Level Architecture
```
Client --> CDN (Open Connect - Netflix's own CDN) --> Video Chunks (cached at ISP edge)
Client --> API Gateway --> Recommendation Service --> ML Model (precomputed offline)
                        --> Watch History Service --> Cassandra (write-heavy, eventually consistent)
                        --> Metadata Service --> Cache (Redis/EVCache)
```

### কেন এভাবে ডিজাইন?
Video content নিজেই **CDN-এ pre-positioned** থাকে (Netflix-এর নিজস্ব **Open Connect** CDN, ISP-এর ভেতরেই সার্ভার বসিয়ে দেয়) যাতে video streaming latency near-zero হয় — এটা সরাসরি origin server থেকে stream করার চেয়ে অনেক বেশি efficient। Recommendation **real-time compute** না করে **precomputed/batch** রাখা হয় (offline ML pipeline প্রতি রাতে চালিয়ে recommendation cache করে রাখা), কারণ real-time personalization compute millions of user-এর জন্য expensive।

### Scaling Strategy
- Watch history-এর মতো write-heavy data-এ **Cassandra** (AP system, eventual consistency, high write throughput)
- Video metadata heavy caching (multiple layer: CDN edge, regional cache, application cache)
- Chaos Engineering (Netflix-এর বিখ্যাত **Chaos Monkey**) দিয়ে proactively failure inject করে resilience test করা

### Trade-offs
Eventual consistency watch history sync-এ acceptable (কয়েক সেকেন্ড delay হলেও সমস্যা না), কিন্তু billing/subscription-এর মতো জায়গায় strong consistency দরকার — তাই Netflix বিভিন্ন data-এর জন্য বিভিন্ন consistency model ব্যবহার করে (polyglot persistence)।

### 🔹 Interview Questions
- Adaptive bitrate streaming কীভাবে কাজ করে এবং API design-এ এটা কীভাবে reflect হয়?
- একটা video হঠাৎ viral হয়ে গেলে (traffic spike) CDN কীভাবে handle করবে?
- Recommendation system precomputed রাখলে "cold start problem" (নতুন user, কোনো history নেই) কীভাবে সমাধান করবে?

---

## 10.3 Uber (Ride System) — Real-Time Matching System Design

### Requirements
- Real-time driver location tracking, rider-driver matching, dynamic pricing (surge), trip lifecycle management

### Non-Functional
- Extremely low latency matching (সেকেন্ডের মধ্যে), high write throughput (location updates প্রতি কয়েক সেকেন্ডে প্রতিটা driver থেকে)

### Core APIs
```
POST /api/driver/location          (frequent, high-throughput)
POST /api/rides/request            (rider requests a ride)
GET  /api/rides/{id}/status
POST /api/rides/{id}/accept        (driver accepts)
GET  /api/pricing/estimate?from=X&to=Y
```

### Data Model
```
DriverLocation(driver_id, lat, lng, timestamp, status)  -- ephemeral, in-memory/geospatial index
Ride(id, rider_id, driver_id, pickup, dropoff, status, fare)
SurgeZone(geohash, multiplier, updated_at)
```

### High-Level Architecture
```
Driver App --location update (every 4s)--> Location Service --> Geospatial Index (Redis Geo / H3)
Rider App --request ride--> Matching Service --queries--> Geospatial Index (find nearby drivers)
                                              --> Dispatch driver --> Notify via Push/WebSocket
                          --> Pricing Service --> Surge calculation (demand/supply ratio per zone)
```

### কেন এভাবে ডিজাইন?
Driver location millions বার update হয় প্রতি সেকেন্ডে — এটা traditional relational DB-তে রাখা অসম্ভব efficient না। তাই **geospatial in-memory index** (Redis Geo, বা Uber-এর নিজস্ব **H3 hexagonal grid system**) ব্যবহার করা হয় যাতে "nearest driver খোঁজা" O(log n)-এর কাছাকাছি সময়ে হয়। Matching-এ **WebSocket/long-polling** ব্যবহার হয় (REST polling না) কারণ real-time bidirectional communication দরকার (driver accept/reject মুহূর্তের মধ্যে rider-কে জানাতে হয়)।

### Scaling Strategy
- Geographic sharding (প্রতিটা city/region আলাদা shard-এ, কারণ cross-city matching দরকার নেই)
- Location data ephemeral রাখা (short TTL, persistent storage-এ না রেখে শুধু active trip-এর জন্য history রাখা)
- Surge pricing asynchronously precompute করা প্রতি কয়েক সেকেন্ডে, real-time নয় প্রতিটা request-এ

### Trade-offs
Strong consistency matching-এ পুরোপুরি সম্ভব না (দুইজন driver-কে একসাথে একই ride offer করা হতে পারে race condition-এ) — তাই **optimistic locking** ব্যবহার হয়: প্রথম driver যে accept করে সে পায়, বাকিদের "already taken" reject message যায়।

### 🔹 Interview Questions
- Nearby driver খোঁজার জন্য geospatial indexing কীভাবে কাজ করে (Geohash/H3/QuadTree)?
- দুইজন driver একসাথে একই ride accept করে ফেললে race condition কীভাবে resolve করবে?
- Surge pricing কীভাবে design করবে যাতে real-time হয় কিন্তু computationally expensive না হয়?

---

## 10.4 WhatsApp (Chat System) — Real-Time Messaging System Design

### Requirements
- One-to-one ও group messaging, message delivery status (sent/delivered/read), end-to-end encryption, offline message delivery

### Non-Functional
- Extremely high availability, low latency (<1s message delivery), massive scale (billions of messages/day)

### Core APIs (WebSocket-heavy, কিছু REST)
```
WS   /connect                      (persistent WebSocket connection)
POST /api/messages/send            (fallback REST for reliability)
GET  /api/messages/{chat_id}?since=timestamp   (sync offline messages)
POST /api/messages/{id}/ack        (delivery/read receipt)
```

### Data Model
```
Message(id, chat_id, sender_id, content_encrypted, timestamp, status)
Chat(id, participant_ids[], type: 'direct'/'group')
DeliveryStatus(message_id, user_id, status, timestamp)
```

### High-Level Architecture
```
Client A --WebSocket--> Connection Gateway (holds persistent connection, tracks online status)
                              |
                        Message Service --> Message Queue (Kafka) --> Persist to DB
                              |
                        Connection Gateway (finds recipient's connection) --> Client B (if online)
                              |
                        If Client B offline --> Push Notification Service (FCM/APNs)
                                             --> Message stored, delivered when B reconnects
```

### কেন এভাবে ডিজাইন?
Persistent **WebSocket connection** ব্যবহার হয় কারণ REST polling billions of user-এর জন্য massively inefficient (প্রতিটা user বারবার "নতুন মেসেজ আছে কিনা" জিজ্ঞেস করলে server overload হয়ে যাবে)। **Connection Gateway** আলাদা layer হিসেবে রাখা হয় যা track করে কোন user কোন gateway server-এ connected আছে (millions of concurrent connection বিভিন্ন gateway instance-এ distribute হয়ে থাকে) — এটাকে "connection state management" বলে, এবং এটা microservices architecture-এর একটা tricky অংশ কারণ stateful connection horizontally scale করা কঠিন।

### Scaling Strategy
- Connection state Redis-এ রাখা (কোন user কোন gateway node-এ আছে) যাতে message routing করা যায়
- Message persistence asynchronous (Kafka দিয়ে) যাতে real-time delivery block না হয় DB write-এর জন্য
- Group message fan-out: বড় group (হাজার member) হলে fan-out-on-write vs fan-out-on-read trade-off বিবেচনা করা

### Trade-offs
End-to-end encryption মানে server নিজেই message content দেখতে পারে না — এটা privacy-র জন্য চমৎকার, কিন্তু server-side search/spam-detection ফিচার করা কঠিন হয়ে যায় কারণ content encrypted। এটা conscious trade-off (privacy vs feature richness)।

### 🔹 Interview Questions
- Online/offline status ও persistent connection কীভাবে scale করবে বিলিয়ন ইউজারের জন্য?
- Message-এর "delivered" ও "read" status কীভাবে reliably track করবে বিশেষ করে user offline থাকলে?
- Group chat-এ (১০০০+ member) message fan-out কীভাবে efficiently করবে?

---

## 10.5 Instagram (Social Media) — Feed & Content System Design

### Requirements
- Photo/video upload, news feed generation, likes/comments, follow system

### Non-Functional
- Read-heavy (feed view >> post creation), eventual consistency acceptable বেশিরভাগ ক্ষেত্রে, high availability

### Core APIs
```
POST /api/posts                    (upload photo/video)
GET  /api/feed?user_id=X&cursor=Y  (paginated feed)
POST /api/posts/{id}/like
POST /api/users/{id}/follow
GET  /api/posts/{id}/comments
```

### Data Model
```
Post(id, user_id, media_url, caption, created_at)
Follow(follower_id, followee_id)
Like(post_id, user_id, timestamp)
FeedCache(user_id, post_ids[])   -- precomputed feed, per user
```

### High-Level Architecture
```
User posts --> Post Service --> Media stored in Object Storage (S3) + CDN
                              --> Fan-out Service --> Push post_id to followers' FeedCache (Redis)
                                                    (fan-out-on-write, for normal users)
Celebrity account (millions of followers) --> Fan-out-on-read instead
                                             (feed merge করা হয় read time-এ, write storm এড়াতে)

Feed Request --> Feed Service --> Read precomputed FeedCache --> Merge with celebrity posts (on-read)
```

### কেন এভাবে ডিজাইন? (Fan-out Hybrid Approach)
সাধারণ user-এর (কয়েকশ follower) পোস্ট করলে **fan-out-on-write** করা হয় — অর্থাৎ পোস্ট হওয়ার সাথে সাথেই প্রতিটা follower-এর feed cache-এ push করে দেয়া হয়, যাতে feed পড়ার সময় fast হয় (precomputed, শুধু read করলেই হয়)। কিন্তু celebrity account (millions follower) পোস্ট করলে fan-out-on-write করলে millions of write একসাথে trigger হবে ("celebrity problem" / "hot key problem") — তাই এক্ষেত্রে **fan-out-on-read** করা হয়: celebrity-এর পোস্ট আলাদাভাবে রাখা হয়, এবং user যখন feed request করে তখন runtime-এ merge করে দেখানো হয়। এই **hybrid approach** Instagram/Twitter উভয়ই ব্যবহার করে।

### Scaling Strategy
- Media storage: object storage (S3) + CDN, কখনো database-এ raw image/video রাখা হয় না
- Feed cache: Redis-এ per-user precomputed list, TTL সহ
- Database sharding: user_id দিয়ে horizontal shard

### Trade-offs
Fan-out-on-write feed read fast করে কিন্তু storage/write overhead বাড়ায় (প্রতিটা follower-এর জন্য duplicate copy)। Fan-out-on-read storage বাঁচায় কিন্তু read-time computation বাড়ায়। Hybrid approach দুটোর best trade-off দেয় কিন্তু complexity বাড়ায় (দুই ধরনের code path maintain করতে হয়)।

### 🔹 Interview Questions
- "Celebrity problem" কী এবং কীভাবে সমাধান করবে (fan-out-on-write vs fan-out-on-read)?
- News feed ranking (chronological না হয়ে relevance-based) কীভাবে design করবে at scale?
- একটা পোস্ট viral হয়ে millions like/comment পেলে সেই counter কীভাবে efficiently update করবে (exact count vs approximate count trade-off)?

---

# 🧾 MODULE 9 & 10 — CHEAT SHEET

| Topic | মূল Concept | কখন ব্যবহার |
|---|---|---|
| Token Bucket | Burst allowed, refill over time | User-facing API, general purpose rate limiting |
| Leaky Bucket | Constant output rate, queue-based | Downstream protection, smooth traffic shaping |
| DRF Throttling | Scope-based rate config | Endpoint-specific limits (login vs search) |
| Amazon Design | Strong consistency (inventory/payment) + eventual (catalog) | E-commerce, transactional correctness critical |
| Netflix Design | CDN pre-positioning + precomputed recommendations | Content delivery, read-heavy media |
| Uber Design | Geospatial indexing + WebSocket real-time matching | Location-based real-time systems |
| WhatsApp Design | Persistent WebSocket + connection state management | Real-time messaging at scale |
| Instagram Design | Hybrid fan-out (write for normal, read for celebrity) | Social feed, read-heavy with hot-key problem |

## Senior Engineer Checklist ✅
- [ ] Rate limiter Redis-এ atomic operation (Lua script) দিয়ে implement করা, race condition-mukto?
- [ ] Sensitive endpoint (login/OTP)-এ আলাদা, কম rate limit আছে কি?
- [ ] System design-এ কোন অংশে strong consistency দরকার আর কোথায় eventual consistency চলবে — এই সিদ্ধান্ত স্পষ্টভাবে justify করা হয়েছে কি?
- [ ] Hot-key/celebrity problem-এর মতো edge case আগে থেকে চিন্তা করা হয়েছে কি?
- [ ] Real-time component (WebSocket/matching)-এ race condition ও failure handling plan আছে কি?

## Key Takeaways
1. Rate limiting algorithm choice ("burst allow করবো নাকি smooth রাখবো") সরাসরি business requirement থেকে আসা উচিত, শুধু technical পছন্দ না।
2. প্রতিটা বড় system design-এ সবচেয়ে গুরুত্বপূর্ণ প্রশ্ন হলো: "কোন অংশে consistency compromise করা যাবে, আর কোথায় করা যাবে না?" — এই সিদ্ধান্তই পুরো architecture-এর দিকনির্দেশনা দেয়।
3. "Hot key" বা "celebrity problem" (একজন user/item-এ অস্বাভাবিক বেশি traffic) প্রায় সব large-scale system-এই দেখা যায় — এটা চিনতে পারা এবং hybrid solution design করতে পারা সিনিয়র সিস্টেম ডিজাইন ইন্টারভিউ-এর সবচেয়ে গুরুত্বপূর্ণ সিগন্যাল।
