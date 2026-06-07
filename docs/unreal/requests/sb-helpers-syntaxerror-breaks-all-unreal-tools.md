---
id: bug-sb-helpers-py-syntaxerror-line-6776-breaks-all-unreal-tools
title: "BUG: sb_helpers.py SyntaxError (line ~6776) breaks ALL unreal_* tools"
status: request
version: 26.607.2326
tags: []
---

# BUG: sb_helpers.py SyntaxError breaks ALL unreal_* tools

**Severity:** blocker — the entire SB↔Unreal bridge is down. Every `unreal_*`
tool (and `unreal_run_python`, `unreal_editor_status`, companion ping) fails
identically, because `sb_helpers.py` is auto-prepended to every script and now
fails to compile.

## Symptom

Any unreal tool returns:

```
{"success":false,"result":"SyntaxError: ':' expected after dictionary key (<string>, line 6776)\n              argument_field := arg_name: path,\n\n                           ^", ...}
```

`unreal_companion_status` → `loaded:false`, `load_probe_error:"companion ping returned no parsed_json"` (same root cause — the ping runs through the broken preamble).

The editor itself is healthy and reachable (the transport returns valid `node`
info: engine 5.7.4, correct project), so this is purely the prepended helper
failing to compile.

## Root cause

`cmd/sb-unreal/sb_helpers.py`, in the `asset_not_found` branch of the
asset-resolver helper (the function whose sibling branches build
`missing_argument` / `wrong_class` dicts):

```python
    obj = _load_asset_robust(path)
    if obj is None:
        return {
            "success": False,
            "error": "asset_not_found",
            argument_field := arg_name: path,          # <-- invalid: walrus as a dict key
            "hint": "Verified failure: EAL + AssetRegistry + StaticLoadObject ...",
        }
```

`argument_field := arg_name: path` is not valid Python (an assignment
expression cannot be a dict key followed by `:`), so the WHOLE ~7699-line
prepended helper fails to compile → every tool that prepends it breaks.

## Proposed fix

Match the sibling branches (`missing_argument` uses `"argument": arg_name`;
`wrong_class` uses `"argument": arg_name` + `"asset_path": path`). Replace the
bad line with those two keys:

```python
            "success": False,
            "error": "asset_not_found",
            "argument": arg_name,
            "asset_path": path,
            "hint": "Verified failure: EAL + AssetRegistry + StaticLoadObject ...",
```

`py_compile` confirms the file is valid Python after this change.

## Deployment note

`sb_helpers.py` is embedded via `//go:embed sb_helpers.py` (see
`cmd/sb-unreal/python_remote.go`). The fix therefore requires **rebuilding the
sb-unreal binary and restarting the daemon** to take effect (restart drops the
MCP transport — see `docs/mcp-transport-auto-recovery.md`).

## Suggested guard (optional)

A pre-ship `py_compile`/import smoke-test of `sb_helpers.py` in CI (or at
daemon startup, returning a clear "helper failed to compile" error instead of
leaking the SyntaxError into every tool result) would turn a total outage into
a single obvious failure.
