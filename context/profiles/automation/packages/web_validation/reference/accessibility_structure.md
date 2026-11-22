# Accessibility Structure Checks

Mechanically run the convergent a11y checks the design principles name — heading outline, landmark roles, accessible names, lang/dir/title/viewport meta, target-size — using lightpanda's semantic tree as the primary instrument and escalating to Playwright for painted geometry.

This reference mechanizes the **convergent** accessibility checks named in
`principles/reference/accessibility_as_quality.md` in `github.com/ctxmod/design/context` (profile `core`) and
`platform/reference/web_accessibility.md` in `github.com/ctxmod/design/context` (profile `web`): heading
outline, landmark roles, accessible names, `lang`/`dir`/title/viewport meta, and
target-size *declarations*. The principle files are the *normative* source; this
is the *operational* companion that runs their "How to verify" items against a
live frontend. Read <ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::web_validation@automation/reference/validation_overview.md" />
first for the lightpanda-vs-Playwright capability map and the default-first rule.

## Why structure-first

`principles/reference/accessibility_as_quality.md` in `github.com/ctxmod/design/context` (profile `core`) splits
verification into **convergent** checks (deterministic pass/fail — automatable in
CI) and **divergent** checks (need a human — ARIA correctness, focus-not-obscured,
meaning-survives-channel-loss). Automated tools detect only **~25–30% of WCAG
criteria**; the convergent set *is* that detectable slice. This reference covers
the structural part of it: heading order, landmark roles, and accessible names —
which the principle lists explicitly as "screen-reader semantics" — plus `lang`,
`dir`, title/meta, and the target-size *declaration*. The divergent remainder
stays manual (see <ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::web_validation@automation/reference/validation_overview.md" /> →
"What stays manual").

Structure is checked first because it is the cheapest, highest-signal pass: one
semantic-tree dump simultaneously yields the heading outline, the landmark
regions, the accessible names, and the form shape — exactly what

- `principles/reference/accessibility_as_quality.md` in `github.com/ctxmod/design/context` (profile `core`)

("Structure as a design lever") and

- `platform/reference/web_accessibility.md` in `github.com/ctxmod/design/context` (profile `web`)

("Semantic HTML is the foundation") tell you to verify.

## The semantic tree as primary instrument

`lightpanda fetch --dump semantic_tree_text <url>` is the gold first look: it
emits the accessibility tree as roles + accessible names (heading, navigation,
link, button, textbox, form, paragraph). One dump answers four checks at once.
Verbatim from the probe, the Next.js home route:

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

Read it as: three `heading` nodes in DOM order, a `navigation 'Primary'` landmark
(from `<nav aria-label="Primary">`), accessible names on every `link`/`button`/
`textbox`, and a `form` containing a label text + `textbox` + `button`. The `[i]`
marker flags an interactive node.

**Boundary — heading level.** lightpanda's tree emits the `heading` *role* but
**not** the level (`h1` vs `h2`). It shows order, not depth. When a check needs
explicit levels (e.g. "is this an `h1` or an `h3`?"), escalate to Playwright's
`browser_snapshot`, which carries `[level=N]` and `/url:` per node. The same
nextjs home route under Playwright:

```yaml
- generic [active] [ref=e1]:
  - main [ref=e2]:
    - heading "Home" [level=1] [ref=e3]
    - navigation "Primary" [ref=e4]:
      - link "Home" [ref=e5] [cursor=pointer]:
        - /url: /
      - link "About" [ref=e6] [cursor=pointer]:
        - /url: /about
      - link "Dashboard" [ref=e7] [cursor=pointer]:
        - /url: /dashboard
    - paragraph [ref=e8]: Minimal Next.js test-bed for frontend-validation references.
    - heading "Counter" [level=2] [ref=e9]
    - generic [ref=e10]:
      - paragraph [ref=e11]: "Count: 0"
      - button "Increment" [ref=e12] [cursor=pointer]
    - heading "Newsletter" [level=2] [ref=e13]
    - generic [ref=e14]:
      - text: Email address
      - textbox "Email address" [ref=e15]
```

Playwright's snapshot also surfaces the `main` landmark (`main [ref=e2]`) that
lightpanda's tree omits, and the link `/url:` targets. **Default-first rule:**
use lightpanda's tree for order/roles/names; escalate to `browser_snapshot` only
when you need heading levels or in-tree link URLs.

## Heading outline checks

The rule, straight from
`platform/reference/web_accessibility.md` in `github.com/ctxmod/design/context` (profile `web`): **one
`<h1>`, nested `<h2>/<h3>` in order, no levels skipped for visual size.** Levels
must be contiguous and start at 1 (an `h1`→`h3` jump is a violation). Restyle with
CSS — never renumber the semantics to get a font size.

Extract the headings cheaply on lightpanda with `page.extract` (the `h1` plus the
`h2` list). Verbatim probe output:

```js
const page = new Page();
await page.goto("http://localhost:3137/");
console.log("EXTRACT " + JSON.stringify(page.extract({
  title: "title",
  h1: "h1",
  headings: ["h2"],
})));
```

```
EXTRACT {"title":"Next.js Example","h1":"Home","headings":["Counter","Newsletter"]}
```

This proves the outline is **`h1` "Home"** then two **`h2`s** ("Counter",
"Newsletter") — a single `h1`, contiguous `1 → 2`, no skipped level: a pass. The
Playwright `browser_snapshot` above corroborates with explicit `[level=1]` on
"Home" and `[level=2]` on both siblings. Because lightpanda's tree omits levels,
the `extract` (`h1` vs `h2` selectors) is how you assert the outline *without*
Playwright; escalate to the snapshot only when the level numbers themselves are
the disputed point.

## Landmark roles

- `principles/reference/accessibility_as_quality.md` in `github.com/ctxmod/design/context` (profile `core`)

("Landmark regions") and
`platform/reference/web_accessibility.md` in `github.com/ctxmod/design/context` (profile `web`) call for
`header`/`nav`/`main`/`footer` (and `form`) present, and unique where expected.
The semantic tree names them directly. From the nextjs dump above:

```
 6 navigation 'Primary'
 19 form
```

`navigation 'Primary'` is the `<nav aria-label="Primary">` landmark; `form` is the
newsletter region. lightpanda does not surface an implicit `main` here — the
Playwright snapshot does (`main [ref=e2]`), so when you need to confirm a `main`
landmark exists (or that a landmark is unique), cross-check with
`browser_snapshot`. The check is: the expected landmark roles appear, named where
ambiguous (a second `nav` would need its own `aria-label` to be distinguishable).

## Accessible names

Every interactive element must have an accessible name —

- `principles/reference/accessibility_as_quality.md` in `github.com/ctxmod/design/context` (profile `core`)

("Labels and names" / convergent "accessible names") and

- `platform/reference/web_accessibility.md` in `github.com/ctxmod/design/context` (profile `web`)

(`<label>`-associated controls, `aria-label` for icon-only buttons). The name is
the quoted string after each role in the tree. From the nextjs dump:

```
  7 [i] link 'Home'
  8 [i] link 'About'
  9 [i] link 'Dashboard'
 17 [i] button 'Increment'
 22 [i] textbox 'Email address'
 23 [i] button 'Subscribe'
```

Every `link`/`button`/`textbox` carries a non-empty name — a pass. **Flag any
empty name** (`button ''`, `link ''`): a nameless control is invisible to screen
readers. Sources of the name: button/link text, `aria-label`, a `<label>` for
inputs, `alt` for images.

**Form inputs must have a `<label>`.** The email input renders as
`textbox 'Email address'` — the accessible name `'Email address'` comes from its
associated `<label>` (the `21 'Email address'` text node sits inside the `form`,
and Playwright's snapshot shows the matching `text: Email address` +
`textbox "Email address"` pairing). The probe confirmed this label-bearing email input
on **all three** test-bed forms — nextjs, vite-csr, and workers-ssr — each emitting
`textbox 'Email address'`. Verify the input is genuinely labelled with `extract`:

```js
console.log("FORM " + JSON.stringify(page.extract({
  formFields: [{ selector: "form input", fields: { type: { attr: "type" }, name: { attr: "name" }, required: { attr: "required" } } }]
})));
```

```
FORM {"formFields":[{"type":"email","name":"email","required":""}]}
```

The tree gives the accessible *name* (proves the label is wired); `extract` gives
the input's `type`/`name`/`required` attributes. A present-but-empty boolean
attribute (`required`) extracts as `""`. Together they confirm the field is a
labelled, required email input.

## `lang` / `dir` / title / viewport meta

`platform/reference/web_accessibility.md` in `github.com/ctxmod/design/context` (profile `web`) requires a
language on the document and a responsive viewport; a meaningful `<title>` is
basic page identity. These read cleanly on lightpanda via `evaluate` (the cascade
isn't involved). Verbatim probe output:

```js
page.evaluate("document.documentElement.lang")                          // LANG
page.evaluate("document.querySelector('meta[name=viewport]').content")  // VIEWPORT
page.evaluate("document.title")                                         // TITLE
```

```
LANG en
VIEWPORT width=device-width, initial-scale=1
TITLE Next.js Example
```

Checks: `document.documentElement.lang` is **non-empty** (`en` — pass); the
viewport meta is **present** with `width=device-width` (responsive — pass); the
`<title>` is **present and descriptive** (pass). The same `evaluate` pattern reads
`document.documentElement.dir` — required to be set (`rtl`/`auto`) for
right-to-left locales. The probe captured LTR apps (`dir` unset), and the RTL obligation
("rethink, don't mirror"; logical-vs-visual DOM order) is depth that belongs to
the localization reference rather than here. For per-locale `lang`/`dir`
correctness see <ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::web_validation@automation/reference/consistency_localization.md" />.

## Target size (SC 2.5.5 / 2.5.8)

`principles/reference/accessibility_as_quality.md` in `github.com/ctxmod/design/context` (profile `core`) names two
distinct conformance levels — **SC 2.5.8 Target Size (Minimum)** 24×24px (AA
floor) and **SC 2.5.5 Target Size (Enhanced)** 44×44px (AAA bar). This check is
**the** canonical lightpanda/Playwright split:

- The *declared* `min-height: 44px` is recoverable as **text** — read the raw
  `<style>`/CSS rather than computed style (lightpanda runs no cascade;
  `getComputedStyle(button).minHeight` returns empty — see
  <ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::web_validation@automation/reference/validation_overview.md" /> capability map).
- The *painted* size — does the button actually render ≥44px after layout? — needs
  geometry, which lightpanda cannot produce (`getBoundingClientRect()` serializes
  to `{}`). **Escalate to Playwright** `browser_evaluate`.

lightpanda's geometry is a placeholder, proving it cannot answer this:

```
BOUNDARY_RECT {}
BUTTON_MINHEIGHT
```

Playwright resolves both the painted rect and the cascaded `min-height`. The probe,
`browser_evaluate` reading `getBoundingClientRect()`:

```
"{"h1_offsetWidth":960,"button_rect":{"x":160,"y":296.09375,"width":109.953125,"height":44,"top":296.09375,"right":269.953125,"bottom":340.09375,"left":160}}"
```

and the identical `getComputedStyle` script lightpanda returned empty for:

```
"{"h1_color":"rgb(26, 26, 26)","h1_fontSize":"39.056px","h1_fontFamily":"system-ui, \"Segoe UI\", Roboto, sans-serif","token_fg":"#1a1a1a","button_minHeight":"44px"}"
```

The painted button is **109.95 × 44px** with a resolved `min-height: 44px` — it
hits the 44px AAA bar (and clears the 24px AA floor). Only Playwright can confirm
the *painted* target; lightpanda can at best confirm the *declaration* is present
in the CSS text.

## Mapping table

Each check → the principle "How to verify" item it satisfies → the tool. Levels
(`AA`/`AAA`) follow
`principles/reference/accessibility_as_quality.md` in `github.com/ctxmod/design/context` (profile `core`).

| Check | Principle "How to verify" item | Tool |
|---|---|---|
| Single `<h1>`, contiguous levels, no skips | a11y-as-quality "Structure as a design lever" + convergent "heading order"; web-accessibility "Semantic HTML is the foundation" (one `h1`, nested in order) | lightpanda tree + `extract` (order); Playwright `browser_snapshot` for explicit `[level=N]` |
| Outline matches reading order | a11y-as-quality "heading hierarchy … visual and semantic order match" | lightpanda tree (DOM order) |
| Landmark roles present + unique | a11y-as-quality convergent "landmark roles" / "Landmark regions"; web-accessibility landmarks | lightpanda tree (`navigation`, `form`); Playwright snapshot for `main` |
| Accessible name on every control | a11y-as-quality convergent "accessible names" / "Labels and names"; web-accessibility `aria-label`/labels | lightpanda tree (name) + `extract` |
| Form input has a `<label>` | web-accessibility "`<label>`-associated controls" (foundation) | lightpanda tree (`textbox 'Email address'`) + `extract` |
| `lang` non-empty | web-accessibility "support … language"; web quality floor | lightpanda `evaluate` |
| `dir` set for RTL | web-accessibility / localization (logical-vs-visual order) | lightpanda `evaluate` → `consistency-localization.md` |
| `<title>` + viewport meta present | web-accessibility "responsive to small screens" (web quality floor) | lightpanda `evaluate` |
| Target size *declared* (`min-height: 44px`) | a11y-as-quality SC 2.5.8 (AA) / SC 2.5.5 (AAA); web-accessibility "Target size" | lightpanda raw `<style>` text (declared only) |
| Target size *painted* (rect ≥ floor) | a11y-as-quality SC 2.5.8 / 2.5.5 (painted) | Playwright `browser_evaluate` `getBoundingClientRect()` + `getComputedStyle` |
| Focus indicator present + ≥ 3:1 contrast (SC 2.4.7) | a11y-as-quality convergent "keyboard operability and focus visibility (SC 2.4.7)" | Playwright only — focus via `browser_press` Tab, then read the focused element's `getComputedStyle` `outline`/`box-shadow` and run the contrast math (ring *presence* + non-text contrast); the per-path keyboard walkthrough stays manual |

## What this reference does not cover

The **divergent** structural judgments stay manual (see
<ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::web_validation@automation/reference/validation_overview.md" />): whether a role is the
*correct* role (a tree shows `navigation` exists, not that it should be a
`navigation`), whether announced ARIA state matches reality, focus order and
focus-not-obscured under sticky chrome (SC 2.4.11 / 2.4.12 — no layout on
lightpanda), and whether meaning survives the loss of a single channel.
**Focus visibility (SC 2.4.7)** is split: the convergent slice — a focus ring is
*present* and meets the ≥ 3:1 non-text-contrast floor — is Playwright-checkable
(focus the element, read its computed `outline`/`box-shadow`, run the contrast
math; see the mapping row above), but verifying focus on *every* keyboard path,
in order, and never trapped is the manual keyboard walkthrough. Style and
contrast checks live in <ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::web_validation@automation/reference/visual_system_styles.md" />.

## Related

- `principles/reference/accessibility_as_quality.md` in `github.com/ctxmod/design/context` (profile `core`)
- `platform/reference/web_accessibility.md` in `github.com/ctxmod/design/context` (profile `web`)
- <ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::browsing@automation/reference/lightpanda.md" />
