# GitHub Actions — Django CI/CD Pipeline সম্পূর্ণ গাইড (বাংলা)

---

## এই Workflow কী করে?

তুমি যখন GitHub এ code push করো, তখন এই pipeline স্বয়ংক্রিয়ভাবে:

```
Code Push হয়
     ↓
Lint চেক (code style ঠিক আছে?)
     ↓
Test চলে (সব test pass হয়?)
     ↓
Docker Image build হয় (শুধু main branch)
     ↓
EC2 তে Deploy হয় (শুধু main branch)
     ↓
S3 তে Backup যায় (শুধু main branch)
```

---

## ফাইল স্ট্রাকচার

```
.github/
└── workflows/
    └── test1.yml    ← এই ফাইলটা
```

GitHub `.github/workflows/` ফোল্ডারের যেকোনো `.yml` ফাইল স্বয়ংক্রিয়ভাবে workflow হিসেবে চেনে।

---

## ধাপ ১ — Workflow এর নাম

```yaml
name: Django Full CI/CD Pipeline
```

GitHub Actions UI তে এই নামে workflow দেখাবে।

---

## ধাপ ২ — Events (কখন চলবে?)

```yaml
on:
  push:
    branches:
      - dev
      - main

  pull_request:
    branches:
      - dev
      - main

  workflow_dispatch:
```

| Event | মানে |
|---|---|
| `push` | `dev` বা `main` branch এ code push হলে চলবে |
| `pull_request` | কেউ PR খুললে চলবে |
| `workflow_dispatch` | GitHub UI থেকে manually চালানো যাবে |

> **কেন `dev` আর `main` দুটোতে?** `dev` এ feature develop হয়, `main` এ production যায়। দুটোতেই lint ও test চলা দরকার।

---

## ধাপ ৩ — Concurrency (একসাথে একটাই চলবে)

```yaml
concurrency:
  group: django-ci-${{ github.ref }}
  cancel-in-progress: true
```

**সমস্যা:** তুমি দ্রুত ৩টা commit push করলে ৩টা workflow একসাথে চলতে শুরু করবে — এটা resource নষ্ট।

**সমাধান:** `cancel-in-progress: true` মানে নতুন workflow শুরু হলে পুরোনোটা বাতিল হয়ে যাবে।

`${{ github.ref }}` মানে branch এর নাম — তাই `main` আর `dev` এর workflow আলাদাভাবে track হয়।

---

## ধাপ ৪ — Global Environment Variables

```yaml
env:
  PYTHON_VERSION: "3.12"

  DB_NAME: test_db
  DB_USER: postgres
  DB_PASSWORD: postgres
  DB_HOST: localhost
  DB_PORT: 5432

  DOCKER_IMAGE: yourdockerhubusername/django-app
```

এগুলো **সব job এ** ব্যবহার করা যাবে `${{ env.VARIABLE_NAME }}` দিয়ে।

> **লক্ষ্য করো:** এখানে `DB_PASSWORD: postgres` দেওয়া আছে কারণ এটা শুধু **test এর জন্য** — real production password না। Real secret গুলো GitHub Secrets এ রাখতে হবে।

---

## ধাপ ৫ — LINT JOB (কোড স্টাইল চেক)

```yaml
lint:
  name: Lint & Format Check
  runs-on: ubuntu-latest
  timeout-minutes: 10
  strategy:
    matrix:
      python-version: [3.11, 3.12]
```

### `runs-on: ubuntu-latest`
GitHub এর server এ Ubuntu Linux এ চলবে। GitHub এই machine ফ্রিতে দেয়।

### `timeout-minutes: 10`
১০ মিনিটের বেশি লাগলে job বন্ধ হয়ে যাবে। Infinite loop বা hang থেকে বাঁচায়।

### `strategy.matrix`
```yaml
matrix:
  python-version: [3.11, 3.12]
```
এটা দিয়ে **একই job দুইবার চলে** — একবার Python 3.11 এ, একবার 3.12 এ।
মানে তুমি নিশ্চিত হতে পারবে যে তোমার code দুটো version এই কাজ করে।

---

### Lint Job এর Steps

#### Step 1 — Checkout Code
```yaml
- name: Checkout Code
  uses: actions/checkout@v4
```
GitHub এর server এ তোমার repository এর code নামিয়ে আনে।
`@v4` মানে এই action এর version 4 ব্যবহার করছে।

#### Step 2 — Setup Python
```yaml
- name: Setup Python
  uses: actions/setup-python@v5
  with:
    python-version: ${{ matrix.python-version }}
```
Server এ Python install করে। `matrix.python-version` থেকে version নেয় — তাই 3.11 ও 3.12 দুটোতেই চলে।

#### Step 3 — Cache Dependencies
```yaml
- name: Cache Dependencies
  uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: ${{ runner.os }}-${{ matrix.python-version }}-${{ hashFiles('**/requirements.txt') }}
```

**Cache কী?** প্রতিবার `pip install` চালালে internet থেকে package নামাতে হয় — এটা সময় নেয়।
Cache দিয়ে একবার নামানো package সংরক্ষণ করা হয়।

**Key কীভাবে কাজ করে:**
- `runner.os` → Ubuntu
- `matrix.python-version` → 3.11 বা 3.12
- `hashFiles('**/requirements.txt')` → requirements.txt এর hash

যদি requirements.txt না বদলায়, key একই থাকে → cache hit → pip install skip হয় → **build দ্রুত হয়।**

#### Step 4 — Install Dependencies
```yaml
- name: Install Dependencies
  run: |
    python -m pip install --upgrade pip
    pip install -r requirements.txt
    pip install flake8 black isort
```

`flake8`, `black`, `isort` — এই তিনটা lint tool install হয়।

#### Step 5 — Flake8
```yaml
- name: Run Flake8
  run: flake8 .
```
**Flake8** Python code এর syntax error, unused import, line too long এই ধরনের সমস্যা ধরে।

#### Step 6 — Black
```yaml
- name: Black Format Check
  run: black . --check
```
**Black** code formatting চেক করে। `--check` মানে শুধু চেক করবে, বদলাবে না।
যদি formatting ঠিক না থাকে, job fail হবে।

#### Step 7 — isort
```yaml
- name: Import Sorting Check
  run: isort . --check-only
```
**isort** Python import গুলো alphabetically sorted কিনা চেক করে।

---

## ধাপ ৬ — TEST JOB

```yaml
test:
  name: Django Test
  needs: lint
```

### `needs: lint`
Lint job সফল না হলে test job শুরুই হবে না। এটা **dependency chain** তৈরি করে।

```
lint ✅ → test শুরু হয়
lint ❌ → test চলে না
```

### PostgreSQL Service
```yaml
services:
  postgres:
    image: postgres:16
    env:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: test_db
    ports:
      - 5432:5432
    options: >-
      --health-cmd pg_isready
      --health-interval 10s
      --health-timeout 5s
      --health-retries 5
```

**Services** মানে job এর পাশাপাশি আরেকটা container চলে।
এখানে PostgreSQL 16 এর একটা container চলে যেটায় Django connect করে test করে।

**Health check কেন?**
PostgreSQL start হতে কিছু সময় লাগে। Health check নিশ্চিত করে যে database ready হওয়ার পরেই Django connect করার চেষ্টা করবে।

| Option | মানে |
|---|---|
| `--health-cmd pg_isready` | এই command দিয়ে check করে DB ready কিনা |
| `--health-interval 10s` | প্রতি ১০ সেকেন্ডে check করে |
| `--health-timeout 5s` | ৫ সেকেন্ডের মধ্যে response না পেলে fail |
| `--health-retries 5` | ৫ বার try করবে |

### Wait for PostgreSQL
```yaml
- name: Wait for PostgreSQL
  run: sleep 10
```
Health check এর পরেও extra ১০ সেকেন্ড wait — নিরাপদ থাকার জন্য।

### Run Migrations
```yaml
- name: Run Migrations
  run: python manage.py migrate
  env:
    DB_NAME: ${{ env.DB_NAME }}
    DB_USER: ${{ env.DB_USER }}
    DB_PASSWORD: ${{ env.DB_PASSWORD }}
    DB_HOST: ${{ env.DB_HOST }}
    DB_PORT: ${{ env.DB_PORT }}
```

Global `env` থেকে DB credentials নিয়ে migration চালায়।

### Run Django Tests
```yaml
- name: Run Django Tests
  run: python manage.py test
```
তোমার সব Django test চলে। কোনো test fail করলে এই job fail হয় এবং পরের job গুলো চলে না।

---

## ধাপ ৭ — DOCKER BUILD JOB

```yaml
docker-build:
  name: Docker Build
  needs: test
  runs-on: ubuntu-latest
  if: github.ref == 'refs/heads/main'
```

### `if: github.ref == 'refs/heads/main'`
**শুধু main branch এ** এই job চলবে। `dev` branch এ push করলে Docker build হবে না।

> **কেন?** `dev` branch এ feature develop হয়, সেটা production এ deploy করা ঠিক না।

### DockerHub Login
```yaml
- name: Login DockerHub
  uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKER_USERNAME }}
    password: ${{ secrets.DOCKER_PASSWORD }}
```

`secrets.DOCKER_USERNAME` — এটা GitHub Secrets থেকে আসে (নিচে বিস্তারিত আছে)।

### Image Build ও Tag
```yaml
- name: Build Docker Image
  run: docker build -t $DOCKER_IMAGE:${{ github.sha }} .

- name: Tag Latest
  run: docker tag $DOCKER_IMAGE:${{ github.sha }} $DOCKER_IMAGE:latest
```

**দুটো tag কেন?**

| Tag | মানে | ব্যবহার |
|---|---|---|
| `${{ github.sha }}` | Commit এর unique ID (যেমন `a3f9b2c`) | কোন commit থেকে image তৈরি সেটা track করতে |
| `latest` | সবচেয়ে নতুন image | Deploy করার সময় সহজে pull করতে |

### Push to DockerHub
```yaml
- name: Push SHA Image
  run: docker push $DOCKER_IMAGE:${{ github.sha }}

- name: Push Latest Image
  run: docker push $DOCKER_IMAGE:latest
```

DockerHub এ দুটো tag push হয়।

---

## ধাপ ৮ — DEPLOY JOB

```yaml
deploy:
  name: Deploy Production
  needs: docker-build
  if: github.ref == 'refs/heads/main'
  environment:
    name: production
```

### `environment: name: production`
GitHub এ "production" নামে একটা environment তৈরি করা যায় যেখানে:
- Deployment history দেখা যায়
- Required reviewers সেট করা যায় (deploy এর আগে কাউকে approve করতে হবে)

### SSH Deploy
```yaml
- name: SSH Deploy
  uses: appleboy/ssh-action@v1.0.3
  with:
    host: ${{ secrets.EC2_HOST }}
    username: ${{ secrets.EC2_USERNAME }}
    key: ${{ secrets.EC2_SSH_KEY }}
    script: |
      docker pull $DOCKER_IMAGE:latest
      docker stop django-app || true
      docker rm django-app || true
      docker run -d \
        --name django-app \
        -p 8000:8000 \
        --env-file .env \
        $DOCKER_IMAGE:latest
```

**`appleboy/ssh-action`** — এটা একটা community action যেটা SSH দিয়ে remote server এ command চালায়।

**Deploy script কী করে:**
1. DockerHub থেকে latest image pull করে
2. পুরোনো container বন্ধ করে (`|| true` মানে container না থাকলেও error দেবে না)
3. পুরোনো container মুছে ফেলে
4. নতুন container চালু করে

---

## ধাপ ৯ — S3 BACKUP JOB

```yaml
backup-to-s3:
  name: Backup to S3
  needs: test
  if: github.ref == 'refs/heads/main'
```

`needs: test` — test pass হলেই backup চলবে।
`docker-build` এর উপর depend করে না — তাই docker-build আর backup **একসাথে** চলতে পারে।

```yaml
- name: Create ZIP
  run: zip -r project.zip .

- name: Configure AWS
  uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: ap-south-1

- name: Upload to S3
  run: aws s3 cp project.zip s3://your-bucket-name/
```

পুরো project zip করে S3 bucket এ upload করে।

---

## Job Execution Flow

```
push to main
      |
      ↓
   lint job
  (3.11 & 3.12)
      |
      ↓ (lint pass হলে)
   test job
  (3.11 & 3.12)
      |
      |─────────────────────┐
      ↓                     ↓
docker-build job      backup-to-s3 job
(একসাথে চলে)          (একসাথে চলে)
      |
      ↓ (docker-build pass হলে)
   deploy job
```

`docker-build` আর `backup-to-s3` একসাথে চলে কারণ দুটোই `test` এর উপর depend করে, একে অপরের উপর না।

---

## GitHub Secrets — কোনটা কেন রাখবে

GitHub Secrets হলো encrypted storage যেখানে sensitive তথ্য রাখা হয়।
**Settings → Secrets and variables → Actions → New repository secret**

| Secret নাম | কী রাখবে | কেন Secret? |
|---|---|---|
| `DOCKER_USERNAME` | তোমার DockerHub username | Public হলে কেউ তোমার account এ image push করতে পারবে |
| `DOCKER_PASSWORD` | DockerHub password বা Access Token | Password কখনো code এ রাখা যাবে না |
| `EC2_HOST` | EC2 server এর IP বা domain | Server IP জানলে attack করা সহজ হয় |
| `EC2_USERNAME` | EC2 login username (যেমন `ubuntu`) | Server access এর তথ্য |
| `EC2_SSH_KEY` | EC2 এর private SSH key (পুরো key) | এটা ফাঁস হলে যে কেউ server এ login করতে পারবে |
| `AWS_ACCESS_KEY_ID` | AWS IAM user এর Access Key ID | AWS resource access করার credential |
| `AWS_SECRET_ACCESS_KEY` | AWS IAM user এর Secret Key | এটা ফাঁস হলে AWS account এ সব করা যাবে |

### DockerHub Access Token কীভাবে বানাবে?
1. DockerHub → Account Settings → Security
2. New Access Token → নাম দাও → Generate
3. এই token টা `DOCKER_PASSWORD` হিসেবে save করো (password এর চেয়ে নিরাপদ)

### EC2 SSH Key কীভাবে পাবে?
```bash
# নতুন key তৈরি করতে
ssh-keygen -t rsa -b 4096 -f github-actions-key

# private key (github-actions-key) → GitHub Secret এ রাখো
# public key (github-actions-key.pub) → EC2 এর ~/.ssh/authorized_keys এ রাখো
```

### AWS IAM Key কীভাবে বানাবে?
1. AWS Console → IAM → Users → তোমার user
2. Security credentials → Create access key
3. Access Key ID → `AWS_ACCESS_KEY_ID`
4. Secret Access Key → `AWS_SECRET_ACCESS_KEY`

> **সতর্কতা:** IAM user কে শুধু S3 এর permission দাও — সব permission দেওয়া বিপজ্জনক।

---

## কোনটা Secret, কোনটা Normal env?

```yaml
# এগুলো Normal env — secret না, test এর জন্য
env:
  DB_NAME: test_db
  DB_USER: postgres
  DB_PASSWORD: postgres   # শুধু test DB, real না

# এগুলো Secret — কখনো code এ লেখা যাবে না
secrets.DOCKER_USERNAME
secrets.DOCKER_PASSWORD
secrets.EC2_HOST
secrets.EC2_SSH_KEY
secrets.AWS_ACCESS_KEY_ID
secrets.AWS_SECRET_ACCESS_KEY
```

---

## Actions ব্যাখ্যা — কোনটা কী করে

| Action | কাজ |
|---|---|
| `actions/checkout@v4` | Repository এর code নামিয়ে আনে |
| `actions/setup-python@v5` | Python install করে |
| `actions/cache@v4` | pip cache সংরক্ষণ করে, build দ্রুত করে |
| `docker/login-action@v3` | DockerHub এ login করে |
| `appleboy/ssh-action@v1.0.3` | SSH দিয়ে remote server এ command চালায় |
| `aws-actions/configure-aws-credentials@v4` | AWS CLI configure করে |

---

## Workflow চালু করার আগে যা করতে হবে

### ১. `DOCKER_IMAGE` বদলাও
```yaml
# এটা বদলাও
DOCKER_IMAGE: yourdockerhubusername/django-app

# তোমার DockerHub username দিয়ে
DOCKER_IMAGE: mizanur/django-app
```

### ২. S3 bucket নাম বদলাও
```yaml
# এটা বদলাও
aws s3 cp project.zip s3://your-bucket-name/

# তোমার bucket নাম দিয়ে
aws s3 cp project.zip s3://my-django-backup/
```

### ৩. GitHub Secrets সেট করো
উপরের টেবিল দেখে সব secret add করো।

### ৪. EC2 তে Docker install করো
```bash
sudo apt update
sudo apt install docker.io -y
sudo usermod -aG docker ubuntu
```

### ৫. EC2 তে `.env` ফাইল রাখো
```bash
# EC2 তে
nano ~/.env
# অথবা
scp .env ubuntu@your-ec2-ip:~/.env
```

---

## সম্পূর্ণ Flow একনজরে

```
তুমি main branch এ push করলে
              |
              ↓
    GitHub Actions শুরু হয়
              |
    ┌─────────────────────┐
    │      LINT JOB       │
    │  Python 3.11 & 3.12 │
    │  flake8, black,     │
    │  isort চেক          │
    └─────────┬───────────┘
              │ pass
              ↓
    ┌─────────────────────┐
    │      TEST JOB       │
    │  Python 3.11 & 3.12 │
    │  PostgreSQL service │
    │  migrate + test     │
    └─────────┬───────────┘
              │ pass
       ┌──────┴──────┐
       ↓             ↓
┌──────────────┐  ┌──────────────┐
│ DOCKER BUILD │  │  S3 BACKUP   │
│ image build  │  │  zip + upload│
│ push to hub  │  └──────────────┘
└──────┬───────┘
       │ pass
       ↓
┌──────────────┐
│    DEPLOY    │
│  SSH to EC2  │
│  pull image  │
│  run container│
└──────────────┘
```

---

## সাধারণ সমস্যা ও সমাধান

| সমস্যা | কারণ | সমাধান |
|---|---|---|
| Lint fail | code formatting ঠিক নেই | `black .` ও `isort .` locally চালাও |
| Test fail | DB connect হচ্ছে না | `sleep 10` বাড়িয়ে `sleep 20` করো |
| Docker push fail | Secret ভুল | DockerHub credentials চেক করো |
| SSH fail | Key ভুল বা EC2 ready না | EC2 security group এ port 22 open আছে কিনা দেখো |
| S3 upload fail | IAM permission নেই | IAM user এ S3 write permission দাও |
