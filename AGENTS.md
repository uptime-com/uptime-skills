# Agent instructions

## Formatting

Run `prettier --write "**/*.md"` before committing any markdown changes. All markdown files must be prettier-formatted.

## Repository structure

```
.claude-plugin/
├── plugin.json          # Plugin metadata (name, description)
└── marketplace.json     # Marketplace registry for plugin discovery
skills/
└── <skill-name>/
    ├── SKILL.md             # Skill file — auto-invoked by Claude based on context
    └── references/          # Supporting data auto-loaded with the skill
agents/
└── <agent-name>.md          # Agent definitions with preloaded skills
.mcp.json                    # MCP server configuration (Uptime.com API via OAuth)
```

- **Skills** are the core product. Each skill is a directory containing `SKILL.md` with YAML frontmatter (`name`, `description`) and operational knowledge.
- **References** are data files (check type matrix, checklists, scripting guides) nested under each skill's `references/` directory. They auto-load when the skill activates. References may be duplicated across skills that share them.
- **`.mcp.json`** connects Claude Code to the Uptime.com MCP server. The `UPTIME_MCP_URL` env var overrides the default endpoint for staging/dev.

## Skill authoring conventions

### Frontmatter

Every skill and reference file must have YAML frontmatter with:

```yaml
---
name: kebab-case-name
description: >-
  Multi-line description. For skills: starts with "This skill should be used when the user asks to..." followed by quoted trigger phrases. For references: starts with "Reference document for..." and lists consumer skills.
---
```

### Content guidelines

- Write in natural language. Use lowercase `should`, `must`, `never` — not RFC 2119 caps (MUST, SHALL).
- Exception: use `Do NOT` for API constraints that cause validation errors (e.g., "do NOT pass `locations` to auto-located checks").
- Keep skill body under 500 lines. Move detail into reference files.
- Use tables for structured data (parameters, field references, gap matrices).
- Use step-numbered workflows for multi-step processes.
- MCP tool names in backticks and `snake_case` (e.g., `list_checks`).
- When Uptime.com marketing names differ from API/engineering names, bridge them explicitly: `CloudStatus (marketed as **Third-party monitoring**)`.

### Cross-references

- Reference files by relative path: `` `references/check-types.md` ``
- Reference skills by name: "see `check-management` skill"
- Keep descriptions updated — list all consumer skills in reference frontmatter.

### Trigger phrase boundaries

Each skill owns a clear intent boundary. When triggers could overlap, add a disambiguation line in the description:

```yaml
description: >-
  ...Covers initial check creation... For modifying existing checks, see check-management.
```

## Terminology

Prefer API/engineering names throughout skills. Bridge marketing names on first mention only.

| Marketing name         | Engineering name           | Notes                        |
| ---------------------- | -------------------------- | ---------------------------- |
| Third-party monitoring | CloudStatus                | Status page feed monitoring  |
| Synthetic Monitoring   | Transaction + API checks   | Umbrella marketing term      |
| Real User Monitoring   | RUM check                  | JavaScript snippet-based     |
| Heartbeat monitoring   | Custom check               | Push-based, not probe-based  |
| Domain Health          | Blacklist + Malware checks | DNS blacklist + malware scan |

## Adding a new skill

1. Create `.claude-plugin/skills/<skill-name>/SKILL.md` with frontmatter.
2. Ensure trigger phrases don't overlap with existing skills.
3. Follow the step-numbered workflow pattern used by other skills.
4. If the skill needs reference data, add files to `<skill-name>/references/`. References auto-load when the skill activates.
5. Add the skill to the README table with trigger examples.

## Adding a new reference

1. Create the reference file under the consuming skill's `references/` directory.
2. If multiple skills need the same reference, copy it to each skill's `references/`.
3. Keep references under 500 lines. Use a table of contents if over 100 lines.

## Release process

### When to release

Tag a new version when a meaningful set of skill changes lands on `main` — new skills, updated reference materials, or corrected workflows.

### Steps

1. **Decide the version bump** using [EffVer](https://effver.org) — version by the effort required from users to adopt the change:
   - **micro** (0.0.x): no effort — typo fixes, wording improvements, minor corrections
   - **meso** (0.x.0): small effort — new skills, new references, changed trigger phrases, major skill rewrites
   - **macro** (x.0.0): significant effort — renamed/removed skills, restructured references, changed plugin name

2. **Update CHANGELOG.md** — prepend a new entry following Keep a Changelog format:

   ```markdown
   ## [X.Y.Z] - YYYY-MM-DD

   ### Added / Changed / Fixed / Removed

   - Description of change
   ```

   Write entries from the user's perspective, not implementation details. Bad: "Updated check-types.md line 49" Good: "Added marketing name bridges for Synthetic Monitoring and Domain Health"

3. **Commit and push** with message: `Release vX.Y.Z`

4. **Create a GitHub Release** — this creates the tag automatically. Use the current version's CHANGELOG section as the release body:
   ```bash
   gh release create vX.Y.Z --title vX.Y.Z --notes "<paste current version's CHANGELOG section>"
   ```
