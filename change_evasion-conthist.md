# Experiment: Add Continuation Histories to Evasion Scoring

## Branch
`experiment/evasion-conthist`

## What Changed

### movepick.cpp (evasion scoring)
**Before:**
```cpp
else  // Type == EVASIONS
{
    if (pos.capture_stage(m))
        m.value = PieceValue[capturedPiece] + (1 << 28);
    else
        m.value = (*mainHistory)[us][m.raw()] + (*continuationHistory[0])[pc][to];
}
```

**After:**
```cpp
else  // Type == EVASIONS
{
    if (pos.capture_stage(m))
        m.value = PieceValue[capturedPiece] + (1 << 28);
    else
        m.value = (*mainHistory)[us][m.raw()] + (*continuationHistory[0])[pc][to]
                + (*continuationHistory[1])[pc][to] + (*continuationHistory[2])[pc][to];
}
```

### search.cpp (qsearch contHist array)
**Before:**
```cpp
const PieceToHistory* contHist[] = {(ss - 1)->continuationHistory};
```

**After:**
```cpp
const PieceToHistory* contHist[] = {(ss - 1)->continuationHistory,
                                    (ss - 2)->continuationHistory,
                                    (ss - 3)->continuationHistory};
```

## Rationale
Quiet move ordering uses contHist[0,1,2,3,5] — five continuation history entries. Evasion ordering only used contHist[0]. When in check, move ordering matters even more since we need to find the best escape quickly. Adding contHist[1] (opponent's last move context) and contHist[2] (our move 2 plies ago) provides richer ordering information for evasions.

The qsearch contHist array also needed expanding from 1 to 3 elements, since qsearch generates evasions when in check and would otherwise access out-of-bounds memory.

## Bench Results
- **Master baseline**: 2288704 nodes
- **Experiment**: 2146348 nodes
- **Perft 5**: 4865609 (correct)

## Fishtest Submission
- **Test repository**: https://github.com/tcberkley/Stockfish
- **Test branch**: experiment/evasion-conthist
- **Base branch**: master
- **Test signature**: 2146348
- **Base signature**: 2288704
- **SPRT Bounds**: Standard STC <0.00, 2.00>
- **TC**: 10+0.1
- **Threads**: 1
- **Hash**: 16
