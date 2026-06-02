---
id: asset-management
title: Asset Management
status: stable
version: 26.602.559
tags: [ unreal, assets ]
---

# Asset Management

UObject asset introspection + mutation. Mostly Python-side; doesn't
require Companion.

See: [unreal deep dive](index.md),
[blueprint authoring](blueprint-authoring.md).

## Read

### `asset_describe`

THE diagnostic before `asset_set_properties` — list every editor
property on a UObject with type + current value. UE 5.7 Python's snake-
case conversion sometimes gives misleading guesses on custom C++
classes (e.g. an FName UPROPERTY `AttachSocketName` is
`attach_socket_name`, not `attach_socket`).

```
unreal_asset_describe asset_path="/Game/Survival/Weapons/WDA_Pistol" name_filter="mesh"
```

Returns:

```jsonc
{
  "success": true,
  "asset": "/Game/.../WDA_Pistol",
  "class": "SurvivalWeaponDataAsset",
  "parent_classes": ["DataAsset", "Object"],
  "property_count": 3,
  "properties": [
    {"name": "mesh",                       "type": "SkeletalMesh",  "value": "None"},
    {"name": "attach_socket_name",         "type": "Name",          "value": "None"},
    {"name": "fire_montage",               "type": "AnimMontage",   "value": "None"}
  ]
}
```

`name_filter` is optional substring (case-insensitive).

`value` is a JSON-safe stringification capped at 200 chars. UObject
values are reported as `/Game/...` paths (not Python repr).

### `asset_set_properties`

Bulk-set top-level properties. Reads each before write, sets, reads
back, compares; returns before/after + errors list.

```
unreal_asset_set_properties asset_path="/Game/.../WDA_Pistol" properties={
  "mesh":         "/Game/.../SK_Pistol",
  "attach_socket_name": "hand_r",
  "fire_montage": "/Game/.../MM_Pistol_Fire_Montage"
}
```

Strings starting with `/Game/` are NOT auto-loaded by this tool — for
that, the helper passes them through `unreal.EditorAssetLibrary.load_asset()`
internally. If the property is a UObject reference, the load happens
correctly; if not, the comparison fails and the error is surfaced.

For pin defaults on Blueprint NODES (not asset CDOs), use
`bp_node_pin_set_object` — see [blueprint authoring](blueprint-authoring.md#object-references-on-pins).

### `assets_query`

AssetRegistry query — fast, no asset loads. Returns
`[{name, package, class}]`.

```
unreal_assets_query
  package_paths=["/Game/Survival"]
  class_names=["SurvivalWeaponDataAsset", "AnimMontage"]
  recursive=true
```

### `read_uasset`

Parse `FPackageFileSummary` from a `.uasset` on disk — engine version,
package name, name/import/export counts. Doesn't require the editor.

Used as a fast pre-flight before more expensive UE operations.

## Dependencies + references

### `asset_dependencies`

What this asset depends on. Bucketed by `/Game`, `/Script`, `/Engine`:

```
unreal_asset_dependencies asset_path="/Game/.../BP_Survivor"
```

Returns:

```jsonc
{
  "asset": "/Game/.../BP_Survivor",
  "hard": {
    "game": ["/Game/.../Skeleton_Survivor", ...],
    "engine": ["/Engine/.../DefaultMaterial", ...],
    "script": ["/Script/Engine.Character", ...]
  },
  "soft": { "game": [...], ... }
}
```

### `asset_referencers`

Inverse — who depends on THIS asset. Critical before delete/rename:

```
unreal_asset_referencers asset_path="/Game/.../AM_HardLanding_Forward"
```

`include_hard` / `include_soft` (default both true).

## DataTable rows

**Companion v1.5.0+**. Stock UE 5.7 Python only exposes
`DataTableFunctionLibrary.fill_data_table_from_csv_string` /
`_from_json_string`, both of which:

- REPLACE the whole table (no per-row write).
- Re-import every row from struct-text, which silently **corrupts**
  UserDefinedStruct fields whose names contain `()`
  (e.g. `Parameters.MaximumRange(InMetres)` becomes `0` on every row
  because the tokenizer treats the parens as a nested struct).

Companion's three new tools do per-row authoring without the round-trip.

### `dt_row_add`

Add a row. With `from_row` set, the new row is a **binary copy** of
that existing row — every byte preserved, parens-named fields
intact. Wraps `UDataTable::AddRow` directly.

```
unreal_dt_row_add dt_path="/Game/Survival/Weapons/DT_Weapons"
                  row_name="AK47"
                  from_row="M416"
  → {success: true, data_table, row_name, from_row}
```

Without `from_row` the row is freshly zeroed.

### `dt_row_set_field`

Set a single (possibly nested) field on a row. Path syntax: `.` as
the segment separator; field names **verbatim** — including names
with `()`. Value goes through `FProperty::ImportText_InContainer` on
the resolved leaf, so it's parsed by the property's own type-aware
parser (NOT struct-text).

```
unreal_dt_row_set_field dt_path=... row_name="AK47"
                        field_path="DisplayName" value="AK-47"

unreal_dt_row_set_field dt_path=... row_name="AK47"
                        field_path="Parameters.MaximumRange(InMetres)"
                        value="300.0"

unreal_dt_row_set_field dt_path=... row_name="AK47"
                        field_path="CoreData.Mesh"
                        value="/Game/Weapons/SK_AK47.SK_AK47"
```

Supports scalar leaves, enum literals, FName, object references
(`/Game/...` paths), and nested struct descent across multiple
`FStructProperty` hops.

### `package_discard_changes`

Drop pending in-memory changes on a package — reload from disk.
Mirror of "Right-click → Revert" in the Content Browser, but
headless.

```
unreal_package_discard_changes asset_path="/Game/.../DT_Weapons"
  → {success: true, asset_path}
```

Use to recover from a botched fill before it gets persisted by
`editor_save_all_dirty` / `editor_restart`. Also paired with
`editor_restart skip_save_all_dirty=true` for the heavier path
(quit without persisting anything).

### End-to-end recipe — "clone M416 → AK-47"

```
# 1. Clone the row byte-for-byte.
unreal_dt_row_add dt_path="/Game/.../DT_Weapons" row_name="AK47" from_row="M416"

# 2. Override the fields that differ.
unreal_dt_row_set_field dt_path=... row_name="AK47"
                        field_path="DisplayName" value="AK-47"
unreal_dt_row_set_field dt_path=... row_name="AK47"
                        field_path="CoreData.Mesh"
                        value="/Game/Weapons/SK_AK47.SK_AK47"
unreal_dt_row_set_field dt_path=... row_name="AK47"
                        field_path="Parameters.MaximumRange(InMetres)"
                        value="300.0"
```

All other rows untouched; all UDS fields including the parens-named
one preserved exactly.

## AnimMontage from template

**Companion v1.6.0+**. UE 5.7 Python can't author
`UAnimMontage::SlotAnimTracks` at all (the property is an unbound
`TArray<FSlotAnimationTrack>`), and `AnimMontageFactory` always lands
new montages on `DefaultGroup.DefaultSlot` (full-body). Per-weapon
**upper-body** reload montages (ALS-style "Arm L"/"Arm R" slots in
the "Layering Override Group") couldn't be created headless.

`anim_montage_create_from_template` clones a known-good template that
already has the desired slot layout and swaps the clip inside every
slot track's segment:

```
unreal_anim_montage_create_from_template
  template_montage_path="/Game/.../ALS_LoadedReload_Montage"
  source_animation_path="/Game/.../ALSP2_Reloading_Rifle_AK47"
  dst_path="/Game/.../MM_AK47_Reload_Montage"
  → {success: true, template, source_animation, dst}
```

What survives the clone:

- Slot LAYOUT (e.g. `Arm L`, `Arm R`, `Spine`, `Head`).
- Group name (e.g. `Layering Override Group`).
- Section markers + chains (`next_section_name`).
- Blend in / out.
- Notifies.

What changes:

- Every `FAnimSegment.AnimReference` is swapped for `source_animation`.
- Segment timings are retimed to the new clip's `GetPlayLength()`
  (`AnimStartTime=0`, `AnimEndTime=length`, `PlayRate=1`, `LoopCount=1`).

Verification after authoring:

```
unreal_anim_sequence_info anim_path=".../MM_AK47_Reload_Montage"
  → inspect section list and play_length

unreal_run_python script="""
import unreal, json
m = unreal.EditorAssetLibrary.load_asset('/Game/.../MM_AK47_Reload_Montage')
slots = unreal.AnimationLibrary.get_montage_slot_names(m)
result = {'slots': [str(s) for s in slots],
          'group': str(m.get_group_name())}
print('<<<SB_JSON>>>'); print(json.dumps(result)); print('<<<END_SB_JSON>>>')
"""
  → slots: ["Arm L", "Arm R", ...], group: "Layering Override Group"
```

In PIE the reload plays upper-body — legs keep locomotion while running.

Skeleton compatibility is checked via `USkeleton::IsCompatibleForEditor`.
Companion refuses if `source_animation`'s skeleton is unrelated;
the caller is responsible for choosing a clip authored on (or
compatible with) the template's skeleton.

Refuses to overwrite an existing asset at `dst_path` — `asset_delete`
first if you mean to.

### End-to-end recipe — per-weapon reload montages

```
# 12-weapon arsenal, one template per family.
TEMPLATES = {
    "Rifle":  "/Game/.../ALS_LoadedReload_Montage",
    "Pistol": "/Game/.../A_HG_Reload_Montage",
    "Shotgun":"/Game/.../A_SG_Reload_Montage",
}
CLIPS = {
    "AK47":  ("Rifle",  "/Game/.../ALSP2_Reloading_Rifle_AK47"),
    "M4A4":  ("Rifle",  "/Game/.../ALSP2_Reloading_Rifle_M4A4"),
    "Famas": ("Rifle",  "/Game/.../ALSP2_Reloading_Rifle_Famas"),
    "C8":    ("Rifle",  "/Game/.../ALSP2_Reloading_Rifle_C8"),
    "AS50":  ("Rifle",  "/Game/.../ALSP2_Reloading_Rifle_AS50"),
    "USP":   ("Pistol", "/Game/.../ALSP2_Reload_Pistol"),
    ...
}

for weapon, (family, clip) in CLIPS.items():
    unreal_anim_montage_create_from_template(
        template_montage_path=TEMPLATES[family],
        source_animation_path=clip,
        dst_path=f"/Game/Weapons/Montages/MM_{weapon}_Reload_Montage",
    )
# then wire each into DT_Weapons via dt_row_set_field on
# "PC ReloadAnimation" (a SoftObjectPath leaf).
```

## Lifecycle

### `asset_duplicate`

```
unreal_asset_duplicate src="/Game/Foo" dst="/Game/Bar"
```

Idempotent — silent success if dst already exists.

### `asset_rename`

```
unreal_asset_rename src="/Game/Foo" dst="/Game/Bar"
```

UE auto-updates references (hard + soft) across the project.

### `asset_delete`

```
unreal_asset_delete asset_path="/Game/Foo" force=false
```

Refuses by default if other assets reference it. `force=true` deletes
anyway — leaves dangling references that compile-error consumers on
next save.

## Patterns

- **"What can I set on this DataAsset?"** → `asset_describe` first.
  Then `asset_set_properties` with the discovered property names.

- **"I'm about to rename / delete this. Will anything break?"** →
  `asset_referencers`. Renaming UE handles automatically; deleting
  needs the consumers fixed (or `force=true` + accepting breakage).

- **"What's actually in this `.uasset`?"** → `read_uasset` for header
  metadata without loading. Cheap pre-flight.

- **"Find all DataTables that reference this monster class."** →
  `assets_query class_names=["DataTable"]` →
  `asset_dependencies` per result → filter by `hard.game` containing
  the class. Or `asset_referencers` on the class itself (faster).

## Cross-references

- [blueprint authoring](blueprint-authoring.md) — `bp_variable_add_typed`
  uses the same `/Game/...` and `/Script/...` paths.
- [unreal plugin tool catalog](../plugins/unreal.md#asset--content-browser).
- [message log](message-log.md) — `LoadErrors` lists assets that failed
  to load.
