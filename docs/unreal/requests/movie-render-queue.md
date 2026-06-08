---
id: request-movie-render-queue-mrq-preset-author-render-submit
title: "Request: Movie Render Queue (MRQ) — preset author + render submit"
status: request
version: 26.608.1430
tags: [ unreal, sequencer, render, cinematic, request ]
---

# Request: Movie Render Queue authoring + submit

**Origin.** v1.14.0 closed Sequencer authoring (`sequencer_*`). The
last cinematic-pipeline mile — actually **rendering** the sequence
to disk — is not headless yet.

## What

Movie Render Queue (MRQ) is the modern cinematic render pipeline.
Mirror the existing sequencer surface for it.

## Tools

| Tool | Purpose |
|---|---|
| `mrq_preset_create(path, settings: {})` | Create a `UMoviePipelineMasterConfig` preset asset with output format / resolution / sample counts / passes. |
| `mrq_preset_get(preset_path)` | Read back preset settings. |
| `mrq_preset_set(preset_path, settings: {})` | Modify preset. |
| `mrq_render_submit(sequence_path, level_path?, preset_path, output_dir)` | Add a job to the MRQ and start rendering. Returns `{job_id}`. |
| `mrq_render_status(job_id)` | Poll job status: `{progress, current_frame, total_frames, eta_ms, last_error?}`. |
| `mrq_render_cancel(job_id)` | Cancel a running job. |

## Why

End-to-end cinematic pipeline: author the sequence (already shipped) →
render to MP4 / EXR sequence (this request). The blocker today is
that even after the agent authors a perfect LevelSequence, rendering
needs a human to open MRQ in the editor.

Concrete uses:

- Headless animation review reels.
- Per-asset preview videos in CI.
- Replay / highlight system rendering match recaps.

## Implementation route

Companion C++ + Python. The `MoviePipeline` module exposes:

- `UMoviePipelineMasterConfig` — preset asset.
- `UMoviePipelineQueue::AllocateNewJob` — add jobs to the global queue.
- `UMoviePipelineExecutorBase` — kick off rendering (in-process or
  out-of-process via `UMoviePipelinePIEExecutor` / new render process).
- Progress callbacks via `OnIndividualJobFinished` / `OnFinished`.

Module deps: `MovieRenderPipelineCore`, `MovieRenderPipelineEditor`.

Render status polling — store a small in-process job table in the
companion; expose via `mrq_render_status`.

## Acceptance

- `mrq_render_submit("/Game/Cinema/LS_Demo", preset:"/Game/Cinema/MRQ_HD",
  output_dir:"D:/renders/")` produces a sequence of EXR / MP4 files
  in `output_dir` matching the preset.
- `mrq_render_status` reflects frame-by-frame progress.
- Settings round-trip: `mrq_preset_set` → `mrq_preset_get` returns the
  same settings.

## Non-goals (defer)

- Distributed / farm rendering (Movie Render Pipeline Render Farm).
- Custom output passes / extensions.
- Real-time MRQ presets editing while a job is running.
