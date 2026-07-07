# Tune Manager

```python
from qtradex import load_tune, save_tune
from qtradex.core.tune_manager import get_path, get_bots, get_tunes, choose_tune
```

The tune manager saves and loads bot parameter configurations (`bot.tune`) as JSON files. Each bot gets its own file in a `tunes/` directory next to the bot's source file.

See [Tune Management](../tune-management.md) for usage guide.

## `qx.save_tune()`

```python
save_tune(bot, identifier=None)
```

Save the current `bot.tune` to a JSON file. Each save gets a timestamp key. Duplicate unlabeled tunes are deduplicated automatically.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `bot` | `BaseBot` | required | Bot instance |
| `identifier` | `str` | `None` | Optional label (timestamp appended automatically) |

## `qx.load_tune()`

```python
tune = load_tune(bot, key=None, sort="roi")
```

Load a previously saved tune. Returns the tune dict — assign it to `bot.tune`.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `bot` | `BaseBot` | required | Bot instance or string path to a tune file |
| `key` | `str` | `None` | Specific tune key. `None` = auto-select by `sort` |
| `sort` | `str` | `"roi"` | Selection: `"roi"` (highest ROI), `"latest"` (most recent best-roi) |

Returns the tune `dict`. Raises `FileNotFoundError` if no tunes exist.

## `get_path()`

```python
path = get_path(bot)
```

Get the `tunes/` directory path for a bot. Creates it if it doesn't exist. Directory is `tunes/` next to the bot's source file.

## `get_bots()`

```python
bots = get_bots(bot)
```

List all stored bot tune file identifiers. Returns sorted `list[str]`.

## `get_tunes()`

```python
tunes = get_tunes(bot)
```

Retrieve all saved tunes for a bot. Returns the full JSON contents `dict`.

## `choose_tune()`

```python
tune = choose_tune(bot, kind="tune")
```

Interactive CLI tune selector. Presents a numbered menu of saved tunes sorted by ROI.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `bot` | `BaseBot` | required | Bot instance or tune file path |
| `kind` | `str` | `"tune"` | `"tune"` returns only the tune dict; `"any"` returns the full entry |

## CLI

```bash
qtradex-tune-manager
```

Launches an interactive browser for saved tunes. Lists bots by file, lets you inspect any saved tune.
