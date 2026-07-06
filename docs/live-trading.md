# Live Trading

## Dispatch

`qx.dispatch(bot, data, wallet)` opens an interactive menu:

- **Backtest** — run against historical data
- **AutoBacktest** — run with automatic parameter sweep
- **Optimize** — tune parameters against fitness function
- **Papertrade** — simulate live trading with PaperWallet
- **Live** — execute on a real exchange
- **MonteCarlo** — randomized parameter testing
- **ShowFillOrders** — review historical fills

## Wallets

| Wallet | Purpose |
|--------|---------|
| `qx.PaperWallet()` | Simulated trading, no real funds |
| `qx.Wallet(api_key, secret)` | Real exchange wallet |

## Data

```python
data = qx.Data(
    exchange="kucoin",
    asset="BTC",
    currency="USDT",
    begin="2020-01-01",
    end="2023-01-01"
)
```

Fetches candles via CCXT (100+ exchanges) or yfinance. Results are
disk-cached for fast re-runs.
