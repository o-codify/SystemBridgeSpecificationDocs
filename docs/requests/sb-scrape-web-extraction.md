---
id: request-sb-scrape-structured-web-scraping-js-rendered-selector-crawl
title: "Request: sb-scrape — structured web scraping (JS-rendered + selector + crawl)"
status: request
version: 26.608.1507
tags: [ sb-core, web, scrape, request ]
---

# Request: sb-scrape — structured web extraction

**Origin.** SB has `network_http_request` (raw HTTP) and the Chrome
MCP (interactive browser). The missing middle: programmatic structured
extraction — fetch a URL, render JS if needed, pick fields by CSS
selector, convert to clean markdown / structured JSON.

## What

Plugin that wraps headless Chrome (already in dep tree via `chromedp`)
for read-only structured extraction. Like `firecrawl` / `puppeteer`
MCP servers, but in-process.

## Tools

| Tool | Purpose |
|---|---|
| `scrape_html(url, selector?, render_js=true, timeout_s=10)` | Fetch + render; if `selector` set, return only matching elements' `outerHTML`. |
| `scrape_text(url, selector?, render_js=true, format="markdown")` | Convert to clean text. `format` ∈ `markdown` / `plain` / `readability`. |
| `scrape_fields(url, schema: {field: selector}, render_js=true)` | Extract a JSON object: each `field` maps to a CSS selector; value is the matched element's `innerText`. |
| `scrape_links(url, render_js=true, same_origin=false)` | All `<a href>` on the page. |
| `crawl_site(root_url, max_pages=20, max_depth=2, same_origin=true, fields?)` | Multi-page crawl with optional `fields` extraction per page. Returns `[{url, status, fields?, links}]`. |
| `scrape_screenshot(url, full_page=true, width=1280, height=800)` | PNG screenshot. Returns attachment ref. |

## Why

- "What's on this page right now?" — `scrape_text` returns clean
  markdown one call, no HTML parsing.
- "Pull all product names + prices" — `scrape_fields({name: ".title",
  price: ".price"})` in one call.
- "Map this docs site" — `crawl_site` walks depth-2 in seconds.
- Reading API docs / release notes / blog posts — currently done via
  WebFetch, which loses structure and can't render JS.

## Implementation route

Go plugin `cmd/sb-scrape/`:

- `github.com/chromedp/chromedp` — already in go.mod (used by
  sb-browser). Reuse the headless Chrome process.
- HTML → markdown: `github.com/JohannesKaufmann/html-to-markdown`.
- Readability: `github.com/go-shiori/go-readability`.
- CSS selector engine: `github.com/PuerkitoBio/goquery`.

Render flow: `chromedp.Navigate` → `chromedp.WaitReady("body")` →
`chromedp.OuterHTML(selector, ...)`. Per-call browser context;
timeout-bounded.

## Safety

- `scrape_*` is read-only; no permission gate.
- Robots.txt respected by `crawl_site` (skip disallowed paths).
- Request rate-limit: 1 req / 250ms across `crawl_site`.
- Per-domain page cap: `crawl_site` won't exceed 100 pages even if
  asked.
- Audit log records URL only (no body).

## Acceptance

- `scrape_text("https://example.com")` returns clean markdown without
  nav/footer noise.
- `scrape_fields(url, {title: "h1", body: "article"})` returns the
  structured object.
- `crawl_site("https://docs.foo.io", max_pages=10)` returns up to 10
  pages with internal links and page text.
- `render_js=true` correctly waits for SPA content; `render_js=false`
  uses a plain HTTP GET (faster for static sites).
- `scrape_screenshot` returns a viewable PNG via attachments.

## Non-goals (defer)

- Authenticated scrapes (cookie / login flow) — Chrome MCP covers it.
- Anti-bot evasion (rotating UA, CAPTCHA solving).
- Sitemap-driven indexing (we have `crawl_site` for now).
- PDF rendering from HTML.
