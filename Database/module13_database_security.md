# MODULE 13: Database Security (ডাটাবেজ সিকিউরিটি)
## বাংলায় সম্পূর্ণ Database Security গাইড — Production Level

> **কেন এই module critical?**
> 2023 সালে বিশ্বের সবচেয়ে বড় data breach গুলোর মূল কারণ ছিল database vulnerability।
> SQL Injection এখনো OWASP Top 10 এ আছে।
> একটা security hole মানে কোম্পানির reputation শেষ, কোটি টাকার জরিমানা।
> Senior engineer হিসেবে security তোমার দায়িত্ব।

---

## 13.1 SQL Injection (SQL ইনজেকশন)

### কী এবং কেন বিপজ্জনক?

```
SQL Injection:
  User input সরাসরি SQL query তে insert হলে
  attacker malicious SQL code inject করতে পারে।

Result:
  - সব data expose হতে পারে
  - Data delete হতে পারে
  - Admin access নেওয়া যায়
  - Database server compromise হতে পারে
```

### Attack Examples

```python
# ❌ Vulnerable code — Raw string concatenation
def get_user(username):
    query = f"SELECT * FROM users WHERE username = '{username}'"
    # username = "admin' OR '1'='1"
    # Query becomes:
    # SELECT * FROM users WHERE username = 'admin' OR '1'='1'
    # সব users return করবে!

# Worse attack:
# username = "'; DROP TABLE users; --"
# Query becomes:
# SELECT * FROM users WHERE username = ''; DROP TABLE users; --'
# users table delete হয়ে যাবে!

# Even worse:
# username = "' UNION SELECT username, password, null FROM admin_users --"
# Admin credentials expose হবে!
```

```sql
-- Attack 1: Authentication Bypass
-- Login form এ:
-- username: admin'--
-- password: anything

SELECT * FROM users
WHERE username = 'admin'--' AND password = 'anything'
-- -- এর পরে সব comment হয়ে যায়!
-- Admin account এ password ছাড়াই login!

-- Attack 2: UNION-based data extraction
-- search: ' UNION SELECT table_name, null FROM information_schema.tables--

SELECT * FROM products WHERE name = ''
UNION SELECT table_name, null FROM information_schema.tables--'
-- সব table names expose!

-- Attack 3: Blind SQL Injection (data না দেখিয়ে true/false দিয়ে data বের করে)
-- search: ' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='admin')='a'--
-- Response টা দেখে বোঝা যায় password এর প্রথম character 'a' কিনা
```

### SQL Injection Prevention

```python
# ✅ Method 1: Parameterized Queries (সবচেয়ে important)
from django.db import connection

def get_user_safe(username):
    with connection.cursor() as cursor:
        cursor.execute(
            "SELECT * FROM users WHERE username = %s",
            [username]  # Parameter — never concatenated!
        )
        return cursor.fetchone()

# ✅ Method 2: Django ORM (automatically parameterized)
def get_user_orm(username):
    return User.objects.filter(username=username).first()
    # Generated: SELECT * FROM users WHERE username = %s
    # ORM automatically escape করে!

# ✅ Method 3: Django ORM filter — সব inputs safe
users = User.objects.filter(
    username=request.GET.get('username'),
    email=request.GET.get('email')
)
# ORM never concatenates — always parameterized

# ❌ NEVER do this even in ORM:
dangerous_query = f"SELECT * FROM users WHERE username = '{username}'"
User.objects.raw(dangerous_query)  # SQL Injection possible!

# ✅ Safe raw query:
User.objects.raw(
    "SELECT * FROM users WHERE username = %s",
    [username]
)
```

### Input Validation Layer

```python
# Extra layer of protection
import re
from django.core.exceptions import ValidationError

def validate_username(username):
    # Only allow alphanumeric and underscore
    if not re.match(r'^[a-zA-Z0-9_]{3,50}$', username):
        raise ValidationError("Invalid username format")
    return username

# DRF Serializer validation
class UserSerializer(serializers.Serializer):
    username = serializers.RegexField(
        regex=r'^[a-zA-Z0-9_]{3,50}$',
        error_messages={'invalid': 'Username must be alphanumeric'}
    )
    email = serializers.EmailField()

# Never trust user input — always validate
```

### WAF (Web Application Firewall)

```
Production SQL Injection Prevention Layers:

1. Application Layer:
   - ORM / Parameterized queries
   - Input validation
   - Output encoding

2. WAF Layer:
   - AWS WAF / Cloudflare WAF
   - SQL Injection pattern detection
   - Rate limiting

3. Database Layer:
   - Least privilege (limited DB user permissions)
   - Stored procedures
   - Audit logging
```

---

## 13.2 ORM Protections (ORM সুরক্ষা)

### Django ORM কীভাবে Protect করে

```python
# Django ORM সবসময় parameterized queries ব্যবহার করে
# কোনো user input সরাসরি SQL এ যায় না

# User input থেকে filter — সম্পূর্ণ safe
search = request.GET.get('search', '')
products = Product.objects.filter(name__icontains=search)

# Generated SQL (psycopg2 level):
# SELECT * FROM products WHERE name ILIKE %s
# Parameters: ['%iphone%']
# 'search' value কখনো SQL এ embed হয় না

# Dangerous patterns যেগুলো ORM protection bypass করে:
# 1. .raw() with string formatting
Product.objects.raw(f"SELECT * FROM products WHERE name='{search}'")  # ❌

# 2. extra() with unsanitized input
Product.objects.extra(where=[f"name='{search}'"])  # ❌ Deprecated + dangerous

# 3. RawSQL with string formatting
from django.db.models.expressions import RawSQL
Product.objects.annotate(x=RawSQL(f"'{search}'", []))  # ❌

# ✅ Safe raw alternatives:
Product.objects.raw("SELECT * FROM products WHERE name=%s", [search])
Product.objects.extra(where=["name=%s"], params=[search])
```

### QuerySet Escaping Details

```python
# Django ORM escaping examples
import django.db.models as models

# String — escaped
User.objects.filter(name="O'Brien")
# SQL: WHERE name = 'O''Brien'  (single quote escaped)

# Integer — type checked
Product.objects.filter(id=int(request.GET.get('id', 0)))

# Boolean — type checked
Product.objects.filter(is_active=True)

# Date — validated
from datetime import date
orders = Order.objects.filter(
    created_at__date=date(2024, 1, 1)
)
```

---

## 13.3 Least Privilege (সর্বনিম্ন অধিকার)

### Database User Permissions

```
Principle of Least Privilege:
  প্রতিটা application user শুধু তার প্রয়োজনীয় permissions পাবে।
  কিছু বেশি না।

Production Application User:
  ✅ SELECT on specific tables
  ✅ INSERT on specific tables
  ✅ UPDATE on specific tables
  ❌ DELETE (যদি soft delete use করো)
  ❌ DROP TABLE
  ❌ CREATE TABLE
  ❌ pg_read_all_data
  ❌ superuser
```

### PostgreSQL User Setup

```sql
-- Production setup: আলাদা users বিভিন্ন purposes এ

-- 1. Application user (web app)
CREATE USER app_user WITH PASSWORD 'strong_password_here';

-- Specific table permissions
GRANT CONNECT ON DATABASE myapp_db TO app_user;
GRANT USAGE ON SCHEMA public TO app_user;

-- Read/Write tables
GRANT SELECT, INSERT, UPDATE ON TABLE
    users, orders, order_items, products, customers
TO app_user;

-- Sequences (for auto-increment)
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO app_user;

-- Read-only tables (reference data)
GRANT SELECT ON TABLE categories, tags, countries TO app_user;

-- 2. Read-only user (reporting, analytics)
CREATE USER readonly_user WITH PASSWORD 'another_strong_password';
GRANT CONNECT ON DATABASE myapp_db TO readonly_user;
GRANT USAGE ON SCHEMA public TO readonly_user;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_user;

-- 3. Migration user (deployment)
CREATE USER migration_user WITH PASSWORD 'migration_password';
GRANT ALL PRIVILEGES ON DATABASE myapp_db TO migration_user;
-- শুধু deployment এর সময় ব্যবহার করো

-- 4. Backup user
CREATE USER backup_user WITH PASSWORD 'backup_password';
GRANT pg_read_all_data TO backup_user;

-- Default privileges (future tables এর জন্য)
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE ON TABLES TO app_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT USAGE, SELECT ON SEQUENCES TO app_user;
```

### Django Multiple Database Users

```python
# settings.py — different users for different operations
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'myapp_db',
        'USER': 'app_user',         # Limited permissions
        'PASSWORD': env('DB_PASSWORD'),
        'HOST': 'db.internal',
    },
}

# Migration এ আলাদা user
# manage.py migrate --database=migration_db

# Django management command override
# অথবা environment variable দিয়ে migration time এ different credentials
```

---

## 13.4 Roles and Permissions (রোল এবং পারমিশন)

### PostgreSQL Role System

```sql
-- Role hierarchy তৈরি করো

-- Base roles
CREATE ROLE reader;
CREATE ROLE writer;
CREATE ROLE admin_role;

-- Permissions assign
GRANT SELECT ON ALL TABLES IN SCHEMA public TO reader;
GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA public TO writer;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO admin_role;

-- Users কে roles assign করো
GRANT reader TO readonly_user;
GRANT writer TO app_user;
GRANT admin_role TO admin_user;

-- Row Level Security (RLS) — per-row access control
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Policy: user শুধু নিজের orders দেখতে পারবে
CREATE POLICY user_orders_policy ON orders
    FOR ALL
    TO app_user
    USING (customer_id = current_setting('app.current_user_id')::INT);

-- Django এ current user set করো
def set_db_user_context(user_id):
    with connection.cursor() as cursor:
        cursor.execute(
            "SET LOCAL app.current_user_id = %s",
            [str(user_id)]
        )

-- Tenant isolation (multi-tenant app)
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON documents
    FOR ALL
    USING (tenant_id = current_setting('app.tenant_id')::INT);
```

### Django Permission System Integration

```python
# Django এর built-in permission system
from django.contrib.auth.models import Permission
from django.contrib.contenttypes.models import ContentType

# Custom permissions
class Order(models.Model):
    class Meta:
        permissions = [
            ('can_refund_order', 'Can refund orders'),
            ('can_view_all_orders', 'Can view all orders'),
            ('can_export_orders', 'Can export order data'),
        ]

# Permission check in views
from django.contrib.auth.decorators import permission_required

@permission_required('orders.can_refund_order', raise_exception=True)
def refund_order(request, order_id):
    ...

# DRF permission
from rest_framework.permissions import BasePermission

class CanRefundOrder(BasePermission):
    def has_permission(self, request, view):
        return request.user.has_perm('orders.can_refund_order')

    def has_object_permission(self, request, view, obj):
        # শুধু নিজের company এর orders refund করতে পারবে
        return obj.company_id == request.user.company_id
```

---

## 13.5 Encryption (এনক্রিপশন)

### Encryption at Rest

```sql
-- PostgreSQL pgcrypto extension
CREATE EXTENSION pgcrypto;

-- Symmetric encryption
-- Store encrypted data
INSERT INTO users (name, ssn_encrypted)
VALUES (
    'Rahim',
    pgp_sym_encrypt('123-45-6789', 'encryption_key_here')
);

-- Decrypt
SELECT
    name,
    pgp_sym_decrypt(ssn_encrypted::bytea, 'encryption_key_here') AS ssn
FROM users;

-- Asymmetric encryption (public/private key)
-- Encrypt with public key
INSERT INTO users (credit_card_encrypted)
VALUES (pgp_pub_encrypt('4111-1111-1111-1111', dearmor('public_key')));

-- Decrypt with private key
SELECT pgp_pub_decrypt(credit_card_encrypted, dearmor('private_key'))
FROM users;
```

### Django Field Encryption

```python
# pip install django-cryptography
from django_cryptography.fields import encrypt

class UserProfile(models.Model):
    user        = models.OneToOneField(User, on_delete=models.CASCADE)
    phone       = encrypt(models.CharField(max_length=20))  # encrypted!
    national_id = encrypt(models.CharField(max_length=20))  # encrypted!
    
    # Non-sensitive data — plain
    city        = models.CharField(max_length=100)
    created_at  = models.DateTimeField(auto_now_add=True)

# Usage — transparent encryption/decryption
profile = UserProfile.objects.create(
    user=user,
    phone='01711234567',        # automatically encrypted in DB
    national_id='1234567890'    # automatically encrypted in DB
)

# Read — automatically decrypted
print(profile.phone)  # '01711234567' (decrypted)

# Database এ কী দেখায়:
# phone: \x7f3a8b2c...  (encrypted bytes)
```

### Password Hashing

```python
# Django built-in — PBKDF2 + SHA256 (default)
from django.contrib.auth.hashers import make_password, check_password

# Hash করো
hashed = make_password('user_password')
# Output: pbkdf2_sha256$600000$salt$hash

# Verify
is_valid = check_password('user_password', hashed)

# Django User model automatically hashes
user = User.objects.create_user(
    username='rahim',
    password='mypassword'  # automatically hashed with PBKDF2
)

# Settings — Argon2 ব্যবহার করো (more secure)
# pip install django[argon2]
PASSWORD_HASHERS = [
    'django.contrib.auth.hashers.Argon2PasswordHasher',  # preferred
    'django.contrib.auth.hashers.PBKDF2PasswordHasher',  # fallback
]

# ❌ কখনো plain text password store করো না
user.password = 'mypassword'  # NEVER!
```

### Encryption in Transit

```python
# settings.py — SSL/TLS database connection
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'OPTIONS': {
            'sslmode': 'require',           # SSL mandatory
            'sslcert': '/path/to/client.crt',
            'sslkey': '/path/to/client.key',
            'sslrootcert': '/path/to/ca.crt',
        },
    }
}

# postgresql.conf
# ssl = on
# ssl_cert_file = 'server.crt'
# ssl_key_file = 'server.key'

# pg_hba.conf — SSL only connections
# hostssl all all 0.0.0.0/0 md5
```

---

## 13.6 Data Masking (ডেটা মাস্কিং)

### কী এবং কেন?

```
Data Masking:
  Sensitive data কে fake কিন্তু realistic data দিয়ে replace করা।
  
Use Cases:
  - Development/Testing environment এ production data ব্যবহার
  - Non-production environment এ real data expose না করা
  - GDPR/compliance requirements
  - Third-party contractor এ data দেওয়া
```

### Static Data Masking

```sql
-- Development database এর জন্য production data mask করো
-- Production থেকে export করার সময়

UPDATE users SET
    email = CONCAT('user_', id, '@masked.com'),
    phone = CONCAT('017', LPAD(id::TEXT, 8, '0')),
    name  = CONCAT('Test User ', id),
    address = 'Masked Address'
WHERE environment = 'production';

-- Credit card masking
UPDATE payment_methods SET
    card_number = CONCAT(
        SUBSTRING(card_number, 1, 4), '****-****-',
        SUBSTRING(card_number, 13, 4)
    );
-- 4111-****-****-1111
```

### Django Dynamic Masking

```python
# API response এ sensitive data mask করো
class UserSerializer(serializers.ModelSerializer):
    phone = serializers.SerializerMethodField()
    email = serializers.SerializerMethodField()

    def get_phone(self, obj):
        # শুধু নিজের phone দেখাবে — অন্যদের masked
        request = self.context.get('request')
        if request and request.user == obj:
            return obj.phone
        return f"***-***-{obj.phone[-4:]}" if obj.phone else None

    def get_email(self, obj):
        request = self.context.get('request')
        if request and request.user == obj:
            return obj.email
        # admin@example.com → a***@example.com
        parts = obj.email.split('@')
        return f"{parts[0][0]}***@{parts[1]}"

    class Meta:
        model = User
        fields = ['id', 'name', 'phone', 'email']
```

### Anonymization for Testing

```python
# management command: anonymize_db.py
from django.core.management.base import BaseCommand
from faker import Faker
import random

fake = Faker('bn_BD')  # Bangla locale

class Command(BaseCommand):
    help = 'Anonymize database for development'

    def handle(self, *args, **options):
        # Users anonymize করো
        for user in User.objects.all():
            user.first_name = fake.first_name()
            user.last_name  = fake.last_name()
            user.email      = fake.email()
            user.phone      = fake.phone_number()

        User.objects.bulk_update(
            User.objects.all(),
            ['first_name', 'last_name', 'email', 'phone'],
            batch_size=1000
        )

        # Payment data anonymize
        PaymentMethod.objects.all().update(
            card_number='4111111111111111',
            card_holder='Test User'
        )

        self.stdout.write("Database anonymized successfully")
```

---

## 13.7 Auditing (অডিটিং)

### কী এবং কেন?

```
Database Auditing:
  কে, কখন, কী করেছে — সব record করা।

Requirements:
  - GDPR: data access log রাখতে হবে
  - PCI DSS: payment data access log
  - HIPAA: medical data access log
  - Financial regulations: transaction audit trail
  - Security: suspicious activity detect
```

### Audit Log Table

```sql
-- Audit log table
CREATE TABLE audit_logs (
    id          BIGSERIAL PRIMARY KEY,
    table_name  VARCHAR(100) NOT NULL,
    operation   VARCHAR(10) NOT NULL,  -- INSERT/UPDATE/DELETE
    record_id   BIGINT,
    old_values  JSONB,
    new_values  JSONB,
    user_id     INT,
    ip_address  INET,
    user_agent  TEXT,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_audit_table_record ON audit_logs(table_name, record_id);
CREATE INDEX idx_audit_user ON audit_logs(user_id);
CREATE INDEX idx_audit_created ON audit_logs(created_at);

-- Trigger for automatic audit
CREATE OR REPLACE FUNCTION audit_trigger_function()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO audit_logs (table_name, operation, record_id, new_values)
        VALUES (TG_TABLE_NAME, 'INSERT', NEW.id, row_to_json(NEW)::jsonb);
        RETURN NEW;
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO audit_logs (table_name, operation, record_id, old_values, new_values)
        VALUES (TG_TABLE_NAME, 'UPDATE', NEW.id,
                row_to_json(OLD)::jsonb, row_to_json(NEW)::jsonb);
        RETURN NEW;
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO audit_logs (table_name, operation, record_id, old_values)
        VALUES (TG_TABLE_NAME, 'DELETE', OLD.id, row_to_json(OLD)::jsonb);
        RETURN OLD;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Apply to sensitive tables
CREATE TRIGGER orders_audit
AFTER INSERT OR UPDATE OR DELETE ON orders
FOR EACH ROW EXECUTE FUNCTION audit_trigger_function();

CREATE TRIGGER payments_audit
AFTER INSERT OR UPDATE OR DELETE ON payments
FOR EACH ROW EXECUTE FUNCTION audit_trigger_function();
```

### Django Audit Mixin

```python
# models.py
from django.utils import timezone

class AuditMixin(models.Model):
    created_at  = models.DateTimeField(auto_now_add=True)
    updated_at  = models.DateTimeField(auto_now=True)
    created_by  = models.ForeignKey(
        'auth.User', null=True,
        on_delete=models.SET_NULL,
        related_name='%(class)s_created'
    )
    updated_by  = models.ForeignKey(
        'auth.User', null=True,
        on_delete=models.SET_NULL,
        related_name='%(class)s_updated'
    )

    class Meta:
        abstract = True

class Order(AuditMixin):
    customer     = models.ForeignKey(Customer, on_delete=models.PROTECT)
    status       = models.CharField(max_length=20)
    total_amount = models.DecimalField(max_digits=12, decimal_places=2)

# Middleware — automatically set created_by/updated_by
import threading
_thread_local = threading.local()

class AuditMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        _thread_local.current_user = request.user
        response = self.get_response(request)
        return response

def get_current_user():
    return getattr(_thread_local, 'current_user', None)
```

### django-simple-history — Full Audit Trail

```python
# pip install django-simple-history
from simple_history.models import HistoricalRecords

class Order(models.Model):
    customer     = models.ForeignKey(Customer, on_delete=models.PROTECT)
    status       = models.CharField(max_length=20)
    total_amount = models.DecimalField(max_digits=12, decimal_places=2)
    history      = HistoricalRecords()  # Magic!

# Usage
order = Order.objects.get(id=1)
order.status = 'shipped'
order.save()

# History দেখো
for history in order.history.all():
    print(f"{history.history_date}: {history.status} by {history.history_user}")

# Specific version restore করো
old_order = order.history.as_of(some_datetime)

# Diff দেখো
new_record = order.history.first()
old_record = order.history.filter(status='pending').first()
delta = new_record.diff_against(old_record)
for change in delta.changes:
    print(f"{change.field}: {change.old} → {change.new}")
```

---

## 13.8 Security Checklist

### Connection Security

```python
# settings.py production security
import environ
env = environ.Env()

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': env('DB_NAME'),
        'USER': env('DB_USER'),         # env var — never hardcode!
        'PASSWORD': env('DB_PASSWORD'), # env var!
        'HOST': env('DB_HOST'),
        'PORT': env('DB_PORT', default='5432'),
        'CONN_MAX_AGE': 60,
        'OPTIONS': {
            'sslmode': 'require',       # Always SSL!
            'connect_timeout': 10,
        },
    }
}

# Never commit credentials
# .env file → .gitignore এ add করো
# Use AWS Secrets Manager / HashiCorp Vault in production
```

### Secrets Management

```python
# AWS Secrets Manager
import boto3
import json

def get_db_credentials():
    client = boto3.client('secretsmanager', region_name='ap-southeast-1')
    secret = client.get_secret_value(SecretId='prod/myapp/db')
    return json.loads(secret['SecretString'])

creds = get_db_credentials()
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'USER': creds['username'],
        'PASSWORD': creds['password'],
        'HOST': creds['host'],
        'NAME': creds['dbname'],
    }
}

# Rotate credentials automatically
# AWS RDS Proxy + Secrets Manager = automatic rotation
```

### PostgreSQL Security Hardening

```sql
-- pg_hba.conf — restrict access
# Local connections: md5 auth
local   all     all                     md5
# SSL only from app servers
hostssl all     app_user  10.0.1.0/24  md5
# Reject everything else
host    all     all       0.0.0.0/0    reject

-- postgresql.conf
listen_addresses = '10.0.1.10'  -- Only app server subnet! Not '0.0.0.0'
log_connections = on
log_disconnections = on
log_failed_connections = on
log_statement = 'ddl'           -- Log DDL statements
```

---

## 13.9 Common Security Mistakes

### ভুল ১: Credentials in Code

```python
# ❌ কখনো করো না!
DATABASES = {
    'default': {
        'PASSWORD': 'mysecretpassword123',  # Git এ চলে যাবে!
    }
}

# ❌ এটাও না
import psycopg2
conn = psycopg2.connect(password="hardcoded_password")

# ✅ Always environment variables
import os
PASSWORD = os.environ.get('DB_PASSWORD')

# ✅ Or python-decouple / django-environ
from decouple import config
PASSWORD = config('DB_PASSWORD')
```

### ভুল ২: Superuser in Production

```sql
-- ❌ app_user = superuser
CREATE USER app_user WITH SUPERUSER PASSWORD '...';

-- ✅ Minimal permissions only
CREATE USER app_user WITH PASSWORD '...';
GRANT SELECT, INSERT, UPDATE ON specific_tables TO app_user;
```

### ভুল ৩: Error Messages Database Info Expose করা

```python
# ❌ Raw database errors user কে দেখাচ্ছো
try:
    result = User.objects.get(id=user_id)
except Exception as e:
    return Response({'error': str(e)})  # DB internals expose!
    # "relation 'users' does not exist" — attacker table name জানলো!

# ✅ Generic error messages
try:
    result = User.objects.get(id=user_id)
except User.DoesNotExist:
    return Response({'error': 'User not found'}, status=404)
except Exception:
    logger.exception(f"Error fetching user {user_id}")
    return Response({'error': 'Internal server error'}, status=500)
```

### ভুল ৪: Unencrypted Backups

```bash
# ❌ Plain text backup
pg_dump mydb > backup.sql  # S3 তে unencrypted!

# ✅ Encrypted backup
pg_dump mydb | gpg --cipher-algo AES256 -c > backup.sql.gpg

# ✅ AWS RDS automated encrypted backups
# RDS encryption at rest enable করো (KMS)
```

---

## 13.10 GDPR Compliance

```python
# Right to be forgotten — user data delete
@transaction.atomic
def delete_user_data(user_id):
    user = User.objects.get(id=user_id)
    
    # Anonymize instead of delete (preserve referential integrity)
    user.email      = f'deleted_{user_id}@deleted.com'
    user.first_name = 'Deleted'
    user.last_name  = 'User'
    user.phone      = None
    user.is_active  = False
    user.save()
    
    # Delete PII from related tables
    UserProfile.objects.filter(user_id=user_id).delete()
    PaymentMethod.objects.filter(user_id=user_id).delete()
    
    # Keep orders (for financial/legal records) but anonymize
    Order.objects.filter(user_id=user_id).update(
        customer_notes='[DELETED]'
    )
    
    # Log the deletion
    AuditLog.objects.create(
        action='USER_DATA_DELETED',
        user_id=user_id,
        performed_by=request.user.id
    )

# Data export (right to access)
def export_user_data(user_id):
    user = User.objects.get(id=user_id)
    data = {
        'profile': UserSerializer(user).data,
        'orders': OrderSerializer(
            Order.objects.filter(user_id=user_id), many=True
        ).data,
        'payment_methods': [
            {'last4': pm.card_number[-4:], 'type': pm.card_type}
            for pm in PaymentMethod.objects.filter(user_id=user_id)
        ]
    }
    return data
```

---

## 13.11 Production Best Practices

```
APPLICATION SECURITY:
1.  সবসময় ORM বা parameterized queries ব্যবহার করো
2.  Raw SQL এ কখনো string concatenation করো না
3.  Input validation সব user inputs এ
4.  Error messages এ database internals expose করো না
5.  Sensitive data response এ mask করো

DATABASE SECURITY:
6.  Least privilege — app user এ minimal permissions
7.  আলাদা users: app, readonly, migration, backup
8.  SSL/TLS সবসময় (sslmode=require)
9.  pg_hba.conf তে IP restriction করো
10. listen_addresses specific IP তে set করো

SECRETS:
11. Credentials কখনো code এ hardcode করো না
12. Environment variables বা Secrets Manager ব্যবহার করো
13. .env file → .gitignore তে add করো
14. Regular credential rotation (AWS Secrets Manager)

AUDITING:
15. Sensitive tables এ audit trigger যোগ করো
16. Access logs enable করো (log_connections)
17. Failed login attempts log করো
18. django-simple-history ব্যবহার করো important models এ

COMPLIANCE:
19. GDPR: right to delete, right to access implement করো
20. PII encryption at rest (pgcrypto / django-cryptography)
21. Backup encryption করো
22. Regular security audit করো
```

---

## 13.12 Senior Engineer Insights

> **War Story — Equifax Breach (2017):**
> 147 million মানুষের personal data চুরি হয়।
> কারণ: Unpatched Apache Struts vulnerability + Database এ least privilege ছিল না।
> একবার app compromise হলে attacker সব database access পেয়ে যায়।
> Lesson: Defense in depth — app security + DB security দুটোই দরকার।

> **LinkedIn Password Breach (2012):**
> 117 million passwords চুরি হয়।
> কারণ: SHA1 hashing (unsalted) ব্যবহার করা হয়েছিল।
> SHA1 rainbow table দিয়ে easily crack করা যায়।
> Lesson: সবসময় bcrypt/Argon2/PBKDF2 ব্যবহার করো salt সহ।

> **Design Decision:**
> Security = Layers। একটা layer ভাঙলে পরের layer আছে।
> 1. Input validation
> 2. ORM (parameterized queries)
> 3. Least privilege DB user
> 4. Network restriction (pg_hba.conf)
> 5. Encryption at rest
> 6. Audit logging
> 7. Monitoring/alerting

---

## 13.13 Interview Questions

### Beginner Level

**Q1: SQL Injection কী এবং কীভাবে prevent করবে?**
> User input সরাসরি SQL query তে concatenate করলে attacker malicious SQL inject করতে পারে। Prevention: ORM ব্যবহার করো (automatically parameterized), raw SQL এ %s placeholder ব্যবহার করো, string concatenation কখনো না।

**Q2: Parameterized query কীভাবে SQL injection prevent করে?**
> Parameterized query তে user input SQL এ embed হয় না — আলাদাভাবে database driver এ পাঠানো হয়। Database input কে data হিসেবে treat করে, code হিসেবে নয়। তাই malicious SQL execute হতে পারে না।

**Q3: Database এ least privilege কেন দরকার?**
> App compromise হলে attacker app user এর permissions এর বেশি কিছু করতে পারবে না। App user শুধু SELECT/INSERT/UPDATE পেলে DROP TABLE করতে পারবে না।

---

### Intermediate Level

**Q4: Row Level Security (RLS) কী এবং কখন ব্যবহার করবে?**
> PostgreSQL এর feature যেখানে একই table এ different users different rows দেখে। Multi-tenant app এ tenant isolation এর জন্য excellent। প্রতিটা query তে WHERE clause manually না দিয়ে database level এ enforce করা যায়।

**Q5: Sensitive data কীভাবে store করবে? (passwords, credit cards, NID)**
> Passwords: never plain text, always bcrypt/Argon2 hash।
> Credit cards: PCI DSS — encrypt + tokenize, raw card number store করো না।
> NID/SSN: pgcrypto বা django-cryptography দিয়ে encrypt।
> Encryption key আলাদা জায়গায় (AWS KMS, HashiCorp Vault)।

**Q6: Audit trail কেন দরকার এবং কীভাবে implement করবে?**
> Compliance (GDPR, PCI DSS), security incident investigation, data integrity verification এর জন্য। Implementation: database trigger (automatic) অথবা django-simple-history (application level)। কে, কখন, কী পরিবর্তন করেছে record রাখো।

---

### Senior / FAANG Level

**Q7: Multi-tenant SaaS application এ data isolation কীভাবে ensure করবে?**

```
Strategy 1 — Separate Databases (strongest isolation):
  Tenant A → Database A
  Tenant B → Database B
  Complete isolation কিন্তু operational overhead বেশি

Strategy 2 — Separate Schemas:
  Tenant A → schema_a.orders
  Tenant B → schema_b.orders
  Good isolation, manageable

Strategy 3 — Shared Tables + RLS (most common):
  All tenants in same table
  tenant_id column + Row Level Security

Implementation:
```
```sql
-- Shared table approach
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation_policy ON orders
    AS PERMISSIVE
    FOR ALL
    TO app_user
    USING (tenant_id = current_setting('app.tenant_id')::INT)
    WITH CHECK (tenant_id = current_setting('app.tenant_id')::INT);
```
```python
# Django middleware — set tenant context
class TenantMiddleware:
    def __call__(self, request):
        if request.user.is_authenticated:
            with connection.cursor() as cursor:
                cursor.execute(
                    "SET LOCAL app.tenant_id = %s",
                    [request.user.tenant_id]
                )
        return self.get_response(request)
```

**Q8: একটা healthcare application এ database security design করো (HIPAA compliance)।**

```
HIPAA Requirements:
  - PHI (Protected Health Information) encrypted at rest
  - Access controls (least privilege)
  - Audit logs (all PHI access)
  - Breach notification
  - Backup encryption

Database Design:
  1. Encryption:
     - pgcrypto for PHI fields (diagnosis, medications)
     - AWS RDS encryption at rest (KMS)
     - SSL/TLS in transit

  2. Access Control:
     - Separate DB users: doctors, nurses, admin, billing
     - Row Level Security: doctors শুধু নিজের patients
     - Column Level Security: billing শুধু financial data

  3. Audit:
     - Trigger on all PHI table access
     - Log: user, timestamp, data accessed, IP
     - Immutable audit log (append-only)

  4. Data Masking:
     - Development/testing: anonymized data
     - API responses: mask non-essential PHI

  5. Key Management:
     - AWS KMS for encryption keys
     - Key rotation every 90 days
     - Different keys for different data classes
```

---

## 13.14 Hands-on Exercises

### Exercise 1
```python
# SQL Injection test করো:
# 1. Vulnerable endpoint তৈরি করো
# 2. Attack করো (safe environment এ)
# 3. ORM দিয়ে fix করো
# 4. sqlmap tool দিয়ে test করো
```

### Exercise 2
```sql
-- PostgreSQL security setup:
-- 1. app_user তৈরি করো (minimal permissions)
-- 2. readonly_user তৈরি করো
-- 3. Row Level Security implement করো orders table এ
-- 4. Audit trigger যোগ করো payments table এ
```

### Exercise 3
```python
# Django security:
# 1. django-simple-history যোগ করো Order model এ
# 2. API response এ phone number mask করো
# 3. Environment variables দিয়ে credentials manage করো
# 4. AuditMiddleware implement করো
```

### Mini Project
```
Secure E-commerce Backend:
1. SQL injection prevention verify করো
2. Least privilege DB user setup করো
3. Sensitive fields encrypt করো (phone, address)
4. Audit log implement করো (orders, payments)
5. Data masking API response এ
6. GDPR: delete user data endpoint
7. Security headers Django settings এ
```

---

## 13.15 Module Summary & Cheat Sheet

```
DATABASE SECURITY CHEAT SHEET
===============================

SQL INJECTION PREVENTION:
  ✅ Django ORM (always parameterized)
  ✅ cursor.execute("SQL %s", [param])
  ✅ Model.objects.raw("SQL %s", [param])
  ❌ f"SQL {user_input}"
  ❌ "SQL " + user_input
  ❌ .raw(f"SQL {variable}")

LEAST PRIVILEGE:
  app_user:       SELECT, INSERT, UPDATE (specific tables)
  readonly_user:  SELECT only
  migration_user: ALL (deployment only)
  backup_user:    pg_read_all_data
  NEVER:          superuser for app

ENCRYPTION:
  Passwords:    Argon2 / bcrypt / PBKDF2 (never plain/MD5/SHA1)
  PII fields:   django-cryptography / pgcrypto
  Backups:      GPG encrypt
  In transit:   sslmode=require
  At rest:      AWS RDS KMS / PostgreSQL TDE

AUDIT:
  DB Triggers:          row_to_json() → audit_logs table
  Django History:       django-simple-history
  Application:          AuditMixin (created_by, updated_by)
  PostgreSQL logs:      log_connections, log_statement='ddl'

ROW LEVEL SECURITY:
  ALTER TABLE t ENABLE ROW LEVEL SECURITY;
  CREATE POLICY name ON t USING (tenant_id = current_tenant());
  SET LOCAL app.tenant_id = X;

SECRETS:
  ❌ Hardcoded credentials
  ✅ Environment variables
  ✅ AWS Secrets Manager
  ✅ HashiCorp Vault
  ✅ .env + .gitignore

GDPR:
  Right to delete: anonymize PII, keep financial records
  Right to access: export all user data
  Consent: log when/how consent given
  Breach: 72hr notification requirement

SECURITY LAYERS:
  1. Input validation (application)
  2. ORM / parameterized queries
  3. Least privilege DB user
  4. Network restriction (pg_hba.conf)
  5. Encryption (at rest + in transit)
  6. Audit logging
  7. Monitoring / alerting
  8. Regular security audits
```
