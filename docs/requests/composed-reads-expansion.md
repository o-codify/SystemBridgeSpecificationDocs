---
id: request-composed-reads-expansion-sb-views-recipe-library
title: "Request: composed-reads expansion (sb-views recipe library)"
status: request
version: 26.608.1635
tags: [ sb-core, views, composition, request ]
---

# Request: composed-reads expansion (sb-views recipes)

**Origin.** github-inside-claude-code's `gh_query_*` family —
composed reads that combine multiple primitive API calls into one
AI-friendly response (`my_dashboard`, `pr_reviews`, `repo_activity`).
SB already has `views.*` for the same idea; under-populated.

## What

Add a registry of pre-built recipes across our plugins, exposed via
`views.run(recipe_name, args)`. Each recipe is a tiny script that
calls primitive tools, merges results, and returns the structured
shape AI actually wants.

## Initial recipes

| View | What it composes |
|---|---|
| `dev_session_overview` | files.tree + git.status + processes.list + recent_changes |
| `project_health` | build.test + lint + deps_outdated + git.log -10 |
| `pr_overview(repo, pr)` | github.pr_read + github.pr_files + github.pr_checks |
| `ci_state(repo)` | github.run_list (last 10) + github.workflow_list + last_failure_summary |
| `unreal_project_state` | unreal.editor_status + assets_scan_offline (cached) + message_log |
| `db_inventory(conn)` | db.schema + per-table db.describe + row_count for top-10 |
| `docker_dev_stack` | docker.compose_ps + docker.stats (per container) + recent compose_logs |

## Why over individual tool calls

- One call = one AI turn vs 5-10
- Cross-plugin joins (git + files + processes) become trivial
- Recipe results are cacheable (TTL on view registration)
- AI doesn't have to memorise tool order or arg shapes per workflow

## Implementation route

`internal/views/recipes/<name>.go` per recipe. Each implements:

```go
type Recipe interface {
    Name() string
    TTLSeconds() int
    Run(ctx, host pluginhost.Host, args map[string]any) (any, error)
}
```

Recipes call `host.Plugin("files").CallTool(...)` etc — uses existing
in-process dispatch. Errors get wrapped in `errcodes.PartialFailure`
(per the errcodes request) so a partial view succeeds with notes.

## Acceptance

- `views.list()` shows 7+ recipes.
- `views.run("dev_session_overview")` returns merged JSON < 200ms.
- Recipes degrade: if one plugin is down, its slice has
  `{available: false, error: "..."}` but the view still succeeds.

## Non-goals (defer)

- User-defined recipes (no DSL).
- Recipe parameter introspection (just a description string).
