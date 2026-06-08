---
id: request-binary-archive-inspection-strings-format-detect-zip-tar
title: "Request: binary + archive inspection (strings / format-detect / zip-tar)"
status: request
version: 26.608.1426
tags: [ sb-core, files, binary, archive, request ]
---

# Request: binary + archive inspection

**Origin.** In the v1.14.0 hotfix the AI verified an embedded
`sb_helpers.py` inside a Go binary by shelling out to a Python
one-liner that scanned for byte strings. Three primitives would have
made it a single tool call.

## What

Three small read-only tools, no shellouts, structured output.

## Tools

| Tool | Purpose |
|---|---|
| `binary_strings(file, min_len=4, limit=200)` | Extract printable strings from any binary (Unix `strings` equivalent). Returns `[{offset, text}]` ordered by offset. |
| `binary_format(file)` | Magic-byte sniff. Returns `{kind, mime, details}` — e.g. `{kind:"pe", details:{arch:"x64", subsystem:"console"}}` for a Windows .exe, or `{kind:"png", details:{width, height}}`. Coverage target: PE / ELF / Mach-O / ZIP / TAR / PNG / JPG / WEBP / PDF / .uasset / .umap. |
| `archive_list(file)` | List entries in a `.zip` / `.tar` / `.tar.gz` / `.tgz`. Each entry: `{path, size, mode, mtime, is_dir}`. |
| `archive_extract(file, member?, dest)` | Extract one member or the whole archive. Safe-path checks (no `..` escape). |

## Why

Concrete patterns that currently shell out:

- "Does this binary embed my new `sb_helpers.py`?" → `binary_strings`
  with the marker text instead of `python -c 'open()'.read()` byte
  scan.
- "What kind of file is this?" → `binary_format` instead of `file`
  utility (which isn't on Windows by default).
- "What's inside this `.zip`?" → `archive_list` instead of `tar -tzf`
  / `7z l`.
- "Pull one file from the archive to inspect" → `archive_extract`
  without leaving artifacts in cwd.

`files_hex` already exists — these complement, not replace.

## Implementation route

Extend the existing `sb-files` plugin (no need for a new binary):

- `binary_strings`: scan for printable runs ≥ min_len, ASCII + UTF-16
  LE (Windows-prevalent).
- `binary_format`: magic-byte table + a few `os.Stat`-based fallbacks.
- `archive_*`: stdlib `archive/zip` + `archive/tar` + `compress/gzip`.

All read-only. Bounded memory (stream-decode archives; `binary_strings`
honours `limit`).

## Acceptance

- `binary_strings("sb-unreal.exe", min_len=6)` finds `SystemBridgeBindings`
  and other long literals.
- `binary_format("sb.exe")` returns `{kind:"pe", details:{arch:"x64"}}`.
- `archive_list("plugin.zip")` returns the same paths as `unzip -l`.
- `archive_extract` refuses an entry whose path resolves above `dest`.

## Non-goals (defer)

- Writing into archives (zip create / tar create).
- Decompression of formats beyond gzip (bz2, xz, zstd) — common enough
  to add later; not blocking.
- Code-signature verification.
