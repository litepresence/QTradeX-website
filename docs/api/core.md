# Core

```python
import qtradex as qx
```

## `qx.dispatch()`

```python
dispatch(bot, data, wallet=None, **kwargs)
```

Interactive CLI menu. Presents options: Backtest, Optimize, Papertrade, Live, [Show Fill Orders](filltest.md), [AutoBacktest](auto-backtest.md), [Monte Carlo](monte-carlo.md). Routes to the appropriate mode with your chosen tune.

The optimizer sub-menu includes [Gravitas](gravitas.md) — a sensitivity scanner for your tune parameters.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `bot` | `BaseBot` | required | Bot instance |
| `data` | `Data` | required | Market data |
| `wallet` | `PaperWallet` | `None` | Starting wallet (default: 1 unit currency, 0 asset) |
| `**kwargs` | — | — | Forwarded to backtest/papertrade/live/optimizer calls |

Returns `None`. Interactive only.

## `qx.backtest()`

```python
metrics = backtest(bot, data, wallet=None, plot=True, block=True,
                   return_states=False, range_periods=True, show=True,
                   fine_data=None, always_trade="smart")
```

Run a historical simulation.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `bot` | `BaseBot` | required | Bot instance |
| `data` | `Data` | required | Market data |
| `wallet` | `PaperWallet` | `None` | Starting wallet |
| `plot` | `bool` | `True` | Show chart |
| `block` | `bool` | `True` | Block until plot window closes |
| `return_states` | `bool` | `False` | Return raw states alongside metrics |
| `range_periods` | `bool` | `True` | Auto-scale `_period` params to candle size |
| `show` | `bool` | `True` | Print results to console |
| `fine_data` | `Data` | `None` | Optional higher-resolution data for fill precision |
| `always_trade` | `bool` / `str` | `"smart"` | `"smart"` = trade only when elapsed time >= fine candle size; `True` = every tick |

Returns `dict` of performance metrics. If `return_states=True`, returns `[metrics, raw_states, processed_states]`.

## `qx.papertrade()`

```python
papertrade(bot, data, wallet=None, tick_size=900, tick_pause=300, **kwargs)
```

Simulated live trading with live data feeds. Uses `PaperWallet` — no real money, no exchange credentials needed. Runs in an infinite loop.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `bot` | `BaseBot` | required | Bot instance |
| `data` | `Data` | required | Market data |
| `wallet` | `PaperWallet` | `None` | Starting wallet |
| `tick_size` | `int` | 900 | Seconds between ticks (15 min) |
| `tick_pause` | `int` | 300 | Seconds to pause after each tick (5 min) |
| `**kwargs` | — | — | Forwarded to internal `backtest()` calls |

Returns `None`. Runs until interrupted.

## `qx.live()`

```python
live(bot, data, api_key, api_secret, dust, tick_size=900, tick_pause=900, cancel_pause=7200, **kwargs)
```

Real trading via exchange API. Uses a live `Wallet` — real orders, real risk. Runs in an infinite loop.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `bot` | `BaseBot` | required | Bot instance |
| `data` | `Data` | required | Market data |
| `api_key` | `str` | required | Exchange API key |
| `api_secret` | `str` | required | Exchange API secret |
| `dust` | `float` | required | Minimum trade amount (smaller skipped) |
| `tick_size` | `int` | 900 | Seconds between ticks (15 min) |
| `tick_pause` | `int` | 900 | Seconds to pause after each tick |
| `cancel_pause` | `int` | 7200 | Seconds between order cancellation sweeps (2 hrs) |
| `**kwargs` | — | — | Forwarded to internal `backtest()` calls |

Returns `None`. Runs until interrupted.

## Internal utilities

These are used by the backtest engine but available for custom use:

```python
from qtradex.core.quant import slice_candles, filter_glitches, preprocess_states
```

### `slice_candles(now, data, candle, depth)`

Efficient candle windowing using binary search. Given a timestamp and a `Data` object, returns a slice of `depth` candles ending at or before `now`.

```python
window = slice_candles(timestamp, my_data, candle_size, 100)
```

### `filter_glitches(days, tune)`

Adjusts training days to skip early exchange data that may contain glitches (bad candles, missing ticks). Called during `Data` construction for certain exchanges. Returns adjusted day count.

### `preprocess_states(states, pair)`

Converts raw backtest states into formatted arrays for metric computation. Returns processed states with wins, losses, balance values, and hold curves.


