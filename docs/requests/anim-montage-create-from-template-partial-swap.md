---
id: request-unreal-anim-montage-create-from-template
title: "Request: unreal_anim_montage_create_from_template подменяет клип не во всех слотах (регрессия)"
status: request
version: 26.606.148
tags: [ unreal, anim, montage, regression ]
---

## Проблема

`unreal_anim_montage_create_from_template(template, source_animation, dst)` должен заменить AnimSequence **во всех** слот-треках шаблона на `source_animation`. Сейчас он подменяет **частично** — в результирующем монтаже остаётся ссылка на исходный клип шаблона, и часть слотов (в т.ч. ключевые `Arm L`/`Arm R`) продолжают играть старый клип.

## Воспроизведение (проект ALS_UltimateWarfare, UE 5.7)

Шаблон `A_HG_Reload_Montage` имеет 3 слота (`Arm L`, `Arm R`, `Spine`), все ссылаются на клип `A_HG_Reload`.

1. `create_from_template(A_HG_Reload_Montage, <любой клип на ALS_Mannequin_Skeleton>, dst)`.
2. Проверка зависимостей `dst`: ожидается только новый клип. **Факт:** в deps присутствуют И новый клип, И исходный `A_HG_Reload`.
3. Подтверждено даже на «родном» клипе `ALSP2_Reload_Pistol` (тот же скелет) — частичная подмена.

При этом РАНЕЕ собранные этим тулом монтажи (`AM_AK47_Reload`, `AM_Pistol_Reload`) имеют в deps ТОЛЬКО новый клип (полная подмена) — т.е. поведение изменилось (регрессия более новой версии companion).

## Влияние

Нельзя headless собрать per-weapon upper-body reload-монтаж: в слотах рук остаётся чужой клип → визуально играет не та анимация. Блокирует перенос LPAMG-перезарядок (клип запечён на ALS-скелет, но монтаж из него собрать нечем). Python не умеет писать `SlotAnimTracks` напрямую — тул единственный путь.

## Желаемое

- `create_from_template` заменяет клип во ВСЕХ сегментах ВСЕХ слот-треков (как делал раньше).
- Опционально: поддержка source-клипа на **compatible** скелете (не только идентичном) — тогда не нужно предварительно запекать.
- Опционально: тул чтения слот-треков монтажа (`slot -> clip`) для верификации.
