> **Note:** Commands in this document use placeholder variables (e.g. `${AWS_ACCOUNT_ID}`). Replace these with your actual values before running.

# Cloudflare DNS Setup

Sentorix uses Cloudflare as the DNS provider for all three domains. Cloudflare is configured in **DNS only mode** (orange cloud disabled / grey cloud enabled) for all records that point to AWS resources.

---

## 1. Domains and Purpose

| Domain | Purpose | Status |
|---|---|---|
| `sentorix.io` | Primary product domain. Hosts the dashboard (`app.sentorix.io`) and gateway API (`api.sentorix.io`). | Active |
| `sentorix.ai` | AI-focused marketing domain. Reserved for future marketing site. | Parked |
| `sentorix.dev` | Developer and documentation domain. Hosts this documentation at `docs.sentorix.dev`. | Active |

---

## 2. DNS Records

### sentorix.io

| Type | Name | Target | Proxy Status | TTL |
|---|---|---|---|---|
| CNAME | `app` | `sentorix-alb-{id}.us-east-1.elb.amazonaws.com` | DNS only (grey) | Auto |
| CNAME | `api` | `sentorix-alb-{id}.us-east-1.elb.amazonaws.com` | DNS only (grey) | Auto |
| CNAME | `www` | `sentorix-alb-{id}.us-east-1.elb.amazonaws.com` | DNS only (grey) | Auto |

> The ALB DNS name (`sentorix-alb-{id}.us-east-1.elb.amazonaws.com`) is visible in the AWS Console under EC2 → Load Balancers → sentorix-alb → DNS name. It is also output by Terraform after `terraform apply`.

### sentorix.dev

| Type | Name | Target | Proxy Status | TTL |
|---|---|---|---|---|
| CNAME | `docs` | _(documentation hosting target)_ | DNS only (grey) | Auto |

### ACM Validation Records (sentorix.io)

ACM certificate validation requires CNAME records in DNS. These are permanent — do not delete them or the certificate will fail to renew.

| Type | Name | Value | Purpose |
|---|---|---|---|
| CNAME | `_acme-challenge.sentorix.io` (or AWS-generated name) | AWS-generated validation value | ACM certificate domain validation |
| CNAME | `_acme-challenge.*.sentorix.io` | AWS-generated validation value | Wildcard validation |

The exact record names and values are shown in the AWS Certificate Manager console under the certificate details. Cloudflare displays these as regular CNAME records once added.

---

## 3. Why Proxy Is Off for ALB Records

Cloudflare offers an "orange cloud" proxy mode that routes all traffic through Cloudflare's network, providing DDoS protection, CDN caching, and additional security features. For Sentorix's ALB records, the proxy is **disabled** (grey cloud). Here is why:

### ACM certificate validation

AWS Certificate Manager validates domain ownership by checking for specific CNAME records in DNS. When Cloudflare's proxy is active, it intercepts DNS queries and may respond with Cloudflare's IP addresses instead of the actual CNAME target. ACM's validation crawler cannot reach the correct target, and validation fails or takes excessively long.

### AWS ALB health checks

The ALB performs health checks against its own backend targets. Some ALB-level functions (target group stickiness, cross-zone load balancing) depend on the ALB receiving the actual client IP. When Cloudflare proxies the connection, the ALB sees Cloudflare's IP address as the source, not the real client IP. This does not break health checks directly, but it does affect:
- IP-based rate limiting in the application
- Accurate geographic logging
- Security group rules that trust specific IP ranges

### TLS termination is already handled by ACM

Cloudflare's proxy provides TLS termination, but ACM already handles this at the ALB level. There is no benefit to double-proxying — the connection is already encrypted with a valid certificate. Adding Cloudflare proxy would add an extra TLS hop with no security benefit.

### When to use the Cloudflare proxy

If Sentorix ever hosts static assets or marketing pages directly on Cloudflare Workers or Pages (without going through the ALB), those records can use the orange cloud proxy for CDN caching benefits.

---

## 4. ACM Certificate

### Certificate Details

| Parameter | Value |
|---|---|
| Certificate ARN | `arn:aws:acm:us-east-1:${AWS_ACCOUNT_ID}:certificate/{id}` |
| Domain | `sentorix.io` |
| Additional names | `*.sentorix.io` |
| Type | Amazon-issued |
| Validation method | DNS |
| Status | Issued |
| Renewal | Automatic (managed by AWS) |

The certificate covers:
- `sentorix.io` (root domain)
- `*.sentorix.io` (all subdomains: `api.sentorix.io`, `app.sentorix.io`, `www.sentorix.io`, etc.)

### How to add validation records in Cloudflare

When the certificate is first requested (or if validation records are missing), follow these steps:

1. Go to the AWS Console → Certificate Manager → select the certificate
2. Expand the domain validation section — you will see one or two CNAME records that need to be created
3. For each record, note the Name and Value fields
4. Go to Cloudflare → sentorix.io → DNS
5. Click "Add record"
6. Type: CNAME
7. Name: paste the "Name" value from ACM (remove the trailing `.sentorix.io.` if Cloudflare auto-appends it)
8. Target: paste the "Value" from ACM
9. Proxy status: DNS only (grey cloud)
10. Save

Validation typically completes within 5–30 minutes after the records are added.

### Checking validation status

```bash
aws acm describe-certificate \
  --certificate-arn arn:aws:acm:us-east-1:${AWS_ACCOUNT_ID}:certificate/{id} \
  --region us-east-1 \
  --profile sentorix \
  --query 'Certificate.DomainValidationOptions[*].{Domain:DomainName,Status:ValidationStatus}'
```

Expected output when valid:
```json
[
  {"Domain": "sentorix.io", "Status": "SUCCESS"},
  {"Domain": "*.sentorix.io", "Status": "SUCCESS"}
]
```

### Certificate renewal

ACM automatically renews managed certificates 60 days before expiry. The DNS validation records must remain in Cloudflare for automatic renewal to succeed. **Do not delete the ACM validation CNAME records.**

---

## 5. Adding a New Subdomain

To add a new subdomain (for example, `status.sentorix.io`):

1. **Deploy the backend first.** Ensure whatever is serving the new subdomain (another ECS service, an S3 static site, etc.) is running and accessible via the ALB or a direct URL.

2. **Add the ALB listener rule** (if routing through the existing ALB):
   - Go to AWS Console → EC2 → Load Balancers → sentorix-alb → Listeners → HTTPS:443
   - Add a new rule: `Host is status.sentorix.io` → forward to the new target group
   - Or update Terraform to add the rule and `terraform apply`

3. **Verify the ACM certificate covers the new subdomain:**
   - The wildcard `*.sentorix.io` certificate already covers any first-level subdomain
   - It does **not** cover second-level subdomains like `api.status.sentorix.io` — those would require a separate certificate

4. **Add the DNS record in Cloudflare:**
   - Go to Cloudflare → sentorix.io → DNS → Add record
   - Type: CNAME
   - Name: `status`
   - Target: `sentorix-alb-{id}.us-east-1.elb.amazonaws.com` (same as other records)
   - Proxy status: DNS only (grey cloud)
   - Save

5. **Verify:**
   ```bash
   curl https://status.sentorix.io
   ```

DNS propagation with Cloudflare typically takes 1–5 minutes for new records.
