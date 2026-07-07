# BaseBot

```python
class MyBot(qx.BaseBot):
    def __init__(self):
        self.tune = { ... }
        self.clamps = { ... }
```

## Methods to override

### `indicators(data)`

```python
def indicators(self, data: Data) -> dict
```

Must override. Takes a `Data` object, returns a dict of indicator arrays keyed by name. Values are numpy arrays, one element per candle, all the same length.

### `strategy(state, indicators)`

```python
def strategy(self, state: dict, indicators: dict) -> Buy | Sell | Thresholds | Hold | None
```

Called every tick. `state` contains: `last_trade`, `unix`, `wallet`, and the current candle OHLCV fields (`open`, `high`, `low`, `close`, `volume`). `indicators` is the single-tick indicator values (scalars, not arrays).

Return a signal or `None` (no action). Default returns `None`.

### `reset()`

```python
def reset(self) -> None
```

Called between backtest runs during optimization. Override to clear any internal state so results don't bleed between runs. Default is no-op.

### `execution(signal, indicators, wallet)`

```python
def execution(self, signal: Signal, indicators: dict, wallet: WalletBase) -> Signal
```

Post-strategy hook. Called with the signal your strategy returned, the full indicator arrays, and the wallet. Modify the signal (e.g., override the execution price) or return `None` to cancel the trade. Default returns the signal unchanged.

### `fitness(states, raw_states, asset, currency)`

```python
def fitness(self, states, raw_states, asset, currency) -> tuple[list, dict]
```

Controls which metrics are computed and whether to add custom metrics. Returns a list of metric keys and an optional dict of custom metrics. Default returns `(["roi", "cagr", "trade_win_rate"], {})`.

Available metric keys: `roi`, `cagr`, `sharpe_ratio`, `sortino_ratio`, `maximum_drawdown`, `calmar_ratio`, `omega_ratio`, `beta`, `alpha`, `info_ratio`, `profit_factor`, `trade_win_rate`, `payoff_ratio`, `skewness`, `kurtosis`, `efficiency_ratio`, `drawdown_duration`, `hurst_exponent`, `dpt`.

Custom metrics are included as `ind:name:low~high` in results.

### `plot(data, states, indicators, block)`

```python
def plot(self, data, states, indicators, block) -> None
```

Override to customize the chart. Default calls `qx.plot()` with an empty indicator format. The `axes` list from `qx.plot()` gives you direct access to matplotlib `Axes` objects for custom drawing.

### `autorange()`

```python
def autorange(self) -> int
```

Returns the warmup period in days. The engine skips this many candles before calling `strategy()`. Auto-computed from tune keys ending in `_period` — takes the maximum value and rounds up. Returns 0 if no `_period` keys exist. Override when your custom indicator needs more warmup than any single tune parameter suggests.

## Attributes your bot defines

| Attribute | Type | Description |
|-----------|------|-------------|
| `self.tune` | `dict` | Tune parameters. Keys ending in `_period` are auto-scaled to candle size (treated as days). Values can be floats or numpy arrays. |
| `self.clamps` | `dict` | Clamp ranges in format `{param: [min, midpoint, max, clamp_flag]}`. Set `clamp_flag=0` on a parameter to skip it during optimization. You can also use the short form `{param: [min, max]}` — the engine expands it to 4 elements automatically. |

## Attributes the framework sets

| Attribute | Type | Description |
|-----------|------|-------------|
| `self.info` | `Info` | Read-only dict. Keys: `mode` (`"backtest"`/`"papertrade"`/`"live"`), `start`, `live_data`, `live_trades`. |
| `self.gravitas` | `float` | Optional — used by the Gravitas optimizer. Not framework-enforced. |

## `Info`

Read-only dict attached to `bot.info`. Attempting to write raises `TypeError`.

Keys: `mode` (the current execution mode), `start` (start timestamp), `live_data` (high-resolution candle data during papertrade/live), `live_trades` (live fill records).
