# Writing a Bot

## BaseBot

Subclass `qx.BaseBot` and override these methods:

| Method | Purpose |
|--------|---------|
| `indicators(self, data)` | Return dict of computed indicator arrays |
| `strategy(self, tick_info, indicators)` | Return a signal (`Buy`, `Sell`, `Thresholds`, `Hold`) |
| `fitness(self)` | Return a score for optimization |
| `plot(self, *args)` | Draw trade/indicator charts |
| `reset(self)` | Reset state between backtest runs |

## Tune parameters

Set `self.tune` as a dict of float values. Keys ending in `_period` are
auto-scaled by candle size — treat them as day counts. Other keys are
never scaled.

```python
self.tune = {
    "fast_ema_period": 10.0,   # scaled to candle size
    "some_threshold": 0.5,     # never scaled
}
```

## Clamps

Clamps define optimization boundaries: `[min, midpoint, max, clamp_flag]`.
Set `clamp_flag` to 0 to skip a parameter during optimization.

```python
self.clamps = {
    "fast_ema_period": [5, 10, 50, 1],
}
```

## Signals

| Signal | Usage |
|--------|-------|
| `qx.Buy(price, max_volume)` | Market buy (or limit if price set) |
| `qx.Sell(price, max_volume)` | Market sell (or limit if price set) |
| `qx.Thresholds(buying, selling)` | Limit order — fills only when crossed |
| `qx.Hold()` | Do nothing |

Set `is_override = True` on a signal for immediate execution as a market order.

## Wallet

Default `PaperWallet` starts with `{asset: 0, currency: 1}` (1 USD).
Check balances with `wallet.balances.get(key, default)`.

## autorange

BaseBot's `autorange()` computes warmup days from the largest `_period`
tune value. Override if your custom indicator logic needs a different
warmup.
