---
title: "Branch Prediction"
source: https://danluu.com/branch-prediction/
author: Dan Luu
company: Personal blog
date_posted: unknown
date_digested: 2026-10-19
---

# Branch Prediction

## What's new to learn

1. **Branch prediction is online supervised learning under hardware constraints**: a 2-bit saturating counter is a momentum-tracking threshold classifier; a correlating predictor is a feature-based linear model; a tournament predictor is a meta-learner (model selection); TAGE is multi-scale ensemble learning — the 50-year predictor evolution is a straight line through ML model capacity history.
2. **The pipeline-penalty framing makes the problem concrete**: at N pipeline stages deep, every misprediction flushes N cycles of work; at 3 GHz with 20 stages that is ~7 ns per miss; this makes even a 1%-improvement in accuracy worth entire silicon die areas, so the predictor is one of the most heavily optimized components in a modern CPU.
3. **Geometric history spacing is the TAGE insight**: to cover correlations at every timescale without starving any scale of training data, you use history lengths that double each level — L_i = α^i — so you spend half the storage at each longer scale, matching the empirical fact that longer-range patterns are geometrically rarer.

## Prerequisites

- **CPU pipelining**: modern CPUs overlap instruction execution across fetch → decode → issue → execute → write-back stages; a branch at fetch time creates a bubble because you don't know which instruction to fetch next until execute resolves the outcome.
- **Speculative execution**: to avoid the bubble, the CPU *guesses* the branch direction, fills the pipeline with the guessed path, and *rolls back* (flushes the pipeline) if wrong; the question is how good the guess is.
- **Branch patterns**: branches fall into two rough classes: loop back-edges (almost always taken), and if/else conditions (variable); the predictor must handle both.

## The core idea

Every conditional branch is a classification problem: given the branch's program counter address and some history of recent branch outcomes, predict *taken* or *not-taken* in one hardware cycle. You have a fixed silicon budget — kilobytes, not megabytes — and you process billions of branches per second.

The fundamental realization: **branch outcomes are not random; they correlate with history**. A loop branch is almost always taken until it isn't. An error-handling branch is almost never taken. Two branches in a tight loop often alternate: "if A taken last time, then B will be taken this time." Exploiting these correlations is the entire game.

The 50-year evolution from 1-bit counters to TAGE is a direct parallel to how supervised learning models grew in capacity — and it hit the same tradeoffs: more model capacity captures more signal but needs more training data, so you have to match capacity to the frequency of the pattern you are trying to predict.

## Mechanics

### Stage 0: Static prediction
Before hardware predictors, compilers could emit static hints or apply heuristics:
- *Always predict not-taken*: simplest, wrong for most loops.
- *BTFNT (Backward Taken, Forward Not Taken)*: backward branches are usually loop back-edges (taken); forward branches are usually if/else exits (not-taken). Gets ~65% accuracy with zero hardware.

### Stage 1: The bimodal predictor (1-bit, then 2-bit)

**1-bit predictor**: a table indexed by low-order bits of the branch PC, one bit per entry storing the last outcome. Predict = last outcome. This mispredicts twice per loop: once on exit, once on re-entry. For a loop that runs 100 iterations, that is 2% miss rate just from loop termination — poor.

**2-bit saturating counter**: replace each 1-bit entry with a 2-bit counter (4 states: Strongly Not Taken → Weakly Not Taken → Weakly Taken → Strongly Taken). Predict taken iff counter ≥ 2. Update: increment on taken, decrement on not-taken, saturate at 0 and 3. Two consecutive mis-outcomes are required to flip the prediction direction.

```
States:    SN (0) → WN (1) → WT (2) → ST (3)
Predict:   [0,1] → not-taken,   [2,3] → taken
Update:    taken → min(state+1, 3),   not-taken → max(state-1, 0)
```

The saturation means a single anomalous outcome (the last iteration of a loop, a rare error path) does not immediately flip the prediction — you need to be wrong *twice* before the predictor changes its mind. This reaches ~85–92% accuracy on integer benchmarks.

### Stage 2: Correlating predictors (two-level adaptive)

The key insight: branches correlate with *other recent branches*, not just their own history. If a null-pointer check just fired (branch A not-taken = no null), the subsequent bounds-check branch is probably also fine (taken). Knowing A's outcome lets you make a better prediction for B.

**Global History Register (GHR)**: a k-bit shift register recording the last k branch outcomes. Shift left on each branch, append 1 for taken, 0 for not-taken.

**Pattern History Table (PHT)**: 2^k entries of 2-bit counters. Index = GHR. On any branch, look up PHT[GHR], predict from that counter, then update it after resolution and shift GHR.

**gshare variant**: XOR the GHR with the branch PC's low bits to index the PHT. This reduces aliasing — two different branches with the same GHR value map to different PHT entries.

```
predict(PC, GHR):
    idx = GHR XOR PC[k:0]     # gshare mixing
    return PHT[idx] >= 2       # taken if counter >= WT

update(PC, GHR, taken):
    idx = GHR XOR PC[k:0]
    update_counter(PHT[idx], taken)
    GHR = (GHR << 1) | taken
```

This captures cross-branch correlation at the cost of one extra indirection. With k=8–12 history bits, accuracy climbs to ~93–97%. (Yeh & Patt 1991; McFarling 1993.)

**Local history predictor**: instead of one global GHR, maintain a per-branch Local History Table (LHT). Each branch PC indexes its own k-bit history, which then indexes the PHT. Better for branches with private periodic patterns; worse for branches that truly depend on other branches.

### Stage 3: Tournament predictor (Alpha 21264)

Both local and global predictors have blind spots: global history predicts correlated branches well but badly for periodic local branches; local history does the reverse. The solution: run **both** predictors and use a **meta-predictor** to select between them.

**Selector table**: one more table indexed by branch PC, each entry a 2-bit counter. Predict using local predictor if selector ≥ 2, else global. Update selector: increment toward local if local was right and global wrong; decrement if global was right and local wrong.

This is model selection / meta-learning: for each branch, the hardware learns *which predictor architecture fits best for this specific branch*. High-IPC loops gravitate toward local; correlated chains gravitate toward global. The Alpha 21264 used ~64 KB of predictor state in total and achieved ~98–99% accuracy on integer workloads. (Kessler 1998.)

### Stage 4: TAGE — tagged geometric history length predictor

TAGE (Seznec & Michaud 2006) is the dominant predictor design today. It wins Championship Branch Prediction (CBP) competitions and underlies Intel, AMD, and Apple Silicon predictors.

**Structure**: a base predictor T0 (bimodal table indexed by PC) plus N tagged component tables T1, T2, ..., TN.

**Geometric history lengths**: table Ti uses the last L_i branch outcomes as context, where:
```
L_0 = 0 (base, no history)
L_1 = 2, L_2 = 4, L_3 = 8, L_4 = 16, L_5 = 32, L_6 = 64, L_7 = 128
```
Each step roughly doubles the history length (geometric ratio α ≈ 2).

**Entry structure** for each Ti slot:
```
{
    ctr : 3-bit signed counter   // prediction: ≥ 0 → taken
    tag : t-bit tag              // hash of PC + last L_i history bits
    u   : 2-bit useful counter  // how recently this entry gave a correct prediction
}
```

**Prediction**:
1. Compute tag_i = hash(PC, last L_i bits of global history) for all i.
2. Find the Ti with the **longest history** (highest i) whose tag matches.
3. Use that Ti's `ctr` for prediction; if no match, use T0.

**Update**:
1. Update the matched component's `ctr` based on actual outcome.
2. Increment `u` if the matched component predicted correctly; decrement if the alternate would have been better.
3. On misprediction: try to **allocate** a new entry in a Ti with longer history than the current provider. Prefer slots with `u = 0` (not recently useful) to evict.

The allocation policy implements an online approximation to the EM algorithm: when the current model is wrong, try a higher-capacity model (longer history). The `u` counter is a staleness tracker that lets genuinely useful entries survive eviction pressure.

TAGE achieves ~99.5% accuracy, roughly 0.5 MPKI (mispredictions per kilo-instructions) on SPECint. A misprediction rate that was 10% in 1985 is now 0.05%.

## Where it breaks

**Aliasing**: multiple branches hash to the same PHT or Ti entry, corrupting each other's counters. Aliasing is destructive if two branches have opposite patterns (one usually-taken plus one usually-not-taken alias → accuracy approaches 50%). Larger tables, gshare mixing, and hashed tags all reduce aliasing at the cost of area.

**Cold start**: a predictor with no training history has no prediction advantage. JIT-compiled code, short-lived processes, and rare code paths all start from scratch. Ahead-of-time profile-guided branch hints are one mitigation.

**Indirect branches** (computed jumps): `jmp [rax]` jumps to a runtime-computed address; TAGE predicts direction (taken or not) but not target. These need a separate Indirect Target Predictor (ITTAGE — TAGE extended to predict target addresses). Virtual dispatch in OOP is a major source of indirect branches.

**Spectre exploitation**: TAGE's strength is also its Achilles' heel. An attacker who can run arbitrary code can train the predictor on their controlled inputs to mistrain it on a victim's branch. This is precisely the Spectre v1 and v2 attack surface. The predictor is *too good* at generalizing across contexts. Mitigations (IBRS, IBPB, eIBRS) flush or partition predictor state at privilege transitions, at significant IPC cost (~10–30% on I/O-bound workloads).

**Aliased indirect branch poisoning** (Spectre v2): the Global Branch Target Buffer is shared across processes; an attacker fills it with their own targets to redirect the victim's indirect branches speculatively. Requires hardware mitigations that degrade indirect branch prediction accuracy.

## Why it works

The deeper principle: **temporal autocorrelation in control flow is real and exploitable at multiple scales**.

Short-range correlations (last 2–4 branches): loop iterations, adjacent null checks. Long-range correlations (last 32–64 branches): outer-loop state influencing inner-loop branches. TAGE captures all of these *simultaneously* because each Ti component handles a different timescale, like a Fourier decomposition of branch history.

The geometric spacing is not arbitrary. It is the same reasoning as exponential smoothing in time-series models: phenomena that repeat at period T contribute signal at the T-bit history level. The number of distinct patterns of length T that repeat often enough to learn from falls geometrically as T grows (longer patterns are rarer), so you should allocate geometrically fewer entries at each longer scale. TAGE does exactly this: Ti has 2× fewer entries than T(i-1) while T(i) covers 2× longer history.

In ML terms:
- **Bimodal** = intercept-only logistic regression (1 feature: branch PC)
- **Correlating** = 1-layer perceptron with k binary features (GHR bits)
- **Tournament** = model selection / mixture of experts
- **TAGE** = multi-scale ensemble: each Ti is a feature-hashed 3-bit classifier at resolution L_i; the prediction is the output of the highest-resolution classifier that has seen this input before — like nearest-neighbor retrieval in a multi-scale index

The "longest match wins" rule in TAGE is structurally identical to longest-prefix match in IP routing: the more specific the context, the more trustworthy the prediction. The same principle underpins decision trees (deepest matching leaf = most specific prediction) and k-nearest neighbor (more specific context → lower variance).

## Going deeper

1. **Seznec & Michaud (2006), "A case for (partially) TAgged GEometric history length branch prediction"** — the original TAGE paper. Dense but the allocation policy section rewards rereading.
2. **"Branch Prediction Is Not a Solved Problem"** (Seznec et al., arXiv:1906.08170, 2019) — shows that even TAGE leaves 30–70% of mispredictions on the table for certain workloads, with neural predictors (LSTM, perceptron-based) catching some that TAGE misses. Illustrates the gap between hardware-constrained and software-unconstrained predictors.
3. **"Spectre Attacks: Exploiting Speculative Execution"** (Kocher et al., 2018) — already in this archive; reading it after this post reveals exactly which branch-predictor properties Spectre exploits and why each mitigation costs what it does.
