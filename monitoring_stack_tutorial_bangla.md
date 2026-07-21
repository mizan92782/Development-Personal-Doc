# Django Application Monitoring Stack — সম্পূর্ণ টিউটোরিয়াল (বাংলা)

এই টিউটোরিয়ালে আমরা তোমার ডায়াগ্রামে দেখানো পুরো monitoring architecture টা ধাপে ধাপে বুঝবো, প্রতিটা component কী কাজ করে সেটা explain করবো, আর সবশেষে Django app-এর সাথে কীভাবে configure করতে হয় সেটা দেখাবো।

---

## ১. পুরো সিস্টেমের High-Level Overview

তোমার ডায়াগ্রামে দুইটা আলাদা "pipeline" চলছে — একটা হলো **Logs**, আরেকটা হলো **Metrics**। দুইটাই আলাদা আলাদা tool দিয়ে collect হয়, কিন্তু শেষে একসাথে **Grafana**-তে visualize হয়।

```
Internet → nginx/caddy → Application (Django)
                              │
              ┌───────────────┴───────────────┐
              │                               │
         Write log                    Expose /metrics
              │                               │
       /var/log/app.log                       │
              │                               │
          Promtail                            │
              │                               │
           push logs                          │
              │                               │
            Loki ──────────┐            Prometheus ← Node Exporter
              │            │                  │
              │            └──────► Grafana ◄─┘
              │                               │
              │                          Alert Rules
              │                               │
              │                        Alertmanager
              │                               │
              │                    ┌──────────┼──────────┐
              │                  Email     Telegram     Slack
```


## How Promtail working
```
Application

        │

Write

        │

        ▼

/logs/app.log

        │

Promtail watches

        │

        ▼

Read new lines

        │

Add labels

        │

        ▼

Loki

        │

        ▼

Grafana

        │

Search

{job="myapp"}

```

**সহজ কথায়:**
- **Logs** যায় → Promtail → Loki → Grafana (দেখার জন্য)
- **Metrics** যায় → Prometheus → Grafana (দেখার জন্য) + Alertmanager (alert পাঠানোর জন্য)

---

## ২. প্রতিটা Component কী কাজ করে

### 🔹 nginx / caddy (Reverse Proxy)
Internet থেকে আসা request প্রথমে এখানে আসে। এটা request-কে তোমার Django app (Gunicorn/uWSGI)-এ forward করে। এছাড়া SSL, rate-limiting, static file serving এর কাজও এখানে হয়।

### 🔹 Application (Django)
তোমার মূল app। এটা দুইটা জিনিস করে:
1. একটা log file-এ (`/var/log/app.log`) log লিখে
2. একটা `/metrics` endpoint expose করে, যেখান থেকে Prometheus data pull করে

### 🔹 Promtail
এটা একটা **log shipping agent**। এটা `/var/log/app.log` ফাইলটা tail করে (মানে নতুন লাইন আসলেই পড়ে) এবং সেই log গুলো Loki-তে push করে দেয়। এটা অনেকটা Filebeat-এর মতো, কিন্তু Loki ecosystem-এর জন্য বানানো।

### 🔹 Loki
এটা Prometheus-এর মতোই কাজ করে, কিন্তু metrics-এর বদলে **logs** store করে। এর সবচেয়ে বড় সুবিধা হলো এটা log-এর পুরো content index করে না (শুধু labels index করে), তাই এটা অনেক lightweight এবং সস্তা storage-এ চলে।

### 🔹 Node Exporter
এটা সার্ভারের **hardware/OS-level metrics** (CPU, RAM, Disk, Network) collect করে এবং একটা `/metrics` endpoint-এ expose করে। Prometheus এখান থেকে data pull করে।

### 🔹 Prometheus
এটা হলো **metrics collection ও storage system**। এটা নির্দিষ্ট সময় পরপর (যেমন প্রতি ১৫ সেকেন্ডে) তোমার Django app-এর `/metrics` এবং Node Exporter-এর `/metrics` endpoint hit করে data নিয়ে নেয় (একে বলে **pull-based scraping**)। এটা time-series database হিসেবে কাজ করে।

### 🔹 Alert Rules
Prometheus-এর ভিতরেই define করা কিছু condition, যেমন: "যদি CPU usage ৮৫%-এর বেশি হয় ৫ মিনিট ধরে, তাহলে alert trigger করো।" এই rule গুলো Prometheus নিজে continuously check করে।

### 🔹 Alertmanager
Alert Rules trigger হলে Prometheus সেই alert **Alertmanager**-এ পাঠায়। Alertmanager-এর কাজ হলো:
- Duplicate alert গুলো group করা
- কাকে (কোন channel-এ) alert পাঠাতে হবে সেটা routing rule অনুযায়ী ঠিক করা
- Email, Telegram, Slack ইত্যাদিতে notification পাঠানো

### 🔹 Grafana
এটা **visualization layer**। এটা Prometheus এবং Loki দুটোকেই datasource হিসেবে ব্যবহার করে dashboard বানায় — একই জায়গা থেকে তুমি metrics (graph) আর logs দুটোই দেখতে পারবে।

---

## ৩. এখন Django-এর সাথে Configuration — ধাপে ধাপে

### Step 1 — প্রয়োজনীয় প্যাকেজ install করা

```bash
pip install django-prometheus python-json-logger
```

`django-prometheus` তোমার Django app-এ স্বয়ংক্রিয়ভাবে `/metrics` endpoint তৈরি করে দেবে (request count, latency, DB queries ইত্যাদি track করে)।

---

### Step 2 — `settings.py`-এ django-prometheus যোগ করা

```python
# settings.py

INSTALLED_APPS = [
    "django_prometheus",   # সবার আগে যোগ করা ভালো
    # ... তোমার বাকি apps
]

MIDDLEWARE = [
    "django_prometheus.middleware.PrometheusBeforeMiddleware",  # সবার প্রথমে
    # ... তোমার বাকি middleware গুলো
    "django_prometheus.middleware.PrometheusAfterMiddleware",   # সবার শেষে
]

# Database metrics চাইলে (optional কিন্তু recommended)
DATABASES = {
    "default": {
        "ENGINE": "django_prometheus.db.backends.postgresql",
        "NAME": "mydb",
        "USER": "myuser",
        "PASSWORD": "mypassword",
        "HOST": "localhost",
        "PORT": "5432",
    }
}
```

---

### Step 3 — `urls.py`-এ `/metrics` endpoint যোগ করা

```python
# urls.py
from django.urls import path, include

urlpatterns = [
    # ... তোমার অন্যান্য url
    path("", include("django_prometheus.urls")),  # এটা /metrics endpoint তৈরি করবে
]
```

এখন `http://your-app:8000/metrics` এ hit করলে Prometheus format-এ metrics দেখতে পাবে।

⚠️ **গুরুত্বপূর্ণ:** Production-এ `/metrics` endpoint public রাখা উচিত না। nginx দিয়ে এই path-টা শুধু internal network (Prometheus server) থেকে access করার জন্য restrict করে দাও:

```nginx
# nginx config
location /metrics {
    allow 10.0.0.0/8;      # শুধু internal network
    deny all;
    proxy_pass http://django_app;
}
```

---

### Step 4 — Django-কে log file-এ log লিখতে বলা (`LOGGING` config)

```python
# settings.py

LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {
        "json": {
            "()": "pythonjsonlogger.jsonlogger.JsonFormatter",
            "format": "%(asctime)s %(levelname)s %(name)s %(message)s",
        },
    },
    "handlers": {
        "file": {
            "level": "INFO",
            "class": "logging.handlers.RotatingFileHandler",
            "filename": "/var/log/app.log",
            "maxBytes": 1024 * 1024 * 50,   # 50MB
            "backupCount": 5,
            "formatter": "json",
        },
        "console": {
            "level": "INFO",
            "class": "logging.StreamHandler",
            "formatter": "json",
        },
    },
    "root": {
        "handlers": ["file", "console"],
        "level": "INFO",
    },
    "loggers": {
        "django": {
            "handlers": ["file", "console"],
            "level": "INFO",
            "propagate": False,
        },
    },
}
```

JSON format-এ log লিখলে পরে Loki/Grafana-তে query করা এবং filter করা অনেক সহজ হয়।

---

### Step 5 — Promtail config (`promtail-config.yaml`)

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: django_app
    static_configs:
      - targets:
          - localhost
        labels:
          job: django_app
          env: production
          __path__: /var/log/app.log
```

---

### Step 6 — Prometheus config (`prometheus.yml`)

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: ["alertmanager:9093"]

rule_files:
  - "alert_rules.yml"

scrape_configs:
  - job_name: "django_app"
    static_configs:
      - targets: ["django_app:8000"]   # তোমার Django app-এর host:port
    metrics_path: /metrics

  - job_name: "node_exporter"
    static_configs:
      - targets: ["node_exporter:9100"]
```

---

### Step 7 — Alert Rules (`alert_rules.yml`)

```yaml
groups:
  - name: django_and_server_alerts
    rules:
      - alert: HighCPUUsage
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 85
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "উচ্চ CPU ব্যবহার সনাক্ত হয়েছে ({{ $labels.instance }})"
          description: "গত ৫ মিনিটে CPU usage 85%-এর বেশি।"

      - alert: DjangoHighErrorRate
        expr: rate(django_http_responses_total_by_status_total{status=~"5.."}[5m]) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Django app-এ 5xx error rate বেশি"
          description: "গত ৫ মিনিটে 5xx response rate অস্বাভাবিক বেশি।"

      - alert: HighMemoryUsage
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 90
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "মেমোরি ব্যবহার বেশি ({{ $labels.instance }})"
```

---

### Step 8 — Alertmanager config (`alertmanager.yml`) — Email, Telegram, Slack

```yaml
route:
  receiver: "default-notifier"
  group_by: ["alertname"]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 3h

receivers:
  - name: "default-notifier"
    email_configs:
      - to: "you@example.com"
        from: "alerts@example.com"
        smarthost: "smtp.gmail.com:587"
        auth_username: "alerts@example.com"
        auth_password: "app_password_here"
        send_resolved: true

    slack_configs:
      - api_url: "https://hooks.slack.com/services/XXX/YYY/ZZZ"
        channel: "#alerts"
        send_resolved: true

    telegram_configs:
      - bot_token: "YOUR_TELEGRAM_BOT_TOKEN"
        chat_id: -1001234567890
        send_resolved: true
```

---

### Step 9 — Grafana Datasource provisioning (`grafana-datasources.yml`)

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true

  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
```

এভাবে Grafana ওপেন করলে দুটো datasource-ই আগে থেকে configured থাকবে — একটাতে metrics graph বানাবে, আরেকটাতে logs explore করবে।

---

### Step 10 — সব একসাথে চালানোর জন্য `docker-compose.yml` (উদাহরণ)

```yaml
version: "3.8"

services:
  django_app:
    build: .
    ports:
      - "8000:8000"
    volumes:
      - ./logs:/var/log

  promtail:
    image: grafana/promtail:2.9.0
    volumes:
      - ./logs:/var/log
      - ./promtail-config.yaml:/etc/promtail/config.yml
    command: -config.file=/etc/promtail/config.yml

  loki:
    image: grafana/loki:2.9.0
    ports:
      - "3100:3100"

  node_exporter:
    image: prom/node-exporter:latest
    ports:
      - "9100:9100"

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./alert_rules.yml:/etc/prometheus/alert_rules.yml
    ports:
      - "9090:9090"

  alertmanager:
    image: prom/alertmanager:latest
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
    ports:
      - "9093:9093"

  grafana:
    image: grafana/grafana:latest
    volumes:
      - ./grafana-datasources.yml:/etc/grafana/provisioning/datasources/datasources.yml
    ports:
      - "3000:3000"
```

চালানোর জন্য:

```bash
docker-compose up -d
```

---

## ৪. Testing / যাচাই করার ধাপ

1. **Django metrics চেক:** `curl http://localhost:8000/metrics` → Prometheus format-এ output দেখাবে
2. **Prometheus targets চেক:** `http://localhost:9090/targets` → দেখো `django_app` আর `node_exporter` UP আছে কিনা
3. **Loki logs চেক:** Grafana → Explore → Loki datasource → query `{job="django_app"}`
4. **Alert চেক:** ইচ্ছাকৃতভাবে কোনো rule trigger করাও (যেমন CPU load বাড়িয়ে) → `http://localhost:9093` এ গিয়ে দেখো alert এসেছে কিনা → তারপর Email/Slack/Telegram-এ notification আসছে কিনা চেক করো

---

## ৫. সংক্ষেপে সারাংশ

| Component | কাজ |
|---|---|
| Promtail | Log file পড়ে Loki-তে পাঠায় |
| Loki | Logs store করে |
| Node Exporter | Server-এর hardware metrics দেয় |
| Prometheus | Metrics collect ও store করে, rule evaluate করে |
| Alertmanager | Alert route করে Email/Telegram/Slack-এ পাঠায় |
| Grafana | সব কিছুর visualization dashboard |

এই পুরো setup টা একবার করে ফেললে তুমি একটা জায়গা থেকেই (Grafana) তোমার Django app-এর performance, error, resource usage, এবং raw logs — সব monitor করতে পারবে, আর সমস্যা হলে automatic notification পাবে।










