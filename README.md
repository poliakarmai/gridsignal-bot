# @Gridbolbot — Bollinger Grid Signals

[![Python](https://img.shields.io/badge/python-3.11%2B-blue)](https://python.org)
[![Version](https://img.shields.io/badge/version-5.0-green)](./AGENTS.md)
[![Bybit](https://img.shields.io/badge/Bybit-API%20v5-orange)](https://bybit.com)

**Telegram-бот для торговых сигналов по стратегии Bollinger Grid.** Бесплатно: 3 скана/день. Pro: безлимит + алерты + приоритет.

**Бот:** [@Gridbolbot](https://t.me/Gridbolbot)

## Возможности

| Режим | Описание |
|-------|---------|
| ⚡ **Scalping x10** | Быстрые входы по BB-полосам на 5-минутном таймфрейме |
| 🔄 **Mean Revert** | Возврат к среднему — вход против движения |
| 💰 **Funding** | Мониторинг ставок финансирования по 200+ тикерам |
| 🔍 **Short Scan** | Быстрый скан одного тикера |
| 🟢 **Green Zone** | Только LONG-сигналы в зелёной зоне |
| 🔄 **Rotation** | Ротация по секторам |

## Команды

| Команда | Назначение |
|---------|-----------|
| `/start` | Приветствие + главное меню |
| `/scan [режим]` | Bollinger Grid сигналы (short/scalp/mean/funding/rotation/green/TICKER) |
| `/pro` | Pro-подписка (Stars + TON) |
| `/status` | Статус подписки |
| `/chart SYMBOL [TF]` | График Bollinger Bands |
| `/top` | Лидерборд по винрейту |
| `/stats` | Статистика сигналов |
| `/rules` | Правила стратегии |
| `/alert SYMBOL [bbXX]` | Алерт на Lower BB |
| `/delalert SYMBOL` | Удалить алерт |
| `/setrisk DEPOSIT` | Установить торговый депозит |
| `/history` | История сигналов |
| `/lang [ru/en]` | Язык интерфейса |
| `/dex SYMBOL` | DexScreener поиск |

## Pro-подписка

| Метод | Цена | Срок |
|-------|:----:|:----:|
| ⭐ Telegram Stars | 300 ⭐ (~400₽) | 30 дней |
| 💎 TON (CryptoBot) | ~2 TON (~400₽) | 30 дней |

**Платёжный флоу:**
1. `/pro` → выбор Stars или TON
2. Stars: `sendInvoice(XTR)` → оплата в Telegram → мгновенная активация
3. TON: `createInvoice(CryptoBot)` → оплата → `/check_ton_<id>` → активация
4. Pro-статус в `pro_users.db`, проверка каждые 30 мин

## Технологии

- **Язык:** Python 3.11+
- **Фреймворк:** `python-telegram-bot`
- **Данные:** Bybit API v5 (через RPC `bybit-ws:8766`)
- **Хранение:** SQLite (пользователи, подписки, инвойсы)
- **Развёртывание:** systemd `gridsignal-bot.service`

## Зависимости

- `bybit-ws` RPC на `localhost:8766` (обязательно)
- `python-telegram-bot` (pip)
- CryptoBot API токен (для TON-платежей)

## Установка

```bash
git clone https://github.com/poliakarmai/gridsignal-bot.git
pip install python-telegram-bot

# .env
GRIDSIGNAL_BOT_TOKEN=<ваш_токен>
CRYPTOBOT_TOKEN=<токен_cryptobot>

# Запуск
systemctl --user enable --now gridsignal-bot
```

## Ссылки

- [AGENTS.md](AGENTS.md) — документация для AI-агентов
- [OpenWiki](openwiki/quickstart.md) — детальная архитектура и стратегии
- [bybit-ws](https://github.com/poliakarmai/bybit-ws) — бэкенд (RPC-сервер)

## Лицензия

MIT
