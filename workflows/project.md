# lax: project

Change which lax project is currently active. Subsequent `/lax:test`, `/lax:sync`, `/lax:update`, and `/lax:propose` runs use the active lax project's GitHub repo and the corresponding Project in the configured ticket management software.

Throughout this file:
- **Ticket** (capital T) = a single record in the configured ticket management software.
- **Project** (capital P) = the workspace/team/board grouping in that software where Tickets live.

The subcommand argument (if provided) is the project key to switch to.

## 1. Load config

Read `${CLAUDE_PLUGIN_DATA}/config.json`. If it doesn't exist, stop and tell the user to run `/lax:setup` first.

## 2. Resolve target project

- If the subcommand argument is provided and matches a key in `projects`, that's the target.
- If the argument is provided but doesn't match, stop and list the available project keys so the user can re-run with a valid one.
- If no argument is provided, list all lax projects (key + repo + Project name, current `active_project` marked) and ask the user which to switch to.

## 3. Update active_project

Write the config back with `active_project` set to the target key. Do not touch `projects`, `last_sync`, `github_handle`, `ticket_provider`, or `ticket_user_id`.

## 4. Confirm

Print the new active lax project: key, `github_repo`, `ticket_project_name` (on `ticket_provider`), and the current `last_sync[<key>]` cursor (or "never synced" if empty).
