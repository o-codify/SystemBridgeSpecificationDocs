---
id: architecture
title: Architecture
status: stable
version: 26.601.1605
tags: [ architecture, internals ]
---

# Architecture

SystemBridge is a **core daemon** + **plugin processes** + **embedded Python
helper** + (for UE) **C++ companion sub-plugin**. Every layer is decoupled
via documented wire formats so each can ship independently.

See also: [overview](README.md), [installation](installation.md),
[plugins index](plugins/index.md).

## Layers

```
┌──────────────────────────────────────────────────────────────┐
│  AI agent (Claude Code, Claude Desktop, any MCP client)      │
└───────────────────────┬──────────────────────────────────────┘
                        │  MCP / stdio (JSON-RPC)
┌───────────────────────▼──────────────────────────────────────┐
│  sb.exe — core                                               │
│   • plugin discovery (manifests + triggers)                  │
│   • tool routing (tool name → plugin)                        │
│   • discover() aggregation                                   │
│   • permissions / consent / .senseignore                     │
│   • event ring per plugin (warn/error history)               │
│   • global config + secrets                                  │
└───────────────────────┬──────────────────────────────────────┘
                        │  stdio (one pipe pair per plugin)
        ┌───────────────┼───────────────┬──────────────┐
        ▼               ▼               ▼              ▼
   files.exe      unreal.exe       git.exe        process.exe  …
   (plugin)       (plugin)         (plugin)       (plugin)
                        │
                        │  UE Python Remote Execution (UDP/TCP)
                        ▼
                  UnrealEditor.exe (project + companion DLL)
```

## Core (`sb.exe`)

Single Go binary. Responsibilities:

- **Plugin registry** — Reads each plugin's `manifest.json` (generated at
  build time from each plugin's `buildManifest()` Go func). Filters by
  `triggers.files_present` (e.g. `*.uproject`) and `triggers.processes`
  (e.g. `UnrealEditor.exe`) — plugins only activate when their domain is
  actually present in cwd.
- **Tool routing** — Every tool the agent sees is namespaced `pluginname_tool`.
  Core dispatches the call to the owning plugin over stdio.
- **`discover()`** — Single cheap call the agent makes once per turn to learn
  what's around: per-plugin `summary` blocks + `events_since_last_call` ring.
  Replaces ~5 manual reconnaissance calls. Each plugin defines a `Summary`
  callback that produces a small map (cached for N seconds via
  `SummaryCacheSeconds`).
- **Permissions** — `Permissions: []string{"read_files:project", ...}` in
  each manifest. The user is asked at install / first-use; subsequent calls
  pass through.
- **Lifecycle hints** — `manifest.Lifecycle{Killable: KillableAlways |
  KillableDeferred | KillableNever}`. Core can idle-kill plugins that have
  no in-memory state; plugins with state (e.g. the unreal log tailer) opt
  into `KillableDeferred`.
- **Event ring** — Each plugin has a `MaxEvents + MaxAgeSeconds` ring;
  events from the plugin (e.g. `unreal_log_error`, `unreal_editor_crashed`,
  `file_changed`) accumulate there and ride along on the next `discover()`.

## Plugins

Go binaries built from `cmd/sb-<name>/`. Each declares a manifest and a tool
set; both are surfaced to the agent through core. Built-in plugins:

| Plugin | Cwd activation | Key tools |
|---|---|---|
| **files** | always | `tree`, `read`, `read_range`, `head`, `tail`, `diff`, `outline`, `write`, `edit`, `move`, `delete`, `stat`, `watch`, `hex`, `read_many`, `recent_changes` |
| **git** | `.git` present | `status`, `diff`, `log_search`, `commit`, `branch`, `checkout`, `add`, … |
| **unreal** | `*.uproject` present OR `UnrealEditor.exe` running | huge — see [unreal plugin](plugins/unreal.md) |
| **process** | always | `list`, `find`, `kill`, `tree`, `ports`, `run_command`, `history` |
| **search** | always | `find_files`, `grep`, `grep_context`, `recent_files`, `xref`, `find_symbol`, `replace` |
| **browser** | always | screenshots, URL inspection, network capture (Chrome MCP) |
| **ide** | IDE detected | `open`, `changes`, `info`, `recent_files`, `run_configurations` |
| **env** | always | `info`, `path`, `tools`, `vars`, `virtualenvs`, `which` |
| **network** | always | `dns`, `http_request`, `tcp_probe`, `websocket`, `config`, `ports` |
| **clipboard** | always | `read`, `write`, plus image / bp-node clipboard variants |
| **screenshot** | always | `capture`, `displays` |
| **cpp** | C++ project | C++ analysis (deps, includes, symbol graph) |

Per-plugin details: [plugins/index.md](plugins/index.md).

## Wire format (plugin ↔ core)

Each plugin process is started with stdio pipes. It implements the **MCP
protocol** as a server; core acts as the client. The agent's call flow:

```
agent → core: tools/call { name: "unreal_pie_start", args: { mode: "PIE" } }
core  → unreal plugin: tools/call { name: "pie_start", args: { mode: "PIE" } }
unreal plugin → UE editor (Python Remote Execution): runs helper script
unreal plugin → core: { content: [{ type: "text", text: <json> }] }
core  → agent: same envelope
```

Tool names are namespaced in the surface visible to the agent but the plugin
itself uses bare names — core does the prefix.

## Embedded Python helper (`sb_helpers.py`)

The unreal plugin ships a ~4000-line Python module via `go:embed`. Every
helper-backed MCP tool is a thin Go schema → dispatch into the bundled
`sb_helpers.<name>(...)` function. The Go side then forwards into the running
UE editor via Python Remote Execution.

The escape-hatch tool `unreal_run_python` accepts arbitrary Python; we prepend
`sb_helpers.py` so the agent's script can call any helper directly. Returned
data is extracted from `<<<SB_JSON>>>` markers (see
[unreal plugin reference](plugins/unreal.md#sb_json-marker-protocol)).

## C++ companion sub-plugin (Unreal only)

Some UE operations are pure C++ — Python can't reach `UEdGraphPin::DefaultObject`,
`FBlueprintEditorUtils::AddFunctionGraph`, `FMessageLogModule`, `TObjectIterator`,
etc. The Companion plugin (`SystemBridgeCompanion` in the user's UE engine or
project plugin folder) is a tiny Editor-only UE plugin that exposes static
`UFUNCTION`s as `unreal.SystemBridgeBindings.<method>(...)`. The Python helper
prefers the companion path when present and falls back to less-capable Python
paths when it's missing.

Source for the companion lives at `cmd/sb-unreal/companion/` and is embedded
into the sb-unreal binary via `go:embed`; `companion_install` extracts it to
the user's engine/project plugin dir and runs `RunUAT BuildPlugin`. See
[companion reference](unreal/companion.md) for the version timeline.

## Manifest schema (per-plugin)

```go
manifest.Manifest{
    Name:        "unreal",
    Version:     "0.1.0",
    Description: "Unreal Engine project introspection …",
    Author:      "github.com/o-codify/SystemBridge",
    Triggers: manifest.Triggers{
        FilesPresent: []string{"*.uproject"},
        Processes:    []string{"UnrealEditor.exe"},
    },
    Permissions: []string{
        "read_files:project", "read_processes",
        "execute_command:unreal_python", "network",
    },
    Lifecycle:   manifest.Lifecycle{Killable: manifest.KillableDeferred},
    EventBuffer: manifest.EventBuffer{MaxEvents: 200, MaxAgeSeconds: 600},
    SummaryCacheSeconds: 5,
    Tools: []manifest.ToolDecl{ {Name: "read_log", Description: "…"}, … },
    Events: []string{"unreal_log_warning", "unreal_log_error", …},
}
```

## Lifecycle

```
sb start
  └─ scan cwd for triggers
  └─ load matching plugin manifests
  └─ spawn each plugin process, hold stdio pipes
  └─ MCP server: tools list = union of all plugins' tools

discover() called
  └─ for each plugin: get Summary (cached up to SummaryCacheSeconds)
  └─ collect events from each plugin's ring (drain)
  └─ return combined map { plugin_name: { summary, events } }

tools/call <name>
  └─ route to plugin
  └─ enforce per-tool permission gates
  └─ stream progress notifications back (long-running build/test/PIE)

sb exit
  └─ Job Object (Windows) terminates all child plugin processes
  └─ each plugin runs its defer cleanup (e.g. python wrapper pool stop)
```

## Project-rooted by design

`sb` resolves all paths relative to the cwd it was started in. The
`pathUnder(target, cwd)` check rejects anything outside. `.senseignore` (same
syntax as `.gitignore`) lets a project opt sensitive files out of the
agent's view — checked in every read/write tool.

This is intentionally narrower than a general-purpose file MCP. A second
project means a second `sb` run (typically Claude Code spawns one per
project, see [installation](installation.md)).
