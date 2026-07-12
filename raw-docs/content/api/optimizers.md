# Optimizers

```python
from qtradex.optimizers import QPSO, LSGA, IPSE, AION, GridSearch, RLPPO, MouseWheelTuner
```

All optimizers follow the same interface:

```python
opt = Optimizer(data, wallet=wallet, options=options)
results = opt.optimize(bot, **kwargs)
```

## Common interface

### `__init__(data, wallet=None, options=None)`

| Parameter | Type | Default |
|-----------|------|---------|
| `data` | `Data` | required |
| `wallet` | `WalletBase` | varies by optimizer |
| `options` | options class instance | `OptionsClass()` |

### `optimize(bot, **kwargs)`

Runs the optimization. `bot` is a `BaseBot` subclass instance. Extra `**kwargs` pass through to `backtest()`. Returns a dict mapping metric names to `(score_dict, bot)` tuples, or `None` for `MouseWheelTuner`.

## Optimizers

### `QPSO` — Quantum Particle Swarm Optimization

Particle-swarm optimizer. Particles mutate across epochs with neuroplastic memory and cyclic simulated annealing.

```python
from qtradex.optimizers import QPSO, QPSOoptions

opts = QPSOoptions()
opt = QPSO(data, options=opts)
best = opt.optimize(bot)
```

**`QPSOoptions` parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `epochs` | `float` | `inf` | Max iterations |
| `improvements` | `int` | 100000 | Stop after this many score improvements |
| `cooldown` | `int` | 0 | Iterations after an improvement before checking improvement limit |
| `lag` | `float` | 0.5 | Feed-back lag positions for score trends |
| `top_percent` | `float` | 0.9 | Top fraction of candidates to keep in plot |
| `plot_period` | `int` | 100 | Iterations between live score plots (0 = never) |
| `fitness_ratios` | `list` or `None` | `None` | Tradeoff between early vs late fitness scores |
| `fitness_period` | `int` | 200 | Split point for fitness ratio scoring |
| `fitness_inversion` | `callable` | rotation lambda | Rotates metric priorities each iteration |
| `cyclic_amplitude` | `float` | 3 | Mutation step oscillation amplitude |
| `cyclic_freq` | `int` | 1000 | Iterations per hot/cold cycle |
| `digress` | `float` | 0.99 | Best-score decay factor (multiply every `digress_freq`) |
| `digress_freq` | `int` | 2500 | Iterations between best-score degradation |
| `temperature` | `float` | 2.0 | Base mutation step size |
| `synapses` | `int` | 50 | Number of successful parameter sets to remember |
| `neurons` | `list` | `[]` | Active parameter indices (empty = all) |
| `show_terminal` | `bool` | True | Print progress to terminal |
| `print_tune` | `bool` | False | Print best tune on completion |

---

### `LSGA` — Local Search Genetic Algorithm

Genetic algorithm that inherits QPSO internals. Adds a population, crossover, walk-forward consistency gate, drawdown gate, directional synapses, and optional momentum / regularization.

```python
from qtradex.optimizers import LSGA, LSGAoptions

opts = LSGAoptions()
opt = LSGA(data, options=opts)
best = opt.optimize(bot)
```

**`LSGAoptions` parameters** (inherits all `QPSOoptions`, adds):

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `population` | `int` | 20 | Candidates per generation |
| `offspring` | `int` | 10 | Offspring generated per generation |
| `top_ratio` | `float` | 0.20 | Fraction kept as elite (higher = more diversity) |
| `processes` | `int` | `cpu_count() or 3` | Parallel workers |
| `erode` | `float` | 0.9999 | Candidate erosion rate |
| `erode_freq` | `int` | 200 | Erosion frequency in iterations |
| `append_tune` | `str` | `""` | File path to write tunes to |
| `skew_check_period` | `int` | 2 | Skew-memory check interval (set high to disable) |
| `skew_mc_iterations` | `int` | 70 | Monte Carlo iterations per skew check |
| `skew_perturbation` | `float` | 0.002 | Perturbation for skew detection |
| `skew_sigma` | `float` | 0.01 | Skew penalty sigma |
| `skew_memory_cap` | `int` | 1000 | Max skew-memory entries |
| `select_data` | `Data`, `None`, or `False` | `None` | Walk-forward consistency gate. `None` = auto-split (2/3 train + 1/3 select), `Data` = explicit select set, `False` = disable |
| `consistency_fn` | `callable` or `None` | `None` | Walk-forward consistency function |
| `consistency_target` | `float` | 1.5 | Target train/select ADR ratio at full intensity |
| `reg_penalty` | `float` | 0.0 | Ridge penalty for params near clamp edges (e.g. 0.15 = up to 15% penalty at edge) |
| `acceptance_temp` | `float` | 0.0 | Stochastic acceptance temperature. 0 = deterministic; >0 = probabilistic (filters noise) |
| `momentum_decay` | `float` | 0.0 | Per-parameter Adam-style momentum. 0 = no momentum; 0.9 = smooth mutation directions across generations |

Overridden defaults from QPSOoptions: `fitness_period=20`, `cyclic_freq=25`, `improvements=10000`, `temperature=1`.

The result dict includes `wf_culled`, `wf_survived`, and `wf_intensity` keys with walk-through gate metadata. Synapses are now directional and weighted by success score — frequently successful parameter combos are preferred during replay, and mutations are biased toward previously successful directions.

---

### `GridSearch` — Random Subspace Grid Search

Samples random 2- or 3-parameter subspaces exhaustively. Good for exploring the parameter landscape without gradient-following.

```python
from qtradex.optimizers import GridSearch, GridSearchOptions

opts = GridSearchOptions()
opt = GridSearch(data, options=opts)
best = opt.optimize(bot)
```

**`GridSearchOptions` parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `iterations` | `int` | 50 | Number of random subspaces to try |
| `grid_dims` | `int` | 2 | Params per subspace (2 or 3) |
| `grid_points` | `int` | 10 | Points per axis. Total evaluations = `grid_points^grid_dims` (100 for 2D, 1000 for 3D) |
| `grid_margin` | `float` | 0.0 | Fraction to trim from each clamp edge. 0.15 restricts search to interior 70% of range |
| `processes` | `int` | `cpu_count() or 4` | Parallel workers |
| `show_terminal` | `bool` | True | Print progress to terminal |
| `print_tune` | `bool` | False | Print best tune on completion |
| `epochs` | `float` | `inf` | Max iterations |
| `improvements` | `int` | `inf` | Stop after this many improvements |

---

### `RLPPO` — Proximal Policy Optimization (optional)

Reinforcement learning optimizer using PPO (stable-baselines3). Optional dependency: `pip install qtradex[rl]`. Falls back to random search if not installed.

```python
from qtradex.optimizers import RLPPO, RLPPOoptions

opts = RLPPOoptions()
opt = RLPPO(data, options=opts)
best = opt.optimize(bot)
```

**`RLPPOoptions` parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `total_timesteps` | `int` | 50000 | Total backtests across training |
| `learning_rate` | `float` | 3e-4 | PPO learning rate |
| `gamma` | `float` | 0.99 | Discount factor |
| `gae_lambda` | `float` | 0.95 | GAE lambda for advantage estimation |
| `clip_range` | `float` | 0.2 | PPO clip range |
| `ent_coef` | `float` | 0.01 | Entropy bonus (higher = more exploration) |
| `n_steps` | `int` | 512 | Steps per PPO update |
| `batch_size` | `int` | 64 | PPO batch size |
| `n_epochs` | `int` | 10 | Epochs per PPO update |
| `walk_forward` | `bool` | True | Split data 2/3 + 1/3; reward = 0.7*train + 0.3*val composite |
| `verbose` | `int` | 1 | SB3 verbosity (0 = silent) |
| `show_terminal` | `bool` | True | Print progress to terminal |
| `print_tune` | `bool` | False | Print best tune on completion |
| `select_data` | `bool` | `False` | No walk-forward split (use `walk_forward` instead) |

The reward function uses a composite score: `sortino * (1 - max_dd) * min(1, trades/30)`. The training environment is a Gymnasium wrapper around the backtest loop.

---

### `IPSE` — Iterative Parametric Space Expansion

Expands the search space outward from profitable regions. Brute-forces one parameter at a time via linear sweep.

```python
from qtradex.optimizers import IPSE, IPSEoptions

opts = IPSEoptions()
opt = IPSE(data, options=opts)
best = opt.optimize(bot)
```

**`IPSEoptions` parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `acceleration` | `float` | 0.8 | Space shrinking rate (lower = faster shrink) |
| `space_size` | `int` | 25 | Candidates per parameter per sweep |
| `processes` | `int` | `cpu_count()` | Parallel workers |
| `show_terminal` | `bool` | True | Print progress to terminal |
| `print_tune` | `bool` | False | Print best tune on completion |

---

### `AION` — Adaptive Intelligent Optimization Network

Multi-agent optimizer with separate Mutator, Filter, Evaluator, and Learner agents sharing state through `OptState`. Uses quantum tunneling, elite preservation, smart skip, and bad-region memory.

```python
from qtradex.optimizers import AION, AIONoptions

opts = AIONoptions()
opt = AION(data, options=opts)
best = opt.optimize(bot)
```

**`AIONoptions` parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `epochs` | `float` | `inf` | Max iterations |
| `improvements` | `int` | 100000 | Stop after this many improvements |
| `cooldown` | `int` | 0 | Iterations after an improvement before checking limit |
| `show_terminal` | `bool` | True | Print progress to terminal |
| `print_tune` | `bool` | True | Print best tune on completion |
| `plot_period` | `int` | 100 | Plot interval |
| `quantum_tunneling_prob` | `float` | 0.05 | Probability of 5x escape jump |
| `min_temperature` | `float` | 0.05 | Minimum mutation temperature |
| `max_temperature` | `float` | 3.0 | Maximum mutation temperature |
| `synapses` | `int` | 50 | Successful parameter sets to remember |
| `neurons` | `list` | `[]` | Active parameter indices (empty = all) |
| `fitness_ratios` | `list` or `None` | `None` | Fit / out-of-fit scoring ratio |
| `enable_cache` | `bool` | True | Cache evaluated (tune → score) pairs |
| `elite_preservation` | `int` | 3 | Number of top candidates preserved per epoch |
| `smart_skip_threshold` | `int` | 10 | Max consecutive skips before forced exploration |
| `bad_region_memory` | `int` | 50 | Max bad-region entries to track |

---

### `MouseWheelTuner` — Interactive GUI

Launches a Tkinter window with scrollable knobs for each tune parameter. No options class.

```python
from qtradex.optimizers import MouseWheelTuner

opt = MouseWheelTuner(data, wallet)
opt.optimize(bot)  # Blocks until window is closed
```

No return value. Manual tuning via the GUI only.

## Result dict

`opt.optimize()` returns `dict[str, tuple[dict, BaseBot]]` mapping metric names to results:

```python
result["roi"]       # -> (score_dict, best_bot_for_roi)
result["sharpe"]    # -> (score_dict, best_bot_for_sharpe)
# ... per metric
```

Each `score_dict` has the metric values for that bot's backtest. Each `best_bot` is a fresh bot instance with the optimal `.tune` set.

`MouseWheelTuner` returns `None`.
