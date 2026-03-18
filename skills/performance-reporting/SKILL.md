---
name: performance-reporting
description: >-
  This skill should be used when the user asks "what's our uptime", "how did we perform last month", "SLA report", "uptime percentage", "response time trends", "downtime summary", "show me outage stats", "annual uptime report", or needs to evaluate monitoring performance over a time period.
---

# Performance reporting

Workflow for generating uptime reports, SLA evaluations, and performance trend analysis using `get_check_stats`.

## Reporting workflow

### Step 1: determine scope

Clarify with the user:

- **Time range**: last 24h, last week, last month, last quarter, last year
- **Scope**: single domain, all domains, specific tag group, specific checks
- **Metrics of interest**: uptime %, response time, outage count, downtime duration

### Step 2: gather data

For domain-level reporting:

1. `list_checks` filtered by tag to get all checks for the domain.
2. `get_check_stats` for each check with the target date range (YYYY-MM-DD format). Supports location filtering for regional breakdowns.

For account-wide reporting:

1. `list_checks` to get all checks.
2. `list_tags` to group by domain/team.
3. `get_check_stats` for representative checks per group.

### Step 3: analyze

Key metrics from `get_check_stats`:

| Metric                      | What it tells you                                                   |
| --------------------------- | ------------------------------------------------------------------- |
| Uptime %                    | Availability over the period. Compare against SLA targets.          |
| Response time (avg/p95)     | Performance trends. Rising times may indicate degradation.          |
| Outage count                | Reliability. Frequent short outages may be worse than one long one. |
| Downtime duration (seconds) | Total unavailability. Convert to minutes/hours for readability.     |

### Step 4: present findings

Structure the report by audience:

**Executive summary** (non-technical stakeholders):

1. Overall uptime % across key services
2. SLA compliance: met or missed, by how much
3. Notable incidents and their impact
4. Trend: improving, stable, or degrading compared to prior period

**Engineering report** (SRE/DevOps):

1. Per-domain uptime breakdown
2. Per-check-type performance (HTTP response times, DNS resolution times)
3. Outage timeline with root causes
4. Regional breakdown (per-location stats if relevant)
5. Recommendations for improving reliability

## SLA evaluation

### Common SLA targets

| SLA tier | Uptime % | Allowed downtime/month | Allowed downtime/year |
| -------- | -------- | ---------------------- | --------------------- |
| 99%      | 99.00%   | ~7h 18m                | ~3d 15h               |
| 99.9%    | 99.90%   | ~43m 50s               | ~8h 46m               |
| 99.95%   | 99.95%   | ~21m 55s               | ~4h 23m               |
| 99.99%   | 99.99%   | ~4m 23s                | ~52m 36s              |

### Evaluating SLA compliance

1. Get uptime % from `get_check_stats` for the SLA period.
2. Compare against the target. Report as met/missed with margin.
3. If missed, list contributing outages with durations.
4. Note any outages during maintenance windows: these may be excluded from SLA calculations depending on the agreement.

### Caveats

- Uptime % from `get_check_stats` is calculated per day. For sub-day precision, cross-reference with `list_outages`.
- Paused checks don't generate downtime data. If a check was paused during an outage, stats won't reflect the real availability.
- Ignored alerts (via `ignore_alert`) are excluded from outage calculations and will affect uptime %.

## Trend analysis

When comparing periods (e.g. this month vs last month):

- **Uptime trend**: improving (higher %), stable, or degrading (lower %)
- **Response time trend**: faster, stable, or slower (watch for gradual increases)
- **Outage frequency**: fewer, same, or more incidents
- **Mean time to recovery (MTTR)**: average outage duration, shorter is better

If trends are negative, recommend a monitoring optimization review.

## Regional performance

Use location filtering in `get_check_stats` to break down performance by region:

- Identify regions with worse uptime or higher latency
- Compare against probe location distribution: poor coverage in a region means less visibility
- If a region consistently underperforms, it may indicate CDN configuration issues, DNS routing problems, or infrastructure gaps in that region
