# Stockfish experimentation ideas (functional / Elo-oriented)

These are **10 non-overlapping, small-diff** experiment ideas aimed at Elo gain via search / move ordering / evaluation glue. Each is written to be **performance-conscious** (integer math, avoid extra work in hot loops) and “Fishtest-ready” as an isolated patch.

---

## 1) Add a *very limited* “quiet checks” stage to quiescence search

- **Target files**: `src/search.cpp`, `src/movepick.cpp`, `src/movepick.h`
- **Change sketch**:
  - Extend `MovePicker` with an additional qsearch stage that emits **quiet moves that give check** (filter `pos.gives_check(move)`), gated tightly:
    - Only when **not in check**, only for **PV qsearch** (or only when \( \beta - \alpha \) is small), and cap to a tiny number (e.g., first 2–4 candidates).
    - Keep existing SEE safety (e.g., reuse `pos.see_ge(move, -75)` like the quiet-check bonus already does).
- **Why it might gain Elo**: Current qsearch is captures-only, so some tactical “quiet check” continuations can be missed at the horizon. A tiny, gated check stage often improves tactical accuracy without huge node blowup.
- **Perf risk**: Moderate if ungated; low if PV-only + hard cap + SEE filter.
- **Validation**: Compare `bench` NPS (must be flat-ish) and tactical suites / fast STC fishtest.

### Testing

- **Experiment branch**: `exp-qsearch-quiet-checks`
- **Patch description (for Fishtest)**: Add a tiny quiet-check stage to PV quiescence search (strictly gated + SEE-filtered) to reduce tactical horizon misses.
- **Build**:

```bash
cd src && make clean && make -j profile-build ARCH=apple-silicon
```

- **Bench**:
  - **Nodes searched**: **2432811** (see `exp-qsearch-quiet-checks` branch for full bench output files)

### Results

(fill in later)

---

## 2) Make `goodQuietThreshold` adaptive instead of a constant

- **Target files**: `src/movepick.cpp`
- **Change sketch**:
  - Replace `constexpr int goodQuietThreshold = -14000;` with a cheap function of `depth`, `ply`, and/or “node importance”:
    - Example: raise threshold (be stricter) when depth is low / moveCount is large, and lower threshold (be more inclusive) at higher depths.
- **Why it might gain Elo**: The current fixed split between “good quiets” and “bad quiets” can be suboptimal across depths and positions; adapting it can improve early cutoffs and PV stability.
- **Perf risk**: Minimal (just arithmetic).
- **Validation**: `bench` signature should change; check for regressions in node growth / NPS.

### Testing

- **Experiment branch**: `exp-adaptive-goodquietthreshold`
- **Patch description (for Fishtest)**: Make the “good quiet” threshold depend on search depth to adapt how aggressively MovePicker splits good vs bad quiets.
- **Build**:

```bash
cd src && make clean && make -j profile-build ARCH=apple-silicon
```

- **Bench**:
  - **Nodes searched**: **2675481** (see `exp-adaptive-goodquietthreshold` branch for full bench output files)

### Results

(fill in later)

---

## 3) Rework the quiet “check bonus” from a step-function into a graded bonus

- **Target files**: `src/movepick.cpp`
- **Change sketch**:
  - Current quiet check bonus is a large fixed add:
    - `(... && pos.see_ge(m, -75)) * 16384`
  - Replace with a **graded** bonus based on cheap signals already computed:
    - scale by piece type (e.g., minor vs queen), by SEE margin bucket, or by whether the destination is threatened by a lesser piece (already computed as `threatByLesser`).
- **Why it might gain Elo**: Treating all “safe checks” equally can over-prioritize noisy checks. Grading helps keep truly forcing checks early while avoiding useless ones.
- **Perf risk**: Minimal (no new expensive calls; reuse existing `see_ge` that’s already evaluated here).
- **Validation**: Watch for NPS change (should be tiny) and PV quality in tactical test positions.

### Testing

- **Experiment branch**: `exp-graded-quiet-check-bonus`
- **Patch description (for Fishtest)**: Grade quiet-check ordering bonus by piece type (smaller for queen checks) to reduce noisy checks while keeping strong minor/pawn checks prioritized.
- **Build**:

```bash
cd src && make clean && make -j profile-build ARCH=apple-silicon
```

- **Bench**:
  - **Nodes searched**: **2861827** (see `exp-graded-quiet-check-bonus` branch for full bench output files)

### Results

(fill in later)

---

## 4) Tighten / relax capture SEE gating in `MovePicker` using a safer floor

- **Target files**: `src/movepick.cpp`
- **Change sketch**:
  - In `GOOD_CAPTURE`, the current SEE gate is `pos.see_ge(*cur, -cur->value / 18)`.
  - Add a **minimum** SEE requirement floor/ceiling to avoid pathological cases (e.g., extremely negative thresholds from large `cur->value`), or blend in victim value:
    - Example: `seeThreshold = max(-X, -cur->value/18)` with small `X`.
- **Why it might gain Elo**: Capture ordering and “good capture” classification are pivotal; making SEE gating less erratic can reduce tactical blunders and improve pruning reliability.
- **Perf risk**: Minimal (pure arithmetic).
- **Validation**: Ensure no big NPS regression; run a short tactics regression set.

---

## 5) Make quiet-move SEE pruning at Step 14 slightly history-aware

- **Target files**: `src/search.cpp`
- **Change sketch**:
  - Quiet pruning currently does: `if (!pos.see_ge(move, -25 * lmrDepth * lmrDepth)) continue;`
  - Make the SEE margin depend (lightly) on existing history aggregate already computed for quiets (continuation + main + pawn history), e.g.:
    - “Good history” → allow slightly worse SEE; “bad history” → require better SEE.
- **Why it might gain Elo**: Reduces false positives on strategically important quiets with good priors, and prunes more aggressively on historically bad moves.
- **Perf risk**: Minimal (reuse `history` already computed in the quiet branch).
- **Validation**: Verify signature changes; check for stability in endgames and sharp tactics.

---

## 6) Add a conservative zugzwang guard for Null Move Pruning (NMP)

- **Target files**: `src/search.cpp`
- **Change sketch**:
  - NMP currently checks `pos.non_pawn_material(us)` but zugzwang-like cases still happen.
  - Add a cheap extra guard in the NMP condition for “low mobility / low material” patterns:
    - e.g., “only minor pieces + pawns” and few pawns, or very low total non-pawn material, or near-zero `pos.rule50_count()` interactions.
- **Why it might gain Elo**: Avoids catastrophic NMP failures in zugzwang-ish endgames, improving tablebase-adjacent accuracy and practical play.
- **Perf risk**: Minimal (a couple of counts / material comparisons).
- **Validation**: Endgame regression + `bench` (NPS should not drop).

---

## 7) Improve ProbCut selectivity using capture history / staticEval context

- **Target files**: `src/search.cpp`, `src/movepick.cpp`
- **Change sketch**:
  - ProbCut uses a SEE threshold: `MovePicker(pos, ttMove, probCutBeta - ss->staticEval, &captureHistory)`.
  - Make the SEE threshold or `probCutDepth` formula slightly dependent on:
    - capture history of the candidate (already available) and/or
    - correction-adjusted confidence (e.g., `abs(correctionValue)` buckets).
- **Why it might gain Elo**: ProbCut is high leverage; better selectivity can prune more where safe and prune less where risky, improving both strength and node efficiency.
- **Perf risk**: Low if kept to arithmetic and existing values.
- **Validation**: `bench` + a targeted fishtest at rapid/fast TC (ProbCut is pruning-heavy there).

---

## 8) Adjust `update_all_stats()` malus scaling to reduce “over-punishing” late quiets

- **Target files**: `src/search.cpp`
- **Change sketch**:
  - Current malus is largely `malus = min(848*depth-207, 2446) - 17*moveCount`, with a mild decay after the 5th quiet.
  - Experiment with making the decay stronger (or earlier) for very late quiets to reduce noise:
    - e.g., decay based on `i` (quiet index) and `depth`, or cap total accumulated malus per node.
- **Why it might gain Elo**: History learning is powerful but can become noisy; reducing punitive updates for very late, low-quality moves can improve signal-to-noise in histories and thus ordering and LMR quality.
- **Perf risk**: None (update-time only; no extra movegen/search work).
- **Validation**: Look for improved stability (less PV thrash) and consistent Elo in STC.

---

## 9) Make continuation-history update weights depth- or node-type aware

- **Target files**: `src/search.cpp`
- **Change sketch**:
  - `update_continuation_histories()` uses fixed weights for offsets \(-1,-2,-3,-4,-6\).
  - Make weights slightly different for:
    - shallow depths vs deep (reduce long-range updates when depth is small),
    - in-check nodes (already partially handled), or PV vs NonPV.
- **Why it might gain Elo**: Continuation history is a major ordering feature; better biasing of which ply-links matter in which contexts can improve move ordering and pruning quality.
- **Perf risk**: None (update-time only).
- **Validation**: `bench` signature change; check no NPS regression; quick fishtest.

---

## 10) Refine the “evalDiff → history” learning rule used for quiet ordering

- **Target files**: `src/search.cpp`
- **Change sketch**:
  - Current rule updates main/pawn history using:
    - `evalDiff = clamp(-((ss-1)->staticEval + ss->staticEval), ...) + const`
  - Ideas to test (pick one):
    - use *difference* (`ss->staticEval - (ss-1)->staticEval`) instead of sum,
    - gate update on node type (NonPV only) or on `ss->ttHit`/`cutNode`,
    - scale by depth bucket (avoid overfitting at shallow depth).
- **Why it might gain Elo**: This update is explicitly meant to “improve quiet move ordering”; tuning it can change which quiets get prioritized early, affecting cutoffs and PV accuracy.
- **Perf risk**: None (update-time only).
- **Validation**: Ensure deterministic behavior is preserved; run `bench` and a small ordering-sensitive STC test.


