---
id: request-bulk-offline-asset-scanner-project-wide-audits-in-seconds-not-minutes
title: "Request: bulk offline asset scanner (project-wide audits in seconds, not minutes)"
status: request
version: 26.607.1731
tags: [ unreal, performance, offline, audit, request ]
---

# Request: bulk offline asset scanner

## Status (2026-Q2)

**Shipped** in sb-unreal master (commit 232a633). Three MCP tools, no
companion required, no editor required:

- ✅ `assets_find_references(target_path, root="/Game")`
- ✅ `assets_find_by_class(classes[], root="/Game")`
- ✅ `assets_scan_offline(root="/Game")`

Implementation: Go-port path (vs C# wrapper), in-process. Files:
`cmd/sb-unreal/uasset.go` + `uasset_tables.go` + `bulk_scan.go` +
`bulk_scan_tools.go` + tests.

The `level_actors_offline` tool from the asset-version-and-metadata
request rides on the same infrastructure — TODO add as a follow-up
using the parsed Exports list on `.umap` files.

---

Original request below.

# Request (original)

**Origin.** Comparative analysis against [UAssetAPI](https://github.com/atenfyr/UAssetAPI) (C# / MIT,
UE 4.13 → 5.7). SystemBridge today reaches every asset through editor RPC
(Python Remote Execution → `EditorAssetLibrary` / `AssetRegistry`). Cost
per RPC: ~30–150ms (UDP discovery + TCP exec + Python preamble + tool body
+ JSON marker scrape). On a 1000-asset project audit that's **30–150
seconds**; the same workload over raw `.uasset` parsing is **1–2 seconds**.

The slowness is fine for single-asset operations on hot assets, but blocks
project-wide auditing. Reasonable requests like "every Blueprint that
references `BP_OldWeapon`", "every DataTable whose row struct is `S_Foo`",
"every level that places `AC_Inventory`" currently cost minutes.

## What

A bulk offline scanner that reads `.uasset` files directly from disk and
answers reference / class / property-shape queries fast. Either:

- **(a) Native-AOT C# wrapper** bundling UAssetAPI in a small standalone
  `sb-asset-reader.exe` invoked from Go. Bundle cost ~30MB. Updates with
  UE versions are free.
- **(b) Go-side parser** in `cmd/sb-unreal/uasset.go` extended to cover
  the property/export types we actually query. More work upfront; zero
  runtime overhead. Drifts with UE versions.

Pick whichever the implementer thinks fits — both produce the same MCP
surface.

## Tools

| Tool | Returns |
|---|---|
| `assets_scan_offline(filter, root="/Game")` | List of `{path, class, references_count, modified_at}` |
| `assets_find_references(target_path, root="/Game")` | List of paths referencing `target_path` |
| `assets_find_by_class(class_path, root="/Game")` | List of asset paths whose top export is the named class |
| `assets_find_by_property(class_path, property, value)` | Asset paths where a top-level property equals a value |

All four operate on disk only — no editor required.

`filter` accepts:

- `class`: exact UClass path or short name (`Blueprint`, `DataTable`, …)
- `references`: asset path; matches assets whose import table includes it
- `name_glob`: filename glob (`SK_*`, `WDA_*`)
- `modified_after`: ISO timestamp
- `path_glob`: package-path glob (`/Game/Weapons/**`)

## Why

The current `unreal_assets_query` is editor-bound and slow on >100 assets.
Bulk audits are the gateway to safe refactors — "rename this BP, can it
break levels?", "is this struct still used anywhere?", "is this anim clip
referenced by any montage?". Without offline-fast answers, AI flows that
need to "look before touching" are economically prohibitive.

## Acceptance

- A 1000-asset project produces a class-grouped index in **under 5
  seconds** (currently 30+).
- `assets_find_references("/Game/Weapons/SK_OldPistol")` returns every
  referencing asset path, with no editor running.
- Results are byte-equivalent to what the editor's
  `AssetRegistry::GetReferencers` returns for the same set.
- Works on both saved-but-not-loaded and editor-cold projects.

## Notes / non-goals

- **Read-only.** No write/mutation surface. Editor-bypass writes are
  unsafe (derived data, generated classes, recompile-on-save).
- **Uncooked editor assets only** for v1. Cooked / `.usmap`-required
  paths can come later if a real need surfaces.
- **Falls back to `AssetRegistry` RPC** when offline parsing fails on
  an asset (corrupt, unknown version, etc.) — never silently misses.
