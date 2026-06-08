---
id: request-author-anim-sm-transition-rules-populate-state-pose
title: "REQUEST: author anim SM transition rules + populate state pose"
status: request
version: 26.608.213
tags: []
---

# REQUEST: author anim state-machine transition rules + populate state pose

## Gap

`unreal_anim_state_machine_add_transition` (v1.13.0) creates a transition with
an **always-true** rule (documented: "Transition rule authoring deferred to
v1.13.x"). `unreal_anim_state_machine_add_state` creates an **empty** state
(no pose / no inner SequencePlayer).

This makes it impossible to add a usable state to an existing, shared state
machine headless: an always-true transition out of a routing conduit hijacks
the conduit (every other branch becomes unreachable / order-dependent), and a
state with no pose outputs a ref pose.

## Concrete use-case

ALS `ALS_AnimBP` → `OverlayLayer` → "Overlay States" SM. Each overlay (Rifle,
Pistol, Hands Tied, …) is: one state whose pose = an `ALS_StanceVariation_*` /
`ALS_Props_*_Poses` asset, reached from the `<--Overlay State->` conduit by a
transition whose rule is `OverlayState == <ThatEnumValue>`, and returning via a
transition `OverlayState != <ThatEnumValue>`.

To add a new "Surrender" overlay (OverlayState enum value 13) headless I need to:
1. add a state and set its pose to a given AnimSequence (looping SequencePlayer);
2. add conduit→state transition with rule `OverlayState == SURRENDER`;
3. add state→conduit transition with rule `OverlayState != SURRENDER`.

Today only the edges can be created (always-true), so steps 1–3 can't be done
safely without raw-python K2 surgery on a shared asset.

## Requested API

- `unreal_anim_state_machine_set_state_pose(blueprint, graph_name, sm_guid,
  state_guid, anim_asset_path, loop=true)` — populate a state's inner graph
  with a SequencePlayer/Evaluator wired to Result.
- `unreal_anim_state_machine_set_transition_rule(blueprint, graph_name,
  sm_guid, transition_guid, rule)` — set a transition's boolean rule. Minimum
  viable `rule`: an enum-compare form, e.g.
  `{"kind":"enum_equals","variable":"OverlayState","enum_path":".../ALS_OverlayState","value":"SURRENDER","negate":false}`.
  A general expression form can come later; enum-equals covers the common
  state-selector pattern.

## Workaround in the meantime

Raw python via companion bindings (`create_k2_node`,
`set_k2_node_switch_enum_target` / variable-ref + enum `Equal`,
`link_k2_nodes_by_pin_names`, `anim_state_machine_state_inner_graph`) — but
this is fragile graph surgery on a shared SM; a typed tool is much safer.
