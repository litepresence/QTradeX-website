# Data

## `qx.Data`

The primary class for fetching, caching, and accessing historical candle data.

```python
Data(
    exchange: str,
    asset: str,
    currency: str,
    begin: Union[str, int, float],
    end: Optional[Union[str, int, float]] = None,
    days: Optional[Union[int, float]] = None,
    candle_size: int = 86400,
    pool: Optional[str] = None,
    api_key: Optional[str] = None,
    intermediary: Optional[str] = None,
    placeholder: bool = False,
)
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `exchange` | `str` | required | Exchange identifier. Any CCXT exchange ID (`"binance"`, `"kucoin"`, `"kraken"`, etc.) or a special source: `"bitshares"`, `"cryptocompare"`, `"alphavantage stocks"`, `"alphavantage forex"`, `"alphavantage crypto"`, `"synthetic"`, `"yahoo"`, `"finance data reader"`. |
| `asset` | `str` | required | Base asset symbol (e.g. `"BTC"`). |
| `currency` | `str` | required | Quote currency symbol (e.g. `"USDT"`). |
| `begin` | `str`, `int`, or `float` | required | Start date. `"YYYY-MM-DD"` string or Unix timestamp in seconds. |
| `end` | `str`, `int`, or `float` | `None` | End date. Same format as `begin`. Defaults to current time if neither `end` nor `days` is set. Mutually exclusive with `days`. |
| `days` | `int` or `float` | `None` | Duration in days before `begin`. Alternative to `end`. Mutually exclusive with `end`. |
| `candle_size` | `int` | `86400` | Candle width in seconds. Common values: `60` (1m), `300` (5m), `900` (15m), `3600` (1h), `14400` (4h), `86400` (1d). |
| `pool` | `Optional[str]` | `None` | Liquidity pool identifier (BitShares DEX only). Raises `ValueError` if set for non-bitshares exchanges. |
| `api_key` | `Optional[str]` | `None` | API key for sources that require one (Alpha Vantage, CryptoCompare). |
| `intermediary` | `Optional[str]` | `None` | Third asset for implied price synthesis. E.g., `intermediary="BTC"` to get `XRP/USD` via `XRP/BTC` + `BTC/USD`. |
| `placeholder` | `bool` | `False` | If `True`, skips data fetching entirely. Used internally by `load_csv`. |

On construction, `Data` fetches and caches the requested candles immediately. Subsequent constructions with the same parameters read from disk cache — instant.

### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `exchange` | `str` | Exchange name as passed |
| `asset` | `str` | Base asset |
| `currency` | `str` | Quote currency |
| `begin` | `int` | Actual start Unix timestamp (quantized to `candle_size`) |
| `end` | `int` | Actual end Unix timestamp |
| `days` | `float` | Duration in days |
| `candle_size` | `int` | Candle size in seconds |
| `base_size` | `int` | Original candle size before composite fetch |
| `pool` | `Optional[str]` | Pool identifier |
| `intermediary` | `Optional[str]` | Intermediary asset |
| `raw_candles` | `dict` | Core candle data — keys `"unix"`, `"open"`, `"high"`, `"low"`, `"close"`, `"volume"`, each a numpy array |
| `fine_data` | `Optional[dict]` | Fine-grained candle data for papertrade/live, same structure as `raw_candles` |

### Methods

**`data[key]`** — Index or slice. Integer index returns a single candle's columns. Slice returns a new `Data` object with the same metadata but a subset of candles.

**`len(data)`** — Number of candles (`len(data.raw_candles["close"])`).

**`data.keys()`**, **`data.values()`**, **`data.items()`** — Delegate to `raw_candles`.

**`data.update_candles(begin, end)`** — Re-fetches data for a new time range. Used internally by papertrade and live modes.

## `qx.load_csv`

Load candle data from a CSV file into a `Data` object.

```python
load_csv(
    exchange: str,
    asset: str,
    currency: str,
    filepath: str,
    begin: Optional[str] = None,
    end: Optional[str] = None,
    candle_size: int = 86400,
    stride: Optional[int] = None,
) -> Data
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `exchange` | `str` | required | Exchange label (metadata, stored in the returned `Data` object) |
| `asset` | `str` | required | Base asset symbol |
| `currency` | `str` | required | Quote currency symbol |
| `filepath` | `str` | required | Path to the CSV file |
| `begin` | `Optional[str]` | `None` | Start date. Inclusive. `"YYYY-MM-DD"` or Unix timestamp. |
| `end` | `Optional[str]` | `None` | End date. Inclusive. |
| `candle_size` | `int` | `86400` | Target candle size in seconds |
| `stride` | `Optional[int]` | `None` | Stride between candles. Defaults to `candle_size` (non-overlapping). |

**CSV format.** Accepts columns `unix, open, high, low, close, volume` for explicit OHLCV, or `unix, price, volume` for discrete tick data (auto-generates OHLCV). Supports `unix_milli` and `unix_micro` timestamp columns.

**Returns.** A `Data` object with `raw_candles` at the requested `candle_size` and `fine_data` set to the raw stride-resolution data. Uses the same disk cache as `Data`.

## Utility functions

These operate on candle dicts (dicts with numpy array keys `"unix"`, `"open"`, `"high"`, `"low"`, `"close"`, `"volume"`).

### `clip_to_time_range(candles, start_unix, end_unix)`

Returns a new candle dict with only candles whose `"unix"` falls within `[start_unix, end_unix]`.

### `invert(candles)`

Inverts a pair (e.g., `BTC/USDT` to `USDT/BTC`). Prices become `1/price`, high/low are swapped, volume adjusts. Accepts both list-of-dicts and dict-of-arrays formats.

### `implied(candles1, candles2)`

Synthesizes an implied price pair from two datasets. Given `XRP/BTC` and `BTC/USDT`, produces `XRP/USDT`. Truncates to the shorter length.

### `interpolate(data, oldperiod, newperiod)`

Cubic-spline interpolation of OHLC data between candle sizes. Volume uses nearest-neighbor scaled by `newperiod / oldperiod`.

### `reaggregate(data, candle_size, stride=None)`

Disaggregates OHLCV candles into discrete price points, then re-aggregates at a new `candle_size` and optional `stride`. Useful for changing candle granularity while preserving OHLC shape.

### `merge_candles(candles, candle_size)`

Combines multiple candle datasets (list of dicts) into one at a common `candle_size`. Quantizes unix times, then merges by taking max(high), min(low), first(open), last(close), max(volume).

### `create_candles(data, width=86400, stride=600)`

Creates OHLCV candles from a list of `(unix, price, volume)` tuples. Uses a sliding window of `width` seconds with `stride` spacing.

### `quantize_unix(unix_array, candle_size)`

Floors timestamps to the nearest candle boundary. Used internally by `merge_candles()` and `Data.__init__()`.

```python
quantize_unix(np.array([100, 200, 300]), 100)  # -> [100, 200, 300]
quantize_unix(np.array([150, 250, 350]), 100)  # -> [100, 200, 300]
```

### `synthesize_high_low(d1_h, d2_h, d1_l, d2_l, d1_o, d2_o, d1_c, d2_c)`

Computes high/low values for implied synthetic pairs. Used by `implied()` to produce realistic candle ranges when combining two price series.

### `BadTimeframeError`

Raised when a requested `candle_size` is not available from the data source.

### `fetch_composite_data(data, new_size)`

Fetches high-resolution candle data for papertrade and live modes. Tries progressively smaller candle sizes `[60, 600, 3600, ...]` until data is found, then re-aggregates to `data.candle_size` with `stride=new_size`. Sets `data.raw_candles` (wide) and `data.fine_data` (fine). Modifies the `Data` object in place and returns it.

## Supported data sources

| Source | `exchange` value | Auth |
|--------|-----------------|------|
| Any CCXT exchange | Exchange ID (e.g. `"binance"`) | None for public candles |
| BitShares DEX | `"bitshares"` | None |
| CryptoCompare | `"cryptocompare"` | API key |
| Alpha Vantage (stocks) | `"alphavantage stocks"` | API key |
| Alpha Vantage (forex) | `"alphavantage forex"` | API key |
| Alpha Vantage (crypto) | `"alphavantage crypto"` | API key |
| Yahoo Finance | `"yahoo"` | None |
| FinanceDataReader | `"finance data reader"` | None |
| Synthetic (test data) | `"synthetic"` | None |

No authentication is needed for public candle data from CCXT exchanges, Yahoo Finance, or synthetic data. Alpha Vantage and CryptoCompare require free API keys passed via the `api_key` parameter.

## Synthetic data

When `exchange="synthetic"`, `Data` uses a harmonic Brownian walk to generate random price data. No API keys, no network, no disk caching — useful for testing strategies, prototyping, and CI.

```python
data = qx.Data("synthetic", "BTC", "USD", days=365, candle_size=86400)
```

The underlying generator is available directly for custom use:

```python
from qtradex.public.klines_synthetic import (
    klines_synthetic, create_dataset, hlocv_data, synthesize
)
```

### Quick start

Three levels of access:

```python
# Full OHLCV as a dict of numpy arrays (direct to Data)
ohlcv = klines_synthetic()
# -> {"unix": ndarray, "open": ndarray, "high": ndarray,
#     "low": ndarray, "close": ndarray, "volume": ndarray}

# Close prices only
dataset = create_dataset()
# -> {"unix": [...], "close": [...]}

# One step at a time
storage = {"log_periodic": 0.00001}
price = synthesize(storage, tick=1)
```

### How it works

The generator composes three layers of randomness:

**Harmonic cycles.** Seven sine waves at different frequencies are summed into a `sine` accumulator. Each harmonic's amplitude decays as `ACCEL / harmonic`, so the first few harmonics dominate the shape. The result feeds into a log-periodic recurrence:

```python
storage["log_periodic"] *= pow(1 + sum_of_sines, tick)
```

**Random walk.** Each step is perturbed by `((1 - STEP) + 2 * STEP * rand)`, scaling the previous value up or down by up to `STEP` (default 7%). This produces the stochastic drift that makes the series look like real price action.

**OHLCV synthesis.** `hlocv_data()` takes the close-price series and expands it to full candles. Each candle's open is the prior candle's close, perturbed by `VOLATILITY` (default 2%). High/low are the open/close range extended by a random fraction of the spread. Volume is `(high - low) / close * 10^VOLUME_SIZE`.

### Constants

| Constant | Default | Description |
|----------|---------|-------------|
| `HARMONICS` | 7 | Number of sine waves summed to form cyclic structure |
| `ACCEL` | `1e-6` | Harmonic amplitude decay factor |
| `STEP` | 0.07 | Random walk step size (fraction of current price) |
| `FREQ` | `2e-4` | Base frequency of the harmonic oscillations |
| `VOLATILITY` | 2.0 | HLOC spread variance (percent) |
| `VOLUME_SIZE` | 5.0 | Volume scale exponent |
| `START` | `1e-5` | Initial price |
| `DEPTH` | 1000 | Number of candles in a generated dataset |

### `synthesize(storage, tick)`

Core step function. Advances the state machine by one tick and returns the next closing price.

```python
storage = {"log_periodic": 0.00001}  # initial state
for t in range(1, 1001):
    price = synthesize(storage, t)
```

The `storage` dict carries state between steps — you can seed it, inspect it, or run it in reverse for debugging.

### `create_dataset()`

Runs `synthesize()` through `DEPTH + 1` ticks, returning `unix` and `close` lists. Timestamps start at `now - (DEPTH + 1) days` with ~1-day spacing (86400 + 200 seconds per candle).

```python
raw = create_dataset()
len(raw["unix"])    # 1001
len(raw["close"])   # 1001
```

### `hlocv_data(data)`

Takes a `{"unix": [...], "close": [...]}` dict and expands it into full OHLCV with numpy arrays. Drops the first and last candle to maintain structural consistency.

```python
candles = hlocv_data(create_dataset())
# -> keys: unix, open, high, low, close, volume (all ndarray)
```

### `klines_synthetic()`

Convenience wrapper — `hlocv_data(create_dataset())` in one call. Returns what `Data` uses internally.

### Characteristics

- **No determinism.** The generator uses `random.random()` without a fixed seed. Every run produces different data.
- **Price scale.** Starting at `0.00001`, the log-periodic recurrence grows the price over time. After 1000 steps prices typically range in single or low double digits.
- **Appearance.** Broad trend cycles (~2-3 per 1000 candles) with noise on every tick. The harmonic component creates visible wave patterns that a strategy can exploit — a good test for overfitting detection.
