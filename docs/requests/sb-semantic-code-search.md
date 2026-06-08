---
id: request-sb-semantic-semantic-style-code-search-bm25-identifier-aware
title: "Request: sb-semantic — semantic-style code search (BM25 + identifier-aware)"
status: request
version: 26.608.1507
tags: [ sb-core, code-search, indexing, request ]
---

# Request: sb-semantic — semantic code search

**Origin.** Comparative scan: Zilliz's `claude-context` hybrid (BM25 +
dense vector) reports ~40% token reduction on code Q&A. SB has grep
(literal) and `search_find_symbol` (definition only). Missing: fuzzy
"what does the auth flow do?" / "find code similar to this" queries.

## What

A search plugin that pre-indexes the project once and answers fuzzy
intent queries fast — without requiring an external embedding model
in v1.

Strategy: **BM25 + identifier-aware tokenization + recency boost**.
Treat each function / class / file as a document; tokenize by
identifier-split (`camelCase` / `snake_case` / `kebab-case`) so a
query like "parse blueprint manifest" matches `parseBlueprintManifest`
without lemmatisation.

Neural embeddings (the "true" semantic part) are a separate v2 ship —
this v1 already gets the user 80% of the way there without dragging
in PyTorch / ONNX.

## Tools

| Tool | Purpose |
|---|---|
| `code_index(root?, force=false)` | Build / refresh the index. Returns `{indexed_files, indexed_symbols, duration_ms, index_size_bytes}`. Auto-detects project root. |
| `code_search(query, k=10, kind?: function/class/file/comment, language?)` | Ranked hits: `{score, file, line, kind, name, snippet}`. |
| `code_similar(file, k=10)` | Files most similar to the given one by token overlap (BM25 against file `file`'s representation). |
| `code_index_stats()` | `{root, indexed_at, files, symbols, languages: {go:1234, ...}}`. |
| `code_index_clear()` | Drop the index. |

## Tokenisation

- Split identifiers: `parseBlueprintManifest` → `parse`, `blueprint`,
  `manifest`. `snake_case` and `kebab-case` handled too.
- Preserve compound forms as additional tokens (so exact-match queries
  still hit).
- Comments parsed separately — query can filter by `kind:comment`.
- Drop noise tokens (`func`, `var`, `if`, `return`, language stopwords).

## Index storage

SQLite, one DB per project, lives at `<project>/.sb_index.sqlite`.

Schema:
```
files       (id, path, sha256, size, indexed_at, language)
symbols     (id, file_id, kind, name, line, snippet_hash)
tokens      (id, symbol_id, token, tf)            -- term-frequency
df_tokens   (token PRIMARY KEY, df)                -- document-frequency
```

BM25 scoring runs in SQL with idf precomputed.

## Why (vs grep)

- "Find the function that sets up MCP transport" → grep needs the
  exact word. `code_search("mcp transport setup")` ranks
  `initMCPTransport`, `setupMCPServer`, etc.
- "Files similar to this one" → grep can't.
- "What does this codebase do with users?" → keyword-fuzzy answer.
- Ranking by relevance, not file order.

## Why (vs full neural embeddings)

- No model download (gigabytes for decent quality).
- No GPU.
- No external API calls (the user excluded LLM-bridge).
- BM25 + good tokenisation gets shockingly far on **identifier-
  dense** corpora — code is mostly named-entity matching.

## Implementation route

Go plugin `cmd/sb-semantic/`:

- Walk + parse via existing `sb-search` parser hooks (Go/Python/TS/
  Rust/C++ already supported there).
- Tokeniser: shared with `sb-files outline` where possible.
- SQLite via `modernc.org/sqlite` (no cgo).
- Index incremental: re-index only files whose mtime > `indexed_at`.

## Safety

- Pure read on indexing pass.
- Index file is gitignored by convention (`.sb_index.sqlite`).
- No telemetry; index never leaves disk.

## Acceptance

- `code_index()` on a 1000-file Go project finishes in <5 seconds.
- `code_search("parse blueprint manifest")` ranks the actual parser
  function above unrelated `parse*` matches.
- `code_similar(file)` returns files in the same package above
  unrelated ones.
- Repeated `code_index` after edits re-indexes only the changed files
  (verifiable via `duration_ms` cliff).

## Non-goals (defer for v2)

- True dense-vector embeddings (would need ONNX model + RAM).
- Cross-project federation.
- Language-server-grade type-aware ranking — that's the `sb-code` LSP
  bridge request.
