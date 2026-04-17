
# Practical 05 — Deploying a Static Website on the Cloud

---

## 📌 Objective

Host a static website using cloud object storage services across AWS S3, Azure Blob Storage, and/or GCP Cloud Storage. Configure permissions and enable public access.

---

## 🧠 Conceptual Background (Know-How)

### What is Static Website Hosting?

A **static website** consists of files (HTML, CSS, JavaScript, images) that are served directly to the browser without server-side processing. There is no backend code executing — what you upload is what the browser receives.

**Static vs Dynamic:**
```
Static Website:
  Browser → CDN/Object Storage → HTML/CSS/JS files returned
  ✅ No server required
  ✅ Extremely fast, cheap, scalable
  ✅ Examples: portfolios, documentation, landing pages

Dynamic Website:
  Browser → Web Server → Application Code → Database → HTML generated
  ✅ Personalized content, user auth, real-time data
  ✅ Examples: e-commerce, social media, SaaS apps
```

### Why Use Object Storage for Static Sites?

| Feature | EC2 Web Server | Object Storage |
|---|---|---|
| Scalability | Requires auto scaling | Automatically scales |
| Availability | Depends on instance | 99.99%+ SLA |
| Cost | $8–$15/month min | Pennies/month |
| Management | OS updates, patches | Zero maintenance |
| CDN Integration | Manual setup | Native (CloudFront, Azure CDN) |

### Object Storage Concepts

```
┌─────────────────────────────────────────────────┐
│  Object Storage Structure                       │
│                                                 │
│  Bucket / Container / Bucket                    │
│  (AWS S3)  (Azure)   (GCP)                      │
│     │                                           │
│     ├── index.html                              │
│     ├── styles.css                              │
│     ├── script.js                               │
│     ├── images/                                 │
│     │   ├── logo.png                           │
│     │   └── banner.jpg                         │
│     └── about.html                             │
│                                                 │
│  Each file = an "object" with:                  │
│    • Key (filename/path)                        │
│    • Value (the actual content/bytes)           │
│    • Metadata (content-type, size, etc.)        │
└─────────────────────────────────────────────────┘
```

---

## 🛠️ Option A — AWS S3 Static Website Hosting

### Step 1 — Create an S3 Bucket

1. Go to **AWS Console → S3 → Create bucket**
2. Configure:
   ```
   Bucket name:       my-static-website-2024  (must be globally unique)
   AWS Region:        us-east-1
   Object Ownership:  ACLs disabled (recommended)
   
   Block Public Access settings:
     ❌ Uncheck "Block all public access"
     ✅ Acknowledge the warning checkbox
   
   Versioning:   Disabled (for simplicity)
   Encryption:   SSE-S3 (server-side encryption, default)
   ```
3. Click **Create bucket**
<img width="1901" height="904" alt="image" src="https://github.com/user-attachments/assets/f1653075-0f84-4fab-8072-3c8ccd5649d1" />

### Step 2 — Upload Website Files

**Create a sample website locally:**

`index.html`:
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>My Cloud Website</title>
    <link rel="stylesheet" href="styles.css">
</head>
<body>
    <div class="container">
        <h1>🌩️ My Cloud-Hosted Website</h1>
        <p>This website is hosted on <strong>AWS S3</strong></p>
        <p>No servers needed — just pure cloud storage!</p>
        <a href="about.html">Learn More</a>
    </div>
    <script src="script.js"></script>
</body>
</html>
```

`styles.css`:
```css
body {
    font-family: 'Segoe UI', sans-serif;
    background: linear-gradient(135deg, #667eea, #764ba2);
    min-height: 100vh;
    display: flex;
    align-items: center;
    justify-content: center;
    margin: 0;
}
.container {
    background: white;
    padding: 40px;
    border-radius: 15px;
    text-align: center;
    box-shadow: 0 20px 60px rgba(0,0,0,0.3);
}
h1 { color: #333; }
a { color: #764ba2; text-decoration: none; font-weight: bold; }
```

`error.html`:
```html
<!DOCTYPE html>
<html>
<head><title>404 Not Found</title></head>
<body>
    <h1>404 - Page Not Found</h1>
    <a href="index.html">Go Home</a>
</body>
</html>
```

**Upload via Console:**
```
S3 → Select bucket → Upload → Add files → Select all your files → Upload
```

**Upload via AWS CLI:**
```bash
# Install AWS CLI first (https://aws.amazon.com/cli/)
aws configure   # Enter your Access Key, Secret, Region

# Upload all files
aws s3 sync ./website-folder/ s3://my-static-website-2024/

# Upload a single file
aws s3 cp index.html s3://my-static-website-2024/

# List bucket contents
aws s3 ls s3://my-static-website-2024/
```

### Step 3 — Enable Static Website Hosting

```
S3 → Select bucket → Properties tab → Static website hosting →
  Edit → Enable
  Hosting type: Host a static website
  Index document: index.html
  Error document: error.html
→ Save changes
```

Note the **Bucket website endpoint** — it looks like:
```
http://my-static-website-2024.s3-website-us-east-1.amazonaws.com
```
<img width="1497" height="731" alt="image" src="https://github.com/user-attachments/assets/b65caef4-7eb6-451d-af08-9e7e492581de" />

### Step 4 — Configure Permissions (Bucket Policy)

```
S3 → Select bucket → Permissions tab → Bucket policy → Edit
```

Paste this policy (replace `my-static-website-2024` with your bucket name):
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::my-static-website-2024/*"
        }
    ]
}
```
<img width="1493" height="693" alt="image" src="https://github.com/user-attachments/assets/6ad6f90e-0a41-4fb0-add2-4ad9ab28be9d" />

Click **Save changes**. You should see a **Publicly accessible** badge on your bucket.

### Step 5 — Access Your Website

Open the bucket website endpoint in your browser:
```
http://my-static-website-2024.s3-website-us-east-1.amazonaws.com
```

🎉 Your static website is live!
<img width="1179" height="735" alt="image" src="https://github.com/user-attachments/assets/f1419334-9436-4f18-9611-a145f4613b4f" />

---





## 📊 Platform Comparison

| Feature | AWS S3 | Azure Blob | GCP Cloud Storage |
|---|---|---|---|
| Static Website Support | ✅ Native | ✅ Native | ✅ Via gsutil |
| Custom Domain (HTTPS) | CloudFront | Azure CDN | Cloud CDN |
| Free Tier Storage | 5 GB (12 mo) | 5 GB (12 mo) | 5 GB forever |
| Pricing (first 50TB) | $0.023/GB | $0.018/GB | $0.020/GB |
| CLI Tool | AWS CLI | Azure CLI | gsutil/gcloud |

---

## 🌐 Bonus: Add a CDN (CloudFront for S3)

For production, always put a CDN in front:

```
User → CloudFront (CDN) → S3 Bucket
        (Edge location       (Origin)
         near user)

Benefits:
  ✅ HTTPS support (free ACM certificate)
  ✅ Content cached at ~450 edge locations worldwide
  ✅ Faster load times globally
  ✅ DDoS protection (AWS Shield Standard free)
  ✅ Custom domain support
```

```
AWS Console → CloudFront → Create distribution
  Origin: your S3 static website endpoint
  Viewer Protocol Policy: Redirect HTTP to HTTPS
  Price Class: Use all edge locations
  Default root object: index.html
→ Create distribution (takes ~15 minutes to deploy)
```

---

## ✅ Learning Outcomes

After completing this practical, you should be able to:

- [ ] Create object storage buckets/containers on AWS, Azure, or GCP
- [ ] Upload static website files to cloud storage
- [ ] Configure static website hosting settings
- [ ] Write and apply bucket policies for public access
- [ ] Access a hosted website via cloud-provided URL
- [ ] Explain the difference between static and dynamic websites
- [ ] Describe the benefits of CDN integration

---

## 💰 Cost Awareness

```
AWS S3 (us-east-1):
  Storage:        $0.023/GB/month
  Requests:       $0.0004 per 1,000 GET requests
  Data transfer:  $0.09/GB (outbound to internet)
  
  A small website with 100 MB + 10,000 visitors/month ≈ $0.03/month
  Free tier: 5 GB, 20,000 GET requests, 2,000 PUT requests (first 12 months)
```

---

## 📚 Further Reading

- [AWS S3 Static Website Hosting](https://docs.aws.amazon.com/AmazonS3/latest/userguide/WebsiteHosting.html)
- [Azure Blob Storage Static Website](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blob-static-website)
- [GCP Cloud Storage Static Website](https://cloud.google.com/storage/docs/hosting-static-website)
- [Amazon CloudFront with S3](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/GettingStartedCreateDistribution.html)
