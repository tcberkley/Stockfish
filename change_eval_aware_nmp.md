# Change: Eval-Aware Null Move Pruning Reduction

## Summary
Add an eval-dependent component to the NMP reduction formula, gated behind depth > 10. When the static eval is significantly above beta at deep search depths, NMP uses a larger reduction, pruning more aggressively in positions where we are comfortably winning.

## Motivation
NMP is the single most impactful pruning technique in Stockfish. The current reduction formula (`R = 7 + depth/3`) is purely depth-dependent. In positions where `eval >> beta`, the null move hypothesis is more likely to hold — we can afford to search the null-move subtree at a shallower depth because the position is strongly in our favor. This is a well-established idea used in engines like Ethereal and Berserk.

## Change

**File:** `src/search.cpp`, line 899

**Before:**
```cpp
Depth R = 7 + depth / 3;
```

**After:**
```cpp
Depth R = 7 + depth / 3 + (depth > 10) * std::min((eval - beta) / 300, 2);
```

**How it works:**
- At depth <= 10: no change (depth gate prevents over-pruning at shallow nodes)
- At depth > 10 with eval close to beta: no change (integer division yields 0)
- At depth > 10 with eval 300+ above beta: R increases by 1
- At depth > 10 with eval 600+ above beta: R increases by 2 (capped)

**Safety mechanisms:**
- Depth gate (>10): shallow searches are more sensitive to over-pruning
- Cap of 2: prevents extreme reductions
- Divisor of 300: conservative threshold — requires a substantial eval margin
- NMP already has a verification search at depth >= 16

## Iterations Tried

| Variant | Nodes | vs Baseline | Notes |
|---------|-------|-------------|-------|
| Baseline | 2,288,704 | — | `R = 7 + depth/3` |
| `/200, cap 3, no gate` | 2,548,482 | +11.4% | Too aggressive, causes re-searches |
| `/400, cap 2, no gate` | 2,542,566 | +11.1% | Still too aggressive without depth gate |
| `/300, cap 2, depth>10` | 2,197,915 | **-4.0%** | Sweet spot — fewer nodes searched |

## Bench Results

### Baseline (unmodified)
```
Nodes searched  : 2,288,704
Nodes/second    : ~902,000
```

### After change
```
Nodes searched  : 2,197,915  (-4.0%)
Nodes/second    : ~880,000
```

- **Node count change confirms functional difference** (deterministic across runs)
- **4% fewer nodes** means the search tree is smaller — more aggressive pruning is holding
- NPS is slightly lower (profile-guided optimization was trained on the old code path), but total search time is comparable
- Perft 5 = 4,865,609 — **move generation correctness verified**

## SPSA Tuning Potential
The three constants in this change are good SPSA candidates:
- **Depth gate threshold** (currently 10): try range 8-14
- **Divisor** (currently 300): try range 150-500
- **Cap** (currently 2): try range 1-4

To expose for SPSA tuning, add to `tune.cpp`:
```cpp
TUNE(nmpEvalGateDepth, 10, 6, 16, 1, 0.0020);
TUNE(nmpEvalDivisor, 300, 100, 600, 25, 0.0020);
TUNE(nmpEvalCap, 2, 1, 5, 1, 0.0020);
```

## Next Steps
1. Push to fork: `git push origin experiment/eval-aware-nmp`
2. Submit to Fishtest STC for ELO validation
3. If STC passes, submit LTC
4. If both pass, consider SPSA tuning the constants
