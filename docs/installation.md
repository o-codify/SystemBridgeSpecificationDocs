---
id: installation
title: Installation
status: stable
version: 26.601.1817
tags: [ installation, setup ]
---

# Installation

The repo's `sb.exe` is also the installer. From a freshly cloned and built
checkout:

```
scripts\build.sh       # produces bin\sb.exe + bin\plugins\*\*.exe + manifests
bin\sb.exe install --method claude-cli
```

This puts `sb.exe` on PATH (User scope on Windows), broadcasts the
environment change so other apps pick it up, and registers the daemon with
the chosen MCP client.

See also: [overview](README.md), [architecture](architecture.md).

## What `sb install` does

```
sb install [--method MODE] [--claude-config PATH] [--dry-run]
```

Modes:

| Method | Wire | Notes |
|---|---|---|
| `claude-cli` | `claude mcp add -s user systembridge <path-to-sb.exe>` | Claude Code's preferred path; writes to `~/.claude/mcp.json` via the official CLI so settings.json conventions are respected. |
| `json-file` | direct write to a JSON config (Claude Desktop) | Used for clients that don't ship a CLI helper. The path defaults to Claude Desktop's per-platform config (`%APPDATA%\Claude\claude_desktop_config.json` on Windows). |

Steps performed:

1. Resolve the running binary's absolute path (`os.Executable()`).
2. Ensure `<repo>/bin` is on the User PATH (Windows: `[Environment]::SetEnvironmentVariable('PATH', ..., 'User')` — this also broadcasts `WM_SETTINGCHANGE`, which `reg.exe` does not).
3. Register the MCP server via the chosen method.
4. Embed and write the `systembridge` skill into `~/.claude/skills/` so the
   agent picks it up cross-session.

The skill is shipped inside `sb.exe` (`go:embed` of
`cmd/sb/skill/systembridge/`). `--dry-run` prints all of the above without
mutating disk or registry.

## Verifying the install

In a fresh Claude Code session:

- The skill should be visible in the available-skills list ("`systembridge`").
- The `discover` tool, namespaced as `mcp__systembridge__discover`, should
  appear when SystemBridge is reachable.
- In a project (cwd has a `.git` / `*.uproject` / `*.go` etc.) `discover()`
  returns a per-plugin summary.

## Uninstall

There is no `sb uninstall` today; manual cleanup:

1. `claude mcp remove systembridge` (Claude Code) or remove the entry from
   the Desktop config file.
2. Remove the `~/.claude/skills/systembridge/` directory.
3. Remove the repo's `bin/` directory from PATH if desired.

## Project-side: nothing to install

SystemBridge does NOT require code in your project. The daemon binds to the
cwd it's launched in. Two optional opt-ins:

- **`.senseignore`** at project root — same syntax as `.gitignore`; matched
  paths are 403'd by read/write tools.
- **SystemBridgeCompanion** (UE only) — install via
  `unreal_companion_install` once per UE engine or project. Without it,
  Python-only fallbacks are used; many advanced tools are unavailable.
  Details in [companion reference](unreal/companion.md).

## Updating

`git pull && scripts/build.sh` rebuilds `bin/sb.exe` and all plugins. Claude
Code spawns a new `sb.exe` per session, so the next restart picks up the new
binary. For the Unreal companion plugin (which is built into UE's plugin
folder), see
[upgrading the companion plugin](unreal/companion.md#upgrading-the-companion).

## Troubleshooting installation

See [troubleshooting](troubleshooting.md) for:

- PATH didn't broadcast → other apps don't see `sb` on PATH (fix: log out / in,
  or check that PowerShell `[Environment]::SetEnvironmentVariable` was used,
  not `reg.exe`).
- "MCP transport closed" right after install → restart Claude Code; the new
  daemon binary needs a fresh stdio session.
- Multiple `sb.exe` instances running → harmless but indicates aborted prior
  sessions; kill all and restart Claude Code.
