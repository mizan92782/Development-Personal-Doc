# LEVEL 19 — Cloud & Deployment Deep Dive

## The Big Picture: How a Domain Name Becomes Your Running App

```
User types "yourapp.com" in browser
          │
          ▼
┌─────────────────┐
│      DNS            │  "yourapp.com" → 203.0.113.42  (translates name to IP address)
└────────┬────────┘
          ▼
┌─────────────────┐
│  Firewall/Security  │  "is this traffic allowed to reach the server at all?"
│  Group                │
└────────┬────────┘
          ▼
┌─────────────────┐
│   Load Balancer      │  distributes traffic across multiple servers
└────────┬────────┘
          ▼
┌─────────────────┐
│      Server           │  runs your app (Nginx → Gunicorn → Django, from Level 14)
│  (SSH to manage it,     │
│   SSL/HTTPS for the      │
│   incoming connection)    │
└────────┬────────┘
          ▼
┌─────────────────┐   ┌─────────────────┐
│      Database         │   │  Object Storage    │  (uploaded files, images, backups)
└─────────────────┘   └─────────────────┘

Deployment gets your CODE into this picture:
Your code ──▶ CI/CD pipeline ──▶ Container Registry ──▶ pulled by the Server
```

---

# PART A — Core Concepts

## 1. Server

A physical or virtual machine that runs your application. In the cloud, this is almost always a
**virtual machine** — a slice of a physical machine's resources, rented by the hour/second.

```
Physical machine (in a data center)
┌─────────────────────────────────────────┐
│   VM 1 (yours)   VM 2 (someone else's)      │
│   VM 3 (yours)   VM 4 (someone else's)      │
└─────────────────────────────────────────┘
   the cloud provider manages the physical hardware; you just get an isolated VM slice
```

## 2. DNS (Domain Name System)

The internet's phone book — translates human-readable names into IP addresses, since that's what
computers actually route traffic by.

```bash
dig yourapp.com          # query DNS, see what IP it resolves to
nslookup yourapp.com
```

```
yourapp.com  ──[DNS lookup]──▶  A record: 203.0.113.42
www.yourapp.com ──[DNS lookup]──▶  CNAME record: yourapp.com  (alias, resolves the same)
```

## 3. Domain

The human-readable name itself (`yourapp.com`), which you register through a **registrar**
(Namecheap, GoDaddy, Route 53) — registering it doesn't host anything, it just reserves the name
and lets you point its DNS records wherever you want.

## 4. IP Address

The actual numeric address a machine is reachable at on a network.

```
IPv4: 203.0.113.42                (4 numbers, 0-255 each — the address space is running out)
IPv6: 2001:0db8::ff00:0042:8329   (much larger address space, gradually replacing IPv4)

Public IP:  reachable from the internet
Private IP: only reachable within a private network (e.g., 10.0.0.5 inside a VPC — see below)
```

## 5. Firewall

A rule set controlling what network traffic is allowed in/out of a machine, based on port,
protocol, and source/destination IP.

```
Rule examples:
  ALLOW  inbound TCP port 443 from anywhere        (public HTTPS traffic)
  ALLOW  inbound TCP port 22  from YOUR_IP only      (SSH — restricted to your IP for security)
  DENY   everything else inbound
```

## 6. Security Group (AWS-specific concept)

AWS's version of a firewall, but attached to individual resources (EC2 instances) rather than the
whole network — and unlike a traditional firewall's explicit deny rules, security groups are
**allow-list only**: everything not explicitly allowed is denied by default.

```
Security Group: "web-server-sg"
  Inbound:
    ALLOW  443 (HTTPS)  from  0.0.0.0/0   (anywhere)
    ALLOW  22  (SSH)     from  YOUR_IP/32   (just you)
  Outbound:
    ALLOW  all traffic   to    0.0.0.0/0   (server can reach out anywhere, e.g. to call APIs)
```

## 7. SSL / 8. HTTPS

Covered in depth in Level 14 — SSL/TLS certificates enable encrypted communication (HTTPS);
without them traffic between client and server is plaintext and interceptable.

```bash
# Getting a free cert via Let's Encrypt, using certbot
sudo certbot --nginx -d yourapp.com
```

## 9. SSH

Secure Shell — an encrypted protocol for remotely logging into and controlling a server.

```bash
ssh -i mykey.pem ubuntu@203.0.113.42     # connect using a private key (never a plain password in prod)
```

```
Your laptop                                    Remote server
┌──────────┐   encrypted tunnel over TCP 22    ┌──────────┐
│  private key │ ─────────────────────────────────▶│  public key   │  the server verifies your
└──────────┘                                    └──────────┘  private key matches a
                                                                registered public key
```

## 10. Load Balancer

Covered in Level 14/17 — distributes traffic across multiple servers for scalability and
availability.

## 11. Object Storage

A storage system for **files/blobs** (images, videos, backups, static assets) — NOT a
filesystem or database, but a flat namespace of key → file mappings, designed to be extremely
durable and to scale to essentially unlimited size without you managing disks at all.

```
Object Storage (e.g., AWS S3)
┌───────────────────────────────────────┐
│  bucket: "myapp-uploads"                    │
│    key: "avatars/user_42.jpg"  → [binary data] │
│    key: "documents/invoice_99.pdf" → [binary]  │
└───────────────────────────────────────┘

Accessed via HTTP(S), not a mounted filesystem:
https://myapp-uploads.s3.amazonaws.com/avatars/user_42.jpg
```

**Why not just store uploaded files on the server's own disk?** Servers are disposable in cloud
architectures (autoscaling, replacements) — anything stored only on one server's local disk is
lost if that server is replaced, and can't be shared across multiple servers/replicas anyway.

## 12. Database (managed)

Covered extensively already — in the cloud, you'd typically use a **managed** database service
(handles backups, patching, replication for you) rather than running Postgres yourself on a bare
server.

## 13. Container Registry

A place to store and distribute Docker images — covered in Level 13's "Registry" section. AWS's
version is **ECR** (below).

## 14. CI/CD

**Continuous Integration** — automatically running tests/checks on every code change.
**Continuous Deployment/Delivery** — automatically shipping code that passes those checks to
production (or staging, awaiting approval).

```
Developer pushes code
        │
        ▼
┌────────────────────┐
│   CI: run tests,        │   if any step fails, STOP — bad code never reaches production
│   linting, build the      │
│   Docker image             │
└──────────┬─────────┘
            ▼
┌────────────────────┐
│   Push image to a        │
│   Container Registry      │
└──────────┬─────────┘
            ▼
┌────────────────────┐
│   CD: deploy the new      │
│   image to production      │
│   (or staging first)        │
└────────────────────┘
```

```yaml
# .github/workflows/deploy.yml — a minimal example
name: Deploy
on:
  push:
    branches: [main]
jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        run: pytest
      - name: Build Docker image
        run: docker build -t myapp:${{ github.sha }} .
      - name: Push to registry
        run: |
          docker tag myapp:${{ github.sha }} myregistry/myapp:latest
          docker push myregistry/myapp:latest
      - name: Deploy
        run: ssh deploy@server "docker pull myregistry/myapp:latest && docker compose up -d"
```

**Why it exists:** removes manual, error-prone deployment steps ("did someone forget to run
migrations? did they test before pushing?") and makes shipping code frequent and low-risk, since
every change goes through the same automated, repeatable gate.

---

# PART B — AWS Fundamentals

## EC2 (Elastic Compute Cloud)

Virtual servers, rentable by the hour/second — the basic "Server" concept from Part A, as an AWS
product. This is where your app would actually run (or where you'd run Docker containers, if not
using a more managed compute service).

```
┌─────────────────────────────────┐
│   EC2 Instance (t3.medium)          │
│   - 2 vCPUs, 4GB RAM                  │
│   - Ubuntu 22.04                       │
│   - has a Security Group attached       │
│   - lives inside a VPC/subnet             │
│   - Public IP: 203.0.113.42                │
└─────────────────────────────────┘
```

```bash
# Typical EC2 workflow
ssh -i mykey.pem ubuntu@<ec2-public-ip>
git clone <repo> && cd repo
docker compose up -d
```

**Instance types** trade off CPU/memory/network for cost — `t3.micro` for a small dev box,
`c5.xlarge` for CPU-heavy workloads, `r5.large` for memory-heavy workloads, etc.

## S3 (Simple Storage Service)

AWS's object storage product — the concrete implementation of the "Object Storage" concept above.

```python
import boto3

s3 = boto3.client("s3")
s3.upload_file("local_photo.jpg", "myapp-uploads", "avatars/user_42.jpg")

url = s3.generate_presigned_url(
    "get_object",
    Params={"Bucket": "myapp-uploads", "Key": "avatars/user_42.jpg"},
    ExpiresIn=3600,   # temporary, expiring URL — for private files you don't want permanently public
)
```

```
Common patterns:
  - Store user uploads (images, documents)
  - Host static website assets (often paired with a CDN — Level 17 — like CloudFront)
  - Store application backups, logs, data exports
  - Serve as the "data lake" for big data pipelines
```

## RDS (Relational Database Service)

A **managed** relational database (Postgres, MySQL, etc.) — AWS handles backups, patching,
failover, and replication setup, so you don't have to run and babysit the database server
yourself.

```
┌─────────────────────────────────────────┐
│                    RDS                       │
│   Automated: backups, patching, failover,       │
│   read replica creation, monitoring               │
│                                                │
│   You just get a connection string:              │
│   postgres://user:pass@mydb.xxxx.rds.amazonaws.com:5432/mydb │
└─────────────────────────────────────────┘
```

```python
DATABASE_URL = "postgres://user:pass@mydb.abc123.us-east-1.rds.amazonaws.com:5432/mydb"
```

**Why use RDS instead of Postgres on an EC2 instance yourself?** You'd otherwise have to build
and maintain your own backup strategy, patching schedule, failover process, and replica setup —
RDS gives you all of that with a few clicks/config options, at the cost of less low-level control
than a self-managed instance.

## IAM (Identity and Access Management)

Controls **who** (users, services, applications) can do **what** (which actions) on **which**
AWS resources. This is the permission system underlying literally everything else in AWS.

```json
// An IAM policy — allows read/write access to ONE specific S3 bucket, nothing else
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::myapp-uploads/*"
    }
  ]
}
```

```
┌──────────┐     assumes      ┌──────────┐    has attached    ┌──────────┐
│  EC2         │ ────────────────▶│   IAM Role      │ ───────────────────▶│  IAM Policy    │
│  instance     │                │  (an identity,     │                    │  (defines exact  │
│               │                │   not a person)      │                    │   permissions)     │
└──────────┘                └──────────┘                    └──────────┘

Your app running on EC2 can access S3 WITHOUT hardcoding any AWS access keys —
it inherits the permissions of the IAM Role attached to the instance.
```

**Why it matters:** the #1 security principle here is **least privilege** — grant only the exact
permissions a resource needs, nothing more. An app that only needs to read from one S3 bucket
should never have a role that can delete any resource in the entire AWS account.

## VPC (Virtual Private Cloud)

An isolated virtual network within AWS where your resources live — lets you control IP ranges,
subnets, and routing, essentially building your own private network inside the cloud.

```
┌────────────────────────── VPC (10.0.0.0/16) ──────────────────────────┐
│                                                                            │
│   ┌──────────── Public Subnet (10.0.1.0/24) ────────────┐                  │
│   │   EC2 (web server) — has a public IP, reachable          │                  │
│   │   from the internet via the Load Balancer                  │                  │
│   └────────────────────────────────────────────┘                  │
│                                                                            │
│   ┌──────────── Private Subnet (10.0.2.0/24) ───────────┐                  │
│   │   RDS database — NO public IP, only reachable            │                  │
│   │   from resources INSIDE the VPC (like the web server)       │                  │
│   └────────────────────────────────────────────┘                  │
└────────────────────────────────────────────────────────────────┘
```

**Why the public/private subnet split matters:** your database should never be directly reachable
from the public internet — putting it in a private subnet means the only way to reach it at all
is through your app servers (in the public subnet), drastically shrinking the attack surface.

## CloudWatch

AWS's monitoring/observability service (Level 18's concepts, as an AWS product) — collects
metrics, logs, and lets you set alarms across your AWS resources.

```
EC2 CPU utilization  ──▶ CloudWatch Metric  ──▶ CloudWatch Alarm (CPU > 80% for 5 min)
                                                          │
                                                          ▼
                                              Trigger: send SNS notification,
                                              or auto-scale (add more EC2 instances)
```

```python
import boto3
cloudwatch = boto3.client("cloudwatch")
cloudwatch.put_metric_data(
    Namespace="MyApp",
    MetricData=[{"MetricName": "OrdersProcessed", "Value": 1, "Unit": "Count"}],
)
```

## ECR (Elastic Container Registry)

AWS's container registry — the concrete AWS implementation of "Container Registry" from Part A,
where your Docker images live before being deployed.

```bash
aws ecr get-login-password | docker login --username AWS --password-stdin <account>.dkr.ecr.<region>.amazonaws.com
docker tag myapp:latest <account>.dkr.ecr.<region>.amazonaws.com/myapp:latest
docker push <account>.dkr.ecr.<region>.amazonaws.com/myapp:latest
```

---

# Putting It All Together — A Typical AWS Deployment

```
                              ┌─────────────┐
  Developer pushes code ────▶ │  CI/CD           │  runs tests, builds Docker image
                              │  (GitHub Actions)  │
                              └──────┬──────┘
                                      │  docker push
                                      ▼
                              ┌─────────────┐
                              │     ECR          │  (Container Registry)
                              └──────┬──────┘
                                      │  pulled by
                                      ▼
              ┌─────────────────── VPC ───────────────────┐
              │                                              │
              │   Public Subnet:                                │
              │   ┌───────────┐        ┌───────────┐              │
              │   │  Load Balancer│───────▶│  EC2 (app)     │              │
              │   └───────────┘        │  (Security Group   │              │
              │                        │   allows :443)       │              │
              │                        └─────┬─────┘              │
              │                                │                    │
              │   Private Subnet:              │                    │
              │   ┌───────────────────────▼─┐                │
              │   │   RDS (Postgres) — no public IP              │                │
              │   └─────────────────────────┘                │
              │                                              │
              └──────────────────────────────────────────┘
                                      │
                       app also talks to (outside the VPC):
                                      ▼
                              ┌─────────────┐        ┌─────────────┐
                              │      S3          │        │  CloudWatch    │
                              │  (uploaded files)  │        │  (monitoring,   │
                              └─────────────┘        │   alarms)        │
                                                      └─────────────┘

              Everything above is governed by IAM: which EC2 instances can
              read/write which S3 buckets, which services can publish metrics, etc.
```

**The one-sentence takeaway:** cloud providers don't introduce fundamentally new concepts beyond
what you already know from Levels 12–17 (servers, networks, storage, databases, monitoring) —
they package them as managed, on-demand, API-driven products (EC2, S3, RDS, CloudWatch) so you
don't have to rack physical hardware, and IAM/VPC/Security Groups are simply how access and
network isolation are enforced across all of them consistently.
