---
id: request-sequencer-level-sequence-authoring-tracks-sections-keys
title: "Request: Sequencer (Level Sequence) authoring — tracks, sections, keys"
status: request
version: 26.607.2208
tags: [ unreal, sequencer, cinematics, authoring, request ]
---

# Request: Sequencer authoring

**Origin.** Comparative coverage analysis. UAssetAPI ships full
`Movies/` struct parsers (`MovieScene`, `MovieSceneTrack`, key channel
data). SystemBridge has `unreal_sequence_info` which returns name +
length and nothing else. This is the biggest read-side coverage gap
in our existing surface.

## What

A Sequencer authoring layer that mirrors how `bp_node_*` / `anim_node_*`
cover Blueprint and AnimGraph. Tracks-as-data, sections-as-data,
keys-as-data — readable, mutable, verifiable.

## Tools

### Read

| Tool | Purpose |
|---|---|
| `sequencer_bindings_list(seq_path)` | Object bindings (actors / components / spawnables) with their guids. |
| `sequencer_tracks_list(seq_path, binding_guid?)` | Tracks on a binding (or master tracks if `binding_guid` omitted). Class, display name, section count. |
| `sequencer_track_info(seq_path, binding_guid, track_guid)` | Sections with start/end times + keys per section. |
| `sequencer_keys_list(seq_path, binding_guid, track_guid, section_guid)` | All keys: time, value, interpolation. |

### Write

| Tool | Purpose |
|---|---|
| `sequencer_track_add(seq_path, binding_guid, track_class)` | Add a track (Transform / Float / Bool / Event / Audio / Camera / Anim). |
| `sequencer_track_remove(seq_path, track_guid)` | Idempotent. |
| `sequencer_section_add(seq_path, track_guid, start, end)` | Returns section_guid. |
| `sequencer_key_add(seq_path, section_guid, channel, time, value, interp="cubic")` | Channel: `Translation.X` / `Rotation.Yaw` / `Float` / `Bool`. |
| `sequencer_key_remove(seq_path, section_guid, channel, time)` | Idempotent. |
| `sequencer_binding_add(seq_path, possessable_path)` | Bind an existing actor; returns binding_guid. |

## Why

Sequencer is THE cinematic / cutscene / scripted-event surface in UE.
We can't currently:

- Author a cutscene headlessly (cinematic prototyping).
- Build a replay/highlight system that authors a sequence per session.
- Verify a sequence ships with the right key counts / track set.
- Migrate a sequence to a new actor binding when an actor's
  class changes.

The runtime API (`UMovieScene`, `UMovieSceneTrack`, `UMovieSceneSection`,
section-specific key channels) is editor-side C++. Python's
`sequencer_scripting` exposes part of it — enough for the read side, not
enough for safe writes (key channels in particular).

## Implementation route

- Companion C++ — channel writes need typed access to
  `TMovieSceneChannelHandle<...>`, which Python can't reach.
- Module deps: `MovieScene`, `MovieSceneTracks`, `LevelSequence`,
  `SequencerScripting` (editor side, builds on the runtime).
- Pure-Python helpers for read paths where `sequencer_scripting`
  already exposes the data.

## Acceptance

- A new `LevelSequence` asset can be built end-to-end headless:
  add binding → add Transform track → add section → write 5 location
  keys with cubic interpolation. After save, the editor opens it and
  plays correctly.
- Same for a Float track on a Material parameter, an Anim track on a
  SkeletalMeshActor, an Event track firing a custom event.
- `sequencer_tracks_list` returns key counts that match what the
  Sequencer editor's curve view shows.

## Non-goals (defer)

- Spawnables creation (tricky lifecycle — defer).
- Cooked sequence reading.
- Sequence baking (rendering / encoding) — Movie Render Queue is its
  own surface.
