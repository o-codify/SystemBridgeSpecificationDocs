---
id: request-sb-config-format-aware-json-yaml-toml-ini-editing
title: "Request: sb-config — format-aware JSON/YAML/TOML/INI editing"
status: request
version: 26.608.1426
tags: [ sb-core, config, editing, request ]
---

# Request: sb-config — format-aware config editing

**Origin.** Self-audit during the v1.14.0 session. AI uses `files_edit`
for config edits today; works, but every call is a manual "find unique
verbatim snippet → write replacement". For structured files (JSON /
YAML / TOML / INI) this loses the format awareness — one misplaced
comma and the file is broken until a separate `Read` confirms.

## What

A small plugin that gives the AI **path-addressed** edits on structured
configs, parsing/serialising via real format libraries so syntax stays
valid.

## Tools

| Tool | Purpose |
|---|---|
| `config_get(file, path)` | Read a value by dotted path (`spec.tools[3].name`). Auto-detects format from extension. Returns `{value, type, format}`. |
| `config_set(file, path, value, type?)` | Set a value at path. Creates intermediate objects/arrays as needed. Atomic write. |
| `config_unset(file, path)` | Delete a key / array element. Idempotent. |
| `config_merge(file, json_object)` | Deep merge an object into the root (object keys overwrite, arrays append by default — `array_strategy` switches to replace/dedup). |
| `config_validate(file, schema?)` | Reparse + optional JSON Schema check. Returns `{ok, errors[]}`. |

Path syntax: dotted (`a.b.c`), with `[N]` for arrays. Quoting via
double-quotes for keys containing dots (`a."key.with.dots".b`). Same
syntax across formats; library serialises back in the original format.

## Why

Currently every config edit is:

1. `files_read` to see structure
2. `files_edit` with verbatim find/replace snippet
3. `files_read` again to verify nothing broke

With `config_set`:

```
config_set(".vscode/settings.json", "editor.tabSize", 4)
```

One call, atomic, can't break syntax. Same for INI (`[section] key`),
YAML (multi-doc supported), TOML.

## Implementation route

Go plugin `cmd/sb-config/`. Parsers:

- JSON: `encoding/json`
- YAML: `gopkg.in/yaml.v3` (preserves comments via Node API)
- TOML: `github.com/pelletier/go-toml/v2`
- INI: `gopkg.in/ini.v1`

Path resolution: small custom parser, shared across formats. Atomic
writes via tmp file + rename (existing `files` plugin pattern).

Auto-detection: file extension (`.json` / `.yaml` / `.yml` / `.toml` /
`.ini`) with an optional `format` override.

## Acceptance

- `config_set("package.json", "version", "1.2.0")` updates one field
  and leaves every other byte untouched (verifiable via byte diff).
- `config_get("Cargo.toml", "dependencies.serde")` returns the inline
  table as JSON.
- `config_set` on a YAML file preserves comments and key order.
- Invalid path syntax returns a structured error pointing to the bad
  token, not a stack trace.

## Non-goals (defer)

- XML (rarely needed, large surface).
- Format conversion (`config_convert json → yaml`).
- Schema-driven autocompletion of values.
