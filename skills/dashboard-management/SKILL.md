---
name: dashboard-management
description: >-
  This skill should be used when the user asks to "create a dashboard", "set up a dashboard", "organize monitoring views", "build a dashboard", "add checks to a dashboard", "monitoring dashboard", or wants to create visual overviews of their monitoring checks. Covers dashboard scoping, check selection, and section settings.
---

# Dashboard management

Workflow patterns for creating and organizing monitoring dashboards.

## Dashboard model

A dashboard has no widgets. It is a set of checks shown in up to three sections, each switched on or off:

| Section  | Toggle                  | What it shows                       | Settings                                                                                         |
| -------- | ----------------------- | ----------------------------------- | ------------------------------------------------------------------------------------------------ |
| Services | `services_show_section` | The selected checks and their state | `services_num_to_show`, `services_primary_sort`, `services_secondary_sort`, `services_include_*` |
| Alerts   | `alerts_show_section`   | Recent alerts                       | `alerts_num_to_show` (5, 10, 15), `alerts_for_all_checks`                                        |
| Metrics  | `metrics_show_section`  | Metrics for the selected checks     | `metrics_for_all_checks`                                                                         |

Checks are selected two ways, combinable:

- `services_tags`: tag names; every check carrying one is included, including checks tagged later.
- `services_selected`: check ids from `list_checks`, for a fixed set.

`services_include_up`, `services_include_down`, `services_include_paused` and `services_include_maintenance` filter the services section by state. `is_pinned` pins the dashboard to the top of the list.

Tag selection is the default choice: it keeps the dashboard current as checks come and go.

## Dashboard creation workflow

### Step 1: determine scope

| Pattern                | Selection                                               | Use case                                         |
| ---------------------- | ------------------------------------------------------- | ------------------------------------------------ |
| Domain dashboard       | `services_tags`: the domain tag                         | Per-service visibility                           |
| Team dashboard         | `services_tags`: the team's tags                        | Team-level overview                              |
| Service tier dashboard | `services_tags`: e.g. `production`                      | Executive or on-call view                        |
| Check type dashboard   | `services_selected`: ids from `list_checks` with `type` | Focused monitoring (certificate expiry overview) |

### Step 2: check capacity and gather checks

Call `get_account_usage` to check whether dashboard limits apply to the account. If limits exist and are near capacity (>80%), warn the user before creating a dashboard.

- `list_tags` to find the tags for a tag-scoped dashboard, or to discover the grouping structure if scope isn't clear.
- `list_checks` with `type` (and `tag`) for the ids of a check-type dashboard.
- `list_dashboards` to see whether a dashboard for this scope already exists.

### Step 3: create dashboard

`create_dashboard` with `name` and the selection from Step 1. Section settings by audience:

| Audience  | Settings                                                                                                                  |
| --------- | ------------------------------------------------------------------------------------------------------------------------- |
| Overview  | Services on, sorted `is_paused,cached_state_is_up` so down checks come first; alerts on                                   |
| On-call   | Services on with `services_include_up: false`, so only checks needing attention show; alerts on, `alerts_num_to_show: 15` |
| Executive | Services and metrics on; alerts off                                                                                       |

The sort values the tools accept: `cached_ordering`, `device__address`, `-created_at`, `is_paused,cached_state_is_up`, `-cached_last_down_alert_at`, `-cached_response_time`.

## Naming conventions

- `{domain} monitoring`: per-domain dashboards
- `{team} overview`: team dashboards
- `{purpose} dashboard`: special-purpose (e.g. "SSL expiry dashboard")

## Updating existing dashboards

`update_dashboard` takes the dashboard `id` and changes only the fields provided. `services_tags` and `services_selected` replace the whole list, so include the entries to keep.

1. `get_dashboard` for the current selection and settings.
2. For a tag-scoped dashboard, new checks appear once tagged; nothing to update.
3. For an id-scoped dashboard, pass the full new `services_selected`: drop deleted checks, add new ones.

Create a new dashboard instead of updating when the scope itself changed (a domain split into several services, a new team took ownership).

## Deleting dashboards

`delete_dashboard` is permanent. Before deleting, confirm the dashboard isn't referenced by other team members or linked from external tools (e.g. Slack bookmarks, wiki pages).

## Linking dashboards to setup workflow

When `monitoring-planning` creates checks for a new domain, Phase 4 creates a dashboard selected by the domain tag, so every monitored domain has one from day one.
