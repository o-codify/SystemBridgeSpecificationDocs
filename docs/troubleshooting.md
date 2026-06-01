---
id: troubleshooting
title: Troubleshooting
status: draft
version: 26.601.1508
tags:
  - troubleshooting
---

# Troubleshooting

Common failure modes and the fix path. Cross-references go back to the
authoritative reference doc for each topic.

## Companion drift

**Symptom.** `editor_status` shows
`companion_version_mismatch: true` — e.g. loaded `1.3.0`, expected `1.3.4`.

**Cause.** sb-unreal was upgraded but the C++ companion DLL in the UE
plugin folder is older.

**Fix.**

```
unreal_companion_rebuild
```

If the editor holds the DLL locked (rebuild fails at the link step):

1. Close the editor.
2. `unreal_companion_rebuild` again.
3. Re-open editor.

See [companion plugin](unreal/companion.md#upgrading-the-companion).

## Editor keeps relaunching after I kill it

**Symptom.** Agent calls `process_kill UnrealEditor.exe`; a new one
shows up ~25s later.

**Cause.** The [crash watcher](unreal/crash-recovery.md) is doing what
it's designed to do — recover from crashes — and treats your kill as a
crash.

**Fix.** Suppress before killing:

```
unreal_editor_set_auto_recover enabled=false ttl_seconds=600
# do your stuff, e.g. project_build
unreal_editor_set_auto_recover enabled=true
```

OR use `unreal_editor_restart` which self-suppresses.

OR (escape hatch) kill the `unreal.exe` plugin daemon — but that disables
the watcher for everything until Claude Code restarts.

## Editor opens but my change isn't visible

**Symptom.** Live Coding said `succeeded`; the behavior didn't change in
PIE.

**Causes.**

1. Live Coding can't reliably patch the kind of change you made (UCLASS
   reflection, ABI). Use `project_build` instead. See
   [headless rebuild](workflows/headless-rebuild.md).
2. The editor was relaunched by the [crash watcher](unreal/crash-recovery.md)
   from a stale DLL — check `editor_status.last_crash.recovered`. If
   recovered: true and the timestamp aligns with the issue, the recovered
   editor has the OLD DLL. Solution: `project_build` (full link) then
   `editor_restart`.
3. The Blueprint that consumes the C++ feature wasn't recompiled. Use
   `bp_compile_and_save` on it.

## Live Coding timeout

**Symptom.** `live_coding_compile` returns `status: "timeout"`.

**Common causes** (per the hint):

1. Action-limit popup is open in `LiveCodingConsole.exe`. Fix:
   `live_coding_compile skip_action_limit=true`. Cold-restarts LCC.
2. Compile genuinely takes longer than `wait_seconds`. Fix: raise
   `wait_seconds` (up to 1800).
3. `LiveCodingConsole.exe` died. Fix: taskkill it, retry — UE respawns.

See [Live Coding](unreal/live-coding.md).

## PIE reports started but never actually plays

**Symptom.** `pie_start` / `pie_run_and_watch` returns
`success: true, started: true` but PIE world never appears. A modal
"Are you sure?" dialog is sitting in the editor.

**Cause.** A Blueprint has `BS_Error` compile status; UE pre-flight
shows the dialog. The legacy tool dispatched anyway and reported success.

**Fix (Companion v1.3.3+).** The new pre-flight catches this and
returns:

```jsonc
{
  "success": false,
  "error": "blueprint_compile_errors",
  "failing_blueprints": [{"path": "...", "status": "BS_Error", "source": "companion_live"}]
}
```

Fix the BP (via `bp_compile_and_save` or the BP editor), then retry.

If you see this without v1.3.3+: upgrade the companion via
`unreal_companion_rebuild` (the v1.3.3 release includes the live-walk
of `UBlueprint::Status` via `TObjectIterator`).

See [PIE workflow](unreal/pie-workflow.md#compile-error-pre-flight).

## "value rejected by schema" when setting an asset on a Blueprint pin

**Symptom.** `bp_node_pin_set_default value="/Game/.../MyAsset"` returns
`set_pin_default_failed`.

**Cause.** `UEdGraphPin::DefaultObject` (a UObject*) is the right slot
for object/class pins; the legacy tool wrote to the FString slot.

**Fix (Companion v1.3.4+).** Use `bp_node_pin_set_object`, OR rely on
the smart-routing in `bp_node_pin_set_default` (paths starting with
`/Game/` / `/Script/` auto-route now).

See [blueprint authoring](unreal/blueprint-authoring.md#object-references-on-pins).

## `bp_variable_add` created an IntProperty

**Symptom.** Adding a variable with `type="Object"` or
`type="/Script/Engine.AnimMontage"` produced an `IntProperty`.

**Cause.** The legacy tool only handles primitives.

**Fix (Companion v1.3.4+).** Use `bp_variable_add_typed`:

```
bp_variable_add_typed
  pin_category="object"
  sub_category_object="/Script/Engine.AnimMontage"
  default_object="/Game/.../MyMontage"
```

See [blueprint authoring](unreal/blueprint-authoring.md#object--class--struct--enum--containers--bp_variable_add_typed).

## `run_python` returns 100KB+ of noise

**Symptom.** A small script (e.g. `dir(unreal)`) returns a 150KB JSON
envelope; the agent's main signal is buried.

**Cause.** UE Python Remote Execution emits every log relay as an
Output event; broad APIs like `dir(unreal)` trigger hundreds of
deprecation warnings.

**Fix.** Already-applied — the unreal plugin handler prunes:

- Drops Info-level Output entries when `parsed_json` is present.
- Caps each entry at 4KB.
- Caps stderr at 4KB.
- Blanks the `Command` echo (was ~150KB).

If you still see bloat: confirm `output_dropped` field in the response
— it tells you how many entries were trimmed.

See [unreal plugin: output pruning](plugins/unreal.md#output-pruning).

## `files.read` returned the whole file even though I passed offset/limit

**Symptom.** AI calls `files_read path=... offset=280 limit=35`, gets
the full file back.

**Cause.** Older sb-files binary didn't support those parameters
(silently dropped). The legacy schema only had `max_bytes`.

**Fix.** Upgrade sb-files (build from master, restart Claude Code).
After the upgrade, `read` accepts `offset` / `limit` matching Claude
Code's built-in Read semantics. Response includes
`slice_mode: true` + `start_line` / `end_line` / `total_lines`.

See [files plugin](plugins/files.md#read).

## MCP transport closed

**Symptom.** Agent sees `MCP error -32603: transport error: transport
closed` on a tool call.

**Causes.**

- Daemon crashed.
- Stdio pipe broken (rare).
- Claude Code spawned multiple daemons; one died.

**Fix.**

1. Retry the tool — MCP auto-reconnects on transient close.
2. If persistent: `tasklist` and check for orphan `sb.exe` /
   `unreal.exe` processes. Kill all of them and restart Claude Code:

```
taskkill /F /IM sb.exe /T
taskkill /F /IM unreal.exe /T
# then restart Claude Code so it spawns a fresh daemon.
```

## PATH change didn't propagate

**Symptom.** After `sb install`, other apps still don't see `sb` on
PATH.

**Cause.** `reg.exe` doesn't broadcast `WM_SETTINGCHANGE`. The
installer uses PowerShell's `[Environment]::SetEnvironmentVariable`
which does — confirm the installer ran with `--method claude-cli` (the
default).

**Fix.**

1. Re-run `sb install --method claude-cli` (idempotent).
2. Log out / in if the broadcast was missed.
3. As a last resort: add `<repo>/bin` to PATH manually in the GUI.

See [installation](installation.md).

## Status bar widget doesn't appear after `companion_rebuild`

**Symptom.** Companion is updated to v1.3.1+ on disk but the LevelEditor
status bar doesn't show `● SB v1.3.x`.

**Cause.** The companion DLL is loaded at editor startup; rebuilding it
while the editor is running doesn't reload it.

**Fix.** Restart the editor. `editor_restart` does this.

## Cross-references

- [overview](README.md)
- [architecture](architecture.md)
- [installation](installation.md)
- [unreal deep dive](unreal/index.md)
- [changelog](changelog.md) — historical context.
