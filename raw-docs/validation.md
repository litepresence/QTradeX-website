# Validation: Monte Carlo & Fill Test

Your optimizer found parameters with amazing backtest results. 200% ROI, smooth equity curve, great Sharpe.

But are those results real, or did you just get lucky?

The optimizer tests hundreds of parameter combinations and picks the best one. With enough trials, some combination will look great by chance — even on random noise. Validation tells you whether your results are signal or overfit.

---

## Monte Carlo

Monte Carlo answers one question: if you perturbed every tune parameter slightly, would your strategy still work?

A strategy that collapses when a parameter moves 1% is fragile. A robust strategy handles small variations gracefully.

### How it works

Pass your bot and data to `monte_carlo`. It runs hundreds of backtests, each with a slightly jittered copy of your tune:

```python
from qtradex.core.monte_carlo import monte_carlo

results = monte_carlo(bot, data, iterations=500, perturbation=0.01)
```

Every parameter within `bot.clamps` gets a random jitter of `±perturbation × (max − min)`. A perturbation of 0.01 with a clamp range of [20, 100] means each parameter shifts by up to ±0.8. The baseline run keeps your original tune. All runs are parallelized across CPU cores.

### Reading the chart

The plot shows a gray fan of individual runs, with colored curves overlaid:

- **Baseline** — your original tune's performance
- **Mean** — the geometric mean across all perturbed runs
- **P5** — the 5th percentile (worst 5% of outcomes)
- **P95** — the 95th percentile
- **Buy & hold** — what you'd get doing nothing
- **Sell & hold** — sitting out entirely
- **Random trade** — random entry and exit

### What to look for

If **P5 is below buy & hold**, that's a red flag. It means 5% of reasonable parameter variations perform worse than doing nothing. Your strategy is sensitive to parameter choices — small changes could wipe out your edge.

**Skew_2d** measures the distribution's shape:

| Value | Meaning |
|-------|---------|
| 0 | Worst-bound — most runs underperform the baseline |
| 0.5 | Centered — symmetric spread |
| 1 | Best-bound — most runs beat the baseline |

A low skew means the baseline sits at the edge of a cliff. One bad parameter combo and you're down.

### When to tune iterations and perturbation

Start with 500 iterations and 0.01 perturbation. Increase **iterations** when the fan is too sparse to read — more runs fill in the shape. Increase **perturbation** when you want to stress-test wider parameter shifts. Decrease it for tighter analysis around your current best.

### Return values

```python
{
    "baseline_final": float,   # Baseline final portfolio value
    "mean_f": float,           # Geometric mean final value
    "std_f": float,            # Standard deviation
    "p5": float,               # 5th percentile
    "p95": float,              # 95th percentile
    "skew_2d": float,          # Distribution shape (0–1)
    "mean_curve": ndarray,     # Mean portfolio curve
    "p5_curve": ndarray,       # 5th percentile curve
    "p95_curve": ndarray,      # 95th percentile curve
}
```

---

## Fill Test

Monte Carlo tests parameter sensitivity. Fill Test tests something different: whether your backtest assumptions match reality.

Every backtest makes assumptions about fills — that you buy at the open, sell at the close, never slip. Real exchanges don't work that way. Your actual fills have slippage, latency, partial fills, and exchange-specific quirks that backtests can't model.

Fill Test answers: did your bot's signals align with the fills you actually got?

### How it works

```python
from qtradex.core.filltest import filltest

filltest(bot, data, api_key, api_secret)
```

It fetches your trade history from the exchange, then walks backward tick by tick, checking whether your bot's strategy agrees with each historical fill. At every tick where a fill exists, it replays the indicator state at that moment and compares it to the signal your bot would have generated.

### When to use it

Run Fill Test after you've been live trading for a month or more. Use it before scaling up — if your signals don't match your fills, something is wrong with your backtest assumptions or your execution. It's also useful for catching configuration drift: did an exchange API change break your bot's logic?

### The plot

The balance curve shows trade markers colored by fill type — green for buys, red for sells. If your bot's signals line up with the fills, the markers tell a coherent story. If they don't, you'll see trades that contradict your strategy, which means your backtest was running on different assumptions than reality.

---

## Your workflow

Run QPSO and get 200% ROI. Good.

Now run Monte Carlo. If P5 is below buy & hold, your strategy is fragile. Switch to LSGA, which has built-in overfit protection via walk-forward validation, an annealed drawdown gate, and skew memory. Run Monte Carlo again. Keep iterating until P5 stays above buy & hold.

Always test on data you did not optimize on. The optimizer saw your full date range. Monte Carlo perturbs parameters but on the same data. The real test is a reserved out-of-sample period — set aside the last 20-30% of your data before optimizing, then run your best parameters against that unseen slice. If performance drops sharply, your strategy is overfit to the optimization window.

After you've traded for a month, run Fill Test. If your signals matched your fills, you can scale up with confidence. If they didn't, investigate what changed — it might be slippage, fill logic, or a misconfigured parameter.

Validation isn't a one-time step. Run Monte Carlo after every optimization pass. Run Fill Test after every trading month. They're your checks against the two biggest strategy killers: overfitting and reality mismatch.

> **Disclaimer.** This is not financial advice. QTradeX is a software framework for strategy development and backtesting. Past performance — whether from backtests, Monte Carlo simulations, or fill tests — does not guarantee future results. Trading real markets involves risk, including the loss of principal. No amount of validation eliminates that risk.

## Next steps

- [Monte Carlo API Reference](api/monte-carlo.md) — full parameter docs
- [Fill Test API Reference](api/filltest.md) — full parameter docs
- [Optimization](optimization.md) — QPSO, LSGA, and the other optimizers
