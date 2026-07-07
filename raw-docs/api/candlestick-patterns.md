# Candlestick Pattern Recognition

> **Under construction.** This module was ported from TA-Lib's candlestick pattern recognition engine. It has not been fully tested against the original, and several patterns are known to be incomplete. Do not use in production without independent verification.

```python
from qtradex.indicators.candle_class import (
    SimpleSeries, EnhancedSeries, Series,
    CandleSetting, RangeType, CandleColor,
    two_crows, three_black_crows, three_inside,
    three_line_strike, three_outside, three_stars_in_south,
    three_white_soldiers, abandoned_baby, advance_block,
    belt_hold, break_away, closing_marubozu,
    conceal_baby_swallow, doji, doji_star,
    evening_star, matching_low, piercing, stick_sandwich,
)
```

Converts OHLCV candle data into bullish/bearish pattern signals. Each function takes a `Series` object and returns a list of integers (`-100`, `0`, or `100`) — one value per candle, where non-zero indicates a pattern match.

---

## Data model

### `Series` (protocol)

The interface your candle data must satisfy:

```python
class Series(Protocol):
    def len(self) -> int: ...
    def high(self, i: int) -> float: ...
    def open(self, i: int) -> float: ...
    def close(self, i: int) -> float: ...
    def low(self, i: int) -> float: ...
```

### `SimpleSeries`

Wraps raw OHLCV lists into a `Series`:

```python
series = SimpleSeries(highs, opens, closes, lows, volumes, rands)
```

### `EnhancedSeries`

Wraps a `Series` with helper methods used internally by every pattern function: `candle_color()`, `real_body()`, `upper_shadow()`, `lower_shadow()`, `average()`, `range_of()`, `is_candle_gap_up()`, `is_candle_gap_down()`, `real_body_gap_up()`, `real_body_gap_down()`, `high_low_range()`.

### `CandleColor`

```python
CandleColor.WHITE  # close >= open  (bullish, value 1)
CandleColor.BLACK  # close < open   (bearish, value -1)
```

### `RangeType`

```python
RangeType.REAL_BODY  # |close - open|
RangeType.HIGH_LOW   # high - low
RangeType.SHADOWS    # upper or lower shadow
```

### `CandleSetting`

```python
CandleSetting(range_type: RangeType, avg_period: int, factor: float)
```

Pre-defined settings used by pattern functions:

| Setting | range_type | avg_period | factor |
|---------|------------|------------|--------|
| `setting_body_long` | REAL_BODY | 10 | 1.0 |
| `setting_body_very_long` | REAL_BODY | 10 | 3.0 |
| `setting_body_short` | REAL_BODY | 10 | 1.0 |
| `setting_body_doji` | HIGH_LOW | 10 | 0.1 |
| `setting_shadow_long` | REAL_BODY | 0 | 1.0 |
| `setting_shadow_very_long` | REAL_BODY | 0 | 2.0 |
| `setting_shadow_short` | SHADOWS | 10 | 1.0 |
| `setting_shadow_very_short` | HIGH_LOW | 10 | 0.1 |
| `setting_near` | HIGH_LOW | 5 | 0.2 |
| `setting_far` | HIGH_LOW | 5 | 0.6 |
| `setting_equal` | HIGH_LOW | 5 | 0.05 |

---

## Pattern functions

All functions share the same signature and return convention:

```python
pattern_fn(series: Series) -> List[int]
```

Returns `[-100, 0, 100]` per candle. `-100` = bearish pattern, `100` = bullish pattern, `0` = no match.

### Two Crows

Bearish reversal. Long white candle, followed by a black candle that gaps up, followed by another black candle that opens inside the second and closes inside the first.

### Three Black Crows

Bearish reversal. Three consecutive long black candles with very short lower shadows, each opening within the previous body, closing progressively lower.

### Three Inside

Bullish/bearish reversal. Long candle of one color, followed by a short candle engulfed by it, then a third candle closing beyond the first's open in the opposite direction.

### Three Line Strike

Bullish/bearish continuation. Three candles of the same color with higher/lower closes, then a fourth candle in the opposite direction that closes beyond the first.

### Three Outside

Bullish/bearish reversal. A two-candle engulfing pattern confirmed by a third candle in the same direction as the second.

### Three Stars in the South

Bullish reversal. Three black candles: first has a long lower shadow, second is smaller but still has a lower shadow, third is a small marubozu within the second's range.

### Three White Soldiers

Bullish reversal. Three white candles with very short upper shadows, each closing higher and opening near the previous body.

### Abandoned Baby

Bullish/bearish reversal. Long candle, a doji that gaps away, then a third candle that gaps back and closes well within the first body. `penetration` parameter controls the close-depth threshold (default `0.3`, meaning 30% into the first body).

### Advance Block

Bearish reversal. Three white soldiers with signs of weakening: shrinking bodies, long upper shadows, or a doji-like third candle.

### Belt Hold

Bullish (white) or bearish (black) single-candle pattern. A long candle with a very short shadow on one side — a white belt hold opens at the low and closes near the high; a black belt hold opens at the high and closes near the low.

### Break Away

Bullish/bearish reversal over five candles. A long candle, then three candles moving in the same direction (gapping and trending), then a fifth candle that closes inside the gap between the first and second.

### Closing Marubozu

Single-candle pattern. A long candle with a very short shadow on the closing side — white has no upper shadow, black has no lower shadow.

### Conceal Baby Swallow

Bullish reversal. Four black candles: two marubozus, then a third that gaps down with a long upper shadow, then a fourth that engulfs the third completely.

### Doji

Single-candle pattern. A candle whose real body is very small compared to the high-low range (controlled by `setting_body_doji`).

### Doji Star

Bullish/bearish reversal. Long candle of one color followed by a doji that gaps away from it.

### Evening Star

Bearish reversal. Long white candle, a short candle that gaps up, then a black candle closing well into the first body. `penetration` parameter (default `0.3`) controls how far into the first body the third candle must close.

### Matching Low

Bullish continuation. Two black candles with the same closing price.

### Piercing

Bullish reversal. Long black candle followed by a white candle that opens below the prior low and closes above the midpoint of the black body.

### Stick Sandwich

Bullish continuation. Three candles: black, white (trading only above the first close), black (closing at the same level as the first).

---

## Usage

```python
import numpy as np
from qtradex.indicators.candle_class import (
    SimpleSeries, doji, piercing, evening_star
)

# Build a series from your candle data
series = SimpleSeries(
    highs=[1.5, 1.6, 1.4, 1.3],
    opens=[1.0, 1.5, 1.3, 1.2],
    closes=[1.5, 1.3, 1.2, 1.1],
    lows=[1.0, 1.3, 1.1, 1.0],
    volumes=[100, 150, 120, 130],
    rands=[0.0, 0.0, 0.0, 0.0],
)

doji_signals = doji(series)           # [0, 0, 0, 100]
piercing_signals = piercing(series)   # [0, 0, 0, 0]
```

Patterns are also available as direct imports for one-off use:

```python
from qtradex.indicators.candle_class import three_white_soldiers
```

---

## Caveats

- These functions are **not** part of the public `qx.*` namespace. Import directly from `qtradex.indicators.candle_class`.
- The test file (`candle_class_tests.py`) requires `talib` — which is **not** a project dependency — and will fail unless installed separately.
- The `rand` parameter in `SimpleSeries` is vestigial (inherited from the TA-Lib port) and has no effect on pattern detection.
