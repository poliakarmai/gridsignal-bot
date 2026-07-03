# AGENTS.md — gridsignal-bot

## OpenWiki

This repository has documentation located in the /openwiki directory.

Start here:
- [OpenWiki quickstart](openwiki/quickstart.md)

OpenWiki includes repository overview, architecture notes, trading strategies, subscription flows, operations, and source maps.

When working in this repository, read the OpenWiki quickstart first, then follow its links to the relevant architecture, strategy, subscription, or operations notes.

---

> Навигация для AI-агентов. @Gridbolbot — Telegram-бот сигналов Bollinger Grid.

## Что это

Telegram-бот, который отдаёт торговые сигналы Bollinger Grid через RPC bybit-ws.  
Бесплатно: 3 скана/день. Pro: безлимит + алерты (Stars/TON).

## Где лежит

| Что | Путь |
|------|------|
| Код бота | `~/.local/bin/gridsignal-bot.py` |
| Pro-пользователи | `~/.local/share/gridsignal-bot/pro_users.db` |
| TON-инвойсы | `~/.local/share/gridsignal-bot/ton_invoices.db` |
| Пользователи | `~/.local/share/gridsignal-bot/users.db` |
| Systemd | `~/.config/systemd/user/gridsignal-bot.service` |

## Как запускать

```bash
# Сервис
systemctl --user [start|stop|restart|status] gridsignal-bot

# Логи
journalctl --user -u gridsignal-bot -f

# Вручную
GRIDSIGNAL_BOT_TOKEN=<токен> python3 ~/.local/bin/gridsignal-bot.py
```

## Зависимости

- `python-telegram-bot` (pip)
- `bybit-ws` RPC на `localhost:8766`
- `sqlite3` (системный)

## Команды бота

| Команда | Что делает |
|---------|-----------|
| `/start` | Приветствие + клавиатура |
| `/scan [short\|scalp\|mean\|funding\|rotation\|green\|TICKER]` | Сигналы Bollinger Grid |
| `/pro` / `/subscribe` | Pro-подписка (Stars + TON) |
| `/status` | Статус подписки |
| `/chart SYMBOL [TF]` | График Bollinger Bands |
| `/top` | Лидерборд по винрейту |
| `/stats` | Статистика и винрейт |
| `/rules` | Стратегия |
| `/alert SYMBOL [bbXX]` | Алерт на Lower BB |
| `/delalert SYMBOL` | Удалить алерт |
| `/setrisk DEPOSIT` | Установить депозит |
| `/history` | История сигналов |
| `/lang [ru\|en]` | Язык |
| `/dex SYMBOL` | DexScreener поиск |

## Платёжный флоу

### Telegram Stars
`/subscribe` → кнопка Stars → `send_invoice(XTR)` → `pre_checkout_query` → `successful_payment` → запись в `pro_users.db`

### TON (CryptoBot)
`/subscribe` → кнопка TON → `createInvoice(CryptoBot API)` → ссылка на оплату → `/check_ton_<id>` → `getInvoices(API)` → запись в `pro_users.db`

## Pro-подписка

- Stars: 300 Stars ≈ 400₽ / 30 дней
- TON: ~2 TON ≈ 400₽ / 30 дней
- Депровижнинг: каждые 30 мин (cron-джоб в боте)
- Админ-уведомление: каждый новый Pro → в чат 5529208670

## Инварианты

1. RPC обязателен — бот не работает без bybit-ws
2. Лимиты через `check_scan_allowed()` — 3/день бесплатно
3. Pro-статус через `is_pro()` — смотрит `pro_users.db`
4. Клавиатура через `MAIN_KEYBOARD` — кнопка «💳 Оплата» всегда видна
5. CRYPTOBOT_TOKEN из env — без него TON не работает

## Конвенции

- Python 3.11+
- Коммиты на русском
- Токен бота из env (`GRIDSIGNAL_BOT_TOKEN`)
- RPC токен авто-загружается из `bybit-ws/state.db`
