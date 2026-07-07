# Gravitas

```python
bot.gravitas = 1.0   # default: unset
```

A hook for sensitivity analysis. The framework doesn't read or act on `bot.gravitas` — it's a value you can optionally reference in your bot's `indicators()`, `strategy()`, or `execution()` to make behavior depend on a single dial.

Available as an optimizer option in `dispatch()`. Select "Gravitas" from the optimizer menu to launch `plot_gravitas()`.

## `plot_gravitas()`

```python
from qtradex.core.dispatch import plot_gravitas
plot_gravitas(bot, data, wallet, **kwargs)
```

Prompts for min gravitas, max gravitas, and number of tests, then runs a backtest at each value and plots ROI vs gravitas. Prompts:

| Prompt | Default | Description |
|--------|---------|-------------|
| Min Gravitas | `0.3` | Lower bound |
| Max Gravitas | `1.7` | Upper bound |
| Number of tests | `200` | Steps between min and max |

## How to use it

The gravitas scan is useful only if your bot references `self.gravitas`. For example, scaling a tune parameter:

```python
class MyBot(qx.BaseBot):
    def __init__(self):
        self.tune = {"threshold": 0.5}
        self.clamps = {"threshold": [0.1, 0.55, 1.0, 1]}

    def strategy(self, state, indicators):
        # Gravitas amplifies or dampens the threshold
        g = getattr(self, "gravitas", 1.0)
        threshold = self.tune["threshold"] * g
        if indicators["rsi"] > 70 * threshold:
            return qx.Sell()
        ...
```

Run `plot_gravitas()` from dispatch's optimizer menu. If your bot doesn't reference `gravitas`, the chart will be flat — every data point is the same backtest.
