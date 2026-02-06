# GNU Backgammon (gnubg) - Codebase Reference

> **NOTE**: This document serves as the authoritative codebase overview. Consult
> this file first before running explore agents or doing broad codebase searches.
> It maps control flows, data structures, key functions with file/line references,
> and architectural patterns. For targeted lookups, use grep/glob directly against
> the specific files listed here.

## Build System

- **Tool**: GNU Autotools (autoconf/automake)
- **Configure**: `configure.ac`
- **Makefiles**: `Makefile.am` in root and subdirectories
- **Key build targets**: `gnubg` (main executable)
- **Optional features**: GTK+ GUI (`--enable-gtk`), Python scripting, 3D board (OpenGL), libcurl
- **Core dependencies**: GLib 2.x, libreadline (optional), GTK+ 2.x/3.x (optional)
- **Main entry**: `gnubg.c` line 4621 (`main()`)

## Directory Layout

| Path | Purpose |
|------|---------|
| `/` (root) | 72 C source files - core engine, CLI, GUI glue |
| `lib/` | Core library: neural networks, common types, SIMD utilities |
| `board3d/` | 3D OpenGL board rendering |
| `doc/` | Documentation |
| `fonts/`, `textures/`, `pixmaps/` | Graphics assets |

## Critical Files

| File | ~Lines | Purpose |
|------|--------|---------|
| `eval.c` | 6285 | Core board evaluation, move generation/scoring, position classification |
| `rollout.c` | 1875 | Monte Carlo rollout simulation |
| `gnubg.c` | 6500+ | Main program, CLI commands, user-facing decision logic |
| `lib/neuralnet.c` | 560 | Neural network forward pass |
| `backgammon.h` | large | Master header with all key types |
| `eval.h` | 597 | Evaluation data structures and function exports |
| `rollout.h` | 171 | Rollout types and exports |
| `lib/gnubg-types.h` | 73 | Core type definitions (`TanBoard`, `matchstate`) |
| `lib/neuralnet.h` | 39 | Neural network structure definition |
| `lib/sigmoid.h` | 169 | Lookup-table sigmoid activation function |
| `bearoff.c` | large | Bearoff database (1-sided, 2-sided) |
| `positionid.c` | - | Position hashing (`PositionKey`, `PositionFromKey`) |

---

## Key Data Structures

### Board Representation
**File**: `lib/gnubg-types.h:25-26`
```c
typedef unsigned int TanBoard[2][25];            // 2 players x 25 points
typedef const unsigned int (*ConstTanBoard)[25];
```
- Index 0-23: Board points 1-24
- Index 24: Bar (captured checkers)
- `TanBoard[0]`: Player 0's checkers
- `TanBoard[1]`: Player 1's checkers

### Match State
**File**: `lib/gnubg-types.h:42-62`
```c
typedef struct _matchstate {
    TanBoard anBoard;
    unsigned int anDice[2];
    int fTurn;           // Player making next decision
    int fMove;           // Player on roll
    int fCubeOwner;
    int nCube;           // Current cube value
    int nMatchTo;        // Match target (0 = money game)
    int anScore[2];
    int fDoubled;
    int fResigned;
} matchstate;
```

### Position Key (Hash)
**File**: `lib/gnubg-types.h:64-66`
```c
typedef union _positionkey {
    unsigned int data[7];  // 28 bytes = 224 bits for position encoding
} positionkey;
```

### Evaluation Context
**File**: `eval.h:92-100`
```c
typedef struct {
    unsigned int fCubeful:1;      // Cubeful or cubeless evaluation
    unsigned int nPlies:3;        // Lookahead depth (0-4)
    unsigned int fUsePrune:1;     // Use move filtering/pruning
    unsigned int fDeterministic:1;
    float rNoise;                 // Noise std dev (for exploration)
} evalcontext;
```

### Cube Decision Info
**File**: `eval.h:177-194`
```c
typedef struct {
    int nCube, fCubeOwner, fMove, nMatchTo;
    int anScore[2];
    float arGammonPrice[4];       // Gammon/backgammon prices
    bgvariation bgv;
} cubeinfo;
```

### Move Structure
**File**: `eval.h:278-289`
```c
typedef struct {
    int anMove[8];                // Move: src/dest pairs (max 4 dice)
    positionkey key;              // Position after move
    unsigned int cMoves, cPips;   // Checkers/pips moved
    float rScore, rScore2;        // Primary/secondary scores
    float arEvalMove[NUM_ROLLOUT_OUTPUTS];    // 7 evaluation outputs
    float arEvalStdDev[NUM_ROLLOUT_OUTPUTS];
} move;
```

### Neural Network
**File**: `lib/neuralnet.h:27-39`
```c
typedef struct _neuralnet {
    unsigned int cInput;          // Input layer size
    unsigned int cHidden;         // Hidden layer size
    unsigned int cOutput;         // Output layer size
    float rBetaHidden;
    float rBetaOutput;
    float *arHiddenWeight;        // Hidden layer weights
    float *arOutputWeight;        // Output layer weights
    float *arHiddenThreshold;     // Biases
    float *arOutputThreshold;
} neuralnet;
```

### Rollout Context
**File**: `eval.h:114-145`
```c
typedef struct {
    evalcontext aecCube[2];       // Cube eval context per player
    evalcontext aecChequer[2];    // Chequer play eval context per player
    unsigned int fCubeful:1;
    unsigned int fVarRedn:1;      // Variance reduction enabled
    unsigned int nTrials;         // Number of rollout games
    unsigned short nTruncate;     // Truncation ply
    unsigned int nLate;           // Late evaluation switch ply
    rng rngRollout;
    unsigned long nSeed;
} rolloutcontext;
```

### Neural Network Output Format
**File**: `eval.h:46-54`
```c
#define NUM_OUTPUTS 5
#define OUTPUT_WIN              0  // Probability of winning
#define OUTPUT_WINGAMMON        1  // Prob of winning gammon
#define OUTPUT_WINBACKGAMMON    2  // Prob of winning backgammon
#define OUTPUT_LOSEGAMMON       3  // Prob of losing gammon
#define OUTPUT_LOSEBACKGAMMON   4  // Prob of losing backgammon

// Rollout-only additional outputs:
#define NUM_ROLLOUT_OUTPUTS 7
#define OUTPUT_EQUITY           5  // Expected value
#define OUTPUT_CUBEFUL_EQUITY   6  // Match-win-chance (cubeful)
```

---

## Position Classification

Positions are classified into types that determine which evaluator is used.
`ClassifyPosition()` in `eval.c` maps a board to one of:

| Class | Meaning | Evaluator |
|-------|---------|-----------|
| `CLASS_OVER` | Game finished | Direct result |
| `CLASS_BEAROFF1` / `CLASS_BEAROFF_OS` | 1-sided bearoff (in-memory / on-disk) | Bearoff database |
| `CLASS_BEAROFF2` / `CLASS_BEAROFF_TS` | 2-sided bearoff (in-memory / on-disk) | Bearoff database |
| `CLASS_RACE` | No contact, pure pip race | Race neural network |
| `CLASS_CRASHED` | Contact, <7 active checkers | Crashed position NN |
| `CLASS_CONTACT` | General contact play | Contact neural network |
| `CLASS_HYPERGAMMON1/2/3` | Reduced-piece variants | Special databases |

The evaluator dispatch table is `acef[pc]()` indexed by position class.

---

## Evaluation Pipeline (Deterministic / n-Ply)

### Function Call Chain

```
EvaluatePosition()                          [eval.c:5526]
  |
  +-- ClassifyPosition()                    [eval.c]
  +-- EvaluatePositionCache()               [eval.c:5497]
        |
        +-- PositionKey() -> 28-byte hash
        +-- Cache lookup (2^19 or 2^23 entries)
        +-- On MISS: EvaluatePositionFull() [eval.c:5403]
              |
              +-- If nPlies == 0 (leaf):
              |     acef[positionClass]()    -> NN or bearoff DB
              |     Returns arOutput[5]
              |
              +-- If nPlies > 0 (internal node):
                    Loop over 36 dice combinations:
                      +-- FindBestMovePlied() or FindBestMoveInEval()
                      |     +-- GenerateMoves()
                      |     +-- ScoreMoves(nPlies - 1)
                      +-- SwapSides() (opponent perspective)
                      +-- Recurse: EvaluatePositionCache(nPlies - 1)
                    Average results (weight: 1 for doubles, 2 for non-doubles)
                    InvertEvaluationR() (flip Win<->Loss)
```

### Key Functions (Evaluation)

| Function | File:Line | Purpose |
|----------|-----------|---------|
| `EvaluatePosition()` | `eval.c:5526` | Main public API for board evaluation |
| `EvaluatePositionFull()` | `eval.c:5403` | Recursive ply-based evaluation |
| `EvaluatePositionCache()` | `eval.c:5497` | Caching wrapper |
| `GeneralEvaluationE()` | `eval.c` (header) | Cubeful evaluation wrapper |
| `GeneralEvaluationEPlied()` | `eval.c` (header) | Ply-based evaluation with recursion |

### Neural Network Forward Pass

| Function | File:Line | Purpose |
|----------|-----------|---------|
| `NeuralNetEvaluate()` | `lib/neuralnet.c:222` | Main NN entry point |
| `Evaluate()` | `lib/neuralnet.c:125` | Full forward pass (hidden + output layers) |
| `EvaluateFromBase()` | `lib/neuralnet.c:174` | Incremental eval (reuses cached hidden layer) |
| `sigmoid()` | `lib/sigmoid.h:141` | Lookup-table activation function |

The sigmoid uses a precomputed table `e[]` for fast approximation. ~99% of activations hit the fast path.

---

## Move Generation and Scoring

### Move Generation
**File**: `eval.c:2878-2900`

```
GenerateMoves(board, dice_n0, dice_n1)
  +-- Sets up anRoll[0..3] (doubles get 4 dice)
  +-- GenerateMovesSub()                    [eval.c:2788-2875]
  |     Recursive depth-first search (depth 0-3):
  |       +-- Check bar: if on bar, must enter first
  |       +-- For each possible checker move:
  |             LegalMove() validates (blocking rules)
  |             Apply move, recurse for next die
  |       +-- SaveMoves() deduplicates identical final positions
  +-- If n0 != n1: repeat with swapped dice order
```

Maximum legal moves: `MAX_MOVES = 3060`

### Move Scoring

| Function | File:Line | Purpose |
|----------|-----------|---------|
| `FindBestMove()` | `eval.c:5656` | Find single best move (public API) |
| `FindBestMovePlied()` | `eval.c:5619` | Best move with ply lookahead |
| `FindnSaveBestMoves()` | `eval.c:5664` | Score all moves with move filtering |
| `ScoreMove()` | `eval.c:5537` | Evaluate single move |
| `ScoreMoves()` | `eval.c:5578` | Batch evaluate move list |
| `GenerateMoves()` | `eval.c:2878` | Generate all legal moves |
| `GenerateMovesSub()` | `eval.c:2788` | Recursive move enumeration |

`ScoreMove()` flow (`eval.c:5537-5575`):
1. `PositionFromKey()` - reconstruct board after move
2. `SwapSides()` - evaluate from opponent's perspective
3. `GeneralEvaluationEPlied(nPlies)` - recursive evaluation
4. `InvertEvaluationR()` - flip back to current player
5. Set `pm->rScore` = cubeful equity (or cubeless)

Move filtering/pruning: `FindnSaveBestMoves()` applies multiple passes at
increasing ply depth, pruning unlikely candidates between passes.

---

## Monte Carlo Rollout Simulation

### Entry Points

| Function | File:Line | Purpose |
|----------|-----------|---------|
| `GeneralEvaluationR()` | `rollout.c:1560` | Rollout single position |
| `GeneralCubeDecisionR()` | `rollout.c:1629` | Rollout cube decision (take/pass or double/no-double) |
| `RolloutGeneral()` | `rollout.c:1274` | Orchestrate rollout trials |
| `BasicCubefulRollout()` | `rollout.c:317` | Core Monte Carlo game simulator |
| `RolloutDice()` | `rollout.c:214` | Dice sampling (quasi-random or pure random) |

### `BasicCubefulRollout()` Algorithm (`rollout.c:317-830`)

This is the core simulation loop. For each trial game:

```
For each trial (1..nTrials):
  Initialize board copy, cube state, finished flags

  For each turn (while cUnfinished > 0):

    1. CUBE DECISION (lines 414-544):
       +-- ClassifyPosition()
       +-- If match play and cube available:
       |     FindCubeDecision() -> double/take/pass
       |     Track doubles taken/dropped
       +-- Truncation check:
             If 2-sided bearoff -> truncate with GeneralEvaluationE()
             If 1-sided bearoff (cubeless) -> truncate with 0-ply eval
             Mark game finished

    2. CHEQUER PLAY (lines 546-676):
       +-- RolloutDice() -> sample dice
       +-- FindBestMove() -> find optimal play
       +-- VARIANCE REDUCTION (if enabled, lines 575-664):
       |     Evaluate all 36 possible rolls at reduced ply
       |     Compute mean evaluation
       |     Track difference from rolled outcome for correction
       +-- Apply chosen move

    3. GAME OVER CHECK (lines 732-765):
       +-- If CLASS_OVER: evaluate final position
       +-- Convert to cubeful equity
       +-- Record win/gammon/backgammon statistics

    4. BOARD UPDATE (lines 767-774):
       +-- SwapSides() (player 0 <-> 1)
       +-- Update cube owner, player-on-roll
       +-- Increment turn counter

  Truncation evaluation (lines 782-829):
    For unfinished games -> evaluate at truncation ply
    Apply variance reduction correction if enabled
    Normalize by cube value for money games
```

### Dice Sampling (`RolloutDice()`, `rollout.c:214-260`)

Two strategies:
- **Quasi-random** (`fRotate=1`): Deterministic permutations via
  `perArray.aaanPermutation[gen][turn][roll]` for variance reduction
- **Pure random**: ISAAC random number generator

### Variance Reduction

When enabled (`fVarRedn=1`), each turn:
1. Evaluate all 36 possible dice outcomes at reduced ply
2. Compute expected (mean) evaluation
3. Track delta between actual rolled outcome and mean
4. Post-hoc correction reduces estimator variance

---

## User Command -> Evaluation Control Flow

### `hint` Command (Move or Cube)

```
User types "hint"
  |
  CommandHint()                             [gnubg.c:2436]
    +-- Dispatch based on game state:
    |
    +-- MOVE PHASE -> hint_move()           [gnubg.c:2335]
    |     +-- RunAsyncProcess(asyncFindMove)  [threaded]
    |     |     +-- FindnSaveBestMoves()    [eval.c:5664]
    |     |           +-- GenerateMoves()   [eval.c:2878]
    |     |           +-- ScoreMoves() x nPlies [eval.c:5578]
    |     |           |     +-- ScoreMove() per move [eval.c:5537]
    |     |           |           +-- GeneralEvaluationEPlied()
    |     |           |                 +-- EvaluatePositionFull() [recursive]
    |     |           +-- qsort() moves by rScore
    |     +-- Display top N moves with evaluations
    |
    +-- CUBE PHASE -> hint_cube()           [gnubg.c:2068]
    |     +-- RunAsyncProcess(asyncCubeDecision)
    |           +-- GeneralCubeDecision()
    |                 +-- If EVAL_EVAL:
    |                 |     GeneralCubeDecisionE() [deterministic]
    |                 +-- If EVAL_ROLLOUT:
    |                       GeneralCubeDecisionR() [rollout.c:1629]
    |                         +-- RolloutGeneral() [rollout.c:1274]
    |                               +-- BasicCubefulRollout() x nTrials
    |                         +-- FindCubeDecision() -> recommend action
    |
    +-- DOUBLE PHASE -> hint_double()       [gnubg.c:2250]
    +-- TAKE PHASE -> hint_take()           [gnubg.c:2296]
```

---

## Data Flow: Evaluating a Board Position

```
Input: TanBoard[2][25] + cubeinfo + evalcontext
  |
  EvaluatePosition()
    +-- ClassifyPosition() -> e.g. CLASS_CONTACT
    +-- EvaluatePositionCache(nPlies=2)
          +-- PositionKey(board) -> 28-byte hash
          +-- Cache lookup -> HIT: return cached | MISS: continue
          +-- EvaluatePositionFull()
                |
                Ply 2: Loop 36 dice combinations:
                  +-- Copy board -> anBoardNew
                  +-- FindBestMovePlied(n0, n1)
                  |     +-- GenerateMoves() + ScoreMoves(nPlies=1)
                  +-- SwapSides() -> opponent's perspective
                  +-- Recurse: EvaluatePositionCache(nPlies=1)
                        |
                        Ply 1: Loop 36 dice:
                          +-- FindBestMoveInEval()
                          +-- Recurse: EvaluatePositionCache(nPlies=0)
                                |
                                Ply 0 (leaf):
                                  +-- CLASS_RACE -> Race NN
                                  +-- CLASS_CONTACT -> Contact NN
                                  +-- CLASS_BEAROFF1 -> Bearoff DB
                                  +-- CLASS_OVER -> Direct result
                                  Returns arOutput[5]
                |
                Accumulate weighted results / 36
                InvertEvaluationR()
                Cache result
  |
  Output: arOutput[5]
    [0] P(win)
    [1] P(win gammon)
    [2] P(win backgammon)
    [3] P(lose gammon)
    [4] P(lose backgammon)
```

---

## Architecture Summary

The engine combines four evaluation strategies:

1. **Neural network evaluation** (leaf nodes) - Fast approximate position values
   via 3-layer feedforward NNs (contact, race, crashed variants)
2. **Multi-ply lookahead** (internal nodes) - Minimax over all 36 dice outcomes,
   recursing up to 4 plies deep
3. **Monte Carlo rollouts** - Unbiased equity estimates via simulated games with
   configurable trial count and truncation
4. **Bearoff databases** - Exact probabilities for endgame positions

Supporting infrastructure:
- **Position caching** (`EvaluatePositionCache`) - hash-indexed cache avoids recomputation
- **Move filtering/pruning** - multi-pass scoring at increasing ply prunes unlikely moves
- **Variance reduction** - quasi-random dice + baseline subtraction in rollouts
- **Incremental NN evaluation** (`EvaluateFromBase`) - reuses hidden layer activations
- **Threading** - `RunAsyncProcess()` for non-blocking evaluation in GUI mode

Two equity modes:
- **Cubeless equity**: raw expected win percentage
- **Cubeful equity**: match-win-chance accounting for doubling cube dynamics

---

## Performance-Critical Hot Paths

These are the functions most relevant to optimization work:

1. **`EvaluatePositionFull()`** (`eval.c:5403`) - The inner loop of ply-based search.
   Iterates 36 dice, finds best move each, recurses. At 2-ply this means
   36 * (move_gen + score) * 36 * leaf_eval = millions of NN evaluations.

2. **`Evaluate()`** (`lib/neuralnet.c:125`) - NN forward pass. Matrix-vector
   multiply for hidden layer (~128-256 neurons x ~250 inputs) is the
   single most expensive operation per leaf evaluation.

3. **`BasicCubefulRollout()`** (`rollout.c:317`) - The Monte Carlo main loop.
   Each trial plays a full game calling `FindBestMove()` + evaluation per turn.
   Thousands of trials needed for statistical significance.

4. **`ScoreMoves()`** (`eval.c:5578`) - Evaluates all legal moves (up to 3060)
   at a given ply. Move filtering reduces this but it's still expensive.

5. **`GenerateMovesSub()`** (`eval.c:2788`) - Recursive move enumeration.
   Must enumerate and deduplicate all legal checker plays for each dice roll.

6. **`sigmoid()`** (`lib/sigmoid.h:141`) - Called once per neuron per layer.
   Uses lookup table but still called millions of times.

7. **Position cache lookups** - Hash computation (`PositionKey`) and cache
   probing happen at every node in the search tree.
