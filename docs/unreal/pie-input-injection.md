---
id: inject-input-into-a-running-pie-session
title: Inject input into a running PIE session
status: request
version: 26.606.148
tags: []
---

# Inject input into a running PIE session

## What

A tool to deliver input to a running PIE session as if the player produced it: press/release a key, fire a named input action or axis, for both the legacy input system (action/axis mappings) and Enhanced Input (Input Actions).

## Goal

Let an author exercise input-driven gameplay headlessly — equip, fire, reload, interact — to test end-to-end behaviour without a human at the keyboard and without depending on which physical key a binding uses.

## Behaviour

- Address input semantically where possible: fire a mapping/action by **name** (e.g. action `Reload`, action `Slot1`, an Enhanced Input `IA_*`), so tests do not hardcode physical keys.
- Also support raw key events by key name with press, release, and tap, plus optional modifiers and a value for axis/analog inputs.
- Route to a chosen local player (default player 0) and the PIE world.
- Detect which input system is active and dispatch through it; if both exist, let the caller specify.
- Error clearly when no PIE session is running, the named action/mapping does not exist, or no player controller is available.

## Why it matters

Input-gated features can only be verified by producing input. With no SB input-injection, a reload triggered by the `Reload` action or a weapon equipped by `Slot1` cannot be tested headlessly; the only fallbacks are screen-level automation or editing instance state that is often locked from edits. Semantic action injection makes gameplay tests reliable and key-binding-independent.

## Acceptance

- Firing a named action that is bound in the project triggers the same Blueprint logic a physical key press would, observable via resulting state.
- Press/release of a raw key produces correct pressed/released transitions (held-key behaviour works, not just taps).
- Works for both legacy mappings and Enhanced Input actions in projects that use either.
- No running PIE, unknown action, or missing player controller each return a distinct, clear error.
