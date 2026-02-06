# Optimization Progress

> Track of completed tasks, open questions, and next steps.
> Detailed analysis lives in [optimization-docs/](optimization-docs/).

## Completed

### 1. Understand rollout architecture
- Confirmed rollouts do NOT use nested rollouts for move selection
- NN evaluation (with optional n-ply lookahead) selects moves inside each trial
- NN only needs correct move *ranking*; absolute equity accuracy only matters
  at truncation points and for variance reduction
- Monte Carlo is effective for backgammon specifically because dice make
  exhaustive deep search intractable (branching factor = moves x 36 per ply)

### 2. Derive rollout cost model
- Full cost formula: `nTrials x avg_turns x per_turn_cost`
- Identified 5 per-turn cost components (move gen, scoring, cube, VR, bookkeeping)
- Recursive ply expansion: `eval_cost(k) = 36 x (move_gen + n_filtered x eval_cost(k-1))`
- Documented in [FINDINGS.md](FINDINGS.md) and
  [optimization-docs/rollout-cost-model.md](optimization-docs/rollout-cost-model.md)

### 3. Identify configuration-dependent cost profiles
- Profile A (default: 0-ply, VR on): VR loop dominates
- Profile B (higher ply): move scoring tree dominates
- Profile C (truncation on): shorter games, truncation accuracy matters
- Extracted all default settings with source file references

### 4. Establish baseline configuration
- Received user's target configuration: "Rollout E" (medium rollout)
- 360 trials, truncate at 7-ply, SE stop at 0.0100 (min 180 trials)
- 1-ply chequer play (early and late), cubeful, VR on, quasi-random dice
- Late evals ON: cube drops from 2-ply to 1-ply after first 2 plays
- Truncation evaluation: 2-ply chequer and cube
- Classified as Profile B + C hybrid (move scoring + truncation)
- Documented in [optimization-docs/baseline-config.md](optimization-docs/baseline-config.md)

## Next Steps

- Profile the actual hot path for the Rollout E configuration
- Benchmark current performance as a quantitative baseline
- Identify specific optimization targets:
  - NN forward pass (highest-volume leaf operation)
  - Move scoring pipeline (1-ply recursive eval per candidate)
  - Variance reduction loop (36 x 1-ply eval per turn)
  - Cache hit rates (more sub-evaluations = more cache value)
- Prototype and benchmark improvements
