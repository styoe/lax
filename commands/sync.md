---
description: Full Ticket <-> GitHub reconciliation since last sync
argument-hint: "[since-date]"
---

Read `${CLAUDE_PLUGIN_ROOT}/workflows/sync.md` and execute it verbatim. Wherever the workflow refers to "the subcommand argument", use the value of `$ARGUMENTS` (an optional ISO-8601 / `YYYY-MM-DD` date). Do not paraphrase — follow the workflow exactly, including its approval-loop discipline (dry-run, present diff, wait for explicit approval, then apply).
