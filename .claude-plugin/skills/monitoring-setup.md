---
name: monitoring-setup
description: >-
  This skill should be used when the user asks to "set up monitoring",
  "create checks for a domain", "monitor a website", "set up uptime checks",
  "add monitoring for <domain>", or mentions comprehensive monitoring coverage.
  Covers initial check creation, constraint awareness, and end-to-end setup
  including dashboards and status pages. For modifying existing checks, see
  check-management.
---

# Monitoring setup — operational knowledge

Practical constraints and workflow patterns for creating monitoring checks.
This complements the MCP server instructions (which cover check types and location strategy)
with operational knowledge discovered during real monitoring setups.

For the full check type matrix with parameters and constraints, see
`references/check-types.md`. For domain-specific checklists, see
`references/domain-monitoring-checklist.md`.

## Quick reference: check categories

### Location-based checks (require explicit locations)

HTTP, DNS, ICMP, TCP, UDP, SMTP, IMAP, POP, SSH, NTP — select 3–5 probe locations,
set sensitivity ≥ 2.

### Auto-located checks (locations assigned by server)

SSL, Blacklist, Malware, WHOIS, RDAP — do NOT pass locations. The server assigns them
automatically. Passing locations will cause a validation error.

### Constrained checks

**Page Speed** has unique restrictions:

- Maximum **1 location** (validation error if more)
- Minimum interval **1440 minutes** (1 day)
- Must use `Dedicated-*` location prefix (e.g. `Dedicated-United Kingdom-London`)
- Standard locations will fail — only dedicated probe nodes run Lighthouse

## End-to-end setup workflow

Complete monitoring setup for a domain follows these phases:

### Phase 1 — Create tag and checks

1. **Tag first** — `create_tag` (e.g. `example.com`) to group all checks.
2. **Batch 1 — location-based** (all parallel): HTTP, DNS (A/MX/NS), ICMP, TCP.
3. **Batch 2 — auto-located** (all parallel): SSL, Blacklist, Malware, WHOIS, RDAP.
4. **Batch 3 — constrained** (last): Page Speed.

All checks within a batch are independent and can be created in a single parallel tool call.

5. **Group check** — after individual checks exist, create a Group check for the domain.
   Use tag-based auto-selection with the domain tag (e.g. `example.com`) so future
   checks are automatically included. Configure alert conditions (e.g. "any member down").

### Phase 2 — Create dashboard

After checks exist, create a dashboard to visualize them:

1. `create_dashboard` with a descriptive name (e.g. "example.com monitoring").
2. Add widgets for the checks created in Phase 1.
3. Group widgets by check type or service function.

### Phase 3 — Suggest status page (for public-facing services)

If the monitored domain is a public-facing service, suggest creating a status page:

- Map checks to status page components (e.g. HTTP check → "Website", DNS → "DNS", etc.)
- See `status-page-management` skill for the full workflow.

Do not create a status page automatically — it's a public-facing asset that requires
user confirmation.

### Phase 4 — Upstream dependency monitoring

After the core checks are in place, detect and offer to monitor upstream dependencies.
See `references/upstream-dependencies.md` for the full detection and monitoring reference.

#### Detection

Identify dependencies using available signals:

1. **DNS inference** — inspect CNAME chains, MX, NS, and SPF records from checks
   created in Phase 1. These reveal CDN, email, and DNS providers without user input.
2. **Ask the user** — present detected providers and prompt for additional dependencies
   (payment processors, auth providers, cloud platform, CI/CD, etc.).

#### User confirmation

Always confirm before creating dependency checks:

> I detected these upstream dependencies from your DNS records:
> - **CDN**: Cloudflare (CNAME → cdn.cloudflare.net)
> - **Email**: Google Workspace (MX → google.com)
> - **DNS**: Cloudflare (NS → cloudflare.com)
>
> Would you like me to monitor their status pages? Are there other
> dependencies I should include (e.g. payment processor, auth provider)?

#### Monitoring setup

For each confirmed dependency:

1. **Create CloudStatus check** — use the MCP server's tools to discover available
   providers and service components, then create a CloudStatus check. Uptime.com
   natively parses status feeds with proper UP/DOWN/MAINTENANCE mapping.
2. **Tag** with `upstream-dependencies` to keep them organized separately.

Add dependency checks to the domain's dashboard in a separate "Dependencies"
widget group.

## DNS layering

For comprehensive DNS coverage, create three checks:

| Record | Target                             | Catches                             |
|--------|------------------------------------|-------------------------------------|
| A      | subdomain (e.g. `www.example.com`) | resolution failures for the service |
| MX     | parent domain (e.g. `example.com`) | mail routing breakage               |
| NS     | parent domain (e.g. `example.com`) | nameserver delegation issues        |

NS breakage cascades into A and MX failures, so it provides the earliest signal.

## Domain vs subdomain rules

| Check type                          | Target                     | Why                                                                |
|-------------------------------------|----------------------------|--------------------------------------------------------------------|
| HTTP, ICMP, SSL, Blacklist, Malware | subdomain or full URL      | checks the actual service endpoint                                 |
| DNS A/AAAA/CNAME                    | subdomain                  | resolves the specific host                                         |
| DNS MX/NS                           | parent (registered) domain | MX and NS are zone-level records                                   |
| WHOIS, RDAP                         | parent (registered) domain | WHOIS/RDAP data exists only for registered domains, not subdomains |

WHOIS and RDAP both require `expect_string` — set it to the domain name being monitored
(e.g. `example.com`). Creating both provides redundancy since WHOIS servers can be unreliable.

## Common pitfalls

- **Page Speed with multiple locations** → "Max 1 locations allowed" — use exactly one `Dedicated-*` location.
- **Page Speed interval < 1440** → "minimum interval for this check type is 1 days" — use 1440 or higher.
- **WHOIS/RDAP on subdomain** → will fail or return no data — always use the registered parent domain.
- **Passing locations to SSL/Blacklist/Malware** → validation error — omit locations entirely.
- **Sensitivity = 1 with many locations** → excessive false positives — use ≥ 2.
- **Missing tag on creation** → checks become ungrouped — always create and assign the tag first.