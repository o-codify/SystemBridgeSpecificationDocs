---
id: live-coding
title: Live Coding
status: stable
version: 26.610.1545
tags: [ unreal, live-coding, cpp ]
---

# Live Coding

`unreal_live_coding_compile` — synchronous incremental C++ compile in
the running editor. Equivalent to Ctrl+Alt+F11 in the editor toolbar,
but headless and with full result reporting.

See: [unreal deep dive](index.md),
[C++ iteration workflow](../workflows/cpp-iteration.md).

## What it does

```
unreal_live_coding_compile [hide_console=true] [skip_action_limit=false] [wait_seconds=120]
```

Five steps:

1. **`hide_console`** (default true) — persist
   `Startup=AutomaticButHidden` into the project's
   `Saved/Config/WindowsEditor/EditorPerProjectUserSettings.ini` under
   `[/Script/LiveCoding.LiveCodingSettings]`. UE applies the `-Hidden`
   flag when spawning `LiveCodingConsole.exe` at editor startup.
   **Takes effect on the NEXT editor restart** (UE reads the .ini at
   startup only). The patch is idempotent.

2. **`skip_action_limit`** (default false) — taskkill
   `LiveCodingConsole.exe` BEFORE the compile to reset its per-session
   15-patch counter. UE re-spawns it cold (~2-5s slower next compile)
   but the agent never hits the "Disable action limit for this session"
   popup.

3. **Snapshot** the editor log size before dispatching.

4. **Fire** `LiveCoding.Compile` via Python (`execute_console_command`)
   in the running editor.

5. **Poll** the log tail from the snapshot point at 500ms intervals.
   Look for `Live coding succeeded` or `Live coding failed` markers
   (regex). On match: parse the new log segment for MSBuild-style
   `error CXXNN:` / `warning CXXNN:` lines and return structured.

Default `wait_seconds=120`, range 5..1800.

## Returned result

```jsonc
{
  "success": true,
  "status": "succeeded",                  // succeeded | failed | timeout | blueprint_only_project | trigger_failed
  "duration_ms": 7727,
  "wait_ms": 7727,
  "cpp_modules": ["GameAnimationSample"],

  // markers
  "started_at": "[2026.05.04-11.58.07:384]",
  "completed_at": "2026-05-04T11:58:14Z",
  "success_line": "[2026.05.04-11.58.14:417][716]LogLiveCoding: Display: Live coding succeeded",
  "fail_line": "",                         // populated when status=failed
  "errors":   ["foo.cpp(42): error C2065: ..."],
  "warnings": ["bar.cpp(7): warning C4101: ..."],
  "log_excerpt": ["...last 80 lines from the new log section..."],
  "new_log_bytes": 1232,

  // side-actions
  "hide_console_applied": false,            // already_set on this project
  "hide_console_status": "already_set",     // or "applied" | "error"
  "live_coding_console_killed": 0           // count if skip_action_limit fired
}
```

## Status values

| Status | Meaning |
|---|---|
| `succeeded` | Live coding marker hit, no errors. The editor's in-memory C++ is patched. |
| `failed` | Live coding marker hit, with errors. Parse `errors[]` to fix. |
| `timeout` | No marker within `wait_seconds`. Common causes listed in the response `hint`. |
| `blueprint_only_project` | Pre-check refused: no `*.Build.cs` files under `Source/`. |
| `trigger_failed` | Python helper failed to dispatch. |

## hide_console honestly

The legacy implementation tried to set
`unreal.LiveCodingSettings.Startup = AutomaticButHidden` via
`set_editor_property`. But `unreal.LiveCodingSettings` is NOT exposed in
UE 5.7's Python bindings — `hasattr(unreal, "LiveCodingSettings")` is
False. The Python path failed silently.

The fix is to write the .ini directly from Go. The .ini file is the same
file UE itself reads at editor startup:

```
<Project>/Saved/Config/WindowsEditor/EditorPerProjectUserSettings.ini

[/Script/LiveCoding.LiveCodingSettings]
Startup=AutomaticButHidden
```

After this is written, the next editor launch spawns
`LiveCodingConsole.exe` with the `-Hidden` flag → no popup window.

`hide_console_status: "already_set"` means the .ini was already correct
(idempotent no-op). `"applied"` means we wrote it just now;
`hide_console_note` mentions the restart requirement.

## skip_action_limit honestly

The action limit (default 15 patches per LCC session) is a counter
inside `LiveCodingConsole.exe`. There is:

- NO persistent setting.
- NO console var.
- NO Python API to flip it.

The popup with "Disable action limit for this session" sets in-memory
state inside that LCC process that survives only until LCC exits.

The only honest bypass: **kill `LiveCodingConsole.exe` before the next
compile**. UE auto-respawns it cold; the counter starts fresh at 0/15.
The cost is one cold start, ~2-5s of extra time.

## Patterns

- **Iterative C++ tweak.** Run `live_coding_compile` after every edit;
  ~5s warm path. If `status: failed`, parse `errors[]`, fix, retry.
  See [C++ iteration](../workflows/cpp-iteration.md).

- **Long session, action limit creeping.** When you've done ~10
  compiles and want the popup gone for the next batch, pass
  `skip_action_limit=true` once. The cold restart pays a one-time cost.

- **Need a real link error report** (cross-module). `project_build`
  is the friend: full UBT pass, structured `errors[]`. Live Coding
  patches the editor in-place but can't catch global issues a full
  link would.

## Limits + when to fall back

- Adding a brand-new UCLASS / extending UCLASS reflection: Live
  Coding sometimes can't patch. `project_build` then `editor_restart`.
- Adding a new Module to the project: `project_build`.
- Headless / CI: use `project_build` (no editor required).
- Iterating method body / inlining changes: Live Coding is fine.

## Cross-references

- [run_cpp](run-cpp.md) — one-off C++ snippets in the live editor; uses
  this compile cycle under the hood.
- [C++ iteration workflow](../workflows/cpp-iteration.md) — end-to-end
  edit / compile / verify loop.
- [unreal plugin tool catalog](../plugins/unreal.md#build--test-out-of-process).
- [troubleshooting](../troubleshooting.md#live-coding-timeout).
