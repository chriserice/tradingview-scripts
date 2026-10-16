# Money Line: Script & Settings Review

**Date:** 2026-10-16 · **Scope:** `moneyline/MoneyLine.pine` (196 lines), `moneyline/README.md`,
and the Python port/backtests in `~/workspace/moneyline_backtest/`.
**Method:** line-by-line audit of the Pine against the Python port; the port was first verified to
reproduce `RESULTS.md` exactly (below). Settings sweep: Yahoo 1-week bars (split-adjusted, not
dividend-adjusted), max available history to 2026-10-16, grid of ATR length {10, 14, 18, 22, 26} ×
multiplier {2.0, 2.5, 3.0, 3.5} (Donchian anchor = ATR length, matching the Pine preset wiring) ×
cooldown {0, 2, 3, 4, 5, 6} = 120 configs × 10 assets, binary long/flat, positions set at bar close
and applied to the next bar (no lookahead). Code: `fetch_universe.py`, `sweep.py`,
`sweep_detail.py`, `daily_check.py`; full grid in `sweep_results.csv` / `sweep_agg.csv`.

**Composite score** (per asset, percentage points), built for Chris's objective — a regime governor
for a long book, so drawdown control bought cheaply, with a noise penalty:

```
score = drawdown avoided (pp) − CAGR surrendered (pp) − 15 × whipsaw rate
```

Config rankings use the **median score across the 8 full-history assets** (BTC, ASTS, MSTR, META,
SCHX, QQQ, NET, RKLB). Buy-and-hold benchmarks are measured after each config's warmup bar, so BH
figures shift ±2pp between configs. MRLN (1.9 yrs) and IBIT (2.8 yrs) were run but excluded from
all rankings — see §2.5. SPCX (20 weekly bars) was excluded entirely.

---

## 1. Script audit

**Port verification first.** The Python engine reproduces the published BTC weekly numbers to the
decimal on the original data (cooldown-4 binary: 33 flips, 40.5% CAGR, −46.7% MaxDD; three-state:
36.2% / −45.2% — identical to `RESULTS.md` and README §5). The slow-line port is faithful: same
Wilder ATR, Donchian anchor, one-directional latch keyed on the prior bar's EMA-3 source vs the
prior bar's stop, wick flips against the **prior** bar's stop (no lookahead), data-driven direction
init. The Pine's preset wiring is also correct (Conservative 22/3.0/22, Balanced 14/2.5/14,
Aggressive 10/2.0/10; Custom reads the manual inputs), and the three alert events in the code match
README §4. The findings below are where code, port, and docs diverge.

### H1 — HIGH: the shipped fast (caution) line has a cooldown; the backtested one does not

The Pine runs the caution band through the *same* `f_moneyline()` as the slow line, so the fast
line gets the 4-bar cooldown and the confirmation streaks. The Python port's internal fast line
(`engine.py`) applies **neither** — it flips on the first cross. Consequences, BTC weekly:

| Fast line | Flips (2018–2026 data) | Direction agreement with the other |
|---|---|---|
| Port (as backtested) | 147 | — |
| Shipped Pine logic (cooled) | 59 | 78% of bars |

Every published three-state number — the old README §5 results column, and the §3 caution narrative —
was computed with the uncooled fast line and therefore **does not describe the script Chris
actually runs**. The shipped CAUTION state is stickier, with far fewer, longer episodes:

| Caution stats (full history, 8 assets) | Time in CAUTION | Episodes (BTC) | Episodes resolving back to RISK-ON |
|---|---|---|---|
| Port fast line (README's basis) | 10–14% of weeks | 57 | 69–86% (BTC 75%, ASTS 77%) |
| Shipped Pine fast line (cooled) | 11–18% of weeks | 22 | **25–74%** (BTC 50%, **ASTS 25%**) |

README §3's guidance "68–90% of caution episodes resolve back to RISK-ON… the default read is
pullback, sit tight" is true of the backtest port and **false of the shipped script**, where on
ASTS three of four caution episodes degrade into a full RISK-OFF. Direction of the error matters:
the shipped three-state behavior is not worse overall — re-run faithfully (§2.4) it scores *better*
than the port version on most assets in this sample (e.g. BTC three-state composite +15.1 faithful
vs +5.8 port; META +31.9 vs +20.9) — but it is *different*, and the docs describe the wrong one.

### H2 — HIGH: cooldown is off by one bar between the backtest and the Pine

Traced in both implementations and confirmed empirically (minimum flip spacing): the Python engine
with cooldown C blocks flips for C−1 bars after a flip; the Pine with cooldown C blocks for C bars.
**The shipped script at its default cooldown of 4 behaves like engine cooldown 5**, and the
configuration `RESULTS.md` validated ("cooldown-4") is what TradingView would produce with the
input set to **3**. At 22/3.0 this is not cosmetic: engine cd4 median composite −1.3 (rank 42/120)
vs engine cd5 −14.0 (rank 104/120) across the current universe (§2).

### M1 — MEDIUM: the Pine header cites different headline numbers than the README

The script's header comment presents "BTC CAGR 37.6% → 40.5%, max drawdown −63.6% → −46.7%" as the
the headline improvement. Those are the **binary cooldown-4** results from `RESULTS.md` round 1. The shipped
default mode is three-state, whose published BTC numbers are 36.2% / −45.2% (README §5). A reader
of either file alone gets a different headline for the same default settings.

### L1 — LOW: warmup/seed differences between Pine and port (immaterial)

Pine `ta.ema`/`ta.atr` seed differently than pandas `ewm`, and the port starts its loop one bar
later (`first = max(atr_len, hh_len)` vs Pine's first non-na bar). Values converge within a few
bars of the 22-bar warmup; no effect on any statistic reported here.

### L2 — LOW: cosmetic / unverifiable items

- `max_lines_count=500` is declared but the script creates no line objects (harmless).
- The status table's "Distance" is `close − line`: in RISK-OFF it is negative and measures the
  distance back *up* to the flip-back level, while README §3 frames distance as room before a
  bearish flip. Reading is long-side-centric; harmless if known.
- README §1's note "Compilation inside TradingView has not been verified yet" still stands — this
  review could not compile-check Pine v6 either. The code uses no constructs known to break v6,
  but the note should stay until someone pastes it into TradingView.
- Intrabar repainting of wick flips and the in-progress bar's line value is inherent to the
  design and is disclosed for alerts in README §4; the same caveat applies to the status table's
  live values and is not stated there.

---

## 2. Settings sweep — is Conservative 22/3.0 + cooldown 4 the right default?

**No. The multiplier is the lever the current default gets wrong.** A 3.5× ATR multiplier at
length 18–22 with cooldown ~4 dominates the grid on Chris's objective, under every weighting
tested, on full history and on both common windows. The shipped 22/3.0 cd4 ranks 42/120.

### 2.1 Top of the grid (median across BTC, ASTS, MSTR, META, SCHX, QQQ, NET, RKLB; full history)

| Config (len/mult/cd) | Median score | Mean score | Median DD avoided | Median CAGR cost | Flips/yr | Whipsaw |
|---|---|---|---|---|---|---|
| **22 / 3.5 / 4** | **+16.5** | +13.5 | +29.4pp | 3.2pp | 3.0 | 49% |
| 18 / 3.5 / 2 | +12.5 | +10.2 | +24.6pp | 3.1pp | 3.3 | 51% |
| 18 / 3.5 / 4 | +12.2 | +15.3 | +23.2pp | 3.2pp | 2.6 | 39% |
| 18 / 3.5 / 5 | +11.7 | +8.9 | +21.3pp | 4.4pp | 2.6 | 37% |
| 22 / 3.5 / 5 | +11.5 | +7.0 | +20.6pp | 3.8pp | 2.7 | 41% |
| 22 / 3.0 / 4 — **shipped, as backtested** | −1.3 | −0.6 | +17.3pp | 9.0pp | 4.2 | 58% |
| 22 / 3.0 / 5 — **shipped, as Pine runs it** | −14.0 | −9.6 | +8.5pp | 13.2pp | 3.9 | 49% |
| 14 / 2.5 / 4 — Balanced preset | −14.1 | −9.8 | +6.2pp | 10.6pp | 5.1 | 55% |
| 10 / 2.0 / 4 — Aggressive preset | −8.8 | +2.4 | +7.9pp | 6.9pp | 6.0 | 56% |

Ranking robustness: with composite weights varied (whipsaw penalty 0 / 15 / 30; CAGR cost weighted
×2; drawdown weighted ×2; drawdown ignored), **22/3.5 cd4 ranks #1 in all six schemes** and the
top three are always 3.5-multiplier configs; shipped 22/3.0 cd4 ranks 33–59. Window sensitivity
(median composite, binary): from 2018 — shipped +1.4 vs 22/3.5 cd4 **+21.1**; from 2021 — shipped
+0.7 vs **+26.7**. The gap is not a 2014–2017 artifact.

### 2.2 Per-asset detail (binary, full history; DD = max drawdown, BH in parentheses)

**Shipped 22/3.0, engine cd4 (as backtested):**

| Asset | Flips/yr | Whip | CAGR (BH) | DD (BH) | Score |
|---|---|---|---|---|---|
| BTC (11.7y) | 5.1 | 64% | 48.7% (65.0%) | −46.7% (−83.0%) | +10.4 |
| ASTS (6.5y) | 6.4 | 60% | 13.6% (24.5%) | −82.8% (−89.6%) | −13.0 |
| MSTR (12.4y) | 4.8 | 62% | 9.0% (20.1%) | −64.8% (−88.6%) | +3.4 |
| META (12.4y) | 3.4 | 45% | 15.5% (21.2%) | −43.8% (−75.1%) | +18.8 |
| SCHX (12.4y) | 2.9 | 56% | 8.0% (11.5%) | **−30.9% (−27.8%)** | −14.9 |
| QQQ (12.4y) | 3.2 | 42% | 10.8% (17.9%) | −33.6% (−34.6%) | −12.5 |
| NET (6.7y) | 3.6 | 46% | 36.4% (52.1%) | −64.6% (−81.1%) | −6.1 |
| RKLB (5.5y) | 5.8 | 66% | 34.4% (33.8%) | −62.9% (−81.0%) | +8.8 |

**Challenger 22/3.5, engine cd4:**

| Asset | Flips/yr | Whip | CAGR (BH) | DD (BH) | Score |
|---|---|---|---|---|---|
| BTC | 4.5 | 60% | 48.1% (65.0%) | −47.7% (−83.0%) | +9.5 |
| ASTS | 5.2 | 59% | 2.9% (24.5%) | −77.1% (−89.6%) | −18.0 |
| MSTR | 3.4 | 48% | 23.4% (20.1%) | −59.0% (−88.6%) | +25.7 |
| META | 2.3 | 21% | 18.8% (21.2%) | −45.9% (−75.1%) | +23.5 |
| SCHX | 2.3 | 43% | 8.4% (11.5%) | **−22.1% (−27.8%)** | −3.8 |
| QQQ | 2.1 | 38% | 14.6% (17.9%) | **−19.8% (−34.6%)** | +5.7 |
| NET | 2.7 | 50% | 46.9% (52.1%) | −43.0% (−81.1%) | +25.4 |
| RKLB | 4.0 | 64% | 49.1% (33.8%) | −46.5% (−81.0%) | +40.3 |

Two facts worth staring at. (1) At the shipped settings the filter's maximum drawdown on SCHX is
**worse than doing nothing** (−30.9% vs −27.8%) and a wash on QQQ — on the smooth indexes its worst
episode is a whipsaw chop, not a bear market, and it pays ~3.5pp and ~7pp of CAGR for the privilege.
At 3.5× both indexes get real drawdown reduction (−22.1%, −19.8% vs BH −27.8%, −34.6%) at similar
or lower CAGR cost. (2) In binary mode ASTS scores negatively at *both* settings (−13.0 / −18.0):
as a binary in/out switch on ASTS the filter mostly costs. Its ASTS value lives entirely in the
three-state sizing (§2.4).

### 2.3 Per-name character

- **META is the textbook case:** 119 of 120 configs score positive; almost any reasonable setting
  keeps most of the return at ~30pp less drawdown.
- **Smooth compounders want it slow and wide:** NET's best configs are the longest/widest
  (26/3.5/cd6, +51.8); tight settings bleed it (Aggressive preset: −51.3).
- **ASTS resists the filter:** only 6/120 configs score positive in binary mode, and its per-asset
  "best" (22/2.0/cd3) flips 11×/yr at 86% whipsaw — an overfit shape, not a recommendation.
- **SCHX is the hardest name to help:** only 3/120 configs positive; the index's bears are either
  too fast (crashes the latch misses) or chop (whipsaw). 3.5× is what finally makes it positive-ish.
- **Per-asset best configs are scattered and whipsaw-heavy** (BTC's best is 22/2.5 at 71% whipsaw;
  MSTR's is 26/2.0 at 83%) — classic single-asset overfitting. They are evidence *against*
  per-name presets, not for them.

### 2.4 Three-state mode, done faithfully

Three-state composites (1.0/0.5/0 sizing) with the fast line computed the way the Pine actually
does it (full engine, cooled) vs the port's uncooled fast line:

| Config | Mode | BTC | ASTS | MSTR | META | SCHX | QQQ | NET | RKLB |
|---|---|---|---|---|---|---|---|---|---|
| 22/3.0 cd4 | port fast | +5.8 | −6.7 | +1.7 | +20.9 | −15.2 | −6.5 | −1.9 | +7.3 |
| 22/3.0 cd4 | **faithful fast** | +15.1 | −0.8 | +1.7 | +31.9 | −12.5 | −6.3 | −13.9 | +14.9 |
| 22/3.5 cd4 | port fast | +5.5 | −11.6 | +15.3 | +23.5 | −3.6 | +7.8 | +33.2 | +30.5 |
| 22/3.5 cd4 | **faithful fast** | +16.5 | −2.7 | +17.9 | +35.9 | −5.3 | +9.4 | +35.4 | +37.7 |

The 3.5 challenger wins in three-state mode too, and the cooled (real) fast line beats the
backtested one on 13 of 16 cells — H1's divergence cuts in Chris's favor on this sample, but it is
still an unvalidated difference, not a tested design choice.

### 2.5 Short-history names (excluded from rankings)

- **MRLN:** Yahoo history starts Dec 2024 (99 weekly bars; ~1.4 yrs after warmup). Its BH CAGR over
  the scored window is ≈ −75% (a single collapse episode), so *any* filter that sits out part of it
  posts absurd composites (+170 to +197). Meaningless as settings evidence; revisit with 3+ years.
- **IBIT:** 2.8 yrs (one partial cycle). Shipped config scores +34.8, but with 2.4 scored years and
  zero whipsaws observed, treat as anecdote. Note MSBT is read off BTC proper in Chris's system,
  so BTC's rows are the operative ones for that exposure.
- **SPCX:** 20 weekly bars — shorter than the indicator's own warmup. No test possible.

### 2.6 Verdict — default settings

1. **Change the default preset from Conservative (22 / 3.0) to (22 / 3.5).** Same length, same
   cooldown family; the wider leash is what the current universe rewards — roughly +18 to +26
   median composite points over the shipped default depending on window, with fewer flips
   (3.0 vs 4.2/yr median) and lower whipsaw (49% vs 58%). 18/3.5/cd4 is statistically
   indistinguishable (better mean, slightly worse median) — 22 keeps continuity with everything
   already written about the system.
2. **Keep cooldown at the backtested strength.** In engine terms cd4; in TradingView, because of
   H2, that is the input value **3** if the goal is to run what was backtested, or **4** if the
   extra blocked bar is accepted deliberately — at 22/3.5 the cd5-equivalent still ranks #5/120
   (+11.5), so either is defensible at the new multiplier, unlike at 3.0 where the extra bar is
   the difference between rank 42 and rank 104. Pick one, document which, and make the Pine and
   the backtest agree.
3. **A single book-wide default remains defensible.** Per-name "bests" are overfit artifacts
   (§2.3). The honest per-name guidance is qualitative, and the README already half-says it:
   expect the filter to earn its keep on META/MSTR/RKLB-type volatility, to be a pure
   drawdown-insurance cost on NET-type compounders, and to be at its weakest as a binary switch
   on ASTS and on the smooth indexes at 3.0×.

---

## 3. Daily-timeframe check (the production gap)

The dashboard machinery computes this regime on **daily** bars (BTC daily governs the dashboard
Bitcoin block; an ASTS daily pair feeds the trim policy), while README §6 says "Weekly only…
other timeframes were not backtested." That sentence is wrong — `RESULTS.md` round 3 *did* test
daily (2018–2026) — but its conclusions were never folded into the README or the script, and the
daily Sharpes printed there are understated ~2.2× (the old `simulate()` annualized daily returns
with √52; corrected here with a `bars_per_year` parameter: round-3 BTC three-state Sharpe 0.34
becomes ≈0.75, matching this rerun). Fresh run, shipped config, 2018-01-01 → 2026-10-16, no
transaction costs:

| Asset (daily) | Mode | Flips/yr | Whip* | CAGR (BH) | MaxDD (BH) | Sharpe |
|---|---|---|---|---|---|---|
| BTC | binary cd4 | 26.6 | 47% | 29.6% (25.3%) | −52.0% (−81.5%) | 0.69 |
| BTC | three-state (faithful) | 26.6 | 47% | 30.6% (25.3%) | −50.4% (−81.5%) | 0.75 |
| BTC | binary cd5 (= Pine) | 24.1 | 40% | 35.6% (25.3%) | −49.3% (−81.5%) | 0.77 |
| ASTS | binary cd4 | 20.1 | 53% | 32.3% (23.3%) | −67.1% (−91.1%) | 0.71 |
| ASTS | three-state (faithful) | 20.1 | 53% | 30.6% (23.3%) | −59.6% (−91.1%) | 0.70 |
| SCHX | binary cd4 | 20.0 | 53% | 8.5% (12.3%) | −20.6% (−34.3%) | 0.71 |
| SCHX | three-state (faithful) | 20.0 | 53% | 7.0% (12.3%) | −17.1% (−34.3%) | 0.64 |

\* Whipsaw on daily = reversal within 6 *days* (same 6-bar definition) — not comparable to the
weekly 6-week figures.

**Verdict on daily use:** not unvalidated chaos, but not the validated weekly signal either.
On BTC and ASTS the daily regime at shipped settings cut maximum drawdown by ~24–31pp over
2018–2026 and did not cost CAGR in this sample (it gained, pre-cost); on SCHX it bought ~14–17pp
of drawdown reduction for ~4–5pp of CAGR — insurance pricing, roughly as the weekly version
behaves at 3.5×. The price of the faster clock is noise: **14–27 flips/yr (vs ~3–5 weekly) with
~half of them whipsawing**, and every number here is pre-cost and pre-slippage, at 20+ state
changes a year. As a *state read* for a dashboard — which is how production uses it — that noise
means the daily regime flips roughly every 2–3 weeks and should be presented as a fast warning
layer, not with the weekly signal's "regime" authority. What is **not** supported by any evidence
collected: treating a daily flip as equivalent to a weekly flip in policy text. The trim policy's
ASTS daily pair inherits exactly this caveat.

---

## 4. Documentation gap list (README not rewritten — for the post-review pass)

Against the repo as it exists, the audit/sweep above, and the documentation standard Chris set
with the LVBC script's README (what it is → what it is good for → how to properly use it →
settings with evidence → limitations):

1. **§7 Files table is wrong twice.** It lists a README filename that does not exist (the file is `README.md`)
   and `MoneyLine_Bullmania_Style.pine` — the v2 script **does not exist in this repo** (nor in
   `~/workspace/moneyline_backtest/`); there is no v2 Pine file anywhere to reference.
2. **§6's "Other timeframes were not backtested" is false** (`RESULTS.md` round 3 tested daily)
   and contradicts production, which runs the regime daily. The daily evidence and its caveats
   (§3 above) belong in the README.
3. **§3's caution statistics describe the backtest port, not the shipped script** (H1): the
   10–13%-of-weeks and 68–90%-resolve figures reproduce only with the uncooled fast line. Under
   the shipped logic: 11–18% of weeks, 25–74% resolve, ASTS degrades 75% of the time. The
   "amber = healthy pullback, sit tight" guidance needs recalibrating to the shipped behavior.
4. **§5's backtest table is stale and window-bound.** It covers 2018–2026 and six assets; the
   current universe adds QQQ and RKLB and 2014–2017 history changes several headlines (e.g.
   "≈3.8 flips/yr on BTC" for the default is 5.1/yr on full history at engine cd4). Either
   re-run the table on the stated window or label the window everywhere the numbers are quoted,
   including the Pine header (M1).
5. **No settings evidence for the default as-shipped.** §4 documents the presets but not that
   (a) the default's multiplier is beaten by 3.5× on the current book (§2), or (b) the cooldown
   input means different things in TradingView vs the backtest (H2). The settings section needs
   a "what the evidence supports now" note and the cooldown-semantics note.
6. **Missing the LVBC-standard structure.** There is no explicit "what this is good for / what it
   is not good for" summary up top (§6 limitations partially covers the negative side, §3 covers
   use); the per-name character findings (§2.3 — earns on META/MSTR/RKLB, insurance-only on NET,
   weakest as a binary switch on ASTS and smooth indexes) have no home in the doc.
7. **§1 install step 5** says "Right-click the chart → the indicator's ⚙️ settings" — settings
   live on the indicator's own gear icon / legend entry, not the chart right-click menu. Trivial
   but it's the first thing a new installer hits.
8. **`RESULTS.md` (in `~/workspace/moneyline_backtest/`) needs two annotations**: the round-3
   daily Sharpes are understated ~2.2× by the √52 annualization (conclusions unchanged), and its
   three-state results were computed with the uncooled fast line (H1) — future readers will
   otherwise treat them as descriptions of the shipped script.

---

## 5. Reproducibility

- Data: `fetch_universe.py` → `*_weekly_u.csv`, `*_daily_u.csv` (Yahoo chart API, fetched
  2026-10-16; original `*_weekly.csv` files untouched, and the engine still reproduces
  `RESULTS.md` from them exactly).
- Sweep: `sweep.py` → `sweep_results.csv` (120 configs × 10 assets), `sweep_agg.csv`;
  `sweep_detail.py` (sensitivity, per-asset, three-state faithful, common windows);
  `daily_check.py`.
- Engine changes (additive only, defaults preserve prior behavior): `simulate()` gained
  `bars_per_year` (default 52) and `fast_dir_override` parameters. The port's fast-line and
  cooldown semantics were **not** changed — H1/H2 are documented, and the faithful variants are
  computed by composing the existing engine, so old and new results remain comparable.

---

## 6. Changes applied from this review (2026-10-16)

Applied to `MoneyLine.pine` (defaults and comments only; no logic changes) and
`README.md` (full rewrite to the LVBC documentation standard):

- **Default preset is now Conservative (22 / 3.5)** per §2.6 item 1. The preset list
  is now: Conservative (22 / 3.5) [default], Balanced (14 / 2.5), Aggressive
  (10 / 2.0), Conservative Legacy (22 / 3.0) [kept for continuity], Custom. The
  multiplier wiring gained one branch for the Legacy preset; length and anchor
  wiring are behavior-identical to before.
- **Cooldown input default changed 4 → 3** per §2.6 item 2, with a code comment at
  the input documenting the H2 semantics (Pine blocks C bars after a flip, the
  reference engine blocks C−1, so input 3 reproduces the backtested "cooldown 4"
  strength). `f_moneyline()` itself is untouched.
- **H1 deliberately not resolved in code** (per §1 / §2.4): the shipped fast line
  keeps its cooldown. README §5 now documents the divergence as an open design
  choice, with the corrected shipped caution statistics (11–18% of weeks in
  CAUTION; episodes resolving back to RISK-ON 25–74%, BTC 50%, ASTS 25%) replacing
  the port-based figures in the §3 guidance.
- **M1 fixed:** the Pine header no longer cites the binary cooldown-4 CAGR/MaxDD
  figures as the headline; it cites the verified flip-count reduction and points
  to the README for current three-state evidence. The stale "~3.8 flips/yr" header
  figure was replaced with a window-qualified range.
- **README gaps from §4 closed:** files table lists only files that exist
  (`MoneyLine.pine`, `README.md`, `REVIEW.md`, the backtest folder); the false
  "other timeframes were not backtested" claim is replaced by a daily-use section
  carrying the §3 numbers and caveats; caution statistics describe the shipped
  script; the §5 sweep (method, top-of-grid, per-name binary tables, three-state
  faithful table, cooldown semantics) is now the settings evidence, with the
  original 2018–2026 six-asset table kept as labeled history in §7; a
  "what it is good for / how to properly use it" structure was added; the §1
  install step now points at the indicator's own gear icon.
- Not applied (out of scope for this pass, noted in README §8): RESULTS.md still
  needs the two annotations from §4 item 8 (daily Sharpe annualization, uncooled
  fast line in its three-state results). TradingView compilation remains
  unverified (L2).
