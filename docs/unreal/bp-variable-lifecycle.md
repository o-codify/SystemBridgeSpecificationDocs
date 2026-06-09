---
id: request-blueprint-variable-lifecycle-repair-broken-type-variables
title: "Request: Blueprint variable lifecycle — repair broken-type variables"
status: request
version: 26.609.216
tags: [ unreal, blueprint, authoring, request ]
---

# Request: Blueprint variable lifecycle — repair broken-type variables

A Companion capability to remove or retype a Blueprint member variable whose **type reference is broken** — its `FEdGraphPinType.PinSubCategoryObject` points at a `UEnum` / `UScriptStruct` / `UClass` asset that no longer exists (null subobject). Today such a variable leaves its Blueprint permanently `BS_Error` and is only fixable by hand in the editor.

## What

Extend the variable tools (or add `bp_variable_repair`) so they act on a variable identified by `(bp_path, variable_name)` **even when its `PinSubCategoryObject` is null**:

- `remove` — delete the variable definition and its orphaned `Get`/`Set` nodes across every graph.
- `retype` — rebind the variable's `FEdGraphPinType` to a valid type via the [`bp_variable_add_typed`](blueprint-authoring.md) taxonomy (`pin_category`, `sub_category_object`), e.g. point it at a freshly created `UEnum`.

The existing `bp_variable_remove` / `bp_variable_retype` assume a well-formed pin type; the broken-subobject case is the gap.

## Goal

Let an agent headlessly lift a Blueprint out of `BS_Error` caused by a variable referencing a deleted type — no manual editor work. This is the only thing blocking a fully headless repair of such Blueprints.

## Behaviour

- Resolve and mutate the variable by name when the type subobject is null.
- `retype` propagates to dependent `K2Node_VariableGet` / `K2Node_VariableSet` so their pin types re-resolve (same reach `bp_variable_rename` already has).
- After the operation the Blueprint compiles clean (`BS_UpToDate`), not `BS_Error`.
- Must **not** force a project-wide recompile: a project can hold unrelated latent-error Blueprints that flip to `BS_Error` and block PIE when globally recompiled. Touch only the target Blueprint.
- Pure stock UE 5.7 Python cannot do this — the pin-type subobject is unbound — so it must go through Companion.

## Acceptance

- **Canonical repro:** `/ALSReplicated/AdvancedLocomotionV4/Blueprints/Components/ALS_SpawnComponent` has member variables `SpawnMode` / `SpawnMethod` typed as deleted `UEnum`s (`PinSubCategoryObject == null`), keeping the Blueprint `BS_Error`.
- `remove` on such a variable deletes it and its Get/Set nodes; the Blueprint then compiles clean.
- `retype` to a newly created `UEnum` rebinds the variable; its `SwitchEnum` and getter nodes resolve against the new enum; the Blueprint compiles clean.
- **Wrong:** the tool reports `success` while the Blueprint stays `BS_Error`, leaves dangling pins, or triggers a project-wide recompile that flips sibling Blueprints to `BS_Error`.

Reference: [Blueprint Authoring (headless)](blueprint-authoring.md).
