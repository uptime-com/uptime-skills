---
name: check-types
description: >-
  Reference document for check type parameters, constraints, and location requirements as the Uptime.com MCP server exposes them. Used by the understanding-check-types skill.
---

# Check type matrix

Every check type has a `create_<type>_check` tool and an `update_<type>_check` tool. An update tool takes the check's `id` and the same fields as its create tool, and changes only the fields provided. Every create tool requires `name` and `contact_groups` (contact group names from `list_contacts`), and takes `tags` (tag names; create them first with `create_tag`).

## Location-based checks

These run from the probe locations passed in `locations` (identifiers from `list_locations`) and take `sensitivity`, the number of locations that must confirm an outage. Give 3–5 locations and sensitivity ≥ 2. `address`, `interval` and `locations` are required unless noted.

| Type | Tool suffix | Also required | Optional                                                                            | Typical interval |
| ---- | ----------- | ------------- | ----------------------------------------------------------------------------------- | ---------------- |
| HTTP | `http`      |               | `expect_string`, `headers`, `send_string`, `username`, `password`, `port`, `notes`  | 1–5 min          |
| DNS  | `dns`       |               | `dns_record_type`, `dns_server`, `expect_string`                                    | 5–10 min         |
| ICMP | `icmp`      |               |                                                                                     | 1–5 min          |
| TCP  | `tcp`       | `port`        | `send_string`, `expect_string`                                                      | 1–5 min          |
| UDP  | `udp`       | `port`        | `send_string`, `expect_string`                                                      | 5–10 min         |
| SMTP | `smtp`      |               | `port` (25, or 465/587), `encryption` (`SSL` or `STARTTLS`), `username`, `password` | 5–10 min         |
| IMAP | `imap`      |               | `port` (143, or 993), `encryption`, `expect_string`                                 | 5–10 min         |
| POP  | `pop`       |               | `port` (110, or 995), `encryption`, `expect_string`                                 | 5–10 min         |
| SSH  | `ssh`       |               | `port` (22)                                                                         | 5–10 min         |
| NTP  | `ntp`       |               | `port` (123), `threshold` (maximum time offset, ms)                                 | 10–30 min        |

- HTTP `address` is a full URL including the scheme (`https://example.com`). `headers` is one `Header: Value` per line.
- UDP needs `send_string` and `expect_string` to get and verify a response from most services.

### DNS record types

| Record type | Target                             | Use case                                  |
| ----------- | ---------------------------------- | ----------------------------------------- |
| A / AAAA    | subdomain (e.g. `www.example.com`) | Host resolution                           |
| CNAME       | subdomain                          | Alias resolution                          |
| MX          | parent domain (e.g. `example.com`) | Mail routing — zone-level record          |
| NS          | parent domain                      | Nameserver delegation — zone-level record |
| TXT         | domain                             | SPF, DKIM, DMARC verification             |

## Scripted checks

Marketed as **Synthetic Monitoring**. Neither takes an `address`: the URLs live in the script.

| Type        | Tool suffix   | Required              | Optional                                                                                    |
| ----------- | ------------- | --------------------- | ------------------------------------------------------------------------------------------- |
| Transaction | `transaction` | `script`, `locations` | `interval`, `threshold` (timeout, s, default 30), `num_retries`, `sensitivity`, `is_paused` |
| API         | `api`         | `script`, `locations` | the Transaction fields, plus `use_ip_version` (`IPV4` or `IPV6`)                            |

Script syntax lives in the `transaction-scripting` and `api-scripting` skills. Transaction checks have an open location problem; see `transaction-scripting`.

## Page Speed

Tool suffix `pagespeed`. Required: `address` (URL), `interval`, `locations`. Optional: `emulated_device` (e.g. `desktop`, `mobile`), `connection_throttling`, `uptime_grade_threshold` (minimum Lighthouse grade).

- Exactly **1 location**, and it must be a `Dedicated-*` location (e.g. `Dedicated-United Kingdom-London`); only dedicated nodes run Lighthouse.
- Minimum interval **1440** minutes. The tool defaults `interval` to 5, so always pass 1440 or more.

## CloudStatus

CloudStatus (marketed as **Third-party monitoring**) follows a cloud provider's official status feed. Tool suffix `cloudstatus`. Required: `service_name`, `locations`. Optional: `is_paused`. No `address`, `interval` or `sensitivity`.

`service_name` must be an exact service name: find the provider with `list_cloudstatus_providers`, then the service with `search_cloudstatus_services` (e.g. `AWS - Amazon Elastic Compute Cloud`).

## Auto-located checks

The server assigns locations; these take no `locations`, `sensitivity` or `interval`. Required: `address`.

| Type      | Tool suffix | Also required   | Optional                                                                                                 |
| --------- | ----------- | --------------- | -------------------------------------------------------------------------------------------------------- |
| SSL       | `ssl`       |                 | `threshold` (days before expiry to alert), `port` (443), `protocol` (`HTTPS`, `IMAPS`, `POP3S`, `SMTPS`) |
| Blacklist | `blacklist` |                 |                                                                                                          |
| Malware   | `malware`   |                 |                                                                                                          |
| WHOIS     | `whois`     | `expect_string` | `threshold` (days before registration expiry to alert)                                                   |
| RDAP      | `rdap`      | `expect_string` | `threshold`                                                                                              |

- Blacklist (DNS blacklists) and Malware (Google Safe Browsing) are marketed together as **Domain Health**.
- WHOIS and RDAP take the registered domain, never a subdomain, with `expect_string` set to that domain. RDAP is the redundancy for unreliable WHOIS servers.

Each measures a single global fact (certificate expiry, registration, listing status), so location is not a parameter. Consequence: no confirmation quorum. A single-probe blip can surface as a one-off alert, and no setting tunes it away. Where multi-location confirmation of connectivity matters, pair the check with a location-based one (e.g. TCP 443 beside SSL) and treat the auto-located check as its specialized signal.

**SSL caveat**: the check assumes one certificate per host. Split-horizon setups (different certificates per region) are not supported.

## Push and passive checks

No probes run against these, so they take no `locations` or `sensitivity`.

| Type      | Tool suffix   | Fields                                 | Notes                                                                                                                  |
| --------- | ------------- | -------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| Heartbeat | `heartbeat`   | `interval` (required)                  | Marketed as **Heartbeat monitoring**. The service pings a generated URL; alerts when no ping arrives within `interval` |
| Webhook   | `webhook`     |                                        | External systems post status to a generated URL                                                                        |
| RUM, RUM2 | `rum`, `rum2` | `address` (required), `threshold` (ms) | Marketed as **Real User Monitoring**. Needs a JavaScript snippet on the site                                           |

## Group checks

Tool suffix `group`. Aggregates other checks; no `address`, `locations` or `interval`.

| Field            | Meaning                                                                     |
| ---------------- | --------------------------------------------------------------------------- |
| `check_tags`     | Tag names whose checks belong to the group; checks tagged later join it too |
| `services`       | Check names to include explicitly                                           |
| `down_condition` | `ANY` or `ALL` members down                                                 |

Tag-based membership is why every check gets its domain tag on creation.

## Not settable through the MCP server

These are real Uptime.com features with no MCP tool field. Recommend them where they help, and tell the user to configure them in the Uptime.com web app:

- Escalations (who is notified when an alert goes unacknowledged)
- Maintenance windows
- SLA targets: uptime threshold, response time threshold
- HTTP method, expected status code, `expect_string` match mode (regex), proxy, certificate verification, minimum TLS version
- SSL certificate matching: fingerprint, issuer, alternative names
- IP version, except on API checks
- Pausing, except on API, Transaction and CloudStatus checks (`is_paused`)
- Contact methods other than email and SMS (phone, integrations)
- Group check N-of-M conditions and response time settings

`get_check` reports type, address, port, status, last response time, interval, sensitivity, locations, tags and notes. It does not report contact groups, escalations, check-specific fields or scripts.
