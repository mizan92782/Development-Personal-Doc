# DevOps-Learning — Docker + Django সম্পূর্ণ গাইড (বাংলা)

---

## প্রজেক্ট স্ট্রাকচার

```
DevOps-Learning/
├── Dockerfile          # Docker image তৈরির নির্দেশনা
├── entrypoint.sh       # Container চালু হলে যা execute হয়
├── .env                # Environment variable গুলো
├── requirements.txt    # Python প্যাকেজ লিস্ট
└── README.md           # এই ফাইল
```

---

## Docker কী এবং কেন দরকার?

সাধারণত একটা Django app চালাতে হলে তোমার মেশিনে Python, pip, সব dependency ইনস্টল করতে হয়।
কিন্তু অন্য কারো মেশিনে গেলে সেটা কাজ নাও করতে পারে — কারণ তার Python version আলাদা, OS আলাদা।

**Docker এই সমস্যা সমাধান করে।**

Docker একটা **container** তৈরি করে যেটার ভেতরে:

- Python আছে
- সব dependency আছে
- তোমার পুরো project আছে

এই container যেকোনো মেশিনে একইভাবে চলবে। Development থেকে Production — সব জায়গায় same behavior।

---

## ধাপ ১ — `.env` ফাইল

```env
DJANGO_ENV=production
```

- এই ফাইলে **secret** এবং **environment-specific** তথ্য রাখা হয়
- `DJANGO_ENV=production` মানে app টা production mode এ চলবে
- এই ফাইল কখনো Docker image এর ভেতরে copy হয় না — শুধু runtime এ inject হয়
- এটা `.dockerignore` এ রাখা উচিত যাতে image এ secret ঢুকে না যায়

> **কেন আলাদা রাখি?** কারণ SECRET_KEY, DATABASE_PASSWORD এই ধরনের তথ্য code এ রাখলে GitHub এ চলে যেতে পারে — সেটা বিপজ্জনক।

---

## ধাপ ২ — `Dockerfile`

```dockerfile
FROM python:3.11-slim
```

- Docker Hub থেকে official Python 3.11 এর slim version নামায়
- slim মানে অপ্রয়োজনীয় OS package ছাড়া — image size ছোট থাকে

```dockerfile
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
```

- `PYTHONDONTWRITEBYTECODE=1` → Python `.pyc` bytecode ফাইল তৈরি করবে না — container এ এগুলো দরকার নেই
- `PYTHONUNBUFFERED=1` → Python এর log সাথে সাথে দেখা যাবে, buffer এ আটকে থাকবে না

```dockerfile
WORKDIR /app
```

- Container এর ভেতরে `/app` folder কে working directory বানায়
- এরপর সব command এই folder থেকে চলবে

```dockerfile
COPY requirements.txt .
RUN pip install --upgrade pip && pip install -r requirements.txt
```

- আগে শুধু `requirements.txt` copy করা হয়
- তারপর সব Python package install করা হয়
- **কেন আগে?** Docker layer caching এর জন্য — requirements না বদলালে পরের build এ pip install আবার চলবে না, সময় বাঁচবে

```dockerfile
COPY . .
```

- পুরো project code container এ copy হয়
- এটা dependency install এর পরে করা হয় — কারণ code বারবার বদলায়, কিন্তু dependency কম বদলায়

```dockerfile
EXPOSE 8000
```

- Container যে port এ listen করবে সেটা document করে
- এটা শুধু documentation — actual port publish হয় `docker run -p` দিয়ে

```dockerfile
RUN chmod +x /app/entrypoint.sh
ENTRYPOINT ["/app/entrypoint.sh"]
```

- `entrypoint.sh` কে executable করা হয়
- Container start হলে এই script টা প্রথমে run হবে

---

## ধাপ ৩ — `entrypoint.sh`

```sh
#!/bin/sh

python manage.py collectstatic --noinput

if [ "$DJANGO_ENV" = "production" ]
then
    gunicorn project.wsgi:application --bind 0.0.0.0:8000
else
    python manage.py runserver 0.0.0.0:8000
fi
```

Container চালু হলে এই script execute হয়:

**`collectstatic`**

- Django এর সব static file (CSS, JS, images) এক জায়গায় collect করে
- এটা build time এ না করে runtime এ করা হয় কারণ তখন `SECRET_KEY` সহ সব env var available থাকে

**`if [ "$DJANGO_ENV" = "production" ]`**

- `.env` থেকে inject হওয়া `DJANGO_ENV` variable চেক করে
- যদি `production` হয় → **gunicorn** দিয়ে server চালায়
- যদি অন্য কিছু হয় → Django এর built-in **runserver** দিয়ে চালায়

**gunicorn vs runserver:**
| | gunicorn | runserver |
|---|---|---|
| ব্যবহার | Production | Development |
| Performance | অনেক বেশি | কম |
| Multiple request | একসাথে handle করতে পারে | পারে না |
| Security | Production ready | নিরাপদ না |

---

## ধাপ ৪ — Image Build করা

```bash
docker build -t my-django-app .
```

এই command চালালে Docker:

```
1. python:3.11-slim image নামায় Docker Hub থেকে
2. ENV variable set করে
3. /app folder তৈরি করে
4. requirements.txt copy করে
5. pip দিয়ে সব package install করে
6. পুরো project copy করে
7. entrypoint.sh কে executable করে
8. সব মিলিয়ে একটা IMAGE তৈরি করে
```

Image মানে একটা **snapshot** — এটা তোমার পুরো app এর ready-to-run প্যাকেজ।

---

## ধাপ ৫ — Container Run করা

```bash
docker run --env-file .env -p 8000:8000 my-django-app
```

- `--env-file .env` → `.env` ফাইলের variable গুলো container এ inject করে
- `-p 8000:8000` → তোমার machine এর 8000 port কে container এর 8000 port এর সাথে connect করে
- এরপর `entrypoint.sh` চলে এবং server উঠে যায়

তারপর browser এ যাও: `http://localhost:8000`

---

## সম্পূর্ণ Flow একনজরে

```
তুমি code লিখলে
        |
        ↓
docker build → Dockerfile পড়ে → Image তৈরি হয়
        |
        | (image এ আছে: Python + packages + তোমার code)
        |
        ↓
docker run --env-file .env -p 8000:8000
        |
        | (.env থেকে DJANGO_ENV inject হয়)
        |
        ↓
Container চালু হয়
        |
        ↓
entrypoint.sh execute হয়
        |
        ↓
collectstatic চলে
        |
        ↓
DJANGO_ENV চেক হয়
        |
   _____|_____
  |           |
production   অন্য
  |           |
gunicorn   runserver
  |           |
  ↓           ↓
localhost:8000 এ app চলে
```

---

## গুরুত্বপূর্ণ বিষয়গুলো মনে রাখো

| বিষয়                       | কারণ                             |
| --------------------------- | -------------------------------- |
| `.env` image এ copy হয় না  | Secret safe রাখতে                |
| `requirements.txt` আগে copy | Docker cache কাজে লাগাতে         |
| `collectstatic` runtime এ   | Build time এ env var থাকে না     |
| `gunicorn` production এ     | Performance ও security এর জন্য   |
| `EXPOSE 8000`               | Documentation, actual publish না |

---

## Docker Compose দিয়ে আরো সহজে (Optional)

```yaml
# docker-compose.yml
services:
  web:
    build: .
    env_file:
      - .env
    ports:
      - "8000:8000"
```

```bash
docker-compose up
```

এই একটা command এই সব হয়ে যাবে — build, env inject, port map, container start।
