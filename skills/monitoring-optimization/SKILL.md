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
2. `list_checks`: all checks with types, targets, and status.
3. `list_tags`: how checks are grouped.
4. `list_contacts`: verify notification routing exists.

All four calls are independent and can run in parallel.

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

| Issue                     | Detection                                       | Recommendation                                    |
| ------------------------- | ----------------------------------------------- | ------------------------------------------------- |
| Sensitivity = 1           | `sensitivity` field on location-based checks    | Increase to >= 2 to reduce false positives        |
| Very long intervals       | `interval` > 10 for HTTP checks                 | Consider 1-5 min for critical services            |
| No contact group          | Empty `contact_groups`                          | Assign a contact group or checks alert silently   |
| No escalation rules       | Missing `escalations` on critical checks        | Add escalation so unacknowledged alerts intensify |
| All checks same locations | Same `locations` array everywhere               | Diversify to catch regional issues                |
| Paused checks             | `is_paused: true`                               | Verify if intentional or forgotten                |
| No tags                   | Empty `tags`                                    | Add domain-based tags for organization            |
| No Group check per domain | Checks exist but no aggregate                   | Create Group with tag-based auto-selection        |
| No maintenance windows    | Critical checks with no scheduled maintenance   | Set up windows for known maintenance periods      |
| IPv4-only on dual-stack   | `use_ip_version` = IPv4 on IPv6-capable targets | Consider `ANY` or explicit IPv6 checks            |

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

Use `update_check` to modify check properties in place. Only specified fields change; omitted fields retain current values.

### Update workflow

1. **Get current state**: `get_check` to see existing configuration.
2. **Apply changes**: `update_check` with only the fields that need to change.
3. **Verify**: `get_check` again to confirm.

### Commonly updated fields

| Field            | Notes                                                                             |
| ---------------- | --------------------------------------------------------------------------------- |
| `interval`       | Minutes between checks. Some types have minimums (e.g. Page Speed >= 1440).       |
| `locations`      | Probe locations. Only for location-based checks; never set on auto-located types. |
| `sensitivity`    | Number of confirming locations before alerting. Use >= 2.                         |
| `contact_groups` | Array of contact group names/IDs.                                                 |
| `is_paused`      | `true` to pause, `false` to resume.                                               |
| `tags`           | Replaces the full tag list. Include existing tags you want to keep.               |

### Batch updates

1. `list_checks` with tag filter to get the target set.
2. Issue `update_check` calls in parallel.
3. Verify a sample with `get_check`.

## Maintenance windows

Scheduled maintenance is preferred over manual pausing:

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

## Known MCP server issues

- **`get_contact` type parsing**: may fail with integer/string type mismatch on the `id` parameter. Use `list_contacts` as a workaround.
