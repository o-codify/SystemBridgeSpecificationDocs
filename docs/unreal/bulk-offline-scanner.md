---
id: bulk-offline-asset-scanner
title: Bulk Offline Asset Scanner
status: stable
version: 26.607.2208
tags: [ unreal, performance, offline, audit, refactor ]
---

# Bulk Offline Asset Scanner

Three sb-unreal tools that walk `.uasset` headers directly on disk —
no editor required, no companion required. Built for project-wide
audits that would otherwise cost minutes through editor RPC.

## Performance baseline

| Workload | Editor RPC | Offline scanner |
|---|---|---|
| One asset (hot cache) | ~30-50ms | ~5-30ms |
| One asset (cold) | 100-300ms | ~5-30ms |
| 1000-asset audit | 30-150 seconds | **1-3 seconds** |

The offline scanner walks the filesystem with `runtime.NumCPU()/2`
parallel parsers and reads only the package header (Names + Imports +
Exports tables). It skips `Saved/`, `Intermediate/`, `Build/`.

## When to reach for this

- "Who breaks if I rename `BP_OldWeapon`?" → `assets_find_references`
- "Show me every DataTable / AnimBlueprint / Material in the project." → `assets_find_by_class`
- "What's this project's asset shape?" → `assets_scan_offline`
- "Diagnose project structure before a refactor." → `assets_scan_offline`

## Tool surface

### `assets_find_references(target_path, root="/Game")`

Returns every asset whose import table references `target_path`. The
`.AssetName` and `_C` suffixes are normalized away so callers can pass
the natural path.

```python
unreal_assets_find_references(
    target_path="/Game/Weapons/SK_Pistol",
)
# → {
#   target: "/Game/Weapons/SK_Pistol",
#   root: "/Game",
#   file_count: 1247,
#   match_count: 23,
#   duration: "1.4s",
#   matches: ["/Game/Characters/BP_Hero", "/Game/Weapons/BP_Pistol", ...]
# }
```

### `assets_find_by_class(classes, root="/Game")`

Returns every asset whose top-level export class matches any of the
candidates. Accepts short class names (`Blueprint`, `DataTable`,
`AnimMontage`, `SkeletalMesh`) or full paths (`/Script/Engine.DataTable`).

```python
unreal_assets_find_by_class(
    classes=["DataTable", "/Script/Engine.AnimMontage"],
)
# → {
#   candidates: [...],
#   file_count: 1247,
#   match_count: 89,
#   matches: [{package: "/Game/Data/DT_Weapons", class: "DataTable"}, ...]
# }
```

### `assets_outdated(root="/Game", current_ue5_version=1009)`

Returns every asset whose saved engine version is older than the
threshold. Default threshold: UE 5.7 (file_version_ue5 = 1009).

```python
unreal_assets_outdated()
# → {
#   root: "/Game",
#   file_count: 1247,
#   outdated_count: 18,
#   by_engine_family: { "ue5": 1240, "ue4": 7 },
#   outdated: [
#     { package: "/Game/Legacy/BP_OldThing", engine_family: "ue4",
#       file_version_ue4: 522, file_version_ue5: 0 },
#     ...
#   ],
# }
```

Useful after a UE upgrade to find what still needs re-saving.

### `assets_scan_offline(root="/Game")`

Project-wide audit. Class histogram + engine-family histogram + top 20
most-referenced packages.

```python
unreal_assets_scan_offline()
# → {
#   root: "/Game",
#   file_count: 1247,
#   duration: "2.1s",
#   by_class: { "Blueprint": 312, "DataTable": 18, "SkeletalMesh": 47, ... },
#   by_engine_family: { "ue5": 1240, "ue4": 7 },
#   top_referenced: [
#     { package: "/Engine/EngineMaterials/DefaultMaterial", refs: 412 },
#     { package: "/Game/Characters/Animations/AnimBP_Base", refs: 87 },
#     ...
#   ],
# }
```

The `top_referenced` list answers "which assets are the most depended-on"
— useful for identifying load-bearing assets that need careful
refactoring.

## Output normalization

`target_path` accepts any of these forms — they all normalize to the
same:

| Input | Normalized |
|---|---|
| `/Game/Weapons/SK_Pistol` | `/Game/Weapons/SK_Pistol` |
| `/Game/Weapons/SK_Pistol.SK_Pistol` | `/Game/Weapons/SK_Pistol` |
| `/Game/Weapons/SK_Pistol.SK_Pistol_C` | `/Game/Weapons/SK_Pistol` |
| `/Script/Engine.Actor` | `/Script/Engine.Actor` (kept — leaf doesn't match dot suffix) |

## Limitations

- **Read-only.** No write/mutation surface. Editor-bypass writes are
  unsafe (derived data, generated classes, recompile-on-save).
- **Uncooked editor assets only.** Cooked / `.usmap`-required paths are
  not yet supported.
- **Top-level export class only.** Multi-export files (rare in `/Game`)
  report the class of the FIRST export. For full per-export breakdown,
  use the editor's AssetRegistry.
- **`root` must be `/Game`-prefixed in v1.** Plugin Content (`/PluginName`)
  not yet supported — file a request if needed.
- **Imports table coverage depends on UE 4.27 + UE 5.1 fields.**
  Implemented for 5.7; older versions may miss the `PackageName` field
  and fall back to `ClassPackage` for reference matching.

## Cross-references

- [plugin: unreal](../plugins/unreal.md#bulk-offline-scanner-no-editor) — tool table.
- [asset management](asset-management.md#read) — robust path resolution
  (the runtime-side counterpart for blind EAL).
- [companion plugin](companion.md) — companion-side `LoadAssetWithFallback`
  for runtime fallback.
