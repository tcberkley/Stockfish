# Experiment History

Archive of all tested experiments. Agents use this for dedup and pattern analysis.

## What Works (from upstream analysis)

Based on recent upstream Stockfish patches that pass Fishtest:

- **Simplifications** (~37% of passes): Removing recently-added terms that turn out to be noise. Examples: simplify probCutDepth term (deleted a complex clamp), simplify malus decay (deleted a branch), remove check term in capture movepick, remove low ply history for evasions, remove secondary TT aging. Run as non-regression tests `<-1.75, 0.25>`.
- **Minimal signal additions** (~20% of passes): 1-3 line diffs connecting existing in-scope data to an existing decision. Examples: "more NMP when improving" (added `+ improving`), incorporate statScore into history bonus (2 lines), ALL node depth-dependent reduction (3 lines).
- **Formula reshapes** (~15% of passes): Change formula shape at a high-traffic decision point. Examples: depth-dependent scaling applied to existing constants.
- **Code reorderings** (~10% of passes): Move logic to a different position in search. Examples: history updates moved above/below cutoff points.
- **Ripple effects**: A patch passes, then the same principle applied to a parallel code path also passes. Look for these after every upstream pass.

**Key insight**: Successful additions are very small (1-4 lines). Successful simplifications delete lines. If a diff is growing complex, it's a warning sign.

## Kill Patterns (summary)

**Killed experiment patterns:** pure parameter shrinks fail, relaxing SE conditions fails, increasing TT-PV reduction fails, optimism direction inversion fails, ProbCut improving modifications fail (beta + depth both killed), aspiration window asymmetry fails, correction divisor amplification fails, evasion contHist enrichment fails (v2 fizzled, v3 killed), opponentWorsening additions fail (NMP, quiet futility — signal too weak), rule50 shape changes fail (material-aware + quadratic both killed), logarithmic LMR moveCount fails, qsearch history updates fail, PieceToHistory D reduction fails, post-LMR history modifications fail (balance update + depth-scaled bonus both killed), fail-high blend modifications fail (multi-cut blend flat at 138k + overshoot-aware killed), negative extension gradation fails, TT PV replacement bonus fails, depth-scaled correction fails, parent statScore in SE beta fails, capture statScore contHist flat at 90k, contHist {4} and removing {5} both fail (skip-4-keep-5 is intentional), correction update modifications fail (TT-accuracy, moveCount, depth-scale all killed), signed LMR correction fails.

**Promising pattern:** Adding missing history context to under-served code paths has consistently failed — evasion-conthist v2 fizzled, v3 killed, capture statScore contHist dead flat at 90k games, contHist[4] killed, removing contHist[5] killed. The {0,1,2,3,5} pattern appears well-tuned.

## Killed Experiments

### 1. Rule50 damping: material-aware [KILLED]
- **Tier:** A | **File:** src/evaluate.cpp:84 | **Branch:** experiment/rule50-material
- **Result:** LLR -0.95 (-2.94,2.94) @ 10144 games — killed. Rule50 shape changes fail (see also #64).

### 2. Optimism material blending inversion [KILLED]
- **Tier:** A | **File:** src/evaluate.cpp:81 | **Branch:** experiment/optimism-invert
- **Result:** LLR -1.09 @ 800 games — killed fast. Optimism growing with material is correct.

### 3. Secondary ProbCut improving term [KILLED]
- **Tier:** A | **File:** src/search.cpp:986 | **Branch:** experiment/probcut2-improving
- **Result:** LLR -0.33 @ 864 games — killed fast. Secondary ProbCut threshold doesn't benefit from improving adjustment.

### 4. Aspiration window asymmetric expansion [KILLED]
- **Tier:** A | **File:** src/search.cpp:416 | **Branch:** experiment/aspiration-asymmetric
- **Result:** Failed — killed early. Symmetric expansion appears optimal.

### 8. Multi-cut blend toward beta [KILLED]
- **Tier:** A | **File:** src/search.cpp:1163 | **Branch:** experiment/multicut-blend
- **Result:** LLR -0.84 (-2.94,2.94) @ 138464 games — dead flat after very large sample.

### 9. LMR correction divisor [KILLED]
- **Tier:** A | **File:** src/search.cpp:1197 | **Branch:** experiment/correction-divisors
- **Result:** LLR 0.18 @ 20.1k games — dead flat after large sample. Correction divisor amplification doesn't help.

### 11. PieceToHistory D=30000 [KILLED]
- **Tier:** B | **File:** src/history.h:145 | **Branch:** experiment/pieceto-d-reduce
- **Result:** LLR -0.33 (-2.94,2.94) @ 2752 games — killed. High saturation limit appears correct; stickiness is a feature not a bug.

### 24. contHist[4] skipped in quiet scoring [KILLED]
- **Tier:** C | **File:** src/movepick.cpp:166-167 | **Branch:** experiment/conthist4-quiet
- **Result:** LLR -0.73 (-2.94,2.94) @ 19840 games — killed. contHist[4] is noise, skip is correct.

### 24b. Remove contHist[5] from quiet scoring [KILLED]
- **Tier:** C | **File:** src/movepick.cpp:167 | **Branch:** experiment/remove-conthist5
- **Result:** LLR -1.43 (-2.94,2.94) @ 15808 games — killed. contHist[5] carries signal despite index 4 being noise; skip-4-keep-5 is intentional.
~~~diff
-            m.value += (*continuationHistory[3])[pc][to];
-            m.value += (*continuationHistory[5])[pc][to];
+            m.value += (*continuationHistory[3])[pc][to];
~~~

### 40. NMP extra R when eval >> beta [KILLED]
- **Branch:** experiment/eval-aware-nmp
- **Result:** LLR 0.02 @ 144k games — basically flat, killed.

### 41. contHist[1,2] in evasion scoring [KILLED]
- **Branch:** experiment/evasion-conthist-v2
- **Result:** LLR 0.03 @ 130.1k games — fizzled completely from LLR 1.31 peak. Evasion contHist enrichment doesn't reliably gain.

### 42. Double TT replacement age weight [KILLED]
- **Branch:** experiment/tt-age-weight
- **Result:** Failed — killed early.

### 43. Soften PV length filter [KILLED]
- **Branch:** experiment/thread-vote-pvlen
- **Result:** LLR -1.65 @ 38k games — killed.

### 44. SE TT depth depth-3 to depth-4 [KILLED]
- **Branch:** experiment/se-depth-relax
- **Result:** LLR -0.78 @ 10k games — relaxing SE conditions doesn't work.

### 45. TT-PV LMR reduction 949 to 1100 [KILLED]
- **Branch:** experiment/ttpv-reduction
- **Result:** LLR -1.44 @ 9.9k games — increasing TT-PV reduction doesn't work.

### 46. History-to-depth divisor 2917 to 2500 [KILLED]
- **Branch:** experiment/hist-depth-divisor
- **Result:** LLR -2.23 @ 6.9k games — pure parameter shrinks in search.cpp fail.

### 47. Qsearch capture history update on first-move fail-high [KILLED]
- **Tier:** A | **File:** src/search.cpp:1691-1692 | **Branch:** experiment/qsearch-caphist
- **Result:** LLR -0.30 (-2.94,2.94) @ 4096 games — killed. Qsearch history updates don't help.

### 48. Capture statScore with continuation history [KILLED]
- **Tier:** A | **File:** src/search.cpp:1215-1217 | **Branch:** experiment/capture-statscore-conthist
- **Result:** LLR -0.07 (-2.94,2.94) @ 90816 games — dead flat after large sample. Capture statScore doesn't benefit from contHist.
~~~diff
-        if (capture)
-            ss->statScore = 892 * int(PieceValue[pos.captured_piece()]) / 128
-                          + captureHistory[movedPiece][move.to_sq()][type_of(pos.captured_piece())];
+        if (capture)
+            ss->statScore = 892 * int(PieceValue[pos.captured_piece()]) / 128
+                          + captureHistory[movedPiece][move.to_sq()][type_of(pos.captured_piece())]
+                          + (*contHist[0])[movedPiece][move.to_sq()] / 2;
~~~

### 50. Evasion scoring enrichment v3 (add half-weight pawnHistory) [KILLED]
- **Tier:** B | **File:** src/movepick.cpp:186-187 | **Branch:** experiment/evasion-conthist-v3
- **Result:** LLR -2.05 (-2.94,2.94) @ 62304 games — killed. Evasion scoring enrichment consistently fails (v2 fizzled, v3 negative).
~~~diff
-                m.value = (*mainHistory)[us][m.raw()] + (*continuationHistory[0])[pc][to];
+                m.value = (*mainHistory)[us][m.raw()] + (*continuationHistory[0])[pc][to]
+                        + sharedHistory->pawn_entry(pos)[pc][to] / 2;
~~~

### 52. Post-LMR history update: add negative on fail-low [KILLED]
- **Tier:** B | **File:** src/search.cpp:1258-1259 | **Branch:** experiment/postlmr-balanced
- **Result:** LLR -0.56 (-2.94,2.94) @ 5856 games — killed. Upward bias in post-LMR history appears intentional.

### 57. NMP reduction + opponentWorsening [KILLED]
- **Tier:** A | **File:** src/search.cpp:899 | **Branch:** experiment/nmp-opponentworsening
- **Result:** LLR -0.35 (-2.94,2.94) @ 1024 games — killed fast. opponentWorsening signal too weak for NMP (see also #61).

### 60. Negative extension gradation [KILLED]
- **Tier:** B | **File:** src/search.cpp:1174-1175 | **Branch:** experiment/negext-gradation
- **Result:** LLR -1.67 (-2.94,2.94) @ 61056 games — killed. Hard -3 negative extension is correct; gradation doesn't help.
~~~diff
-            else if (ttData.value >= beta)
-                extension = -3;
+            else if (ttData.value >= beta)
+                extension = -2 - (ttData.value >= beta + 200);
~~~

### 61. Quiet futility margin + improving/opponentWorsening [KILLED]
- **Tier:** B | **File:** src/search.cpp:1097-1098 | **Branch:** experiment/quiet-futility-flags
- **Result:** LLR -0.36 (-2.94,2.94) @ 1120 games — killed fast. opponentWorsening additions consistently fail; improving alone untested here.

### 63. MoveCount LMR adjustment: linear → logarithmic [KILLED]
- **Tier:** A | **File:** src/search.cpp:1196 | **Branch:** experiment/lmr-movecount-log
- **Result:** LLR -0.61 (-2.94,2.94) @ 8864 games — killed. Linear moveCount scaling is correct for LMR.

### 64. Rule50 damping: linear → quadratic [KILLED]
- **Tier:** A | **File:** src/evaluate.cpp:84 | **Branch:** experiment/rule50-quadratic
- **Result:** LLR -0.46 (-2.94,2.94) @ 2464 games — killed. Linear rule50 damping is correct (see also #1).

### 68. TT replacement ignores PV status [KILLED]
- **Tier:** B | **File:** src/tt.cpp:101,237-239 | **Branch:** experiment/tt-replace-pv
- **Result:** LLR -1.22 (-2.94,2.94) @ 18688 games — killed. PV bonus in eviction doesn't help; save-only asymmetry appears intentional.
~~~diff
File: src/tt.cpp:237-239
-    if (replace->depth8 - replace->relative_age(generation8)
-        > tte[i].depth8 - tte[i].relative_age(generation8))
+    if (replace->depth8 - replace->relative_age(generation8) + 2 * bool(replace->genBound8 & 0x4)
+        > tte[i].depth8 - tte[i].relative_age(generation8) + 2 * bool(tte[i].genBound8 & 0x4))
~~~
Note: PV is bit 2 of genBound8 (extracted as `genBound8 & 0x4`). The `2 *` mirrors the save condition's `2 * pv` bonus. genBound8 is private so the probe() method in TranspositionTable (a friend class) can access it directly.

### 73. Signed correction in LMR [KILLED]
- **Tier:** A | **File:** src/search.cpp:1197 | **Branch:** experiment/lmr-signed-correction
- **Result:** LLR -0.80 (-2.94,2.94) @ 6464 games — killed. Signed LMR correction fails.
- **Rationale:** `r -= std::abs(correctionValue) / 26878` always decreases reduction regardless of correction direction. This is theoretically backwards for one direction: positive correction (eval underestimates → safe → increase reduction), negative correction (eval overestimates → danger → decrease reduction). Currently both decrease reduction. Flip from `abs` to signed so the correction direction matters. Item 9 (killed) changed the divisor magnitude; item 39 removes the term entirely. Neither tried the directional approach.
~~~diff
- r -= std::abs(correctionValue) / 26878;
+ r += correctionValue / 26878;
~~~

### 74. TT prediction accuracy → correction update scaling [KILLED]
- **Tier:** A | **File:** src/search.cpp:1475-1479 | **Branch:** experiment/corr-tt-accuracy
- **Result:** LLR -0.85 (-2.94,2.94) @ 13120 games — killed. Correction update TT-accuracy scaling fails.
- **Rationale:** Correction history updates with a fixed formula `(bestValue - staticEval) * depth / (bestMove ? 10 : 8)`. Positions where the TT move was correct (bestMove == ttData.move) are already well-predicted — update correction LESS. Positions where a different move turned out best are surprises — update MORE. Creates a feedback loop: correction learns faster from surprises, slower from confirmations.
~~~diff
-     if (!ss->inCheck && !(bestMove && pos.capture(bestMove))
-         && (bestValue > ss->staticEval) == bool(bestMove))
-     {
-         auto bonus = std::clamp(int(bestValue - ss->staticEval) * depth / (bestMove ? 10 : 8),
-                                 -CORRECTION_HISTORY_LIMIT / 4, CORRECTION_HISTORY_LIMIT / 4);
+     if (!ss->inCheck && !(bestMove && pos.capture(bestMove))
+         && (bestValue > ss->staticEval) == bool(bestMove))
+     {
+         int  ttHit  = bestMove == ttData.move;
+         auto bonus = std::clamp(int(bestValue - ss->staticEval) * depth / (bestMove ? 10 : 8)
+                                   * (5 - 2 * ttHit) / 4,
+                                 -CORRECTION_HISTORY_LIMIT / 4, CORRECTION_HISTORY_LIMIT / 4);
~~~
Note: When ttMove matches (ttHit=1): bonus *= 3/4 (damped). When ttMove misses (ttHit=0): bonus *= 5/4 (amplified). Net effect is slightly larger corrections since misses are more common.

### 75. Depth-scaled correction in eval [KILLED]
- **Tier:** A | **File:** src/search.cpp:714 | **Branch:** experiment/corr-depth-scale
- **Result:** LLR -0.61 (-2.94,2.94) @ 1664 games — killed fast. Depth-scaling correction doesn't help.
~~~diff
-     const auto correctionValue      = correction_value(*this, pos, ss);
+     const auto correctionValue      = correction_value(*this, pos, ss) * 16 / (8 + depth);
~~~
Note: At depth 4: multiplier = 16/12 = 1.33 (slightly stronger). At depth 8: 16/16 = 1.0 (same). At depth 16: 16/24 = 0.67 (weaker near root). Leaves (depth≤2) get ~2x correction weight. Only modifies the main search call site (line 714), not qsearch (line 1563).

### 77. moveCount in correction update [KILLED]
- **Tier:** B | **File:** src/search.cpp:1478 | **Branch:** experiment/corr-movecount
- **Result:** LLR -0.36 (-2.94,2.94) @ 1632 games — killed fast. Correction update moveCount scaling fails.
- **Rationale:** Correction bonus is `(bestValue - staticEval) * depth / (bestMove ? 10 : 8)`. A position searched to depth 10 with 30 moves explored is a much more reliable signal than depth 10 with 2 moves. moveCount measures how exhaustively we searched. Scale bonus by moveCount to weight exhaustive searches more.
~~~diff
-         auto bonus = std::clamp(int(bestValue - ss->staticEval) * depth / (bestMove ? 10 : 8),
+         auto bonus = std::clamp(int(bestValue - ss->staticEval) * depth / (bestMove ? 10 : 8)
+                                   * std::min(moveCount, 16) / 8,
~~~
Note: moveCount=8 → multiplier=1.0 (neutral). moveCount=16+ → 2.0 (double). moveCount=1 → 0.125 (near-zero). Searches that explored many moves update correction more aggressively.

### 78. Parent statScore in singular extension beta [KILLED]
- **Tier:** B | **File:** src/search.cpp:1133 | **Branch:** experiment/se-parent-statscore
- **Result:** LLR -0.59 (-2.94,2.94) @ 3648 games — killed. Parent statScore doesn't improve SE threshold.
~~~diff
- Value singularBeta  = ttData.value - (58 + 67 * (ss->ttPv && !PvNode)) * depth / 57;
+ Value singularBeta  = ttData.value - (58 + 67 * (ss->ttPv && !PvNode)) * depth / 57
+                      + std::clamp((ss - 1)->statScore / 16384, -2, 2);
~~~
Note: statScore ranges roughly ±30000. Dividing by 16384 gives ±2 pawns of adjustment. If opponent's move was clearly best (high statScore), we're less likely forced → raise beta → less extension. If opponent had many good options (low statScore for the move chosen), we might be forced → lower beta → more extension.

### 81. Post-LMR confirmation bonus depth-scaled [KILLED]
- **Tier:** A | **File:** src/search.cpp:1259 | **Branch:** experiment/postlmr-depth-bonus
- **Result:** Killed. Post-LMR bonus depth-scaling fails.
- **Rationale:** Post-LMR continuation history update uses a **fixed** bonus of 1342 regardless of depth. Every other history bonus in the codebase scales with depth (line 1832: `min(124*depth-84, 1376)`). A depth-20 LMR confirmation is far more informative than depth-3 but gets identical credit.
~~~diff
-                update_continuation_histories(ss, movedPiece, move.to_sq(), 1342);
+                int postLMRBonus = std::min(124 * newDepth - 84, 1376);
+                update_continuation_histories(ss, movedPiece, move.to_sq(), postLMRBonus);
~~~
Note: Crossover at newDepth ~11.5. Reduces bonus at shallow depths, slightly increases at deep.

### 79. Fail-high blend uses beta overshoot [KILLED]
- **Tier:** B | **File:** src/search.cpp:1408 | **Branch:** experiment/failhigh-overshoot
- **Result:** Killed. Fail-high overshoot-aware blending fails.
- **Rationale:** `bestValue = (bestValue * depth + beta) / (depth + 1)` blends linearly with depth. But the confidence of a fail-high depends on HOW MUCH we exceeded beta, not just search depth. Marginal fail-highs should blend more aggressively toward beta; confident fail-highs should keep their value. Item 35 adds a constant offset; this adjusts dynamically by overshoot.
~~~diff
- bestValue = (bestValue * depth + beta) / (depth + 1);
+ bestValue = (bestValue * std::max(1, depth - (bestValue - beta > 100 ? 0 : 1)) + beta)
+           / (std::max(1, depth - (bestValue - beta > 100 ? 0 : 1)) + 1);
~~~
Note: When overshoot > 100cp: use normal blend (depth). When overshoot ≤ 100cp: blend more aggressively toward beta (depth-1 effective weight).

### 82. ProbCut depth + improving [KILLED]
- **Tier:** A | **File:** src/search.cpp:948 | **Branch:** experiment/probcut-depth-improving
- **Result:** Killed. ProbCut depth + improving fails.
- **Rationale:** ProbCut beta uses `improving` (line 938) but ProbCut depth is a fixed `depth-4`. When improving, search one ply deeper for more reliable ProbCut confirmation. NOT the killed "secondary ProbCut improving term" (which was about beta, not depth).
~~~diff
-        Depth      probCutDepth = depth - 4;
+        Depth      probCutDepth = depth - 4 + improving;
~~~

## Passed Experiments

(none yet)
