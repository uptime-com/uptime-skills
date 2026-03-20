---
name: transaction-step-reference
description: >-
  Complete reference for Transaction check step types, parameters, selectors, variables, and pagespeed integration. Used by transaction-scripting skill.
---

# Transaction check step reference

Complete catalog of step types for Uptime.com Transaction check scripts. For workflow guidance and scripting patterns, see the transaction-scripting skill.

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

The key is `step_def` (not `step_type`). Values use `element` for selectors (not `search_text`).

## Selectors

Steps targeting DOM elements accept an `element` parameter. Formats are tried in this order:

| Format     | Example                          | Description                                    |
| ---------- | -------------------------------- | ---------------------------------------------- |
| Shadow DOM | `div.outer#shadow-root.inner`    | Navigates shadow boundaries via `#shadow-root` |
| XPath      | `//*[@id='main']/form/input[1]`  | Detected by leading `/`                        |
| CSS        | `div.container > input#username` | Standard CSS selectors                         |
| Element ID | `my-input`                       | Looked up as `#my-input` with fallback         |
| Name       | `email`                          | Looked up as `*[name="email"]` with fallback   |

Selectors resolve across all frames (main page + iframes) automatically. Shadow DOM supports multiple boundary crossings: `parent#shadow-root child#shadow-root deeper`.

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

| Type      | `options`                                               | `options2`        | Result                          |
| --------- | ------------------------------------------------------- | ----------------- | ------------------------------- |
| `element` | CSS selector, XPath, ID, or name                        | -                 | Text content of matched element |
| `attr`    | CSS selector, XPath, ID, or name                        | Attribute name    | Value of HTML attribute         |
| `date`    | Moment.js format string (e.g. `YYYY-MM-DD`, `HH:mm:ss`) | Offset in seconds | Formatted UTC date/time         |
| `random`  | Range (`1000-9999`) or pick list (`2:a,b,c,d`)          | -                 | Random value                    |
| `const`   | Literal value                                           | -                 | Constant string                 |

**Random value specifier formats:**

- `Number` or `Min-Max`: random integer in range. `100` means 0-100. `1705-1980` means 1705-1980.
- `[Count:]item1,item2,...`: pick Count items (default 1) from the comma-separated list and concatenate. Example: `3:1,a,b,c,37` could produce `1b37`.

**Date format:** uses [Moment.js format strings](https://momentjs.com/docs/#/displaying/format/). The offset in `options2` is an integer number of seconds added to current UTC time (negative values supported). Leave blank for no adjustment.

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

Navigate to a new URL.

| Parameter         | Required | Description                                                          |
| ----------------- | -------- | -------------------------------------------------------------------- |
| `url`             | Yes      | URL to navigate to. Supports `$VARIABLE$`                            |
| `wait_until`      | No       | `load` (default), `domcontentloaded`, `networkidle0`, `networkidle2` |
| `timeout`         | No       | Seconds (default 30, max 900)                                        |
| `skip_navigation` | No       | Boolean. Skip waiting for navigation (useful for SPAs)               |

`wait_until` options:

- `load` (default): wait for the load event
- `domcontentloaded`: wait for the DOMContentLoaded event
- `networkidle0`: no more than 0 network connections for at least 500ms
- `networkidle2`: no more than 2 network connections for at least 500ms

### Interaction

| Step              | Purpose                               | Key parameters                                                                    |
| ----------------- | ------------------------------------- | --------------------------------------------------------------------------------- |
| `C_MOUSE_CLICK`   | Click element (realistic mouse event) | `element`, `button` (left/middle/right), `click_count` (1/2/3), `skip_navigation` |
| `C_CLICK_ELEMENT` | Click element (deprecated)            | `element`. Use `C_MOUSE_CLICK` instead                                            |
| `C_HOVER_ELEMENT` | Hover over element                    | `element`                                                                         |
| `C_FOCUS_ELEMENT` | Focus element                         | `element`                                                                         |
| `C_FILL_FIELD`    | Clear and type text                   | `element`, `text` (supports `$VAR$`), `typing_delay` (0-1000ms, default 20ms)     |
| `C_CHECK_BOX`     | Check checkbox/radio                  | `element`                                                                         |
| `C_UNCHECK_BOX`   | Uncheck checkbox                      | `element`                                                                         |
| `C_SUBMIT_FORM`   | Submit form containing element        | `element`, `skip_navigation`                                                      |

`C_MOUSE_CLICK` scrolls the element into view if needed, then uses mouse to click in the center of the element. If the element is detached from DOM, the step errors.

`C_FILL_FIELD` supports text inputs, password fields, hidden fields, `<textarea>`, and `<select>` elements. For `<select>`, the `text` value must match an option's text or value attribute.

### Wait

| Step                          | Purpose                          | Key parameters                                      |
| ----------------------------- | -------------------------------- | --------------------------------------------------- |
| `C_WAIT_FOR_ELEMENT`          | Wait for element to appear       | `element`, `timeout` (seconds, default 25, max 900) |
| `C_WAIT_FOR_NOT_ELEMENT`      | Wait for element to disappear    | `element`, `timeout` (seconds, default 25, max 900) |
| `C_WAIT_FOR_ELEMENT_TEXT`     | Wait for element to contain text | `element`, `text`, `is_regex`, `timeout`            |
| `C_WAIT_FOR_NOT_ELEMENT_TEXT` | Wait for text to change          | `element`, `text`, `is_regex`, `timeout`            |
| `C_WAIT_FOR_ONE_SECOND`       | Pause 1 second                   | (none)                                              |

## Validation steps

Validations assert conditions. Failure stops execution immediately.

### HTTP/Network

| Step                                  | Validates                    | Key parameters               |
| ------------------------------------- | ---------------------------- | ---------------------------- |
| `V_HTTP_STATUS_CODE_IS`               | Status code matches          | `http_status`                |
| `V_HTTP_HEADER_CONTAINS_TEXT`         | Header contains text         | `header`, `text`, `is_regex` |
| `V_HTTP_HEADER_DOES_NOT_CONTAIN_TEXT` | Header does not contain text | `header`, `text`, `is_regex` |

HTTP header validations check only the last HTTP request prior to the current step. The validation fails if the specified header is not found in the response.

### URL and title

| Step                       | Validates                         | Key parameters     |
| -------------------------- | --------------------------------- | ------------------ |
| `V_URL_CONTAINS`           | Current URL contains text         | `text`, `is_regex` |
| `V_URL_DOES_NOT_CONTAIN`   | Current URL does not contain text | `text`, `is_regex` |
| `V_TITLE_CONTAINS`         | Page title contains text          | `text`, `is_regex` |
| `V_TITLE_DOES_NOT_CONTAIN` | Page title does not contain text  | `text`, `is_regex` |

All support `$VARIABLE$` interpolation in `text`.

### Element

| Step                              | Validates                                     | Key parameters                |
| --------------------------------- | --------------------------------------------- | ----------------------------- |
| `V_ELEMENT_EXISTS`                | Element exists                                | `element`                     |
| `V_ELEMENT_DOES_NOT_EXIST`        | Element does not exist                        | `element`                     |
| `V_ELEMENT_CONTAINS_TEXT`         | Element text contains text (case-insensitive) | `element`, `text`, `is_regex` |
| `V_ELEMENT_DOES_NOT_CONTAIN_TEXT` | Element text does not contain text            | `element`, `text`, `is_regex` |
| `V_BOX_IS_CHECKED`                | Checkbox/radio is checked                     | `element`                     |
| `V_BOX_IS_NOT_CHECKED`            | Checkbox/radio is not checked                 | `element`                     |

`V_ELEMENT_CONTAINS_TEXT` will fail if the matched element contains no text at all. Use `V_ELEMENT_DOES_NOT_CONTAIN_TEXT` to handle empty elements.

For `<select>`: checks selected option value and text. For `<input>`: checks value and textContent.

## Pagespeed steps

Integrate Google Lighthouse for performance measurement. Only **one pagespeed action** per transaction check.

| Step                           | Purpose                               |
| ------------------------------ | ------------------------------------- |
| `C_PAGESPEED_NAVIGATE`         | Navigate to `url` and capture metrics |
| `C_PAGESPEED_SNAPSHOT`         | Capture metrics of current page state |
| `C_PAGESPEED_START_TIMESPAN`   | Start measuring a time period         |
| `C_PAGESPEED_END_TIMESPAN`     | End measuring a time period           |
| `C_PAGESPEED_START_NAVIGATION` | Start measuring a navigation event    |
| `C_PAGESPEED_END_NAVIGATION`   | End measuring a navigation event      |

START/END pairs must match type. Results include Performance, Accessibility, Best Practices, and SEO scores.

### Pagespeed settings

Set in `C_AUTH_AND_SETTINGS`:

| Parameter                          | Description                         | Values                          |
| ---------------------------------- | ----------------------------------- | ------------------------------- |
| `pagespeed_connection_throttling`  | Simulate network conditions         | See connection throttling table |
| `pagespeed_emulated_device`        | Simulate device hardware            | See emulated device table       |
| `pagespeed_uptime_grade_threshold` | Fail check if grade below threshold | `A`, `B`, `C`, `D`, `F`         |

**Connection throttling options:**

| Value            | Description                   |
| ---------------- | ----------------------------- |
| `UNTHROTTLED`    | No throttling                 |
| `BROADBAND_FAST` | 25ms RTT, 20 Mbps             |
| `BROADBAND`      | 30ms RTT, 5 Mbps              |
| `BROADBAND_SLOW` | 50ms RTT, 1.5 Mbps            |
| `LTE`            | 100ms RTT, 15/10 Mbps down/up |
| `FOUR_G`         | 125ms RTT, 9 Mbps             |
| `FOUR_G_SLOW`    | 150ms RTT, 1 Mbps             |
| `THREE_G`        | 200ms RTT, 1.6 Mbps           |
| `THREE_G_SLOW`   | 250ms RTT, 750 Kbps           |

**Emulated devices:**

| Category | Values                                                                                                                                             |
| -------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| Desktop  | `DEFAULT` (1920x1080), `DESKTOP_LOW_END` (1366x768, 2x CPU slowdown)                                                                               |
| iPhone   | `IPHONE_11`, `IPHONE_11_PRO`, `IPHONE_11_PRO_MAX`, `IPHONE_12`, `IPHONE_12_PRO_MAX`, `IPHONE_13`, `IPHONE_13_PRO_MAX`                              |
| Pixel    | `GOOGLE_PIXEL_6`, `GOOGLE_PIXEL_6_PRO`                                                                                                             |
| Samsung  | `SAMSUNG_GALAXY_S10`, `SAMSUNG_GALAXY_S10_PLUS`, `SAMSUNG_GALAXY_S21`, `SAMSUNG_GALAXY_S21_PLUS`, `SAMSUNG_GALAXY_S21_ULTRA`, `SAMSUNG_GALAXY_S22` |
| iPad     | `IPAD_AIR`, `IPAD_MINI`, `IPAD_PRO`                                                                                                                |
| Generic  | `MOBILE_MID_TIER` (360x800, 4x CPU), `MOBILE_LOW_END` (360x800, 10x CPU)                                                                           |

## Authentication and settings

### `C_AUTH_AND_SETTINGS`

Must be the **first step** and can only appear **once**.

| Parameter               | Description                                                                                   |
| ----------------------- | --------------------------------------------------------------------------------------------- |
| `username` / `password` | HTTP Basic Auth credentials                                                                   |
| `headers`               | Extra HTTP headers (`Key: Value`, one per line). Supports `$VAR$`                             |
| `filter_urls`           | Block requests matching regex patterns (one per line). Useful for blocking analytics/tracking |
| `viewport_size`         | Predefined viewport (see below)                                                               |
| `is_mobile`             | Enable mobile User-Agent, meta viewport, and touch events                                     |
| `totp_secret`           | Base32-encoded TOTP secret for 2FA                                                            |
| `totp_period`           | TOTP period in seconds (default 30, range 30-60)                                              |
| `totp_digits`           | Number of TOTP digits (default 6, range 1-10)                                                 |
| `no_screenshots`        | Disable screenshot capture. Useful for pages with sensitive information                       |

Pagespeed settings (`pagespeed_connection_throttling`, `pagespeed_emulated_device`, `pagespeed_uptime_grade_threshold`) are also set here. See Pagespeed section above.

### `viewport_size` values

Use the dimension string, not the label.

| Value             | Device             |
| ----------------- | ------------------ |
| `""` (empty/omit) | Default (1366x768) |
| `1366x768`        | Small Laptop       |
| `1600x900`        | Laptop             |
| `1920x1080`       | Large Desktop      |
| `1024x1366`       | iPad Pro           |
| `768x1024`        | iPad               |
| `1024x768`        | iPad Landscape     |
| `414x736`         | iPhone 6/7/8 Plus  |
| `411x731`         | Pixel 2            |
| `375x812`         | iPhone X           |
| `375x667`         | iPhone 6/7/8       |
| `320x568`         | iPhone 5/SE        |

## Execution flow

1. Browser acquired (Chromium via Puppeteer)
2. Page initialized with configured viewport, auth, headers, URL filters
3. Steps execute sequentially; variables resolved, elements found across frames
4. On failure: error screenshot captured (unless disabled), execution stops
5. Pagespeed report generated if applicable
6. Cleanup: page and browser context closed

**Timeouts:** Global default 60 seconds (180 seconds for pagespeed checks). Step-level timeouts override for individual steps (default 25s for wait operations, 30s for navigation).

**Automatic behaviors:** Dialogs (alert/confirm/prompt) auto-accepted. Console messages captured. Network requests recorded for waterfall view.
