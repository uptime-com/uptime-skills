---
name: transaction-scripting
description: >-
  This skill should be used when the user asks to "create a transaction check", "write a browser test", "add a smoke test", "synthetic monitoring script", "transaction check script", "browser automation check", "test signup flow", "test login flow", or needs to author multi-step browser interaction scripts for Uptime.com Transaction checks.
---

# Transaction check scripting

Transaction checks monitor web user flows by executing sequential steps in a real Chromium browser. Each step is a command (action) or validation (assertion). If any step fails, execution stops and an alert is raised.

See `references/scripting-txn.md` for the complete step type reference, selector formats, variable system, and pagespeed integration.
