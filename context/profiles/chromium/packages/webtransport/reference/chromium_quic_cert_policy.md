# Chromium / WebTransport certificate and handshake policy

Empirically pinned Chromium/WebKit certificate and handshake behavior on
QUIC/HTTP3 and WebTransport — the TCP/QUIC trust divergence, no client certs
over QUIC, the serverCertificateHashes 14-day cap, the short-lived pinned-leaf
pattern, Safari WebTransport requirements, and quic-go / webtransport-go
compatibility.

> Verified against Chromium 151 (chromium-for-testing 151.0.7922.34
> and branded snap 151.0.7922.108) and macOS/iOS Safari 26.5 during
> a 2026-08 WebTransport migration. The QUIC trust gap may be an
> upstream Chromium bug that gets fixed — re-verify on newer Chromium
> versions before assuming either direction.

## TCP/QUIC trust divergence

Chromium applies **different certificate verification on the QUIC/h3
path than on TCP**:

- The QUIC/WebTransport verifier does not honor user-installed NSS CA
  roots — neither `~/.pki/nssdb` nor `/etc/pki/nssdb`. The same chain
  that validates on TCP/h2 fails the WT handshake with
  `ERR_QUIC_PROTOCOL_ERROR.QUIC_TLS_CERTIFICATE_UNKNOWN
  (CERTIFICATE_VERIFY_FAILED)`.
- Public / Chrome-Root-Store CAs verify fine on QUIC in the same
  sessions where an NSS-user CA fails.
- Netlog signature: the handshake aborts in-handshake at
  `handshake.cc:320 OPENSSL_internal:CERTIFICATE_VERIFY_FAILED`, TLS
  alert 46 ("certificate unknown"), and **no `CERT_VERIFIER_JOB` is
  ever created** on the QUIC path.
- No switch rescues it: `--ignore-certificate-errors`,
  `--ignore-certificate-errors-spki-list`,
  `--allow-insecure-localhost`, the OS trust store, profile-dir NSS
  DBs, and `CertificateTrustConfiguration` policy files all fail;
  `--allow-unknown-root-cert` is absent from the build. **Do not
  re-attempt bypass flags** — every flag was exhausted against three
  server configurations (a composed edge, a minimal h3 probe, and a
  production edge).

Practical consequence: an h3/WT server deployed with a self-signed or
private-CA cert is unreachable over HTTP/3 from stock Chromium even
when the TCP side works. Browser-over-QUIC needs either a
publicly-trusted server cert (ACME) or explicit trust distribution
(pinning, below).

**Diagnosis rule:** before assuming a browser-over-QUIC failure is
client-cert related, check netlog for the signature above — the
handshake dies at server-cert verification before the client-cert step
is ever reached. Early incident reports ("headless cannot present
client certs", "Chromium never presents them over QUIC") were both
confounded by this gap.

## Client certificates over QUIC

- Chromium (M151) presents **no TLS client certs during QUIC
  handshakes**. A server CertificateRequest makes the client abort
  with `certificate unknown / CERTIFICATE_VERIFY_FAILED` (quic_error
  199). No cert picker is shown for QUIC, and the per-origin
  client-cert cache is not consulted (it is wired only into TCP
  sockets). Chromium then falls back to TCP/h2, where mTLS works
  normally.
- Upstream: QUICHE added QUIC TLS client-cert support on 2026-05-13
  (commit `7fa9e94d64`) behind restart flag
  `quiche_restart_flag_quic_client_cert_support` (default false), and
  Chromium's net/quic layer has no client-cert plumbing at all.
- **Requesting** (not requiring) client certs over QUIC does not break
  Chromium: a Go `tls.VerifyClientCertIfGiven` h3 listener completes
  Chromium WT sessions cert-less exactly like `NoClientCert` (QUICHE's
  `TlsClientHandshaker::OnClientCertRequested` continues with an empty
  cert list). It still breaks Safari: a live macOS Safari 26.5 retest
  against a `VerifyClientCertIfGiven` h3 listener failed all six WT dials
  pre-CONNECT, so keep h3 listeners at `NoClientCert` until a
  WebKit-capable client proves tolerance.
- WebKit (Safari) has **no client-cert plumbing** in its WebTransport
  stack at all (literal FIXME in
  `Source/WebKit/NetworkProcess/webtransport/cocoa/NetworkTransportSessionCocoa.mm`).
- Headless Chromium cannot present a TLS client certificate on any
  transport; browser mTLS end-to-end tests need a headed browser.

## serverCertificateHashes pinning and the 14-day cap

`new WebTransport(url, {serverCertificateHashes})` bypasses CA
verification and works against an entirely untrusted CA:

- The hash is the **full-DER-cert SHA-256, not SPKI**.
- Chromium **rejects pinned certs whose validity period exceeds 14
  days** (14 days passes, 30 days fails).
- WebKit's cert-hash validator rejects only
  `validityPeriod > twoWeeks` — strictly greater, second precision,
  algorithm string "sha-256", full DER. An exactly-14-day leaf
  marshals to exactly 1209600 whole seconds and passes.

The pattern that closes the private-CA trust gap: the h3 listener serves a
**short-lived (≤14 days) leaf**, rotated before half-life, and the client
fetches the SHA-256/DER pin fresh on every connect attempt from a
client-authenticated TCP leg. Browser WT against a private CA means pins over
a short-lived leaf fetched from the client-authenticated TCP leg — never
system-trust fallback. Long-lived or CA-trusted h3 certs do not work for the
browser path.

## Serving h3 from Go (quic-go)

- **Never wrap the h3 UDP socket in a `net.PacketConn` wrapper.** A
  wrapper disables quic-go's `OOBCapablePacketConn` fast path; on a
  multi-homed host (two public /32s) replies then leave from the
  route-preferred source IP instead of the dialed one, and connected
  client UDP sockets drop them silently. Loopback tests cannot catch
  this (source == destination); only an off-host client can. Keep
  inbound visibility in a qlog tracer instead of wrapping the conn.
- **Alt-Svc and Safari.** CFNetwork never retries 421 (WebKit bug 219939)
  and Safari coalesces Alt-Svc-cached loads onto h3, so cert-less h3
  non-WT responses should be 200 + `Alt-Svc: clear` (a bounded-reload
  bounce page for navigations, a tiny body for fetches) — never 421. WebKit
  resolves WT endpoints by URL (`nw_endpoint_create_url`), not by the
  Alt-Svc map. Advertising h3 via Alt-Svc is safest kept behind an
  explicit opt-in until Safari round-trips are verified.

## Library compatibility and Safari settings

**quic-go v0.60.0 + webtransport-go v0.11.0** is the last verified
Safari-compatible pair. macOS/iOS Safari 26.5 (WebKit nw) cancels the
WebTransport session pre-CONNECT — stream-0 RESET with
`H3_REQUEST_CANCELLED`, zero HEADERS bytes, then `H3_NO_ERROR` connection
close — against every quic-go v0.61 server, while passing v0.60 and
wtransport/quinn. A full bisect rig eliminated SETTINGS shape (including
values — the 2^60 trio passes), all transport params, TLS tickets, cert path,
pinning, source IP, cert-request, Alt-Svc, and QUIC v2. Verdict: v0.61
changed the early 1-RTT flight composition (new_connection_id emission
timing, handshake_done/new_token frame order, keep-alive pings) and nw
abandons the session in response.

- webtransport-go v0.11's Dialer registers an unconditional
  connection-teardown `AfterFunc` that races `WT_CLOSE_SESSION`; removing
  it needs a local fork.
- v0.11 is single-session-per-conn — a client dials one QUIC conn per
  session (multi-session ClientConn arrived in v0.12).
- **Never bump quic-go past v0.60 or webtransport-go past v0.11 without a
  real-Safari 26.5 verification round** (a repro rig of variant listeners
  on a port range plus a variants page).

Safari requires the `WT_INITIAL_MAX_*` SETTINGS trio alongside
`SETTINGS_WT_MAX_SESSIONS`: `STREAMS_UNI` (0x2b64), `STREAMS_BIDI`
(0x2b65), `DATA` (0x2b61). Without them Safari refuses the session
before sending the extended CONNECT — client symptom is a
`WebTransportError` with an EMPTY message ~50 ms after construction,
with infinite retry. Chromium tolerates the omission, so the server
looks healthy until Safari is tested. webtransport-go v0.11 advertises
all four unconditionally; v0.12+ derives the trio from
`*webtransport.Config` fields (zero ⇒ not sent), and hand-injecting
into `H3.AdditionalSettings` cannot work (`Config.addSettings` deletes
the trio before re-adding). Any Safari-facing webtransport-go server
must keep a non-nil `Config` with `MaxIncomingStreams`,
`MaxIncomingUniStreams`, and `MaxIncomingData` all > 0. Guard it with a
regression test that does a real QUIC dial and parses the server SETTINGS
via `(*http3.ClientConn).Settings().Other`. Loopback-only servers dialed
by Go clients are unaffected — the requirement lives in WebKit's client,
not in webtransport-go.

`WebTransport.datagrams` in Safari 26.4+ is mixed-shape: a
`.readable` property plus a `createWritable()` method, no `.writable`
— a client adapter must detect per-direction.

## Diagnosis signals and local recipes

- Decisive server-side signal for Safari WT attempts: the
  `quic connection created` line (quic-go qlog tracer).
  `connection_closed {application_error: 0x100}` is the
  cancel-before-CONNECT signature.
- A qlog directory for a systemd unit with `PrivateTmp` must not be under
  /var/tmp; use a path under /var/lib.
- NSS trust for local CAs: `certutil -A -d sql:~/.pki/nssdb -t
  "CT,C,C"` with `HOME` pointing at the profile root;
  `update-ca-certificates` alone is not enough for Chromium on Linux.
- Chromium never attempts QUIC to a bare loopback-IP origin without
  prior Alt-Svc or `--origin-to-force-quic-on`; map a hostname with
  `--host-resolver-rules` for realistic testing.
- The `--auto-select-certificate-for-urls` switch no longer exists in
  M151; use the `AutoSelectCertificateForUrls` managed policy.

## Related

- <ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::tls_keys@chromium/reference/webcrypto_keys_and_tls.md" />
  — why in-browser-generated keys never reach the TLS client-auth stack.
- <ctxmod_link path="ctx:github.com/ctxmod/browser/context::shared::policy_research@chromium/reference/chrome_policy_sources.md" />
  — where to research Chrome policies such as `AutoSelectCertificateForUrls`.
