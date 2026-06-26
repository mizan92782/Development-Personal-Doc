# `systemctl` — সম্পূর্ণ টিউটোরিয়াল
> Linux Service Management মাস্টারক্লাস | Ubuntu / Debian / Fedora / CentOS

---

## 📚 সূচিপত্র

1. [systemctl কী?](#1-systemctl-কী)
2. [systemd কী এবং কেন?](#2-systemd-কী-এবং-কেন)
3. [Service কী?](#3-service-কী)
4. [Linux Boot Process — Service কীভাবে চালু হয়](#4-linux-boot-process--service-কীভাবে-চালু-হয়)
5. [systemctl কমান্ড — সম্পূর্ণ রেফারেন্স টেবিল](#5-systemctl-কমান্ড--সম্পূর্ণ-রেফারেন্স-টেবিল)
6. [প্রতিটি কমান্ডের বিস্তারিত ব্যাখ্যা](#6-প্রতিটি-কমান্ডের-বিস্তারিত-ব্যাখ্যা)
7. [Service File — গভীর ধারণা](#7-service-file--গভীর-ধারণা)
8. [Unit File এর Section গুলো বিস্তারিত](#8-unit-file-এর-section-গুলো-বিস্তারিত)
9. [নিজের Service তৈরি করা (Real-world Examples)](#9-নিজের-service-তৈরি-করা-real-world-examples)
10. [journalctl — Log Management](#10-journalctl--log-management)
11. [systemctl Targets — Runlevel এর আধুনিক রূপ](#11-systemctl-targets--runlevel-এর-আধুনিক-রূপ)
12. [Service Dependencies — After, Requires, Wants](#12-service-dependencies--after-requires-wants)
13. [Restart Policy — Crash হলে কী করবে](#13-restart-policy--crash-হলে-কী-করবে)
14. [Environment Variables in Service](#14-environment-variables-in-service)
15. [Security Hardening — Service Sandboxing](#15-security-hardening--service-sandboxing)
16. [Systemd Timers — Cron এর বিকল্প](#16-systemd-timers--cron-এর-বিকল্প)
17. [Real-world Deployment Workflows](#17-real-world-deployment-workflows)
18. [Troubleshooting Guide](#18-troubleshooting-guide)
19. [চিট শিট — Quick Reference](#19-চিট-শিট--quick-reference)

---

## 1. `systemctl` কী?

`systemctl` হলো **systemd**-এর command-line interface (CLI) tool।

এটি দিয়ে আপনি Linux-এর যেকোনো **service** (background program) কে:
- চালু বা বন্ধ করতে পারবেন
- Boot-এর সময় automatically চালু/বন্ধ করতে পারবেন
- Status এবং log দেখতে পারবেন
- নিজের service তৈরি করতে পারবেন

```
systemctl COMMAND SERVICE_NAME
```

**উদাহরণ:**
```bash
systemctl status nginx
sudo systemctl start nginx
sudo systemctl enable nginx
```

---

## 2. `systemd` কী এবং কেন?

### সংজ্ঞা
`systemd` হলো Linux-এর **init system** এবং **service manager**।

"Init system" মানে হলো — Linux kernel চালু হওয়ার পর সবার আগে যে program চলে, সেটাই init system। এর **PID (Process ID) = 1**।

### ইতিহাস

| যুগ | Init System | সমস্যা |
|-----|-------------|--------|
| পুরনো Unix | `SysV init` | ধীর, sequential boot |
| মাঝামাঝি | `Upstart` | ভালো কিন্তু জটিল |
| আধুনিক | `systemd` | দ্রুত, parallel boot, শক্তিশালী |

### systemd কীভাবে কাজ করে?

```
Kernel চালু হলো
       │
       ▼
   systemd (PID 1)
       │
       ├── network.service
       ├── sshd.service
       ├── nginx.service
       ├── mysql.service
       ├── docker.service
       └── redis.service
```

systemd সবকিছু **parallel** এ চালু করে, তাই boot অনেক দ্রুত হয়।

### কোন Distro তে কী?

| Linux Distribution | Init System |
|--------------------|-------------|
| Ubuntu 15.04+      | systemd ✅  |
| Debian 8+          | systemd ✅  |
| Fedora 15+         | systemd ✅  |
| CentOS 7+          | systemd ✅  |
| Arch Linux         | systemd ✅  |
| Alpine Linux       | OpenRC ❌   |

---

## 3. Service কী?

Service (বা **Daemon**) হলো এমন একটি program যা:
- **Background**-এ সবসময় চলে
- কোনো user interaction ছাড়াই কাজ করে
- সাধারণত `d` দিয়ে শেষ হয় (nginx**d**, ssh**d**, mysql**d**)

### উদাহরণ

| Service | কী করে |
|---------|--------|
| `nginx` / `apache2` | Web request গ্রহণ করে |
| `mysql` / `postgresql` | Database request সামলায় |
| `sshd` | Remote login সম্ভব করে |
| `docker` | Container চালায় |
| `redis` | Cache/Queue পরিচালনা করে |
| `cron` | Scheduled task চালায় |
| `ufw` | Firewall পরিচালনা করে |

### Service এর জীবনচক্র

```
installed ──► enabled ──► active (running)
                │               │
                ▼               ▼
            disabled        inactive
                                │
                                ▼
                            failed
```

---

## 4. Linux Boot Process — Service কীভাবে চালু হয়

```
BIOS/UEFI
    │
    ▼
Bootloader (GRUB)
    │
    ▼
Linux Kernel load হয়
    │
    ▼
systemd শুরু হয় (PID = 1)
    │
    ▼
default.target লোড হয়
(সাধারণত multi-user.target অথবা graphical.target)
    │
    ▼
WantedBy=multi-user.target দেওয়া সব service চালু হয়
    │
    ▼
Login prompt আসে
```

তাই আপনি যখন `systemctl enable nginx` করেন, nginx কে `multi-user.target`-এর সাথে link করা হয়, ফলে boot-এ automatically চালু হয়।

---

## 5. systemctl কমান্ড — সম্পূর্ণ রেফারেন্স টেবিল

> এখানে প্রতিটি কমান্ড, কেন ব্যবহার করবেন তা সহ দেওয়া হলো।

### 🔵 Service Control কমান্ড

| কমান্ড | কেন ব্যবহার করবেন |
|--------|-------------------|
| `systemctl status <service>` | Service চলছে কি না, শেষ কয়েকটি log দেখতে |
| `sudo systemctl start <service>` | Service এখনই চালু করতে |
| `sudo systemctl stop <service>` | Service এখনই বন্ধ করতে |
| `sudo systemctl restart <service>` | Service বন্ধ করে আবার চালু করতে |
| `sudo systemctl reload <service>` | Process বন্ধ না করে config পুনরায় পড়তে |
| `sudo systemctl kill <service>` | Service কে force kill করতে |
| `sudo systemctl enable <service>` | Boot-এ automatically চালু হওয়ার ব্যবস্থা করতে |
| `sudo systemctl disable <service>` | Boot-এ automatically চালু হওয়া বন্ধ করতে |
| `sudo systemctl enable --now <service>` | এখনই চালু করো + boot-এও চালু রাখো |
| `sudo systemctl disable --now <service>` | এখনই বন্ধ করো + boot-এও বন্ধ রাখো |
| `sudo systemctl mask <service>` | Service কে সম্পূর্ণ ব্লক করতে (start করাও যাবে না) |
| `sudo systemctl unmask <service>` | Mask সরাতে |

### 🟢 Service Status/Query কমান্ড

| কমান্ড | কেন ব্যবহার করবেন |
|--------|-------------------|
| `systemctl is-active <service>` | চলছে কি না জানতে (script-এ ব্যবহারযোগ্য) |
| `systemctl is-enabled <service>` | Boot-এ চালু হবে কি না জানতে |
| `systemctl is-failed <service>` | Crash করেছে কি না জানতে |
| `systemctl list-units` | সব active unit দেখতে |
| `systemctl list-units --type=service` | শুধু service গুলো দেখতে |
| `systemctl list-unit-files` | সব service file এবং তাদের enable/disable অবস্থা |
| `systemctl --failed` | Failed হয়ে থাকা সব service দেখতে |
| `systemctl show <service>` | Service-এর সব property দেখতে |

### 🟡 System Control কমান্ড

| কমান্ড | কেন ব্যবহার করবেন |
|--------|-------------------|
| `sudo systemctl daemon-reload` | নতুন বা পরিবর্তিত service file systemd-এ লোড করতে |
| `sudo systemctl reboot` | System restart করতে |
| `sudo systemctl poweroff` | System বন্ধ করতে |
| `sudo systemctl suspend` | RAM তে রেখে sleep mode-এ যেতে |
| `sudo systemctl hibernate` | Disk-এ state save করে বন্ধ করতে |
| `sudo systemctl rescue` | Rescue/single-user mode-এ যেতে |
| `sudo systemctl emergency` | Emergency mode-এ যেতে |
| `systemctl get-default` | Default boot target দেখতে |
| `sudo systemctl set-default <target>` | Default boot target পরিবর্তন করতে |

### 🔴 Log কমান্ড (journalctl)

| কমান্ড | কেন ব্যবহার করবেন |
|--------|-------------------|
| `journalctl -u <service>` | একটি service-এর সব log দেখতে |
| `journalctl -u <service> -f` | Live/real-time log দেখতে |
| `journalctl -u <service> -n 50` | শেষ ৫০টি log entry দেখতে |
| `journalctl -u <service> --since "1 hour ago"` | শেষ ১ ঘণ্টার log দেখতে |
| `journalctl -p err` | শুধু error log দেখতে |
| `journalctl --disk-usage` | Log কত জায়গা নিচ্ছে জানতে |
| `sudo journalctl --vacuum-time=7d` | ৭ দিনের পুরনো log মুছতে |

---

## 6. প্রতিটি কমান্ডের বিস্তারিত ব্যাখ্যা

### `systemctl status`

```bash
systemctl status nginx
```

**Output বিশ্লেষণ:**

```
● nginx.service - A high performance web server
     Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2024-01-15 10:30:00 UTC; 2h ago
       Docs: man:nginx(8)
   Main PID: 1234 (nginx)
      Tasks: 3 (limit: 2345)
     Memory: 6.8M
        CPU: 345ms
     CGroup: /system.slice/nginx.service
             ├─1234 nginx: master process /usr/sbin/nginx
             └─1235 nginx: worker process
```

| Field | মানে |
|-------|------|
| `Loaded` | Service file কোথায় এবং enable/disable কিনা |
| `Active: active (running)` | এখন চলছে ✅ |
| `Active: inactive (dead)` | বন্ধ আছে |
| `Active: failed` | Error হয়ে বন্ধ হয়েছে ❌ |
| `Main PID` | মূল process-এর ID |
| `Memory` | কতটুকু RAM ব্যবহার করছে |

---

### `start` vs `stop` vs `restart` vs `reload`

```bash
sudo systemctl start nginx     # চালু করো
sudo systemctl stop nginx      # বন্ধ করো
sudo systemctl restart nginx   # বন্ধ করে আবার চালু করো
sudo systemctl reload nginx    # process রেখে শুধু config reload করো
```

**Restart vs Reload পার্থক্য:**

```
RESTART:                    RELOAD:
Process বন্ধ হয়              Process চলতে থাকে
    │                           │
    ▼                           ▼
নতুন করে শুরু হয়          Config পুনরায় পড়ে
    │                           │
    ▼                           ▼
Downtime হয় (~1-2 sec)    Zero downtime ✅
```

> **Best Practice:** Nginx config পরিবর্তনের পর `reload` ব্যবহার করুন, তাহলে চলমান connection বাধাগ্রস্ত হবে না।

---

### `enable` vs `disable`

```bash
sudo systemctl enable nginx   # boot-এ চালু হবে
sudo systemctl disable nginx  # boot-এ চালু হবে না
```

এটি আসলে একটি **symbolic link** তৈরি/মুছে দেয়:

```bash
# enable করলে এরকম link তৈরি হয়:
/etc/systemd/system/multi-user.target.wants/nginx.service
    → /lib/systemd/system/nginx.service
```

---

### `enable --now` (সবচেয়ে বেশি ব্যবহৃত)

```bash
sudo systemctl enable --now nginx
```

এটি একসাথে দুটো কাজ করে:
1. এখনই `start` করে
2. Boot-এ `enable` করে

Deployment-এ এটি সবচেয়ে বেশি ব্যবহার করা হয়।

---

### `mask` — অ্যাডভান্সড লকিং

```bash
sudo systemctl mask nginx    # সম্পূর্ণ ব্লক
sudo systemctl unmask nginx  # ব্লক সরাও
```

`mask` করলে এমনকি `start` কমান্ডও কাজ করে না। এটি নিরাপত্তার জন্য ব্যবহার করা হয়। যেমন — কোনো vulnerable service যা চালানো একদমই উচিত না।

```
mask vs disable:
disable → boot-এ চালু হবে না, কিন্তু manually start করা যাবে
mask   → কোনোভাবেই start করা যাবে না
```

---

### `daemon-reload` — কখন অবশ্যই দিতে হবে

```bash
sudo systemctl daemon-reload
```

এটি দিতে হয় যখন:
- নতুন `.service` file তৈরি করেছেন
- কোনো `.service` file পরিবর্তন করেছেন
- `/etc/systemd/system/` এ কিছু যোগ করেছেন

**দিতে ভুললে কী হয়?**
পুরনো configuration দিয়ে কাজ হবে, নতুন পরিবর্তন কাজ করবে না।

---

## 7. Service File — গভীর ধারণা

Service File হলো একটি configuration file যা systemd কে বলে দেয় কীভাবে একটি program চালাতে হবে।

### কোথায় থাকে?

| Path | ব্যবহার |
|------|---------|
| `/lib/systemd/system/` | Package থেকে install হওয়া service (read-only) |
| `/etc/systemd/system/` | Admin-তৈরি custom service (এখানে নিজের service রাখুন) |
| `~/.config/systemd/user/` | User-level service (sudo ছাড়া) |

> **Priority:** `/etc/systemd/system/` > `/lib/systemd/system/`
> অর্থাৎ `/etc/` তে রাখলে default কে override করা যায়।

### Service File এর মৌলিক কাঠামো

```ini
[Unit]
Description=সার্ভিসের বিবরণ
After=network.target

[Service]
Type=simple
User=username
WorkingDirectory=/path/to/project
ExecStart=/path/to/command
Restart=always

[Install]
WantedBy=multi-user.target
```

---

## 8. Unit File এর Section গুলো বিস্তারিত

### `[Unit]` Section

```ini
[Unit]
Description=My Application    # মানুষ পড়ার জন্য নাম
Documentation=https://docs.example.com  # Doc link
After=network.target           # network চালু হওয়ার পর চালাও
After=postgresql.service       # postgres চালু হওয়ার পর চালাও
Requires=postgresql.service    # postgres না থাকলে চালাবে না
Wants=redis.service            # redis থাকলে ভালো, না থাকলেও চলবে
```

| Directive | মানে |
|-----------|------|
| `After=` | কোন service-এর পরে চালু হবে |
| `Before=` | কোন service-এর আগে চালু হবে |
| `Requires=` | এই service না থাকলে এটি চালু হবে না |
| `Wants=` | এই service থাকলে ভালো, কিন্তু required না |
| `BindsTo=` | নির্ভরশীল service বন্ধ হলে এটিও বন্ধ হবে |

---

### `[Service]` Section

```ini
[Service]
Type=simple                    # default, ExecStart = main process
User=www-data                  # কোন user দিয়ে চলবে
Group=www-data                 # কোন group
WorkingDirectory=/var/www/app  # কোন directory থেকে চলবে
ExecStart=/usr/bin/node server.js  # মূল command
ExecStop=/bin/kill -SIGTERM $MAINPID  # বন্ধের command
ExecReload=/bin/kill -HUP $MAINPID   # reload command
Restart=always                 # crash হলে আবার চালাও
RestartSec=5                   # restart-এর আগে ৫ সেকেন্ড অপেক্ষা
StandardOutput=journal         # stdout journald-এ যাবে
StandardError=journal          # stderr journald-এ যাবে
```

### Service Type সমূহ

| Type | কখন ব্যবহার করবেন |
|------|-------------------|
| `simple` | সাধারণ foreground process (default) |
| `forking` | Process নিজেই fork করে background-এ যায় (পুরনো daemon) |
| `oneshot` | একবার চলে শেষ হয়ে যায় (script/migration) |
| `notify` | Process ready হলে systemd-কে notify করে |
| `exec` | `simple`-এর মতো, আরো accurate |

---

### `[Install]` Section

```ini
[Install]
WantedBy=multi-user.target   # সাধারণ server-এর জন্য
WantedBy=graphical.target    # Desktop GUI সহ
```

| Target | মানে |
|--------|------|
| `multi-user.target` | Text-mode server (সবচেয়ে বেশি ব্যবহৃত) |
| `graphical.target` | Desktop environment সহ |
| `network.target` | Network চালু হলে |

---

## 9. নিজের Service তৈরি করা (Real-world Examples)

### Example 1: Django + Gunicorn Service

```bash
sudo nano /etc/systemd/system/myshop.service
```

```ini
[Unit]
Description=Django Shop Application
After=network.target
After=postgresql.service

[Service]
Type=exec
User=ubuntu
Group=ubuntu
WorkingDirectory=/home/ubuntu/myshop
ExecStart=/home/ubuntu/myshop/venv/bin/gunicorn \
    --workers 3 \
    --bind unix:/run/myshop.sock \
    config.wsgi:application
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now myshop
sudo systemctl status myshop
```

---

### Example 2: Node.js / Express Service

```ini
[Unit]
Description=Node.js Express API Server
After=network.target

[Service]
Type=simple
User=nodeuser
WorkingDirectory=/var/www/api
ExecStart=/usr/bin/node /var/www/api/server.js
Restart=always
RestartSec=10
Environment=NODE_ENV=production
Environment=PORT=3000
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

---

### Example 3: Spring Boot JAR Service

```ini
[Unit]
Description=Spring Boot Application
After=syslog.target network.target

[Service]
Type=simple
User=springuser
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/java -jar /opt/myapp/myapp.jar
ExecStop=/bin/kill -15 $MAINPID
Restart=on-failure
RestartSec=10
StandardOutput=journal
StandardError=journal
Environment="SPRING_PROFILES_ACTIVE=prod"
Environment="JAVA_OPTS=-Xms512m -Xmx1024m"

[Install]
WantedBy=multi-user.target
```

---

### Example 4: Celery Worker Service (Django)

```ini
[Unit]
Description=Celery Worker for Django
After=network.target redis.service

[Service]
Type=forking
User=ubuntu
WorkingDirectory=/home/ubuntu/myproject
ExecStart=/home/ubuntu/myproject/venv/bin/celery \
    -A config worker \
    --loglevel=info \
    --pidfile=/run/celery/worker.pid
ExecStop=/bin/kill -TERM $MAINPID
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

---

## 10. `journalctl` — Log Management

`journalctl` হলো systemd-এর log viewer। সব service-এর log এখানে থাকে।

```bash
# একটি service-এর সব log
journalctl -u nginx

# Live log (Ctrl+C দিয়ে বের হন)
journalctl -u nginx -f

# শেষ ১০০টি line
journalctl -u nginx -n 100

# শেষ ১ ঘণ্টার log
journalctl -u nginx --since "1 hour ago"

# নির্দিষ্ট সময়ের log
journalctl -u nginx --since "2024-01-15 10:00:00" --until "2024-01-15 11:00:00"

# শুধু error এবং তার উপরের level
journalctl -u nginx -p err

# Log levels: debug, info, notice, warning, err, crit, alert, emerg

# সব service-এর log
journalctl

# Boot থেকে এখন পর্যন্ত
journalctl -b

# আগের boot-এর log
journalctl -b -1

# Log size দেখুন
journalctl --disk-usage

# পুরনো log মুছুন
sudo journalctl --vacuum-time=30d   # ৩০ দিনের বেশি পুরনো মুছুন
sudo journalctl --vacuum-size=500M  # ৫০০MB এর বেশি হলে পুরনো মুছুন
```

---

## 11. `systemctl` Targets — Runlevel এর আধুনিক রূপ

পুরনো SysV init-এ ছিল **Runlevel** (0-6)। systemd-এ এর পরিবর্তে **Target** ব্যবহার হয়।

| SysV Runlevel | systemd Target | মানে |
|---------------|----------------|------|
| 0 | `poweroff.target` | System বন্ধ |
| 1 | `rescue.target` | Single-user/rescue mode |
| 3 | `multi-user.target` | Network সহ, GUI ছাড়া (server-এর জন্য) |
| 5 | `graphical.target` | Desktop GUI সহ |
| 6 | `reboot.target` | Reboot |

```bash
# বর্তমান default target দেখুন
systemctl get-default

# Server কে graphical mode থেকে text mode-এ নিয়ে যান (server optimization)
sudo systemctl set-default multi-user.target

# এখনই একটি target-এ যান
sudo systemctl isolate multi-user.target
```

---

## 12. Service Dependencies — `After`, `Requires`, `Wants`

### বাস্তব উদাহরণ: Django stack

```
Browser Request
      │
      ▼
   Nginx (web server)
      │
      ▼
  Gunicorn/Django (app server)
      │
      ├──► PostgreSQL (database)
      └──► Redis (cache/queue)
```

এই dependency service file-এ এভাবে লেখা হয়:

```ini
# myshop.service
[Unit]
After=network.target
After=postgresql.service
After=redis.service
Wants=redis.service
Requires=postgresql.service
```

এর মানে:
- `network` চালু হওয়ার পর চালাও
- `postgresql` চালু না হওয়া পর্যন্ত অপেক্ষা করো, না থাকলে চালাবে না
- `redis` চালু হলে পরে চালাও, কিন্তু redis না থাকলেও চলবে

---

## 13. Restart Policy — Crash হলে কী করবে

```ini
[Service]
Restart=always        # সবসময় restart করো
RestartSec=5          # ৫ সেকেন্ড পর

# অন্যান্য option:
# Restart=no          → কখনোই restart করবে না
# Restart=on-success  → সফলভাবে exit করলে restart
# Restart=on-failure  → failure/crash হলে restart (production recommended)
# Restart=on-abnormal → abnormal exit-এ restart
# Restart=always      → যেকোনো কারণে exit হলে restart
```

### Production Recommendation

```ini
Restart=on-failure
RestartSec=5
StartLimitIntervalSec=60
StartLimitBurst=3
```

এর মানে: Failure-এ restart, ৫ সেকেন্ড পর। কিন্তু ৬০ সেকেন্ডে ৩ বারের বেশি crash করলে আর restart করবে না (loop থেকে বাঁচাবে)।

---

## 14. Environment Variables in Service

### পদ্ধতি ১: সরাসরি Service File-এ

```ini
[Service]
Environment="DATABASE_URL=postgresql://user:pass@localhost/db"
Environment="SECRET_KEY=mysecretkey"
Environment="DEBUG=False"
```

### পদ্ধতি ২: `.env` File থেকে (নিরাপদ)

```ini
[Service]
EnvironmentFile=/etc/myapp/myapp.env
```

`/etc/myapp/myapp.env` ফাইলে:
```
DATABASE_URL=postgresql://user:pass@localhost/db
SECRET_KEY=mysecretkey
DEBUG=False
```

```bash
sudo chmod 600 /etc/myapp/myapp.env   # শুধু root পড়তে পারবে
```

> **Security Tip:** কখনো password বা secret key সরাসরি service file-এ রাখবেন না। `.env` file ব্যবহার করুন এবং `chmod 600` দিন।

---

## 15. Security Hardening — Service Sandboxing

systemd অনেক powerful security feature দেয়:

```ini
[Service]
# Filesystem restrictions
ProtectSystem=strict          # /usr, /boot, /etc read-only করে দাও
ProtectHome=true              # /home, /root access বন্ধ
ReadWritePaths=/var/lib/myapp # শুধু এই path-এ লিখতে পারবে

# Privilege restrictions
NoNewPrivileges=true          # process নতুন privilege নিতে পারবে না
PrivateTmp=true               # নিজস্ব /tmp directory পাবে
PrivateDevices=true           # hardware device access বন্ধ

# Network restrictions
PrivateNetwork=false          # network access আছে (true করলে বন্ধ)

# User/Group
User=myappuser
Group=myappgroup
```

এগুলো ব্যবহার করলে service compromise হলেও system-এর ক্ষতি কম হবে।

---

## 16. Systemd Timers — Cron এর বিকল্প

`systemd` দিয়ে scheduled task চালানো যায়, cron-এর মতো।

**উদাহরণ: প্রতিদিন রাত ২টায় database backup**

`/etc/systemd/system/db-backup.service`:
```ini
[Unit]
Description=Database Backup

[Service]
Type=oneshot
User=ubuntu
ExecStart=/home/ubuntu/scripts/backup.sh
```

`/etc/systemd/system/db-backup.timer`:
```ini
[Unit]
Description=Run DB Backup Daily

[Timer]
OnCalendar=*-*-* 02:00:00    # প্রতিদিন রাত ২টায়
Persistent=true               # missed হলে পরের বার চালাবে

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl enable --now db-backup.timer
systemctl list-timers   # সব active timer দেখুন
```

### Cron vs Systemd Timer

| Feature | Cron | Systemd Timer |
|---------|------|---------------|
| Log | আলাদা setup লাগে | journalctl এ automatically |
| Dependencies | নেই | সব systemd feature ব্যবহার করা যায় |
| Missed run | skip করে | Persistent=true দিলে পরে চালায় |
| Security | সীমিত | Full sandboxing |
| Complexity | সহজ | একটু জটিল |

---

## 17. Real-world Deployment Workflows

### Django VPS Deployment সম্পূর্ণ Flow

```bash
# ১. Project setup
git clone https://github.com/user/myproject.git /home/ubuntu/myproject
cd /home/ubuntu/myproject
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# ২. Service file তৈরি
sudo nano /etc/systemd/system/myproject.service

# ৩. Service activate
sudo systemctl daemon-reload
sudo systemctl enable --now myproject
sudo systemctl status myproject

# ৪. Log দেখা
journalctl -u myproject -f

# ৫. Code update করার সময়
git pull origin main
pip install -r requirements.txt
python manage.py migrate
sudo systemctl restart myproject
sudo systemctl status myproject
```

---

### Nginx + Gunicorn + Django Full Stack

```bash
# Nginx configure
sudo nano /etc/nginx/sites-available/myproject
sudo ln -s /etc/nginx/sites-available/myproject /etc/nginx/sites-enabled/
sudo nginx -t                           # config test
sudo systemctl reload nginx             # zero-downtime reload

# Stack status একসাথে দেখা
systemctl status nginx myproject postgresql redis
```

---

## 18. Troubleshooting Guide

### Service চালু হচ্ছে না?

```bash
# ১. Status দেখুন
systemctl status myservice

# ২. Log দেখুন
journalctl -u myservice -n 50

# ৩. Error কোথায়?
journalctl -u myservice -p err

# ৪. Service file সঠিক আছে কি না test করুন
systemd-analyze verify /etc/systemd/system/myservice.service
```

### সাধারণ সমস্যা এবং সমাধান

| সমস্যা | সম্ভাব্য কারণ | সমাধান |
|--------|--------------|--------|
| `Failed to start` | ExecStart path ভুল | `which gunicorn` দিয়ে সঠিক path বের করুন |
| `Permission denied` | User-এর permission নেই | `User=` সঠিক দিন, directory permission ঠিক করুন |
| `Unit not found` | daemon-reload দেননি | `sudo systemctl daemon-reload` |
| `Address already in use` | Port already occupied | `sudo lsof -i :PORT` দিয়ে দেখুন |
| Service keeps restarting | App crash হচ্ছে | `journalctl -u service -f` দিয়ে live দেখুন |

---

### `systemd-analyze` — Boot Performance বিশ্লেষণ

```bash
# মোট boot time দেখুন
systemd-analyze

# প্রতিটি service কত সময় নিল
systemd-analyze blame

# Dependency tree দেখুন
systemd-analyze critical-chain nginx.service
```

---

## 19. চিট শিট — Quick Reference

```bash
# ══════════════════════════════════════════
#           SERVICE MANAGEMENT
# ══════════════════════════════════════════
systemctl status <svc>          # Status দেখুন
sudo systemctl start <svc>      # চালু করুন
sudo systemctl stop <svc>       # বন্ধ করুন
sudo systemctl restart <svc>    # Restart করুন
sudo systemctl reload <svc>     # Config reload করুন
sudo systemctl enable <svc>     # Boot-এ enable করুন
sudo systemctl disable <svc>    # Boot-এ disable করুন
sudo systemctl enable --now <svc>   # Enable + এখনই start
sudo systemctl mask <svc>       # সম্পূর্ণ block করুন

# ══════════════════════════════════════════
#           STATUS CHECK
# ══════════════════════════════════════════
systemctl is-active <svc>       # active/inactive
systemctl is-enabled <svc>      # enabled/disabled
systemctl is-failed <svc>       # failed কি না
systemctl --failed               # সব failed service
systemctl list-units --type=service   # সব running service

# ══════════════════════════════════════════
#           LOGS
# ══════════════════════════════════════════
journalctl -u <svc>             # সব log
journalctl -u <svc> -f          # Live log
journalctl -u <svc> -n 100      # শেষ ১০০ line
journalctl -u <svc> -p err      # শুধু error

# ══════════════════════════════════════════
#           SYSTEM CONTROL
# ══════════════════════════════════════════
sudo systemctl daemon-reload    # নতুন service file লোড করুন
sudo systemctl reboot           # Reboot
sudo systemctl poweroff         # Shutdown

# ══════════════════════════════════════════
#           CUSTOM SERVICE WORKFLOW
# ══════════════════════════════════════════
# ১. Service file তৈরি করুন
sudo nano /etc/systemd/system/myapp.service

# ২. Reload করুন
sudo systemctl daemon-reload

# ৩. Enable + Start করুন
sudo systemctl enable --now myapp

# ৪. Status দেখুন
sudo systemctl status myapp

# ৫. Live log দেখুন
journalctl -u myapp -f
```

---

## 🎯 সংক্ষেপ

`systemctl` শেখা মানে Linux server-এ পূর্ণ নিয়ন্ত্রণ পাওয়া। আপনি যদি Django, Spring Boot, Node.js বা যেকোনো backend প্রযুক্তিতে কাজ করতে চান এবং VPS-এ deploy করতে চান — তাহলে এই কমান্ডগুলো আপনার প্রতিদিনের হাতিয়ার।

**মূল বিষয়গুলো মনে রাখুন:**
- Service file → `/etc/systemd/system/` এ রাখুন
- পরিবর্তনের পর সবসময় `daemon-reload` দিন
- `enable --now` = একসাথে start + boot-এ enable
- Log দেখুন `journalctl -u servicename -f` দিয়ে
- `reload` > `restart` (যদি supported হয়)

---

*এই টিউটোরিয়ালটি Ubuntu/Debian ভিত্তিক। CentOS/Fedora-তেও প্রায় একই, শুধু package manager (`yum`/`dnf`) আলাদা।*
