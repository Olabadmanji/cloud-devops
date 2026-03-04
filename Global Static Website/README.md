
# Global Static Website on AWS

A globally distributed static website built on AWS. This project demonstrates high-performance web hosting, automated CI/CD, and multi-region disaster recovery — architected to mirror how real enterprises serve web content at scale.

## Project Overview

This project hosts a static website on AWS infrastructure that eliminates single points of failure, automates the entire deployment process, and serves every visitor from the edge node closest to them — regardless of where they are in the world. The architecture covers four core cloud engineering disciplines: storage, content delivery, DevOps automation, and resilience.

### Architecture

1. **Source:** Developer pushes code to **GitHub**.
2. **CI/CD:** **AWS CodePipeline** detects changes and auto-deploys to the primary S3 bucket.
3. **Origin:** **Amazon S3** (us-east-1) hosts the static files.
4. **Distribution:** **Amazon CloudFront** caches content across 400+ Edge Locations for global low latency.
5. **Security:** **AWS Certificate Manager (ACM)** provides SSL/TLS for HTTPS delivery.
6. **Disaster Recovery:** **S3 Cross-Region Replication (CRR)** automatically syncs data to us-west-2.

---

## Tech Stack

* **Storage:** Amazon S3 (Primary & Replica)
* **CDN:** Amazon CloudFront
* **CI/CD:** AWS CodePipeline & GitHub Webhooks
* **Security:** AWS Certificate Manager (ACM) & IAM

---

## Key Features

* **Global Content Delivery:** Leverages CloudFront to serve users from the edge node closest to them (Lagos, London, or Tokyo), drastically reducing TTFB (Time to First Byte).
* **Automated CI/CD:** Zero-touch deployments. Every `git push` triggers a pipeline that extracts and updates the site in seconds.
* **Automatic HTTPS:** Enforces secure connections via SSL termination at the CloudFront edge. S3 static website hosting only supports HTTP — there is no option to enable HTTPS directly on an S3 endpoint. CloudFront solves this by sitting in front of S3 and handling SSL termination at the edge. This means the encrypted HTTPS connection exists between the visitor's browser and the CloudFront edge node closest to them.
* **Multi-Region Resilience:** Real-time replication to a secondary geographic region (us-west-2) to ensure data durability and disaster recovery. If us-east-1 experienced a full regional outage, the data exists intact in us-west-2 and could serve as the origin for a secondary CloudFront distribution — achieving near-zero RPO (Recovery Point Objective) since replication is real-time.
* **Cache Invalidation:** Implementation of cache invalidation (`/*`) to ensure fresh content delivery post-deployment. CloudFront's job is to cache files at edge locations so it doesn't have to hit S3 on every request. When you deploy new files to S3, CloudFront doesn't automatically know the content changed — it keeps serving the cached version until the cache expires or is manually invalidated. The /* path tells CloudFront to clear every cached file across all edge locations.

In a production pipeline, this invalidation would be automated via a Lambda function triggered at the end of CodePipeline — making the entire deployment fully hands-off after a git push.

---

## 🚀 Deployment Steps

### 1. Storage & Hosting (S3)

* Created a primary bucket in `us-east-1` with **Static Website Hosting** enabled.
* Configured a **Bucket Policy** to allow public `s3:GetObject` access for the website endpoint.
* Enabled **Versioning** to support replication and rollbacks.

### 2. Global Distribution (CloudFront)

* Deployed a CloudFront distribution using the S3 website endpoint as the origin.
* Configured **Viewer Protocol Policy** to "Redirect HTTP to HTTPS."
* Set the **Cache Policy** to `CachingOptimized` to maximize edge performance.

### 3. CI/CD Pipeline (CodePipeline)

* Established a webhook connection between **GitHub** and **AWS**.
* Configured the **Deploy Stage** to target the S3 bucket with the "Extract file before deploy" option enabled.
* Verified that site updates reflect globally within minutes of a code commit.

### 4. Disaster Recovery (CRR)

* Provisioned a replica bucket in `us-west-2`.
* Configured an **S3 Replication Rule** to automatically sync all objects from the primary to the secondary region.

---

## 🔍 Validation & Testing

* **Cache Verification:** Confirmed `x-cache: Hit from cloudfront` in browser developer tools.
* **Security Check:** Verified the SSL padlock icon and 443/HTTPS redirection.
* **Pipeline Success:** Confirmed green status in CodePipeline across Source and Deploy stages.
* **DR Audit:** Verified that files uploaded to `us-east-1` appeared in `us-west-2` with `ReplicationStatus: COMPLETED`.

---

## What I Would Add Next in Production

* **Automated CloudFront Invalidations** — Add a Lambda function as a final CodePipeline stage to automatically trigger a /* invalidation after every successful deployment, making the pipeline completely hands-off end to end.

* **Custom Domain via Route 53** — Connect a custom domain using Route 53 alias A records pointing to the CloudFront distribution, paired with an ACM certificate covering both the root domain and www subdomain.

* **Route 53 Failover Routing** — Add health checks on the primary CloudFront distribution and a failover routing policy that automatically redirects DNS to a secondary CloudFront distribution backed by the us-west-2 replica bucket if the primary becomes unhealthy — achieving a fully automated DR failover.

* **CloudWatch Alarms** — Set up metric alarms on CloudFront 4xx and 5xx error rates to get notified immediately of any content delivery issues.

* **AWS WAF** — Attach a Web Application Firewall to the CloudFront distribution for rate limiting, IP blocking, and DDoS protection at the edge.