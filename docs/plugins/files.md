---
id: plugin-files
title: "Plugin: files"
status: stable
version: 26.601.1817
tags: [ plugin, files ]
---

# Plugin: `files`

Project-scoped file access. Always active. Replaces `cat`/`head`/`tail`/`ls`/
`grep -l` for typical agent flows.

See also: [plugins index](index.md), [search plugin](index.md#search) (for
content scans, prefer search-plugin grep over scripting around `files.read`).

## Permission scope

- `read_files:project` — every read tool gates here.
- `write_files:project` — gated on `write`, `append`, `edit`, `create_dir`,
  `move`, `delete`. Requested in manifest so the consent prompt surfaces it.

Every path:

1. Must resolve under cwd (`pathUnder(target, cwd)`). Symlinks chasing is
   not enabled — lexical containment only.
2. Must NOT match `.senseignore` (same syntax as `.gitignore`). The plugin
   returns `permission_denied:senseignore` on a hit.
3. May refuse binary content via `binary_file:` heuristic (NUL byte in
   leading sniff) — unless the caller used `hex` explicitly.

## Tools

### `tree`

List a directory recursively with bounded depth. Capped at 2000 entries by
default (`limit` up to 10000). Excludes common build/cache dirs
(`.git`, `node_modules`, `bin`, `obj`, `Saved`, `Intermediate`, …).

Args:

- `path` — defaults to cwd.
- `depth` — default 3, ≥1.
- `limit` — default 2000, max 10000.

Returns `entries: [{name, kind, size_bytes?}]` with a `truncated:true` +
`hint` when the cap fires. The hint nudges the agent toward a narrower
query — e.g. `unreal_assets_query` for UE Content/, or `search.find_files`
for glob lookups.

### `read`

Read a text file. Refuses binary content. Two modes:

- **Whole-file** (default) — returns content up to `max_bytes`
  (default 102400 = 100 KB) and `truncated: bool`.
- **Line-slice** — pass `offset` (1-indexed start line) and/or `limit`
  (number of lines). When sliced, the response includes
  `start_line` / `end_line` / `total_lines` / `slice_mode: true` so the
  agent can page. Matches Claude Code's built-in Read tool semantics.

When `offset` is past EOF the response is `content: ""` with the real
`total_lines` so the agent sees no-more-data without a separate stat.

### `read_range`

Explicit `line_start`/`line_end` window (1-indexed, inclusive). Capped at
1000 lines per call. Use when you prefer the explicit vocabulary; `read`
with `offset`/`limit` is otherwise equivalent.

### `head`

First N lines (default 50, max 1000). Convenience over `read_range 1..N`.

### `tail`

Last N lines (default 50, max 1000). Reads backward from EOF in 64KB
chunks — works on huge logs without loading the whole file.

### `diff`

Unified diff between two project files. Line-based LCS. Returns
`identical: true` if files are byte-equal.

### `outline`

Symbol-only view: function / class / struct / enum / interface signatures
with line numbers. Languages: Go, Python, JavaScript, TypeScript, Rust,
C++. Pure regex extractors (no compiler invocation) so it's fast on big
files but won't catch every esoteric form.

Use case: "show me the API surface of `xxx.go` without reading the
implementations."

### `read_many`

Multiple files in one call. Optional per-file line ranges. Saves N
round-trips when the agent knows the set upfront (e.g. follow-up after
`grep` returned 5 hits).

Args:

```
{ files: [
    { path, line_start?, line_end?, max_bytes? },
    …
]}
```

### `stat`

Cheap metadata, no content: size, mtime, kind (file / dir / symlink),
ext, binary heuristic, and (for text) line count.

### `hex`

Hex dump of a byte range. Useful for binary inspection (file headers,
format probing). Args: `path`, `offset` (bytes), `length` (default 256,
max 65536).

### `recent_changes`

Files modified in the last N seconds (default 86400 = 24h). Sorted by
mtime desc. Capped at `limit` entries. Useful as "what was I working on
yesterday?" or after a build.

### `watch`

Subscribe to filesystem events. Events appear on the next `discover()` in
the files plugin's event ring:

- `file_changed`
- `file_created`
- `file_deleted`
- `file_watch_error` (subscription failure / permission)

Caller passes `paths: [string]` (relative globs) and gets a subscription
ID back. Unwatch via `unwatch(subscription_id)`.

### `write`

Atomic write to a file via tmp + rename. Gated by `write_files:project`.

```
{ path, content, mode?, overwrite? }
```

- `overwrite=false` (default) refuses to clobber existing files.
- `mode` is an optional octal mode (Unix; ignored on Windows).

### `append`

Append bytes to a file. Creates the file if missing. Gated.

### `edit`

Literal find/replace, one or all occurrences. Same semantics as Claude
Code's built-in Edit:

```
{ path, old_string, new_string, replace_all? }
```

Refuses on ambiguous match (multiple hits) unless `replace_all=true`.
Atomic via tmp + rename. Gated.

### `create_dir`

`mkdir -p`. Idempotent. Gated.

### `move`

Rename / move within cwd. `overwrite=false` by default. Gated.

### `delete`

Delete file or directory. Idempotent — `already_absent: true` if the path
didn't exist. `recursive=true` required for non-empty dirs. Gated.

## Discover summary

Each `discover()` returns:

```
files: {
  cwd: "C:/Users/…/MyProject",
  topLevelEntries: [{name, kind, size_bytes?}, …],   // capped at 100
  recentChanges:   [{path, mtime, size_bytes}, …],   // last 10 files
  truncated: bool,
  events_since_last_call: [ {kind: "file_changed", path, …}, … ]
}
```

`topLevelEntries` is capped to 100 to keep `discover()` output small; the
agent uses `tree` for deeper navigation.

## Events

| Event | Severity | Payload |
|---|---|---|
| `file_changed` | info | `{path, mtime}` |
| `file_created` | info | `{path, mtime}` |
| `file_deleted` | info | `{path}` |
| `file_watch_error` | warn | `{path, error}` |
| `summary_complete` | info | (debug) summary built |

## Patterns

- **"Read the failing test."** → `grep` (search plugin) → `read` with
  `offset`/`limit` around the hit line.
- **"What's in this 50 MB log?"** → `tail` with `lines=200` first;
  `read_range` for windows.
- **"Show me the function signatures in foo.go."** → `outline`.
- **"Refactor a constant across 6 files."** → `search.grep` to find →
  `read_many` to confirm context → `edit` per file.
