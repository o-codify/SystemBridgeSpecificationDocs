---
id: unreal-deep-dive
title: Unreal — Deep Dive
status: stable
version: 26.602.1331
tags: [ unreal, ue, index ]
---

# Unreal — Deep Dive

The Unreal plugin is the biggest surface in SystemBridge. This section
covers the patterns and sub-systems beyond a bare tool list.

For the tool catalog (every UE tool with one-line descriptions), see
[the unreal plugin reference](../plugins/unreal.md).

## Pages

- [Companion plugin](companion.md) — the C++ Editor-only sub-plugin, its
  version timeline, what each version unlocks.
- [Blueprint authoring](blueprint-authoring.md) — headless BP editing:
  graphs, nodes, pins, variables, function overrides, **object/asset
  references**.
- [PIE workflow](pie-workflow.md) — Play-In-Editor lifecycle, watch tools,
  compile-error pre-flight.
- [Live Coding](live-coding.md) — synchronous incremental C++ compile.
- [Crash recovery](crash-recovery.md) — persistent watcher, auto-recover
  semantics, suppression layers.
- [Message Log](message-log.md) — UE's in-memory categorized messages
  (LoadErrors, MapCheck, AssetCheck, …).
- [Asset management](asset-management.md) — `assets_query`,
  `asset_describe`, references / dependencies.
- [editor_status reference](#editor_status) (this page).

## When to use what

| Goal | Start with |
|---|---|
| "Is the editor responsive and on the right project?" | [`editor_status`](#editor_status) |
| "Did anything break while I was away?" | `editor_status.last_crash` + [crash recovery](crash-recovery.md) |
| "Compile C++ and iterate." | [live_coding_compile](live-coding.md) |
| "Edit a Blueprint without clicking." | [blueprint authoring](blueprint-authoring.md) |
| "Launch PIE and watch it." | [`pie_run_and_watch`](pie-workflow.md) |
| "Find package-load errors / MapCheck warnings." | [message log](message-log.md) |
| "What does this DataAsset have on it?" | [asset_describe](asset-management.md#asset_describe) |
| "Rebuild the project headless (UBT)." | `project_build` |
| "Restart the editor cleanly." | `editor_restart` (auto-suppresses [watcher](crash-recovery.md)) |

## Two execution paths recap

1. **In-editor (Python Remote Execution).** Most tools — uses
   `unreal.SystemBridgeBindings.*` and `unreal.EditorAssetLibrary` /
   `unreal.LevelEditorSubsystem` / friends.
2. **Out-of-process.** `project_build` calls UnrealBuildTool directly,
   `project_run_tests` spawns `UnrealEditor.exe -nullrhi`, `editor_restart`
   does `taskkill` + relaunch. These work even when the editor is dead.

## Companion gates capability

Every tool that needs C++ reflection power (TObjectIterator,
FBlueprintEditorUtils, FMessageLog, K2Node mutators, …) requires the
[Companion plugin](companion.md). Helpers detect Companion at runtime
(`hasattr(unreal, "SystemBridgeBindings")`) and either:

- Use Companion when present (fast / authoritative).
- Fall back to a Python approximation when missing.
- OR return `{success: false, reason: "companion_unavailable"}` with a hint
  to run `companion_install`.

`unreal_editor_status` reports `companion_loaded` + `companion_version`;
mismatch with `expectedCompanionVersion` triggers `companion_rebuild`.

## editor_status

This is the entry point for any multi-step UE flow. Always cheap (~300ms
round-trip) and packed with diagnostics. Cheaper than failing 10 tool calls
deep with a confusing "no editors discovered" error.

Returns:

```jsonc
{
  "alive": true,
  "project_name": "AGLS",
  "project_directory": "C:/Users/User/Documents/Unreal Projects/AGLS/",
  "current_level": "/Game/Maps/Demo.Demo",
  "pie_active": false,
  "engine_version": "5.7.4-...",

  // companion drift signal
  "companion_loaded": true,
  "companion_version": "1.7.0",
  "expected_companion_version": "1.7.0",
  "companion_version_mismatch": false,   // true → run companion_rebuild

  // cwd consistency
  "cwd_project_name": "AGLS",
  "project_matches_cwd": true,
  "round_trip_ms": 142,

  // crash watcher
  "last_crash": {
    "at": "...",
    "summary": "Critical error: ...",
    "crash_dir": "C:/Users/.../Saved/Crashes/UECC-...",
    "editor_was_alive_at_detect": true,
    "recovered": true,
    "dialogs_killed": 1,
    "recovery_elapsed_ms": 31000
  },

  // auto-recovery state (see crash-recovery.md)
  "auto_recover": {
    "enabled": true,
    "suppress_until": "...",
    "last_recovery_at": "...",
    "cooldown_active": true
  }
}
```

Use it BEFORE any multi-step UE workflow:

- `alive: false` → editor down; `editor_restart` or have the user launch it.
- `project_matches_cwd: false` → wrong editor focused; the agent should
  pass `project_dir` explicitly to subsequent tools.
- `companion_version_mismatch: true` → run `companion_rebuild`; otherwise
  some Companion-backed tools may misbehave silently.
- `last_crash.at` recent + `recovered: true` → editor came back from a
  crash; if the recovery loaded a stale DLL, build artifacts may be
  outdated (rebuild before iterating).

## Status-bar widget

Companion v1.3.1+ adds a pill to the LevelEditor status bar:

```
[● green] SB v1.7.0
```

Hover for tooltip listing the bindings that are armed. Useful confirmation
without round-tripping `companion_status`.

## Cross-references

- [companion plugin](companion.md)
- [PIE workflow](pie-workflow.md)
- [blueprint authoring](blueprint-authoring.md)
- [Live Coding](live-coding.md)
- [crash recovery](crash-recovery.md)
- [message log](message-log.md)
- [asset management](asset-management.md)
