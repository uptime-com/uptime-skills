# Changelog

## [Unreleased]

### Added

- reporting-issues skill: when a skill contradicts the MCP tools or lacks guidance, Claude drafts a GitHub issue for this repository for you to review and submit

### Changed

- Signing in to the Uptime.com MCP server no longer uses a pre-registered OAuth client id; Claude Code identifies itself to uptime.com on its own
- Skills, references and agents match the tools of Uptime.com MCP server 0.18.0: per-type `update_<type>_check` tools, `email_list`/`sms_list` on contacts, `threshold` for SSL, WHOIS and RDAP expiry warnings, `service_name` and `locations` for CloudStatus
- dashboard-management describes dashboards as check selections (by tag or check id) with services, alerts and metrics sections, replacing the widget model the server does not have
- status-page-management covers components linked to checks through `service_id`, group components, and the `INCIDENT`/`SCHEDULED_MAINTENANCE` incident types, and hands follow-up incident updates to the web app
- Skills name the settings the MCP tools cannot change (escalations, maintenance windows, SLA targets, pausing most check types) and send the user to the web app for them
- README permission example uses the plugin's tool names and lists every read-only tool

### Removed

- The `get_contact` known-issue workaround

## [0.5.1] - 2026-03-18

### Changed

- Inlined scripting references directly into transaction-scripting and api-scripting SKILL.md files
- Removed separate reference files for scripting skills (content now in skill body)
- Agents preload skills with full content, eliminating filesystem search at runtime

## [0.5.0] - 2026-03-18

### Added

- Transaction scripting agent: delegates to isolated context with full scripting reference preloaded
- API scripting agent: delegates to isolated context with full scripting reference preloaded
- Agents preload corresponding skills via `skills:` frontmatter, guaranteeing reference auto-load

## [0.4.0] - 2026-03-18

### Added

- New skills: monitoring-planning, understanding-check-types, transaction-scripting, api-scripting, monitoring-optimization, performance-reporting
- Performance reporting skill with SLA evaluation, trend analysis, and regional breakdowns
- False positive tuning and false negative detection guidance in incident-triage
- Alert ignoring workflow (`ignore_alert`) in incident-triage
- Contact creation workflow (Phase 0) in monitoring-planning

### Changed

- Redesigned skill set: 9 skills centered around references, zero duplication
- Restructured to directory-per-skill pattern (SKILL.md + references/) for auto-loading
- Merged monitoring-audit and check-management into monitoring-optimization
- Inlined upstream dependency content into monitoring-planning and incident-triage

### Removed

- monitoring-setup, monitoring-audit, check-management (consolidated into new skills)
- Shared references directory (references now live under each skill)
- upstream-dependencies.md reference file (content inlined)

## [0.3.0] - 2026-03-18

### Added

- Auth and read-only tool permissions documentation in README
- Direct scripting references in monitoring-setup and check-management skills
- Trigger phrases for transaction checks, smoke tests, and API checks
- Prettier and EditorConfig for consistent markdown formatting

### Fixed

- Skills now reference scripting-txn and scripting-api directly (one hop, not two)

## [0.2.0] - 2026-03-18

### Added

- Transaction Check scripting reference (scripting-txn): step types, selectors, variables, validations, pagespeed
- API Check scripting reference (scripting-api): HTTP request steps, selectors, variables, assertions, authentication patterns

### Changed

- Renamed reference files for consistent naming: checklist-domain-monitoring, scripting-txn, scripting-api

## [0.1.0] - 2026-03-18

### Added

- Six skills: monitoring-setup, check-management, incident-triage, monitoring-audit, dashboard-management, status-page-management
- Three reference documents: check-types matrix, domain-monitoring-checklist, upstream-dependencies
- Marketing-to-engineering name bridges: Synthetic Monitoring, Real User Monitoring, Heartbeat, Domain Health, Third-party monitoring
- Marketplace manifest for self-hosted plugin distribution
- MCP server configuration with OAuth authentication
- AGENTS.md with skill authoring conventions and release process
