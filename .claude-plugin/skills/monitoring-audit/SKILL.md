---
name: monitoring-audit
description: >-
  This skill should be used when the user asks to "audit monitoring", "review check coverage", "what am I monitoring", "monitoring gaps", "check coverage report", "review my checks", "find missing monitors", or wants to assess their monitoring completeness and configuration quality. Covers coverage analysis, gap detection, and configuration recommendations.
---

# Monitoring audit — operational knowledge

Workflow patterns for reviewing monitoring coverage, identifying gaps, and recommending improvements.

For check type coverage by domain pattern, see `references/checklist-domain-monitoring.md`.

## Audit workflow

### Step 1 — Inventory

Gather the full picture:

1. `list_checks` — get all checks with their types, targets, and status.
2. `list_tags` — see how checks are grouped.
3. `list_contacts` — verify notification routing exists.

### Step 2 — Group by domain

Organize checks by their target domain to see coverage per service:

- Extract the registered domain from each check's address
- Group checks under their domain
- Note which check types exist for each domain

### Step 3 — Gap analysis

For each domain, compare against the recommended coverage from `references/checklist-domain-monitoring.md`:

**Critical gaps** (should almost always exist):

- No HTTP check → service availability not monitored
- No SSL check → certificate expiry not monitored
- No DNS A check → name resolution not monitored

**Important gaps**:

- No DNS NS check → nameserver health not monitored (cascading risk)
- No WHOIS/RDAP → domain registration expiry not monitored
- Mail-sending domain without SMTP check → delivery not monitored
- Mail-sending domain without Blacklist check → IP reputation not monitored

**Nice-to-have gaps**:

- No Page Speed → performance regression not tracked
- No Malware check → compromise detection missing
- Single DNS record type → limited DNS coverage
- No Group check → no aggregate view of domain health (standard for production domains)

### Step 4 — Configuration review

Check for suboptimal configurations:

| Issue                     | Detection                                       | Recommendation                                                                        |
| ------------------------- | ----------------------------------------------- | ------------------------------------------------------------------------------------- |
| Sensitivity = 1           | `sensitivity` field on location-based checks    | Increase to ≥ 2 to reduce false positives                                             |
| Very long intervals       | `interval` > 10 for HTTP checks                 | Consider 1–5 min for critical services                                                |
| No contact group          | Empty `contact_groups`                          | Assign a contact group or checks alert silently                                       |
| No escalation rules       | Missing `escalations` on critical checks        | Add escalation so unacknowledged alerts intensify                                     |
| All checks same locations | Same `locations` array everywhere               | Diversify to catch regional issues                                                    |
| Paused checks             | `is_paused: true`                               | Verify if intentional or forgotten — suggest maintenance windows for recurring pauses |
| No tags                   | Empty `tags`                                    | Add domain-based tags for organization                                                |
| No Group check per domain | Checks exist but no aggregate Group check       | Create Group with tag-based auto-selection                                            |
| No maintenance windows    | Critical checks with no scheduled maintenance   | Set up windows for known maintenance periods                                          |
| Default timeout only      | No custom `timeout` set                         | Tune timeout for services with known latency profiles                                 |
| IPv4-only on dual-stack   | `use_ip_version` = IPv4 on IPv6-capable targets | Consider `ANY` or explicit IPv6 checks                                                |

### Step 5 — Upstream dependency audit

Assess whether critical upstream dependencies are monitored. See `references/upstream-dependencies.md` for the full detection and monitoring reference.

#### Detection

1. **DNS inference** — inspect existing DNS check results for CNAME chains, MX records, NS records, and SPF TXT records that reveal infrastructure providers (CDN, email, DNS hosting, email delivery services).
2. **Check address analysis** — scan existing check addresses for known third-party domains (e.g. `api.stripe.com`, `*.auth0.com`). These are explicit dependencies.
3. **Ask the user** — present detected providers and ask about additional dependencies not visible from DNS or existing checks.

#### Dependency gap analysis

For each detected dependency, check whether a corresponding status page monitoring check exists:

| Finding                                             | Severity  | Action                             |
| --------------------------------------------------- | --------- | ---------------------------------- |
| Dependency detected, no CloudStatus check           | Important | Recommend adding CloudStatus check |
| No `upstream-dependencies` tag on dependency checks | Low       | Recommend tagging for organization |

#### User confirmation

Before recommending new dependency checks, confirm detected dependencies:

> I identified these upstream providers from your DNS records and existing checks:
>
> - **CDN**: Cloudflare (from CNAME chain)
> - **Email**: Google Workspace (from MX records)
> - **Payments**: Stripe (from existing check on api.stripe.com)
>
> None of these have status page monitoring. Would you like me to set that up? Are there other dependencies I should include?

### Step 6 — Report

Structure the audit report as:

1. **Inventory summary**: N checks across M domains, K tags
2. **Coverage by domain**: table showing which check types exist per domain
3. **Gaps found**: prioritized list of missing checks with severity
4. **Upstream dependencies**: detected providers, which have status monitoring, which don't
5. **Configuration issues**: suboptimal settings that should be adjusted
6. **Recommendations**: specific checks to create, ordered by priority

## Tag hygiene

Tags are the primary organizational mechanism. Common issues:

- **Ungrouped checks**: checks with no tags — hard to manage at scale
- **Stale tags**: tags with no checks (orphaned after deletions)
- **Inconsistent naming**: mix of `example.com`, `Example.com`, `example` for the same domain

Recommend a consistent tagging strategy:

- One tag per registered domain (e.g. `example.com`)
- Optional tags for environment (`production`, `staging`)
- Optional tags for team ownership (`team-platform`, `team-frontend`)

## Location distribution review

For location-based checks, review probe location distribution:

- **Geographic diversity**: locations should span multiple regions
- **Consistency**: related checks should use the same locations for comparable results
- **Dedicated locations**: Page Speed must use `Dedicated-*` locations

If all checks use the same 2-3 locations, a regional network issue could cause false negatives. Recommend 3–5 locations across at least 2 continents for production services.

## Interval recommendations

| Service tier                    | HTTP interval | DNS interval | Other intervals |
| ------------------------------- | ------------- | ------------ | --------------- |
| Critical (revenue-generating)   | 1 min         | 5 min        | 5–10 min        |
| Standard (internal tools)       | 5 min         | 10 min       | 10–30 min       |
| Low priority (informational)    | 10 min        | 30 min       | 60 min          |
| Auto-located (SSL, WHOIS, etc.) | —             | —            | 60–1440 min     |
