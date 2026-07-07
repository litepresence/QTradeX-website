# Custom Indicators & Decorators

You have an indicator idea that isn't in the 100+ Tulip indicators. Or someone posted a cool indicator on TradingView and you want to port it to QTradeX.

You have options.

## The `qi` indicators

A hundred indicators covers a lot of ground. It doesn't cover everything.

QTradeX ships extra compiled indicators under `qx.qi` (or `from qtradex import qi`). These are Cython-accelerated, take numpy arrays in, return numpy arrays out — same as the Tulip wrappers.

A few you'll reach for first:

```python
import qtradex as qx

# SuperTrend — trend following with ATR bands
st = qx.qi.supertrend(high, low, close, period=10, mult_top=3.0, mult_bottom=3.0)

# Ichimoku — the full cloud
tenkan, kijun, senkou_a, senkou_b, chikou = qx.qi.ichimoku(
    high, low, close, tenkan=9, kijun=26, senkou_b=52
)

# Heikin-Ashi candles — smoothed OHLC for trend clarity
ha_open, ha_high, ha_low, ha_close = qx.qi.heikin_ashi(data)

# Keltner Channels — volatility-based envelope
upper, middle, lower = qx.qi.keltner(
    high, low, close, atr_period=10, ma_period=20, ma_type=2, multiplier=1.5
)

# Donchian Channels — highest high / lowest low over N periods
upper, middle, lower = qx.qi.donchian(high, low, period=20)
```

Several of these accept a `ma_type` parameter — an integer that selects which moving average to use internally.

| Int | MA type |
|-----|---------|
| 1 | DEMA |
| 2 | EMA |
| 3 | HMA |
| 4 | KAMA |
| 5 | LinReg |
| 6 | SMA |
| 7 | TEMA |
| 8 | TRIMA |
| 9 | TSF |
| 10 | Wilders |
| 11 | WMA |
| 12 | ZLEMA |

Pass `ma_type=6` for SMA, `ma_type=2` for EMA, and so on.

The full list is in the [API reference](api/indicators.md#qi-custom-indicators). But the `qi` indicators still represent someone else's idea. What about yours?

## `@float_period` decorator

You'll probably write a custom indicator that takes a period parameter — lookback length, window size, that kind of thing. You'll want to optimize it.

Here's the problem. The optimizer treats every parameter as continuous. It tries 14.3, 14.7, 15.1, and expects each small change to tell it something about direction. But an integer period creates a staircase. Period 14 and period 15 are two different computations — there's nothing at 14.3 that tells the optimizer which way to go.

The `@float_period` decorator fixes that. It takes the zero-based index of each period argument and turns it into a smooth blend:

```python
from qtradex.indicators import float_period

@float_period(1)            # second argument is a float period
def my_indicator(close, lookback):
    ...
```

When you call `my_indicator(close, 14.3)`, the decorator calls the real function at `floor(14.3)` and `ceil(14.3)`, then blends the two results:

```python
result = 0.7 * my_indicator(close, 14) + 0.3 * my_indicator(close, 15)
```

The weight matches how close the float is to each integer. At 14.3, you get 70% of the period-14 result and 30% of the period-15 result. At 14.9, it's 10% and 90%.

The optimizer now sees a continuous slope. It can follow the gradient.

If your indicator takes multiple periods, pass multiple indices:

```python
@float_period(1, 2, 3)
def my_macd(close, short, long, signal):
    ...
```

All built-in `ti` and `qi` functions are already decorated. You only need this for your own functions that take period arguments.

## `@cache` decorator

During optimization, your indicator gets called thousands of times — once per backtest, thousands of backtests. If it's expensive, that's real wall time.

The `@cache` decorator keeps an LRU cache (maxsize 256) keyed by the md5 hash of the inputs. Same data in, cached result out.

```python
from qtradex.indicators.cache_decorator import cache

@cache
def expensive_indicator(close, period):
    # ... heavy computation ...
    return result
```

All `ti` and `qi` functions are pre-decorated. For your custom indicators, apply it yourself.

It composes with `@cache`. Put `@float_period` on top — it calls the cached function for each floor/ceil value, so the individual calls are cached independently:

```python
@float_period(1)
@cache
def my_indicator(close, lookback):
    ...
```

## `derivative()` and `lag()`

Two small utilities that show up in custom indicator code often enough to have their own import.

```python
from qtradex.indicators import derivative, lag
```

`derivative(array, period=1)` is the discrete first difference. `period=1` is `np.diff`. With `period>1` it computes `array[i + period] - array[i]`. Result is `n - period` elements.

`lag(array, amount)` shifts an array backward by `amount` positions, dropping trailing elements. Returns unchanged if `amount == 0`.

These don't do anything you couldn't write yourself. They save you the boilerplate when composing operations.

## Complete example

Rolling correlation between BTC and ETH. When the correlation drops below a threshold, the assumption is the pair will revert — a mean-reversion trade on the spread.

Start with the indicator:

```python
import numpy as np
from qtradex.indicators import float_period, cache

@cache
@float_period(2)
def rolling_correlation(a_close, b_close, lookback):
    n = min(len(a_close), len(b_close))
    result = np.full(n, np.nan)
    for i in range(lookback, n):
        result[i] = np.corrcoef(
            a_close[i - lookback : i],
            b_close[i - lookback : i],
        )[0, 1]
    return result
```

This is deliberately unoptimized — a loop calling `np.corrcoef` on every candle. That's fine. The `@cache` decorator ensures identical `(a_close, b_close, lookback)` tuples never re-compute.

Now use it in a bot. Load BTC as the main data. The ETH data lives at module level so it's fetched once, not on every bot instantiation:

```python
import qtradex as qx

eth_data = qx.Data(
    exchange="kucoin", asset="ETH",
    currency="USDT", begin="2020-01-01", end="2023-01-01",
)

class CorrelatedPairsBot(qx.BaseBot):
    def __init__(self):
        self.tune = {
            "corr_lookback": 20.0,
            "corr_entry": 0.8,
        }
        self.clamps = {
            "corr_lookback": [5, 52.5, 100, 1],
            "corr_entry": [0.5, 0.75, 1.0, 1],
        }

    def indicators(self, data):
        return {
            "corr": rolling_correlation(
                data["close"],
                eth_data["close"],
                self.tune["corr_lookback"],
            ),
        }

    def strategy(self, state, ind):
        if ind["corr"] < self.tune["corr_entry"]:
            return qx.Buy(reason="correlation breakdown")
        return qx.Hold()
```

Run it:

```python
data = qx.Data(
    exchange="kucoin", asset="BTC",
    currency="USDT", begin="2020-01-01", end="2023-01-01",
)
bot = CorrelatedPairsBot()
qx.dispatch(bot, data)
```

Select "Backtest" to see where correlation breakdowns occurred. Then select "Optimize" — the `@float_period` decorator on `corr_lookback` means the optimizer can find the exact lookback that triggers at the right moments, not just one of the integer steps.

The `@cache` decorator means every backtest that happens to use the same lookback (during QPSO's exploration, for example) reuses the computed result. The first backtest computes it; the next hundred with the same lookback read from cache.

## Next steps

- [Full indicator API reference](api/indicators.md) — `ti`, `qi`, every decorator, every utility
- [Getting Started: step 3 — periods are days, not candles](getting-started.md#step-3-periods-are-days-not-candles)
