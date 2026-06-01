---
id: workflow-headless-rebuild
title: "Workflow: Headless Rebuild"
status: review
version: 26.601.1529
tags:
  - workflow
  - build
  - ubt
---

# Workflow: Headless Rebuild

Full UnrealBuildTool pass — for changes Live Coding can't patch
(new UCLASS, new module, ABI shifts).

See: [C++ iteration](cpp-iteration.md), [Live Coding](../unreal/live-coding.md),
[crash recovery](../unreal/crash-recovery.md).

## Why this workflow exists

`project_build` invokes UnrealBuildTool which produces / links
`UnrealEditor-<Project>.dll`. That DLL is **locked while the editor
holds it**. Two consequences:

1. The editor must be closed (or `editor_restart` mid-build is fine).
2. The [crash watcher](../unreal/crash-recovery.md) must be suppressed —
   otherwise it relaunches the editor mid-build and breaks the link.

## Steps

```
# 0. Save anything dirty.
unreal_editor_save_all_dirty

# 1. Suppress the crash watcher for the build duration.
unreal_editor_set_auto_recover enabled=false ttl_seconds=600

# 2. Close the editor. Three options:
#    a) unreal_editor_restart force=true       (auto-suppresses watcher; restarts after)
#    b) unreal_editor_restart force=true wait_for_completion=false   (just kills it)
#    c) The agent does taskkill via the process plugin:
#       process_kill image="UnrealEditor.exe"

# 3. Run UBT.
unreal_project_build target="MyGameEditor" configuration="Development" platform="Win64"
  → success: true | false
    errors:   [{file, line, code, message}, ...]
    warnings: [{file, line, code, message}, ...]
    duration_ms: 90000

# 4. Re-arm the watcher (if step 1 used a long TTL you can just wait).
unreal_editor_set_auto_recover enabled=true

# 5. Re-open the editor.
#    a) Manual launch by the user.
#    b) unreal_editor_restart (if you used 2a, this part already happened).
```

## Triage on `project_build` failure

```jsonc
{
  "success": false,
  "errors": [
    {"file": "MyGame/Private/Foo.cpp", "line": 42, "code": "C2065",
     "message": "'BarType': undeclared identifier"},
    ...
  ],
  "warnings": [...]
}
```

Edit the source files via `files.edit`, then re-run `project_build`.
For multi-file fixes: `files.read_many` to confirm context before
editing all.

`project_build` returns within 5-30s on a tight incremental edit; full
rebuilds take 10-30min depending on the project. The default timeout is
10 minutes; pass `timeout_seconds=1800` for full rebuilds.

## Comparison with Live Coding

| Aspect | Live Coding | project_build |
|---|---|---|
| Editor must be running? | Yes | No (preferable: closed) |
| Patches affect running editor? | Yes (hot-reload) | No (next launch) |
| Handles new UCLASS? | Sometimes-no | Yes |
| Handles new module? | No | Yes |
| Cross-module ABI changes? | No | Yes |
| Speed on a 1-line change | ~7s | ~30-60s |
| Action-limit popup possible? | Yes (every 15 patches) | No |

Rule of thumb:

- Method body / inline tweak → Live Coding.
- Anything header-level → `project_build`.
- "Live coding succeeded but the change isn't visible" → ABI / reflection
  drift; full rebuild via `project_build`.

## Tests

```
unreal_project_run_tests filter="Project.Functional" timeout_seconds=600
```

Spawns `UnrealEditor.exe -nullrhi` (headless) + quit-on-completion.
Parses `LogAutomationController` into per-test `{name, status}` + summary.
Works without the editor running interactively.

## Cross-references

- [unreal plugin tool catalog](../plugins/unreal.md#build--test-out-of-process)
- [crash recovery](../unreal/crash-recovery.md) — for the suppression
  layering this workflow depends on.
- [C++ iteration](cpp-iteration.md) — the faster sibling workflow.
