---
id: request-research-federation-stackexchange-hackernews-wikipedia
title: "Request: research federation (StackExchange / HackerNews / Wikipedia)"
status: request
version: 26.608.1636
tags: [ sb-core, research, web, request ]
---

# Request: research federation surface

**Origin.** github-inside-claude-code's `gh_research_*` — federates
to StackExchange / Wikipedia / HackerNews / Tavily for AI-friendly
research queries.

SB has `sb-scrape` (chromedp HTML) and `network_http_request` (raw),
but no convenience wrappers for the few common research backends.

## What

A small plugin `sb-research` with specialised JSON-aware adapters.
No new infrastructure — just typed wrappers over their public APIs.

## Tools

| Tool | Purpose |
|---|---|
| `research_stackoverflow(query, tags?, top=5)` | StackExchange API; returns ranked Q&A summaries. |
| `research_hackernews(query, type="story"|"comment", top=10)` | Algolia HN API. |
| `research_wikipedia(query, lang="en", summary_only=true)` | MediaWiki API. |
| `research_npm(package)` | npm registry metadata + last 5 versions. |
| `research_pypi(package)` | PyPI metadata. |
| `research_crates(crate)` | crates.io metadata. |
| `research_godoc(import_path)` | go.dev package summary. |
| `research_man_page(name, section?)` | manpages.org HTML→text. |

## Why dedicated tools

- StackOverflow API returns structured `{score, answer_count, accepted,
  excerpt}` — no HTML parsing needed.
- Package registries (npm/PyPI/crates) need different endpoints; one
  tool per source is cleaner than a parameterised generic.
- AI can call these without thinking about rate limits / auth (each
  adapter handles).

## Implementation route

Go plugin `cmd/sb-research/`. Each adapter is ~50 lines:
- Build URL with query
- GET, parse JSON
- Translate to a uniform `{source, query, hits: [...]}` shape

No external deps — stdlib `net/http` + `encoding/json` per source.

## Rate limits

Each adapter honours the source's documented limit. StackExchange:
30 req/sec without API key, 10000/day with. HN Algolia: no limit
practically. Wikipedia: 200 req/sec.

## Acceptance

- `research_stackoverflow("go context cancel", top=3)` returns 3
  ranked Q&A summaries with scores + links.
- `research_pypi("requests")` returns version list + summary.
- Rate-limit hit returns `errcodes.RateLimited` (not raw HTTP 429
  message).

## Non-goals (defer)

- Tavily / SerpAPI / paid search backends (API key management out of
  scope for v1).
- Caching of results.
- Cross-source ranking ("ask all 3, merge").
