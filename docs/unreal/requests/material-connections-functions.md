---
id: request-material-connections-read-material-functions-authoring
title: "Request: material connections read + material functions authoring"
status: request
version: 26.608.1429
tags: [ unreal, material, authoring, request ]
---

# Request: material connections + material functions

**Origin.** Material authoring shipped in companion v0.7, but two
gaps remain that real shader work hits:

1. **Connections read**: `bp_node_pin_links` exists for K2 graphs,
   but `material_expression_*` has no equivalent — you can list
   expressions and add edges, but can't enumerate "what is plugged
   into this Lerp's A input?".
2. **Material functions** (`MF_*`): currently no authoring at all —
   can't create/list/edit Material Function assets, which are the
   primary reuse mechanism in any non-trivial material library.

## What

### Read

| Tool | Purpose |
|---|---|
| `material_expression_pin_links(mat_path, expression_guid, pin_name)` | For one input/output pin on one expression, returns `[{other_expression_guid, other_pin_name, direction}]`. Mirror of `bp_node_pin_links`. |
| `material_connections_list(mat_path)` | Dump every connection in the material as `[{from_guid, from_pin, to_guid, to_pin}]`. For audit / diff / refactor. |

### Material functions (new asset type)

| Tool | Purpose |
|---|---|
| `material_function_create(path, inputs?: [], outputs?: [])` | Create a `UMaterialFunction` asset, optionally pre-populated with input/output nodes. |
| `material_function_expressions_list(func_path)` | Same shape as `material_expressions_list` but for the MF graph. |
| `material_function_expression_add(func_path, class, x, y)` | Add an expression node inside the MF. |
| `material_function_compile(func_path)` | Recompile. |
| `material_function_call_node_add(mat_path, func_path, x, y)` | Place a MaterialFunctionCall in a material that references the MF. |

## Why

- "Audit which Lerps in `M_HeroBase` use the wrong A input" → currently
  impossible headless; the read tools can list expressions but not
  edges.
- "Migrate a 30-material library to use a shared `MF_HeroPalette`" →
  needs both function authoring AND the call-node binding tool.
- Symmetry with K2 — same `bp_node_pin_links` mental model.

## Implementation route

Material connection read sits naturally in Python via
`MaterialEditingLibrary.get_inputs_for_material_expression` /
`get_outputs_for_material_expression` (already used internally for
the existing connect tools — read path was just never surfaced).

Material function authoring also rides on `MaterialEditingLibrary` —
`UMaterialFunction` extends `UMaterialFunctionInterface` which has
the same `Expressions` collection. Companion C++ may help for
`UMaterialFunction::FunctionExpressions` access if Python proves
insufficient on UE 5.7.

## Acceptance

- For a known material with a Lerp whose A is bound to a
  TextureSample, `material_expression_pin_links(mat, lerp_guid,
  "A")` returns `[{other_expression_guid: <TextureSample>, other_pin_name:
  "RGB"}]`.
- `material_function_create("/Game/MF_Test", inputs: [{name:"In",
  type:"float"}], outputs:[{name:"Out", type:"float"}])` produces an
  asset that opens in the editor with the inputs/outputs visible.
- After `material_function_call_node_add(...)` + connect + compile,
  the parent material's preview reflects the function's behaviour.

## Non-goals (defer)

- Material Layers / Layer Blends (separate authoring surface).
- Substrate (UE 5.5+ next-gen shading model).
- HLSL custom expression body editing (rare).
