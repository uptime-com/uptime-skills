---
name: transaction-scripting
description: >-
  This skill should be used when the user asks to "create a transaction check", "write a browser test", "add a smoke test", "synthetic monitoring script", "transaction check script", "browser automation check", "test signup flow", "test login flow", or needs to author multi-step browser interaction scripts for Uptime.com Transaction checks.
---

# Transaction check scripting

Transaction checks monitor web user flows by executing sequential steps in a real Chromium browser. Each step is a command (action) or validation (assertion). If any step fails, execution stops and an alert is raised.

For the complete step type catalog, parameters, selectors, and variables, see `references/step-reference.md`.

## Script format

A transaction script is a JSON array of step objects executed in order:

```json
[
  { "step_def": "C_OPEN_URL", "values": { "url": "https://example.com" } },
  { "step_def": "C_FILL_FIELD", "values": { "element": "input[name='email']", "text": "user@example.com" } },
  { "step_def": "C_MOUSE_CLICK", "values": { "element": "button[type='submit']" } },
  { "step_def": "V_URL_CONTAINS", "values": { "text": "/dashboard" } }
]
```

The key is `step_def` (not `step_type`). Values use `element` for selectors (not `search_text`).

## Scripting workflow

1. Identify the user flow to monitor (login, checkout, signup, etc.)
2. Break it into discrete steps: navigate, interact, wait, validate
3. Use `C_AUTH_AND_SETTINGS` as the first step if you need custom viewport, headers, or TOTP
4. Add validation steps after key transitions to catch failures early
5. Use `C_WAIT_FOR_ELEMENT` before interacting with dynamically loaded content
6. Present the monitoring plan as a numbered list explaining what each step does in plain language
7. Confirm with the user before creating the check via `create_transaction_check`

## Common pitfalls

- **Using `C_CLICK_ELEMENT`**: deprecated. Use `C_MOUSE_CLICK` instead (realistic mouse events).
- **Missing waits on SPAs**: single-page apps need `C_WAIT_FOR_ELEMENT` or `wait_until: networkidle0` since navigation events don't fire on client-side routing.
- **Interacting before element exists**: always wait for dynamic content. The default wait timeout is 25 seconds.
- **Selecting by unstable attributes**: prefer `data-testid` or `name` attributes over auto-generated class names.
- **Long scripts without checkpoints**: add `V_URL_CONTAINS` or `V_ELEMENT_EXISTS` after each major page transition to pinpoint failures.
