---
id: pie-input-injection
title: PIE Input Injection
status: stable
version: 26.606.219
tags: [ unreal, pie, input, enhanced-input, automation, companion ]
---

# PIE Input Injection

SystemBridgeCompanion v1.12.0+ exposes input injection into the running
PIE session — both Enhanced Input actions (by `UInputAction` asset) and
raw keys (by `FKey` name). The same mechanism the editor uses when a
human presses a key.

Use it to exercise input-driven gameplay headlessly: equip a weapon,
fire, reload, interact, walk — without a person at the keyboard and
without depending on which physical key a binding uses.

## When to reach for this

- End-to-end behaviour tests: trigger → observe state.
- Verifying a Blueprint's `OnInputAction(Reload)` handler.
- Driving a PIE walkthrough script.

For lower-level cases — calling a specific Blueprint event/function
directly without going through input — see [runtime invoke](runtime-invoke.md).

## Tool surface

| Tool | Purpose |
| ---- | ------- |
| `pie_input_inject` | Inject one input event. Pass `action_path` (Enhanced Input) OR `key_name` (raw FKey). |

Sends to a chosen `player_index` (default 0). Detects the PIE world
automatically via `GEditor->GetWorldContexts()` — no PIE → returns
`success: false`.

## Enhanced Input — `action_path`

Pass the UInputAction asset path:

```
pie_input_inject(action_path="/Game/Input/IA_Reload.IA_Reload")
# → fires the IA_Reload action with default scalar 1.0.
```

For Axis1D actions, set `amount`:

```
pie_input_inject(
    action_path="/Game/Input/IA_Walk.IA_Walk",
    amount=0.75,
)
```

For Axis2D / Axis3D actions, pass `value_vector`:

```
pie_input_inject(
    action_path="/Game/Input/IA_Look.IA_Look",
    value_vector=[120.0, 5.0, 0.0],      # mouse delta x/y/0
)
```

Behind the scenes:
`ULocalPlayer::GetSubsystem<UEnhancedInputLocalPlayerSubsystem>()` →
`InjectInputForAction(Action, FInputActionValue, mods, triggers)`.
Modifiers / triggers default to empty — Enhanced Input narrows the
value to the action's declared `ValueType`.

## Raw keys — `key_name`

Pass an FKey name and an event:

```
pie_input_inject(
    key_name="SpaceBar",
    event="pressed",
)
pie_input_inject(
    key_name="SpaceBar",
    event="released",
)
```

`event` values:

| `event` | Equivalent FInputEvent |
| ------- | ---------------------- |
| `pressed` (default) | `IE_Pressed` |
| `released` | `IE_Released` |
| `repeat` | `IE_Repeat` |
| `doubleclick` | `IE_DoubleClick` |

For axis-like keys (gamepad stick, mouse), pass `amount` (pressure /
analog value 0–1).

FKey names follow UE's convention:
`SpaceBar`, `LeftMouseButton`, `MouseScrollUp`, `Gamepad_FaceButton_Bottom`,
`Gamepad_LeftStick_Up`, etc.

Behind the scenes: `APlayerController::InputKey(FInputKeyParams)` — same
entry point Slate routes physical key events through.

## Held keys

Pressing + releasing in one tool call is two invocations:

```
pie_input_inject(key_name="W", event="pressed")
# … move forward for 0.5s …
pie_input_inject(key_name="W", event="released")
```

Use [`pie_run_and_watch`](pie-workflow.md) or `tasks_start` to interleave
input with observation.

## End-to-end recipe — verify reload

```python
# 1. Start PIE.
unreal_pie_start()

# 2. Inject the reload action.
unreal_pie_input_inject(
    action_path="/Game/Input/IA_Reload.IA_Reload",
)

# 3. Observe state via runtime_invoke or actor_transform_query.
result = unreal_runtime_invoke(
    player_index=0,
    function_name="GetAmmoInMagazine",
)
# → result.output is "30" (or whatever the magazine size is).
```

## Errors

| Error | Meaning |
| ----- | ------- |
| `companion_unavailable` | Companion v1.12.0+ not loaded — call `companion_status` / `companion_rebuild`. |
| `neither_action_nor_key_supplied` | Pass `action_path` or `key_name`. |
| `input_action_not_found` | The asset path didn't resolve to a `UInputAction`. |
| `success: false`, kind: enhanced | No PIE world / no LocalPlayer / no EnhancedInput subsystem. Make sure PIE is running and the project uses Enhanced Input. |
| `success: false`, kind: key | No PIE PlayerController or the FKey name is invalid. |

## Cross-references

- [companion plugin](companion.md) — v1.12.0 entry.
- [runtime invoke](runtime-invoke.md) — direct event/function call when
  you'd rather not go through input.
- [PIE workflow](pie-workflow.md) — the surrounding lifecycle
  (start / watch / stop).
