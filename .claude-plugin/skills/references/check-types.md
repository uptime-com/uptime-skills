---
name: check-types
description: >-
  Reference document for check type parameters, constraints, and location
  requirements. Used by monitoring-setup and monitoring-audit skills.
---

# Check type matrix

Complete reference for all check types supported by the Uptime.com MCP server.

## Location-based checks

These checks run from explicit probe locations. Specify 3–5 locations and set
sensitivity ≥ 2 to avoid false positives.

| Type        | Required fields                                    | Key constraints                          | Typical interval |
|-------------|----------------------------------------------------|------------------------------------------|------------------|
| HTTP        | `address` (URL), `locations`                       | URL must include protocol (https://)     | 1–5 min          |
| DNS         | `address` (domain), `dns_record_type`, `locations` | See record types below                   | 5–10 min         |
| ICMP (Ping) | `address` (hostname/IP), `locations`               | Target must respond to ICMP              | 1–5 min          |
| TCP         | `address` (hostname), `port`, `locations`          | Requires explicit port number            | 1–5 min          |
| UDP         | `address` (hostname), `port`, `locations`          | Requires explicit port and expect_string | 5–10 min         |
| SMTP        | `address` (mail server), `port`, `locations`       | Default port 25; use 587 for submission  | 5–10 min         |
| IMAP        | `address` (mail server), `port`, `locations`       | Default port 143 (993 for TLS)           | 5–10 min         |
| POP         | `address` (mail server), `port`, `locations`       | Default port 110 (995 for TLS)           | 5–10 min         |
| SSH         | `address` (hostname), `port`, `locations`          | Default port 22                          | 5–10 min         |
| NTP         | `address` (NTP server), `locations`                | Validates time synchronization           | 10–30 min        |

### DNS record types

| Record type | Target                             | Use case                                  |
|-------------|------------------------------------|-------------------------------------------|
| A / AAAA    | subdomain (e.g. `www.example.com`) | Host resolution                           |
| CNAME       | subdomain                          | Alias resolution                          |
| MX          | parent domain (e.g. `example.com`) | Mail routing — zone-level record          |
| NS          | parent domain                      | Nameserver delegation — zone-level record |
| TXT         | domain                             | SPF, DKIM, DMARC verification             |
| SOA         | parent domain                      | Zone authority                            |

## Auto-located checks

These checks have locations assigned by the server. Do NOT pass `locations` — it
will cause a validation error.

| Type      | Required fields                                | Key constraints                                               | Typical interval |
|-----------|------------------------------------------------|---------------------------------------------------------------|------------------|
| SSL       | `address` (hostname)                           | Monitors certificate expiry and chain. See SSL fields below   | 60–1440 min      |
| Blacklist | `address` (hostname/IP)                        | Marketed as **Domain Health** — checks DNS blacklists (RBLs)  | 1440 min         |
| Malware   | `address` (URL)                                | Marketed as **Domain Health** — scans for malware indicators  | 1440 min         |
| WHOIS     | `address` (registered domain), `expect_string` | Domain only — no subdomains. Set expect_string to domain name | 1440 min         |
| RDAP      | `address` (registered domain), `expect_string` | Domain only — no subdomains. Redundancy for WHOIS             | 1440 min         |

## Constrained checks

| Type       | Required fields                          | Key constraints                                               | Typical interval |
|------------|------------------------------------------|---------------------------------------------------------------|------------------|
| Page Speed | `address` (URL), `locations` (exactly 1) | Must use `Dedicated-*` location prefix. Min interval 1440 min | 1440 min         |

## Cloud status checks

| Type | Required fields | Key constraints | Typical interval |
|------|----------------|-----------------|------------------|
| CloudStatus | `cloudstatusconfig.service_name` | Must match an exact component name. Use MCP tools to discover available services | Derived from feed |

CloudStatus (marketed as **Third-party monitoring**) monitors a cloud provider's
official status page natively within Uptime.com. No `address` or `locations`
needed — the service is identified by `service_name`. Use for upstream dependency
monitoring.

## Aggregate checks

| Type  | Required fields | Key constraints | Typical interval |
|-------|----------------|-----------------|------------------|
| Group | Checks (manual selection or auto-select by tags) | No address — aggregates other checks. Alert conditions: any/all down | Derived from member checks |

### Group check details

Group checks combine multiple individual checks into one logical unit:

- **Manual selection**: pick specific checks to include
- **Auto-select by tags**: dynamically include all checks with matching tags
  (new checks tagged later are automatically included)
- **Alert conditions**: configure when the group is considered down
  (any member down, all members down, or N-of-M threshold)
- **Uptime % calculation**: group-level SLA computed from members
- **Response time**: copy from a single check or average across a check type

Tag-based auto-selection reinforces the "tag first" pattern — when checks are
properly tagged on creation, group checks automatically include them.

## Synthetic / advanced checks

Marketed as **Synthetic Monitoring** — these checks simulate user interactions
rather than simple endpoint polling.

| Type | Required fields | Key constraints | Typical interval |
|------|----------------|-----------------|------------------|
| Transaction | `address` (URL), script steps | Multi-step browser interaction. See `references/scripting-txn.md` for full scripting reference. Default IP version differs (not IPv4) | 5–30 min |
| API | `address` (URL), request config | Multi-step API calls with assertions. See `references/scripting-api.md` for full scripting reference. Default IP version differs | 5–30 min |
| RUM | Site tag (JavaScript snippet) | Marketed as **Real User Monitoring** — no probe locations, measures actual visitor performance | Continuous |
| Custom (Heartbeat) | Heartbeat URL (generated) | Marketed as **Heartbeat monitoring** — process pushes to Uptime.com, alerts if heartbeat stops | Configurable timeout |

**RUM-specific field**: `max_load_time` — alert if average page load exceeds this
threshold (seconds). Only applicable to RUM checks.

## Field reference

### Common optional fields (all check types)

| Field            | Description                                                         |
|------------------|---------------------------------------------------------------------|
| `name`           | Display name (auto-generated from address if omitted)               |
| `tags`           | Array of tag names for grouping. Also drives Group check auto-selection |
| `contact_groups` | Array of contact group names                                        |
| `interval`       | Minutes between checks                                              |
| `sensitivity`    | Number of confirming locations (≥ 2 recommended for location-based) |
| `timeout`        | Seconds before check times out. Detects firewalls, overloaded servers |
| `is_paused`      | Create in paused state                                              |
| `notes`          | Free-text notes visible in dashboard                                |
| `use_ip_version` | `IPV4`, `IPV6`, or `ANY`. Default IPv4 (Transaction/API default differently) |
| `include_in_global_metrics` | Include check in dashboards and SLA calculations          |
| `escalations`    | Alert escalation rules (separate from contact groups — see below)   |
| `maintenance`    | Scheduled maintenance windows (recurring pauses)                    |
| `uptime_sla_threshold` | Target uptime percentage for SLA tracking                      |
| `response_time_sla` | Target response time (ms) for SLA tracking                       |

### Escalations vs contact groups

Contact groups define *who* gets notified. Escalations define *when* and *how*
notifications intensify:

- Contact groups receive the initial alert
- Escalation rules trigger if the alert is not acknowledged within a time window
- Escalations can notify different contacts or use different channels (e.g. phone call after SMS)

### Maintenance windows

Scheduled maintenance is preferred over manual pausing:

- Define recurring windows (e.g. every Sunday 02:00–04:00)
- Checks pause automatically during the window
- No manual intervention needed — no risk of forgetting to resume
- Alerts auto-resolve when a check enters maintenance (`bulk_alert_resolution_on_pause`)

### HTTP-specific fields

| Field                   | Description                                                           |
|-------------------------|-----------------------------------------------------------------------|
| `expect_string`         | String that must appear in response body                              |
| `expect_string_type`    | Matching mode: `STRING` (contains), `REGEX`, or exact match           |
| `http_method`           | `GET`, `POST`, `HEAD`, etc.                                          |
| `send_string`           | POST body data (URL-encoded for HTTP, raw for TCP/UDP)                |
| `expect_status_code`    | HTTP status codes considered successful (default: 2xx). Customize for APIs returning 3xx, etc. |
| `headers`               | Custom request headers                                                |
| `username` / `password` | Basic auth credentials. Use unprivileged test accounts                |
| `proxy_url`             | Route check through a proxy server                                    |
| `verify_ssl_cert`       | Toggle SSL/TLS certificate validation on HTTPS checks                 |
| `min_tls_version`       | Enforce minimum TLS version (e.g. TLS 1.2)                           |
| `block_urls`            | URLs to exclude from loading                                          |
| `emulate_device`        | Simulate specific device/browser                                      |
| `connection_throttling` | Simulate network conditions (3G, slow connection, etc.)               |

### DNS-specific fields

| Field       | Description                                          |
|-------------|------------------------------------------------------|
| `dns_record_type` | Record type to query (A, AAAA, CNAME, MX, NS, TXT, SOA) |
| `dns_server` | Custom DNS resolver (default: system resolver)       |
| `expect_string` | Expected record value in response                  |
| `force_host_resolution` | Pin IP address resolution                   |

### SSL-specific fields

| Field                     | Description                                                |
|---------------------------|------------------------------------------------------------|
| `before_expiry`           | Alert N days before certificate expires                    |
| `min_tls_version`         | Enforce minimum SSL/TLS version                            |
| `match_cert_fingerprint`  | Verify SHA-1 certificate fingerprint (pin specific cert)   |
| `match_issuer_name`       | Verify expected Certificate Authority                      |
| `match_additional_names`  | Verify Subject Alternative Names (SANs)                    |
| `validate_crl_url`        | CRL validation endpoint URL                                |

### SMTP-specific fields

| Field        | Description                                       |
|--------------|---------------------------------------------------|
| `encryption` | Enable TLS encryption for SMTP connection         |
| `username` / `password` | SMTP auth credentials                  |