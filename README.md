# decision_dash

Decision-first trading dashboard — the ground-up rebuild of
[trading_dash](https://github.com/ambermlysak/trading_dash)'s presentation layer.
Verdict first, with the trigger and invalidation levels; evidence on demand.

**Live: <https://ambermlysak.github.io/decision_dash/>** — all phases shipped.

- `index.html` — the app. Hash-routed:
  - `#today` — regime line · action queue · levels watch · Radar · your movers ·
    sector heat · session brief (every brief published so far today — 06:00
    open, 11:30 pulse, 13:15 close — stacked, each with its own as-of) · calendar
  - `#names` — the 6-column watchlist table, attention-sorted
  - `#options` — lanes A–F over the long screen; `#options/NVDA` deep-links a row
  - `#income` — **Diversify**: categorized sleeves (income / cyclical / value /
    defensive), no AI verdicts
  - `#ticker/NVDA` — the per-ticker page: hero decision block + 4 evidence groups
- `probe.html` — the CORS canary. One live `/market/snapshot` round-trip with its
  provenance. When the app goes dark, this separates "the Worker or CORS broke"
  from "our rendering broke" in a single page load. Keep it working.
- `CLAUDE.md` — the as-built design contract, the honesty rules, the Worker
  interface facts, and the known-gaps list. **Read this first.**
- `DESIGN.md` — why the rebuild looks like this: the diagnosis, the cut list and
  the round-2 decisions, annotated where as-built diverged.
- `mockup.html` — the **historical** visual reference (dummy data). It predates
  the build and disagrees with it in places; `CLAUDE.md` lists where and why.

## Architecture

Frontend-only. Consumes the same deployed Cloudflare Worker as trading_dash
(`stock-research-worker`) — no Worker code lives here. Worker changes are made in
the trading_dash repo and deployed manually.

## Quick start

```powershell
npx http-server C:\dev\decision_dash -p 8123
# http://localhost:8123 — localhost is allowlisted by the Worker
```

Pages deploy: push to `main`; GitHub Pages serves the last pushed commit. The
site's Pages origin is present in the Worker's `ALLOWED_ORIGINS` — the entry is
per-origin, so `https://ambermlysak.github.io` was already allowlisted from
trading_dash and needed no change.

**Nothing on any page spends a Claude call on load, with one exception:** the
ticker page generates a verdict when the store holds no record for the current
trading day. It attempts that at most once per ticker per page session, never
retries a failure automatically, and never spends when the store read itself
failed. `refresh verdict` (which always regenerates) and `Analyze earnings` stay
click-only and are labelled as spending.
