---
id: request-animbp-node-reassign-bp-component-class-remove-gaps
title: "Request: AnimBP node reassign + BP component class/remove gaps"
status: request
version: 26.607.1610
tags: [ unreal, blueprint, animbp, request ]
---

# Request: AnimBP node reassign + BP component class/remove gaps

Найдено при интеграции рук LPAMG в ALS_UltimateWarfare (подмена источника
оверлей-позы в `ALS_AnimBP`, добавление/удаление компонента на персонаже).
Каждый пункт сейчас обходится сырым `unreal_run_python`; хотелось бы штатные тулы.

## 1. `unreal_bp_set_component_property` не умеет ClassProperty

Установка `anim_class` (а также любого `ClassProperty`) на компоненте падает:

```
TypeError: NativizeClass: Cannot nativize 'str' as 'Class'
(allowed Class type: 'AnimInstance')
```

Тул автозагружает `/Game/...` как `UObject`, но для `ClassProperty` нужен
`UClass` (напр. `.ABP_..._C`). Use-case: задать AnimBP-класс
скелетал-меш-компонента в BP.

**Желаемо:** для `ClassProperty` грузить значение как класс
(`load_object`/`load_class` по пути с суффиксом `_C`) и присваивать.
Либо отдельный тул `unreal_bp_component_set_anim_class(bp_path,
component_var_name, anim_bp_path)`.

**Обход:** Python — `SubobjectDataSubsystem.k2_gather_subobject_data_for_blueprint`
→ найти шаблон компонента → `set_editor_property('anim_class', load_object(...))`
+ `set_editor_property('animation_mode', AnimationMode.ANIMATION_BLUEPRINT)`
→ `compile_blueprint` → `EditorAssetLibrary.save_loaded_asset(bp, False)`.

## 2. Нет тула удаления компонента из BP

Есть `unreal_bp_component_add`, но нет `remove`. Use-case: откатить ошибочно
добавленный компонент.

**Желаемо:** `unreal_bp_component_remove(bp_path, component_var_name)`.

**Обход:** Python — `SubobjectDataSubsystem.delete_subobject(root_handle,
target_handle, bp)`; при этом `SubobjectDataBlueprintFunctionLibrary.get_object`
помечен deprecated (рекомендуют `GetAssociatedObject`), но рабочий.

## 3. Переназначение ассета у аним-узла по текущей ссылке

Самый болезненный. Задача: «в AnimBP заменить все узлы, ссылающиеся на ассет A,
на ассет B» (точно по ссылке, не по заголовку). Проблемы текущего инструментария:
- `unreal_anim_blueprint_nodes` даёт имена узлов, но они **повторяются** между
  графами и без GUID/пути графа.
- `unreal_anim_blueprint_graphs` даёт **пути графов состояний**, но без узлов/GUID.
- `unreal_anim_node_set_inner_property` требует `node_guid`, который ниоткуда
  чисто не достать по графу.
- `unreal_anim_blueprint_set_node_asset_override` берёт `node_name` (коллизии
  между графами) и, по опыту проекта, ненадёжен.
- В Python у `AnimationStateGraph` свойство `Nodes` **protected** → перечислить
  узлы графа нельзя.

**Желаемо (любое):**
- (a) `unreal_anim_blueprint_nodes` с полем `graph_path` + `node_guid` +
  `current_asset` на каждый узел (тогда можно адресно через `set_inner_property`);
- (b) тул-замена по значению: `unreal_anim_reassign_asset(abp_path, from_asset,
  to_asset, graph_filter?)` — заменяет `sequence`/`blendspace` у всех узлов,
  ссылающихся на `from_asset`.

**Обход (работает):** Python — взять пути графов из `unreal_anim_blueprint_graphs`,
грузить узлы напрямую `load_object(None, graph_path +
'.AnimGraphNode_SequenceEvaluator_'+N)` перебором N, проверять
`node.get_editor_property('node').get_editor_property('sequence')`, переставлять
и `node.set_editor_property('node', inner)`, затем compile+save.
