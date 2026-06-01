---
id: request-replication-authoring-rpc-events-replicated-repnotify-variables
title: "Request: Replication authoring (RPC events + replicated/RepNotify variables)"
status: request
version: 26.601.1606
tags: [ unreal, blueprint, authoring, replication, request ]
---

# Request: Replication authoring (RPC events + replicated/RepNotify variables)

**Status: feature request / not yet implemented.** Drafted by the agent after hitting a hard wall building multiplayer Blueprint logic via SystemBridge. Companion at time of writing: **v1.3.4**.

This is the replication counterpart to [Blueprint Authoring](../blueprint-authoring.md). Object-pin defaults and typed variables (v1.3.4) closed the "set an asset reference" gap; this closes the "make it networked" gap.

## Motivating case

Project `ALS_UltimateWarfare` is a **networked** ALS game (`ALSReplicated` plugin). A height-based hard/medium landing montage was wired into `Event On Landed` via `Play Anim Montage`. It works on the owning client, but **other players see only the stock landing** — `Play Anim Montage` is local.

Stock ALS replicates `roll`/`breakfall` correctly via two standard patterns, both of which SystemBridge currently cannot author headless:

1.  **RPC events.** `Roll Event` / `Breakfall Event` are **Server** custom events; inside `Rep Roll` / `Rep Breakfall` collapsed graphs they call `Client Roll` / `Client Breakfall` **NetMulticast** custom events, which each client runs locally → `Montage Play`.
2.  **Replicated + RepNotify variable.** `MovementAction` is replicated with `OnRep_MovementAction`; a server-side change fires OnRep on every client.

Without these, **no** networked feature can be built (action sync, state sync, replicated damage, …). Hence high, general priority — not landing-specific.

## Why headless is hard in UE

Same root cause family as the object-pin gap (see [Blueprint Authoring → Why headless is hard](../blueprint-authoring.md#why-headless-is-hard-in-ue)):

-   `UK2Node_CustomEvent` net mode + reliability live in `FunctionFlags` (`FUNC_Net | FUNC_NetMulticast | FUNC_NetServer | FUNC_NetClient | FUNC_NetReliable`). Stock Python `get_editor_property("function_flags" / "is_net_reliable" / "custom_function_name")` → *"Failed to find property"*. `export_text()` does not surface the flags either. So they can be neither read nor written.
-   Custom-event **typed parameters** live in `UK2Node_EditablePinBase::UserDefinedPins` — not addable from Python.
-   BP variable replication lives in `FBPVariableDescription.PropertyFlags` (`CPF_Net`, `CPF_RepNotify`) + `RepNotifyFunc`. `BlueprintEditorLibrary` exposes only `replace_variable_references`; no replication setter.

## Reproduction (what fails today, v1.3.4)

```
# 1) custom event is created but stays LOCAL — no way to make it multicast.
bp_custom_event_add(bp_path, graph_name="EventGraph",
                    custom_event_name="Multicast_PlayLandMontage")
  → node created; net mode unsettable, no typed param addable.

# 2) reading/writing its flags via python fails:
node.get_editor_property("function_flags")   → Failed to find property
node.get_editor_property("is_net_reliable")  → Failed to find property

# 3) variable replication: no tool, no python path.
#    bp_variable_add[_typed] create the var but cannot mark it Replicated/RepNotify.
```

## Requested surface

### A. RPC custom events + typed params

Extend `bp_custom_event_add` (or add `bp_custom_event_configure`):

```
bp_custom_event_configure(
  bp_path, graph_name, node_guid,
  net_mode = "multicast" | "server" | "client" | "notreplicated",  # default notreplicated
  reliable = true|false,        # default true when replicated
  call_in_editor = false,
)
```

Typed parameter add (custom events and, generally, function entries):

```
bp_event_add_param(
  bp_path, graph_name, node_guid,
  param_name = "Montage",
  pin_category = "object",                       # reuse bp_variable_add_typed taxonomy
  sub_category_object = "/Script/Engine.AnimMontage",
  container = "single",
) → new output pin on the event
```

Engine hooks:

-   Flags: set bits on `UK2Node_CustomEvent::FunctionFlags` (`FUNC_Net` + one of Multicast/Server/Client, `+ FUNC_NetReliable`), then `ReconstructNode()` + mark BP modified. Mirrors the editor's right-click → "Replicates" dropdown.
-   Params: `UK2Node_EditablePinBase::CreateUserDefinedPin(name, PinType, EGPD_Output)` then `ReconstructNode()`. Build `FEdGraphPinType` with the same builder already used by `bp_variable_add_typed`.
-   Validation note: Server/Client RPCs require a replicated owning actor; Multicast must be invoked from the server. SB just sets flags — graph correctness is the author's responsibility.

### B. Replicated / RepNotify variables

```
bp_variable_set_replication(
  bp_path, variable_name,
  mode = "none" | "replicated" | "repnotify",
  rep_notify_function = "OnRep_LandState",   # repnotify only; auto-create empty OnRep if missing
)
```

Engine hooks:

-   Find `FBPVariableDescription` in `Blueprint->NewVariables` by name.
-   `replicated`: `PropertyFlags |= CPF_Net`.
-   `repnotify`: `PropertyFlags |= CPF_Net | CPF_RepNotify`, set `RepNotifyFunc = FName(rep_notify_function)`; create the `OnRep_*` BP function if absent (empty body is fine — UE recognizes it as OnRep).
-   `FBlueprintEditorUtils::MarkBlueprintAsStructurallyModified` + compile. (Check 5.7 for an existing `SetVariableReplicationFlag` / repnotify helper to reuse.)

## Acceptance (multiplayer PIE, 2 players, Listen Server)

1.  **A:** Create NetMulticast Reliable `Multicast_PlayLandMontage(Montage)`, body = `Montage Play`; invoke on the server on hard landing → the **second** player (simulated proxy) shows the same landing animation, not stock.
2.  **B:** Mark `LandState` (enum/byte) RepNotify with `OnRep_LandState` that plays the montage; server-side change fires OnRep on clients → visible on all.

Either path, taken to "the medium/hard landing animation shows synchronously on other players with no manual editor work", closes this request.

## Once implemented

Agent will wrap the landing montage in a NetMulticast Reliable event (called on the server) — or a RepNotify state var — and verify with a 2-player PIE. Until then the landing feature is **client-local only** (tracked in the game's own backlog/tech-debt).
