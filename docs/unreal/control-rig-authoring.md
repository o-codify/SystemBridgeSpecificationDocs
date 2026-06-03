---
id: control-rig-authoring-headless
title: Control Rig Authoring (headless)
status: stable
version: 26.603.1703
tags: [ unreal, control-rig, rigvm, animation, authoring, companion ]
---

# Control Rig Authoring (headless)

SystemBridgeCompanion v1.9.0+ exposes Control Rig (RigVM) authoring so an
AI agent can stand up a procedural-animation rig from scratch without
opening the editor's rig view.

A Control Rig is a `UControlRigBlueprint` asset whose nodes live in a
`URigVMGraph`. Mutations go through `URigVMController` (the RigVMDeveloper
module); after structural edits the rig VM is recompiled and the asset is
saved.

UE 5.7 Python doesn't bind `URigVMController` at all, so v1.9 wraps it in
C++ and exposes one tool per operation. Asset creation rides on pure-Python
`unreal.ControlRigBlueprintFactory` — no companion needed for that step.

## When to reach for this

Control Rig is the canonical procedural-animation solver in UE: bone
constraints, IK (Two Bone IK, FBIK, Aim), Get/Set Transform, math. An
AnimGraph's `Control Rig` node forwards the AnimInstance pose into the rig
and reads the solved pose back. Reach for this when you need:

- Procedural reach / IK for hands and feet against world targets.
- Per-rig adjustments fed by one anim-state struct from the AnimBP.
- A solver authored *as data* so different characters can swap rigs.

For simple cases — a single Two Bone IK in the AnimGraph — the
[blueprint authoring](blueprint-authoring.md) tools are sufficient and
don't require Control Rig.

## Tool surface

All tools take an `asset_path` to the ControlRigBlueprint and (where
relevant) an optional `graph_name` — leave empty for the main `RigGraph`.

| Tool | Purpose |
| ---- | ------- |
| `control_rig_create` | Create a new ControlRigBlueprint asset. Pure Python. Idempotent. |
| `control_rig_graphs_list` | List rig graphs (v1.9: main RigGraph only). |
| `control_rig_nodes_list` | List nodes with name, title, RigUnit struct path, position, pin count. |
| `control_rig_node_add` | Add a RigUnit node by `UScriptStruct` path. Returns the new node's name. |
| `control_rig_node_remove` | Remove a node by name. Idempotent. |
| `control_rig_pin_set_default` | Set a pin default using RigVM's text serializer. |
| `control_rig_add_link` | Add an exec/data link between two pins. |
| `control_rig_compile` | RecompileVM + save. |

> v1.9 covers the main rig graph. Function-library graphs and rig variables
> (the AnimBP-facing contract) land in v1.10 — track [companion timeline](companion.md#version-timeline).

## Quick start — Two Bone IK rig

Stand up a minimal rig that solves one chain.

```python
# 1. Asset
unreal_control_rig_create(asset_path="/Game/Rigs/CR_Hero",
                          preview_skeletal_mesh="/Game/Mannequin/Meshes/SK_Mannequin.SK_Mannequin")

# 2. Add a Two Bone IK unit (struct path varies per UE release;
#    use the editor's "Show in Content Browser" on a hand-placed unit
#    to discover the canonical path).
unreal_control_rig_node_add(
    asset_path="/Game/Rigs/CR_Hero",
    rig_unit_struct="/Script/ControlRig.RigUnit_TwoBoneIKSimplePerItem",
    x=400, y=0,
)
# → {"success": true, "node": "RigUnit_TwoBoneIKSimplePerItem"}

# 3. Set a pin default (RigVM text serializer — vectors as "(X=..,Y=..,Z=..)").
unreal_control_rig_pin_set_default(
    asset_path="/Game/Rigs/CR_Hero",
    node_name="RigUnit_TwoBoneIKSimplePerItem",
    pin_path="EffectorTransform.Translation",
    value="(X=50.0,Y=0.0,Z=120.0)",
)

# 4. Wire it into the rig's forward-solve entry. The forward-solve entry
#    node is created automatically; query it.
nodes = unreal_control_rig_nodes_list(asset_path="/Game/Rigs/CR_Hero")
entry = next(n for n in nodes["nodes"]
             if n["title"].startswith("Forwards Solve"))["name"]

unreal_control_rig_add_link(
    asset_path="/Game/Rigs/CR_Hero",
    src_node=entry, src_pin="ExecuteContext",
    dst_node="RigUnit_TwoBoneIKSimplePerItem", dst_pin="ExecuteContext",
)

# 5. Compile.
unreal_control_rig_compile(asset_path="/Game/Rigs/CR_Hero")
```

The rig is now usable from an AnimGraph `Control Rig` node — point it at
`/Game/Rigs/CR_Hero` via the AnimBP's class default.

## Pin paths

`pin_path` uses dot notation rooted at the node (no node name prefix):

- `Item.Type` — `Item` is an `FRigElementKey` struct pin; `.Type` is its
  `ERigElementType` enum sub-pin.
- `EffectorTransform.Translation.X` — nested struct walk.
- `ExecuteContext` — execution pin, no sub-pins.

For `control_rig_add_link` the pin path is split — `src_pin` / `dst_pin`
are the dot-notation tail, the node name is its own argument.

## Pin value format

`control_rig_pin_set_default` accepts the value as a string. RigVM uses
its own text serializer:

| UE type | Example value |
| ------- | ------------- |
| `FVector` | `(X=1.0,Y=0.0,Z=2.5)` |
| `FRotator` | `(Pitch=0.0,Yaw=90.0,Roll=0.0)` |
| `FTransform` | `(Rotation=(X=0,Y=0,Z=0,W=1),Translation=(X=1,Y=2,Z=3),Scale3D=(X=1,Y=1,Z=1))` |
| `FName` | `Hand_L` (bare) |
| `float` / `int` | `0.5`, `42` |
| `bool` | `True` / `False` |
| `FRigElementKey` | `(Name="Hand_L",Type=Bone)` |

For nested struct pins you can either set the whole struct on the parent
pin, or pick a single leaf with a longer `pin_path`. The latter survives
upstream type changes better.

## Finding RigUnit struct paths

A RigUnit's `UScriptStruct` path looks like
`/Script/<Module>.RigUnit_<Name>`. Discover them by:

- Opening a hand-authored rig in the editor, right-clicking the node,
  "Show in Content Browser" (the struct lives in /Script/...).
- Reading [SystemBridge sb-unreal source](https://github.com/o-codify/SystemBridge/tree/master/cmd/sb-unreal) — the bindings use the same paths.
- Iterating `unreal.UnitStruct` subclasses from Python in a probe script.

Common units:

| Struct path | What it does |
| ----------- | ------------ |
| `/Script/ControlRig.RigUnit_GetTransform` | Read a bone/control transform. |
| `/Script/ControlRig.RigUnit_SetTransform` | Write a bone/control transform. |
| `/Script/ControlRig.RigUnit_TwoBoneIKSimplePerItem` | Two Bone IK solver. |
| `/Script/ControlRig.RigUnit_FullBodyIK` | FBIK solver. |
| `/Script/ControlRig.RigUnit_AimItem` | Aim a bone at a target. |

## Constraints and gotchas

- **One graph in v1.9.** Function-library graphs land in v1.10. If you need
  to share sub-graphs across rigs today, duplicate them by re-running the
  same node-add sequence per rig.
- **No variable authoring in v1.9.** The AnimBP-facing variable contract
  is created in v1.10. As a workaround, expose data via input *pins* on
  the forward-solve entry — defaults survive, dynamic inputs do not.
- **Compile after structural changes.** `control_rig_compile` is not
  free; batch your `node_add` + `add_link` + `pin_set_default` calls
  and compile once at the end. The Python helpers save the asset on
  every mutation so the file is recoverable mid-batch.
- **Pin paths are case-sensitive.** `executeContext` won't match;
  `ExecuteContext` will.

## Round-trip

A rig built headlessly is a normal `UControlRigBlueprint` asset:

- It opens in the editor as if hand-authored.
- It survives save / reload / pkg discard / rename.
- The AnimGraph `Control Rig` node accepts it like any other.
- Live Coding doesn't apply (RigVM is interpreted, not compiled C++).

## Cross-references

- [companion plugin](companion.md) — version timeline; v1.9.0 entry.
- [blueprint authoring](blueprint-authoring.md) — for AnimGraph-level IK
  when a full Control Rig is overkill.
- [data authoring](data-authoring.md) — for the data assets that
  typically feed the rig's variables (once v1.10 lands).
