---
name: checklist-domain-monitoring
description: >-
  Reference document mapping domain patterns (SaaS, e-commerce, API, email)
  to recommended check types and configurations. Used by monitoring-setup
  and monitoring-audit skills.
---

# Domain monitoring checklist

Recommended check coverage by domain pattern. Use this to determine which checks
to create for a given type of service.

## Core checks (every domain)

These checks should exist for virtually every monitored domain:

| Check  | Priority | Why                                                        |
|--------|----------|------------------------------------------------------------|
| HTTP   | Critical | Confirms the service responds and returns expected content |
| SSL    | Critical | Certificate expiry and chain validation                    |
| DNS A  | Critical | Name resolution for the service endpoint                   |
| DNS NS | High     | Nameserver health — NS failure cascades into everything    |
| WHOIS  | Medium   | Domain registration expiry warning                         |
| RDAP   | Medium   | Redundancy for WHOIS (WHOIS servers can be unreliable)     |

## SaaS / Web application

Public-facing web application with user login, dashboard, API.

| Check            | Configuration notes                                                             |
|------------------|---------------------------------------------------------------------------------|
| HTTP (main site) | Check homepage or health endpoint. Use `expect_string` for content verification |
| HTTP (app/login) | Check authenticated endpoint if possible, or login page                         |
| HTTP (API)       | Hit a lightweight API endpoint (e.g. `/health`, `/api/v1/status`)               |
| DNS A            | www subdomain                                                                   |
| DNS MX           | Parent domain — needed if the app sends email (password resets, notifications)  |
| DNS NS           | Parent domain                                                                   |
| SSL              | Main domain                                                                     |
| Page Speed       | Homepage — monitors real user experience degradation                            |
| Blacklist        | Catch reputation issues early                                                   |
| WHOIS + RDAP     | Domain registration monitoring                                                  |

## E-commerce

Online store with payments, product catalog, checkout flow.

| Check                 | Configuration notes                                                  |
|-----------------------|----------------------------------------------------------------------|
| HTTP (storefront)     | Homepage with `expect_string` for key content                        |
| HTTP (checkout)       | Checkout page availability (not the payment processor itself)        |
| HTTP (API/search)     | Product search or catalog API endpoint                               |
| TCP (payment gateway) | Port 443 to payment processor hostname — detects connectivity issues |
| DNS A + MX + NS       | Full DNS layering — email is critical for order confirmations        |
| SSL                   | Main domain — expired SSL kills checkout trust                       |
| Page Speed            | Storefront — page speed directly impacts conversion                  |
| Blacklist + Malware   | E-commerce sites are high-value targets                              |
| WHOIS + RDAP          | Domain expiry would be catastrophic                                  |

## API service

Backend API consumed by mobile apps, integrations, or other services.

| Check                  | Configuration notes                                                          |
|------------------------|------------------------------------------------------------------------------|
| HTTP (health endpoint) | `/health` or `/status` with `expect_string` for JSON response                |
| HTTP (key endpoint)    | A representative API call that exercises the main code path                  |
| TCP                    | API port (usually 443) — catches networking/firewall issues faster than HTTP |
| DNS A                  | API subdomain (e.g. `api.example.com`)                                       |
| DNS NS                 | Parent domain                                                                |
| SSL                    | API domain — clients will reject expired certs                               |
| WHOIS + RDAP           | Domain registration                                                          |

## Email / mail server

Dedicated email infrastructure (MX, SMTP, IMAP/POP).

| Check        | Configuration notes                                    |
|--------------|--------------------------------------------------------|
| SMTP         | Mail server hostname, port 25 (or 587 for submission)  |
| IMAP         | Port 993 (TLS) or 143                                  |
| POP          | Port 995 (TLS) or 110 — if POP is supported            |
| DNS MX       | Parent domain — the most critical DNS record for email |
| DNS A        | Mail server hostname                                   |
| DNS NS       | Parent domain                                          |
| DNS TXT      | Check for SPF record presence                          |
| SSL          | Mail server hostname — TLS for SMTP/IMAP/POP           |
| Blacklist    | Critical — IP blacklisting breaks email delivery       |
| WHOIS + RDAP | Domain registration                                    |

## Coverage gap indicators

When auditing existing monitoring, these patterns suggest gaps:

| Symptom                                    | Likely missing                                     |
|--------------------------------------------|----------------------------------------------------|
| No DNS checks                              | DNS A + NS at minimum                              |
| HTTP only, no SSL                          | SSL certificate monitoring                         |
| No WHOIS/RDAP                              | Domain registration expiry alerting                |
| MX records but no SMTP check               | SMTP connectivity monitoring                       |
| No Page Speed                              | Performance regression detection (SaaS/e-commerce) |
| Single HTTP check for multi-service domain | Separate checks per service endpoint               |
| No Blacklist check on mail-sending domain  | IP reputation monitoring                           |