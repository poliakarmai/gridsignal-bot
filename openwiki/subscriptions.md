# Pro Subscriptions

GridSignal Bot monetizes through a **Pro subscription** that removes the free tier limit (3 scans/day). Two payment methods are supported: Telegram Stars and TON via CryptoBot.

---

## Free vs Pro

| Feature | Free | Pro |
|---|---|---|
| Scans per day | 3 | Unlimited |
| Cooldown | 60s | 60s |
| Alerts | Up to 5 | Unlimited |
| Daily digest | Yes | Yes |
| Price | Free | ~300 Stars or ~2 TON / 30 days |

---

## Payment Flows

### Telegram Stars (`/gridsignal-bot.js:1099-1110`)

Telegram's native payment system. No provider token needed since XTR is a Telegram-native currency.

1. User sends `/subscribe` or `/pro` or taps "💳 Оплата"
2. Bot checks `is_pro()` — if already Pro, shows status; otherwise shows payment method selection
3. User taps "Stars (300)"
4. Bot calls `send_invoice()` with `currency="XTR"` and `prices=[{label: "Pro (30 days)", amount: 300}]`
5. Telegram shows native payment sheet
6. `pre_checkout_query` handler auto-approves (`pre_checkout_handler`, line 1051)
7. `successful_payment_handler` (line 1056) activates Pro for 30 days
8. Admin notification sent to chat `5529208670`

### TON (CryptoBot) (`/gridsignal-bot.js:58-111, 1111-1128`)

Uses [@CryptoBot](https://t.me/CryptoBot) merchant API:

1. User selects "TON"
2. `create_ton_invoice()` POSTs to `https://pay.crypt.bot/api/createInvoice` with `asset=TON`, `amount=2`
3. Bot sends invoice URL + "✅ I paid" button
4. User pays in CryptoBot, returns and taps "✅ I paid"
5. `check_ton_payment()` POSTs to `https://pay.crypt.bot/api/getInvoices` to verify
6. If `status == "paid"`, Pro is activated for 30 days

**Important**: TON flow requires `CRYPTOBOT_TOKEN` environment variable. Without it, TON payment silently returns `(None, None)`.

---

## Database Schema (`/gridsignal-bot.js:113-128`)

**`pro_users` table** (in `pro_users.db`):

```sql
CREATE TABLE pro_users (
    user_id INTEGER PRIMARY KEY,
    username TEXT,
    paid_at TEXT,
    expires_at TEXT,
    active INTEGER DEFAULT 1
);
```

**`ton_invoices` table** (in `ton_invoices.db`):

```sql
CREATE TABLE ton_invoices (
    invoice_id INTEGER PRIMARY KEY,
    user_id INTEGER,
    status TEXT DEFAULT 'pending',
    created_at TEXT
);
```

---

## Deprovisioning (`/gridsignal-bot.js:1156-1174`)

Every 30 minutes, `deprovision_expired()`:
1. Queries `WHERE active=1 AND expires_at < datetime('now')`
2. Sets `active=0` for all expired records
3. Sends each user a notification: "⚠️ GridSignal Pro истёк. Продлить: /subscribe"
4. Returns count of deprovisioned users (logged but unused)

---

## Checking Pro Status (`/gridsignal-bot.js:121-128`)

```python
def is_pro(user_id: int) -> bool:
    conn = init_pro_db()
    row = conn.execute(
        "SELECT 1 FROM pro_users WHERE user_id=? AND active=1 AND expires_at > datetime('now')",
        (user_id,)
    ).fetchone()
    conn.close()
    return row is not None
```

This is called:
- In `cmd_subscribe` — to show Pro status or payment options
- In `check_scan_allowed` — indirectly via the limit (Pro users bypass `MAX_SCANS_PER_DAY` only, but the cooldown still applies)
- In `cmd_status` — to display expiration date

---

## Important Notes

- **Pro does NOT remove the 60s cooldown** — `check_scan_allowed()` applies cooldown regardless of Pro status
- **Admin notification** is fire-and-forget (wrapped in try/except, silently fails if bot can't message admin)
- **Admin chat ID** is hardcoded as `5529208670` — if you change the admin, update this value
- **Price constants**: `PRO_PRICE_STARS = 300`, `TON_PRICE = 2.0` — update these to change pricing
- **Subscription duration**: 30 days hardcoded via `timedelta(days=30)` in both payment handlers
- **Deprovisioning lag**: Up to 30 minutes (job runs every 30 min) before expired users lose access
