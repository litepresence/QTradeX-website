# Fill Test

```python
from qtradex.core.filltest import filltest
```

Simulates what your bot *would have done* against your actual historical fills from the exchange. Useful for reconciling backtest assumptions with reality — did your bot's signals align with the fills you actually got?

```python
filltest(bot, data, api_key, api_secret, tick_size=7200)
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `bot` | `BaseBot` | required | Bot instance |
| `data` | `Data` | required | Market data (begin/end adjusted automatically to match fill window) |
| `api_key` | `str` | required | Exchange API key |
| `api_secret` | `str` | required | Exchange API secret |
| `tick_size` | `int` | `7200` | Tick size in seconds (default 2 hours). Period params are scaled: `tune_value × (86400 / tick_size)` |

Returns `None`. Plots the results with trade markers.

## How it works

1. Fetches your trade history from the exchange via `execution.fetch_my_trades()`.
2. Walks backward from `now`, tick by tick, matching fills to candles.
3. At each tick where a fill exists, checks whether your bot's strategy agrees — replaying the indicator state at that moment.
4. Plots the resulting balance curve with trade markers colored by fill type (green for buys, red for sells).

This is the same function called by dispatch's "Show Fill Orders" option.
