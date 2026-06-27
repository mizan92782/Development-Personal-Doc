# Stripe Checkout Session — A Complete Backend Developer's Tutorial

> **Goal:** By the end of this tutorial, you will understand `stripe.checkout.Session.create()` not just as an API reference, but as an *architecture* — what each parameter does, what the user sees, what Stripe stores, and what comes back in the Webhook.

---

## Table of Contents

1. [What Is a Checkout Session?](#1-what-is-a-checkout-session)
2. [The Lifecycle of a Checkout Session](#2-the-lifecycle-of-a-checkout-session)
3. [The Full Structure of Session.create()](#3-the-full-structure-of-sessioncreate)
4. [Parameter Deep Dive](#4-parameter-deep-dive)
   - [payment_method_types](#41-payment_method_types)
   - [mode](#42-mode)
   - [line_items](#43-line_items)
   - [success_url & cancel_url](#44-success_url--cancel_url)
   - [metadata](#45-metadata)
   - [customer & customer_email](#46-customer--customer_email)
   - [shipping_address_collection](#47-shipping_address_collection)
   - [phone_number_collection](#48-phone_number_collection)
   - [billing_address_collection](#49-billing_address_collection)
   - [allow_promotion_codes & discounts](#410-allow_promotion_codes--discounts)
   - [automatic_tax](#411-automatic_tax)
   - [expires_at](#412-expires_at)
   - [locale](#413-locale)
   - [payment_intent_data](#414-payment_intent_data)
5. [What Stripe Creates for You](#5-what-stripe-creates-for-you)
6. [Mental Model: 4 Logical Sections](#6-mental-model-4-logical-sections)
7. [What Comes Next](#7-what-comes-next)

---

## 1. What Is a Checkout Session?

### The Common Misconception

Most beginners assume that calling:

```python
stripe.checkout.Session.create(...)
```

**processes a payment**. It does not.

### What It Actually Does

Calling `Session.create()` tells Stripe to **build a hosted payment page** for a specific user, with specific products, and specific rules. Stripe returns a URL. Your user visits that URL and *then* pays.

Think of it like placing a restaurant order for your customer — you choose the table, the menu, and the rules. The customer still has to show up and eat.

Here is the conversation that happens between Django and Stripe:

**You (Django):**
> "Hey Stripe — create a payment page for this user. Show them this product at this price. If they pay, send them here. If they cancel, send them there. Oh, and remember this Order ID."

**Stripe:**
> "Done. Here's the URL: `https://checkout.stripe.com/pay/cs_test_xxx`"

The user then visits that URL, enters their card details, and pays — all on Stripe's hosted page.

---

## 2. The Lifecycle of a Checkout Session

```
Your Django Backend
       │
       │  stripe.checkout.Session.create(...)
       ▼
    Stripe API
       │
       │  Creates a Session (cs_test_xxxxx)
       │  Returns a Checkout URL
       ▼
    User's Browser
       │
       │  User opens the URL
       │  User enters card details
       │  Payment is processed
       ▼
    Stripe
       │
       │  Fires a Webhook event
       │  (checkout.session.completed)
       ▼
    Your Django Backend
       │
       │  Receives the event
       │  Fulfills the order
       ▼
    Session Expires / Ends
```

**Key insight:** A Checkout Session is a **temporary context** for a payment. Once the payment is complete (or the session expires), it is done.

---

## 3. The Full Structure of Session.create()

Below is the full shape of the function with all commonly used parameters. You will almost never use all of them at once — pick what fits your use case.

```python
import stripe

session = stripe.checkout.Session.create(
    # --- Payment Configuration ---
    payment_method_types=["card"],
    mode="payment",
    automatic_tax={"enabled": True},

    # --- Product Configuration ---
    line_items=[
        {
            "price_data": {
                "currency": "usd",
                "unit_amount": 1000,
                "product_data": {
                    "name": "MacBook",
                    "description": "Apple M4 Pro",
                    "images": ["https://example.com/macbook.jpg"],
                },
            },
            "quantity": 2,
        }
    ],

    # --- User Experience Configuration ---
    success_url="https://yoursite.com/payment-success",
    cancel_url="https://yoursite.com/payment-cancel",
    customer_email="user@example.com",
    shipping_address_collection={"allowed_countries": ["US", "CA"]},
    phone_number_collection={"enabled": True},
    billing_address_collection="required",
    allow_promotion_codes=True,
    locale="auto",
    expires_at=1700000000,  # Unix timestamp

    # --- Backend Configuration ---
    metadata={"order_id": 501, "user_id": 5},
    customer="cus_xxxxxxxxx",
    payment_intent_data={},
)

print(session.url)  # Redirect the user to this URL
```

---

## 4. Parameter Deep Dive

### 4.1 `payment_method_types`

```python
payment_method_types=["card"]
```

**Purpose:** Tells Stripe which payment methods to show the user.

| Value | What it enables |
|---|---|
| `"card"` | Visa, Mastercard, Amex, etc. |
| `"ideal"` | Dutch bank transfers (iDEAL) |
| `"bancontact"` | Belgian payment method |
| `"sepa_debit"` | European bank debits |

**What the user sees (with only `"card"`):**

```
┌─────────────────────────────┐
│  Card Number                │
│  [__________________]       │
│  MM / YY    CVC             │
│  [______]  [____]           │
│  Name on Card               │
│  [__________________]       │
└─────────────────────────────┘
```

Adding more methods adds tabs or additional payment options above the card form, depending on the user's country and your Stripe account settings.

---

### 4.2 `mode`

```python
mode="payment"   # or "subscription" or "setup"
```

This is the **purpose** of the checkout.

| Mode | What happens | Real-world example |
|---|---|---|
| `"payment"` | One-time charge | Buying a laptop |
| `"subscription"` | Recurring charge | Netflix, Spotify |
| `"setup"` | Save card, no charge | Uber (card saved before first ride) |

- In `"subscription"` mode, Stripe charges the card automatically on a schedule (monthly, yearly, etc.). The user enters their card once, and Stripe does the rest.
- In `"setup"` mode, no money is collected — Stripe simply stores the card for future use.

---

### 4.3 `line_items`

```python
line_items=[
    {
        "price_data": { ... },
        "quantity": 2,
    }
]
```

This is the **heart of the Checkout Page** — it defines what products the user is buying.

#### `price_data`

Describes the price of a single product.

```python
"price_data": {
    "currency": "usd",       # which currency
    "unit_amount": 1000,     # price per unit (in smallest unit)
    "product_data": {
        "name": "MacBook",
        "description": "Apple M4 Pro",
        "images": ["https://example.com/macbook.jpg"],
    },
}
```

**`currency`** — The ISO currency code.

```python
"currency": "usd"   # displays as $
"currency": "eur"   # displays as €
"currency": "bdt"   # displays as ৳
```

**`unit_amount`** — The price **in the smallest currency unit**. Stripe never uses decimals.

| Currency | 10 USD / 10 EUR / 10 GBP | Stripe value |
|---|---|---|
| USD | 10 dollars | `1000` (cents) |
| EUR | 10 euros | `1000` (cents) |
| GBP | 10 pounds | `1000` (pence) |
| JPY | 10 yen | `10` (yen, no subdivision) |

> ✅ The user always sees the correct formatted amount on the page (`$10.00`). The raw number (`1000`) stays internal.

**`product_data`** — What the user sees for the product.

```python
"product_data": {
    "name": "MacBook",                              # required
    "description": "Apple M4 Pro",                 # optional
    "images": ["https://example.com/image.jpg"],   # optional, shows product image
}
```

**`quantity`** — How many units.

```
MacBook
$1,000.00 × 2

──────────────
Total: $2,000.00
```

#### Multiple Products

Pass multiple objects in the list:

```python
line_items=[
    {"price_data": {...laptop...}, "quantity": 2},
    {"price_data": {...mouse...},  "quantity": 3},
    {"price_data": {...keyboard...}, "quantity": 1},
]
```

Stripe will display each item with its subtotal and compute the grand total automatically.

---

### 4.4 `success_url` & `cancel_url`

```python
success_url="https://yoursite.com/payment-success",
cancel_url="https://yoursite.com/payment-cancel",
```

**`success_url`** — Where the browser goes after a successful payment.

Typically you display here:
- ✅ Payment Successful
- Order summary
- "Continue Shopping" button

**`cancel_url`** — Where the browser goes if the user clicks the back/cancel button on the Stripe page.

Typically you display here:
- ❌ Payment Cancelled
- "Try Again" button

> **Pro tip:** You can append Stripe template variables to `success_url` to receive the session ID:
> ```python
> success_url="https://yoursite.com/success?session_id={CHECKOUT_SESSION_ID}"
> ```
> This lets you retrieve the session on your success page to confirm details.

---

### 4.5 `metadata`

```python
metadata={
    "order_id": "501",
    "user_id": "5",
}
```

**This is one of the most important parameters for your backend.**

Metadata is:
- **Never shown to the user** — it is purely internal
- **Stored by Stripe** in the session object
- **Returned in the Webhook** when `checkout.session.completed` fires

This is how you connect a Stripe payment to your own database records. Without metadata, when Stripe tells you "a payment was completed," you have no idea *which order* it was for.

**In your Webhook handler:**

```python
@csrf_exempt
def stripe_webhook(request):
    event = stripe.Webhook.construct_event(...)

    if event["type"] == "checkout.session.completed":
        session = event["data"]["object"]
        order_id = session["metadata"]["order_id"]   # ← your data, back again
        user_id  = session["metadata"]["user_id"]

        # Fulfill the order in your database
        Order.objects.filter(id=order_id).update(status="paid")
```

---

### 4.6 `customer` & `customer_email`

**`customer_email`** — Pre-fills the email field on the Checkout Page.

```python
customer_email="alice@example.com"
```

The user still sees the field but does not have to type their email.

**`customer`** — If the user already exists as a Stripe Customer, pass their Stripe Customer ID.

```python
customer="cus_xxxxxxxxx"
```

Stripe will then use:
- Their saved email
- Their saved billing address
- Their previously used cards (if applicable)

Use `customer` when you have a returning user. Use `customer_email` when the user is new or you only have their email.

---

### 4.7 `shipping_address_collection`

```python
shipping_address_collection={
    "allowed_countries": ["US", "CA", "GB"]
}
```

Adds a shipping address form to the Checkout Page:

```
Shipping Address
────────────────
Name         [____________]
Country      [US ▼        ]
Address      [____________]
City         [____________]
State        [____________]
ZIP / Postal [____________]
```

Only countries in `allowed_countries` appear in the dropdown. This protects you from accepting orders you cannot ship to.

After payment, the shipping address is available in the session object returned by the Webhook.

---

### 4.8 `phone_number_collection`

```python
phone_number_collection={"enabled": True}
```

Adds a phone number field to the Checkout Page. Useful for delivery notifications or SMS receipts.

---

### 4.9 `billing_address_collection`

```python
billing_address_collection="required"   # or "auto"
```

| Value | Behavior |
|---|---|
| `"auto"` (default) | Stripe decides based on card type |
| `"required"` | Always shows billing address form |

Use `"required"` when you need the billing address for tax compliance or fraud prevention.

---

### 4.10 `allow_promotion_codes` & `discounts`

**Option A — Let the user enter a code:**

```python
allow_promotion_codes=True
```

Adds a promotion code box to the Checkout Page:

```
Promotion Code
[____________________] [Apply]
```

The user types a coupon code you created in your Stripe Dashboard.

**Option B — Apply a discount from the backend:**

```python
discounts=[{"coupon": "SAVE20"}]
```

The discount is applied silently — the user sees the reduced price but does not enter a code. Useful for loyalty discounts or referral programs.

> You cannot use both `allow_promotion_codes=True` and `discounts` at the same time.

---

### 4.11 `automatic_tax`

```python
automatic_tax={"enabled": True}
```

Stripe calculates and adds tax automatically, based on the user's location and the type of product. Requires Stripe Tax to be configured in your Dashboard.

Supported for many countries and US states. Where supported, you do not need to calculate tax yourself.

---

### 4.12 `expires_at`

```python
import time
expires_at = int(time.time()) + (30 * 60)  # 30 minutes from now

stripe.checkout.Session.create(
    ...,
    expires_at=expires_at,
)
```

By default, Stripe Checkout Sessions expire after **24 hours**. Use `expires_at` to set a shorter window.

After expiry, the URL returns an error page. This is useful for:
- Flash sales (time-limited pricing)
- Preventing session hoarding
- Security (reducing the window for link sharing)

---

### 4.13 `locale`

```python
locale="fr"   # French
locale="de"   # German
locale="auto" # Detect from browser
```

Controls the language of the Checkout Page. Stripe supports 30+ locales. `"auto"` detects the user's browser language and uses it automatically — a good default for international sites.

---

### 4.14 `payment_intent_data`

```python
payment_intent_data={
    "capture_method": "manual",         # Authorize now, capture later
    "statement_descriptor": "MYSTORE",  # What appears on the bank statement
    "metadata": {"internal_ref": "abc"} # Additional metadata on the PaymentIntent
}
```

Stripe creates a PaymentIntent behind the scenes when you use `mode="payment"`. This parameter lets you configure that PaymentIntent before it is created.

Common uses:

| Option | Purpose |
|---|---|
| `capture_method: "manual"` | Authorize the card now, charge it later (e.g., after shipping) |
| `statement_descriptor` | Control what the customer sees on their bank statement |
| `metadata` | Extra data attached to the PaymentIntent (separate from session metadata) |

---

## 5. What Stripe Creates for You

When you call `Session.create()`, Stripe builds and stores a complete session object internally. Conceptually, it looks like this:

```
Checkout Session
──────────────────────────────────────────
Session ID      cs_test_a1b2c3d4e5f6
Mode            payment
Status          open

Products
  ├── MacBook  ×2  @  $1,000.00  =  $2,000.00
  └── Mouse    ×1  @  $20.00     =  $20.00

Currency        USD
Total           $2,020.00

Customer Email  alice@example.com
Metadata        order_id=501, user_id=5

Success URL     https://yoursite.com/payment-success
Cancel URL      https://yoursite.com/payment-cancel

Expires At      2024-11-15 14:30:00 UTC
──────────────────────────────────────────
Checkout URL    https://checkout.stripe.com/pay/cs_test_a1b2c3d4e5f6
```

Your job is simply to redirect the user to that URL.

---

## 6. Mental Model: 4 Logical Sections

Instead of memorizing a flat list of parameters, group them into four categories:

```
stripe.checkout.Session.create(

    # ── 1. PAYMENT CONFIGURATION ────────────────────────────────────
    # HOW will the payment work?
    mode="payment",
    payment_method_types=["card"],
    automatic_tax={"enabled": True},

    # ── 2. PRODUCT CONFIGURATION ────────────────────────────────────
    # WHAT is being sold?
    line_items=[
        {
            "price_data": {
                "currency": "usd",
                "unit_amount": 9900,
                "product_data": {"name": "MacBook"},
            },
            "quantity": 1,
        }
    ],

    # ── 3. USER EXPERIENCE CONFIGURATION ────────────────────────────
    # WHAT does the user see and interact with?
    success_url="https://yoursite.com/success",
    cancel_url="https://yoursite.com/cancel",
    customer_email="alice@example.com",
    shipping_address_collection={"allowed_countries": ["US"]},
    allow_promotion_codes=True,
    locale="auto",

    # ── 4. BACKEND CONFIGURATION ────────────────────────────────────
    # HOW does your system track this payment?
    metadata={"order_id": "501", "user_id": "5"},
    customer="cus_xxxxxxxxx",
    expires_at=1700000000,
    payment_intent_data={},
)
```

Mastering these four sections gives you **80–90% of Stripe Checkout knowledge**. The remaining 10–20% is edge cases, advanced tax rules, and platform-specific features.

---

## 7. What Comes Next

Once you understand `Session.create()`, the natural next steps are:

| Topic | What you will learn |
|---|---|
| **Webhooks** | How to receive `checkout.session.completed` and fulfill orders |
| **PaymentIntent** | The underlying payment object Stripe creates inside each session |
| **Customers API** | How to create and manage Stripe Customers for returning users |
| **Subscriptions** | Using `mode="subscription"` with recurring `price_data` |
| **Refunds** | How to refund a completed PaymentIntent |
| **Stripe Dashboard** | How to view sessions, payments, and customers visually |

> **Recommended learning order:**
> `Checkout Session` → `Webhooks` → `PaymentIntent` → `Customers` → `Subscriptions` → `Refunds`

---

*Happy building! Stripe's official documentation is at [stripe.com/docs](https://stripe.com/docs) — once you have this mental model, the docs will make much more sense.*
