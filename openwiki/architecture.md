# Architecture

## Overview

GridSignal Bot is a single-process Python Telegram bot that acts as a signal layer between **Bybit perpetual futures data** and **Telegram users**. It does not execute trades — it produces trading signals that users can act on manually on Bybit.

```
User (Telegram)
    │
    ▼  Telegram Bot API (polling)
┌─────────────────────┐
│  gridsignal-bot.py  │
│                     │
│  ┌───────────────┐  │
│  │ Command       │  │
│  │ Handlers      │  │
│  │ (18 handlers) │  │
│  └──────┬────────┘  │
│         │           │
│  ┌──────▼────────┐  │
│  │ RPC Client     │──┼──► HTTP POST localhost:8766/scan
│  │ (urllib)       │  │        (bybit-ws RPC)
│  └──────┬────────┘  │
│         │           │
│  ┌──────▼────────┐  │
│  │ Subprocess    │──┼──► bybit cli (bb, raw)
│  │ Calls         │  │        (fallback for charts, alerts)
│  └──────┬────────┘  │
│         │           │
│  ┌──────▼────────┐  │
│  │ SQLite DB     │  │   users.db, pro_users.db, ton_invoices.db
│  │ Layer         │  │
│  └───────────────┘  │
│                     │
│  ┌───────────────┐  │
│  │ Background    │  │   daily digest, reset counts,
│  │ Job Queue     │  │   deprovision expired, check alerts,
│  └───────────────┘  │  check outcomes
└─────────────────────┘
```

---

## Component Breakdown

### 1. Telegram Command Handlers (`/gridsignal-bot.py:2420-2447`)

All 18 commands are registered in `main()`. Each handler:
- Opens a DB connection (`init_db()`)
- Calls `get_user()` to ensure the user exists
- Calls `log_event()` for analytics
- Performs the command logic
- Closes the DB connection

**Important**: DB connections are opened and closed per-request. This is a deliberate pattern to avoid threading issues with `python-telegram-bot`'s async nature. See `check_same_thread=False` and `PRAGMA journal_mode=WAL` in `init_db()`.

### 2. RPC Scan Layer (`/gridsignal-bot.py:307-356`)

The primary scan method `get_cached_scan()`:
- Builds a cache key from `interval` and `mode`
- Checks in-memory cache (`_cache` dict, 120s TTL)
- If cache miss: sends HTTP POST to `localhost:8766/scan` with `{"mode": ..., "interval": ..., "limit": 5}`
- Auth header: `Authorization: Bearer <token from bybit-ws/state.db>`
- Saves results to DB via `save_signals()`
- Applies filters (symbol, green-only)

**Why RPC and not direct API calls?** The `bybit-ws` daemon maintains persistent WebSocket connections to Bybit, caches ticker/kline data, and calculates Bollinger Bands server-side. This avoids raw API calls from the bot and keeps data fresh via WS push.

### 3. Subprocess Calls (`/gridsignal-bot.py:1303-1317, 1404-1419, 1473-1494, 2043-2046`)

Some features bypass RPC and call `bybit` CLI directly via `subprocess.run()`:
- **Charts** (`cmd_chart`) — fetches kline data via `bybit raw GET /v5/market/kline`
- **Alerts** (`check_alerts`) — fetches current BB data via `bybit bb SYMBOL D`
- **Inline query** — imports `gridsignal_scanner.score_coin` via subprocess
- **Outcome tracking** — batch price fetch via `bybit raw GET /v5/market/tickers`

**Important for changes**: If you modify the `bybit` CLI or change the `bybit-ws` RPC format, you must update both the RPC client AND any subprocess calls. They use different data paths.

### 4. Database Layer (`/gridsignal-bot.py:161-304`)

Four SQLite databases:

| Database | File | Purpose |
|---|---|---|
| `users.db` | `~/.local/share/gridsignal-bot/users.db` | Users, signals, events, alerts |
| `pro_users.db` | `~/.local/share/gridsignal-bot/pro_users.db` | Pro subscription records |
| `ton_invoices.db` | `~/.local/share/gridsignal-bot/ton_invoices.db` | TON payment tracking |
| `bybit-ws/state.db` | `~/.local/share/bybit-ws/state.db` | RPC auth token (read-only, external) |

**Schema notes** (`users.db`):
- `users` — user_id, username, deposit, scans_today, last_scan_ts, subscribed, joined_at, lang
- `signals` — ts, symbol, score, price, bb values, entry, tp1, tp2, sl, outcome, pnl_pct, timeframe, user_id, mode
- `events` — user_id, event type, timestamp (analytics)
- `alerts` — user_id, symbol, lower_bb, active, bb_zone

**Schema notes** (`pro_users.db`):
- `pro_users` — user_id, username, paid_at, expires_at, active

### 5. Background Jobs (`/gridsignal-bot.py:2449-2465`)

| Job | Interval | Purpose |
|---|---|---|
| `send_daily_digest` | Daily at 10:00 MSK | Morning top-3 signal summary |
| `reset_daily_counts` | Midnight UTC + every hour | Reset scan counters |
| `deprovision_expired` | Every 30 min | Deactivate expired Pro subscriptions |
| `check_alerts` | Every 2 min | Check price against alert thresholds |
| `check_outcomes` | Every hour | Evaluate if signals hit TP/SL/expired |

### 6. i18n (`/gridsignal-bot.py:361-527`)

Inline translation dictionary `T` with keys for Russian and English. The `t(key, lang, *args)` function looks up translations with fallback to Russian. The `lang` field is per-user in `users` table.

---

## Key Data Flow: A `/scan` Request

1. User sends `/scan` (or presses keyboard button)
2. `cmd_scan()` handler runs (`/gridsignal-bot.py:810`)
3. Parses arguments (interval, mode, symbol, filters)
4. Calls `check_scan_allowed()` — enforces cooldown (60s) and daily limit (3 scans free)
5. Sends "Scanning..." message
6. Calls `get_cached_scan()` — RPC request to `bybit-ws` or cache
7. If signals returned: formats them with `format_signal_full()` (respecting LONG/SHORT/x10 modes)
8. Adds position sizing if user has deposit set (`/setrisk`)
9. Edits message with formatted signals + share button
10. Calls `update_scan_count()` — atomically increments scan counter
11. Saves to DB via `save_signals()` (inside `get_cached_scan`)

---

## Invariants (from AGENTS.md)

1. **RPC is mandatory** — bot does not work without `bybit-ws` running on `localhost:8766`
2. **Limits via `check_scan_allowed()`** — 3/day free, 60s cooldown
3. **Pro status via `is_pro()`** — checks `expires_at > now()` in `pro_users.db`
4. **Keyboard always shows payment** — `💳 Оплата` button is always visible
5. **`CRYPTOBOT_TOKEN` from env** — without it, TON payment fails silently
6. **RPC token auto-loaded** from `bybit-ws/state.db` at startup and on cache miss

---

## Source Map

| Area | Lines | Key Functions |
|---|---|---|
| Imports & constants | 1–48 | `RPC_URL`, `DB_PATH`, `PRO_PRICE_STARS` |
| TON / Pro payment | 50–128 | `create_ton_invoice`, `check_ton_payment`, `init_pro_db`, `is_pro` |
| Cache | 130–133 | `_cache`, `CACHE_TTL` |
| DB init & helpers | 161–304 | `init_db`, `get_user`, `check_scan_allowed`, `save_signals` |
| RPC scanning | 307–356 | `get_cached_scan` |
| i18n | 361–527 | `T` dict, `t()` function |
| Signal formatting | 530–682 | `format_signal_long_full`, `format_signal_x10_full`, `format_signal_short_full` |
| Command handlers | 688–1241 | `cmd_start`, `cmd_scan`, `cmd_subscribe`, `cmd_chart`, etc. |
| Alert system | 1246–1448 | `cmd_alert`, `check_alerts` |
| Inline query | 1453–1531 | `inline_query` |
| Fear & Horoscope | 1548–1825 | `cmd_fear`, `cmd_horoscope` |
| Stats & leaderboard | 1828–2007 | `cmd_stats`, `cmd_leaderboard` |
| Charts | 2012–2129 | `cmd_chart` (mplfinance) |
| DEX search | 2289–2401 | `cmd_dex`, `_do_dex_search` |
| Button handlers | 2223–2282 | `handle_buttons` |
| Background jobs | 2449–2465 | Job queue registration |
| Main | 2408–2472 | `main()`, app builder |

---

## Things to Watch Out For

- **Thread safety**: `init_db()` uses `check_same_thread=False` and WAL mode. Background jobs (check_alerts, check_outcomes) open their own connections. Be careful adding shared state.
- **Subprocess latency**: Some commands spawn subprocesses (`bybit bb`, `bybit raw`). These have 10–25s timeouts. Adding more subprocess calls could impact responsiveness.
- **`bybit-ws` dependency**: The RPC endpoint format is defined by `bybit-ws`. If `bybit-ws` changes its RPC protocol, the scan and chart functionality breaks.
- **Inline query uses subprocess import**: `inline_query` spawns a Python subprocess that imports `gridsignal_scanner`. This is fragile — if the scanner module path changes, inline queries break silently.
- **Rate limits**: Free tier is 3 scans/day. Pro removes the limit. The `update_scan_count()` function uses an atomic `WHERE scans_today < ?` guard but there's still a TOCTOU window between `check_scan_allowed()` and `update_scan_count()`.
