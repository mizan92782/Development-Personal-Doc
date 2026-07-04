# 🚀 মডিউল ৪: PERFORMANCE OPTIMIZATION
### FAANG-Level Backend Engineer হওয়ার যাত্রা — চতুর্থ মডিউল

---

## ভূমিকা

API ঠিকঠাক কাজ করা একটা জিনিস, আর API **স্কেলে** ঠিকঠাক এবং দ্রুত কাজ করা সম্পূর্ণ আলাদা একটা challenge। এই মডিউলে আমরা শিখব কীভাবে একটা slow API-কে production-grade fast সার্ভিসে রূপান্তর করা যায়। কভার করব:

1. N+1 Query Problem
2. Caching (Redis)
3. Query Optimization
4. Pagination Strategies

---

# 📌 টপিক ১: N+1 Query Problem

## ১.১ কেন শিখব?
N+1 problem হলো backend ইন্টারভিউতে সবচেয়ে বেশি জিজ্ঞেস করা performance-related প্রশ্ন। এটা এমন একটা সাইলেন্ট বাগ যা লোকালে ১০টা ডেটায় ধরা পড়ে না, কিন্তু প্রোডাকশনে ১০ লাখ ডেটায় পুরো সার্ভিস ডাউন করে দিতে পারে।

## ১.২ এটা কোন সমস্যা সমাধান করে?
আসলে N+1 কোনো সমস্যা সমাধান করে না — এটা নিজেই একটা সমস্যা যা সমাধান করতে শিখতে হয়। ORM (Object-Relational Mapping) ব্যবহার করার সময় related ডেটা lazy-load করার কারণে এটা ঘটে।

## ১.৩ Production Failure যদি Ignore করি
একটা ব্লগ লিস্টিং পেজে ১০০টা পোস্ট দেখাতে হলে, যদি প্রতিটা পোস্টের author আলাদা করে query করা হয়, তাহলে ১টা মূল query + ১০০টা author query = ১০১টা DB কল। ১০০ ইউজার একসাথে এই পেজ খুললে, ডেটাবেসে ১০,১০০টা query একসাথে হিট করবে — এটা সহজেই DB connection pool exhaust করে পুরো সার্ভিস ক্র্যাশ করিয়ে দিতে পারে।

## ১.৪ Real-World Examples
- **Instagram**: প্রথম দিকে তাদের feed-এ N+1 সমস্যা ছিল প্রতিটা পোস্টের জন্য আলাদা like-count query করার কারণে — পরে `select_related`/batch loading দিয়ে fix করা হয়
- **Shopify**: GraphQL API-তে DataLoader pattern চালু করেছিল ঠিক এই N+1 সমস্যা সমাধানের জন্য (প্রতিটা প্রোডাক্টের জন্য আলাদা inventory query এড়াতে)
- **Facebook**: তাদের internal tooling-এ N+1 detection টুল আছে যা automatically dev-এর কাছে এলার্ট পাঠায় যদি কোনো PR-এ এই প্যাটার্ন দেখা যায়

## ১.৫ Concept ব্যাখ্যা
N+1 problem ঘটে যখন:
- ১টা query করে **N**টা প্যারেন্ট রেকর্ড আনা হয়
- তারপর প্রতিটা প্যারেন্টের জন্য আলাদা **১টা** query করে related ডেটা আনা হয় (মোট N বার)
- ফলাফল: **১ + N** টা query, যেখানে আদর্শভাবে এটা ১ বা ২টা query-তেই সম্ভব

```
সমস্যাযুক্ত প্যাটার্ন:
SELECT * FROM posts;                     -- ১টা query, ১০০টা পোস্ট
SELECT * FROM users WHERE id = 1;         -- পোস্ট ১-এর author
SELECT * FROM users WHERE id = 2;         -- পোস্ট ২-এর author
... (আরও ৯৮টা)                            -- মোট ১০১টা query!
```

## ১.৬ Django REST Framework Implementation

```python
# ❌ সমস্যাযুক্ত কোড (N+1 সৃষ্টি করে)
class PostListView(APIView):
    def get(self, request):
        posts = Post.objects.all()  # ১টা query
        data = []
        for post in posts:
            data.append({
                "title": post.title,
                "author": post.author.username,  # প্রতিটা post-এ আলাদা query!
            })
        return Response(data)

# ✅ সমাধান: select_related (Foreign Key/OneToOne-এর জন্য JOIN করে)
class PostListView(APIView):
    def get(self, request):
        posts = Post.objects.select_related('author').all()  # মাত্র ১টা query, JOIN দিয়ে
        serializer = PostSerializer(posts, many=True)
        return Response(serializer.data)

# ✅ ManyToMany/Reverse FK-এর জন্য prefetch_related ব্যবহার করুন
class PostListView(APIView):
    def get(self, request):
        posts = Post.objects.select_related('author').prefetch_related('comments', 'tags').all()
        serializer = PostSerializer(posts, many=True)
        return Response(serializer.data)
```

## ১.৭ Spring Boot Equivalent

```java
// ❌ সমস্যাযুক্ত: Lazy fetching প্রতিটা author-এর জন্য আলাদা query করে
@Entity
public class Post {
    @ManyToOne(fetch = FetchType.LAZY)
    private User author;
}

// ✅ সমাধান: JPQL-এ JOIN FETCH ব্যবহার করুন
@Query("SELECT p FROM Post p JOIN FETCH p.author")
List<Post> findAllWithAuthor();

// ✅ অথবা @EntityGraph ব্যবহার করুন
@EntityGraph(attributePaths = {"author", "comments"})
@Query("SELECT p FROM Post p")
List<Post> findAllWithDetails();
```

## ১.৮ Internal Request Lifecycle (গভীর ব্যাখ্যা)
1. View/Controller `Post.objects.all()` কল করে — ORM এখনো DB-তে query পাঠায়নি (lazy evaluation)
2. Serializer যখন প্রতিটা post object-এর `author` attribute access করে, ORM তখনই (lazily) একটা নতুন query trigger করে
3. যদি `select_related` থাকে, ORM প্রথম query-তেই একটা SQL `JOIN` তৈরি করে সব ডেটা একসাথে নিয়ে আসে — ফলে দ্বিতীয়বার DB hit করার প্রয়োজন হয় না
4. `prefetch_related` সামান্য আলাদা: এটা related ডেটার জন্য একটা **আলাদা কিন্তু একটামাত্র** query করে (`WHERE id IN (...)`), তারপর Python-এ ম্যাপ করে দেয় — কারণ ManyToMany/reverse FK-তে SQL JOIN দিয়ে এক query করলে duplicate row সমস্যা হতো

## ১.৯ Architecture Diagram (Text-based)

```
❌ N+1 প্যাটার্ন:
[App] -> Query 1: GET posts -> [DB]
[App] -> Query 2: GET author(post1) -> [DB]
[App] -> Query 3: GET author(post2) -> [DB]
...
[App] -> Query N+1: GET author(postN) -> [DB]

✅ select_related প্যাটার্ন (JOIN):
[App] -> Query 1: GET posts JOIN authors -> [DB] -> সব ডেটা একসাথে

✅ prefetch_related প্যাটার্ন (Batch IN query):
[App] -> Query 1: GET posts -> [DB]
[App] -> Query 2: GET comments WHERE post_id IN (1,2,3...100) -> [DB]
```

## ১.১০ Performance Impact
১০১টা query বনাম ১-২টা query-র পার্থক্য নাটকীয়। প্রতিটা DB round-trip-এ network latency (সাধারণত ১-৫ms লোকাল নেটওয়ার্কে, কিন্তু cross-region হলে ৫০ms+) যুক্ত হয়। ১০০টা sequential query মানে অন্তত ১০০-৫০০ms অতিরিক্ত latency, যেখানে JOIN দিয়ে এটা single-digit ms-এ চলে আসতে পারে।

## ১.১১ Common Mistakes (Junior vs Senior)
- **জুনিয়র**: ORM-এর lazy loading-এর internal behavior না বুঝে loop-এর ভেতরে related object access করে; Local dev-এ ১০টা রেকর্ড নিয়ে টেস্ট করে এবং সমস্যা ধরতে পারে না
- **সিনিয়র**: Code review-এ ORM query log (Django Debug Toolbar / Hibernate SQL logging) চেক করা একটা habit বানিয়ে নেয়; production-scale ডেটা দিয়ে load test করে query count যাচাই করে

## ১.১২ Debugging Strategy
- Django: `django-debug-toolbar` বা `connection.queries` দিয়ে actual query count দেখুন
- Spring Boot: `spring.jpa.show-sql=true` এবং Hibernate statistics enable করে query count মনিটর করুন
- যদি কোনো endpoint-এর query count রেকর্ড সংখ্যার সাথে proportionally বাড়ে (১০ রেকর্ডে ১১ query, ১০০ রেকর্ডে ১০১ query), এটা N+1-এর স্পষ্ট লক্ষণ

## ১.১৩ Optimization Techniques
- `select_related` (Foreign Key, OneToOne — SQL JOIN ব্যবহার করে)
- `prefetch_related` (ManyToMany, reverse FK — batch IN query ব্যবহার করে)
- GraphQL/REST batch resolvers-এ **DataLoader** pattern
- `only()`/`values()` দিয়ে শুধু প্রয়োজনীয় column আনা (অতিরিক্ত ডেটা না টানা)

## ১.১৪ Trade-offs (কখন এড়িয়ে যাবেন)
অতিরিক্ত `prefetch_related` ব্যবহার করলে memory খরচ বাড়ে কারণ সব related ডেটা আগেই মেমোরিতে লোড হয়ে যায় — যদি আসলে সেই related ডেটা ব্যবহারই না হয়, এটা অপ্রয়োজনীয় ওভারহেড তৈরি করে। তাই blindly সব জায়গায় prefetch না করে actual usage অনুযায়ী optimize করা উচিত।

## ১.১৫ Production Best Practices
- প্রতিটা list endpoint-এর জন্য query count নির্দিষ্ট এবং constant রাখা (record সংখ্যার সাথে স্কেল না করা) — এটা একটা automated test দিয়ে enforce করা যায় (`assertNumQueries`)
- CI/CD pipeline-এ query count regression test যুক্ত করুন

## ১.১৬ Security Considerations
N+1 নিজে security ইস্যু না, কিন্তু এটা একটা **resource exhaustion / DoS vector** হতে পারে — যদি কোনো attacker জানে একটা endpoint N+1 সমস্যাযুক্ত, সে বড় ডেটাসেট রিকোয়েস্ট করে DB-কে overload করে দিতে পারে।

## ১.১৭ Senior Engineer Design Mindset
সিনিয়র ইঞ্জিনিয়ার প্রতিটা loop-এর ভেতরে DB access দেখলেই সতর্ক হয়ে যায় এবং নিজেকে প্রশ্ন করে: *"এই loop-টা কি একটা batch query-তে রূপান্তর করা সম্ভব?"*

---

## 🧠 ইন্টারভিউ সেকশন: N+1 Query Problem

### 🔹 Beginner প্রশ্ন
1. **N+1 problem কী?**
   *উত্তর*: যখন একটা মূল query করে N-টা রেকর্ড আনা হয়, তারপর প্রতিটা রেকর্ডের জন্য আলাদা করে একটা related ডেটার query করা হয় — মোট ১ + N টা query হয়ে যায়, যেখানে এটা ১-২টা query-তেই সম্ভব হতো।
2. **select_related কী কাজ করে?**
   *উত্তর*: এটা SQL JOIN ব্যবহার করে Foreign Key/OneToOne related ডেটা একটা single query-তেই নিয়ে আসে।
3. **কীভাবে বুঝবেন আপনার API-তে N+1 সমস্যা আছে?**
   *উত্তর*: Query log/debug toolbar দেখে query count চেক করুন — যদি record সংখ্যা বাড়ার সাথে query count proportionally বাড়ে, তাহলে N+1 আছে।

### 🔹 Intermediate প্রশ্ন
1. **select_related আর prefetch_related-এর পার্থক্য কী?**
   *উত্তর*: `select_related` SQL JOIN ব্যবহার করে এক query-তে সব আনে (Foreign Key/OneToOne-এর জন্য উপযোগী)। `prefetch_related` আলাদা একটা batch query করে Python-level-এ ম্যাপ করে (ManyToMany/reverse FK-এর জন্য, যেখানে JOIN করলে duplicate row সমস্যা হতো)।
2. **GraphQL-এ N+1 সমস্যা কীভাবে দেখা দেয় এবং DataLoader কীভাবে সমাধান করে?**
   *উত্তর*: প্রতিটা resolver স্বাধীনভাবে চলে, তাই প্রতিটা field resolve করার সময় আলাদা DB কল হতে পারে। DataLoader একটা request-এর মধ্যে সব pending কল-গুলো ব্যাচ করে রাখে এবং একটা single tick-এর শেষে সবগুলো একসাথে একটা batch query-তে পাঠায়।
3. **N+1 সমস্যাটা কেন লোকাল ডেভেলপমেন্টে প্রায়ই অলক্ষিত থাকে?**
   *উত্তর*: লোকাল ডেটাসেট সাধারণত ছোট (১০-২০ রেকর্ড), তাই ১১-২১টা query-র extra latency খালি চোখে noticeable হয় না। প্রোডাকশনে হাজার হাজার রেকর্ডে এই সমস্যা ম্যাগনিফাই হয়ে যায়।

### 🔹 Advanced/Senior প্রশ্ন
1. **একটা ৩-লেভেল nested relationship (Post -> Comments -> Replies) এ N+1 কীভাবে handle করবেন?**
   *উত্তর*: প্রতিটা লেভেলের জন্য `prefetch_related` chain করতে হবে — Django-তে `Prefetch` object ব্যবহার করে নেস্টেড prefetch ডিজাইন করা যায় (`prefetch_related(Prefetch('comments', queryset=Comment.objects.prefetch_related('replies')))`)। লক্ষ্য রাখতে হবে এতে query count বাড়ে না, বরং প্রতিটা লেভেলে একটা করে batch query হয়।
2. **N+1 সমাধান করার পরেও যদি performance খারাপ থাকে, পরের ধাপ কী হবে?**
   *উত্তর*: এরপর caching layer (Redis) যুক্ত করা, প্রয়োজনীয় column-এ database index চেক করা, এবং query execution plan (`EXPLAIN ANALYZE`) দেখে আরও deep-level optimization করা উচিত।
3. **Production-এ N+1 regression কীভাবে প্রতিরোধ করবেন একটা বড় টিমে যেখানে অনেকে কোড পুশ করে?**
   *উত্তর*: CI pipeline-এ automated query-count assertion test যুক্ত করা (যেমন Django-র `assertNumQueries`), এবং APM টুল (New Relic, Datadog) দিয়ে production-এ প্রতিটা endpoint-এর average query count মনিটর করে threshold-ভিত্তিক alert সেটআপ করা।

### 🔹 FAANG-Level System Design প্রশ্ন
1. **Instagram-এর মতো একটা feed system ডিজাইন করুন যেখানে প্রতিটা পোস্টের সাথে author info, like count, এবং top ৩টা কমেন্ট দেখাতে হবে — N+1 এড়িয়ে কীভাবে এটা efficient রাখবেন?**
   *উত্তর*: Author info-র জন্য `select_related`, like count-এর জন্য denormalized counter field (annotation বা cached counter, প্রতিবার count() না করে), এবং top কমেন্টের জন্য একটা batch query (`prefetch_related` with limited queryset) — সব মিলিয়ে এক রিকোয়েস্টে fixed সংখ্যক query (৩-৪টা), record সংখ্যার সাথে না বেড়ে।

### 🔹 Scenario-Based প্রশ্ন
1. **প্রোডাকশনে একটা API রেসপন্স টাইম গত সপ্তাহে ২০০ms ছিল, এখন ২ সেকেন্ড হয়ে গেছে — কোনো কোড পরিবর্তন হয়নি কিন্তু ডেটা বেড়েছে। কী হতে পারে এবং কীভাবে ডিবাগ করবেন?**
   *উত্তর*: এটা ক্লাসিক N+1 প্যাটার্ন যা ডেটা বাড়ার সাথে স্কেল করেছে। প্রথমে query log/APM দিয়ে query count চেক করব, কনফার্ম করব এটা record-count-এর সাথে proportional কিনা, তারপর `select_related`/`prefetch_related` যুক্ত করে fix করব এবং একই load দিয়ে আবার টেস্ট করব।

---

## 🛠 Hands-On Practice
- একটা `Post`-`Author`-`Comment` মডেল বানান, ইচ্ছাকৃতভাবে N+1 সমস্যা তৈরি করুন, তারপর `select_related`/`prefetch_related` দিয়ে fix করুন এবং Django Debug Toolbar-এ query count-এর পার্থক্য দেখুন
- Spring Boot-এ JPA lazy loading দিয়ে একই সমস্যা reproduce করুন এবং `JOIN FETCH` দিয়ে সমাধান করুন

---

# 📌 টপিক ২: Caching (Redis)

## ২.১ কেন শিখব?
Caching হলো performance optimization-এর সবচেয়ে high-impact টেকনিক — সঠিক caching strategy একটা ১ সেকেন্ডের API কে ১ms-এ নামিয়ে আনতে পারে। কিন্তু ভুল caching ডেটা corruption এবং stale data সমস্যা তৈরি করে, তাই এটা সাবধানে শেখা জরুরি।

## ২.২ এটা কোন সমস্যা সমাধান করে?
একই ডেটার জন্য বারবার expensive operation (DB query, complex calculation, external API call) করার বদলে, একবার ফলাফল হিসাব করে মেমোরিতে রেখে দেওয়া, যাতে পরের রিকোয়েস্টগুলো সরাসরি সেই সংরক্ষিত ফলাফল থেকে দ্রুত উত্তর পায়।

## ২.৩ Production Failure যদি Ignore করি
একটা e-commerce সাইটের হোমপেজে যদি প্রতিটা রিকোয়েস্টে "today's deals" লিস্ট সরাসরি DB থেকে complex query করে আনা হয়, এবং একসাথে ১০,০০০ ইউজার পেজ লোড করে (যেমন Black Friday sale), DB-তে identical query ১০,০০০ বার হিট করবে — অথচ ডেটা একই। এটা সহজেই DB-কে অভারলোড করে পুরো সাইট ডাউন করে দিতে পারে।

## ২.৪ Real-World Examples
- **Twitter/X**: Celebrity-দের টুইট কোটি কোটি বার ভিউ হয় — প্রতিটা ভিউ-এর জন্য DB hit করলে সিস্টেম ভেঙে যেত, তাই এগুলো heavily cached থাকে
- **Amazon**: প্রোডাক্ট পেজের price/inventory ডেটা প্রায়ই Redis-এ cache করা থাকে কারণ এটা মিলিয়ন মিলিয়ন বার রিড হয় কিন্তু কম ফ্রিকোয়েন্সিতে আপডেট হয়
- **Facebook**: তাদের নিজস্ব Memcached-এর একটা বিশাল distributed deployment আছে যা billions of request per second handle করতে সাহায্য করে

## ২.৫ Concept ব্যাখ্যা

**Cache Strategy গুলো:**

| Strategy | কীভাবে কাজ করে | কখন ব্যবহার করবেন |
|---|---|---|
| Cache-Aside (Lazy Loading) | App প্রথমে cache চেক করে, না পেলে DB থেকে আনে এবং cache-এ লেখে | সবচেয়ে common, general purpose |
| Write-Through | লেখার সময় cache আর DB দুটোতেই একসাথে লেখা হয় | যখন read consistency খুব গুরুত্বপূর্ণ |
| Write-Behind (Write-Back) | শুধু cache-এ লেখা হয়, পরে async-ভাবে DB-তে sync হয় | উচ্চ-write throughput প্রয়োজন হলে |
| Read-Through | Cache layer নিজেই DB থেকে ডেটা fetch করার দায়িত্ব নেয় | Cache library/abstraction layer থাকলে |

```
Cache-Aside Flow:
Client -> App: GET /product/1
App -> Redis: GET product:1
   ├── Cache HIT  -> Redis থেকে রিটার্ন (দ্রুত, ~1ms)
   └── Cache MISS -> App -> DB query -> App -> Redis-এ লেখে (TTL সহ) -> Client-কে রিটার্ন
```

## ২.৬ Django REST Framework Implementation

```python
from django.core.cache import cache
import redis

class ProductDetailView(APIView):
    def get(self, request, pk):
        cache_key = f"product:{pk}"
        cached_data = cache.get(cache_key)

        if cached_data:
            return Response(cached_data)  # Cache HIT

        # Cache MISS - DB থেকে আনতে হবে
        product = Product.objects.select_related('category').get(pk=pk)
        data = ProductSerializer(product).data

        cache.set(cache_key, data, timeout=300)  # 5 মিনিট TTL
        return Response(data)

    def put(self, request, pk):
        # Update করার সময় cache invalidate করতে হবে — না হলে stale data থাকবে
        product = Product.objects.get(pk=pk)
        serializer = ProductSerializer(product, data=request.data)
        serializer.is_valid(raise_exception=True)
        serializer.save()
        cache.delete(f"product:{pk}")  # Cache invalidation
        return Response(serializer.data)
```

```python
# settings.py - Redis কনফিগারেশন
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
        'OPTIONS': {'CLIENT_CLASS': 'django_redis.client.DefaultClient'}
    }
}
```

## ২.৭ Spring Boot Equivalent

```java
@Service
public class ProductService {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    public ProductDTO getProduct(Long id) {
        String cacheKey = "product:" + id;
        ProductDTO cached = (ProductDTO) redisTemplate.opsForValue().get(cacheKey);

        if (cached != null) {
            return cached;  // Cache HIT
        }

        ProductDTO product = productRepository.findByIdWithCategory(id);  // Cache MISS
        redisTemplate.opsForValue().set(cacheKey, product, Duration.ofMinutes(5));
        return product;
    }

    public ProductDTO updateProduct(Long id, ProductDTO dto) {
        ProductDTO updated = productRepository.update(id, dto);
        redisTemplate.delete("product:" + id);  // Cache invalidation
        return updated;
    }
}

// অথবা Spring-এর built-in annotation দিয়ে সহজভাবে
@Cacheable(value = "products", key = "#id")
public ProductDTO getProduct(Long id) { ... }

@CacheEvict(value = "products", key = "#id")
public ProductDTO updateProduct(Long id, ProductDTO dto) { ... }
```

## ২.৮ Internal Request Lifecycle
1. Request আসে View/Controller-এ
2. App প্রথমে Redis-কে একটা GET কমান্ড পাঠায় (in-memory key-value lookup, সাব-মিলিসেকেন্ডে রেসপন্স)
3. যদি key পাওয়া যায় (cache hit), deserialize করে সরাসরি response পাঠানো হয় — DB একদমই touch হয় না
4. যদি key না পাওয়া যায় (cache miss), DB query execute হয়, ফলাফল serialize করে Redis-এ SET কমান্ডের মাধ্যমে TTL সহ সংরক্ষণ করা হয়, তারপর response পাঠানো হয়

## ২.৯ Architecture Diagram

```
                  +-----------+
   Request ----> | App Server | <---- Response
                  +-----------+
                    |       |
              (1) Check    (3) Write on miss
                    |       |
                  +-----------+
                  |   Redis   |  <- in-memory, সাব-মিলিসেকেন্ড latency
                  +-----------+
                    |
              (2) Miss হলেই
                    |
                  +-----------+
                  | Database  |  <- ধীর, কিন্তু source of truth
                  +-----------+
```

## ২.১০ Performance Impact
Database query সাধারণত ১০-১০০ms লাগে (complex query-তে আরও বেশি), কিন্তু Redis lookup সাধারণত ১ms-এর নিচে। একটা high-traffic endpoint-এ যদি ৯৫% cache hit rate থাকে, average response time-এ বিশাল উন্নতি আসে এবং DB-এর উপর load অনেকটাই কমে যায়।

## ২.১১ Common Mistakes (Junior vs Senior)
- **জুনিয়র**: ডেটা cache করে কিন্তু update/delete অপারেশনে cache invalidate করতে ভুলে যায় — ফলে ইউজাররা পুরনো (stale) ডেটা দেখতে থাকে
- **সিনিয়র**: cache invalidation strategy আগে থেকেই ডিজাইন করে (TTL + explicit invalidation on write), এবং কখন cache stampede হতে পারে (একসাথে অনেক রিকোয়েস্ট miss হয়ে একসাথে DB hit করা) তা বিবেচনা করে সমাধান (locking/jitter) রাখে

## ২.১২ Debugging Strategy
- Redis `MONITOR` কমান্ড দিয়ে real-time দেখুন কোন key-গুলো GET/SET হচ্ছে
- Cache hit/miss ratio metric track করুন (Redis `INFO stats` কমান্ডে `keyspace_hits`/`keyspace_misses` পাওয়া যায়)
- যদি ইউজার stale data রিপোর্ট করে, প্রথমে চেক করুন cache invalidation logic ঠিকমতো trigger হচ্ছে কিনা update path-এ

## ২.১৩ Optimization Techniques
- **TTL জিটার**: সব key-র TTL একদম same না রেখে একটা ছোট random variation যুক্ত করুন, যাতে একসাথে অনেক key expire হয়ে cache stampede না তৈরি হয়
- **Cache Warming**: Deployment-এর পর প্রথম রিকোয়েস্টেই cache miss না হোক, সেজন্য আগে থেকেই popular ডেটা cache-এ load করে রাখা
- **Compression**: বড় object Redis-এ রাখার আগে compress করলে memory খরচ কমে

## ২.১৪ Trade-offs (কখন এড়িয়ে যাবেন)
যদি ডেটা প্রতি সেকেন্ডে পরিবর্তিত হয় (যেমন stock trading price), caching বরং stale data দেখানোর ঝুঁকি বাড়ায় — এই ক্ষেত্রে caching না করে অথবা খুব ছোট TTL (sub-second) ব্যবহার করতে হবে। এছাড়া caching একটা অতিরিক্ত infrastructure component (Redis) যুক্ত করে, যা operational complexity বাড়ায়।

## ২.১৫ Production Best Practices
- Cache key naming convention সুসংগঠিত রাখুন (`resource_type:id:field`)
- প্রতিটা cache entry-তে অবশ্যই একটা TTL সেট করুন — কখনো infinite cache রাখবেন না
- Cache layer down হয়ে গেলেও সিস্টেম graceful-ভাবে DB-তে fallback করতে পারে এমন ডিজাইন রাখুন

## ২.১৬ Security Considerations
Redis ডিফল্টভাবে authentication ছাড়া চলে — production-এ অবশ্যই password (`requirepass`) সেট করুন এবং network-level access control (firewall, VPC) দিয়ে শুধু application server-কেই access দিন, কারণ open Redis instance একটা পরিচিত attack vector।

## ২.১৭ Senior Engineer Design Mindset
সিনিয়র ইঞ্জিনিয়ার caching যুক্ত করার আগে প্রশ্ন করে: *"এই ডেটা কতবার read হয় বনাম write হয়? Stale data দেখানো কতটা acceptable? Cache miss হলে সিস্টেম কীভাবে graceful-ভাবে fallback করবে?"*

---

## 🧠 ইন্টারভিউ সেকশন: Caching (Redis)

### 🔹 Beginner প্রশ্ন
1. **Caching কী এবং কেন এটা performance বাড়ায়?**
   *উত্তর*: Caching মানে একবার হিসাব করা ডেটা মেমোরিতে রেখে দেওয়া যাতে পরের বার আবার সেই ভারী operation (DB query) না করতে হয়। মেমোরি অ্যাক্সেস ডিস্ক/DB অ্যাক্সেসের চেয়ে বহুগুণ দ্রুত, তাই এটা response time নাটকীয়ভাবে কমায়।
2. **Redis কী এবং এটা কেন এত জনপ্রিয় caching solution?**
   *উত্তর*: Redis একটা in-memory key-value store যা অত্যন্ত দ্রুত (সাব-মিলিসেকেন্ড) রিড/রাইট দিতে পারে, এবং বিভিন্ন ডেটা স্ট্রাকচার (string, list, set, hash) সাপোর্ট করে যা সহজ caching ছাড়াও আরও জটিল ইউজ-কেসে কাজে লাগে।
3. **Cache hit আর cache miss-এর পার্থক্য কী?**
   *উত্তর*: Cache hit মানে requested ডেটা cache-এ পাওয়া গেছে, সরাসরি রিটার্ন করা যায়। Cache miss মানে cache-এ নেই, তাই মূল সোর্স (DB) থেকে আনতে হয়, এবং পরের বারের জন্য cache-এ লিখে রাখা হয়।

### 🔹 Intermediate প্রশ্ন
1. **Cache invalidation কী এবং এটা কেন কঠিন বলা হয় ("There are only two hard things in Computer Science: cache invalidation and naming things")?**
   *উত্তর*: Cache invalidation মানে যখন আসল ডেটা পরিবর্তন হয়, পুরনো cache entry-কে সরিয়ে ফেলা বা আপডেট করা। এটা কঠিন কারণ distributed সিস্টেমে কখন এবং কোন cache entry invalidate করতে হবে তা track করা জটিল — ভুল হলে stale data দেখানো হয়, যা সাবটল বাগ তৈরি করে যা ডিটেক্ট করা কঠিন।
2. **TTL (Time To Live) কী এবং এটার মান কীভাবে ঠিক করবেন?**
   *উত্তর*: TTL হলো একটা cache entry কতক্ষণ valid থাকবে তার সময়সীমা, এর পরে স্বয়ংক্রিয়ভাবে expire হয়ে যায়। এটা নির্ভর করে ডেটা কত ফ্রিকোয়েন্টলি পরিবর্তন হয় তার উপর — ঘন ঘন পরিবর্তনশীল ডেটায় ছোট TTL, কম পরিবর্তনশীল ডেটায় (যেমন product category list) বড় TTL ব্যবহার করা হয়।
3. **Cache Stampede কী এবং কীভাবে প্রতিরোধ করবেন?**
   *উত্তর*: Cache Stampede ঘটে যখন একটা popular cache key একই সময়ে expire হয়ে যায় এবং একসাথে হাজার হাজার রিকোয়েস্ট cache miss পেয়ে একসাথে DB-তে হিট করে, DB-কে অভারলোড করে দেয়। প্রতিরোধের উপায়: TTL-এ jitter (random variation) যুক্ত করা, distributed lock ব্যবহার করে শুধু একটা রিকোয়েস্টকে DB query করতে দেওয়া এবং বাকিদের wait করানো (lock-and-recompute pattern)।

### 🔹 Advanced/Senior প্রশ্ন
1. **Write-Through আর Cache-Aside strategy-র মধ্যে আপনি কোনটা বেছে নেবেন একটা high-write system-এ এবং কেন?**
   *উত্তর*: High-write system-এ Write-Through প্রতিবার write-এ cache আপডেট করার extra latency যুক্ত করে, যা throughput কমিয়ে দিতে পারে। Cache-Aside-এ write শুধু DB-তে যায় এবং cache invalidate করা হয় (পরের read-এ lazily পুনরায় লোড হয়) — এটা write path-কে দ্রুত রাখে। তবে যদি read consistency খুবই critical হয় (financial data), Write-Through বা Write-Behind বিবেচনা করতে হবে।
2. **Distributed সিস্টেমে multiple cache node-এর মধ্যে consistency কীভাবে মেইনটেইন করবেন?**
   *উত্তর*: Redis Cluster-এর মতো distributed cache-এ consistent hashing ব্যবহার করে key-গুলো node-এ বিতরণ করা হয়। Strong consistency সাধারণত sacrifice করা হয় availability এবং performance-এর জন্য (CAP theorem অনুযায়ী) — eventual consistency accept করা হয়, এবং critical write-এর পর explicit cache invalidation broadcast করা হয় সব node-এ।
3. **আপনি কীভাবে decide করবেন কোন ডেটা cache করার উপযোগী এবং কোনটা না?**
   *উত্তর*: যেই ডেটা frequently read হয় কিন্তু infrequently write হয় (high read-to-write ratio), সেটা caching-এর প্রধান candidate। এছাড়া যে ডেটা compute-করতে expensive (complex aggregation, external API call), সেটাও caching-যোগ্য। কিন্তু highly volatile বা strictly consistent দরকার এমন ডেটা (real-time stock price, account balance) caching-এর জন্য ঝুঁকিপূর্ণ।

### 🔹 FAANG-Level System Design প্রশ্ন
1. **Twitter-এর মতো একটা সিস্টেম ডিজাইন করুন যেখানে একজন সেলিব্রিটির টুইট মিলিয়ন মিলিয়ন বার ভিউ হয় — caching strategy কী হবে?**
   *উত্তর*: টুইট কনটেন্ট নিজে Redis-এ heavily cache করা হবে দীর্ঘ TTL সহ (কারণ টুইট তৈরির পর rarely পরিবর্তন হয়)। Like/Retweet count-এর মতো দ্রুত পরিবর্তনশীল counter আলাদাভাবে handle করা হবে — সরাসরি DB-তে increment না করে Redis counter ব্যবহার করে periodically DB-তে sync করা হবে (write-behind pattern), যাতে হাই-ফ্রিকোয়েন্সি write DB-কে অভারলোড না করে।
2. **একটা multi-region সিস্টেমে (US আর Europe) caching ডিজাইন করার সময় কী কী চ্যালেঞ্জ সামনে আসবে?**
   *উত্তর*: প্রতিটা region-এর নিজস্ব local cache থাকলে latency কমে, কিন্তু region-গুলোর মধ্যে cache consistency বজায় রাখা কঠিন — একটা region-এ write হলে অন্য region-এর cache invalidate করার জন্য একটা cross-region pub/sub mechanism (Redis Pub/Sub বা message queue) দরকার হয়, যা কিছুটা eventual consistency lag তৈরি করে যা business requirement-এর সাথে balance করতে হয়।

### 🔹 Scenario-Based প্রশ্ন
1. **ইউজাররা রিপোর্ট করছে যে প্রোডাক্ট আপডেট করার পরেও পুরনো price দেখাচ্ছে — root cause কী হতে পারে এবং কীভাবে ফিক্স করবেন?**
   *উত্তর*: এটা cache invalidation মিসিং হওয়ার ক্লাসিক উদাহরণ — update endpoint DB আপডেট করছে কিন্তু সংশ্লিষ্ট cache key delete/update করছে না। ফিক্স: update logic-এ explicit `cache.delete()` যুক্ত করা, এবং ভবিষ্যতে এই বাগ এড়াতে একটা integration test লেখা যা যাচাই করে update-এর পর cache সঠিকভাবে invalidate হয়েছে কিনা।

---

## 🛠 Hands-On Practice
- DRF-এ একটা product detail endpoint বানান cache-aside pattern দিয়ে, এবং update হলে cache invalidate করুন
- ইচ্ছাকৃতভাবে cache stampede সিমুলেট করুন (একসাথে অনেক রিকোয়েস্ট পাঠিয়ে একটা key expire হওয়ার মুহূর্তে) এবং TTL jitter দিয়ে সমাধান করুন

---

# 📌 টপিক ৩: Query Optimization

## ৩.১ কেন শিখব?
একটা ভালো ডিজাইন করা API-ও যদি ভিতরে অপ্টিমাইজ-না-করা database query চালায়, পুরো সিস্টেম স্লো হয়ে যাবে। Query optimization হলো সেই স্কিল যা একজন সিনিয়র ব্যাকএন্ড ইঞ্জিনিয়ারকে জুনিয়র থেকে আলাদা করে।

## ৩.২ এটা কোন সমস্যা সমাধান করে?
Database engine একটা query কীভাবে execute করবে তার একাধিক উপায় থাকতে পারে (different execution plan) — কোনটা সবচেয়ে কম সময়ে, কম resource ব্যবহার করে ফলাফল দিতে পারে, তা নির্ধারণ করাই query optimization।

## ৩.৩ Production Failure যদি Ignore করি
একটা টেবিলে ১০ মিলিয়ন রো আছে এবং একটা column-এ index নেই, তাহলে `WHERE email = '...'` খুঁজতে DB-কে পুরো টেবিল scan করতে হবে (full table scan) — যা সেকেন্ডে চলে যেতে পারে, যেখানে একটা index থাকলে এটা মিলিসেকেন্ডে সম্পন্ন হতো। হাই-ট্রাফিক সিস্টেমে এটা DB CPU exhaust করে cascading failure তৈরি করতে পারে।

## ৩.৪ Real-World Examples
- **LinkedIn**: তাদের member search feature বিশাল ডেটাসেটে দ্রুত রাখতে composite index এবং query rewriting ব্যাপকভাবে ব্যবহার করে
- **Uber**: তাদের geo-spatial query (কাছের ড্রাইভার খোঁজা) optimize করতে বিশেষ spatial index (geohashing, quad-tree) ব্যবহার করে কারণ সাধারণ B-tree index এই ইউজ-কেসে যথেষ্ট দ্রুত না
- **Pinterest**: তাদের ডেটাবেস স্কেলিং সমস্যার একটা বড় অংশ সমাধান হয়েছিল ভালো indexing strategy এবং query rewriting দিয়ে, আগে নতুন hardware-এ ইনভেস্ট করার বদলে

## ৩.৫ Concept ব্যাখ্যা

**মূল optimization টেকনিকগুলো:**
- **Indexing**: একটা column-এ index থাকলে DB binary-search-এর মতো দ্রুত খুঁজে পায়, full table scan এড়ানো যায়
- **EXPLAIN/EXPLAIN ANALYZE**: Query execution plan দেখায় — DB কি index ব্যবহার করছে, নাকি full scan করছে
- **Avoiding SELECT \***: প্রয়োজনীয় column-ই select করা, অতিরিক্ত ডেটা transfer এড়ানো
- **Composite Index**: একাধিক column একসাথে frequently filter হলে, সেগুলোর উপর একটা যুগ্ম index তৈরি করা

```sql
-- Index ছাড়া (Full Table Scan - স্লো)
EXPLAIN SELECT * FROM orders WHERE customer_email = 'a@b.com';
-- Seq Scan on orders  (cost=0.00..50000.00 rows=1)

-- Index তৈরি করার পর
CREATE INDEX idx_customer_email ON orders(customer_email);
EXPLAIN SELECT * FROM orders WHERE customer_email = 'a@b.com';
-- Index Scan using idx_customer_email  (cost=0.42..8.44 rows=1)
```

## ৩.৬ Django REST Framework Implementation

```python
# models.py - Index ডিফাইন করা
class Order(models.Model):
    customer_email = models.EmailField(db_index=True)  # Single column index
    status = models.CharField(max_length=20)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        indexes = [
            models.Index(fields=['status', 'created_at']),  # Composite index
        ]

# views.py - শুধু প্রয়োজনীয় ফিল্ড আনা (SELECT * এড়ানো)
class OrderListView(APIView):
    def get(self, request):
        # ❌ খারাপ: সব column আনছে
        # orders = Order.objects.all()

        # ✅ ভালো: শুধু প্রয়োজনীয় ফিল্ড
        orders = Order.objects.filter(status='pending').only('id', 'customer_email', 'created_at')
        return Response(list(orders.values()))
```

```python
# Django-তে query execution plan দেখা
print(Order.objects.filter(status='pending').explain())
```

## ৩.৭ Spring Boot Equivalent

```java
// Entity-এ index ডিফাইন করা
@Entity
@Table(name = "orders", indexes = {
    @Index(name = "idx_customer_email", columnList = "customerEmail"),
    @Index(name = "idx_status_created", columnList = "status, createdAt")
})
public class Order {
    private String customerEmail;
    private String status;
    private LocalDateTime createdAt;
}

// শুধু প্রয়োজনীয় ফিল্ড আনার জন্য projection ব্যবহার
public interface OrderSummary {
    Long getId();
    String getCustomerEmail();
    LocalDateTime getCreatedAt();
}

@Query("SELECT o.id as id, o.customerEmail as customerEmail, o.createdAt as createdAt FROM Order o WHERE o.status = :status")
List<OrderSummary> findSummaryByStatus(@Param("status") String status);
```

## ৩.৮ Internal Request Lifecycle
1. Query DB engine-এ পৌঁছালে, প্রথমে **query planner/optimizer** চালু হয়, যা বিভিন্ন execution strategy বিশ্লেষণ করে (index ব্যবহার করবে নাকি full scan, কোন join order ব্যবহার করবে)
2. Planner table statistics (row count, value distribution) দেখে সবচেয়ে কম-cost প্ল্যান বেছে নেয়
3. সিলেক্টেড প্ল্যান অনুযায়ী actual execution হয় — ডিস্ক I/O বা memory buffer থেকে ডেটা read হয়
4. ফলাফল অ্যাপ্লিকেশনে রিটার্ন হয়

## ৩.৯ Architecture Diagram

```
[Query] -> [Query Parser] -> [Query Optimizer/Planner]
                                      |
                        বিভিন্ন execution plan মূল্যায়ন
                                      |
                    Index আছে? ----- Yes -----> [Index Scan] (দ্রুত)
                                      |
                                     No
                                      |
                              [Sequential/Full Table Scan] (স্লো)
                                      |
                              [Result Set] -> [Application]
```

## ৩.১০ Performance Impact
একটা সঠিক index একটা query-কে O(n) (full scan) থেকে O(log n) (B-tree index lookup)-এ নিয়ে আসতে পারে। ১ মিলিয়ন রো-র টেবিলে এটা সেকেন্ড থেকে মিলিসেকেন্ডে রূপান্তর হতে পারে।

## ৩.১১ Common Mistakes (Junior vs Senior)
- **জুনিয়র**: প্রতিটা column-এ index যুক্ত করে দেয় ভেবে "index মানেই দ্রুত", না বুঝে যে অতিরিক্ত index write performance কমিয়ে দেয় (প্রতিটা INSERT/UPDATE-এ সব index আপডেট করতে হয়)
- **সিনিয়র**: query pattern বিশ্লেষণ করে শুধু সেই column-গুলোতে index দেয় যেগুলো frequently `WHERE`/`JOIN`/`ORDER BY`-তে ব্যবহার হয়, এবং read/write ratio বিবেচনা করে index-এর সংখ্যা ব্যালেন্স করে

## ৩.১২ Debugging Strategy
- ধীর query চিহ্নিত করতে DB-র **slow query log** enable করুন (MySQL: `slow_query_log`, PostgreSQL: `log_min_duration_statement`)
- `EXPLAIN ANALYZE` দিয়ে actual execution time এবং plan দেখুন, "Seq Scan" দেখলে index মিসিং সন্দেহ করুন

## ৩.১৩ Optimization Techniques
- Composite index design করুন যেখানে multiple column একসাথে filter হয় (column order গুরুত্বপূর্ণ — সবচেয়ে selective column আগে)
- `COUNT(*)` এর বদলে existing approximate count বা cached counter ব্যবহার করুন বড় টেবিলে
- Pagination-এ `OFFSET` বড় হলে স্লো হয়ে যায় — cursor-based pagination বিবেচনা করুন (পরের টপিকে বিস্তারিত)

## ৩.১৪ Trade-offs (কখন এড়িয়ে যাবেন)
Write-heavy টেবিলে অতিরিক্ত index থাকলে প্রতিটা write operation স্লো হয়ে যায় কারণ প্রতিটা index আপডেট করতে হয়। তাই index তৈরি করার আগে read vs write frequency বিবেচনা করতে হবে।

## ৩.১৫ Production Best Practices
- প্রতিটা নতুন query ডিপ্লয় করার আগে `EXPLAIN ANALYZE` দিয়ে যাচাই করুন
- Database monitoring-এ slow query alert সেটআপ করুন (যেমন একটা query ১০০ms-এর বেশি সময় নিলে এলার্ট)
- Migration-এর সময় বড় টেবিলে index তৈরি করলে এটা locking সমস্যা তৈরি করতে পারে — `CONCURRENTLY` (PostgreSQL) বা অনুরূপ non-blocking পদ্ধতি ব্যবহার করুন

## ৩.১৬ Security Considerations
ORM ব্যবহার না করে raw SQL লিখলে SQL injection-এর ঝুঁকি থাকে — সবসময় parameterized query ব্যবহার করুন, string concatenation দিয়ে query বানাবেন না।

## ৩.১৭ Senior Engineer Design Mindset
সিনিয়র ইঞ্জিনিয়ার একটা নতুন ফিচার ডিজাইন করার সময়ই চিন্তা করে কোন query pattern তৈরি হবে এবং সেই অনুযায়ী আগে থেকেই index ডিজাইন করে — পরে production-এ সমস্যা হওয়ার পর reactive fix করার বদলে proactive ডিজাইন করে।

---

## 🧠 ইন্টারভিউ সেকশন: Query Optimization

### 🔹 Beginner প্রশ্ন
1. **Database index কী এবং এটা কীভাবে কাজ করে?**
   *উত্তর*: Index একটা আলাদা ডেটা স্ট্রাকচার (সাধারণত B-tree) যা একটা column-এর ভ্যালু-গুলো সাজানো অবস্থায় রাখে, যাতে DB পুরো টেবিল scan না করে সরাসরি দ্রুত খুঁজে পেতে পারে — ঠিক যেমন একটা বইয়ের ইনডেক্স পেজ দিয়ে আপনি পুরো বই না পড়েই নির্দিষ্ট টপিক খুঁজে পান।
2. **SELECT \* কেন এড়ানো উচিত?**
   *উত্তর*: এটা প্রয়োজনের চেয়ে বেশি ডেটা DB থেকে অ্যাপ্লিকেশনে নিয়ে আসে, যা network bandwidth এবং memory খরচ বাড়ায়, এবং DB-কে অপ্রয়োজনীয় column পড়তে বাধ্য করে।
3. **EXPLAIN কমান্ড কী কাজে লাগে?**
   *উত্তর*: এটা দেখায় DB কীভাবে একটা query execute করার প্ল্যান করছে — index ব্যবহার হচ্ছে কিনা, কত রো scan হচ্ছে — যা optimization-এর জন্য প্রথম ডায়াগনস্টিক টুল।

### 🔹 Intermediate প্রশ্ন
1. **Composite index-এ column-এর order কেন গুরুত্বপূর্ণ?**
   *উত্তর*: Composite index leftmost-prefix নিয়মে কাজ করে — যদি `(status, created_at)` index থাকে, এটা `status` দিয়ে filter করা query-তে ব্যবহার হবে, কিন্তু শুধু `created_at` দিয়ে filter করলে এই index ব্যবহার হবে না। তাই সবচেয়ে বেশি ব্যবহৃত/selective column আগে রাখা উচিত।
2. **অতিরিক্ত index থাকলে কী সমস্যা হতে পারে?**
   *উত্তর*: প্রতিটা INSERT/UPDATE/DELETE অপারেশনে সব সংশ্লিষ্ট index আপডেট করতে হয়, ফলে write performance কমে যায় এবং অতিরিক্ত storage space ব্যবহৃত হয়। তাই unused/redundant index periodically review করে সরিয়ে ফেলা উচিত।
3. **N+1 problem আর Query Optimization-এর মধ্যে সম্পর্ক কী?**
   *উত্তর*: N+1 হলো query *সংখ্যা*-র সমস্যা (অনেকগুলো ছোট query), আর Query Optimization হলো প্রতিটা individual query কতটা efficient তার সমস্যা (index, execution plan)। দুটোই dependent — N+1 সমাধান করার পরও যদি সেই batch query-টা নিজেই index ছাড়া স্লো হয়, পুরো সমাধান অসম্পূর্ণ থাকবে।

### 🔹 Advanced/Senior প্রশ্ন
1. **আপনি কীভাবে একটা production database-এ slow query চিহ্নিত করবেন এবং তা ফিক্স করার প্রসেস কী হবে?**
   *উত্তর*: প্রথমে slow query log/APM tool (Datadog, New Relic) দিয়ে সবচেয়ে ধীর এবং সবচেয়ে frequently-called query-গুলো চিহ্নিত করব। তারপর `EXPLAIN ANALYZE` দিয়ে execution plan বিশ্লেষণ করে দেখব index মিসিং কিনা, নাকি query structure-ই অদক্ষ (যেমন অপ্রয়োজনীয় subquery)। ফিক্স করার পর staging-এ production-scale ডেটা দিয়ে যাচাই করে তারপর rollout করব, এবং deploy-এর পর monitoring দিয়ে নিশ্চিত করব উন্নতি হয়েছে।
2. **বড় একটা টেবিলে (১০০ মিলিয়ন+ রো) নতুন index তৈরি করার সময় কী সতর্কতা নেবেন?**
   *উত্তর*: সাধারণ `CREATE INDEX` টেবিল লক করে ফেলতে পারে যা production traffic ব্লক করবে। PostgreSQL-এ `CREATE INDEX CONCURRENTLY` ব্যবহার করব যা non-blocking ভাবে index তৈরি করে (একটু ধীর হলেও), এবং off-peak সময়ে এই অপারেশন শিডিউল করব।
3. **Query optimization আর caching — এই দুটোর মধ্যে আপনি কীভাবে priority ঠিক করবেন একটা স্লো API ফিক্স করার সময়?**
   *উত্তর*: প্রথমে query optimization করব কারণ এটা root cause ঠিক করে এবং caching ছাড়াও সিস্টেম দ্রুত থাকে — caching একটা band-aid, mandatory fix না। যদি query optimization করার পরও latency target পূরণ না হয় (বিশেষ করে খুব heavy aggregation-এর ক্ষেত্রে), তখন caching যুক্ত করব অতিরিক্ত layer হিসেবে।

### 🔹 FAANG-Level System Design প্রশ্ন
1. **Uber-এর "কাছের ড্রাইভার খুঁজুন" ফিচারের জন্য আপনি কীভাবে database query optimize করবেন যখন মিলিয়ন মিলিয়ন ড্রাইভার লোকেশন রিয়েল-টাইমে আপডেট হচ্ছে?**
   *উত্তর*: সাধারণ B-tree index geo-spatial query (lat/long radius search)-এর জন্য অদক্ষ। এর বদলে geohashing বা quad-tree-ভিত্তিক spatial index (PostGIS, বা Redis Geo commands) ব্যবহার করব, যা দ্রুত "radius-এর মধ্যে" lookup দিতে পারে। এছাড়া driver location আপডেট খুব high-frequency, তাই এটা সরাসরি persistent DB-তে না লিখে প্রথমে in-memory store (Redis)-এ রেখে periodically batch-sync করা যেতে পারে।

### 🔹 Scenario-Based প্রশ্ন
1. **আপনার টিমের একটা রিপোর্টিং query, যা মাসিক sales aggregate করে, প্রোডাকশনে ৩০ সেকেন্ড সময় নিচ্ছে এবং DB-কে ব্লক করছে — কীভাবে handle করবেন?**
   *উত্তর*: প্রথমে `EXPLAIN ANALYZE` দিয়ে bottleneck খুঁজব — সম্ভবত missing index বা inefficient JOIN। দ্রুত mitigation হিসেবে এই heavy query-টা read replica-তে রিডাইরেক্ট করব যাতে primary DB ব্লক না হয়। দীর্ঘমেয়াদী সমাধান হিসেবে এই aggregation pre-compute করে একটা materialized view বা scheduled batch job-এ রূপান্তর করব, যাতে রিয়েল-টাইমে heavy computation না করতে হয়।

---

## 🛠 Hands-On Practice
- একটা টেবিলে ১০,০০০+ ডামি রেকর্ড ইনসার্ট করুন, একটা column-এ index ছাড়া এবং সহ query চালিয়ে `EXPLAIN ANALYZE`-এর পার্থক্য পরিমাপ করুন
- একটা composite index ডিজাইন করুন এমন একটা query-র জন্য যেখানে দুটো column একসাথে filter হয়

---

# 📌 টপিক ৪: Pagination Strategies

## ৪.১ কেন শিখব?
যদি একটা API-তে ১ মিলিয়ন রেকর্ড থাকা টেবিল pagination ছাড়াই পুরো রিটার্ন করার চেষ্টা করে, এটা মেমোরি crash করিয়ে দিতে পারে এবং ক্লায়েন্ট অ্যাপকে freeze করিয়ে দিতে পারে। Pagination হলো বড় ডেটাসেট manageable chunk-এ ভাগ করে দেওয়ার কৌশল।

## ৪.২ এটা কোন সমস্যা সমাধান করে?
একসাথে বিশাল পরিমাণ ডেটা client-কে না পাঠিয়ে, ছোট ছোট "পেজ"-এ ভাগ করে পাঠানো — যাতে response size manageable থাকে এবং ক্লায়েন্ট দ্রুত প্রথম কিছু রেজাল্ট দেখাতে পারে।

## ৪.৩ Production Failure যদি Ignore করি
Instagram-এর মতো একটা ফিড যদি pagination ছাড়া পুরো history একসাথে লোড করার চেষ্টা করে, এটা শুধু সার্ভার মেমোরি crash-ই করবে না, মোবাইল ক্লায়েন্ট অ্যাপও সম্ভবত OutOfMemory crash করবে কারণ কোটি কোটি পোস্ট একসাথে রেন্ডার করার চেষ্টা করবে।

## ৪.৪ Real-World Examples
- **Instagram/Twitter Feed**: Cursor-based (infinite scroll) pagination ব্যবহার করে, কারণ ডেটা ক্রমাগত নতুন যুক্ত হতে থাকে এবং offset-based pagination-এ "duplicate/skip" সমস্যা হতো
- **GitHub API**: Cursor-based pagination ব্যবহার করে (`page` parameter-এর সাথে `Link` header দিয়ে next/prev URL দেয়)
- **Stripe API**: `starting_after`/`ending_before` cursor parameter ব্যবহার করে বড় transaction history paginate করতে

## ৪.৫ Concept ব্যাখ্যা

| Strategy | কীভাবে কাজ করে | সুবিধা | অসুবিধা |
|---|---|---|---|
| Offset-based | `LIMIT 10 OFFSET 20` | বোঝা সহজ, র‍্যান্ডম পেজে jump করা যায় | বড় offset-এ স্লো হয়ে যায়, নতুন ডেটা যুক্ত হলে item skip/duplicate হতে পারে |
| Cursor-based | পরের পেজের জন্য last item-এর একটা reference (cursor) পাঠানো হয় | কনস্ট্যান্ট পারফরম্যান্স, real-time ডেটাতে consistent | র‍্যান্ডম পেজে jump করা যায় না |
| Keyset Pagination | Cursor-based-এর variant, ইনডেক্সড কলামের ভ্যালু (যেমন `id > last_id`) ব্যবহার করে | অত্যন্ত দ্রুত, index-friendly | sorting শুধু indexed column-এ সীমিত |

```sql
-- ❌ Offset-based (বড় offset-এ স্লো)
SELECT * FROM posts ORDER BY created_at DESC LIMIT 10 OFFSET 100000;
-- DB-কে আগের ১,০০,০০০ রো স্কিপ করতে হয়, যদিও সেগুলো রিটার্ন হয় না

-- ✅ Cursor/Keyset-based (দ্রুত, কারণ index সরাসরি ব্যবহার হয়)
SELECT * FROM posts WHERE id < 100000 ORDER BY id DESC LIMIT 10;
```

## ৪.৬ Django REST Framework Implementation

```python
from rest_framework.pagination import PageNumberPagination, CursorPagination

# ❌ Offset-based (সাধারণ কিন্তু বড় ডেটাসেটে স্লো)
class StandardPagination(PageNumberPagination):
    page_size = 20

# ✅ Cursor-based (বড় ডেটাসেট/ফিডের জন্য উপযোগী)
class FeedCursorPagination(CursorPagination):
    page_size = 20
    ordering = '-created_at'  # অবশ্যই একটা ইনডেক্সড, ইউনিক-ইশ ফিল্ড

class PostListView(generics.ListAPIView):
    queryset = Post.objects.all().order_by('-created_at')
    serializer_class = PostSerializer
    pagination_class = FeedCursorPagination
```

## ৪.৭ Spring Boot Equivalent

```java
// ❌ Offset-based
@GetMapping("/posts")
public Page<PostDTO> getPosts(@RequestParam int page, @RequestParam int size) {
    return postRepository.findAll(PageRequest.of(page, size, Sort.by("createdAt").descending()));
}

// ✅ Keyset/Cursor-based
@GetMapping("/posts")
public List<PostDTO> getPosts(@RequestParam(required = false) Long lastId, @RequestParam int size) {
    if (lastId == null) {
        return postRepository.findTop20ByOrderByIdDesc();
    }
    return postRepository.findTop20ByIdLessThanOrderByIdDesc(lastId);
}
```

## ৪.৮ Internal Request Lifecycle
- **Offset-based**: DB engine প্রথম থেকে শুরু করে `OFFSET` সংখ্যক রো স্কিপ করে (এমনকি যদি একটা index থাকে, তাও স্কিপ করা রো-গুলো traverse করতে হয়), তারপর `LIMIT` সংখ্যক রো রিটার্ন করে
- **Cursor/Keyset-based**: DB index-এর মধ্যে সরাসরি `WHERE id < last_id` কন্ডিশন দিয়ে ঠিক জায়গা থেকে শুরু করে, কোনো রো স্কিপ করার প্রয়োজন হয় না — index seek সরাসরি কাঙ্ক্ষিত পয়েন্টে যায়

## ৪.৯ Architecture Diagram

```
Offset-based (পেজ ৫০০০, প্রতি পেজে ২০টা):
[DB] -- স্কিপ করতে হবে প্রথম ৯৯,৯৮০টা রো -- তারপর ২০টা রিটার্ন (স্লো!)

Cursor-based:
[Client পাঠায়: cursor = last_seen_id]
[DB] -- Index seek সরাসরি last_seen_id-এর পরের পজিশনে যায় -- ২০টা রিটার্ন (দ্রুত, কনস্ট্যান্ট টাইম)
```

## ৪.১০ Performance Impact
Offset-based pagination-এ পেজ নম্বর যত বাড়ে, query তত স্লো হয় (linear growth, কারণ DB-কে আগের সব রো স্কিপ করতে হয়)। Cursor-based pagination-এ যেকোনো পেজের জন্য response time প্রায় কনস্ট্যান্ট থাকে, কারণ এটা সরাসরি index দিয়ে নির্দিষ্ট পজিশনে jump করে।

## ৪.১১ Common Mistakes (Junior vs Senior)
- **জুনিয়র**: সবসময় offset-based pagination ব্যবহার করে কারণ এটা বোঝা সহজ, না বুঝে যে এটা বড় ডেটাসেটে এবং real-time changing ডেটাতে সমস্যা তৈরি করে (item skip/duplicate)
- **সিনিয়র**: ব্যবহারের প্রসঙ্গ বিবেচনা করে সঠিক strategy বেছে নেয় — admin dashboard-এ যেখানে নির্দিষ্ট পেজে jump করা প্রয়োজন, সেখানে offset-based ঠিক আছে; কিন্তু infinite-scroll ফিড-এ cursor-based আবশ্যক

## ৪.১২ Debugging Strategy
যদি ইউজাররা রিপোর্ট করে ফিড-এ "একই পোস্ট দুইবার দেখা যাচ্ছে" বা "পোস্ট মিসিং", এটা সাধারণত offset-based pagination-এর সাথে real-time changing ডেটার সংঘাতের লক্ষণ — নতুন আইটেম যুক্ত হলে পরবর্তী পেজের offset শিফট হয়ে যায়।

## ৪.১৩ Optimization Techniques
- Cursor value-কে encode (base64) করে opaque রাখুন, যাতে ক্লায়েন্ট এটার internal structure নিয়ে নির্ভরশীল না হয়ে যায় এবং ভবিষ্যতে implementation পরিবর্তন সহজ থাকে
- প্রতিটা response-এ `has_next`/`next_cursor` field যুক্ত করুন যাতে ক্লায়েন্ট সহজে বুঝতে পারে আরও ডেটা আছে কিনা

## ৪.১৪ Trade-offs (কখন এড়িয়ে যাবেন)
Cursor-based pagination-এ ইউজার সরাসরি "পেজ ৫০-এ যান" করতে পারে না — এটা শুধু sequential next/prev navigation সাপোর্ট করে। তাই যেখানে random page access প্রয়োজন (যেমন একটা admin panel-এ "পেজ নম্বর টাইপ করে যাও"), সেখানে offset-based pagination-ই বেশি উপযোগী, যদিও বড় ডেটাসেটে পারফরম্যান্স ট্রেড-অফ মেনে নিতে হয়।

## ৪.১৫ Production Best Practices
- প্রতিটা list API-তে একটা সর্বোচ্চ page size limit রাখুন (যেমন max ১০০), যাতে কোনো ক্লায়েন্ট ভুলে বা ইচ্ছাকৃতভাবে অতিরিক্ত বড় request না পাঠাতে পারে
- Pagination metadata (total count, next cursor) consistent format-এ রিটার্ন করুন সব endpoint-এ

## ৪.১৬ Security Considerations
যদি `total_count` রিটার্ন করা হয়, নিশ্চিত করুন এটা শুধু authorized ডেটার count দেখাচ্ছে — অন্যথায় এটা তথ্য leak করতে পারে (যেমন কতগুলো "hidden" রেকর্ড আছে তা বুঝে ফেলা)।

## ৪.১৭ Senior Engineer Design Mindset
সিনিয়র ইঞ্জিনিয়ার pagination ডিজাইন করার সময় চিন্তা করে ডেটা কত দ্রুত পরিবর্তিত হয় এবং ক্লায়েন্টের ব্যবহারের প্যাটার্ন কেমন (random access প্রয়োজন, নাকি sequential scroll) — তারপর সঠিক strategy বেছে নেয়, default-ভাবে কোনো একটা পদ্ধতি সব জায়গায় ব্যবহার করে না।

---

## 🧠 ইন্টারভিউ সেকশন: Pagination Strategies

### 🔹 Beginner প্রশ্ন
1. **Pagination কী এবং কেন প্রয়োজন?**
   *উত্তর*: Pagination হলো বড় ডেটাসেটকে ছোট ছোট "পেজ"-এ ভাগ করে ক্লায়েন্টকে পাঠানোর কৌশল, যাতে একসাথে অতিরিক্ত ডেটা ট্রান্সফার এবং মেমোরি ওভারলোড না হয়।
2. **LIMIT এবং OFFSET কী কাজ করে SQL-এ?**
   *উত্তর*: `LIMIT` বলে দেয় কতগুলো রো রিটার্ন করতে হবে, এবং `OFFSET` বলে দেয় শুরু থেকে কতগুলো রো স্কিপ করে তারপর রিটার্ন শুরু করতে হবে।
3. **Cursor-based pagination কী?**
   *উত্তর*: এটা এমন একটা pagination পদ্ধতি যেখানে পেজ নম্বরের বদলে, পরের পেজের জন্য একটা "cursor" (সাধারণত শেষ আইটেমের একটা reference) ব্যবহার করা হয়, যা DB-কে সরাসরি সেই পয়েন্ট থেকে শুরু করতে দেয়।

### 🔹 Intermediate প্রশ্ন
1. **বড় OFFSET ভ্যালুতে offset-based pagination কেন স্লো হয়ে যায়?**
   *উত্তর*: DB engine-কে `OFFSET`-এ উল্লেখিত সংখ্যক রো প্রথমে traverse/স্কিপ করতে হয়, যদিও সেগুলো রিটার্ন হয় না — এই স্কিপ করার কাজে সময় ব্যয় হয়, এবং অফসেট যত বড় হয়, তত বেশি সময় লাগে (linear time complexity)।
2. **Real-time changing ডেটায় (যেমন একটা লাইভ ফিড) কেন offset-based pagination সমস্যা তৈরি করে?**
   *উত্তর*: যদি ব্যবহারকারী পেজ ১ দেখার পর কেউ একটা নতুন আইটেম যুক্ত করে, তাহলে পেজ ২ দেখার সময় সব আইটেম একটা পজিশন শিফট হয়ে যায় — ফলে একটা আইটেম দুইবার দেখা যেতে পারে, অথবা একটা আইটেম স্কিপ হয়ে যেতে পারে। Cursor-based pagination এই সমস্যা এড়ায় কারণ এটা absolute position-এর বদলে relative reference (cursor) ব্যবহার করে।
3. **Pagination response-এ কী কী মেটাডেটা থাকা উচিত একটা ভালো API ডিজাইনে?**
   *উত্তর*: `next_cursor`/`next_page_url`, `has_more`/`has_next`, এবং প্রয়োজনে `total_count` (যদি গণনা করা ব্যয়বহুল না হয়) — যাতে ক্লায়েন্ট সহজে বুঝতে পারে আরও ডেটা আনতে হবে কিনা এবং কীভাবে।

### 🔹 Advanced/Senior প্রশ্ন
1. **Keyset pagination কীভাবে implement করবেন যেখানে sorting একাধিক column-এ (যেমন `created_at` তারপর `id` টাই-ব্রেকার হিসেবে) করতে হয়?**
   *উত্তর*: Composite cursor ব্যবহার করতে হবে — cursor-এ `created_at` এবং `id` দুটোই এনকোড করে রাখব, এবং query-তে `WHERE (created_at, id) < (last_created_at, last_id)` এর মতো একটা row-value comparison ব্যবহার করব, যা ensures করে একই `created_at` ভ্যালু থাকলেও সঠিক ক্রমে এবং duplicate/skip ছাড়া পেজিনেশন হবে।
2. **আপনি কীভাবে একটা API-তে `total_count` দেখাবেন যখন টেবিলে কোটি কোটি রেকর্ড আছে এবং `COUNT(*)` নিজেই ব্যয়বহুল?**
   *উত্তর*: এক্স্যাক্ট কাউন্ট না দিয়ে একটা approximate count ব্যবহার করব (PostgreSQL-এ `pg_class.reltuples` থেকে estimate পাওয়া যায়), বা একটা পৃথক denormalized counter মেইনটেইন করব যা প্রতিটা insert/delete-এ ইনক্রিমেন্ট/ডিক্রিমেন্ট হয়, যাতে রিয়েল-টাইমে এক্সপেনসিভ aggregate query চালাতে না হয়।
3. **Cursor-based pagination-এ cursor কীভাবে নিরাপদ এবং tamper-resistant করবেন?**
   *উত্তর*: Cursor-কে base64-encode করে opaque বানাব এবং ভিতরে একটা signature (HMAC) যুক্ত করব, যাতে ক্লায়েন্ট সরাসরি cursor-এর মান দেখে বা পরিবর্তন করে অন্য ইউজারের ডেটা access করার চেষ্টা করতে না পারে।

### 🔹 FAANG-Level System Design প্রশ্ন
1. **Instagram-এর ইনফিনিট স্ক্রল ফিড ডিজাইন করুন — কোন pagination strategy বেছে নেবেন এবং কেন, বিশেষ করে যখন নতুন পোস্ট প্রতি সেকেন্ডে যুক্ত হচ্ছে?**
   *উত্তর*: Cursor-based pagination অবশ্যই বেছে নেব, কারণ এটা ensure করে যে নতুন পোস্ট যুক্ত হলেও ইউজারের বর্তমান স্ক্রল পজিশন থেকে কোনো item skip/duplicate হবে না — কারণ cursor একটা fixed reference point (যেমন timestamp + id), absolute position না। এছাড়া প্রতিটা পেজের response time কনস্ট্যান্ট থাকবে যতই ইউজার গভীরে স্ক্রল করুক, যা Instagram-এর মতো high-engagement প্রোডাক্টের জন্য অপরিহার্য।

### 🔹 Scenario-Based প্রশ্ন
1. **প্রোডাক্ট টিম রিপোর্ট করছে ইউজাররা ফিড স্ক্রল করার সময় কিছু পোস্ট দুইবার দেখছে — আপনি কীভাবে এটা ডায়াগনোজ করবেন এবং ফিক্স করবেন?**
   *উত্তর*: প্রথমে চেক করব কোন pagination strategy ব্যবহার হচ্ছে — যদি offset-based হয়, এটাই root cause কারণ নতুন পোস্ট যুক্ত হওয়ায় পজিশন শিফট হচ্ছে। ফিক্স হিসেবে cursor-based (keyset) pagination-এ মাইগ্রেট করব, যেখানে cursor একটা stable reference (যেমন last seen post-এর timestamp+id) ব্যবহার করবে যা নতুন ডেটা যুক্ত হলেও প্রভাবিত হবে না।

---

## 🛠 Hands-On Practice
- DRF-এ একই dataset-এ `PageNumberPagination` এবং `CursorPagination` দুটো implement করুন, একটা বড় offset (যেমন পেজ ৫০০) রিকোয়েস্ট করে response time তুলনা করুন
- একটা "live feed" সিমুলেট করুন যেখানে pagination করার মাঝখানে নতুন আইটেম যুক্ত হয়, এবং দেখুন offset-based pagination-এ কীভাবে duplicate/skip সমস্যা হয়

---

# 📋 মডিউল ৪ — সামারি শীট / চিট শীট

| টপিক | মূল কথা |
|---|---|
| N+1 Query Problem | Loop-এর ভেতরে DB কল এড়ান — `select_related`/`prefetch_related`/DataLoader ব্যবহার করুন |
| Caching (Redis) | High-read, low-write ডেটা cache করুন; invalidation strategy আগেই ডিজাইন করুন |
| Query Optimization | সঠিক জায়গায় index দিন, `EXPLAIN ANALYZE` দিয়ে যাচাই করুন, `SELECT *` এড়ান |
| Pagination | বড় ডেটাসেট/রিয়েল-টাইম ফিডে cursor-based ব্যবহার করুন; offset-based শুধু ছোট/স্ট্যাটিক ডেটাসেটে |

## ✅ Key Takeaways
- Performance সমস্যার ৮০% সমাধান হয় তিনটা মূল জায়গায়: query count কমানো (N+1), caching, এবং সঠিক indexing
- "লোকালে কাজ করছে" মানেই "প্রোডাকশনে স্কেলে কাজ করবে" না — সবসময় production-scale ডেটা দিয়ে চিন্তা করা উচিত
- Caching এবং pagination উভয়ই trade-off নিয়ে আসে (consistency vs speed, random access vs performance) — context বুঝে সিদ্ধান্ত নিতে হয়, একটা "one-size-fits-all" সমাধান নেই

## 🎯 Senior Engineer Checklist
- [ ] প্রতিটা list endpoint-এর query count constant (record সংখ্যার সাথে না বাড়া) কিনা পরীক্ষা করা হয়েছে
- [ ] Frequently-read, infrequently-written ডেটার জন্য caching layer আছে এবং invalidation logic ঠিকঠাক কিনা
- [ ] Frequently filtered/sorted column-গুলোতে সঠিক index আছে কিনা, এবং `EXPLAIN ANALYZE` দিয়ে যাচাই করা হয়েছে কিনা
- [ ] বড় বা রিয়েল-টাইম পরিবর্তনশীল ডেটাসেটে cursor-based pagination ব্যবহার হচ্ছে কিনা
- [ ] Production monitoring-এ slow query এবং cache hit-ratio ট্র্যাক করা হচ্ছে কিনা

---

➡️ **পরবর্তী মডিউল**: মডিউল ৫ — Security (JWT/Token Auth, RBAC, Rate Limiting, SQL Injection Prevention, CSRF/XSS Protection)
