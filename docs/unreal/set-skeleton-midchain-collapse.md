---
id: set-skeleton-bone-name-map-mid-chain-bone-collapse-is-broken-5-3-spine-reduction
title: "set_skeleton bone_name_map: mid-chain bone collapse is broken (5→3 spine reduction)"
status: request
version: 26.610.1737
tags: [ unreal, skeletal-mesh, set_skeleton, bone_name_map, request ]
---

# set_skeleton bone_name_map: mid-chain collapse is broken

Follow-up to the shipped `unreal_skeletal_mesh_set_skeleton` `bone_name_map` feature.
The rename half works; the **collapse half fails for mid-chain bones that have a
surviving child**, leaving garbage `SB_DROP__*` bones with non-zero skin weights
and scrambling the spine weighting. This blocks the canonical N→M chain reduction
the feature was added for.

## Exact repro (real case)

Sidekick part mesh with a 5-bone spine + 2-bone neck, re-targeted onto the ALS
3-spine / 1-neck skeleton:

```
set_skeleton(
  mesh_path = ".../SK_SCFI_CIVL_09_10TORS_HU01",
  target_skeleton_path = ".../ALS_Mannequin_Skeleton",   # clean: spine_01/02/03, neck_01, head
  bone_name_map = {"spine_05":"spine_03", "spine_03":"spine_02"},
  missing_bone_policy = "remap_to_parent",
  save = true)
```

Source spine chain: `spine_01→spine_02→spine_03→spine_04→spine_05→neck_01→neck_02→head`.
Intent (user's "1,3,5" endpoints+middle): keep SK `spine_01`→`spine_01`,
`spine_03`→`spine_02`, `spine_05`→`spine_03`; **drop** SK `spine_02`, `spine_04`,
`neck_02`, merging their weights into the surviving chain and reparenting the kept
children to the grandparent.

## Reported vs actual

Tool returned: `matched:68, renamed:2, preempted:["spine_02"], remapped:31,
target_modified:false`, and listed `spine_04`, `neck_02` under
`unmatched:[{action:"removed"}]`.

Actual persisted mesh (re-read from disk after save) — bone chain became:

```
spine_01 → SB_DROP__spine_02 → spine_02 → spine_04 → spine_03 → neck_01 → neck_02 → head
```

- `spine_05` correctly renamed away; `spine_03`/`spine_02` renames applied. ✔
- But `SB_DROP__spine_02` (the preempted original `spine_02`), `spine_04`, and
  `neck_02` are **still present** — they were NOT removed despite being reported
  `removed`. All three are exactly the bones that have a **surviving child**
  (`SB_DROP__spine_02`→`spine_04`; `spine_04`→`spine_03`; `neck_02`→`head`).
  Only leaf-ish bones (metacarpals, eyes, jaw, *Attach) actually got removed
  (110→99 bones, ~11 removed, not the ~32 reported).

Skin-weight audit (run_cpp, summed `InfluenceWeights` per global bone over LOD0):

| bone | weight sum | note |
|---|---|---|
| `spine_02` (kept, was SK spine_03) | 87,718,190 | ok |
| `SB_DROP__spine_02` (should be gone) | **29,930,796** | ~16% of mesh still bound to a drop-marked bone |
| `spine_04` (should be gone) | 0 | weightless leftover |
| `neck_02` (should be gone) | 0 | weightless leftover |
| `spine_03` (kept top, was SK spine_05) | **0** | top-of-spine ended up with NO weight |
| total | 181,925,160 | |

So the result is not merely cosmetic: a drop-marked bone retains 16% of the skin,
and the kept top-of-spine bone ends up weightless — the deformation is wrong, not
just the hierarchy.

## Likely root cause

`remap_to_parent` is implemented via `IMeshUtilities::RemoveBonesFromMesh`, whose
semantics remove a bone **together with its entire subtree**. A mid-chain bone
whose child is *kept* therefore cannot be removed that way (removing it would take
the kept child with it), so it is silently skipped — but its name has already been
mangled to `SB_DROP__` and the weight transfer is left half-done.

## Requested behaviour

`bone_name_map` collapse (and `remap_to_parent` generally) should support
**merge-bone-into-parent**, not just remove-subtree:

1. For each bone to drop, **reparent its surviving children to its parent** first,
   then delete only that node.
2. **Transfer its skin weights to the parent** (or, for a remap target, to the
   mapped survivor) and re-normalize — guarantee the dropped bone ends at 0 weight
   and no vertex is left bound to it.
3. Never leave `SB_DROP__*` placeholder bones in the saved asset; if a drop cannot
   be completed, **fail the whole op** (transactional) rather than persist a
   half-collapsed, mis-weighted mesh.
4. Verify post-op by **world position of the deformed result**, not by bone name —
   for this case the success check is: after leader-pose drives the mesh from the
   ALS mannequin, no spine/neck vertex band sits above `neck_01`.

## Workaround attempted

None viable with the current tool: `merge_into_target` keeps `spine_04/05` (they
ride free above the driven `spine_03` under leader-pose and overshoot `neck_01` by
~7 cm — the original bug). `remap_to_parent` produces the broken state above.
Source mesh + skeleton were restored from backup; no batch applied.
