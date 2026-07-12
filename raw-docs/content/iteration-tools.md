# Iteration Tools: AutoBacktest & Gravitas

You're iterating on indicator logic. Tweak, save, run. Tweak, save, run. Each manual cycle costs seconds of context switching — just enough to break flow.

Or you've added a parameter and want to know how sensitive the strategy is to it. Is ROI changing meaningfully, or are you just turning a knob that doesn't matter?

Two tools handle these two modes of iteration.

## AutoBacktest

### The problem

You change one line in your bot — adjust an EMA period, flip a comparison sign — and then:

1. Switch to the terminal.
2. Press up-arrow to find the last `python bot.py` command.
3. Wait for the import, autorange warmup, and backtest to run.
4. Look at the output. Decide on the next tweak.
5. Repeat.

That switch-and-wait cycle adds up. Over an hour of tuning, it steals minutes. Over a week, it can cost more time than the optimization runs.

### The solution

`auto_backtest(bot, data)` watches your bot script for file changes and re-runs the backtest automatically.

```python
import qtradex as qx
from my_bot import EmaCrossover

bot = EmaCrossover()
data = qx.data("BTC", "1d", 365)

qx.auto_backtest(bot, data)
```

Save your bot file. The terminal window that's running `auto_backtest` picks up the change and runs the backtest again. The plot updates. You see if the change helped.

### What happens on reload

When it detects a save, `auto_backtest` reloads the bot module, rebuilds the class, and re-runs the full cycle.

If the new version of your bot doesn't redefine `self.tune`, the previous tune values carry over. You can tweak indicator logic without losing your parameters. If you add, remove, or rename a tune key, the original tune no longer fits — the bot falls back to defaults.

### When to use it

AutoBacktest is for logic iteration — changing indicators, strategy rules, signal logic. It's not for tuning parameters. If you're trying to find the right `fast_ema` value, use the optimizer. AutoBacktest just saves the manual re-run step.

Stop it with Ctrl-C.

## Gravitas

### The concept

Gravitas is a single scalar you can reference in your strategy. It's not a tune parameter. The framework doesn't read or act on it. It's just a dial you provide.

```python
class MyBot(qx.BaseBot):
    def __init__(self):
        self.gravitas = 1.0
```

In your strategy or indicators, reference it anywhere:

```python
def indicators(self, data, states):
    threshold = self.gravitas * 1.5
    ...
```

The dial does nothing unless your code uses it. That's the point — you decide what it means.

### Running the scan

From the dispatch optimizer menu, select "Gravitas". The framework prompts for three values:

- **Min gravitas** — the low end of the dial
- **Max gravitas** — the high end
- **Number of tests** — how many points between min and max to evaluate

It runs a backtest at each gravitas value and plots ROI vs gravitas.

### Reading the plot

The shape of that plot tells you something about your strategy.

A flat line means your bot doesn't reference `self.gravitas` anywhere, or the logic referencing it has no effect on results. Every data point is the same backtest.

A slope means the dial affects your strategy. The steeper the slope, the more sensitive.

A curve with a peak tells you the optimal operating range. If ROI rises to a plateau then drops, you want the flattest part of that peak — the region where small changes in the dial don't swing results much. That's where your strategy is robust.

### When to use it

Gravitas is for sensitivity analysis. Before you commit to a threshold or scaling factor, run the scan to see what range produces stable results. If a 10% change in gravitas swings ROI by 50%, you're in a fragile region — the strategy depends too heavily on the exact value.

## Which to use when

| Situation | Tool |
|---|---|
| Changing indicator logic or strategy rules | AutoBacktest |
| Sensitivity analysis on a threshold or scaling factor | Gravitas |

AutoBacktest is for speed. Gravitas is for understanding.

---

## Next steps

- [AutoBacktest API reference](api/auto-backtest.md)
- [Gravitas API reference](api/gravitas.md)
- [Advanced bot building](advanced-bot-building.md) — custom warmup, plotting, lifecycle hooks
