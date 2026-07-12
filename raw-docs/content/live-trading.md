# Live Trading

You've backtested. You've optimized. At some point you need to see if your bot works on real data, with real ticks, in real time — without risking real money yet. That's what papertrade is for. Live trading is the same loop, but with real exchange orders.

Both modes start from `dispatch()`. Pick 2 for papertrade, 3 for live.

The dispatcher also includes backtest, optimize, fill orders history, and Monte Carlo. You'll find them when you need them.

## Papertrade — the safe rehearsal

Papertrade runs your bot against live market data as it arrives, using a virtual `PaperWallet`. No real money moves. Its purpose is to catch problems that backtesting missed: stale data, unexpected exchange behavior, your bot reacting badly to real-time price action.

### What happens when you choose it

1. **Set up the window.** Papertrade computes `bot.autorange() * 6` days of historical data as a warmup window. This runs a full backtest over that window so your indicators stabilize and the plot shows recent history. The wallet starts with 1 unit of currency — neutral, just for the warmup.

2. **Start the tick loop.** Every `tick_size` seconds (default 15 minutes), it fetches fresh candles from the exchange, runs the backtest engine over the sliding window (to update the plot), then calls your bot's `strategy()` and `execution()` on the latest tick only.

 3. **Execute in memory.** If your bot returns a signal, `trade()` applies it to the `PaperWallet` — adjusting balances in memory, no API calls. The result prints to console:

    ```
    Open:   12345.67
    Close:  12380.00
    Data latency: 30.0

    BUY - at 10.0 seconds since last trade
    Execution price: 12345.67

    Balances before: {'BTC': 0.0, 'USDT': 1.0}
    Balances after:  {'BTC': 8.33e-05, 'USDT': 0.0}
    ```

    The output shows OHLC, the signal type (Buy, Sell, or Thresholds), and before/after balances — everything you need to verify the bot is acting as expected. A `Thresholds` signal prints threshold prices instead of the execution price line.

4. **Tick pause.** It waits `tick_pause` seconds (default 5 minutes) before the next tick. During this wait, the matplotlib plot stays open and updates with each new candle.

### The data flow

You don't re-download your full history every tick. On each tick, `data.update_candles()` fetches only the new candles since the last fetch. Historical data is reused from a disk cache (`qtradex/common/pipe/`) — thread-safe JSON files keyed by exchange, asset, currency, and candle size.

There's one difference from backtest mode to watch for. In backtest mode, indicator values are merged into the tick data passed to your strategy. In papertrade, they're passed as a separate `indicators` dict — the tick data only contains the raw OHLCV row: `{"open": ..., "high": ..., "close": ..., "unix": now, "wallet": wallet}`.

This is an unintended consequence of how the two modes handle data internally, not a design choice. Don't rely on indicators being present in `tick_info` — always use the separate `indicators` argument.

## Live — real orders, real exchange

`live()` shares the same loop structure as `papertrade()`. The differences are about safety and reality.

**Wallet.** Instead of a `PaperWallet`, live mode creates a real `Wallet` that calls `exchange.fetch_balance()` to get your actual balances. Your `strategy()` and `execution()` receive a *copy* of the wallet — they can inspect balances but modifying the copy doesn't touch the real one.

**Execution.** Instead of in-memory balance adjustment, live mode places real limit orders through CCXT:

| Your signal | What gets placed |
|---|---|
| `Buy(price, volume)` | Limit buy at `price` (if `wallet.currency * price > dust`) |
| `Sell(price, volume)` | Limit sell at `price` (if `wallet.asset > dust`) |
| `Thresholds(buy, sell, volume)` | Both a limit buy and a limit sell |
| `Hold()` | Nothing (cancel cycle still runs on its timer) |

**Cancel cycle.** Every `cancel_pause` seconds (default 2 hours), all open orders for your symbol are cancelled and the wallet is refreshed before placing new ones. This prevents stale orders from accumulating.

**Dust filter.** Trades below `dust` (default 1e-8) are skipped entirely. A buy where `balance_in_currency * price < dust` produces nothing. Same for sells where the asset balance is below dust.

**Data retry.** Data fetching is wrapped in a retry loop that catches all exceptions and retries every 5 seconds. If the exchange goes down, the bot waits — it doesn't crash.

## Authentication

If your API key leaks, someone drains your account. So credentials are never stored in files or environment variables. `dispatch()` always prompts interactively:

```
Enter API key:    ****
Enter API secret: ****
```

For BitShares, the authentication is username + WIF private key. For every other exchange (Binance, Kraken, KuCoin, Coinbase, etc.), it's API key + secret. The exchange is determined by whatever you passed to `Data(exchange="...")`.

This means live trading always goes through the interactive CLI — you can't script it without a wrapper.

## Supported exchanges

| | Data fetching | Live orders |
|---|---|---|
| CCXT exchanges (Binance, Kraken, KuCoin, Coinbase, Bybit, OKX, ...) | ✓ | ✓ (authenticated) |
| BitShares | ✓ | ✓ (WIF key) |
| CryptoCompare, Alpha Vantage, Yahoo Finance | ✓ | — |
| Synthetic data | ✓ | — |

Data fetching works with any of the above — no authentication needed for public candle data. Live order placement requires the exchange to support CCXT's `create_limit_order` interface.

## What the plot shows

Both modes open a matplotlib window with the latest candles and trade markers. The difference: in papertrade, markers show virtual fills. In live mode, markers show real fills from `execution.fetch_my_trades()` — so you can see which orders actually got filled vs which are still sitting on the book.

## Edge cases

- **No trade on first tick.** If your bot doesn't return a signal on the very first tick, papertrade reuses the last trade from the warmup backtest (if any). Live mode just waits.
- **Exchange goes down.** Live mode retries data fetches indefinitely with 5-second pauses. The loop doesn't crash, but ticks will be missed until the exchange recovers.
- **Order doesn't fill.** Limit orders may sit on the book. The cancel cycle handles this — stale orders are cancelled and replaced with fresh ones based on the latest signal.
- **State doesn't leak between ticks.** Both modes call `bot.reset()` between ticks so strategy state from one tick doesn't carry over to the next.
