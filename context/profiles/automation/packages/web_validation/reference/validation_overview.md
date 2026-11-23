# Frontend Validation Overview

Entry point for frontend-validation references — the honest lightpanda-vs-Playwright capability map, the default-first rule (lightpanda for text/structure/behavior, Playwright for pixels/geometry/cascade/console), the canonical convergent-check workflow, and what stays manual.

This is the entry point for the `automation` profile's validation references. It
operationalizes the design-principle corpus's **"How to verify"** sections:
turning normative obligations (contrast, heading order, landmark roles, target
sizes, focus, hierarchy) into mechanical checks an agent runs against a live
rendered frontend using the two browser tools available to this profile —
**lightpanda** (text-only, Chrome-free, token-cheap; the `lightpanda` MCP
server the `browsing` package declares) and **Playwright MCP** (real layout
engine, visual).

The principle files remain the *normative* source; these references are the
*operational* companion. They do not restate principle content — they cite it
and show how to check it.

## Purpose: convergent vs. divergent

`principles/reference/accessibility_as_quality.md` in `github.com/ctxmod/design/context` (profile `core`) splits
verification into two kinds, and that split is the backbone of every reference
here:

- **Convergent checks** automate well — deterministic pass/fail signals. Contrast
  ratios (SC 1.4.3 / 1.4.6 / 1.4.11), heading order, landmark roles, accessible
  names, declared target sizes, `lang`/`dir`, meta. Automated tools detect only
  ~25–30% of WCAG criteria, but these are the ones they detect.
- **Divergent checks** need manual judgment — a tool cannot decide them.
  Correctness of ARIA patterns and announced state, pointer-gesture and dragging
  alternatives (SC 2.5.1 / 2.5.7), focus-not-obscured under sticky chrome
  (SC 2.4.11 / 2.4.12), whether meaning survives the loss of a single channel,
  and painted balance / squint-test focal point (the *one focal point* obligation
  in `principles/reference/visual_hierarchy.md` in `github.com/ctxmod/design/context` (profile `core`)).

These references mechanize the **convergent** set and tell you honestly where the
**divergent** set begins.

## Capability map (probed live; the corrected version)

> This map is load-bearing. It was established by probing both tools against
> three real apps (Next.js SSR, Vite CSR, Cloudflare Workers SSR). It **corrects**
> the original plan on one point: lightpanda's `getComputedStyle` does **not** run
> the CSS cascade — it returns UA defaults (`color` → `rgb(0,0,0)`) and empty
> strings for custom-property tokens, **regardless of how the CSS is delivered**
> (external `<link>`, inline `<style>`, or inline `style=`). Likewise lightpanda
> geometry (`offsetWidth`, `getBoundingClientRect`) returns ~0/empty placeholders
> — there is no layout engine. So style/token/cascade checks **and** geometry
> checks escalate to Playwright (or, for token *values* only, parse the raw
> `<style>` text). Source: the probe report.

> **Version drift (checked 2026-11-23).** The probes behind this map ran against
> lightpanda 0.3.x and the curated MCP subset. Since then: 0.4.0 made external
> stylesheets, iframes and workers opt-in (`--load-resources stylesheet` is
> required before raw linked-CSS text can be read); 1.0.0 split the style
> cascade into property groups, honours `!important` on tracked stylesheet
> properties, resolves `width`/`height` from stylesheet rules and makes
> `getComputedStyle` results read-only; and the full tool set (outside the
> curated subset) has a `consoleLogs` tool that drains captured console
> messages. So the **computed-style, custom-property and console rows below
> may have moved — re-probe them on the pinned release before relying on
> either verdict.** The no-paint, no-layout rows (geometry, target size,
> responsive matrix, screenshots, network list) still hold: lightpanda does
> not render pages.

| Check | lightpanda | Playwright | Notes |
|---|---|---|---|
| Semantic/a11y tree | YES `fetch --dump semantic_tree_text` | YES `browser_snapshot` | lightpanda omits heading level; Playwright includes `[level=N]` + `/url:` |
| Structured extraction | YES `page.extract` | via `browser_evaluate` | lightpanda is the token-cheap primary |
| Headings / landmarks / names | YES (tree) | YES (snapshot) | both |
| `lang` / viewport / title | YES `evaluate` | YES | both |
| `matchMedia(prefers-color-scheme)` | YES (`false` here) | YES | both |
| Click / fill / press → re-extract | YES (Counter 0→3; fill verified) | YES | both |
| CSR/hydration wait | YES `waitForSelector`/`waitForState`; `goto` returns at `load` | YES `browser_wait_for` | must wait for async content |
| **Computed/cascaded styles** | **NO** — UA default `rgb(0,0,0)` / empty for external CSS, inline `<style>`, AND inline `style=` | **YES** — `rgb(26,26,26)`, `39.056px`, real | **corrects plan**: read raw `<style>` text for token *values*, else escalate |
| Custom-property (`--token`) read | **NO** — `getPropertyValue` → `''` | **YES** — `#1a1a1a` | parse CSS text on lightpanda |
| Geometry (`offsetWidth`/rect) | **NO** — `5` / `{}` placeholders | **YES** — `960` / full rect | Playwright only |
| Target size (SC 2.5.5 / 2.5.8) | **NO** (no `minHeight` resolved) | **YES** — `44px` | Playwright only |
| `document.styleSheets` iteration | **NO** — "not iterable" | YES | use raw text on lightpanda |
| Responsive / breakpoints | **NO** | **YES** `browser_resize` (375px → h1 343px) | Playwright only |
| Screenshot / painted pixels | **NO** | **YES** `browser_take_screenshot` (png base64, ~23 KB) | Playwright only |
| Console messages | **NO** in the probed subset (a `consoleLogs` tool exists in the full set — re-probe) | **YES** `browser_console_messages` (favicon 404; clean hydration) | Playwright for the probed path |
| Network requests | **NO** | **YES** `browser_network_requests` (`static:true` for assets) | Playwright only |

## The default-first rule

**lightpanda first; escalate to Playwright only for what lightpanda physically
cannot do.** lightpanda is text-only, Chrome-free, and token-cheap, so it is the
default instrument for the bulk of convergent checks:

- **lightpanda (strong):** semantic/a11y tree (`semantic_tree_text`), structured
  extraction (`extract`), interaction (`click`/`fill`/`press`/`selectOption`),
  navigation + hydration wait (`waitForSelector`/`waitForState`), DOM
  text/attributes, `document.documentElement.lang`, viewport/title meta,
  `matchMedia('(prefers-color-scheme: dark)')` (declared), and recovering token
  *values* by parsing raw `<style>` text.
- **Escalate to Playwright** (the things with no text-only equivalent): painted
  pixels (`browser_take_screenshot`), real geometry and target-size
  (`browser_evaluate` reading `offsetWidth`/`getBoundingClientRect`), cascaded
  computed styles and resolved token values (`browser_evaluate` running
  `getComputedStyle`), the responsive/viewport matrix (`browser_resize`), runtime
  console signals (`browser_console_messages`), network/asset fetches
  (`browser_network_requests`), and the painted a11y snapshot
  (`browser_snapshot`, which also carries explicit heading `[level=N]`).

The single most important correction vs. the original plan: **computed styles and
token reads do NOT work on lightpanda.** Recover token *values* by parsing the raw
`<style>`/CSS text, or escalate to Playwright for the resolved/cascaded value.

## Canonical workflow

1. **Bring the dev server up.** Per app: `pnpm dev` (Next.js / Vite) or
   `wrangler dev` (Workers). Note the port.
2. **Navigate and read.** `lightpanda fetch --dump semantic_tree_text <url>` for
   the structure/a11y outline, or `markdown {url}` for a one-call navigate-and-read.
   This is the cheapest first look and the gold signal for structure. For
   client-rendered routes, wait for content — see
   <ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::browsing@automation/reference/rendering_models.md" /> (the
   `--wait-until load` vs `networkidle` probe, and `waitForSelector` in scripts).
3. **Run convergent checks.**
   - **Structure / a11y** via lightpanda's semantic tree (heading outline,
     landmark roles, accessible names, form shape) — see
     <ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::web_validation@automation/reference/accessibility_structure.md" />.
   - **Styles / tokens / contrast / type-scale / target-size** via Playwright
     `browser_evaluate` running the real cascade (or parse the raw `<style>` for
     token *presence*/declared hex on lightpanda) — see
     <ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::web_validation@automation/reference/visual_system_styles.md" />.
4. **Functional interaction** via lightpanda: drive state with
   `click`/`fill`/`press` and observe it by re-extracting — see
   <ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::web_validation@automation/reference/functional_testing.md" />.
5. **Consistency / localization**: same-role-same-name patterns across routes,
   `lang`/`dir` per locale, logical-vs-visual DOM order — see
   <ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::web_validation@automation/reference/consistency_localization.md" />.
6. **Escalate divergent checks to manual** (next section), and report every
   finding against the named principle and success criterion.

## What stays manual (divergent)

No tool fully decides these — budget keyboard-only and screen-reader walkthroughs,
plus a squint-test on a real screenshot:

- **ARIA correctness & announced state** — a tree dump shows a role exists, not
  whether it is the *right* role or whether the announced state matches reality.
- **Focus-not-obscured** (SC 2.4.11 / 2.4.12) — requires watching focus move under
  sticky headers/footers at real layout; lightpanda has no layout, and a tool
  can't judge "obscured."
- **Pointer-gesture / dragging alternatives** (SC 2.5.1 / 2.5.7) — whether a
  single-pointer path exists *and is a first-class affordance* is a design call.
- **Painted balance & one-focal-point** (the squint test in
  `principles/reference/visual_hierarchy.md` in `github.com/ctxmod/design/context` (profile `core`)) — needs the
  painted pixels (`browser_take_screenshot`), then human judgment of dominance.
- **Meaning surviving channel loss** — color-not-alone, motion-not-alone: a tool
  flags color usage, not whether the redundant channel actually conveys the
  meaning.

## Proven snippets

Every output below is pasted verbatim from the probe report;
nothing here is unverified.

### lightpanda — semantic tree (structure / a11y, the default first look)

```
$ LIGHTPANDA_DISABLE_TELEMETRY=true lightpanda fetch --dump semantic_tree_text http://localhost:3137/
1 RootWebArea 'Next.js Example'
 5 heading 'Home'
 6 navigation 'Primary'
  7 [i] link 'Home'
  8 [i] link 'About'
  9 [i] link 'Dashboard'
 10 paragraph
  11 'Minimal Next.js test-bed for frontend-validation references.'
 12 heading 'Counter'
 14 paragraph
  15 'Count:'
  16 '0'
 17 [i] button 'Increment'
 18 heading 'Newsletter'
 19 form
  21 'Email address'
  22 [i] textbox 'Email address'
  23 [i] button 'Subscribe'
```

This one dump yields the heading outline, the `navigation 'Primary'` landmark
(from `<nav aria-label="Primary">`), accessible names for every link/button/
textbox, and the form structure. `[i]` marks an interactive node. Note lightpanda
does **not** emit heading levels — infer order from nesting; escalate to
`browser_snapshot` when explicit `[level=N]` assertions matter.

### Playwright — real geometry + resolved styles (the lightpanda contrast)

The **identical** `getComputedStyle` script returns UA defaults / empty on
lightpanda but the real cascade on Playwright. On lightpanda:

```
H1_COLOR rgb(0, 0, 0)
H1_FONTSIZE
TOKEN_COLOR_FG ''
BOUNDARY_OFFSETWIDTH 5
BOUNDARY_RECT {}
```

`browser_evaluate` with
`() => JSON.stringify({ h1_offsetWidth: document.querySelector('h1').offsetWidth, button_rect: JSON.parse(JSON.stringify(document.querySelector('button').getBoundingClientRect())) })`:

```
"{"h1_offsetWidth":960,"button_rect":{"x":160,"y":296.09375,"width":109.953125,"height":44,"top":296.09375,"right":269.953125,"bottom":340.09375,"left":160}}"
```

`browser_evaluate` running the same `getComputedStyle` reads that lightpanda
returned empty for:

```
"{"h1_color":"rgb(26, 26, 26)","h1_fontSize":"39.056px","h1_fontFamily":"system-ui, \"Segoe UI\", Roboto, sans-serif","token_fg":"#1a1a1a","button_minHeight":"44px"}"
```

Playwright resolves the real cascade: `h1_color = rgb(26,26,26)` (token
`--color-fg #1a1a1a`), `h1_fontSize = 39.056px` (`--step-4` 2.441rem × 16),
`token_fg = #1a1a1a`, `button_minHeight = 44px` (the real SC 2.5.5 floor).
**Therefore WCAG contrast math, type-scale verification, and target-size checks
belong on Playwright, not lightpanda.**

### lightpanda — token values via raw `<style>` text (the workaround)

When CSS is delivered inline, token *values* are recoverable by reading the raw
stylesheet text — `getComputedStyle` still fails, but the text is intact:

```
INLINE_STYLE_TEST color=rgb(0, 0, 0) width= elstyle=rgb(1,2,3)
STYLE_TAG_PRESENT len=1829 head= :root {   --color-bg: #ffffff;   --color-fg: #1a1a1a;   --c
```

`getComputedStyle().color` is still `rgb(0,0,0)` (and `.width` empty) even for a
synthetic element whose inline `el.style.color` reads back `rgb(1,2,3)`. But
`document.querySelector('style').textContent` returns the full 1829-char token
block with the declared hex values visible. Use this for token *presence* and
*declared* values; escalate to Playwright for *resolved* values.

### lightpanda — interaction drives and observes state

```
COUNT_BEFORE {"count":"Count: 0"}
COUNT_AFTER {"count":"Count: 3"}
```

Three `page.click({ selector: "button[type='button']" })` calls drive React state;
re-extraction observes the change. Functional testing works on lightpanda without
the cascade — see <ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::web_validation@automation/reference/functional_testing.md" />.

## Related

- <ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::browsing@automation/reference/lightpanda.md" />
- `principles/reference/accessibility_as_quality.md` in `github.com/ctxmod/design/context` (profile `core`)
- `principles/reference/visual_hierarchy.md` in `github.com/ctxmod/design/context` (profile `core`)

## Sources

- https://github.com/lightpanda-io/browser/releases/tag/0.4.0 — external stylesheets, iframes and workers opt-in via `--load-resources`; accessed 2026-11-23
- https://github.com/lightpanda-io/browser/releases/tag/1.0.0 — style cascade split into property groups, `!important` on tracked properties, width/height from stylesheet rules, read-only `getComputedStyle`; accessed 2026-11-23
- https://raw.githubusercontent.com/lightpanda-io/browser/1.0.5/src/browser/tools.zig — 1.0.5 browser tool enum (includes `consoleLogs`, `screenshot`); accessed 2026-11-23
