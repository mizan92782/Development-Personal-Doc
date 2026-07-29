# AWS সার্ভিস সেটআপ গাইড — RDS, ElastiCache, S3 (বাংলা)

> এই ডকুমেন্টে Django প্রজেক্টে AWS RDS (Database), ElastiCache (Redis Cache), এবং S3 (Static/Media Files) সম্পূর্ণ সেটআপ বাংলায় ধাপে ধাপে বর্ণনা করা হয়েছে।

---

## কেন AWS Managed Service ব্যবহার করব?

| বিষয় | Docker Container | AWS Managed Service |
|---|---|---|
| Database backup | নিজে করতে হবে | AWS automatically করে |
| Scaling | নিজে করতে হবে | AWS automatically করে |
| Security patch | নিজে করতে হবে | AWS automatically করে |
| High availability | কঠিন | built-in |
| Data loss risk | বেশি | অনেক কম |

---

## সম্পূর্ণ Architecture

```
Browser
   |
   ↓
EC2 / ECS (Docker container — শুধু web app)
   |
   |── RDS PostgreSQL    (database)
   |── ElastiCache Redis (cache/session)
   └── S3 Bucket         (static/media files)
```

---

## ধাপ ১ — AWS RDS (PostgreSQL) সেটআপ

### ১.১ RDS Instance তৈরি

1. AWS Console → **RDS** → **Create database**
2. নিচের options select করো:

```
Engine:              PostgreSQL
Version:             16
Template:            Free tier (dev) / Production (prod)
DB instance class:   db.t3.micro (free tier)
Storage:             20 GB (gp2)
```

3. Credentials সেট করো:

```
Master username:     postgres
Master password:     <strong-password>
```

4. Connectivity সেট করো:

```
VPC:                    default বা তোমার VPC
Public access:          No (EC2 থেকে private access)
VPC security group:     নতুন একটা তৈরি করো — নাম দাও "rds-sg"
```

5. **Create database** click করো — ৫-১০ মিনিট লাগবে

### ১.২ RDS Security Group সেট করো

RDS তে connect করতে EC2 এর Security Group কে allow করতে হবে:

1. AWS Console → **EC2** → **Security Groups**
2. `rds-sg` select করো → **Inbound rules** → **Edit inbound rules**
3. নিচের rule যোগ করো:

```
Type:        PostgreSQL
Protocol:    TCP
Port:        5432
Source:      EC2 এর Security Group ID (sg-xxxxxxxx)
```

> এটা না করলে Django থেকে RDS তে connect হবে না — `connection refused` error আসবে

### ১.৩ RDS Endpoint নাও

1. RDS Console → তোমার database select করো
2. **Connectivity & security** tab এ যাও
3. **Endpoint** copy করো — এরকম দেখাবে:

```
mydb.xxxxxxxxxx.ap-southeast-1.rds.amazonaws.com
```

### ১.৪ `.env` এ RDS Credentials বসাও

```env
DATABASE_HOST=mydb.xxxxxxxxxx.ap-southeast-1.rds.amazonaws.com
DATABASE_PORT=5432
DATABASE_NAME=mydb
DATABASE_USER=postgres
DATABASE_PASSWORD=<your-rds-password>
```

### ১.৫ Django `settings.py` তে Database Config

```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "HOST": os.environ.get("DATABASE_HOST"),
        "PORT": os.environ.get("DATABASE_PORT"),
        "NAME": os.environ.get("DATABASE_NAME"),
        "USER": os.environ.get("DATABASE_USER"),
        "PASSWORD": os.environ.get("DATABASE_PASSWORD"),
    }
}
```

### ১.৬ `requirements.txt` এ যোগ করো

```
psycopg2-binary
```

---

## ধাপ ২ — AWS ElastiCache (Redis) সেটআপ

### ২.১ ElastiCache Cluster তৈরি

1. AWS Console → **ElastiCache** → **Create cluster**
2. নিচের options select করো:

```
Cluster engine:      Redis
Cluster mode:        Disabled (simple setup)
Node type:           cache.t3.micro (free tier)
Number of replicas:  0 (dev) / 1 (prod)
```

3. Connectivity সেট করো:

```
VPC:                 RDS এর মতো same VPC
Subnet group:        নতুন তৈরি করো
Security group:      নতুন একটা তৈরি করো — নাম দাও "redis-sg"
```

4. **Create** click করো

### ২.২ ElastiCache Security Group সেট করো

1. `redis-sg` select করো → **Inbound rules** → **Edit inbound rules**
2. নিচের rule যোগ করো:

```
Type:        Custom TCP
Protocol:    TCP
Port:        6379
Source:      EC2 এর Security Group ID (sg-xxxxxxxx)
```

> এটা না করলে Django থেকে Redis তে connect হবে না

### ২.৩ ElastiCache Endpoint নাও

1. ElastiCache Console → তোমার cluster select করো
2. **Primary endpoint** copy করো — এরকম দেখাবে:

```
myredis.xxxxxx.0001.apse1.cache.amazonaws.com
```

### ২.৪ `.env` এ Redis Credentials বসাও

```env
REDIS_HOST=myredis.xxxxxx.0001.apse1.cache.amazonaws.com
REDIS_PORT=6379
```

### ২.৫ Django `settings.py` তে Redis Config

```python
CACHES = {
    "default": {
        "BACKEND": "django_redis.cache.RedisCache",
        "LOCATION": f"redis://{os.environ.get('REDIS_HOST')}:{os.environ.get('REDIS_PORT')}/0",
        "OPTIONS": {
            "CLIENT_CLASS": "django_redis.client.DefaultClient",
        }
    }
}

# Session এও Redis use করতে চাইলে
SESSION_ENGINE = "django.contrib.sessions.backends.cache"
SESSION_CACHE_ALIAS = "default"
```

### ২.৬ `requirements.txt` এ যোগ করো

```
django-redis
```

---

## ধাপ ৩ — AWS S3 (Static & Media Files) সেটআপ

### ৩.১ S3 Bucket তৈরি

1. AWS Console → **S3** → **Create bucket**
2. নিচের options দাও:

```
Bucket name:         my-django-app-static
Region:              ap-southeast-1 (তোমার region)
Block public access: OFF করো (static files public হতে হবে)
```

3. **Create bucket** click করো

### ৩.২ S3 Bucket Policy সেট করো

Static files publicly readable করতে হবে:

1. Bucket → **Permissions** → **Bucket policy** → **Edit**
2. নিচের policy paste করো:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::my-django-app-static/static/*"
        }
    ]
}
```

> Media files private রাখতে চাইলে `static/*` শুধু রাখো, `media/*` এ আলাদা permission দাও

### ৩.৩ IAM User তৈরি করো (S3 Access এর জন্য)

Django কে S3 তে upload করতে হলে একটা IAM user দরকার:

1. AWS Console → **IAM** → **Users** → **Create user**
2. Username দাও: `django-s3-user`
3. **Attach policies directly** → **AmazonS3FullAccess** select করো
4. User তৈরির পর → **Security credentials** → **Create access key**
5. `Access key ID` এবং `Secret access key` copy করো — একবারই দেখাবে

> IAM user না বানিয়ে EC2 IAM Role ব্যবহার করা আরো secure — production এ সেটাই করা উচিত

### ৩.৪ `.env` এ S3 Credentials বসাও

```env
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
AWS_STORAGE_BUCKET_NAME=my-django-app-static
AWS_S3_REGION_NAME=ap-southeast-1
```

### ৩.৫ Django `settings.py` তে S3 Config

```python
import os

if os.environ.get("AWS_STORAGE_BUCKET_NAME"):
    # S3 configured থাকলে S3 use করবে
    DEFAULT_FILE_STORAGE = "storages.backends.s3boto3.S3Boto3Storage"
    STATICFILES_STORAGE = "storages.backends.s3boto3.S3StaticStorage"

    AWS_ACCESS_KEY_ID = os.environ.get("AWS_ACCESS_KEY_ID")
    AWS_SECRET_ACCESS_KEY = os.environ.get("AWS_SECRET_ACCESS_KEY")
    AWS_STORAGE_BUCKET_NAME = os.environ.get("AWS_STORAGE_BUCKET_NAME")
    AWS_S3_REGION_NAME = os.environ.get("AWS_S3_REGION_NAME")
    AWS_S3_FILE_OVERWRITE = False
    AWS_DEFAULT_ACL = None

    STATIC_URL = f"https://{AWS_STORAGE_BUCKET_NAME}.s3.amazonaws.com/static/"
    MEDIA_URL = f"https://{AWS_STORAGE_BUCKET_NAME}.s3.amazonaws.com/media/"
else:
    # S3 না থাকলে local filesystem use করবে
    STATIC_URL = "/static/"
    STATIC_ROOT = BASE_DIR / "staticfiles"
    MEDIA_URL = "/media/"
    MEDIA_ROOT = BASE_DIR / "media"
```

### ৩.৬ `requirements.txt` এ যোগ করো

```
django-storages
boto3
```

---

## ধাপ ৪ — সম্পূর্ণ `.env` ফাইল

```env
DJANGO_ENV=production

# AWS RDS PostgreSQL
DATABASE_HOST=mydb.xxxxxxxxxx.ap-southeast-1.rds.amazonaws.com
DATABASE_PORT=5432
DATABASE_NAME=mydb
DATABASE_USER=postgres
DATABASE_PASSWORD=<your-rds-password>

# AWS ElastiCache Redis
REDIS_HOST=myredis.xxxxxx.0001.apse1.cache.amazonaws.com
REDIS_PORT=6379

# AWS S3
AWS_ACCESS_KEY_ID=<your-access-key-id>
AWS_SECRET_ACCESS_KEY=<your-secret-access-key>
AWS_STORAGE_BUCKET_NAME=my-django-app-static
AWS_S3_REGION_NAME=ap-southeast-1
```

---

## ধাপ ৫ — সম্পূর্ণ `requirements.txt`

```
django
gunicorn
psycopg2-binary
django-redis
django-storages
boto3
```

---

## ধাপ ৬ — Deploy করো

```bash
# AWS compose file দিয়ে build এবং run
docker-compose -f docker-compose.aws.yml --env-file .env up --build
```

Container চালু হলে `entrypoint.sh` এ:

```
migrate চলবে     → RDS এ schema তৈরি হবে
collectstatic    → S3 bucket এ static files upload হবে
gunicorn চালু    → app ready
```

---

## সমস্যা ও সমাধান

| সমস্যা | কারণ | সমাধান |
|---|---|---|
| `connection refused` RDS | Security Group এ 5432 open নেই | EC2 SG কে RDS SG তে allow করো |
| `connection refused` Redis | Security Group এ 6379 open নেই | EC2 SG কে Redis SG তে allow করো |
| S3 upload fail | IAM permission নেই | AmazonS3FullAccess policy দাও |
| Static files 403 | Bucket policy নেই | Bucket policy তে GetObject allow করো |
| `collectstatic` fail | AWS credentials `.env` এ নেই | `.env` এ credentials check করো |

---

## Security Best Practices

- `.env` ফাইল কখনো GitHub এ push করো না — `.gitignore` এ রাখো
- Production এ IAM User এর বদলে **EC2 IAM Role** ব্যবহার করো — credentials `.env` এ রাখতে হবে না
- RDS এবং ElastiCache **Public access: No** রাখো — শুধু EC2 থেকে access হবে
- S3 Bucket এ শুধু `static/*` public রাখো, `media/*` private রাখো
- RDS password strong রাখো এবং AWS Secrets Manager এ store করো

---

## সম্পূর্ণ Flow একনজরে

```
docker-compose -f docker-compose.aws.yml up --build
        |
        ↓
web container চালু হয়
        |
        ↓
entrypoint.sh চলে
        |
        ├── migrate → RDS PostgreSQL এ schema তৈরি
        |
        ├── collectstatic → S3 Bucket এ static files upload
        |
        ↓
gunicorn চালু হয়
        |
        ↓
Browser request আসে
        |
        ├── Dynamic data → RDS থেকে
        ├── Cache/Session → ElastiCache থেকে
        └── Static/Media → S3 থেকে সরাসরি
```
