# LEVEL 2 — API Design (Beyond Just Creating Endpoints)

Anyone can make an endpoint that returns data. **API design** is about making endpoints that are *predictable*, *consistent*, and *scalable* — so any developer (including future-you) can guess how the API behaves without reading docs. This guide covers every concept you listed, with diagrams and the *why* behind each.

```
GET    /api/users/123     → Read one user
POST   /api/users          → Create a user
PATCH  /api/users/123     → Update part of a user
DELETE /api/users/123     → Delete a user
```

Notice: the **URL never changes verb-by-verb** (`/getUser`, `/createUser` ❌). The **HTTP method** carries the action. This one idea is the foundation of REST.

---

## 1. REST — Resource-Oriented Design

**REST (Representational State Transfer)** is a design style where everything in your system is modeled as a **resource** (a noun), and you act on it using standard HTTP methods (verbs).

```
❌ Action-oriented (RPC-style)          ✅ Resource-oriented (REST-style)
POST /createUser                        POST   /users
POST /getUserById                       GET    /users/123
POST /updateUser                        PATCH  /users/123
POST /deleteUser                        DELETE /users/123
```

**Why it matters:** with REST, once you learn the pattern for `/users`, you already know how `/products`, `/orders`, `/comments` will behave — same verbs, same status codes, same shape. This consistency is what lets teams and frontend/backend developers move fast without constantly asking "wait, how does this endpoint work?"

```
                 RESOURCE = noun
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
     /users          /orders         /products
        │               │               │
   ┌────┴────┐     ┌────┴────┐     ┌────┴────┐
   GET       │     GET       │     GET       │
   POST    verbs   POST    verbs   POST    verbs
   PATCH     │     PATCH     │     PATCH     │
   DELETE    │     DELETE    │     DELETE    │
```

---

## 2. URL Structure

Good URL structure reads like a **sentence describing a hierarchy of resources**, not a list of function names.

```
GET /api/users                    → collection of users
GET /api/users/123                → one specific user
GET /api/users/123/orders          → orders belonging to user 123
GET /api/users/123/orders/45       → one specific order of user 123
```

```
/api  /users  /123  /orders  /45
  │      │      │       │      │
  │      │      │       │      └── specific order ID
  │      │      │       └── nested resource (orders belonging to user)
  │      │      └── specific resource ID
  │      └── resource collection (plural noun)
  └── namespace / API root
```

### Rules of thumb

| Do | Don't |
|---|---|
| `/users` (plural noun) | `/getAllUsers` |
| `/users/123` | `/user?id=123` |
| `/users/123/orders` | `/getOrdersForUser?userId=123` |
| lowercase, hyphenated | `/Users/GetUserOrders` |

**Necessity:** predictable URLs mean less documentation needed, fewer bugs from guessing wrong endpoint names, and easier caching/logging/monitoring since URL patterns map directly to resources.

---

## 3. HTTP Methods (mapped to CRUD)

Already covered in Level 1 — but in API design, this mapping must be **strictly consistent** across your entire API, not just "mostly followed."

```
Resource: /api/products/55

GET     /api/products/55    → 200 OK           (read)
POST    /api/products       → 201 Created      (create)
PUT     /api/products/55   → 200 OK           (full replace)
PATCH   /api/products/55   → 200 OK           (partial update)
DELETE  /api/products/55   → 204 No Content    (delete)
```

**Necessity:** if `POST /users/123` sometimes means "update" in one part of your API and `PATCH /orders/5` means "replace" elsewhere, every consumer of your API has to special-case each endpoint — this destroys the predictability REST is supposed to give you.

---

## 4. Status Codes (as design tool, not afterthought)

Status codes aren't just "what happened" — they're a **contract**. Clients (frontend, mobile apps, other services) branch their logic based on the status code *before* even looking at the body.

```
Client logic (pseudocode):

if status == 200 or 201:
    show success
elif status == 401:
    redirect to login
elif status == 403:
    show "not allowed" message
elif status == 404:
    show "not found" page
elif status == 422:
    show validation errors from body
elif status >= 500:
    show "something went wrong, try again"
```

**Necessity:** returning `200 OK` with `{ "error": "not found" }` in the body forces every client to parse the body just to know if something failed — status codes let clients make fast, cheap decisions using metadata alone.

---

## 5. Pagination

When a resource collection can have thousands/millions of rows, you **never** return all of them at once — you return a "page" at a time.

```
GET /api/products?page=2&limit=20

              Page 1              Page 2 (requested)        Page 3
        ┌─────────────────┐  ┌─────────────────┐   ┌─────────────────┐
        │ items 1–20       │  │ items 21–40      │   │ items 41–60      │
        └─────────────────┘  └─────────────────┘   └─────────────────┘
```

Common styles:

```
1) Offset/limit (page-based)
   GET /products?page=2&limit=20
   → skip 20, take next 20

2) Cursor-based (for large/real-time data)
   GET /products?cursor=eyJpZCI6NDB9&limit=20
   → "give me 20 items after this cursor position"
```

Example response shape:

```json
{
  "data": [ ... 20 products ... ],
  "pagination": {
    "page": 2,
    "limit": 20,
    "total_items": 483,
    "total_pages": 25
  }
}
```

**Necessity:** without pagination, `GET /products` on a table with 2 million rows would try to serialize and send all 2 million rows in one response — slow, memory-heavy, and likely to crash the server or the client.

---

## 6. Filtering

Filtering lets the client narrow down a collection to only the resources matching certain conditions.

```
GET /api/products?category=electronics&in_stock=true&min_price=100
```

```
All products (1000)
        │
        ▼  category=electronics
   Electronics (200)
        │
        ▼  in_stock=true
   In-stock electronics (150)
        │
        ▼  min_price=100
   In-stock electronics ≥ $100 (60)   ← final result sent to client
```

**Necessity:** without server-side filtering, the client would have to download the *entire* dataset and filter it locally — wasteful and often impossible at scale.

---

## 7. Sorting

Sorting controls the **order** of the returned items — usually via a `sort` query param.

```
GET /api/products?sort=-created_at
                        │
                        └── "-" prefix = descending
                            no prefix   = ascending
```

```
sort=price        → cheapest first
sort=-price       → most expensive first
sort=-created_at  → newest first
sort=name,-price  → sort by name, then by price descending (tie-breaker)
```

**Necessity:** sorting on the server (using database indexes) is far more efficient than fetching everything and sorting in the client/app — especially on mobile devices or large datasets.

---

## 8. Searching

Search lets the client find resources matching a **text query**, usually across one or more fields.

```
GET /api/products?search=wireless+headphones
```

```
                 search="wireless headphones"
                              │
                              ▼
          ┌───────────────────────────────────┐
          │   Search against: name, description │
          └───────────────────────────────────┘
                              │
                ┌─────────────┼─────────────┐
                ▼             ▼             ▼
       "Wireless Headphones" "Bluetooth      "USB Headphones"
        (name match)          Wireless Mic"   (no match, excluded)
                               (partial match)
```

**Necessity:** search is fundamentally different from filtering — filtering matches *exact* values (`category=electronics`), while search matches *fuzzy/partial text*, often needing full-text search engines (Postgres `tsvector`, Elasticsearch) once data grows large.

---

## 9. Combining Pagination + Filtering + Sorting + Searching

This is where they all come together in one real request:

```
GET /api/products?page=2&limit=20&sort=-created_at&category=electronics&search=headphones
```

```
Step 1: search="headphones"       → find matching products
Step 2: category="electronics"     → filter down to electronics only
Step 3: sort=-created_at           → order by newest first
Step 4: page=2&limit=20            → return items 21–40 of that result
```

```
     All Products
          │
          ▼  search
     Matches "headphones"
          │
          ▼  filter
     ...and category=electronics
          │
          ▼  sort
     ...ordered by newest first
          │
          ▼  paginate
     Return page 2 (items 21–40)
```

---

## 10. Versioning

As your API evolves, you'll need to make **breaking changes** (renamed fields, changed response shape, removed endpoints). Versioning lets you do that **without breaking existing clients**.

```
/api/v1/users     ← old clients keep using this, untouched
/api/v2/users     ← new clients use the new shape
```

Common approaches:

```
1) URL versioning (most common, most visible)
   /api/v1/users
   /api/v2/users

2) Header versioning
   Accept: application/vnd.myapi.v2+json

3) Query param versioning
   /api/users?version=2
```

```
                    Mobile App (old, v1)
                            │
                            ▼
     ┌───────────────────────────────────────┐
     │              API Gateway               │
     └───────────────────────────────────────┘
            │                        │
            ▼                        ▼
      /api/v1/users            /api/v2/users
      (old response shape)     (new response shape)
            │                        │
            ▼                        ▼
      Old Web App                New Web App
```

**Necessity:** without versioning, changing a field name from `full_name` to `name` would silently break every mobile app already published in app stores — those apps can't force-update instantly, so v1 must keep working while v2 rolls out.

---

## 11. Error Handling

A well-designed API returns errors in a **consistent, predictable shape** — not a random string, not a stack trace, not different formats per endpoint.

```
❌ Inconsistent errors                 ✅ Consistent error shape
"User not found"                       {
{ "err": true }                          "error": {
"Something broke, code 55"                  "code": "USER_NOT_FOUND",
                                             "message": "User with id 123
                                                 does not exist",
                                             "status": 404
                                          }
                                        }
```

```
Client                              Server
  │  GET /users/999                   │
  ├────────────────────────────────────►
  │                                    │  User not found in DB
  │  ◄────────────────────────────────────
  │  404 Not Found                    │
  │  {                                │
  │    "error": {                    │
  │      "code": "USER_NOT_FOUND",   │
  │      "message": "..."            │
  │    }                             │
  │  }                                │
```

**Necessity:** a predictable error shape lets the frontend write **one** generic error-handling function instead of custom parsing logic for every single endpoint.

---

## 12. Validation

Validation ensures the **incoming data is correct and safe** *before* it ever touches your database or business logic.

```
Client sends:
{
  "email": "not-an-email",
  "age": -5,
  "name": ""
}
```

```
                Request Body
                     │
                     ▼
          ┌─────────────────────┐
          │   Validation Layer    │
          │  - email format?      │
          │  - age >= 0?          │
          │  - name not empty?    │
          └─────────────────────┘
                     │
        ┌────────────┴────────────┐
        ▼                         ▼
     ❌ Fails                   ✅ Passes
  422 Unprocessable Entity      Continue to
  {                              business logic
    "errors": {
      "email": "Invalid email format",
      "age": "Must be >= 0",
      "name": "Name is required"
    }
  }
```

**Necessity:** validating at the API boundary (before hitting the database) prevents corrupt data, protects against injection attacks, and gives the client **immediately useful** error messages instead of a confusing 500 error from a database constraint failure three layers deep.

---

## 13. Full Request Lifecycle: Request → Validation → Database → Response

Let's trace the exact example you gave:

```
GET /api/products?page=2&limit=20&sort=-created_at
```

```
┌──────────────────────────────────────────────────────────────────┐
│ 1. CLIENT SENDS REQUEST                                            │
│    GET /api/products?page=2&limit=20&sort=-created_at              │
│    Headers: Authorization, Accept: application/json                │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ 2. ROUTING                                                          │
│    Server matches URL pattern → /api/products → ProductController  │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ 3. AUTHENTICATION / AUTHORIZATION (if required)                    │
│    Verify token → confirm identity → check permissions             │
│    Fail → 401 / 403                                                 │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ 4. VALIDATION of query params                                      │
│    page must be int >= 1     ✓                                     │
│    limit must be int, max 100 ✓                                    │
│    sort must be an allowed field ✓                                 │
│    Fail → 400 Bad Request                                           │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ 5. BUSINESS LOGIC / SERVICE LAYER                                   │
│    Translate params into a query plan:                              │
│    - offset = (page-1) * limit = 20                                │
│    - order by created_at DESC                                      │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ 6. DATABASE QUERY                                                   │
│    SELECT * FROM products                                          │
│    ORDER BY created_at DESC                                        │
│    LIMIT 20 OFFSET 20;                                              │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ 7. SERIALIZATION                                                    │
│    Convert DB rows → JSON objects                                  │
│    Wrap with pagination metadata                                   │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│ 8. RESPONSE SENT                                                    │
│    200 OK                                                           │
│    {                                                                 │
│      "data": [ ...20 products... ],                                │
│      "pagination": { "page": 2, "limit": 20, "total": 483 }         │
│    }                                                                 │
└──────────────────────────────────────────────────────────────────┘
```

---

## Quick-Reference: Why Each Concept Exists

| Concept | Necessity in one line |
|---|---|
| REST / resource-oriented | Predictable, consistent API shape across the whole system |
| URL structure | Self-explanatory endpoints, no need to memorize function names |
| HTTP methods | Action is carried by the verb, not invented in the URL |
| Status codes | Client can branch logic fast, without parsing the body |
| Pagination | Prevents sending/loading massive datasets in one response |
| Filtering | Lets the server (not client) narrow down large datasets efficiently |
| Sorting | Server-side ordering is faster and more scalable than client-side |
| Searching | Handles fuzzy/partial text matches, not just exact filters |
| Versioning | Lets you evolve the API without breaking old clients |
| Error handling | One consistent shape → one generic client-side error handler |
| Validation | Stops bad/malicious data before it reaches the database |
