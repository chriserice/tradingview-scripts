# Money Line (Bullmania-Style): README

A trend-regime filter for TradingView, in the Chandelier Exit / Supertrend family.
Long-only. Green means full risk, amber means half risk, red means step aside.

> **Origin.** Independent reconstruction of the "Money Line" published by Bullmania /
> Ivan on Tech (bullmania.com/moneyline). The original is closed-source; this is **not**
> the official indicator. The improvements on top of the reconstruction (cooldown,
> three-state caution band) were backtested before being coded. Settings were
> re-reviewed against the current book in October 2026; see
> [Settings and the evidence](#5-settings-and-the-evidence) and `REVIEW.md`.

---

## 1. What it is

Two ATR trailing stops run on the same chart. The slow line (thick) is the primary
trend line: in an uptrend it sits at `highest(high, 22) − 3.5 × ATR(22)` under the
default preset, and it only ever ratchets up, never down. The fast line (thin) is the
same engine on a tighter leash (2.0× ATR). Price relative to the two lines gives three
regime states:

| Where price is | State | Sizing read |
|---|---|---|
| Above both lines | RISK-ON | Full risk budget |
| Below thin, above thick | CAUTION | Half size, do less |
| Below thick | RISK-OFF | Step aside, raise cash |

Think of it as a tripwire system. The thin line is the early-warning tripwire, the
thick line is the wall. Tripping the wire means "pay attention and reduce"; hitting
the wall means "the trend is over."

The states are **position-sizing instructions, not direction calls**. The engine is
long/flat only: red means step aside, never short. It is a regime and permission
layer for a long book, not an entry signal, and it is lagging by design.

---

## 2. What it is good for

- **Governing a risk budget on a long book.** Its whole job is drawdown control
  bought cheaply: sit out the worst stretches, give back as little return as
  possible for the privilege. On the current book the winning settings avoid a
  median ~29 points of drawdown for a median ~3 points of CAGR (see §5).
- **Names with real two-way volatility.** It earns its keep on META, MSTR and
  RKLB-type assets. META is the textbook case: 119 of 120 tested configurations
  score positive, keeping most of the return at roughly 30 points less drawdown.
- **A sizing dial, not just a switch.** Used three-state (full / half / out), it
  does its best work. On ASTS it only earns anything in this mode: as a binary
  in/out switch on ASTS the filter mostly costs (only 6 of 120 configurations
  score positive in binary mode).
- **Drawdown insurance on smooth compounders.** On a clean uptrend name like NET,
  buy-and-hold wins outright on return. The filter's value there is only that the
  ride is survivable: in the original tests it took NET's worst drawdown from
  −81% to about −59%. Expect insurance, not outperformance.

What it is **not** good for:

- Timing tops, bottoms, or V-turns. It will always be late; that is the design.
- Judging a thesis-driven core position. On jump-repricing stories (a binary
  uncertainty resolving: funding, launches, contracts) the line flips bullish
  after most of the move and whipsaws through the digestion. Price trend is not
  business value.
- Smooth indexes at the old 3.0× multiplier. At the previous default the filter's
  maximum drawdown on SCHX was **worse than doing nothing** (−30.9% vs −27.8%
  buy-and-hold) and a wash on QQQ, because its worst episode there was whipsaw
  chop, not a bear market. At 3.5× both indexes get real protection (§5). This is
  why the default multiplier changed.

---

## 3. How to properly use it

### Installation

1. Open [TradingView](https://www.tradingview.com) → **Pine Editor** (bottom panel).
2. Delete the template code, paste the full contents of `MoneyLine.pine`.
3. Click **Add to chart**.
4. Set the chart timeframe to **Weekly (1W)**. That is the timeframe the settings
   evidence in §5 was built on. (Daily bars are a separate question; see §6.)
5. (Recommended) Open the indicator's own settings (the gear icon on the
   indicator's legend entry) and confirm the **Conservative (22 / 3.5)** preset
   is selected.

> Compilation inside TradingView has not been verified yet. If the Pine compiler
> throws an error, copy the exact message; it is fixable.

### Reading the three states

The status table (top-right) always shows the current state. Read it as a sizing
instruction.

**🟢 RISK-ON: full risk budget.** Both lines bullish, trend intact on the wide
and the tight leash. Deploy per standing policy: new positions, full size, normal
leverage rules.

**🟡 CAUTION: half size, do less.** Slow line bullish, fast line bearish: the
trend is bending, not broken. Know the shipped script's actual behavior here,
because it is stickier than the original backtests suggested (the full story is
in §5, "The caution band: shipped vs backtested"):

- In the shipped script, CAUTION occupies roughly **11–18% of weeks**, in fewer,
  longer episodes.
- Only about **25–74% of caution episodes resolve back to RISK-ON** across the
  book (BTC 50%). On ASTS only about **25%** resolve; three of four degrade into
  a full RISK-OFF. So on ASTS, amber is a serious warning, not a shrug.

The default read is still "pullback inside an intact trend, sit at half size",
but do not assume resolution. The status table tells you which flavor you are
looking at:

- **Healthy pullback:** distance to the flip level is wide (1.5+ ATR), slow line
  still rising or flat. The trend is breathing. Do not add aggressively, do not
  cut. Existing positions stay.
- **Trend decay:** distance compressing under ~1 ATR, slow line flattening. The
  red flip is loading. Reduce, initiate nothing new, no leverage.

The two classic mistakes: treating amber as red (selling the pullback; on a
re-rating name that is selling the dip right before the next leg) and treating
amber as green (pressing size into a trend that is actually topping). Amber means
*do less*: no new full-size risk, no panic selling, watch the distance. If the
distance starts expanding again, the episode is resolving; if it keeps
compressing, the flip is coming and you will see it before it happens.

**🔴 RISK-OFF: step aside.** Slow line bearish, trend broken on the primary
timeframe. De-risk: no new longs, raise cash, wheel and put-selling paused.
**Red is long/flat, never a short signal.**

### The two numbers that drive sizing

- **Distance (ATR and %)**: how much adverse move before the flip level breaks.
  Your heat gauge. Wide means room; narrow means fragile. One quirk: in RISK-OFF
  the number is negative and measures the distance back *up* to the flip-back
  level, not room before a bearish flip.
- **Bars Since Flip**: trend maturity. A flip 2 bars old is fragile (hence the
  cooldown); a trend 20 bars old with wide distance is mature and trustworthy.

One live-chart caveat: on the in-progress bar, the line value and wick-based flips
can repaint if price un-crosses before the bar closes. The status table's live
values carry the same caveat as intrabar alerts (§5).

---

## 4. The two lines and flip mechanics

### Thick line: the slow line (primary trend)

- Formula (uptrend, default preset): `highest(high, 22) − 3.5 × ATR(22)`, plotted
  **linewidth 2**.
- It **latches**: while price stays above it, the line only ever ratchets *up*,
  never down. It is the trend's invalidation level; a cross below it flips the
  regime bearish.
- In a downtrend it mirrors: `lowest(low, 22) + 3.5 × ATR(22)`, ratcheting only down.
- The **"Flip Level"** label on the right edge of the chart marks its current value.

### Thin line: the fast line (caution band)

- Same engine with a **2.0× ATR** multiplier, plotted **linewidth 1,
  semi-transparent**.
- A tighter leash: it flips bearish on shallower pullbacks, *while the slow line
  still holds*. That disagreement is the CAUTION state.

### Flip mechanics (the details that matter)

- **Wick-based by default.** A weekly *wick* piercing the line arms the flip, not
  just the close. More responsive, slightly noisier. (Toggle: "Use Wicks for Flip".)
- **Cooldown.** After any flip, opposite flips are ignored for a few bars. This
  was the single biggest backtest win: it kills the flip-flop weeks where the line
  reverses, re-reverses, and re-re-reverses around choppy price action. The
  shipped input default is 3, which is exactly the backtested strength; the
  one-bar semantics quirk behind that is explained in §5.
- **Confirmation source** is a 3-bar EMA of the close, which smooths single-week
  spikes.
- **Markers:** large `bullish` / `bearish` labels mark slow-line flips; small amber
  triangles mark fast-line transitions (▽ = caution flagged, △ = caution lifted).

**Alerts**: three dynamic events: slow-line bullish flip, slow-line bearish flip,
fast-band caution flag. Messages carry the ticker, timeframe, price, and flip
level. Leave **"Alert intrabar" off** (default): intrabar alerts can repaint if
price un-crosses before the weekly close. To wire up: TradingView's ⏰ **Create
Alert** → Condition: *Money Line* → choose the event (e.g. "flipped BEARISH") →
Create.

---

## 5. Settings and the evidence

### Settings reference

**Core Methodology**

| Setting | Default | What it does |
|---|---|---|
| Preset | Conservative (22 / 3.5) | One-click parameter sets (see below) |
| ATR Length / Multiplier / Donchian Anchor (Custom) | 22 / 3.0 / 22 | Manual inputs, used only with the Custom preset |
| Use Wicks for Flip | On | Wicks arm flips (responsive) vs. EMA-close only (quieter) |
| Confirmation Source / Smoothing | close / EMA 3 | The price series the engine evaluates |

**Presets** (all evaluated on weekly bars; evidence below):

| Preset | ATR len / mult | Character |
|---|---|---|
| **Conservative (22 / 3.5)** *(default)* | 22 / 3.5 | The sweep winner. Fewest flips, least whipsaw, real drawdown protection on the whole book |
| Balanced (14 / 2.5) | 14 / 2.5 | Middle ground on paper. Scored poorly in the sweep (median composite −14.1) |
| Aggressive (10 / 2.0) | 10 / 2.0 | Most responsive, most whipsaw. Not recommended on volatile names; it badly bleeds smooth compounders |
| Conservative Legacy (22 / 3.0) | 22 / 3.0 | The pre-October-2026 default, kept for continuity. Beaten clearly by 3.5× (below) |
| Custom | your inputs | Full manual control |

**Noise Controls**

| Setting | Default | Notes |
|---|---|---|
| Flip Cooldown (bars) | 3 | Input 3 in this script = the backtested "cooldown 4" strength. See the semantics note below before changing it |
| Flip Confirmation (bars) | 1 | Tested at 2: consistently harmful (lag cost more than noise saved). Leave at 1 |
| Caution Band (three-state) | On | Uncheck to revert to binary bull/bear |
| Caution Band ATR Multiplier | 2.0 | Tighter = earlier warnings, more amber time |

### The October 2026 settings sweep

The defaults above come from a full re-review (`REVIEW.md`), not from the original
2018–2026 tests. Method: Yahoo weekly bars (split-adjusted), maximum available
history to 2026-10-16, 120 configurations (ATR length {10, 14, 18, 22, 26} ×
multiplier {2.0, 2.5, 3.0, 3.5} × cooldown {0, 2, 3, 4, 5, 6}), binary long/flat,
positions set at bar close and applied to the next bar (no lookahead). Each
configuration is scored per asset with a composite built for a long book's
objective:

```
score = drawdown avoided (pp) − CAGR surrendered (pp) − 15 × whipsaw rate
```

Rankings use the median score across the 8 full-history assets (BTC, ASTS, MSTR,
META, SCHX, QQQ, NET, RKLB). Top of the grid:

| Config (len / mult / cooldown) | Median score | Mean score | Median DD avoided | Median CAGR cost | Flips/yr | Whipsaw |
|---|---|---|---|---|---|---|
| **22 / 3.5 / 4** | **+16.5** | +13.5 | +29.4pp | 3.2pp | 3.0 | 49% |
| 18 / 3.5 / 2 | +12.5 | +10.2 | +24.6pp | 3.1pp | 3.3 | 51% |
| 18 / 3.5 / 4 | +12.2 | +15.3 | +23.2pp | 3.2pp | 2.6 | 39% |
| 18 / 3.5 / 5 | +11.7 | +8.9 | +21.3pp | 4.4pp | 2.6 | 37% |
| 22 / 3.5 / 5 | +11.5 | +7.0 | +20.6pp | 3.8pp | 2.7 | 41% |
| 22 / 3.0 / 4 (old default, as backtested) | −1.3 | −0.6 | +17.3pp | 9.0pp | 4.2 | 58% |
| 22 / 3.0 / 5 (old default, as the script ran it) | −14.0 | −9.6 | +8.5pp | 13.2pp | 3.9 | 49% |
| 14 / 2.5 / 4 (Balanced preset) | −14.1 | −9.8 | +6.2pp | 10.6pp | 5.1 | 55% |
| 10 / 2.0 / 4 (Aggressive preset) | −8.8 | +2.4 | +7.9pp | 6.9pp | 6.0 | 56% |

**Verdict.** The multiplier was the lever the old default got wrong. 22 / 3.5 with
cooldown 4 ranks **#1 of 120 under all six weighting schemes tested** (whipsaw
penalty varied, CAGR cost doubled, drawdown doubled, drawdown ignored) and on both
common windows: from 2018, median composite +21.1 vs the old default's +1.4; from
2021, +26.7 vs +0.7. The gap is not an artifact of the early years. 18 / 3.5 is
statistically indistinguishable (better mean, slightly worse median); 22 was kept
for continuity with everything already written about the system.

### The cooldown semantics note (read this before touching the input)

The cooldown input does not mean the same thing in this script and in the Python
reference engine the backtests were run on. The script blocks flips for **C bars**
after a flip; the engine blocks for **C−1 bars** at the same input value. So:

- The strength the backtests validated ("cooldown 4") is input **3** in
  TradingView. That is now the shipped default.
- The old shipped default of 4 was silently running one bar stronger than
  anything backtested. At 22 / 3.0 that mattered a lot: engine cooldown 4 scored
  −1.3 (rank 42/120) vs cooldown 5 at −14.0 (rank 104/120).
- At the new 3.5× multiplier the extra bar is no longer a cliff: the
  cooldown-5-equivalent still ranks #5/120 (+11.5). So input 4 is defensible at
  3.5× if you choose it deliberately. What is not defensible is not knowing
  which strength you are running.

### Per-name results (binary mode, full history)

Buy-and-hold in parentheses. DD = maximum drawdown.

**Old default 22 / 3.0 (engine cooldown 4, as originally backtested):**

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

**Current default 22 / 3.5 (engine cooldown 4):**

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

Two things to stare at. First, the indexes: at 3.0× the filter's worst drawdown on
SCHX was worse than buy-and-hold and QQQ was a wash, paid for with ~3.5 and ~7
points of CAGR respectively. At 3.5× both get real drawdown reduction at similar
or lower return cost. Second, ASTS in binary mode scores negatively at *both*
multipliers. A single book-wide default stays defensible because per-name "best"
settings are scattered, whipsaw-heavy overfits (BTC's best is 22 / 2.5 at 71%
whipsaw; MSTR's is 26 / 2.0 at 83%). Treat per-name tuning as a story you tell
yourself, not a setting.

More per-name character:

- **META is benign under almost anything:** 119 of 120 configurations positive.
- **Smooth compounders want it slow and wide:** NET's best configurations are the
  longest and widest tested (26 / 3.5, cooldown 6, score +51.8); tight settings
  bleed it (the Aggressive preset scores −51.3 on NET).
- **SCHX is the hardest name to help:** only 3 of 120 configurations score
  positive. Its bears are either too fast for the latch or pure chop. The 3.5×
  multiplier is what finally gets it close to positive.
- **Short-history names prove nothing yet.** MRLN (history from Dec 2024, ~1.9
  years) posts absurd composites (+170 to +197) because its buy-and-hold window
  is one collapse episode; ignore them as settings evidence. IBIT (2.8 years, one
  partial cycle) scores +34.8 at the old default, but treat that as anecdote.
  SPCX has 20 weekly bars, shorter than the indicator's own warmup, so no test
  is possible. (MSBT exposure is read off BTC proper in this system, so the BTC
  rows are the operative ones.)

### Three-state results, computed the way the script actually runs

Composite scores, 1.0 / 0.5 / 0 sizing, full history:

| Config | Mode | BTC | ASTS | MSTR | META | SCHX | QQQ | NET | RKLB |
|---|---|---|---|---|---|---|---|---|---|
| 22 / 3.0 cd4 | backtest-port fast line | +5.8 | −6.7 | +1.7 | +20.9 | −15.2 | −6.5 | −1.9 | +7.3 |
| 22 / 3.0 cd4 | **shipped fast line** | +15.1 | −0.8 | +1.7 | +31.9 | −12.5 | −6.3 | −13.9 | +14.9 |
| 22 / 3.5 cd4 | backtest-port fast line | +5.5 | −11.6 | +15.3 | +23.5 | −3.6 | +7.8 | +33.2 | +30.5 |
| 22 / 3.5 cd4 | **shipped fast line** | +16.5 | −2.7 | +17.9 | +35.9 | −5.3 | +9.4 | +35.4 | +37.7 |

The 3.5× default wins in three-state mode too, and ASTS improves from −18.0
(binary, 3.5×) to −2.7 (three-state, shipped): still slightly negative, but its
value clearly lives in the half-size state rather than the in/out switch.

### The caution band: shipped vs backtested (an open design choice)

This needs saying plainly, because the original documentation got it wrong. The
shipped script runs the fast (caution) line through the *same* engine as the slow
line, cooldown and confirmation streaks included. The Python backtest engine's
fast line applied **neither**: it flipped on the first cross. Every three-state
number published before October 2026, including the history table in §7 and the
caution narrative in earlier versions of this README, was computed with the
uncooled fast line and therefore did not describe the script you actually run.

The shipped behavior, measured:

| Caution stats (full history, 8 assets) | Backtest port (uncooled) | Shipped script (cooled) |
|---|---|---|
| Fast-line flips, BTC | 147 | 59 |
| Time in CAUTION | 10–14% of weeks | 11–18% of weeks |
| Caution episodes, BTC | 57 | 22 |
| Episodes resolving back to RISK-ON | 69–86% (BTC 75%, ASTS 77%) | **25–74%** (BTC 50%, ASTS 25%) |

So the shipped CAUTION state is stickier: fewer, longer episodes, and much more
likely to be a real trend change in progress, especially on ASTS. The old guidance
"most caution episodes resolve, sit tight" was true of the port and false of the
script; §3 above is calibrated to the shipped numbers.

The twist: re-run faithfully, the shipped (cooled) version scores *better* than
the backtested one on 13 of 16 asset/configuration cells in this sample (BTC
three-state composite +15.1 faithful vs +5.8 port; META +31.9 vs +20.9). The
divergence cuts in the user's favor here, but it remains an **open design
choice**: the cooled fast line was never a tested decision, it fell out of code
reuse. It is documented here, deliberately not silently "fixed" in either
direction. If that ever changes, this section and the three-state table need a
re-run, not an edit.

---

## 6. Daily use: what the evidence supports

Production machinery around this system computes the regime on **daily** bars
(the BTC daily state governs a dashboard Bitcoin block; an ASTS daily pair feeds
a trim policy), so the review tested daily properly. Earlier documentation said
other timeframes "were not backtested"; that was wrong. Daily was tested in the
original work (its Sharpe ratios were understated ~2.2× by an annualization bug,
since corrected; conclusions unchanged) and was re-run fresh for the review:
shipped configuration, 2018-01-01 to 2026-10-16, no transaction costs.

| Asset (daily) | Mode | Flips/yr | Whip* | CAGR (BH) | MaxDD (BH) | Sharpe |
|---|---|---|---|---|---|---|
| BTC | binary, engine cd4 | 26.6 | 47% | 29.6% (25.3%) | −52.0% (−81.5%) | 0.69 |
| BTC | three-state (shipped fast line) | 26.6 | 47% | 30.6% (25.3%) | −50.4% (−81.5%) | 0.75 |
| BTC | binary, cd5 (= script input 4) | 24.1 | 40% | 35.6% (25.3%) | −49.3% (−81.5%) | 0.77 |
| ASTS | binary, engine cd4 | 20.1 | 53% | 32.3% (23.3%) | −67.1% (−91.1%) | 0.71 |
| ASTS | three-state (shipped fast line) | 20.1 | 53% | 30.6% (23.3%) | −59.6% (−91.1%) | 0.70 |
| SCHX | binary, engine cd4 | 20.0 | 53% | 8.5% (12.3%) | −20.6% (−34.3%) | 0.71 |
| SCHX | three-state (shipped fast line) | 20.0 | 53% | 7.0% (12.3%) | −17.1% (−34.3%) | 0.64 |

\* Daily whipsaw = reversal within 6 *days* (same 6-bar rule). Not comparable to
the weekly 6-week figures.

**Honest status.** Daily is not unvalidated chaos, and it is not the weekly
signal either. On BTC and ASTS the daily regime cut maximum drawdown by ~24–31
points over 2018–2026 and did not cost CAGR in this sample (it gained, before
costs). On SCHX it bought ~14–17 points of drawdown reduction for ~4–5 points of
CAGR: insurance pricing, roughly how the weekly version behaves at 3.5×.

The cost is noise. Daily flips **14–27 times a year** (vs ~3–5 weekly) with
roughly half of them whipsawing, and every figure above is before transaction
costs and slippage at 20+ state changes a year. The daily state flips roughly
every 2–3 weeks, which is not a "regime" in the weekly sense.

**How to treat it:** daily is defensible as a **fast warning layer**, a state
read that says conditions are deteriorating or improving right now. It is **not**
equivalent to a weekly flip, and no evidence collected supports giving a daily
flip the same weight in policy text. Any policy wired to a daily state (the ASTS
trim pair included) inherits that caveat.

---

## 7. Backtest history (2018–2026, original six assets)

The original tests, kept for the record. Weekly bars, 2018–2026, long/flat,
positions change after the signal bar closes (no lookahead). Whipsaw = signal
reversing within 6 bars. "Before" is the plain reconstruction with no noise
controls; "After" adds the cooldown and the three-state band. Two caveats: this
window is shorter than the §5 sweep's full history, and the "After" three-state
numbers were computed with the backtest port's uncooled fast line (see §5), so
they describe that variant, not the shipped script exactly.

| Asset | Before flips (whip) | After flips (whip) | Before CAGR / MaxDD | After CAGR / MaxDD | Buy&hold CAGR / MaxDD |
|---|---|---|---|---|---|
| BTC | 67 (79%) | 33 (58%) | 37.6% / −63.6% | 36.2% / −45.2% | 35.9% / −80.3% |
| ASTS | 63 (84%) | 31 (65%) | −16.7% / −82.1% | 8.8% / −74.2% | 37.5% / −84.1% |
| SCHX (SPX proxy) | 34 (65%) | 24 (46%) | 10.2% / −25.0% | 6.8% / −23.0% | 13.0% / −32.3% |
| MSTR | 110 (85%) | 44 (59%) | 31.8% / −71.2% | 29.7% / −61.3% | 36.2% / −86.3% |
| NET | 38 (68%) | 24 (46%) | 49.6% / −60.3% | 38.0% / −58.5% | 57.1% / −81.1% |
| META | 44 (61%) | 30 (40%) | 18.0% / −43.4% | 16.2% / −42.0% | 17.9% / −76.0% |

What that round established:

1. **The plain reconstruction is far noisier than marketed.** The "handful of
   trades per year" claim does not survive contact with volatile assets: MSTR
   flipped ~13×/year, BTC ~7.7×/year, with 79–85% of flips whipsawing. Flip rate
   scales with idiosyncratic volatility.
2. **The cooldown was the dominant fix**: best return profile on BTC, ASTS and
   MSTR (on MSTR the cooldown-only variant even beat buy-and-hold on return:
   39.8% vs 36.2%, with lower drawdown).
3. **On smooth compounders (NET), buy-and-hold wins outright.** The filter's
   value there is drawdown control only (−81% to −59%). Do not expect it to beat
   a clean uptrend; expect it to make the ride survivable.
4. **Rejected then, still rejected:** 2-bar confirmation (lag hurt everywhere),
   volatility-adaptive multiplier (mixed to negative).

Full engine, data and results: `~/workspace/moneyline_backtest/` (`RESULTS.md`,
plus the sweep files from the October 2026 review: `sweep_results.csv`,
`sweep_agg.csv`, and the `sweep.py` / `sweep_detail.py` / `daily_check.py`
scripts that produced §5 and §6).

---

## 8. Limitations: how NOT to use it

- **It is lagging by design.** It will always be late to V-turns and will never
  catch exact tops or bottoms. It is a regime/permission layer, not an entry
  signal.
- **It cannot time re-ratings.** On jump-repricing stories the line flips bullish
  after most of the move and whipsaws through the digestion. Use it on broad
  benchmarks (BTC, SCHX) to govern the overall risk budget, not on a
  thesis-driven core position to decide whether the thesis is intact.
- **Red means step aside, not short.** The engine is long/flat only.
- **Weekly is the validated timeframe.** Daily has now been tested too (§6), and
  the honest summary is "defensible warning layer, not a weekly-equivalent
  signal, with real noise costs." Other timeframes remain untested.
- **The caution band carries an open design choice** (§5): the shipped fast line
  is cooled, the originally backtested one was not. Know which behavior the
  numbers you are quoting came from.
- **Short histories prove nothing.** Anything said about MRLN, IBIT or SPCX
  settings is anecdote until they have 3+ years of tape.
- **Live values can repaint.** On the in-progress bar the plotted line and
  wick-triggered flips can change before the close; the status table's live
  read carries the same caveat as intrabar alerts.
- **TradingView compilation is still unverified.** The code uses no constructs
  known to break Pine v6, but nobody has pasted this exact file into TradingView
  and compiled it as part of these reviews.
- **Not financial advice.** Regime states are conditions, not orders.

---

## 9. Files

| File | What |
|---|---|
| `MoneyLine.pine` | The indicator: paste into TradingView Pine Editor |
| `README.md` | This file |
| `REVIEW.md` | The October 2026 script and settings review this README's §5 and §6 are drawn from |
| `~/workspace/moneyline_backtest/` | Python port, backtest engine, `RESULTS.md`, and the 2026 sweep data and scripts |

*Dashboard integration: a morning pull script computes this regime (daily bars
for BTC and the ASTS pair, per §6's caveats) and feeds the Horizon dashboard.*
