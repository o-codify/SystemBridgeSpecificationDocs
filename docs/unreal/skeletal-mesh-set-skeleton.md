---
id: unreal-skeletal-mesh-set-skeleton-re-target-mesh-onto-another-skeleton
title: unreal_skeletal_mesh_set_skeleton (re-target mesh onto another skeleton)
status: request
version: 26.609.2027
tags: [ unreal, skeletal-mesh, skeleton, skin-weights, request ]
---

# unreal_skeletal_mesh_set_skeleton (re-target mesh onto another skeleton)

## What
A SystemBridge tool that re-binds an existing `USkeletalMesh` onto a **different target `USkeleton`** headless — the programmatic equivalent of the editor's *Skeleton → Assign Skeleton*. Today this is impossible from Python/SB: `SkeletalMesh.skeleton` is read-only, `USkeleton` exposes no `merge_all_bones_to_bone_tree`, and there is no SB tool for it. The only routes are the editor UI (a manual click, not headless) or an FBX re-import (needs the original source file, which is frequently absent).

## Goal
Make a mesh authored for skeleton A play skeleton B's animations **natively** (no IK-retarget approximation, no leader-pose), so every shared bone is exactly where B's animation puts it. Concretely: bind a Synty/UE5-mannequin mesh onto the project's `ALS_Mannequin_Skeleton` so ALS animations drive it directly.

## Behaviour
Inputs:
- `mesh_path` — the `USkeletalMesh` to re-target.
- `target_skeleton_path` — the `USkeleton` to bind it to.
- `missing_bone_policy` (enum) — how to handle mesh bones absent from the target skeleton:
  - `remap_to_parent` — collapse their skin weights onto the nearest ancestor that exists in the target (mesh keeps animating; the extra joints become static).
  - `merge_into_target` — add the missing bones to the target skeleton's bone tree (target grows; nothing is lost).
  - `fail` — abort and report the unmatched bones.
- `save` (bool) — commit + save the modified mesh (and target skeleton, if grown).

Operations / what it must do:
- Map the mesh's reference-skeleton bones to the target by **name**.
- Recompute/remap per-vertex skin weights so vertices weighted to unmatched bones follow `missing_bone_policy`; re-normalize weights.
- Set the mesh's `Skeleton` to the target and rebuild render data so the asset opens and animates without errors.
- Leave the mesh's bind-pose geometry unchanged (re-target, not reshape).

Outputs:
- success flag, the new skeleton, counts of `{matched, remapped, merged}` bones, and the list of unmatched bone names with the action taken.

Constraints / edge cases:
- Bone hierarchies that share names but differ in orientation/length must still bind (vertices follow the target bones).
- Modular part meshes (a reduced bone subset) must work, not only full-body meshes.
- Must not require the original import FBX.

## Acceptance
- After the call, the mesh's `skeleton` equals the target, and playing one of the target skeleton's `AnimSequence`s on a component using the mesh produces correct deformation — shared-name bones land exactly at the animation's positions (delta ≈ 0), with no compile/build errors and the asset openable in the editor.
- `remap_to_parent` on a mesh with target-absent bones (e.g. finger metacarpals, extra spine bones) yields a mesh that still animates its shared bones; the absent joints are static, not garbage.
- Wrong: any path that needs a manual editor click, the source FBX, or that silently drops geometry / leaves the mesh in a non-openable state.

## Related capability
A companion tool, `unreal_skeletal_mesh_transfer_skin_weights` (proximity/closest-point weight transfer from a source mesh already bound to the target skeleton), would cover the case where bones cannot be matched by name and weights must be inferred from geometry. Same goal — bind a mesh to a target skeleton headless — by a different method.
