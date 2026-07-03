# AWS S3 সম্পূর্ণ টিউটোরিয়াল (বাংলা)
> Server Terminal থেকে S3 কনফিগার, ম্যানেজ এবং ব্যবহার করার পূর্ণ গাইড

---

## ১. AWS CLI ইনস্টল করা

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install awscli -y

# Version চেক
aws --version
```

---

## ২. AWS CLI কনফিগার করা

```bash
aws configure
```

এটা রান করলে ৪টা জিনিস চাইবে:

```
AWS Access Key ID: <YOUR_AWS_ACCESS_KEY>
AWS Secret Access Key: <YOUR_AWS_SECRET_KEY>
Default region name: ap-southeast-1
Default output format: json
```

> ⚠️ Access Key এবং Secret Key কখনো public করবেন না।

কনফিগ সঠিক হয়েছে কিনা চেক করুন:

```bash
aws sts get-caller-identity
```

---

## ৩. S3 Bucket দেখা ও তৈরি করা

### সব bucket লিস্ট দেখুন
```bash
aws s3 ls
```

### নতুন bucket তৈরি করুন
```bash
aws s3 mb s3://your-bucket-name --region ap-southeast-1
```

### bucket এর ভেতরের ফাইল দেখুন
```bash
aws s3 ls s3://your-bucket-name/
```

### সব ফাইল recursive দেখুন
```bash
aws s3 ls s3://your-bucket-name/ --recursive
```

### নির্দিষ্ট folder দেখুন
```bash
aws s3 ls s3://your-bucket-name/media/certificate_template/
```

---

## ৪. ফাইল Transfer করা (Local → S3)

### একটা ফাইল upload করুন
```bash
aws s3 cp /home/ubuntu/media/image.png s3://your-bucket-name/media/image.png
```

### পুরো folder upload করুন
```bash
aws s3 cp /home/ubuntu/media/ s3://your-bucket-name/media/ --recursive
```

### S3 থেকে server এ download করুন
```bash
aws s3 cp s3://your-bucket-name/media/image.png /home/ubuntu/media/image.png
```

### পুরো folder download করুন
```bash
aws s3 cp s3://your-bucket-name/media/ /home/ubuntu/media/ --recursive
```

### S3 এর এক folder থেকে আরেক folder এ copy করুন
```bash
aws s3 cp s3://your-bucket-name/media/tenant_dp/ s3://your-bucket-name/media/practitioner_dp/ --recursive
```

### Sync করুন (শুধু নতুন/পরিবর্তিত ফাইল)
```bash
aws s3 sync /home/ubuntu/media/ s3://your-bucket-name/media/
```

---

## ৫. ফাইল Delete করা

### একটা ফাইল delete করুন
```bash
aws s3 rm s3://your-bucket-name/media/old-image.png
```

### পুরো folder delete করুন
```bash
aws s3 rm s3://your-bucket-name/media/tenant_dp/ --recursive
```

---

## ৬. Public Access কনফিগার করা

### Block Public Access বন্ধ করুন (bucket level)
```bash
aws s3api put-public-access-block \
  --bucket your-bucket-name \
  --public-access-block-configuration BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false
```

### Public Access status চেক করুন
```bash
aws s3api get-public-access-block --bucket your-bucket-name
```

---

## ৭. Bucket Policy ম্যানেজ করা

### Public Read Policy যোগ করুন

```bash
cat > /tmp/bucket-policy.json << 'EOF'
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::your-bucket-name/*"
        }
    ]
}
EOF

aws s3api put-bucket-policy --bucket your-bucket-name --policy file:///tmp/bucket-policy.json
```

### বর্তমান Policy দেখুন
```bash
aws s3api get-bucket-policy --bucket your-bucket-name
```

### Policy delete করুন
```bash
aws s3api delete-bucket-policy --bucket your-bucket-name
```

---

## ৮. Django এ S3 কনফিগার করা

### প্রয়োজনীয় package install করুন
```bash
pip install boto3 django-storages
```

### `.env` ফাইলে যোগ করুন
```env
USE_S3=True
AWS_ACCESS_KEY_ID=your-access-key
AWS_SECRET_ACCESS_KEY=your-secret-key
AWS_STORAGE_BUCKET_NAME=your-bucket-name
AWS_S3_REGION_NAME=ap-southeast-1
```

### `storage_backends.py` তৈরি করুন
```python
from storages.backends.s3boto3 import S3Boto3Storage

class MediaStorage(S3Boto3Storage):
    location = 'media'
    file_overwrite = False
```

### `settings.py` তে কনফিগার করুন
```python
if USE_S3:
    DEFAULT_FILE_STORAGE = 'yourproject.storage_backends.MediaStorage'
    MEDIA_URL = f'https://{AWS_STORAGE_BUCKET_NAME}.s3.{AWS_S3_REGION_NAME}.amazonaws.com/media/'
```

---

## ৯. সমস্যা খুঁজে বের করা

### ফাইল S3 এ আছে কিনা চেক করুন
```bash
aws s3 ls s3://your-bucket-name/media/certificate_template/
```

### URL সরাসরি test করুন
```bash
curl -I "https://your-bucket-name.s3.ap-southeast-1.amazonaws.com/media/image.png"
```

- `200 OK` → ফাইল আছে এবং public
- `403 Forbidden` → ফাইল আছে কিন্তু private অথবা policy নেই
- `404 Not Found` → ফাইল নেই

### AWS credentials সঠিক কিনা চেক করুন
```bash
aws sts get-caller-identity
```

### Account level block চেক করুন
```bash
aws s3control get-public-access-block --account-id your-account-id
```

---

## ১০. পুরনো Media ফাইল S3 তে Transfer করা

যখন local storage থেকে S3 তে migrate করবেন:

### Step 1 — server এ ফাইল খুঁজুন
```bash
find /home/ubuntu -name "*.png" -o -name "*.jpg" -o -name "*.jpeg" 2>/dev/null
```

### Step 2 — Docker container এ ফাইল খুঁজুন
```bash
docker exec container_name ls /app/media/
```

### Step 3 — Docker থেকে server এ copy করুন
```bash
docker cp container_name:/app/media/ /home/ubuntu/media/
```

### Step 4 — Server থেকে S3 তে upload করুন
```bash
aws s3 cp /home/ubuntu/media/ s3://your-bucket-name/media/ --recursive
```

### Step 5 — S3 তে সব ফাইল গেছে কিনা verify করুন
```bash
aws s3 ls s3://your-bucket-name/media/ --recursive
```

---

## ১১. S3 Folder Rename করা

S3 তে সরাসরি rename হয় না — copy করে পুরনোটা delete করতে হয়:

```bash
# নতুন নামে copy করুন
aws s3 cp s3://your-bucket-name/media/old-folder/ s3://your-bucket-name/media/new-folder/ --recursive

# পুরনো folder delete করুন
aws s3 rm s3://your-bucket-name/media/old-folder/ --recursive
```

---

## ১২. Bucket Size এবং File Count দেখা

```bash
# Total size দেখুন
aws s3 ls s3://your-bucket-name/ --recursive --human-readable --summarize
```

---

## ১৩. দরকারি Commands চিটশিট

| কাজ | Command |
|-----|---------|
| Bucket list | `aws s3 ls` |
| Folder দেখুন | `aws s3 ls s3://bucket/folder/` |
| File upload | `aws s3 cp file.png s3://bucket/file.png` |
| Folder upload | `aws s3 cp folder/ s3://bucket/folder/ --recursive` |
| File download | `aws s3 cp s3://bucket/file.png ./file.png` |
| File delete | `aws s3 rm s3://bucket/file.png` |
| Folder delete | `aws s3 rm s3://bucket/folder/ --recursive` |
| Sync | `aws s3 sync ./media/ s3://bucket/media/` |
| Policy দেখুন | `aws s3api get-bucket-policy --bucket bucket-name` |
| Public access চেক | `aws s3api get-public-access-block --bucket bucket-name` |
| URL test | `curl -I "https://bucket.s3.region.amazonaws.com/file.png"` |
