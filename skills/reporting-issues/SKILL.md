---
name: reporting-issues
description: >-
  This skill should be used when an instruction in one of the uptime plugin's skills, references or agents contradicts what an Uptime.com MCP tool accepts or returns, when those skills have no guidance the task needs, or when the user asks to "report a bug", "report an issue with the plugin", "request a feature", or "file feedback on the skills". For a fault in the MCP server itself, use the server's search_issues and report_issue tools instead.
---

# Reporting issues with the skill package

The uptime plugin's skills, references and agents are tracked at `https://github.com/uptime-com/uptime-skills/issues`. When one of them is wrong, inconsistent or missing something, tell the user and offer to draft a report there.

## Which tracker

| What went wrong                                                                    | Tracker                                                |
| ---------------------------------------------------------------------------------- | ------------------------------------------------------ |
| A skill names a tool, field or value the server does not have, or rejects          | uptime-skills (this skill)                             |
| A skill's workflow, advice or example contradicts another skill or the tool output | uptime-skills                                          |
| A task the user wants has no skill guidance, or a skill should cover more          | uptime-skills, as a feature request                    |
| A tool fails, returns contradictory output, or a needed tool does not exist        | uptime-mcp, through `search_issues` and `report_issue` |

When unsure, the test is whether the server behaved as its own tool description says. If it did, the skill is what needs fixing.

## Workflow

1. **Search first.** Open `https://github.com/uptime-com/uptime-skills/issues?q=is%3Aissue+<keywords>` for the user, with the skill name and the symptom as keywords, or run `gh issue list -R uptime-com/uptime-skills --state all --search "<keywords>"` where `gh` is available. If an issue, open or closed, describes the same thing, the report belongs there as a comment.
2. **Draft** the report with the user's agreement, in the shape below.
3. **Hand over a link** the user opens, reviews and submits. Nothing is filed without them.

URL-encode every value in the links below. The title is one line naming the skill and the fault or the missing guidance, e.g. `monitoring-optimization: names a nonexistent tool`.

### Bug or inconsistency

The repository's bug form, with its fields prefilled by `id`:

```
https://github.com/uptime-com/uptime-skills/issues/new?template=bug_report.yml&title=<title>&component=<component>&trigger-prompt=<prompt>&expected=<expected>&actual=<actual>&mcp-errors=<errors>
```

| Field            | Content                                                                                                   |
| ---------------- | --------------------------------------------------------------------------------------------------------- |
| `component`      | Exactly one of `skill: <skill-name>`, `agent: <agent-name>`, `reference material`, `plugin configuration` |
| `trigger-prompt` | The user's request that led there                                                                         |
| `expected`       | What the skill said, quoted, with the file it came from                                                   |
| `actual`         | What the tool accepted or returned instead                                                                |
| `mcp-errors`     | Any tool error text, verbatim                                                                             |

The form also asks for the Claude Code version (`claude --version`) and the model; leave those for the user to fill in.

### Feature request

```
https://github.com/uptime-com/uptime-skills/issues/new?title=<title>&body=<body>
```

The body names the task the user was doing, what guidance was missing, and what the skill should have said.

## Before the user submits

Issues are public. Point out anything in the draft that looks sensitive or identifying, such as tokens, email addresses, phone numbers, domains, URLs, IP addresses, or account, contact, check and status page names. Whether to keep it is the user's call.
