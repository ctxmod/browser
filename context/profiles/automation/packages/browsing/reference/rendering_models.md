# Rendering Models

Detect which rendering model a route uses (SSR, CSR, hybrid/CSR-island, edge SSR) by contrasting raw HTML against the post-hydration tree, and the per-model wait/probe strategy — goto returns at load, so client-filled content needs waitForSelector/networkidle before re-extract.

Before any structure, style, or behavior check, an agent has to know **when the
content it wants to assert on actually exists in the DOM**. A route that ships
its content in the first byte (SSR) is probed differently from one that ships an
empty mount and fills it after the JS bundle runs (CSR). Get this wrong and you
will read an empty tree, conclude the page is broken, and report a phantom
failure.

This reference gives the **generalizable detect-then-wait procedure**: detect the
rendering model from a raw-HTML-vs-hydrated contrast, then apply the matching
wait strategy before extracting. It is the prerequisite for
<ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::web_validation@automation/reference/functional_testing.md" /> (you cannot drive or
observe state you have not waited for) and a companion to
<ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::web_validation@automation/reference/validation_overview.md" />. Every output below is
pasted verbatim from the probe report and the three test-bed app reports.

## The models (and where you see each)

- **SSR** — content is in the initial HTML response. A raw `curl` already shows
  the headings/nav/form. Seen on Next.js `/` and `/about`.
- **CSR** — the served `<body>` is an empty mount (`<div id="root"></div>`); JS
  fills everything after the bundle runs. Seen on the Vite SPA.
- **Hybrid / CSR-island** — an SSR shell ships immediately, but one region is a
  client island that hydrates and fetches its own data; that region shows a
  placeholder (`Loading…`) in the initial HTML and appears only after hydration.
  Seen on Next.js `/dashboard`.
- **Edge SSR** — server-rendered at the edge (Cloudflare workerd), no Node APIs,
  no client bundle. From the agent's side it reads **identically to SSR**:
  content is in the first byte. Seen on the Workers app.

## The detection probe

The generalizable test is a **two-reads contrast**: read the page with no JS,
then read it after JS/hydration, and compare.

1. **Raw, no JS:** `curl -s <url>` (or `lightpanda fetch --dump
   semantic_tree_text --wait-until load <url>`, which captures the pre-settle
   state).
2. **After JS/hydration:** `lightpanda fetch --dump semantic_tree_text <url>`
   (default `--wait-until done` settles ~5 s and runs JS) or `--wait-until
   networkidle` for the post-data state. Playwright's `browser_snapshot` after
   navigate is the heavier equivalent.

Then classify:

- Meaningful content already in the raw HTML → **SSR** (or edge SSR).
- Raw `<body>` is an empty `#root`, full tree only after JS → **pure CSR**.
- Shell present in raw HTML but a region is `Loading…`/absent until a
  `waitForSelector` → **hybrid / CSR island**.

### Real contrasts from the probe

**Next.js `/` — content in the initial HTML (SSR).** The raw `curl` already
carries the `<h1>`, nav, both `<h2>`s, and the form:

```
$ curl -s http://localhost:3137/ | grep -o '<main>.*</main>'
<main><h1>Home</h1><nav aria-label="Primary"><a href="/">Home</a>
<a href="/about">About</a><a href="/dashboard">Dashboard</a></nav>
<p>Minimal Next.js test-bed for frontend-validation references.</p>
<h2>Counter</h2><div><p data-testid="count">Count: <!-- -->0</p>
<button type="button">Increment</button></div><h2>Newsletter</h2>
<form><label for="email">Email address</label>
<input id="email" type="email" placeholder="you@example.com" required="" name="email"/>
<button type="submit">Subscribe</button></form></main>
```

`/about` is the same — full content in the initial HTML:

```
$ curl -s http://localhost:3137/about | grep -o '<main>.*</main>'
<main><h1>About</h1><nav aria-label="Primary">…</nav>
<h2>Purpose</h2><p>This is a minimal Next.js example…</p>
<h2>Rendering models</h2><p>Home and About are server-rendered (SSR).…</p></main>
```

**Next.js `/dashboard` — `Loading…` only in the SSR HTML; metrics appear after
hydration (hybrid / CSR island).** The shell (`h1`, nav, `h2`) is server-rendered,
but the fetched list is provably absent at load:

```
$ curl -s http://localhost:3137/dashboard | grep -o '<main>.*</main>'
<main><h1>Dashboard</h1><nav aria-label="Primary">…</nav>
<h2>Metrics</h2><p data-testid="loading">Loading…</p></main>

$ curl -s http://localhost:3137/dashboard | grep -c 'Deploys'               # fetched item
0
$ curl -s http://localhost:3137/dashboard | grep -c 'data-testid="metrics"' # the <ul>
0
$ curl -s http://localhost:3137/dashboard | grep -c 'data-testid="loading"' # placeholder
1
```

**Vite — empty `#root` in `curl`; full tree after JS (pure CSR).** The entire raw
`<body>` is just the mount node + the module script:

```
$ curl -s http://localhost:5174/
...
  <body>
    <!-- Pure CSR: the body ships empty. All content is injected by JS post-load. -->
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
...
$ curl -s http://localhost:5174/ | grep -c 'id="root"'        # mount present
1
$ curl -s http://localhost:5174/ | grep -c 'Deploys'          # metrics item — absent
0
$ curl -s http://localhost:5174/ | grep -c 'Newsletter'       # h2/form — absent
0
```

After JS, the same route's lightpanda tree is fully populated — h1, the metrics
list, and the form, none of which existed in the `curl`:

```
$ lightpanda fetch --dump semantic_tree_text http://localhost:5174/
1 RootWebArea 'Vite CSR example'
 4 [i]
  6 heading 'Vite CSR'
  7 paragraph
   8 'Pure client-side-rendered SPA. The body is empty at load; this content is injected by JavaScript after the bundle runs.'
  9 heading 'Metrics'
  12 'Deploys'
  13 ':'
  14 '128'
  ...
  23 heading 'Newsletter'
  24 form
   27 [i] textbox 'Email address'
   28 [i] button 'Subscribe'
```

**Workers — full content in the initial HTML (edge SSR).** Every element is in
the first response; there is no client bundle:

```
$ curl -s http://127.0.0.1:8791/ | grep -o '<main>.*</main>'
<main><h1>Workers SSR</h1><nav aria-label="Primary"><a href="/">Home</a>
<a href="/about">About</a></nav><p>Edge server-rendered HTML on the Cloudflare
workerd runtime. Every element below is present in the initial response — no
client JS injects it.</p><h2>Features</h2><ul data-testid="features">
<li><strong>Edge SSR</strong>…</li>…</ul><h2>Newsletter</h2>
<form method="post" action="/subscribe">…</form></main>
```

Edge SSR reads identically to a server-framework SSR app from the agent's side:
content is present immediately, one-shot `fetch`/`extract` suffices, and `load`
== `networkidle` for the structural content.

## Per-model wait strategy

**`page.goto` returns at the `load` event** — before any post-load `setTimeout`
or data-fetch fires. So for CSR and hybrid routes you **must** wait after `goto`
and re-extract; for SSR and edge SSR there is nothing to wait for.

| Model | Wait after navigate | Why |
|---|---|---|
| SSR | none | content is in the first byte |
| Edge SSR | none | `load` == `networkidle` for structural content; no client bundle |
| Pure CSR | `waitForSelector(<expected>)` or `waitForState({state:'networkidle'})`, then re-extract | empty `#root` until the bundle runs |
| Hybrid / CSR island | `waitForSelector(<island's expected node>)`, then re-extract | shell is immediate; the island fills later |

The decisive before/after, from the probe timing section. At `--wait-until load`
the dashboard shows only the placeholder; at `--wait-until networkidle` the
metrics are present:

```
$ lightpanda fetch --dump semantic_tree_text --wait-until load http://localhost:3137/dashboard
... 10 heading 'Metrics'
    11 paragraph
     12 'Loading…'                 <-- placeholder only; NO metrics

$ lightpanda fetch --dump semantic_tree_text --wait-until networkidle http://localhost:3137/dashboard
... 10 heading 'Metrics'
     13 'Deploys'                  <-- metrics now present
     15 'Sessions'
     17 'Errors'
```

The same load-vs-networkidle gap on the pure-CSR Vite route:

```
$ lightpanda fetch --dump semantic_tree_text --wait-until load http://localhost:5174/
... 9 heading 'Metrics'
   10 paragraph
    11 'Loading…'                  <-- shell rendered, metrics absent

$ lightpanda fetch --dump semantic_tree_text --wait-until networkidle http://localhost:5174/
... 9 heading 'Metrics'
   12 'Deploys' : 128              <-- metrics present
   16 'Latency p95' : 42ms
   20 'Uptime' : 99.98%
```

## Toolchain notes

Short, practical: the rendering model is a property of the *route*, not just the
framework — but the toolchain tells you what to expect.

- **Next.js (App Router)** — React Server Components by default; `"use client"`
  marks a client island. Static/server routes are SSR (content in the first
  byte, as `/` and `/about`); a client island that fetches its own data is the
  hybrid case (`/dashboard`: SSR shell + `Loading…`, metrics after hydration).
  Probe per route, not per app.
- **Vite** — pure-CSR SPA. The served `<body>` is an empty `#root`; everything is
  JS-injected. Always wait for content. Production build preserves the empty-body
  property (the built `dist/index.html` body is still just the mount node).
- **Turbopack** — Next's dev bundler default now. It changes dev-time behavior
  (fast refresh, chunk shapes) but not the SSR/CSR/hybrid split of a route, so
  the detection probe is unchanged. Caveat: dev and prod can differ in timing and
  chunking — validate against `pnpm build` output when timing matters, not just
  `pnpm dev`.
- **Cloudflare Workers** — Hono + JSX (or a framework's Workers adapter) on the
  **workerd** runtime: edge SSR, **no Node APIs**. There is no client bundle, so
  it probes exactly like server SSR — content in the first byte, no hydration
  wait. The newsletter form here is a real `method="post" action="/subscribe"`,
  not a `preventDefault` SPA form, which is useful for navigation-flow testing.

For the design-time consequences of which model and how much JS a route ships,
see `platform/reference/rendering_performance.md` in `github.com/ctxmod/design/context` (profile `web`).

## Capability honesty

Reuse the canonical capability table from
<ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::web_validation@automation/reference/validation_overview.md" />.
The point relevant here: **rendering-model detection works on lightpanda** because
it is a **text contrast** — content present vs. absent in the semantic tree /
extracted fields, before vs. after a wait. lightpanda's `waitForSelector` /
`waitForState` and `--wait-until load|done|networkidle` are sufficient to classify
SSR vs CSR vs hybrid vs edge and to wait for client-filled content.

What lightpanda does **not** do is measure **computed/layout differences across
models** — there is no layout engine, no CSS cascade. `getComputedStyle` returns
UA defaults (`color` → `rgb(0,0,0)`), custom-property reads return `''`, and
geometry is a placeholder (`offsetWidth` → `5`, `getBoundingClientRect()` → `{}`).
So "is this content server- or client-rendered?" is a lightpanda check; "did
hydration shift layout / change paint" is **not** — escalate to Playwright
(`browser_evaluate` for the resolved cascade, `browser_take_screenshot`,
`browser_console_messages` to catch hydration errors).

## Proven snippets

Every output below is pasted verbatim from the probe report
or the test-bed app reports; nothing here is unverified.

### Detection probe on an SSR route (content present, no wait)

A raw read already carries the content — classify as SSR, extract immediately
(Workers edge SSR):

```
$ curl -s http://127.0.0.1:8791/ | grep -c '<h1>Workers SSR</h1>'   # 1
$ curl -s http://127.0.0.1:8791/ | grep -c 'data-testid="features"' # 1  (the <ul>)
$ curl -s http://127.0.0.1:8791/ | grep -c 'Edge SSR'               # 1  (list item)
```

A one-shot lightpanda `extract` confirms the populated tree with no wait:

```
EXTRACT {"title":"Workers SSR example","h1":"Workers SSR","navText":["Home","About"],"headings":["Features","Newsletter"],"features":["Edge SSR","No Node APIs","Hydration-ready"],"formFields":[{"type":"email","name":"email"}],"formAction":[{"action":"/subscribe","method":"post"}]}
```

### Detection probe on a CSR route (empty raw body, full tree after JS)

Raw `curl` shows an empty `#root` and zero content; the post-JS tree is full —
classify as pure CSR:

```
$ curl -s http://localhost:5174/ | grep -c 'id="root"'   # 1  (mount present)
$ curl -s http://localhost:5174/ | grep -c 'Deploys'     # 0  (metrics absent)
$ curl -s http://localhost:5174/ | grep -c 'Newsletter'  # 0  (h2/form absent)
```

### Wait-then-extract on the hybrid dashboard (the load-vs-after contrast)

`goto` returns at `load` (before the async data), so probe → wait → probe. The
extraction flips from empty to populated across the wait:

```js
const page = new Page();
await page.goto("http://localhost:5174/", { waitUntil: "load" });
console.log("BEFORE " + JSON.stringify(page.extract({ loading: "[data-testid='loading']", metrics: ["ul[data-testid='metrics'] li"] })));
page.waitForSelector({ selector: "ul[data-testid='metrics']", timeout: 3000 });
console.log("AFTER " + JSON.stringify(page.extract({ loading: "[data-testid='loading']", metrics: ["ul[data-testid='metrics'] li"] })));
```

```
BEFORE {"loading":"Loading…","metrics":[]}
AFTER {"loading":null,"metrics":["Deploys: 128","Latency p95: 42ms","Uptime: 99.98%"]}
```

`waitForSelector` (hydration/element) or `waitForState({state:'networkidle'})`
(data-fetch settle) flips the same extraction from empty to populated. The same
pattern applies to the Next.js `/dashboard` island — wait for
`[data-testid="metrics"]` before asserting the list exists. For edge SSR there is
no such gap; one read suffices. Once content is present, drive and observe state
per <ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::web_validation@automation/reference/functional_testing.md" />.

## Related

- <ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::browsing@automation/reference/lightpanda.md" />
- `platform/reference/rendering_performance.md` in `github.com/ctxmod/design/context` (profile `web`)
