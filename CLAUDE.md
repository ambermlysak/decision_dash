# CLAUDE.md — decision_dash

This file provides guidance to Claude Code when working with code in this repository.

## What this is

A decision-first trading dashboard: the ground-up rebuild of `trading_dash`'s
presentation layer around one principle — **verdict first, with the trigger level
and the invalidation level, evidence on demand**. `DESIGN.md` is the design
rationale and cut list.

**Status: all phases 0–7 are SHIPPED.** The dashboard is feature-complete against
the design contract and live at
<https://ambermlysak.github.io/decision_dash/>. See the build plan below for the
commit of each phase.

**`mockup.html` is the HISTORICAL design reference, not a spec.** It was the
visual argument for the rebuild and it diverges from as-built wherever reality
won. Read it for layout intent; do not treat a disagreement with the running app
as a bug in the app. The known divergences:

| mockup shows | as-built | why |
|---|---|---|
| queue chip `TRIGGERED 10:42` | `AT/THROUGH LEVEL` · `APPROACHING` · `INTACT`, with an as-of | we hold ONE 15-min-delayed snapshot, not an intraday tape. Whether a level was touched between fetches is unknowable here, so a trigger *time* is fiction. |
| an **Income** tab | **Diversify** | the sleeve is categorized, not income-only |
| `ADD` / `HOLD` verdict badges on the sleeve | no verdicts at all | no AI touches the Diversify path in v1 |
| a per-name qualified/ordinary tax tag | omitted, with the Worker's reason on hover | not derivable — see the known gaps |

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
   · Radar · session-brief stack + timeline · calendar · your-movers · sector heat strip),
   **Names**, **Options**, **Diversify** — plus the per-ticker page (hero decision
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
   deadline is that day's close. **As built**, the Worker classifies from
   `calendarEvents.earnings.earningsDate` (04:00–09:30 ET = BMO, ≥16:00 ET = AMC)
   and ships `earningsTs` / `earningsSession` / `earningsIsEstimate` on every
   watchlist row; `unknown` → assume the earlier deadline **and label the
   assumption**. Estimated dates render `est.`
5. **Radar** (off-watchlist discovery): gates = market cap > $10B, price > $20,
   listed options with real OI, average dollar-volume floor; **max 5
   candidates/day** (3 mover + 2 sector slots), each with a mechanical one-line
   why and one-click add-to-watchlist. Adoption grows the sweep universe BOTH
   dashboards share. **As built**, `/api/radar` sources are Yahoo screeners
   (`day_gainers`, `most_actives`) plus the banked sector picks, ranked by rvol —
   not the S&P 500 + Nasdaq 100 sweep this document originally specified.
6. **Diversify is deliberately quiet** (was "Income"). Categorized sleeves —
   `income` | `cyclical` | `value` | `defensive` — with the category **assigned by
   the user** on each `income:tickers` entry and **never inferred**. It surfaces
   in Today only on events: ex-div ≤ 7d, a cut, payout > 90%, or price entering a
   stated add zone; it never feeds the action queue. **v1 ships NO AI verdicts** —
   no rating, no call, no badge, because no AI touches this path. The coverage
   strip is **counts** labelled *"list coverage, not portfolio weights — position
   sizes unknown"*, never percentages: this app has no position sizes, so a
   share-of-portfolio figure would measure nothing.
7. **Click-through is universal, and internal.** Any ticker → its ticker page. An
   option rec → Options with that name's row expanded. Hash deep-links
   (`#ticker/NVDA`, `#options/NVDA`) are the mechanism. **The interim
   old-terminal links are retired** — Names, the queue, the Today modules, the
   Radar rows, the Diversify rows and the Options rows all route internally.
   **Exactly ONE deliberate escape hatch survives**, on the ticker page: *"open X
   in the old terminal ↗"*, for the cards phase 5 did not carry over (chart
   patterns, the V/OI table, the projection overlay). Do not reintroduce others;
   do not remove that one.
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
- **AI spend on load — the rule and its ONE exception (amended phase 9).** The
  rule was "nothing on any page spends a Claude call on load". It is now:
  **nothing spends on load, EXCEPT the ticker page, which generates a verdict
  when the store holds no record for the current trading day.** That one path is
  `POST /api/ai/synthesis/:ticker` from `maybeAutoVerdict()`. Everything else —
  `/api/earnings`, `/market/sectors` uncached — stays button-only and is never
  wired to load. The exception is bounded by four guardrails, all tested:
  one attempt per ticker per page session (`verdictAutoTried`), one in-flight
  generation per ticker (`verdictJobs` — rapid re-loads join it rather than
  spending again), never on an unknown state (a network or 5xx failure on the
  store read renders the error and spends nothing), and no auto-retry after a
  failure — the button is the retry path.
- **`POST /api/ai/synthesis/:ticker` works for OFF-WATCHLIST tickers** and writes
  `analysis:{TICKER}`. Measured 2026-08-19 on COST (404 before, 200 after,
  ~20s). It returns `{ticker, type, analysis:{…}, ts, _meta}` — the record is
  nested under `.analysis` and its `ts` is **null there**; only the envelope and
  the stored record carry the timestamp, so the caller must re-read
  `/api/analysis/:ticker` rather than trust the POST body. `_meta.ttlSeconds` is
  172800 (48h), so a record outlives its trading day: **"not expired" and "is
  today's" are different questions** and the design contract asks the second one.
- A single negative probe right after a Worker deploy is UNCONFIRMED — stale
  isolates serve pre-deploy code for ~a minute. Re-probe before concluding.
- **`/api/short/:ticker` returns `periods` NEWEST-FIRST.** Reading it as
  oldest-first inverted the direction word: the Positioning header rendered
  "Shorts rising" beside a line reading "−9.7% vs prior", the label and the
  number contradicting each other on one screen. **Sort on `settlementDate`;
  never trust array order for a time series.** Found in phase 5.
- **`/api/daily` carries THREE narrative records in ONE per-PT-date object**, and
  a re-run of the briefing job **REPLACES THAT OBJECT WHOLE**. The slots are the
  06:00 open (top-level `headline` + `open{}`), the 11:30 pulse (`midday{}` —
  its body field is **`narrative`**, not `body`) and the 13:15 close (`eod{}` —
  `body`, plus `complete`). Only the close carries `complete`; each slot carries
  its own `_instr`. **Measured 2026-08-19: at 17:56 PT the payload served open
  06:02 / midday 11:31 / eod 13:13; at 18:11 PT a fresh `_instr.phase: briefing`
  run had rewritten it to open 18:11 / eod 18:11 with `midday` GONE.** So the
  slots do not simply accumulate to the rollover — an earlier one can be dropped
  by a later write. Making the Worker merge instead of replace is a trading_dash
  task. Phase 8's frontend answer is a per-slot bank in `localStorage`
  (`dd:brief-slots`) — see the session-brief rules below.
- **The verdict store holds TWO record shapes and they must render identically.**
  The nightly batch pass writes `{rating, confidence, recommendation, drivers[],
  summary, ts}`; `/api/ai/synthesis` writes `{rating, confidence, action,
  factors{}, thesis, trend, pattern, summary, ts}` — **no `recommendation`, no
  `drivers`**. Measured 2026-08-19: NVDA is the first shape, KO and a freshly
  generated COST the second. Every surface read only `recommendation`, so a
  verdict generated by the button rendered "the record carries a rating but no
  one-line call" while carrying one under `action`. Read both through
  `verdictCall()` / `verdictDrivers()`; never read the raw field.
- `/api/analysis/:ticker` **404s for any ticker with no stored record** (every
  off-watchlist name) and carries **no `_meta`** — just
  `rating/confidence/recommendation/drivers/summary/ts`. A 404 here is the
  normal off-watchlist state, not a failure, and must never trigger generation.
- `/api/track/:ticker` can return `calibration.byRating: null` outright (seen on
  SMR) — guard it before indexing, and render the payload's own resolution
  reason instead of empty tiles.
- **A hash route that reads the shared store must start the shared sweep.**
  Deep-linking straight to `#ticker/NVDA` rendered a watchlist name as
  off-watchlist, with its levels and session tag missing, because only
  `#today`/`#names`/`#options` triggered `loadShared()`.
- **`/api/radar` is banked once per PT date** (`ttlSeconds` ~36h, `cached: true`,
  `ptDate`). Render its banked time; do not refetch on a tab switch — a discovery
  list that changed every time you looked at it would not be the day's list. The
  envelope carries `refused`/`reason`, `complete`, `sources[]` (with per-source
  `ok`/`reason`), `funnel{}`, `gates{}` and `considered`; each is its own named
  state and `complete: false` renders the candidates that DID clear alongside the
  named down source.
- **`/api/watchlist/save` and `/api/income/save` REPLACE the whole list.** Every
  mutation must re-read the server copy at click time and edit that — never a
  locally-held snapshot, which silently drops anything changed elsewhere since the
  fetch. Both Workers snapshot the pre-write value (`watchlist:prev` /
  `income:prev`) and log a shrink past 30%, which makes a clobber recoverable but
  does not prevent it. Proven in phase 6 by corrupting the local list to 3 names
  and confirming the outgoing POST still carried 40.
- **`income:tickers` is a DIFFERENT list from `watchlist:tickers`** and holds
  `{ ticker, addBelow, category }` entries. `/api/income/save` allowlists exactly
  those three fields and strips the rest. The category round-trips verbatim; the
  batch row reports `categorySource` / `addBelowSource` so a rendered category can
  be traced to the saved entry.
- **`exDivIsPast` is load-bearing on income rows.** Yahoo's `exDividendDate` is
  routinely the *most recent past* one rather than the next, so a past date must
  render "last <date>", never as upcoming. Funds publish no ex-div date at all and
  the next is deliberately NOT projected from cadence — a projected date renders
  identically to a declared one.

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
- **HOLDs are never queue candidates on verdict grounds** (amended 2026-08-20).
  A HOLD is a verdict, not an action. `recRank === 0` gets NO tier and renders in
  the scored table with its own reason, `HOLD — not actionable`, listed in the
  footnote separately from the no-record names — "the model published HOLD" and
  "no record exists" are different facts and must not share a label. The ONE
  exception is deliberate: **tiers 1–2 are clock-driven, not verdict-driven**, so
  a HOLD-rated name reporting earnings still cards, because the deadline is a
  fact of the calendar and does not depend on the rating. The old
  `'HOLD — fills only an otherwise-empty slot'` why-string described a rule NO
  CODE ENFORCED and rendered on a card sitting beside five others: a provenance
  line asserting a false claim about our own derivation. It is deleted.
- **Tier-5 order is |recRank| FIRST; a loaded structure is a tiebreak only**
  (amended 2026-08-20). `hasStructure` used to sort ahead of `|recRank|` inside
  tier 5. `state.longRows` is populated by expanding a row on the Options tab,
  so that made **queue membership a function of which Options rows happened to
  be loaded**. Measured on live Pages 2026-08-20: `longRows` held exactly one
  symbol, AAPL (HOLD, recRank 0), and it took the last slot ahead of MU (74),
  NVDA (73), TSM (72) and AMZN (72), all `hasStructure: false`. Conviction is the
  verdict; a loaded long row is a cache state, and **the verdict must never rank
  behind the cache**. Neither membership nor order may depend on a load state.
- **The cap is TIER-AWARE** (amended 2026-08-20). Tiers 1–4 always render as
  full cards, uncapped — an arithmetic limit must not evict a real deadline.
  Tier-5 conviction fills only top the queue up to `QUEUE_CAP` (6) total; when
  tiers 1–4 already number ≥6, zero fills render. `buildQueue()` returns
  `core` / `fillPool` / `fills` alongside `selected` and `cut` so the split is
  checkable. **An empty queue renders a stated finding naming what it looked
  at**, never a blank and never a padded slot: a quiet day is the answer.
- **Queue expansion state lives in `state`, NEVER as a class on the DOM.**
  `renderQueue()` rebuilds its whole subtree with `innerHTML` on the 30-second
  staleness cycle and on every `renderToday()`. Measured 2026-08-20: after
  reopening, `before === after` is **false** — the node is destroyed, not merely
  declassed — so `classList.toggle('open')` on `#qcut` or on a `.qcard` was wiped
  seconds after the click. `state.queueCutOpen` / `queueCutSyms` /
  `queueOpenSyms` outlive the render and persist until the user collapses them.
  **No timer is involved and none may be added** — re-rendering less often would
  hide the bug rather than fix it, and the 30-second re-derive is what keeps the
  queue's clock-driven states honest. Cut chips are clickable and expand IN
  PLACE through the same `queueCard()` renderer (`opts.cut`), so an expanded chip
  carries identical provenance, sub-lines and as-ofs to a queue card — there is
  no second card renderer and no sub-queue.

## Session-brief facts (phase 8)

- **The brief is a STACK, not a card.** Every slot published for the current
  trading day renders in session order, each with its own as-of. Rendering only
  the 06:00 record — which is what phases 2–7 did — meant the 11:30 and 13:15
  narratives were fetched on every load and thrown away, while the timeline
  reported that they existed. Fetching a fact and refusing to show it is the
  same class of bug as not having it.
- **The trading day is `ptPartsOf(queueNow()).iso` — the queue's clock, not a
  second one.** The old `briefTimeline()` called a private `ptMinutesNow()`,
  which `?clock=` could not reach, so the rollover was untestable. That function
  is deleted; do not reintroduce a local clock in this module.
- **Every rendered brief is stamped with ITS OWN PT date and must match today's.**
  `/daily` has a 24h TTL, so between the rollover and the next 06:00 run the
  endpoint still serves YESTERDAY's whole object. A record from another PT date
  renders as `other-day` on the timeline and NEVER as a card. Verified: clock
  2026-08-19 23:59 → 3 cards; 2026-08-20 00:01 → 0 cards.
- **The per-slot bank (`localStorage` `dd:brief-slots`) exists because the Worker
  replaces the whole payload** (see the interface facts above). Rules, all
  load-bearing: only content with a real ts on today's PT date is banked;
  **newest ts wins per slot**, so a revision replaces and an absence never does;
  **one PT date at a time** — a bank stamped with any other date is dropped
  whole, which IS the rollover reset; **`?clock=` never writes**, so a test clock
  cannot corrupt real state; and a card served from the bank rather than the live
  payload **says so on its face** (`held locally · not in the current /daily`) —
  "we still hold this" and "the Worker still serves this" are different facts.
  When storage is unavailable the footer names it and the page degrades to
  payload-only rather than silently showing less.
- **A slot that has not run renders as a state, never as content.** The states
  are `today` · `held` · `undated` · `other-day` · `refused` (`eod.complete ===
  false`, the Worker's own placeholder) · `generating` (`middayLoading` /
  `eodLoading`) · `no-session` (weekend) · `pending` · `missing`. `complete ===
  false` IS THE TEST, never `!complete` — absent means an older Worker, and a
  real summary must not be relabelled as a failed one.
- **As-ofs render in PT** (`fmtClockPt`), because the slots are named for PT cron
  times; printing the reader's local clock beside a slot labelled "11:30" reads
  as the job running hours late.

## Render-scope facts (2026-08-20)

- **`index.html` is ONE `<script>` block (lines ~778–4720), so every render helper
  shares one global scope and a duplicate `function` name silently replaces the
  earlier one for EVERY caller.** Phase 7 added a second `function trendCell(r)`
  for Diversify beside the phase-1 `function trendCell(s)` that renders the Names
  column. The later declaration won, so Names called the Diversify renderer for
  four commits. **Name cell helpers for their surface** (`divTrendCell`,
  `optionsCell`) and grep for the bare name before adding one.
- **A cell helper that returns a bare `<span>` where the caller concatenates
  `<td>`s does not render in the wrong style — it leaves the table entirely.**
  HTML foster-parenting hoists non-cell content out of a `<table>` and paints it
  *above* the table. The symptom was 39 italic `trend n/a` spans (one per row)
  running together above the Names header. **Every `*Cell()` called from a `<tr>`
  concatenation must return its own `<td>`;** a helper called from a flex row
  returns a `<span>`, and the two are not interchangeable.
- **The visible spill was the harmless half of that bug.** Losing one `<td>`
  left 5 cells under 6 `<th>`, so every column after Key level shifted LEFT:
  earnings dates rendered under the **Trend** header and the options
  `not loaded` state under **Earnings**, while Options sat empty. That is
  honesty rule 1 — one quantity under another's label — and it reads as
  plausible data, which is why it survived four commits unreported. **Assert the
  `<td>` count per row against the `<th>` count** when a table renderer changes;
  `renderNames()` now verifies at 6/6 across 39 rows.

## Design system

Same tokens as trading_dash. CSS custom properties in `:root`; never hardcode a
hex. `--bull #23d18b` / `--bear #f25f5c` / `--amber #f4b740` (neutral + stale) /
`--cyan #5ec5ea` (data accent) / `--violet #b48ead` / `--bg-0..3` / `--ink-0..3`.

**Contrast is a constraint, not a preference (phase 9).** Every ink tier meets
WCAG AA (4.5:1) for body text against the whole `bg-0..bg-3` surface range,
worst case measured on `bg-3`:

| token | value | worst-case | role |
|---|---|---|---|
| `--ink-0` | `#f5f1eb` | 15.32:1 | primary |
| `--ink-1` | `#c9c5be` | 10.02:1 | secondary body |
| `--ink-2` | `#a9a6b5` | 7.23:1 | muted labels, nav — **was `#888591`, 4.77:1** |
| `--ink-3` | `#858391` | 4.64:1 | metadata, provenance, `.dim` — **was `#5a5862`, 2.47:1** |

`--ink-3` is the most-used ink on the site (49 CSS references) and sat under even
the 3:1 large-text floor. It powers `.dim`, every `.why` provenance string and
every "not published" reason — exactly the text that explains why a number is
missing. Unreadable, that text is the blank cell honesty rule 2 forbids.

**This is a DARK theme: raising contrast means moving the grays LIGHTER.** The
tiers keep ~1.4–1.55× contrast steps, so muted still reads as muted. Four tiers
were kept rather than consolidated — they carry distinct roles and the token
names are part of this contract.

**Never dim text with `opacity`.** It composites against the background and
silently halves contrast: `.tl .t.pend` at `opacity: 0.5` measured **2.15:1**,
the least readable text on the site, on the states whose whole job is to explain
an absence. Dim by choosing a lower ink tier. `opacity` on a *row* of already-AA
text is fine above ~0.72 (`.optrow.dimrow`, `.optcell.faded` measure 6.17:1);
below that it fails. Disabled controls (`.btn:disabled` at 0.5) are exempt —
WCAG 1.4.3 excludes inactive UI components.
Fonts: Fraunces (display serif), Geist (body), JetBrains Mono (numbers/labels).
Status is never conveyed by color alone.

## Build plan — ALL PHASES SHIPPED

| phase | what | commit |
|---|---|---|
| — | Scaffold: docs, design reference, gitignore | `c0e9895` |
| 0 | **Probe page** — live `/api/market/snapshot` round-trip in a browser; now `probe.html`, the permanent CORS canary | `668ed1a` |
| 1 | **Names** — 6-column table on `/api/watchlist/batch`, attention sort, expanders, one-verdict-store rendering, click-throughs | `e75ea7a` |
| 2 | **Today** static modules — regime line, session brief + timeline, calendar, levels watch, your-movers, sector heat | `f707f82` |
| 3 | **Action queue** — a pure function of the shared store + a PT clock; phase awareness, deadline logic, `?clock=` test override | `37ea389` |
| 4 | **Options** — verdict layer over `/api/long/batch` + `/api/long/:ticker`, lanes A–F, legend modal, deep links | `49bd5fb` |
| 5 | **Ticker page** — hero decision block + 4 evidence groups; base-rate-beside-rate on Record; no AI spend on load | `e4cbac1` |
| 6 | **Radar** — `/api/radar`, read-then-append adoption | `8408918` |
| 7 | **Diversify** — `/api/income/*`, categorized sleeves, no verdicts | `f832412` |
| 8 | **Session-brief stack** — all slots published for the current trading day, stacked with per-slot as-of; per-slot bank against the Worker's whole-payload rewrite | `504d1bb` |
| 9 | **Ticker verdict auto-fetch + refresh button in every branch; two record shapes normalised; WCAG AA ink palette** | `ecc8086` |
| fix | **Names column shift** — phase 7's `trendCell` shadowed phase 1's; renamed to `divTrendCell`. Names lost its Trend `<td>`, spilling 39 spans above the table and shifting every later column left | `b5d7f7b` |
| 10 | **Queue amendments** — HOLDs excluded on verdict grounds, tier-aware cap (1–4 uncapped, tier-5 fills to 6), tier-5 ordered by \|recRank\| with structure as a tiebreak, clickable cut chips expanding through `queueCard()`, expansion state moved into `state` so the 30-second re-render preserves it | this commit |

Doc-sync commits: `557329e` (phase-1 payload facts), `e626a64` (phase-5 payload
facts).

### Known gaps — carried forward from the phase reports

These are stated, not fixed. Each is a real limit of the built surfaces.

1. **No NYSE holiday calendar client-side** (phase 3). Session phase and
   next-trading-day use a weekday rule only, so a market holiday reads as a
   trading day — measured: Thanksgiving 2026-11-26 reads `intraday`. The Worker's
   `NYSE_HOLIDAYS` table is not published on any endpoint this page fetches;
   shipping it on one is a trading_dash task. The phase line and queue footnote
   label the derivation so the reading is at least visible.
2. **Post-DST AMC prints classify `unknown`** (phase 2/3). Yahoo appears to encode
   after-close prints at a fixed `20:00Z`, which is 16:00 ET under EDT but 15:00 ET
   under EST — mid-session, so it fails the ≥16:00 cut. Expect roughly half the
   year's AMC names to read `unknown` and fall back to the earlier (BMO) deadline.
   Widening the cut to 15:00 would be guessing at Yahoo's intent. BMO is
   unaffected. Full write-up in trading_dash's CLAUDE.md.
3. **Tax character (qualified vs ordinary) is NOT derivable and is not shipped**
   (phase 7). DESIGN.md round-2 item 3 asked for it. No Yahoo module carries it,
   and it depends on the issuer's own 1099-DIV allocation *and* on the holder's
   holding period — neither of which the Worker can see. The Diversify tab
   surfaces the Worker's `taxCharacterNote` explaining the omission rather than
   estimating. Genuinely unbuilt, not deferred.
4. **`zeroFrac` classification caveats** (phase 7). `distributionKind` is decided
   by what fraction of consecutive payments repeat, against a 0.50 threshold.
   SCHD and VYM measure ~8% and classify `variable`, so their `growth5y` and `cut`
   render n/a-with-reason — **correct, not a bug**: a pass-through fund's payment
   moves almost every period, so a period-over-period decline is a fluctuation and
   a point-to-point 5y ratio is two noisy draws. The consequence is that no cut can
   ever be detected on a variable payer, which is a real blind spot rather than a
   clean refusal. A name sitting near the threshold could also flip kind between
   runs; nothing pins it.
5. **No position sizes anywhere.** The app knows which names are on a list, never
   how much is held. Every "coverage" figure is a count for this reason.
6. **Radar's universe is the Yahoo screeners, not the S&P 500 + Nasdaq 100 sweep**
   originally specified (design contract #5). Same gates, different population.
7. **The session-brief bank is per browser profile AND per ORIGIN** (phase 8).
   `localhost:8123` and `ambermlysak.github.io` keep separate banks, so a slot
   banked while testing locally is not there on the live site — measured
   2026-08-19: the local copy held the 11:31 midday and rendered it `held
   locally`; the live site, which never saw that payload, correctly rendered
   `missing`. More generally the bank can only hold a slot this browser actually
   received: if the Worker rewrites `/daily` and drops the 11:30 pulse before you
   load the page that day, that pulse is gone for good on this surface and
   nothing client-side can recover it. The real fix is making the Worker merge
   rather than replace, which is a trading_dash task. The stack degrades
   correctly (the slot reads `missing`, not blank), but it degrades.
8. **`?clock=` cannot exercise the bank's WRITE path** (phase 8), by design — a
   test clock that could write would be able to corrupt real state. The write
   path is therefore verified on the real clock with recorded payloads, and the
   read/reset path with the test clock. Nothing verifies the two together.

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
