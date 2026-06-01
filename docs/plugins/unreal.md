---
id: plugin-unreal
title: "Plugin: unreal"
status: stable
version: 26.601.2308
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
| `bp_set_component_property` | no | Set a property on an SCS-template component (uses SubobjectDataSubsystem). |
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
