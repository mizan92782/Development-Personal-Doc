# LEVEL 13 — Docker Deep Dive

## The Core Idea

Docker packages your app + all its dependencies (Python version, system libraries, config) into
one portable unit that runs **identically** anywhere — your laptop, a teammate's laptop, a CI
runner, a production server. It does this using Linux kernel features (namespaces + cgroups) to
give each container an **isolated view** of processes, network, and filesystem, without the
overhead of a full virtual machine.

```
┌─────────────────────────────── Your Machine (Host) ───────────────────────────────┐
│                                                                                       │
│   ┌───────────────────┐  ┌───────────────────┐  ┌───────────────────┐               │
│   │    Container A       │  │    Container B       │  │    Container C       │               │
│   │  (your API)           │  │  (Postgres)           │  │  (Redis)              │               │
│   │                       │  │                       │  │                       │               │
│   │  isolated filesystem   │  │  isolated filesystem   │  │  isolated filesystem   │               │
│   │  isolated processes     │  │  isolated processes     │  │  isolated processes     │               │
│   │  isolated network (own  │  │  isolated network (own  │  │  isolated network (own  │               │
│   │  localhost!)             │  │  localhost!)             │  │  localhost!)             │               │
│   └───────────────────┘  └───────────────────┘  └───────────────────┘               │
│                                                                                       │
│   All share the SAME Linux kernel underneath — this is why containers are            │
│   much lighter than full VMs (no separate OS/kernel per container)                    │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

**Docker vs a VM, in one picture:**
```
Virtual Machine                          Docker Container
┌─────────────┐                         ┌─────────────┐
│    App        │                         │    App        │
├─────────────┤                         ├─────────────┤
│  Guest OS      │  ← full OS, heavy         │  (shares host   │  ← just a process,
│  (own kernel)   │     boots slowly           │   kernel)         │     starts instantly
├─────────────┤                         ├─────────────┤
│  Hypervisor     │                         │  Docker Engine   │
├─────────────┤                         ├─────────────┤
│  Host OS         │                         │  Host OS          │
└─────────────┘                         └─────────────┘
```

---

## 1. Image

A **blueprint** — a read-only template containing your app's filesystem: code, dependencies, OS
libraries, everything needed to run it. Images don't run; **containers** (instances of an image)
do.

```bash
docker images                 # list images you have locally
docker build -t myapp:1.0 .    # build an image from a Dockerfile
docker pull python:3.12-slim    # download an image from a registry
```

Analogy: an image is like a **class**; a container is an **instance** of that class.

## 2. Container

A **running instance** of an image — an isolated process with its own filesystem, network stack,
and process tree, but sharing the host's kernel.

```bash
docker run -d --name myapp_container myapp:1.0    # start a container from an image
docker ps                                           # list running containers
docker ps -a                                        # list ALL containers, including stopped ones
docker stop myapp_container
docker rm myapp_container                            # remove a stopped container
docker exec -it myapp_container bash                  # get an interactive shell INSIDE a running container
```

```
     Image (myapp:1.0)             docker run              docker run
    [ read-only template ]  ──────────────────▶  Container 1 (running)
                             ──────────────────▶  Container 2 (running)
                             ──────────────────▶  Container 3 (running)

    One image → many independent containers, each with its own writable layer on top
```

## 3. Layer

Every instruction in a Dockerfile (`FROM`, `RUN`, `COPY`, ...) creates a **new layer** — a diff on
top of the previous one. Layers are cached and reused across builds, which is why Dockerfile
*order* matters a lot for build speed.

```dockerfile
FROM python:3.12-slim         # Layer 1 — base OS + Python
COPY requirements.txt .        # Layer 2
RUN pip install -r requirements.txt   # Layer 3 — expensive, but cached if requirements.txt unchanged
COPY . .                        # Layer 4 — your actual code (changes often)
```

```
┌─────────────────────────────┐
│  Layer 4: your app code         │  ← changes constantly, rebuilt every time
├─────────────────────────────┤
│  Layer 3: installed pip packages │  ← cached! only rebuilds if requirements.txt changed
├─────────────────────────────┤
│  Layer 2: requirements.txt copied │
├─────────────────────────────┤
│  Layer 1: python:3.12-slim base   │  ← cached, rarely changes
└─────────────────────────────┘

Putting COPY . . AFTER pip install means code changes don't invalidate the expensive
pip install layer — this ordering is the #1 Dockerfile optimization.
```

## 4. Dockerfile

The recipe for building an image, step by step.

```dockerfile
FROM python:3.12-slim              # start from an existing base image

WORKDIR /app                        # set the working directory inside the container

COPY requirements.txt .              # copy just this file first (for layer caching)
RUN pip install --no-cache-dir -r requirements.txt

COPY . .                             # now copy the rest of the app code

ENV PORT=8000                         # set an environment variable inside the image
EXPOSE 8000                            # documents which port the container listens on

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]  # default command on `docker run`
```

## 5. Volume

Containers are **ephemeral** by design — delete a container and its filesystem changes vanish
with it. A volume is a mechanism to **persist data** outside the container's lifecycle (crucial
for databases) or to share files between host and container (great for live-reloading code in
dev).

```bash
docker run -v mydata:/var/lib/postgresql/data postgres    # named volume, managed by Docker
docker run -v $(pwd):/app myapp                             # bind mount — maps a HOST directory directly
```

```
Without a volume:                          With a volume:
┌───────────┐                            ┌───────────┐        ┌──────────┐
│ Container    │ writes data                │ Container    │◀──────▶│  Volume     │  data lives here,
│              │ ──────▶ [lost on           │              │        │  (on host    │  survives container
│  (Postgres)   │           container         │  (Postgres)   │        │   disk)      │  restarts/removal
│              │           removal!)          │              │        │             │
└───────────┘                            └───────────┘        └──────────┘
```

## 6. Network

By default, Docker creates an isolated **virtual network** for your containers to talk to each
other, with its own internal DNS — containers can reach each other **by container/service name**,
not by IP.

```bash
docker network create mynet
docker run --network mynet --name api myapp
docker run --network mynet --name db postgres
# from inside "api" container: connect to "db:5432" — Docker's internal DNS resolves "db" for you
```

```
┌────────────────────────── Docker Network "mynet" ──────────────────────────┐
│                                                                                │
│   ┌───────────┐        can reach each other by NAME       ┌───────────┐         │
│   │ Container "api" │ ─────────────────────────────────▶ │ Container "db"  │         │
│   │             │ ◀───────────────────────────────────── │             │         │
│   └───────────┘        (Docker's built-in DNS resolves     └───────────┘         │
│                          "db" → the db container's internal IP)                    │
└──────────────────────────────────────────────────────────────────────────────┘
```

## 7. Port Mapping

A container's internal port is invisible to the outside world **unless explicitly mapped** to a
port on the host machine. This is the `-p HOST:CONTAINER` flag.

```bash
docker run -p 8080:8000 myapp
#              │     └── port INSIDE the container (what your app is listening on)
#              └──────── port ON YOUR HOST MACHINE (what you actually connect to)
```

```
Host machine                          Container
┌───────────────┐                   ┌───────────────┐
│  localhost:8080  │ ───mapped to───▶ │  0.0.0.0:8000    │  ← your app is listening
│  (what YOU hit    │                   │  (inside the      │     on this port INSIDE
│   in your browser) │                   │   container)       │     the container
└───────────────┘                   └───────────────┘
```

## 8. Environment Variables

Passed into a container at runtime, same concept as Level 12 but scoped to the container —
standard way to inject config/secrets without baking them into the image.

```bash
docker run -e DATABASE_URL="postgres://..." -e DEBUG=false myapp
```

```dockerfile
# Or set a default in the Dockerfile (overridable at `docker run` time)
ENV DEBUG=true
```

## 9. Registry

A place to store and distribute images — Docker Hub is the default public one; companies often
run private registries (AWS ECR, GCP Artifact Registry, self-hosted).

```bash
docker tag myapp:1.0 myregistry.com/myteam/myapp:1.0
docker push myregistry.com/myteam/myapp:1.0     # upload
docker pull myregistry.com/myteam/myapp:1.0      # download (e.g., on a production server)
```

```
┌────────────┐   docker push    ┌─────────────┐   docker pull    ┌────────────┐
│ Dev machine   │ ───────────────▶│  Registry     │───────────────▶│ Production    │
│ (builds image) │                  │ (stores images)│                  │ server         │
└────────────┘                  └─────────────┘                  └────────────┘
```

## 10. Docker Compose

Defines and runs **multi-container** applications with one file + one command, instead of a
sequence of manual `docker run` commands. It creates a shared network automatically and lets
containers reach each other by service name.

```yaml
# docker-compose.yml
services:
  api:
    build: .
    ports:
      - "8080:8000"
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/mydb   # note: "db", NOT "localhost"!
    depends_on:
      - db
      - redis

  db:
    image: postgres:16
    environment:
      - POSTGRES_PASSWORD=pass
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7

volumes:
  pgdata:
```

```bash
docker compose up -d        # build (if needed) and start ALL services
docker compose down          # stop and remove all services
docker compose logs -f api    # follow logs for one service
```

```
┌─────────────────────── docker-compose network ───────────────────────┐
│                                                                          │
│   ┌───────────┐        ┌───────────┐        ┌───────────┐               │
│   │  api           │───────▶│  db            │        │  redis          │               │
│   │  (port 8080     │        │  (postgres)     │        │  (redis)        │               │
│   │   → 8000)        │        │                 │        │                 │               │
│   └───────────┘        └───────────┘        └───────────┘               │
│         │                                                                │
└─────────┼────────────────────────────────────────────────────────────┘
          ▼
   localhost:8080  (what YOU hit from your browser/host)
```

---

## ⚠️ The Big One: `localhost` Inside a Container vs on the Host

This single misunderstanding causes an enormous number of "it works on my machine but not in
Docker" bugs. Here's the key fact:

> **`localhost` inside a container refers to the container itself — NOT the host machine, and
> NOT other containers.**

```
┌─────────────────────── Host Machine ───────────────────────┐
│                                                                │
│   Host's "localhost" = the host machine itself                  │
│                                                                │
│   ┌──────────────────────┐   ┌──────────────────────┐            │
│   │   Container "api"        │   │   Container "db"          │            │
│   │                           │   │                           │            │
│   │  Its OWN "localhost" =    │   │  Its OWN "localhost" =    │            │
│   │  ONLY itself — cannot     │   │  ONLY itself — cannot     │            │
│   │  see the host's           │   │  see the host's           │            │
│   │  localhost, and CANNOT    │   │  localhost, and CANNOT    │            │
│   │  see "db" via localhost    │   │  see "api" via localhost   │            │
│   └──────────────────────┘   └──────────────────────┘            │
│                                                                │
└────────────────────────────────────────────────────────────┘

Each box has its OWN, separate, isolated "localhost" — they are three
completely different "localhost"s that don't refer to the same thing!
```

### Scenario 1 — Connecting from your app INSIDE a container to a DB running on your HOST machine

```python
# BROKEN — "localhost" inside the container means the CONTAINER, not your host machine
DATABASE_URL = "postgres://user:pass@localhost:5432/mydb"
```

Why it fails: your Postgres is running directly on your laptop (not in Docker), listening on your
laptop's `localhost:5432`. But your app, running *inside* a container, has its own isolated
network namespace — its `localhost` points at the container, where nothing is listening on 5432.

**Fix:** use the special DNS name Docker provides for reaching the host from inside a container:

```python
# On Docker Desktop (Mac/Windows) — works out of the box
DATABASE_URL = "postgres://user:pass@host.docker.internal:5432/mydb"
```

```bash
# On Linux, host.docker.internal isn't automatic — add this to docker-compose.yml or `docker run`:
extra_hosts:
  - "host.docker.internal:host-gateway"
```

### Scenario 2 — Two containers trying to talk to each other via `localhost`

```yaml
# docker-compose.yml
services:
  api:
    environment:
      - DATABASE_URL=postgres://user:pass@localhost:5432/mydb   # BROKEN
  db:
    image: postgres:16
```

```
┌───────────────┐                          ┌───────────────┐
│  Container "api" │  "connect to localhost"   │  Container "db"   │
│                   │ ─────────────X            │                   │
│                   │   (fails — api's own       │  Postgres IS       │
│                   │    localhost has no         │  running here,      │
│                   │    postgres running)         │  but "api" never    │
│                   │                             │  actually asked      │
│                   │                             │  to reach IT)         │
└───────────────┘                          └───────────────┘
```

**Fix:** use the **service name** as the hostname — this is what Docker Compose's internal DNS is
for. It's not `localhost`, it's literally the name you gave the service in the compose file:

```yaml
services:
  api:
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/mydb   # "db" = the OTHER container's service name
  db:
    image: postgres:16
```

### Scenario 3 — Confusion about port mapping and `localhost` from your BROWSER

```bash
docker run -p 8080:8000 myapp
```

```
Your browser hits:  http://localhost:8080
                            │
                            │  this "localhost" = YOUR HOST MACHINE
                            │  (this one is fine, because port MAPPING
                            │   bridges host:8080 → container:8000)
                            ▼
                   Docker forwards to container's port 8000
```

This one *works* precisely because `-p 8080:8000` explicitly bridges the host's network to the
container's internal port. Without that mapping, nothing on your host could reach into the
container at all.

### The Rule of Thumb

| From | To | Use |
|---|---|---|
| Your browser/host machine | A container (with `-p` mapping) | `localhost:<host_port>` |
| One container | Another container (same Docker network / same Compose file) | the **service name**, e.g. `db`, `redis` |
| A container | Something running directly on the host (not in Docker) | `host.docker.internal` (with the Linux extra_hosts fix if needed) |
| A container | Itself | `localhost` (rarely what you actually want) |

**One-sentence rule:** inside a container, `localhost` never means "the host machine" and never
means "another container" — it only ever means "this container." Everything else needs an
explicit hostname: the service name (Compose/network DNS) or `host.docker.internal` (host
machine).

---

## Putting It All Together

```
┌──────────────────────────────────────────────────────────────────────┐
│                          docker-compose.yml                              │
│                                                                            │
│   Dockerfile ──build──▶ Image ──run──▶ Container "api" (port 8080→8000)   │
│                                              │                              │
│                                     connects to "db" (service name,          │
│                                     NOT localhost) via the shared              │
│                                     Compose network                             │
│                                              │                              │
│                                              ▼                              │
│                                    Container "db" (Postgres)                  │
│                                              │                              │
│                                     data persisted to a named VOLUME           │
│                                     (survives container restarts)              │
│                                                                            │
│   Image pushed to a REGISTRY so production servers can `docker pull` it      │
└──────────────────────────────────────────────────────────────────────┘
```
