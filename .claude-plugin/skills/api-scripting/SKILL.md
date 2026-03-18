---
name: api-scripting
description: >-
  This skill should be used when the user asks to "create an API check", "write an API monitoring script", "multi-step API test", "API check script", "monitor API endpoint", "test API flow", or needs to author multi-step HTTP request sequences with assertions for Uptime.com API checks.
---

# API check scripting

API checks execute multi-step HTTP request sequences with assertions. Scripts are JSON arrays of step objects executed sequentially. If any assertion fails, execution stops and an alert is raised.

See `references/scripting-api.md` for the complete step type reference, variable system, selector formats, and authentication patterns.
