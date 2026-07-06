# Optimization

## Period scaling

Tune keys ending in `_period` or `_periods` are auto-scaled by the engine:

```python
scaled_value = value * (86400 / candle_size)
```

A value of `75.0` with hourly candles becomes `75 * (86400 / 3600) = 1800`.

## Clamps format

```python
self.clamps = {
    "param_name": [min, midpoint, max, clamp_flag],
}
```

- `min`, `max` — search bounds
- `midpoint` — initial/default value
- `clamp_flag` — `1` to optimize, `0` to skip

## Optimizers

| Optimizer | Description |
|-----------|-------------|
| QPSO | Quantum-behaved Particle Swarm |
| LSGA | Linear Selection Genetic Algorithm |
| IPSE | Iterative Parameter Sweep Engine |
| AION | AI-driven optimizer |
| MouseWheelTuner | Interactive manual tuning |

## Tune caching

Optimized tunes are saved per-bot in a `tunes/` directory next to the bot
source file. The `load_tune()` helper loads the best known tune.
