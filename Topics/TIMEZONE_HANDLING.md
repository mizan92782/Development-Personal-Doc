# Timezone Handling for Multi-Country Users

## The Problem

When users from different countries use the app, their actions happen at different local times.
Without proper timezone handling, everyone sees UTC time which is confusing.

```
User from USA    → action at 10:00 PM local time → DB stores 03:00 UTC (next day!)
User from BD     → action at 10:00 PM local time → DB stores 16:00 UTC
User from UK     → action at 10:00 PM local time → DB stores 21:00 UTC
```

---

## The Solution: Store UTC, Show Local

**Golden Rule:**
```
DB always stores → UTC
API always returns → User's local time
```

This is the industry standard used by Google, Facebook, Twitter, etc.

---

## How It Actually Works

### Step 1 — Frontend Auto-Detects Timezone

Every browser/device knows its own timezone. JavaScript can read it:

```js
Intl.DateTimeFormat().resolvedOptions().timeZone
// USA user    → "America/New_York"
// BD user     → "Asia/Dhaka"
// UK user     → "Europe/London"
// Japan user  → "Asia/Tokyo"
```

Frontend sends this in every API request as a header:

```js
axios.defaults.headers['X-Timezone'] = Intl.DateTimeFormat().resolvedOptions().timeZone;
```

---

### Step 2 — Middleware Catches the Timezone

Every API request passes through `TimezoneMiddleware` before reaching the view.

**File:** `pro_utilities/timezone_middleware.py`

```python
import pytz
from rest_framework_simplejwt.authentication import JWTAuthentication


class TimezoneMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        tz_name = request.headers.get('X-Timezone')
        if tz_name:
            try:
                pytz.timezone(tz_name)  # validate timezone name
                try:
                    result = JWTAuthentication().authenticate(request)
                    if result:
                        user, _ = result
                        profile = getattr(user, 'profile', None)
                        if profile and profile.timezone != tz_name:
                            profile.timezone = tz_name
                            profile.save(update_fields=['timezone'])
                except Exception:
                    pass
            except pytz.UnknownTimeZoneError:
                pass
        return self.get_response(request)
```

**Why JWTAuthentication() directly?**

Django's `AuthenticationMiddleware` only handles session-based auth.
JWT tokens are decoded inside DRF views — so `request.user` is `AnonymousUser` at middleware level.
We manually call `JWTAuthentication().authenticate(request)` to decode the JWT and get the real user.

**What happens for different users:**

```
Anonymous user  → result is None → skips silently ✅
Logged-in user  → saves timezone to profile ✅
Invalid token   → exception caught → skips silently ✅
No X-Timezone header → skips entirely ✅
```

---

### Step 3 — Timezone Saved in UserProfile

**File:** `authentication/model/profile_mod.py`

```python
class UserProfile(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE, related_name="profile")
    first_name = models.CharField(max_length=255, blank=True, null=True)
    last_name = models.CharField(max_length=255, blank=True, null=True)
    dp_image = models.ImageField(upload_to='tenant_dp/', blank=True, null=True)
    phone_number = PhoneNumberField(blank=True, null=True)
    timezone = models.CharField(max_length=50, default='UTC', blank=True)  # ← auto-saved
    create_at = models.DateTimeField(auto_now_add=True)
```

- Default is `UTC` for new users
- Auto-updates every time user makes a request with `X-Timezone` header
- User never sees or manually sets this field — fully automatic

---

### Step 4 — Timezone Conversion Utility

**File:** `pro_utilities/timezone_utils.py`

```python
import pytz


def convert_to_user_timezone(dt, user):
    """Convert a UTC datetime to the user's local timezone."""
    if dt is None:
        return None
    try:
        profile = user.profile
        tz = pytz.timezone(profile.timezone or 'UTC')
    except Exception:
        tz = pytz.UTC
    return dt.astimezone(tz).isoformat()


def get_user_tz(user):
    """Get pytz timezone object for a user."""
    try:
        tz_name = user.profile.timezone or 'UTC'
        return pytz.timezone(tz_name)
    except Exception:
        return pytz.UTC
```

---

### Step 5 — Serializers Convert Datetime Fields

Any serializer that has a datetime field uses `convert_to_user_timezone`:

**Example — Profile Serializer:**

```python
from pro_utilities.timezone_utils import convert_to_user_timezone

class UserProfileSerializer(serializers.ModelSerializer):
    create_at = serializers.SerializerMethodField()

    def get_create_at(self, obj):
        return convert_to_user_timezone(obj.create_at, obj.user)
```

**Example — Learning Session Serializer:**

```python
class LearningSessionSerializer(serializers.ModelSerializer):
    started_at = serializers.SerializerMethodField()
    completed_at = serializers.SerializerMethodField()

    def get_started_at(self, obj):
        return convert_to_user_timezone(obj.started_at, obj.user)

    def get_completed_at(self, obj):
        return convert_to_user_timezone(obj.completed_at, obj.user)
```

---

## Full Request Flow

```
1. User in USA opens app
        ↓
2. Frontend reads timezone → "America/New_York"
        ↓
3. Frontend sends every request with header:
   X-Timezone: America/New_York
        ↓
4. TimezoneMiddleware intercepts request
        ↓
5. Middleware decodes JWT → gets user
        ↓
6. Saves "America/New_York" to user.profile.timezone in DB
        ↓
7. Request reaches the view/serializer
        ↓
8. Serializer calls convert_to_user_timezone(obj.started_at, obj.user)
        ↓
9. UTC time in DB → converted to "America/New_York"
        ↓
10. API response returns local time to user
    "started_at": "2025-01-15T05:00:00-05:00"  ← New York time ✅
```

---

## Real Example

**Same action, 3 different users:**

| User | Location | DB stores (UTC) | API returns |
|------|----------|-----------------|-------------|
| Alice | New York | 2025-01-15 10:00 UTC | 2025-01-15T05:00:00-05:00 |
| Rahim | Dhaka | 2025-01-15 10:00 UTC | 2025-01-15T16:00:00+06:00 |
| John | London | 2025-01-15 10:00 UTC | 2025-01-15T10:00:00+00:00 |

All 3 see their own correct local time. DB has one clean UTC value.

---

## Middleware Registration

**File:** `lifechoice/settings.py`

```python
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "whitenoise.middleware.WhiteNoiseMiddleware",
    "corsheaders.middleware.CorsMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "pro_utilities.timezone_middleware.TimezoneMiddleware",  # ← here
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
]
```

Also `x-timezone` must be in CORS allowed headers:

```python
CORS_ALLOW_HEADERS = [
    "authorization",
    "content-type",
    ...
    "x-timezone",  # ← required
]
```

---

## Frontend Implementation

### Axios Setup (one time only)

```js
// src/api/axios.js

import axios from 'axios';

const api = axios.create({
  baseURL: 'https://api.ikonskills.ac',
});

// Auto-detect and attach timezone to every request
api.defaults.headers['X-Timezone'] = Intl.DateTimeFormat().resolvedOptions().timeZone;

// Attach JWT token
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('access_token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export default api;
```

### Display Time to User (optional)

API already returns local time as ISO string. Frontend can display it directly:

```js
// API returns: "2025-01-15T05:00:00-05:00"

const date = new Date("2025-01-15T05:00:00-05:00");
console.log(date.toLocaleString());
// → "1/15/2025, 5:00:00 AM"  (shown in user's local format)
```

---

## What NOT to Do

| ❌ Wrong | ✅ Correct |
|----------|-----------|
| Store local time in DB | Store UTC in DB |
| Convert time before saving | Save raw, convert on read |
| Ask user to select timezone manually | Auto-detect from browser |
| Hardcode timezone in backend | Read from user profile |
| Return UTC to frontend | Return converted local time |

---

## Files Changed in This Project

| File | Change |
|------|--------|
| `authentication/model/profile_mod.py` | Added `timezone` field |
| `authentication/migrations/0014_userprofile_timezone.py` | Migration for timezone field |
| `pro_utilities/timezone_middleware.py` | New — auto-captures timezone |
| `pro_utilities/timezone_utils.py` | New — reusable conversion utility |
| `authentication/serializer/profile_ser.py` | `create_at` now returns local time |
| `learning/serializer/learning_ser.py` | `started_at`, `completed_at`, `assessed_at`, `timestamp` return local time |
| `lifechoice/settings.py` | Middleware registered, `x-timezone` in CORS headers |
