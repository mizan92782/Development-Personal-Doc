# Linux Process & Port Management (Industry Level)

---

# Module 1: Process Fundamentals (Theory)

## Lesson 1: Process কী?

### Program vs Process

| Program | Process |
|---------|---------|
| Disk-এ থাকা Static Code | Memory-তে চলমান Program |
| একটি ফাইল | একটি Instance |
| `manage.py` ফাইল | `python manage.py runserver` চালানো |

### Process Lifecycle

```
New → Ready → Running → Waiting → Ready → Running → Terminated
                  ↓
               Stopped
                  ↓
               Zombie
```

### Process States

| State | অর্থ |
|-------|-------|
| **New** | Process তৈরি হচ্ছে |
| **Ready** | CPU পাওয়ার অপেক্ষায় |
| **Running** | CPU-তে চলছে |
| **Waiting (Sleeping)** | I/O বা Event-এর অপেক্ষায় |
| **Stopped** | Signal দিয়ে থামানো হয়েছে |
| **Zombie** | শেষ হয়েছে কিন্তু Parent এখনও wait() করেনি |

### Process Control Block (PCB)

Kernel প্রতিটি Process-এর জন্য একটি PCB রাখে। এতে থাকে:

- PID (Process ID)
- Process State
- CPU Registers
- Memory Info
- Open File Descriptors
- Priority

---

## Lesson 2: Process Creation

### fork() এবং exec()

```
Parent Process (bash)
        │
     fork()
        │
   ┌────┴────┐
   │         │
Parent     Child (copy of parent)
   │         │
   │       exec()
   │         │
   │     New Program (python)
   │         │
  wait()   exit()
   │
   └── Parent continues
```

| Concept | অর্থ |
|---------|-------|
| `fork()` | Parent-এর exact copy তৈরি করে |
| `exec()` | Child-এর memory-তে নতুন Program load করে |
| **PID** | প্রতিটি Process-এর unique ID |
| **PPID** | Parent Process-এর ID |

### যখন আপনি `python manage.py runserver` চালান, ভিতরে কী ঘটে?

```
1. Bash → fork() → Child Process তৈরি
2. Child → exec("python", "manage.py", "runserver")
3. Python interpreter load হয়
4. Django application start হয়
5. Port 8000-এ listen শুরু
6. Bash wait করে (Foreground) অথবা ছেড়ে দেয় (Background)
```

```bash
# PID এবং PPID দেখুন
echo $$          # Current shell PID
ps -ef | grep python
```

---

## Lesson 3: Process Scheduling

| Concept | অর্থ |
|---------|-------|
| **Scheduler** | কোন Process কখন CPU পাবে তা নির্ধারণ করে |
| **Time Slice** | প্রতিটি Process কতটুকু CPU সময় পাবে |
| **Context Switching** | এক Process থেকে আরেক Process-এ CPU-র নিয়ন্ত্রণ যাওয়া |
| **Priority** | কোন Process আগে চলবে (0-139) |
| **Nice Value** | User-level Priority (-20 থেকে +19, কম = বেশি Priority) |
| **Real-Time Processes** | নির্দিষ্ট সময়ে চলতে হবে এমন Process |

```bash
# Nice value দেখুন
ps -el | head -5

# Nice value দিয়ে process চালু করুন
nice -n 10 python manage.py runserver
```

---

# Module 2: Process Management Commands

> প্রতিটি কমান্ডের জন্য — কী, কেন, কীভাবে, বাস্তব ব্যবহার।

---

## `ps` — Process Snapshot

**কী?** একটি নির্দিষ্ট মুহূর্তে চলমান Process-এর তালিকা।

**কেন?** কোন Process চলছে, তার PID কী — জানতে।

```bash
ps aux                    # সব Process (বিস্তারিত)
ps -ef                    # সব Process (Full Format)
ps aux | grep python      # Python Process খুঁজুন
ps -p 1234                # নির্দিষ্ট PID
```

**Output বোঝা:**
```
USER    PID  %CPU %MEM    VSZ   RSS  STAT  COMMAND
mizan  1234   2.5  1.2  12345  4567  S     python manage.py
```

| Column | অর্থ |
|--------|-------|
| PID | Process ID |
| %CPU | CPU ব্যবহার |
| %MEM | Memory ব্যবহার |
| STAT | S=Sleep, R=Running, Z=Zombie |

---

## `top` — Real-time Process Monitor

**কী?** Live Process এবং System Resource Monitor।

```bash
top
top -u mizan        # নির্দিষ্ট User-এর Process
top -p 1234         # নির্দিষ্ট PID Monitor
```

**top চলাকালীন shortcuts:**
| Key | কাজ |
|-----|-----|
| `q` | বের হওয়া |
| `k` | Process Kill করা |
| `M` | Memory অনুযায়ী sort |
| `P` | CPU অনুযায়ী sort |
| `1` | প্রতিটি CPU আলাদাভাবে দেখা |

---

## `htop` — Interactive Process Viewer

**কী?** `top`-এর উন্নত version, colorful এবং user-friendly।

```bash
htop
sudo apt install htop    # না থাকলে install করুন
```

**Interview Question:** `top` vs `htop` পার্থক্য কী?
> `htop`-এ mouse support, color, horizontal/vertical scroll, Tree view আছে। `top` সব system-এ default থাকে।

---

## `pstree` — Process Tree

**কী?** Process-এর Parent-Child সম্পর্ক tree আকারে দেখায়।

```bash
pstree
pstree -p           # PID সহ
pstree -u mizan     # নির্দিষ্ট User
```

---

## `pidof` — Program-এর PID খুঁজুন

```bash
pidof python3
pidof nginx
```

---

## `pgrep` — Pattern দিয়ে Process খুঁজুন

```bash
pgrep python
pgrep -l python        # নাম সহ
pgrep -u mizan nginx   # নির্দিষ্ট User-এর nginx
```

---

## `pkill` — Pattern দিয়ে Process Kill করুন

```bash
pkill python
pkill -9 python        # Force kill
pkill -u mizan python  # নির্দিষ্ট User-এর Python
```

---

## `kill` — PID দিয়ে Signal পাঠান

```bash
kill 1234           # SIGTERM (graceful)
kill -9 1234        # SIGKILL (force)
kill -15 1234       # SIGTERM (explicit)
kill -l             # সব Signal দেখুন
```

---

## `killall` — নাম দিয়ে সব Process Kill করুন

```bash
killall python3
killall -9 nginx
```

---

## `jobs`, `bg`, `fg`

```bash
jobs           # Background jobs দেখুন
bg %1          # Job 1 Background-এ চালু করুন
fg %1          # Job 1 Foreground-এ আনুন
```

---

## `nohup` — Terminal বন্ধ হলেও চলবে

```bash
nohup python manage.py runserver &
nohup ./script.sh > output.log 2>&1 &
```

---

## `nice` এবং `renice`

```bash
nice -n 10 python manage.py runserver    # কম Priority দিয়ে start
renice -n 5 -p 1234                      # চলমান Process-এর Priority পরিবর্তন
```

---

## `watch` — Command বারবার চালান

```bash
watch -n 2 ps aux | grep python    # প্রতি 2 সেকেন্ডে
watch -n 1 free -h                 # Memory দেখুন
```

---

## `time` — Command কতক্ষণ লাগল

```bash
time python manage.py migrate
```

Output:
```
real    0m2.345s
user    0m1.234s
sys     0m0.111s
```

---

# Module 3: Signals

## Signal কী?

Signal হলো Process-এ পাঠানো একটি notification। Process সেই Signal অনুযায়ী কাজ করে।

```
Process ← Signal ← Kernel / User / Another Process
```

## গুরুত্বপূর্ণ Signals

| Signal | Number | অর্থ |
|--------|--------|-------|
| `SIGTERM` | 15 | Gracefully বন্ধ হওয়ার request |
| `SIGKILL` | 9 | জোর করে বন্ধ (ignore করা যায় না) |
| `SIGINT` | 2 | Ctrl+C চাপলে |
| `SIGHUP` | 1 | Terminal বন্ধ হলে / Config reload |
| `SIGSTOP` | 19 | Process pause করুন (ignore করা যায় না) |
| `SIGCONT` | 18 | Paused Process চালু করুন |

```bash
kill -15 1234    # SIGTERM — Django gracefully বন্ধ হবে
kill -9 1234     # SIGKILL — জোর করে বন্ধ
kill -1 1234     # SIGHUP  — Nginx config reload
kill -19 1234    # SIGSTOP — Pause
kill -18 1234    # SIGCONT — Resume
```

## `kill -9` বনাম `kill -15` — পার্থক্য কী? ভিতরে কী ঘটে?

| | `kill -15` (SIGTERM) | `kill -9` (SIGKILL) |
|--|---------------------|---------------------|
| **কে handle করে?** | Process নিজে | Kernel |
| **Ignore করা যায়?** | হ্যাঁ | না |
| **Cleanup হয়?** | হ্যাঁ (DB connection বন্ধ, file flush) | না |
| **কখন ব্যবহার?** | সবসময় প্রথমে এটি | শুধু যখন -15 কাজ না করে |

> **Best Practice:** সবসময় আগে `kill -15`, কাজ না হলে `kill -9`।

---

# Module 4: Background Processes

## Foreground vs Background

```bash
python manage.py runserver          # Foreground — terminal block হয়
python manage.py runserver &        # Background — terminal free থাকে
```

## `&` — Background-এ চালান

```bash
python manage.py runserver &
# Output: [1] 5678  ← Job number এবং PID
```

## `jobs` — চলমান Jobs দেখুন

```bash
jobs
# [1]+  Running    python manage.py runserver &
```

## `bg` এবং `fg`

```bash
# Ctrl+Z দিয়ে Foreground process pause করুন
# তারপর background-এ পাঠান
bg %1

# আবার foreground-এ আনুন
fg %1
```

## `nohup` — Session-independent Process

```bash
nohup python manage.py runserver > server.log 2>&1 &
```

Terminal বন্ধ করলেও Process চলতে থাকবে।

## `disown` — Job list থেকে সরান

```bash
python manage.py runserver &
disown %1    # এখন terminal বন্ধ করলেও চলবে
```

---

# Module 5: Process Monitoring

## `top` / `htop`

```bash
top
htop
```

## `vmstat` — Virtual Memory Statistics

```bash
vmstat 1 5    # প্রতি 1 সেকেন্ডে, 5 বার
```

## `uptime` — System কতক্ষণ চলছে + Load Average

```bash
uptime
# 10:30:00 up 5 days,  2:15,  2 users,  load average: 0.52, 0.58, 0.59
```

**Load Average বোঝা (1 CPU-র জন্য):**
| Value | অর্থ |
|-------|-------|
| 0.5 | CPU 50% busy |
| 1.0 | CPU 100% busy (সর্বোচ্চ ভালো) |
| 1.5+ | Overloaded |

## `free` — Memory Usage

```bash
free -h
```

```
              total    used    free   shared  buff/cache  available
Mem:           7.7G    3.2G    1.1G    456M       3.4G       3.8G
Swap:          2.0G    0.0G    2.0G
```

## `iostat` — CPU এবং Disk I/O

```bash
iostat
iostat -x 1    # Extended, প্রতি 1 সেকেন্ডে
```

## `sar` — System Activity Reporter

```bash
sar -u 1 5     # CPU usage প্রতি 1 সেকেন্ডে, 5 বার
sar -r 1 5     # Memory usage
```

---

# Module 6: Networking Basics

Port বোঝার আগে এই concepts জানা দরকার।

| Concept | অর্থ |
|---------|-------|
| **IP Address** | Network-এ একটি Device-এর ঠিকানা |
| **Loopback (127.0.0.1)** | নিজের কম্পিউটার (localhost) |
| **MAC Address** | Hardware-এর Physical ঠিকানা |
| **Socket** | IP Address + Port = একটি Connection endpoint |
| **Client** | যে Connection করে |
| **Server** | যে Connection accept করে |
| **TCP** | Reliable, connection-based (HTTP, SSH) |
| **UDP** | Fast, connectionless (DNS, Video) |

---

# Module 7: Port কী?

> আপনার কম্পিউটার একটি Apartment Building।
> - **IP Address** = Building-এর ঠিকানা
> - **Port** = প্রতিটি Flat-এর নম্বর

```
192.168.1.10:8000
     │          │
  Building    Flat 8000
  Address     (Django)
```

**Port Range:**
| Range | ধরন |
|-------|-----|
| 0 - 1023 | Well-known ports (root লাগে) |
| 1024 - 49151 | Registered ports |
| 49152 - 65535 | Dynamic/Private ports |

---

# Module 8: Common Ports

| Port | Service |
|------|---------|
| 22 | SSH |
| 80 | HTTP |
| 443 | HTTPS |
| 3306 | MySQL |
| 5432 | PostgreSQL |
| 6379 | Redis |
| 8000 | Django Development Server |
| 8080 | Alternative HTTP |
| 27017 | MongoDB |
| 5672 | RabbitMQ |

---

# Module 9: Port Commands

## `ss` — Socket Statistics (Modern `netstat`)

```bash
ss -tulpn
```

| Flag | অর্থ |
|------|-------|
| `-t` | TCP |
| `-u` | UDP |
| `-l` | Listening ports |
| `-p` | Process দেখান |
| `-n` | Numeric (hostname resolve না করে) |

```bash
ss -tulpn | grep :8000    # Port 8000 দেখুন
ss -tulpn | grep LISTEN   # শুধু listening ports
```

---

## `netstat` — Network Statistics (Legacy)

```bash
netstat -tulpn            # সব listening ports
netstat -tulpn | grep 8000
```

> **Note:** অনেক modern system-এ `ss` preferred। `netstat` install: `sudo apt install net-tools`

---

## `lsof` — List Open Files (Ports সহ)

```bash
lsof -i :8000             # Port 8000 কে ব্যবহার করছে
lsof -i tcp               # সব TCP connections
lsof -i -P -n | grep LISTEN
```

---

## `fuser` — কোন Process একটি Port ব্যবহার করছে

```bash
fuser 8000/tcp            # PID দেখাবে
fuser -k 8000/tcp         # Process kill করুন
```

---

## `nc` (netcat) — Network Swiss Army Knife

```bash
nc -zv 127.0.0.1 8000    # Port open আছে কিনা test করুন
nc -l 9000                # Port 9000-এ listen করুন
```

---

## `telnet` — Port Test করুন

```bash
telnet localhost 8000
telnet 192.168.1.10 5432  # Remote PostgreSQL test
```

---

## `nmap` — Network Scanner

```bash
nmap localhost                      # সব open ports
nmap -p 8000 localhost              # নির্দিষ্ট port
nmap -sV localhost                  # Service version সহ
```

---

# Module 10: Find Process Using a Port

## Port 8000 কে ব্যবহার করছে?

```bash
# Method 1: ss (Recommended)
ss -tulpn | grep :8000

# Method 2: lsof
lsof -i :8000

# Method 3: fuser
fuser 8000/tcp

# Method 4: netstat
netstat -tulpn | grep :8000
```

**Output example (`lsof`):**
```
COMMAND   PID   USER   FD   TYPE  NODE NAME
python   5678  mizan   5u  IPv4  TCP  *:8000 (LISTEN)
```

এখান থেকে PID = `5678`

---

# Module 11: Free a Port

## নিরাপদভাবে Process বন্ধ করুন

```bash
# Step 1: কে Port ব্যবহার করছে?
lsof -i :8000

# Step 2: PID নিন (ধরুন 5678)
# Step 3: Gracefully বন্ধ করুন
kill -15 5678

# Step 4: নিশ্চিত হন
lsof -i :8000

# Step 5: এখনও চললে Force kill
kill -9 5678

# অথবা একধাপে (fuser দিয়ে)
fuser -k 8000/tcp
```

> **সতর্কতা:** `kill -9` ব্যবহারের আগে নিশ্চিত হন যে সঠিক Process kill করছেন।

---

# Module 12: Real Django Examples

## Scenario: Port 8000 Busy

```
python manage.py runserver → Error: Port 8000 already in use
```

## Step-by-Step সমাধান

```bash
# Step 1: কে Port 8000 দখল করছে?
lsof -i :8000

# Step 2: Output দেখুন
# COMMAND   PID   USER
# python   5678  mizan

# Step 3: সেটি কী Process?
ps -p 5678 -f

# Step 4: Gracefully বন্ধ করুন
kill -15 5678

# Step 5: নিশ্চিত করুন Port Free হয়েছে
ss -tulpn | grep :8000

# Step 6: আবার Server চালু করুন
python manage.py runserver
```

## Django Server Background-এ চালানো (Production-এর মতো)

```bash
# Background-এ চালান, log file-এ output
nohup python manage.py runserver 0.0.0.0:8000 > django.log 2>&1 &

# PID মনে রাখুন
echo $! > django.pid

# পরে বন্ধ করতে
kill $(cat django.pid)
```

## একাধিক Django Server চালানো

```bash
python manage.py runserver 8000 &
python manage.py runserver 8001 &
python manage.py runserver 8002 &

# সব দেখুন
ss -tulpn | grep python
```

---

## Quick Reference

```bash
# Process দেখুন
ps aux | grep <name>
pgrep -l <name>

# Process বন্ধ করুন
kill -15 <PID>           # Graceful
kill -9 <PID>            # Force

# Port খুঁজুন
lsof -i :<port>
ss -tulpn | grep :<port>

# Port free করুন
fuser -k <port>/tcp

# Background
<command> &              # Background
nohup <command> &        # Terminal-independent background
jobs                     # List
fg %1                    # Foreground-এ আনুন
```
