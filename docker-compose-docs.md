# Docker Compose সম্পূর্ণ ডকুমেন্টেশন (বাংলা)

> এই ফাইলে `docker-compose.yml` এর প্রতিটা line কী করে, কেন দরকার, এবং না থাকলে কী সমস্যা হবে — সব বিস্তারিত বাংলায় লেখা আছে।

---

## Docker Compose কী?

`docker-compose.yml` একটা configuration ফাইল যেটা দিয়ে একসাথে **একাধিক container** manage করা যায়।

আমাদের app এ ৩টা service আছে:
- `web` → Django app
- `db` → PostgreSQL database
- `redis` → Redis cache

এই তিনটা আলাদা আলাদা container এ চলে কিন্তু একসাথে কাজ করে।

```bash
# একটা command এ সব চালু হয়
docker-compose up --build
```

---

## সম্পূর্ণ ফাইল স্ট্রাকচার

```
docker-compose.yml
├── services
│   ├── web        → Django application
│   ├── db         → PostgreSQL database
│   └── redis      → Redis cache
├── volumes        → Persistent data storage
└── networks       → Isolated communication
```

---

## `services` — সার্ভিস কী?

```yaml
services:
```

- প্রতিটা service মানে একটা আলাদা container
- প্রতিটার নিজস্ব image, config, network আছে
- এগুলো একে অপরের সাথে কথা বলতে পারে

> না থাকলে: Docker Compose কিছুই বুঝবে না — এটা root level এর required field

---

## `web` Service — Django App

---

### `build`

```yaml
build:
  context: .
  dockerfile: Dockerfile
```

- `context: .` → current folder থেকে build করবে
- `dockerfile: Dockerfile` → কোন Dockerfile ব্যবহার করবে সেটা বলে দেয়

> না থাকলে: Docker জানবে না image কোথা থেকে build করবে — container উঠবেই না

---

### `container_name`

```yaml
container_name: project_api
```

- container এর একটা নির্দিষ্ট নাম দেয়
- না দিলে Docker random নাম দেয় যেমন `devops-learning_web_1`
- এই নাম দিয়ে `docker logs project_api` বা `docker exec -it project_api sh` চালানো যায়

> না থাকলে: random নাম পাবে, manage করতে কষ্ট হবে

---

### `env_file`

```yaml
env_file:
  - .env
```

- `.env` ফাইলের সব variable container এ inject করে
- যেমন `DJANGO_ENV=production`, `POSTGRES_PASSWORD=postgres123`
- এই variable গুলো `entrypoint.sh` এ `$DJANGO_ENV` হিসেবে access হয়

> না থাকলে: `DJANGO_ENV` পাবে না, `entrypoint.sh` এ condition কাজ করবে না, database connect হবে না

---

### `ports`

```yaml
ports:
  - "8000:8000"
```

- format হলো `host_port:container_port`
- তোমার machine এর `8000` port কে container এর `8000` port এর সাথে connect করে
- browser এ `http://localhost:8000` দিলে container এর Django app এ যাবে

> না থাকলে: container চলবে কিন্তু browser থেকে access করা যাবে না

---

### `restart`

```yaml
restart: always
```

- container crash করলে বা server reboot হলে automatically restart হবে
- options:
  - `no` → কখনো restart হবে না
  - `always` → সবসময় restart হবে
  - `on-failure` → শুধু error এ restart হবে
  - `unless-stopped` → manually stop না করলে সবসময় চলবে

> না থাকলে: server reboot হলে বা container crash করলে manually `docker-compose up` চালাতে হবে

---

### `depends_on`

```yaml
depends_on:
  db:
    condition: service_healthy
  redis:
    condition: service_healthy
```

- `web` container চালু হওয়ার আগে `db` এবং `redis` কে **healthy** হতে হবে
- `condition: service_healthy` মানে শুধু container start হলেই হবে না — healthcheck pass করতে হবে
- শুধু `depends_on: - db` লিখলে container start হলেই web চলে যেত, কিন্তু postgres ready না হলে Django crash করত

> না থাকলে: web container আগে উঠে যাবে, database ready হওয়ার আগেই connect করতে যাবে, `connection refused` error আসবে

---

### `healthcheck` (web)

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8000"]
  interval: 30s
  timeout: 10s
  retries: 5
  start_period: 20s
```

- প্রতি `30s` এ একবার `curl` দিয়ে Django app check করে
- `timeout: 10s` → ১০ সেকেন্ডের মধ্যে response না আসলে failed ধরবে
- `retries: 5` → ৫ বার fail হলে container কে `unhealthy` mark করবে
- `start_period: 20s` → প্রথম ২০ সেকেন্ড check করবে না, Django কে boot হওয়ার সময় দেবে

> না থাকলে: Django crash করলেও Docker জানবে না, `depends_on` এর condition কাজ করবে না

---

### `deploy.resources`

```yaml
deploy:
  resources:
    limits:
      cpus: "1.0"
      memory: 512M
    reservations:
      cpus: "0.25"
      memory: 256M
```

- `limits` → container সর্বোচ্চ এতটুকু CPU/memory নিতে পারবে
- `reservations` → container এর জন্য এতটুকু সবসময় reserved থাকবে
- একটা container সব resource খেয়ে ফেললে অন্য container গুলো মরে যায়

> না থাকলে: Django app memory leak হলে বা traffic বাড়লে পুরো server এর memory শেষ হয়ে যাবে, সব container crash করবে

---

### `logging`

```yaml
logging:
  driver: "json-file"
  options:
    max-size: "10m"
    max-file: "5"
```

- `json-file` → Docker এর default log driver, log JSON format এ রাখে
- `max-size: "10m"` → একটা log file সর্বোচ্চ ১০MB হবে
- `max-file: "5"` → সর্বোচ্চ ৫টা log file রাখবে, পুরনো delete হবে

> না থাকলে: log file unlimited বাড়তে থাকবে, কয়েক সপ্তাহে disk full হয়ে যাবে, server down হবে

---

### `networks` (web)

```yaml
networks:
  - app_network
```

- web container কে `app_network` এ যুক্ত করে
- এই network এ থাকা container গুলো নাম দিয়ে একে অপরকে চেনে
- Django settings এ `HOST=db` লিখলেই postgres connect হবে, IP লাগবে না

> না থাকলে: container গুলো একে অপরকে খুঁজে পাবে না, database connection fail করবে

---

## `db` Service — PostgreSQL

---

### `image`

```yaml
image: postgres:16
```

- Docker Hub থেকে official PostgreSQL 16 image নামায়
- নিজে build করতে হয় না — ready-made image ব্যবহার হয়

> না থাকলে: database থাকবে না, Django এর সব data কোথায় যাবে?

---

### `volumes` (db)

```yaml
volumes:
  - postgres_data:/var/lib/postgresql/data/
```

- PostgreSQL এর সব data `/var/lib/postgresql/data/` এ থাকে
- এটা `postgres_data` named volume এ map করা হয়েছে
- container delete করলেও data থাকবে কারণ volume আলাদা জায়গায় থাকে

> না থাকলে: `docker-compose down` করলে সব database data চিরতরে মুছে যাবে

---

### `healthcheck` (db)

```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U $POSTGRES_USER -d $POSTGRES_DB"]
  interval: 10s
  timeout: 5s
  retries: 5
  start_period: 10s
```

- `pg_isready` → PostgreSQL এর built-in tool যেটা connection check করে
- প্রতি `10s` এ check করে postgres সত্যিকারের connection নিতে পারছে কিনা
- `$POSTGRES_USER` এবং `$POSTGRES_DB` `.env` থেকে আসে

> না থাকলে: `depends_on condition: service_healthy` কাজ করবে না, web container আগে উঠে database crash করবে

---

## `redis` Service — Cache

---

### `image`

```yaml
image: redis:7
```

- Docker Hub থেকে official Redis 7 image নামায়
- Session, cache, Celery task queue এর জন্য ব্যবহার হয়

---

### `volumes` (redis)

```yaml
volumes:
  - redis_data:/data
```

- Redis এর data `/data` এ থাকে
- container restart হলেও cache data থাকবে

> না থাকলে: container restart হলে সব cache মুছে যাবে, user session হারিয়ে যাবে, সবাইকে আবার login করতে হবে

---

### `healthcheck` (redis)

```yaml
healthcheck:
  test: ["CMD", "redis-cli", "ping"]
  interval: 10s
  timeout: 5s
  retries: 5
  start_period: 5s
```

- `redis-cli ping` → Redis `PONG` দিয়ে respond করলে healthy
- web container এর `depends_on` এই healthcheck এর উপর নির্ভর করে

> না থাকলে: Redis ready হওয়ার আগেই Django connect করতে যাবে, Celery বা cache error আসবে

---

## `volumes` — Persistent Storage

```yaml
volumes:
  postgres_data:
  redis_data:
```

- এগুলো **named volume** — Docker নিজে manage করে
- container এর বাইরে আলাদা জায়গায় data রাখে
- `docker-compose down` করলেও data থাকে
- শুধু `docker-compose down -v` করলে volume সহ মুছে যায়

> না থাকলে: প্রতিবার container restart এ সব data হারাবে — production এ এটা বিপর্যয়কর

---

## `networks` — Isolated Communication

```yaml
networks:
  app_network:
    driver: bridge
```

- `bridge` driver → একটা virtual network তৈরি করে
- এই network এর বাইরে থেকে `db` বা `redis` directly access করা যাবে না
- container গুলো service name দিয়ে একে অপরকে চেনে:
  - Django settings এ `DATABASE_HOST=db`
  - Redis settings এ `REDIS_HOST=redis`

> না থাকলে: container গুলো isolate থাকবে না, বাইরে থেকে database directly attack করা সম্ভব হবে

---

## সব একসাথে — Startup Flow

```
docker-compose up --build
        |
        ↓
db container চালু হয়
        |
        ↓ (healthcheck চলে প্রতি 10s)
pg_isready pass করে → db = healthy
        |
        ↓
redis container চালু হয়
        |
        ↓ (healthcheck চলে প্রতি 10s)
redis-cli ping → PONG → redis = healthy
        |
        ↓
web container চালু হয় (db ও redis healthy হওয়ার পর)
        |
        ↓
entrypoint.sh চলে
        |
        ↓
collectstatic → migrate → gunicorn/runserver
        |
        ↓
localhost:8000 এ app ready
```

---

## সমস্যা ও সমাধান একনজরে

| Configuration | না থাকলে কী হবে |
|---|---|
| `env_file` | DJANGO_ENV পাবে না, database connect হবে না |
| `depends_on condition` | Database ready হওয়ার আগে Django crash করবে |
| `healthcheck` | Crash detect হবে না, depends_on কাজ করবে না |
| `restart: always` | Server reboot এ manually চালু করতে হবে |
| `volumes` | Container restart এ সব data মুছে যাবে |
| `networks` | Container গুলো একে অপরকে খুঁজে পাবে না |
| `deploy.resources` | একটা container সব memory খেয়ে সব crash করাবে |
| `logging limits` | Log file disk ভরিয়ে server down করবে |
| `ports` | Browser থেকে app access করা যাবে না |

---

## Run করার Command

```bash
# প্রথমবার বা Dockerfile বদলালে
docker-compose up --build

# পরের বার
docker-compose up

# Background এ চালাতে
docker-compose up -d

# সব বন্ধ করতে
docker-compose down

# সব বন্ধ + volume মুছতে (সব data যাবে)
docker-compose down -v

# logs দেখতে
docker-compose logs -f web
```
