# lax: setup

Bootstrap the `lax` plugin or add another lax project to it. Each `/lax:setup` run configures **one lax project** — a (GitHub repo, Project) pair under a short name. Run `setup` again to add or edit additional lax projects. Walk through this checklist in order; report each step's outcome and only continue once the prior step is healthy.

Throughout this file:
- **Ticket** (capital T) = a single record in the configured ticket management software (e.g. a Linear issue, Jira ticket, GitHub Issue, Asana task, Trello card).
- **Project** (capital P) = the workspace/team/board grouping in that software where Tickets live (e.g. a Linear team, a Jira project, a GitHub repo's Issues, an Asana project, a Trello board).

**Subcommand argument shortcut:** if `$ARGUMENTS` equals `permissions`, skip directly to step 8 (permissions) and skip steps 1–7. If `config.json` doesn't exist yet, error out and tell the user to run full `/lax:setup` first.

## 1. GitHub CLI

Run `gh auth status`. If it fails or reports unauthenticated, tell the user to run `gh auth login` in a separate shell, then re-invoke `/lax:setup`. Do not proceed past this step until `gh auth status` is clean.

## 2. Pick the ticket management software

Read `${CLAUDE_PLUGIN_DATA}/config.json` if it exists. If `ticket_provider` is already set, show it and confirm with the user before changing.

If unset (or the user wants to change), present this list and ask the user to pick one (or enter a custom one):

1. **Linear**
2. **Jira** (Atlassian)
3. **GitHub Issues**
4. **Asana**
5. **Trello**
6. **Custom** — user supplies a short name (lowercase, hyphen-separated)

Save the choice as `ticket_provider` in the config. The plugin ships **without** any MCP server pre-registered — the user is responsible for wiring up their chosen provider's MCP server in step 3 (a one-time task per machine).

## 3. Wire and verify ticket-provider connectivity

The plugin ships provider-neutral, so this step depends on `ticket_provider`:

### 3a. Make sure the provider's MCP server is registered

Check whether a server matching the provider is already visible to Claude Code (via `~/.claude/.mcp.json`, `<project-root>/.mcp.json`, or `/mcp` in-session registration).

**If not registered, give the user a copy-paste snippet** scoped to their choice. For Linear, suggest adding to `<project-root>/.mcp.json`:

```json
{
  "mcpServers": {
    "linear": {
      "type": "sse",
      "url": "https://mcp.linear.app/sse"
    }
  }
}
```

(Note: Linear's `sse` transport is being deprecated in favor of `mcp` at `https://mcp.linear.app/mcp` — prefer that if available.)

For other providers, list known MCP servers if one is widely available (GitHub Issues has a community/Anthropic-published MCP server; Atlassian, Asana, Trello vary — point the user at their provider's documentation). For **Custom**, the user supplies their own.

After the user adds the entry, ask them to run `/mcp` to enable the server and complete any OAuth flow.

### 3b. Verify with a viewer/me query

Call the provider's MCP — for Linear that's a simple call against the `linear` server (e.g. fetch the authenticated viewer). For other providers, use their analogous "who am I?" tool.

If the call errors with auth, send the user back to `/mcp` to re-auth, then re-invoke `/lax:setup`. Do not proceed past this step until the viewer/me call succeeds.

If no MCP server exists for the chosen provider yet, warn the user that `/lax:sync`, `/lax:update`, and `/lax:propose` will not work until one is wired up — but allow setup to continue so the rest of the config is captured (the user can come back to this step by re-running `/lax:setup`).

## 4. Resolve identity

- GitHub handle: `gh api user --jq .login`
- Ticket-provider user: from the connectivity check in step 3, capture the user identifier in whatever shape the provider returns (Linear user ID, Jira accountId, GitHub login, Asana GID, Trello idMember, etc.). Store as `ticket_user_id`.

## 5. Pick the lax project to configure

Show the user the existing `projects` map (keys only) and ask:
- A short project key (e.g. `widget`, `gizmo`) — used by `/lax:project` and as the `last_sync` key. Lowercase, hyphen-separated. If the key already exists, confirm overwrite.
- A `ticket_project_id` + `ticket_project_name` — these correspond to the **Project** in the configured provider (Linear team, Jira project, GitHub Issues repo, Asana project, Trello board). Pick from the user's available Projects when possible (use the provider's MCP to list them).
- A `github_repo` in `owner/repo` form (single repo per lax project).
- The new project's `allowed_packs` defaults to `[]` (all disabled) at this step — the permissions step (step 8) is where the user opts in.

## 6. Persist config

Write `${CLAUDE_PLUGIN_DATA}/config.json` with this shape:

```json
{
  "github_handle": "...",
  "ticket_provider": "linear",
  "ticket_user_id": "...",
  "active_project": "...",
  "projects": {
    "<key>": {
      "github_repo": "owner/repo",
      "ticket_project_id": "...",
      "ticket_project_name": "..."
    }
  },
  "last_sync": {
    "<key>": {}
  }
}
```

Merge rules:
- If the file already exists, read it first.
- **Never clobber other projects** in the `projects` map — only add or overwrite the one being configured.
- **Never clobber `last_sync` keys** for any project. If the project is new, initialize `last_sync[<key>] = {}`.
- If `active_project` is unset (first run), set it to the lax project being configured. If it's already set, leave it alone — the user can change it with `/lax:project`.
- `github_handle`, `ticket_provider`, and `ticket_user_id` are global; set them on first run, leave alone afterward unless the user explicitly chose to change provider in step 2.

## 7. Confirm

Print the resolved config back to the user (one field per line, with the `projects` map fully expanded) so they can verify before any sync runs. Highlight which lax project is currently `active_project` and which `ticket_provider` is in use.

## 8. Configure permissions (per lax project)

This is the last step of normal setup, **and** the only step run when `$ARGUMENTS == "permissions"`.

### 8a. Pick the lax project to configure

If running as part of normal setup, the target lax project is the one just configured in step 5. If running via `$ARGUMENTS == "permissions"`:
- If the user provided a second arg (`/lax:setup permissions <key>`), use that key.
- Else, default to `active_project` from config but list the available projects and confirm with the user before proceeding.

If `config.json` is missing or has no projects, error out and tell the user to run full `/lax:setup` first.

### 8b. Read current per-project allowed_packs

Look up `projects[<target>].allowed_packs` in `config.json`. If unset, treat as `[]` (all packs disabled for that project).

### 8c. Show packs and prompt

For each pack defined in CLAUDE.md ("Pack catalog"), mark `[x]` if it's in the project's `allowed_packs`, else `[ ]`. Show a numbered list scoped to the target project:

```
Permission packs for project <target>:

  1. [x] gh-read           — GitHub read via gh CLI
  2. [x] lax-state         — Read/write Lax config + log files
  3. [ ] linear-read       — Linear MCP read tools
  4. [ ] linear-write      — Linear MCP write tools  (REQUIRED for /lax:sync, /lax:update, /lax:propose)
  5. [x] linear-mcp-probe  — curl health checks against mcp.linear.app

Toggle: type pack numbers (e.g. "2,4" or "2 4"), "all" to enable all, "none" to disable all,
or "done" to accept current state.
```

Loop on user input until "done": flip the targeted packs in the working set, re-display.

### 8d. Persist per-project + recompute union

When the user types "done":
1. Write the working set as `projects[<target>].allowed_packs` in `config.json`. Do not touch any other project's `allowed_packs`, `last_sync`, or other fields.
2. Recompute the **union**: for every project in `projects`, gather `allowed_packs`, dedupe, expand each pack to its rule list (per the catalog in CLAUDE.md, with `<plugin-data>` resolved to `$HOME/.claude/plugins/data` and similar).
3. Read `<project-root>/.claude/settings.local.json` (create with `{"permissions": {"allow": []}, "enabledMcpjsonServers": []}` if missing).
4. Compute the diff against `permissions.allow`:
   - **Add**: rules in the union that are missing from `permissions.allow`.
   - **Remove**: rules in `permissions.allow` that **exactly match** a Lax pack rule **and** that are not in the union (i.e. no project enables them anymore). Never remove entries that don't match a Lax pack rule string-for-string — those are user-added or come from other tools.
5. Apply the diff. Preserve `enabledMcpjsonServers`, `hooks`, `env`, and all other settings unchanged.

### 8e. Confirm

Print:
- The project's new `allowed_packs` (one per line).
- The diff applied to `permissions.allow` (added / removed strings).
- A warning if any required pack is disabled for that project — name the workflows that will halt because of it (e.g. "linear-write disabled → `/lax:sync`, `/lax:update`, `/lax:propose` will halt with `permission_pack_missing` for this project until enabled").
