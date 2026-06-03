---
id: transform-query-sockets-bones-actors
title: Transform Query (sockets, bones, actors)
status: stable
version: 26.603.2027
tags: [ unreal, transforms, sockets, bones, queries, pure-python ]
---

# Transform Query (sockets, bones, actors)

Read transforms out of Unreal — socket definitions on a mesh asset,
reference-pose bone transforms, and live actor/component/bone/socket
world transforms during PIE.

All pure-Python: no companion required.

## When to reach for this

Foundational read primitive for anything spatial:

- Place IK targets relative to a weapon mesh's `GripSocket`.
- Derive grip / attach offsets from the character mesh's `hand_l`
  reference-pose transform.
- Verify a procedurally-driven pose matches an authored one.
- Align effects to sockets without round-tripping through Blueprint
  variables.

Without this you'd run a `unreal_run_python` escape hatch or hand-author
constants. With it, tools can read socket / bone transforms directly
and bake correct values.

## Tool surface

### Static (asset-time, no PIE)

| Tool | Purpose |
| ---- | ------- |
| `mesh_sockets_list` | List sockets on a SkeletalMesh or StaticMesh. Each entry: name, parent_bone, relative location/rotation/scale. |
| `mesh_socket_transform` | A SkeletalMesh socket's relative transform PLUS its parent bone's component-space transform. Static ref-pose, no PIE. |
| `skeleton_bones_list` | Bones on a Skeleton (accepts Skeleton OR SkeletalMesh path). index, name, parent_index, parent_name. |
| `skeleton_bone_transform` | A bone's reference-pose transform in `"bone"` (local) or `"component"` space. |

### Live (PIE / level)

| Tool | Purpose |
| ---- | ------- |
| `actor_transform_query` | Live world transform for an actor by display label. Optional `component_name` / `bone_name` / `socket_name` resolve a deeper transform. `relative_to` returns target relative to a reference. |

## Output shape

All transforms are returned as:

```jsonc
{
  "location": [x, y, z],
  "rotation": [pitch, yaw, roll],
  "scale":    [x, y, z]
}
```

— `FRotator` form for rotation (degrees, pitch/yaw/roll), matching the
rest of the `unreal_*` tool surface.

## Examples

### Read a weapon's grip socket

```python
unreal_mesh_sockets_list(asset_path="/Game/Weapons/SM_Pistol")
# → {sockets: [{name: "GripSocket", parent_bone: "", relative_location: [0, 0, 10], ...}]}

unreal_mesh_socket_transform(
    asset_path="/Game/Characters/SK_Mannequin",
    socket_name="hand_l_socket",
)
# → relative + parent_bone_component_space
```

### Bake grip offset between weapon-grip and hand-bone

```python
hand = unreal_skeleton_bone_transform(
    asset_path="/Game/Characters/SK_Mannequin", bone_name="hand_l",
    space="component",
)["component"]
grip = unreal_mesh_socket_transform(
    asset_path="/Game/Weapons/SM_Pistol", socket_name="GripSocket",
)
# Compose grip relative to hand offline; bake into a DataAsset.
```

### Live: weapon-socket world transform relative to camera

```python
unreal_actor_transform_query(
    actor_label="BP_Hero_C_0",
    component_name="WeaponMesh",
    socket_name="MuzzleSocket",
    relative_to={"actor_label": "BP_Hero_C_0",
                 "component_name": "Camera"},
)
# → {world: {...}, relative: {...}}
```

## Caveats

- **5.7 ReferenceSkeleton binding is partial.** `skeleton_bone_transform`
  is best-effort: when the Python binding for `FReferenceSkeleton` is
  unavailable on the loaded UE build, the tool returns
  `bone_not_found_or_unsupported_api`. Fall back to
  `actor_transform_query` on a PIE actor that uses the skeleton.
- **`actor_transform_query` requires an active editor world.** Outside
  PIE it returns the editor-level world transform of the actor in the
  current level.
- **Actor lookup is by display label.** Use `unreal_level_actor_list` to
  discover labels. Two actors sharing a label is unsupported — the first
  match wins.

## Cross-references

- [companion plugin](companion.md) — v1.11.0 entry (this page ships
  alongside the AnimGraph release).
- [animgraph authoring](animgraph-authoring.md) — pairs naturally:
  read a socket location with `mesh_socket_transform`, drive a Control
  Rig effector pin with the result.
- [asset management](asset-management.md) — the broader read-side
  introspection surface.
