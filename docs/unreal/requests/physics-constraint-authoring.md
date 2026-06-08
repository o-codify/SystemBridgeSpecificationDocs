---
id: request-physics-constraint-authoring-world-partition-cells-read
title: "Request: physics constraint authoring + world partition cells read"
status: request
version: 26.608.1430
tags: [ unreal, physics, world-partition, request ]
---

# Request: physics constraint authoring + WP cells read

**Origin.** Two smaller items grouped: both are blockers for specific
project types but neither warrants a standalone request.

## Physics constraints (write)

### What

`UPhysicsConstraintComponent` and `APhysicsConstraintActor` are the
authoring surface for joints (hinge / ball / prismatic). Currently no
headless way to add or configure.

### Tools

| Tool | Purpose |
|---|---|
| `physics_constraint_add(bp_or_actor, name, comp_a, comp_b)` | Add a UPhysicsConstraintComponent linking two components (or actors). Returns the new component name. |
| `physics_constraint_set(target, constraint_name, axis_swing1=, axis_swing2=, axis_twist=, linear_x/y/z=, motor_*=)` | Set the constraint profile (free / limited / locked per axis, soft/hard limits, motors). |
| `physics_constraint_list(target)` | List constraints with current profile. |

### Why

Doors, ragdolls, vehicle suspension, ropes. All current solutions are
manually-placed PhysicsConstraintActors in Blueprint editors.

### Implementation route

Companion C++. UPhysicsConstraintComponent's profile is exposed as
`FConstraintInstance` — Python can't reach the typed setters today.
Mirror `bp_set_component_property_typed` pattern for the per-axis
flags.

### Acceptance

- A door BP with two static meshes + a hinge constraint configured
  via tools opens correctly under PIE physics.
- `physics_constraint_list` returns the profile in a roundtrippable
  form.

---

## World Partition cells (read)

### What

For big-world projects using WP, AI needs to know which cells contain
which actors without loading the whole map.

### Tools

| Tool | Purpose |
|---|---|
| `world_partition_cells_list(map_path)` | List WP cells: `[{name, bounds, loaded, actor_count}]`. |
| `world_partition_actor_cell(map_path, actor_path)` | Which cell does this actor live in? |
| `world_partition_cell_actors(map_path, cell_name)` | Actors in one cell. |

### Why

"Where does this NPC live in the WP grid?" / "Which cells are
streaming-relevant for the Goblin Camp area?" — without these,
big-world refactor is blind.

### Implementation route

`UWorldPartition::ForEachActor` + `FActorContainer`. Pure read; no
mutations.

### Acceptance

- For a WP-enabled map, `world_partition_cells_list` returns the same
  cell grid the editor's WP outliner shows.
- Actor → cell lookup matches.

---

## Why bundle the two

Both are single-feature "complete a UE subsystem" requests; small
surface; same companion-C++ implementation pattern.

## Non-goals

- WP cell loading control (streaming policy mutation).
- Constraint break/restore at runtime.
- PhAT-editor parity (physics asset bone constraints — separate
  authoring surface).
