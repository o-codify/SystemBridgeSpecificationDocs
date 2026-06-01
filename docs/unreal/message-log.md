---
id: message-log
title: Message Log
status: draft
version: 26.601.1517
tags:
  - unreal
  - messagelog
  - diagnostics
---

# Message Log

UE's editor keeps a **categorized in-memory message list** that's
separate from the chronological `<Project>.log` file. It's the "Журнал
сообщений" / "Message Log" window in the editor. Examples of categories:

- `LoadErrors` — package load failures.
- `MapCheck` — World / level checker (lighting, BSP, NavMesh).
- `AssetCheck` — Asset Validator failures.
- `BlueprintLog` — BP compile warnings.
- `PIE` — Play-In-Editor errors.
- `SourceControl` — RCS issues.
- `PackagingResults` / `LightingResults` / `HLODResults`.

Without Companion these messages are invisible — `read_log` only sees
the chronological log file. Companion v1.2+ adds two tools:

- `unreal_editor_message_log_categories` — populated categories with
  counts.
- `unreal_editor_message_log` — entries from one category.

See: [unreal deep dive](index.md),
[companion plugin v1.2](companion.md).

## When to use this

Whenever the chronological log isn't enough:

- **Package load failures** at editor startup that appear in the
  `LoadErrors` tab and only there — these are exactly the warnings the
  agent missed when working on a project full of red BPs.
- **Map Check warnings** ("Foo has lighting needing rebuild", "missing
  collision", "NavMesh out of date") that only show in `MapCheck`.
- **Asset Validator** results — these are user-authored validators
  (`UEditorValidatorBase`) and their output only goes to `AssetCheck`.

## `editor_message_log_categories`

No args. Returns populated categories only (empty categories omitted),
sorted by total count desc:

```jsonc
{
  "success": true,
  "count": 3,
  "categories": [
    {"category": "LoadErrors", "info": 0, "warning": 0, "error": 6,  "critical": 0, "total": 6},
    {"category": "MapCheck",   "info": 5, "warning": 2, "error": 0,  "critical": 0, "total": 7},
    {"category": "BlueprintLog","info":0, "warning": 3, "error": 1,  "critical": 0, "total": 4}
  ]
}
```

Covered categories (probed; absent if never populated):

```
LoadErrors  MapCheck  AssetCheck  AssetTools  BlueprintLog
AnimBlueprintLog  BuildAndSubmitErrors  EditorErrors  HLODResults
LightingResults  PackagingResults  PIE  SlateStyleLog  SourceControl
AutomationTestingLog  UDNParser  VirtualizationResults  WorldComposition
```

## `editor_message_log`

```
unreal_editor_message_log category="LoadErrors" min_severity=2 max_messages=50
```

- `category` — FName key (case-insensitive).
- `min_severity` — 0=Info, 1=Warning, 2=Error, 3=CriticalError. Default 0.
- `max_messages` — 1..1000. Default 100. Most recent first.

Returns:

```jsonc
{
  "success": true,
  "category": "LoadErrors",
  "count": 6,
  "messages": [
    {
      "severity": "error",
      "text": "Cannot load package '/Game/GASP/.../PSD_Dense_Stand_Run_Pivots'. Invalid package end tag.",
      "object_path": "/Game/GASP/.../PSD_Dense_Stand_Run_Pivots"
    },
    {
      "severity": "error",
      "text": "Invalid summary for package '/Game/GASP/.../MetaHuman_Clothing_LODSettings'.",
      "object_path": "/Game/GASP/.../MetaHuman_Clothing_LODSettings"
    },
    ...
  ]
}
```

`object_path` is extracted from the first object / asset token in the
message — handy for follow-up `asset_referencers` / `asset_dependencies`
calls.

## Failure modes

```
{success: false, reason: "companion_unavailable", hint: "Run companion_install ..."}
```

The Message Log module is part of UnrealEd; in non-editor commandlet
runs `editor_message_log` returns an empty list (category isn't
registered) — that's expected, not an error.

## Patterns

- **First call after `editor_status`** in any audit flow:
  `editor_message_log_categories` shows which buckets have anything.
- **Find the offending asset.** When `LoadErrors` reports a path,
  follow up with `asset_referencers` to see who else depended on it —
  the broken asset may have already been removed manually.
- **Pre-PIE check.** Map Check warnings (no NavMesh) don't block PIE
  but cause runtime issues; surface them BEFORE `pie_run_and_watch`.

## Cross-references

- [companion plugin v1.2](companion.md) — where Message Log access was
  added.
- [unreal plugin tool catalog](../plugins/unreal.md#logs--diagnostics).
- [troubleshooting](../troubleshooting.md) — "Editor opens but assets
  are missing".
