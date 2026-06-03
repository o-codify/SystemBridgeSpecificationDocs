---
id: request-sb-mcp-transport-auto-recovery-after-drop-daemon-restart
title: "Request: SB MCP transport auto-recovery after drop / daemon restart"
status: request
version: 26.603.1942
tags: [ mcp, transport, reliability, daemon, request ]
---

# Request: SB MCP transport auto-recovery after drop / daemon restart

## Status

`request` — reliability gap observed in the field (2026-06-03/04). Filed by AI
agent. Not project-specific; affects any SystemBridge client session.

## Problem

When the SystemBridge MCP transport (the connection between the MCP client and
the SB MCP server / daemon) drops, it does **not** re-establish itself
automatically. Every subsequent SB tool call then fails hard with:

```
MCP error -32603: transport error: transport closed
```

…even though **both ends are alive**: the UE editor is running, the SB
companion is loaded, and the client (Claude Desktop) is running. The only way
back is manual intervention (user re-triggers / reconnects). The agent is fully
blocked from all `sb-*` / `unreal_*` / spec tools in the meantime.

## When it happens

Reproduced during a **companion version bump / daemon restart**
(v1.10.0 → v1.10.1): the daemon restarted to load the fix, the MCP transport
closed, and it stayed closed. `unreal_editor_status` returned
`transport error: transport closed` on repeated calls until the user manually
restored the connection; only then did the same call succeed
(`companion_version: 1.10.1`). Likely also triggers on any daemon crash/restart,
socket timeout, or the editor restart cycle the SB tools themselves perform
(`unreal_editor_restart` / `editor_launch`).

## Impact

- A self-inflicted SB operation (companion upgrade, `editor_restart`) can leave
  the agent unable to use ANY SB tool, with no automatic path back.
- Errors are opaque (`transport closed`) — no signal that a reconnect is in
  progress or possible.

## Requested behavior

1. **Auto-reconnect with backoff.** When the MCP transport drops, the SB MCP
   server (and/or the client transport layer) should automatically attempt to
   re-establish the connection (retry with backoff for a bounded window)
   instead of staying closed.
2. **Survive daemon/companion restarts.** After a daemon restart or companion
   version bump, the transport should come back on its own once the daemon is
   listening again — no manual step.
3. **Transparent retry for in-flight calls.** A tool call that hits a transient
   transport drop should wait for reconnect (up to a timeout) and retry once,
   rather than failing immediately.
4. **Clear status on failure.** If reconnect can't be achieved, return a status
   that says "transport reconnecting / daemon unreachable, retry in Ns" rather
   than a bare `transport closed`, so the agent can wait+retry instead of
   treating it as fatal.

## Acceptance

- Bump the companion (daemon restart) mid-session: within ~N seconds the next
  SB tool call succeeds automatically, no user action.
- Kill/restart the SB daemon: the transport re-establishes on its own.
- A transient drop during a single tool call is retried transparently and
  succeeds.

## Workaround today

User manually re-establishes the MCP connection; the agent polls
`unreal_editor_status` until it stops returning `transport closed`.
