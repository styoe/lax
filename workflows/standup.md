# lax: standup

Produce a read-only standup write-up of the user's recent work in the **active lax project**. Pull activity from both the configured ticket management software and GitHub for the requested window, then format a short narrative the user can paste into a standup channel or chat.

No writes, no approval loop, no `last_sync` updates — this workflow is purely informational.

Throughout this file:
- **Ticket** (capital T) = a single record in the configured ticket management software.
- **Project** (capital P) = the workspace/team/board grouping in that software where Tickets live.

The subcommand argument is the **window**: `day` or `week`. Default to `week` if unset.

## 1. Load config + window + permission check

Read `${CLAUDE_PLUGIN_DATA}/config.json`. Resolve `projects[active_project]` to get `github_repo`, `ticket_project_id`, `ticket_project_name`, and `allowed_packs`. Also read the global `ticket_provider`, `ticket_user_id`, and `github_handle`. If config is missing or has no active project, stop and tell the user to run `/lax:setup` first.

**Permission check.** This workflow requires packs `lax-state`, `gh-read`, `<provider>-read` (e.g. `linear-read` for Linear). It does **not** require `<provider>-write` — standup never writes. If any required pack is missing from `projects[active_project].allowed_packs`, halt with a `permission_pack_missing` message naming the missing pack(s) and pointing the user to `/lax:setup permissions`.

**Resolve the window.** Normalize `$ARGUMENTS`:

- empty / unset → `week`
- `day` / `d` / `1d` / `today` → `day`
- `week` / `w` / `7d` → `week`
- anything else → halt with: `unknown window "<value>"; valid values are "day" or "week"`.

Compute `since` from the chosen window:
- `day` → now − 24h (ISO-8601)
- `week` → now − 7d (ISO-8601)

This workflow does **not** generate a `session_id` or write to `log.jsonl` — it has no programmatic writes to log.

## 2. Gather GitHub activity

Spawn the `github-activity-gatherer` subagent. Pass it: `handle = github_handle`, `repos = [github_repo]`, `since`. Expect the structured summary it returns (PRs authored, PRs reviewed, commits, comments, plus a `Themes` prose section).

If the gatherer reports `gh` is unauthenticated or missing, halt with that error verbatim and route the user to `gh auth login`.

## 3. Gather Ticket activity

Use the configured ticket provider's MCP server (matching `ticket_provider`) to fetch — all scoped to `ticket_project_id`:

- Tickets **updated** within the window (since `since`).
- Tickets currently **assigned to `ticket_user_id`** that are in an open / in-progress state, even if they haven't moved in the window. (These surface as "still on my plate" in the write-up.)
- For each Ticket above, the most recent comment by `ticket_user_id` within the window if available — useful for capturing the user's own status updates.

For each Ticket capture: ID, title, state, assignee, updated date, URL, plus any user comment excerpts gathered.

## 4. Join the two sides (lightweight, in-workflow)

This workflow does **not** spawn the `ticket-reconciler` subagent — its contract is for `sync`/`update`/`propose` modes, and a standup write-up doesn't need its full reasoning. Do the join here in the orchestrator with a simple match priority:

1. Direct Ticket-ID reference in PR title, body, or branch name (highest confidence).
2. Existing PR URL recorded as a Ticket link.
3. Branch-name pattern (e.g. `<ticket-id>-...`).

If a PR matches a Ticket, group them together. If a PR matches no Ticket, list it under "Untracked work" in step 5. If a Ticket has no PR activity in the window, it goes under "In progress" or "Up next" depending on its state.

Do not flag low-confidence matches with prompts — for a standup write-up, just take the best match silently and move on. Confidence-grading and approval loops belong in `/lax:sync`, not here.

## 4.5. Backfill GitHub context for under-explained Tickets

The configured window (`day` or `week`) defines what counts as **in-window activity** in the final write-up — that boundary does not move. But Tickets routinely close days after the work actually shipped: a Ticket marked Done six days ago likely traces back to a PR merged eight or ten days ago, which falls outside a 7-day GitHub gather and produces a write-up entry with no attributable evidence ("closed administratively"). That answer is almost never the truth — the work happened, just earlier than the window — and a useful standup needs the real attribution.

After the initial join, identify **under-explained Tickets** — Tickets whose state changed within the primary window (most importantly: transitioned to a closed/Done/cancelled state, but also newly-started Tickets and Tickets that moved between in-progress states) for which the join found no matching PR, commit, or branch in the gathered GitHub activity. If there are none, skip this step.

If there are any, **incrementally widen the GitHub lookback in one-week steps, up to 40 days total**:

1. Re-spawn `github-activity-gatherer` with `since` set one additional 7-day step further back (so first retry covers ~14d, then ~21d, ~28d, ~35d, capping at 40d). Pass the same `handle` and `repos`. Pass the under-explained Ticket list so the gatherer can prioritize keyword/topic/branch matches against those Ticket titles, descriptions, and `gitBranchName` values.
2. Re-run the join (step 4) for the under-explained Tickets only — the rest of the write-up keeps using the original in-window data.
3. As soon as a Ticket finds attributable evidence, lock it in and stop expanding for that Ticket. Stop the whole backfill once every under-explained Ticket has evidence, or once the cap is hit.
4. Any Ticket still under-explained at 40 days is genuinely unattributable from GitHub — surface it in the write-up as `closed administratively — no GH evidence in last 40 days` rather than fabricating a match.

When backfilled evidence is from outside the primary window, **say so explicitly in the write-up**: e.g. "AMP-44 — Ticket closed today; actual work shipped 2026-04-23 → 2026-04-28 via PRs #39, #50, #51". Do not silently file pre-window PRs under "Shipped this week" — the reader needs to know the work predates the window. The convention is to keep the Ticket in the relevant section but cite the out-of-window PR dates as evidence.

This step adds latency (extra subagent spawns) and is read-only. It runs only when needed — if the initial join attributes everything, no backfill happens.

## 5. Produce the standup write-up

Print the result directly to the user — this is the deliverable. Use the structure below, skipping any section that has no items so the output stays tight.

```
Standup — <active_project> (<github_repo> ↔ <ticket_project_name> on <ticket_provider>)
Window: last <day|week> (since <YYYY-MM-DD>)

### Shipped
- <Ticket ID or "—"> <Ticket title or PR title> — <evidence: "PR #N merged <date>", optional Ticket state>
- ...

### In progress
- <Ticket ID> <Ticket title> — <state> — <evidence: "PR #N open", "draft", or "no PR yet">
- ...

### Reviews & collaboration
- <PR URL> — reviewed (<state>) <date>
- <comment URL> — "<short excerpt>"
- ...

### Blockers / open questions
- <anything explicitly blocked, stale review, or note from the gatherer's Themes that doesn't fit above>
- ...

### Up next
- <Open Ticket assigned to me with no PR activity in the window> — <state>
- ...

### Untracked work
- <PR with no Ticket match> — <PR URL> — <state>
- ...

---

**Summary**

<one-paragraph spoken-style narrative the user can paste verbatim into a standup chat: "This <day|week> I shipped …, I'm currently …, blocked on …, next up …">
```

Rules for the sections:

- **Shipped**: PRs merged in the window. Pair with their matched Ticket if any (show Ticket ID + title; the PR is evidence). For unmatched merged PRs, leave the Ticket column as `—`.
- **In progress**: open PRs the user authored (any state: open/draft) and Tickets in an in-progress state with PR activity in the window. De-duplicate against Shipped.
- **Reviews & collaboration**: PRs the user reviewed in the window plus substantive comments the gatherer surfaced. Skip if both are empty.
- **Blockers / open questions**: anything the gatherer flagged in `Themes` that signals friction (e.g. "flaky test investigation, no fix yet"), plus PRs sitting in review with `updatedAt` more than 3 days before `now`. If nothing qualifies, omit the section.
- **Up next**: open Tickets assigned to the user that had no PR activity in the window. Cap at 5 — sort by `updated` desc, drop the rest with a `(+N more)` tail line.
- **Untracked work**: PRs with no Ticket match. This is intentionally surfaced so the user can spot reconciliation gaps; suggest `/lax:propose` or `/lax:sync` in a one-line tail if this section is non-empty.

The **Summary** paragraph at the end is the headline deliverable — make it sound like a human standup, not a bulleted list. Reference 1–3 of the most notable items (a shipped PR, a meaningful in-progress effort, the next concrete thing). Keep it under 120 words.

## 6. Health check

If either fetch fails, name the failure precisely (e.g. "Ticket provider MCP returned auth error — re-auth and retry"; "`gh` is not authenticated — run `gh auth login`"). Do not retry silently. Do not print a partial standup with one side missing — halt and explain.
