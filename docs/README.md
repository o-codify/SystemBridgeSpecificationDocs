---
id: systembridge-overview
title: SystemBridge — Overview
status: stable
version: 26.601.1817
tags: [ overview, index ]
---

# SystemBridge

A local MCP daemon that gives an AI coding agent structured, token-efficient
access to a developer's environment — files, git, processes, the browser, the
running editor, and (deeply) Unreal Engine — through 12+ specialized plugins
and a thin core that owns lifecycle, discovery, consent, and events.

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
- [Plugins](plugins/index.md) — registry of all 12 plugins, summary tables,
  per-plugin pages.
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

## Repository

`github.com/o-codify/SystemBridge` — Go core, Go plugins, embedded Python
helper for the Unreal plugin, and the C++ Unreal companion sub-plugin source.
