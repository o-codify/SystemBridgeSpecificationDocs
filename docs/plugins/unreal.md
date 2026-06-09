---
id: plugin-unreal
title: "Plugin: unreal"
status: stable
version: 26.609.233
tags: [ plugin, unreal, ue ]
---

# Plugin: `unreal`

Largest plugin in the suite. Activates on `*.uproject` in cwd OR
`UnrealEditor.exe` running.

This page is the **tool catalog**. For end-to-end recipes and the
companion plugin's role, jump to the [unreal deep dive](../unreal/index.md).

See also: [plugins index](index.md), [overview](../README.md).

## Activation + scope

```
manifest.Triggers{
  FilesPresent: ["*.uproject"],
  Processes:    ["UnrealEditor.exe", "UE5Editor.exe", "UE4Editor.exe"],
}
```

The plugin auto-detects the project via cwd-rooted `FindUProject`. Every
helper-backed tool resolves the project context per call so multi-editor
setups work (a running AGLS editor + a running GameAnimationSample editor
each get their own Python Remote Execution route).

## Two execution paths

Every meaningful Unreal tool either:

1. **Runs inside the editor** via Python Remote Execution (UDP discovery + TCP
   exec). `unreal_run_python` is the explicit version; most other tools
   wrap a function in the embedded `sb_helpers.py`.
2. **Runs out-of-process** as a separate UE invocation: `project_build`,
   `project_run_tests`, `editor_restart` use UnrealBuildTool / `-nullrhi` /
   `taskkill` directly.

Tools that need C++ reach (`UEdGraphPin::DefaultObject`, `TObjectIterator`,
`FMessageLog`, `FBlueprintEditorUtils::AddFunctionGraph`, …) require the
[SystemBridgeCompanion sub-plugin](../unreal/companion.md).

## Tool catalog

### Project + Editor lifecycle

| Tool | Purpose |
|---|---|
| [`editor_status`](../unreal/index.md#editor_status) | Is the editor alive, reachable, loaded with cwd's project? Includes `last_crash`, `auto_recover` block. Always call this first in multi-step flows. |
| `editor_restart` | Save → quit → wait → relaunch. Auto-suppresses [crash watcher](../unreal/crash-recovery.md). Optional `skip_save_all_dirty` quits without persisting (use when the in-memory state is known bad). |
| `editor_launch` | Cold-start a closed editor: resolve engine binary from EngineAssociation, spawn detached with optional level + extra_args, poll until alive. Idempotent. The cold-start counterpart to `editor_restart`. |
| `editor_save_all_dirty` | Saves all dirty `/Game` assets. |
| `editor_load_level` | Open a different level by package path. |
| `editor_take_screenshot` | HighResShot via the viewport; returns a PNG path. |
| `editor_set_auto_recover` | Toggle the [persistent crash watcher](../unreal/crash-recovery.md) with optional TTL. Use BEFORE intentional kill cycles. |

### Logs + diagnostics

| Tool | Purpose |
|---|---|
| `read_log` | Chronological `<Project>.log` lines, optionally filtered by severity. |
| `editor_message_log` | Entries from one of UE's FMessageLog categories (LoadErrors / MapCheck / AssetCheck / BlueprintLog / PIE / …). Separate from the chronological log. Requires Companion. See [message log reference](../unreal/message-log.md). |
| `editor_message_log_categories` | Summary counts per populated category. |

### Python remote execution

| Tool | Purpose |
|---|---|
| `run_python` | Escape hatch. Sends an arbitrary Python script to the running editor via Remote Execution. Returns parsed `<<<SB_JSON>>>` blocks (see [protocol](#sb_json-marker-protocol)). Prepends `sb_helpers.py` so any helper is callable inline. |

### Asset / Content browser

| Tool | Purpose |
|---|---|
| `assets_query` | AssetRegistry query — returns `[{name, package, class}, …]`. Fast, no asset loads. |
| `asset_describe` | All editor properties on a UObject with type + current value. Use BEFORE `asset_set_properties` to learn real UPROPERTY names. |
| `asset_set_properties` | Bulk-set top-level UObject properties. Persist + verify. |
| `asset_dependencies` | Hard + soft deps of an asset, bucketed by `/Game`/`/Script`/`/Engine`. |
| `asset_referencers` | Who depends on this asset (hard + soft). Run before delete/rename. |
| `asset_rename` | Rename / move; UE updates references automatically. |
| `asset_duplicate` | Duplicate to a new path. Idempotent. |
| `asset_delete` | Delete; refuses by default if referenced. |
| `read_uasset` | Parse the `FPackageFileSummary` header — engine version, class names, etc. — without loading the asset. |
| `dt_row_add` | **v1.5+** Add a row to a UDataTable. Optional `from_row` clones an existing row byte-for-byte (preserves parens-named UDS fields the stock fill APIs corrupt). |
| `dt_row_set_field` | **v1.5+** Set a single (possibly nested) field on a DataTable row. Dotted path; FProperty::ImportText on the leaf — NOT struct-text. Works with names containing `()`. |
| `package_discard_changes` | **v1.5+** Drop pending in-memory changes on a package — reload from disk. Headless equivalent of Content Browser's Revert. |

### Levels + actors

| Tool | Purpose |
|---|---|
| `level_actor_list` | Actors in the current level with stable Outliner labels, classes, transforms. |
| `level_actor_spawn` | Spawn with a stable Outliner label. |
| `level_actor_set_property` | Find by label, set one editor property; persist + verify. |
| `level_actor_destroy` | Find by label and destroy. |

### Blueprints (read + write)

See [blueprint authoring](../unreal/blueprint-authoring.md) for the canonical
end-to-end flow.

| Tool | Companion? | Purpose |
|---|---|---|
| `bp_get_info` | no | Comprehensive structure: generated class, components, CDO sample. |
| `bp_components_list` | no | SCS components with `var_name` + class. |
| `bp_set_component_property` | no | Set a property on an SCS-template component (uses SubobjectDataSubsystem). **String-only** — see `bp_set_component_property_typed` (v1.12) for numerics / structs / asset paths. |
| `bp_compile_and_save` | no | Compile + save. Required after edits. |
| `bp_variable_add` | no | Add a primitive member variable. Legacy: non-primitive types silently became Int. |
| `bp_variable_add_typed` | **v1.3.4+** | Add a member variable with FULL type control (object, class, soft_object, soft_class, interface, struct, enum, all primitives, all containers). Optional `default_object` writes the CDO. |
| `bp_variable_remove` / `bp_variable_rename` | no | Member variable lifecycle. |
| `bp_function_graph_add` / `bp_function_graph_remove` / `bp_function_graph_rename` | no | Fresh function graphs. |
| `bp_overridable_functions` | **v1.3+** | List parent functions that can be overridden — equivalent to the editor's "Override Function" dropdown. |
| `bp_override_function` | **v1.3+** | Create an override graph for a parent function. Wraps `FBlueprintEditorUtils::AddFunctionGraph<UFunction>` so K2Node_FunctionEntry binds to the parent (Parent: super calls work). |
| `bp_graphs_list` / `bp_graph_lookup` / `bp_graph_nodes_list` | **v1.0+** | Enumerate graphs and nodes (UE 5.7 Python can't — `UEdGraph.Nodes` is protected). |
| `bp_node_create` | **v1.0+** | Create a K2Node on a graph at (x, y) given a UClass path. |
| `bp_node_link_pins` / `bp_node_break_link` / `bp_node_pin_links` | **v1.0+** | Pin wiring + introspection. |
| `bp_node_remove` / `bp_node_set_position` / `bp_node_reconstruct` | **v1.0+** | Node lifecycle. |
| `bp_node_pin_set_default` | **v1.0+ (smart routing v1.3.4+)** | Set a pin's default — auto-routes object/asset paths to TrySetDefaultObject. |
| `bp_node_pin_set_object` | **v1.3.4+** | Explicit object-default setter — see [blueprint authoring](../unreal/blueprint-authoring.md#object-references-on-pins). |
| `bp_object_literal_create` | **v1.3.4+** | K2Node_Literal node carrying an object reference. |
| `bp_node_call_function_target` | **v1.0+** | Configure a `K2Node_CallFunction` to call a specific UFUNCTION. |
| `bp_node_variable_target` | **v1.0+** | Configure a `K2Node_VariableGet/Set` to reference a member variable. |
| `bp_node_dynamic_cast_target` | **v1.0+** | Configure a `K2Node_DynamicCast` to cast to a target UClass. |
| `bp_node_struct_target` | **v1.0+** | Configure `K2Node_MakeStruct`/`BreakStruct` for a specific `UScriptStruct`. |
| `bp_node_switch_enum_target` | **v1.0+** | Configure `K2Node_SwitchEnum` for a specific `UEnum`. |
| `bp_node_get_subsystem_target` | **v1.0+** | Configure `K2Node_GetSubsystem`. |
| `bp_node_input_action_target` | **v1.0+** | Configure `K2Node_EnhancedInputAction`. |
| `bp_event_node_add` / `bp_event_find` / `bp_custom_event_add` | **v1.0+** | Event-graph entry points (ReceiveBeginPlay / Tick / custom). |
| `bp_custom_event_configure` | **v1.4+** | Configure replication on a K2Node_CustomEvent — net_mode (multicast / server / client / notreplicated), reliable, call_in_editor. The editor's right-click "Replicates" dropdown headlessly. |
| `bp_event_add_param` | **v1.4+** | Add a typed output parameter to a custom event / function entry. Same pin_category taxonomy as `bp_variable_add_typed`. |
| `bp_variable_set_replication` | **v1.4+** | Set replication on a member variable — none / replicated / repnotify. Auto-creates `OnRep_<var>` function graph for repnotify. |

### PIE (Play In Editor)

See [PIE workflow](../unreal/pie-workflow.md).

| Tool | Companion? | Purpose |
|---|---|---|
| `pie_start` | optional | Real PIE with possession via Companion v1.1+; falls back to Simulate without. Includes pre-flight for Blueprint compile errors (Companion v1.3.4+). |
| `pie_stop` | no | Stop the active PIE session. |
| `pie_possess` | optional | F8 equivalent — flip Play ↔ Simulate. |
| `pie_run_and_watch` | optional | Start PIE, watch for N seconds, return `pie_running` / `pie_exited_clean` / `crashed` / `editor_crashed` with log excerpt on failure. |
| `pie_console_command` | no | Issue a console command into the PIE world (`stat fps`, `showdebug abilitysystem`, …). |

### Build + test (out-of-process)

| Tool | Purpose |
|---|---|
| `project_build` | UnrealBuildTool. Structured errors (`{file, line, col?, code, message}`) so the agent can act per-line. Works without editor running. |
| `project_run_tests` | UE Automation tests via `UnrealEditor.exe -nullrhi`. Parses `LogAutomationController` into `[{name, status}]`. |
| `live_coding_compile` | Synchronous incremental C++ compile (Ctrl+Alt+F11 equivalent). Triggers, polls log for `Live coding succeeded` / `failed`, returns parsed errors + warnings. See [Live Coding reference](../unreal/live-coding.md). |

### Companion lifecycle

| Tool | Purpose |
|---|---|
| `companion_status` | Whether the C++ sub-plugin is installed/built/loaded; detects version drift. |
| `companion_install` | Install the companion (`scope=project|engine`); requires C++ toolchain. |
| `companion_uninstall` | Reverse. For project scope: also removes `.uplugin` entry + gitignore line. |
| `companion_rebuild` | Force clean rebuild — after sb-unreal upgrades. |

### Materials

| Tool | Purpose |
|---|---|
| `material_instance_set_scalar` | Scalar parameter on a Material Instance Constant; auto-saves + verifies. |
| `material_instance_set_vector` | Vector (FLinearColor) parameter on an MIC. |
| `material_expressions_list` | All `UMaterialExpression` on a Material with name + class + position + inputs/outputs. Requires Companion. |
| `material_expression_add` | Spawn a new `UMaterialExpression`. UE 5.5+ (uses `ExpressionCollection`). |
| `material_expression_connect` | Wire one expression's output to another's input. |
| `material_expression_connect_property` | Wire an expression's output to one of the material's main outputs (BaseColor / Metallic / Roughness / …). |
| `material_expression_remove` | Delete an expression. Idempotent. |
| `material_compile` | Recompile + save after edits. |

### UMG (Widget Blueprints)

| Tool | Purpose |
|---|---|
| `widget_tree_list` | Every `UWidget` in a `UWidgetBlueprint`'s WidgetTree. |
| `widget_add` | Add a widget; empty `parent_name` installs as root. |
| `widget_remove` / `widget_rename` | Lifecycle. |
| `widget_set_slot_property` | Set a property on a widget's parent slot (CanvasPanelSlot.Anchors, OverlaySlot.Padding, GridSlot.Row, …) using UE serialization-form values. |

### Behavior Trees + Blackboard

| Tool | Purpose |
|---|---|
| `bt_create_pair` | Scaffold a BehaviorTree + Blackboard asset pair via factories. |
| `bt_info` | Read BT metadata — linked Blackboard. |
| `bt_nodes_list` | Every node in the BT's runtime tree with parent + child_index. |
| `bt_root_set` | Set the root composite (Selector / Sequence / Parallel). |
| `bt_node_add` / `bt_node_remove` | Children. |
| `bb_info` | Blackboard metadata. |
| `bb_keys_list` | Every key with name + type + flags. |
| `bb_key_add` / `bb_key_remove` | Key lifecycle. |

### Input Mapping Contexts

| Tool | Purpose |
|---|---|
| `imc_mapping_list` | List all `EnhancedActionKeyMapping`. |
| `imc_mapping_add` | Add one (idempotent — skips if action+key already mapped). |

### Niagara

| Tool | Purpose |
|---|---|
| `niagara_system_info` | Metadata + emitter list. |
| `niagara_set_user_parameter` | Set a user-exposed parameter (float / int / bool). Best-effort. |

### Sequences + animation

| Tool | Purpose |
|---|---|
| `sequence_info` | LevelSequence metadata. |
| `anim_sequence_info` | AnimSequence/Montage metadata. |
| `anim_blueprint_graphs` | Animation graphs (state machines, sub-graphs) in an AnimBlueprint. |
| `anim_blueprint_nodes` | Anim nodes filtered by class; includes referenced anim assets. |
| `anim_blueprint_set_node_asset_override` | Override the anim asset on a SequencePlayer / BlendSpacePlayer node. |
| `anim_montage_create_from_template` | **v1.6+** Create a new UAnimMontage by duplicating a template (preserves slot layout, group, sections, blends, notifies) and replacing the AnimSequence in every segment. Headless authoring of upper-body / additive-slot montages. |
| `anim_montage_add_notify` | **v1.7+** Add a skeleton/named FAnimNotifyEvent at a chosen time + optional notify-state duration. Resolves or creates the NotifyTrack; registers the name on the skeleton; idempotent on name+time+track. |
| `anim_montage_remove_notify_by_name` | **v1.7+** Remove every FAnimNotifyEvent on the montage with the given name. Returns count_removed. |
| `bp_create` / `dataasset_create` / `struct_create` / `enum_create` | pure Python | Create a new BP / DataAsset instance / UserDefinedStruct / UserDefinedEnum asset via the matching Factory. |
| `bp_variable_set_default` | **v1.8+** | Set the default of an existing BP member variable on the CDO. value_kind: auto / string / object / class / soft_object / soft_class / gameplaytag. |
| `bp_variables_list` | **v1.8+** | List a Blueprint's member variables with type info. NewVariables is protected in UE 5.7 Python. |
| `dt_row_get` | **v1.8+** | Read a DataTable row as a flat {field_path: value} JSON map. Inverse of dt_row_set_field. |
| `dt_export` | pure Python | Export a whole DataTable as JSON or CSV. |
| `gameplaytag_add` / `add_many` / `remove` / `list` / `refresh` | **v1.8+** | Headless GameplayTag CRUD — write to source .ini AND register in-memory so the tag is usable immediately (no editor restart). |
| `struct_member_add` / `remove` / `rename` / `members_list` | **v1.8+** | UserDefinedStruct member authoring via FStructureEditorUtils. Same pin_category taxonomy as bp_variable_add_typed. |
| `enum_entry_add` / `set_display_name` / `remove` | **v1.8+** | UserDefinedEnum entry authoring via FEnumEditorUtils. |
| `enum_entries_list` | pure Python | List a UserDefinedEnum's entries. |

### Control Rig (RigVM)

| Tool | Companion | Purpose |
|---|---|---|
| `control_rig_create` | pure Python | Create a new ControlRigBlueprint via `ControlRigBlueprintFactory`. Optional `preview_skeletal_mesh`. Idempotent. |
| `control_rig_graphs_list` | **v1.9+** | List rig graphs. v1.9 reports the main RigGraph; function-library graphs land in v1.10. |
| `control_rig_nodes_list` | **v1.9+** | List nodes with name / title / RigUnit struct path / position / pin count. |
| `control_rig_node_add` | **v1.9+** | Add a RigUnit node by `UScriptStruct` path. Returns the new node's name. |
| `control_rig_node_remove` | **v1.9+** | Remove a node by name. Idempotent. |
| `control_rig_pin_set_default` | **v1.9+** | Set a pin default. `pin_path` is dot-notation (`Item.Type`, `Translation.X`); value uses RigVM's text serializer. |
| `control_rig_add_link` | **v1.9+** | Add an exec/data link between two pins (`PinName` or `Pin.SubPin`). |
| `control_rig_compile` | **v1.9+** | RecompileVM + save. |
| `control_rig_variable_add` | **v1.10+** | Add a rig member variable. Public (direction=input/output) variables become input pins on the AnimGraph Control Rig node. Mirrors `bp_variable_add_typed` taxonomy. |
| `control_rig_variables_list` | **v1.10+** | List rig member variables (name / cpp_type / cpp_type_object / direction / default_value). |
| `control_rig_variable_remove` | **v1.10+** | Remove a rig variable by name. Idempotent. |
| `control_rig_variable_get_node_add` / `set_node_add` | **v1.10+** | Add a Get/Set variable node to a rig graph; reads CPPType + CPPTypeObject from the variable definition. |

### AnimGraph (v1.11)

| Tool | Companion | Purpose |
|---|---|---|
| `anim_node_add` | **v1.11+** | Add a `UAnimGraphNode_*` subclass by class path. Returns `node_guid`. |
| `anim_node_set_inner_property` | **v1.11+** | Write a UPROPERTY on the inner `FAnimNode_*` runtime struct; node reconstructs so class-driven pin sets update. |
| `anim_node_expose_pin` | **v1.11+** | Toggle entries in `ShowPinForProperties` (and `CustomPinProperties` since v1.11.1 — where Control Rig / Linked Anim Layer expose bindable variables). Reconstructs. |
| `anim_node_list_exposable_pins` | **v1.11.1+** | Enumerate both pin lists with `source` tag (`show` / `custom`) — the discovery surface for `anim_node_expose_pin`. |
| `anim_node_info` | **v1.11+** | Read class, inner struct, position, pins, exposed-pin properties. |
| `anim_node_remove` | **v1.11+** | Remove by guid (routes through `bp_node_remove` — gets the v1.10.2 variable-snapshot guard). |

Linking pose AND data pins reuses `bp_node_link_pins` / `bp_node_break_link` / `bp_node_pin_links`.

### Transforms (sockets, bones, actors)

| Tool | Companion | Purpose |
|---|---|---|
| `mesh_sockets_list` | pure Python | Sockets on SkeletalMesh / StaticMesh (name, parent_bone, relative xform). v1.12 switched to the public `num_sockets()` / `get_socket_by_index()` API — v1.11 hit the protected `Sockets` UPROPERTY and dumped a ~270KB traceback. |
| `mesh_socket_transform` | pure Python | A SkeletalMesh socket's relative xform + parent bone in component space. |
| `mesh_socket_add` | **v1.12+** | Create or update a SkeletalMesh socket headlessly — sets `parent_bone`, which stock UE 5.7 Python can't (`VisibleAnywhere + BlueprintReadOnly`). Idempotent on socket name. Optional `add_to_skeleton`. |
| `skeleton_bones_list` | pure Python | Bones with parent_index / parent_name. v1.12 walks `SkeletalMeshEditorSubsystem.get_bone_tree` first — returned empty on some 5.7 meshes via the old path. |
| `skeleton_bone_transform` | pure Python | A bone's reference-pose transform in `bone` or `component` space. |
| `actor_transform_query` | pure Python | Live world transform; optional component / bone / socket / relative_to. |

### BP variable lifecycle (v1.12 + v1.13.4 bug-fix)

Same public tools as v1.12, with v1.13.4 repairing the broken-type
case: a variable whose `FEdGraphPinType.PinSubCategoryObject` points
at a deleted `UEnum` / `UScriptStruct` / `UClass` used to leave its
Blueprint stuck in `BS_Error` even after `bp_variable_remove_direct`,
because the orphan `K2Node_VariableGet/Set` nodes still referenced
the now-missing var by name. v1.13.4 sweeps those nodes by name
across all four graph kinds and reports the actual post-compile
status. See [companion v1.13.4](../unreal/companion.md#version-timeline)
for the C++ details.

| Tool | Companion | Purpose |
|---|---|---|
| `bp_variable_remove_direct` | **v1.12+ (fixed v1.13.4)** | Surgical remove via `FBlueprintEditorUtils::RemoveMemberVariable`. No collateral sweep of unrelated vars (unlike `bp_variable_remove`'s `remove_unused_variables` path). v1.13.4 added an orphan-`K2Node_VariableGet/Set`-by-name sweep across Ubergraphs / Function / Macro / Delegate graphs so the BP actually compiles clean afterwards. Returns `nodes_swept` + `blueprint_status` (`up_to_date` / `error` / …). Works on broken-typed vars (deleted-enum etc.) where the orphan nodes would otherwise pin the BP to `BS_Error`. |
| `bp_variable_rename_atomic` | **v1.12+** | Atomic rename via `FBlueprintEditorUtils::RenameMemberVariable`. Preserves type + metadata; does NOT leave the old var behind (the way `bp_variable_rename` does). |
| `bp_variable_retype` | **v1.12+ (fixed v1.13.4)** | In-place retype via `FBlueprintEditorUtils::ChangeMemberVariableType`. Same `pin_category` vocab as `bp_variable_add_typed`. v1.13.4 added broken-`PinSubCategoryObject` repair: sanitizes `NewVariables[i].VarType` to a Bool placeholder when the existing type needs a subobject but has none, retypes, then reconstructs every `K2Node_Variable` referencing the var so pin sets resolve against the new definition. Returns `blueprint_status` for end-to-end verification. |

### Runtime invoke + PIE input (v1.12)

| Tool | Companion | Purpose |
|---|---|---|
| `runtime_invoke` | **v1.12+** | Call a BP-exposed event/function on a live PIE/level object. Target via `player_index` (controlled pawn) or `actor_label` (display label) + optional `component_name`. `args` is a flat list of strings in `ImportText` format — one per declared parameter, in order. Returns the function's return value exported as text. |
| `pie_input_inject` | **v1.12+** | Deliver input to PIE. Pass `action_path` (Enhanced Input `UInputAction`) OR `key_name` (raw FKey). `event` ∈ {pressed, released, repeat, doubleclick} for keys. `amount` is scalar value / pressure. `value_vector` covers Axis2D/Axis3D actions. Routes to a chosen `player_index`. |

### Inspection (single node)

| Tool | Companion | Purpose |
|---|---|---|
| `bp_node_inspect_by_guid` | **v1.12+** | Return ONE node's full info (class, title, position, pins with `linked_count` + types) instead of dumping the whole graph through `bp_graph_nodes_list`. Use it for wiring in 600+ node event graphs that exceed the tool output limit. |

### SCS component template (v1.12)

| Tool | Companion | Purpose |
|---|---|---|
| `bp_set_component_property_typed` | **v1.12+** | Typed SCS template setter — writes to the COMPONENT TEMPLATE so spawned instances inherit (not just the CDO). Accepts numerics, booleans, `FVector` / `FRotator` / `FTransform` / `FLinearColor` literals, enum names, asset paths for object refs. v1.12.1: `ClassProperty` (e.g. `anim_class`) now resolves correctly (tries `_C` suffix + `LoadObject->GeneratedClass`). Replaces `bp_set_component_property` which treats `value` as a string. |
| `bp_component_remove` | **v1.12.1+** | Remove a component from a Blueprint's SCS. Symmetric counterpart to `bp_component_add` (`SubobjectDataSubsystem.delete_subobject` + compile + save). Idempotent. |
| `anim_reassign_asset` | **v1.12.1+** | By-value asset reassignment in an AnimBP: walks every `AnimGraphNode_*` whose inner `FAnimNode_*` references `from_asset` and rewrites to `to_asset`. Optional `graph_filter` substring scopes to one state-machine graph. Returns per-node trail. |
| `asset_version_info` | **v1.12.2+** | Saved engine version + custom-version container + warnings for an asset. Diagnoses 'why won't this load' failures (engine-skew, custom-version mismatch). |
| `asset_metadata` | **v1.12.2+** | Per-object metadata key-value pairs for the asset + its immediate subobjects (`UMetaData::GetMapForObject`). Author-time hints, tooltips, categories. |
| `bp_function_bytecode` | **v1.12.3+** | Line-by-line Kismet bytecode disassembly for one BP function. v1.12.5 decodes operands (FProperty names, UFunction names, typed constants, casts, jumps, switch cases). Diagnoses 'graph compiles but runtime is wrong' — self-context bugs become 'EX_LocalOutVariable without preceding EX_Self'. |

### Anim state-machine pose + transition rule (v1.13.2 + v1.13.3 bug-fix)

Completes the v1.13.0 SM authoring — the always-true transitions and
empty states are no longer the only option. v1.13.3 fixed three bugs
shaken out in live ALS overlay-SM use: a compile-step `TypeError` that
hid successful edits behind `success:false`, an `enum_value` resolver
that didn't recognize UserDefinedEnum display names, and — critically —
a silently-refused enum-byte schema link that left the rule evaluating
to literal-vs-literal 0==0 (state was unreachable even with the
"right" rule). See [companion v1.13.3 row](../unreal/companion.md#version-timeline)
for the full root-cause + fix writeup.

| Tool | Companion | Purpose |
|---|---|---|
| `anim_state_machine_set_state_pose` | **v1.13.2+ (fixed v1.13.3)** | Populate a state's inner anim graph with a SequencePlayer wired to the OutputAnimGraphNode Result pin. Replaces any existing anim node. Idempotent. v1.13.3: compile step now uses the UBlueprint (not its EdGraph outer), exception repr truncated to 200 chars with `edit_applied:true` instead of echoing the 381 KB helper source. |
| `anim_state_machine_set_transition_rule_enum_equals` | **v1.13.2+ (fixed v1.13.3)** | Set a transition's boolean rule to `<variable> == <enum>::<value>` (or `!=` if negate). Rebuilds the transition's BoundGraph from scratch — VarGet → EqualEqual_ByteByte ← ByteLiteral → TransitionResult. v1.13.3: accepts enum value by display name / `NewEnumeratorN` / numeric; stamps `PinSubCategoryObject = EnumObj` on EqualEqual's byte inputs so the K2 schema actually accepts the variable→byte link (without this it silently degraded to literal-vs-literal). |
| `anim_state_machine_get_transition_rule` | **v1.13.3+** | Read back a transition's rule expression as a pipe-separated descriptor: `{rule_text \| has_var_get \| has_compare \| compare_fn \| enum_path \| enum_value_index \| negated \| wired_to_result}`. Use to verify that `set_transition_rule_enum_equals` actually wired up — debugging-blind was the v1.13.2 pain point. |

### Macros / interfaces / function locals (v1.13.1)

| Tool | Companion | Purpose |
|---|---|---|
| `bp_macros_list` | **v1.13.1+** | List macro graphs defined on a Blueprint. |
| `bp_macro_create` | **v1.13.1+** | Create a new macro graph (`FBlueprintEditorUtils::AddMacroGraph`). |
| `bp_node_macro_instance_target` | **v1.13.1+** | Wire a `K2Node_MacroInstance` to a named macro on the same BP or an external BPL. |
| `bp_interfaces_list` | **v1.13.1+** | List interfaces implemented by a Blueprint. |
| `bp_interface_add` | **v1.13.1+** | Implement an interface (`FBlueprintEditorUtils::ImplementNewInterface`). Idempotent; auto `_C` resolution. |
| `bp_interface_remove` | **v1.13.1+** | Symmetric remove. |
| `bp_function_locals_list` | **v1.13.1+** | List local variables on a function graph (walks `UK2Node_FunctionEntry::LocalVariables`). |
| `bp_function_local_add` | **v1.13.1+** | Add a typed local variable. Same pin_category vocab as `bp_variable_add_typed`. |

### AnimGraph state machines (v1.13.0)

| Tool | Companion | Purpose |
|---|---|---|
| `anim_state_machine_states_list` | **v1.13.0+** | List states with `{guid, name, kind, asset_path, out_transitions}`. kind ∈ {state, conduit, entry, transition}. |
| `anim_state_machine_add_state` | **v1.13.0+** | Add a UAnimStateNode. Returns guid. Optional state_name renames the BoundGraph. |
| `anim_state_machine_add_conduit` | **v1.13.0+** | Add a UAnimStateConduitNode (transition routing). |
| `anim_state_machine_add_transition` | **v1.13.0+** | Add a transition between two states / conduits. Auto-wires From→Trans→To. Transition rule (boolean) authoring deferred. |
| `anim_state_machine_entry_state` | **v1.13.0+** | Get the guid of the entry state. |

### Sequencer (Level Sequence) authoring (v1.14.0)

Tracks-as-data, sections-as-data, keys-as-data. Pure Python via UE 5.7's
`MovieSceneScriptingChannel` API — no companion required. Mirror coverage
to how `bp_node_*` covers Blueprints.

| Tool | Companion | Purpose |
|---|---|---|
| `sequencer_bindings_list` | none | List object bindings (possessables + spawnables) with class + track_count. Discovery surface — guids feed every other tool. |
| `sequencer_tracks_list` | none | List tracks on a binding (or master tracks if `binding_guid` empty). Track `name` is the value to pass back as `track_guid`. |
| `sequencer_track_info` | none | List sections on a track with each section's channel inventory (name + type + key_count). Section index is the section_guid. |
| `sequencer_keys_list` | none | List keys on a section, optionally restricted to one channel. Each key: {channel, type, time, value, interpolation}. |
| `sequencer_track_add` | none | Add a track to a binding (or master). Vocab: `transform / float / bool / integer / byte / vector / color / event / audio / camera_cut / anim / subscene / actor_ref` or a raw `MovieScene*Track` class name. |
| `sequencer_track_remove` | none | Remove a track by name. Idempotent. |
| `sequencer_section_add` | none | Add a section to a track with `[start_frame, end_frame)`. Frames are tick-resolution (see `sequence_info`). Returns `section_index` to pass back as `section_guid`. |
| `sequencer_key_add` | none | Write a key on a section's channel. Channel FNames vary by track — discover via `sequencer_track_info`. Common: `Location.X/Y/Z`, `Rotation.X/Y/Z`, `Value`, `R/G/B/A`. `interpolation` ∈ {auto, user, break, linear, constant}. |
| `sequencer_key_remove` | none | Remove every key at the given time on a channel. Idempotent. |
| `sequencer_binding_add` | none | Bind an existing level actor as a possessable. `actor_path` accepts label or full object path. Returns `binding_guid`. The actor must already exist in the active level. |

### Level actor attach + material connections (v1.15.0)

Pure Python via `EditorActorSubsystem.attach_to_actor` and
`MaterialEditingLibrary.get_input_connection` — no companion bump.

| Tool | Companion | Purpose |
|---|---|---|
| `level_actor_attach` | none | Attach actor to parent (optionally to a named socket). `rule` ∈ snap / keep_world / keep_relative. Idempotent. |
| `level_actor_detach` | none | Detach from current parent. `rule` ∈ keep_world / keep_relative. |
| `level_actor_attachments_list` | none | `{parent: {label, socket} \| null, children: [{label, socket}]}`. |
| `material_expression_pin_links` | none | Connections on one pin: `[{other_guid, other_pin, direction}]`. Mirror of `bp_node_pin_links`. |
| `material_connections_list` | none | Every edge in the material graph: `[{from_guid, from_pin, to_guid, to_pin}]`. Audit / diff / refactor. |

### Movie Render Queue / World Partition / Niagara / Physics (v1.16.0)

Pure Python; each tool degrades with a structured `*_unavailable` reason
if its subsystem isn't bound in this engine build.

| Tool | Companion | Purpose |
|---|---|---|
| `mrq_preset_create` | none | `UMoviePipelineMasterConfig` asset. Vocab: output_format ∈ png/jpg/bmp/exr/mp4, resolution WIDTHxHEIGHT, sample_count, output_dir. |
| `mrq_render_submit` | none | Enqueue + start MRQ render of a LevelSequence with a preset via `MoviePipelinePIEExecutor` (in-process). |
| `mrq_queue_list` | none | Jobs currently in the MRQ. |
| `world_partition_cells_list` | none | Runtime cells on WP-enabled maps. Returns `not_wp_enabled` typed error otherwise. |
| `niagara_emitter_list` | none | Emitters on a NiagaraSystem with name + enabled. Module-stack mutation requires companion C++. |
| `physics_constraint_list` | none | `UPhysicsConstraintComponent`s on an actor with their bound actors. |
| `physics_constraint_actor_spawn` | none | Spawn a `PhysicsConstraintActor`. Configure ConstraintActor1/2 via `level_actor_set_property`; deep profile (per-axis limits, motors) requires companion C++. |

### Bulk offline scanner (no editor)

| Tool | Companion | Purpose |
|---|---|---|
| `assets_find_references` | none | List every asset whose import table references `target_path`. Walks /Game/** uassets on disk. 1000-asset project in 1-3s. Refactor planning. |
| `assets_find_by_class` | none | List every asset whose top-level export class matches any candidate (short name or full path). "all DataTables", "all AnimBPs" audits. |
| `assets_scan_offline` | none | Project-wide audit: class histogram + engine-family histogram + top 20 referenced packages. Diagnose project structure. |
| `assets_outdated` | none | List assets saved with an older UE engine version. Useful after a UE upgrade — flags files that need re-saving. Threshold default 5.7 (file_version_ue5 = 1009). |

## SB_JSON marker protocol

Helpers and `run_python` return structured data by printing a marker block:

```python
print('<<<SB_JSON>>>')
print(json.dumps({...}))
print('<<<END_SB_JSON>>>')
```

The Go side scrapes the first parseable block and surfaces it as the tool's
JSON result. Side-channel raw stdout / stderr is also returned but capped
(see next section).

## Output pruning

UE Python Remote Execution returns every log line as an Output event;
scripts that touch broad APIs (`dir(unreal)`) can produce 100KB+ of noise.
The handler prunes:

- When `parsed_json` is present (the deliverable): drop Info entries
  entirely, keep up to 50 Warning/Error entries.
- When `parsed_json` is absent: keep last 50 entries of any type.
- Per entry: 4KB cap with `...[truncated N chars]` marker.
- `Stderr`: 4KB cap.
- The original `Command` echo (~150KB on helper-prepended scripts) is always
  blanked.

`output_dropped` + `output_note` describe what was trimmed.

## Discover summary

```
unreal: {
  project_present: bool,
  project_name: "...",
  uproject_path: "...",
  engine_version: "5.7",
  modules: [...],
  plugins_enabled: [...],
  log_path: "...",
  log_size: int,
  log_mtime: "RFC3339",
  log_tailer_active: bool,
  events_since_last_call: [
    {kind: "unreal_log_warning", category, message, frame, raw},
    {kind: "unreal_log_error",   ...},
    {kind: "unreal_editor_crashed", crash_dir, summary, recovered, ...},
  ]
}
```

## Cross-references

- [unreal deep dive](../unreal/index.md) — overview of everything UE-specific.
- [companion plugin](../unreal/companion.md) — what each version adds.
- [blueprint authoring](../unreal/blueprint-authoring.md) — full BP editing flow.
- [PIE workflow](../unreal/pie-workflow.md).
- [Live Coding](../unreal/live-coding.md).
- [crash recovery](../unreal/crash-recovery.md).
- [message log](../unreal/message-log.md).
