# 💳 Stripe Payment System — সম্পূর্ণ গাইড (বাংলা)

> এই ডকুমেন্টে Ikon Skills এর Stripe payment system কীভাবে কাজ করে সেটা Backend + Frontend মিলিয়ে ধাপে ধাপে বাংলায় বোঝানো হয়েছে।

---

## 📌 সিস্টেমে কোন কোন Part আছে

| Part | কী করে |
|------|--------|
| **Frontend (React)** | User এর card নেয়, Stripe কে দেয়, API call করে |
| **Stripe.js** | Card নম্বর directly Stripe server এ পাঠায় — Backend কখনো দেখে না |
| **Backend (Django)** | PaymentIntent তৈরি করে, enrollment save করে |
| **Stripe Server** | টাকা process করে, webhook event পাঠায় |
| **Webhook** | Stripe থেকে আসা event handle করে, DB update করে |

---

## 🗺️ সম্পূর্ণ Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         USER (Browser)                              │
│                                                                     │
│   1. Micro-Credential page এ আসে                                   │
│   2. "Buy Now" বাটনে ক্লিক করে                                     │
│   3. Card details দেয় (CardElement এ)                              │
└─────────────────────┬───────────────────────────────────────────────┘
                      │
                      │ card details (number, expiry, cvv)
                      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      STRIPE.JS (Frontend Library)                   │
│                                                                     │
│   stripe.createPaymentMethod({ type: 'card', card: CardElement })  │
│                                                                     │
│   ⚠️  Card number কখনো তোমার Backend এ যায় না!                    │
│   Stripe নিজেই card নেয় এবং একটা token দেয়                        │
│                                                                     │
│   Return করে: payment_method_id = "pm_1ABC123..."                  │
└─────────────────────┬───────────────────────────────────────────────┘
                      │
                      │ payment_method_id শুধু পাঠায়
                      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    BACKEND API (Django)                             │
│                                                                     │
│   POST /enrollment/enrollments/                                     │
│   Body: {                                                           │
│     "micro_credential_id": 5,                                       │
│     "payment_method_id": "pm_1ABC123..."                            │
│   }                                                                 │
│                                                                     │
│   ধাপ ১: DB থেকে Stripe secret key নাও                             │
│   ধাপ ২: Micro-credential exist করে কিনা চেক করো                  │
│   ধাপ ৩: User আগে কিনেছে কিনা চেক করো (race condition safe)       │
│   ধাপ ৪: Current price নাও DB থেকে (যেমন $14.99)                  │
│   ধাপ ৫: Stripe এ PaymentIntent তৈরি করো                           │
└─────────────────────┬───────────────────────────────────────────────┘
                      │
                      │ PaymentIntent create request
                      │ amount: 1499 cents ($14.99)
                      │ payment_method: "pm_1ABC123..."
                      │ confirm: True
                      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      STRIPE SERVER                                  │
│                                                                     │
│   Card টা valid কিনা চেক করে                                       │
│   Bank এ charge করে                                                 │
│                                                                     │
│   ৩টা result হতে পারে:                                             │
│                                                                     │
│   ✅ succeeded        → payment হয়ে গেছে                           │
│   🔐 requires_action  → 3D Secure / OTP দরকার (Live mode)          │
│   ❌ failed           → card decline / insufficient funds           │
└──────┬──────────────────────┬──────────────────────┬───────────────┘
       │                      │                      │
       │ succeeded            │ requires_action      │ failed
       ▼                      ▼                      ▼
┌────────────┐     ┌──────────────────┐    ┌──────────────────┐
│  Backend   │     │    Frontend      │    │    Backend       │
│            │     │                 │    │                  │
│ DB তে save │     │ 3DS popup দেখাও │    │ 400 error return │
│ enrollment │     │ (Stripe handle) │    │ "Payment failed" │
│ করো        │     │                 │    │                  │
│            │     │ User OTP দেয়    │    └────────┬─────────┘
│ 201 return │     │                 │             │
└─────┬──────┘     │ ✅ সফল হলে      │             │
      │            │ → success page  │             │
      │            │                 │             │
      │            │ ❌ ব্যর্থ হলে   │             │
      │            │ → failed page   │             ▼
      │            └────────┬────────┘    ┌──────────────────┐
      │                     │             │    Frontend      │
      │                     │             │  failed page দেখাও│
      │                     │             └──────────────────┘
      ▼                     ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    STRIPE WEBHOOK                                   │
│                                                                     │
│   Stripe আলাদাভাবে তোমার server এ event পাঠায়                     │
│   URL: https://api.ikonskills.ac/stripe/webhook                    │
│                                                                     │
│   Events:                                                           │
│   • payment_intent.succeeded    → DB status = completed            │
│   • payment_intent.failed       → DB status = failed               │
│   • charge.succeeded            → PaymentTransaction create        │
│   • charge.failed               → PaymentTransaction failed        │
│                                                                     │
│   ⚠️  Webhook signature verify করে — fake request ঠেকায়           │
└─────────────────────────────────────────────────────────────────────┘
                      │
                      │ সব শেষে
                      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      EMAIL NOTIFICATION                             │
│                                                                     │
│   👤 User কে পাঠায়:  "Purchase Confirmed" email                    │
│   👨‍💼 Admin কে পাঠায়: "New Enrollment Alert" email                 │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 📊 টাকার Flow ($14.99 এর উদাহরণ)

```
User এর Card
     │
     │ $14.99 charge হয়
     ▼
Stripe এর Account (intermediate)
     │
     │ Stripe তার fee রাখে (~2.9% + $0.30 = ~$0.73)
     │
     │ বাকি $14.26 তোমার Stripe account এ জমা হয়
     ▼
Client এর Stripe Balance
     │
     │ প্রতি সপ্তাহে/মাসে Bank account এ transfer
     ▼
Client এর Bank Account
```

---

## 🔐 3D Secure (3DS) কী এবং কেন হয়

```
Live mode এ কিছু card এ Bank এর verification লাগে

User card দেয়
      ↓
Stripe → Bank কে জিজ্ঞেস করে "এই card এ 3DS আছে?"
      ↓
      ├── না → সরাসরি payment ✅
      │
      └── হ্যাঁ → requires_action status আসে
                        ↓
                  Frontend এ Stripe popup আসে
                        ↓
                  User এর Phone এ Bank এর OTP আসে
                        ↓
                  User OTP দেয়
                        ↓
                  ├── সঠিক → payment.succeeded ✅
                  └── ভুল  → payment.failed ❌
```

---

## 🔗 Webhook কী এবং কেন দরকার

Webhook হলো Stripe এর পাঠানো **automatic notification**।

```
সমস্যা যদি Webhook না থাকতো:
─────────────────────────────
User payment করলো
      ↓
Backend "succeeded" দেখলো
      ↓
DB তে save করলো
      ↓
কিন্তু পরে Stripe বললো "আসলে card decline হয়েছে"
      ↓
User access পেয়ে গেছে কিন্তু টাকা আসেনি! ❌


Webhook থাকলে:
──────────────
Stripe সরাসরি তোমার server এ event পাঠায়
      ↓
Backend DB update করে সঠিক status দিয়ে ✅
      ↓
সব সময় accurate থাকে
```

### Webhook Security কীভাবে কাজ করে

```
Stripe event পাঠায়
      ↓
Header এ Stripe-Signature থাকে
      ↓
Backend: stripe.Webhook.construct_event(payload, signature, webhook_secret)
      ↓
      ├── Signature match → valid event ✅ → process করো
      └── Signature mismatch → fake request ❌ → 400 return
```

---

## 💻 Backend Code Flow (Django)

### ধাপ ১ — Payment API (`POST /enrollment/enrollments/`)

```python
# enrollment/view/enrollment_view.py

def create(self, request):

    # ১. Input নাও
    micro_credential_id = request.data.get('micro_credential_id')
    payment_method_id   = request.data.get('payment_method_id')  # Stripe থেকে আসা token

    # ২. DB থেকে Stripe key নাও (admin panel থেকে change করা যায়)
    stripe_config    = get_active_stripe_config()
    stripe.api_key   = stripe_config.get('STRIPE_SECRET_KEY')

    # ৩. Already enrolled কিনা চেক করো (race condition prevent)
    with db_transaction.atomic():
        existing = MCAccess.objects.select_for_update().filter(
            user=request.user,
            micro_credential=micro_credential,
            is_active=True
        ).first()
        if existing:
            return error("Already enrolled")

    # ৪. Price নাও DB থেকে
    current_price = MicroCredentialPricing.get_current_price()  # যেমন 14.99
    amount_cents  = int(current_price * 100)                    # 1499

    # ৫. Stripe এ PaymentIntent তৈরি করো
    payment_intent = stripe.PaymentIntent.create(
        amount          = amount_cents,       # 1499 cents
        currency        = 'usd',
        payment_method  = payment_method_id,  # Frontend থেকে আসা
        confirm         = True,               # সাথে সাথে charge করো
        metadata        = {
            'user_id'             : request.user.id,
            'micro_credential_id' : micro_credential_id,
        }
    )

    # ৬. Result handle করো
    if payment_intent.status == 'succeeded':
        # DB তে save করো + email পাঠাও
        return success(201)

    elif payment_intent.status == 'requires_action':
        # Frontend কে client_secret দাও 3DS এর জন্য
        return response(202, client_secret=payment_intent.client_secret)

    else:
        return error(400, "Payment failed")
```

### ধাপ ২ — Webhook (`POST /stripe/webhook`)

```python
# enrollment/webhooks/stripe_webhook.py

def stripe_webhook(request):

    # ১. Stripe config DB থেকে নাও
    stripe_config  = get_active_stripe_config()
    stripe.api_key = stripe_config.get('STRIPE_SECRET_KEY')
    webhook_secret = stripe_config.get('STRIPE_WEBHOOK_SECRET')

    # ২. Signature verify করো (security)
    event = stripe.Webhook.construct_event(
        payload    = request.body,
        sig_header = request.META.get('HTTP_STRIPE_SIGNATURE'),
        secret     = webhook_secret
    )

    # ৩. Event type অনুযায়ী কাজ করো
    if event.type == 'payment_intent.succeeded':
        # DB তে status = 'completed' করো

    elif event.type == 'payment_intent.payment_failed':
        # DB তে status = 'failed' করো

    elif event.type == 'charge.succeeded':
        # PaymentTransaction record তৈরি করো

    return HttpResponse(200)  # Stripe কে জানাও "পেয়েছি"
```

---

## 💻 Frontend Code Flow (React)

### ধাপ ১ — Stripe Setup (`App.jsx`)

```jsx
import { loadStripe } from '@stripe/stripe-js';
import { Elements } from '@stripe/react-stripe-js';

// Backend API থেকে publishable key নাও
// GET /stripe-config/stripe-config/public-key/
const stripePromise = loadStripe('pk_live_51Snikf...');

function App() {
    return (
        <Elements stripe={stripePromise}>
            <YourRoutes />
        </Elements>
    );
}
```

### ধাপ ২ — Payment Form (`PaymentForm.jsx`)

```jsx
import { useStripe, useElements, CardElement } from '@stripe/react-stripe-js';

export default function PaymentForm({ microCredentialId }) {
    const stripe   = useStripe();
    const elements = useElements();
    const navigate = useNavigate();

    const handleSubmit = async (e) => {
        e.preventDefault();

        // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
        // STEP 1: Stripe থেকে payment_method_id নাও
        // Card number কখনো Backend এ যাবে না
        // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
        const { error, paymentMethod } = await stripe.createPaymentMethod({
            type : 'card',
            card : elements.getElement(CardElement),
        });

        if (error) {
            // Card invalid → error দেখাও
            setError(error.message);
            return;
        }

        // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
        // STEP 2: Backend API call করো
        // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
        const response = await fetch('/enrollment/enrollments/', {
            method  : 'POST',
            headers : {
                'Content-Type'  : 'application/json',
                'Authorization' : `Bearer ${token}`,
            },
            body: JSON.stringify({
                micro_credential_id : microCredentialId,
                payment_method_id   : paymentMethod.id,  // "pm_1ABC..."
            }),
        });

        const data = await response.json();

        // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
        // STEP 3: Response অনুযায়ী কাজ করো
        // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

        if (response.status === 201) {
            // ✅ সরাসরি সফল
            navigate('/payment/success');

        } else if (response.status === 202) {
            // 🔐 3DS দরকার — Stripe popup দেখাবে
            const { error: confirmError, paymentIntent } =
                await stripe.confirmCardPayment(data.data.client_secret);

            if (confirmError) {
                // OTP ভুল বা cancel
                navigate('/payment/failed', {
                    state: { message: confirmError.message }
                });
            } else if (paymentIntent.status === 'succeeded') {
                // OTP সফল
                navigate('/payment/success');
            }

        } else {
            // ❌ Payment failed
            setError(data.message);
        }
    };

    return (
        <form onSubmit={handleSubmit}>
            <CardElement />
            <button type="submit">Pay $14.99</button>
        </form>
    );
}
```

### ধাপ ৩ — Success ও Failed Page

```jsx
// PaymentSuccess.jsx
export default function PaymentSuccess() {
    return (
        <div>
            <h1>✅ Payment Successful!</h1>
            <p>আপনার Micro-Credential কেনা হয়েছে। শেখা শুরু করুন!</p>
            <a href="/dashboard">Dashboard এ যান</a>
        </div>
    );
}

// PaymentFailed.jsx
export default function PaymentFailed() {
    const { state } = useLocation();
    return (
        <div>
            <h1>❌ Payment Failed</h1>
            <p>{state?.message || 'Payment সফল হয়নি। আবার চেষ্টা করুন।'}</p>
            <button onClick={() => navigate(-1)}>আবার চেষ্টা করুন</button>
        </div>
    );
}
```

---

## 📧 Email Notification Flow

```
Payment Succeeded (status 201)
            │
            ├──→ 👤 User এর email এ পাঠায়:
            │       Subject: "🎉 Purchase Confirmed: [MC Name]"
            │       Content: MC নাম, domain, price, transaction ID,
            │                competency count, purchase time
            │
            └──→ 👨‍💼 Admin এর email এ পাঠায়:
                    Subject: "Ikon Practitioner Enrolled a Micro-Credential"
                    Content: User নাম, email, MC নাম, price, transaction ID
```

---

## ⚡ সম্পূর্ণ Sequence — এক নজরে

```
FRONTEND                    BACKEND                     STRIPE
   │                           │                           │
   │── CardElement এ card দেয় ─┤                           │
   │                           │                           │
   │── createPaymentMethod() ──────────────────────────────▶│
   │◀─────────── pm_1ABC... ───────────────────────────────┤
   │                           │                           │
   │── POST /enrollments/ ────▶│                           │
   │   {mc_id, pm_1ABC...}     │                           │
   │                           │── PaymentIntent.create() ▶│
   │                           │◀─ {status, client_secret} ┤
   │                           │                           │
   │                    ┌──────┴──────┐                    │
   │                    │   status?   │                    │
   │                    └──┬──┬──┬───┘                    │
   │                       │  │  │                        │
   │         succeeded ────┘  │  └──── requires_action    │
   │                          │                           │
   │◀── 201 enrollment data ──┘        │                  │
   │                                   │                  │
   │◀─────────── 202 client_secret ────┘                  │
   │                                                      │
   │── confirmCardPayment(client_secret) ────────────────▶│
   │                      [3DS Popup দেখায়]               │
   │◀─────────── succeeded/failed ────────────────────────┤
   │                                                      │
   │                           │◀── Webhook event ────────┤
   │                           │    payment_intent.succeeded
   │                           │── DB update ─────────────┤
   │                           │── Email send ─────────────┤
   │                           │                           │
   ▼                           ▼                           ▼
Success/Failed Page      DB Updated               Stripe Dashboard
```

---

## 🔑 Key Summary — মনে রাখার বিষয়

| বিষয় | কারণ |
|-------|------|
| Card number Backend এ যায় না | Stripe.js সরাসরি Stripe server এ পাঠায় — PCI compliance |
| `pk_live_` শুধু Frontend এ | Public key — expose করা safe |
| `sk_live_` শুধু Backend এ | Secret key — কখনো Frontend এ না |
| `whsec_` শুধু Backend এ | Webhook verify করতে লাগে |
| Webhook আলাদাভাবে আসে | Payment এর পরেও Stripe নিজে event পাঠায় DB accurate রাখতে |
| 202 মানে শেষ না | 3DS pending — Frontend কে `confirmCardPayment()` করতে হবে |
| DB থেকে key নেওয়া | Admin panel থেকে key change করলে restart ছাড়াই কাজ করে |

---

## 🌐 API Endpoints Summary

| Method | URL | কাজ | Auth |
|--------|-----|-----|------|
| `GET` | `/enrollment/enrollments/pricing/` | Current price দেখো | Public |
| `POST` | `/enrollment/enrollments/` | MC কেনো | Login লাগবে |
| `GET` | `/enrollment/enrollments/` | আমার সব enrollment | Login লাগবে |
| `POST` | `/stripe/webhook` | Stripe event receive | Stripe only |
| `GET` | `/stripe-config/stripe-config/public-key/` | Publishable key নাও | Public |

---

*এই document টি Ikon Skills payment system এর internal reference হিসেবে তৈরি করা হয়েছে।*
