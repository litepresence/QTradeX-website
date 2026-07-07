# Utilities

```python
from qtradex import expand_bools, rotate, truncate
```

## `rotate()`

```python
rotate(data) -> dict | list
```

Transpose between list-of-dicts and dict-of-lists.

```python
# dict-of-lists → list-of-dicts
rotate({"open": [1, 2], "close": [3, 4]})
# → [{"open": 1, "close": 3}, {"open": 2, "close": 4}]

# list-of-dicts → dict-of-lists
rotate([{"a": 1}, {"a": 2}])
# → {"a": array([1, 2])}
```

## `truncate()`

```python
truncate(*args) -> tuple
```

Truncate multiple arrays to the length of the shortest, keeping the newest (rightmost) values.

```python
a = [1, 2, 3, 4, 5]
b = [10, 20, 30]
truncate(a, b)  # → ([3, 4, 5], [10, 20, 30])
```

## `expand_bools()`

```python
expand_bools(bool_list, side="both") -> list
```

Expand `True` values to adjacent positions.

| `side` | Behavior |
|--------|----------|
| `"both"` | Each `True` also sets neighbors to `True` |
| `"right"` | Each `True` also sets the next element to `True` |
| `"left"` | Each `True` also sets the previous element to `True` |

```python
expand_bools([False, True, False], "both")  # → [True, True, True]
expand_bools([True, False, False], "right")  # → [True, True, False]
```

## `it()`

```python
it(style, text) -> str
```

Wrap text in ANSI color codes for terminal output.

| `style` | Color |
|---------|-------|
| `"black"`, `"red"`, `"green"`, `"yellow"`, `"blue"`, `"purple"`, `"cyan"`, `"white"` | Standard foreground colors |
| `"default"` | Terminal default |

```python
print(it("green", "profit"), it("red", "loss"))
```

## `sigfig()`

```python
sigfig(number, sig) -> float
```

Round a number to `sig` significant figures.

```python
sigfig(3.14159, 3)  # → 3.14
sigfig(1234567, 3)  # → 1230000.0
sigfig(0.000123, 2)  # → 0.00012
```

## `satoshi()` / `satoshi_str()`

```python
satoshi(number) -> float
satoshi_str(number) -> str
```

Round a price to 8 decimal places (the standard satoshi precision).

```python
satoshi(0.123456789)      # → 0.12345679
satoshi_str(0.123456789)  # → "0.12345679"
```

## `print_table()`

```python
print_table(data, x_pos=-1, y_pos=0, render=False, colors=None, pallete=None)
```

Print a formatted table to the terminal. `data` is a list-of-lists (rows then columns). Handles numpy arrays (renders as colored sparklines) and floats (formatted with `sigfig`). Returns the rendered text when `render=True`.

```python
print_table([["Metric", "Value"], ["ROI", 1.2345], ["Sharpe", 0.9876]])
```

## `trace()`

```python
trace(error) -> str
```

Print a formatted stack trace on exception. Returns the error string.

## `format_timeframe()` / `unformat_timeframe()`

```python
format_timeframe(seconds) -> str
unformat_timeframe(timeframe) -> int
```

Convert between seconds and candle-size string notation.

```python
format_timeframe(300)        # → "5m"
format_timeframe(86400)      # → "1d"
format_timeframe(604800)     # → "1w"
unformat_timeframe("4h")     # → 14400
unformat_timeframe("1d")     # → 86400
```

## `read_file()` / `write_file()`

```python
read_file(path) -> str
write_file(path, contents)
```

Read or write a file. `write_file` serializes via `json.dumps` (with `NdarrayEncoder` for numpy arrays).

## `json_ipc()`

```python
json_ipc(doc="", text=None, initialize=False, append=False)
```

Concurrent interprocess communication via JSON files. Used internally by live trading and data caching (`qtradex/common/pipe/`). Reads when `text` is `None`, writes otherwise. Mitigates race conditions with retry backoff and postscript clipping.

On write, the text is wrapped in `<<< JSON IPC >>>` tags that the reader uses to validate it received a complete message (file truncation during concurrent writes won't produce valid JSON).

```python
# write
json_ipc("my_data.json", '{"price": 50000}')

# read
data = json_ipc("my_data.json")  # -> {"price": 50000}

# append (comptroller IPC)
json_ipc("audit.log", '{"trade": "buy"}', append=True)
```

Background: the `pipe/` directory lives at `qtradex/common/pipe/` and is created on first use. Append operations go to a `pipe/comptroller/` subdirectory. You can tail the files in real time:

```bash
tail -F qtradex/common/pipe/my_data.json
```

## `block_print()` / `enable_print()`

```python
from qtradex.common.utilities import block_print, enable_print
```

Suppress and restore stdout. Useful for silencing noisy third-party libraries during imports or setup.

```python
block_print()       # stdout -> /dev/null
# noisy stuff here
enable_print()      # stdout restored
```

## `print_elapsed()`

```python
from qtradex.common.utilities import print_elapsed
```

Decorator that prints the wall-clock execution time of a function.

```python
@print_elapsed
def my_function():
    ...
```

## `parse_date()`

```python
from qtradex.common.utilities import parse_date
```

Parse `"YYYY-MM-DD"` strings or Unix timestamps into Unix timestamps.

```python
parse_date("2024-01-15")   # -> 1705276800
parse_date(1705276800)      # -> 1705276800 (passthrough)
```

## `to_iso_date()` / `from_iso_date()`

```python
from qtradex.common.utilities import to_iso_date, from_iso_date
```

Convert between Unix timestamps and ISO8601 strings.

```python
to_iso_date(1705276800)        # -> "2024-01-15T00:00:00"
from_iso_date("2024-01-15T00:00:00")  # -> 1705276800
```

## CLI helpers

```python
from qtradex.core.ui_utilities import logo, select, get_number
```

**`logo(animate=False)`** — Print the QTradeX ASCII flame logo. Optional animation effect.

**`select(options)`** — Print a numbered menu and return the user's choice.

**`get_number(options)`** — Get a number from the user given a list of valid choices.

Used internally by `qx.dispatch()`.

## Constants

```python
from qtradex.common.utilities import PATH, NIL
```

- **`PATH`** — `str`. Package root path (ends with `qtradex/common/`). The `pipe/` IPC directory lives here.
- **`NIL`** — `float` = `10e-10`. Near-zero sentinel used in balance computations and comparison contexts.

## Internal utilities

These are used internally by the framework but available if needed:

- **`NonceSafe`** — Context manager for process-safe nonce generation (file-lock based).
- **`NdarrayEncoder`** / **`NdarrayDecoder`** — JSON encoder/decoder for numpy arrays.
- **`Period`**, **`FloatPeriod`**, **`IntPeriod`** — Period type classes. `period(num)` returns `IntPeriod` for integers, `FloatPeriod` for floats.
- **`race_read()`** / **`race_write()`** — Lower-level concurrent file I/O with exponential backoff retry.
- **`red_to_green_fade(value)`** — Maps 0–255 to xterm256 color codes (red → green). Used internally by `print_table()`.
- **`strip_ansi(string)`** / **`ljust_ansi(string, length)`** — ANSI-safe string formatting. Used internally by `print_table()`.
