---
id: request-headless-control-rig-rigvm-graph-authoring
title: "Request: headless Control Rig (RigVM) graph authoring"
status: request
version: 26.603.1633
tags: [ unreal, control-rig, rigvm, animation, authoring, request ]
---

# Request: headless Control Rig (RigVM) graph authoring

## Status

`request` — confirmed gap (UE 5.7.4, SystemBridgeCompanion 1.8.0). Filed
2026-06-03. A procedural weapon-interaction layer (and many other procedural
animation features) needs a **Control Rig** asset whose graph solves bones
(Two Bone IK, FBIK, Aim, transform get/set) from inputs. SB currently has rich
Blueprint/AnimGraph authoring but **no Control Rig authoring** at all.

## The gap

Probed the SB unreal tool surface: there are `unreal_bp_node_*`,
`unreal_anim_blueprint_*`, `unreal_material_*`, etc., but **zero** Control Rig
operations. You cannot, headless:

- create a Control Rig asset (`UControlRigBlueprint`),
- add/list/remove RigVM nodes in its graph (RigUnit nodes like
  `Two Bone IK`, `Full Body IK`, `Aim`, `Get Transform`, `Set Transform`,
  `Set Control`, math),
- expose Control Rig variables (the contract the AnimBP feeds),
- wire RigVM pins,
- compile the Control Rig.

Control Rig graphs are **RigVM**, a different graph model from K2/`UEdGraph`,
so the existing `bp_node_*` tools do not apply. UE 5.7 Python exposure of
RigVM authoring is very limited, so this needs companion C++ wrappers
(`URigVMController` is the editor entry point: `AddUnitNode`, `AddVariableNode`,
`AddLink`, `SetPinDefaultValue`, plus `UControlRigBlueprint` compile).

## Why it matters

HLS (and UE generally) treats Control Rig as the canonical procedural
animation solver. Its weapon-interaction spec prescribes:
"Control Rig node in the Animation Blueprint" fed one coherent anim-state
struct, solving hands/arms/spine against procedural targets. Without headless
Control Rig authoring, an AI agent cannot build the canonical solver — only
approximate it with AnimGraph `Two Bone IK` nodes (which SB *can* author), or
require a human to hand-author the rig.

## Requested capability

New companion-backed tools (names illustrative):

```
unreal_control_rig_create(package_path, name, parent? )            -> {cr_path}
unreal_control_rig_graphs_list(cr_path)                            -> [graphs]
unreal_control_rig_nodes_list(cr_path, graph)                      -> [{node_id, unit, pins}]
unreal_control_rig_node_add(cr_path, graph, rig_unit_struct_path, x, y) -> {node_id, pins}
   # rig_unit_struct_path e.g. "/Script/ControlRig.RigUnit_TwoBoneIKSimplePerItem"
unreal_control_rig_node_remove(cr_path, graph, node_id)
unreal_control_rig_pin_set_default(cr_path, graph, node_id, pin, value)
unreal_control_rig_link(cr_path, graph, src_node, src_pin, dst_node, dst_pin)
unreal_control_rig_variable_add(cr_path, name, cpp_type, sub_object?)  # the AnimBP-facing contract
unreal_control_rig_variables_list(cr_path)
unreal_control_rig_compile(cr_path)
```

Implement via `URigVMController` (AddUnitNode / AddVariableNode / AddLink /
SetPinDefaultValue / RemoveNode) + `UControlRigBlueprint` compile + save.
Mirror the ergonomics of the existing `bp_node_*` tools (stable node ids,
pin enumeration, type-validated links).

## Acceptance

- Create a Control Rig asset for a Mannequin-style skeleton headless.
- Add a `Two Bone IK` rig unit per arm, expose input variables
  (e.g. `LeftHandTargetWorld`, `RightHandTargetWorld`, elbow poles, alphas),
  wire them, set defaults, compile clean — no editor UI.
- The resulting Control Rig is usable from an AnimGraph `Control Rig` node and
  produces the expected bone solve in PIE.
- Round-trips through save/reload; indistinguishable from editor-authored.

## Workaround until available

- Use AnimGraph `Two Bone IK` nodes (authorable today via `unreal_bp_node_*`)
  driven by externally-computed world targets — HLS explicitly allows the IK
  method to "vary by character rig," and names Two Bone IK as the MVP solver.
- Or have a human author the Control Rig graph in-editor, with automation
  building everything else (components, data assets, AnimInstance, wiring).
