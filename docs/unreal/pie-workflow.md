---
id: pie-workflow
title: PIE Workflow
status: review
version: 26.601.1527
tags:
  - unreal
  - pie
  - playtest
---

# PIE Workflow

Play-In-Editor is the canonical way to verify a change without packaging.
SystemBridge's PIE tools give the agent a closed-loop experience: start,
watch, detect crashes, recover, retry.

See: [unreal deep dive](index.md), [crash recovery](crash-recovery.md),
[blueprint authoring](blueprint-authoring.md).

## Tools

| Tool | Purpose |
|---|---|
| `pie_start` | Start PIE (or SIE). Pre-flight scans for BP compile errors before dispatching. |
| `pie_stop` | Stop active PIE. |
| `pie_possess` | F8 equivalent — flip Play ↔ Simulate (camera-only). |
| `pie_run_and_watch` | Start PIE, watch for N seconds, return structured status + log excerpt on failure. |
| `pie_console_command` | Execute a console command in the PIE world (`stat fps`, `showdebug …`). |

## `pie_start` — modes

```
unreal_pie_start mode="PIE"        # real PIE with Pawn possession (Companion v1.1+)
unreal_pie_start mode="Simulate"   # SIE; no possession; F8 to flip
```

WITHOUT Companion v1.1, both modes fall back to `editor_play_simulate`
(SIE). The response includes a `warning` + hint to install Companion.

## Compile-error pre-flight

**Companion v1.3.3+**. UE's PIE pre-flight checks live
`UBlueprint::Status` — if any loaded BP has `BS_Error`, UE shows a
modal "Are you sure?" dialog that blocks the play session until clicked.

Previously the dispatched `RequestPlaySession` returned to the agent with
`success: true, started: true` and `pie_active` flipped — but PIE never
actually ran. The dialog held the game thread modal.

The new pre-flight scans:

1. **First**: Companion's `GetBlueprintsWithCompileErrors()` — walks all
   loaded `UBlueprint` via `TObjectIterator`. This is the same source UE
   itself uses for the dialog → authoritative.
2. **Fallback** (no Companion): AssetRegistry tag `BlueprintStatus`.
   On-disk cached, may be stale for unsaved edits.

On hit, `pie_start` returns:

```jsonc
{
  "success": false,
  "error": "blueprint_compile_errors",
  "failing_blueprints": [
    {"path": "/Game/MP/BP_AGLS_MP_PlayerCharacter", "status": "BS_Error",
     "source": "companion_live"}
  ],
  "hint": "Fix them first, or call bp_compile_and_save / project_build."
}
```

`source: "companion_live"` is the authoritative path; `"asset_registry"`
is the fallback. Capped at 20 entries.

## `pie_run_and_watch`

Single-call wrapper that gives the agent a closed loop:

```
unreal_pie_run_and_watch wait_seconds=10
```

Steps:

1. Snapshot pre-state: `Saved/Crashes/` dirs, `<Project>.log` size, the
   set of running `CrashReportClient*` PIDs (so stale dialogs from prior
   sessions don't false-positive).
2. Same compile-error pre-flight as `pie_start`.
3. Dispatch PIE.
4. Watch for `wait_seconds`. Every second, check:
   - `UnrealEditor.exe` still alive (no → `editor_crashed`).
   - New `CrashReportClient.exe` PID (yes → `crashed`).
   - New crash dir under `Saved/Crashes/` (yes → `crashed`).
   - Fatal markers in `<Project>.log` since the start (yes → `crashed`).
5. Settle 5s after detection — if the editor dies in the settle window,
   promote to `editor_crashed` and auto-recover via the watcher.
6. Return status + log excerpt + crash artifacts.

Status enum:

| Status | Meaning |
|---|---|
| `pie_running` | PIE is alive at end of watch window. Default success. |
| `pie_exited_clean` | PIE world disappeared but editor still alive (user/agent stopped it). |
| `crashed` | PIE-level fatal (assertion in game code). Editor likely survived. |
| `editor_crashed` | Editor process died. Watcher relaunches per [crash recovery](crash-recovery.md) rules. |

Returns:

```jsonc
{
  "started": true,
  "status": "crashed",
  "elapsed_ms": 5230,
  "wait_seconds": 10,
  "editor_alive": true,
  "crash_dialogs": 1,
  "crash_dir": "C:/.../Saved/Crashes/UECC-Windows-...",
  "log_excerpt": ["...last 50 fatal-section lines..."],
  "crash_summary": "Assertion failed: Foo != nullptr [/Game/.../BP_Foo.cpp:42]",
  ...
}
```

## `pie_possess` (F8 equivalent)

Toggle between Play (with Pawn possession) and Simulate (camera control,
no Pawn). Pre-condition: a play session must already be active. Without
Companion v1.1+, no-op.

```
# typical sequence on older companion:
unreal_pie_start mode="PIE"     # actually starts SIE on companion < 1.1
unreal_pie_possess              # flips to Play
```

With Companion v1.1+, `pie_start mode=PIE` skips the F8 — it starts PIE
directly.

## `pie_console_command`

Issue a console command into the active PIE world:

```
unreal_pie_console_command command="stat fps"
unreal_pie_console_command command="showdebug abilitysystem"
unreal_pie_console_command command="ke * MyEvent"
```

The command runs on the PIE GameViewport's world (not the editor world).

## End-to-end recipe

```
# 0. Sanity check.
editor_status            # alive? project_matches_cwd? auto_recover.enabled?

# 1. Save dirty assets before PIE (catches "you wouldn't have run with these").
editor_save_all_dirty

# 2. Run + watch.
pie_run_and_watch wait_seconds=15

# 3. Triage by status.
#    pie_running       → done; pie_stop when satisfied.
#    crashed           → read crash_summary + log_excerpt; bp_compile_and_save
#                        any BP referenced in the trace; retry.
#    editor_crashed    → editor_status.last_crash; if recovered:true, retry.
#                        if recovered:false → editor_restart + companion_rebuild.

# 4. Console probes while live.
pie_console_command command="stat unit"
pie_console_command command="showdebug enhancedinput"

# 5. Stop.
pie_stop
```

## Patterns

- **"Did my BP edit break anything?"** → `pie_run_and_watch` reads the
  compile-error pre-flight; if `pie_running` returns, you're past the
  static check. Crashes in the watch window get summarized in
  `log_excerpt`.

- **"Does the game spawn the player correctly?"** → `pie_start mode=PIE`
  (real PIE, possession). Check `editor_status.pie_active`. To peek
  state: `pie_console_command "stat ai"` etc.

- **"It runs but the player isn't possessed."** → Companion missing or
  v<1.1 → falls back to SIE. Either install/upgrade or
  `pie_possess` to F8 into Play.

## Cross-references

- [blueprint authoring](blueprint-authoring.md) — fixes that need to
  happen before PIE will run cleanly.
- [crash recovery](crash-recovery.md) — what happens between watches.
- [companion plugin](companion.md) — v1.1 unlocks real PIE; v1.3.3 PIE
  pre-flight.
