# lax: sync

Run a full reconciliation of GitHub activity against the **active lax project**'s Tickets.

Throughout this file:
- **Ticket** (capital T) = a single record in the configured ticket management software.
- **Project** (capital P) = the workspace/team/board grouping in that software where Tickets live.

The subcommand argument (if provided) is an optional `since-date` (ISO-8601 or `YYYY-MM-DD`).

## 1. Load state + permission check

Read `${CLAUDE_PLUGIN_DATA}/config.json`. Resolve `projects[active_project]` to get `github_repo`, `ticket_project_id`, `ticket_project_name`, and `allowed_packs`. Also read the global `ticket_provider` and `github_handle`. If config is missing or has no active project, stop and tell the user to run `/lax:setup` first.

**Permission check.** This workflow requires packs `lax-state`, `gh-read`, `<provider>-read`, `<provider>-write` (e.g. `linear-read` and `linear-write` for Linear). If any required pack is missing from `projects[active_project].allowed_packs`, halt with a `permission_pack_missing` message naming the missing pack(s) and pointing the user to `/lax:setup permissions`.

Generate a `session_id` of the form `sync-<ISO timestamp now>` and append a `session_start` entry to the write log (see CLAUDE.md "Every write is logged"), recording the workflow args. Reuse this `session_id` for every log entry produced in this run.

## 2. Decide the cutoff date

If the subcommand argument was provided, use it as `since` and skip the rest of this step.

Otherwise, propose a cutoff. The goal is to avoid reconciling stale items — pick a date that captures the user's recent activity without dragging in old, settled work.

### 2a. Probe both sides (light — dates only)

- **Tickets:** fetch up to 20 of the most recently updated Tickets in `ticket_project_id` (sort by updated date desc). Capture only `updatedAt` per Ticket.
- **GitHub:** fetch up to 20 of the user's most recently updated PRs in `github_repo`:
  ```
  gh search prs --author=<github_handle> --repo=<github_repo> --sort=updated --limit=20 --json updatedAt
  ```

### 2b. Build candidate cutoffs

Compute candidates from the probe data and the config. Drop any candidate that resolves to a date in the future or duplicates another candidate (within a day).

1. **Last sync** — `last_sync[active_project].sync` if it exists.
2. **After the last quiet period** — the date just after the most recent gap > 7 days in the combined updated-date timeline (Tickets + PRs merged, sorted desc). If no such gap exists in the probed data, omit this option.
3. **7 days ago** — `now - 7d`.
4. **30 days ago** — `now - 30d`.
5. **Earliest item in the probed window** — the oldest `updatedAt` across the 40 probed items (ensures a fully bounded window).
6. **Custom** — user enters an ISO-8601 / `YYYY-MM-DD` date.

For each candidate, count how many of the 40 probed items fall after it (Tickets vs PRs broken out) so the user can see the rough scope.

### 2c. Present and wait

Show the candidates as a numbered list with rationale and counts, e.g.:

```
Cutoff options:
  1. 2026-04-12 — last sync — would include 8 Tickets, 6 PRs
  2. 2026-04-22 — after the last quiet period (gap 2026-04-15..2026-04-22) — 5 Tickets, 4 PRs
  3. 2026-04-28 — 7 days ago — 3 Tickets, 3 PRs
  4. 2026-04-05 — 30 days ago — 12 Tickets, 9 PRs
  5. 2026-03-10 — earliest probed item — 20 Tickets, 20 PRs
  6. custom — enter your own date

Pick a number, or enter an ISO-8601 / YYYY-MM-DD date.
```

Wait for the user's choice. If they enter a date directly (not a number), validate it parses; reject and re-prompt if not. Do not proceed past this step without an explicit selection.

The chosen value becomes `since` for the rest of the workflow.

## 3. Gather GitHub activity

Spawn the `github-activity-gatherer` subagent. Pass it: GitHub handle, the active lax project's single `github_repo`, `since` date. Expect a structured summary back (PRs authored/reviewed, commits, comments, plus theme clusters).

## 4. Pull Tickets

Use the configured ticket provider's MCP server (matching `ticket_provider`) to fetch (scoped to `ticket_project_id`):
- Open Tickets assigned to the user
- Tickets updated or closed within the time window (since `since`)
- Tickets that already reference any PR URLs returned in step 3

## 5. Reconcile

Spawn the `ticket-reconciler` subagent in `sync` mode. Pass it the GitHub activity, the Ticket set, and the `ticket_provider` value as inputs. Expect a structured proposal: Ticket updates, Ticket creates, PR<->Ticket link mappings.

## 6. Approve and apply (per-item edit walk)

First, print a one-paragraph summary of the proposal (counts of updates and creates, anything ambiguous flagged). Then walk through every proposed item one at a time, updates first (in reconciler order), then creates. For each item:

1. Print the item in full — Ticket ID for updates (or "new Ticket" for creates), proposed fields, evidence, confidence.
2. List the editable fields:
   - **Update**: `proposed_state`, `add_links`, `comment_to_add`.
   - **Create**: `title`, `description`, `suggested_links`.
3. Present a numbered menu of choices and wait for the user to pick a number (or type the action name):
   ```
   1. apply
   2. edit
   3. skip
   4. apply all remaining as-is
   5. skip all remaining
   6. stop
   ```
4. If they pick **edit** (2), prompt for the field name (or a concrete change like "remove link 2", "change title to ..."), re-show the item with the edit applied, and re-present the same numbered menu. Loop on the same item until they pick apply, skip, or one of the bulk options.
5. On apply, behavior splits by item type:

   **For an `update` item (existing Ticket):**
   - Append the Lax footer to the `comment_to_add` (and to any updated description) just before sending — blank line, then `*Done with [Lax](https://example.com)*`. Skip the append if the body already contains `[Lax](https://example.com)`.
   - Write via the configured ticket provider's MCP server (state change if any, plus comment).
   - Append a log entry to `log.jsonl` (current `session_id`, action, ticket_id, comment_id if any, fields_written, evidence, confidence, result).

   **For a `create` item (new Ticket):**
   - Format the description in the prospective `## Problem` / `## Proposed solution` form (see CLAUDE.md "Created Tickets follow a prospective format"). Do not include a retroactive "what was done" section.
   - Infer assignee + state per CLAUDE.md "Created Tickets get assignee + state inferred from evidence" (`assignee: "me"`; state from PR evidence: all merged → Done, any open → In Progress, else provider default).
   - Append the Lax footer to the description (idempotency rule above).
   - Create the Ticket with `links` set from `suggested_links` so PRs become proper link-attachments.
   - Post the "what was done" comment on the new Ticket as a second write — narrative covering the actual outcome, PR refs, and any in-flight notes — Lax footer appended.
   - Log both writes to `log.jsonl` (one `issue_create` entry, one `comment_create` entry).

   Confirm the write(s) back to the user. Move on to the next item.
6. On skip, do not write; move on.

Bulk choices (4–6 in the menu) take effect immediately and short-circuit the rest of the walk. **Stop** (6) halts the walk and does **not** update `last_sync` — leave it for the next run.

After the last item completes (without a stop), update `last_sync[active_project].sync` in the config file to the current ISO-8601 timestamp. Print a final summary listing what was applied vs skipped. Do not touch other projects' `last_sync` entries.
