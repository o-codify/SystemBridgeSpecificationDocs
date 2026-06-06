---
id: invoke-a-blueprint-event-or-function-on-a-runtime-object-pie
title: Invoke a Blueprint event or function on a runtime object (PIE)
status: request
version: 26.606.148
tags: []
---

# Invoke a Blueprint event or function on a runtime object (PIE)

## What

A tool to call a Blueprint-exposed custom event or function, by name, on a live object in a running PIE session — an actor or one of its components — passing typed arguments.

## Goal

Let an author drive and verify gameplay logic headlessly: trigger a specific event on the possessed pawn, a component, or another actor, then observe the result — without simulating user input or reverse-engineering input bindings.

## Behaviour

- Target selection: by player index (e.g. controlled pawn of player 0), by actor name/label in the PIE world, or by class plus optional component variable name to reach a sub-component.
- Input: target, `function_name` (event or callable function), and an ordered/named argument list with typed values (scalars, structs, object references by runtime handle or path, enum names).
- Coerce arguments to the function signature; return any output/return values.
- Run against the PIE world, not the editor world.
- Error clearly when the target is not found, the function does not exist on it, or arguments do not match the signature.

## Why it matters

There is currently no SB way to call a Blueprint custom event/function on a runtime object; verifying a feature requires the raw-Python escape hatch (`call_method` on an object fetched via `GameplayStatics`). A first-class tool makes automated end-to-end checks (trigger → observe state) reproducible.

## Acceptance

- Calling a known parameterless custom event on the controlled pawn's component executes it and reflects the resulting state change (e.g. a flag flips, a value updates).
- Passing typed arguments (float, vector, object reference) invokes the matching overload and returns its outputs.
- Unknown target / unknown function / signature mismatch each return a distinct, clear error and invoke nothing.
