---
id: crash-recovery
title: Crash Recovery
status: stable
version: 26.601.1605
tags: [ unreal, crash, watchdog ]
---

# Crash Recovery

The Unreal plugin runs a persistent crash watcher for its entire lifetime
— a background goroutine that polls every 5s and reacts to editor death
or new crash signals.

See: [unreal deep dive](index.md), [PIE workflow](pie-workflow.md).

## Why a persistent watcher

`pie_run_and_watch` detects crashes only inside its `wait_seconds`
window. PIE that crashes 5 minutes later went unnoticed — the agent saw
no signal until a follow-up tool call failed.

The watcher closes that gap. It runs from sb-unreal startup to shutdown
and catches crashes at any time.

## Detection signals

Each tick (5s), the watcher checks three independent signals:

1. **New crash dir** under `<Project>/Saved/Crashes/` — UE's canonical
   crash artifact path.
2. **New fatal marker** in `<Project>/Saved/Logs/<Project>.log` since
   the previous tick — `Critical error:`, `Fatal error:`,
   `Assertion failed:`.
3. **Editor process died** — `UnrealEditor.exe` was alive on the previous
   tick, not alive now.

Any one fires `unreal_editor_crashed` event + caches `last_crash` on
plugin state.

## On detection

The watcher does three things:

1. **Kill CrashReportClient dialogs.** Always — they block subsequent
   compiles / PIE if left up.
2. **Decide whether to auto-recover.** See the suppression layers below.
3. **Cache `last_crash` + emit event** so `editor_status.last_crash` and
   the next `discover()` surface the incident.

## Auto-recovery

When the editor process is gone AND auto-recovery is armed:

1. Resolve the engine root for the project's `EngineAssociation`.
2. Launch `UnrealEditor.exe` with the `.uproject` (`launchEditorDetached`).
3. Poll `editor_status` every 3s up to `crashRecoveryWaitTimeout` (240s).
4. On alive: stamp `lastRecoveryAt`, set `recovered: true` on `last_crash`,
   record `recovery_elapsed_ms`.

## Suppression layers

The watcher has three layered protections against fighting the user /
agent's intentional kills (e.g. UBT rebuild cycle):

### 1. Cooldown — `recoveryCooldown = 5 * time.Minute`

After a successful recovery, the watcher won't re-recover within 5
minutes. Catches:

- Agent killing the just-recovered editor to do something different.
- Recovered-but-broken editor (stale DLL, missing class registration)
  that crashes a second time — re-relaunching would just loop.

### 2. Build-tool detection

If any of these processes are running when the editor dies, presumed
intentional shutdown:

- `UnrealBuildTool.exe`
- `MSBuild.exe`
- `cl.exe`
- `link.exe`
- `LiveCodingConsole.exe`

These hold UE's DLLs locked anyway, so a relaunch would race the build.

### 3. Explicit AI override — `editor_set_auto_recover`

Use this BEFORE intentional kills:

```
unreal_editor_set_auto_recover(enabled=false, ttl_seconds=600)
# ... agent kills editor + runs UBT directly ...
unreal_editor_set_auto_recover(enabled=true)
# or just wait for TTL to expire — auto-restores
```

TTL is recommended (max 3600) so you don't leave the watcher disabled
forever by accident.

### 4. editor_restart self-suppression

`unreal_editor_restart` calls `SetAutoRecover(false, ttl=…)` automatically
for the duration of its kill-then-start cycle. No manual suppression
needed for the canonical restart path.

## Surface in `editor_status`

```jsonc
"auto_recover": {
  "enabled": true,
  "suppress_until": "2026-05-04T10:30:00Z",      // optional, when TTL'd
  "last_recovery_at": "2026-05-04T10:23:00Z",
  "cooldown_active": true                         // within 5 min of last recovery
},
"last_crash": {
  "at": "2026-05-04T10:22:30Z",
  "summary": "Critical error: Assertion failed: ...",
  "crash_dir": "C:/.../Saved/Crashes/UECC-Windows-...",
  "editor_was_alive_at_detect": true,
  "recovered": true,
  "recovery_error": "",                           // OR "skipped:<reason>"
  "dialogs_killed": 1,
  "recovery_elapsed_ms": 31000
}
```

When recovery was deliberately skipped, `recovery_error` reads
`skipped:auto_recover_disabled`, `skipped:cooldown_active`, or
`skipped:build_tool_running:UnrealBuildTool.exe` — concrete reason so the
agent can act.

## Event surface

`unreal_editor_crashed` event in `discover()`:

```jsonc
{
  "kind": "unreal_editor_crashed",
  "crash_dir": "...",
  "summary": "...",
  "editor_was_alive": true,
  "recovered": true,
  "dialogs_killed": 1,
  "recovery_error": "",
  "detected_at": "2026-05-04T10:22:30Z"
}
```

Severity `error` — the agent treats this as a priority signal.

## Patterns

- **Default operation.** Do nothing — the watcher handles spontaneous
  crashes silently and the agent sees `last_crash` on its next
  `editor_status`.

- **Intentional editor kill for clean build.** Wrap with
  `editor_set_auto_recover(false, ttl_seconds=600)` →
  do work → `editor_set_auto_recover(true)`. Or simpler:
  just call `unreal_editor_restart` which self-suppresses.

- **Cooldown firing unwantedly.** Wait 5 min, then retry. The cooldown
  prevents fighting your intent — if you really need to immediately
  re-launch, explicitly call `editor_set_auto_recover(true)` to clear
  the disable, then `editor_restart`.

- **Recovered editor is broken.** Check `editor_status` after each
  recovery: `companion_version`, `project_matches_cwd`. If the
  recovered editor loaded a stale DLL, `companion_rebuild` then
  `editor_restart`.

## Cross-references

- [editor_status](index.md#editor_status) — surfaces last_crash + auto_recover.
- [PIE workflow](pie-workflow.md) — `pie_run_and_watch` reads same signals
  in its watch window.
- [troubleshooting](../troubleshooting.md#editor-keeps-relaunching).
