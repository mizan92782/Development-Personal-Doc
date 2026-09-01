# LEVEL 14 — Nginx / Reverse Proxy Deep Dive

## The Architecture

```
                    ┌───────────┐
                    │  Internet    │
                    └─────┬─────┘
                          │  HTTPS request to yourapp.com
                          ▼
                    ┌───────────┐
                    │   :443       │   ← Nginx listens here, publicly exposed
                    └─────┬─────┘
                          ▼
                    ┌───────────┐
                    │   Nginx       │   ← reverse proxy: terminates SSL, adds headers,
                    │               │      serves static files, then forwards the request
                    └─────┬─────┘
                          │  plain HTTP, internal only
                          ▼
                    ┌───────────┐
                    │   :8000       │   ← Gunicorn listens here, NOT exposed to the internet
                    └─────┬─────┘
                          ▼
                    ┌───────────┐
                    │  Gunicorn     │   ← WSGI server, manages worker processes
                    └─────┬─────┘
                          ▼
                    ┌───────────┐
                    │  Django       │   ← your actual application code
                    └───────────┘
```

**Why not just expose Gunicorn directly to the internet on :443?** Gunicorn (and most app
servers) are deliberately simple — they're good at running your application code, but bad at the
things a real internet-facing server needs to handle safely and efficiently: TLS termination,
serving static files without touching Python, buffering slow clients, rate limiting, and load
balancing across multiple app instances. Nginx is purpose-built and highly optimized for exactly
those jobs, so the standard pattern is: **Nginx handles the internet-facing, network-level
concerns; your app server handles business logic only.**

---

## 1. Reverse Proxy

A **forward proxy** sits in front of clients (hiding who's making requests, e.g. a corporate
proxy). A **reverse proxy** sits in front of servers (hiding how many/what servers exist,
presenting one unified front to the internet).

```
Forward proxy:                             Reverse proxy:
┌────────┐                                ┌────────┐
│ Client   │──▶ Proxy ──▶ Internet             │ Client   │──▶ Internet
└────────┘   (hides the client)            └────────┘
                                                    │
                                                    ▼
                                              ┌────────┐
                                              │  Nginx    │  (hides the backend servers)
                                              └───┬────┘
                                        ┌──────────┼──────────┐
                                        ▼          ▼          ▼
                                    Server A   Server B   Server C
```

```nginx
# /etc/nginx/sites-available/myapp
server {
    listen 80;
    server_name yourapp.com;

    location / {
        proxy_pass http://127.0.0.1:8000;   # forward the request to Gunicorn
    }
}
```

From the client's perspective, they only ever talk to Nginx — they have no idea (and don't need
to) that Gunicorn, Django, or multiple backend servers exist behind it.

---

## 2. SSL/TLS Termination

"Termination" means Nginx is the one that decrypts incoming HTTPS traffic and encrypts outgoing
responses — your app server never has to deal with certificates or encryption at all, it just
speaks plain HTTP internally.

```
┌────────┐   HTTPS (encrypted)   ┌───────┐   HTTP (plain)   ┌───────────┐
│ Client   │ ─────────────────────▶│ Nginx    │ ────────────────▶│ Gunicorn     │
│          │ ◀─────────────────────│ (decrypts/ │ ◀────────────────│ (never sees   │
└────────┘                        │  encrypts) │                  │  TLS at all)    │
                                   └───────┘                  └───────────┘
```

```nginx
server {
    listen 443 ssl;
    server_name yourapp.com;

    ssl_certificate     /etc/letsencrypt/live/yourapp.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourapp.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8000;   # plain HTTP from here on — internal, trusted network
    }
}
```

**Why centralize TLS at Nginx instead of in each app server?** One place to manage certificates,
renewals, and cipher configuration — instead of configuring TLS separately in every backend
service, especially valuable once you have multiple backend servers behind a load balancer.

---

## 3. HTTPS & 4. HTTP → HTTPS Redirect

Plain HTTP traffic should never reach your app in production — anyone on the network path could
read or tamper with it. The standard setup: listen on port 80 *only* to immediately redirect to
443.

```nginx
server {
    listen 80;
    server_name yourapp.com;
    return 301 https://$host$request_uri;    # permanent redirect to HTTPS, same path
}

server {
    listen 443 ssl;
    server_name yourapp.com;
    ssl_certificate     /etc/letsencrypt/live/yourapp.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourapp.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8000;
    }
}
```

```
Client requests http://yourapp.com/login
          │
          ▼
   Nginx :80  ──301 redirect──▶  Client re-requests https://yourapp.com/login
                                            │
                                            ▼
                                    Nginx :443 (actually serves it)
```

---

## 5. Proxy Headers

Once Nginx sits in front of your app, from Django/FastAPI's point of view, **every single
request appears to come from Nginx itself** (127.0.0.1) — the original client's real IP,
original protocol (https), and original host would be lost unless Nginx explicitly forwards them
via headers.

```nginx
location / {
    proxy_pass http://127.0.0.1:8000;

    proxy_set_header Host              $host;               # preserve the original Host header
    proxy_set_header X-Real-IP         $remote_addr;         # the ACTUAL client's IP
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;  # chain of proxy IPs
    proxy_set_header X-Forwarded-Proto $scheme;              # was the original request http or https?
}
```

```
Without these headers:                     With these headers:
request.META["REMOTE_ADDR"]                 request.META["REMOTE_ADDR"] = 127.0.0.1 (Nginx)
  = 127.0.0.1  (WRONG! that's Nginx,          request.META["HTTP_X_REAL_IP"] = 203.0.113.42 (correct!)
   not the real visitor)
```

```python
# Django settings.py — tell Django to trust and use these forwarded headers
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")
USE_X_FORWARDED_HOST = True
```

**Why this matters:** rate limiting, logging, geo-blocking, and "is this request secure" checks
all silently break if your app thinks every request is coming from `127.0.0.1` over plain HTTP.

---

## 6. Static Files

Serving a JS/CSS/image file through Django/Python is wasteful — the Python process spins up just
to read a file off disk and stream it back, when Nginx (written in C, purpose-built for this) can
do it far faster and without touching your app server at all.

```nginx
server {
    location /static/ {
        alias /opt/myapp/staticfiles/;      # Nginx serves these directly — Django never runs
    }

    location /media/ {
        alias /opt/myapp/media/;
    }

    location / {
        proxy_pass http://127.0.0.1:8000;    # only DYNAMIC requests go to your app
    }
}
```

```
Request for /static/style.css  ──▶  Nginx reads it straight off disk  ──▶  response
                                     (Gunicorn/Django never even wake up)

Request for /api/users          ──▶  Nginx forwards to Gunicorn  ──▶  Django  ──▶  response
```

---

## 7. Load Balancing

When one app server instance isn't enough, run several and let Nginx distribute incoming
requests across them — this is what lets you scale horizontally.

```nginx
upstream myapp_backend {
    server 127.0.0.1:8001;
    server 127.0.0.1:8002;
    server 127.0.0.1:8003;
}

server {
    listen 443 ssl;
    location / {
        proxy_pass http://myapp_backend;    # Nginx picks one of the 3 servers per request
    }
}
```

```
                         ┌──────────────┐
                         │     Nginx       │
                         └───────┬──────┘
                    round-robin distribution
             ┌───────────────┼───────────────┐
             ▼               ▼               ▼
     ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
     │ Gunicorn :8001 │ │ Gunicorn :8002 │ │ Gunicorn :8003 │
     └─────────────┘ └─────────────┘ └─────────────┘
```

Default algorithm is round-robin; other strategies exist:

```nginx
upstream myapp_backend {
    least_conn;                # send to whichever server has the FEWEST active connections
    server 127.0.0.1:8001;
    server 127.0.0.1:8002 weight=3;   # gets 3x the traffic of the others (e.g. a bigger machine)
    server 127.0.0.1:8003 backup;      # only used if the others are all down
}
```

---

## 8. Timeouts

Without timeouts, a single slow/hung upstream server (or a slow client) can tie up Nginx's
resources indefinitely, degrading service for everyone else.

```nginx
location / {
    proxy_pass http://127.0.0.1:8000;

    proxy_connect_timeout 5s;    # max time to establish a connection to the backend
    proxy_send_timeout    30s;   # max time to send the request to the backend
    proxy_read_timeout    30s;   # max time to wait for the backend's response
}
```

```
Client ──▶ Nginx ──▶ [waiting for Gunicorn response...]
                              │
                    30s elapses, no response
                              │
                              ▼
                    Nginx gives up, returns 504 Gateway Timeout to the client
                    (instead of hanging forever)
```

---

## 9. Connection Limits

Protects your backend from being overwhelmed — either by legitimate traffic spikes or malicious
abuse (a crude DoS, or one misbehaving client hammering an endpoint).

```nginx
http {
    limit_conn_zone $binary_remote_addr zone=addr:10m;
    limit_req_zone   $binary_remote_addr zone=req_limit:10m rate=10r/s;

    server {
        location / {
            limit_conn addr 20;              # max 20 simultaneous connections per client IP
            limit_req  zone=req_limit burst=20 nodelay;   # max ~10 requests/sec per IP, allowing bursts

            proxy_pass http://127.0.0.1:8000;
        }
    }
}
```

```
Client sends 100 requests in 1 second
          │
          ▼
   Nginx allows ~10/sec + a burst allowance, rejects/delays the rest
   with 503/429 — protects Gunicorn/Django from ever seeing the flood
```

This is a complementary, network-layer defense to the application-level rate limiter you built
with Redis in Level 10 — Nginx can reject abusive traffic before it even costs you a Python
process cycle.

---

## Putting It All Together — A Real Production Config

```nginx
# Redirect all HTTP to HTTPS
server {
    listen 80;
    server_name yourapp.com;
    return 301 https://$host$request_uri;
}

# Main HTTPS server
server {
    listen 443 ssl;
    server_name yourapp.com;

    ssl_certificate     /etc/letsencrypt/live/yourapp.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourapp.com/privkey.pem;

    # Static files served directly — never touches the app
    location /static/ {
        alias /opt/myapp/staticfiles/;
    }

    # Everything else goes to the app, load-balanced across 3 Gunicorn instances
    location / {
        limit_req zone=req_limit burst=20 nodelay;

        proxy_pass http://myapp_backend;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 5s;
        proxy_read_timeout    30s;
    }
}

upstream myapp_backend {
    least_conn;
    server 127.0.0.1:8001;
    server 127.0.0.1:8002;
    server 127.0.0.1:8003;
}
```

```
                     ┌──────────┐
Internet ──HTTPS───▶ │  :443       │
                     │  Nginx       │ ── serves /static/ directly
                     └────┬─────┘
                          │  plain HTTP, load-balanced, with proxy headers,
                          │  timeouts, and rate limiting all applied
              ┌───────────┼───────────┐
              ▼           ▼           ▼
        Gunicorn:8001 Gunicorn:8002 Gunicorn:8003
              │           │           │
              ▼           ▼           ▼
                     Django app code
                    (all 3 talk to the
                     same Postgres/Redis)
```

**The one-sentence takeaway:** Nginx exists to handle everything about serving traffic on the
open internet that has nothing to do with your business logic — TLS, static files, load
distribution, abuse protection, timeouts — so your app server and application code can stay
focused purely on processing requests.
