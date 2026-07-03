# AI Agent Prompt — Docker + CI/CD Infrastructure Files Generator

## How to Use

নিচের পুরো prompt টা copy করে যেকোনো AI agent (Claude, ChatGPT, Cursor, etc.) কে দাও।
তোমার project-এর তথ্য দিয়ে `[BRACKETS]` অংশগুলো পরিবর্তন করো।

---

## THE PROMPT

```
আমার একটা [FRAMEWORK] backend project আছে। আমাকে নিচের সব infrastructure file
সম্পূর্ণ implementation সহ তৈরি করে দাও।

## Project Information

- Framework: [Django / FastAPI / Node.js / etc.]
- Python/Node Version: [3.11 / 18 / etc.]
- Project Name: [your-project-name]
- Django/App Module Name: [your_module_name] (e.g. lifechoice)
- WSGI/Entry point: [your_module.wsgi:application]
- Celery App: [yes/no] — App name: [your_module]
- Celery Beat Scheduler: [yes/no]
- Database: PostgreSQL
- Cache/Broker: Redis (with password)
- Static Files: WhiteNoise
- Media Files: AWS S3 (django-storages)
- Web Server (production): Gunicorn
- Container Registry: AWS ECR
- Deployment Target: AWS EC2
- CI/CD: GitHub Actions
- Default App Port: [8000]
- Dev exposed port: [8001]
- Prod exposed port: [8005]

## GitHub Secrets Required (list them in comments)

- SECRET_KEY
- AWS_ACCESS_KEY_ID
- AWS_SECRET_ACCESS_KEY
- AWS_REGION
- ECR_REGISTRY (e.g. 123456789.dkr.ecr.ap-southeast-1.amazonaws.com)
- ECR_REPOSITORY (e.g. my-backend)
- EC2_HOST
- EC2_USER
- EC2_SSH_PRIVATE_KEY
- DEPLOY_PATH (e.g. /home/ubuntu/myapp)

## Files to Generate

### 1. Dockerfile
Requirements:
- Multi-stage build with 3 stages: base, development, production
- base stage: python:[VERSION] — install all requirements
- development stage: FROM base — runserver command
- production stage: python:[VERSION]-slim — copy from base, install gunicorn, run gunicorn
- WORKDIR /app
- Copy requirements.txt first (layer caching)
- No .env file copied (use env_file in compose)

### 2. .dockerignore
Requirements:
- Ignore: __pycache__, *.pyc, .git, .env, venv/, db.sqlite3
- Ignore: .vscode, .idea, *.log, node_modules
- Ignore: docker-compose*.yml, Dockerfile*, *.md
- Ignore: .coverage, htmlcov/, .pytest_cache/
- Keep: requirements.txt, manage.py, all app folders

### 3. docker-compose.dev.yml
Requirements:
- Services: db (postgres:15-alpine), redis (redis:7-alpine), web, celery, celery-beat
- db: named container, healthcheck with pg_isready, volume for data, port 5431:5432
- redis: named container, password via command, port 6378:6379
- web: build target=development, volume mount . :/app (hot reload), port [DEV_PORT]:8000
- web: env_file .env, environment overrides: DEBUG=True, LOCAL_RUN=False, DB_HOST=db, REDIS_HOST=redis
- celery: same build, celery worker command, concurrency=2
- celery-beat: same build, beat command with DatabaseScheduler
- All services depend on db (healthy) and redis (started)
- Named volumes: postgres_data, static_volume, media_volume

### 4. docker-compose.prod.yml
Requirements:
- Services: db, redis, web, celery, celery-beat
- db: postgres:15-alpine, no port exposed (internal only), healthcheck, named volume
- redis: redis:7-alpine, password from env, healthcheck, named volume
- web: image from ECR → [ECR_REGISTRY]/[ECR_REPOSITORY]:${IMAGE_TAG:-latest}
- web: restart always, port [PROD_PORT]:8000, depends on db+redis healthy
- web: command → collectstatic + gunicorn with 3 workers, access/error logs
- web: volume for staticfiles → /home/ubuntu/staticfiles:/app/staticfiles
- celery: same ECR image, worker command
- celery-beat: same ECR image, beat command with DatabaseScheduler
- Named volumes: postgres_data_prod, redis_data_prod
- NO build context in prod (uses pre-built ECR image)

### 5. .github/workflows/prod-ci.yaml
Requirements:
- Trigger: push to main branch
- 4 jobs in order: build → test → push-ecr → deploy

Job 1 — build:
- ubuntu-latest
- Setup Python [VERSION]
- Cache pip dependencies
- pip install -r requirements.txt
- python manage.py check (with dummy env vars)

Job 2 — test:
- needs: build
- PostgreSQL service container (postgres:15-alpine)
- pip install
- python manage.py migrate
- python manage.py test
- flake8 lint (E9, F63, F7, F82 only)

Job 3 — push-ecr:
- needs: test
- Configure AWS credentials from secrets
- Login to ECR
- Set image tag = github.sha
- docker/setup-buildx-action
- docker/build-push-action → target=production, push=true
- Tags: ECR_REGISTRY/ECR_REPOSITORY:sha AND :latest
- Build cache: type=gha

Job 4 — deploy:
- needs: push-ecr
- SCP docker-compose.prod.yml to EC2 (appleboy/scp-action)
- SSH to EC2 (appleboy/ssh-action):
  * aws ecr get-login-password | docker login
  * export IMAGE_TAG
  * docker compose down → pull → up -d
  * docker image prune -f

## Additional Requirements

- All services should have restart: always in production
- Use healthchecks for db and redis in both dev and prod
- Celery and celery-beat should wait for db healthy + redis healthy
- In CI test job, use dummy SECRET_KEY and env vars (no real secrets)
- Comments in YAML files explaining each section
- .env.example file showing all required environment variables

## .env.example to Generate

Include all these variables with placeholder values:
- SECRET_KEY, DEBUG, LOCAL_RUN, ALLOWED_HOSTS
- POSTGRES_DB, POSTGRES_USER, POSTGRES_PASSWORD, DB_HOST, DB_PORT
- REDIS_HOST, REDIS_PORT, REDIS_PASSWORD, REDIS_DB
- EMAIL_BACKEND, EMAIL_HOST, EMAIL_PORT, EMAIL_USE_TLS, EMAIL_HOST_USER, EMAIL_HOST_PASSWORD
- AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_STORAGE_BUCKET_NAME, AWS_S3_REGION_NAME, USE_S3
- CORS_ALLOWED_ORIGINS, CSRF_TRUSTED_ORIGINS
- Any other project-specific variables

Generate all files with complete, production-ready implementation.
No placeholder code — everything must work as-is after filling in real values.
```

---

## তোমার Project-এর জন্য Fill করা Values

এই project (lifechoice) এর জন্য values:

| Variable | Value |
|----------|-------|
| Framework | Django |
| Python Version | 3.11 |
| Project Name | lifechoice |
| Django Module | lifechoice |
| WSGI | lifechoice.wsgi:application |
| Celery App | lifechoice |
| Celery Beat | Yes |
| Dev Port | 8001 |
| Prod Port | 8005 |
| ECR Registry | 797671034027.dkr.ecr.ap-southeast-1.amazonaws.com |
| ECR Repository | ikon-backend |
| AWS Region | ap-southeast-1 |

---

## GitHub Secrets Setup

GitHub → Repository → Settings → Secrets and variables → Actions → New repository secret

| Secret Name | Value |
|-------------|-------|
| `AWS_ACCESS_KEY_ID` | IAM User Access Key |
| `AWS_SECRET_ACCESS_KEY` | IAM User Secret Key |
| `AWS_REGION` | ap-southeast-1 |
| `ECR_REGISTRY` | 797671034027.dkr.ecr.ap-southeast-1.amazonaws.com |
| `ECR_REPOSITORY` | ikon-backend |
| `EC2_HOST` | EC2 Public IP or domain |
| `EC2_USER` | ubuntu |
| `EC2_SSH_PRIVATE_KEY` | .pem file content |
| `DEPLOY_PATH` | /home/ubuntu/lifechoice |

---

## CI/CD Flow (কীভাবে কাজ করে)

```
git push → main branch
        ↓
Job 1: BUILD
  → pip install
  → manage.py check
        ↓
Job 2: TEST
  → PostgreSQL container start
  → migrate + test + lint
        ↓
Job 3: PUSH TO ECR
  → Docker build (production stage)
  → Tag with git SHA
  → Push to AWS ECR
        ↓
Job 4: DEPLOY TO EC2
  → SCP docker-compose.prod.yml to server
  → SSH into server
  → Pull new image from ECR
  → docker compose up -d
  → Prune old images
```

---

## নতুন Project-এ ব্যবহার করতে

1. উপরের prompt copy করো
2. `[BRACKETS]` এর জায়গায় তোমার project info দাও
3. AI agent-কে দাও
4. Generated files গুলো project-এ রাখো
5. GitHub Secrets set করো
6. `git push origin main` করো — বাকি সব automatic
