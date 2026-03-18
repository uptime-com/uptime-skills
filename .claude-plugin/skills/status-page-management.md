---
name: status-page-management
description: >-
  This skill should be used when the user asks to "create a status page",
  "set up a status page", "add a component to status page",
  "post an incident", "update incident status", "manage status page",
  "public status page", or needs to create and manage public or private
  status pages with components and incidents.
---

# Status page management — operational knowledge

Workflow patterns for creating status pages, mapping components to checks,
and managing incidents through their lifecycle.

## Status page creation workflow

### Step 1 — Plan components

A status page represents services as **components** that map to monitoring checks.
Plan the component structure before creating:

| Component | Maps to | Rationale |
|-----------|---------|-----------|
| Website | HTTP check (main site) | User-facing web availability |
| API | HTTP check (API endpoint) | Developer/integration availability |
| DNS | DNS A check | Name resolution |
| Email | SMTP check + DNS MX | Mail delivery chain |
| CDN / Static Assets | HTTP check (CDN URL) | Asset delivery |

Group related components. Users of your status page don't need to see every
internal check — aggregate into meaningful service categories.

### Step 2 — Create status page

`create_status_page` with:

- **Name**: public-facing name (e.g. "Acme Corp Status")
- **Subdomain or custom domain**: the URL where the page will be accessible
- **Visibility**: public (anyone) or private (authenticated users)

### Step 3 — Add components

For each planned component:

- Create the component on the status page
- Link it to the corresponding monitoring check(s)
- Set the initial status (operational)

### Step 4 — Verify

- `get_status_page` to confirm the page is configured correctly
- Components should reflect current check states

## Component mapping patterns

### Simple mapping (1:1)

One component per check. Works for small setups:

```
Website → HTTP check
API → HTTP check (API)
Database → TCP check (port 5432)
```

### Aggregated mapping (N:1)

Multiple checks feed one component. Better for public-facing pages where
users don't need internal granularity:

```
Website → HTTP + SSL + DNS A + Page Speed
  (component is "degraded" if any sub-check alerts)

Email → SMTP + DNS MX + Blacklist
  (component is "down" if SMTP fails, "degraded" if Blacklist alerts)
```

### Tiered components

Use component groups to organize by service tier:

```
Core Services:
  - Website
  - API
  - Authentication

Infrastructure:
  - DNS
  - CDN
  - Email

Monitoring:
  - SSL Certificates
  - Domain Registration
```

## Incident lifecycle

Status page incidents communicate service disruptions to users. They follow
a defined lifecycle:

### 1. Create incident

When an outage is confirmed (not just a single alert):

- **Title**: clear, user-facing description (e.g. "Website experiencing intermittent errors")
- **Status**: `investigating`
- **Affected components**: set to appropriate state (`degraded_performance`, `partial_outage`, `major_outage`)
- **Message**: what you know so far

### 2. Update incident

As investigation progresses, post updates:

| Status | When to use |
|--------|-------------|
| `investigating` | Initial state — looking into the issue |
| `identified` | Root cause found, working on fix |
| `monitoring` | Fix deployed, watching for recurrence |
| `resolved` | Incident is over, service restored |

Each update should include a message explaining what changed.

### 3. Resolve incident

When the issue is confirmed fixed:

- Set status to `resolved`
- Set affected components back to `operational`
- Include a brief summary of what happened and what was done

## Public vs private status pages

| Aspect | Public | Private |
|--------|--------|---------|
| Audience | Customers, users, the internet | Internal teams, specific stakeholders |
| Component detail | Aggregated, user-friendly names | Can be more granular/technical |
| Incident language | Non-technical, user-impact focused | Can include technical details |
| Creation requires | User confirmation (public-facing asset) | Less formal |

**Always ask before creating a public status page** — it's a commitment to
maintain and represents the organization publicly.

## Best practices

- **Don't auto-create status pages**: unlike checks and dashboards, status pages
  are public-facing. Always confirm with the user first.
- **Keep components user-facing**: "Website" not "HTTP check on www.example.com"
- **Update incidents promptly**: stale "investigating" status erodes trust
- **Use component groups**: organize by service area, not by check type
- **Map critical checks**: every component should have at least one monitoring
  check backing it so component status can be automated