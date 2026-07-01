---
name: guide-check-selection
description: >-
  Problem-oriented guide for selecting check types and configuring them. Decision trees by detection goal, concrete configuration recipes with intervals, sensitivity, and expect_string patterns for common scenarios. Used by monitoring-planning skill.
---

# Check selection and configuration guide

How to choose the right check type for your monitoring goal and configure it effectively. For API-level field reference, see `check-types.md`. For domain-pattern checklists, see `checklist-domain-monitoring.md`.

## Detection goal decision tree

Start from what you need to detect, not which check type to use.

| Detection goal                    | Check type   | Key configuration                                          | Why this check                                               |
| --------------------------------- | ------------ | ---------------------------------------------------------- | ------------------------------------------------------------ |
| Website is down                   | HTTP         | `address`: full URL, `expect_string`: page content marker  | Validates both connectivity and correct response content     |
| API endpoint not responding       | HTTP         | `address`: health endpoint, `expect_string`: JSON field    | Health endpoints return structured status                    |
| Network-level connectivity loss   | TCP          | `address`: hostname, `port`: service port                  | Catches firewall/routing issues faster than HTTP             |
| Host unreachable                  | ICMP         | `address`: hostname or IP                                  | Lowest overhead; detects network-layer failures              |
| SSL certificate expiring          | SSL          | `address`: hostname, `before_expiry`: days warning         | Dedicated cert chain validation with expiry countdown        |
| DNS resolution failing            | DNS (A)      | `address`: subdomain, `dns_record_type`: A                 | Detects resolution failures for the specific service         |
| Nameserver delegation broken      | DNS (NS)     | `address`: parent domain, `dns_record_type`: NS            | NS failure cascades into all DNS; earliest signal            |
| Mail routing broken               | DNS (MX)     | `address`: parent domain, `dns_record_type`: MX            | MX record loss breaks inbound email                          |
| Mail server not accepting mail    | SMTP         | `address`: mail server, `port`: 25 or 587                  | Tests actual SMTP handshake                                  |
| Mailbox access broken             | IMAP or POP  | `address`: mail server, `port`: 993 (IMAP) or 995 (POP)    | Tests mailbox protocol connectivity                          |
| Domain registration expiring      | WHOIS + RDAP | `address`: registered domain, `expect_string`: domain name | Both for redundancy; WHOIS servers can be unreliable         |
| IP/domain blacklisted             | Blacklist    | `address`: hostname or IP                                  | Catches DNS blacklist (RBL) entries affecting deliverability |
| Site compromised with malware     | Malware      | `address`: URL                                             | Scans for known malware indicators                           |
| Page load performance degradation | Page Speed   | `address`: URL, 1 `Dedicated-*` location, interval >= 1440 | Lighthouse-based real performance measurement                |
| Upstream provider outage          | CloudStatus  | `cloudstatusconfig.service_name`: provider component       | Native status feed parsing with UP/DOWN/MAINTENANCE mapping  |
| Multi-step user flow broken       | Transaction  | `address`: start URL, scripted steps                       | Browser-based interaction simulating real user journeys      |
| API workflow broken               | API          | `address`: base URL, scripted request steps                | Multi-step API call sequences with assertions                |
| Aggregate service health          | Group        | Tag-based auto-selection with domain tag                   | Combines multiple checks into one logical health indicator   |
| Process/cron stopped running      | Custom       | Heartbeat URL (generated), configurable timeout            | Push-based; the monitored process reports in, not polled     |
| Real user experience degradation  | RUM          | Site tag (JavaScript snippet), `max_load_time` threshold   | Measures actual visitor performance, not synthetic probes    |

When two check types could work, prefer the one that catches the failure closer to the user's experience. HTTP catches more than TCP; TCP catches more than ICMP. Use the simpler check as a complement, not a replacement.

## Configuration recipes

Concrete configurations for common scenarios. Values are recommendations; adjust based on your SLA requirements.

### High-traffic production website

Revenue-generating public site where minutes of downtime matter.

| Check      | address               | interval | sensitivity | locations | Other key fields                                           |
| ---------- | --------------------- | -------- | ----------- | --------- | ---------------------------------------------------------- |
| HTTP       | `https://example.com` | 1        | 3           | 5         | `expect_string`: content marker, `expect_status_code`: 200 |
| SSL        | `example.com`         | 60       | -           | -         | `before_expiry`: 30                                        |
| DNS A      | `www.example.com`     | 5        | 2           | 3         | `dns_record_type`: A                                       |
| DNS NS     | `example.com`         | 5        | 2           | 3         | `dns_record_type`: NS                                      |
| ICMP       | `example.com`         | 3        | 2           | 3         | -                                                          |
| Page Speed | `https://example.com` | 1440     | -           | 1         | Must use `Dedicated-*` location                            |
| Blacklist  | `example.com`         | 1440     | -           | -         | -                                                          |
| WHOIS      | `example.com`         | 1440     | -           | -         | `expect_string`: `example.com`                             |
| RDAP       | `example.com`         | 1440     | -           | -         | `expect_string`: `example.com`                             |

Why sensitivity 3 on HTTP: with 5 locations and sensitivity 3, a transient issue at one probe won't trigger an alert. Three independent confirmations provide high confidence the outage is real.

### API service

Backend API consumed by mobile apps, SPAs, or third-party integrations.

| Check  | address                          | interval | sensitivity | locations | Other key fields                                            |
| ------ | -------------------------------- | -------- | ----------- | --------- | ----------------------------------------------------------- |
| HTTP   | `https://api.example.com/health` | 1        | 2           | 3-5       | `expect_string`: `"status":"ok"`, `expect_status_code`: 200 |
| HTTP   | `https://api.example.com/v1/...` | 5        | 2           | 3         | Representative endpoint; `expect_status_code` per API spec  |
| TCP    | `api.example.com`                | 1        | 2           | 3         | `port`: 443                                                 |
| SSL    | `api.example.com`                | 60       | -           | -         | `before_expiry`: 30; API clients reject expired certs       |
| DNS A  | `api.example.com`                | 5        | 2           | 3         | `dns_record_type`: A                                        |
| DNS NS | `example.com`                    | 5        | 2           | 3         | `dns_record_type`: NS                                       |

Why TCP alongside HTTP: TCP detects port-level and network-level failures faster than HTTP. If the load balancer drops connections, TCP fails immediately while HTTP waits for timeout.

### Marketing / brochure site

Low-traffic informational site where 5-minute detection delay is acceptable.

| Check      | address               | interval | sensitivity | locations | Other key fields                     |
| ---------- | --------------------- | -------- | ----------- | --------- | ------------------------------------ |
| HTTP       | `https://example.com` | 5        | 2           | 3         | `expect_string`: brand name or title |
| SSL        | `example.com`         | 1440     | -           | -         | `before_expiry`: 14                  |
| DNS A      | `www.example.com`     | 10       | 2           | 3         | `dns_record_type`: A                 |
| Page Speed | `https://example.com` | 1440     | -           | 1         | Must use `Dedicated-*` location      |
| WHOIS      | `example.com`         | 1440     | -           | -         | `expect_string`: `example.com`       |

Why lower sensitivity and fewer locations: false positives are more disruptive than a few extra minutes of detection time for non-critical sites.

### Internal tool / admin panel

Internal-facing application with limited user base.

| Check | address                     | interval | sensitivity | locations | Other key fields                   |
| ----- | --------------------------- | -------- | ----------- | --------- | ---------------------------------- |
| HTTP  | `https://admin.example.com` | 5        | 2           | 3         | `expect_string`: login page marker |
| SSL   | `admin.example.com`         | 1440     | -           | -         | `before_expiry`: 14                |
| DNS A | `admin.example.com`         | 10       | 2           | 3         | `dns_record_type`: A               |

Internal tools often have restricted network access. Verify probe locations can reach the target before configuring, or use a Custom (Heartbeat) check if the tool is behind a firewall.

### Mail infrastructure

Dedicated email system with SMTP, IMAP/POP access.

| Check     | address            | interval | sensitivity | locations | Other key fields                                 |
| --------- | ------------------ | -------- | ----------- | --------- | ------------------------------------------------ |
| SMTP      | `mail.example.com` | 5        | 2           | 3         | `port`: 25; add port 587 as separate check       |
| IMAP      | `mail.example.com` | 5        | 2           | 3         | `port`: 993, `encryption`: TLS                   |
| DNS MX    | `example.com`      | 5        | 2           | 3         | `dns_record_type`: MX                            |
| DNS A     | `mail.example.com` | 5        | 2           | 3         | `dns_record_type`: A                             |
| DNS NS    | `example.com`      | 5        | 2           | 3         | `dns_record_type`: NS                            |
| SSL       | `mail.example.com` | 60       | -           | -         | `before_expiry`: 30                              |
| Blacklist | `mail.example.com` | 1440     | -           | -         | Critical for mail servers; blacklisting = bounce |

Why SMTP on both ports: port 25 handles server-to-server delivery, port 587 handles client submission. They can fail independently.

## expect_string patterns

Content verification prevents "check is green but site is broken" scenarios. An HTTP 200 with an error page, a cached stale response, or a load balancer default page all return success status codes.

### HTTP checks

**Good patterns** (stable content that indicates the real application is serving):

| Scenario               | expect_string example       | Why it works                        |
| ---------------------- | --------------------------- | ----------------------------------- |
| Homepage               | `<title>My Company</title>` | Title tag is stable, unique to site |
| Health endpoint (JSON) | `"status":"ok"`             | Structured health response          |
| Health endpoint (text) | `healthy`                   | Simple text health indicator        |
| Login page             | `name="password"`           | Password field proves login form    |
| API docs               | `swagger` or `openapi`      | Docs framework marker               |
| Specific text on page  | `Welcome to Example`        | Brand-specific content              |

**Bad patterns** (content that changes, causing false alerts):

| Pattern              | Problem                                          |
| -------------------- | ------------------------------------------------ |
| Timestamps           | Changes every request                            |
| Session tokens/nonce | Unique per request                               |
| User counts/stats    | Dynamic numbers                                  |
| "Last updated" dates | Changes on content updates                       |
| Full HTML blocks     | Whitespace or formatting changes break the match |
| Empty string         | Matches everything, verifies nothing             |

### DNS checks

| Record type | expect_string usage           | Example            |
| ----------- | ----------------------------- | ------------------ |
| A           | Expected IP address           | `93.184.216.34`    |
| MX          | Expected mail server hostname | `mail.example.com` |
| NS          | Expected nameserver           | `ns1.example.com`  |
| TXT         | SPF or verification string    | `v=spf1`           |

DNS expect_string is optional but valuable: it catches DNS hijacking or misconfigured record changes, not just resolution failures.

### WHOIS and RDAP

Always set `expect_string` to the registered domain name (e.g., `example.com`). This is required for both check types. It verifies the WHOIS/RDAP response contains the expected domain, confirming the lookup returned the right record.

## Interval and sensitivity matrix

Recommended combinations by check type and service criticality. "-" means the field does not apply: the check is location-independent (measures a global fact like cert expiry or domain registration) and accepts neither `locations` nor `sensitivity`. That set is SSL, Blacklist, Malware, WHOIS, RDAP, CloudStatus, RUM. These have no confirmation quorum, so a single-probe blip can surface as a one-off alert; pair with a location-based check for connectivity confirmation when needed.

| Check type   | Critical service      | Standard service       | Low priority           |
| ------------ | --------------------- | ---------------------- | ---------------------- |
| HTTP         | 1 min / sensitivity 3 | 5 min / sensitivity 2  | 10 min / sensitivity 2 |
| DNS          | 5 min / sensitivity 2 | 10 min / sensitivity 2 | 30 min / sensitivity 2 |
| ICMP         | 3 min / sensitivity 2 | 5 min / sensitivity 2  | 10 min / sensitivity 2 |
| TCP          | 1 min / sensitivity 2 | 5 min / sensitivity 2  | 10 min / sensitivity 2 |
| SMTP / IMAP  | 5 min / sensitivity 2 | 10 min / sensitivity 2 | 30 min / sensitivity 2 |
| SSL          | 60 min / -            | 360 min / -            | 1440 min / -           |
| Blacklist    | 1440 min / -          | 1440 min / -           | 1440 min / -           |
| WHOIS / RDAP | 1440 min / -          | 1440 min / -           | 1440 min / -           |
| Page Speed   | 1440 min / -          | 1440 min / -           | 1440 min / -           |

### Sensitivity trade-offs

| Sensitivity | Locations | Behavior                                                            |
| ----------- | --------- | ------------------------------------------------------------------- |
| 1           | any       | Single failure triggers alert. Fast detection, more false positives |
| 2           | 3+        | Two locations must confirm. Good balance for most services          |
| 3           | 5         | Three confirmations. High confidence, slightly slower detection     |
| 4-5         | 5+        | Very conservative. Use only when false positives are costly         |

Rule of thumb: sensitivity should be at least 2 and never exceed (locations - 1). If you have 3 locations, use sensitivity 2. If you have 5 locations, sensitivity 2-3 is the sweet spot.

## Location selection strategy

### Geographic spread

Choose locations that represent your user base:

| User base     | Recommended locations                         | Count |
| ------------- | --------------------------------------------- | ----- |
| Global        | US East, US West, Europe, Asia, South America | 5     |
| North America | US East, US West, Canada                      | 3     |
| Europe        | UK, Germany, Netherlands                      | 3     |
| Asia-Pacific  | Tokyo, Singapore, Sydney                      | 3     |
| Single region | 2-3 locations within or near the region       | 2-3   |

### When to use 3 vs 5 locations

- **3 locations** (sensitivity 2): sufficient for most services. Lower cost, adequate coverage.
- **5 locations** (sensitivity 2-3): use for critical, revenue-generating services where you want higher confidence and regional visibility.
- **1 location**: only for Page Speed (required constraint) or internal services reachable from a single region.

### Dedicated vs standard locations

- **Standard locations** (`US-East-Virginia`, `EU-West-London`, etc.): used by all location-based checks except Page Speed.
- **Dedicated locations** (`Dedicated-United Kingdom-London`, etc.): required for Page Speed checks. Standard locations will fail for Page Speed.

Do not mix dedicated and standard locations on the same check. Page Speed uses exactly 1 dedicated location; all other checks use standard locations.

## When NOT to use a check type

| Check type  | Skip when                                                            | Use instead                              |
| ----------- | -------------------------------------------------------------------- | ---------------------------------------- |
| ICMP        | Target is a cloud service that drops ICMP (most do)                  | TCP on the service port, or HTTP         |
| UDP         | You're not monitoring a UDP-specific service (DNS, NTP, game server) | TCP or HTTP for standard web services    |
| Page Speed  | You already have one per domain (Lighthouse is expensive)            | HTTP with expect_string for availability |
| Page Speed  | The target is an API endpoint (no visual rendering)                  | HTTP with response time thresholds       |
| WHOIS       | Domain uses a new gTLD with limited WHOIS support                    | RDAP (the modern replacement protocol)   |
| Malware     | Target is an API-only service (no browser-rendered content)          | HTTP checks for availability             |
| POP         | The mail system only supports IMAP                                   | IMAP check                               |
| Transaction | The flow is API-only with no browser UI                              | API check with scripted steps            |
| RUM         | You cannot inject JavaScript into the page (third-party site)        | Page Speed or HTTP checks                |
| Custom      | The service is reachable from external probes                        | HTTP, TCP, or other probe-based checks   |
