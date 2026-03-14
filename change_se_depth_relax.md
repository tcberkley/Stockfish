# Experiment: SE Depth Relax

## Change
In `src/search.cpp`, relaxed the singular extension TT depth threshold from `depth - 3` to `depth - 4`.

```cpp
// Before
&& ttData.depth >= depth - 3 && !is_shuffling(move, ss, pos))
// After
&& ttData.depth >= depth - 4 && !is_shuffling(move, ss, pos))
```

## Rationale
The singular extension check verifies whether a TT move is "singular" (significantly better than alternatives) by performing a reduced-depth search excluding that move. The TT depth threshold (`depth - 3`) gates which positions are eligible for this analysis.

Relaxing to `depth - 4` allows more TT entries to qualify, increasing the frequency of singular extension checks. The `(*Scaler)` comment in the source notes that "lower extension margins scale well," suggesting this region of the search is amenable to loosening.

**Expected effect**: More moves receive singular extensions, potentially improving tactical accuracy at the cost of additional search overhead. Net effect on strength is uncertain and requires Fishtest validation.

## Bench Results
- **Master baseline**: 2,288,704 nodes
- **This patch**: 2,570,388 nodes
- **Perft 5**: 4,865,609 (unchanged, correctness verified)

## Fishtest Submission
```
Test branch: tcberkley/Stockfish experiment/se-depth-relax
Base branch: official-stockfish/Stockfish master
TC: STC (10+0.1) then LTC (60+0.6)
Threads: 1
Hash: 16
```
