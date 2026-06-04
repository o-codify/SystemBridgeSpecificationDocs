---
id: blueprint-variable-lifecycle-remove-rename-retype-are-unreliable
title: "Blueprint variable lifecycle: remove / rename / retype are unreliable"
status: request
version: 26.604.2113
tags: []
---

# Blueprint variable lifecycle: remove / rename / retype are unreliable

**Status: request.** Observed on `ALS_UltimateWarfare` (UE 5.7.4, Companion 1.11.1), 2026-06.

[blueprint-authoring.md](blueprint-authoring.md#rename--remove) presents
`bp_variable_rename` / `bp_variable_remove` as simple, surgical refactors.
In practice all three variable-lifecycle operations have sharp edges that
cost a lot of debugging time, and one of them silently caused collateral
damage. Filing the gap.

## 1. `bp_variable_remove` — blind, collateral, can't remove broken-typed

Reported method (from the tool result): `replace_refs_to_sentinel +
remove_unused_variables`. Consequences observed:

- **Collateral removal.** `remove_unused_variables` is project-blind: it
  removes *every* variable not referenced in the BP's own graphs — not
  just the named target. On `ALS_SpawnComponent` a stray `Exception`
  string var (unused in graphs, possibly read externally) would have been
  nuked along with the target. An earlier session: it nearly removed
  legit `HasAmmo?`/`ShotCount`/`WeaponData` (used by other BPs, not in
  graphs). **Mitigation we had to invent:** add a dummy `VariableGet` for
  every legit-but-graph-unused member var before calling remove, then
  delete the dummies after — impractical on a large BP.

- **Blocked by any live reference.** If the target still has a Get/Set
  node anywhere, removal silently fails: `{removed:true,
  still_present:true, success:false}` (contradictory flags). Caller must
  first repoint/delete all references — but there's no targeted way to do
  that without the rename/replace tools (also broken, see below).

- **Cannot remove a broken-typed variable at all.** A variable whose
  type is an unresolved/null `UEnum` (the enum asset was deleted) is
  skipped by `remove_unused_variables` entirely — returns
  `removed:true, still_present:true, success:false` repeatedly, even with
  zero references, even with the BP editor open, even after clearing
  `instance_editable`. `BlueprintEditorLibrary.remove_unused_variables`
  returns `0`. There is **no headless path** to delete such a variable;
  it can only be deleted by hand in the editor (right-click → Delete).
  This blocks any headless repair of a BP that references a deleted
  enum/struct/class.

## 2. `bp_variable_rename` — NOT atomic

Documented as "refactors Get/Set across all graphs." Actual method (from
the tool result): `add+replace_refs (UE 5.7 Python has no atomic rename —
old definition kept)`. So it:

- **Creates a NEW variable** under `new_name` and repoints node refs to it.
- **Leaves the OLD variable definition in place** (now orphaned) — which
  then needs `bp_variable_remove` (see §1, may be impossible).
- **Mis-detects the type** when the source var is broken: renaming a
  broken null-enum var produced `detected_type: Int` — a junk Int copy.
- Does not preserve metadata (category/replication), per the tool note.

Net: "rename" leaves the BP dirtier than before (old var + possibly a
wrong-typed new var), the opposite of a refactor.

## 3. In-place RETYPE — missing entirely

There is no tool (and no headless Python path) to change an existing
variable's type. `UBlueprint.NewVariables` is protected; the type lives
in an `FBPVariableDescription.VarType` (`FEdGraphPinType`) that can't be
read or mutated from Python. So "the enum this var points at was deleted,
re-point it at the recreated enum" is impossible — the only route is
add-new + replace_variable_references + remove-old, and remove-old fails
(§1). We hit a hard dead end repairing `ALS_SpawnComponent` this way.

## Desired API

- `bp_variable_remove` should remove **only** the named variable
  (snapshot NewVariables, restore everything except the target — the
  "gate" pattern), and should handle broken-typed vars (operate on the
  `FBPVariableDescription` directly via
  `FBlueprintEditorUtils::RemoveMemberVariable`, which works in the editor
  UI regardless of type). Reference handling: optionally orphan refs
  (current) or refuse with a list of referencing nodes.
- `bp_variable_rename` should be a true rename of the existing
  `FBPVariableDescription` (`FBlueprintEditorUtils::RenameMemberVariable`),
  preserving type + metadata, leaving no old/junk var.
- New `bp_variable_retype(bp_path, name, pin_category, sub_category_object,
  container)` — rebuild the var's `FEdGraphPinType` in place
  (`FBlueprintEditorUtils::ChangeMemberVariableType`).

All three are thin wraps of `FBlueprintEditorUtils` members that the
editor's "My Blueprint" panel already calls — they just aren't exposed to
Python, which is exactly the gap Companion exists to fill.

## Repro / workaround used

- Repro: create a `UserDefinedEnum`, make a BP var of that type, delete
  the enum asset, try to `bp_variable_remove` the now-broken var → it
  won't delete.
- Workaround for ADDING typed vars + wiring: use the
  `SystemBridgeBindings` directly inside `unreal_run_python` on the loaded
  BP object (`add_typed_member_variable`, `set_k2_node_variable_reference`,
  `replace_variable_references`) — but none of these can DELETE a
  broken-typed var either.
