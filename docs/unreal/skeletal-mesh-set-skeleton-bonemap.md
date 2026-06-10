---
id: set-skeleton-follow-up-explicit-bone-name-map-for-n-m-chain-remap
title: "set_skeleton follow-up: explicit bone_name_map for N→M chain remap"
status: request
version: 26.610.1618
tags: [ unreal, skeletal-mesh, skeleton, skin-weights, request ]
---

# set_skeleton follow-up: explicit bone_name_map for N→M chain remap

Follow-up to the shipped `unreal_skeletal_mesh_set_skeleton` tool. It matches bones by **name only**, which is wrong when a chain has more bones on the source than the target.

## Problem
A Sidekick spine has 5 bones (`spine_01..05`); the ALS spine has 3 (`spine_01..03`). Name matching binds `spine_01/02/03 → spine_01/02/03` and collapses the source's `spine_04/05` onto their parent `spine_03` (`remap_to_parent`). Result: the **upper torso/shoulders bunch at mid-back** instead of distributing up toward the neck. The correct mapping is by **position along the chain (endpoints + middle)**: `spine_01 → spine_01`, `spine_03 → spine_02`, `spine_05 → spine_03`, with the in-between source bones remapped.

## What
Add an optional input to `unreal_skeletal_mesh_set_skeleton`:
- `bone_name_map` — a `{source_bone: target_bone}` dictionary applied **before** name matching. Listed source bones bind to the named target bone regardless of their own name; unlisted bones fall back to name matching, then `missing_bone_policy`.

## Acceptance
- With `bone_name_map = {"spine_05":"spine_03","spine_03":"spine_02"}`, the re-targeted mesh's torso distributes across the target's full spine — shoulder/upper-chest geometry tracks the target's TOP spine bone, not its middle.
- Verification is **by world position of the deformed mesh**, not bone-name coincidence: after re-target onto the same skeleton, same-named bones trivially coincide (0 delta), so a name-based check proves nothing. What matters is that a source body region ends up at the target's corresponding position.

## Operational note (caller-side, important)
`set_skeleton` with `save=true` **overwrites the asset in place** and removes bones; with the source FBX absent it cannot be undone by re-import. Callers MUST duplicate the mesh and re-target the copy — never the original pack asset. Worth surfacing this in the tool description, and/or defaulting `save` to false / writing to a `*_ALS` copy.
