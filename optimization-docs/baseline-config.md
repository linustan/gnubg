# Baseline Configuration: Rollout E (Medium Rollout)

This is the reference configuration for evaluating optimization progress.
All performance benchmarks should be measured against this configuration.

## Summary

| Setting | Value |
|---------|-------|
| Trials | 360 |
| Truncation ply | 7 |
| SE stop threshold | 0.0100 |
| Minimum trials (SE) | 180 |
| Bearoff truncation | Exact bearoff database |
| Cubeful | Yes |
| Variance reduction | Yes |
| Quasi-random dice | Yes |

## Evaluation Settings by Game Phase

### First Two Plays (Early)

| Setting | Value |
|---------|-------|
| Checkerplay | 1-ply |
| Cube action | 2-ply |
| Cubeful eval | Yes |
| Neural net pruning | Yes |

### Subsequent Plays (Late)

| Setting | Value |
|---------|-------|
| Checkerplay | 1-ply |
| Cube action | 1-ply |
| Cubeful eval | Yes |
| Neural net pruning | Yes |

### Evaluation at Truncation Point

| Setting | Value |
|---------|-------|
| Checkerplay | 2-ply |
| Cube action | 2-ply |
| Cubeful eval | Yes |
| Neural net pruning | Yes |

## Differences from Code Defaults

Compared to the hardcoded defaults in `gnubg.c:285-337`:

| Setting | Code Default | Rollout E |
|---------|-------------|-----------|
| Trials | 1296 | **360** |
| Truncation | OFF | **ON at 7-ply** |
| Chequer ply (early) | 0 | **1** |
| Chequer ply (late) | 0 | **1** |
| Cube ply (late) | 2 | **1** |
| Late evals | OFF | **ON** (switch after 2 plays) |
| Min trials (SE stop) | 324 | **180** |
| SE stop enabled | OFF (`FALSE`) | **ON** |

Settings unchanged from defaults: cubeful=ON, variance reduction=ON,
quasi-random dice=ON, early cube ply=2, truncation chequer ply=2,
truncation cube ply=2, pruning=ON.

## Cost Profile Analysis

This configuration is a **Profile B + C hybrid**:

- **Profile B** (move scoring dominates): 1-ply chequer play means every
  candidate move triggers a search over 36 dice outcomes at the leaf.
  This is ~36x more expensive per move than the default 0-ply.

- **Profile C** (truncation active): Games truncate at 7 plies (~7 turns),
  much shorter than natural game length (~50+ turns). This reduces per-trial
  cost but makes the truncation evaluation (2-ply) accuracy important.

### Per-Turn Cost Breakdown (Approximate)

The dominant costs per turn are:

1. **Move scoring at 1-ply**: For each candidate move (after filtering, ~8-15),
   evaluate 36 dice outcomes x move_gen + best_move x 0-ply NN eval.
   Cost: `n_filtered x 36 x NN_eval` per turn.

2. **Variance reduction at 1-ply**: 36 dice outcomes evaluated with the same
   1-ply depth as chequer play. Cost: `36 x (move_gen + eval_cost(1))`.
   With 1-ply, VR is expensive but still provides statistical benefit.

3. **Cube decisions**: 2-ply for the first 2 turns, then 1-ply. The 2-ply cube
   evals in early turns are the most expensive individual operations.

4. **Truncation evaluation**: At ply 7, remaining games are evaluated at 2-ply
   for both chequer and cube. This is a one-time cost per trial but at the
   highest ply depth in the configuration.

### Optimization Targets (Priority Order)

1. **NN forward pass** (`lib/neuralnet.c:125`): Called at every leaf of every
   1-ply search, every VR evaluation, and every 2-ply truncation/cube eval.
   This is the single highest-volume operation.

2. **Move scoring pipeline** (`eval.c:5578`): The 1-ply chequer play means
   `ScoreMoves()` is called frequently with nontrivial recursive evaluation.
   Move filtering efficiency directly impacts total cost.

3. **Variance reduction loop** (`rollout.c:575-664`): With 1-ply chequer,
   the VR side computation (36 dice x 1-ply eval) is substantial.

4. **Cache hit rates**: Higher ply means more sub-evaluations, which increases
   the value of cache hits. Improving cache utilization could reduce redundant
   NN evaluations.
