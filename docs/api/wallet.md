# Wallet

```python
from qtradex import PaperWallet, Wallet
```

## `PaperWallet`

Mutable paper wallet — balances passed in directly. Used for backtesting and paper trading.

```python
PaperWallet(balances=None, fee=1.0)
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `balances` | `dict` | `{}` | Initial balances: `{"BTC": 0.5, "USDT": 10000}` |
| `fee` | `float` | `1.0` | Fee in percent |

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `_protect()` | `()` | Lock wallet — prevents further balance changes |
| `_release()` | `()` | Unlock wallet — re-enables balance writes |

### Inherited from `WalletBase`

| Method | Signature | Description |
|--------|-----------|-------------|
| `copy()` | `()` | Returns a new `PaperWallet` with copied balances |
| `value()` | `(pair, price=None)` | Total value = `sqrt((asset * price + currency)^2 / price)`. Refreshes if method exists. |
| `items()` | `()` | `dict.items()` on balances |
| `keys()` | `()` | `dict.keys()` on balances |
| `values()` | `()` | `dict.values()` on balances |

Dict-style access: `wallet["BTC"]` reads, `wallet["BTC"] = 0.5` writes (if not protected).

## `Wallet`

Live wallet — wraps a CCXT exchange. Always read-only.

```python
Wallet(exchange)
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `exchange` | CCXT exchange | Must have `fetch_balance()` method |

Fetches live balances from the exchange on construction.

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `refresh()` | `()` | Pulls latest balances via `exchange.fetch_balance()["free"]` |

`__setitem__` is a no-op — live wallet balances are always read-only. `value()` triggers an automatic `refresh()` before computing.

## `WalletBase`

Common base class. Not instantiated directly by users. Provides dict-like interface and value calculation shared by both wallet types.
