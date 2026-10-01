---
name: ketch-websearch
description: Use the ketch CLI instead of built-in web search/fetch tools for browsing, searching, scraping, code search, and library docs lookup. Use this skill whenever a task requires searching the web, fetching a URL's content, crawling a site, searching open-source code, or looking up library documentation.
---

This skill replaces the default web search / web fetch tools with the `ketch` CLI (https://github.com/1broseidon/ketch), a stateless, single-binary agentic research tool. `ketch` is already installed. Prefer it over other browsing tools for all web research tasks in this project.

## When to use

Use `ketch` for:
- **Web search** — general queries, current events, package/library updates, troubleshooting error messages
- **Scraping a known URL** — fetching a page's content (HTML or PDF) as clean markdown
- **Crawling a site** — gathering multiple pages from one domain (e.g. docs sites)
- **Library documentation lookup** — version-aware docs via Context7
- **OSS code search** — grepping real-world code across public repos instead of guessing API usage
- **Already-fetched HTML** — piping HTML you already have (e.g. from `curl`) through extraction without a second fetch

## Core commands

### Search the web
```bash
ketch search "<query>"
```
- `-l, --limit N` — max results (default 5)
- `--scrape` — fetch and extract full content from each result, not just snippets (saves a separate scrape round-trip)
- `--multi[=backend1,backend2]` — federate across multiple backends, rank-fused (RRF); bare `--multi` queries all usable backends
- `--random[=backend1,backend2]` — shuffle backends, try one, fall back on failure; good for spreading rate limits
- `-b, --backend <name>` — pin a specific backend (auto, brave, ddg, searxng, exa, firecrawl, keenable, tavily, parallel, serpbase, degoog, serply, youcom). Mutually exclusive with `--multi`/`--random`.
- `--trim` — strip markdown formatting, keep plain text
- `--minimal` — one result per line, tab-separated (url/title/snippet); good for quick scans or piping into other tools
- `--max-chars N` — truncate markdown output
- `--tag <name>` — bookmark results under a tag for later recall (works with `--scrape` too)

Example — search with full content:
```bash
ketch search "FastAPI OpenTelemetry native support" --scrape --limit 5
```

Example — broader/more resilient results:
```bash
ketch search "breaking change in library X 2.0" --multi
```

### Scrape URL(s)
```bash
ketch scrape <url> [url2 url3 ...]
```
- Accepts one or more positional URLs, a JSON array, a file of URLs, or stdin — no `--batch` flag needed
- `--select "<css selector>"` — extract specific page elements, skips readability
- `--force-browser` — always render via headless Chrome (use for JS-heavy SPAs if auto-detection misses it)
- `--concurrency N` — max concurrent requests for multi-URL scraping (default 5)
- `--no-cache` — bypass the page cache (fetches are cached ~72h by default; repeat scrapes return instantly otherwise)
- `--trim` — plain text output without markdown syntax
- `--max-chars N` — truncate output
- `--raw` — raw HTML instead of markdown (rejected for PDFs)
- `--cookie-file <path>` — attach a Netscape `cookies.txt` jar for pages gated behind login/consent walls
- PDFs are auto-detected (MIME type or `%PDF-` signature) and text-extracted automatically — no extra flag needed

### Crawl a site
```bash
ketch crawl <seed-url> --depth 2 --allow "<path-substring>"
```
- Use for pulling multiple related pages (e.g. a full doc section) in one pass; streams results as found
- `--sitemap` — treat the seed URL as a sitemap instead of BFS-crawling from it
- `--deny <regex>` — exclude matching paths
- `--concurrency N` — worker pool size (default 8)
- `--background` — run long crawls asynchronously; check with `ketch crawl status`, stop with `ketch crawl stop`

### Library documentation
```bash
ketch docs "<library or query>"
```
- Backed by Context7; version-aware curated snippets
- `--library <id>` — skip name resolution if the Context7 library ID is already known
- `--tokens N` — Context7 token budget (default 4000) — raise for deeper lookups, lower to save context
- `--resolve` — just resolve the library name to an ID instead of searching

### OSS code search
```bash
ketch code "<query>" --lang go
```
- Use instead of guessing an API's real-world usage pattern — greps 1M+ public repos (via grep.app by default; no token needed)
- `-b, --backend <grepapp|sourcegraph|github>` — github requires `gh auth login` / `$GITHUB_TOKEN` / configured token
- `--regex` — interpret query as regex (grepapp, sourcegraph only)
- `--minimal` — tab-separated url/repo/snippet for quick scanning

### Extract already-fetched HTML
```bash
curl -L <url> | ketch extract
cat page.html | ketch extract --select article --max-chars 4000
```
- Never fetches, caches, or renders a browser — use when HTML is already in hand to avoid a redundant network round-trip
- `--url <url>` — supply source URL for metadata/relative-link resolution when piping raw HTML

### Bookmarking research (tags)
```bash
ketch search "topic" --scrape --tag my-project
ketch tag show my-project --limit 20
ketch tag add my-project https://example.com/read-later   # no network
ketch tag remove my-project <url>
```
- `--tag` works on `search`, `code`, `docs`, `scrape`, and `crawl`
- Useful for multi-session research — bookmarks survive cache expiry/clear

### Diagnostics
```bash
ketch doctor          # health check of every backend, browser, cache, tag index
ketch config          # discover effective config + active backends as JSON (one call, no --help parsing needed)
ketch cache           # page-cache stats; `ketch cache clear` to reset
```

## Guidance / best practices

- **Default to `ketch search`** for anything time-sensitive (new releases, current prices, breaking changes) instead of relying on built-in knowledge.
- **Use `--scrape` on search** when snippets aren't enough to answer confidently — avoids a separate scrape round-trip.
- **For a single known URL, go straight to `ketch scrape`** rather than searching for it.
- **For library/API documentation questions, prefer `ketch docs`** over generic web search — Context7 results are curated and version-aware, which reduces hallucinated API signatures.
- **For "how is this used in real code" questions, prefer `ketch code`** over search — it returns actual OSS source with repo/line context.
- **Use `--multi` when result quality/coverage matters** (e.g. deep research, conflicting information) and `--random` when you want one provider's results without burning rate limits across all backends.
- **Use `--minimal`** when scanning many results quickly or piping into further processing.
- **Rely on the page cache** (default 72h TTL) for repeated lookups in the same task — don't pass `--no-cache` unless freshness matters (e.g. checking if a fix just shipped) or the content is sensitive/authenticated.
- **For JS-heavy SPAs**, `ketch scrape`/`ketch crawl` auto-detect and fall back to headless Chrome; only pass `--force-browser` if auto-detection misses it. Run `ketch browser install` once if headless Chrome isn't set up yet.
- **For PDFs**, no special handling needed — `ketch scrape` auto-extracts text. Scanned/image-only PDFs will return a precondition error (exit 5) requiring OCR.
- **Use `--tag`** to checkpoint useful sources during multi-step research so they can be revisited without re-searching.
- **Use `--json`** (global flag, available on every command) when output needs to be parsed programmatically rather than read directly.
- **Check exit codes for control flow in scripts**: `2` bad input, `3` not found, `4` upstream/network failure, `5` missing precondition (e.g. no API key, scanned PDF), `6` cancelled.
- Respect robots.txt / site terms as normal; this tool does not bypass access restrictions. Cookie files are the operator's responsibility to use only with their own sessions.
