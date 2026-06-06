---
id: runtime-invoke-pie
title: Runtime Invoke (PIE)
status: stable
version: 26.606.219
tags: [ unreal, pie, runtime, blueprint, automation, companion ]
---

# Runtime Invoke (PIE)

SystemBridgeCompanion v1.12.0+ exposes a `runtime_invoke` tool that
calls a Blueprint-exposed event or function on a live object in PIE —
the controlled pawn, a level actor, or one of their sub-components —
with typed arguments.

For the user-facing route — driving gameplay through input — see
[PIE input injection](pie-input.md). `runtime_invoke` is the direct
route: you specify the object and the function name, the daemon
allocates a parameter frame and calls `ProcessEvent`.

## Tool surface

| Tool | Purpose |
| ---- | ------- |
| `runtime_invoke` | Call a UFunction on a live PIE/level object. |

## Target selection

Pass ONE of:

| Parameter | Meaning |
| --------- | ------- |
| `player_index >= 0` | Controlled pawn of player N. |
| `actor_label` | Display label of an actor in the PIE/level world (use `unreal_level_actor_list` to enumerate). |

Optionally pass `component_name` to drill into a sub-component on the
chosen target — name is matched against the SCS variable name (e.g.
`"Mesh"`, `"InputBinder"`, `"Camera"`).

## Arguments

`args` is a flat list of strings, one per declared parameter in
declared order. Each string is parsed by `FProperty::ImportText` into
the parameter's reflected type — the same format
`bp_node_pin_set_default` accepts.

| UE type | Example arg |
| ------- | ----------- |
| scalar | `"2.5"`, `"42"`, `"True"` |
| `FName` | `"Hand_L"` |
| `FString` | `"Hello"` |
| `FVector` | `"(X=1.0,Y=0.0,Z=-90.0)"` |
| `FRotator` | `"(Pitch=0,Yaw=90,Roll=0)"` |
| `FTransform` | `"(Rotation=(...),Translation=(...),Scale3D=(...))"` |
| enum | enum entry name |
| object / class | asset / class path |

Out-params and return values are read back after the call; the
return value (if any) is reported in `output` as exported text.

## Examples

### Call a parameterless event on the controlled pawn

```python
unreal_runtime_invoke(
    player_index=0,
    function_name="StartReload",
)
# → {success: true, target: "BP_Hero_C_0", function: "StartReload", output: ""}
```

### Call a function with typed args

```python
unreal_runtime_invoke(
    player_index=0,
    function_name="TeleportPlayer",
    args=[
        "(X=500.0,Y=0.0,Z=200.0)",         # NewLocation: FVector
        "(Pitch=0,Yaw=90,Roll=0)",         # NewRotation: FRotator
        "False",                            # bSweep: bool
    ],
)
```

### Read a value back

```python
result = unreal_runtime_invoke(
    player_index=0,
    function_name="GetAmmoInMagazine",
)
# result["output"] is the return value exported as text, e.g. "30".
```

### Target a sub-component

```python
unreal_runtime_invoke(
    actor_label="BP_DamageZone_1",
    component_name="DamageEmitter",
    function_name="Activate",
    args=["True"],                          # bReset: bool
)
```

### Target by actor label

```python
unreal_runtime_invoke(
    actor_label="BP_Vendor_C_2",
    function_name="OpenShop",
)
```

## Errors

| Error | Meaning |
| ----- | ------- |
| `companion_unavailable` | Companion v1.12.0+ not loaded. |
| `target_not_found` | Neither `player_index` nor `actor_label` resolved a live object. |
| `output` is empty for a non-void function | Function returned its default-constructed return value (e.g. argument parse failed). Check arg types against the function signature. |

## How it works

1. Resolve target (player pawn / level actor / component).
2. `Target->FindFunction(FunctionName)` → `UFunction*`.
3. Allocate a parameter frame on the stack; zero + initialize each
   `FProperty` to its default.
4. Walk input params; for each, `FProperty::ImportText_InContainer`
   on the supplied arg string.
5. `Target->ProcessEvent(Func, ParmsMem)`.
6. Read the return param via `ExportText_InContainer`; destroy
   parameters.

This is the same path the editor's "Call In Editor" button uses, with
explicit args instead of a property panel.

## Cross-references

- [companion plugin](companion.md) — v1.12.0 entry.
- [PIE input injection](pie-input.md) — alternative route via the
  input system.
- [PIE workflow](pie-workflow.md) — the surrounding lifecycle.
