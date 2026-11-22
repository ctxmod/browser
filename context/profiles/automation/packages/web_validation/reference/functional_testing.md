# Functional Testing

Efficient interaction testing on lightpanda — navigate-and-read in one pass, the drive-then-re-extract loop (click/fill/press → assert state), form-value round-trips, navigation/route assertions, the $LP_* credential pattern for auth-gated flows, and the stale-node rule (re-inspect after every page change).

Functional testing is the **drive-then-observe** loop: perform an action
(`click`/`fill`/`press`/`selectOption`), then re-extract to assert the state
actually changed. lightpanda runs the page's real JavaScript, so React/Vue/Hono
state, controlled inputs, and route transitions all behave as they would in a
browser — you just have to *re-read* after every action, because the action
changes the DOM out from under any node references you were holding.

This reference is the behavioral half of the validation set. It assumes you have
already navigated and waited for content per
<ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::browsing@automation/reference/rendering_models.md" /> (you cannot drive or
observe state you have not waited for) and sits under the capability map in
<ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::web_validation@automation/reference/validation_overview.md" />. The credential and
primitive details it leans on live in
<ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::browsing@automation/reference/lightpanda.md" />. Every output below is pasted
verbatim from the probe report; nothing here is unverified.

## Navigate-and-read in one pass

Prefer a single navigate-and-read over a bare `goto` followed by a separate read
— the extra round-trip buys nothing. Two cheap one-call forms:

- `markdown {url}` — navigate and return the page as markdown in one call (best
  when you just want to *see* the content).
- `lightpanda fetch --dump semantic_tree_text <url>` — navigate and return the
  structure/a11y outline in one call (best when you want roles/names/headings).

In a PandaScript, the equivalent is `goto` + `extract` in the same run, holding
the result in a local variable. The one trap: **`page.goto` returns at the
`load` event**, before any post-load `setTimeout` or client data-fetch fires. For
SSR/edge-SSR routes that is fine (content is in the first byte); for CSR/hybrid
routes you must `waitForSelector`/`waitForState` before extracting — the full
wait-rule is in <ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::browsing@automation/reference/rendering_models.md" />.

A single navigate-and-read against the SSR Next.js route yields the whole
structure in one call:

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

## The interaction → re-extract loop

The core pattern: capture the state, perform the action, re-extract, compare. The
proven case is the Next.js `Counter` island — three `click`s drive React state
from `0` to `3`, and a re-`extract` observes it:

```js
console.log("COUNT_BEFORE " + JSON.stringify(page.extract({ count: "[data-testid='count']" })));
page.click({ selector: "button[type='button']" });
page.click({ selector: "button[type='button']" });
page.click({ selector: "button[type='button']" });
console.log("COUNT_AFTER " + JSON.stringify(page.extract({ count: "[data-testid='count']" })));
```

```
COUNT_BEFORE {"count":"Count: 0"}
COUNT_AFTER {"count":"Count: 3"}
```

The assertion is the *delta*: extract before, act, extract after, and compare the
two values — never assume the action took. The re-extract is what makes the test
real; without it you have only sent a click into the void.

### Extraction-schema syntax (so your assertions read what you think)

The re-extract is only as good as its schema. The forms that work and the ones
that throw (learned during the probe):

- A **bare-string list** `["h2"]` or `[{ selector, limit }]` returns an **array
  of text** — e.g. `["Counter","Newsletter"]`.
- To read **attributes** you must use the `fields` form:
  `[{ selector, fields: { href: { attr: "href" } } }]`.
- A boolean attribute that is present-but-empty (`required`) extracts as `""`.
- **There is no self-text token.** `fields: { text: "." }` throws
  `exception: The string did not match the expected pattern` (an invalid CSS
  selector). To get an element's own text **and** an attribute, run the bare-list
  form for the text and the `fields` form for the attribute separately, or select
  a child node.

The Next.js extract exercises all four:

```js
const page = new Page();
await page.goto("http://localhost:3137/");
console.log("EXTRACT " + JSON.stringify(page.extract({
  title: "title",
  h1: "h1",
  navText: ["nav[aria-label='Primary'] a"],
  navHrefs: [{ selector: "nav[aria-label='Primary'] a", fields: { href: { attr: "href" } } }],
  headings: ["h2"],
  formFields: [{ selector: "form input", fields: { type: { attr: "type" }, name: { attr: "name" }, required: { attr: "required" } } }]
})));
```

```
EXTRACT {"title":"Next.js Example","h1":"Home","navText":["Home","About","Dashboard"],"navHrefs":[{"href":"http://localhost:3137/"},{"href":"http://localhost:3137/about"},{"href":"http://localhost:3137/dashboard"}],"headings":["Counter","Newsletter"],"formFields":[{"type":"email","name":"email","required":""}]}
```

## Form validation

For a controlled input, the cheapest assertion is the **value round-trip**: read
the field value, `fill` it, read it back. `fill` does not require the CSS cascade,
so it works on lightpanda regardless of rendering model. The pure-CSR Vite app
(empty → typed value):

```js
console.log("EMAIL_BEFORE '" + page.evaluate("document.querySelector('#email').value") + "'");
page.fill({ selector: "#email", value: "agent@example.com" });
console.log("EMAIL_AFTER '" + page.evaluate("document.querySelector('#email').value") + "'");
```

```
EMAIL_BEFORE ''
EMAIL_AFTER 'agent@example.com'
```

The same round-trip on the edge-SSR Workers form, whose `<form>` is a **real
POST** (`action="/subscribe" method="post"`), not a `preventDefault` SPA form
:

```
EMAIL_BEFORE ''
EMAIL_AFTER 'edge@example.com'
```

You can also assert the form *contract* up front by extracting the field
attributes and the form action/method — the Workers extract shows `type:"email"`,
`name:"email"`, and `action:"/subscribe" method:"post"`:

```
EXTRACT {"title":"Workers SSR example","h1":"Workers SSR","navText":["Home","About"],"navHrefs":[{"href":"http://127.0.0.1:8791/"},{"href":"http://127.0.0.1:8791/about"}],"headings":["Features","Newsletter"],"features":["Edge SSR","No Node APIs","Hydration-ready"],"formFields":[{"type":"email","name":"email"}],"formAction":[{"action":"/subscribe","method":"post"}]}
```

For *displayed* validation feedback (an error message that the framework renders
after a bad submit), fill an invalid value, `press({ key: "Enter" })` or click
submit, then re-extract the message region — the same drive-then-re-extract loop.
Whether the **announced** validation state is correct (the `aria-live` /
`aria-invalid` semantics named in
`platform/reference/web_accessibility.md` in `github.com/ctxmod/design/context` (profile `web`)) is a
divergent check: the tree shows a message node exists, not whether a screen
reader announces it correctly.

## Navigation / route assertions

To assert a navigation, capture the **route signature** before and after: the
`<h1>` (or another route marker) and the URL. Extract the link targets first, so
you know where each link goes — the home extract above gives the nav hrefs
(`/`, `/about`, `/dashboard`). Then navigate (click the link, or `goto` the href)
and, on a CSR/hybrid route, `waitForSelector` the new route's signature before
re-extracting.

The decisive point is what survives the navigation: **nothing DOM-shaped does.**
After the move, re-read the tree. The `/about` route's own dump confirms the new
`<h1>` (`heading 'About'`) and that the nav landmark is a *fresh* set of nodes,
not the home page's:

```
$ lightpanda fetch --dump semantic_tree_text http://localhost:3137/about
1 RootWebArea 'Next.js Example'
 5 heading 'About'
 6 navigation 'Primary'
  7 [i] link 'Home'
  8 [i] link 'About'
  9 [i] link 'Dashboard'
 10 heading 'Purpose'
 11 paragraph
  12 'This is a minimal Next.js example for frontend-validation references.'
 13 heading 'Rendering models'
 14 paragraph
  15 'Home and About are server-rendered (SSR). The Dashboard route fetches its data on the client, so its list is absent from the initial HTML.'
```

The assertion is the route-marker delta: home extracted `h1:"Home"`; after
navigating to `/about` the re-read shows `heading 'About'`. Same nav landmark,
same accessible names — but a different `<h1>` and different body content prove
the route changed.

## The stale-node rule (load-bearing)

After **any** page-changing action — a click that navigates, a form submit, a
`fill` that triggers a route change, or any `goto` — **discard every node id and
tree snapshot you captured before the action, and re-`tree`/re-`extract`.** Node
ids and semantic-tree snapshots do **not** survive a navigation; the numeric ids
in one tree dump (`7 [i] link 'Home'`) are not stable across reads and certainly
not across routes.

This is why the loops above re-`extract` rather than reusing a prior result: the
assertion must read the *current* DOM. The same rule is stated in
<ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::browsing@automation/reference/lightpanda.md" /> (agent variables persist across
calls for one script run, but **page** state — and any node reference into it —
is invalidated by navigation). Practical discipline:

- Never cache a `backendNodeId` (or a Playwright `[ref=eN]`) across an action that
  could change the page. Re-query by selector each time.
- After a click/submit that navigates, re-dump the tree before asserting anything
  about the new page.
- Prefer selector-based primitives (`click({ selector })`, `fill({ selector })`)
  over holding a node handle — selectors are re-resolved against the live DOM on
  every call.

## Credentials for auth-gated flows

To drive a login or any auth-gated interaction, pass `$LP_*` placeholders
directly as `fill` values (or in `goto` URLs / selectors). They resolve **inside
the lightpanda process**, so secrets never enter the script text or the LLM
prompt, and resolved values are scrubbed back to placeholders in recordings.
Site-scoped names follow `LP_<SITE>_<FIELD>` (e.g. `$LP_HN_USERNAME`,
`$LP_HN_PASSWORD`), falling back to `$LP_USERNAME` / `$LP_PASSWORD`. To discover
which names are set, call the MCP `getEnv` tool with **no** argument — it returns
names only, never values; never pass a credential name to `getEnv`.

A login flow is the interaction loop with credential placeholders, followed by a
`waitForSelector` on a post-auth route signature and a re-extract to assert you
landed:

```js
const page = new Page();
await page.goto("https://example.com/login");
page.fill({ selector: "input[name='acct']", value: "$LP_SITE_USERNAME" });
page.fill({ selector: "input[name='pw']",   value: "$LP_SITE_PASSWORD" });
page.press({ key: "Enter" });
page.waitForSelector({ selector: "#dashboard", timeout: 3000 });
// stale-node rule: the login page is gone — re-extract the authed route
page.extract({ user: "[data-testid='current-user']" });
```

This pattern is covered in depth in
<ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::browsing@automation/reference/lightpanda.md" /> — see its credentials section;
do not duplicate the placeholder rules, cite them.

## lightpanda vs Playwright for functional tests

Functional behavior works on **both** tools — clicks, fills, presses, and route
transitions drive real state on each. **lightpanda is the default**: it is
text-only, Chrome-free, and token-cheap, and the entire drive-then-re-extract
loop runs on it (the Counter `0→3` and the form round-trips above are all
lightpanda). Reach for Playwright only when the *interaction itself* needs a
capability lightpanda physically lacks:

- **Console errors during the interaction** — `browser_console_messages` captures
  runtime errors and hydration warnings emitted while you click/submit (lightpanda
  has no console capture). In the probe it surfaced a real favicon 404 and
  confirmed clean hydration on `/dashboard`.
- **A screenshot of the resulting state** — `browser_take_screenshot` is the only
  way to capture the painted result of an interaction (lightpanda has no paint
  engine).

For the full per-capability split, reuse the canonical capability table in
<ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::web_validation@automation/reference/validation_overview.md" />.
The one-line rule: drive and assert state on lightpanda; escalate to Playwright
when the interaction must also yield console signals or painted pixels.

## Related

- `platform/reference/web_accessibility.md` in `github.com/ctxmod/design/context` (profile `web`)
