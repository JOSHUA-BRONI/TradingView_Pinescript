# Session Extreme Revisit Alert System — Pro (SERAS·Pro)

A professional institutional-grade Pine Script indicator for TradingView that identifies London and New York session high/low candles, marks them as reaction zones, and fires alerts when price later revisits those zones — filtered by HTF trend bias, kill zone timing, and optional candle confirmation.

---

## What This System Does

At the close of each London or New York trading session, the script identifies:
- The **candle with the highest wick** during that session → **Supply Zone**
- The **candle with the lowest wick** during that session → **Demand Zone**

Each identified candle's full range (wick high to wick low) becomes a **reaction zone** that extends into the future. When price later revisits that range from a different session, an alert fires.

The core idea is institutional: the candle that forms a session extreme is where aggressive order flow, trapped participants, or unfinished institutional business exists. Price returning to that range often produces a measurable reaction.

---

## Improvements Over Basic Version

| Feature | Basic | Pro |
|---|---|---|
| Zone memory | 1 zone per side (overwritten daily) | Last 3 zones per side (configurable, up to 5) |
| Alert quality filter | None | HTF trend bias (EMA filter) |
| Entry timing | Any time of day | Kill zones only (optional) |
| Confirmation | First touch only | First touch OR pin bar confirmation |
| Displacement check | None | Session must close N× ATR from extreme |
| Zone lifetime | 500 bars (~1 day on 5m) | 10,000 bars (~35 days on 5m) |
| Alert messages | Static | Dynamic (live zone levels, session, HTF bias, kill zone status) |
| Signal quality tiers | 1 tier | 2 tiers (entry touch + confirmed rejection) |

---

## How Zone Detection Works

### During the Session
Every bar within the London or New York session window is checked:
- If the current high exceeds the running session high → update the stored **high candle** (H, L, O, C, bar index)
- If the current low is below the running session low → update the stored **low candle**

### At Session Close
The script evaluates the stored extreme candle against institutional filters:

1. **Size filter** — candle range ≥ `Min Candle Size × ATR-14`
2. **Wick ratio filter** — `(upper wick + lower wick) / total range ≥ Min Wick Ratio`
3. **Displacement filter** — session closing price is at least `Displacement Threshold × ATR` away from the extreme (confirms price moved off the zone)

If all enabled filters pass, a **colored zone box** is drawn extending to the right from the extreme candle.

### Zone Memory (Array-Based)
Each zone type (London High, London Low, NY High, NY Low) maintains an array of the last N zones (default: 3). When a new session creates a new zone:
- It is added to the front of the array
- If the array exceeds the cap, the oldest zone is evicted (its box is deleted)

This means Monday's London High zone is **not** lost when Tuesday's London session closes — it stays active until it expires or is evicted by a newer zone.

---

## How Alert Logic Works

### Filters Applied Before Any Alert Fires

| Filter | Description |
|---|---|
| **HTF Bias** | Supply zone alerts only when the HTF EMA confirms bearish trend. Demand zone alerts only when HTF EMA confirms bullish trend. Prevents shorting into an uptrend or longing into a downtrend. |
| **Kill Zone** | Alerts only fire during the configured high-probability time windows (London Open, NY Open, London Close). |
| **Same-session exclusion** | A zone never alerts during the session that created it — only during later sessions. |

### Signal Modes

**First Touch**
- Alert fires on the **first bar** that price enters the zone (wick or body overlap)
- Does not fire repeatedly while price stays inside the zone
- Best for early entries and scalping

**Rejection Candle**
- Alert fires when a **pin bar** closes inside the zone:
  - At a supply zone: bearish pin bar (close < open, upper wick ≥ body × ratio)
  - At a demand zone: bullish pin bar (close > open, lower wick ≥ body × ratio)
- Higher quality — confirms rejection, not just touch
- Slightly delayed vs first touch

### Visual Signals on Chart

| Marker | Meaning |
|---|---|
| Small red triangle (above bar) | Supply zone first touch |
| Small green triangle (below bar) | Demand zone first touch |
| Large dark red triangle (above bar) | Supply zone — confirmed bearish pin bar |
| Large dark green triangle (below bar) | Demand zone — confirmed bullish pin bar |

---

## Zone Expiry

Zones are retired when either condition is met:

1. **Full Mitigation** (if enabled): A candle closes beyond the zone boundary — supply zone retires when `close > zone top`, demand zone retires when `close < zone bottom`. This means institutional orders at that level were absorbed.

2. **Maximum Age**: Zone has been active for more than `Max Active Bars`. Default is **10,000 bars**, which equals approximately:
   - ~35 days on a 5-minute chart
   - ~7 days on a 15-minute chart
   - ~6 weeks on a 1-hour chart

Expired zones turn grey and stop extending — they remain visible for historical reference.

---

## How to Set Up Alerts in TradingView

### Option A — Dynamic Alerts (Recommended)
Includes live zone levels, session name, HTF bias, and kill zone status in the message.

1. Open the Alerts panel → **Create Alert**
2. Condition: select this indicator
3. Select **"Any alert() function call"**
4. Set notification method (app, email, webhook, etc.)
5. Save

### Option B — Specific Static Alerts
Six pre-defined conditions available from the indicator's alert condition list:
- `Supply Zone Revisit`
- `Demand Zone Revisit`
- `Supply Zone Confirmed (Pin Bar)`
- `Demand Zone Confirmed (Pin Bar)`
- `Any Zone Revisit`
- `Any Confirmed Signal`

---

## Recommended Settings by Use Case

### Swing Trading (Daily bias, HTF entries)
| Setting | Value |
|---|---|
| HTF Timeframe | 1D or 4H |
| HTF EMA Length | 50 |
| Signal Mode | Rejection Candle |
| Kill Zones | Enabled |
| Max Zones per Side | 3 |
| Chart Timeframe | 15m or 30m |

### Scalping / Intraday
| Setting | Value |
|---|---|
| HTF Timeframe | 1H |
| HTF EMA Length | 20 |
| Signal Mode | First Touch |
| Kill Zones | Enabled |
| Max Zones per Side | 2 |
| Chart Timeframe | 1m or 5m |

### Zone Research / Backtesting
| Setting | Value |
|---|---|
| Enable Filters | Off |
| Enable HTF Bias | Off |
| Kill Zones | Off |
| Signal Mode | First Touch |
| Max Zones per Side | 5 |

---

## Recommended Chart Timeframe

### Primary Recommendation — 5-minute

The ICT/institutional standard for session-based analysis. This is where the system performs best.

- London (3h) = **36 candles** — enough to cleanly identify the true extreme candle
- New York (4h) = **48 candles** — same quality
- Kill zone windows (2h) = **24 candles** — enough bars to detect pin bars and time entries
- ATR and wick ratio filters produce meaningful, well-calibrated results

### 15-minute — Best Balance (Semi-Active)

- London = 12 candles, NY = 16 candles
- Cleaner candle structure, less noise than 5m
- Good for traders who don't want to watch every tick
- Kill zone windows give ~8 candles — sufficient for confirmation

### 1-hour — Swing Trading Only

- London = only **3 candles**, NY = **4 candles**
- Zone detection still works but precision drops — the extreme candle spans a wide range, producing a thick zone
- Best suited for multi-day swing positions where exact entry timing is less critical

### 1-minute — Not Recommended

- Too many candles per session, too much noise
- Institutional filters become difficult to calibrate correctly
- Many false zone touches per session before a real reaction develops

### The Two-Timeframe Approach

This system separates two roles:

| Role | Timeframe | Purpose |
|---|---|---|
| Detection chart | 5m or 15m | Where zones are identified and alerts fire |
| HTF EMA bias | 4H (default in settings) | Determines trend direction filter |

Run the indicator on a **5m chart** while the bias filter reads the **4H trend**. This gives intraday precision with higher-timeframe context built in.

### Quick Reference

| Trading Style | Chart Timeframe |
|---|---|
| Active intraday / ICT-style | 5m |
| Semi-active, 1–3 trades per day | 15m |
| Swing trading, overnight holds | 1H |

Start with 5m. If alerts feel too frequent, switch to 15m or tighten the filters (raise ATR multiplier, switch to Rejection Candle mode).

---

## Session & Kill Zone Defaults (UTC)

| Zone | Time (UTC) | Purpose |
|---|---|---|
| London Session | 07:00 – 10:00 | Zone detection window |
| New York Session | 12:00 – 16:00 | Zone detection window |
| London Open Kill Zone | 07:00 – 09:00 | Highest probability alert window |
| NY Open Kill Zone | 12:00 – 14:00 | Highest probability alert window |
| London Close Kill Zone | 15:00 – 16:00 | Session overlap window |

All session times are fully configurable. Adjust for your broker's timezone if needed.

---

## Limitations to Be Aware Of

1. **Intraday timeframes only** — Session detection requires sub-daily bars. Works on 1m to 4H. Not compatible with Daily or Weekly charts.

2. **Alerts fire on real-time bars only** — Historical bars on the chart show markers but TradingView does not send historical alerts. Only live market bars trigger notifications.

3. **One signal per zone type per bar** — If multiple zones of the same type are touched simultaneously, the alert message reflects the newest zone. All signal conditions still trigger correctly.

4. **HTF EMA is a simple trend proxy** — Price above/below a 50 EMA is not a complete trend definition. For more nuanced trend analysis, combine with manual higher-timeframe structure analysis.

5. **Rejection candle filters are simplified** — The pin bar detection uses body vs wick ratio. It does not account for context (e.g., a pin at the edge of a zone vs deep inside it).

6. **Volume data not used** — Real institutional detection would incorporate volume profile or order flow. This script uses wick ratio and ATR as proxies since volume data quality varies by broker.

---

## Institutional Interpretation Guide

| Signal | Interpretation |
|---|---|
| Price touches **Supply Zone** during kill zone, HTF bearish | High-probability institutional supply reaction. Look for distribution, short setups, or continuation of downtrend. |
| Price touches **Demand Zone** during kill zone, HTF bullish | High-probability institutional demand reaction. Look for accumulation, long setups, or continuation of uptrend. |
| **Confirmed rejection** (pin bar) at Supply Zone | Institutional participants actively rejected price. Higher confidence in a reversal or continuation short. |
| **Confirmed rejection** (pin bar) at Demand Zone | Institutional participants actively supported price. Higher confidence in a reversal or continuation long. |
| Zone turns **grey** (expired) | Zone was fully mitigated (closed through) or too old. No longer considered an active institutional reference. |

---

## File Structure

```
Session_Test_PineScript/
├── session_extreme_revisit.pine   ← Main indicator script
└── README.md                      ← This file
```

---

## Changelog

| Version | Changes |
|---|---|
| v1.0 | Basic session high/low zone detection, single zone per side, first-touch alerts |
| v2.0 (Pro) | Multi-zone array memory, HTF bias filter, kill zone filter, rejection candle confirmation, displacement filter, dynamic alert messages, very long zone lifetime defaults |
