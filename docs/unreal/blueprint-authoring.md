---
id: blueprint-authoring-headless
title: Blueprint Authoring (headless)
status: stable
version: 26.601.1933
tags: [ unreal, blueprint, authoring ]
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
  src_node_guid, src_pin,      # NB: param is `src_pin`, not `src_pin_name`
  dst_node_guid, dst_pin,      # NB: param is `dst_pin`, not `dst_pin_name`
)
```

> **Verified 2026-06-02.** The exact pin-link param names are `src_pin` /
> `dst_pin` (passing `src_pin_name`/`dst_pin_name` errors with
> `src_pin is required`). `bp_custom_event_add` takes `custom_event_name`.

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

## Replication

**v1.4.0+**. Networked features (RPC events, replicated state,
RepNotify) were impossible to author headless before v1.4 — UE 5.7's
stock Python can't touch `K2Node_CustomEvent::FunctionFlags` or
`FBPVariableDescription::PropertyFlags`. Three new tools:

### RPC custom events — `bp_custom_event_configure`

Configure replication on a `K2Node_CustomEvent`. Mirrors the editor's
right-click "Replicates" dropdown:

```
# 1. Create the event (existing v1.0 tool).
bp_custom_event_add(bp_path, graph_name="EventGraph",
                    custom_event_name="Multicast_PlayLandMontage")
  → {node_guid, pins: [...]}

# 2. Mark it NetMulticast Reliable.
bp_custom_event_configure(bp_path, graph_name="EventGraph",
                          node_guid="<new>",
                          net_mode="multicast", reliable=True)
  → {success: true, net_mode: "multicast", reliable: true}
```

`net_mode`:

| Tag | Flags set on `FunctionFlags` |
|---|---|
| `multicast` | `FUNC_Net \| FUNC_NetMulticast` |
| `server` | `FUNC_Net \| FUNC_NetServer` (clients call, server runs) |
| `client` | `FUNC_Net \| FUNC_NetClient` (server calls, owning client runs) |
| `notreplicated` | clears all net flags (local-only event) |

`reliable=true` adds `FUNC_NetReliable`. `call_in_editor=true` flips
the node's `bCallInEditor` UPROPERTY (the "Call In Editor" checkbox
in the node detail panel).

Validation note: Server/Client RPCs require a replicated owning
actor; Multicast must be invoked from the server. Companion just
sets the flags — graph correctness is the author's responsibility.

### Typed event parameters — `bp_event_add_param`

Custom events default to zero parameters. Add typed output pins:

```
bp_event_add_param(bp_path, graph_name, node_guid="<custom-event>",
                   param_name="Montage",
                   pin_category="object",
                   sub_category_object="/Script/Engine.AnimMontage")
  → {success: true, param_name: "Montage"}
```

The new parameter shows as an OUTPUT pin on the event node so wired
graphs receive its value (the call-site provides it as an INPUT).

`pin_category` / `sub_category_object` / `container` use the same
taxonomy as [`bp_variable_add_typed`](#object--class--struct--enum--containers--bp_variable_add_typed).

Works on any `UK2Node_EditablePinBase` — function entries are also
fair game.

### Replicated / RepNotify variables — `bp_variable_set_replication`

Flip a member variable to replicated or RepNotify mode:

```
# Plain replicated — server pushes to clients.
bp_variable_set_replication(bp_path, variable_name="Health",
                            mode="replicated")

# RepNotify — server change fires OnRep on every client.
bp_variable_set_replication(bp_path, variable_name="MovementAction",
                            mode="repnotify",
                            rep_notify_function="OnRep_MovementAction")
  → auto-creates the OnRep_MovementAction function graph if absent.

# Clear replication.
bp_variable_set_replication(bp_path, variable_name="Health",
                            mode="none")
```

Behind the scenes Companion sets `CPF_Net` (or `CPF_Net | CPF_RepNotify`)
on the variable's `FBPVariableDescription.PropertyFlags`, sets
`RepNotifyFunc`, and for `repnotify` mode creates an empty
`OnRep_<var>` function graph if the BP doesn't already have one
(UE recognizes a function with the same name as `RepNotifyFunc`
as the OnRep handler — empty body is fine).

`rep_notify_function` defaults to `OnRep_<variable_name>` when
empty.

### End-to-end multiplayer recipe

For the height-based hard landing montage that needed to replicate
to other players:

```
# 1. Add a NetMulticast Reliable event with a Montage parameter.
bp_custom_event_add(bp_path, "EventGraph", "Multicast_PlayLandMontage")
bp_custom_event_configure(bp_path, "EventGraph", node_guid="<event>",
                          net_mode="multicast", reliable=True)
bp_event_add_param(bp_path, "EventGraph", node_guid="<event>",
                   param_name="Montage",
                   pin_category="object",
                   sub_category_object="/Script/Engine.AnimMontage")

# 2. Wire the event's Montage output into Play Anim Montage.
bp_node_create(..., node_class_path="/Script/BlueprintGraph.K2Node_CallFunction",
               x=400, y=80)
bp_node_call_function_target(..., function_owner_class_path="/Script/Engine.KismetSystemLibrary",
                             function_name="PlayAnimMontage")
bp_node_link_pins(..., src_pin_name="then",    dst_pin_name="execute")
bp_node_link_pins(..., src_pin_name="Montage", dst_pin_name="Anim Montage")

# 3. On Landed (server-only path) calls the multicast.
#    Add a K2Node_CallFunction targeting Multicast_PlayLandMontage,
#    with the desired montage object set on its input.
```

After compile + 2-player PIE, both the listen-server and the
simulated proxy play the hard-landing montage synchronously.

> **Verified 2026-06-02** in `ALS_UltimateWarfare` (Companion v1.4.0),
> 2-player PIE, bidirectional (host-falls→client-sees and
> client-falls→host-sees), hard and medium tiers. Two gotchas worth
> recording for anyone verifying replication headlessly:
>
> 1. **Python `call_method` / ProcessEvent bypasses RPC net routing.**
>    Invoking a custom event from Python runs its *body locally* and does
>    NOT send the multicast/server RPC over the wire (a Server event body
>    runs on the caller, a Multicast plays only locally). So you cannot
>    confirm replication by calling the event from Python — drive the real
>    gameplay path (e.g. a genuine fall so the compiled `OnLanded` graph
>    fires) instead.
> 2. **Catch transient montages with a per-tick latch, not one-shot
>    polls.** A landing montage lasts ~1.5–3.4 s; reasoning latency between
>    one-shot `unreal_run_python` polls easily overshoots the window.
>    Register `unreal.register_slate_post_tick_callback(cb)` that records
>    the first non-None `anim_instance.get_current_active_montage()` on
>    each world, arm it, trigger the fall, then read the latch — timing
>    becomes irrelevant. (`bp_graph_nodes_list` also does not surface an
>    object pin's `DefaultObject`, so set/track montage defaults via
>    `bp_node_pin_set_object`, not by reading the node dump.)

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
