---
id: changelog
title: Changelog
status: stable
version: 26.607.1700
tags: [ changelog ]
---

# Changelog

Milestones for SystemBridge as a whole — daemon (`sb.exe`), plugins,
and the Unreal Companion sub-plugin. Individual companion versions are
also captured in [companion plugin reference](unreal/companion.md).

## Recent: Companion v1.11.x — AnimGraph authoring + transform queries

| Date | Tag | What |
|---|---|---|
| 2026-Q2 | **Companion v1.12.1** | Closes the AnimBP / BP component tooling gaps request. `bp_set_component_property_typed` ClassProperty resolution fix (path → `_C` → `GeneratedClass`); new `bp_component_remove` (SubobjectDataSubsystem.delete_subobject + compile + save); `anim_blueprint_nodes` now returns `graph_path` + `node_guid` + `current_asset` per entry; new `anim_reassign_asset(from, to, graph_filter)` for by-value asset rewrite across an AnimBP. |
| 2026-Q2 | **Companion v1.12.0** | Eight spec requests in one drop: BP variable lifecycle (`bp_variable_remove_direct` / `_rename_atomic` / `_retype` — surgical, atomic, no collateral), SkeletalMesh socket authoring + read fixes (`mesh_socket_add` sets parent bone; helper switched to public socket-by-index API), SCS typed property setter (`bp_set_component_property_typed`), runtime PIE invoke (`runtime_invoke`), Enhanced Input + raw key injection (`pie_input_inject`), single-node inspect (`bp_node_inspect_by_guid`), `LoadAssetWithFallback` (defensive AssetRegistry path when EAL goes blind in 5.7.4), AnimMontage clip-swap regression fix (`FArchiveReplaceObjectRef` sweep across the duplicated montage). Build.cs picks up `AssetRegistry`, `EditorScriptingUtilities`, `InputCore`. |
| 2026-Q2 | **Companion v1.11.1** | Two v1.11.0 bug fixes. `anim_node_add` couldn't return the guid for `AnimGraphNode_ControlRig` (and other subclasses that don't expose NodeGuid as a Python attribute) — the node WAS created; only the readback failed. Switched to the companion `get_node_guid` binding. Docstring/example path corrected: Control Rig anim node is `/Script/ControlRigDeveloper.AnimGraphNode_ControlRig`. `anim_node_expose_pin` couldn't toggle Control Rig rig variables because `UAnimGraphNode_CustomProperty` subclasses store bindable-variable pins in `CustomPinProperties`, not `ShowPinForProperties`. Now walks both via `FArrayProperty` reflection. New `anim_node_list_exposable_pins` enumerates both lists with `source` tags so callers can discover the bindable surface. |
| 2026-Q2 | **Companion v1.11.0** | AnimGraph authoring — 5 tools mirroring the `bp_node_*` surface for `UAnimGraphNode_*` (ControlRig / TwoBoneIK / ModifyBone / LayeredBoneBlend / SequencePlayer / …). `anim_node_add` instantiates a node by class path; `anim_node_set_inner_property` writes UPROPERTYs on the inner `FAnimNode_*` runtime struct via `FProperty::ImportText_InContainer` (dot-notation, object/class leaves accept asset paths, reconstruct fires so class-driven pin sets update); `anim_node_expose_pin` toggles `ShowPinForProperties`; `anim_node_info` gives read parity; `anim_node_remove` routes through `bp_node_remove`. Linking reuses existing K2 link tools — AnimGraph nodes ARE UEdGraphNodes. Build.cs picks up `AnimGraph` + `AnimGraphRuntime`. Closes the AnimGraph half of headless authoring. See [animgraph authoring](unreal/animgraph-authoring.md). |
| 2026-Q2 | sb-unreal | 5 transform-query tools — the missing read-side primitive for IK / attachment / VFX alignment. `mesh_sockets_list`, `skeleton_bones_list`, `skeleton_bone_transform`, `mesh_socket_transform`, `actor_transform_query` (with optional `relative_to`). All pure Python — no companion required. See [transform query](unreal/transform-query.md). |
| 2026-Q2 | sb-unreal | `expectedCompanionVersion` externalised to a `expected_companion_version.txt` sidecar next to the binary. Bumping the expected companion version no longer requires rebuilding `sb-unreal.exe` — and so no longer requires a Claude Desktop restart, which is what was triggering the "transport closed" cascade users reported. See [mcp transport recovery](mcp-transport-auto-recovery.md). |

## Companion v1.10.x — Control Rig rig-variable authoring

| Date | Tag | What |
|---|---|---|
| 2026-Q2 | **Companion v1.10.1** | Bug fix for v1.10.0 rig variables. `URigVMController::AddVariableNode` was leaving broken nodes bound to `@@` because variables were added via `FBlueprintEditorUtils::AddMemberVariable` (BP-only) instead of the canonical `URigVMBlueprint::AddMemberVariable` (BP + RigVM model). Switched to the canonical surface for add / remove / list; CPPType comes from `RigVMTypeUtils::CPPTypeFromPinType`. Get/set node creation sanity-checks the returned binding and rolls back any half-bound node. Closes the v1.10.0 follow-up bug report. |
| 2026-Q2 | **Companion v1.10.0** | Rig-variable authoring — completes the AnimBP-facing dynamic-input contract v1.9 left as static-defaults-only. `ControlRigVariableAdd` (FBlueprintEditorUtils::AddMemberVariable + direction-derived CPF flags; public variables become input pins on the AnimGraph Control Rig node), `ControlRigVariablesList` (name / cpp_type / cpp_type_object / direction / default — direction recovered from a `RigVar|Input/Output/Hidden` category tag), `ControlRigVariableRemove` (idempotent), `ControlRigVariableGetNodeAdd` + `SetNodeAdd` (`URigVMController::AddVariableNode` with CPPType + CPPTypeObject derived from the existing BP variable description). 5 MCP tools. Pure-C++ `BPTypeToRigVMType` maps PC_* + sub-object into RigVM's `FVector` / `int32` / `TArray<FTransform>` etc. format. See [control rig authoring → rig variables](unreal/control-rig-authoring.md#rig-variables-v110). |

## Companion v1.9.x — Control Rig (RigVM) authoring

| Date | Tag | What |
|---|---|---|
| 2026-Q2 | **Companion v1.9.0** | Control Rig authoring — wraps `URigVMController` (RigVMDeveloper) so an agent can stand up a procedural-animation rig headlessly. `ControlRigGraphsList` (main RigGraph; function-library deferred to v1.10), `ControlRigNodesList` (name / title / RigUnit struct / position / pin count), `ControlRigAddUnitNode` (FRigUnit-derived `UScriptStruct` → node name), `ControlRigRemoveNode` (idempotent), `ControlRigPinSetDefault` (dot-notation pin path + RigVM text serializer), `ControlRigAddLink` (exec/data link, "Pin.SubPin" paths), `ControlRigCompile` (`RecompileVM` + `MarkBlueprintAsModified`). Uses `*Legacy.h` headers post-Modular-Rig migration. Asset creation rides on pure-Python `ControlRigBlueprintFactory`. 8 MCP tools. Closes the rig-authoring gap — UE 5.7 Python doesn't bind `URigVMController` at all. See [control rig authoring](unreal/control-rig-authoring.md). |

## Companion v1.8.x — headless data authoring

| Date | Tag | What |
|---|---|---|
| 2026-Q2 | **Companion v1.8.0** | Headless data authoring — three coalesced requests closed. **BP/DT**: `SetBlueprintVariableDefault` (typed CDO write — object / class / soft_object / soft_class / gameplaytag / auto), `GetBlueprintVariables` (NewVariables is protected), `GetDataTableRowFieldsJson` (flat dotted-path map). **GameplayTags**: `GameplayTagAdd / Remove / List / Refresh` via `IGameplayTagsEditorModule` + `EditorRefreshGameplayTagTree` — no editor restart per tag addition. **UserDefinedStruct / UserDefinedEnum**: `StructMember*` via `FStructureEditorUtils`, `EnumEntry*` via `FEnumEditorUtils` — closes the DataTable-from-scratch flow. 21 new MCP tools, 11 of which are pure Python (asset creation + dt_export + enum_entries_list). See [data authoring](unreal/data-authoring.md). |

## Companion v1.7.x — AnimMontage notify authoring

| Date | Tag | What |
|---|---|---|
| 2026-Q2 | **Companion v1.7.0** | `AddAnimMontageNotify` (skeleton/named FAnimNotifyEvent + optional notify-state duration, resolves or creates the NotifyTrack, registers the name on the skeleton, idempotent on name+time+track) + `RemoveAnimMontageNotifyByName`. UE 5.7 Python can't write a *named* notify — `AnimationLibrary.add_animation_notify_event` requires a UClass, passing None makes a notify literally named "None". Two MCP tools: `anim_montage_add_notify`, `anim_montage_remove_notify_by_name`. See [asset management → AnimMontage notifies](unreal/asset-management.md#animmontage-notifies). |

## Companion v1.6.x — AnimMontage slot-track authoring

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
