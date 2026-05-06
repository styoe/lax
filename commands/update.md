---
description: Fill a single Ticket from GitHub activity
argument-hint: "<ticket-id>"
---

Read `${CLAUDE_PLUGIN_ROOT}/workflows/update.md` and execute it verbatim. Wherever the workflow refers to "the subcommand argument", use the value of `$ARGUMENTS` (the Ticket ID, required). Do not paraphrase — follow the workflow exactly, including its approval-loop discipline (dry-run, present diff, wait for explicit approval, then apply).
