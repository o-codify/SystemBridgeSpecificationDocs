---
id: request-level-actor-attach-parent-socket-spawn-with-properties
title: "Request: level actor attach (parent/socket) + spawn-with-properties"
status: request
version: 26.608.1429
tags: [ unreal, level, actor, request ]
---

# Request: level actor attach + spawn-with-properties

**Origin.** Self-audit. Authoring a hero rig in a level currently
needs: `level_actor_spawn` weapon → `level_actor_spawn` character →
[no headless way to attach the weapon to the character's hand
socket]. The agent has to round-trip through the editor GUI.

## What

Two small additions complete the level authoring surface.

## Tools

| Tool | Purpose |
|---|---|
| `level_actor_attach(actor_label, parent_actor_label, socket_name?, rule="snap"\|"keep_world"\|"keep_relative")` | Attach an actor (or one of its components) to a parent actor's component / socket. Idempotent on same target. |
| `level_actor_detach(actor_label, rule="keep_world")` | Detach. Symmetric. |
| `level_actor_attachments_list(actor_label)` | Inspect: returns `{parent, parent_socket, children: [{label, attach_socket}]}`. |
| `level_actor_spawn_with_properties(class_path, location, rotation, properties: {})` | Combined spawn + initial property set. Saves a round-trip. |

`actor_label` accepts the same forms as `sequencer_binding_add` —
display label or full object path.

## Why

- Hero + weapon + accessory placement headless.
- Vehicle + driver + passenger rigs in test maps.
- Set-dressing scripts that group props under a parent.

## Implementation route

Pure Python via `unreal.EditorActorSubsystem` + `actor.attach_to_actor`
(`AttachToActor` on UE 5.7 is bound). Socket resolution by name on
the parent's RootComponent.

For component-level attach (target a specific component on the
parent), the optional `parent_component_name` parameter. Falls back
to RootComponent.

## Acceptance

- `level_actor_attach("BP_Weapon_C_0", "BP_Hero_C_0", "WeaponSocket_R")`
  attaches the weapon to the hero's right-hand socket and survives
  level save/reload.
- `level_actor_spawn_with_properties("/Script/Engine.PointLight", ...,
  {Intensity: 5000.0, LightColor: [255,200,150]})` returns the spawned
  actor's label + applies both intensity and tint.
- `level_actor_attachments_list("BP_Hero_C_0")` reflects the attached
  weapon under `children`.

## Non-goals (defer)

- Editor-only "attach via drag in Outliner" UX parity (we have it as a
  data op, the GUI niceties aren't useful headless).
- Cross-level attach (referring to actors in unloaded sub-levels).
