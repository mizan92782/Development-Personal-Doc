# 🧱 MODULE 1: API FUNDAMENTALS (FAANG Level)

---

## 1.1 API কী এবং কেন দরকার

### 1. Topic Name
API (Application Programming Interface) Fundamentals

### 2. কেন শিখবে?
API হলো দুইটা software system-এর মধ্যে communication-এর "চুক্তি" (contract)। এটা না বুঝে কোনো backend engineer সঠিকভাবে system design করতে পারবে না — কারণ প্রায় প্রতিটা modern application (mobile app, web app, third-party integration) API-এর মাধ্যমেই ডেটা আদান-প্রদান করে।

### 3. Problem এটা কী সমাধান করে
API ছাড়া প্রতিটা client (mobile, web, partner company)-কে সরাসরি database access দিতে হতো — যা security risk, tight coupling, এবং maintainability নষ্ট করে। API একটা abstraction layer তৈরি করে যাতে internal implementation পরিবর্তন হলেও client অপ্রভাবিত থাকে।

### 4. Production Failure যদি Ignore করো
Client-কে সরাসরি database access দিলে (API layer ছাড়া) schema change করলেই সব client ভেঙে যাবে, এবং কোনো authentication/authorization layer না থাকায় data breach-এর ঝুঁকি বহুগুণ বেড়ে যায়।

### 5. Real-world উদাহরণ
- **Amazon**: প্রতিটা internal team API দিয়ে communicate করে (Jeff Bezos-এর বিখ্যাত "API mandate" ২০০২ সালে, যেখানে বলা হয় সব team-কে service interface API-এর মাধ্যমে expose করতে হবে)
- **Stripe**: পুরো business model-ই API-first — developer-রা Stripe-এর API দিয়ে payment integrate করে
- **Twitter/X**: third-party app তাদের API দিয়ে tweet post/read করে

### 6. API Concept Explanation
API-এর তিনটা প্রধান স্টাইল:
- **REST**: resource-based, HTTP method ব্যবহার করে (সবচেয়ে জনপ্রিয়)
- **RPC (Remote Procedure Call)**: function call-এর মতো (gRPC আধুনিক উদাহরণ, high-performance microservice communication-এ ব্যবহৃত)
- **GraphQL**: client নিজেই বলে দেয় কোন ডেটা দরকার, single endpoint দিয়ে flexible query

### 7. Django REST Framework Implementation
```python
# একটা basic API endpoint - REST style
from rest_framework.views import APIView
from rest_framework.response import Response

class ProductListView(APIView):
    def get(self, request):
        products = Product.objects.all()
        return Response({"products": [p.name for p in products]})
```

### 8. Spring Boot Equivalent
```java
@RestController
@RequestMapping("/api/products")
public class ProductController {
    @GetMapping
    public List<Product> getProducts() {
        return productService.findAll();
    }
}
```

### 9. Internal Request Lifecycle
Client request পাঠায় → Network-এর মধ্য দিয়ে server-এ পৌঁছায় → Web server (Nginx) → Application server (Gunicorn/Tomcat) → Framework routing (URL resolve) → View/Controller execute → Database query → Response serialize → Client-এ ফেরত।

### 10. Architecture Diagram (Text)
```
Client (Mobile/Web) --HTTP Request--> Server --> Business Logic --> Database
                    <--HTTP Response--
```

### 11. Performance Impact
API design-এর initial decision (REST vs GraphQL vs RPC) পরবর্তীতে পুরো system-এর latency ও scalability প্রভাবিত করে — যেমন REST-এ over-fetching/under-fetching সমস্যা হতে পারে যা GraphQL সমাধান করে, কিন্তু GraphQL নিজে caching-কে জটিল করে তোলে।

### 12. Common Mistakes (Junior vs Senior)
- Junior: API design না ভেবে সরাসরি code লেখা শুরু করে
- Senior: প্রথমে resource/contract design করে, তারপর implementation করে
- Junior: HTTP method-এর সঠিক semantic না মেনে সবকিছু POST দিয়ে করে
- Senior: GET/POST/PUT/DELETE-এর সঠিক ব্যবহার মেনে চলে (idempotency অনুযায়ী)

### 13. Debugging Strategy
API issue debug করার সময় প্রথমে request/response cycle-এর কোন ধাপে সমস্যা (network, server, application logic, database) সেটা isolate করা।

### 14. Optimization Techniques
Response payload ছোট রাখা (শুধু দরকারি field পাঠানো), compression (gzip) ব্যবহার করা, connection keep-alive রাখা।

### 15. Trade-offs (কখন ব্যবহার করবে না)
Internal, high-performance microservice communication-এ REST-এর JSON overhead এড়াতে gRPC/Protocol Buffer ব্যবহার করা ভালো — REST সব জায়গায় সেরা সমাধান না।

### 16. Production Best Practices
API-কে সবসময় versioned, documented (OpenAPI/Swagger), এবং backward-compatible রাখা।

### 17. Security Considerations
প্রতিটা API endpoint-এ authentication, input validation, এবং rate limiting থাকা উচিত — "internal API" বলে security relax করা একটা common mistake যা পরবর্তীতে exploit হতে পারে।

### 18. Senior Engineer Design Mindset
সিনিয়র ইঞ্জিনিয়ার API design করার আগে ভাবে: "এই API-এর consumer কে হবে, এবং তাদের actual need কী?" — শুধু database table-কে সরাসরি JSON-এ convert করে দেয়া API design না, এটা একটা anti-pattern।

---

## 1.2 REST vs RPC vs GraphQL

### API Concept Comparison

| বৈশিষ্ট্য | REST | RPC (gRPC) | GraphQL |
|---|---|---|---|
| Data Fetching | Fixed structure per endpoint | Function-call style | Client-defined query |
| Over/Under-fetching | সমস্যা হতে পারে | N/A (RPC নির্দিষ্ট) | সমাধান করে |
| Caching | সহজ (HTTP caching) | কঠিন | জটিল (query variable) |
| Performance | মাঝারি (JSON overhead) | দ্রুত (binary protocol) | নির্ভর করে query-এর উপর |
| Use Case | Public API, CRUD apps | Internal microservice communication | Complex client (mobile app with varying data needs) |

### Real-world উদাহরণ
- **REST**: GitHub API, Twitter API — public, widely-consumed API-এর জন্য industry standard
- **gRPC**: Netflix, Google internal microservices — high-performance, low-latency service-to-service call-এর জন্য
- **GraphQL**: Facebook (এটাই তৈরি করেছে), GitHub-এর নতুন API v4 — complex, nested data-এর জন্য যেখানে client ভিন্ন ভিন্ন ডেটা চায় (মোবাইল vs ওয়েব)

### Django/Spring Implementation Snapshot
```python
# REST (DRF)
class OrderView(APIView):
    def get(self, request, id):
        return Response(OrderSerializer(order).data)

# GraphQL (Graphene-Django)
class OrderType(DjangoObjectType):
    class Meta:
        model = Order

class Query(graphene.ObjectType):
    order = graphene.Field(OrderType, id=graphene.Int())
    def resolve_order(self, info, id):
        return Order.objects.get(id=id)
```
```java
// gRPC (Spring Boot + gRPC)
service OrderService {
    rpc GetOrder (OrderRequest) returns (OrderResponse);
}
```

### Common Mistakes
Junior: প্রতিটা প্রজেক্টে GraphQL ব্যবহার করে ফেলে কারণ এটা "modern" মনে হয়, complexity বিবেচনা না করেই।
Senior: client-এর actual need বিশ্লেষণ করে সঠিক style বেছে নেয় — simple CRUD-এ REST-ই যথেষ্ট, over-engineering এড়ায়।

### Trade-offs
GraphQL flexibility দেয় কিন্তু server-side caching ও rate limiting জটিল করে তোলে (কারণ প্রতিটা query ভিন্ন হতে পারে)। gRPC দ্রুত কিন্তু browser-এ সরাসরি ব্যবহার করা কঠিন (gRPC-Web দরকার হয়)।

---

## 1.3 HTTP Methods ও Status Codes

### API Concept Explanation
- **GET**: resource read করা, safe ও idempotent (side-effect থাকা উচিত না)
- **POST**: নতুন resource তৈরি করা, idempotent না (একই request দুইবার পাঠালে দুইটা resource তৈরি হতে পারে)
- **PUT**: resource সম্পূর্ণ replace করা, idempotent
- **PATCH**: resource আংশিক update করা
- **DELETE**: resource মুছে ফেলা, idempotent

### Status Code Category
```
2xx — Success (200 OK, 201 Created, 204 No Content)
3xx — Redirection (301 Moved Permanently, 304 Not Modified)
4xx — Client Error (400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 429 Too Many Requests)
5xx — Server Error (500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable)
```

### Common Mistakes (খুবই ইন্টারভিউ-common)
Junior: সব success response-এ `200` ব্যবহার করে, resource creation-এও `200` দেয় (`201` হওয়া উচিত)।
Junior: authentication ব্যর্থ হলে `403` দেয়, কিন্তু সঠিক হলো `401` (unauthenticated) — `403` হলো authenticated কিন্তু permission নেই (authorized না)।
Senior: প্রতিটা status code-এর semantic meaning সঠিকভাবে মেনে চলে, client-কে যথাযথ signal দেয়।

### Idempotency — খুবই গুরুত্বপূর্ণ Interview Topic
Idempotent মানে একই request একাধিকবার পাঠালেও ফলাফল একই থাকবে। `GET`, `PUT`, `DELETE` idempotent, কিন্তু `POST` না। Payment API-এর মতো জায়গায় POST idempotent বানাতে **Idempotency Key** ব্যবহার করা হয়:
```python
# Idempotency key দিয়ে duplicate payment রোধ করা
class PaymentView(APIView):
    def post(self, request):
        idempotency_key = request.headers.get('Idempotency-Key')
        existing = Payment.objects.filter(idempotency_key=idempotency_key).first()
        if existing:
            return Response(PaymentSerializer(existing).data)  # আগের result-ই ফেরত
        # নতুন payment process করো
```

### Real-world উদাহরণ
Stripe-এর API-তে `Idempotency-Key` header mandatory suggest করা হয় সব POST request-এ, কারণ network timeout-এ client retry করলে যেন duplicate charge না হয়।

---

## 1.4 Stateless Systems

### API Concept Explanation
REST-এর একটা মূল constraint হলো **statelessness** — প্রতিটা request-এ সব প্রয়োজনীয় তথ্য থাকতে হবে, server কোনো client-এর "আগের অবস্থা" মনে রাখবে না। এটাই horizontal scaling সহজ করে তোলে, কারণ যেকোনো server যেকোনো request handle করতে পারে (session affinity লাগে না)।

### Production Failure যদি Ignore করো
Stateful design (server-এ session data in-memory রাখা) করলে load balancer-কে "sticky session" রাখতে হয় — একই user সবসময় একই server-এ যেতে হবে, যা load distribution অসম করে দেয় এবং সেই server down হলে user-এর session হারিয়ে যায়।

### Senior Engineer Mindset
Stateless design করলে caching, retry, ও horizontal scaling সহজ হয় — তাই senior engineer সবসময় server-side state minimize করার চেষ্টা করে, session data client-এ (JWT-তে) অথবা centralized store (Redis)-এ রাখে, in-memory local state-এ না।

### 🔹 Interview Questions (Module 1 সম্পূর্ণ)

**Beginner**
1. API কী এবং এটা কেন দরকার?
2. REST, RPC, ও GraphQL-এর মধ্যে মূল পার্থক্য কী?
3. GET আর POST-এর মধ্যে পার্থক্য কী?
4. `401` আর `403` status code-এর পার্থক্য কী?

**Intermediate**
5. Idempotency কী এবং কোন HTTP method idempotent?
6. Statelessness কেন REST API-এর মূল constraint এবং এটা scaling-এ কীভাবে সাহায্য করে?
7. কখন REST-এর বদলে GraphQL ব্যবহার করবে?

**Advanced/Senior**
8. Payment API-এর মতো non-idempotent operation-কে কীভাবে idempotent বানাবে?
9. gRPC কেন REST-এর চেয়ে দ্রুত internal microservice communication-এ, technical কারণ ব্যাখ্যা করো (HTTP/2, Protocol Buffers)।
10. Stateless architecture-এ user session data কোথায় এবং কীভাবে রাখবে?

**FAANG System Design**
11. একটা multi-region deployment-এ statelessness কীভাবে global scaling-কে সহজ করে তোলে ব্যাখ্যা করো একটা real example দিয়ে।
12. একটা public API design করো (যেমন Twitter API-এর মতো) যেখানে REST vs GraphQL-এর সিদ্ধান্ত justify করতে হবে।

**Scenario-Based**
13. একটা client complain করছে network issue-এর কারণে তাদের payment request দুইবার গিয়েছে এবং দুইবার charge হয়ে গেছে — root cause এবং fix কী? (উত্তর: idempotency key না থাকা, POST অ-idempotent হওয়া)

---

# 🏗 MODULE 2: API DESIGN PRINCIPLES (FAANG Level)

---

## 2.1 RESTful Design ও Resource Modeling

### 1. Topic Name
Resource-Oriented API Design

### 2. কেন শিখবে?
ভালো API design সরাসরি developer experience, maintainability, এবং scalability প্রভাবিত করে। খারাপ design করা API পরে পরিবর্তন করা extremely costly, কারণ অনেক client ইতিমধ্যে সেটার উপর নির্ভরশীল হয়ে যায়।

### 3-4. Problem ও Production Failure
Action-based URI (`/getUserOrders`, `/createNewOrder`) ব্যবহার করলে API অসংগত (inconsistent) হয়ে যায় এবং caching/tooling সঠিকভাবে কাজ করে না। Resource-based design (`/users/{id}/orders`) স্ট্যান্ডার্ড HTTP semantics ব্যবহার করে যা tooling (browser cache, API gateway) automatically বুঝতে পারে।

### 5. Real-world উদাহরণ
GitHub API-এর design ব্যাপকভাবে RESTful resource modeling-এর "gold standard" হিসেবে গণ্য করা হয় — `/repos/{owner}/{repo}/issues/{issue_number}` এর মতো hierarchical, predictable resource path।

### 6. API Concept — Resource Modeling Rules
- URI-তে noun ব্যবহার করো, verb না (`/orders` না `/getOrders`)
- Plural noun ব্যবহার করো consistency-র জন্য (`/products` না `/product`)
- Hierarchical relationship প্রকাশ করো nested path দিয়ে (`/users/{id}/orders`)
- Action যা resource-এর সাথে মেলে না তার জন্য sub-resource তৈরি করো (`/orders/{id}/cancel` POST দিয়ে, `/cancelOrder` না)

### 7. DRF Implementation
```python
# ভালো design - ViewSet দিয়ে resource-oriented routing
class OrderViewSet(viewsets.ModelViewSet):
    queryset = Order.objects.all()
    serializer_class = OrderSerializer

    @action(detail=True, methods=['post'])
    def cancel(self, request, pk=None):
        order = self.get_object()
        order.status = 'cancelled'
        order.save()
        return Response({"status": "cancelled"})

# router auto-generates: GET/POST /orders, GET/PUT/DELETE /orders/{id}, POST /orders/{id}/cancel
router = DefaultRouter()
router.register('orders', OrderViewSet)
```

### 8. Spring Boot Equivalent
```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    @GetMapping("/{id}")
    public Order getOrder(@PathVariable Long id) { ... }

    @PostMapping("/{id}/cancel")
    public ResponseEntity<?> cancelOrder(@PathVariable Long id) { ... }
}
```

### 9-10. Lifecycle ও Diagram
```
/users                 --> Collection resource (GET: list, POST: create)
/users/{id}            --> Single resource (GET, PUT, PATCH, DELETE)
/users/{id}/orders     --> Nested collection (user-এর orders)
/orders/{id}/cancel    --> Action on resource (POST, কারণ state পরিবর্তন করছে)
```

### 11. Performance Impact
Consistent resource design HTTP caching (`ETag`, `Cache-Control`) সঠিকভাবে কাজ করতে দেয় — inconsistent action-based URI caching layer-কে বিভ্রান্ত করে।

### 12. Common Mistakes (Junior vs Senior)
- Junior: `/getUserById?id=5`, `/deleteUser?id=5` — verb-based, query param দিয়ে ID পাঠায়
- Senior: `GET /users/5`, `DELETE /users/5` — resource-based, path parameter ব্যবহার করে
- Junior: gভুলভাবে nested resource ৪-৫ লেভেল গভীর করে ফেলে (`/companies/{id}/departments/{id}/teams/{id}/members/{id}/tasks`)
- Senior: nesting সর্বোচ্চ ২ লেভেলে রাখে, তারপর top-level resource + filter ব্যবহার করে (`/tasks?team_id=X`)

### 13-18. Debugging, Optimization, Trade-off, Best Practices, Security, Mindset
Best practice: HTTP method দিয়ে action বোঝানো, URI দিয়ে শুধু resource identify করা। Senior mindset: "এই resource-এর সাথে অন্য resource-এর সম্পর্ক কী, এবং URI structure কি সেই সম্পর্ক স্বাভাবিকভাবে প্রকাশ করে?"

---

## 2.2 Pagination, Filtering, Sorting

### 1-2. Topic ও কেন শিখবে
বড় dataset পুরোটা একসাথে response-এ পাঠানো অসম্ভব ও অকার্যকর — millions of record একসাথে পাঠালে memory এবং network দুটোই overload হয়ে যাবে। Pagination, filtering, sorting এই সমস্যার সমাধান।

### 3-4. Problem ও Failure
Pagination ছাড়া কোনো API-তে `/products` কল করলে যদি ডেটাবেসে ১০ মিলিয়ন প্রোডাক্ট থাকে, পুরো response তৈরি করতে গিয়েই server crash করতে পারে বা timeout হয়ে যাবে।

### 6. API Concept — Pagination Types
- **Offset-based**: `?limit=20&offset=40` — সহজ কিন্তু বড় offset-এ slow (DB-কে সব skip করা row scan করতে হয়) এবং নতুন data insert হলে page shift হয়ে যেতে পারে (inconsistent result)
- **Cursor-based**: `?cursor=eyJpZCI6NDB9&limit=20` — একটা opaque pointer ব্যবহার করে পরবর্তী page-এর শুরু চিহ্নিত করে, বড় dataset-এ efficient এবং consistent, real-time feed (Twitter, Instagram)-এ ব্যবহৃত

### 7. DRF Implementation
```python
# Cursor-based pagination (বড় dataset-এর জন্য recommended)
from rest_framework.pagination import CursorPagination

class OrderCursorPagination(CursorPagination):
    page_size = 20
    ordering = '-created_at'

class OrderViewSet(viewsets.ModelViewSet):
    pagination_class = OrderCursorPagination
    queryset = Order.objects.all()

# Filtering ও sorting
class OrderFilter(django_filters.FilterSet):
    status = django_filters.CharFilter()
    min_price = django_filters.NumberFilter(field_name='total', lookup_expr='gte')

    class Meta:
        model = Order
        fields = ['status', 'min_price']

# usage: GET /orders?status=shipped&min_price=100&ordering=-created_at
```

### 8. Spring Boot Equivalent
```java
@GetMapping("/orders")
public Page<Order> getOrders(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size,
        @RequestParam(defaultValue = "createdAt,desc") String sort) {
    Pageable pageable = PageRequest.of(page, size, Sort.by(sort.split(",")[0]));
    return orderRepository.findAll(pageable);
}
```

### 9-10. Lifecycle ও Diagram
```
Client --> GET /orders?cursor=XYZ&limit=20&status=shipped
       --> DB query: WHERE id > decode(cursor) AND status='shipped' ORDER BY id LIMIT 20
       --> Response: {results: [...], next_cursor: "encoded_last_id"}
```

### 11. Performance Impact
Offset-based pagination বড় offset-এ O(n) হয়ে যায় (DB-কে সব আগের row skip করতে হয়), কিন্তু cursor-based pagination indexed column ব্যবহার করে O(log n)-এর কাছাকাছি থাকে — বড় scale-এ এটাই সবচেয়ে গুরুত্বপূর্ণ পার্থক্য।

### 12. Common Mistakes
Junior: সব ক্ষেত্রে offset-based pagination ব্যবহার করে, বড় dataset-এও (Instagram feed-এর মতো জায়গায় এটা ব্যবহার করলে page ১০০০-এ গিয়ে extremely slow হয়ে যাবে)।
Senior: real-time/infinite-scroll feed-এ cursor-based, admin panel-এর মতো "page X-এ যাও" প্রয়োজনে offset-based ব্যবহার করে (কারণ cursor-এ arbitrary page jump করা যায় না)।

### 15. Trade-offs
Cursor-based pagination-এ "মোট কত page আছে" বা "সরাসরি ৫ নম্বর page-এ যাও" এই ধরনের feature দেয়া কঠিন — যেখানে এই feature দরকার (যেমন admin dashboard), সেখানে offset-based-ই সহজ সমাধান।

---

## 2.3 API Idempotency Deep Dive (Design Perspective)

### API Concept
Design-level এ idempotency নিশ্চিত করতে হলে resource creation-এর সময় client-generated ID অথবা idempotency key pattern ব্যবহার করা হয়, যাতে network retry duplicate resource তৈরি না করে।

### Real-world উদাহরণ
AWS API-গুলোতে (EC2 instance launch) `ClientToken` parameter থাকে ঠিক এই কারণে — network timeout হয়ে client retry করলেও একই instance দুইবার launch হবে না।

### Common Mistakes
Junior ভাবে idempotency শুধু GET/PUT/DELETE-এর জন্য প্রযোজ্য, POST কখনো idempotent বানানো যায় না — এটা ভুল ধারণা, design pattern দিয়ে POST-ও idempotent বানানো যায়।

### 🔹 Interview Questions (Module 2 সম্পূর্ণ)

**Beginner**
1. Resource-based URI design কী এবং action-based URI-এর চেয়ে কেন ভালো?
2. Pagination কেন দরকার বড় dataset-এ?
3. Filtering ও sorting কীভাবে API design-এ implement করা হয়?

**Intermediate**
4. Offset-based ও cursor-based pagination-এর মধ্যে পার্থক্য কী?
5. Nested resource URI কতটা গভীর হওয়া উচিত এবং কেন?
6. Client-generated idempotency key কীভাবে duplicate resource creation prevent করে?

**Advanced/Senior**
7. Cursor-based pagination কীভাবে implement করবে যেখানে sorting field-এ duplicate value থাকতে পারে (যেমন একই timestamp-এ একাধিক record)?
8. একটা API design করো যেখানে filtering, sorting, এবং cursor pagination একসাথে কাজ করবে consistent ভাবে।
9. Real-time feed-এ (নতুন ডেটা ক্রমাগত insert হচ্ছে) pagination কীভাবে design করবে যাতে duplicate/skip না হয়?

**FAANG System Design**
10. Instagram/Twitter-এর মতো infinite-scroll feed-এর pagination system design করো ১০০ মিলিয়ন ইউজারের স্কেলে।
11. একটা e-commerce search API design করো যেখানে filtering (price, category, rating), sorting, এবং pagination সব efficient ভাবে কাজ করবে বিলিয়ন প্রোডাক্টের ডেটাসেটে।

**Scenario-Based**
12. একজন ইউজার complain করছে feed scroll করার সময় কিছু পোস্ট দুইবার দেখাচ্ছে বা কিছু পোস্ট মিস হয়ে যাচ্ছে — root cause কী হতে পারে? (উত্তর: offset-based pagination ব্যবহার করা হয়েছে এমন dataset-এ যেখানে নতুন data ক্রমাগত insert/delete হচ্ছে — cursor-based-এ migrate করা উচিত)

### 🧾 Interview Answers (Detailed)

**Q12: Feed-এ duplicate/missing পোস্ট সমস্যার root cause এবং fix**
Bengali Explanation: এই সমস্যাটা ক্লাসিক "offset-based pagination-এর সাথে dynamic dataset"-এর conflict। ধরা যাক ইউজার page ১ দেখলো (item ১-২০), এর মধ্যে নতুন ৫টা পোস্ট insert হলো টপে। যখন ইউজার page ২ request করে (offset=20), ডেটাবেস আবার নতুন করে count করে ২০ নম্বর position থেকে item দেয় — কিন্তু নতুন ৫টা item এর কারণে যেটা আগে position ২১-৪০ এ ছিল সেটা এখন position ২৬-৪৫ এ চলে গেছে, ফলে ৫টা item ডুপ্লিকেট দেখায় (আগে page ১-এ যেটা ছিল সেটাই আবার page ২-তে চলে আসে)।
Real-world example: এই কারণেই Twitter, Instagram, Facebook সবাই তাদের feed API-তে cursor-based pagination ব্যবহার করে — cursor একটা নির্দিষ্ট item-কে পয়েন্ট করে (যেমন "id > 12345 থেকে পরের আইটেম দাও"), নতুন insert হলেও এই reference point পরিবর্তন হয় না।
When to use offset-based: static বা কম পরিবর্তনশীল dataset-এ (যেমন admin-এর product catalog management panel, যেখানে "সরাসরি ১০ নম্বর page-এ যাও" ফিচার দরকার)।
When NOT to use: real-time feed, notification list, বা কোনো frequently-updated dataset।
Trade-offs: Cursor-based pagination-এ "মোট কতগুলো page আছে" বা "শেষ page-এ যাও" এই ধরনের UX ফিচার দেয়া কঠিন — এটা মেনে নিতে হয় consistency-র বিনিময়ে।
Common mistake: candidates শুধু বলে "caching সমস্যা" — কিন্তু আসল root cause হলো pagination strategy-এর ভুল choice, caching এখানে root cause না।
Senior-level answer structure: Symptom বর্ণনা → Root cause (offset + dynamic data) → Solution (cursor-based migrate) → Trade-off বলা।

---

# 🧾 MODULE 1 & 2 — CHEAT SHEET

| Topic | মূল Concept | Key Point |
|---|---|---|
| API Fundamentals | Client-server contract, abstraction layer | REST vs RPC vs GraphQL — context অনুযায়ী choose করো |
| HTTP Methods | GET/POST/PUT/PATCH/DELETE semantics | Idempotency মনে রাখো: GET/PUT/DELETE idempotent, POST না |
| Status Codes | 2xx/3xx/4xx/5xx categories | 401 = unauthenticated, 403 = unauthorized (permission নেই) |
| Statelessness | Server কোনো client state রাখে না | Horizontal scaling সহজ করে, sticky session এড়ায় |
| Resource Modeling | Noun-based URI, HTTP method দিয়ে action | `/orders/{id}/cancel` (POST), না `/cancelOrder` |
| Pagination | Offset-based vs Cursor-based | Dynamic/real-time data-এ cursor-based, static-এ offset-based |
| Idempotency Key | Client-generated token duplicate prevent করে | Payment/order creation API-তে mandatory practice |

## Senior Engineer Checklist ✅
- [ ] প্রতিটা endpoint resource-based (noun), action-based (verb) না?
- [ ] HTTP method ও status code সঠিক semantic মেনে ব্যবহার করা হচ্ছে কি?
- [ ] বড়/dynamic dataset-এ cursor-based pagination ব্যবহার করা হয়েছে কি, offset-based না?
- [ ] Non-idempotent operation (payment, order creation)-এ idempotency key implement করা হয়েছে কি?
- [ ] API design করার আগে consumer-এর actual need বিশ্লেষণ করা হয়েছে, নাকি সরাসরি DB table JSON-এ convert করা হয়েছে?
- [ ] Nested resource URI ২ লেভেলের বেশি গভীর হয়ে যায়নি তো?

## Key Takeaways
1. API design হলো প্রথমেই "চিন্তা করার" কাজ, "কোড লেখার" কাজ না — resource, HTTP semantics, ও consumer need আগে বোঝা জরুরি।
2. Pagination strategy বেছে নেয়া নির্ভর করে dataset কতটা dynamic তার উপর — এটা প্রায়ই ইন্টারভিউতে ভুল বোঝা একটা টপিক।
3. Idempotency শুধু "সংজ্ঞা মুখস্থ রাখার" বিষয় না, এটা বাস্তব production bug (duplicate payment, duplicate order) prevent করার একটা core design principle।

---

📌 **পরবর্তী ধাপ**: Module 3 (DRF Deep Dive: Serializers, Views/ViewSets, Authentication, Permissions, Middleware, Validation) এবং Module 4 (Performance Optimization: N+1 problem, Caching, Query optimization, Pagination strategies) — এখনো এই বিস্তারিত ফরম্যাটে বাকি আছে। বললেই তৈরি করে দেবো।
