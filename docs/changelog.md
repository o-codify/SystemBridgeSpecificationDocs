---
id: changelog
title: Changelog
status: stable
version: 26.610.1553
tags: [ changelog ]
---

# Changelog

Milestones for SystemBridge as a whole — daemon (`sb.exe`), plugins,
and the Unreal Companion sub-plugin. Individual companion versions are
also captured in [companion plugin reference](unreal/companion.md).

## Recent: run_cpp — C++ snippets in the live editor

| Date | Tag | What |
|---|---|---|
| 2026-Q2 | sb-unreal | `run_cpp` + `run_cpp_status` — the native analogue of `run_python`. Splices a C++ snippet into the fixed `RunScratch()` body of a tool-generated **project-scope** SystemBridgeScratch plugin, recompiles via Live Coding (~4 s), generation-stamp-verifies the patch actually loaded, invokes via Remote Execution. Compile errors come back structured `{file, line, code, message}` by re-running cl.exe on the scratch TU with UBT's own .rsp (UE 5.7.4 LCC keeps diagnostics window-only). Companion untouched — Live Coding can't patch precompiled Rocket plugins, which is exactly why the scratch is project-scope. Risk: destructive. See [run_cpp reference](unreal/run-cpp.md). |

## Recent: bulk offline asset scanner (no companion / no editor)

| Date | Tag | What |
|---|---|---|
| 2026-Q2 | sb-unreal | Three new offline tools that walk `.uasset` headers directly: `assets_find_references` (who references this?), `assets_find_by_class` (every BP / DataTable / AnimBP / ...), `assets_scan_offline` (project-wide audit). 1000-asset project completes in 1-3 seconds; equivalent editor-RPC paths take 30-150 seconds. Go-port path (vs C# wrapper) — in-process, no shellout. |

## Recent: Companion v1.11.x — AnimGraph authoring + transform queries

| Date | Tag | What |
|---|---|---|
| 2026-Q2 | **sb cross-cutting** | Strategic 6-ship from `github-inside-claude-code` comparative analysis. (1) `internal/errcodes` — 14 typed codes (NotFound / AuthMissing / RateLimited / Conflict / Network / Timeout / …) + `Error` struct + `ToMCPResult()` shim + default-recoverable hints; adopted by sb-github & sb-research. (2) `manifest.ToolDecl.RiskLevel` — low / medium / high on writes; validator enforces; surfaced via `mcp.tools_list`; tagged across files / db / docker / config / github. (3) sb-github plugin — 40 tools (PR / issue / repo / CI / release / search / session) via gh CLI; full errcodes integration via `mapGhError` heuristic. (4) `watch.*` core tools + `internal/watch` registry — long-running observer pattern for CI follows / build watches / render queue progress. (5) sb-views recipes +6 (now 11 total): `project_health`, `pr_overview`, `ci_state`, `db_inventory`, `docker_dev_stack`, `dev_session_overview`. (6) sb-research plugin — 7 federation adapters: stackoverflow / hackernews / wikipedia / npm / pypi / crates / godoc. |
| 2026-Q2 | **sb-code v0.1** | LSP bridge — 8 tools: definition / references / hover / diagnostics / symbols_in_file / workspace_symbols / server_status / server_restart. v0.1 manages a long-running gopls child process per project root; reused across calls; full LSP framing over stdio. End-to-end verified: `workspace_symbols("buildManifest")` returns 20 hits across the project. Other languages (pyright / typescript-language-server / rust-analyzer / clangd) tracked as v0.2 follow-up. |
| 2026-Q2 | **sb-build** | Auto-detect toolchain runner. 8 tools: build / test / lint / format / deps_outdated / deps_install / deps_upgrade / toolchain_detect. Adapters: Go (`go test -json` NDJSON parser), Rust (cargo `--message-format json`), Node (npm / pnpm / yarn detected from lockfile; jest/vitest `--json`), Python (pip / uv / poetry; pytest text scrape), Make. Normalised test output: `{passed, failed, skipped, failures: [{name, file, line, message}]}`. |
| 2026-Q2 | **sb-semantic** | BM25 code search with identifier-aware tokenisation. `parseBlueprintManifest` → `{parse, blueprint, manifest}` so a query "parse blueprint manifest" matches without lemmatisation. Per-project SQLite index at `<project>/.sb_index.sqlite`. Incremental (mtime-driven). 5 tools: index / search / similar / stats / clear. End-to-end on this codebase: 271 files / 2574 symbols indexed in 3.7s; query "manifest tool registration" ranks `tool_registry_test.go` first. |
| 2026-Q2 | **sb-scrape** | Structured web extraction. 6 tools: html / text (markdown / plain / readability) / fields (CSS-selector schema) / links / crawl (depth + same-origin) / screenshot. chromedp + html-to-markdown + go-readability + goquery. Falls back to static fetch when Chrome isn't available. 250 ms rate limit on `crawl`; 100-page cap. |
| 2026-Q2 | **sb-docker** | 23 tools. Container lifecycle (ps / inspect / logs / exec / start / stop / restart / rm / stats), images / volumes / networks, compose (up / down / ps / logs / build). Docker SDK over unix socket / Windows named pipe; compose v2 shelled out. Degrades cleanly when daemon isn't running. |
| 2026-Q2 | **sb-db** | SQL + Redis. 13 tools. PostgreSQL (pgx) / MySQL (go-sql-driver) / SQLite (modernc.org/sqlite — pure-Go, no cgo) / Redis (go-redis). Connection aliases in `~/.systembridge/db_conns.toml` — DSN never appears in audit log. 1000-row cap per query, 5 s default timeout, `read_only=true` defence in depth. |
| 2026-Q2 | **sb-config** | Format-aware JSON / YAML / TOML / INI editing by dotted path. 6 tools: get / set / unset / merge / validate / detect. Atomic writes via tmp + rename; JSON value coerce; deep merge with replace/append array strategy. YAML preserves comments + key order via Node API. |
| 2026-Q2 | **sb-files: binary + archive** | 4 new tools in sb-files. `binary_strings` (ASCII + UTF-16 LE scan), `binary_format` (magic-byte sniff covering PE / ELF / Mach-O / ZIP-and-subtypes / TAR / GZ / PNG / JPG / WEBP / BMP / PDF / .uasset / SQLite / text), `archive_list` (zip / tar / tar.gz), `archive_extract` with zip-slip protection. |
| 2026-Q2 | **sb cross-cutting** | `mcp.*` self-introspection (servers_list / tools_list / call / audit_search / stats) + `doctor.run(fix?)` with detect+remediate for the v1.14.0 hotfix failure modes (stale *.exe~ backups, manifest.json drift, sprawling sb.exe processes). `scripts/build.sh` gains py_compile pre-flight over every embedded `cmd/**/*.py` so SyntaxErrors fail the build, not runtime. |
| 2026-Q2 | **sb-unreal v1.16.0** | Pure-Python MRQ / WP cells / Niagara list / physics constraint. 7 tools. `mrq_preset_create` + `mrq_render_submit` (in-process PIE executor) + `mrq_queue_list` close the cinematic pipeline end-to-end. `world_partition_cells_list` reads runtime cells. `niagara_emitter_list` and `physics_constraint_*` are scoped to what's Python-bound; deeper mutation tracked as companion C++ follow-up. |
| 2026-Q2 | **sb-unreal v1.15.0** | Level actor attach + material connections read. 5 tools. `level_actor_attach / detach / attachments_list` (rule ∈ snap/keep_world/keep_relative, optional socket). `material_expression_pin_links` + `material_connections_list` — mirror of `bp_node_pin_links` for materials; was the asymmetric gap. |
| 2026-Q2 | **sb-unreal v1.14.0** | Sequencer (Level Sequence) authoring — 10 tools closing the last big read+write coverage gap. Read: `sequencer_bindings_list` / `sequencer_tracks_list` / `sequencer_track_info` / `sequencer_keys_list`. Write: `sequencer_track_add` / `sequencer_track_remove` / `sequencer_section_add` / `sequencer_key_add` / `sequencer_key_remove` / `sequencer_binding_add`. Pure Python via UE 5.7's `MovieSceneScriptingChannel` API — typed key channels reachable from Python without a companion C++ bump. Vocab kept short: track_class accepts `transform / float / bool / integer / byte / vector / color / event / audio / camera_cut / anim / subscene / actor_ref`. Closes the sequencer-authoring spec request. |
| 2026-Q2 | **Companion v1.12.3** | BP compiled bytecode inspector — `bp_function_bytecode(bp_path, function_name)` returns line-by-line disassembly. Covers ~80 common EX_* opcodes; unknown ones as `EX_0xNN (raw)`. Diagnoses 'graph looks right but runtime is wrong' cases. Closes the bp-compiled-bytecode-inspector roadmap request. |
| 2026-Q2 | **Companion v1.13.1** | Macros + interface impl + function locals — 8 new tools closing three Blueprint authoring gaps that real projects hit. Macros: `bp_macros_list` / `bp_macro_create` / `bp_node_macro_instance_target`. Interface implementation: `bp_interfaces_list` / `bp_interface_add` (idempotent) / `bp_interface_remove`. Function locals: `bp_function_locals_list` / `bp_function_local_add`. |
| 2026-Q2 | **Companion v1.13.0** | AnimGraph state machine authoring — six tools (states list / add state / add conduit / add transition / entry state / inner graph). Closes the biggest remaining AnimGraph gap. Transition rule authoring deferred. |
| 2026-Q2 | **Companion v1.12.5** | Bytecode operand decode — extends `bp_function_bytecode` with per-opcode operand decode (FProperty names, UFunction names, typed constants, casts, jumps). ~40 opcodes. Diagnoses 'graph compiles but runtime is wrong' concretely. |
| 2026-Q2 | **Companion v1.12.4** | Polish pass: structured `_expect_*` helpers (every not_found / wrong_class carries an actionable `hint`), universal `limit`/`offset` pagination on list tools (anim_blueprint_nodes / bp_variables_list / anim_blueprint_graphs), back-compat aliases preserve pre-v1.12.4 response keys. Closes the AnimBP output-overflow request. |
| 2026-Q2 | sb-unreal | Tool registry consistency tests (Go) — catches drift between sb_helpers.py / *_tools.go / main.go manifest entries early; current floor 80, ceiling 400, currently 189. |
| 2026-Q2 | **Companion v1.12.2** | `asset_version_info` + `asset_metadata` — first two of the asset-version-and-metadata roadmap. Diagnose 'why won't this asset load' (engine-skew + custom-version mismatch) and read per-object metadata (author hints, tooltips, categories). `level_actors_offline` deferred to bulk-offline-scanner work. |
| 2026-Q2 | **Companion v1.12.1** | Closes the AnimBP / BP component tooling gaps request. `bp_set_component_property_typed` ClassProperty resolution fix (path → `_C` → `GeneratedClass`); new `bp_component_remove` (SubobjectDataSubsystem.delete_subobject + compile + save); `anim_blueprint_nodes` now returns `graph_path` + `node_guid` + `current_asset` per entry; new `anim_reassign_asset(from, to, graph_filter)` for by-value asset rewrite across an AnimBP. |
| 2026-Q2 | **Companion v1.12.0** | Eight spec requests in one drop: BP variable lifecycle (`bp_variable_remove_direct` / `_rename_atomic` / `_retype` — surgical, atomic, no collateral), SkeletalMesh socket authoring + read fixes (`mesh_socket_add` sets parent bone; helper switched to public socket-by-index API), SCS typed property setter (`bp_set_component_property_typed`), runtime PIE invoke (`runtime_invoke`), Enhanced Input + raw key injection (`pie_input_inject`), single-node inspect (`bp_node_inspect_by_guid`), `LoadAssetWithFallback` (defensive AssetRegistry path when EAL goes blind in 5.7.4), AnimMontage clip-swap regression fix (`FArchiveReplaceObjectRef` sweep across the duplicated montage). Build.cs picks up `AssetRegistry`, `EditorScriptingUtilities`, `InputCore`. |
| 2026-Q2 | **Companion v1.11.1** | Two v1.11.0 bug fixes. `anim_node_add` couldn't return the guid for `AnimGraphNode_ControlRig` (and other subclasses that don't expose NodeGuid as a Python attribute) — the node WAS created; only the readback failed. Switched to the companion `get_node_guid` binding. Docstring/example path corrected: Control Rig anim node is `/Script/ControlRigDeveloper.AnimGraphNode_ControlRig`. `anim_node_expose_pin` couldn't toggle Control Rig rig variables because `UAnimGraphNode_CustomProperty` subclasses store bindable-variable pins in `CustomPinProperties`, not `ShowPinForProperties`. Now walks both via `FArrayProperty` reflection. New `anim_node_list_exposable_pins` enumerates both lists with `source` tags so callers can discover the bindable surface. |
| 2026-Q2 | **Companion v1.11.0** | AnimGraph authoring — 5 tools mirroring the `bp_node_*` surface for `UAnimGraphNode_*` (ControlRig / TwoBoneIK / ModifyBone / LayeredBoneBlend / SequencePlayer / …). `anim_node_add` instantiates a node by class path; `anim_node_set_inner_property` writes UPROPERTYs on the inner `FAnimNode_*` runtime struct via `FProperty::ImportText_InContainer` (dot-notation, object/class leaves accept asset paths, reconstruct fires so class-driven pin sets update); `anim_node_expose_pin` toggles `ShowPinForProperties`; `anim_node_info` gives read parity; `anim_node_remove` routes through `bp_node_remove`. Linking reuses existing K2 link tools — AnimGraph nodes ARE UEdGraphNodes. Build.cs picks up `AnimGraph` + `AnimGraphRuntime`. Closes the AnimGraph half of headless authoring. See [animgraph authoring](unreal/animgraph-authoring.md). |
| 2026-Q2 | sb-unreal | 5 transform-query tools — the missing read-side primitive for IK / attachment / VFX alignment. `mesh_sockets_list`, `skeleton_bones_list`, `skeleton_bone_transform`, `mesh_socket_transform`, `actor_transform_query` (with optional `relative_to`). All pure Python — no companion required. See [transform query](unreal/transform-query.md). |
| 2026-Q2 | sb-unreal | `expectedCompanionVersion` externalised to a `expected_companion_version.txt` sidecar next to the binary. Bumping the expected companion version no longer requires rebuilding `sb-unreal.exe` — and so no longer requires a Claude Desktop restart, which is what was triggering the "transport closed" cascade users reported. See [mcp transport recovery](mcp-transport-auto-recovery.md). |

## Companion v1.10.x — Control Rig rig-variable authoring

| Date | Tag | What |
|---|---|---|
| 2026-Q2 | **Companion v1.10.1** | Bug fix for v1.10.0 rig variables. `URigVMController::AddVariableNode` was leaving broken nodes bound to `@@` because variables were added via `FBlueprintEditorUtils::AddMemberVariable` (BP-only) instead of the canonical `URigVMBlueprint::AddMemberVariable` (BP + RigVM model). Switched to the canonical surface for add / remove / list; CPPType comes from `RigVMTypeUtils::CPPTypeFromPinType`. Get/set node creation sanity-checks the returned binding and rolls back any half-bound node. Closes the v1.10.0 follow-up bug report. |
| 2026-Q2 | **Companion v1.10.0** | Rig-variable authoring — completes the AnimBP-facing dynamic-input contract v1.9 left as static-defaults-only. `ControlRigVariableAdd` (FBlueprintEditorUtils::AddMemberVariable + direction-derived CPF flags; public variables become input pins on the AnimGraph Control Rig node), `ControlRigVariablesList` (name / cpp_type / cpp_type_object / direction / default — direction recovered from a `RigVar|Input/Output/Hidden` category tag), `ControlRigVariableRemove` (idempotent), `ControlRigVariableGetNodeAdd` + `SetNodeAdd` (`URigVMController::AddVariableNode` with CPPType + CPPTypeObject derived from the existing BP variable description). 5 MCP tools. Pure-C++ `BPTypeToRigVMType` maps PC_* + sub-object into RigVM's `FVector` / `int32` / `TArray<FTransform>` etc. format. See [control rig authoring → rig variables](unreal/control-rig-authoring.md#rig-variables-v110). |

## Companion v1.9.x — Control Rig (RigVM) authoring

| Date | Tag | What |
|---|---|---|
| 2026-Q2 | **Companion v1.9.0** | Control Rig authoring — wraps `URigVMController` (RigVMDeveloper) so an agent can stand up a procedural-animation rig headlessly. `ControlRigGraphsList` (main RigGraph; function-library deferred to v1.10), `ControlRigNodesList` (name / title / RigUnit struct / position / pin count), `ControlRigAddUnitNode` (FRigUnit-derived `UScriptStruct` → node name), `ControlRigRemoveNode` (idempotent), `ControlRigPinSetDefault` (dot-notation pin path + RigVM text serializer), `ControlRigAddLink` (exec/data link, "Pin.SubPin" paths), `ControlRigCompile` (`RecompileVM` + `MarkBlueprintAsModified`). Uses `*Legacy.h` headers post-Modular-Rig migration. Asset creation rides on pure-Python `ControlRigBlueprintFactory`. 8 MCP tools. Closes the rig-authoring gap — UE 5.7 Python doesn't bind `URigVMController` at all. See [control rig authoring](unreal/control-rig-authoring.md). |

## Companion v1.8.x — headless data authoring

| Date | Tag | What |
|---|---|---|
| 2026-Q2 | **Companion v1.8.0** | Headless data authoring — three coalesced requests closed. **BP/DT**: `SetBlueprintVariableDefault` (typed CDO write — object / class / soft_object / soft_class / gameplaytag / auto), `GetBlueprintVariables` (NewVariables is protected), `GetDataTableRowFieldsJson` (flat dotted-path map). **GameplayTags**: `GameplayTagAdd / Remove / List / Refresh` via `IGameplayTagsEditorModule` + `EditorRefreshGameplayTagTree` — no editor restart per tag addition. **UserDefinedStruct / UserDefinedEnum**: `StructMember*` via `FStructureEditorUtils`, `EnumEntry*` via `FEnumEditorUtils` — closes the DataTable-from-scratch flow. 21 new MCP tools, 11 of which are pure Python (asset creation + dt_export + enum_entries_list). See [data authoring](unreal/data-authoring.md). |

## Companion v1.7.x — AnimMontage notify authoring

| Date | Tag | What |
|---|---|---|
| 2026-Q2 | **Companion v1.7.0** | `AddAnimMontageNotify` (skeleton/named FAnimNotifyEvent + optional notify-state duration, resolves or creates the NotifyTrack, registers the name on the skeleton, idempotent on name+time+track) + `RemoveAnimMontageNotifyByName`. UE 5.7 Python can't write a *named* notify — `AnimationLibrary.add_animation_notify_event` requires a UClass, passing None makes a notify literally named "None". Two MCP tools: `anim_montage_add_notify`, `anim_montage_remove_notify_by_name`. See [asset management → AnimMontage notifies](unreal/asset-management.md#animmontage-notifies). |

## Companion v1.6.x — AnimMontage slot-track authoring

| Date | Tag | What |
|---|---|---|
| 2026-Q2 | **Companion v1.6.0** | `CreateAnimMontageFromTemplate` duplicates a template montage and swaps the AnimSequence inside every slot track's segment for a new clip. Preserves slot layout (e.g. ALS "Arm L"/"Arm R"), group name ("Layering Override Group"), section markers, blend in/out, notifies. Skeleton compatibility check via `IsCompatibleForEditor`. Closes the per-weapon upper-body reload gap — UE 5.7 Python can't author `UAnimMontage::SlotAnimTracks` at all and `AnimMontageFactory` always lands on `DefaultGroup.DefaultSlot` (full-body). One MCP tool: `anim_montage_create_from_template`. See [asset management → AnimMontage from template](unreal/asset-management.md#animmontage-from-template). |

## Companion v1.5.x — DataTable authoring + cold-start

| Date | Tag | What |
|---|---|---|
| 2026-Q2 | **Companion v1.5.0** | DataTable row authoring without struct-text round-trip. `AddDataTableRow` (UDataTable::AddRow + optional binary memcpy from existing row), `SetDataTableRowField` (dotted-path resolver + FProperty::ImportText on leaf — works on UserDefinedStruct fields whose names contain `()` that the stock fill APIs corrupt), `DiscardPackageChanges` (EditorLoadingAndSavingUtils::ReloadPackages — drop dirty in-memory package). Three MCP tools: `dt_row_add`, `dt_row_set_field`, `package_discard_changes`. Closes the gameplay-DataTable authoring gap. See [asset management → DataTable rows](unreal/asset-management.md#datatable-rows). |
| 2026-Q2 | sb-unreal | New tool `editor_launch` — cold-start a closed UE editor. Resolves engine binary from EngineAssociation, spawns detached + polls until alive. Idempotent. Closes the no-tool-for-cold-start gap (workaround was shelling out to `Start-Process UnrealEditor.exe`). |
| 2026-Q2 | sb-unreal | `editor_restart` adds `skip_save_all_dirty` — quit without persisting dirty assets. For recovery from a known-bad in-memory state. |

## Companion v1.4.x — multiplayer authoring

| Date | Tag | What |
|---|---|---|
| 2026-Q2 | **Companion v1.4.0** | Replication authoring. `ConfigureCustomEvent` writes UK2Node_CustomEvent::FunctionFlags (Multicast / Server / Client + Reliable) + `bCallInEditor`. `AddEditableEventParam` builds typed UserDefinedPins on custom events / function entries. `SetVariableReplication` sets CPF_Net / CPF_RepNotify on FBPVariableDescription, sets RepNotifyFunc, auto-creates OnRep_<var> stub function graph. Three MCP tools: `bp_custom_event_configure`, `bp_event_add_param`, `bp_variable_set_replication`. Closes the multiplayer authoring gap — networked features were impossible to author headless before. See [blueprint authoring → replication](unreal/blueprint-authoring.md#replication). |

## Companion v1.3.x — Blueprint authoring depth

| Date | Tag | What |
|---|---|---|
| 2026-Q1 | **Companion v1.3.4** | Object / asset references in Blueprints headlessly: `SetPinDefaultObject` (writes `UEdGraphPin::DefaultObject`), `AddTypedMemberVariable` (full FEdGraphPinType: object/class/soft_*/interface/struct/enum/containers + CDO default), `CreateObjectLiteralNode` (`K2Node_Literal` + `SetObjectRef`). Smart-routing in `bp_node_pin_set_default` for asset-path values. Closes the long-standing "value rejected by schema" failure on object pins. |
| 2026-Q1 | **Companion v1.3.3** | PIE pre-flight: live in-memory BP error scan via `TObjectIterator<UBlueprint>`. AssetRegistry-tag scan was missing BPs whose error was live but not yet serialized. `pie_start` now refuses with a structured error instead of dispatching into UE's modal dialog. |
| 2026-Q1 | **Companion v1.3.2** | Drop `meta=(ScriptMethod)` from pure-static UFUNCTIONs (`CompanionVersion`, `StartPlayInEditor`, `ToggleBetweenPIEAndSIE`, `GetMessageLog`, `ListMessageLogCategories`). UE was warning at every editor launch about ScriptMethod requiring a UObject first-arg. |
| 2026-Q1 | **Companion v1.3.1** | Editor status-bar widget on `LevelEditor.StatusBar.ToolBar`. Shows `● SB v1.3.x` permanently — no need to round-trip `companion_status` to confirm bindings are loaded. |
| 2026-Q1 | **Companion v1.3.0** | Function override authoring — `ListOverridableFunctions` + `OverrideParentFunction`. Wraps `FBlueprintEditorUtils::AddFunctionGraph<UFunction>` so K2Node_FunctionEntry binds to the parent (Parent: super calls work) and FunctionResult is auto-added. |

## Recent: daemon-side improvements

| Date | Component | What |
|---|---|---|
| 2026-Q1 | sb-unreal | `pie_start` AssetRegistry-tag pre-check for BP compile errors (later superseded by v1.3.3 in-memory walk). |
| 2026-Q1 | sb-unreal | `run_python` output pruning: Info entries dropped when `parsed_json` present; 4KB per-entry cap; `Command` echo blanked (was ~150KB). |
| 2026-Q1 | sb-files | `read` accepts `offset` / `limit` matching Claude Code's built-in Read semantics. Response includes `slice_mode`, `start_line`, `end_line`, `total_lines`. |
| 2026-Q1 | sb-unreal | `editor_set_auto_recover` tool — explicit AI override for the crash watcher with optional TTL. |
| 2026-Q1 | sb-unreal | Persistent crash watcher gets three suppression layers (cooldown, build-tool detection, explicit override). `editor_restart` auto-suppresses for its duration. |
| 2026-Q1 | sb-unreal | `editor_message_log` + `editor_message_log_categories` tools (Companion v1.2). |
| 2026-Q1 | sb-unreal | `asset_describe` tool — list all editor properties on any UObject. |
| 2026-Q1 | sb-unreal | `live_coding_compile` is now synchronous + persists `Startup=AutomaticButHidden` via direct INI write + optional `skip_action_limit`. |
| 2026-Q1 | sb-unreal | Persistent crash watcher + auto-recover (initial version). |
| 2026-Q1 | various | Fixed GitHub URL across plugin manifests (`anthropics/SystemBridge` → `o-codify/SystemBridge`). |

## Earlier: Companion v0.x → v1.2.x

| Version | Theme |
|---|---|
| v1.2.0 | FMessageLog access |
| v1.1.0 | PIE lifecycle (StartPlayInEditor, ToggleBetweenPIEAndSIE) |
| v1.0.0 | Stability boundary; full authoring surface |
| v0.9.0 | Blackboard + BehaviorTree authoring |
| v0.8.0 | UMG WidgetTree authoring |
| v0.7.0 | Material expression enumeration |
| v0.6.0 | Subsystem + Enhanced Input |
| v0.5.0 | Cast / MakeStruct / SwitchEnum / repositioning |
| v0.4.0 | Pin defaults, variable refs, event nodes |
| v0.3.0 | Pin introspection, node remove, link break, ReconstructNode |
| v0.2.0 | K2Node graph editing |
| v0.1.0 | Read-only introspection |

See [companion plugin reference](unreal/companion.md) for per-version
detail.

## Breaking changes

- **Companion source URL** (`CreatedByURL` in the `.uplugin`) changed
  from `github.com/anthropics/SystemBridge` to
  `github.com/o-codify/SystemBridge`. No behavior change — metadata only.
- **`bp_variable_add` with non-primitive types now warns / fails
  loudly** (was silent IntProperty). Use
  `bp_variable_add_typed` instead.
- **`unreal_companion_install`** now requires explicit `scope=project|engine`.

## Convention

The agent generally doesn't care about per-Companion version detail until
`editor_status` says `companion_version_mismatch: true`. The "current
expected" is encoded in `expectedCompanionVersion` inside `sb-unreal.exe`
and bumped per release.

## Cross-references

- [companion plugin](unreal/companion.md) — full per-version detail
  with code-level notes.
- [troubleshooting](troubleshooting.md) — drift, broken DLL, missing
  tools.
