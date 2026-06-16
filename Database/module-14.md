# MODULE 14: Backup and Recovery (ব্যাকআপ এবং রিকভারি)
## বাংলায় সম্পূর্ণ Backup & Recovery গাইড — Production Level

> **কেন এই module জীবন বাঁচাবে?**
> Production এ database crash, accidental DELETE, ransomware attack —
> এই মুহূর্তে backup strategy না থাকলে কোম্পানি শেষ।
> "Backup না থাকা মানে data নেই" — এটা production reality।
> Senior engineer হিসেবে backup এবং recovery plan তোমার দায়িত্ব।
> অনেক company এই ভুলে কোটি টাকা হারিয়েছে।

---

## 14.1 কেন Backup দরকার?

### Real Production Incidents

```
Incident 1 — GitLab (2017):
  একজন engineer ভুলে production database DELETE করে।
  Backup ছিল কিন্তু restore test করা হয়নি।
  Backup corrupt ছিল!
  6 hours of data lost। $1M+ loss।
  Lesson: Backup test করো — untested backup = no backup।

Incident 2 — MySpace (2019):
  3 years এর music data permanently deleted।
  Migration এ bug।
  Backup থেকে restore possible ছিল না।
  50 million songs lost forever।

Incident 3 — Toy startup (2020):
  Ransomware attack।
  Production database encrypted।
  No offsite backup।
  Company shut down।

Incident 4 — Bangladeshi Bank:
  DBA ভুলে WHERE clause ছাড়া UPDATE চালায়।
  সব customers এর balance 0 হয়ে যায়।
  PITR দিয়ে 2 hours আগের state restore করা হয়।
```

### Backup এর প্রয়োজনীয়তা

```
কেন data হারায়:
  1. Human error (most common!)     → Accidental DELETE/DROP
  2. Software bugs                  → Bad migration
  3. Hardware failure               → Disk crash
  4. Ransomware/Cyberattack         → Data encrypted/deleted
  5. Natural disaster               → Data center fire/flood
  6. Corruption                     → Bit rot, storage failure
```

---

## 14.2 Backup Types (ব্যাকআপের ধরন)

### Full Backup

```
সম্পূর্ণ database এর snapshot।
সবচেয়ে simple কিন্তু সবচেয়ে বড় file।

✅ Simple restore
✅ Self-contained
❌ Storage বেশি লাগে
❌ সময় বেশি লাগে (large databases এ)
❌ প্রতিদিন full backup = expensive
```

### Incremental Backup

```
শেষ backup এর পর থেকে যা পরিবর্তন হয়েছে শুধু সেটা।

Sunday:   Full backup (10GB)
Monday:   Incremental (500MB) — Sunday এর পর changes
Tuesday:  Incremental (300MB) — Monday এর পর changes
...

✅ Storage efficient
✅ দ্রুত
❌ Restore জটিল (Full + সব Incremental apply করতে হবে)
```

### Differential Backup

```
শেষ Full backup এর পর থেকে সব changes।

Sunday:   Full backup (10GB)
Monday:   Differential (500MB)  — Sunday এর পর সব changes
Tuesday:  Differential (800MB)  — Sunday এর পর সব changes
Wednesday:Differential (1.2GB)  — Sunday এর পর সব changes

✅ Incremental থেকে simple restore (Full + latest Differential)
❌ Incremental থেকে বড় file
```

### WAL Archiving (Continuous Backup)

```
PostgreSQL WAL files continuously archive করা।

WAL = প্রতিটা transaction এর log।
WAL archive = প্রতিটা মুহূর্তের state।
PITR = যেকোনো সময়ে restore করা যায়।

✅ Point-in-Time Recovery (PITR) — most powerful!
✅ Minimal data loss (seconds)
❌ Storage বেশি লাগে
❌ Setup complex
```

---

## 14.3 pg_dump — Logical Backup

### Basic Usage

```bash
# Full database backup
pg_dump -U postgres -h localhost mydb > backup.sql

# Compressed backup
pg_dump -U postgres -h localhost -Fc mydb > backup.dump
# -Fc = custom format (compressed, faster restore)

# Specific table backup
pg_dump -U postgres -t orders -t order_items mydb > orders_backup.sql

# Schema only (no data)
pg_dump -U postgres --schema-only mydb > schema.sql

# Data only (no schema)
pg_dump -U postgres --data-only mydb > data.sql

# Exclude specific table
pg_dump -U postgres -T audit_logs mydb > backup_no_audit.sql

# Parallel backup (faster for large DBs)
pg_dump -U postgres -Fd -j 4 mydb -f /backup/mydb_dir/
# -Fd = directory format, -j 4 = 4 parallel workers

# Remote database backup
pg_dump -U postgres -h production.db.internal \
    -p 5432 mydb > backup.sql

# With password (use .pgpass file instead!)
# ~/.pgpass: hostname:port:database:username:password
echo "production.db.internal:5432:mydb:postgres:password" > ~/.pgpass
chmod 600 ~/.pgpass
```

### Restore with pg_restore

```bash
# SQL format restore
psql -U postgres -d mydb < backup.sql

# Custom format restore
pg_restore -U postgres -d mydb backup.dump

# Parallel restore (faster)
pg_restore -U postgres -d mydb -j 4 backup.dump

# Specific table restore
pg_restore -U postgres -d mydb -t orders backup.dump

# Create database and restore
createdb -U postgres newdb
pg_restore -U postgres -d newdb backup.dump

# Restore with verbose output
pg_restore -U postgres -d mydb --verbose backup.dump
```

### Automated Backup Script

```bash
#!/bin/bash
# /usr/local/bin/backup_postgres.sh

DB_NAME="myapp_db"
DB_USER="postgres"
DB_HOST="localhost"
BACKUP_DIR="/var/backups/postgresql"
S3_BUCKET="s3://my-company-backups/postgresql"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/${DB_NAME}_${DATE}.dump"
RETENTION_DAYS=30

# Create backup directory
mkdir -p ${BACKUP_DIR}

echo "Starting backup: ${DATE}"

# Backup
pg_dump -U ${DB_USER} -h ${DB_HOST} -Fc ${DB_NAME} > ${BACKUP_FILE}

if [ $? -eq 0 ]; then
    echo "Backup successful: ${BACKUP_FILE}"
    
    # Compress
    gzip ${BACKUP_FILE}
    
    # Upload to S3
    aws s3 cp ${BACKUP_FILE}.gz ${S3_BUCKET}/
    
    if [ $? -eq 0 ]; then
        echo "Uploaded to S3 successfully"
        rm ${BACKUP_FILE}.gz  # Local copy delete (S3 এ আছে)
    fi
    
    # Delete old local backups
    find ${BACKUP_DIR} -name "*.dump.gz" -mtime +${RETENTION_DAYS} -delete
    
else
    echo "Backup FAILED!" >&2
    # Alert পাঠাও (PagerDuty/Slack)
    curl -X POST https://hooks.slack.com/... \
        -d '{"text":"🚨 Database backup FAILED!"}'
    exit 1
fi

echo "Backup completed: $(date)"
```

```python
# Django management command হিসেবে
# management/commands/backup_database.py
from django.core.management.base import BaseCommand
import subprocess
import boto3
from datetime import datetime

class Command(BaseCommand):
    help = 'Backup PostgreSQL database to S3'

    def handle(self, *args, **options):
        timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
        filename  = f'backup_{timestamp}.dump'

        # pg_dump চালাও
        result = subprocess.run([
            'pg_dump',
            '-Fc',
            '-h', settings.DATABASES['default']['HOST'],
            '-U', settings.DATABASES['default']['USER'],
            settings.DATABASES['default']['NAME'],
            '-f', f'/tmp/{filename}'
        ], capture_output=True)

        if result.returncode != 0:
            self.stderr.write(f'Backup failed: {result.stderr}')
            return

        # S3 তে upload করো
        s3 = boto3.client('s3')
        s3.upload_file(f'/tmp/{filename}', 'my-backups', filename)
        
        self.stdout.write(f'Backup completed: {filename}')
```

---

## 14.4 pg_basebackup — Physical Backup

```bash
# Physical backup (entire data directory)
pg_basebackup -U replicator -h localhost \
    -D /var/backups/base_backup \
    -Ft -z -P
# -Ft = tar format
# -z  = gzip compression
# -P  = progress

# WAL streaming সহ (backup চলাকালীন changes include করো)
pg_basebackup -U replicator -h localhost \
    -D /var/backups/base_backup \
    -Ft -z -P -Xs
# -Xs = stream WAL during backup

# Replica তৈরিতেও ব্যবহার হয়
pg_basebackup -U replicator -h primary.db.internal \
    -D /var/lib/postgresql/14/main \
    -P -Xs --checkpoint=fast
```

---

## 14.5 PITR — Point-in-Time Recovery

### কী এবং কেন সবচেয়ে গুরুত্বপূর্ণ?

```
PITR = যেকোনো সময়ের state এ ফিরে যাওয়া।

Example:
  আজ দুপুর 2:30 PM এ developer ভুলে চালায়:
  DELETE FROM orders;  -- WHERE clause ভুলে গেছে!

  PITR দিয়ে 2:29 PM এর state restore করা যাবে।
  শুধু 1 minute এর data হারাবে।

  Full backup এ: শেষ backup এর পর সব data হারাবে (hours/days!)
```

### WAL Archiving Setup

```bash
# postgresql.conf
wal_level = replica
archive_mode = on
archive_command = 'test ! -f /archive/%f && cp %p /archive/%f'
# %p = WAL file path
# %f = WAL file name

# S3 তে archive করো (production recommended)
archive_command = 'aws s3 cp %p s3://my-wal-archive/wal/%f'

# অথবা WAL-G ব্যবহার করো (best tool for WAL archiving)
archive_command = 'wal-g wal-push %p'
restore_command = 'wal-g wal-fetch %f %p'
```

### WAL-G (Best Practice Tool)

```bash
# WAL-G — production grade backup tool
# pip install wal-g অথবা binary download

# Environment variables
export WALG_S3_PREFIX=s3://my-backups/walg
export AWS_REGION=ap-southeast-1
export PGHOST=localhost
export PGUSER=postgres

# Full backup
wal-g backup-push /var/lib/postgresql/14/main

# List backups
wal-g backup-list

# Output:
# name                          last_modified        wal_segment_backup_start
# base_000000010000000000000012 2024-06-01T02:00:00Z 000000010000000000000012
# base_000000010000000000000034 2024-06-08T02:00:00Z 000000010000000000000034

# PITR restore
# Step 1: postgresql.conf এ restore_command set করো
restore_command = 'wal-g wal-fetch %f %p'

# Step 2: recovery.signal file তৈরি করো
touch /var/lib/postgresql/14/main/recovery.signal

# Step 3: postgresql.conf এ target time set করো
recovery_target_time = '2024-06-15 14:29:00'
recovery_target_action = 'promote'  # recovery শেষে normal operation শুরু

# Step 4: PostgreSQL restart করো
systemctl restart postgresql

# PostgreSQL automatically WAL replay করবে target time পর্যন্ত
```

### PITR Step-by-Step Example

```bash
# Scenario: 2024-06-15 14:30 তে accidental DELETE

# Step 1: সর্বশেষ base backup থেকে restore করো
mkdir /var/lib/postgresql/14/main_recovery
wal-g backup-fetch /var/lib/postgresql/14/main_recovery LATEST

# Step 2: recovery config
cat >> /var/lib/postgresql/14/main_recovery/postgresql.conf << EOF
restore_command = 'wal-g wal-fetch %f %p'
recovery_target_time = '2024-06-15 14:29:59'
recovery_target_action = 'promote'
EOF

# Step 3: recovery signal
touch /var/lib/postgresql/14/main_recovery/recovery.signal

# Step 4: Start recovery instance
pg_ctl -D /var/lib/postgresql/14/main_recovery start

# PostgreSQL:
# LOG: starting point-in-time recovery to 2024-06-15 14:29:59
# LOG: restored log file "..." from archive
# LOG: redo starts at ...
# LOG: consistent recovery state reached at ...
# LOG: recovery stopping before commit of transaction ..., time 2024-06-15 14:30:01
# LOG: pausing at the end of recovery

# Step 5: Data verify করো
psql -p 5433 -c "SELECT COUNT(*) FROM orders;"  -- data আছে!

# Step 6: Promote অথবা original DB তে import করো
pg_ctl -D /var/lib/postgresql/14/main_recovery promote
```

---

## 14.6 Backup Strategies

### 3-2-1 Rule

```
3-2-1 Backup Rule (Industry Standard):

3 = ৩টা copy রাখো
    - Original production data
    - Local backup
    - Offsite backup

2 = ২ ধরনের media তে
    - Disk (fast restore)
    - Cloud/Tape (durable)

1 = ১টা offsite location এ
    - AWS S3 (different region)
    - Google Cloud Storage
    - Azure Blob Storage

Example:
  Production DB (AWS us-east-1)
  → Local backup (same server or NFS)
  → S3 bucket (us-east-1)
  → S3 bucket (ap-southeast-1) ← different region!
```

### Backup Schedule

```
Production Recommended Schedule:

Daily (2 AM):
  Full backup → S3 (retain 30 days)
  
Continuous:
  WAL archiving → S3 (retain 7 days)
  = PITR possible for last 7 days

Weekly:
  Full backup → S3 Glacier (retain 1 year)
  
Monthly:
  Full backup → S3 Glacier Deep Archive (retain 7 years)
  = Compliance/legal requirement

Cron setup:
0 2 * * *   /usr/local/bin/backup_postgres.sh  # Daily 2 AM
0 2 * * 0   /usr/local/bin/weekly_backup.sh    # Weekly Sunday
0 2 1 * *   /usr/local/bin/monthly_backup.sh   # Monthly 1st
```

### RTO এবং RPO

```
RTO (Recovery Time Objective):
  "কত দ্রুত restore করতে হবে?"
  Banking: 1 hour
  E-commerce: 4 hours
  Internal tools: 24 hours

RPO (Recovery Point Objective):
  "কত পুরনো data নিয়ে চলতে পারবো?"
  Banking: 0 (no data loss)
  E-commerce: 1 hour
  Internal tools: 24 hours

RTO/RPO অনুযায়ী backup strategy choose করো:

RPO = 0:      Synchronous replication + PITR
RPO = minutes: WAL archiving + PITR
RPO = hours:   Hourly full backup
RPO = days:    Daily full backup
```

---

## 14.7 Disaster Recovery (ডিজাস্টার রিকভারি)

### DR Plan

```
Disaster Recovery Plan (DRP):

1. Risk Assessment:
   কী কী disaster হতে পারে?
   - Hardware failure
   - Data center fire
   - Ransomware
   - Human error
   - Network outage

2. Recovery Objectives:
   RTO: 4 hours
   RPO: 1 hour

3. Recovery Procedures:
   Step-by-step runbook।
   কে কী করবে।
   Communication plan।

4. Testing:
   কমপক্ষে quarterly DR drill।
   Restore actually test করো।

5. Documentation:
   Updated runbooks।
   Architecture diagrams।
   Contact list।
```

### Failover Architecture

```
Active-Passive (Warm Standby):

Primary (us-east-1):
  PostgreSQL Primary
  WAL streaming → Replica

Replica (ap-southeast-1):
  PostgreSQL Standby (hot standby)
  Reads possible
  
Failover:
  Primary down → Promote replica → DNS update → App reconnects
  Downtime: 1-5 minutes

Active-Active (Multi-master):
  Both sites accept writes
  Complex conflict resolution
  PostgreSQL BDR (Bi-Directional Replication)
  Mostly for specialized use cases
```

### Automated Failover — Patroni

```yaml
# Patroni — PostgreSQL High Availability
# patroni.yml

scope: postgres-cluster
name: postgres1

restapi:
  listen: 0.0.0.0:8008
  connect_address: 192.168.1.1:8008

etcd:
  hosts: etcd1:2379,etcd2:2379,etcd3:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 30
    maximum_lag_on_failover: 1048576  # 1MB

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 192.168.1.1:5432
  data_dir: /var/lib/postgresql/14/main
  parameters:
    wal_level: replica
    hot_standby: on
    max_wal_senders: 5
    max_replication_slots: 5
```

```bash
# Patroni দিয়ে manual failover
patronictl -c /etc/patroni.yml failover postgres-cluster
# Primary down হলে automatic failover হবে
# New primary elect হবে (raft consensus)

# Cluster status
patronictl -c /etc/patroni.yml list
# Member   Host        Role    State   TL  Lag in MB
# postgres1 192.168.1.1 Leader  running 1   |
# postgres2 192.168.1.2 Replica running 1   0
# postgres3 192.168.1.3 Replica running 1   0
```

---

## 14.8 AWS RDS Backup

```python
# AWS RDS automated backups

# Terraform config
resource "aws_db_instance" "production" {
  identifier        = "myapp-production"
  engine            = "postgres"
  engine_version    = "15.3"
  instance_class    = "db.r6g.xlarge"
  
  # Backup configuration
  backup_retention_period   = 35      # 35 days retention
  backup_window             = "02:00-03:00"  # 2 AM UTC
  maintenance_window        = "sun:03:00-sun:04:00"
  
  # Multi-AZ for HA
  multi_az = true
  
  # Encryption
  storage_encrypted = true
  kms_key_id       = aws_kms_key.rds.arn
  
  # Performance Insights
  performance_insights_enabled = true
  
  # Enhanced monitoring
  monitoring_interval = 60
  monitoring_role_arn = aws_iam_role.rds_monitoring.arn
}

# Read replica
resource "aws_db_instance" "replica" {
  identifier          = "myapp-replica"
  replicate_source_db = aws_db_instance.production.identifier
  instance_class      = "db.r6g.large"
  
  backup_retention_period = 0  # Replica তে backup দরকার নেই
}
```

```python
# Django + RDS — PITR restore (AWS Console/CLI)

# AWS CLI দিয়ে restore
import boto3

rds = boto3.client('rds', region_name='ap-southeast-1')

# Point-in-time restore
response = rds.restore_db_instance_to_point_in_time(
    SourceDBInstanceIdentifier='myapp-production',
    TargetDBInstanceIdentifier='myapp-restored',
    RestoreTime='2024-06-15T14:29:59Z',  # Target time
    DBInstanceClass='db.r6g.xlarge',
    MultiAZ=False,
    PubliclyAccessible=False,
)

print(f"Restoring to: {response['DBInstance']['DBInstanceIdentifier']}")
# এটা নতুন instance তৈরি করবে — production এ traffic দেওয়ার আগে verify করো
```

---

## 14.9 Backup Verification (সবচেয়ে গুরুত্বপূর্ণ!)

### GitLab Incident থেকে শিক্ষা

```
GitLab এর problem ছিল:
  Backup ছিল — কিন্তু test করা হয়নি।
  Restore করতে গিয়ে দেখা গেল backup corrupt।

Rule:
  "An untested backup is not a backup."
  
Backup verification করো:
  1. Weekly restore test করো staging environment এ
  2. Data integrity check করো
  3. Application actually কাজ করে verify করো
  4. Restore time measure করো (RTO match করে?)
```

### Automated Backup Verification

```bash
#!/bin/bash
# verify_backup.sh — Weekly cron job

BACKUP_FILE=$(aws s3 ls s3://my-backups/ --recursive | sort | tail -1 | awk '{print $4}')
VERIFY_DB="backup_verify_$(date +%Y%m%d)"

echo "Verifying backup: ${BACKUP_FILE}"

# S3 থেকে download করো
aws s3 cp s3://my-backups/${BACKUP_FILE} /tmp/latest_backup.dump

# নতুন database এ restore করো
createdb -U postgres ${VERIFY_DB}
pg_restore -U postgres -d ${VERIFY_DB} /tmp/latest_backup.dump

if [ $? -eq 0 ]; then
    # Data integrity check
    PROD_COUNT=$(psql -U postgres -d myapp_db -t -c "SELECT COUNT(*) FROM orders;")
    BACKUP_COUNT=$(psql -U postgres -d ${VERIFY_DB} -t -c "SELECT COUNT(*) FROM orders;")
    
    if [ "${PROD_COUNT}" -eq "${BACKUP_COUNT}" ]; then
        echo "✅ Backup verification PASSED"
        echo "Orders count: ${BACKUP_COUNT}"
    else
        echo "❌ Backup verification FAILED - count mismatch!"
        echo "Production: ${PROD_COUNT}, Backup: ${BACKUP_COUNT}"
        # Alert পাঠাও
    fi
    
    # Cleanup
    dropdb -U postgres ${VERIFY_DB}
    rm /tmp/latest_backup.dump
else
    echo "❌ Restore FAILED!"
    # Critical alert
fi
```

```python
# Django management command: verify_backup
class Command(BaseCommand):
    help = 'Verify latest backup integrity'

    def handle(self, *args, **options):
        # Restore করো temp database তে
        temp_db = f'verify_{timezone.now().strftime("%Y%m%d")}'
        
        try:
            # Restore
            subprocess.run(['createdb', temp_db], check=True)
            subprocess.run([
                'pg_restore', '-d', temp_db, '/tmp/latest.dump'
            ], check=True)
            
            # Integrity checks
            checks = [
                ('orders', 'SELECT COUNT(*) FROM orders'),
                ('users', 'SELECT COUNT(*) FROM users'),
                ('payments', 'SELECT COUNT(*) FROM payments'),
            ]
            
            for table, query in checks:
                prod_count   = self._query(settings.DATABASES['default'], query)
                backup_count = self._query({'NAME': temp_db}, query)
                
                if prod_count != backup_count:
                    self.stderr.write(
                        f'MISMATCH in {table}: prod={prod_count}, backup={backup_count}'
                    )
                    send_alert(f'Backup verification failed for {table}')
                    return
            
            self.stdout.write('✅ Backup verification PASSED')
            
        finally:
            subprocess.run(['dropdb', temp_db])
```

---

## 14.10 Replication Recovery

### Primary Failure Scenario

```bash
# Scenario: Primary server down

# Step 1: Replica এর lag check করো
psql -h replica -c "SELECT now() - pg_last_xact_replay_timestamp() AS lag;"

# Step 2: Promote replica to primary
pg_ctl -D /var/lib/postgresql/14/main promote
# অথবা
touch /tmp/postgresql.trigger  # trigger file method

# Patroni দিয়ে (automatic):
# Primary down হলে Patroni automatically promote করে

# Step 3: DNS update করো (old primary → new primary)
# AWS Route53
aws route53 change-resource-record-sets \
    --hosted-zone-id XXXXX \
    --change-batch '{
        "Changes": [{
            "Action": "UPSERT",
            "ResourceRecordSet": {
                "Name": "db.myapp.com",
                "Type": "CNAME",
                "TTL": 60,
                "ResourceRecords": [{"Value": "new-primary.internal"}]
            }
        }]
    }'

# Step 4: App reconnect
# Connection pool flush করো (pgBouncer reload)
pgbouncer -R /etc/pgbouncer/pgbouncer.ini
```

### Old Primary Recovery (Rejoin as Replica)

```bash
# Old primary fixed হলে replica হিসেবে rejoin করো

# Step 1: Data sync (pg_rewind)
pg_rewind \
    --target-pgdata=/var/lib/postgresql/14/main \
    --source-server="host=new-primary user=replicator"

# Step 2: standby.signal তৈরি করো
touch /var/lib/postgresql/14/main/standby.signal

# Step 3: primary_conninfo set করো
cat >> /var/lib/postgresql/14/main/postgresql.conf << EOF
primary_conninfo = 'host=new-primary port=5432 user=replicator'
EOF

# Step 4: Start
pg_ctl -D /var/lib/postgresql/14/main start
# Automatically replicate করবে new primary থেকে
```

---

## 14.11 Django Migration Safety

### Safe Migration Practices

```python
# ❌ Dangerous migration — production এ table lock হবে
class Migration(migrations.Migration):
    operations = [
        migrations.AddField(
            model_name='order',
            field=models.CharField(max_length=100, default='pending'),
        ),
    ]
    # Large table এ এটা lock করবে!

# ✅ Safe approach — zero downtime migration

# Step 1: Nullable column যোগ করো
class Migration(migrations.Migration):
    operations = [
        migrations.AddField(
            model_name='order',
            field=models.CharField(max_length=100, null=True),
        ),
    ]

# Step 2: Backfill data (separate migration বা management command)
class Migration(migrations.Migration):
    atomic = False  # Large table এ non-atomic

    def forwards(self, apps, schema_editor):
        Order = apps.get_model('orders', 'Order')
        Order.objects.filter(new_field=None).update(new_field='default_value')

# Step 3: NOT NULL constraint যোগ করো (after backfill)
class Migration(migrations.Migration):
    operations = [
        migrations.AlterField(
            model_name='order',
            field=models.CharField(max_length=100, default='pending'),
        ),
    ]
```

### Pre-Migration Backup

```python
# settings.py অথবা custom management command
class Command(BaseCommand):
    help = 'Backup before migration'

    def handle(self, *args, **options):
        timestamp = timezone.now().strftime('%Y%m%d_%H%M%S')
        
        # Migration এর আগে backup নাও
        subprocess.run([
            'pg_dump', '-Fc',
            '-f', f'/tmp/pre_migration_{timestamp}.dump',
            settings.DATABASES['default']['NAME']
        ], check=True)
        
        self.stdout.write(f'Pre-migration backup: pre_migration_{timestamp}.dump')
        self.stdout.write('Now run: python manage.py migrate')
```

---

## 14.12 Common Mistakes

### ভুল ১: Backup Test না করা

```
❌ Backup নেওয়া হচ্ছে কিন্তু কখনো restore test করা হয়নি।
→ Crisis moment এ restore fail করবে।
→ Corrupt backup, wrong permissions, missing WAL files।

✅ Weekly automated restore test করো।
✅ Quarterly DR drill করো।
✅ Restore time measure করো।
```

### ভুল ২: Same Server এ Backup

```
❌ Production server এ backup রাখা।
→ Server crash হলে backup ও যাবে।
→ Ransomware attack এ backup ও encrypt হবে।

✅ Offsite backup — S3, different AZ, different region।
✅ Immutable backup — S3 Object Lock (ransomware protection)।
```

### ভুল ৩: Backup Monitoring না করা

```python
# ❌ Backup silently fail করছে কিনো জানো না

# ✅ Backup success/failure monitor করো
import boto3

def check_recent_backup():
    s3 = boto3.client('s3')
    
    response = s3.list_objects_v2(
        Bucket='my-backups',
        Prefix='postgresql/'
    )
    
    if not response.get('Contents'):
        send_alert('No backups found!')
        return
    
    latest = max(response['Contents'], key=lambda x: x['LastModified'])
    age_hours = (datetime.now(UTC) - latest['LastModified']).seconds / 3600
    
    if age_hours > 25:  # 25 hours — daily backup missed!
        send_alert(f'Latest backup is {age_hours:.1f} hours old!')

# Cron: প্রতিদিন সকালে check করো
```

### ভুল ৪: Encryption ছাড়া Backup

```bash
# ❌ Plain text backup S3 তে
pg_dump mydb > backup.sql
aws s3 cp backup.sql s3://my-backups/

# ✅ Encrypted backup
pg_dump -Fc mydb | \
    gpg --cipher-algo AES256 --batch --yes \
        --passphrase-file /etc/backup-key.txt -c | \
    aws s3 cp - s3://my-backups/backup_$(date +%Y%m%d).dump.gpg

# অথবা S3 Server-side encryption
aws s3 cp backup.dump s3://my-backups/ \
    --server-side-encryption aws:kms \
    --ssekms-key-id arn:aws:kms:...
```

---

## 14.13 Production Best Practices

```
BACKUP:
1.  3-2-1 rule follow করো
2.  Daily full backup + continuous WAL archiving
3.  Backup encrypt করো (GPG / S3 KMS)
4.  Offsite backup (different region S3)
5.  Backup monitoring — daily alert if backup missing
6.  S3 Object Lock — immutable backup (ransomware protection)
7.  Retention policy — 30 days daily, 1 year monthly

RESTORE/VERIFICATION:
8.  Weekly automated restore test করো
9.  Quarterly DR drill করো
10. RTO/RPO document করো এবং test করো
11. Restore runbook maintain করো

HIGH AVAILABILITY:
12. Patroni দিয়ে automatic failover setup করো
13. Hot standby replica minimum 1টা রাখো
14. Replication lag monitor করো
15. pg_rewind দিয়ে failed primary rejoin করো

MIGRATION SAFETY:
16. Migration এর আগে backup নাও
17. Large table এ zero-downtime migration করো
18. Staging এ আগে test করো
19. Rollback plan রাখো সবসময়

TOOLS:
  pg_dump/pg_restore  → Logical backup
  pg_basebackup       → Physical backup
  WAL-G               → WAL archiving + PITR (recommended)
  Patroni             → HA + automatic failover
  Barman              → Backup manager for PostgreSQL
```

---

## 14.14 Senior Engineer Insights

> **War Story — GitLab (2017):**
> Lead engineer production database DELETE করেন accidentally।
> 5 different backup methods ছিল — কিন্তু কেউ কাজ করেনি:
> - Regular backup: replication lag এ sync হচ্ছিল না
> - pg_dump: wrong server তে চলছিল
> - Snapshots: কিছুক্ষণ আগে disabled
> - S3 backup: setup misconfigured
> - Replica: delete replicate হয়ে গিয়েছিল
>
> Lesson: শুধু backup নেওয়া যথেষ্ট নয়। প্রতিটা backup path test করো।

> **Amazon এর Rule:**
> "Hope is not a strategy. Test your backups."
> Amazon এর teams quarterly "Game Day" করে — intentionally failures inject করে।
> Backup restore করে দেখে আসলে কাজ করে কিনা।

> **Design Decision:**
> WAL-G + S3 + Patroni = best production PostgreSQL HA setup।
> RDS ব্যবহার করলে: Multi-AZ + automated backup + PITR automatically হয়।
> Self-managed PostgreSQL তে এই সব manually setup করতে হবে।

---

## 14.15 Interview Questions

### Beginner Level

**Q1: pg_dump এবং pg_basebackup এর পার্থক্য কী?**
> pg_dump: Logical backup — SQL statements export করে। Portable, table-level restore possible। Large DB তে slow।
> pg_basebackup: Physical backup — data directory copy করে। Faster, PITR possible। Platform-specific।

**Q2: RPO এবং RTO কী?**
> RPO (Recovery Point Objective): কত পুরনো data নিয়ে চলতে পারবো? কতটুকু data হারানো acceptable?
> RTO (Recovery Time Objective): কত দ্রুত system restore করতে হবে? কতক্ষণ downtime acceptable?

**Q3: PITR কী এবং কেন দরকার?**
> Point-in-Time Recovery — যেকোনো specific সময়ের database state এ ফিরে যাওয়া। WAL archiving দিয়ে possible। Accidental DELETE/UPDATE এর পর exact সময়ে restore করা যায়। Full backup থেকে restore করার চেয়ে অনেক বেশি precise।

---

### Intermediate Level

**Q4: 3-2-1 backup rule কী?**
> 3 copies of data, 2 different storage media, 1 offsite location। Production data + local backup + S3 (different region)। এটা hardware failure, ransomware, natural disaster সব থেকে protect করে।

**Q5: Backup কেন test করতে হবে?**
> GitLab incident — backup ছিল কিন্তু restore করা যায়নি (corrupt/misconfigured)। Untested backup = no backup। Weekly restore test করো, RTO measure করো, data integrity check করো।

**Q6: Zero-downtime migration কীভাবে করবে?**
> Step 1: Nullable column যোগ করো (no lock)। Step 2: Background এ data backfill করো। Step 3: NOT NULL constraint যোগ করো। Expand-Contract pattern। Large table এ atomic=False migration।

---

### Senior / FAANG Level

**Q7: Production PostgreSQL HA architecture design করো।**

```
Architecture:

Primary (AZ-1):
  PostgreSQL 15
  WAL streaming → Replica 1 (sync)
  WAL streaming → Replica 2 (async)
  WAL archiving → S3 (continuous)
  Patroni (leader election via etcd)

Replica 1 (AZ-2, Sync):
  Hot standby
  Reads possible
  Patroni follower
  Failover candidate

Replica 2 (AZ-3, Async):
  Read-only queries
  Analytics
  Patroni follower

etcd cluster (3 nodes):
  Patroni consensus
  Leader election

pgBouncer (all nodes):
  Connection pooling
  Failover transparent to app

Backup:
  WAL-G → S3 (daily full + continuous WAL)
  Retention: 35 days
  Cross-region copy: ap-southeast-1 → us-east-1

Monitoring:
  Patroni API → health check
  Replication lag → alert > 1 second
  Backup age → alert > 25 hours
  pg_stat_statements → slow query alert

Failover:
  Primary down → Patroni promotes Replica 1 (sync)
  DNS TTL 60s → app reconnects within 2 minutes
  Old primary → pg_rewind → rejoin as replica
```

**Q8: Accidental DROP TABLE হলে কী করবে?**

```
Immediate Actions:
1. PANIC করো না — stay calm
2. Database কে immediately isolate করো
   (app থেকে connection cut, no more writes)
3. Incident timeline document করো
   ("DROP TABLE ঠিক কখন হয়েছে?")

Recovery Options (best to worst):

Option 1 — PITR (if WAL archiving enabled):
  wal-g backup-fetch to new location
  recovery_target_time = '2024-06-15 14:29:59' (1 min before incident)
  Restore করো, verify করো, promote করো
  Data loss: ~seconds to minutes

Option 2 — Replica (if replica কাছাকাছি):
  Replica promote করো (DROP TABLE replicate হওয়ার আগেই)
  Very small window — act fast!

Option 3 — Latest pg_dump backup:
  Restore করো
  Data loss: last backup থেকে

Option 4 — pg_undrop (community tool):
  Dead tuples থেকে recover করার চেষ্টা
  VACUUM হওয়ার আগেই try করতে হবে
  Success guaranteed না

Post-Recovery:
  Root cause analysis
  Process improvement (peer review for DDL)
  Automated backup test schedule
  pg_audit enable করো (DDL logging)
```

---

## 14.16 Hands-on Exercises

### Exercise 1
```bash
# Backup এবং restore practice:
# 1. pg_dump দিয়ে backup নাও
# 2. নতুন database এ restore করো
# 3. Data integrity verify করো
# 4. Restore time measure করো
```

### Exercise 2
```bash
# WAL archiving setup:
# 1. archive_mode enable করো
# 2. Local directory তে WAL archive করো
# 3. Test transaction চালাও
# 4. PITR restore করো (5 minutes আগে)
```

### Exercise 3
```python
# Automated backup script:
# 1. Daily backup script লেখো
# 2. S3 তে upload করো
# 3. Email/Slack notification
# 4. Backup verification automated করো
```

### Mini Project
```
Complete Backup Strategy:
1. WAL-G দিয়ে WAL archiving setup করো
2. Daily pg_dump backup script
3. S3 এ encrypted upload
4. Weekly automated restore verification
5. Backup age monitoring
6. Django management command: backup + verify
7. Simulate: accidental DELETE → PITR recovery
8. Document RTO/RPO
```

---

## 14.17 Module Summary & Cheat Sheet

```
BACKUP & RECOVERY CHEAT SHEET
================================

BACKUP TYPES:
  pg_dump      → Logical, portable, table-level restore
  pg_basebackup→ Physical, fast, PITR capable
  WAL archiving→ Continuous, PITR (most powerful)
  WAL-G        → Best tool for WAL + base backup

3-2-1 RULE:
  3 copies → Production + Local + S3
  2 media  → Disk + Cloud
  1 offsite → Different region

PITR SETUP:
  postgresql.conf:
    wal_level = replica
    archive_mode = on
    archive_command = 'wal-g wal-push %p'
    restore_command = 'wal-g wal-fetch %f %p'
  
  Recovery:
    recovery_target_time = '2024-06-15 14:29:00'
    recovery_target_action = 'promote'
    touch recovery.signal

RTO / RPO:
  RPO=0:        Sync replication
  RPO=seconds:  WAL archiving + PITR
  RPO=hours:    Hourly backup
  
  RTO target → test করে verify করো

BACKUP SCHEDULE:
  Daily 2 AM:  Full backup → S3 (30 day retention)
  Continuous:  WAL → S3 (7 day retention)
  Weekly:      Full → S3 Glacier (1 year)
  Monthly:     Full → S3 Glacier Deep Archive (7 years)

FAILOVER (Patroni):
  Primary down → Leader election (etcd)
  Sync replica promoted → DNS update
  Old primary → pg_rewind → rejoin as replica

MONITORING:
  Backup age:       alert if > 25 hours
  Replication lag:  alert if > 1 second
  Restore test:     weekly automated
  DR drill:         quarterly

TOOLS:
  WAL-G         → WAL archiving + PITR
  Patroni       → HA + automatic failover
  pgBouncer     → Connection pooling + failover
  Barman        → PostgreSQL backup manager
  pg_rewind     → Rejoin failed primary as replica

KEY RULES:
  "Untested backup = No backup"
  "Test restore weekly"
  "Offsite backup always"
  "Encrypt backups"
  "Monitor backup age"
  "Document RTO/RPO"
  "Quarterly DR drill"
```
