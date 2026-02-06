# Optimization Findings

## Rollout Cost Model

The Monte Carlo rollout (`BasicCubefulRollout()` in `rollout.c:317`) is the
primary optimization target. Each rollout's total cost is:

```
total_cost = nTrials x avg_turns_per_game x per_turn_cost
```

Where per-turn cost breaks down as:

| Component | Cost driver | Notes |
|-----------|-------------|-------|
| Move generation | `GenerateMoves()` combinatorial enumeration | Up to 3060 candidates, typically 10-30 |
| Move scoring | `n_candidates x eval_cost(chequer_ply)` | Dominant term at higher ply |
| Cube decisions | `FindCubeDecision()` at `cube_ply` depth | Every turn when cube is available |
| Variance reduction | `36 x (move_gen + best_move_eval)` per turn | When enabled, can dominate total cost |
| Board bookkeeping | `SwapSides()`, `PositionKey()`, copies | Cheap individually, called millions of times |

### Evaluation cost expands recursively with ply

```
eval_cost(0) = 1 NN forward pass
eval_cost(k) = 36 x (move_gen + n_filtered_moves x eval_cost(k-1))
```

### NN forward pass

The leaf cost is a matrix-vector multiply in `Evaluate()` (`lib/neuralnet.c:125`):
~128-256 hidden neurons x ~250 inputs. This is the single most-called expensive
operation.

### Cache behavior in rollouts

The evaluation cache (`EvaluatePositionCache`) has lower hit rates during
rollouts than during deterministic n-ply search, because each trial follows a
unique game trajectory. Cache still helps within a single trial's sub-searches.

## Cost Profiles by Configuration

### Profile A: Default settings (0-ply chequer, VR on)

Variance reduction dominates. Each turn evaluates all 36 possible dice outcomes
as a side computation. The actual move selection (0-ply NN per candidate) is
cheap by comparison. Optimization target: the VR loop and NN forward pass.

### Profile B: Higher chequer ply (1-2 ply, VR on or off)

Move scoring dominates. Each candidate move triggers a recursive search over 36
dice outcomes. Move filtering becomes critical for pruning. Optimization target:
move scoring pipeline, move filtering efficiency, cache hit rates.

### Profile C: Truncation ON (ply ~10)

Games are shorter (~10 turns vs 50+), reducing per-trial cost. Truncation
evaluation accuracy matters more since NN equity values are used directly for
unfinished games. Optimization target depends on Profile A vs B within the
truncated game length.

## Why NN Evaluation Inside Rollouts

The rollout does not use nested rollouts. The NN (with optional n-ply lookahead)
selects moves during simulated games. The rollout's equity estimate comes from
game outcomes (wins/losses/gammons), not from NN equity values. The NN only needs
correct move *ranking*, not accurate absolute equity, except at truncation points
and for variance reduction corrections.

Monte Carlo is particularly effective for backgammon because dice randomness makes
exhaustive deep search intractable (branching factor = moves x 36 per ply).
Rollouts sample dice sequences and play forward to game end, converging to the
true expectation over many trials.

## Default Rollout Settings

| Setting | Default | Source |
|---------|---------|--------|
| Chequer ply | 0 | `gnubg.c:289` |
| Cube ply | 2 | `gnubg.c:287` |
| Variance reduction | ON | `gnubg.c:313` |
| Truncation | OFF (`fDoTruncate`) | `gnubg.c:321` |
| Truncation ply | 10 | `gnubg.c:319` |
| Trials | 1296 | `gnubg.c:317` |
| Quasi-random dice | ON | `gnubg.c:315` |
| Cubeful | ON | `gnubg.c:311` |
| Late evals | OFF | `gnubg.c:323` |

## Baseline Configuration: Rollout E (Medium Rollout)

The user's target configuration for optimization. Full details in
[optimization-docs/baseline-config.md](optimization-docs/baseline-config.md).

Key settings: **360 trials, truncate at 7-ply, 1-ply chequer play, VR on,
quasi-random dice, late evals after 2 plays (cube drops to 1-ply)**.

This is a **Profile B + C hybrid**: 1-ply chequer play makes move scoring
dominant (~36x more per move than 0-ply default), while 7-ply truncation
keeps games short (~7 turns vs 50+). Primary optimization targets:

1. NN forward pass (highest volume leaf operation)
2. Move scoring pipeline (1-ply recursive eval per candidate)
3. Variance reduction loop (36 x 1-ply eval per turn)
4. Cache hit rates (more sub-evaluations = more cache value)

## Open Questions

(None currently — baseline configuration established.)

## Detailed Analysis

See [optimization-docs/](optimization-docs/) for detailed breakdowns:
- [rollout-cost-model.md](optimization-docs/rollout-cost-model.md) - Full cost model derivation
- [baseline-config.md](optimization-docs/baseline-config.md) - Baseline configuration and cost analysis
