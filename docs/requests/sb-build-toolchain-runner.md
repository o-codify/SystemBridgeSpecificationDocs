---
id: request-sb-build-auto-detect-build-test-lint-runner
title: "Request: sb-build — auto-detect build / test / lint runner"
status: request
version: 26.608.1427
tags: [ sb-core, build, test, lint, request ]
---

# Request: sb-build — auto-detect toolchain runner

**Origin.** Self-audit. Every dev session ends up shelling out for
`go build` / `go test` / `npm install` / `npm test` / `cargo build`
/ `pytest` / `make`. Each AI client implements its own quirky parse
of "did the build succeed". Structured output once at the SB layer
would feed every tool.

## What

A single plugin that detects the project's build toolchain by markers
and exposes a uniform `{success, stdout, structured}` surface across
languages.

## Tools

| Tool | Purpose |
|---|---|
| `build_run(target?, args?)` | Build the project. Auto-detects from markers. Returns `{success, exit_code, toolchain, duration_ms, output, errors[]}`. |
| `tests_run(filter?, package?, verbose=false)` | Run tests. Parses each toolchain's output into `{passed, failed, skipped, failures: [{name, file, line, message}]}`. |
| `lint_run(fix=false)` | Run the project linter. Returns `{issues: [{file, line, severity, code, message}]}`. |
| `format_run(check=false)` | Run the formatter. With `check=true`, returns the list of files that would change. |
| `deps_outdated()` | List packages with newer versions: `[{name, current, latest, kind}]`. |
| `deps_install(name?, version?, kind="prod"\|"dev")` | Add a package. No args → install from lockfile. |
| `deps_upgrade(name?)` | Upgrade one package or all to latest compatible. |
| `toolchain_detect()` | Standalone "what kind of project is this?". |

## Toolchain detection (markers in priority order)

| Marker | Toolchain |
|---|---|
| `go.mod` | Go |
| `package.json` | Node (npm / pnpm / yarn — sub-detected from lockfile) |
| `Cargo.toml` | Rust |
| `pyproject.toml` / `setup.py` / `requirements.txt` | Python (uv / poetry / pip — sub-detected) |
| `*.csproj` / `*.sln` | .NET |
| `pom.xml` / `build.gradle*` | Java / Kotlin |
| `Makefile` | make (fallback) |
| `.uproject` | Unreal (delegate to `unreal_project_build`) |

Multiple detected → returns the highest-priority and notes the others
in `{toolchain, others_detected: [...]}`.

## Why

- AI shouldn't have to know "is this jest or vitest?" — it just calls
  `tests_run("LoginPage")`.
- Structured failure list = AI can decide which file to look at
  without re-parsing stdout.
- Cross-language uniformity = code-review / CI-style flows generalise.

## Implementation route

Go plugin `cmd/sb-build/`. Each toolchain is a small adapter
(`build_go.go`, `build_node.go`, …) implementing one interface:

```go
type toolchain interface {
    Detect(root string) (bool, map[string]any)
    Build(ctx, args) (Result, error)
    Test(ctx, filter, pkg) (Result, error)
    Lint(ctx, fix bool) (Result, error)
    Format(ctx, check bool) (Result, error)
    DepsOutdated(ctx) ([]Dep, error)
    DepsInstall(ctx, name, version, kind string) error
    DepsUpgrade(ctx, name string) error
}
```

Test output parsing — per toolchain, but every adapter normalises to
the same `{passed, failed, skipped, failures: [{name, file, line, message}]}`
shape.

## Acceptance

- `tests_run("Login")` in a Jest project returns `{passed: 5, failed: 1,
  failures: [{name: "Login renders email field", file: "Login.test.tsx",
  line: 14, message: "expected ... but got ..."}]}`.
- `build_run()` in `cmd/sb-unreal` runs `go build ./cmd/sb-unreal` and
  returns `{success: true, toolchain: "go", duration_ms: ~2000}`.
- `deps_outdated()` in a Node project returns the same list as
  `npm outdated --json` but normalised.
- `lint_run()` against a misformatted Go file lists the offences with
  file:line:col.

## Non-goals (defer)

- Test debugger (run with debugger attached).
- Bazel / Buck — long tail.
- Cross-toolchain mono-repos (one toolchain per call is enough).
- Watch mode (`tests_run --watch`) — stateful, separate request.
