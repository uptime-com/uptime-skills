---
name: incident-triage
description: >-
  This skill should be used when the user asks to "check alerts", "investigate an outage", "why is my site down", "triage incidents", "what's alerting", "show current outages", "diagnose downtime", or needs to investigate monitoring alerts and outages. Covers alert review, outage analysis, upstream provider correlation, and escalation guidelines.
---

# Incident triage

Workflow for investigating alerts, outages, and service degradation.

## Alert review workflow

### Step 1: get current alerts

`list_alerts` to see all active alerts. Key fields:

| Field        | Meaning                                            |
| ------------ | -------------------------------------------------- |
| `alert_type` | Check type that triggered (HTTP, DNS, SSL, etc.)   |
| `is_up`      | `false` = currently down, `true` = recovered       |
| `created_at` | When the alert fired                               |
| `output`     | Raw check output, the most useful diagnostic field |

### Step 2: investigate the check

`get_check` on the alerting check to see:

- Current configuration (is it checking the right thing?)
- Last response time and status
- Whether the check is paused
- Contact groups (is anyone being notified?)

### Step 3: get outage details

`list_outages` filtered by check to see the timeline:

- Outage start and end times
- Duration
- Which probe locations detected the failure

## Correlation patterns

Multiple simultaneous alerts often point to a root cause upstream of any individual check.

### DNS + HTTP failures

| Alerts firing         | Likely root cause                                                |
| --------------------- | ---------------------------------------------------------------- |
| DNS A + HTTP          | DNS resolution failure; HTTP can't connect because DNS is broken |
| DNS NS + DNS A + HTTP | Nameserver failure, cascading into all resolution                |
| DNS MX + SMTP         | DNS-level mail routing failure                                   |

**Triage action**: check DNS checks first. If NS is down, that's the root cause.

### SSL + HTTP failures

| Alerts firing                            | Likely root cause                                      |
| ---------------------------------------- | ------------------------------------------------------ |
| SSL + HTTP (certificate error in output) | Expired or misconfigured certificate                   |
| SSL only (HTTP still passing)            | Certificate issue browsers warn on but don't block yet |

### Widespread failures (many checks, many domains)

If checks across multiple unrelated domains fail simultaneously:

- Likely a probe location issue, not a target issue
- Check if all failing checks share the same probe locations
- May indicate a monitoring platform issue rather than a real outage

### Single check failure

- HTTP down + DNS OK + ICMP OK: application-level issue (web server, load balancer)
- ICMP down + everything else down: host/network unreachable
- TCP port check down + HTTP OK: possible firewall change on the specific port

## Upstream provider correlation

**This step is mandatory during every triage.** Many outages that appear local are actually caused by infrastructure provider incidents.

### Always check major providers

- Check existing CloudStatus checks; if upstream dependency monitoring is configured, they will already show provider incidents
- If no CloudStatus checks exist, use web search to check for ongoing incidents at providers identified via DNS inference

Key providers to check: Cloudflare, AWS, Google Cloud, Microsoft Azure, Fastly, Akamai.

### When to suspect upstream cause

- Multiple unrelated domains failing simultaneously
- Failures concentrated at specific probe locations (regional provider outage)
- DNS timeouts across many checks (upstream DNS provider issue)
- SSL/TLS handshake failures across domains (CDN or certificate provider issue)
- Outage timing coincides with a known provider incident

### DNS-based provider detection

Even without explicit dependency monitoring, infer providers from DNS records:

| Record type | Pattern                | Provider         |
| ----------- | ---------------------- | ---------------- |
| CNAME       | `*.cloudfront.net`     | AWS CloudFront   |
| CNAME       | `*.cdn.cloudflare.net` | Cloudflare CDN   |
| CNAME       | `*.fastly.net`         | Fastly           |
| NS          | `*.cloudflare.com`     | Cloudflare DNS   |
| NS          | `awsdns-*`             | AWS Route 53     |
| MX          | `*.google.com`         | Google Workspace |
| MX          | `*.outlook.com`        | Microsoft 365    |

If DNS checks show CNAME pointing to `*.cloudfront.net` and AWS CloudFront is reporting an incident, that's the root cause.

### Reporting upstream correlation

> **Upstream incident detected**: Cloudflare is reporting degraded performance in EU regions (started 14:23 UTC). Your checks failing from EU probe locations are likely caused by this. No action needed on your side; monitor Cloudflare's status page for resolution.

## Escalation guidelines

### Check escalation rules

Before triaging, note whether alerting checks have escalation rules. If escalation rules exist, someone may already be notified. If not and the outage is critical, manually notify appropriate stakeholders.

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

- Only 1 of N probe locations reports failure: likely probe issue
- Alert fires and clears within one check interval: transient network issue
- Output shows timeout but other check types pass: check-specific timeout

## Communicating findings

1. **Summary**: how many alerts, how many active vs resolved
2. **Root cause assessment**: what's most likely causing the alerts
3. **Impact**: which services/domains are affected
4. **Recommended action**: what to do next
