---
name: api-scripting
description: >-
  This skill should be used when the user asks to "create an API check", "write an API monitoring script", "multi-step API test", "API check script", "monitor API endpoint", "test API flow", or needs to author multi-step HTTP request sequences with assertions for Uptime.com API checks.
---

# API check scripting

API checks execute multi-step HTTP request sequences with assertions. Scripts are JSON arrays of step objects executed sequentially. If any assertion fails, execution stops and an alert is raised.

## Step structure

```json
[{ "step_def": "C_GET", "values": { "url": "https://api.example.com" } }, { "step_def": "V_HTTP_STATUS_CODE_SUCCESSFUL" }]
```

Each step has `step_def` (type identifier) and optional `values` (arguments). All string values support `$VARIABLE$` interpolation.

## HTTP request steps

All standard HTTP methods are available:

| Step       | Method |
| ---------- | ------ |
| `C_GET`    | GET    |
| `C_HEAD`   | HEAD   |
| `C_POST`   | POST   |
| `C_PUT`    | PUT    |
| `C_PATCH`  | PATCH  |
| `C_DELETE` | DELETE |

### Request parameters

| Parameter               | Description                                    |
| ----------------------- | ---------------------------------------------- |
| `url`                   | Request URL. Supports `$VAR$`                  |
| `username` / `password` | Basic auth credentials (override defaults)     |
| `headers`               | Map of custom HTTP headers                     |
| `content_type`          | Request content type (e.g. `application/json`) |
| `data`                  | Raw request body                               |
| `form`                  | URL-encoded form data (map of key-value pairs) |
| `files`                 | File uploads (map of field name to file URL)   |
| `certificate`           | Client TLS certificate (PEM)                   |
| `key`                   | Client TLS private key (PEM)                   |
| `passphrase`            | Passphrase for encrypted TLS key               |

### Example: POST with JSON body

```json
{
  "step_def": "C_POST",
  "values": {
    "url": "https://api.example.com/users",
    "content_type": "application/json",
    "data": "{\"name\": \"John\", \"email\": \"john@example.com\"}",
    "headers": { "X-API-Key": "$API_KEY$" }
  }
}
```

## Settings and authentication

#### `C_SETTINGS_AND_AUTH`

Configures defaults applied to all subsequent requests. Optional; place first if used.

| Parameter                            | Description                                                          |
| ------------------------------------ | -------------------------------------------------------------------- |
| `username` / `password`              | Default basic auth                                                   |
| `headers`                            | Default headers for all requests                                     |
| `content_type`                       | Default content type                                                 |
| `follow_redirects`                   | Follow HTTP redirects (default: true, max 10)                        |
| `insecure_tls`                       | Skip TLS certificate verification                                    |
| `connect_to`                         | DNS resolution override (`[[host, port, target_host, target_port]]`) |
| `certificate` / `key` / `passphrase` | Default client TLS certificate (mTLS)                                |

### Authentication patterns

**Basic auth**: set `username` / `password` in settings or per-request.

**Bearer token**: use a request step to authenticate, extract token with `C_SET_VARIABLE_SELECTOR`, then use `$TOKEN$` in headers of subsequent requests.

**Client certificate (mTLS)**: set `certificate`, `key`, and optionally `passphrase` in settings.

**API key**: set as a default header:

```json
{ "headers": { "Authorization": "Bearer $API_KEY$", "X-API-Key": "key-value" } }
```

## Variables

Set variables during execution and reference them in later steps via `$VARIABLE_NAME$` syntax. Names are case-insensitive (stored uppercase). Unresolved variables remain literal.

### Variable setters

| Step                           | Source                             | Key parameters                                                                    |
| ------------------------------ | ---------------------------------- | --------------------------------------------------------------------------------- |
| `C_SET_VARIABLE_LITERAL`       | Static value                       | `name`, `value`                                                                   |
| `C_SET_VARIABLE_SELECTOR`      | Response body (JSON/XML/HTML path) | `name`, `selector`, `regex`, `group`                                              |
| `C_SET_VARIABLE_HEADER`        | Response header                    | `name`, `header`, `regex`, `group`                                                |
| `C_SET_VARIABLE_RANDOM_CHOICE` | Random from list                   | `name`, `variants` (comma-separated)                                              |
| `C_SET_VARIABLE_RANDOM_RANGE`  | Random integer                     | `name`, `left`, `right`                                                           |
| `C_SET_VARIABLE_DATETIME`      | Formatted date/time                | `name`, `format` (strftime), `location` (IANA tz), `offset` (seconds or duration) |

Note: `C_SET_VARIABLE` is a legacy alias for `C_SET_VARIABLE_SELECTOR`.

### Regex extraction

The `regex` and `group` parameters on selector/header variable setters allow substring extraction:

- `regex`: Go regex pattern
- `group`: `0` for full match, `1+` for capture groups, or named group from `(?P<name>...)` pattern

### Example: extract token from login response

```json
[
  {
    "step_def": "C_POST",
    "values": {
      "url": "https://api.example.com/login",
      "content_type": "application/json",
      "data": "{\"email\": \"user@example.com\", \"password\": \"secret\"}"
    }
  },
  {
    "step_def": "C_SET_VARIABLE_SELECTOR",
    "values": { "name": "TOKEN", "selector": "data.access_token" }
  },
  {
    "step_def": "C_GET",
    "values": {
      "url": "https://api.example.com/profile",
      "headers": { "Authorization": "Bearer $TOKEN$" }
    }
  },
  { "step_def": "V_HTTP_STATUS_CODE_SUCCESSFUL" }
]
```

## Assertions

### Status code

| Step                            | Validates                         |
| ------------------------------- | --------------------------------- |
| `V_HTTP_STATUS_CODE_IS`         | Exact match (`status_code` param) |
| `V_HTTP_STATUS_CODE_SUCCESSFUL` | Any 2xx status                    |

### HTTP protocol

| Step                   | Validates                               |
| ---------------------- | --------------------------------------- |
| `V_HTTP_PROTO`         | Exact protocol string (e.g. `HTTP/2.0`) |
| `V_HTTP_PROTO_MAJOR_1` | HTTP/1.x                                |
| `V_HTTP_PROTO_MAJOR_2` | HTTP/2                                  |

### Response body

| Step                           | Validates                         |
| ------------------------------ | --------------------------------- |
| `V_BODY_CONTAINS_TEXT`         | Body contains `value`             |
| `V_BODY_DOES_NOT_CONTAIN_TEXT` | Body does not contain `value`     |
| `V_BODY_MATCHES_REGEX`         | Body matches regex `value`        |
| `V_BODY_DOES_NOT_MATCH_REGEX`  | Body does not match regex `value` |

### Response headers

| Step                                 | Validates                           |
| ------------------------------------ | ----------------------------------- |
| `V_HTTP_HEADER_MATCHES_TEXT`         | Header `header` equals `value`      |
| `V_HTTP_HEADER_DOES_NOT_MATCH_TEXT`  | Header does not equal `value`       |
| `V_HTTP_HEADER_MATCHES_REGEX`        | Header matches regex `value`        |
| `V_HTTP_HEADER_DOES_NOT_MATCH_REGEX` | Header does not match regex `value` |

### Selector-based (JSON/XML/HTML)

| Step                              | Validates                                                                          |
| --------------------------------- | ---------------------------------------------------------------------------------- |
| `V_SELECTOR_MATCHES_VALUE`        | Extracted value equals `value`                                                     |
| `V_SELECTOR_DOES_NOT_MATCH_VALUE` | Extracted value does not equal `value`                                             |
| `V_SELECTOR_MATCHES_REGEX`        | Extracted value matches regex `value`                                              |
| `V_SELECTOR_DOES_NOT_MATCH_REGEX` | Extracted value does not match regex                                               |
| `V_NUMERIC_VALUE_MEETS_CONDITION` | Numeric comparison: `condition` (`<`, `<=`, `==`, `>=`, `>`, `!=`) against `value` |

All selector assertions take a `selector` parameter (see Selectors section).

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
