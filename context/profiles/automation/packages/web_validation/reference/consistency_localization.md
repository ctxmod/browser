# Consistency & Localization

Cross-route consistency (same role → same accessible-name pattern and token vocabulary across routes) plus i18n checks — lang/dir correctness, logical-DOM-vs-visual order, the RTL rethink-not-mirror obligation, and translatable-content shape — run on lightpanda's semantic tree, escalating RTL geometry to Playwright.

This reference operationalizes two related obligations on a live frontend:

- **Cross-route consistency** — the same role rendered on different routes should
  carry the **same accessible-name pattern** and draw from the **same token
  vocabulary**. A `nav` is the same `nav` everywhere; an `h1` is one per page
  everywhere; a primary `button` keeps its name shape and its token usage. This is
  the runtime expression of the role/slot anchoring in
  `components/reference/taxonomy.md` in `github.com/ctxmod/design/context` (profile `core`) — recurring
  roles have recurring semantics, so drift between routes is a defect.
- **Internationalization** — the convergent slice of
  `principles/reference/internationalization.md` in `github.com/ctxmod/design/context` (profile `core`): `lang`
  set and correct, `dir` set for RTL locales, logical (DOM/semantic) order
  matching the intended visual scan, and translatable content shaped so grammar
  survives translation.

The principle files are the *normative* source; this is the *operational*
companion. Read <ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::web_validation@automation/reference/validation_overview.md" /> first
for the lightpanda-vs-Playwright capability map and the default-first rule, and
<ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::web_validation@automation/reference/accessibility_structure.md" /> for the
single-route structural checks this reference extends across routes and locales.

> **Honesty note on RTL.** The probe test-bed apps (nextjs, vite-csr, workers-ssr)
> are all **LTR** — `document.documentElement.dir` is unset on every route. So the
> consistency, `lang`, and logical-order checks below are **run in-repo** against
> real probe output, but the **RTL-specific** checks (`dir="rtl"`, mirrored
> navigation, logical-order-under-RTL) are **patterns, not run-in-repo** — there is
> no RTL route to probe. Each RTL snippet is marked **PATTERN (not run in-repo)**
> explicitly. Source for all runnable output: the probe report.

## 1. Consistency across routes (the core check)

The check: visit each route, extract the **same role** (the `nav` links, the `h1`,
the primary `button`), and assert the accessible-name pattern is uniform — same
nav label set, exactly one `h1`, names following the same shape. Per
`components/reference/taxonomy.md` in `github.com/ctxmod/design/context` (profile `core`), a role's
semantics are anchored once and reused; a nav whose link set or labels *change
between routes* (beyond the active item) is drift to flag.

`page.goto` + `page.extract` per route is the proven lightpanda primary (the probe
verified it on the home route). Loop the routes and emit a per-route row; the
agent then compares rows for drift:

```js
const page = new Page();
const routes = ["/", "/about"];
for (const r of routes) {
  await page.goto("http://localhost:3137" + r);
  console.log("ROUTE " + r + " " + JSON.stringify(page.extract({
    h1: "h1",
    navText: ["nav[aria-label='Primary'] a"],
    navHrefs: [{ selector: "nav[aria-label='Primary'] a", fields: { href: { attr: "href" } } }],
    headings: ["h2"],
  })));
}
```

The home-route row is **verbatim** from the probe's proven `page.extract` run:

```
ROUTE / {"h1":"Home","navText":["Home","About","Dashboard"],"navHrefs":[{"href":"http://localhost:3137/"},{"href":"http://localhost:3137/about"},{"href":"http://localhost:3137/dashboard"}],"headings":["Counter","Newsletter"]}
```

The `/about` row's values are read from the probe's **verbatim `/about` semantic-tree
dump** (the same `page.extract` call applied to the second route — the probe captured
`/about` as a tree rather than re-running the extract):

```
$ lightpanda fetch --dump semantic_tree_text http://localhost:3137/about
1 RootWebArea 'Next.js Example'
 5 heading 'About'
 6 navigation 'Primary'
  7 [i] link 'Home'
  8 [i] link 'About'
  9 [i] link 'Dashboard'
 10 heading 'Purpose'
 ...
```

→ giving the comparison table the agent asserts against:

| Route | `h1` | `nav` links (names) | one `h1`? |
|---|---|---|---|
| `/` | `Home` | `Home`, `About`, `Dashboard` | yes |
| `/about` | `About` | `Home`, `About`, `Dashboard` | yes |

**Pass:** the `nav 'Primary'` landmark carries the **identical three-link set with
identical accessible names** on both routes (only the `h1` changes, as it should —
it names the page); exactly one `h1` per route. **What to flag:** a nav whose link
set or labels differ between routes (beyond the active-state marker), a route with
zero or multiple `h1`s, or a primary action whose name shape drifts (`Subscribe`
on one route, `Sign up now!` for the same role on another). Note workers-ssr ships
a **two-link** nav (`Home`, `About`) — consistent *within* that app; consistency
is asserted per app, not across unrelated apps.

## 2. Token usage consistency (same component → same token)

The consistency principle's token side: the same role should resolve through the
**same semantic token**, not an ad-hoc value per route. The mechanics are owned by
<ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::web_validation@automation/reference/visual_system_styles.md" /> — do not re-derive
them here. The split that matters for *consistency*:

- **lightpanda (presence):** "is the token system referenced consistently?" — parse
  the raw `<style>` text and confirm the same `--token` vocabulary is declared.
  The probe confirmed all three test-bed apps share **one identical token set**
  (`--color-bg/-fg/-muted/-accent/-border`, `--step-0..4`, `--space-1..5`,
  `--font-body`), and the home route's `:root{…}` block reads back intact:

  ```
  STYLE_TAG_PRESENT len=1829 head= :root {   --color-bg: #ffffff;   --color-fg: #1a1a1a;   --c
  ```

- **Playwright (values):** whether the same role *resolves* to the same value
  across routes needs the cascade — `getComputedStyle(...).getPropertyValue('--…')`
  or a per-route computed read. lightpanda's `getComputedStyle` returns UA defaults
  for every delivery method (the load-bearing probe correction), so resolved
  cross-route token comparison is Playwright-only. See
  <ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::web_validation@automation/reference/visual_system_styles.md" /> §"Token presence vs
  token values".

## 3. `lang` correctness

`principles/reference/internationalization.md` in `github.com/ctxmod/design/context` (profile `core`) treats locale
as a token mode; the floor is that the document **declares its language** and that
the declared language **matches the delivered content**. Read it on lightpanda via
`evaluate` (no cascade involved). Verbatim probe:

```js
page.evaluate("document.documentElement.lang")   // LANG
```

```
LANG en
```

**Pass:** `document.documentElement.lang` is non-empty (`en`) and matches the
English content rendered in the tree. **What to flag** (a real, common bug): a
missing or empty `lang` (`""`) — screen readers then guess the language and
mispronounce content — or a `lang` that contradicts the content (`lang="en"` on a
page rendering German). The check is two-part: non-empty **and** consistent with
the content the semantic tree actually shows.

## 4. `dir` for RTL — PATTERN (not run in-repo)

`principles/reference/internationalization.md` in `github.com/ctxmod/design/context` (profile `core`) is explicit
that **RTL is spatial rethinking, not mirroring**: switching to a RTL locale
reorders navigation, flips directional icons (but not clocks/logos/play-buttons),
and must come from logical properties + the `dir` attribute, not a one-off reversed
stylesheet. The convergent, runtime-checkable part is: **`<html dir="rtl">` is set
for RTL locales, and the logical DOM order is correct** (visual mirroring is then
CSS's job via logical properties).

The `dir` read uses the **identical proven `evaluate` mechanism** that the probe
validated for `lang`/viewport/title:

```js
page.evaluate("document.documentElement.dir")    // DIR
```

On the **LTR** test-bed this returns the empty string (`dir` unset) — expected, and
why the RTL assertion itself cannot be exercised here. **PATTERN (not run in-repo)
— no RTL route exists in the test-bed.** Against an RTL locale the check would be:

```js
// PATTERN — not run in-repo (test-bed is LTR; no dir="rtl" route to probe)
const dir = page.evaluate("document.documentElement.dir");
// assert: for an RTL locale (ar/he/fa), dir === "rtl"  (not "" and not "ltr")
```

**What to flag:** an RTL-locale route whose `dir` is unset or `ltr` (the
`direction: rtl`-as-the-whole-story anti-pattern), and — escalated to Playwright,
because it needs real layout — navigation/icons that are *visually* mirrored by
hand rather than by logical properties (`browser_resize` + `browser_take_screenshot`
to see the painted RTL layout; logical-order itself is checkable on the tree, per
§5). lightpanda confirms `dir` and the logical order is correct; only Playwright
can confirm the *painted* RTL layout reorders rather than naively mirrors
(directional icons flipped, clocks/logos/charts **not** flipped).

## 5. Logical vs visual order

`principles/reference/visual_hierarchy.md` in `github.com/ctxmod/design/context` (profile `core`) requires a
**semantic match**: "read the heading-level / landmark outline (or run an
accessibility tree dump); its order must equal the visual order." The semantic tree
*is* the logical (DOM/reading) order, so one dump answers it. Verbatim probe, nextjs
home route:

```
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
```

The logical order is `h1 'Home'` → nav → intro paragraph → `h2 'Counter'` + its
control → `h2 'Newsletter'` + its form — top-to-bottom, matching the intended
visual scan (title, then navigation, then content sections in order). **Pass:** DOM
order equals the visual scan path; no section is reordered by CSS away from its
semantic position. **What to flag:** a tree whose order contradicts the visual
layout (e.g. a footer node appearing before main content, or a heading that visually
leads a section sitting after it in the DOM) — the screen-reader / tab order then
diverges from what sighted users see.

**For RTL this check is *more* load-bearing, not less:** the logical (DOM) order is
what the principle governs; visual mirroring is purely CSS over that same logical
order. So the *same* tree dump validates the logical order under RTL — only the
painted left/right mirroring (Playwright) differs. (The RTL run itself is **PATTERN
(not run in-repo)** — same reason as §4.)

## 6. Translatable-content shape (light — mostly a source pointer)

The grammar-survives-translation obligation is largely a **source-level i18n
review** (are strings externalized; are they concatenated in JS in a way that
breaks word order or pluralization in other locales). At the *rendered* layer you
can do a light sanity pass: `extract` the visible text and check for **naive
interpolation artifacts** that leaked through — un-substituted placeholders
(`Hello {name}`, `%s items`, `${count}`), doubled spaces from empty interpolation,
or sentence fragments concatenated without spacing.

```js
console.log("TEXT " + JSON.stringify(page.extract({
  intro: "main p",
  headings: ["h2"],
})));
```

```
TEXT {"intro":"Minimal Next.js test-bed for frontend-validation references.","headings":["Counter","Newsletter"]}
```

The probe rendered text shows **no** placeholder artifacts (no `{…}`/`%s`/`${…}`,
clean spacing) — a pass at the rendered layer. **What to flag:** a literal
`{placeholder}` / `%s` / `${var}` surfacing in rendered text (interpolation that
didn't run), or strings glued together such that translation would scramble word
order. Keep this check light: it can only catch artifacts that *reached the DOM*.
Whether translatable strings are correctly externalized and grammatically composed
is a **source review**, not a runtime probe — escalate there.

## Mapping: check → principle obligation → tool

Each check maps to an obligation in
`principles/reference/internationalization.md` in `github.com/ctxmod/design/context` (profile `core`) (or, for
cross-route role consistency,
`components/reference/taxonomy.md` in `github.com/ctxmod/design/context` (profile `core`)), then to the
tool. lightpanda owns tree/extraction/`lang`/`dir`/logical-order (text); Playwright
owns the RTL **layout/geometry** the cascade-and-paint require.

| Check | Principle obligation | Tool |
|---|---|---|
| Same role → same accessible-name set across routes | taxonomy: recurring role → recurring semantics | lightpanda `extract` per route (run in-repo) |
| Same component → same token vocabulary | i18n: locale re-resolves the *same* semantic roles, no new layer | lightpanda raw `<style>` (presence); Playwright resolved values |
| `lang` set + matches content | i18n: locale is a token mode; language declared correctly | lightpanda `evaluate` (run in-repo) |
| `dir="rtl"` for RTL locales | i18n: "`direction: rtl` as the whole RTL story" is an anti-pattern | lightpanda `evaluate` — **PATTERN (not run in-repo)** |
| Logical (DOM) order = visual scan | i18n + visual-hierarchy "Semantic match" | lightpanda semantic tree (run in-repo) |
| RTL layout reorders, not naive-mirrors (icons flip, clocks/logos don't) | i18n: "Flip only directional icons"; logical properties + `dir` | Playwright `browser_resize` + `browser_take_screenshot` — **PATTERN (not run in-repo)** |
| Translatable-content shape (rendered artifacts) | i18n: string-only i18n / broken-grammar anti-patterns | lightpanda `extract` (light; source review is the real check) |

## Proven snippets (real probe output)

Every output below is **verbatim** from the probe report.
RTL-specific snippets are marked **PATTERN (not run in-repo)** because the test-bed
is LTR-only.

### A. Multi-route consistency extraction (run in-repo)

Home route, verbatim from the probe's `page.extract` run:

```
ROUTE / {"h1":"Home","navText":["Home","About","Dashboard"],"navHrefs":[{"href":"http://localhost:3137/"},{"href":"http://localhost:3137/about"},{"href":"http://localhost:3137/dashboard"}],"headings":["Counter","Newsletter"]}
```

`/about` route, read from the probe's verbatim `/about` semantic-tree dump (same role
extraction, second route): `h1 'About'`, `nav 'Primary'` → `Home`/`About`/
`Dashboard`. → the nav link-name set is **identical across both routes**; only the
`h1` changes. Consistency holds.

### B. `lang` read (run in-repo)

```
LANG en
```

`document.documentElement.lang` — non-empty and matches the English content. The
**`dir`** read uses the identical `evaluate` mechanism and returns `""` on the LTR
test-bed (`dir` unset).

### C. Semantic (logical) order read (run in-repo)

```
1 RootWebArea 'Next.js Example'
 5 heading 'Home'
 6 navigation 'Primary'
 10 paragraph
 12 heading 'Counter'
 17 [i] button 'Increment'
 18 heading 'Newsletter'
 19 form
```

DOM/semantic order = the intended top-to-bottom visual scan — a semantic-match
pass. For RTL this same dump is the logical-order check (visual mirroring is CSS).

### D. RTL `dir` + logical-order assertion — PATTERN (not run in-repo)

```js
// PATTERN — not run in-repo: the probe test-bed is LTR-only; no dir="rtl" route exists.
// Against an RTL locale (ar/he/fa) the convergent checks would be:
const dir = page.evaluate("document.documentElement.dir");   // assert === "rtl"
// then dump the semantic tree and assert the LOGICAL order is correct
// (the same §5 check); visual left/right mirroring is CSS over this logical order
// and is confirmed painted-correct via Playwright browser_take_screenshot —
// directional icons flipped, clocks/logos/charts NOT flipped.
```

This is the one check set this reference cannot exercise in-repo. It is included so
the obligation is documented; mark any report using it as pattern-derived until an
RTL route is added to the test-bed.

## What this reference does not cover

- **Resolved cross-route token values** and contrast/type-scale per route — those
  are the cascade checks in
  <ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::web_validation@automation/reference/visual_system_styles.md" /> (Playwright-only).
- **Single-route structural a11y** (heading outline, landmarks, names, target size)
  — see <ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::web_validation@automation/reference/accessibility_structure.md" />.
- **Source-level i18n correctness** — string externalization, ICU/plural rules,
  per-locale typography token resolution (CJK line-height, Arabic tracking, German
  density flex). These are normative obligations in
  `principles/reference/internationalization.md` in `github.com/ctxmod/design/context` (profile `core`) verified by
  source review and a real RTL/CJK build, not by probing the LTR test-bed.

## Related

- `principles/reference/internationalization.md` in `github.com/ctxmod/design/context` (profile `core`)
- `components/reference/taxonomy.md` in `github.com/ctxmod/design/context` (profile `core`)
