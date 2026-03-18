# Changelog

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
