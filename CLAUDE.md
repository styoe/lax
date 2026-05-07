# lax — Ticket ↔ GitHub sync (Claude Code plugin)

A Claude Code plugin that keeps Tickets in lockstep with actual Git activity. It reads GitHub via the `gh` CLI, reads/writes the configured ticket management software via that provider's MCP server, and reconciles the two with subagents under a strict approval loop.

The plugin ships **provider-neutral** — no MCP server is pre-registered. `/lax:setup` walks the user through wiring up the MCP server matching their chosen provider (Linear, Jira, GitHub Issues, Asana, Trello, or a custom one).

## Terminology

Throughout this plugin:
- **Ticket** (capital T) — a single record in the configured ticket management software (a Linear issue, Jira ticket, GitHub Issue, Asana task, Trello card, or equivalent in a custom provider).
- **Project** (capital P) — the workspace/team/board grouping in that software where Tickets live (a Linear team, a Jira project, a GitHub repo's Issues, an Asana project, a Trello board, or equivalent in a custom provider).
- **lax project** (lowercase) — the plugin's internal pairing of one GitHub repo with one Project, stored under a short key in the config. Switch between them with `/lax:project`.

The user picks the ticket management software during `/lax:setup`. Linear, Jira, GitHub Issues, Asana, Trello, and custom providers are all supported — the user wires up the matching MCP server themselves in setup step 3.

## Layout

```
.claude-plugin/
  plugin.json                 # plugin manifest (name, version, description, author, license, ...)
  marketplace.json            # makes this repo installable as a marketplace
commands/                     # one slash command per subcommand — thin wrappers around workflows/
  setup.md  project.md  test.md  sync.md  update.md  propose.md  standup.md
workflows/                    # full procedure for each subcommand, executed verbatim
  setup.md  project.md  test.md  sync.md  update.md  propose.md  standup.md
agents/                       # two subagents the workflows spawn
  github-activity-gatherer.md   # gh CLI -> structured JSON + theme clusters
  ticket-reconciler.md          # pure reasoning, no tools, returns proposal
LICENSE                       # MIT
.claude/settings.local.json   # gitignored — local perms + the user's enabledMcpjsonServers
```

The plugin deliberately does **not** ship a `.mcp.json` of its own — that would force a Linear MCP registration on every user, including those on Jira/etc. Each user wires up their provider's MCP server in their own `.mcp.json` (project-root or `~/.claude/`) during `/lax:setup` step 3.

## How invocation flows

1. User runs `/lax:<sub> [args]` (e.g. `/lax:test`, `/lax:sync 2026-04-01`). Plugin commands are always namespaced as `/<plugin>:<command>` — this is a hard Claude Code constraint.
2. The matching `commands/<sub>.md` file is a thin wrapper: it instructs Claude to read `${CLAUDE_PLUGIN_ROOT}/workflows/<sub>.md` and execute it **verbatim** (no paraphrasing), passing `$ARGUMENTS` through as "the subcommand argument" wherever the workflow refers to it.
3. The workflow file owns the full procedure including the approval loop.

## Subcommands

All read/write subcommands operate on the **active lax project** — a `(github_repo, Project)` pair stored under a short key in `projects{}`. Switch lax projects with `/lax:project`.

- **setup** — verify `gh auth status`, pick the ticket management software (Linear, Jira, GitHub Issues, Asana, Trello, or custom) and verify its MCP, resolve GH handle / Ticket-provider user, then configure **one lax project** (key + repo + Project). Re-run to add or edit additional lax projects, or to change provider. Merges into `projects{}` — never clobbers other projects or `last_sync` keys.
- **project [key]** — change `active_project`. With no arg, lists lax projects and prompts. No writes to the ticket provider or GitHub.
- **test** — read-only smoke test for the active lax project: 3 latest Tickets (in the active Project) + 3 latest GitHub PRs (in the active repo) side by side. No writes.
- **sync** — full reconciliation for the active lax project since `last_sync[<active>].sync` (or arg date). Spawns gatherer → pulls Tickets → spawns reconciler in `sync` mode → approval loop → applies via the configured provider's MCP → updates `last_sync[<active>].sync`.
- **update <ticket-id>** — fill one Ticket from GH evidence in the active repo. Reconciler runs in `update` mode (≤ 1 update item). Approval loop, then write.
- **propose [days]** — find work in the active repo with no matching Ticket in the active Project. Reconciler runs in `propose` mode (creates only). Per-item approval, then create Tickets and update `last_sync[<active>].propose`.
- **standup [day|week]** — read-only standup write-up of recent Ticket + GitHub activity for the active lax project. Default window `week`. Spawns the gatherer, pulls Tickets via the configured provider's MCP, joins the two sides in-workflow (no reconciler), and prints a sectioned narrative ending in a paste-ready summary paragraph. No writes; does not update `last_sync`.

## Subagents

- **github-activity-gatherer** (tools: `Bash`, `Read`) — gathers PRs authored/reviewed, commits, substantive comments via `gh`. Returns a structured JSON shape plus a `Themes` prose section. Read-only; never touches the ticket provider.
- **ticket-reconciler** (tools: `Read`) — pure reasoning. Inputs: GH activity summary + Ticket set + `ticket_provider` + `mode` (`sync`|`update`|`propose`). Output: `{summary, updates[], creates[], unmatched_activity[], unmatched_tickets[]}`. Matching priority: direct Ticket-ID reference → existing PR link → branch-name pattern → keyword/title overlap. Many-to-many PR↔Ticket links are supported by design.

## Conventions to respect

- **Approval loop is mandatory.** Every write workflow follows: dry-run → show diff → wait for explicit user approval → apply. Partial approval ("apply 1, 3, 4 only") is supported. Never write to the ticket provider before confirmation.
- **Ship subset first, then expand.** Current scope is the 6 workflows above; lock down subagent contracts on this subset before adding more.
- **Sloppy PR↔Ticket linkage is the core problem domain** — branches/titles/bodies often only implicitly reference Tickets. Cast wide nets when gathering, but mark uncertain matches `confidence: low` and let the user decide.
- **Provider-agnostic by design.** Workflows refer to "the configured ticket provider's MCP server"; only `setup` mentions specific providers by name (and only inside the per-provider wiring snippets in step 3). The `ticket-reconciler` subagent reasons about Tickets generically, regardless of provider. Nothing in the plugin assumes Linear at runtime — it's only the most-tested provider and the one with worked-out MCP wiring guidance in setup.
- **Lax-authored content carries a footer.** Any text Lax writes to the ticket provider — Ticket comments and Ticket descriptions (new or updated) — has this exact footer appended at **write time** (after user approval, not shown in the proposal diff), separated from the body by a blank line:
  ```
  *Done with [Lax](https://github.com/styoe/lax)*
  ```
  Pure markdown only — no HTML. Linear and similar Tiptap-based renderers strip arbitrary HTML, so `<sub>`/`<small>` would render as literal text. Italic + a markdown link is the most cross-provider-portable approximation of "small attribution text". Idempotent — fuzzy detection: skip the append if the body already contains the substring `github.com/styoe/lax` (case-insensitive), regardless of surrounding markdown. The URL is the invariant token, so this matches the canonical `*Done with [Lax](https://github.com/styoe/lax)*` and any renderer-mangled variant such as `*Done with *[*Lax*](<https://github.com/styoe/lax>)` (which Linear's description editor produces) or a bare-URL `Done with https://github.com/styoe/lax`. The footer applies only to comments and descriptions — not to titles, labels, state changes, or links.
- **Created Tickets follow a prospective format.** When Lax creates a new Ticket (`/lax:propose` or `/lax:sync` create items), the description is written as if authored *before* the work started — never as a retroactive narrative. Required structure:
  ```
  ## Problem

  <one paragraph: why this work was needed; the constraint, pain, or goal>

  ## Proposed solution

  - [ ] <step 1>
  - [ ] <step 2>
  - [ ] ...
  ```
  No "Outcome" / "What was done" section in the description. Instead, the actual outcome (PRs merged, what shipped, links) is posted **as a comment on the same Ticket immediately after creation**, in a single write. Both description and comment get the Lax footer.
- **Created Tickets get assignee + state inferred from evidence.** On create, default `assignee` to the active user (`assignee: "me"`) unless the user edits otherwise. Infer `state` from the linked PR evidence:
  - All linked PRs merged → `Done`
  - Any linked PR open / draft / in-flight → `In Progress`
  - No linked PRs and no clear in-flight signal → leave provider default (Backlog/Todo)
  The user can override via the per-item edit menu.
- **Permission packs are per-lax-project.** Each lax project records which packs it has opted into in `projects[<key>].allowed_packs`. Workflows halt at startup if the active project's `allowed_packs` is missing a required pack — the user can run `/lax:setup permissions` to opt in. Claude Code's permission gate cannot enforce per-lax-project scoping (it's per-cwd, not per-lax-project), so Lax enforces it itself.

  **Pack catalog** (each pack expands to a list of Claude Code permission rules; path placeholders like `<plugin-data>` are resolved at setup time using `$HOME`):
  - **gh-read** — `Bash(gh auth *)`, `Bash(gh api *)`, `Bash(gh search *)`. Required by every workflow that pulls GitHub state.
  - **lax-state** — `Read(<plugin-data>/**)`, `Write(<plugin-data>/**)`, `Edit(<plugin-data>/**)`, `Bash(mkdir -p ~/.claude/plugins/data/lax)`, `Bash(printf *)`, `Bash(wc *)`, `Bash(ls *plugins/data/lax/*)`. Required by every workflow (config + log).
  - **linear-read** — `mcp__linear__list_issues`, `mcp__linear__get_issue`, `mcp__linear__list_users`, `mcp__linear__list_teams`, `mcp__linear__list_comments`, `mcp__linear__list_projects`, `mcp__linear__get_user`. Read-only ticket queries (only when `ticket_provider: linear`).
  - **linear-write** — `mcp__linear__save_comment`, `mcp__linear__save_issue`, `mcp__linear__create_attachment`. Required by sync, update, propose.
  - **linear-mcp-probe** — `Bash(curl ... mcp.linear.app/sse ...)`, `Bash(curl ... mcp.linear.app/mcp ...)`. Used only by setup health checks.

  **Per-workflow required packs:**
  - `setup` — `lax-state`; plus `linear-mcp-probe` + `linear-read` when configuring a Linear-backed project.
  - `test` / `standup` — `lax-state`, `gh-read`, `linear-read` (or the configured provider's read pack).
  - `sync` / `update` / `propose` — `lax-state`, `gh-read`, `linear-read`, `linear-write`.

  **`settings.local.json` mirrors the union of all projects' enabled packs.** When the user toggles a project's packs via `/lax:setup permissions`, Lax recomputes the union and writes it to `<project-root>/.claude/settings.local.json` `permissions.allow`. Adds are exact-string additions of the pack's rules. Removes are surgical: only strings that exactly match a Lax pack rule **and** that no other project still enables. User-added entries that don't match Lax's pack catalog are never touched.

  For non-Linear providers, the user wires their own MCP server and adds the corresponding `<provider>-read` / `<provider>-write` packs themselves; Lax v1 maintains pack catalogs only for Linear.
- **Every write is logged.** Each successful programmatic write to the ticket provider (comment create/update, Ticket create, Ticket field update) is appended as a single JSON line to `${CLAUDE_PLUGIN_DATA}/log.jsonl` (fallback `~/.claude/plugins/data/lax/log.jsonl`). One `session_id` is generated at workflow start (form: `<workflow>-<ISO timestamp>`) and reused for every entry produced by that run, including a `session_start` entry written before the first action. Entry shape:
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
  Append-only; never rewrite or truncate prior lines. The log is per-install (all projects share one file); filter by `project` / `session_id` when querying.
- **Persistent state** lives in `${CLAUDE_PLUGIN_DATA}/config.json`. Shape:
  ```json
  {
    "github_handle": "...",
    "ticket_provider": "linear|jira|github-issues|asana|trello|<custom>",
    "ticket_user_id": "...",
    "active_project": "<key>",
    "projects": {
      "<key>": { "github_repo": "owner/repo", "ticket_project_id": "...", "ticket_project_name": "...", "allowed_packs": ["gh-read", "lax-state", "linear-read", "linear-write", "linear-mcp-probe"] }
    },
    "last_sync": { "<key>": { "sync": "...", "propose": "..." } }
  }
  ```
  One lax project = one repo + one Project. Always merge, never clobber other projects or other `last_sync` keys. `ticket_provider` and `ticket_user_id` are global — one provider per lax install.
- **Subagent boundaries are strict**: gatherer never reasons about Tickets; reconciler never calls tools. The orchestrator (workflow file) is the only place that bridges the two systems.
- **Don't paraphrase workflow files** when dispatching — execute them as written.
- **Keep `README.md` in sync with the plugin code.** When you change anything user-facing — add/remove/rename a command, change command args, change a workflow's behavior, change the config schema, change the permission pack catalog, change the Lax footer / Ticket-format / logging conventions, or change the project layout — update `README.md` in the same edit. The README is the user's authoritative reference; CLAUDE.md is the implementer's reference. They must agree. Internal-only changes (prompt wording inside a subagent, refactoring evidence-string formatting, etc.) don't need a README update — only changes a user would notice or rely on.
