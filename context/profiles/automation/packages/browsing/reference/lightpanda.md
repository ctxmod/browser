# Lightpanda PandaScript

Lightpanda PandaScript — deterministic, LLM-free web extraction in a dedicated V8 context via the Page object API (goto/extract/evaluate), run with `lightpanda run`; output via top-level `return`, $LP_* credential placeholders, list-to-detail pattern. Current against Lightpanda 1.0.x (checked 2026-11-23).

A PandaScript is a web automation script run directly by the Lightpanda
browser — no Node/Python, no Puppeteer/Playwright, no CDP serialization
cost, no LLM required: vanilla JavaScript plus a small set of native
primitives. It is reproducible, deterministic, and token-free.

Run (replay) with:

```
lightpanda run <script>.js
cat <script>.js | lightpanda run -     # or from stdin
```

`lightpanda agent` is the interactive, LLM-driven REPL that *records* a
session; its `/save` command writes the PandaScript that `lightpanda run`
replays without a model.

Always disable telemetry in any process that launches lightpanda:

```
LIGHTPANDA_DISABLE_TELEMETRY=true
```

## Runtime model (read this first)

- **Dedicated V8 context**, separate from the page's JavaScript. There is
  no `window`, `document`, DOM, `localStorage`, `navigator`, or page
  global state in the agent script. Read page data with `extract(...)`, or
  run page JavaScript explicitly with `evaluate(...)`.
- **Not Node.js.** No `require`, `process`, `fs`, `path`, npm loading, or
  filesystem/network API. Standard ECMAScript built-ins (`JSON.parse`,
  `JSON.stringify`, arrays, objects, loops) are available.
- **Agent variables persist across primitive calls and navigations** for
  one script run; a later `lightpanda run script.js` starts fresh. Page
  scripts cannot see agent variables, and vice versa.
- **Output is the top-level `return` value** (objects/arrays printed as
  JSON). A bare trailing expression is **not** printed in 1.0.x — scripts
  written for the 0.3.x "completion value" convention must end in `return`.
  Use `console.log(...)` only for debug output (it does not JSON-format
  objects).

## Async form

Scripts use the **`Page` object API**; the body runs inside an async
wrapper, so **top-level `await` is allowed**. Exactly two methods are
asynchronous and must be awaited — **`goto` and `evaluate`**. Every other
method (`extract`, `click`, `fill`, the `waitFor*` family, …) is synchronous
and blocking:

```js
const page = new Page();
await page.goto("https://example.com");
const title = await page.evaluate("document.title");
return title;
```

Forgetting `await` on `evaluate` yields `[object Promise]` where a value was
expected. (The 0.3.x-era standalone primitives — bare `extract(...)` with no
`Page`, synchronous, no top-level `await` — are not the documented 1.0.x
surface; write new scripts against `Page`.)

## Structured extraction

`page.extract(schema)` is the workhorse. It returns extracted data as a
normal JavaScript value (object for an object schema — even single-field;
array for a bare array schema), so local logic can use it directly.

```js
const page = new Page();
await page.goto("https://news.ycombinator.com/");
const data = page.extract({
  title: "title",
  stories: [{
    selector: "tr.athing",
    limit: 5,
    fields: {
      id:    { attr: "id" },
      title: ".titleline > a",
      url:   { selector: ".titleline > a", attr: "href" }
    }
  }]
});
return data; // -> { title, stories: [...] }
```

Equivalent forms: `page.extract({ title: "h1" })`,
`page.extract({ schema: { title: "h1" } })`,
`page.extract('{ "title": "h1" }')`. The wrapped `{ schema: ... }` form
does **not** accept `save=` in scripts — keep results in local variables.

## Page JavaScript (`evaluate`)

`page.evaluate(scriptString)` is the escape hatch into the page's JS
context (where `window`/`document` exist):

```js
await page.goto("https://example.com");
const title = await page.evaluate("document.title");
```

Keep the boundary clear: page `evaluate` cannot see agent variables, cannot
call `goto`/`extract`, and agent code cannot touch `document` directly.
Prefer `extract(...)`; use `evaluate(...)` only for page behavior extraction
cannot express. `page.waitForScript(expr)` evaluates in the page context
repeatedly until truthy or timeout.

## Credentials (`$LP_*`)

Pass `$LP_*` placeholders directly in any string argument (selectors, fill
values, goto URLs). They resolve inside the Lightpanda process, so secrets
never enter the script text or LLM prompt. In recordings, resolved values
are scrubbed back to placeholders.

Site-scoped vars follow `LP_<SITE>_<FIELD>` (e.g. `$LP_HN_USERNAME`,
`$LP_GH_TOKEN`); fall back to `$LP_USERNAME` / `$LP_PASSWORD`. To discover
what's available, call the MCP `getEnv` tool with **no** name (returns
names only, never values) — never pass a credential name to `getEnv`.

```js
const page = new Page();
await page.goto("https://news.ycombinator.com/login");
page.fill({ selector: "input[name='acct']", value: "$LP_HN_USERNAME" });
page.fill({ selector: "input[name='pw']",   value: "$LP_HN_PASSWORD" });
page.press({ key: "Enter" });
```

## Interaction primitives

Action primitives operate on the current page; most take one object whose
fields match the browser tool schema: `click`, `fill`, `press`, `hover`,
`selectOption`, `setChecked`, `scroll`, `waitForSelector`, `waitForScript`,
`waitForState`. Notes: `setChecked` defaults to checking; `press`'s leading
positional is an optional **selector**, not `key` — use `press({ key:
"Enter" })` to press on the focused element.

## Error handling

Primitive failures throw JavaScript `Error` and stop execution unless
caught:

```js
try {
  page.waitForSelector({ selector: "#dashboard", timeout: 1000 });
} catch (err) {
  console.error("dashboard did not appear:", err.message);
  throw err;
}
```

Common failures:
- `ReferenceError: document is not defined` — used DOM in the agent
  context. Use `extract(...)` or `page.evaluate(...)`.
- `ReferenceError: require is not defined` — agent scripts are not Node.
- `[object Promise]` where a value was expected — `evaluate` is async;
  `await` it.
- `no page loaded - run page.goto(url) first` — a page primitive ran before
  navigation.
- `invalid arguments` — wrong argument count or shape, or a value that is
  not JSON-serializable.
- `extract: no schema selector matched any element` — every schema field
  missed (an empty region yields `null`/`[]` per field, not this error).

## List-to-detail example (canonical)

Open Hacker News, extract five stories, then visit each comments page and
extract comments — looping in the agent script, not the page:

```js
const HN = "https://news.ycombinator.com";
const page = new Page();
await page.goto(HN);
const { stories } = page.extract({
  stories: [{
    selector: "tr.athing",
    limit: 5,
    fields: {
      id:    { attr: "id" },
      title: ".titleline > a",
      url:   { selector: ".titleline > a", attr: "href" }
    }
  }]
});
for (const story of stories) {
  story.comments = [];
  if (!story.id) continue;
  await page.goto(`${HN}/item?id=${story.id}`);
  const { comments } = page.extract({
    comments: [{
      selector: "tr.athing.comtr:has(.commtext)",
      limit: 3,
      fields: { author: ".hnuser", text: ".commtext" }
    }]
  });
  story.comments = comments;
}
return stories; // printed as JSON
```

## Where PandaScript fits

- **Interactive MCP**: `lightpanda mcp` (stdio) serves interactive light
  browsing alongside Playwright (visual). The `browsing` package declares it
  as the `lightpanda` server with a curated tool subset
  (`tools/lightpanda/server.yaml`); a harness may proxy that subset under its
  own tool namespace. The server can trim its own `tools/list` with
  `--tools goto,tree,…` (`lightpanda mcp --list-tools` prints the names).
- **Shared MCP over HTTP**: `lightpanda mcp --port <n>` serves JSON-RPC at
  `/mcp`, one browsing session (page, cookies, storage) per `Mcp-Session-Id`;
  agents that send the same id share a context, and `session_new` /
  `session_list` / `session_close` manage them explicitly.
- **Scripts through MCP**: the MCP-only `save` tool writes the current session
  as a PandaScript and `run` executes one (source or saved path) in the
  caller's session, returning its console output and `return` value — pages
  close when it ends, cookies and storage persist, default timeout 60 s.
- PandaScript (`lightpanda run <script>.js`) is the deterministic,
  replayable extraction format — preferred over interactive MCP calls when
  the extraction shape is known up front and token efficiency matters.

## Version and behaviour notes (1.0.x)

- **1.0.0 (2026-10-02)** left beta; the latest patch release checked is
  **1.0.5** (2026-11-20). Pin a bare-numbered release tag (e.g. `1.0.5`), not
  `nightly`.
- **CORS is enforced by default** since 1.0.0 for page-initiated
  `fetch`/XHR (CDP-driven navigation is unaffected); `--disable-features cors`
  is the escape hatch, not a default.
- **Sub-resources are opt-in** since 0.4.0: iframes, workers and *external
  stylesheets* load only with `--load-resources iframe|worker|stylesheet`
  (repeat the flag). Any check that reads linked CSS text needs
  `--load-resources stylesheet`.
- `--dump png` / `--dump pdf` (and the `screenshot` primitive) render the
  page's **markdown**, not the visual page — they are not screenshots.
- Lightpanda speaks **CDP and WebDriver BiDi** (`serve --protocol`), so
  Puppeteer and Playwright `connectOverCDP` clients can drive it; it remains a
  non-rendering engine, so any client step that needs paint or geometry
  still escalates to a real browser.

Lightpanda is AGPL-3.0, used here as an external subprocess (not linked Go
code), reached via PATH.

## Sources

- https://lightpanda.io/docs/usage/pandascript — PandaScript: `lightpanda run`, top-level `return` output, async `goto`/`evaluate`, Page API, extract schema forms, error list; accessed 2026-11-23
- https://raw.githubusercontent.com/lightpanda-io/browser/main/README.md — agent mode, `/save`, `lightpanda run -`, MCP `--tools`, HTTP transport and sessions, `run`/`save` tools, dump modes, CDP and BiDi; main at 1.1.0-dev, accessed 2026-11-23
- https://lightpanda.io/blog/posts/lightpanda-1-0 — 1.0 out of beta, CORS default, no rendering or real screenshots, `--load-resources` (0.4.0); accessed 2026-11-23
- https://github.com/lightpanda-io/browser/releases/tag/1.0.0 — 1.0.0 breaking changes (CORS default, read-only getComputedStyle, locale) and notable changes; accessed 2026-11-23
- https://github.com/lightpanda-io/browser/releases/tag/0.4.0 — opt-in iframes/workers/stylesheets, BiDi, markdown-rendered png/pdf dumps; accessed 2026-11-23
- https://hub.docker.com/r/lightpanda/browser/tags — release tag list (1.0.0 2026-10-02 … 1.0.5 2026-11-20); accessed 2026-11-23
