---
id: control-rig-authoring-headless
title: Control Rig Authoring (headless)
status: stable
version: 26.603.1728
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

> v1.9 covers the main rig graph and pin defaults. **Rig variables** (the
> dynamic AnimBP-facing contract) land in v1.10 — see
> [Rig variables (v1.10)](#rig-variables-v110) below. Function-library graphs
> are deferred to v1.11 — track [companion timeline](companion.md#version-timeline).

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
- **Rig variables require v1.10+.** v1.9 can only set static pin defaults,
  which cannot express per-frame data. v1.10 adds rig variables — see
  [Rig variables (v1.10)](#rig-variables-v110) below.
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

## Rig variables (v1.10)

Rig variables are member variables on the `UControlRigBlueprint` that the
RigVM reads and writes via variable nodes. Public ones become input pins
on the AnimGraph `Control Rig` node — which is how an AnimBP feeds the rig
per-frame data (hand IK targets, elbow poles, alphas, contact states).

This is the dynamic-input contract v1.9 left as static-defaults-only.

### Tool surface

| Tool | Purpose |
| ---- | ------- |
| `control_rig_variable_add` | Add a rig member variable. |
| `control_rig_variables_list` | List variables — name, cpp_type, cpp_type_object, direction, default. |
| `control_rig_variable_remove` | Remove a variable by name. Idempotent. |
| `control_rig_variable_get_node_add` | Add a Get-variable node to the rig graph. |
| `control_rig_variable_set_node_add` | Add a Set-variable node to the rig graph. |

### Direction

Every rig variable has a direction:

| Direction | Meaning |
| --------- | ------- |
| `input` (default) | Public, settable. Appears as an input pin on the AnimGraph `Control Rig` node. The AnimBP writes it per-frame. |
| `output` | Public, read-only. The AnimBP can read it back. |
| `hidden` | Private — rig-internal scratch only. |

The direction is recorded in the variable's BP category as
`RigVar|Input` / `RigVar|Output` / `RigVar|Hidden`, so `control_rig_variables_list`
can recover it after save/reload. CPF flags are set accordingly
(`CPF_DisableEditOnInstance` for hidden, `CPF_BlueprintReadOnly` for output).

### Type vocabulary

`cpp_type` follows the existing `bp_variable_add_typed` taxonomy. The most
common types:

| `cpp_type` | RigVM type | `sub_object_path` required |
| ---------- | ---------- | -------------------------- |
| `bool` | `bool` | — |
| `int` / `int32` | `int32` | — |
| `int64` | `int64` | — |
| `float` / `double` / `real` | `double` (default) / `float` | — |
| `name` | `FName` | — |
| `string` | `FString` | — |
| `struct` | struct's CPP name (`FVector`, `FTransform`, ...) | yes |
| `enum` / `byte` | enum CPP type | yes for typed enums |
| `object` / `class` / `softobject` / `softclass` / `interface` | object pointer | yes |

`container` is `single` (default), `array`, or `set`. Map is reserved.

### Example — dynamic hand IK

Extend the [quick start](#quick-start--two-bone-ik-rig) so the effector
target comes from the AnimBP instead of a static pin default.

```python
# Add an input variable for the world-space hand target.
unreal_control_rig_variable_add(
    asset_path="/Game/Rigs/CR_Hero",
    name="LeftHandTargetWorld",
    cpp_type="struct",
    sub_object_path="/Script/CoreUObject.Transform",
    direction="input",
)

# Drop a Get-variable node in the rig graph.
get_node = unreal_control_rig_variable_get_node_add(
    asset_path="/Game/Rigs/CR_Hero",
    variable_name="LeftHandTargetWorld",
    x=200, y=0,
)["node"]

# Wire it into the Two Bone IK effector instead of the static default.
unreal_control_rig_add_link(
    asset_path="/Game/Rigs/CR_Hero",
    src_node=get_node, src_pin="Value",
    dst_node="RigUnit_TwoBoneIKSimplePerItem",
    dst_pin="EffectorTransform",
)

unreal_control_rig_compile(asset_path="/Game/Rigs/CR_Hero")
```

After compile, the AnimGraph `Control Rig` node exposes a
`LeftHandTargetWorld` input pin. Wire it from the AnimBP's per-frame anim
state (e.g. a struct member computed by `UProceduralWeaponManipulationComponent`)
and the solved hand tracks the live target.

### Set-variable nodes

`control_rig_variable_set_node_add` is symmetric with the Get counterpart —
use it when a rig writes back to an `output` variable (e.g. the rig
reports the solved hand world transform so the AnimBP can react). Same
variable, opposite direction at compile time.

### Caveats

- **Public visibility is binary.** v1.10 treats `input` and `output` both
  as public; only `hidden` is private. Output is marked read-only at the
  BP layer (CPF_BlueprintReadOnly), but appears on the AnimGraph node
  identically.
- **TMap not supported.** The container parameter accepts `single`,
  `array`, `set`. TMap rig variables are uncommon and reserved.
- **Edit-in-editor parity.** A rig variable added headlessly is
  indistinguishable from one created via the Variables panel — it survives
  save/reload and shows up in the editor's variables list with its
  recorded `RigVar|*` category.

## Cross-references

- [companion plugin](companion.md) — version timeline; v1.9.0 / v1.10.0 entries.
- [blueprint authoring](blueprint-authoring.md) — for AnimGraph-level IK
  when a full Control Rig is overkill.
- [data authoring](data-authoring.md) — for the data assets that
  typically feed the rig's variables.
