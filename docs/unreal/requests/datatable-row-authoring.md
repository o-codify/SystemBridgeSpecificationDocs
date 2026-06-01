---
id: request-headless-datatable-row-authoring-add-edit-struct-rows
title: "Request: headless DataTable row authoring (add/edit struct rows)"
status: request
version: 26.601.2245
tags: []
---

# Request: headless DataTable row authoring (add/edit struct rows)

## Problem
There is no reliable way to **add or edit a DataTable row** headless when the
row struct is a UserDefinedStruct (the common case for gameplay tables).

Observed in `ALS_UltimateWarfare` adding an "AK-47" row to `DT_Weapons`
(row struct `ST_WeaponData`, nested CoreData/Parameters/Features/Animations):

- The only bulk-write Python APIs are
  `DataTableFunctionLibrary.fill_data_table_from_csv_string` /
  `fill_data_table_from_json_string`. Both **re-import every row from
  struct-text** and **REPLACE the whole table**.
- A field named **`MaximumRange(InMetres)`** (parentheses in a UserDefinedStruct
  field name) does **not round-trip**: export writes
  `MaximumRange(InMetres)=200.000000`, but on import the struct-text tokenizer
  treats `(InMetres)` as a nested struct and fails → the field is silently set
  to `0` **for every row**. JSON has the same problem (it stores each struct
  column as the same struct-text string).
- There is **no Python API to set a single row's struct field** or to
  add/duplicate a row from an in-memory struct instance. `DataTable` exposes
  only `does_row_exist` / `get_row_names` / `get_row_struct`;
  `DataTableFunctionLibrary` has export/fill/`get_data_table_column_as_string`
  (read) / `remove_data_table_row` — but no row setter / adder.
- `UPackage` in this Python build has no `set_dirty_flag` / `is_dirty`, and
  `EditorLoadingAndSavingUtils.reload_packages` does **not** discard a dirty
  in-memory package — so a botched fill is sticky, and
  `unreal_editor_restart` runs `save_all_dirty`, persisting the corrupted
  table to disk.

Net effect: any attempt to add one row via fill silently **corrupts the
parens field on all existing rows**, and it cannot be undone headless (the
running editor memory-maps/locks the .uasset, so `git checkout` / `cp` fail
with "unable to unlink" / "Device or resource busy"; reload won't drop the
dirty version). Recovery required a fragile dance: a background loop that
rewrites the HEAD `.uasset` bytes during the brief window when
`unreal_editor_restart` has the editor closed.

## Requested capability
A Companion-backed tool family for DataTable rows that does NOT go through
struct-text round-trip:

- `dt_row_add(dt_path, row_name, from_row=None)` — add a row, optionally
  initialized as a **binary copy of an existing row's struct** (so all fields,
  including parens-named UDS fields, are preserved). Wraps
  `UDataTable::AddRow` / row-struct memcpy.
- `dt_row_set_field(dt_path, row_name, field_path, value)` — set a single
  (possibly nested) struct field by name via the property system (FProperty
  ImportText on the leaf scalar, not the whole struct), so
  `Parameters.MaximumRange(InMetres)` can be written directly.
- `dt_row_remove(dt_path, row_name)` (already exists as
  `remove_data_table_row`).
- Mark the package dirty + `bp_compile_and_save`-style save.

Minimum viable: `dt_row_add` with `from_row` (duplicate) + `dt_row_set_field`.
That covers the "clone M416 → AK-47, change Mesh + a few params" workflow with
zero data loss.

## Also useful (safety)
- A `package_discard_changes(asset_path)` / force-reload-from-disk that drops a
  dirty in-memory package (so a bad edit can be reverted without restart).
- `unreal_editor_restart` option to **skip save_all_dirty** (quit without
  saving) — currently it always saves dirty, which persisted the corruption.

## Current workaround
None that's clean. The DataTable row must be authored **manually in the
editor**. The corruption is only recoverable by overwriting the `.uasset` from
git during the editor-closed window of a restart.

## Acceptance
- Adding a row to a UserDefinedStruct DataTable (incl. a field whose name
  contains parentheses) preserves all other rows' fields exactly.
- A single nested struct field can be set without touching siblings.
