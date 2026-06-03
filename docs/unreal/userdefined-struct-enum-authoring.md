---
id: request-author-userdefinedstruct-members-userdefinedenum-entries-headless
title: "Request: author UserDefinedStruct members & UserDefinedEnum entries (headless)"
status: request
version: 26.603.1452
tags: [ unreal, blueprint, authoring, struct, enum, request ]
---

# Request: author UserDefinedStruct members & UserDefinedEnum entries (headless)

## Status

`request` — capability gap confirmed in the field, no SB/companion tool covers it
yet. Filed by AI agent (2026-06-03) while implementing an HLS-style weapon
interaction data model in `ALS_UltimateWarfare`.

## The gap

[Blueprint Authoring (headless)](blueprint-authoring.md) covers Blueprint member
variables, function graphs, nodes, and pins. It does **not** cover authoring the
two *data-definition* asset kinds:

- **`UserDefinedStruct`** — can be *created* (`StructureFactory`) but its members
  cannot be added/typed/renamed/removed from automation.
- **`UserDefinedEnum`** — can be *created* (`EnumFactory`) but its enumerators
  (display names + values) cannot be added/edited from automation.

This blocks any data model that needs a project-authored struct or enum — most
notably **a new `DataTable`, which requires a `UserDefinedStruct` row type**.

## Evidence (UE 5.7.4, companion 1.7.0)

Probed live in-editor via `unreal_run_python`:

- `unreal.EnumFactory`, `unreal.StructureFactory`, `unreal.DataTableFactory`
  all exist → asset creation works.
- `unreal.UserDefinedStruct` / `unreal.UserDefinedEnum` expose only generic
  `UObject` methods (`set_editor_property`, `call_method`, `modify`, …) — no
  member/enumerator editing surface.
- `unreal.EnumEditorUtils` → **absent**. `unreal.StructureEditorUtils` →
  **absent**. (These are the C++ entry points the editor UI uses:
  `FStructureEditorUtils::AddVariable/RemoveVariable/RenameVariable/ChangeVariableType`,
  `FEnumEditorUtils::AddNewEnumeratorForUserDefinedEnum` /
  `SetEnumeratorDisplayName`.)
- `unreal.SystemBridgeBindings` has `add_typed_member_variable`,
  `add_data_table_row`, `set_data_table_row_field`, etc. — but **nothing** for
  struct members or enum entries.
- A freshly `StructureFactory`-created `UserDefinedStruct` lands with only its
  default member and offers no Python path to add more.

Net: `add_typed_member_variable` already proves the companion can build a proper
`FEdGraphPinType` for object/struct/enum/containers on a **Blueprint**. The same
`FEdGraphPinType` construction is what `FStructureEditorUtils::AddVariable`
needs — so this is a small extension of existing companion code, not new ground.

## Why it matters

Without it, the only fully-automatable homes for project-authored data are:

- **Blueprint classes** (DataAsset subclass, or an ActorComponent) whose
  *variables* stand in for struct fields — authorable today via
  `add_typed_member_variable`.
- **GameplayTags / `Name`** instead of enums for state identifiers.

That works, but it forces design decisions (DataAsset instead of DataTable;
tags instead of type-safe enums) purely because of tooling, not architecture.

## Requested capability

New companion UFUNCTIONs + SB tools (names illustrative):

### UserDefinedStruct

```
unreal_struct_create(package_path, name)                       -> {struct_path}
unreal_struct_member_add(struct_path, member_name, pin_category,
                         sub_category_object?, container_type?, default_value?)
unreal_struct_member_remove(struct_path, member_name)
unreal_struct_member_rename(struct_path, old_name, new_name)
unreal_struct_member_set_type(struct_path, member_name, pin_category, ...)
unreal_struct_members_list(struct_path) -> [{name, category, sub_object, container}]
```

- `pin_category` / `sub_category_object` / `container_type` should mirror
  `add_typed_member_variable` exactly (reuse the same `FEdGraphPinType` builder).
- Implement via `FStructureEditorUtils::AddVariable` etc., then
  `FStructureEditorUtils::OnStructureChanged` + compile + save.
- Newly created `UserDefinedStruct`s come with a default member — the tool
  should optionally clear it once a real member is added.

### UserDefinedEnum

```
unreal_enum_create(package_path, name)                         -> {enum_path}
unreal_enum_entry_add(enum_path, display_name)                 # append
unreal_enum_entry_set_display_name(enum_path, index, display_name)
unreal_enum_entry_remove(enum_path, index)
unreal_enum_entries_list(enum_path) -> [{index, name, display_name}]
```

- Implement via `FEnumEditorUtils::AddNewEnumeratorForUserDefinedEnum` and
  `FEnumEditorUtils::SetEnumeratorDisplayName`, then compile + save.

## Acceptance

- Can create `ST_Foo` and add members of every `pin_category`
  (`bool/int/real/name/struct/enum/object/class/soft_class` + array/map/set)
  headlessly, then use `ST_Foo` as a `DataTable` row struct and populate rows
  with `add_data_table_row` / `set_data_table_row_field`.
- Can create `E_Bar` with N named enumerators headlessly, then use it as a
  `bp_variable_add_typed` enum sub-object and in `K2Node_SwitchEnum`.
- Round-trips through save/reload; resulting assets are indistinguishable from
  editor-authored ones.

## Workaround until available

- Use a `DataAsset`/`PrimaryDataAsset` **Blueprint** whose variables are the
  "fields" (authored with `add_typed_member_variable`), one instance asset per
  data row, instead of a `DataTable` + `UserDefinedStruct`.
- Use `GameplayTag` or `Name` variables instead of `UserDefinedEnum`.
- Or have a human author the empty `UserDefinedStruct` / `UserDefinedEnum`
  once in-editor, then drive everything else from automation.
