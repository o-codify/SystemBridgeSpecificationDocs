---
id: request-bugs-anim-sm-transition-rule-state-pose-tools-1-13-2
title: "REQUEST/BUGS: anim SM transition-rule + state-pose tools (1.13.2)"
status: request
version: 26.608.2234
tags: []
---

# anim state-machine: transition-rule + state-pose tools — status & bugs

The two requested tools shipped in **companion 1.13.2**:
`unreal_anim_state_machine_set_state_pose` and
`unreal_anim_state_machine_set_transition_rule_enum_equals`. They let me add a
new "Surrender" overlay state to `ALS_AnimBP` OverlayLayer "Overlay States" SM
(state pose + conduit↔state transitions with `OverlayState ==/!= Surrender`
rules). Functionally they create the right nodes. Two bugs found in live use:

## Bug 1 — both tools throw at their final `CompileBlueprint`

`set_state_pose` and `set_transition_rule_enum_equals` return `success:false`:
```
TypeError: BlueprintEditorLibrary: Failed to convert parameter 'blueprint'
when calling function 'BlueprintEditorLibrary.CompileBlueprint'
```
(helper lines ~9061 / ~9106). The actual edit (SequencePlayer→Result;
EqualEqual_ByteByte→TransitionResult) **is applied** — only the trailing
in-tool compile throws, so a successful edit is reported as a failure.

- Workaround used: ignore the tool error, compile externally —
  `unreal.BlueprintEditorLibrary.compile_blueprint(load_asset(abp_path))`
  works with the loaded UBlueprint object, then `save_loaded_asset`. Result:
  `bps_with_errors: []`, all nodes wired.
- Fix: pass the loaded UBlueprint object (not a path string / generated class /
  None) to `CompileBlueprint`.
- Also: on failure the result blob is ~381 KB (echoes the full helper source),
  tripping MCP output limits. Return a short error instead of echoing the script.

## Bug 2 — `enum_value` resolves only by internal enumerator name

Doc says `enum_value` = "Enum entry display name, case-sensitive". In practice
the display name (`"Surrender"`) → `set_transition_rule_failed: not
resolvable`; the internal enumerator name (`"NewEnumerator13"`) resolved fine.
For UserDefinedEnums the display name is what `enum_entry_add` takes and what
authors see, so prefer accepting the display name (or fix the doc).

`enum_path` should be the full object path with `.Name` suffix
(`/…/ALS_OverlayState.ALS_OverlayState`), matching a variable's
`sub_category_object`.

## Original request (kept for context)

Needed typed authoring of (1) a state's pose and (2) a transition's boolean
rule, because `add_transition` only makes always-true edges and `add_state`
makes empty states — unusable on a shared SM (an always-true edge out of a
routing conduit hijacks every other branch). Both now exist; this doc tracks
the two bugs above to `stable`.
