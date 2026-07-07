# Advanced Bot Building

A working bot is just the start. Real strategies need custom warmup, richer charts, data alignment across indicators, and lifecycle hooks. Each section below starts with a problem you'll hit — then the tool that solves it.

## Custom autorange

The default `autorange()` scans your tune for the largest `_period` value and skips that many days of warmup. That works for most bots.

But say your custom indicator needs 500 days of data to stabilize — more than any single `_period` key implies. Your strategy receives garbage until the indicator warms up.

Override `autorange()` to set a higher floor:

```python
class MyBot(qx.BaseBot):
    def autorange(self):
        return 250  # 250 days of warmup minimum
```

Return an integer number of days. The engine handles the candle-size conversion. You can read `self.tune` and compute whatever you need — for example, if your custom indicator needs more warmup than any single tune parameter suggests.

## Plot overrides

`qx.plot()` draws indicator lines and trade markers. That covers most cases. But what if you want shaded buy/sell zones instead of just line crossovers?

You can get the raw matplotlib Axes objects and call any API on them.

### Fill a single channel

Start with one shaded zone — say a green buy region between a support and selloff line:

```python
def plot(self, data, states, indicators, block):
    axes = qx.plot(self.info, data, states, indicators, False, (
        ("ma3", "LONG", "white", 0, "Extinction Event"),
    ))

    axes[0].fill_between(
        states["dates"],
        indicators["selloff"],
        indicators["support"],
        color="lime", alpha=0.3,
        where=[t == "bull" for t in indicators["trend"]],
        step="post",
    )

    qx.plotmotion(block)
```

`axes[0]` is a matplotlib Axes object — anything you can do with matplotlib, you can do here.

### Add a second channel

Now overlay a red sell zone between despair and resistance lines:

```python
    axes[0].fill_between(
        states["dates"],
        indicators["resistance"],
        indicators["despair"],
        color="tomato", alpha=0.4,
        where=qx.expand_bools([t == "bear" for t in indicators["trend"]]),
        step="post",
    )

    axes[0].legend()
```

### `qx.expand_bools`

Expands boolean regions by one tick so single-candle flips don't create zero-width wedges in `fill_between`:

```python
qx.expand_bools([False, True, True, False])         # → [True, True, True, True]  (both sides)
qx.expand_bools([False, True, True, False], "right") # → [False, True, True, True]
```

### `qx.plotmotion(block)`

Controls whether the plot window blocks execution. Pass `block=False` to `qx.plot()` if you need to add custom matplotlib calls after it, then call `qx.plotmotion(block)` yourself to render:

```python
axes = qx.plot(self.info, data, states, indicators, False, (...))
# ... custom matplotlib calls ...
qx.plotmotion(block)
```

## Shifting data with `qx.lag`

You want to compare today's price to where it was 5 days ago — computing a slope, or checking whether momentum is accelerating.

`qx.lag` shifts an array backward by N positions:

```python
qx.lag(prices, 5)   # drops last 5 elements, shifts everything earlier by 5
```

Returns the array unchanged if `amount` is 0. Under the hood it's a simple slice: `array[:-amount]`.

## Truncating arrays with `qx.truncate`

Indicators with different warmup lengths produce arrays of different sizes. Before you can compare or combine them, they need to align.

`qx.truncate` cuts multiple lists to the length of the shortest, keeping only the most recent elements:

```python
close = [1, 2, 3, 4, 5]
volume = [100, 200]
qx.truncate(close, volume)  # → ([4, 5], [100, 200])
```

## Numpy arrays in tune

Tune values don't have to be floats. Drop in a numpy array as a tune entry — the optimizer handles it:

```python
self.tune = {
    "weight_vector": np.array([0.3, 0.5, 0.2]),
}
```

Useful when your strategy depends on a distribution or ratio that needs to stay normalized.

## The `reset()` hook

Optimization runs hundreds of backtests in sequence. If your bot accumulates state across runs (counters, last signal, cached values), results from one run bleed into the next.

`reset()` is called between runs to clear that state:

```python
def reset(self):
    self.my_counter = 0
    self.last_signal = None
```

## The `execution()` hook

Say your strategy signals a market buy. But you want to place a limit order at yesterday's close instead — a more favorable entry if the price dips.

`execution()` runs after `strategy()` returns a signal but before the trade is placed. Modify the signal there:

```python
def execution(self, signal, indicators, wallet):
    if isinstance(signal, qx.Buy):
        signal.price = indicators["fast_ema"][-1]
    return signal
```

Must return a signal (or `None` to cancel the trade). The base class just returns the signal as-is.
