---
id: bug-anim-node-add-fails-reading-node-guid-for-control-rig-node
title: "Bug: anim_node_add fails reading node_guid for Control Rig node"
status: request
version: 26.603.2045
tags: []
---

# Bug: `unreal_anim_node_add` fails reading `node_guid` for Control Rig node

> Status: request (bug). Companion v1.11.0.

## Symptom

`unreal_anim_node_add(node_class_path="/Script/ControlRigDeveloper.AnimGraphNode_ControlRig")`
returns `success:false` with:

```
AttributeError: 'AnimGraphNode_ControlRig' object has no attribute 'node_guid'
```

The node **is** created in the AnimGraph (confirmed via
`unreal_bp_graph_nodes_list` — a new `AnimGraphNode_ControlRig` appears with a
valid guid), but the tool throws while building its return payload, so the
caller never receives the guid.

## Likely cause

The success path reads `node.node_guid` (or similar python attribute) which does
not exist on this UAnimGraphNode subclass. Use the editor-node guid accessor
(e.g. `get_editor_property('NodeGuid')` / the same path other anim nodes use)
rather than a `.node_guid` attribute.

## Workaround

Fetch the guid afterward via `unreal_bp_graph_nodes_list` and filter by class.

## Note

`node_class_path` also differs from the tool's doc example
(`/Script/AnimGraph.AnimGraphNode_ControlRig` → not found; the working path is
`/Script/ControlRigDeveloper.AnimGraphNode_ControlRig`). Worth correcting the
example or resolving by short class name.
