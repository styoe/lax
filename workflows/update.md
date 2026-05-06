# lax: update

Fill in a single Ticket using GitHub evidence, scoped to the **active lax project**.

Throughout this file:
- **Ticket** (capital T) = a single record in the configured ticket management software.
- **Project** (capital P) = the workspace/team/board grouping in that software where Tickets live.

The subcommand argument is the **Ticket ID** (required). If it's missing, ask the user for it before proceeding.

## 1. Load config + Ticket + permission check

Read `${CLAUDE_PLUGIN_DATA}/config.json` and resolve `projects[active_project]` to get `github_repo`, `ticket_project_id`, and `allowed_packs`. Also read the global `ticket_provider`. If config is missing or has no active project, stop and tell the user to run `/lax:setup` first.

**Permission check.** This workflow requires packs `lax-state`, `gh-read`, `<provider>-read`, `<provider>-write`. If any required pack is missing from `projects[active_project].allowed_packs`, halt with a `permission_pack_missing` message naming the missing pack(s) and pointing the user to `/lax:setup permissions`.

Generate a `session_id` of the form `update-<ISO timestamp now>` and append a `session_start` entry to the write log (see CLAUDE.md "Every write is logged"), recording the target ticket ID. Reuse this `session_id` for every log entry produced in this run.

Use the configured ticket provider's MCP server to fetch the Ticket. Read its title, description, current state, assignee, and any existing GH links. If the Ticket's Project doesn't match the active lax project's `ticket_project_id`, warn the user and ask whether to proceed or switch lax projects first.

## 2. Find related GH activity

Spawn the `github-activity-gatherer` subagent. Pass it the active lax project's single `github_repo` and tell it to search for PRs/commits/comments matching the Ticket title keywords, the Ticket ID itself, and any PR URLs already linked on the Ticket. Cast a moderately wide net — PR hygiene may be sloppy and the link to the Ticket may be implicit (branch name pattern, partial keyword match).

## 3. Propose update

Spawn the `ticket-reconciler` subagent in `update` mode with the Ticket, the activity, and the `ticket_provider` value as inputs. Expect a single proposal containing: an updated description, a state change suggestion (e.g. move to Done if PRs are merged), and PR links to add.

## 4. Approve and apply (with edits)

Show the diff (current vs proposed) to the user. List the editable fields: `proposed_state`, `add_links`, `comment_to_add`, and any updated `description`.

Present a numbered menu of choices and wait for the user to pick a number (or type the action name):

```
1. apply
2. edit
3. skip
```

If they pick **edit** (2), prompt for the field name (or a concrete change), re-show the diff with the edit applied, and re-present the same menu. Loop until they pick apply or skip.

On **apply** (1), append the Lax footer (see CLAUDE.md) to any comment and to any updated description just before sending — blank line, then `*Done with [Lax](https://github.com/styoe/lax)*`. Skip the append if the body already contains `[Lax](https://github.com/styoe/lax)`. Then write via the configured ticket provider's MCP server and append a log entry to `log.jsonl` (with the current `session_id`, action, ticket_id, comment_id if any, fields_written, result). Do not write before approval.
