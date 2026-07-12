# Stock Exchange Lookup

```python
from qtradex.public.stock_exchange_lookup import EXCHANGE_LOOKUP
```

A dictionary mapping ~200 global stock exchange identifiers to their quote currencies. Used internally when fetching data from Alpha Vantage and Finance Data Reader to determine the quote currency for stock tickers.

```python
EXCHANGE_LOOKUP["NASDAQ"]    # "USD"
EXCHANGE_LOOKUP["LSE"]       # "GBP"
EXCHANGE_LOOKUP["TSE"]       # "JPY"
EXCHANGE_LOOKUP["EURONEXT"]  # "EUR"
```

Coverage includes major indices and exchanges across North America, Europe, Asia-Pacific, and emerging markets.

| Region | Examples |
|--------|----------|
| North America | `NYSE`, `NASDAQ`, `TSX`, `BOVESPA`, `BOLSA` |
| Europe | `LSE`, `FRA`, `EPA`, `XETRA`, `BME`, `SWX`, `BIT` |
| Asia-Pacific | `TSE`, `HKE`, `SSE`, `ASX`, `NSE`, `KRX` |
| Middle East / Africa | `TADAWUL`, `QSE`, `JSE`, `EGX`, `NSE` |
| Indices | `SPX`, `DJI`, `IXIC`, `FTSE`, `DAX`, `CAC`, `NIKKEI` |
