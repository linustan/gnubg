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

### 4. Formulate user question
- Asked user which settings they've changed from defaults
- Their answer determines which profile to optimize for

## Blocked / Waiting

- **User response needed**: Which rollout settings have they modified?
  (chequer ply, cube ply, VR, truncation, trials)

## Next Steps (pending user input)

- Profile the actual hot path based on user's configuration
- Identify specific optimization targets (NN forward pass, VR loop, cache, etc.)
- Prototype and benchmark improvements
