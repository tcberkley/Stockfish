# Experiment: TT-PV Reduction Increase

## Change
In `src/search.cpp`, increased the TT-PV LMR reduction bonus from 949 to 1100.

```cpp
// Before
if (ss->ttPv)
    r += 949;
// After
if (ss->ttPv)
    r += 1100;
```

## Rationale
The source comment at this line explicitly states "Larger values scale well", indicating this parameter benefits from being pushed higher. Increasing the reduction at TT-PV nodes means moves in positions previously identified as principal variation nodes get reduced more aggressively in LMR, saving search time for more critical branches.

The change increases the reduction from ~0.93 ply to ~1.07 ply at TT-PV nodes.

## Bench Results
- **Master baseline**: 2,288,704 nodes
- **This patch**: 2,385,897 nodes
- **Perft 5**: 4,865,609 (unchanged, correctness verified)

## Fishtest Submission
```
Test repository: https://github.com/tcberkley/Stockfish
Test branch: experiment/ttpv-reduction
Base branch: master
Test signature: 2385897
Base signature: 2288704
SPRT Bounds: Standard STC <0.00, 2.00>
TC: 10+0.1
Threads: 1
Hash: 16
```
