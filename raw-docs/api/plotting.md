# Plotting

```python
from qtradex import plot, plotmotion
```

## `plot()`

```python
axes = plot(
    info,
    data,
    states,
    indicators,
    block,
    indicator_fmt,
    style="dark_background",
)
```

Called by the engine after a backtest, or explicitly from a bot's `plot()` override. Returns a list of `matplotlib.axes.Axes` objects.

| Parameter | Type | Description |
|-----------|------|-------------|
| `info` | `dict` | Bot's `self.info` — contains `mode`, `start`, `live_data`, `live_trades` |
| `data` | `Data` | The data object (provides `.asset`, `.currency`, `.exchange` for labels) |
| `states` | `dict` | Rotated backtest states dict (keys: `unix`, `open`, `high`, `low`, `close`, `trades`, `balances`, ...) |
| `indicators` | `dict` | Dict of indicator name → numpy array |
| `block` | `bool` | `True` = block until window closes, `False` = interactive mode |
| `indicator_fmt` | `list[tuple]` | Format spec — see below |
| `style` | `str` | Matplotlib style name (default: `"dark_background"`) |

### `indicator_fmt`

A list of 5-element tuples, one per indicator line to draw:

```python
indicator_fmt = [
    (key, name, color, idx, title),
    ...
]
```

| Position | Name | Type | Description |
|----------|------|------|-------------|
| 0 | `key` | `str` | Key into the `indicators` dict |
| 1 | `name` | `str` | Legend label |
| 2 | `color` | `str` | Matplotlib color for the line |
| 3 | `idx` | `int` | Subplot row: `0` = main price axis, `>=1` = dedicated subplot below |
| 4 | `title` | `str | None` | Watermark title for that subplot panel (becomes main title if `idx=0`) |

Example that puts a moving average on the price axis and RSI below:

```python
indicator_fmt = [
    ("ma_50", "MA 50", "yellow", 0, None),
    ("rsi", "RSI", "cyan", 1, "RSI"),
]
```

### Return value

`axes` is a list of `matplotlib.axes.Axes` objects:

| Index | Content |
|-------|---------|
| `axes[0]` | Main price axis (log-scale, candle body, trade markers) |
| `axes[1]` | First indicator subplot (`idx=1`) |
| `axes[n]` | nth indicator subplot |
| `axes[-1]` | Balances axis (bottom, log-scale) |

The engine synchronizes x-limits across all axes. All axes except the last hide x-tick labels.

## `plotmotion()`

```python
plotmotion(block)
```

Toggles matplotlib's interactive mode:

| Parameter | Behavior |
|-----------|----------|
| `block=True` | `plt.ioff(); plt.show()` — blocks until window closes |
| `block=False` | `plt.ion(); plt.pause(0.00001)` — returns immediately, updates on next call |

Used internally by the engine. In a custom `plot()` override you typically don't need to call it — the engine handles the final `plotmotion()` call.
