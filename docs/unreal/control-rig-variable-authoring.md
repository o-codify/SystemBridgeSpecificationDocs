---
id: request-control-rig-rig-variable-authoring-animbp-facing-input-output-contract
title: "Request: Control Rig rig-variable authoring (AnimBP-facing input/output contract)"
status: request
version: 26.603.1712
tags: [ unreal, control-rig, rigvm, variables, authoring, request ]
---

# Request: Control Rig rig-variable authoring

## Status

`request` — confirmed gap (UE 5.7.4, SystemBridgeCompanion 1.9.0). Filed
2026-06-03. Follows the now-shipped
[Control Rig authoring](control-rig-authoring.md) (v1.9), which explicitly
defers this: *"No variable authoring in v1.9. The AnimBP-facing variable
contract is created in v1.10... dynamic inputs do not [survive]."* This
request makes that need explicit rather than assuming a future version
will deliver it.

## The gap

v1.9 can author a Control Rig **graph** (nodes, links, pin defaults,
compile) but cannot create **rig variables**. Rig variables are the
mechanism by which the AnimGraph `Control Rig` node feeds **dynamic,
per-frame** data into the rig: an input rig variable becomes an input pin
on the Control Rig AnimGraph node, wired from AnimInstance/AnimBP data.

Without rig-variable authoring you can only set **static pin defaults** —
so a headless-built rig cannot receive live targets (e.g. per-frame hand
IK target transforms computed from weapon sockets). That blocks the
canonical HLS procedural-animation pattern: *"AnimInstance pushes one
coherent anim-state struct → Control Rig solves against it."*

## Why it matters

The HLS weapon-interaction spec prescribes a Control Rig fed by
`UProceduralWeaponManipulationComponent` → `UWeaponAnimInstance` with
per-frame hand targets, elbow poles, alphas, contact states. Every one
of those is a dynamic rig input. Procedural reach/IK for hands and feet
against world targets — the main reason to use Control Rig — requires
dynamic inputs. Static pin defaults cannot express them.

## Requested capability

New companion-backed tools (names illustrative; mirror the existing
`control_rig_*` ergonomics):

```
unreal_control_rig_variable_add(asset_path, name, cpp_type,
                                sub_object?, direction)   # direction: input | output | hidden
unreal_control_rig_variables_list(asset_path)            -> [{name, cpp_type, direction}]
unreal_control_rig_variable_remove(asset_path, name)
unreal_control_rig_variable_get_node_add(asset_path, graph, name, x, y)  # VariableNode (get)
unreal_control_rig_variable_set_node_add(asset_path, graph, name, x, y)  # VariableNode (set)
```

- `cpp_type` covers at least: `bool, int32, float/double, FName, FVector,
  FRotator, FTransform, FGameplayTag`, and struct/enum by sub-object path
  (mirror `bp_variable_add_typed` taxonomy).
- `direction = input` → the variable is exposed as a pin on the AnimGraph
  `Control Rig` node (settable per frame). `output` → readable back by the
  AnimBP. Implement via `URigVMController::AddExposedPin` /
  `AddVariableNode` + the blueprint's variable model, then RecompileVM.
- Variable get/set nodes so the rig graph can read/write them.

## Acceptance

- Headless: create an `input` rig variable `LeftHandTargetWorld`
  (`FTransform`) on a Control Rig; after compile it appears as an input
  pin on the AnimGraph `Control Rig` node and can be driven per-frame from
  the AnimBP; a `VariableNode` get inside the rig feeds a `Two Bone IK`
  effector; the solved hand tracks the live target in PIE.
- Round-trips through save/reload; indistinguishable from editor-authored.

## Workaround until available

- Use an AnimGraph `Two Bone IK` node (authorable today via
  `unreal_bp_node_*`) driven by an AnimBP variable target — HLS names
  Two Bone IK as the MVP solver and leaves "AnimGraph node vs Control Rig
  asset" as a per-rig choice. Migrate to Control Rig once rig variables
  exist.
- Or hand-author rig variables in the editor; automation builds the rest.
