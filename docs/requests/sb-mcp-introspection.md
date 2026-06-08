---
id: request-mcp-self-introspection-servers-tools-call-audit
title: "Request: MCP self-introspection (servers / tools / call / audit)"
status: request
version: 26.608.1427
tags: [ sb-core, mcp, introspection, request ]
---

# Request: MCP self-introspection

**Origin.** v1.14.0 hotfix. To verify the `sb_helpers.py` fix worked
end-to-end I had to hand-craft a JSON-RPC stdin stream to `sb.exe` to
list tools + ping the editor. The MCP debugging surface should not
require shellouts.

## What

A small set of tools that let the AI inspect the MCP layer itself —
including SB's own.

## Tools

| Tool | Purpose |
|---|---|
| `mcp_servers_list()` | List every connected MCP server (in this session or known to the host). Per server: `{name, transport, status, uptime_ms, tool_count, last_error?}`. |
| `mcp_tools_list(server?)` | Flat inventory of tools across servers (or one). Per tool: `{server, name, description, params_schema?, dangerous?}`. |
| `mcp_call(server, tool, args)` | Invoke an arbitrary tool on an arbitrary connected server. Returns the raw result. Useful for testing / one-offs. |
| `mcp_audit_search(pattern, since?, until?, limit=50)` | Grep the audit log (`~/.systembridge/logs/audit.log`) with optional time range. Returns `{entries: [{ts, server, tool, status, duration_ms, args_summary, error?}]}`. |
| `mcp_stats(window="1h")` | Per-tool call counts + median/p95 duration + error rate over a window. Same data `sb stats` shows on CLI. |

## Why

- **Diagnostics.** When a tool stops working, "is the MCP server up?"
  + "is this tool registered?" is the first triage step. Currently
  requires shellouts.
- **Discovery.** `mcp_tools_list("notion")` answers "what can I do
  with notion?" without making the AI hunt through documentation.
- **Cross-server orchestration.** `mcp_call("hls", "list_docs")` lets
  the AI call into a peer MCP without the parent harness having to
  load that tool's schema first.
- **Post-mortem.** "Why did the last unreal call fail?" → `mcp_audit_search("pattern")`
  instead of opening the log file.

## Implementation route

These live in **sb core**, not as a plugin — the data is already in
sb.exe's process state (active plugin connections, tool registry,
audit logger):

- `mcp_servers_list` / `mcp_tools_list` — read directly from
  `pluginManager.Active()` and the registered tool registry.
- `mcp_call` — re-uses the existing dispatch path.
- `mcp_audit_search` — opens the audit log read-only with `Tail`-
  style seeking.
- `mcp_stats` — wraps the existing `sb stats` aggregator.

No new dependencies. ~3-4 hours.

## Acceptance

- `mcp_servers_list()` returns SB itself + every other server the host
  has connected, distinguishing "in-process plugin" vs "remote MCP".
- `mcp_tools_list("unreal")` returns the same 199 tools `sb test unreal`
  enumerates.
- `mcp_call("systembridge", "files_stat", {path: "go.mod"})` returns
  the same result as the direct tool.
- `mcp_audit_search("unreal_run_python", since="1h")` returns recent
  invocations with timing.

## Non-goals (defer)

- Live tail of MCP traffic (stream, not query) — stateful.
- Resource enumeration (`mcp_resources_list`) — MCP resources are less
  used; add when there's demand.
- Cross-host federation (calling tools on remote MCP servers) — out of
  scope.
