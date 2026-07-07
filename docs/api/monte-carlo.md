# Monte Carlo

```python
from qtradex.core.monte_carlo import monte_carlo
```

Runs your bot on hundreds of slightly perturbed versions of your tune to test whether the strategy is robust or just overfit to one parameter set. Parallelized across CPU cores.

```python
results = monte_carlo(bot, data, wallet=None,
                      iterations=500, perturbation=0.01,
                      plot=True, block=True, **kwargs)
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `bot` | `BaseBot` | required | Bot instance |
| `data` | `Data` | required | Market data |
| `wallet` | `PaperWallet` | `None` | Starting wallet (default: 1 unit currency, 0 asset) |
| `iterations` | `int` | `500` | Number of perturbed tune runs |
| `perturbation` | `float` | `0.01` | Each tune parameter is jittered by ±`perturbation` × span of its clamp range |
| `plot` | `bool` | `True` | Show the Monte Carlo chart |
| `block` | `bool` | `True` | Block until plot window closes |
| `**kwargs` | — | — | Forwarded to internal `backtest()` calls |

Each perturbed run keeps the same data — only `bot.tune` changes. Every parameter within `bot.clamps` gets a random jitter within `±perturbation × (max − min)`. The baseline run uses the original tune.

## Return value

```python
{
    "baseline_final": float,   # Baseline final portfolio value (normalized)
    "baseline_ret": dict,      # Baseline metrics dict
    "baseline_vals": ndarray,  # Baseline ROI curve per tick
    "mean_f": float,           # Geometric mean final value across runs
    "std_f": float,            # Std dev of final values
    "p5": float,               # 5th percentile final value
    "p95": float,              # 95th percentile final value
    "skew_2d": float,          # 2D skew (0 = worst-bound, 0.5 = centered, 1 = best-bound)
    "mean_curve": ndarray,     # Mean portfolio curve per tick
    "p5_curve": ndarray,       # 5th percentile curve
    "p95_curve": ndarray,      # 95th percentile curve
    "timestamps": list,        # x-axis timestamps
    "all_aligned": ndarray,    # All individual curves interpolated to baseline timeline
    "mc_results": list,        # [{unix, roi}] for each run
}
```

## Chart

The plot overlays the baseline, buy & hold, sell & hold, and random-trade benchmarks against a gray fan of individual MC runs, plus the mean, P5, and P95 curves. If a strategy's P5 is below buy & hold, that's a red flag — the strategy may be sensitive to parameter choices.
