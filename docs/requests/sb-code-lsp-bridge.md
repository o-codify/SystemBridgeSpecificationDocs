---
id: request-sb-code-lsp-bridge-definition-references-hover-diagnostics
title: "Request: sb-code — LSP bridge (definition / references / hover / diagnostics)"
status: request
version: 26.608.1428
tags: [ sb-core, lsp, code-intel, request ]
---

# Request: sb-code — Language Server Protocol bridge

**Origin.** Self-audit. 80% of AI's `Grep` calls are "where is this
symbol used?" / "where is X defined?". Text grep gives false positives
(string literals, comments, partial matches). The Language Server
Protocol answers these structurally per language.

This is **the largest payoff** improvement, and the highest-effort —
filed for execution as a separate spec sprint.

## What

A plugin that manages language-server child processes per project and
exposes their answers as structured MCP tools.

## Tools

| Tool | Purpose |
|---|---|
| `code_definition(file, line, col)` | "Go to definition". Returns `{file, range, content_preview}`. |
| `code_references(file, line, col, include_decl=true)` | "Find all usages". Returns `[{file, line, col, snippet}]`. |
| `code_hover(file, line, col)` | Type info + docstring at position. Returns `{type, doc, signature}`. |
| `code_diagnostics(file?)` | Errors / warnings currently reported. Returns `[{file, severity, line, col, code, message}]`. |
| `code_symbols(file)` | Top-level symbol outline. Returns nested `[{name, kind, range, children}]`. |
| `code_workspace_symbols(query)` | Project-wide symbol search. Returns ranked symbol hits. |
| `code_rename(file, line, col, new_name)` | Cross-file safe rename. Returns `{files_changed, edits: [{file, edits[]}]}`. Apply in same call (`dry_run=false`) or preview. |
| `code_signature_help(file, line, col)` | Parameter list at a call site. |
| `code_completion(file, line, col)` | Completion candidates. Useful for "what methods does this type have?". |

## Why

- **Replaces grep for symbol queries.** "Find every caller of
  `parseManifest`" goes from manual filtering of 50 grep hits (some
  real, some in comments) to one structured answer.
- **Catches bugs without running tests.** `code_diagnostics` is the
  editor's red squiggles via MCP — type errors / import errors found
  before commit.
- **Safe rename.** Cross-file rename without breaking string-literal
  references AI didn't know about.
- **Symbol-first refactoring.** "Inline this function" / "extract
  this expression" become reachable when we have ranges + symbol
  info structurally.

## Supported language servers (priority order)

| Server | Toolchain marker |
|---|---|
| `gopls` | `go.mod` |
| `pyright` (or `pylsp`) | `pyproject.toml` / `setup.py` |
| `typescript-language-server` | `package.json` |
| `rust-analyzer` | `Cargo.toml` |
| `clangd` | `compile_commands.json` |
| `omnisharp` / `csharp-ls` | `*.csproj` |

Detection mirrors `sb-build`. Multi-language project → one server per
language; tool calls route by file extension.

## Implementation route

Go plugin `cmd/sb-code/`. Per-project server manager:

- On first call, start the language server as a child process via
  stdin/stdout JSON-RPC.
- Cache by `(project_root, language)`. Idle eviction after 30 min.
- `didOpen` / `didChange` / `didClose` driven by file events from
  the `files_watch` system (or per-call open-on-demand).
- All LSP requests serialised per-server (LSP is single-threaded
  per session).
- Health checks: ping `initialize` every 5 min; auto-restart on crash.

Server discovery — first PATH, then well-known install dirs (e.g.
`%USERPROFILE%\go\bin\gopls.exe`), then a structured "server not
installed; try `go install golang.org/x/tools/gopls@latest`" error.

## Acceptance

- `code_references("cmd/sb-unreal/main.go", 42, 12)` for a function
  symbol returns every call site across the repo, including in test
  files.
- `code_diagnostics()` in a Go project with a deliberate type error
  returns `[{severity: "error", message: "undefined: foo"}]`.
- `code_rename` previews changes and `apply=true` writes them
  atomically (no half-renamed state).
- A project with no LSP installed returns a structured error pointing
  at the install command, not a stack trace.

## Non-goals (defer)

- Code-action support (LSP "quick fixes").
- Live linting via diagnostics push (event stream model).
- Multi-root workspaces — one root per project is enough.
- Formatting through LSP — `sb-build` `format_run` already covers this.

## Effort estimate

1–2 weeks of focused work — language-server lifecycle is the bulk
(handshake, capability negotiation, document sync, crash recovery).
Per-language schema mismatches (some servers don't implement every
method) need fallbacks.
