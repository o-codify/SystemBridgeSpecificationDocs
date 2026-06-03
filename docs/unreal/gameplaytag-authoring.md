---
id: request-author-refresh-gameplaytags-headlessly
title: "Request: author & refresh GameplayTags headlessly"
status: request
version: 26.603.1452
tags: [ unreal, gameplaytags, authoring, request ]
---

# Request: author & refresh GameplayTags headlessly

## Status

`request` — confirmed gap (UE 5.7.4, companion 1.7.0). Filed by AI agent
2026-06-03 while building an HLS-style weapon interaction layer that uses
GameplayTags as state identifiers (EWeaponPoseState / EWeaponInteractionPhase /
contact states / EHand / GripPoseId stand-ins).

## The gap

There is no SB tool and no exposed Python API to register or refresh project
GameplayTags. Probed live:

- `unreal.GameplayTagsManager` → **absent**
- `unreal.GameplayTagsSettings` → **absent**
- `unreal.GameplayTagsEditorModule` → **absent**
- `unreal.BlueprintGameplayTagLibrary` → **absent**
- `unreal.GameplayTagLibrary` → present, but it's the runtime BP library:
  `make_literal_gameplay_tag` is a passthrough (takes an FGameplayTag, not a
  name), `is_gameplay_tag_valid` / `get_tag_name` work for inspection. No
  `request_gameplay_tag`, no add, no refresh.

So new tags cannot be created from automation, and the in-memory tag tree
cannot be refreshed.

## Workaround used

1. Wrote `+GameplayTagList=(Tag="...",DevComment="...")` lines into
   `Config/DefaultGameplayTags.ini` via the files plugin.
2. **Restarted the editor** (`unreal_editor_restart`) so the tag tree reloads.
3. Constructed valid `FGameplayTag` values in Python via
   `t = unreal.GameplayTag(); t.import_text("Tag.Name")` (validates against the
   registry; unregistered names resolve to an empty/None tag).

The editor restart is the painful part — any tag addition forces a full
cold-restart cycle.

## Requested capability

```
unreal_gameplaytag_add(tag, dev_comment?, source_ini?)      -> {added}
unreal_gameplaytag_add_many(tags[])                          -> {added[], skipped[]}
unreal_gameplaytag_remove(tag)                               -> {removed}
unreal_gameplaytag_list(prefix?)                             -> [{tag, source}]
unreal_gameplaytag_refresh()        # rebuild tag tree in-memory, no restart
unreal_gameplaytag_is_valid(tag)                             -> {valid}
```

- `add` should write to `Config/DefaultGameplayTags.ini` (or a named source)
  AND register in-memory so the tag is immediately usable without a restart —
  the editor's own Tag Manager UI does this via
  `UGameplayTagsManager::AddNewGameplayTagToINI` +
  `EditorRefreshGameplayTagTree`.
- `refresh` exposes `UGameplayTagsManager::EditorRefreshGameplayTagTree()`.

## Acceptance

- Add 50+ tags headlessly; they are immediately valid (no editor restart) and
  settable as `bp_node_pin_set_default` values and DataAsset/BP property
  defaults.
- `list` round-trips what was added; `remove` cleans up.

## Related

See [UserDefinedStruct/Enum authoring request](userdefined-struct-enum-authoring.md)
— same family of "data-definition authoring is C++/editor-only" gaps. Together
they block building project-authored type/data models fully headlessly.
