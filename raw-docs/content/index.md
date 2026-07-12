# QTradeX Docs

!!! info "SDK commit reference"
    These docs correspond to SDK commit [`e5f62bd`](https://github.com/squidKid-deluxe/QTradeX-Algo-Trading-SDK/commit/e5f62bd). Run `git pull` or `pip install qtradex` to ensure your SDK matches.

```python
import qtradex as qx

class EMACrossBot(qx.BaseBot):
    def indicators(self, data):
        return {
            "fast_ema": qx.ti.ema(data["close"], 10),
            "slow_ema": qx.ti.ema(data["close"], 50),
        }
    def strategy(self, state, indicators):
        if indicators["fast_ema"] > indicators["slow_ema"]:
            return qx.Buy()
        elif indicators["fast_ema"] < indicators["slow_ema"]:
            return qx.Sell()
        return qx.Hold()
```

A bot class. Two indicators. A decision rule. That's the skeleton of every QTradeX strategy.

[Get Started &rarr;](getting-started.md)

---

### Explore

- **[Advanced Bot Building](advanced-bot-building.md)** — custom warmup, plot overrides, execution hooks
- **[Optimization](optimization.md)** — QPSO, LSGA, IPSE, AION
- **[Live Trading](live-trading.md)** — paper trade, live orders, authentication
- **[API Reference](api/basebot.md)** — BaseBot, Data, signals, optimizers

---

### Community & Resources

- **GitHub** — [QTradeX-Algo-Trading-SDK](https://github.com/squidKid-deluxe/QTradeX-Algo-Trading-SDK) — source code, issues, star history
- **DeepWiki** — [QTradeX SDK](https://deepwiki.com/squidKid-deluxe/QTradeX-Algo-Trading-SDK) — auto-generated AI-readable docs
- **AI Agents** — [QTradeX-AI-Agents](https://github.com/squidKid-deluxe/QTradeX-AI-Agents) — 22+ community bot strategies to learn from (or run as-is)
- **Telegram** — [@qtradex_sdk](https://t.me/qtradex_sdk) — discussion, support, and strategy talk
- **AI Strategies DeepWiki** — [QTradeX AI Agents](https://deepwiki.com/squidKid-deluxe/QTradeX-AI-Agents) — deep-dive into each community bot
