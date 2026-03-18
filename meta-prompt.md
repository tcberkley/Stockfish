# Meta-Prompt: Stockfish Experiment Idea Discovery

You are generating novel experiment ideas for Stockfish. Your output will be appended to `TODO.md` — a structured queue of testable patches submitted to Fishtest (SPRT framework, ~5k–150k games per test).

## Objective

Produce TODO.md items that represent testable patches. **Simplifications account for ~37% of upstream Stockfish patches that pass Fishtest** — they are the single most productive category. Signal additions, formula reshapes, and structural changes also pass, but less frequently than simplifications.

Quality bar: an idea is good if a strong chess engine developer would say "huh, I hadn't thought of that — and it's not obviously wrong." Ideas that are minor parameter nudges, already killed, or duplicates of existing items are worthless. **Aim for ~40% simplifications, ~30% signal additions, ~30% reshapes/structural in your output.**

---

## Ground Rules

1. **Read `TODO.md` fully before generating anything.** Pay special attention to the killed-experiment patterns listed at the top and all `[KILLED]` items. Understand *why* each failed — the failure reason constrains what's worth trying next.

2. **Every idea must be verified against current source.** Read the actual code at the line you're targeting. Confirm variable names, types, data flow, and surrounding context. A diff that doesn't compile is worse than no diff.

3. **Dedup against all existing TODO items** — including killed ones. If your idea is a variant of something already tried, you must explain what's structurally different and why the prior failure doesn't apply.

4. **Output format must match TODO.md conventions exactly** (see Output Format section below).

---

## Thinking Frameworks

These are open-ended provocations to guide discovery. Don't treat them as a checklist — use whichever ones spark genuine insight for the subsystem you're examining. **Frameworks 1-2 are the highest-value — simplifications and upstream-inspired changes account for the majority of passing patches.**

### 1. "What recently-added term might be noise?"

**This is the single highest-value framework.** Simplifications account for ~37% of upstream Stockfish patches that pass Fishtest. They run as non-regression tests `<-1.75, 0.25>` which only need to prove they don't LOSE Elo — a much lower bar than proving Elo gain.

For every term, condition, or code path added in the last 6 months (`git log upstream/master --oneline --since="6 months ago"`):
- Was it added as part of a larger feature, with this term as a secondary effect?
- Has the surrounding code changed enough that this term's original justification may no longer apply?
- Does removing it produce a simpler formula that bench-tests differently?

Simplification candidates: `std::clamp()` with wide bounds (is the clamp ever active?), complex conditional expressions where one branch dominates, terms with small coefficients relative to other terms in the same formula, dual conditions that overlap (both pruning the same cases).

A successful simplification diff should DELETE lines or terms, not add them.

### 2. "What's the natural next step after a recent upstream patch?"

Study the last 5-10 patches touching the subsystem you're examining (`git log upstream/master --oneline -20 -- src/<file>`). Each patch changes the local equilibrium — what was optimal before the patch may not be optimal after. Specifically:
- If upstream simplified formula A, maybe the SAME simplification applies to formula B (which is structurally similar but was not touched).
- If upstream added signal X to decision Y, maybe signal X is now ALSO useful for decision Z.
- If upstream removed term T from path P, maybe the same term T is also dead weight in path Q.

This "ripple effect" reasoning is how many upstream developers find patches: they study a recent pass, then look for the same principle applied elsewhere.

### 3. "What information is being destroyed?"

Where does the code collapse signal via `abs()`, `clamp()`, thresholds, or type coercion? Every `abs()` discards sign. Every `clamp()` destroys magnitude beyond the bounds. Every `bool()` cast collapses a spectrum to binary. Ask: is the destroyed information actually noise, or does it carry signal that downstream consumers could use?

### 4. "What does the search tree know here that it isn't using?"

At any decision point, inventory what's in scope: `ss->` fields, parent stack entries, TT data, history tables, position features (material, game phase, move count, rule50). Which of these are available but untouched? The most promising discoveries come from connecting an existing signal to an existing decision that currently ignores it.

### 5. "What assumption is baked into this formula's shape?"

Linear, quadratic, constant, piecewise, exponential — every formula embeds an assumption about how its inputs relate to the correct output. Is that assumption justified by chess theory, or is it an artifact of incremental tuning? A linear formula assumes uniform marginal returns. A constant assumes no relationship at all. Question the shape, not just the coefficients.

### 6. "Where do two subsystems make independent decisions that could be coordinated?"

Pruning and extension both decide whether to search deeper — but they don't talk to each other. Move ordering and reduction both assess move quality — with different signals. Evaluation and time management both estimate position difficulty — independently. Search and correction history both estimate eval accuracy — on different timescales. Look for places where coordination could reduce contradictions.

### 7. "What would a control systems engineer change?"

Think in terms of feedback loops, damping, overshoot, steady-state error, and phase lag. History tables are integrators — do they have appropriate decay (anti-windup)? Aspiration windows are a control loop — is the gain well-tuned for the error signal? Iterative deepening is a successive approximation — where does it converge slowly?

### 8. "What's the weakest link in the information chain?"

Trace a signal from creation to consumption. Example: a move's history score is created during update, stored in a table, read during move ordering, and used in LMR. Where in that chain is the signal most degraded — by table saturation, by stale entries, by lossy combination with other signals? Strengthening the weakest link matters more than amplifying the strongest.

### 9. "Where does the code treat different situations identically when they're not?"

Same pruning threshold for opening and endgame. Same reduction formula for quiet moves and checks. Same history update weight for deep and shallow searches. Same TT replacement policy for PV and non-PV nodes. Whenever you find a single formula applied to structurally different cases, ask whether differentiation would help.

### 10. "What's the implicit prior, and is it calibrated?"

Every default value, initial fill, and fallback constant is an implicit Bayesian prior. `captureHistory.fill(-689)` says "assume captures are slightly bad until proven otherwise." Is that calibrated to reality? What would the empirically correct prior be? Miscalibrated priors poison early iterations of search.

### 11. "Where is granularity being wasted or missing?"

Some decisions use rich, continuous signals (history scores spanning ±30000). Others use binary flags (`improving`, `ttPv`, `inCheck`). Is there a continuous signal available where a binary is used? Is a continuous signal being used where the function is actually step-shaped?

---

## Research Protocol

1. **Explore subsystems deeply, one at a time.** Use Explore agents to read entire files — don't skim. Understand the theoretical basis of each heuristic before looking for improvements.

2. **Map the information flow.** For each subsystem, trace: what inputs does it consume? What decisions does it influence? What signals are available but unused?

3. **Cross-reference relentlessly.** For every idea:
   - Check all existing TODO items (READY, SUBMITTED, KILLED) for overlap
   - Check killed patterns at the top of TODO.md
   - Verify the code at the exact lines you're targeting
   - Confirm variable availability and types

4. **Study upstream before proposing.** Run `git log upstream/master --oneline -50` and read diffs of recent passing patches in the subsystem you're exploring. Classify each as SIMPLIFY/ADD-SIGNAL/RESHAPE/REORDER/REFINE. Match your output distribution to upstream: ~40% simplifications, ~30% signal additions, ~30% other.

5. **Estimate impact direction and magnitude.** Will this make the engine search more or fewer nodes? Will it affect all positions or only a subset? Changes that affect a small fraction of positions dramatically are more likely to pass than changes that affect all positions slightly.

6. **Prefer ideas that are falsifiable at small sample sizes.** Fishtest kills bad ideas fast (often <5k games) but takes 50k–150k to confirm marginal gains. Ideas with clear directional predictions are better than "maybe this helps."

---

## Output Format

Each item must follow this exact structure:

```markdown
### N. Short descriptive title [READY]
- **Tier:** A|B|C|D | **File:** src/file.cpp:line | **Branch:** experiment/branch-name
- **Rationale:** Why this should work. Reference the thinking framework that inspired it. Explain what's structurally different from any related killed experiments.
~~~diff
- old code (exact, from source)
+ new code (must compile)
~~~
```

Notes:
- Tier S = simplifications that remove dead terms/conditions (run as non-regression `<-1.75, 0.25>`)
- Tier A = structural shape changes SPSA cannot reach (new conditionals, formula transformations)
- Tier B = under-tuned subsystems outside SPSA scope (template constants, iteration-level parameters)
- Tier C = asymmetries, arbitrary skips, inconsistent gating
- Tier D = parameter nudges, low probability
- Number items sequentially from the highest existing number in TODO.md
- Branch names must be `experiment/descriptive-slug`
- Diff blocks must show exact current code (verified by reading the source) and proposed change
- If a note is needed to explain the math or edge cases, add it after the diff block
