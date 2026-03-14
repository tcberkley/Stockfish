# Stockfish Improvement Ideas

Concrete ideas for ELO gains, organized by category. Each item includes affected files, difficulty, expected impact, and how to validate.

**Ground rule**: only Fishtest can validate ELO. Local tests prove correctness, not strength.

---

## 0. Workflow & Validation Reference

### Compile
```bash
cd src
make -j profile-build ARCH=native
```

### Verify correctness
```bash
./stockfish bench
# Changed node count = functional change confirmed
# Identical node count = refactor only (no functional diff)
```

### Expose a parameter for SPSA tuning
```cpp
// In the relevant .cpp file:
TUNE(my_param, 50, 0, 200, 5, 0.0020);
//   name     init min  max step  lr
```
Then `./stockfish tune` emits UCI options for all tunable params.

### Fishtest submission
1. Push branch to your GitHub fork of Stockfish
2. Submit at https://tests.stockfishchess.org/
3. STC (short time control) first — if it passes, LTC (long time control)
4. Only merge if LTC shows statistically significant gain (or non-regression for simplifications)

---

## 1. SPSA Parameter Tuning Targets

Highest-probability gains — parameterize via `TUNE()`, let Fishtest SPSA optimize.

| Parameter | Current Value | File | Difficulty | Impact |
|---|---|---|---|---|
| Razoring thresholds | 507, 312 | `search.cpp` | Easy | Low-Med |
| NMP reduction formula | 7 + depth/3 | `search.cpp` | Easy | Medium |
| Futility margin weights | 2661 improving, 355 opponentWorsening | `search.cpp` | Easy | Medium |
| ProbCut beta | 229 - 63*improving | `search.cpp` | Easy | Low-Med |
| Singular extension beta | (58+67*ttPv)*depth/57 — (*Scaler) | `search.cpp` | Easy | Medium |
| Double/triple extension margins | various | `search.cpp` | Easy | Low-Med |
| Move count pruning formula | (3+depth²)/(2-improving) | `search.cpp` | Easy | Medium |
| Continuation history pruning threshold | -3826*depth | `search.cpp` | Easy | Low-Med |
| LMR base constant | 2809/128 | `search.cpp` | Easy | Medium |
| LMR adjustment weights | 690 base, 70/move, etc. | `search.cpp` | Easy | Medium |
| Aspiration window delta | 5 + threadIdx%8 + meanSquaredScore/9968 | `search.cpp` | Easy | Low-Med |
| IIR conditions | depth >= 6, priorReduction <= 3 | `search.cpp` | Easy | Low |
| Adaptive LMR re-search thresholds | bestValue+50 / bestValue+9 | `search.cpp` | Easy | Low-Med |

**How to test**: Expose with `TUNE()`, submit SPSA job on Fishtest. Each param independently.

---

## 2. Simplifications — Code Deletion for NPS Gains

Removing complex heuristics can gain NPS speed that outweighs accuracy loss. Test as non-regression on Fishtest.

- [ ] **Remove shuffling detection logic** (`search.cpp` ~lines 144-150)
  - Why: adds complexity for rare edge cases; NPS cost may exceed benefit
  - Impact: Low-Med | Difficulty: Easy
  - Test: Fishtest non-regression (STC+LTC)

- [ ] **Simplify hindsight depth adjustment** (`search.cpp` ~lines 754-757)
  - Why: two separate conditions may not both be needed
  - Impact: Low | Difficulty: Easy

- [ ] **Remove small ProbCut idea** (`search.cpp` ~lines 985-989)
  - Why: secondary heuristic, may not pull its weight
  - Impact: Low | Difficulty: Easy

- [ ] **Simplify correction history** — drop minor piece correction (`search.cpp`, `history.h`)
  - Why: fewer history tables = less cache pressure, faster updates
  - Impact: Low-Med | Difficulty: Medium

- [ ] **Remove allNode LMR scaling** (`search.cpp` ~lines 1227-1228)
  - Why: small adjustment, cost of branch/computation may not justify benefit
  - Impact: Low | Difficulty: Easy

- [ ] **Remove `opponentWorsening` flag entirely** (`search.cpp`)
  - Why: added complexity may be captured by other heuristics (futility, NMP)
  - Impact: Low-Med | Difficulty: Easy

---

## 3. Evaluation Improvements

| Idea | File | Difficulty | Impact |
|---|---|---|---|
| **50-move rule damping**: currently linear (`v -= v*rule50/199`) — try sigmoid or piecewise for better draw modeling near rule50 boundary | `evaluate.cpp` | Medium | Low-Med |
| **Tempo bonus**: no bonus for side to move (~15-25cp in classical engines) — test small NNUE-aware tempo | `evaluate.cpp` | Medium | Low |
| **PSQT/positional blend weights** (125/131) — retune ratio via SPSA | `evaluate.cpp` | Easy | Low |
| **Complexity scaling divisors** (476 for optimism, 18236 for nnue) — expose to SPSA | `evaluate.cpp` | Easy | Low-Med |
| **Material blending base** (77871) — tune | `evaluate.cpp` | Easy | Low |

---

## 4. Move Ordering

| Idea | File | Difficulty | Impact |
|---|---|---|---|
| **Continuation history ply selection**: currently skips ply 4 (uses 0,1,2,3,5) — test including ply 4 or different spacing | `movepick.cpp`, `history.h` | Medium | Low-Med |
| **Capture scoring multiplier**: 7× victim value — try 8× or 9× | `movepick.cpp` | Easy | Low |
| **Check bonus** (16384) is hardcoded — tune relative to history scales | `movepick.cpp` | Easy | Low |
| **Good quiet threshold** (-14000) — tune per depth | `movepick.cpp` | Medium | Low-Med |
| **Recapture history table**: add explicit table tracking recapture success | `history.h`, `movepick.cpp` | Hard | Medium |
| **Threat-aware history**: incorporate threat info into history updates | `history.h`, `search.cpp` | Hard | Medium |

---

## 5. Transposition Table

| Idea | File | Difficulty | Impact |
|---|---|---|---|
| **Age multiplier** in replacement (8×) — retune | `tt.cpp` | Easy | Low |
| **PV node depth bonus** (+2 ply equivalent) — adjust | `tt.cpp` | Easy | Low |
| **4-entry clusters** (if cache line allows) — more entries per probe | `tt.cpp` | Hard | Low-Med |

---

## 6. Time Management

| Idea | File | Difficulty | Impact |
|---|---|---|---|
| **Logarithmic constants** (0.3128, -0.4354) — retune via SPSA | `timeman.cpp` | Easy | Low-Med |
| **Eval-instability feedback**: if score swings wildly between iterations, allocate more time | `timeman.cpp`, `search.cpp` | Medium | Medium |
| **Critical position detection**: spend more time when eval drops sharply | `timeman.cpp`, `search.cpp` | Medium | Medium |
| **Ponder boost** (25%) — may be suboptimal, retune | `timeman.cpp` | Easy | Low |

---

## 7. Threading

| Idea | File | Difficulty | Impact |
|---|---|---|---|
| **Thread specialization**: instead of all threads searching identically, assign different search parameters per thread | `thread.cpp`, `search.cpp` | Hard | Medium |
| **Thread depth differentiation**: vary iterative deepening start depth per thread to increase diversity | `thread.cpp` | Medium | Low-Med |
| **Vote-based move selection weights**: tune how thread votes are aggregated for best move | `thread.cpp` | Medium | Low-Med |

---

## 8. Correction History Tuning

| Parameter | Current Value | File | Difficulty |
|---|---|---|---|
| Weights | 11433/8823/12749/8022 | `search.cpp` | Easy |
| Update weights | 155/128 minor, 181/128 nonpawn | `search.cpp`, `history.h` | Easy |
| Bonus calculation divisors | 10 with bestMove, 8 without | `search.cpp` | Easy |

All are strong SPSA candidates — expose via `TUNE()` and run Fishtest.

---

## 9. NNUE Training

Requires cloning `official-stockfish/nnue-pytorch` separately.

- [ ] **Training data generation**: modify position sampling/filtering (e.g., skip early game positions, weight tactical positions higher)
- [ ] **Learning rate schedule**: experiment with cosine annealing or warm restarts
- [ ] **Batch shuffling improvements**: ensure training data diversity within batches
- [ ] **Network architecture**: experiment with layer sizes or activation functions (HalfKAv2_hm currently)
- [ ] **Quantization-aware training**: reduce inference cost without accuracy loss

Difficulty: Hard | Impact: High (new nets are historically the biggest ELO jumps)

---

## Active Experiment Queue

Focused, single-commit changes ready for Fishtest. Each gets its own branch.

| # | Change | Branch | File | Status |
|---|--------|--------|------|--------|
| 1 | SE: relax TT depth `depth-3` → `depth-4` | `experiment/se-depth-relax` | `search.cpp:1131` | [x] Built & verified, bench 2570388 |
| 2 | Rule50: quadratic damping instead of linear | `experiment/rule50-quadratic` | `evaluate.cpp:84` | [ ] |
| 3 | Razoring: linear instead of quadratic scaling | `experiment/razoring-linear` | `search.cpp:880` | [ ] |
| 4 | NMP verify threshold `16` → `14` | `experiment/nmp-verify-14` | `search.cpp:916` | [ ] |

---

## Priority Order

1. **Active experiments** (above) — focused single changes, ready for Fishtest
2. **SPSA parameter tuning** (Section 1) — lowest effort, highest probability of gain
3. **Simplifications** (Section 2) — easy to test, NPS gains compound
4. **Correction history tuning** (Section 8) — easy SPSA targets
5. **Evaluation improvements** (Section 3) — moderate effort, clear targets
6. **Move ordering** (Section 4) — medium effort, measurable impact
7. **Time management** (Section 6) — medium effort, real-game impact
8. **TT improvements** (Section 5) — small gains likely
9. **Threading** (Section 7) — hard, but thread-count-dependent gains
10. **NNUE training** (Section 9) — highest ceiling, highest effort
