---
name: guide-alerting-patterns
description: >-
  Patterns for designing alert routing, escalation chains, and contact group strategies. Covers notification design for different team sizes, service tiers, and on-call structures. Used by monitoring-planning skill.
---

# Alerting patterns

How to design notification routing so the right people learn about outages at the right time. For contact group creation steps, see the monitoring-planning skill Phase 0. For field-level reference on escalations and contact groups, see `check-types.md`.

## Alert routing principles

1. **Every check needs a contact group.** Checks without contact groups alert silently: the outage is logged but nobody is notified. This is the most common alerting mistake.
2. **Group by responsibility, not by check type.** The team that owns a service should receive all alerts for that service, regardless of whether the alert comes from an HTTP, DNS, or SSL check.
3. **Fewer groups with clear ownership beat many fine-grained groups.** Each contact group adds routing complexity. Start simple, split only when different teams genuinely need different alerts.
4. **Alert fatigue is the real enemy.** An overwhelmed team ignores all alerts, including real outages. Every design decision should consider whether it increases or decreases noise.

## Contact group patterns

### Single team

One team handles all monitoring. The simplest pattern.

| Group name | Members           | Assigned to |
| ---------- | ----------------- | ----------- |
| `ops-team` | All ops engineers | Every check |

When to use: small team (< 10 engineers), single product, no on-call rotation. Split when alert volume makes a single channel noisy.

### Service-owner model

Each service or domain has a dedicated owner team.

| Group name              | Members          | Assigned to                         |
| ----------------------- | ---------------- | ----------------------------------- |
| `example-com-owners`    | Team A engineers | All checks tagged `example.com`     |
| `api-service-owners`    | Team B engineers | All checks tagged `api.example.com` |
| `upstream-dependencies` | Platform team    | All CloudStatus checks              |

When to use: multiple teams, each owning distinct services. The domain tag naturally maps checks to owners. New checks inherit the right contact group when they follow the tag convention.

### Tiered model

Separate groups for different alert urgency levels.

| Group name        | Channels         | Assigned to                                        |
| ----------------- | ---------------- | -------------------------------------------------- |
| `critical-alerts` | SMS + phone call | HTTP checks on revenue services, SSL on production |
| `standard-alerts` | Email            | DNS, WHOIS, Blacklist, Page Speed, internal tools  |
| `security-alerts` | Email + Slack    | Malware, Blacklist for customer-facing domains     |

When to use: when you need different notification channels for different severity levels. Assign based on the impact of the check failing, not the check type itself.

### On-call rotation

Contact group integrated with an on-call schedule.

| Group name       | Members                                   | Assigned to              |
| ---------------- | ----------------------------------------- | ------------------------ |
| `oncall-primary` | Current on-call engineer (rotated weekly) | Critical checks          |
| `oncall-backup`  | Backup on-call                            | Used in escalation chain |
| `team-channel`   | Full team (email or Slack)                | Informational checks     |

When to use: teams with formal on-call rotations. The primary group receives immediate alerts; the backup group is an escalation target.

## Escalation chain design

Escalations are separate from contact groups. Contact groups define who gets the initial alert. Escalations define what happens when nobody acknowledges the alert within a time window.

### Standard escalation pattern

| Step | Delay after alert | Action      | Target                |
| ---- | ----------------- | ----------- | --------------------- |
| 1    | 0 min             | Email + SMS | Primary contact group |
| 2    | 10 min            | Phone call  | Primary contact group |
| 3    | 30 min            | Email + SMS | Backup/manager group  |
| 4    | 60 min            | Phone call  | Backup/manager group  |

### Which checks need escalations

| Service tier                  | Escalation needed | Reasoning                                                     |
| ----------------------------- | ----------------- | ------------------------------------------------------------- |
| Critical (revenue-generating) | Yes               | Unacknowledged outage = revenue loss every minute             |
| Standard (internal tools)     | Optional          | Team email may be sufficient if checked regularly             |
| Low priority (informational)  | No                | Email notification only; no urgency                           |
| Auto-located (SSL, WHOIS)     | No                | Daily checks; certificate expiry is not a 10-minute emergency |

Exception: SSL checks on production domains with short expiry windows (< 7 days remaining) may warrant escalation, since an expired certificate causes immediate user-facing errors.

### Escalation timing guidance

| Delay     | Too aggressive                            | Appropriate for                                 |
| --------- | ----------------------------------------- | ----------------------------------------------- |
| < 5 min   | Engineer hasn't had time to see the alert | Almost never; only fully automated responses    |
| 5-10 min  | Tight but workable for critical services  | Revenue-critical services during business hours |
| 15-30 min | -                                         | Standard production services                    |
| 30-60 min | -                                         | Escalation to management/backup                 |
| > 60 min  | -                                         | Final escalation for persistent outages         |

The first escalation step should give the primary responder enough time to see and acknowledge the alert. For most teams, 10-15 minutes is the minimum reasonable delay.

## Mapping checks to contact groups

### By domain tag

The most common and recommended pattern. All checks tagged with a domain tag are assigned to the team that owns that domain.

Workflow:

1. Create a contact group for the team (e.g., `platform-team`).
2. When creating checks, always include both the domain `tags` and the team's `contact_groups`.
3. The Group check for the domain uses the same contact group.

This works naturally with the monitoring-planning workflow where every check gets a domain tag on creation.

### By check function

Some organizations route specific check types to specialized teams:

| Check function              | Contact group    | Rationale                                   |
| --------------------------- | ---------------- | ------------------------------------------- |
| SSL checks                  | `security-team`  | Security team manages certificate lifecycle |
| Blacklist + Malware         | `security-team`  | Security owns reputation monitoring         |
| DNS checks                  | `dns-ops`        | Dedicated DNS/network operations team       |
| CloudStatus (upstream deps) | `platform-team`  | Platform team coordinates vendor issues     |
| Everything else             | `service-owners` | Service team handles application checks     |

Only use this pattern when teams have genuinely distinct responsibilities. Most organizations are better served by the domain tag pattern.

### Group check aggregation

Group checks reduce noise by consolidating individual check alerts into one signal:

- **Without Group check**: each failing check sends its own alert. If DNS, HTTP, and SSL all fail at once (common during an outage), the team gets 3+ alerts for one incident.
- **With Group check**: configure individual checks to alert silently (or to a low-priority group), and let the Group check alert the primary team.

To implement this pattern:

1. Create individual checks with `contact_groups` set to a low-priority group (or the same team, for audit trail).
2. Create a Group check with tag-based auto-selection.
3. Set the Group check's `contact_groups` to the primary on-call group.
4. Add escalation rules to the Group check, not individual checks.

This gives one alert per domain outage rather than one per failing check.

## Reducing alert noise

### Sensitivity tuning

Sensitivity < 2 on location-based checks is the most common source of false positives. A single probe location can fail due to transient network issues, ISP routing changes, or probe maintenance. Always use sensitivity >= 2.

See the interval and sensitivity matrix in `guide-check-selection.md` for recommended values.

### Maintenance windows

Suppress alerts during known downtime instead of manually pausing checks:

- Define recurring windows (e.g., every Sunday 02:00-04:00 UTC) for scheduled maintenance.
- Checks pause and resume automatically within the window.
- Alerts auto-resolve when a check enters maintenance (`bulk_alert_resolution_on_pause`).
- No risk of forgetting to resume a manually paused check.

Use maintenance windows for predictable, recurring downtime. Use manual pausing only for unplanned, one-off situations.

### Runbook links in notes

Use the `notes` field on checks to help responders act quickly:

```
Runbook: https://wiki.example.com/runbooks/api-down
Escalation: #platform-oncall in Slack
Last known issue: DNS propagation delay (2024-01-15)
```

Good notes reduce mean time to resolution by giving the responder context without searching.

### Group check thresholds

Not every member failure is a full outage. Configure Group check alert conditions based on the service architecture:

| Architecture          | Alert condition  | Reasoning                                          |
| --------------------- | ---------------- | -------------------------------------------------- |
| Single server         | Any member down  | Any failure is a real outage                       |
| Load-balanced cluster | N-of-M threshold | One backend down is degraded, not fully down       |
| Active-passive        | All members down | Primary failure with failover is expected behavior |
| Microservices         | Any member down  | Each component is independently critical           |

## Common alerting mistakes

| Mistake                               | Impact                                               | Fix                                                    |
| ------------------------------------- | ---------------------------------------------------- | ------------------------------------------------------ |
| No contact group on checks            | Outages logged but nobody notified                   | Assign a contact group to every check on creation      |
| Single "everyone" group on all checks | Alert fatigue, diffusion of responsibility           | Split by service ownership or severity tier            |
| No escalation on critical checks      | Unacknowledged alerts go nowhere                     | Add escalation chain with increasing urgency           |
| Escalation too aggressive (< 5 min)   | Phone calls before anyone can investigate            | Set first escalation at 10-15 min minimum              |
| Sensitivity = 1 on production checks  | False positives from transient probe issues          | Use sensitivity >= 2 with 3+ locations                 |
| No maintenance windows                | Alerts fire during planned downtime                  | Configure recurring maintenance windows                |
| Alerting not tested                   | Assumed working until a real outage reveals it isn't | Create a test check that will fail to verify the chain |
| Individual alerts without Group check | N alerts for one incident (DNS + HTTP + SSL)         | Use Group check aggregation pattern                    |
