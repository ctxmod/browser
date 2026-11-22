# Visual System & Styles

Machine-checkable design-system checks on a live frontend — token presence (lightpanda raw <style>) vs token values + contrast-ratio math + type-scale + target-size (Playwright cascade), dark-mode declaration, responsive viewport/breakpoints, font/weight consistency — and the painted checks that stay manual.

This reference operationalizes the **design-system / token** obligations in the
color, typography, spacing-scale, dark-mode, and responsive-layout principles:
the subset that is mechanically checkable on a live rendered page **without
painted pixels**. It is the operational companion to those normative files; it
cites them and shows how to run their "How to verify" sections, it does not
restate them. Start at
<ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::web_validation@automation/reference/validation_overview.md" /> for the tool split and
the canonical workflow.

> **The load-bearing correction (read this first).** lightpanda's
> `getComputedStyle` does **not** run the CSS cascade. It returns UA defaults
> (`color` → `rgb(0,0,0)`) and empty strings for custom-property tokens
> (`getPropertyValue('--color-fg')` → `''`) — for **every** delivery method
> (external `<link>`, inline `<style>`, inline `style=`). It has no layout engine,
> so geometry (`offsetWidth`/`getBoundingClientRect`) returns `5`/`{}`
> placeholders. Therefore every check in this file that needs a **resolved/
> cascaded value** (contrast math, type-scale px, target size) runs on
> **Playwright** `browser_evaluate`. lightpanda can still answer **"does a token
> system exist?"** by parsing the raw `<style>` text — *presence*, not resolved
> values. Source: the probe report.

## What is machine-checkable vs. manual

| Concern | Checkable | How | Manual |
|---|---|---|---|
| Token system exists | YES (presence) | lightpanda: parse raw `<style>` text for `--token` declarations | — |
| Token resolved value | YES (values) | Playwright: `getComputedStyle(documentElement).getPropertyValue('--token')` | — |
| Contrast ratio | YES | Playwright: read cascaded fg/bg → WCAG math in-agent | painted-on-image contrast |
| Type scale | YES | Playwright: `getComputedStyle(hN).fontSize` per step | — |
| Body size floor | YES | Playwright: body `fontSize ≥ 16px` | — |
| Spacing rhythm | YES (declared) | lightpanda CSS text / Playwright `--space-*` values | painted vertical rhythm |
| Dark-mode declaration | YES | parse CSS for `@media (prefers-color-scheme: dark)`; `matchMedia` callable; `theme-color` meta | whether the dark palette is *correct* |
| Responsive viewport meta | YES | read `meta[name=viewport]` content | — |
| Breakpoint reflow | YES | Playwright `browser_resize` + re-read | painted balance per width |
| Font / weight consistency | YES | Playwright: `fontFamily` / `fontWeight` cascade | — |
| Painted balance / focal point | **NO** | — | the **squint test** in `principles/reference/visual_hierarchy.md` in `github.com/ctxmod/design/context` (profile `core`) — needs a screenshot + human judgment |
| Layout rhythm / whitespace as content | **NO** | — | manual; no layout engine on lightpanda, no judgment in any tool |

The dividing line is the **cascade and the paint**. Declared *structure* (a token
exists, a media query exists, a viewport meta exists) is text and is checkable
cheaply on lightpanda. Resolved *values* (the actual px, the actual rgb) need
Playwright's real engine. **Painted aesthetics** — does the page have one focal
point, does the whitespace breathe, is the composition balanced — are not decided
by any tool; they need a real screenshot (`browser_take_screenshot`) and a human
squint, per `principles/reference/visual_hierarchy.md` in `github.com/ctxmod/design/context` (profile `core`).

The three test-bed apps (the probe test-bed) share one token vocabulary —
`--color-bg/-fg/-muted/-accent/-border`, `--step-0..4` (a 1.250 major-third type
scale), `--space-1..5`, `--font-body` — delivered three ways (nextjs external
`<link>`, vite imported CSS module, workers-ssr inline `<style>`). The
delivery method does **not** change the lightpanda cascade boundary; it only
changes whether the raw-`<style>` presence path is available in-document.

## 1. Color tokens & contrast (cascade + in-agent WCAG math)

`principles/reference/color.md` in `github.com/ctxmod/design/context` (profile `core`) mandates WCAG ratios
(SC 1.4.3 body 4.5:1, SC 1.4.6 AAA, SC 1.4.11 UI components 3:1) **against the
actual background**, and warns that color is *relative* — verify it in situ, in
the real layout. The recipe:

1. On **Playwright** `browser_evaluate`, read the element's cascaded `color` and
   walk its ancestor chain for the first non-transparent `backgroundColor`
   (a transparent element inherits its painted bg from an ancestor — that is why
   the chain walk matters, not just `getComputedStyle(el).backgroundColor`).
2. Parse both to RGB triples.
3. Compute relative luminance and the WCAG contrast ratio **in the agent** (the
   math, below), and compare to the threshold for the element's text size
   (4.5:1 body, 3:1 large ≥ 18px / ≥ 14px bold).

The extraction snippet (returns `{fg, bg}` plus the painted size so the agent
picks the right threshold):

```js
() => {
  const el = document.querySelector('h1');
  const cs = getComputedStyle(el);
  let bg = 'rgba(0, 0, 0, 0)', n = el;
  while (n) {                                  // walk up for the first opaque bg
    const c = getComputedStyle(n).backgroundColor;
    if (c && c !== 'rgba(0, 0, 0, 0)' && c !== 'transparent') { bg = c; break; }
    n = n.parentElement;
  }
  return JSON.stringify({ fg: cs.color, bg, fontSize: cs.fontSize, fontWeight: cs.fontWeight });
}
```

Real probe reads (the **identical** `getComputedStyle` script returned
`H1_COLOR rgb(0, 0, 0)` and empty on lightpanda — see §"Token presence" — but the
real cascade on Playwright):

```
h1_color   = rgb(26, 26, 26)     // = token --color-fg #1a1a1a
h1_fontSize = 39.056px           // large text → 3:1 threshold
page background = rgb(255,255,255)  // --color-bg #ffffff
```

The agent then does the ratio math (WCAG relative luminance, sRGB):

```js
// sRGB channel → linear
const lin = c => { c /= 255; return c <= 0.04045 ? c/12.92 : ((c+0.055)/1.055)**2.4; };
const L = ([r,g,b]) => 0.2126*lin(r) + 0.7152*lin(g) + 0.0722*lin(b);
const ratio = (fg, bg) => { const a = L(fg)+0.05, b = L(bg)+0.05; return (Math.max(a,b)/Math.min(a,b)); };
// fg rgb(26,26,26), bg rgb(255,255,255):
ratio([26,26,26], [255,255,255]);   // -> 17.4
```

`rgb(26,26,26)` on `rgb(255,255,255)` computes to **≈ 17.4 : 1** — clears the
3:1 large-text floor and the 4.5:1 body floor with room to spare. Flag any pair
below its threshold, and record the named SC (1.4.3 / 1.4.11).

> **Caveat (manual escalation).** This validates the **declared token contrast** —
> solid fg on solid bg. It does **not** validate text painted over an image or a
> gradient (`color.md`: "contrast is against the actual background, including
> overlays and images. Text on a photo needs a scrim."). Painted-on-image contrast
> needs `browser_take_screenshot` + human judgment — manual.

## 2. Type scale (computed font-size steps)

`principles/reference/typography.md` in `github.com/ctxmod/design/context` (profile `core`) requires the scale to
be a **modular ratio** (the apps use 1.250 major third) with perceptible jumps,
and a hard **body floor ≥ 16px (1rem)**. Read `fontSize` on `h1..h4` and body via
Playwright (lightpanda returns `H1_FONTSIZE` *empty*), then assert the steps
follow the declared ratio and body clears the floor.

Real probe read: `h1_fontSize = 39.056px`. That is `--step-4` = `2.441rem × 16`
(`2.441 = 1.25⁴`), the top of the major-third ramp. With a 16px (`--step-0`,
1rem) body the steps are `16 → 20 → 25 → 31.25 → 39.06`, each `≈ ×1.25`. Assert:

- Body running text `fontSize ≥ 16px` (the floor; flag `< 16px`, captions `< ~13px`).
- Adjacent heading steps differ by the declared ratio (≈ 1.25) within tolerance;
  a `< ~1.15` jump reads "as a mistake" per the principle (no perceptible
  hierarchy).
- Steps come from the token set, not arbitrary px (cross-check the `--step-*`
  values via the token read in §6 / §"Token values").

```js
() => JSON.stringify(['h1','h2','h3','h4','p'].map(s => {
  const el = document.querySelector(s);
  return el ? { tag: s, px: getComputedStyle(el).fontSize } : { tag: s, px: null };
}))
// h1 → "39.056px" (proven); ratios between successive present steps ≈ 1.25
```

## 3. Spacing rhythm (declared scale, not arbitrary)

`principles/reference/spacing_scale.md` in `github.com/ctxmod/design/context` (profile `core`): every gap should
be a **step on one scale** derived from a base unit; a `13px` next to a `16px` is
a defect. This is a **declared-token** check — assert the `--space-1..5` tokens
exist and form a consistent (geometric-ish) scale, not that the painted vertical
rhythm is correct (that needs layout → manual).

- **lightpanda (presence):** parse the raw `<style>` text for `--space-` declarations
  (see §"Token presence" — the `:root{…}` block is readable as text even though
  the cascade is not).
- **Playwright (values):** `getComputedStyle(documentElement).getPropertyValue('--space-3')`
  resolves each token; assert they grow on one base (e.g. 4px) and don't include
  off-scale magic numbers.

Painted vertical rhythm — equal-space-above-and-below-a-heading bugs, whether the
whitespace actually does the grouping work — is **manual**; the principle's
proximity-law checks need a real layout.

## 4. Dark-mode declaration presence

`principles/reference/dark_mode.md` in `github.com/ctxmod/design/context` (profile `core`): the pattern is
**mandated when offered** (the feature itself is optional). So the machine check
is "**is dark mode declared, and declared via the right mechanism?**" — not
"is the dark palette correct" (that is per-mode contrast, run §1 twice; and
"no hardcoded dark hex in components" is a source-grep, not a runtime check).

Checkable signals:

- A `@media (prefers-color-scheme: dark)` block exists — **parse the raw CSS text**
  (lightpanda can read `<style>` text; Playwright too). Presence of the at-rule is
  the "offered" signal.
- `matchMedia('(prefers-color-scheme: dark)')` is callable and returns a boolean
  (works on both tools). Real probe: `MATCHMEDIA_DARK false` — the harness's OS
  preference was light; the call *succeeding* is the capability signal, the
  boolean reflects the environment, not a defect.
- A `theme-color` meta is present (the apps ship dual light/dark `theme-color`),
  signaling the product surfaces a mode to the UA.

```js
matchMedia('(prefers-color-scheme: dark)').matches      // probe -> false (callable; env is light)
```

If `@media (prefers-color-scheme: dark)` is present, escalate to running the §1
contrast recipe **against the dark background** — `dark-mode.md` is explicit that
light-mode ratios do not transfer and each mode is verified independently.

## 5. Responsive viewport meta + breakpoints

`platform/reference/responsive_layout.md` in `github.com/ctxmod/design/context` (profile `web`): layouts
must **reflow** across viewports, breakpoints are a **named scale** placed where
content breaks, not device chasing.

- **Viewport meta (presence, both tools):** read `meta[name=viewport]` content.
  Real probe: `VIEWPORT width=device-width, initial-scale=1` on all three apps.
  Flag a missing meta or a `user-scalable=no` / `maximum-scale=1` that blocks
  zoom (an accessibility regression).
- **Breakpoint reflow (Playwright only):** lightpanda has no layout, so the
  breakpoint matrix is Playwright `browser_resize` across widths, re-reading
  layout/`fontSize` at each. Real probe: `browser_resize {width:375,height:667}`
  then re-evaluating `{ innerWidth, h1.offsetWidth }` returned:

```
{"innerWidth":375,"h1w":343}
```

The viewport genuinely changed to 375px and the `h1` reflowed from **960px**
(default desktop width, §"Geometry" of the probe report) to **343px** — a real layout re-flow,
impossible on lightpanda. Walk the named breakpoint scale (`sm`/`md`/`lg`/…),
re-read at each, and assert the content reflows (column count, nav bar→drawer)
rather than merely shrinking. Whether each width is *balanced* is the manual
squint test.

## 6. Font / weight consistency

`principles/reference/typography.md` in `github.com/ctxmod/design/context` (profile `core`): identity lives in
large/short text, readability in long/small; **weight is for hierarchy** (reserve
bold), and a single ad-hoc family per call site is a smell. Read the cascade via
Playwright (`fontFamily` is *empty* on lightpanda):

- **One body family.** Real probe: `h1_fontFamily = system-ui, "Segoe UI", Roboto,
  sans-serif` — the `--font-body` stack. Assert the body family is consistent
  across roles; flag multiple unrelated families introduced ad hoc.
- **Weights from the declared scale.** Read `fontWeight` per role; assert weights
  come from the system's ladder (e.g. 400/600/700) and that bold is used for
  hierarchy, not ambient emphasis. Adjacent weights (400 vs 500) barely differ —
  the principle prefers the extremes for drama.

```js
() => JSON.stringify({
  body: getComputedStyle(document.body).fontFamily,
  h1w:  getComputedStyle(document.querySelector('h1')).fontWeight
})
// body family resolves to the --font-body stack on Playwright; empty on lightpanda
```

## Token presence (lightpanda) vs token values (Playwright)

The split that recurs in every check above, shown concretely. **The same token
read returns nothing useful on lightpanda and the real value on Playwright.**

**lightpanda — presence only, via raw `<style>` text.** `getComputedStyle` and
`getPropertyValue` both fail, but the stylesheet text is intact:

```
TOKEN_COLOR_FG ''                                  // getPropertyValue('--color-fg') -> empty
INLINE_STYLE_TEST color=rgb(0, 0, 0) width= elstyle=rgb(1,2,3)
STYLE_TAG_PRESENT len=1829 head= :root {   --color-bg: #ffffff;   --color-fg: #1a1a1a;   --c
```

`document.querySelector('style').textContent` returns the full 1829-char
`:root{…}` block with the declared hex visible — so lightpanda answers **"does a
token system exist, and what are its *declared* values?"** by parsing this text.
It cannot answer the *resolved* (cascaded/inherited/computed) value: even a
synthetic element whose inline `el.style.color` reads back `rgb(1,2,3)` still
yields `getComputedStyle().color === rgb(0,0,0)`.

**Playwright — resolved values, via the real cascade.** The same
`getPropertyValue` / `getComputedStyle` calls resolve:

```
token_fg          = #1a1a1a       // getComputedStyle(documentElement).getPropertyValue('--color-fg')
h1_color          = rgb(26, 26, 26)
h1_fontSize       = 39.056px
button_minHeight  = 44px          // SC 2.5.5 target-size floor, resolved
```

**Rule:** use lightpanda for token *presence* / *declared* values (cheap, text);
use Playwright for *resolved* values, contrast math, type-scale px, and target
size. Never trust a lightpanda `getComputedStyle`/`getPropertyValue` result.

## Mapping: check → principle → tool

| Check | Principle ("How to verify") | lightpanda (presence) | Playwright (values) |
|---|---|---|---|
| Token system exists | color.md / design-tokens | YES — raw `<style>` text | YES — `getPropertyValue` |
| Contrast ratio (SC 1.4.3/1.4.11) | `principles/reference/color.md` in `github.com/ctxmod/design/context` (profile `core`) | NO (no cascade) | YES — fg/bg + in-agent math |
| Type scale + body floor | `principles/reference/typography.md` in `github.com/ctxmod/design/context` (profile `core`) | NO (`fontSize` empty) | YES — `fontSize` per step |
| Spacing scale declared | `principles/reference/spacing_scale.md` in `github.com/ctxmod/design/context` (profile `core`) | YES — `--space-*` in CSS text | YES — resolved `--space-*` |
| Dark-mode declared | `principles/reference/dark_mode.md` in `github.com/ctxmod/design/context` (profile `core`) | YES — `@media` text + `matchMedia` | YES — same + per-mode contrast |
| Viewport meta | `platform/reference/responsive_layout.md` in `github.com/ctxmod/design/context` (profile `web`) | YES — `meta[name=viewport]` | YES |
| Breakpoint reflow | `platform/reference/responsive_layout.md` in `github.com/ctxmod/design/context` (profile `web`) | NO (no layout) | YES — `browser_resize` |
| Font / weight consistency | `principles/reference/typography.md` in `github.com/ctxmod/design/context` (profile `core`) | NO (`fontFamily` empty) | YES — `fontFamily`/`fontWeight` |
| Painted balance / focal point | `principles/reference/visual_hierarchy.md` in `github.com/ctxmod/design/context` (profile `core`) | NO | NO — screenshot + human (manual) |

## Proven snippets (real probe output)

Every output below is pasted verbatim from the probe report.
Each is marked **presence-on-lightpanda** or **values-on-Playwright**.

### A. Token value read — `--color-fg` (values-on-Playwright)

lightpanda returns empty; Playwright resolves it:

```
lightpanda:  TOKEN_COLOR_FG ''
Playwright:  token_fg = #1a1a1a
```

`getComputedStyle(document.documentElement).getPropertyValue('--color-fg')` — the
identical call, empty string on lightpanda, real hex on Playwright.

### B. Computed color + font-size (values-on-Playwright)

```
lightpanda:  H1_COLOR rgb(0, 0, 0)   H1_FONTSIZE (empty)
Playwright:  {"h1_color":"rgb(26, 26, 26)","h1_fontSize":"39.056px","button_minHeight":"44px"}
```

`rgb(0,0,0)` on lightpanda is the **UA default**, not the token; Playwright
resolves the real cascade (`rgb(26,26,26)` = `--color-fg #1a1a1a`; `39.056px` =
`--step-4`; `44px` = the SC 2.5.5 floor).

### C. Contrast `{fg, bg}` extraction → in-agent ratio (values-on-Playwright)

The §1 ancestor-walk snippet returns `{fg, bg}`; the agent computes the ratio.
With the real probe reads `fg = rgb(26,26,26)`, `bg = rgb(255,255,255)`:

```
ratio([26,26,26],[255,255,255]) -> 17.4   // ≥ 4.5:1 body and ≥ 3:1 large — pass
```

The extraction runs on Playwright (cascade); the WCAG math runs in the agent.
Caveat: validates declared token contrast only — painted-on-image contrast is
manual.

### D. Dark-mode `matchMedia` (presence; callable on both tools)

```
MATCHMEDIA_DARK false
```

`matchMedia('(prefers-color-scheme: dark)').matches` — the call *succeeds* on both
tools (the capability signal); `false` reflects the light-set environment, not a
defect. Pair with parsing the CSS text for a `@media (prefers-color-scheme: dark)`
block to confirm the mode is *offered*.

### E. Token presence via raw `<style>` text (presence-on-lightpanda)

```
STYLE_TAG_PRESENT len=1829 head= :root {   --color-bg: #ffffff;   --color-fg: #1a1a1a;   --c
```

`document.querySelector('style').textContent` — the full token block is readable
as text even though the cascade is not. This is lightpanda's path for "does a
token system exist, and what are its *declared* values?".

### F. Responsive reflow (values-on-Playwright)

```
{"innerWidth":375,"h1w":343}
```

`browser_resize {width:375}` then re-read: the `h1` reflowed from 960px (desktop)
to 343px — a real layout change, impossible on lightpanda.

## Related

- `principles/reference/color.md` in `github.com/ctxmod/design/context` (profile `core`)
- `principles/reference/typography.md` in `github.com/ctxmod/design/context` (profile `core`)
- `principles/reference/spacing_scale.md` in `github.com/ctxmod/design/context` (profile `core`)
- `principles/reference/dark_mode.md` in `github.com/ctxmod/design/context` (profile `core`)
- `platform/reference/responsive_layout.md` in `github.com/ctxmod/design/context` (profile `web`)
