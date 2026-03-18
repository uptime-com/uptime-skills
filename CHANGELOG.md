# Changelog

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
