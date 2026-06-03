---
id: control-rig-anim-node-expose-drive-rig-variables-as-pins
title: "Control Rig anim node: expose & drive rig variables as pins"
status: request
version: 26.603.2045
tags: []
---

# Control Rig anim node: expose & drive rig variables as pins

> Status: request. Found while wiring a Control Rig into an AnimGraph with the
> new v1.11 anim-node tools. UE-generic.

## Problem

`unreal_anim_node_add(AnimGraphNode_ControlRig)` + `unreal_anim_node_set_inner_property(ControlRigClass=...)`
work: the node is created and bound to a rig class. But the rig's **input
variables do not become pins**, so there is no way to feed them.

Observed with `CR_WeaponHold` (6 input variables: `LeftHandTargetWorld`,
`RightHandTargetWorld` (FTransform), `LeftHandAlpha`, `RightHandAlpha` (double),
`LeftElbowPoleWorld`, `RightElbowPoleWorld` (FVector)) — all `direction: input`
per `unreal_control_rig_variables_list`:

- After `ControlRigClass` is set and the AnimBP is compiled,
  `unreal_anim_node_info` shows only `Source / Alpha / bAlphaBoolEnabled /
  AlphaCurveName / Pose`. No rig-variable pins.
- `unreal_anim_node_expose_pin(property_name="LeftHandTargetWorld")` →
  `success:false` ("no entry / not an optional pin candidate").

Root cause: the Control Rig anim node is a `UAnimGraphNode_CustomProperty`.
Its rig-variable pins are **not** `ShowPinForProperties` entries (which is what
`unreal_anim_node_expose_pin` toggles). They are driven by the node's custom
exposed-variable list (editor-node side, e.g. `ExposedPropertyNames` +
`FAnimNode_ControlRig::InputMapping`), populated in the editor by the "Variables"
section checkboxes. There is currently no SB path to that.

## Requested capability (UE-generic)

For `UAnimGraphNode_CustomProperty` nodes (Control Rig anim node, Linked Anim
Graph/Layer nodes), support exposing and driving the custom variables:

- `unreal_anim_node_list_exposable_variables` — list a CustomProperty node's
  bindable variables (name, type, direction), distinct from ShowPin properties.
- `unreal_anim_node_expose_variable` — toggle a rig/custom variable as a pin
  (drive the editor node's exposed list + reconstruct), so it appears as a pin.
- Then the existing `unreal_bp_node_link_pins` should be able to wire those pins
  from data nodes (variable-get / property-access) like any other data pin.

(If `expose_pin` is intended to also cover CustomProperty variables, that would
satisfy this — today it returns success:false for them.)

## Why generic

Driving a Control Rig in an AnimGraph from AnimBP data is THE standard UE
procedural-animation pattern; exposing the rig's variables as pins is the only
way to feed it per-frame. Same applies to Linked Anim Layer inputs.

## Motivating case

Procedural weapon hold: feeding component computes both hands' world targets +
elbow poles; `CR_WeaponHold` (Two Bone IK per arm, full-transform effectors)
solves them. The node is placed and class-bound; only the variable-pin exposure
is missing to connect the component's outputs.
