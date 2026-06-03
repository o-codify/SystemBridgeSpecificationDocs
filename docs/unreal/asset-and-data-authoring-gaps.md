---
id: request-headless-asset-creation-data-authoring-bp-dataasset-create-var-defaults-dt-read-var-list
title: "Request: headless asset creation & data authoring (BP/DataAsset create, var defaults, DT read, var list)"
status: request
version: 26.603.1452
tags: [ unreal, blueprint, dataasset, datatable, authoring, request ]
---

# Request: headless asset creation & data authoring

## Status

`request` — confirmed gaps (UE 5.7.4, companion 1.7.0). Filed 2026-06-03 while
building per-weapon DataAsset profiles + a runtime component for an HLS weapon
interaction layer. Each item below was hit in the field and worked around with
`run_python`; the dedicated tools exist for everything else (variables, nodes,
components, DT writes) so these close obvious holes.

## Gaps & workarounds

### 1. Create a Blueprint / DataAsset asset (with parent class)
No SB tool creates a new `UBlueprint` (or a BP-defined `DataAsset` subclass).
`bp_*` tools all assume the BP already exists.
- Workaround: `run_python` — `unreal.BlueprintFactory` (set `parent_class`) +
  `AssetTools.create_asset(...)`; for DataAsset instances `unreal.DataAssetFactory`
  with `data_asset_class`.
- Request: `unreal_bp_create(package_path, name, parent_class_path)` and
  `unreal_dataasset_create(package_path, name, data_asset_class_path)`.

### 2. Set a Blueprint member-variable DEFAULT value
`bp_variable_add_typed` can set a default only for *object* vars (`default_object`).
There is no tool to set the default of an existing variable, especially typed
defaults (struct / FGameplayTag / FVector / soft-class / bool / enum) on the
BP's CDO.
- Workaround: `run_python` — `cdo = unreal.get_default_object(bp.generated_class());
  cdo.set_editor_property(name, value); EditorAssetLibrary.save_loaded_asset(bp)`.
  Note: `SoftClassPath` was rejected for a SoftClass property — had to pass the
  loaded `UClass` object instead.
- Request: `unreal_bp_variable_set_default(bp_path, name, value, value_kind?)`
  accepting the same canonical serialization `set_pin_default_value` uses, plus
  object/class-by-path and gameplaytag-by-name.

### 3. Set complex property values on a DataAsset / UObject
`unreal_asset_set_properties` exists but: "strings starting with /Game/ are NOT
auto-loaded", and it could not take an FGameplayTag (constructed value) or a
SoftClass. So filling DataAsset instances with tags / soft-class / struct values
required `run_python`.
- Request: extend `asset_set_properties` to accept `{"__gameplaytag__":"X.Y"}`,
  `{"__object__":"/Game/..."}`, `{"__class__":"/Game/..._C"}`,
  `{"__struct__":"(...)"}` value envelopes, or add
  `unreal_asset_set_property_typed`.

### 4. Read a DataTable row / values
Write side is covered (`unreal_dt_row_add`, `unreal_dt_row_set_field`) but there
is no read tool. To get M416/M9 socket fields I had to
`run_python` → `DataTableFunctionLibrary.export_data_table_to_json_string`.
- Request: `unreal_dt_row_get(dt_path, row_name)` →
  flattened field map, and/or `unreal_dt_export(dt_path, format=json|csv)`.

### 5. List a Blueprint's member variables
No tool lists a BP's variables with types. `bp.get_editor_property('new_variables')`
is **protected** (fails). I had to scan graph nodes for `K2Node_VariableGet/Set`
titles to discover names — lossy and indirect.
- Request: `unreal_bp_variables_list(bp_path)` →
  `[{name, pin_category, sub_object, container, default}]` (mirror of the
  `add_typed_member_variable` taxonomy).

## Acceptance

A DataAsset-backed data model can be built end-to-end with dedicated tools:
create the DataAsset class, add typed fields, create instances, set typed
defaults (incl. gameplaytag/soft-class/struct), list the fields back, and read
source DataTable rows — **without any `run_python`**.

## Related

[UserDefinedStruct/Enum authoring](userdefined-struct-enum-authoring.md) and
[GameplayTag authoring](gameplaytag-authoring.md) — same headless-authoring
family.
