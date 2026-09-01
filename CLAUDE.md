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
   `/api/queue` is a later trading_dash task if warranted. **One card kind is
   not assembled client-side: `print-tape`**, which renders a banked
   `/api/printtape` record — the Worker measures the print against the tape and
   publishes the verdict, and the page renders THAT with its as-ofs rather than
   re-deriving it.
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
- **`GET /api/earnings/:ticker` — the full response shape, recorded so nobody
  rediscovers it by spending.** Read out of trading_dash's
  `handleEarningsAnalysis` / `gatherEarningsFacts`; confirmed on the wire
  against NVDA 2026-08-27 (`cached: true`, no call spent).
  - **top level:** `ticker` · `facts{}` · `analysis{}|null` · `ts` · `_meta{}`.
    Plus `cached: true` **only on a KV hit** — the flag is ABSENT on a fresh
    generation, so cache state is `=== true` vs field-absent and **never a
    truthiness test**. Plus `error` (a string) with `analysis: null` when the
    symbol has neither a report date nor any EPS history. Plus `factsOnly: true`
    under `?facts=1`.
  - **`facts`:** `ticker` · `company` · `reportDate` · `dateSource` ·
    `history[]{quarter, epsActual, epsEstimate, surprisePct}` ·
    `revenue[]{quarter, revenue, earnings}` ·
    `nextQuarter{epsEstimate, revenueEstimate, epsRevisedUp, epsRevisedDown, growthPct}` ·
    `currentQuarter{epsEstimate, growthPct}` ·
    `profile{marketCap, trailingPE, forwardPE, profitMargin, revenueGrowth, targetMean, currentPrice}` ·
    `reaction{}|null` · `news[]` · `newsStatus` · `newsWindow{}|null` · `newsSource|null`.
  - **`reaction`:** `reportDate` · `reactionDate` · `timing` · `isPartial` ·
    `priorClose` · `reactionClose` · `openGapPct` · `day1Pct` · `day5Pct` ·
    `sinceReportPct` · `volumeVsAvg` · `sessionsSince`.
  - **`analysis`:** `quarter` · `verdict` (BEAT|MISS|MIXED) · `headline` ·
    `scorecard[]{metric, actual, estimate, result}` · `highlights[]` ·
    `callCommentary[]{theme, detail, source}` · `priceAction` · `watchNext[]`.
  - **`_meta`:** `source` (`'Yahoo + Claude synthesis'`) · `ok` · `delayed` ·
    `note` · `asOf` · `ttlSeconds` (43200) · `fetchedAt`. **NO `model` field
    anywhere** — the model name is not published and must not be inferred from
    `source`.
- **`/api/earnings` gotchas, all of which the block names on its face.**
  `_meta.asOf` is `facts.reportDate` — a **calendar date for the report, not a
  fetch time**; `_meta.fetchedAt` is the fetch time, and **on a cache hit it is
  the ORIGINAL generation**, banked in KV with the rest of the object, not this
  request. `facts.history` is capped at **FOUR** quarters (`.slice(-4)`) and
  ships **oldest-first**. There is **no session (BMO/AMC) field and no
  `isEstimate` flag** on this payload at all: `reaction.timing` answers a
  different question (which session *traded* the print) and must not be
  substituted for one. `dateSource` is one of `Yahoo earnings call date` ·
  `Yahoo earnings calendar` · a price-gap fallback whose own string starts
  `inferred from…` (this is the only estimate signal, and it is what the
  `inferred` tag renders from) · `null`. The freshness window is **12h**
  (`EARNINGS_TTL`) while the KV `expirationTtl` is **48h**, so a record can be
  in KV and still regenerate — **"not expired" and "still a cache hit" are
  different questions**, as with `analysis:`. Reported revenue carries **no
  consensus anywhere**, so no revenue quarter can ever be scored a beat or a
  miss, and the Worker rewrites any scorecard entry with a missing estimate to
  `result: n/a` before it ships.
- **`?facts=1` is the SPEND-FREE probe, and it sits BEHIND the cache check.**
  On a cached ticker it returns the whole banked payload (`cached: true`,
  analysis included); on an uncached one it gathers the upstream facts and
  returns `analysis: null, factsOnly: true` **without reaching the model**. Use
  it to establish cache state before clicking anything — that is how NVDA was
  confirmed a hit and LLY confirmed a miss without spending. It also means
  `factsOnly` is a THIRD cache state: it carries no `cached` flag yet spends
  nothing, so reporting a bare missing `cached` as "this spent a call" would
  assert a charge that never happened.
- **`/api/earnings` 403s without an `Origin` header.** A curl with only
  `x-dash-key` returns `{"error":"Forbidden"}` and 21 bytes. Send
  `-H "Origin: https://ambermlysak.github.io"`. A 403 here is the allowlist, not
  a bad key — and it is easy to misread as the record being missing.
- **`facts.revenue` quarters are FISCAL labels (`3Q2025`, `1Q2026`), not ISO
  dates, and do NOT sort lexicographically.** Sorting them as strings put
  `4Q2025` and `3Q2025` ahead of `1Q2026`, so the newest quarter rendered last
  while the model's own prose walked the same series forward in time — two
  orderings of one series on one screen. Parse `NQYYYY`; where a label does not
  parse, render payload order and **say so** rather than imposing an order the
  page cannot back. Caught by reading the painted DOM, not the JSON.
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
- **The `/api/daily` slot-merge deploy is UNCONFIRMED from this repo as of
  2026-08-20 11:05 PT.** The cleanup task expected merged records to carry a
  `ptDate`; the live payload carries **none** — not at the top level, not on the
  `open` slot (`keys=body,headline`), and there is no `midday`/`eod` yet today.
  This does NOT prove the deploy is missing: `_meta.asOf` is
  `2026-08-20T13:01:07.331Z` (06:01 PT) with `ttlSeconds: 86400`, so the object
  on the wire was **written by this morning's 06:00 cron** and, if `ptDate` is
  stamped on WRITE rather than on read, its absence is expected until the next
  slot writes. `/api/analysis` normalises on READ and visibly changed; `/daily`
  shows no read-path change. **The frontend cannot distinguish "write-path
  change, not yet exercised" from "not deployed" — the 11:30 PT midday run is
  the first write that would settle it.** Until then the whole-payload-rewrite
  fact below stands as written and the per-slot bank stays.
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
- **The verdict store now serves ONE record shape — the Worker normalises at
  the READ boundary** (Worker deploy confirmed live 2026-08-20). Records read
  `{rating, confidence, recommendation, drivers[]|null, summary, ts, schemaEra,
  driversSource}`. A legacy record keeps `schemaEra: 'legacy-synthesis'`, has its
  `recommendation` mapped from the retired `action`, and carries `drivers: null`
  with `driversSource: 'not-in-legacy-synthesis-shape'`; a nightly-batch record
  reads `schemaEra: 'canonical'` with `driversSource: 'record'`. The retired
  fields (`action`, `factors{}`, `trend`, `pattern`) are **gone from the read
  path** — measured absent on COST/WMT/KMB/MO/CLX/KO.
  **The client-side normalisers `verdictCall()` / `verdictDrivers()` are DELETED**
  (they had become identity functions); every surface reads `recommendation` and
  `drivers` directly. History, because the failure is worth keeping: the store
  used to hold two shapes, every surface read only `recommendation`, and a
  verdict generated by the button rendered "the record carries a rating but no
  one-line call" while carrying one under `action`.
- **THREE endpoints serve the verdict record and they do NOT carry the same
  metadata.** Measured 2026-08-20 before deleting the normalisers:
  `GET /api/analysis/:ticker` carries `schemaEra` + `driversSource`;
  `GET /api/watchlist/batch` is normalised (KO/MO/CLX carry `recommendation` as a
  string and `drivers: null`, no `action`, no `factors`) but publishes **no
  `schemaEra` and no `driversSource`**; the `POST /api/ai/synthesis` response
  body is normalised too — `.analysis` carries `recommendation` + `drivers[]` and
  no `action` — but carries **no `schemaEra`, no `driversSource`, and its inner
  `ts` is still null**. So a drivers-absent reason may quote `driversSource` only
  where the payload actually has one, and must say the payload publishes none
  where it does not. **Do not fabricate the reason from the field's absence.**
- **`drivers: null` is now a ROUTINE state, not an error**, and every render site
  must name it. It is null on every legacy record, arriving pre-normalised. The
  Names expander is the only surface that renders drivers; it renders
  `drivers n/p` with the payload's own `driversSource` on hover. Honesty rule 7:
  a refused measurement is a finding, and a null drivers list is the Worker
  telling us the source record never had one.
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
- **`/api/long/batch` carries a `top3` RIDER — the daily top-3 options ranking**
  (phase 11), the same envelope pattern as `macro`. **THREE ENVELOPE STATES AND
  THEY ARE NOT INTERCHANGEABLE**, which the Worker's own `readTop3()` comment
  states outright: an **absent key** is a Worker predating the rider
  (field-absent); **`top3: null`** is this Worker reporting that KV holds no
  `top3:{PT-date}` record; a **record carrying `entries: []`** is the gates
  having run with nothing surviving — a valid published result. The value alone
  cannot separate the first two, so the frontend captures
  `state.top3KeyPresent` (a `hasOwnProperty` on the envelope) **beside**
  `state.top3`. A fourth state is ours, not the Worker's: the rider rides on
  that envelope, so **a long batch that failed to fetch is not field-absent** and
  must never be rendered as a claim about what the Worker publishes.
  `top3State()` returns exactly one of `failed` · `not-loaded` · `field-absent` ·
  `no-record` · `record`.
- **The `top3:` key is written by a 1:15pm PT cron, and `readTop3()` SERVES THE
  NEWEST SURVIVING RECORD — not today's key alone** (trading_dash `b730011`, then
  `50bf2e5` on 2026-08-31). It reads `top3:{ptDate()}` and, on a miss, walks back
  up to `TOP3_SERVE_WALKBACK_DAYS` (**5**) calendar days; retention is `TOP3_TTL`
  (**7 days**, raised from 36h in the same commit). So the ordinary reading on a
  trading morning, on a Monday, and after a holiday is the **previous session's
  record, served unmodified under its own `ptDate` and `asOf`** — the Worker
  attaches no `served` marker, because a second field claiming the same fact is a
  second field that can disagree. `top3: null` therefore now means **nothing
  survives anywhere in the serving window**, which is a stronger claim than the
  old "not 1:15pm PT yet"; the strip's null copy says the stronger thing.
  A re-run rewrites the key whole. A partial run still writes but does not stamp
  its dedup key (`complete: false` + `incompleteReason`); **`complete === false`
  is the test, never `!complete`.**
- **The serving window is NOT PUBLISHED on any payload this page fetches.**
  `gates` carries every weight, anchor and floor — and it rides on a *record*,
  so the one state that has to describe the window (no record) has nothing to
  read it from. `TOP3_WALKBACK_DAYS` / `TOP3_RETENTION_DAYS` in `index.html` are
  therefore **hand-carried copies** of the Worker's constants, they say so
  everywhere they render (`TOP3_WINDOW_NOTE`, on the null state, the stale badge
  and both modal branches), and **a Worker-side change to either would not show
  up on this screen**. Publishing the window on the envelope is a trading_dash
  task.
- **A NEW TTL ONLY APPLIES TO RECORDS WRITTEN AFTER THE DEPLOY** — `expirationTtl`
  is set at `put()` time. Friday 2026-08-28's record was written under the old 36h
  and expired ~01:15 PT Sunday, so on Monday 2026-08-31 the walk-back had nothing
  in range to find and the live envelope served `top3: null` regardless of whether
  `50bf2e5` was deployed. **The frontend cannot distinguish "walk-back not
  deployed" from "deployed, everything in range aged out"** — both are `null`, and
  the null copy names both. The first live stale render is possible only the
  morning after the first 1:15pm PT run that writes under the 7-day TTL.
- **The score is a weighted composite over FIXED anchors, and it is
  decomposable.** Six components (`probMarket` · `probMeasured` · `sharpe` ·
  `win` · `beEm` · `dollars`), each clipped to [0,1], each shipping its raw
  input, its anchor, the clipped component, its weight and the points it
  contributed. **No score may render as a bare number** — the strip prints it
  against the record's own `weightTotal` and expands into the full table.
  `beEm` is INVERTED and its `anchor` is a **span**, not a threshold, so it is
  labelled as one; printing it as a divisor would assert arithmetic the record
  does not do. **Every weight, anchor, floor and rule string on this surface —
  including the ? modal's gate section — is rendered FROM `top3.gates`**, never
  typed into markup, for the same reason `state.longGates` is.
- **`E[$]` is in the blend and `E[R]` is deliberately not.** The candidate's own
  `upsideTruncatedReason` is the argument: an uncapped long is scored only as far
  as the largest observed move while "capped structures are not truncated this
  way", and the ranking mixes lane B (uncapped) with lane C (capped). E[R] still
  renders in the breakdown, explicitly labelled *not scored*.
- **A missing score input EXCLUDES the candidate; it never scores zero.** Same
  for the concentration gate — a missing `expectancyEpisodesTo50` FAILS it. The
  strip renders each entry's `episodes-to-50%`, `¼-Kelly`, `P(BE) | cov 1y/3y`
  and `gap` **with `drift` adjacent** (through the same `pcovCell()` and
  `gapDriftBit()` the candidate table uses — one renderer, so the two surfaces
  cannot drift apart).
- **A `top3` entry is FLAT where a long row is lane-nested.** `expiry`/`dte` live
  on the lane entry in `/api/long/:ticker` and are carried down onto the entry
  here, so `structureLabel()` is reached through a shim (`top3StructureLabel()`)
  that rebuilds the `{L, c}` shape rather than a second label renderer.
- **`/api/radar` is banked once per PT date** (`ttlSeconds` ~36h, `cached: true`,
  `ptDate`). Render its banked time; do not refetch on a tab switch — a discovery
  list that changed every time you looked at it would not be the day's list. The
  envelope carries `refused`/`reason`, `complete`, `sources[]` (with per-source
  `ok`/`reason`), `funnel{}`, `gates{}` and `considered`; each is its own named
  state and `complete: false` renders the candidates that DID clear alongside the
  named down source.
- **`GET /api/printtape?date=` — PRINT vs TAPE, the earnings-divergence records**
  (trading_dash `02aac87`, **schema 2 since `b6b90c1`**). KV assembly only: zero fetches, zero writes, nothing
  recomputed. `requireSecret` (`x-dash-key`), **not `aiGuard`** — it cannot spend,
  and debiting the Claude ceiling for a read would let a page poll exhaust the
  crons' budget. `date` defaults to **`etToday()`**, and every key is ET-dated
  (`printtape:{TICKER}:{ET-DATE}`), retained `PRINTTAPE_TTL` = **7d**. Written by
  four crons outside the branch chain: 05:30 / 06:15 PT for BMO names, 13:30 /
  14:30 PT for AMC names.
  - **Envelope:** `records[]` · `meta{date, asOf, ran, ranReason, eligible[],
    measured[], skipped[], unreadable[], passes[], divergencePct, schema}` ·
    `_meta`. **THERE IS NO `meta.scanOk`.** `scanOk` / `scanReason` are published
    **per pass** inside `meta.passes[]`, because the BMO and AMC passes succeed
    and fail independently and one top-level flag would have to pick one of them
    to describe. Do not synthesise one. `scanOk === false` IS THE TEST — absent
    means a Worker that does not publish it.
  - **`meta.ran === false` and `records: []` are DIFFERENT FACTS.** `ran: false`
    is "no pass wrote a day index" (a weekend, an NYSE holiday, or a date past the
    TTL); `ran: true` with an empty `records` is the job having run and found that
    nobody on the watchlist reported — the ~95% answer, and a measurement. The
    endpoint comment says this outright and the frontend keeps them apart.
  - **Record:** `schema` · `ticker` · `reportDate` · `session` (`bmo`|`amc`|
    `unknown`) · `earningsTs` · `earningsIsEstimate` · `pass` · `ts` · `print` ·
    `tape` · `implied` · `divergent` · `refusalReason` · `divergenceTest` ·
    `guidance` · `baseRate` · `passes[]` · **`consensusSource`** ·
    **`consensusBankedTs`** (both schema 2, both RECORD-LEVEL), plus
    `carriedForward` **only after a merge** (field-absent on a single-pass
    record, never `null` there) and `carriedOverFrom` only on a carry-over pass.
  - **`divergent` IS THREE-VALUED AND `null` IS A REFUSAL, NOT A "NO".** It fires
    on exactly one direction — EPS beat AND revenue beat AND `changePct <= -3.0`,
    read from `tape[tape.usedWindow]` and from nowhere else.
    An unknown session, an unpublished actual or a missing consensus means the
    question could not be asked; `refusalReason` always says which.
  - **The print's numbers are FLAT; `print.eps` and `print.revenue` are the
    REFUSAL sub-objects.** Measurements live at `epsActual` / `epsEst` /
    `epsSurprisePct` / `revActual` / `revEst` / `revSurprisePct`; an `eps` or
    `revenue` KEY appears only when that half refused, and the two halves refuse
    INDEPENDENTLY. A whole-block refusal is `print.status`.
  - **An actual and a consensus are only ever compared WITHIN ONE QUARTER**
    (`print.quarter`, string equality on the period-end). This is the bug the
    whole record exists to avoid: taking the newest of each would have printed
    PANW at a confident 13% miss that never happened.
  - **TWO PASSES BECAUSE YAHOO'S ACTUALS LAG AND ITS CONSENSUS EXPIRES.** Pass 1
    banks the consensus while `earningsTrend.0q` still names the reported quarter;
    pass 2 catches the actual, by which time the quarter's REVENUE consensus
    exists in no module at all. So a single line of numbers can have **two read
    times behind it** — `print.carriedFromTs` / `carriedFromPass` /
    `carriedFields` — and the provenance footer says *"consensus banked <time>,
    actual <time>"* rather than one as-of.
  - **`PRINTTAPE_SCHEMA` IS **2** AND THE TEST IS STRICT EQUALITY, on this page as
    on the Worker.** Schema 2 moved the tape's numbers: a schema-1 record carries
    `changePct` at the TOP of the `tape` block, a schema-2 reader looks for it
    inside a window. A schema-1 record read by this code therefore finds
    `tape.pre`/`tape.post` absent and would render a real 14% reaction as
    `tape n/p` — a MISREAD, not a gap. `ptSchemaOk()` gates before any field is
    read; an off-schema record renders `record schema N — this page reads schema 2`
    with the record's own ts, in its own **Not read** group, **outside the cap**
    (whether it would have been actionable is exactly what cannot be said) and
    named in the queue footnote. **The Worker's endpoint applies the same equality
    one layer up**, so a record normally cannot reach this gate at all: an
    off-schema DAY INDEX reads as absent (`ran: false`) and an off-schema record
    lands in `meta.unreadable[]`. Measured 2026-09-01 after the deploy — every
    date from 08-28 to 09-02 answered `ran: false` because the day indices were
    schema 1. The frontend gate is for the case the endpoint's guard does not
    cover: a FUTURE schema this page predates.
  - **`meta.ranReason` NAMES TWO CAUSES AND THERE ARE THREE.** Its text is "the
    job did not run that day, or the day is older than PRINTTAPE_TTL"; the third
    is a day index at another schema reading as absent. The `did-not-run` state
    renders the Worker's string verbatim and APPENDS the third cause rather than
    substituting for it — the frontend cannot tell which one applies. Publishing
    the schema-mismatch cause on `ranReason` is a trading_dash task.
  - **`tape` IS A PAIR OF WINDOWS AND NOTHING IS HOISTED.** `tape.pre` and
    `tape.post`, each independently a reading or a `{status, reason}` refusal,
    plus `tape.sessionWindows` (which were attempted: `bmo` → pre only, `amc` →
    both) and **`tape.usedWindow`** naming the one the verdict read — the freshest
    by `quoteTime`, **re-derived after every merge and never carried**. The
    Worker publishes NO top-level `changePct`, because a duplicated number beside
    the window it came from is a second field that can disagree with the first;
    `ptTapeBit()` reads `tape[w]` and never a copy. A window that refused is
    `{status: 'unavailable'|'not-applicable', reason}` — `not-applicable` is a
    BMO print's post-market, `unavailable` is a quote older than the print
    (the staleness guard, which is what makes the two-window read safe without
    consulting a clock).
    **THE CARD RENDERS BOTH READINGS WHEN BOTH EXIST**, ordered by `quoteTime`
    (chronological — for an AMC print that is post then pre, the order they
    happened in), each with its own PT quote time, and the used one is named IN
    WORDS (`(verdict on pre)`) as well as inked, because status is never conveyed
    by colour alone. One reading present → it renders alone with the other's
    reason on hover. The extended-hours caveat renders ONCE.
  - **`consensusSource` AND `consensusBankedTs` ARE RECORD-LEVEL, NOT ON `print`.**
    Read out of `printTapeMeasure`'s return, where they sit beside
    `schema`/`ticker`/`pass`. **Reading them off `print` renders "consensus
    provenance not published by this Worker" on a record that publishes them** —
    a false claim about the Worker, not merely a missing line. Found on the
    painted card, not in the JSON. They answer two different questions:
    `consensusSource` (`pre-banked` | `live-pass`) is where THIS record's estimate
    figures came from; `consensusBankedTs` is whether a bank was taken at all, and
    `pre-banked` with a null ts is its own named state on the footer.
    **A `live-pass` record carries a caveat that is not decoration:** Yahoo's
    `earningsTrend.0q` rolls forward once the actual is ingested — measured by the
    Worker at 48 minutes on MDB, 2026-09-01 — and the reported quarter's REVENUE
    consensus survives that roll in no module, so a refusal on a live-pass record
    may be the roll rather than a number Yahoo never published.
  - **`meta.passes[].carryOver` IS `{date, screened[], measured[], skipped[],
    divergent[]}` OR `null`, AND AN EMPTY `screened` IS AMBIGUOUS ON THE WIRE.**
    The Worker's 05:30 / 06:15 PT BMO passes re-measure the PRIOR session's AMC
    reporters (both AMC passes sit inside the ninety minutes after the print and
    Yahoo's actuals lag it by hours — MDB 2026-09-01, EPS +18.09% against a
    post-market −14.56%, refused for a revenue actual that had not published).
    When the prior day's index is missing the Worker **logs it to its console and
    publishes no field**, so `screened: []` reads identically to "the prior day
    held no AMC names". `printTapeCarryNote()` therefore names the cause from the
    PRIOR DATE'S OWN ENVELOPE, which this page already fetches — `did-not-run`,
    `scan-failed`, `route-absent`, `failed` — and says the status is not published
    where neither payload answers. **That is a derivation from two payloads, not a
    field**; publishing a carry-over reason on the day index is a trading_dash
    task. `hasOwnProperty('carryOver')` separates an older Worker (key absent)
    from this one (`?? null`, so the key is always present).
  - **`tape.volume` IS REFUSED UNCONDITIONALLY AND THE REFUSAL IS THE
    MEASUREMENT, and it now rides on EACH WINDOW.** Yahoo publishes no extended-hours volume field anywhere, and
    the v8 1m `includePrePost` feed cannot be separated from the closing auction
    (measured: pre-market 0 on every bar for 5 of 5 names; 4 of 5 carried the
    whole post-window figure on the single session-boundary bar). `volumeStatus`
    + `volumeReason` ride on every tape block. **There is no volume line on the
    card** because there is no honest number to put on one — the reason renders
    in the provenance footer instead.
  - **`implied` IS NOT AN EARNINGS-ISOLATED MOVE.** It is the long row's
    front-expiry ATM-IV move over that expiry's whole DTE, and labelling it "the
    implied earnings move" would be the HV-as-IV failure again. `basis` says so in
    words and `straddlesReport` says whether the expiry is even on the far side of
    the print.
  - **`guidance` is `null` on a non-divergent record — not asked, never
    speculatively.** One Claude call per ticker per report, only on
    `divergent === true`. Its `source` is the earnings **news window**, not the
    release: there is no 8-K or press-release feed wired to this Worker, and
    `class: 'not-found'` is the honest answer when the window carried nothing.
    `PRINTTAPE_GUIDANCE_CLASSES` = `raised | held | cut | not-found`.
  - **`baseRate` is always `{status:'not-measured'}`** — scoring "how often does a
    beat-and-fade recover" needs a logged history of these records resolving
    forward and none exists; the feature only runs forward from its first deploy.
- **`GET /api/calendar/holidays?date=` — THE NYSE CALENDAR, and it retired this
  page's weekday arithmetic** (trading_dash `b6b90c1`). A pure computation over
  the Worker's `NYSE_HOLIDAY_TABLE` — zero fetches, zero binding ops, nothing
  written — gated with `requireSecret` like `/api/printtape`, for the same reason
  (it cannot spend, and debiting the Claude ceiling for a read would let a page
  poll exhaust the crons' budget). Response: `date` · `isTradingDay` · `reason`
  (`weekend` | `nyse-holiday` | `weekday`) · `prevTradingDay` · `nextTradingDay` ·
  `prevTradingDayIsPriorCalendarDay` · `holidays[]{date, name}` (13 entries) ·
  `through` (`2027-12-31`) · `calendarStale` · `calendarStaleNote` ·
  `earlyCloses: null` **with its reason** (early closes are modelled nowhere in
  that Worker) · `_meta`.
  - **ONE COPY OF THE CALENDAR.** Every trading-day walk on this page goes
    through `nextTradingDayIso` / `prevTradingDayIso` / `tradingDayStatusIso`,
    which read `state.calendar`. Two copies is how they drift, which is why the
    table is fetched rather than typed into `index.html` — the opposite of the
    `TOP3_WALKBACK_DAYS` situation, where the constants ARE hand-carried and say
    so everywhere they render.
  - **FETCHED ONCE PER PAGE LOAD, INSIDE `loadShared()` AND BEFORE
    `printTapeDates()`.** The ordering is structural: a date list built before the
    calendar arrived would be a weekday walk that no later fetch could correct.
    The focus re-read does NOT refetch (a pure computation over a hardcoded list
    cannot return a different answer), but **a failure is not cached** — only a
    success is — so one blip does not strand the session on the weekday rule.
    Measured: 1 request on load, still 1 after a second sweep, 2 after a poisoned
    failure, recovering to `rule: 'calendar'`.
  - **THE FALLBACK IS THE OLD BEHAVIOUR AND IT IS RENDERED, NEVER ASSUMED.**
    `calendarSource()` returns `{ok, rule, reason, n, through}`; the phase line
    reads `NYSE calendar` or an amber `holidays not modelled`, and the queue
    footnote prints the failure verbatim. `tradingDayStatusIso()` returns a
    `source` of `calendar` · `weekday` (the fallback) · `calendar-stale` (past
    `through`, where every weekday reads as open, holidays included) · `weekend`
    (unambiguous under both rules, so it carries NO "read by weekday" caveat —
    a caveat beside a Saturday would be doubt about a reading that has none).
  - **VERIFIED AGAINST THE WORKER'S OWN WALK, 8 dates, 0 disagreements**
    (2026-09-04/07/08, 2026-11-25/26/27, 2026-12-25, 2027-01-01). `loadShared()`
    also cross-checks our walk against the envelope's own `prevTradingDay` /
    `nextTradingDay` on every load and logs AGREES / DISAGREES — two
    implementations of one walk is how they drift.

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
- **CLOSED 2026-09-01: the frontend now HAS the NYSE holiday calendar.** It was
  a weekday rule only, so a market holiday read as a trading day (measured:
  Thanksgiving 2026-11-26 read `intraday`; it reads `post-close` now). The table
  arrives from `GET /api/calendar/holidays` — see the interface facts above — and
  everything that walks a trading day goes through the same two functions.
  **The weekday rule survives as the NAMED FALLBACK**, on a failed fetch or past
  the table's runway, and the phase line and queue footnote say which rule
  answered. `sessionPhase()` now reads `tradingDayStatusIso(pt.iso).open`, so a
  closure is post-close regardless of hour; the session brief's `no-session`
  state gates on the same rule the Worker's cron does and distinguishes a weekend
  from a closure.
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
- **The top-3 rider is ONE INFORMATIONAL LINE, not a card and not a tier**
  (phase 11). It renders below the cards, takes part in NO selection arithmetic,
  and says so on its face. The reason is the tier-aware cap: tiers 1–4 are
  clock-driven and a deadline is a fact of the calendar, so a *ranking* must
  never be able to cost one a slot. `buildQueue()` computes `top3Line` beside the
  tiers — so the line stays a pure function of the shared store plus the clock
  and prints in `dumpQueueTable()` — but `selected`/`core`/`fills` never see it.
  It renders **only** for entries whose record is keyed to the CURRENT trading
  day: an aged ranking belongs on the Options strip with its as-of, not in a
  queue line that would read as today's news. Absent, null, stale and failed
  records render nothing here, because the strip is where every one of those
  states is named and a queue line reporting the absence of an informational
  rider is noise, not a finding.
- **The `print-tape` card kind is a SEPARATE ROW FAMILY, not a tier** (phases 14,
  15). `buildQueue()` returns `ptRows` / `ptRest` / `ptGate` / `ptDayStates`
  beside the tiers, and
  the rows are deliberately kept OUT of `scored`: `scored` is one row per
  watchlist symbol keyed by symbol, and it is what `dumpQueueTable()` prints and
  what the HOLD / no-record footnote counts, while a print-tape record is keyed by
  ticker AND report date. Folding them in would put two rows under one symbol and
  corrupt both. The queue stays a pure function of the shared store plus the
  clock — `printTapeCard()` reads nothing `buildQueue()` did not already resolve.
  - **THE SCHEMA GATE COMES BEFORE THE DIVERGENCE TEST** (phase 15). `divergent`
    is a field this page would be reading out of a record whose field layout it
    does not know, so an off-schema record never answers a question here — it goes
    to `ptGate`, renders the gate card, and stays out of the cap arithmetic
    entirely. Its window is filtered on `reportDate` and `session` only, the two
    fields that key the record and that no schema bump has moved, so an aged one
    still drops.
  - **ONLY `divergent === true` CARDS.** `false` is a real answer and `null` is a
    refusal; both render on the ticker page, where the question was asked about
    that one name. Neither is an action, so neither takes a card.
  - **PLACEMENT: above the tier-3 level cards, below the clock-driven tiers 1–2.**
    A print the tape sold is a live setup, but an earnings deadline is a fact of
    the calendar and outranks it.
  - **THE CAP STAYS TIER-AWARE.** These cards displace conviction FILLS
    (`QUEUE_CAP - core.length - ptRows.length`) and can never displace a tier 1–4
    card. Demoted cards sit in the **Rest** group, outside the cap arithmetic
    entirely — they are no longer competing for a pre-open slot.
  - **THE ACTION WINDOW IS THE NEXT REGULAR OPEN (06:30 PT), because options do
    not trade extended hours.** BMO → the report day's own open; AMC (and
    `unknown`) → the next session's. `printTapeWindow()` returns `prep` →
    `reacting` (the Rest group, chip *open passed — reaction in progress*) →
    `dropped` at that session's 13:00 PT close. Verified at every boundary on
    `?clock=`: 06:29 PT prep / 06:31 PT Rest / 12:59 PT Rest / 13:01 PT gone.
    **The chip's clock state outranks guidance** — a demoted card says the open
    passed whatever guidance found, because that is a fact of the calendar and
    guidance is a reading.
  - **TWO DATES ARE FETCHED PER LOAD AND THE SECOND IS NOT A NICETY.** A record is
    keyed to its ET REPORT date but its card lives on the NEXT open, so an AMC
    print on D is banked under D while its card renders on D+1 — by which point a
    single-date fetch is asking for D+1 and the record is not in that answer.
    `printTapeDates()` returns `[etToday, prevTradingDay]` and the union is what
    the queue reads. **The prior trading day is the NYSE calendar's, not a weekday
    walk** (amended 2026-09-01): over Labor Day 2026-09-07 the session before
    Tuesday 2026-09-08 is FRIDAY 2026-09-04, and the weekday walk asked for the
    Monday the exchange was shut — `ran: false`, and Friday's AMC prints
    unreachable from Tuesday's page while their records sat inside retention.
    Verified on `?clock=2026-09-08`: `['2026-09-08','2026-09-04']` with the
    calendar, `['2026-09-08','2026-09-07']` without it. Measured: exactly 2 `/printtape` requests per load, still 2
    after repeated re-renders. **No polling** — it rides `loadShared()`, so it
    refetches only when the sweep does.
  - **`?clock=` REACHES THIS FETCH, DELIBERATELY.** Everywhere else the test clock
    moves a derivation and never a payload. This endpoint is DATE-ADDRESSED — the
    date is the record's identity, not a staleness property — so a frozen queue
    clock asking for the live date would run one day's window logic over another
    day's records. Every as-of still comes from the record's own timestamps on the
    real clock.
  - **EPS RENDERS AT THE PRECISION THE SURPRISE WAS COMPUTED AT** (`fmtEps`, up to
    5dp, trailing zeros trimmed). Found live on PANW: `fmt(v, 2)` rendered the
    0.97745 consensus as 0.98, against which the Worker's own +4.35% does not
    reconcile — 1.02/0.98 is +4.08%. Two numbers contradicting each other on one
    line, and the kind of thing only a rendered screen shows.
  - **A failed eligibility scan renders an AMBER LINE ABOVE THE CARDS, never an
    empty group.** `scanOk === false` means NO name was checked, which is a
    failure and not a quiet day. Every other day-state (`did-not-run` ·
    `none-reported` · `route-absent` (HTTP 404, an older Worker) · `failed` ·
    `not-loaded` · `records`) is named in the **queue footnote on every path**,
    including the ones that render no card and no banner — a state nothing renders
    is a state nobody can check.
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
- **The `held locally` marker is ALREADY divergence-only — do not "fix" it.**
  Verified by reading and on the live page 2026-08-20. `briefSlotStates()`
  computes `fromLive = Boolean(rec && live && live.ts === rec.ts)` and sets
  `st = fromLive ? 'today' : 'held'`. A banked slot whose record is still the one
  on the wire renders as an ordinary card with **no marker**; the marker appears
  only when the bank holds a record `/daily` no longer serves, or serves at a
  different `ts`. It was never an unconditional "this came from the bank" badge,
  so a Worker that merges instead of replacing needs no frontend change here —
  the marker simply stops appearing as divergence stops happening.
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

- **A badge hand-written in markup is a badge free to lie** (phase 12).
  `#build-tag` in the nav read `verdict-p11-1` and **nothing in the page ever
  wrote to it** — the visible build stamp and `window.DD_BUILD` were two
  independent strings, and the visible one is exactly what a "is the browser
  running the bundle I just pushed?" check reads. It is now rendered from
  `DD_BUILD`, same as every other badge here: from the source of truth, never
  typed beside it.
- **COPY THAT DESCRIBES THE WORKER'S READ PATH IS A CLAIM ABOUT THE WORKER, AND
  IT EXPIRES WHEN THE WORKER CHANGES** (2026-08-31). The top-3 null state read
  "KV holds no `top3:` record for the current PT date … the key is date-scoped,
  so this is also the expected reading outside a trading day" — three sentences
  of provenance, all correct when written, all false the moment `readTop3()`
  learned to walk back. Nothing broke and no test failed: a state string is not
  a number and no assertion covers it. **When a Worker fact recorded in this
  file changes, grep the rendered copy for the old fact in the same task** —
  `date-scoped`, `for the current PT date` and `has not run yet today` were the
  three phrases, all in one branch. The strings that explain an absence are
  exactly the strings nothing exercises.
- **A FIELD READ OFF THE WRONG OBJECT RENDERS AS A CLAIM ABOUT THE WORKER**
  (phase 15). `consensusSource` and `consensusBankedTs` are RECORD-level, beside
  `schema`/`ticker`/`pass`; the renderer was handed `rec.print` and so read
  `undefined` on every card — printing *"consensus provenance not published by
  this Worker"* on three records that publish it. **A missing-field branch whose
  copy asserts something about the SOURCE is the dangerous kind**: it does not
  look like a bug, it looks like a finding, and it is the exact inverse of
  honesty rule 7. Nothing in a string test would have caught it either, because
  the string it produced is a string the page is supposed to be able to produce.
  Caught by reading the painted `.qsrc` against a fixture whose value was known.
  **Check the field's home in the Worker's own return literal before writing the
  branch that explains its absence.**
- **The string tests cannot see the DOM, and one of these bugs only exists
  there** (phase 12). The earnings block passed 53 string assertions with a
  revenue row ordered `4Q2025 · 3Q2025 · 1Q2026` — wrong, and contradicting the
  model's own chronological prose three inches above it. Nothing in a
  substring test could catch that; reading the **painted** `innerText` did.
  Verification for a render change ends in a real HTML parser (headless Chrome
  over CDP works with no install: `chrome --headless=new
  --remote-debugging-port=9222`), where `<td>`-per-row against `<th>`, orphaned
  cells, foster-parenting, computed opacity and `scrollWidth` overflow are all
  answerable and none of them are answerable from a string.

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
| 10 | **Queue amendments** — HOLDs excluded on verdict grounds, tier-aware cap (1–4 uncapped, tier-5 fills to 6), tier-5 ordered by \|recRank\| with structure as a tiebreak, clickable cut chips expanding through `queueCard()`, expansion state moved into `state` so the 30-second re-render preserves it | `4c48c4d`… |
| 11 | **Daily top-3 options ranking** — the `top3` rider stored beside `macro`, the Top plays strip above the Options table (decomposable score, P(BE)\|cov, ¼-Kelly, episodes-to-50%, gap+drift, pool counts, exclusions), one informational queue line, a modal section rendered from `gates`, and `window.DD_BUILD` | this commit |
| 13 | **Top-3 serving window** — the Worker now walks back 5 calendar days over 7-day retention, so `top3: null` stopped meaning "no record under today's PT date"; the null copy, the modal's serving-window paragraph and the state comments now state the claim the Worker actually makes, and the stale render was re-checked against fixtures on both clocks | this commit |
| 14 | **Print vs tape** — `GET /api/printtape?date=` in the shared sweep (two ET dates, no polling); a `print-tape` card kind above the level cards for every `divergent: true` record inside its pre-open window, demoting to a **Rest** group once the open passes; a *Print vs tape* hero vline under Catalyst rendering all three answers; the scan-failure banner and a day-state clause on the queue footnote | this commit |
| 15 | **Print-vs-tape schema 2 + the NYSE calendar** — a strict schema gate rendering an off-schema record as a stated fact in its own uncapped **Not read** group; the tape rendered as the PAIR of windows it now is, both readings with their own PT quote times and the verdict's window named in words; `consensusSource` / `consensusBankedTs` on the footer with the quarter-roll caveat on a live pass; `GET /api/calendar/holidays` replacing every weekday walk on the page (phase, deadlines, print-tape dates, the brief's no-session state) with a named weekday fallback; and a carry-over clause on the queue footnote derived from the prior date's own envelope | this commit |
| 12 | **Ticker Earnings block** — `/api/earnings` rendered in four parts (header + facts + model read + provenance) in place of a `JSON.stringify(...).slice(0,240)` fallthrough that was the only branch that ever ran; the payload shape recorded in this file so it is never rediscovered by spending; `#build-tag` wired to `DD_BUILD` | this commit |

Doc-sync commits: `557329e` (phase-1 payload facts), `e626a64` (phase-5 payload
facts).

### Known gaps — carried forward from the phase reports

These are stated, not fixed. Each is a real limit of the built surfaces.

1. **~~No NYSE holiday calendar client-side~~ — CLOSED 2026-09-01 (phase 15).**
   It was a weekday rule only and Thanksgiving 2026-11-26 measured `intraday`.
   `GET /api/calendar/holidays` now publishes the Worker's own table and every
   trading-day walk on this page reads it; 2026-11-26 measures `post-close`, and
   the walk agrees with the Worker's on 8 dates checked. **What remains is not
   the gap but its fallback:** when the fetch fails, or past the table's runway
   (`through` = `2027-12-31`, beyond which every weekday reads as open), the
   weekday rule answers — rendered in amber on the phase line, named verbatim in
   the queue footnote, and reported as `source: 'weekday'` / `'calendar-stale'` by
   `tradingDayStatusIso()`. **Early closes are still modelled nowhere**: the
   endpoint returns `earlyCloses: null` WITH its reason, so the 1:00pm ET closes
   after Thanksgiving and on Christmas Eve read as full sessions on this page
   exactly as they do in the Worker's crons.
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
8. **The top-3 strip has never been rendered against a LIVE published record**
   (phase 11). The `top3:{PT-date}` key is written by a 1:15pm PT cron and the
   work was done that morning, so the live envelope carried `top3: null`
   throughout — which is exactly the *no-record* state, and that state WAS
   verified live. The three-entry, fewer-than-three, zero-entry and stale states
   were verified against a record built by running the Worker's own
   `top3Subscores` / `top3GateCandidate` / `top3Entry` / `top3GatesDeclared`
   over the live `/api/long/batch` rows and the live `analysis:` verdicts — real
   numbers through the real ranking code, but not a record the Worker itself
   wrote and served. **First load after a 1:15pm PT run is the check that closes
   this**, and the fields to confirm are the ones the fixture had to supply
   rather than read: `sweep.*`, `rowSource`/`rowAgeMs`, and `asOf` as sweep
   completion. **Still open on 2026-08-31**, and now with a second half: the
   *stale* render has also never been exercised against a live record. Measured
   that morning at 08:29 PT — key present, `top3: null`, `top3State()` =
   `no-record` — so the strip never reached the record branch at all. That is
   expected arithmetic rather than a fault (see the write-time TTL fact above),
   and it means the live stale check needs the morning after the first 1:15pm PT
   run under the 7-day TTL. The stale path is verified only against fixtures,
   on both the real clock and `?clock=`.
9. **The print-tape card has never rendered from a LIVE record at all, and the
   reason changed on 2026-09-01** (phases 14, 15). Phase 14 left this open with
   three live records that were all `divergent: null` — only the REFUSED state was
   exercised. Schema 2 then made it stricter: **the Worker's endpoint applies
   strict schema equality to the DAY INDEX**, so those schema-1 indices now read
   as absent and `/api/printtape` answers `ran: false` for every date. Measured
   2026-09-01 after the deploy: 08-28, 08-31, 09-01 and 09-02 all `ran: false`,
   `records: []`, with real schema-1 `printtape:{PANW,DELL,MDB}:2026-09-01`
   records sitting in KV behind them. **So the schema-1 records do NOT reach the
   frontend gate — they never leave the Worker**, and the live page correctly
   renders `did-not-run` on both dates rather than a broken card. The frontend
   gate could therefore only be exercised against a FIXTURE.
   **Everything schema-2 is verified against fixtures only**
   (`scratchpad/fixtures.js`, built field-for-field from `printTapeMeasure` /
   `printTapeTapeFrom` / `printTapeDivergence`, every envelope labelled FIXTURE
   in the provenance footer through the ordinary `_meta.source` path — there is
   no fixture code path in `index.html`): pre+post both readable with
   `usedWindow: 'pre'`, post-only, pre-only (a BMO print's `not-applicable`
   post), `consensusSource` at both values and `pre-banked` with a null bank ts,
   `divergent: false`, and a schema-1 record hitting the gate. **The check that
   closes this is the first load after a pass writes a schema-2 record**, and the
   fields to confirm are the ones the fixture had to supply rather than read:
   `tape.pre`/`tape.post` on a real two-window merge, `usedWindow` re-derived
   after it, `consensusBankedTs` from a real pre-bank, and `guidance.class` /
   `quote` on a real Claude classification.
10. **The two-date fetch's holiday gap is CLOSED; its cousin is not** (phases 14,
   15). `printTapeDates()` walks the NYSE calendar now, so after a Monday holiday
   it asks for the Friday. **But the fallback still inherits the old behaviour**:
   with the calendar fetch failed, the walk is the weekday rule again and an AMC
   print on the Friday before a Monday holiday is unreachable from Tuesday's page
   — one wasted KV assembly answering `ran: false`, not a wrong render, and the
   footnote says the calendar is unavailable. Nothing verifies the two together
   on a real holiday, because the next one is 2026-09-07.
11. **`?clock=` cannot exercise the bank's WRITE path** (phase 8), by design — a
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
