---
description: >-
  Validate frontend quality (functionality, structure, styles,
  consistency, i18n) against a live rendered app using lightpanda and
  Playwright. Use for: validate frontend, browser test, a11y check,
  design validation, check rendering, frontend quality, accessibility
  check, visual validation.
binds_to_depth: main...
---

# Frontend Validation

Validate frontend quality against a live rendered app via lightpanda and
Playwright.

## Purpose

Use when validating frontend quality — functionality, structure, styles,
consistency, or internationalization — against a live rendered app. Covers
convergent checks (automatable pass/fail: contrast, heading order, landmarks,
target sizes, functional state) and the escalation path for divergent checks
needing manual judgment. Assumes lightpanda (the `lightpanda` MCP server the
`browsing` package declares) and Playwright MCP are available in the session.

## Methodology

**Default-first.** lightpanda is the default instrument — text-only,
Chrome-free, token-cheap. Use it for text, structure, behavior, and
interaction. Escalate to Playwright only for what lightpanda physically cannot
do: pixels, geometry, real cascade, resolved tokens, console, network, and the
viewport matrix.

**Workflow.** (1) Bring the dev server up and note the port. (2) Navigate and
read via lightpanda (`semantic_tree_text`, or `markdown` for a one-call read);
wait for hydration on client-rendered routes. (3) Run convergent checks —
structure and a11y via lightpanda's semantic tree; styles, tokens, contrast,
type-scale, and target-size via Playwright's real cascade. (4) Drive functional
state via lightpanda (`click`/`fill`/`press`) and re-extract to observe it.
(5) Check consistency and localization — same-role-same-name across routes,
per-locale `lang`/`dir`, logical-vs-visual DOM order. (6) Escalate divergent
checks (ARIA correctness, focus-not-obscured, gesture alternatives, painted
balance, channel-loss) to manual, reporting each finding against its named
principle and success criterion.

For depth on any step, read the matched reference below.

## Context

### Browser validation references

- <ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::web_validation@automation/reference/validation_overview.md" />
- <ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::web_validation@automation/reference/accessibility_structure.md" />
- <ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::web_validation@automation/reference/functional_testing.md" />
- <ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::web_validation@automation/reference/consistency_localization.md" />
- <ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::web_validation@automation/reference/visual_system_styles.md" />
- <ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::web_validation@automation/reference/playwright_webkit_facts.md" />
- <ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::browsing@automation/reference/lightpanda.md" />
- <ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::browsing@automation/reference/rendering_models.md" />

### Design principles

In `github.com/ctxmod/design/context`, profile `core`, package `principles`:

- `principles/reference/accessibility_as_quality.md`
- `principles/reference/visual_hierarchy.md`
- `principles/reference/color.md`
- `principles/reference/typography.md`
- `principles/reference/internationalization.md`
