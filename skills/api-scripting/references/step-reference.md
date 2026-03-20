---
name: api-step-reference
description: >-
  Complete reference for API check step types, parameters, selectors, variables, and assertions. Used by api-scripting skill.
---

# API check step reference

Complete catalog of step types for Uptime.com API check scripts. For workflow guidance and authentication patterns, see the api-scripting skill.

## Step structure

```json
[{ "step_def": "C_GET", "values": { "url": "https://api.example.com" } }, { "step_def": "V_HTTP_STATUS_CODE_SUCCESSFUL" }]
```

Each step has `step_def` (type identifier) and optional `values` (arguments). All string values support `$VARIABLE$` interpolation.

## HTTP request steps

| Step       | Method |
| ---------- | ------ |
| `C_GET`    | GET    |
| `C_HEAD`   | HEAD   |
| `C_POST`   | POST   |
| `C_PUT`    | PUT    |
| `C_PATCH`  | PATCH  |
| `C_DELETE` | DELETE |

Note: `C_HEAD` is supported by the runner but is not available in the GUI visual editor.

### Request parameters

| Parameter               | Description                                              |
| ----------------------- | -------------------------------------------------------- |
| `url`                   | Request URL (required). Supports `$VAR$`                 |
| `username` / `password` | Basic auth credentials (override defaults from settings) |
| `headers`               | Map of custom HTTP headers. Supports `$VAR$` in values   |
| `content_type`          | Request content type (e.g. `application/json`)           |
| `data`                  | Raw request body                                         |
| `form`                  | URL-encoded form data (map of key-value pairs)           |
| `files`                 | File uploads (map of field name to download URL)         |
| `certificate`           | Client TLS certificate (PEM)                             |
| `key`                   | Client TLS private key (PEM)                             |
| `passphrase`            | Passphrase for encrypted TLS key                         |

`GET`, `HEAD`, and `DELETE` accept only `url` and `headers`. `POST`, `PUT`, and `PATCH` accept all parameters.

**File uploads:** the `files` map value is a URL where the file can be downloaded from (http://, https://, or file://), not inline file content. Maximum 10MB per file.

**Content type handling:**

- Default (raw): sends `data` as-is
- `application/x-www-form-urlencoded`: URL-encodes `form` data
- `multipart/form-data`: multipart form with file support

## Settings and authentication

### `C_SETTINGS_AND_AUTH`

Configures defaults applied to all subsequent requests. Optional; place first if used.

| Parameter                            | Description                                        |
| ------------------------------------ | -------------------------------------------------- |
| `username` / `password`              | Default basic auth                                 |
| `headers`                            | Default headers for all requests. Supports `$VAR$` |
| `content_type`                       | Default content type (default: `application/json`) |
| `follow_redirects`                   | Follow HTTP redirects (default: true, max 10 hops) |
| `insecure_tls`                       | Skip TLS certificate verification (default: false) |
| `connect_to`                         | DNS resolution override (see below)                |
| `certificate` / `key` / `passphrase` | Default client TLS certificate (mTLS)              |

**`connect_to` format:** array of 4-string arrays: `[[original_host, original_port, target_host, target_port]]`. Empty strings match any value. Implements curl-like `--connect-to` functionality for routing requests to alternative hosts without changing the URL.

**Redirect behavior:** follows up to 10 redirects. Authorization header is preserved on same-host redirects (RFC 7235 aware).

**Default User-Agent:** `Mozilla/5.0 (compatible; Uptime/1.0; http://uptime.com)`

### Authentication patterns

**Basic auth:** set `username` / `password` in settings or per-request.

**Bearer token:** use a request step to authenticate, extract token with `C_SET_VARIABLE_SELECTOR`, then use `$TOKEN$` in headers of subsequent requests.

**Client certificate (mTLS):** set `certificate`, `key`, and optionally `passphrase` in settings. Supports encrypted PEM keys (requires DEK-Info header).

**API key:** set as a default header:

```json
{ "headers": { "Authorization": "Bearer $API_KEY$", "X-API-Key": "key-value" } }
```

## Variables

Set variables during execution and reference them in later steps via `$VARIABLE_NAME$` syntax. Names are case-insensitive (stored uppercase), alphanumeric + underscore only. Unresolved variables remain literal.

### Variable setters

| Step                           | Source                             | Key parameters                         |
| ------------------------------ | ---------------------------------- | -------------------------------------- |
| `C_SET_VARIABLE_LITERAL`       | Static value                       | `name`, `value`                        |
| `C_SET_VARIABLE_SELECTOR`      | Response body (JSON/XML/HTML path) | `name`, `selector`, `regex`, `group`   |
| `C_SET_VARIABLE_HEADER`        | Response header                    | `name`, `header`, `regex`, `group`     |
| `C_SET_VARIABLE_RANDOM_CHOICE` | Random from comma-separated list   | `name`, `variants`                     |
| `C_SET_VARIABLE_RANDOM_RANGE`  | Random integer from range          | `name`, `left`, `right`                |
| `C_SET_VARIABLE_DATETIME`      | Formatted date/time                | `name`, `format`, `location`, `offset` |

`C_SET_VARIABLE` is a deprecated alias for `C_SET_VARIABLE_SELECTOR`. Use `C_SET_VARIABLE_SELECTOR` instead.

### `C_SET_VARIABLE_RANDOM_RANGE` details

Range is **exclusive on the right side**: if range is 0-100, the variable can be 0 but not 100.

- `left`: start of range, inclusive (default: `0`)
- `right`: end of range, exclusive (default: `100`)

### `C_SET_VARIABLE_DATETIME` details

| Parameter  | Description                                                                    | Default             |
| ---------- | ------------------------------------------------------------------------------ | ------------------- |
| `name`     | Variable name                                                                  | -                   |
| `format`   | strftime format string                                                         | `%Y-%m-%d %H:%M:%S` |
| `location` | IANA timezone (e.g. `America/New_York`, `UTC`)                                 | `UTC`               |
| `offset`   | Offset from current time: seconds (float) or Go duration (e.g. `1h30m`, `-2h`) | `0`                 |

**Common strftime directives:**

| Directive | Meaning                     |
| --------- | --------------------------- |
| `%Y`      | Year with century (2026)    |
| `%y`      | Year without century (26)   |
| `%m`      | Month as decimal (01-12)    |
| `%d`      | Day of month (01-31)        |
| `%H`      | Hour, 24-hour clock (00-23) |
| `%M`      | Minute (00-59)              |
| `%S`      | Second (00-59)              |

### Regex extraction

The `regex` and `group` parameters on `C_SET_VARIABLE_SELECTOR` and `C_SET_VARIABLE_HEADER` allow substring extraction:

- `regex`: Go regex pattern (RE2 syntax, not PCRE)
- `group`: `0` or empty for full match, `1+` for capture groups, or named group from `(?P<name>...)` pattern

## Assertions

### Status code

| Step                            | Validates      | Parameters    |
| ------------------------------- | -------------- | ------------- |
| `V_HTTP_STATUS_CODE_IS`         | Exact match    | `status_code` |
| `V_HTTP_STATUS_CODE_SUCCESSFUL` | Any 2xx status | (none)        |

### HTTP protocol

| Step                   | Validates                                             | Parameters |
| ---------------------- | ----------------------------------------------------- | ---------- |
| `V_HTTP_PROTO`         | Exact protocol: `HTTP/2.0`, `HTTP/1.1`, or `HTTP/1.0` | `value`    |
| `V_HTTP_PROTO_MAJOR_1` | HTTP/1.x                                              | (none)     |
| `V_HTTP_PROTO_MAJOR_2` | HTTP/2                                                | (none)     |

Note: `V_HTTP_PROTO_MAJOR_1` and `V_HTTP_PROTO_MAJOR_2` are supported by the runner but are not available in the GUI visual editor.

### Response body

| Step                           | Validates                  | Parameters |
| ------------------------------ | -------------------------- | ---------- |
| `V_BODY_CONTAINS_TEXT`         | Body contains text         | `value`    |
| `V_BODY_DOES_NOT_CONTAIN_TEXT` | Body does not contain text | `value`    |
| `V_BODY_MATCHES_REGEX`         | Body matches regex (RE2)   | `value`    |
| `V_BODY_DOES_NOT_MATCH_REGEX`  | Body does not match regex  | `value`    |

### Response headers

| Step                                 | Validates                   | Parameters        |
| ------------------------------------ | --------------------------- | ----------------- |
| `V_HTTP_HEADER_MATCHES_TEXT`         | Header equals value         | `header`, `value` |
| `V_HTTP_HEADER_DOES_NOT_MATCH_TEXT`  | Header does not equal value | `header`, `value` |
| `V_HTTP_HEADER_MATCHES_REGEX`        | Header matches regex        | `header`, `value` |
| `V_HTTP_HEADER_DOES_NOT_MATCH_REGEX` | Header does not match regex | `header`, `value` |

Header name matching is case-insensitive.

### Selector-based (JSON/XML/HTML)

| Step                              | Validates                                                                          | Parameters                       |
| --------------------------------- | ---------------------------------------------------------------------------------- | -------------------------------- |
| `V_SELECTOR_MATCHES_VALUE`        | Extracted value equals text                                                        | `selector`, `value`              |
| `V_SELECTOR_DOES_NOT_MATCH_VALUE` | Extracted value does not equal text                                                | `selector`, `value`              |
| `V_SELECTOR_MATCHES_REGEX`        | Extracted value matches regex                                                      | `selector`, `value`              |
| `V_SELECTOR_DOES_NOT_MATCH_REGEX` | Extracted value does not match regex                                               | `selector`, `value`              |
| `V_NUMERIC_VALUE_MEETS_CONDITION` | Numeric comparison: `condition` (`<`, `<=`, `==`, `>=`, `>`, `!=`) against `value` | `selector`, `condition`, `value` |

All `value` parameters support `$VARIABLE$` interpolation. All selector assertions take a `selector` parameter (see Selectors section).

## Selectors

Selectors extract values from response bodies. Format depends on content type (auto-detected from `Content-Type` header):

### JSON (JSONPath)

| Selector                | Extracts                               |
| ----------------------- | -------------------------------------- |
| `id`                    | Simple field (auto-prefixed to `$.id`) |
| `address.city`          | Nested object field                    |
| `phoneNumbers[0].type`  | Array element                          |
| `phoneNumbers.length()` | Array length                           |
| `$.length()`            | Object key count                       |

### XML (XPath 1.0)

| Selector              | Extracts               |
| --------------------- | ---------------------- |
| `//element[2]/text()` | Text of second element |
| `//element[2]/@attr`  | Attribute value        |

### HTML (CSS selectors)

| Selector                      | Extracts           |
| ----------------------------- | ------------------ |
| `h1`                          | First H1 text      |
| `div.container p:first-child` | CSS selector match |

Empty selector returns the entire response body.

**Auto-detection logic:** `application/json` uses JSONPath; `application/xml` uses XPath; HTML content uses CSS selectors. If Content-Type is undefined, the runner attempts JSON parsing first, then falls back to CSS selectors.
