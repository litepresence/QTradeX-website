# Getting Started

## Installation

```bash
pip install qtradex

Optional extras:

- `pip install qtradex[rl]` — RLPPO optimizer (installs PyTorch + stable-baselines3)
- `pip install qtradex[bitshares]` — BitShares decentralized exchange support
```

Published to PyPI on every release — `pip install` always gets the latest version.

> **Dev install**: `git clone` + `pip install -e .` if you want to modify QTradeX core itself.
> **Platform**: Python >= 3.9, Cython >= 3, numpy < 2. Primary target is Linux. WSL works too — that's how most Windows users should run it.

---

## Your first bot

You'll build an EMA crossover bot piece by piece. Each step adds one concept — by the end you'll see how they fit together.

### Step 1: What's a bot?

A trading bot looks at candle data and decides what to do. In QTradeX you write a class:

```python
import qtradex as qx

class EMACrossBot(qx.BaseBot):
    pass
```

That empty class already *is* a bot. It won't do anything interesting yet — but it's enough to prove the wiring works.

### Step 2: Tell it what to tweak

Bots have knobs — parameters you can tune later to find what works best. You define them in `__init__`:

```python
class EMACrossBot(qx.BaseBot):
    def __init__(self):
        self.tune = {
            "fast_ema_period": 10.0,
            "slow_ema_period": 50.0,
        }
```

Two knobs: a fast EMA (10 periods) and a slow one (50). The numbers are your starting guesses — optimizers try values around them.

Set boundaries so the optimizer doesn't try nonsense values:

```python
        self.clamps = {
            "fast_ema_period": [5, 7.5, 50, 1],
            "slow_ema_period": [20, 35, 100, 1],
        }
```

Each entry has `[min, midpoint, max, clamp_flag]`. The optimizer searches between min and max starting from the midpoint. Set `clamp_flag=0` to skip a parameter during optimization.

### Step 3: Periods are days, not candles

Keys ending in `_period` are always treated as days, not candles. The engine scales them to whatever candle size you feed the bot.

Set `"fast_ema_period": 10.0` and with hourly data the engine uses 240 periods (10 days × 24 hours). With daily data it uses 10. You think in days, it handles the math.

Periods can also be **floats**, not just integers. That's unusual for indicators — most frameworks force you to pick SMA(14) or SMA(15), with nothing in between. QTradeX blends the two:

```
A = qx.ti.sma(close, 14)
B = qx.ti.sma(close, 15)

qx.ti.sma(close, 14.3)  →  0.7 * A + 0.3 * B
```

**Why floats?** During optimization, the optimizer needs to know which direction improves performance. Integer periods create a staircase — period 14 and period 15 are both valid, but the optimizer can't tell if 14.3 is heading toward 14 or toward 15. Float periods smooth that staircase into a slope the optimizer can follow. The best period might be 14.7, and with floats the optimizer can find it.

This is applied automatically to all built-in indicators. You can also add it to custom indicators — see the [indicators API reference](api/indicators.md#float_period-decorator) for the decorator syntax.

This also drives automatic warmup — next step.

### Step 4: Add indicators

Indicators transform raw candle data into signals your strategy can use. You return a dict of them:

```python
    def indicators(self, data):
        return {
            "fast_ema": qx.ti.ema(data["close"], self.tune["fast_ema_period"]),
            "slow_ema": qx.ti.ema(data["close"], self.tune["slow_ema_period"]),
        }
```

`qx.ti` is QTradeX's wrapper around Tulip Indicators — 100+ technical indicators, all vectorized. Each call returns a numpy array, one value per candle.

But there's a problem. Your strategy runs on every candle starting from candle zero. The EMA needs 10 periods of data before a real value exists. Watch what happens before that:

```python
>>> qx.ti.ema([1, 2, 3], 10)
array([nan, nan, nan])
```

Your first trades would be on `NaN`. Garbage in, garbage out.

That warmup is handled automatically. `BaseBot.autorange()` scans your tune values, finds the largest `_period` key, and tells the engine to skip that many candles before calling `strategy()`. Your strategy never sees an uninitialized indicator.

The default is already correct for this bot. You only override it for custom warmup (Advanced Bot Building covers that). The engine handles warmup — you write the logic for when the bot is ready to trade.

### Step 5: Make decisions — the strategy

The strategy method is the bot's brain. It runs once per candle, receives the current tick info and your computed indicators, and returns a signal:

```python
    def strategy(self, state, indicators):
        fast = indicators["fast_ema"]
        slow = indicators["slow_ema"]
        if fast > slow:
            return qx.Buy()
        elif fast < slow:
            return qx.Sell()
        return qx.Hold()
```

When the fast EMA crosses above the slow EMA, buy. When it crosses below, sell. Otherwise, do nothing.

QTradeX gives you four signals:

| Signal | What it does |
|--------|-------------|
| `qx.Buy()` | Market buy |
| `qx.Sell()` | Market sell |
| `qx.Thresholds(buying, selling)` | Limit orders — fill only when price crosses your level |
| `qx.Hold()` | Do nothing |

Pass `reason="..."` to tag a trade. The string shows up in backtest results so you can see why it happened:

```python
return qx.Buy(reason="fast EMA crossed above slow EMA")
```

### Step 6: See what happened

Charts make backtests real. Override `plot` to pick which indicators to draw:

```python
    def plot(self, *args):
        qx.plot(self.info, *args, (
            ("fast_ema", "EMA Fast", "white", 0, "EMA Cross"),
            ("slow_ema", "EMA Slow", "cyan", 0, "EMA Cross"),
        ))
```

Each tuple describes one indicator: the key name, the label, the color, which y-axis pane, and the axis group. You can overlay multiple indicators on the same pane or split them across separate ones.

### Step 7: Give it data and run

```python
data = qx.Data(
    exchange="kucoin",
    asset="BTC",
    currency="USDT",
    begin="2020-01-01",
    end="2023-01-01",
)
bot = EMACrossBot()
qx.dispatch(bot, data)
```

`qx.Data()` fetches historical candles via CCXT (100+ exchanges) and caches them so subsequent runs are instant.

`qx.dispatch()` opens an interactive menu. Select "Backtest" to see how your bot performed. (You can also optimize, paper trade, or go live from the same menu.)

Save it all in `ema_bot.py` and run:

```bash
python ema_bot.py
```

## What you just built

- A bot class with tunable parameters (`tune`, `clamps`)
- Two technical indicators (`indicators`)
- A decision rule that reads them (`strategy`)
- A visual plot (`plot`)
- A data source and the dispatch runner

That's the full skeleton. Every QTradeX bot follows this same structure — only the logic inside each method changes. The framework handles backtesting, optimization, caching, and deployment so you can focus on the strategy itself.

## Next steps

Your first bot runs. Now you've got options depending on what you want to do next:

- **[Advanced Bot Building](advanced-bot-building.md)** — custom autorange, plot overrides, execution hooks
- **[Optimization](optimization.md)** — tune parameters automatically with QPSO, LSGA, IPSE, or AION
- **[Live Trading](live-trading.md)** — paper trade first, then go live with real exchange orders
- **[API Reference](api/basebot.md)** — BaseBot, Data, signals, and optimizers reference
- **Community** — [GitHub](https://github.com/squidKid-deluxe/QTradeX-Algo-Trading-SDK) / [DeepWiki](https://deepwiki.com/squidKid-deluxe/QTradeX-Algo-Trading-SDK) / [Telegram](https://t.me/qtradex_sdk) / [AI Agents](https://github.com/squidKid-deluxe/QTradeX-AI-Agents)
