# lax: test

Read-only smoke test that confirms both integrations are alive for the **active lax project**. No writes, no approval loop — just fetch and display.

Throughout this file:
- **Ticket** (capital T) = a single record in the configured ticket management software.
- **Project** (capital P) = the workspace/team/board grouping in that software where Tickets live.

## 1. Resolve active lax project + permission check

Read `${CLAUDE_PLUGIN_DATA}/config.json`. Resolve `projects[active_project]` to get `ticket_project_id`, `ticket_project_name`, `github_repo`, and `allowed_packs`. Also read the global `ticket_provider`. If config is missing or has no active project, stop and tell the user to run `/lax:setup` first.

**Permission check.** This workflow requires packs `lax-state`, `gh-read`, `<provider>-read` (e.g. `linear-read` for Linear). If any required pack is missing from `projects[active_project].allowed_packs`, halt with a `permission_pack_missing` message naming the missing pack(s) and pointing the user to `/lax:setup permissions`.

## 2. Tickets: 3 latest

Use the configured ticket provider's MCP server to fetch the 3 most recently updated Tickets in `ticket_project_id` (sorted by updated date descending, limit 3). For `ticket_provider: linear`, use the `linear` MCP server. For other providers, use the MCP server / CLI the user wired up in setup.

For each Ticket, capture: identifier (e.g. `ENG-123`), title, state, assignee, updated date, URL.

## 3. GitHub: 3 latest PRs

Determine the GitHub handle from `github_handle` in config (fall back to `gh api user --jq .login` if missing).

Fetch the 3 most recent PRs authored by the user **in the active lax project's repo**, sorted by updated date:

```
gh search prs --author=<handle> --repo=<github_repo> --sort=updated --limit=3 --json number,title,state,repository,updatedAt,url
```

## 4. Display

Print two short sections side by side (or sequentially if formatting is awkward). Lead with a one-line header naming the active lax project (`<active_project>` — `<github_repo>` ↔ `<ticket_project_name>` on `<ticket_provider>`).

**Tickets (3 latest):**
1. `<ID>` — `<title>` — `<state>` — updated `<updatedAt>`
2. ...
3. ...

**GitHub (3 latest PRs):**
1. `<repo>#<number>` — `<title>` — `<state>` — updated `<updatedAt>`
2. ...
3. ...

## 5. Health check

If either fetch fails, name the failure precisely (e.g. "Ticket provider MCP returned auth error — re-auth and retry"; "`gh` is not authenticated — run `gh auth login`"). Do not retry silently.
