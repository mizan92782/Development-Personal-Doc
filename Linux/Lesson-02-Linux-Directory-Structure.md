# Lesson 2: Linux Directory Structure (File System Hierarchy)

আজ আমরা Linux-এর সবচেয়ে গুরুত্বপূর্ণ Topic শিখব।

যদি Linux-কে একটি শহর ধরি, তাহলে প্রতিটি Directory হলো সেই শহরের আলাদা বিভাগ।

| Directory | উপমা |
|-----------|-------|
| `/home` | মানুষের বাসা |
| `/etc` | সরকারি অফিস (Configuration) |
| `/var` | গুদাম (Logs, Database, Cache) |
| `/usr` | Library + Software |
| `/tmp` | অস্থায়ী জিনিস রাখার ঘর |

---

## Linux File System Tree

```
                     /
                     │
 ┌───────────────────┼─────────────────────┐
 │                   │                     │
home               etc                  var
 │                   │                     │
mizan          nginx.conf             log
 │                   │                     │
Documents       passwd               nginx.log
```

এখানে `/` হলো **Root Directory**।
Linux-এ সবকিছু `/` থেকে শুরু হয়।

---

## ১. / (Root Directory)

এটাই পুরো Linux-এর শুরু।

```bash
cd /
```

আপনি পুরো Operating System-এর সর্বোচ্চ Directory-তে চলে যান।

```bash
ls /
```

Output:
```
bin  boot  dev  etc  home  lib  media  mnt  opt
proc  root  run  sbin  srv  sys  tmp  usr  var
```

---

## ২. /home

এটি সবচেয়ে বেশি ব্যবহৃত Directory। এখানে সাধারণ User-দের ব্যক্তিগত ফাইল থাকে।

```
/home/
    mizan/
        Documents/
        Downloads/
        Pictures/
        Videos/
```

আপনার Ubuntu User যদি `mizan` হয়:

```bash
pwd
```

Output:
```
/home/mizan
```

**এখানে কী থাকে?**
- Documents, Downloads, Desktop
- Flutter Project, Django Project, Source Code

উদাহরণ — Django Project:
```
/home/mizan/shop_project
```

---

## ৩. /root

> অনেকে ভুল করে মনে করেন Root Directory আর Root User একই — না।

| Path | অর্থ |
|------|-------|
| `/` | Root Directory (পুরো System-এর শুরু) |
| `/root` | Root User-এর Home Directory |
| `/home/mizan` | mizan User-এর Home Directory |

---

## ৪. /etc

Linux-এর **Configuration Center**। সব Configuration File এখানে থাকে।

```
/etc/nginx/
/etc/ssh/
/etc/systemd/
/etc/passwd
```

**Django Deployment-এ:**

```bash
# Nginx Config
/etc/nginx/sites-available/

# Systemd Service File
/etc/systemd/system/
```

---

## ৫. /var

**VAR = Variable** — যেসব Data পরিবর্তন হয়।

- Logs
- Database
- Cache, Mail, Web Files

### /var/log — সব Log এখানে

```
/var/log/syslog
/var/log/auth.log
/var/log/nginx/
```

> Server-এ Error হলে প্রথমে `/var/log/` দেখুন।

### /var/www — Web Server Files

Apache বা Nginx-এর Website Files এখানে থাকে।

---

## ৬. /usr

এখানে **Installed Software** থাকে।

> `USR` মানে User না — এটি ঐতিহাসিক কারণে এই নাম।

```
/usr/bin
/usr/lib
/usr/share
/usr/local
```

### /usr/bin — Programs

```
python  git  nano  vim  node
```

```bash
which python3
# Output: /usr/bin/python3
```

---

## ৭. /bin

খুব গুরুত্বপূর্ণ Basic Commands এখানে থাকে।

```
ls  cp  mv  cat  echo
```

```bash
which ls
# Output: /bin/ls
```

> **Note:** বর্তমান Ubuntu-তে `/bin` অনেক সময় `/usr/bin`-এর সাথে merge করা থাকে।

---

## ৮. /sbin

**System Administration Commands:**

```
shutdown  reboot  fsck
```

---

## ৯. /boot

Operating System Boot Files:
- Linux Kernel
- Boot Loader (GRUB)

---

## ১০. /tmp

**Temporary Files** — Program চালালে এখানে Temporary File তৈরি হয়।

> Reboot করলে অনেক ক্ষেত্রে এগুলো Delete হয়ে যায়।

---

## ১১. /dev

Linux-এর সবচেয়ে মজার Directory।

> **"Everything is a file"** — Linux-এর দর্শন।

| Device | File |
|--------|------|
| Keyboard | ফাইল |
| Mouse | ফাইল |
| Hard Disk | ফাইল |
| USB | ফাইল |

```
/dev/sda      → প্রথম Hard Disk
/dev/sdb      → দ্বিতীয় Hard Disk
/dev/null     → Black Hole (সব হারিয়ে যায়)
/dev/random   → Random Data
```

### /dev/null — Special File

```bash
echo hello > /dev/null
# Output: কিছুই না
```

---

## ১২. /proc

এটি আসলে Real Directory না — **Virtual File System**।

এখানে Process Information থাকে।

```bash
cat /proc/cpuinfo    # CPU তথ্য
cat /proc/meminfo    # Memory তথ্য
```

---

## ১৩. /media

USB Drive লাগালে এখানে **Auto Mount** হয়।

---

## ১৪. /mnt

**Manual Mount** করার জন্য ব্যবহৃত হয়।

---

## ১৫. /opt

Third-party Software যেমন Google Chrome, Android Studio কখনও কখনও এখানে Install হয়।

---

## সবচেয়ে গুরুত্বপূর্ণ Directory (মনে রাখুন)

| Directory | কাজ |
|-----------|-----|
| `/` | Root Directory |
| `/home` | User-এর ফাইল |
| `/root` | Root User-এর Home |
| `/etc` | Configuration Files |
| `/var` | Logs, Cache, Database |
| `/usr` | Installed Software |
| `/bin` | Basic Commands |
| `/sbin` | System Commands |
| `/boot` | Boot Files |
| `/tmp` | Temporary Files |
| `/dev` | Devices |
| `/proc` | Process Information |
| `/media` | USB, CD |
| `/mnt` | Manual Mount |
| `/opt` | Third-party Applications |
