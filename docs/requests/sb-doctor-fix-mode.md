---
id: request-sb-doctor-fix-auto-clean-stale-processes-manifest-drift-locked-binaries
title: "Request: sb doctor --fix — auto-clean stale processes, manifest drift, locked binaries"
status: request
version: 26.608.1428
tags: [ sb-core, doctor, quality-of-life, request ]
---

# Request: sb doctor — fix mode + missing diagnostics

**Origin.** v1.14.0 hotfix surfaced several "should have caught
itself" failure modes that bit the AI:

1. `bin/plugins/unreal/manifest.json` was stale relative to
   `bin/plugins/unreal/unreal.exe` after a `go build` bypassing
   `scripts/build.sh`. MCP client received the old tool list →
   transport mismatch.
2. 6 leftover `sb.exe` processes from past sessions — confused
   the "which one is the host using?" triage.
3. Plugin `.exe~` tilde-backup left behind by Windows file lock
   during rebuild.
4. Walrus-in-dict-key SyntaxError in embedded `sb_helpers.py`
   shipped because no pre-flight Python compile check ran.

All four are detectable + fixable mechanically. They shouldn't
need a human (or AI) to notice.

## What

Extend the existing `sb doctor` command with detection of each
failure mode, plus a `--fix` switch that applies the obvious
remediation.

Also expose as MCP tools so the AI can run the same checks
introspectively.

## Diagnostics + remediations

| Check | Detect | Fix |
|---|---|---|
| Manifest drift | mtime(`unreal.exe`) > mtime(`manifest.json`), or `--print-manifest` SHA ≠ stored manifest | Re-run `--print-manifest > manifest.json` |
| Stale sb.exe processes | More than 2 `sb.exe` running (parent process dead / no MCP transport active) | `process_kill` non-parent stale ones |
| Plugin `.exe~` backups | Find `*.exe~` under `bin/plugins/` | Delete (the live `.exe` is the one being used) |
| Embedded helper Python | `py_compile` the embedded `sb_helpers.py` from each plugin binary | No fix — fail the build / report |
| Audit log growth | Audit log > 100 MB | Rotate via `sb logs rotate` (also new) |
| Skill drift | `~/.claude/skills/systembridge/` vs embedded `cmd/sb/skill/systembridge/` differ | Re-sync from embedded |
| Plugin version drift | Plugin reports a different version than its sidecar manifest | Re-print manifest |

## Tools (MCP surface)

| Tool | Purpose |
|---|---|
| `doctor_run(fix=false, checks?: [...])` | Run all checks (or a subset). With `fix=true`, applies the remediations. Returns `{checks: [{name, ok, message, fix_applied?}]}`. |
| `doctor_check(name)` | Single-check shortcut. Same return shape. |

## Why

The v1.14.0 hotfix loop was: notice the AI's call returns SyntaxError →
investigate → discover the manifest mismatch → kill processes → rebuild
manifest → re-test. Six steps. `doctor_run(fix=true)` should make this
one call.

Even without `--fix`, just having `doctor_run` report "manifest is stale
by 2 hours" is useful — AI can call it after any build and know if the
deploy was clean.

## Implementation route

Each check is a Go function returning `Result { Name, OK, Message, Fix?
func(ctx) error }`. The doctor command iterates, prints (or returns
structured), and conditionally calls `Fix`.

Most checks are file-stat + magic-byte / shell-out free:

- Manifest drift: stat + `--print-manifest` to a temp file + compare.
- Stale processes: existing `process_list` already discovers them.
- Tilde backups: `filepath.Walk` for `*.exe~`.
- Skill drift: `filepath.Walk` + hash compare.

## Pre-ship build hook (separate but related)

`scripts/build.sh` should add a pre-flight `py_compile` step for
**every embedded Python source** under `cmd/sb-*/`:

```bash
for py in $(find ./cmd -name '*.py'); do
    python -m py_compile "$py" || { echo "py_compile failed: $py"; exit 1; }
done
```

Catches the walrus class of bug. Can also live in a Go test
(`python_compile_test.go` already shipped for `sb_helpers.py`, but
should be generalised).

## Acceptance

- After `go build -o bin/plugins/unreal/unreal.exe ./cmd/sb-unreal`
  WITHOUT manifest regen, `doctor_run()` reports `manifest_drift:
  fail` with mtime + content-hash info.
- `doctor_run(fix=true)` regenerates the manifest.
- `doctor_run()` lists every `sb.exe` instance and flags ones whose
  parent is dead.
- The pre-ship `py_compile` step rejects a commit with a SyntaxError
  in any `cmd/**/*.py`.

## Non-goals (defer)

- Health monitoring (periodic doctor checks in background) — push out.
- Self-repair of remote MCP server crashes — they own their lifecycle.
