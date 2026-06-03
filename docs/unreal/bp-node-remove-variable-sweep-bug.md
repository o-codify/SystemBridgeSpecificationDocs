---
id: bug-bp-node-remove-sweeps-unreferenced-member-variables
title: "Bug: bp_node_remove sweeps unreferenced member variables"
status: request
version: 26.603.1942
tags: []
---

# Bug: `unreal_bp_node_remove` sweeps unreferenced member variables

> Status: **bug report**. Observed with companion v1.10.1.

## Symptom

Calling `unreal_bp_node_remove` on a single node also **silently deletes member
variables that have no remaining references in the graph**. While building one
graph I removed a few scratch nodes; afterward `unreal_bp_variables_list` showed
six unrelated variables gone:
`RightHandTarget, PreviousPoseState, StepAlpha, HandToHandSupportContactState,
ShoulderContactState, bMirroredPresentation` — every member variable that
happened to be declared-but-not-yet-referenced. They were valid, intentional
state-contract variables awaiting wiring.

## Likely cause

`unreal_bp_node_remove` appears to run a "remove unused variables" / clean-up
pass (the explicit `unreal_bp_variable_remove` tool documents a
`replace_refs_to_sentinel + remove_unused_variables` method; node-remove seems
to share it). Sweeping ALL unreferenced variables as a side effect of removing
one node is destructive and surprising.

## Expected

`unreal_bp_node_remove` should remove only the requested node (and, at most, the
pins/links of that node). It must **not** delete member variables — especially
not ones unrelated to the removed node. Declaring a variable before wiring it is
a normal authoring order.

## Impact

Data loss during graph authoring; silent (no error/warning in the tool result).
Worked around by re-adding the swept variables, but any not-yet-noticed loss
would ship. Please scope variable cleanup to an explicit opt-in flag, or remove
it from `node_remove` entirely.
