# Beardo Pivot

A TradingView Pine Script v6 overlay that plots quarterly floor-trader pivot points (PP, R1, R2, S1, S2) from the previous quarter's High, Low, and Close. Current-quarter levels are solid rays spanning this quarter; previous-quarter levels are dashed and confined to last quarter so you can see confluence.

Pine version: `@version=6`

## Formulas

From the prior quarter's High (H), Low (L), and Close (C):

```
PP = (H + L + C) / 3
R1 = (2 × PP) − L
R2 = PP + (H − L)
S1 = (2 × PP) − H
S2 = PP − (H − L)
```

PP is the quarter's equilibrium. Price above PP is a bullish bias; price below PP is a bearish bias. R1/R2 are high-timeframe resistance and long targets. S1/S2 are high-timeframe support and short targets.

Worked QQQ Q2 2026 example (H 748.65, L 571.92, C 736.40):

| Level | Value  |
| ----- | ------ |
| PP    | 685.66 |
| R1    | 799.39 |
| R2    | 862.39 |
| S1    | 622.66 |
| S2    | 508.93 |

## Features

- **Auto or Manual source** — Auto pulls prior-quarter H/L/C via `request.security(..., "3M")` on any ticker. Manual accepts hardcoded H/L/C (pre-loaded with the QQQ Q2 2026 values above).
- **Quarter-pinned lines** — current-quarter levels start at the first bar of this quarter and extend right; previous-quarter levels span only last quarter.
- **Previous-quarter overlay** — dimmed dashed "prev PP / R1 / R2 / S1 / S2" for confluence with this quarter's levels. On by default.
- **Info table** — Level / Price / Delta vs current close. Off by default; position top-right or bottom-right.
- **Background bias tint** — teal above PP, red below. Off by default.
- **Alerts** — `alertcondition` on close crossing PP, R1, R2, S1, or S2.
- **Data window** — all ten levels (current + previous) available as plots.

## Settings

| Input | Default | Notes |
| ----- | ------- | ----- |
| Source | Auto (Previous Quarter) | Switch to Manual to lock H/L/C |
| Previous Quarter High / Low / Close | 748.65 / 571.92 / 736.40 | Used only when Source = Manual |
| Show PP, R1, R2, S1, S2 | on | Per-level visibility |
| Show Previous Quarter Pivots | on | Dashed dimmed overlay |
| Show Bullish/Bearish Background | off | PP bias tint |
| Show Info Table | off | Level / Price / Delta |
| Table Position | Top Right | Or Bottom Right |
| PP / Resistance / Support colors | white / red / teal | |
| Line Width | 2 | 1–4 |

## Usage

1. Open [beardo_pivot.pine](beardo_pivot.pine) and copy the full script.
2. In TradingView, open the Pine Editor.
3. Paste the script and click **Add to chart**.
4. Optional: Settings → Style to toggle previous-quarter overlay, table, and background.

Best on daily or weekly charts. Confirm bias on weekly/daily, time entries on 4H/1H. Do not trade a touch in isolation — wait for a rejection candle, failed break, or confirmed reclaim.

### Bias and targets

- Above PP: look for longs on pullbacks to PP or S1; targets R1 then R2.
- Below PP: look for shorts on rallies to PP or R1; targets S1 then S2.
- Level flip: R1 breaks and holds as support → long toward R2. Same in reverse for S1.

### Covered calls (wheel)

If you are long the underlying, R1 and R2 are natural covered-call strikes because they are structural resistance, not arbitrary OTM picks.

- Conservative: sell calls near R1 while price is between PP and R1. Roll up to R2 if a weekly close holds above R1.
- Aggressive: after R1 flips to support, sell calls near R2.
- Prefer 1–2 strikes above the pivot so price can tag the level without immediately threatening assignment. Check IV rank and GEX walls before selling.

## Alerts

Right-click the indicator → Add alert. Conditions:

- Price crosses PP
- Price crosses R1
- Price crosses R2
- Price crosses S1
- Price crosses S2

Use "Once Per Bar Close" to avoid wick noise.

## Implementation notes

Lines are anchored with chart `bar_index`, not 3M timestamps. A 1 ms time delta (`t` to `t+1`) maps to the same weekly/monthly bar; Pine then treats `x1 == x2` as a vertical line. Current-quarter `x2` is always `bar_index + 1` so the ray stays horizontal on every timeframe. See [TradingView: vertical lines](https://www.tradingview.com/pine-script-docs/faq/visuals/).

Pine v6 constraints this script already accounts for:

- No `var` parameters on user-defined functions (CE10013)
- No nested function definitions inside `if` blocks (CE10156)
- `is_new_quarter()` is evaluated on every bar, not inside a short-circuit `or`

## Installation

Clone this repository:

```
git clone https://github.com/chriserice/tradingview-scripts.git
```

Or download `beardo_pivot.pine` and paste it into TradingView's Pine Editor.

## License

This project is licensed under the MIT License. See the [LICENSE](../LICENSE) file for details. The Pine source header uses Mozilla Public License 2.0, which TradingView requires for published scripts.

## Notes

- Quarterly pivots reset on the first bar of January, April, July, and October.
- Auto mode needs enough history for at least two completed 3M bars; otherwise use Manual.
- Confluence (a current level sitting on a previous-quarter level, a round strike, or a GEX wall) is materially stronger than any single level alone.
