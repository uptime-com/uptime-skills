---
name: api-scripting
description: >-
  This skill should be used when the user asks to "create an API check", "write an API monitoring script", "multi-step API test", "API check script", "monitor API endpoint", "test API flow", or needs to author multi-step HTTP request sequences with assertions for Uptime.com API checks.
---

# API check scripting

API checks execute multi-step HTTP request sequences with assertions. Scripts are JSON arrays of step objects executed sequentially. If any assertion fails, execution stops and an alert is raised.

For the complete step type catalog, parameters, selectors, and variables, see `references/step-reference.md`.

## Script format

```json
[{ "step_def": "C_GET", "values": { "url": "https://api.example.com/health" } }, { "step_def": "V_HTTP_STATUS_CODE_SUCCESSFUL" }]
```

The key is `step_def` (not `step_type`). All string values support `$VARIABLE$` interpolation.

## Authentication patterns

### Bearer token flow

The most common pattern: authenticate, extract token, use in subsequent requests.

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
  { "step_def": "V_HTTP_STATUS_CODE_IS", "values": { "status_code": "200" } },
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

### Static API key

Set in `C_SETTINGS_AND_AUTH` as a default header:

```json
{
  "step_def": "C_SETTINGS_AND_AUTH",
  "values": { "headers": { "X-API-Key": "your-api-key" }, "content_type": "application/json" }
}
```

### Client certificate (mTLS)

Set `certificate`, `key`, and optionally `passphrase` in `C_SETTINGS_AND_AUTH`.

## Scripting workflow

1. Understand the API flow to monitor
2. Determine authentication method (bearer token, API key, basic auth, mTLS)
3. Build the script: authenticate, then exercise the key endpoints
4. Add assertions after each request (`V_HTTP_STATUS_CODE_SUCCESSFUL` at minimum)
5. Use `C_SET_VARIABLE_SELECTOR` to chain responses between steps
6. Present the monitoring plan as a numbered list explaining what each step does in plain language
7. Confirm with the user before creating the check via `create_api_check`

## Common pitfalls

- **Using `C_SET_VARIABLE`**: deprecated. Use `C_SET_VARIABLE_SELECTOR` instead.
- **Missing assertions**: every request should have at least a status code assertion. Without one, a 500 error goes undetected.
- **Hardcoded tokens**: use the bearer token flow pattern to authenticate dynamically. Static tokens expire.
- **Wrong selector format**: JSON responses use dot notation (`data.user.email`), XML uses XPath (`//user/email/text()`). The runner auto-detects from Content-Type.
- **Regex syntax**: API checks use Go RE2 regex, not PCRE. Lookaheads and backreferences are not supported.
