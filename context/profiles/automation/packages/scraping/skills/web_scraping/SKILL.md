---
description: >-
  Extract structured data from web pages using lightpanda PandaScript
  (page.extract, navigate-and-read, list-to-detail) and the
  rendering-model wait strategy, escalating to Playwright only for
  JS-execution, console, network, or screenshot needs. Use for: scrape,
  extract data, web scraping, crawl, pandascript, data extraction.
gated_by_depth: main...
---

# Web Scraping

Extract structured data from web pages via lightpanda PandaScript and
Playwright.

## Purpose

Use when extracting structured data from web pages — list-to-detail
scraping, auth-gated content, CSR/JS-rendered content, or structured
extraction at scale. Covers the lightpanda-first extraction methodology
(PandaScript `page.extract` for deterministic token-free scraping;
`markdown`/`semantic_tree_text` for quick reads) and the escalation path to
Playwright for JS execution, console, network, or screenshot needs. Assumes
lightpanda (the `lightpanda` MCP server the `browsing` package declares) and
Playwright MCP are available in the session.

## Methodology

**lightpanda-first.** PandaScript is the default extraction instrument —
deterministic, LLM-free, token-free, Chrome-free. Use `page.extract` for
structured data (object/array schemas against selectors), `markdown {url}`
or `fetch --dump semantic_tree_text` for one-call navigate-and-reads.
Escalate to Playwright only when scraping needs JS execution the page
itself won't run, console output, network capture, or screenshots.

**Workflow.** (1) Detect the rendering model (SSR content is in the first
byte; CSR ships an empty mount) and wait accordingly — `goto` returns at
`load`, so CSR/hybrid routes need `waitForSelector`/`waitForState` before
`extract` or you read an empty tree. (2) For auth-gated content, drive
the login flow with `$LP_*` credential placeholders in `fill` values,
then `waitForSelector` the post-auth marker before extracting. (3) The
canonical shape is list-to-detail: extract the index list, loop in the
agent script visiting each detail page, extract per-item fields, hold
results in local variables. (4) End the script with the bare result
expression — objects/arrays print as JSON automatically.

For depth on any step, read the matched reference in Context below.

## Context

### Browser scraping references

- <ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::browsing@automation/reference/lightpanda.md" />
- <ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::web_validation@automation/reference/functional_testing.md" />
- <ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::browsing@automation/reference/rendering_models.md" />
- <ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::web_validation@automation/reference/validation_overview.md" />
