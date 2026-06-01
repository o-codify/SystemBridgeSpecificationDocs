---
id: request-launch-a-closed-editor-cold-start
title: "Request: launch a closed editor (cold start)"
status: request
version: 26.601.1906
tags: []
---

# Request: launch a closed editor (cold start)

## Problem
SystemBridge can talk to a **running** UnrealEditor (`unreal_editor_status`,
`unreal_editor_restart`, `unreal_run_python`, all `unreal_bp_*`, PIE, …), but
there is **no tool to start the editor when it is not running**. If the editor
process is closed, every `unreal_*` tool fails with
`no UE editor found via multicast` and the only way forward is to launch
`UnrealEditor.exe` from an OS shell (PowerShell `Start-Process` /
`Bash`) — i.e. stepping outside SB.

`unreal_editor_restart` does not cover this: it restarts an **already running**
editor; it cannot bring up a cold one.

## Why it matters
A headless authoring/PIE session that finds the editor closed (machine
rebooted, previous session exited, crash without auto-recover) cannot recover
through SB alone. The agent is forced to shell out, which violates the
"SB / Python / C++ only" working rule and is brittle (hard-coded engine path,
project path, no readiness signal).

## Requested capability
A tool, e.g. `unreal_editor_launch`, that:

- Resolves the engine binary from the project's engine association (the
  `.uproject` `EngineAssociation` + registry/Launcher install path), so the
  caller does not hard-code `…/UE_5.7/Engine/Binaries/Win64/UnrealEditor.exe`.
- Takes the project path (defaults to the session's `cwd` project, same
  resolution `unreal_editor_status` already uses).
- Launches detached and **blocks until the bridge is reachable** (companion
  loaded + Python Remote Execution up), with a timeout — returning the same
  payload shape as `unreal_editor_status` on success, or a structured timeout
  error. This removes the manual "sleep 90s then poll status" dance.
- Optional args: `level` to open, `wait_ready_timeout_s`, `extra_args`
  (e.g. `-log`).
- No-op (returns current status) if an editor for that project is already
  running, to stay idempotent.

## Current workaround (outside SB)
```
Start-Process -FilePath "<engine>/Engine/Binaries/Win64/UnrealEditor.exe" \
  -ArgumentList '"<project>.uproject"'
# then poll unreal_editor_status until alive:true
```
Used only because no SB tool exists; should be retired once the tool lands.

## Acceptance
- With the editor closed, a single tool call brings it up and returns
  `alive:true` with companion/version info, no shell access required.
- Idempotent when already running.
