---
id: request-sb-docker-docker-compose-container-management
title: "Request: sb-docker — Docker / compose container management"
status: request
version: 26.608.1506
tags: [ sb-core, docker, containers, request ]
---

# Request: sb-docker — container management

**Origin.** Standard dev-environment surface. SB today has zero Docker
tooling; AI uses `docker ps` / `docker logs` / `docker exec` via Bash
constantly.

## What

Go plugin that talks to the Docker daemon (REST over Unix socket /
Windows named pipe) and exposes container + compose operations.

## Tools

### Containers

| Tool | Purpose |
|---|---|
| `docker_ps(all=false, filter?)` | List containers. Each: `{id, name, image, state, status, ports, created_at}`. |
| `docker_inspect(container)` | Full inspect JSON for one container (image, env, mounts, networks, config). |
| `docker_logs(container, tail=100, since?, follow=false)` | Stdout+stderr; `follow=true` returns an attachment ref (streaming). |
| `docker_exec(container, cmd, args?, stdin?)` | One-shot exec. Returns `{stdout, stderr, exit_code, duration_ms}`. Gated by `write_docker`. |
| `docker_start(container)` / `docker_stop(container, timeout_s=10)` / `docker_restart(container)` | Lifecycle. Idempotent. Gated. |
| `docker_rm(container, force=false)` | Remove. Gated. |
| `docker_stats(container?)` | CPU / mem / net / block I/O snapshot. |

### Images / Volumes / Networks

| Tool | Purpose |
|---|---|
| `docker_images_list(filter?)` | Image list with sizes + tags + layers. |
| `docker_image_inspect(image)` | Full image inspect. |
| `docker_image_pull(ref)` / `docker_image_rm(image, force=false)` | Gated. |
| `docker_volumes_list()` / `docker_volume_inspect(name)` | |
| `docker_networks_list()` / `docker_network_inspect(name)` | |

### Compose

| Tool | Purpose |
|---|---|
| `compose_up(file?, services?, detach=true)` | `docker compose up` wrapper. Resolves to `docker-compose.yml` in cwd if `file` omitted. |
| `compose_down(file?, remove_volumes=false)` | |
| `compose_ps(file?)` | Compose-aware ps — groups by service. |
| `compose_logs(file?, service?, tail=100)` | |
| `compose_build(file?, services?, no_cache=false)` | |

## Why

- `docker ps` + parse output is the #1 daily Bash escape for
  back-end / DevOps sessions.
- `docker_inspect` + `docker_logs` are the triage staples — one tool
  call vs `docker inspect | jq ...` shell ritual.
- Compose authoring: the AI can `compose_up` + `compose_ps` + verify
  services healthy in one fast loop.

## Implementation route

Go plugin `cmd/sb-docker/`:

- `github.com/docker/docker/client` — official SDK. Talks over local
  socket (`unix:///var/run/docker.sock`) or Windows named pipe
  (`npipe:////./pipe/docker_engine`).
- Compose: shell out to `docker compose` (Compose v2 is plugin-style;
  no stable Go API surface). One subprocess; structured output via
  `--format json`.
- Connection-test on plugin start; degraded mode (`docker_available:
  false`) if daemon's not running.

## Safety

- Read-only by default. Write tools (`exec`, `start`, `stop`, `rm`,
  `image_pull`, `compose_up`, `compose_down`) gated by `write_docker`.
- `docker_exec` warns if container runs as root + cmd looks
  destructive (`rm -rf`, etc.).
- `compose_up` refuses if the compose file lives outside cwd.

## Acceptance

- `docker_ps()` returns the same containers `docker ps -a` shows.
- `docker_logs("myapi", tail=50)` returns last 50 lines.
- `docker_inspect` returns parsed JSON ready for structural queries.
- `compose_up` brings up services AND `compose_ps` reflects them as
  running.
- Plugin starts cleanly when Docker daemon's not running and reports
  `docker_available: false`.

## Non-goals (defer)

- Build context streaming (`docker_build`) — heavy, can shell out to
  `docker build` if needed.
- Swarm mode.
- Remote daemons (TLS auth) — local only for v1.
- Buildx / multi-platform builds.
