# Tune Management

An optimizer runs for hours and finds a great parameter set. Close the terminal and those results are gone — unless they're saved. The tune manager makes sure your best results survive between sessions.

See the [Tune Manager API reference](api/tune-management.md) for full function signatures.

## Saving tunes

Saving is automatic from every optimizer and from the MouseWheelTuner's Save button. You can also save manually:

```python
qx.save_tune(bot, "my_best_so_far")
```

Before writing, the function checks if the exact same tune values already exist in the file. Duplicate unlabeled entries are removed, but labeled ones (with underscores in the key) are preserved so your "BEST ROI TUNE" history accumulates over multiple runs.

## Loading tunes back

`qx.load_tune(bot)` reads the file and returns the tune dict with the highest ROI. Call it before your dispatch loop to pick up where your last optimization left off:

```python
bot = EmaCrossover()
bot.tune = qx.load_tune(bot)   # best ROI from cache
```

You can also sort by most recent instead of best ROI:

```python
bot.tune = qx.load_tune(bot, sort="latest")
```

During `qx.dispatch()`, the launcher presents an interactive menu: use best ROI tune, use most recent, use your current defaults, or browse the full history via `choose_tune()` — a numbered list where you pick one.

## Where tunes live

Tune files are stored in a `tunes/` directory right next to your bot source file. If your bot is at `strategies/ema_crossover.py`, the tunes go in `strategies/tunes/`. The directory is created automatically on first save.

## File naming

Each bot gets one JSON file, named `{module}_{param_count}.json`. So a bot with 4 tune parameters in `ema_crossover.py` produces `ema_crossover_4.json`.

The key is `len(bot.tune)` — the number of tune parameters. Two bots with the same module name and same number of parameters share a file. If you copy-paste a bot skeleton and only change the logic, your cached tunes still apply.

## JSON format

A tune file looks like this:

```json
{
    "source": "import qtradex as qx\nclass EmaCrossover(qx.Bot):\n    ...",
    "BEST ROI TUNE_Mon Jul  6 14:22:01 2026": {
        "tune": {"slow_ema": 44, "fast_ema": 12},
        "results": {"roi": 12.3, "sharpe": 1.4, "max_drawdown": -8.1}
    },
    "BEST SHARPE TUNE_Mon Jul  6 14:22:01 2026": {
        "tune": {"slow_ema": 38, "fast_ema": 9},
        "results": {"roi": 10.1, "sharpe": 1.8, "max_drawdown": -5.2}
    }
}
```

Every entry stores both the tune values and the backtest results that produced them. The `"source"` key embeds your bot's full source code — a record of what code generated those numbers.

Each tune key is a label with a timestamp appended (`"BEST ROI TUNE_Mon Jul  6 ..."`). This means the same optimizer run can save multiple bests — one per metric — and every save is uniquely identifiable without overwriting previous results.

## NumPy arrays in tunes

Tune files stay human-readable even when parameters are numpy arrays — common for indicator weight vectors or neural network coefficients:

```json
{"numpy_array": [1.0, 2.0, ...], "dtype": "float64"}
```

Arrays are serialized as lists with a type annotation and deserialized back automatically.

## CLI tool

There's also a standalone CLI for browsing and loading tunes outside of a bot run:

```bash
qtradex-tune-manager
```

It scans `./tunes/` in the current directory, lists available tune files sorted by modification time, and lets you pick a specific entry interactively.

## What's not handled

- No locking. Two processes saving to the same tune file simultaneously can lose data (last write wins, full file rewrite).
- Tune files are never cleaned up automatically. History accumulates indefinitely.
- No versioning scheme beyond the embedded source code and timestamps.
