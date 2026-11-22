# Browser

Browser automation, web validation and Chromium platform facts for ctxmod (`github.com/ctxmod/browser/context`). Requires `github.com/ctxmod/design/context`.

Profiles:

- `automation` (default; extends the design module's `web` profile): packages `browsing` (Lightpanda and PandaScript, rendering models, the `lightpanda` MCP server, the `browser_procedure` agent), `web_validation` (validation references, Playwright WebKit facts, the `frontend_validation` skill), `scraping` (the `web_scraping` skill), and the opt-in `browsing_agent` (procedure-depth instructions for `browser_procedure`).
- `chromium`: versioned Chromium/WebKit platform facts — packages `tls_keys` (WebCrypto keys and TLS client auth), `webtransport` (QUIC/WebTransport certificate policy, quic-go/webtransport-go compatibility), `policy_research` (Chrome policy research sources).

Select a profile by its qualified name, e.g. `github.com/ctxmod/browser/context/profiles/chromium`.
