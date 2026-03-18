---
name: check-management
description: >-
  This skill should be used when the user asks to "update a check", "modify check settings", "change check interval", "change contact group", "edit check configuration", "edit transaction script", "edit API check script", "delete a check", "simulate an outage", "set up a demo", "test monitoring", or needs to modify existing checks. Covers update patterns, contact group management, and outage simulation. For creating new checks from scratch, see monitoring-setup.
---

# Check management — operational knowledge

Practical patterns for managing existing monitoring checks via the MCP server.

For Transaction check scripting, see `references/scripting-txn.md`. For API check scripting, see `references/scripting-api.md`.

## Updating existing checks

Use `update_check` to modify check properties in place. The tool accepts partial updates — only the fields you specify are changed; omitted fields retain their current values.

### Update workflow

1. **Get current state**: `get_check` to see existing configuration.
2. **Apply changes**: `update_check` with only the fields that need to change.
3. **Verify**: `get_check` again to confirm the update took effect.

### Commonly updated fields

| Field            | Notes                                                                                    |
| ---------------- | ---------------------------------------------------------------------------------------- |
| `interval`       | Minutes between checks. Some types have minimums (e.g. Page Speed ≥ 1440).               |
| `locations`      | Probe locations. Only for location-based checks — never set on auto-located types.       |
| `sensitivity`    | Number of confirming locations before alerting. Use ≥ 2.                                 |
| `contact_groups` | Array of contact group names/IDs.                                                        |
| `is_paused`      | `true` to pause, `false` to resume. Useful during maintenance.                           |
| `tags`           | Array of tag names. Replaces the full tag list — include existing tags you want to keep. |

### Batch updates

When updating multiple checks (e.g. changing interval for all checks in a tag group):

1. `list_checks` with tag filter to get the target set.
2. Issue `update_check` calls in parallel — updates are independent.
3. Verify a sample with `get_check`.

## Contact group patterns

| Group pattern                               | Use case                                                              |
| ------------------------------------------- | --------------------------------------------------------------------- |
| `Default`                                   | Production notifications — real alerts to real people                 |
| Dedicated group (e.g. `demo-notifications`) | Scoped notifications for a specific purpose without polluting Default |

## Pausing, maintenance windows, and resuming checks

### Manual pause/resume

Use `update_check` with `is_paused: true/false` for ad-hoc pausing. Paused checks:

- Retain their configuration and history
- Stop generating alerts
- Can be resumed instantly
- Open alerts auto-resolve when paused (`bulk_alert_resolution_on_pause`)

### Scheduled maintenance windows (preferred)

For recurring maintenance, use scheduled maintenance windows instead of manual pausing:

- Define recurring windows (e.g. every Sunday 02:00–04:00 UTC)
- Checks pause and resume automatically — no risk of forgetting to unpause
- Maintenance state is visible in dashboards and reports
- Better audit trail than manual pause/resume

Use manual pausing only for unplanned, one-off situations. Use maintenance windows for anything predictable.

### Escalations

Escalations are separate from contact groups and define how alerts intensify over time:

- Contact groups receive the initial alert
- Escalation rules trigger if the alert is not acknowledged within a configured window
- Can notify different contacts or use different channels (e.g. phone call after SMS goes unacknowledged)

When setting up or reviewing checks, verify that critical checks have both a contact group _and_ escalation rules configured.

Delete only when a check is permanently no longer needed.

## Outage simulation for demos

### Using HTTPBin

If an HTTPBin instance is available (e.g. `httpbin.org`):

- **Trigger outage**: `update_check` — change the address to `/status/503` (or 500, 502, etc.)
- **Resolve outage**: `update_check` — change it back to `/` or `/status/200`

### Flapping simulation

No well-known third-party service provides automated flapping endpoints. Options for simulating flapping (alternating 200/503):

- **Manual toggling**: switch the check address between `/status/200` and `/status/503` during a live demo — interactive and easy to narrate.
- **Self-hosted endpoint**: deploy a small handler that returns status based on time:
  ```
  if (minute / 2) % 2 == 0 → 200 else → 503
  ```
- **httpstat.us**: returns static status codes per request (`httpstat.us/503`), no auto-flapping, but useful for one-shot tests.

### Monitoring uncontrolled domains

Monitoring domains you don't control (e.g. `example.com`) produces noisy results. IANA/ICANN reserved domains may return 403 intermittently due to rate limiting or bot filtering, creating flapping outage records. Useful as a cautionary demo example.

## Deleting checks

Use `delete_check` only for permanent removal. Deletion is irreversible — the check's history (outages, response times) is lost. Prefer pausing for temporary deactivation.

When deleting multiple checks, batch all `delete_check` calls in parallel.

## CloudStatus checks

CloudStatus (also known as **Third-party monitoring** in Uptime.com's product) is a check type that monitors a cloud provider's status page natively within Uptime.com. Use the MCP server's tools to discover available providers and service components.

CloudStatus provides native status mapping (UP/DOWN/MAINTENANCE), proper alerting integration, and structured historical data. Use it for upstream dependency monitoring.

## Known MCP server issues

- **`get_contact` type parsing**: may fail with integer/string type mismatch on the `id` parameter. Use `list_contacts` as a workaround for basic info.
