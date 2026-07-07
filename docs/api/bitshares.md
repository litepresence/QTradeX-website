# BitShares Integration

QTradeX supports the BitShares DEX as both a data source and a live trading venue. The integration spans three layers: low-level RPC utilities for blockchain queries, a `BitsharesExchange` class for order management, and a Kibana-based data fetcher for historical swap data.

---

## BitShares nodes

```python
from qtradex.common.bitshares_nodes import bitshares_nodes
```

A list of active BitShares WebSocket node URLs used by default across all RPC functions. Pass a custom list to `wss_handshake(node=...)` to override.

```python
bitshares_nodes
# -> ["wss://node1.bitshares.eu/ws", "wss://api.bitshares.bhuz.info/ws", ...]
```

## RPC Utilities

```python
from qtradex.public.rpc import (
    wss_handshake, wss_query,
    rpc_get_objects, rpc_get_multiple_objects,
    rpc_market_history, rpc_lookup_asset_symbols,
    rpc_account_by_name, rpc_book, rpc_last,
    rpc_open_orders, rpc_pool_book,
    get_bitshares_balances,
    id_from_name, id_to_name, precision,
)
```

Low-level WebSocket JSON-RPC wrappers for the BitShares blockchain. All RPC functions take a connection object returned by `wss_handshake()` as their first argument.

### Connection

```python
rpc = wss_handshake(node=None)  # node: str URL or list of URLs, or None for defaults
```

Creates a WebSocket connection to a BitShares node. If `node` is a list, it tries each one in order until a connection succeeds. If `None`, uses the built-in node list from `qtradex.common.bitshares_nodes`.

```python
wss_query(rpc, params)  # params: ["api", "method", [args]]
```

Low-level JSON-RPC call. All other RPC functions build on this.

### Asset Lookup

```python
rpc_lookup_asset_symbols(rpc, asset, currency)
# -> [{"id": "1.3.x", "symbol": "BTC", "precision": 8}, ...]

id_from_name(rpc, object_name)           # "BTC" -> "1.3.x"
id_to_name(rpc, object_id)               # "1.3.x" -> "BTC"
precision(rpc, object_id)                # "1.3.x" -> 8
```

Resolve between human-readable asset names and BitShares object IDs. `precision()` is cached to `precisions.txt`. `id_from_name`/`id_to_name` are cached to `ids_to_names.txt` / `names_to_ids.txt`.

### Market Data

```python
rpc_market_history(rpc, currency_id, asset_id, period, start_unix, stop_unix)
# -> kline data list

rpc_last(rpc, pair)                # "BTC:USD" -> latest price as float
rpc_book(rpc, asset, currency, depth=3)
# -> {"askp": [...], "askv": [...], "bidp": [...], "bidv": [...]}

rpc_pool_book(pool_data)
# -> {"askp": [...], "askv": [...], "bidp": [...], "bidv": [...]}
```

`rpc_book` fetches the order book for a traditional (order-book) market. `rpc_pool_book` computes the implied order book of a liquidity pool from its `balance_a` / `balance_b` / `asset` / `currency` state using constant-product (`x * y = k`) math.

### Account

```python
rpc_account_by_name(rpc, account)    # "username" -> account object
rpc_open_orders(rpc, account_name, storage, market)
# -> {"bids": [...], "asks": [...], "bid_sum": ..., "ask_sum": ...}
get_bitshares_balances(rpc, api)     # via API dict with "pair", "user_id"
# -> {"asset_free": ..., "currency_free": ...}
```

### Objects

```python
rpc_get_objects(rpc, obj_id)               # "1.3.0" -> single object
rpc_get_multiple_objects(rpc, assets_list)  # ["1.3.0", "1.3.1"] -> list
```

---

## `BitsharesExchange`

```python
from qtradex.private.bitshares_exchange import BitsharesExchange
```

Requires the optional `bitshares-signing` package (`pip install qtradex[bitshares]`).

Implements the same interface as CCXT exchange wrappers used by `Execution` and live trading, adapted for the BitShares DEX.

```python
exchange = BitsharesExchange(user="your_username", wif="your_private_key")
```

| Method | Description |
|--------|-------------|
| `fetch_my_trades(symbol)` | Historical fills for an account on a market (`"BTC/USD"`) |
| `create_order(symbol, order_type, side, amount, price)` | Place limit or swap order |
| `cancel_order(order_id, symbol)` | Cancel a single order |
| `cancel_orders(ids, symbol)` | Cancel multiple orders |
| `cancel_all_orders(symbol)` | Cancel all orders on a market |
| `fetch_open_order(order_id, _)` | Get a specific open order |
| `fetch_open_orders(symbol)` | Get all open orders on a market |
| `fetch_balance()` | Account balances |
| `fetch_ticker(symbol)` | Current ticker for a market |

When `exchange_id="bitshares"` is passed to `Execution`, this class is used automatically:

```python
exec = Execution("bitshares", asset, currency, api_key=username, api_secret=wif)
```

---

## Kibana/Elasticsearch Data

```python
from qtradex.public.kibana import get_klines
```

Fetches historical swap data from BitShares liquidity pools via Kibana/Elasticsearch. The query builder in `kibana_queries.py` constructs the Elasticsearch request body to retrieve swap event data, which is then converted to OHLCV candles.

This is an internal data source used by the `Data` class when `exchange="bitshares"` and a `pool` parameter is provided. It is also available directly for custom data-fetching scripts.
