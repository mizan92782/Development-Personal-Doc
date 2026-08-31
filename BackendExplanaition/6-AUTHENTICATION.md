# LEVEL 6 — Authentication & Security (Crystal Clear Concept)

This is the level where mistakes don't just slow your app down — they leak user data, get accounts hijacked, or make headlines. This guide covers **every major authentication mechanism**, how each one actually works step-by-step, why it exists, when to use it over the others, and how they compare — plus password security, JWTs, OAuth2, and access control (RBAC).

---

## 1. Authentication vs Authorization (the foundation)

```
Authentication
      ↓
"Who are you?"
(proving identity — login, token, biometrics)

Authorization
      ↓
"What are you allowed to do?"
(checking permissions — can you delete THIS post?)
```

```
                 ┌─────────────────┐
   Request  ───► │  Authentication  │  → fails → 401 Unauthorized
                 │  "Who is this?"  │
                 └────────┬─────────┘
                          │ identity confirmed
                          ▼
                 ┌─────────────────┐
                 │  Authorization   │  → fails → 403 Forbidden
                 │ "Allowed to do   │
                 │  THIS action?"   │
                 └────────┬─────────┘
                          │ permission granted
                          ▼
                     Request proceeds
```

Every authentication *mechanism* below answers the first question differently — but they all feed into the same authorization step afterward.

---

## 2. Password Security — Never Store Raw Passwords

```
❌ NEVER DO THIS
password ──────────────────► database
   "mypassword123"              "mypassword123"   ← if the DB leaks,
                                                       every account is
                                                       compromised instantly
```

```
✅ ALWAYS DO THIS
password
   ↓
  hash
   ↓
database
   "mypassword123"  →  hash()  →  "$2b$12$KIXQ...9fH2"  →  stored
```

### Hashing

A **hash function** turns input into a fixed-length, irreversible string. You can't "un-hash" it back into the original password — you can only hash a *guess* and compare.

```
"mypassword123"  → SHA/bcrypt/argon2 →  a1b2c3d4e5f6...  (one-way, irreversible)

Same input → always same output:
"mypassword123" → always → a1b2c3d4e5f6...

Tiny change → totally different output:
"mypassword124" → always → 9f8e7d6c5b4a...   (avalanche effect)
```

**Why plain hashing alone isn't enough:** attackers precompute **rainbow tables** — huge lookup tables mapping common passwords to their hash values. If two users both use `"password123"`, they'd have the **identical hash**, and a rainbow table cracks both instantly.

### Salting

A **salt** is random data added to the password **before** hashing, unique per user, so identical passwords produce **different** hashes.

```
User A: password="123456" + salt="x7Gk2p" → hash("123456x7Gk2p") → hashA
User B: password="123456" + salt="pQ9mZa" → hash("123456pQ9mZa") → hashB

hashA ≠ hashB, even though both users typed the exact same password!
```

```
             password + salt
                    │
                    ▼
              hash function
                    │
                    ▼
        stored: { hash, salt }   ← salt is stored alongside (not secret),
                                    its only job is to make each hash unique
```

**Why this matters:** rainbow tables become useless — an attacker would need a separate table *per user's unique salt*, which is computationally infeasible at scale. Modern algorithms like **bcrypt** and **argon2** handle salting automatically and are deliberately *slow* (computationally expensive) to make brute-forcing painfully slow, even with powerful hardware.

### Password Verification

Since hashing is one-way, login doesn't "decrypt" the stored hash — it **re-hashes the login attempt** and compares.

```
Login attempt: "mypassword123"
                    │
                    ▼
        hash("mypassword123" + stored_salt)
                    │
                    ▼
         compare to stored hash
                    │
        ┌───────────┴───────────┐
        ▼                       ▼
      Match                  No match
   → 200 OK, log in         → 401 Unauthorized
```

```sql
-- Nothing like this ever happens:
SELECT * FROM users WHERE password = 'mypassword123';  ❌

-- Instead:
1. SELECT hash, salt FROM users WHERE email = '...';
2. computed_hash = bcrypt.hash(input_password, salt)
3. if computed_hash == stored_hash: login succeeds
```

---

## 3. Session-Based Authentication (Cookies)

**How it works:** after login, the **server** creates and stores a session, and gives the browser a small reference (a cookie) to that session.

```
1) Login request
   Client → Server: POST /login { email, password }

2) Server verifies password, creates a session in its store
   Session Store (Redis/DB):
   ┌───────────┬────────────────┐
   │ session_id │  user_id: 42   │
   │ "abc123"   │  expires: ...  │
   └───────────┴────────────────┘

3) Server responds with a cookie
   Server → Client: Set-Cookie: session_id=abc123

4) Browser stores it, sends it automatically on every future request
   Client → Server: Cookie: session_id=abc123

5) Server looks up "abc123" in the session store → finds user_id=42
   → knows who's making the request, without re-verifying the password
```

```
   Browser                    Server                  Session Store
      │   Cookie: session_id=abc123  │                       │
      ├──────────────────────────────►                       │
      │                               │  lookup "abc123"       │
      │                               ├────────────────────────►
      │                               │  ◄────────────────────────
      │                               │  { user_id: 42 }        │
      │  ◄─────────────────────────── │                       │
      │  200 OK (personalized data)   │                       │
```

### Why use it

- **Server has full control** — can instantly invalidate a session (e.g. force logout, ban a user) by deleting it from the store.
- Good for **traditional web apps** where client and server are tightly coupled (server-rendered pages, same-domain cookies).

### Downsides

- **Stateful** — the server must store and look up session data on every request, which doesn't scale as easily across multiple servers without a shared session store (e.g. Redis).
- **CSRF risk** — cookies are sent automatically by the browser, so extra protection (CSRF tokens) is needed for state-changing requests.
- Awkward for **mobile apps / cross-domain APIs**, since cookie behavior differs across platforms.

---

## 4. Token-Based Authentication (JWT)

**How it works:** after login, the server issues a **self-contained token** (a JWT) that the client stores and sends manually with every request — the server doesn't need to store anything.

### JWT Structure

```
JWT = Header . Payload . Signature

eyJhbGciOiJIUzI1NiJ9 . eyJ1c2VyX2lkIjo0Mn0 . SflKxwRJSMeKKF2QT4fw...
      ▲                        ▲                       ▲
   Header                  Payload                 Signature
  (algorithm)          (user_id, expiry,       (proves it wasn't
                        claims — readable        tampered with)
                        by anyone, NOT secret!)
```

```
Decoded Payload example:
{
  "user_id": 42,
  "role": "admin",
  "exp": 1735689600      ← expiration timestamp
}
```

**Important:** the payload is **base64-encoded, not encrypted** — anyone can read it. Never put secrets (passwords, credit card numbers) inside a JWT payload. The signature only proves it **wasn't tampered with**, not that it's private.

### How Verification Works

```
1) Login → Server verifies credentials → signs a JWT with a secret key
   Server → Client: { "token": "eyJhbGc...xyz" }

2) Client stores the token (localStorage / memory / secure storage)

3) Client sends it manually on every request
   Authorization: Bearer eyJhbGc...xyz

4) Server verifies the SIGNATURE using its secret key
   (no database lookup needed — the token is self-contained!)

5) If signature valid + not expired → request proceeds
```

```
   Client                                  Server
      │  Authorization: Bearer <JWT>         │
      ├───────────────────────────────────────►
      │                          Verify signature using secret key
      │                          (NO database/session lookup needed)
      │  ◄───────────────────────────────────────
      │  200 OK (if valid)                    │
```

### Why use it

- **Stateless** — server doesn't store sessions, so it scales horizontally with zero shared state (any server instance can verify any token independently).
- Works naturally across **mobile apps, SPAs, microservices, cross-domain APIs**.

### Downsides

- **Can't be revoked instantly** — once issued, a JWT is valid until it expires; there's no built-in "delete this session" like with server-side sessions (workarounds exist: short expiry + refresh tokens, or a server-side blocklist, which reintroduces state).
- If the secret key leaks, **all tokens** can be forged.
- Payload is visible to anyone who has the token (though tamper-proof).

### Session vs JWT — Side by Side

| | Session (Cookie) | JWT (Token) |
|---|---|---|
| Where is state stored | Server (session store) | Client (self-contained token) |
| Scalability | Needs shared session store | Naturally stateless, scales easily |
| Instant revocation | Easy (delete from store) | Hard (must wait for expiry or use blocklist) |
| Best for | Traditional web apps, same-domain | Mobile apps, SPAs, APIs, microservices |
| Sent via | Cookie (automatic) | Authorization header (manual) |

---

## 5. Access Token vs Refresh Token

Using one long-lived JWT is risky — if stolen, it's valid for a long time. The fix: **two tokens with different lifespans and purposes**.

```
Access Token                         Refresh Token
- Short-lived (e.g. 15 minutes)      - Long-lived (e.g. 7–30 days)
- Sent with every API request        - Sent ONLY to get a new access token
- If stolen, damage window is small  - Stored more securely, used rarely
```

```
1) Login → Server issues BOTH tokens
   { "access_token": "...", "refresh_token": "..." }

2) Client uses access_token for normal requests
   Authorization: Bearer <access_token>

3) After 15 minutes, access_token expires
   Request → 401 Unauthorized (expired)

4) Client sends refresh_token to a dedicated endpoint
   POST /auth/refresh   { "refresh_token": "..." }

5) Server verifies refresh_token → issues a NEW access_token
   (user stays logged in without re-entering their password)
```

```
   Access token lifecycle:

   [Login] ──15min──► [Expires] ──refresh──► [New access token] ──15min──►...
                            │
                            ▼
                  refresh_token used here
                  (itself expires after days/weeks,
                   forcing full re-login eventually)
```

**Why this matters:** it balances security and user experience — short-lived access tokens limit the damage if one leaks, while refresh tokens avoid forcing the user to log in again every 15 minutes.

---

## 6. Cookies vs Sessions (clearing up the relationship)

These aren't two competing systems — a **cookie is the delivery mechanism**, a **session is the server-side data** it points to.

```
Cookie  = the small piece of data stored in the browser
          (could hold a session_id, OR a JWT, OR anything else)

Session = server-side storage of "who is this user" + their state
          (usually looked up VIA a cookie holding a session_id)
```

```
"Session-based auth" typically means:
   Cookie → holds session_id → server looks up session data

"JWT-based auth" can ALSO be delivered via a cookie:
   Cookie → holds the JWT itself → server verifies signature,
                                    no session lookup needed
```

**Necessity:** don't confuse "uses a cookie" with "is session-based" — you can absolutely store a JWT inside a cookie (common for web apps wanting both httpOnly cookie security AND stateless verification).

---

## 7. OAuth2 — Delegated Authorization ("Login with Google")

OAuth2 is often misunderstood as "a login system" — it's actually a protocol for **delegated authorization**: letting one app access your data on another platform, **without ever seeing your password** for that platform.

```
Scenario: "Sign in with Google" on a third-party app "MyApp"

                    MyApp                        Google
                      │                             │
1) User clicks         │  "Login with Google"        │
   "Login with Google" ├────────────────────────────►│
                      │                             │
2) Google asks user     │  ◄────────────────────────────
   to approve access    │  Login page + consent screen
   ("MyApp wants to     │  "MyApp wants: your email,
    see your email")    │   profile photo — Allow?"
                      │                             │
3) User approves         │  ────────────────────────────►
                      │                             │
4) Google redirects      │  ◄────────────────────────────
   back with a code      │  redirect_uri?code=AUTH_CODE
                      │                             │
5) MyApp's SERVER         │  exchange code for token
   exchanges the code    ├────────────────────────────►
   for an access token   │  ◄────────────────────────────
                      │  access_token (for Google APIs) │
                      │                             │
6) MyApp uses the token  │  GET /userinfo (with token)  │
   to fetch user's        ├────────────────────────────►
   email/profile          │  ◄────────────────────────────
                      │  { email, name, picture }     │
```

```
Key insight: MyApp NEVER sees the user's Google password.
             MyApp only receives a limited-scope token,
             good for exactly what the user approved
             ("see your email" — NOT "post on your behalf"
              unless that scope was separately requested/approved).
```

### Why use it

- User doesn't need to create/remember yet another password for your app.
- Your app never handles/stores a password for a service it doesn't own — massively reduces liability if your database ever leaks.
- User can revoke access anytime from Google's account settings, without changing their password.

### When it's the right choice

```
Building a consumer app that wants "Sign in with Google/GitHub/Facebook"?
→ OAuth2 (specifically the "Authorization Code" flow)

Building an internal API that needs to access a user's Google Drive files
on their behalf, even when they're not actively using your app?
→ OAuth2 (with refresh tokens for long-term access)

Building your own app's login system, no third-party involved?
→ You don't need OAuth2 — plain username/password + JWT/session is enough
```

---

## 8. Other Authentication Mechanisms (situational, but important to know)

### API Keys

A single static secret string identifying an **application/service**, not an individual user — common for server-to-server communication.

```
Client (a server, script, or third-party integration)
   │  X-API-Key: sk_live_51H8f2j...
   ├────────────────────────────►
   │  Server checks key against a database of valid keys
```

**When to use:** machine-to-machine integrations (e.g. a weather API, a payment gateway SDK) where there's no "login" concept — just "is this a recognized, authorized client application."

**Downside:** if a key leaks, it must be manually revoked/rotated — there's no per-request expiry like JWTs have.

### Basic Authentication

Username and password sent **on every single request**, base64-encoded (not encrypted!) in the header.

```
Authorization: Basic dXNlcjpwYXNzd29yZA==
                       ▲
              base64("user:password") — trivially reversible,
              NOT secure without HTTPS
```

**When to use:** rarely, in modern systems — mostly for quick internal tools, dev/staging environments behind a firewall, or simple server-to-server auth where HTTPS is guaranteed and a lightweight approach is fine.

### Multi-Factor Authentication (MFA / 2FA)

Requires **two or more** independent proofs of identity — typically "something you know" (password) + "something you have" (a code from your phone).

```
Step 1: password       → "something you know"
Step 2: OTP/authenticator app code → "something you have"
(sometimes) Step 3: fingerprint/face → "something you are"
```

```
Login attempt
      │
      ▼
Password correct? ──No──► 401
      │ Yes
      ▼
Send/request OTP code
      │
      ▼
OTP correct? ──No──► 401
      │ Yes
      ▼
Login succeeds
```

**Why it matters:** even if a password is stolen (phishing, breach, reused password), the attacker still can't log in without the second factor — dramatically reduces account takeover risk.

### Single Sign-On (SSO)

One login grants access to **multiple related applications**, common in companies (log in once, access email, HR system, Slack, etc. without re-entering credentials).

```
User logs into Identity Provider (e.g. Okta, Azure AD) ONCE
              │
   ┌──────────┼──────────┬──────────┐
   ▼          ▼          ▼          ▼
 App 1      App 2      App 3      App 4
(email)    (HR)      (Slack)    (Wiki)
   All trust the SAME identity provider —
   no separate login needed for each app
```

**When to use:** enterprise/organizational systems with many internal tools — built on top of protocols like SAML or OAuth2/OIDC.

---

## 9. Superiority Comparison — Which Mechanism, When?

```
┌────────────────────┬─────────────────────────────────────────────┐
│ Situation            │ Best fit                                     │
├────────────────────┼─────────────────────────────────────────────┤
│ Traditional server-   │ Session + Cookie                             │
│ rendered web app      │                                               │
├────────────────────┼─────────────────────────────────────────────┤
│ Mobile app / SPA /    │ JWT (access + refresh tokens)                 │
│ public API             │                                               │
├────────────────────┼─────────────────────────────────────────────┤
│ Microservices          │ JWT — stateless, any service can verify       │
│ talking to each other  │ independently without a shared session store │
├────────────────────┼─────────────────────────────────────────────┤
│ "Sign in with Google"  │ OAuth2 (Authorization Code flow)              │
│ / third-party login    │                                               │
├────────────────────┼─────────────────────────────────────────────┤
│ Server-to-server /     │ API Key                                       │
│ third-party integration│                                               │
├────────────────────┼─────────────────────────────────────────────┤
│ Extra protection for   │ MFA/2FA, layered on top of ANY of the above  │
│ sensitive accounts     │                                               │
├────────────────────┼─────────────────────────────────────────────┤
│ Enterprise, many        │ SSO (built on SAML/OAuth2/OIDC)               │
│ internal apps            │                                               │
└────────────────────┴─────────────────────────────────────────────┘
```

**Key insight:** these aren't strictly ranked "better vs worse" — they solve **different problems**. A real production system often uses several *together*: JWT for API auth + OAuth2 for third-party login + MFA layered on top + API keys for server-to-server calls.

---

## 10. RBAC — Role-Based Access Control

Once you know **who** someone is (authentication), you need to decide **what they can do** (authorization). RBAC is the most common pattern: assign users to **roles**, and roles to **permissions**.

```
        Users                Roles              Permissions
   ┌───────────┐        ┌───────────┐        ┌────────────────┐
   │  Alice     │───────►│   Admin    │───────►│ create_post      │
   │            │        │            │        │ delete_post      │
   │            │        │            │        │ ban_user          │
   └───────────┘        └───────────┘        │ view_analytics    │
   ┌───────────┐        ┌───────────┐        └────────────────┘
   │  Bob       │───────►│   Editor   │───────►┌────────────────┐
   └───────────┘        └───────────┘        │ create_post      │
   ┌───────────┐        ┌───────────┐        │ edit_post        │
   │  Charlie   │───────►│   Viewer   │       └────────────────┘
   └───────────┘        └───────────┘───────►┌────────────────┐
                                              │ view_post        │
                                              └────────────────┘
```

```sql
-- Simplified schema
users (id, name, role_id)
roles (id, name)              -- "admin", "editor", "viewer"
permissions (id, name)        -- "create_post", "delete_post", ...
role_permissions (role_id, permission_id)   -- many-to-many
```

### Checking Authorization with RBAC

```
Request: DELETE /posts/55   (by Bob, role = "editor")
              │
              ▼
   Does "editor" role have "delete_post" permission?
              │
        ┌─────┴─────┐
        ▼           ▼
       No           Yes
        │            │
        ▼            ▼
   403 Forbidden   Proceed with deletion
```

**Why use RBAC over checking `if user.name == "admin"` scattered everywhere:**

```
❌ Hardcoded checks                     ✅ RBAC
if user.email == "admin@co.com":         if user.role.has_permission("delete_post"):
    allow_delete()                            allow_delete()

→ Adding a new admin means editing        → Adding a new admin means just
  code in every place this check           assigning them the "admin" role —
  appears                                   zero code changes
```

### RBAC vs Permissions (fine-grained control)

Sometimes roles alone aren't precise enough — you need **object-level** permissions (e.g. "Bob can edit *his own* posts, but not everyone's").

```
Role-based (coarse):              Permission + ownership check (fine-grained):
"editor" can edit_post             "editor" can edit_post
→ ANY post                         AND post.author_id == current_user.id
                                    → only THEIR OWN posts
```

```
Request: PATCH /posts/55  (Bob, role="editor", post.author_id=99)
              │
              ▼
   Does Bob have "edit_post" permission?  → Yes
              │
              ▼
   Is Bob the author of post 55?  → No (author_id=99, Bob=42)
              │
              ▼
        403 Forbidden
```

---

## 11. Putting It All Together — A Realistic Modern Auth Flow

```
┌──────────────────────────────────────────────────────────────────┐
│ 1. REGISTRATION                                                     │
│    password → salted + hashed (bcrypt/argon2) → stored              │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ 2. LOGIN                                                             │
│    verify password → check MFA code (if enabled)                    │
│    → issue access_token (15 min) + refresh_token (7 days)           │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ 3. API REQUESTS                                                      │
│    Authorization: Bearer <access_token>                             │
│    → server verifies JWT signature (stateless, no DB lookup)         │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ 4. AUTHORIZATION CHECK (RBAC)                                        │
│    decoded token → user_id, role                                    │
│    → does role have permission for this action?                     │
│    → is user the owner of this specific resource (if needed)?       │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ 5. TOKEN EXPIRY                                                      │
│    access_token expires after 15 min                                │
│    → client sends refresh_token → gets a new access_token           │
│    → refresh_token itself expires after 7 days → full re-login       │
└──────────────────────────────────────────────────────────────────┘
```

---

## Quick-Reference: Why Each Concept Exists

| Concept | Necessity in one line |
|---|---|
| Hashing | Makes stored passwords irreversible even if the database leaks |
| Salting | Prevents identical passwords from producing identical hashes |
| Password verification | Compares a re-hashed login attempt, never decrypts anything |
| Session + Cookie | Server-controlled auth, easy instant revocation, best for classic web apps |
| JWT (Access token) | Stateless, scalable auth for APIs, mobile apps, microservices |
| Refresh token | Lets access tokens stay short-lived without forcing frequent re-logins |
| OAuth2 | Lets users log in via a trusted provider without sharing their password with your app |
| API Key | Identifies an application/service, not an individual human user |
| Basic Auth | Simple credential-per-request scheme, only safe over HTTPS, mostly legacy/internal use |
| MFA/2FA | Adds a second proof of identity so a leaked password alone isn't enough |
| SSO | One login across many internal/enterprise apps |
| RBAC | Manages "what can this user do" without scattering hardcoded checks everywhere |
| Object-level permissions | Adds ownership checks when role alone is too coarse |
