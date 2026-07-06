# Data API

## `qx.Data(exchange, asset, currency, begin, end)`

Fetch historical candle data.

**Parameters:**

| Param | Type | Description |
|-------|------|-------------|
| `exchange` | str | Exchange name (e.g. `"kucoin"`, `"binance"`) |
| `asset` | str | Base asset (e.g. `"BTC"`) |
| `currency` | str | Quote currency (e.g. `"USDT"`) |
| `begin` | str | Start date `"YYYY-MM-DD"` |
| `end` | str | End date `"YYYY-MM-DD"` |

Supports 100+ exchanges via CCXT and yfinance as a fallback.

## `qx.load_csv(path)`

Load candle data from a CSV file.
