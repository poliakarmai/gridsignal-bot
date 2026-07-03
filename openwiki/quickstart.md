# GridSignal Bot — OpenWiki

> **@Gridbolbot** — Telegram bot that produces Bollinger Grid trading signals for Bybit USDT perpetual futures, with 8 scanning strategies, multi-timeframe charts, price alerts, and Pro subscription via Telegram Stars and TON.

---

## Quick Start

### Prerequisites

- Python 3.11+
- `pip install python-telegram-bot` (the `python-telegram-bot` package)
- `bybit-ws` v6.0+ running on `localhost:8766` (provides a WebSocket cache layer over Bybit API v5; bot talks to it via HTTP RPC)
- `sqlite3` (system)
- A Telegram bot token from [@BotFather](https://t.me/BotFather)
- (Optional) CryptoBot API token for TON payments

### Run the Bot

```bash
export GRIDSIGNAL_BOT_TOKEN=<your-telegram-bot-token>
python3 gridsignal-bot.py
```

Or via systemd user service:

```bash
systemctl --user [start|stop|restart|status] gridsignal-bot
journalctl --user -u gridsignal-bot -f
```

### Environment Variables

| Variable | Required | Purpose |
|---|---|---|
| `GRIDSIGNAL_BOT_TOKEN` | Yes | Telegram Bot API token |
| `CRYPTOBOT_TOKEN` | No (no TON without it) | CryptoBot API token for TON payments |
| `GRIDSIGNAL_PRO_CHANNEL` | No | Optional private channel for Pro users |

### Data Files

| Path | Purpose |
|---|---|
| `~/.local/share/gridsignal-bot/users.db` | User profiles, scan counts, deposits, signals, events, alerts |
| `~/.local/share/gridsignal-bot/pro_users.db` | Pro subscription records |
| `~/.local/share/gridsignal-bot/ton_invoices.db` | TON payment invoice tracking |
| `~/.local/share/bybit-ws/state.db` | bybit-ws RPC auth token (loaded automatically) |
| `~/.config/systemd/user/gridsignal-bot.service` | systemd unit file |

---

## Repository Structure

This is a single-file Python project.

```
gridsignal-bot/
├── gridsignal-bot.py   # The entire Telegram bot (~2472 lines)
├── README.md           # Project description (Russian)
├── AGENTS.md           # AI agent navigation guide
├── openwiki/           # This documentation
│   ├── quickstart.md   # ← You are here
│   ├── architecture.md
│   ├── trading-strategies.md
│   ├── subscriptions.md
│   └── operations.md
```

---

## What This Bot Does

1. **Market scanning** — queries Bybit data via `bybit-ws` RPC to calculate Bollinger Bands (20,2) for 200+ USDT perpetuals
2. **8 scan strategies** — LONG, SHORT, Scalp x10, Mean Revert x10, Funding Momentum x10, Funding Rotation, Green-only, and per-ticker scans
3. **Signal scoring** — 9-metric scoring system (Tier, BB position, volume, funding rate, RSI(14), volatility, etc.)
4. **Multi-timeframe charts** — generates candlestick + Bollinger Band charts via `mplfinance`
5. **Price alerts** — notify when price touches Lower BB or enters a configurable BB zone
6. **Signal history & outcome tracking** — records TP1/TP2/SL/EXPIRED outcomes, calculates winrate, PnL, leaderboards
7. **Pro subscriptions** — Telegram Stars (XTR) and TON (via CryptoBot) payment with auto-deprovisioning
8. **Daily digest** — morning summary for subscribed users
9. **DexScreener integration** — search token data across DEXes
10. **Fear & Greed index** — based on average BB position of top-10 coins

---

## Key Concepts (at a glance)

- **Bollinger Grid** — The core strategy: place limit orders at specific deviations from Bollinger Bands (lower for LONG, upper proximity for SHORT)
- **RPC layer** — Bot does **not** call Bybit API directly. It calls a local `bybit-ws` HTTP RPC at `localhost:8766/scan` which maintains a WebSocket cache
- **Free tier** — 3 scans per day per user. Pro removes the limit
- **Scoring** — 0–10 scale; ≥7 = excellent, 5.5–6.9 = good, 3.5–5.4 = caution

---

## Bot Commands

| Command | Description |
|---|---|
| `/start` | Welcome + keyboard |
| `/scan [short|scalp|mean|funding|rotation|green|TICKER]` | Bollinger Grid signals |
| `/pro` / `/subscribe` | Pro subscription (Stars + TON) |
| `/status` | Subscription status |
| `/chart SYMBOL [TF]` | Bollinger Bands chart (TF: D/W/M/5/3) |
| `/top` | 24h gainers/losers leaderboard |
| `/stats` | Personal or admin statistics |
| `/rules` | Strategy rules |
| `/alert SYMBOL [bbXX]` | Alert on Lower BB or BB zone |
| `/delalert SYMBOL` | Remove alert |
| `/setrisk DEPOSIT` | Set deposit for position sizing |
| `/history` | Scan history |
| `/lang [ru|en]` | Language |
| `/dex SYMBOL` | DexScreener search |
| `/fear` | Fear & Greed index |
| `/horoscope` | Crypto horoscope |
| `/contact` | Contact author |

---

## Next Steps

- **[Architecture](architecture.md)** — system components, message flow, data layer
- **[Trading Strategies](trading-strategies.md)** — how each scan mode works, scoring, risk management
- **[Subscriptions](subscriptions.md)** — Pro tier, payment flows, deprovisioning
- **[Operations](operations.md)** — deployment, dependencies, monitoring, invariants
