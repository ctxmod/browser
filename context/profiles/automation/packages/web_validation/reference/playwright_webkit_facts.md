# Playwright WebKit facts

Verified Playwright WebKit behaviour that looks CI-only and costs a wrong
diagnosis before it is settled.

## WebKit video last-frame freeze

Under concurrent load (CI runners, or 4 pages at once locally),
Playwright's WebKit can stop a muted autoplaying `<video>` at
`currentTime` ≈ duration − 0.0007 with `paused === false` and
`ended === false`, and **never fires `ended`**. A single local run never
shows it, so the bug looks CI-only. (Found 2026-10-31 in a UI that
auto-sequences clips by advancing on `ended`; a "pause near the end
counts as the end" heuristic did NOT fix it — no pause event fires at
all.)

**How to apply:**

- Reproduce with a probe that opens 4 pages concurrently and logs media
  events (`play`/`waiting`/`suspend`/`pause`/`ended`).
- Fix: on `timeupdate` within ~0.3 s of the end, set a timer for
  (remaining + 0.25) s; if the element is still unpaused and within
  0.05 s of duration, report the end. Emit `ended` once per play-through;
  clear the timer on seek and unmount.
- Validate with `--repeat-each=10 --project=webkit` plus the concurrent
  probe (4/16 stalls before the fix, 0/32 after).
