# Candlestick Patterns in Your Strategy

> **Under construction.** This module was ported from TA-Lib's candlestick pattern recognition engine. It has not been fully tested against the original, and several patterns are known to be incomplete. Do not use in production without independent verification.

You can spot a doji star or engulfing pattern on the chart. You want your bot to see them too.

## Building a `Series`

Candlestick pattern detection needs all four OHLC values plus volume — not just close like most indicators. QTradeX wraps them into a `SimpleSeries` object:

```python
from qtradex.indicators.candle_class import SimpleSeries

series = SimpleSeries(
    high=data["raw_candles"]["high"],
    open=data["raw_candles"]["open"],
    close=data["raw_candles"]["close"],
    low=data["raw_candles"]["low"],
    volumes=data["raw_candles"]["volume"],
    rands=None,
)
```

The `rands` parameter is vestigial — a leftover from the TA-Lib port. It does nothing. Pass `None`.

## Single-pattern strategy

Pattern functions return `[-100, 0, 100]` where `-100` is bearish, `100` is bullish, and `0` means no match.

A doji star formation after an uptrend suggests a reversal. Check the return value and sell:

```python
def strategy(self, state, indicators):
    series = SimpleSeries(
        high=data["raw_candles"]["high"],
        open=data["raw_candles"]["open"],
        close=data["raw_candles"]["close"],
        low=data["raw_candles"]["low"],
        volumes=data["raw_candles"]["volume"],
        rands=None,
    )
    doji = doji_star(series)

    if doji == 100:
        return qx.Sell(reason="bearish doji star")
    elif evening_star(series) == 100:
        return qx.Sell(reason="evening star")

    return qx.Hold()
```

`doji_star()` returning `100` means a bearish doji star pattern. `evening_star()` returning `100` is the same — the convention is always `-100` for bearish, `100` for bullish, regardless of pattern name.

## Multi-pattern confirmation

One pattern is a hint. Two patterns on the same candle are harder to ignore.

Combine signals with boolean logic to require confirmation:

```python
def strategy(self, state, indicators):
    series = SimpleSeries(...)

    piercing = piercing_pattern(series) == -100   # bullish reversal
    soldiers = three_white_soldiers(series) == -100

    if piercing and soldiers:
        return qx.Buy(reason="piercing + three white soldiers confirmed")

    return qx.Hold()
```

Both must fire on the same candle before the bot buys. This filters out weak signals that a single pattern would trigger.

## Combining with indicators

Don't trust a doji star alone. Confirm with RSI divergence or a volume spike.

Mix pattern signals with `qx.ti` indicators:

```python
def strategy(self, state, indicators):
    series = SimpleSeries(...)
    rsi = qx.ti.rsi(data["close"], 14)

    if doji_star(series) == 100 and rsi[-1] > 70:
        return qx.Sell(reason="doji star + overbought RSI")

    return qx.Hold()
```

The doji star suggests a reversal. The RSI above 70 confirms the market was overbought. Either alone is weaker than both together.

## Caveats

Candlestick pattern detection is **under construction**. Here's what you need to know before using it in production:

- **Not all patterns are complete.** The TA-Lib port covers the most common patterns. Some exotic patterns are missing.
- **Import path.** Pattern functions are **not** in the public `qx.*` namespace. Import from `qtradex.indicators.candle_class`:

```python
from qtradex.indicators.candle_class import (
    doji_star,
    evening_star,
    piercing_pattern,
    three_white_soldiers,
)
```

- **TA-Lib requirement.** The test suite depends on `talib` which is **not** a QTradeX dependency. Numerical results match TA-Lib on the tests that exist, but there is no CI gate.
- **Verify independently.** Test each pattern against known chart examples before trusting it in a live strategy. The functions produce plausible-looking results — that does not mean they are correct for every market condition.

## Next steps

- [Candlestick patterns API reference](api/candlestick-patterns.md) — full list of available patterns and their signatures
- [Custom indicators](custom-indicators.md) — writing your own pattern detectors when the built-in list isn't enough
