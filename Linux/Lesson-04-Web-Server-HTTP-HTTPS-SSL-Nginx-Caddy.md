# Industry Level Web Server Course
## HTTP, HTTPS, SSL/TLS, Nginx, Caddy — সম্পূর্ণ গাইড

---

## Roadmap

```
Internet
      │
      ▼
DNS
      │
      ▼
HTTP / HTTPS
      │
      ▼
Nginx / Caddy
      │
      ▼
Gunicorn / Uvicorn
      │
      ▼
Django Application
      │
      ▼
PostgreSQL
```

---

# Part 1 – How the Internet Works

## মূল Concepts

| Concept | অর্থ |
|---------|-------|
| **Client** | যে Request করে (Browser) |
| **Server** | যে Response দেয় |
| **IP Address** | Network-এ Device-এর ঠিকানা (e.g. `142.250.80.46`) |
| **Domain** | IP-এর মানুষ-পাঠযোগ্য নাম (e.g. `google.com`) |
| **DNS** | Domain → IP তে রূপান্তর করে |
| **Port** | একটি Server-এ নির্দিষ্ট Service-এর দরজা |
| **Socket** | IP + Port = একটি Connection endpoint |
| **TCP** | Reliable, ordered data delivery |

## একটি Request-এর যাত্রা

```
Browser → DNS Lookup → IP Address
       → TCP Connection (Port 443)
       → TLS Handshake
       → HTTP Request পাঠানো
       → Server Response
       → Browser-এ Render
```

```bash
# DNS lookup দেখুন
nslookup google.com
dig google.com

# TCP connection test করুন
telnet google.com 80
curl -v https://google.com
```

---

# Part 2 – HTTP

## HTTP কী?

**HyperText Transfer Protocol** — Client এবং Server-এর মধ্যে Data আদান-প্রদানের নিয়ম।

```
Browser
   │
   │  GET / HTTP/1.1
   │  Host: example.com
   ▼
Server
   │
   │  HTTP/1.1 200 OK
   │  Content-Type: text/html
   │  <html>...</html>
   ▼
Browser
```

## HTTP Methods

| Method | কাজ | উদাহরণ |
|--------|-----|---------|
| `GET` | Data নিয়ে আসা | Page load, API read |
| `POST` | নতুন Data পাঠানো | Form submit, Login |
| `PUT` | পুরো Resource update | Profile update |
| `PATCH` | আংশিক update | শুধু নাম পরিবর্তন |
| `DELETE` | Resource মুছে ফেলা | Post delete |

## HTTP Headers

```
GET /api/users HTTP/1.1
Host: example.com
Authorization: Bearer <token>
Content-Type: application/json
Accept: application/json
```

## HTTP Status Codes

| Code | অর্থ |
|------|-------|
| `200` | OK — সফল |
| `201` | Created — নতুন তৈরি হয়েছে |
| `301` | Moved Permanently — চিরতরে সরে গেছে |
| `302` | Found — সাময়িকভাবে অন্য জায়গায় |
| `400` | Bad Request — ভুল Request |
| `401` | Unauthorized — Login দরকার |
| `403` | Forbidden — Permission নেই |
| `404` | Not Found — পাওয়া যায়নি |
| `500` | Internal Server Error — Server-এর ভুল |
| `502` | Bad Gateway — Upstream Server সমস্যা |
| `503` | Service Unavailable — Server বন্ধ |

```bash
# HTTP Response দেখুন
curl -I http://example.com
curl -v http://example.com
```

---

# Part 3 – HTTPS

## HTTPS কী?

**HTTP + TLS (Transport Layer Security)** = HTTPS

HTTP-তে Data **Plain Text** আকারে যায়। যে কেউ মাঝপথে পড়তে পারে।

```
HTTP:   Browser → [username=mizan&password=1234] → Server   ← যে কেউ পড়তে পারে!
HTTPS:  Browser → [x#9$kL@2mQ!...encrypted...] → Server    ← পড়া অসম্ভব
```

## HTTPS কেন দরকার?

- Password, Credit Card চুরি ঠেকাতে
- Data tampering রোধ করতে
- Browser-এ 🔒 দেখাতে
- Google SEO Ranking-এর জন্য
- আধুনিক Browser HTTP Block করতে পারে

## SSL vs TLS

> **SSL** পুরনো (Deprecated)। **TLS** হলো নতুন এবং নিরাপদ version।
> কিন্তু মানুষ এখনও "SSL" বলতে TLS-ই বোঝায়।

| Version | অবস্থা |
|---------|--------|
| SSL 2.0, 3.0 | Deprecated (ব্যবহার করবেন না) |
| TLS 1.0, 1.1 | Deprecated |
| TLS 1.2 | Acceptable |
| TLS 1.3 | Recommended ✓ |

## HTTPS কীভাবে কাজ করে? — TLS Handshake

```
Browser                                    Server
   │                                          │
   │──── ClientHello (TLS version, ciphers) ──▶│
   │                                          │
   │◀─── ServerHello + Certificate ───────────│
   │                                          │
   │  [Certificate Verify করো]               │
   │  [Server কি বিশ্বস্ত?]                  │
   │                                          │
   │──── Session Key (Encrypted) ────────────▶│
   │                                          │
   │◀════ Encrypted Communication ════════════│
```

**Step by step:**
1. Browser → Server: "আমি TLS 1.3 এবং এই Cipher Suites support করি"
2. Server → Browser: Certificate পাঠায় (Public Key সহ)
3. Browser: Certificate CA দিয়ে verify করে
4. Browser: Session Key তৈরি করে, Server-এর Public Key দিয়ে Encrypt করে পাঠায়
5. Server: নিজের Private Key দিয়ে Decrypt করে Session Key পায়
6. এখন দুজনেই Session Key দিয়ে Encrypt করে কথা বলে

---

# Part 4 – SSL Certificate

## Certificate কী?

Certificate হলো একটি Digital Document যা প্রমাণ করে:
- এই Server সত্যিই `example.com`
- এই Public Key এই Server-এর

## Certificate Authority (CA)

CA হলো বিশ্বস্ত Third Party যারা Certificate Issue করে।

```
Root CA (DigiCert, Let's Encrypt, Comodo)
    │
    ▼
Intermediate CA
    │
    ▼
Your Certificate (example.com)
```

| ধরন | উদাহরণ |
|-----|---------|
| **Root CA** | DigiCert, GlobalSign, Let's Encrypt |
| **Intermediate CA** | Root CA-র নিচের স্তর |
| **Leaf Certificate** | আপনার domain-এর certificate |

## Certificate-এ কী থাকে?

```bash
# Certificate দেখুন
openssl x509 -in certificate.crt -text -noout
```

- Domain Name (CN / SAN)
- Public Key
- Issuer (CA)
- Valid From / To
- Digital Signature

---

# Part 5 – Public Key Cryptography

## সবচেয়ে গুরুত্বপূর্ণ অংশ

**Key Pair** = Private Key + Public Key

```
Private Key  →  গোপন, শুধু আপনার কাছে
Public Key   →  সবাইকে দিতে পারবেন
```

## Encryption ও Decryption

```
Sender:   Message + Public Key  → Encrypted Message
Receiver: Encrypted Message + Private Key → Original Message
```

> Public Key দিয়ে Encrypt, Private Key দিয়ে Decrypt।

## Signing ও Verification

```
Signer:   Message + Private Key  → Signature
Verifier: Message + Signature + Public Key → Valid / Invalid
```

> Private Key দিয়ে Sign, Public Key দিয়ে Verify।

## Private Key

```
private.key  ← এটি গোপন
```

> ⚠️ **কখনো GitHub-এ Upload করবেন না।**
> `.gitignore`-এ রাখুন।

```bash
# .gitignore
*.key
*.pem
private/
```

## Public Key / Certificate

```
certificate.crt  ← এটি সবাইকে দেওয়া যায়
```

---

# Part 6 – Let's Encrypt

## Free SSL Certificate

**Let's Encrypt** হলো একটি Free, Automated, Open Certificate Authority।

```
আপনার Server
      │
      │  "আমি example.com-এর মালিক"
      ▼
Let's Encrypt
      │
      │  Domain Verify করে (HTTP Challenge / DNS Challenge)
      ▼
Certificate Issue করে (90 দিনের জন্য)
      │
      ▼
Auto Renew (Certbot / Caddy)
```

## Certificate কীভাবে Issue হয়? — ACME Protocol

```bash
# Let's Encrypt verify করে যে আপনি domain-এর মালিক
# HTTP-01 Challenge:
# /.well-known/acme-challenge/<token> এ একটি file রাখতে হয়

# DNS-01 Challenge:
# DNS TXT record যোগ করতে হয়
```

---

# Part 7 – Caddy

## Caddy কী?

Caddy একটি আধুনিক Web Server — Go ভাষায় লেখা।

**বিশেষ সুবিধা:**
- ✅ Automatic HTTPS
- ✅ Automatic Certificate Renewal
- ✅ সহজ Configuration
- ✅ Built-in TLS 1.3
- ✅ HTTP/2 ও HTTP/3 support

## Caddy Install (Ubuntu)

```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```

## Basic Caddyfile

```
example.com {
    reverse_proxy localhost:8000
}
```

মাত্র এটুকু লিখলে Caddy:
- Let's Encrypt থেকে Certificate নামাবে
- HTTPS Enable করবে
- HTTP → HTTPS Redirect করবে
- Certificate Auto Renew করবে

## Multiple Sites

```
example.com {
    reverse_proxy localhost:8000
}

api.example.com {
    reverse_proxy localhost:9000
}

blog.example.com {
    reverse_proxy localhost:8001
}
```

## Static Files

```
example.com {
    root * /var/www/html
    file_server
}
```

## Reverse Proxy with Headers

```
example.com {
    reverse_proxy localhost:8000 {
        header_up Host {host}
        header_up X-Real-IP {remote_host}
        header_up X-Forwarded-For {remote_host}
        header_up X-Forwarded-Proto {scheme}
    }
}
```

## Reverse Proxy Flow

```
Browser
   │
   ▼
Caddy (Port 443, HTTPS)
   │
   ▼
Gunicorn (Port 8000, localhost)
   │
   ▼
Django
```

## Caddy Commands

```bash
sudo systemctl start caddy
sudo systemctl stop caddy
sudo systemctl reload caddy     # Config reload (no downtime)
sudo systemctl status caddy

caddy validate --config /etc/caddy/Caddyfile   # Config test
caddy fmt --overwrite /etc/caddy/Caddyfile     # Format করুন

# Logs দেখুন
journalctl -u caddy -f
```

---

# Part 8 – Nginx

## Nginx কী?

- Web Server
- Reverse Proxy
- Load Balancer
- Static File Server

## Install

```bash
sudo apt install nginx
sudo systemctl enable nginx
sudo systemctl start nginx
```

## HTTP Configuration

```nginx
# /etc/nginx/sites-available/example.com

server {
    listen 80;
    server_name example.com www.example.com;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /static/ {
        alias /home/mizan/shop_project/staticfiles/;
    }

    location /media/ {
        alias /home/mizan/shop_project/media/;
    }
}
```

## HTTPS Configuration

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate     /etc/ssl/certs/example.crt;
    ssl_certificate_key /etc/ssl/private/private.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## HTTP → HTTPS Redirect

```nginx
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;
}
```

## Nginx Commands

```bash
sudo nginx -t                    # Config test
sudo systemctl reload nginx      # Reload (no downtime)
sudo systemctl restart nginx     # Restart

# Logs
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log
```

---

# Part 9 – SSL Files

| File | অর্থ |
|------|-------|
| `private.key` | Private Key — গোপন রাখুন |
| `certificate.crt` | আপনার Domain-এর Certificate |
| `fullchain.pem` | Certificate + Intermediate CA chain |
| `chain.pem` | শুধু Intermediate CA |

**Let's Encrypt ফাইলের location:**
```
/etc/letsencrypt/live/example.com/
├── cert.pem        → certificate
├── chain.pem       → intermediate chain
├── fullchain.pem   → cert + chain
└── privkey.pem     → private key
```

---

# Part 10 – OpenSSL

```bash
# Version দেখুন
openssl version

# Certificate তথ্য দেখুন
openssl x509 -in certificate.crt -text -noout

# Private Key তৈরি করুন
openssl genrsa -out private.key 2048

# CSR (Certificate Signing Request) তৈরি করুন
openssl req -new -key private.key -out request.csr

# Certificate verify করুন
openssl verify -CAfile chain.pem certificate.crt

# Remote server-এর certificate দেখুন
openssl s_client -connect example.com:443

# Private Key check করুন
openssl rsa -in private.key -check
```

---

# Part 11 – Self-Signed Certificate

Development-এর জন্য নিজে Certificate তৈরি।

```bash
# একটি command-এ Self-Signed Certificate
openssl req -x509 -newkey rsa:4096 \
    -keyout private.key \
    -out certificate.crt \
    -days 365 \
    -nodes \
    -subj "/CN=localhost"
```

> ⚠️ Self-Signed Certificate Browser-এ Warning দেখাবে। শুধু Development-এ ব্যবহার করুন।

---

# Part 12 – Let's Encrypt + Certbot

## Nginx-এর জন্য

```bash
# Certbot install
sudo apt install certbot python3-certbot-nginx

# Certificate নিন
sudo certbot --nginx -d example.com -d www.example.com

# Manual certificate (standalone)
sudo certbot certonly --standalone -d example.com

# Renewal test করুন
sudo certbot renew --dry-run

# Auto renewal (already setup by certbot)
systemctl status certbot.timer
```

---

# Part 13 – Caddy Automatic HTTPS

Caddy-এর সবচেয়ে বড় সুবিধা — আপনাকে কিছুই করতে হয় না।

```
# Caddyfile
example.com {
    reverse_proxy localhost:8000
}
```

Caddy নিজে থেকে করবে:

```
DNS Check (example.com কি এই Server-এ point করছে?)
      │
      ▼
Let's Encrypt ACME Challenge
      │
      ▼
Certificate Download
      │
      ▼
HTTPS Enable
      │
      ▼
HTTP → HTTPS Auto Redirect
      │
      ▼
90 দিন পরে Auto Renew
```

---

# Part 14 – Django Deployment

## Full Stack

```
Internet (User)
      │
      ▼
DNS (example.com → Server IP)
      │
      ▼
Caddy অথবা Nginx (Port 443, HTTPS)
      │
      ▼
Gunicorn (Port 8000, Unix Socket)
      │
      ▼
Django Application
      │
      ▼
PostgreSQL (Port 5432)
```

## Gunicorn Setup

```bash
pip install gunicorn

# চালু করুন
gunicorn shop_project.wsgi:application \
    --bind 127.0.0.1:8000 \
    --workers 3

# Systemd service হিসেবে
# /etc/systemd/system/gunicorn.service
```

```ini
[Unit]
Description=Gunicorn Django Server
After=network.target

[Service]
User=mizan
WorkingDirectory=/home/mizan/shop_project
ExecStart=/home/mizan/venv/bin/gunicorn shop_project.wsgi:application --bind 127.0.0.1:8000 --workers 3
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable gunicorn
sudo systemctl start gunicorn
```

---

# Part 15 – Security

## HSTS (HTTP Strict Transport Security)

Browser-কে বলে: এই site সবসময় HTTPS-এ যাও।

```nginx
# Nginx
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```

```
# Caddyfile
example.com {
    header Strict-Transport-Security "max-age=31536000; includeSubDomains"
    reverse_proxy localhost:8000
}
```

## Security Headers

```nginx
add_header X-Content-Type-Options nosniff;
add_header X-Frame-Options DENY;
add_header X-XSS-Protection "1; mode=block";
add_header Referrer-Policy "strict-origin-when-cross-origin";
```

## TLS Best Practices

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 1d;
```

---

# Part 16 – Performance

## Compression

```nginx
# Nginx — Gzip
gzip on;
gzip_types text/plain text/css application/json application/javascript;
gzip_min_length 1000;
```

```
# Caddy — Brotli + Gzip
example.com {
    encode zstd gzip
    reverse_proxy localhost:8000
}
```

## Caching (Static Files)

```nginx
location /static/ {
    alias /home/mizan/shop_project/staticfiles/;
    expires 30d;
    add_header Cache-Control "public, immutable";
}
```

## Load Balancing (Nginx)

```nginx
upstream django {
    server 127.0.0.1:8000;
    server 127.0.0.1:8001;
    server 127.0.0.1:8002;
}

server {
    location / {
        proxy_pass http://django;
    }
}
```

---

# Part 17 – Debugging

## `curl` — HTTP Request Test

```bash
curl https://example.com
curl -I https://example.com              # Headers only
curl -v https://example.com             # Verbose (full details)
curl -k https://example.com             # SSL verify skip
curl -L http://example.com              # Follow redirects
```

## `openssl s_client` — SSL Debug

```bash
openssl s_client -connect example.com:443
openssl s_client -connect example.com:443 -tls1_3
```

## Logs দেখুন

```bash
# Nginx
tail -f /var/log/nginx/error.log
tail -f /var/log/nginx/access.log

# Caddy
journalctl -u caddy -f

# Gunicorn
journalctl -u gunicorn -f

# Systemd
systemctl status nginx
systemctl status caddy
```

## Port Check

```bash
ss -tulpn | grep 443
ss -tulpn | grep 80
ss -tulpn | grep 8000
```

---

# Part 18 – Production Deployment (Step by Step)

## Ubuntu VPS থেকে Live Django App

```bash
# 1. Server Update
sudo apt update && sudo apt upgrade -y

# 2. Dependencies Install
sudo apt install python3-pip python3-venv postgresql nginx -y

# 3. Django Project Clone
cd /home/mizan
git clone https://github.com/mizan/shop_project.git
cd shop_project

# 4. Virtual Environment
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
pip install gunicorn

# 5. PostgreSQL Setup
sudo -u postgres psql
# CREATE DATABASE shopdb;
# CREATE USER shopuser WITH PASSWORD '<password>';
# GRANT ALL PRIVILEGES ON DATABASE shopdb TO shopuser;

# 6. Django Settings (Production)
# DEBUG = False
# ALLOWED_HOSTS = ['example.com']

# 7. Static Files
python manage.py collectstatic
python manage.py migrate

# 8. Gunicorn Service চালু করুন
sudo systemctl start gunicorn

# 9. Caddy Install ও Configure
sudo apt install caddy
sudo nano /etc/caddy/Caddyfile
# example.com {
#     reverse_proxy localhost:8000
# }
sudo systemctl reload caddy

# 10. DNS: example.com → Server IP (Namecheap / Cloudflare)
# কিছুক্ষণ অপেক্ষা করুন...
# Caddy নিজেই HTTPS করে নেবে ✓
```

---

# Caddy বনাম Nginx

| Feature | Caddy | Nginx |
|---------|-------|-------|
| Configuration | খুব সহজ | তুলনামূলক জটিল |
| HTTPS | স্বয়ংক্রিয় | Certbot বা ম্যানুয়াল |
| SSL Renewal | স্বয়ংক্রিয় | Certbot Timer বা Cron |
| Performance | খুব ভালো | খুব ভালো |
| Static Files | হ্যাঁ | হ্যাঁ |
| Reverse Proxy | হ্যাঁ | হ্যাঁ |
| Load Balancer | হ্যাঁ | হ্যাঁ |
| Learning Curve | সহজ | মাঝারি |
| Production Use | বাড়ছে | Industry Standard |

> **Recommendation:** নতুন Project-এ **Caddy** ব্যবহার করুন। Legacy বা বড় Infrastructure-এ **Nginx**।

---

## Quick Reference

```bash
# Certificate দেখুন
openssl s_client -connect example.com:443 | openssl x509 -noout -dates

# Caddy reload
sudo systemctl reload caddy

# Nginx test ও reload
sudo nginx -t && sudo systemctl reload nginx

# Certbot renew
sudo certbot renew

# Port check
ss -tulpn | grep -E '80|443|8000'

# Logs
journalctl -u caddy -f
tail -f /var/log/nginx/error.log
```
