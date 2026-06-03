---
id: data-authoring-headless
title: Data Authoring (headless)
status: stable
version: 26.603.1516
tags: [ unreal, data, datatable, gameplaytags, userdefined-struct, userdefined-enum ]
---

# Data Authoring (headless)

The headless authoring pipeline (BP nodes, AnimMontages, etc.) needs to
sit on top of project-authored **data** to be useful. Companion v1.8
adds the missing slice: create assets, set typed defaults, read
DataTables, manage GameplayTags, author UserDefinedStruct members and
UserDefinedEnum entries.

This page covers the **v1.8.0** tool surface. For closely-adjacent
authoring see:

- [Blueprint authoring](blueprint-authoring.md) — graphs, nodes,
  variables, function overrides, object refs (v1.0 → v1.4).
- [Asset management](asset-management.md) — `asset_describe`,
  DataTable rows, AnimMontage from template, AnimMontage notifies.
- [Companion plugin](companion.md) — version timeline + install.

## Why headless data authoring is hard in UE

UE 5.7 Python is read-mostly for these surfaces:

- `BlueprintFactory` / `DataAssetFactory` / `StructureFactory` /
  `EnumFactory` exist → asset CREATION works.
- `UBlueprint::NewVariables` is **protected** → can't read variable
  type info.
- `UserDefinedStruct` / `UserDefinedEnum` expose only generic
  `UObject` methods → can't add/remove/rename members.
- `EnumEditorUtils` / `StructureEditorUtils` are absent from Python.
- `UGameplayTagsManager`, `GameplayTagsSettings`,
  `BlueprintGameplayTagLibrary`, `GameplayTagsEditorModule` are all
  absent → no headless tag CRUD.
- `DataTableFunctionLibrary` covers WRITE side via fill / RemoveRow,
  but the fill APIs corrupt parens-named UDS fields (see
  [asset management](asset-management.md#datatable-rows)) and there
  is no per-row READ.
- BP variable CDO defaults need typed import paths Python's
  `set_editor_property` can't always supply (SoftClass, GameplayTag).

Companion v1.8 wraps the C++ entry points and ships pure-Python
helpers for the bits that don't need C++.

## Asset creation

All pure Python — Companion not required for these four.

### `bp_create`

Create a new `UBlueprint` asset with the given parent class.
`BlueprintFactory.parent_class` + `AssetTools.create_asset`.

```
unreal_bp_create
  package_path="/Game/Blueprints"
  name="BP_MyActor"
  parent_class_path="/Script/Engine.Actor"
  → {success: true, bp_path: "/Game/Blueprints/BP_MyActor"}
```

`parent_class_path` accepts native classes (`/Script/Engine.Character`)
and BP classes (`/Game/.../BP_Parent.BP_Parent_C` — note the `_C`).

### `dataasset_create`

Create a new DataAsset INSTANCE of the given DataAsset subclass.

```
unreal_dataasset_create
  package_path="/Game/Data/Weapons"
  name="DA_AK47"
  data_asset_class_path="/Script/MyGame.WeaponDataAsset"
```

### `struct_create`

Create an empty `UserDefinedStruct` via `StructureFactory`. The new
struct lands with one default member; use `struct_member_add` to
build the real schema, then `struct_member_remove` to drop the
default if you don't want it.

### `enum_create`

Create an empty `UserDefinedEnum` via `EnumFactory`. Use
`enum_entry_add` to populate it.

## Blueprint variable defaults + introspection

### `bp_variable_set_default`

Set the default value of an EXISTING BP member variable on the BP's
generated CDO. `bp_variable_add_typed` accepts a default only at
create time (and only for object vars); this tool covers the
"existing variable, set default later" case for all kinds.

```
unreal_bp_variable_set_default
  bp_path="/Game/.../BP_Foo"
  variable_name="Health"
  value="100.0"
  value_kind="auto"
```

`value_kind` chooses the import strategy:

| Kind | What |
|---|---|
| `auto` / `string` | `FProperty::ImportText_InContainer` — primitives, struct literals, FName, enums. |
| `object` | `StaticLoadObject(Value)` → `SetObjectPropertyValue`. |
| `class` | `StaticLoadClass(Value)` → `SetObjectPropertyValue`. |
| `soft_object` / `soft_class` | Writes `FSoftObjectPath` / `FSoftClassPath`. |
| `gameplaytag` | `UGameplayTagsManager::RequestGameplayTag(Value)` → assigns to `FGameplayTag` field. |

After the write Companion calls `Blueprint::MarkPackageDirty` +
`MarkBlueprintAsModified` and Python compiles + saves.

### `bp_variables_list`

Enumerate a BP's member variables with their type info — mirrors the
`bp_variable_add_typed` taxonomy so the output round-trips into
re-authoring.

```
unreal_bp_variables_list bp_path="/Game/.../BP_Foo"
  → {variables: [
      {name: "Health", pin_category: "real", sub_category_object: "",
       container: "single", default: "100.0"},
      {name: "EquippedWeapon", pin_category: "object",
       sub_category_object: "/Script/MyGame.WeaponDataAsset",
       container: "single", default: ""},
      {name: "ActiveTags", pin_category: "struct",
       sub_category_object: "/Script/GameplayTags.GameplayTagContainer",
       container: "single", default: ""},
      ...
    ]}
```

`new_variables` is protected in UE 5.7 Python — Companion walks it
in C++ and surfaces the data.

## DataTable read

The write half (`dt_row_add`, `dt_row_set_field`,
`package_discard_changes`) shipped in v1.5; v1.8 adds the read half.

### `dt_row_get`

Read a row as a flat `{field_path: stringified_value}` map. Inverse
of `dt_row_set_field` — keys returned are valid `field_path` inputs.
Nested struct hops dotted; names verbatim including parens.

```
unreal_dt_row_get
  dt_path="/Game/.../DT_Weapons"
  row_name="AK47"
  → {fields: {
      "DisplayName": "AK-47",
      "CoreData.Mesh": "/Game/Weapons/SK_AK47.SK_AK47",
      "Parameters.MaximumRange(InMetres)": "300.0",
      ...
    }}
```

UObject references are surfaced as `/Game/...` paths; scalars via
`FProperty::ExportText`.

### `dt_export`

Bulk export the WHOLE table as JSON or CSV — pure Python via the
stock `DataTableFunctionLibrary`. Useful when you need the entire
table at once for analysis or generation.

```
unreal_dt_export dt_path="/Game/.../DT_Weapons" format="json"
  → {format: "json", content: "{ \"AK47\": { ... }, ... }"}
```

## GameplayTags

Stock UE 5.7 Python exposes no headless GameplayTag CRUD. v1.8
wraps `IGameplayTagsEditorModule` + `UGameplayTagsManager`. **No
editor restart needed** for tag addition — the in-memory tree is
refreshed via `EditorRefreshGameplayTagTree`.

### `gameplaytag_add`

Add a tag:

```
unreal_gameplaytag_add
  tag="State.Weapon.Idle"
  dev_comment="Weapon is holstered and idle."
  source_ini=""    # default: "DefaultGameplayTags.ini"
  → {success: true, tag: "State.Weapon.Idle", ...}
```

Writes `+GameplayTagList=(Tag="State.Weapon.Idle",DevComment="...")` to
the source `.ini` AND registers in-memory. Idempotent — re-adding an
existing tag is a no-op success.

### `gameplaytag_add_many`

Batch add. Accepts list of strings OR list of dicts:

```
unreal_gameplaytag_add_many tags=[
  "State.Weapon.Idle",
  "State.Weapon.Aiming",
  {"tag": "State.Weapon.Reloading", "dev_comment": "Reload in progress"},
]
  → {added: [...], skipped: [...]}
```

### `gameplaytag_remove`

Remove a previously-added tag from its source .ini and from memory.

### `gameplaytag_list`

Enumerate registered tags. Optional `prefix` filter:

```
unreal_gameplaytag_list prefix="State.Weapon"
  → {count: 3, tags: [
      {tag: "State.Weapon.Idle", dev_comment: "...", source: "DefaultGameplayTags.ini", is_explicit: true},
      ...
    ]}
```

### `gameplaytag_refresh`

Rebuild the in-memory tree from sources — call after editing .ini
files outside this companion.

## UserDefinedStruct authoring

`struct_create` makes the empty asset; the four `struct_member_*`
tools add/remove/rename members and list current ones. Same
`pin_category` taxonomy as `bp_variable_add_typed`.

### `struct_member_add`

```
unreal_struct_member_add
  struct_path="/Game/Data/ST_WeaponStats"
  member_name="Damage"
  pin_category="real"

unreal_struct_member_add
  struct_path=...
  member_name="DamageTags"
  pin_category="struct"
  sub_category_object="/Script/GameplayTags.GameplayTagContainer"

unreal_struct_member_add
  struct_path=...
  member_name="FireMontage"
  pin_category="object"
  sub_category_object="/Script/Engine.AnimMontage"
```

### `struct_member_remove` / `struct_member_rename` / `struct_members_list`

Self-explanatory. Each saves the struct asset on success.

### End-to-end — DataTable from scratch

The killer use case: build a new DataTable whose row struct is
authored alongside it.

```
# 1. Author the struct.
unreal_struct_create package_path="/Game/Data" name="ST_WeaponRow"
unreal_struct_member_add  struct_path="/Game/Data/ST_WeaponRow"
                          member_name="DisplayName" pin_category="string"
unreal_struct_member_add  ... member_name="Mesh" pin_category="object"
                              sub_category_object="/Script/Engine.SkeletalMesh"
unreal_struct_member_add  ... member_name="Damage" pin_category="real"
unreal_struct_member_add  ... member_name="ReloadMontage" pin_category="object"
                              sub_category_object="/Script/Engine.AnimMontage"

# 2. Strip the default placeholder member (if any).
unreal_struct_members_list struct_path=...
# remove anything you don't want

# 3. Create the DataTable on top of it.
# (DataTableFactory needs a row struct — set via run_python or
#  AssetTools::CreateAssetWithDialog flow; see workflows below.)

# 4. Populate rows.
unreal_dt_row_add dt_path=... row_name="AK47"
unreal_dt_row_set_field dt_path=... row_name="AK47" field_path="DisplayName" value="AK-47"
unreal_dt_row_set_field dt_path=... row_name="AK47" field_path="Damage" value="40.0"
unreal_dt_row_set_field dt_path=... row_name="AK47" field_path="Mesh"
                        value="/Game/Weapons/SK_AK47.SK_AK47"
```

## UserDefinedEnum authoring

### `enum_entry_add`

Append an enumerator with a display name:

```
unreal_enum_entry_add enum_path="/Game/Data/E_WeaponState"
                      display_name="Idle"
unreal_enum_entry_add ... display_name="Aiming"
unreal_enum_entry_add ... display_name="Reloading"
```

### `enum_entry_set_display_name`

Edit an existing entry by index.

### `enum_entry_remove`

Drop by index.

### `enum_entries_list`

Pure Python (no companion) — walks `UEnum.num_enums()` +
`get_display_name_text_by_index`:

```
unreal_enum_entries_list enum_path="/Game/Data/E_WeaponState"
  → {entries: [
      {index: 0, name: "E_WeaponState::NewEnumerator0", display_name: "Idle"},
      {index: 1, name: "E_WeaponState::NewEnumerator1", display_name: "Aiming"},
      {index: 2, name: "E_WeaponState::NewEnumerator2", display_name: "Reloading"},
    ]}
```

## Patterns

- **DataAsset-backed catalog.** `bp_create` the DataAsset BP class,
  `bp_variable_add_typed` for each field, then `dataasset_create` per
  row + `asset_set_properties` to populate.

- **DataTable-backed catalog.** `struct_create` + N
  `struct_member_add` for the row schema, `dt_row_add` for each row,
  `dt_row_set_field` for the cells, `dt_row_get` to round-trip.

- **Tag-authored state machine.** `gameplaytag_add_many` for the
  initial vocab, `bp_variable_add_typed pin_category="struct"
  sub_category_object="/Script/GameplayTags.GameplayTag"` for state
  fields, `bp_variable_set_default value_kind="gameplaytag"` for
  per-class defaults.

## Cross-references

- [companion plugin](companion.md) — v1.8 timeline entry.
- [asset management](asset-management.md) — DataTable rows, AnimMontage
  authoring (the v1.5–v1.7 surface this builds on top of).
- [blueprint authoring](blueprint-authoring.md) — graph + variable
  authoring.
- [unreal plugin tool catalog](../plugins/unreal.md) — full tool list
  with Companion-required column.
