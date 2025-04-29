# Extended Price Indicator

A TradingView Pine Script indicator that plots a red dot above bars when the selected signal price (default: High) exceeds a user-defined multiplier times a moving average (SMA or EMA). Includes alert functionality to notify when the signal price crosses the threshold.

## Features
- **Adjustable Moving Average**: Choose between Simple Moving Average (SMA) or Exponential Moving Average (EMA) with a customizable period (default: 200).
- **Flexible Signal Threshold**: Compare the threshold against the bar’s Close, High (default), Low, or Open price.
- **Customizable Multiplier**: Set the multiplier for the moving average threshold (e.g., 3.0 for 3x MA, default: 3.0).
- **Alerts**: Receive notifications when the signal price crosses above the threshold.
- **Visual Indicator**: Red dots plotted above the bar’s high with an ATR-based offset for visibility.

## Usage
1. Copy the script from `extended_price_indicator.pine`.
2. Open TradingView and navigate to the Pine Editor (bottom panel).
3. Paste the script and click “Add to Chart.”
4. Adjust settings in the indicator’s dialog:
   - **MA Period**: Number of bars for the moving average (e.g., 50, 200).
   - **Moving Average Type**: Choose `SMA` or `EMA`.
   - **Multiplier**: Set the threshold multiplier (e.g., 2.0, 3.0).
   - **Signal Threshold**: Select `Close`, `High` (default), `Low`, or `Open`.
5. To set alerts:
   - Click the “Alerts” icon in TradingView or right-click the chart and select “Add Alert.”
   - Choose “Extended Price Indicator” and select “Price Crosses MA Threshold.”
   - Set frequency (e.g., “Once Per Bar Close”) and notification method (e.g., email, app).
   - Click “Create.”

## Example
The indicator plots a red dot above a bar when the High price exceeds 3x the 200-day SMA. An alert might read: “Price High crossed above 3.0x SMA(200) at 150.25.”

## Installation
- Clone this repository: `git clone https://github.com/your-username/extended-price-indicator.git`
- Or download `extended_price_indicator.pine` and paste it into TradingView’s Pine Editor.

## License
This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

## Contributing
Feel free to submit issues or pull requests for improvements, such as additional moving average types or signal thresholds.

## Notes
- Test the indicator on volatile symbols or higher timeframes (e.g., daily) to ensure the signal price reaches the threshold.
- Adjust the ATR offset (in the script) if dots are too close or far from the bars.