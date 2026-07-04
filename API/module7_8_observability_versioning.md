# 📊 MODULE 7: OBSERVABILITY (FAANG Level)

---

## 7.1 Logging

### 1. Topic Name
Structured Logging

### 2. কেন শিখবে?
Production system-এ কিছু ভুল হলে debugger attach করা যায় না — একমাত্র log-ই বলে দেয় ভেতরে কী ঘটেছিল। Millions of request-এর মধ্যে একটা নির্দিষ্ট user-এর সমস্যা খুঁজে বের করতে হলে ভালো logging ছাড়া সম্ভবই না।

### 3. Problem এটা কী সমাধান করে
- Production-এ কী ঘটছে তার visibility দেয়
- Post-incident root cause analysis সহজ করে
- Audit trail তৈরি করে (কে কখন কী করলো)

### 4. Production Failure যদি Ignore করো
Unstructured `print()` বা plain text log দিয়ে millions of line-এর মধ্যে একটা নির্দিষ্ট request খুঁজে বের করা কার্যত অসম্ভব হয়ে যায়। ২০২১ সালের বিখ্যাত Facebook outage-এর সময় engineer-রা নিজেদের internal tool-এই access পাচ্ছিলো না, কারণ observability system নিজেই একই infrastructure-এর উপর নির্ভরশীল ছিল — এখান থেকে শেখার বিষয় হলো logging/monitoring system-কে core service থেকে আলাদা রাখা উচিত।

### 5. Real-world উদাহরণ
- **Netflix**: প্রতিটা microservice request-এ `correlation_id` attach করে, যাতে একটা user request ১৫টা service-এ গেলেও পুরো trail follow করা যায়
- **Amazon**: CloudWatch Logs দিয়ে centralized logging, structured JSON format-এ
- **Google**: internal logging pipeline (Dapper-এর পূর্বসূরি) থেকেই distributed tracing-এর ধারণা এসেছে

### 6. API Concept Explanation
Structured logging মানে log entry plain text না হয়ে JSON/key-value format-এ থাকা, যাতে machine parse করতে পারে:
```json
{"timestamp": "2026-07-04T10:00:00Z", "level": "ERROR", "request_id": "abc-123",
 "user_id": 4521, "endpoint": "/api/orders", "message": "Payment gateway timeout", "duration_ms": 3200}
```
Log Level বোঝা জরুরি: `DEBUG` < `INFO` < `WARNING` < `ERROR` < `CRITICAL`। Production-এ সাধারণত `INFO` বা তার উপরে রাখা হয় (DEBUG সবসময় on রাখলে volume/cost বেড়ে যায়)।

### 7. Django REST Framework Implementation
```python
import logging, json

logger = logging.getLogger('api')

class RequestLoggingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        request_id = request.headers.get('X-Request-ID', str(uuid.uuid4()))
        response = self.get_response(request)
        logger.info(json.dumps({
            "request_id": request_id,
            "path": request.path,
            "method": request.method,
            "status": response.status_code,
            "user": getattr(request.user, 'id', None),
        }))
        return response
```

### 8. Spring Boot Equivalent
```java
@Slf4j
@Component
public class RequestLoggingFilter extends OncePerRequestFilter {
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws ServletException, IOException {
        MDC.put("requestId", UUID.randomUUID().toString());
        long start = System.currentTimeMillis();
        chain.doFilter(req, res);
        log.info("path={} status={} duration={}ms", req.getRequestURI(), res.getStatus(),
                  System.currentTimeMillis() - start);
        MDC.clear();
    }
}
```

### 9. Internal Request Lifecycle
Request আসে → middleware/filter `request_id` generate/extract করে → পুরো request lifecycle-এ এই id প্রতিটা log line-এ attach থাকে → log aggregator (ELK/Splunk/CloudWatch)-এ পাঠানো হয় → engineer পরে `request_id` দিয়ে search করে পুরো trail দেখে।

### 10. Architecture Diagram (Text)
```
App Server --logs (JSON)--> Log Shipper (Fluentd/Filebeat) --> Log Aggregator (ELK/CloudWatch)
                                                                      |
                                                              Dashboard/Alerting (Kibana/Grafana)
```

### 11. Performance Impact
Synchronous logging (disk-এ সরাসরি write) request latency বাড়ায়। Senior practice: async logging (buffer + background flush) ব্যবহার করা, এবং log volume নিয়ন্ত্রণে sampling করা high-traffic endpoint-এ।

### 12. Common Mistakes (Junior vs Senior)
- Junior: `print(user.password)` এর মতো sensitive data log করে ফেলে
- Senior: PII/sensitive data mask করে log করে (`email: a***@example.com`)
- Junior: request_id ছাড়া log রাখে — distributed system-এ trace করা অসম্ভব হয়ে যায়
- Senior: প্রতিটা log-এ correlation/request id মেন্ডেটরি রাখে

### 13. Debugging Strategy
Correlation ID দিয়ে সব service-এর log একসাথে filter করে পুরো request journey দেখা। Log level সাময়িকভাবে DEBUG-এ বাড়িয়ে নির্দিষ্ট issue reproduce করা।

### 14. Optimization Techniques
Log sampling (উচ্চ ট্রাফিক endpoint-এ প্রতিটা request log না করে ১০%-এ sample করা), async log shipping, log retention policy (৩০-৯০ দিন পর archive/delete)।

### 15. Trade-offs (কখন কম করবে)
অতিরিক্ত verbose logging storage cost অনেক বাড়ায় এবং search performance কমায়। ছোট application-এ enterprise-grade log pipeline (ELK stack) over-engineering হতে পারে — শুরুতে simple centralized log (CloudWatch) যথেষ্ট।

### 16. Production Best Practices
Structured (JSON) log, correlation ID mandatory, sensitive data masking, log level environment অনুযায়ী configurable, centralized aggregation।

### 17. Security Considerations
Log-এ password, token, credit card info কখনো raw রাখা উচিত না — GDPR/PCI-DSS compliance ভঙ্গ হতে পারে। Log access নিজেই RBAC দিয়ে protect করা উচিত।

### 18. Senior Engineer Design Mindset
সিনিয়র ইঞ্জিনিয়ার ভাবে: "৩ মাস পর যদি এই bug production-এ আসে, আমি কি বর্তমান log দিয়ে root cause বের করতে পারবো?" — এই প্রশ্ন থেকেই logging strategy design হয়, শুধু "কাজ করছে" এটুকু যথেষ্ট না।

### 🔹 Interview Questions

**Beginner**
1. Structured logging কী এবং এটা plain text logging থেকে কীভাবে ভালো?
2. Log level (DEBUG, INFO, WARNING, ERROR)-এর পার্থক্য কী?
3. Log aggregation কেন দরকার?

**Intermediate**
4. Correlation ID / Request ID কী এবং distributed system-এ এটা কেন জরুরি?
5. Synchronous vs asynchronous logging — কোনটা কখন ব্যবহার করবে?
6. Sensitive data logging কীভাবে prevent করবে?

**Advanced/Senior**
7. High-throughput system-এ logging cost ও performance impact কীভাবে balance করবে?
8. Log sampling strategy কীভাবে design করবে যাতে rare error miss না হয়?
9. Multi-region deployment-এ centralized logging pipeline কীভাবে design করবে?

**FAANG System Design**
10. Netflix-scale system-এ প্রতিদিন petabytes log data efficiently store ও query করার জন্য pipeline design করো।
11. Log pipeline নিজেই fail করলে (log aggregator down) কীভাবে graceful degrade করবে?

**Scenario-Based**
12. Production-এ একটা critical bug হয়েছে কিন্তু log-এ পর্যাপ্ত তথ্য নেই root cause বের করার জন্য — ভবিষ্যতে এটা এড়াতে কী করবে?

### 🧾 Interview Answers (Detailed)

**Q7: High-throughput system-এ logging cost ও performance কীভাবে balance করবে?**
Bengali Explanation: প্রথমে বুঝতে হবে কোন log গুরুত্বপূর্ণ — error/warning সবসময় ১০০% log করা উচিত, কিন্তু success case (INFO level)-এ sampling করা যায় (যেমন প্রতি ১০০ request-এ ১টা log)। Asynchronous logging ব্যবহার করে main request thread-কে block না করা, এবং log buffer local-এ রেখে batch-এ ship করা (network overhead কমাতে)।
Real-world example: Google-এর internal logging system-এ error/critical log সবসময় সম্পূর্ণ capture হয়, কিন্তু debug-level trace log-এ head-based sampling apply করা হয়।
When to use sampling: high-QPS (queries per second) read-heavy endpoint-এ (যেমন search, feed)।
When NOT to use sampling: payment, authentication-এর মতো critical path-এ — এখানে প্রতিটা log দরকার হতে পারে audit/compliance-এর জন্য।
Trade-offs: sampling করলে storage cost কমে কিন্তু rare intermittent bug miss হওয়ার ঝুঁকি থাকে — তাই error rate spike হলে temporarily sampling rate কমিয়ে ১০০% এ নেয়ার mechanism রাখা senior-level practice।
Common mistake: candidates শুধু "log level কমিয়ে দাও" বলে — কিন্তু সঠিক উত্তর হলো selective sampling + async shipping + tiered retention।
Senior-level answer structure: Problem (cost vs visibility trade-off) → Solution (sampling + async + tiered storage) → Trade-off বলা → Real example দেয়া।

---

## 7.2 Monitoring

### 1-2. Topic ও কেন শিখবে
Monitoring হলো system health real-time দেখার ক্ষমতা — CPU, memory, error rate, latency ইত্যাদি। এটা ছাড়া আপনি জানবেনই না system কখন খারাপ অবস্থায় যাচ্ছে, যতক্ষণ না user complain করে।

### 3-4. Problem ও Production Failure
Monitoring ছাড়া "silent failure" ধরা পড়ে না — যেমন একটা background job আস্তে আস্তে slow হয়ে গিয়ে queue backlog তৈরি করছে, কিন্তু কেউ জানছে না যতক্ষণ না পুরো system জ্যাম হয়ে যায়।

### 5. Real-world উদাহরণ
- Amazon-এর প্রতিটা service-এ SLA (৯৯.৯৯%) monitor করা হয় real-time dashboard দিয়ে
- Google-এর SRE (Site Reliability Engineering) team সম্পূর্ণভাবে monitoring/alerting-এর উপর নির্ভর করে "error budget" মেপে

### 6. API Concept
Monitoring মূলত তিনটা স্তরে হয়: **Infrastructure** (CPU, memory, disk), **Application** (error rate, latency, throughput), **Business** (order count, revenue, signup rate)। Health check endpoint (`/health`, `/readiness`, `/liveness`) দিয়ে service-এর অবস্থা report করা হয়।

### 7. DRF Implementation
```python
# health check endpoint
from django.http import JsonResponse
from django.db import connection

def health_check(request):
    try:
        connection.cursor().execute("SELECT 1")
        db_status = "ok"
    except Exception:
        db_status = "down"
    return JsonResponse({"status": "ok" if db_status == "ok" else "degraded", "db": db_status})
```

### 8. Spring Boot Equivalent
```java
// Spring Boot Actuator built-in monitoring endpoints
management.endpoints.web.exposure.include=health,metrics,prometheus
```
```java
@Component
public class CustomHealthIndicator implements HealthIndicator {
    public Health health() {
        return isDbHealthy() ? Health.up().build() : Health.down().withDetail("db", "unreachable").build();
    }
}
```

### 9-10. Lifecycle ও Diagram
```
Application --exposes--> /health, /metrics endpoints
Monitoring Tool (Prometheus) --scrapes-->  metrics periodically
                                    --> Alerting Rules (Alertmanager) --> PagerDuty/Slack
```

### 11. Performance Impact
Health check endpoint lightweight রাখা উচিত (heavy DB query না) — কারণ load balancer প্রতি কয়েক সেকেন্ডে এটা call করে, ভারী হলে নিজেই bottleneck তৈরি করবে।

### 12. Common Mistakes
Junior: শুধু "server up আছে কিনা" চেক করে, downstream dependency (DB, cache, third-party API) চেক করে না।
Senior: readiness probe-এ critical dependency চেক করে, কিন্তু liveness probe হালকা রাখে (infinite restart loop এড়াতে)।

### 13-18. Debugging, Optimization, Trade-off, Best Practices, Security, Mindset
Golden signals মনে রাখা: **Latency, Traffic, Errors, Saturation** (Google SRE বই থেকে)। Senior mindset: "আমি কি alert পাওয়ার আগে user complain করার আগে সমস্যা ধরতে পারবো?"

### 🔹 Interview Questions
**Beginner**: Monitoring কী এবং কেন দরকার? Health check endpoint কী?
**Intermediate**: Liveness probe ও Readiness probe-এর পার্থক্য কী (Kubernetes context-এ)?
**Senior**: SLA, SLO, SLI-এর মধ্যে পার্থক্য কী এবং কীভাবে design করবে?
**FAANG**: Google-scale search engine-এর জন্য monitoring dashboard design করো যা real-time-এ ৪টা golden signal track করে।
**Scenario**: Health check pass করছে কিন্তু user error পাচ্ছে — কী ভুল হতে পারে?

---

## 7.3 Metrics

### 1-2. Topic ও কেন শিখবে
Metrics হলো numeric data over time (time-series) — যেমন request count, latency p99, error rate। এটা দিয়েই trend, anomaly, capacity planning করা হয়।

### 3-4. Problem ও Failure
Metrics ছাড়া capacity planning করা যায় না — কবে server আরও লাগবে সেটা জানা যায় না, ফলে হঠাৎ traffic spike-এ system crash করে (Black Friday-এর মতো high-traffic event-এ e-commerce site-গুলোর জন্য এটা critical)।

### 5. Real-world উদাহরণ
Prometheus + Grafana combo industry-standard — Uber, Airbnb, SoundCloud সবাই এটা ব্যবহার করে। Amazon CloudWatch Metrics নিজস্ব AWS ecosystem-এর জন্য।

### 6. API Concept — Metric Types
- **Counter**: শুধু বাড়ে (total requests served)
- **Gauge**: up-down হতে পারে (current active connections)
- **Histogram**: distribution বোঝায় (latency buckets: p50, p95, p99)
- **Summary**: histogram-এর মতো কিন্তু client-side percentile calculate করে

### 7. DRF Implementation
```python
from prometheus_client import Counter, Histogram

REQUEST_COUNT = Counter('api_requests_total', 'Total API requests', ['method', 'endpoint', 'status'])
REQUEST_LATENCY = Histogram('api_request_duration_seconds', 'Request latency')

class MetricsMiddleware:
    def __call__(self, request):
        with REQUEST_LATENCY.time():
            response = self.get_response(request)
        REQUEST_COUNT.labels(request.method, request.path, response.status_code).inc()
        return response
```

### 8. Spring Boot Equivalent (Micrometer)
```java
@Timed(value = "api.request.duration", description = "API request latency")
@GetMapping("/orders")
public List<Order> getOrders() { ... }
```

### 9-10. Lifecycle ও Diagram
```
App --exposes /metrics endpoint (Prometheus format)--> Prometheus Server (scrape every 15s)
                                                              --> Grafana Dashboard
                                                              --> Alertmanager (threshold breach)
```

### 11. Performance Impact
Metric collection lightweight হওয়া উচিত — high-cardinality label (যেমন user_id প্রতিটা metric-এ label হিসেবে ব্যবহার) Prometheus-কে overload করে দেয়, এটা common production mistake।

### 12. Common Mistakes
Junior: প্রতিটা unique user_id/request_id-কে metric label বানায় — cardinality explosion হয়ে monitoring system-ই crash করে।
Senior: label-এ শুধু bounded set of value ব্যবহার করে (method, status_code, endpoint — user_id না)।

### 13-18. Debugging, Optimization, Trade-off, Best Practices
p50/p95/p99 latency আলাদা track করা জরুরি — average latency misleading হতে পারে (কিছু slow outlier request average-এ hide হয়ে যায়, কিন্তু p99 সেটা দেখিয়ে দেয়)।

### 🔹 Interview Questions
**Beginner**: Metrics কী এবং logging থেকে কীভাবে আলাদা?
**Intermediate**: p50, p95, p99 latency বলতে কী বোঝায় এবং কেন average latency যথেষ্ট না?
**Senior**: High-cardinality metric problem কী এবং কীভাবে prevent করবে?
**FAANG**: একটা global CDN service-এর জন্য real-time metrics pipeline design করো যা প্রতি সেকেন্ডে millions of data point handle করে।
**Scenario**: Average latency ভালো দেখাচ্ছে কিন্তু কিছু user slow response-এর complain করছে — root cause কীভাবে বের করবে?

---

## 7.4 Tracing

### 1-2. Topic ও কেন শিখবে
একটা single user request microservices architecture-এ ১৫-২০টা service touch করতে পারে। Distributed tracing ছাড়া বোঝা অসম্ভব কোন service-এ delay হচ্ছে।

### 3-4. Problem ও Failure
Tracing ছাড়া "কেন এই request ৫ সেকেন্ড লাগলো" প্রশ্নের উত্তর দেয়া যায় না যখন request ১০টা service touch করে — প্রতিটা service-এর log আলাদা করে দেখতে হবে, যা কার্যত অব্যবহারিক বড় scale-এ।

### 5. Real-world উদাহরণ
Google-এর **Dapper** paper (2010) distributed tracing-এর ভিত্তি স্থাপন করেছিল, যেখান থেকে Zipkin, Jaeger, ও OpenTelemetry-এর জন্ম। Uber নিজস্ব **Jaeger** তৈরি করেছে এবং open-source করেছে।

### 6. API Concept
প্রতিটা request একটা **Trace ID** পায় (পুরো journey identify করতে), এবং প্রতিটা service-এর মধ্যে কাজকে **Span** বলা হয় (parent-child relationship সহ)। এই span গুলো একসাথে দেখলে পুরো request-এর waterfall diagram পাওয়া যায়।

### 7. DRF Implementation
```python
from opentelemetry import trace
from opentelemetry.instrumentation.django import DjangoInstrumentor

DjangoInstrumentor().instrument()
tracer = trace.get_tracer(__name__)

def get_order(request, order_id):
    with tracer.start_as_current_span("fetch_order_from_db"):
        order = Order.objects.get(id=order_id)
    with tracer.start_as_current_span("call_payment_service"):
        payment_status = call_payment_api(order.payment_id)
    return JsonResponse({"order": order.id, "payment": payment_status})
```

### 8. Spring Boot Equivalent
```java
// Spring Cloud Sleuth / Micrometer Tracing + Zipkin
@NewSpan("fetch-order")
public Order getOrder(Long id) {
    return orderRepository.findById(id).orElseThrow();
}
```

### 9. Internal Lifecycle
Request → Trace ID generate/propagate হয় header-এ (`traceparent`) → প্রতিটা service নিজের span তৈরি করে trace-এ যোগ করে → সব span collector (Jaeger/Zipkin)-এ পাঠানো হয় → UI-তে waterfall visualize করা হয়।

### 10. Architecture Diagram
```
API Gateway [Span 1: 50ms]
  └─ Order Service [Span 2: 200ms]
       └─ Payment Service [Span 3: 150ms]   <-- এখানেই delay!
       └─ Inventory Service [Span 4: 30ms]
Total Trace Duration: 250ms
```

### 11. Performance Impact
Tracing instrumentation overhead থাকে (প্রতিটা span তৈরি ও export করতে কিছুটা CPU/network লাগে), তাই ১০০% trace না করে sampling (যেমন ১-১০%) করা হয় high-traffic system-এ।

### 12. Common Mistakes
Junior: trace context propagate করে না service-to-service call-এ (async message queue/HTTP call-এ header/context হারিয়ে ফেলে) — ফলে trace ভেঙে যায় (broken trace)।
Senior: context propagation library ব্যবহার করে সব communication layer-এ (HTTP, message queue) trace ID carry করে।

### 13-18. Debugging, Optimization, Trade-off, Best Practices
Trace sampling rate ঠিক করা critical — error trace সবসময় ১০০% capture করা উচিত (tail-based sampling), normal trace-এ lower rate রাখা যায়।

### 🔹 Interview Questions
**Beginner**: Distributed tracing কী এবং কেন দরকার microservices-এ?
**Intermediate**: Trace, Span, এবং Trace ID-এর সম্পর্ক ব্যাখ্যা করো।
**Senior**: Head-based sampling vs tail-based sampling-এর পার্থক্য কী এবং কখন কোনটা ব্যবহার করবে?
**FAANG**: Uber-scale ride-matching system-এ একটা request ২০টা service touch করে — tracing infrastructure কীভাবে design করবে যাতে overhead কম থাকে?
**Scenario**: একটা trace-এ মাঝখানের একটা service-এর span missing — এটা কেন হতে পারে এবং কীভাবে fix করবে?

---

# 🔄 MODULE 8: API VERSIONING (FAANG Level)

---

## 8.1 Versioning Strategies

### 1-2. Topic ও কেন শিখবে
API একবার publish হয়ে গেলে সেটা অনেক client (mobile app, third-party integration) ব্যবহার করে। Breaking change করলে সব client একসাথে ভেঙে যাবে — তাই versioning strategy দরকার পুরনো client চালু রেখে নতুন feature আনার জন্য।

### 3-4. Problem ও Production Failure
Versioning ছাড়া API পরিবর্তন করলে পুরনো mobile app version (যা user আপডেট করেনি) crash করতে শুরু করবে। বাস্তব উদাহরণ: যদি কোনো ই-কমার্স app response field-এর নাম পরিবর্তন করে (`total_price` থেকে `totalPrice`), পুরনো app version-এর সব order-flow ভেঙে যাবে যতক্ষণ না সবাই আপডেট করছে — যা কখনোই ১০০% হয় না।

### 5. Real-world উদাহরণ
- **Stripe**: date-based versioning (`2023-10-16`) — প্রতিটা account নিজস্ব version pin করে রাখতে পারে
- **GitHub API**: header-based versioning (`X-GitHub-Api-Version`)
- **Twitter/X API**: URI-based versioning (`/1.1/`, `/2/`)

### 6. API Concept — Strategies
- **URI Versioning**: `/api/v1/orders`, `/api/v2/orders` — সহজ, discoverable, কিন্তু URL duplication হয়
- **Header Versioning**: `Accept: application/vnd.company.v2+json` — URL clean থাকে, কিন্তু কম discoverable
- **Query Parameter**: `/api/orders?version=2` — সহজ কিন্তু caching-এ সমস্যা করতে পারে
- **Date-based Versioning**: Stripe-এর মতো, প্রতিটা account নিজস্ব pinned date ধরে behavior পায়

### 7. DRF Implementation
```python
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.URLPathVersioning',
    'DEFAULT_VERSION': 'v1',
    'ALLOWED_VERSIONS': ['v1', 'v2'],
}

class OrderView(APIView):
    def get(self, request, *args, **kwargs):
        if request.version == 'v2':
            serializer = OrderSerializerV2(orders, many=True)
        else:
            serializer = OrderSerializerV1(orders, many=True)
        return Response(serializer.data)
```

### 8. Spring Boot Equivalent
```java
@GetMapping(value = "/orders", headers = "X-API-Version=2")
public List<OrderV2Dto> getOrdersV2() { ... }

@GetMapping(value = "/orders", headers = "X-API-Version=1")
public List<OrderV1Dto> getOrdersV1() { ... }
```

### 9. Internal Request Lifecycle
Request আসে → versioning middleware URL/header/param থেকে version resolve করে → সেই version-এর controller/serializer route করে → response সেই version-এর schema অনুযায়ী তৈরি হয়।

### 10. Architecture Diagram
```
Client (App v1) --> /api/v1/orders --> Old Serializer/Logic --> Old Response Shape
Client (App v2) --> /api/v2/orders --> New Serializer/Logic --> New Response Shape
                        (একই database, শুধু presentation layer আলাদা)
```

### 11. Performance Impact
একাধিক version maintain করা মানে duplicate code path, যা test/maintenance overhead বাড়ায়। কিন্তু response transform layer ব্যবহার করলে business logic একটাই রাখা যায়, শুধু output shape ভিন্ন হয়।

### 12. Common Mistakes (Junior vs Senior)
- Junior: version বাড়িয়ে পুরনো version সাথে সাথে বন্ধ করে দেয়
- Senior: deprecation timeline (৬-১২ মাস) দিয়ে gradual migration করায়, দুই version parallel চালায়
- Junior: প্রতিটা ছোট change-এই নতুন major version বানায়
- Senior: backward-compatible change (নতুন optional field যোগ) version bump ছাড়াই করে

### 13. Debugging Strategy
Version-wise traffic monitor করা (কতজন এখনো v1 ব্যবহার করছে) — deprecation decision data-driven করতে।

### 14. Optimization Techniques
Shared business logic layer রেখে শুধু serializer/DTO আলাদা রাখা যাতে duplicate logic না হয়।

### 15. Trade-offs (কখন ব্যবহার করবে না)
Internal-only API (যেখানে client তুমি নিজেই control করো, যেমন internal microservice communication) versioning-এর ভারী overhead না করে সরাসরি coordinate করে দুই পাশ একসাথে deploy করা যায়।

### 16. Production Best Practices
Semantic versioning মেনে চলা (major.minor.patch)। Backward-compatible change-এ version bump না করা — শুধু breaking change-এ নতুন major version।

### 17. Security Considerations
পুরনো deprecated version-এ security patch backport করতে হবে যতদিন সেটা active থাকে — না হলে পুরনো client vulnerable থেকে যায়।

### 18. Senior Engineer Design Mindset
সিনিয়র ইঞ্জিনিয়ার design করার সময়ই ভাবে "এই field ভবিষ্যতে পরিবর্তন হতে পারে কিনা" এবং extensible schema বানায় (additive change আগে থেকে চিন্তা করে), যাতে version bump করার প্রয়োজনই কম পড়ে।

### 🔹 Interview Questions

**Beginner**
1. API versioning কী এবং কেন দরকার?
2. URI versioning ও header versioning-এর পার্থক্য কী?
3. Breaking change আর non-breaking change-এর মধ্যে পার্থক্য কী?

**Intermediate**
4. Semantic versioning (major.minor.patch) কীভাবে কাজ করে?
5. একটা field response থেকে সরিয়ে ফেলা কেন breaking change, আর নতুন field যোগ করা কেন না?
6. Query parameter versioning-এর সমস্যা কী (caching-এর প্রেক্ষিতে)?

**Advanced/Senior**
7. একটা API-তে দুই version parallel maintain করার সময় code duplication কীভাবে কমাবে?
8. Stripe-এর date-based versioning approach কীভাবে কাজ করে এবং এটার সুবিধা কী?
9. Database schema change করতে হলে zero-downtime migration কীভাবে API versioning-এর সাথে coordinate করবে?

**FAANG System Design**
10. একটা payment API-তে ৫ বছর ধরে ব্যবহৃত পুরনো client আছে — নতুন architecture-এ migrate করার versioning strategy design করো।
11. GraphQL-এর মতো schema যেখানে versioning-এর ধারণা REST-এর থেকে ভিন্ন — সেটা কীভাবে evolve করে?

**Scenario-Based**
12. একটা mobile app-এর পুরনো version (৩ বছর আগের) এখনও ৫% ইউজার ব্যবহার করছে, কিন্তু সেই version-এর API maintain করা costly হয়ে যাচ্ছে — কী করবে?

### 🧾 Interview Answers (Detailed)

**Q7: দুই version parallel maintain করার সময় code duplication কীভাবে কমাবে?**
Bengali Explanation: সবচেয়ে ভালো approach হলো business logic layer এবং presentation layer আলাদা করা। Core business logic (order calculate করা, payment process করা) একটাই version-এ রাখা হয়, আর প্রতিটা API version-এর জন্য শুধু একটা thin serializer/adapter layer রাখা হয় যেটা core output-কে সেই version-এর expected shape-এ transform করে।
Real-world example: Stripe তাদের internal object model একরকম রাখে, কিন্তু request আসার সময় pinned API version অনুযায়ী internal middleware সেই object-কে transform করে পুরনো shape-এ রিটার্ন করে — এটাকে তারা "version transformation layer" বলে।
When to use: যখন একাধিক version-এর মধ্যে শুধু response shape/field naming ভিন্ন, business logic একই।
When NOT to use: যদি দুই version-এর মধ্যে actual business logic-ই আলাদা (যেমন v2-তে সম্পূর্ণ নতুন pricing model), তখন duplication এড়ানোর চেষ্টা code-কে জটিল করে ফেলতে পারে — সেক্ষেত্রে সরাসরি আলাদা path রাখাই ভালো।
Trade-offs: transformation layer maintain করা extra complexity যোগ করে, কিন্তু duplicate business logic bug-prone (এক জায়গায় fix করলে আরেক জায়গায় miss হওয়ার ঝুঁকি) এর চেয়ে ভালো।
Common mistake: candidates বলে "শুধু if-else দিয়ে version চেক করবো" — যেটা scale করলে code অগোছালো হয়ে যায়। সঠিক উত্তর হলো layered architecture।
Senior-level answer structure: Problem (duplication risk) → Solution (business logic + adapter layer separation) → Real example (Stripe) → Trade-off।

---

## 8.2 Backward Compatibility

### 1-2. Topic ও কেন শিখবে
Backward compatibility মানে নতুন change করলেও পুরনো client কাজ করতে থাকা উচিত। এটা না মানলে প্রতিটা ছোট change-এ সব client force-update করাতে হবে — যা practically অসম্ভব।

### 3-4. Problem ও Failure
"Robustness Principle" (Postel's Law) না মানলে API breaking হয়ে যায় সহজেই। বাস্তব উদাহরণ: response-এ কোনো field-এর data type পরিবর্তন করলে (string থেকে integer) statically-typed client (Java/Swift mobile app) সাথে সাথে crash করবে parsing error দিয়ে।

### 5. Real-world উদাহরণ
Google-এর Protocol Buffers design principle-ই হলো backward/forward compatibility centric — field number কখনো reuse করা যায় না, নতুন field সবসময় optional হতে হয়।

### 6. API Concept — কী করলে Breaking, কী করলে না
**Safe (Non-breaking)**: নতুন optional field যোগ করা, নতুন endpoint যোগ করা, নতুন optional query parameter যোগ করা।
**Breaking**: existing field remove/rename করা, field-এর data type পরিবর্তন করা, required field নতুন করে mandatory বানানো, error response format পরিবর্তন করা, endpoint remove করা।

### 7. DRF Implementation
```python
class OrderSerializer(serializers.ModelSerializer):
    # নতুন field যোগ করা - backward compatible, পুরনো client শুধু ignore করবে
    estimated_delivery = serializers.DateField(required=False, allow_null=True)

    class Meta:
        model = Order
        fields = ['id', 'total_price', 'status', 'estimated_delivery']
        # পুরনো field কখনো remove না করে deprecated হিসেবে মার্ক করা, ধীরে সরানো
```

### 8. Spring Boot Equivalent
```java
public class OrderDto {
    private Long id;
    private BigDecimal totalPrice;
    @Deprecated
    private String legacyStatus; // পুরনো client-এর জন্য এখনো রাখা হয়েছে
    private String status; // নতুন field
}
```

### 9-10. Lifecycle ও Diagram
```
Old Client --> sends request expecting old schema --> API (returns both old + new fields) --> Old Client works fine
New Client --> sends request using new fields    --> API (returns new fields)              --> New Client works fine
```

### 11. Performance Impact
পুরনো ও নতুন উভয় field response-এ রাখলে payload size সামান্য বাড়ে, কিন্তু এটা compatibility-এর জন্য acceptable trade-off।

### 12. Common Mistakes
Junior: field rename করে ফেলে (`user_name` → `username`) না ভেবে যে পুরনো client এখনো `user_name` expect করছে।
Senior: নতুন field যোগ করে, পুরনোটা deprecated মার্ক করে রাখে যতদিন সব client migrate না করে।

### 13-18. Debugging, Optimization, Trade-off, Best Practices, Security, Mindset
Contract testing (Pact-এর মতো tool) ব্যবহার করে automated ভাবে verify করা যে change backward compatible কিনা, manual review-এর উপর নির্ভর না করে।

### 🔹 Interview Questions
**Beginner**: Backward compatibility কী এবং কেন গুরুত্বপূর্ণ?
**Intermediate**: কোন ধরনের API change breaking এবং কোনটা non-breaking?
**Senior**: Contract testing কী এবং কীভাবে এটা backward compatibility নিশ্চিত করতে সাহায্য করে?
**FAANG**: একটা large-scale system-এ database schema change করতে হবে যেটা API response-কে প্রভাবিত করবে — zero downtime এবং backward compatibility বজায় রেখে কীভাবে করবে?
**Scenario**: একজন developer accidentally একটা required field response থেকে সরিয়ে দিয়েছে যা production-এ কয়েক ঘণ্টা পর কয়েকটা client crash করিয়েছে — এটা ভবিষ্যতে prevent করতে কী process যোগ করবে? (উত্তর: automated contract test + API schema diff CI check)

---

## 8.3 Deprecation

### 1-2. Topic ও কেন শিখবে
পুরনো API version/field চিরকাল রাখা যায় না — maintenance cost বাড়ে। Deprecation হলো নিয়ন্ত্রিতভাবে, client-দের জানিয়ে, সময় দিয়ে পুরনো জিনিস সরিয়ে ফেলার প্রক্রিয়া।

### 3-4. Problem ও Failure
হঠাৎ করে কোনো নোটিশ ছাড়া endpoint বন্ধ করে দিলে third-party integration ভেঙে যায় — এটা company-এর reputation-এর জন্যও খারাপ। Twitter API-এর ইতিহাসে একাধিকবার হঠাৎ deprecation নিয়ে developer community-তে বড় ধরনের ক্ষোভ তৈরি হয়েছিল।

### 5. Real-world উদাহরণ
- **Google**: প্রতিটা deprecated API-তে কমপক্ষে ১ বছরের notice period দেয়, deprecation policy publicly document করা থাকে
- **Stripe**: deprecated field response-এ রেখে দেয় কিন্তু documentation-এ clearly mark করে, changelog দিয়ে notify করে

### 6. API Concept
Deprecation lifecycle: **Announce** → **Warn (response header/log-এ notice)** → **Sunset date নির্ধারণ** → **Monitor usage** → **Remove**।
`Deprecation` ও `Sunset` HTTP header (RFC 8594) ব্যবহার করে client-কে programmatically জানানো যায়।

### 7. DRF Implementation
```python
class LegacyOrderView(APIView):
    def get(self, request):
        response = Response(serializer.data)
        response['Deprecation'] = 'true'
        response['Sunset'] = 'Sat, 31 Dec 2026 23:59:59 GMT'
        response['Link'] = '<https://api.example.com/docs/migration-v2>; rel="successor-version"'
        logger.warning(f"Deprecated endpoint called by client_id={request.user.id}")
        return response
```

### 8. Spring Boot Equivalent
```java
@GetMapping("/v1/orders")
public ResponseEntity<List<OrderDto>> getOrdersLegacy() {
    return ResponseEntity.ok()
        .header("Deprecation", "true")
        .header("Sunset", "Wed, 31 Dec 2026 23:59:59 GMT")
        .body(orders);
}
```

### 9-10. Lifecycle ও Diagram
```
Announce Deprecation (T+0) --> Add warning headers/logs (T+0 to T+N)
   --> Monitor which clients still use it --> Direct outreach to top users
   --> Sunset date reached --> Remove endpoint (with final grace period fallback if needed)
```

### 11. Performance Impact
Deprecation নিজে performance-কে সরাসরি প্রভাবিত করে না, কিন্তু deprecated code path maintain করা technical debt তৈরি করে যা দীর্ঘমেয়াদে development velocity কমায়।

### 12. Common Mistakes
Junior: হঠাৎ endpoint সরিয়ে ফেলে, কোনো notice/monitoring ছাড়া।
Senior: usage metrics দিয়ে ট্র্যাক করে কে এখনো ব্যবহার করছে, gradual sunset timeline অনুসরণ করে, প্রয়োজনে directly client-দের সাথে যোগাযোগ করে migrate করাতে সাহায্য করে।

### 13-18. Debugging, Optimization, Trade-off, Best Practices, Security, Mindset
Best practice: deprecation timeline public documentation-এ রাখা, changelog মেইনটেইন করা, এবং critical client-দের জন্য extended support period offer করা প্রয়োজনে। Senior mindset: "deprecation হলো technical decision না শুধু, এটা একটা communication ও trust-building process client-দের সাথে।"

### 🔹 Interview Questions
**Beginner**: API Deprecation কী এবং এটা কীভাবে client-কে জানানো হয়?
**Intermediate**: `Deprecation` ও `Sunset` HTTP header কীভাবে কাজ করে?
**Senior**: একটা API-এর deprecation policy কীভাবে design করবে যাতে client trust বজায় থাকে?
**FAANG**: একটা enterprise API-তে হাজার হাজার third-party integration আছে — একটা major breaking change roll out করার সম্পূর্ণ deprecation ও migration strategy design করো।
**Scenario**: Sunset date পার হয়ে যাওয়ার পরও ৫% traffic এখনো পুরনো endpoint-এ আসছে — এখন কী করবে? (উত্তর: extended grace period সহ direct outreach, অথবা read-only fallback রাখা, কিন্তু নির্দিষ্ট সময়ের মধ্যে হার্ড কাট-অফ)

---

# 🧾 MODULE 7 & 8 — CHEAT SHEET

| Topic | মূল Concept | Key Tool/Library |
|---|---|---|
| Logging | Structured (JSON), correlation ID, sensitive data masking | ELK Stack, CloudWatch Logs, Fluentd |
| Monitoring | Health check, golden signals (Latency, Traffic, Errors, Saturation) | Spring Boot Actuator, Django health check |
| Metrics | Counter/Gauge/Histogram, p50/p95/p99 | Prometheus + Grafana, Micrometer |
| Tracing | Trace ID, Span, waterfall visualization | Jaeger, Zipkin, OpenTelemetry |
| Versioning Strategy | URI/Header/Date-based versioning | DRF Versioning, Spring `@RequestMapping headers` |
| Backward Compatibility | Additive change safe, remove/rename unsafe | Contract testing (Pact), API schema diff |
| Deprecation | Announce → Warn → Sunset → Remove | `Deprecation`/`Sunset` HTTP headers (RFC 8594) |

## Senior Engineer Checklist ✅
- [ ] প্রতিটা log entry-তে request/correlation ID আছে কি?
- [ ] Sensitive data (password, token, PII) log-এ mask করা হয়েছে কি?
- [ ] Health check endpoint downstream dependency (DB, cache) verify করে কি, কিন্তু lightweight আছে কি?
- [ ] Metrics-এ high-cardinality label (user_id-এর মতো) avoid করা হয়েছে কি?
- [ ] Distributed trace context প্রতিটা service call/message queue-তে propagate হচ্ছে কি?
- [ ] Breaking change করার আগে backward compatibility impact analyze করা হয়েছে কি?
- [ ] Deprecated endpoint-এ `Deprecation`/`Sunset` header ও usage monitoring আছে কি?
- [ ] Version bump শুধু প্রকৃত breaking change-এই হচ্ছে, প্রতিটা ছোট change-এ না?

## Key Takeaways
1. **Observability** (Logging + Monitoring + Metrics + Tracing) একসাথে কাজ করে "কী ঘটলো", "কতটা খারাপ", এবং "কোথায় ঘটলো" এই তিনটা প্রশ্নের উত্তর দিতে — শুধু একটা থাকলে পুরো ছবি পাওয়া যায় না।
2. **Versioning ও Backward Compatibility** মূলত trust-building exercise — client-রা জানে তাদের integration হঠাৎ ভেঙে যাবে না, এটাই একটা platform-কে দীর্ঘমেয়াদে reliable বানায়।
3. Production-grade system-এ "monitor করতে পারা" এবং "gracefully change করতে পারা" — এই দুটো ক্ষমতাই একজন junior আর senior engineer-এর মধ্যে সবচেয়ে বড় পার্থক্য তৈরি করে।
