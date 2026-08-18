<!--
This file is managed by Terraform (uptf repo, github/templates/pull_request.md).
Edits made directly in this repository will be overwritten on the next apply.

Soft guideline, not hard requirement. Fill relevant sections, remove the rest,
delete these comments.

Prose volume tracks payoff, not effort. The diff ships alongside this
description, so never re-describe it: write only what the diff cannot show.
Length is earned by a constraint that forced the design, an approach tried and
rejected, an ordering requirement for rollout, or a behavior change with blast
radius. A typo fix, a dependency bump, or a rename gets a title and a line.
Delete every section that would only restate the diff.

Guidance for AI agents authoring this PR:
- Write each section from the actual diff and commit history, not assumptions.
- Be concrete and itemized. State observable behavior changes, not intentions.
- Delete sections that do not apply (e.g. Deployment notes when none).
- Fill the Agent sign-off section. It is not optional and is not deletable.

PR title MUST follow Conventional Commits:
  <type>(<scope>): <short description>
  e.g. feat(scheduler): label emitted-requests counter

  type: fix | feat | build | chore | ci | docs | style | refactor | perf | test | revert
  Breaking change: append ! before the colon (feat!:).
  chore, revert, build commits are excluded from release notes.
-->

## What

<!-- Short summary of what this PR changes. Itemized list helps the reviewer. -->

## Why

<!-- Motivation or context. Link related issues ("Closes #86") and PRs. -->

## How

<!-- Implementation notes: design, trade-offs, things to watch. -->

## Testing

<!-- What you ran / how the reviewer can verify. Logs, screenshots, dashboards. -->

## Deployment notes

<!-- Optional. Rollout impact, config/secret changes, rollback, breaking changes.
     Delete if none. -->

## Agent sign-off

<!-- Required when an AI agent authored any part of this change. Delete the
     section only for a change written entirely by a human. Tick a box only
     after verifying it line by line against the final diff, not against what
     you set out to do. If a box does not hold, fix the diff; do not silently
     untick it or delete the section. An unfilled, deleted, or falsely ticked
     sign-off is grounds for closing this PR without review. -->

- Model: <!-- name and version, e.g. claude-opus-5 -->
- Harness: <!-- name and version, e.g. Claude Code 2.0.31 -->

- [ ] Diff self-reviewed for antipatterns and slop: no dead or commented-out
      code, no speculative abstraction, no copy-paste duplication, nothing
      outside the requested scope
- [ ] Comments self-reviewed: none restate what the code does, none narrate the
      change event ("now uses X", "previously...", "refactored to...", "fixes
      the bug where..."). Standing explanation lives in package and symbol
      documentation and in tests; change-event explanation lives in this
      description
