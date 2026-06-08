---
id: request-watch-streaming-pattern-for-long-running-observers
title: "Request: watch / streaming pattern for long-running observers"
status: request
version: 26.608.1635
tags: [ sb-core, streaming, events, tasks, request ]
---

# Request: watch / streaming pattern for long-running ops

**Origin.** github-inside-claude-code's `gh_watch_pr / gh_watch_ci /
gh_watch_workflow` — long-running observers that stream events. SB
has request/response and `tasks.start` async wrapper; missing piece is
a clean "subscribe to events" pattern for CI watches, build watches,
file watches, render queue progress.

## What

A `watch_*` convention that:

1. Starts a watcher and returns `{watch_id, poll_url?}` immediately
2. Watcher accumulates events in the existing `eventbuf.Ring` per
   plugin
3. AI calls `watch_poll(watch_id)` to drain since last cursor
4. AI calls `watch_stop(watch_id)` to terminate

```
{
  "id":"watch_a1b2",
  "status":"running",
  "started_at":"...",
  "next_cursor":"42",
  "events":[
    {"kind":"step_started","ts":"...","payload":{"step":"build"}},
    {"kind":"step_succeeded","ts":"...","payload":{...}}
  ]
}
```

## Tools (core surface)

| Tool | Purpose |
|---|---|
| `watch.list()` | All active watchers across plugins. |
| `watch.poll(watch_id, since_cursor?, max=100)` | Drain events since cursor. |
| `watch.stop(watch_id)` | Terminate a watcher. |
| `watch.status(watch_id)` | Snapshot. |

Plugins implement watchers as goroutines and report progress via the
existing `EmitEvent` mechanism — no SDK change needed, just convention.

## Pattern adopters

- `sb-github.github_pr_watch_checks(pr)` → polls Actions runs every 3s
- `sb-build.tests_watch(filter)` → streams test progress live
- `sb-docker.compose_logs_follow(file)` → tails logs
- `sb-unreal.pie_watch()` → already exists, would conform to convention

## Acceptance

- `watch.poll` returns event delta + next cursor; idempotent on the
  cursor.
- `watch.stop` terminates the goroutine cleanly.
- A killed plugin auto-terminates its watchers.
- `watch.list` shows source plugin, started_at, latest event time per
  active watcher.

## Non-goals (defer)

- Server-sent events (SSE) — polling is sufficient for v1.
- Cross-plugin event federation.
- Persistent watch state across sb.exe restarts.
