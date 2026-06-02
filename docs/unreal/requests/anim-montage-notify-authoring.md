---
id: request-author-animmontage-notifies-named-skeleton-at-a-chosen-time
title: "Request: author AnimMontage notifies (named/skeleton) at a chosen time"
status: request
version: 26.602.1228
tags: []
---

# Request: author AnimMontage notifies (named/skeleton) at a chosen time

Status: **request**

## Problem

ALS weapon reload refills ammo only when the reload montage contains a
**named skeleton AnimNotify `ReloadWeapon`** (the AnimBP catches it via
`AnimNotify_ReloadWeapon` and calls the weapon's ammo math). We need to place
this notify on per-weapon reload montages built headless.

UE 5.7 Python (`unreal.AnimationLibrary`) cannot create a *named* skeleton
notify at an arbitrary time:

- `add_animation_notify_event(m, track, time, notify_class)` requires a notify
  **class**; passing `None` creates a skeleton notify named literally `"None"`.
- The notify name cannot be set afterwards: `get_animation_notify_events()`
  returns **copies**, so `set_editor_property("notify_name", ...)` does not
  persist; and the `notifies` array is protected for direct read/write.
- Class-based notifies do **not** trigger the `AnimNotify_<Name>` AnimBP event
  (only skeleton/named notifies do), so substituting a class is not an option.
- `copy_anim_notifies_from_sequence(src, dst, delete_existing)` is the only way
  to get a correctly-named skeleton notify, but it copies at the source's
  **absolute** time and **drops** any notify beyond the destination length —
  so a short clip (e.g. 2.0 s pistol vs the source notify at 2.648 s) loses it.

### Current workaround (shipped, imperfect)

- Rifles (clip ≥ 2.667 s): `copy_anim_notifies_from_sequence(A_SG_Reload_Montage, montage, delete_existing=True)` → clean `[OverlayOverride@0.169, ReloadWeapon@2.648]`.
- Pistol (clip 2.0 s): rebuilt from a rifle template via
  `unreal_anim_montage_create_from_template` so the montage is long enough
  (2.667 s) for the 2.648 s notify to fire — but this leaves a ~0.67 s
  end-of-montage hold (clip shorter than montage).

## Proposed tool

`unreal_anim_montage_add_notify` (Companion):

- Args: `montage_path`, `notify_name` (string; skeleton/named notify),
  `trigger_time` (float, seconds), `track_name` (optional, default first track),
  optional `notify_state_duration` for notify-state.
- Behaviour: insert an FAnimNotifyEvent with the given `NotifyName` (no UAnimNotify
  object → skeleton notify) at `trigger_time`, registering the name on the
  skeleton if needed. Save the montage. Idempotent on (name, time, track).
- Optional companion op `unreal_anim_montage_move_notify` /
  `remove_notify_by_name` for editing.

This lets us place `ReloadWeapon` at, e.g., `0.7 * clip_length` on each montage
and trim the pistol montage back to its true 2.0 s (no end hold).

## Done another way meanwhile

Ammo refill now works for all 12 weapons via the copy/reclone workaround above
(commit 39a1feb in the game repo). This request only removes the remaining
timing imperfection.
