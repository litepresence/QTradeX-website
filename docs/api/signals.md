# Signals

```python
from qtradex import Buy, Sell, Thresholds, Hold
```

## `Buy`

```python
Buy(
    price: float = None,      # Execution price override
    maxvolume: float = inf,   # Volume cap
    reason: str = None        # Optional label for trade history
)
```

Long entry signal. `is_override = True` — executes as a market order immediately.

| Attribute | Type | Description |
|-----------|------|-------------|
| `price` | `float | None` | Fill price override (default: market price) |
| `maxvolume` | `float` | Volume cap |
| `reason` | `str | None` | Optional label |
| `unix` | `int` | Timestamp (engine fills on execution) |
| `profit` | `float` | Profit (engine fills) |
| `is_override` | `bool` | Always `True` — market order |

## `Sell`

```python
Sell(
    price: float = None,
    maxvolume: float = inf,
    reason: str = None
)
```

Short entry or long-exit signal. Same structure as `Buy`. Market order.

## `Thresholds`

```python
Thresholds(
    buying: float,          # Price to buy if not in position
    selling: float,         # Price to sell if in position
    maxvolume: float = inf  # Maximum order volume
)
```

A limit-order signal. Doesn't execute immediately — the engine watches the market and converts it to a `Buy` or `Sell` when the price crosses a threshold:

- If the wallet holds **currency** (not in position): waits for `low < buying`, then places a `Buy` limit at `buying`.
- If the wallet holds **asset** (in position): waits for `high > selling`, then places a `Sell` limit at `selling`.

In live mode this becomes two simultaneous limit orders — a straddle around the current price.

| Attribute | Type | Description |
|-----------|------|-------------|
| `buying` | `float` | Entry threshold price |
| `selling` | `float` | Exit threshold price |
| `maxvolume` | `float` | Volume cap |
| `price` | `None` | Not set by constructor (engine fills from execution) |
| `unix` | `int` | Timestamp (engine fills on placement) |

## `Hold`

```python
Hold()
```

Neutral signal. Prevents execution hooks from triggering but lets them know the strategy evaluated. In live mode this cancels all open orders.
