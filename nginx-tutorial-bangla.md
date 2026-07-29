# Nginx সম্পূর্ণ টিউটোরিয়াল (বাংলা)

---

## Nginx কী এবং কেন ব্যবহার করি?

Nginx একটি **web server** এবং **reverse proxy server**।

আমাদের Django app সরাসরি internet-এ expose করা safe না। তাই Nginx মাঝখানে বসে:
- User এর request নেয়
- Gunicorn (Django) এ পাঠায়
- Response ফিরিয়ে দেয়

```
User → Nginx (80/443) → Gunicorn (8005) → Django App
```

---

## Nginx Configuration এর Block Structure

```
http {                          ← সব কিছুর parent block (nginx.conf এ থাকে)
    server {                    ← একটি virtual host / domain
        location / {            ← নির্দিষ্ট URL path এর জন্য rule
        }
    }
}
```

আমাদের `app.config` ফাইলে শুধু `server {}` block থাকে কারণ এটি `http {}` এর ভেতরে include হয়।

---

## আমাদের সম্পূর্ণ Config ব্যাখ্যা

### Block 1 — HTTP Server (Port 80)

```nginx
server {
    listen 80;
    server_name domain www.domain;

    location /.well-known/acme-challenge/ {
        root /var/www/html;
    }

    return 301 https://$host$request_uri;
}
```

| Attribute | মানে | না থাকলে কী হবে |
|---|---|---|
| `listen 80` | Port 80 এ HTTP request শোনো | Nginx কোনো HTTP request পাবে না |
| `server_name domain www.domain` | এই domain এর জন্য এই block কাজ করবে | সব domain এই block এ আসবে |
| `location /.well-known/acme-challenge/` | Certbot SSL verify করার জন্য এই path দরকার | SSL certificate renew হবে না |
| `root /var/www/html` | Certbot verification file এই folder এ খুঁজবে | 404 error আসবে, SSL renew fail করবে |
| `return 301 https://$host$request_uri` | HTTP কে HTTPS এ redirect করো | User HTTP তে থাকবে, connection insecure হবে |

> `$host` = request এর domain name  
> `$request_uri` = URL এর path + query string (যেমন `/api/users?page=1`)

---

### Block 2 — HTTPS Server (Port 443)

```nginx
server {
    listen 443 ssl;
    server_name domain www.domain;

    ssl_certificate /etc/letsencrypt/live/domain/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/domain/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ...
}
```

| Attribute | মানে | না থাকলে কী হবে |
|---|---|---|
| `listen 443 ssl` | Port 443 এ HTTPS শোনো | HTTPS কাজ করবে না |
| `ssl_certificate` | Public certificate file এর path | SSL handshake fail, browser error দেখাবে |
| `ssl_certificate_key` | Private key file এর path | SSL কাজ করবে না |
| `ssl_protocols TLSv1.2 TLSv1.3` | শুধু এই দুটো secure protocol allow করো | পুরনো insecure TLSv1.0/1.1 দিয়ে attack হতে পারে |

---

### Block 3 — Django Proxy (location /)

```nginx
location / {
    proxy_pass http://127.0.0.1:8005;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

| Attribute | মানে | না থাকলে কী হবে |
|---|---|---|
| `proxy_pass http://127.0.0.1:8005` | Request টি Gunicorn এ পাঠাও | Django app কাজ করবে না |
| `proxy_set_header Host $host` | Original domain name Django কে জানাও | Django `ALLOWED_HOSTS` check fail করতে পারে |
| `proxy_set_header X-Real-IP $remote_addr` | User এর real IP Django কে পাঠাও | Django সবসময় Nginx এর IP (127.0.0.1) দেখবে |
| `proxy_set_header X-Forwarded-For` | Proxy chain এর সব IP পাঠাও | Rate limiting, logging, IP ban কাজ করবে না |
| `proxy_set_header X-Forwarded-Proto $scheme` | HTTP নাকি HTTPS সেটা জানাও | Django `request.is_secure()` সবসময় False দেবে |

> `$remote_addr` = client এর IP address  
> `$scheme` = `http` অথবা `https`

---

### Block 4 — Static Files (location /static/)

```nginx
location /static/ {
    alias /home/ubuntu/ikon/staticfiles/;

    access_log off;
    expires 30d;
    add_header Cache-Control "public, max-age=2592000";
}
```

| Attribute | মানে | না থাকলে কী হবে |
|---|---|---|
| `alias /home/ubuntu/ikon/staticfiles/` | এই folder থেকে static file serve করো | Django কে দিয়ে serve হবে, অনেক slow হবে |
| `access_log off` | Static file এর log রেখো না | Log file অনেক বড় হয়ে disk full হবে |
| `expires 30d` | Browser কে বলো 30 দিন cache রাখতে | প্রতিবার CSS/JS re-download হবে, site slow হবে |
| `add_header Cache-Control "public"` | CDN ও cache করতে পারবে | CDN caching কাজ করবে না |

> **`alias` vs `root` পার্থক্য:**
> - `root` ব্যবহার করলে: `/static/` + `/home/ubuntu/ikon/staticfiles/` = `/home/ubuntu/ikon/staticfiles/static/` (ভুল path)
> - `alias` ব্যবহার করলে: `/static/` replace হয়ে `/home/ubuntu/ikon/staticfiles/` হয় (সঠিক)

---

### Block 5 — Media Files (location /media/)

```nginx
location /media/ {
    alias /home/ubuntu/ikon/media/;

    access_log off;
    expires 30d;
    add_header Cache-Control "public";
}
```

Static files এর মতোই, তবে এটি user uploaded files (images, documents) এর জন্য।

---

## Standard Attributes যা যোগ করা উচিত

### SSL Hardening
```nginx
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 1d;
ssl_prefer_server_ciphers on;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
```

### Security Headers
```nginx
add_header X-Frame-Options "SAMEORIGIN";
add_header X-Content-Type-Options "nosniff";
add_header X-XSS-Protection "1; mode=block";
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```

### Gzip Compression
```nginx
gzip on;
gzip_types text/plain text/css application/json application/javascript;
gzip_min_length 1024;
```

### Timeout & Upload Size
```nginx
client_max_body_size 20M;
client_body_timeout 12s;
keepalive_timeout 65s;
proxy_read_timeout 60s;
```

---

## Multi-Server Setup — Load Balancing Tutorial

### ধারণা

একটি server এ অনেক বেশি traffic আসলে সেটি slow বা crash করতে পারে। তখন একাধিক server এ load ভাগ করে দেওয়া হয় — এটাই **Load Balancing**।

```
                        ┌─── Server 1 (Gunicorn :8001)
User → Nginx → upstream ─── Server 2 (Gunicorn :8002)
                        └─── Server 3 (Gunicorn :8003)
```

---

### upstream Block

`upstream` block এ সব backend server এর list দেওয়া হয়।

```nginx
upstream django_backend {
    server 127.0.0.1:8001;
    server 127.0.0.1:8002;
    server 127.0.0.1:8003;
}

server {
    listen 443 ssl;

    location / {
        proxy_pass http://django_backend;  # upstream এর নাম দাও
    }
}
```

---

## Load Balancing Algorithms

### 1. Round Robin (Default)

প্রতিটি request একে একে সব server এ যায়।

```nginx
upstream django_backend {
    server 127.0.0.1:8001;
    server 127.0.0.1:8002;
    server 127.0.0.1:8003;
}
```

```
Request 1 → Server 1
Request 2 → Server 2
Request 3 → Server 3
Request 4 → Server 1  (আবার শুরু)
```

**কখন ব্যবহার করবে:** সব server এর capacity সমান হলে।

---

### 2. Weighted Round Robin

বেশি powerful server বেশি request পাবে।

```nginx
upstream django_backend {
    server 127.0.0.1:8001 weight=3;  # 3টি request পাবে
    server 127.0.0.1:8002 weight=1;  # 1টি request পাবে
}
```

```
Request 1,2,3 → Server 1
Request 4     → Server 2
Request 5,6,7 → Server 1  (আবার)
```

**কখন ব্যবহার করবে:** Server গুলোর RAM/CPU আলাদা হলে।

---

### 3. Least Connections

যে server এ সবচেয়ে কম active connection আছে সেখানে request যাবে।

```nginx
upstream django_backend {
    least_conn;
    server 127.0.0.1:8001;
    server 127.0.0.1:8002;
    server 127.0.0.1:8003;
}
```

**কখন ব্যবহার করবে:** কিছু request দ্রুত শেষ হয়, কিছু অনেক সময় নেয় (যেমন file upload)।

---

### 4. IP Hash

একই user সবসময় একই server এ যাবে (session sticky)।

```nginx
upstream django_backend {
    ip_hash;
    server 127.0.0.1:8001;
    server 127.0.0.1:8002;
}
```

**কখন ব্যবহার করবে:** Session বা login state server এ store হলে (Redis না থাকলে)।

---

### 5. Server Down হলে কী হবে? (Failover)

```nginx
upstream django_backend {
    server 127.0.0.1:8001;
    server 127.0.0.1:8002;
    server 127.0.0.1:8003 backup;  # শুধু বাকিরা down হলে কাজ করবে
}
```

`max_fails` এবং `fail_timeout` দিয়ে auto-failover:

```nginx
upstream django_backend {
    server 127.0.0.1:8001 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:8002 max_fails=3 fail_timeout=30s;
}
```

> `max_fails=3` → 3 বার fail করলে সেই server কে temporarily বাদ দাও  
> `fail_timeout=30s` → 30 সেকেন্ড পর আবার try করো

---

## Multi-Server এর সম্পূর্ণ Config উদাহরণ

```nginx
# সব backend server এর list
upstream django_backend {
    least_conn;                                          # algorithm
    server 10.0.0.1:8000 max_fails=3 fail_timeout=30s; # Server 1
    server 10.0.0.2:8000 max_fails=3 fail_timeout=30s; # Server 2
    server 10.0.0.3:8000 backup;                        # Backup server
}

server {
    listen 80;
    server_name domain www.domain;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name domain www.domain;

    ssl_certificate /etc/letsencrypt/live/domain/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/domain/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";
    add_header Strict-Transport-Security "max-age=31536000" always;

    # Gzip
    gzip on;
    gzip_types text/plain text/css application/json application/javascript;

    # Upload size
    client_max_body_size 20M;

    location / {
        proxy_pass http://django_backend;  # upstream এ পাঠাও

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_read_timeout 60s;
        proxy_connect_timeout 60s;
    }

    location /static/ {
        alias /home/ubuntu/ikon/staticfiles/;
        expires 30d;
        add_header Cache-Control "public, max-age=2592000";
        access_log off;
    }

    location /media/ {
        alias /home/ubuntu/ikon/media/;
        expires 30d;
        add_header Cache-Control "public";
        access_log off;
    }
}
```

---

## Quick Reference — সব Attribute এক জায়গায়

| Attribute | Block | কাজ |
|---|---|---|
| `listen` | server | কোন port এ শুনবে |
| `server_name` | server | কোন domain এর জন্য |
| `ssl_certificate` | server | SSL public cert path |
| `ssl_certificate_key` | server | SSL private key path |
| `ssl_protocols` | server | কোন TLS version allow |
| `return 301` | server/location | Redirect করো |
| `proxy_pass` | location | কোথায় forward করবে |
| `proxy_set_header` | location | Header যোগ করো |
| `alias` | location | Folder path map করো |
| `root` | location | Base folder set করো |
| `expires` | location | Browser cache time |
| `add_header` | location/server | HTTP header যোগ করো |
| `access_log off` | location | Log বন্ধ করো |
| `gzip on` | http/server | Compression চালু করো |
| `client_max_body_size` | http/server | Max upload size |
| `upstream` | http | Backend server group |
| `least_conn` | upstream | Algorithm: least connection |
| `ip_hash` | upstream | Algorithm: same user same server |
| `weight` | upstream server | Server এর priority |
| `backup` | upstream server | Fallback server |
| `max_fails` | upstream server | কতবার fail হলে বাদ দেবে |
| `fail_timeout` | upstream server | কতক্ষণ পর আবার try করবে |
