# Trading Strategies

The bot implements **Bollinger Grid** — a mean-reversion strategy that places limit orders at statistically significant deviations from the 20-period simple moving average (middle band).

---

## Core Strategy: Bollinger Grid

Bollinger Bands (20, 2):
- **Middle Band** = 20-period SMA of closing price
- **Upper Band** = Middle Band + 2 × standard deviation
- **Lower Band** = Middle Band − 2 × standard deviation

**The grid principle**: The bot places limit orders at fixed offsets from bands rather than at the bands themselves, creating a "grid" of entry points.

### LONG Strategy (`/gridsignal-bot.py:564-598`)

- **Entry**: Limit order at `Lower BB × 0.97` (3% below the Lower Band)
- **TP1**: Middle Band (regression to mean)
- **TP2**: Upper Band (full grid exhaustion)
- **SL**: `Lower BB × 0.93` (7% below Lower Band)
- **Leverage**: 3x

### SHORT Strategy (`/gridsignal-bot.py:638-674`)

- **Entry**: Limit order at `price × 1.02` (2% above market, waiting for a bounce rejection)
- **TP1**: Middle Band
- **TP2**: Lower Band
- **SL**: `entry × (1 + sl_pct)` where `sl_pct = 5%` for Tier A/B, `7%` for Tier C/D
- **Leverage**: 3x

---

## The 8 Scan Modes

### 1. `/scan` (default LONG) (`/gridsignal-bot.py:810-920`)
Top-5 LONG signals with highest score across 200+ USDT perpetuals. Timeframe: Daily.

### 2. `/scan short` (SHORT) (`/gridsignal-bot.py:835-837`)
Top-5 SHORT signals. Mirrors LONG logic but inverts entries, targets, and SL.

### 3. `/scan scalp` (Scalp x10) (`/gridsignal-bot.py:837-838`)
Fast scalping signals on short timeframes. Uses 10x leverage. Shorter TP targets, tighter SL. Entry at more aggressive price levels.

### 4. `/scan mean` (Mean Revert x10) (`/gridsignal-bot.py:839-840`)
Mean reversion on 10x leverage. Targets coins that have deviated significantly from their mean and are likely to snap back. Uses RSI extremes as additional filter.

### 5. `/scan funding` (Funding Momentum x10) (`/gridsignal-bot.py:841-842`)
Trades based on funding rate divergence. High negative funding = long pressure; high positive = short pressure. Combines BB position with funding rate direction. 10x leverage.

### 6. `/scan rotation` (Funding Rotation) (`/gridsignal-bot.py:843-844, 873-894`)
Detects existing positions where current funding is unfavorable and suggests rotating to a correlated coin with better funding. Requires open positions data from `api.fetch_positions()`. This is the only mode that **reads user positions** and suggests switches rather than new entries.

### 7. `/scan green` (Green/ Buy Zone) (`/gridsignal-bot.py:833-834`)
Filters signals to show only coins where BB position < 25% (deep in the buy zone). Same logic as LONG but with stricter entry filter.

### 8. `/scan TICKER` (Per-ticker) (`/gridsignal-bot.py:847-852`)
Scans a specific symbol. Falls through to `get_cached_scan()` with `symbol=` parameter.

---

## Scoring System (`/gridsignal-bot.py:710-775`)

Score is calculated server-side by `bybit-ws` (not in the bot code). The bot displays the score and applies it for formatting. Known scoring factors (from `/help`):

| Metric | Weight Impact |
|---|---|
| **Tier** (S/A/B/C/D) | Fundamental tier via CoinGecko/volume |
| **BB Daily position** | How close to Lower Band (0–100%) |
| **24h Volume** | Liquidity filter |
| **Days falling/rising** | Trend strength for LONG/SHORT |
| **Weekly + Monthly BB** | Multi-timeframe confluence |
| **Funding rate** | Perpetuals funding direction |
| **Volatility** | ATR-based band width assessment |
| **Bounce quality** | Historical rejection pattern at lower band |
| **RSI(14)** | Relative Strength Index oversold/overbought |

### Score Zones

| Score | Label | Quality |
|---|---|---|
| ≥7.0 | 🔥 | Excellent entry |
| 5.5–6.9 | ✅ | Good entry |
| 3.5–5.4 | ⚠️ | Caution |

---

## Risk Management (`/gridsignal-bot.py:781-807`)

The `/rules` command outlines position management rules (not enforced by bot — advisory):

- **Position lifecycle**: Daily limit orders are GTC; M5 Grid max 2 hours; M3 Turbo max 30 minutes
- **When to cancel**: Price moves above Lower BB >1 hour, BB position >40%, or time expiry
- **Stop-loss discipline**: No averaging down; 3 consecutive losses = stop for the day; 3% daily drawdown = close everything
- **Entry conditions**: Don't enter if Daily BB >80%; Weekly >75% → tighten SL; >90% → take profit
- **Max risk**: 5 concurrent positions, max $40 margin per coin
- **Leverage**: 3x (Daily), 10x (M3/scalp/mean_revert/funding)

### Position Sizing (`/gridsignal-bot.py:587-598, 628-635`)

When user sets deposit via `/setrisk DEPOSIT`:
- **LONG/SHORT (3x)**: `margin = deposit × 2%`, `qty = (margin × 3) / entry`
- **x10 modes**: `margin = deposit × 1%`, `qty = (margin × 10) / entry`

---

## Outcome Tracking (`/gridsignal-bot.py:1700-1808`)

Every hour, `check_outcomes()` runs in the background:
1. Fetches all signals without an outcome
2. Batch-fetches current prices via `bybit raw GET /v5/market/tickers`
3. For each signal, checks if price has reached TP1/TP2/SL or expired
4. SHORT outcomes: price ≤ TP2 → TP2, price ≤ TP1 → TP1, price ≥ SL → SL
5. LONG outcomes: price ≥ TP2 → TP2, price ≥ TP1 → TP1, price ≤ SL → SL
6. Expiry per timeframe: D=7d, W=14d, M=30d, M5=2d, M3=1d
7. Sends "🎯 TP hit!" notifications only for TP outcomes (SL is silent)
8. Records PnL percentage

**Important for changes**: The outcome tracking has a short-only and long-only branch. If you add a new mode that doesn't fit LONG/SHORT semantics, you need to extend the outcome logic.

---

## Chart Generation (`/gridsignal-bot.py:2012-2129`)

`/chart SYMBOL [TF]`:
1. Fetches 100 kline candles via `bybit raw GET /v5/market/kline`
2. Calculates Bollinger Bands (20, 2) with pandas rolling window
3. Generates a candlestick chart with BB overlay via `mplfinance`
4. Uploads to Telegram as a photo with caption containing key levels
5. Temp file cleaned up in `finally` block

**Dependencies**: `matplotlib`, `mplfinance`, `pandas` (imported lazily inside the handler)

---

## Source Map: Strategy Logic

| Function | Lines | Mode |
|---|---|---|
| `format_signal_long_full` | 564–598 | LONG |
| `format_signal_short_full` | 638–674 | SHORT |
| `format_signal_x10_full` | 601–635 | Scalp/Mean/Funding x10 |
| `cmd_scan` (mode parsing) | 810–920 | All modes |
| `check_funding_rotation` (imported) | 874 | Rotation |
| `check_outcomes` | 1700–1808 | TP/SL tracking |
| `format_signal_short` | 543–551 | Digest format |
| `cmd_rules` | 781–807 | Risk rules |
