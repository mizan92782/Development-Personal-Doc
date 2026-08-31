# LEVEL 9 — Django / FastAPI Internals: The Real Request Lifecycle

Goal: when you write `@app.get("/users")` or `def my_view(request):`, you should be able to
picture **every hop** the request takes before your function even runs, and every hop the
response takes on its way back out.

---

# PART A — FastAPI Request Lifecycle

## The Big Picture

```
┌──────────┐
│  Client  │  HTTP GET /users
└────┬─────┘
     │  raw bytes over TCP socket
     ▼
┌─────────────────────────────────────────────────────────────┐
│                    ASGI SERVER (uvicorn)                     │
│  parses raw HTTP → builds an ASGI "scope" dict                │
└────┬───────────────────────────────────────────────────────┘
     │ calls: await app(scope, receive, send)
     ▼
┌─────────────────────────────────────────────────────────────┐
│                  MIDDLEWARE STACK (onion layers)              │
│   CORS → GZip → Auth → Custom middleware → ...                │
└────┬───────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────┐
│                         ROUTING                                │
│   Starlette Router matches path "/users" + method GET          │
│   to the registered endpoint function                          │
└────┬───────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────┐
│                  DEPENDENCY INJECTION                          │
│   FastAPI resolves every Depends(...) in the signature:        │
│   DB sessions, current_user, query params, etc.                │
└────┬───────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────┐
│                   VALIDATION (Pydantic)                        │
│   Path/query/body params are parsed & type-checked against      │
│   the function's type hints / Pydantic models                  │
└────┬───────────────────────────────────────────────────────┘
     │  (if invalid → 422 Unprocessable Entity, short-circuits here)
     ▼
┌─────────────────────────────────────────────────────────────┐
│                      YOUR ENDPOINT                              │
│              async def get_users(...): ...                     │
└────┬───────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────┐
│                        DATABASE                                 │
│    await db.execute(...) — non-blocking I/O via async driver   │
└────┬───────────────────────────────────────────────────────┘
     │  returns raw rows / ORM objects
     ▼
┌─────────────────────────────────────────────────────────────┐
│               RESPONSE SERIALIZATION (Pydantic)                 │
│   Return value validated against response_model, converted      │
│   to JSON-safe dict, then to bytes                              │
└────┬───────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────────┐
│               MIDDLEWARE STACK (unwinding, reverse order)       │
└────┬───────────────────────────────────────────────────────┘
     │  await send({"type": "http.response.start", ...})
     │  await send({"type": "http.response.body", ...})
     ▼
┌──────────┐
│  Client  │  ← HTTP response
└──────────┘
```

## Step-by-step, with code

### 1. ASGI Server

FastAPI itself doesn't listen on a socket — **uvicorn** (or hypercorn/daphne) does. It speaks raw
HTTP/WebSocket over TCP, parses it, and hands off to your app using the **ASGI protocol**: a
simple async callable contract.

```python
# This is roughly what uvicorn calls on every request:
async def app(scope, receive, send):
    """
    scope: dict describing the request (method, path, headers, query_string...)
    receive: async callable to pull the request body in chunks
    send: async callable to push the response back out
    """
    ...
```

FastAPI's `app = FastAPI()` object literally implements this `__call__(scope, receive, send)`
interface — that's the entire contract between the server and your framework.

Running it:
```bash
uvicorn main:app --host 0.0.0.0 --port 8000
```

### 2. Middleware

Middleware wraps the whole app in layers, like an onion. Each layer can inspect/modify the
request on the way in, and the response on the way out — and can short-circuit entirely (e.g.,
reject unauthenticated requests before they ever reach routing).

```python
from starlette.middleware.base import BaseHTTPMiddleware
import time

class TimingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        start = time.perf_counter()
        response = await call_next(request)     # <- calls the NEXT layer inward
        response.headers["X-Process-Time"] = str(time.perf_counter() - start)
        return response                           # <- then unwinds back outward

app.add_middleware(TimingMiddleware)
```

Execution order: middleware registered first wraps *outermost*, so requests pass through it
first going in, and last coming back out.

### 3. Routing

Starlette's `Router` holds a list of `(path_pattern, method, endpoint)` entries, compiled to
regex. On each request it walks the list and matches `scope["path"]` + `scope["method"]`.

```python
@app.get("/users")
async def get_users():
    ...

# Internally roughly equivalent to:
router.add_route(path="/users", methods=["GET"], endpoint=get_users)
```

Path parameters like `/users/{id}` compile to a regex with named groups, extracted into
`scope["path_params"]`.

### 4. Dependency Injection

Every `Depends(...)` in your function signature is resolved **before** your function body runs.
FastAPI builds a dependency graph, resolves it (including nested dependencies), and caches
results per-request if `use_cache=True` (the default).

```python
from fastapi import Depends

async def get_db():
    db = SessionLocal()
    try:
        yield db          # anything before yield = setup, after = teardown
    finally:
        db.close()

async def get_current_user(token: str = Depends(oauth2_scheme)):
    return decode_token(token)

@app.get("/users")
async def get_users(
    db: Session = Depends(get_db),
    user: User = Depends(get_current_user),
):
    ...
```

`Depends` functions using `yield` behave like context managers — FastAPI runs the "after yield"
teardown code even if the endpoint raises an exception.

### 5. Validation (Pydantic)

Type hints in your function signature aren't just documentation — FastAPI actively uses them at
runtime, via Pydantic, to parse and validate incoming data.

```python
from pydantic import BaseModel

class UserCreate(BaseModel):
    name: str
    age: int

@app.post("/users")
async def create_user(user: UserCreate):
    # by the time you get here, `user.name` is guaranteed a str,
    # `user.age` is guaranteed an int — or the client already got a 422
    ...
```

If the client sends `{"name": "Bob", "age": "not-a-number"}`, FastAPI never calls your function
at all — it returns `422 Unprocessable Entity` with a structured error body automatically.

### 6. The Endpoint

This is finally *your* code — everything above is machinery to get clean, validated,
dependency-resolved arguments into this function.

```python
@app.get("/users")
async def get_users(db: Session = Depends(get_db)):
    result = await db.execute(select(UserModel))
    return result.scalars().all()
```

### 7. Database

With an async driver (`asyncpg`, `databases`, async SQLAlchemy), `await db.execute(...)` releases
control back to the event loop while waiting on the network round-trip to Postgres — this is the
sync-vs-async payoff from Level 8 in action.

### 8. Response Serialization

If you declared `response_model=UserOut`, FastAPI validates *your return value* against that
model too (filtering out fields not declared in it — handy for hiding `password_hash` etc.),
then serializes to JSON.

```python
class UserOut(BaseModel):
    id: int
    name: str

@app.get("/users", response_model=list[UserOut])
async def get_users(db: Session = Depends(get_db)):
    return await fetch_users(db)   # extra fields silently stripped to match UserOut
```

### 9. Response

The serialized bytes get wrapped in ASGI messages and sent back through `send()`, bubbling back
out through every middleware layer in reverse order, and finally out through uvicorn to the
client socket.

---

# PART B — Django Request Lifecycle

## The Big Picture

```
┌──────────┐
│  Client  │  HTTP GET /users/
└────┬─────┘
     ▼
┌─────────────────────────────────────────────────────────────┐
│                  WSGI / ASGI SERVER                             │
│   gunicorn/uwsgi (WSGI) or uvicorn/daphne (ASGI)                │
│   builds an "environ" dict (WSGI) or "scope" dict (ASGI)        │
└────┬───────────────────────────────────────────────────────┘
     ▼
┌─────────────────────────────────────────────────────────────┐
│                  MIDDLEWARE (top → bottom on the way IN)        │
│  SecurityMiddleware → SessionMiddleware → CsrfMiddleware →       │
│  AuthenticationMiddleware → your custom middleware               │
└────┬───────────────────────────────────────────────────────┘
     ▼
┌─────────────────────────────────────────────────────────────┐
│                     URL RESOLVER                                 │
│   Walks ROOT_URLCONF, matches "/users/" against urlpatterns       │
└────┬───────────────────────────────────────────────────────┘
     ▼
┌─────────────────────────────────────────────────────────────┐
│                         VIEW                                     │
│    def user_list(request): / class UserListView(APIView):        │
└────┬───────────────────────────────────────────────────────┘
     ▼
┌─────────────────────────────────────────────────────────────┐
│                          ORM                                     │
│   User.objects.filter(...) → builds SQL → executes on DB          │
└────┬───────────────────────────────────────────────────────┘
     ▼
┌─────────────────────────────────────────────────────────────┐
│                       SERIALIZER (DRF)                            │
│   Converts model instances ↔ Python dicts ↔ JSON                  │
└────┬───────────────────────────────────────────────────────┘
     ▼
┌─────────────────────────────────────────────────────────────┐
│              MIDDLEWARE (bottom → top on the way OUT)             │
└────┬───────────────────────────────────────────────────────┘
     ▼
┌──────────┐
│  Client  │  ← HTTP response
└──────────┘
```

## Step-by-step, with code

### 1. WSGI / ASGI

Django's entry point is a callable object, same idea as ASGI above but WSGI is **synchronous**.

```python
# wsgi.py
def application(environ, start_response):
    """
    environ: dict with all request data (method, path, headers as env vars, etc.)
    start_response: callback to set status code + headers
    """
    ...
    start_response("200 OK", [("Content-Type", "application/json")])
    return [b'{"ok": true}']
```

Django 3.1+ also supports **ASGI** (`asgi.py`) for async views, run via uvicorn/daphne instead of
gunicorn — this is what enables `async def` views in Django.

### 2. Middleware

Django middleware is a chain of callables, each wrapping the next, defined in
`settings.MIDDLEWARE` (order matters — it's a literal onion, same concept as FastAPI's).

```python
class SimpleTimingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response   # the next layer inward

    def __call__(self, request):
        start = time.time()
        response = self.get_response(request)   # calls into the next middleware / view
        response["X-Process-Time"] = str(time.time() - start)
        return response
```

```python
# settings.py
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "myapp.middleware.SimpleTimingMiddleware",
]
```

### 3. URL Resolver

Django compiles `urlpatterns` into a tree of regex/path converters and walks it top-down looking
for the first match.

```python
# urls.py
from django.urls import path
urlpatterns = [
    path("users/", views.user_list),
    path("users/<int:pk>/", views.user_detail),
]
```

`<int:pk>` is a **path converter** — Django extracts `pk` from the URL, casts it to `int`, and
passes it as a keyword argument to your view function.

### 4. View

The resolved view is called with `request` plus any captured URL kwargs.

```python
# Function-based view
def user_list(request):
    users = User.objects.all()
    data = [{"id": u.id, "name": u.name} for u in users]
    return JsonResponse(data, safe=False)

# Class-based view (Django REST Framework)
from rest_framework.views import APIView
from rest_framework.response import Response

class UserListView(APIView):
    def get(self, request):
        users = User.objects.all()
        serializer = UserSerializer(users, many=True)
        return Response(serializer.data)
```

### 5. ORM

`User.objects.filter(...)` doesn't hit the database immediately — Django QuerySets are **lazy**.
The SQL is only built and executed when the queryset is actually evaluated (iterated, listed,
counted, etc.).

```python
qs = User.objects.filter(is_active=True)   # no SQL run yet
print(qs.query)                             # inspect the SQL Django WILL run
users = list(qs)                            # NOW it hits the database
```

### 6. Serializer (DRF)

Django REST Framework serializers do for Django what Pydantic does for FastAPI: convert between
Python/ORM objects and JSON-safe primitives, with validation.

```python
from rest_framework import serializers

class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ["id", "name", "email"]

serializer = UserSerializer(user_instance)
serializer.data   # {"id": 1, "name": "Alice", "email": "a@x.com"}

# Deserializing incoming data:
serializer = UserSerializer(data=request.data)
serializer.is_valid(raise_exception=True)
serializer.save()
```

### 7. Response

`Response`/`JsonResponse` gets converted back to raw bytes + status + headers, and the WSGI/ASGI
server writes it back to the client socket, unwinding back out through the middleware chain in
reverse order.

---

# Side-by-Side Comparison

| Stage | FastAPI | Django |
|---|---|---|
| Server protocol | ASGI (async native) | WSGI (sync) or ASGI (Django 3.1+) |
| Routing | Starlette router, decorator-based | `urlpatterns`, regex/path converter tree |
| Input validation | Pydantic, via type hints | Django Forms / DRF Serializers |
| Data layer | Any (often SQLAlchemy, async drivers) | Built-in ORM, lazy QuerySets |
| Output shaping | Pydantic `response_model` | DRF Serializers |
| Dependency injection | Explicit, via `Depends(...)` | Not built-in (uses class attributes / mixins instead) |
| Async by default | Yes | Opt-in (async views, async ORM support growing) |

**The core insight to internalize:** both frameworks are really just an ordered pipeline of
*middleware → routing → input transformation → your business logic → output transformation →
middleware unwind*. Once you see it as one shape with different named stages, jumping between
frameworks (or debugging "why isn't my middleware running" / "why is this field missing from the
response") becomes a matter of asking "which stage of the pipeline is misbehaving?"
