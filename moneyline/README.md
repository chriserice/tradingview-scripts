# Money Line v3 (Bullmania-Style) — README

A trend-regime filter for TradingView, in the Chandelier Exit / Supertrend family.
Long-only, weekly timeframe. Green means full risk, amber means half risk, red means step aside.

> **Origin.** Independent reconstruction of the "Money Line" published by Bullmania /
> Ivan on Tech (bullmania.com/moneyline). The original is closed-source; this is **not**
> the official indicator. v3 adds improvements that were backtested before being coded —
> see [Backtests](#backtests) below.

---

## 1. Installation

1. Open [TradingView](https://www.tradingview.com) → **Pine Editor** (bottom panel).
2. Delete the template code, paste the full contents of `MoneyLine_v3.pine`.
3. Click **Add to chart**.
4. Set the chart timeframe to **Weekly (1W)** — that's the timeframe everything was validated on.
5. (Recommended) Right-click the chart → the indicator's ⚙️ settings → confirm the
   **Conservative (22 / 3.0)** preset is selected.

> Compilation inside TradingView has not been verified yet. If the Pine compiler
> throws an error, copy the exact message — it's fixable.

---

## 2. The two lines — thick vs. thin

There are two ATR trailing stops on the chart. They run the same engine; only the
leash length differs.

### Thick line — the slow line (primary trend)

- Formula (uptrend): `highest(high, 22) − 3.0 × ATR(22)`, plotted **linewidth 2**.
- It **latches**: while price stays above it, the line only ever ratchets *up*,
  never down. It is the trend's invalidation level — price closing/piercing below it
  flips the regime bearish.
- In a downtrend it mirrors: `lowest(low, 22) + 3.0 × ATR(22)`, ratcheting only down.
- The **"Flip Level"** label on the right edge of the chart marks its current value.

### Thin line — the fast line (caution band)

- Same engine with a **2.0× ATR** multiplier, plotted **linewidth 1, semi-transparent**.
- A tighter leash: it flips bearish on shallower pullbacks, *while the slow line still holds*.
- That disagreement — slow bullish, fast bearish — is the **CAUTION** state.

### How to read them together

| Where price is | Meaning |
|---|---|
| Above both lines | Trend intact, full risk (RISK-ON) |
| Below thin, above thick | Pullback inside an intact trend (CAUTION) |
| Below thick | Trend broken (RISK-OFF) |

Think of it as a tripwire system: the thin line is the early-warning tripwire, the
thick line is the wall. Tripping the wire means "pay attention and reduce"; hitting
the wall means "the trend is over."

### Flip mechanics (the details that matter)

- **Wick-based by default.** A weekly *wick* piercing the line arms the flip, not just
  the close. More responsive, slightly noisier. (Toggle: "Use Wicks for Flip".)
- **4-bar cooldown.** After any flip, opposite flips are ignored for 4 weekly bars.
  This was the single biggest backtest win — it kills the flip-flop weeks where the
  line reverses, re-reverses, and re-re-reverses around choppy price action.
- **Confirmation source** is a 3-bar EMA of the close, which smooths single-week spikes.
- **Markers:** large `bullish` / `bearish` labels mark slow-line flips; small amber
  triangles mark fast-line transitions (▽ = caution flagged, △ = caution lifted).

---

## 3. The three states — how to interpret them

The status table (top-right) always shows the current state. Read it as a
**position-sizing instruction**, not a direction call.

### 🟢 RISK-ON — full risk budget

Both lines bullish. The trend is intact on both the wide and the tight leash.
Deploy per your standing policy. New positions, full size, normal leverage rules.

### 🟡 CAUTION — half size, do less

Slow line bullish, fast line bearish. **The trend is bending, not broken.**
Across backtests this state appears roughly **one week in eight** (~10–13% of weeks),
and **68–90% of distinct caution episodes resolve back to RISK-ON** — only a tenth to a
third degrade into a full RISK-OFF flip. So the default read is *"pullback, sit
tight at half size"* — but don't assume it. The status table tells you which flavor
you're looking at:

- **Healthy pullback:** distance to the flip level is wide (1.5+ ATR), slow line still
  rising or flat. The trend is breathing. Don't add aggressively, don't cut.
  Existing positions stay.
- **Trend decay:** distance compressing under ~1 ATR, slow line flattening. The red
  flip is loading. Reduce, don't initiate anything new, no leverage.

**The two classic mistakes:** treating amber as red (selling the pullback — on a
re-rating name that's selling the dip right before the next leg) and treating amber
as green (pressing size into a trend that's actually topping). Amber means *do less*:
no new full-size risk, no panic selling, watch the distance. If the distance starts
expanding again, the episode is resolving; if it keeps compressing, the flip is
coming and you'll see it before it happens.

### 🔴 RISK-OFF — step aside

Slow line bearish. The trend is broken on the primary timeframe. De-risk: no new
longs, raise cash, wheel/put-selling paused. **Red is long/flat, never a short signal.**

### The two numbers that drive sizing

- **Distance (ATR and %)** — how much adverse move before the flip level breaks.
  This is your heat gauge. Wide = room; narrow = fragile.
- **Bars Since Flip** — trend maturity. A flip 2 bars old is fragile (hence the
  cooldown); a trend 20 bars old with wide distance is mature and trustworthy.

---

## 4. Settings reference

**Core Methodology**

| Setting | Default | What it does |
|---|---|---|
| Preset | Conservative (22 / 3.0) | One-click parameter sets (see below) |
| ATR Length / Multiplier / Donchian Anchor (Custom) | 22 / 3.0 / 22 | Manual inputs, used only with the Custom preset |
| Use Wicks for Flip | On | Wicks arm flips (responsive) vs. EMA-close only (quieter) |
| Confirmation Source / Smoothing | close / EMA 3 | The price series the engine evaluates |

**Presets** (all backtested on weekly bars):

| Preset | ATR len / mult | Character |
|---|---|---|
| **Conservative (22 / 3.0)** *(default)* | 22 / 3.0 | Fewest flips (~3.8/yr on BTC), maximum noise filtering |
| Balanced (14 / 2.5) | 14 / 2.5 | Middle ground — faster, choppier |
| Aggressive (10 / 2.0) | 10 / 2.0 | Most responsive, most whipsaw — not recommended on volatile names |
| Custom | your inputs | Full manual control |

**v3 Noise Controls**

| Setting | Default | Notes |
|---|---|---|
| Flip Cooldown (bars) | 4 | Backtested optimum on BTC/ASTS/SPX weekly. 0 = v2 behavior |
| Flip Confirmation (bars) | 1 | Tested at 2 — consistently harmful (lag cost > noise saved). Leave at 1 |
| Caution Band (three-state) | On | Uncheck to revert to binary bull/bear |
| Caution Band ATR Multiplier | 2.0 | Tighter = earlier warnings, more amber time |

**Alerts** — three dynamic events: slow-line bullish flip, slow-line bearish flip,
fast-band caution flag. Messages carry the ticker, timeframe, price, and flip level.
Leave **"Alert intrabar" off** (default): intrabar alerts can repaint if price
un-crosses before the weekly close. To wire up: TradingView's ⏰ **Create Alert** →
Condition: *MoneyLine v3* → choose the event (e.g. "flipped BEARISH") → Create.

---

## 5. Backtests

Engine and full results: `~/workspace/moneyline_backtest/` (`RESULTS.md`).
Weekly bars, 2018–2026, long/flat, positions change after the signal bar closes
(no lookahead). Whipsaw = signal reversing within 6 bars.

| Asset | v2 flips (whipsaw) | v3 flips (whipsaw) | v2 CAGR / MaxDD | v3 CAGR / MaxDD | Buy&hold CAGR / MaxDD |
|---|---|---|---|---|---|
| BTC | 67 (79%) | 33 (58%) | 37.6% / −63.6% | 36.2% / −45.2% | 35.9% / −80.3% |
| ASTS | 63 (84%) | 31 (65%) | −16.7% / −82.1% | 8.8% / −74.2% | 37.5% / −84.1% |
| SCHX (SPX proxy) | 34 (65%) | 24 (46%) | 10.2% / −25.0% | 6.8% / −23.0% | 13.0% / −32.3% |
| MSTR | 110 (85%) | 44 (59%) | 31.8% / −71.2% | 29.7% / −61.3% | 36.2% / −86.3% |
| NET | 38 (68%) | 24 (46%) | 49.6% / −60.3% | 38.0% / −58.5% | 57.1% / −81.1% |
| META | 44 (61%) | 30 (40%) | 18.0% / −43.4% | 16.2% / −42.0% | 17.9% / −76.0% |

**What the data says:**

1. **v2 is far noisier than marketed.** The "handful of trades per year" claim doesn't
   survive contact with volatile assets — MSTR flipped ~13×/year, BTC ~7.7×/year,
   with 79–85% of flips whipsawing. Flip rate scales with idiosyncratic volatility.
2. **The 4-bar cooldown was the dominant fix** — best return profile on BTC, ASTS,
   and MSTR (on MSTR it even beat buy-and-hold: 39.8% vs 36.2%).
3. **The three-state caution band gave the best drawdown on nearly every asset.**
   That's the amber state's entire job, and it works.
4. **On smooth compounders (NET), buy-and-hold wins outright.** The filter's value
   there is drawdown control only (−81% → −59%). Don't expect it to beat a clean
   uptrend — expect it to make the ride survivable.
5. **Rejected:** 2-bar confirmation (lag hurt everywhere), volatility-adaptive
   multiplier (mixed/negative).

---

## 6. Limitations — how NOT to use it

- **It's lagging by design.** It will always be late to V-turns and will never catch
  exact tops or bottoms. It's a regime/permission layer, not an entry signal.
- **It can't time re-ratings.** On jump-repricing stories (a binary uncertainty
  resolving — funding, launches, contracts), the line flips bullish *after* most of
  the move and whipsaws through the digestion. Use it on broad benchmarks (BTC,
  SCHX) to govern the overall risk budget — not on a thesis-driven core position
  to decide whether the thesis is intact. Price trend ≠ business value.
- **Red means step aside, not short.** The engine is long/flat only.
- **Weekly only, until proven otherwise.** Other timeframes were not backtested.
- **Not financial advice.** Regime states are conditions, not orders.

---

## 7. Files

| File | What |
|---|---|
| `MoneyLine_v3.pine` | The indicator — paste into TradingView Pine Editor |
| `MoneyLine_v3_README.md` | This file |
| `MoneyLine_Bullmania_Style.pine` | The original v2 reconstruction (untouched reference) |
| `~/workspace/moneyline_backtest/` | Python port, backtest engine, full results |

*Dashboard integration: a morning pull script computes the v3 regime for BTC + SCHX
and feeds the Horizon dashboard's "Market Regime" cell —
`~/workspace/goals/horizon-dashboard/hidden_files/pull-moneyline-regime.py`.*
