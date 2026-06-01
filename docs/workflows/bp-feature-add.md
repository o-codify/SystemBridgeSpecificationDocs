---
id: workflow-blueprint-feature-add
title: "Workflow: Blueprint Feature Add"
status: stable
version: 26.601.1817
tags: [ workflow, blueprint ]
---

# Workflow: Blueprint Feature Add

Add a feature to a Blueprint headlessly. End-to-end example: "On Landed,
play a HardLanding montage" — touches nodes, exec wiring, object pin
defaults, variables, and PIE verification.

See: [blueprint authoring reference](../unreal/blueprint-authoring.md),
[PIE workflow](../unreal/pie-workflow.md),
[asset management](../unreal/asset-management.md).

## Preconditions

- Companion ≥ 1.3.4 (object reference setters).
- The target Blueprint exists.
- The target asset (montage / texture / class / …) exists.

## Steps

### 1. Discover the graph + nodes

```
unreal_bp_graphs_list bp_path="/Game/Blueprints/BP_PlayerChar"
  → {graphs: [{name:"EventGraph", kind:"Ubergraph", node_count:5},
              {name:"OnLanded",   kind:"Function",  node_count:1}, ...]}

unreal_bp_graph_nodes_list bp_path=... graph_name="OnLanded"
  → {nodes: [{node_guid:"...", class:"K2Node_FunctionEntry",
              title:"On Landed", x:0, y:0, pins:[...]}]}
```

If the event isn't yet overridden:

```
unreal_bp_overridable_functions bp_path=...
  → {functions: [
      {name:"ReceiveOnLanded", flavor:"BlueprintImplementableEvent", ...}, ...]}

unreal_bp_override_function bp_path=... function_name="ReceiveOnLanded"
  → {success: true, newly_created: true}
```

### 2. Create the Play Anim Montage call

```
unreal_bp_node_create bp_path=... graph_name="ReceiveOnLanded"
  node_class_path="/Script/BlueprintGraph.K2Node_CallFunction"
  x=600 y=120
  → {node_guid: "<new-guid>", pins: [...]}

unreal_bp_node_call_function_target bp_path=... graph_name=...
  node_guid="<new-guid>"
  function_owner_class_path="/Script/Engine.KismetSystemLibrary"
  function_name="PlayAnimMontage"
  → {success: true, pins_after: [...]}
```

### 3. Set the object reference on the AnimMontage pin

```
# Confirm the pin shape first if uncertain:
unreal_bp_node_pin_links bp_path=... graph_name=... node_guid="<new-guid>" pin_name="Anim Montage"
  → returns pin info; pin_category should be "object"

# Set it:
unreal_bp_node_pin_set_object bp_path=... graph_name=...
  node_guid="<new-guid>"
  pin_name="Anim Montage"
  object_path="/Game/Animations/Fall/AM_HardLanding_Forward"
  → {success: true, object_path: "..."}
```

If you didn't know the exact property/pin name, use:

```
# Find what the parent function (K2Node_FunctionEntry on OnLanded) outputs.
unreal_bp_graph_nodes_list bp_path=... graph_name="ReceiveOnLanded"
# Inspect a node's pin schema by class via run_python — or simpler, just
# create the node and read its returned pin list.
```

### 4. Wire exec

```
# The OnLanded entry node's "then" exec → the PlayAnimMontage call's "execute" pin.
unreal_bp_node_link_pins bp_path=... graph_name=...
  src_node_guid="<entry-guid>"   src_pin_name="then"
  dst_node_guid="<new-guid>"     dst_pin_name="execute"
```

### 5. Pass the In Skeletal Mesh Component too

`PlayAnimMontage` requires the target SkeletalMeshComponent. You can:

- Drag from `Self` → cast / Get Mesh / wire.
- Or use a `K2Node_VariableGet` for the Character's `Mesh` component.

```
unreal_bp_node_create ... node_class_path="/Script/BlueprintGraph.K2Node_VariableGet" x=400 y=200
unreal_bp_node_variable_target ... node_guid="<getter>" variable_name="Mesh"
unreal_bp_node_link_pins ...
  src_node_guid="<getter>" src_pin_name="Mesh"
  dst_node_guid="<new-guid>" dst_pin_name="In Skeletal Mesh Component"
```

### 6. Compile + save

```
unreal_bp_compile_and_save bp_path="/Game/Blueprints/BP_PlayerChar"
  → {success: true}
```

### 7. Verify

```
unreal_editor_save_all_dirty
unreal_pie_run_and_watch wait_seconds=15
  → status: "pie_running"
```

Trigger the scenario (jump + fall, or `pie_console_command "ke * Fall"`)
and observe via game logs / a temporary `print_string` node.

## Alternative: object literal node

If you'd rather have an explicit literal in the graph (useful when the
same asset feeds multiple pins):

```
unreal_bp_object_literal_create bp_path=... graph_name=...
  object_path="/Game/Animations/Fall/AM_HardLanding_Forward"
  x=400 y=200
  → {node_guid: "<literal>", output_pin: "Value"}

unreal_bp_node_link_pins ...
  src_node_guid="<literal>" src_pin_name="Value"
  dst_node_guid="<new-guid>" dst_pin_name="Anim Montage"
```

## Alternative: typed member variable

Cleaner pattern — variable + Get:

```
unreal_bp_variable_add_typed bp_path=...
  name="HardLandMontage"
  pin_category="object"
  sub_category_object="/Script/Engine.AnimMontage"
  default_object="/Game/Animations/Fall/AM_HardLanding_Forward"

unreal_bp_node_create ... node_class_path="/Script/BlueprintGraph.K2Node_VariableGet"
unreal_bp_node_variable_target ... variable_name="HardLandMontage"
unreal_bp_node_link_pins ... src_pin_name="HardLandMontage"  dst_pin_name="Anim Montage"
```

This survives moving the asset later, and other graphs can read the
variable too.

## Cross-references

- [blueprint authoring reference](../unreal/blueprint-authoring.md)
- [PIE workflow reference](../unreal/pie-workflow.md)
- [asset management](../unreal/asset-management.md) — for finding
  `/Script/...` and `/Game/...` paths.
- [unreal plugin tool catalog](../plugins/unreal.md#blueprints-read--write)
