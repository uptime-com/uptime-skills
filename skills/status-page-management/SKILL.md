---
name: status-page-management
description: >-
  This skill should be used when the user asks to "create a status page", "set up a status page", "add a component to status page", "post an incident", "schedule maintenance on a status page", "manage status page", "public status page", or needs to create and manage public or private status pages with components and incidents.
---

# Status page management

Workflow patterns for creating status pages, mapping components to checks, and posting incidents.

## Status page creation workflow

### Step 1: plan components

A status page represents services as **components**. A component links to at most one check through `service_id`; a **group component** (`is_group: true`) holds other components through their `group_id`. Plan the structure before creating:

| Component           | Linked check              | Rationale                          |
| ------------------- | ------------------------- | ---------------------------------- |
| Website             | HTTP check (main site)    | User-facing web availability       |
| API                 | HTTP check (API endpoint) | Developer/integration availability |
| DNS                 | DNS A check               | Name resolution                    |
| Email               | SMTP check                | Mail delivery                      |
| CDN / Static Assets | HTTP check (CDN URL)      | Asset delivery                     |

Users of a status page don't need every internal check. Where several checks back one service, link the component to a Group check that aggregates them (see `monitoring-planning`), rather than to one of its members.

### Step 2: create status page

Always ask before creating a public status page: it represents the organization publicly and is a commitment to maintain.

`create_status_page` with:

| Field                                      | Values                                                 |
| ------------------------------------------ | ------------------------------------------------------ |
| `name`                                     | Public-facing name (e.g. "Acme Corp Status")           |
| `page_type`                                | `PUBLIC`, `PUBLIC_SLA`, or `INTERNAL`                  |
| `visibility_level`                         | `PUBLIC` (anyone), `UPTIME_USERS`, or `EXTERNAL_USERS` |
| `slug`, `cname`                            | The page's URL slug, or a custom domain pointed at it  |
| `allow_subscriptions`                      | Lets visitors subscribe to incident notifications      |
| `timezone`, `contact_email`, `description` | Shown on the page                                      |

### Step 3: add components

For each planned component, `create_status_page_component` with `status_page_id`, `name`, `description`, and `service_id` (the check id from `list_checks`). Create group components first, then pass their id as `group_id` on the members.

### Step 4: verify

`get_status_page` for the page settings, `list_status_page_components` for the components and their current status, and `get_status_page_component` for a component's linked check (`ServiceID`) and group.

## Tiered components

Group components organize the page by service tier:

```
Core Services (group)
  - Website      → HTTP check
  - API          → HTTP check
  - Authentication → HTTP check (login endpoint)

Infrastructure (group)
  - DNS          → DNS A check
  - CDN          → HTTP check (CDN URL)
  - Email        → SMTP check
```

## Incidents

`create_status_page_incident` with `status_page_id` and `name`, plus:

| Field                     | Meaning                                                            |
| ------------------------- | ------------------------------------------------------------------ |
| `incident_type`           | `INCIDENT` (default) or `SCHEDULED_MAINTENANCE`                    |
| `description`             | The first update's text; it is posted in the `investigating` state |
| `starts_at`, `ends_at`    | ISO 8601 (`YYYY-MM-DDThh:mm:ssZ`); `starts_at` defaults to now     |
| `update_component_status` | Whether the incident updates component status                      |
| `notify_subscribers`      | Whether subscribers are notified                                   |

Create an incident when an outage is confirmed, not on a single alert. Write the `name` and `description` for the page's audience: user impact on a public page, technical detail on an internal one.

For planned work, `SCHEDULED_MAINTENANCE` with `starts_at` and `ends_at` set in the future.

`update_status_page_incident` changes only `name`, `starts_at` and `ends_at`. The tools cannot post follow-up updates (identified, monitoring, resolved) or set component status; tell the user to post those in the Uptime.com web app, and offer the update text. `get_status_page_incident` shows the updates posted so far and the affected components.

## Best practices

- **Confirm before creating**: unlike checks and dashboards, status pages are public-facing.
- **Keep components user-facing**: "Website", not "HTTP check on www.example.com".
- **Back every component with a check** through `service_id`, so its status can follow monitoring.
- **Update incidents promptly**: a stale "investigating" erodes trust; remind the user to post updates as the investigation moves.
