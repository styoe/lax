---
description: Verify ticket provider + GitHub auth and bootstrap lax plugin state (or jump straight to the permissions step with arg "permissions")
argument-hint: "[permissions]"
---

Read `${CLAUDE_PLUGIN_ROOT}/workflows/setup.md` and execute it verbatim. Do not paraphrase — follow its instructions exactly, including its approval-loop discipline (dry-run, present diff, wait for explicit approval, then apply). If `$ARGUMENTS` equals `permissions`, the workflow's argument-shortcut sends control straight to the permissions step (step 8) and skips the rest.
