# Getting Started

## Installation

```bash
pip install qtradex
```

For the latest development version:

```bash
git clone https://github.com/squidKid-deluxe/QTradeX-Algo-Trading-SDK.git QTradeX
cd QTradeX
pip install -e .
```

Requires Python >= 3.9, Cython >= 3, and numpy < 2. Linux only.

## Your first bot

Here's a complete EMA crossover bot:

```python
import qtradex as qx

class EMACrossBot(qx.BaseBot):
    def __init__(self):
        self.tune = {"fast_ema_period": 10.0, "slow_ema_period": 50.0}
        self.clamps = {
            "fast_ema_period": [5, 10, 50, 1],
            "slow_ema_period": [20, 50, 100, 1],
        }

    def indicators(self, data):
        return {
            "fast_ema": qx.ti.ema(data["close"], self.tune["fast_ema"]),
            "slow_ema": qx.ti.ema(data["close"], self.tune["slow_ema"]),
        }

    def strategy(self, tick_info, indicators):
        fast = indicators["fast_ema"]
        slow = indicators["slow_ema"]
        if fast > slow:
            return qx.Buy()
        elif fast < slow:
            return qx.Sell()
        return qx.Thresholds(buying=fast * 0.8, selling=fast * 1.2)

    def plot(self, *args):
        qx.plot(self.info, *args, (
            ("fast_ema", "EMA 1", "white", 0, "EMA Cross"),
            ("slow_ema", "EMA 2", "cyan", 0, "EMA Cross"),
        ))

data = qx.Data(exchange="kucoin", asset="BTC", currency="USDT",
               begin="2020-01-01", end="2023-01-01")
bot = EMACrossBot()
qx.dispatch(bot, data)
```

## Running your bot

Save the file, run it:

```bash
python my_bot.py
```

`qx.dispatch()` opens an interactive CLI where you can backtest, optimize, paper trade, or go live.
