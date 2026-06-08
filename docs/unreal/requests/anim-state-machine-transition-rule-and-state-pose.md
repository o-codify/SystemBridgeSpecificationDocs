---
id: request-bugs-anim-sm-transition-rule-state-pose-tools-1-13-2
title: "REQUEST/BUGS: anim SM transition-rule + state-pose tools (1.13.2)"
status: request
version: 26.608.2332
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

## КРИТично: правило не гейтит переход (состояние недостижимо)

Глубокая проверка показала, что `set_transition_rule_enum_equals` не просто
падает на финальном `CompileBlueprint` — **созданное правило фактически не
работает**: переход никогда не истинный, и целевое состояние недостижимо.

Проверка (ALS `ALS_AnimBP`, overlay-SM, новое состояние Surrender, value=13):
1. Задал правило conduit→Surrender `OverlayState == <Surrender>` тулом,
   скомпилировал внешне (`bps_with_errors: []`).
2. В PIE выставил `OverlayState=13` → состояние Surrender НЕ входит (замер
   костей: поза не меняется; стейт-машина залипает в предыдущем состоянии).
3. **Решающий тест:** временно подменил позу состояния Surrender на заведомо
   «руки вверх» (`ALS_StanceVariation_HandsTied`). При `OverlayState=13` руки
   всё равно НЕ поднялись (hand Z ≈ как в Default), т.е. проблема НЕ в клипе и
   НЕ в значении enum, а в том, что **правило перехода не пропускает** (либо
   `bCanEnterTransition` не подключён, либо «replace» поверх старого правила не
   срабатывает — тул вызывался дважды с разными значениями).

Для сравнения: штатные переходы (например conduit→Hands Tied) работают — то
есть механизм ALS исправен, неисправен именно вывод тула.

**Нужно:** (1) чинить `CompileBlueprint`-шаг; (2) гарантировать, что
`EqualEqual`→`TransitionResult.bCanEnterTransition` реально соединён и что
повторный вызов корректно ЗАМЕНЯЕТ правило; (3) добавить **ридер** правила
перехода (`get_transition_rule`) — сейчас правило невозможно прочитать/проверить
ни одним тулом, отладка вслепую. Также принимать значение enum по числу/числовому
value, а не только по внутреннему имени `NewEnumeratorN` (имена в
UserDefinedEnum непоследовательны и сбивают).
