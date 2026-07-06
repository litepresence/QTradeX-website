# Signals API

| Signal | Signature | Behavior |
|--------|-----------|----------|
| `qx.Buy` | `(price=None, max_volume=None)` | Market buy. If `price` set, limit order. |
| `qx.Sell` | `(price=None, max_volume=None)` | Market sell. If `price` set, limit order. |
| `qx.Thresholds` | `(buying=None, selling=None)` | Limit order — fills only when price crosses threshold. |
| `qx.Hold` | `()` | No action. |

Set `signal.is_override = True` for immediate market execution regardless
of price conditions.
