---
name: monitoring-planning
description: >-
  This skill should be used when the user asks to "set up monitoring", "plan monitoring for a domain", "monitor a website", "add monitoring for <domain>", "what checks do I need", or mentions comprehensive monitoring coverage. Covers domain analysis, check selection, phased setup workflow, upstream dependency detection, and CloudStatus monitoring for third-party providers.
---

# Monitoring planning

End-to-end workflow for planning and creating monitoring checks for a domain.

For check type parameters and constraints, see `references/check-types.md`. For domain-specific check recommendations, see `references/checklist-domain-monitoring.md`. For check selection and configuration guidance, see `references/guide-check-selection.md`. For alert routing and escalation design, see `references/guide-alerting-patterns.md`.

## Execution style

Execute the workflow directly. Do not present a plan for approval or ask the user to confirm each batch of checks. The tool invocation confirmations provide sufficient user control.

Only prompt the user when you need to:

- Clarify a vague or ambiguous request (e.g. which subdomains to monitor)
- Gather technical details you cannot infer (e.g. expected response content, auth requirements)
- Confirm actions with external visibility (status pages, CloudStatus dependencies)

## Quick reference: check categories

### Location-based checks (require explicit locations)

HTTP, DNS, ICMP, TCP, UDP, SMTP, IMAP, POP, SSH, NTP. Select 3-5 probe locations, set sensitivity >= 2.

### Constrained checks

**Page Speed** has unique restrictions:

- Maximum **1 location** (validation error if more)
- Minimum interval **1440 minutes** (1 day)
- Must use `Dedicated-*` location prefix (e.g. `Dedicated-United Kingdom-London`)
- Standard locations will fail; only dedicated probe nodes run Lighthouse

## Precondition: verify MCP tooling

Before starting any workflow, confirm that Uptime.com MCP tools are available (e.g. `list_checks`, `create_http_check`, `list_contacts`). If no `uptime` MCP tools are present:

1. Stop immediately. Do not proceed with domain analysis or check planning.
2. Tell the user: "The Uptime.com MCP server is not connected. Please run `/mcp` to check the server status and authenticate."
3. Do not attempt to look up the Uptime.com API documentation online or construct raw API payloads as a workaround.

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

### Phase 1: create tag

Create a tag before any checks. The tag is the primary organizational unit in Uptime.com: it drives Group check auto-selection, dashboard filtering, and SLA reporting scope.

Tag naming conventions:

- Use the registered domain as the tag name (e.g. `example.com`, not `Example` or `www.example.com`)
- For multi-environment setups, include the environment: `example.com/production`, `example.com/staging`
- For upstream dependencies, use a dedicated tag: `upstream-dependencies`

### Phase 2: create checks

1. **Batch 1, location-based** (all parallel): HTTP, DNS (A/MX/NS), ICMP, TCP.
2. **Batch 2, auto-located** (all parallel): SSL, Blacklist, Malware, WHOIS, RDAP.
3. **Batch 3, constrained** (last): Page Speed.

All checks within a batch are independent and can be created in a single parallel tool call.

Every check must include:

- `tags`: always pass the domain tag. Every check must be tagged from the moment of creation. Untagged checks are invisible to Group checks, excluded from tag-based dashboards, and hard to manage at scale.
- `contact_groups`: at least one contact group so alerts are not silent.
- `name`: use a consistent pattern: `<domain> <check-type>` (e.g. `example.com HTTP`, `example.com DNS A`, `realworld.show SSL`).
- `notes`: brief description of purpose when not obvious from the name.

### Phase 3: create Group check

After individual checks exist, create a Group check for the domain:

- Use tag-based auto-selection with the domain tag so future checks are automatically included.
- Configure alert conditions (e.g. "any member down").
- Tag the Group check itself with the same domain tag.

### Phase 4: create dashboard

1. `create_dashboard` with a descriptive name (e.g. "example.com monitoring").
2. Add widgets for the checks created in Phase 1.
3. Group widgets by check type or service function.

### Phase 5: suggest status page (for public-facing services)

If the domain is public-facing, suggest creating a status page. Do not create automatically; it's a public asset that requires user confirmation. See `status-page-management` skill for the full workflow.

### Phase 6: upstream dependency monitoring

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
- **Sensitivity = 1 with many locations**: excessive false positives. Use >= 2.
- **Missing tag on creation**: checks become ungrouped. Always create and assign the tag first.
