---
id: plugin-research
title: "Plugin: research"
status: stable
version: 26.608.1655
tags: [ plugin, research, stackoverflow, wikipedia, npm, pypi ]
---

# Plugin: `research`

Federation to common research backends — Stack Overflow, Hacker News,
Wikipedia, npm, PyPI, crates.io, go.dev. Each adapter translates the
source's JSON into a uniform `{source, query, hits: [...]}` (or
`{source, name, latest, description, ...}` for registry lookups).

No API key required for any v1 source. stdlib `net/http` +
`encoding/json` — zero new deps.

## Tools

| Tool | Source | Purpose |
|---|---|---|
| `research.stackoverflow(query, tags?, top?)` | api.stackexchange.com | Ranked Q&A with score, accepted flag, tags. |
| `research.hackernews(query, type?, top?)` | hn.algolia.com | Stories or comments. `type` ∈ story / comment. |
| `research.wikipedia(query, lang?, summary_only?)` | wikipedia.org REST | Search + optional top-hit summary. |
| `research.npm(package)` | registry.npmjs.org | Latest version, description, license, version count. |
| `research.pypi(package)` | pypi.org JSON API | Latest version, summary, homepage, project URL. |
| `research.crates(crate)` | crates.io API v1 | Latest version, downloads, repository, license. |
| `research.godoc(import_path)` | proxy.golang.org + pkg.go.dev | Version list + doc URL. |

## Examples

```
research.stackoverflow({query: "go context cancel", tags: "go", top: 3})
→ count=3, quota_remaining=998, hits=[
    {question_id, title, link, score, answer_count, is_answered, tags, ...}
  ]

research.pypi({package: "requests"})
→ {name: "requests", latest: "2.34.2", description: "Python HTTP for Humans.",
   homepage: "...", version_count: 195}

research.wikipedia({query: "Unreal Engine", summary_only: true})
→ {count: 5, results: [...], top_summary: {title, extract, url}}
```

## Rate limits

Each adapter honours the source's documented limit:

- StackExchange: 30 req/sec without API key, 10000/day.
- HN Algolia: no documented limit (practically generous).
- Wikipedia: 200 req/sec.
- Package registries (npm / PyPI / crates / go proxy): generous.

A 429 maps to `errcodes.RateLimited` (recoverable) with a "back off"
suggestion.

## Errors

All adapters use `errcodes`:

| Wire | Code | Recoverable |
|---|---|---|
| 404 | `not_found` | no |
| 429 | `rate_limited` | yes |
| transport failure | `network` | yes |
| 5xx / parse failure | `internal_error` | no |
| missing required arg | `validation_failed` | no |

## Cross-references

- [Plugin: scrape](scrape.md) — generic HTML extraction (when no API)
- [Plugin: network](network.md) — raw HTTP
