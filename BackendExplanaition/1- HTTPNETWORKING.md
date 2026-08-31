# LEVEL 1 — HTTP & Networking (Backend Fundamentals)

This is the single most important concept area for any backend developer. Everything else (APIs, auth, microservices, caching) is built on top of HTTP. This guide explains every core piece with diagrams so the *why* sticks, not just the *what*.

---

## 1. What is HTTP?

**HTTP (HyperText Transfer Protocol)** is a **request–response** protocol. A client (browser, mobile app, Postman, another server) sends a **request**, and a server sends back a **response**. That's it — nothing happens unless the client asks first.

```
   CLIENT                              SERVER
   ┌──────────┐        Request          ┌──────────┐
   │ Browser/ │  ───────────────────►   │  Backend │
   │  App     │                         │  Server  │
   │          │  ◄───────────────────   │          │
   └──────────┘        Response         └──────────┘
```

### Anatomy of a Request

```
POST /api/users/login HTTP/1.1          ← Method + Path + Version
Host: example.com                        ┐
Content-Type: application/json           │  Headers
Authorization: Bearer eyJhbGciOi...      ┘
                                          
{                                        ┐
  "email": "user@example.com",           │  Body (payload)
  "password": "12345"                    │
}                                        ┘
```

### Anatomy of a Response

```
HTTP/1.1 200 OK                          ← Status line
Content-Type: application/json           ┐  Headers
Set-Cookie: session_id=abc123            ┘
                                          
{                                        ┐
  "id": 1,                                │  Body
  "name": "John Doe"                      │
}                                        ┘
```

**Key idea:** A request always has a method + path. A response always has a status code. Headers describe *metadata* on both sides; the body carries the actual *data*.

---

## 2. HTTP Methods

Each method tells the server **what kind of action** you want to perform. Think of them like verbs mapped to CRUD (Create, Read, Update, Delete).

| Method | CRUD Action | Meaning | Has Body? |
|---|---|---|---|
| **GET** | Read | Fetch a resource | No |
| **POST** | Create | Create a new resource | Yes |
| **PUT** | Update (full) | Replace a resource entirely | Yes |
| **PATCH** | Update (partial) | Modify part of a resource | Yes |
| **DELETE** | Delete | Remove a resource | Usually no |

```
GET    /users/5        → Fetch user 5
POST   /users           → Create a new user
PUT    /users/5        → Replace user 5 entirely
PATCH  /users/5        → Update only some fields of user 5
DELETE /users/5        → Delete user 5
```

### PUT vs PATCH (the classic confusion)

Both **update** a resource — the difference is **how much of it you send**.

```
Original resource:
{
  "id": 5,
  "name": "John",
  "email": "john@example.com",
  "age": 25
}
```

**PUT** → you send the **entire object**. Anything you omit gets wiped/reset.

```
PUT /users/5
{
  "name": "John",
  "email": "john@example.com",
  "age": 26
}
```
Result: full record replaced. If you forget a field, it may become `null`.

**PATCH** → you send **only the field(s) you want to change**.

```
PATCH /users/5
{
  "age": 26
}
```
Result: only `age` changes, everything else stays untouched.

> **Rule of thumb:** PUT = "here is the complete new version." PATCH = "here is just the diff."

---

## 3. HTTP Status Codes

Status codes are grouped by their **first digit** — this alone tells you the category before you even read the details.

```
1xx → Informational   (rarely used directly)
2xx → Success          ✅
3xx → Redirection      ↪️
4xx → Client Error     ❌ (you made a mistake)
5xx → Server Error     💥 (server made a mistake)
```

### 2xx — Success

| Code | Name | Meaning |
|---|---|---|
| **200** | OK | Standard success (GET, PUT, PATCH) |
| **201** | Created | New resource was created (POST) |
| **204** | No Content | Success, but nothing to return (e.g. DELETE) |

```
POST /users        → 201 Created  (new user made)
GET  /users/5       → 200 OK       (returns user data)
DELETE /users/5     → 204 No Content (deleted, nothing to send back)
```

### 4xx — Client Errors ("you did something wrong")

| Code | Name | Meaning |
|---|---|---|
| **400** | Bad Request | Malformed/invalid request data |
| **401** | Unauthorized | You are **not logged in** / no valid credentials |
| **403** | Forbidden | You **are** logged in, but not allowed to do this |
| **404** | Not Found | Resource doesn't exist |
| **409** | Conflict | Request conflicts with current state (e.g. duplicate email) |
| **422** | Unprocessable Entity | Data is well-formed but fails validation |
| **429** | Too Many Requests | Rate limit exceeded |

### 5xx — Server Errors ("we did something wrong")

| Code | Name | Meaning |
|---|---|---|
| **500** | Internal Server Error | Generic server crash/bug |
| **502** | Bad Gateway | A gateway/proxy got an invalid response from upstream server |
| **503** | Service Unavailable | Server is overloaded or down for maintenance |

### 401 vs 403 (the classic confusion)

Both mean "you can't do this" — but the *reason* is completely different.

```
401 Unauthorized
   "I don't know who you are."
   → Missing / invalid / expired login credentials.
   → Fix: log in / send a valid token.

403 Forbidden
   "I know who you are, but you're not allowed."
   → You ARE authenticated, but lack permission.
   → Fix: nothing you can do — you need higher privileges.
```

```
Request without token → Server: "Who are you?"           → 401
Request with valid token, but role = "user" trying        
  to access an admin-only route  → Server: "I know you,
  but no." → 403
```

> **Memory trick:** 401 = "please log in." 403 = "you're logged in, but banned from this door."

---

## 4. Headers

Headers are **metadata** attached to a request or response — extra information about the message, separate from the actual data (body).

```
Content-Type: application/json      ← "the body is JSON"
Authorization: Bearer <token>       ← "here's who I am"
Cookie: session_id=abc123           ← "here's my session"
User-Agent: Mozilla/5.0...          ← "here's my browser/device"
Accept: application/json            ← "I want JSON back"
```

### Content-Type

Tells the receiver **what format the body is in**, so it can be parsed correctly.

```
Content-Type: application/json        → body is JSON
Content-Type: application/x-www-form-urlencoded → form data
Content-Type: multipart/form-data     → file upload
Content-Type: text/html               → HTML page
```

If the server sends `Content-Type: application/json` but the body is actually plain text, the client will try to `JSON.parse()` it and fail.

### Authorization

Carries the **credentials** used to identify/authenticate the request — most commonly a token.

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6...
                ▲
                └── scheme (Bearer, Basic, etc.)
```

```
Client                          Server
  │  Authorization: Bearer <JWT>  │
  ├───────────────────────────────►
  │                                │  Verify token signature
  │                                │  Extract user identity
  │  ◄─────────────────────────────┤
  │  200 OK + protected data       │
```

---

## 5. Cookies

A cookie is a small piece of data the **server asks the browser to store**, and the browser **automatically sends it back** on every future request to that domain.

```
1) Login request
   Client → Server: POST /login

2) Server responds and sets a cookie
   Server → Client: Set-Cookie: session_id=abc123

3) Browser stores it automatically

4) Every future request, browser attaches it
   Client → Server: Cookie: session_id=abc123
```

Cookies are commonly used for **session-based authentication** — the server looks up `session_id` in its store (often Redis!) to know who's making the request, without the client needing to manually attach a token every time.

---

## 6. Idempotency

An operation is **idempotent** if calling it **once or many times produces the same end result**.

```
PUT /users/5  { "age": 26 }

Call it once  → age = 26
Call it twice → age = 26
Call it 100x  → age = 26   ✅ Same result every time → idempotent
```

```
POST /orders   { "item": "Pizza" }

Call it once  → 1 order created
Call it twice → 2 orders created
Call it 100x  → 100 orders created  ❌ Different result each time → NOT idempotent
```

| Method | Idempotent? |
|---|---|
| GET | ✅ Yes |
| PUT | ✅ Yes |
| DELETE | ✅ Yes (deleting an already-deleted item still results in "it's gone") |
| PATCH | ⚠️ Usually, but depends on implementation |
| POST | ❌ No (each call typically creates something new) |

**Why it matters:** if a network request times out and the client retries automatically, an idempotent endpoint is safe to retry blindly. A non-idempotent one (like POST for payment) could accidentally double-charge a user — this is why payment APIs often require a separate **idempotency key**.

---

## 7. Statelessness

HTTP is **stateless** — the server does **not remember anything** about previous requests by default. Every single request must carry **all the information** the server needs to process it (e.g. the auth token), because the server treats each request as if it's talking to a stranger every time.

```
Request 1: GET /profile     Authorization: Bearer TOKEN123
   → Server processes it, then FORGETS everything.

Request 2: GET /orders      Authorization: Bearer TOKEN123
   → Server has no memory of Request 1.
   → It must re-verify TOKEN123 all over again.
```

```
❌ Stateful thinking (wrong for HTTP):
   "The user already logged in 2 minutes ago, so the
    server should just remember them."

✅ Stateless reality:
   "Every request must prove who it is, every single time —
    that's what the Authorization header / cookie is for."
```

This is exactly *why* tokens, cookies, and sessions exist: to work around statelessness by giving the client something to send back each time, letting the server reconstruct "who is this?" on every request instead of relying on memory.

---

## 8. Authentication vs Authorization (the classic confusion)

These sound similar but answer two completely different questions.

```
AUTHENTICATION                    AUTHORIZATION
"Who are you?"                    "What are you allowed to do?"

Login with email/password    →    Checking if this user's role
Verifying a JWT token              can access /admin/dashboard
Checking session cookie            Checking if this user owns
                                    this specific resource
```

```
                 ┌─────────────────┐
   Request  ───► │  Authentication  │  → 401 if this fails
                 │  "Who is this?"  │     (no/invalid credentials)
                 └────────┬─────────┘
                          │ identity confirmed
                          ▼
                 ┌─────────────────┐
                 │  Authorization   │  → 403 if this fails
                 │ "Are they allowed│     (identity known, but
                 │  to do THIS?"    │      not permitted)
                 └────────┬─────────┘
                          │ permission granted
                          ▼
                     Request proceeds
                     → 200 / 201 / 204
```

**Real example:**

```
1. User logs in with email + password
   → Authentication: server confirms "this is user #42"

2. User tries to DELETE another user's post
   → Authorization: server checks "does user #42 have
      permission to delete THIS post?"
   → If not → 403 Forbidden
```

> **Memory trick:** Authentication = ID card at the door. Authorization = whether your ID lets you into the VIP room.

---

## 9. Putting It All Together

A full request lifecycle, combining everything above:

```
Client                                          Server
  │                                                │
  │  PATCH /users/5                                │
  │  Authorization: Bearer <token>                 │
  │  Content-Type: application/json                │
  │  { "age": 26 }                                 │
  ├────────────────────────────────────────────────►
  │                                                 │
  │                          1. Parse method + path │
  │                          2. Authenticate token  │  → fail? 401
  │                          3. Authorize action     │  → fail? 403
  │                          4. Validate body        │  → fail? 400/422
  │                          5. Apply partial update │
  │                                                 │
  │  ◄────────────────────────────────────────────────
  │  200 OK                                         │
  │  Content-Type: application/json                │
  │  { "id": 5, "age": 26 }                         │
  │                                                 │
```

---

## Quick-Reference Cheat Sheet

| Concept | One-line takeaway |
|---|---|
| GET vs POST | Read vs Create |
| PUT vs PATCH | Full replace vs partial update |
| 401 vs 403 | Not identified vs identified-but-not-allowed |
| 400 vs 422 | Malformed request vs valid format but failed business validation |
| 502 vs 503 | Bad response from upstream vs server itself is down/overloaded |
| Authentication vs Authorization | Who you are vs what you can do |
| Headers vs Body | Metadata about the message vs the actual data |
| Idempotent vs not | Same result no matter how many times you call it, or not |
| Statelessness | Server remembers nothing — every request must self-identify |
