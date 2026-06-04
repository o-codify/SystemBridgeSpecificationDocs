---
id: path-based-tools-fail-when-editorassetlibrary-goes-blind-use-registry-load-asset-fallback
title: Path-based tools fail when EditorAssetLibrary goes blind (use registry/load_asset fallback)
status: request
version: 26.604.2113
tags: []
---

# Path-based tools fail when EditorAssetLibrary goes blind

**Status: request.** Observed on `ALS_UltimateWarfare` (UE 5.7.4, Companion 1.11.1), 2026-06.

## Symptom

In an otherwise healthy editor session, `EditorAssetLibrary` path lookups
went **blind** for *every* asset, including ones that were clearly loaded
and valid:

```
EditorAssetLibrary.does_asset_exist('/Game/Game/Blueprints/Weapon/BP_Weapon_Base')      → False
EditorAssetLibrary.does_asset_exist('/ALSReplicated/.../ALS_WarfarePlayer_Base')          → False  (a plugin BP)
EditorAssetLibrary.does_asset_exist('/Game/Game/Blueprints/Weapon/BP_M416')               → False
EditorAssetLibrary.load_asset('/Game/Game/Blueprints/Weapon/BP_Weapon_Base')              → None
EditorAssetLibrary.list_assets('/Game/Game/Blueprints/Weapon', recursive=False)           → []   (empty!)
```

…yet the same assets were fully present and loadable by other means in the
**same** `unreal_run_python` call:

```
AssetRegistry.get_assets(ARFilter(package_paths=['/Game/Game/Blueprints/Weapon']))
  → returns BP_Weapon_Base, BP_M416, BP_M9  (package_name + object_path correct)
unreal.load_asset('/Game/Game/Blueprints/Weapon/BP_Weapon_Base')        → valid Blueprint object
unreal.load_object(None, '/Game/.../BP_Weapon_Base.BP_Weapon_Base')      → valid object
```

`AssetRegistry.scan_paths_synchronous(force_rescan=True)` did NOT fix it;
`is_loading_assets()` was `False`. Root cause not diagnosed (looks like an
EAL/ContentBrowser data-source view that diverged from the in-memory
registry — possibly a UE 5.7 `does_asset_exist` regression around object-
path vs package-path, since `.Asset`-suffixed paths also returned False).

## Impact

Every **path-based companion MCP tool** failed because they resolve the
asset through the blind EAL path. E.g.:

```
unreal_bp_variable_add_typed bp_path="/Game/Game/Blueprints/Weapon/BP_Weapon_Base" ...
  → {"error":"blueprint_not_found","success":false}
```

This silently broke `unreal_bp_*`, `unreal_anim_*` (the path-taking ones),
etc. for the whole session — a confusing failure, since the asset
obviously exists.

> Note: `unreal_bp_variable_remove` later worked on a `/ALSReplicated/...`
> plugin path in the same session, so the blindness was intermittent /
> per-path, not total — which makes it harder to diagnose and more
> important for the tools to be defensive.

## Workaround used

Bypass the path-based tools entirely: `unreal.load_asset(path)` (the
global, which kept working) to get the object, then drive everything via
`unreal.SystemBridgeBindings.*` on the loaded object inside
`unreal_run_python` — `get_blueprint_graph_by_name`, `create_k2_node`,
`set_k2_node_call_function_target`, `link_k2_nodes_by_pin_names`,
`set_anim_node_inner_property`, etc. all take the live object/graph, not a
path, so they were unaffected. Whole-graph rebuilds were authored this way.

## Request

Companion path-based tools should not rely solely on
`EditorAssetLibrary.load_asset` / `does_asset_exist` for resolution. When
that returns nothing, fall back to:

1. `AssetRegistry.get_asset_by_object_path` / `get_assets(ARFilter)`, then
   `AssetData.get_asset()`; and/or
2. global `LoadObject` / `unreal::LoadObject` on the package path.

A `blueprint_not_found` that is actually "EAL path lookup is blind but the
asset is loadable" is a footgun; the fallback would make the tools robust
to this state. At minimum, the error payload could note "asset resolvable
via registry but not via EAL — retry via run_python + SystemBridgeBindings"
to point the caller at the workaround.
