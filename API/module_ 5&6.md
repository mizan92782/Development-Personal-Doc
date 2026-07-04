# 🔐 MODULE 5: API SECURITY (FAANG Level)

---

## 5.1 JWT & Token Authentication

### 1. Topic Name
JWT (JSON Web Token) ও Token-based Authentication

### 2. কেন শিখবে?
Session-based auth স্কেল করা কঠিন কারণ server-এ session state রাখতে হয়। Millions of users হলে load balancer-এর প্রতিটা node-এ session sync করা expensive। JWT stateless — এটাই কারণ Google, Amazon, Netflix সবাই token-based auth ব্যবহার করে distributed system-এ।

### 3. Problem এটা কী সমাধান করে
- Stateless authentication (server-এ session store করার দরকার নেই)
- Multiple microservices-এর মধ্যে identity share করা সহজ
- Mobile app + web app একই auth mechanism ব্যবহার করতে পারে
- Horizontal scaling সহজ হয় কারণ কোনো sticky session লাগে না

### 4. Production Failure যদি Ignore করো
- Session-based auth দিয়ে ১০টা server চালালে Redis-এ session store না করলে user login/logout inconsistent হবে
- JWT-এর expiry না রাখলে leak হওয়া token দিয়ে attacker চিরকাল access পাবে (Facebook-এর ২০১৮ access token leak এর মতো, ৫০ মিলিয়ন account affected হয়েছিল)
- Secret key rotate না করলে compromised key দিয়ে forged token বানানো যাবে

### 5. Real-world উদাহরণ
- **Netflix**: microservices-এর মধ্যে internal JWT দিয়ে service-to-service auth করে
- **Amazon**: AWS Cognito token-based auth ব্যবহার করে millions of users handle করতে
- **Uber**: driver ও rider app দুটোতেই JWT access + refresh token pattern ব্যবহার করে

### 6. API Concept Explanation
JWT-এর তিনটা অংশ: `Header.Payload.Signature`
- Header: algorithm info (HS256/RS256)
- Payload: claims (user_id, role, exp)
- Signature: tamper-proof করার জন্য secret/private key দিয়ে sign করা

Access token short-lived (৫-১৫ মিনিট), Refresh token long-lived (৭-৩০ দিন) — এই দুটো আলাদা রাখাই senior-level practice, কারণ access token leak হলেও damage window ছোট থাকে।

### 7. Django REST Framework Implementation
```python
# settings.py
INSTALLED_APPS += ['rest_framework_simplejwt']

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    )
}

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=10),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),
    'ROTATE_REFRESH_TOKENS': True,
    'BLACKLIST_AFTER_ROTATION': True,
    'ALGORITHM': 'RS256',   # HS256 না, কারণ multi-service verify করতে হলে public key দরকার
}

# urls.py
from rest_framework_simplejwt.views import TokenObtainPairView, TokenRefreshView
urlpatterns = [
    path('token/', TokenObtainPairView.as_view()),
    path('token/refresh/', TokenRefreshView.as_view()),
]
```

### 8. Spring Boot Equivalent
```java
@Component
public class JwtFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res,
                                     FilterChain chain) throws ServletException, IOException {
        String token = extractToken(req);
        if (token != null && jwtUtil.validateToken(token)) {
            Authentication auth = jwtUtil.getAuthentication(token);
            SecurityContextHolder.getContext().setAuthentication(auth);
        }
        chain.doFilter(req, res);
    }
}
```

### 9. Internal Request Lifecycle
1. Client `/login` এ credentials পাঠায়
2. Server verify করে access + refresh token generate করে
3. Client প্রতিটা request-এ `Authorization: Bearer <token>` header পাঠায়
4. Middleware/Filter token verify করে (signature + expiry check)
5. Valid হলে `request.user` populate হয়ে view/controller-এ যায়
6. Token expire হলে refresh endpoint দিয়ে নতুন access token নেয়া হয়

### 10. Architecture Diagram (Text)
```
Client --login--> Auth Service --(access+refresh token)--> Client
Client --API call + Bearer token--> API Gateway --verify signature--> Microservice
                                            |
                                    (no DB call needed - stateless)
```

### 11. Performance Impact
JWT verify করতে DB hit লাগে না (stateless) → latency কম। কিন্তু RS256 asymmetric signature verify HS256-এর চেয়ে CPU-heavy, তাই high-throughput API-তে caching/short-circuit দরকার হতে পারে।

### 12. Common Mistakes (Junior vs Senior)
- **Junior**: token-এ password বা sensitive data রাখে payload-এ (payload শুধু base64, encrypted না!)
- **Senior**: শুধু `user_id`, `role`, `exp` রাখে; sensitive data কখনো JWT payload-এ রাখে না
- **Junior**: access token-এর expiry অনেক লম্বা রাখে (১ দিন+)
- **Senior**: access token ছোট রাখে, refresh token দিয়ে rotate করে

### 13. Debugging Strategy
- jwt.io দিয়ে token decode করে claims verify করা
- Clock skew issue (server time mismatch) চেক করা `exp`/`iat` mismatch-এর জন্য
- 401 vs 403 আলাদা করে দেখা — token invalid নাকি permission নেই

### 14. Optimization Techniques
- Public key caching (RS256 verify-এর জন্য key বারবার fetch না করা)
- Token blacklist-এর জন্য Redis ব্যবহার (logout/revoke হ্যান্ডেল করতে)

### 15. Trade-offs (কখন ব্যবহার করবে না)
- যদি instant revocation দরকার হয় (banking-এর মতো) তাহলে pure stateless JWT ঝুঁকিপূর্ণ — session-based অথবা short-lived token + server-side check ভালো
- Single monolith app-এ session auth এর simplicity JWT-এর complexity-এর চেয়ে ভালো হতে পারে

### 16. Production Best Practices
- সবসময় HTTPS ব্যবহার (token sniffing ঠেকাতে)
- RS256 ব্যবহার করো multi-service architecture-এ
- Refresh token rotation + blacklist রাখো
- Token-এ minimal claims রাখো

### 17. Security Considerations
- `alg: none` attack ঠেকাতে library-তে algorithm hardcode করে verify করো
- Secret key env variable-এ রাখো, code-এ hardcode না
- Token httpOnly cookie-তে রাখা XSS থেকে বাঁচায় (localStorage-এ রাখলে XSS দিয়ে চুরি সম্ভব)

### 18. Senior Engineer Design Mindset
সিনিয়র ইঞ্জিনিয়ার ভাবে: "যদি এই token leak হয়, ড্যামেজ কতটুকু হবে এবং কতক্ষণ থাকবে?" — এই প্রশ্ন থেকেই token lifetime, revocation strategy design হয়।

### 🔹 Interview Questions

**Beginner**
1. JWT কী এবং এটার তিনটা অংশ কী কী?
2. Session-based auth আর token-based auth-এর মধ্যে পার্থক্য কী?
3. Access token আর refresh token-এর কাজ কী আলাদা?

**Intermediate**
4. JWT কীভাবে tamper-proof হয়?
5. HS256 আর RS256-এর মধ্যে পার্থক্য কী এবং কখন কোনটা ব্যবহার করবে?
6. Token expire হয়ে গেলে flow কী হয়?

**Advanced/Senior**
7. Compromised JWT কীভাবে revoke করবে যেখানে JWT stateless?
8. Microservices architecture-এ service-to-service authentication কীভাবে design করবে?
9. Token replay attack কীভাবে ঠেকাবে?

**FAANG System Design**
10. Netflix-এর মতো ১০০+ microservice-এ centralized auth কীভাবে design করবে যাতে single point of failure না হয়?
11. Billions of requests handle করা API-তে JWT verification latency কীভাবে কমাবে?

**Scenario-Based**
12. একজন user complain করছে সে logout করার পরও তার পুরনো device-এ এখনও login আছে — কেন হচ্ছে এবং কীভাবে fix করবে?

### 🧾 Interview Answers (Detailed)

**Q7: Compromised JWT কীভাবে revoke করবে?**
Bengali Explanation: যেহেতু JWT নিজেই stateless এবং server কোনো record রাখে না, তাই "revoke" করতে হলে server-side একটা lightweight blacklist/deny-list রাখতে হয় — সাধারণত Redis-এ token-এর `jti` (JWT ID) claim store করে। প্রতিটা request-এ verify করার সময় blacklist চেক করা হয় — এটা O(1) lookup, তাই stateless-এর performance benefit বেশিরভাগ বজায় থাকে।
Real-world example: Google তাদের OAuth token revocation endpoint-এ এই approach ব্যবহার করে।
When to use: high-security application (banking, admin panel)।
When NOT to use: যদি extremely high throughput আর latency-critical system হয় যেখানে Redis lookup-ও ব্যয়বহুল, তখন short access-token lifetime (২-৫ মিনিট) দিয়ে natural expiry-র উপর নির্ভর করা ভালো।
Trade-offs: blacklist রাখলে pure statelessness নষ্ট হয়, কিন্তু security বাড়ে।
Common mistake: candidates বলে "JWT revoke করা যায় না" — এটা ভুল, ঠিক উত্তর হলো blacklist/deny-list pattern।
Senior-level answer structure: Problem → কেন pure JWT revoke কঠিন → solution (blacklist + short expiry + refresh rotation) → trade-off বলা।

---

## 5.2 RBAC (Role-Based Access Control)

### 1-2. Topic ও কেন শিখবে
RBAC হলো user-কে role assign করে সেই role অনুযায়ী permission দেয়ার system। এটা ছাড়া প্রতিটা user-এর জন্য আলাদা permission check করতে হতো — যা মেইনটেইন করা অসম্ভব বড় system-এ।

### 3-4. Problem ও Production Failure
Problem: "কে কী করতে পারবে" এটা centralized ভাবে manage করা।
Failure: RBAC না থাকলে permission logic ছড়িয়ে যায় পুরো codebase-এ (if-else hell), এবং একদিন কোনো developer ভুল করে admin-only endpoint সবার জন্য খুলে দেয় — এটাই privilege escalation vulnerability-র সবচেয়ে কমন কারণ।

### 5. Real-world উদাহরণ
- **AWS IAM**: policy-based RBAC-এর extreme example
- **Google Workspace**: admin, editor, viewer role
- **GitHub**: repo-তে read/write/admin role

### 6. API Concept
User → Role → Permission mapping। একজন user একাধিক role পেতে পারে, একটা role একাধিক permission ধারণ করতে পারে (many-to-many)।

### 7. DRF Implementation
```python
class IsAdminOrReadOnly(permissions.BasePermission):
    def has_permission(self, request, view):
        if request.method in permissions.SAFE_METHODS:
            return True
        return request.user.groups.filter(name='Admin').exists()

class OrderViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated, IsAdminOrReadOnly]
```

### 8. Spring Boot Equivalent
```java
@PreAuthorize("hasRole('ADMIN')")
@DeleteMapping("/orders/{id}")
public ResponseEntity<?> deleteOrder(@PathVariable Long id) { ... }
```

### 9. Internal Lifecycle
Request → Authentication filter (কে?) → Authorization check (role/permission আছে কি?) → View/Controller execute।

### 10. Architecture Diagram
```
User --> [Role: Editor] --> [Permissions: read, write] --> Resource Access Check --> Allow/Deny
```

### 11. Performance Impact
Permission check প্রতিটা request-এ হয় বলে DB query cache না করলে N+1 problem তৈরি হয় (user-এর role/permission বারবার query করা)।

### 12. Common Mistakes
Junior: frontend-এ role check করে backend-এ করে না (attacker সরাসরি API call করে bypass করে)।
Senior: backend-এ সবসময় enforce করে, frontend শুধু UX-এর জন্য hide করে।

### 13-14. Debugging ও Optimization
Debug: 403 error এলে user-এর role + required permission log করে compare করা। Optimization: role/permission cache করা (Redis, ৫-১০ মিনিট TTL)।

### 15. Trade-offs
Simple app-এ শুধু "is_admin" boolean flag যথেষ্ট — full RBAC over-engineering হতে পারে ছোট system-এ।

### 16-18. Best Practices, Security, Mindset
Principle of least privilege মেনে চলা — default deny, explicit allow। Senior engineer সবসময় ভাবে "এই role compromise হলে blast radius কত বড়?"

### 🔹 Interview Questions
**Beginner**: RBAC কী? Role আর Permission-এর পার্থক্য কী? Group-based permission কীভাবে কাজ করে?
**Intermediate**: RBAC vs ABAC (Attribute-Based) পার্থক্য কী? Multi-tenant system-এ RBAC কীভাবে design করবে?
**Senior**: Dynamic permission (runtime-এ পরিবর্তনযোগ্য) কীভাবে design করবে বড় scale-এ caching সহ?
**FAANG System Design**: Google Drive-এর মতো file-sharing system-এ folder-level ও file-level permission inheritance কীভাবে design করবে?
**Scenario**: একজন employee role change হওয়ার পরও পুরনো permission দিয়ে access পাচ্ছে — root cause কী হতে পারে? (উত্তর: caching stale data)

---

## 5.3 Rate Limiting

### 1-2. Topic ও কেন শিখবে
Rate limiting API-কে abuse, DDoS, এবং resource exhaustion থেকে রক্ষা করে। এক client যদি সেকেন্ডে হাজার request পাঠায়, বাকি সবার জন্য service down হয়ে যেতে পারে।

### 3-4. Problem ও Failure
Failure example: Twitter/X-এর মতো platform যদি rate limit না রাখতো, একটা bot script পুরো API infrastructure overload করে দিতে পারতো — ২০১২ সালে অনেক startup precisely এই কারণে outage-এ পড়েছিল।

### 5. Real-world উদাহরণ
- GitHub API: ৫০০০ requests/hour per authenticated user
- Twitter API: strict rate limit tier অনুযায়ী
- Stripe API: burst + sustained rate limit combo

### 6. API Concept — Algorithms
- **Token Bucket**: bucket-এ token জমা হয় fixed rate-এ, request এলে token খরচ হয়, বাকি না থাকলে reject
- **Leaky Bucket**: fixed rate-এ request process হয়, queue overflow হলে drop
- **Fixed Window**: প্রতি মিনিটে N request, কিন্তু window boundary-তে burst সমস্যা হতে পারে
- **Sliding Window Log**: প্রতিটা request timestamp রেখে accurate কিন্তু memory-heavy

### 7. DRF Implementation
```python
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.UserRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'user': '100/minute',
        'anon': '20/minute',
    }
}

class BurstRateThrottle(UserRateThrottle):
    scope = 'burst'
    rate = '10/second'
```

### 8. Spring Boot Equivalent (Bucket4j দিয়ে)
```java
Bucket bucket = Bucket.builder()
    .addLimit(Bandwidth.classic(100, Refill.greedy(100, Duration.ofMinutes(1))))
    .build();

if (bucket.tryConsume(1)) {
    // process request
} else {
    return ResponseEntity.status(429).body("Too Many Requests");
}
```

### 9. Internal Lifecycle
Request আসে → API Gateway/Middleware token bucket চেক করে → token available হলে proceed, না হলে `429 Too Many Requests` return করে।

### 10. Architecture Diagram
```
Client --> API Gateway [Rate Limiter: Redis-backed counter] --> 429 or Forward to Service
```

### 11. Performance Impact
Rate limiter নিজেই যদি slow হয় (প্রতি request-এ DB hit) তাহলে এটা bottleneck হয়ে যায়। তাই in-memory (Redis) counter ব্যবহার করা হয়।

### 12. Common Mistakes
Junior: single server-এ in-memory counter রাখে — multiple server-এ scale করলে limit bypass হয়ে যায় (round-robin করে limit avoid করা যায়)।
Senior: distributed rate limiting করে centralized Redis দিয়ে।

### 13-14. Debugging ও Optimization
429 response-এ `Retry-After` header দেয়া উচিত যাতে client জানে কতক্ষণ wait করতে হবে। Optimization: Redis Lua script দিয়ে atomic increment+check করা (race condition এড়াতে)।

### 15. Trade-offs
খুব strict rate limit legitimate user-কে block করে দিতে পারে; খুব loose limit abuse ঠেকায় না — এটা balance করা দরকার, user tier অনুযায়ী different limit রাখা ভালো।

### 16-18. Best Practices, Security, Mindset
Per-IP এবং per-user দুই লেভেলেই limit রাখা ভালো। Senior mindset: "attacker যদি ১০০০টা fake account বানায়, তখন কী হবে?" — এই প্রশ্ন থেকে IP-based + account-based rate limiting দুটোই দরকার হয়।

### 🔹 Interview Questions
**Beginner**: Rate limiting কী এবং কেন দরকার? Token bucket algorithm কীভাবে কাজ করে?
**Intermediate**: Token bucket vs leaky bucket পার্থক্য কী? Distributed system-এ rate limiting কীভাবে করবে?
**Senior**: Multi-region deployment-এ globally consistent rate limit কীভাবে design করবে (network latency সহ)?
**FAANG System Design**: Design a rate limiter for an API Gateway handling millions of requests/sec across multiple data centers।
**Scenario**: একটা client complain করছে মাঝে মাঝে সঠিকভাবে rate limit না হয়ে বেশি request pass হয়ে যাচ্ছে — কারণ কী হতে পারে? (উত্তর: race condition, non-atomic counter increment)

---

## 5.4 SQL Injection Prevention

### 1-2. Topic ও কেন শিখবে
SQL Injection এখনো OWASP Top 10-এর মধ্যে থাকে কারণ একটা ভুল raw query পুরো database leak/delete করে দিতে পারে।

### 3-4. Problem ও Failure
বিখ্যাত উদাহরণ: 2017 সালে Equifax breach আংশিকভাবে improper input validation-এর কারণে হয়েছিল, ১৪৭ মিলিয়ন মানুষের ডেটা leak হয়েছিল।

### 5. Real-world উদাহরণ
বড় কোম্পানি সবসময় ORM (Django ORM, Hibernate) ব্যবহার করে raw SQL এড়িয়ে চলে, বিশেষ করে user input directly query-তে না বসিয়ে।

### 6. API Concept
Attacker input-এ SQL syntax inject করে (`' OR '1'='1`) query-র logic পরিবর্তন করে দেয়। Parameterized query/prepared statement এটা ঠেকায় কারণ input কখনো code হিসেবে treat হয় না, শুধু data হিসেবে হয়।

### 7. DRF Implementation
```python
# ভুল (vulnerable):
User.objects.raw(f"SELECT * FROM users WHERE username = '{username}'")

# সঠিক (Django ORM auto-parameterize করে):
User.objects.filter(username=username)

# raw SQL লাগলেও parameterized:
User.objects.raw("SELECT * FROM users WHERE username = %s", [username])
```

### 8. Spring Boot Equivalent
```java
// ভুল:
String query = "SELECT * FROM users WHERE username = '" + username + "'";

// সঠিক (PreparedStatement/JPA):
@Query("SELECT u FROM User u WHERE u.username = :username")
User findByUsername(@Param("username") String username);
```

### 9-10. Lifecycle ও Diagram
Request → Input validation layer → ORM/Parameterized query → DB। Raw string concatenation কখনো database-এ যাওয়া উচিত না।

### 11. Performance Impact
Parameterized query prepared statement caching-এর সুবিধা দেয়, তাই security-এর পাশাপাশি performance-ও ভালো হয়।

### 12. Common Mistakes
Junior string formatting/concatenation দিয়ে query বানায়। Senior সবসময় ORM/parameterized query ব্যবহার করে, raw SQL এড়ায়।

### 13-18. Debugging, Optimization, Trade-off, Best Practices
Debugging: query logs review করে string concatenation খোঁজা। Best practice: input sanitization + WAF (Web Application Firewall) + least-privilege DB user (application-এর DB user-এর DROP TABLE permission থাকা উচিত না)।

### 🔹 Interview Questions
**Beginner**: SQL Injection কী? এটা কীভাবে prevent করা যায়?
**Intermediate**: ORM কীভাবে SQL Injection prevent করে internally?
**Senior**: Legacy system-এ raw SQL থেকে migrate না করেও কীভাবে SQLi risk কমাবে?
**FAANG**: একটা large-scale system-এ automated security scanning pipeline কীভাবে design করবে SQLi detect করতে?
**Scenario**: Code review-এ f-string দিয়ে বানানো query দেখলে কী বলবে এবং কীভাবে fix suggest করবে?

---

## 5.5 CSRF / XSS Protection

### 1-2. Topic ও কেন শিখবে
CSRF (Cross-Site Request Forgery) ও XSS (Cross-Site Scripting) web application-এর সবচেয়ে কমন vulnerability, বিশেষ করে যেসব API browser session/cookie ব্যবহার করে।

### 3-4. Problem ও Failure
CSRF: attacker user-কে না জানিয়ে তার logged-in session ব্যবহার করে action করিয়ে নেয় (যেমন fund transfer)।
XSS: attacker malicious script inject করে অন্য user-এর browser-এ চালায়, session cookie চুরি করে।

### 5. Real-world উদাহরণ
2010 সালের বিখ্যাত Samy worm (MySpace) XSS দিয়ে ১ মিলিয়নের বেশি profile infect করেছিল।

### 6. API Concept
CSRF token: প্রতিটা state-changing request-এ একটা unpredictable token পাঠাতে হয় যা attacker জানে না।
XSS prevention: output encoding + Content Security Policy (CSP) + input sanitization।

### 7. DRF Implementation
```python
# Django built-in CSRF middleware (session-based auth-এর জন্য)
MIDDLEWARE = ['django.middleware.csrf.CsrfViewMiddleware']

# JWT/token-based API-তে সাধারণত CSRF দরকার হয় না কারণ cookie ব্যবহার হয় না,
# কিন্তু cookie-based JWT storage করলে CSRF token লাগবেই

# XSS: DRF response auto-escape করে না serializer output,
# তাই output রেন্ডার করার সময় frontend-এ escape করতে হয়
```

### 8. Spring Boot Equivalent
```java
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.csrf(csrf -> csrf.csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse()));
        return http.build();
    }
}
```

### 9-10. Lifecycle ও Diagram
```
Form submit --> CSRF token verify (server-side) --> Allow/Reject
User input --> Output encoding/Sanitization --> Render in browser (safe)
```

### 11. Performance Impact
Minimal — token verify করা lightweight operation, কিন্তু CSP header misconfigure করলে legitimate resource block হয়ে যেতে পারে।

### 12. Common Mistakes
Junior: user input সরাসরি HTML-এ render করে (`dangerouslySetInnerHTML` React-এ ভুলভাবে ব্যবহার) — এটাই XSS-এর সবচেয়ে common source।
Senior: সবসময় output encode/sanitize করে, trusted library (DOMPurify) ব্যবহার করে।

### 13-18. Debugging, Optimization, Best Practices, Security
Best practice: httpOnly + Secure + SameSite cookie attribute ব্যবহার করা, CSP header strict রাখা, input validation + output encoding দুটোই করা (একটা যথেষ্ট না)।

### 🔹 Interview Questions
**Beginner**: CSRF ও XSS-এর পার্থক্য কী?
**Intermediate**: SameSite cookie attribute কীভাবে CSRF prevent করতে সাহায্য করে?
**Senior**: SPA (Single Page Application) + REST API architecture-এ CSRF protection কীভাবে design করবে যেখানে token cookie-তে থাকে?
**FAANG**: Facebook-scale social media platform-এ user-generated content (comments, posts) থেকে XSS কীভাবে prevent করবে?
**Scenario**: একটা comment section-এ user `<script>` tag পোস্ট করলে অন্য user-এর browser-এ execute হয়ে যাচ্ছে — root cause এবং fix কী?

---

# ⚙️ MODULE 6: SCALING APIs (FAANG Level)

---

## 6.1 Load Balancing

### 1-2. Topic ও কেন শিখবে
একটা single server কখনোই millions of concurrent user handle করতে পারে না। Load balancer ট্রাফিক multiple server-এ ভাগ করে দেয়, যা horizontal scaling-এর ভিত্তি।

### 3-4. Problem ও Failure
Load balancer ছাড়া single server crash হলে পুরো service down হয়ে যায় (single point of failure)। Uneven distribution হলে একটা server overload হয়ে বাকিগুলো idle থাকে।

### 5. Real-world উদাহরণ
- Netflix তাদের নিজস্ব **Zuul** (পরে **Eureka + Ribbon**) ব্যবহার করে client-side load balancing-এর জন্য
- Amazon **ELB (Elastic Load Balancer)** ব্যবহার করে
- Google **Maglev** নামের custom load balancer বানিয়েছে internal scale-এর জন্য

### 6. API Concept — Algorithms
- **Round Robin**: পালাক্রমে প্রতিটা server-এ request পাঠানো
- **Least Connections**: যে server-এ সবচেয়ে কম active connection সেখানে পাঠানো
- **IP Hash**: client IP অনুযায়ী নির্দিষ্ট server-এ পাঠানো (sticky session-এর জন্য)
- **Weighted Round Robin**: শক্তিশালী server-কে বেশি ট্রাফিক দেয়া

### 7. DRF Context
DRF নিজে load balancer না, কিন্তু Gunicorn/uWSGI worker process + Nginx reverse proxy দিয়ে setup করা হয়:
```nginx
upstream django_app {
    least_conn;
    server 10.0.0.1:8000;
    server 10.0.0.2:8000;
    server 10.0.0.3:8000;
}
server {
    location / {
        proxy_pass http://django_app;
    }
}
```

### 8. Spring Boot Equivalent
Spring Cloud LoadBalancer ব্যবহার করে client-side load balancing:
```java
@LoadBalanced
@Bean
public RestTemplate restTemplate() {
    return new RestTemplate();
}
```

### 9. Internal Lifecycle
Client request → DNS → Load Balancer → Health check pass করা servers-এর মধ্য থেকে algorithm অনুযায়ী একটা select → forward।

### 10. Architecture Diagram
```
                 ┌──────────────┐
Client --------> │ Load Balancer│
                 └──────┬───────┘
          ┌─────────────┼─────────────┐
     ┌────▼───┐    ┌────▼───┐    ┌────▼───┐
     │Server 1│    │Server 2│    │Server 3│
     └────────┘    └────────┘    └────────┘
```

### 11. Performance Impact
সঠিক algorithm choice latency অনেক কমায়। Least Connections heavy/slow request-এর জন্য ভালো, কিন্তু Round Robin simple ও lightweight।

### 12. Common Mistakes
Junior: health check ছাড়া load balancer setup করে — dead server-এ ট্রাফিক পাঠাতে থাকে।
Senior: active health check + graceful degradation implement করে (unhealthy server auto-remove হয়)।

### 13-14. Debugging ও Optimization
Debug: uneven traffic distribution দেখলে algorithm ও health check config চেক করা। Optimization: connection pooling + keep-alive ব্যবহার করা।

### 15. Trade-offs
Sticky session (IP Hash) session affinity দেয় কিন্তু uneven load তৈরি করতে পারে। Stateless design সবসময় better scaling দেয়।

### 16-18. Best Practices, Security, Mindset
Multi-AZ (Availability Zone) deployment করা যাতে একটা data center down হলেও service চলে। Senior mindset: "যদি একটা পুরো region down হয়ে যায়, তাহলে কী হবে?"

### 🔹 Interview Questions
**Beginner**: Load balancer কী এবং কেন দরকার?
**Intermediate**: Round Robin vs Least Connections কখন কোনটা ব্যবহার করবে?
**Senior**: Layer 4 vs Layer 7 load balancing-এর পার্থক্য এবং trade-off কী?
**FAANG**: YouTube-scale video streaming platform-এর জন্য global load balancing কীভাবে design করবে (geo-routing সহ)?
**Scenario**: একটা server বাকিগুলোর চেয়ে ৫ গুণ বেশি ট্রাফিক পাচ্ছে — root cause কী হতে পারে এবং কীভাবে debug করবে?

---

## 6.2 Microservices Architecture

### 1-2. Topic ও কেন শিখবে
Monolith বড় হয়ে গেলে deploy, scale, এবং maintain করা কঠিন হয়ে যায়। Microservices আলাদা আলাদা service independently scale/deploy করার সুযোগ দেয়।

### 3-4. Problem ও Failure
Monolith failure: একটা ছোট bug পুরো application crash করাতে পারে, এবং একটা feature deploy করতে পুরো app redeploy করতে হয়।
Microservices না করলে Amazon-এর মতো কোম্পানির পক্ষে হাজারো team-কে independently ship করা সম্ভব হতো না।

### 5. Real-world উদাহরণ
Amazon-এর প্রতিটা product page শত শত microservice call করে (recommendation, pricing, inventory, review — সব আলাদা service)। Netflix ৭০০+ microservice চালায়।

### 6. API Concept
প্রতিটা microservice নিজস্ব database, নিজস্ব deployment lifecycle রাখে। Services একে অপরের সাথে REST/gRPC/message queue দিয়ে communicate করে।

### 7. DRF Context
Django-তে microservice বানানোর সময় প্রতিটা app আলাদা project হিসেবে থাকে, shared database এড়িয়ে চলা হয়:
```python
# Order service শুধু order-related data রাখে,
# user info দরকার হলে User Service-কে API call করে, direct DB join করে না
```

### 8. Spring Boot Equivalent
Spring Boot microservices-এর জন্য সবচেয়ে জনপ্রিয় framework — Spring Cloud দিয়ে service discovery (Eureka), config server, API gateway সহজেই বানানো যায়।

### 9. Internal Lifecycle
Client → API Gateway → Service Discovery (কোন service কোথায় আছে) → নির্দিষ্ট microservice → response aggregate → client।

### 10. Architecture Diagram
```
Client --> API Gateway --> [Auth Service]
                        --> [Order Service] --> Order DB
                        --> [Payment Service] --> Payment DB
                        --> [Notification Service] --> Message Queue
```

### 11. Performance Impact
Network call overhead বাড়ে (in-process call vs network call)। Distributed tracing ছাড়া latency debug করা কঠিন হয়ে যায়।

### 12. Common Mistakes
Junior: microservice বানায় কিন্তু shared database ব্যবহার করে (এটা "distributed monolith" — সবচেয়ে খারাপ combination)।
Senior: প্রতিটা service-এর own database, event-driven communication (Kafka/SQS) ব্যবহার করে tight coupling এড়ায়।

### 13-14. Debugging ও Optimization
Distributed tracing (Jaeger, Zipkin) essential যখন একটা request ১০টা service-এ যায়। Optimization: async communication যেখানে real-time response দরকার নেই।

### 15. Trade-offs
ছোট team/startup-এর জন্য microservices premature optimization হতে পারে — operational complexity (deployment, monitoring, debugging) অনেক বেড়ে যায়। "Monolith first" approach বেশিরভাগ ক্ষেত্রে ভালো শুরু।

### 16-18. Best Practices, Security, Mindset
Service boundary business domain অনুযায়ী design করা (Domain-Driven Design)। Senior mindset: "এই দুইটা service কি সবসময় একসাথে deploy হয়? তাহলে কি এদের আসলেই আলাদা হওয়া উচিত?"

### 🔹 Interview Questions
**Beginner**: Microservices কী এবং monolith থেকে কীভাবে আলাদা?
**Intermediate**: Microservices-এ data consistency কীভাবে maintain করে (distributed transaction)?
**Senior**: Saga pattern কী এবং কখন 2PC (Two-Phase Commit)-এর বদলে এটা ব্যবহার করবে?
**FAANG**: Amazon-এর order processing system design করো যেখানে inventory, payment, shipping আলাদা service।
**Scenario**: একটা checkout flow-তে payment success হলো কিন্তু order confirmation fail করলো — কীভাবে handle করবে যাতে data inconsistent না থাকে?

---

## 6.3 API Gateway

### 1-2. Topic ও কেন শিখবে
Microservices architecture-এ client-কে সরাসরি প্রতিটা service-এর সাথে connect করানো complex ও insecure। API Gateway single entry point হিসেবে কাজ করে।

### 3-4. Problem ও Failure
Gateway ছাড়া client-কে ২০টা microservice-এর endpoint জানতে হবে, প্রতিটাতে আলাদা auth করতে হবে — যা frontend team-এর জন্য악악악 nightmare।

### 5. Real-world উদাহরণ
- Netflix **Zuul**
- Amazon **API Gateway** (AWS service)
- Kong, Apigee — enterprise API gateway solutions

### 6. API Concept
Gateway centralized ভাবে করে: authentication, rate limiting, request routing, response aggregation, logging, caching।

### 7-8. Implementation Concept
```yaml
# Spring Cloud Gateway config example
routes:
  - id: order-service
    uri: lb://ORDER-SERVICE
    predicates:
      - Path=/api/orders/**
    filters:
      - AuthenticationFilter
      - RateLimiter
```
DRF-তে সাধারণত Django নিজে gateway হয় না — Kong/Nginx/AWS API Gateway সামনে বসিয়ে ব্যবহার করা হয়।

### 9. Internal Lifecycle
Client → API Gateway (auth check → rate limit check → route resolve) → appropriate microservice → response → gateway (optional transform) → client।

### 10. Architecture Diagram
```
Mobile/Web Client --> API Gateway --> [Auth, Rate Limit, Routing, Caching]
                                   --> Service A / Service B / Service C
```

### 11. Performance Impact
Extra network hop latency যোগ করে, কিন্তু caching ও connection pooling দিয়ে সেটা compensate করা যায়।

### 12. Common Mistakes
Junior: gateway-কে business logic দিয়ে ভারী করে ফেলে (gateway শুধু routing/cross-cutting concern-এর জন্য, business logic না)।
Senior: gateway lightweight রাখে, শুধু cross-cutting concerns handle করে।

### 13-18. Debugging, Optimization, Trade-offs, Best Practices
Single point of failure এড়াতে gateway multiple instance-এ deploy করে load balancer-এর পেছনে রাখা হয়। Best practice: gateway-তে circuit breaker + timeout config রাখা downstream failure isolate করতে।

### 🔹 Interview Questions
**Beginner**: API Gateway কী এবং এর মূল দায়িত্ব কী?
**Intermediate**: API Gateway ও Load Balancer-এর মধ্যে পার্থক্য কী?
**Senior**: API Gateway নিজেই bottleneck/single point of failure হয়ে গেলে কীভাবে সমাধান করবে?
**FAANG**: একটা multi-tenant SaaS platform-এর জন্য API Gateway design করো যেখানে প্রতিটা tenant-এর আলাদা rate limit ও routing rule আছে।
**Scenario**: Gateway downtime-এর কারণে পুরো platform down হয়ে গেছে — এই architecture-এর দুর্বলতা কী এবং কীভাবে fix করবে?

---

## 6.4 Circuit Breaker

### 1-2. Topic ও কেন শিখবে
Distributed system-এ একটা service slow/down হয়ে গেলে সেটা cascading failure তৈরি করতে পারে — পুরো system down হয়ে যেতে পারে একটা service-এর কারণে। Circuit breaker এটা প্রতিরোধ করে।

### 3-4. Problem ও Failure
2015 সালে Amazon-এর একটা internal service failure cascading effect তৈরি করে বড় outage ঘটিয়েছিল কারণ downstream service-গুলো retry করতে করতে resource exhaust করে ফেলেছিল।

### 5. Real-world উদাহরণ
Netflix-এর **Hystrix** library circuit breaker pattern-এর সবচেয়ে বিখ্যাত implementation (এখন Resilience4j জনপ্রিয়)।

### 6. API Concept — States
- **Closed**: normal operation, request pass হয়
- **Open**: failure threshold পার হলে, সব request সাথে সাথে reject হয় (downstream-কে সময় দেয়া recover করার জন্য)
- **Half-Open**: কিছুক্ষণ পর একটা test request পাঠানো হয় — success হলে Closed, fail হলে আবার Open

### 7. DRF Implementation
```python
from pybreaker import CircuitBreaker

payment_breaker = CircuitBreaker(fail_max=5, reset_timeout=30)

@payment_breaker
def call_payment_service(data):
    return requests.post('http://payment-service/charge', json=data, timeout=3)
```

### 8. Spring Boot Equivalent (Resilience4j)
```java
@CircuitBreaker(name = "paymentService", fallbackMethod = "fallbackPayment")
public PaymentResponse processPayment(PaymentRequest req) {
    return paymentClient.charge(req);
}

public PaymentResponse fallbackPayment(PaymentRequest req, Throwable t) {
    return PaymentResponse.pending("Payment service unavailable, queued for retry");
}
```

### 9. Internal Lifecycle
Call downstream service → success/fail track করে → failure threshold পার হলে circuit Open → পরের request-গুলো সরাসরি fallback response পায় (downstream call না করেই) → timeout পর Half-Open হয়ে test করে।

### 10. Architecture Diagram
```
Service A --> [Circuit Breaker: CLOSED] --> Service B (healthy)
Service A --> [Circuit Breaker: OPEN] --> Fallback Response (Service B down, fail fast)
```

### 11. Performance Impact
Circuit Open অবস্থায় downstream call skip করে দেয় বলে latency drastically কমে যায় (fail fast) এবং resource (thread, connection) বাঁচে।

### 12. Common Mistakes
Junior: infinite retry করে downstream service-এ যখন সেটা already struggling — এটা "retry storm" তৈরি করে situation আরও খারাপ করে।
Senior: circuit breaker + exponential backoff combine করে ব্যবহার করে।

### 13-18. Debugging, Optimization, Trade-offs, Best Practices
Threshold খুব sensitive রাখলে temporary blip-এও circuit open হয়ে যাবে (false positive)। Best practice: fallback response meaningful রাখা (empty না, cached/default data দেয়া)।

### 🔹 Interview Questions
**Beginner**: Circuit Breaker pattern কী এবং কেন দরকার?
**Intermediate**: Circuit breaker-এর তিনটা state ব্যাখ্যা করো।
**Senior**: Circuit breaker ও retry mechanism একসাথে কীভাবে design করবে যাতে retry storm না হয়?
**FAANG**: একটা payment system design করো যেখানে third-party payment gateway মাঝে মাঝে slow/down হয়, কিন্তু checkout flow যেন গ্রেসফুলভাবে degrade করে।
**Scenario**: একটা downstream service slow হয়ে যাওয়ায় upstream service-এর সব thread block হয়ে গেছে — কীভাবে এটা prevent করবে?

---

## 6.5 Retry Strategies

### 1-2. Topic ও কেন শিখবে
Network call মাঝে মাঝে transient failure-এর কারণে fail করে (temporary network glitch)। Retry সঠিকভাবে করলে reliability বাড়ে, ভুলভাবে করলে system-কে আরও খারাপ অবস্থায় ফেলে দেয়।

### 3-4. Problem ও Failure
Naive retry (immediate, unlimited) downstream service overload করে দেয় ঠিক যখন সেটা recover করার চেষ্টা করছে — এটাকে "retry storm" বলে, এবং এটাই বহু বড় outage-এর কারণ হয়েছে (AWS S3 2017 outage-এর মতো ঘটনায় recovery slow হওয়ার একটা কারণ ছিল এই ধরনের cascading load)।

### 5. Real-world উদাহরণ
AWS SDK-তে built-in exponential backoff + jitter থাকে সব API call-এর জন্য।

### 6. API Concept
- **Fixed Delay Retry**: প্রতিবার একই সময় wait করে retry
- **Exponential Backoff**: প্রতিবার wait time দ্বিগুণ করে (1s, 2s, 4s, 8s...)
- **Exponential Backoff + Jitter**: randomness যোগ করে যাতে সব client একসাথে retry না করে (thundering herd এড়াতে)

### 7. DRF Implementation
```python
import time, random

def call_with_retry(func, max_retries=4, base_delay=1):
    for attempt in range(max_retries):
        try:
            return func()
        except TransientError:
            if attempt == max_retries - 1:
                raise
            delay = base_delay * (2 ** attempt) + random.uniform(0, 1)
            time.sleep(delay)
```

### 8. Spring Boot Equivalent
```java
@Retryable(
    value = {TransientException.class},
    maxAttempts = 4,
    backoff = @Backoff(delay = 1000, multiplier = 2, random = true)
)
public Response callExternalService() { ... }
```

### 9. Internal Lifecycle
Call fail হয় → error transient কিনা check করে (৫xx, timeout — retry করা উচিত; ৪xx validation error — retry করা উচিত না) → backoff calculate করে wait → retry → max attempts পার হলে fail/fallback।

### 10. Architecture Diagram
```
Request --fail--> Wait 1s --> Retry --fail--> Wait 2s --> Retry --fail--> Wait 4s --> Retry --success/fail
```

### 11. Performance Impact
সঠিক retry latency-কে predictable রাখে; ভুল retry (immediate, unlimited) latency spike ও downstream overload তৈরি করে।

### 12. Common Mistakes
Junior: সব ধরনের error-এ retry করে, এমনকি ৪০০ (bad request)-এও — যা কখনো succeed করবে না।
Senior: শুধু transient/idempotent-safe error-এ retry করে, retryable vs non-retryable error আলাদা করে।

### 13-14. Debugging ও Optimization
Debug: retry count ও delay log করে pattern বোঝা। Optimization: idempotency key ব্যবহার করা যাতে duplicate retry duplicate side-effect (যেমন double payment) তৈরি না করে।

### 15. Trade-offs
Retry latency বাড়ায় user-facing request-এ (user wait করছে) — সেখানে retry limit কম রাখা উচিত, background job-এ বেশি retry করা যায়।

### 16-18. Best Practices, Security, Mindset
Retry + Circuit Breaker + Timeout — এই তিনটা একসাথে ব্যবহার করাই senior-level resilience pattern। Senior mindset: "এই operation কি idempotent? না হলে retry করলে duplicate effect হবে কিনা?"

### 🔹 Interview Questions
**Beginner**: Retry strategy কী এবং কেন দরকার?
**Intermediate**: Exponential backoff কীভাবে কাজ করে এবং জিটার (jitter) কেন যোগ করা হয়?
**Senior**: কোন ধরনের error retry করা উচিত এবং কোনটা করা উচিত না — কীভাবে সিদ্ধান্ত নেবে?
**FAANG**: একটা distributed payment system design করো যেখানে retry duplicate charge তৈরি না করে (idempotency)।
**Scenario**: একটা batch job ১০০০ external API call করে, ৫% call transient error দেয় — retry strategy কীভাবে design করবে যাতে external service overload না হয়?

---

# 🧾 MODULE 5 & 6 — CHEAT SHEET

| Topic | মূল Concept | Key Tool/Library |
|---|---|---|
| JWT Auth | Stateless token, access+refresh split | SimpleJWT (DRF), Spring Security JWT |
| RBAC | Role → Permission mapping, least privilege | Django Groups/Permissions, Spring `@PreAuthorize` |
| Rate Limiting | Token bucket, per-user/IP limit | DRF Throttling, Bucket4j |
| SQL Injection Prevention | Parameterized query, ORM | Django ORM, JPA/Hibernate |
| CSRF/XSS | Token verification, output encoding | Django CSRF middleware, Spring Security, DOMPurify |
| Load Balancing | Round Robin, Least Connections | Nginx, AWS ELB, Spring Cloud LoadBalancer |
| Microservices | Independent deploy, own DB per service | Spring Cloud, Django multi-project |
| API Gateway | Single entry point, cross-cutting concerns | Kong, AWS API Gateway, Spring Cloud Gateway |
| Circuit Breaker | Closed/Open/Half-Open states, fail fast | Resilience4j, pybreaker |
| Retry Strategy | Exponential backoff + jitter, idempotency | Spring `@Retryable`, custom backoff logic |

## Senior Engineer Checklist ✅
- [ ] প্রতিটা endpoint-এ authentication + authorization দুটোই enforce করা হয়েছে কি?
- [ ] Sensitive data JWT payload-এ নেই তো?
- [ ] Rate limiting per-user এবং per-IP দুই লেভেলেই আছে কি?
- [ ] সব query parameterized/ORM-based, raw string concatenation নেই তো?
- [ ] Downstream service call-এ timeout + circuit breaker + retry (backoff সহ) আছে কি?
- [ ] Retry logic শুধু idempotent operation-এ প্রয়োগ হচ্ছে কি?
- [ ] Single point of failure (gateway, load balancer) redundant ভাবে deploy করা হয়েছে কি?
- [ ] প্রতিটা microservice-এর নিজস্ব database, shared DB নেই তো?

## Key Takeaways
1. Security ও Scaling একে অপরের সাথে সম্পর্কিত — একটা দুর্বল হলে অন্যটাও ঝুঁকিতে পড়ে (যেমন: rate limiting না থাকলে DDoS attack scaling-কেও break করে দেয়)।
2. Production-এ "happy path" কাজ করা যথেষ্ট না — failure scenario (downstream down, token leak, traffic spike) হ্যান্ডেল করাই সিনিয়র ইঞ্জিনিয়ারের কাজ।
3. প্রতিটা pattern-এর trade-off আছে — blindly সব pattern apply করা over-engineering, কিছুই না করা under-engineering। সঠিক scale ও context বুঝে decision নেয়াই আসল skill।
