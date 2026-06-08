---
id: plugin-config
title: "Plugin: config"
status: stable
version: 26.608.1652
tags: [ plugin, config, json, yaml, toml, ini ]
---

# Plugin: `config`

Format-aware editing of structured config files (JSON / YAML / TOML / INI)
by **dotted path**. Atomic writes, format detection, preserves original
shape on round-trip.

Solves the problem that `files.edit` works at byte level — one stray
comma and your config is broken. `config.set` parses, mutates by path,
and re-serialises through the format library — syntax stays valid by
construction.

## Tools

| Tool | Purpose |
|---|---|
| `config.get(file, path, format?)` | Read a value by dotted path. Returns `{value, type, format, exists}`. |
| `config.set(file, path, value, format?)` | Set a value. Creates intermediate objects/arrays. **Risk: medium.** Atomic (tmp + rename). |
| `config.unset(file, path, format?)` | Delete a key / array element. **Risk: high.** Idempotent. |
| `config.merge(file, object, path?, array_strategy?, format?)` | Deep-merge a JSON object into the file. **Risk: medium.** |
| `config.validate(file, schema?, format?)` | Reparse; optional JSON Schema check. |
| `config.detect(file)` | Auto-detect format from extension or content sniff. |

## Path syntax

Shared across all four formats:

```
a.b.c                   nested keys
a.b[0]                  array index
a."key.with.dots".b     quoted key (for keys containing dots)
[0][1]                  nested array
```

## Examples

Update `package.json` version (one call, no syntax risk):

```
config.set({
  file: "package.json",
  path: "version",
  value: "\"1.2.0\""
})
```

Deep-merge YAML feature flags:

```
config.merge({
  file: ".github/workflows/ci.yml",
  path: "jobs.test.env",
  object: "{\"CI\":\"true\",\"NODE_ENV\":\"test\"}"
})
```

## Implementation notes

- JSON: `encoding/json` with `UseNumber()` to preserve int/float distinction.
- YAML: `gopkg.in/yaml.v3` — comments + key order preserved via Node API.
- TOML: `github.com/pelletier/go-toml/v2`.
- INI: `gopkg.in/ini.v1` — top-level scalars become DEFAULT keys; sub-objects become named sections.
- Atomic write: `<dir>/.config-*.tmp` → fsync → rename.
- JSON value coerce: pass strings as `"foo"`, numbers as `42`, booleans as `true`. Falls back to plain string if JSON parse fails.

## Cross-references

- [Plugin: files](files.md) — byte-level editing (when path-based mutation isn't enough)
- [Architecture: risk labels](../architecture-risk-labels.md)
