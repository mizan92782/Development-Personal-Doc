# LEVEL 12 — Linux for Backend Developers

Your app doesn't run in a vacuum — it runs as a **process**, on an **OS**, talking through
**sockets**, reading **file descriptors**, controlled by **permissions**, often managed as a
**service** by `systemd`. Knowing the commands is only useful once you understand what they're
actually manipulating underneath.

```
┌───────────────────────────────────────────────────────────────┐
│                          LINUX KERNEL                             │
│                                                                    │
│   ┌───────────┐   ┌───────────┐   ┌───────────┐                    │
│   │ Process A   │   │ Process B   │   │ Process C   │  ← your apps       │
│   │ (your API)   │   │ (Postgres)   │   │ (Redis)      │                    │
│   └─────┬─────┘   └─────┬─────┘   └─────┬─────┘                    │
│         │                │                │                          │
│    file descriptors,  file descriptors, file descriptors,              │
│    sockets, memory     sockets, memory   sockets, memory               │
│         │                │                │                          │
│   ┌─────▼────────────────▼────────────────▼─────┐                    │
│   │         Kernel: scheduling, memory,             │                    │
│   │         networking, filesystem, permissions      │                    │
│   └─────────────────────────────────────────────┘                    │
└───────────────────────────────────────────────────────────────┘
```

---

# PART A — Core Concepts

## 1. Process

A process is a **running instance of a program** — it has its own memory space, its own PID
(process ID), and its own set of open file descriptors.

```bash
ps aux                 # list all running processes
ps aux | grep python   # find your python processes specifically
```

```
PID   USER   %CPU  %MEM  COMMAND
1234  alice   2.1   1.5  python manage.py runserver
1235  alice   0.0   0.3  postgres
1236  alice   0.1   0.2  redis-server
```

Every process has a **parent process** (the one that spawned it) — this forms a tree, viewable
with `pstree`. When your app server (e.g. gunicorn) forks worker processes, those workers are
children of the master process.

## 2. Thread

A thread is a unit of execution **within** a process. Threads in the same process **share memory**
(unlike separate processes, which don't) — this is why threads are lighter-weight but also why
they need synchronization (locks) to avoid corrupting shared data.

```
┌─────────────────────────── Process (PID 1234) ───────────────────────────┐
│                                                                              │
│   Shared memory: heap, global variables, open file descriptors               │
│                                                                              │
│   ┌───────────┐    ┌───────────┐    ┌───────────┐                            │
│   │  Thread 1   │    │  Thread 2   │    │  Thread 3   │  ← each has its own      │
│   │  (own stack) │    │  (own stack) │    │  (own stack) │     execution stack     │
│   └───────────┘    └───────────┘    └───────────┘                            │
└──────────────────────────────────────────────────────────────────────────┘
```

```bash
ps -eLf | grep gunicorn   # -L shows individual threads (LWP column) within processes
```

## 3. Port

A port is a **numbered endpoint** (0–65535) on a machine that identifies which application should
receive incoming network traffic. Your API listening on port 8000, Postgres on 5432, Redis on
6379 — the kernel routes incoming packets to the right process based on the destination port.

```bash
sudo lsof -i :8000          # what process is listening on port 8000?
sudo ss -tulpn | grep 8000  # modern alternative to netstat
```

```
Incoming request to  192.168.1.10:8000
                              │
                              ▼
                     ┌─────────────────┐
                     │  Kernel routing   │  "port 8000 → PID 1234 (your API)"
                     └─────────────────┘
                              │
                              ▼
                     Your API process receives it
```

## 4. Socket

A socket is the actual **communication endpoint** — the combination of an IP address + port + protocol
that a process uses to send/receive data over a network (or even between processes on the same
machine, via Unix domain sockets). "Opening a connection" means creating a socket.

```python
# What "listening on port 8000" looks like at the socket level
import socket

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)   # create a TCP socket
s.bind(("0.0.0.0", 8000))                                 # bind it to a port
s.listen()                                                 # start listening for connections
conn, addr = s.accept()                                    # accept an incoming connection
```

Every open TCP connection is a socket (technically a `(local IP, local port, remote IP, remote
port)` tuple) — this is why a server can handle thousands of simultaneous client connections all
on the same listening port: each individual connection is a distinct socket.

## 5. File Descriptor

Everything in Linux — actual files, network sockets, pipes, even stdin/stdout/stderr — is
represented as a **file descriptor**: a small integer the kernel uses as a handle to that
resource.

```
FD 0  →  stdin
FD 1  →  stdout
FD 2  →  stderr
FD 3  →  (the next one your program opens — e.g. a log file, a DB connection socket, ...)
```

```bash
ls -l /proc/1234/fd/    # list every open file descriptor for process 1234
```

```python
f = open("app.log", "w")     # kernel allocates a new file descriptor for this
print(f.fileno())            # e.g. 3 — the actual integer handle
```

**Why this matters operationally:** every open file *and* every open socket/DB connection counts
against a process's file descriptor limit (`ulimit -n`). "Too many open files" errors under load
usually mean connections aren't being closed properly (a classic connection-pool leak).

## 6. Environment Variable

Key-value pairs available to a process, inherited from its parent, commonly used for
configuration (DB URLs, secret keys, feature flags) — kept out of source code intentionally.

```bash
export DATABASE_URL="postgres://user:pass@localhost/mydb"
echo $DATABASE_URL
env                        # list all env vars for the current shell
```

```python
import os
db_url = os.environ.get("DATABASE_URL", "sqlite:///default.db")
```

```
┌──────────┐  spawns   ┌──────────┐
│  Shell     │ ────────▶│  Process   │  inherits ALL of the shell's
│  (has env   │           │  (gets a    │  environment variables at
│   vars set)  │           │   COPY)      │  the moment it's spawned
└──────────┘           └──────────┘
```

## 7. Permission

Every file/directory has an owner, a group, and permission bits for **read (r)**, **write (w)**,
**execute (x)**, applied separately to the **owner**, the **group**, and **everyone else**.

```bash
ls -l app.py
# -rwxr-xr--  1 alice  developers  1024  Sep 1 10:00 app.py
#  │││ │││ │││
#  │││ │││ └── others: r-- (read only)
#  │││ └────── group:  r-x (read + execute)
#  └────────── owner:  rwx (read + write + execute)
```

```bash
chmod 755 script.sh     # owner: rwx (7), group: r-x (5), others: r-x (5)
chmod +x script.sh       # just add execute permission for everyone
chown alice:developers app.py    # change owner + group
```

```
      Owner    Group    Others
       rwx      r-x      r--
        7        5        4     →  chmod 754 file
```

## 8. Service

A "service" is a long-running background program managed by the OS's init system (`systemd` on
most modern Linux distros) — things like your database, your app server, nginx. Services can be
started/stopped/enabled-on-boot uniformly, regardless of what language they're written in.

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Backend API
After=network.target postgresql.service

[Service]
User=appuser
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/venv/bin/gunicorn myapp.wsgi:application
Restart=always
Environment=DATABASE_URL=postgres://...

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl start myapp
sudo systemctl enable myapp    # auto-start on boot
sudo systemctl status myapp
```

## 9. Daemon

A daemon is a process that runs **detached from any terminal**, in the background, typically
started at boot and running for the system's entire lifetime (the "d" in `sshd`, `systemd`,
`crond` stands for daemon). A service (above) is usually *how* you manage a daemon.

```
Normal process:        tied to your terminal session — dies when you log out
Daemon process:         detached from any terminal — keeps running after you log out,
                        typically has PID 1 (systemd) as its ultimate ancestor
```

---

# PART B — Commands, With What They're Actually Doing

## File & Directory Navigation

```bash
ls -la              # list files, -l = long format (permissions/owner/size), -a = include hidden
cd /var/log          # change directory
cp app.py app_backup.py       # copy a file
cp -r src/ backup/             # copy a directory recursively
mv old_name.py new_name.py    # move/rename (same syscall under the hood — rename())
rm file.txt                    # delete a file
rm -rf build/                  # delete a directory recursively, force (NO CONFIRMATION — careful)
```

## Searching

```bash
# grep — search file CONTENTS for a pattern
grep "ERROR" app.log                  # find lines containing "ERROR"
grep -r "TODO" ./src/                  # recursive search across a directory
grep -i "error" app.log                # case-insensitive
grep -n "def get_user" app.py          # show line numbers

# find — search the FILESYSTEM for files matching criteria
find /var/log -name "*.log"                     # find files by name pattern
find . -type f -mtime -1                         # files modified in the last 1 day
find . -size +100M                               # files larger than 100MB
find . -name "*.pyc" -delete                     # find AND delete in one command
```

```
grep = "search INSIDE files"          find = "search FOR files"
```

## Reading Files

```bash
cat config.yaml           # dump entire file to stdout
less app.log               # page through a large file interactively (q to quit, / to search)
head -n 20 app.log          # first 20 lines
tail -n 20 app.log           # last 20 lines
tail -f app.log               # FOLLOW mode — stream new lines as they're written (great for live logs)
```

```
head ─▶ [ first lines ] ─────────────────── [ ... ] ─────────────────── [ last lines ] ◀─ tail
```

## Permissions

```bash
chmod 644 config.yaml         # owner: rw-, group: r--, others: r--  (typical for config files)
chmod +x deploy.sh             # make a script executable
chown www-data:www-data /var/www/myapp    # give ownership to the web server user
```

## Process Management

```bash
ps aux                       # snapshot of all running processes
ps aux | grep gunicorn         # find a specific process
top                            # live, auto-refreshing view of processes (CPU/mem usage)
htop                            # nicer, colorized version of top (often needs separate install)
kill 1234                        # send SIGTERM (graceful shutdown request) to PID 1234
kill -9 1234                      # send SIGKILL (force kill, no cleanup) to PID 1234
```

```
kill (SIGTERM)  ──▶ "please shut down gracefully" (process CAN catch this and clean up)
kill -9 (SIGKILL) ──▶ "die immediately" (process CANNOT catch or ignore this — kernel force-kills it)
```

## Services & Logs (systemd)

```bash
systemctl status nginx          # is it running? recent log lines?
systemctl start nginx
systemctl stop nginx
systemctl restart nginx
systemctl enable nginx           # start automatically on boot
journalctl -u nginx               # view logs for a specific systemd service
journalctl -u nginx -f              # follow live, like `tail -f` but for systemd's journal
journalctl --since "1 hour ago"      # time-filtered logs
```

## Networking & File Transfer

```bash
curl https://api.example.com/users          # make an HTTP request from the command line
curl -X POST -d '{"name":"Bob"}' -H "Content-Type: application/json" https://api.example.com/users
curl -I https://example.com                  # HEAD request — just headers, useful for quick checks

ssh alice@192.168.1.10                        # remote shell into another machine
ssh -i ~/.ssh/mykey.pem alice@server.com        # using a specific private key

scp local_file.txt alice@server:/remote/path/     # copy a file TO a remote machine
scp alice@server:/remote/file.txt ./               # copy a file FROM a remote machine

rsync -avz ./local_dir/ alice@server:/remote_dir/   # sync directories, only transfers CHANGED bytes
                                                      # (-a archive mode, -v verbose, -z compress)
```

```
scp  = simple copy, always transfers the WHOLE file
rsync = smart sync, transfers only the DIFFERENCE — much faster for repeated syncs/large dirs
```

---

# Putting It Together — A Real Debugging Scenario

Your API is returning 502 errors in production. Here's how these tools chain together to find out
why:

```bash
# 1. Is the app process even running?
ps aux | grep gunicorn

# 2. Check the systemd service status and recent logs
systemctl status myapp
journalctl -u myapp -n 100 --no-pager

# 3. Is it actually listening on the expected port?
sudo ss -tulpn | grep 8000

# 4. Tail the live application log while you reproduce the issue
tail -f /var/log/myapp/app.log

# 5. Search historical logs for the specific error
grep -n "Traceback" /var/log/myapp/app.log | tail -20

# 6. Check system resources — is it out of memory / file descriptors?
top
ls /proc/$(pgrep -f gunicorn | head -1)/fd | wc -l    # count open file descriptors

# 7. Test connectivity to a dependency directly
curl -I http://localhost:6379    # is Redis even reachable?

# 8. If you need to restart it cleanly
sudo systemctl restart myapp
```

```
  ps/systemctl ──▶ is it running?
        │
        ▼
  journalctl/tail ──▶ what is it saying?
        │
        ▼
  grep ──▶ find the specific error in the noise
        │
        ▼
  ss/curl ──▶ is the network/dependency layer OK?
        │
        ▼
  top/proc ──▶ is it a resource exhaustion problem?
        │
        ▼
  systemctl restart ──▶ apply the fix, then watch logs again to confirm
```

This is the actual day-to-day value of knowing Linux deeply as a backend developer: not
memorizing commands, but knowing **which layer of the system** (process, network, filesystem,
resource limits) each tool lets you inspect, so you can narrow down a production issue instead of
guessing.
