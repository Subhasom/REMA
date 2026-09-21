# REMA

**An RSI momentum indicator with EMA/WMA confirmation for TradingView (Pine Script v6)**

| | |
|---|---|
| **Author** | Subhasom Mandal ([github.com/Subhasom](https://github.com/Subhasom)) |
| **Current release** | 2.5.22092026.0225 |
| **Platform** | TradingView, Pine Script v6, separate-pane indicator (draws markers and candle colours on the price chart) |
| **Files** | Indicator: `REMA.2.5.22092026.0225.pine`<br>Strategy: `REMA.STR.2.5.22092026.0225.pine` — "REMA Strategy"; same signals, adds orders and alert() messages<br>TradingView description: `REMA_TradingView_About.txt` |

> ⚠️ **Disclaimer:** REMA is an analysis tool, not financial advice. Past signals do not guarantee future results. Always test on your own instruments and timeframes, and use proper risk management.

---

## Table of contents

1. [What REMA does](#1-what-rema-does)
2. [Installation](#2-installation)
3. [Reading the chart](#3-reading-the-chart)
4. [How it works: full logic](#4-how-it-works-full-logic)
5. [Parameter reference](#5-parameter-reference)
6. [How to use it: practical guide](#6-how-to-use-it-practical-guide)
7. [Alerts](#7-alerts)
7b. [The strategy build (REMA.STR)](#7b-the-strategy-build-remastr)
8. [Limitations and honest notes](#8-limitations-and-honest-notes)
9. [Release versioning](#9-release-versioning)
10. [Credits and licensing](#10-credits-and-licensing)
11. [Implementation notes](#11-implementation-notes)

---

## 1. What REMA does

REMA reads trend and momentum from a single RSI by layering two moving averages on top of it:

- **RSI against the 50 level.** A 9-period RSI is plotted against a 50 midline. Above 50 the market has an upward bias; below 50 it has a downward bias.
- **Price line.** A **3 EMA of RSI** (green) follows the RSI closely and shows short-term momentum. When it hugs the RSI, momentum is active.
- **Strength line.** A **21 WMA of RSI** (red) is slow and decides the major trend. Its position relative to the RSI matters most.
- **Trade markers.** When the RSI, Price line and Strength line line up on one side of 50, REMA marks an entry (**Buy** or **Short**). When the alignment breaks, it marks the exit (**Sell** or **Cover**).
- **Chart integration.** Markers are drawn in the REMA pane and on the price chart, and chart candles are recoloured at 50% transparency so the markers stand out.

---

## 2. Installation

1. Open [TradingView](https://www.tradingview.com) and any chart.
2. Open the **Pine Editor** tab at the bottom of the screen.
3. Delete the template code, then paste the full contents of `REMA.<master>.<release>.DDMMYYYY.HHMM.pine` (currently `REMA.2.5.22092026.0225.pine`).
4. Click **Save**, then **Add to chart**.

> 💡 **When updating to a new release**, remove the old indicator from the chart and add the new one. TradingView remembers old settings (especially Style-tab checkboxes), so new defaults (for example, Short and Cover hidden) only apply to a freshly added indicator.

REMA opens in its own pane below the price chart. The price-chart markers and candle colours are drawn from that pane, so nothing needs to be added to the main chart separately.

---

## 3. Reading the chart

### In the REMA pane

| Element | Look | Meaning |
|---|---|---|
| **RSI - Plot** | Black line, width 2 | RSI of the source over the chosen length (default 9). |
| **Line -50** | Thin green line at 50 | The bias midline. Above = upward bias, below = downward bias. |
| **RSI Fill** | Shading between RSI and 50 | Red while RSI is above 50, aqua while below, at 50% transparency. |
| **Strength** | Red line | 21 WMA of RSI. Slow line for the major trend. |
| **Price** | Green line | 3 EMA of RSI. Fast line for short-term momentum. |
| **Buy** | Green triangle ▲ at the bottom, "BUY" | Long entry. |
| **Sell** | Red triangle ▼ at the top, "SELL" | Long exit. |
| **Short** | Blue diamond ◆ at the top, "SHORT" | Short entry. Hidden by default; tick it in the **Style** tab. |
| **Cover** | Orange circle ● at the bottom, "COVER" | Short exit. Hidden by default; tick it in the **Style** tab. |

### On the price chart

| Element | Look | Meaning |
|---|---|---|
| **Buy** | Green triangle ▲ below the bar, no text | Long entry. |
| **Sell** | Red triangle ▼ above the bar, no text | Long exit. |
| **Short** | Small blue diamond ◆ above the bar, no text | Short entry. Hidden by default; tick **Short (chart)** in the **Style** tab. |
| **Cover** | Small orange circle ● below the bar, no text | Short exit. Hidden by default; tick **Cover (chart)** in the **Style** tab. |
| **Transparent candles** | Candles tinted at 50% | Up candles in #089981, down candles in #F23645, at the chosen transparency. |

> **Important:** unlike a plain direction-change indicator, REMA's markers **are** trade pairs. Every Sell closes the previous Buy and every Cover closes the previous Short, because the script tracks one position at a time.

---

## 4. How it works: full logic

The pipeline runs on every bar:

```
Price data
   │
   ├─► 1. RSI (length 9) on the source
   │
   ├─► 2. Price line = EMA(RSI, 3)   Strength line = WMA(RSI, 21)
   │
   ├─► 3. Trend conditions (bull / bear alignment)
   │
   ├─► 4. Crosses (Price vs Strength, RSI vs 50)
   │
   ├─► 5. Position state machine ──► ▲ Buy / ▼ Sell / ◆ Short / ● Cover
   │
   └─► 6. Plots, chart markers, candle colours, alerts
```

### 4.1 The three lines

| Line | Formula | Role |
|---|---|---|
| **RSI** | `ta.rsi(source, 9)` | Market strength. Its side of 50 sets the bias. |
| **Price** | `ta.ema(RSI, 3)` | Fast. Shows short-term momentum. |
| **Strength** | `ta.wma(RSI, 21)` | Slow. Decides the major trend. |

The **gap** between the three lines is a rough measure of speed: the wider it is, the faster price tends to move.

### 4.2 Trend conditions

| Condition | Requires |
|---|---|
| **Bullish** | RSI > 50 **and** Strength < RSI **and** Price > Strength |
| **Bearish** | RSI < 50 **and** Strength > RSI **and** Price < Strength |

In words: for a bullish reading, RSI is above 50, the slow Strength line sits *inside* the strength zone (below RSI), and the fast Price line is above Strength. Bearish is the mirror image.

### 4.3 Exit triggers

| Exit | Fires when |
|---|---|
| **Sell** (close long) | Price line crosses **under** Strength, **or** RSI crosses **below** 50 |
| **Cover** (close short) | Price line crosses **over** Strength, **or** RSI crosses **above** 50 |

### 4.4 Position state machine

REMA keeps a single position state: **long**, **short** or **flat**. On each eligible bar:

1. **Exits first.** If long and a Sell trigger fires → **Sell**, go flat. If short and a Cover trigger fires → **Cover**, go flat.
2. **Then entries.** If flat and the Bullish condition holds (and long signals are on) → **Buy**, go long. Otherwise, if flat and the Bearish condition holds → **Short**, go short.

Because exits run before entries, a Sell and a Short (or a Cover and a Buy) can print on the same bar when the market flips.

Short and Cover are always calculated, even when their markers are hidden. This never moves a Buy or Sell: a Buy needs both RSI > 50 and Price > Strength, and the first of those two crosses already fires the Cover, so the Buy lands on the same bar it would if no short had been open. The same holds in reverse for Sell and Short.

With **Confirm signals on bar close** on (the default), the state machine only runs on confirmed bars, so markers never appear and vanish mid-bar.

---

## 5. Parameter reference

### General

| Parameter | Default | Description |
|---|---|---|
| Length | 9 | RSI length. |
| Source | close | Price used for the RSI. |
| Timeframe | Chart | TradingView's built-in timeframe selector. Set a higher timeframe to compute REMA there; gaps are left between higher-timeframe bars. |

### Markers

| Parameter | Default | Description |
|---|---|---|
| Show long signals (BUY / SELL) | ✅ On | Allow long entries and their exits. |
| Also show markers on price chart | ✅ On | Draw the text-free symbols on the price chart. Pane markers are unaffected. |
| Confirm signals on bar close | ✅ On | Only evaluate signals on confirmed bars. Turn off to see signals intrabar (they can repaint). |

> **Show long signals** changes what trades are tracked, not just what is drawn. Short and Cover have no input: they are always tracked and are shown or hidden from the **Style** tab only.

### Candles

| Parameter | Default | Description |
|---|---|---|
| Transparent chart candles | ✅ On | Recolour the price chart's candles. |
| Up candle | #089981 | Colour for candles where close ≥ open. |
| Down candle | #F23645 | Colour for candles where close < open. |
| Transparency % | 50 | 0 = solid, 100 = invisible. |

### Style tab

Line colours, widths and visibility for **Line -50**, **RSI - Plot**, **RSI Fill**, **Strength**, **Price**, the eight marker plots (four in the pane, four marked "(chart)") and **Candle Colour** can be changed from TradingView's **Style** tab.

| Style-tab plot | Default |
|---|---|
| Buy, Sell, Buy (chart), Sell (chart) | ✅ Checked |
| Short, Cover, Short (chart), Cover (chart) | ⬜ Unchecked |

---

## 6. How to use it: practical guide

**Getting started**

1. Add REMA with default settings. Only Buy and Sell are shown.
2. Check how the Buy/Sell pairs behave on your instrument and timeframe before trusting them.
3. Tick **Short** / **Cover** (pane) and **Short (chart)** / **Cover (chart)** in the **Style** tab only if you trade both directions.

**Suggested starting points**

| Situation | Try |
|---|---|
| Intraday trading | Chart on 5–15 minutes, with REMA's **Timeframe** or a second copy on 30–60 minutes for direction. |
| Positional trading | 1-hour chart. |
| Investing | Daily, weekly or monthly chart. |
| Too many signals in a range | Use a higher timeframe, or only take Buys when a higher-timeframe copy is also bullish. |
| Custom chart colours lost | Set your colours in the **Candles** group, or switch off **Transparent chart candles**. |
| Chart looks cluttered | Turn off **Also show markers on price chart** and read signals from the pane. |

**Good practice**

- Avoid longs while RSI is below 50; look for shorts only while RSI stays below 50.
- Use markers as **context**, not automatic entries. Combine them with support/resistance, VWAP or your own levels.
- Always use a stop. REMA's exits are indicator-based and do not limit loss size.

---

## 7. Alerts

Create alerts from TradingView's **Alert** menu → condition **REMA**:

| Alert | Message | Fires when |
|---|---|---|
| Buy | `REMA ▲ Buy on {{ticker}} at {{close}}` | A Buy (long entry) prints |
| Sell | `REMA ▼ Sell on {{ticker}} at {{close}}` | A Sell (long exit) prints |
| Short | `REMA ◆ Short on {{ticker}} at {{close}}` | A Short (short entry) prints |
| Cover | `REMA ● Cover on {{ticker}} at {{close}}` | A Cover (short exit) prints |

Each message carries the same symbol drawn on the chart, so an alert can be matched to its marker at a glance.

Alerts follow **Show long signals**, because it decides which long trades exist. Short and Cover alerts fire whether or not their markers are ticked in the Style tab, and all alerts ignore **Also show markers on price chart**, which only affects drawing. `alertcondition()` uses the settings saved when the alert was created, so changing a toggle later does not affect a running alert.

Use **"Once Per Bar Close"** so alerts match the confirmed markers.

---

## 7b. The strategy build (REMA.STR)

`REMA.STR` is the same script declared as a `strategy()` instead of an `indicator()`, published as **"REMA Strategy"**. Signals, markers, Style-tab defaults and candle tinting are identical. It adds orders, runtime alerts and a **Trading** input group.

**Orders.** Buy opens a long and Sell closes it. With **Trade short side** on, Short opens a short and Cover closes it. Orders fill at the next bar's open.

| Setting (group **Trading**) | Default | Description |
|---|---|---|
| Enable strategy orders | ✅ On | Off = no orders; markers are unaffected. |
| Enable trade alerts | ✅ On | Off = no alert() messages. |
| Trade short side (orders & alerts) | ⬜ Off | On = Short/Cover also place orders and send alerts. Independent of the Style-tab markers. |

**Creating the alert.** Strategies do not support `alertcondition()`, so messages are sent with `alert()`. Condition = **REMA Strategy**, then choose **"Any alert() function call"**. One alert covers every signal:

```
REMA ▲ Buy on HDFCBANK at 708.25
REMA ▼ Sell on HDFCBANK at 712.40
```

Messages use the same wording and symbols as the indicator's alerts, with the ticker and price filled in by the script.

**Backtest properties** (Settings → Properties): 1,00,000 initial capital, 100% of equity per trade, 0.05% commission per side, 1 tick slippage, no pyramiding, orders filled on the next bar's open.

**No Timeframe option.** TradingView strategies cannot use the `timeframe` parameter, so the strategy always runs on the chart's timeframe.

---

## 8. Limitations and honest notes

- **Signals lag.** Every line is derived from RSI, and the Strength line is a 21-bar average of it. Entries come after the move has started, and exits after it has turned.
- **Choppy markets whipsaw.** When RSI oscillates around 50, entries and exits alternate quickly. A higher timeframe reduces this.
- **No statistics in the indicator.** The indicator shows trade pairs but does not measure profit, costs or winrate. Use the strategy build's Strategy Tester for that, and set commission and slippage to match your broker.
- **Exits are not stops.** A Sell or Cover fires on an indicator cross, which can be far from entry in a fast move.
- **Unconfirmed mode repaints.** With **Confirm signals on bar close** off, a marker can appear during a bar and disappear before it closes.
- **Higher timeframe gaps.** Running REMA on a higher timeframe than the chart leaves gaps between higher-timeframe bars, and signals only appear when those bars close.
- **Candle recolouring overrides chart colours.** While REMA is on the chart, it replaces the chart's own candle colours and may tint wicks and borders as well as bodies.

---

## 9. Release versioning

Releases follow the format:

```
<master release>.<release version>.<DDMMYYYY>.<HHMM>
```

- **Master release** marks a major generation of the script. It changes only when explicitly requested. Currently **2**.
- **Release version** is set when a new release is requested. Builds within the same release are told apart by their date and time.
- Date and time are in IST (UTC+5:30).
- The indicator file is named `REMA.<master>.<release>.<DDMMYYYY>.<HHMM>.pine`, e.g. `REMA.2.5.22092026.0225.pine`.
- The strategy file is named `REMA.STR.<master>.<release>.<DDMMYYYY>.<HHMM>.pine`, e.g. `REMA.STR.2.5.22092026.0225.pine`. It always carries the same release as the indicator it mirrors.

| Release | Date | Summary |
|---|---|---|
| 2.5.22092026.0225 | 22 Sep 2026 | Naming cleanup: header description and code comments reworded. README and TradingView description added. "Show short signals" input removed: Short and Cover are always tracked and are hidden from the Style tab by default; chart diamond and circle made tiny. Strategy build (REMA.STR, "REMA Strategy") added at the same release. |
| 2.5.22092026.0217 | 22 Sep 2026 | Alerts renamed Buy, Sell, Short, Cover, with symbol messages. Markers restyled: Buy ▲ green, Sell ▼ red, Short ◆ blue, Cover ● orange; Short and Cover lose their text on the chart. |
| 2.5.22092026.0209 | 22 Sep 2026 | Transparent chart candles added (50% by default) with a new **Candles** input group. |
| 2.5.22092026.0208 | 22 Sep 2026 | Built from 2.4.22092026.0158. Buy and Sell on the price chart become text-free triangles; pane markers unchanged. |
| 2.4.22092026.0203 | 22 Sep 2026 | Superseded. Converted Buy and Sell to triangles in both pane and chart; not carried forward. |
| 2.4.22092026.0158 | 22 Sep 2026 | Short recoloured orange and Sell red. Short signals off by default. |
| 2.3.22092026.0152 | 22 Sep 2026 | Buy, Sell, Short and Cover markers added with a position state machine, bar-close confirmation, chart markers and alerts. |
| 2.1.22092026.0143 | 22 Sep 2026 | First release. Pine v4 → v6 conversion, renamed REMA. |

Release 2.2 was not used.

---

## 10. Credits and licensing

The RSI / EMA / WMA base was converted to Pine Script v6 from an open-source Pine v4 script.

The **Pine v6 conversion**, the **trend conditions and position state machine**, the **Buy / Sell / Short / Cover markers**, the **price-chart markers**, the **transparent candles**, the **alert set** and the **REMA.STR** strategy build are the work of **Subhasom Mandal**.

---

## 11. Implementation notes

The Pine source carries short section comments; the details live here, in the order the code runs.

**Declaration.** `indicator()` replaces v4's `study()`, and `timeframe = ""` with `timeframe_gaps = true` replaces `resolution = ""`. This adds TradingView's Timeframe selector to the settings.

**Inputs.** Typed input functions (`input.int`, `input.source`, `input.bool`, `input.color`) replace v4's `input(..., type = ...)`. Grouped as the ungrouped Length/Source, then **Markers** and **Candles**.

**Calculations.** `ta.rsi`, `ta.ema` and `ta.wma` replace the un-namespaced v4 calls.

**Crosses.** `ta.crossunder` and `ta.crossover` are computed at global scope on every bar, not inside the `if` blocks. Pine's `ta.*` functions keep internal history and must run on every bar to return correct values.

**State machine.** `var int pos` holds the position across bars (1 = long, −1 = short, 0 = flat). The four signal flags are reset to `false` each bar. The block is gated by `not confirmBar or barstate.isconfirmed`; on historical bars `barstate.isconfirmed` is always true, and on the live bar Pine rolls `var` values back on each tick, so the state only commits at bar close.

**Plots.** v4's `transp = 50` on `fill()` was removed in v5, so the fill colour is built with `color.new(..., 50)`. The fill uses two `plot()` handles (RSI and the 50 line) rather than `hline()`, because `fill()` cannot mix the two types.

**Chart markers.** Drawn with `plotshape(..., force_overlay = true)`, which lets a pane indicator draw on the main chart. The toggle is applied to the series (`showOnChart and buySig`) because `force_overlay` only accepts a constant.

**Style-tab defaults.** The four Short/Cover plots carry `display = display.none`, which leaves them unchecked in the Style tab rather than removing them. Ticking them there shows them again; no input is involved.

**Candle colours.** `barcolor()` recolours the main chart's candles even though REMA lives in its own pane. Returning `na` when the toggle is off leaves the chart's own colours untouched.

**Alerts.** `alertcondition()` messages are compile-time constants; `{{ticker}}` and `{{close}}` are substituted by TradingView when the alert fires.

**Strategy build.** `strategy()` replaces `indicator()` and drops `timeframe`, which strategies do not accept. Orders are placed with `strategy.entry` and `strategy.close` after the state machine, exits before entries, with `process_orders_on_close = false` so they fill at the next open. Alerts use `alert(..., alert.freq_once_per_bar_close)` with the message built at runtime from `syminfo.ticker` and `str.tostring(close, format.mintick)`.
