# Browser automation overview

The `automation` profile is the umbrella tier of the agent web-browsing and
scraping stack: agent-driven web browsing and scraping concepts — Lightpanda
light browsing, PandaScript extraction, and Chrome-free scraping patterns. It
provides the *concepts* and *reference material* an agent needs to browse and
extract web content efficiently — without committing to a frontend framework,
a harness, or a specific browser backend.

It is deliberately upstream of plumbing:

- **MCP server wiring.** The `browsing` package declares the Lightpanda
  (light/text, Chrome-free) stdio MCP server with a curated tool subset. The
  Playwright (visual, Chrome-dependent) MCP server is the escalation path for
  visual, cascade, screenshot, console and network work; a harness wires it
  (or uses the token-cheaper `playwright-cli` — see
  <ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::browsing@automation/reference/playwright.md" />).
  The reference context here is what makes those tools and PandaScript usable.
- **Frontend implementation** (React, Tailwind, shadcn) belongs to the
  frontend-framework modules (`github.com/ctxmod/typescript/context`), which
  require this module so a frontend-web session also gains browsing and
  scraping context for design reference gathering and content extraction.
- **Standalone scraping** (no frontend) adopts this profile directly.

## Two-tier architecture

```
automation            ← this profile: browsing/scraping concepts + references
  └── typescript      ← frontend-web: also gets browsing context
```

Each tier specializes the one above it. A scraping-only session takes
`automation`; a frontend-web session takes the typescript module's profiles
and inherits browsing context.

The browser references cite design-tier vocabulary throughout —
accessibility, color, typography, spacing, the component taxonomy — so this
profile extends the design module's `web` profile, and adopting `automation`
delivers the design context those citations resolve against.

## Why a profile, not a global reference

Lightpanda and PandaScript are relevant to two audiences — frontend-web
(visual reference gathering, design extraction) and general web scraping
(data extraction, research). A profile scopes the reference to sessions that
opt in, and frontend-web inherits it without duplicating content. Other
browsing and scraping references can join this profile later.
