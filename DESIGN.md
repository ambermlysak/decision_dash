# Dashboard Simplification Review — decision-first reconstruction

Reviewed live: dashboard.html (all 6 tabs) and index.html (NVDA), 2026-08-18, plus CLAUDE.md / ARCHITECTURE.md.
Scope per your answers: both pages, aggressive cuts, time-of-day aware, interactive mockup (separate file — nothing existing touched).

---

> ## BUILT — status as of 2026-08-19
>
> **This document is the design rationale and the cut list. It is NOT the
> as-built spec.** Everything below was the argument for the rebuild; the
> dashboard now exists and won some arguments back. `CLAUDE.md` holds the
> as-built design contract and the known-gaps list — read that for what the app
> does, and this for why.
>
> All phases 0–7 shipped, live at <https://ambermlysak.github.io/decision_dash/>.
> Four tabs — **Today · Names · Options · Diversify** — plus the per-ticker page.
>
> **Where as-built diverges from this document:**
>
> | here | as-built | why |
> |---|---|---|
> | tab 3 is "Income" (§Round 2 #3) | **Diversify** — categorized sleeves: `income` \| `cyclical` \| `value` \| `defensive`, category user-assigned per entry and never inferred | income-only was too narrow for what the sleeve is actually for |
> | the sleeve uses "the same verdict format" (§Round 2 #3) | **no verdicts at all in v1** — no rating, no call, no badge | no AI touches that path, so there is nothing to render a verdict from |
> | tax character flagged qualified vs ordinary (§Round 2 #3) | **not shipped**, with the reason on screen | not derivable — no Yahoo module carries it and it depends on the issuer's 1099-DIV allocation and the holder's holding period |
> | Radar universe = S&P 500 + Nasdaq 100 (§Round 2 #2) | Yahoo screeners (`day_gainers`, `most_actives`) + banked sector picks, ranked by rvol | same gates, different population — what `/api/radar` could actually source |
> | `mockup.html`'s queue chip `TRIGGERED 10:42` | `AT/THROUGH LEVEL` · `APPROACHING` · `INTACT`, each with an as-of | one 15-min-delayed snapshot is not an intraday tape; a trigger *time* is unknowable and would be fiction |
> | BMO/AMC from `earningsTimestamp` (§Round 2 #1) | from `calendarEvents.earnings.earningsDate`, shipped as `earningsTs`/`earningsSession`/`earningsIsEstimate` | `earningsTimestamp` is on an endpoint the batch does not fetch; `calendarEvents` was already in hand at zero added cost |
>
> `mockup.html` remains the historical visual reference. Where it disagrees with
> the running app, the app is right and the table above says why.

---

## Diagnosis — what slows decisions today

1. **The decision is spread across six tabs.** A single trade idea on AMD lives in four places: the verdict in Watchlist, the option expression in Long, the sector context in Sectors, the catalyst in Market. You are the join. Nothing on the platform ever says "here are the 3 things worth acting on right now."

2. **Density without hierarchy.** A Watchlist row carries ~25 data points across 3 lines and 14 columns; an expanded Long row shows ~19 columns per candidate plus an 8-metric sub-line, times 6 lanes. Everything renders at equal visual weight, so the screen answers every question except "so what do I do."

3. **Methodology prose on screen.** The Long tab shows ~250 words of legend *above the first data row*. Sectors is ~1,100 words of narrative for 11 sectors. These are reference material rendered as if they were signal. Reading is not deciding.

4. **Surfaces that serve no decision you actually make.** Live today: the momentum scanner's top names were $2–$12 small caps (PFSA +181%, AIXC +245% at 478× volume) — names you'd never trade; Market Movers ≥10% is the same population; the IPO calendar was SPACs; post-market movers was empty; Midday is a blank tab most of the day. That's roughly a third of the platform's surface area.

5. **Four unreconciled Claude verdict streams.** Watchlist recommendation, per-ticker AI rating, sector opportunity/avoid (22 picks), and Market-tab watchlist signals all issue verdicts independently. Same model, four surfaces, no single queue.

6. **The highest-value screen defaults to empty.** Long opens with "not loaded" on 32 of 33 rows and offers a 96-second Load-all. The screen that expresses your primary strategy (buying options on conviction) is the one that can't say anything on open.

## The reconstruction — 3 tabs + a rebuilt ticker page

**Principle: verdict first, one line, with the trigger level and the invalidation level. Evidence on demand. A number earns pixels only if it can change an action.**

### 1. TODAY (replaces Market + Midday + Sectors + Scanner)

Time-aware — same surface, reordered by session phase:

- **Regime line** (one line, always on top): stance chip (MIXED etc.), VIX + term direction, 10Y, SPY trend, days to next FOMC/CPI. This compresses the current stance card + macro chip + half the index strip.
- **Action queue** (the centerpiece, ≤6 cards): ranked list assembled from the existing verdict streams. Each card = ticker · action verb · trigger level · structure to use · invalidation · why (one clause) · source + as-of. Morning = the plan; intraday = triggered / approaching / invalidated; post-close = what resolved + tomorrow's queue.
- **Levels watch**: watchlist names within ~3% of their key level (the Worker already computes levelPct — this is just a filter).
- **Session brief**: the day's narrative record, 6:00 briefing → 11:30 pulse → 1:15 close. Three sentences visible per brief, detail expands. This replaces three separate narrative blocks (brief + pre-market + stance prose) and the entire Midday tab. **Revised in phase 8 — this originally read "ONE narrative card that updates through the day", and as built the card never updated: it rendered the 6:00 brief and nothing else, with the other two slots reduced to existence ticks on the timeline. "Updates" was the wrong model anyway. The 11:30 pulse does not supersede the 6:00 plan, it is the day's second reading of it, and at 2pm you want to see what you were told at 6am beside what you were told at 11:30. So: every slot published for the current trading day renders, stacked in session order, each with its own as-of; a slot that has not run renders as a state on the timeline and never as content. The timeline stays — it is the one place a slot's state is reported.**
- **Calendar**: next 5 events that touch your names (FOMC, CPI/PCE, your earnings). Not a card per event — a strip.
- **Your movers**: watchlist names moving ±3%, instead of market-wide ≥10% junk.
- **Sector heat**: 11 chips, one line (XLK +0.2 …). Click a chip → the existing prose + opportunity/avoid on demand.

### 2. NAMES (rebuilt Watchlist)

14 columns → **6**: Ticker+price/Δ · Verdict (badge + the one-line call — this column is the point of the row) · Key level (distance, sup/res) · Trend (SMA-cross glyph + gap) · Earnings (countdown, ⚠ inside 7d) · Options chip (vol cheap/rich + best structure's E[R], read from the long: KV row already fetched by the batch).

Default sort: **attention** — triggered > within 3% of level > earnings ≤7d > rest. Everything cut from the row (RSI, SWING σ, FWD P/E, SHORT %, UPSIDE, 52W bar, SECTOR, driver chips) moves to the expander, which also picks up the golden-cross setup detail. Nothing is deleted from the payload — it just stops costing a column.

### 3. OPTIONS (rebuilt Long)

- **Legend → a “?” modal.** All of it. The screen keeps zero words of methodology above the data.
- **Collapsed row = a verdict, not a coordinate line**: ticker · vol state chip (cheap/rich/collecting) · **best candidate across all six lanes** (structure + strike + expiry) · its E[R] · BE/EM · cov vs P(BE) · next catalyst. One line answers "is there a trade here."
- **Expand → lanes, each with a one-line lane verdict** ("Lane E: gated — no catalyst inside expiry" / "Lane A: refuses coverage at this horizon — see why"), then the candidate table cut to 8 decision columns: contract · debit · breakeven (% move) · BE/EM · P(BE) | cov · E[R] · liquidity health (one dot summarizing spread/OI) · read. The other ~11 columns (extr%, lev, carry, θ, vega, E[$], gap, drift, Sharpe, ¼-Kelly, episodes…) live one level deeper, per candidate.
- Load-state honesty stays; stale/refusal states stay — refusals render as findings, exactly as your rules require, just at lane level once instead of 19 columns wide.

### 4. TICKER PAGE (index.html: 14 cards → hero + plan + 4 groups)

- **Hero = the decision**: verdict badge · the one-line call with add/trim/stop levels rendered as a level ladder against current price · best option structure (from the long: row) · next catalyst + countdown · confidence (ordinal, as now).
- **4 collapsed evidence groups** replace 14 scrolling cards:
  - *Technicals* — chart, indicators, S/R, swing/EMA (current cards 01, 07, 11)
  - *Positioning* — short interest + insider + 13F merged (03, 04, 10). One verdict line each ("Shorts falling 6 settlements", "No Form 4s in 90d", "19/20 managers, as-of Jun 30 — 49d stale"), detail inside.
  - *Street & story* — analyst actions + sentiment + news (09, 12, News)
  - *Record* — recommendation history + calibration **with base rate beside every rate** (15; closes the known outstanding item)
- Catalysts move into the hero; the earnings-analysis panel stays button-triggered inside it.

## Cut list (removed outright)

| Cut | Why |
|---|---|
| Midday tab | One daily generation ≠ a tab. Folds into the Today session brief. |
| Scanner presets: Momentum/HOD, Pre-Market Gappers, All Movers | Population mismatch — surfaces $2–12 small caps you don't trade. Golden Cross survives as a setups module in Names. |
| Sectors as a tab (the 11 prose paragraphs) | ~1,100 words/day for near-zero decision yield. Becomes a one-line heat strip; prose on click. |
| IPO calendar | SPACs. No decision you make touches it. |
| Market movers ≥10% + post-market movers | Same junk population as the scanner; replaced by watchlist movers. |
| News cards beyond 3 | 8 cards → top 3 with a one-clause "so what" for your names; rest behind "more". |
| Index strip beyond 6 instruments | ES, NQ, 10Y, VIX, WTI, Gold. Russell/Dow/Silver on expand. |
| Long-tab on-screen legend | → modal. |
| Watchlist columns: RSI, SWING, FWD P/E, SHORT %, UPSIDE, 52W bar, SECTOR, driver chips | → row expander. |
| index.html patterns card ("confidence 87%") | A self-reported pattern confidence rendered as a percentage is exactly what honesty rule 6 bans elsewhere. Reinstate only with a measured hit rate beside it; until then it's decoration that reads as measurement. |
| Options V/OI card as a full table | → summary chips (P/C ratio, top 2 strikes by V/OI) inside the hero's options line; table on demand. |
| Watchlist signals + sector picks as separate verdict surfaces | Become *inputs* to the Today action queue — one verdict stream, one place. |

## What is deliberately kept

- **All honesty machinery** — provenance badges (collapsed to one as-of dot per card, expandable), stale/amber aging, refusal-with-reason states, base-rate-beside-rate, ordinal confidence, the deliberate card-08 scar. Simplification must not delete the epistemics; it consolidates where they render.
- **The six-lane structure** of Long — it's load-bearing analysis; it just stops being the default view.
- **The watchlist one-line calls** ("Buy dips toward $460–480; add above $561, stop $424") — these are the single most decision-shaped strings on the platform and become the spine of both the queue and the Names rows.
- **Worker, crons, KV, all data collection** — untouched. This is a presentation-layer rebuild.

## Round 2 decisions (2026-08-18, after Amber's review)

1. **Earnings timing (BMO/AMC) is a first-class input.** A BMO print means the hold/exit decision deadline is the *prior* session's close and the report day is reaction-only; AMC means the deadline is that day's close. Source: Yahoo `earningsTimestamp` / `earningsTimestampStart` carries the datetime — a time in the 04:00–09:30 ET window classifies BMO, post-16:00 AMC, else "timing unknown → assume the earlier deadline and label the assumption." This folds into the already-open confirmed-vs-estimated-dates item (ARCHITECTURE.md not-yet-done #2): the queue renders `BMO`/`AMC` and `est.` tags, and a gate working from an estimate says so.
2. **Scanner → Radar.** Same engine, new population: universe = S&P 500 + Nasdaq 100 (not "most active"); gates = cap > $10B, price > $20, listed options with real OI, average dollar-volume floor. Sources feeding it: golden-cross sweep over the universe, sector opportunity picks (already produce off-watchlist names), relative-strength-on-red-tape, analyst upgrade stream. Cap: 5 candidates/day, each with a one-line why + one-click add-to-watchlist. Discovery outputs into the same verdict pipeline — a Radar name gets the same one-line-call treatment once adopted.

   > **AS BUILT.** The gates, the 5/day cap and the one-click adopt all shipped as written (3 mover + 2 sector slots, ranked by rvol). **The universe did not**: `/api/radar` sources the Yahoo screeners (`day_gainers`, `most_actives`) plus the banked sector picks, not an S&P 500 + Nasdaq 100 sweep — so the population is "what moved, gated hard" rather than "the index, swept". The `why` is mechanical and rendered verbatim; adoption is read-then-append against the live server list, and grows the sweep universe both dashboards share.
3. **Income sleeve = new quiet tab.** Dividend stocks/ETFs fold in cleanly — same Yahoo stack (dividendYield, exDividendDate, trailing distributions, all free), same verdict format, different cadence. It deliberately does NOT feed the daily action queue except on events: ex-div inside 7d, a cut, payout > 90%, or price entering a stated add zone. Covered-call ETFs (JEPI/JEPQ type) belong here — they answer the "premium selling ROI doesn't justify my time" problem structurally. Tax character flagged per name (qualified vs ordinary) given the tax-minimization goal.

   > **AS BUILT — this one changed the most.** It shipped as **Diversify**, not Income: categorized sleeves (`income` / `cyclical` / `value` / `defensive`), with the category **assigned by the user** on each `income:tickers` entry and never inferred. Income rows carry the full detail; the other three are compact because their detail already lives on the ticker page. The quiet-by-default rule held exactly as written — ex-div ≤ 7d joins Today's calendar, and cut / payoutHigh / inAddZone render a compact events line that is **absent** (empty HTML, not an empty shell) when there is nothing. It never feeds the queue. **Two parts did not ship:** the "same verdict format" — v1 has no verdicts, because no AI touches that path — and the qualified-vs-ordinary tax flag, which is not derivable from anything available and is omitted with its reason on screen rather than estimated. A coverage strip of **counts** was added, labelled *"list coverage, not portfolio weights — position sizes unknown"*, because the app has no position sizes and a percentage would measure nothing.
4. **One verdict store — the freshness bug is a design rule now.** Current behavior: the watchlist renders a cached nightly `analysis:` record while the ticker page runs a fresh `synthesize()` on load, so the two can disagree on the same day. New rule: **one ticker = one verdict record per trading day, one KV key, every surface renders that record with its as-of.** The ticker page revalidates and writes back to the same key; Names re-reads on tab focus and on the existing 30-second staleness timer, so a refresh anywhere propagates everywhere. Two surfaces disagreeing is treated as a rule-5-class provenance bug, not cosmetics.
5. **Click-through navigation.** Names ticker → ticker page; the option-rec cell → Options tab with that name's row expanded (hash deep-links already exist; reuse them: `#ticker/NVDA`, `#options/NVDA`).

   > **AS BUILT, and finished.** Every internal ticker click — Names, the queue, the Today modules, Radar rows, Diversify rows, Options rows — routes to `#ticker/X`. The interim links out to the old terminal, which phases 1–4 used while our own ticker page did not exist, were all retired in phase 5. **One deliberate escape hatch remains**, on the ticker page itself: *"open X in the old terminal ↗"*, for the cards phase 5 did not carry over (chart patterns, the V/OI table, the projection overlay).

## New build = new folder, new repo, shared Worker

Build this as a sibling project and leave trading_dash untouched:

- **New folder + repo**: `C:\dev\decision_dash`, its own git repo and its own GitHub Pages site. No shared files with trading_dash — copy nothing except a seeded CLAUDE.md carrying over the honesty rules, verified constants (CIKs, thresholds), and the working-pattern rules (manual deploy, PowerShell 5.1).
- **Share the existing Worker.** All the data the new frontend needs already comes out of `stock-research-worker` endpoints. The new page is just another consumer. The **only** touch to the existing project is one line: add the new Pages origin to `ALLOWED_ORIGINS` in worker.js (+ the usual manual `npx wrangler deploy`). Both dashboards then run side by side against the same KV/crons — same data, zero duplication, and the old dashboard keeps working as the fallback while the new one matures.
- Cap-budget note: a second frontend adds page-load KV reads against the same 10,000/invocation pool — request-path, per-invocation, so no interaction with the cron `_instr` rule; the per-request numbers in ARCHITECTURE.md stay valid.
- New endpoints (e.g. `/api/queue`, `/api/radar`, `/api/income/batch`) go into the same Worker when they're pure KV assembly; if the Worker file's growth becomes a problem, a second Worker sharing the KV namespace is the later escape hatch — not needed to start.

## Implementation reality check

Nearly everything above is frontend-only: the Worker already computes verdicts, key levels (levelPct/levelKind), cross state, best-candidate expectancy, macro state, and the econ calendar. The action queue is a client-side join over payloads the page already fetches (analysis:, long batch, econ-calendar, movers). Optional later: a `/api/queue` endpoint assembling it from KV (~N+3 binding ops, no new fetches, fits rule #1 trivially). Cutting Scanner/IPO/movers also removes their page-load requests — primeTabs() gets cheaper, not more expensive.

> **AS BUILT — this held.** The action queue is a pure client-side function of
> the shared store plus a PT clock, with no fetching of its own; `/api/queue` was
> never needed and is not built. Three Worker changes *were* required and were
> done as trading_dash tasks: earnings session timing on the watchlist batch
> (`earningsTs` / `earningsSession` / `earningsIsEstimate`), `/api/radar`, and the
> `/api/income/*` endpoints. The Pages origin needed no `ALLOWED_ORIGINS` edit
> after all — `https://ambermlysak.github.io` was already allowlisted, because the
> entry is per-origin and trading_dash shares it.
>
> One structural decision that is not in this document and should be: **the two
> surfaces share ONE data store.** The watchlist sweep (the list, the batch
> chunked at 15, one long batch) runs once per load, and Today, Names, Options and
> the ticker page all render from those same objects. Two tabs disagreeing about a
> verdict is impossible by construction rather than by discipline — which is what
> round-2 decision #4 was actually asking for. Measured: four tab switches cost
> zero fetches.
