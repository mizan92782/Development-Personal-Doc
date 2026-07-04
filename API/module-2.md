# 🧱 মডিউল ১: API FUNDAMENTALS
### FAANG-Level Backend Engineer হওয়ার যাত্রা — প্রথম মডিউল

---

## ভূমিকা

এই মডিউলে আমরা API-এর একদম গোড়া থেকে শুরু করব — কিন্তু "জুনিয়র লেভেলে" না, বরং একজন **Senior/Staff Engineer** যেভাবে চিন্তা করে সেই দৃষ্টিকোণ থেকে। আমরা কভার করব:

1. API কী এবং কেন এটা এত গুরুত্বপূর্ণ
2. REST vs RPC vs GraphQL
3. HTTP Methods
4. HTTP Status Codes
5. Stateless Systems

প্রতিটি টপিকের সাথে DRF (Django REST Framework) এবং Spring Boot-এর কোড, Production-level insight, এবং FAANG ইন্টারভিউ প্রশ্ন-উত্তর থাকবে।

---

# 📌 টপিক ১: API কী? (What is an API?)

## ১.১ কেন শিখব?
API ছাড়া আজকের কোনো সফটওয়্যার সিস্টেম চলতে পারে না। Amazon-এর প্রতিটি মাইক্রোসার্ভিস একে অপরের সাথে API-এর মাধ্যমে কথা বলে। আপনি যদি API-এর internal কাজ না বোঝেন, তাহলে আপনি কখনোই একটা scalable সিস্টেম ডিজাইন করতে পারবেন না — শুধু কোড লিখতে পারবেন।

## ১.২ এটা কোন সমস্যা সমাধান করে?
দুটো আলাদা সিস্টেম (যেমন Mobile App আর Backend Server) যাতে একে অপরের internal implementation না জেনেও একসাথে কাজ করতে পারে — সেই **contract** তৈরি করে API। এটা একটা **abstraction layer**।

## ১.৩ Production-এ Ignore করলে কী হয়?
- কোনো clear contract ছাড়া API বানালে frontend আর backend টিম একে অপরের কোড নিয়ে কনফিউজড হয়ে যায়
- Versioning ছাড়া API দিলে এক ক্লায়েন্ট আপডেট করলে আরেক ক্লায়েন্ট ভেঙে যায় (breaking change)
- 2012 সালে Netflix-এর internal API redesign না করার কারণে তাদের মাইক্রোসার্ভিস আর্কিটেকচারে যাওয়া কঠিন হয়ে গিয়েছিল

## ১.৪ Real-World Examples
- **Amazon**: প্রতিটি প্রোডাক্ট পেজ আসলে ৫০+ মাইক্রোসার্ভিস API call-এর সমষ্টি (price service, inventory service, recommendation service)
- **Netflix**: তাদের প্রতিটি ডিভাইস (TV, Mobile, Web) একই Backend API ব্যবহার করে কিন্তু আলাদা UI দেখায়
- **Uber**: Driver App আর Rider App দুটোই একই Trip Service API ব্যবহার করে কিন্তু ভিন্নভাবে

## ১.৫ API Concept ব্যাখ্যা
API (Application Programming Interface) হলো একটা **নিয়মের সেট** যা বলে দেয়:
- কোন URL-এ request পাঠাতে হবে
- কী ফরম্যাটে data পাঠাতে হবে (JSON সাধারণত)
- কী response পাওয়া যাবে
- কী Authentication লাগবে

```
Client (Mobile/Web)  --->  API (Contract)  --->  Server (Business Logic + DB)
                     <---  Response (JSON) <---
```

## ১.৬ Django REST Framework Implementation

```python
# views.py
from rest_framework.decorators import api_view
from rest_framework.response import Response

@api_view(['GET'])
def hello_api(request):
    return Response({"message": "Hello from API", "status": "success"})
```

```python
# urls.py
from django.urls import path
from .views import hello_api

urlpatterns = [
    path('api/hello/', hello_api),
]
```

## ১.৭ Spring Boot Equivalent

```java
@RestController
@RequestMapping("/api")
public class HelloController {

    @GetMapping("/hello")
    public ResponseEntity<Map<String, String>> helloApi() {
        Map<String, String> response = new HashMap<>();
        response.put("message", "Hello from API");
        response.put("status", "success");
        return ResponseEntity.ok(response);
    }
}
```

## ১.৮ Internal Request Lifecycle (গভীর ব্যাখ্যা)
1. ক্লায়েন্ট একটা HTTP request পাঠায় (DNS resolution → TCP connection → TLS handshake)
2. Load Balancer request রিসিভ করে সঠিক সার্ভারে route করে
3. Web Server (Nginx/Gunicorn) request রিসিভ করে WSGI/ASGI অ্যাপে পাস করে
4. Django/Spring middleware chain (auth, logging, CORS) execute হয়
5. URL Router সঠিক View/Controller খুঁজে বের করে
6. View business logic execute করে, DB query করে
7. Serializer/DTO data কে JSON-এ রূপান্তর করে
8. Response client-এর কাছে ফিরে যায়

## ১.৯ Architecture Diagram (Text-based)

```
[Client] -> [DNS] -> [Load Balancer] -> [Web Server]
              |
        [Middleware Chain: Auth -> Logging -> CORS]
              |
        [URL Router] -> [View/Controller]
              |
        [Business Logic] -> [Database/Cache]
              |
        [Serializer] -> [JSON Response]
              |
        [Client রিসিভ করল]
```

## ১.১০ Performance Impact
প্রতিটি API call-এ network latency (RTT), serialization overhead, এবং DB query time যুক্ত হয়। একটা পেজে যদি ১০টা আলাদা API call লাগে, এবং প্রতিটার latency ১০০ms হয়, total ১ সেকেন্ড লেগে যেতে পারে — যদি parallel না করা হয়।

## ১.১১ Common Mistakes (Junior vs Senior)
- **জুনিয়র**: এক endpoint-এ সব ডেটা পাঠিয়ে দেয় (over-fetching), বা প্রতিটা ছোট তথ্যের জন্য আলাদা API বানায় (under-fetching, N+1 calls)
- **সিনিয়র**: Resource-ভিত্তিক ডিজাইন করে এবং client-এর প্রয়োজন বুঝে response shape ঠিক করে (consider GraphQL বা field selection)

## ১.১২ Debugging Strategy
- প্রথমে নিশ্চিত করুন request আদৌ সার্ভারে পৌঁছেছে কিনা (access log চেক)
- তারপর middleware-level সমস্যা (auth/CORS) চেক করুন
- শেষে business logic এবং DB query চেক করুন (Postman/curl দিয়ে আলাদা করে টেস্ট করুন)

## ১.১৩ Optimization Techniques
- Response compression (gzip)
- Connection pooling
- Proper indexing on DB queries triggered by the API
- Batch endpoints যখন একসাথে অনেক resource লাগে

## ১.১৪ Trade-offs (কখন ব্যবহার করবেন না)
সব কিছুর জন্য API দরকার নেই — একই সার্ভিসের ভেতরে দুটো module যদি একই প্রসেসে চলে, সরাসরি function call ব্যবহার করাই ভালো; অযথা HTTP overhead যোগ করা ঠিক না।

## ১.১৫ Production Best Practices
- প্রতিটা API versioned হওয়া উচিত (`/api/v1/...`)
- Consistent error format থাকা উচিত
- Rate limiting এবং authentication বাধ্যতামূলক
- Documentation (Swagger/OpenAPI) আপ-টু-ডেট রাখা

## ১.১৬ Security Considerations
- কখনো sensitive data (password, token) plain response-এ পাঠাবেন না
- HTTPS বাধ্যতামূলক
- Input validation সার্ভার সাইডে করতে হবে, ক্লায়েন্টকে বিশ্বাস করা যাবে না

## ১.১৭ Senior Engineer Design Mindset
একজন সিনিয়র ইঞ্জিনিয়ার API ডিজাইন করার আগে চিন্তা করে: *"এই API কে ব্যবহার করবে? কত স্কেলে? কী data দরকার এবং কী দরকার নেই? এটা backward compatible থাকবে কীভাবে?"*

---

## 🧠 ইন্টারভিউ সেকশন: API কী?

### 🔹 Beginner প্রশ্ন
1. **API কী এবং এটা কেন প্রয়োজন?**
   *উত্তর*: API হলো দুটো সফটওয়্যার সিস্টেমের মধ্যে যোগাযোগের নিয়ম। এটা প্রয়োজন কারণ এটা client এবং server-এর implementation আলাদা রাখে — যাতে একটা পরিবর্তন হলে অন্যটা না ভাঙে। উদাহরণ: Amazon অ্যাপ ব্যাকএন্ড না জেনেও প্রোডাক্ট তথ্য দেখাতে পারে কারণ একটা ফিক্সড API contract আছে।

2. **API আর Library-র মধ্যে পার্থক্য কী?**
   *উত্তর*: Library একই প্রসেসে in-memory function call হিসেবে চলে, কিন্তু API সাধারণত network-এর মাধ্যমে আলাদা সার্ভিসে যায় (যদিও internal API-ও থাকতে পারে)। Library faster কিন্তু একই codebase-এর মধ্যে আবদ্ধ; API distributed সিস্টেমে কাজ করার সুযোগ দেয়।

3. **JSON কেন এত জনপ্রিয় API response format?**
   *উত্তর*: JSON human-readable, lightweight, এবং প্রায় সব প্রোগ্রামিং ল্যাঙ্গুয়েজে সহজে parse করা যায়। XML-এর তুলনায় ছোট এবং কম complex।

### 🔹 Intermediate প্রশ্ন
1. **একটা API request সার্ভারে পৌঁছানোর সম্পূর্ণ lifecycle ব্যাখ্যা করুন।**
   *উত্তর*: DNS resolution থেকে শুরু করে Load Balancer, Web server, Middleware chain, Router, View/Controller, Database, Serializer, এবং সবশেষে Response — এই প্রতিটা স্টেপ আগে বর্ণনা করা হয়েছে। ইন্টারভিউতে এই ধাপগুলো ক্রমানুসারে বলা গুরুত্বপূর্ণ।

2. **Over-fetching এবং Under-fetching কী, এবং কীভাবে সমাধান করবেন?**
   *উত্তর*: Over-fetching মানে প্রয়োজনের চেয়ে বেশি ডেটা আসা (যেমন পুরো user object আসলো অথচ শুধু নাম লাগতো); under-fetching মানে একটা request-এ পুরো প্রয়োজন না মেটা ফলে আরও request লাগে। সমাধান: field selection (sparse fieldsets), GraphQL, বা client-specific endpoint।

3. **আপনি কীভাবে একটা নতুন API ডিজাইন করার আগে decide করবেন এটা REST হবে নাকি RPC হবে?**
   *উত্তর*: যদি resource-ভিত্তিক CRUD অপারেশন হয় তাহলে REST; যদি একটা specific action/command (যেমন `calculateTax`) হয় তাহলে RPC-style বেশি স্বাভাবিক।

### 🔹 Advanced/Senior প্রশ্ন
1. **একটা API ১ মিলিয়ন req/sec হ্যান্ডেল করতে হবে — আপনার ডিজাইন approach কী হবে?**
   *উত্তর*: Horizontal scaling, caching layer (Redis/CDN), stateless design যাতে কোনো লোড ব্যালেন্সার সহজে distribute করতে পারে, database read replicas, এবং asynchronous processing যেখানে সম্ভব। এছাড়া rate limiting এবং circuit breaker দরকার যাতে একটা downstream সার্ভিস ফেল করলে পুরো সিস্টেম না ভেঙে যায়।

2. **API Contract পরিবর্তন করতে হবে কিন্তু পুরনো ক্লায়েন্টরা ভাঙা যাবে না — কীভাবে হ্যান্ডেল করবেন?**
   *উত্তর*: Versioning (URL/Header-based), additive changes (নতুন optional field যুক্ত করা, পুরনো field না সরানো), এবং deprecation policy সহ migration window দেওয়া।

3. **Internal API এবং Public API ডিজাইনে পার্থক্য কী হওয়া উচিত?**
   *উত্তর*: Internal API-তে performance এবং flexibility প্রাধান্য পায় (gRPC ব্যবহার করা যায়), কিন্তু Public API-তে stability, documentation, backward compatibility এবং strict versioning জরুরি কারণ external client-দের আপডেট করা আপনার নিয়ন্ত্রণে নেই।

### 🔹 FAANG-Level System Design প্রশ্ন
1. **Amazon-এর প্রোডাক্ট পেজ ডিজাইন করুন — এটা কতগুলো API call নিয়ে গঠিত হতে পারে এবং কেন?**
   *উত্তর (সংক্ষেপে)*: Product Info Service, Pricing Service, Inventory Service, Reviews Service, Recommendation Service — প্রতিটা আলাদা মাইক্রোসার্ভিস, parallel API call হিসেবে aggregate করা হয় একটা BFF (Backend-for-Frontend) লেয়ারে, যাতে ক্লায়েন্টকে একবারেই পুরো ডেটা দেওয়া যায়।

2. **একটা API Gateway কীভাবে শত শত মাইক্রোসার্ভিসের জন্য একটা একক entry point হিসেবে কাজ করে?**
   *উত্তর*: API Gateway routing, authentication, rate limiting, এবং request aggregation কেন্দ্রীভূত করে রাখে, যাতে প্রতিটা মাইক্রোসার্ভিসকে আলাদা করে এই কাজ implement করতে না হয়।

### 🔹 Scenario-Based প্রশ্ন
1. **প্রোডাকশনে একটা API হঠাৎ ৫০০ এরর দিচ্ছে, কিন্তু লোকালে ঠিকঠাক চলছে — আপনি কীভাবে ডিবাগ করবেন?**
   *উত্তর*: প্রথমে production logs/monitoring (যেমন CloudWatch, Datadog) চেক করব, error trace দেখব। তারপর চেক করব production-এ কোনো config/env variable মিসিং কিনা, DB connection বা downstream dependency ফেল করছে কিনা। শেষে replicate করার চেষ্টা করব একটা staging environment-এ।

---

## 🛠 Hands-On Practice
- DRF-এ একটা সাধারণ `health-check` API বানান যা সার্ভার আপ আছে কিনা জানায়
- Spring Boot-এ একই endpoint বানান এবং তুলনা করুন কোডের পার্থক্য
- একটা API-র response-এ ইচ্ছাকৃতভাবে অতিরিক্ত ডেটা যুক্ত করুন এবং বুঝুন over-fetching কেমন লাগে

---

# 📌 টপিক ২: REST vs RPC vs GraphQL

## ২.১ কেন শিখব?
ইন্টারভিউতে এটা সবচেয়ে কমন প্রশ্নগুলোর একটা: "REST কেন ব্যবহার করবেন, GraphQL কেন না?" — এই উত্তর ভালোভাবে দিতে পারলে আপনাকে আলাদা করে দেখাবে।

## ২.২ সমস্যা সমাধান
তিনটাই একই সমস্যা সমাধান করে — client-server communication — কিন্তু আলাদা trade-off-এ। REST সহজ এবং standardized; RPC দ্রুত এবং action-oriented; GraphQL flexible querying দেয়।

## ২.৩ Production Failure যদি ভুল চয়েস করেন
- একটা মোবাইল অ্যাপে যদি REST দিয়ে অনেক nested resource আনতে হয় (under-fetching সমস্যা), অ্যাপ স্লো হয়ে যাবে কারণ অনেকগুলো network round-trip লাগবে
- GraphQL ব্যবহার করলে কিন্তু caching strategy না বানালে, CDN-level caching হারিয়ে যায় (যেহেতু GraphQL সাধারণত POST request ব্যবহার করে)

## ২.৪ Real-World Examples
- **Facebook/Meta**: GraphQL নিজেই তৈরি করেছিল কারণ তাদের News Feed-এ অনেক ধরনের nested ডেটা দরকার হতো একসাথে
- **Google**: তাদের internal মাইক্রোসার্ভিস কমিউনিকেশনের জন্য gRPC (RPC-based) ব্যবহার করে কারণ এটা অনেক দ্রুত (binary protocol)
- **Netflix**: পাবলিক-ফেসিং API-গুলোর জন্য REST ব্যবহার করে কারণ এটা simple এবং debug করা সহজ

## ২.৫ Concept ব্যাখ্যা

| বিষয় | REST | RPC | GraphQL |
|---|---|---|---|
| স্টাইল | Resource-based | Action/Function-based | Query-based |
| URL প্যাটার্ন | `/users/1` | `/calculateTax` | একটাই endpoint `/graphql` |
| Over/Under-fetching | প্রায়ই হয় | হয় না (নির্দিষ্ট কাজ) | একদমই হয় না (client ঠিক করে) |
| Caching | সহজ (HTTP cache) | কঠিন | কঠিন (POST-based) |
| শেখার curve | সহজ | মাঝারি | কঠিন |
| Best for | পাবলিক API | Internal microservices | Complex/nested ডেটা চাহিদা |

## ২.৬ DRF Implementation (REST style)

```python
class UserViewSet(viewsets.ModelViewSet):
    queryset = User.objects.all()
    serializer_class = UserSerializer
```
এটা স্বয়ংক্রিয়ভাবে `/users/`, `/users/{id}/` ইত্যাদি resource-based endpoint তৈরি করে।

## ২.৭ Spring Boot Equivalent (REST)

```java
@RestController
@RequestMapping("/users")
public class UserController {
    @GetMapping("/{id}")
    public UserDTO getUser(@PathVariable Long id) { ... }

    @PostMapping
    public UserDTO createUser(@RequestBody UserDTO dto) { ... }
}
```

## ২.৮ Internal Request Lifecycle (পার্থক্য সহ)
- **REST**: প্রতিটা HTTP verb (GET/POST) আলাদা route-এ ম্যাপ হয়ে নির্দিষ্ট controller-এ যায়
- **RPC/gRPC**: একটা single function call serialize হয়ে (Protocol Buffer দিয়ে) সরাসরি সার্ভার-সাইড function execute করে — HTTP semantics নেই
- **GraphQL**: একটা single POST request একটা query parse করে, resolver chain execute করে, এবং exact requested fields রিটার্ন করে

## ২.৯ Architecture Diagram

```
REST:      Client -> GET /users/1        -> UserController.get()
RPC/gRPC:  Client -> call(getUser, id=1) -> Server function execute
GraphQL:   Client -> POST /graphql {query} -> Resolver chain -> exact fields
```

## ২.১০ Performance Impact
gRPC সাধারণত সবচেয়ে fast (binary protocol, HTTP/2 multiplexing)। REST সবচেয়ে বেশি overhead বহন করে (JSON parsing, multiple round trips)। GraphQL একটা request-এ সব আনে কিন্তু resolver-level complexity (N+1 problem) থাকতে পারে যদি ঠিকমতো batching না করা হয় (DataLoader pattern)।

## ২.১১ Common Mistakes
- **জুনিয়র**: GraphQL ব্যবহার করে কিন্তু resolver-এ N+1 query সমস্যা তৈরি করে
- **সিনিয়র**: DataLoader/batching ব্যবহার করে এবং query complexity limit করে (depth limiting) যাতে কেউ malicious deeply-nested query না পাঠাতে পারে

## ২.১২ Debugging Strategy
- REST: HTTP status code এবং endpoint আলাদাভাবে চেক করা সহজ
- GraphQL: একটাই endpoint, তাই error response-এর `errors` array এবং query structure গভীরভাবে দেখতে হয়
- gRPC: status code + trailer metadata চেক করতে হয়, tooling (grpcurl) লাগে

## ২.১৩ Optimization Techniques
- REST-এ: field filtering query param (`?fields=name,email`)
- GraphQL-এ: persisted queries, DataLoader batching
- gRPC-এ: connection reuse, streaming RPC যখন বড় ডেটাসেট লাগে

## ২.১৪ Trade-offs
- GraphQL শক্তিশালী কিন্তু caching ও security (query complexity attack) জটিল করে তোলে
- gRPC দ্রুত কিন্তু browser থেকে সরাসরি ব্যবহার করা কঠিন (gRPC-Web লাগে), এবং human-readable না
- REST সহজ কিন্তু flexible querying দেয় না

## ২.১৫ Production Best Practices
- পাবলিক-ফেসিং API-তে REST বা GraphQL — যেটা ক্লায়েন্টদের প্রয়োজন অনুযায়ী ভালো ফিট করে
- Internal মাইক্রোসার্ভিসে gRPC প্রাধান্য পায় উচ্চ throughput-এর জন্য

## ২.১৬ Security Considerations
- GraphQL-এ query depth/complexity limiting আবশ্যক, না হলে DoS attack সম্ভব
- gRPC-এ TLS encryption এবং service-to-service authentication (mTLS) গুরুত্বপূর্ণ

## ২.১৭ Senior Engineer Mindset
সিনিয়র ইঞ্জিনিয়ার প্রশ্ন করে: "ক্লায়েন্ট কে? Browser, mobile, নাকি অন্য মাইক্রোসার্ভিস? Network constraints কী? Team-এর GraphQL/gRPC operational maturity কতটুকু?" — শুধু "GraphQL hype" দেখে চয়েস করে না।

---

## 🧠 ইন্টারভিউ সেকশন: REST vs RPC vs GraphQL

### 🔹 Beginner প্রশ্ন
1. **REST API-র মূল নীতিগুলো কী কী?**
   *উত্তর*: Statelessness, resource-based URL, standard HTTP verbs ব্যবহার, এবং uniform interface — এই মূল নীতিগুলো REST-এর ভিত্তি।
2. **GraphQL-এ একটাই endpoint থাকে কেন?**
   *উত্তর*: কারণ client query-র মধ্যেই বলে দেয় কী ডেটা লাগবে, সার্ভার তা resolve করে — তাই আলাদা আলাদা resource URL-এর প্রয়োজন নেই।
3. **RPC-র full form এবং এর মূল ধারণা কী?**
   *উত্তর*: Remote Procedure Call — মনে হয় যেন লোকাল ফাংশন কল করছেন, কিন্তু আসলে এটা নেটওয়ার্কের মাধ্যমে রিমোট সার্ভারে execute হয়।

### 🔹 Intermediate প্রশ্ন
1. **GraphQL কীভাবে over-fetching সমস্যা সমাধান করে, উদাহরণ দিয়ে বলুন।**
   *উত্তর*: REST-এ `/users/1` কল করলে পুরো user object আসে, এমনকি যদি শুধু নাম লাগে। GraphQL-এ client লিখতে পারে `{ user(id: 1) { name } }` — শুধু নাম-ই রিটার্ন হবে।
2. **gRPC কেন HTTP/JSON-এর চেয়ে দ্রুত?**
   *উত্তর*: gRPC Protocol Buffers (বাইনারি ফরম্যাট) ব্যবহার করে যা JSON-এর চেয়ে ছোট এবং parse করা দ্রুত, এবং HTTP/2 ব্যবহার করে একই কানেকশনে multiplexed streams চালায়।
3. **আপনি কীভাবে decide করবেন কোনো প্রজেক্টে REST নাকি GraphQL ব্যবহার করবেন?**
   *উত্তর*: যদি client-side ডেটা প্রয়োজনীয়তা ঘন ঘন পরিবর্তন হয় এবং nested resource বেশি থাকে (যেমন social feed), GraphQL উপযোগী। সহজ CRUD হলে REST যথেষ্ট এবং দ্রুত implement করা যায়।

### 🔹 Advanced/Senior প্রশ্ন
1. **GraphQL-এ N+1 query সমস্যা কীভাবে হয় এবং সমাধান কী?**
   *উত্তর*: প্রতিটা resolver যদি আলাদা করে DB কল করে (যেমন প্রতিটা পোস্টের জন্য আলাদা author lookup), তাহলে ১০০টা পোস্টের জন্য ১০১টা কোয়েরি হয়ে যায়। DataLoader pattern ব্যবহার করে এই কল-গুলো ব্যাচ করে একটা single কোয়েরিতে রূপান্তর করা হয়।
2. **আপনি কীভাবে GraphQL API-কে malicious deeply-nested query থেকে রক্ষা করবেন?**
   *উত্তর*: Query depth limiting, complexity scoring (প্রতিটা field-এর জন্য একটা cost assign করা এবং total cost limit সেট করা), এবং timeout enforcement।
3. **gRPC streaming কী এবং এটা কোন ধরনের ইউজ-কেসে দরকার?**
   *উত্তর*: gRPC চারভাবে কাজ করতে পারে — unary, server streaming, client streaming, bidirectional streaming। লাইভ লোকেশন ট্র্যাকিং (Uber) বা রিয়েল-টাইম চ্যাট-এর মতো ইউজ-কেসে streaming RPC দরকার হয়।

### 🔹 FAANG-Level System Design প্রশ্ন
1. **Facebook News Feed ডিজাইন করুন — কেন তারা REST-এর বদলে GraphQL বেছে নিয়েছিল?**
   *উত্তর*: News Feed-এ প্রতিটা পোস্টের সাথে author, likes, comments, reactions, media — সব আলাদা nested object থাকে, এবং প্রতিটা ডিভাইস (mobile vs web) আলাদা subset দরকার করে। GraphQL দিয়ে প্রতিটা ক্লায়েন্ট নিজের প্রয়োজন অনুযায়ী query করতে পারে, একটাই backend থেকে, ফলে আলাদা endpoint বানাতে হয় না।
2. **একটা মাইক্রোসার্ভিস আর্কিটেকচারে Order Service আর Payment Service-এর মধ্যে কমিউনিকেশনে আপনি কোন প্রোটোকল বেছে নেবেন এবং কেন?**
   *উত্তর*: Internal, latency-sensitive, high-throughput কমিউনিকেশনের জন্য gRPC ভালো ফিট, কারণ এটা দ্রুত এবং strongly-typed contract (Protobuf schema) দেয় যা দুই টিমের মধ্যে contract মিসম্যাচ কমায়।

### 🔹 Scenario-Based প্রশ্ন
1. **আপনার GraphQL সার্ভারে একটা single query সার্ভারকে স্লো করে দিচ্ছে — সমাধান কী করবেন?**
   *উত্তর*: প্রথমে query complexity analyze করব, resolver-level কোথায় N+1 হচ্ছে দেখব, DataLoader/batching যুক্ত করব, এবং প্রয়োজনে query depth limit সেট করব production-এ যাতে ভবিষ্যতে এমন না হয়।

---

## 🛠 Hands-On Practice
- একই "ব্লগ পোস্ট + কমেন্ট" ফিচার REST আর GraphQL দুইভাবে ডিজাইন করুন এবং তুলনা করুন কতগুলো network call লাগে
- DRF-এ একটা nested serializer বানান (Post + Comments) এবং দেখুন over-fetching কতটা হয়

---

# 📌 টপিক ৩: HTTP Methods

## ৩.১ কেন শিখব?
HTTP methods ভুল ব্যবহার করলে API non-idempotent হয়ে যায়, যা production-এ duplicate orders বা data corruption-এর মতো সমস্যা তৈরি করতে পারে।

## ৩.২ সমস্যা সমাধান
HTTP method একটা request-এর **intent** প্রকাশ করে — এটা কি ডেটা পড়ছে, তৈরি করছে, পুরোপুরি replace করছে, না partially update করছে, না ডিলিট করছে।

## ৩.৩ Production Failure
GET request দিয়ে ডেটা পরিবর্তন করলে (যেমন একটা লিংকে ক্লিক করলেই একাউন্ট ডিলিট হয়ে যায়), search engine crawler বা browser prefetching ভুলবশত সেটা trigger করে দিতে পারে — এটা একটা বহু পুরনো এবং পরিচিত production bug প্যাটার্ন।

## ৩.৪ Real-World Examples
- **Amazon Cart API**: আইটেম যুক্ত করতে POST, quantity আপডেট করতে PATCH, পুরো cart replace করতে PUT, রিমুভ করতে DELETE
- **Stripe Payment API**: Idempotency key-সহ POST ব্যবহার করে যাতে network retry-তে duplicate চার্জ না হয়

## ৩.৫ Concept ব্যাখ্যা

| Method | উদ্দেশ্য | Idempotent? | Safe? |
|---|---|---|---|
| GET | ডেটা পড়া | হ্যাঁ | হ্যাঁ |
| POST | নতুন resource তৈরি | না | না |
| PUT | পুরো resource replace | হ্যাঁ | না |
| PATCH | আংশিক আপডেট | না (সাধারণত) | না |
| DELETE | resource মুছে ফেলা | হ্যাঁ | না |

*Idempotent মানে একই request একাধিকবার করলেও ফলাফল একই থাকে।*

## ৩.৬ DRF Implementation

```python
class ProductDetailView(APIView):
    def get(self, request, pk):
        product = Product.objects.get(pk=pk)
        return Response(ProductSerializer(product).data)

    def put(self, request, pk):
        product = Product.objects.get(pk=pk)
        serializer = ProductSerializer(product, data=request.data)
        serializer.is_valid(raise_exception=True)
        serializer.save()
        return Response(serializer.data)

    def patch(self, request, pk):
        product = Product.objects.get(pk=pk)
        serializer = ProductSerializer(product, data=request.data, partial=True)
        serializer.is_valid(raise_exception=True)
        serializer.save()
        return Response(serializer.data)

    def delete(self, request, pk):
        Product.objects.filter(pk=pk).delete()
        return Response(status=204)
```

## ৩.৭ Spring Boot Equivalent

```java
@RestController
@RequestMapping("/products")
public class ProductController {

    @GetMapping("/{id}")
    public ProductDTO get(@PathVariable Long id) { ... }

    @PutMapping("/{id}")
    public ProductDTO replace(@PathVariable Long id, @RequestBody ProductDTO dto) { ... }

    @PatchMapping("/{id}")
    public ProductDTO update(@PathVariable Long id, @RequestBody Map<String, Object> updates) { ... }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) { ... }
}
```

## ৩.৮ Internal Request Lifecycle
HTTP method request line-এর প্রথম শব্দে থাকে (`GET /products/1 HTTP/1.1`)। Web server এটা parse করে router-এ পাস করে, এবং router method-based dispatch করে সঠিক handler function-এ পাঠায় — DRF-এ এটা `APIView`-র `get`/`post`/`put` মেথড নেমিং কনভেনশনে হয়, Spring Boot-এ annotation-ভিত্তিক (`@GetMapping` ইত্যাদি)।

## ৩.৯ Architecture Diagram

```
Request Line: PATCH /products/42 HTTP/1.1
        |
   [Router] -- method=PATCH --> ProductDetailView.patch()
        |
   [Partial Update Logic] -> [DB UPDATE only changed fields]
        |
   [Response: 200 OK + updated resource]
```

## ৩.১০ Performance Impact
PUT সম্পূর্ণ resource পাঠাতে হয়, যা বড় object-এর জন্য বেশি bandwidth খরচ করে; PATCH ছোট payload পাঠায় ফলে নেটওয়ার্ক খরচ কম, কিন্তু server-side logic একটু জটিল হয় (merge করতে হয়)।

## ৩.১১ Common Mistakes
- **জুনিয়র**: সব কিছুর জন্য POST ব্যবহার করে (এমনকি data fetch করার জন্যও), idempotency ভঙ্গ করে
- **সিনিয়র**: প্রতিটা method-এর semantic meaning বজায় রাখে, এবং non-idempotent POST অপারেশনে Idempotency-Key header support করে (যেমন Stripe করে)

## ৩.১২ Debugging Strategy
405 Method Not Allowed এরর পেলে চেক করুন route-এ ঠিক method allow করা আছে কিনা; ভুল ডেটা আপডেট হলে চেক করুন PUT না PATCH ব্যবহার করা হচ্ছে — কারণ PUT ভুলবশত missing fields-কে null করে দিতে পারে।

## ৩.১৩ Optimization Techniques
DELETE-এ soft-delete (একটা `is_deleted` flag) ব্যবহার করুন যদি audit history লাগে, hard delete না করে।

## ৩.১৪ Trade-offs
PUT সহজ কিন্তু পুরো object পাঠাতে হয় (bandwidth খরচ); PATCH দক্ষ কিন্তু standard নাই (RFC 6902 JSON Patch বা simple key-value, টিম-ভিত্তিক ডিসিশন লাগে)।

## ৩.১৫ Production Best Practices
- নেটওয়ার্ক retry-তে duplicate POST রোধের জন্য Idempotency-Key header ব্যবহার করুন
- DELETE-এ সবসময় soft delete বিবেচনা করুন financial/critical data-র জন্য

## ৩.১৬ Security Considerations
GET request-এ কখনো sensitive ডেটা (পাসওয়ার্ড, টোকেন) query parameter-এ পাঠাবেন না — কারণ এটা browser history, server log-এ থেকে যায়।

## ৩.১৭ Senior Engineer Mindset
সিনিয়র ইঞ্জিনিয়ার প্রতিটা endpoint ডিজাইন করার আগে চিন্তা করে — "এই অপারেশন কি network retry হলে নিরাপদ থাকবে?" (idempotency নিয়ে চিন্তা)।

---

## 🧠 ইন্টারভিউ সেকশন: HTTP Methods

### 🔹 Beginner প্রশ্ন
1. **GET আর POST-এর পার্থক্য কী?**
   *উত্তর*: GET ডেটা read করার জন্য, কোনো side-effect থাকা উচিত না, এবং parameters URL-এ যায়। POST নতুন resource তৈরি বা side-effect আছে এমন অপারেশনের জন্য, ডেটা body-তে যায়।
2. **PUT আর PATCH-এর পার্থক্য কী?**
   *উত্তর*: PUT পুরো resource replace করে; PATCH শুধু উল্লেখিত fields আপডেট করে।
3. **Idempotent মানে কী?**
   *উত্তর*: একই request বারবার পাঠালেও ফলাফল একইরকম থাকা — যেমন DELETE একবার বা দশবার করলেও resource ডিলিট-ই থাকবে।

### 🔹 Intermediate প্রশ্ন
1. **POST কেন idempotent না, এবং এটা production-এ কী সমস্যা তৈরি করতে পারে?**
   *উত্তর*: POST প্রতিবার নতুন resource তৈরি করে। নেটওয়ার্ক টাইমআউট হয়ে ক্লায়েন্ট যদি retry করে, দুইটা order তৈরি হয়ে যেতে পারে — এটা duplicate payment/order-এর মতো serious bug তৈরি করে।
2. **PUT request-এ যদি কিছু field missing থাকে, কী হয়?**
   *উত্তর*: PUT সম্পূর্ণ replace করে, তাই missing field-গুলো null/default হয়ে যেতে পারে — এটাই PUT-এর semantic, যা অনেক জুনিয়র ভুল বোঝে এবং ডেটা হারিয়ে ফেলে।
3. **DELETE-এ response body থাকা কি উচিত?**
   *উত্তর*: সাধারণত `204 No Content` রিটার্ন করা হয়, body ছাড়া — কারণ resource মুছে গেছে তাই দেখানোর কিছু নেই।

### 🔹 Advanced/Senior প্রশ্ন
1. **আপনি কীভাবে একটা non-idempotent POST অপারেশনকে নিরাপদ করবেন network retry-র বিরুদ্ধে?**
   *উত্তর*: ক্লায়েন্ট একটা unique `Idempotency-Key` header পাঠাবে; সার্ভার এই key cache/DB-তে রেখে দেবে কিছু সময়ের জন্য, এবং একই key দিয়ে দ্বিতীয়বার রিকোয়েস্ট এলে আগের response-ই রিটার্ন করবে, নতুন resource তৈরি করবে না।
2. **PATCH-এর জন্য আপনি কোন স্ট্যান্ডার্ড ফলো করবেন বড় টিমে?**
   *উত্তর*: RFC 6902 (JSON Patch) বা RFC 7396 (JSON Merge Patch) — দুটোই স্ট্যান্ডার্ডাইজড অ্যাপ্রোচ, যা টিমের মধ্যে অস্পষ্টতা কমায়, যদিও অনেক টিম সরলতার জন্য নিজের custom key-value PATCH বডি ব্যবহার করে।
3. **HTTP method ভুল ব্যবহারের কারণে কোনো বাস্তব security সমস্যা হতে পারে?**
   *উত্তর*: যদি GET দিয়ে state পরিবর্তনকারী অপারেশন (যেমন একাউন্ট ডিলিট) করা হয়, CSRF attack-এ একটা সাধারণ `<img>` ট্যাগ দিয়েও victim-এর ব্রাউজার থেকে সেই request trigger করানো সম্ভব, কারণ GET request সহজেই cross-origin ভাবে trigger হয়ে যায়।

### 🔹 FAANG-Level System Design প্রশ্ন
1. **Stripe-এর মতো একটা Payment API ডিজাইন করুন যেখানে network retry-তে duplicate charge না হয়।**
   *উত্তর*: প্রতিটা payment creation request-এ client একটা unique idempotency key পাঠাবে। সার্ভার সেই key-কে database-এ unique constraint দিয়ে রাখবে, এবং একই key দিয়ে আসা পরের request-গুলোকে প্রসেস না করে আগের সফল response cache থেকে রিটার্ন করবে।

### 🔹 Scenario-Based প্রশ্ন
1. **আপনার টিমের একজন ডেভেলপার সব অপারেশনের জন্যই POST ব্যবহার করছে — আপনি কীভাবে এটা code review-এ address করবেন?**
   *উত্তর*: আমি ব্যাখ্যা করব কেন method semantics গুরুত্বপূর্ণ — caching, idempotency, এবং API consumer-দের জন্য predictability। তারপর নির্দিষ্ট উদাহরণ দেখাব কোথায় GET/PUT/DELETE বেশি যুক্তিযুক্ত, এবং ভবিষ্যতে এই ভুল এড়াতে একটা team-wide style guide প্রস্তাব করব।

---

## 🛠 Hands-On Practice
- DRF-এ একটা `Task` resource বানান যেখানে PUT এবং PATCH দুটোই implement করুন এবং পার্থক্য টেস্ট করুন
- একটা POST endpoint-এ Idempotency-Key support যুক্ত করুন (cache দিয়ে)

---

# 📌 টপিক ৪: HTTP Status Codes

## ৪.১ কেন শিখব?
ভুল status code ব্যবহার করলে ক্লায়েন্ট error handling ভুলভাবে করে — যেমন একটা validation error-কে `500` দিয়ে দিলে monitoring system মনে করবে এটা server crash, যা একদম ভুল alert তৈরি করবে।

## ৪.২ সমস্যা সমাধান
Status code একটা সংক্ষিপ্ত, standardized উপায়ে বলে দেয় request-এর ফলাফল কী হলো — সফল, ক্লায়েন্টের ভুল, নাকি সার্ভারের সমস্যা।

## ৪.৩ Production Failure
সব error-এর জন্য `200 OK` দিয়ে body-তে `{"success": false}` পাঠানো একটা পরিচিত anti-pattern — এটা monitoring/alerting, load balancer health check, এবং client-side error handling সব নষ্ট করে দেয়।

## ৪.৪ Real-World Examples
- **Google API**: প্রতিটা error response একটা consistent JSON structure-সহ সঠিক status code রিটার্ন করে (৪০০, ৪০৩, ৪২৯ ইত্যাদি)
- **GitHub API**: Rate limit ছাড়ালে `403`/`429` রিটার্ন করে সাথে `Retry-After` header

## ৪.৫ Concept ব্যাখ্যা

| Range | মানে | উদাহরণ |
|---|---|---|
| 2xx | সফল | 200 OK, 201 Created, 204 No Content |
| 3xx | Redirect | 301 Moved Permanently, 304 Not Modified |
| 4xx | ক্লায়েন্টের ভুল | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict, 422 Unprocessable Entity, 429 Too Many Requests |
| 5xx | সার্ভারের ভুল | 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable |

## ৪.৬ DRF Implementation

```python
from rest_framework.exceptions import ValidationError
from rest_framework import status

class OrderCreateView(APIView):
    def post(self, request):
        if not request.data.get('product_id'):
            raise ValidationError("product_id is required")  # 400 স্বয়ংক্রিয়ভাবে

        if Order.objects.filter(idempotency_key=request.data.get('key')).exists():
            return Response({"error": "duplicate order"}, status=status.HTTP_409_CONFLICT)

        order = Order.objects.create(...)
        return Response(OrderSerializer(order).data, status=status.HTTP_201_CREATED)
```

## ৪.৭ Spring Boot Equivalent

```java
@PostMapping
public ResponseEntity<?> createOrder(@RequestBody OrderRequest req) {
    if (req.getProductId() == null) {
        return ResponseEntity.badRequest().body("product_id is required"); // 400
    }
    if (orderRepository.existsByIdempotencyKey(req.getKey())) {
        return ResponseEntity.status(HttpStatus.CONFLICT).body("duplicate order"); // 409
    }
    Order order = orderService.create(req);
    return ResponseEntity.status(HttpStatus.CREATED).body(order); // 201
}
```

## ৪.৮ Internal Request Lifecycle
View/Controller business logic execute করার পর একটা status code সেট করে response object-এ; এই status code HTTP response-এর প্রথম লাইনে (status line) যুক্ত হয়ে client-এর কাছে যায়, যেখানে client-side HTTP library এই কোড দেখেই decide করে success/error handler কোনটা চালাবে।

## ৪.৯ Architecture Diagram

```
[View Logic] -> validation fail? -> 400/422
             -> auth fail?       -> 401/403
             -> not found?       -> 404
             -> conflict?        -> 409
             -> success?         -> 200/201/204
             -> unexpected crash?-> 500 (logged + alerted)
```

## ৪.১০ Performance Impact
সঠিক status code (যেমন `304 Not Modified`) ব্যবহার করলে ক্লায়েন্ট পুরো response body re-download না করে cached ভার্সন ব্যবহার করতে পারে — এটা bandwidth এবং latency উভয়ই বাঁচায়।

## ৪.১১ Common Mistakes
- **জুনিয়র**: সব error-এ `400` বা সব ভুলেই `500` দিয়ে দেয়, distinction করে না
- **সিনিয়র**: প্রতিটা error scenario-র জন্য নির্দিষ্ট, semantically সঠিক status code বেছে নেয় এবং সাথে structured error body দেয়

## ৪.১২ Debugging Strategy
৫xx এরর দেখলে সার্ভার লগ/stack trace দেখুন (এটা সার্ভার-সাইড বাগ); ৪xx দেখলে client request validate করুন (এটা client-সাইড ইস্যু, সার্ভার ঠিক আছে)।

## ৪.১৩ Optimization Techniques
`304 Not Modified` + ETag/Last-Modified header ব্যবহার করে unnecessary data transfer কমান।

## ৪.১৪ Trade-offs
HTTP status code সীমিত granularity দেয় — তাই অনেক টিম status code-এর পাশাপাশি একটা application-level error code (যেমন `"error_code": "INSUFFICIENT_STOCK"`) যুক্ত করে আরও স্পেসিফিক তথ্য দেয়।

## ৪.১৫ Production Best Practices
- Consistent error response format রাখুন সব endpoint-এ (যেমন `{"error": {"code": ..., "message": ...}}`)
- `5xx` কোড পেলে অবশ্যই alerting/monitoring trigger হওয়া উচিত

## ৪.১৬ Security Considerations
`401` (Unauthorized — কে আপনি, প্রমাণ করুন) আর `403` (Forbidden — আমি জানি আপনি কে, কিন্তু অনুমতি নেই) এর পার্থক্য বজায় রাখুন; কিন্তু কোনো resource-এর existence leak করার ক্ষেত্রে কখনো কখনো ইচ্ছাকৃতভাবে `404` রিটার্ন করা হয় `403`-এর বদলে যাতে attacker resource আছে কিনা বুঝতে না পারে।

## ৪.১৭ Senior Engineer Mindset
সিনিয়র ইঞ্জিনিয়ার status code-কে শুধু "ঠিক বা ভুল" না দেখে একটা **communication tool** হিসেবে দেখে — যা monitoring, client error-handling, এবং debugging সবকিছুর ভিত্তি তৈরি করে।

---

## 🧠 ইন্টারভিউ সেকশন: HTTP Status Codes

### 🔹 Beginner প্রশ্ন
1. **401 আর 403-এর পার্থক্য কী?**
   *উত্তর*: 401 মানে ইউজার authenticate হয়নি (login লাগবে); 403 মানে ইউজার authenticated কিন্তু এই resource-এ অনুমতি নেই।
2. **201 Created কখন ব্যবহার করা হয়?**
   *উত্তর*: যখন POST request দিয়ে সফলভাবে একটা নতুন resource তৈরি হয়।
3. **404 আর 400-এর পার্থক্য কী?**
   *উত্তর*: 400 মানে request malformed/invalid; 404 মানে requested resource খুঁজে পাওয়া যায়নি।

### 🔹 Intermediate প্রশ্ন
1. **422 Unprocessable Entity আর 400 Bad Request-এর পার্থক্য কী?**
   *উত্তর*: 400 সাধারণত syntactically invalid request (যেমন malformed JSON) বোঝায়; 422 মানে request syntactically ঠিক আছে কিন্তু semantic/validation rule ভঙ্গ করেছে (যেমন email ফরম্যাট ভুল)। অনেক টিম তবে ব্যবহারিকভাবে দুটোকেই একসাথে গণ্য করে।
2. **429 Too Many Requests কখন রিটার্ন করা উচিত, এবং সাথে আর কী হেডার পাঠানো ভালো?**
   *উত্তর*: Rate limit ছাড়ালে; সাথে `Retry-After` header পাঠানো উচিত যাতে client জানে কতক্ষণ পর আবার চেষ্টা করতে হবে।
3. **502 আর 503-এর পার্থক্য কী?**
   *উত্তর*: 502 Bad Gateway মানে upstream সার্ভার থেকে invalid response এসেছে (যেমন proxy/load balancer ব্যাকএন্ড থেকে ভুল উত্তর পেয়েছে); 503 Service Unavailable মানে সার্ভার সাময়িকভাবে overloaded/maintenance-এ আছে।

### 🔹 Advanced/Senior প্রশ্ন
1. **কেন কিছু সিস্টেম ইচ্ছাকৃতভাবে 403-এর বদলে 404 রিটার্ন করে?**
   *উত্তর*: Security-র জন্য — যদি একটা private resource-এ unauthorized access হয়, 403 রিটার্ন করলে attacker বুঝে যায় resource আছে কিন্তু অনুমতি নেই (information leak)। 404 রিটার্ন করলে attacker জানতে পারে না resource আদৌ আছে কিনা।
2. **আপনি কীভাবে status code আর application-level error code একসাথে ডিজাইন করবেন একটা বড় পাবলিক API-তে?**
   *উত্তর*: HTTP status code দিয়ে broad category বোঝাব (4xx/5xx), এবং body-তে একটা structured `error_code` (যেমন `"INSUFFICIENT_STOCK"`) ও human-readable message দেব, যাতে client প্রোগ্রামেটিক্যালি নির্দিষ্ট কেস হ্যান্ডেল করতে পারে এবং user-কে বোধগম্য মেসেজ দেখাতে পারে।
3. **Load Balancer health check-এ কোন status code গুরুত্বপূর্ণ ভূমিকা রাখে এবং কেন?**
   *উত্তর*: `200 OK` মানে instance healthy, কোনো `5xx` এলে Load Balancer সেই instance-কে rotation থেকে সরিয়ে দেয় — তাই health check endpoint-এর status code ভুল হলে পুরো traffic-routing লজিক ভেঙে যায়।

### 🔹 FAANG-Level System Design প্রশ্ন
1. **একটা বিশাল-স্কেল API-তে error monitoring এবং alerting সিস্টেম কীভাবে status code-এর উপর নির্ভর করে ডিজাইন করবেন?**
   *উত্তর*: প্রতিটা request-এর status code metrics-এ (যেমন Prometheus) ট্যাগ করে রাখব — `5xx` rate একটা থ্রেশহোল্ড ছাড়ালে automatically alert trigger হবে (PagerDuty)। `4xx` rate spike হলে এটা সাধারণত client bug বা attack নির্দেশ করতে পারে, তাই আলাদা dashboard-এ ট্র্যাক করব।

### 🔹 Scenario-Based প্রশ্ন
1. **আপনার API দেখছে `5xx` error rate হঠাৎ বেড়ে গেছে — কী স্টেপ নেবেন?**
   *উত্তর*: প্রথমে dashboard/logs দিয়ে দেখব কোন endpoint-এ স্পাইক হয়েছে এবং কোন সময় থেকে — recent deployment-এর সাথে correlate করব। তারপর dependency (DB, downstream service) healthy কিনা চেক করব। প্রয়োজনে সাম্প্রতিক deploy rollback করব এবং root cause analysis পরে করব।

---

## 🛠 Hands-On Practice
- একটা DRF API-তে ইচ্ছাকৃতভাবে ভুল status code (সব ক্ষেত্রে 200) ব্যবহার করুন, তারপর সঠিক করুন এবং পার্থক্য বুঝুন client error handling-এ
- একটা `429` rate-limit response বানান `Retry-After` header সহ

---

# 📌 টপিক ৫: Stateless Systems

## ৫.১ কেন শিখব?
Statelessness হলো REST API আর scalable distributed systems-এর সবচেয়ে গুরুত্বপূর্ণ ভিত্তি। এটা না বুঝলে আপনি কখনো বুঝবেন না কেন Load Balancing, Horizontal Scaling এত সহজ হয়ে যায় ঠিকমতো ডিজাইন করলে।

## ৫.২ সমস্যা সমাধান
যদি প্রতিটা request-এর জন্য সার্ভারকে আগের request-এর "memory" রাখতে হয় (state), তাহলে ক্লায়েন্টকে একই সার্ভারে বারবার যেতে হবে (sticky session) — এটা horizontal scaling-কে কঠিন করে দেয়। Stateless ডিজাইনে যেকোনো request যেকোনো সার্ভার হ্যান্ডেল করতে পারে।

## ৫.৩ Production Failure যদি Stateful ডিজাইন করেন
পুরনো session-based ওয়েব অ্যাপগুলোতে in-memory session রাখা হতো একটা নির্দিষ্ট সার্ভারে। সেই সার্ভার ক্র্যাশ করলে বা restart হলে, সব ইউজার logged out হয়ে যেত — এবং Load Balancer-এ sticky session সেটআপ করতে হতো যা scaling এবং failover জটিল করে তোলে।

## ৫.৪ Real-World Examples
- **JWT (JSON Web Token)**: Google, Amazon, Netflix — সবাই stateless authentication-এর জন্য JWT ব্যবহার করে, যেখানে token-এর মধ্যেই সব তথ্য এনকোডেড থাকে, সার্ভারে কোনো session store লাগে না
- **Netflix**: তাদের মাইক্রোসার্ভিসগুলো stateless, ফলে কোনো instance ক্র্যাশ করলে Load Balancer সহজেই অন্য instance-এ route করে দেয় — কোনো ডেটা হারায় না

## ৫.৫ Concept ব্যাখ্যা
Stateless মানে সার্ভার কোনো request-এর মধ্যে ক্লায়েন্ট সম্পর্কে কোনো তথ্য মনে রাখে না। প্রতিটা request-এ client-কে তার প্রয়োজনীয় সব context (যেমন auth token) পাঠাতে হয়।

```
Stateful (পুরনো পদ্ধতি):
Client -> Login -> Server creates Session in Memory -> SessionID Cookie
Client -> Next Request + SessionID -> Server খুঁজে দেখে Memory-তে Session আছে কিনা

Stateless (আধুনিক পদ্ধতি):
Client -> Login -> Server returns JWT Token (self-contained)
Client -> Next Request + JWT Token -> Server token verify করে (signature check), কোনো DB/Memory lookup লাগে না
```

## ৫.৬ DRF Implementation (Stateless JWT)

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    )
}

# views.py - কোনো session লাগছে না, token-ই যথেষ্ট
class ProfileView(APIView):
    permission_classes = [IsAuthenticated]
    def get(self, request):
        return Response({"username": request.user.username})
```

## ৫.৭ Spring Boot Equivalent

```java
// JWT filter token verify করে, কোনো server-side session লাগে না
@Component
public class JwtFilter extends OncePerRequestFilter {
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain) {
        String token = extractToken(req);
        if (jwtUtil.validateToken(token)) {
            Authentication auth = jwtUtil.getAuthentication(token);
            SecurityContextHolder.getContext().setAuthentication(auth);
        }
        chain.doFilter(req, res);
    }
}
```

## ৫.৮ Internal Request Lifecycle
প্রতিটা request-এ আসা JWT token-এর signature সার্ভার তার secret/public key দিয়ে verify করে — এটা একটা pure computation, কোনো ডেটাবেস বা মেমোরি lookup করার দরকার হয় না, ফলে এটা যেকোনো সার্ভার instance-এ independently verify করা যায়।

## ৫.৯ Architecture Diagram

```
[Client] --(request + JWT)--> [Load Balancer]
                                     |
                    -----------------------------------
                    |                |                |
              [Server A]       [Server B]       [Server C]
                    |                |                |
            (যেকোনো সার্ভার একইভাবে token verify করতে পারে — কোনো shared session store লাগে না)
```

## ৫.১০ Performance Impact
Stateless সিস্টেম horizontal scaling-এ সরাসরি সাহায্য করে — নতুন সার্ভার instance যুক্ত করলেই কোনো session migration/replication ছাড়াই কাজ করে। কিন্তু JWT token সাধারণত session ID-এর চেয়ে বড় (encoded data বহন করে), ফলে প্রতিটা request-এ সামান্য বেশি bandwidth লাগে।

## ৫.১১ Common Mistakes
- **জুনিয়র**: JWT-এর মধ্যে sensitive ডেটা (পাসওয়ার্ড, খুব বেশি personal info) এনকোড করে রাখে — ভুলে যায় JWT payload শুধু base64-encoded, encrypted না (যে কেউ decode করে পড়তে পারে)
- **সিনিয়র**: JWT-তে শুধু minimal claims (user id, role, expiry) রাখে এবং sensitive ডেটা আলাদাভাবে secure চ্যানেলে আনে

## ৫.১২ Debugging Strategy
ইউজার বারবার logout হয়ে যাচ্ছে এমন বাগ পেলে চেক করুন token expiry সঠিকভাবে সেট আছে কিনা, এবং multiple server instance-এর clock sync ঠিক আছে কিনা (clock skew JWT validation ফেল করাতে পারে)।

## ৫.১৩ Optimization Techniques
Token refresh strategy ব্যবহার করুন (short-lived access token + long-lived refresh token) যাতে security আর UX-এর মধ্যে ব্যালেন্স থাকে।

## ৫.১৪ Trade-offs
Stateless JWT token revoke করা কঠিন (যতক্ষণ না expire হয়, এটা valid থাকে) — তাই logout/ban করার immediate প্রয়োজন থাকলে একটা blacklist (Redis-এ) মেইনটেইন করতে হয়, যা আবার একটু statefulness ফিরিয়ে আনে।

## ৫.১৫ Production Best Practices
- Token-এ short expiry রাখুন এবং refresh token দিয়ে renew করুন
- Sensitive কোনো ডেটা JWT payload-এ রাখবেন না কারণ এটা encrypted না, শুধু signed

## ৫.১৬ Security Considerations
- JWT signature verification বাধ্যতামূলক, না হলে কেউ token tamper করে privilege escalate করতে পারে
- Algorithm `none` বা client-নির্ধারিত algorithm কখনো trust করবেন না (এটা একটা পরিচিত JWT vulnerability)

## ৫.১৭ Senior Engineer Mindset
সিনিয়র ইঞ্জিনিয়ার জানে statelessness একটা trade-off, perfect সমাধান না — scalability-র জন্য কিছু complexity (যেমন token revocation) accept করতে হয়, এবং সেই complexity ম্যানেজ করার জন্য সঠিক টুল (Redis blacklist, short expiry) বেছে নেয়।

---

## 🧠 ইন্টারভিউ সেকশন: Stateless Systems

### 🔹 Beginner প্রশ্ন
1. **Stateless মানে কী?**
   *উত্তর*: সার্ভার কোনো একটা request থেকে আগের request-এর তথ্য মনে রাখে না — প্রতিটা request independent এবং সম্পূর্ণ context সহ আসে।
2. **Session-based authentication আর Token-based (JWT) authentication-এর পার্থক্য কী?**
   *উত্তর*: Session-based-এ সার্ভার মেমোরি/DB-তে session ডেটা রাখে এবং ক্লায়েন্টকে শুধু একটা ID দেয়; Token-based-এ পুরো তথ্য (signed) token-এর মধ্যেই থাকে, সার্ভারে কিছু রাখতে হয় না।
3. **HTTP নিজেই stateless প্রোটোকল কেন বলা হয়?**
   *উত্তর*: কারণ প্রতিটা HTTP request আলাদা এবং স্বয়ংসম্পূর্ণ — প্রোটোকল নিজেই আগের request-এর কোনো তথ্য মনে রাখে না, এটা application layer-এ (cookies/session) যুক্ত করতে হয়।

### 🔹 Intermediate প্রশ্ন
1. **Stateless ডিজাইন কীভাবে horizontal scaling সহজ করে?**
   *উত্তর*: যখন কোনো সার্ভার instance-কে session data নিয়ে চিন্তা করতে হয় না, তখন Load Balancer যেকোনো request যেকোনো instance-এ পাঠাতে পারে — নতুন instance যুক্ত করা বা পুরনো instance সরানো কোনো ডেটা migration ছাড়াই সম্ভব হয়।
2. **JWT কি encrypted, নাকি শুধু signed?**
   *উত্তর*: ডিফল্টভাবে JWT শুধু signed (HMAC/RSA দিয়ে) — তাই কেউ এর content পড়তে পারে (base64 decode করে), কিন্তু modify করতে পারবে না signature ভেঙে যাবে। যদি গোপনীয়তা দরকার, JWE (encrypted JWT) ব্যবহার করতে হয়।
3. **Stateless সিস্টেমে logout হ্যান্ডেল করা কঠিন কেন?**
   *উত্তর*: কারণ token নিজেই valid থাকে যতক্ষণ না expire হয় — সার্ভারে কোনো "session" নেই যা delete করা যায়। সমাধান হলো blacklist বা short-lived token + refresh strategy।

### 🔹 Advanced/Senior প্রশ্ন
1. **আপনি একটা মাইক্রোসার্ভিস আর্কিটেকচারে stateless authentication কীভাবে ডিজাইন করবেন যাতে প্রতিটা সার্ভিস independently token verify করতে পারে?**
   *উত্তর*: একটা centralized Auth Service token issue করবে asymmetric key (RSA/ECDSA) দিয়ে sign করে। বাকি সব মাইক্রোসার্ভিস শুধু public key দিয়ে signature verify করবে — কোনো network call Auth Service-এ করার প্রয়োজন হবে না প্রতিটা request-এ, যা latency কমায় এবং Auth Service-কে single point of failure হওয়া থেকে রক্ষা করে।
2. **Stateless এবং Stateful সিস্টেমের মধ্যে কখন একটা hybrid approach ভালো?**
   *উত্তর*: যখন immediate token revocation দরকার (যেমন security incident-এ user ban করা) — তখন একটা lightweight stateful component (Redis blacklist with TTL) যুক্ত করা হয়, যেটা মূল stateless ডিজাইনের performance benefit প্রায় বজায় রেখেও critical security feature দেয়।
3. **Clock skew কীভাবে distributed সিস্টেমে JWT validation-এ সমস্যা তৈরি করতে পারে?**
   *উত্তর*: যদি একটা সার্ভারের সিস্টেম ক্লক অন্য সার্ভারের চেয়ে আগে/পরে থাকে, token expiry (`exp`) বা issued-at (`iat`) ভুলভাবে validate হতে পারে — কোনো ভ্যালিড token-কে invalid মনে হতে পারে বা উল্টোটা। সমাধান: NTP দিয়ে সব সার্ভারের ক্লক sync রাখা এবং validation-এ একটা সামান্য clock skew tolerance (leeway) যুক্ত করা।

### 🔹 FAANG-Level System Design প্রশ্ন
1. **আপনি একটা গ্লোবাল-স্কেল authentication সিস্টেম ডিজাইন করছেন (Google-এর মতো) যা মিলিয়ন মিলিয়ন রিকোয়েস্ট হ্যান্ডেল করবে — Stateless নাকি Stateful যাবেন এবং কেন?**
   *উত্তর*: মূলত Stateless (JWT/OAuth token) যাব কারণ এটা horizontal scaling-কে trivial করে দেয় এবং একটা single Auth DB-এর উপর bottleneck তৈরি হয় না। কিন্তু critical operation-গুলোর জন্য (token revocation, security ban) একটা distributed cache (Redis cluster) ব্যবহার করব যা globally replicate করা থাকবে যাতে revocation দ্রুত propagate হয়।

### 🔹 Scenario-Based প্রশ্ন
1. **একজন ইউজার রিপোর্ট করছে যে তাকে ব্যান করার পরও সে এখনও সিস্টেমে একশন নিতে পারছে — কেন এমন হতে পারে এবং কীভাবে ঠিক করবেন?**
   *উত্তর*: এটা সম্ভবত হচ্ছে কারণ ইউজারের কাছে এখনও একটা valid, unexpired JWT token আছে এবং সিস্টেমে কোনো revocation mechanism নেই। সমাধান: একটা token blacklist (user id + ban timestamp) Redis-এ যুক্ত করব যা প্রতিটা request-এ চেক হবে, এবং ভবিষ্যতে token expiry সময় কমিয়ে এই উইন্ডো ছোট করব।

---

## 🛠 Hands-On Practice
- DRF-এ JWT authentication সেটআপ করুন (`djangorestframework-simplejwt`) এবং দেখুন কীভাবে কোনো session store ছাড়াই কাজ করে
- একটা Redis-ভিত্তিক token blacklist বানান যাতে logout/ban হলে token সাথে সাথে invalid হয়ে যায়

---

# 📋 মডিউল ১ — সামারি শীট / চিট শীট

| টপিক | মূল কথা |
|---|---|
| API কী | Client আর Server-এর মধ্যে একটা contract/abstraction layer |
| REST vs RPC vs GraphQL | REST=resource-based, RPC=action-based (gRPC দ্রুত), GraphQL=flexible query |
| HTTP Methods | GET/POST/PUT/PATCH/DELETE — প্রতিটার নির্দিষ্ট semantic ও idempotency আছে |
| Status Codes | 2xx সফল, 4xx client ভুল, 5xx server ভুল — ভুল কোড monitoring ভাঙে |
| Stateless | প্রতিটা request independent — scaling সহজ করে, কিন্তু revocation জটিল করে |

## ✅ Key Takeaways
- API ডিজাইন শুধু "কাজ করা" না, বরং **scalability, security, এবং backward compatibility** নিয়ে চিন্তা করা
- HTTP method/status code-এর সঠিক ব্যবহার production monitoring এবং client-side error handling-এর ভিত্তি
- Statelessness আধুনিক distributed system scaling-এর মূল চাবিকাঠি, কিন্তু এর trade-off (revocation) বুঝে সমাধান বের করতে হয়

## 🎯 Senior Engineer Checklist
- [ ] প্রতিটা endpoint resource-based এবং সঠিক HTTP method ব্যবহার করছে কিনা
- [ ] সব error scenario-র জন্য সঠিক, semantically meaningful status code আছে কিনা
- [ ] Non-idempotent অপারেশন (POST) network retry-র বিরুদ্ধে সুরক্ষিত কিনা (Idempotency-Key)
- [ ] Authentication stateless (JWT/OAuth) কিনা, এবং revocation strategy আছে কিনা
- [ ] API versioned এবং backward-compatible কিনা

---

➡️ **পরবর্তী মডিউল**: মডিউল ২ — API Design Principles (RESTful design, Resource modeling, URI design, Pagination/Filtering/Sorting, Idempotency গভীরভাবে)
