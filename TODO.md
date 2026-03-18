# Stockfish Experiment Ideas

Machine-parseable experiment queue. The `/experiment` skill scans for the first `[READY]` item.
See `HISTORY.md` for all killed experiments and patterns.

**Priority:** Simplifications (remove dead terms) ≈ signal additions at high-traffic points > formula reshapes > parameter tweaks.
Items within each tier ordered READY first, then by probability.

**Promising patterns (from upstream):** Simplifications of recently-added terms that turn out to be noise (~55% of upstream passes). Minimal signal additions connecting existing in-scope data to existing decisions (1-3 line diffs). Code reorderings that move logic above/below cutoff points. Ripple effects from recent upstream patches (same principle applied to parallel code).

**Killed experiment patterns:** pure parameter shrinks fail, relaxing SE conditions fails, increasing TT-PV reduction fails, optimism direction inversion fails, ProbCut improving modifications fail (beta + depth both killed), aspiration window asymmetry fails, correction divisor amplification fails, evasion contHist enrichment fails (v2 fizzled, v3 killed), opponentWorsening additions fail (NMP, quiet futility — signal too weak), rule50 shape changes fail (material-aware + quadratic both killed), logarithmic LMR moveCount fails, qsearch history updates fail, PieceToHistory D reduction fails, post-LMR history modifications fail (balance update + depth-scaled bonus both killed), fail-high blend modifications fail (multi-cut blend flat at 138k + overshoot-aware killed), negative extension gradation fails, TT PV replacement bonus fails, depth-scaled correction fails, parent statScore in SE beta fails, capture statScore contHist flat at 90k, contHist {4} and removing {5} both fail (skip-4-keep-5 is intentional), correction update modifications fail (TT-accuracy, moveCount, depth-scale all killed), signed LMR correction fails.

---

### 83. Futility improving: depth gate only [SUBMITTED]
- **Category:** ADD-SIGNAL | **Tier:** A | **File:** src/search.cpp:907 | **Branch:** experiment/futility-improving-gate
- **Rationale:** When improving (eval better than 2 plies ago), tighten the futility depth cutoff by 1 (depth < 14 instead of 15). Upstream already moved the gate from 16 to 15 unconditionally; this makes it conditional on improving so the gate is tighter in improving positions and unchanged otherwise.
~~~diff
-        if (!ss->ttPv && depth < 15 && eval - futility_margin(depth) >= beta && eval >= beta
+        if (!ss->ttPv && depth < 15 - improving && eval - futility_margin(depth) >= beta && eval >= beta
             && (!ttData.move || ttCapture) && !is_loss(beta) && !is_win(eval))
~~~

### 84. Futility improving: depth gate + ttMove relaxation [SUBMITTED]
- **Category:** ADD-SIGNAL | **Tier:** A | **File:** src/search.cpp:907 | **Branch:** experiment/futility-improving-combined
- **Rationale:** Two changes when improving: tighten depth gate by 1 (depth < 14) AND allow pruning even when a quiet TT move exists. Both say "in improving positions, trust the eval-based pruning decision more." The ttMove relaxation targets the LTC failure mode of the gate-only patch.
~~~diff
-        if (!ss->ttPv && depth < 15 && eval - futility_margin(depth) >= beta && eval >= beta
-            && (!ttData.move || ttCapture) && !is_loss(beta) && !is_win(eval))
+        if (!ss->ttPv && depth < 15 - improving && eval - futility_margin(depth) >= beta && eval >= beta
+            && (!ttData.move || ttCapture || improving) && !is_loss(beta) && !is_win(eval))
~~~

### 85. Simplify futility margin: remove opponentWorsening [READY]
- **Category:** SIMPLIFY | **Tier:** B | **File:** src/search.cpp:883 | **Branch:** experiment/futility-simplify-opponentworsening
- **Rationale:** The `355 * opponentWorsening` term contributes only 19-27cp to the futility margin (compared to `2661 * improving` which contributes 140-188cp). Kill patterns note "opponentWorsening additions fail (NMP, quiet futility — signal too weak)". If the signal is too weak to add value anywhere new, the same logic argues for removing it from this formula. Runs as a non-regression test `<-1.75, 0.25>` — lower bar than a gain test.
~~~diff
         return futilityMult * d
-                 - (2661 * improving + 355 * opponentWorsening) * futilityMult / 1024  //
+                 - 2661 * improving * futilityMult / 1024
              + std::abs(correctionValue) / 176900;
~~~

**Kill pattern check:** "opponentWorsening additions fail" confirms the signal is weak — this is removal, not addition, which has a lower bar (non-regression). No other kill pattern applies.
