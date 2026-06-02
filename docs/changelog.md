---
id: changelog
title: Changelog
status: stable
version: 26.602.559
tags: [ changelog ]
---

# Changelog

Milestones for SystemBridge as a whole — daemon (`sb.exe`), plugins,
and the Unreal Companion sub-plugin. Individual companion versions are
also captured in [companion plugin reference](unreal/companion.md).

## Recent: Companion v1.6.x — AnimMontage slot-track authoring

| Date | Tag | What |
|---|---|---|
| 2026-Q2 | **Companion v1.6.0** | `CreateAnimMontageFromTemplate` duplicates a template montage and swaps the AnimSequence inside every slot track's segment for a new clip. Preserves slot layout (e.g. ALS "Arm L"/"Arm R"), group name ("Layering Override Group"), section markers, blend in/out, notifies. Skeleton compatibility check via `IsCompatibleForEditor`. Closes the per-weapon upper-body reload gap — UE 5.7 Python can't author `UAnimMontage::SlotAnimTracks` at all and `AnimMontageFactory` always lands on `DefaultGroup.DefaultSlot` (full-body). One MCP tool: `anim_montage_create_from_template`. See [asset management → AnimMontage from template](unreal/asset-management.md#animmontage-from-template). |

## Companion v1.5.x — DataTable authoring + cold-start

| Date | Tag | What |
|---|---|---|
| 2026-Q2 | **Companion v1.5.0** | DataTable row authoring without struct-text round-trip. `AddDataTableRow` (UDataTable::AddRow + optional binary memcpy from existing row), `SetDataTableRowField` (dotted-path resolver + FProperty::ImportText on leaf — works on UserDefinedStruct fields whose names contain `()` that the stock fill APIs corrupt), `DiscardPackageChanges` (EditorLoadingAndSavingUtils::ReloadPackages — drop dirty in-memory package). Three MCP tools: `dt_row_add`, `dt_row_set_field`, `package_discard_changes`. Closes the gameplay-DataTable authoring gap. See [asset management → DataTable rows](unreal/asset-management.md#datatable-rows). |
| 2026-Q2 | sb-unreal | New tool `editor_launch` — cold-start a closed UE editor. Resolves engine binary from EngineAssociation, spawns detached + polls until alive. Idempotent. Closes the no-tool-for-cold-start gap (workaround was shelling out to `Start-Process UnrealEditor.exe`). |
| 2026-Q2 | sb-unreal | `editor_restart` adds `skip_save_all_dirty` — quit without persisting dirty assets. For recovery from a known-bad in-memory state. |

## Companion v1.4.x — multiplayer authoring

| Date | Tag | What |
|---|---|---|
| 2026-Q2 | **Companion v1.4.0** | Replication authoring. `ConfigureCustomEvent` writes UK2Node_CustomEvent::FunctionFlags (Multicast / Server / Client + Reliable) + `bCallInEditor`. `AddEditableEventParam` builds typed UserDefinedPins on custom events / function entries. `SetVariableReplication` sets CPF_Net / CPF_RepNotify on FBPVariableDescription, sets RepNotifyFunc, auto-creates OnRep_<var> stub function graph. Three MCP tools: `bp_custom_event_configure`, `bp_event_add_param`, `bp_variable_set_replication`. Closes the multiplayer authoring gap — networked features were impossible to author headless before. See [blueprint authoring → replication](unreal/blueprint-authoring.md#replication). |

## Companion v1.3.x — Blueprint authoring depth

| Date | Tag | What |
|---|---|---|
| 2026-Q1 | **Companion v1.3.4** | Object / asset references in Blueprints headlessly: `SetPinDefaultObject` (writes `UEdGraphPin::DefaultObject`), `AddTypedMemberVariable` (full FEdGraphPinType: object/class/soft_*/interface/struct/enum/containers + CDO default), `CreateObjectLiteralNode` (`K2Node_Literal` + `SetObjectRef`). Smart-routing in `bp_node_pin_set_default` for asset-path values. Closes the long-standing "value rejected by schema" failure on object pins. |
| 2026-Q1 | **Companion v1.3.3** | PIE pre-flight: live in-memory BP error scan via `TObjectIterator<UBlueprint>`. AssetRegistry-tag scan was missing BPs whose error was live but not yet serialized. `pie_start` now refuses with a structured error instead of dispatching into UE's modal dialog. |
| 2026-Q1 | **Companion v1.3.2** | Drop `meta=(ScriptMethod)` from pure-static UFUNCTIONs (`CompanionVersion`, `StartPlayInEditor`, `ToggleBetweenPIEAndSIE`, `GetMessageLog`, `ListMessageLogCategories`). UE was warning at every editor launch about ScriptMethod requiring a UObject first-arg. |
| 2026-Q1 | **Companion v1.3.1** | Editor status-bar widget on `LevelEditor.StatusBar.ToolBar`. Shows `● SB v1.3.x` permanently — no need to round-trip `companion_status` to confirm bindings are loaded. |
| 2026-Q1 | **Companion v1.3.0** | Function override authoring — `ListOverridableFunctions` + `OverrideParentFunction`. Wraps `FBlueprintEditorUtils::AddFunctionGraph<UFunction>` so K2Node_FunctionEntry binds to the parent (Parent: super calls work) and FunctionResult is auto-added. |

## Recent: daemon-side improvements

| Date | Component | What |
|---|---|---|
| 2026-Q1 | sb-unreal | `pie_start` AssetRegistry-tag pre-check for BP compile errors (later superseded by v1.3.3 in-memory walk). |
| 2026-Q1 | sb-unreal | `run_python` output pruning: Info entries dropped when `parsed_json` present; 4KB per-entry cap; `Command` echo blanked (was ~150KB). |
| 2026-Q1 | sb-files | `read` accepts `offset` / `limit` matching Claude Code's built-in Read semantics. Response includes `slice_mode`, `start_line`, `end_line`, `total_lines`. |
| 2026-Q1 | sb-unreal | `editor_set_auto_recover` tool — explicit AI override for the crash watcher with optional TTL. |
| 2026-Q1 | sb-unreal | Persistent crash watcher gets three suppression layers (cooldown, build-tool detection, explicit override). `editor_restart` auto-suppresses for its duration. |
| 2026-Q1 | sb-unreal | `editor_message_log` + `editor_message_log_categories` tools (Companion v1.2). |
| 2026-Q1 | sb-unreal | `asset_describe` tool — list all editor properties on any UObject. |
| 2026-Q1 | sb-unreal | `live_coding_compile` is now synchronous + persists `Startup=AutomaticButHidden` via direct INI write + optional `skip_action_limit`. |
| 2026-Q1 | sb-unreal | Persistent crash watcher + auto-recover (initial version). |
| 2026-Q1 | various | Fixed GitHub URL across plugin manifests (`anthropics/SystemBridge` → `o-codify/SystemBridge`). |

## Earlier: Companion v0.x → v1.2.x

| Version | Theme |
|---|---|
| v1.2.0 | FMessageLog access |
| v1.1.0 | PIE lifecycle (StartPlayInEditor, ToggleBetweenPIEAndSIE) |
| v1.0.0 | Stability boundary; full authoring surface |
| v0.9.0 | Blackboard + BehaviorTree authoring |
| v0.8.0 | UMG WidgetTree authoring |
| v0.7.0 | Material expression enumeration |
| v0.6.0 | Subsystem + Enhanced Input |
| v0.5.0 | Cast / MakeStruct / SwitchEnum / repositioning |
| v0.4.0 | Pin defaults, variable refs, event nodes |
| v0.3.0 | Pin introspection, node remove, link break, ReconstructNode |
| v0.2.0 | K2Node graph editing |
| v0.1.0 | Read-only introspection |

See [companion plugin reference](unreal/companion.md) for per-version
detail.

## Breaking changes

- **Companion source URL** (`CreatedByURL` in the `.uplugin`) changed
  from `github.com/anthropics/SystemBridge` to
  `github.com/o-codify/SystemBridge`. No behavior change — metadata only.
- **`bp_variable_add` with non-primitive types now warns / fails
  loudly** (was silent IntProperty). Use
  `bp_variable_add_typed` instead.
- **`unreal_companion_install`** now requires explicit `scope=project|engine`.

## Convention

The agent generally doesn't care about per-Companion version detail until
`editor_status` says `companion_version_mismatch: true`. The "current
expected" is encoded in `expectedCompanionVersion` inside `sb-unreal.exe`
and bumped per release.

## Cross-references

- [companion plugin](unreal/companion.md) — full per-version detail
  with code-level notes.
- [troubleshooting](troubleshooting.md) — drift, broken DLL, missing
  tools.
