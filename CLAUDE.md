# CLAUDE.md — decision_dash

This file provides guidance to Claude Code when working with code in this repository.

## What this is

A decision-first trading dashboard: the ground-up rebuild of `trading_dash`'s
presentation layer around one principle — **verdict first, with the trigger level
and the invalidation level, evidence on demand**. `DESIGN.md` is the design
rationale and cut list; `mockup.html` is the visual reference. Read both before
building anything.

**This repo contains NO Worker code.** It is a frontend-only sibling of
`C:\dev\trading_dash` and consumes the same deployed Cloudflare Worker
(`stock-research-worker`). The two dashboards run side by side against the same
KV and crons; the old one stays live as the fallback and nothing done here may
break it.

## Relationship to trading_dash — read this first

- **The Worker lives in `C:\dev\trading_dash` and is edited THERE, under that
  repo's CLAUDE.md rules.** Any Worker change this project needs — a new endpoint,
  an `ALLOWED_ORIGINS` entry, a `CORS_ALLOW_HEADERS` entry — is a trading_dash
  task, committed in that repo. Amber deploys the Worker manually
  (`npx wrangler deploy`); **never run or assume a deploy**.
- **trading_dash's `CLAUDE.md` and `ARCHITECTURE.md` are authoritative** for
  endpoint behavior, KV keys, subrequest budgets, and every named failure mode.
  Do not re-derive or assume endpoint behavior — read those docs. When Amber's
  memory conflicts with them, the docs win and you should say so.
- **First-time setup dependency:** this site's GitHub Pages origin must be added
  to `ALLOWED_ORIGINS` in trading_dash's `worker.js` and deployed before any page
  here can call the API. Until then every fetch fails CORS — that is the expected
  state, not a bug here.

## The design contract

These decisions are settled (rationale in `DESIGN.md`). Do not relitigate them
silently; flag if a task conflicts.

1. **Surfaces:** four tabs — **Today** (regime line · action queue · levels watch
   · Radar · session-brief timeline · calendar · your-movers · sector heat strip),
   **Names**, **Options**, **Income** — plus the per-ticker page (hero decision
   block + 4 collapsed evidence groups: Technicals / Positioning / Street & story
   / Record). No Midday tab, no momentum scanner, no IPO calendar, no market-wide
   ≥10% movers, no sector prose on the main surface.
2. **The action queue is time-aware.** Morning = the plan; intraday = triggered /
   approaching / invalidated; post-close = what resolved + tomorrow. Each card:
   ticker · action · trigger · structure · invalidation · one-clause why · source
   + as-of. Assembled **client-side from payloads the page already fetches**
   (analysis records, long batch, econ calendar, movers). A server-side
   `/api/queue` is a later trading_dash task if warranted.
3. **One verdict store.** One ticker = one verdict record per trading day
   (`analysis:{TICKER}`). Every surface renders THAT record with its as-of. The
   ticker page revalidates and writes back to the same key; Names re-reads on tab
   focus and on the 30-second staleness sweep. **Two surfaces disagreeing on a
   verdict is a provenance bug, not cosmetics** — this rule exists because the old
   dashboard's watchlist and ticker page could disagree on the same day.
4. **Earnings timing is a first-class input.** BMO → the hold/exit deadline was
   the *prior* session's close and the report day is reaction-only. AMC → the
   deadline is that day's close. Timing classified from Yahoo
   `earningsTimestamp`/`earningsTimestampStart` (04:00–09:30 ET = BMO, post-16:00
   = AMC); unknown → assume the earlier deadline **and label the assumption**.
   Estimated dates render `est.` and any gate working from an estimate says so.
5. **Radar** (off-watchlist discovery): universe = S&P 500 + Nasdaq 100; gates =
   market cap > $10B, price > $20, listed options with real OI, average
   dollar-volume floor; **max 5 candidates/day**, each with a one-line why and
   one-click add-to-watchlist. Feeds the same verdict pipeline once adopted.
   Requires a new Worker endpoint (trading_dash task).
6. **Income is deliberately quiet.** Quarterly cadence; it surfaces in Today only
   on events: ex-div ≤ 7d, a cut, payout > 90%, or price entering a stated add
   zone. Tax character flagged per name (qualified vs ordinary). Requires a
   Worker endpoint for the batch (trading_dash task).
7. **Click-through is universal.** Any ticker → its ticker page. An option rec →
   Options with that name's row expanded. Hash deep-links (`#ticker/NVDA`,
   `#options/NVDA`) are the mechanism.
8. **Methodology prose never sits above data.** Legends live in a `?` modal.

## Honesty rules (inherited from trading_dash — non-negotiable)

The full set with their incident histories is in trading_dash's docs. The ones a
frontend can violate, condensed; every one of these shipped broken over there at
least once:

1. Never display one quantity under another's label (HV is not IV).
2. Never present a hardcoded or generated number as computed. An empty cell
   renders `—` with a reason on hover — a wrong number is more expensive than a
   blank.
3. When a source is unavailable, render unavailable with a reason — never a
   fallback value. A plausible stand-in is indistinguishable from the real thing.
4. Every displayed number carries a source and an as-of. Badges are rendered from
   the `_meta` a fetch returned, **never hand-written in markup**. Staleness
   re-evaluates on a 30-second timer; `delayed` (property of the source) and
   `stale` (property of our copy) are different states and a badge can be both.
5. Model confidence renders ordinal (Low / Moderate / High), never a percentage.
   Numbers scored against realized outcomes (Brier) stay numeric.
6. **`x == null` and `x === 0` must never render the same way.** Guard the null
   before the arithmetic — `(null * 100).toFixed(0)` is `"0"`, a fabricated
   measurement. **A rendered bar is a rendered number too** (a null divided to 0
   and clamped to a bar floor draws a real bar for a missing measurement).
7. **A refused measurement is a finding, not an absence.** Coverage that declines
   to publish renders dim and neutral, naming its own numbers — never a blank.
   `return ''` in a render helper is where this hides: ask whether it withholds a
   *control* (fine) or a *fact* (bug).
8. No hit rate, win rate or accuracy figure renders without its base rate beside
   it with the signed edge — not in a tooltip, beside the number. A rate below
   its base rate must never drive ranking or selection.
9. **The frontend is ALWAYS newer than the Worker for a while.** GitHub Pages
   deploys on push; the Worker deploys manually. Handle a field the deployed
   Worker doesn't send yet as its own named state (`field-absent`), not a falsy
   early-return that paints nothing. Test new field consumption against the
   *currently deployed* Worker before pushing.
10. Sparklines that suppress the zero baseline label min and max on the axis as
    required elements.

## Worker interface facts (learned the hard way over there)

- `API_BASE` and `DASH_KEY` sit at the top of each page's script block. `DASH_KEY`
  is sent as `x-dash-key`. **Any NEW custom request header is a two-repo change**:
  it must be added to `CORS_ALLOW_HEADERS` in trading_dash's `worker.js` or the
  browser silently blocks the request at preflight — nothing arrives at the
  Worker, nothing appears in its logs.
- **`file://` is rejected** (Origin: null). Test over http:
  `npx http-server <explicit path> -p 8123` — `http://localhost:*` and
  `http://127.0.0.1:*` are allowlisted. Start it with an explicit path, never
  `.`, and confirm the served byte count matches the file on disk before
  debugging the page.
- **An edit is live only after `git push`.** Pages serves the last pushed commit.
  When a change doesn't take: curl the live HTML and grep for it, then confirm
  the *browser* is running that bundle (assert a new identifier in the page —
  curl bypasses the browser cache).
- Batch endpoints (`/api/watchlist/batch`, `/api/long/batch`) are KV reads — one
  per symbol. Cheap, not free; don't add polling loops.
- **`/api/watchlist/batch` SILENTLY TRUNCATES at 30 symbols** (`.slice(0, 30)` in
  `handleWatchlistBatch`) and **requires `?symbols=`** — it does not read the
  saved list. A 39-name watchlist requested in one call returns 30 names, no
  error, no flag: a quietly short table that looks complete. **Chunk at 15**
  (what trading_dash does) **and assert the returned union against what you
  asked for**, rendering the difference as a named state. Found in phase 1;
  `/api/long/batch` caps at 60 (`LONG_MAX_SYMBOLS`) and needs no chunking today.
- `/api/long/batch` is a **cache read, not a computation**: uncached tickers come
  back in a top-level `missing[]` array with a per-symbol `not-loaded` reason,
  not in `rows`. In practice almost every row is not-loaded until someone expands
  it. "Not loaded" is a load state and must not render like "no trade here".
- The watchlist batch carries **no `earningsTimestamp`** — only a preformatted
  `earningsDate` string and an integer `daysToEarnings`. BMO/AMC therefore cannot
  be derived on this surface; render no timing tag rather than inferring one.
  Adding the timestamp is a trading_dash task.
- **`daysToEarnings` goes negative** when the stored date is behind the report
  (seen at −13). Any "earnings within N days" filter needs a lower bound of 0 or
  it pulls stale past-dated rows into the urgent group.
- **`recRank` is `+confidence` for BUY, `0` for HOLD, `−confidence` for SELL.**
  `0` is a real rank and `null` is the absence of a record; sorting them together
  buries a no-record row among the HOLDs.
- Button-triggered AI endpoints (earnings analysis) are **never** wired to page
  load — they spend Claude calls against the daily ceiling.
- A single negative probe right after a Worker deploy is UNCONFIRMED — stale
  isolates serve pre-deploy code for ~a minute. Re-probe before concluding.

## Queue clock facts (phase 3)

- **The action queue is a pure function of the shared store plus a PT clock**
  (`buildQueue()`), and must stay one — that property is what makes the
  selection hand-checkable against the table `dumpQueueTable()` prints. No
  fetching, no hidden inputs.
- **`?clock=<ISO>` freezes the queue's clock** for driving all three phases in
  verification. It renders a red ⏱ TEST CLOCK chip whenever active and touches
  nothing else — staleness badges stay on the real clock, because our payload
  copies really are as old as they are.
- **KNOWN GAP: the frontend has no NYSE holiday calendar.** Phase and
  next-trading-day use a weekday rule only, so a market holiday reads as a
  trading day here (measured: Thanksgiving 2026-11-26 reads `intraday`). The
  Worker's `NYSE_HOLIDAYS` table is not published on any endpoint this page
  fetches; shipping it on one is a trading_dash task if the gap starts to bite.
  The phase line and queue footnote label the derivation.
- **BMO's deadline is the PRIOR trading day's close; AMC's is the report day's
  own close** (13:00 PT = 16:00 ET). `unknown` and field-absent both assume the
  earlier (BMO) deadline and say so on the card — with different wording,
  because "the Worker could not classify" and "an older Worker never sent the
  field" are different facts.
- **No trigger times, ever.** The page holds one 15-minute-delayed snapshot,
  not an intraday tape: whether a level was touched between fetches is
  unknowable here. State chips render the CURRENT price↔level relationship
  with its as-of (AT/THROUGH · APPROACHING · INTACT). The mockup's
  "TRIGGERED 10:42" is fiction this page refuses to produce.

## Design system

Same tokens as trading_dash. CSS custom properties in `:root`; never hardcode a
hex. `--bull #23d18b` / `--bear #f25f5c` / `--amber #f4b740` (neutral + stale) /
`--cyan #5ec5ea` (data accent) / `--violet #b48ead` / `--bg-0..3` / `--ink-0..3`.
Fonts: Fraunces (display serif), Geist (body), JetBrains Mono (numbers/labels).
Status is never conveyed by color alone.

## Build plan

Phase 0 must complete before anything else; later phases are ordered by
dependency, not priority.

| phase | what | depends on |
|---|---|---|
| 0 | Scaffold: repo, GitHub Pages, origin allowlisted in the Worker, a probe page proving a live `/api/market/snapshot` round-trip in a browser | ALLOWED_ORIGINS edit + manual deploy (trading_dash) |
| 1 | **Names** — 6-column table on `/api/watchlist/batch`, attention sort, expanders, one-verdict-store rendering, click-throughs | 0 |
| 2 | **Today** static modules — regime line, session-brief timeline, calendar (BMO/AMC/est. tags), levels watch, your-movers, sector heat strip | 0 |
| 3 | **Action queue** — client-side assembly, phase awareness (morning/intraday/post-close), deadline logic | 1, 2 |
| 4 | **Options** — verdict layer over `/api/long/batch` + `/api/long/:ticker`, lane verdicts, legend modal, deep links | 1 |
| 5 | **Ticker page** — hero decision block + 4 evidence groups; base-rate-beside-rate on the Record group | 1, 4 |
| 6 | **Radar** — needs `/api/radar` built in trading_dash first | Worker task |
| 7 | **Income** — needs `/api/income/batch` built in trading_dash first | Worker task |

## Working pattern

- Windows PowerShell 5.1; project at `C:\dev\decision_dash`; not VS Code.
- Claude Code auto-commits and auto-pushes at task completion. One commit per
  logical task. Never force-push, never rewrite published history, never commit
  secrets. Hold the commit when a task ends broken, unverified, or on an open
  question.
- Worker deploys are Amber's, manual, from trading_dash. Never run
  `npx wrangler deploy`.
- Kill background processes (http-server) when a task completes.

## Verification standard

Same bar as trading_dash, abbreviated:

- Print, don't assert — computed beside expected, with deviations. "Verified"
  without a number is not verification.
- **A newly rendered figure gets eyes on it in a browser before the commit is
  done.** A value that is only wrong at the render layer (unit slip inside a
  `<td>`, percent vs fraction, wrong cell) is unreachable by any test that stops
  at the JSON.
- An empty comparison is not a pass — assert a non-zero population on both sides
  and print the count beside the verdict.
- Name the verification method's blind spot (curl cannot see CORS preflights or
  the browser cache; a DOM shim cannot see layout).
- Verify against a second case; one passing ticker is a coincidence.

## Before / after every task

Read `CLAUDE.md` and `DESIGN.md` first; do not work from assumptions carried from
prompts or prior sessions — if an instruction contradicts the code or these docs,
say so before acting. Update the docs in the same task that changes what they
describe. Report what you could not verify, separately and explicitly. When a bug
is found, add a rule here naming the specific failure that produced it.
