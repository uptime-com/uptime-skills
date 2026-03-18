---
name: incident-triage
description: >-
  This skill should be used when the user asks to "check alerts",
  "investigate an outage", "why is my site down", "triage incidents",
  "what's alerting", "show current outages", "diagnose downtime",
  or needs to investigate monitoring alerts and outages.
  Covers alert review, outage analysis, and correlation patterns.
---

# Incident triage — operational knowledge

Workflow patterns for investigating alerts, outages, and service degradation
using the Uptime.com MCP server.

## Alert review workflow

When the user reports alerts or wants to see what's happening:

### Step 1 — Get current alerts

`list_alerts` to see all active alerts. Key fields:

| Field        | Meaning                                             |
|--------------|-----------------------------------------------------|
| `alert_type` | Check type that triggered (HTTP, DNS, SSL, etc.)    |
| `is_up`      | `false` = currently down, `true` = recovered        |
| `created_at` | When the alert fired                                |
| `output`     | Raw check output — the most useful diagnostic field |

### Step 2 — Investigate the check

`get_check` on the alerting check to see:

- Current configuration (is it checking the right thing?)
- Last response time and status
- Whether the check is paused
- Contact groups (is anyone being notified?)

### Step 3 — Get outage details

`list_outages` filtered by check to see the timeline:

- Outage start and end times
- Duration
- Which probe locations detected the failure

## Correlation patterns

Multiple simultaneous alerts often point to a root cause upstream of any
individual check. Common correlation patterns:

### DNS + HTTP failures

| Alerts firing         | Likely root cause                                                 |
|-----------------------|-------------------------------------------------------------------|
| DNS A + HTTP          | DNS resolution failure — HTTP can't connect because DNS is broken |
| DNS NS + DNS A + HTTP | Nameserver failure — cascading into all resolution                |
| DNS MX + SMTP         | DNS-level mail routing failure                                    |

**Triage action**: Check the DNS checks first. If NS is down, that's the root cause.

### SSL + HTTP failures

| Alerts firing                            | Likely root cause                                           |
|------------------------------------------|-------------------------------------------------------------|
| SSL + HTTP (certificate error in output) | Expired or misconfigured certificate                        |
| SSL only (HTTP still passing)            | Certificate issue that browsers warn on but don't block yet |

**Triage action**: Check SSL output for expiry date and chain issues.

### Widespread failures (many checks, many domains)

If checks across multiple unrelated domains fail simultaneously:

- Likely a probe location issue, not a target issue
- Check if all failing checks share the same probe locations
- May indicate a monitoring platform issue rather than a real outage

### Single check failure

If only one check type is failing for a domain while others pass:

- HTTP down + DNS OK + ICMP OK → application-level issue (web server, load balancer)
- ICMP down + everything else down → host/network unreachable
- TCP port check down + HTTP OK → possible firewall change on the specific port

### Upstream provider correlation

**This step is mandatory during every triage**, regardless of whether the user has
set up upstream dependency monitoring. Many outages that appear to be local are
actually caused by major infrastructure provider incidents.

#### Always check major providers

When investigating any outage, proactively check for ongoing incidents at
major infrastructure providers:

- Check existing CloudStatus checks — if the user has upstream dependency
  monitoring configured, their CloudStatus checks will already show provider
  incidents
- If no CloudStatus checks exist, use web search to check for ongoing
  incidents at providers identified via DNS inference (see below)

Key providers to check: Cloudflare, AWS, Google Cloud, Microsoft Azure,
Fastly, Akamai.

#### When to suspect upstream cause

Signals that an outage may be upstream-driven:

- Multiple unrelated domains failing simultaneously
- Failures concentrated at specific probe locations (regional provider outage)
- DNS timeouts across many checks (upstream DNS provider issue)
- SSL/TLS handshake failures across domains (CDN or certificate provider issue)
- Outage timing coincides with a known provider incident

#### DNS-based provider detection

Even without explicit dependency monitoring, you can infer which providers are
involved by examining DNS records from existing checks. See
`references/upstream-dependencies.md` for CNAME, MX, NS, and SPF patterns that
reveal infrastructure providers.

If DNS checks show CNAME → `*.cloudfront.net`, and AWS CloudFront is reporting
an incident, that's your root cause — not the customer's origin server.

#### Reporting upstream correlation

When an upstream incident is found, include it prominently in the triage report:

> **Upstream incident detected**: Cloudflare is reporting degraded performance
> in EU regions (started 14:23 UTC). Your checks failing from EU probe locations
> are likely caused by this. No action needed on your side — monitor Cloudflare's
> status page for resolution.

## Escalation guidelines

### Check escalation rules

Before triaging, note whether alerting checks have escalation rules configured.
Escalations are separate from contact groups — they define what happens when an
alert goes unacknowledged:

- If escalation rules exist, someone may already be notified at a higher level
- If no escalation rules exist and the outage is critical, manually notify
  appropriate stakeholders — the alert may only be going to a first-tier contact group

### Immediate action needed

- HTTP check on production service with `is_up: false` for > 5 minutes
- SSL certificate expiring within 24 hours
- DNS NS failure (cascading impact)
- All checks for a domain failing simultaneously

### Can wait / investigate further

- Single probe location failure (likely probe issue)
- Blacklist alert (check if it's a false positive)
- WHOIS/RDAP alert (registration expiry is usually weeks away)
- Page Speed degradation (performance issue, not outage)

### False positive indicators

- Only 1 of N probe locations reports failure → likely probe issue, not real outage
- Alert fires and clears within one check interval → transient network issue
- Output shows timeout but other check types pass → check-specific timeout, not real downtime

## Communicating findings

When reporting triage results to the user, structure as:

1. **Summary**: How many alerts, how many are active vs resolved
2. **Root cause assessment**: What's most likely causing the alerts (with correlation reasoning)
3. **Impact**: Which services/domains are affected
4. **Recommended action**: What to do next (investigate further, contact hosting, wait for DNS propagation, etc.)