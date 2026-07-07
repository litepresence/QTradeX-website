# Indicators

```python
from qtradex import ti, qi
from qtradex.indicators import derivative, lag
```

## `ti` — Tulipy wrapper

`ti` wraps the [tulipy](https://github.com/cirla/tulipy) library. ~100 technical indicators, all with `@cache` (LRU, maxsize 256). Period parameters accept floats — the engine interpolates between `floor` and `ceil` values.

See the [tulipy indicator reference](https://tulipindicators.org/usage) for per-indicator parameter details and expected array inputs.

```python
ti.sma(close, period=14)         # Simple moving average
ti.ema(close, period=30)         # Exponential moving average
ti.rsi(close, period=14)         # Relative Strength Index
ti.macd(close, 12, 26, 9)        # MACD (returns macd, signal, histogram)
ti.bbands(close, 20, 2)          # Bollinger Bands (upper, middle, lower)
```

### Calling convention

```python
result = ti.<function>(*arrays, *periods, *options)
```

Arguments pass through to `tulipy` directly. Arrays are positional, followed by period integers/floats, then additional options — depends on the specific indicator.

Some functions return a single array, others return a tuple (e.g. `macd` returns 3, `bbands` returns 3, `stoch` returns 2).

### Indicators by category

**Moving averages:** `sma`, `ema`, `dema`, `tema`, `trima`, `wma`, `hma`, `kama`, `zlema`, `vidya`, `vwma`, `linreg`, `tsf`

**Oscillators:** `rsi`, `stoch`, `stochrsi`, `cci`, `cmo`, `apo`, `ppo`, `mom`, `macd`, `trix`, `willr`, `ultosc`, `fisher`, `mfi`

**Volatility:** `bbands`, `atr`, `natr`, `volatility`, `mass`

**Volume:** `obv`, `ad`, `adosc`, `kvo`, `vosc`, `mfi`, `nvi`, `pvi`, `emv`

**Price transforms:** `avgprice`, `medprice`, `typprice`, `wcprice`, `tr`

**Math ops (on arrays):** `add`, `sub`, `mul`, `div`, `abs`, `sqrt`, `ln`, `log10`, `exp`, `sin`, `cos`, `tan`, `ceil`, `floor`, `round`, `min`, `max`, `sum`

**Other:** `adx`, `adxr`, `aroon`, `aroonosc`, `bop`, `cvi`, `di`, `dm`, `dpo`, `dx`, `fosc`, `kvo`, `lag`, `msw`, `psar`, `qstick`, `roc`, `rocr`, `stddev`, `stderr`, `var`, `vhf`, `wad`, `wilders`, `decay`, `edecay`, `crossover`, `crossany`

## `@float_period` decorator

```python
from qtradex.indicators import float_period
```

A decorator factory that lets indicator functions accept float period values. The engine calls the function at `floor(arg)` and `ceil(arg)`, then blends results with a weighted average. See [Getting Started](../getting-started.md#step-3-periods-are-days-not-candles) for the rationale.

### Usage

Pass zero-based positional indices for each float-eligible parameter:

```python
@float_period(1)            # second argument is a float period
def my_indicator(close, lookback):
    ...

@float_period(1, 2, 3)     # second, third, fourth arguments
def my_macd(close, short, long, signal):
    ...
```

### Behavior

For each float period index, the decorator computes `floor(arg)` and `ceil(arg)`, calls the function twice (all-floored, all-ceiled), and returns the element-wise weighted sum:

```python
result = floor_weight * func(*floored_args) + ceil_weight * func(*ceiled_args)
```

where `floor_weight = ceil(arg) - arg` and `ceil_weight = arg - floor(arg)`.

### Which indicators have it

All `ti` functions with period parameters are pre-decorated. Most `qi` functions are too — except those that take deviation/brick percentages instead of periods (`zigzag`, `kagi`, `renko`, `tick_indicator`, `trin_indicator`, `vhf`).

### Cache interaction

Composes with `@cache` (LRU, maxsize 256). The two calls (floor/ceil) are cached independently — calling `ti.sma(close, 7)` after `ti.sma(close, 7.3)` is a cache hit.

## `qi` — Custom indicators

Cython-compiled indicators unique to QTradeX. All accept and return `numpy.ndarray` of `float64`.

```python
qi.heikin_ashi(data)       # Heikin-Ashi candles
qi.ichimoku(high, low, close, tenkan, kijun, senkou_b, span)
qi.vortex(high, low, close, window)    # Vortex indicator
qi.kst(close, roc1, roc2, roc3, roc4, smoothing)  # Know Sure Thing
qi.frama(close, period, fractal_period) # Fractal Adaptive MA
qi.zigzag(close, deviation)             # Zig Zag
qi.ravi(high, low, close, short, long)  # RAVI
qi.aema(close, period, alpha=0.1)       # Adaptive EMA
qi.typed_macd(close, short, long, signal, ma_type)  # MACD with selectable MA
qi.typed_bbands(close, ma_period, ma_type, std_period, deviations)
qi.tsi(close, long_period, short_period)  # True Strength Index
qi.smi(close, high, low, k_period, d_period)  # Stochastic Momentum Index
qi.eri(high, low, close, ma_period, ma_type)  # Elder Ray Index
qi.supertrend(high, low, close, period, mult_top, mult_bottom)
qi.arsi(close, length=14)              # Adaptive RSI
qi.keltner(high, low, close, atr_period, ma_period, ma_type, multiplier=1.5)
qi.donchian(high, low, period)
qi.kagi(close, reversal_percent)
qi.renko(close, brick_percent)
qi.tick_indicator(close)
qi.trin_indicator(close, volume)
qi.market_profile(close, volume, bin_size)
qi.price_action(close, lookback, threshold=0.01)
qi.holt_winters_des(x, span, beta)
qi.ulcer_index(close, window)
qi.trix(close, window)                  # (different from ti.trix — triple EMA)
qi.earsi(close, auto_min, auto_max, auto_avg)  # Ehlers Adaptive RSI
qi.vhf(data, period)                    # Vertical Horizontal Filter
```

### `ma_type` values

Functions that accept `ma_type` use this lookup:

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

## `@cache` decorator

```python
from qtradex.indicators.cache_decorator import cache
```

LRU cache (maxsize 256) for functions that take numpy arrays. Hashes arrays via md5 so identical inputs return cached results. All `ti` and `qi` functions are pre-decorated. Use it on your own custom indicators when they are called repeatedly with the same data:

```python
@cache
def my_indicator(close, period):
    # expensive computation
    return result
```

The decorator composes with `@float_period` — apply `@cache` on top.

## `derivative()`

```python
derivative(array, period=1) -> ndarray
```

Discrete first difference. `period=1` uses `np.diff`. `period>1` computes `array[i + period] - array[i]`. Result length is `n - period`.

## `lag()`

```python
lag(array, amount) -> ndarray
```

Shifts an array backward by `amount` (drops trailing elements). Returns unchanged if `amount == 0`.

## `fitness()`

```python
fitness(keys, states, raw_states, asset, currency) -> dict
```

Orchestrates metric computation. `keys` is a list of metric name strings. Returns `{metric_name: value}`.

Available metric keys: `roi`, `cagr`, `sharpe_ratio`, `sortino_ratio`, `maximum_drawdown`, `calmar_ratio`, `omega_ratio`, `beta`, `alpha`, `info_ratio`, `profit_factor`, `trade_win_rate`, `payoff_ratio`, `skewness`, `kurtosis`, `efficiency_ratio`, `drawdown_duration`, `hurst_exponent`, `dpt` (days per trade with target), `percent_cheats`, `ind:name:low~high` (indicator-based constraint).
