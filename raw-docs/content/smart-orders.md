# Smart Order Execution

> **Experimental.** This feature is largely untested and may have bugs. The smart order strategies (drip, iceberg, top-of-book) run in background threads and interact with live exchange APIs. Test thoroughly on testnet before using with real funds.

Your strategy says "buy 10 BTC." You check the order book. There's 2 BTC at the best ask, then another 3, then the depth falls off. A single market order at that size will eat through three price levels before it fills. You just paid a premium because you asked for everything at once.

The problem is size. The solution is splitting, timing, and hiding intent.

## Setting up Execution directly

`Execution` wraps a CCXT exchange for order management. You give it an exchange, your credentials, and the trading pair:

```python
from qtradex.private.execution import Execution

exec = Execution("binance", "BTC", "USDT", api_key="...", api_secret="...")
```

Used internally by live trading and fill test. Available directly for custom scripts and one-off orders.

```python
exec.create_order("buy", "limit", 0.1, 60000)
exec.fetch_open_orders()
exec.cancel_all_orders()
```

The full list of order methods is in the [execution API reference](api/execution.md). This page focuses on the smart order strategies — the ones that solve the "too large for one shot" problem.

## Smart order strategies

Each strategy runs in a background thread. You call one method, it places orders until the total amount is exhausted or you tell it to stop.

### Drip by time

Split a large order into equal chunks, placed at regular intervals. The simplest way to avoid spiking the price.

```python
exec.drip_by_time("buy", total_amount=10, chunks=20, pause_seconds=60)
```

This buys 0.5 BTC every 60 seconds until all 10 BTC are filled. The market has time to absorb each chunk. No single trade moves the price.

Problem: you want to buy 10 BTC over 2 hours without anyone noticing your size. Drip by time is your baseline — it spreads out the impact and makes your order look like many small unrelated trades.

### Drip to limit

Same pattern, but with a hard price ceiling. The drip pauses if the market moves against your limit.

```python
exec.drip_to_limit(
    side="buy", total_amount=10, chunks=20,
    pause_seconds=60, limit_price=62000
)
```

Buys 0.5 BTC every 60 seconds, but only while the price stays at or below 62000. If BTC spikes above your limit, the loop waits until it comes back down before placing the next chunk.

Problem: you want to accumulate a position, but not above a certain price. Drip to limit gives you a ceiling — you drip in below your target, and the pause prevents you from buying the top.

### Drip to market

Waits for favorable market movement between each chunk. Instead of placing on a fixed clock, it checks the ticker and only places when the price moves in your direction.

```python
exec.drip_to_market(side="sell", total_amount=10, chunks=20)
```

Sells 0.5 BTC, then waits for the price to tick up before selling the next chunk. If the market is dropping, it holds. If the market rallies, it sells into strength.

Problem: you want to sell into strength, not at random intervals. Drip to market is patient — it waits for a green candle, then places the next chunk.

### Iceberg

Keeps only a small visible order on the book. When it fills, a new one replaces it at the same size. The market never sees your full hand.

```python
exec.iceberg_order(side="buy", total_amount=10, iceberg_limit=58000)
```

Shows a small order on the book. When it fills, another small order appears. This repeats until all 10 BTC are bought or the killswitch fires. The `iceberg_limit` is your max price — it only places orders when the price is at or below that level.

Problem: you don't want anyone to see how large your position is. A 10 BTC order sitting on the book invites front-running. Iceberg hides your size behind a sliver of visible volume.

The order sizing is automatic — it divides total by 10 and places orders of that size. Simple and effective.

### Top of book

Places your full order at the best bid or ask, then continuously cancels and replaces it as the order book moves. You stay first in line at any price.

```python
exec.top_of_book(side="buy", total_amount=10)
```

Re-places a 10 BTC buy order at the current best bid every 5 seconds. If the book shifts, your order shifts with it. You're always at the front.

Problem: you want to be first in line at any price. A static limit order gets buried as the book moves. Top of book keeps you at the front — your order is always the most aggressively priced. You'll fill faster than any static order.

## Using smart orders from `execution()`

The `execution()` hook runs after your strategy returns a signal. You start a smart order strategy there instead of placing a single trade.

Set up an `Execution` instance and attach it to your bot before dispatch:

```python
from qtradex.private.execution import Execution

class MyBot(qx.BaseBot):
    def strategy(self, state, indicators):
        # ... your logic ...
        return qx.Buy(maxvolume=10)
```

Wire it up:

```python
bot = MyBot()
bot.exec = Execution("binance", "BTC", "USDT", api_key="...", api_secret="...")
data = qx.Data("binance", "BTC", "USDT", days=365)
qx.dispatch(bot, data)
```

Inside `execution()`, check for the attribute and start a drip:

```python
    def execution(self, signal, indicators, wallet):
        if isinstance(signal, qx.Buy) and getattr(self, "exec", None):
            self.exec.drip_by_time("buy", 10, 20, 60)
            return None  # cancel the single order; the drip handles it
        return signal
```

The strategy signals "buy 10 BTC." Instead of placing one market order, `execution()` starts a drip and returns `None` to suppress the default trade. The drip runs in a background thread — your bot continues to its next tick while orders place in the background.

## Killswitch and thread safety

Every smart order strategy runs in its own background thread. Each one checks a shared `killswitch` flag between chunks. When the flag is set, the loop exits cleanly.

```python
exec.cancel_all_orders()    # sets killswitch, cancels all open orders
exec.cancel_order_threads() # sets killswitch, leaves orders on the book
```

`cancel_all_orders()` is the hard stop — it kills the threads and pulls every open order off the book. `cancel_order_threads()` just stops the threads; existing fillable orders stay.

Don't forget to clean up. If you start a drip and walk away, the thread keeps running. A bot shutdown or error handler should call one of these.

```python
try:
    exec.drip_by_time("buy", 10, 20, 60)
    # ... wait for fills ...
finally:
    exec.cancel_all_orders()
```

Thread safety comes from the killswitch pattern — it's a simple list with one boolean element, shared by reference. No locks needed. No race conditions on the cancellation path.

## Next steps

- [Execution API reference](api/execution.md) — full method signatures and parameters
- [Live trading](live-trading.md) — using Execution in a live bot loop
- [Advanced bot building](advanced-bot-building.md) — `execution()` hook, plot overrides, and more
