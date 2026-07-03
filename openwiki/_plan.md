# OpenWiki Plan — gridsignal-bot

## Repository Summary

Single-file Python Telegram bot (`gridsignal-bot.py`, ~2472 lines) that provides Bollinger Grid trading signals for Bybit futures. Supports 8 scanning strategies (LONG, SHORT, Scalp x10, Mean Revert x10, Funding x10, Rotation, Green-only, Ticker-specific). Monetized via Pro subscription (Telegram Stars + TON CryptoBot).

## Pages to Create

### 1. openwiki/quickstart.md (entrypoint)
- Overview, what the bot does, tech stack
- Table of contents linking to all other pages
- Quick start: setup, environment variables, run
- Source evidence: /README.md, /AGENTS.md, /gridsignal-bot.py:2408-2472

### 2. openwiki/architecture.md
- System architecture diagram (text)
- FSM / Message flow: user → Telegram → Bot → RPC/bybit-cli → Bybit
- Key components: Telegram handlers, RPC scanning, DB layer, Pro payment, background jobs
- Source evidence: /gridsignal-bot.py:10-43, 130-361, 2408-2472

### 3. openwiki/trading-strategies.md
- The 8 scan strategies explained
- Bollinger Grid mechanics (LONG vs SHORT)
- Scoring system (9 metrics, tiers, score zones)
- Position sizing and risk management
- Source evidence: /gridsignal-bot.py:554-682, 810-920, 1700-1808, 743-776

### 4. openwiki/subscriptions.md
- Pro subscription overview (free tier limits)
- Telegram Stars payment flow
- TON (CryptoBot) payment flow
- Deprovisioning, admin notifications
- Source evidence: /gridsignal-bot.py:42-111, 113-128, 1003-1173

### 5. openwiki/operations.md
- Deployment (systemd, environment)
- Dependencies (python-telegram-bot, bybit-ws RPC, sqlite3)
- Data files and paths
- Logging and monitoring
- Source evidence: /AGENTS.md, /gridsignal-bot.py:16-21, 2408-2472

