---
id: mcp-transport-avoiding-daemon-restart-drops
title: "MCP transport: avoiding daemon-restart drops"
status: stable
version: 26.603.2027
tags: [ mcp, transport, reliability, daemon, companion ]
---

# MCP transport: avoiding daemon-restart drops

The MCP transport between the client (Claude Desktop) and the
SystemBridge daemon (`sb.exe`) is stdio. When `sb.exe` exits — typically
because the user rebuilt it to bump an embedded constant — the parent's
stdio pipe closes. Claude Desktop's MCP client decides whether to
re-spawn; on a manual restart of the client itself it does, but a
mid-session drop has historically required the user to manually restore
the connection.

The fix shipped with v1.11.0 sidesteps this for the most common cause:
companion-version bumps.

## What changed in v1.11.0

`expectedCompanionVersion` was a Go `const` baked into the `sb-unreal`
binary. Every time the companion DLL was rebuilt to a new version, the
user had to:

1. Edit the Go source.
2. Rebuild `sb-unreal.exe`.
3. Restart Claude Desktop so the new binary loaded.

Step 3 closed the stdio pipe — the reported "transport closed" failure
cascade.

v1.11.0 reads the expected version from a sidecar file
`expected_companion_version.txt` placed next to the `sb-unreal` binary,
falling back to the embedded `defaultExpectedCompanionVersion` if the
sidecar is absent. The sidecar is read on demand, mtime-cached.

The new upgrade flow is:

```
# 1. Rebuild the DLL through UAT.
"<UE>/Engine/Build/BatchFiles/RunUAT.bat" BuildPlugin \
    -Plugin=<plugin>.uplugin -Package=<tmp> -Rocket -TargetPlatforms=Win64

# 2. Sync Binaries/ + Intermediate/ into the installed plugin path.

# 3. Edit the sidecar to match (no sb.exe rebuild).
echo 1.11.0 > <path-to-sb-unreal-binary-dir>/expected_companion_version.txt

# 4. Restart UE so the new DLL loads.
```

No Claude Desktop restart needed; no MCP transport drop.

## Sidecar file format

- Plain text, first non-empty / non-comment line wins.
- Lines starting with `#` are ignored.
- Trailing whitespace and `\r` are stripped.
- Absent / empty / unreadable → falls back to the embedded default.

```
# expected_companion_version.txt
1.11.0
```

## What this does NOT fix

Genuine MCP-client auto-reconnect after a parent-process stdio close is
the responsibility of the MCP client (Claude Desktop), not the
SystemBridge daemon. If the user kills `sb.exe` directly (or it crashes),
the client still needs to re-spawn it — and depending on client version
that may or may not be automatic.

The sidecar pattern simply avoids the *need* to restart `sb.exe` for the
most common cause: companion version bumps.

## Cross-references

- [companion plugin](unreal/companion.md) — version timeline; the
  sidecar pairs with the version drift detection in
  `companion_status`.
- [troubleshooting](troubleshooting.md#companion-drift) — what to do
  when versions diverge.
