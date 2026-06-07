---
id: systembridgecompanion-plugin
title: SystemBridgeCompanion Plugin
status: stable
version: 26.607.1701
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
[● green dot] SB v1.12.1
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
| **1.12.1** | Closes the AnimBP / BP component tooling gaps request. (a) `bp_set_component_property_typed` now resolves `ClassProperty` values correctly — v1.12.0 tried `StaticLoadClass("/Game/.../ABP_Foo.ABP_Foo")` which lands on the BP asset, not its generated class; v1.12.1 tries path-as-is → `_C` suffix → `LoadObject<UBlueprint>->GeneratedClass`. (b) New `bp_component_remove` — symmetric counterpart to `bp_component_add` via `SubobjectDataSubsystem.delete_subobject` + compile + save, idempotent. (c) `anim_blueprint_nodes` now returns `graph_path` + `node_guid` + `current_asset` on every entry. (d) New `anim_reassign_asset(abp_path, from_asset, to_asset, graph_filter)` — by-value rewrite that walks every `AnimGraphNode_*` whose inner `FAnimNode_*` matches `from_asset` and replaces with `to_asset`. |
| **1.12.0** | Eight spec requests closed in one drop. **BP variable lifecycle**: `RemoveBlueprintVariableDirect` / `RenameBlueprintVariableAtomic` / `RetypeBlueprintVariable` (canonical `FBlueprintEditorUtils`). **SkeletalMesh sockets**: `AddSkeletalMeshSocket` writes parent bone; `mesh_sockets_list` switched to public `num_sockets()` / `get_socket_by_index()`; `skeleton_bones_list` walks `SkeletalMeshEditorSubsystem.get_bone_tree` first. **SCS typed**: `SetSCSComponentPropertyTyped`. **Runtime invoke**: `InvokeFunctionOnObject` builds a parameter frame from string args and calls `ProcessEvent`. **PIE input**: `InjectEnhancedInputAction` + `InjectKeyInputPIE`. **Single-node inspect**: `bp_node_inspect_by_guid`. **Asset path blind**: `LoadAssetWithFallback` (EAL → AssetRegistry → StaticLoadObject). **AnimMontage regression**: `CreateAnimMontageFromTemplate` sweeps template-anim refs via `FArchiveReplaceObjectRef`. Build.cs picks up `AssetRegistry`, `EditorScriptingUtilities`, `InputCore`. |
| **1.11.0** | AnimGraph authoring — closes the AnimGraph half of headless authoring (previously only K2 graphs were editable). `CreateAnimGraphNode` instantiates any `UAnimGraphNode_*` subclass (ControlRig, TwoBoneIK, ModifyBone, LayeredBoneBlend, SequencePlayer, …) by class path. `SetAnimNodeInnerProperty` writes a UPROPERTY on the inner `FAnimNode_*` runtime struct via `FProperty::ImportText_InContainer` — dot-notation paths, object/class leaves accept asset paths. Auto-reconstructs so class-driven pin sets update (e.g. `ControlRigClass` exposes the rig's variables as pins). `SetAnimNodeExposePin` toggles entries in `UAnimGraphNode_Base::ShowPinForProperties` and reconstructs. `GetAnimNodeInfo` gives read-side parity (class, inner struct name, position, pins with linked counts + types, exposed pin properties). Linking pose AND data pins reuses existing `LinkPins` — UAnimGraphNodes ARE UEdGraphNodes. Build.cs picks up `AnimGraph` + `AnimGraphRuntime`. See [animgraph authoring](animgraph-authoring.md). |
| **1.10.1** | Bug fix for rig variables shipped in v1.10.0. v1.10.0 created rig variables via `FBlueprintEditorUtils::AddMemberVariable` — that writes `UBlueprint::NewVariables` but does NOT register the variable with the RigVM model. `URigVMController::AddVariableNode` then couldn't resolve the name, left a broken `URigVMVariableNode` bound to `@@` (missing variable), and every subsequent rig-* call failed with "Variable Node @@ is using a missing variable" — including compile and removes of unrelated nodes. **Fix**: route through the canonical `URigVMBlueprint::AddMemberVariable` / `RemoveMemberVariable` / `GetMemberVariables` surface, which publishes to both the BP variable list and the RigVM model. CPPType derivation now uses `RigVMTypeUtils::CPPTypeFromPinType` (controller-canonical). Belt-and-braces: get/set node creation now sanity-checks the returned `URigVMVariableNode`'s binding and `RemoveNode`s any half-bound node rather than poisoning the graph. |
| **1.10.0** | Control Rig rig-variable authoring — `ControlRigVariableAdd` (`FBlueprintEditorUtils::AddMemberVariable` + direction-derived `CPF_*` flags; public variables become input pins on the AnimGraph Control Rig node), `ControlRigVariablesList` (name / `cpp_type` / `cpp_type_object` / direction / default), `ControlRigVariableRemove` (idempotent), `ControlRigVariableGetNodeAdd` and `ControlRigVariableSetNodeAdd` (`URigVMController::AddVariableNode` with CPPType + CPPTypeObject derived from the existing BP variable description). Closes the v1.9 follow-up: static pin defaults couldn't express per-frame data (hand IK targets, elbow poles, alphas) — rig variables make the AnimBP-facing dynamic-input contract authorable. See [control rig authoring](control-rig-authoring.md#rig-variables-v110). |
| **1.9.0** | Control Rig (RigVM) authoring — `ControlRigGraphsList` (main RigGraph; function library deferred to v1.10), `ControlRigNodesList` (name / title / RigUnit struct / position / pin count), `ControlRigAddUnitNode` (FRigUnit-derived `UScriptStruct` → new node name), `ControlRigRemoveNode` (idempotent), `ControlRigPinSetDefault` (dot-notation pin path + RigVM text serializer), `ControlRigAddLink` (exec/data link, "Pin.SubPin" paths), `ControlRigCompile` (`RecompileVM` + `MarkBlueprintAsModified`). All wrap `URigVMController` (RigVMDeveloper) and `UControlRigBlueprint` (ControlRigDeveloper, post-Modular-Rig `*Legacy.h` headers). Asset creation rides on pure-Python `ControlRigBlueprintFactory`. UE 5.7 Python doesn't bind `URigVMController` at all. See [control rig authoring](control-rig-authoring.md). |

## Future roadmap (v1.12+)

- Control Rig function library — graph enumeration + `AddFunctionToLibrary` + reusable sub-graphs.
- Full BT graph node authoring (BTGraph + runtime in lockstep).
- Decorator / service attachment.
- AnimBP state machines.
- Skeleton sockets / morph targets.

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
