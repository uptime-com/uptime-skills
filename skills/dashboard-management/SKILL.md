---
name: dashboard-management
description: >-
  This skill should be used when the user asks to "create a dashboard", "set up a dashboard", "organize monitoring views", "build a dashboard", "add widgets", "monitoring dashboard", or wants to create visual overviews of their monitoring checks. Covers dashboard creation patterns and widget organization.
---

# Dashboard management — operational knowledge

Workflow patterns for creating and organizing monitoring dashboards.

## Dashboard creation workflow

### Step 1 — Determine scope

Dashboards work best when they have a clear purpose. Common patterns:

| Pattern                | Scope                             | Use case                                         |
| ---------------------- | --------------------------------- | ------------------------------------------------ |
| Domain dashboard       | All checks for one domain         | Per-service visibility                           |
| Team dashboard         | All checks owned by a team        | Team-level overview                              |
| Service tier dashboard | All critical/production checks    | Executive or on-call view                        |
| Check type dashboard   | All checks of one type (e.g. SSL) | Focused monitoring (certificate expiry overview) |

### Step 2 — Check capacity and gather checks

Call `get_account_usage` to check whether dashboard or widget limits apply to the account. If limits exist and are near capacity (>80%), warn the user before creating new dashboards or widgets.

Gather checks based on the chosen scope:

- `list_checks` with tag filter for domain/team dashboards
- `list_checks` then filter by check type for type-specific dashboards
- `list_tags` to discover grouping structure if scope isn't clear

### Step 3 — Create dashboard

`create_dashboard` with:

- **Name**: descriptive, matching the scope (e.g. "example.com monitoring", "Platform team overview")
- **Description**: brief note on what this dashboard covers and who it's for

### Step 4 — Add widgets

Add widgets for the checks gathered in Step 2. Widget selection depends on what information matters for the dashboard's audience.

## Widget selection guidance

| Widget type       | Best for             | Notes                                             |
| ----------------- | -------------------- | ------------------------------------------------- |
| Check status      | At-a-glance up/down  | Use for overview dashboards — shows current state |
| Response time     | Performance trending | Best for HTTP and TCP checks — shows latency      |
| Uptime percentage | SLA reporting        | Shows uptime over a time window                   |
| Alert history     | Incident patterns    | Shows when and how often alerts fired             |

### Widget configuration

When adding a widget, specify:

- **Check**: which monitoring check the widget displays data for
- **Widget type**: one of the types above
- **Time range**: the lookback window for historical data (e.g. 24h, 7d, 30d)

Response time and uptime percentage widgets are most useful with longer time ranges (7d+). Check status widgets are best with short/live views.

### Widget types by check type

Not all widget types make sense for all check types:

| Check type          | Check status | Response time | Uptime % | Alert history |
| ------------------- | ------------ | ------------- | -------- | ------------- |
| HTTP                | Yes          | Yes           | Yes      | Yes           |
| DNS, ICMP, TCP, UDP | Yes          | Yes           | Yes      | Yes           |
| SSL, WHOIS, RDAP    | Yes          | No            | Yes      | Yes           |
| Blacklist, Malware  | Yes          | No            | No       | Yes           |
| Page Speed          | Yes          | No            | No       | No            |
| Group               | Yes          | No            | Yes      | Yes           |
| Transaction, API    | Yes          | Yes           | Yes      | Yes           |

SSL, WHOIS, RDAP, Blacklist, and Malware are auto-located checks with infrequent intervals, so response time widgets add no value. Page Speed produces Lighthouse scores, not uptime data.

### Recommended layouts

**Domain dashboard**: group widgets by check type

1. HTTP checks (status + response time)
2. DNS checks (status)
3. SSL/Certificate checks (status + expiry)
4. Infrastructure checks (ICMP, TCP — status)
5. Registration checks (WHOIS, RDAP — status)

**On-call dashboard**: prioritize actionable information

1. Currently alerting checks (status, filtered to down)
2. Recently recovered checks
3. Response time anomalies

**Executive dashboard**: focus on outcomes

1. Uptime percentages for key services
2. Page Speed scores
3. SSL certificate expiry countdown

## Naming conventions

Consistent dashboard names help with discovery:

- `{domain} monitoring` — per-domain dashboards
- `{team} overview` — team dashboards
- `{purpose} dashboard` — special-purpose (e.g. "SSL expiry dashboard")

## Updating existing dashboards

When monitoring evolves, dashboards need to keep up.

### Adding checks to a dashboard

1. `get_dashboard` to see current widgets.
2. Identify which new checks are missing.
3. Add widgets for each new check, matching the existing layout pattern.

### Rebuilding a dashboard

When a dashboard has drifted significantly (many stale widgets, missing checks):

1. `list_checks` with the domain tag to get the current check set.
2. Compare against existing widgets.
3. Remove widgets for deleted checks.
4. Add widgets for new checks.
5. Reorder to match the layout conventions above.

### When to create a new dashboard vs update

- **Update**: checks were added or removed, but the scope is the same
- **New dashboard**: monitoring scope changed (e.g. a domain was split into multiple services, or a new team took ownership)

## Maintenance

Dashboards can drift as checks are added or removed:

- When creating new checks, add them to the relevant dashboard
- When deleting checks, associated widgets may need cleanup
- Periodically review dashboards for stale or orphaned widgets
- When `monitoring-optimization` finds configuration issues, update widgets to reflect changes

## Deleting dashboards

Use `delete_dashboard` only for permanent removal. Deletion removes all widgets. If a dashboard is temporarily unneeded, consider removing widgets instead and keeping the dashboard shell for later reuse.

Before deleting, confirm the dashboard isn't referenced by other team members or linked from external tools (e.g. Slack bookmarks, wiki pages).

## Linking dashboards to setup workflow

When `monitoring-planning` creates checks for a new domain, Phase 2 creates a dashboard. This ensures every monitored domain has a corresponding dashboard from day one. The dashboard should include widgets for all checks created during setup.
