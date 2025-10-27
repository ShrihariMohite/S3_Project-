# 🌐 Static Website Deployment — AWS S3 + GitHub Actions + Cloudflare + No-IP

This project demonstrates a **fully automated static site deployment** pipeline:
- Code pushed to GitHub → auto-deployed to **AWS S3**
- Fronted by **Cloudflare Worker** (for HTTPS proxy)
- Accessed via a **free No-IP domain** redirect

---

## 📁 Project Structure

my-static-site/
├── index.html
├── style.css
└── .github/
└── workflows/
└── deploy-to-s3.yml
---------------------------------------------------------------------------------------------------------

## 🧱 Part A — Local Setup

1. Create the folder and files:
   ```bash
   mkdir my-static-site && cd my-static-site
   touch index.html style.css
 -------------------------------------------------------------------------------------------------------------  
🪣 Part B — AWS S3 Setup

Create a new S3 bucket (must be globally unique).

Uncheck “Block all public access”.

Enable Versioning in Properties → Versioning.

Enable Static website hosting → set index document = index.html.

Copy your Website Endpoint (example):http://my-static-site-two.s3-website.ap-south-1.amazonaws.com

Add this bucket policy
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadForGetBucketObjects",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-static-site-two/*"
    }
  ]
}
---------------------------------------------------------------------------------------------------------
Part C — IAM User (GitHub Actions Deployer)

Create an IAM user: github-actions-deployer

Programmatic access only.

Inline policy:

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket", "s3:GetBucketLocation"],
      "Resource": "arn:aws:s3:::my-static-site-two"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:PutObjectAcl", "s3:GetObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::my-static-site-two/*"
    }
  ]
}
------------------------------------------------------------------------------------------------------------

Part D — GitHub Actions Workflow
Create file .github/workflows/deploy-to-s3.yml:
name: Deploy to S3

on:
  push:
    branches: [ "main" ]

permissions:
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Install awscli
        run: python3 -m pip install --upgrade pip awscli

      - name: Sync HTML (short cache)
        run: |
          aws s3 cp index.html s3://${{ secrets.S3_BUCKET }}/index.html --acl public-read \
            --cache-control "max-age=3600, must-revalidate" --content-type "text/html"

      - name: Sync assets (long cache)
        run: |
          aws s3 sync . s3://${{ secrets.S3_BUCKET }}/ \
            --exclude ".git/*" --exclude ".github/*" --exclude "README.md" \
            --acl public-read --delete --exclude "index.html" \
            --cache-control "max-age=31536000, immutable"


            
Add repo secrets:
  
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_REGION
S3_BUCKET

-----------------------------------------------------------------------------------------------------

Part E — Cloudflare Worker (HTTPS Proxy)

Cloudflare Dashboard → Workers & Pages → Create Worker

Replace the default code with:
addEventListener("fetch", event => {
  event.respondWith(handle(event.request));
});

async function handle(request) {
  const incoming = new URL(request.url);
  const target = `http://my-static-site-two.s3-website.ap-south-1.amazonaws.com${incoming.pathname}${incoming.search}`;

  const headers = new Headers(request.headers);
  headers.set("Host", "my-static-site-two.s3-website.ap-south-1.amazonaws.com");

  const origin = await fetch(target, { method: request.method, headers });
  const body = await origin.arrayBuffer();
  return new Response(body, {
    status: origin.status,
    statusText: origin.statusText,
    headers: origin.headers
  });
}

Click Save and Deploy

Copy the Worker URL:https://my-static-proxy.<account>.workers.dev

---------------------------------------------------------------------------------------------------------------------------------

Part F — No-IP Free Domain

Log in to https://my.noip.com

Create Hostname

Type: URL

Host: mystaticsite

Domain: choose a free one (e.g., servesarcasm.com or ddns.net)

Protocol: HTTPS

URL/IP: paste your Worker URL
https://my-static-proxy.<account>.workers.dev

TTL: Auto

Click Create

Visit:http://mystaticsite.ddns.net

-------------------------------------------------------------------------------------------------------------

📅 Report Template

Project: Static Site on S3 + Cloudflare
Date: 2025-10-26
Repo: https://github.com/your-account/my-static-site

Branch: main
Live URL: https://mystaticsite.ddns.net

AWS:

S3 bucket: my-static-site-two

Region: ap-south-1

Versioning: Enabled

Public bucket: Yes (quick flow)

CI/CD:

Workflow: .github/workflows/deploy-to-s3.yml

Secrets: AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_REGION, S3_BUCKET

Deploy method: aws s3 sync

CDN/HTTPS:

Cloudflare Worker proxy: ✅

No-IP redirect: ✅

Cache policy: short for HTML, long for assets
