# Rollout Cost Model - Detailed Derivation

## Overview

This document captures the full cost model for `BasicCubefulRollout()`
(`rollout.c:317-830`), the core Monte Carlo simulation loop in gnubg.

## Per-Trial Execution Flow

Each of the `nTrials` simulated games executes:

```
for each turn (while game not over):
    1. Cube decision    -> FindCubeDecision() at aecCube[].nPlies
    2. Dice roll        -> RolloutDice() (quasi-random or pure random)
    3. Move selection   -> FindBestMove() at aecChequer[].nPlies
    4. Variance reduction (if enabled):
       -> Evaluate all 36 dice outcomes at reduced ply
       -> Compute mean, track delta for correction
    5. Game-over check  -> If CLASS_OVER, record outcome
    6. Board swap       -> SwapSides(), update cube owner
```

## Cost Breakdown by Component

### Move Generation: `GenerateMoves()` (eval.c:2878)

- Recursive depth-first search via `GenerateMovesSub()` (eval.c:2788)
- For non-doubles: called twice (swapped dice order), deduplicated
- Cost: O(branching^depth) where depth=2 (normal) or 4 (doubles)
- Typical output: 10-30 unique moves; max theoretical: 3060

### Move Scoring: `ScoreMoves()` (eval.c:5578)

Per candidate move:
1. `PositionFromKey()` - reconstruct board
2. `SwapSides()` - opponent perspective
3. `GeneralEvaluationEPlied(nPlies)` - recursive eval
4. `InvertEvaluationR()` - flip back

At 0-ply: 1 NN eval per candidate
At k-ply: 36 x (move_gen + n_filtered x eval(k-1)) per candidate

Move filtering (`FindnSaveBestMoves`): multiple passes at increasing ply,
pruning between passes. Default filter: accept 8 moves at threshold 0.16.

### Cube Decision: `FindCubeDecision()`

Evaluates no-double vs double/take vs double/pass. Uses `aecCube[].nPlies`
(default: 2-ply). Called every turn when cube is available in match play.

### Variance Reduction (rollout.c:575-664)

When `fVarRedn=1`, every turn:
- Loop over all 36 dice combinations (21 unique, weighted)
- For each: `FindBestMove()` + evaluate at reduced ply
- Compute weighted mean across all 36
- Track cumulative delta: actual_outcome - mean

This multiplies per-turn cost by roughly 36x (for the move-selection component).
The payoff: dramatically fewer trials needed for the same confidence interval.

### NN Forward Pass: `Evaluate()` (lib/neuralnet.c:125)

The leaf operation. Two matrix-vector multiplies:
1. Hidden layer: W_h (cHidden x cInput) . input + bias -> sigmoid activation
2. Output layer: W_o (cOutput x cHidden) . hidden + bias -> sigmoid activation

Typical sizes: cInput ~250, cHidden ~128-256, cOutput ~5.
Uses lookup-table sigmoid (lib/sigmoid.h:141) for speed.

`EvaluateFromBase()` (neuralnet.c:174) provides incremental evaluation reusing
cached hidden layer activations when only a few inputs change.

### Cache: `EvaluatePositionCache()` (eval.c:5497)

- Hash: `PositionKey()` -> 28-byte key
- Lookup in hash table (2^19 or 2^23 entries)
- Hit: return cached arOutput[5], skip NN
- Miss: call `EvaluatePositionFull()`, store result
- Rollout cache hit rates are lower than deterministic search

## Total Cost Formula

```
total = nTrials x avg_turns x (
    move_gen_cost
  + n_candidates x eval_cost(chequer_ply)
  + cube_eval_cost(cube_ply)
  + vr_enabled * 36 x (move_gen_cost + eval_cost(vr_ply))
)

where:
  eval_cost(0) = cache_lookup + (1 - hit_rate) x NN_forward_pass
  eval_cost(k) = 36 x (move_gen + n_filtered x eval_cost(k-1))
```
