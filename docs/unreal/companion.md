---
id: systembridgecompanion-plugin
title: SystemBridgeCompanion Plugin
status: stable
version: 26.609.228
tags: [ unreal, companion, cpp ]
---

# SystemBridgeCompanion Plugin

A tiny Editor-only UE plugin that exposes C++ internals to Python as
`unreal.SystemBridgeBindings.*`. Source lives in the sb-unreal binary via
`go:embed` (`cmd/sb-unreal/companion/`); install / rebuild is driven by
`unreal_companion_install` / `unreal_companion_rebuild`.

See: [unreal deep dive](index.md), [installation](../installation.md).

## Why Companion exists

UE 5.7's stock Python is read-mostly for editor internals. These are
all C++-only and would otherwise require shell escapes through `Cmd.exe`
or unreliable Slate clicks:

- `UEdGraphPin::DefaultObject` — the slot for object/asset pin defaults.
- `TObjectIterator<UBlueprint>` — walk live in-memory BPs.
- `FBlueprintEditorUtils::AddFunctionGraph<UFunction>` — override entry point.
- `FMessageLog` — the editor's Message Log window contents.
- `UMaterial::Expressions` (UE 5.5+: `ExpressionCollection`).
- `UWidgetTree` mutation, BehaviorTree runtime, K2Node creation, …

The Python helpers detect Companion via `hasattr(unreal, "SystemBridgeBindings")`.
When present, the helper uses the binding. When missing, it either falls
back to a Python approximation or returns
`{success: false, reason: "companion_unavailable"}` with a clear hint.

## Install / scopes

```
unreal_companion_install scope=project        # plugin under <project>/Plugins
unreal_companion_install scope=engine         # plugin under <UE>/Engine/Plugins/Marketplace
```

- **Project scope** — best for trying out / shipping the companion with a
  specific project. Adds an entry to the `.uproject` and writes a line to
  `.gitignore` (so the rebuilt binaries don't enter git by default).
- **Engine scope** — best for multiple projects on the same UE install.
  The plugin lives under the engine and is available to every project
  using that engine version.

Both scopes run `RunUAT BuildPlugin` against the staged source. Build
takes 30s – 3min depending on caches.

## Detecting drift

The Go binary embeds an `expectedCompanionVersion`. `unreal_companion_status`
compares it to the loaded `CompanionVersion()` and surfaces
`companion_version_mismatch: true` when they differ. Fix:

```
unreal_companion_rebuild
```

The rebuild extracts the embedded source freshly, runs UAT, and syncs the
new DLL into the installed location. Editor must be closed (or the
build will hit "file in use" on the DLL link step).

## Status bar widget

v1.3.1+ adds a permanent widget on the LevelEditor's status bar:

```
[● green dot] SB v1.13.1
```

Tooltip lists what's armed and the repo URL. Hover-to-confirm replaces
having to call `companion_status`.

## Version timeline

| Version | What was added |
|---|---|
| **0.1.0** | Read-only introspection: `CompanionVersion`, `GetBlueprintGraphs`, `GetBlackboardKeys`. |
| **0.2.0** | K2Node graph editing: enumeration, creation, pin linking. |
| **0.3.0** | Full pin introspection (types, links, defaults), node removal, link breaking, `K2Node_CallFunction` target setup, `ReconstructNode`. |
| **0.4.0** | Pin default-value setting (string slot), variable-reference K2Nodes, event-node creation (BeginPlay/Tick/CustomEvent), event lookup. Closed the gap that prevented v0.3 graphs from actually executing. |
| **0.5.0** | DynamicCast (TargetType), Make/BreakStruct (StructType), SwitchEnum (Enum), node repositioning, per-pin link introspection. |
| **0.6.0** | Subsystem access (`K2Node_GetSubsystem`) + Enhanced Input action events (`K2Node_EnhancedInputAction` ↔ `UInputAction` asset). |
| **0.7.0** | Material expression graph enumeration. The verbs (add / connect / remove / recompile) ride on stock `MaterialEditingLibrary`; Companion only fills the read-side gap. |
| **0.8.0** | UMG WidgetTree authoring — list / add / remove / rename / slot-property. Stock UE Python doesn't bind WidgetTree at all. |
| **0.9.0** | Blackboard key authoring + BehaviorTree runtime-tree node lifecycle. |
| **1.0.0** | Stability boundary. Replaced `Modify()` with bare `MarkPackageDirty()` in BT mutations to avoid the PostEditChange cascade that dropped just-added children mid-session. v1.0 covers full authoring of K2Node graphs / Material expressions / UMG WidgetTree / BehaviorTree+Blackboard end-to-end. |
| **1.1.0** | PIE lifecycle: `StartPlayInEditor` (real PIE with possession) and `ToggleBetweenPIEAndSIE` (F8 equivalent). Stock UE 5.7 Python only exposes `editor_play_simulate` (SIE only); v1.1 gives parity with the editor toolbar. |
| **1.2.0** | FMessageLog access — `GetMessageLog(category)` + `ListMessageLogCategories()`. These messages live in the Message Log window, separate from `<Project>.log`. Without this, package-load errors / MapCheck / AssetCheck weren't visible to the agent. See [message log](message-log.md). |
| **1.3.0** | Function override authoring — `ListOverridableFunctions` + `OverrideParentFunction`. UE 5.7 Python's `add_function_graph` creates fresh functions only; v1.3 wraps `FBlueprintEditorUtils::AddFunctionGraph` for parent overrides (BlueprintNativeEvent / BlueprintImplementableEvent). See [blueprint authoring](blueprint-authoring.md#function-overrides). |
| **1.3.1** | Editor status-bar widget on LevelEditor.StatusBar.ToolBar. |
| **1.3.2** | Drop `meta=(ScriptMethod)` from pure-static UFUNCTIONs — UE was warning at every editor launch about `CompanionVersion`, `StartPlayInEditor`, `ToggleBetweenPIEAndSIE`, `GetMessageLog`, `ListMessageLogCategories` not having a UObject first-arg. |
| **1.3.3** | PIE pre-flight: live in-memory BP error scan via `TObjectIterator<UBlueprint>`. The previous AssetRegistry-tag scan missed BPs whose status was live-in-memory but not yet serialized. Used by `pie_start` to refuse with a structured error instead of dispatching into UE's modal compile-error dialog. |
| **1.3.4** | Object/asset references in Blueprints (headless): `SetPinDefaultObject` (writes to `Pin->DefaultObject`, the slot string defaults can't touch), `AddTypedMemberVariable` (full FEdGraphPinType — object / class / soft_object / soft_class / interface / struct / enum / containers), `CreateObjectLiteralNode` (`K2Node_Literal` + `SetObjectRef`). See [blueprint authoring](blueprint-authoring.md). |
| **1.4.0** | Replication authoring — `ConfigureCustomEvent` (UK2Node_CustomEvent::FunctionFlags: Multicast / Server / Client + Reliable + bCallInEditor), `AddEditableEventParam` (typed UserDefinedPin on custom events / function entries), `SetVariableReplication` (CPF_Net / CPF_RepNotify on FBPVariableDescription + auto-create OnRep_* stub). Closes the multiplayer authoring gap — networked features were impossible to author headless before. See [blueprint authoring → replication](blueprint-authoring.md#replication). |
| **1.5.0** | DataTable row authoring + package safety — `AddDataTableRow` (binary memcpy from an existing row OR fresh zeroed buffer, no struct-text round-trip), `SetDataTableRowField` (FProperty::ImportText_InContainer on a dotted-path leaf — works on nested UserDefinedStruct fields including names with parens like `Parameters.MaximumRange(InMetres)` that the stock fill APIs corrupt), `DiscardPackageChanges` (drops dirty in-memory package, re-loads from disk — headless equivalent of Content Browser's Revert). Closes the gameplay-DataTable authoring gap. See [asset management → DataTable rows](asset-management.md#datatable-rows). |
| **1.6.0** | AnimMontage slot-track authoring — `CreateAnimMontageFromTemplate` duplicates a template montage (preserving its slot layout, group name, sections, blends, notifies) and swaps the AnimSequence inside every segment for a new clip. UE 5.7 Python can't author `UAnimMontage::SlotAnimTracks` at all (unbound TArray) and `AnimMontageFactory` always lands new montages on `DefaultGroup.DefaultSlot` (full-body). Unblocks per-weapon upper-body reload authoring — e.g. ALS "Arm L"/"Arm R" + "Layering Override Group" reloads, per gun, from a single template. See [asset management → AnimMontage from template](asset-management.md#animmontage-from-template). |
| **1.7.0** | AnimMontage notify authoring — `AddAnimMontageNotify` (skeleton/named FAnimNotifyEvent at a chosen time + optional notify-state duration; resolves or creates the track; registers the name on the skeleton; idempotent on name+time+track) and `RemoveAnimMontageNotifyByName` (TArray::RemoveAll predicate). UE 5.7 Python's `AnimationLibrary.add_animation_notify_event` needs a UClass — passing None creates a notify literally named "None"; the notifies array is protected for direct write. Unblocks placing "ReloadWeapon" (and friends) at a chosen time on per-weapon montages. See [asset management → AnimMontage notifies](asset-management.md#animmontage-notifies). |
| **1.8.0** | Headless data authoring — three coalesced gaps. **BP/DT**: `SetBlueprintVariableDefault` (typed CDO write with kind dispatch: object / class / soft_object / soft_class / gameplaytag / auto), `GetBlueprintVariables` (NewVariables is protected in Python), `GetDataTableRowFieldsJson` (flat dotted-path map, inverse of dt_row_set_field). **GameplayTags**: `GameplayTagAdd / Remove / List / Refresh` via `IGameplayTagsEditorModule` + `EditorRefreshGameplayTagTree` — no more editor restart per tag addition. **UserDefinedStruct / UserDefinedEnum**: `StructMemberAdd / Remove / Rename / MembersList` via `FStructureEditorUtils`, `EnumEntryAdd / SetDisplayName / Remove` via `FEnumEditorUtils`. Closes the DataTable-from-scratch flow. Pure-Python helpers ship the asset creation half (`bp_create`, `dataasset_create`, `struct_create`, `enum_create`, `dt_export`, `enum_entries_list`). See [data authoring](data-authoring.md). |
| **1.13.4** | Repair broken-typed Blueprint member variables — closes the `bp-variable-lifecycle` spec request. Variables whose `FEdGraphPinType.PinSubCategoryObject` points at a deleted `UEnum` / `UScriptStruct` / `UClass` used to leave their Blueprint permanently `BS_Error` (only fixable by hand in the editor). Two new bindings: `RepairBlueprintVariableRemove(BP, name)` walks every `K2Node_Variable*` across all four graph kinds (Ubergraphs / Function / Macro / Delegate) whose `VariableReference.MemberName == name`, breaks each one's pin links, removes the node, THEN calls `FBlueprintEditorUtils::RemoveMemberVariable` — the orphan sweep is what makes the BP land `BS_UpToDate` (without it, dangling Get/Set nodes kept it in `BS_Error`). Returns nodes_swept. `RepairBlueprintVariableRetype(BP, name, cat, sub_obj_path, container)` sanitizes `NewVariables[i].VarType` to a Bool placeholder when the existing type needs a subobject but has none, calls `ChangeMemberVariableType` to the target type, then `ReconstructNode`-s every `K2Node_Variable` referencing the var so pin sets resolve. Plus `GetBlueprintCompileStatus(BP)` exposes `Blueprint->Status` as `up_to_date` / `dirty` / `error` / `up_to_date_with_warnings` / `being_created` / `unknown` for callers to verify the BP actually compiled clean. `bp_variable_remove_direct` / `bp_variable_retype` now route through the repair variants (same MCP API, more robust internals) and surface `nodes_swept` + `blueprint_status` in their response. Touches only the target Blueprint — no project-wide recompile that would flip sibling Blueprints with unrelated latent errors into `BS_Error`. |
| **1.13.3** | Bug-fix follow-up to v1.13.2 + a debug reader. Three bugs found in live ALS overlay SM use: (1) `compile_blueprint` TypeError because the helper passed `node.get_outer()` (an EdGraph) where a UBlueprint is required — now resolves UBlueprint via `_expect_animbp` first; on any exception the result is `{success:false, edit_applied:true, exception: <200-char repr>}` instead of echoing the 381 KB helper source. (2) `enum_value` couldn't resolve UserDefinedEnum display names ("Surrender" → not_resolvable; only internal `NewEnumeratorN` worked) — new `ResolveEnumEntryValue` walks numeric → display-name (`GetDisplayNameTextByIndex`) → plain-name → post-`::` segment. (3) **Critical**: the rule didn't actually gate the transition — state stayed unreachable. Root cause: K2 schema silently refused the variable-out → byte-in link because EqualEqual_ByteByte input pins were plain bytes while the variable-out was an enum-typed byte; the compare degraded to literal-vs-literal which evaluated true once and then ignored the variable. Fix: stamp `PinSubCategoryObject = EnumObj` on both A and B inputs BEFORE making links, verify `LinkedTo` after `MakeLinkTo` (which is void in 5.7), `BreakAllPinLinks` on ResultBoolIn for clean replace, ReconstructNode on VarGet and Eq. New `AnimStateMachineGetTransitionRule` reader returns a pipe-separated `{rule_text \| has_var_get \| has_compare \| compare_fn \| enum_path \| enum_value_index \| negated \| wired_to_result}` for verifying that a written rule landed correctly. One new MCP tool: `anim_state_machine_get_transition_rule`. |
| **1.13.2** | Anim state-machine transition rule + state pose authoring — closes the v1.13.0 follow-up gap. `AnimStateMachineSetStatePose(SMNode, stateGuid, animPath, loop)` rebuilds a state's inner BoundGraph with a fresh `UAnimGraphNode_SequencePlayer` wired to the `UAnimGraphNode_StateResult` Pose pin (FAnimNode_SequencePlayer.Sequence + loop set via reflection on the inner struct). `AnimStateMachineSetTransitionRuleEnumEquals(SMNode, transGuid, varName, enumPath, valueName, negate)` rebuilds a transition's BoundGraph as `K2Node_VariableGet → EqualEqual_ByteByte / NotEqual_ByteByte ← ByteLiteral(enumValueIndex) → TransitionResult.bCanEnterTransition`. Enum entries resolved by display name. Together these let an agent add a usable state to an existing shared SM (ALS overlay pattern, etc.) headlessly. Two new MCP tools: `anim_state_machine_set_state_pose`, `anim_state_machine_set_transition_rule_enum_equals`. Closes the anim-state-machine-transition-rule-and-state-pose spec request. |
| **1.13.1** | Three Blueprint authoring families closed in one drop. **Macros**: `bp_macros_list`, `bp_macro_create` (FBlueprintEditorUtils::AddMacroGraph), `bp_node_macro_instance_target` (wire K2Node_MacroInstance to a named macro on the same BP or an external BPL). **Interface implementation**: `bp_interfaces_list`, `bp_interface_add` (FBlueprintEditorUtils::ImplementNewInterface; idempotent; auto _C resolution), `bp_interface_remove`. **Function locals**: `bp_function_locals_list` (walks UK2Node_FunctionEntry::LocalVariables), `bp_function_local_add` (FBlueprintEditorUtils::AddLocalVariable). Eight new tools. |
| **1.13.0** | AnimGraph state machine authoring — biggest remaining AnimGraph coverage gap. Six tools: `anim_state_machine_states_list` (per-state guid + name + kind + asset_path + out_transitions), `anim_state_machine_add_state`, `anim_state_machine_add_conduit`, `anim_state_machine_add_transition` (auto-wires From→Trans→To pins), `anim_state_machine_entry_state` (follow UAnimStateEntryNode output), inner-graph access for population via existing anim_node_* tools. Transition rule (boolean condition) authoring deferred to v1.13.x — newly-added transitions ship with default always-true rule. |
| **1.12.5** | Bytecode operand decode. Extends `bp_function_bytecode` from opcode-sequence-only to a proper recursive walker with per-opcode operand decode: FProperty names, UFunction names, typed constants (FVector / FRotator / FTransform / int / float / FName / FString), cast targets, jump offsets, switch case counts. ~40 opcodes covered; rest halt the walker cleanly. Now diagnoses 'graph compiles but runtime is wrong' concretely (the self-context bug example shows EX_LocalOutVariable without a preceding EX_Self). |
| **1.12.4** | Polish pass. (a) Structured `_expect_*` helpers in sb_helpers.py replacing raw EAL load + isinstance — every 'not_found' / 'wrong_class' goes through the v1.12 fallback chain and carries an actionable `hint` field. Entry-point refactor: bp_overridable_functions / bp_components_list / bp_get_info / bp_variables_list / anim_blueprint_graphs / anim_blueprint_nodes. Closes a class of EAL-blind bugs for the public surface. (b) Universal `limit` / `offset` pagination via `_wrap_paginated`. Applied to anim_blueprint_nodes (closes the 600+ node AnimBP output-overflow request) + bp_variables_list + anim_blueprint_graphs. Back-compat: legacy `{nodes/graphs/variables/count}` fields preserved alongside the new `{items/total/has_more/next_offset}` envelope. (c) Hint field on common errors. (d) New `graph_filter` + `asset_filter` substring matches on anim_blueprint_nodes. |
| **1.16.0** (sb-unreal) | Pure-Python ship — no companion DLL bump. 7 tools. **MRQ**: `mrq_preset_create` (UMoviePipelineMasterConfig with output_format / resolution / sample_count / output_dir), `mrq_render_submit` (PIE executor in-process), `mrq_queue_list`. **WP**: `world_partition_cells_list` (runtime cells on WP-enabled maps; returns `not_wp_enabled` typed error otherwise). **Niagara read**: `niagara_emitter_list`. **Physics**: `physics_constraint_list` (components on actor + bound actors), `physics_constraint_actor_spawn` (basic spawn; deep profile mutation deferred to companion C++). Each tool degrades gracefully with a structured `mrq_unavailable` / `physics_constraint_unavailable` reason if its subsystem isn't bound in this engine build. |
| **1.15.0** (sb-unreal) | Pure-Python ship — no companion DLL bump. 5 tools. **Level actor attach**: `level_actor_attach` (parent / socket, `rule` ∈ snap / keep_world / keep_relative), `level_actor_detach`, `level_actor_attachments_list` (parent + children with sockets). **Material connections read**: `material_expression_pin_links` (mirror of `bp_node_pin_links` for materials, was the asymmetric gap), `material_connections_list` (full edge dump for audit / diff / refactor). Closes 2 spec requests. |
| **1.14.0** (sb-unreal) | Pure-Python ship via UE 5.7 `MovieSceneScriptingChannel`. 10 sequencer tools. **Read**: `sequencer_bindings_list` (possessables + spawnables with class + track_count), `sequencer_tracks_list`, `sequencer_track_info` (sections with channel inventory), `sequencer_keys_list`. **Write**: `sequencer_track_add` (vocab: transform/float/bool/integer/byte/vector/color/event/audio/camera_cut/anim/subscene/actor_ref), `sequencer_track_remove`, `sequencer_section_add` (tick-resolution frame range), `sequencer_key_add` (typed value coerce; interpolation ∈ auto/user/break/linear/constant), `sequencer_key_remove` (idempotent), `sequencer_binding_add` (bind level actor by label or path). End-to-end cinematic pipeline closes — author + render via MRQ. |
| **1.13.2** | Anim state-machine transition rule + state pose authoring — closes the v1.13.0 follow-up gap. `AnimStateMachineSetStatePose(SMNode, stateGuid, animPath, loop)` rebuilds a state's inner BoundGraph with a fresh `UAnimGraphNode_SequencePlayer` wired to the `UAnimGraphNode_StateResult` Pose pin (FAnimNode_SequencePlayer.Sequence + loop set via reflection on the inner struct). `AnimStateMachineSetTransitionRuleEnumEquals(SMNode, transGuid, varName, enumPath, valueName, negate)` rebuilds a transition's BoundGraph as `K2Node_VariableGet → EqualEqual_ByteByte / NotEqual_ByteByte ← ByteLiteral(enumValueIndex) → TransitionResult.bCanEnterTransition`. Enum entries resolved by display name. Together these let an agent add a usable state to an existing shared SM (ALS overlay pattern, etc.) headlessly. Two new MCP tools: `anim_state_machine_set_state_pose`, `anim_state_machine_set_transition_rule_enum_equals`. Closes the anim-state-machine-transition-rule-and-state-pose spec request. |
| **1.13.1** | Three Blueprint authoring families closed in one drop. **Macros**: `bp_macros_list`, `bp_macro_create` (`FBlueprintEditorUtils::AddMacroGraph`), `bp_node_macro_instance_target` (wire K2Node_MacroInstance to a named macro on the same BP or an external BPL). **Interface impl**: `bp_interfaces_list`, `bp_interface_add` (`FBlueprintEditorUtils::ImplementNewInterface`, idempotent, auto `_C` resolution), `bp_interface_remove`. **Function locals**: `bp_function_locals_list` (walks `UK2Node_FunctionEntry::LocalVariables`), `bp_function_local_add` (`FBlueprintEditorUtils::AddLocalVariable`). 8 tools. |
| **1.13.0** | AnimGraph state machine authoring — biggest remaining AnimGraph gap. 6 tools: `anim_state_machine_states_list` (per-state guid + name + kind + asset_path + out_transitions), `anim_state_machine_add_state`, `anim_state_machine_add_conduit`, `anim_state_machine_add_transition` (auto-wires From→Trans→To), `anim_state_machine_entry_state`, plus inner-graph access via existing `anim_node_*`. Transition rule (boolean condition) authoring deferred. |
| **1.12.5** | Bytecode operand decode. Extends `bp_function_bytecode` from opcode-sequence-only to a recursive walker with per-opcode operand decode: FProperty names, UFunction names, typed constants (FVector / FRotator / FTransform / int / float / FName / FString), cast targets, jump offsets, switch case counts. ~40 opcodes covered; rest halt the walker cleanly. Now diagnoses 'graph compiles but runtime is wrong' concretely (the self-context bug shows `EX_LocalOutVariable` without a preceding `EX_Self`). |
| **1.12.4** | Polish pass. (a) Structured `_expect_*` helpers in `sb_helpers.py` replacing raw EAL load + isinstance — every 'not_found' / 'wrong_class' carries an actionable `hint` field. Refactor: `bp_overridable_functions` / `bp_components_list` / `bp_get_info` / `bp_variables_list` / `anim_blueprint_graphs` / `anim_blueprint_nodes`. (b) Universal `limit` / `offset` pagination via `_wrap_paginated`. (c) Hint field on common errors. (d) `graph_filter` + `asset_filter` substring matches on anim_blueprint_nodes. |
| **1.12.3** | BP compiled bytecode inspector — `GetFunctionBytecode(UBlueprint*, FName)` reads `UFunction::Script` and produces a line-by-line disassembly. Covers ~80 most common EX_* opcodes (EX_Let / EX_LocalVariable / EX_VirtualFunction / EX_StructConst / EX_DynamicCast / etc.); unknown ones surface as `EX_0xNN (raw)`. Diagnoses 'graph compiles but runtime is wrong' — compiler-pruned branches, missing self-context (the self-context bug in [blueprint-authoring.md](blueprint-authoring.md)), wrong cast paths. Full operand decode deferred; opcode sequence alone is usually enough to spot pruning / branch logic problems. |
| **1.12.2** | Two diagnostic tools from the asset-version-and-metadata roadmap request. `GetAssetVersionInfo(UObject*)` returns saved engine version + `FCustomVersionContainer` + warnings (engine-skew note when major/minor differ). Reads `UPackage::GetLinker()->Summary` first; falls back to disk `FPackageFileSummary` via `IFileManager`. `GetAssetMetadata(UObject*)` walks `UPackage::GetMetaData()->GetMapForObject(O)` for the asset + its immediate subobjects — author-time hints, tooltips, categories. Diagnoses 'why won't this load' (engine skew + custom-version mismatch are usual culprits, previously invisible) and supports refactor tooling that respects existing categorization. `level_actors_offline` (the third item from the request) deferred to the bulk-offline-scanner work item. |
| **1.12.1** | Closes the AnimBP / BP component tooling gaps request. (a) `bp_set_component_property_typed` now resolves `ClassProperty` values correctly — v1.12.0 tried `StaticLoadClass("/Game/.../ABP_Foo.ABP_Foo")` which lands on the BP asset, not its generated class; v1.12.1 tries path-as-is → `_C` suffix → `LoadObject<UBlueprint>->GeneratedClass`. (b) New `bp_component_remove` — symmetric counterpart to `bp_component_add` via `SubobjectDataSubsystem.delete_subobject` + compile + save, idempotent. (c) `anim_blueprint_nodes` now returns `graph_path` + `node_guid` + `current_asset` on every entry. (d) New `anim_reassign_asset(abp_path, from_asset, to_asset, graph_filter)` — by-value rewrite that walks every `AnimGraphNode_*` whose inner `FAnimNode_*` matches `from_asset` and replaces with `to_asset`. |
| **1.12.0** | Eight spec requests closed in one drop. **BP variable lifecycle**: `RemoveBlueprintVariableDirect` / `RenameBlueprintVariableAtomic` / `RetypeBlueprintVariable` (canonical `FBlueprintEditorUtils`). **SkeletalMesh sockets**: `AddSkeletalMeshSocket` writes parent bone; `mesh_sockets_list` switched to public `num_sockets()` / `get_socket_by_index()`; `skeleton_bones_list` walks `SkeletalMeshEditorSubsystem.get_bone_tree` first. **SCS typed**: `SetSCSComponentPropertyTyped`. **Runtime invoke**: `InvokeFunctionOnObject` builds a parameter frame from string args and calls `ProcessEvent`. **PIE input**: `InjectEnhancedInputAction` + `InjectKeyInputPIE`. **Single-node inspect**: `bp_node_inspect_by_guid`. **Asset path blind**: `LoadAssetWithFallback` (EAL → AssetRegistry → StaticLoadObject). **AnimMontage regression**: `CreateAnimMontageFromTemplate` sweeps template-anim refs via `FArchiveReplaceObjectRef`. Build.cs picks up `AssetRegistry`, `EditorScriptingUtilities`, `InputCore`. |
| **1.11.0** | AnimGraph authoring — closes the AnimGraph half of headless authoring (previously only K2 graphs were editable). `CreateAnimGraphNode` instantiates any `UAnimGraphNode_*` subclass (ControlRig, TwoBoneIK, ModifyBone, LayeredBoneBlend, SequencePlayer, …) by class path. `SetAnimNodeInnerProperty` writes a UPROPERTY on the inner `FAnimNode_*` runtime struct via `FProperty::ImportText_InContainer` — dot-notation paths, object/class leaves accept asset paths. Auto-reconstructs so class-driven pin sets update (e.g. `ControlRigClass` exposes the rig's variables as pins). `SetAnimNodeExposePin` toggles entries in `UAnimGraphNode_Base::ShowPinForProperties` and reconstructs. `GetAnimNodeInfo` gives read-side parity (class, inner struct name, position, pins with linked counts + types, exposed pin properties). Linking pose AND data pins reuses existing `LinkPins` — UAnimGraphNodes ARE UEdGraphNodes. Build.cs picks up `AnimGraph` + `AnimGraphRuntime`. See [animgraph authoring](animgraph-authoring.md). |
| **1.10.1** | Bug fix for rig variables shipped in v1.10.0. v1.10.0 created rig variables via `FBlueprintEditorUtils::AddMemberVariable` — that writes `UBlueprint::NewVariables` but does NOT register the variable with the RigVM model. `URigVMController::AddVariableNode` then couldn't resolve the name, left a broken `URigVMVariableNode` bound to `@@` (missing variable), and every subsequent rig-* call failed with "Variable Node @@ is using a missing variable" — including compile and removes of unrelated nodes. **Fix**: route through the canonical `URigVMBlueprint::AddMemberVariable` / `RemoveMemberVariable` / `GetMemberVariables` surface, which publishes to both the BP variable list and the RigVM model. CPPType derivation now uses `RigVMTypeUtils::CPPTypeFromPinType` (controller-canonical). Belt-and-braces: get/set node creation now sanity-checks the returned `URigVMVariableNode`'s binding and `RemoveNode`s any half-bound node rather than poisoning the graph. |
| **1.10.0** | Control Rig rig-variable authoring — `ControlRigVariableAdd` (`FBlueprintEditorUtils::AddMemberVariable` + direction-derived `CPF_*` flags; public variables become input pins on the AnimGraph Control Rig node), `ControlRigVariablesList` (name / `cpp_type` / `cpp_type_object` / direction / default), `ControlRigVariableRemove` (idempotent), `ControlRigVariableGetNodeAdd` and `ControlRigVariableSetNodeAdd` (`URigVMController::AddVariableNode` with CPPType + CPPTypeObject derived from the existing BP variable description). Closes the v1.9 follow-up: static pin defaults couldn't express per-frame data (hand IK targets, elbow poles, alphas) — rig variables make the AnimBP-facing dynamic-input contract authorable. See [control rig authoring](control-rig-authoring.md#rig-variables-v110). |
| **1.9.0** | Control Rig (RigVM) authoring — `ControlRigGraphsList` (main RigGraph; function library deferred to v1.10), `ControlRigNodesList` (name / title / RigUnit struct / position / pin count), `ControlRigAddUnitNode` (FRigUnit-derived `UScriptStruct` → new node name), `ControlRigRemoveNode` (idempotent), `ControlRigPinSetDefault` (dot-notation pin path + RigVM text serializer), `ControlRigAddLink` (exec/data link, "Pin.SubPin" paths), `ControlRigCompile` (`RecompileVM` + `MarkBlueprintAsModified`). All wrap `URigVMController` (RigVMDeveloper) and `UControlRigBlueprint` (ControlRigDeveloper, post-Modular-Rig `*Legacy.h` headers). Asset creation rides on pure-Python `ControlRigBlueprintFactory`. UE 5.7 Python doesn't bind `URigVMController` at all. See [control rig authoring](control-rig-authoring.md). |

## Future roadmap

**Companion C++ (not yet shipped):**

- **Niagara module-stack mutation** — add / remove / set inputs on emitter modules (`UNiagaraScriptStack::AddScript`). Read shipped pure-Python in v1.16.0.
- **Physics constraint profile mutation** — per-axis limits / motors via `FConstraintInstance`. Basic spawn shipped pure-Python in v1.16.0.
- **World Partition deeper read** — actor → cell mapping, per-cell actor enumeration. Cell list shipped pure-Python in v1.16.0.
- **Transition rule authoring** — boolean condition graphs on AnimGraph state-machine transitions (v1.13.0 follow-up).
- **Material function authoring** — `material_function_create / expressions_list / expression_add / compile` + `material_function_call_node_add`. Connections read shipped in v1.15.0.

**Pure-Python opportunities (no companion bump):**

- Sequencer transform-curve smoothing utilities.
- Bulk actor-property mutation across level loaded subsets.

## Upgrading the companion

After `git pull` on the sb-unreal repo:

```
scripts/build.sh                 # rebuild sb.exe (embeds new companion source)
                                 # restart Claude Code so the new binary loads
unreal_companion_status          # see version drift
unreal_companion_rebuild         # rebuild the .dll on disk
                                 # restart UE editor to load the new DLL
```

For users who can't / don't want to invoke `unreal_companion_rebuild` from
the agent, manually:

```
1. Close UE editor + LiveCodingConsole + sb daemons.
2. Copy source from <repo>/cmd/sb-unreal/companion/ over the engine plugin
   path: <UE>/Engine/Plugins/Marketplace/SystemBridgeCompanion/
3. Run: "C:\Program Files\Epic Games\UE_5.7\Engine\Build\BatchFiles\RunUAT.bat" BuildPlugin -Plugin=<path>.uplugin -Package=<tmp> -Rocket -TargetPlatforms=Win64
4. Sync <tmp>/Binaries/ and <tmp>/Intermediate/ back into the plugin dir.
5. Restart UE.
```

## Cross-references

- [unreal plugin tool catalog](../plugins/unreal.md) — shows which tools
  require which Companion version (table column "Companion?").
- [blueprint authoring](blueprint-authoring.md) — Companion is heaviest there.
- [troubleshooting](../troubleshooting.md#companion-drift).
