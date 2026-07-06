# BaseBot API

## Constructor

Override to set `self.tune` and `self.clamps`.

## Methods

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `indicators` | `(self, data: dict) -> dict` | dict of array-like | Compute indicators per candle |
| `strategy` | `(self, tick_info, indicators) -> Signal` | Buy/Sell/Thresholds/Hold | Decision logic per tick |
| `fitness` | `(self) -> float` | Score | Fitness for optimization |
| `plot` | `(self, *args)` | None | Custom chart rendering |
| `reset` | `(self)` | None | Reset state for new backtest |
| `execution` | `(self, action, data, wallet, fill)` | None | Post-trade hook |
| `autorange` | `(self) -> int` | Warmup candles | Auto-computed from `_period` tune values |

## Public API imports from `qtradex`

`Data`, `load_csv`, `BaseBot`, `backtest`, `dispatch`, `live`, `papertrade`,
`load_tune`, `PaperWallet`, `Wallet`, `Buy`, `Sell`, `Thresholds`, `Hold`,
`ti`, `plot`, `plotmotion`, `derivative`, `fitness`, `float_period`, `lag`,
`expand_bools`, `rotate`, `truncate`.
