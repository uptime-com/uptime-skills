# Uptime.com Skills for Claude Code

AI agent skills and context for working with [Uptime.com](https://uptime.com) monitoring infrastructure.

## Installation

Add the marketplace and install the plugin:

```bash
/plugin marketplace add uptime-com/uptime-skills
/plugin install uptime@uptime-com
```

After installation, authenticate with the Uptime.com MCP server:

```bash
/mcp
```

Select the `uptime` server and follow the browser login flow. Tokens are
stored securely and refreshed automatically.

### Team installation

Add to your project's `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "uptime-com": {
      "source": { "source": "github", "repo": "uptime-com/uptime-skills" }
    }
  },
  "enabledPlugins": {
    "uptime@uptime-com": true
  },
  "permissions": {
    "allow": [
      "mcp__uptime__list_checks",
      "mcp__uptime__list_tags",
      "mcp__uptime__list_contacts",
      "mcp__uptime__list_alerts",
      "mcp__uptime__list_outages",
      "mcp__uptime__get_check",
      "mcp__uptime__get_status_page"
    ]
  }
}
```

## What's included

### MCP Server

Connects Claude Code to
the [Uptime.com MCP server](https://support.uptime.com/hc/en-us/articles/30498498498964-MCP-Model-Context-Protocol) with
OAuth authentication.

### Skills

Skills are auto-invoked by Claude based on conversation context.

| Skill                      | Triggered by                                 |
|----------------------------|----------------------------------------------|
| **monitoring-setup**       | "set up monitoring", "add smoke test", "create transaction check" |
| **check-management**       | "update check", "edit transaction script"    |
| **incident-triage**        | "site is down", "investigate outage"         |
| **monitoring-audit**       | "audit monitoring", "review check coverage"  |
| **dashboard-management**   | "create dashboard", "add widgets"            |
| **status-page-management** | "set up status page", "add components"       |

### Reference materials

Supporting context automatically loaded by skills:

- **check-types** — check type matrix with capabilities and use cases
- **scripting-txn** — Transaction Check step types, selectors, variables, and validations
- **scripting-api** — API Check request steps, selectors, variables, and assertions
- **checklist-domain-monitoring** — standard monitoring patterns per domain
- **upstream-dependencies** — detecting and monitoring third-party dependencies

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- [Uptime.com](https://uptime.com) account

## Contributing

<!-- TODO -->

## License

MIT
