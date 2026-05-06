# lax — Ticket ↔ GitHub sync for Claude Code

> **lax** *(adj.)*  
> c. 1400, "loose", from Latin *laxus* "wide, spacious, roomy," figuratively "loose, free, wide" , from PIE \*lag-so-, suffixed form of root \*sleg- "be slack, be languid."

`lax` is a Claude Code plugin that keeps your tickets in lockstep with your actual Git activity. It reads GitHub via the `gh` CLI, reads/writes the configured ticket management software via that provider's MCP server, and reconciles the two with subagents under a strict, per-item approval loop.

Provider-agnostic: Linear, Jira, GitHub Issues, Asana, Trello, or any custom provider you can wire up an MCP server for. The plugin ships provider-neutral (no MCP server pre-registered) — `/lax:setup` walks you through wiring up your chosen provider's MCP in a one-time step.

## Why lax

PR ↔ ticket linkage is sloppy in real teams. Branches don't always reference Ticket IDs. PR descriptions get skipped. Direct-to-`main` commits never see a PR at all. Tickets stay open long after their work shipped, or get marked Done before any code actually merged. The result: your ticket tracker drifts out of sync with what really happened in Git, and the only way to reconstruct reality is to ask the person who did the work — or read every commit by hand.

`lax` is for cleaning that up after the fact, on demand:

- **You ship first, track later.** `lax` doesn't try to enforce discipline at PR time. It accepts that real-world linkage is sloppy and reconciles retroactively.
- **Wide-net reconciler.** Matches by direct Ticket ID first, then existing PR links, then branch patterns, then keyword/title overlap. Uncertain matches are flagged `low confidence` for your judgment.
- **Human in the loop on every write.** Every Ticket update or new Ticket goes through a per-item approval menu — see the evidence, edit if needed, apply or skip. Bulk-approve only when you've earned the trust.
- **Audit trail.** Every write appends a JSON line to a local log, so "what did `lax` change last Tuesday?" is a `grep`, not a guess.
- **No vendor lock-in.** Linear, Jira, GitHub Issues, Asana, Trello, or anything you can wire to an MCP server.

The name comes from the etymology above: this tool is for the lax bookkeeper who ships code first and updates the tracker second (or never). It doesn't moralize about the habit — it just cleans up after it.

---

## Quick start

```bash
# 1. Make sure the gh CLI is authenticated
gh auth status

# 2. (In Claude Code) install the plugin if you haven't already
#    Then bootstrap it:
/lax:setup

# 3. Smoke test — pulls 3 latest Tickets + 3 latest PRs side by side
/lax:test

# 4. Run a full reconciliation since some date
/lax:sync 2026-04-01
```

Walk through `/lax:setup` once. After that the day-to-day commands are `/lax:test`, `/lax:sync`, `/lax:update <ticket-id>`, `/lax:propose [days]`, and `/lax:standup [day|week]`. Switch contexts with `/lax:project [key]`.

---

## Terminology

`lax` uses these terms consistently — they show up in every workflow file and in the prompts you'll see:

- **Ticket** (capital T) — a single record in the configured ticket management software (a Linear issue, Jira ticket, GitHub Issue, Asana task, Trello card, or equivalent in a custom provider).
- **Project** (capital P) — the workspace/team/board grouping in that software where Tickets live (a Linear team, a Jira project, a GitHub repo's Issues, an Asana project, a Trello board, or equivalent in a custom provider).
- **lax project** (lowercase) — `lax`'s internal pairing of one GitHub repo with one Project, stored under a short key in the config. Switch between them with `/lax:project`.

A typical setup has one or two lax projects: e.g. `widget` → (`acme/widget`, Linear team "Widget"), `gizmo` → (`acme/gizmo`, Linear team "Gizmo").

---

## Commands

All commands are namespaced under `/lax:` (a Claude Code constraint for plugin commands). Each command is a thin wrapper that reads its workflow file from `workflows/` and executes it verbatim.

### `/lax:setup [permissions]`

Bootstrap the plugin or add another lax project to it. Walks an 8-step checklist:

1. `gh auth status` check
2. Pick the ticket management software (Linear / Jira / GitHub Issues / Asana / Trello / custom)
3. Verify provider connectivity (Linear MCP, or instructions to wire your own)
4. Resolve identity (GH handle + provider user id)
5. Pick the lax project to configure (key + repo + Project)
6. Persist `config.json` (additive; never clobbers other projects)
7. Confirm
8. Configure permissions per lax project (toggleable pack checkboxes)

Re-run to add or edit additional lax projects. Pass the argument `permissions` to jump straight to step 8: `/lax:setup permissions [project-key]`.

### `/lax:project [key]`

Switch which lax project is currently active. Subsequent `test`/`sync`/`update`/`propose` runs use that project's GitHub repo and Project. With no argument, lists all lax projects and prompts. No writes to the ticket provider or GitHub.

### `/lax:test`

Read-only smoke test for the active lax project: 3 latest Tickets in the active Project + 3 latest GitHub PRs in the active repo, side by side. Confirms both integrations are alive. No writes.

### `/lax:sync [since-date]`

Full reconciliation of GitHub activity against the active lax project's Tickets. Takes an optional `since-date` (ISO-8601 / `YYYY-MM-DD`) — if omitted, `lax` proposes a cutoff based on probing both timelines (last sync, gap detection, 7d/30d, earliest-probed-item, custom).

Steps:
1. Load config + permission check
2. Decide cutoff date (interactive when no arg)
3. Spawn `github-activity-gatherer` subagent — gathers PRs/commits/comments via `gh`
4. Pull Tickets in window via the configured provider's MCP
5. Spawn `ticket-reconciler` subagent — pure reasoning, returns a structured proposal (updates + creates)
6. Per-item edit walk — apply / edit / skip each item; bulk shortcuts available

After the last item, updates `last_sync[<active>].sync` to the current ISO timestamp.

### `/lax:update <ticket-id>`

Fill in a single Ticket using GitHub evidence in the active repo. The reconciler runs in `update` mode (≤ 1 update item). Useful when you know which Ticket needs attention and want a focused, fast pass.

### `/lax:propose [days]`

Find work in the active repo with no matching Ticket in the active Project, and propose creating Tickets for it. Default lookback is 7 days. The reconciler runs in `propose` mode (creates only). Per-item edit walk, then creates the Tickets and updates `last_sync[<active>].propose`.

Each created Ticket gets a prospective-format description (`## Problem` / `## Proposed solution`) plus an outcome comment with PR refs and what shipped — see "How writes work" below.

### `/lax:standup [day|week]`

Read-only standup write-up of recent activity for the active lax project. Pulls the user's GitHub activity (PRs authored / reviewed, commits, comments) via the gatherer and the active Project's recent Ticket activity via the configured provider's MCP, joins the two sides, and prints a sectioned narrative — Shipped / In progress / Reviews / Blockers / Up next / Untracked — ending in a paste-ready summary paragraph.

Default window is `week`; pass `day` for a 24-hour window. No writes, no approval loop, does not update `last_sync`.

If a Ticket changed state inside the configured window but the join finds no matching PR/commit/branch in that window, `lax` automatically widens the GitHub lookback in one-week steps — `week` → ~14d → ~21d → ~28d → ~35d → capped at 40d — re-running the join only for the under-explained Tickets until each has attributable evidence (or the cap is hit). The window for what counts as "in-window activity" in the write-up doesn't move; only the GitHub lookback used to attribute Tickets does. Pre-window evidence is cited explicitly (e.g. "Ticket closed today; PR merged 13 days ago"), never silently filed under "this week".

---

## How writes work — the approval loop

Every write goes through a per-item edit walk. There is **no** bulk-on-bulk-off blanket approval; each Ticket update or create is shown to you with its evidence and a numbered menu:

```
1. apply        (or "create" for new Tickets)
2. edit
3. skip
4. apply all remaining as-is
5. skip all remaining
6. stop
```

Picking **edit** prompts for the field name (or a concrete change like "remove link 2", "change title to ..."), re-shows the item with the edit applied, and re-presents the menu. Loop until apply or skip. **Stop** halts the walk and does not update `last_sync` (so you can resume next run).

### Lax footer

Every comment and Ticket description Lax writes gets this footer appended at write time, after a blank line:

```
*Done with [Lax](https://github.com/styoe/lax)*
```

Appended after your approval — it isn't shown in the proposal diff, so your edits to the body don't fight with the rule. Idempotent: skipped if the body already contains `[Lax](https://github.com/styoe/lax)`.

The Lax convention is pure markdown (no HTML), because Linear/Tiptap and similar renderers strip arbitrary HTML. Italic + a markdown link is the most cross-provider-portable approximation of "small attribution text".

### Prospective Ticket format (creates)

When `lax` creates a new Ticket (via `/lax:propose` or `/lax:sync` create items), the description is written as if authored *before* the work started — never as a retroactive narrative. Required structure:

```markdown
## Problem

<one paragraph: why this work was needed; the constraint, pain, or goal>

## Proposed solution

- [ ] <step 1>
- [ ] <step 2>
- [ ] ...
```

The actual outcome (PRs merged, what shipped, links) is posted **as a comment on the same Ticket immediately after creation**, in a single second write. Both description and comment get the Lax footer.

### Inferred state and assignee on create

For new Tickets:
- `assignee` defaults to the active user (`"me"`) unless you edit otherwise.
- `state` is inferred from PR evidence: all linked PRs merged → `Done`, any open/draft → `In Progress`, no PR signal → provider default (Backlog/Todo).

You can override either via the edit menu.

---

## Persistent log

Every successful programmatic write is appended as a single JSON line to `~/.claude/plugins/data/lax/log.jsonl` (or `${CLAUDE_PLUGIN_DATA}/log.jsonl` if the env var is set). One `session_id` is generated at workflow start (form: `<workflow>-<ISO timestamp>`) and reused for every entry produced by that run, including a `session_start` entry written before the first action.

Entry shape:

```json
{
  "ts": "<ISO timestamp of write>",
  "session_id": "<workflow>-<session_started_at_iso>",
  "workflow": "sync|propose|update",
  "project": "<active_project_key>",
  "ticket_provider": "<provider>",
  "action": "session_start|comment_create|comment_update|issue_create|issue_update",
  "ticket_id": "PROJ-42",
  "comment_id": "<id-if-applicable>",
  "fields_written": { },
  "evidence": [],
  "confidence": "high|medium|low|null",
  "result": "success|error",
  "error": null
}
```

Append-only; never rewritten. Filter by `project` / `session_id` when querying. Useful for "what did `lax` do last Tuesday?" or "what got written to PROJ-42?".

---

## Configuration

State lives in `~/.claude/plugins/data/lax/config.json` (or `${CLAUDE_PLUGIN_DATA}/config.json`):

```json
{
  "github_handle": "octocat",
  "ticket_provider": "linear",
  "ticket_user_id": "00000000-0000-0000-0000-000000000000",
  "active_project": "widget",
  "projects": {
    "widget": {
      "github_repo": "acme/widget",
      "ticket_project_id": "00000000-0000-0000-0000-000000000000",
      "ticket_project_name": "Widget",
      "allowed_packs": [
        "gh-read",
        "lax-state",
        "linear-read",
        "linear-write",
        "linear-mcp-probe"
      ]
    }
  },
  "last_sync": {
    "widget": { "sync": "2026-05-06T10:09:30Z", "propose": "..." }
  }
}
```

`ticket_provider`, `ticket_user_id`, and `github_handle` are global. Each project records its own `github_repo`, `ticket_project_id`, `ticket_project_name`, and `allowed_packs`. `last_sync` is keyed per project per workflow. `lax` always merges and never clobbers other projects' entries.

---

## Permissions (per-lax-project)

Each lax project records which Claude Code permission packs it has opted into in `projects[<key>].allowed_packs`. Workflows halt at startup if the active project's `allowed_packs` is missing a required pack.

### Pack catalog

| Pack | Permissions | Required by |
|---|---|---|
| **gh-read** | `Bash(gh auth *)`, `Bash(gh api *)`, `Bash(gh search *)` | every workflow that hits GitHub |
| **lax-state** | `Read/Write/Edit(<plugin-data>/**)`, `Bash(mkdir -p ~/.claude/plugins/data/lax)`, `Bash(printf *)`, `Bash(wc *)`, `Bash(ls *plugins/data/lax/*)` | every workflow (config + log) |
| **linear-read** | `mcp__linear__list_issues`, `mcp__linear__get_issue`, `mcp__linear__list_users`, `mcp__linear__list_teams`, `mcp__linear__list_comments`, `mcp__linear__list_projects`, `mcp__linear__get_user` | test, sync, update, propose, setup |
| **linear-write** | `mcp__linear__save_comment`, `mcp__linear__save_issue`, `mcp__linear__create_attachment` | sync, update, propose |
| **linear-mcp-probe** | `Bash(curl ... mcp.linear.app/sse ...)`, `Bash(curl ... mcp.linear.app/mcp ...)` | setup health checks |

### How toggling works

Run `/lax:setup permissions` (or `/lax:setup permissions <project-key>`). You get a checkbox prompt scoped to that project:

```
Permission packs for project widget:

  1. [x] gh-read           — GitHub read via gh CLI
  2. [x] lax-state         — Read/write Lax config + log files
  3. [x] linear-read       — Linear MCP read tools
  4. [ ] linear-write      — Linear MCP write tools
  5. [x] linear-mcp-probe  — curl health checks

Toggle: 2,4 / all / none / done
```

When you type `done`, `lax` writes the new pack set to `projects[<target>].allowed_packs`, then **recomputes the union across all projects** and surgically syncs `<project-root>/.claude/settings.local.json`:
- **Adds** rules in the union that are missing from `permissions.allow`.
- **Removes** rules in `permissions.allow` that exactly match a Lax pack rule and that no project still enables. User-added entries that don't match a Lax pack rule are never touched.

### Why per-project + a union

Claude Code's permission gate is per-cwd, not per-lax-project — it cannot natively scope "only when active_project=widget". So `lax` enforces per-project gating itself by halting workflows whose required packs aren't in the active project's `allowed_packs`. The union in `settings.local.json` is just there to suppress UI prompts for any tool *some* project legitimately uses.

For non-Linear providers, you wire the matching MCP server yourself and add the corresponding `<provider>-read` / `<provider>-write` permissions manually; `lax` v1 maintains pack catalogs only for Linear.

---

## Project layout

```
.claude-plugin/
  plugin.json                 # plugin manifest (name, version, description, author, license, ...)
  marketplace.json            # makes this repo installable as a marketplace ("/plugin marketplace add styoe/lax")
commands/                     # one slash command per subcommand — thin wrappers around workflows/
  setup.md  project.md  test.md  sync.md  update.md  propose.md  standup.md
workflows/                    # full procedure for each subcommand, executed verbatim
  setup.md  project.md  test.md  sync.md  update.md  propose.md  standup.md
agents/                       # two subagents the workflows spawn
  github-activity-gatherer.md   # gh CLI -> structured JSON + theme clusters
  ticket-reconciler.md          # pure reasoning, no tools, returns proposal
.claude/settings.local.json   # gitignored — local perms + your enabledMcpjsonServers
CLAUDE.md                     # working agreement + conventions for Claude when editing this plugin
README.md                     # this file
LICENSE                       # MIT
.gitignore                    # ignores .claude/settings.local.json + OS/editor noise
```

The plugin deliberately does **not** ship a `.mcp.json` — that would force a Linear MCP registration on every user, including those on Jira/etc. You wire your provider's MCP server in your own `.mcp.json` (project-root or `~/.claude/.mcp.json`) during `/lax:setup` step 3, which provides a copy-paste snippet for Linear and pointers for other providers.

Plugin data (config + log) lives outside the repo at `~/.claude/plugins/data/lax/`.

---

## Subagents

`lax` deliberately splits the workflow across two single-purpose subagents and an orchestrator (the workflow file itself). The contracts are strict.

### `github-activity-gatherer`

Tools: `Bash`, `Read`. Pulls PRs authored, PRs reviewed, commits, and substantive comments via the `gh` CLI for a given user, repo, and time window. Returns structured JSON plus a `Themes` prose section clustering related work. Read-only — never touches the ticket provider.

### `ticket-reconciler`

Tools: `Read`. Pure reasoning, no tool calls. Inputs: GitHub activity summary + Ticket set + `ticket_provider` + `mode` (`sync` / `update` / `propose`). Output: `{summary, updates[], creates[], unmatched_activity[], unmatched_tickets[]}`.

Matching priority: direct Ticket-ID reference → existing PR link → branch-name pattern → keyword/title overlap. Many-to-many PR↔Ticket links are explicitly supported. Confidence is graded honestly; sloppy linkage produces low-confidence matches the user can decide on.

For `creates`, the reconciler returns descriptions in the prospective `## Problem` / `## Proposed solution` form plus a separate `outcome_comment` field — the orchestrator posts the description and comment as two separate writes.

---

## Extending lax

To add a new command (e.g. `/lax:pr-desc`):

1. Create `commands/<name>.md` — frontmatter with `description` (and `argument-hint` if applicable). Body is a thin wrapper that reads `${CLAUDE_PLUGIN_ROOT}/workflows/<name>.md` and executes it verbatim.
2. Create `workflows/<name>.md` — the full procedure. Reuse the patterns: load config + permission check, do the work, walk the user through approvals, log writes, update `last_sync` per workflow.
3. Update CLAUDE.md and README.md to mention the new command and any new conventions.
4. If you need new permission rules, extend the pack catalog in CLAUDE.md and `workflows/setup.md` step 8.
5. Subagent boundaries are strict — gatherer never reasons about Tickets; reconciler never calls tools. The orchestrator (workflow file) is the only place that bridges the two systems.

---

## Limitations / non-goals (v1)

- Pack catalog is Linear-only. Other providers work but require manual MCP wiring + manual permission allow-listing.
- `lax` doesn't write to PRs (no PR description updates, no PR comments). That's planned (`/lax:pr-desc`) but not built.
- `lax` doesn't manage Ticket dependencies / blockers, cycles, projects-as-Linear-projects, milestones. Out of scope for v1.
- Setup doesn't auto-add user-side identity (e.g., your GitHub email ↔ Linear user mapping when they differ); it captures one user identity globally.
- The Linear MCP wiring snippet in `/lax:setup` step 3 currently shows the `sse` transport, which Linear is deprecating in favor of `mcp` (at `https://mcp.linear.app/mcp`). The snippet should be updated to recommend the new transport once it's the default.
