# Rate of Change

A TradingView Pine Script v6 pane indicator for Visser-style multi-horizon momentum. It wraps rate-of-change in a z-scored overbought/oversold heatmap, with optional relative benchmark, acceleration (second derivative), swing divergences, and a clean default OB/OS glance view.

Pine version: `@version=6`  
Indicator title in TradingView: **Rate of Change** (shorttitle `ROC`)

## Core formulas

```
ROC(N)     = 100 × (price − price[N]) / price[N]
RelROC     = AssetROC − BenchmarkROC
Accel      = ShortROC_now − ShortROC[max(3, round(shortLen/4))]
Z          = (ROC − SMA(ROC, L)) / Stdev(ROC, L)
OB band    = mean + zHot × stdev
OS band    = mean + zCold × stdev
```

- **ROC** = net displacement over N bars (path-agnostic).
- **Z-score** = whether that displacement is extreme **for this asset** (cross-name heat).
- **Acceleration** = “getting less bad / more bad” (second derivative of short ROC).
- **Relative ROC** = out/underperformance vs a benchmark (default `BITSTAMP:BTCUSD`).

## Features

- **Multi-horizon ROC** — Short (20), Medium (60), Long (100); EMA smooth optional (default 3).
- **Z-scored OB/OS** — default ±2σ over 252 bars; Fixed % and Both modes available.
- **Dynamic bands** — OB/OS thresholds drawn in ROC% space (mean ± z×σ).
- **Heat-colored histogram** — red = overbought, light blue = oversold, dark blue = neutral+, gray = neutral−.
- **Relative benchmark** — asset ROC − benchmark ROC via `request.security` (alerts + badge; off-chart by default).
- **Acceleration** — Δ short ROC; toggleable plot; washout-repair and chase-risk combo alerts.
- **200-MA slope** — ROC of SMA(200); toggleable.
- **Swing divergences** — 5/5 pivots on price vs short ROC.
- **Status badge** — OVERBOUGHT / OVERSOLD / NEUTRAL ±, ROC, Z, vs BM, BM feed health.
- **Alerts** — `alertcondition` dropdown + `alert()` on confirmed last bar.

## Default view (minimal glance)

On by default:

| Element | Role |
| ------- | ---- |
| Short ROC histogram | Primary momentum series |
| Dynamic ±2σ bands | OB/OS thresholds |
| Heat background | Red/blue when extreme |
| Zero line | Sign of short ROC |
| Status badge | One-look regime |

Off by default (still computed for alerts): Medium/Long ROC, Relative ROC line, Acceleration, 200-MA slope, fill-to-zero.

## Settings

| Input | Default | Notes |
| ----- | ------- | ----- |
| Short / Medium / Long ROC Length | 20 / 60 / 100 | Multi-horizon stack |
| ROC Smoothing (EMA) | 3 | 1 = raw |
| Enable Relative Benchmark | on | Absolute ROC still plots if BM dies |
| Benchmark Symbol | BITSTAMP:BTCUSD | Prefer exchange-qualified tickers |
| Heat Measured On | Short Absolute | Or Short Relative / Medium Absolute |
| Overheat Mode | Z-Score | Or Fixed % / Both |
| Z Lookback / Hot / Cold | 252 / +2.0 / −2.0 | On weekly charts try 52–104 lookback |
| Short ROC as Histogram | on | Clean glance bars |
| Status Badge | on | Includes BM feed health row |

### Benchmark suggestions

| Chart | Benchmark |
| ----- | --------- |
| MSTR, MTPLF, crypto beta | `BITSTAMP:BTCUSD` or `BINANCE:BTCUSDT` |
| BTC | `SPY` or `QQQ` |
| TSLA / AI equities | `QQQ` |

## Histogram color legend

| Color | Meaning |
| ----- | ------- |
| Bright red | **Overbought** — z ≥ ~+2 (extreme heat) |
| Light / cyan blue | **Oversold** — z ≤ ~−2 (extreme washout) |
| Dark blue | Neutral **positive** ROC (not extreme) |
| Gray | Neutral **negative** ROC (not extreme) |

**Below zero ≠ oversold.** Gray below zero is normal soft momentum; oversold requires light blue bars or badge `OVERSOLD`.

## Usage

1. Open [Rate_of_Change.pine](Rate_of_Change.pine) and copy the full script.
2. In TradingView, open the Pine Editor.
3. Paste → **Add to chart** (non-overlay pane).
4. Set benchmark for the asset class; leave default view minimal for glance work.
5. Optional: enable Acceleration / Relative ROC under **Default View (toggle extras)**.

Best on **daily** charts for the default lengths. On **weekly**, shorten z-lookback (52–104) so “extreme” reflects the current regime.

### Playbook (book scan)

1. **Z / badge** — is momentum extreme for this name?
2. **Zero side** — net up or net down over short ROC?
3. **Acceleration** (alert or toggle) — improving or fading?
4. **vs BM** — alpha vs beta?
5. **Divergence** — swing confirmation failure?

Stronger repair sketch: **oversold + accel up** (and optional bullish divergence).  
Chase-risk sketch: **overbought + still accelerating**.

## Alerts

Right-click the indicator → Add alert. Named conditions include:

- Overheated ON / OFF
- Oversold ON / OFF
- Short ROC > 0 / < 0
- Relative ROC > 0 / < 0
- Acceleration UP / DOWN
- MA Slope UP / DOWN
- Bullish / Bearish Divergence
- Overheated + Still Accelerating
- Oversold + Accel UP (Less Bad)

Or use **Any alert() function call** for dynamic messages. Prefer **Once Per Bar Close**.

## Stability (blank pane after idle)

Symptom: ROC pane empty, scale collapsed near ±0.5, while price/RSI still render; browser refresh restores plots.

Cause: `request.security` on the benchmark could abort the **entire** script when the secondary feed failed to resolve after idle/reconnect (especially bare `BTCUSD`).

Mitigations in this build:

- `ignore_invalid_symbol=true` on the benchmark security call
- Default benchmark `BITSTAMP:BTCUSD` (exchange-qualified)
- Absolute ROC isolated from benchmark success
- Heat falls back to absolute short ROC if relative BM is dead
- Status badge **BM feed: ok / stale/na** row
- Badge recreated if missing after partial recalc; `merge_cells` once
- `alert()` gated to confirmed last bar to avoid recalc storms

If the pane still blanks with **BM feed: ok**, treat it as a TradingView frontend glitch (toggle indicator eye, then refresh).

## Implementation notes

Pine v6 constraints accounted for:

- No nested function definitions inside `if` blocks
- Helpers stay top-level
- `request.security` always called (Pine consistency) but cannot kill absolute plots when invalid
- Built-in `ta.roc` for the core series

## Installation

```
git clone https://github.com/chriserice/tradingview-scripts.git
```

Or download `Rate_of_Change.pine` and paste it into TradingView's Pine Editor.

## License

This project is licensed under the MIT License. See the [LICENSE](../LICENSE) file for details. The Pine source header uses Mozilla Public License 2.0, which TradingView requires for published scripts.

## Notes

- ROC measures **how far** price moved; RSI measures **how one-sided** the path was. This script is ROC-first with z-score heat.
- Z-score ±2 is heat, not an automatic fade/short signal — strong trends can stay elevated.
- Divergences use 5/5 pivots and lag ~5 bars by design.
- On low timeframes, expect more zero crosses and noisier acceleration; daily/4H is the intended use.
