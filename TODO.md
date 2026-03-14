# Stockfish Improvement Ideas

Concrete ideas for Elo gains from exhaustive code review. Each item includes source location, expected Elo, and difficulty. Sorted by expected value within each category.

**Ground rule**: only Fishtest can validate Elo. Local tests prove correctness, not strength.

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

### Perft validation
```bash
echo -e "position startpos\ngo perft 5\nquit" | ./stockfish | grep "Nodes searched"
# Expected: 4865609
```

### Expose a parameter for SPSA tuning
```cpp
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

## 1. Move Ordering Improvements (movepick.cpp)

| # | Idea | Location | Expected Elo | Difficulty |
|---|------|----------|---|---|
| 1 | **Evasions missing continuation histories** — QUIETS use contHist[0,1,2,3,5] but EVASIONS only use contHist[0]. Add contHist[1] and contHist[2] to evasion scoring. | `movepick.cpp:182-188` | +5 to +15 | Easy |
| 2 | **Check bonus SEE threshold** — current `-75` requires SEE >= -75 for check bonus. Try `-125` (more permissive, allows more quiet checks). | `movepick.cpp:170` | +2 to +8 | Easy |
| 3 | **Good quiet threshold** — fixed `-14000`; could be depth-dependent (e.g., `-14000 + 500*depth`). | `movepick.cpp:210` | +2 to +8 | Medium |
| 4 | **Missing continuationHistory[4] in quiet scoring** — indices 0,1,2,3,5 used; index 4 (ply-5) skipped. Search updates all 6 indices (`search.cpp:1873`) but scoring reads only 5. | `movepick.cpp:161-167` | +1 to +5 | Trivial |
| 5 | **Capture MVV multiplier** — current `7 * PieceValue[capturedPiece]` dominates capture history. Try `5 * PieceValue` to let capture history matter more. | `movepick.cpp:155-156` | +1 to +5 | Easy |

---

## 2. Evaluation Improvements (evaluate.cpp)

| # | Idea | Location | Expected Elo | Difficulty |
|---|------|----------|---|---|
| 6 | **Rule50 damping not material-aware** — `v -= v * rule50 / 199` applies uniformly. Should damp less with heavy material (rook endgames are less draw-prone). | `evaluate.cpp:83-84` | +3 to +10 | Medium |
| 7 | **NNUE complexity scaling** — fixed divisors 476/18236 for optimism/eval complexity. Cap complexity or use piecewise scaling. | `evaluate.cpp:76-78` | +1 to +5 | Medium |
| 8 | **Optimism weight coefficient** — optimism = ~9% of blend; increases with material (counterintuitive). Test inverse: more optimism in simple positions. | `evaluate.cpp:81` | +1 to +5 | Medium |

---

## 3. Search Improvements (search.cpp)

| # | Idea | Location | Expected Elo | Difficulty |
|---|------|----------|---|---|
| 9 | **LMR deeper/shallower thresholds asymmetric** — `doDeeperSearch` requires bestValue + 50; `doShallowerSearch` only +9. Adjust toward symmetry (e.g., 35/25). | `search.cpp:1250-1251` | +0.5 to +2 | Easy |
| 10 | **Futility correction value underweighted** — `abs(correctionValue) / 176900` has almost no impact. Try `/150000` or `/120000` for more influence. | `search.cpp:884` | +0.5 to +1.5 | Easy |
| 11 | **Quiet history penalty decay too aggressive** — decay factor 963/1024 ~ 0.94x per move; 10th move gets 57% penalty. Try 980/1024 or add floor. | `search.cpp:1842-1846` | +0.5 to +2 | Easy |
| 12 | **Multi-cut return without margin** — returns shallow-search value directly as beta cutoff. Could soft-bound: `min(value, beta + 40)`. | `search.cpp:1160-1163` | +0.5 to +1.5 | Easy |

---

## 4. Transposition Table (tt.cpp)

| # | Idea | Location | Expected Elo | Difficulty |
|---|------|----------|---|---|
| 13 | **TT replacement age weight** — `depth8 - relative_age` weights depth:age at 1:1. Try `depth8 - 2 * relative_age` to evict stale entries faster. | `tt.cpp:237-238` | +2 to +8 | Trivial |

---

## 5. Time Management (timeman.cpp)

| # | Idea | Location | Expected Elo | Difficulty |
|---|------|----------|---|---|
| 14 | **Search feedback for time allocation** — no feedback from search quality/instability to time budget. Add eval instability metric to increase time in critical positions. | `timeman.cpp` | +5 to +15 | Medium-Hard |
| 15 | **Ponder time scaling** — fixed 25% bonus; should scale with time control. More bonus for long TC, less for blitz. | `timeman.cpp:136-137` | +1 to +5 | Easy |

---

## 6. Threading (thread.cpp)

| # | Idea | Location | Expected Elo | Difficulty |
|---|------|----------|---|---|
| 16 | **Thread voting PV length filter** — `int(newThreadPV.size() > 2)` zeroes out short-PV threads entirely. Could use `(newThreadPV.size() + 1)` as multiplier instead. | `thread.cpp:387-389` | +2 to +8 | Easy |
| 17 | **Thread voting baseline constant** — fixed `+ 14`; tune range 5-25. | `thread.cpp:360` | +1 to +3 | Easy |

---

## 7. History Table Architecture (history.h, search.cpp)

| # | Idea | Location | Expected Elo | Difficulty |
|---|------|----------|---|---|
| 18 | **Continuation history bonuses irregular** — {1106,705,316,572,126,427} has no clear pattern. Test geometric decay or simpler progression. | `search.cpp:1873` | +1 to +3 | Easy |
| 19 | **PieceToHistory D vs ButterflyHistory D mismatch** — 30000 vs 7183 (4.2x difference); may cause temporal inconsistency. Test reducing PieceToHistory D to 15000. | `history.h:135,145` | +0 to +3 | Easy |

---

## 8. SPSA Tuning Targets

Parameterize via `TUNE()`, let Fishtest SPSA optimize.

| # | Parameter | Current | Location | Difficulty |
|---|-----------|---------|----------|---|
| 20 | Optimism scaling | 142, 86 | `search.cpp:359` | Easy |
| 21 | NMP condition constants | 17, 50, 359 | `search.cpp:893` | Easy |
| 22 | SE beta margin | 58, 67, 57 | `search.cpp:1133` | Easy |
| 23 | LMR cut node reduction | 3582, 1015 | `search.cpp:1201` | Easy |
| 24 | LMR TT move bonus | 2069 | `search.cpp:1213` | Easy |
| 25 | History-to-depth divisor (*Scaler: lower = better) | 2917 | `search.cpp:1095` | Easy |
| 26 | TT-PV reduction (*Scaler: higher = better) | 949 | `search.cpp:1047` | Easy |
| 27 | Aspiration window expansion | delta/3 | `search.cpp:416` | Easy |
| 28 | Capture SEE margin | 185*depth, captHist/28 | `search.cpp:1077` | Easy |
| 29 | SEE margin for quiets | 25*lmrDepth^2 | `search.cpp:1114` | Easy |
| 30 | Razoring thresholds | 507, 312 | `search.cpp:873` | Easy |
| 31 | NMP reduction | 7 + depth/3 | `search.cpp:899` | Easy |
| 32 | ProbCut beta | 229 - 63*improving | `search.cpp:938` | Easy |
| 33 | Futility pruning base | 77, 22 | `search.cpp:880` | Easy |

---

## 9. Simplification Opportunities

Test as non-regression on Fishtest — gains come from NPS improvement exceeding accuracy loss.

- [ ] **Consolidate LMR adjustments** — 8+ coefficients could reduce to fewer knobs (`search.cpp:1195-1228`)
- [ ] **Simplify correction history weights** — 4 arbitrary multipliers (`search.cpp:92`)
- [ ] **Remove opponentWorsening flag** — may be captured by other heuristics (futility, NMP)

---

## 10. Algorithmic Ideas

Novel heuristics — higher risk, potentially high reward.

- [ ] **Eval trend heuristic** — track eval over 4+ plies for pruning decisions. `improving` only looks back 2 plies.
- [ ] **Per-NNUE-confidence LMR** — use nnueComplexity in search, not just eval. Already computed but unused in search decisions.
- [ ] **Position-specific time allocation** — branching factor awareness for time budgeting.

---

## Active Experiment Queue (top 10 by expected value)

| # | Change | Expected Elo | Difficulty | Status |
|---|--------|---|---|---|
| 1 | Add contHist[1,2] to evasion scoring | +5 to +15 | Easy | [x] Bench: 2146348 |
| 2 | TT replacement: double age weight | +2 to +8 | Trivial | [ ] Ready |
| 3 | Check bonus: relax SEE to -125 | +2 to +8 | Easy | [ ] Ready |
| 4 | Thread voting: soften PV length filter | +2 to +8 | Easy | [ ] Ready |
| 5 | Add continuationHistory[4] to quiet scoring | +1 to +5 | Trivial | [ ] Ready |
| 6 | Rule50 damping: material-aware | +3 to +10 | Medium | [ ] Ready |
| 7 | Good quiet threshold: depth-dependent | +2 to +8 | Medium | [ ] Ready |
| 8 | Capture MVV: reduce from 7x to 5x | +1 to +5 | Easy | [ ] Ready |
| 9 | LMR deeper/shallower: symmetrize 35/25 | +0.5 to +2 | Easy | [ ] Ready |
| 10 | Quiet penalty decay: slow to 980/1024 | +0.5 to +2 | Easy | [ ] Ready |

---

## Active Fishtest Submissions

| Branch | Change | Status |
|--------|--------|--------|
| `experiment/eval-aware-nmp` | Eval-aware NMP: extra R when eval >> beta (divisor 250, max +2, depth > 10) | ACTIVE: +1.02 Elo, LLR 0.80/2.94 @ 48k games (2026-03-14) |

## Killed Fishtest Tests

| Branch | Change | Result |
|--------|--------|--------|
| `experiment/se-depth-relax` | SE TT depth `depth-3` -> `depth-4` | Failed |
| `experiment/ttpv-reduction` | TT-PV LMR reduction 949 -> 1100 | Failed |
| `experiment/hist-depth-divisor` | History-to-depth divisor 2917 -> 2500 | Failed |
