# Data Deep Dive

You have data from somewhere else — a CSV export, a custom API, a different exchange. Or you want XRP/USD but your exchange only has XRP/BTC and BTC/USDT. Or you want to test a strategy without any API keys at all.

These are data problems. QTradeX has tools for each of them.

## Loading your own CSV files

`qx.Data()` handles fetching from exchanges. But your data might not come from an exchange at all — maybe it's a CSV you exported from another platform, collected from a sensor, or scraped from a website.

Pass the file to `qx.load_csv`:

```python
data = qx.load_csv(
    exchange="my_source",
    asset="BTC",
    currency="USD",
    filepath="btc_usd_2024.csv",
)
```

The CSV needs columns named `unix`, `open`, `high`, `low`, `close`, `volume`. Timestamps in seconds are fine. You can also use `unix_milli` or `unix_micro` for millisecond or microsecond precision.

For tick data — a log of every trade, not aggregated candles — use `unix, price, volume` columns instead. QTradeX auto-generates OHLCV candles from ticks.

The result is a `Data` object, same as if you'd fetched from an exchange. It uses the same disk cache, so loading the same file twice is instant.

### Stride

Candles usually tile end-to-end: one closes, the next opens. But you can set a gap between them with `stride`:

```python
data = qx.load_csv(..., stride=7200)  # 2-hour stride
```

Each candle still spans `candle_size` seconds, but starts `stride` seconds after the previous one. Useful when your raw data has irregular spacing and you want overlapping or gapped windows.

## Synthetic data

You don't have data yet — you're prototyping, writing tests, or running CI. You need something that looks like real prices but doesn't require API keys, network access, or disk space.

Set `exchange="synthetic"`:

```python
data = qx.Data("synthetic", "BTC", "USD", days=365, candle_size=86400)
```

This generates a harmonic Brownian walk — a random price series with trends, noise, and candle structure. Each run produces different data. No two look alike.

### What's under the hood

The generator is available directly for custom use:

```python
from qtradex.public.klines_synthetic import klines_synthetic
```

It layers three mechanisms:

1. **Harmonic cycles.** Seven sine waves at different frequencies are summed. The first few harmonics dominate, creating visible wave patterns — about 2--3 full cycles per 1000 candles. This gives the data structure that a strategy can latch onto (which also makes it useful for testing overfitting).

2. **Random walk.** Each step multiplies the previous price by a random factor between `(1 - STEP)` and `(1 + STEP)` — default 7% max perturbation per step. This produces the stochastic drift that makes prices look real.

3. **OHLCV expansion.** The close-price series gets expanded into full candles. Each candle's open is the previous close, perturbed by `VOLATILITY`. High/low extend the open-close range by a random fraction of the spread. Volume scales with the price range.

You can tweak the behavior through module constants:

```python
import qtradex.public.klines_synthetic as ks
ks.STEP = 0.03       # gentler random walk
ks.VOLATILITY = 1.0  # tighter candle ranges
ks.HARMONICS = 4     # fewer trend cycles
```

Where does synthetic data make sense? Testing indicator logic, verifying your strategy doesn't crash on edge cases, running in CI where every dependency must be local. It does not replace real market data — the harmonic cycles are too clean, the noise too uniform. But it's better than testing against nothing.

## Implied pairs with `intermediary`

You want XRP/USD. Your exchange has XRP/BTC and BTC/USDT. You could fetch both separately and merge them yourself — but that's tedious and easy to get wrong.

Pass `intermediary` to `Data`:

```python
data = qx.Data(
    exchange="kucoin",
    asset="XRP",
    currency="USD",
    intermediary="BTC",
    begin="2024-01-01",
    end="2024-06-01",
)
```

QTradeX fetches both pairs — XRP/BTC and BTC/USD — and synthesizes the implied cross rate. The result is a single `Data` object that looks exactly like native XRP/USD candles.

### How it works

Under the hood, `implied()` computes the cross rate at each timestamp. For close prices it's simple: multiply the two series together. High and low are trickier — you can't just multiply highs and expect a realistic range.

`synthesize_high_low()` handles that. It computes four candidate values from the high/low pairs of both legs, then picks the realistic extremes. The result is a candle range that reflects what XRP/USD *would* have traded at, given the constituent markets.

You can call these functions directly:

```python
from qtradex.public.data_utils import implied, synthesize_high_low

xrp_usd = implied(xrp_btc, btc_usd)
```

But in practice you'll use the `intermediary` parameter on `Data`. It handles fetching, timing alignment, and candle construction.

## Transforming candle data

You have candle data. Now you need it in a different shape.

### `merge_candles()`

You're aggregating across multiple sources — say, combining BTC/USDT from three exchanges into one consolidated feed.

`merge_candles()` takes a list of candle dicts and produces one at a common `candle_size`:

```python
from qtradex.public.utilities import merge_candles

merged = merge_candles([binance_candles, kraken_candles, coinbase_candles], candle_size=86400)
```

For each time bucket it takes the first open, last close, max high, min low, and max volume. Quantization aligns timestamps to candle boundaries automatically.

### `reaggregate()`

You have 1-minute candles and you want 15-minute candles. Simple aggregation (open-first, close-last, etc.) loses information — the internal price path within each 1-minute candle is discarded.

`reaggregate()` takes a different approach: it disaggregates each candle into its implied open-high-low-close points, then re-aggregates at the target size. The result preserves OHLC shape more faithfully than a simple resample.

```python
from qtradex.public.utilities import reaggregate

fifteen_min = reaggregate(one_min_candles, candle_size=900)
```

Pass `stride` to create overlapping or gapped windows, same as `load_csv`.

### `interpolate()`

You have daily data but your strategy needs hourly indicators. Resampling introduces artifacts — `interpolate()` uses cubic spline interpolation across each OHLC channel to produce a smoother transition:

```python
from qtradex.public.utilities import interpolate

hourly = interpolate(daily_candles, oldperiod=86400, newperiod=3600)
```

Volume is scaled by `newperiod / oldperiod` with nearest-neighbor matching.

The caveat: interpolation introduces data that never existed. The resulting candles are plausible estimates, not real observations. Use it for prototyping or visualization, not for production backtesting where precision matters.

## Data slicing

### `clip_to_time_range()`

You fetched a year of data but only want to test on Q3:

```python
from qtradex.public.utilities import clip_to_time_range

q3_candles = clip_to_time_range(all_candles, start_unix, end_unix)
```

Returns a new candle dict with only candles whose timestamps fall within the range.

### `invert()`

Your strategy expects USDT/BTC but your exchange gives you BTC/USDT:

```python
from qtradex.public.utilities import invert

usdt_btc = invert(btc_usdt_candles)
```

Prices become `1/price`. High and low are swapped (what was the high in BTC/USDT is the low in USDT/BTC). Volume adjusts to preserve quote-currency flow.

## fine_data

Backtests run on candles. A candle hides the price movement inside it — open, high, low, close compress thousands of trades into four numbers. That's fine for backtesting, but papertrade and live modes execute real (or simulated) fills. The fill price matters.

If the candle opens at 100 and closes at 110, did the price go straight up? Or did it dip to 95 first? A market buy at the open fills differently in each case.

`fine_data` is a higher-resolution version of `raw_candles`, stored as a separate candle dict on the `Data` object. These finer candles aren't used for strategy decisions — they're used for fill simulation. A buy at the open is evaluated against the actual 1-minute bars inside that day, giving a more accurate fill price.

You rarely interact with `fine_data` directly. By default, `Data()` sets it to `None` — a plain backtest uses the candle data as-is. When you enter papertrade or live mode, the engine calls `fetch_composite_data()` internally to pull higher-resolution candles and populate `fine_data`. That's why those modes are more accurate than a simple backtest — and why they take slightly longer to initialize (the finer data needs to be fetched and cached too).

## Next steps

- **API Reference**: [`api/data.md`](api/data.md) — full parameter docs for every function
- **Getting Started**: [`getting-started.md`](getting-started.md) — build your first bot with any data source
