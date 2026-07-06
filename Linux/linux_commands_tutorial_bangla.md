# লিনাক্স কমান্ড লাইন মাস্টার গাইড (বাংলা টিউটোরিয়াল)

এই টিউটোরিয়ালে ১৫টি গুরুত্বপূর্ণ কমান্ড-লাইন টুল বিস্তারিতভাবে আলোচনা করা হয়েছে — প্রতিটির **গঠন (syntax)**, **ব্যবহারের নিয়ম**, এবং **বাস্তব উদাহরণ** সহ। DevOps, Backend Development, Server Management এবং দৈনন্দিন লিনাক্স কাজে এই কমান্ডগুলো অপরিহার্য।

---

## ১. grep — Search (টেক্সট খোঁজা)

### এটা কী?
`grep` (Global Regular Expression Print) দিয়ে ফাইল বা আউটপুটের ভেতর নির্দিষ্ট প্যাটার্ন বা শব্দ খুঁজে বের করা হয়।

### গঠন (Syntax)
```bash
grep [options] "pattern" filename
```

### গুরুত্বপূর্ণ অপশন
| অপশন | কাজ |
|---|---|
| `-i` | Case-insensitive সার্চ |
| `-r` বা `-R` | Recursive — ফোল্ডারের ভেতরে সব ফাইলে খোঁজা |
| `-n` | লাইন নম্বর দেখায় |
| `-v` | যা ম্যাচ হয়নি তা দেখায় (invert match) |
| `-w` | পুরো শব্দ ম্যাচ করে |
| `-c` | কতগুলো ম্যাচ হয়েছে তার সংখ্যা দেখায় |
| `-E` | Extended regex ব্যবহার করে |

### বাস্তব উদাহরণ
```bash
# একটি ফাইলে "error" শব্দ খোঁজা (case-insensitive)
grep -i "error" app.log

# পুরো প্রজেক্ট ফোল্ডারে "TODO" খোঁজা, লাইন নম্বরসহ
grep -rn "TODO" ./src

# লগ ফাইলে যেসব লাইনে "error" নেই সেগুলো দেখানো
grep -v "error" server.log

# একাধিক প্রসেস থেকে nginx প্রসেস খোঁজা
ps aux | grep nginx
```

### Specific Use Case
- সার্ভারের লগ ফাইল থেকে নির্দিষ্ট error ট্রেস করা
- কোডবেসে কোনো ফাংশন/ভ্যারিয়েবল কোথায় ব্যবহৃত হয়েছে তা খুঁজে বের করা
- `ps aux | grep` দিয়ে চলমান প্রসেস চেক করা

---

## ২. sed — Replace / Edit (টেক্সট পরিবর্তন)

### এটা কী?
`sed` (Stream Editor) দিয়ে ফাইলের ভেতরে টেক্সট খোঁজা এবং পরিবর্তন (replace) করা যায়, ফাইল না খুলেই।

### গঠন (Syntax)
```bash
sed 's/pattern/replacement/flags' filename
```

### গুরুত্বপূর্ণ অংশ
- `s` = substitute (পরিবর্তন করা)
- `g` = পুরো লাইনে সব ম্যাচ পরিবর্তন হবে (global)
- `-i` = সরাসরি ফাইলে পরিবর্তন সেভ হবে (in-place)

### বাস্তব উদাহরণ
```bash
# "old" কে "new" দিয়ে প্রতিস্থাপন (শুধু স্ক্রিনে দেখাবে)
sed 's/old/new/' file.txt

# পুরো লাইনে সব "old" কে "new" করা
sed 's/old/new/g' file.txt

# সরাসরি ফাইলে পরিবর্তন সেভ করা
sed -i 's/localhost/127.0.0.1/g' config.env

# নির্দিষ্ট লাইন নম্বর ডিলিট করা (যেমন ৩ নম্বর লাইন)
sed -i '3d' file.txt

# একাধিক ফাইলে একসাথে replace করা
sed -i 's/v1.0/v2.0/g' *.txt
```

### Specific Use Case
- Deployment এর সময় config ফাইলে environment variable বদলানো
- বাল্ক ফাইলে ভার্সন নম্বর বা URL আপডেট করা
- CI/CD script এ automated টেক্সট পরিবর্তন

---

## ৩. awk — Text Processing (কলাম ভিত্তিক প্রসেসিং)

### এটা কী?
`awk` একটি প্রোগ্রামিং ভাষার মতো টুল, যা টেক্সট ফাইলকে **কলাম (field)** ও **রো (record)** হিসেবে প্রসেস করে। CSV, লগ ফাইল অ্যানালাইসিসে অসাধারণ কার্যকর।

### গঠন (Syntax)
```bash
awk '{ action }' filename
awk -F"delimiter" '{ print $column }' filename
```

- `$1, $2, $3...` = ১ম, ২য়, ৩য় কলাম
- `$0` = পুরো লাইন
- `-F` = ডিলিমিটার (default স্পেস)
- `NR` = বর্তমান লাইন নম্বর
- `NF` = মোট কলাম সংখ্যা

### বাস্তব উদাহরণ
```bash
# প্রতিটি লাইনের ২য় কলাম প্রিন্ট করা
awk '{ print $2 }' data.txt

# CSV ফাইল থেকে ৩য় কলাম বের করা (কমা দিয়ে সেপারেটেড)
awk -F"," '{ print $3 }' data.csv

# শর্তসাপেক্ষে প্রিন্ট করা (যেমন ৫০০ এর বেশি মেমোরি ব্যবহারকারী প্রসেস)
ps aux | awk '$4 > 5.0 { print $2, $11 }'

# কলাম সমষ্টি (sum) বের করা
awk '{ sum += $1 } END { print sum }' numbers.txt

# লাইন নম্বরসহ প্রিন্ট করা
awk '{ print NR, $0 }' file.txt
```

### Specific Use Case
- Access log থেকে IP address বা status code এক্সট্র্যাক্ট করা
- CSV/ডেটা ফাইল থেকে নির্দিষ্ট কলাম আলাদা করে রিপোর্ট বানানো
- সার্ভার মনিটরিং স্ক্রিপ্টে সংখ্যাসূচক ডেটা যোগ/বিশ্লেষণ করা

---

## ৪. find — File Search (ফাইল খোঁজা)

### এটা কী?
`find` দিয়ে ফোল্ডার স্ট্রাকচারের ভেতরে নাম, টাইপ, সাইজ, তারিখ ইত্যাদি শর্ত অনুযায়ী ফাইল/ফোল্ডার খোঁজা যায়।

### গঠন (Syntax)
```bash
find [path] [options] [expression]
```

### গুরুত্বপূর্ণ অপশন
| অপশন | কাজ |
|---|---|
| `-name` | নাম অনুযায়ী খোঁজা |
| `-type f/d` | ফাইল (f) নাকি ফোল্ডার (d) |
| `-size` | সাইজ অনুযায়ী |
| `-mtime` | কতদিন আগে পরিবর্তিত হয়েছে |
| `-exec` | পাওয়া ফাইলের উপর কমান্ড রান করা |

### বাস্তব উদাহরণ
```bash
# .log এক্সটেনশনের সব ফাইল খোঁজা
find /var/log -name "*.log"

# ৭ দিনের বেশি পুরনো ফাইল খুঁজে ডিলিট করা
find /tmp -type f -mtime +7 -exec rm {} \;

# ১০০MB এর চেয়ে বড় ফাইল খোঁজা
find / -type f -size +100M

# শুধু ফোল্ডার খোঁজা
find . -type d -name "node_modules"

# খুঁজে পাওয়া সব ফাইলের পারমিশন পরিবর্তন
find . -type f -name "*.sh" -exec chmod +x {} \;
```

### Specific Use Case
- পুরনো/অপ্রয়োজনীয় লগ বা ক্যাশ ফাইল অটোমেটিক ডিলিট করা (cron job)
- ডিস্ক স্পেস বাঁচাতে বড় ফাইল খুঁজে বের করা
- `node_modules` বা `.git` এর মতো ফোল্ডার একসাথে খুঁজে ক্লিন করা

---

## ৫. jq — JSON Parsing

### এটা কী?
`jq` একটি কমান্ড-লাইন JSON প্রসেসর — API রেসপন্স বা JSON ফাইল থেকে ডেটা এক্সট্র্যাক্ট, ফিল্টার ও ফরম্যাট করার জন্য ব্যবহৃত হয়।

### গঠন (Syntax)
```bash
jq 'filter' file.json
command | jq 'filter'
```

### বাস্তব উদাহরণ
```bash
# পুরো JSON সুন্দরভাবে ফরম্যাট করে দেখানো (pretty print)
cat data.json | jq '.'

# নির্দিষ্ট ফিল্ড বের করা
cat user.json | jq '.name'

# nested ফিল্ড এক্সেস করা
cat user.json | jq '.address.city'

# array থেকে সব আইটেমের একটি ফিল্ড বের করা
cat users.json | jq '.[].email'

# curl API রেসপন্স থেকে সরাসরি ডেটা বের করা
curl -s https://api.github.com/users/octocat | jq '.name, .public_repos'

# শর্তসাপেক্ষে ফিল্টার করা
cat users.json | jq '.[] | select(.age > 18)'
```

### Specific Use Case
- API রেসপন্স টেস্ট করার সময় দরকারি ডেটা দ্রুত বের করা
- CI/CD পাইপলাইনে JSON কনফিগ ফাইল পার্স করা
- Automation script এ API থেকে পাওয়া JSON প্রসেস করা

---

## ৬. curl — API Testing

### এটা কী?
`curl` দিয়ে যেকোনো URL/API-তে HTTP রিকোয়েস্ট পাঠানো যায় — GET, POST, PUT, DELETE ইত্যাদি। API টেস্টিং ও ডিবাগিংয়ের সবচেয়ে জনপ্রিয় টুল।

### গঠন (Syntax)
```bash
curl [options] URL
```

### গুরুত্বপূর্ণ অপশন
| অপশন | কাজ |
|---|---|
| `-X` | HTTP method নির্ধারণ (GET, POST...) |
| `-H` | Header পাঠানো |
| `-d` | Data/body পাঠানো (POST/PUT) |
| `-o` | রেসপন্স ফাইলে সেভ করা |
| `-I` | শুধু header দেখানো |
| `-s` | Silent mode (progress bar ছাড়া) |
| `-v` | Verbose (ডিবাগ তথ্যসহ) |

### বাস্তব উদাহরণ
```bash
# সিম্পল GET রিকোয়েস্ট
curl https://api.example.com/users

# JSON বডি সহ POST রিকোয়েস্ট
curl -X POST https://api.example.com/users \
  -H "Content-Type: application/json" \
  -d '{"name": "Rafi", "age": 25}'

# Authorization header সহ রিকোয়েস্ট
curl -H "Authorization: Bearer TOKEN123" https://api.example.com/profile

# রেসপন্স ফাইলে সেভ করা
curl -o output.json https://api.example.com/data

# শুধু status code/header চেক করা
curl -I https://example.com
```

### Specific Use Case
- ব্যাকএন্ড API ডেভেলপ করার সময় দ্রুত এন্ডপয়েন্ট টেস্ট করা
- সার্ভার আপ আছে কিনা তা health-check করা
- Webhook বা থার্ড-পার্টি API ইন্টিগ্রেশন ডিবাগ করা

---

## ৭. docker — Container Management

### এটা কী?
`docker` দিয়ে অ্যাপ্লিকেশনকে **container** এ প্যাকেজ করে যেকোনো এনভায়রনমেন্টে একইভাবে চালানো যায়।

### গঠন (Syntax)
```bash
docker [command] [options]
```

### গুরুত্বপূর্ণ কমান্ড
| কমান্ড | কাজ |
|---|---|
| `docker build` | Image তৈরি করা |
| `docker run` | Container চালু করা |
| `docker ps` | চলমান container লিস্ট |
| `docker stop/start` | Container বন্ধ/চালু করা |
| `docker logs` | Container এর লগ দেখা |
| `docker exec` | চলমান container এর ভেতরে কমান্ড চালানো |
| `docker-compose up` | মাল্টি-কন্টেইনার অ্যাপ চালু করা |

### বাস্তব উদাহরণ
```bash
# Dockerfile থেকে image বানানো
docker build -t myapp:1.0 .

# Container ব্যাকগ্রাউন্ডে চালানো, পোর্ট ম্যাপিং সহ
docker run -d -p 8080:80 --name myapp-container myapp:1.0

# চলমান সব container দেখা
docker ps

# একটি container এর লগ live দেখা
docker logs -f myapp-container

# চলমান container এর ভেতরে bash শেলে ঢোকা
docker exec -it myapp-container bash

# docker-compose দিয়ে একাধিক সার্ভিস চালু করা
docker-compose up -d
```

### Specific Use Case
- Local ডেভেলপমেন্ট এনভায়রনমেন্ট প্রোডাকশনের মতো সেটআপ করা
- Microservices আর্কিটেকচার ডিপ্লয় করা
- CI/CD পাইপলাইনে consistent বিল্ড ও টেস্ট এনভায়রনমেন্ট তৈরি করা

---

## ৮. git — Version Control

### এটা কী?
`git` দিয়ে কোডের ভার্সন ট্র্যাক করা, দলগতভাবে কাজ করা এবং কোডের ইতিহাস ম্যানেজ করা হয়।

### গঠন (Syntax)
```bash
git [command] [options]
```

### গুরুত্বপূর্ণ কমান্ড
| কমান্ড | কাজ |
|---|---|
| `git clone` | রিপো ডাউনলোড করা |
| `git status` | বর্তমান অবস্থা দেখা |
| `git add` | পরিবর্তন স্টেজ করা |
| `git commit` | পরিবর্তন সেভ করা |
| `git push/pull` | রিমোটে পাঠানো/আনা |
| `git branch` | ব্রাঞ্চ তৈরি/লিস্ট |
| `git merge` | ব্রাঞ্চ একত্র করা |
| `git log` | কমিট হিস্ট্রি দেখা |

### বাস্তব উদাহরণ
```bash
# রিপো ক্লোন করা
git clone https://github.com/user/repo.git

# পরিবর্তনগুলো স্টেজ ও কমিট করা
git add .
git commit -m "Fix login bug"

# রিমোটে পাঠানো
git push origin main

# নতুন ব্রাঞ্চ তৈরি করে সুইচ করা
git checkout -b feature/new-ui

# অন্য ব্রাঞ্চ মার্জ করা
git merge feature/new-ui

# শেষ ৫টি কমিট দেখা (এক লাইনে)
git log --oneline -5
```

### Specific Use Case
- টিমে কোড কোলাবোরেশন ও কনফ্লিক্ট রেজোলিউশন
- ফিচার ব্রাঞ্চ তৈরি করে আলাদাভাবে ডেভেলপ করা
- প্রোডাকশনে সমস্যা হলে আগের কমিটে rollback করা

---

## ৯. systemctl — Service Management

### এটা কী?
`systemctl` দিয়ে লিনাক্স সার্ভিস (যেমন nginx, mysql, ssh) চালু, বন্ধ, রিস্টার্ট এবং boot-এ অটো-স্টার্ট কনফিগার করা হয়।

### গঠন (Syntax)
```bash
systemctl [command] [service-name]
```

### গুরুত্বপূর্ণ কমান্ড
| কমান্ড | কাজ |
|---|---|
| `start` | সার্ভিস চালু করা |
| `stop` | সার্ভিস বন্ধ করা |
| `restart` | রিস্টার্ট করা |
| `status` | বর্তমান অবস্থা দেখা |
| `enable` | সিস্টেম বুটের সাথে অটো-স্টার্ট চালু করা |
| `disable` | অটো-স্টার্ট বন্ধ করা |

### বাস্তব উদাহরণ
```bash
# nginx সার্ভিস চালু করা
sudo systemctl start nginx

# সার্ভিসের বর্তমান অবস্থা দেখা
sudo systemctl status nginx

# কনফিগ পরিবর্তনের পর রিস্টার্ট করা
sudo systemctl restart nginx

# সার্ভার রিবুট হলেও যেন সার্ভিস অটো চালু হয়
sudo systemctl enable nginx

# সব চলমান সার্ভিস লিস্ট করা
systemctl list-units --type=service --state=running
```

### Specific Use Case
- সার্ভারে ওয়েব সার্ভার/ডাটাবেজ সার্ভিস ম্যানেজ করা
- ডিপ্লয়মেন্টের পর অ্যাপ্লিকেশন সার্ভিস রিস্টার্ট করা
- সার্ভার বুট হওয়ার সাথে সাথে প্রয়োজনীয় সার্ভিস চালু নিশ্চিত করা

---

## ১০. journalctl — Log Checking

### এটা কী?
`journalctl` দিয়ে systemd দ্বারা পরিচালিত সিস্টেম ও সার্ভিসের লগ দেখা যায়।

### গঠন (Syntax)
```bash
journalctl [options]
```

### গুরুত্বপূর্ণ অপশন
| অপশন | কাজ |
|---|---|
| `-u [service]` | নির্দিষ্ট সার্ভিসের লগ |
| `-f` | Live/follow মোডে লগ দেখা |
| `-b` | শেষ বুট থেকে লগ |
| `--since` | নির্দিষ্ট সময় থেকে লগ |
| `-p err` | শুধু error লগ দেখানো |

### বাস্তব উদাহরণ
```bash
# nginx সার্ভিসের লগ দেখা
journalctl -u nginx

# লাইভ লগ দেখা (নতুন এন্ট্রি আসলে সাথে সাথে দেখাবে)
journalctl -u nginx -f

# শুধু আজকের লগ দেখা
journalctl --since today

# শুধু error লেভেলের লগ দেখা
journalctl -p err

# শেষ ৫০ লাইন লগ দেখা
journalctl -u nginx -n 50
```

### Specific Use Case
- সার্ভিস ক্র্যাশ হলে কারণ খুঁজে বের করা
- সার্ভার বুট সমস্যা ডিবাগ করা
- প্রোডাকশন ইস্যু রিয়েল-টাইমে মনিটর করা

---

## ১১. rsync — Deployment (ফাইল সিঙ্ক্রোনাইজেশন)

### এটা কী?
`rsync` দিয়ে দুই ফোল্ডার/সার্ভারের মধ্যে দ্রুত ও efficient ভাবে ফাইল সিঙ্ক করা যায় — শুধু পরিবর্তিত অংশ ট্রান্সফার হয় বলে এটা `cp` বা `scp` এর চেয়ে অনেক দ্রুত।

### গঠন (Syntax)
```bash
rsync [options] source destination
```

### গুরুত্বপূর্ণ অপশন
| অপশন | কাজ |
|---|---|
| `-a` | Archive mode (permission, timestamp সব বজায় রাখে) |
| `-v` | Verbose — কী ট্রান্সফার হচ্ছে দেখায় |
| `-z` | ট্রান্সফারের সময় compress করে |
| `--delete` | সোর্সে না থাকা ফাইল ডেস্টিনেশন থেকে ডিলিট করে |
| `-e ssh` | SSH ব্যবহার করে রিমোট সার্ভারে সিঙ্ক |

### বাস্তব উদাহরণ
```bash
# লোকাল ফোল্ডার সিঙ্ক করা
rsync -av /home/user/project/ /backup/project/

# রিমোট সার্ভারে ডিপ্লয় করা (SSH দিয়ে)
rsync -avz -e ssh ./build/ user@server:/var/www/myapp/

# সোর্সে না থাকা ফাইল ডেস্টিনেশন থেকেও মুছে ফেলা (mirror sync)
rsync -av --delete ./dist/ user@server:/var/www/app/

# ড্রাই-রান (আসলে কপি না করে শুধু কী হবে দেখা)
rsync -avn ./project/ /backup/project/
```

### Specific Use Case
- প্রোডাকশন সার্ভারে বিল্ড ফাইল ডিপ্লয় করা
- সার্ভার ব্যাকআপ নেওয়া
- দুই সার্ভারের মধ্যে ডেটা সিঙ্ক্রোনাইজেশন

---

## ১২. tail -f — Live Logs

### এটা কী?
`tail` ফাইলের শেষের কয়েক লাইন দেখায়; `-f` (follow) অপশন দিয়ে লাইভ/রিয়েল-টাইম লগ মনিটর করা যায়।

### গঠন (Syntax)
```bash
tail [options] filename
```

### গুরুত্বপূর্ণ অপশন
| অপশন | কাজ |
|---|---|
| `-f` | ফাইলে নতুন লাইন যোগ হলে সাথে সাথে দেখানো |
| `-n [number]` | নির্দিষ্ট সংখ্যক শেষ লাইন দেখানো |
| `-F` | ফাইল rotate হলেও ফলো করে |

### বাস্তব উদাহরণ
```bash
# লগ ফাইলের শেষ ১০ লাইন দেখা (ডিফল্ট)
tail app.log

# লাইভ মোডে লগ ফাইল মনিটর করা
tail -f app.log

# শেষ ৫০ লাইন দেখে তারপর লাইভ ফলো করা
tail -n 50 -f app.log

# একসাথে একাধিক লগ ফাইল লাইভ দেখা
tail -f access.log error.log

# grep এর সাথে কম্বাইন করে নির্দিষ্ট error লাইভ দেখা
tail -f app.log | grep "ERROR"
```

### Specific Use Case
- প্রোডাকশন সার্ভারে রিয়েল-টাইমে অ্যাপ্লিকেশন এরর মনিটর করা
- ডিপ্লয়মেন্টের সময় লাইভ লগ দেখে ইস্যু ধরা
- API রিকোয়েস্ট আসা-যাওয়া লাইভ ট্র্যাক করা

---

## ১৩. xargs — Command Chaining

### এটা কী?
`xargs` একটি কমান্ডের আউটপুটকে আরেকটি কমান্ডের **argument** হিসেবে পাঠাতে ব্যবহৃত হয় — pipe (`|`) দিয়ে যা সরাসরি সম্ভব না তা `xargs` দিয়ে সম্ভব হয়।

### গঠন (Syntax)
```bash
command1 | xargs [options] command2
```

### গুরুত্বপূর্ণ অপশন
| অপশন | কাজ |
|---|---|
| `-I {}` | প্রতিটি ইনপুটকে `{}` দিয়ে রিপ্রেজেন্ট করে |
| `-n [number]` | একবারে কতগুলো আর্গুমেন্ট পাঠাবে |
| `-p` | রান করার আগে কনফার্মেশন চাইবে |
| `-0` | null-delimited ইনপুট হ্যান্ডেল করে (স্পেস থাকা ফাইলনামের জন্য নিরাপদ) |

### বাস্তব উদাহরণ
```bash
# find দিয়ে পাওয়া ফাইলগুলো একসাথে ডিলিট করা
find . -name "*.tmp" | xargs rm

# প্রতিটি ফাইলে আলাদাভাবে কমান্ড চালানো
find . -name "*.txt" | xargs -I {} cp {} /backup/

# স্পেসসহ ফাইলনাম নিরাপদে হ্যান্ডেল করা
find . -name "*.log" -print0 | xargs -0 rm

# একসাথে একাধিক প্যাকেজ ইনস্টল করা (লিস্ট থেকে)
cat packages.txt | xargs sudo apt-get install -y

# চালানোর আগে কনফার্মেশন নেওয়া
find . -name "*.bak" | xargs -p rm
```

### Specific Use Case
- হাজার হাজার ফাইলের উপর বাল্ক অপারেশন (ডিলিট/কপি/মুভ) চালানো
- `find` এর রেজাল্টের উপর কমান্ড চালিয়ে ব্যাচ প্রসেসিং করা
- স্ক্রিপ্টে একাধিক ইনপুট থেকে প্যারালাল/সিকোয়েনশিয়াল কমান্ড রান করা

---

## ১৪. cut — Field Extraction

### এটা কী?
`cut` দিয়ে প্রতিটি লাইন থেকে নির্দিষ্ট কলাম বা ক্যারেক্টার পজিশন এক্সট্র্যাক্ট করা যায় — `awk` এর চেয়ে সিম্পল কাজের জন্য দ্রুত।

### গঠন (Syntax)
```bash
cut [options] filename
```

### গুরুত্বপূর্ণ অপশন
| অপশন | কাজ |
|---|---|
| `-d` | ডিলিমিটার নির্ধারণ (যেমন কমা, ট্যাব) |
| `-f` | কোন ফিল্ড/কলাম চাই তা নির্ধারণ |
| `-c` | ক্যারেক্টার পজিশন অনুযায়ী কাটা |

### বাস্তব উদাহরণ
```bash
# /etc/passwd থেকে ইউজারনেম বের করা (কোলন ডিলিমিটার)
cut -d: -f1 /etc/passwd

# CSV ফাইল থেকে ১ম ও ৩য় কলাম বের করা
cut -d, -f1,3 data.csv

# প্রতিটি লাইনের প্রথম ৫টি ক্যারেক্টার বের করা
cut -c1-5 file.txt

# df কমান্ডের আউটপুট থেকে শুধু ব্যবহৃত স্পেস% কলাম বের করা
df -h | cut -d' ' -f5-6
```

### Specific Use Case
- লগ ফাইল থেকে নির্দিষ্ট কলাম (যেমন IP address, timestamp) দ্রুত বের করা
- সিস্টেম কনফিগ ফাইল (`/etc/passwd`, `/etc/group`) থেকে তথ্য বের করা
- সিম্পল CSV প্রসেসিং যেখানে পুরো `awk` দরকার নেই

---

## ১৫. sort ও uniq — Data Processing

### এটা কী?
`sort` ডেটা সাজায় (alphabetically/numerically), আর `uniq` পরপর থাকা ডুপ্লিকেট লাইন সরিয়ে দেয় বা কতবার এসেছে তা গোনে। এই দুটি প্রায়ই একসাথে ব্যবহৃত হয়।

### গঠন (Syntax)
```bash
sort [options] filename
uniq [options] filename
```

### গুরুত্বপূর্ণ অপশন

**sort:**
| অপশন | কাজ |
|---|---|
| `-n` | সংখ্যা অনুযায়ী সর্ট করা |
| `-r` | উল্টো ক্রমে (reverse) সর্ট করা |
| `-k` | নির্দিষ্ট কলাম অনুযায়ী সর্ট করা |
| `-u` | ইউনিক করে সর্ট করা |

**uniq:**
| অপশন | কাজ |
|---|---|
| `-c` | কতবার প্রতিটি লাইন এসেছে তা গোনে |
| `-d` | শুধু ডুপ্লিকেট লাইন দেখায় |

### বাস্তব উদাহরণ
```bash
# ফাইলের লাইনগুলো বর্ণানুক্রমে সাজানো
sort names.txt

# সংখ্যা অনুযায়ী সর্ট করা
sort -n numbers.txt

# **গুরুত্বপূর্ণ**: uniq ঠিকভাবে কাজ করতে হলে আগে sort করতে হবে
sort access.log | uniq

# কতবার প্রতিটি IP এসেছে তা গোনা (সবচেয়ে বেশি ব্যবহৃত প্যাটার্ন)
awk '{print $1}' access.log | sort | uniq -c | sort -nr

# শুধু ডুপ্লিকেট এন্ট্রি খুঁজে বের করা
sort emails.txt | uniq -d
```

### Specific Use Case
- Access log বিশ্লেষণ করে সবচেয়ে বেশি ভিজিট করা IP/URL বের করা (`awk + sort + uniq -c` কম্বো)
- ডেটাসেট থেকে ডুপ্লিকেট এন্ট্রি খুঁজে বের করে ক্লিন করা
- বড় ডেটা ফাইল অ্যানালাইসিসের আগে সর্ট করে অর্গানাইজ করা

---

## বোনাস: কমান্ডগুলো একসাথে কম্বাইন করা (Real-world Pipeline)

লিনাক্সের আসল শক্তি হলো `|` (pipe) দিয়ে একাধিক কমান্ড একসাথে চেইন করা। নিচের উদাহরণটি দেখুন:

```bash
# Access log থেকে সবচেয়ে বেশি রিকোয়েস্ট পাঠানো টপ ৫টি IP বের করা
cat access.log | awk '{print $1}' | sort | uniq -c | sort -nr | head -5

# API থেকে ডেটা এনে, JSON পার্স করে, নির্দিষ্ট ফাইল খুঁজে সেটা আপডেট করা
curl -s https://api.example.com/config | jq '.version' 
find . -name "config.yaml" | xargs sed -i 's/version:.*/version: 2.0/'

# চলমান docker container গুলোর লগে error খোঁজা
docker ps -q | xargs -I {} docker logs {} 2>&1 | grep -i error
```

---

## সারসংক্ষেপ টেবিল

| কমান্ড | মূল কাজ | কখন ব্যবহার করবেন |
|---|---|---|
| grep | সার্চ | টেক্সট/লগে প্যাটার্ন খুঁজতে |
| sed | এডিট | ফাইলে টেক্সট রিপ্লেস করতে |
| awk | প্রসেসিং | কলাম-ভিত্তিক ডেটা বিশ্লেষণে |
| find | ফাইল খোঁজা | ফাইল সিস্টেমে সার্চ করতে |
| jq | JSON পার্সিং | API রেসপন্স হ্যান্ডেল করতে |
| curl | API টেস্টিং | HTTP রিকোয়েস্ট পাঠাতে |
| docker | কন্টেইনার | অ্যাপ প্যাকেজিং ও ডিপ্লয়ে |
| git | ভার্সন কন্ট্রোল | কোড ম্যানেজমেন্টে |
| systemctl | সার্ভিস ম্যানেজমেন্ট | সার্ভিস চালু/বন্ধ করতে |
| journalctl | লগ চেকিং | সিস্টেম লগ দেখতে |
| rsync | ডিপ্লয়মেন্ট | ফাইল সিঙ্ক/ব্যাকআপে |
| tail -f | লাইভ লগ | রিয়েল-টাইম মনিটরিংয়ে |
| xargs | কমান্ড চেইনিং | বাল্ক অপারেশনে |
| cut | ফিল্ড এক্সট্র্যাকশন | কলাম বের করতে |
| sort, uniq | ডেটা প্রসেসিং | সাজানো ও ডুপ্লিকেট বের করতে |

---

### শেখার পরামর্শ
১. প্রতিটি কমান্ড আলাদাভাবে টার্মিনালে নিজে চালিয়ে দেখুন।
২. একটি ছোট টেস্ট ফাইল/লগ তৈরি করে তার উপর প্র্যাকটিস করুন।
৩. `man command_name` (যেমন `man grep`) দিয়ে বিস্তারিত ডকুমেন্টেশন পড়ুন।
৪. ধীরে ধীরে কমান্ডগুলো `|` দিয়ে কম্বাইন করে জটিল pipeline বানানো প্র্যাকটিস করুন — এটাই আসল দক্ষতা।
