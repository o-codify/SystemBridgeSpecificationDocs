---
id: bug-control-rig-variable-get-node-add-can-t-bind-to-variable-add-d-rig-variables
title: "Bug: control_rig_variable_get_node_add can't bind to variable_add'd rig variables"
status: request
version: 26.603.1744
tags: [ unreal, control-rig, rigvm, variables, bug ]
---

# Bug: `control_rig_variable_get_node_add` can't bind to `variable_add`'d rig variables

## Status

`request` (bug report) — SystemBridgeCompanion **v1.10.0**, UE 5.7.4.
Filed 2026-06-03. Blocks the canonical procedural-animation path that v1.10
was added to enable (feeding per-frame data into a Control Rig).

## Symptom

`unreal_control_rig_variable_get_node_add` fails with:

```
RuntimeError: SystemBridgeBindings: Variable Node @@ is using a missing
variable. Consider recreating the node.
```

…even though the variable exists and is listed by
`unreal_control_rig_variables_list`. Worse: the failed call **still creates
a broken `VariableNode`** in the graph (title is correct, e.g.
"Get LeftHandTargetWorld", but its variable binding is "missing"). That
broken node then makes **every subsequent `control_rig_*` call fail**
(including `compile` and even `node_remove` of *other* nodes) with the same
error, until the broken node is removed. The `@@` suggests an empty/unset
variable name on the created node.

## Exact repro (verified)

```
unreal_control_rig_create(asset_path="/Game/.../CR_WeaponHold",
                          preview_skeletal_mesh="/.../AnimMan.AnimMan")
unreal_control_rig_variable_add(asset_path=..., name="LeftHandTargetWorld",
                                cpp_type="struct",
                                sub_object_path="/Script/CoreUObject.Transform",
                                direction="input")           # success
unreal_control_rig_variables_list(asset_path=...)            # lists it, cpp_type FTransform, direction input
unreal_control_rig_compile(asset_path=...)                   # success
unreal_control_rig_variable_get_node_add(asset_path=...,
                                variable_name="LeftHandTargetWorld")  # FAILS: "missing variable"
```

Same failure for `cpp_type="real"` variables and after a clean recompile.
`variable_set_node_add` is presumably affected too (same binding path).

## Expected

`variable_get_node_add(variable_name=X)` creates a Get `VariableNode` bound
to rig variable X (as `variables_list` reports it), so its output pin can be
linked to other nodes (e.g. a `Two Bone IK` `Effector`/`PoleVector`/`Weight`).
On failure it must NOT leave a broken node in the graph.

## Likely cause

The variable created by `variable_add` and the variable the get-node tries to
bind live in different collections / scopes (e.g. blueprint member variable
vs RigVM local variable), or the binding sets an empty name. The fix likely
needs the get-node to resolve the same variable model `variable_add` writes to
(and `AddVariableNodeFromObjectPath` / `AddVariableNode` with the correct
`InVariableName` + CPP type from the rig's variable description).

## Impact

This is the **only** missing link for headless canonical Control Rig
procedural animation: asset + variables + RigUnit nodes + pin defaults +
exec/data links + compile all work; only reading a rig variable inside the
graph (to drive a solver input dynamically) is broken. Without it, a
headless rig cannot consume per-frame inputs.

## Workaround

- Use an AnimGraph `Two Bone IK` node driven by an AnimBP variable target
  (no Control Rig variable needed) — HLS names Two Bone IK as the MVP solver.
- Or hand-author the variable get nodes in the rig editor.
