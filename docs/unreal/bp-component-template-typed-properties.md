---
id: set-typed-properties-on-a-blueprint-component-template-scs
title: Set typed properties on a Blueprint component template (SCS)
status: request
version: 26.606.148
tags: []
---

# Set typed properties on a Blueprint component template (SCS)

## What

The ability to set a property on a component **template** in a Blueprint's construction script (SCS) using typed values — numbers, booleans, vectors, rotators, transforms, and struct/enum literals — not only strings. This is the gap behind the existing `unreal_bp_set_component_property` tool.

## Goal

Let an author change the default value a placed component will have on spawned actors (e.g. a float duration, a transform offset on a component added to a character), and have that value actually take effect at runtime.

## Behaviour

- Input: `bp_path`, `component_var_name`, `property`, `value`.
- `value` must accept typed inputs:
  - scalars: integer, float/double, boolean;
  - common structs by literal: `Vector`, `Rotator`, `Transform`, `LinearColor`;
  - enum entries by name; object/asset references by path.
- Coerce the value to the property's reflected type; if it cannot, return a precise type-mismatch error naming the property and expected type.
- Write to the SCS node's component template so the change persists in the asset and applies to every instance spawned from the Blueprint.

## Why it matters

Two related problems block setting component defaults headlessly:

- `unreal_bp_set_component_property` currently treats `value` as a string and fails to nativize numeric and struct values — e.g. `"2.5"` cannot become a `double`, and a `Transform` literal is rejected ("Struct has 3 initialization parameters, but the given sequence had 80 elements").
- `unreal_bp_variable_set_default` sets the **class** default, which does **not** propagate to a component already placed on an actor via SCS — the placed template keeps its older captured value. There is no SB tool to edit the SCS template, forcing a fallback to raw Python (`set_editor_property` on the `..._C:Component_GEN_VARIABLE` archetype, then `save_loaded_asset`).

## Acceptance

- Setting a `double` property to `2.5` and a `Transform` property to a translation offset on a component template both succeed via typed values.
- After save and a fresh spawn (e.g. PIE), the spawned component instance reports the new values — confirming the SCS template, not just the class default, was updated.
- A value that cannot be coerced to the property type yields a clear error identifying the property and expected type, and writes nothing.
