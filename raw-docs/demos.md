# Demos

The SDK ships with several demo scripts in the `demos/` directory. These are self-contained examples you can run directly to explore QTradeX features.

| Demo | File | Description |
|------|------|-------------|
| **Extinction Event** | `demos/extinction_event.py` | A complete EMA crossover bot with multi-level support/resistance channels. Demonstrates `BaseBot` subclassing, `tune`/`clamps`, multi-indicator `indicators()`, trend-detect `strategy()`, and `plot()` with shaded zones. Runs standalone — uses `qx.dispatch()`. |
| **Extinction Event v2** | `demos/extinction_event_v2.py` | Architectural preview of the upcoming multi-asset engine (supports N assets). Uses `qx.Allocation` and `qx.Limit` signals, multi-asset `Data(assets=[...])`, and the new `execution(allocation, indicators, wallet, prices)` signature. Mirrors v1 for result comparison once the v2 engine is built. Not yet ready for production use. |
| **Skew Landscape** | `demos/demo_skew_landscape.py` | Visualizes how LSGA's 2D skew memory evolves over generations. Replaces `qx.backtest` with a fake noise-landscape backtest and animates the optimizer's population. |
| **Baseline Comparison** | `demos/run_v1_baseline.py` | Baseline performance benchmark against a previous version of the strategy. |

Run any demo directly:

```bash
python demos/extinction_event.py
```

Or use dispatch for interactive mode:

```python
import qtradex as qx
from extinction_event import ExtinctionEvent

bot = ExtinctionEvent()
data = qx.Data("binance", "BTC", "USDT", days=365)
qx.dispatch(bot, data)
```

---

## Community Strategies

The [QTradeX-AI-Agents](https://github.com/squidKid-deluxe/QTradeX-AI-Agents) repo collects curated trading strategies built on the SDK — 22 bots including EMA crossovers, multi-indicator confluence strategies, Renko-RSI hybrids, Fourier-filtered signals, and the I-Ching hexagram bot. Known-good strategies are noted in the README. Clone it and drop them into your workflow.
