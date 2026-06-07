---
id: request-asset-version-metadata-diagnostics-asset-version-info-level-as-data
title: "Request: asset version + metadata diagnostics (`asset_version_info`, level-as-data)"
status: request
version: 26.607.1704
tags: [ unreal, diagnostic, metadata, level, request ]
---

# Request: asset version + metadata diagnostics

**Origin.** UAssetAPI exposes `EngineVersion`, `ObjectVersion`, custom
versions, `FMetaData`, and `LevelExport` (actors-as-data) at byte level.
SystemBridge currently surfaces none of these — `asset_describe` gives
top-level UPROPERTYs only.

Three small diagnostic tools, low-effort, high-value.

## 1. `asset_version_info(asset_path)`

Returns the saved engine version + custom versions on a `.uasset`.

```jsonc
{
  "asset_path": "/Game/.../BP_OldThing",
  "saved_engine_version": "5.5.4",
  "current_engine_version": "5.7.0",
  "package_file_version": 1018,
  "custom_versions": [
    { "guid": "...", "name": "BlueprintsObjectVersion", "version": 47 },
    { "guid": "...", "name": "AnimationObjectVersion",  "version": 12 },
    …
  ],
  "warnings": [
    "saved with 5.5; some 5.7 reflection changes may not have migrated"
  ]
}
```

**Why.** "Why won't this asset load?" is a recurring AI debugging
question. The answer is often "saved with an older engine + a custom
version mismatch". Currently invisible. UAssetAPI exposes this via
`EngineVersion.cs` + `ObjectVersion.cs`.

**Route.** Companion C++: `UPackage::GetSavedHash` + `FPackageFileSummary`
provides engine version. Custom versions live in
`FCustomVersionContainer`. All read-only, very cheap.

## 2. `asset_metadata(asset_path)`

Returns the asset's `FMetaData` block — per-object string key-value
metadata that authors and editor tooling use for hints / categorization.

```jsonc
{
  "asset_path": "/Game/.../BP_Hero",
  "metadata": {
    "BP_Hero": {
      "Category": "Characters",
      "EditCondition": "bUseRig",
      "Tooltip": "The hero character base"
    },
    "BP_Hero.Mesh": {
      "OverrideNativeName": "BP_Hero_Mesh"
    }
  }
}
```

**Why.** Author-time hints, tooltips, edit conditions, category
groupings. Currently invisible. Useful for refactor tooling
("which BPs are tagged Deprecated?") and for AI-driven authoring
that wants to respect existing categorization.

**Route.** `UPackage::GetMetaData()` returns a `UMetaData*`; iterating
its `ObjectMetaDataMap` gives `{UObject*, TMap<FName,FString>}`. Trivial
walk.

## 3. `level_actors_offline(level_path)`

Reads actor placements from a `.umap` file without loading the level.
UAssetAPI's `LevelExport` exposes this as data.

```jsonc
{
  "level_path": "/Game/Maps/Demo",
  "actor_count": 247,
  "actors": [
    {
      "name": "BP_Hero_C_0",
      "class": "/Game/.../BP_Hero.BP_Hero_C",
      "location": [0, 0, 90],
      "rotation": [0, 0, 0],
      "is_component_owner_template": false
    },
    …
  ]
}
```

**Why.** Refactor planning: "every level where `BP_OldPickup` is
placed". Currently we'd `editor_load_level` each level → `actor_list`
→ unload — minutes per level. Offline read is milliseconds.

**Route.** Pairs naturally with the **bulk offline scanner** request.
Either bundled with that or shipped first as a single tool.

## Acceptance

- `asset_version_info` returns mismatched-engine warning for an asset
  saved in 5.5 and loaded into 5.7.
- `asset_metadata` returns the same keys the editor's Details panel
  shows in the "Edit Condition" / "Tooltip" overrides.
- `level_actors_offline` returns the same actor count as
  `editor_load_level` + `level_actor_list` for the same map, **without
  loading the level**.

## Why bundle the three

All three are read-only file-format introspection. Sharing the
parse-once-on-disk path makes sense. Combined effort is much smaller
than the sum of three separate work items.
