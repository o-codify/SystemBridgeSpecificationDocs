---
id: workflows-index
title: Workflows — Index
status: draft
version: 26.601.1517
tags:
  - workflows
  - index
---

# Workflows — Index

Canonical end-to-end recipes — the agent's "if-this-then-that" playbook.

## Pages

- [C++ iteration](cpp-iteration.md) — edit C++, hot-reload via Live
  Coding, verify in PIE.
- [PIE debugging](pie-debug.md) — start PIE, watch, triage crashes.
- [Headless rebuild](headless-rebuild.md) — kill editor, run UBT, restart.
- [Blueprint feature add](bp-feature-add.md) — discover graph, create
  nodes, set object references, compile.

## Choosing a workflow

| If the work involves … | Start here |
|---|---|
| Editing `.cpp` / `.h` files in a UE module | [C++ iteration](cpp-iteration.md) |
| Adding a node to a Blueprint | [BP feature add](bp-feature-add.md) |
| Running and debugging gameplay | [PIE debugging](pie-debug.md) |
| A clean UBT-only build (no editor reload) | [headless rebuild](headless-rebuild.md) |
| Auditing project health | [unreal/message-log.md](../unreal/message-log.md) → `editor_message_log_categories` first |

## Universal preconditions

For ANY UE workflow:

1. `unreal_editor_status` first. Confirms:
   - Editor alive.
   - `project_matches_cwd` (otherwise pass `project_dir` explicitly).
   - `companion_loaded` + no version drift.
   - `auto_recover.enabled` if you plan intentional kills.
   - `last_crash` — if recent and unrecovered, fix before continuing.

2. If you're going to edit Blueprints headlessly, confirm Companion
   ≥ 1.3.4 (object-ref support). Lower versions still work but with
   reduced coverage.

3. For C++ iteration without Live Coding (`project_build`), the editor
   must be CLOSED so the DLL link step succeeds. Suppress the
   [crash watcher](../unreal/crash-recovery.md) first with
   `editor_set_auto_recover(false, ttl=600)`.

## Cross-references

- [unreal deep dive](../unreal/index.md)
- [troubleshooting](../troubleshooting.md)
