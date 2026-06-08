---
id: systembridge-overview
title: SystemBridge — Overview
status: stable
version: 26.608.1659
tags: [ overview, index ]
---

# SystemBridge

A local MCP daemon that gives an AI coding agent structured, token-efficient
access to a developer's environment — files, git, processes, the browser, the
running editor, databases, containers, GitHub, CI / build / test / lint, LSP
code intel, web scraping, semantic code search, research backends, and
(deeply) Unreal Engine — through **19 specialized plugins** and a thin core
that owns lifecycle, discovery, consent, events, structured errors, risk
labels, and watchers.

The agent talks to one MCP endpoint (`sb.exe`). `sb` spawns per-capability
plugin processes via stdio, exposes their tools, and aggregates a low-cost
`discover()` summary so the agent doesn't have to list-and-stat the world on
every turn.

```
   AI agent ── MCP/stdio ──▶  sb.exe (core)
                                │
                  ┌─────────────┼─────────────┐
                  ▼             ▼             ▼
              files.exe    unreal.exe    git.exe   …
              (plugin)     (plugin)      (plugin)
```

## Table of contents

- [Architecture](architecture.md) — daemon + plugin model, MCP wire, manifests,
  lifecycle, permissions, events.
- [Installation](installation.md) — `sb install`, Claude Code wiring, embedded
  skill, PATH propagation.
- [Plugins](plugins/index.md) — registry of all 19 plugins, summary tables,
  per-plugin pages.
- [Architecture: structured error codes](architecture-errcodes.md) — typed
  `errcodes` vocabulary shared across plugins.
- [Architecture: risk labels](architecture-risk-labels.md) — low / medium /
  high on write tools, surfaced via `mcp.tools_list`.
- [Architecture: watch / streaming](architecture-watch.md) — long-running
  observer convention + `watch.list/poll/stop/status` core tools.
- [Architecture: MCP introspection](architecture-mcp-introspection.md) —
  `mcp.*` + `doctor.*` for self-introspection and health.
- [Unreal deep dive](unreal/index.md) — the largest plugin surface, with its
  C++ companion sub-plugin, headless Blueprint authoring, PIE lifecycle, Live
  Coding, crash recovery, and the editor Message Log.
- [Workflows](workflows/index.md) — canonical end-to-end recipes (C++
  iteration, PIE debugging, BP editing, headless rebuild).
- [Troubleshooting](troubleshooting.md) — common failure modes (companion
  drift, locked DLLs, MCP transport errors, dialog blocks on PIE).
- [Changelog](changelog.md) — versioned milestones, breaking changes.

## What SystemBridge is for

Three loosely-coupled goals:

1. **Token efficiency.** Replace `cat file | head`, `ls -R`, `git log --all`,
   `tasklist`, and a dozen ad-hoc bash idioms with structured tool calls that
   return *only* what the agent asked for in a small JSON envelope. The
   [files](plugins/files.md) plugin alone replaces ~5 commonly used shell
   habits.

2. **Privileged access without shell escape.** Many useful operations need
   editor-internal state — the current PIE world, a Blueprint's compile
   status, the contents of UE's Message Log window. These can't be reached
   from a shell. SystemBridge surfaces them through typed tools with
   per-permission consent.

3. **A native escape hatch where the agent owns the abstraction.** Tools like
   `unreal_run_python` accept arbitrary scripts and return parsed
   `<<<SB_JSON>>>` blocks. When the agent does need to reach lower, the
   envelope (timeouts, output capping, project resolution) is still done by
   the daemon.

## What SystemBridge is NOT

- Not an MCP server "for files in general" — it's bound to the **current
  project directory** (the cwd `sb` was started in). Paths must resolve under
  cwd; symlink escapes are refused; `.senseignore` is honored.
- Not a remote agent. The daemon runs locally next to the IDE / editor.
- Not opinionated about which AI vendor connects. Standard MCP wire.

## Quick orientation by use case

- "What changed in the project recently?" → [files plugin](plugins/files.md)
  `recent_changes` + [git plugin](plugins/index.md#git) `recent_commits`.
- "Edit C++, see if it compiles and hot-reloads in UE." →
  [C++ iteration workflow](workflows/cpp-iteration.md).
- "Override a Blueprint event from Python." →
  [blueprint authoring](unreal/blueprint-authoring.md) →
  `bp_overridable_functions` then `bp_override_function`.
- "PIE crashed silently, find out why." → [PIE workflow](unreal/pie-workflow.md)
  uses `pie_run_and_watch` + `editor_status.last_crash`.
- "Set a `AnimMontage` reference on a Blueprint pin without clicking." →
  [blueprint authoring](unreal/blueprint-authoring.md#object-references-on-pins)
  → `bp_node_pin_set_object`.
- "Make a Blueprint event multicast / replicate a variable." →
  [blueprint authoring → replication](unreal/blueprint-authoring.md#replication)
  → `bp_custom_event_configure` + `bp_variable_set_replication`.
- "Clone a DataTable row and tweak a field with parens in its name." →
  [asset management → DataTable rows](unreal/asset-management.md#datatable-rows)
  → `dt_row_add` + `dt_row_set_field`.
- "Editor is closed; bring it up." → `unreal_editor_launch` (cold-start
  counterpart to `editor_restart`).
- "Make a per-weapon upper-body reload montage — same slot layout as
  an existing ALS reload, different clip." →
  [asset management → AnimMontage from template](unreal/asset-management.md#animmontage-from-template)
  → `anim_montage_create_from_template`.
- "Drop a skeleton notify (e.g. `ReloadWeapon`) at a specific time on
  a montage." →
  [asset management → AnimMontage notifies](unreal/asset-management.md#animmontage-notifies)
  → `anim_montage_add_notify`.
- "Build a project-authored DataTable from scratch (struct + rows)." →
  [data authoring](unreal/data-authoring.md) → `struct_create` +
  `struct_member_add` + `dt_row_add` + `dt_row_set_field`.
- "Add 50 GameplayTags without the editor restart cycle." →
  [data authoring → GameplayTags](unreal/data-authoring.md#gameplaytags)
  → `gameplaytag_add_many`.
- "Create a new Blueprint subclass or DataAsset instance." →
  [data authoring → asset creation](unreal/data-authoring.md#asset-creation)
  → `bp_create` / `dataasset_create`.
- "Place an AnimGraph node — e.g. a Control Rig consumer — in an AnimBP and wire it." →
  [animgraph authoring](unreal/animgraph-authoring.md)
  → `anim_node_add` + `anim_node_set_inner_property` + `anim_node_expose_pin` + `bp_node_link_pins`.
- "Read a mesh socket / bone / actor world transform headlessly." →
  [transform query](unreal/transform-query.md)
  → `mesh_sockets_list` / `skeleton_bone_transform` / `actor_transform_query`.
- "Stand up a Control Rig solver headlessly — add a Two Bone IK unit and wire it." →
  [control rig authoring](unreal/control-rig-authoring.md)
  → `control_rig_create` + `control_rig_node_add` + `control_rig_add_link` + `control_rig_compile`.
- "Feed per-frame data from the AnimBP into a Control Rig (hand IK targets, alphas)." →
  [control rig authoring → rig variables](unreal/control-rig-authoring.md#rig-variables-v110)
  → `control_rig_variable_add` (direction=input) + `control_rig_variable_get_node_add` + `control_rig_add_link`.
- "Refactor a Blueprint variable's name / type / removal cleanly — including a broken-typed var that won't delete." →
  [blueprint authoring → variable lifecycle](unreal/blueprint-authoring.md#variable-lifecycle--add-set-rename-remove-retype)
  → `bp_variable_rename_atomic` / `bp_variable_retype` / `bp_variable_remove_direct`.
- "Set a typed default (vector, transform, double, asset path) on a Blueprint component template so spawned instances inherit it." →
  [blueprint authoring → SCS component templates](unreal/blueprint-authoring.md#scs-component-templates--bp_set_component_property_typed)
  → `bp_set_component_property_typed`.
- "Author a SkeletalMesh socket headlessly with parent bone + offset." →
  [transform query → author a socket](unreal/transform-query.md#author-a-socket-on-a-skeletalmesh--mesh_socket_add)
  → `mesh_socket_add`.
- "Fire an input action (Reload, Slot1, Fire) in PIE without a keyboard." →
  [PIE input injection](unreal/pie-input.md)
  → `pie_input_inject` with `action_path`.
- "Call a Blueprint event/function on a live PIE object with typed args; read the return value." →
  [runtime invoke](unreal/runtime-invoke.md)
  → `runtime_invoke`.
- "Inspect ONE node's pins in a 600-node event graph without dumping the whole graph." →
  [blueprint authoring → single-node inspection](unreal/blueprint-authoring.md#single-node-inspection--bp_node_inspect_by_guid)
  → `bp_node_inspect_by_guid`.
- "Who breaks if I rename this asset?" or "every DataTable in the project" — fast project-wide audits without spending minutes on editor RPC. →
  [bulk offline scanner](unreal/bulk-offline-scanner.md)
  → `assets_find_references` / `assets_find_by_class` / `assets_scan_offline`.

## Repository

`github.com/o-codify/SystemBridge` — Go core, Go plugins, embedded Python
helper for the Unreal plugin, and the C++ Unreal companion sub-plugin source.
