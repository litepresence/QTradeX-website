# Execution

```python
from qtradex.private.execution import Execution
```

Wraps a CCXT exchange for order management. Used internally by live trading and fill test, but available directly for custom execution scripts.

```python
exec = Execution(exchange_id, asset, currency, api_key=None, api_secret=None)
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `exchange_id` | `str` | required | Exchange identifier (any CCXT ID, or `"bitshares"`) |
| `asset` | `str` | required | Base asset symbol (e.g. `"BTC"`) |
| `currency` | `str` | required | Quote currency symbol (e.g. `"USDT"`) |
| `api_key` | `str` | `None` | API key |
| `api_secret` | `str` | `None` | API secret |

## Order methods

| Method | Description |
|--------|-------------|
| `create_order(side, type, amount, price)` | Create a limit order. `side`: `"buy"` or `"sell"`. Returns order dict or error string. |
| `cancel_order(order_id)` | Cancel a single open order. |
| `cancel_orders(order_ids)` | Cancel multiple open orders. |
| `cancel_all_orders()` | Cancel all open orders for this symbol. Sets `killswitch` to prevent re-placement by active threads. |
| `fetch_open_order(order_id)` | Get details of a specific open order. |
| `fetch_open_orders()` | Get all open orders for this symbol. |
| `fetch_my_trades()` | Get historical fills for this symbol. |
| `fetch_ticker(symbol, params=None)` | Get current ticker for any symbol. |

## Smart order strategies

These run in background threads and respect a `killswitch` — call `cancel_all_orders()` or `cancel_order_threads()` to stop them.

| Method | Signature | Description |
|--------|-----------|-------------|
| `create_market_order(side, amount, depth_percent=50)` | Creates a limit order at `depth_percent` below/above the market price |
| `drip_by_time(side, total_amount, chunks, pause_seconds)` | Places `chunks` equal orders every `pause_seconds`, totalling `total_amount` |
| `drip_to_limit(side, total_amount, chunks, pause_seconds, limit_price)` | Same as drip, but pauses if price crosses `limit_price` |
| `drip_to_market(side, total_amount, chunks)` | Same as drip, but waits for market movement between each chunk |
| `iceberg_order(side, total_amount, iceberg_limit)` | Places small orders to maintain position under `iceberg_limit` until exhausted |
| `top_of_book(side, total_amount)` | Continuously re-places a full-size order at the top of the order book |

## BitShares

When `exchange_id="bitshares"`, the `Execution` class uses `BitsharesExchange` instead of CCXT. Authentication uses username (`api_key`) and WIF private key (`api_secret`).
