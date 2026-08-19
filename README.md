# decision_dash

Decision-first trading dashboard — the ground-up rebuild of
[trading_dash](https://github.com/ambermlysak/trading_dash)'s presentation layer.
Verdict first, with the trigger and invalidation levels; evidence on demand.

- `index.html` — the app: header + four hash-routed tabs (`#today`, `#names`,
  `#options`, `#income`). Names is built; the rest are placeholders naming their
  phase.
- `probe.html` — the CORS canary. One live `/market/snapshot` round-trip with its
  provenance. When the app goes dark, this separates "the Worker or CORS broke"
  from "our rendering broke" in a single page load. Keep it working.
- `DESIGN.md` — the full design rationale, cut list, and round-2 decisions
- `mockup.html` — the interactive design reference (dummy data; open in a browser)
- `CLAUDE.md` — working rules; read its design contract before building

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
site's Pages origin must be present in the Worker's `ALLOWED_ORIGINS` (a
trading_dash change) before API calls succeed.
