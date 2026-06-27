# Stripe Webhook — সম্পূর্ণ বাংলা টিউটোরিয়াল

> **লক্ষ্য:** Stripe Webhook কীভাবে কাজ করে, তোমার কোডের প্রতিটি লাইন কী করছে, কেন দরকার, এবং `event` object-এ আর কী কী থাকে — সব বিস্তারিত বুঝবে।

---

## সূচিপত্র

1. [Webhook আসলে কী?](#1-webhook-আসলে-কী)
2. [Webhook ছাড়া কী সমস্যা হয়?](#2-webhook-ছাড়া-কী-সমস্যা-হয়)
3. [Stripe Webhook কীভাবে কাজ করে?](#3-stripe-webhook-কীভাবে-কাজ-করে)
4. [তোমার কোড — লাইন বাই লাইন](#4-তোমার-কোড--লাইন-বাই-লাইন)
5. [event object-এর সম্পূর্ণ গঠন](#5-event-object-এর-সম্পূর্ণ-গঠন)
6. [event\['type'\] — সব ধরনের Event](#6-eventtype--সব-ধরনের-event)
7. [session object-এর সব Attribute](#7-session-object-এর-সব-attribute)
8. [তোমার কোডের Bug — টাইপো ধরা](#8-তোমার-কোডের-bug--টাইপো-ধরা)
9. [সম্পূর্ণ Production-Ready Webhook](#9-সম্পূর্ণ-production-ready-webhook)
10. [Django-তে Setup করার ধাপ](#10-django-তে-setup-করার-ধাপ)

---

## 1. Webhook আসলে কী?

### সাধারণ ভাষায়

ধরো তুমি একটি Pizza Order করলে।

দোকান তোমাকে বলল না যে Pizza কখন Ready হবে। তুমি প্রতি ৫ মিনিটে ফোন করে জিজ্ঞেস করছো —

> "Pizza Ready হয়েছে?"
> "এখনো না।"
> "এখন?"
> "না।"

এটাকে বলে **Polling** — খুব অকার্যকর।

এর বদলে যদি দোকান **নিজে তোমাকে** SMS পাঠায় যখন Pizza Ready হয় — এটাই **Webhook**।

### Technical ভাষায়

```
তুমি Stripe-কে বলো:
"যখনই কোনো Payment Complete হবে,
আমার এই URL-এ একটি POST Request পাঠাও।"

URL: https://yoursite.com/webhook/stripe/
```

Stripe সেই URL মনে রাখে। Payment হলে সাথে সাথে তোমার Server-এ Data পাঠায়।

---

## 2. Webhook ছাড়া কী সমস্যা হয়?

### একটি বাস্তব সমস্যা দেখো

```
User Payment করলো  ✓
Stripe টাকা পেলো  ✓

কিন্তু তোমার Database আপডেট হলো না  ✗
Order এখনো "pending" আছে  ✗
User আবার Payment করার চেষ্টা করছে  ✗
```

`success_url`-এ Redirect হলে তো Order আপডেট করা যায় — মনে হতে পারে।

কিন্তু তিনটি সমস্যা আছে:

| সমস্যা | ব্যাখ্যা |
|---|---|
| **Browser বন্ধ** | User Payment-এর পর Browser বন্ধ করে দিতে পারে। `success_url`-এ যাওয়াই হলো না। |
| **Network Error** | Redirect-এর সময় Internet চলে গেল। |
| **Security** | `success_url` যে কেউ সরাসরি Visit করতে পারে — Payment ছাড়াই। |

Webhook এই সব সমস্যা সমাধান করে। Stripe সরাসরি তোমার Server-এ আসে — User-এর Browser-এর উপর নির্ভর করে না।

---

## 3. Stripe Webhook কীভাবে কাজ করে?

```
User Card দিলো
       │
       ▼
Stripe Payment Process করলো
       │
       ├──► User-কে success_url-এ Redirect করলো
       │
       └──► তোমার Webhook URL-এ POST Request পাঠালো
                    │
                    ▼
            Django তোমার View-তে Request পেলো
                    │
                    ▼
            Signature Verify করলো (আসল Stripe কিনা?)
                    │
                    ▼
            event['type'] চেক করলো
                    │
                    ▼
            Database আপডেট করলো
                    │
                    ▼
            HTTP 200 Return করলো
                    │
                    ▼
            Stripe জানলো — "ঠিকমতো পেয়েছে"
```

---

## 4. তোমার কোড — লাইন বাই লাইন

```python
def productWebhook(request):
```

এটি একটি সাধারণ Django View। Stripe এই View-তে POST Request পাঠাবে।

> **গুরুত্বপূর্ণ:** এই View-তে `@csrf_exempt` Decorator লাগাতে হবে, না হলে Django সব Request Block করে দেবে। (নিচে বিস্তারিত আছে।)

---

```python
payload = request.body
```

### `payload` কী?

Stripe যে Raw Data পাঠিয়েছে — সেটাই Payload।

এটি **bytes** format-এ আসে, JSON নয়।

উদাহরণ হিসেবে এটি দেখতে এমন:

```
b'{"id":"evt_xxx","object":"event","type":"checkout.session.completed","data":{...}}'
```

**কেন `request.body` ব্যবহার করতে হয়?**

Signature Verification-এ Stripe ঠিক এই Raw Bytes ব্যবহার করে। যদি তুমি `request.POST` বা `json.loads(request.body)` আগেই করো, Signature মিলবে না।

---

```python
signature = request.META['HTTP_STRIPE_SIGNATURE']
```

### `signature` কী?

Stripe প্রতিটি Request-এর সাথে একটি বিশেষ Header পাঠায়:

```
Stripe-Signature: t=1700000000,v1=abc123def456...
```

Django-তে সব HTTP Header `request.META`-তে `HTTP_` prefix দিয়ে থাকে।

তাই `Stripe-Signature` → `HTTP_STRIPE_SIGNATURE`

### কেন Signature দরকার?

ধরো কোনো হ্যাকার তোমার Webhook URL খুঁজে পেলো।

সে নিজে নিজে একটি Request পাঠালো:

```json
{
  "type": "checkout.session.completed",
  "data": { "object": { "metadata": { "order_id": "1000" } } }
}
```

তোমার Server এটি সত্যি Stripe ভেবে Order `paid` করে দিলো!

Signature থাকায় এটি সম্ভব নয়। Signature তোমার `STRIPE_WEBHOOK_SECRET` দিয়ে তৈরি — হ্যাকারের কাছে সেটি নেই।

---

```python
try:
    event = stripe.Webhook.construct_event(
        payload,
        signature,
        settings.STRIPE_WEBHOOK_SECRET
    )
except Exception:
    return HttpResponse(status=400)
```

### `stripe.Webhook.construct_event()` কী করে?

এই একটি Function তিনটি কাজ একসাথে করে:

| কাজ | বিবরণ |
|---|---|
| **Signature Verify** | এটি আসলেই Stripe থেকে এসেছে কিনা নিশ্চিত করে |
| **Replay Attack প্রতিরোধ** | পুরনো Request পুনরায় পাঠানো আটকায় (timestamp চেক) |
| **Parse** | Raw bytes কে Python dict-এ রূপান্তর করে |

#### তিনটি Parameter:

**`payload`** — Raw request body (bytes)।
Stripe এই exact bytes দিয়ে Signature তৈরি করেছিল, তাই unmodified রাখতে হবে।

**`signature`** — `HTTP_STRIPE_SIGNATURE` header-এর value।
Stripe পাঠানো cryptographic signature।

**`settings.STRIPE_WEBHOOK_SECRET`** — তোমার `.env` বা `settings.py`-তে রাখা Secret।
Stripe Dashboard থেকে পাওয়া যায়। এটি দিয়ে Signature যাচাই হয়।

#### Exception হলে কী হয়?

```python
except Exception:
    return HttpResponse(status=400)
```

যদি Signature মিলে না — মানে Request Stripe থেকে আসেনি — `400 Bad Request` Return করো।

Stripe এটি দেখে বুঝবে কিছু একটা ভুল হয়েছে।

> **ভালো Practice:** শুধু `Exception` না ধরে নির্দিষ্ট Exception ধরা উচিত:
> ```python
> except stripe.error.SignatureVerificationError:
>     return HttpResponse(status=400)
> except ValueError:
>     return HttpResponse(status=400)
> ```

---

```python
if event['type'] == 'checkout.session.completed':
```

### `event['type']` কী?

Stripe অনেক ধরনের Event পাঠাতে পারে। এই line দিয়ে তুমি শুধু Payment Complete-এর Event ধরছো।

(সব Event-এর তালিকা Section 6-এ আছে।)

---

```python
session = event['data']['object']
```

### এই Structure কেন?

`event` object সবসময় এই Shape-এ আসে:

```python
{
    "id": "evt_xxx",
    "type": "checkout.session.completed",
    "data": {
        "object": {
            # এখানে আসল Session Data থাকে
            "id": "cs_test_xxx",
            "metadata": { "order_id": "501" },
            ...
        }
    }
}
```

তাই `event['data']['object']` মানে — Checkout Session object নিজে।

---

```python
metadata = session['metadata']
order_id = metadata["order_id"]
```

### `metadata` কোথা থেকে এলো?

তুমি `Session.create()`-এ যা দিয়েছিলে:

```python
# Session তৈরির সময়
stripe.checkout.Session.create(
    ...
    metadata={
        "order_id": "501",
        "user_id": "5",
    }
)
```

Stripe এটি সংরক্ষণ করে রেখেছিল। Payment Complete হলে হুবহু ফেরত দেয়।

---

```python
order = Order.objects.get(id=order_id)
order.payment_status = "paid"
order.save()
```

এখানে তুমি Database-এ Order খুঁজে বের করে `paid` করে দিচ্ছো।

---

```python
return HttpResponse(status=200)
```

### কেন `200` Return করতেই হবে?

Stripe যদি `200` না পায়, সে মনে করে Webhook Delivery ব্যর্থ হয়েছে।

তখন Stripe **পুনরায় পাঠানোর চেষ্টা করে** — ৩ দিন ধরে, বারবার।

| Response | Stripe কী করে |
|---|---|
| `200` | ✅ সফল — আর পাঠাবে না |
| `400` | ❌ ব্যর্থ — Retry করবে না (তোমার ভুল) |
| `500` | ❌ ব্যর্থ — Retry করবে (Server-এর সমস্যা) |
| Timeout | ❌ ব্যর্থ — Retry করবে |

---

## 5. event object-এর সম্পূর্ণ গঠন

Stripe যা পাঠায় তার পুরো structure:

```python
event = {
    # ── Event-এর নিজের তথ্য ────────────────────────────────────
    "id": "evt_1ABC2DEF3GHI",        # এই Event-এর Unique ID
    "object": "event",               # সবসময় "event"
    "type": "checkout.session.completed",  # কী ঘটেছে
    "created": 1700000000,           # Unix Timestamp
    "livemode": False,               # True = Production, False = Test
    "api_version": "2023-10-16",     # কোন API Version ব্যবহার হয়েছে
    "request": {
        "id": "req_xxx",             # কোন API Request এটি Trigger করেছে
        "idempotency_key": None,
    },
    "pending_webhooks": 1,           # আরও কতটি Webhook পাঠানো বাকি

    # ── আসল Data ────────────────────────────────────────────────
    "data": {
        "object": {
            # এখানে Checkout Session object থাকে
            # (Section 7-এ বিস্তারিত)
        },
        "previous_attributes": {
            # যদি কোনো Object Update হয়, আগের value এখানে থাকে
            # checkout.session.completed-এ এটি সাধারণত খালি
        }
    }
}
```

### প্রতিটি Top-Level Field:

| Field | Type | কেন দরকার |
|---|---|---|
| `id` | string | প্রতিটি Event Unique। Duplicate চেক করতে ব্যবহার করো। |
| `object` | string | সবসময় `"event"` — এটি একটি Event object বলে নিশ্চিত করে। |
| `type` | string | কী ঘটেছে তা জানায়। তুমি এটি দিয়ে `if` করো। |
| `created` | integer | কখন হয়েছে (Unix timestamp)। Log-এ রাখতে কাজে লাগে। |
| `livemode` | boolean | `True` মানে Real Money। `False` মানে Test। |
| `api_version` | string | কোন Stripe API Version ব্যবহার হয়েছে। |
| `pending_webhooks` | integer | কতটি Webhook Endpoint এখনো এই Event পায়নি। |
| `data.object` | dict | আসল Checkout Session, Payment, Customer — যা ঘটেছে তার Object। |
| `data.previous_attributes` | dict | Update Event-এ আগের মান দেখায়। |

---

## 6. `event['type']` — সব ধরনের Event

Stripe ডজনেরও বেশি Event পাঠাতে পারে। সবচেয়ে গুরুত্বপূর্ণগুলো:

### Checkout Session Events

```python
"checkout.session.completed"      # Payment সফল — সবচেয়ে বেশি ব্যবহার হয়
"checkout.session.expired"        # Session Expire হয়ে গেছে (User Pay করেনি)
"checkout.session.async_payment_succeeded"  # Delayed payment সফল (bank transfer)
"checkout.session.async_payment_failed"     # Delayed payment ব্যর্থ
```

### Payment Events

```python
"payment_intent.succeeded"        # Payment সম্পন্ন
"payment_intent.payment_failed"   # Payment ব্যর্থ
"payment_intent.canceled"         # Payment বাতিল
"payment_intent.created"          # নতুন Payment তৈরি
"payment_intent.requires_action"  # 3D Secure বা অন্য কিছু দরকার
```

### Refund Events

```python
"charge.refunded"                 # Refund হয়েছে
"charge.refund.updated"           # Refund Update হয়েছে
```

### Subscription Events

```python
"customer.subscription.created"   # নতুন Subscription
"customer.subscription.updated"   # Subscription পরিবর্তন হয়েছে
"customer.subscription.deleted"   # Subscription বাতিল হয়েছে
"invoice.paid"                    # মাসিক Invoice পরিশোধ হয়েছে
"invoice.payment_failed"          # মাসিক Invoice পরিশোধ ব্যর্থ
```

### Customer Events

```python
"customer.created"                # নতুন Customer
"customer.updated"                # Customer তথ্য পরিবর্তন
"customer.deleted"                # Customer মুছে ফেলা হয়েছে
```

### একাধিক Event Handle করার Pattern:

```python
def productWebhook(request):
    # ... payload, signature, construct_event ...

    event_type = event['type']

    if event_type == 'checkout.session.completed':
        handle_checkout_completed(event['data']['object'])

    elif event_type == 'checkout.session.expired':
        handle_checkout_expired(event['data']['object'])

    elif event_type == 'invoice.paid':
        handle_invoice_paid(event['data']['object'])

    elif event_type == 'customer.subscription.deleted':
        handle_subscription_cancelled(event['data']['object'])

    return HttpResponse(status=200)
```

---

## 7. session object-এর সব Attribute

`event['data']['object']` মানে Checkout Session। এর ভিতরে অনেক তথ্য থাকে:

```python
session = {
    # ── পরিচয় ───────────────────────────────────────────────────
    "id": "cs_test_a1b2c3",          # Session-এর Unique ID
    "object": "checkout.session",    # Object type

    # ── Status ──────────────────────────────────────────────────
    "status": "complete",            # open | complete | expired
    "payment_status": "paid",        # paid | unpaid | no_payment_required

    # ── Payment তথ্য ─────────────────────────────────────────────
    "payment_intent": "pi_xxx",      # এই Session-এর PaymentIntent ID
    "amount_total": 2000,            # Total amount (Cents-এ) — $20.00
    "amount_subtotal": 2000,         # Tax বাদে amount
    "currency": "usd",               # কোন Currency
    "payment_method_types": ["card"],# কী দিয়ে Pay করা হয়েছে

    # ── Customer তথ্য ────────────────────────────────────────────
    "customer": "cus_xxx",           # Stripe Customer ID (থাকলে)
    "customer_email": "a@b.com",     # Customer-এর Email
    "customer_details": {
        "email": "a@b.com",
        "name": "Alice",
        "phone": "+1234567890",
        "address": {
            "city": "New York",
            "country": "US",
            "line1": "123 Main St",
            "line2": None,
            "postal_code": "10001",
            "state": "NY",
        },
        "tax_exempt": "none",
        "tax_ids": [],
    },

    # ── Shipping তথ্য ────────────────────────────────────────────
    "shipping_details": {
        "name": "Alice",
        "address": {
            "city": "New York",
            "country": "US",
            "line1": "123 Main St",
            "line2": None,
            "postal_code": "10001",
            "state": "NY",
        },
    },
    "shipping_cost": {
        "amount_total": 500,          # Shipping cost (cents)
        "shipping_rate": "shr_xxx",
    },

    # ── তোমার Custom Data ─────────────────────────────────────────
    "metadata": {
        "order_id": "501",            # তুমি যা দিয়েছিলে
        "user_id": "5",
    },

    # ── URL ──────────────────────────────────────────────────────
    "url": "https://checkout.stripe.com/pay/cs_test_xxx",  # Checkout URL
    "success_url": "https://yoursite.com/success",
    "cancel_url": "https://yoursite.com/cancel",

    # ── Tax তথ্য ─────────────────────────────────────────────────
    "total_details": {
        "amount_discount": 0,         # Discount amount
        "amount_shipping": 500,       # Shipping amount
        "amount_tax": 180,            # Tax amount
    },
    "automatic_tax": {
        "enabled": True,
        "status": "complete",
    },

    # ── Promotion / Discount ──────────────────────────────────────
    "discounts": [],                  # Apply হওয়া Discount-গুলো
    "allow_promotion_codes": True,

    # ── অন্যান্য ─────────────────────────────────────────────────
    "mode": "payment",               # payment | subscription | setup
    "livemode": False,               # Test mode কিনা
    "created": 1700000000,           # Session তৈরির সময়
    "expires_at": 1700086400,        # Session Expire-এর সময়
    "locale": "auto",
    "line_items": None,              # সাধারণত Webhook-এ Expand করতে হয়
    "subscription": None,            # Subscription mode-এ Subscription ID
}
```

### সবচেয়ে বেশি ব্যবহৃত Attribute:

```python
session = event['data']['object']

# Order পূরণ করতে
order_id       = session['metadata']['order_id']
payment_status = session['payment_status']    # "paid"
amount_total   = session['amount_total']      # 2000 (cents)
currency       = session['currency']          # "usd"

# Customer তথ্য সংরক্ষণ করতে
customer_email = session['customer_email']
customer_name  = session['customer_details']['name']
customer_phone = session['customer_details']['phone']

# Shipping Address সংরক্ষণ করতে
shipping = session['shipping_details']['address']
city     = shipping['city']
country  = shipping['country']

# Stripe-এর নিজস্ব ID গুলো
session_id     = session['id']               # cs_test_xxx
payment_intent = session['payment_intent']   # pi_xxx (Refund-এ লাগবে)
```

---

## 8. তোমার কোডের Bug — টাইপো ধরা

তোমার কোডে একটি গুরুত্বপূর্ণ ভুল আছে:

```python
# ❌ ভুল — "heckout" লেখা আছে, "c" নেই
if event['type'] == 'heckout.session.completed':
```

```python
# ✅ সঠিক
if event['type'] == 'checkout.session.completed':
```

এই ভুলের কারণে কোনো Event কখনোই match হবে না। Order কখনো `paid` হবে না। কিন্তু Django কোনো Error-ও দেবে না — Silent Bug।

এছাড়া আরও একটি ছোট সমস্যা:

```python
# ❌ দুইবার একই কাজ করছো
metadata = session['metadata']
metadata = session["metadata"]   # এই line অপ্রয়োজনীয়
```

---

## 9. সম্পূর্ণ Production-Ready Webhook

```python
import stripe
import logging
from django.http import HttpResponse
from django.views.decorators.csrf import csrf_exempt
from django.views.decorators.http import require_POST
from django.conf import settings
from .models import Order

logger = logging.getLogger(__name__)


@csrf_exempt          # Stripe-এর Request-এ CSRF Token নেই, তাই Exempt করতে হবে
@require_POST         # শুধু POST Request Accept করবে
def stripe_webhook(request):

    # ── Step 1: Raw Data ও Signature নাও ─────────────────────────────
    payload   = request.body
    signature = request.META.get('HTTP_STRIPE_SIGNATURE')

    if not signature:
        logger.warning("Webhook received without Stripe-Signature header")
        return HttpResponse(status=400)

    # ── Step 2: Stripe-এর পরিচয় যাচাই করো ──────────────────────────
    try:
        event = stripe.Webhook.construct_event(
            payload,
            signature,
            settings.STRIPE_WEBHOOK_SECRET
        )
    except stripe.error.SignatureVerificationError:
        logger.error("Invalid Stripe signature — possible fake request")
        return HttpResponse(status=400)
    except ValueError:
        logger.error("Invalid payload received")
        return HttpResponse(status=400)

    # ── Step 3: Event Type অনুযায়ী কাজ করো ──────────────────────────
    event_type = event['type']
    logger.info(f"Stripe Webhook received: {event_type}")

    if event_type == 'checkout.session.completed':
        handle_checkout_completed(event['data']['object'])

    elif event_type == 'checkout.session.expired':
        handle_checkout_expired(event['data']['object'])

    elif event_type == 'payment_intent.payment_failed':
        handle_payment_failed(event['data']['object'])

    # ── Step 4: সবসময় 200 Return করো ────────────────────────────────
    return HttpResponse(status=200)


def handle_checkout_completed(session):
    """Payment সফল হলে Order আপডেট করো।"""

    # Metadata থেকে তোমার Data নাও
    metadata = session.get('metadata', {})
    order_id = metadata.get('order_id')

    if not order_id:
        logger.error(f"No order_id in metadata for session {session['id']}")
        return

    # Duplicate Event থেকে রক্ষা পাও
    # (Stripe একই Event একাধিকবার পাঠাতে পারে)
    try:
        order = Order.objects.get(id=order_id)
    except Order.DoesNotExist:
        logger.error(f"Order {order_id} not found")
        return

    if order.payment_status == 'paid':
        logger.info(f"Order {order_id} already paid — skipping duplicate webhook")
        return

    # Order Update করো
    order.payment_status  = 'paid'
    order.stripe_session  = session['id']
    order.payment_intent  = session.get('payment_intent')
    order.save()

    logger.info(f"Order {order_id} marked as paid")


def handle_checkout_expired(session):
    """Session Expire হলে Order Cancel করো।"""
    metadata = session.get('metadata', {})
    order_id = metadata.get('order_id')

    if order_id:
        Order.objects.filter(id=order_id, payment_status='pending').update(
            payment_status='cancelled'
        )
        logger.info(f"Order {order_id} marked as cancelled (session expired)")


def handle_payment_failed(payment_intent):
    """Payment ব্যর্থ হলে Log করো।"""
    logger.warning(f"Payment failed: {payment_intent['id']}")
    # এখানে User-কে Email পাঠাতে পারো
```

---

## 10. Django-তে Setup করার ধাপ

### ধাপ ১ — settings.py

```python
# settings.py

STRIPE_SECRET_KEY      = "sk_test_xxxxxxxxxx"
STRIPE_PUBLISHABLE_KEY = "pk_test_xxxxxxxxxx"
STRIPE_WEBHOOK_SECRET  = "whsec_xxxxxxxxxx"   # Stripe Dashboard থেকে পাবে
```

### ধাপ ২ — urls.py

```python
# urls.py

from django.urls import path
from . import views

urlpatterns = [
    path('webhook/stripe/', views.stripe_webhook, name='stripe-webhook'),
]
```

### ধাপ ৩ — Stripe Dashboard-এ Webhook Register করো

```
Stripe Dashboard
  → Developers
  → Webhooks
  → Add Endpoint
  → URL: https://yoursite.com/webhook/stripe/
  → Events:
      ✓ checkout.session.completed
      ✓ checkout.session.expired
      ✓ payment_intent.payment_failed
  → Add Endpoint
  → Signing Secret কপি করে settings.py-তে রাখো
```

### ধাপ ৪ — Local Testing (Stripe CLI)

Production-এ যাওয়ার আগে Local-এ Test করো:

```bash
# Stripe CLI Install করো
brew install stripe/stripe-cli/stripe

# Login করো
stripe login

# Webhook Forward করো তোমার Local Server-এ
stripe listen --forward-to localhost:8000/webhook/stripe/

# আরেকটি Terminal-এ Test Event পাঠাও
stripe trigger checkout.session.completed
```

---

## সংক্ষেপে মনে রাখার বিষয়

| বিষয় | মনে রাখো |
|---|---|
| `payload` | Raw bytes — কখনো আগেই Parse করবে না |
| `signature` | `HTTP_STRIPE_SIGNATURE` Header — ছাড়া Request Reject |
| `construct_event()` | Verify + Parse একসাথে করে |
| `event['type']` | কী ঘটেছে তা বলে |
| `event['data']['object']` | আসল Session / Payment Object |
| `metadata` | তোমার Order ID এখানেই থাকে |
| `200` Return | Stripe-কে জানাও — "পেয়েছি" |
| `@csrf_exempt` | না দিলে সব Request Block হবে |
| Duplicate Check | একই Event দুইবার আসতে পারে — চেক করো |

---

*Webhook বুঝলে Stripe-এর বাকি সব — Refund, Subscription, PaymentIntent — অনেক সহজ লাগবে।*
