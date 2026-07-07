# BitShares Integration Walkthrough

Centralized exchanges require KYC, have withdrawal limits, and can freeze your funds. BitShares is a DEX where you control your keys. QTradeX supports it end-to-end — from historical data to live trading.

## Setup

Install the extras package for BitShares transaction signing:

```bash
pip install qtradex[bitshares]
```

This pulls in the `bitshares-signing` dependency needed to create and broadcast orders. Without it, you can still query the chain — you just can't trade.

BitShares nodes are public and change over time. QTradeX ships a curated list you can inspect:

```python
from qtradex.common.bitshares_nodes import bitshares_nodes

bitshares_nodes
# -> ["wss://node1.bitshares.eu/ws", "wss://api.bitshares.bhuz.info/ws", ...]
```

Every connection function uses this list by default. If one node is down, it tries the next. You never configure node URLs yourself.

## Data source

Historical price data on BitShares doesn't come from a central API. Swap events on liquidity pools are logged to Kibana/Elasticsearch. QTradeX queries that index and converts the raw swap records into OHLCV candles.

You pull data the same way you would for Binance or KuCoin — through `qx.Data`:

```python
data = qx.Data(
    exchange="bitshares",
    asset="BTC",
    currency="USD",
    pool="1.7.x",       # liquidity pool ID on BitShares
    begin="2024-01-01",
    end="2024-06-01",
)
```

The `pool` parameter is required for BitShares. It tells the data fetcher which liquidity pool to read swap events from. Without it, there's no way to construct a price series.

What comes back is a standard candle dict: `{"open": [...], "high": [...], "close": [...], "volume": [...], "unix": [...]}`. Same shape as any other exchange. Your indicators and strategy don't know or care that the source was a DEX swap log.

This only covers swap markets. For traditional order-book markets on BitShares, you need the RPC layer instead.

## RPC utilities

The DEX exposes real-time chain state through WebSocket JSON-RPC. You connect, query, and disconnect — no API keys needed for public data.

Start with a connection:

```python
from qtradex.public.rpc import wss_handshake

rpc = wss_handshake()          # uses built-in node list with fallback
rpc = wss_handshake(node="wss://my-node.example/ws")  # or pick your own
```

Once connected, you can query anything on chain.

### Asset resolution

BitShares uses internal object IDs (like `1.3.861`) instead of ticker symbols. You'll need to translate back and forth:

```python
from qtradex.public.rpc import rpc_lookup_asset_symbols, id_from_name, id_to_name, precision

rpc_lookup_asset_symbols(rpc, "BTC", "USD")
# -> [{"id": "1.3.861", "symbol": "BTC", "precision": 8}, ...]

id_from_name(rpc, "BTC")      # -> "1.3.861"
id_to_name(rpc, "1.3.861")    # -> "BTC"
precision(rpc, "1.3.861")     # -> 8
```

`precision()` is cached to a local file. `id_from_name` and `id_to_name` are cached too. After the first call, asset resolution is instant.

### Market data queries

Three functions cover the common cases:

```python
from qtradex.public.rpc import rpc_book, rpc_last, rpc_market_history

rpc_last(rpc, "BTC:USD")       # latest price as a float

rpc_book(rpc, "BTC", "USD", depth=5)
# -> {"askp": [...], "askv": [...], "bidp": [...], "bidv": [...]}

rpc_market_history(rpc, currency_id, asset_id, period, start_unix, stop_unix)
# -> kline data list
```

`rpc_book` fetches the order book for traditional (non-swap) markets. `rpc_market_history` returns kline data — the on-chain equivalent of historical candles for order-book markets.

### Pool order book

Liquidity pools don't have an order book. But you can compute what the order book *would* look like from the pool's reserves using constant-product math:

```python
from qtradex.public.rpc import rpc_pool_book

pool_data = rpc_get_objects(rpc, "1.7.x")  # raw pool state
book = rpc_pool_book(pool_data)
# -> {"askp": [...], "askv": [...], "bidp": [...], "bidv": [...]}
```

The implied book shows what price you'd get for a given swap size, derived from `x * y = k`.

### Wallet balances

Check what's in a BitShares wallet:

```python
from qtradex.public.rpc import get_bitshares_balances

api = {"pair": "BTC:USD", "user_id": "1.2.12345"}
balances = get_bitshares_balances(rpc, api)
# -> {"asset_free": 0.5, "currency_free": 1000.0}
```

Returns free (non-frozen) balances for both sides of the pair.

## Live trading

BitShares authentication is different from every other exchange. You don't use API keys and secrets. You use your BitShares account name and your WIF (Wallet Import Format) private key.

These are never stored in files. `dispatch()` prompts for them interactively:

```
Enter API key:    ****    (your BitShares username)
Enter API secret: ****    (your WIF private key)
```

Behind the scenes, the `BitsharesExchange` class handles signing and broadcasting:

```python
from qtradex.private.bitshares_exchange import BitsharesExchange

exchange = BitsharesExchange(user="your_username", wif="your_private_key")
```

You'll rarely instantiate it directly. When you pass `exchange_id="bitshares"` to `Execution`, it picks `BitsharesExchange` automatically:

```python
exec = Execution("bitshares", asset, currency, api_key=username, api_secret=wif)
```

The interface is the same as CCXT exchange wrappers:

| Method | What it does |
|--------|-------------|
| `create_order(symbol, type, side, amount, price)` | Place limit or swap order |
| `cancel_order(order_id, symbol)` | Cancel a single order |
| `cancel_orders(ids, symbol)` | Cancel multiple orders |
| `cancel_all_orders(symbol)` | Cancel all orders on a market |
| `fetch_open_orders(symbol)` | Get all open orders on a market |
| `fetch_open_order(order_id, _)` | Get a specific open order |
| `fetch_balance()` | Account balances (from chain) |
| `fetch_my_trades(symbol)` | Historical fills for your account |
| `fetch_ticker(symbol)` | Current ticker for a market |

The cancel cycle works the same way as CCXT exchanges. Every `cancel_pause` seconds (default 2 hours), stale orders are cancelled and replaced with fresh ones based on the latest signal. This matters more on BitShares because DEX orders don't expire automatically.

## Full bot example

Let's wire up a complete BitShares bot. It uses the same `BaseBot` structure you'd use anywhere else — only the data source and exchange change.

### Step 1: Set up data with a pool ID

```python
import qtradex as qx

data = qx.Data(
    exchange="bitshares",
    asset="BTC",
    currency="USD",
    pool="1.7.x",
    begin="2024-01-01",
    end="2024-06-01",
)
```

You need to know the pool ID for the market you want. Look it up on a BitShares block explorer or use `rpc_lookup_asset_symbols` to find the asset IDs and cross-reference with pool state.

### Step 2: Write the bot

A simple EMA crossover — same as the getting-started example, but pointed at BitShares data:

```python
class EMACrossBot(qx.BaseBot):
    def __init__(self):
        self.tune = {
            "fast_ema_period": 10.0,
            "slow_ema_period": 50.0,
        }
        self.clamps = {
            "fast_ema_period": [5, 7.5, 50, 1],
            "slow_ema_period": [20, 35, 100, 1],
        }
 
    def indicators(self, data):
        return {
            "fast_ema": qx.ti.ema(data["close"], self.tune["fast_ema_period"]),
            "slow_ema": qx.ti.ema(data["close"], self.tune["slow_ema_period"]),
        }
 
    def strategy(self, state, indicators):
        if indicators["fast_ema"] > indicators["slow_ema"]:
            return qx.Buy(reason="fast crossed above slow")
        elif indicators["fast_ema"] < indicators["slow_ema"]:
            return qx.Sell(reason="fast crossed below slow")
        return qx.Hold()

    def plot(self, *args):
        qx.plot(self.info, *args, (
            ("fast_ema", "EMA Fast", "white", 0, "EMA Cross"),
            ("slow_ema", "EMA Slow", "cyan", 0, "EMA Cross"),
        ))
```

Nothing here is BitShares-specific. The same bot class works with data from any exchange.

### Step 3: Run via dispatch

```python
bot = EMACrossBot()
qx.dispatch(bot, data)
```

The interactive menu gives you three paths:

1. **Backtest** — runs against the historical data you fetched. No blockchain connection needed, no authentication. Fast iteration.
2. **Papertrade** — connects to the BitShares chain for live data but uses a virtual wallet. Your strategy runs on real ticks but no real money moves.
3. **Live** — connects to the chain, prompts for your username and WIF key, and places real orders.

A typical workflow: backtest first to validate the idea, papertrade for a few days to catch runtime issues, then go live.

### What live mode looks like

When you select live mode, `dispatch()` detects `exchange="bitshares"` and asks for BitShares credentials. You type your username and WIF key at the prompt — they're not stored on disk.

The tick loop starts. Every `tick_size` seconds, it fetches fresh candles from the Kibana data source (swap events converted to candles). Your `strategy()` runs on the latest tick. If it returns a `Buy` or `Sell` signal, a limit order is placed on the BitShares DEX through the WebSocket connection.

Orders that don't fill get cancelled and replaced on the next cancel cycle. If the chain node goes down, the retry loop waits 5 seconds and tries again — the bot doesn't crash.

## What's different from CCXT exchanges

If you've traded on Binance, Kraken, or KuCoin through QTradeX, a few things work differently on BitShares:

**No free public data API.** CCXT exchanges have public REST endpoints for candle data. BitShares doesn't. Historical swap data comes from Kibana/Elasticsearch. On-chain queries (`rpc_book`, `rpc_last`) work in real time but don't provide historical klines — use `rpc_market_history` for that.

**Authentication is username + WIF key.** Not API key + secret. The interactive prompt is the same, but the underlying signing mechanism is different. WIF keys are derived from your BitShares wallet and grant full account access — treat them with the same care as an exchange API secret.

**Two market types.** BitShares has traditional order-book markets (like any CEX) and liquidity pool swap markets (like Uniswap). `Data(exchange="bitshares")` with a `pool` parameter reads historical swap candles — open, high, low, close, volume. That's your historical price data. For order-book snapshots (current bids and asks), use `rpc_book` directly. The Data class handles history; the RPC layer handles the live book.

## Next steps

- **[BitShares API reference](api/bitshares.md)** — full function docs for RPC, BitsharesExchange, and Kibana data
- **[Wallet & auth](api/wallet.md)** — how the wallet system handles DEX balances
- **[Execution](api/execution.md)** — `Execution` class details for all exchange types
- **[Live trading](live-trading.md)** — papertrade and live mode mechanics
