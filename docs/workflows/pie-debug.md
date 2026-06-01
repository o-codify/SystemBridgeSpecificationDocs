---
id: workflow-pie-debugging
title: "Workflow: PIE Debugging"
status: draft
version: 26.601.1506
tags:
  - workflow
  - pie
---

# Workflow: PIE Debugging

Start a Play-In-Editor session, watch it for issues, triage and fix.

See: [PIE workflow reference](../unreal/pie-workflow.md),
[crash recovery](../unreal/crash-recovery.md),
[workflows index](index.md).

## Steps

```
# 0. Snapshot state.
unreal_editor_status
  → alive: true, project_matches_cwd: true,
    last_crash: null (or recovered:true if recent), auto_recover.enabled: true.

# 1. Save dirty.
unreal_editor_save_all_dirty

# 2. Optional but cheap: surface any project health issues that aren't in <Project>.log.
unreal_editor_message_log_categories
  → if LoadErrors or MapCheck are populated, drill in BEFORE PIE.
unreal_editor_message_log category="LoadErrors" min_severity=2

# 3. Run with watch.
unreal_pie_run_and_watch wait_seconds=15
```

## Decision tree by status

### `pie_running`

PIE is alive and behaving. Issue console commands to probe state:

```
unreal_pie_console_command command="stat fps"
unreal_pie_console_command command="showdebug abilitysystem"
unreal_pie_console_command command="stat ai"
```

When done: `unreal_pie_stop`.

### `pie_exited_clean`

Editor's still alive, PIE world disappeared. Either you / the agent
stopped it, or game code called `UKismetSystemLibrary::QuitGame`. Check
`log_excerpt` for the trigger.

### `crashed`

PIE-level fatal — usually an assertion fired in C++ or BP. Editor likely
still alive. Returned payload:

```jsonc
{
  "status": "crashed",
  "crash_dir": "C:/.../Saved/Crashes/UECC-...",
  "crash_summary": "Assertion failed: Pawn != nullptr [...]",
  "log_excerpt": [
    "[2026.05.04-...:LogTemp] Some setup line",
    "[...] Critical error: Assertion failed: Pawn != nullptr ...",
    "[...] Stack: ...",
    ...
  ],
  "editor_alive": true
}
```

Fix path:

1. Read `crash_summary` — identifies the failing assertion or check.
2. The stack frames in `log_excerpt` give file:line — open with
   `files_read offset=<line-10> limit=20` for context.
3. Edit the source.
4. Hot-reload via [C++ iteration](cpp-iteration.md) (or recompile BP via
   `bp_compile_and_save`).
5. Retry `pie_run_and_watch`.

### `editor_crashed`

Editor process died. Watcher kicks in per
[crash recovery](../unreal/crash-recovery.md) rules. Next
`unreal_editor_status` shows whether `last_crash.recovered: true`.

If recovered: the editor came back, but maybe with a stale DLL. Confirm
your change is in effect (e.g. re-call a tool whose behavior depends on
the change).

If NOT recovered (suppressed by cooldown / build tool detection / disabled):
- `recovery_error: "skipped:build_tool_running:UnrealBuildTool.exe"` — UBT
  was running; wait for it, then `editor_restart`.
- `recovery_error: "skipped:cooldown_active"` — you just recovered; wait
  5 min OR `editor_set_auto_recover(true)` + `editor_restart`.
- `recovery_error: "skipped:auto_recover_disabled"` — you explicitly
  disabled; `editor_set_auto_recover(true)` + `editor_restart`.

### Pre-flight: `blueprint_compile_errors`

`pie_run_and_watch` (Companion v1.3.3+) won't dispatch into UE if any BP
has compile errors:

```jsonc
{
  "success": false,
  "error": "blueprint_compile_errors",
  "failing_blueprints": [
    {"path": "/Game/MP/BP_PlayerCharacter", "status": "BS_Error",
     "source": "companion_live"}
  ]
}
```

Fix path:

1. `bp_get_info` on the failing BP to see what's broken.
2. Open the editor's Blueprint editor manually (or use
   [blueprint authoring](../unreal/blueprint-authoring.md) tools).
3. `bp_compile_and_save` once fixed.
4. Retry `pie_run_and_watch`.

Or, if the BP itself isn't the agent's concern, can use
`project_build` to flush all BP compile errors via a full pass.

## End-to-end example

Goal: verify a HardLanding montage plays when Character lands.

```
# 0. Sanity.
unreal_editor_status
  → alive:true, project_matches_cwd:true, last_crash:null

# 1. Save.
unreal_editor_save_all_dirty

# 2. Run.
unreal_pie_run_and_watch wait_seconds=15
  → status: "pie_running"

# 3. Trigger the scenario (jump + fall) — via PIE console:
unreal_pie_console_command command="ke * SimulateFall"
# Or for a non-event scenario, the agent watches and we let the user play.

# 4. Stop.
unreal_pie_stop
```

If the montage didn't play:

```
# Inspect runtime state via run_python (escape hatch).
unreal_run_python script="""
import unreal
pc = unreal.GameplayStatics.get_player_controller(unreal.EditorLevelLibrary.get_pie_worlds()[0], 0)
char = pc.get_controlled_pawn()
am = char.get_mesh().anim_instance.is_any_montage_playing()
import json
print('<<<SB_JSON>>>'); print(json.dumps({'is_playing': am})); print('<<<END_SB_JSON>>>')
"""
```

## Cross-references

- [PIE workflow reference](../unreal/pie-workflow.md)
- [crash recovery](../unreal/crash-recovery.md)
- [C++ iteration](cpp-iteration.md) — for C++ fixes after a crash.
- [BP feature add](bp-feature-add.md) — for BP fixes.
