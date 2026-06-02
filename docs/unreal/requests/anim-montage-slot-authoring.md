---
id: request-animmontage-slot-track-authoring-upper-body-reloads
title: "Request: AnimMontage slot-track authoring (upper-body reloads)"
status: draft
version: 26.602.545
tags: []
---

# Request: AnimMontage slot-track authoring (headless)

**Status:** request

## Problem

The headless weapon pipeline (ALS_UltimateWarfare) needs to assign per-weapon
reload `AnimMontage`s that play **upper-body only**. In ALS V4, upper-body
montages live in slot group **"Layering Override Group"** on slots **`Arm L` +
`Arm R`** (+ `Spine`/`Head`). Montages created by `AnimMontageFactory` land on
`DefaultGroup.DefaultSlot` (FULL body), so during reload the **legs freeze and
locomotion is lost**.

UE 5.7 Python exposes **no way to author montage slot tracks**:
- `AnimMontage.get_editor_property("slot_anim_tracks" / "SlotAnimTracks")` → `Failed to find property` (also `slot_name`, `slot_animation_track`). So slot tracks are neither readable nor writable from Python.
- `AnimMontageFactory` properties = only `source_animation`, `target_skeleton` (no slot).
- `AnimationLibrary.get_montage_slot_names(m)` is READ-only; `m.get_group_name()` read-only.
- `create_slot_animation_as_dynamic_montage_with_blend_settings(seq, slot, ...)` makes a **transient, single-slot** dynamic montage — not a saved asset, and one slot can't cover both arms.

Net: per-weapon AGLS reloads cannot be made upper-body headless today; we had to fall back to the stock ALS reloads (`ALS_LoadedReload_Montage` / `A_HG_Reload_Montage` / `A_SG_Reload_Montage`).

## Requested capability

The Companion is C++ and has full access to `FSlotAnimationTrack` / `FAnimSegment` on `UAnimMontage`. One (or both) of:

**A. `unreal_anim_montage_create_from_template`** (simplest, most robust)
- args: `template_montage_path`, `source_animation_path`, `dst_path`
- Duplicate a template montage that already has the correct slot structure (e.g. `ALS_LoadedReload_Montage` — slots `Arm L`/`Arm R`, plus its sections + blend in/out), then replace the `AnimSequence` referenced in every slot track's segment(s) with `source_animation`.
- Result: a new montage with identical slot/section/blend layout but the new clip → instantly upper-body.

**B. `unreal_anim_montage_set_slot_tracks`** (general)
- args: `montage_path`, `slots: [{ slot_name, segments: [{ anim_path, start?, end?, play_rate?, loop? }] }]`, optional `blend_in` / `blend_out`
- Authors `SlotAnimTracks` directly so any montage can be built on arbitrary slots (e.g. `Arm L` + `Arm R`) from a sequence.

Either resolves the blocker. (A) is preferred — it clones a known-good ALS template so blends/sections/curves match the existing reloads.

## Verification

After authoring:
- `AnimationLibrary.get_montage_slot_names(m)` == `["Arm L", "Arm R", ...]`
- `m.get_group_name()` == `"Layering Override Group"`
- PIE: reload plays upper-body — legs keep locomotion while running.

## How it will be used

Wire AGLS per-weapon reload sequences — `ALSP2_Reloading_Rifle_{AK47,M4A4,Famas,C8,AS50}`, `ALSP2_Reload_Pistol` (already imported, on the AGLS `ALS_Mannequin_Skeleton` which our skeleton lists as compatible) — into `DT_Weapons.PC ReloadAnimation` as upper-body montages for the 12-weapon arsenal. Currently those imported clips sit unused pending this tool.
