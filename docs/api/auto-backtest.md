# AutoBacktest

```python
from qtradex.core.auto_backtest import auto_backtest
```

Watches your bot script for changes and re-runs the backtest automatically. Saves the iteration cycle — edit, save, see results instantly.

```python
auto_backtest(bot, data, wallet=None, **kwargs)
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `bot` | `BaseBot` | required | Bot instance (the current version of your bot) |
| `data` | `Data` | required | Market data |
| `wallet` | `PaperWallet` | `None` | Starting wallet (default: 1 unit currency, 0 asset) |
| `**kwargs` | — | — | Forwarded to internal `backtest()` calls; `plot` and `block` are forced |

Runs in an infinite loop. Every second it checks whether the bot's source file has changed. On change, it reloads the module, preserves the original tune if the new bot doesn't redefine it, re-runs the backtest, and updates the plot. Hit Ctrl-C to stop.

## What survives a reload

If the new version of your bot defines a different `self.tune`, the new tune is used. Otherwise the original tune carries over — so you can tweak indicator logic without losing your parameters.
