# LEVEL 16 — Testing Deep Dive

## The Testing Pyramid

Different test types trade off **speed vs realism**. The pyramid shape is a guideline: have many
fast, cheap tests at the bottom, and progressively fewer, slower, more realistic tests as you go
up — because a full suite of only end-to-end tests would be too slow to run constantly, while a
suite of only unit tests could miss integration bugs.

```
                    ▲
                   ╱ ╲          End-to-End Tests
                  ╱   ╲         (few, slow, most realistic —
                 ╱     ╲         full app + real browser/client)
                ╱───────╲
               ╱         ╲      API Tests
              ╱           ╲     (test HTTP endpoints end-to-end
             ╱             ╲     within the app, no browser)
            ╱───────────────╲
           ╱                 ╲  Integration Tests
          ╱                   ╲ (test how multiple components
         ╱                     ╲ work together — e.g. view + DB)
        ╱───────────────────────╲
       ╱                         ╲  Unit Tests
      ╱                           ╲ (many, fast, isolated —
     ╱_____________________________╲ test ONE function/class alone)

     speed:   fastest ────────────────────────────────▶ slowest
     realism: least  ────────────────────────────────▶ most
```

## 1. Unit Test

Tests a **single function or class in isolation**, with all its dependencies faked/mocked out.
Fast because there's no database, no network, no filesystem — just pure logic.

```python
# utils.py
def calculate_discount(price, percent):
    if not (0 <= percent <= 100):
        raise ValueError("Percent must be between 0 and 100")
    return price * (1 - percent / 100)

# test_utils.py
def test_calculate_discount():
    assert calculate_discount(100, 20) == 80

def test_calculate_discount_invalid_percent():
    import pytest
    with pytest.raises(ValueError):
        calculate_discount(100, 150)
```

## 2. Integration Test

Tests how **two or more real components work together** — e.g., does your view function actually
save the right data to a real (test) database? Slower than unit tests since real I/O is involved.

```python
def test_create_user_saves_to_database(db_session):
    user = create_user(db_session, name="Alice", email="alice@example.com")
    fetched = db_session.query(User).filter_by(email="alice@example.com").first()
    assert fetched is not None
    assert fetched.name == "Alice"
```

## 3. API Test

Tests your actual HTTP endpoints — request in, response out — verifying status codes, response
bodies, and side effects, without needing a real browser.

```python
def test_get_users_endpoint(client):
    response = client.get("/users")
    assert response.status_code == 200
    assert isinstance(response.json(), list)
```

## 4. End-to-End (E2E) Test

Tests the **entire system as a real user would experience it** — often driving an actual browser
(Selenium/Playwright) against a fully running app + database + frontend. Slowest, most realistic,
most brittle (UI changes can break them even when logic is fine).

```python
# Using Playwright, testing a real running instance of the app
def test_user_can_register_and_login(page):
    page.goto("http://localhost:8000/register")
    page.fill("#email", "test@example.com")
    page.fill("#password", "secret123")
    page.click("button[type=submit]")
    page.wait_for_url("**/dashboard")
    assert page.inner_text("h1") == "Welcome, test@example.com"
```

```
Unit           →  does calculate_discount() do correct math?
Integration    →  does create_user() actually write to the DB correctly?
API            →  does POST /users return 201 and the right JSON?
End-to-End     →  can a real user click through registration in a real browser?
```

---

## pytest Fundamentals

```python
# test_math.py
def test_addition():
    assert 2 + 2 == 4

def test_string_upper():
    assert "hello".upper() == "HELLO"
```

```bash
pytest                       # run all tests
pytest test_math.py            # run one file
pytest -k "addition"             # run tests matching a keyword
pytest -v                          # verbose output
pytest --cov=myapp                  # coverage report (needs pytest-cov)
```

### Fixtures

A fixture is a reusable piece of **setup (and teardown)** that tests can request as an argument
— pytest handles the wiring automatically based on parameter names.

```python
import pytest

@pytest.fixture
def sample_user():
    return {"name": "Alice", "email": "alice@example.com"}

def test_user_has_email(sample_user):
    assert "email" in sample_user

# Fixtures with setup AND teardown, using yield
@pytest.fixture
def db_connection():
    conn = create_connection()     # setup — runs before the test
    yield conn                     # this is what gets injected into the test
    conn.close()                    # teardown — runs after the test, even if it failed
```

```
┌────────────┐    inject     ┌────────────┐    cleanup     ┌────────────┐
│  setup code   │ ────────────▶│  test runs    │ ────────────▶│  teardown code │
│  (before yield)│               │  (uses fixture)│               │  (after yield)  │
└────────────┘               └────────────┘               └────────────┘
```

**Fixture scope** controls how often it's recreated:

```python
@pytest.fixture(scope="function")  # default — fresh instance for EVERY test
@pytest.fixture(scope="module")     # created once, shared across all tests in one file
@pytest.fixture(scope="session")     # created once for the ENTIRE test run
```

### Mocking

Replaces a real dependency (an external API call, the current time, a slow/unreliable service)
with a fake, controllable stand-in — so your test is fast, deterministic, and doesn't actually
hit the network/third party.

```python
from unittest.mock import patch, MagicMock

def send_welcome_email(email):
    import requests
    requests.post("https://emailservice.com/send", json={"to": email})

def test_send_welcome_email_calls_api():
    with patch("requests.post") as mock_post:
        send_welcome_email("alice@example.com")
        mock_post.assert_called_once_with(
            "https://emailservice.com/send",
            json={"to": "alice@example.com"},
        )
    # no real HTTP request was ever made!

# pytest-mock plugin, cleaner syntax via the `mocker` fixture
def test_send_welcome_email_with_mocker(mocker):
    mock_post = mocker.patch("requests.post")
    send_welcome_email("bob@example.com")
    mock_post.assert_called_once()
```

```
Without mocking:                          With mocking:
Test ──▶ real requests.post() ──▶ REAL      Test ──▶ FAKE requests.post() ──▶ (no real
         network call to emailservice.com            network call — just records that
         (slow, flaky, costs money/quota)             it WOULD have been called, and how)
```

### Test Database

Never run tests against your real/production database — use a dedicated test DB that's created
fresh, populated, and torn down for each test (or test session) so tests are isolated and
repeatable.

```python
# conftest.py — shared fixtures available to all test files automatically
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

TEST_DATABASE_URL = "postgresql://test_user:test_pass@localhost/test_db"

@pytest.fixture(scope="session")
def engine():
    return create_engine(TEST_DATABASE_URL)

@pytest.fixture
def db_session(engine):
    Base.metadata.create_all(engine)          # create fresh tables
    Session = sessionmaker(bind=engine)
    session = Session()
    yield session
    session.rollback()                          # undo anything the test did
    session.close()
    Base.metadata.drop_all(engine)               # tear everything down — next test starts clean
```

```
┌────────────────────────────────────────────────────────┐
│  For EVERY test:                                           │
│  1. Create fresh tables    →  2. Run the test  →  3. Roll back│
│     (clean slate)                                    and drop  │
│                                                       tables      │
│  Guarantees no test's leftover data can affect ANY other test    │
└────────────────────────────────────────────────────────┘
```

An even faster common pattern: wrap each test in a DB **transaction that's rolled back** at the
end instead of recreating tables every time — same isolation guarantee, much faster.

---

## Real Test Examples for Common Backend Scenarios

### User Registration

```python
def test_register_user_success(client, db_session):
    response = client.post("/register", json={
        "email": "newuser@example.com",
        "password": "SecurePass123",
    })
    assert response.status_code == 201
    user = db_session.query(User).filter_by(email="newuser@example.com").first()
    assert user is not None
    assert user.password != "SecurePass123"   # must be hashed, never stored in plaintext!

def test_register_duplicate_email_fails(client, existing_user):
    response = client.post("/register", json={
        "email": existing_user.email,          # already taken
        "password": "AnotherPass123",
    })
    assert response.status_code == 400
    assert "already exists" in response.json()["detail"].lower()
```

### Login

```python
def test_login_success(client, existing_user):
    response = client.post("/login", json={
        "email": existing_user.email,
        "password": "correct_password",
    })
    assert response.status_code == 200
    assert "access_token" in response.json()

def test_login_wrong_password(client, existing_user):
    response = client.post("/login", json={
        "email": existing_user.email,
        "password": "wrong_password",
    })
    assert response.status_code == 401

def test_login_nonexistent_user(client):
    response = client.post("/login", json={
        "email": "ghost@example.com",
        "password": "whatever",
    })
    assert response.status_code == 401   # don't leak WHICH part was wrong (email vs password)
```

### Permission (Authentication/Authorization Testing)

```python
def test_admin_endpoint_rejects_regular_user(client, regular_user_token):
    response = client.get(
        "/admin/dashboard",
        headers={"Authorization": f"Bearer {regular_user_token}"},
    )
    assert response.status_code == 403

def test_admin_endpoint_allows_admin(client, admin_user_token):
    response = client.get(
        "/admin/dashboard",
        headers={"Authorization": f"Bearer {admin_user_token}"},
    )
    assert response.status_code == 200

def test_protected_endpoint_requires_auth(client):
    response = client.get("/profile")                # no Authorization header at all
    assert response.status_code == 401

def test_user_cannot_access_another_users_data(client, user_a_token, user_b):
    response = client.get(
        f"/users/{user_b.id}/private-notes",
        headers={"Authorization": f"Bearer {user_a_token}"},
    )
    assert response.status_code == 403   # authenticated, but not AUTHORIZED for this resource
```

### CRUD

```python
def test_create_product(client, admin_token):
    response = client.post(
        "/products",
        json={"name": "Widget", "price": 9.99},
        headers={"Authorization": f"Bearer {admin_token}"},
    )
    assert response.status_code == 201
    assert response.json()["name"] == "Widget"

def test_read_product(client, sample_product):
    response = client.get(f"/products/{sample_product.id}")
    assert response.status_code == 200
    assert response.json()["id"] == sample_product.id

def test_update_product(client, sample_product, admin_token):
    response = client.put(
        f"/products/{sample_product.id}",
        json={"price": 12.99},
        headers={"Authorization": f"Bearer {admin_token}"},
    )
    assert response.status_code == 200
    assert response.json()["price"] == 12.99

def test_delete_product(client, sample_product, admin_token):
    response = client.delete(
        f"/products/{sample_product.id}",
        headers={"Authorization": f"Bearer {admin_token}"},
    )
    assert response.status_code == 204
    assert client.get(f"/products/{sample_product.id}").status_code == 404
```

### Invalid Input

```python
def test_create_product_missing_required_field(client, admin_token):
    response = client.post(
        "/products",
        json={"price": 9.99},                       # missing "name"
        headers={"Authorization": f"Bearer {admin_token}"},
    )
    assert response.status_code == 422

def test_create_product_wrong_type(client, admin_token):
    response = client.post(
        "/products",
        json={"name": "Widget", "price": "not-a-number"},
        headers={"Authorization": f"Bearer {admin_token}"},
    )
    assert response.status_code == 422

@pytest.mark.parametrize("bad_email", [
    "not-an-email",
    "",
    "missing@domain",
    "@nodomain.com",
])
def test_register_rejects_invalid_email(client, bad_email):
    response = client.post("/register", json={"email": bad_email, "password": "Pass123"})
    assert response.status_code == 422
```

`@pytest.mark.parametrize` runs the same test logic against multiple inputs — great for
systematically covering a batch of invalid-input edge cases without repeating code.

### Database Failures

```python
def test_handles_database_connection_error(client, mocker):
    mocker.patch(
        "myapp.db.session.execute",
        side_effect=OperationalError("connection refused", None, None),
    )
    response = client.get("/users")
    assert response.status_code == 503        # graceful failure, not a raw 500 crash/traceback
    assert "temporarily unavailable" in response.json()["detail"].lower()

def test_handles_unique_constraint_violation(client, existing_user, mocker):
    mocker.patch(
        "myapp.db.session.commit",
        side_effect=IntegrityError("duplicate key", None, None),
    )
    response = client.post("/register", json={
        "email": existing_user.email,
        "password": "Pass123",
    })
    assert response.status_code == 400          # translated into a clean client error

def test_transaction_rolls_back_on_failure(db_session, mocker):
    mocker.patch.object(db_session, "commit", side_effect=Exception("boom"))
    with pytest.raises(Exception):
        create_order_with_payment(db_session, order_data, payment_data)
    # verify NEITHER the order NOR the payment was partially saved
    assert db_session.query(Order).count() == 0
    assert db_session.query(Payment).count() == 0
```

This last test demonstrates verifying **atomicity** — a multi-step operation should either fully
succeed or leave no partial trace, never a half-completed state.

---

## Putting It Together — A conftest.py Skeleton

```python
# conftest.py
import pytest
from fastapi.testclient import TestClient
from myapp.main import app
from myapp.database import Base, get_db, SessionLocal
from myapp.security import create_access_token

@pytest.fixture(scope="session")
def engine():
    return create_engine("postgresql://test:test@localhost/test_db")

@pytest.fixture
def db_session(engine):
    Base.metadata.create_all(engine)
    session = SessionLocal(bind=engine)
    yield session
    session.rollback()
    session.close()
    Base.metadata.drop_all(engine)

@pytest.fixture
def client(db_session):
    def override_get_db():
        yield db_session
    app.dependency_overrides[get_db] = override_get_db   # swap real DB for test DB
    yield TestClient(app)
    app.dependency_overrides.clear()

@pytest.fixture
def existing_user(db_session):
    user = User(email="existing@example.com", password=hash_password("correct_password"))
    db_session.add(user)
    db_session.commit()
    return user

@pytest.fixture
def admin_token(db_session):
    admin = User(email="admin@example.com", is_admin=True)
    db_session.add(admin)
    db_session.commit()
    return create_access_token({"sub": admin.email})
```

```
┌──────────────────────────────────────────────────────────────┐
│                        conftest.py                                │
│    (fixtures automatically available to EVERY test file)            │
│                                                                    │
│    engine → db_session → client → existing_user, admin_token        │
│    (each builds on the one before it, injected automatically)         │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
              test_registration.py, test_login.py,
              test_permissions.py, test_crud.py, ...
              (each just requests the fixtures it needs by name)
```

**The one-sentence takeaway:** good backend tests aren't about hitting 100% coverage for its own
sake — they're about proving that registration, auth, permissions, CRUD, bad input, and failure
modes all behave correctly and predictably, using fixtures for clean setup/teardown and mocking
to isolate your code from unreliable externals.
