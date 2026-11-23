# ByteHunter Portfolio: Static Website on AWS (S3 + CloudFront + Route 53)

A personal portfolio website hosted on AWS using a secure, low-cost, serverless architecture. The site is served over HTTPS on a custom domain (`bytehunter.online`) through a CloudFront CDN, with files stored in a **private** S3 bucket.

**Live site:** https://bytehunter.online

---

## Architecture

```
                    +------------------+
   Visitor  ----->  |    Route 53      |   DNS (A / Alias records)
                    +--------+---------+
                             |
                             v
                    +------------------+
                    |   CloudFront     |   CDN + HTTPS (ACM certificate)
                    |   (OAC enabled)  |
                    +--------+---------+
                             |  signed requests (OAC)
                             v
                    +------------------+
                    |  S3 Bucket       |   Private, Block Public Access ON
                    |  (static files)  |
                    +------------------+
```

## Tech Stack

| Component | Purpose |
|---|---|
| **Amazon S3** | Stores the static website files (HTML, CSS, JS, images) |
| **Amazon CloudFront** | Global CDN, HTTPS termination, caching |
| **Origin Access Control (OAC)** | Lets only CloudFront read from the private bucket |
| **AWS Certificate Manager (ACM)** | Free SSL/TLS certificate (issued in `us-east-1`) |
| **Amazon Route 53** | DNS hosting for the custom domain |
| **Domain registrar (Hostinger)** | Domain registration, nameservers pointed to Route 53 |

## Features

- HTTPS everywhere, with HTTP redirected to HTTPS
- Custom domain with both root (`bytehunter.online`) and `www` support
- Private S3 bucket (no public access)
- Global caching through CloudFront edge locations
- Very low running cost (roughly $0.50 to $1 per month)

---

## Deployment Guide

### 1. Prepare the site
Unzip the portfolio files. Make sure `index.html` sits at the **root** of the folder (not inside an extra subfolder).

### 2. Create the S3 bucket
- Create a bucket (for example `my-portfolio-site`).
- Keep **Block all public access = ON**.
- Upload the **contents** of the portfolio folder (not the zip file).

### 3. Move DNS to Route 53
- Create a public hosted zone for the domain in Route 53.
- Copy the 4 NS records from Route 53.
- In the registrar (Hostinger), switch to **Custom nameservers** and paste the 4 AWS nameservers.
- Verify propagation using a DNS checker (NS lookup should show `awsdns` servers).

### 4. Request an SSL certificate
- In **ACM, region `us-east-1`**, request a certificate for `bytehunter.online` and `www.bytehunter.online`.
- Validate with DNS (CNAME records are added to Route 53 automatically).
- Wait until the status is **Issued**.

### 5. Create the CloudFront distribution
- Origin: the S3 bucket (regular bucket endpoint, **not** the website endpoint).
- Origin access: **Origin Access Control (OAC)**.
- Viewer protocol policy: **Redirect HTTP to HTTPS**.
- Alternate domain names (CNAMEs): `bytehunter.online`, `www.bytehunter.online`.
- Custom SSL certificate: the ACM certificate covering both names.
- Default root object: `index.html`.

### 6. Add the bucket policy
Allow only the CloudFront distribution to read objects:

```json
{
  "Version": "2012-10-17",
  "Statement": {
    "Sid": "AllowCloudFrontServicePrincipalReadOnly",
    "Effect": "Allow",
    "Principal": { "Service": "cloudfront.amazonaws.com" },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::<YOUR_BUCKET_NAME>/*",
    "Condition": {
      "StringEquals": {
        "AWS:SourceArn": "arn:aws:cloudfront::<ACCOUNT_ID>:distribution/<DISTRIBUTION_ID>"
      }
    }
  }
}
```

### 7. Point the domain to CloudFront
In the Route 53 hosted zone, create two **A (Alias)** records targeting the CloudFront distribution:
- Name: *(empty)* for the root domain
- Name: `www`

---

## Updating the Website

1. Upload the new files to the S3 bucket (overwrite existing files).
2. Invalidate the CloudFront cache:
   - CloudFront, then **Invalidations**, then **Create invalidation**, path `/*`

## Troubleshooting Notes (Issues I Ran Into)

| Problem | Cause | Fix |
|---|---|---|
| Certificate stuck on "Unable to validate" | Domain DNS was still served by the registrar, not Route 53 | Changed nameservers at the registrar to the Route 53 NS values |
| "Certificate doesn't cover the alternate domain name (CNAME)" | Distribution had an older certificate attached | Selected the ACM certificate that includes both root and `www` |
| Domain already attached to another distribution | Old distribution from a previous project used the same domain | Edited that distribution's origin to the new bucket |
| `AccessDenied` XML error | Missing or wrong bucket policy, missing default root object, or cached error | Added bucket policy, set `index.html` as default root object, created an invalidation |
| `DNS_PROBE_FINISHED_NXDOMAIN` | No A (Alias) records in Route 53 | Created A (Alias) records for root and `www` |

## Security

- S3 bucket is private; access is limited to CloudFront through OAC and a scoped bucket policy
- HTTPS enforced with a free ACM certificate
- AWS Shield Standard (DDoS protection) is included with CloudFront
- Recommended: enable MFA on the AWS root account, use an IAM user for daily work, enable 2FA and domain lock at the registrar, and set up a billing budget alert

## Estimated Cost

| Service | Approx. monthly cost |
|---|---|
| Route 53 hosted zone | ~$0.50 |
| S3 storage and requests | ~$0.00 |
| CloudFront | ~$0.00 (within free tier for small traffic) |
| ACM certificate | Free |
| **Total** | **~$0.50 to $1** |

## Cleanup

To avoid charges when the site is no longer needed:
1. Disable and delete the CloudFront distribution
2. Empty and delete the S3 bucket
3. Delete the Route 53 hosted zone
4. Delete unused ACM certificates (optional, they are free)

## License

This project is open source and available under the [MIT License](LICENSE).
