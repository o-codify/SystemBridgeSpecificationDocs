---
id: plugins-index
title: Plugins — Index
status: stable
version: 26.608.2205
tags: [ plugins, index ]
---

# Plugins — Index

All built-in plugins. Each plugin is an independent Go binary at
`cmd/sb-<name>/`, surfacing tools through core via stdio MCP.

See: [architecture](../architecture.md), [installation](../installation.md),
[unreal deep dive](../unreal/index.md).

## Activation triggers

A plugin only activates when its triggers match the current project. Saves
process slots + keeps the agent's tool list focused.

| Trigger source | Example |
|---|---|
| `manifest.AlwaysActive=true` | `files`, `process`, `env`, `screenshot`, `clipboard`, `network`, `search` |
| `triggers.files_present: ["*.uproject"]` | `unreal` |
| `triggers.files_present: [".git"]` (head dir) | `git` |
| `triggers.processes: ["UnrealEditor.exe"]` | `unreal` (also activates without `*.uproject` when the editor is running) |
| IDE process detection (`Code.exe`, `idea64.exe`, …) | `ide` |

## At a glance

| Plugin | Page | Standout tools |
|---|---|---|
| `files` | [files](files.md) | `tree`, `read` (with offset/limit), `read_range`, `outline`, `edit` (find/replace), `recent_changes`, `watch` |
| `unreal` | [unreal](unreal.md) | `pie_run_and_watch`, `live_coding_compile`, `bp_*` (Blueprint authoring), `editor_message_log`, `editor_status` |
| `git` | [#git](#git) | `status`, `diff`, `log_search`, `recent_commits`, `commit`, `branch_*`, `cherry_pick`, `stash` |
| `process` | [#process](#process) | `list`, `find`, `kill`, `ports`, `run_command`, `history` |
| `search` | [#search](#search) | `grep` (ripgrep wrapper), `find_files`, `xref`, `find_symbol`, `replace` |
| `browser` | [#browser](#browser) | screenshot + DOM inspection (Chrome MCP integration) |
| `ide` | [#ide](#ide) | `open`, `changes` (unsaved buffers), `recent_files`, `run_configurations` |
| `env` | [#env](#env) | `tools` (which compilers / runtimes are present), `path`, `vars`, `virtualenvs` |
| `network` | [#network](#network) | `dns`, `http_request`, `tcp_probe`, `websocket`, port scan |
| `clipboard` | [#clipboard](#clipboard) | `read`/`write` text, image, and BP-node-template variants |
| `screenshot` | [#screenshot](#screenshot) | `capture` (per-display) |
| `cpp` | [#cpp](#cpp) | header dependency graph, include analysis |
| `config` | [config](config.md) | format-aware JSON / YAML / TOML / INI editing by dotted path |
| `db` | [db](db.md) | PostgreSQL / MySQL / SQLite + Redis query and introspection |
| `docker` | [docker](docker.md) | container lifecycle + images / volumes / networks + compose |
| `scrape` | [scrape](scrape.md) | chromedp + html-to-markdown + readability + crawl |
| `semantic` | [semantic](semantic.md) | BM25 + identifier-aware code search (per-project SQLite index) |
| `build` | [build](build.md) | auto-detect Go / Node / Rust / Python / Make: build/test/lint/format/deps |
| `code` | [code](code.md) | LSP bridge (v0.1 gopls): definition / references / hover / diagnostics |
| `github` | [github](github.md) | gh CLI wrapper: 40 tools for PR / issue / CI / release / search |
| `research` | [research](research.md) | StackOverflow / HackerNews / Wikipedia / npm / PyPI / crates / go.dev |

---

## files

[Dedicated page](files.md).

Always active. The token-efficiency workhorse — replaces `cat`, `head`, `tail`,
`ls -R`, `grep -l` for typical agent flows. Key feature is that every read
respects `.senseignore` and refuses paths outside cwd.

Highlights:

- `read` accepts both `max_bytes` (binary cap) and `offset` / `limit`
  (1-indexed line slice — matches Claude Code's built-in Read semantics).
- `read_range` and `head` / `tail` for explicit windowing.
- `outline` produces a symbol-only view (Go, Python, JS/TS, Rust, C++).
- `recent_changes` walks mtimes — answers "what was edited lately?".
- `watch` subscribes to FS events surfaced through `discover()`.
- `edit` does literal find/replace with the same `replace_all` semantics
  as Claude Code's built-in Edit.

## unreal

[Dedicated page](unreal.md) — and a whole sub-section:
[unreal/](../unreal/index.md).

By far the largest plugin: ~80 tools spanning project introspection, Python
remote execution, headless Blueprint authoring, Material editing, UMG, BT,
asset queries, PIE lifecycle, Live Coding, log surfacing, and editor
recovery.

The plugin works WITHOUT the optional [SystemBridgeCompanion](../unreal/companion.md)
sub-plugin for read-only-ish tools (`read_log`, `editor_status`,
`assets_query`, basic `level_actor_*`). Authoring tools (`bp_node_create`,
`bp_override_function`, `widget_add`, `bt_node_add`, `editor_message_log`,
`bp_node_pin_set_object`, `bp_variable_add_typed`, …) require Companion.

## git

`git status`, `git diff`, `git log -S/-G`, `git recent_commits` plus the
write side (`commit`, `branch_*`, `checkout`, `add`, `reset`, `revert`,
`stash`, `cherry_pick`, `rebase`, `pull`, `push`, `fetch`, `merge`, `tag`,
`config`, `blame`). Long-form: each maps to `git <subcommand>` with parsed
output.

Tools the agent reaches for most often:

- `status` — structured (modified / staged / untracked separately).
- `diff` — defaults to working-tree vs HEAD; accepts `ref_a`, `ref_b`.
- `log_search` — pickaxe (`-S`) and content (`-G`) searches.
- `recent_commits` — last N commits with shortlog + per-commit file lists.
- `commit` — pre-checks for sensitive files (`.env`, `credentials.json`)
  before allowing them to be staged.

## process

`tasklist` / `ps`-equivalent + write side.

- `list` / `find` — by name / pid / regex.
- `tree` — parent → child traversal.
- `ports` — `netstat`-equivalent: which PID owns port N.
- `kill` — by PID or by exact image name.
- `run_command` — spawn a process; agent gets `pid`, stdout/stderr tail,
  exit code on completion.
- `history` — recent process events the daemon observed.

## search

Higher-level alternatives to `grep` / `find` that integrate with the
project's index:

- `grep` — ripgrep wrapper; honors `.senseignore`; structured output with
  `file:line:col:match`.
- `grep_context` — like `grep -A -B` with bounded context.
- `find_files` — glob (`**/*.go`); fast.
- `recent_files` — `recently modified` view via FS mtime + IDE recent list.
- `find_symbol` — language-aware symbol lookup (defers to `outline`).
- `xref` — best-effort cross-reference (mostly relies on text scan).
- `replace` — bulk find/replace across files with diff preview.

## browser

For web work the integration leans on Claude-in-Chrome MCP tools. The `sb`
browser plugin provides:

- Screenshot of a tab / page.
- Recent navigation history.
- Network capture summary (when the Chrome extension is connected).

For full DOM interaction (click, fill, navigate) prefer the Chrome MCP
directly — `sb-browser` is intentionally minimal.

## ide

Light-touch integration with whatever IDE the user is in:

- `info` — which IDE is running, version, project root.
- `recent_files` — pulled from IDE-specific recent lists (JetBrains
  `.idea`, VS Code `state.vscdb`, etc.).
- `changes` — unsaved buffers (when the IDE exposes a side channel).
- `open` — open a file at line:col in the running IDE.
- `run_configurations` — list of saved run configs / launch.json entries.

## env

Pure introspection — no writes:

- `info` — OS, arch, hostname, uptime.
- `path` — parsed PATH.
- `which` — locate a command on PATH.
- `tools` — versioned report of common compilers / runtimes (`go`, `python`,
  `node`, `dotnet`, `rustc`, `clang`, `cl`, …).
- `vars` — env vars (filtered).
- `virtualenvs` — detected Python venvs / conda envs / Node `package.json`
  locations.

## network

- `dns` — A/AAAA/CNAME resolution.
- `http_request` — issue HTTP and return status, headers, body (capped).
- `tcp_probe` — connect-test a host:port.
- `websocket` — open WS, send N frames, read reply.
- `config` — local NIC config.
- `ports` — port → owner mapping (similar to `process.ports`).

## clipboard

- `read` / `write` text.
- `read_image` / `write_image`.
- `clipboard_bp_node_template` — UE-specific: capture a Blueprint
  copy/paste text blob from the clipboard. UE's BP editor accepts these
  blobs via Ctrl+V to instantiate node subgraphs — useful when the agent
  has built one elsewhere.

## screenshot

- `capture` — per-display PNG to a path.
- `displays` — list connected displays + their primary/secondary status.

## cpp

C++ static analysis, useful for both standalone C++ projects and UE C++
modules:

- Header dependency graphs.
- Include path analysis.
- Symbol references across translation units (best-effort).

(Originally part of the unreal plugin; split out so non-UE C++ projects
also benefit.)

---

## Cross-references

- [files in depth](files.md)
- [unreal in depth](unreal.md), plus the
  [unreal deep dive](../unreal/index.md) section.
- [Common workflows](../workflows/index.md) show how these plugins
  compose.
