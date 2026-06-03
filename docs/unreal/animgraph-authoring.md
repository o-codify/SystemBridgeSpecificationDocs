---
id: animgraph-authoring-anim-nodes-control-rig-pose-links
title: AnimGraph Authoring (anim nodes, Control Rig, pose links)
status: request
version: 26.603.1941
tags: []
---

# AnimGraph Authoring (anim nodes, Control Rig, pose links)

> Status: **requested** (not yet implemented). Proposed by AI while wiring a
> procedural weapon-hold Control Rig into an existing AnimBP. UE-generic.

## Problem

SystemBridge can author regular Blueprint graphs (`unreal_bp_node_create`,
`unreal_bp_node_link_pins`, `unreal_bp_node_call_function_target`, etc.) but
has **no equivalent for AnimGraphs**. The existing anim tools are read/limited:

- `unreal_anim_blueprint_graphs` — lists graphs (names only).
- `unreal_anim_blueprint_nodes` — lists anim nodes (class + name only; **no
  pins, no links, no properties**).
- `unreal_anim_blueprint_set_node_asset_override` — swaps a sequence asset.

There is **no way** to:

- add an AnimGraph node (e.g. `AnimGraphNode_ControlRig`,
  `AnimGraphNode_TwoBoneIK`, `AnimGraphNode_ModifyBone`, `LayeredBoneBlend`),
- wire **pose links** (FPoseLink) between anim nodes,
- set an anim node's properties (e.g. a Control Rig node's `ControlRigClass`,
  alpha, bone names, blend weights),
- expose an anim node's inner properties **as pins** and drive them from
  AnimBP variables / property access (the standard way to feed per-frame data
  such as IK targets into a Control Rig node),
- read an anim node's pins/links/properties for verification.

This blocks the canonical Unreal procedural-animation workflow: feed a Control
Rig node in the AnimGraph from an AnimBP-facing data contract. We built the
Control Rig (`unreal_control_rig_*` works headless) and the feeding component,
but cannot place/wire the Control Rig node into the AnimGraph.

## Requested capability (UE-generic)

Mirror the regular-BP node tools for AnimGraphs:

- `unreal_anim_node_add` — add an anim node of a given class to a named anim
  graph; return guid + pins.
- `unreal_anim_node_set_property` — set a UPROPERTY on the anim node's inner
  `FAnimNode_*` struct (e.g. `ControlRigClass`, `LocationSpace`, `BoneToModify`,
  `Alpha`, blend weights).
- `unreal_anim_node_expose_pin` — toggle "expose as pin" for an inner property
  (so it can be driven), and `unreal_anim_node_link_pins` — wire both **pose**
  pins and **data** pins (including from property-access / variable-get nodes).
- `unreal_anim_node_pins` — read an anim node's pins/links/exposed properties
  (the missing read side; current `unreal_anim_blueprint_nodes` returns only
  class+name).
- For Control Rig nodes specifically: after `ControlRigClass` is set, the node
  must reconstruct so the rig's exposed input/output variables appear as pins
  (same as the editor does), so callers can drive them.

Outputs should match the regular-BP node tools' shape (guid, pins with
name/category/direction/links).

## Why generic, not project-specific

Control Rig nodes, Two Bone IK, Modify Bone, Layered Bone Blend, slots, and
pose wiring are core to **every** Unreal animation project. Headless AnimGraph
authoring is the missing half of the existing headless Blueprint authoring.

## Concrete motivating case

Procedural weapon hold: a feeding `ActorComponent` computes both hands' world
targets + elbow poles each frame; a Control Rig (`CR_WeaponHold`, Two Bone IK
per arm) solves them. The only missing step is placing an
`AnimGraphNode_ControlRig(CR_WeaponHold)` into the character AnimGraph, driving
its exposed `LeftHandTargetWorld/RightHandTargetWorld/...` input pins from the
component, and wiring its pose in/out — none of which current tools can do.
