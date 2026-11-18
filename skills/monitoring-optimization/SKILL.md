---
name: monitoring-optimization
description: >-
  This skill should be used when the user asks to "audit monitoring", "optimize checks", "review check coverage", "fill monitoring gaps", "what am I monitoring", "find missing monitors", "update a check", "change check interval", "change contact group", "review my checks", or wants to assess monitoring completeness, fix configuration issues, or modify existing checks.
---

# Monitoring optimization

Workflow for reviewing existing monitoring, identifying gaps, optimizing configuration, and managing checks.

## Audit workflow

### Step 1: inventory

Gather the full picture:

1. `get_account_usage`: plan limits and current consumption (check slots, per-type limits).
2. `list_checks`: every check's id, name, type and address. It pages at 25 by default; raise `page_size` or walk `page` until the header's total is covered.
3. `list_tags`: how checks are grouped.
4. `list_contacts`: verify notification routing exists.

All four calls are independent and can run in parallel. Configuration (status, interval, sensitivity, locations, tags, notes) comes from `get_check` per check; run those in parallel too. `list_checks` filters by `tag`, `type`, `is_paused` and `search` to narrow the set first.

### Step 2: group by domain

Organize checks by target domain:

- Extract the registered domain from each check's address
- Group checks under their domain
- Note which check types exist per domain

### Step 3: gap analysis

For each domain, compare against recommended coverage:

**Critical gaps** (should almost always exist):

- No HTTP check: service availability not monitored
- No SSL check: certificate expiry not monitored
- No DNS A check: name resolution not monitored

**Important gaps**:

- No DNS NS check: nameserver health not monitored (cascading risk)
- No WHOIS/RDAP: domain registration expiry not monitored
- Mail-sending domain without SMTP check: delivery not monitored
- Mail-sending domain without Blacklist check: IP reputation not monitored

**Nice-to-have gaps**:

- No Page Speed: performance regression not tracked
- No Malware check: compromise detection missing
- Single DNS record type: limited DNS coverage
- No Group check: no aggregate view of domain health (standard for production domains)

### Step 4: configuration review

Check for suboptimal configurations:

| Issue                     | Detection                                           | Recommendation                             |
| ------------------------- | --------------------------------------------------- | ------------------------------------------ |
| Sensitivity = 1           | `Sensitivity` in `get_check` on location-based ones | Increase to >= 2 to reduce false positives |
| Very long intervals       | `Interval` > 10 minutes on HTTP checks              | Consider 1-5 min for critical services     |
| All checks same locations | Same `Locations` on every check                     | Diversify to catch regional issues         |
| Paused checks             | `list_checks` with `is_paused: true`                | Verify if intentional or forgotten         |
| No tags                   | No `Tags` line in `get_check`                       | Add domain-based tags for organization     |
| No Group check per domain | No group-type check under the domain `tag`          | Create a Group check with `check_tags`     |

`get_check` does not report contact groups, escalations or maintenance windows. Ask the user to confirm in the web app that critical checks have a contact group and escalation rules, and that known maintenance periods have windows.

### Step 5: upstream dependency audit

Scan existing check addresses for known third-party domains (e.g. `api.stripe.com`, `*.auth0.com`). These are dependencies that should have CloudStatus monitoring.

For each detected dependency without a CloudStatus check, recommend adding one.

### Step 6: report

Structure the audit report as:

1. **Account usage**: plan limits vs. current consumption, remaining capacity, any types at >80% utilization
2. **Inventory summary**: N checks across M domains, K tags
3. **Coverage by domain**: table showing which check types exist per domain
4. **Gaps found**: prioritized list of missing checks with severity
5. **Upstream dependencies**: detected providers, which have status monitoring
6. **Configuration issues**: suboptimal settings that should be adjusted
7. **Recommendations**: specific checks to create, ordered by priority (flagging any that would exceed plan limits)

## Modifying existing checks

Each check type has its own update tool, `update_<type>_check` (e.g. `update_http_check`, `update_dns_check`), taking the check's `id`. Only the fields provided change; omitted fields keep their values.

### Update workflow

1. **Get current state**: `get_check` for the existing configuration and the `Type` that picks the update tool.
2. **Apply changes**: the type's update tool with only the fields that need to change.
3. **Verify**: `get_check` again to confirm.

### Commonly updated fields

| Field            | Notes                                                                                                                                            |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `interval`       | Minutes between checks. Some types have minimums (e.g. Page Speed >= 1440). Auto-located types (SSL, Blacklist, Malware, WHOIS, RDAP) take none. |
| `locations`      | Probe location identifiers. Location-based checks, Page Speed and CloudStatus only.                                                              |
| `sensitivity`    | Number of confirming locations before alerting. Use >= 2. Location-based and scripted checks only.                                               |
| `contact_groups` | Contact group names.                                                                                                                             |
| `is_paused`      | `true` to pause, `false` to resume. API, Transaction and CloudStatus checks only.                                                                |
| `tags`           | Replaces the full tag list. Include existing tags you want to keep.                                                                              |

### Batch updates

1. `list_checks` with `tag` (and `type`) to get the target set.
2. Issue the matching `update_<type>_check` calls in parallel.
3. Verify a sample with `get_check`.

## Maintenance windows

Scheduled maintenance is preferred over manual pausing. Neither windows nor pausing of most check types is available through the MCP tools, so recommend them and have the user set them in the web app:

- Define recurring windows (e.g. every Sunday 02:00-04:00 UTC)
- Checks pause and resume automatically
- Better audit trail than manual pause/resume

Use manual pausing only for unplanned, one-off situations.

## Escalations

Escalations are separate from contact groups. They define how alerts intensify:

- Contact groups receive the initial alert
- Escalation rules trigger if the alert is not acknowledged within a configured window
- Can notify different contacts or use different channels (e.g. phone call after SMS)

Verify critical checks have both a contact group _and_ escalation rules.

## Tag hygiene

Common issues:

- **Ungrouped checks**: no tags, hard to manage at scale
- **Stale tags**: tags with no checks (orphaned after deletions)
- **Inconsistent naming**: mix of `example.com`, `Example.com`, `example`

Recommend: one tag per registered domain, optional tags for environment and team.

## Interval recommendations

| Service tier                    | HTTP interval | DNS interval | Other intervals |
| ------------------------------- | ------------- | ------------ | --------------- |
| Critical (revenue-generating)   | 1 min         | 5 min        | 5-10 min        |
| Standard (internal tools)       | 5 min         | 10 min       | 10-30 min       |
| Low priority (informational)    | 10 min        | 30 min       | 60 min          |
| Auto-located (SSL, WHOIS, etc.) | -             | -            | 60-1440 min     |

## Deleting checks

Use `delete_check` only for permanent removal. Deletion is irreversible; check history is lost. Prefer pausing for temporary deactivation.
