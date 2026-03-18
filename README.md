# Uptime.com Skills for Claude Code

AI agent skills and context for working with [Uptime.com](https://uptime.com) monitoring infrastructure.

## Installation

```bash
/plugin install uptime@claude-plugins-official
```

<!-- TODO: team installation via .claude/settings.json -->

## What's included

### MCP Server

Connects Claude Code to
the [Uptime.com MCP server](https://support.uptime.com/hc/en-us/articles/30498498498964-MCP-Model-Context-Protocol) with
OAuth authentication.

### Skills

Skills are auto-invoked by Claude based on conversation context.

| Skill                      | Triggered by                                 |
|----------------------------|----------------------------------------------|
| **monitoring-setup**       | "set up monitoring", "add checks for domain" |
| **check-management**       | "update check", "change alert contacts"      |
| **incident-triage**        | "site is down", "investigate outage"         |
| **monitoring-audit**       | "audit monitoring", "review check coverage"  |
| **dashboard-management**   | "create dashboard", "add widgets"            |
| **status-page-management** | "set up status page", "add components"       |

### Reference materials

Supporting context automatically loaded by skills:

- **check-types** — check type matrix with capabilities and use cases
- **domain-monitoring-checklist** — standard monitoring patterns per domain
- **upstream-dependencies** — detecting and monitoring third-party dependencies

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- [Uptime.com](https://uptime.com) account

## Contributing

<!-- TODO -->

## License

MIT
