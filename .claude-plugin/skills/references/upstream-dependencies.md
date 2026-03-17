---
name: upstream-dependencies
description: >-
  Reference for detecting upstream service dependencies via DNS and known
  provider patterns. Used by monitoring-setup and monitoring-audit skills.
---

# Upstream dependency detection and monitoring

## Detection methods

### 1. DNS record inference

DNS records reveal infrastructure providers without any user input:

| Record type | What to look for                            | Reveals                 |
|-------------|---------------------------------------------|-------------------------|
| CNAME chain | `*.cloudfront.net`                          | AWS CloudFront CDN      |
| CNAME chain | `*.cdn.cloudflare.net`                      | Cloudflare CDN          |
| CNAME chain | `*.fastly.net`                              | Fastly CDN              |
| CNAME chain | `*.akamaiedge.net`, `*.akamai.net`          | Akamai CDN              |
| CNAME chain | `*.azureedge.net`                           | Azure CDN               |
| CNAME chain | `*.herokuapp.com`                           | Heroku                  |
| CNAME chain | `*.netlify.app`                             | Netlify                 |
| CNAME chain | `*.vercel-dns.com`                          | Vercel                  |
| MX          | `*.google.com`, `*.googlemail.com`          | Google Workspace        |
| MX          | `*.outlook.com`, `*.protection.outlook.com` | Microsoft 365           |
| MX          | `*.pphosted.com`                            | Proofpoint              |
| MX          | `*.mimecast.com`                            | Mimecast                |
| NS          | `*.cloudflare.com`                          | Cloudflare DNS          |
| NS          | `awsdns-*`                                  | AWS Route 53            |
| NS          | `*.azure-dns.*`                             | Azure DNS               |
| NS          | `ns*.google.com`                            | Google Cloud DNS        |
| TXT (SPF)   | `include:_spf.google.com`                   | Google email sending    |
| TXT (SPF)   | `include:spf.protection.outlook.com`        | Microsoft email sending |
| TXT (SPF)   | `include:sendgrid.net`                      | SendGrid                |
| TXT (SPF)   | `include:amazonses.com`                     | AWS SES                 |
| TXT (SPF)   | `include:mailgun.org`                       | Mailgun                 |

To perform DNS inference: create DNS checks during Phase 1, then inspect
the returned records for provider signatures before proceeding to dependency
monitoring.

### 2. Ask the user

Prompt with common dependency categories:

> I've set up monitoring for your domain. Would you also like to monitor
> upstream dependencies? Common ones include:
> - **CDN**: Cloudflare, CloudFront, Fastly, Akamai
> - **DNS provider**: Cloudflare, Route 53, Google Cloud DNS
> - **Email provider**: Google Workspace, Microsoft 365, SendGrid
> - **Auth provider**: Auth0, Okta, Firebase Auth
> - **Payment processor**: Stripe, PayPal, Adyen
> - **Cloud platform**: AWS, GCP, Azure
> - **CI/CD**: GitHub, GitLab, CircleCI
> - **Database hosting**: MongoDB Atlas, PlanetScale, Supabase
>
> Which of these (or others) does your service depend on?

### 3. Existing check analysis (audit only)

During monitoring audits, scan existing check addresses for known third-party
domains. If checks target `api.stripe.com` or `auth0.com`, those are
dependencies that should have CloudStatus monitoring.

## Creating dependency monitoring checks

Use CloudStatus checks — Uptime.com's native cloud provider status monitoring:

- Uptime.com parses and normalizes the provider's status feed automatically
- Proper UP/DOWN/MAINTENANCE status mapping per provider
- Integrates with contact groups, escalations, and SLA reporting
- Use the MCP server's tools to discover available providers and service components

## Organizing dependency checks

- Tag all dependency checks separately from direct service checks
  (e.g. `upstream-dependencies` tag)
- Consider a dedicated "Dependencies" dashboard grouping all upstream monitors
- In the audit report, list dependencies as a separate section from direct checks
