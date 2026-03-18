---
name: monitoring-planning
description: >-
  This skill should be used when the user asks to "set up monitoring", "plan monitoring for a domain", "monitor a website", "add monitoring for <domain>", "what checks do I need", or mentions comprehensive monitoring coverage. Covers domain analysis, check selection, phased setup workflow, upstream dependency detection, and CloudStatus monitoring for third-party providers.
---

# Monitoring planning

End-to-end workflow for planning and creating monitoring checks for a domain.

For check type parameters and constraints, see `references/check-types.md`. For domain-specific check recommendations, see `references/checklist-domain-monitoring.md`.

## Quick reference: check categories

### Location-based checks (require explicit locations)

HTTP, DNS, ICMP, TCP, UDP, SMTP, IMAP, POP, SSH, NTP. Select 3-5 probe locations, set sensitivity >= 2.

### Auto-located checks (locations assigned by server)

SSL, Blacklist, Malware, WHOIS, RDAP. Do NOT pass locations. The server assigns them automatically. Passing locations will cause a validation error.

### Constrained checks

**Page Speed** has unique restrictions:

- Maximum **1 location** (validation error if more)
- Minimum interval **1440 minutes** (1 day)
- Must use `Dedicated-*` location prefix (e.g. `Dedicated-United Kingdom-London`)
- Standard locations will fail; only dedicated probe nodes run Lighthouse

## Setup workflow

### Phase 0: verify notification contacts

Before creating checks, ensure notification contacts exist:

1. `list_contacts` to see existing contact groups.
2. If a suitable group exists (e.g. "Default"), use it for all checks.
3. If the user needs a dedicated contact group, `create_contact` with:
   - **name**: descriptive (e.g. "Platform team", "On-call SRE")
   - **email**: notification email addresses
   - **sms**: phone numbers in E.164 format (e.g. `+15551234567`) for SMS alerts

Checks without a contact group alert silently (logged but no notification sent).

### Phase 1: create tag and checks

1. **Tag first**: `create_tag` (e.g. `example.com`) to group all checks.
2. **Batch 1, location-based** (all parallel): HTTP, DNS (A/MX/NS), ICMP, TCP.
3. **Batch 2, auto-located** (all parallel): SSL, Blacklist, Malware, WHOIS, RDAP.
4. **Batch 3, constrained** (last): Page Speed.

All checks within a batch are independent and can be created in a single parallel tool call.

5. **Group check**: after individual checks exist, create a Group check for the domain. Use tag-based auto-selection with the domain tag so future checks are automatically included.

### Phase 2: create dashboard

1. `create_dashboard` with a descriptive name (e.g. "example.com monitoring").
2. Add widgets for the checks created in Phase 1.
3. Group widgets by check type or service function.

### Phase 3: suggest status page (for public-facing services)

If the domain is public-facing, suggest creating a status page. Do not create automatically; it's a public asset that requires user confirmation. See `status-page-management` skill for the full workflow.

### Phase 4: upstream dependency monitoring

Detect and offer to monitor upstream dependencies using CloudStatus checks.

#### DNS-based provider detection

DNS records reveal infrastructure providers without user input:

| Record type | Pattern                                     | Reveals                 |
| ----------- | ------------------------------------------- | ----------------------- |
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

Inspect DNS check results from Phase 1 for these patterns before proceeding.

#### User confirmation

Always confirm before creating dependency checks:

> I detected these upstream dependencies from your DNS records:
>
> - **CDN**: Cloudflare (CNAME -> cdn.cloudflare.net)
> - **Email**: Google Workspace (MX -> google.com)
> - **DNS**: Cloudflare (NS -> cloudflare.com)
>
> Would you like me to monitor their status pages? Are there other dependencies I should include (e.g. payment processor, auth provider)?

#### Creating CloudStatus checks

For each confirmed dependency:

1. **Create CloudStatus check**: use the MCP server's tools to discover available providers and service components. Uptime.com natively parses status feeds with proper UP/DOWN/MAINTENANCE mapping.
2. **Tag** with `upstream-dependencies` to keep them organized separately.

Add dependency checks to the domain's dashboard in a separate "Dependencies" widget group.

## DNS layering

For comprehensive DNS coverage, create three checks:

| Record | Target                             | Catches                             |
| ------ | ---------------------------------- | ----------------------------------- |
| A      | subdomain (e.g. `www.example.com`) | resolution failures for the service |
| MX     | parent domain (e.g. `example.com`) | mail routing breakage               |
| NS     | parent domain (e.g. `example.com`) | nameserver delegation issues        |

NS breakage cascades into A and MX failures, so it provides the earliest signal.

## Domain vs subdomain rules

| Check type                          | Target                     | Why                                                |
| ----------------------------------- | -------------------------- | -------------------------------------------------- |
| HTTP, ICMP, SSL, Blacklist, Malware | subdomain or full URL      | checks the actual service endpoint                 |
| DNS A/AAAA/CNAME                    | subdomain                  | resolves the specific host                         |
| DNS MX/NS                           | parent (registered) domain | MX and NS are zone-level records                   |
| WHOIS, RDAP                         | parent (registered) domain | WHOIS/RDAP data exists only for registered domains |

WHOIS and RDAP both require `expect_string` set to the domain name (e.g. `example.com`). Creating both provides redundancy since WHOIS servers can be unreliable.

## Common pitfalls

- **Page Speed with multiple locations**: "Max 1 locations allowed". Use exactly one `Dedicated-*` location.
- **Page Speed interval < 1440**: "minimum interval for this check type is 1 days". Use 1440 or higher.
- **WHOIS/RDAP on subdomain**: will fail or return no data. Always use the registered parent domain.
- **Passing locations to SSL/Blacklist/Malware**: validation error. Omit locations entirely.
- **Sensitivity = 1 with many locations**: excessive false positives. Use >= 2.
- **Missing tag on creation**: checks become ungrouped. Always create and assign the tag first.
