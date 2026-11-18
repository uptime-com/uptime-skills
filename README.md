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

Select the `uptime` server and follow the browser login flow. Claude Code stores the token and refreshes it. The server accepts only this OAuth sign-in; Uptime.com account API tokens are not accepted.

### Updating

```bash
/plugin update uptime@uptime-com
```

### Team installation

Add to your project's `.claude/settings.json`. The `allow` list pre-approves the read-only tools; tools that create, change or delete prompt as usual.

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
    "allow": ["mcp__plugin_uptime_uptime__list_checks", "mcp__plugin_uptime_uptime__get_check", "mcp__plugin_uptime_uptime__get_check_stats", "mcp__plugin_uptime_uptime__list_tags", "mcp__plugin_uptime_uptime__get_tag", "mcp__plugin_uptime_uptime__list_contacts", "mcp__plugin_uptime_uptime__get_contact", "mcp__plugin_uptime_uptime__list_alerts", "mcp__plugin_uptime_uptime__get_alert", "mcp__plugin_uptime_uptime__list_outages", "mcp__plugin_uptime_uptime__get_outage", "mcp__plugin_uptime_uptime__list_locations", "mcp__plugin_uptime_uptime__get_location", "mcp__plugin_uptime_uptime__list_dashboards", "mcp__plugin_uptime_uptime__get_dashboard", "mcp__plugin_uptime_uptime__list_status_pages", "mcp__plugin_uptime_uptime__get_status_page", "mcp__plugin_uptime_uptime__list_status_page_components", "mcp__plugin_uptime_uptime__get_status_page_component", "mcp__plugin_uptime_uptime__list_status_page_incidents", "mcp__plugin_uptime_uptime__get_status_page_incident", "mcp__plugin_uptime_uptime__list_cloudstatus_providers", "mcp__plugin_uptime_uptime__search_cloudstatus_services", "mcp__plugin_uptime_uptime__get_account_usage", "mcp__plugin_uptime_uptime__search_issues"]
  }
}
```

## What's included

### MCP Server

Connects Claude Code to the hosted [Uptime.com MCP server](https://github.com/uptime-com/uptime-mcp) at `https://mcp.uptime.com/mcp`, with OAuth authentication. Set `UPTIME_MCP_URL` to point the plugin at another instance.

When a tool misbehaves, the server's `search_issues` and `report_issue` tools draft a GitHub issue for you to review and submit; nothing is filed without you.

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
| **dashboard-management**      | "create dashboard", "add checks to a dashboard"     |
| **status-page-management**    | "set up status page", "add components"              |
| **reporting-issues**          | a skill contradicts a tool, "report a bug"          |

### Agents

| Agent                     | Handles                                                              |
| ------------------------- | -------------------------------------------------------------------- |
| **transaction-scripting** | Writing Transaction check scripts, with the step reference preloaded |
| **api-scripting**         | Writing API check scripts, with the step reference preloaded         |

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- [Uptime.com](https://uptime.com) account

## Reporting issues

Report wrong or inconsistent skill guidance, and request new guidance, in this repository's [issue tracker](https://github.com/uptime-com/uptime-skills/issues). Claude drafts the report for you when a skill turns out to be wrong; you review and submit it.

Faults in the MCP server itself go to the [uptime-mcp tracker](https://github.com/uptime-com/uptime-mcp/issues).

## License

MIT
