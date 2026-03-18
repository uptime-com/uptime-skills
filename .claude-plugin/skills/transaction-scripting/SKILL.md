---
name: transaction-scripting
description: >-
  This skill should be used when the user asks to "create a transaction check", "write a browser test", "add a smoke test", "synthetic monitoring script", "transaction check script", "browser automation check", "test signup flow", "test login flow", or needs to author multi-step browser interaction scripts for Uptime.com Transaction checks.
---

# Transaction check scripting

Transaction checks monitor web user flows by executing sequential steps in a real Chromium browser. Each step is a command (action) or validation (assertion). If any step fails, execution stops and an alert is raised.

## Step structure

Each step is a JSON object. A transaction script is an array of steps executed in order.

```json
{
  "step_def": "STEP_TYPE",
  "values": {
    "param1": "value1",
    "param2": "value2"
  }
}
```

## Selectors

Steps targeting DOM elements accept an `element` parameter. Formats are tried in this order:

| Format       | Example                          | Description                                    |
| ------------ | -------------------------------- | ---------------------------------------------- |
| Shadow DOM   | `div.outer#shadow-root.inner`    | Navigates shadow boundaries via `#shadow-root` |
| XPath        | `//*[@id='main']/form/input[1]`  | Detected by leading `/`                        |
| CSS          | `div.container > input#username` | Standard CSS selectors                         |
| Element ID   | `my-input`                       | Looked up as `#my-input` with fallback         |
| Element name | `email`                          | Looked up as `*[name="email"]` with fallback   |

Selectors resolve across all frames (main page + iframes) automatically.

## Variables

Capture dynamic values during execution and reuse in later steps via `$VARIABLE_NAME$` syntax. Names are case-insensitive, stored uppercase, alphanumeric + underscore only.

### `C_SET_VARIABLE`

| Parameter  | Required | Description             |
| ---------- | -------- | ----------------------- |
| `name`     | Yes      | Variable name           |
| `type`     | Yes      | Source type (see below) |
| `options`  | Depends  | Primary option          |
| `options2` | Depends  | Secondary option        |

**Variable types:**

| Type      | `options`                                      | `options2`        | Result                          |
| --------- | ---------------------------------------------- | ----------------- | ------------------------------- |
| `element` | CSS selector                                   | -                 | Text content of matched element |
| `attr`    | CSS selector                                   | Attribute name    | Value of HTML attribute         |
| `date`    | Format string (e.g. `YYYY-MM-DD`)              | Offset in seconds | Formatted UTC date/time         |
| `random`  | Range (`1000-9999`) or pick list (`2:a,b,c,d`) | -                 | Random value                    |
| `const`   | Literal value                                  | -                 | Constant string                 |

### TOTP variables

When `totp_secret` is set in `C_AUTH_AND_SETTINGS`, these become available:

| Variable         | Description                   |
| ---------------- | ----------------------------- |
| `$TOTP_TOKEN$`   | Full OTP code (e.g. `123456`) |
| `$TOTP_TOKEN_N$` | Nth digit of the OTP code     |

The system waits for a fresh token if fewer than 2 seconds of validity remain.

## Command steps

### Navigation

#### `C_OPEN_URL`

| Parameter         | Required | Description                                                          |
| ----------------- | -------- | -------------------------------------------------------------------- |
| `url`             | Yes      | URL to navigate to. Supports `$VARIABLE$`                            |
| `wait_until`      | No       | `load` (default), `domcontentloaded`, `networkidle0`, `networkidle2` |
| `timeout`         | No       | Seconds. Default: 30                                                 |
| `skip_navigation` | No       | Boolean. Skip waiting for navigation (useful for SPAs)               |

### Interaction

| Step              | Purpose                              | Key parameters                                                            |
| ----------------- | ------------------------------------ | ------------------------------------------------------------------------- |
| `C_MOUSE_CLICK`   | Click element (realistic)            | `element`, `button` (left/middle/right), `click_count`, `skip_navigation` |
| `C_CLICK_ELEMENT` | Click element (JS-level, deprecated) | `element`                                                                 |
| `C_HOVER_ELEMENT` | Hover over element                   | `element`                                                                 |
| `C_FOCUS_ELEMENT` | Focus element                        | `element`                                                                 |
| `C_FILL_FIELD`    | Clear and type text                  | `element`, `text` (supports `$VAR$`), `typing_delay` (0-1000ms)           |
| `C_CHECK_BOX`     | Check checkbox/radio                 | `element`                                                                 |
| `C_UNCHECK_BOX`   | Uncheck checkbox                     | `element`                                                                 |
| `C_SUBMIT_FORM`   | Submit form containing element       | `element`, `skip_navigation`                                              |

### Wait

| Step                          | Purpose                          | Key parameters                           |
| ----------------------------- | -------------------------------- | ---------------------------------------- |
| `C_WAIT_FOR_ELEMENT`          | Wait for element to appear       | `element`, `timeout` (default 25s)       |
| `C_WAIT_FOR_NOT_ELEMENT`      | Wait for element to disappear    | `element`, `timeout`                     |
| `C_WAIT_FOR_ELEMENT_TEXT`     | Wait for element to contain text | `element`, `text`, `is_regex`, `timeout` |
| `C_WAIT_FOR_NOT_ELEMENT_TEXT` | Wait for text to change          | `element`, `text`, `is_regex`, `timeout` |
| `C_WAIT_FOR_ONE_SECOND`       | Pause 1 second                   | (none)                                   |

## Validation steps

Validations assert conditions. Failure stops execution immediately.

### HTTP/Network

| Step                                  | Validates                         |
| ------------------------------------- | --------------------------------- |
| `V_HTTP_STATUS_CODE_IS`               | Status code matches `http_status` |
| `V_HTTP_HEADER_CONTAINS_TEXT`         | Header `header` contains `text`   |
| `V_HTTP_HEADER_DOES_NOT_CONTAIN_TEXT` | Header does not contain `text`    |

### URL and title

| Step                       | Validates                           |
| -------------------------- | ----------------------------------- |
| `V_URL_CONTAINS`           | Current URL contains `text`         |
| `V_URL_DOES_NOT_CONTAIN`   | Current URL does not contain `text` |
| `V_TITLE_CONTAINS`         | Page title contains `text`          |
| `V_TITLE_DOES_NOT_CONTAIN` | Page title does not contain `text`  |

All support `is_regex` (boolean) and `$VARIABLE$` interpolation in `text`.

### Element

| Step                              | Validates                                       |
| --------------------------------- | ----------------------------------------------- |
| `V_ELEMENT_EXISTS`                | Element exists                                  |
| `V_ELEMENT_DOES_NOT_EXIST`        | Element does not exist                          |
| `V_ELEMENT_CONTAINS_TEXT`         | Element text contains `text` (case-insensitive) |
| `V_ELEMENT_DOES_NOT_CONTAIN_TEXT` | Element text does not contain `text`            |
| `V_BOX_IS_CHECKED`                | Checkbox/radio is checked                       |
| `V_BOX_IS_NOT_CHECKED`            | Checkbox/radio is not checked                   |

For `<select>`: checks selected option value and text. For `<input>`: checks value and textContent.

## Pagespeed steps

Integrate Google Lighthouse for performance measurement. Only **one pagespeed action** per transaction check.

| Step                                                          | Purpose                                |
| ------------------------------------------------------------- | -------------------------------------- |
| `C_PAGESPEED_NAVIGATE`                                        | Navigate to `url` and capture metrics  |
| `C_PAGESPEED_SNAPSHOT`                                        | Capture metrics of current page state  |
| `C_PAGESPEED_START_TIMESPAN` / `C_PAGESPEED_END_TIMESPAN`     | Measure metrics over a time period     |
| `C_PAGESPEED_START_NAVIGATION` / `C_PAGESPEED_END_NAVIGATION` | Measure metrics for a navigation event |

START/END pairs must match type. Results include Performance, Accessibility, Best Practices, and SEO scores.

## Authentication and settings

#### `C_AUTH_AND_SETTINGS`

Must be the **first step** and can only appear **once**.

| Parameter               | Description                                                       |
| ----------------------- | ----------------------------------------------------------------- |
| `username` / `password` | HTTP Basic Auth credentials                                       |
| `headers`               | Extra HTTP headers (`Key: Value`, one per line). Supports `$VAR$` |
| `filter_urls`           | Block requests matching regex patterns (one per line)             |
| `viewport_size`         | Predefined viewport (see below)                                   |
| `is_mobile`             | Enable mobile User-Agent and touch events                         |
| `totp_secret`           | Base32-encoded TOTP secret for 2FA                                |
| `totp_period`           | TOTP period in seconds (default: 30)                              |
| `totp_digits`           | Number of TOTP digits (default: 6)                                |
| `no_screenshots`        | Disable screenshot capture                                        |

**Viewports:** Small Laptop (1366x768), Laptop (1600x900), Large Desktop (1920x1080), iPad Pro (1024x1366), iPad (768x1024), iPad Landscape (1024x768), iPhone 6/7/8 Plus (414x736), Pixel 2 (411x731), iPhone X (375x812), iPhone 6/7/8 (375x667), iPhone 5/SE (320x568).

## Execution flow

1. Browser acquired (Chromium via Puppeteer)
2. Page initialized with configured viewport, auth, headers, URL filters
3. Steps execute sequentially; variables resolved, elements found across frames
4. On failure: error screenshot captured, execution stops
5. Pagespeed report generated if applicable
6. Cleanup: page and browser context closed

**Timeouts:** Global (entire script, default 30s) and step-level (individual steps can override).

**Automatic behaviors:** Dialogs (alert/confirm/prompt) auto-accepted. Console messages captured. Network requests recorded for waterfall view.

**Exit codes:** 0 = success, 1 = general error, 2 = browser error, 3 = global timeout, 4 = step failure.
