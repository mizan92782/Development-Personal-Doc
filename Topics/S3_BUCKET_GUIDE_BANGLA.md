# AWS S3 Bucket — সম্পূর্ণ গাইড (বাংলা)

## S3 কী?

AWS S3 (Simple Storage Service) হলো Amazon-এর cloud storage সার্ভিস।
সহজ কথায়, এটা একটা **অনলাইন হার্ড ড্রাইভ** যেখানে তুমি ফাইল রাখতে পারো।

```
তোমার সার্ভার  →  ফাইল আপলোড  →  S3 Bucket  →  ইন্টারনেট থেকে অ্যাক্সেস
```

**এই প্রজেক্টে S3 ব্যবহার হয়:**
- User-এর profile picture সংরক্ষণ
- Certificate ফাইল সংরক্ষণ
- যেকোনো user-uploaded ফাইল

---

## S3 এর মূল ধারণাসমূহ

### Bucket
S3-এ ফাইল রাখার জায়গাকে **Bucket** বলে। এটা অনেকটা একটা বড় ফোল্ডারের মতো।

```
Bucket Name: ikonskills-797671034027-ap-southeast-1-an
Region: ap-southeast-1 (Singapore)
```

### Object
Bucket-এর ভেতরে প্রতিটা ফাইলকে **Object** বলে।

```
Object URL:
https://ikonskills-797671034027-ap-southeast-1-an.s3.ap-southeast-1.amazonaws.com/media/tenant_dp/photo.jpg
```

### Key
প্রতিটা Object-এর path/নাম হলো **Key**।

```
Key: media/tenant_dp/photo.jpg
```

---

## Public vs Private Access — পার্থক্য কী?

### Private Access (ডিফল্ট)
```
কেউ URL দিয়ে সরাসরি ফাইল দেখতে পারবে না
শুধু AWS credentials থাকলে অ্যাক্সেস পাবে
```

### Public Access
```
যেকেউ URL দিয়ে ফাইল দেখতে পারবে
Browser-এ URL paste করলেই ছবি দেখা যাবে
```

### এই প্রজেক্টে কোনটা দরকার?

| ফাইল টাইপ | Access | কারণ |
|-----------|--------|------|
| Profile Picture | Public Read | সবাই দেখতে পারবে |
| Certificate | Public Read | শেয়ার করা যাবে |
| Private Documents | Private | শুধু owner দেখবে |

---

## Block Public Access — কী এবং কেন?

AWS S3-এ ডিফল্টভাবে **Block Public Access** চালু থাকে।
এটা একটা safety lock যা সব public access বন্ধ রাখে।

**৪টা সেটিং আছে:**

```
1. Block public ACLs
   → ACL দিয়ে public করা যাবে না

2. Ignore public ACLs
   → আগে থেকে দেওয়া public ACL কাজ করবে না

3. Block public bucket policies
   → Bucket policy দিয়ে public করা যাবে না

4. Restrict public bucket policies
   → Public policy থাকলেও anonymous access বন্ধ
```

**Profile picture public করতে হলে এই ৪টাই OFF করতে হবে।**

---

## Bucket Policy কী?

Bucket Policy হলো একটা **JSON নিয়মের তালিকা** যা বলে দেয় কে কী করতে পারবে।

### Policy-র মূল অংশ:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "নিয়মের নাম",
      "Effect": "Allow বা Deny",
      "Principal": "কে",
      "Action": "কী করতে পারবে",
      "Resource": "কোন ফাইলে"
    }
  ]
}
```

### প্রতিটা অংশের মানে:

| অংশ | মানে | উদাহরণ |
|-----|------|---------|
| `Version` | Policy version | সবসময় `"2012-10-17"` |
| `Sid` | নিয়মের নাম | `"PublicReadGetObject"` |
| `Effect` | Allow বা Deny | `"Allow"` |
| `Principal` | কে অ্যাক্সেস পাবে | `"*"` মানে সবাই |
| `Action` | কী করতে পারবে | `"s3:GetObject"` মানে পড়তে পারবে |
| `Resource` | কোন ফাইলে | `"arn:aws:s3:::bucket-name/media/*"` |

---

## এই প্রজেক্টের Bucket Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::ikonskills-797671034027-ap-southeast-1-an/media/*"
    }
  ]
}
```

**এই Policy কী বলছে:**
```
যেকেউ (Principal: *)
media/ ফোল্ডারের যেকোনো ফাইল (Resource: .../media/*)
শুধু পড়তে পারবে (Action: s3:GetObject)
```

**কী করতে পারবে না:**
```
❌ ফাইল আপলোড করতে পারবে না (s3:PutObject নেই)
❌ ফাইল ডিলিট করতে পারবে না (s3:DeleteObject নেই)
❌ Bucket-এর তালিকা দেখতে পারবে না (s3:ListBucket নেই)
```

---

## S3 Actions — সব ধরনের Permission

```
s3:GetObject      → ফাইল পড়া/দেখা/ডাউনলোড
s3:PutObject      → ফাইল আপলোড
s3:DeleteObject   → ফাইল ডিলিট
s3:ListBucket     → Bucket-এর ফাইল তালিকা দেখা
s3:GetBucketPolicy → Bucket policy দেখা
s3:PutBucketPolicy → Bucket policy পরিবর্তন
```

---

## IAM User Permission — Django-র জন্য

Django যে AWS credentials ব্যবহার করে সেই IAM User-এর এই permission দরকার:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::ikonskills-797671034027-ap-southeast-1-an/*"
    }
  ]
}
```

**মানে:**
```
Django (IAM User) → আপলোড, দেখা, ডিলিট করতে পারবে ✅
সাধারণ User     → শুধু দেখতে পারবে (Bucket Policy থেকে) ✅
হ্যাকার         → কিছুই করতে পারবে না ✅
```

---

## AWS Console-এ Setup করার ধাপ

### ধাপ ১: Block Public Access বন্ধ করো

```
1. AWS S3 Console → Bucket select করো
2. Permissions tab → Block public access → Edit
3. সব ৪টা checkbox uncheck করো
4. Save changes → "confirm" টাইপ করো → Confirm
```

### ধাপ ২: Bucket Policy যোগ করো

```
1. Permissions tab → Bucket policy → Edit
2. নিচের JSON paste করো
3. Save changes
```

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::ikonskills-797671034027-ap-southeast-1-an/media/*"
    }
  ]
}
```

### ধাপ ৩: IAM User তৈরি করো

```
1. AWS IAM Console → Users → Create user
2. Username দাও (যেমন: lifechoice-s3-user)
3. Attach policies directly → Create inline policy
4. উপরের IAM JSON paste করো
5. Access Key তৈরি করো → .env-এ রাখো
```

---

## Django-তে S3 Implementation

### .env ফাইল

```env
USE_S3=True
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
AWS_STORAGE_BUCKET_NAME=ikonskills-797671034027-ap-southeast-1-an
AWS_S3_REGION_NAME=ap-southeast-1
```

### settings.py — S3 Configuration

```python
if USE_S3 and AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY and AWS_STORAGE_BUCKET_NAME:
    AWS_S3_REGION_NAME = config("AWS_S3_REGION_NAME", default="us-east-1")
    AWS_S3_CUSTOM_DOMAIN = f"{AWS_STORAGE_BUCKET_NAME}.s3.{AWS_S3_REGION_NAME}.amazonaws.com"
    AWS_S3_OBJECT_PARAMETERS = {"CacheControl": "max-age=86400"}  # ১ দিন cache
    AWS_DEFAULT_ACL = None          # ACL ব্যবহার করবো না
    AWS_S3_FILE_OVERWRITE = False   # একই নামে ফাইল overwrite হবে না
    AWS_QUERYSTRING_AUTH = False    # URL-এ signature থাকবে না (public URL)
    DEFAULT_FILE_STORAGE = "lifechoice.storage_backends.MediaStorage"
    MEDIA_URL = f"https://{AWS_S3_CUSTOM_DOMAIN}/media/"
```

### storage_backends.py

```python
from storages.backends.s3boto3 import S3Boto3Storage


class MediaStorage(S3Boto3Storage):
    location = 'media'        # S3-এ media/ ফোল্ডারে রাখবে
    file_overwrite = False    # একই নামে overwrite করবে না
```

### AWS_QUERYSTRING_AUTH = False কেন?

```
False → URL হবে:
https://bucket.s3.amazonaws.com/media/photo.jpg
(সবসময় কাজ করে, expire হয় না)

True → URL হবে:
https://bucket.s3.amazonaws.com/media/photo.jpg?X-Amz-Signature=abc&X-Amz-Expires=3600
(১ ঘণ্টা পর expire হয়ে যায়, ছবি দেখা যাবে না)
```

---

## ফাইল আপলোড ও ডিলিট — Django Model

```python
# Model-এ ImageField ব্যবহার করলে Django স্বয়ংক্রিয়ভাবে S3-এ আপলোড করে
class UserProfile(models.Model):
    dp_image = models.ImageField(upload_to='tenant_dp/', blank=True, null=True)
    # আপলোড হবে → s3://bucket/media/tenant_dp/filename.jpg
```

**আপলোড হলে URL হবে:**
```
https://ikonskills-797671034027-ap-southeast-1-an.s3.ap-southeast-1.amazonaws.com/media/tenant_dp/photo.jpg
```

---

## Public vs Private ফাইল — কোড দিয়ে নিয়ন্ত্রণ

### Public ফাইল (Profile Picture)

```python
# storage_backends.py
class MediaStorage(S3Boto3Storage):
    location = 'media'
    file_overwrite = False
    # AWS_QUERYSTRING_AUTH = False → public URL
```

### Private ফাইল (যদি দরকার হয়)

```python
# storage_backends.py
class PrivateMediaStorage(S3Boto3Storage):
    location = 'private'
    file_overwrite = False
    default_acl = 'private'
    custom_domain = False       # presigned URL ব্যবহার করবে
    querystring_auth = True     # URL expire হবে
    querystring_expire = 3600   # ১ ঘণ্টা পর expire

# Model-এ ব্যবহার:
class SensitiveDocument(models.Model):
    file = models.FileField(storage=PrivateMediaStorage())
```

**Private URL দেখতে হবে:**
```
https://bucket.s3.amazonaws.com/private/doc.pdf?X-Amz-Expires=3600&X-Amz-Signature=xyz
(১ ঘণ্টা পর এই URL কাজ করবে না)
```

---

## Access Control সারসংক্ষেপ

```
┌─────────────────────────────────────────────────────────┐
│                    S3 Bucket                            │
│                                                         │
│  media/ ফোল্ডার                                         │
│  ├── Bucket Policy → সবাই GetObject করতে পারবে         │
│  ├── profile pics → public URL ✅                       │
│  └── certificates → public URL ✅                      │
│                                                         │
│  private/ ফোল্ডার (যদি থাকে)                           │
│  ├── Bucket Policy → কোনো public access নেই            │
│  └── শুধু presigned URL দিয়ে অ্যাক্সেস                 │
│                                                         │
│  Django (IAM User)                                      │
│  └── PutObject + GetObject + DeleteObject ✅            │
└─────────────────────────────────────────────────────────┘
```

---

## সমস্যা ও সমাধান

### সমস্যা ১: AccessDenied Error
```xml
<Error>
  <Code>AccessDenied</Code>
  <Message>Access Denied</Message>
</Error>
```
**কারণ:** Block Public Access চালু আছে বা Bucket Policy নেই
**সমাধান:** Block Public Access বন্ধ করো + Bucket Policy যোগ করো

### সমস্যা ২: URL Expire হয়ে যাচ্ছে
```
URL কিছুক্ষণ পর কাজ করছে না
```
**কারণ:** `AWS_QUERYSTRING_AUTH = True`
**সমাধান:** `AWS_QUERYSTRING_AUTH = False` করো settings.py-তে

### সমস্যা ৩: ফাইল আপলোড হচ্ছে না
```
S3 credentials error
```
**কারণ:** .env-এ `USE_S3=False` বা credentials ভুল
**সমাধান:** .env চেক করো, IAM User-এর PutObject permission আছে কিনা দেখো

---

## এই প্রজেক্টে পরিবর্তিত ফাইলসমূহ

| ফাইল | পরিবর্তন |
|------|----------|
| `lifechoice/settings.py` | S3 configuration, `AWS_QUERYSTRING_AUTH = False` যোগ |
| `lifechoice/storage_backends.py` | `MediaStorage` class |
| `.env` | AWS credentials, `USE_S3=True` |
| AWS Console | Block Public Access বন্ধ, Bucket Policy যোগ |

---

## requirements.txt-এ দরকারি Package

```
boto3==1.34.0
django-storages==1.14.2
```

ইনস্টল করতে:
```bash
pip install boto3 django-storages
```
