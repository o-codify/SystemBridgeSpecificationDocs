---
id: animgraph-authoring-headless
title: AnimGraph Authoring (headless)
status: stable
version: 26.603.2108
tags: [ unreal, animgraph, anim-blueprint, control-rig, authoring, companion ]
---

# AnimGraph Authoring (headless)

SystemBridgeCompanion v1.11.0+ exposes AnimGraph authoring so an AI
agent can place and wire AnimGraph nodes — including the `Control Rig`
node that feeds the rig built via [control rig authoring](control-rig-authoring.md) —
without opening the AnimBP editor.

Previously the only authoring surface was K2 graphs. v1.11 mirrors the
existing `bp_node_*` tools for AnimGraphs.

## Tool surface

| Tool | Purpose |
| ---- | ------- |
| `anim_node_add` | Instantiate any `UAnimGraphNode_*` subclass by class path. Returns `node_guid`. |
| `anim_node_set_inner_property` | Write a UPROPERTY on the inner `FAnimNode_*` runtime struct. Reconstructs the node so class-driven pin sets update. |
| `anim_node_list_exposable_pins` | **v1.11.1** — list every property toggleable into a pin, across `ShowPinForProperties` AND `CustomPinProperties` (Control Rig / Linked Anim Layer use the latter). Each entry: `{source: "show"|"custom", property, exposed}`. |
| `anim_node_expose_pin` | Toggle "expose as pin". Walks both `ShowPinForProperties` and `CustomPinProperties` (v1.11.1+) — the latter is where Control Rig bindable variables live. Reconstructs. |
| `anim_node_info` | Read node_class, inner_struct, position, pins (with `linked_count` and types), exposed pin properties (combined Show + Custom). |
| `anim_node_remove` | Remove by guid. Routes through `bp_node_remove` (gets the v1.10.2 variable-snapshot guard). |

Pose AND data wiring reuse `bp_node_link_pins` / `bp_node_break_link` /
`bp_node_pin_links` — `UAnimGraphNode_*` IS a `UEdGraphNode`, so the
K2 link/inspect tools work identically.

## Common node class paths

| `node_class_path` | Inner `FAnimNode_*` struct |
| ----------------- | -------------------------- |
| `/Script/ControlRigDeveloper.AnimGraphNode_ControlRig` | `AnimNode_ControlRig` |
| `/Script/AnimGraph.AnimGraphNode_TwoBoneIK` | `AnimNode_TwoBoneIK` |
| `/Script/AnimGraph.AnimGraphNode_ModifyBone` | `AnimNode_ModifyBone` |
| `/Script/AnimGraph.AnimGraphNode_LayeredBoneBlend` | `AnimNode_LayeredBoneBlend` |
| `/Script/AnimGraph.AnimGraphNode_SequencePlayer` | `AnimNode_SequencePlayer` |
| `/Script/AnimGraph.AnimGraphNode_BlendSpaceGraph` | `AnimNode_BlendSpaceGraph` |

Inner-property names live on the FAnimNode_* struct. Read them with
`anim_node_info`'s `inner_struct` field, then look up the struct in the
engine source or via `unreal.SystemBridgeBindings.struct_members_list` —
the same names you'd see in the editor's Details panel apply.

## Quick start — wire a Control Rig node into an AnimBP

Continues the [control rig authoring](control-rig-authoring.md) quick
start. The rig is built; now place its consumer in the AnimBP.

```python
ABP = "/Game/Characters/ABP_Hero"

# 1. Add a Control Rig node into the main AnimGraph.
add = unreal_anim_node_add(
    blueprint=ABP, graph_name="AnimGraph",
    node_class_path="/Script/AnimGraph.AnimGraphNode_ControlRig",
    x=400, y=0,
)
cr_node = add["node_guid"]

# 2. Point it at the rig built earlier.
unreal_anim_node_set_inner_property(
    blueprint=ABP, graph_name="AnimGraph", node_guid=cr_node,
    property_path="ControlRigClass",
    value="/Game/Rigs/CR_Hero.CR_Hero_C",
)
# Reconstruct auto-fires; the rig's input variables now appear as pins.

# 3. Expose the rig's public input variable as a pin.
unreal_anim_node_expose_pin(
    blueprint=ABP, graph_name="AnimGraph", node_guid=cr_node,
    property_name="LeftHandTargetWorld", expose=True,
)

# 4. Wire the AnimBP's Output Pose to consume the rig's pose output.
nodes = unreal_anim_blueprint_nodes(blueprint=ABP, graph_name="AnimGraph")
out = next(n for n in nodes
           if "AnimGraphNode_Root" in n["node_class"])["node_guid"]

unreal_bp_node_link_pins(
    bp_path=ABP, graph_name="AnimGraph",
    src_node_guid=cr_node, src_pin="Pose",
    dst_node_guid=out,     dst_pin="Result",
)

# 5. Feed LeftHandTargetWorld from an AnimBP variable.
# (variable already exists; create a Get-variable node and link it.)
get = unreal_bp_node_create(
    bp_path=ABP, graph_name="AnimGraph",
    node_class="/Script/BlueprintGraph.K2Node_VariableGet", x=0, y=0,
)
unreal_bp_node_variable_target(
    bp_path=ABP, graph_name="AnimGraph",
    node_guid=get["node_guid"], variable_name="LeftHandTargetWorld",
)
unreal_bp_node_link_pins(
    bp_path=ABP, graph_name="AnimGraph",
    src_node_guid=get["node_guid"], src_pin="LeftHandTargetWorld",
    dst_node_guid=cr_node,          dst_pin="LeftHandTargetWorld",
)

unreal_bp_compile_and_save(bp_path=ABP)
```

The AnimGraph now runs the rig and drives its hand-target input from an
AnimBP variable updated each frame.

## Inner-property semantics

`anim_node_set_inner_property` writes via
`FProperty::ImportText_InContainer` after walking the dot-notation path
from the inner FAnimNode_* struct. For most primitive types the value is
the obvious text form:

| Type | Example value |
| ---- | ------------- |
| `float` / `double` | `0.85` |
| `bool` | `True` / `False` |
| `FName` | `Hand_L` (bare) |
| `FVector` | `(X=1.0,Y=0.0,Z=0.0)` |
| `FRotator` | `(Pitch=0,Yaw=90,Roll=0)` |
| `FBoneReference` | `(BoneName="hand_l")` |
| Object / class | asset/class path (e.g. `/Game/Rigs/CR_Hero.CR_Hero_C`) |

For object and class leaves the tool detects an asset path and loads it
via `StaticLoadObject` / `StaticLoadClass` before assignment — mirrors
`bp_node_pin_set_object` ergonomics.

## "Expose as pin" semantics

AnimGraph nodes embed their input properties as inner-struct UPROPERTYs;
which of those surface as input pins is controlled by two TArrays of
`FOptionalPinFromProperty`:

- **`ShowPinForProperties`** — on `UAnimGraphNode_Base`. Lists the
  node's own optional pins (e.g. `Alpha` on most blend nodes).
- **`CustomPinProperties`** — on `UAnimGraphNode_CustomProperty` (the
  subclass used by the Control Rig anim node, Linked Anim Layer nodes,
  etc.). Lists the variables sourced from the *bound class*: a Control
  Rig's public input/output rig variables, a Linked Anim Layer's
  inputs, etc.

`anim_node_expose_pin` (v1.11.1+) tries `ShowPinForProperties` first
then falls through to `CustomPinProperties`. Properties absent from both
return `success: false` — there's no underlying mechanism to create a
pin for them.

When you don't know which list a property lives in (which is the common
case with Control Rig), call `anim_node_list_exposable_pins` first — it
returns one entry per toggleable property tagged with `source: "show"`
or `source: "custom"`.

After `ControlRigClass` is set on a Control Rig anim node, the rig's
public variables appear in `CustomPinProperties` via reconstruct; from
then on `anim_node_expose_pin(property_name="<rig var>")` flips the
show flag and the pin appears.

## Reading state

`anim_node_info` returns the same `FSBPinInfo` shape the K2 tools use —
`name`, `direction`, `pin_category` / `pin_sub_category` /
`pin_sub_category_object`, `linked_count`, `default_value`. Use it before
linking to discover the right pin names, and after editing to verify the
pin set.

For per-pin links (the OTHER side of every connection), call
`bp_node_pin_links` — same tool, works on AnimGraph nodes.

## Caveats

- **Anim Class Defaults pins** (per-property "Anim Class Defaults" tab
  inputs) are a different surface from `ShowPinForProperties`. v1.11
  covers the latter only.
- **State machines** (`UAnimStateMachineGraph`, transitions) are not yet
  in scope — `anim_node_add` works on the main AnimGraph and on any
  AnimGraph schema sub-graph it can find by name, but state-machine
  state authoring is deferred.
- **Custom AnimGraph node classes** from project modules work via their
  path (e.g. `/Script/MyGame.AnimGraphNode_Custom`) provided the class
  derives from `UAnimGraphNode_Base` and is loaded.

## Cross-references

- [companion plugin](companion.md) — v1.11.0 entry.
- [control rig authoring](control-rig-authoring.md) — builds the rig
  this page consumes.
- [blueprint authoring](blueprint-authoring.md) — the K2 graph
  authoring tools whose surface this page parallels.
- [transform query](transform-query.md) — read sockets / bones /
  actor transforms (useful for verifying IK targets).
