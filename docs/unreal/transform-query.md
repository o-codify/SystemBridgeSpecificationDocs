---
id: transform-query-sockets-bones-components
title: Transform Query (sockets, bones, components)
status: request
version: 26.603.1942
tags: []
---

# Transform Query (sockets, bones, components)

> Status: **requested** (not yet implemented). Proposed by AI while implementing
> procedural weapon interaction; the underlying need is UE-generic.

## Problem

There is currently no SystemBridge tool to **read transforms** out of Unreal.
`unreal_asset_describe` exposes top-level UObject properties but **not**:

- a SkeletalMesh / StaticMesh asset's **socket** definitions and their local
  transforms (name, parent bone, relative location/rotation/scale),
- a Skeleton / SkeletalMesh **reference-pose bone** transform (bone-space or
  component-space),
- a live **component / bone / socket world transform** during PIE.

This blocks any headless work that must reason about spatial relationships:
placing IK targets on a mesh, deriving grip/attach offsets, aligning effects to
sockets, validating that a procedural pose matches an authored one, etc. The
only workarounds are `unreal_run_python` (an escape hatch we want to avoid) or a
manual round-trip through the editor UI.

## Requested capability (UE-generic)

Read-only transform queries. Two layers:

### 1. Static (editor, no PIE) — from assets

- `unreal_mesh_sockets_list` — for a SkeletalMesh or StaticMesh asset, return
  each socket: `name`, `parent_bone`, `relative_location`, `relative_rotation`,
  `relative_scale`. Optionally also resolve a socket's **component-space**
  transform against the reference pose.
- `unreal_skeleton_bone_transform` — for a Skeleton/SkeletalMesh, return a
  named bone's reference-pose transform in `bone` and/or `component` space; and
  list bones with parent indices.

### 2. Live (during PIE) — from spawned actors/components

- `unreal_actor_transform_query` — given a level/PIE actor (and optionally a
  component name, bone name, or socket name), return the resolved **world**
  transform (and optionally relative-to-parent). Should also support resolving
  one transform **relative to another** (e.g. socket-of-A relative to
  component-of-B) so callers can derive offsets without doing matrix math by
  hand.

All outputs as `{location:[x,y,z], rotation:[pitch,yaw,roll], scale:[x,y,z]}`
(or quaternion variant), matching how other `unreal_*` tools return structs.

## Why generic, not project-specific

Every Unreal project that does IK, attachment, VFX alignment, weapon/tool
handling, or procedural animation needs to know where sockets and bones are in
space. This is a foundational read primitive, not a feature of any one game.

## Concrete motivating case

Procedural weapon hold: to author or verify grip offsets (hand-relative-to-
weapon, weapon-relative-to-body) we must read the weapon mesh's `GripSocket`
local transform and the character's `hand_l/hand_r/spine_03` transforms. Today
this can only be done at runtime inside a Blueprint, forcing a fully
runtime-computed design even when static authoring would be simpler. A static
socket/bone read would let tools bake correct values directly into a profile.
