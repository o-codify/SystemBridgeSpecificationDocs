---
id: request-niagara-emitter-module-authoring-currently-info-only
title: "Request: Niagara emitter module authoring (currently info-only)"
status: request
version: 26.608.1429
tags: [ unreal, niagara, vfx, request ]
---

# Request: Niagara emitter module authoring

**Origin.** v1.x ships `niagara_system_info` (read) +
`niagara_set_user_parameter` (very limited write). The actual
emitter — module stack, scratch pads, sim stages — is unauthorable
headless today.

## What

The minimum write surface to author a basic emitter end-to-end:
spawn rate / lifetime / velocity / colour over time.

## Tools

### Read

| Tool | Purpose |
|---|---|
| `niagara_emitter_list(system_path)` | List emitters in a system: `{name, enabled, sim_target, gpu}`. |
| `niagara_modules_list(system_path, emitter_name)` | List modules per stack (`SystemSpawn`, `EmitterSpawn`, `EmitterUpdate`, `ParticleSpawn`, `ParticleUpdate`, etc.): `[{stack, index, script_path, enabled}]`. |
| `niagara_module_inputs(system_path, emitter_name, stack, index)` | List the input bindings on one module: `[{name, type, value_summary, linked_to?}]`. |

### Write

| Tool | Purpose |
|---|---|
| `niagara_module_add(system_path, emitter_name, stack, script_path, position=-1)` | Add a module script (FNiagaraScript). |
| `niagara_module_remove(system_path, emitter_name, stack, index)` | Remove. Idempotent. |
| `niagara_module_set_input(system_path, emitter_name, stack, index, input_name, value)` | Set an input value. Typed coerce mirrors `bp_variable_set_default`. |
| `niagara_emitter_set_property(system_path, emitter_name, property, value)` | Top-level emitter properties (Lifetime, SpawnRate, GpuComputeSim, etc.). |
| `niagara_compile(system_path)` | Compile + save. |

## Why

Niagara is the runtime VFX system. Currently the AI can read a system
but to author one it has to: hand-author the `.uasset` file
(out-of-scope), or invoke editor through GUI Slate (unreliable). The
result is that an agent setting up muzzle flash / impact / footstep
VFX has to ask a human to "go click in the editor".

## Implementation route

Companion C++ — Niagara's UObject API is editor-side and not
Python-bound:

- `UNiagaraSystem::GetEmitterHandles` for read.
- `UNiagaraEmitterEditorData` + `UNiagaraStackEditorData` for module
  stack manipulation.
- `UNiagaraScriptStack::AddScript` / `RemoveScript`.
- Input bindings live on `UNiagaraNodeFunctionCall` parameters.

Module deps: `Niagara`, `NiagaraEditor`, `NiagaraShader`.

## Acceptance

- `niagara_module_add("/Game/VFX/NS_Test", "Burst",
  "ParticleSpawn", "/Niagara/CommonModules/InitializeParticle")`
  produces a working spawn module.
- `niagara_module_set_input(..., "Lifetime", 0.5)` applies and saves.
- After `niagara_compile`, the system previews in the Niagara editor
  with the authored behaviour.

## Non-goals (defer)

- Scratch-pad authoring (custom HLSL inside the system).
- Data interface bindings (skeletal-mesh-driven systems, GPU mesh
  particles).
- Niagara renderers (mesh / ribbon / light renderer customisation).
