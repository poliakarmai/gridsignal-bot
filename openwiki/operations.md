# Operations

Deployment, configuration, logging, and maintenance notes for GridSignal Bot.

---

## Deployment

### Systemd User Service (`/AGENTS.md`)

The bot runs as a systemd user service:

```bash
# Start
systemctl --user start gridsignal-bot

# Stop
systemctl --user stop gridsignal-bot

# Restart
systemctl --user restart gridsignal-bot

# Status
systemctl --user status gridsignal-bot

# Logs
journalctl --user -u gridsignal-bot -f
```

Unit file: `~/.config/systemd/user/gridsignal-bot.service`

### Manual Run

```bash
GRIDSIGNAL_BOT_TOKEN=<token> python3 ~/.local/bin/gridsignal-bot.py
```

The bot binary is installed at `~/.local/bin/gridsignal-bot.py`.

---

## Dependencies

| Dependency | Version | Purpose |
|---|---|---|
| `python-telegram-bot` | any recent | Telegram Bot API framework (async) |
| `bybit-ws` | v6.0+ | WebSocket cache layer for Bybit data + RPC server |
| `sqlite3` | system | Local databases |
| `matplotlib` | any | Chart rendering (imported lazily in cmd_chart) |
| `mplfinance` | any | Candlestick chart library (imported lazily) |
| `pandas` | any | Kline data handling for charts (imported lazily) |

**Install dependencies**:

```bash
pip install python-telegram-bot matplotlib mplfinance pandas
```

Note: `bybit-ws` must be installed and running as a separate daemon. Bot does not install it.

---

## Infrastructure Dependencies

### `bybit-ws` RPC Daemon

The bot requires `bybit-ws` running on `localhost:8766` with:
- WebSocket connections to Bybit API v5 (maintained by bybit-ws)
- HTTP RPC endpoint at `localhost:8766/scan`
- RPC auth token stored in `~/.local/share/bybit-ws/state.db`

**Without bybit-ws, the bot starts but all scan and chart features fail.**

### `bybit` CLI

Some features call the `bybit` CLI directly via subprocess (see architecture doc). The CLI must be at `~/.local/bin/bybit`. Used for:
- Chart kline data
- Alert BB checking
- Inline query scoring
- Outcome tracking price checks

---

## Environment Configuration

| Variable | Required | Default | Notes |
|---|---|---|---|
| `GRIDSIGNAL_BOT_TOKEN` | Yes | — | From @BotFather |
| `CRYPTOBOT_TOKEN` | No | "" | TON payments disabled if missing |
| `GRIDSIGNAL_PRO_CHANNEL` | No | "" | Optional Pro channel ID |

The RPC token is auto-loaded from `bybit-ws/state.db` — not from environment.

---

## Data Files & Paths

All paths are relative to `$HOME`:

| File | Path | Grows With |
|---|---|---|
| Bot script | `~/.local/bin/gridsignal-bot.py` | Static |
| Scanner module | `~/.local/bin/gridsignal_scanner.py` | External dependency |
| bybit CLI | `~/.local/bin/bybit` | External dependency |
| Users DB | `~/.local/share/gridsignal-bot/users.db` | Users, signals, events |
| Pro DB | `~/.local/share/gridsignal-bot/pro_users.db` | Pro subscriptions |
| TON invoices DB | `~/.local/share/gridsignal-bot/ton_invoices.db` | TON payment tracking |
| RPC state DB | `~/.local/share/bybit-ws/state.db` | External (read-only) |
| Systemd unit | `~/.config/systemd/user/gridsignal-bot.service` | Static |

---

## Monitoring

### Key Log Patterns

```
[GridSignal] RPC token loaded        # Successful RPC auth
[GridSignal] RPC token not found     # bybit-ws not initialized
[GridSignal] RPC HTTP 401           # Auth token expired/invalid
[GridSignal] Scan error: ...         # bybit-ws unreachable or error
[TON] Invoice error: ...             # CryptoBot API failure
```

### Health Check

The bot does not expose a health endpoint. Basic checks:
1. `systemctl --user status gridsignal-bot` — is the service running?
2. `journalctl --user -u gridsignal-bot -n 20 --no-pager` — recent logs
3. Send any command to the bot on Telegram — does it respond?

### Metrics

The `/stats` command (admin-only for global stats, user-only for personal) provides:
- Total users, active users (7d/30d)
- Scans today/this week
- Subscriptions count, active alerts
- Overall winrate (binary + PnL-weighted)
- Median/average PnL
- Best/worst signals
- Top tickers by winrate

---

## Maintenance

### Database Cleanup

The `signals` table in `users.db` grows unboundedly. Consider:
- Adding a retention policy (e.g., delete signals older than 90 days)
- Running `VACUUM` periodically on `users.db` and `pro_users.db`

### Token Rotation

- Telegram bot token: rotate in `@BotFather` and update `GRIDSIGNAL_BOT_TOKEN`
- CryptoBot token: rotate via @CryptoBot merchant settings
- RPC token: auto-loaded from `bybit-ws/state.db` — restart bybit-ws to regenerate

### Updating the Bot

1. Replace the script at `~/.local/bin/gridsignal-bot.py`
2. Restart: `systemctl --user restart gridsignal-bot`

---

## Invariants (Critical)

These invariants from `AGENTS.md` must never be violated:

1. **RPC is mandatory** — bot is non-functional without the `bybit-ws` daemon
2. **Scan limits** — enforced by `check_scan_allowed()`; free users get 3/day
3. **Pro status** — verified via `is_pro()` against `pro_users.db`
4. **Payment button always visible** — `MAIN_KEYBOARD` always includes "💳 Оплата"
5. **CRYPTOBOT_TOKEN** — required for TON payments; without it, TON flow fails silently
6. **RPC token auto-load** — fetched from `bybit-ws/state.db` at startup and on cache miss

---

## Known Edge Cases

- **First-time user with no profile**: `get_user()` auto-creates a row in `users` table on first lookup
- **User blocks bot**: Wrapped in `try/except` in all notification contexts (digest, alerts, deprovision)
- **Daily limit reset race**: `update_scan_count()` uses atomic `WHERE scans_today < ?` but `check_scan_allowed()` is a separate read — user could squeeze 1 extra scan at the boundary
- **Inline query for non-USDT**: Bot auto-appends `USDT` if missing; silently ignores invalid symbols
- **TON payment without CryptoBot**: If `CRYPTOBOT_TOKEN` is empty, `create_ton_invoice` returns `(None, None)` — Stars is the fallback
