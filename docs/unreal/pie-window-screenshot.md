---
id: no-dedicated-tool-captures-the-pie-game-window-editor-take-screenshot-grabs-the-editor-viewport-pie-console-command-returns-no-world
title: No dedicated tool captures the PIE game window (editor_take_screenshot grabs the editor viewport; pie_console_command returns no_world)
status: request
version: 26.616.259
tags: [ unreal, screenshot, pie, take_screenshot, pie_console_command, request ]
---

# Capturing the PIE game window has no dedicated tool

When PIE plays in a **separate game window** (the common "New Editor Window (PIE)"
play setting), there is no SystemBridge tool that captures that window. The two
tools that look like they should each fail in a different way, and the only thing
that works is a raw `HighResShot` console command pushed into the PIE world via
`unreal_run_python`. This makes visual verification of PIE-only state (anything an
actor builds in `BeginPlay`, procedural components, gameplay spawns) effectively
impossible through the dedicated tools.

## Context / why it matters

A C++ `USceneComponent` builds its visual primitives in `BeginPlay`, so the actor
is **only** posed in the PIE world — in the editor world it is an empty,
unbuilt actor. To verify it, you must screenshot the running PIE game. With the
current tools the only image you can get back is the editor level viewport, which
shows the unbuilt (invisible) editor-world copy. Every screenshot looked
identical and ignored PIE-side camera changes, which is what surfaced the gap.

## Tool-by-tool behaviour observed (UE 5.7.4, companion 1.13.6)

### 1. `unreal_editor_take_screenshot` → captures the EDITOR viewport, not PIE

- The tool's own description says *"WORKS IN PIE MODE … PIE mode would render
  synchronously inside the script."* In practice, with PIE in a separate window,
  it saved a shot of the **editor level viewport** (fixed editor camera), not the
  PIE client viewport.
- Proof: while PIE was running I repositioned the **PIE** player camera
  (`set_control_rotation` + pawn teleport, confirmed via
  `PlayerCameraManager.get_camera_location/rotation` pointing at the test actor).
  Every `unreal_editor_take_screenshot` came back with the **same** editor-camera
  framing, unaffected by the PIE camera. The PIE-only actor never appeared.
- Files landed in `Saved/Screenshots/WindowsEditor/sb_<ts>.png`.

### 2. `unreal_pie_console_command` → `{"error":"no_world","success":false}`

- Returns `no_world` **even while PIE is active**. Confirmed active in the same
  moment: `unreal_editor_status` → `pie_active:true`, and
  `UnrealEditorSubsystem.get_game_world()` resolved a valid PIE world
  (`gw != get_editor_world()`).
- So the command never reaches the PIE viewport, and `HighResShot` (or any other
  console command meant for the running game) can't be issued through this tool.

## What actually works (the workaround)

Issue the console command into the resolved PIE world from `unreal_run_python`:

```python
ues = unreal.get_editor_subsystem(unreal.UnrealEditorSubsystem)
gw  = ues.get_game_world()                       # the PIE world while playing
unreal.SystemLibrary.execute_console_command(gw, "HighResShot 1920x1080")
```

This renders and saves the **PIE client viewport** (the separate game window).
Output is `Saved/Screenshots/WindowsEditor/HighresScreenshot#####.png` with an
auto-incrementing index (NOT the filename the editor-screenshot tool reports). To
get a clean shot of a PIE-only actor I also had to hide the possessed player pawn
(`pawn.set_actor_hidden_in_game(True)`) and drive the PIE camera with
`find_look_at_rotation` — all via `run_python`, none of it exposed as tools.

## Likely root cause

- `unreal_editor_take_screenshot` targets the editor's active viewport client
  (`AutomationLibrary`/editor `FScreenshotRequest`) rather than the PIE/game
  viewport client. When PIE is its own window these are different viewport
  clients, so the editor one is grabbed.
- `unreal_pie_console_command` resolves its world differently from
  `UnrealEditorSubsystem.get_game_world()` and comes up empty for separate-window
  PIE — it should resolve and execute against the active PIE world the same way
  Python does.

## Requested behaviour

1. **A dedicated PIE-window screenshot path.** Either a new
   `unreal_pie_take_screenshot` tool, or a `target: "pie" | "editor"` parameter on
   `unreal_editor_take_screenshot`, that captures the **active PIE client
   viewport** (separate window or in-viewport PIE alike). Internally this is the
   `execute_console_command(PIE_world, "HighResShot WxH")` path shown above, or
   `FScreenshotRequest` against the PIE viewport client.
2. **Return the real saved path.** For the PIE/`HighResShot` path the actual file
   is the auto-incremented `HighresScreenshot#####.png`; the tool should return
   that resolved path (resolve the newest file after the request settles), not a
   requested-but-unused filename.
3. **Fix `unreal_pie_console_command` `no_world`.** It should resolve the active
   PIE world (matching `UnrealEditorSubsystem.get_game_world()`) and execute there
   while `pie_active:true`. Today it is unusable for any in-game console command
   in separate-window PIE.
4. (Nice to have) An option to **suppress the possessed player pawn / HUD** for a
   clean capture, since verifying spawned/AI actors usually means the player body
   is between the camera and the subject.

## Environment

- UE 5.7.4 (`5.7.4-51494982+++UE5+Release-5.7`), project `ALS_UltimateWarfare`.
- SystemBridgeCompanion 1.13.6; PIE started via `unreal_pie_start`
  (`"Real PIE with Pawn possession (companion path)"`), playing in a separate
  game window.
- Verified working capture: `execute_console_command(get_game_world(),
  "HighResShot 1920x1080")` → `Saved/Screenshots/WindowsEditor/HighresScreenshot00002.png`.
