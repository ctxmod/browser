# Chrome/Chromium policy research sources

Verified retrieval sources for Chrome/Chromium policy and documentation
research: `chromeenterprise.google/policies` is a JS SPA with no extractable
text, the canonical policy definitions live as YAML in the Chromium source
mirror, Edge's per-policy pages mirror them verbatim server-side, the
DuckDuckGo HTML endpoint is a working search fallback, and
issues.chromium.org, chromestatus.com and Stack Overflow are blocked or empty
to text fetchers.

Verified 2026-08-20 during a Chrome-extension research phase. These sources
cost several wasted fetches to discover; any future session researching
Chrome policies, crbugs, or chromestatus should go straight to them.

## Canonical policy text: the Chromium source mirror

`chromeenterprise.google/policies/*` (any URL shape, including the
newer `/policies/<policy-slug>/` form) is a JS SPA — a text fetcher returns
only the newsletter-form shell. The canonical policy text lives in the
Chromium tree at
`components/policy/resources/templates/policy_definitions/**/*.yaml`,
fetched raw from the `chromium/chromium` GitHub mirror:

```
raw.githubusercontent.com/chromium/chromium/main/<path>
```

The YAML path is the most authoritative source — it IS the source of
the rendered docs. Notes on the mirror:

- GitHub API file reads on `chromium/chromium` work for files < ~1 MB;
  directory listings also work.
- `chrome/browser/policy/` does NOT contain the definitions YAML — it
  is under `components/policy/`, grouped by category directory, e.g.
  `policy_definitions/ContentSettings/AutoSelectCertificateForUrls.yaml`
  (re-verified 2026-11-23; a policy is not always where its name suggests,
  so list the category directories rather than guessing).
- `chrome/VERSION` on the same mirror gives the trunk milestone (MAJOR=160 on
  2026-11-23) — a quick extractable check of how far trunk is ahead of the
  stable milestone a policy note was written against.

## Cross-check mirror: Microsoft Edge per-policy pages

The Microsoft Edge per-policy pages
(`learn.microsoft.com/en-us/deployedge/microsoft-edge-policies/<policy>`)
are a server-rendered verbatim mirror of the policy definitions —
useful for cross-checking a policy's semantics and support matrix
without touching the SPA.

## Policy changes per release

The Chrome browser Enterprise and Education release notes — the per-milestone
list of new, changed and deprecated policies — moved off
`support.google.com/chrome/a/answer/7679408`, which now carries ChromeOS notes
plus links to past browser releases (newest listed on 2026-11-23: Chrome 154,
2026-09-23). The browser notes now live at
`chromeenterprise.google/intl/en_en/resources/release-notes/`, which returns
only a template shell to a text fetcher (an unfilled `__RELEASE_DATE__`
placeholder, no entries). For an extractable change feed, diff the policy
definitions YAML between milestone branches on the source mirror instead.

## Search fallback: DDG HTML endpoint

The DuckDuckGo JSON API can fail for a whole session with HTTP 202.
`https://html.duckduckgo.com/html/?q=<query>` fetched as a web page
works as the search fallback and returns titles + snippets.

## Blocked or empty sources

- `issues.chromium.org` and `chromestatus.com` are JS SPAs with
  near-empty extraction.
- `chromestatuslite.com/feature/<id>` mirrors feature status
  server-side.
- Stack Overflow 403s even after browser fallback — DDG-HTML snippets
  are the practical route.

## developer.chrome.com slug churn

Slugs move often. Current placements as of 2026-08:

- Service-worker lifecycle:
  `develop/concepts/service-workers/lifecycle` (plural).
- CSP: `reference/manifest/content-security-policy`.
- Self-hosting: `how-to/distribute/host-extensions`.

Fetch `/docs/extensions/how-to` first to discover current slugs rather
than trusting a remembered path.

## Where researched policies land

Snap-packaged Chromium reads managed policies from
`/etc/chromium-browser/policies/managed/` only, and
`AutoSelectCertificateForUrls` is the policy that governs UI-less
client-cert fetches.

## Sources

- https://raw.githubusercontent.com/chromium/chromium/main/components/policy/resources/templates/policy_definitions/ContentSettings/AutoSelectCertificateForUrls.yaml — canonical policy YAML path, fetched 2026-11-23
- https://raw.githubusercontent.com/chromium/chromium/main/chrome/VERSION — trunk MAJOR=160, fetched 2026-11-23
- https://support.google.com/chrome/a/answer/7679408 — release notes moved notice; past browser releases listed through Chrome 154 (2026-09-23); accessed 2026-11-23
- https://chromeenterprise.google/intl/en_en/resources/release-notes/ — current release-notes location; template shell to text fetchers; accessed 2026-11-23
