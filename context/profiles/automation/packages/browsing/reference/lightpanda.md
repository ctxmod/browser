# Lightpanda PandaScript

Lightpanda PandaScript — deterministic, LLM-free web extraction in a dedicated V8 context via the Page object API (goto/extract/evaluate); sync agent primitives, $LP_* credential placeholders, list-to-detail pattern.

A PandaScript is a web automation script run directly by the Lightpanda
browser — no Node/Python, no Puppeteer/Playwright, no CDP serialization
cost, no LLM required: vanilla JavaScript plus a small set of native
primitives. It is reproducible, deterministic, and token-free.

Run with:

```
lightpanda agent <script>.js
```

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
  one script run; a later `lightpanda agent script.js` starts fresh. Page
  scripts cannot see agent variables, and vice versa.
- **Completion value is printed automatically** (objects/arrays as JSON).
  End the script with the bare expression you want as output. Use
  `console.log(...)` only for debug output (it does not JSON-format objects).

## Async form (resolved)

Two equivalent surfaces exist; the canonical scripts use the **`Page`
object API**, which compiles as a module and **does allow top-level
`await`**:

```js
const page = new Page();
await page.goto("https://example.com");
const title = page.evaluate("document.title");
title; // printed automatically
```

The **standalone agent-context primitives** path (bare `extract(...)`,
`evaluate(...)` with no `Page`) is synchronous and blocking: top-level
`await` is a `SyntaxError` there, and `.then` callbacks only run after the
script body finishes. Prefer the `Page` object form for navigation flows.

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
data; // -> { title, stories: [...] }
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
const title = page.evaluate("document.title");
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
- `no page loaded - run goto(url) first` — a page primitive ran before
  navigation.
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
stories; // printed automatically as JSON
```

## Where PandaScript fits

- **Interactive MCP**: `lightpanda mcp` (stdio) serves interactive light
  browsing alongside Playwright (visual). The `browsing` package declares it
  as the `lightpanda` server with a curated tool subset
  (`tools/lightpanda/server.yaml`); a harness may proxy that subset under its
  own tool namespace.
- PandaScript (`lightpanda agent <script>.js`) is the deterministic,
  replayable extraction format — preferred over interactive MCP calls when
  the extraction shape is known up front and token efficiency matters.

Lightpanda is AGPL-3.0, used here as an external subprocess (not linked Go
code), reached via PATH. Pin release tags (bare-numbered, e.g. `0.3.3`).

