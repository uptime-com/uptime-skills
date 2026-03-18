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

## False positives

False positives are alerts that fire when the service is actually healthy.

### Indicators

- Only 1 of N probe locations reports failure: likely a probe or regional network issue
- Alert fires and clears within one check interval: transient network blip
- Output shows timeout but other check types for the same target pass: check-specific timeout, not real downtime
- Repeated short-lived alerts from the same location: that location may have connectivity issues

### Tuning recommendations

If false positives are frequent, recommend adjusting the monitoring configuration:

| Problem                         | Fix                                                                  |
| ------------------------------- | -------------------------------------------------------------------- |
| Single-location flapping        | Increase sensitivity to >= 2 (require multiple locations to confirm) |
| Probe location unreliable       | Replace with a different location in the same region                 |
| Timeout-based false alerts      | Increase `timeout` value to accommodate normal latency variance      |
| Interval too aggressive         | Increase interval for non-critical checks (e.g. 1 min -> 5 min)      |
| All checks share same locations | Diversify locations across regions to reduce correlated false alerts |

## False negatives

False negatives are real outages that monitoring fails to detect. These are more dangerous than false positives because they create a false sense of security.

### Indicators

- Users report downtime but no alerts fired
- Outage visible in external tools (e.g. Down Detector) but not in Uptime.com
- Post-incident review reveals the service was down for minutes/hours without alerting
- CloudStatus shows upstream provider incident but no corresponding alerts on dependent checks

### Common causes

| Cause                    | Why it happens                                                                 | Fix                                                             |
| ------------------------ | ------------------------------------------------------------------------------ | --------------------------------------------------------------- |
| Missing check types      | Only HTTP monitored, but DNS was the actual failure point                      | Add DNS, SSL, ICMP checks for comprehensive coverage            |
| Wrong endpoint monitored | Health endpoint returns 200 even when the app is broken                        | Monitor a functional endpoint that exercises the real code path |
| `expect_string` not set  | HTTP check passes on any 200 response, even error pages                        | Add `expect_string` to verify response content                  |
| Too few locations        | All probes are in one region; regional outage goes undetected from that region | Use 3-5 locations across multiple continents                    |
| Check is paused          | Forgotten manual pause or stale maintenance window                             | Review paused checks; convert to scheduled maintenance windows  |
| No upstream monitoring   | Provider outage causes degradation but no check covers the dependency          | Add CloudStatus checks for critical upstream providers          |

### When false negatives are frequent

Frequent false negatives indicate the monitoring strategy needs a broader review. Recommend invoking the `monitoring-optimization` skill to run a full audit: gap analysis, configuration review, and upstream dependency check.

## Ignoring alerts

Use `ignore_alert` to exclude a confirmed false positive from outage calculations. This is important for accurate SLA reporting: ignored alerts don't count as downtime.

When to ignore:

- Confirmed false positive (probe issue, not real outage)
- Alert caused by planned maintenance that wasn't covered by a maintenance window
- One-time transient error that doesn't reflect real availability

When NOT to ignore:

- Real outages, even brief ones: they should be reflected in uptime stats
- Alerts you haven't investigated yet: investigate first, ignore after

Always confirm with the user before ignoring alerts, as it affects SLA metrics.

## Communicating findings

1. **Summary**: how many alerts, how many active vs resolved
2. **Root cause assessment**: what's most likely causing the alerts
3. **Impact**: which services/domains are affected
4. **Recommended action**: what to do next
