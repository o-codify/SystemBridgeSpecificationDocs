---
id: workflow-c-iteration
title: "Workflow: C++ Iteration"
status: draft
version: 26.601.1517
tags:
  - workflow
  - cpp
  - live-coding
---

# Workflow: C++ Iteration

Edit a C++ source file in a UE module, hot-reload it with Live Coding,
verify in PIE.

This is the most common UE workflow and runs purely inside SystemBridge
tools.

See: [Live Coding reference](../unreal/live-coding.md),
[PIE workflow](../unreal/pie-workflow.md),
[workflows index](index.md).

## Preconditions

- Editor is running (Companion shows `SB v1.3.x` in the status bar).
- The change is patchable by Live Coding (method bodies / inline edits;
  NOT new UCLASS / module additions — those need `project_build`).
- The project has at least one C++ module (`*.Build.cs` under `Source/`).
  `live_coding_compile` pre-checks this; returns
  `blueprint_only_project` otherwise.

## Steps

```
# 1. Find the symbol / line to edit.
search_grep query="RegenDelaySeconds" dir="Source/MyGame"

# 2. Read the surrounding context.
files_read path="Source/MyGame/.../Foo.cpp" offset=280 limit=35

# 3. Make the change.
files_edit path="Source/MyGame/.../Foo.cpp"
    old_string="<exact slice>"
    new_string="<modified slice>"

# 4. Live Coding hot-reload (synchronous).
unreal_live_coding_compile
  → status: "succeeded"
    duration_ms: 7727
    success_line: "[...] LogLiveCoding: Display: Live coding succeeded"

# 5. Verify in PIE.
unreal_editor_save_all_dirty
unreal_pie_run_and_watch wait_seconds=15
  → status: "pie_running"     # change is live and behaves correctly
```

## Triage on `live_coding_compile` failure

```jsonc
{
  "status": "failed",
  "errors": [
    "Foo.cpp(284): error C2065: 'BarSeconds': undeclared identifier"
  ],
  "fail_line": "...Live coding failed...",
  "log_excerpt": [...]
}
```

- Parse `errors[]` → fix → re-run `live_coding_compile`.
- If multiple files: `read_many` to confirm context, then `edit` per file.
- If the error mentions a class that has no UFUNCTION/UPROPERTY — likely
  a missing include. The error line tells you which file.

## Triage on `timeout`

```jsonc
{
  "status": "timeout",
  "hint": "No 'Live coding succeeded/failed' marker within 120s. Possible
           causes: (1) action-limit popup is open in LiveCodingConsole.exe
           — pass skip_action_limit=true to reset, (2) compile genuinely
           takes >wait_seconds — raise the budget, (3) LCC.exe died —
           taskkill it and retry."
}
```

Common path: `skip_action_limit=true` and retry. Costs ~5s cold-start.

## When Live Coding can't patch

UE's Live Coding has known limits. Use `project_build` for:

- Adding a new UCLASS / extending existing UCLASS reflection.
- Adding a new module to the project.
- ABI changes in headers (member layout).

Workflow with `project_build`:

```
# 1. Save + close editor (project_build needs DLL writable).
unreal_editor_save_all_dirty
unreal_editor_set_auto_recover enabled=false ttl_seconds=600
unreal_editor_restart force=true     # this internally suppresses the watcher
                                     # OR — agent just taskkills the editor

# 2. UBT.
unreal_project_build
  → errors:   [{file, line, code, message}, ...]
    warnings: [...]

# 3. Re-open editor.
#    (after editor_restart this happens implicitly)

# 4. PIE.
unreal_pie_run_and_watch wait_seconds=15
```

## Common patterns

- **Tighten a constant.** Edit + `live_coding_compile`. ~7s feedback.

- **Adjust a method body.** Edit + `live_coding_compile`. Same.

- **Add a public function to a UCLASS.** Often Live Coding fails (UFUNCTION
  changes need a real link). Switch to `project_build`. Plan: edit →
  `project_build` → restart → `pie_run_and_watch`.

- **Refactor across modules.** Always `project_build`. Live Coding can
  patch one module at a time reliably, multi-module patches go wrong.

## Cross-references

- [Live Coding reference](../unreal/live-coding.md)
- [PIE workflow](../unreal/pie-workflow.md)
- [crash recovery](../unreal/crash-recovery.md) — for the suppression
  needed before agent-initiated editor kills.
- [files plugin](../plugins/files.md), [search plugin](../plugins/index.md#search).
