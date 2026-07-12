# Optimization

A bot starts with your best guesses — tune values you picked by eye. Optimization automates the search for better ones.

Every optimizer follows the same loop: tweak the tune, run a backtest, keep results that improve your metrics, repeat. They differ in *how* they tweak, *how fast* they find good regions, and *how* they avoid getting stuck.

## Which optimizer should you use?

||QPSO|LSGA|IPSE|AION|GridSearch|RLPPO|MouseWheel|
|---|---|---|---|---|---|---|---|
|Best for|First pass|Overfit prevention|Deterministic search|Expensive backtests|Landscape exploration|RL-based search|Final polish|
|Threading|Single|Parallel|Parallel|Single|Parallel|Single|Single|
|Overfit protection|No|Walk-forward + DD gate + skew|No|No|No|Walk-forward composite|N/A|
|Requires install|—|—|—|—|—|`pip install qtradex[rl]`|`tkinter`|
|Population|No|Yes|Yes|No|Yes|No|N/A|

Quick rule of thumb: start with **QPSO**. If you see overfitting, switch to **LSGA**. If you want predictable, repeatable results, use **IPSE**. If backtests are slow and you want efficiency, run **AION**. To map the landscape coarsely, try **GridSearch**. For experimental RL-based search, try **RLPPO**. Finish with **MouseWheelTuner** for hand-tuning.

---

## QPSO — Quantum Particle Swarm Optimization

### When to use

First pass on any bot. QPSO is the simplest optimizer and will find improvements on almost any strategy. Run it first, see what it finds, then switch to a more targeted optimizer if you need more.

### How it explores

The mutation step size oscillates between small and large on a sine wave — cyclic simulated annealing. When the wave is high, the optimizer takes big jumps; when low, it refines locally. The cycle repeats forever rather than cooling monotonically, so the optimizer never locks into pure exploitation.

### Why more than one metric

QPSO doesn't optimize for a single number. It maintains a separate best bot for every evaluation metric in the backtest output — ROI, Sharpe ratio, maximum drawdown, win rate, and any others. Each iteration picks a metric to optimize using weighted random selection, and the weights rotate so no single metric dominates forever.

### Learning from past improvements

When a parameter change produces an improvement, QPSO remembers which parameters were changed. Future mutations can replay those same parameter combinations. The memory prunes to the most recent 50 successes.

### Why old bests get stale

Every 2500 iterations, all best scores are multiplied by 0.99. They degrade over time, making it progressively easier to accept new candidates. A solution that looked unbeatable 10,000 iterations ago is now easier to replace.

**Key options:**

| Option | Default | What it controls |
|--------|---------|-----------------|
| `temperature` | 2 | Base step size. Higher = bigger jumps. |
| `cyclic_amplitude` | 3 | How much the step size oscillates |
| `cyclic_freq` | 1000 | How many iterations between hot/cold cycles |
| `epochs` | ∞ | Max iterations |
| `improvements` | 100000 | Stop after this many improvements |
| `synapses` | 50 | How many successful parameter sets to remember |

---

## LSGA — Local Search Genetic Algorithm

### When to use

LSGA is QPSO with guardrails against overfitting. Use it when you suspect your strategy is fitting to historical noise rather than real patterns. The walk-forward gate, drawdown gate, and skew memory give it a built-in reality check that QPSO lacks. It's also good when you have multiple cores — the population evaluation can run in parallel.

### What's added

**Population and crossover.** Each generation starts from the best bot for the current target metric, creates a population of mutated clones, evaluates them in parallel, picks the elite, and blends them via crossover (weighted averaging of parameters).

**Walk-forward consistency.** The data is split 2/3 for training and 1/3 for selection. Every candidate is backtested on both slices. If the training performance is much better than the selection performance, the candidate is culled — it's overfit. The gate relaxes for low-ROI strategies so they're not unfairly eliminated.

**Drawdown gate.** Built into the consistency check. Candidates with excessive drawdown (above an annealed threshold that starts at 80% and tightens to ~35% as candidates improve) on either the train or select slice are culled.

**Directional synapses.** When LSGA records a successful parameter mutation, it doesn't just remember which parameters were changed — it remembers which direction and how often that combination succeeded. Future mutations are biased toward those directions, and frequently successful combinations are preferred during synapse replay.

**Skew memory.** When LSGA finds an improvement, it runs a local sanity check: it perturbs the winning parameters slightly, runs ~70 backtests, and measures the skew of the return distribution. If the region has suspiciously low skew, that region is marked with a penalty. Future candidates near it get their scores suppressed, pushing exploration elsewhere.

**Parameter regularization.** Optional ridge penalty (`reg_penalty`) pushes parameters away from clamp boundaries, where overfitting often concentrates. At 0.15, the penalty reduces scores by up to 15% for parameters at their clamp edges.

**Stochastic acceptance.** Optional (`acceptance_temp`). Instead of always accepting an improvement, accept it probabilistically. Small improvements get rejected, preventing LSGA from polishing noise.

**Momentum.** Optional (`momentum_decay`). Per-parameter Adam-style momentum tracking smooths mutation directions across generations, reducing oscillation in noisy landscapes.

### Key differences from QPSO

| Aspect | QPSO | LSGA |
|--------|------|------|
| Candidates per iteration | 1 | population (default 20) |
| Crossover | No | Yes (blend crossover) |
| Walk-forward gate | No | Yes (train/select split) |
| Drawdown gate | No | Yes (annealed DD culling) |
| Directional synapses | No | Yes (weighted, directional) |
| Anti-overfit memory | No | Yes (skew memory) |
| Regularization | No | Yes (clamp-boundary penalty) |
| Stochastic acceptance | No | Yes (noise filter) |
| Momentum | No | Yes (per-parameter) |
| Parallel workers | No | Yes (multiprocessing) |

### Quick settings reference

```python
from qtradex.optimizers import LSGA, LSGAoptions

opts = LSGAoptions()

# Walk-forward gate (default: on, auto-split)
opts.select_data = False        # disable gate (use full data for training)
opts.consistency_target = 1.5   # tighten train/select ratio

# Greediness control
opts.temperature = 0.5          # smaller mutation steps
opts.acceptance_temp = 0.1       # probabilistic filter: 5% improvement = 50% chance, 1% = 10%
opts.reg_penalty = 0.15         # penalize params near clamp edges

# Generalization
opts.momentum_decay = 0.0       # 0.9 for Adam-style momentum
opts.top_ratio = 0.20           # higher = more diversity (default)
```

---

## IPSE — Iterative Parametric Space Expansion

### When to use

IPSE is for when you want complete control and predictability. Same starting point always produces the same result — no randomness. It's also the best choice when you have few parameters (2-4) where interactions are limited, or when you want to understand how each parameter affects performance in isolation.

### How it works

Instead of random mutations, IPSE brute-forces one parameter at a time. For each tunable parameter, it generates a linear sweep of candidate values (default 25) across its range. It backtests all 25 in parallel, picks the value that best improves the current target metric, and moves on to the next parameter. Then it repeats.

After each full pass, IPSE shrinks each parameter's search range toward the current best value:

```
new_min = 0.8 * old_min + 0.2 * current_value
new_max = 0.8 * old_max + 0.2 * current_value
```

This focuses the search window while keeping it wide enough to escape local optima across passes.

### Why this works

Coordinate descent with range shrinking is predictable. The linear sweep gives you a complete picture of how each parameter affects performance in isolation, and multiprocessing makes the sweeps fast.

The tradeoff: IPSE can miss interactions between parameters. If `fast_ema` and `slow_ema` are only good together at specific combinations, sweeping them individually might never find the pair.

**Key options:**

| Option | Default | What it controls |
|--------|---------|-----------------|
| `space_size` | 25 | Candidates per parameter per sweep |
| `acceleration` | 0.8 | Range shrinking speed (higher = slower) |

---

## AION — Adaptive Intelligent Optimization Network

### When to use

AION is the best choice when backtests are slow or noisy and you want to maximize the number of unique parameter combinations evaluated per hour. It also handles high-dimensional parameter spaces (10+ parameters) better than the other optimizers.

### The problem it solves

Most optimizers evaluate every candidate they generate. If a candidate lands in a region you already know is bad, that backtest is wasted compute. AION learns which regions are bad and skips them.

### How it works

AION runs as a pipeline of cooperating agents, each focused on one job:

```
STATE → MUTATOR → FILTER → EVALUATOR → LEARNER → STATE (loop)
```

**Mutator** generates candidate parameter sets using Lévy flights — heavy-tailed random walks that make both small refinements and large jumps. It biases steps based on gradient memory: if a parameter has been moving in a successful direction, it keeps going that way. A small probability (5%) multiplies the step by 5x to escape local optima in one bound.

**Filter** decides whether to skip a backtest without running it. It divides each parameter's range into bins and tracks which bins have historically produced poor results. If a candidate lands in enough bad bins, the Filter skips it.

**Evaluator** runs the actual backtest — or returns a cached result if the same tune has been tested before.

**Learner** updates everything after each evaluation: gradient directions, elite pool, synapse combinations, neuron importance scores, and the parameter history the Filter uses.

### The smart skip

By learning which parameter regions are bad and skipping them, AION evaluates far more total candidates than the other optimizers in the same wall-clock time. Safety valves prevent it from over-skipping: a max skip rate of 60%, forced exploration rolls, and emergency resets if it skips too many in a row.

### Single-threaded tradeoff

AION runs one backtest at a time. On a big CPU, LSGA will finish faster in wall-clock time because its population evaluations run in parallel. But AION uses less total CPU time because its filter skips so many evaluations. If you have 20+ cores and the backtest is fast, LSGA is likely faster end-to-end. If compute resources are limited or the backtest is expensive, AION's efficiency wins.

---

## GridSearch — Random Subspace Grid Search

`pip install qtradex` (included by default).

### The problem it solves

The other optimizers follow gradients. QPSO mutates toward better regions. LSGA breeds better populations. IPSE climbs coordinate by coordinate. They all exploit what they've learned so far, which means they can get stuck — not in a local optimum (they're good at escaping those), but in a *path dependence*. The final parameters depend on where they started and which random mutations happened to pay off first.

GridSearch doesn't follow a path. It samples.

### How it works

You have N tunable parameters. Each iteration picks a random 2 or 3 of them — say `roc_period` and `roc_threshold` — and grids them exhaustively. For each parameter, it generates `grid_points` evenly spaced values across the clamp range. If `grid_points=10` and the subspace is 2D, that's 100 points. All 100 backtests run in parallel. The best point becomes the new baseline for the other (non-gridded) parameters.

Next iteration picks a different random subspace — maybe `atr_multiplier` and `position_pct` — and repeats.

Over enough iterations, every pair or triple of parameters gets explored in combination with everything else held at their best known values. The result isn't guaranteed to be the global optimum, but it's not path-dependent. Two runs with the same settings produce the same result.

### Grid margin

By default, the grid spans the full clamp range: from `lo` to `hi` inclusive. The edges of a parameter's range are where overfitting concentrates — LSGA consistently pushes `position_pct` to 1.0 because more risk produces more return in-sample. If you know the edges are dangerous, you can trim them.

`grid_margin` shrinks the effective range on both sides by a fraction of the span. At `grid_margin=0.15`, a parameter with clamps `[0.3, 1.0]` gets searched from `0.3 + 0.15*0.7 = 0.405` to `1.0 - 0.15*0.7 = 0.895`. The interior 70%. The margin is the same for all params — it's a fraction, not an absolute value, so it scales with range width.

### Two-stage pipeline

GridSearch explores. LSGA exploits. They complement each other.

The pattern: run GridSearch for 20-30 iterations to find a promising neighborhood, then feed the best result into LSGA with a low temperature (0.3) for refinement. The grid finds the broad region, LSGA climbs to the peak within it. This consistently finds different (not always better) solutions than LSGA alone.

```python
from qtradex.optimizers import GridSearch, GridSearchOptions
from qtradex.optimizers import LSGA, LSGAoptions

# Stage 1: explore
gs_opts = GridSearchOptions()
gs_opts.iterations = 20
gs_opts.grid_margin = 0.15
grid = GridSearch(data, wallet, gs_opts)
grid_bots = grid.optimize(bot)

# Stage 2: refine
best_grid_tune = grid_bots["sortino_ratio"][1].tune
refine_bot = deepcopy(bot)
refine_bot.tune = best_grid_tune

lsga_opts = LSGAoptions()
lsga_opts.temperature = 0.3
lsga = LSGA(data, wallet, lsga_opts)
final = lsga.optimize(refine_bot)
```

### Key options

| Option | Default | What it controls |
|--------|---------|-----------------|
| `iterations` | 50 | Number of random subspaces to try. With 7 parameters and 2-param subspaces, 20 iterations covers most pairs once. |
| `grid_dims` | 2 | Params per subspace. 2 = 100 points/iteration, 3 = 1000 points/iteration. 3 is more thorough but slower. |
| `grid_points` | 10 | Points per axis. 10 = coarse, 20 = finer but 4x more backtests. With margin=0.15 and 10 points, step size is ~7% of range. |
| `grid_margin` | 0.0 | Fraction trimmed from each clamp edge. 0.15 = interior 70% only. Keeps search away from boundary overfitting. |

---

## RLPPO — Proximal Policy Optimization (experimental)

Optional dependency: `pip install qtradex[rl]` or `pip install stable-baselines3 gymnasium`.

### Why reinforcement learning?

Every optimizer in QTradeX so far uses hand-crafted search rules. Mutate randomly. Breed the best. Sweep linearly. These rules work because they encode human intuition about what makes a good search: explore when stuck, exploit when ahead, don't revisit bad regions.

But hand-crafted rules have a ceiling. They can't learn. If LSGA's greedy improvement strategy is suboptimal for your specific parameter landscape, it will never discover a better strategy — it will just keep doing greedy improvement.

RLPPO replaces hand-crafted rules with a neural network that learns *how to search* through trial and error.

### How it works

The optimizer wraps the backtest loop as a Gymnasium environment:

```
obs = normalized tune params + recent performance
action = delta for each tune parameter (continuous, in [-1, 1])
reward = composite score (sortino * (1 - max_dd) * min(1, trades/30))
```

Proximal Policy Optimization (PPO) trains a policy network over thousands of episodes. Each episode is one backtest. The policy observes the current parameter values and outputs a delta: increase `roc_period` by 0.3, decrease `roc_threshold` by 0.02, keep everything else. The environment applies the delta, runs the backtest, and returns the composite score as reward.

Over time, the policy learns which parameter changes tend to produce higher composite scores. It learns interactions: "when `roc_period` is low, a high `roc_threshold` works better" — patterns that hard-coded rules might miss.

### Walk-forward composite

The problem with optimizing on a single data slice: the policy learns to memorize. It finds parameters that work on the training window but collapse out of sample — the same overfitting pattern every optimizer exhibits.

RLPPO addresses this by default with a walk-forward split. The data is divided 2/3 for training and 1/3 for validation. Every episode backtests on both slices independently. The reward is:

```
reward = 0.7 * composite(train_slice) + 0.3 * composite(val_slice)
```

The policy must perform on both slices. It can't maximize the training score at the expense of the validation score — that would reduce the weighted reward. The 70/30 split keeps the primary signal on the training data while maintaining pressure toward generalization.

### Where it stands

RLPPO works. It trains successfully — the policy converges, parameters improve, the reward increases over 50,000 timesteps. It found the correct `roc_period=5, roc_threshold=0.1` sweet spot on Pure Momentum in testing.

But it hasn't beaten LSGA yet. The same overfitting problem that plagues every optimizer also plagues RLPPO — the policy learns to widen stops and extend hold times to maximize in-sample returns, then fails on unseen data. The walk-forward composite helps but doesn't eliminate the gap.

The ceiling isn't the RL approach. It's the reward function. The current composite (`sortino * (1 - DD) * trade_mult`) is the same objective every other optimizer uses, so RL has no information advantage. A better reward — one that directly encodes walk-forward consistency or risk-adjusted return — would change that.

### Key options

| Option | Default | What it controls |
|--------|---------|-----------------|
| `total_timesteps` | 50000 | Total backtests across training. More = better policy, but each backtest takes wall-clock time. |
| `walk_forward` | True | Split data 2/3 + 1/3 for train/val composite reward. Disable to train on full data (risks overfit). |
| `learning_rate` | 3e-4 | PPO learning rate. Standard Adam default. Lower = more stable but slower. |
| `n_steps` | 512 | Steps collected before each PPO update. Higher = more stable gradients. |
| `batch_size` | 64 | Minibatch size for PPO updates. |
| `ent_coef` | 0.01 | Entropy bonus. Higher = more exploration. Lower = more exploitation. |

---

## MouseWheelTuner — Manual fine-tuning by hand

### When to use

None of the above optimizers will get you all the way. They'll find a good region, but the final polish is often best done by hand. The MouseWheelTuner is also useful for sensitivity analysis — scroll a parameter and watch the metrics change in real time.

### How it works

It opens a tkinter window with one knob per tune parameter. Scroll your mouse wheel over a knob to nudge the value up or down (hold Shift for finer control). On every scroll, a backtest runs in the background and results appear as formatted JSON.

Undo and redo have full history stacks. When you find a set you like, name it and hit Save — it writes the tune to a file in your bot's `tunes/` directory.

---

> Tune caching (saving, loading, the JSON format, the CLI, and the interactive dispatch menu) has its own page at [Tune Management](tune-management.md).
