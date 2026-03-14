# Experiment: History-to-Depth Scaling Divisor

## Change
In `src/search.cpp`, reduced the history-to-depth scaling divisor from 2917 to 2500.

```cpp
// Before
lmrDepth += history / 2917;
// After
lmrDepth += history / 2500;
```

## Rationale
The source comment at this line states "Generally, lower divisors scale well." A lower divisor means good history scores contribute more to the LMR depth adjustment, giving moves with proven track records deeper searches. This trades some search breadth for more targeted depth on historically successful moves.

## Bench Results
- **Master baseline**: 2,288,704 nodes
- **This patch**: 2,761,000 nodes
- **Perft 5**: 4,865,609 (unchanged, correctness verified)

## Fishtest Submission
```
Test repository: https://github.com/tcberkley/Stockfish
Test branch: experiment/hist-depth-divisor
Base branch: master
Test signature: 2761000
Base signature: 2288704
SPRT Bounds: Standard STC <0.00, 2.00>
TC: 10+0.1
Threads: 1
Hash: 16
```
