---
id: skeletalmesh-socket-read-author-ue-5-7-regressions-missing-authoring
title: SkeletalMesh socket read + author (UE 5.7 regressions + missing authoring)
status: request
version: 26.604.1239
tags: []
---

# SkeletalMesh socket read + author (UE 5.7 regressions + missing authoring)

## What

Three related gaps around SkeletalMesh sockets on UE 5.7:

1. **`unreal_mesh_sockets_list` and `unreal_mesh_socket_transform` fail** on a SkeletalMesh — they try to read the protected `USkeletalMesh::Sockets` UPROPERTY and return an exception payload (the failure also dumps the whole companion preamble, ~270 KB, over the tool output limit).
2. **`unreal_skeleton_bones_list` returns `bones: []`** for a SkeletalMesh that demonstrably has bones (sockets on it resolve to real parent bones such as `WeaponRoot` / `Magazine`).
3. **No socket authoring tool** — there is no way to add or edit a named socket (name + parent bone + relative transform) on a SkeletalMesh headlessly.

## Goal

Let an author (a) list and read socket transforms on any SkeletalMesh, (b) enumerate the mesh's bones, and (c) create/edit a named socket — all headless, without the editor UI.

## Behaviour

### Read (fix existing)

- `unreal_mesh_sockets_list` should enumerate sockets via the **public** API rather than the protected `Sockets` field — e.g. `USkeletalMesh::GetNumSockets()` / `GetSocketByIndex(i)` (Python: `num_sockets()` / `get_socket_by_index(i)`), or `GetActiveSocketList()`. For each socket return `name`, `parent_bone`, `relative_location/rotation/scale`.
- `unreal_mesh_socket_transform` likewise should not touch the protected field; compute from the socket's relative transform plus the parent bone's ref-pose component transform.
- `unreal_skeleton_bones_list` should return the actual bone list for a SkeletalMesh/Skeleton (index, name, parent). It currently returns empty for at least some meshes.
- On failure, return a short structured error — never echo the companion preamble into the result.

### Author (new tool)

- New tool e.g. `unreal_mesh_socket_add` / `unreal_mesh_socket_set`:
  - Input: `mesh_path`, `socket_name`, `parent_bone`, `relative_location`, `relative_rotation`, `relative_scale` (optional), `add_to_skeleton` (bool, optional).
  - Creates the socket if absent, or updates the existing socket of that name.
  - Persists + verifies (reads the socket back and reports the stored values), then saves the asset.

## Why it matters

- Reading sockets is a prerequisite for IK/attachment authoring; today it has to be done via `unreal_run_python` against the public `find_socket` / `num_sockets` API as an escape hatch.
- Socket authoring is currently **not possible** through the scriptable API: `SkeletalMeshSocket.SocketName` and `.BoneName` are `VisibleAnywhere` + `BlueprintReadOnly`, so `set_editor_property` rejects them as read-only. `USkeletalMesh.add_socket(socket)` exists and `rename_socket(old, new)` can set the *name*, but there is no scriptable way to set the socket's **parent bone** — a freshly added socket keeps `BoneName = None`. Authors are forced to either add the socket by hand in Persona or anchor off an existing bone with a runtime offset instead of a real socket.

## Acceptance

- `unreal_mesh_sockets_list` returns the socket list for a SkeletalMesh with no exception and no preamble dump.
- `unreal_mesh_socket_transform` returns a transform for a named socket on a SkeletalMesh.
- `unreal_skeleton_bones_list` returns a non-empty bone list for a mesh that has bones.
- The new authoring tool creates a socket with the requested name **and parent bone**, the socket survives a save/reload, and `find_socket(name)` resolves it with the correct `bone_name`.
