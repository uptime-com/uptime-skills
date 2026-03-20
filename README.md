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

Select the `uptime` server and follow the browser login flow. Tokens are stored securely and refreshed automatically.

### Updating

```bash
/plugin update uptime@uptime-com
```

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
    "allow": ["mcp__uptime__list_checks", "mcp__uptime__list_tags", "mcp__uptime__list_contacts", "mcp__uptime__list_alerts", "mcp__uptime__list_outages", "mcp__uptime__get_check", "mcp__uptime__get_status_page", "mcp__uptime__get_account_usage"]
  }
}
```

## What's included

### MCP Server

Connects Claude Code to the [Uptime.com MCP server](https://support.uptime.com/hc/en-us/articles/30498498498964-MCP-Model-Context-Protocol) with OAuth authentication.

### Skills

Skills are auto-invoked by Claude based on conversation context.

| Skill                         | Triggered by                                        |
| ----------------------------- | --------------------------------------------------- |
| **monitoring-planning**       | "set up monitoring", "plan checks for domain"       |
| **understanding-check-types** | "what check types", "how does HTTP check work"      |
| **transaction-scripting**     | "create transaction check", "add smoke test"        |
| **api-scripting**             | "create API check", "API monitoring script"         |
| **monitoring-optimization**   | "audit monitoring", "optimize checks", "fill gaps"  |
| **performance-reporting**     | "what's our uptime", "SLA report", "monthly report" |
| **incident-triage**           | "site is down", "investigate outage"                |
| **dashboard-management**      | "create dashboard", "add widgets"                   |
| **status-page-management**    | "set up status page", "add components"              |

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- [Uptime.com](https://uptime.com) account

## Contributing

<!-- TODO -->

## License

MIT
