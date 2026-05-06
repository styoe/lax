# lax: propose

Find work the user did in the **active lax project**'s repo that has no matching Ticket and propose creating Tickets for it.

Throughout this file:
- **Ticket** (capital T) = a single record in the configured ticket management software.
- **Project** (capital P) = the workspace/team/board grouping in that software where Tickets live.

The subcommand argument (if provided) is the **lookback window in days**. Default to 7 if unset.

## 1. Load config + window + permission check

Read `${CLAUDE_PLUGIN_DATA}/config.json` and resolve `projects[active_project]` to get `github_repo`, `ticket_project_id`, and `allowed_packs`. Also read the global `ticket_provider`. If config is missing or has no active project, stop and tell the user to run `/lax:setup` first.

**Permission check.** This workflow requires packs `lax-state`, `gh-read`, `<provider>-read`, `<provider>-write`. If any required pack is missing from `projects[active_project].allowed_packs`, halt with a `permission_pack_missing` message naming the missing pack(s) and pointing the user to `/lax:setup permissions`.

Convert the day count to a `since` date (now minus N days, ISO-8601).

Generate a `session_id` of the form `propose-<ISO timestamp now>` and append a `session_start` entry to the write log (see CLAUDE.md "Every write is logged"). Reuse this `session_id` for every log entry produced in this run.

## 2. Gather GitHub activity

Spawn the `github-activity-gatherer` subagent for the window, scoped to the active lax project's single `github_repo`.

## 3. Gather existing Tickets

Use the configured ticket provider's MCP server to fetch Tickets in `ticket_project_id` updated within the window, plus any Tickets in that Project that already reference PRs returned in step 2. The reconciler needs this to avoid proposing duplicates.

## 4. Identify gaps

Spawn `ticket-reconciler` in `propose` mode. It should return clusters of GH activity that look like one unit of work but have no matching Ticket. For each cluster, expect a draft Ticket title, description, and suggested PR links. New Tickets should target `ticket_project_id`.

## 5. Approve and create (per-item edit walk)

Walk through every proposed new Ticket one at a time, in reconciler order. For each:

1. Print the proposal in full — title, description, suggested links, evidence, confidence.
2. List the editable fields: `title`, `description`, `suggested_links`.
3. Present a numbered menu of choices and wait for the user to pick a number (or type the action name):
   ```
   1. create
   2. edit
   3. skip
   4. create all remaining as-is
   5. skip all remaining
   6. stop
   ```
4. If they pick **edit** (2), prompt for the field name (or a concrete change like "change title to ..."), re-show the proposal with the edit applied, and re-present the same numbered menu. Loop on the same item until they pick create, skip, or one of the bulk options.
5. On create:
   a. **Format the description** in the prospective `## Problem` / `## Proposed solution` form (see CLAUDE.md "Created Tickets follow a prospective format"). Do not include a retroactive "what was done" section.
   b. **Infer state and assignee** (see CLAUDE.md "Created Tickets get assignee + state inferred from evidence"): default `assignee: "me"`, default `state` from linked PR evidence (all merged → Done; any open → In Progress; otherwise provider default).
   c. **Append the Lax footer** to the description just before sending — blank line, then `*Done with [Lax](https://github.com/styoe/lax)*`. Skip the append if the description already contains `[Lax](https://github.com/styoe/lax)`.
   d. **Create the Ticket** via the configured ticket provider's MCP server (in the active lax project's Project) with `links` set from `suggested_links` so PRs become proper link-attachments.
   e. **Post the "what was done" comment** on the new Ticket as a second write — narrative covering the actual outcome, PR refs, and any in-flight notes — Lax footer appended.
   f. **Log both writes** to `log.jsonl` (current `session_id`; one `issue_create` entry, one `comment_create` entry).
   g. Confirm both writes back to the user. Move on.
6. On skip, do not create; move on.

Bulk choices (4–6 in the menu) take effect immediately and short-circuit the rest of the walk. **Stop** (6) halts and does **not** update `last_sync.propose` — leave it for the next run.

After the last item completes (without a stop), update `last_sync[active_project].propose` in the config file to the current ISO-8601 timestamp. Do not touch other projects' `last_sync` entries.
