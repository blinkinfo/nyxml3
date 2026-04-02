# AutoPoly - BTC 5-Min Pattern Trading Bot

<div align="center">

Automated Polymarket trading bot for BTC 5-minute Up/Down binary options, powered by **6-candle historical pattern matching** and controlled entirely through **Telegram**.

[Features](#-features) • [How It Works](#-how-it-works) • [Setup](#-setup) • [Commands](#-telegram-bot-commands) • [Architecture](#-architecture)

</div>

---

## 💡 What It Does

Every 5 minutes, Polymarket opens a binary market: **"Will BTC go Up or Down in the next 5 minutes?"**

AutoPoly watches BTC price candles on Coinbase, matches the recent 6-candle pattern against a table of 22 historically-validated patterns, and if there's a match -- executes a Fill-Or-Kill order on the predicted side.

**The edge:** Certain recurring 6-candle sequence patterns (e.g., `DDDDDD`, `UUDUUU`, `DUDUDU`) have historically shown directional bias. The bot trades only on high-confidence patterns and skips everything else.

---

## ⚡ Features

- **Pattern Strategy** -- Matches 6-candle BTC-USD sequences against 22 historically-validated patterns
- **Smart Signal Filtering** -- Only trades on known patterns; skips the rest
- **Live Telegram Dashboard** -- Real-time signal alerts, trade resolutions, P&L analytics
- **Auto-Trading** -- Optional FOK (Fill-Or-Kill) market orders executed automatically
- **Demo Mode** -- Paper-trade with a simulated $1,000 bankroll to test strategies risk-free
- **Auto-Redeem** -- Periodically scans and auto-redeems resolved winning positions on-chain
- **Export Data** -- Download full signal/trade history as CSV or Excel
- **SQLite Persistence** -- All data survives restarts; unresolved signals auto-recover
- **Single-Chat Auth** -- Locked to your Telegram chat ID
- **One-Click Deploy** -- Railway-ready with `Procfile`

---

## ⚙️ How It Works

### The Pattern Strategy

**Every 5 minutes at T-85s** (85 seconds before the current slot ends), the bot:

1. **Fetches** the last 10 confirmed-closed 5-minute BTC-USD candles from Coinbase
2. **Drops** the tail candle (still-forming at T-85s) for data safety
3. **Builds a 6-char pattern** from the newest 6 confirmed candles: `[N-1][N-2][N-3][N-4][N-5][N-6]`
   - `U` = candle closed >= opened (green)
   - `D` = candle closed < opened (red)
4. **Looks up** the pattern in `PATTERN_TABLE` (22 pre-defined patterns)
5. **If matched** → fires a signal with the predicted side (Up or Down)
6. **If no match** → skips this slot, notifies you on Telegram

### The 22 Patterns

| Pattern | Prediction | Pattern | Prediction |
|---------|-----------|---------|------------|
| `DDDDDD` | UP | `UDUUDU` | DOWN |
| `DUUUDU` | DOWN | `DUUDDD` | UP |
| `DUUUUD` | DOWN | `UDDUDD` | DOWN |
| `UDDUUU` | UP | `DUUUUU` | DOWN |
| `DUDDUD` | DOWN | `UUDUUD` | UP |
| `DUUUDD` | DOWN | `DDUDDD` | UP |
| `UDDUUD` | UP | `DUDDDU` | DOWN |
| `DUDUDU` | DOWN | `UUDUUU` | DOWN |
| `UDDDDU` | UP | `DDUDDU` | UP |
| `UUUDUD` | DOWN | | |
| `DUDUUU` | UP | | |
| `UUUUUD` | DOWN | | |
| `DDDUUD` | DOWN | | |

> **Note:** These 22 patterns are currently hardcoded. They were derived from historical analysis. You can extend the table or modify the strategy in `core/strategies/pattern_strategy.py`.

### Signal Flow

```
[Every 5 min at T-85s]
  ├─ 1. PatternStrategy.check_signal() fetches 10 confirmed candles from Coinbase
  ├─ 2. Builds 6-char U/D pattern, looks up in PATTERN_TABLE
  ├─ 3a. Match found → fetches slot prices (Gamma + CLOB), returns signal
  ├─ 3b. No match → returns skip, logs it, notifies you
  ├─ 4. Signal logged to DB
  ├─ 5. TradeManager.check() → always allowed (passthrough)
  ├─ 6. Demo mode → deducts bankroll, creates dummy trade
  ├─ 7. Autotrading → places FOK order with retry logic
  └─ 8. Schedules resolution for slot_end + 30s

[Resolution: slot_end + 30s]
  ├─ 1. Polls Coinbase for the slot's 5-min candle (up to 5 retries)
  ├─ 2. close >= open → "Up", else "Down"
  ├─ 3. P&L calculated: win = amount * (1/entry - 1), loss = -amount
  ├─ 4. Updates signal + trade in DB
  └─ 5. Sends resolution notification on Telegram

[Background jobs]
  ├─ Reconciler: every 5 min, retries resolution for persistent queue items
  └─ Auto-Redeem: every 5 min if enabled, scans & reclaims resolved positions
```

### Order Execution

When autotrading is enabled:
- Uses **Fill-Or-Kill (FOK)** market orders via `py-clob-client`
- **Up to 3 retries** with exponential backoff (2s → 4s → 5s)
- **Time fence:** aborts if < 30 seconds remain in the slot
- **Duplicate guard:** checks DB before each retry to prevent double-fills

### Candle Timing Safety

The bot drops the most recent candle from the Coinbase response because at T-85s, the current 5-minute slot is still open. This ensures the pattern is built from **confirmed, closed candles only**.

---

## 🚀 Setup

### Prerequisites

- **Python 3.10+**
- **Polymarket account** with funded wallet (Polygon)
- **Telegram bot token** from [@BotFather](https://t.me/BotFather)
- **Ethereum private key** for your Polymarket wallet

### Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `POLYMARKET_PRIVATE_KEY` | ✅ | - | Ethereum private key for Polymarket wallet |
| `POLYMARKET_FUNDER_ADDRESS` | ✅ | - | Your Polygon wallet address |
| `POLYMARKET_SIGNATURE_TYPE` | ❌ | `2` | Signature type for CLOB authentication |
| `TELEGRAM_BOT_TOKEN` | ✅ | - | Bot token from @BotFather |
| `TELEGRAM_CHAT_ID` | ✅ | - | Your authorized Telegram chat ID |
| `TRADE_AMOUNT_USDC` | ❌ | `1.0` | Default trade size in USDC |
| `STRATEGY_NAME` | ❌ | `pattern` | Active strategy module name |
| `FOK_MAX_RETRIES` | ❌ | `3` | Maximum FOK order retry attempts |
| `FOK_RETRY_DELAY_BASE` | ❌ | `2.0` | Base retry delay in seconds |
| `FOK_RETRY_DELAY_MAX` | ❌ | `5.0` | Maximum retry delay in seconds |
| `FOK_SLOT_CUTOFF_SECONDS` | ❌ | `30` | Abort order if less than this time remains |
| `AUTO_REDEEM_INTERVAL_MINUTES` | ❌ | `5` | Auto-redeem scan interval |
| `POLYGON_RPC_URL` | ❌ | `https://polygon-rpc.com` | RPC endpoint for on-chain redemptions |
| `DB_PATH` | ❌ | `autopoly.db` | SQLite database file path |

### Telegram Bot Setup

1. Open Telegram and message [@BotFather](https://t.me/BotFather)
2. Send `/newbot` and follow the prompts to create your bot
3. Copy the bot token → set as `TELEGRAM_BOT_TOKEN`
4. Message your new bot, then visit `https://api.telegram.org/bot<TOKEN>/getUpdates`
5. Find your `chat.id` in the response → set as `TELEGRAM_CHAT_ID`

### Local Development

```bash
# Clone the repo
git clone https://github.com/blinkinfo/patbot.git
cd patbot

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
cp .env.example .env  # then edit with your values

# Run
python main.py
```

### Railway Deployment

1. Fork this repository
2. Go to [Railway](https://railway.app) and create a new project from GitHub
3. Add all environment variables in the Railway dashboard
4. Deploy -- the `Procfile` (`worker: python main.py`) runs automatically

---

## 📱 Telegram Bot Commands

### Navigation

| Command | Description |
|---------|-------------|
| `/start` | Welcome message + main menu |
| `/help` | Command reference + strategy explanation |
| `/status` | Portfolio overview, balance, bot uptime, last signal |
| `/settings` | Toggle autotrade, change trade amount, manage demo mode |

### Analytics

| Command | Description |
|---------|-------------|
| `/signals` | Signal performance dashboard - win rate, streaks, recent history |
| `/trades` | Trade P&L dashboard - deployed capital, net P&L, ROI |
| `/demo` | Demo trading dashboard with simulated bankroll tracking |
| `/redemptions` | On-chain redemption history |

### Actions

| Command | Description |
|---------|-------------|
| `/redeem` | Scan and redeem resolved winning positions (dry-run first) |

### Inline Buttons

The bot responds with inline keyboards for:
- **Time filters:** Last 10 | Last 50 | All Time (for signals/trades/demo)
- **Export:** Download CSV or Excel
- **Toggles:** Autotrade on/off, Auto-redeem on/off, Demo mode on/off
- **Inputs:** Trade amount setting, Demo bankroll setting & reset

---

## 🏗️ Architecture

### Project Structure

```
patbot/
├── main.py                    # Entry point - startup, DB init, bot polling
├── config.py                  # Environment config + hardcoded constants
├── requirements.txt           # Python dependencies
├── Procfile                   # Railway: worker: python main.py
│
├── bot/                       # Telegram bot layer
│   ├── handlers.py            # All command & callback handlers
│   ├── keyboards.py           # Inline keyboard layouts
│   ├── formatters.py          # Message formatting utilities
│   └── middleware.py          # Chat ID auth guard
│
├── core/                      # Trading engine
│   ├── strategy.py            # Strategy orchestrator (registry-based)
│   ├── scheduler.py           # APScheduler: trading loop, resolution, reconciliation
│   ├── trader.py              # FOK order execution with retry logic
│   ├── resolver.py            # Slot resolution via Coinbase candles
│   ├── trade_manager.py       # Pre-trade gate (passthrough)
│   ├── redeemer.py            # On-chain CTF redemption via web3.py
│   ├── pending_queue.py       # Persistent retry queue (JSON-backed)
│   └── strategies/            # Strategy plugins
│       ├── __init__.py        # Registry: "pattern" -> PatternStrategy
│       ├── base.py            # Abstract BaseStrategy interface
│       └── pattern_strategy.py # 6-candle pattern matching (THE active strategy)
│
├── db/                        # Database layer
│   ├── models.py              # SQLite schema + init/migrate
│   └── queries.py             # All CRUD + analytics helpers
│
├── polymarket/                # Polymarket API layer
│   ├── client.py              # ClobClient wrapper (L2 credential derivation)
│   ├── markets.py             # Slot boundaries, Gamma + CLOB price fetching
│   └── account.py             # Balance, positions, connection status
│
└── data/                      # Runtime data (auto-created)
    └── pending_slots.json     # Persistent unresolved slot queue
```

### Database Schema

**4 Tables:**

| Table | Purpose |
|-------|---------|
| `signals` | Every signal check - side, price, match/skip, win/loss |
| `trades` | Executed/Filled orders - amount, P&L, retry count, status |
| `settings` | Key-value config - autotrade, trade amount, demo mode |
| `redemptions` | On-chain redemption records - tx hash, gas, status |

Default settings seeded on first run:
- `autotrade_enabled`: false
- `demo_trade_enabled`: false
- `auto_redeem_enabled`: false
- `demo_bankroll_usdc`: 1000.00
- `trade_amount_usdc`: from config

### Key Technical Decisions

- **Async-first:** `aiosqlite` and `httpx.AsyncClient` throughout; `py-clob-client` (sync) wrapped in `asyncio.to_thread()`
- **Strategy Registry:** Strategies are pluggable via `core/strategies/__init__.py` -- add new strategies by registering them there
- **Graceful Degradation:** If Coinbase API is unavailable, the bot skips the slot and retries next cycle
- **Persistent Queue:** Unresolved slots stored in `data/pending_slots.json` and retried by the reconciler every 5 minutes
- **Startup Recovery:** On boot, immediately resolves any unresolved signals from previous run

---

## 📦 Dependencies

```
py-clob-client>=0.34.0    # Polymarket order execution
python-telegram-bot>=20.0 # Telegram bot framework (async)
httpx>=0.25.0             # HTTP client for APIs
apscheduler>=3.10.0       # Task scheduling
python-dotenv>=1.0.0      # Environment variable loading
aiosqlite>=0.19.0         # Async SQLite
openpyxl>=3.1.0           # Excel export
web3>=6.0.0               # On-chain redemption transactions
```

---

## 🔄 Extending: Adding New Strategies

The bot supports pluggable strategies:

1. Create a new class in `core/strategies/your_strategy.py` that extends `BaseStrategy`
2. Implement `async def check_signal() -> dict[str, Any] | None`
3. Register it in `core/strategies/__init__.py`: `STRATEGIES["your_strategy"] = YourStrategy`
4. Set `STRATEGY_NAME=your_strategy` in your environment variables

See `pattern_strategy.py` for a complete reference implementation.

---

## ⚠️ Risk Warning

**This is experimental software.** Trading binary options carries significant risks. The pattern strategy is based on historical analysis but **does not guarantee future results**. Only trade with funds you can afford to lose.

- Always test in **demo mode** first using `/settings` > toggle demo trading
- Monitor resolution accuracy via `/signals` and `/demo` before enabling real trades
- The bot makes autonomous trading decisions -- review your strategy regularly

---

## 📄 License

MIT
