---
id: blueprint-authoring-headless
title: Blueprint Authoring (headless)
status: draft
version: 26.601.1502
tags:
  - unreal
  - blueprint
  - authoring
---

# Blueprint Authoring (headless)

End-to-end editing of UBlueprints without touching the editor UI. Requires
[SystemBridgeCompanion](companion.md) v1.0+ for most operations; specific
features are gated on later versions and called out per section.

See also: [unreal plugin reference](../plugins/unreal.md),
[unreal deep dive](index.md).

## Why headless is hard in UE

UE's stock Python (5.7) is read-mostly:

- `UBlueprint.FunctionGraphs` / `MacroGraphs` / `UbergraphPages` are
  public-but-unbound `TArray<UEdGraph*>` — you can't enumerate graphs.
- `UEdGraph.Nodes` is protected.
- `UEdGraphPin::DefaultObject` is a `UObject*` field — Python's
  `set_editor_property("default_value", "...")` writes a string, not the
  object slot.
- `FBlueprintEditorUtils::AddFunctionGraph` (the editor's "Override
  Function" entry point) isn't exposed.
- `K2Node_Literal::SetObjectRef`, `KismetEditorUtilities`, … same.

Companion exposes each through a static UFUNCTION.

## Reading: enumerate, inspect

| Goal | Tool | Companion? |
|---|---|---|
| List all graphs on a BP | [`bp_graphs_list`](../plugins/unreal.md#blueprints-read--write) | v1.0+ |
| Look up a graph by name | `bp_graph_lookup` | v1.0+ |
| List nodes on a graph | `bp_graph_nodes_list` | v1.0+ |
| List pins on a node | (returned with the node list) | v1.0+ |
| Show what pins link to | `bp_node_pin_links` | v1.0+ |
| Components on a BP | `bp_components_list` | no |
| Comprehensive BP shape | `bp_get_info` | no |

`bp_graph_nodes_list` returns each node with stable `node_guid`, class path,
title, position, and pin metadata. The GUID is the only valid handle for
follow-up edits — names aren't stable.

## Member variables

### Primitives — `bp_variable_add`

Works for `Real / Int / Bool / String / Name / Vector / Rotator / Transform`.

```
bp_variable_add(
  bp_path="/Game/Blueprints/BP_Foo",
  name="Health",
  type="Real",
  default="100.0",
)
```

### Object / Class / Struct / Enum / containers — `bp_variable_add_typed`

**v1.3.4+**. The legacy `bp_variable_add` silently turned non-primitive
`type` into IntProperty. `bp_variable_add_typed` builds a proper
`FEdGraphPinType` via Companion:

```
bp_variable_add_typed(
  bp_path,
  name="HardLandMontage",
  pin_category="object",
  sub_category_object="/Script/Engine.AnimMontage",
  default_object="/Game/Animations/Fall/AM_HardLanding_Forward",
)
```

`pin_category`:

| Tag | Meaning |
|---|---|
| `object` / `class` / `soft_object` / `soft_class` / `interface` | UObject-ish references |
| `struct` | `UScriptStruct` (FVector, FRotator, custom USTRUCT) |
| `enum` (or `byte`) | `UEnum` |
| `bool` / `int` / `int64` / `real` / `string` / `name` / `text` | primitives |

`sub_category_object` is REQUIRED for non-primitives:

- For `object` / `class` / `interface`: full path of the UClass.
  E.g. `/Script/Engine.AnimMontage`, `/Script/Engine.Texture2D`,
  `/Game/Blueprints/BP_Survivor.BP_Survivor_C` (note `_C` for BP class
  references).
- For `struct`: `/Script/CoreUObject.Vector`, `/Game/Data/F_MyStruct`.
- For `enum`: `/Game/Data/E_PlayerState`.

`container`: `single` / `array` / `set` / `map`. Default single.

`default_object` (optional): asset path loaded into the BP CDO via
`FObjectPropertyBase::SetObjectPropertyValue` — so the variable's editor
default is set without manual editing.

### Rename / remove

```
bp_variable_rename(bp_path, old_name, new_name)   # refactors Get/Set across all graphs
bp_variable_remove(bp_path, name)                 # orphans existing Get/Set nodes
```

## Function graphs

### Fresh new function

```
bp_function_graph_add(bp_path, name="DoStuff", category="Combat")
```

Creates an empty graph. Body must be authored via subsequent
`bp_node_create` / `bp_node_link_pins` calls — UE 5.7 Python can't
construct K2Nodes natively, hence the per-K2Node-class helpers in the
next section.

### Function overrides

**v1.3+**. Override a parent function — equivalent to clicking
"Override Function" → `<name>` in the editor.

```
# Step 1: list what's overridable.
bp_overridable_functions(bp_path)
  → {functions: [
      {name: "ReceiveBeginPlay",  flavor: "BlueprintImplementableEvent",
       owner_class: "/Script/Engine.Actor", already_overridden: false},
      {name: "K2_OnBecomeViewTarget", flavor: "BlueprintNativeEvent", ...},
      ...
    ]}

# Step 2: create the override graph.
bp_override_function(bp_path, function_name="ReceiveBeginPlay")
  → {success: true, newly_created: true}
```

Wraps `FBlueprintEditorUtils::AddFunctionGraph<UFunction>` so:

- `K2Node_FunctionEntry` binds its function reference to the parent
  function — `Parent:` super calls work.
- `K2Node_FunctionResult` is auto-added when the parent returns / has
  out-params.
- Compile + save happens on success.

Idempotent: re-running on an already-overridden function returns
`{success: true, newly_created: false}`.

`bp_overridable_functions` walks the parent class chain and filters to
UFUNCTIONs flagged `FUNC_BlueprintEvent` (Native + Implementable),
skipping `Final` / `Private`. That's the same set the editor's dropdown
shows.

### Rename / remove

```
bp_function_graph_rename(bp_path, old_name, new_name)   # auto-updates internal calls
bp_function_graph_remove(bp_path, name)
```

## Nodes — creating and wiring

### Create a node

```
bp_node_create(
  bp_path, graph_name,
  node_class_path="/Script/BlueprintGraph.K2Node_CallFunction",
  x=200, y=100,
) → {node_guid, pins: [...]}
```

`node_class_path` is the full UClass path of a `UK2Node` subclass.
Companion creates it via `NewObject<>` + `AllocateDefaultPins` and stamps
a fresh `NodeGuid`.

### Configure the node

A bare `K2Node_CallFunction` doesn't yet know which function to call —
configure it after creation:

| K2Node class | Configurator |
|---|---|
| `K2Node_CallFunction` | `bp_node_call_function_target(bp_path, graph_name, node_guid, function_owner_class_path, function_name)` |
| `K2Node_VariableGet` / `K2Node_VariableSet` | `bp_node_variable_target(bp_path, graph_name, node_guid, variable_name, member_owner_class_path)` |
| `K2Node_DynamicCast` | `bp_node_dynamic_cast_target(..., target_class_path)` |
| `K2Node_MakeStruct` / `K2Node_BreakStruct` | `bp_node_struct_target(..., struct_path)` |
| `K2Node_SwitchEnum` | `bp_node_switch_enum_target(..., enum_path)` |
| `K2Node_GetSubsystem` | `bp_node_get_subsystem_target(..., subsystem_class_path)` |
| `K2Node_EnhancedInputAction` | `bp_node_input_action_target(..., input_action_path)` |

Each configurator reconstructs pins automatically. After a configurator
the node's pin set is final — wire it now.

### Events

```
bp_event_node_add(bp_path, graph_name, parent_event_name="ReceiveBeginPlay")
```

For ReceiveBeginPlay / ReceiveTick / any other parent-event override. Use
`bp_event_find` first to avoid duplicates.

```
bp_custom_event_add(bp_path, graph_name, name="OnFoo", x, y)
```

For new user-named events other graphs can call.

### Wire pins

```
bp_node_link_pins(
  bp_path, graph_name,
  src_node_guid, src_pin_name,
  dst_node_guid, dst_pin_name,
)
```

Schema validates type compatibility — incompatible types return
`{success: false}` with the pin type info so the agent can `MakeStruct`
or `Cast` to bridge.

Inverse: `bp_node_break_link(src..., dst...)`. Idempotent.

`bp_node_pin_links(bp_path, graph_name, node_guid, pin_name)` enumerates
the OTHER side of every link on a pin — for tracing data flow.

## Pin defaults (the value when not wired)

### String / scalar defaults — `bp_node_pin_set_default`

For primitives and struct literals:

```
bp_node_pin_set_default(bp_path, graph_name, node_guid, pin_name="bAllowMontage", value="true")

# canonical struct serialization:
bp_node_pin_set_default(..., pin_name="Color", value="(R=1.0,G=0.5,B=0.0,A=1.0)")
bp_node_pin_set_default(..., pin_name="Loc",   value="(X=10.0,Y=20.0,Z=0.0)")
```

### Object references on pins

**v1.3.4+**. Object pins (`AnimMontage`, `Texture2D`, `BP class`, …) store
their default in `UEdGraphPin::DefaultObject` (a `UObject*`), NOT in the
string slot. Two routes:

**Smart routing (preferred)** — `bp_node_pin_set_default` auto-detects:

```
bp_node_pin_set_default(
  bp_path, graph_name, node_guid,
  pin_name="Anim Montage",
  value="/Game/Animations/Fall/AM_HardLanding_Forward",
)
# → if pin is object-shaped AND value looks like an asset path, routes
#   to TrySetDefaultObject. Response shows slot:"default_object".
```

**Explicit** — `bp_node_pin_set_object` for hard errors when the pin
isn't object-shaped:

```
bp_node_pin_set_object(
  bp_path, graph_name, node_guid,
  pin_name="Anim Montage",
  object_path="/Game/Animations/Fall/AM_HardLanding_Forward",
)
```

Object_path accepts:

- `/Game/Folder/Asset` — short (Companion appends `.Asset`).
- `/Game/Folder/Asset.Asset` — full.
- `ClassName'/Game/Folder/Asset.Asset'` — reference form (UE's
  `ToObjectReference` output).
- `/Script/Engine.AnimMontage` — for class pins.

Supports pin categories: object, class, soft_object, soft_class, interface.

### Object literal node — `bp_object_literal_create`

**v1.3.4+**. Alternative to pin defaults — explicit literal node:

```
bp_object_literal_create(
  bp_path, graph_name,
  object_path="/Game/Animations/Fall/AM_HardLanding_Forward",
  x=400, y=200,
) → {node_guid, output_pin: "Value"}

# then wire it:
bp_node_link_pins(..., src_pin_name="Value", dst_pin_name="Anim Montage")
```

Useful when the same asset literal feeds multiple pins, or when you'd
rather see the literal explicitly in the graph.

## Lifecycle: move, remove, reconstruct

```
bp_node_set_position(bp_path, graph_name, node_guid, x, y)
bp_node_remove(bp_path, graph_name, node_guid)            # severs links first
bp_node_reconstruct(bp_path, graph_name, node_guid)       # re-runs AllocateDefaultPins
                                                          # after a property mutation
```

## Compile + save

After ANY structural edit:

```
bp_compile_and_save(bp_path)
```

Configurator helpers (call_function_target / variable_target / etc.) and
the lifecycle helpers usually mark the BP dirty themselves; explicit
`bp_compile_and_save` makes the change persist + visible to the next
tool.

`unreal_pie_start` does its own compile-error pre-flight (see
[PIE workflow](pie-workflow.md)); BPs with BS_Error block PIE.

## End-to-end: "On Landed, play HardLanding montage"

```
# 1. Discover the graph + nodes we'll touch.
bp_graphs_list("/Game/Blueprints/BP_PlayerChar")
bp_graph_nodes_list(..., graph_name="EventOnLanded")

# 2. Add the call to Play Anim Montage.
bp_node_create(..., node_class_path="/Script/BlueprintGraph.K2Node_CallFunction",
               x=600, y=120)
bp_node_call_function_target(..., node_guid="<new>",
    function_owner_class_path="/Script/Engine.KismetSystemLibrary",
    function_name="PlayAnimMontage")

# 3. Set the object pin default.
bp_node_pin_set_object(..., node_guid="<new>",
    pin_name="Anim Montage",
    object_path="/Game/Animations/Fall/AM_HardLanding_Forward")

# 4. Wire exec from On Landed → Play Anim Montage.
bp_node_link_pins(..., src_pin_name="then", dst_pin_name="execute")

# 5. Compile + save.
bp_compile_and_save("/Game/Blueprints/BP_PlayerChar")

# 6. Verify with PIE.
pie_run_and_watch(wait_seconds=10)
```

## Cross-references

- [companion plugin](companion.md) — which version unlocks what.
- [PIE workflow](pie-workflow.md) — runs after BP edits to verify.
- [asset management](asset-management.md) — for `asset_describe` before
  you know exact property names.
- [unreal plugin tool catalog](../plugins/unreal.md) — full tool list.
